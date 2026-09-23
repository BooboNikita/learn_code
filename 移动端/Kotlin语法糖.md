# Kotlin 语法糖对照（Android 常用）

> Kotlin 相对 Java 的"精简魔法"：一行 Kotlin 往往对应 Java 十几行样板代码。本文按 **语法糖 → Kotlin 写法 → 反编译/手写 Java 等价物** 对照，重点覆盖 Android 日常开发高频用法。理解"糖"背后的原理，面试与排查问题都更有底气。

## 一、空安全操作符

### 1.1 安全调用 `?.` 与 Elvis `?:`

```kotlin
val name = user?.profile?.nickname ?: "匿名"
```

```java
String name;
if (user != null && user.getProfile() != null && user.getProfile().getNickname() != null) {
    name = user.getProfile().getNickname();
} else {
    name = "匿名";
}
```

> 原理：`?.` 是一串判空跳转指令，每个点都短路为 null；`?:` 是 null 分支判断。Java 里三层 if-null 链，Kotlin 一行搞定。

### 1.2 非空断言 `!!`

```kotlin
val name = user!!.name    // 你保证 user 一定非空，否则运行时抛 NPE
```

```java
if (user == null) {
    throw new NullPointerException("user must not be null");
}
String name = user.getName();
```

> 面试考点：`!!` 不是消除 NPE，而是把判空责任转移给程序员，应谨慎使用。

## 二、属性（Property）与 getter/setter

```kotlin
class User(var name: String, val id: Long)   // var 有 getter+setter，val 只有 getter
user.name = "Tom"     // 属性赋值
println(user.name)    // 属性读取
```

```java
public class User {
    private String name;
    private final long id;
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public long getId() { return id; }
}
user.setName("Tom");
System.out.println(user.getName());
```

> 自定义 getter：`val isAdult get() = age >= 18`，Java 侧就是 `isAdult()` 方法。

## 三、`data class`

```kotlin
data class User(val name: String, val age: Int)

val u1 = User("Tom", 18)
val u2 = u1.copy(age = 20)
println(u1)                    // User(name=Tom, age=18)
val (name, age) = u2           // 解构
```

```java
// 手写等价物（IDE 生成）
public class User {
    private final String name;
    private final int age;
    public User(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
    @Override public boolean equals(Object o) { /* 逐字段比较 */ }
    @Override public int hashCode() { /* Objects.hash(name, age) */ }
    @Override public String toString() { return "User(name=" + name + ", age=" + age + ")"; }
    public User copy(String name, int age) { return new User(name, age); }
}
// Java 没有解构，只能手动取值：String name = u2.getName();
```

> Java 侧要调用 `copy` 需显式传全部参数；Kotlin 的 copy 还支持"只改某个字段"的命名参数糖。

## 四、扩展函数与扩展属性

```kotlin
fun String.isPhone(): Boolean = length == 11 && all { it.isDigit() }
fun Context.dp2px(dp: Int): Int = (dp * resources.displayMetrics.density).toInt()
"13800138000".isPhone()
```

```java
// 反编译后是普通静态方法，接收者是第一个隐藏参数
public final class StringExtKt {
    public static final boolean isPhone(String $this) {
        return $this.length() == 11 && ...;
    }
}
// Java 调用（静态方法）：
StringExtKt.isPhone("13800138000");
```

> 关键点：扩展函数**不参与多态**（静态分发），同名成员函数优先；Android 里的 `dp2px`、`toast` 工具函数都是这么封装的。

## 五、作用域函数（apply/let/run/also/with）

```kotlin
val tv = TextView(context).apply {
    text = "Hello"
    textSize = 18f
}
name?.let { println(it.length) } ?: run { println("name is null") }
```

```java
TextView tv = new TextView(context);
tv.setText("Hello");
tv.setTextSize(18f);
// 可空判断需要手写：
if (name != null) {
    System.out.println(name.length());
} else {
    System.out.println("name is null");
}
```

> 记忆口诀：**let/also 用 it，run/apply 用 this；let/run 返回最后一行，also/apply 返回对象本身**。Java 里没有对应物，全是手写命令式代码。

## 六、when 表达式与 if 作为表达式

```kotlin
val x = when (view.id) {
    R.id.btn_ok -> "确定"
    R.id.btn_cancel -> "取消"
    else -> "其他"
}
val max = if (a > b) a else b
```

```java
String x;
switch (view.getId()) {
    case R.id.btn_ok:     x = "确定"; break;
    case R.id.btn_cancel: x = "取消"; break;
    default:              x = "其他";
}
final int max = a > b ? a : b;   // Java 需三元表达式
```

> `when` 还可以匹配类型（`is String`）、区间（`in 1..10`）、多个条件（逗号分隔），`switch` 都做不到。

## 七、智能转换（Smart Cast）

```kotlin
if (x is String) {
    println(x.length)     // 编译器自动把 x 收窄为 String，无需强转
}
```

```java
if (x instanceof String) {
    System.out.println(((String) x).length());   // Java 必须显式强转
}
```

## 八、lambda 与 SAM 转换

### 8.1 lambda 直接传参

```kotlin
button.setOnClickListener { v -> onButtonClick(v) }
val names = list.map { it.uppercase() }
```

```java
button.setOnClickListener(new View.OnClickListener() {
    @Override public void onClick(View v) { onButtonClick(v); }
});
List<String> names = new ArrayList<>();
for (String s : list) names.add(s.toUpperCase());
// 或 Java 8：list.stream().map(String::toUpperCase).collect(...);
```

### 8.2 SAM 转换（接口只有一个抽象方法时）

```kotlin
// Kotlin 任何单抽象方法接口都可以写成 lambda，不限于 Java 接口
view.postDelayed({ doSomething() }, 1000)
```

```java
view.postDelayed(new Runnable() {
    @Override public void run() { doSomething(); }
}, 1000);
```

> 原理：编译器自动推断接口类型并生成匿名实现类；`it` 是单参数 lambda 的默认名字。

## 九、字符串模板

```kotlin
val msg = "用户 $name 年龄 ${user.age}，${if (isVip) "VIP" else "普通"}"
```

```java
String msg = "用户 " + name + " 年龄 " + user.getAge() + "，"
           + (isVip ? "VIP" : "普通");
// 或 String.format：String.format("用户 %s 年龄 %d，%s", name, user.getAge(), ...)
```

## 十、集合工厂函数与不可变集合

```kotlin
val list = listOf(1, 2, 3)              // 只读 List
val map = mapOf("a" to 1, "b" to 2)     // to 是中缀函数，等价 Pair("a", 1)
list.forEach { println(it) }
val doubled = list.map { it * 2 }
```

```java
List<Integer> list = Collections.unmodifiableList(Arrays.asList(1, 2, 3));
Map<String, Integer> map = new HashMap<>();
map.put("a", 1); map.put("b", 2);
for (Integer i : list) System.out.println(i);
List<Integer> doubled = new ArrayList<>();
for (Integer i : list) doubled.add(i * 2);
```

> 注意：Kotlin 的 `listOf` 是**只读视图**（不是不可变拷贝），返回 Java 侧仍是 `java.util.ArrayList` 等，别当成深层不可变。

## 十一、object 单例与 companion object

```kotlin
object AppConfig {
    const val BASE_URL = "https://api.example.com"
    val cache = HashMap<String, String>()
}

class Utils {
    companion object {
        const val TAG = "Utils"
        fun dp2px(dp: Int) = dp * 2
    }
}
```

```java
// object 反编译：饿汉式单例
public final class AppConfig {
    public static final String BASE_URL = "https://api.example.com";
    public static final AppConfig INSTANCE = new AppConfig();
    private AppConfig() {}
}

// companion object 反编译：类内嵌套单例 + 静态字段
public final class Utils {
    public static final String TAG = "Utils";
    public static final Companion Companion = new Companion(null);
    public static final class Companion {
        public final int dp2px(int dp) { return dp * 2; }
    }
}
// 注意：无 @JvmStatic 时 Java 调用是 Utils.Companion.dp2px(1)
```

## 十二、`by lazy` / `Delegates.observable` 委托

```kotlin
val viewModel by lazy { UserViewModel() }           // 首次访问才创建
var name by Delegates.observable("") { _, old, new ->
    Log.d("Tag", "$old -> $new")
}
```

```java
// lazy 等价物：双检锁单例式初始化
private UserViewModel viewModel;
private volatile boolean initialized;
public UserViewModel getViewModel() {
    if (!initialized) {
        synchronized (this) {
            if (!initialized) { viewModel = new UserViewModel(); initialized = true; }
        }
    }
    return viewModel;
}

// observable 等价物：setter 内回调
private String name = "";
public void setName(String value) {
    String old = this.name;
    this.name = value;
    Log.d("Tag", old + " -> " + value);
}
```

## 十三、sealed class + when 穷尽

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<Item>) : UiState()
    data class Error(val msg: String) : UiState()
}
fun render(state: UiState) = when (state) {
    UiState.Loading -> showLoading()
    is UiState.Success -> showList(state.data)
    is UiState.Error -> showError(state.msg)
}   // 无需 else，漏分支编译报错
```

```java
// Java 无密封类（16 前有），手写基类 + 私有构造，易漏分支：
public abstract class UiState {
    private UiState() {}
    public static final class Loading extends UiState {}
    public static final class Success extends UiState { /* ... */ }
    public static final class Error extends UiState { /* ... */ }
}
// 调用处必须写 if-else 链，新加子类不会自动暴露问题
```

## 十四、默认参数与命名参数

```kotlin
fun load(url: String, cache: Boolean = true, timeout: Int = 30_000) {}
load("https://a.com", timeout = 5_000)    // 跳过中间参数，用名字指定
```

```java
// Java 没有默认参数，只能方法重载三份
public void load(String url) { load(url, true, 30000); }
public void load(String url, boolean cache) { load(url, cache, 30000); }
public void load(String url, boolean cache, int timeout) { /* 实现 */ }
// Java 调用也只能按顺序传：load("https://a.com", true, 5000);
```

> 互操作提示：加 `@JvmOverloads` 可自动生成上面三个重载，Java 侧才能享受默认值。

## 十五、顶层函数与 `@file:JvmName`

```kotlin
// utils.kt
fun dp2px(context: Context, dp: Int) = (dp * context.resources.displayMetrics.density).toInt()
```

```java
// 反编译进文件名+Kt 的类
public final class UtilsKt {
    public static final int dp2px(Context context, int dp) { /* ... */ }
}
// Java 调用：UtilsKt.dp2px(context, 8);
```

> 顶层函数 + 扩展函数是 Android 工具类（如 `Context.dp2px`）的标准封装方式，Java 时代是写满 static 方法的 XXUtils 类。

## 十六、is / in 类型与区间判断

```kotlin
when (count) {
    in 0..9 -> "个位数"
    10 -> "正好十"
    else -> "大于十"
}
if (value is String) ...   // 见第七节智能转换
```

```java
if (count >= 0 && count <= 9) { /* ... */ }
else if (count == 10) { /* ... */ }
else { /* ... */ }
```

## 十七、Android KTX 扩展（语言之上的"官方糖"）

Jetpack 在语言特性之上又包了一层扩展，Android 项目最常用的糖：

```kotlin
// View 相关
view.isVisible = false                        // 替代 view.visibility = View.GONE
view.setOnClickListener { }                   // KTX 提供的 lambda 重载

// 生命周期协程
viewLifecycleOwner.lifecycleScope.launch { }  // 替代手动管理 Job + 生命周期监听
viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) { }

// Fragment 事务
fragmentManager.commit {
    replace(R.id.container, MyFragment())
    addToBackStack(null)
}

// Bundle / SharedPreferences
val bundle = bundleOf("key" to 1, "name" to "Tom")
sharedPrefs.edit { putString("token", token) }   // 自动 apply()
```

```java
// 等价 Java 样板（节选）
view.setVisibility(View.GONE);
view.setOnClickListener(new View.OnClickListener() {
    @Override public void onClick(View v) { }
});

// 协程 + 生命周期：手写 Job 保存、onDestroy 中 cancel
// Fragment 事务：
fragmentManager.beginTransaction()
        .replace(R.id.container, new MyFragment())
        .addToBackStack(null)
        .commit();
// SharedPreferences：
SharedPreferences.Editor editor = sharedPrefs.edit();
editor.putString("token", token);
editor.apply();
```

> KTX 本质仍是**扩展函数**，只是由官方维护，与 Jetpack 生命周期深度绑定。

## 十八、语法糖速查表

| Kotlin 糖 | Java 等价物 | 高频场景 |
| --------- | ----------- | -------- |
| `?.` / `?:` | 多层 if-null + 三元 | 空安全 |
| `data class` | 手写 equals/hashCode/copy/toString | 数据模型 |
| 扩展函数 | static 工具方法 | dp2px、Context 工具 |
| `apply`/`let` | 手写变量声明 + 逐行赋值 | View 配置、可空处理 |
| `when` | switch + if-else 链 | 分支逻辑 |
| lambda / SAM | 匿名内部类 | 点击事件、回调 |
| `$` 模板 | 字符串拼接 / String.format | 日志、文案 |
| `listOf`/`mapOf` | Arrays.asList / HashMap.put | 集合初始化 |
| `object` / `companion` | 饿汉单例 / 嵌套单例 | 常量、单例 |
| `by lazy` | 双检锁 | 懒加载对象 |
| 默认参数 | 方法重载 | 可选配置 |
| `by viewModels()` | ViewModelProvider.get | Fragment/Activity |
| `bundleOf` / `commit{}` | Bundle.putXxx / FragmentTransaction | Android KTX |

## 小结

- Kotlin 语法糖 ≈ **编译器替你写 Java 样板**：属性、data class、扩展函数、作用域函数都是"语法 → 字节码模式"的一一映射。
- 最容易被问倒的是：**扩展函数是静态分发不参与多态**、`object` 反编译为饿汉单例、协程糖（`launch`/`repeatOnLifecycle`）背后仍是状态机与生命周期监听。
- Android 项目落地：工具类用**扩展函数**，状态用 **data class + sealed class**，异步用 **coroutine + KTX 作用域**，常量放 **companion object**。
- 混编注意：需要 Java 侧"直接调用"的糖，补 `@JvmStatic` / `@JvmOverloads` / `@JvmField` / `@file:JvmName` 等注解（详见面试题 3.2 表格）。
