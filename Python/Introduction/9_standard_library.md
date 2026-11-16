Python 的"自带电池"里有大量高质量标准库，熟练掌握几个高频模块，能让代码短一半、bug 少一半。本篇挑出六个最常用也最容易用错的：`collections`、`itertools`、`functools`、`datetime`、`re`、`argparse`，每个都给出典型用法和陷阱。

# 1. collections

## 1.1 Counter

```python
from collections import Counter

words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
counter = Counter(words)

print(counter)                 # Counter({'apple': 3, 'banana': 2, 'cherry': 1})
print(counter["missing"])      # 0，不存在的键返回 0，不报 KeyError
print(counter.most_common(2))  # [('apple', 3), ('banana', 2)]
```

★ **关键认知**：`Counter` 访问不存在的键返回 `0` 而不是抛 `KeyError`，这是它比普通字典更适合计数累加的根本原因——`counter[x] += 1` 可以无脑写，不需要先判断键是否存在。

## 1.2 defaultdict

```python
from collections import defaultdict

names = ["张三", "张伟", "李四", "王五"]
groups = defaultdict(list)

for name in names:
    groups[name[0]].append(name)

print(dict(groups))
# {'张': ['张三', '张伟'], '李': ['李四'], '王': ['王五']}
```

⚠ **`defaultdict` 访问不存在的键会"顺手创建"它**：

```python
groups = defaultdict(list)
print(groups["不存在"])     # []
print("不存在" in groups)   # True！键被创建了
```

只想查询而不创建，用 `groups.get(key)`。

## 1.3 deque

```python
from collections import deque

q = deque([1, 2, 3])
q.append(4)          # 右端入队
q.appendleft(0)      # 左端入队
print(q.popleft())   # 0，O(1)
print(q.pop())       # 4，O(1)
```

⚠ **不要用 `list.pop(0)` 当队列**。列表头部删除是 O(n)，数据量大时是性能瓶颈，用 `deque` 才是 O(1)。

# 2. itertools

## 2.1 无限迭代器

```python
from itertools import count, cycle, repeat, islice

for i in count(10):
    if i > 13:
        break
    print(i)        # 10 11 12 13

print(list(islice(cycle("AB"), 5)))   # ['A', 'B', 'A', 'B', 'A']
print(list(repeat("x", 3)))           # ['x', 'x', 'x']
```

⚠ **无限迭代器不能直接 `list()`**，否则内存爆掉。必须配合 `break`、`islice` 或 `zip` 截断。

## 2.2 组合与排列

```python
from itertools import product, permutations, combinations

print(list(product("AB", repeat=2)))
# [('A','A'), ('A','B'), ('B','A'), ('B','B')]

print(list(permutations("ABC", 2)))
# [('A','B'), ('A','C'), ('B','A'), ...]

print(list(combinations("ABC", 2)))
# [('A','B'), ('A','C'), ('B','C')]
```

| 函数 | 顺序 | 允许重复 | 结果数 |
|------|------|---------|--------|
| `product` | 有序 | 是 | n 的 r 次方 |
| `permutations` | 有序 | 否 | n! / (n-r)! |
| `combinations` | 无序 | 否 | C(n, r) |

## 2.3 常用工具

```python
from itertools import chain, groupby

print(list(chain([1, 2], [3, 4], [5])))   # [1, 2, 3, 4, 5]
```

`groupby` 要求**先按分组键排序**，否则相同的键会被拆成多组：

```python
from itertools import groupby

data = [("a", 1), ("b", 2), ("a", 3)]
data.sort(key=lambda x: x[0])       # 必须先排序

for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# a [('a', 1), ('a', 3)]
# b [('b', 2)]
```

★ **关键认知**：`itertools` 的函数几乎都返回**迭代器**，是惰性的——不立即计算，只在被消费时逐个产出。这让"处理超大数据"成为可能。

# 3. functools

## 3.1 cache

```python
from functools import cache

@cache
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(100))            # 瞬间出结果
print(fib.cache_info())    # 命中/未命中统计
```

`@cache` 等价于 `lru_cache(maxsize=None)`，3.9+ 可用。

⚠ **被缓存的参数必须可哈希**。传 `list`、`dict` 会直接 `TypeError`，需要先转元组。

★ **关键认知**：缓存用参数做键，**参数相同就返回同一个对象**。如果返回可变对象，调用方修改它会污染缓存。缓存函数应返回不可变值或副本。

## 3.2 partial 与 wraps

`partial` 用于固定部分参数，例如 `double = partial(mul, 2)`（需 `from operator import mul`）。

```python
from functools import wraps
import time

def timer(func):
    @wraps(func)                    # 保留 __name__、__doc__
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} 耗时 {time.perf_counter() - start:.4f}s")
        return result
    return wrapper

@timer
def slow():
    """一个慢函数。"""
    time.sleep(0.1)

slow()
print(slow.__name__)   # slow，没有 @wraps 会是 wrapper
```

★ **关键认知**：装饰器本质是"用新函数替换旧函数"。不加 `@wraps`，原函数的 `__name__`、`__doc__`、类型注解全部丢失，调试和文档生成都会出问题。**写装饰器第一件事就是加 `@wraps(func)`**。

# 4. datetime

## 4.1 时间点与时间差

```python
from datetime import datetime, date, timedelta

now = datetime.now()
today = date.today()

delta = timedelta(days=7, hours=3)
print(now + delta)                # 七天三小时后
print((datetime(2026, 11, 6) - datetime(2026, 1, 1)).days)   # 309
```

## 4.2 时区

```python
from datetime import datetime, timezone, timedelta

utc_now = datetime.now(timezone.utc)        # 带时区，推荐
tz8 = timezone(timedelta(hours=8))
local = datetime.now(tz8)

aware = datetime(2026, 11, 6, 10, 0, tzinfo=timezone.utc)
beijing = aware.astimezone(tz8)              # 时区转换
```

⚠ **`datetime.now()` 返回"无时区"的 naive 时间**，它不知道自己在哪个时区，跨时区转换会出错。**生产代码应统一用 aware 时间**（`datetime.now(timezone.utc)`），只在展示时转本地时区。

## 4.3 格式化

```python
now = datetime(2026, 11, 6, 10, 30, 0)

print(now.strftime("%Y-%m-%d %H:%M:%S"))   # 2026-11-06 10:30:00
print(now.strftime("%Y年%m月%d日"))         # 2026年11月06日

parsed = datetime.strptime("2026-11-06 10:30:00", "%Y-%m-%d %H:%M:%S")
```

★ **关键认知**：`strftime` 是"时间 → 字符串"（f 代表 format），`strptime` 是"字符串 → 时间"（p 代表 parse）。**带 `str` 的是字符串结果**。解析失败抛 `ValueError`，外部输入要捕获。

# 5. re

## 5.1 基本匹配

```python
import re

text = "订单号 A123，金额 99 元；订单号 B456，金额 120 元。"

print(re.search(r"[A-Z]\d{3}", text).group())   # A123
print(re.findall(r"[A-Z]\d{3}", text))          # ['A123', 'B456']
print(re.match(r"订单", text))                   # 匹配成功
print(re.match(r"A123", text))                   # None，只从开头匹配
```

⚠ **正则字符串一律用原始字符串 `r"..."`**，否则 `\d`、`\b` 会被当成转义序列处理。

★ **关键认知**：`re.match` 只匹配**开头**，`re.search` 扫描**整个字符串**，`re.fullmatch` 要求**整串完全匹配**。初学最常犯的错是用 `match` 却期待"任意位置匹配"。

## 5.2 分组与替换

```python
text = "张三 20 岁，李四 30 岁"
pattern = r"(?P<name>\w+)\s+(?P<age>\d+)\s*岁"
print(re.search(pattern, text).groupdict())   # {'name': '张三', 'age': '20'}

print(re.findall(r"\d+", "a1b22c333"))       # ['1', '22', '333']
print(re.sub(r"\d+", "#", "a1b22c333"))      # a#b#c#
print(re.sub(r"\d+", lambda m: str(int(m.group()) * 2), "a1b22"))
# a2b44
```

# 6. argparse

```python
import argparse

def main():
    parser = argparse.ArgumentParser(description="统计文件行数")
    parser.add_argument("path", help="文件路径")
    parser.add_argument("-n", "--number", type=int, default=10, help="显示前 N 行")
    parser.add_argument("--verbose", action="store_true", help="输出详细信息")

    args = parser.parse_args()
    print(f"路径：{args.path}，行数：{args.number}，详细：{args.verbose}")

if __name__ == "__main__":
    main()
```

```bash
python count.py data.txt -n 5 --verbose
```

★ **关键认知**：`argparse` 自动生成 `-h/--help`、处理类型转换、参数错误时打印用法并退出。**不要自己解析 `sys.argv`**——手写解析既不健壮也不友好。复杂 CLI 可进一步看 `click` 或 `typer`。

# 7. 常见错误

| 错误 | 现象 | 正确做法 |
|------|------|---------|
| `list.pop(0)` 当队列 | O(n) 性能差 | `deque` |
| 无限迭代器 `list()` | 内存爆掉 | `islice` / `break` |
| `groupby` 未排序 | 相同键被拆组 | 先 `sort` |
| 缓存可变参数 | `TypeError` | 转元组 |
| 装饰器漏 `@wraps` | 元信息丢失 | 加 `@wraps(func)` |
| `datetime.now()` | naive 时间 | `datetime.now(timezone.utc)` |
| 正则不用 `r""` | 转义错误 | 原始字符串 |
| `re.match` 当 `search` | 只匹配开头 | 用 `search` |

# 8. 练习

1. 用 `Counter` 统计一段英文文本的词频，输出出现次数最多的 3 个单词；再用 `defaultdict(list)` 按首字母分组。
2. 用 `itertools.combinations` 从列表中找出所有两数之和等于目标值的组合，要求不重复。
3. 用 `argparse` 写一个 `wc` 风格小工具：接收文件路径和 `--lines` 开关，输出总行数或总字符数；参数错误时依赖 `argparse` 自动报错。

# 9. 小结

| 模块 | 核心工具 | 一句话用途 |
|------|---------|-----------|
| `collections` | `Counter`/`defaultdict`/`deque` | 计数、分组、队列 |
| `itertools` | `product`/`combinations`/`islice` | 惰性组合与截取 |
| `functools` | `cache`/`partial`/`wraps` | 缓存、固定参数、装饰器 |
| `datetime` | `datetime`/`timezone`/`timedelta` | 时间点、时区、时间差 |
| `re` | `search`/`findall`/`sub` | 文本匹配与替换 |
| `argparse` | `ArgumentParser` | 命令行参数 |

下一篇进入并发，讲清楚 GIL 到底是什么、为什么线程适合 IO 密集、多进程如何绕开 GIL，以及锁、队列和 `concurrent.futures` 的正确用法。
