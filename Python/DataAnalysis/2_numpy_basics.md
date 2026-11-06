NumPy 是整个 Python 数据栈的地基。Pandas 的列、Matplotlib 的坐标、机器学习模型的输入，底层都是 `ndarray`。理解它的内存模型、形状规则和广播机制，后面写 Pandas 才不会靠猜。

# 1. ndarray 是什么

## 1.1 与 Python 列表的区别

Python 列表可以混装任意对象，每个元素都是一个指针，分散在内存各处。`ndarray` 要求**所有元素类型相同**，数据在内存中连续排列，还额外记录了形状、步长和类型。

```python
import numpy as np

lst = [1, 2, 3]
arr = np.array([1, 2, 3])

print(type(arr))        # <class 'numpy.ndarray'>
print(arr.dtype)        # int64（平台相关，64 位系统通常是 int64）
print(arr.shape)        # (3,)
print(arr.nbytes)       # 24，3 个 int64，每个 8 字节
```

★ **关键认知**：`ndarray` 快在**同类型 + 连续内存 + C 循环**。它把"逐元素解释执行"换成了"一次对整块内存做运算"，所以 `arr * 2` 比 `[x * 2 for x in lst]` 快一到两个数量级。

## 1.2 创建数组的常用方式

```python
import numpy as np

print(np.array([1, 2, 3]))              # 从列表创建
print(np.zeros((2, 3)))                 # 2 行 3 列全 0
print(np.ones((2, 3), dtype=int))       # 指定 int 类型
print(np.full((2, 2), 7))               # 全部填 7
print(np.arange(0, 10, 2))              # [0 2 4 6 8]，不含终点
print(np.linspace(0, 1, 5))             # [0. 0.25 0.5 0.75 1.]，含终点
print(np.eye(3))                        # 3 阶单位矩阵
```

`arange` 按**步长**生成，`linspace` 按**个数**生成。做数值实验时优先 `linspace`，因为终点可控、浮点误差更小。

## 1.3 shape、dtype、ndim

| 属性 | 含义 | 示例 |
|------|------|------|
| `shape` | 各维度长度，元组 | `(3, 4)` |
| `ndim` | 维度数，等于 `len(shape)` | `2` |
| `dtype` | 元素类型 | `float64` |
| `size` | 元素总数 | `12` |
| `nbytes` | 占用字节数 | `96` |

```python
arr = np.arange(12).reshape(3, 4)

print(arr.shape)     # (3, 4)
print(arr.ndim)      # 2
print(arr.size)      # 12
print(arr.dtype)     # int64
print(arr.nbytes)    # 96
```

⚠ **`reshape` 的元素总数必须一致**。`np.arange(10).reshape(3, 4)` 会抛 `ValueError`，因为 10 个元素凑不出 3×4。

用 `-1` 可以让 NumPy 自动推算一个维度：

```python
arr = np.arange(12).reshape(3, -1)   # 自动算成 (3, 4)
```

# 2. 索引与切片

## 2.1 基本索引

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6]])

print(arr[0, 1])       # 2，第 0 行第 1 列
print(arr[1])          # [4 5 6]，取第 1 行
print(arr[:, 0])       # [1 4]，取第 0 列
print(arr[0:2, 1:])    # [[2 3] [5 6]]，行切片 + 列切片
```

`arr[1]` 取的是整行，不是单个元素；要取元素必须写两个下标。这是从列表思维切换到数组思维的第一道坎。

## 2.2 布尔索引

```python
arr = np.array([3, 8, 1, 9, 5])

mask = arr > 4
print(mask)            # [False  True False  True  True]
print(arr[mask])       # [8 9 5]
print(arr[arr > 4])    # 等价写法
```

布尔索引的结果是**新数组**，不是视图。用多个条件时，`and`/`or` 不行，必须用 `&`/`|`，并且每个条件要加括号：

```python
print(arr[(arr > 2) & (arr < 9)])   # [3 8 5]
```

⚠ **`arr > 2 & arr < 9` 是错的**，`&` 的优先级高于比较运算符，会先算 `2 & arr`。必须写 `(arr > 2) & (arr < 9)`。

## 2.3 视图与副本

切片返回的是**视图**（view），与原数组共享内存；花式索引和布尔索引返回**副本**（copy）。

```python
arr = np.arange(6)
view = arr[:3]          # 切片：视图
view[0] = 100
print(arr)              # [100   1   2   3   4   5] 原数组被改

arr = np.arange(6)
copy = arr[[0, 1, 2]]   # 花式索引：副本
copy[0] = 100
print(arr)              # [0 1 2 3 4 5] 原数组不变
```

★ **关键认知**：视图不复制数据，省内存也省时间，但**修改视图会改到原数组**。判断标准是"索引是否连续"：基本切片是视图，花式索引、布尔索引是副本。不确定时用 `.base` 检查或显式 `.copy()`。

```python
print(arr[:3].base is arr)   # True，说明是视图
```

# 3. 广播

## 3.1 广播规则

两个形状不同的数组做运算时，NumPy 会尝试"广播"到相同形状。规则从**尾部维度**开始逐一对齐：

1. 维度相等，或其中一个为 1，则兼容。
2. 缺失的维度视为 1。
3. 无法对齐时抛 `ValueError`。

```text
A: (3, 4)       B: (4,)        →  (3, 4)    B 被广播到每一行
A: (3, 1)       B: (1, 4)      →  (3, 4)    两个方向都扩展
A: (3, 4)       B: (3,)        →  报错      尾部 4 与 3 不兼容
```

## 3.2 典型例子

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6]])       # shape (2, 3)

row = np.array([10, 20, 30])      # shape (3,)
print(arr + row)
# [[11 22 33]
#  [14 25 36]]  每一行都加同一个行向量

col = np.array([[100], [200]])    # shape (2, 1)
print(arr + col)
# [[101 102 103]
#  [204 205 206]]  每一列加同一个列向量
```

★ **关键认知**：广播的本质是**不实际复制数据**，而是让计算时按规则重复读取同一块内存。因此它既省内存又快。但它也容易悄悄产生意料之外的形状，运算后要检查 `result.shape`。

⚠ **广播不会替你检查业务含义**。`(2, 3)` 加 `(3,)` 看起来合理，但如果你本意是"每行加不同的值"，却传成了 `(3,)`，代码照样运行，结果全错。

# 4. 向量化

## 4.1 用数组运算替代循环

```python
import numpy as np

arr = np.arange(100000)

# 错误示范：Python 逐元素循环
result = [x * 2 + 1 for x in arr]

# 正确做法：整数组向量化
result = arr * 2 + 1
```

向量化不仅代码短，而且把循环交给了底层 C，通常快几十倍。

## 4.2 通用函数 ufunc

NumPy 的加减乘除、`np.sqrt`、`np.exp`、`np.log`、`np.abs` 等都是 ufunc，逐元素作用且返回新数组。

```python
arr = np.array([1.0, 4.0, 9.0])

print(np.sqrt(arr))          # [1. 2. 3.]
print(np.where(arr > 4, 1, 0))   # [0 0 1]，条件选择
```

⚠ **`np.where` 的两个分支都会先求值**。像 `np.where(x > 0, 1 / x, 0)` 在 `x=0` 时仍可能触发除零警告，需要先处理分母。

## 4.3 聚合也是向量化

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6]])

print(arr.sum())          # 21，全部求和
print(arr.mean())         # 3.5
print(arr.sum(axis=0))    # [5 7 9]，按列求和
print(arr.sum(axis=1))    # [6 15]，按行求和
```

# 5. 常用统计与 axis

## 5.1 常用函数

| 函数 | 作用 | 备注 |
|------|------|------|
| `sum` / `mean` | 求和 / 均值 | 空数组求均值得到 `nan` |
| `std` / `var` | 标准差 / 方差 | 默认总体标准差（`ddof=0`） |
| `min` / `max` | 最值 | 可用 `argmin`/`argmax` 取位置 |
| `median` | 中位数 | 对异常值更稳健 |
| `percentile` | 分位数 | `np.percentile(a, 95)` |
| `cumsum` | 累加 | 返回同形状数组 |

```python
scores = np.array([88, 92, 79, 95, 61])

print(scores.mean())                  # 83.0
print(scores.std())                   # 12.25...
print(np.median(scores))              # 88.0
print(np.percentile(scores, 90))      # 93.8
print(scores.cumsum())                # [ 88 180 259 354 415]
```

## 5.2 axis 到底怎么记

`axis=0` 表示"沿着第 0 维（行）压缩"，结果消掉行，得到每列一个值；`axis=1` 沿着列压缩，得到每行一个值。

```text
arr.shape = (3, 4)

axis=0：跨行聚合，结果 shape (4,)   ← 列方向压缩
axis=1：跨列聚合，结果 shape (3,)   ← 行方向压缩
```

★ **关键认知**：记 `axis` 不要背"0 是行、1 是列"，而要记**"这个轴被消掉了"**。`sum(axis=0)` 消掉第 0 维，剩下的就是列数。这个记法对任意维数组都成立。

⚠ **`std` 默认是总体标准差（除以 n）**。做样本推断时要写 `ddof=1`，否则结果偏小，样本量小时尤其明显。

# 6. 常见坑

| 坑 | 现象 | 正确做法 |
|----|------|---------|
| Python 列表当数组用 | 运算慢、无广播 | 先 `np.array(...)` |
| 用 `and`/`or` 组合条件 | `ValueError` | 用 `&`/`|` 并加括号 |
| 切片后改视图 | 原数组被意外修改 | 需要独立数据时 `.copy()` |
| 广播形状不检查 | 结果正确但含义错误 | 运算后打印 `shape` 核对 |
| 混淆 `/` 与 `//` | 得到浮点或向下取整 | 明确用真除或整除 |
| 用 `np.float_` / `np.int_` | NumPy 2.0 已移除别名 | 用 `np.float64` / `int` |
| 把 `nan` 当普通值比较 | `nan == nan` 为 `False` | 用 `np.isnan` |

关于空值：

```python
arr = np.array([1.0, np.nan, 3.0])
print(arr.sum())              # nan，nan 会污染聚合
print(np.nansum(arr))         # 4.0，跳过 nan
print(np.isnan(arr))          # [False  True False]
```

⚠ **NumPy 2.0 移除了 `np.float_`、`np.complex_` 等别名**，旧教程里的写法会直接 `AttributeError`。新代码统一用 `np.float64`、`np.complex128`。

# 7. 练习

1. 创建 `(4, 5)` 的随机整数数组，分别求每列最大值、每行均值，并说明两个结果的 shape。
2. 给定 `arr = np.arange(20).reshape(4, 5)`，用布尔索引取出所有大于 10 的偶数，再用切片取出第 2、3 行的后 3 列。
3. 用广播实现"对 10 个样本、3 个特征的数据做标准化"：每个特征减去该列均值、再除以该列标准差，要求不写循环。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `ndarray` | 同类型、连续内存，运算由底层 C 循环完成 |
| shape/dtype | 形状与类型决定内存布局，运算前先核对 |
| 索引 | 基本切片是视图，布尔/花式索引是副本 |
| 广播 | 从尾部对齐，维度相等或为 1 才兼容，不实际复制 |
| 向量化 | 用整数组运算替代 Python 循环 |
| axis | 记"被消掉的轴"，0 消行、1 消列 |
| 缺失值 | `nan` 会污染聚合，用 `nansum`/`isnan` 处理 |

下一篇进入 Pandas，在 `ndarray` 之上加"行标签和列标签"，讲清 Series 与 DataFrame 的索引语义、列操作、CSV/Excel 读写和 dtype 选择。
