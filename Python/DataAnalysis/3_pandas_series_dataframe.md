NumPy 解决"数值算得快"，Pandas 解决"表格管得清"。它给数组加上了行标签和列标签，于是"按城市汇总""按日期对齐""按列名取数"这些业务动作才有直接表达。本篇把 Series、DataFrame、索引、读写和类型五个基础讲透。

# 1. Series

## 1.1 创建与组成

Series 是"带标签的一维数组"，由**值（values）**和**索引（index）**两部分组成。

```python
import pandas as pd

s = pd.Series([10, 20, 30], index=["a", "b", "c"], name="score")

print(s)
# a    10
# b    20
# c    30
# Name: score, dtype: int64

print(s.values)    # [10 20 30]，底层是 ndarray
print(s.index)     # Index(['a', 'b', 'c'], dtype='object')
print(s.name)      # score
```

不指定 `index` 时，Pandas 自动生成 `RangeIndex(0, 1, 2, ...)`。

## 1.2 索引与取值

```python
s = pd.Series([10, 20, 30], index=["a", "b", "c"])

print(s["b"])         # 20，按标签
print(s[1])           # 20，整数位置（标签不是整数时）
print(s[["a", "c"]])  # 按标签列表取多个，返回 Series
print(s.iloc[0])      # 10，明确按位置
print(s.loc["a"])     # 10，明确按标签
```

★ **关键认知**：`[]` 的语义是"标签优先"——当索引是整数时，`s[1]` 可能按标签也可能按位置，历史上极易混淆。**生产代码一律用 `.loc`（标签）和 `.iloc`（位置）**，不要依赖 `[]` 的模糊行为。

## 1.3 索引对齐

两个 Series 运算时，Pandas 会按**索引标签对齐**，而不是按位置：

```python
a = pd.Series([1, 2, 3], index=["x", "y", "z"])
b = pd.Series([10, 20, 30], index=["z", "y", "x"])

print(a + b)
# x    31
# y    22
# z    13
# dtype: int64
```

★ **关键认知**：对齐是 Pandas 最重要的语义，也是最容易出错的地方。**只有标签相同的位置才相加，对不上的位置变成 `NaN`**。它让"顺序不同的两张表合并"变得安全，也让"以为按位置算"的代码悄悄出错。

⚠ 对齐产生 `NaN` 后，整列 dtype 会被提升为 `float64`。后续如果做整数判断，先 `fillna` 或 `dropna`。

# 2. DataFrame

## 2.1 创建

DataFrame 是"带行索引和列名的二维表"，可以看作共享同一行索引的多个 Series。

```python
import pandas as pd

df = pd.DataFrame({
    "name": ["张三", "李四", "王五"],
    "age": [23, 31, 28],
    "city": ["北京", "上海", "北京"],
})

print(df)
#   name  age city
# 0  张三   23   北京
# 1  李四   31   上海
# 2  王五   28   北京

print(df.shape)     # (3, 3)
print(df.columns)   # Index(['name', 'age', 'city'], dtype='object')
print(df.index)     # RangeIndex(start=0, stop=3, step=1)
print(df.dtypes)
```

## 2.2 列操作

```python
df = pd.DataFrame({"a": [1, 2], "b": [3, 4]})

print(df["a"])            # 取一列，返回 Series
print(df[["a", "b"]])     # 取多列，返回 DataFrame（注意双层方括号）

df["c"] = df["a"] + df["b"]      # 新增列，按行对齐
df["d"] = 1                       # 标量广播到整列
df["e"] = [10, 20]                # 列表必须长度一致
```

⚠ **`df["a"]` 是一列，`df[["a"]]` 是只含一列的表**。前者是 Series，后者是 DataFrame，后续 `.assign`、`.merge`、绘图时的行为不同，别混。

删除列：

```python
df = df.drop(columns=["d", "e"])   # 返回新表，不修改原表
df = df.drop(columns=["d"], inplace=True)   # 原地修改，不推荐
```

## 2.3 新增与重命名

```python
df = pd.DataFrame({"a": [1, 2], "b": [3, 4]})

df = df.assign(total=df["a"] + df["b"])   # 链式风格，返回新表
```

★ **关键认知**：Pandas 大部分操作**默认返回新对象**，原对象不变。想"改成功"必须把返回值赋回去（`df = df.drop(...)`）。忘记赋值是初学者最常见的"代码没生效"原因。

# 3. 索引

## 3.1 loc 与 iloc

| 方式 | 依据 | 切片右端 | 示例 |
|------|------|---------|------|
| `.loc` | 标签 | **包含** | `df.loc[0:2]` 取 0、1、2 |
| `.iloc` | 位置 | **不包含** | `df.iloc[0:2]` 取 0、1 |

```python
df = pd.DataFrame({"a": [10, 20, 30]}, index=["x", "y", "z"])

print(df.loc["x":"y"])     # 包含 y，两行
print(df.iloc[0:2])        # 不包含 2，两行
print(df.loc["y", "a"])    # 20，标量
print(df.iloc[1, 0])       # 20
```

⚠ **`loc` 切片包含右端，`iloc` 不包含**。这是两套规则，硬记即可，不要试图统一。

## 3.2 布尔筛选

```python
df = pd.DataFrame({
    "city": ["北京", "上海", "北京", "广州"],
    "amount": [120, 80, 200, 50],
})

mask = df["amount"] > 100
print(df[mask])
print(df.loc[mask, ["city", "amount"]])

# 多条件：用 & | ~，每个条件加括号
print(df[(df["city"] == "北京") & (df["amount"] > 100)])
```

## 3.3 set_index 与 reset_index

```python
df = pd.DataFrame({"city": ["北京", "上海"], "amount": [120, 80]})

indexed = df.set_index("city")     # 把列变成行索引
print(indexed.loc["北京"])

flat = indexed.reset_index()       # 索引还原成普通列
```

★ **关键认知**：行索引是**语义标签**，不是"第几行"。需要按城市、按日期快速查找或对齐时，把它们设为索引；需要当普通字段参与运算时，`reset_index` 还原。索引选错，后面全是 `iloc` 和位置猜测。

# 4. 读写数据

## 4.1 CSV

```python
import pandas as pd

df = pd.read_csv("data/raw/sales.csv")
print(df.head())
print(df.info())          # 列名、非空数量、dtype、内存占用
print(df.describe())      # 数值列统计

df.to_csv("data/processed/sales.csv", index=False, encoding="utf-8-sig")
```

- `index=False`：不把行号写进文件，否则下次读入会多一列 `Unnamed: 0`。
- `encoding="utf-8-sig"`：带 BOM，Excel 打开中文不乱码。

## 4.2 Excel

```python
# 读取指定工作表；sheet_name=None 返回全部工作表的字典
df = pd.read_excel("data/raw/sales.xlsx", sheet_name="订单")

# 写出多个工作表
with pd.ExcelWriter("report.xlsx", engine="openpyxl") as writer:
    df.to_excel(writer, sheet_name="明细", index=False)
    summary.to_excel(writer, sheet_name="汇总", index=False)
```

读写 Excel 需要额外安装 `openpyxl`：

```bash
pip install openpyxl
```

## 4.3 读入时的关键参数

| 参数 | 作用 | 典型值 |
|------|------|--------|
| `sep` | 分隔符 | `","`、`"\t"` |
| `encoding` | 文件编码 | `"utf-8"`、`"gbk"` |
| `dtype` | 指定列类型 | `{"id": "string"}` |
| `parse_dates` | 解析日期列 | `["date"]` |
| `usecols` | 只读部分列 | `["id", "amount"]` |
| `na_values` | 额外缺失标记 | `["-", "NULL"]` |
| `nrows` | 只读前 n 行 | 调试用 |

★ **关键认知**：**读入时就指定 dtype，比读进来再转换更省内存、更少歧义**。尤其 ID、电话号码、邮编这类"看起来像数字的编号"，必须按字符串读，否则前导零会丢、长数字会变科学计数法。

# 5. 数据类型

## 5.1 常见 dtype

| dtype | 含义 | 说明 |
|-------|------|------|
| `int64` / `float64` | 数值 | 缺失值会让整数列变成 float |
| `bool` | 布尔 | 不支持缺失 |
| `object` | 任意 Python 对象 | 通常是字符串，也可能是混合类型 |
| `string` | 专用字符串类型 | 推荐显式使用 |
| `category` | 有限取值 | 重复多的低基数列省内存 |

## 5.2 可空类型

传统 `int64` 不能存 `NaN`，有一个缺失值整列就退化成 `float64`。Pandas 提供可空扩展类型：

```python
s = pd.Series([1, 2, None], dtype="Int64")      # 注意大写 I
print(s)              # 1, 2, <NA>
print(s.dtype)        # Int64

b = pd.Series([True, None, False], dtype="boolean")
t = pd.Series(["a", None], dtype="string")
```

⚠ **`Int64` 与 `int64` 是两种类型**。扩展类型用 `<NA>` 表示缺失，传统类型用 `NaN`；两者比较和聚合行为有差异，混用会看不懂结果。

## 5.3 类型转换

```python
df = pd.DataFrame({
    "amount": ["120", "80", "abc"],
    "date": ["2026-01-01", "bad", "2026-03-01"],
})

# 安全转换：无法解析的变 NaN，不抛异常
df["amount"] = pd.to_numeric(df["amount"], errors="coerce")
print(df["amount"].dtype)          # float64
df["date"] = pd.to_datetime(df["date"], errors="coerce")
```

★ **关键认知**：`astype` 遇到非法值会直接抛错，`to_numeric`/`to_datetime` 配 `errors="coerce"` 才会把非法值变 `NaN`。**清洗外部数据时优先用后者**，先转成功再检查有多少 `NaN`，别让一个脏值中断整个流程。

# 6. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| 用 `df["a"]` 当表 | 得到 Series，后续操作报错 | 取表用 `df[["a"]]` |
| 忘记赋值 | 操作"没生效" | `df = df.drop(...)` |
| 链式赋值 | `SettingWithCopyWarning` | 一步完成：`df.loc[mask, "col"] = v` |
| `loc`/`iloc` 混用 | 取错行 | 标签用 `loc`，位置用 `iloc` |
| 读 CSV 不加 `index=False` | 多出 `Unnamed: 0` | 写出时加 `index=False` |
| 用 `inplace=True` | 返回值是 `None`，链式断裂 | 优先重新赋值 |

关于链式赋值：

```python
# 错误示范：可能只改到临时副本
df[df["amount"] > 100]["flag"] = 1

# 正确做法：一次定位到位
df.loc[df["amount"] > 100, "flag"] = 1
```

# 7. 练习

1. 创建含 `name`、`city`、`amount` 三列的 DataFrame，按城市筛选金额大于 100 的记录，只保留 `name` 和 `amount` 两列。
2. 造两个索引顺序不同的 Series，相加观察对齐结果，再用 `fillna(0)` 处理缺失并说明 dtype 变化。
3. 从一个 CSV 读入数据，故意把 ID 列按默认类型读一次、按 `string` 读一次，对比两者差异并解释原因。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Series | 带标签的一维数组，运算按标签对齐 |
| DataFrame | 共享行索引的二维表，列是 Series |
| 索引 | `loc` 按标签且含右端，`iloc` 按位置且不含右端 |
| 列操作 | `df["a"]` 是 Series，`df[["a"]]` 是表 |
| 读写 | `index=False`、`utf-8-sig`、读入时定 dtype |
| 类型 | 优先显式 dtype，可空类型用 `Int64`/`string` |
| 转换 | 外部数据用 `to_numeric(..., errors="coerce")` |

下一篇进入数据清洗，系统处理缺失值、重复值、类型错误、异常值和脏字符串，把"能读进来"的数据变成"能分析"的数据。
