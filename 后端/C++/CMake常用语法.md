# CMake 常用语法与知识总结

> CMake 是 C/C++ 生态事实上的构建标准。它本身不编译代码，而是读取 `CMakeLists.txt`，为不同平台与工具链生成 Makefile、Ninja、Visual Studio 等构建文件。掌握 CMake，是从「能写 C++」到「能把工程交付出去」的关键一步。

## 一、CMake 是什么

- **定位**：构建系统生成器（Build System Generator），不是编译器，也不是构建系统本身。
- **三段式流程**：
  1. **配置（Configure）**：读取 `CMakeLists.txt`，解析变量、目标、依赖，生成缓存 `CMakeCache.txt`。
  2. **生成（Generate）**：输出底层构建文件（Makefile / build.ninja / *.sln）。
  3. **构建（Build）**：调用 Make/Ninja/MSBuild 真正编译链接。
- **跨平台**：一套 `CMakeLists.txt` 可切换 Linux/macOS/Windows、GCC/Clang/MSVC、Make/Ninja/VS。
- **版本与兼容**：以 `cmake_minimum_required()` 声明最低版本并锁定 policy 行为；现代写法建议 **CMake ≥ 3.16**，推荐 **3.20+**。
- **核心思想（现代 CMake）**：一切以 **Target（目标）** 为中心，用 `target_*` 命令描述「编译这个目标需要什么」，替代全局的 `include_directories` / `link_directories`。

## 二、最小可用示例与标准流程

### 最小 `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.16)   # 必须放在最前（对 target 生效前）
project(Demo VERSION 1.0 LANGUAGES CXX)

add_executable(demo main.cpp)          # 生成可执行文件 demo
```

### 命令行标准流程（out-of-source 构建）

```bash
cmake -S . -B build                                  # 配置：源码=当前目录，构建=build/
cmake --build build -j 8                             # 构建（并行）
./build/demo                                         # 运行
```

| 命令 | 说明 |
| ---- | ---- |
| `cmake -S <src> -B <build>` | 配置并生成构建文件（源目录 / 构建目录） |
| `cmake --build <build> -j N` | 构建，`-j N` 并行 N 个任务 |
| `cmake --build <build> --target <tgt>` | 只构建指定目标 |
| `cmake --build <build> --config Release` | 多配置生成器（VS/Xcode）选择配置 |
| `cmake --build <build> --clean-first` | 先清理再构建 |
| `cmake -S . -B build -DCMAKE_BUILD_TYPE=Release` | 单配置生成器指定构建类型 |
| `cmake -S . -B build -G Ninja` | 指定生成器（Make/Ninja/VS/Xcode） |
| `cmake -S . -B build -DCMAKE_CXX_COMPILER=clang++` | 指定编译器 |
| `cmake -S . -B build -LAH` | 列出所有缓存变量（含高级） |
| `cmake --install build --prefix /usr/local` | 安装到指定前缀 |
| `ctest --test-dir build` | 运行测试 |
| `cmake --fresh -S . -B build` | 忽略旧缓存，重新配置（3.24+） |

> **强烈建议 out-of-source 构建**：源码目录与构建目录分离，产物不污染源码，清理只需删 `build/`。

## 三、核心概念：以 Target 为中心

| 概念 | 说明 |
| ---- | ---- |
| Target（目标） | 构建的基本单元：可执行文件、库、自定义目标、导入目标等 |
| Property（属性） | 挂在目标上的配置，如 include 目录、编译选项、链接库 |
| Usage Requirement | 某个目标「被别人使用时」需要传递的信息（头文件路径、宏等） |
| Generator（生成器） | 底层构建系统，如 Unix Makefiles、Ninja、Visual Studio |
| Cache 变量 | 记录在 `CMakeCache.txt`，跨次配置保留，如 `CMAKE_BUILD_TYPE` |

### 目标类型

| 命令 | 类型 | 说明 |
| ---- | ---- | ---- |
| `add_executable(tgt src...)` | 可执行文件 | 最终程序 |
| `add_library(tgt STATIC src...)` | 静态库 | `.a` / `.lib` |
| `add_library(tgt SHARED src...)` | 动态库 | `.so` / `.dll` / `.dylib` |
| `add_library(tgt INTERFACE)` | 接口库（头文件库） | 只传递使用要求，不产出文件 |
| `add_library(tgt OBJECT src...)` | 对象库 | 只编译不链接，可被多目标复用 |
| `add_library(tgt ALIAS real)` | 别名目标 | 常配合命名空间，如 `add_library(mylib::mylib ALIAS mylib)` |
| `add_custom_target(name ...)` | 自定义目标 | 执行任意命令，如生成、格式化、文档 |

## 四、常用命令速查

| 命令 | 作用 |
| ---- | ---- |
| `cmake_minimum_required(VERSION 3.x)` | 声明最低版本与 policy |
| `project(Name VERSION 1.0 LANGUAGES C CXX)` | 声明工程名/版本/语言 |
| `add_executable` / `add_library` | 创建目标 |
| `add_subdirectory(dir)` | 加入子目录（含其 `CMakeLists.txt`） |
| `target_sources(tgt PRIVATE ...)` | 为已存在目标追加源文件 |
| `target_include_directories(tgt PUBLIC ...)` | 头文件搜索路径 |
| `target_link_libraries(tgt PRIVATE lib)` | 链接库 |
| `target_compile_options(tgt PRIVATE -Wall)` | 编译选项 |
| `target_compile_definitions(tgt PRIVATE DEBUG=1)` | 定义预处理宏 |
| `target_compile_features(tgt PRIVATE cxx_std_17)` | 要求语言特性/标准 |
| `set_target_properties(tgt PROPERTIES ...)` | 批量设置目标属性 |
| `find_package(Foo REQUIRED)` | 查找外部依赖 |
| `include(CMakeDependentOption)` | 引入内置模块（模块名即文件名） |
| `option(VAR "desc" ON)` | 定义布尔开关 |
| `set(VAR val CACHE STRING "desc")` | 定义缓存变量 |
| `message(STATUS "msg")` | 打印信息（STATUS/FATAL_ERROR/WARNING） |
| `configure_file(in out)` | 由模板生成文件（常注入编译期常量） |
| `file(GLOB ...)` | 收集文件（不推荐用于源文件，见「常见坑」） |
| `install(...)` | 定义安装规则 |
| `enable_testing()` + `add_test(...)` | 定义测试 |

## 五、变量与作用域

```cmake
set(MY_VAR "hello")                 # 普通变量
set(COUNT 3)                        # 数字也是字符串，比较时注意
set(OPT "x" CACHE STRING "说明")     # 缓存变量，持久化到 CMakeCache.txt
option(ENABLE_X "开启 X" ON)         # 等价于布尔型缓存变量

message(STATUS "value = ${MY_VAR}")  # 引用用 ${}
message(STATUS "env = $ENV{PATH}")   # 环境变量用 $ENV{}
```

- **大小写**：命令名不区分大小写（`add_executable` == `ADD_EXECUTABLE`），但**变量名区分大小写**。
- **作用域**：存在「目录作用域 + 函数作用域」两种：
  - `add_subdirectory` 创建子目录作用域：**父目录变量对子目录可见**，但子目录内 `set` **不会**回传父目录，除非 `set(... PARENT_SCOPE)`。
  - `function()` 内部是独立作用域，同样需要 `PARENT_SCOPE` 才能向外写。
  - `macro()` 是**文本替换**，没有独立作用域，会直接污染调用处。
- **列表**：用分号分隔的字符串即列表，`set(LIST a b c)`，遍历用 `foreach(item IN LISTS LIST)`。

### 常用内置变量

| 变量 | 含义 |
| ---- | ---- |
| `CMAKE_SOURCE_DIR` | 顶层源码目录 |
| `CMAKE_BINARY_DIR` | 顶层构建目录 |
| `PROJECT_SOURCE_DIR` / `PROJECT_BINARY_DIR` | 最近一次 `project()` 的目录 |
| `CMAKE_CURRENT_SOURCE_DIR` / `CMAKE_CURRENT_BINARY_DIR` | 当前处理的目录 |
| `CMAKE_CURRENT_LIST_DIR` / `CMAKE_CURRENT_LIST_FILE` | 当前 `CMakeLists.txt` 所在目录/路径 |
| `CMAKE_BUILD_TYPE` | Debug / Release / RelWithDebInfo / MinSizeRel（单配置） |
| `CMAKE_CXX_STANDARD` / `CMAKE_CXX_STANDARD_REQUIRED` | C++ 标准及是否强制 |
| `CMAKE_INSTALL_PREFIX` | 安装前缀 |
| `CMAKE_MODULE_PATH` | 额外 `FindXxx.cmake` 搜索路径 |
| `CMAKE_TOOLCHAIN_FILE` | 交叉编译工具链文件 |
| `CMAKE_EXPORT_COMPILE_COMMANDS` | 生成 `compile_commands.json`（供 clangd 等） |

```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)            # 关闭 GNU 扩展，严格 ISO
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)    # 生成 compile_commands.json
```

## 六、PUBLIC / PRIVATE / INTERFACE（现代 CMake 的灵魂）

这三者决定「使用要求」如何传播：

| 关键字 | 本目标编译需要 | 依赖它的目标需要 |
| ------ | :------------: | :--------------: |
| `PRIVATE` | ✅ | ❌ |
| `INTERFACE` | ❌ | ✅ |
| `PUBLIC` | ✅ | ✅（= PRIVATE + INTERFACE） |

```cmake
add_library(math STATIC math.cpp)

# 头文件目录：自己编需要(私有)，使用者也需要(公开) → PUBLIC
target_include_directories(math PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>)

# 内部实现细节，不暴露给使用者 → PRIVATE
target_compile_definitions(math PRIVATE MATH_INTERNAL=1)

add_executable(app main.cpp)
# 链接 math 后，app 自动获得 math 的 PUBLIC/INTERFACE 头文件路径
target_link_libraries(app PRIVATE math)
```

**记忆口诀**：`链接谁，就自动继承谁对外公开的使用要求`——头文件路径、宏、标准都由库作者声明，使用者无需重复。

## 七、依赖管理：find_package 与第三方库

```cmake
find_package(Threads REQUIRED)                    # 内置模块
find_package(OpenCV 4 REQUIRED COMPONENTS core imgproc)

target_link_libraries(app PRIVATE
    Threads::Threads                              # 现代库提供命名空间目标
    OpenCV::OpenCV)
```

- **MODULE 模式**：查找 `FindXxx.cmake`（内置或 `CMAKE_MODULE_PATH` 下），常用于系统库。
- **CONFIG 模式**：查找库自带的 `XxxConfig.cmake` / `xxx-config.cmake`，现代库首选。
- 结果变量：`Xxx_FOUND`、`Xxx_INCLUDE_DIRS`、`Xxx_LIBRARIES`；**优先用 `Xxx::Xxx` 导入目标**，而非裸变量。
- `REQUIRED`：找不到直接报错；`QUIET`：静默。

### 拉取源码依赖

```cmake
include(FetchContent)
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1)
FetchContent_MakeAvailable(fmt)                   # 拉取 + 加入构建
target_link_libraries(app PRIVATE fmt::fmt)
```

- `FetchContent`：配置阶段下载并 `add_subdirectory`，适合轻量源码依赖。
- `ExternalProject_Add`：构建阶段下载编译，适合非 CMake 工程。
- 包管理器：**vcpkg**（`vcpkg.cmake` 工具链 + 清单模式）、**Conan**（profile + 生成 `CMakeDeps`）。

## 八、条件、循环、函数与宏

```cmake
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(app PRIVATE -g -O0)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    target_compile_definitions(app PRIVATE WIN32_LEAN_AND_MEAN)
else()
    target_compile_options(app PRIVATE -O2)
endif()

foreach(file IN LISTS SRC_LIST)
    message(STATUS "src: ${file}")
endforeach()

foreach(i RANGE 1 3)              # 1 2 3
    message(STATUS "i=${i}")
endforeach()
```

### 常用判断形式

| 写法 | 含义 |
| ---- | ---- |
| `if(DEFINED VAR)` | 变量是否已定义 |
| `if(TARGET name)` | 目标是否存在 |
| `if(EXISTS path)` / `if(IS_DIRECTORY path)` | 文件/目录是否存在 |
| `if(VAR)` | 变量为「真值」 |
| `if(A STREQUAL B)` / `MATCHES` / `VERSION_LESS` | 字符串相等/正则/版本比较 |
| `if(NOT X)` / `if(A AND B)` / `if(A OR B)` | 逻辑运算 |

> **真值规则**：变量值为 `0`、`OFF`、`NO`、`FALSE`、`N`、`IGNORE`、`NOTFOUND`、空串，或以 `-NOTFOUND` 结尾时，判为 **假**，其余为真。
> **陷阱**：`if(VAR)` 会自动按「变量名」取值，所以**不要写 `if(${VAR})`**——当值含空格或为空时会解析出错。比较字符串请显式加引号：`if("${VAR}" STREQUAL "x")`。

### 函数 vs 宏

| 维度 | `function()` | `macro()` |
| ---- | ------------ | --------- |
| 作用域 | 独立作用域 | 无，文本替换 |
| 变量回传 | 需 `PARENT_SCOPE` | 直接修改外层（污染） |
| 参数 | `ARGN`/`ARGV`/`ARGC` | 同上 |
| 选择建议 | **默认用 function** | 仅在需要「文本替换」语义时用 macro |

```cmake
function(add_warnings tgt)
    target_compile_options(${tgt} PRIVATE -Wall -Wextra)
endfunction()

add_warnings(app)   # 复用编译选项
```

## 九、生成器表达式（Generator Expressions）

在**生成阶段**（而非配置阶段）才求值的表达式，形如 `$<...>`，用于按配置/平台/目标差异化设置：

| 表达式 | 含义 |
| ------ | ---- |
| `$<CONFIG:Debug>` | 当前配置是否为 Debug（1/0） |
| `$<$<CONFIG:Debug>:-g>` | 条件成立才加入 `-g` |
| `$<BUILD_INTERFACE:...>` | 仅在构建树中使用 |
| `$<INSTALL_INTERFACE:...>` | 仅在安装后使用 |
| `$<TARGET_FILE:tgt>` | 目标的产物完整路径 |
| `$<TARGET_PROPERTY:tgt,prop>` | 读取目标属性 |
| `$<IF:cond,true,false>` | 三元表达式 |

```cmake
target_compile_options(app PRIVATE
    $<$<CONFIG:Debug>:-g;-O0>
    $<$<CONFIG:Release>:-O2>)
```

## 十、安装、打包与预设

### install —— 定义安装规则

```cmake
include(GNUInstallDirs)                 # 提供标准目录变量

install(TARGETS math app
        EXPORT MathTargets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# 导出目标，供其他工程 find_package(Math) 使用
install(EXPORT MathTargets
        NAMESPACE math::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/math)
```

- `GNUInstallDirs` 提供 `bin` / `lib` / `include` 等标准相对路径。
- 安装后其他项目即可用 `find_package(Math CONFIG REQUIRED)` 找到并链接 `math::math`。
- 打包可配合 **CPack**：`include(CPack)` 后 `cpack` 生成 `.deb` / `.rpm` / `.zip` / NSIS 安装包。

### CMakePresets.json —— 固化常用配置

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "release",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release", "CMAKE_EXPORT_COMPILE_COMMANDS": "ON" }
    }
  ],
  "buildPresets": [{ "name": "release", "configurePreset": "release" }]
}
```

```bash
cmake --preset release
cmake --build --preset release
```

## 十一、现代 CMake 最佳实践

1. **一切围绕 Target**：用 `target_include_directories` / `target_link_libraries` / `target_compile_options` 替代全局的 `include_directories` / `link_directories`。
2. **明确可见性**：接口用 `PUBLIC`/`INTERFACE`，实现细节用 `PRIVATE`，别滥用 `PUBLIC`。
3. **源文件显式列出**，避免 `file(GLOB)`（新增文件不会重触发配置）。
4. **库给命名空间别名**：`add_library(mylib::mylib ALIAS mylib)`，使用端写法统一。
5. **声明 C++ 标准**用 `target_compile_features(tgt PRIVATE cxx_std_17)`，或至少设置 `CMAKE_CXX_STANDARD` + `CMAKE_CXX_STANDARD_REQUIRED ON`。
6. **out-of-source 构建**，并把 `build/`、`CMakeCache.txt`、`CMakeFiles/` 加入 `.gitignore`。
7. **out-of-source + 现代标准**：`cmake_minimum_required(VERSION 3.16)` 起，尽早声明以锁定 policy。
8. **可移植**：路径拼接用 `${CMAKE_CURRENT_SOURCE_DIR}/...`，判断平台用 `CMAKE_SYSTEM_NAME`，而非硬编码。
9. **可配置化**：用 `option()`/缓存变量暴露开关，用 `configure_file` 注入版本号等编译期常量。
10. **工具链友好**：开启 `CMAKE_EXPORT_COMPILE_COMMANDS`，供 clangd / IDE 索引。

## 十二、常见坑

- **`cmake_minimum_required` 必须在 `project()` 之前**，否则部分 policy 不生效。
- **别写 `if(${VAR})`**：`if` 会自动解引用变量名，加 `${}` 反而在值为空/含空格时出错。
- **`file(GLOB)` 收集源文件**：新增文件不会自动重跑 CMake，易踩「编译不到新文件」的坑。
- **变量修改不回传**：子目录或函数内 `set` 需 `PARENT_SCOPE`，否则父级看不到。
- **`CMAKE_BUILD_TYPE` 对多配置生成器无效**：VS/Xcode 下应使用 `--config Release`。
- **`link_directories` / `include_directories` 顺序敏感且全局污染**，现代项目应改用 `target_*`。
- **路径含空格要加引号**：`"${CMAKE_SOURCE_DIR}/my dir"`。
- **修改 `CMakeLists.txt` 后记得重新配置**；遇到诡异缓存问题可删 `build/` 或 `--fresh` 重来。
- **`project()` 之后的相对路径基于 `CMAKE_CURRENT_SOURCE_DIR`**，跨目录拼路径要显式基于目标目录。

## 十三、复习要点（面试向）

**Q1：CMake 和 Make 的区别？**
考察点：构建系统 vs 构建系统生成器。
要点：Make 直接用 Makefile 描述构建规则，平台/编译器绑定强；CMake 是更高层描述（跨平台、跨编译器），生成 Makefile/Ninja/VS 工程等，再用底层工具构建。CMake 管「生成」，Make 管「执行」。

**Q2：什么是 Target？为什么强调 target_* 命令？**
考察点：现代 CMake 核心模型。
要点：Target 是构建单元（可执行/库/自定义目标），属性挂在目标上，依赖信息随目标传递。`target_*` 让使用要求（头文件路径、宏、链接库）按依赖关系精确传播，避免全局命令的污染和顺序问题。

**Q3：PUBLIC / PRIVATE / INTERFACE 的区别？**
考察点：使用要求的可见性传递。
要点：PRIVATE 只自己用；INTERFACE 只给使用者用（自己不用，如纯头文件库）；PUBLIC 两者都用。链接一个目标时，会自动继承其 PUBLIC/INTERFACE 的使用要求。

**Q4：变量的作用域如何理解？**
考察点：目录作用域与函数作用域。
要点：父目录变量对子目录可见，子目录 `set` 不回传，需 `PARENT_SCOPE`；`function` 有独立作用域，`macro` 是文本替换无作用域。缓存变量跨配置持久化。

**Q5：find_package 的 MODULE 与 CONFIG 模式？**
考察点：依赖查找机制。
要点：MODULE 找 `FindXxx.cmake`（系统库常用）；CONFIG 找库安装时导出的 `XxxConfig.cmake`（现代库首选）。优先使用 `Xxx::Xxx` 导入目标而非裸变量。

**Q6：为什么 CMake 不推荐 file(GLOB) 收集源文件？**
考察点：构建正确性。
要点：GLOB 的结果在配置阶段确定，新增/删除源文件不会自动触发重新配置，导致漏编或残留；显式列出源文件更可靠，或在 `CONFIGURE_DEPENDS` 场景下谨慎使用。

**Q7：生成器表达式的作用？**
考察点：配置期 vs 生成期的差异。
要点：`$<...>` 在生成阶段求值，可按配置（Debug/Release）、平台、目标属性动态生成选项；`$<BUILD_INTERFACE>` / `$<INSTALL_INTERFACE>` 用于区分构建树与安装后的头文件路径。

**Q8：如何把一个库交付给他人使用？**
考察点：安装与导出。
要点：`install(TARGETS ... EXPORT ...)` 安装产物，`install(EXPORT ... NAMESPACE ...)` 导出目标，配合 `GNUInstallDirs` 与 config 文件，使他人可用 `find_package` 直接链接。

## 十四、速查表：一段完整的现代模板

```cmake
cmake_minimum_required(VERSION 3.16)
project(MyApp VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(MYAPP_BUILD_TESTS "构建测试" OFF)

# ---- 库 ----
add_library(mymath STATIC src/math.cpp)
target_include_directories(mymath PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>)
target_compile_options(mymath PRIVATE -Wall -Wextra)

# ---- 可执行文件 ----
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mymath)

if(MYAPP_BUILD_TESTS)
    enable_testing()
    add_executable(test_math tests/test_math.cpp)
    target_link_libraries(test_math PRIVATE mymath)
    add_test(NAME math COMMAND test_math)
endif()
```

## 推荐资料

- [CMake 官方文档](https://cmake.org/documentation/)（命令与变量权威查询）
- 《CMake Best Practices》：现代 CMake 工程实践
- 《CMake Cookbook》：按场景组织的配方集
- [Modern CMake](https://cliutils.gitlab.io/modern-cmake/)（在线，入门到进阶）
