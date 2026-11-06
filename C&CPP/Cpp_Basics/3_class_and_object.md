类是 C++ 把"数据"和"操作数据的代码"绑在一起的机制。很多人学完语法后仍然把类当成"带函数的结构体"，结果封装形同虚设、`static` 用错、对象什么时候被创建和销毁也说不清。本篇从访问控制讲到对象内存布局，把"一个对象到底是什么"这件事讲清楚。

# 1. 类与对象基础

## 1.1 从结构体到类

```cpp
struct Point {          // struct 默认成员 public
    int x;
    int y;
};

class Point2 {          // class 默认成员 private
    int x;
    int y;
};
```

`struct` 和 `class` 在 C++ 里唯一的语法差别是**默认访问权限**：`struct` 默认 `public`，`class` 默认 `private`。习惯上，纯数据聚合用 `struct`，有不变式和行为的抽象用 `class`。

对象是类的实例：

```cpp
Point p{1, 2};          // p 是一个对象，占用两块 int 的空间
Point q = p;            // 拷贝，q 有自己的 x、y
```

★ **关键认知**：类定义本身**不占运行时空间**，它只是编译期的蓝图。只有创建对象时才分配内存，每个对象各自持有一份非静态成员。

## 1.2 访问控制

| 关键字 | 谁能访问 | 典型用途 |
|--------|----------|----------|
| `public` | 任何人 | 对外接口 |
| `protected` | 本类与派生类 | 给子类用的实现细节 |
| `private` | 仅本类（及友元） | 内部状态、不变式 |

```cpp
class BankAccount {
public:
    void deposit(double amount) {
        if (amount > 0) {          // 通过接口维护不变式
            balance_ += amount;
        }
    }
    double balance() const { return balance_; }

private:
    double balance_ = 0.0;         // 外部无法直接改，只能走 deposit
};
```

★ **关键认知**：封装的本质不是"加个 getter/setter"，而是**让所有修改都经过能维护不变式的入口**。如果每个成员都配一对无脑的 get/set，那和 public 没有区别，只是多了噪音。

⚠ **关键认知**：`private` 是编译期检查，不是安全边界。用指针偏移、`reinterpret_cast` 仍然能碰到私有成员，但那属于未定义行为。不要用"能绕过"来否定封装的价值。

## 1.3 成员函数

成员函数可以直接访问成员，不需要显式传对象：

```cpp
class Counter {
public:
    void inc() { ++count_; }              // 隐式使用 this->count_
    int value() const { return count_; }
private:
    int count_ = 0;
};

Counter c;
c.inc();                                  // 编译器把 &c 作为隐藏参数传入
```

成员函数在类内定义时默认是 `inline` 的；也可以只在类内声明，类外定义：

```cpp
class Counter {
public:
    void inc();
    int value() const;
private:
    int count_ = 0;
};

void Counter::inc() { ++count_; }         // 类外定义要带类名前缀
int Counter::value() const { return count_; }
```

# 2. this 指针

## 2.1 this 是什么

在每个**非静态成员函数**里，`this` 是一个指向当前对象的指针：

```cpp
class Widget {
public:
    void set(int v) {
        this->value = v;      // 显式写 this，和直接写 value 等价
    }
    Widget& self() {
        return *this;         // 返回当前对象的引用
    }
private:
    int value = 0;
};
```

`this` 的类型：

| 函数形式 | this 的类型 |
|----------|-------------|
| 普通成员函数 | `Widget*` |
| const 成员函数 | `const Widget*` |
| 静态成员函数 | 没有 this |

★ **关键认知**：`this` 是成员函数的一个**隐藏参数**。调用 `c.inc()` 时，编译器实际做的是把 `&c` 传进去。这也解释了为什么静态成员函数里没有 `this`——它不属于任何对象。

## 2.2 返回 *this 实现链式调用

```cpp
class Builder {
public:
    Builder& name(std::string n) {
        name_ = std::move(n);
        return *this;         // 返回引用，避免拷贝
    }
    Builder& age(int a) {
        age_ = a;
        return *this;
    }
private:
    std::string name_;
    int age_ = 0;
};

Builder b;
b.name("Tom").age(18);        // 链式调用
```

⚠ **关键认知**：链式调用返回 `*this` 的**引用**而不是值，否则每次都会拷贝一个临时对象，且后续修改落在临时对象上。返回引用的前提是对象本身还活着，不要对局部对象这么写。

# 3. static 成员

## 3.1 静态成员变量

静态成员属于**类**，不属于某个对象，所有对象共享一份：

```cpp
class Ticket {
public:
    Ticket() { ++created_; }
    static int created() { return created_; }
private:
    static int created_;      // 类内声明
};

int Ticket::created_ = 0;    // 类外定义并初始化（C++17 前必须这样写）
```

C++17 起可以用 `inline` 直接在类内定义：

```cpp
class Ticket {
public:
    static inline int created = 0;   // C++17：类内定义，无需类外再写
};
```

★ **关键认知**：静态成员变量**不占对象空间**，它存放在静态存储区，生命周期从程序启动到结束。`sizeof(Ticket)` 不会因为多了个 `static int` 而增大。

## 3.2 静态成员函数

静态成员函数没有 `this`，只能访问静态成员：

```cpp
class Config {
public:
    static Config& instance() {      // 单例的常见写法
        static Config c;             // 函数内静态对象，首次调用时构造
        return c;
    }
    void setDebug(bool v) { debug_ = v; }

private:
    Config() = default;              // 构造函数私有，外部不能直接创建
    bool debug_ = false;
};

Config::instance().setDebug(true);
```

⚠ **关键认知**：函数内 `static` 局部变量的初始化在 C++11 起是**线程安全**的（magic statics），这是 Meyers 单例能成立的前提。但它只保证初始化安全，不保证对象后续的并发访问安全。

# 4. 对象生命周期

## 4.1 创建与销毁

一个对象的完整生命周期：

```text
分配内存
   │
   ▼
调用构造函数（初始化成员）
   │
   ▼
对象可用（调用成员函数）
   │
   ▼
调用析构函数（释放资源）
   │
   ▼
回收内存
```

```cpp
{
    Point p{1, 2};       // 进入作用域：分配 + 构造
    p.x = 3;             // 使用
}                        // 离开作用域：析构 + 回收
```

## 4.2 三种存储位置

| 存储区 | 创建方式 | 生命周期 | 典型场景 |
|--------|----------|----------|----------|
| 栈 | 局部变量 | 作用域结束自动销毁 | 绝大多数局部对象 |
| 堆 | `new` / `make_unique` | 手动或智能指针管理 | 需要超出作用域存活 |
| 静态区 | 全局、`static` | 程序运行期间 | 全局配置、单例 |

```cpp
void demo() {
    Point a{1, 2};                    // 栈
    Point* b = new Point{3, 4};       // 堆，需 delete
    static Point c{5, 6};             // 静态区，只构造一次
    delete b;                         // 不 delete 就泄漏
}
```

★ **关键认知**：**优先用栈对象**。栈对象随作用域自动析构，天然满足 RAII，不会泄漏。只有需要"活得比作用域长"或"大小在运行期决定"时才上堆，并且用智能指针管理（第 10 篇）。

## 4.3 对象内存布局

以只有一个 `int` 和一个 `double` 的类为例：

```text
class Sample {
    int    a;      // 偏移 0
    double b;      // 偏移 8（对齐到 8 字节）
};
```

```text
Sample 对象（16 字节）
┌──────────────┬──────────────┐
│   a (4 字节)  │  填充 (4 字节) │
├──────────────┴──────────────┤
│        b (8 字节)            │
└─────────────────────────────┘
```

成员按声明顺序排列，但可能插入**填充字节**以满足对齐要求。这就是为什么 `sizeof` 常常大于成员大小之和，也是为什么调整成员顺序有时能减小对象体积。

⚠ **关键认知**：空类 `sizeof` 是 1 而不是 0，因为标准要求每个对象有唯一地址。含虚函数的类还会多一个**虚表指针**（通常 8 字节），这部分第 5 篇讲。

# 5. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| `private member is inaccessible` | 外部直接访问私有成员 | 提供公开接口 |
| 静态成员链接错误 | 只声明未定义 | 类外定义，或 C++17 用 `inline` |
| 修改一个对象的静态成员影响所有对象 | 静态成员本就共享 | 需要各自一份就改成非静态 |
| `delete` 后继续使用 | 悬垂指针 | 置空或改用智能指针 |
| 对象大小比预期大 | 内存对齐填充 | 调整成员顺序，按大小降序排 |
| 忘记 `const` 导致 const 对象无法调用 | 成员函数没标 const | 只读函数加 `const` |

⚠ **关键认知**：`sizeof` 的结果受编译器和平台影响，不要把它写死在协议或序列化代码里。跨平台传输布局，应该显式序列化字段，而不是 `memcpy` 整个对象。

# 6. 动手练习

1. 设计一个 `Rectangle` 类，成员私有，提供 `area()` 和 `scale(double)`，验证外部无法直接改宽高。
2. 给类加一个静态计数器，统计创建了多少个对象，分别用类外定义和 C++17 `inline` 两种写法实现。
3. 写一个链式调用的 `QueryBuilder`，支持 `.where(...).orderBy(...).limit(...)`，思考返回引用和返回值的行为差异。

# 7. 小结

| 概念 | 要点 |
|------|------|
| class/struct | 默认访问权限不同，其余等价 |
| 封装 | 所有修改经过能维护不变式的接口 |
| this | 非静态成员函数的隐藏参数 |
| 静态成员 | 属于类，所有对象共享，不占对象空间 |
| 栈对象 | 自动析构，优先使用 |
| 堆对象 | 需手动或智能指针管理 |
| 对象布局 | 按声明顺序 + 对齐填充 |

下一篇进入构造与析构，讲清楚初始化列表为什么必须用、拷贝构造和赋值运算符的区别、RAII 如何把资源绑到对象生命周期上，以及 rule of three/five。
