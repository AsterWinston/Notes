训练速度往往不取决于模型，而取决于数据能不能及时喂进来。`Dataset` 定义"一条样本怎么取"，`DataLoader` 负责"按什么顺序、用什么 batch、由几个进程送过来"。这两层拆开后，换数据源不影响训练循环，换 batch 策略也不影响数据读取。

# 1. 数据管道的分工

```text
原始数据（文件 / 数据库 / 内存数组）
        │
        ▼
┌──────────────────┐
│ Dataset          │  取单条样本：dataset[i]
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Sampler          │  决定取哪些下标、按什么顺序
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ collate_fn       │  把多条样本拼成一个 batch
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ DataLoader       │  多进程、预取、锁页内存
└────────┬─────────┘
         │
         ▼
    训练循环 for batch in loader
```

★ **关键认知**：`DataLoader` 不负责"怎么读文件"，只负责**调度**。读文件、解码图片、做增强都放在 `Dataset.__getitem__` 里；拼 batch、补长度放在 `collate_fn` 里。职责分清，才能安全地开多进程。

# 2. Dataset

## 2.1 map-style：按下标取

最常见的形式，实现 `__len__` 和 `__getitem__`：

```python
import torch
from torch.utils.data import Dataset

class ArrayDataset(Dataset):
    def __init__(self, features, labels):
        self.features = features   # 假设是 float32 张量
        self.labels = labels       # 假设是 int64 张量

    def __len__(self):
        return len(self.labels)

    def __getitem__(self, idx):
        return self.features[idx], self.labels[idx]

features = torch.randn(100, 8)
labels = torch.randint(0, 3, (100,))
ds = ArrayDataset(features, labels)

print(len(ds))          # 100
x, y = ds[0]            # 取第 0 条
print(x.shape, y)
```

★ **关键认知**：`__getitem__` 应当只依赖 `idx` 和数据集自身的只读状态。如果在里面修改全局计数器、随机种子或缓存，多 worker 下每个进程各有一份状态，结果会随 worker 数变化。

内置数据集也遵循同样接口，例如 `TensorDataset`：

```python
from torch.utils.data import TensorDataset

ds = TensorDataset(features, labels)   # 按最后一个维度对齐
print(ds[0])
```

## 2.2 iterable-style：按流读取

数据量太大、或者来自日志流时，无法预先知道长度，用 `IterableDataset`：

```python
from torch.utils.data import IterableDataset

class LineDataset(IterableDataset):
    def __init__(self, path):
        self.path = path

    def __iter__(self):
        with open(self.path, encoding="utf-8") as f:
            for line in f:
                text = line.strip()
                if text:
                    yield torch.tensor(len(text), dtype=torch.float32), text
```

⚠ 对 `IterableDataset`，不能使用 `shuffle=True`，传入会直接报错；`len(loader)` 也可能没有意义。需要打乱时要自己实现缓冲池，或者先落成可索引的格式。

## 2.3 返回值的约定

一条样本通常返回 `(features, label)`，也可以返回字典：

```python
def __getitem__(self, idx):
    return {
        "input_ids": self.input_ids[idx],
        "attention_mask": self.mask[idx],
        "labels": self.labels[idx],
    }
```

只要 `collate_fn` 能处理，返回什么结构都行。团队项目里推荐字典，字段名比位置更不容易搞错。

# 3. DataLoader

## 3.1 基本参数

```python
from torch.utils.data import DataLoader

loader = DataLoader(
    ds,
    batch_size=16,      # 每个 batch 的样本数
    shuffle=True,       # 每个 epoch 重新打乱
    drop_last=True,     # 丢弃最后一个不完整 batch
    num_workers=0,      # 0 表示主进程读取
)

for x, y in loader:
    print(x.shape, y.shape)
    break
```

| 参数 | 作用 | 建议 |
|------|------|------|
| `batch_size` | 每批样本数 | 受显存限制，常用 32/64/128 |
| `shuffle` | 是否打乱 | 训练开，验证关 |
| `drop_last` | 丢掉尾批 | 有 BatchNorm 且尾批很小时开 |
| `num_workers` | 读取进程数 | CPU 核数的 1/4 到 1/2 起步 |
| `pin_memory` | 锁页内存 | 有 GPU 时开 |
| `persistent_workers` | 保持 worker 存活 | 多 epoch 且 `num_workers > 0` 时开 |

`shuffle=True`、`sampler`、`batch_sampler` 三者互斥，不能同时指定。

## 3.2 num_workers 的边界

`num_workers=0` 表示在主进程里读，调试最方便，但数据预处理会阻塞训练。设成大于 0 后，每个 worker 是一个独立进程，通过队列把 batch 送给主进程。

```python
loader = DataLoader(
    ds,
    batch_size=32,
    shuffle=True,
    num_workers=4,
    persistent_workers=True,  # 避免每个 epoch 重新创建进程
    prefetch_factor=2,        # 每个 worker 预取 2 个 batch
)
```

★ **关键认知**：`num_workers` 不是越大越好。进程创建、数据拷贝、内存占用都会随之上来；当读取本身是瓶颈时，增加 worker 才有收益。先用 `num_workers=0` 跑通，再逐步加到吞吐不再提升为止。

⚠ Windows 使用 `spawn` 启动子进程，`Dataset` 必须定义在模块顶层并且可 pickle。写在函数里、带 lambda、或持有不可序列化的对象，都会在开 worker 时报错。入口脚本还要有 `if __name__ == "__main__":` 保护。

## 3.3 pin_memory 与设备

`pin_memory=True` 会把 batch 放进锁页内存，让 CPU 到 GPU 的拷贝可以异步进行。它只在有 CUDA 时有意义：

```python
loader = DataLoader(ds, batch_size=64, pin_memory=True, num_workers=4)

for x, y in loader:
    x = x.to("cuda", non_blocking=True)
    y = y.to("cuda", non_blocking=True)
```

⚠ 不要在 `Dataset` 里直接创建 CUDA 张量。worker 是独立进程，各自持有 CUDA 上下文既浪费显存，也容易出问题。正确做法是 worker 返回 CPU 张量，主进程再搬到 GPU。

# 4. collate_fn

## 4.1 默认拼接

默认的 `collate_fn` 会把一个 batch 的样本按字段堆叠：

```python
# 每条样本是 (shape=(8,), scalar)
# 默认输出：features (B, 8)，labels (B,)
```

默认行为要求同一字段形状一致。图像尺寸不一、文本长度不一时会直接报错。

## 4.2 变长序列

自定义 `collate_fn`，用 `pad_sequence` 补齐：

```python
from torch.nn.utils.rnn import pad_sequence

def collate_fn(batch):
    xs, ys = zip(*batch)                       # 拆成两个元组
    xs = pad_sequence(xs, batch_first=True, padding_value=0)
    ys = torch.stack(ys)
    return xs, ys

loader = DataLoader(ds, batch_size=8, collate_fn=collate_fn)
```

如果序列很长，补齐会浪费大量计算，可以按长度分桶后再采样。另一种做法是返回长度列表，在模型里用 mask 处理。

★ **关键认知**：`collate_fn` 运行在 worker 进程里，不能依赖主进程的全局状态。需要随机增强时，用 `worker_init_fn` 给每个 worker 设种子，或者用 `torch.Generator` 显式传入。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| Windows 下开 worker 卡死 | 缺少 `__main__` 保护 | 入口加 `if __name__ == "__main__":` |
| `Dataset` 无法 pickle | 定义在函数内或含 lambda | 移到模块顶层，去掉不可序列化对象 |
| 变长样本拼接报错 | 默认 `collate_fn` 要求同形状 | 自定义 `collate_fn` 补齐 |
| 训练越跑越慢 | worker 数过多、内存抖动 | 降低 `num_workers`，检查数据集 |
| 验证集顺序每次不同 | 忘记 `shuffle=False` | 验证 loader 关掉 shuffle |
| `len(loader)` 不存在 | 使用了 `IterableDataset` | 自己记录样本数或按 batch 计数 |
| 尾批只有 1 条导致 BatchNorm 报错 | 没有 `drop_last` | 训练时 `drop_last=True` |

# 6. 练习

1. 实现一个 `Dataset`，返回 `(随机向量, 类别)`，用 `DataLoader` 取三个 batch，打印每个 batch 的形状。
2. 构造变长序列数据，自定义 `collate_fn` 用 `pad_sequence` 补齐，验证输出形状和 padding 位置。
3. 用 `random_split` 把数据集切成训练集和验证集，分别创建 `DataLoader`，确认验证集没有打乱。

```python
# 练习 3 的起点
from torch.utils.data import random_split

train_ds, val_ds = random_split(ds, [80, 20])
train_loader = DataLoader(train_ds, batch_size=16, shuffle=True)
val_loader = DataLoader(val_ds, batch_size=16, shuffle=False)
print(len(train_loader), len(val_loader))
```

# 7. 小结

| 主题 | 要点 |
|------|------|
| Dataset | map-style 实现 `__len__`、`__getitem__`；流式用 `__iter__` |
| DataLoader | 负责采样、批处理、多进程与预取 |
| shuffle | 训练开，验证关；与 sampler 互斥 |
| num_workers | 0 便于调试，>0 提升吞吐；Windows 注意 `__main__` |
| pin_memory | 有 GPU 时开，配合 `non_blocking=True` |
| collate_fn | 处理变长、异构样本；运行在 worker 中 |
| 设备 | worker 返回 CPU 张量，主进程再搬到 GPU |

下一篇进入训练循环，把模型、数据、损失和优化器串起来，讲清每个步骤的顺序、指标如何累计，以及早停该怎么写。
