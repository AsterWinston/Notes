前九篇拆开了工具和概念，这一篇把它们串起来：从一份带脏数据的订单表出发，完成读入、体检、清洗、连接、特征、聚合、时间序列和可视化，最后给出结论。案例代码可以直接复制运行。

# 1. 案例背景与目标

## 1.1 业务问题

某电商想回答三个问题：

1. 各城市的销售额和客单价如何？
2. 月度销售趋势是否健康？
3. 哪些数据质量问题会影响结论？

## 1.2 数据说明

| 文件 | 内容 | 已知问题 |
|------|------|---------|
| `orders.csv` | 订单明细 | 金额缺失、千分位字符串、重复订单、日期为文本 |
| `users.csv` | 用户维表 | 城市名带空格、有"北京市"与"北京"两种写法 |

```text
orders.csv                users.csv
order_id, user_id,         user_id, city, level
order_date, amount,
updated
      │                        │
      └──────── merge ─────────┘
                 │
                 ▼
        清洗 + 特征 + 聚合
                 │
                 ▼
        城市汇总 / 月度趋势 / 图表
```

# 2. 生成与加载数据

## 2.1 造一份可复现的数据

```python
import numpy as np
import pandas as pd
from pathlib import Path

rng = np.random.default_rng(42)
n = 200
Path("data/raw").mkdir(parents=True, exist_ok=True)

orders = pd.DataFrame({
    "order_id": [f"A{i:04d}" for i in range(n)],
    "user_id": rng.integers(1, 31, n),
    "order_date": pd.to_datetime("2026-01-01")
                  + pd.to_timedelta(rng.integers(0, 90, n), unit="D"),
    "amount": rng.normal(120, 40, n).round(2).clip(10, None),
})

# 制造缺失金额
orders.loc[orders.sample(8, random_state=1).index, "amount"] = np.nan
# 制造千分位字符串
orders["amount"] = orders["amount"].astype(object)
orders.loc[orders.sample(5, random_state=2).index, "amount"] = "1,200"

# 制造重复订单：同一 order_id 出现两次，后一条 updated 更晚
dups = orders.sample(5, random_state=3).copy()
orders = pd.concat([orders, dups], ignore_index=True)
orders["updated"] = orders["order_date"]
orders.loc[orders.index[-5:], "updated"] = (
    orders.loc[orders.index[-5:], "order_date"] + pd.Timedelta(days=1)
)

users = pd.DataFrame({
    "user_id": range(1, 31),
    "city": rng.choice(["北京", "北京市", "上海 ", " 上海", "广州"], 30),
    "level": rng.choice(["普通", "银牌", "金牌"], 30),
})

orders.to_csv("data/raw/orders.csv", index=False)
users.to_csv("data/raw/users.csv", index=False)
```

`random_state` 和 `default_rng(42)` 保证每次生成同样的数据，结论可复现。

## 2.2 读入与体检

```python
orders_raw = pd.read_csv("data/raw/orders.csv", dtype={"order_id": "string"})
users = pd.read_csv("data/raw/users.csv")

print(orders_raw.shape)                              # (205, 5)
print(orders_raw.isna().sum())                       # amount 有缺失
print(orders_raw.duplicated(subset=["order_id"]).sum())   # 5 条重复主键
print(orders_raw.dtypes)                             # 日期、金额都是 object
print(orders_raw["amount"].head())
```

★ **关键认知**：体检的顺序是**形状 → 缺失 → 重复 → 类型 → 抽样**。先知道数据"有多大、脏在哪"，再决定清洗策略；一上来就 `dropna()` 或 `astype`，不是报错就是悄悄丢数据。

# 3. 清洗

## 3.1 去重与缺失

```python
orders = (
    orders_raw
    .assign(
        order_date=lambda d: pd.to_datetime(d["order_date"], errors="coerce"),
        updated=lambda d: pd.to_datetime(d["updated"], errors="coerce"),
        amount=lambda d: pd.to_numeric(
            d["amount"].astype(str).str.replace(",", "", regex=False),
            errors="coerce",
        ),
    )
    .sort_values("updated")
    .drop_duplicates(subset=["order_id"], keep="last")
)

# 金额缺失用中位数填充，并记录填充数量
missing_before = orders["amount"].isna().sum()
orders["amount"] = orders["amount"].fillna(orders["amount"].median())
print(f"填充金额缺失 {missing_before} 条")
print(orders.shape)
```

去重必须按 `order_id` 并保留 `updated` 最新的一条，而不是简单删完全重复行——重复订单的金额可能已经被修正过。

## 3.2 城市名清洗

```python
users["city"] = (
    users["city"]
    .str.strip()
    .replace({"北京市": "北京"})
)

print(users["city"].value_counts())
print(users["city"].isna().sum())
```

## 3.3 清洗后的核对

```python
print(orders["order_date"].isna().sum())    # 日期解析失败数
print(orders.duplicated(subset=["order_id"]).sum())   # 应为 0
```

★ **关键认知**：每一步清洗后都要**核对行数和关键指标**。从 205 行去掉 5 条重复应得 200 行；如果变成 198，说明有别的行被误删，必须停下来查。

# 4. 连接维表

```python
df = orders.merge(
    users[["user_id", "city", "level"]],
    on="user_id",
    how="left",
    validate="many_to_one",
)

print(df.shape)                 # 行数应仍为 200
print(df["city"].isna().sum())  # 是否有用户匹配不上
```

- `how="left"` 保留全部订单，不能因为维表缺用户就丢单。
- `validate="many_to_one"` 确保用户表 `user_id` 唯一，防止连接膨胀。
- 连接后行数不变，才说明没有多对多问题。

# 5. 特征构造

```python
df["month"] = df["order_date"].dt.to_period("M").astype(str)
df["weekday"] = df["order_date"].dt.day_name()
df["is_weekend"] = df["order_date"].dt.weekday >= 5
df["amount_band"] = pd.cut(
    df["amount"],
    bins=[0, 100, 200, np.inf],
    labels=["低", "中", "高"],
)

print(df[["order_date", "month", "weekday", "is_weekend", "amount_band"]].head())
```

特征要**从业务含义出发**：月份用于趋势，周末标记用于消费习惯，金额分箱用于结构分析。⚠ **不要为了"特征多"而堆无意义的列**，那只会增加维护成本。

# 6. 分组分析

```python
city_summary = (
    df.groupby("city", as_index=False)
      .agg(
          orders=("order_id", "count"),
          total=("amount", "sum"),
          avg_amount=("amount", "mean"),
      )
      .sort_values("total", ascending=False)
)

print(city_summary)
```

⚠ **对 `category` 列分组要加 `observed=True`**，否则所有类别组合都会出现，包括数据里根本没出现的组合，结果里会多出一堆 0。

# 7. 时间序列

```python
monthly = (
    df.set_index("order_date")
      .resample("ME")["amount"]
      .sum()
)

print(monthly.pct_change(fill_method=None).round(3))   # 环比增长率
```

`ME` 表示月末。用旧别名 `M` 在 Pandas 2.2 起会有弃用警告。若数据不是每天都有，`resample` 会自动按区间聚合，不需要先补日期；但要做"每日连续"的滚动窗口时，仍需先 `asfreq("D")`。

# 8. 可视化

```python
import matplotlib.pyplot as plt

plt.rcParams["font.sans-serif"] = ["Microsoft YaHei", "SimHei", "PingFang SC"]
plt.rcParams["axes.unicode_minus"] = False

fig, axes = plt.subplots(1, 2, figsize=(12, 4))

city_summary.plot(
    x="city", y="total", kind="bar",
    ax=axes[0], legend=False, color="#2E86AB",
)
axes[0].set_title("各城市销售额")
axes[0].set_xlabel("")
axes[0].set_ylabel("销售额")

monthly.plot(ax=axes[1], marker="o", color="#E4572E")
axes[1].set_title("月度销售额趋势")
axes[1].set_xlabel("")

fig.tight_layout()
fig.savefig("data/processed/sales_report.png", dpi=200, bbox_inches="tight")
plt.show()
```

# 9. 结论怎么写

```text
1. 城市维度：销售额最高的城市是 __，客单价最高的是 __。
2. 结构维度：中高金额订单贡献了 __% 的销售额。
3. 时间维度：月度趋势为 __，最近一个月环比 __%。
4. 数据限制：金额缺失 __ 条已用中位数填充；城市名已合并同义写法；
   结论不覆盖缺失用户信息的订单。
```

★ **关键认知**：结论必须**带数字、带口径、带限制**。"北京卖得最好"不是结论，"北京销售额 8.6 万元，占总销售额 32%，客单价 128 元，高于整体均值"才是。同时交代缺失值怎么处理，读者才知道结论的边界。

# 10. 常见错误总表

| 阶段 | 错误 | 后果 | 正确做法 |
|------|------|------|---------|
| 体检 | 不看缺失率 | 清洗策略拍脑袋 | 先 `isna().mean()` |
| 去重 | 只删完全重复行 | 主键重复仍在 | `subset` + `keep` |
| 类型 | 直接 `astype(float)` | 千分位报错 | 先去噪再转换 |
| 连接 | 默认 `inner` | 丢订单 | `how="left"` |
| 分组 | `category` 不加 `observed` | 出现空组合 | `observed=True` |
| 时间 | 用 `M` 别名 | 弃用警告 | `ME` |
| 结论 | 不写口径 | 无法复核 | 数字 + 定义 + 限制 |

# 11. 自检题

1. 一份订单表连接用户表后行数从 10 万变成 12 万，最可能的原因是什么？如何用代码验证？
2. 金额列同时有 `NaN`、`"1,200"`、`"800元"`，写出完整的清洗步骤，并说明每一步的作用。
3. 做 `resample` 前为什么必须把日期设为索引并排序？乱序会出什么问题？
4. `rolling("7D")` 和 `rolling(7)` 的区别是什么？分别在什么场景使用？
5. 判断：`df[df["a"] > 1]["b"] = 0` 能可靠地修改原表吗？为什么？

# 12. 学习路线

```text
入门：环境与工作流 → NumPy → Pandas
  → 清洗 → 分组聚合 → 连接
  → 时间序列 → 可视化 → 性能与内存
  → 完整案例与复盘
```

| 阶段 | 目标 | 验收标准 |
|------|------|---------|
| 入门 | 能读写表格、做基础筛选 | 独立完成一次 CSV 汇总 |
| 清洗 | 能定义并执行清洗规则 | 每一步有行数与理由记录 |
| 分析 | 能分组、连接、做时间趋势 | 结论可被他人复现 |
| 工程 | 代码可复用、内存可控 | 百万行数据不 OOM |
| 表达 | 图表与结论清晰 | 图能脱离正文被读懂 |

★ **关键认知**：数据分析的成长路径不是"学更多函数"，而是**积累检查清单**。每次踩坑后把"连接前查基数""读入时定 dtype""清洗每步记行数"写进自己的清单，下次就不会重复犯错。

# 13. 小结

| 主题 | 要点 |
|------|------|
| 流程 | 体检 → 清洗 → 连接 → 特征 → 聚合 → 时间 → 可视化 → 结论 |
| 清洗 | 每步核对行数，策略要有业务理由 |
| 连接 | 保留明细用 left，validate 防膨胀 |

到这里，DataAnalysis 模块的十篇就结束了。建议回到第 1 篇，用自己手头的一份真实数据完整走一遍流程：先体检，再清洗，最后用一张图和一个数字说清结论。能独立复现这条链路，才算真正掌握。
