C++17 引入了一批"词汇类型"（vocabulary types）：`optional`、`variant`、`any`、`string_view`。它们的共同点是把某种常见意图写进类型，而不是靠约定和注释。比如"这个函数可能没有返回值"用 `optional` 表达，比返回 `-1` 或空指针更清楚。但这些类型各有一条适用边界，用错地方比不用更糟。本篇逐个讲清用法、边界和坑。

# 1. 为什么需要词汇类型

在没有这些类型时，代码里充满了"用特殊值表示不存在"的约定：

```cpp
int findIndex(const std::string& s);   // 返回 -1 表示没找到
const char* getName();                 // 返回 nullptr 表示没有
void* getUserData();                   // 返回 nullptr，且不知道类型
```

这些约定的问题：特殊值可能和合法值冲突（如果 -1 是合法下标呢）、类型信息丢失、调用方容易忘记检查。

★ **关键认知**：词汇类型的价值是**让"可能没有"、"多选一"、"类型不定"成为类型系统的一部分**，编译器能帮你检查，读代码的人不用查文档。它们不是"更花哨的写法"，而是把隐含约定显式化。

# 2. optional

## 2.1 基本用法

`std::optional<T>` 表示"要么有一个 `T`，要么什么都没有"。

```cpp
#include <optional>
#include <string>

std::optional<int> parseInt(const std::string& s) {
    try {
        return std::stoi(s);
    } catch (...) {
        return std::nullopt;        // 表示失败
    }
}

auto v = parseInt("42");
if (v) {                            // 有值
    std::cout << *v << '\n';        // 解引用取值
}
auto empty = parseInt("abc");
std::cout << empty.value_or(0);     // 没有值时返回默认值 0
```

| 操作 | 作用 |
|------|------|
| `has_value()` / `operator bool` | 是否有值 |
| `*opt` / `opt->member` | 访问值，无值时未定义行为 |
| `value()` | 访问值，无值时抛 `bad_optional_access` |
| `value_or(default)` | 有值则返回，否则返回默认值 |
| `reset()` | 清空 |
| `emplace(...)` | 原地构造值 |

## 2.2 与指针、特殊值对比

| 方案 | 表达力 | 开销 | 风险 |
|------|--------|------|------|
| 返回 `T` + 特殊值 | 弱，需约定 | 无 | 特殊值与合法值冲突 |
| 返回 `T*` | 中 | 指针大小，可能堆分配 | 空指针、所有权不清 |
| 返回 `optional<T>` | 强 | 值 + 一个 bool | 解引用前需检查 |
| 抛异常 | 强 | 异常路径开销 | 不适合"常见失败" |

```cpp
// 用 optional 表达"可能取不到配置"
std::optional<std::string> getConfig(const std::string& key);
```

★ **关键认知**：`optional` 适合"**失败是正常情况、不是异常**"的场景，比如查找、解析、可选项读取。真正的错误（文件损坏、网络中断）仍应抛异常或返回错误码。`optional` 只表达"有没有"，不携带"为什么没有"。

## 2.3 常见坑

```cpp
std::optional<int> opt;      // 空
int x = *opt;                // 未定义行为！没有检查
int y = opt.value();         // 抛 bad_optional_access
int z = opt.value_or(0);     // 安全，返回 0
```

⚠ **关键认知**：`optional<T&>` **不合法**。标准明确禁止 `optional` 持有引用，因为引用不可重新绑定，`optional` 的赋值语义无法实现。需要"可能没有的引用"时，用 `T*` 或 `std::reference_wrapper<T>`。

⚠ **关键认知**：`optional<T>` 会为 `T` 预留空间并额外存一个 bool，所以它比 `T` 大。不要把大对象塞进 `optional` 频繁拷贝；需要时用 `optional<std::unique_ptr<T>>` 或直接传引用。

# 3. variant

## 3.1 基本用法

`std::variant<T1, T2, ...>` 表示"**在若干类型中恰好是其中一个**"，是一个类型安全的 union。

```cpp
#include <variant>
#include <string>

std::variant<int, double, std::string> value;
value = 42;                       // 当前持有 int
value = 3.14;                     // 变成 double
value = std::string("hello");     // 变成 string

std::cout << value.index();       // 当前是第几个类型，从 0 开始
```

## 3.2 visit

访问 `variant` 的标准方式是 `std::visit`，它对当前持有的类型分派：

```cpp
#include <variant>

std::variant<int, std::string> v = 42;

std::visit([](const auto& x) {
    std::cout << x << '\n';
}, v);

// 也可以给不同类型写不同的重载
struct Printer {
    void operator()(int x) const { std::cout << "int: " << x << '\n'; }
    void operator()(const std::string& s) const { std::cout << "str: " << s << '\n'; }
};
std::visit(Printer{}, v);
```

★ **关键认知**：`std::visit` 要求访问器能处理 `variant` 的**所有**备选类型，否则编译失败。这是好事——它保证你不会漏掉某个分支。如果确实只想处理部分类型，可以配合 `if constexpr` 或 `overloaded` 技巧。

## 3.3 valueless_by_exception

`variant` 在切换类型时，如果旧值析构抛异常、或新值构造抛异常，可能进入"没有值"的状态：

```cpp
std::variant<int, std::string> v = "hello";
try {
    v = 42;   // 若某步抛异常，v 可能 valueless
} catch (...) {
    if (v.valueless_by_exception()) {
        // 既不是 int 也不是 string
    }
}
```

⚠ **关键认知**：`valueless_by_exception()` 是 `variant` 最反直觉的地方——一个"必须恰好持有一个类型"的容器，理论上可能什么也不持有。实践中少见，但写健壮代码时在 `visit` 前应检查，或确保备选类型的构造/析构不抛异常。

## 3.4 与继承对比

| 维度 | `variant` | 继承 + 虚函数 |
|------|-----------|---------------|
| 类型集合 | 编译期封闭 | 可运行时扩展 |
| 分派 | `visit`，可内联 | 虚表间接调用 |
| 内存 | 栈上，大小取最大备选 | 通常堆分配 |
| 值语义 | 支持拷贝/移动 | 需自己管理 |
| 适用 | 类型集合固定的"多选一" | 开放的多态体系 |

★ **关键认知**：`variant` 适合**类型集合固定且封闭**的场景，比如"配置值只能是 int/double/string"、"解析结果只能是数字/字符串/错误"。需要运行时添加新类型时，才用继承和多态。用 `variant` 硬凑开放体系，会写出大量 `visit` 分支。

# 4. any

## 4.1 用法

`std::any` 能装**任意类型**的单个值，类型信息在运行时保存：

```cpp
#include <any>
#include <string>

std::any a = 42;
a = std::string("hello");
a = 3.14;

if (a.type() == typeid(double)) {
    double d = std::any_cast<double>(a);
}

// 错误类型转换会抛 bad_any_cast
try {
    int i = std::any_cast<int>(a);
} catch (const std::bad_any_cast&) {
    // 类型不匹配
}
```

## 4.2 边界

| 维度 | `any` | `variant` |
|------|-------|-----------|
| 类型集合 | 任意，运行时 | 编译期列出 |
| 类型检查 | 运行时 | 编译期 |
| 性能 | 可能堆分配 | 栈上 |
| 可读性 | 弱，看不出可能是什么 | 强 |
| 适用 | 插件参数、通用容器 | 明确的多选一 |

⚠ **关键认知**：`std::any` 是这几个类型里**最不该常用**的。它把类型信息推到运行时，调用方必须 `any_cast` 并处理失败，本质上是"带类型检查的 `void*`"。只有在确实无法在编译期知道类型时（比如脚本绑定、通用属性系统）才用。能用 `variant` 就用 `variant`。

# 5. string_view

## 5.1 用法

`std::string_view` 是只读、不拥有的字符视图，常用于函数参数：

```cpp
#include <string_view>

std::size_t countVowels(std::string_view text) {
    std::size_t n = 0;
    for (char c : text) {
        if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') ++n;
    }
    return n;
}

std::string s = "hello world";
countVowels(s);          // 从 string 构造
countVowels("literal");  // 从字面量构造
countVowels(s.substr(0, 5));  // 从临时 string 构造，函数调用期间安全
```

## 5.2 生命周期

```cpp
std::string_view bad() {
    std::string local = "temporary";
    return local;                 // 错误：local 销毁，视图悬垂
}

std::string_view alsoBad = std::string("temp");  // 临时对象立刻销毁
```

```text
string_view 只保存：
┌──────────────┬──────────┐
│ 数据指针      │  长度    │
└──────┬───────┴──────────┘
       ▼
   实际字符内存（由别人拥有）
```

★ **关键认知**：`string_view` 不延长被观察数据的生命周期，也不保证 `\0` 结尾。它只能用于"源数据确定活得比自己久"的只读场景。从临时 `std::string` 构造视图、把视图存进成员变量、跨作用域返回视图，都是悬垂的高发区。

## 5.3 不能隐式转回 string

`string_view` 不能隐式转换成 `std::string`，需要显式构造：

```cpp
std::string_view sv = "hello";
// std::string s = sv;          // 编译错误
std::string s(sv);              // 正确：显式构造，发生拷贝
```

⚠ **关键认知**：`string_view::data()` 不保证以 `\0` 结尾，直接传给 `printf("%s")`、`strlen` 等 C 函数可能越界读。需要 C 字符串时先构造 `std::string`，或确认视图确实覆盖到结尾的 `\0`。

# 6. 适用边界

| 需求 | 首选 | 不要用 |
|------|------|--------|
| 可能没有返回值 | `optional<T>` | 特殊值、裸指针 |
| 多选一、集合固定 | `variant<Ts...>` | `any`、`void*` |
| 只读字符串参数 | `string_view` | 临时 `string` 的视图 |
| 运行时任意类型 | `any` | `variant` 装不下时 |
| 需要错误原因 | 错误码/异常 | `optional`（不带原因） |
| 可能没有的引用 | `T*`、`reference_wrapper` | `optional<T&>`（非法） |

# 7. 常见坑

| 坑 | 后果 | 修复 |
|----|------|------|
| 未检查就 `*opt` | 未定义行为 | 先 `if (opt)` 或 `value_or` |
| 写 `optional<T&>` | 编译失败 | 用 `T*` |
| `variant` 漏处理某类型 | `visit` 编译失败 | 处理所有备选 |
| 忽略 `valueless_by_exception` | 访问异常 | 检查或保证不抛 |
| 用 `any` 替代明确类型 | 运行时错误、可读性差 | 优先 `variant` |
| 从临时 `string` 造视图 | 悬垂 | 先存成 `string` |
| 把 `string_view` 传给 C 函数 | 越界读 | 先转 `std::string` |

⚠ **关键认知**：这四种类型的共同前提是"**你得清楚它有没有值、是什么类型、数据活多久**"。它们把约定搬进了类型，但没有替你做检查。`optional` 要判空，`variant` 要覆盖所有分支，`string_view` 要保证生命周期，`any` 要处理类型不匹配。

# 8. 动手练习

1. 写一个 `findUser(id)` 函数，返回 `optional<User>`，分别处理"找到"和"没找到"，对比用 `value_or` 和 `if` 两种写法。
2. 用 `variant<int, double, string>` 表示一个配置值，用 `std::visit` 打印它，并尝试增加一种类型，观察编译器如何提示漏掉的分支。
3. 写一个接收 `string_view` 的分词函数，分别传入 `std::string`、字面量和 `substr` 结果，说明哪些调用是安全的。

# 9. 小结

| 类型 | 表达 | 边界 |
|------|------|------|
| `optional<T>` | 有值 / 无值 | 不携带原因，不能是引用 |
| `variant<Ts...>` | 多选一 | 集合编译期固定，`visit` 要全覆盖 |
| `any` | 任意单值 | 运行时类型检查，尽量少用 |
| `string_view` | 只读字符视图 | 不拥有、不保 `\0`、不延长寿命 |

下一篇进入容器选型与最佳实践，用一张总表把前面所有容器串起来，再讲缓存局部性、分配次数、异常安全等级和一组可以直接落地的编码规范。
