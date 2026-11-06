优化器决定"参数往哪走、走多大步"，学习率调度决定"步子随时间怎么变"。模型不收敛，八成不是网络结构的问题，而是这两者的配置或调用顺序有问题。本篇把 SGD、Adam、weight decay、scheduler 和梯度裁剪串起来讲。

# 1. 优化器在做什么

训练的目标是最小化损失 `L(θ)`。优化器拿到每个参数的梯度 `g = ∂L/∂θ`，按某种规则更新：

```text
θ ← θ - lr × 更新量(g)
```

不同优化器的区别就在"更新量"怎么算：SGD 直接用梯度，Momentum 叠加历史方向，Adam 同时估计梯度的一阶矩和二阶矩。

# 2. SGD 与 Adam

## 2.1 SGD 与 Momentum

```python
import torch

optimizer = torch.optim.SGD(
    model.parameters(),
    lr=0.1,              # 学习率
    momentum=0.9,        # 动量，缓解震荡、加速收敛
    weight_decay=1e-4,   # L2 正则
    nesterov=True,       # Nesterov 动量
)
```

Momentum 维护一个速度项：

```text
v ← momentum × v + g
θ ← θ - lr × v
```

它在方向一致的维度上加速，在来回震荡的维度上抵消。`nesterov=True` 会先用当前速度预估下一步位置，再在那个位置算梯度，通常比普通动量略好。

## 2.2 Adam 与 AdamW

```python
optimizer = torch.optim.Adam(
    model.parameters(),
    lr=1e-3,
    betas=(0.9, 0.999),  # 一阶、二阶矩的衰减率
    eps=1e-8,            # 防止除零
)

# 解耦权重衰减，Transformer 类模型更常用
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=3e-4,
    weight_decay=0.01,
)
```

Adam 对每个参数自适应调整步长，前期收敛快、对学习率不敏感，适合稀疏梯度和大模型。AdamW 把 weight decay 从梯度里拆出来，直接作用在参数上，正则效果更符合预期。

| 优化器 | 优点 | 缺点 | 适用 |
|--------|------|------|------|
| SGD | 泛化好、显存省 | 对学习率敏感、收敛慢 | CNN、需要精细调参 |
| SGD + Momentum | 收敛更稳 | 多一个超参 | 传统视觉任务 |
| Adam | 自适应、上手快 | 泛化有时略差 | 快速实验、稀疏特征 |
| AdamW | 正则解耦、稳定 | 显存略高 | Transformer、大模型 |

★ **关键认知**：优化器没有绝对优劣。SGD 在图像分类上常能调到更好的最终精度，AdamW 在 NLP 和大模型上更省心。先用 AdamW 快速验证流程能跑通，再根据任务决定是否换 SGD 精调。

## 2.3 学习率是第一超参

学习率过大，loss 震荡甚至发散；过小，收敛慢且容易停在平台。经验起点：

| 优化器 | 常见起点 |
|--------|----------|
| SGD | 0.1、0.01 |
| Adam / AdamW | 1e-3、3e-4、1e-4 |

先用小数据跑几十步，观察 loss 是否能稳定下降，再上全量数据。

# 3. 参数组与 weight decay

## 3.1 给不同层不同学习率

`optimizer` 接受参数组列表，每组可以有自己的超参：

```python
optimizer = torch.optim.AdamW([
    {"params": model.backbone.parameters(), "lr": 1e-5, "weight_decay": 0.01},
    {"params": model.head.parameters(), "lr": 1e-3, "weight_decay": 0.01},
])
```

微调预训练模型时很常用：主干用小学习率，新加的分类头用大学习率。

## 3.2 偏置和归一化层不衰减

weight decay 的本意是抑制大权重，偏置和 LayerNorm/BatchNorm 的缩放参数通常不需要：

```python
decay, no_decay = [], []
for name, param in model.named_parameters():
    if not param.requires_grad:
        continue
    if param.ndim < 2 or "norm" in name or "bias" in name:
        no_decay.append(param)
    else:
        decay.append(param)

optimizer = torch.optim.AdamW([
    {"params": decay, "weight_decay": 0.01},
    {"params": no_decay, "weight_decay": 0.0},
], lr=3e-4)
```

★ **关键认知**：对 SGD 来说，`weight_decay` 近似 L2 正则；对 Adam 来说，传统实现里的 L2 会被自适应学习率缩放，效果和预期不一致。要真正的解耦衰减，用 `AdamW`。

⚠ 不要对已经冻结的参数建参数组，`requires_grad=False` 的参数不会更新，但会白白占用优化器状态。

# 4. 学习率调度

## 4.1 常见 scheduler

| scheduler | 行为 | 适用 |
|-----------|------|------|
| `StepLR` | 每隔固定 epoch 乘 `gamma` | 传统训练 |
| `MultiStepLR` | 在指定 epoch 降一次 | 里程碑式训练 |
| `ExponentialLR` | 每 epoch 指数衰减 | 简单衰减 |
| `CosineAnnealingLR` | 余弦曲线降到接近 0 | 现代训练常用 |
| `CosineAnnealingWarmRestarts` | 周期性重启 | 需要跳出局部最优 |
| `OneCycleLR` | 先升后降 | 快速收敛 |
| `ReduceLROnPlateau` | 指标不降时降 | 不确定训练长度 |

```python
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer,
    T_max=50,       # 半个余弦周期，通常设为总 epoch 数
    eta_min=1e-6,   # 最小学习率
)
```

曲线大致是：

```text
lr
 │\
 │ \
 │  \
 │   \___
 │       \___
 └────────────── epoch
 0             T_max
```

## 4.2 调用顺序

PyTorch 1.1 之后，标准顺序是**先 `optimizer.step()`，再 `scheduler.step()`**：

```python
for epoch in range(epochs):
    for x, y in train_loader:
        optimizer.zero_grad()
        loss = criterion(model(x), y)
        loss.backward()
        optimizer.step()

    scheduler.step()   # 每个 epoch 结束后调整
    print(optimizer.param_groups[0]["lr"])
```

`scheduler` 修改的是 `optimizer.param_groups` 里的 `lr`，所以必须在优化器创建之后构造它。

⚠ 顺序反了会触发警告 "Detected call of `lr_scheduler.step()` before `optimizer.step()`"，并且第一个 epoch 的学习率就会被跳过。

## 4.3 按 epoch 还是按 step

| 调度器 | 建议调用频率 |
|--------|--------------|
| `StepLR`、`MultiStepLR` | 每 epoch |
| `CosineAnnealingLR` | 每 epoch |
| `OneCycleLR` | 每 batch，需给 `total_steps` |
| `ReduceLROnPlateau` | 每 epoch，传验证指标 |

按 batch 调度的例子：

```python
scheduler = torch.optim.lr_scheduler.OneCycleLR(
    optimizer,
    max_lr=1e-3,
    total_steps=epochs * len(train_loader),
)

for epoch in range(epochs):
    for x, y in train_loader:
        ...
        optimizer.step()
        scheduler.step()   # 每个 batch 都调
```

`ReduceLROnPlateau` 是例外，它需要指标：

```python
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
    optimizer, mode="min", factor=0.1, patience=5
)

# 验证之后
val_loss = evaluate(...)
scheduler.step(val_loss)
```

★ **关键认知**：`ReduceLROnPlateau` 看的是指标是否改善，不是按固定节奏衰减。配合早停时，先让它降学习率，再给几轮耐心，最后才停。

# 5. 梯度裁剪

RNN、Transformer 和强化学习里，梯度偶尔会变得很大，导致一步更新把参数打飞。裁剪把梯度范数限制在上限内：

```python
loss.backward()

torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

optimizer.step()
```

它按所有参数梯度的全局范数缩放：

```text
if ||g|| > max_norm:
    g ← g × max_norm / ||g||
```

还有按值裁剪的 `clip_grad_value_`，但不常用。绝大多数场景用 `clip_grad_norm_`，`max_norm` 从 1.0 起步。

★ **关键认知**：梯度裁剪只限制单步更新幅度，不修复错误的数据或损失。loss 持续爆炸时，先降学习率、检查标签和数值稳定性，再把裁剪当保险，而不是当解药。

⚠ 裁剪必须在 `backward()` 之后、`step()` 之前。放在 `step()` 之后，参数已经更新了，裁的是下一步不再使用的梯度。

# 6. 梯度累积

显存不够时，用多个小 batch 累积出大 batch 的效果：

```python
accum_steps = 4
optimizer.zero_grad(set_to_none=True)

for step, (x, y) in enumerate(train_loader):
    logits = model(x)
    loss = criterion(logits, y) / accum_steps   # 缩放，保持梯度量级
    loss.backward()

    if (step + 1) % accum_steps == 0:
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
```

等效 batch size 是 `batch_size × accum_steps`。注意学习率通常要按等效 batch 调整，scheduler 也要按实际 `step()` 次数推进。

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| loss 变 NaN | 学习率过大、梯度爆炸 | 降 lr，加梯度裁剪 |
| loss 完全不降 | 优化器没拿到参数、lr 太小 | 检查 `model.parameters()`，打印 lr |
| 学习率一直是初始值 | 忘记调用 `scheduler.step()` | 按频率调用 |
| 第一个 epoch 学习率就变了 | scheduler 调用早于 optimizer | 调整顺序 |
| 恢复训练后 lr 从头开始 | 没加载 scheduler 状态 | 保存并恢复 `scheduler.state_dict()` |
| weight decay 没效果 | 用了 Adam 的 L2 而非 AdamW | 换 `AdamW` |
| 裁剪后 loss 仍爆炸 | 裁剪位置错或 max_norm 太大 | 放在 step 前，降低阈值 |

# 8. 练习

1. 用同一个模型分别跑 SGD（lr=0.1, momentum=0.9）和 AdamW（lr=1e-3），比较前 10 个 epoch 的 loss 曲线。
2. 给模型的主干和分类头设置不同学习率，打印 `optimizer.param_groups` 确认生效。
3. 用 `OneCycleLR` 按 batch 调度，记录每个 step 的学习率，画出先升后降的形状。

```python
# 练习 2 的起点
optimizer = torch.optim.AdamW([
    {"params": model[0].parameters(), "lr": 1e-5},
    {"params": model[-1].parameters(), "lr": 1e-3},
], weight_decay=0.01)

for group in optimizer.param_groups:
    print(group["lr"])
```

# 9. 小结

| 主题 | 要点 |
|------|------|
| SGD | 简单、泛化好，靠 momentum 稳定方向 |
| Adam | 自适应步长，上手快；AdamW 解耦 weight decay |
| 参数组 | 不同层可设不同 lr 与衰减 |
| 衰减 | 偏置、归一化层通常不衰减 |
| 调度 | 先 `optimizer.step()` 再 `scheduler.step()` |
| 频率 | 多数按 epoch，`OneCycleLR` 按 batch，`Plateau` 传指标 |
| 裁剪 | `backward` 后、`step` 前，常用 `max_norm=1.0` |
| 累积 | 小 batch 累加等效大 batch，损失要缩放 |

下一篇进入 GPU 与混合精度，讲清设备选择、显存构成、`autocast` 与 `GradScaler` 的配合，以及多卡训练的边界。
