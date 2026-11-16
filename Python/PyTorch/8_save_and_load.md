训练几小时后发现忘了保存，或者恢复训练后 loss 从零开始，通常不是模型的问题，而是保存的内容不完整。`torch.save` 本身很简单，难的是想清楚"要存什么、怎么安全地读回来"。

# 1. 保存什么

## 1.1 三种常见做法

| 做法 | 保存内容 | 优点 | 风险 |
|------|----------|------|------|
| 保存整个模型 | 结构 + 权重 | 一行搞定 | 依赖类定义，跨版本易坏 |
| 保存 `state_dict` | 只有张量 | 稳定、可迁移 | 需要自己重建模型 |
| 保存 checkpoint | 权重 + 优化器 + 进度 | 可完整恢复训练 | 文件更大 |

★ **关键认知**：正式项目应保存 **`state_dict` 或 checkpoint**，不要 `torch.save(model)` 保存整个对象。整个模型用 pickle 序列化了类路径和结构，代码一重构、目录一改，旧文件就可能加载失败。

## 1.2 state_dict 里有什么

```python
import torch
from torch import nn

model = nn.Sequential(
    nn.Linear(8, 16),
    nn.ReLU(),
    nn.Linear(16, 2),
)

state = model.state_dict()
for key, value in state.items():
    print(key, tuple(value.shape))
```

输出类似：

```text
0.weight (16, 8)
0.bias (16,)
2.weight (2, 16)
2.bias (2,)
```

它就是一个有序字典，键是参数名，值是张量。`register_buffer` 注册的缓冲也会出现在里面，所以 BatchNorm 的 `running_mean` 能一起保存。

⚠ `state_dict` 里的张量仍然指向模型参数的内存。要长期保存或做快照，先 `clone()` 再存，否则后续训练会改掉你以为已经保存的值。

# 2. 保存与加载权重

## 2.1 最小流程

```python
# 保存
torch.save(model.state_dict(), "model.pt")

# 加载：必须先构造同样的结构
model = nn.Sequential(
    nn.Linear(8, 16),
    nn.ReLU(),
    nn.Linear(16, 2),
)
state = torch.load("model.pt", map_location="cpu", weights_only=True)
model.load_state_dict(state)
model.eval()
```

`load_state_dict` 默认 `strict=True`，键必须完全一致。做部分加载时：

```python
result = model.load_state_dict(state, strict=False)
print(result.missing_keys)     # 模型有、文件没有
print(result.unexpected_keys)  # 文件有、模型没有
```

迁移学习里经常用 `strict=False` 跳过分类头，但要确认跳过的确实是预期的那几层，而不是整段主干都没加载上。

★ **关键认知**：加载权重前，模型结构必须与保存时**逐层一致**。`state_dict` 只存数值，不存结构信息；形状对不上会直接报 size mismatch，键对不上则取决于 `strict`。

## 2.2 设备与半精度

```python
# 在有 GPU 的机器上加载 CPU 保存的文件
state = torch.load("model.pt", map_location="cuda:0", weights_only=True)

# 没有 GPU 时加载 GPU 保存的文件
state = torch.load("model.pt", map_location="cpu", weights_only=True)
```

`map_location` 决定张量落到哪个设备。不指定时，会按保存时的设备恢复，在无 GPU 的机器上直接报错。

如果保存的是 half 精度权重，而模型是 float32，加载后可以转换：

```python
model.load_state_dict(state)
model = model.float()
```

# 3. checkpoint

## 3.1 保存完整训练状态

```python
checkpoint = {
    "epoch": epoch,
    "model": model.state_dict(),
    "optimizer": optimizer.state_dict(),
    "scheduler": scheduler.state_dict(),
    "best_acc": best_acc,
    "config": {"lr": 3e-4, "batch_size": 32},
}

if scaler is not None:                    # 用了 AMP 才保存
    checkpoint["scaler"] = scaler.state_dict()

torch.save(checkpoint, f"ckpt_epoch{epoch}.pt")
```

结构图：

```text
checkpoint.pt
├── epoch           训练到第几轮
├── model           模型参数
├── optimizer       动量、Adam 一阶二阶矩
├── scheduler       学习率进度
├── scaler          AMP 缩放系数
├── best_acc        早停与最佳模型判断
└── config          超参，方便复现
```

## 3.2 恢复训练

```python
def load_checkpoint(path, model, optimizer, scheduler, device, scaler=None):
    ckpt = torch.load(path, map_location=device, weights_only=True)

    model.load_state_dict(ckpt["model"])
    optimizer.load_state_dict(ckpt["optimizer"])
    scheduler.load_state_dict(ckpt["scheduler"])

    if scaler is not None and "scaler" in ckpt:
        scaler.load_state_dict(ckpt["scaler"])

    start_epoch = ckpt["epoch"] + 1
    best_acc = ckpt.get("best_acc", 0.0)
    return start_epoch, best_acc
```

使用：

```python
start_epoch, best_acc = load_checkpoint("ckpt_epoch10.pt", model, optimizer, scheduler, device)

for epoch in range(start_epoch, max_epochs):
    ...
```

★ **关键认知**：只恢复模型权重，优化器的动量和 Adam 状态会丢失，学习率调度也会从头开始，表现为"恢复后 loss 突然跳一下"。要无缝续训，必须把 optimizer、scheduler、scaler 和 epoch 一起保存。

⚠ 恢复时先 `model.to(device)` 再 `load_state_dict`，并确认优化器状态也在目标设备上。跨设备恢复后用一个小 batch 试跑一步，检查参数设备和 `optimizer.state` 里的张量是否一致。

## 3.3 保存频率与保留策略

| 策略 | 适用 | 说明 |
|------|------|------|
| 只存最佳 | 最终交付 | 文件小，无法续训 |
| 每 N 个 epoch | 长训练 | 兼顾恢复与磁盘 |
| 滚动保留最近 K 个 | 长期任务 | 防止磁盘写满 |
| 最佳 + 最新 | 最常用 | 兼顾交付与恢复 |

# 4. 安全与兼容

## 4.1 weights_only

`torch.load` 底层用 pickle，恶意文件可以在反序列化时执行任意代码。PyTorch 2.6 起 `weights_only` 默认是 `True`，只允许加载张量和基础容器；更早的版本默认是 `False`。

```python
# 只含权重、优化器状态等基础对象：安全模式
state = torch.load("model.pt", map_location="cpu", weights_only=True)

# 含自定义类、numpy 数组等：确认文件来源可信才关闭
ckpt = torch.load("trusted.pt", map_location="cpu", weights_only=False)
```

★ **关键认知**：`weights_only=True` 不是"只能加载权重"，而是"只允许安全类型"，普通 checkpoint 里的 dict、int、list、tensor 都能通过。只有确实包含自定义对象时才需要 `False`，并且必须确保文件来源可信。

⚠ 不要把用户上传的文件直接丢给 `torch.load(..., weights_only=False)`。推理服务加载外部模型时优先用 safetensors 这类纯张量格式，或者先做格式校验。

## 4.2 版本兼容

| 问题 | 原因 | 处理 |
|------|------|------|
| 类路径找不到 | 保存了整个模型 | 改存 `state_dict` |
| 键名带 `module.` | 保存了 DDP 包装后的模型 | 用 `model.module.state_dict()` 或加载时去掉前缀 |
| 形状不匹配 | 模型结构改了 | 用 `strict=False` 并手动核对 |
| 新版本打不开旧文件 | pickle 协议或类定义变化 | 尽量统一版本，或在旧环境转存 |
| dtype 不一致 | 保存了 half 权重 | 加载后 `.float()` 或 `.half()` |

DDP 保存的推荐写法：

```python
state = model.module.state_dict() if hasattr(model, "module") else model.state_dict()
torch.save(state, "model.pt")
```

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `Missing key(s) in state_dict` | 模型结构或键名不一致 | 打印两边键名对比 |
| `size mismatch` | 层维度改了 | 检查模型定义与旧权重 |
| 加载后预测全是同一类 | 忘记 `eval()` 或没加载成功 | 切模式，检查 `missing_keys` |
| 无 GPU 机器加载失败 | 没设 `map_location` | 加 `map_location="cpu"` |
| 续训 loss 跳变 | 没恢复优化器和调度器 | 保存完整 checkpoint |
| 文件很大 | 存了整个优化器 + 多个副本 | 只存必要字段，定期清理 |
| 加载报权限或反序列化错误 | `weights_only` 与内容不匹配 | 确认来源，按需设置 |

# 6. 练习

1. 训练一个三分类小模型，分别保存 `state_dict` 和完整 checkpoint，比较文件大小与加载方式。
2. 训练 3 个 epoch 后保存 checkpoint，再新起一个脚本恢复，确认 `start_epoch`、学习率和优化器动量都正确。
3. 故意修改模型的一层维度，用 `strict=False` 加载，打印 `missing_keys` 和 `unexpected_keys`，解释发生了什么。

```python
# 练习 2 的检查点
ckpt = torch.load("ckpt.pt", map_location="cpu", weights_only=True)
print(ckpt.keys())
print(ckpt["epoch"], ckpt["optimizer"]["param_groups"][0]["lr"])
```

# 7. 小结

| 主题 | 要点 |
|------|------|
| 保存对象 | 优先 `state_dict` 或 checkpoint，不存整个模型 |
| 权重 | 结构与键名必须一致，`strict=False` 只用于有意跳过 |
| checkpoint | model + optimizer + scheduler + scaler + epoch |
| 设备 | `map_location` 控制落点，恢复前先搬模型 |
| 续训 | 优化器状态和调度进度都不能丢 |
| 安全 | `weights_only=True` 优先，外部文件用 safetensors |
| 兼容 | DDP 保存 `module.state_dict()`，版本尽量统一 |

下一篇进入可视化与部署，讲清 TensorBoard 怎么记录指标、TorchScript 与 ONNX 如何导出，以及推理服务的边界在哪里。
