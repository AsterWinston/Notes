STL 是 C++ 最值钱的部分：容器、迭代器、算法、lambda 组合起来，能覆盖日常八成的数据处理需求。但"会用 `push_back`"和"用对容器"是两回事。本篇把常用容器的底层结构、迭代器失效规则、算法与 lambda 的配合讲清楚，并给出一张能直接照着选的类型表。

# 1. 容器概览

## 1.1 顺序容器

| 容器 | 底层结构 | 随机访问 | 中间插入 | 典型场景 |
|------|----------|----------|----------|----------|
| `vector` | 连续数组 | O(1) | O(n) | 默认首选 |
| `deque` | 分段数组 | O(1) | O(n) | 两端频繁进出 |
| `list` | 双向链表 | O(n) | O(1) | 频繁中间插入删除 |
| `forward_list` | 单向链表 | O(n) | O(1) | 极致省内存 |
| `array` | 固定数组 | O(1) | 不支持 | 编译期定长 |

★ **关键认知**：**默认用 `vector`**。它的连续内存对缓存友好，实际性能常常吊打链表，即使中间插入是 O(n)。链表只有在"已经持有插入位置的迭代器、且频繁插入删除"时才占优，而这类场景比想象中少。

## 1.2 关联容器

| 容器 | 底层结构 | 有序 | 查找 | 典型场景 |
|------|----------|------|------|----------|
| `map` | 红黑树 | 是 | O(log n) | 需要有序遍历 |
| `set` | 红黑树 | 是 | O(log n) | 有序去重集合 |
| `unordered_map` | 哈希表 | 否 | 平均 O(1) | 纯查找，不关心顺序 |
| `unordered_set` | 哈希表 | 否 | 平均 O(1) | 哈希去重 |

## 1.3 容器适配器

| 适配器 | 默认底层 | 语义 |
|--------|----------|------|
| `stack` | `deque` | 后进先出 |
| `queue` | `deque` | 先进先出 |
| `priority_queue` | `vector` + 堆 | 每次取最大（默认） |

适配器不是容器，只暴露受限接口，不能遍历。

# 2. vector 与 string

## 2.1 vector 的内存增长

```cpp
std::vector<int> v;
v.reserve(100);          // 预分配 100 容量，避免反复扩容
for (int i = 0; i < 100; ++i) {
    v.push_back(i);      // 不会触发重新分配
}
```

`size()` 是元素个数，`capacity()` 是已分配空间能容纳的个数：

```text
vector 对象
┌──────┬──────┬──────────┐
│ data │ size │ capacity │
└──┬───┴──────┴──────────┘
   ▼
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┘
  ←── size = 4 ──→
  ←──────── capacity = 8 ────────→
```

扩容时通常按 1.5 或 2 倍增长，申请新内存、移动/拷贝旧元素、释放旧内存。

★ **关键认知**：`reserve` 只改容量不改大小，`resize` 会真正构造或销毁元素。已知元素数量时先 `reserve`，能避免多次扩容和元素搬移，是最容易拿到的性能优化之一。

## 2.2 迭代器失效

```cpp
std::vector<int> v = {1, 2, 3, 4};
auto it = v.begin();
v.push_back(5);          // 可能扩容，it 失效！
// *it;                  // 未定义行为
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) {
        it = v.erase(it);    // erase 返回下一个有效迭代器
    } else {
        ++it;
    }
}
```

| 操作 | 失效范围 |
|------|----------|
| `push_back` 触发扩容 | 全部迭代器、指针、引用 |
| `push_back` 未扩容 | 仅 `end()` |
| `insert`/`erase` | 插入/删除点之后的全部 |
| `reserve` | 全部 |
| `clear` | 全部 |

★ **关键认知**：`vector` 的任何可能触发扩容的操作都会让**所有**迭代器、指针、引用失效，因为元素被搬到了新内存。`erase` 返回的是下一个有效迭代器，循环删除必须用返回值更新，不能 `++it`。

⚠ **关键认知**：不要保存 `vector` 元素的指针或引用跨越 `push_back`。`int& r = v[0]; v.push_back(1); r = 10;` 在扩容后写的是已释放内存，是典型的隐蔽 bug。

## 2.3 string 要点

```cpp
std::string s = "hello";
s += " world";                  // 追加，可能扩容
s.substr(0, 5);                 // 返回新字符串（拷贝）
s.find("world");                // 找不到返回 npos
if (s.find("xyz") == std::string::npos) { }
const char* raw = s.c_str();    // 只读视图，s 修改后可能失效
std::string_view sv = s;        // C++17 只读视图，不拷贝
```

★ **关键认知**：`c_str()` 返回的指针在字符串修改或销毁后失效，不要长期保存。需要只读、不拷贝地传递字符串片段时，用 `std::string_view`，但要保证底层字符串活得比 view 长。

# 3. map 与 unordered_map

```cpp
std::map<std::string, int> ages;
ages["Tom"] = 18;               // 不存在则插入默认值
ages.insert({"Jerry", 20});     // 已存在则不覆盖
if (auto it = ages.find("Tom"); it != ages.end()) {
    std::cout << it->second << '\n';
}

for (const auto& [name, age] : ages) {   // 结构化绑定
    std::cout << name << ": " << age << '\n';
}
```

| 维度 | `map` | `unordered_map` |
|------|-------|-----------------|
| 底层 | 红黑树 | 哈希表 |
| 查找 | O(log n) | 平均 O(1)，最坏 O(n) |
| 有序遍历 | 支持 | 不支持 |
| 迭代器稳定性 | 插入不失效 | rehash 后全部失效 |
| 自定义键 | 需要 `<` | 需要 `hash` 和 `==` |
| 内存 | 较高（节点） | 较高（桶 + 节点） |

★ **关键认知**：`operator[]` 在键不存在时会**插入一个默认值**，这会改变容器大小并可能触发 rehash。只想查询时用 `find` 或 C++20 的 `contains`，不要用 `operator[]` 探测。

⚠ **关键认知**：`unordered_map` 的哈希函数质量直接影响性能。用自定义类型作键时，要同时提供 `std::hash` 特化和 `operator==`；哈希分布不均会让查找退化到 O(n)。

# 4. 迭代器

## 4.1 五个分类

| 分类 | 能力 | 代表 |
|------|------|------|
| 输入迭代器 | 单向读 | `istream_iterator` |
| 输出迭代器 | 单向写 | `ostream_iterator` |
| 前向迭代器 | 单向读写、多次遍历 | `forward_list` |
| 双向迭代器 | 可前后移动 | `list`、`map` |
| 随机访问迭代器 | 可跳跃、比较大小 | `vector`、`deque` |

```cpp
std::vector<int> v = {3, 1, 2};
std::sort(v.begin(), v.end());          // 需要随机访问迭代器
auto it = v.begin() + 2;                // 只有随机访问才能 +

std::list<int> l = {3, 1, 2};
// std::sort(l.begin(), l.end());       // 错误：list 不是随机访问
l.sort();                                // 用成员函数
```

★ **关键认知**：算法的接口按**最低迭代器要求**设计。`std::find` 只需要输入迭代器，`std::sort` 需要随机访问。把 `list` 传给 `std::sort` 会编译失败，不是运行时问题——这是在编译期就能发现的类型约束。

## 4.2 区间约定

STL 统一使用**左闭右开**区间 `[first, last)`：

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
std::sort(v.begin(), v.end());        // end() 指向最后一个元素之后
std::size_t n = std::distance(v.begin(), v.end());   // 5
```

```text
[begin, end)
┌───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │   │
└───┴───┴───┴───┴───┴───┘
 ▲                   ▲
begin               end（不指向元素）
```

★ **关键认知**：右开区间让"空区间"可以统一表示为 `first == last`，循环条件也统一成 `it != last`，不需要额外处理"最后一个元素"。这是 STL 设计里最优雅的约定之一。

# 5. 算法

```cpp
#include <algorithm>

std::vector<int> v = {5, 2, 8, 1, 9};

std::sort(v.begin(), v.end());                   // 升序
std::sort(v.begin(), v.end(), std::greater<>{}); // 降序

auto it = std::find(v.begin(), v.end(), 8);      // 查找
bool has = std::any_of(v.begin(), v.end(),
                       [](int x) { return x > 7; });
int count = std::count_if(v.begin(), v.end(),
                          [](int x) { return x % 2 == 0; });

std::vector<int> out;
std::transform(v.begin(), v.end(), std::back_inserter(out),
               [](int x) { return x * 2; });
```

| 需求 | 算法 |
|------|------|
| 排序 | `sort`、`stable_sort`、`partial_sort` |
| 查找 | `find`、`find_if`、`binary_search` |
| 计数 | `count`、`count_if` |
| 条件判断 | `all_of`、`any_of`、`none_of` |
| 变换 | `transform`、`copy`、`remove_if` |
| 累积 | `accumulate`、`reduce` |
| 最值 | `min_element`、`max_element` |

★ **关键认知**：优先用算法而不是手写循环。算法名字直接表达意图（`any_of` 比一个带 `break` 的 for 更清楚），而且实现经过充分优化。手写循环只在逻辑确实无法用算法表达时才用。

⚠ **关键认知**：`std::remove` / `remove_if` **不删除元素**，只是把保留的元素前移，返回新的逻辑结尾。真正的删除要配合 `erase`，也就是常说的 erase-remove 惯用法：

```cpp
v.erase(std::remove_if(v.begin(), v.end(),
                       [](int x) { return x < 3; }),
        v.end());
```

# 6. lambda

```cpp
int threshold = 5;

auto isBig = [threshold](int x) {          // 按值捕获
    return x > threshold;
};

auto isBigRef = [&threshold](int x) {      // 按引用捕获
    return x > threshold;
};

auto generic = [](auto a, auto b) {        // C++14 泛型 lambda
    return a + b;
};
```

| 捕获 | 写法 | 注意 |
|------|------|------|
| 按值 | `[x]` | 拷贝，lambda 内修改不影响外部 |
| 按引用 | `[&x]` | 外部变量必须活得比 lambda 长 |
| 全部按值 | `[=]` | 隐式捕获，C++20 起不推荐 |
| 全部按引用 | `[&]` | 悬垂风险高 |
| 移动捕获 | `[p = std::move(ptr)]` | C++14，接管所有权 |
| 初始化捕获 | `[n = 0]() mutable { return ++n; }` | 有状态 lambda |

★ **关键认知**：lambda 的 `operator()` 默认是 `const` 的，按值捕获的变量在 lambda 内不能修改，除非加 `mutable`。这也是"为什么我的计数器不累加"的常见原因。

⚠ **关键认知**：按引用捕获的 lambda 不能比被捕获对象活得更久。把 lambda 存进容器、传给异步任务、注册为回调时，优先按值捕获，或用 `shared_ptr` 明确共享所有权。悬垂引用在 lambda 里非常难查。

# 7. 选型表

| 需求 | 首选 | 备选 |
|------|------|------|
| 通用序列 | `vector` | `deque` |
| 频繁头部插入 | `deque` | `list` |
| 有序键值 | `map` | 排序后的 `vector` |
| 纯查找键值 | `unordered_map` | `map` |
| 去重 | `unordered_set` | `set` |
| 固定长度 | `array` | `vector` + `reserve` |
| 栈/队列 | `stack`/`queue` | `deque` 手动 |
| 只读字符串片段 | `string_view` | `const std::string&` |

# 8. 常见坑

| 坑 | 后果 | 修复 |
|----|------|------|
| 循环中 `push_back` 持有旧迭代器 | 迭代器失效 | 用索引或先 `reserve` |
| `erase` 后继续 `++it` | 跳过元素或崩溃 | 用 `erase` 返回值 |
| `map::operator[]` 查询 | 意外插入 | 用 `find`/`contains` |
| `remove` 后忘了 `erase` | 元素没删 | erase-remove 惯用法 |
| lambda 按引用捕获局部变量后异步使用 | 悬垂引用 | 按值捕获或 `shared_ptr` |
| 用 `list` 调 `std::sort` | 编译失败 | 用 `l.sort()` |
| `vector<bool>` 当普通容器 | 代理引用、不能取地址 | 用 `deque<bool>` 或 `vector<char>` |

⚠ **关键认知**：`std::vector<bool>` 是标准库里的特化，按位存储，`operator[]` 返回代理对象而不是 `bool&`。`auto& b = v[0];` 编译不过，取地址也不行。需要真正布尔数组时换 `std::deque<bool>` 或 `std::vector<char>`。

# 9. 动手练习

1. 读入 1000 个随机整数放进 `vector`，分别在不 `reserve` 和 `reserve(1000)` 的情况下统计扩容次数（用 `capacity()` 变化观察）。
2. 用 `map` 统计一段文本中每个单词出现次数，再换成 `unordered_map`，比较输出顺序和查找写法。
3. 用 `remove_if` + `erase` 删除 `vector` 中所有偶数，再用 `transform` 把剩余元素翻倍输出。

# 10. 小结

| 概念 | 要点 |
|------|------|
| 默认容器 | `vector`，连续内存、缓存友好 |
| 扩容 | 先 `reserve` 避免反复搬移 |
| 迭代器失效 | 扩容全失效，`erase` 用返回值 |
| `map` vs `unordered_map` | 有序 O(log n) vs 无序平均 O(1) |
| 区间 | 左闭右开 `[first, last)` |
| 算法 | 优先于手写循环 |
| erase-remove | `remove` 只前移，必须配 `erase` |
| lambda | 默认 const，异步场景慎用引用捕获 |

下一篇进入现代 C++ 与总结，讲清楚 `auto`、范围 for、结构化绑定、`optional`/`variant`、`string_view` 的正确用法，并给出贯穿整个模块的最佳实践清单。
