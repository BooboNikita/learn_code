# React Native 基础

> React Native（RN）是 Meta 开源的跨端框架：用 React 的声明式范式写业务，用 JavaScript/TypeScript 描述 UI，最终**渲染成平台原生控件**（不是 WebView）。本文按「定位 → 架构 → 渲染 → 组件 → 原生互操作 → 工程化」梳理速查要点，配合 [React Native 面试题](ReactNative面试题.md) 使用。

## 一、React Native 是什么

### 1.1 定位与设计理念

- 一句话：**Learn once, write anywhere**（注意不是 Write once, run anywhere）。
- 三层结构：
  - **JS 层**：React 组件、业务逻辑，跑在 JS 引擎（Hermes）里。
  - **中间层**：C++ 核心（Yoga、Shadow Tree、JSI），负责布局计算与通信。
  - **原生层**：Android/iOS 原生控件（`android.view.View` / `UIView`），负责最终绘制。
- 核心理念：**声明式 UI（UI = f(state)）+ 原生渲染 + 平台能力复用**。
- JSX 是语法糖，最终由 Babel 转成 `React.createElement` 调用，与 React 完全同源。

### 1.2 与主流方案对比

| 维度 | React Native | H5 + WebView | Flutter | 纯原生 |
| --- | --- | --- | --- | --- |
| 渲染载体 | **平台原生控件** | 浏览器内核（DOM） | 自绘引擎（Skia/Impeller） | 原生控件 |
| 渲染线程 | 原生负责绘制，JS 只描述 | WebView 合成 | 自建管线（UI + Raster） | 平台 |
| 语言 | JS / TS（+ 少量原生） | JS / TS | Dart | Kotlin / Swift |
| 一致性 | 各平台控件**不完全一致** | 一致（浏览器) | **高度一致** | 各自原生 |
| 包体积 | 中（带 JS 引擎） | 小（带内核则大） | 大（带引擎） | 小 |
| 生态/热更新 | 生态最大，可热更新 | 可热更新 | 生态较小 | 不可热更 |
| 典型短板 | 桥接/异步带来的抖动（新架构已大幅改善） | 性能与体验上限低 | Dart 生态与包体积 | 双端成本高 |

> 关键辨析：**RN 不"画"UI，它"指挥"原生控件**；Flutter 才是自己画每一个像素。所以 RN 的视觉与手势天然贴合平台，但跨端一致性不如 Flutter。

### 1.3 技术栈全景

- 语言：JavaScript / **TypeScript（0.71 起官方模板默认 TS）**。
- UI：React（函数组件 + Hooks 为主）。
- 布局：**Yoga**（C++ 实现的 Flexbox 子集）+ `StyleSheet`。
- 打包：**Metro**（不是 Webpack）。
- 引擎：**Hermes**（RN 专用 JS 引擎，构建期编译为字节码）。
- 导航：React Navigation（JS 实现）/ react-native-navigation（原生导航）。
- 原生工程：Android（Gradle + Kotlin/Java）、iOS（CocoaPods + Objective-C/Swift）。

## 二、架构演进：从 Bridge 到新架构

理解 RN 的一切性能与"卡顿"话题，都绕不开这次架构重写。

### 2.1 老架构（Bridge，2015–2024）

```
┌─────────── JS 线程 ───────────┐
│  React 代码（Reconciler）      │
│  生成 UI 操作指令              │
└───────────────┬───────────────┘
                │ ① JSON 序列化
                ▼
┌──────────── Bridge（异步队列，批量投递）────────────┐
└───────────────┬───────────────────────┬────────────┘
                │                       │
                ▼                       ▼
        ┌───────────────┐      ┌──────────────────┐
        │ Shadow 线程    │      │ Native/UI 线程    │
        │ Yoga 计算布局  │─────▶│ 创建/更新原生控件  │
        └───────────────┘      └──────────────────┘
```

- **三线程**：JS 线程（跑 React 与业务）、Shadow 线程（跑 Yoga 布局）、Native/UI 主线程（创建与更新原生控件、处理手势与绘制）。
- **通信只能走 Bridge**：所有跨线程数据都被**序列化成 JSON 消息**放进异步队列，批量、定时投递。
- 副作用（后来成为新架构要解决的问题）：
  - **异步**：JS 与原生无法同步取值 → `measure` 只能回调，`setNativeProps` 变成"逃生舱"。
  - **序列化开销**：大数组/大对象过桥要编解码，是主要瓶颈。
  - **队列阻塞**：JS 线程一忙，UI 更新就排队 → 掉帧、白屏、手势无响应。
  - **启动昂贵**：所有 NativeModule 在启动时集中初始化。

### 2.2 新架构四件套

| 模块 | 定位 | 解决什么 |
| --- | --- | --- |
| **JSI**（JavaScript Interface） | JS 引擎与 C++ 之间的**通用抽象层**，引擎无关（Hermes/JSC/V8 都能接） | 让 JS 能**持有原生对象引用并同步调用**，取代 JSON 异步队列 |
| **TurboModules** | 新一代原生模块体系，基于 JSI，**按需懒加载 + 类型安全** | 启动时不再初始化所有模块；调用不再序列化 |
| **Fabric** | 新一代渲染器，**C++ 共享影子树**，同步布局与挂载 | 消除布局往返延迟，支持并发渲染（React 18/19 的 Concurrent、Suspense、Transition） |
| **Codegen** | 从 TS/Flow 类型定义**生成** JS ↔ 原生胶水代码（C++/Java/ObjC） | 接口编译期校验，去掉手写样板与运行时反射 |

- **Bridgeless（无桥模式）**：以上全部启用、把旧 Bridge **彻底移除**的最终形态。0.82 起成为**唯一**架构。

```
JS（Hermes 字节码）
   │  JSI：直接 C++ 调用 / 持有 HostObject（同步）
   ▼
React Common（C++）
   ├── Fabric 渲染器 ── React Shadow Tree（C++，JS 与原生共享）
   ├── Yoga 布局引擎
   └── TurboModules ──── 原生模块实现（Kotlin/Swift）
   ▼
原生 UI 线程：同步创建/挂载原生控件 → 上屏
```

### 2.3 深入：JSI 与 HostObject 机制

**为什么 JS 能和 C++ 直接通信？**

- 前提：**JS 引擎本身就是 C++ 库**。Hermes / JSC / V8 都是 C/C++ 实现，各自导出一套原生 API：JSC 的 `JSContextRef` / `JSObjectRef`，V8 的 `v8::Isolate` / `v8::FunctionTemplate`，Hermes 的 `HermesRuntime`。
- 所以"JS 调 C++"物理上就是 **C++ 调 C++**。RN 做的事不是"建立一条通道"，而是**把这些各不相同的原生 API 抽象成一套统一接口**，这就是 JSI。

```cpp
class Runtime {                       // jsi::Runtime，各引擎各自实现
 public:
  virtual Value evaluateJavaScript(...) = 0;
  virtual Object global() = 0;
  virtual Value getProperty(const Object&, const PropNameID&) = 0;
  virtual void setProperty(Object&, const PropNameID&, const Value&) = 0;
  virtual Function createFunctionFromHostFunction(...) = 0;
};
```

- `jsi::Value` **不是 JSON**，而是 C++ 里表示 JS 值的类型（tagged union；Hermes 进一步用 **NaN-boxing** 把类型和值压进 64 位），**可以持有 JS 堆对象的引用**——这是零拷贝的前提。
- 实现类：`HermesRuntime` / `JSCRuntime` / `V8Runtime`。上层（Fabric、TurboModules）只依赖 `jsi::Runtime`，不关心底层是谁。

**关键机制：HostObject / HostFunction**——让一个 C++ 对象在 JS 里"长得像普通对象"：

```cpp
class DeviceInfo : public jsi::HostObject {
 public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    if (name.utf8(rt) == "getBatteryLevel") {
      return jsi::Function::createFromHostFunction(
        rt, name, 0,
        [](jsi::Runtime&, const jsi::Value&, const jsi::Value*, size_t) {
          return jsi::Value(0.82);          // 直接返回，没有任何序列化
        });
    }
    return jsi::Value::undefined();
  }
};

rt.global().setProperty(rt, "DeviceInfo",
  jsi::Object::createFromHostObject(rt, std::make_shared<DeviceInfo>()));
```

JS 侧调用 `DeviceInfo.getBatteryLevel()` 的真实链路：

| 步骤 | 发生什么 |
| --- | --- |
| ① | 引擎做属性查找，发现这是 **HostObject** 类型的特殊对象（背后是个 C++ 指针） |
| ② | 引擎**回调** C++ 虚函数 `HostObject::get()`，同步、就地执行 |
| ③ | 返回一个 `jsi::Function`——它在 JS 堆里有身份，函数体是 C++ lambda |
| ④ | 引擎调用它，直接进入你的 C++ 代码 |
| ⑤ | 返回 `jsi::Value`，本来就在 JS 堆里，直接交给 JS |

- 方向是**"引擎回调 C++"（upcall）**，不是 JS 往 C++ 发消息。既然是函数调用，就天然同步、天然有返回值。
- **同一次调用栈 + 引用传递（而非值传递）** = 既不需要序列化，也不需要跨线程排队。这就是同步能力真正的来源。

**边界：JSI 只到 C++ 为止**

JSI 的**全部接口都是 C++ 类型**——`jsi::Runtime`、`HostObject`、`HostFunction`，没有任何 Java/ObjC 入口。所以经 JSI 调用的一定是 **C++ 函数**：

| 层 | 内容 | 如何被调用 |
| --- | --- | --- |
| JS 线程 / JS 堆 | 业务代码、React 渲染 | —— |
| **C++ 层（JSI 的边界）** | `jsi::Runtime`、`HostObject`、`HostFunction`、Yoga、Shadow Tree | JS 经 JSI **直接**调用 |
| 平台层 | Android 的 Java/Kotlin、iOS 的 ObjC/Swift | **由 C++ 中转**调用 |

平台代码并不在 JS 的调用目标里，它是 **C++ 那一边继续往下走**才到达的：

- **Android**：C++ 通过 **JNI**（RN 用 `facebook::jni` 封装）调到 Java/Kotlin。
- **iOS**：`.mm`（Objective-C++）里 C++ 与 ObjC 可混编，C++ 中能直接写 `[obj doSomething]` 或调用 Swift。

三条真实路径：

| 场景 | 链路 | 是否到平台层 |
| --- | --- | --- |
| 读布局、调 Hermes | JS → C++ → 返回 | ❌ 不到 |
| TurboModule | JS → Codegen 生成的 C++ spec（**纯转发**）→ JNI / ObjC++ → Java / ObjC | ✅ 到 |
| Fabric 挂载 | JS → C++ ShadowTree → mounting → JNI / ObjC++ → `View` / `UIView` | ✅ 到 |

所以答案要分两层：

- **JS 经 JSI 调用的目标 = C++ 方法**（确定、无例外）。
- **平台原生方法是 C++ 的实现细节**。里面是纯 C++ 还是转调平台，完全由那个 C++ 函数自己决定 —— 这也解释了为什么**纯 C++ 实现的 TurboModule 能双端共用一份**，不必写两遍 Java 和 ObjC。

> 术语提醒：老架构语境里的 "native module" 通常特指 **Java/ObjC 模块**（那时 C++ 不参与）；新架构语境里的 "native" 往往**包含 C++ 层**。回答时点明"我指平台层还是 C++ 层"，可以省掉一轮解释。

**代价与约束**

- **线程亲和性**：`jsi::Value` 只在所属 JS 线程上安全（Hermes 的 runtime 是单线程的）。Reanimated 的 worklet 跑在**另一个独立 JS runtime**（UI runtime）里，而非跨线程共享同一个 runtime。
- **生命周期与 GC**：`HostObject` 横跨 JS GC 与 C++ 所有权两个世界，跨边界持有对象要格外小心。
- **同步就可能阻塞**：一个慢的 C++ 实现会直接拖住 JS 线程，同步 API 要克制使用。
- **与引擎版本耦合**：JSI 是 C++ ABI 层面的东西，要跟引擎一起升级。

**类比**：`JSI : React Native ≈ N-API / node-addon-api : Node.js`。区别在目标——N-API 追求跨 Node 版本的 ABI 稳定，JSI 追求**引擎无关**（RN 本身完成过从 JSC 到 Hermes 的替换）。

### 2.4 深入：Fabric 渲染器

Fabric 是 RN 的**新渲染器**：C++ 重写、双端共享，取代老架构的 `UIManager` 渲染路径。它管着"三棵树"：

| 树 | 位置 | 职责 |
| --- | --- | --- |
| React Element 树 | JS 堆 | `render()` 返回的 React 元素，声明式描述 |
| **React Shadow 树** | **C++（ReactCommon）** | **不可变**的影子节点，持有 Yoga 布局结果 |
| Host View 树 | 原生 | 真实的 `UIView` / `android.view.View` |

四个核心机制：

1. **不可变影子树 + 克隆**
   - 更新时**克隆受影响的节点**（而非就地修改）→ 在克隆树上跑 Yoga → diff 新旧 → 产出 **mutation list**。
   - 为什么必须不可变：**不可变 = 可并发**。React 能在后台以低优先级准备一棵树，被更高优先级的工作打断时**整棵丢弃**——因为它从未碰过用户正在看的视图。
   - 这是 React 18/19 的 `useTransition` / `Suspense` / 优先级调度能在 RN 上真正生效的前提，也是 Fabric 与老渲染器最本质的区别。
2. **同步布局（靠 JSI）**
   - 布局可在提交前**同步读取**，`onLayout` 的时序竞态与 `measure(callback)` 的往返基本消失；文本测量在 C++ 里统一做，双端结果一致。
   - 为什么"换了渲染器就变同步"，见下方 [为什么可能同步](#为什么可能同步机制拆解)。
3. **Commit → Mounting**
   - 提交阶段把 mutation list（创建/更新/删除节点）交给 mounting 层执行；mounting 可按更新优先级调度到 UI 线程或 JS 线程。
4. **视图扁平化（View Flattening）**
   - 只用于布局、没有任何视觉属性的 `View` 可以**完全不创建宿主视图**，深层嵌套树的实际视图数明显减少。
   - 副作用：对被"拍平"的 View 调 `findNodeHandle` 拿不到东西。

#### 为什么可能同步（机制拆解）

"同步"不是 Fabric 优化出来的，而是**影子树换了位置**之后自然出现的结果。

**老架构的异步是架构选择，不是技术限制：**

1. 影子树在**原生侧**（Java/ObjC），JS 侧只有 React 的元素/实例树——布局结果 JS 根本看不到。
2. 两边之间**只有 Bridge 一条通道**：一个消息队列，跨线程投递 + JSON 序列化。
3. 队列语义决定了**投递方无法决定对端何时处理**。想拿返回值就只能回调，于是 `measure` 必然是"发起 → 等 → 回调"三段式。

**Fabric 改变了三件事，缺一不可：**

| 条件 | 老架构 | Fabric |
| --- | --- | --- |
| 影子树位置 | 原生侧，JS 够不着 | **C++ 层**，与 JS 引擎同进程、同地址空间 |
| 访问方式 | Bridge 消息（**值传递** + 排队） | **JSI 函数调用**（**引用传递** + 直接压栈） |
| 布局计算的性质 | —— | Yoga 是**纯计算**：输入节点树 + 约束，输出 frame，无副作用、不依赖 UI 线程状态 |

三条合起来，`measure()` 就退化成一次普通 C++ 函数调用：JS 线程压栈 → 进 C++ 影子树 → 读（必要时先算）布局 → **在同一栈帧返回**。既然是函数调用，就天然有返回值、天然同步。

> 关键结论：**Yoga 布局在两边都是同步算的——差别只在 JS 能不能拿到。**

#### 同步的边界（别答错）

同步的是「JS ↔ 影子树 / C++」这一段，**不是整条 UI 更新链路**：

| 环节 | 是否同步 |
| --- | --- |
| JS 读布局、读写影子树属性 | ✅ 同步 |
| TurboModule 声明的同步方法 | ✅ 同步 |
| `setState` → render → diff | ❌ 仍是 React 自己的调度 |
| commit → mounting → 上屏 | ❌ mount 仍可按优先级调度到 UI 或 JS 线程 |

所以 `setState` 之后**不能立即**通过 ref 读到新布局，那要等下一次 commit。另外，同步调用跑在 JS 线程的调用栈上——**一个慢的 C++ 实现会直接卡住 JS 线程**，这也是为什么并非所有 API 都被做成同步的。

**事件优先级**：离散事件（点击）以更高优先级分发，连续事件（滚动）较低——点一下不会排在滚动更新队列的后面。

**迁移影响**

| 老 API | 现在 |
| --- | --- |
| `UIManager`（JS 侧命令式操作） | Fabric UIManager（C++），不再直接使用 |
| `requireNativeComponent` + `RCTViewManager` | `codegenNativeComponent` + Fabric Component |
| `UIManager.dispatchViewManagerCommand` | `codegenNativeCommands` |
| `findNodeHandle` + `measure` | 直接在原生 ref 上调 `.measure()` |
| `setNativeProps` | **不支持**，改用 state / `Animated` / Reanimated |

- 老视图组件靠 **interop 兼容层**仍能运行，但 prop 更新可能滞后一帧，属于过渡方案（不要长期依赖）。

### 2.5 新老架构对比

| 维度 | 老架构 | 新架构 |
| --- | --- | --- |
| 通信方式 | Bridge，异步 + JSON 序列化 | JSI，同步 C++ 调用 |
| 影子树位置 | 原生侧（Java/ObjC） | **C++ 共享层**（JS 与原生都能访问） |
| 布局计算 | Shadow 线程异步 | C++ 同步（可用并发渲染调度） |
| 原生模块 | 启动时全量初始化 | TurboModule 懒加载 |
| 类型安全 | 运行时反射，易错 | Codegen 编译期校验 |
| 并发特性 | 不支持 | 支持 React 18/19 Concurrent / Suspense |
| 兼容性 | —— | 旧模块/旧视图需迁移，否则走 interop 兼容层 |

### 2.6 版本演进要点

| 版本 | 关键变化 |
| --- | --- |
| 0.59 | 支持 Hooks |
| 0.60 | 迁移 AndroidX、引入 **autolinking**（自动链接原生依赖） |
| 0.62 | 默认集成 Flipper 调试 |
| 0.64 | iOS 可选启用 Hermes |
| 0.68 | **新架构 opt-in**（Fabric / TurboModules 可试用） |
| 0.70 | Hermes 成为 Android 默认引擎 |
| 0.71 | 官方模板默认 **TypeScript** |
| 0.74 | 内置 **Yoga 3.0**（与 Web 的 Flexbox 语义更对齐） |
| **0.76**（2024.10） | **新架构默认开启**（仍可用 `newArchEnabled=false` 回退） |
| 0.81 | **最后一个支持旧架构的版本**；iOS 预编译构建开始试验 |
| **0.82**（2025.10） | **新架构成为唯一架构**，回退开关失效；Hermes V1 实验性 opt-in；升级到 React 19.1.1 |
| 0.84 | **Hermes V1 成为默认引擎**；iOS 默认下载预编译 `.xcframework`（编译时间大幅下降） |
| 0.85 | 移除旧桥最后的核心实现（桥彻底"死"了） |
| 0.86（2026.06） | 持续演进；新架构 + Hermes V1 成为生态基线 |

> 记忆锚点：**0.76 默认新架构 → 0.81 最后能回退 → 0.82 只能新架构**。做版本升级时，**别跨版本硬跳**，先落到 0.81 打开新架构跑通，再升 0.82+。

## 三、线程模型

### 3.1 三条线程的分工

| 线程 | 职责 | 不该做什么 |
| --- | --- | --- |
| JS 线程 | 执行 JS Bundle、React 渲染（Reconciler）、业务逻辑、状态更新 | 重计算、大 JSON 解析、同步阻塞 |
| UI / 主线程（原生） | 创建/更新原生控件、手势与触摸分发、绘制、系统回调 | —— |
| Shadow / 后台线程 | Yoga 布局计算、影子树维护（新架构下移至 C++ 共享层） | —— |

- **JS 是单线程的**：所有组件渲染与业务逻辑排队执行，一段耗时 JS 会**同时**卡住 UI 更新与手势响应。
- 因此重活要"切线程"或"交给原生" —— **两者动机不同，别混为一谈**：
  - **交到别的 JS 运行时**（只解决"阻塞"，**不会更快**）：Reanimated 的 **worklet** 跑在独立的 UI runtime、`react-native-worklets` / 多 JS 线程方案、WebView 变通方案。它们**仍然是 JS，单次计算速度不变**。
  - **交给原生**（真正的加速）：在 TurboModule 里用 C++ 跑 —— 收益来自**多核并行、SIMD、以及能链接成熟原生库**，详见 [8.4 落地场景](#84-落地场景把-cpu-密集计算交给原生)。
  - 轻量动画 / 手势：`Animated` + `useNativeDriver`，属性更新直接在原生侧驱动，不下发 JS。
- 新架构下，布局与挂载不再往返 JS 线程，**卡顿的主要来源收敛为"JS 线程执行时间 + 原生渲染耗时"两项**。

### 3.2 与 Android/iOS 线程的对应关系

- RN 的"UI 线程"就是 Android 的 main thread / iOS 的 main thread，**原生模块默认也在这个线程**——这一点和 Android 原生开发一致，也是排查 ANR（Android 无响应）的切入点。

## 四、渲染流程

### 4.1 一次更新的完整链路

```
① setState / 状态变化
        ▼
② React Reconciler（ReactFabric）在 JS 线程做 diff
        ▼
③ 生成/更新 React Shadow Tree（C++，节点是 ReactShadowNode）
        ▼
④ Yoga 在 C++ 中计算布局（测量 → 定位两趟递归，flexbox 子集）
        ▼
⑤ Commit：新树提交 + 挂载指令（Mounting）
        ▼
⑥ UI 线程同步创建/更新原生控件 → 交给系统绘制上屏
```

- 新架构的关键点：**影子树在 C++**，JS 侧与原生侧看到的是同一份布局结果，布局可以在提交前同步拿到。
- 老架构等价流程由 Bridge 承担 ③④⑤ 的排队与序列化，所以才有明显延迟。

### 4.2 Yoga：布局计算流程

**定位**：RN 的布局引擎，用 C/C++ 实现 Flexbox 子集，双端共用同一份算法 → 三端布局一致。它本身是个独立的通用库，RN 只是它的调用方之一。

#### 4.2.1 入口与递归

- 唯一入口 `YGNodeCalculateLayout(root, availableWidth, availableHeight, direction)`，内部转给递归函数 `calculateLayoutInternal`。
- 计算是**自顶向下的一次递归**：每个节点要么被完整解算一次，要么**整棵子树被跳过**（见 4.2.5，另见 4.2.3 对"自顶向下"的辨析）。
- 解算完把 `left/top/width/height`（相对父节点的 frame）写进节点的布局缓存 `cachedLayout`，RN 侧据此生成 `LayoutMetrics`。

#### 4.2.2 单个节点内部：四步

| 步骤 | 做什么 | 关键点 |
| --- | --- | --- |
| ① 测量主轴基准 | 为每个子节点解析 `computedFlexBasis` | 优先级：`flex-basis` → 主轴上的确定尺寸 → **递归 measure 子节点** |
| ② 分行与弹性分配 | 按 `flexWrap` 组成 FlexLine，再用 `flexGrow` / `flexShrink` 分配主轴的剩余 / 不足空间 | 主轴定下来后，交叉轴才由内容决定 |
| ③ 对齐 | `justifyContent`（主轴）、`alignItems` / `alignSelf`（交叉轴）、`alignContent`（多行之间）、基线对齐 | 基线对齐需要 **baseline 函数**，文本行才有共同基线 |
| ④ 定位与像素圆整 | 写回 `left/top/width/height`，按 `pointScaleFactor` 圆整到物理像素 | 不圆整会出现亚像素模糊与 1px 缝隙；RN 传入的是 `PixelRatio.get()` |

- **绝对定位节点在主流程之后单独处理**：它们不参与 flex 空间分配，但要等父节点的 padding/border 与 `position: static` 的祖先确定位置后才计算。
- 各步骤对应的源码文件：`algorithm/CalculateLayout.cpp`（主流程）、`algorithm/FlexLine.cpp`（分行）、`algorithm/Baseline.cpp`（基线）、`algorithm/PixelGrid.cpp`（圆整）。

#### 4.2.3 「自顶向下」指的是约束方向，不是信息只往下走

先破一个疑惑：子节点的内容尺寸**确实会决定父节点的大小**（`height: 'auto'` 的容器高度就是被子节点撑出来的）。那为什么还说自顶向下？

因为「自顶向下」描述的只是**约束（可用空间）的传递方向**。一次递归里信息实际走三次，方向是交替的：

| 次序 | 方向 | 传什么 |
| --- | --- | --- |
| ① 约束下传 | 父 → 子 | `availableWidth/Height` + 测量模式（SizingMode） |
| ② 尺寸上传 | 子 → 父 | 子节点的自然尺寸（`computedFlexBasis`，**只是假设值**） |
| ③ 定位下传 | 父 → 子 | `left/top`（相对父节点的 frame）+ 对齐/拉伸后的最终尺寸 |

**但这三次不是三次全局扫描，而是围绕每个节点、在局部交织发生的。** 宏观上整轮布局只有**两趟递归遍历**：

| 趟次 | `performLayout` | 信息方向 | 做什么 |
| --- | --- | --- | --- |
| 测量趟 | `false` | **下行 + 上行** | 下行是函数调用（传约束），上行是它的**返回值**（传尺寸）；只定尺寸，不写 position |
| 定位趟 | `true` | **纯下行** | 写 `left/top` 与最终尺寸；再次进入时命中缓存，不重复测量 |

关键在于：**① 和 ② 属于同一趟、同一次递归调用**——"下行"是调用展开，"上行"是它的返回值，二者是同一次调用/返回的两端，不是两次独立的遍历。只有 ③ 才是第二次进入。子节点的调用区间**嵌套**在父节点的区间内，所以「下 - 上」这个模式在树的每一层都重复一次，而不是整棵树先统一走完下行、再统一走完上行。

> 一句话记：**两趟遍历，三种方向；下行是调用，上行是返回，第二趟才是定位。**

所以准确的心智模型是**递归下降 + 回溯**（树的深度优先后序处理），而不是单向扫描。

**为什么必须两趟**：这里存在一个真实的依赖环——父节点要先知道自己的尺寸，才能给子节点定位（`left/top` 相对父节点、`alignItems: stretch` 与交叉轴对齐都依赖父尺寸）；而父节点在内容驱动时，又必须等所有子节点测完才知道自己多大。**破环的唯一办法就是把「测量」和「定位」拆成两趟**——这正是 4.2.2 那四步为什么是这个顺序。

几个直接推论：

- **测量结果是"假设值"**：`flex-basis` 只给出主轴上的猜测尺寸，要等弹性分配（`flexGrow`/`flexShrink`）之后才定最终值。所以同一个子节点可能被以**不同约束测量多次**——这正是 measure cache 存在的理由。
- **父节点尺寸不确定时，该轴上的百分比退化为 `auto`**：`height: '50%'` 只有父高度确定才有意义，否则就是循环依赖，只能放弃百分比。
- **滚动容器是"内容驱动"的极端**：滚动轴上给子节点的是 `Undefined`（max-content），即父节点根本不约束子节点，内容想多大就多大。
- **最终尺寸要读布局结果**，不要读 measure 的返回值——那可能只是某一轮的假设值。

#### 4.2.4 三种测量模式（SizingMode）

递归下传时，每个轴都带一个**约束模式**，对应 CSS 的三种尺寸语义：

| 内部名 | 公开名 | 含义 | 对应 CSS |
| --- | --- | --- | --- |
| `MaxContent` | `YGMeasureModeUndefined` | 无约束，返回内容自然尺寸 | `max-content` |
| `FitContent` | `YGMeasureModeAtMost` | 不超过给定值 | `fit-content` |
| `StretchFit` | `YGMeasureModeExactly` | 必须正好是这个尺寸 | `stretch-fit` |

- 模式由上下文决定：父节点沿交叉轴拉伸子节点时是 `Exactly`；`alignItems: flex-start` 时是 `AtMost`；滚动容器在滚动轴上是 `Undefined`。
- 模式**同时是缓存的 key**，模式变了缓存必然失效。

#### 4.2.5 measure 函数、缓存与脏跳过

**measure 函数：Yoga 怎么知道文本有多高**

- Yoga 是通用布局引擎，**不知道**文本 / 图片的自然尺寸。做法是给叶子节点挂一个 **measure 函数**（`YGNodeSetMeasureFunc`），需要时才回调测量；用 `void*` context 把应用数据带进回调。
- **挂 measure 函数的节点不能有布局子节点**（否则直接抛错）——对应 RN 里 `Text` 内部的文本节点。
- 注意"**不要假设 measure 一定会被调用**"：节点已有确定尺寸时会走缓存。最终尺寸要读节点的布局结果，而不是自己缓存 measure 的返回值。

**两类缓存 + 脏跳过**

| 机制 | 位置 | 作用 |
| --- | --- | --- |
| `cachedLayout` / `cachedMeasurements[]` | Yoga 每个节点 | 按 `(width, widthMode, height, heightMode)` 复用测量结果，避免重复调用 measure 函数 |
| 脏标记 + `generationCount` | Yoga `calculateLayoutInternal` | 未脏且 owner 方向未变的子树**整棵跳过**，直接复用上次结果 |
| `dirtyLayout()` → `layoutIfNeeded()` | RN 影子树 | props 变化时把祖先链标脏；commit 时**只在树脏时才调 Yoga** |

- 这是**两层独立的跳过**：RN 可能整轮都不算，Yoga 内部还会逐子树细粒度跳过。所以"改了样式但布局没重算"和"没改样式却全树重算"都有可能发生，排查时先确认脏标记的来源。
- 脏标记是**向上传播**的：任何子节点标脏，从根到它的整条路径都要重算。

#### 4.2.6 RN 侧的接入点

- **`YogaLayoutableShadowNode`**：持有 `YGNode`，把 `ViewProps` 转成 Yoga style（`setYogaProps`），并把子影子节点挂成 Yoga 的子节点。
- **文本测量**走 `TextShadowNode` 上的 measure 函数（内部是 `TextLayoutManager` + 文本布局缓存）——这也是**文本内容变化必须标脏重新布局**的原因。
- 布局结果 → `LayoutMetrics`（frame: origin + size）→ mounting 阶段据此设置原生视图的位置与尺寸；`onLayout` 回调与 `measure()` 读的都同一份数据。
- RN 里 `YGConfigSetUseWebDefaults` 是**关闭**的，所以 `flexDirection` 默认是 `column`（见 4.3）。

#### 4.2.7 为什么快（与浏览器对比）

| 维度 | 浏览器 CSS | Yoga |
| --- | --- | --- |
| 计算性质 | 与 DOM / 渲染树耦合 | **纯计算**：输入节点树 + 约束，输出 frame，无副作用 |
| 内容尺寸来源 | 布局过程中反复回查 DOM → **reflow 抖动** | 由 measure 函数**预先提供**，且有缓存 |
| 线程 | 主线程，会阻塞渲染 | 可离线 / 后台算（老架构 Shadow 线程；新架构在 C++ 层直接算） |
| 尺寸传播 | 复杂的多轮 reflow | 测量 + 定位两趟递归走完，不回查外部状态 |
| 像素对齐 | 设备像素比 + 独立合成 | 一次 `pointScaleFactor` 圆整 |

> **一个常见误答**："Yoga 是单遍布局"并不准确。它是"**测量（定 flex basis）+ 定位**"两趟递归，只是在同一轮遍历里完成、不需要回查外部状态，因此没有浏览器那种多轮 reflow。面试时把"两趟但无外部往返"讲出来，比笼统说"单遍"更站得住。

**版本**：对齐 Web 的改动在 **Yoga 3.0（RN 0.74）**落地——`gap`、`flex` 语义更接近 CSS，新增 `alignContent: 'space-evenly'`，并修正了 `row-reverse` 下 margin/padding/border 边缘翻转的老 bug（**这是 breaking change**，依赖旧行为的代码需要调整）。

### 4.3 Flexbox：RN 与 Web 的差异（高频考点）

| 项 | Web | React Native |
| --- | --- | --- |
| `flexDirection` 默认值 | `row` | **`column`** |
| `flex: 1` 含义 | grow 1 / shrink 1 / **basis 0%** | grow 1 / shrink 1 / **basis 0%**（行为接近） |
| `display` | 有 `block/inline/flex/grid` | 只有 flex 语义，`display: 'none'` 用于隐藏 |
| 单位 | px / em / rem / % | **无单位数字 = dp**（密度无关像素），不支持 % 之外的单位 |
| 文本 | 任意元素可含文本 | **文本必须包在 `<Text>` 里**（`View` 不能直接放字符串） |
| 样式继承 | 大多数属性可继承 | **不可继承**（仅嵌套 `<Text>` 内部继承字体相关） |
| 定位 | `static/relative/absolute/fixed/sticky` | 只有 `relative`（默认）与 `absolute` |
| 媒体查询 | 支持 | 无，用 `useWindowDimensions` / `Dimensions` |
| 阴影 | `box-shadow` | iOS `shadowColor/Offset/Opacity/Radius`；Android 用 `elevation` |
| 圆角/边框 | `border-radius` | `borderRadius`，不支持不同边不同色（部分） |

### 4.4 样式系统

- `StyleSheet.create({...})`：返回**注册后的 ID**，避免每次渲染新建对象，也便于静态校验。
- 样式属性是 **camelCase**（`backgroundColor`），值是数字或字符串，不是 CSS 字符串。
- 数组写法即"优先级合并"：`style={[styles.a, active && styles.b, {color: 'red'}]}`，后者覆盖前者。
- `StyleSheet.flatten` / `StyleSheet.compose` 用于合并；`StyleSheet.hairlineWidth` 是最细（1 物理像素）线宽。
- 主题方案：Context + 自定义 hook，或 `react-native-unistyles` / NativeWind（Tailwind 风格）等。

### 4.5 PixelRatio 与单位

- `dp`（Android）/ `pt`（iOS）是逻辑单位，`PixelRatio.get()` 拿到设备像素比，`PixelRatio.roundToNearestPixel()` 把尺寸对齐到物理像素以避免毛边。
- 设计稿按 750 / 375 宽出图时，通常按 375 基准换算或封装 `scale` 工具。

## 五、核心组件与 API

### 5.1 基础组件对应关系

| RN 组件 | 类比（Web / 原生） | 说明 |
| --- | --- | --- |
| `View` | `div` / `ViewGroup` | 布局容器，**不能直接放文本** |
| `Text` | `span` / `TextView` | 唯一承载文本的组件，支持 `numberOfLines`、`onPress` |
| `Image` | `img` / `ImageView` | `source={{uri}}` 或 `require('./a.png')`；支持 `resizeMode` |
| `ScrollView` | `overflow: scroll` | **一次性渲染全部子元素**，只适合少量内容 |
| `FlatList` | 虚拟列表 | 长列表首选，基于 `VirtualizedList` 懒渲染 |
| `SectionList` | 分组列表 | 带分组的 FlatList |
| `TextInput` | `input` | 受控/非受控、`keyboardType`、`returnKeyType` |
| `Pressable` | 按钮基元 | 新推荐，支持 `pressed` 状态回调 |
| `TouchableOpacity/Highlight` | 旧按钮 | 旧 API，仍广泛存在 |
| `Modal` | 弹窗 | 原生弹层 |
| `SafeAreaView` | —— | 适配刘海/安全区 |
| `KeyboardAvoidingView` | —— | 键盘遮挡处理 |
| `RefreshControl` | 下拉刷新 | 配 `FlatList.refreshing` |
| `ActivityIndicator` | loading | 原生菊花 |
| `Switch` / `Slider` | 开关 / 滑块 | —— |

### 5.2 常用 API / 模块

- `Dimensions`、`useWindowDimensions()`、`Platform.OS` / `Platform.select()`、`PixelRatio`。
- `AppState`（前后台）、`Linking`（唤起外部 App / 深链）、`PermissionsAndroid`、`Alert`、`Vibration`。
- `Keyboard`、`LayoutAnimation`、`InteractionManager`（等动画/手势结束再做重活）。
- `NativeModules`、`NativeEventEmitter` / `DeviceEventEmitter`（原生事件）。
- `Animated`、`PanResponder`（手势）。
- `StyleSheet`、`AccessibilityInfo`、`AppRegistry`。

### 5.3 组件与生命周期

- **函数组件 + Hooks 是现代写法**：`useState` / `useEffect` / `useMemo` / `useCallback` / `useRef` / 自定义 Hook。
- `useEffect` 的清理函数对应老架构 `componentWillUnmount`：**取消订阅、清定时器、中断请求**，否则泄漏。
- `useMemo` / `useCallback` / `React.memo` 是列表与子树渲染优化的主力，但**不要滥用**（本身有比较成本）。
- `ErrorBoundary`（类组件 + `componentDidCatch`）捕获子树渲染错误，做兜底 UI；**无法捕获事件回调与异步错误**（要全局 `ErrorUtils.setGlobalHandler`）。

### 5.4 直接操作原生视图：`setNativeProps`（已废弃）

- 挂在**原生组件 ref** 上的方法，直接改写底层原生视图属性，**绕过 render / diff / React 树**：

```jsx
const ref = useRef(null);
<View ref={ref} style={{ opacity: 1 }} />
ref.current.setNativeProps({ style: { opacity: 0.5 } });   // 不触发任何 React 流程
```

- 老架构下它是**性能逃生舱**：UI 更新要走 `setState → render → diff → Bridge 异步下发`，每帧调用太慢；`setNativeProps` 把中间全部砍掉，所以快到可以每帧执行。
- 与 `setState` 的关键差异：

| 维度 | `setState` | `setNativeProps` |
| --- | --- | --- |
| 更新路径 | render → diff → 挂载 → 原生 | 直达原生 |
| React 树 | 同步更新，是唯一事实来源 | **不更新，树与视图脱节** |
| 持久性 | 下一次 render 会重新下发，稳定 | **会被下一次 render 覆盖** |
| 时机 | 异步批处理 | 同步、即时生效 |

- 两个隐蔽语义：**prop 会"粘住"**（原生视图缓存上一次收到的 prop，只传部分 prop 时其余保留）；**`style` 是 merge 而不是替换**。合起来意味着原生视图里存在 **render 函数从未声明的隐式状态**——这正是排查困难、也是 Fabric 要修掉的问题。
- **新架构不再支持**：官方迁移文档明确写着 "setNativeProps will not be supported in the post-Fabric world"。Fabric 的影子树不可变、由 React 完全持有，与"绕过 React 直接改视图"从根本上是冲突的。由于 0.82 起新架构是唯一架构，新代码不应再考虑它。
- 替代方案：

| 原来的用法 | 现在的做法 |
| --- | --- |
| 改样式 / 无障碍属性 | 提升为 **state**，由 render 声明 |
| 每帧高频动画 | `Animated` + **`useNativeDriver: true`**（限 transform / opacity） |
| 复杂手势、布局类属性动画 | **Reanimated**（worklet 跑 UI 线程） |
| `<TextInput>` 直接改 text | 受控 `value`，或 `ref.current.clear()` |
| 命令式操作原生视图 | `codegenNativeCommands` |
| `findNodeHandle` 测量 | 直接在原生 ref 上调 `.measure()` |

## 六、列表与性能优化

### 6.1 列表选型

| 组件 | 机制 | 适用 |
| --- | --- | --- |
| `ScrollView` | 全量渲染 | 少量静态内容（表单、详情页） |
| `FlatList` | VirtualizedList，按窗口懒渲染 + 回收 | 通用长列表（**首选**） |
| `SectionList` | 分组虚拟列表 | 通讯录、分类列表 |
| `FlashList`（Shopify） | 单元**回收复用**（recycle） | 超长列表、复杂 cell，性能更好 |

### 6.2 FlatList 优化清单

| 手段 | 作用 |
| --- | --- |
| `keyExtractor` | 稳定 key，避免重复渲染与状态错位 |
| `getItemLayout` | 固定行高时**跳过测量**，滚动与定位更快 |
| `initialNumToRender` | 首屏渲染数量，影响首屏速度 |
| `maxToRenderPerBatch` / `updateCellsBatchingPeriod` | 每批渲染量与批次间隔 |
| `windowSize` | 渲染窗口（可视区上下各若干屏） |
| `removeClippedSubviews` | 移除屏幕外视图（Android 收益明显，注意副作用） |
| `React.memo` + `useCallback` | 避免 cell 因父组件重渲染而重渲染 |
| **避免内联函数/内联对象** | `renderItem={({item}) => ...}` 每次新建，破坏 memo |
| 图片固定宽高 + 缓存 | 减少布局抖动与网络请求 |
| `onEndReachedThreshold` | 无限滚动触底阈值调优 |

- **列表卡顿的典型根因**：cell 太重（嵌套 FlatList、大图无尺寸）、key 不稳定导致整列重建、`renderItem` 频繁新建、每次滚动都 setState。

## 七、状态管理与导航

### 7.1 状态管理选型

| 方案 | 特点 | 场景 |
| --- | --- | --- |
| `useState` / `useReducer` | 组件内状态 | 局部 UI 状态 |
| **Context** | React 内置，跨层传递 | 主题、语言、登录态等**低频**变化 |
| **Zustand / Jotai / MobX** | 轻量、按需订阅 | 中小型应用（Zustand 目前最流行） |
| **Redux Toolkit** | 规范化、可预测、中间件生态成熟 | 大型应用、复杂异步与调试需求 |
| **React Query / SWR** | 服务端状态（缓存、重试、失效、乐观更新） | **把"服务端数据"从全局状态里剥离** |
| MMKV / AsyncStorage | 本地持久化 | 缓存与偏好设置（MMKV 性能更好） |

- 实践建议：**服务端状态交给 React Query，客户端状态交给 Zustand/Redux**，别把两者混在一个 store 里。

### 7.2 导航

- **React Navigation**：纯 JS 实现，生态最全，含 `Stack / Tab / Drawer`，配 `react-native-screens`（原生屏幕优化）与 `react-native-safe-area-context`。
  - 深层链接（deep link）通过 `linking` 配置映射到路由。
  - 原生栈 `@react-navigation/native-stack` 基于原生导航容器，转场更接近原生。
- **react-native-navigation（Wix）**：原生导航实现，性能好但集成成本高。
- 页面间传参：路由 params（**只传可序列化数据**，别传函数/大对象），函数用 Context/事件解耦。

## 八、原生互操作（RN ↔ 原生）

### 8.1 四类通信方式

| 方式 | 方向 | 说明 |
| --- | --- | --- |
| **Native Module**（TurboModule） | JS → 原生 | JS 调用原生能力（相机、蓝牙、SDK） |
| **Native UI Component**（Fabric Component） | 原生 → JS | 把原生视图暴露成 JSX 组件（地图、播放器） |
| **Events** | 原生 → JS | `NativeEventEmitter` / `DeviceEventEmitter` 上报事件与数据 |
| **JSI HostObject** | 双向，**同步** | 新架构能力：JS 直接持有 C++ 对象引用并同步调用 |

### 8.2 TurboModule 怎么写（spec + 三端实现）

**① spec —— 唯一真相源**（文件名**必须以 `Native` 开头**，否则 Codegen 会静默忽略）

```ts
// specs/NativeCalculator.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  add(a: number, b: number): number;                    // 同步：不返回 Promise
  multiply(a: number, b: number): Promise<number>;      // 异步：Promise
  divide(                                               // 回调：可多次返回（流式）
    a: number, b: number,
    onSuccess: (result: number) => void,
    onError: (error: string) => void,
  ): void;
  getConstants(): { PI: number; E: number };            // 常量：可同步读取
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeCalculator');
```

方法形态与端上签名的对应关系：

| spec 写法 | 语义 | Android (Kotlin) | iOS (ObjC++) |
| --- | --- | --- | --- |
| `(a: number) => number` | **同步** | `fun add(a: Double): Double` | `- (NSNumber *)add:(double)a` |
| `(a: number) => Promise<number>` | 异步 | 末尾多一个 `promise: Promise` 参数 | `resolve:` + `reject:` 两个 block |
| `(cb: (r) => void) => void` | 回调 | 末尾多一个 `cb: Callback` 参数 | `RCTResponseSenderBlock` |
| `getConstants(): {...}` | 常量 | `getTypedExportedConstants()` | `- (NSDictionary *)getConstants` |

**② package.json 声明**

```json
"codegenConfig": {
  "name": "AppSpecs",
  "type": "modules",
  "jsSrcsDir": "specs",
  "android": { "javaPackageName": "com.yourapp.specs" }
}
```

Codegen 产物（**不要手改**）：Android 在 `android/build/generated/source/codegen/`（Java/Kotlin 抽象类 + JNI 胶水），iOS 在 `ios/build/generated/ios/`（`XxxSpec.h` 协议 + `XxxSpecJSI`）。

**③ Android 实现**

```kotlin
// android/app/src/main/java/com/yourapp/CalculatorModule.kt
class CalculatorModule(reactContext: ReactApplicationContext) :
    NativeCalculatorSpec(reactContext) {                     // 继承 Codegen 生成的抽象类

  override fun getName() = NAME
  override fun add(a: Double, b: Double): Double = a + b     // 同步
  override fun multiply(a: Double, b: Double, promise: Promise) = promise.resolve(a * b)
  override fun divide(a: Double, b: Double, onSuccess: Callback, onError: Callback) {
    onSuccess.invoke(a / b)
  }
  override fun getTypedExportedConstants(): Map<String, Any> = mapOf("PI" to Math.PI)

  companion object { const val NAME = "NativeCalculator" }
}
```

```kotlin
// CalculatorPackage.kt —— 优先用 BaseReactPackage（懒加载，启动更快）
class CalculatorPackage : BaseReactPackage() {
  override fun getModule(name: String, reactContext: ReactApplicationContext): NativeModule? =
      if (name == CalculatorModule.NAME) CalculatorModule(reactContext) else null

  override fun getReactModuleInfoProvider() = ReactModuleInfoProvider {
    mapOf(CalculatorModule.NAME to ReactModuleInfo(
      _name = CalculatorModule.NAME,
      _className = CalculatorModule.NAME,
      _canOverrideExistingModule = false,
      _needsEagerInit = false,          // 懒加载：用到才初始化
      isCxxModule = false,
      isTurboModule = true,
    ))
  }
}
```

注册：在 `MainApplication.kt` 的 `getPackages()` 中 `add(CalculatorPackage())`。

**④ iOS 实现**（**模块入口必须是 ObjC/ObjC++**（`.mm`）；Swift 只能被它调用，不能直接作为模块入口）

```objc
// RCTCalculator.mm
#import "RCTCalculator.h"
@implementation RCTCalculator

RCT_EXPORT_MODULE(NativeCalculator)          // 名字必须与 getEnforcing 一致

- (NSNumber *)add:(double)a b:(double)b { return @(a + b); }   // 同步

- (void)multiply:(double)a b:(double)b
         resolve:(RCTPromiseResolveBlock)resolve
          reject:(RCTPromiseRejectBlock)reject { resolve(@(a * b)); }

- (NSDictionary *)getConstants { return @{@"PI": @(M_PI)}; }

- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params {
  return std::make_shared<facebook::react::NativeCalculatorSpecJSI>(params);
}
@end
```

**⑤ JS 调用**

```ts
import NativeCalculator from './specs/NativeCalculator';

const sum = NativeCalculator.add(5, 3);                  // 同步，直接拿到返回值
const product = await NativeCalculator.multiply(5, 3);   // 异步
```

**⚠️ 三处名字必须完全一致** —— 这是"模块找不到"最常见的原因：

| 位置 | 值 |
| --- | --- |
| spec：`getEnforcing('NativeCalculator')` | `NativeCalculator` |
| Android：`NAME` | `NativeCalculator` |
| iOS：`RCT_EXPORT_MODULE(...)` | `NativeCalculator` |

不一致时 `getEnforcing` 会**直接抛错**（用 `get` 则拿到 `null`，报错更晚、更难查）。

**其他常见坑**

- **改了 spec 名字后 iOS 要清缓存**：`rm -rf ios/build`，否则旧产物残留导致签名对不上。
- **同步方法会阻塞 JS 线程**：不要在同步方法里做重活。
- **`Object` / `Array` 跨边界是递归转换**，大数组可能比自己算还慢（见 [8.4](#84-落地场景把-cpu-密集计算交给原生)）。
- **无平台依赖的算法优先写成纯 C++ 模块**，一份代码双端通用（见 [8.4](#84-落地场景把-cpu-密集计算交给原生)）。
- 老写法（新旧兼容通过 **interop 兼容层**）：iOS `RCT_EXPORT_MODULE` + `RCT_EXPORT_METHOD`，Android `ReactContextBaseJavaModule` + `@ReactMethod`，JS 侧 `NativeModules.XXX`。**常量与同步方法**在旧桥下最易出问题，迁移时重点回归。

### 8.3 原生视图组件（Fabric Component）

```ts
import { codegenNativeComponent } from 'react-native';
export default codegenNativeComponent<Props>('MyNativeView');
```

- 需要原生侧提供 **Component Descriptor**（iOS）/ **ViewManager + Delegate**（Android，新架构用 `Fabric ViewManager`）。
- 旧 `RCTViewManager` 在 Fabric 下**默认不生效**，需要 interop 层（只是迁移过渡，不要长期依赖）。

### 8.4 落地场景：把 CPU 密集计算交给原生

#### 先分清三条出路

| 出路 | 本质 | 适用 |
| --- | --- | --- |
| **TurboModule（纯 C++）** | 真的换执行体 | 无平台依赖的算法：加解密、压缩、图像/音频处理、解析 |
| **TurboModule（平台实现）** | 真的换执行体 | 需要平台 API，或要链接原生库（OpenCV / ONNX / FFmpeg / libsodium） |
| Reanimated worklet | **仍是 JS，只是换线程** | 每帧都要跑、必须不阻塞 JS 线程，但计算量本身不大 |

> ⚠️ 常见误解：**worklet 不会让单次计算变快**。Hermes 单线程才是 CPU 密集任务的硬天花板，C++ 的收益主要来自**能用多核 + 能链接成熟库**，而不是"编译得更优"。

#### 动手：纯 C++ TurboModule 五步

**① 写 spec**（**文件名必须以 `Native` 开头**，否则 Codegen 直接忽略）

```ts
// specs/NativeFastCodec.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  readonly encodeAll: (chunks: string[]) => string;   // 不返回 Promise = 同步调用
}
export default TurboModuleRegistry.getEnforcing<Spec>('FastCodec');
```

**② package.json 声明**

```json
"codegenConfig": {
  "name": "AppSpecs",
  "type": "modules",
  "jsSrcsDir": "specs",
  "android": { "javaPackageName": "com.sampleapp.specs" }
}
```

**③ 实现**（放 `shared/` 目录，与 `android`/`ios` 同级，**双端共用一份代码**）

```cpp
// shared/NativeFastCodec.h
#pragma once
#include <ReactCommon/TurboModule.h>
#include "AppSpecs/NativeFastCodec.h"        // Codegen 生成，路径由 codegenConfig.name 决定

namespace facebook::react {
class NativeFastCodec : public NativeFastCodecCxxSpec {
 public:
  explicit NativeFastCodec(std::shared_ptr<CallInvoker> jsInvoker);
  std::string encodeAll(jsi::Runtime& rt, std::vector<std::string> chunks);
};
} // namespace facebook::react
```

```cpp
// shared/NativeFastCodec.cpp
#include "NativeFastCodec.h"

namespace facebook::react {
NativeFastCodec::NativeFastCodec(std::shared_ptr<CallInvoker> jsInvoker)
    : NativeFastCodecCxxSpec(std::move(jsInvoker)) {}

std::string NativeFastCodec::encodeAll(jsi::Runtime& rt, std::vector<std::string> chunks) {
  // 这里就是纯 C++：可以开线程池、写 SIMD、链接第三方库
  return "encoded";
}
} // namespace facebook::react
```

**④ 注册**：Android 在 `MainApplication.getPackages()` 中加 Codegen 生成的 `<name>Package`；iOS 在 AppDelegate 里通过 `TurboModuleProvider` 注册 factory。

**⑤ JS 直接用**：`import FastCodec from './specs/NativeFastCodec'; FastCodec.encodeAll(chunks)`。

- Codegen 会把**同步方法**生成成直接返回 `jsi::Value` 的实现，跨边界没有 Promise 排队（这也是上一节"TurboModule 是 `jsi::HostObject`"的具体体现）。

#### 真正容易翻车的地方：跨边界的数据开销

新架构下**一次 JSI 调用本身很便宜**（亚毫秒级），但**参数的转换不一定便宜**：

| 类型 | 跨边界行为 |
| --- | --- |
| `number` / `boolean` | 值拷贝，最便宜 |
| `string` | 拷贝（大字符串有成本） |
| `Object` / `Array` | **递归转换**：每个字段都要转成 `jsi::Value`，10 万条记录**光转换就可能比计算本身还慢** |

由此推出的实践原则：

- **别传大数组 / 大对象**。传**句柄或 ID**，让 C++ 侧自己持有数据（先 `loadDataset(id)`，之后只传 `id`）。
- **减少调用次数比减小单次体积更重要**：1000 次小调用合并成 1 次批量调用。
- **二进制数据用 `ArrayBuffer`**：官方 Codegen 类型表（string/boolean/Object/Array/Function/Promise/Type Union/Callback/number）**未列入它**，属较新加入的能力，且已在双端做零拷贝优化；若版本较旧或需要更细的内存控制，可退回**手写 JSI 绑定**（实现 `RCTTurboModuleWithJSIBindings`），直接操作 `jsi::ArrayBuffer` 的底层内存。
- **不要用 JSON 字符串传结构**：序列化 + 反序列化是双倍开销，且丢掉类型安全。

#### 什么时候**不该**交给原生

- 计算量本来就小（几毫秒内）：跨边界与维护成本不划算。
- 每帧调用的高频小计算：TurboModule 的调度开销可能反而更慢，优先 worklet。
- **先量再优化**：用 React Native DevTools 的 Performance 面板确认瓶颈确实在 JS 执行时间，再决定搬 —— Hermes 没有 JIT 不等于它慢。

## 九、动画

| 方案 | 机制 | 适用 |
| --- | --- | --- |
| `Animated`（RN 内置） | `Animated.Value` + `timing/spring/decay` | 常规动画 |
| `Animated` + `useNativeDriver: true` | 动画计算**下发到原生 UI 线程**，绕过 JS 线程 | 只支持 `transform` / `opacity` 等**非布局属性**，性能最好 |
| `LayoutAnimation` | 一次性布局变化动画 | 列表增删、展开收起 |
| **Reanimated 3+** | worklet 直接跑在 **UI 线程**，可与手势同步 | 复杂手势、跟手动画、布局动画（现代首选） |
| `react-native-gesture-handler` | 原生手势系统 | 与 Reanimated 搭配 |
| `moti` / `react-native-animatable` | 上层封装 | 快速实现 |

- **`useNativeDriver` 的边界**：不能驱动 `width/height/top/left` 等布局属性（会在提交前就抛错）。0.85.1 起引入共享动画后端，布局属性也能走原生驱动（仍属实验特性）。
- 动画交互中避免频繁 `setState`（每帧触发 React 渲染），优先使用动画值直接驱动 `style`。

## 十、工程化、调试与部署

### 10.1 Metro（打包器）

- RN 专用打包器，特点：**单 bundle 全量打包**、按需构建、Babel transform、HMR（热替换）。
- 关键配置（`metro.config.js`）：
  - `inlineRequires`（把 import 变为**惰性 require**，明显改善启动耗时）。
  - `resolver` / `transformer` 定制、`watchFolders`（monorepo）。
- 产物：`index.android.bundle`（Android）/ `main.jsbundle`（iOS）。

### 10.2 Bundle 加载与 Hermes

- **开发环境**：从 Metro Dev Server（`localhost:8081`）拉 bundle，配合 Fast Refresh。
- **生产环境**：bundle 打进 App 资源，运行时解压到应用私有目录，由 Hermes 加载。
- **Hermes**：Meta 为 RN 定制的 JS 引擎，**构建期 AOT 编译为字节码**，省去运行时 parse/compile：
  - 启动更快、内存占用更低、包更小。
  - **不带 JIT**（CPU 密集的纯 JS 计算不如 JSC/V8）—— 但比"没 JIT"更根本的限制是**它单线程**，交给原生真正的收益在**多核与原生库**，见 [8.4](#84-落地场景把-cpu-密集计算交给原生)。
  - 0.70 起 Android 默认，iOS 逐步默认；**0.84 起 Hermes V1 成为默认引擎**（字节码与集成面兼容，无需改代码）。

### 10.3 调试工具

| 工具 | 用途 |
| --- | --- |
| **React Native DevTools** | 官方推荐（基于 CDP，替代已归档的 Flipper），支持断点、Console、React 组件树检查 |
| React DevTools | 检查组件树、props、状态、Profiler |
| Perf Monitor / Performance 面板 | 帧率、JS 线程耗时、内存 |
| Systrace / Perfetto（Android） | 原生侧线程与渲染耗时 |
| Xcode Instruments（iOS） | 卡顿、内存、能耗 |
| `console.log` / LogBox | 快速排查（LogBox 会聚合同类告警） |
| Detox / Maestro | E2E 自动化测试 |
| Jest + RN Testing Library | 单元与组件测试 |

### 10.4 打包发布与热更新

- **autolinking**（0.60+）：原生依赖自动链接，无需手动 `react-native link`。
- Android：Gradle 产物 AAB/APK，注意 Hermes 字节码编译任务（`hermesc`）与新架构的 CMake 编译时间（0.84 起 iOS 用预编译 `.xcframework`，编译时间大幅缩短）。
- iOS：CocoaPods 安装与管理。
- **Expo vs RN CLI**：
  - Expo（托管）：开箱即用、EAS Build 云构建、更新方便，原生定制通过 **Config Plugin / prebuild** 落地。
  - RN CLI（裸工程）：完全掌控 Android/iOS 工程，适合深度定制。
- **热更新**：本质是**下发新的 JS Bundle**（CodePush / EAS Update / 自建服务）→ 应用启动或指定时机加载新 bundle。
  - 限制：**只能更新 JS 与资源，不能改原生代码**；iOS 审核对"动态下发改变主要功能"有约束，需遵守平台政策。
  - 稳健做法：灰度、版本回滚、签名校验、与原生版本（`versionName`/`buildNumber`）绑定。

## 十一、性能与启动优化

| 方向 | 手段 |
| --- | --- |
| JS 线程 | 减少重渲染（memo / useCallback / 拆组件）、避免大 JSON 解析、把重计算放原生或 worklet、`InteractionManager` 延后重活 |
| 渲染 / 布局 | 减少 View 嵌套层级、避免频繁改变布局属性、动画走 `useNativeDriver` |
| 列表 | 见第六节；超长列表换 FlashList |
| 资源 | 图片按尺寸加载 + 缓存（FastImage / expo-image）、字体子集化、WebP |
| 启动（冷启动） | `inlineRequires`、Hermes 字节码、**TurboModule 懒加载**（0.76+ 默认收益）、延迟初始化、减少启动期同步 IO、Splash + 骨架屏 |
| 包体积 | R8/ProGuard（Android）、资源裁剪、`abiFilters` 裁剪 so、按需引入 SDK、升级到新架构（可移除大量旧桥代码） |
| 内存 | 清理 `useEffect` 订阅与定时器、大图解码尺寸控制、避免缓存无上限、关注原生侧内存（图片/Video） |

- **白屏排查顺序**：① 日志里 bundle 是否加载成功 → ② JS 是否在首屏前抛错（可加全局错误兜底）→ ③ 是否被大量同步初始化阻塞 → ④ 首屏是否依赖网络/字体/图片。
- **卡顿定位**：先看 Perf Monitor 的 **JS 线程 vs UI 线程**耗时，谁高查谁；JS 高 → 重渲染/重计算；UI 高 → 视图层级、阴影、大图、原生绘制。

## 十二、常见坑与排查

| 现象 | 常见原因 |
| --- | --- |
| 启动白屏 | bundle 加载失败 / JS 早期异常 / 同步初始化过多 |
| 列表空白但可滚动 | 未设高度（`flex: 1` 缺失）或 `getItemLayout` 行高与实际不符 |
| 文本不显示 | 文本没包在 `<Text>` 里；或 `View` 缺少 `flex` 约束导致高度为 0 |
| `flex: 1` 不生效 | 父容器无确定高度（Android 需 `flex: 1` 链路上溯到根） |
| Android 阴影无效 | Android 用 `elevation`，iOS 才用 `shadow*` 系列 |
| 键盘遮挡输入框 | `KeyboardAvoidingView` / `react-native-keyboard-controller` |
| 状态更新不触发渲染 | 直接修改对象/数组（未返回新引用）；闭包捕获旧值（用函数式 `setState`） |
| 内存泄漏 / 警告 | `useEffect` 未清理订阅、定时器、监听器；组件卸载后 setState |
| 新架构下第三方库报错 | 该库未适配（旧 `RCTViewManager` / `UIManager.dispatchViewManagerCommand`）→ 查库的 issue 与适配版本 |
| 升级后编译失败 | 新旧架构混用 / 依赖锁版本不一致 → 先升到 0.81 开新架构跑通再往上 |

## 十三、推荐资料

- [React Native 官方文档](https://reactnative.dev/docs/getting-started)
- [新架构介绍（About the New Architecture）](https://reactnative.dev/architecture/landing-page)
- [React Native Blog（版本发布说明，了解演进最可靠）](https://reactnative.dev/blog)
- [React Native Directory（库的新架构适配状态）](https://reactnative.directory/)
- [React Navigation 文档](https://reactnavigation.org/)
- [Hermes 引擎仓库](https://github.com/facebook/hermes)
- 《React Native 跨平台移动应用开发》系列资料 / Expo 官方文档
