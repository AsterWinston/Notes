现代 C++（C++11 之后）并没有改变语言的核心模型，但它把大量"安全、好写"的工具塞进了标准库和语法层：`auto` 少写类型、范围 for 少写下标、结构化绑定少写 `first`/`second`、`optional`/`variant` 把"可能没有"和"多选一"变成类型。本篇挑最常用的几个讲清楚用法和边界，最后给出一份能落地的实践清单，把这个模块串起来。

# 1. auto

## 1.1 推导规则

`auto` 的推导规则和模板参数推导一致，**忽略顶层 const 和引用**：

```cpp
int x = 1;
const int cx = 2;
int& rx = x;

auto a = x;      // int
auto b = cx;     // int（const 被忽略）
auto c = rx;     // int（引用被忽略，拷贝一份）

auto& d = x;     // int&
const auto& e = x;  // const int&
auto* f = &x;    // int*
```

```text
auto 推导步骤：
1. 去掉实参的引用和顶层 const
2. 用剩余类型匹配 auto
3. 声明里的 &、const、* 再叠加上去
```

★ **关键认知**：`auto` 不是"动态类型"，类型在**编译期完全确定**，只是你不用手写。`auto a = 1; a = "str";` 依然编译错误，`a` 永远只能是 `int`。

## 1.2 常见陷阱

```cpp
std::vector<int> v = {1, 2, 3};
auto size = v.size();        // std::size_t，无符号
for (auto i = 0; i < v.size(); ++i) {
    // i 是 int，和 size_t 比较会有符号/无符号警告
}
// 正确：用 size_t 或 std::size
for (std::size_t i = 0; i < v.size(); ++i) { }
```

⚠ **关键认知**：`auto` 会**丢掉引用和 const**。范围 for 里 `for (auto x : v)` 每轮都拷贝一份元素；想避免拷贝且不修改，写 `for (const auto& x : v)`。这是 `auto` 最常见的性能坑。

⚠ **关键认知**：代理对象。`std::vector<bool>` 的 `operator[]` 返回代理类，`auto b = v[0];` 得到的是代理而不是 `bool`。需要真实值时显式写 `bool b = v[0];`。类似的还有某些表达式模板库。

# 2. 范围 for

```cpp
std::vector<int> v = {1, 2, 3};

for (int x : v) {          // 拷贝每个元素
    std::cout << x;
}

for (const auto& x : v) {  // 只读，不拷贝（推荐默认写法）
    std::cout << x;
}

for (auto& x : v) {        // 可修改原元素
    x *= 2;
}
```

它本质上是让编译器生成一个从 `begin()` 到 `end()` 的迭代器循环，`x` 是解引用后的元素副本。

★ **关键认知**：范围 for 的默认写法是 `const auto&`。它同时满足"不拷贝"和"不修改"两个安全目标；需要修改时再改成 `auto&`。`for (auto x : v)` 只有在内建类型（`int`、`double`）且确实要副本时才合适。

⚠ **关键认知**：范围 for 遍历时**不要修改容器结构**。`push_back` 可能导致迭代器失效，`erase` 会让范围 for 内部的迭代器失效。需要边遍历边删除，用带 `erase` 返回值的普通循环，或 erase-remove 惯用法。

⚠ **关键认知**：对临时对象使用范围 for 是安全的（生命周期被延长），但 `for (auto x : getVector())` 每轮拷贝元素，`for (const auto& x : getVector())` 才不拷贝。返回临时容器时，优先 `const auto&`。

# 3. 结构化绑定

C++17 起可以一次性拆开 pair、tuple、结构体：

```cpp
std::map<std::string, int> ages = {{"Tom", 18}};
for (const auto& [name, age] : ages) {    // 拆 map 的 pair
    std::cout << name << ": " << age << '\n';
}
std::pair<int, std::string> p = {1, "one"};
auto [id, text] = p;                      // 拆 pair
std::tuple<int, double, std::string> t = {1, 2.5, "x"};
auto [a, b, c] = t;                       // 拆 tuple
```

也可以拆普通结构体：

```cpp
struct Point {
    int x;
    int y;
};

Point pt{1, 2};
auto [x, y] = pt;          // x = 1, y = 2
```

★ **关键认知**：结构化绑定默认是**拷贝**，`auto [x, y] = pt;` 修改 `x` 不影响 `pt`。要修改原对象，写 `auto& [x, y] = pt;`；只读且不拷贝，写 `const auto& [x, y] = pt;`。

⚠ **关键认知**：结构化绑定不能用于**位域**，也不能在 lambda 捕获列表里直接捕获单个绑定名（C++20 前）。需要捕获时，先赋给普通变量，或整体捕获。

# 4. optional 与 variant

## 4.1 optional：可能没有值

```cpp
#include <optional>
std::optional<int> findIndex(const std::vector<int>& v, int target) {
    for (std::size_t i = 0; i < v.size(); ++i) {
        if (v[i] == target) {
            return static_cast<int>(i);
        }
    }
    return std::nullopt;       // 明确表示"没有"
}

if (auto idx = findIndex(v, 42)) {
    std::cout << "找到: " << *idx << '\n';
} else {
    std::cout << "没找到\n";
}
```

| 写法 | 语义 |
|------|------|
| `std::optional<T>` | 要么有 `T`，要么没有 |
| `has_value()` / `operator bool` | 是否有值 |
| `value()` | 有值返回，无值抛 `bad_optional_access` |
| `value_or(default)` | 无值时返回默认值 |
| `std::nullopt` | 空值常量 |

★ **关键认知**：`optional` 解决的是"**函数可能返回一个值，也可能返回不了**"这个语义，比返回指针（可能为空）或特殊值（如 `-1`）更清晰。但它不携带错误原因，需要原因时用 `std::expected`（C++23）或自定义 `Result`。

⚠ **关键认知**：`optional<T>` 里存的 `T` 仍然需要完整类型，且 `optional<bool>` 占 2 字节（值 + 标志），不是特化的 1 位。不要把它当成"零开销的 may-be"。

## 4.2 variant：多选一

```cpp
#include <variant>

std::variant<int, double, std::string> value;
value = 42;                 // 存 int
value = 3.14;               // 存 double
value = std::string("hi");  // 存 string

if (std::holds_alternative<int>(value)) {
    std::cout << std::get<int>(value);
}

std::visit([](auto&& v) {
    std::cout << v << '\n';   // 自动匹配当前类型
}, value);
```

★ **关键认知**：`variant` 是**类型安全**的联合体。它知道当前存的是哪个类型，访问错类型会抛 `std::bad_variant_access`，不会像 C 的 `union` 那样读出乱码。`std::visit` 是处理多类型的标准方式，配合泛型 lambda 可以写得很简洁。

⚠ **关键认知**：`variant` 的大小等于**最大成员的大小 + 类型标签 + 对齐填充**，不是各成员之和。存大对象时要注意内存占用，必要时用 `std::unique_ptr` 间接持有。

# 5. string_view

```cpp
#include <string_view>

void print(std::string_view sv) {     // 不拷贝，接受 string、字面量、char*
    std::cout << sv << '\n';
}

std::string s = "hello";
print(s);              // string → string_view，零拷贝
print("world");        // 字面量 → string_view，零拷贝
print(s.substr(1, 3)); // 注意：substr 返回 string，这里仍有一次构造
```

| 对比 | `const std::string&` | `std::string_view` |
|------|---------------------|-------------------|
| 接受字面量 | 需构造临时 string | 直接引用 |
| 拷贝 | 无 | 无 |
| 是否拥有数据 | 引用 | 不拥有 |
| 生命周期 | 引用有效即可 | 底层数据必须存活 |
| 能否修改 | 不能 | 不能 |

★ **关键认知**：`string_view` 只持有**指针 + 长度**，不拥有数据。它的价值是让"只读字符串参数"统一成一种类型，避免 `const char*` 和 `std::string` 两套重载。

⚠ **关键认知**：`string_view` 最常见的 bug 是**悬垂**：

```cpp
std::string_view bad() {
    std::string s = "temporary";
    return s;                 // 返回后 s 销毁，view 悬垂
}

std::string_view sv = std::string("temp");  // 临时 string 析构，sv 悬垂
```

规则很简单：**view 的寿命不能超过它引用的字符串**。函数返回、存成员、跨线程传递时，要么改成 `std::string`，要么确保底层数据生命周期足够长。

# 6. if constexpr 与初始化语句

## 6.1 if constexpr

编译期分支，未选中分支不会被实例化：

```cpp
template <typename T>
std::string describe(const T& value) {
    if constexpr (std::is_integral_v<T>) {
        return "整数: " + std::to_string(value);
    } else if constexpr (std::is_floating_point_v<T>) {
        return "浮点: " + std::to_string(value);
    } else {
        return std::string(value);      // 只在 T 是字符串类时编译
    }
}
```

★ **关键认知**：`if constexpr` 和普通 `if` 的关键区别是**未选中的分支在编译期被丢弃**。所以上面 `std::string(value)` 不会在 `T = int` 时报错。这是模板里替代 SFINAE 的现代写法。

## 6.2 if/switch 初始化语句

C++17 允许在条件前声明变量，作用域限定在分支内：

```cpp
if (auto it = m.find(key); it != m.end()) {
    std::cout << it->second;
}   // it 在这里销毁

switch (auto code = compute(); code) {
    case 0: break;
    default: break;
}
```

⚠ **关键认知**：初始化语句里的变量在整个 `if/else` 链里可见，但出了语句就销毁。不要把它当成"函数级变量"用，需要更长的生命周期就老老实实写在外面。

# 7. 最佳实践清单

| 主题 | 做法 |
|------|------|
| 资源管理 | RAII，rule of zero，优先标准容器和智能指针 |
| 所有权 | 默认 `unique_ptr`，共享才 `shared_ptr` |
| 参数传递 | 只读大对象 `const T&`，要改 `T&`，要接管 `unique_ptr<T>` |
| 返回值 | 依赖 RVO/NRVO，不要 `return std::move(local)` |
| 类型推导 | `auto` 用于迭代器和明显类型，接口处写明类型 |
| 遍历 | `for (const auto& x : c)` 为默认 |
| 错误处理 | 异常用于异常情况，可预期失败用 `optional`/`expected` |
| 常量 | `constexpr` 优先于 `const`，`const` 优先于可变 |
| 头文件 | 前置声明减少依赖，不在头文件 `using namespace` |
| 编译 | 开 `-Wall -Wextra`，开发期开 sanitizer |
| 性能 | 先测量再优化，`reserve`、移动、避免无谓拷贝 |

★ **关键认知**：现代 C++ 的核心不是"用了多少新语法"，而是**让编译器替你检查更多东西**：所有权写进类型、空值写进类型、编译期常量写进类型。类型越精确，运行期的问题越少。

⚠ **关键认知**：不要为了"现代"而现代。`auto` 满天飞会让代码失去类型信息，`std::function` 到处传会比模板慢，`shared_ptr` 滥用会让生命周期更难推理。每个特性都有适用边界，判断标准始终是**可读性、正确性、性能**三者的平衡。

# 8. 模块知识地图

```text
1 工具链与编译链接
        ▼
2 引用与 const
        ▼
┌───────────────┬───────────────┐
▼               ▼
3 类与对象      7 模板基础
▼               ▼
4 构造/析构/RAII 11 STL 概览
▼               ▼
5 继承与多态    10 智能指针与移动
▼               ▼
6 运算符重载    8 异常与 RAII
        └───────┬───────┘
                ▼
        9 命名空间与模块
                ▼
        12 现代 C++ 与总结
```

# 9. 动手练习

1. 写一个函数返回 `std::optional<std::string>`，在 `main` 里用 `if (auto v = f())` 和 `value_or` 两种方式处理。
2. 用 `std::variant<int, std::string>` 模拟一个简化的 JSON 值，用 `std::visit` 打印当前类型。
3. 写一个接收 `std::string_view` 的函数，分别传 `std::string`、字面量、`substr` 结果，并构造一个悬垂案例观察崩溃或乱码。

# 10. 小结

| 概念 | 要点 |
|------|------|
| `auto` | 编译期推导，丢引用和顶层 const |
| 范围 for | 默认 `const auto&`，遍历中不改结构 |
| 结构化绑定 | 默认拷贝，要改加 `&` |
| `optional` | 可能没有值，不带错误原因 |
| `variant` | 类型安全的联合体，用 `visit` 处理 |
| `string_view` | 不拥有的只读视图，注意生命周期 |
| `if constexpr` | 编译期分支，未选中不实例化 |
| 最佳实践 | 类型精确、RAII、先测量再优化 |

到这里，C++ 基础模块的十二篇就串成了一条线：从编译链接到对象模型，从继承多态到模板与 STL，最后落到现代 C++ 的工程习惯。接下来可以按兴趣深入 CMake 构建、并发编程或性能优化，也可以回到任意一篇把示例敲一遍——这门语言的手感，最终是靠编译器和调试器喂出来的。
