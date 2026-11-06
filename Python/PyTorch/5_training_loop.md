训练循环是把前面几篇的东西串起来的地方：模型、数据、损失、优化器、指标、保存。看起来只是几行代码，但顺序错一步，结果就可能完全不对。本篇给出一份可以直接运行的模板，并解释每个位置为什么这么放。

# 1. 整体结构

```text
for epoch in range(max_epochs):
    │
    ├─ 训练阶段
    │    model.train()
    │    for batch in train_loader:
    │        zero_grad → forward → loss → backward → step
    │
    ├─ 验证阶段
    │    model.eval()
    │    with no_grad:
    │        for batch in val_loader:
    │            forward → loss → 累计指标
    │
    └─ 收尾
         记录日志 → 调整学习率 → 早停判断 → 保存最佳模型
```

★ **关键认知**：训练和验证的区别不只是"要不要反向"，还包括**模式切换**和**是否建图**。训练要 `model.train()` 且保留梯度；验证要 `model.eval()` 且关闭梯度。三件事少一件，指标就不可信或显存会爆。

# 2. 训练阶段

## 2.1 模式与梯度清零

```python
model.train()                       # Dropout 生效，BatchNorm 用 batch 统计
optimizer.zero_grad(set_to_none=True)  # 清掉上一步残留的梯度
```

`zero_grad` 放在 batch 循环内部、`backward` 之前。放在循环外只清一次，梯度会跨 batch 累积。

## 2.2 四个步骤

```python
for x, y in train_loader:
    x = x.to(device, non_blocking=True)
    y = y.to(device, non_blocking=True)

    optimizer.zero_grad(set_to_none=True)  # 1. 清零
    logits = model(x)                      # 2. 前向
    loss = criterion(logits, y)            # 3. 算损失
    loss.backward()                        # 4. 反向
    optimizer.step()                       # 5. 更新参数
```

顺序不能乱：`backward` 之前必须已经算过 loss；`step` 必须在 `backward` 之后；下一步开始前必须清零。

⚠ 不要在 `loss.backward()` 和 `optimizer.step()` 之间插入会修改参数的代码。如果要做梯度裁剪，放在两者之间；如果要累积梯度，则推迟 `step()`。

## 2.3 指标累计

不能直接对每个 batch 的 loss 求平均，因为最后一个 batch 可能更小。正确做法是按样本数加权：

```python
total_loss = 0.0
total_correct = 0
total_samples = 0

for x, y in train_loader:
    ...
    batch_size = x.size(0)
    total_loss += loss.item() * batch_size       # item 转成 Python 数字，脱离计算图
    pred = logits.argmax(dim=1)
    total_correct += (pred == y).sum().item()
    total_samples += batch_size

train_loss = total_loss / total_samples
train_acc = total_correct / total_samples
```

★ **关键认知**：累计指标时一定要用 `.item()` 或 `.detach()`。把带 `grad_fn` 的张量一直加进列表，会让整张计算图无法释放，几个 epoch 后显存就被"历史指标"吃光。

完整的训练函数：

```python
def train_one_epoch(model, loader, optimizer, criterion, device):
    model.train()
    total_loss = 0.0
    total_correct = 0
    total_samples = 0

    for x, y in loader:
        x = x.to(device, non_blocking=True)
        y = y.to(device, non_blocking=True)

        optimizer.zero_grad(set_to_none=True)
        logits = model(x)
        loss = criterion(logits, y)
        loss.backward()
        optimizer.step()

        total_loss += loss.item() * x.size(0)
        total_correct += (logits.argmax(dim=1) == y).sum().item()
        total_samples += x.size(0)

    return total_loss / total_samples, total_correct / total_samples
```

# 3. 验证阶段

## 3.1 切换模式与关闭梯度

```python
@torch.no_grad()
def evaluate(model, loader, criterion, device):
    model.eval()
    total_loss = 0.0
    total_correct = 0
    total_samples = 0

    for x, y in loader:
        x = x.to(device, non_blocking=True)
        y = y.to(device, non_blocking=True)

        logits = model(x)
        loss = criterion(logits, y)

        total_loss += loss.item() * x.size(0)
        total_correct += (logits.argmax(dim=1) == y).sum().item()
        total_samples += x.size(0)

    return total_loss / total_samples, total_correct / total_samples
```

`@torch.no_grad()` 是装饰器写法，等价于把整个函数体包进 `with torch.no_grad():`。验证时不需要反向，关掉建图能显著降低显存。

★ **关键认知**：`model.eval()` 和 `torch.no_grad()` 是两件独立的事。前者改层的行为（Dropout、BatchNorm），后者改自动求导。评估时两个都要，只做其中一个都是错的。

## 3.2 验证集不参与调参

验证集用于选模型和早停，测试集只在最后看一次。如果反复用测试集调超参数，测试集就变成了验证集，最终指标会偏乐观。数据划分应该是：

```text
训练集：更新参数
验证集：选 epoch、调超参、早停
测试集：最后一次评估，只报告不调整
```

# 4. 早停与最佳模型

## 4.1 保存最佳权重

每个 epoch 结束后比较验证指标，只保留最好的：

```python
best_acc = 0.0
best_state = None

for epoch in range(max_epochs):
    train_loss, train_acc = train_one_epoch(model, train_loader, optimizer, criterion, device)
    val_loss, val_acc = evaluate(model, val_loader, criterion, device)

    if val_acc > best_acc:
        best_acc = val_acc
        best_state = {k: v.detach().cpu().clone() for k, v in model.state_dict().items()}
        print(f"epoch {epoch}: 保存最佳模型，val_acc={val_acc:.4f}")

model.load_state_dict(best_state)   # 训练结束后恢复最佳权重
```

实际项目更推荐直接存文件，而不是把所有张量留在内存里。保存与恢复的细节放到第 8 篇。

## 4.2 早停实现

验证指标连续若干轮没有提升就停止：

```python
class EarlyStopping:
    def __init__(self, patience=5, min_delta=0.0):
        self.patience = patience
        self.min_delta = min_delta
        self.best = None
        self.counter = 0

    def step(self, score):
        # 返回 True 表示当前是最好的，需要保存
        if self.best is None or score > self.best + self.min_delta:
            self.best = score
            self.counter = 0
            return True
        self.counter += 1
        return False

    @property
    def should_stop(self):
        return self.counter >= self.patience
```

用法：

```python
stopper = EarlyStopping(patience=3)

for epoch in range(max_epochs):
    train_loss, train_acc = train_one_epoch(model, train_loader, optimizer, criterion, device)
    val_loss, val_acc = evaluate(model, val_loader, criterion, device)

    improved = stopper.step(val_acc)
    print(f"epoch {epoch:02d} train_loss={train_loss:.4f} "
          f"val_loss={val_loss:.4f} val_acc={val_acc:.4f}")

    if improved:
        torch.save(model.state_dict(), "best.pt")

    if stopper.should_stop:
        print(f"早停于 epoch {epoch}")
        break
```

★ **关键认知**：早停监控的应该是**验证指标**，不是训练 loss。训练 loss 几乎总在下降，用它做早停等于永远不会停；只有验证指标才反映泛化能力。

⚠ 早停的 `patience` 要结合学习率策略。如果用了 `ReduceLROnPlateau`，学习率刚降下来时指标可能短暂停滞，耐心太小会过早停止。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| loss 不下降 | 忘记 `zero_grad` 或学习率过大 | 检查顺序，先降学习率 |
| 训练指标好、验证差 | 过拟合或数据泄漏 | 加正则、检查划分 |
| 验证结果每次不同 | 忘记 `model.eval()` | 评估前切换模式 |
| 显存随 epoch 增长 | 指标张量没 `.item()` | 累计前转成标量 |
| 最后一个 batch 报错 | BatchNorm 遇到单样本 | `drop_last=True` |
| loss 变成 NaN | 学习率过大、除零、标签越界 | 降学习率，检查数据和损失 |
| 验证 loss 远高于训练 | Dropout/BatchNorm 没切对，或分布不同 | 确认 `eval()` 与数据一致性 |
| 早停后指标反而差 | 没恢复最佳权重 | 保存并在最后 `load_state_dict` |

# 6. 练习

1. 用 `TensorDataset` 造一份二分类数据，跑通上面的训练函数，打印每个 epoch 的 loss 和 accuracy。
2. 给训练循环加上早停，`patience=3`，观察在第几个 epoch 停止，并确认最终加载的是最佳权重。
3. 把批量大小从 32 改成 8，观察训练 loss 的波动和每个 epoch 的耗时，解释差异。

```python
# 练习 1 的数据起点
from torch.utils.data import TensorDataset, DataLoader

X = torch.randn(1000, 20)
y = torch.randint(0, 2, (1000,))
train_loader = DataLoader(TensorDataset(X, y), batch_size=32, shuffle=True)

model = torch.nn.Sequential(
    torch.nn.Linear(20, 64),
    torch.nn.ReLU(),
    torch.nn.Linear(64, 2),
)
criterion = torch.nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
```

# 7. 小结

| 主题 | 要点 |
|------|------|
| 顺序 | `zero_grad → forward → loss → backward → step` |
| 模式 | 训练 `train()`，验证 `eval()` |
| 梯度 | 验证用 `no_grad`，训练保留图 |
| 指标 | 按样本数加权，累计前 `.item()` |
| 早停 | 监控验证指标，连续未提升则停 |
| 最佳模型 | 保存最佳权重，训练结束再恢复 |
| 数据划分 | 训练调参、验证选模、测试只报告 |

下一篇进入优化器与学习率调度，讲清 SGD 与 Adam 的差别、weight decay 的正确用法、常见 scheduler 的调用顺序，以及梯度裁剪。
