模板是 C++ 泛型编程的基础，`std::vector`、`std::sort`、`std::unique_ptr` 全靠它实现。初学时容易把它当成"带尖括号的宏"，结果被推导失败、链接错误、代码膨胀轮番教育。本篇从函数模板和类模板的基本写法入手，讲清楚实例化、推导、特化，以及可变参数模板的入门用法。

# 1. 函数模板

## 1.1 基本写法

```cpp
template <typename T>
T maxOf(const T& a, const T& b) {
    return a > b ? a : b;
}

int m1 = maxOf(3, 7);              // T 推导为 int
double m2 = maxOf(2.5, 1.5);       // T 推导为 double
```

`typename` 和 `class` 在这里等价，习惯上类型参数用 `typename`，模板模板参数用 `class`。

★ **关键认知**：模板**本身不是代码**，它是一张"生成代码的图纸"。编译器在你**使用**它的时候，才用具体类型替换 `T`，生成一个真正的函数，这个过程叫**实例化**。`maxOf<int>` 和 `maxOf<double>` 是两个完全独立的函数。

## 1.2 推导规则

推导看实参类型，**不看返回值**：

```cpp
template <typename T>
void f(T x);          // 按值：忽略顶层 const 和引用

template <typename T>
void g(T& x);         // 按引用：保留 const

template <typename T>
void h(const T& x);   // const 引用：T 是不带 const 的原始类型

int a = 1;
const int b = 2;

f(a);                 // T = int
f(b);                 // T = int（顶层 const 被忽略）
g(b);                 // T = const int
h(b);                 // T = int（const 在参数上，不在 T 上）
```

★ **关键认知**：`T` 的推导结果由**形参形式**决定。`T`、`T&`、`const T&`、`T&&` 四种写法推导出的 `T` 各不相同，这就是为什么标准库接口对引用和 const 如此讲究。搞不清时，用 `static_assert(std::is_same_v<T, 期望类型>)` 打印验证。

## 1.3 显式指定与默认参数

当推导有歧义或需要强制类型时，显式写尖括号：

```cpp
template <typename T>
T add(T a, T b) { return a + b; }

// add(1, 2.5);        // 错误：T 同时被推导为 int 和 double
add<double>(1, 2.5);   // 显式指定 T = double，1 被转换

template <typename T = int>
T zero() { return T{}; }

int z1 = zero();       // 用默认参数 T = int
double z2 = zero<double>();
```

⚠ **关键认知**：两个参数都参与同一个 `T` 的推导时，实参类型必须**完全一致**（忽略引用和顶层 const）。`add(1, 2.5)` 推导失败不是 bug，是设计——如果想支持混合类型，得用两个模板参数。

# 2. 类模板

## 2.1 定义与使用

```cpp
template <typename T>
class Box {
public:
    explicit Box(T value) : value_(std::move(value)) {}
    const T& get() const { return value_; }
    void set(T value) { value_ = std::move(value); }

private:
    T value_;
};

Box<int> a(42);
Box<std::string> b("hello");
```

C++17 起支持类模板参数推导（CTAD）：

```cpp
Box c(42);                  // 自动推导为 Box<int>
Box d(std::string("hi"));   // Box<std::string>
```

★ **关键认知**：类模板的每个实例都是**独立的类型**。`Box<int>` 和 `Box<double>` 之间没有继承关系，不能互相赋值，也不能互相转换（除非你显式提供转换构造）。

## 2.2 成员函数定义

成员函数在类外定义时，要带上模板头：

```cpp
template <typename T>
class Box {
public:
    T get() const;
private:
    T value_;
};

template <typename T>
T Box<T>::get() const {        // 注意 Box<T>:: 前缀
    return value_;
}
```

## 2.3 非类型模板参数

模板参数不一定是类型，也可以是编译期常量：

```cpp
template <typename T, std::size_t N>
class FixedArray {
public:
    T& operator[](std::size_t i) { return data_[i]; }
    constexpr std::size_t size() const { return N; }

private:
    T data_[N];                // N 是编译期常量，可以做数组大小
};

FixedArray<int, 4> arr;
arr[0] = 1;
static_assert(arr.size() == 4);
```

★ **关键认知**：非类型模板参数让"大小"成为**类型的一部分**。`FixedArray<int, 4>` 和 `FixedArray<int, 8>` 是不同类型，前者不能赋给后者。这既是安全保证，也意味着每种大小都会生成一份代码。

⚠ **关键认知**：C++17 前非类型参数只能是整型、枚举、指针、引用；C++20 起允许浮点数和字面量类类型。跨版本项目里别默认浮点模板参数可用。

# 3. 特化

## 3.1 全特化

为某个具体类型提供专门实现：

```cpp
template <typename T>
struct Printer {
    static void print(const T& v) {
        std::cout << v << '\n';
    }
};

template <>
struct Printer<bool> {                    // 全特化：专门处理 bool
    static void print(bool v) {
        std::cout << (v ? "true" : "false") << '\n';
    }
};

Printer<int>::print(1);                   // 1
Printer<bool>::print(true);               // true（而不是 1）
```

## 3.2 偏特化

只固定部分参数，或对参数形式做限定：

```cpp
template <typename T, typename U>
struct Pair {
    static constexpr const char* name = "generic";
};

template <typename T>
struct Pair<T, T> {                       // 两个参数相同时
    static constexpr const char* name = "same";
};

template <typename T>
struct Pair<T*, T*> {                     // 两个都是指针时
    static constexpr const char* name = "pointer";
};
```

⚠ **关键认知**：**函数模板不支持偏特化**，只能全特化。需要偏特化时，通常改写成类模板 + 静态成员函数，或者用 `if constexpr`、SFINAE、concepts 来做条件分支。这是初学者最容易踩的语法限制。

# 4. 可变参数模板

## 4.1 基本形式

```cpp
template <typename... Args>
void printAll(Args... args) {
    ((std::cout << args << ' '), ...);     // C++17 折叠表达式
    std::cout << '\n';
}

printAll(1, 2.5, "hi");                    // 1 2.5 hi
```

`Args...` 是**参数包**，可以接受任意个类型；`args...` 是**函数参数包**。

## 4.2 递归展开（C++17 前写法）

```cpp
void print() {                             // 递归终止
    std::cout << '\n';
}

template <typename T, typename... Rest>
void print(const T& first, const Rest&... rest) {
    std::cout << first << ' ';
    print(rest...);                        // 每次剥掉一个参数
}
```

## 4.3 获取参数个数与转发

```cpp
template <typename... Args>
std::size_t count(Args&&...) {
    return sizeof...(Args);                // sizeof... 返回包中元素个数
}

template <typename... Args>
void forwardAll(Args&&... args) {
    printAll(std::forward<Args>(args)...); // 完美转发，第 10 篇详述
}
```

★ **关键认知**：可变参数模板是 `std::make_unique`、`std::emplace_back`、`std::thread` 构造函数的基础。它们的共同套路是"接收任意参数 → 原样转发给目标构造函数"，避免了中间拷贝。

⚠ **关键认知**：参数包**不能直接用下标访问**，只能通过递归、折叠表达式或初始化列表展开。想取第 N 个参数，通常要配合 `std::tuple` 和 `std::get`，而不是 `args[N]`。

# 5. 模板与头文件

模板的定义通常必须放在头文件里：

```cpp
// box.h
template <typename T>
class Box {
public:
    T get() const { return value_; }
private:
    T value_;
};
```

原因：编译器在**使用处**才实例化，如果定义在 `.cpp` 里，别的翻译单元看不到模板体，就无法生成代码。

```text
box.h + main.cpp  ──► 编译器看到模板定义和 Box<int> 使用 ──► 生成 Box<int>
box.cpp 单独编译   ──► 没有使用点，不生成任何实例
```

当确实想把实现藏进 `.cpp` 时，用**显式实例化**：

```cpp
// box.cpp
#include "box.h"
template class Box<int>;        // 只生成 Box<int>
template class Box<double>;
```

⚠ **关键认知**：显式实例化要求你**提前知道所有会用到的类型**。库作者常用这种做法缩减编译时间，代价是用户不能用未列出的类型。一般项目里，模板就老老实实放头文件。

# 6. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| `undefined reference` 到模板函数 | 定义在 `.cpp`，使用处看不到 | 定义移到头文件 |
| 推导失败 `no matching function` | 多个参数推导出不同 `T` | 显式指定类型或用两个参数 |
| 编译时间暴涨 | 模板实例化过多 | 减少组合、用显式实例化、拆头文件 |
| 报错信息极长 | 模板嵌套太深 | 从**第一行**和**最后一行**读，定位实际类型 |
| 想偏特化函数模板 | 语言不支持 | 改成类模板或 `if constexpr` |
| `>>` 被当成右移 | C++11 前的老问题 | C++11 起已自动处理，无需空格 |

⚠ **关键认知**：读模板报错时，**不要从第一行开始逐字读**。第一行通常是"某个实例化失败"，真正的类型不匹配往往在最后几行的 `required from` 链里。先找 "no matching function" 和 "candidate" 附近的实际类型。

# 7. 动手练习

1. 写一个函数模板 `clamp(value, low, high)`，要求三个参数类型一致，并测试 `int`、`double`、`std::string`。
2. 写一个类模板 `Stack<T>`，用 `std::vector<T>` 实现 `push/pop/top/empty`，再用 `Stack<int>` 和 `Stack<std::string>` 各实例化一次。
3. 用可变参数模板 + 折叠表达式实现 `sumAll(args...)`，支持任意个可相加的参数。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 模板 | 生成代码的图纸，使用时实例化 |
| 推导 | 由形参形式决定，不看返回值 |
| 类模板 | 每个实例是独立类型 |
| 非类型参数 | 编译期常量，可做数组大小 |
| 全特化 | 为具体类型提供专门实现 |
| 偏特化 | 只支持类模板，函数模板不支持 |
| 可变参数 | 参数包 + 折叠表达式/递归展开 |
| 定义位置 | 通常放头文件，或在 `.cpp` 显式实例化 |

下一篇进入异常与 RAII，讲清楚抛出捕获的流程、栈展开时发生了什么、`noexcept` 的意义，以及异常安全等级怎么划分。
