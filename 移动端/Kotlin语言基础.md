# Kotlin 语言基础

> Kotlin 是 JetBrains 开发的现代 JVM 语言，2019 年起成为 Android 官方首选语言。它"兼容 Java、优于 Java"，既可用于服务端，也是 Android 客户端开发的事实标准。

## 一、Kotlin 是什么

- **开发者**：JetBrains（IDEA/IntelliJ 的母公司），2011 年启动，2016 年发布 1.0。
- **定位**：运行在 JVM 上的静态类型语言，语法现代、简洁、安全。
- **重要节点**：
  - 2017 年 Google 宣布 **Android 官方支持 Kotlin**。
  - 2019 年 Google 宣布 **Kotlin-first**：新 API 优先提供 Kotlin 版本。
  - 2023 年 Google I/O 表示 **65%+ 的 Android 新项目已是 Kotlin 为主**。
- **多平台**：一套语言可编译到 JVM、Android（DEX）、JavaScript、Native（iOS/嵌入式/Wasm）。

## 二、Kotlin 与 JVM 的关系

### 2.1 运行原理

Kotlin 编译器（`kotlinc`）把 `.kt` 源码编译成 **JVM 字节码**（`.class`），与 Java 字节码格式完全一致，因此：

```
.kt 源码 ──kotlinc──▶ .class 字节码 ──▶ JVM 运行（服务端）
                                └─────▶ dx/d8 ──▶ .dex ──▶ Android ART 运行
```

- 服务端场景：Kotlin 就是"另一种写 Java 字节码的语言"，依赖同样打进 jar。
- Android 场景：`.class` 再经 d8/dex 编译为 DEX，运行在 ART（Android Runtime）上，同样复用 JVM 系生态。

### 2.2 与 Java 完全互操作

- Kotlin **可以调用任何 Java 库**：Spring、Guava、Retrofit 直接用。
- Java **也可以调用 Kotlin**：Kotlin 编译出的类对 Java 是"普通类"，只是部分语法糖需要注解辅助（见面试题互操作部分）。
- 同一个工程允许 **Kotlin 与 Java 混编**（Android Studio / Gradle 原生支持），便于老项目渐进迁移。

### 2.3 三种运行形态

| 形态 | 编译目标 | 典型场景 |
| ---- | -------- | -------- |
| Kotlin/JVM | JVM 字节码 | Android App、后端服务（Spring Boot 亦支持 Kotlin） |
| Kotlin/JS | JavaScript | 前端、浏览器 |
| Kotlin/Native | 平台二进制 | iOS、macOS、嵌入式（底层仍是 LLVM） |

- **KMP（Kotlin Multiplatform）**：一份业务代码共享到 Android + iOS + 后端，UI 各自实现，是当前跨端热门方向。

## 三、Kotlin 与 Java 的关系（对比）

> Kotlin 不是 Java 的替代品，而是"更现代、更安全、更简洁的 JVM 语言"，两者生态互通。

### 3.1 核心差异总览

| 维度 | Java | Kotlin |
| ---- | ---- | ------ |
| 类型推导 | `String s = "x"` | `val s = "x"`（可省略类型） |
| 可变性 | `final` 默认不可变，其余默认可变 | **`val` 不可变 / `var` 可变，默认推荐 val** |
| 空安全 | 默认允许空，全靠运行时判断 | **编译期区分可空 `?` 与不可空**，默认不可空 |
| 数据类 | 手写 getter/setter/equals/hashCode/toString | **`data class` 一行全生成** |
| 属性访问 | `getX()/setX()` 方法 | **属性语法** `obj.x`，背后仍是 getter/setter |
| 静态成员 | `static` 关键字 | **伴生对象 companion object**（无 static） |
| 构造函数 | 构造器 + 重载 | 主/次构造 + **默认参数 + 命名参数** |
| 字符串 | `"a" + x` | **字符串模板** `"a $x"` |
| 集合 | 可变 API 为主 | **不可变/可变分离**（`listOf` vs `mutableListOf`） |
| 函数 | 方法必须属类；无函数类型 | **函数是一等公民**，lambda、函数类型、顶层函数 |
| 扩展能力 | 无 | **扩展函数/扩展属性**（给已有类加方法） |
| 智能类型转换 | `instanceof` + 手动强转 | `is` 检查后**自动转换** |
| when/switch | `switch` 只支持少数类型 | **`when` 表达式**，强大且可作为表达式 |
| 受检异常 | 强制处理 | **无受检异常**（避免无意义 try-catch） |
| 协程 | 线程为主（CompletableFuture 等） | **协程原生支持**（轻量、结构化并发） |
| 空指针 | 运行期 NPE | 编译期消灭大部分 NPE |
| 原始类型 | int/long 等 | 统一对象，编译期自动优化为原始类型 |
| 继承 | 默认可继承 | **默认 final**，需显式 `open` |
| 单例 | 手写 | **`object` 声明即单例** |

### 3.2 Java 8 特性在 Kotlin 中的体现

Java 近年引入的 lambda、Optional、stream 等，Kotlin 在 2011 年设计时就内置且更自然：

- Java lambda → Kotlin lambda（可作为参数直接传，无 SAM 样板限制）。
- `Optional<T>` → Kotlin **可空类型**（`T?`）+ 安全调用 `?.`，语言级而非容器级。
- Stream API → Kotlin 集合原生链式操作 `filter/map`（无需 `.stream()`）。
- `record`（Java 16+）→ Kotlin `data class`。

### 3.3 Kotlin 相对 Java 的"代价"

- 编译速度略慢于 Java（KAPT/KSP 注解处理）。
- 字节码体积略大、方法数略多（Android 中注意 multidex / R8）。
- 过度使用语法糖会降低可读性（如疯狂链式调用）。
- 需要学习"Kotlin 惯用法"才能真正发挥优势，新手容易写"用 Kotlin 语法的 Java"。

## 四、Kotlin 在 Android 中的使用

### 4.1 为什么 Android 用 Kotlin

1. **官方 Kotlin-first**：Android 文档与示例全面转向 Kotlin，新 Jetpack API 先出 Kotlin 版。
2. **空安全**：Android 组件生命周期复杂（`Bundle` 取值、findViewById 后置空），Kotlin 可空类型从源头减少 NPE。
3. **协程**：天然契合 Android 异步需求——网络请求、IO、主线程切换比回调/线程干净得多。
4. **Jetpack Compose**：现代声明式 UI 完全基于 Kotlin（`@Composable`），是当前新项目首选 UI 方案。
5. **与 Java 混编**：老项目可渐进迁移，无一次性重写风险。

### 4.2 Android 开发中的典型用法

**（1）声明式 UI：Jetpack Compose**

```kotlin
@Composable
fun Greeting(name: String) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text(text = "Hello, $name!", style = MaterialTheme.typography.headlineMedium)
        Button(onClick = { /* 状态驱动 UI */ }) { Text("点击") }
    }
}
```

**（2）协程 + 网络请求（Retrofit + suspend）**

```kotlin
interface ApiService {
    @GET("user/{id}")
    suspend fun getUser(@Path("id") id: Long): User   // 挂起函数，无回调
}

// ViewModel 中：
viewModelScope.launch {
    val user = api.getUser(1L)            // 挂起，不阻塞主线程
    _uiState.value = UiState.Success(user) // 自动回主线程更新
}
```

**（3）空安全处理可空 UI 状态**

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding   // 延迟初始化（有保证非空）
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        val name: String? = intent.getStringExtra("name")   // 可能为空
        binding.tvName.text = name ?: "未命名"              // 空合并兜底
    }
}
```

**（4）扩展函数简化 UI 操作**

```kotlin
fun Context.showToast(msg: String) =
    Toast.makeText(this, msg, Toast.LENGTH_SHORT).show()

// 任意 Activity 中：showToast("保存成功")
fun View.visible() { visibility = View.VISIBLE }
fun View.gone()    { visibility = View.GONE }
```

**（5）sealed class 建模 UI 状态**

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<Item>) : UiState()
    data class Error(val msg: String) : UiState()
}
// when 分支穷尽，编译器强制覆盖所有情况
```

### 4.3 Android 生态与 Kotlin 结合的关键库

- **Jetpack Compose**：声明式 UI 框架（官方推荐替代 XML+View）。
- **Coroutines + Flow**：异步与响应式数据流（替代 RxJava 的新选择）。
- **ViewModel + LiveData/StateFlow**：生命周期感知的状态管理。
- **Android KTX**：官方为 Kotlin 提供的扩展库（如 `lifecycleScope`、`viewModels()` 委托）。
- **Room/Retrofit/OkHttp**：Java 库但官方均提供 Kotlin 协程友好 API。

## 五、Kotlin 核心语法速览（面试高频）

### 5.1 val / var 与类型

```kotlin
val a = 1          // 只读引用（类似 final），初始化后不可重新赋值
var b = "x"        // 可变
val c: Long = 1L   // 显式类型
```

> `val` 只是"引用不可变"，若指向可变对象（`val list = mutableListOf()`），对象内容仍可变。

### 5.2 空安全三兄弟

```kotlin
var name: String? = null   // 可空类型，结尾 ?
val len = name?.length      // 安全调用：为 null 则结果 null（不崩）
val l2  = name!!.length     // 非空断言：为 null 抛 NPE（慎用）
val l3  = name?.length ?: 0 // Elvis 空合并：null 时用兜底值
```

### 5.3 函数与默认/命名参数

```kotlin
fun greet(name: String, prefix: String = "Hello") = "$prefix, $name"
greet("Kotlin")                // 默认参数
greet("Kotlin", prefix = "Hi") // 命名参数，可读性好
```

### 5.4 扩展函数

```kotlin
fun String.isEmail(): Boolean = contains("@")
"abc@x.com".isEmail()   // 像成员方法一样调用
// 原理：编译成静态方法，第一个参数为接收者（见面试题）
```

### 5.5 when 表达式

```kotlin
fun describe(x: Any): String = when (x) {
    is String -> "字符串: $x"     // 智能转换，之后 x 自动是 String
    0, 1     -> "数字 0/1"
    in 2..10 -> "2~10 之间"
    else     -> "其他"
}
```

### 5.6 数据类与解构

```kotlin
data class User(val id: Long, val name: String)   // 自动生成 equals/hashCode/toString/copy/componentN
val (id, name) = User(1L, "张三")   // 解构声明
val u2 = user.copy(name = "李四")   // copy 修改部分字段
```

### 5.7 伴生对象（替代 static）

```kotlin
class Utils {
    companion object {
        const val TAG = "Utils"          // 编译期常量
        @JvmStatic fun create() = Utils() // Java 侧可直接 Utils.create() 调用
    }
}
```

### 5.8 lateinit 与 lazy

```kotlin
lateinit var adapter: MyAdapter      // 延迟初始化，用于"构造后赋值且非空"
val config: Config by lazy { Config.load() }  // 首次访问才初始化（线程安全）
```

### 5.9 协程最小示例

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch { delay(100); println("World") }   // 启动协程，非阻塞
    println("Hello")
}
// 输出 Hello → World
```

### 5.10 sealed class 与 object

```kotlin
sealed class Result             // 密封类：子类在同一个文件内穷尽，适合状态建模
object Loading : Result()       // object：单例对象
data class Success(val data: Any) : Result()
```

## 六、学习路线小结

1. 先掌握基础：`val/var`、空安全、函数、`when`、集合操作。
2. 再学面向对象 Kotlin 化：data class、扩展、object、sealed、泛型 in/out。
3. 理解与 Java/JVM 的关系：字节码、互操作注解、混编。
4. 重点攻克协程：suspend 原理、Dispatchers、作用域、Flow。
5. 落地 Android：Compose + ViewModel + Retrofit + Room 搭一个完整 Demo。
