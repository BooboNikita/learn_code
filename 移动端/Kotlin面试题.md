# Kotlin 面试题

> Kotlin 高频面试题整理：基础语法 → 空安全 → Java/JVM 关系与互操作 → 协程 → Android 使用 → 进阶原理。每题为「考察点 + 回答要点」。

## 一、基础语法

### 1.1 `val` 与 `var` 的区别？

- `val`：**只读引用**（final 变量），初始化后不能重新赋值；`var`：可变，可重新赋值。
- 注意：`val` 不保证"指向的对象不可变"——`val list = mutableListOf(...)` 仍可增删元素。
- 结论：能 `val` 就 `val`，不可变性让代码更安全、更易推理（与 Java 中"默认可变、需 final"相反）。

```kotlin
val name = "Kotlin"               // 只读引用
name = "Java"                     // 编译错误：Val cannot be reassigned
var count = 0                     // 可变引用
count++                           // OK
val list = mutableListOf(1, 2, 3)
list.add(4)                       // OK：val 保证的是"引用不可变"，对象内容仍可变
```

### 1.2 Kotlin 相比 Java 有哪些主要优点？

- **空安全**：编译期通过 `?` / `?.` / `?:` 消灭大部分 NPE。
- **简洁**：类型推导、data class、字符串模板、lambda，样板代码大幅减少。
- **不可变优先**：`val`、不可变集合（`listOf`）。
- **扩展函数**：无需继承即可给类加方法。
- **函数一等公民**：lambda、函数类型、顶层函数。
- **协程**：语言级支持轻量并发，避免回调地狱。
- **智能转换**：`is` 检查后自动转型。
- **无受检异常**、默认 `final` 类、`when` 表达式更强大。

### 1.3 Kotlin 的 `==` 与 `===` 区别？（对应 Java 的 equals / ==）

- `==`：**结构相等**，默认调用 `equals()`（与 Java `equals` 对应）；`data class` 自动实现。
- `===`：**引用相等**，即 Java 的 `==`（比较地址）。
- 面试延伸：`val a = 1000; val b = 1000; a == b` 为 true；基本类型包装在 `-128..127` 内 `===` 也为 true（缓存）。

```kotlin
data class User(val id: Int)
val u1 = User(1); val u2 = User(1)
u1 == u2     // true：equals 比较内容
u1 === u2    // false：两个不同对象，地址不同

val a = 1000; val b = 1000
a == b       // true
a === b      // false：超出 -128..127 不走缓存，各自装箱
```

### 1.4 `data class` 自动生成了什么？有何限制？

- 自动生成：`equals()`、`hashCode()`、`toString()`、`copy()`、`componentN()`（解构）。
- 限制/注意：
  - 主构造函数必须至少有一个参数，且参数都要标 `val`/`var`。
  - 生成的 `equals` 只比较**主构造函数里的属性**，类体中的属性不参与。
  - `copy` 是浅拷贝。
  - data class 不能是 open（默认 final），可加 `data` 到 sealed 子类。

```kotlin
data class Person(val name: String, val age: Int) {
    var nickname: String = ""      // 类体属性：不参与 equals/hashCode/copy
}
val p1 = Person("Tom", 18)
val p2 = p1.copy(age = 20)         // 浅拷贝并修改部分属性
val (name, age) = p1               // componentN 解构
p1 == Person("Tom", 18)            // true：只比较主构造函数里的属性
```

### 1.5 `object`、`companion object` 与 Java `static` 的关系？

- `object`：声明**单例对象**，Kotlin 没有 Java 的 `static`，用 object 表示"类级别的单例"。
- `companion object`（伴生对象）：挂在类上的 object，用于放置静态成员。反编译后就是类里的 `static INSTANCE` + 静态方法。
- Java 侧调用注意：伴生对象里默认方法编译为伴生类实例方法，Java 需 `Utils.Companion.create()`；加 **`@JvmStatic`** 后可直接 `Utils.create()`。
- 常量用 `const val`（编译期常量，内联进字节码）修饰在 companion 中。

```kotlin
class Utils {
    companion object {
        const val TAG = "Utils"
        @JvmStatic fun create() = Utils()
    }
}
// Java: Utils.Companion.create() 或 Utils.create()（加 @JvmStatic 后）
```

### 1.6 扩展函数是什么？原理？

- 在**类外部**给类"添加"方法，无需继承与修改原类：

```kotlin
fun String.isPhone(): Boolean = length == 11 && all { it.isDigit() }
"13800138000".isPhone()
```

- **原理**：编译成普通**静态函数**，接收者作为第一个隐藏参数：

```java
// 反编译示意
public static final boolean isPhone(String $this) { ... }
```

- 注意：
  - 扩展函数**不是成员函数**，是静态分发——同名同签名时，成员函数优先。
  - 不参与多态（调用取决于**静态类型**），也没有真正"覆盖"。
  - 扩展函数不能访问私有成员。

### 1.7 `lateinit` 与 `by lazy` 的区别？

| 维度 | lateinit var | by lazy {} |
| ---- | ------------ | ---------- |
| 类型 | 仅用于 `var` | 仅用于 `val` |
| 支持类型 | 非空对象（不能是基本类型/可空） | 任意类型 |
| 初始化时机 | 后续手动赋值 | **首次访问时**自动初始化 |
| 线程安全 | 不保证 | 默认线程安全（LazyThreadSafetyMode.SYNCHRONIZED） |
| 未初始化访问 | 抛 UninitializedPropertyAccessException | 正常执行初始化 |

- Android 典型用法：`lateinit` 用于依赖注入/onCreate 中初始化的 View、Adapter；`lazy` 用于属性按需懒加载。

```kotlin
class DetailActivity : AppCompatActivity() {
    lateinit var adapter: ItemAdapter          // 承诺"稍后 onCreate 里一定赋值"
    val config: Config by lazy { loadConfig() } // 首次访问 config 时才真正加载

    override fun onCreate(savedInstanceState: Bundle?) {
        adapter = ItemAdapter()                 // 此刻初始化 lateinit
    }
}
```

### 1.8 `sealed class`（密封类）是什么？为何常用于 Android 状态建模？

- 密封类：**子类受限**——所有直接子类必须与它同文件（Kotlin 1.5+ 放宽为同模块、同包）。
- 与 `when` 配合可**穷尽分支**，无需 else，漏分支编译报错。
- 常用于 UI 状态/网络结果建模：Loading / Success / Error 三类明确枚举。

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<Item>) : UiState()
    data class Error(val msg: String) : UiState()
}
fun render(state: UiState) = when (state) {   // 编译器要求覆盖全部子类
    UiState.Loading -> ...
    is UiState.Success -> ...
    is UiState.Error -> ...
}
```

### 1.9 智能转换（Smart Cast）是什么？

- 用 `is` 判断后，编译器在作用域内**自动把类型收窄**，无需手动强转：

```kotlin
fun f(x: Any) {
    if (x is String) println(x.length)   // x 自动转为 String，无需 (x as String)
}
```

- 失效场景：`var` 可能被并发修改的属性（跨线程）、自定义 getter 的属性无法智能转换，需手动 `as`。

### 1.10 Kotlin 泛型的 `in`/`out` 与 Java 通配符对应关系？

- `out T`（协变）：只能读不能写，等价 Java `? extends T`，如 `List<out Animal>`。
- `in T`（逆变）：只能写不能读，等价 Java `? super T`，如 `Comparable<in Dog>`。
- Kotlin 支持**声明处变型**（在类声明处标 in/out），Java 只能用**使用处通配符**。
- `*` 星投影 ≈ Java 的裸类型 `List<?>`。

```kotlin
open class Animal
class Dog : Animal()

// out 协变（生产者，只读）：Dog 列表可以当 Animal 列表用
fun printAnimals(animals: List<out Animal>) {
    val a: Animal = animals[0]          // 只能读出 Animal
    // animals.add(Dog())               // 编译报错：协变集合禁止写入
}
printAnimals(listOf(Dog()))             // List<Dog> 可传入

// in 逆变（消费者，只写）：Animal 列表可以当 Dog 列表用
fun fillDogs(dst: MutableList<in Dog>) {
    dst.add(Dog())                      // 只能写入 Dog
}
fillDogs(mutableListOf<Animal>())       // 父类型容器可传入
```

### 1.11 Kotlin 有哪些作用域函数？如何区分？

| 函数 | 上下文对象 | 返回值 | 典型用途 |
| ---- | ---------- | ------ | -------- |
| `let` | `it` | lambda 最后一行 | 可空对象处理 + 转换结果 |
| `run` | `this` | lambda 最后一行 | 对象配置 + 返回计算结果 |
| `apply` | `this` | **对象本身** | 链式配置属性（Android 中 view 配置最常用） |
| `also` | `it` | **对象本身** | 副作用操作（日志、计数） |
| `with` | `this` | lambda 最后一行 | 对同一对象多次操作（非扩展） |

```kotlin
val tv = TextView(this).apply {      // 配置完返回自身
    text = "Hello"
    textSize = 18f
}
val len = name?.let { it.length } ?: 0   // 可空安全处理
val info = with(person) { "$name-$age" } // this 指向 person，返回拼接结果
listOf(1, 2).also { println(it) }        // it 指向列表本身，打日志后原样返回
val size = run {                         // 独立作用域：临时变量不污染外层
    val tmp = listOf(1, 2, 3)
    tmp.size                             // 返回 3
}
```

## 二、空安全

### 2.1 Kotlin 如何实现空安全？

- **类型系统区分可空/不可空**：默认类型不可空，声明可空加 `?`（`String?`）。
- **编译期拦截**：可空值不能直接调用方法/赋值给不可空变量，否则编译报错。
- 配套操作符：
  - `?.` 安全调用：为 null 跳过并返回 null。
  - `?:` Elvis 空合并：null 时给默认值。
  - `!!` 非空断言：确信非空，但为 null 时抛 NPE（应谨慎使用）。
  - `lateinit`/`by lazy`：处理"延迟赋值但语义非空"的场景。

```kotlin
val a: String? = null
println(a?.length)      // null
println(a?.length ?: 0) // 0
println(a!!.length)     // 抛 NullPointerException
```

### 2.2 Kotlin 能完全避免 NPE 吗？

- 不能 100%，但把 NPE 从"默认发生"变为"显式发生"：
  - `!!` 使用不当仍会抛 NPE。
  - Java 代码传入的空值 Kotlin 侧类型系统拦不住（**平台类型 `T!`**，需自己注意）。
  - 泛型擦除、外部框架（反射、反序列化）可能绕过检查。
- 建议：新代码优先 `?.`/`?:`，`!!` 仅用于确实不可能是 null 且想快速失败的地方。

### 2.3 平台类型（Platform Type）是什么？

- Java 代码没有空注解时，Kotlin 不知道其可空性 → 类型显示为 `String!`（平台类型）。
- **按不可空使用**：运行期若 Java 返回 null 会崩；按可空使用则安全。
- 应对：优先在 Java 侧加 JetBrains 空注解（`@Nullable/@NonNull`），或 Kotlin 侧一律按可空处理。

```java
// Java：无任何空注解，返回类型在 Kotlin 眼里是 String!（平台类型）
public String getName() { return null; }    // 运行期可能真的返回 null
```

```kotlin
val name: String = javaUser.name    // 按非空使用 → 返回 null 时立即抛 NPE
val safe: String? = javaUser.name   // 按可空处理 → 安全，配合 ?. 使用
```

## 三、与 Java / JVM 的关系与互操作

### 3.1 Kotlin 与 JVM 是什么关系？它是如何运行的？

- Kotlin/JVM 编译产物是**标准 JVM 字节码**（`.class`），在 JVM 上运行，与 Java 共享运行时与生态。
- 编译器链路：`kotlinc` 将 `.kt` → 字节码；Android 上再由 d8 转 `.dex` 跑在 ART。
- 因此 Kotlin 不是解释型脚本，性能与 Java 同级（同一字节码，JIT 同样生效）。
- 三个平台：Kotlin/JVM、Kotlin/JS、Kotlin/Native（LLVM）。

### 3.2 Kotlin 和 Java 能互相调用吗？有哪些注意事项？

- 能，**双向互操作**，同一工程可混编。
- Kotlin 调 Java：直接调用（注意平台类型、Java 的 setter 会变属性语法、SAM 转换）。
- Java 调 Kotlin 的语法糖需要注解辅助：

| 注解 | 作用 |
| ---- | ---- |
| `@JvmStatic` | companion object 中方法变真正静态方法 |
| `@JvmField` | 暴露字段，不生成 getter/setter |
| `@JvmOverloads` | 默认参数生成多组 Java 重载方法 |
| `@JvmName` | 指定字节码方法/类名（顶层函数默认类名 `FileKt`） |
| `@Throws` | 声明抛出的受检异常，让 Java 侧能捕获 |
| `@JvmMultifileClass` | 多个文件顶层函数合并到一个类 |

```kotlin
class UserApi {
    @JvmField
    val cache: MutableMap<String, String> = HashMap()   // Java 直接 userApi.cache 字段访问

    @JvmOverloads
    fun load(id: String = "1", force: Boolean = false) {}
    // Java 侧自动得到三个重载：load() / load(String) / load(String, boolean)
}
```

### 3.3 Kotlin 顶层函数/属性在 Java 中如何访问？

- 顶层函数编译进以文件名命名的类：`utils.kt` 顶层函数 `fun create()` → Java 调 `UtilsKt.create()`。
- 用 `@file:JvmName("Utils")` 自定义类名；`@file:JvmMultifileClass` 合并多文件。

```kotlin
// utils.kt
fun create(): Utils = Utils()     // Java 调用：UtilsKt.create()

// 另一个文件（注解写在文件首行，作用于整个文件）
@file:JvmName("StringUtils")
package com.example
fun create2(): Utils = Utils()    // Java 调用：StringUtils.create2()
```

### 3.4 Kotlin 有静态成员吗？为什么用 companion object？

- 语言层面**没有 static**。设计考虑：静态成员无面向对象语义（无多态），而 companion object 本身是对象，可以继承、作为参数传递。
- companion object 反编译后仍是普通类 + `Companion` 静态字段（`INSTANCE`），只是加了一层。
- 需要真静态时用 `@JvmStatic`。

### 3.5 Kotlin 有受检异常吗？为什么？

- **没有**（Java 有 checked exception）。
- 理由：受检异常在实践中常被无意义 catch 吞掉、污染接口签名、降低 lambda/函数式代码流畅度；Kotlin 认为"异常不应该是类型系统的一部分"。
- Java 调 Kotlin 方法若需捕获，用 `@Throws(Exception::class)` 声明。

### 3.6 Kotlin 基本类型与 Java 的 int/long 有什么区别？

- Kotlin 没有原始类型，一切都是对象：`Int`、`Long`、`Double` 等。
- 但**编译期智能处理**：能优化成 JVM 原始类型 `int/long` 时自动优化，避免装箱开销（无法确定时退化为包装类型）。
- 集合中 `List<Int>` 底层仍是装箱的 `Integer`。

```kotlin
val a: Int = 42                       // 局部变量 → 编译为原始类型 int，无装箱
val b: Int? = a                       // 需要可空 → 必须装箱为 Integer
val list: List<Int> = listOf(1, 2)    // 集合元素全部装箱为 Integer
```

### 3.7 Kotlin 的 `Unit` 与 Java `void` 的区别？

- Java `void` 不是类型，方法声明为无返回。
- Kotlin `Unit` 是**真实类型**（单例 object），可作为泛型参数、可赋值给变量、可作函数返回值：`fun f(): Unit`、`Function<Unit>`。
- 与 `Nothing` 区分：`Nothing` 表示"永远不返回"（如 `throw`、`error()`、`TODO()`），是所有类型的子类型。

```kotlin
fun fail(msg: String): Nothing = throw IllegalArgumentException(msg)
val host: String = config.host ?: fail("host 不能为空")  // Nothing 可赋给任何类型，分支类型仍是 String
val u: Unit = println("hi")                              // Unit 是真实类型，可以赋值给变量
```

## 四、协程

### 4.1 协程是什么？与线程有什么区别？

- **协程（Coroutine）**：轻量级并发单元，跑在线程之上，通过**挂起/恢复**实现非阻塞。
- 对比：

| 维度 | 线程 | 协程 |
| ---- | ---- | ---- |
| 创建开销 | 大（内核级，约 1MB 栈） | 极小（用户态对象） |
| 数量级 | 千级 | 十万/百万级 |
| 切换 | 内核调度、有上下文切换开销 | 语言级挂起，无内核切换 |
| 阻塞 | `sleep` 阻塞线程 | `delay` 只挂起当前协程 |
| 并发模型 | 回调/锁易乱 | 顺序代码写异步（suspend） |

- 一句话：**协程不是"更快的线程"，而是"可挂起的计算"，靠线程池执行、靠状态机恢复**。

```kotlin
runBlocking {
    repeat(100_000) {
        launch { delay(1_000) }   // 十万个协程同时"等待"，底层只用少量线程
    }                             // 换成 Thread(100_000 个) 会直接 OOM
}
```

### 4.2 `suspend` 关键字的原理？

- `suspend` 标记**挂起函数**，只能在协程或其他 suspend 函数中被调用。
- 编译器将 suspend 函数编译为 **CPS（Continuation Passing Style）+ 状态机**：
  - 隐藏的 `Continuation` 参数（回调）记录"恢复点"。
  - 函数体被拆成多个状态，遇到 `delay`/网络 IO 等真正挂起点时返回，之后由调度器回调 continuation 恢复执行。
- 所以挂起**不阻塞线程**，线程空闲可执行其他协程。

```kotlin
// 源码写法
suspend fun fetchUser(id: Int): User = api.getUser(id)

// 编译器实际生成（示意）：多出隐藏的 Continuation 参数，返回值变为 Object（状态机）
fun fetchUser(id: Int, completion: Continuation<User>): Any? {
    // 内部按挂起点拆成多个分支状态，恢复时从上次的 label 继续执行
}
```

### 4.3 `Dispatchers` 有哪几种？

| 调度器 | 说明 | 用途 |
| ------ | ---- | ---- |
| `Dispatchers.Main` | Android 主线程（UI 线程） | 更新 UI、收集数据流 |
| `Dispatchers.IO` | IO 线程池（适合阻塞 IO） | 网络、磁盘、数据库 |
| `Dispatchers.Default` | CPU 密集型线程池（核数相关） | 计算、解析、排序 |
| `Dispatchers.Unconfined` | 不限定（一般不用） | 极少数场景 |

- `withContext(Dispatchers.IO)` 切换上下文且**结束后自动回到原上下文**。
- 注意：IO 与 Default 共享底层线程池，可复用线程。

```kotlin
viewModelScope.launch {                            // 1. Main 主线程启动
    val user = withContext(Dispatchers.IO) {       // 2. 切到 IO 做网络请求
        api.getUser(id)
    }
    val sorted = withContext(Dispatchers.Default) { // 3. 切到 Default 做 CPU 排序
        user.items.sortedByDescending { it.time }
    }
    binding.name.text = user.name                  // 4. 自动回到 Main，直接更新 UI
}
```

### 4.4 `launch` 与 `async` 的区别？

- `launch`：启动协程，返回 **`Job`**，无返回值（Unit），fire-and-forget。
- `async`：启动协程，返回 **`Deferred<T>`**（继承 Job），通过 `await()` 获取结果，用于"并发执行再汇总"。
- 抛异常语义不同：`launch` 直接抛；`async` 在 `await()` 时才抛（根 async 除外，Kotlin 1.7+ 调整）。
- 两者都需在 **CoroutineScope** 中启动（结构化并发）。

```kotlin
scope.launch { doSomething() }                       // 无返回
val d: Deferred<Int> = scope.async { compute() }     // 有返回
val r = d.await()                                    // 挂起等待结果
```

### 4.5 什么是结构化并发？为什么重要？

- **协程必须在 CoroutineScope 中启动**，子协程的生命周期绑定父作用域：
  - 父作用域取消 → 所有子协程自动取消。
  - 父协程会等待所有子协程完成。
  - 异常会沿层级传播。
- 好处：不再需要手动管理每个任务的取消与生命周期，避免**协程泄漏**（如 Activity 销毁后协程还在跑）。

```kotlin
viewModelScope.launch {              // 父作用域
    launch { fetchDetail() }         // 子协程 1
    launch { fetchComments() }       // 子协程 2
    // 无需逐个记录 Job：ViewModel 销毁时 viewModelScope.cancel()
    // → 两个子协程自动取消，不会在页面销毁后继续跑
}
```

### 4.6 协程如何取消？`cancel()` 后还在执行吗？

- `job.cancel()` / `scope.cancel()` 发出取消信号；`isActive`/`ensureActive()` 检查。
- **取消是协作式的**：只有挂起点（`delay`、`withContext`、挂起 IO）才会抛 `CancellationException` 退出。
- 纯 CPU 循环不检查取消则不会停止（需 `isActive` 判断）。
- 取消异常是**正常流程**，不应 catch 后吞掉，否则协程无法取消。

```kotlin
val job = launch(Dispatchers.Default) {
    while (isActive) {               // CPU 密集循环没有挂起点，必须手动检查
        computeChunk()
    }
}
job.cancel()

// 错误示范：catch (e: Exception) { } 会把 CancellationException 一并吞掉
// → 协程"取消"后仍在继续跑；正确做法是 rethrow 或使用 runCatching 后重新抛出
```

### 4.7 `runBlocking` 与协程的关系？为什么 Android 主线程不能用？

- `runBlocking` 会**阻塞当前线程**直到内部协程完成，是"桥接"阻塞世界与协程世界用的（测试、main 函数）。
- Android 主线程调用 `runBlocking` 会**卡死 UI**，严禁使用；替代：viewModelScope/lifecycleScope 的 `launch`。
- 理解即可：非阻塞的协程不能从阻塞的 API 直接创建，需要桥接。

```kotlin
fun main() = runBlocking {           // main / 单元测试中桥接用
    launch { delay(100); println("world") }
    println("hello")                 // 先输出 hello，100ms 后输出 world
}
// Android 主线程等价写法 = 卡死 UI，严禁：runBlocking { api.getUser(id) }
```

### 4.8 `CoroutineScope` 与 `GlobalScope` 的区别？

- `GlobalScope`：**独立生命周期**，不与任何组件绑定，无法被统一取消 → 容易泄漏，**不推荐**（测试/少量场景除外）。
- `CoroutineScope`：由 `CoroutineScope()`/`MainScope()` 或组件的扩展创建（如 `viewModelScope`），随组件销毁自动取消。
- Android 常用作用域：`viewModelScope`（ViewModel 清除时取消）、`lifecycleScope`（Lifecycle 销毁时取消）、`repeatOnLifecycle`。

```kotlin
// 反面：GlobalScope 泄漏
GlobalScope.launch { uploadLogs() }      // Activity 销毁后仍在跑，且无法统一取消

// 正面：生命周期绑定
class MyViewModel : ViewModel() {
    fun load() = viewModelScope.launch { api.getUser(1) }
}   // ViewModel.onCleared() 时 viewModelScope 自动 cancel
```

### 4.9 Kotlin Flow 是什么？与 RxJava 的区别？

- **Flow**：冷数据流（Kotlin 官方响应式方案），基于协程，具备背压、取消、操作符（map/filter/collect 等）。
- 对比 RxJava：

| 维度 | RxJava | Flow |
| ---- | ------ | ---- |
| 归属 | ReactiveX 第三方库 | Kotlin 官方 + kotlinx.coroutines |
| 学习成本 | 高（大量操作符/概念） | 低，操作符与集合相似 |
| 协程集成 | 需适配器 | 原生 suspend/结构化并发 |
| 背压 | 策略多（复杂） | 挂起式天然背压 |
| 生命周期 | 需手动 dispose | 随作用域自动取消 |

- Android 状态流：`StateFlow`（状态持有，类似 LiveData 的协程版）、`SharedFlow`（事件流）。
- Flow 是**冷流**：只有 collect 时才开始；用 `stateIn/sharedIn` 可转热流。

```kotlin
// 冷流：声明时什么也不做，collect 时才逐个 emit
val userFlow: Flow<User> = flow {
    emit(api.getUser(1))
}.map { it.copy(name = it.name.trim()) }

// 转热流：作为 ViewModel 的状态持有
val state: StateFlow<UiState> = userFlow
    .map { UiState.Success(it) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Loading)
```

### 4.10 网络请求协程化（Retrofit）为什么快？

```kotlin
viewModelScope.launch {
    val user = withContext(Dispatchers.IO) { api.getUser(id) }  // 或 Retrofit suspend 自动切 IO
    binding.name.text = user.name   // 已回到主线程
}
```

- Retrofit 的 `suspend fun` 内部把回调封装成了挂起函数：请求发生在 OkHttp 线程池，结果通过 continuation 恢复。
- 相比回调嵌套：**代码顺序化**，取消自动传导（协程取消 → 请求取消）。

## 五、Android 开发中的 Kotlin

### 5.1 Google 为什么推行 Kotlin-first？

- 官方统计 Kotlin 显著降低崩溃率（尤其空指针）与代码量；开发者满意度高。
- Kotlin 与 Android 契合：空安全 + 协程 + Compose，新 API（Jetpack、Compose）都以 Kotlin 设计。
- 生态策略：优先 Kotlin 文档与示例，Java 示例逐步退居二线，但 Java 仍被支持。

### 5.2 Jetpack Compose 与传统 View 体系的区别？

- **View 体系**：XML 布局 + findViewById/ViewBinding + 命令式更新（`setText` 等），状态靠手动同步。
- **Compose**：Kotlin 声明式 UI——`@Composable` 函数描述 UI，状态变化自动重组（Recomposition）更新界面，无 findViewById。
- Compose 核心概念：`remember`/`mutableStateOf`（状态）、重组、`StateFlow` 收集、`LaunchedEffect`。
- 现状：新项目推荐 Compose；大型老项目仍大量 View，可混用（AndroidView 互操作）。

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("点击 $count") }   // 状态变 → 自动重组
}
```

### 5.3 ViewModel + StateFlow 的作用？与 LiveData 对比？

- **ViewModel**：存 UI 状态，屏幕旋转等配置变化时**不销毁**；配合 `viewModelScope`。
- LiveData：生命周期感知的观察者模式，主线程回调，老方案。
- StateFlow：协程版状态流——更纯粹、无生命周期概念，需自己配合 `repeatOnLifecycle` 收集；支持协程操作符。
- 建议：新项目用 `StateFlow` + `collectAsState()`（Compose）或 `repeatOnLifecycle`；LiveData 维护老代码。

```kotlin
class UserViewModel : ViewModel() {
    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state: StateFlow<UiState> = _state.asStateFlow()   // 对外只读，防 UI 误写

    fun load() = viewModelScope.launch {
        runCatching { api.getUser(1) }
            .onSuccess { _state.value = UiState.Success(it) }
            .onFailure { _state.value = UiState.Error(it.message ?: "未知错误") }
    }
}
```

### 5.4 Kotlin 协程在 Android 里如何避免"后台任务泄漏"？

- 不要用 `GlobalScope`；**用生命周期绑定的作用域**：
  - Activity/Fragment → `lifecycleScope`。
  - ViewModel → `viewModelScope`（`viewModelScope` 内部是 `SupervisorJob + Dispatchers.Main.immediate`）。
- 销毁自动 cancel；UI 收集 Flow 用 `repeatOnLifecycle(Lifecycle.State.STARTED)` 保证只在可见时收集。

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {   // 可见才开始收集
        viewModel.state.collect { render(it) }     // 进入后台自动停止，省流量省电
    }
}
```

### 5.5 Android 里 `when` + sealed class 的好处举例？

- 见 1.8：用 sealed class 建模 `UiState`，`when` 穷尽分支，编译器保证**每个状态都有处理**，新增状态时强制 review 所有调用点，避免遗漏。

### 5.6 Compose 里为什么推荐 Kotlin 的不可变与 Flow？

- 声明式 UI 依赖"状态 → UI"的确定性映射：不可变数据 + 单向数据流更容易推导重组范围。
- `StateFlow`/`collectAsState` 与 Compose 重组天然配合，状态更新精准触发重组而非整页刷新。

```kotlin
@Composable
fun UserScreen(vm: UserViewModel = viewModel()) {
    val state by vm.state.collectAsState()   // state 变化 → 只有依赖它的部分重组
    when (val s = state) {
        is UiState.Success -> UserList(s.data)   // data class 不可变，重组范围可控
        else -> LoadingIndicator()
    }
}
```

### 5.7 Kotlin 与 Java 混编时 Android 项目注意什么？

- Gradle 配置 Kotlin 插件与 Java 编译共存（`kotlinOptions.jvmTarget` 与 `compileOptions` 对齐）。
- KAPT/KSP 注解处理器选型（Room 等建议 KSP，更快）。
- Java 调用 Kotlin 语法糖加互操作注解（见 3.2 表格）。
- 方法数/包体积：Kotlin 标准库 + 协程库会增加体积，开启 R8 压缩。

## 六、进阶原理与开放题

### 6.1 `inline` 内联函数原理？`reified` 是什么？

- **内联**：函数体在**编译期展开到调用处**，消除 lambda 的额外类/方法开销；lambda 支持非局部返回（`return` 跳出外层函数）。
- `reified`（具体化泛型）：只有内联函数才能用，让泛型类型参数在运行时**保留真实类型**（泛型擦除的例外）：

```kotlin
inline fun <reified T> Any.asType(): T? = this as? T
"hi".asType<String>()   // 无需传 Class，运行时拿到真实类型
```

- 注意：内联增大字节码体积，只适合小函数。

### 6.2 委托 `by` 的几种用法？

- **类委托**：`class C : I by impl` 接口实现委托给成员，少写转发代码。
- **属性委托**：`by lazy`（懒加载）、`by Delegates.observable`（监听变化）、`by map`（从 Map 取属性，适合解析 JSON/配置）、`by remember`（Compose）。
- 原理：委托属性编译时生成 `getValue/setValue` 的辅助类调用。

```kotlin
// 属性变化监听
var name: String by Delegates.observable("<unset>") { _, old, new ->
    println("$old -> $new")
}

// 从 Map 取属性：JSON/配置解析常用
val prefs = mapOf("theme" to "dark", "lang" to "zh")
val theme: String by prefs          // theme == "dark"

// 类委托：接口实现转发给成员，省掉手写转发代码
interface Repo { fun get(): String }
class RepoImpl : Repo { override fun get() = "data" }
class RepoView(repo: Repo) : Repo by repo
```

### 6.3 `apply`/`also` 的返回值设计动机？

- `apply/also` 返回接收者本身 → 便于链式配置；`let/run` 返回 lambda 结果 → 便于转换。
- 面试常问"为什么需要两个返回自身的？"：`apply` 用 `this`（配置对象成员自然），`also` 用 `it`（强调副作用、不想遮蔽 this）。

```kotlin
user.apply { name = "Tom"; age = 20 }          // this：成员直接写，适合批量配置
user.also { log("created user ${it.name}") }   // it：与外层同名属性不冲突，适合副作用
```

### 6.4 Kotlin 协程的异常传播机制？

- 默认：子协程异常会**向上传播**到父作用域并取消兄弟协程（结构化并发）。
- 处理方式：
  - `CoroutineExceptionHandler`：根协程未捕获异常的统一处理器。
  - `SupervisorJob`/`supervisorScope`：**隔离异常**，子协程失败不影响兄弟与父。
  - `try/catch` 包 `await()` 或协程体。
- `viewModelScope` 使用 SupervisorJob，故其子协程异常需自己 catch（官方推荐 catch 后转 UiState.Error）。

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
scope.launch { throw IllegalStateException("任务 A 失败") }   // 只取消自己
scope.launch { println("任务 B 不受影响，继续执行") }
// 若换成普通 Job：任务 A 的异常会取消整个 scope，任务 B 也跟着死掉
```

### 6.5 手写/设计题：如何用协程实现"并发请求两个接口再合并"？

```kotlin
viewModelScope.launch {
    val a = async { api.getA() }        // 并发
    val b = async { api.getB() }
    val combined = a.await() + b.await() // 都完成才继续
    _state.value = UiState.Success(combined)
}
```

- 延伸：失败处理用 `supervisorScope` + `runCatching`；需要超时用 `withTimeout`。

### 6.6 说说你项目中 Kotlin 的使用与踩坑（开放题）

常见加分点：空安全减少 NPE、协程替代回调、sealed 状态建模、扩展函数封装 UI、data class 不可变 UI 状态；坑：`!!` 滥用、平台类型、作用域未绑定导致泄漏、`async` 异常静默、过度链式可读性差、混编互操作注解缺失。

## 七、复习路线小结

1. 基础语法 + 空安全是 Kotlin 面试的"送分题"，务必能讲清 `?. ?: !!` 与 val/var。
2. 与 Java/JVM 的关系题重在**互操作注解表**与"没有 static/受检异常的设计原因"。
3. 协程必考：suspend 原理（CPS 状态机）、Dispatcher、launch/async、结构化并发、Flow。
4. Android 部分准备一段"用 viewModelScope + StateFlow + Compose"的完整描述。
5. 进阶题把 inline/reified、委托、作用域函数、SupervisorJob 异常隔离讲透即可。
