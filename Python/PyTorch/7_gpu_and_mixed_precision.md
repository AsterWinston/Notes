GPU 训练的三个问题几乎每个人都会遇到：设备对不上、显存不够、精度和速度怎么平衡。本篇先把 device 管理和显存构成讲清楚，再讲 AMP 的正确写法，最后划清单卡、DataParallel 和 DDP 的边界。

# 1. device 管理

## 1.1 选择设备

```python
import torch

if torch.cuda.is_available():
    device = torch.device("cuda")
elif torch.backends.mps.is_available():   # Apple Silicon
    device = torch.device("mps")
else:
    device = torch.device("cpu")

print(device)
print(torch.cuda.get_device_name(0) if device.type == "cuda" else "无 CUDA")
```

不要硬编码 `.cuda()`。跨平台代码统一用 `torch.device` 加 `.to(device)`，在没有 GPU 的机器上也能跑。

## 1.2 模型与数据同设备

```python
model = model.to(device)
x = x.to(device, non_blocking=True)
y = y.to(device, non_blocking=True)
```

`non_blocking=True` 只在 `pin_memory=True` 的 DataLoader 下有异步效果，其余情况会被忽略。

★ **关键认知**：设备不一致的报错通常出现在**第一个需要两个张量同设备的算子**上，而不是 `.to()` 那一行。看到 "Expected all tensors to be on the same device" 时，顺着前向逐层打印 `param.device` 和 `x.device`，比盯着报错行更有效。

⚠ 模型里有普通张量属性（不是 `Parameter`、不是 buffer）时，`.to(device)` 不会搬它，前向时就会设备冲突。用 `register_buffer` 注册，或者在 `forward` 里显式 `.to(x.device)`。

## 1.3 显存构成

```text
显存占用
├── 模型参数           params × 4 字节（fp32）
├── 梯度               params × 4 字节
├── 优化器状态         Adam 约 params × 8 字节（一阶、二阶矩）
├── 激活值             与 batch size、序列长度、层数成正比
├── 临时缓冲区         cuDNN workspace、通信缓冲
└── 碎片               频繁分配释放留下的空洞
```

一个 1 亿参数的模型，fp32 参数约 400 MB，梯度 400 MB，Adam 状态约 800 MB，加起来已经 1.6 GB，还没算激活。真正吃显存的往往是大 batch 下的激活值。

```python
print(torch.cuda.memory_allocated() / 1024**2, "MB")      # 当前张量占用
print(torch.cuda.max_memory_allocated() / 1024**2, "MB")  # 峰值
print(torch.cuda.memory_reserved() / 1024**2, "MB")       # 缓存池保留
```

★ **关键认知**：`torch.cuda.empty_cache()` 释放的是**缓存池里没被使用的块**，不会释放还活着的张量。OOM 时先减小 batch、清理无用引用、用混合精度，`empty_cache` 只能缓解碎片，不能解决根因。

# 2. 混合精度

## 2.1 float16 与 bfloat16

| 类型 | 位宽 | 指数位 | 尾数位 | 特点 |
|------|------|--------|--------|------|
| float32 | 32 | 8 | 23 | 基准精度 |
| float16 | 16 | 5 | 10 | 速度快，动态范围小，易下溢 |
| bfloat16 | 16 | 8 | 7 | 范围同 fp32，精度略低，训练更稳 |

float16 的尾数够用但指数范围窄，很小的梯度会变成 0；bfloat16 指数范围大，不容易下溢，但尾数少、精度粗。Ampere 及更新的 GPU 对两者都有硬件支持。

## 2.2 autocast

`autocast` 会自动把适合的算子转到低精度，把数值敏感的算子（如 softmax、LayerNorm、损失）留在 float32：

```python
with torch.amp.autocast("cuda", dtype=torch.float16):
    logits = model(x)
    loss = criterion(logits, y)
```

不需要手动把模型和输入转成 half。`autocast` 只在上下文内生效，出了 `with` 张量仍是 float32。

```python
# CPU 上也可以做 bf16 混合精度
with torch.amp.autocast("cpu", dtype=torch.bfloat16):
    logits = model(x)
```

## 2.3 GradScaler

float16 下梯度容易下溢。`GradScaler` 先把 loss 放大，反向得到放大后的梯度，更新前再缩放回去：

```python
scaler = torch.amp.GradScaler("cuda")

for x, y in train_loader:
    x = x.to(device, non_blocking=True)
    y = y.to(device, non_blocking=True)

    optimizer.zero_grad(set_to_none=True)

    with torch.amp.autocast("cuda", dtype=torch.float16):
        logits = model(x)
        loss = criterion(logits, y)

    scaler.scale(loss).backward()                    # 放大后反向

    scaler.unscale_(optimizer)                       # 还原梯度，才能裁剪
    torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)

    scaler.step(optimizer)                           # 内部会检查是否溢出
    scaler.update()                                  # 调整缩放系数
```

★ **关键认知**：`GradScaler` 只对 float16 必要。bfloat16 的指数范围和 float32 一样，不需要缩放，直接 `loss.backward()` 即可。看到别人用 bf16 还套 scaler，多半是从 fp16 模板抄过来的。

⚠ 裁剪梯度前必须 `scaler.unscale_(optimizer)`。不还原就裁剪，实际裁的是放大后的梯度，阈值完全失真。`scaler.step()` 内部会跳过溢出步，所以不要自己再判断梯度是否 NaN。

## 2.4 评估时的混合精度

```python
model.eval()
with torch.inference_mode(), torch.amp.autocast("cuda", dtype=torch.float16):
    logits = model(x_val)
```

推理不需要 `GradScaler`，因为不反向。导出 ONNX 或 TorchScript 时是否保留 autocast，要按目标运行环境决定。

# 3. 多卡边界

## 3.1 DataParallel

```python
model = torch.nn.DataParallel(model, device_ids=[0, 1])
```

单进程多线程，每个 batch 在主卡切分再分发。实现简单，但受 Python GIL 限制、主卡显存压力大，现在已经不推荐用于正式训练。

## 3.2 DistributedDataParallel

DDP 每个进程一张卡，各自前向反向，最后对梯度做 all-reduce：

```text
进程 0 (GPU 0)        进程 1 (GPU 1)
   forward               forward
   backward              backward
      └──── all-reduce 梯度 ────┘
   optimizer.step()      optimizer.step()
```

启动方式：

```bash
torchrun --nproc_per_node=2 train.py
```

代码骨架：

```python
import os
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

def main():
    dist.init_process_group(backend="nccl")
    local_rank = int(os.environ["LOCAL_RANK"])
    torch.cuda.set_device(local_rank)

    model = MyModel().to(local_rank)
    model = DDP(model, device_ids=[local_rank])

    sampler = torch.utils.data.DistributedSampler(train_ds)
    loader = torch.utils.data.DataLoader(
        train_ds, batch_size=32, sampler=sampler, num_workers=4,
    )

    for epoch in range(epochs):
        sampler.set_epoch(epoch)   # 保证每个 epoch 的划分不同
        for x, y in loader:
            ...
            optimizer.step()

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
```

★ **关键认知**：DDP 里每个进程都是**完整的训练脚本**，模型、优化器、数据加载器各一份。DDP 只负责梯度同步，不会帮你切数据，所以必须用 `DistributedSampler`；它也不会帮你算全局 batch，实际等效 batch 是 `per_gpu_batch × world_size`。

| 维度 | DataParallel | DDP |
|------|--------------|-----|
| 进程 | 单进程多线程 | 多进程 |
| 主卡压力 | 大 | 均衡 |
| 速度 | 慢，受 GIL 限制 | 接近线性加速 |
| 推荐度 | 仅调试 | 正式训练 |

⚠ 不要把 DataParallel 和 DDP 混用，也不要在 DDP 包装后再改模型结构。需要保存时用 `model.module.state_dict()`，或者直接保存 `model.state_dict()`（新版会去掉 `module.` 前缀）。

# 4. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| `CUDA out of memory` | batch 太大、激活未释放 | 降 batch、开 AMP、检查 `.item()` |
| 设备不一致 | 有张量没搬到 GPU | 检查 buffer 和中间张量 |
| AMP 训练 loss 为 NaN | 没用 scaler 或 scaler 用法错 | 按 `scale → backward → unscale → step → update` |
| 裁剪阈值不生效 | 在 unscale 前裁剪 | 先 `scaler.unscale_` |
| 多卡速度没提升 | 数据没切分或卡在 IO | 用 `DistributedSampler`，加 worker |
| 多卡指标对不上 | 只在一个进程里算指标 | 跨进程 all-reduce 后再统计 |
| 单卡能跑多卡 OOM | 每卡 batch 没减半 | 按卡数调整 per-gpu batch |

显存排查顺序：

```text
1. 把 batch_size 减半，确认是否激活占用
2. 开 AMP，观察峰值显存
3. 检查是否有张量被列表持有（指标、loss 历史）
4. 检查是否误在 Dataset 里创建 CUDA 张量
5. 最后才考虑 gradient checkpointing 或模型并行
```

# 5. 练习

1. 写一个函数自动选择 `cuda`、`mps` 或 `cpu`，并把模型和一个 batch 搬过去，打印前后的 `device`。
2. 给第 5 篇的训练循环加上 AMP，分别在 fp16 和 bf16 下跑一个 epoch，记录峰值显存和耗时。
3. 用 `torchrun --nproc_per_node=2` 跑一个最小 DDP 脚本，确认两个进程都能打印自己的 `local_rank`。

```python
# 练习 2 的骨架
scaler = torch.amp.GradScaler("cuda", enabled=(device.type == "cuda"))

with torch.amp.autocast("cuda", dtype=torch.float16, enabled=(device.type == "cuda")):
    logits = model(x)
    loss = criterion(logits, y)

scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
```

# 6. 小结

| 主题 | 要点 |
|------|------|
| 设备 | 统一用 `torch.device`，模型与数据同设备 |
| 显存 | 参数、梯度、优化器状态、激活四块，激活最容易被忽视 |
| 缓存 | `empty_cache` 只释放空闲缓存，不解决根因 |
| AMP | `autocast` 自动选精度，fp16 需配 `GradScaler` |
| 顺序 | `scale → backward → unscale → clip → step → update` |
| bf16 | 范围大，通常不需要 scaler |
| 多卡 | 正式训练用 DDP，配 `DistributedSampler` |
| 等效 batch | 每卡 batch × 卡数，学习率要相应调整 |

下一篇进入保存与加载，讲清 `state_dict` 里到底有什么、checkpoint 如何完整恢复训练、`map_location` 与 `weights_only` 的安全边界。
