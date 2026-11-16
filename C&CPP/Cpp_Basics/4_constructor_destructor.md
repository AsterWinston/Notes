构造函数和析构函数是对象生命周期的两端，也是 C++ 把"资源管理"变成语言机制的地方。学不好这一块，写出来的类要么泄漏、要么双重释放、要么在拷贝时悄悄共享了不该共享的指针。本篇从初始化列表讲到 RAII 和 rule of three/five，帮你把对象的"生老病死"安排明白。

# 1. 构造函数

## 1.1 默认构造与重载

构造函数与类同名、无返回类型，可重载：

```cpp
class Point {
public:
    Point() : x_(0), y_(0) {}              // 默认构造
    Point(int x, int y) : x_(x), y_(y) {}  // 带参构造
private:
    int x_;
    int y_;
};

Point a;          // 调用默认构造
Point b{1, 2};    // 调用带参构造
```

★ **关键认知**：只要你自己写了任何构造函数，编译器就**不再生成默认构造函数**。如果还需要无参构造，必须显式写 `Point() = default;`。这是"类突然不能默认创建了"的最常见原因。

## 1.2 初始化列表

成员应在初始化列表里初始化，而不是在函数体里赋值：

```cpp
class Person {
public:
    Person(std::string name, int age)
        : name_(std::move(name)),   // 直接构造成员
          age_(age) {}
private:
    std::string name_;
    int age_;
};
```

函数体赋值的写法会先默认构造成员再赋值，多一次开销；初始化列表则直接构造成员。

★ **关键认知**：**成员初始化的顺序由声明顺序决定，不是初始化列表的书写顺序**。如果两者不一致，编译器可能给出 `-Wreorder` 警告，而且逻辑上后写的可能读到还没初始化的成员：

```cpp
class Bad {
public:
    Bad() : b_(0), a_(b_) {}   // 实际先初始化 a_，此时 b_ 还未初始化
private:
    int a_;
    int b_;
};
```

⚠ **关键认知**：以下成员**必须**用初始化列表，不能在函数体里赋值：`const` 成员、引用成员、没有默认构造函数的类类型成员。因为它们没有"先默认构造再赋值"这条路。

## 1.3 explicit 与委托构造

单参数构造函数会带来隐式转换：

```cpp
class Meters {
public:
    Meters(double v) : v_(v) {}
    double value() const { return v_; }
private:
    double v_;
};

void print(Meters m);
print(3.5);        // 隐式把 3.5 转成 Meters(3.5)，可能不是你的本意
```

加 `explicit` 禁止隐式转换：

```cpp
class Meters {
public:
    explicit Meters(double v) : v_(v) {}
    // ...
};

// print(3.5);            // 错误
print(Meters(3.5));       // 必须显式构造
```

★ **关键认知**：单参数构造函数默认加 `explicit`，除非你确实想要隐式转换（比如 `std::string` 从 `const char*` 构造这类约定俗成的场景）。

委托构造让一个构造函数调用另一个：

```cpp
class Config {
public:
    Config() : Config(8080) {}                 // 委托给下面的构造函数
    explicit Config(int port) : port_(port) {}
private:
    int port_;
};
```

# 2. 析构函数

## 2.1 何时调用

析构函数在对象销毁时自动调用：

```cpp
class File {
public:
    explicit File(const char* path) : fp_(std::fopen(path, "r")) {}
    ~File() { if (fp_) std::fclose(fp_); }   // 释放资源
private:
    std::FILE* fp_;
};
```

调用时机：

| 对象类型 | 析构时机 |
|----------|----------|
| 栈对象 | 离开作用域 |
| 堆对象 | `delete` 时 |
| 静态/全局对象 | 程序结束 |
| 临时对象 | 表达式结束 |

析构顺序与构造顺序相反：后构造的先析构。

```text
构造顺序：成员 → 自身
析构顺序：自身 → 成员（逆序）
```

## 2.2 析构函数与继承

如果类可能被继承并通过基类指针删除，析构函数必须是虚函数。否则只调用基类析构，派生类资源泄漏。这一点第 5 篇会详细展开，这里先记住结论。

⚠ **关键认知**：析构函数里不要抛出异常。栈展开过程中再抛异常会直接 `std::terminate`。析构函数默认是 `noexcept` 的，真要处理错误只能吞掉或记录日志。

# 3. 拷贝控制

## 3.1 拷贝构造

用同类型对象初始化新对象时调用：

```cpp
class Buffer {
public:
    explicit Buffer(std::size_t n) : data_(new int[n]), size_(n) {}
    Buffer(const Buffer& other)                 // 拷贝构造
        : data_(new int[other.size_]),          // 重新分配
          size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
    }
    ~Buffer() { delete[] data_; }
private:
    int* data_;
    std::size_t size_;
};

Buffer a(10);
Buffer b = a;      // 调用拷贝构造，b 有独立内存
```

## 3.2 拷贝赋值

给已存在的对象赋值时调用：

```cpp
Buffer& operator=(const Buffer& other) {
    if (this == &other) {          // 自赋值检查，必须做
        return *this;
    }
    int* new_data = new int[other.size_];   // 先分配新内存
    std::copy(other.data_, other.data_ + other.size_, new_data);
    delete[] data_;                          // 再释放旧内存
    data_ = new_data;
    size_ = other.size_;
    return *this;
}
```

★ **关键认知**：拷贝构造和拷贝赋值的区别在于**对象是否已经存在**。构造是"从零造一个"，赋值是"把已有对象改成另一个的样子"。所以赋值要负责释放旧资源，构造不需要。

⚠ **关键认知**：赋值运算符必须先分配新资源再释放旧资源，或者用 copy-and-swap。如果先 `delete[] data_` 再 `new`，一旦 `new` 抛异常，对象就处于半销毁状态。自赋值检查也不能省，`a = a` 在"先删后拷"的写法下会读到已释放内存。

## 3.3 浅拷贝与深拷贝

| 类型 | 行为 | 后果 |
|------|------|------|
| 浅拷贝 | 只复制指针值 | 两个对象共享同一块内存，双重释放 |
| 深拷贝 | 复制指针指向的内容 | 各自独立，安全但有开销 |

```cpp
// 编译器生成的默认拷贝构造就是浅拷贝
Buffer b = a;    // 若没写拷贝构造：b.data_ 和 a.data_ 指向同一块内存
```

```text
浅拷贝：
a.data_ ──┐
          ├──► 同一块堆内存
b.data_ ──┘

深拷贝：
a.data_ ────► 堆内存 A
b.data_ ────► 堆内存 B（内容相同）
```

⚠ **关键认知**：只要类里**直接持有需要释放的资源**（裸指针、文件句柄、socket），就必须自己写拷贝构造和赋值，否则默认浅拷贝会导致双重释放。更现代的做法是用 `std::vector`、`std::string`、智能指针持有资源，让它们替你处理拷贝。

# 4. RAII

## 4.1 核心思想

RAII（Resource Acquisition Is Initialization）把资源生命周期绑定到对象生命周期：

```text
构造成功 → 资源被获取
对象存活 → 资源可用
析构调用 → 资源被释放
```

只要对象能正常析构（包括异常栈展开时），资源就一定会释放。

```cpp
class MutexGuard {
public:
    explicit MutexGuard(std::mutex& m) : m_(m) { m_.lock(); }
    ~MutexGuard() { m_.unlock(); }
    MutexGuard(const MutexGuard&) = delete;             // 不许拷贝
    MutexGuard& operator=(const MutexGuard&) = delete;
private:
    std::mutex& m_;
};

void work() {
    std::mutex m;
    MutexGuard guard(m);      // 构造即加锁
    // ... 即使这里抛异常，guard 析构也会解锁
}
```

★ **关键认知**：RAII 是 C++ 资源管理的基石。`std::vector` 管内存、`std::string` 管字符、`std::unique_ptr` 管堆对象、`std::lock_guard` 管锁，全都是这个模式。理解了 RAII，就理解了为什么 C++ 很少需要 `finally`。

# 5. rule of three/five/zero

## 5.1 三条规则

| 规则 | 内容 | 适用标准 |
|------|------|----------|
| rule of three | 需要自定义析构、拷贝构造、拷贝赋值中的任意一个，通常三个都要 | C++98 |
| rule of five | 再加上移动构造、移动赋值 | C++11 |
| rule of zero | 不直接管理资源，五个都不写，交给成员对象 | 现代 C++ 首选 |

★ **关键认知**：**rule of zero 才是现代 C++ 的目标**。用 `std::vector`、`std::string`、`std::unique_ptr` 作为成员，编译器生成的五个特殊成员函数就都是正确的，你一个都不用写。手写这五个函数是最后手段，不是标配。

```cpp
// rule of zero：不写析构、拷贝、移动，全部正确
class Person {
private:
    std::string name_;
    std::vector<int> scores_;
    std::unique_ptr<Address> addr_;
};
```

⚠ **关键认知**：一旦你手写了析构函数，编译器就**不会自动生成移动构造和移动赋值**，拷贝操作仍会生成（但已废弃）。这会导致本可移动的场景退化成拷贝，性能悄悄下降。写了析构就考虑把五个都显式处理，或者干脆用 rule of zero。

# 6. 常见错误

| 现象 | 原因 | 修复 |
|------|------|------|
| 类无法默认构造 | 定义了其他构造函数 | 加 `= default` |
| 成员值不符合预期 | 初始化顺序与声明顺序不一致 | 调整声明顺序，开 `-Wreorder` |
| 双重释放 | 默认浅拷贝共享指针 | 深拷贝或禁用拷贝 |
| 自赋值崩溃 | 赋值运算符没检查 `this == &other` | 加自赋值检查或 copy-and-swap |
| 移动后源对象状态异常 | 移动后未置空源指针 | 移动后把源置为有效可析构状态 |
| const 成员赋值报错 | 在函数体里赋值 | 改用初始化列表 |

⚠ **关键认知**：移动构造和移动赋值应标 `noexcept`。否则 `std::vector` 在扩容时不敢用移动，只能退回复制，性能大打折扣。移动操作本身不该失败，标 `noexcept` 是承诺也是优化开关。

# 7. 动手练习

1. 写一个管理 `int*` 数组的类，实现构造、析构、拷贝构造、拷贝赋值，并测试自赋值。
2. 用 rule of zero 重写上面的类，成员改成 `std::vector<int>`，确认不再需要手写任何特殊成员函数。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 初始化列表 | 成员真正初始化的地方，顺序按声明 |
| `explicit` | 禁止单参数构造的隐式转换 |
| 析构 | 逆序释放，不应抛异常 |
| 拷贝构造 | 用已有对象造新对象 |
| 拷贝赋值 | 改已存在对象，需处理自赋值与旧资源 |
| 深浅拷贝 | 持有资源就要深拷贝或禁用拷贝 |
| RAII | 资源生命周期绑定对象生命周期 |
| rule of zero | 用 RAII 成员，五个特殊函数都不写 |

下一篇进入继承与多态，讲清楚虚函数表、`override`/`final`、抽象类、虚析构的必要性，以及对象切片这个经典坑。
