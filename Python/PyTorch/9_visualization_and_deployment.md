训练跑通只是第一步。要知道模型有没有在学、该不该停，需要可视化；要把它交给别人用，需要导出和推理。可视化、序列化格式、服务边界是三件不同的事，混在一起做很容易留下"本地能跑、线上结果不对"的隐患。

# 1. 训练可视化

## 1.1 SummaryWriter

TensorBoard 是 PyTorch 默认支持的可视化工具，写入端就是 `SummaryWriter`：

```python
from torch.utils.tensorboard import SummaryWriter

writer = SummaryWriter("runs/exp1")   # 日志目录

for epoch in range(epochs):
    train_loss, train_acc = train_one_epoch(...)
    val_loss, val_acc = evaluate(...)

    writer.add_scalar("loss/train", train_loss, epoch)
    writer.add_scalar("loss/val", val_loss, epoch)
    writer.add_scalar("acc/train", train_acc, epoch)
    writer.add_scalar("acc/val", val_acc, epoch)
    writer.add_scalar("lr", optimizer.param_groups[0]["lr"], epoch)

writer.close()
```

启动服务：

```bash
tensorboard --logdir runs --port 6006
```

浏览器打开 `http://localhost:6006`。多个实验放在 `runs/` 下的不同子目录，TensorBoard 会自动对比。

## 1.2 常用记录类型

| 方法 | 记录内容 | 用途 |
|------|----------|------|
| `add_scalar` | 单个数值 | loss、accuracy、学习率 |
| `add_scalars` | 多个同轴曲线 | 多组实验对比 |
| `add_histogram` | 分布 | 权重、梯度的健康度 |
| `add_image` | 图像张量 | 输入样本、分割结果 |
| `add_graph` | 计算图 | 检查网络结构 |
| `add_text` | 文本 | 配置、备注 |

```python
# 权重分布，排查梯度消失或爆炸
for name, param in model.named_parameters():
    writer.add_histogram(name, param.detach().cpu(), epoch)

# 记录一张输入图像，数据格式为 (C, H, W)
writer.add_image("sample", x[0].detach().cpu(), epoch, dataformats="CHW")

# 记录模型结构
writer.add_graph(model, torch.randn(1, 8))
```

★ **关键认知**：可视化的价值不在"画得好看"，而在**把训练过程变成可比较的曲线**。训练 loss 降、验证 loss 升是过拟合；两条都平是学习率太小或数据有问题；loss 突然变 NaN 要回看学习率和梯度。没有曲线，只能靠猜。

⚠ 每一步都写 `add_scalar` 会让日志膨胀、训练变慢。按 step 记录高频指标，按 epoch 记录权重直方图；长期训练要定期清理旧日志。

## 1.3 记录哪些指标

最小集合：

```text
loss/train, loss/val        判断拟合与泛化
acc/train, acc/val          判断任务指标
lr                          判断调度是否生效
grad_norm                   判断梯度是否爆炸
```

排查阶段再加：

```text
param/mean, param/std       权重是否退化
activation/mean             激活是否饱和
throughput/samples_per_s    数据管道是否成瓶颈
```

# 2. TorchScript

## 2.1 为什么要导出

Python 模型依赖解释器和项目代码，部署到 C++ 服务或移动端时不一定有这些环境。TorchScript 把模型转成可独立加载的图表示，运行时不依赖原来的 Python 类。

## 2.2 trace 与 script

```python
import torch

model.eval()
example = torch.randn(1, 8)

# 方式一：追踪，用一组输入跑一遍，记录实际执行的算子
traced = torch.jit.trace(model, example)
traced.save("model_traced.pt")

# 方式二：脚本化，直接编译 Python 子集，保留控制流
scripted = torch.jit.script(model)
scripted.save("model_scripted.pt")
```

| 方式 | 原理 | 支持动态控制流 | 适用 |
|------|------|----------------|------|
| `trace` | 记录一次执行的算子序列 | 否 | 结构固定、无 if/for 依赖输入 |
| `script` | 编译 Python 源码 | 是 | 含条件分支、循环的模型 |

★ **关键认知**：`trace` 只记录"这一组输入实际走过的路径"。模型里有 `if x.sum() > 0` 这类依赖输入的分支，换个输入就会走错分支。结构固定用 `trace` 最省事，有数据相关控制流必须用 `script`。

加载与推理：

```python
loaded = torch.jit.load("model_traced.pt", map_location="cpu")
loaded.eval()

with torch.inference_mode():
    out = loaded(torch.randn(1, 8))
```

推理前可以再优化：

```python
optimized = torch.jit.optimize_for_inference(traced)
optimized.save("model_opt.pt")
```

⚠ `torch.jit.trace` 会给出 "TracerWarning"，提示某处可能丢失控制流。不要忽略它，逐条确认那些分支是否真的与输入无关。

## 2.3 torch.compile 不是导出格式

`torch.compile(model)` 在 Python 进程内把热点图编译成更快的算子，提升训练或推理速度，但它**不是跨语言、跨进程的序列化格式**。要交付给别的运行时，仍然用 TorchScript、ONNX 或框架自己的格式。

# 3. ONNX

## 3.1 导出

ONNX 是跨框架的模型交换格式，PyTorch 模型可以导出后交给 ONNX Runtime、TensorRT 等执行：

```python
import torch

model.eval()
example = torch.randn(1, 8)

torch.onnx.export(
    model,
    (example,),
    "model.onnx",
    input_names=["input"],
    output_names=["logits"],
    opset_version=17,
    do_constant_folding=True,
)
```

导出前必须 `model.eval()`，否则 Dropout、BatchNorm 会带着训练行为进入图。

## 3.2 动态维度

默认导出会把输入形状写死。需要支持任意 batch 或变长序列时，用 `dynamic_axes` 标记：

```python
torch.onnx.export(
    model,
    (example,),
    "model_dynamic.onnx",
    input_names=["input"],
    output_names=["logits"],
    dynamic_axes={
        "input": {0: "batch", 1: "seq_len"},
        "logits": {0: "batch"},
    },
    opset_version=17,
)
```

验证导出结果：

```python
import numpy as np
import onnxruntime as ort

session = ort.InferenceSession("model_dynamic.onnx", providers=["CPUExecutionProvider"])
x = np.random.randn(4, 8).astype(np.float32)   # batch 换成 4 也能跑
out = session.run(None, {"input": x})
print(out[0].shape)
```

★ **关键认知**：导出成功不等于结果正确。必须用同一份输入分别跑 PyTorch 和 ONNX Runtime，比较输出的最大绝对误差。预处理、归一化、通道顺序任何一处不一致，导出的图都"能跑但不对"。

## 3.3 常见限制

| 限制 | 表现 | 处理 |
|------|------|------|
| 不支持的算子 | 导出报错 | 换等价实现或升级 opset |
| 动态控制流 | 导出后行为固定 | 改用 `script` 或改写为张量运算 |
| 自定义算子 | 目标运行时找不到 | 提供插件或退回 TorchScript |
| 输入形状写死 | 换 batch 报错 | 配置 `dynamic_axes` |
| 训练态残留 | 结果与评估不一致 | 导出前 `eval()` |

PyTorch 2.x 也在推进基于 `torch.export` 和 dynamo 的新导出路径，但老项目、老运行时仍以 `torch.onnx.export` 为主。选哪条路取决于目标推理引擎的支持程度，不要只看 PyTorch 版本。

# 4. 推理服务的边界

## 4.1 推理模式

```python
model.eval()

@torch.inference_mode()
def predict(x):
    logits = model(x)
    return logits.argmax(dim=1)
```

`eval()` 改层行为，`inference_mode()` 关梯度并省掉版本计数。两者都要。

## 4.2 预处理必须与训练一致

线上最常见的事故不是模型错了，而是预处理错了：

```text
训练时：RGB、除以 255、按 ImageNet 均值方差归一化、Resize 到 224
线上时：BGR、没除 255、没归一化、Resize 到 256 再中心裁剪
```

结果就是离线指标 95%，线上一塌糊涂。把预处理封装成一个函数，训练和推理共用同一份代码，是成本最低的保险。

## 4.3 批处理与延迟

```text
请求到达
  │
  ▼
预处理（CPU）
  │
  ▼
批处理队列 ── 攒到 batch_size 或超时
  │
  ▼
模型推理（GPU）
  │
  ▼
后处理（CPU）──► 返回
```

| 策略 | 吞吐 | 延迟 | 适用 |
|------|------|------|------|
| 单条推理 | 低 | 低 | 实时交互 |
| 固定 batch | 高 | 中 | 离线批处理 |
| 动态批处理 | 高 | 可控 | 在线服务 |

★ **关键认知**：GPU 在小 batch 下利用率很低，但攒 batch 会增加排队延迟。线上要在"吞吐"和"P99 延迟"之间取平衡，通常设一个最大等待时间，超时就用当前攒到的样本先跑。

## 4.4 服务化选择

| 方案 | 特点 | 适用 |
|------|------|------|
| 自己写 FastAPI | 灵活，需自己做批处理、并发、监控 | 小规模、定制逻辑 |
| TorchServe | 官方，支持模型归档与多模型 | 中等规模 PyTorch 服务 |
| Triton Inference Server | 多框架、动态批处理、GPU 调度强 | 大规模 GPU 推理 |
| ONNX Runtime | 跨平台、CPU 优化好 | 边缘、非 Python 环境 |

不要在每次请求里重新加载模型，也不要把模型放在 Web 进程的全局变量里反复 `.to(device)`。启动时加载一次、预热几次、常驻内存。

# 5. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| TensorBoard 没有曲线 | 日志目录不对或没 `flush` | 确认 `--logdir`，关闭 writer |
| 导出的模型结果不对 | 忘记 `eval()` | 导出前切换评估模式 |
| ONNX 换 batch 报错 | 没设 `dynamic_axes` | 标记动态维度 |
| TorchScript 分支走错 | 用了 `trace` 但有数据相关 if | 改用 `script` |
| 线上精度骤降 | 预处理不一致 | 训练与推理共用预处理函数 |
| 推理显存持续上涨 | 没关梯度或缓存了输入 | `inference_mode`，避免持有张量 |
| 服务启动很慢 | 每次请求加载模型 | 启动时加载并预热 |

# 6. 练习

1. 用 `SummaryWriter` 记录 10 个 epoch 的 train/val loss 和学习率，启动 TensorBoard 确认曲线可见。
2. 把一个含 `ReLU` 的 MLP 分别用 `trace` 和 `script` 导出，比较文件大小，并用同一输入验证输出一致。
3. 导出支持动态 batch 的 ONNX，分别用 batch=1 和 batch=8 推理，比较 PyTorch 与 ONNX Runtime 的输出误差。

# 7. 小结

| 主题 | 要点 |
|------|------|
| TensorBoard | `SummaryWriter` 记录标量、直方图、图像、计算图 |
| 指标 | loss、accuracy、lr、grad_norm 是最小集合 |
| TorchScript | `trace` 记录执行路径，`script` 保留控制流 |
| ONNX | 跨框架交换，注意 `eval()`、`dynamic_axes`、opset |
| 验证 | 导出后必须对比数值误差，不只看能否运行 |
| 推理 | `eval()` + `inference_mode()`，预处理与训练一致 |
| 服务 | 启动加载、预热、动态批处理，按延迟目标取舍 |

下一篇是模块收尾，讲训练排错的顺序、过拟合与数据泄漏的识别，以及高频自检题和整个 PyTorch 学习路径的总结。
