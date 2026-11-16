真实分析很少只面对一张表：订单在订单表，商品在商品表，用户在用户表。要回答"每个城市的客单价"，就得先把它们按共同的键拼起来。连接写错，后面的聚合再精确也是错的。本篇把 `merge`、`join`、`concat` 的语义、连接类型和性能讲清楚。

# 1. 为什么需要连接

## 1.1 宽表与长表

```text
订单表 orders                 用户表 users
┌────────┬────────┬────────┐  ┌────────┬────────┐
│ order  │ user_id│ amount │  │ user_id│ city   │
├────────┼────────┼────────┤  ├────────┼────────┤
│ A001   │ 1      │ 100    │  │ 1      │ 北京   │
│ A002   │ 2      │ 80     │  │ 2      │ 上海   │
│ A003   │ 1      │ 60     │  │ 3      │ 广州   │
└────────┴────────┴────────┘  └────────┴────────┘
                 │ 按 user_id 连接
                 ▼
┌────────┬────────┬────────┬────────┐
│ order  │ user_id│ amount │ city   │
├────────┼────────┼────────┼────────┤
│ A001   │ 1      │ 100    │ 北京   │
│ A002   │ 2      │ 80     │ 上海   │
│ A003   │ 1      │ 60     │ 北京   │
└────────┴────────┴────────┴────────┘
```

## 1.2 连接的本质

连接就是**按一个或多个键，把两行从两张表里匹配到一起**。理解三件事就不会乱：

1. **键相等才匹配**：`orders.user_id == users.user_id`。
2. **匹配结果可能是 0 行、1 行或多行**：这由两边键的重复程度决定。
3. **连接类型决定"匹配不上的行"怎么处理**：保留、丢弃，还是补空。

★ **关键认知**：连接前必须检查**键的唯一性**。左表键唯一、右表键唯一是一对一；左表重复是一对多；两边都重复会导致**笛卡尔式膨胀**，行数暴涨，这是连接事故的头号原因。

# 2. merge：按列连接

## 2.1 how 的四种取值

```python
import pandas as pd

orders = pd.DataFrame({
    "order": ["A001", "A002", "A003", "A004"],
    "user_id": [1, 2, 1, 9],
    "amount": [100, 80, 60, 50],
})

users = pd.DataFrame({
    "user_id": [1, 2, 3],
    "city": ["北京", "上海", "广州"],
})

print(orders.merge(users, on="user_id", how="inner"))
print(orders.merge(users, on="user_id", how="left"))
print(orders.merge(users, on="user_id", how="right"))
print(orders.merge(users, on="user_id", how="outer"))
```

| how | 保留哪些行 | 未匹配的一侧 |
|-----|-----------|-------------|
| `inner` | 两边都匹配上 | 丢弃 |
| `left` | 左表全部 | 右表字段填 `NaN` |
| `right` | 右表全部 | 左表字段填 `NaN` |
| `outer` | 两边全部 | 缺失侧填 `NaN` |
| `cross` | 笛卡尔积 | 不按键匹配 |

```text
inner:      左 ∩ 右
left:       左 全部 + 右匹配部分
right:      右 全部 + 左匹配部分
outer:      左 ∪ 右
cross:      左 × 右（行数 = 左行数 × 右行数）
```

★ **关键认知**：**默认是 `inner`**。很多人以为 `merge` 默认保留左表，结果发现行少了，其实是内连接把没匹配上的订单丢了。需要保留明细时，几乎都应该显式写 `how="left"`。

## 2.2 on、left_on、right_on

键名相同时用 `on`，不同时分别指定：

```python
orders.merge(users, left_on="user_id", right_on="uid")
```

如果两边键名不同又不想结果里保留两份，用 `suffixes` 或连接后删列：

```python
merged = orders.merge(users, left_on="user_id", right_on="uid")
merged = merged.drop(columns=["uid"])
```

多键连接传列表：

```python
left.merge(right, on=["date", "store_id"], how="left")
```

## 2.3 键冲突与 suffixes

两张表有同名列（非连接键）时，Pandas 自动加后缀 `_x`、`_y`：

```python
left = pd.DataFrame({"id": [1], "value": [10]})
right = pd.DataFrame({"id": [1], "value": [20]})

print(left.merge(right, on="id"))
#    id  value_x  value_y
# 0   1       10       20
```

显式指定更清楚：

```python
left.merge(right, on="id", suffixes=("_order", "_user"))
```

⚠ **`_x`/`_y` 是默认占位符，不是业务语义**。多表连续连接后，`value_x` 到底来自哪张表会彻底失忆。每次连接都显式写 `suffixes`。

## 2.4 indicator 与 validate

`indicator=True` 会增加一列 `_merge`，标明每行来自哪边：

```python
result = orders.merge(users, on="user_id", how="left", indicator=True)
print(result["_merge"].value_counts())
# both          3
# left_only     1
```

这比手动 `isna()` 更直观，适合核对"有多少订单没匹配上用户"。

`validate` 用来强制检查连接基数，防止意外膨胀：

```python
# 断言右表 user_id 唯一，否则抛 MergeError
orders.merge(users, on="user_id", how="left", validate="many_to_one")
```

| validate 取值 | 含义 |
|---------------|------|
| `"one_to_one"` | 两边键都唯一 |
| `"one_to_many"` | 左唯一，右可重复 |
| `"many_to_one"` | 左可重复，右唯一 |
| `"many_to_many"` | 两边都可重复 |

★ **关键认知**：**连接后先看行数**。`left` 连接的行数应该等于左表行数（右表键唯一时）；如果变多了，说明右表键重复，产生了膨胀。`validate="many_to_one"` 能让这个问题在代码里直接报错，而不是在结论里悄悄错。

# 3. join：按索引连接

`DataFrame.join` 是 `merge` 的语法糖，默认按**索引**对齐，默认 `how="left"`：

```python
orders_idx = orders.set_index("user_id")
users_idx = users.set_index("user_id")

print(orders_idx.join(users_idx, how="left"))
```

| 对比 | `merge` | `join` |
|------|---------|--------|
| 连接依据 | 列或索引 | 默认索引 |
| 默认 how | `inner` | `left` |
| 适合 | 显式列键、复杂连接 | 索引已对齐的快速拼接 |
| 参数 | 更丰富 | 更简洁 |

⚠ **`join` 默认是左连接，`merge` 默认是内连接**。两者默认行为相反，混用时最容易出错。

# 4. concat：拼接

`concat` 不按键匹配，只是把多张表**堆在一起**。

## 4.1 纵向拼接（axis=0）

```python
jan = pd.DataFrame({"city": ["北京"], "amount": [100]})
feb = pd.DataFrame({"city": ["上海"], "amount": [80]})

combined = pd.concat([jan, feb], ignore_index=True)
print(combined)
#   city  amount
# 0   北京     100
# 1   上海      80
```

- `ignore_index=True`：重建 0 到 n-1 的行号；不加会保留原索引，出现重复行号。
- `keys=["jan", "feb"]`：增加一层来源标识，生成 MultiIndex。

```python
pd.concat([jan, feb], keys=["jan", "feb"])
```

## 4.2 横向拼接（axis=1）

```python
left = pd.DataFrame({"a": [1, 2]}, index=[0, 1])
right = pd.DataFrame({"b": [3, 4]}, index=[0, 1])

pd.concat([left, right], axis=1)
#    a  b
# 0  1  3
# 1  2  4
```

⚠ **`axis=1` 按索引对齐，不是按位置**。索引不一致时会补 `NaN`；如果本意是"按顺序并排"，先 `reset_index(drop=True)`。

★ **关键认知**：`concat` 处理**结构相同的数据堆叠**，`merge` 处理**结构不同但键相同的数据关联**。用 `concat` 去"按 user_id 拼两张表"是典型误用，结果只会按行号硬对齐。

# 5. 连接基数与行数变化

| 左表键 | 右表键 | 基数 | 结果行数 |
|--------|--------|------|---------|
| 唯一 | 唯一 | 一对一 | 不超过左表行数 |
| 重复 | 唯一 | 多对一 | 等于左表行数 |
| 唯一 | 重复 | 一对多 | 等于右表匹配行数 |
| 重复 | 重复 | 多对多 | 可能远大于两表 |

```python
# 检查键是否唯一
print(users["user_id"].is_unique)          # True / False
print(users["user_id"].duplicated().sum()) # 重复数量

# 检查连接前后行数
before = len(orders)
after = len(orders.merge(users, on="user_id", how="left"))
print(before, after)
```

★ **关键认知**：多对多连接不是"错误"，但必须是有意为之。比如"订单 × 促销活动"按日期区间匹配，本来就会膨胀；如果没意识到，会把销售额重复计算好几倍。

# 6. 性能

连接是内存密集操作，数据量大时差异明显。

## 6.1 让键的类型更省

```python
# 宽整型占内存、哈希也慢；两边统一降成 int32
users["user_id"] = users["user_id"].astype("int32")
orders["user_id"] = orders["user_id"].astype("int32")

# 字符串键可转 category，但两边类别集合必须一致
# orders["city"] = orders["city"].astype("category")
```

## 6.2 减少参与连接的数据

```python
# 只取需要的列，别把整张宽表拖进来
small_users = users[["user_id", "city"]]
result = orders.merge(small_users, on="user_id", how="left")
```

## 6.3 优先索引对齐

```python
# 两边都以 user_id 为索引时，join 走哈希查找，通常比列 merge 快
orders.set_index("user_id").join(users.set_index("user_id"), how="left")
```

| 优化手段 | 效果 | 注意 |
|---------|------|------|
| 只选必要列 | 减少内存与拷贝 | 连接后不用再 drop |
| 键转 `category`/窄整型 | 降低哈希成本 | 两边 dtype 要兼容 |
| 先过滤再连接 | 减少行数 | 过滤条件不能依赖右表 |
| 避免多对多 | 防止行数爆炸 | 用 `validate` 提前拦截 |

⚠ **连接前先过滤，往往比连接后过滤快一个数量级**。但若过滤条件用到右表字段，就必须先连接，别为了性能把逻辑写错。

# 7. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 默认 `inner` 丢行 | 明细变少 | 需要保留写 `how="left"` |
| 右表键重复 | 行数翻倍 | 先检查 `is_unique` 或 `validate` |
| `_x`/`_y` 分不清 | 不知道列来自哪张表 | 显式 `suffixes` |
| `join` 当 `merge` | 默认连接类型不同 | 记住 `join` 默认 `left` |
| 连接后不核对行数 | 悄悄膨胀 | 比较 `before`/`after` |
| 键类型不一致 | 匹配不上或变慢 | 两边统一 dtype |

键类型不一致的典型例子：一边 `int64`，一边 `string`，Pandas 可能不报错但一行都匹配不上，结果全是 `NaN`。

# 8. 练习

1. 造订单表和用户表，分别用四种 `how` 连接，记录每种结果的行数，并解释差异。
2. 故意让右表键重复，观察连接后行数变化；再用 `validate="many_to_one"` 复现报错。
3. 把两张结构相同的月度表用 `concat` 纵向合并，要求索引连续，并增加一列标明来源月份。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 连接本质 | 按键匹配，匹配数由键的重复程度决定 |
| merge | 默认 `inner`，保留明细用 `left` |
| validate | 强制检查基数，防止多对多膨胀 |
| concat | 纵向堆叠或横向按索引对齐 |
| 性能 | 选必要列、窄 dtype、先过滤、索引连接 |

下一篇进入时间序列，讲清 `DatetimeIndex`、`resample` 重采样、`rolling` 窗口、时区处理和缺失时间补全，把"按时间看趋势"这件事做对。
