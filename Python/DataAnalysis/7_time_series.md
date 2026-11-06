时间序列是数据分析里最容易被低估的部分。看起来只是"按日期排序"，实际上索引类型、重采样边界、时区、缺失日期每一个都能让结果错位。本篇从 `DatetimeIndex` 讲起，把重采样、滚动窗口、时区和缺失时间补齐讲透。

# 1. DatetimeIndex

## 1.1 时间索引的意义

把时间列设为索引后，Pandas 才能做按时间切片、按月重采样、按时间窗口滚动。普通字符串列做不到这些。

```python
import pandas as pd

df = pd.DataFrame({
    "date": ["2026-01-01", "2026-01-02", "2026-01-03"],
    "sales": [100, 120, 90],
})

df["date"] = pd.to_datetime(df["date"])
df = df.set_index("date")

print(df.index)          # DatetimeIndex
print(df.index.dtype)    # datetime64[ns]
```

## 1.2 生成时间索引

```python
# 固定频率：按天
daily = pd.date_range("2026-01-01", periods=5, freq="D")
print(daily)
# DatetimeIndex(['2026-01-01', ..., '2026-01-05'])

# 按小时
hourly = pd.date_range("2026-01-01", periods=4, freq="h")

# 按月首
month_start = pd.date_range("2026-01-01", periods=4, freq="MS")

# 指定起止日期
rng = pd.date_range("2026-01-01", "2026-01-31", freq="D")
```

| 频率别名 | 含义 | 备注 |
|---------|------|------|
| `D` | 日 | 自然日 |
| `W` | 周 | 默认周日为一周结束 |
| `MS` | 月初 | Month Start |
| `ME` | 月末 | Pandas 2.2 起用 `ME` 替代 `M` |
| `QE` / `YE` | 季末 / 年末 | 替代旧的 `Q` / `A` |
| `h` | 小时 | Pandas 2.2 起推荐小写 |

## 1.3 解析字符串

```python
s = pd.Series(["2026-01-01", "2026/01/02", "20260103"])

# 统一格式时显式给 format，最快也最安全
parsed = pd.to_datetime(s, format="mixed", errors="coerce")
print(parsed)
# 0   2026-01-01
# 1   2026-01-02
# 2   2026-01-03

print(parsed.dt.year)      # 提取年
print(parsed.dt.day_name())  # 星期几
```

★ **关键认知**：`to_datetime` 的 `errors="coerce"` 会把无法解析的值变成 `NaT`（时间版的 `NaN`），不会中断整个流程。**外部数据一律加 `errors="coerce"`，再统计 `isna()` 的数量**。

⚠ **不要依赖自动推断格式**。`"01/02/2026"` 到底是 1 月 2 日还是 2 月 1 日，取决于系统区域设置。跨环境运行时，自动解析会给出不同结果。

# 2. 时间索引的切片

```python
df = pd.DataFrame(
    {"sales": range(1, 11)},
    index=pd.date_range("2026-01-01", periods=10, freq="D"),
)

print(df.loc["2026-01-03"])              # 单日
print(df.loc["2026-01-03":"2026-01-05"]) # 日期区间，包含右端
print(df.loc["2026-01"])                 # 整个 1 月，部分字符串匹配
print(df.loc["2026"])                    # 整年
```

★ **关键认知**：时间索引的**部分字符串切片**是 Pandas 的独有能力。`df.loc["2026-01"]` 会自动匹配该月所有日期，不需要先构造起止时间。这是把日期设为索引的最大收益之一。

⚠ 时间切片要求索引**有序且唯一**。乱序索引可能返回空结果或触发警告，先 `sort_index()`。

# 3. 重采样

## 3.1 resample 的基本用法

重采样是"改变时间频率"：日 → 月是降采样，月 → 日是升采样。

```python
df = pd.DataFrame(
    {"sales": [10, 12, 9, 15, 20, 18, 25, 22]},
    index=pd.date_range("2026-01-01", periods=8, freq="D"),
)

# 按周汇总，取每周日和
weekly = df.resample("W").sum()
print(weekly)

# 按月汇总，取每月末
monthly = df.resample("ME").sum()
```

`resample` 返回一个"重采样器"，必须再跟聚合函数（`sum`/`mean`/`max`/`agg`）才会真正计算。

## 3.2 多列不同聚合

```python
df = pd.DataFrame({
    "sales": [10, 12, 9, 15, 20, 18, 25, 22],
    "orders": [2, 3, 1, 4, 5, 3, 6, 4],
}, index=pd.date_range("2026-01-01", periods=8, freq="D"))

result = df.resample("W").agg({
    "sales": "sum",
    "orders": "mean",
})
```

## 3.3 label 与 closed

降采样时，"这一周算哪一天"由 `label` 决定，"区间左闭还是右闭"由 `closed` 决定。

```python
# 默认：label 为右端点，区间左闭右开
df.resample("W").sum()

# 标签用左端点
df.resample("W", label="left").sum()

# 区间右闭
df.resample("W", closed="right").sum()
```

```text
按周重采样（默认 label=right, closed=left）：
1 月 1 日（周四）到 1 月 4 日（周日） → 归入 1 月 4 日这一周
```

⚠ **不显式指定 `label` 和 `closed` 时，周、月的归属常与直觉相反**。做报表对账时，务必先拿一小段数据手工核对边界。

★ **关键认知**：重采样改变的是**时间粒度**，不是简单排序。`sum` 会把区间内所有记录加起来，`mean` 会按区间内记录数平均。区间划分错了，数值全错。

# 4. 滚动窗口与滞后

## 4.1 rolling

```python
df["ma3"] = df["sales"].rolling(window=3).mean()      # 3 期移动平均
df["sum7"] = df["sales"].rolling("7D").sum()          # 按 7 天时间窗
```

- `window=3`：按**行数**取最近 3 行。
- `window="7D"`：按**时间长度**取最近 7 天，要求 `DatetimeIndex` 且有序。

## 4.2 shift、diff、pct_change

```python
df["prev"] = df["sales"].shift(1)             # 上一期
df["diff"] = df["sales"].diff()               # 与上一期差
df["growth"] = df["sales"].pct_change(fill_method=None)   # 环比增长率
df["next"] = df["sales"].shift(-1)            # 下一期（会引入未来信息）
```

| 方法 | 作用 | 常见用途 |
|------|------|---------|
| `shift(1)` | 整体下移一行 | 构造滞后特征 |
| `diff()` | 当前减上一期 | 增量、变化量 |
| `pct_change()` | 变化率 | 环比、同比增长 |
| `rolling(3)` | 滚动统计 | 平滑、波动率 |

⚠ **`shift(-1)` 会把未来值带到当前行**。做预测特征时这是**数据泄漏**，模型离线评估很好、上线全崩。构造特征时只允许用当前及过去的信息。

★ **关键认知**：`pct_change` 的分母是上一期，上一期为 0 时结果是 `inf`，上一期缺失时是 `NaN`。增长率入表前先处理这两种值，否则排序和绘图都会出问题。

# 5. 时区

## 5.1 naive 与 aware

没有时区信息的时间叫 naive，有时区的叫 aware。二者不能直接比较。

```python
# naive：不知道自己在哪个时区
naive = pd.Timestamp("2026-01-01 10:00:00")

# aware：明确带时区
aware = pd.Timestamp("2026-01-01 10:00:00", tz="Asia/Shanghai")
```

## 5.2 localize 与 convert

```python
# naive → aware：赋予时区（不改变钟表时间）
local = naive.tz_localize("Asia/Shanghai")

# aware → aware：转换时区（改变钟表时间，时间点不变）
utc = local.tz_convert("UTC")

print(local)   # 2026-01-01 10:00:00+08:00
print(utc)     # 2026-01-01 02:00:00+00:00
```

★ **关键认知**：`tz_localize` 是**贴标签**，`tz_convert` 是**换时区**。前者回答"这个钟表时间属于哪个时区"，后者回答"同一时刻在另一个时区是几点"。用反了会差出整整几个小时。

## 5.3 夏令时的坑

```python
# 不存在的时刻（春季跳过的 1 小时）和重复的时刻（秋季回拨的 1 小时）
pd.Timestamp("2026-03-08 02:30").tz_localize(
    "America/New_York", ambiguous="NaT", nonexistent="NaT"
)
```

⚠ **跨时区数据统一存 UTC，展示时再转本地时区**。存储用本地时间，遇到夏令时切换会出现重复或缺失的一小时，后续对齐全乱。

# 6. 缺失时间

## 6.1 asfreq 补空

```python
# 原始数据缺少 1 月 3 日
df = df.drop(index=pd.Timestamp("2026-01-03"))

# 补齐为连续日频，缺失位置为 NaN
full = df.asfreq("D")
print(full)
```

`asfreq` 只改变频率、不填值。要填值再跟 `ffill`/`interpolate`：

```python
full["sales"] = full["sales"].ffill()          # 前值填充
full["sales"] = full["sales"].interpolate()    # 线性插值
```

## 6.2 reindex 到完整时间轴

```python
full_index = pd.date_range(df.index.min(), df.index.max(), freq="D")
full = df.reindex(full_index)
```

★ **关键认知**：**缺失日期不是缺失值**。直接对原始数据 `rolling("7D")`，Pandas 会按真实时间跨度计算，但行与行之间可能隔着几天。想得到"每个自然日一行"的连续序列，必须先 `asfreq` 或 `reindex` 补齐时间轴。

⚠ **补齐后再聚合要小心**。插值出来的值参与 `sum` 会虚增总量；如果指标是"累计量"，补出来的行应填 0 而不是前值。

# 7. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 日期列留作字符串 | 无法切片、重采样 | `to_datetime` 后设索引 |
| 索引未排序 | 切片返回空或错位 | `sort_index()` |
| 自动推断日期格式 | 跨环境结果不同 | 显式 `format` |
| `M`/`Q`/`Y` 别名 | Pandas 2.2 起弃用 | 用 `ME`/`QE`/`YE` |
| 忽略 `label`/`closed` | 周月归属错 | 显式指定并核对 |
| `shift(-1)` 做特征 | 数据泄漏 | 只用过去信息 |
| 时区用 `convert` 当 `localize` | 时间差几小时 | 区分贴标签与换算 |
| 缺失日期直接 fillna | 时间轴仍不连续 | 先 `asfreq`/`reindex` |

# 8. 练习

1. 生成 2026 年 1 月的日频销售数据，按周和按月分别求和，并解释两种结果的日期标签含义。
2. 在日频数据中人为删掉两天，用 `asfreq` 补齐后分别尝试前向填充和线性插值，比较差异。
3. 把一批 `Asia/Shanghai` 的时间转成 UTC 再转回本地，验证时间点不变；再用 `tz_localize` 对 naive 时间做同样操作，说明两者区别。

# 9. 小结

| 主题 | 要点 |
|------|------|
| DatetimeIndex | 时间序列的入口，支持部分字符串切片 |
| 解析 | `to_datetime` + `errors="coerce"` + 显式 `format` |
| 频率 | `D`/`W`/`MS`/`ME`/`QE`/`YE`/`h` |
| resample | 改变时间粒度，注意 `label` 与 `closed` |
| rolling | 按行数或按时间长度，构造平滑与特征 |
| shift/diff | 滞后、增量；`shift(-1)` 会泄漏未来 |
| 时区 | 存 UTC，展示转本地；`localize` 与 `convert` 不同 |
| 缺失时间 | 先补时间轴，再决定填 0、前值还是插值 |

下一篇进入可视化，讲清 Matplotlib 的 Figure/Axes 结构、常见图表的选择、Seaborn 的统计图、中文显示配置，以及保存与可访问性。
