清洗完数据，下一步是从明细里提炼结论：每个城市卖了多少、每类商品的平均客单价、每个用户最近 7 天的消费趋势。这些问题背后是同一套机制——**分组、聚合、再展开**。本篇把 `groupby`、`agg`、透视表、窗口函数和多级索引连起来讲。

# 1. groupby 的机制

## 1.1 拆分-应用-合并

`groupby` 不是"一个函数"，而是三个阶段的组合：

```text
原始表
┌──────┬────────┬────────┐
│ city │ product│ amount │
├──────┼────────┼────────┤
│ 北京 │ A      │ 100    │
│ 上海 │ A      │ 80     │
│ 北京 │ B      │ 60     │
│ 上海 │ B      │ 120    │
└──────┴────────┴────────┘
        │ 1. 按 city 拆分
        ▼
 ┌─────────────┐   ┌─────────────┐
 │ 北京组       │   │ 上海组       │
 │ A 100       │   │ A 80        │
 │ B 60        │   │ B 120       │
 └─────────────┘   └─────────────┘
        │ 2. 对每组应用 sum()
        ▼
 ┌─────────────┐   ┌─────────────┐
 │ 北京 160     │   │ 上海 200     │
 └─────────────┘   └─────────────┘
        │ 3. 合并结果
        ▼
   city  amount
   北京    160
   上海    200
```

★ **关键认知**：`groupby` 的核心是**"按某个键把行分成若干组，每组独立计算，再把结果拼回一张表"**。理解这三步，`agg`、`transform`、`filter` 的差异就一目了然——它们只是"每组算出什么、怎么拼回去"不同。

## 1.2 基本用法

```python
import pandas as pd

df = pd.DataFrame({
    "city": ["北京", "上海", "北京", "上海", "北京"],
    "product": ["A", "A", "B", "B", "A"],
    "amount": [100, 80, 60, 120, 40],
    "qty": [2, 1, 3, 4, 1],
})

# 单列分组，取一列聚合
print(df.groupby("city")["amount"].sum())
# city
# 北京    200
# 上海    200
# Name: amount, dtype: int64

# 多列分组
print(df.groupby(["city", "product"])["amount"].sum())
```

`as_index=False` 可以把分组键还原成普通列，方便后续链式操作：

```python
result = (
    df.groupby("city", as_index=False)["amount"]
      .sum()
      .sort_values("amount", ascending=False)
)
```

⚠ **默认 `as_index=True`，分组键会变成索引**。后面若直接 `merge` 或取列，容易因为索引对不上而出错。需要平表时显式写 `as_index=False` 或 `reset_index()`。

# 2. 聚合

## 2.1 agg 的多指标聚合

```python
result = df.groupby("city").agg({
    "amount": ["sum", "mean", "max"],
    "qty": "sum",
})
print(result)
#       amount             qty
#          sum   mean  max  sum
# city
# 北京     200   66.7  100    6
# 上海     200  100.0  120    5
```

聚合后列变成**多级列索引**（MultiIndex），访问要写 `result[("amount", "sum")]`，或者重命名压平：

```python
result.columns = ["amount_sum", "amount_mean", "amount_max", "qty_sum"]
```

## 2.2 命名聚合（推荐）

Pandas 0.25 之后支持"命名聚合"，直接给每个结果起名，避免多级列：

```python
result = df.groupby("city").agg(
    total=("amount", "sum"),
    avg_amount=("amount", "mean"),
    max_qty=("qty", "max"),
    orders=("amount", "size"),
)
print(result)
#       total  avg_amount  max_qty  orders
# city
# 北京    200       66.67        3       3
# 上海    200      100.00        4       2
```

★ **关键认知**：**优先用命名聚合**。它输出的列名扁平、可读，后续选列、写文件、画图都省事；传统字典聚合只在需要"一列多个统计"且能接受多级列时使用。

## 2.3 transform：把结果广播回原表

`transform` 返回与原表**同长度**的结果，常用于算占比、组内标准化：

```python
df["city_total"] = df.groupby("city")["amount"].transform("sum")
df["share"] = df["amount"] / df["city_total"]

print(df)
#   city product  amount  qty  city_total  share
# 0  北京       A     100    2         200   0.50
# 1  上海       A      80    1         200   0.40
```

## 2.4 filter：按组筛选

```python
# 只保留总金额大于 150 的城市
big_cities = df.groupby("city").filter(lambda g: g["amount"].sum() > 150)
```

| 方法 | 返回形状 | 用途 |
|------|---------|------|
| `agg` | 每组一行 | 汇总指标 |
| `transform` | 与原表同长 | 组内占比、标准化 |
| `filter` | 原表子集 | 按组条件筛行 |
| `apply` | 任意 | 复杂自定义逻辑，最慢 |

⚠ **`apply` 最灵活也最慢**。能用 `agg`/`transform` 表达的，不要用 `apply`；后者对每组调用一次 Python 函数，数据量大时性能差距明显。

# 3. 透视表

## 3.1 pivot_table

透视表是 `groupby` 的"二维展示版"：一个维度做行，一个维度做列。

```python
pivot = df.pivot_table(
    values="amount",
    index="city",
    columns="product",
    aggfunc="sum",
    fill_value=0,
    margins=True,          # 增加"总计"行列
    margins_name="总计",
)
print(pivot)
```

| 参数 | 作用 |
|------|------|
| `values` | 要聚合的数值列 |
| `index` | 行分组键 |
| `columns` | 列分组键 |
| `aggfunc` | 聚合函数，默认 `mean` |
| `fill_value` | 空组合填充值 |
| `margins` | 是否加总计 |

⚠ **`pivot_table` 的默认聚合是 `mean`，不是 `sum`**。不写 `aggfunc` 很容易得到"平均值"却误以为是总额。

## 3.2 crosstab：频次透视

`crosstab` 专门做交叉计数，默认 `aggfunc="count"`：

```python
pd.crosstab(df["city"], df["product"])
# product  A  B
# city
# 北京      2  1
# 上海      1  1

```

★ **关键认知**：`pivot_table` 面向**数值聚合**，`crosstab` 面向**频次统计**。想看"每个城市买了几次 A"用 `crosstab`，想看"每个城市 A 的销售额"用 `pivot_table`。

# 4. 窗口函数

## 4.1 rolling：滚动窗口

```python
ts = pd.DataFrame({
    "date": pd.date_range("2026-01-01", periods=8, freq="D"),
    "sales": [10, 12, 9, 15, 20, 18, 25, 22],
}).set_index("date")

ts["ma3"] = ts["sales"].rolling(window=3).mean()      # 3 日均线
```

窗口默认右对齐，前两个位置因为不足 3 个观测是 `NaN`。`min_periods=1` 可以让开头也有值：

```python
ts["ma3"] = ts["sales"].rolling(window=3, min_periods=1).mean()
```

## 4.2 expanding 与 ewm

```python
ts["cum_avg"] = ts["sales"].expanding().mean()        # 从起点累计到当前
ts["ewm"] = ts["sales"].ewm(span=3, adjust=False).mean()   # 指数加权
```

| 窗口 | 含义 | 典型用途 |
|------|------|---------|
| `rolling(3)` | 最近 3 个观测 | 移动平均、波动率 |
| `expanding()` | 从第一个到当前 | 累计均值、累计转化率 |
| `ewm(span=3)` | 指数衰减权重 | 更重视近期数据 |

★ **关键认知**：滚动窗口**不会减少行数**，它把每个位置替换成"以该位置结尾的窗口统计量"。所以它常用来构造特征，而不是做最终汇总。

⚠ **`rolling` 默认按位置数窗口，不是按时间长度**。数据有缺失日期时，`window=7` 是"7 行"而不是"7 天"。要按时间长度，用 `rolling("7D")`，前提是 `DatetimeIndex` 且已排序。

# 5. 多级索引

## 5.1 生成与访问

多列 `groupby` 默认生成 MultiIndex：

```python
summary = df.groupby(["city", "product"])["amount"].sum()
print(summary)
# city  product
# 北京  A          140
#      B           60
# 上海  A           80
#      B          120

print(summary.loc["北京"])              # 取第一层
print(summary.loc[("北京", "A")])       # 取到具体组合
print(summary.xs("A", level="product")) # 跨层取某一层值
```

## 5.2 stack 与 unstack

```python
wide = summary.unstack()        # 把 product 层转到列
long = wide.stack()             # 再转回长表

# 压平列名
wide.columns = [f"{city}_{product}" for city, product in wide.columns]
```

⚠ **`stack` 默认会丢弃缺失组合**（Pandas 2.1 起 `dropna` 行为有调整）。需要保留所有组合时显式传 `dropna=False`。

## 5.3 压平与还原

```python
flat = summary.reset_index().rename(columns={"amount": "total"})   # 压平并改名
```

★ **关键认知**：多级索引不是"更高级的表"，而是**给每行一个复合主键**。需要人读、写 CSV、交给绘图库时，通常要 `reset_index` 压平；需要做二维对齐时，才保留索引。

# 6. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 忘了 `as_index=False` | 分组键变成索引 | 需要平表时显式关闭 |
| `pivot_table` 没写 `aggfunc` | 得到均值而非总额 | 明确指定聚合函数 |
| 多级列直接取列 | `KeyError` | 用元组 `[(..., ...)]` |
| 分组列含 `NaN` | 默认被丢弃 | `dropna=False` |

分组键含缺失时用 `groupby("city", dropna=False)` 保留 `NaN` 组。

# 7. 练习

1. 用 `groupby` + 命名聚合，计算每个城市的订单数、总金额、平均金额，并按总金额降序输出。
2. 用 `transform` 计算每笔订单占所在城市总额的比例，再筛出占比超过 40% 的订单。
3. 构造 10 天的销售数据，计算 3 日滚动均值、累计均值和指数加权均值，比较三者在开头的差异。

# 8. 小结

| 主题 | 要点 |
|------|------|
| groupby | 拆分 → 应用 → 合并，键可单列或多列 |
| agg | 每组一行，优先命名聚合得到扁平列名 |
| transform | 与原表同长，用于占比与组内标准化 |
| pivot_table | 二维聚合，默认 `mean`，注意 `aggfunc` |
| 窗口 | `rolling` 按行或按时间，`expanding`、`ewm` |
| 多级索引 | 复合主键，展示前 `reset_index` 压平 |

下一篇进入表连接，讲清 `merge` 的四种连接类型、键冲突与 `suffixes`、`concat` 的拼接方向，以及连接顺序和索引对性能的影响。
