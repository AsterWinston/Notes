# 1. 为什么用 vLLM

结论先行：vLLM 是面向生产的高吞吐大模型推理引擎，用 PagedAttention（分页注意力）与连续批处理（Continuous Batching）把 GPU 利用率拉满，同时提供 OpenAI 兼容 API，让上层框架几乎零改造接入。

```
原生 HuggingFace 推理：慢、吞吐低
vLLM：
  ├── PagedAttention（分页 KV Cache）
  ├── 连续批处理
  ├── 高吞吐（数倍提升）
  └── OpenAI 兼容 API
```

## 1.1 核心机制

### 1.1.1 PagedAttention

传统推理为每个请求预留一段连续显存存放 KV Cache（Key-Value Cache，键值缓存），长度按最大值分配，碎片严重。PagedAttention 借鉴操作系统分页思想，把 KV Cache 切成固定大小的 block，按需分配、可非连续存放，显存浪费从数倍降到接近零。

| 方案 | 显存分配 | 碎片 | 可复用 |
|------|----------|------|--------|
| 连续预留 | 按 max_len | 高 | 否 |
| PagedAttention | 按 block 按需 | 低 | 是（前缀共享） |

### 1.1.2 连续批处理

传统静态批处理要等整批最慢的请求结束才能释放，GPU 空转。连续批处理在每个解码步（decode step）动态替换已完成的序列，新请求随到随入，吞吐随并发近似线性提升。

```
静态批： [====长请求====]
         [=短=] (空转等待)
连续批： [=短=][=新=][==长==]
         [=新=] 随到随插，无空转
```

### 1.1.3 前缀缓存

多个请求共享相同前缀（如系统提示、few-shot 示例）时，前缀缓存（Prefix Caching）复用已算好的 KV block，避免重复计算，首 token 延迟显著下降。

```
请求A: [系统提示][问题A]
请求B: [系统提示][问题B]
         └── 复用 ──┘
```

## 1.2 与其它方案对比

| 方案 | 吞吐 | 易用性 | 适用 |
|------|:----:|:------:|------|
| HF Transformers | 低 | 高 | 调试 |
| vLLM | 高 | 中 | 生产 |
| Ollama | 低 | 极高 | 本地 |
| TensorRT-LLM | 极高 | 低 | 极致性能 |

# 2. 安装与启动

结论先行：安装只需一条 pip 命令，启动核心是选对张量并行度、最大长度与显存利用率三个参数。

## 2.1 安装

```bash
pip install vllm
```

## 2.2 启动服务

```bash
# 启动 OpenAI 兼容服务
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --tensor-parallel-size 2 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.9 \
  --enable-prefix-caching
```

## 2.3 常用参数

| 参数 | 说明 |
|------|------|
| `--tensor-parallel-size` | 张量并行卡数 |
| `--max-model-len` | 最大序列长度 |
| `--gpu-memory-utilization` | 显存利用率 |
| `--enable-prefix-caching` | 前缀缓存 |
| `--quantization` | 量化（awq/gptq） |

### 2.3.1 参数怎么选

| 场景 | 建议 |
|------|------|
| 单卡放不下 | 增大 `tensor-parallel-size` |
| 长上下文 | 增大 `max-model-len`，但 KV Cache 同步增大 |
| 高并发 | 降低 `max-model-len` 以腾出 KV Cache |
| 显存吃紧 | `gpu-memory-utilization` 设 0.85~0.9 |

### 2.3.2 边界与坑

- `gpu-memory-utilization` 设 1.0 会因其它进程占用而启动失败，务必留余量。
- `tensor-parallel-size` 必须整除注意力头数，否则报错。
- `max-model-len` 设得比模型支持上限还大会直接报错，不会自动截断。

### 2.3.3 启动自检

| 检查 | 命令/日志 |
|------|-----------|
| 模型下载完整 | 启动日志无权重缺失 |
| 显存够用 | 日志显示 KV Cache 分配量 |
| 端口可达 | `curl /health` |
| 版本匹配 | CUDA / torch / vLLM 版本 |

# 3. OpenAI 兼容 API

结论先行：vLLM 的 HTTP 接口与 OpenAI 协议一致，任何支持 OpenAI 的客户端只需改 base-url 与 api-key 即可接入。

## 3.1 直接调用

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-7B-Instruct",
    "messages": [{"role": "user", "content": "你好"}],
    "temperature": 0.7
  }'
```

> **可以直接用 Spring AI / LangChain4j 的 OpenAI 适配接入**。

## 3.2 Spring AI 接入

```yaml
spring:
  ai:
    openai:
      base-url: http://localhost:8000
      api-key: dummy
      chat:
        options:
          model: Qwen/Qwen2.5-7B-Instruct
```

### 3.2.1 关键点

| 配置 | 说明 |
|------|------|
| `base-url` | 指向 vLLM，不加 `/v1`（框架自动拼接） |
| `api-key` | 任意非空占位符 |
| `model` | 必须与 `vllm serve` 的模型名一致 |

### 3.2.2 边界与坑

- 不同框架对 `base-url` 是否含 `/v1` 约定不同，需实测。
- 流式（stream）返回的 SSE 格式虽兼容，但个别字段缺失，客户端要做好容错。

### 3.2.3 流式调用

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-7B-Instruct",
       "messages":[{"role":"user","content":"你好"}],
       "stream":true}'
```

流式返回 SSE，逐块产出 token，适合对话式交互；批处理场景关闭流式更高效。

# 4. 量化部署

结论先行：量化把权重从 FP16 压到 4bit，显存降到约三成，质量略降，是单卡跑大模型的关键手段。

## 4.1 量化启动

```bash
# AWQ 量化模型
vllm serve Qwen/Qwen2.5-7B-Instruct-AWQ \
  --quantization awq \
  --max-model-len 8192

# GPTQ
vllm serve model-gptq-4bit --quantization gptq
```

## 4.2 量化对比

| 量化 | 显存 | 速度 | 质量 |
|------|:----:|:----:|:----:|
| FP16 | 100% | 1x | 最好 |
| AWQ | ~30% | 快 | 略降 |
| GPTQ | ~30% | 快 | 略降 |

## 4.3 量化选型

### 4.3.1 AWQ 与 GPTQ

| 维度 | AWQ | GPTQ |
|------|-----|------|
| 原理 | 保护重要权重通道 | 逐层误差补偿 |
| 生态 | 广泛 | 广泛 |
| 校准 | 需少量数据 | 需少量数据 |

### 4.3.2 边界与反例

- 量化对数学推理、代码等精确任务影响大于闲聊。
- 量化不减少 KV Cache 显存，长上下文仍可能 OOM。
- 非对称量化、group size 不同，同一模型不同量化版效果差异明显。

### 4.3.3 其它精度

| 精度 | 显存 | 说明 |
|------|:----:|------|
| FP8 | ~50% | 需 Hopper 及以上 |
| INT8 | ~50% | 兼容性好 |
| 4bit | ~25% | 质量损失可见 |

# 5. 性能调优

结论先行：调优的本质是在"显存—吞吐—延迟"三角里找平衡点，核心是控制并发序列数与批处理 token 上限。

## 5.1 关键参数

| 参数 | 说明 |
|------|------|
| `max-num-seqs` | 最大并发序列数 |
| `max-num-batched-tokens` | 批处理 token 上限 |
| `gpu-memory-utilization` | 显存利用率（0.9） |
| `enable-chunked-prefill` | 分块预填充 |
| `tensor-parallel-size` | 多卡张量并行 |

## 5.2 调优命令

```bash
vllm serve model \
  --max-num-seqs 256 \
  --max-num-batched-tokens 8192 \
  --gpu-memory-utilization 0.9 \
  --enable-chunked-prefill
```

## 5.3 指标权衡

### 5.3.1 延迟与吞吐

```
低延迟（在线对话）          高吞吐（离线批处理）
  ↓ 小 batch                ↓ 大 batch
  max-num-seqs 小            max-num-seqs 大
  TTFT 低、TPS 高           总吞吐高、单请求慢
```

### 5.3.2 chunked prefill

分块预填充把长 prompt 的 prefill 拆成多块与 decode 交错执行，避免长请求阻塞短请求，显著改善 P99 延迟。

### 5.3.3 边界与坑

- `max-num-batched-tokens` 过大导致显存峰值高，可能 OOM。
- 只提高 `max-num-seqs` 而不给足 KV Cache 会触发抢占（preemption），吞吐反而下降。
- 前缀缓存对重复系统提示收益巨大，对无共享前缀的负载几乎无效。

## 5.4 吞吐压测

```bash
# 用内置基准压测
vllm bench throughput \
  --model Qwen/Qwen2.5-7B-Instruct \
  --num-prompts 1000 \
  --request-rate 20
```

关注输出吞吐（tokens/s）、TTFT 与 P99，压测数据用于容量规划。

# 6. 部署形态

结论先行：从单机单卡到多机多卡，部署形态由模型大小与并发量决定；K8s 化是生产标配。

```
单机单卡：小模型
单机多卡：张量并行
多机多卡：Ray + 张量/流水线并行
K8s：容器化 + HPA
```

## 6.1 形态选择

| 形态 | 适用 | 关键参数 |
|------|------|----------|
| 单机单卡 | 7B 及以下 | 无 |
| 单机多卡 | 14B~70B | `tensor-parallel-size` |
| 多机多卡 | 70B 以上 | Ray + TP/PP |
| K8s | 生产集群 | HPA + GPU 调度 |

## 6.2 K8s 部署

```yaml
# K8s 部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          args: ["--model", "Qwen/Qwen2.5-7B-Instruct",
                 "--gpu-memory-utilization", "0.9"]
          resources:
            limits:
              nvidia.com/gpu: 1
```

### 6.2.1 生产要点

- 用 readiness/liveness 探针检查 `/health`。
- GPU 节点用污点与亲和性调度，避免与训练任务抢卡。
- 多副本前置 Service + 负载均衡。

### 6.2.2 边界与坑

- `replicas > 1` 时若共享同一 PVC 缓存模型会拖慢启动，建议预热镜像。
- 张量并行跨节点对网络带宽极敏感，需高速互联（如 InfiniBand）。

### 6.2.3 自动扩缩容

| 指标 | 扩缩容依据 |
|------|-----------|
| 队列长度 | 最直接 |
| GPU 利用率 | 辅助 |
| TTFT P99 | 体验红线 |

GPU 场景下 HPA 需配合自定义指标（如队列长度），仅靠 CPU 指标无效。

# 7. 监控

结论先行：没有监控就没有生产；至少盯住队列、TTFT、TPS、GPU 利用率与显存五项。

## 7.1 核心指标

| 指标 | 说明 |
|------|------|
| 请求队列长度 | 积压情况 |
| TTFT | 首 token 延迟 |
| TPS | 生成速度 |
| GPU 利用率 | 资源使用 |
| 显存占用 | 容量 |

## 7.2 采集

```bash
# vLLM 内置 Prometheus 指标
curl http://localhost:8000/metrics
```

## 7.3 告警阈值

### 7.3.1 建议阈值

| 指标 | 告警线 | 说明 |
|------|--------|------|
| 队列长度 | 持续 > 100 | 需扩容 |
| TTFT P99 | > 2s | 体验下降 |
| GPU 利用率 | < 50% 且队列高 | 配置不当 |
| 显存占用 | > 95% | 有 OOM 风险 |

### 7.3.2 边界与反例

- GPU 利用率高不等于健康：可能是抢占导致的无效重算。
- 只看均值会掩盖长尾，务必看 P95/P99。

### 7.3.3 日志与追踪

- 每个请求打 request-id，串联网关到 vLLM 日志。
- 记录 prompt/completion token 数，便于计费与容量分析。
- 慢请求单独采样，分析是否长 prompt 或抢占导致。

# 8. 与微调模型结合

结论先行：vLLM 支持运行时挂载 LoRA 适配器，一份基座可服务多个微调变体，显著节省显存。

## 8.1 加载 LoRA

```bash
# 部署 LoRA 微调后的模型
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --enable-lora \
  --lora-modules my-lora=/path/to/lora-adapter
```

> vLLM 支持动态加载 LoRA 适配器。

## 8.2 多适配器

### 8.2.1 使用方式

请求时把 `model` 字段填 LoRA 名即可路由到对应适配器：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"my-lora","messages":[{"role":"user","content":"你好"}]}'
```

### 8.2.2 边界与坑

- LoRA 适配器的基座必须与 `vllm serve` 的基座完全一致。
- 适配器数量多时显存与调度开销上升，需评估 `max-loras` 上限。
- 已合并进权重的模型无需 LoRA 参数，直接 serve 即可。

### 8.2.3 适配器热更新

部分版本支持运行时加载/卸载 LoRA，无需重启服务，适合多租户快速上线新微调版本。

# 9. 生产注意事项

结论先行：上线前把显存、监控、冗余、限流、版本五件事逐一确认。

| 风险 | 建议 |
|------|------|
| 显存估算不足 | 预留 KV Cache 空间 |
| 无监控 | 监控队列与 GPU |
| 单点 | 多副本 + 负载均衡 |
| 无限流 | 网关限流 |
| 版本不兼容 | 锁定版本 |

## 9.1 容量估算

```
KV Cache ≈ 2 × 层数 × 头数 × 头维 × 序列长度 × 并发数 × 精度字节
```

显存预算 = 权重 + KV Cache + 激活 + 框架开销，务必留 10% 余量。

## 9.2 上线清单

```
□ 显存预算含 KV Cache 与余量
□ Prometheus 指标已接入告警
□ 多副本 + LB，无单点
□ 网关限流与鉴权
□ vLLM 与 CUDA 版本已锁定
□ 模型与镜像已预热
```

## 9.3 故障预案

```
OOM          → 降 max-model-len / 降并发 / 用量化
启动失败     → 检查显存余量与版本
请求超时     → 查队列、降并发、扩副本
输出乱码     → 检查 tokenizer / 模型版本
服务无响应   → 探针重启 + 查看 GPU 状态
```

# 10. 小结

结论先行：vLLM 是高吞吐生产推理的默认选择，核心价值在于把 GPU 利用率与接入成本同时做到最优。

## 10.1 关键结论

- vLLM 用 PagedAttention + 连续批处理实现高吞吐。
- 提供 OpenAI 兼容 API，可直接用 Spring AI/LangChain4j 接入。
- 支持量化（AWQ/GPTQ）降低显存。

## 10.2 实践要点

- 支持张量并行与 LoRA 动态加载。
- 生产需监控队列、TTFT、GPU 利用率。
