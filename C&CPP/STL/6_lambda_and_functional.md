lambda 是 C++11 带来的最实用的特性之一：它让你在调用点就地定义一个小函数，不必为三行逻辑单独写一个类。但 lambda 的捕获列表也是一堆坑的源头——按引用捕获的变量在闭包存活期间销毁、`mutable` 为什么改了值却没生效、`std::function` 存下 lambda 时到底拷贝了什么。本篇把 lambda 的语法、捕获语义、函数对象和 `std::function` 的开销讲清楚。

# 1. lambda 是什么

## 1.1 最小示例

```cpp
#include <algorithm>
#include <vector>

std::vector<int> v = {3, 1, 4, 1, 5};

std::sort(v.begin(), v.end(),
          [](int a, int b) { return a > b; });   // 就地定义降序比较

int threshold = 3;
auto above = std::count_if(v.begin(), v.end(),
                           [threshold](int x) { return x > threshold; });
```

一个 lambda 由四部分组成：

```text
[capture](parameters) -> return_type { body }
    ↑         ↑              ↑            ↑
  捕获列表   参数列表      返回类型(可省)  函数体
```

## 1.2 编译器生成了什么

lambda 本质上是编译器自动生成的一个**匿名函数对象**（闭包类型），捕获的变量成为它的成员：

```cpp
int threshold = 3;
auto f = [threshold](int x) { return x > threshold; };

// 大致等价于：
struct __Lambda {
    int threshold;                       // 按值捕获 → 成员
    bool operator()(int x) const {       // 默认 const
        return x > threshold;
    }
};
```

★ **关键认知**：lambda 不是"函数指针"，它是一个**带 `operator()` 的类实例**。捕获的变量是成员变量，调用 lambda 就是调用 `operator()`。理解这一点，捕获、`mutable`、生命周期这些行为都能自己推出来。

# 2. 捕获列表

## 2.1 值捕获与引用捕获

| 写法 | 含义 | 修改是否影响外部 | 风险 |
|------|------|------------------|------|
| `[x]` | 按值拷贝 | 否 | 拷贝成本 |
| `[&x]` | 按引用绑定 | 是 | 外部变量销毁后悬垂 |
| `[=]` | 隐式按值捕获用到的 | 否 | C++20 起不推荐 |
| `[&]` | 隐式按引用捕获用到的 | 是 | 悬垂风险高 |
| `[x, &y]` | 混合 | 视变量而定 | — |

```cpp
int a = 1, b = 2;

auto byValue = [a]() { return a; };          // 拷贝 a 的当前值
auto byRef = [&a]() { a += 1; };             // 绑定 a 本身
byRef();
// a 变成 2，byValue 里的 a 仍是 1
```

★ **关键认知**：按值捕获发生在**lambda 创建的那一刻**，之后外部变量怎么变都不会影响闭包里的副本。按引用捕获绑定的是变量本身，闭包和外部看到的是同一块内存。选择哪种，取决于"你要快照还是要同步"。

## 2.2 初始化捕获与移动捕获

C++14 起可以在捕获列表里写初始化表达式，这解决了"移动-only 对象无法按值捕获"的问题：

```cpp
#include <memory>

auto ptr = std::make_unique<int>(42);
// auto f = [ptr]() { return *ptr; };        // 错误：unique_ptr 不能拷贝
auto f = [p = std::move(ptr)]() {            // 正确：移动进闭包
    return *p;
};
// 此时 ptr 为空，闭包独占所有权
```

也可以给捕获的变量起新名字、做计算：

```cpp
int base = 10;
auto add = [offset = base * 2](int x) { return x + offset; };
```

## 2.3 捕获 this

在成员函数里写 `[this]` 捕获的是**当前对象的指针**，不是对象副本：

```cpp
class Widget {
public:
    int value = 0;
    auto makeCallback() {
        return [this]() { return value; };   // 通过 this 访问成员
    }
};
```

⚠ **关键认知**：`[this]` 捕获的是裸 `this` 指针。如果对象先于闭包销毁，闭包再访问成员就是悬垂。C++17 起可以用 `[*this]` 按值拷贝整个对象，避免这个风险；异步回调、事件注册这类"闭包可能活得更久"的场景，优先用 `[*this]` 或 `shared_from_this()`。

## 2.4 捕获时机

```cpp
int x = 1;
auto f = [x]() { return x; };
x = 100;
std::cout << f();     // 输出 1，不是 100
```

⚠ **关键认知**：`[=]` 和 `[&]` 都是**隐式捕获**，会让读者看不出闭包依赖了哪些变量，也可能意外捕获不该捕获的东西。C++20 明确不推荐 `[=]`，现代写法是**显式列出每个捕获**，依赖关系一目了然。

# 3. mutable

## 3.1 为什么需要

lambda 的 `operator()` 默认是 `const` 的，所以按值捕获的成员在函数体内不能修改：

```cpp
int count = 0;
auto bad = [count]() { ++count; };   // 编译错误：count 是 const 成员

auto good = [count]() mutable { ++count; };  // 加 mutable 后可改
```

## 3.2 值捕获的状态

```cpp
auto counter = [n = 0]() mutable { return ++n; };

std::cout << counter();   // 1
std::cout << counter();   // 2
std::cout << counter();   // 3
// 每次调用改的是闭包内部的 n，不是外部变量
```

```text
闭包对象
┌──────────────┐
│ n = 0        │  ← 内部状态，每次调用累加
│ operator()   │
└──────────────┘
外部没有任何变量被修改
```

★ **关键认知**：`mutable` 允许修改的是**闭包自己的成员**（按值捕获的副本），不是外部变量。想要"有状态的函数"，这正是需要的；但如果你以为 `mutable` 能改外部变量，那说明应该用引用捕获。

⚠ **关键认知**：`mutable` lambda 的 `operator()` 不是 `const`，因此不能赋值给要求 const 可调用的 `std::function` 场景时要留意；另外，同一个 lambda 表达式生成的闭包类型是**唯一**的，两个写法相同的 lambda 也是不同类型。

# 4. 函数对象

## 4.1 仿函数

在 lambda 之前，定制行为靠"重载了 `operator()` 的类"：

```cpp
struct Greater {
    bool operator()(int a, int b) const {
        return a > b;
    }
};

std::sort(v.begin(), v.end(), Greater{});
```

标准库自带一批：`std::less`、`std::greater`、`std::plus`、`std::multiplies`。

## 4.2 lambda 与仿函数

| 维度 | lambda | 具名仿函数 |
|------|--------|------------|
| 定义位置 | 就地 | 需要单独写类 |
| 类型 | 每个表达式唯一匿名类型 | 有名字，可复用 |
| 捕获状态 | 支持 | 通过成员变量 |
| 可读性 | 短逻辑更好 | 复杂逻辑更好 |
| 跨函数复用 | 不方便 | 方便 |

★ **关键认知**：lambda 和仿函数在底层是同一套机制，选择只看**复用范围**。只在一个调用点用一次，写 lambda；逻辑复杂、需要在多处复用、或需要具名类型做模板参数时，写仿函数。

# 5. std::function

## 5.1 类型擦除

`std::function` 是一个"能装下任何可调用对象"的容器，代价是类型擦除：

```cpp
#include <functional>

std::function<int(int)> f;

f = [](int x) { return x + 1; };          // 装 lambda
f = [](int x) { return x * 2; };          // 换成另一个 lambda
f = std::negate<int>{};                   // 换成仿函数

int y = f(10);
```

```text
std::function<int(int)>
┌────────────────────────────┐
│ 类型擦除层                  │
│  ┌──────────┐              │
│  │ 具体闭包  │──operator()──► 间接调用
│  └──────────┘              │
└────────────────────────────┘
  可能堆分配 + 一次间接跳转
```

## 5.2 开销

| 方面 | 说明 |
|------|------|
| 存储 | 小对象可能内联，大的闭包会堆分配 |
| 调用 | 通过虚函数/函数指针间接调用，难以内联 |
| 拷贝 | 拷贝 `std::function` 会拷贝它持有的闭包 |
| 空状态 | 默认构造的 `std::function` 调用会抛 `bad_function_call` |

★ **关键认知**：`std::function` 的价值是**统一类型**——把各种可调用对象存进同一个容器、同一个成员变量。代价是间接调用和可能的堆分配。它是"需要类型擦除时"的工具，不是"所有回调都该用它"。

⚠ **关键认知**：能直接用模板参数或 `auto` 接收 lambda 时，不要经过 `std::function`。模板会保留具体类型、允许内联；`std::function` 会擦掉类型、增加一次间接跳转。只有在"运行时要换不同回调、必须统一类型"时才用它。

## 5.3 何时用

| 场景 | 选择 |
|------|------|
| 算法谓词 | 直接传 lambda，不用 `std::function` |
| 类成员保存回调 | `std::function` 合理 |
| 回调注册表（多种签名统一） | `std::function` |
| 性能敏感的逐元素调用 | 模板 / lambda |
| 需要判空 | `std::function` + `if (f)` |

# 6. 闭包生命周期

## 6.1 悬垂引用

```cpp
std::function<int()> makeBad() {
    int local = 42;
    return [&local]() { return local; };   // 错误：local 已销毁
}
```

调用过程是：栈上创建 `local`，闭包捕获 `&local`，函数返回时 `local` 销毁，闭包从此持有一个悬垂引用，之后调用即未定义行为。

## 6.2 异步回调

```cpp
void registerCallback(std::function<void()> cb);

void setup() {
    int counter = 0;
    registerCallback([&counter]() { ++counter; });  // 危险
}
// setup 返回后 counter 销毁，回调触发时悬垂
```

★ **关键认知**：闭包不会延长被引用对象的生命周期。把 lambda 存起来（放进容器、注册为回调、交给异步任务）时，按引用捕获的局部变量必须活得比闭包久。做不到就按值捕获，或用 `shared_ptr` 明确共享所有权。

⚠ **关键认知**：异步场景优先**按值捕获**或**移动捕获**所有权。`[&]` 在同步调用里通常安全，一旦闭包被存起来跨作用域使用，就变成定时炸弹。这类 bug 往往在压力测试或偶发路径上才暴露，极难定位。

# 7. 常见坑

| 坑 | 后果 | 修复 |
|----|------|------|
| 按引用捕获局部变量后存起来 | 悬垂引用 | 按值捕获或 `shared_ptr` |
| 以为 `[=]` 能修改外部变量 | 编译失败 | 加 `mutable` 或用引用 |
| `[this]` 后对象先销毁 | 悬垂 | `[*this]` 或 `shared_from_this` |
| 用 `std::function` 包所有 lambda | 间接调用、堆分配 | 能模板就模板 |
| 默认构造的 `std::function` 直接调 | 抛异常 | 先判空 |

⚠ **关键认知**：每个 lambda 表达式的类型都是**独一无二**的，即使参数和函数体完全一样。所以 `auto a = []{}; auto b = []{};` 的 `a` 和 `b` 类型不同，不能放进同一个 `vector`，除非用 `std::function` 擦除类型。

# 8. 动手练习

1. 写一个返回 lambda 的函数，分别用 `[&]` 和 `[=]` 捕获局部变量，调用返回的闭包，观察哪个是安全的。
2. 用 `[n = 0]() mutable` 实现一个计数器，连续调用三次打印 1、2、3；再把它赋给 `std::function<int()>`，验证行为一致。
3. 用 `std::function` 实现一个简易回调注册表，存入两个不同的 lambda，遍历调用；再把 `std::function` 换成模板参数版本，比较代码与调用方式。

# 9. 小结

| 概念 | 要点 |
|------|------|
| lambda 本质 | 编译器生成的匿名函数对象 |
| 值捕获 | 创建时拷贝快照，之后不同步 |
| 引用捕获 | 绑定变量本身，警惕悬垂 |
| 初始化捕获 | `[p = std::move(ptr)]` 支持移动 |
| `[this]` | 捕获裸指针，`[*this]` 才拷贝对象 |
| `mutable` | 允许改闭包自己的成员 |
| 仿函数 | 有名字的可调用类，适合复用 |
| `std::function` | 类型擦除，统一类型但有开销 |
| 生命周期 | 闭包不延长被捕获对象的寿命 |

下一篇进入 `optional`、`variant`、`any` 和 `string_view`，讲清楚这些"词汇类型"各自解决什么问题、适用边界在哪，以及 `optional<T&>` 为什么不存在、`variant` 的 `valueless_by_exception` 是什么。
