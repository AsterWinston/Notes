运算符重载让自定义类型用起来像内建类型：`a + b`、`cout << x`、`v[i]` 都能按你的语义工作。但它也是最容易被滥用的特性——把 `+` 定义成删除、把 `==` 和 `!=` 写得互相矛盾，都会让代码变得难以理解。本篇讲清楚哪些能重载、该写成成员还是非成员，以及流运算符、比较运算符、下标运算符的实现套路。

# 1. 基本规则

## 1.1 哪些能重载

大部分运算符都能重载，少数不能：

| 可以重载 | 不能重载 |
|----------|----------|
| `+ - * / %` | `::` 作用域解析 |
| `== != < > <= >=` | `.` 成员访问 |
| `[] () ->` | `.*` 成员指针访问 |
| `<< >> & \| ^ ~ !` | `?:` 三目 |
| `= += -= *= /=` | `sizeof` |
| `++ --` | `alignof` |
| `new delete` | `typeid` |

重载的限制：

- 不能改变运算符的**优先级**和**结合性**。
- 不能改变**操作数个数**（一元仍是一元）。
- 至少要有一个操作数是**用户自定义类型**，不能重载两个 `int` 的 `+`。

★ **关键认知**：运算符重载的本质是**函数调用**。`a + b` 就是 `operator+(a, b)` 的语法糖。理解这一点，就能明白为什么参数个数、返回值、const 修饰都按普通函数规则来。

```text
a + b      ──►  operator+(a, b)
a += b     ──►  a.operator+=(b)
v[i]       ──►  v.operator[](i)
os << p    ──►  operator<<(os, p)
```

## 1.2 成员还是非成员

| 运算符 | 推荐形式 | 原因 |
|--------|----------|------|
| `=` `[]` `()` `->` | 必须是成员 | 语言规定 |
| `+= -= *= /=` | 成员 | 修改左操作数 |
| `++ --` | 成员 | 修改自身 |
| 一元 `-` `+` `!` | 成员 | 操作自身 |
| 二元 `+ - * /` | 非成员（友元） | 允许左操作数隐式转换 |
| `== != <` | 非成员（友元） | 两侧对称 |
| `<< >>` 流 | 非成员 | 左操作数是 `ostream` |

```cpp
class Vec {
public:
    Vec operator+(const Vec& rhs) const;   // 成员：左操作数必须是 Vec
};

Vec operator+(const Vec& lhs, const Vec& rhs);  // 非成员：两侧都能隐式转换
```

★ **关键认知**：二元运算符优先写成**非成员**。成员版本要求左操作数已经是该类对象，`1 + vec` 这种写法就无法工作；非成员版本允许编译器对两个参数分别做隐式转换，对称性更好。

# 2. 算术与复合赋值

```cpp
class Vec2 {
public:
    Vec2(double x, double y) : x_(x), y_(y) {}

    Vec2& operator+=(const Vec2& rhs) {     // 复合赋值：返回引用
        x_ += rhs.x_;
        y_ += rhs.y_;
        return *this;
    }

    double x() const { return x_; }
    double y() const { return y_; }

private:
    double x_;
    double y_;
};

// 非成员：用 += 实现 +，避免重复逻辑
Vec2 operator+(Vec2 lhs, const Vec2& rhs) {
    lhs += rhs;                             // lhs 是副本，可以安全修改
    return lhs;                             // 返回值，不能返回引用
}
```

★ **关键认知**：`+` 返回**新值**，`+=` 返回**引用**。`+` 如果返回引用，要么返回局部对象（悬垂），要么修改了某个操作数（违反直觉）。用"按值传左操作数 + 复用 `+=`"是标准写法，既简洁又天然支持移动。

⚠ **关键认知**：不要为每个运算符都手写一遍逻辑。先实现 `+=`，再用它实现 `+`；先实现 `==` 和 `<`，其余比较用它们组合。逻辑只有一份，才不会出现 `a == b` 为真但 `a != b` 也为真的诡异情况。

# 3. 流运算符

```cpp
class Point {
public:
    Point(int x, int y) : x_(x), y_(y) {}

    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        return os << '(' << p.x_ << ", " << p.y_ << ')';
    }

    friend std::istream& operator>>(std::istream& is, Point& p) {
        char lparen, comma, rparen;
        if (is >> lparen >> p.x_ >> comma >> p.y_ >> rparen) {
            // 读取成功，p 已被更新
        }
        return is;
    }

private:
    int x_;
    int y_;
};

Point p(1, 2);
std::cout << p << '\n';        // 输出 (1, 2)
```

★ **关键认知**：流运算符**必须是非成员函数**，因为左操作数是 `std::ostream`，你没法往标准库里加成员函数。声明为 `friend` 是为了访问私有成员；如果能用公开接口，就不用 `friend`。

⚠ **关键认知**：`operator<<` 必须返回 `os` 的引用，否则 `cout << a << b` 无法链式调用。`operator>>` 必须接收非 const 引用，因为要写入对象。这两个签名几乎不该有别的写法。

# 4. 比较运算符

C++20 之前要成套手写：

```cpp
class Version {
public:
    Version(int major, int minor) : major_(major), minor_(minor) {}

    friend bool operator==(const Version& a, const Version& b) {
        return a.major_ == b.major_ && a.minor_ == b.minor_;
    }

    friend bool operator!=(const Version& a, const Version& b) {
        return !(a == b);                     // 用 == 实现 !=
    }

    friend bool operator<(const Version& a, const Version& b) {
        if (a.major_ != b.major_) {
            return a.major_ < b.major_;
        }
        return a.minor_ < b.minor_;
    }

    friend bool operator>(const Version& a, const Version& b) {
        return b < a;
    }

    friend bool operator<=(const Version& a, const Version& b) {
        return !(b < a);
    }

    friend bool operator>=(const Version& a, const Version& b) {
        return !(a < b);
    }

private:
    int major_;
    int minor_;
};
```

C++20 用 `<=>` 一次生成全部：

```cpp
#include <compare>

class Version {
public:
    Version(int major, int minor) : major_(major), minor_(minor) {}

    auto operator<=>(const Version&) const = default;   // 生成全部比较
    bool operator==(const Version&) const = default;

private:
    int major_;
    int minor_;
};
```

★ **关键认知**：`operator<=>` 返回比较类别（`strong_ordering`、`weak_ordering`、`partial_ordering`），编译器据此合成 `< <= > >=`。成员按声明顺序依次比较，和 `std::tuple` 的规则一致。C++20 项目里优先用默认实现。

⚠ **关键认知**：`==` 和 `<` 的语义必须**自洽**：如果 `a == b` 为真，就不能有 `a < b` 或 `b < a` 为真。手写六个比较时最容易出现这种矛盾，用 `<=>` 或"只实现两个、其余派生"能规避。

# 5. 下标与函数调用

## 5.1 operator[]

```cpp
class IntArray {
public:
    explicit IntArray(std::size_t n) : data_(new int[n]), size_(n) {}

    int& operator[](std::size_t i) { return data_[i]; }              // 可写
    const int& operator[](std::size_t i) const { return data_[i]; }  // 只读

    std::size_t size() const { return size_; }

    ~IntArray() { delete[] data_; }

private:
    int* data_;
    std::size_t size_;
};

IntArray a(10);
a[0] = 42;                 // 调用非 const 版本
const IntArray& ca = a;
int v = ca[0];             // 调用 const 版本，只读
```

★ **关键认知**：`operator[]` 必须成对提供 const 和非 const 版本。只提供非 const 版本，const 对象就无法下标访问；只提供 const 版本，就无法通过下标写入。`std::vector`、`std::map` 都是这么做的。

## 5.2 operator()

```cpp
class Multiplier {
public:
    explicit Multiplier(int factor) : factor_(factor) {}
    int operator()(int x) const { return x * factor_; }
private:
    int factor_;
};

Multiplier triple(3);
std::cout << triple(5);    // 15，对象像函数一样被调用
```

这种对象叫**函数对象**（functor），能携带状态，常作为算法参数，也是 lambda 的底层形态。

# 6. 递增递减与类型转换

```cpp
class Counter {
public:
    Counter& operator++() {          // 前置：先加，返回引用
        ++value_;
        return *this;
    }

    Counter operator++(int) {        // 后置：int 是哑参数，用于区分
        Counter old = *this;         // 保存旧值
        ++value_;
        return old;                  // 返回旧值的副本
    }

    explicit operator bool() const { // 显式转换，避免意外隐式转换
        return value_ != 0;
    }

private:
    int value_ = 0;
};
```

★ **关键认知**：前置 `++` 返回引用、后置 `++` 返回值，这个差别不是风格问题——后置必须返回自增前的快照，必然产生一个副本。循环里优先用 `++it` 而不是 `it++`，对迭代器能省一次拷贝。

⚠ **关键认知**：类型转换运算符默认允许隐式转换，容易引发意外。C++11 起用 `explicit operator bool()` 可以只在布尔语境（`if`、`&&`、`!`）里生效，这是 `std::unique_ptr`、`std::optional` 的标准做法。转换运算符别滥用，多一个隐式转换就多一条难以追踪的调用路径。

# 7. 边界与坑

| 坑 | 说明 | 正确做法 |
|----|------|----------|
| `+` 返回引用 | 悬垂或改坏操作数 | 返回值 |
| `==` 与 `!=` 不一致 | 两套独立逻辑 | 用一个实现另一个 |
| `[]` 缺 const 版本 | const 对象无法访问 | 成对实现 |
| 流运算符返回 void | 无法链式调用 | 返回 `ostream&` |
| 重载 `&&` `\|\|` | 丢失短路求值 | 一般不要重载 |
| 重载 `,` | 丢失求值顺序保证 | 几乎不要重载 |
| 滥用 `operator+` 做副作用 | 语义反直觉 | 运算符语义要贴合习惯 |

⚠ **关键认知**：不要重载 `&&`、`||`、`,`。重载后它们变成普通函数调用，**短路求值和顺序点保证都会丢失**，`p && p->foo()` 可能直接崩溃。这三个运算符在实践中基本只用于极端库场景。

# 8. 动手练习

1. 为 `Vec2` 实现 `+`、`-`、`*`（数乘）、`==`，要求 `+` 用 `+=` 实现，`!=` 用 `==` 实现。
2. 为 `Point` 实现 `operator<<` 和 `operator>>`，使 `(1, 2)` 格式能往返读写，并测试链式 `cout << a << b`。
3. 实现一个带 const/非 const 双版本 `operator[]` 的 `Matrix` 类，验证 const 对象只能读。

# 9. 小结

| 概念 | 要点 |
|------|------|
| 本质 | 运算符是函数调用的语法糖 |
| 成员/非成员 | 赋值、下标、调用必须成员；二元算术和比较优先非成员 |
| 返回值 | `+` 返回值，`+=` 返回引用 |
| 流运算符 | 非成员，返回流引用 |
| 比较 | C++20 用 `<=>` 一次生成 |
| 下标 | 成对提供 const 和非 const |
| 前置/后置 ++ | 前置返回引用，后置返回值 |
| 不做的事 | 不重载 `&&`、`\|\|`、`,` |

下一篇进入模板基础，讲清楚函数模板和类模板怎么实例化、模板参数如何推导、特化与可变参数模板的入门用法。
