# C++ 面试题

> C++ 高频面试题整理，按 基础语法 → 内存管理 → 面向对象 → 现代 C++ → STL → 并发 → 手写代码 分类。每题给出考察点与回答要点，考前快速过一遍即可。

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

### 7.6 用两个栈实现队列 / 两个队列实现栈

- 两栈实现队列：入队压 s1；出队时若 s2 空，把 s1 全部倒入 s2，再弹 s2 顶（**摊还 O(1)**）。
- 队列实现栈：入栈入队 q1；出栈时把前 n-1 个转到 q2，弹出 q1 剩余，交换 q1/q2。

### 7.7 查找内存泄漏 / 优化经验

- 见 **2.2 内存泄漏检测与避免**：Valgrind / ASan / 代码审查 + RAII 重构。
- 常见追问：`shared_ptr` 循环引用导致泄漏 → 用 `weak_ptr` 打破（见 4.1/4.2）。

## 八、复习路线小结

1. 语法层面把 `const`/`static`/指针引用/拷贝语义吃透（最高频）。
2. 多态原理要能**手画 vtable/vptr 图**并解释动态绑定。
3. 内存管理答出 RAII、三/五法则、智能指针对比。
4. 现代 C++ 把移动语义 + 完美转发讲清楚是加分项。
5. STL 会画 vector 扩容过程、map 底层、迭代器失效表。
6. 并发题会写**线程安全单例**与**条件变量范式**。
7. 手写题至少准备 String、shared_ptr、单例、链表题四件套。
