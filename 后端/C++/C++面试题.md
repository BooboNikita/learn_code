# C++ 面试题

> C++ 高频面试题整理，按 基础语法 → 内存管理 → 面向对象 → 现代 C++ → STL → 并发 → 手写代码 分类。每题给出考察点与回答要点，重要知识点配可运行 demo（含预期输出），考前快速过一遍即可。

## 一、基础语法

### 1.1 `const` 有哪些用法？顶层 const 与底层 const 的区别？

**考察点**：`const` 的修饰对象与编译期保护。

**要点**：
- 修饰普通变量：值不可修改。
- 修饰指针（**顶层 const**，`int* const p`）：指针本身不可改，指向的值可改。
- 修饰指向内容（**底层 const**，`const int* p`）：指向的值不可改，指针可改。
- 修饰函数参数：`void f(const std::string& s)` 防止被改并允许传临时量。
- 修饰成员函数：`void show() const` 承诺不修改成员，`this` 退化为 `const T*`，只能调 const 成员函数。
- 修饰返回值：防止返回的引用/指针被外部修改。

```cpp
const int* p1;        // 底层 const：*p1 不可改
int* const p2 = &a;   // 顶层 const：p2 不可改
const int* const p3;  // 两者都是
```

> 判断技巧：`const` 在 `*` 左边 → 修饰指向对象（底层）；在 `*` 右边 → 修饰指针本身（顶层）。

### 1.2 `static` 有哪些作用？

- **静态局部变量**：只初始化一次，生命周期到程序结束，函数内保持状态（如单次初始化标记）。
- **静态全局变量 / 静态函数**：限制为**本编译单元可见**（内部链接），避免多文件同名冲突。
- **静态成员变量**：属于类而非对象，所有实例共享，需在类外定义。
- **静态成员函数**：无 `this` 指针，只能访问静态成员，可通过 `类名::` 调用。

```cpp
class Counter {
public:
    static int count_;      // 声明
    static void inc() { ++count_; }
};
int Counter::count_ = 0;    // 类外定义（C++17 起可用 inline 省去）
```

### 1.3 指针与引用的区别？

- 引用是变量的**别名**，必须在定义时初始化且**不能改绑**；指针可空、可随时改指。
- 引用**不占额外语义上的内存**（实现上常是指针），指针本身是存地址的变量。
- `sizeof(引用)` = 被引用对象大小；`sizeof(指针)` = 地址大小（64 位为 8）。
- 引用**更安全**（无空引用），指针更灵活（可做容器元素、可运算）。
- 参数传递：小对象按值、大对象用 `const 引用`，需要改对象/可能为空时用指针。
- 有指针的**多级**（`int**`），没有多级引用。

### 1.4 `struct` 与 `class` 的区别？

- 唯一语法区别：**默认访问权限**不同——`struct` 默认 `public`，`class` 默认 `private`。
- 默认继承方式也不同：`struct` 默认 public 继承，`class` 默认 private 继承。
- 语义上习惯：`struct` 存纯数据（POD），`class` 封装行为。
- `struct` 也可以有成员函数、继承、多态（C++ 中不是 C 的 struct）。

### 1.5 空类的大小是多少？含虚函数的类呢？

- 空类大小为 **1 字节**：编译器塞一个占位字节，保证不同对象地址不同。
- 含**虚函数**的类：多一个**虚表指针 vptr**，64 位平台通常为 **8 字节**（对齐后）。
- 继承空基类可能被空基类优化（EBO），子类可仍是 1 字节。
- 内存对齐会放大：`int + char` 类大小是 8 而非 5。

```cpp
struct Empty {};                        // sizeof == 1
struct VEmpty { virtual void f(); };    // sizeof == 8（64 位，只有 vptr，空类占位字节被优化掉）
struct A { int i; char c; };            // sizeof == 8（4 + 1 对齐到 8）
struct B : Empty { char c; };           // sizeof == 1（EBO 空基类优化）
```

### 1.6 宏、内联函数、`constexpr` 的区别？

| 维度 | 宏 #define | inline 函数 | constexpr |
| ---- | ---------- | ----------- | --------- |
| 类型检查 | 无（纯文本替换） | 有 | 有 |
| 调试 | 困难 | 可调试 | 可调试 |
| 求值时机 | 预处理期 | 运行期（可内联） | **编译期** |
| 作用域 | 无（全局污染） | 遵循作用域 | 遵循作用域 |
| 适用 | 常量/极简片段 | 短小频繁调用的函数 | 必须编译期算出的值 |

```cpp
#define SQUARE(x) ((x) * (x))     // 必须疯狂加括号，否则 int i=3; SQUARE(++i) 出事
inline int add(int a, int b) { return a + b; }
constexpr int square(int x) { return x * x; }  // 编译期可算
```

### 1.7 `sizeof` 与 `strlen` 的区别？

- `sizeof` 是**运算符，编译期**求值，统计的是**字节数**（含 `\0`），对指针得到的是指针大小。
- `strlen` 是**函数，运行期**遍历，统计到 `\0` 为止的字符数（不含 `\0`）。
- 数组名作参数退化为指针后，`sizeof` 会失效（只拿到 8）。

```cpp
char str[] = "hello";          // 数组长度 6（含 \0）
sizeof(str);   // 6
strlen(str);   // 5
char* p = str; // 退化为指针
sizeof(p);     // 8（64 位下是指针大小，不是 6！）
```

### 1.8 `new`/`delete` 与 `malloc`/`free` 的区别？

- `new` 是**运算符**，`malloc` 是**库函数**。
- `new` 除了分配内存还会**调用构造函数**；`delete` 先**调用析构函数**再释放内存；`malloc/free` 只操作内存。
- `new` 返回**类型化指针**，无需强转；`malloc` 返回 `void*`，需要手动强转并指定字节数。
- `new` 失败抛 `std::bad_alloc`；`malloc` 失败返回 `NULL`。
- `new[]/delete[]` 配套（记录元素个数）；`malloc` 配 `free`，不能混用。
- 底层：`new` 内部通常调用 `malloc` 分配，再构造对象。

### 1.9 C++ 内存分区（程序地址空间）？

```
高地址
┌──────────────┐
│    栈 Stack   │  局部变量、函数参数，向下增长，自动回收
├──────────────┤
│      ↓       │
│      ↑       │
├──────────────┤
│    堆 Heap    │  new/malloc 动态分配，向上增长，需手动释放
├──────────────┤
│ 全局/静态区   │  全局变量、static 变量（.bss 未初始化 / .data 已初始化）
├──────────────┤
│  常量区(.rodata)│  字符串字面量、const 全局常量，只读
├──────────────┤
│    代码区     │  函数机器码，只读
低地址
```

- 栈：空间小（默认约 8MB），递归过深 → **栈溢出**。
- 堆：空间大，需配对释放，否则**内存泄漏**。
- `static`/全局：程序启动分配，结束释放。

### 1.10 C++ 与 C 的区别？

- 面向对象：类、继承、多态、封装（C 无）。
- 模板/泛型（C 无）。
- 异常处理（C 无，用错误码）。
- 运算符重载、函数重载、引用、命名空间、bool/string（C 无原生）。
- **`extern "C"`**：C++ 调用 C 库时，防止名字修饰（name mangling）导致链接失败：

```cpp
extern "C" {
    #include "c_header.h"
}
```

### 1.11 `explicit` 有什么作用？什么时候必须加？

**考察点**：隐式类型转换带来的意外行为。

**回答要点**：

- 单参数（或除首参外都有默认值）的构造函数，默认是**转换构造函数**，允许隐式转换 `Meter m = 3.0;`。
- 加 `explicit` 后**禁止隐式转换**，只能显式构造 `Meter m(3.0)` / `Meter(3.0)` / `static_cast<Meter>(3.0)`。
- 转换运算符也可加 `explicit`（C++11 起），典型是 `explicit operator bool()`——`if (obj)` 仍可用（上下文转换），但 `bool b = obj + 1;` 这类意外不再发生。
- **常见踩坑**：容器 `emplace_back` / `push_back` 与单参构造函数组合、函数参数处的隐式转换、`vector<int> v(10)` vs `vector<int> v{10}`。

**Demo：加与不加 explicit 的行为差异**

```cpp
#include <iostream>
using namespace std;

class Meter {
    double v_;
public:
    explicit Meter(double v) : v_(v) {}
    explicit operator bool() const { return v_ != 0.0; }   // 显式 bool 转换
    double value() const { return v_; }
};
void printMeter(Meter m) { cout << m.value() << '\n'; }

int main() {
    Meter a(3.0);          // OK：直接初始化
    // Meter b = 3.0;      // 编译错误：explicit 禁止隐式转换（去掉 explicit 则通过）
    // printMeter(3.0);    // 编译错误：形参处不做隐式转换
    printMeter(Meter(3.0));   // OK：显式构造 → 3
    printMeter(a);            // 3

    cout << boolalpha << static_cast<bool>(a) << '\n';  // true
    // if (a) 语句中仍可用 explicit operator bool（上下文转换）
}
```

## 二、内存管理

### 2.1 深拷贝与浅拷贝？何时必须自己写拷贝构造？

- **浅拷贝**：默认逐字节拷贝，指针成员会"复制地址"→ 两个对象指向同一块内存 → 析构时**双重释放**。
- **深拷贝**：为指针成员**重新分配内存并复制内容**。
- 类含**裸指针/动态资源**且使用默认拷贝时，必须自定义拷贝构造 + 拷贝赋值 + 析构（**三法则 Rule of Three**）。
- C++11 后更推荐**智能指针成员**替代裸指针（拷贝即自动深拷贝语义）。

```cpp
class String {
    char* data_;
public:
    String(const char* s) {                      // 构造：分配
        data_ = new char[strlen(s) + 1];
        strcpy(data_, s);
    }
    String(const String& o) {                    // 拷贝构造：深拷贝
        data_ = new char[strlen(o.data_) + 1];
        strcpy(data_, o.data_);
    }
    String& operator=(const String& o) {         // 拷贝赋值：先处理自赋值/旧内存
        if (this != &o) {
            delete[] data_;
            data_ = new char[strlen(o.data_) + 1];
            strcpy(data_, o.data_);
        }
        return *this;
    }
    ~String() { delete[] data_; }                // 析构
};
```

### 2.2 什么是内存泄漏？如何检测与避免？

- **定义**：动态分配的内存失去指针引用后无法释放，长期运行内存持续上涨。
- **产生**：`new` 后没 `delete`、异常路径中漏释放、容器存裸指针忘清理。
- **避免**：优先 RAII（智能指针、容器管理资源）；**绝不**用裸 `new` 管理资源。
- **检测**：
  - Valgrind：`valgrind --leak-check=full ./a.out`（Linux）。
  - AddressSanitizer（编译加 `-fsanitize=address`，GCC/Clang 均可）。
  - 重载 `new/delete` 做统计；工具如 Visual Studio 的 CRT 泄漏检测。

### 2.3 什么是 RAII？

- **RAII（Resource Acquisition Is Initialization）**：资源在**构造函数中获取**、在**析构函数中释放**。
- 栈对象离开作用域必然调用析构 → 资源必然被释放，**异常安全**。
- 典型应用：智能指针、`std::lock_guard`、`std::fstream`、`std::unique_ptr`。

```cpp
void f() {
    std::unique_ptr<Foo> p = std::make_unique<Foo>();  // 构造获取资源
    std::lock_guard<std::mutex> lk(mtx_);              // 构造上锁
    do_work();                                          // 中途抛异常也没关系
}   // 离开作用域：p 析构释放内存，lk 析构解锁 —— 自动完成
```

### 2.4 什么是内存对齐？为什么要对齐？

- **内存对齐**：变量地址是其大小的整数倍（如 `int` 4 字节对齐），结构体按最大成员对齐并**尾部填充**。
- **原因**：CPU 访问对齐数据是单次取址，跨边界访问可能两次取址甚至硬件报错；保证原子性、性能。
- 影响：成员排列顺序影响结构体大小。

```cpp
struct A { char c; int i; };      // 1 + 3填充 + 4 = 8
struct B { char c; char d; int i; }; // 1+1+2填充+4 = 8（重排可省内存）
#pragma pack(1)                    // 也可以紧凑打包（牺牲性能换空间）
struct C { char c; int i; };      // = 5
```

### 2.5 为什么 `vector` 扩容用拷贝/移动而 `realloc` 不行？

- C 的 `realloc` 依赖"原指针可原地扩展"，失败时移动后**自动释放旧块**，且不知道对象语义。
- C++ 对象有构造/析构，扩容必须**在新位置构造、旧位置析构**，不能盲目拷贝字节。
- 移动元素后旧指针语义变化（迭代器失效），需要显式控制——所以 STL 自行管理（`new` + move + `delete`）。

### 2.6 堆内存申请失败会怎样？如何防护？

- `new` 失败默认抛 `std::bad_alloc`；`malloc` 失败返回 `NULL`。
- 服务器常见处理：申请前评估大小、捕获 `bad_alloc`、做降级/限流；Linux 下注意 **overcommit**（malloc 可能先成功，真正触碰内存时才 OOM Kill）。
- 可用 `new (std::nothrow)` 获得返回空指针的版本。

### 2.7 野指针、悬垂指针、double free 有什么区别？

**考察点**：指针生命周期管理，区分三种典型错误。

| 类型 | 定义 | 典型成因 |
| ---- | ---- | -------- |
| **野指针** | 指向**未知/随机**地址的指针 | 未初始化 `int* p;` |
| **悬垂指针** | 指向**已释放/已失效**内存的指针 | `delete` 后继续用、返回局部变量地址、迭代器失效 |
| **double free** | 同一块内存被释放两次 | 两个指针指向同一块、拷贝后各析构一次 |

**回答要点**：

- 三者共同点：**解引用即未定义行为**（可能崩溃、可能静默改错数据，比崩溃更可怕）。
- 防护：定义即初始化为 `nullptr`；`delete` 后立刻置空（`delete p; p = nullptr;`）；优先 `unique_ptr/shared_ptr` 与容器，从源头避免裸 `new`。
- 悬垂的现代变体：`shared_ptr` **循环引用**导致对象永不释放（是"泄漏"不是悬垂）；`weak_ptr` 的 `expired()`/`lock()` 正是为安全探测悬垂而设计。
- 检测手段：ASan（`-fsanitize=address`）能精确定位 use-after-free / double-free；Valgrind 同样有效。

**Demo：用 weak_ptr 安全探测"对象已销毁"**

```cpp
#include <iostream>
#include <memory>
using namespace std;

struct Res {
    int id;
    Res(int i) : id(i) { cout << "Res(" << id << ") 构造\n"; }
    ~Res() { cout << "Res(" << id << ") 析构\n"; }
};

int main() {
    weak_ptr<Res> wp;                      // 弱引用，不延长生命周期
    {
        shared_ptr<Res> sp = make_shared<Res>(1);
        wp = sp;
        cout << "use_count=" << wp.use_count()
             << " expired=" << wp.expired() << '\n';   // 1 false
    }                                       // sp 析构 → 对象释放
    cout << "离开作用域后 expired=" << wp.expired() << '\n';   // true

    shared_ptr<Res> sp2 = wp.lock();         // 升级失败返回空，不会 UAF
    cout << "lock() 得到 " << (sp2 ? "有效" : "空指针") << '\n';
    // 反面教材（未定义行为，切勿模仿）：
    // Res* raw = new Res(9); delete raw; cout << raw->id;   // 悬垂后解引用
    // delete raw;                                            // double free
}
```

```text
Res(1) 构造
use_count=1 expired=false
Res(1) 析构
离开作用域后 expired=true
lock() 得到 空指针
```

### 2.8 什么是 placement new？有什么用？

**考察点**：在指定内存上构造对象，STL 与内存池的底层手法。

**回答要点**：

- `new (ptr) T(args...)` 在**已分配的原始内存** `ptr` 上构造对象，**不分配内存**，只在原地调构造函数。
- 三要素：**(1) 有对齐的内存**（`alignas(T) char buf[sizeof(T)]` / 内存池 / `mmap`）；**(2) placement new 构造**；**(3) 必须手动 `obj->~T()` 析构**。
- 用途：内存池/对象池、`std::vector` 的扩容（分配裸内存 → 逐个构造）、`std::optional/variant` 的实现、固定地址的硬件寄存器映射。
- 注意：不能用 `delete` 释放（它不知道内存从哪来），必须显式调用析构 + 自行释放原始内存。

**Demo：栈缓冲区上原地构造并手动析构**

```cpp
#include <iostream>
#include <new>          // placement new 需要
using namespace std;

struct P {
    int x;
    P(int v) : x(v) { cout << "P ctor " << x << '\n'; }
    ~P() { cout << "P dtor " << x << '\n'; }
};

int main() {
    alignas(P) unsigned char buf[sizeof(P)];      // 一块未初始化的原始内存（栈上）
    cout << "buf 地址=" << static_cast<void*>(buf)
         << " size=" << sizeof(P) << '\n';        // size=4

    P* obj = new (buf) P(7);                      // 原地构造，不分配堆内存
    cout << "obj->x=" << obj->x << '\n';          // 7

    obj->~P();                                    // 必须手动析构
    // buf 本身是栈内存，随作用域自动回收，无需 delete
}
```

### 2.9 什么是三法则 / 五法则 / 零法则？

**考察点**：特殊成员函数的成组定义规则（与 2.1 深拷贝强相关）。

**回答要点**：

- **三法则（Rule of Three）**：需要自定义**析构 / 拷贝构造 / 拷贝赋值**中任意一个时，通常三个都要写（管着裸资源）。
- **五法则（Rule of Five）**：C++11 加了**移动构造 / 移动赋值**，若类**可移动**（能廉价转移资源），则应一并定义，共 5 个特殊成员函数。
- **零法则（Rule of Zero）**：**首选**——用 `string/vector/unique_ptr/shared_ptr` 等已管理资源的成员，则**一个都不用写**，编译器默认生成的版本全部正确。
- 关键坑：**声明了移动操作后，拷贝操作会被隐式 delete**（反之亦然）——定义任意一个，就要把其余四个显式 `= default` 或 `= delete` 补齐，避免语义意外。
- 移动构造/赋值要加 `noexcept`（原因见 4.5）。

**Demo：完整五法则，观察每个特殊成员被调用的时机**

```cpp
#include <iostream>
#include <cstring>
#include <utility>
using namespace std;

class Buffer {
    size_t n_;
    char*  p_;
public:
    explicit Buffer(const char* s) : n_(strlen(s) + 1), p_(new char[n_]) {
        memcpy(p_, s, n_); cout << "  ctor\n";
    }
    Buffer(const Buffer& o) : n_(o.n_), p_(new char[n_]) {       // 3. 拷贝构造
        memcpy(p_, o.p_, n_); cout << "  copy ctor\n";
    }
    Buffer(Buffer&& o) noexcept : n_(o.n_), p_(o.p_) {           // 5. 移动构造：偷指针
        o.p_ = nullptr; o.n_ = 0; cout << "  move ctor\n";
    }
    Buffer& operator=(const Buffer& o) {                          // 4. 拷贝赋值
        cout << "  copy assign\n";
        if (this != &o) { Buffer tmp(o); swap(tmp); }             // copy-and-swap
        return *this;
    }
    Buffer& operator=(Buffer&& o) noexcept {                      // 6. 移动赋值
        cout << "  move assign\n";
        if (this != &o) { delete[] p_; n_ = o.n_; p_ = o.p_; o.p_ = nullptr; o.n_ = 0; }
        return *this;
    }
    ~Buffer() { delete[] p_; cout << "  dtor\n"; }                // 2. 析构
    void swap(Buffer& o) noexcept { std::swap(n_, o.n_); std::swap(p_, o.p_); }
};

int main() {
    Buffer a("hello");                 // ctor
    Buffer b(a);                       // copy ctor
    Buffer c(std::move(a));            // move ctor
    Buffer d("x"); d = std::move(b);   // ctor + move assign
    // 结束作用域：4 个 dtor
}
```

```text
  ctor
  copy ctor
  move ctor
  ctor
  move assign
  dtor
  dtor
  dtor
  dtor
```

> 写 demo 时的一个真实收获：这个类成员声明顺序写成 `p_` 在前、`n_` 在后，而初始化列表写 `n_(...), p_(new char[n_])`，编译器立刻给出 `-Wreorder` 警告，甚至 `n_` 未初始化——印证 3.7「初始化顺序由声明顺序决定」。

## 三、面向对象与多态

### 3.1 什么是虚函数？多态的实现原理？

**考察点**：动态绑定机制、vtable/vptr 内存模型。

**回答要点**：

- **虚函数**：`virtual` 修饰的成员函数，通过基类指针/引用调用时**动态绑定**到实际对象版本。
- **实现**：类含虚函数 → 编译器为每个类生成一张**虚表（vtable，函数指针数组，一个类一份，存于只读数据段）**；每个对象头部放一个**虚表指针（vptr，一个对象一份，随对象初始化）**。
- 调用流程：取对象 vptr → 查 vtable 中对应槽位 → 间接调用函数。运行期决定，代价是**一次间接寻址 + 无法内联**。
- 多态三要素：**继承 + 虚函数重写 + 基类指针/引用调用**，三者缺一不可。
- 追问：vtable 编译期生成，vptr 在构造函数中由编译器插入的代码写入——这就是构造函数不能是虚函数的原因（见 3.2）。

**Demo 1：基本多态 + 静态绑定对比**

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    void eat()        { cout << "animal eat\n"; }   // 非虚：静态绑定
    virtual void speak() { cout << "animal speak\n"; } // 虚：动态绑定
    virtual ~Animal() = default;
};
class Dog : public Animal {
public:
    void eat()   { cout << "dog eat\n"; }           // 隐藏，不是重写
    void speak() override { cout << "wang\n"; }     // 重写
};

int main() {
    Dog dog;
    Animal& r = dog;          // 基类引用
    r.eat();                  // animal eat  —— 非虚函数，编译期看引用类型
    r.speak();                // wang        —— 虚函数，运行期看真实对象

    Animal a;
    Animal* p = &a;  p->speak();   // animal speak
    p = &dog;        p->speak();   // wang：同一句调用，结果随对象变 → 多态
}
```

**Demo 2：验证 vptr 的存在与 vtable 共享**

```cpp
class NoVirtual { int x_; };        // 无虚函数
class HasVirtual { int x_; public: virtual void f(); };

// 64 位平台（GCC/Clang）：
static_assert(sizeof(NoVirtual) == 4);    // 只有 int
static_assert(sizeof(HasVirtual) == 16);  // vptr(8) + int(4) + 对齐到 8 的倍数

// 同一个类的所有对象共享同一张 vtable：
HasVirtual a, b;
// (*(void**)&a) == (*(void**)&b) —— 两个对象头部的 vptr 相同（可用打印验证）
void* vptr_a = *(void**)&a;
void* vptr_b = *(void**)&b;
cout << boolalpha << (vptr_a == vptr_b) << endl;   // true
```

### 3.2 构造函数为什么不能是虚函数？析构函数为什么必须是 virtual？

**考察点**：对象构造/析构与虚机制建立的先后关系。

**回答要点**：

- **构造函数不能虚**：
  - 虚调用依赖对象里的 vptr，而 vptr 是在**构造过程中**才被赋值的——鸡生蛋问题；
  - 构造时类型是确定的（`new Dog` 就是 Dog），不需要动态绑定；
  - 从 vtable 角度：vptr 都还没写入，无从查表。
- **析构函数要虚**：通过基类指针 `delete` 派生类对象时，若析构非虚，**静态绑定**到基类析构 → 派生类部分资源**泄漏**。虚析构会被编译器特殊处理：`delete p` 时先虚派发到最终派生类析构，再自动逐层向上析构。
- 规则：**只要类可能被继承并通过基类指针管理，析构函数就声明 `virtual`**；不打算被继承的类加 `final` 或析构用 `protected` 非虚（阻止基类指针 delete）。
- 追问：**虚析构本身是虚函数**，也会占用 vtable 槽位；一个含虚函数的类，其析构是否 virtual 不影响对象大小。

**Demo：非虚析构导致泄漏（用打印模拟资源释放）**

```cpp
#include <iostream>
using namespace std;

struct Bad {
    ~Bad() { cout << "~Bad\n"; }                    // 非虚析构
};
struct BadChild : Bad {
    int* data_;
    BadChild()  : data_(new int[100]) { }
    ~BadChild() { cout << "~BadChild (释放 data_)\n"; delete[] data_; }
};

struct Good {
    virtual ~Good() { cout << "~Good\n"; }          // 虚析构
};
struct GoodChild : Good {
    int* data_;
    GoodChild()  : data_(new int[100]) { }
    ~GoodChild() { cout << "~GoodChild (释放 data_)\n"; delete[] data_; }
};

int main() {
    Bad* b = new BadChild;
    delete b;      // 只打印 ~Bad！BadChild 析构没执行 → data_ 泄漏 100*4 字节
    Good* g = new GoodChild;
    delete g;      // 依次打印 ~GoodChild (释放 data_) → ~Good，完整释放
}
```

### 3.3 构造/析构函数里调用虚函数会多态吗？

**考察点**：构造/析构期间 vptr 的指向变化。

**回答要点**：

- **不会多态**。执行到某层的构造函数时，vptr 被**临时指向该层类的 vtable**：
  - 构造顺序：基类构造 → vptr 切到基类表 → 基类构造体内虚调用解析为**基类版本**；
  - 进入派生类构造体前，vptr 才切到派生类表。
- 析构对称：进入派生类析构体时 vptr 已切回派生类表，析构完基类前又切回基类表。
- **原因（安全角度）**：基类构造时派生类成员尚未初始化，若派发到派生类版本，会操作未初始化的数据 → C++ 有意如此设计。
- 最佳实践：构造/析构中**不要调用虚函数**；需要"构造后初始化"时用工厂函数或两段式 `init()`。

**Demo：构造期虚调用永远打到当前层**

```cpp
#include <iostream>
using namespace std;

class Base {
public:
    Base() { hook(); }                 // 期望调用派生类版本？不会！
    virtual void hook() { cout << "Base::hook\n"; }
    virtual ~Base() = default;
};
class Derived : public Base {
    int id_;
public:
    Derived() : id_(42) { hook(); }    // 此时 vptr 已切到 Derived，才会多态
    void hook() override { cout << "Derived::hook, id_=" << id_ << '\n'; }
};

int main() {
    Derived d;
    // 输出：
    // Base::hook            ← Base() 内调用，vptr 还指着 Base 的表
    // Derived::hook, id_=42 ← Derived() 体内调用，vptr 已切到 Derived
}
```

### 3.4 重载（overload）、重写（override）、重定义（hide）的区别？

**考察点**：三种"同名函数"关系的辨析，高频陷阱题。

**回答要点**：

| 维度 | 重载 overload | 重写 override | 隐藏 hide |
| ---- | ------------- | ------------- | --------- |
| 作用域 | **同一类**中 | 基类 ↔ 派生类 | 基类 ↔ 派生类 |
| 条件 | 同名不同参（仅返回值不同不行） | 基类 `virtual` + **签名一致**（协变返回除外） | 派生类任意同名函数（非重写） |
| 绑定时机 | 编译期（静态） | 运行期（动态） | 编译期（静态） |
| 防错手段 | — | 加 `override` 让编译器检查 | 用 `using Base::f;` 把基类重载引入派生类 |

- 隐藏的坑：只要派生类有同名函数，**基类的所有重载版本全部被隐藏**（名字查找先于重载决议）。

**Demo：一例看清三者**

```cpp
#include <iostream>
using namespace std;

class Base {
public:
    void f()               { cout << "Base::f()\n"; }       // 将被隐藏
    void f(int)            { cout << "Base::f(int)\n"; }    // 将被隐藏（连带）
    virtual void g()       { cout << "Base::g()\n"; }       // 将被重写
};

class Derived : public Base {
public:
    void f()               { cout << "Derived::f()\n"; }    // 隐藏 Base 的两个 f
    void g() override      { cout << "Derived::g()\n"; }    // 重写
    // using Base::f;      // 取消注释后 d.f(1) 才能编译通过
};

int main() {
    Derived d;
    Base* p = &d;

    d.f();       // Derived::f()   —— 隐藏，静态绑定
    // d.f(1);   // 编译错误！Base::f(int) 被名字隐藏，哪怕参数匹配也不参与查找
    d.Base::f(1);  // Base::f(int) —— 显式限定才能调

    p->g();      // Derived::g()   —— 重写，动态绑定
}
```

### 3.5 什么是纯虚函数与抽象类？接口类怎么定义？

**考察点**：抽象类语义、接口设计规范。

**回答要点**：

- `virtual void f() = 0;` 为**纯虚函数**；含有纯虚函数的类是**抽象类**，**不能实例化**，只能作基类。
- 派生类必须**实现全部**纯虚函数才能实例化，否则它也是抽象类（可用来做"半成品中间层"）。
- 纯虚函数**可以有函数体**（提供公共默认实现，派生类用 `Base::f()` 显式调用），但类仍抽象。
- 接口类规范：**全纯虚函数 + 虚析构 + 无数据成员**；C++20 可用 `= 0` 配合 concepts 表达能力约束。
- 与虚析构的关系：接口类即使没有资源也**必须虚析构**，因为使用方一定通过基类指针 delete。

**Demo：抽象类 + 接口风格的完整用例**

```cpp
#include <iostream>
#include <memory>
#include <vector>
using namespace std;

class IShape {                          // 接口类
public:
    virtual double area() const = 0;
    virtual void   print() const = 0;
    virtual ~IShape() = default;        // 接口类必备
};

class Circle : public IShape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return 3.14159 * r_ * r_; }
    void   print() const override {
        cout << "Circle r=" << r_ << " area=" << area() << '\n';
    }
};

class Rect : public IShape {
    double w_, h_;
public:
    Rect(double w, double h) : w_(w), h_(h) {}
    double area() const override { return w_ * h_; }
    void   print() const override {
        cout << "Rect " << w_ << "x" << h_ << " area=" << area() << '\n';
    }
};

int main() {
    // IShape s;                       // 编译错误：抽象类不能实例化
    vector<unique_ptr<IShape>> shapes;
    shapes.push_back(make_unique<Circle>(2.0));
    shapes.push_back(make_unique<Rect>(3.0, 4.0));

    double total = 0;
    for (auto& s : shapes) {           // 面向接口编程
        s->print();
        total += s->area();
    }
    cout << "total = " << total << '\n';
    // Circle r=2 area=12.5664
    // Rect 3x4 area=12
    // total = 24.5664
}
```

### 3.6 菱形继承的 DDD（钻石问题）？虚继承如何解决？

**考察点**：多继承二义性、虚基类机制。

**回答要点**：

- 菱形继承：B、C 都继承 A，D 同时继承 B、C → **D 中含两份 A 子对象**，访问 A 成员产生歧义，数据也可能不一致。
- 解法：**虚继承** `class B : virtual public A`，B/C 共享同一个 A 子对象（虚基类），D 中只保留一份。
- 实现：虚基类子对象由**最终派生类**直接初始化（D 的构造函数初始化列表里直接写 `A(...)`），B/C 对 A 的初始化在最终派生类构造时被跳过。
- 代价：访问虚基类成员需要额外间接（类似 vtable 的 vbtable/vptr 指针），对象变大、构造复杂；**日常开发优先用组合或单继承 + 接口替代多继承**。

**Demo：普通菱形 vs 虚继承**

```cpp
#include <iostream>
using namespace std;

// ---- 普通菱形：两份 A ----
namespace bad {
struct A { int v = 1; };
struct B : A {};
struct C : A {};
struct D : B, C {};
}
// ---- 虚继承：一份 A ----
namespace good {
struct A { int v = 1; };
struct B : virtual A {};
struct C : virtual A {};
struct D : B, C {};
}

int main() {
    bad::D d1;
    // d1.v = 2;            // 编译错误：二义性，d1 有两份 v（B::A::v 和 C::A::v）
    d1.B::v = 2;            // 只改了 B 路径那份，C::A::v 仍是 1 → 数据不一致
    cout << d1.B::v << ' ' << d1.C::v << '\n';   // 2 1

    good::D d2;
    d2.v = 2;               // OK：只有一份 A 子对象
    cout << d2.v << ' ' << d2.B::v << '\n';      // 2 2 —— 三个名字同一个变量

    cout << sizeof(bad::D) << ' '                // 8：两份 int
         << sizeof(good::D) << '\n';             // 24（arm64 clang）：2 个 vbptr + 1 份 A
}
```

### 3.7 初始化列表的作用？成员初始化顺序？

- **作用**：必须用它初始化——`const` 成员、引用成员、没有默认构造函数的类成员、基类。
- 用初始化列表可**少一次默认构造**（直接拷贝初始化，而非先默认构造再赋值），性能更好。
- **初始化顺序与声明顺序一致**，与初始化列表书写顺序无关（警惕警告：`-Wreorder`）。

```cpp
class A {
    int& ref_;
    const int c_;
    int b_, a_;
public:
    A(int& r) : ref_(r), c_(1), a_(1), b_(2) {}  // 实际顺序：ref_ c_ b_ a_
};
```

### 3.8 拷贝构造/拷贝赋值的参数为什么必须用引用？

- 如果按值传参，拷贝形参时需要调用拷贝构造本身 → **无限递归**。

### 3.9 什么是对象切片（object slicing）？

**考察点**：按值使用多态对象导致的类型信息丢失，多态题的高频追问。

**回答要点**：

- **定义**：把派生类对象**按值**赋给基类对象（或按值传参、放入 `vector<Base>`）时，只拷贝基类子对象部分，**派生类新增成员与虚表指针被"切掉"**。
- 后果：**多态失效**——调用的是基类版本；派生类成员丢失，甚至出现"部分构造"的诡异状态。
- 三种典型触发场景：`Base b = derived;`、`void f(Base b);`、`vector<Base> v; v.push_back(derived);`
- **正确做法**：
  - 函数参数用 `const Base&` / `Base*`；
  - 需要存异构对象用 `vector<unique_ptr<Base>>`（或 `shared_ptr`）；
  - 想按值传递语义，用 **clone 惯用法**（基类虚 `clone()` 返回 `unique_ptr<Base>`）。
- 同理：**基类拷贝构造应保护**（`protected` 或 `= delete`）来主动防切片。
- 加分点：`Base` 若含纯虚函数则无法按值实例化，切片问题自然被编译器挡住——这也是接口类都用纯虚的原因之一。

**Demo：切片 vs 引用，同一对象两种结果**

```cpp
#include <iostream>
#include <memory>
#include <vector>
using namespace std;

struct Base {
    int b = 1;
    virtual void who() const { cout << "Base\n"; }
    virtual ~Base() = default;
};
struct Derived : Base {
    int d = 2;
    void who() const override { cout << "Derived\n"; }
};

void takeByValue(Base s)      { s.who(); }   // 切片：只拷 Base 部分
void takeByRef(const Base& s) { s.who(); }   // 正确：保留完整对象

int main() {
    Derived ds;
    cout << "sizeof(Base)=" << sizeof(Base)
         << " sizeof(Derived)=" << sizeof(Derived) << '\n';   // 16 16

    cout << "按值传参: "; takeByValue(ds);      // Base    ← 多态失效（切片）
    cout << "按引用传参: "; takeByRef(ds);      // Derived ← 多态正常

    vector<Base> sliced;
    // sliced.push_back(ds);    // 同样切片，且无法存派生类信息
    vector<unique_ptr<Base>> ok;                 // 正确姿势
    ok.push_back(make_unique<Derived>());
    ok[0]->who();                                // Derived

    // Base b = ds;  b.who();   // 输出 Base：最典型的切片写法
}
```

## 四、现代 C++（C++11 及以后）

### 4.1 智能指针的实现原理？各自特点？

- `unique_ptr`：**独占**所有权，禁止拷贝（拷贝构造被 delete），支持移动；可自定义删除器；**零额外开销**，性能同裸指针。
- `shared_ptr`：**引用计数**，拷贝计数 +1，析构计数 -1，归零才释放；计数线程安全（对象本身不保证）。控制块存计数 + 删除器 + 弱计数。
- `weak_ptr`：**弱引用**，不增加计数；用于**打破循环引用**；访问前需 `lock()` 升级为 shared_ptr（成功则对象仍存活）。
- 三者都基于 RAII：析构自动释放 → 内存安全、异常安全。

```cpp
std::shared_ptr<int> a = std::make_shared<int>(1);
auto b = a;            // 计数 2
// 循环引用场景
struct Node { std::shared_ptr<Node> next; std::weak_ptr<Node> prev; };
```

### 4.2 `make_shared` 与 `new shared_ptr` 的区别？

- `make_shared` **一次分配**（对象 + 控制块在同一内存块），少一次 malloc，缓存友好。
- `make_shared` **异常安全**：`f(shared_ptr<A>(new A), g())` 若 `g()` 先抛异常会泄漏裸指针，`make_shared` 无此问题。
- 缺点：控制块和对象同块，`weak_ptr` 还存在时对象内存无法提前释放（都随控制块一起存活）。

### 4.3 shared_ptr 的线程安全性？

- 同一 shared_ptr 的**引用计数读写是原子的**，多个线程同时拷贝/析构是安全的。
- 但**指向的对象本身不线程安全**；对同一 shared_ptr 变量同时写（reset/赋值）也不安全。
- 结论：多线程安全共享对象，仍需外部加锁或用 `std::atomic`。

### 4.4 右值引用、移动语义、完美转发？

- **左值**：有名字、可取地址；**右值**：临时量、即将销毁，可被"窃取"资源。
- 右值引用 `T&&` 只能绑右值 → 移动构造把对方资源"偷"过来，置空对方，**O(1)**。
- `std::move(x)`：`static_cast<T&&>(x)`，把左值**标记为右值**，触发移动而非拷贝。
- **完美转发**：模板中 `T&&`（转发引用）+ `std::forward<T>(x)`，保持实参的左/右值属性传给下一层。

```cpp
class Buffer {
    char* p_;
public:
    Buffer(Buffer&& o) noexcept : p_(o.p_) { o.p_ = nullptr; }  // 移动构造：偷指针
    Buffer& operator=(Buffer&& o) noexcept { ... }
};

template <typename T>
void wrapper(T&& x) {   // 转发引用
    inner(std::forward<T>(x));  // 保持左值/右值属性
}
```

### 4.5 移动构造/移动赋值为什么要加 `noexcept`？

- STL 容器（尤其 vector）扩容时：如果移动构造**可能抛异常**，为保证强异常安全，只能退化为**拷贝**。
- 声明 `noexcept` 后容器才敢放心使用移动 → 扩容才真正 O(1) 高效。
- 标准规定：`std::vector` 仅在移动构造为 `noexcept`（或不可用）时采用移动扩容。

### 4.6 lambda 的本质？可以捕获什么？

- lambda 本质是**匿名函数对象**（编译器生成一个类，重载 `operator()`）。
- 捕获方式：`[]` 不捕获、`[=]` 按值、`[&]` 按引用、`[a, &b]` 混合、`[this]`、C++14 起初始化捕获 `[x = std::move(v)]`。
- 按值捕获的变量在**定义时拷贝**；`mutable` 允许修改按值捕获的副本。
- 泛型 lambda（C++14）：参数写 `auto`。

```cpp
int base = 10;
auto f = [base](int x) mutable { return base += x; };  // 拷贝的 base 可改
auto g = [&base](int x) { return base += x; };         // 引用捕获，影响外部
```

### 4.7 `std::function` 与函数指针的区别？

- 函数指针只能指向**普通函数/静态函数**（签名匹配）。
- `std::function` 是**可调用对象包装器**：普通函数、lambda、仿函数、成员函数都能装；可被赋值、作容器元素、作回调参数。
- 代价：可能有堆分配与间接调用开销。

```cpp
std::function<int(int, int)> op;
op = [](int a, int b) { return a + b; };   // 装 lambda
op = std::multiplies<int>();               // 装仿函数
```

### 4.8 `auto` 会推导出引用/const 吗？

- `auto` 默认**剥掉引用与顶层 const**：`const int& r = x; auto a = r;` → `a` 是 `int`（拷贝）。
- 想保留引用/const 用 `auto&` / `const auto&`。
- 推断规则与模板参数推导一致（C++14 起函数返回类型也可 `auto`，但推导发生在调用时）。

### 4.9 `std::optional`、`std::variant`、`std::any` 区别？

- `optional<T>`：**有值 or 无值**（替代 "返回值 + 是否成功标志"）；`*o` / `o.value()` / `o.has_value()`。
- `variant<A,B,C>`：**编译期固定的类型安全 union**，同一时刻存其中一种；`std::get<T>` / `std::visit`。
- `any`：**运行期任意类型**（内部可能堆分配），`any_cast<T>` 取出，类型不符抛异常。

```cpp
std::optional<int> o = parse("42");
if (o) { /* 有值 */ }
std::variant<int, std::string> v = std::string("hi");
auto s = std::get<std::string>(v);
```

### 4.10 `override` 与 `final` 的作用？

- `override`：显式声明"重写基类虚函数"，签名不匹配时**编译报错**（防手滑写错签名变成隐藏）。
- `final`：类上加 `final` 禁止被继承；虚函数上加 `final` 禁止派生类重写。

```cpp
struct Base { virtual void f(int); virtual void g(); };

struct Derived : Base {
    void f(int) override;      // OK
    // void f(double) override;  // 编译错误：没有可重写的基类版本
    void g() final;            // 孙辈不能再重写 g
};
struct Grand final : Derived { // final 类：到此为止，不许再继承
    // void g() override;      // 编译错误：g 已被 final
};
// struct X : Grand {};        // 编译错误：Grand 是 final 类
```

### 4.11 什么是 RVO / NRVO？返回值是怎么"省掉"拷贝的？

**考察点**：拷贝省略（copy elision），移动语义之外的又一性能关键点。

**回答要点**：

- **RVO（Return Value Optimization）**：返回**临时对象** `return T();` 时，直接在调用方预留的空间里构造，**一次拷贝/移动都没有**。
- **NRVO（Named RVO）**：返回**具名局部变量** `T t; return t;`，编译器可把 `t` 直接建在返回值位置，同样省掉拷贝/移动。
- C++17 起 **RVO 是强制的**（prvalue 直接初始化目标对象，语言层面取消拷贝）；**NRVO 仍是允许的优化**（非强制，但所有主流编译器都做）。
- 结论：**`return std::move(local);` 是反优化**——会把本该省略的构造变成一次真实移动，还可能抑制 NRVO。
- 与 4.5 的关系：容器扩容依赖 `noexcept` 移动；返回值优化则连移动都不需要。

**Demo：无拷贝无移动的返回**

```cpp
#include <iostream>
using namespace std;

struct Tracker {
    Tracker()                 { cout << "Tracker()\n"; }
    Tracker(const Tracker&)   { cout << "Tracker(copy)\n"; }
    Tracker(Tracker&&) noexcept { cout << "Tracker(move)\n"; }
};

Tracker makeURVO() { return Tracker(); }        // RVO：连构造都只有一次
Tracker makeNRVO() { Tracker t; return t; }     // NRVO：t 直接建在返回值槽位

int main() {
    cout << "makeURVO():\n";  makeURVO();       // 只打印 Tracker()
    cout << "makeNRVO():\n";  makeNRVO();       // 只打印 Tracker()（无 copy/move）
    // 对比：若写 return std::move(t); 则会看到 Tracker(move)
}
```

```text
makeURVO():
Tracker()
makeNRVO():
Tracker()
```

### 4.12 引用折叠是什么？完美转发为什么能"保持左右值"？

**考察点**：转发引用 `T&&` 的推导规则，4.4 完美转发的底层原理。

**回答要点**：

- **引用折叠**规则（4 种，只有 `& &&` 折叠为 `&`）：
  - `T& &` → `T&`；`T& &&` → `T&`；`T&& &` → `T&`；`T&& &&` → `T&&`。
  - 口诀：**只要出现左值引用，结果就是左值引用**。
- **转发引用**（又称万能引用）：模板参数中的 `T&&` 不是右值引用，而是转发引用：
  - 传**左值**实参 → `T` 推导为 `U&`，`T&&` 折叠为 `U&` → 形参是左值引用；
  - 传**右值**实参 → `T` 推导为 `U`，`T&&` 即 `U&&` → 形参是右值引用。
- 但**形参本身是左值**（有名字就是左值），所以要 `std::forward<T>(x)` 才能还原原始值类别；`std::forward` 本质是**有条件**的 `static_cast<T&&>`。
- 区分：`std::move` 无条件转右值；`std::forward` 按 `T` 保守还原；`auto&&` 也是转发引用。

**Demo：一个模板函数透明转发左值/右值**

```cpp
#include <iostream>
using namespace std;

void classify(int&)  { cout << "  lvalue -> int&\n"; }
void classify(int&&) { cout << "  rvalue -> int&&\n"; }

template <typename T>
void pass(T&& x) {                 // T&&：转发引用
    // 若直接写 classify(x) → 永远命中 int&（x 本身是左值）
    classify(std::forward<T>(x));  // 还原原始值类别
}

int main() {
    int v = 1;
    pass(v);      // 左值 → lvalue -> int&
    pass(2);      // 右值 → rvalue -> int&&
}
```

### 4.13 `string_view`、结构化绑定、`constexpr` 家族？

**考察点**：C++17 / C++20 常用特性，考察是否跟进现代写法。

**回答要点**：

- **`std::string_view`（C++17）**：**非拥有**的字符串视图（指针 + 长度），构造/切分**零拷贝**；代价是**不保证以 `\0` 结尾**，且**不管理生命周期**（悬垂风险自担），适合只读参数。
- **结构化绑定（C++17）**：`auto [k, v] = *it;` / `for (const auto& [k, v] : m)`，本质是绑定到成员/元素的引用，可读性大幅提升；配合 `if` 初始化语句 `if (auto it = m.find(k); it != m.end())` 更好用。
- **`constexpr` 家族**：
  - `constexpr`（C++11）：**可用于编译期求值**（也允许运行期）；
  - `consteval`（C++20）：**必须**编译期求值（立即函数）；
  - `constinit`（C++20）：变量**必须**静态初始化（解决静态初始化顺序问题，变量本身不是 const）。
- 关联点：`constexpr` 与内联、宏的对比见 1.6；`constexpr` 函数可用于 `static_assert`、数组长度、模板非类型参数。

**Demo：三件套各一例**

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <map>
using namespace std;

constexpr int square(int x) { return x * x; }        // 编译期可算

int main() {
    // 1) string_view：零拷贝切片
    string_view sv = "hello world";
    cout << sv.substr(0, 5) << '\n';                  // hello（无堆分配）
    constexpr size_t n = 7;
    cout << sv.substr(0, n) << '\n';

    // 2) 结构化绑定：map 遍历 & if 初始化语句
    map<string, int> m{{"a", 1}, {"b", 2}};
    for (const auto& [k, v] : m) cout << k << '=' << v << ' ';
    cout << '\n';
    if (auto it = m.find("a"); it != m.end())
        cout << "found a=" << it->second << '\n';

    // 3) constexpr：编译期常量
    static_assert(square(5) == 25);                   // 编译期断言通过
    constexpr int arr[square(3)] = {};                // 编译期长度 = 9
    cout << "arr size = " << sizeof(arr) / sizeof(arr[0]) << '\n';   // 9
}
```

```text
hello
hello w
a=1 b=2
found a=1
arr size = 9
```

## 五、STL 容器与算法

### 5.1 `vector` 的扩容机制？

- 内存**连续**，尾部增删 O(1)，中间插入 O(n)。
- 容量不足时按**倍数扩容**（GCC 约 2 倍、MSVC 约 1.5 倍），分配新内存 → 移动/拷贝旧元素 → 释放旧内存。
- 均摊复杂度：插入 n 次约 O(1)。
- **迭代器/指针/引用在扩容后全部失效**。
- 技巧：已知规模用 `reserve()` 预分配，避免多次扩容；`shrink_to_fit()` 归还多余内存。

```cpp
std::vector<int> v;
v.reserve(1000);       // 预分配，后面 push_back 不触发扩容
v.push_back(1);
```

### 5.2 `map` 与 `unordered_map` 的区别？怎么选？

| 维度 | map | unordered_map |
| ---- | --- | ------------- |
| 底层 | **红黑树**（有序） | **哈希表**（桶 + 拉链/开放寻址） |
| 有序性 | 按键排序 | 无序 |
| 查询 | O(log n) | 平均 O(1)，最坏 O(n)（冲突） |
| 插入/删除 | O(log n) | 平均 O(1)；rehash 时可能整体重建 |
| 内存 | 节点小但分散 | 桶数组 + 节点，可能更高 |
| 需要 | 有序遍历、范围查询 | 大量查找、不在意顺序 |
| 自定义 key | 需 `operator<` | 需 hash 函数 + `operator==` |

> 面试加分点：数据量小或字符串 key 时，unordered_map 未必更快（hash 计算开销）；对实时性要求高且怕 hash 攻击时 map 更稳。

### 5.3 `vector` 与 `list` 的区别？

- `vector`：连续内存、随机访问 O(1)、尾插 O(1)、中间插入 O(n)、缓存友好。
- `list`：**双向链表**、任意位置插入删除 O(1)（已知迭代器）、无随机访问 O(n)、节点分散缓存不友好、每个节点多 2 个指针开销。
- 现代建议：**默认 vector**；只有频繁在头部/中部插入删除且不随机访问时才用 list。
- C++11 后 list 插入不使迭代器失效，vector 扩容会使迭代器失效（区别于 list）。

### 5.4 迭代器失效的场景？

- `vector`：插入/删除（尤其**扩容**）使指向其后元素的迭代器失效；`erase` 返回下一个有效迭代器。
- `deque`：中间插入使所有迭代器失效；两端操作只使指向被删元素的失效。
- `list/map/set/unordered_*`：删除的**那个迭代器**失效，其它不受影响（链表/树节点独立）。
- 遍历删除的正确姿势：

```cpp
// vector：利用 erase 返回值
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) it = v.erase(it);
    else ++it;
}
// C++20 更简单：std::erase_if(v, pred);
```

### 5.5 `deque` 的实现与特点？

- **双端队列**：分段连续内存（map 指针数组 + 若干连续缓冲区）。
- 头尾插入删除 O(1)，随机访问 O(1)（多一次间接），无扩容"整块搬移"问题。
- 内存不整体连续 → `vector` 的"数据指针 + size"式用法不适用。

### 5.6 `std::sort` 用什么算法实现？

- 经典实现为 **IntroSort（内省排序）**：快速排序为主，递归深度超阈值（约 2log n）时切到**堆排序**（防最坏 O(n²)），元素少于阈值（如 16）用**插入排序**（小规模最快）。
- `std::stable_sort` 用归并排序，保证相等元素相对顺序。
- `std::sort` 不保证稳定；**比较函数必须满足严格弱序**（`a < b` 写法，不能 `a <= b`，否则未定义行为）。

### 5.7 `resize` 与 `reserve` 的区别？

- `reserve(n)`：只预留**容量**，不改变 size，不构造元素。
- `resize(n)`：改变**元素个数**，多出的元素会默认构造（或指定值）。
- 读下标 `v[i]` 要求 i < size；`capacity` 决定何时扩容。

### 5.8 `emplace_back` 与 `push_back` 的区别？

- `push_back(x)`：传入**已构造对象**，可能多一次移动/拷贝（构造临时量再移入）。
- `emplace_back(args...)`：直接在容器内存里**就地构造**，用参数转发给构造函数，**省一次移动**。

```cpp
v.push_back(std::string(10, 'a'));   // 构造 string → 移入（可能）
v.emplace_back(10, 'a');             // 直接用 (10,'a) 就地构造，无临时对象
```

### 5.9 STL 六大组件？

- **容器**（vector/list/map...）
- **算法**（sort/find/for_each...）
- **迭代器**（连接容器与算法，五种：输入/输出/前向/双向/随机访问）
- **仿函数/函数对象**（可调用对象，如 greater<int>）
- **适配器**（stack/queue/priority_queue 容器适配器；back_inserter 等）
- **空间配置器 allocator**（管理内存分配）

### 5.10 `string` 的 COW / SSO 了解吗？

- 旧实现曾用 **COW（写时复制）**：多个 string 共享字符缓冲，写时才复制 → 多线程下性能差、易踩坑（已被废弃）。
- 现代实现多用 **SSO（小字符串优化）**：长度 ≤ 15（GCC）时直接存在对象内部栈区，**零堆分配**；超过才堆分配。
- 这也是"为什么 `sizeof(std::string)` 是 32 而不是指针大小"的原因。

### 5.11 `map::operator[]` 有什么陷阱？

**考察点**：接口副作用，实际写代码最容易踩的坑之一。

**回答要点**：

- `map::operator[](key)` **找不到时会插入** `{key, T{}}`（值默认构造），**size 悄悄变大**；`unordered_map` 同理。
- 两个后果：**语义被改**（只想查询却改了容器）、**编译不过**（`T` 无默认构造函数时 `operator[]` 不可用）。
- **只读查询的正确姿势**：
  - `m.at(key)`——找不到抛 `std::out_of_range`；
  - `m.find(key)` / `m.count(key)`——先判存在再取值；
  - C++20 起可用 `m.contains(key)`；
  - `const map` 上**不能**用 `operator[]`（因为它可能插入），只有 5.10 里的只读接口可用。
- 加分点：`m[k]` 的插入语义有时正是想要的（计数器 `++count[word]` 很优雅），关键是**知道**它的行为。

**Demo：一次"只想读"的操作把容器改大了**

```cpp
#include <iostream>
#include <map>
#include <string>
using namespace std;

int main() {
    map<string, int> m;
    cout << "初始 size=" << m.size() << '\n';          // 0

    int v = m["missing"];          // 只是想读，却插入 {missing, 0}
    cout << "读 m[\"missing\"] 后 size=" << m.size()
         << " 值=" << v << '\n';                       // 1 0

    // 正确姿势：find / count / contains 不会改容器
    cout << "find(\"nope\") == end ? "
         << (m.find("nope") == m.end()) << '\n';       // 1 (true)
    cout << "最终 size=" << m.size() << '\n';          // 仍是 1

    // 常见误用：函数里只读地传 const map& 时想用 [] → 编译错误
    // 计数器场景 [] 反而方便：++count[word];
}
```

```text
初始 size=0
读 m["missing"] 后 size=1 值=0
find("nope") == end ? 1
最终 size=1
```

### 5.12 什么是 erase-remove 惯用法？

**考察点**：STL 算法与容器成员的职责分工，5.4 迭代器失效的延伸。

**回答要点**：

- **为什么不能只用 `remove`**：`std::remove/remove_if` 是**算法**，只认识迭代器、不知道容器，它把保留的元素前移并返回**新的逻辑结尾**，但**不改 size**，尾部残留元素处于"已移动后的有效但无意义"状态。
- **惯用法**：`v.erase(std::remove_if(v.begin(), v.end(), pred), v.end());` —— 算法负责搬移，成员 `erase` 负责真正删除区间。
- 复杂度 O(n)；返回值就是新 end，配合 `erase` 一次搞定。
- 简化写法：**C++20** 提供 `std::erase_if(v, pred)` / `std::erase(v, val)`，一行搞定。
- 注意：`list` 有成员 `remove/remove_if`（直接删节点，O(n) 且不失效其它迭代器）；关联容器（`map/set`）不能用 remove（元素不可搬移），应遍历 `erase` 或 C++20 `erase_if`。

**Demo：删掉所有偶数**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> v{1, 2, 3, 4, 5, 6};

    // 错误示范：只调 remove 不改 size
    // auto newEnd = remove_if(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
    // v.size() 仍是 6，尾部是"移过来的垃圾值"

    v.erase(remove_if(v.begin(), v.end(),
                      [](int x) { return x % 2 == 0; }),
            v.end());

    for (int x : v) cout << x << ' ';      // 1 3 5
    cout << " (size=" << v.size() << ")\n"; // (size=3)

    // C++20 一行版：std::erase_if(v, [](int x){ return x % 2 == 0; });
}
```

### 5.13 `vector<bool>` 有什么坑？

**考察点**：模板特化带来的接口偏差，经典冷门题。

**回答要点**：

- `vector<bool>` 是**标准库的显式特化**，为省空间把每个 `bool` **压成 1 bit** 存储。
- 代价：`operator[]` 返回的是**代理对象** `vector<bool>::reference`（不是 `bool&`），于是：
  - **不能取地址**：`bool* p = &vb[0];` 编译错误；
  - `auto b = vb[0];` 拿到的是代理，不是 `bool`（写成 `bool b = vb[0];` 才安全）；
  - 依赖 `T&` 的泛型代码（模板、`std::swap` 的朴素写法等）可能编译失败或行为异常；
  - 不满足标准"容器"的全部要求。
- 结论：需要**真正的 `bool` 数组**语义时，用 `std::vector<char>`、`std::deque<bool>`，或 C++26 的 `std::vector<bool>` 替代方案；纯粹当位图用 `std::bitset<N>`（编译期长度）或 `boost::dynamic_bitset`。
- 加分点：与 5.1 呼应——`vector<bool>` 的"迭代器/引用失效"规则也和普通 vector 不同。

**Demo：代理类型露馅**

```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
    vector<bool> vb{true, false, true};

    bool b0 = vb[0];                       // 隐式转成真 bool：安全
    // bool* p = &vb[0];                   // 编译错误：取不到地址（返回代理对象）
    // bool& r = vb[0];                    // 编译错误：同理

    cout << boolalpha << "vb[0]=" << b0
         << " vb[1]=" << vb[1] << '\n';     // true false

    cout << "sizeof(bool)=" << sizeof(bool) << '\n';                        // 1
    cout << "sizeof(vector<bool>::reference)="
         << sizeof(vector<bool>::reference) << '\n';                        // 16（代理对象，含指针+位掩码）

    // 替代方案：需要真数组语义
    vector<char> bits{1, 0, 1};
    cout << "vector<char> 可取地址: " << static_cast<bool>(&bits[0] != nullptr) << '\n';
}
```

```text
vb[0]=true vb[1]=false
sizeof(bool)=1
sizeof(vector<bool>::reference)=16
vector<char> 可取地址: true
```

## 六、并发编程

### 6.1 线程安全单例怎么写？懒汉式 DCL 的问题？

- **饿汉式**：静态实例，启动即创建，天然线程安全；缺点是无论用不用都创建、初始化顺序不可控。

```cpp
class Singleton {
public:
    static Singleton& get() { static Singleton inst; return inst; }  // C++11 起标准保证线程安全
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
private:
    Singleton() = default;
};
```

- C++11 后**推荐**：函数内静态局部变量（标准保证首次初始化线程安全，MSVC/GCC 均用线程安全的魔法静态实现）。
- 老式 **DCL（双重检查锁）** 曾因指令重排有问题，需要 `volatile` 或 C++11 原子；现在手写 DCL 属过时做法（除非解释原理：外层无锁快路径 + 内层加锁 + 内存序）。

### 6.2 互斥锁的几种形式？lock_guard 与 unique_lock 区别？

- `std::mutex`：基础互斥，不能递归。
- `recursive_mutex`：同一线程可重复加锁（配合计数）。
- `timed_mutex`：带超时（try_lock_for/until）。
- `shared_mutex`：读写锁（shared 读 / unique 写）。
- `lock_guard`：**构造上锁、析构解锁**，简单场景首选，不可手动控制。
- `unique_lock`：更灵活，可延迟加锁、`unlock()` 手动解锁、与条件变量配合；代价是略大。

```cpp
std::mutex m;
{ std::lock_guard<std::mutex> lk(m); /* 临界区 */ }  // 自动解锁

std::unique_lock<std::mutex> lk(m);   // 条件变量必须用它
cv.wait(lk, [] { return ready; });    // 等待时自动释放锁
```

### 6.3 什么是死锁？如何避免？

- **定义**：两个及以上线程各自持锁等待对方释放 → 互相阻塞。
- **产生四条件**：互斥、占有且等待、不可剥夺、循环等待。
- **避免**：
  - 一次锁多个用 `std::lock(m1, m2)`（C++11 原子地多锁）。
  - 固定加锁顺序（如都先 A 后 B）。
  - 减少锁粒度、缩短持锁时间；避免持锁调用外部不可控代码。
  - 用 RAII（lock_guard）防异常路径漏解锁。

### 6.4 条件变量为什么必须配 unique_lock？为何要配合 while 判断？

- `wait` 需要原子地"释放锁 + 阻塞 + 被唤醒后重新拿锁"，`unique_lock` 支持这种操作。
- **虚假唤醒**（spurious wakeup）与竞态要求：被唤醒后必须用 `while (!pred())` **再检查条件**，而不是简单 `if`。
- 规范写法：

```cpp
std::mutex m;
std::condition_variable cv;
bool ready = false;

// 生产者：{ std::lock_guard lk(m); ready = true; } cv.notify_one();
// 消费者：
std::unique_lock<std::mutex> lk(m);
cv.wait(lk, [] { return ready; });   // 等价于 while(!ready) cv.wait(lk);
```

### 6.5 原子操作与 `volatile` 的区别？

- `std::atomic<T>`：保证读改写**原子性** + 内存序约束，可安全跨线程。
- `volatile`：只告诉编译器"别优化掉读写"，**不保证原子性**，也不提供线程间同步——C++ 里它用于访问内存映射 I/O 等硬件场景。
- 结论：多线程共享计数/标志用 `std::atomic`；`volatile` 不能替代互斥/原子。

```cpp
std::atomic<int> counter{0};
counter.fetch_add(1);         // 线程安全自增
```

### 6.6 线程间通信方式有哪些？

- 锁 + 条件变量（最常用）、`std::atomic` 标志、`std::future/promise`（一次性结果传递）、消息队列（生产者-消费者）、无锁结构（进阶）。
- 进程间（IPC）另说：管道、共享内存、Socket、信号。

### 6.7 `std::thread` 使用注意？

- 必须 `join()` 或 `detach()` 二选一，否则析构 `std::terminate`。
- 传参注意：默认拷贝进线程；用 `std::ref(x)` 传引用。
- 优先 `std::async`/线程池，避免频繁创建线程（创建开销大）。

```cpp
std::thread t([]{ do_work(); });
t.join();                       // 等它结束
// std::jthread（C++20）：析构自动 join，还支持 stop_token 协作取消
```

### 6.8 `std::atomic` 的内存序（memory_order）是什么？

**考察点**：原子性 ≠ 可见性/有序性，并发进阶必问。

**回答要点**：

- 原子操作默认 `memory_order_seq_cst`（顺序一致），最安全也最慢；其余顺序在**保证原子性**的前提下放松**重排限制**。
- 四种常用语义：
  - `relaxed`：只保证原子性，不保证与其他变量的可见顺序（适合纯计数器）；
  - `release`（写）/ `acquire`（读）：**配对**使用，建立 happens-before——`release` 之前的写，对读到该值的 `acquire` 线程**一定可见**；
  - `acq_rel`：读改写操作（如 `fetch_add`）同时具备两者；
  - `seq_cst`：全局唯一总顺序，多个原子变量之间也不许乱序。
- 要点：**没有同步关系的原子变量之间可以任意重排**，所以"标志位 + 数据"必须用 release/acquire 配对，否则数据可能读到旧值。
- 与 6.5 的关系：`volatile` 既不保证原子也不提供内存序，不能替代 `atomic`；`atomic` 解决"原子 + 可见性"，互斥锁解决"临界区"。

**Demo：release/acquire 保证数据可见（消息传递范式）**

```cpp
#include <iostream>
#include <atomic>
#include <thread>
using namespace std;

atomic<bool> ready{false};
int payload = 0;                       // 普通变量，靠 ready 的同步保护

int main() {
    thread producer([] {
        payload = 42;
        ready.store(true, memory_order_release);        // 保证 payload 的写先于 ready 可见
    });
    thread consumer([] {
        while (!ready.load(memory_order_acquire)) { }   // 看到 true ⇒ payload 一定已写入
        cout << "consumer 读到 payload=" << payload << '\n';
    });
    producer.join();
    consumer.join();
    // 输出：consumer 读到 payload=42
    // 若把两处都改成 memory_order_relaxed，理论上可能读到 payload=0（数据竞争）
}
```

### 6.9 `std::call_once` / `std::once_flag` 有什么用？

**考察点**：一次性初始化的标准做法，优于手写 DCL。

**回答要点**：

- 保证**多线程下某个函数只执行一次**，即使被多个线程同时调用；其余线程会阻塞等待首次执行完成。
- 相比手写 DCL（6.1）：无需自己处理内存序，语义清晰，异常安全——**首次执行抛异常时不会标记为已完成**，其他线程可重试。
- 典型用途：单例初始化、全局资源/配置懒加载、注册一次性回调。
- 与函数内静态局部变量（6.1 推荐写法）的关系：静态局部量本身就是"保证只初始化一次"，且更简洁；当初始化逻辑复杂或需要显式控制时机时用 `call_once`。
- C++17 前 `call_once` 可能依赖 `pthread_once`，现代实现为无锁快路径。

**Demo：两个线程竞争，init 只跑一次**

```cpp
#include <iostream>
#include <mutex>
#include <thread>
using namespace std;

once_flag flag;

int main() {
    auto init = [] { cout << "init 只执行一次\n"; };
    thread t1([&] { call_once(flag, init); });
    thread t2([&] { call_once(flag, init); });
    t1.join();
    t2.join();
    // 输出一次：init 只执行一次
}
```

### 6.10 手写一个简易线程池？

**考察点**：线程复用 + 任务队列 + 条件变量，并发编程的"综合大题"。

**回答要点**：

- **为什么需要**：`std::thread` 创建/销毁开销大（内核态、栈分配），高并发下频繁创建会拖垮性能；线程池**预创建固定数量线程 + 任务队列**复用。
- 组成：`vector<thread>`（worker）、`queue<function<void()>>`（任务）、`mutex` + `condition_variable`（同步）、`stop` 标志（优雅退出）。
- 核心逻辑：
  - worker 循环：`cv.wait(lk, pred)` 等任务 → 取出任务 → **解锁后**执行（避免持锁执行耗时任务）；
  - `submit`：加锁入队 → `notify_one`；
  - 析构：置 `stop` → `notify_all` → `join` 全部 worker（`jthread` 可自动 join）。
- 加分点：`wait` 的谓词必须包含 `stop || !tasks.empty()`（否则析构时可能永久阻塞）；任务队列空且 stop 时才退出；线程数一般设为 `hardware_concurrency()` 或 `2 * CPU + 1`（I/O 密集可更多）。

**Demo：3 个 worker 消费 6 个任务**

```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <atomic>
using namespace std;

class ThreadPool {
    vector<thread> workers_;
    queue<function<void()>> tasks_;
    mutex m_;
    condition_variable cv_;
    bool stop_ = false;
public:
    explicit ThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i) {
            workers_.emplace_back([this] {
                for (;;) {
                    function<void()> task;
                    {
                        unique_lock<mutex> lk(m_);
                        cv_.wait(lk, [this] { return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;   // 退出条件
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }                                        // 先解锁
                    task();                                  // 再执行任务
                }
            });
        }
    }
    template <typename F>
    void submit(F&& f) {
        { lock_guard<mutex> lk(m_); tasks_.emplace(std::forward<F>(f)); }
        cv_.notify_one();
    }
    ~ThreadPool() {
        { lock_guard<mutex> lk(m_); stop_ = true; }
        cv_.notify_all();
        for (auto& t : workers_) t.join();
    }
};

int main() {
    ThreadPool pool(3);
    atomic<int> done{0};
    for (int i = 0; i < 6; ++i)
        pool.submit([&done] {
            this_thread::sleep_for(chrono::milliseconds(10));
            ++done;
        });
    this_thread::sleep_for(chrono::milliseconds(100));
    cout << "完成 " << done << " 个任务\n";   // 完成 6 个任务
}   // 析构：置 stop → notify_all → join 全部 worker
```

## 七、手写代码（高频）

### 7.1 手写 `String` 类（浅拷贝陷阱 + 三大函数）

见 **2.1 深拷贝与浅拷贝** 的完整实现。要点：深拷贝、自赋值检查、`delete[]` 配对、C++11 还可加移动构造（`noexcept`）与拷贝交换惯用法：

```cpp
String& operator=(String o) {   // 传值拷贝 + 交换（copy-and-swap）
    swap(*this, o);             // 异常安全且天然处理自赋值
    return *this;
}
```

### 7.2 手写 `shared_ptr`（简化版）

```cpp
template <typename T>
class SharedPtr {
    T* ptr_;
    int* count_;                        // 引用计数
public:
    explicit SharedPtr(T* p = nullptr) : ptr_(p), count_(p ? new int(1) : nullptr) {}
    SharedPtr(const SharedPtr& o) : ptr_(o.ptr_), count_(o.count_) {
        if (count_) ++*count_;
    }
    ~SharedPtr() { release(); }
    SharedPtr& operator=(const SharedPtr& o) {   // copy-and-swap
        if (this != &o) { SharedPtr(o).swap(*this); }
        return *this;
    }
    T* get() const { return ptr_; }
    T& operator*() const { return *ptr_; }
private:
    void release() {
        if (count_ && --*count_ == 0) { delete ptr_; delete count_; }
    }
    void swap(SharedPtr& o) { std::swap(ptr_, o.ptr_); std::swap(count_, o.count_); }
};
```

要点：拷贝加计数、析构减计数到 0 才释放、赋值用 copy-and-swap 保证异常安全。

### 7.3 手写线程安全单例

见 **6.1 线程安全单例**——函数内静态局部变量版本最简洁可靠。

### 7.4 手写 `vector` 的 push_back 扩容

要点（伪码）：容量不足 → `capacity * 2` → 新分配 → 移动旧元素 → 释放旧内存 → 更新 size/capacity。追问常考：为什么 2 倍？——均摊 O(1)（1.5~2 倍之间取，2 倍最简单证明均摊常数）。

关键实现细节：扩容用的是**裸内存分配（`::operator new` / allocator）+ placement new 构造**，而不是 `new[]`（见 2.8、2.5）；旧元素**移动**构造到新址后要**手动析构**，再释放旧内存。

```cpp
#include <iostream>
#include <utility>
using namespace std;

template <typename T>
class MiniVector {
    T* data_ = nullptr;
    size_t size_ = 0, cap_ = 0;
public:
    ~MiniVector() { delete[] data_; }
    size_t size() const { return size_; }
    size_t capacity() const { return cap_; }
    void push_back(const T& v) {
        if (size_ == cap_) grow(cap_ ? cap_ * 2 : 1);   // 2 倍扩容
        new (data_ + size_) T(v);                        // 新位置构造
        ++size_;
    }
private:
    void grow(size_t newCap) {
        cout << "  扩容: " << cap_ << " -> " << newCap << '\n';
        T* np = static_cast<T*>(::operator new(newCap * sizeof(T)));  // 只分配裸内存
        for (size_t i = 0; i < size_; ++i) {
            new (np + i) T(std::move(data_[i]));   // 移动构造到新位置
            data_[i].~T();                          // 析构旧对象
        }
        ::operator delete(data_);                   // 释放旧内存
        data_ = np;
        cap_ = newCap;
    }
};

int main() {
    MiniVector<int> v;
    for (int i = 0; i < 9; ++i) {
        v.push_back(i);
        cout << "push " << i << " -> size=" << v.size() << " cap=" << v.capacity() << '\n';
    }
}
```

```text
  扩容: 0 -> 1
push 0 -> size=1 cap=1
  扩容: 1 -> 2
push 1 -> size=2 cap=2
  扩容: 2 -> 4
push 2 -> size=3 cap=4
push 3 -> size=4 cap=4
  扩容: 4 -> 8
push 4 -> size=5 cap=8
```

> 追问：若把 `push_back` 改成 `push_back(T&&)` + `emplace_back`，可省掉一次拷贝（见 5.8）；把 `grow` 里的移动换成拷贝，复杂度从均摊 O(1) 退化（这正是 4.5 要求移动构造 `noexcept` 的原因）。

### 7.5 判断链表是否有环 / 反转链表

```cpp
// 快慢指针判环：快指针每次 2 步、慢指针 1 步，相遇即有环
bool hasCycle(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}
```

```cpp
// 迭代反转：三指针 prev / cur / next，逐个改向
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) {
        ListNode* nxt = head->next;   // 先存后继，防止断链
        head->next = prev;            // 反转当前节点指向
        prev = head;                  // prev 前移
        head = nxt;                   // head 前移
    }
    return prev;                      // 新头
}
// 递归版本（面试追问）：
// ListNode* reverseList(ListNode* head) {
//     if (!head || !head->next) return head;
//     ListNode* nh = reverseList(head->next);
//     head->next->next = head;  head->next = nullptr;
//     return nh;
// }
```

```text
1 -> 2 -> 3  反转后  3 -> 2 -> 1
```

> 判环追问：找到环的**入口**——相遇后把一个指针放回头部，两指针同速前进，再次相遇点即入口。

### 7.6 用两个栈实现队列 / 两个队列实现栈

- 两栈实现队列：入队压 s1；出队时若 s2 空，把 s1 全部倒入 s2，再弹 s2 顶（**摊还 O(1)**）。
- 队列实现栈：入栈入队 q1；出栈时把前 n-1 个转到 q2，弹出 q1 剩余，交换 q1/q2。

### 7.7 查找内存泄漏 / 优化经验

- 见 **2.2 内存泄漏检测与避免**：Valgrind / ASan / 代码审查 + RAII 重构。
- 常见追问：`shared_ptr` 循环引用导致泄漏 → 用 `weak_ptr` 打破（见 4.1/4.2）。

## 八、复习路线小结

1. 语法层面把 `const`/`static`/指针引用/拷贝语义/`explicit` 吃透（最高频）。
2. 内存管理答出 RAII、三/五/零法则、野指针与悬垂、智能指针对比、`placement new`。
3. 多态原理要能**手画 vtable/vptr 图**并解释动态绑定；能讲清**对象切片**与虚析构。
4. 现代 C++ 把移动语义 + 完美转发（引用折叠）+ RVO 讲清楚是加分项。
5. STL 会画 vector 扩容过程、map 底层、迭代器失效表，知道 `map::operator[]` 副作用与 erase-remove。
6. 并发题会写**线程安全单例**与**条件变量范式**，能讲 `atomic` 内存序并手写线程池。
7. 手写题至少准备 String、shared_ptr、单例、MiniVector、链表题五件套。
