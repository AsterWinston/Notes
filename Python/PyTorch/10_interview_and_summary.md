学完前面九篇，你已经有能力写出一份能跑的 PyTorch 训练代码。但"能跑"和"结果可信"之间还差一层：会排错、能识别过拟合和数据泄漏、知道每个 API 背后的边界。本篇用排错框架把这些串起来，并给出一份高频自检题。

# 1. 排错框架

## 1.1 按阶段定位

训练出问题，不要一上来就改模型。按数据、模型、损失、优化器、环境五层往下查，每层只回答一个问题：

```text
loss 不下降
  │
  ├─ 数据对不对？  形状、标签、归一化、重复样本
  ├─ 模型通不通？  forward 形状、参数是否 requires_grad
  ├─ 损失配不配？  任务与 criterion 是否匹配
  ├─ 优化器转不转？参数是否传进去、lr 是否合理、是否清零
  └─ 环境一致吗？  device、dtype、AMP、随机种子、版本
```

★ **关键认知**：排错的第一原则是**缩小范围**，不是"多试几个超参"。先让模型在一个极小数据集上过拟合，能过拟合说明模型和训练循环基本正确，问题在数据规模或泛化；过拟合不了，说明管道里有硬 bug。

## 1.2 从曲线读问题

```text
loss
 │\
 │ \        train
 │  \____
 │       \  val
 │        \____
 └────────────── epoch
```

| 曲线形态 | 判断 | 优先动作 |
|----------|------|----------|
| train 降、val 降后回升 | 过拟合 | 早停、正则、加数据 |
| train 和 val 都高且平 | 欠拟合或 bug | 检查数据与标签，增大模型或 lr |
| train 降、val 剧烈震荡 | 验证集太小或分布差异 | 增大验证集，检查预处理 |
| loss 突然 NaN | 梯度爆炸、lr 过大 | 降 lr，加裁剪，查除零 |
| loss 完全不动 | 参数没更新、lr 为 0 | 检查优化器参数与 `zero_grad` |
| 指标不升但 loss 降 | 指标与损失不匹配 | 检查评估逻辑与类别不平衡 |

## 1.3 最小复现

固定随机种子，用 8 到 32 条样本训练几十步：

```python
import torch

torch.manual_seed(42)
torch.cuda.manual_seed_all(42)

# 只取一个小 batch 反复训练
x, y = next(iter(train_loader))
x, y = x.to(device), y.to(device)

for step in range(50):
    optimizer.zero_grad(set_to_none=True)
    logits = model(x)
    loss = criterion(logits, y)
    loss.backward()
    optimizer.step()
    if step % 10 == 0:
        print(step, loss.item())
```

如果 loss 不能降到接近 0，问题一定在数据、模型或训练顺序，而不是"数据不够"。这个测试能在几分钟内排除大部分低级错误。

⚠ 小 batch 过拟合测试要关掉 Dropout 或接受它的噪声，并且用 `model.train()`。如果模型里有 BatchNorm，batch 太小会导致统计量不稳定，可以临时换 LayerNorm 或增大到 32 条。

# 2. 过拟合与欠拟合

## 2.1 怎么判断

| 现象 | 过拟合 | 欠拟合 |
|------|--------|--------|
| 训练 loss | 很低 | 高 |
| 验证 loss | 明显更高，甚至回升 | 与训练接近但都高 |
| 训练准确率 | 很高 | 低 |
| 验证准确率 | 低或先升后降 | 低 |
| 数据量 | 相对模型偏少 | 相对模型偏多或特征不足 |

## 2.2 应对手段

| 手段 | 作用 | 注意 |
|------|------|------|
| 增加数据 | 最根本 | 成本最高 |
| 数据增强 | 扩大有效样本 | 验证集不能增强 |
| weight decay | 抑制大权重 | 偏置和归一化层通常不衰减 |
| Dropout | 随机失活 | 评估时必须 `eval()` |
| 早停 | 在验证指标拐点停 | 需要可靠的验证集 |
| 减小模型 | 降低容量 | 可能转为欠拟合 |
| 标签平滑 | 缓解过度自信 | 与交叉熵配合 |
| 冻结主干 | 减少可训练参数 | 微调场景常用 |

★ **关键认知**：正则化不是越多越好。所有正则手段都在"降低方差"的同时抬高偏差；验证 loss 不降反升时，说明约束过头了。调参的方向应该由验证曲线决定，而不是把所有技巧都加上。

# 3. 数据泄漏

## 3.1 常见来源

数据泄漏指训练阶段看到了本不该看到的信息，导致离线指标虚高、线上崩盘：

| 泄漏 | 例子 | 为什么错 |
|------|------|----------|
| 预处理泄漏 | 在全量数据上算均值方差 | 验证集统计量进了训练 |
| 划分泄漏 | 同一用户/病人的样本分到两边 | 模型记住了实体 |
| 重复样本 | 训练集和验证集有完全相同的图 | 等于把答案给模型 |
| 时间泄漏 | 随机切分时间序列 | 用未来预测过去 |
| 目标泄漏 | 特征由标签计算而来 | 特征里直接含答案 |
| 调参泄漏 | 用测试集早停、选模型 | 测试集变成了验证集 |
| 重采样泄漏 | 先 SMOTE 再划分 | 合成样本跨越训练与验证 |

## 3.2 排查清单

```text
1. 划分是否在预处理之前？
2. 标准化、编码、特征选择是否只在训练集上 fit？
3. 训练集和验证集有没有重复或近重复样本？
4. 时间序列是否按时间切分？
5. 分组数据是否按组切分（GroupKFold）？
6. 每个特征在预测时刻是否真的可得？
7. 测试集是否只被使用了一次？
```

★ **关键认知**：数据泄漏的典型信号是"指标好得不真实"。一个二分类任务突然到 99.9%，先不要庆祝，先查标签是不是漏进了特征、验证集是不是混进了训练集。

# 4. 高频自检题

## 4.1 概念题

**问：为什么每步要 `optimizer.zero_grad()`？**

`backward()` 把梯度累加到 `.grad`，不清零会跨 batch 叠加，等效学习率越来越大。梯度累积是主动利用这个特性，正常训练必须每步清零。

**问：`model.eval()` 和 `torch.no_grad()` 有什么区别？**

前者切换层的行为，关闭 Dropout、让 BatchNorm 用滑动统计量；后者关闭自动求导。评估时两者都要。

**问：`view` 和 `reshape` 的区别？**

`view` 要求内存连续，只改解释方式；`reshape` 在不连续时自动复制。不确定时用 `reshape`，确定连续且想零拷贝时用 `view`。

**问：`Adam` 和 `AdamW` 的区别？**

Adam 把 weight decay 混进梯度，会被自适应学习率缩放；AdamW 把衰减解耦，直接作用在参数上，正则效果更符合预期。

**问：为什么 AMP 需要 `GradScaler`？**

float16 的动态范围小，小梯度会下溢成 0。scaler 先放大 loss，反向后再还原，避免梯度丢失。bfloat16 范围大，通常不需要。

**问：`num_workers` 越大越好吗？**

不是。进程创建、内存拷贝和上下文切换都有成本。读取是瓶颈时增加 worker 才有收益，加到吞吐不再提升就停。

**问：`DataParallel` 和 DDP 怎么选？**

正式训练用 DDP。DataParallel 是单进程多线程，受 GIL 限制、主卡压力大，只适合快速调试。

**问：保存整个模型和保存 `state_dict` 哪个好？**

`state_dict`。它只含张量，不依赖类定义，跨版本和跨项目更稳定。整个模型用 pickle 序列化了代码路径，重构后容易失效。

## 4.2 代码题

下面这段训练循环有三处问题，先自己找：

```python
model.train()
for x, y in train_loader:
    x, y = x.to(device), y.to(device)

    logits = model(x)
    loss = criterion(logits, y)
    loss.backward()
    optimizer.step()

    total_loss += loss
    correct += (logits.argmax(dim=1) == y).sum()
```

问题与修正：

| 问题 | 后果 | 修正 |
|------|------|------|
| 没有 `zero_grad` | 梯度跨 batch 累积 | 更新前 `optimizer.zero_grad(set_to_none=True)` |
| 累计 `loss` 未 `.item()` | 计算图无法释放，显存上涨 | `total_loss += loss.item() * x.size(0)` |
| 指标没有按样本数加权 | 尾批较小时均值偏差 | 同时累计 `total_samples` 再相除 |

修正后：

```python
model.train()
total_loss = 0.0
total_correct = 0
total_samples = 0

for x, y in train_loader:
    x, y = x.to(device), y.to(device)

    optimizer.zero_grad(set_to_none=True)
    logits = model(x)
    loss = criterion(logits, y)
    loss.backward()
    optimizer.step()

    total_loss += loss.item() * x.size(0)
    total_correct += (logits.argmax(dim=1) == y).sum().item()
    total_samples += x.size(0)

print(total_loss / total_samples, total_correct / total_samples)
```

## 4.3 场景题

**场景一：loss 完全不降，怎么办？**

先跑小 batch 过拟合测试。能过拟合就查数据规模和泛化；不能过拟合就依次查标签是否正确、模型输出维度、损失函数是否匹配、优化器是否拿到参数、学习率是否为 0。

**场景二：训练正常，验证指标每次不一样？**

检查是否忘记 `model.eval()`，验证集是否 `shuffle=True`，BatchNorm 是否在小 batch 上不稳定，以及指标是否用了随机增强后的数据。

**场景三：多卡训练后指标对不上？**

每个进程只看到自己的一部分数据，指标需要跨进程 all-reduce；确认用了 `DistributedSampler` 并调用了 `set_epoch`，学习率按全局 batch 调整过。

**场景四：GPU 显存不够？**

按顺序尝试：减小 batch、开 AMP、清理指标历史中的计算图、减少 worker 预取、用梯度累积、最后才考虑梯度检查点或模型并行。

# 5. 训练排错速查表

| 现象 | 先查 | 再查 |
|------|------|------|
| loss 不降 | 小 batch 能否过拟合 | 标签、损失、优化器参数 |
| loss 变 NaN | 学习率、梯度范数 | 除零、log(0)、AMP 溢出 |
| 显存上涨 | 指标是否 `.item()` | 是否持有张量、worker 预取 |
| 验证差 | 过拟合、数据泄漏 | 预处理一致性、划分方式 |
| 速度慢 | 数据管道、`num_workers` | 同步点、小 batch、CPU 瓶颈 |
| 多卡不加速 | 是否用 DDP 与 DistributedSampler | 通信、每卡 batch |
| 恢复训练异常 | 优化器与调度器状态 | epoch、scaler、设备 |

# 6. 练习

1. 找一段有 bug 的训练代码（可以自己写），用本篇的清单逐项检查，记录每个问题的现象和修正。
2. 取 16 条样本做小 batch 过拟合测试，要求 loss 降到接近 0；如果做不到，按五层框架定位原因。
3. 审查一个你熟悉的数据集划分，列出至少三种可能的泄漏，并写出对应的修正方案。

```python
# 练习 2 的检查点
assert model.training is True
assert optimizer.param_groups[0]["lr"] > 0
assert next(model.parameters()).requires_grad is True
print("基础检查通过，开始过拟合小 batch")
```

# 7. 小结

## 7.1 全模块回顾

| 篇目 | 核心问题 | 一句话答案 |
|------|----------|------------|
| 1 | Tensor 是什么 | 带设备与 dtype 的多维数组，支持广播和索引 |
| 2 | 梯度怎么来 | 前向建图，反向按链式法则累积 |
| 3 | 模型怎么组织 | `nn.Module` 自动注册参数与子模块 |
| 4 | 数据怎么喂 | Dataset 取样本，DataLoader 负责调度与拼批 |
| 5 | 循环怎么写 | 清零、前向、损失、反向、更新，验证关梯度 |
| 6 | 参数怎么更新 | 优化器定方向，scheduler 调步长，裁剪防爆炸 |
| 7 | 怎么用 GPU | 统一 device，AMP 省显存，DDP 做多卡 |
| 8 | 怎么保存 | 存 state_dict 或完整 checkpoint，安全加载 |
| 9 | 怎么交付 | TensorBoard 观察，TorchScript/ONNX 导出 |
| 10 | 怎么排错 | 缩小范围、看曲线、查泄漏、做小 batch 验证 |

## 7.2 继续深入的方向

```text
基础已覆盖
  │
  ├─ 视觉：torchvision、数据增强、检测与分割
  ├─ 序列：RNN、Transformer、注意力实现
  ├─ 大模型：FSDP、DeepSpeed、混合并行
  ├─ 性能：profiler、torch.compile、算子融合
  └─ 部署：Triton、ONNX Runtime、量化与蒸馏
```

★ **关键认知**：PyTorch 的 API 很多，但训练的主干始终是"数据 → 模型 → 损失 → 反向 → 更新 → 评估"。遇到新任务时，先把它映射回这条主干，再决定需要补哪一块，而不是从框架文档第一页开始背。

⚠ 不要在没有验证集和固定种子的情况下比较两个模型。随机初始化、数据顺序、Dropout 都会带来波动，单次运行的差异很可能只是噪声。

到这里，PyTorch 模块的十篇笔记就结束了。下一步建议选一个真实数据集，从数据管道开始独立写一遍完整训练脚本，再逐步加入 AMP、DDP 和导出，把纸面上的顺序变成手上的肌肉记忆。
