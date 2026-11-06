STL 算法是"把意图写进函数名"的典范。`any_of` 比一个带 `break` 的 for 循环更清楚，`transform` 比手写逐元素赋值更不容易出错。但算法库有一个反直觉的设计：`std::remove` 并不会删除元素。不理解这一点，就会写出"调了 remove 但容器 size 没变"的困惑。本篇从排序、查找讲到变换与累积，重点拆开 erase-remove 惯用法。

# 1. 算法库的全貌

## 1.1 头文件

| 头文件 | 内容 |
|--------|------|
| `<algorithm>` | 排序、查找、变换、删除、比较 |
| `<numeric>` | `accumulate`、`reduce`、`iota`、`inner_product` |
| `<functional>` | `greater`、`multiplies` 等仿函数 |
| `<iterator>` | `back_inserter`、`distance`、`advance` |

## 1.2 算法与区间

STL 算法接收的是**一对迭代器** `[first, last)`，而不是容器本身。这带来两个后果：算法可以作用在原生数组、`vector`、`deque` 等任何提供合适迭代器的序列上；调用时必须保证区间有效。

```cpp
#include <algorithm>
#include <vector>

std::vector<int> v = {5, 2, 8, 1, 9};
std::sort(v.begin(), v.end());     // 传容器，得到的是迭代器区间

int arr[] = {3, 1, 2};
std::sort(arr, arr + 3);           // 原生数组同样可用
```

★ **关键认知**：算法按**最低迭代器要求**设计。`std::find` 只需要输入迭代器，`std::sort` 需要随机访问迭代器。把 `std::list` 传给 `std::sort` 会直接编译失败——这不是运行时问题，而是类型系统在编译期就告诉你不合适。

# 2. 排序

## 2.1 sort

```cpp
#include <algorithm>

std::vector<int> v = {5, 2, 8, 1, 9};
std::sort(v.begin(), v.end());                    // 升序
std::sort(v.begin(), v.end(), std::greater<>{});  // 降序
```

`std::sort` 的平均复杂度是 O(n log n)，最坏也是 O(n log n)（C++11 起要求内省排序）。它**不稳定**：相等元素的相对顺序可能改变。

## 2.2 其他排序

| 算法 | 复杂度 | 特点 |
|------|--------|------|
| `sort` | O(n log n) | 不稳定，最快，默认首选 |
| `stable_sort` | O(n log² n) 或 O(n log n) | 稳定，相等元素保持原序 |
| `partial_sort` | O(n log k) | 只保证前 k 个有序 |
| `nth_element` | 平均 O(n) | 只保证第 n 位就位，左边都不大于它 |
| `is_sorted` | O(n) | 检查是否已排序 |

```cpp
std::vector<int> v = {9, 1, 8, 2, 7, 3};

std::partial_sort(v.begin(), v.begin() + 3, v.end());
// 前 3 个是最小的 3 个，且已升序

std::nth_element(v.begin(), v.begin() + 3, v.end());
// v[3] 是第 4 小的元素，左侧都 <= 它，右侧都 >= 它
```

★ **关键认知**：只需要"前 k 小"时不要对整个数组 `sort`。`partial_sort` 的 O(n log k) 和 `nth_element` 的平均 O(n) 在大数据量下差距明显。求 Top-K 或中位数时，这两个算法是正确工具。

## 2.3 自定义比较

```cpp
struct Person {
    std::string name;
    int age;
};

std::vector<Person> people = {{"Tom", 30}, {"Jerry", 20}};

std::sort(people.begin(), people.end(),
          [](const Person& a, const Person& b) {
              return a.age < b.age;      // 按年龄升序
          });
```

比较器同样必须满足**严格弱序**。写 `<=` 会让相等元素也被判定为"小于"，`sort` 可能越界访问，直接崩溃。

⚠ **关键认知**：`sort` 的比较器返回 `bool`，语义是"a 是否应排在 b 前面"。用 `<=` 或 `>=` 违反严格弱序，是 `sort` 段错误最常见的原因。要排序的字段相等时，写法必须是 `a.key < b.key`，绝不能是 `a.key <= b.key`。

# 3. 查找

## 3.1 find 与 find_if

```cpp
std::vector<int> v = {5, 2, 8, 1, 9};

auto it = std::find(v.begin(), v.end(), 8);      // 线性查找
if (it != v.end()) {
    std::cout << "找到，下标 " << (it - v.begin()) << '\n';
}

auto even = std::find_if(v.begin(), v.end(),
                         [](int x) { return x % 2 == 0; });
```

两者都是 O(n)，对无序数据只能这样线性扫描。

## 3.2 binary_search 与 lower_bound

数据**已排序**时，用二分查找把复杂度降到 O(log n)：

```cpp
std::vector<int> v = {1, 2, 2, 2, 5, 8};   // 必须已排序

bool has = std::binary_search(v.begin(), v.end(), 5);  // true/false

auto lo = std::lower_bound(v.begin(), v.end(), 2);     // 第一个 >= 2
auto hi = std::upper_bound(v.begin(), v.end(), 2);     // 第一个 > 2
std::size_t count = hi - lo;                           // 等于 2 的个数：3
```

★ **关键认知**：`binary_search` 只回答"在不在"，不告诉你位置。要位置用 `lower_bound`，要"所有等于 x 的元素"用 `equal_range`。对未排序数据调用二分查找不会报错，只会给出错误结果——这是很难查的 bug。

## 3.3 条件判断

```cpp
bool anyEven = std::any_of(v.begin(), v.end(),
                           [](int x) { return x % 2 == 0; });
bool allPositive = std::all_of(v.begin(), v.end(),
                               [](int x) { return x > 0; });
bool noneNegative = std::none_of(v.begin(), v.end(),
                                 [](int x) { return x < 0; });
```

| 算法 | 含义 | 空区间时 |
|------|------|----------|
| `all_of` | 是否全部满足 | true（空真） |
| `any_of` | 是否至少一个满足 | false |
| `none_of` | 是否全部不满足 | true |

★ **关键认知**：`all_of` 对空区间返回 `true`，这是数学上"空真"的约定，符合逻辑但常让初学者意外。写断言或校验逻辑时，要先判断容器是否为空，否则"空集合全部满足"可能掩盖问题。

# 4. 变换 transform

`transform` 把区间内每个元素映射成新值，可原地也可写入另一容器：

```cpp
#include <algorithm>
#include <iterator>

std::vector<int> v = {1, 2, 3, 4};

// 原地翻倍
std::transform(v.begin(), v.end(), v.begin(),
               [](int x) { return x * 2; });

// 写入另一个容器：必须用 back_inserter，否则目标没有空间
std::vector<int> out;
std::transform(v.begin(), v.end(), std::back_inserter(out),
               [](int x) { return x + 1; });
```

⚠ **关键认知**：`transform` 的输出区间必须**已经能容纳结果**。如果目标 `vector` 是空的却直接传 `out.begin()`，就是往没有元素的内存里写，未定义行为。写入空容器时必须用 `std::back_inserter`，它会自动 `push_back`。

# 5. 累积 accumulate

```cpp
#include <numeric>

std::vector<int> v = {1, 2, 3, 4, 5};

int sum = std::accumulate(v.begin(), v.end(), 0);          // 15
int product = std::accumulate(v.begin(), v.end(), 1,
                              std::multiplies<>{});         // 120
std::vector<std::string> words = {"a", "b"};
std::string joined = std::accumulate(
    words.begin(), words.end(), std::string{},   // 初值决定返回类型
    [](const std::string& acc, const std::string& s) {
        return acc + s;
    });
```

★ **关键认知**：`accumulate` 的第三个参数是**初值，也决定了返回类型**。累加 `double` 却把初值写成 `0`，整个计算会按 `int` 进行，小数被截断。想要浮点结果，初值必须写 `0.0`。

⚠ **关键认知**：对浮点数用 `accumulate` 存在累积误差，而且不保证结合顺序。需要精确或可并行的求和时，考虑 `std::reduce`（C++17，允许任意顺序）或 Kahan 求和，但要先确认数值稳定性要求。

# 6. remove_if 与 erase-remove

## 6.1 remove 为什么不删除

`std::remove` 和 `std::remove_if` 只做一件事：把**要保留的元素前移**，返回新的逻辑结尾。它们不改变容器大小，因为算法只拿到迭代器，根本不知道容器的 `size`。

```cpp
std::vector<int> v = {1, 2, 3, 4, 5, 6};
auto newEnd = std::remove_if(v.begin(), v.end(),
                             [](int x) { return x % 2 == 0; });
// v 现在是 {1, 3, 5, ?, ?, ?}，size 仍是 6
// newEnd 指向第 4 个位置
```

```text
删除前：
┌───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │
└───┴───┴───┴───┴───┴───┘

remove_if 之后（保留奇数，前移）：
┌───┬───┬───┬───┬───┬───┐
│ 1 │ 3 │ 5 │ ? │ ? │ ? │   size 仍为 6
└───┴───┴───┴───┴───┴───┘
            ▲
          newEnd
```

## 6.2 erase-remove 惯用法

真正删除要两步：先 `remove_if` 把保留元素前移，再用 `erase` 截断尾部：

```cpp
v.erase(std::remove_if(v.begin(), v.end(),
                       [](int x) { return x % 2 == 0; }),
        v.end());
```

★ **关键认知**：erase-remove 的本质是"**移动 + 截断**"。`remove` 把要保留的元素紧凑地搬到前面，返回新结尾；`erase` 从新结尾删到旧结尾。这样每次删除只需 O(1) 的移动，整体 O(n)，比循环里逐个 `erase` 的 O(n²) 快得多。

## 6.3 C++20 的 erase / erase_if

C++20 为容器提供了成员版的 `std::erase` 和 `std::erase_if`，一步到位：

```cpp
#include <vector>

std::vector<int> v = {1, 2, 3, 4, 5, 6};
std::erase_if(v, [](int x) { return x % 2 == 0; });  // 直接删除，size 变小
std::erase(v, 3);                                     // 删除所有等于 3 的元素
```

⚠ **关键认知**：C++20 的 `std::erase_if(v, pred)` 是**真正的删除**，和 `std::remove_if` 完全不同。看到 `erase` 才改变 `size`，看到 `remove` 只搬移元素。名字相近、语义相差一个数量级，阅读旧代码时尤其要分清。

# 7. 谓词与 lambda

"谓词"就是返回 `bool` 的可调用对象，用来描述条件：

```cpp
auto isAdult = [](const Person& p) { return p.age >= 18; };

std::count_if(people.begin(), people.end(), isAdult);
std::find_if(people.begin(), people.end(), isAdult);
std::any_of(people.begin(), people.end(), isAdult);
```

谓词应当是**无副作用**的纯函数。算法可能按任意次数、任意顺序调用它，依赖调用次数或修改外部状态的谓词会得到不可预期的结果。

⚠ **关键认知**：不要在被传给算法的 lambda 里修改容器本身。比如在 `remove_if` 的谓词里 `push_back`，会在 `remove_if` 遍历区间时改变容器结构，导致迭代器失效和未定义行为。谓词只读，改动放到算法之外。

# 8. 复杂度速查

| 算法 | 复杂度 |
|------|--------|
| `sort` | O(n log n) |
| `stable_sort` | O(n log² n) 或 O(n log n) |
| `partial_sort` | O(n log k) |
| `nth_element` | 平均 O(n) |
| `find` / `find_if` / `count_if` | O(n) |
| `binary_search` / `lower_bound` | O(log n) |
| `transform` / `copy` / `remove_if` | O(n) |
| `accumulate` | O(n) |

# 9. 常见坑

| 坑 | 后果 | 修复 |
|----|------|------|
| `remove` 后不 `erase` | 元素没删，size 不变 | erase-remove |
| 比较器用 `<=` | `sort` 越界崩溃 | 用 `<` |
| 对未排序数据二分 | 结果错误 | 先排序 |
| `transform` 写空容器 | 越界写 | 用 `back_inserter` |
| `accumulate` 初值类型错 | 精度丢失 | 初值写 `0.0` |
| 谓词有副作用 | 结果不可预期 | 谓词保持纯函数 |
| 循环里逐个 `erase` | O(n²) | erase-remove |

# 10. 动手练习

1. 读入 20 个整数，用 `sort` 升序、`stable_sort` 按奇偶分组，观察相等元素的相对顺序是否改变。
2. 用 `remove_if` + `erase` 删除 `vector` 中所有负数，再用 C++20 的 `std::erase_if` 重写，比较代码长度。
3. 用 `accumulate` 计算一组 `double` 的平均值，故意把初值写成 `0`，观察结果与 `0.0` 的差异。

# 11. 小结

| 概念 | 要点 |
|------|------|
| 区间 | 算法接收 `[first, last)`，不接收容器 |
| `sort` | O(n log n)，不稳定，比较器要严格弱序 |
| Top-K | `partial_sort` / `nth_element`，别全排序 |
| 二分 | 只能用于已排序数据 |
| `transform` | 输出空容器必须用 `back_inserter` |
| `accumulate` | 初值决定返回类型 |
| `remove` | 只前移，不删除，size 不变 |
| erase-remove | `remove_if` 配 `erase`，O(n) |
| C++20 | `std::erase_if` 才是真删除 |

下一篇进入迭代器与 ranges，讲清楚五类迭代器的能力差异、失效规则的统一视角、range-based for 的展开方式，以及 C++20 ranges 的视图与管道写法。
