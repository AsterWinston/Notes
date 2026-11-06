反向传播是深度学习训练的核心，而 PyTorch 的 `autograd` 把这件事做成了"前向正常算，反向一行 `backward()`"。但真正用起来，梯度为什么是 `None`、为什么越累越大、为什么第二次反向就报错，几乎每个初学者都会踩一遍。本篇把计算图、梯度累积和控制求导讲透。

# 1. 自动求导解决什么问题

## 1.1 手写梯度的代价

一个简单的损失函数可能由几十个算子组成。手写链式法则不仅容易错，而且每改一次模型结构就要重推一遍。自动求导把"记录运算"和"应用链式法则"交给框架，人只需要定义前向。

## 1.2 一个最小示例

```python
import torch

# 标量张量，开启梯度追踪
x = torch.tensor(2.0, requires_grad=True)

# 前向：y = x^2 + 3x
y = x ** 2 + 3 * x

# 反向：dy/dx = 2x + 3 = 7
y.backward()

print(x.grad)  # tensor(7.)
```

`requires_grad=True` 是开关。没有它，PyTorch 不会为这个张量记录任何运算，`x.grad` 永远是 `None`。

★ **关键认知**：自动求导不是"自动推导公式"，而是**在前向时把每个算子及其输入记录下来，反向时按相反顺序调用每个算子的梯度函数**。所以前向必须真的执行过，图才存在；图一旦释放，就不能再次反向。

# 2. 计算图

## 2.1 前向即建图

上面代码执行时，内存里形成的结构大致是：

```text
x (leaf, requires_grad=True)
├──► pow(2) ──┐
│              ├──► add ──► y (requires_grad=True, grad_fn=AddBackward0)
└──► mul(3) ──┘
```

每个非叶子节点都带一个 `grad_fn`，它知道"怎么把上游传来的梯度传给自己的输入"。反向就是从 `y` 出发，沿着这些函数往回走。

```python
print(y.grad_fn)     # <AddBackward0 object>
print(y.grad_fn.next_functions)  # 指向上游的 PowBackward 和 MulBackward
```

## 2.2 叶子节点与 grad_fn

```python
x = torch.tensor(2.0, requires_grad=True)
y = x ** 2

print(x.is_leaf)   # True，用户直接创建的张量
print(y.is_leaf)   # False，运算产生的中间结果
print(y.grad_fn)   # PowBackward0
print(x.grad_fn)   # None，叶子没有 grad_fn
```

默认只有叶子节点会保存 `.grad`。如果想看中间结果的梯度，需要显式保留：

```python
y.retain_grad()    # 在 backward 之前调用
y.backward()
print(y.grad)      # tensor(1.)
```

★ **关键认知**：`.grad` 只对**叶子张量**自动累积，这是为了省显存——中间结果可能有几十个，全都存梯度会爆炸。模型参数是叶子，所以能看到梯度；中间激活不是，需要调试时用 `retain_grad()` 或 hook。

# 3. backward 的细节

## 3.1 标量与向量

`backward()` 不带参数时，只允许在**标量**上调用。向量输出需要传入一个与输出同形状的"上游梯度"：

```python
x = torch.tensor([1.0, 2.0, 3.0], requires_grad=True)
y = x ** 2                  # 形状 (3,)

# y.backward()              # 报错：grad can be implicitly created only for scalar outputs
y.backward(torch.ones_like(y))  # 等价于对 sum(y) 求导
print(x.grad)               # tensor([2., 4., 6.])
```

这里的 `torch.ones_like(y)` 表示"最终损失对 y 每个元素的梯度都是 1"，通常对应 `loss = y.sum()`。如果损失是加权和，就把权重传进去。

## 3.2 梯度累积

PyTorch 的 `backward()` 是**累加**而不是覆盖：

```python
x = torch.tensor(2.0, requires_grad=True)

(x ** 2).backward()
print(x.grad)   # tensor(4.)

(x ** 3).backward()
print(x.grad)   # tensor(16.)，4 + 12，第二次的结果叠加上去了
```

训练循环里必须每步清零，否则梯度会跨 batch 累积：

```python
optimizer.zero_grad(set_to_none=True)  # 推荐，省一次写零
loss.backward()
optimizer.step()
```

★ **关键认知**：梯度累积是**特性**不是 bug。显存不够时，可以故意不清零，把多个小 batch 的梯度加起来再 `step()`，等效于大 batch。但正常训练忘记清零，表现为 loss 震荡、参数更新幅度越来越大。

## 3.3 retain_graph 与多次反向

默认 `backward()` 结束后会释放计算图。对同一张图再次反向会报错：

```python
x = torch.tensor(2.0, requires_grad=True)
y = x ** 2
y.backward()
# y.backward()  # RuntimeError: Trying to backward through the graph a second time
```

确实需要多次反向时传 `retain_graph=True`：

```python
y.backward(retain_graph=True)
y.backward()   # 这次可以，但图在使用后释放
```

大多数场景不需要它。想对多个损失反向，更常见的写法是把损失相加后只调用一次：

```python
total = loss_a + loss_b
total.backward()
```

⚠ `retain_graph=True` 会保留整张图，显存占用上升。若只是为了拿梯度做分析，用 `torch.autograd.grad` 更合适，它不会往 `.grad` 里累积。

# 4. 控制求导

## 4.1 requires_grad 的开关

```python
x = torch.randn(3, 3)
x.requires_grad_(True)     # 原地打开
print(x.requires_grad)     # True

# 冻结一部分参数：微调时常用
for p in model.parameters():
    p.requires_grad = False
```

冻结后这些参数不参与反向，但前向照常。解冻时把 `requires_grad` 设回 `True` 即可。

## 4.2 detach

`detach()` 返回一个与当前张量共享存储、但脱离计算图的新张量：

```python
x = torch.tensor(2.0, requires_grad=True)
y = x ** 2

z = y.detach()
print(z.requires_grad)  # False
print(z.grad_fn)        # None

# 共享存储：改 z 会影响 y
z.add_(1)
print(y)                # tensor(5., grad_fn=PowBackward0)
```

如果需要"值相同但完全独立"，用 `y.detach().clone()`。

★ **关键认知**：`detach()` 切断的是**梯度路径**，不是内存关系。它常用于目标网络、停止梯度、把张量转成 NumPy 前脱图。想彻底独立就再 `clone()`。

## 4.3 no_grad 与 inference_mode

评估和推理不需要建图，关闭它能省显存、提速：

```python
model.eval()
with torch.no_grad():
    logits = model(features)
    pred = logits.argmax(dim=1)

# 更严格的推理模式
with torch.inference_mode():
    logits = model(features)
```

两者对比：

| 特性 | `no_grad` | `inference_mode` |
|------|-----------|------------------|
| 关闭梯度记录 | 是 | 是 |
| 版本计数检查 | 保留 | 关闭，更快 |
| 产物能否再参与求导 | 可以 | 不可以 |
| 适用场景 | 评估、部分冻结训练 | 纯推理 |

⚠ `inference_mode` 下创建的张量不能再用于需要梯度的计算。如果后续还要接着训练，用 `no_grad`；确定只做推理时才用 `inference_mode`。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `x.grad` 是 `None` | 没开 `requires_grad` 或没反向 | 检查开关，确认调用过 `backward` |
| 梯度越来越大 | 忘记 `zero_grad` | 每步更新前清零 |
| 第二次反向报错 | 计算图已释放 | 需要时 `retain_graph=True` |
| 非标量反向报错 | 输出不是标量 | 传 `gradient` 参数或先归约 |
| 修改叶子张量报错 | 对 `requires_grad` 的叶子做原地操作 | 用 `detach()` 或新建张量 |
| 评估时显存不降 | 忘记 `no_grad` | 包住评估循环 |
| 损失反传但参数不动 | 参数被冻结或优化器没包含 | 检查 `requires_grad` 与参数组 |

# 6. 练习

1. 对 `f(x) = x^3 + 2x` 在 `x = 3` 处求导，先用纸笔算出结果，再用 PyTorch 验证。
2. 构造两个损失 `loss_a` 和 `loss_b`，分别反向并观察 `x.grad` 的累加；然后插入 `zero_grad` 再跑一次，对比结果。
3. 模拟一个目标网络：用 `detach()` 生成不参与梯度的目标值，计算 MSE 并反向，确认只有在线网络的参数拿到梯度。

```python
# 练习 1 的起点
x = torch.tensor(3.0, requires_grad=True)
f = x ** 3 + 2 * x
f.backward()
print(x.grad)  # 应该是 29
```

# 7. 小结

| 主题 | 要点 |
|------|------|
| 计算图 | 前向时动态构建，反向按相反顺序执行 |
| 叶子节点 | 用户创建，默认保存 `.grad`；中间结果需 `retain_grad` |
| backward | 标量可直接调用，向量需传上游梯度 |
| 累积 | `.grad` 默认相加，训练每步要 `zero_grad` |
| detach | 切断梯度但共享存储，彻底独立要 `clone` |
| 关闭求导 | 评估用 `no_grad`，纯推理用 `inference_mode` |
| 多次反向 | 默认释放图，必要时 `retain_graph=True` |

下一篇进入 `nn.Module`，讲清模型如何组织参数、`forward` 与 `__call__` 的关系、子模块的注册方式，以及权重初始化的正确位置。
