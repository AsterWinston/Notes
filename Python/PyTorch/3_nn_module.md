`nn.Module` 是 PyTorch 里所有神经网络模型的基类。它看似只是"把层装在一起"，真正的作用是**自动注册参数、切换训练状态、递归管理子模块**。不理解这三点，就会写出"模型能跑但参数没更新""`.to(device)` 漏掉一半层"这类隐蔽 bug。

# 1. 为什么需要 nn.Module

## 1.1 参数管理的痛点

如果手写模型，参数就是一个普通张量列表。保存时要自己收集，搬到 GPU 时要自己遍历，优化器还要再收集一遍。层一多，漏掉一个就是灾难。

`nn.Module` 通过属性赋值自动完成注册：

```python
import torch
from torch import nn

class Tiny(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc = nn.Linear(4, 2)      # 自动注册为子模块
        self.scale = nn.Parameter(torch.ones(2))  # 自动注册为参数

    def forward(self, x):
        return self.fc(x) * self.scale
```

只要写成 `self.xxx = nn.Module(...)` 或 `self.xxx = nn.Parameter(...)`，它就会出现在 `parameters()`、`state_dict()` 和 `.to(device)` 的遍历结果里。

★ **关键认知**：`nn.Module` 的核心不是 `forward`，而是**属性赋值触发的注册机制**。用 `list`、`dict` 直接装层不会注册，`model.parameters()` 看不到，`.to(device)` 也不会搬——这是新手最隐蔽的错误之一。

## 1.2 训练状态

`model.train()` 和 `model.eval()` 会递归切换所有子模块的状态。`Dropout` 在训练时随机置零、评估时关闭；`BatchNorm` 在训练时用当前 batch 统计量、评估时用滑动平均。忘记切换，评估结果就会不稳定。

# 2. 定义一个模块

## 2.1 `__init__` 与 `forward`

一个标准模块包含两部分：`__init__` 里声明层，`forward` 里定义计算流程。

```python
class MLP(nn.Module):
    def __init__(self, in_dim=784, hidden=128, num_classes=10):
        super().__init__()                    # 必须调用，否则注册机制不工作
        self.fc1 = nn.Linear(in_dim, hidden)
        self.act = nn.ReLU()
        self.dropout = nn.Dropout(0.2)
        self.fc2 = nn.Linear(hidden, num_classes)

    def forward(self, x):
        x = self.fc1(x)        # (N, in_dim) -> (N, hidden)
        x = self.act(x)
        x = self.dropout(x)
        return self.fc2(x)     # (N, num_classes)
```

使用时调用 `model(x)` 而不是 `model.forward(x)`：

```python
model = MLP()
x = torch.randn(32, 784)
logits = model(x)
print(logits.shape)   # torch.Size([32, 10])
```

⚠ `model(x)` 会走 `__call__`，其间会执行前向 hook、处理 `torch.compile` 包装；直接调 `model.forward(x)` 会绕过这些机制。除非在 `forward` 内部复用逻辑，否则永远用 `model(x)`。

## 2.2 组合子模块

模块可以嵌套，父模块会自动递归管理子模块：

```python
class Block(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.norm = nn.LayerNorm(dim)
        self.fc = nn.Linear(dim, dim)
        self.act = nn.GELU()

    def forward(self, x):
        return x + self.act(self.fc(self.norm(x)))  # 残差连接

class Net(nn.Module):
    def __init__(self, dim=64, depth=4):
        super().__init__()
        self.blocks = nn.ModuleList([Block(dim) for _ in range(depth)])
        self.head = nn.Linear(dim, 10)

    def forward(self, x):
        for block in self.blocks:
            x = block(x)
        return self.head(x)
```

结构图：

```text
x (N, 64)
  │
  ▼
┌───────────────────────────────┐
│ Block × 4                     │
│  LayerNorm → Linear → GELU    │
│      └──── 残差加回 ────┘      │
└───────────────────────────────┘
  │
  ▼
Linear(64 → 10)
```

## 2.3 Sequential、ModuleList、ModuleDict

| 容器 | 是否注册 | 是否有 `forward` | 适用场景 |
|------|----------|------------------|----------|
| `nn.Sequential` | 是 | 有，按顺序执行 | 线性堆叠的层 |
| `nn.ModuleList` | 是 | 无，需要自己循环 | 循环、跳连、可变深度 |
| `nn.ModuleDict` | 是 | 无，按 key 取 | 多分支、多任务头 |
| 普通 `list` | 否 | 无 | 只放非模块对象 |

```python
# 顺序结构，最省事
self.stem = nn.Sequential(
    nn.Linear(784, 256),
    nn.ReLU(),
    nn.Linear(256, 128),
)

# 需要循环或条件分支时用 ModuleList
self.layers = nn.ModuleList([nn.Linear(128, 128) for _ in range(3)])
for layer in self.layers:
    x = layer(x)
```

★ **关键认知**：判断标准不是"能不能存"，而是"`.parameters()` 能否遍历到"。凡是参与训练、需要保存或搬设备的层，必须用 `nn.Module` 容器包起来。

# 3. 参数与缓冲

## 3.1 Parameter

`nn.Parameter` 是自动注册为可训练参数的张量：

```python
class Scale(nn.Module):
    def __init__(self):
        super().__init__()
        self.weight = nn.Parameter(torch.ones(1))

    def forward(self, x):
        return x * self.weight
```

普通张量赋值不是参数：

```python
self.register_buffer("running_mean", torch.zeros(dim))  # 缓冲：保存但不训练
self.some_tensor = torch.zeros(dim)                     # 普通属性：不保存、不搬设备
```

## 3.2 register_buffer

缓冲适合存"需要跟着模型走、但不参与梯度"的状态，例如 BatchNorm 的滑动均值、位置编码表：

```python
class WithBuffer(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.register_buffer("pos", torch.zeros(1, dim))

    def forward(self, x):
        return x + self.pos
```

缓冲会出现在 `state_dict()` 里，也会被 `.to(device)` 搬走，但不会进优化器。

★ **关键认知**：`Parameter` 与 `buffer` 都会跟随模型搬设备并进入 `state_dict`，区别只在优化器是否更新它。需要持久化但不需要训练的张量，一律用 `register_buffer`，不要用普通属性。

## 3.3 查看参数

```python
model = MLP()
for name, p in model.named_parameters():
    print(name, tuple(p.shape), p.requires_grad)

total = sum(p.numel() for p in model.parameters() if p.requires_grad)
print(f"可训练参数：{total}")
```

输出类似：

```text
fc1.weight (128, 784) True
fc1.bias (128,) True
fc2.weight (10, 128) True
fc2.bias (10,) True
```

⚠ `nn.Linear` 的权重形状是 `(out_features, in_features)`，不是转置。自己手写矩阵乘时最容易搞反。

# 4. 初始化

## 4.1 默认初始化

`nn.Linear`、`nn.Conv2d` 在构造时会做默认初始化（通常是 Kaiming 均匀分布），大多数情况下够用。需要自定义时，在 `__init__` 末尾统一处理。

## 4.2 用 apply 批量初始化

```python
def init_weights(m):
    if isinstance(m, nn.Linear):
        nn.init.kaiming_normal_(m.weight, nonlinearity="relu")
        nn.init.zeros_(m.bias)

model = MLP()
model.apply(init_weights)   # 递归应用到每个子模块
```

常用初始化：

| 函数 | 适用 | 特点 |
|------|------|------|
| `kaiming_normal_` | ReLU 网络 | 保持方差，配合 ReLU |
| `xavier_uniform_` | tanh、线性层 | 兼顾前后向方差 |
| `normal_` | 自定义 | 指定均值和标准差 |
| `zeros_` / `ones_` | 偏置 | 常见偏置初始化 |
| `orthogonal_` | RNN、深层网络 | 保持正交性 |

★ **关键认知**：初始化只应在**训练开始前**做一次。加载 `state_dict` 之后再 `apply` 初始化，会把刚加载的权重覆盖掉，表现为"训练完全不收敛"。

# 5. train 与 eval

```python
model.train()   # 训练模式：Dropout 生效，BatchNorm 用 batch 统计量
model.eval()    # 评估模式：Dropout 关闭，BatchNorm 用 running 统计量

with torch.no_grad():
    logits = model(x_val)
```

⚠ `model.eval()` **不会**关闭梯度，它只切换层的行为。评估时还要加 `torch.no_grad()` 或 `torch.inference_mode()`，否则照样建图、占显存。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 参数没更新 | 层放在普通 list 里，没注册 | 改用 `ModuleList` |
| `.to(device)` 后仍报设备不一致 | 有张量是普通属性 | 用 `register_buffer` 或 `nn.Parameter` |
| `forward` 报维度错 | 线性层输入维度不符 | 打印 `x.shape` 与 `in_features` |
| 评估结果每次不同 | 忘记 `model.eval()` | 评估前切换模式 |
| 加载权重后效果变差 | 加载后又执行了初始化 | 调整顺序，先初始化再加载 |
| 忘记 `super().__init__()` | 注册机制未初始化 | 首行补上 |

# 7. 练习

1. 用 `nn.ModuleList` 写一个深度可配置的 MLP，要求 `depth` 改变时参数量随之变化，并打印每层形状。
2. 给模型加一个可学习的缩放参数 `nn.Parameter`，验证它出现在 `named_parameters()` 里，而普通张量属性不出现。
3. 构造一个带 `Dropout` 的模型，分别在 `train()` 和 `eval()` 下对同一输入前向两次，比较输出差异。

```python
# 练习 3 的起点
model = nn.Sequential(nn.Linear(10, 10), nn.Dropout(0.5))
x = torch.randn(4, 10)

model.train()
print(model(x))
model.eval()
print(model(x))   # 两次 eval 输出应一致
```

# 8. 小结

| 主题 | 要点 |
|------|------|
| 注册 | 属性赋值为 `Module` 或 `Parameter` 才会被管理 |
| forward | 定义计算，外部用 `model(x)` 调用 |
| 容器 | 顺序用 `Sequential`，循环用 `ModuleList`，分支用 `ModuleDict` |
| 参数 | `Parameter` 可训练，`buffer` 保存但不训练 |
| 初始化 | 训练前用 `apply` 统一处理，加载后不要再初始化 |
| 模式 | `train` 与 `eval` 影响 Dropout、BatchNorm，评估还要关梯度 |

下一篇进入数据管道，讲清 `Dataset` 的两种风格、`DataLoader` 的批处理与多进程参数，以及 `collate_fn` 如何处理变长样本。
