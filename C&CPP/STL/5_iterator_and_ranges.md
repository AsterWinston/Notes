迭代器是 STL 把"容器"和"算法"粘在一起的胶水。它看起来像指针，行为也像指针，但背后是一套按能力分级的抽象：`find` 只需要能往前走，`sort` 却要求能随机跳转。理解这套分级，你就能解释为什么 `list` 不能 `sort`、为什么 `vector` 一扩容所有迭代器就作废。本篇还讲 range-based for 的真实展开方式，以及 C++20 ranges 带来的惰性视图与管道写法。

# 1. 迭代器是什么

## 1.1 指针的泛化

```cpp
int arr[] = {1, 2, 3};
int* p = arr;        // 指针可以遍历数组
++p;                 // 移动到下一个元素
int x = *p;          // 解引用取值

std::vector<int> v = {1, 2, 3};
auto it = v.begin(); // 迭代器以同样的方式使用
++it;
int y = *it;
```

迭代器把"解引用 `*`、自增 `++`、比较 `!=`"这套操作抽象出来，让算法不必关心底层是数组、链表还是平衡树。

★ **关键认知**：迭代器不是"容器里的指针"，而是**一套统一的访问协议**。算法只依赖这套协议，因此同一份 `std::find` 可以作用于数组、`vector`、`list`。这就是 STL 能把算法复用起来的原因。

## 1.2 半开区间

STL 统一使用**左闭右开**区间 `[first, last)`：`first` 指向第一个元素，`last` 指向最后一个元素的**下一个位置**。

```text
[begin, end)
┌───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │   │
└───┴───┴───┴───┴───┴───┘
 ▲                   ▲
begin               end（不指向元素）
```

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
std::size_t n = std::distance(v.begin(), v.end());   // 5
for (auto it = v.begin(); it != v.end(); ++it) {     // 统一用 != end()
    // ...
}
```

★ **关键认知**：右开区间让"空区间"统一表示为 `first == last`，循环条件统一写成 `it != last`，不需要为"最后一个元素"写特判。这是 STL 设计里最优雅的约定之一，也是 `end()` 不能解引用的原因。

# 2. 五类迭代器

## 2.1 分类表

| 分类 | 能力 | 代表 |
|------|------|------|
| 输入迭代器 | 单向读，只能遍历一次 | `istream_iterator` |
| 输出迭代器 | 单向写 | `ostream_iterator`、`back_inserter` |
| 前向迭代器 | 单向读写，可多次遍历 | `forward_list` |
| 双向迭代器 | 可前进可后退 | `list`、`map`、`set` |
| 随机访问迭代器 | 可跳跃、可相减、可比较大小 | `vector`、`deque`、数组 |

## 2.2 能力如何影响算法

```cpp
std::vector<int> v = {3, 1, 2};
std::sort(v.begin(), v.end());     // 需要随机访问
auto it = v.begin() + 2;           // 只有随机访问才能 +

std::list<int> l = {3, 1, 2};
// std::sort(l.begin(), l.end());  // 编译失败：list 不是随机访问
l.sort();                          // 用成员函数

auto lit = l.begin();
// lit + 1;                        // 编译失败：双向迭代器不能跳跃
std::advance(lit, 1);              // 通用做法，内部按类别选择策略
```

★ **关键认知**：算法对迭代器的要求写在**编译期**。`std::sort` 的模板参数要求随机访问迭代器，传 `list` 直接报编译错误，而不是运行时才出错。这种"错误尽早暴露"是泛型编程的价值之一。

## 2.3 常见容器的迭代器类别

| 容器 | 迭代器类别 | 能否 `it + n` |
|------|-----------|----------------|
| `vector` | 随机访问 | 能 |
| `deque` | 随机访问 | 能 |
| `array` | 随机访问 | 能 |
| `list` | 双向 | 不能 |
| `forward_list` | 前向 | 不能 |
| `map` / `set` | 双向 | 不能 |
| `unordered_map` / `set` | 前向 | 不能 |

# 3. 迭代器失效

## 3.1 统一视角

失效的根本原因只有两个：**元素被搬到了新地址**，或者**元素被销毁了**。

```text
失效的两种来源：

1. 重新分配（vector 扩容、unordered rehash）
   ┌───┬───┬───┐
   │ 1 │ 2 │ 3 │  旧内存
   └───┴───┴───┘
        ↓ 搬到新地址，旧迭代器全部悬空

2. 元素销毁（erase、clear）
   迭代器指向的对象不复存在，解引用即未定义行为
```

## 3.2 各容器规则

| 容器 | 插入是否失效 | 删除是否失效 |
|------|--------------|--------------|
| `vector` | 扩容则全部失效，否则仅 `end()` | 删除点及之后 |
| `deque` | 中间插入全部失效，两端插入可能影响 | 视位置而定 |
| `list` | 不失效 | 仅被删元素 |
| `map` / `set` | 不失效 | 仅被删元素 |
| `unordered_*` | rehash 则全部失效 | 仅被删元素 |

```cpp
std::list<int> l = {1, 2, 3};
auto it = std::next(l.begin());     // 指向 2
l.push_front(0);                    // list 插入不影响 it
std::cout << *it;                   // 仍然是 2，安全

std::vector<int> v = {1, 2, 3};
auto vit = v.begin() + 1;           // 指向 2
v.insert(v.begin(), 0);             // 元素整体后移，vit 失效
// std::cout << *vit;               // 未定义行为
```

★ **关键认知**：链表和树的迭代器"插入不失效"，是因为节点各自独立分配，插入只改指针、不搬动已有节点。`vector` 的迭代器会失效，是因为元素必须连续存放，一插就得整体挪位。**容器怎么存，决定了迭代器怎么失效**。

⚠ **关键认知**：`vector` 的 `erase` 返回的是**被删元素之后**的新迭代器，而 `list`/`map` 的 `erase` 返回的是下一个元素的迭代器（C++11 起）。两种容器都能写 `it = c.erase(it)`，但 `vector` 会搬移后续元素，之前的迭代器全部作废。

# 4. range-based for

## 4.1 展开方式

```cpp
std::vector<int> v = {1, 2, 3};

for (int x : v) {
    std::cout << x;
}

// 大致等价于：
for (auto it = v.begin(), end = v.end(); it != end; ++it) {
    int x = *it;
    std::cout << x;
}
```

编译器会把范围 for 展开成迭代器循环，并**把 `end()` 缓存下来**。这意味着在循环体内修改容器（导致 `end()` 变化）是危险的。

## 4.2 引用与拷贝

```cpp
for (int x : v)          // 拷贝每个元素，修改 x 不影响容器
for (int& x : v)         // 引用，可修改元素
for (const int& x : v)   // 只读引用，避免拷贝，推荐
for (auto& x : v)        // 自动推导 + 引用
```

★ **关键认知**：范围 for 默认**按值拷贝**。遍历大对象（`std::string`、自定义结构体）时写 `for (auto x : v)` 会为每个元素做一次拷贝，性能可能差一个数量级。只读遍历应写 `for (const auto& x : v)`，需要修改才写 `for (auto& x : v)`。

## 4.3 遍历中修改容器

```cpp
std::vector<int> v = {1, 2, 3, 4};

// 错误：push_back 可能扩容，迭代器失效
for (int x : v) {
    if (x == 2) v.push_back(99);
}

// 正确：先收集，再统一修改
std::vector<int> toAdd;
for (int x : v) {
    if (x == 2) toAdd.push_back(99);
}
v.insert(v.end(), toAdd.begin(), toAdd.end());
```

⚠ **关键认知**：range-based for 的循环变量背后是迭代器，在循环体内 `push_back`、`erase`、`insert` 都可能让它失效。范围 for 适合**只读遍历**；需要边遍历边改结构时，改用显式迭代器并处理好返回值。

# 5. C++20 ranges

## 5.1 ranges 与 views

C++20 的 `<ranges>` 允许直接传容器，并用 `views` 构造惰性视图：

```cpp
#include <ranges>
#include <vector>

std::vector<int> v = {1, 2, 3, 4, 5, 6};

auto even = v | std::views::filter([](int x) { return x % 2 == 0; });
auto squared = even | std::views::transform([](int x) { return x * x; });

for (int x : squared) {
    std::cout << x << ' ';      // 4 16 36
}
```

## 5.2 管道写法

`|` 把多个视图串起来，读起来像数据流：

```text
v ──filter(even)──► 2 4 6 ──transform(x*x)──► 4 16 36 ──take(2)──► 4 16
```

```cpp
auto result = v
    | std::views::filter([](int x) { return x % 2 == 0; })
    | std::views::transform([](int x) { return x * x; })
    | std::views::take(2);

for (int x : result) {
    std::cout << x << ' ';      // 4 16
}
```

## 5.3 惰性求值

```cpp
auto view = v | std::views::transform([](int x) {
    std::cout << "计算 " << x << '\n';
    return x * 2;
});
// 到这里为止，没有任何元素被计算
for (int x : view) { /* 此时才逐个计算 */ }
```

★ **关键认知**：`views` 是**惰性**的，构造视图不产生新容器、不立即计算，只有遍历时才逐元素求值。这避免了中间结果的内存分配，但也意味着视图不拥有数据——底层容器销毁后，视图随即悬垂。

⚠ **关键认知**：视图引用底层数据，**不延长生命周期**。`auto view = makeVector() | std::views::filter(...);` 中临时 `vector` 在语句结束就销毁，`view` 立刻悬垂。视图只能用于"源数据确定活得更久"的场景，跨函数返回视图尤其危险。

# 6. 常见坑

| 坑 | 后果 | 修复 |
|----|------|------|
| 对 `list` 用 `std::sort` | 编译失败 | 用 `l.sort()` |
| 对双向迭代器做 `it + n` | 编译失败 | 用 `std::advance` |
| 范围 for 按值遍历大对象 | 多次拷贝 | 用 `const auto&` |
| 范围 for 内 `push_back` | 迭代器失效 | 先收集再统一修改 |
| 保存 `end()` 跨插入 | `end()` 可能失效 | 每次重新取 |
| 返回局部容器的视图 | 悬垂 | 返回容器本身 |
| 解引用 `end()` | 未定义行为 | 只比较，不解引用 |

⚠ **关键认知**：`end()` 不是"最后一个元素"，而是一个**哨兵位置**。对 `vector` 扩容后，旧的 `end()` 也一并失效；把它缓存起来跨插入使用，是很多人没意识到的 bug。

# 7. 动手练习

1. 用 `std::distance` 分别计算 `vector`、`list` 的首尾距离，观察 `vector` 是 O(1)、`list` 是 O(n)。
2. 写一个 range-based for 循环删除 `vector` 中的偶数，先故意写错看现象，再改成显式迭代器 `it = v.erase(it)`。
3. 用 C++20 ranges 从一组整数中筛出偶数、平方、取前 3 个，再用传统 `transform` + `copy_if` 实现同样功能，比较代码可读性。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 迭代器 | 统一的访问协议，不是裸指针 |
| 半开区间 | `[first, last)`，空区间 `first == last` |
| 五类迭代器 | 输入、输出、前向、双向、随机访问 |
| 算法约束 | 迭代器类别在编译期检查 |
| 失效根因 | 元素被搬移或被销毁 |
| range-based for | 展开成迭代器循环，默认按值拷贝 |
| ranges | 惰性视图、管道、不拥有数据 |
| 视图生命周期 | 不延长源数据寿命，警惕悬垂 |

下一篇进入 lambda 与函数对象，讲清楚捕获列表的几种写法、`mutable` 为什么存在、`std::function` 的类型擦除开销，以及闭包比捕获对象活得更久时会发生什么。
