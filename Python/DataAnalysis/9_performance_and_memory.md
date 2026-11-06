数据量从几万行涨到几百万行，同一段代码可能从"秒出"变成"跑一晚上"甚至"内存爆掉"。性能问题很少靠换机器解决，更多靠**换写法、换类型、换读取方式**。本篇讲如何定位瓶颈，以及向量化、dtype、分块和 copy 这四件最有效的事。

# 1. 先找到瓶颈

## 1.1 三类常见瓶颈

```text
程序慢
  │
  ├── 算法层：逐行 Python 循环、重复计算、O(n²) 操作
  ├── 类型层：object 存字符串、float64 装小整数、未用 category
  └── IO 层：整表读入、读不需要的列、反复读写磁盘
```

## 1.2 测量优先

不要凭感觉优化。先用计时和数据规模定位：

```python
import time

import pandas as pd

start = time.perf_counter()

df = pd.read_csv("data/raw/big.csv")
df["amount"] = df["amount"] * 1.1
result = df.groupby("city")["amount"].sum()

print(f"耗时 {time.perf_counter() - start:.2f}s")
```

逐段计时，找出占比最大的那一段，只优化它。

★ **关键认知**：**先测量，再优化**。凭直觉猜瓶颈，十有八九优化了不重要的 5%。用 `time.perf_counter()` 给每段计时，用 `df.memory_usage(deep=True)` 看内存，比"感觉慢"可靠得多。

# 2. 向量化

## 2.1 不要逐行循环

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({"a": np.arange(1_000_000), "b": np.arange(1_000_000)})

# 错误示范：iterrows，每行构造一个 Series，极慢
result = []
for _, row in df.iterrows():
    result.append(row["a"] * 2 + row["b"])
df["c"] = result

# 正确做法：整列运算
df["c"] = df["a"] * 2 + df["b"]
```

`iterrows` 每行都要创建一个 Series 对象，百万行就是百万次对象构造。向量化把运算交给 NumPy 的 C 循环，通常快几十到上百倍。

## 2.2 条件逻辑用 np.where / np.select

```python
# 单条件
df["level"] = np.where(df["a"] > 500_000, "高", "低")

# 多条件，按顺序匹配
conditions = [df["a"] < 300_000, df["a"] < 700_000]
choices = ["低", "中"]
df["level"] = np.select(conditions, choices, default="高")
```

## 2.3 字符串操作用 .str

```python
df["city"] = df["city"].str.strip().str.lower()

# 不要写
# df["city"] = df["city"].map(lambda x: x.strip().lower())
```

`.str` 系列是向量化实现，比 `map(lambda ...)` 逐元素调用 Python 函数快得多。

★ **关键认知**：判断一段代码能否向量化，看它是否"对每一行做同样的事"。如果是，就一定能写成整列运算或 `np.where`；只有跨行依赖（累计、前值）才需要 `cumsum`、`shift`、`rolling` 这些专门工具。

⚠ **`apply` 不是向量化**。它只是"更优雅的循环"，底层仍逐行调用 Python。能不用就不用；必须用时可先缩小数据范围。

# 3. dtype 优化

## 3.1 数值降位

Pandas 默认用 `int64` 和 `float64`，很多场景根本用不到这么大范围。

```python
df = pd.DataFrame({
    "small": np.arange(1000),
    "big": np.arange(1000) * 1_000_000,
})

print(df.dtypes)
print(df.memory_usage(deep=True))

# 按实际范围降位
df["small"] = df["small"].astype("int16")
df["big"] = df["big"].astype("int64")       # 已够用
```

| dtype | 字节 | 范围 |
|-------|------|------|
| `int8` | 1 | -128 ~ 127 |
| `int16` | 2 | -32768 ~ 32767 |
| `int32` | 4 | 约 ±21 亿 |
| `int64` | 8 | 约 ±9.2e18 |
| `float32` | 4 | 约 7 位有效数字 |
| `float64` | 8 | 约 15 位有效数字 |

⚠ **浮点降位会丢精度**。`float32` 只有约 7 位有效数字，金额、坐标累计误差可能明显。**金额、ID 不要降 float**；特征工程里对精度不敏感的列才考虑 `float32`。

## 3.2 category 省内存

低基数的字符串列（城市、性别、状态）用 `category` 存储，底层只存整数编码加一张字典。

```python
df["city"] = df["city"].astype("category")

print(df["city"].cat.categories)   # 查看类别
print(df["city"].nunique())        # 类别数量
```

★ **关键认知**：`category` 适合**重复率高、取值有限**的列。唯一值上万的列转 `category` 反而更占内存，还会拖慢排序和连接。经验阈值：唯一值占比低于 5% 到 10% 才值得。

## 3.3 string 与 object

Pandas 2.x 的 `object` 列通常存的是 Python 字符串，每个元素一个指针加对象头，很占内存。可空字符串类型更紧凑：

```python
df["name"] = df["name"].astype("string")
```

也可以用 `convert_dtypes()` 让 Pandas 自动推断更合适的可空类型：

```python
df = df.convert_dtypes()
```

⚠ **`convert_dtypes` 会改变 dtype 语义**，把 `int64` 变成 `Int64`、`object` 变成 `string`。下游若有依赖旧 dtype 的代码，先小范围验证。

# 4. 分块读取

## 4.1 chunksize

文件大到一次读不进内存时，分块处理：

```python
import pandas as pd

total = 0
for chunk in pd.read_csv("data/raw/big.csv", chunksize=100_000):
    total += chunk["amount"].sum()

print(total)
```

每块都是一个小 DataFrame，处理完即释放。适合"逐块聚合"这类可累加的任务。

## 4.2 只读需要的列和行

```python
df = pd.read_csv(
    "data/raw/big.csv",
    usecols=["date", "city", "amount"],   # 不读的列不占内存
    dtype={"city": "category"},           # 读入即压缩
    parse_dates=["date"],
    nrows=100_000,                        # 调试时先读一部分
)
```

★ **关键认知**：**最省内存的列，是根本没读进来的列**。`usecols` 和 `dtype` 在读取阶段就生效，比读进来再删列、再转换省一大截，也更快。

⚠ **分块后不能直接做跨块排序或全量 `groupby`**。分块适合求和、计数、增量写入；需要全局排序时，要么先落盘再外部排序，要么换用 DuckDB、Polars 这类能流式处理更大数据的工具。

# 5. 测量内存

## 5.1 memory_usage

```python
print(df.memory_usage(deep=True).sort_values(ascending=False))

print(df.memory_usage(deep=True).sum() / 1024**2, "MB")
```

- 不加 `deep=True` 只统计指针，不算字符串内容，会严重低估。
- 对象列必须 `deep=True` 才看得准。

## 5.2 估算

```python
rows = 1_000_000
cols = 20

# 全 float64 粗估：行 × 列 × 8 字节
print(rows * cols * 8 / 1024**2, "MB")
```

实际还要加索引、字符串对象、中间结果的 overhead。**经验上给 3 到 5 倍余量**，否则一次 `merge` 或 `sort` 的临时副本就可能 OOM。

⚠ **中间结果比原表更占内存**。`merge`、`sort_values`、`groupby` 都可能产生整表副本。处理大表时，先删掉不用的列，再执行这些操作。

# 6. 避免不必要的 copy

## 6.1 视图与副本

```python
import pandas as pd

df = pd.DataFrame({"a": [1, 2, 3], "b": [4, 5, 6]})

subset = df[["a"]]        # 列选择通常返回新对象
view = df["a"]            # 返回 Series 视图（行为随版本和操作而变）

# 需要独立数据时显式 copy
independent = df[["a"]].copy()
```

## 6.2 Copy-on-Write

Pandas 2.x 提供 Copy-on-Write（CoW），让"切片不复制、修改时才复制"成为默认语义：

```python
pd.options.mode.copy_on_write = True
```

开启后：

```python
df = pd.DataFrame({"a": [1, 2, 3]})
sub = df[["a"]]
sub.iloc[0, 0] = 100
print(df)   # 原表不变，修改触发复制
```

这既省内存，又消除了"改切片却改到原表"的隐蔽 bug。Pandas 3.0 会默认开启。

★ **关键认知**：**CoW 不是"永远不复制"，而是"延迟到写时才复制"**。读多写少时省内存；频繁修改时仍会复制，只是复制发生在赋值那一刻。

⚠ **不要在循环里对大表反复 `.copy()`**。每次 copy 都是整表内存翻倍。需要逐步构造结果时，先收集到列表，最后 `pd.concat` 一次。

# 7. 其他有效手段

| 手段 | 做法 | 适用 |
|------|------|------|
| 只算需要的列 | `df[["a", "b"]].groupby(...)` | 宽表聚合 |
| 先过滤再计算 | 先 `query` 再 `groupby` | 稀疏子集分析 |
| 避免链式索引赋值 | `df.loc[mask, "col"] = v` | 减少中间副本 |
| 用 `numpy` 算数值 | 纯数值运算降到 ndarray | 大量数学计算 |
| 换执行引擎 | Polars、DuckDB | 单机超大数据 |
| 落盘中间结果 | `to_parquet` | 多次复用清洗结果 |

```python
# 中间结果存 parquet（需 pyarrow），比 CSV 小且读得快
df.to_parquet("data/processed/clean.parquet", index=False)
df = pd.read_parquet("data/processed/clean.parquet")
```

# 8. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 用 `iterrows` | 百万行跑几分钟 | 向量化 |
| `apply(lambda ...)` | 只是优雅的循环 | `np.where`/`.str` |
| 无脑 `float32` | 金额精度丢失 | 金额保持 `float64` |
| 高基数列转 `category` | 更慢更占内存 | 唯一值占比低才用 |
| `memory_usage()` 不加深 | 低估字符串内存 | `deep=True` |
| 大表反复 `copy` | 内存翻倍 | 延迟复制、批量 concat |
| 分块后全局排序 | 结果不完整 | 换工具或先落盘 |
| 读全表再选列 | IO 和内存浪费 | 读入时 `usecols` |

# 9. 练习

1. 造一个百万行的 DataFrame，分别用 `iterrows`、`apply` 和向量化计算同一列，记录三者耗时并解释差异。
2. 对一份含城市、状态、金额的数据，把低基数列转为 `category`，比较转换前后的 `memory_usage(deep=True)`。
3. 用 `chunksize` 分块统计一个大 CSV 的总金额和行数，再与一次性读入的结果对比，验证一致性。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 先计时、看内存，只优化最慢的一段 |
| 向量化 | 整列运算替代 `iterrows`/`apply` |
| 条件逻辑 | `np.where`、`np.select` |
| dtype | 数值降位、低基数转 `category`、字符串用 `string` |
| 分块 | `chunksize` 处理超内存文件，适合可累加任务 |
| 内存 | `memory_usage(deep=True)`，给中间结果留余量 |
| copy | 视图/副本要分清，CoW 延迟复制 |
| IO | `usecols`、`dtype`、`parquet` |

下一篇是收尾案例，把读入、清洗、连接、分组、时间序列和可视化串成一次完整分析，并整理常见错误、自检题和学习路线。
