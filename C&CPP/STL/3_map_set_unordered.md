关联容器解决的是"按键找值"这一问题。`map` 和 `set` 用红黑树保证有序，`unordered_map` 和 `unordered_set` 用哈希表换取平均常数时间。两者接口很像，底层和行为却差别很大：一个能范围查询、插入不失效；一个查找更快、但 rehash 会让迭代器全部作废。本篇把这两类容器的机制、比较器与哈希函数的写法、以及自定义类型作键的条件讲清楚。

# 1. 关联容器总览

## 1.1 有序与无序

| 维度 | 有序（`map`/`set`） | 无序（`unordered_map`/`unordered_set`） |
|------|---------------------|------------------------------------------|
| 底层结构 | 红黑树 | 哈希表（桶数组 + 节点） |
| 查找 | O(log n) | 平均 O(1)，最坏 O(n) |
| 遍历顺序 | 按键升序 | 不保证，可能随 rehash 改变 |
| 范围查询 | 支持（`lower_bound`） | 不支持 |
| 键的要求 | 可比较（`<`） | 可哈希 + 可判等（`==`） |
| 迭代器稳定性 | 插入不失效 | rehash 后全部失效 |

`set` 可以看成"只有键、没有值的 `map`"，`multimap`/`multiset` 则允许重复键。日常用得最多的是 `map`、`set`、`unordered_map`、`unordered_set`。

## 1.2 接口速览

| 操作 | `map` | `unordered_map` | 说明 |
|------|-------|-----------------|------|
| 插入 | `insert`、`emplace` | 同左 | 已存在则不覆盖 |
| 查找 | `find`、`count` | 同左 | 找不到返回 `end()` |
| 是否存在 | `contains`（C++20） | 同左 | 比 `count` 更直观 |
| 下标访问 | `operator[]` | 同左 | 不存在会插入默认值 |
| 删除 | `erase` | 同左 | 可按迭代器或键删除 |
| 有序操作 | `lower_bound`、`upper_bound` | 无 | 依赖有序性 |

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> ages;
    ages["Tom"] = 18;                 // 不存在则插入默认值再赋值
    ages.insert({"Jerry", 20});       // 已存在则不覆盖
    ages.emplace("Spike", 3);         // 原地构造
    for (const auto& [name, age] : ages) {   // 结构化绑定，按键有序
        std::cout << name << ": " << age << '\n';
    }
    return 0;
}
```

★ **关键认知**：`map` 的迭代器指向 `std::pair<const Key, T>`，键是 `const` 的，不能通过迭代器修改键。因为键决定了元素在红黑树中的位置，改了键就破坏了整棵树的结构。需要改键时只能删除旧节点再插入新节点。

# 2. map 与 set 的底层

## 2.1 红黑树

`map`/`set` 的底层通常是红黑树，一种近似平衡的二叉搜索树。它保证从根到叶的最长路径不超过最短路径的两倍，因此树高始终是 O(log n)。

```text
          ┌───┐
          │ 4 │  黑
          └─┬─┘
      ┌─────┴─────┐
    ┌─┴─┐       ┌─┴─┐
    │ 2 │ 红    │ 6 │ 红
    └─┬─┘       └─┬─┘
   ┌──┴──┐     ┌──┴──┐
 ┌─┴─┐ ┌─┴─┐ ┌─┴─┐ ┌─┴─┐
 │ 1 │ │ 3 │ │ 5 │ │ 7 │
 └───┘ └───┘ └───┘ └───┘
 每次比较排除一半子树 → 查找 O(log n)
```

插入和删除后，红黑树通过旋转和重新染色恢复平衡。相比普通二叉搜索树，它避免了"有序数据依次插入导致退化成链表"的问题。

★ **关键认知**：红黑树的 O(log n) 是**最坏情况保证**，不是平均。对实时系统或对延迟敏感的场景，这个确定性比哈希表的"平均 O(1)、最坏 O(n)"更可靠。

## 2.2 常用操作

```cpp
#include <map>

std::map<int, std::string> m = {{1, "one"}, {2, "two"}, {3, "three"}};

m.find(2);                 // 返回迭代器，找不到是 end()
m.count(2);                // 0 或 1
m.contains(2);             // C++20，返回 bool
m.lower_bound(2);          // 第一个 >= 2 的位置
m.upper_bound(2);          // 第一个 > 2 的位置
m.equal_range(2);          // pair(lower, upper)，范围查询常用

m.erase(2);                // 按键删除，返回删除个数
m.size();                  // 元素个数
m.empty();                 // 是否为空
```

范围查询是 `map` 相对 `unordered_map` 的核心优势：

```cpp
auto lo = m.lower_bound(3);   // 第一个键 >= 3
auto hi = m.upper_bound(5);   // 第一个键 > 5
for (auto it = lo; it != hi; ++it) {
    // 遍历键在 [3, 5] 内的元素
}
```

## 2.3 operator[] 的陷阱

`operator[]` 的语义是"**不存在就插入一个默认值，再返回引用**"。它要求值类型可默认构造。

```cpp
std::map<std::string, int> m;
int x = m["missing"];   // 插入 {"missing", 0}，size 从 0 变 1
bool empty = m.empty(); // false！查询竟然改变了容器
```

```text
执行 m["missing"] 之前：{}          size = 0
执行之后：            {"missing":0}  size = 1
```

★ **关键认知**：只想查询时绝不要用 `operator[]`。它会插入默认值、改变 `size`、可能触发哈希表 rehash，还会对值类型提出"可默认构造"的要求。查存在性用 `find` 或 `contains`，写入才用 `operator[]`。

## 2.4 try_emplace 与 insert_or_assign

C++17 补上两个语义更明确的接口：

| 接口 | 键已存在时 | 键不存在时 |
|------|------------|------------|
| `insert` / `emplace` | 不覆盖 | 插入 |
| `operator[]` | 返回引用，可改 | 插入默认值 |
| `insert_or_assign` | 覆盖值 | 插入 |
| `try_emplace` | 不构造值 | 原地构造 |

```cpp
std::map<std::string, std::string> m;
m.try_emplace("key", "value");      // 不存在才构造
m.insert_or_assign("key", "new");   // 存在则覆盖，不存在则插入
```

⚠ **关键认知**：`try_emplace` 的关键优势是"**键已存在时不构造值**"。如果用 `emplace("key", expensive())`，`expensive()` 会先被求值，即使最终没插入也白算一遍。`try_emplace` 把参数直接转发给构造函数，避免了这种浪费。

# 3. 比较器与自定义键

## 3.1 自定义比较器

`map`/`set` 默认用 `std::less<Key>`，也就是 `operator<`。想按键降序排列时，传入自定义比较器：

```cpp
#include <map>

struct Desc {
    bool operator()(int a, int b) const {
        return a > b;            // 降序：注意是 >
    }
};
std::map<int, std::string, Desc> m;   // 键从大到小遍历

auto cmp = [](int a, int b) { return a > b; };          // C++20
std::map<int, std::string, decltype(cmp)> n(cmp);
```

比较器必须满足**严格弱序**：对任意 a、b，`cmp(a, b)` 和 `cmp(b, a)` 不能同时为真。写成 `<=` 是经典错误——它会让"相等"也返回真，破坏树的排序假设。

⚠ **关键认知**：比较器写 `<=` 或 `>=` 是 `map`/`set` 最隐蔽的 bug 之一。标准要求严格弱序，`<=` 违反"不可自反"，表现为某些元素"插进去了却查不到"。判断相等要用 `!(a < b) && !(b < a)`，而不是 `a <= b`。

## 3.2 自定义类型作键

用结构体作 `map` 的键，需要提供比较方式，二选一：

```cpp
struct Point { int x; int y; };

// 方式一：重载 operator<
bool operator<(const Point& a, const Point& b) {
    return a.x < b.x || (a.x == b.x && a.y < b.y);
}

// 方式二：外部比较器
struct PointLess {
    bool operator()(const Point& a, const Point& b) const {
        return a.x < b.x || (a.x == b.x && a.y < b.y);
    }
};

std::map<Point, int> m1;              // 用方式一
std::map<Point, int, PointLess> m2;   // 用方式二
```

★ **关键认知**：比较器必须能对任意两个键给出**稳定且一致**的顺序。只比较 `x` 而忽略 `y`，会让所有 `x` 相同的点被视为"相等"，后插入的键无法进入容器。写自定义键的比较器时，一定要把所有参与身份判断的字段都比到。

# 4. unordered 的底层

## 4.1 哈希表与桶

`unordered_map` 内部是一个"桶数组"，每个桶挂着若干节点：

```text
键 "apple" --hash--> 42 --% bucket_count--> bucket[2]

bucket[0] → 空
bucket[1] → ("kiwi":3) → ("pear":5)   ← 冲突，同桶链式存放
bucket[2] → ("apple":1)
```

查找分两步：先算哈希值定位桶，再在桶内逐个比较键。如果哈希均匀，每个桶平均只有常数个元素，所以平均 O(1)。

## 4.2 哈希函数

标准库为常见类型提供了 `std::hash` 特化。用自定义类型作键时，需要提供哈希函数和 `operator==`：

```cpp
#include <functional>
#include <unordered_map>

struct Point {
    int x; int y;
    bool operator==(const Point& o) const { return x == o.x && y == o.y; }
};

struct PointHash {
    std::size_t operator()(const Point& p) const {
        std::size_t h1 = std::hash<int>{}(p.x);
        std::size_t h2 = std::hash<int>{}(p.y);
        return h1 ^ (h2 << 1);       // 混合两个字段，避免大量冲突
    }
};

std::unordered_map<Point, int, PointHash> m;
```

从 C++20 起，也可以只特化 `std::hash<Point>`，让 `unordered_map` 使用默认哈希。

★ **关键认知**：哈希函数的好坏直接决定 `unordered_map` 的性能。一个"合法但很差"的哈希（比如永远返回 0）会让所有键挤进同一个桶，查找退化成 O(n)。写哈希时要把所有参与判等的字段都混进去，并尽量让结果分布均匀。

## 4.3 负载因子与 rehash

负载因子 = 元素个数 / 桶个数。超过阈值（默认约 1.0）时，容器会扩容并把所有元素重新分配到新桶，这就是 rehash。

```cpp
std::unordered_map<int, int> m;
m.max_load_factor(0.7f);                 // 调低阈值，减少冲突
m.reserve(1000);                         // 预留至少 1000 个元素的空间
std::cout << m.bucket_count() << '\n';   // 当前桶数
std::cout << m.load_factor() << '\n';    // 当前负载因子
```

```text
rehash 前：桶少，链长          rehash 后：桶翻倍，元素重新散布
bucket[0] → A → B → C          bucket[0] → A
bucket[1] → D                  bucket[1] → C
                               bucket[2] → B
                               bucket[3] → D
```

⚠ **关键认知**：rehash 会让**所有迭代器失效**，但元素的引用和指针仍然有效——节点本身没被销毁，只是桶指针重排了。已知要插入大量元素时，先 `reserve` 避免多次 rehash，是 `unordered_map` 最实用的优化。

# 5. 性能对比与选择

| 场景 | 首选 | 理由 |
|------|------|------|
| 需要按键有序遍历 | `map`/`set` | 哈希容器无序 |
| 需要范围查询 | `map`/`set` | `lower_bound` 只有有序容器有 |
| 纯单键查找、数据量大 | `unordered_map`/`unordered_set` | 平均 O(1) |
| 键的哈希很难写好 | `map`/`set` | 不依赖哈希质量 |
| 对最坏延迟敏感 | `map`/`set` | O(log n) 有保证 |
| 元素很少（几十个） | `vector` + 线性查找 | 常数小，缓存友好 |

★ **关键认知**：`unordered_map` 不一定比 `map` 快。元素少时 `map` 的常数开销可能更小；哈希函数差时 `unordered_map` 会退化；而 `map` 的有序性还能顺便支持范围查询。选之前先问清楚"要不要顺序"和"键好不好哈希"。

# 6. 常见坑

| 坑 | 后果 | 修复 |
|----|------|------|
| 用 `operator[]` 查询 | 意外插入默认值 | 用 `find`/`contains` |
| 比较器写 `<=` | 元素查不到、行为异常 | 严格弱序，用 `<` |
| 自定义键只比较部分字段 | 键被误判相等 | 比较全部身份字段 |
| 哈希只用一个字段 | 冲突严重，退化 O(n) | 混合所有字段 |
| rehash 后继续用旧迭代器 | 未定义行为 | 先 `reserve`，或重新取迭代器 |
| 忘记给自定义键写 `==` | 编译失败 | 补 `operator==` |
| 边遍历边 `erase` | 迭代器失效 | 用 `it = m.erase(it)` |

⚠ **关键认知**：`map` 和 `unordered_map` 的 `erase` 都会返回下一个有效迭代器（C++11 起），所以循环删除要写成 `it = m.erase(it)`。旧教材里"先 `++it` 再 `erase(prev)`"是 C++98 的兼容技巧，现代 C++ 不需要，也不该再用。

# 7. 动手练习

1. 用 `map<string, int>` 统计一段文本的词频，再改成 `unordered_map`，比较输出顺序并统计两种实现的耗时。
2. 定义一个 `Point{x, y}`，分别作为 `map` 的键和 `unordered_map` 的键，实现比较器、哈希函数和 `operator==`，验证插入与查找。
3. 用 `lower_bound`/`upper_bound` 从 `map<int, string>` 中取出键在 `[10, 20]` 的所有元素，再思考用 `unordered_map` 时为什么做不到。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 有序容器 | 红黑树，O(log n)，支持范围查询 |
| 无序容器 | 哈希表，平均 O(1)，顺序不保证 |
| `operator[]` | 不存在会插入默认值，查询别用 |
| 比较器 | 严格弱序，用 `<` 不用 `<=` |
| 哈希函数 | 要混合所有字段、分布均匀 |
| rehash | 迭代器全失效，引用仍有效 |
| `try_emplace` | 键存在时不构造值 |
| 循环删除 | `it = m.erase(it)` |

下一篇进入算法库，讲清楚 `sort`、`find`、`transform`、`accumulate`、`remove_if` 的用法与复杂度，以及为什么 `remove` 不删除元素、erase-remove 惯用法到底在做什么。
