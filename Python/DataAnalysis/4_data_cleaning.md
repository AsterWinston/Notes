真实数据几乎不可能直接拿来分析：缺字段、重复行、数字存成文本、日期格式混乱、极端值混在正常值里。清洗占掉一个分析项目大半时间，而且它的每一步都必须可解释、可复现。本篇按"缺失、重复、类型、异常、字符串"五类问题逐项拆解。

# 1. 清洗的目标与原则

## 1.1 四类典型问题

```text
原始数据
   │
   ├── 缺失值：空、NULL、"-"、"未知"
   ├── 重复值：完全重复、业务主键重复
   ├── 类型错误：数字存成字符串、日期是文本
   ├── 异常值：录入错误、单位错误、真实极端值
   └── 脏字符串：空格、大小写、全半角、附带单位
   │
   ▼
可分析数据
```

★ **关键认知**：清洗不是"把数据变好看"，而是**让每一列的取值符合分析所需的语义**。同一列里混着"北京""北京市""北京 "，聚合时就会拆成三组，这不是格式问题，是结论错误。

## 1.2 三条基本原则

1. **原始数据只读**：所有清洗从 `data/raw` 读，结果写 `data/processed`，任何时候都能重跑。
2. **每步可追溯**：删除多少行、填充了什么值，都记录下来。
3. **先诊断后动手**：先统计缺失率、重复率、取值范围，再决定策略，不要上来就 `dropna()`。

```python
import pandas as pd

df = pd.read_csv("data/raw/orders.csv")

print(df.shape)                       # 行列数
print(df.isna().mean().sort_values(ascending=False))   # 各列缺失率
print(df.duplicated().sum())          # 完全重复行数
print(df.describe(include="all").T)   # 数值与类别列概览
```

# 2. 缺失值

## 2.1 识别

Pandas 用 `NaN`（数值）、`NaT`（时间）、`None`/`NA`（对象与扩展类型）表示缺失。`isna()` 统一识别。

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    "name": ["张三", "李四", None, "王五"],
    "age": [23, np.nan, 28, 31],
    "city": ["北京", " ", "上海", None],
})

print(df.isna())
print(df.isna().sum())          # 每列缺失数量
print(df.isna().mean())         # 每列缺失比例
```

⚠ **空字符串、空格、`"-"`、`"NULL"` 不是 `NaN`**。`isna()` 认不出来，必须在读入时用 `na_values` 声明，或读入后用 `replace` 处理：

```python
df = pd.read_csv("orders.csv", na_values=["", "-", "NULL", "未知"])
df["city"] = df["city"].str.strip().replace("", np.nan)
```

## 2.2 删除

```python
df.dropna()                                   # 任一行有缺失就删
df.dropna(how="all")                          # 整行全缺才删
df.dropna(subset=["amount", "date"])          # 只要求关键列非空
df.dropna(thresh=3)                           # 至少 3 个非空值才保留
```

★ **关键认知**：删除缺失值会**改变样本分布**。如果高收入用户恰好不爱填收入，删掉缺失行就等于删掉了高收入群体，结论会有系统偏差。删除前先问：缺失是随机的，还是和业务有关？

## 2.3 填充

```python
df["age"] = df["age"].fillna(df["age"].median())        # 中位数，抗异常值
df["city"] = df["city"].fillna("未知")                  # 类别列填"未知"
df["amount"] = df["amount"].fillna(0)                   # 金额缺失按 0 处理

# 用前一个有效值填充（时间序列常用）
df["price"] = df["price"].ffill()
# 用后一个有效值填充
df["price"] = df["price"].bfill()
```

| 策略 | 适用 | 风险 |
|------|------|------|
| 均值 | 近似正态、无极端值 | 被异常值拉偏 |
| 中位数 | 偏态分布、有异常值 | 忽略组间差异 |
| 固定值（0/未知） | 业务上缺失即无 | 混淆"没有"和"不知道" |
| 前向/后向填充 | 时间序列 | 连续缺失会传播旧值 |

⚠ **Pandas 2.1 起 `fillna(method="ffill")` 已弃用**，改用 `.ffill()` / `.bfill()`。旧教程里的写法会报 `FutureWarning`，Pandas 3.0 会移除。

## 2.4 插值

```python
# 线性插值：按索引等距补点
df["price"] = df["price"].interpolate(method="linear")
# 时间索引可改用 method="time"，按真实时间间隔补点
```

★ **关键认知**：插值只适合**连续变化且缺口不大**的指标，比如温度、价格。对离散的类别、计数、ID 插值没有意义，只会造出"半个人"这种不存在的值。

# 3. 重复值

## 3.1 判断

```python
df = pd.DataFrame({
    "order_id": [1001, 1001, 1002, 1003],
    "amount": [120, 120, 80, 50],
    "updated": ["2026-01-01", "2026-02-01", "2026-01-03", "2026-01-04"],
})

print(df.duplicated().sum())                      # 1，完全重复的行
print(df.duplicated(subset=["order_id"]).sum())   # 1，业务主键重复
```

## 3.2 去重策略

```python
# 完全重复：保留第一条
df = df.drop_duplicates()

# 主键重复：按更新时间保留最新一条
df = (
    df.sort_values("updated")
      .drop_duplicates(subset=["order_id"], keep="last")
)
```

★ **关键认知**：去重前必须明确**什么是"重复"**。完全相同的两行可能是两笔真实交易；同一 `order_id` 的两行则通常是更新记录，要按时间取最新。不定义业务主键就去重，等于随机删数据。

# 4. 类型转换

类型错误的信号：数字列能比较大小却无法求和、日期列无法按时间排序、ID 排序结果像字典序。

```python
import pandas as pd

df = pd.DataFrame({
    "id": ["001", "002", "003"],
    "amount": ["1,200", "800", "N/A"],
    "date": ["2026/01/05", "2026-01-06", "20260107"],
})

# ID 必须保持字符串，否则前导零丢失
df["id"] = df["id"].astype("string")

# 去掉千分位后再转数值，非法值变 NaN
df["amount"] = (
    df["amount"].str.replace(",", "", regex=False)
      .pipe(pd.to_numeric, errors="coerce")
)

# 多种日期格式：先统一分隔符，再解析
df["date"] = pd.to_datetime(
    df["date"].str.replace("/", "-", regex=False),
    errors="coerce",
    format="mixed",
)
```

⚠ **`format="mixed"` 是 Pandas 2.0 引入的**，用于一列里混有多种日期格式。旧版本只能逐行解析或先统一格式。解析后要检查 `isna()`，确认没有悄悄变成 `NaT`。

★ **关键认知**：类型转换要**先清洗再转换**。"1,200" 带千分位、"120 元" 带单位、"12.5%" 带百分号，直接 `astype(float)` 全部失败。先用字符串方法去掉噪声，再转数值。

# 5. 异常值

## 5.1 先看分布

```python
print(df["amount"].describe())
print(df["amount"].quantile([0.01, 0.25, 0.5, 0.75, 0.99]))
```

## 5.2 IQR 与 z-score

```python
q1 = df["amount"].quantile(0.25)
q3 = df["amount"].quantile(0.75)
iqr = q3 - q1

lower = q1 - 1.5 * iqr
upper = q3 + 1.5 * iqr

outliers = df[(df["amount"] < lower) | (df["amount"] > upper)]
print(len(outliers), lower, upper)
```

z-score 适合近似正态的数据：`(x - mean) / std` 绝对值大于 3 视为异常，但均值和标准差本身会被异常值污染。

| 方法 | 适用 | 局限 |
|------|------|------|
| IQR | 偏态、分布未知 | 对极端偏态仍可能漏判 |
| z-score | 近似正态 | 均值和标准差本身被异常值污染 |
| 业务阈值 | 有明确上下限 | 需要领域知识 |

## 5.3 处理方式

```python
# 1. 确认是录入错误：改正
df.loc[df["amount"] == -1, "amount"] = 1

# 2. 确认是单位错误：换算
df.loc[df["amount"] > 100000, "amount"] /= 100

# 3. 真实极端值保留；无法判断的单独标记，不直接删除
df["is_outlier"] = (df["amount"] < lower) | (df["amount"] > upper)
```

★ **关键认知**：**异常值不等于错误值**。电商大额订单、服务器流量尖峰都是真实数据，直接删掉会抹掉最重要的信息。先查来源，再决定是修正、保留还是标记。

⚠ **不要用"3 倍标准差"一刀切**。它假设正态分布，对金额、流量这类长尾数据会误删大量正常值。优先用 IQR 或业务阈值。

# 6. 字符串清洗

## 6.1 去空格与统一大小写

```python
df["city"] = df["city"].str.strip()          # 去首尾空格
df["city"] = df["city"].str.replace(r"\s+", "", regex=True)   # 去所有空白
df["email"] = df["email"].str.lower()        # 统一小写
df["code"] = df["code"].str.upper()
```

## 6.2 提取与匹配

```python
# 从"北京市朝阳区"中提取城市
df["city_clean"] = df["address"].str.extract(r"^(.+?市)")[0]

# 判断是否包含关键词，缺失值按 False 处理
mask = df["remark"].str.contains("退款", na=False)
```

⚠ **`str.contains` 遇到 `NaN` 默认返回 `NaN`**，在布尔索引里会报"无法将 NA 解释为布尔值"。养成加 `na=False` 的习惯。

## 6.3 分类映射

```python
mapping = {
    "北京": "华北", "北京市": "华北", "天津": "华北",
    "上海": "华东", "上海市": "华东", "杭州": "华东",
}

df["region"] = df["city"].str.strip().map(mapping)

# 映射不上的值会变成 NaN，用 fillna 兜底
df["region"] = df["region"].fillna("其他")
```

★ **关键认知**：字符串清洗的终点是**受控词表**。把自由文本映射成有限的类别（`category` dtype），既消除歧义，又省内存，还能让分组统计稳定。

# 7. 清洗顺序

```text
1. 读入时声明 na_values、dtype、parse_dates
2. 去重（先按业务主键）
3. 统一字符串（空格、大小写、全半角）
4. 类型转换（数值、日期、类别）
5. 处理缺失（删除或填充）
6. 处理异常值（核对、修正、标记）
7. 输出并记录每步行数变化
```

★ **关键认知**：顺序会影响结果。先去空格再映射，否则"北京 "匹配不上；先转类型再处理缺失，否则字符串列的 `"N/A"` 不会被识别。**每一步后打印 `df.shape`**，行数突变的地方就是问题所在。

# 8. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 直接 `dropna()` | 样本偏差 | 先看缺失率与业务含义 |
| `fillna(method=...)` | `FutureWarning` | 用 `.ffill()`/`.bfill()` |
| 数字列带千分位 | 转换失败 | 先去 `,` 再 `to_numeric` |
| 按 z-score 删异常 | 误删长尾正常值 | IQR 或业务阈值 |

# 9. 练习

1. 造一份含 `" "`、`"NULL"`、`NaN` 三种缺失形式的数据，统一转成 `NaN`，统计各列缺失率，并给出你选择删除还是填充的理由。
2. 给一份带重复 `order_id` 和 `updated` 时间的数据，按"每个订单保留最新记录"去重，并验证结果唯一。
3. 对含 `"1,200"`、`"800元"`、`"12.5%"` 的金额列做清洗：去掉符号、统一单位、转为浮点数，并说明每一种噪声分别怎么处理。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 缺失值 | 先识别真实缺失，再决定删除或填充 |
| 重复值 | 先定义业务主键，再谈去重 |
| 类型 | 先清噪声再转换，非法值 `errors="coerce"` |
| 异常值 | 不等于错误值，IQR 优于无脑 z-score |
| 顺序 | 去重 → 字符串 → 类型 → 缺失 → 异常，每步记录 |

下一篇进入分组聚合，讲清 `groupby` 的拆分-应用-合并机制、`agg` 的多指标聚合、透视表、窗口函数和多级索引，把清洗后的数据真正变成结论。
