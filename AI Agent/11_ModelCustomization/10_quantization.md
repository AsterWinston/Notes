# 1. 为什么量化

量化（Quantization）是把模型中以高精度浮点存储的参数、激活值或缓存，转换成低位宽整数（或低精度浮点）表示的技术。它不改变模型结构，只改变数值的"存储与计算精度"，因此在部署阶段几乎总能用极小的改造成本，换来显存、吞吐与成本的显著收益。

## 1.1 显存瓶颈

```
显存不足是最常见的部署瓶颈：
  70B FP16 需要 140GB → 单卡装不下
  INT4 量化后 35GB → 单卡可跑
```

模型权重的显存占用可用下式估算：

```
权重显存 ≈ 参数量 × 每参数字节数

  70B × 2 字节（FP16）  = 140 GB
  70B × 1 字节（INT8）  =  70 GB
  70B × 0.5 字节（INT4）=  35 GB
```

推理时的显存并非只有权重，还包括 KV Cache、激活值与框架开销。以 70B 模型为例：

| 组成 | FP16 占用 | 说明 |
|------|:---------:|------|
| 权重 | 140 GB | 与参数量线性相关 |
| KV Cache | 随上下文增长 | 长上下文时可达数十 GB |
| 激活值 | 数 GB | 与 batch、序列长度相关 |
| 框架开销 | 数 GB | CUDA 上下文、通信缓冲 |

> 详细原理见 `2_LLM/Inference/4_quantization.md`，本文聚焦部署实践。

## 1.2 量化的三类收益

### 1.2.1 显存收益

显存收益是最直接的动机。位宽从 FP16 降到 INT4，理论压缩比为 4 倍，实际因为分组缩放因子（scale）与零点（zero point）的额外开销，约为 3.5 倍。这使得原本需要多卡张量并行（Tensor Parallelism）的模型，可以退回到单卡部署。

### 1.2.2 吞吐收益

低位宽带来两个吞吐优势：

- 权重读取带宽减半或更多，内存带宽（Memory Bandwidth）瓶颈缓解，解码（decode）阶段更快。
- 权重占用减少，剩余显存可分配给更大的批（batch）或更长的上下文，提升并发吞吐。

### 1.2.3 成本收益

在同等吞吐目标下，量化可以用更少、更便宜的 GPU 完成，直接降低单位 token 的推理成本。对自建推理服务，这是最具商业价值的收益。

## 1.3 量化不改变什么

量化通常不改变模型能力边界，但会带来一定的质量损失（Quality Loss）。它无法：

- 弥补训练数据的缺失；
- 提升模型的推理或知识能力；
- 修复对齐或安全问题。

因此量化属于"部署优化"，不是"能力增强"。

# 2. 量化基础原理

理解公式有助于判断量化格式与参数选择，也能解释为什么某些模型量化后掉点严重。

## 2.1 数值表示与位宽

| 表示 | 位宽 | 取值范围（近似） | 特点 |
|------|:----:|------------------|------|
| FP32 | 32 | 约 ±3.4e38 | 训练默认 |
| FP16 | 16 | 约 ±65504 | 推理常用 |
| BF16 | 16 | 约 ±3.4e38 | 动态范围大 |
| FP8 | 8 | 视格式 | H100 等新硬件 |
| INT8 | 8 | -128 ~ 127 | 通用 |
| INT4 | 4 | -8 ~ 7 | 高压缩 |

位宽越小，表示越粗糙，量化误差（Quantization Error）越大。

## 2.2 量化公式

### 2.2.1 对称量化

对称量化把浮点值线性映射到以 0 为中心的整数区间：

```
q = round(x / s)          # 量化
x̂ = q × s                 # 反量化
s = max(|x|) / (2^(b-1) - 1)
```

其中 `s` 是缩放因子（scale），`b` 是位宽。对称量化不需要零点，适合权重分布近似对称的场景。

### 2.2.2 非对称量化

非对称量化引入零点 `z`，以覆盖非对称分布：

```
q = round(x / s) + z
x̂ = (q - z) × s
s = (max(x) - min(x)) / (2^b - 1)
z = round(-min(x) / s)
```

非对称量化对激活值更友好，但多一个 `z` 的存储与计算开销。

### 2.2.3 分组量化

整层共享一个 `s` 时，个别离群值（outlier）会拉大动态范围，导致其余数值精度崩塌。分组量化（Group-wise Quantization）把权重按固定长度分组，每组独立计算 `s` 与 `z`：

```
q_group_size = 128   # 每 128 个权重共享一组量化参数
```

组越小精度越高，但额外存储越多。128 是实践中的常见折中。

## 2.3 量化粒度

| 粒度 | 说明 | 精度 | 开销 |
|------|------|:----:|:----:|
| Per-tensor | 整层共享 | 低 | 最小 |
| Per-channel | 每通道独立 | 中 | 小 |
| Per-group | 每 128 权重 | 高 | 中 |
| Per-token | 按 token 动态 | 高 | 中 |

## 2.4 PTQ 与 QAT

```
PTQ（Post-Training Quantization，训练后量化）
  - 不需要重新训练，只需少量校准数据
  - AWQ / GPTQ / GGUF 均属此类
  - 成本低，是部署主流

QAT（Quantization-Aware Training，量化感知训练）
  - 训练时模拟量化误差
  - 精度最好，但需完整训练流程
  - 成本高，多用于极低位宽
```

## 2.5 量化误差来源

量化误差并非单一来源，定位掉点问题时需要逐项排查：

### 2.5.1 截断与舍入

- 截断误差（Clipping Error）：超出 `[min, max]` 的值被裁剪，离群值尤其严重。
- 舍入误差（Rounding Error）：`round` 操作引入的固有偏差，位宽越低越明显。

### 2.5.2 离群值处理

激活值中的离群通道（outlier channel）会拉大动态范围，使多数正常值挤在少数几个量化级别上。常见对策：

| 对策 | 说明 |
|------|------|
| 分组量化 | 缩小统计范围 |
| 激活感知缩放 | AWQ 的核心思想 |
| 混合精度 | 离群通道保留高位宽 |
| 平滑（SmoothQuant） | 把激活难度迁移到权重 |

# 3. 量化格式对比

## 3.1 主流格式一览

| 格式 | 位宽 | 推理框架 | 特点 |
|------|:----:|----------|------|
| GPTQ | 4bit | vLLM、AutoGPTQ | GPU 友好 |
| AWQ | 4bit | vLLM、TensorRT | 精度好 |
| GGUF | 2~8bit | llama.cpp、Ollama | CPU 友好 |
| bitsandbytes | 4/8bit | Transformers | 易用 |
| FP8 | 8bit | vLLM（H100） | 新硬件 |
| INT8 | 8bit | 多数 | 损失小 |

## 3.2 格式选择决策

```
部署硬件是什么？
├─ NVIDIA GPU（数据中心）
│   ├─ 追求精度 → AWQ
│   ├─ 追求生态广 → GPTQ
│   └─ H100/H200 → FP8
├─ Apple Silicon / CPU
│   └─ GGUF（Q4_K_M 起步）
└─ 快速实验
    └─ bitsandbytes（无需离线转换）
```

## 3.3 格式互操作

不同格式之间一般不能直接互转，常见路径是"回到 FP16 再重新量化"：

```
HF FP16 ──→ AWQ ──→ vLLM
        ──→ GPTQ ─→ vLLM
        ──→ GGUF ─→ llama.cpp / Ollama
```

因此建议保留一份 FP16 权重作为"母版"，所有量化版本都由它派生。

## 3.4 格式选择清单

| 场景 | 推荐格式 | 理由 |
|------|----------|------|
| 数据中心 GPU、高并发 | AWQ 4bit | 精度与吞吐兼顾 |
| 生态兼容优先 | GPTQ 4bit | 框架支持最广 |
| 单机 CPU / Mac | GGUF Q4_K_M | 无需 GPU |
| H100 及以上 | FP8 | 硬件原生支持 |
| 快速验证 | bitsandbytes | 在线量化零转换 |
| 显存仍充足 | INT8 | 损失极小 |

# 4. AWQ 量化

## 4.1 AWQ 原理

AWQ（Activation-aware Weight Quantization，激活感知权重量化）的核心观察是：并非所有权重同等重要，只有与大幅值激活通道相连的少量权重才是关键的。AWQ 在量化前按激活统计对这些关键通道做缩放保护，从而在 4bit 下保留更多精度。

## 4.2 AWQ 实操

### 4.2.1 量化脚本

```python
from awq import AutoAWQForCausalLM
from transformers import AutoTokenizer

model_path = "Qwen/Qwen2.5-7B"
quant_path = "Qwen2.5-7B-AWQ"

model = AutoAWQForCausalLM.from_pretrained(model_path)
tokenizer = AutoTokenizer.from_pretrained(model_path)

quant_config = {"zero_point": True, "q_group_size": 128, "w_bit": 4, "version": "GEMM"}

model.quantize(tokenizer, quant_config=quant_config)
model.save_quantized(quant_path)
tokenizer.save_pretrained(quant_path)
```

### 4.2.2 部署

```bash
# 用 vLLM 部署 AWQ 模型
vllm serve Qwen2.5-7B-AWQ --quantization awq
```

## 4.3 参数详解

| 参数 | 含义 | 建议 |
|------|------|------|
| `w_bit` | 权重位宽 | 4 |
| `q_group_size` | 分组大小 | 128 |
| `zero_point` | 是否非对称 | True |
| `version` | 计算核 | GEMM |

`version` 选 `GEMM` 适合大多数 GPU；`GEMV` 适合 batch=1 的低延迟场景。

## 4.4 AWQ 常见问题

### 4.4.1 量化失败

- 显存不足：先在 CPU 上加载或使用更小的校准集。
- 版本不匹配：`autoawq`、`transformers`、CUDA 三者需兼容。

### 4.4.2 精度不足

- 降低 `q_group_size`（如 64）可提升精度，代价是模型略大。
- 校准集换成更贴近业务的样本。
- 对敏感层（如首尾层）保留 FP16，做混合精度。

# 5. GPTQ 量化

## 5.1 GPTQ 原理

GPTQ（Generative Pre-trained Transformer Quantization）基于逐层误差最小化：它按列依次量化权重，并用 Hessian 矩阵的二阶信息补偿已量化列带来的误差，使量化后该层输出尽可能接近原始输出。

## 5.2 GPTQ 实操

```python
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

quantize_config = BaseQuantizeConfig(
    bits=4,
    group_size=128,
    desc_act=False)

model = AutoGPTQForCausalLM.from_pretrained(model_path, quantize_config)
model.quantize(calibration_dataset)
model.save_quantized("model-gptq-4bit")
```

`desc_act`（activation order）开启后按激活重要性排序，精度略好但推理更慢，需框架支持。

## 5.3 与 AWQ 对比

| 维度 | AWQ | GPTQ |
|------|-----|------|
| 精度 | 通常略好 | 好 |
| 量化速度 | 快 | 较慢 |
| 推理速度 | 快 | 快 |
| 生态 | 广 | 最广 |
| 校准数据 | 需要 | 需要 |

# 6. GGUF 量化

## 6.1 GGUF 概述

GGUF（GPT-Generated Unified Format）是 llama.cpp 生态的模型格式，支持 2~8bit 多级量化，天然适合 CPU 与 Apple Silicon，并被 Ollama 直接使用。

## 6.2 量化流程

```bash
# 转换 HF 模型为 GGUF
python convert_hf_to_gguf.py ./model --outfile model-f16.gguf

# 量化
./llama-quantize model-f16.gguf model-q4_k_m.gguf Q4_K_M
```

## 6.3 量化级别选择

| 级别 | 质量 | 大小 |
|------|------|------|
| Q8_0 | 最好 | 大 |
| Q6_K | 很好 | 中 |
| Q5_K_M | 好 | 中 |
| Q4_K_M | 平衡（推荐） | 小 |
| Q3_K_M | 一般 | 更小 |
| Q2_K | 差 | 最小 |

K 系列（K-quants）对部分张量使用更高位宽，质量优于同尺寸的旧版 Q4_0/Q5_0。

## 6.4 与 Ollama 集成

```bash
# 写 Modelfile
cat > Modelfile <<'EOF'
FROM ./model-q4_k_m.gguf
PARAMETER num_ctx 8192
PARAMETER temperature 0.7
EOF

# 创建并运行
ollama create my-model -f Modelfile
ollama run my-model
```

# 7. 精度与效果

## 7.1 位宽与损失

| 位宽 | 显存 | 质量损失 | 建议 |
|:----:|:----:|:--------:|------|
| FP16 | 100% | 0 | 显存充足 |
| INT8 | 50% | 极小 | 通用 |
| INT4 | 25% | 小 | 推荐 |
| INT3 | 19% | 中 | 慎用 |
| INT2 | 12.5% | 大 | 不推荐 |

## 7.2 质量评估方法

量化损失必须在任务数据上测量，常用方法：

- 困惑度（Perplexity，PPL）：在验证语料上计算，越低越好，但与人感相关性有限。
- 任务指标：在业务评测集上跑准确率、F1、通过率。
- 人工盲评：对生成质量做主观对比。

## 7.3 INT4 拐点分析

> **INT4 是性价比拐点**：显存降 75%，质量损失可接受。

从 INT4 再降到 INT3/INT2，显存收益递减而质量损失陡增，因此除非极端受限，不建议低于 INT4。

## 7.4 掉点归因

量化后掉点通常集中在少数敏感层，定位思路如下：

### 7.4.1 层敏感度分析

逐层替换为 FP16，观察指标恢复情况，找出贡献最大的层：

```
for layer in model.layers:
    mixed = quantized_with_fp16(layer)
    delta = evaluate(mixed) - evaluate(quantized)
    print(layer, delta)   # delta 越大，该层越敏感
```

### 7.4.2 混合精度策略

对敏感层保留 INT8/FP16，其余层保持 INT4，可在几乎不增加显存的前提下挽回大部分精度。

# 8. 校准数据

## 8.1 校准数据的作用

```
PTQ（训练后量化）需要校准数据：
  - 从训练/业务数据中采样
  - 通常 128~512 条
  - 代表性很重要
```

校准数据用于估计激活分布，从而确定缩放因子。数据不具代表性会显著放大量化误差。

## 8.2 数据构造

```python
calibration_data = load_dataset("业务数据", split="train[:256]")
model.quantize(calibration_data)
```

建议：

- 条数 128~512 条即可，过多收益递减。
- 覆盖真实业务分布，包含长尾与边界样本。
- 序列长度尽量接近推理时的典型长度。

## 8.3 常见陷阱

| 陷阱 | 后果 | 对策 |
|------|------|------|
| 用通用语料校准领域模型 | 领域掉点大 | 用业务数据 |
| 校准集过小 | 缩放因子不稳 | 至少 128 条 |
| 校准集与推理长度差异大 | 激活估计偏差 | 对齐长度分布 |

# 9. KV Cache 量化

## 9.1 KV Cache 显存

长上下文推理中，KV Cache 可能成为主要显存消耗，其大小近似为：

```
KV Cache ≈ 2 × 层数 × 头数 × 头维度 × 序列长度 × batch × 字节数
```

## 9.2 量化方案

```bash
# vLLM 支持 KV Cache 量化
vllm serve model \
  --quantization awq \
  --kv-cache-dtype fp8        # KV Cache 用 FP8
```

## 9.3 收益与风险

| 优点 | 缺点 |
|------|------|
| 支持更长上下文 | 可能影响质量 |
| 支持更大批 | 部分硬件不支持 |

KV Cache 量化与权重量化相互独立，可组合使用。

## 9.4 配置示例

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="Qwen2.5-7B-AWQ",
    quantization="awq",
    kv_cache_dtype="fp8",
    max_model_len=32768,
    gpu_memory_utilization=0.90,
)

out = llm.generate(["你好"], SamplingParams(max_tokens=64))
```

# 10. 量化后评估

## 10.1 评测流程

```
1. 准备业务评测集（含目标、通用、安全样本）
2. 分别评测 FP16 与量化版本
3. 计算各项指标差值
4. 若损失超过阈值 → 调整量化参数或退回 INT8
```

## 10.2 评测代码

```python
def evaluate_quantized(original, quantized, test_cases):
    original_score = evaluate(original, test_cases)
    quantized_score = evaluate(quantized, test_cases)

    print(f"原始: {original_score}")
    print(f"量化: {quantized_score}")
    print(f"损失: {original_score - quantized_score}")

    assert quantized_score >= original_score - 0.03, "量化损失过大"
```

## 10.3 验收标准

| 指标 | 建议阈值 |
|------|----------|
| 目标能力损失 | ≤ 3% |
| 通用能力损失 | ≤ 2% |
| 格式遵循 | 无退化 |
| 延迟 | 不劣于 FP16 |

> **量化后必须评测**，不能假设"损失很小"。

# 11. 生产注意事项

## 11.1 常见风险

| 风险 | 建议 |
|------|------|
| 量化损失被低估 | 量化后评测 |
| 校准数据不代表性 | 用业务数据 |
| 框架不兼容 | 确认支持格式 |
| 只量化权重 | KV Cache 也可量化 |
| 盲目低位宽 | INT4 是拐点 |

## 11.2 工程实践

```
- 保留 FP16 母版权重，量化版本全部派生
- 量化脚本纳入版本管理，参数写死在配置中
- 上线前在灰度环境对比 FP16 与量化版
- 监控线上质量指标，出现回退可快速切回
```

## 11.3 部署清单

- [ ] 确认硬件支持目标格式（AWQ/GPTQ/FP8）。
- [ ] 校准数据来自业务分布。
- [ ] 已完成量化后评测并达标。
- [ ] 已评估 KV Cache 量化收益。
- [ ] 已准备 FP16 回退路径。

# 12. 小结

## 12.1 核心结论

- 量化用低精度表示参数，大幅降低显存。
- 对称/非对称、分组量化决定精度上限。
- GPU 用 AWQ/GPTQ 4bit，CPU 用 GGUF。
- INT4 是性价比拐点。
- KV Cache 也可量化，支持更长上下文。
- 量化后必须评测效果损失。

## 12.2 落地清单

- 保留 FP16 母版，量化版本全部派生。
- 校准数据取自业务分布，128~512 条。
- 上线前完成评测并设定回滚阈值。
- 灰度发布，监控质量与延迟指标。
