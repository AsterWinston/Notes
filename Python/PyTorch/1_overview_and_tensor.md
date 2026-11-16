PyTorch 是当前深度学习研究和工程落地最常用的框架之一。它的核心抽象不是"层"，而是一个带设备信息、支持自动求导的多维数组——Tensor。本篇把 Tensor 的创建、形状、类型、设备、索引和运算讲清楚，为后面所有章节打底。

# 1. PyTorch 的定位

## 1.1 与 NumPy 的分工

NumPy 擅长 CPU 上的数值计算，PyTorch 则把同样的数组能力扩展到了 GPU，并在数组之上加了自动求导和神经网络模块。两者不是替代关系，而是分工关系：

| 维度 | NumPy | PyTorch |
|------|-------|---------|
| 核心类型 | `ndarray` | `Tensor` |
| 硬件 | 以 CPU 为主 | CPU、CUDA、MPS 等 |
| 自动求导 | 不支持 | 支持 |
| 神经网络 | 需要自己写 | `torch.nn` 内置 |
| 生态 | 科学计算 | 训练、推理、部署 |

如果只是做数据清洗和统计分析，NumPy 更轻；只要涉及梯度、GPU 或模型训练，就应该用 PyTorch。

## 1.2 动态图与 Python 优先

PyTorch 采用**动态计算图**：前向计算每执行一行 Python 代码，图就实时构建出来，因此可以用普通的 `if`、`for` 和调试器控制模型行为。

★ **关键认知**：动态图让"写模型"接近"写 Python"。你可以在 `forward` 里打印形状、加断点、根据输入走不同分支，而不需要先编译一张静态图。代价是每次前向都会重新建图，但 PyTorch 2.x 的 `torch.compile` 会在后台把热点图编译成更快的算子，兼顾了灵活性与性能。

# 2. Tensor 的创建

## 2.1 从 Python 数据创建

最直接的方式是 `torch.tensor`，它会复制数据并推断类型：

```python
import torch

# 从嵌套列表创建二维张量
x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
print(x)
print(x.shape)    # torch.Size([2, 2])
print(x.dtype)    # torch.float32，浮点默认是 32 位
print(x.device)   # cpu

# 整数列表会被推断为 int64
labels = torch.tensor([0, 1, 1, 0])
print(labels.dtype)  # torch.int64
```

★ **关键认知**：`torch.tensor` 会**复制**输入数据，而 `torch.as_tensor` 在类型和设备兼容时尽量共享内存。数据量大且不需要副本时用 `as_tensor`，需要独立副本时用 `tensor`。

## 2.2 常用工厂函数

实际训练里更多是用工厂函数直接生成张量：

| 函数 | 作用 | 常见用途 |
|------|------|----------|
| `torch.zeros(shape)` | 全 0 | 初始化偏置、掩码 |
| `torch.ones(shape)` | 全 1 | 掩码、计数 |
| `torch.full(shape, v)` | 填充固定值 | 常数张量 |
| `torch.empty(shape)` | 未初始化 | 追求速度、随后覆盖 |
| `torch.arange(start, end, step)` | 等差序列 | 位置索引 |
| `torch.linspace(start, end, steps)` | 等分序列 | 可视化坐标 |
| `torch.rand(shape)` | `[0, 1)` 均匀分布 | 随机初始化 |
| `torch.randn(shape)` | 标准正态分布 | 噪声、初始化 |
| `torch.randint(low, high, shape)` | 随机整数 | 伪造标签 |
| `torch.eye(n)` | 单位矩阵 | 单位阵、one-hot |

```python
zeros = torch.zeros(2, 3)          # 2 行 3 列，float32
ones = torch.ones(2, 3, dtype=torch.int64)  # 显式指定整型
seq = torch.arange(0, 10, 2)       # tensor([0, 2, 4, 6, 8])
grid = torch.linspace(0, 1, 5)     # tensor([0.0000, 0.2500, 0.5000, 0.7500, 1.0000])
noise = torch.randn(3, 4)          # 标准正态噪声
mask = torch.zeros_like(noise)     # 继承形状、dtype 和 device
```

⚠ `torch.empty` 不会清零，里面是内存里的残留值。除非常确定马上会覆盖全部元素，否则不要直接读取它。

## 2.3 dtype 的选择

dtype 决定每个元素占多少字节，也决定哪些运算能配对：

| dtype | 字节 | 说明 |
|-------|------|------|
| `torch.float32` | 4 | 默认浮点，训练首选 |
| `torch.float64` | 8 | 高精度，速度慢、显存翻倍 |
| `torch.float16` | 2 | 半精度，需配合 AMP |
| `torch.bfloat16` | 2 | 动态范围更大，适合训练 |
| `torch.int64` | 8 | 默认整型，索引和标签 |
| `torch.int32` | 4 | 节省内存的整型 |
| `torch.bool` | 1 | 掩码与比较结果 |
| `torch.uint8` | 1 | 图像像素、量化 |

```python
a = torch.tensor([1, 2, 3])                  # int64
b = torch.tensor([1.0, 2.0, 3.0])            # float32
c = a.to(torch.float32)                      # 显式转换

# 整型与浮点混算时，结果会向浮点提升
print((a + b).dtype)                         # torch.float32
```

★ **关键认知**：分类任务里，标签用 `int64`，模型输出用 `float32`；交叉熵损失要求前者是 `LongTensor`、后者是浮点。dtype 不匹配是新手最常见的报错来源之一。

# 3. 形状与设备

## 3.1 形状的三个视角

```python
x = torch.randn(2, 3, 4)
print(x.shape)   # torch.Size([2, 3, 4])
print(x.ndim)    # 3，维度数
print(x.numel()) # 24，元素总数
```

常见的维度语义：

| 形状 | 含义 |
|------|------|
| `(N,)` | N 个标量，一维 |
| `(N, C)` | N 个样本，每个 C 维特征 |
| `(N, C, H, W)` | 图像批次：样本、通道、高、宽 |
| `(T, N, C)` | 序列：时间步、批次、特征 |

`unsqueeze` 和 `squeeze` 用来增删长度为 1 的维度：

```python
v = torch.tensor([1.0, 2.0, 3.0])  # (3,)
col = v.unsqueeze(1)                # (3, 1)，变成列向量
row = v.unsqueeze(0)                # (1, 3)，变成行向量
back = col.squeeze(1)               # (3,)，再删掉
```

## 3.2 设备管理

设备用 `torch.device` 表示。最稳妥的写法是先探测再选择：

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

x = torch.randn(3, 4)
x = x.to(device)          # 搬到目标设备

# 也可以直接在创建时指定
y = torch.zeros(2, 2, device=device)

# 需要回到 CPU 时，先 cpu() 再转 NumPy
arr = x.cpu().numpy()
```

★ **关键认知**：模型和数据必须位于**同一设备**，否则前向会报 "Expected all tensors to be on the same device"。跨设备复制很慢，训练循环里不要在每一步来回搬数据，应该一次性把模型和每个 batch 放到目标设备。

⚠ 只有 CUDA 张量能调用 `.cuda()`；在没有 GPU 的机器上会直接报错。跨平台代码统一用 `.to(device)`，不要硬编码 `.cuda()`。

# 4. 索引与变形

## 4.1 基础索引

PyTorch 的索引规则与 NumPy 基本一致：

```python
x = torch.arange(12).reshape(3, 4)
print(x)

print(x[0])        # 第 0 行，形状 (4,)
print(x[:, 1])     # 第 1 列，形状 (3,)
print(x[1, 2])     # 单个元素，0 维张量
print(x[1, 2].item())  # 转成 Python 数字 6
print(x[-1, -1])   # 最后一个元素 11
print(x[0:2, 1:3]) # 子矩阵，形状 (2, 2)
```

## 4.2 布尔索引与高级索引

布尔索引会先展平再筛选，返回一维张量：

```python
x = torch.arange(12).reshape(3, 4)
mask = x % 2 == 0
print(mask)
print(x[mask])       # 取出所有偶数，一维

# 按索引列表取行
print(x[[2, 0]])     # 第 2 行和第 0 行，顺序保留

# 按列索引取元素
print(x[:, [3, 1]])
```

需要按另一个张量里的下标取值时，用 `gather`：

```python
scores = torch.tensor([[0.1, 0.6, 0.3],
                       [0.7, 0.2, 0.1]])
idx = torch.tensor([[1], [0]])       # 每个样本选一个类别
picked = scores.gather(1, idx)       # 沿第 1 维取
print(picked)                        # tensor([[0.6000], [0.7000]])
```

★ **关键认知**：`gather` 的索引张量与输出形状一致，它按维度逐元素取值，不理解"行号"这种语义。批量取行用 `index_select` 或高级索引更直观，按位置取分类分数才用 `gather`。

## 4.3 view、reshape、permute

```python
x = torch.arange(12)
a = x.view(3, 4)        # 要求内存连续，失败会报错
b = x.reshape(3, 4)     # 能 view 就 view，否则复制
c = b.permute(1, 0)     # 转置，返回的是非连续视图
print(c.is_contiguous())  # False

# 把 (N, C, H, W) 变成 (N, H, W, C)，常用于可视化
img = torch.randn(8, 3, 32, 32)
hwc = img.permute(0, 2, 3, 1)   # (8, 32, 32, 3)
```

`view` 只改变解释方式，不复制数据；`reshape` 在内存布局不允许时自动复制。`permute` 交换维度顺序但底层存储不变，所以结果通常不连续。

⚠ 对 `permute` 的结果直接调用 `view` 会报 "view size is not compatible with input tensor's size and stride"，先 `.contiguous()` 再 `view`。

# 5. 运算与广播

## 5.1 逐元素、矩阵与归约

```python
a = torch.tensor([[1.0, 2.0], [3.0, 4.0]])
b = torch.tensor([[5.0, 6.0], [7.0, 8.0]])

print(a + b)          # 逐元素加
print(a * b)          # 逐元素乘，不是矩阵乘
print(a @ b)          # 矩阵乘，等价于 torch.matmul
print(a.sum())        # 全部求和，0 维
print(a.mean(dim=1))  # 按行求均值，形状 (2,)
print(a.max(dim=1))   # 返回 (values, indices) 两个张量
```

`sum`、`mean`、`max` 这类归约默认会消掉被归约的维度；保留维度用 `keepdim=True`，方便后续广播：

```python
x = torch.randn(4, 3)
s = x.sum(dim=1, keepdim=True)   # (4, 1)
normalized = x / s               # (4, 3) / (4, 1)，按行归一化
```

## 5.2 广播规则

广播从**最右边的维度**开始对齐，逐维比较：

```text
A: (3, 1)
B:    (4)
   ↓ 右对齐
A: (3, 1)
B: (1, 4)
结果: (3, 4)
```

两条规则：

1. 维度相等，或其中一个为 1，则可以广播。
2. 缺失的维度会在左边补 1。

★ **关键认知**：广播不复制数据，只改变步长，所以几乎不额外占显存。但两个形状"碰巧能广播"时很容易掩盖逻辑错误，例如把 `(N, 1)` 和 `(N,)` 相加得到 `(N, N)`。写完运算后打印一次形状，比等报错更省时间。

## 5.3 原地运算

带下划线的版本会直接修改原张量，例如 `x.add_(1)`、`x.mul_(2)`。

原地操作省内存，但会破坏自动求导依赖的原始值。叶子张量如果 `requires_grad=True`，禁止原地修改。

⚠ 训练中不要随意对参与求导的张量做原地操作，常见报错是 "a leaf Variable that requires grad is being used in an in-place operation"。需要修改时先 `detach()` 或新建张量。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `mat1 and mat2 shapes cannot be multiplied` | 矩阵维度不匹配 | 打印形状，检查转置或 `view` |
| `expected scalar type Long but found Float` | 标签或索引 dtype 错 | 用 `.long()` 转成 int64 |
| `Expected all tensors on same device` | 模型和数据设备不同 | 统一 `.to(device)` |
| `view size is not compatible` | 张量非连续 | 先 `.contiguous()` |
| `Boolean value of Tensor is ambiguous` | 把多元素张量当条件 | 用 `.item()` 或 `.all()`、`.any()` |
| `index out of bounds` | 索引超过维度长度 | 检查 `shape` 与索引范围 |

# 7. 练习

1. 创建一个形状为 `(5, 3)` 的随机张量，按行做 L2 归一化，要求使用 `keepdim=True`，并验证每行平方和为 1。
2. 给定标签 `torch.tensor([0, 2, 1])`，用 `torch.eye(3)` 或 `scatter` 构造对应的 one-hot 矩阵，形状为 `(3, 3)`。
3. 用 `torch.topk` 找出每行最大的两个元素，构造一个同形状的布尔掩码，并只保留这些位置的值，其余置 0。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 带自动求导和 GPU 支持的多维数组框架 |
| 创建 | `tensor` 复制，`as_tensor` 尽量共享；工厂函数按需生成 |
| 形状 | `shape`、`ndim`、`numel`；`unsqueeze` 增维，`squeeze` 删维 |
| dtype | 浮点默认 float32，标签默认 int64，混算会提升 |
| 设备 | 用 `torch.device` 统一管理，模型与数据必须同设备 |
| 索引 | 基础切片、布尔掩码、高级索引、`gather` 按位置取 |
| 变形 | `view` 要求连续，`reshape` 可复制，`permute` 换轴 |
| 广播 | 右对齐，维度相等或为 1；用形状检查防隐性错误 |

下一篇进入自动求导，讲清计算图如何随前向构建、`backward` 如何累积梯度，以及 `detach`、`no_grad` 与 `inference_mode` 的边界。
