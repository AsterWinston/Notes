# 1. 为什么用 Ollama

结论先行：Ollama 把"下载模型、量化、加载、开服务"四步压成一条命令，是本地跑大模型最省心的方案，代价是吞吐与并发能力有限。

```
Ollama 让本地跑大模型变得极简：
  ollama pull qwen2.5:7b
  ollama run qwen2.5:7b
```

## 1.1 核心优势

| 优点 | 说明 |
|------|------|
| 极简 | 一条命令 |
| 跨平台 | Win/Mac/Linux |
| 模型库 | 丰富的现成模型 |
| 量化 | 自动选择量化版本 |
| OpenAI 兼容 | 可对接框架 |

## 1.2 工作原理

```
ollama pull → 下载 GGUF（含量化）到本地
ollama run  → 加载权重，起本地推理进程
ollama serve → 常驻服务，暴露 11434 端口
```

### 1.2.1 与直接加载 GGUF 的区别

Ollama 内部封装了 llama.cpp 推理后端，用户无需关心量化格式、线程数、显存/内存分配，默认参数即可运行。

### 1.2.2 边界与反例

- Ollama 面向单用户、低并发场景，不做连续批处理优化。
- 首次 `run` 会触发加载，冷启动有明显延迟，之后常驻内存。

### 1.2.3 安全边界

- 默认仅本机可访问，暴露到公网前必须加鉴权与限流。
- 模型下载来自官方仓库，导入第三方 GGUF 需自行校验来源。

# 2. 安装与使用

结论先行：安装后所有操作围绕 `pull / run / list / rm` 四条命令展开。

## 2.1 基础命令

```bash
# 安装后拉取模型
ollama pull qwen2.5:7b
ollama pull llama3.1:8b
ollama pull deepseek-r1:7b

# 运行
ollama run qwen2.5:7b

# 列出已下载
ollama list

# 删除
ollama rm qwen2.5:7b
```

## 2.2 服务管理

| 命令 | 作用 |
|------|------|
| `ollama serve` | 启动常驻服务（默认 11434） |
| `ollama ps` | 查看当前加载的模型 |
| `ollama show <model>` | 查看模型详情 |
| `ollama cp` | 复制模型 |
| `ollama push` | 推送到仓库 |

### 2.2.1 环境变量

```bash
# 允许外部访问
set OLLAMA_HOST=0.0.0.0:11434

# 修改模型存储目录
set OLLAMA_MODELS=D:\ollama\models

# 保持模型常驻显存
set OLLAMA_KEEP_ALIVE=24h
```

### 2.2.2 边界与坑

- 默认只监听 127.0.0.1，局域网访问需设置 `OLLAMA_HOST`。
- `OLLAMA_KEEP_ALIVE` 默认为 5 分钟，超时会卸载模型导致下次请求冷启动。

### 2.2.3 命令速查

| 需求 | 命令 |
|------|------|
| 查看模型信息 | `ollama show qwen2.5:7b` |
| 查看运行中模型 | `ollama ps` |
| 复制并改名 | `ollama cp a b` |
| 从文件创建 | `ollama create x -f Modelfile` |

# 3. 模型选择

结论先行：按显存选参数量，显存不足会退化为 CPU 推理，速度骤降。

## 3.1 显存对照

| 模型 | 参数量 | 显存需求 |
|------|:------:|:--------:|
| qwen2.5:1.5b | 1.5B | ~2GB |
| qwen2.5:7b | 7B | ~6GB |
| qwen2.5:14b | 14B | ~10GB |
| qwen2.5:32b | 32B | ~20GB |
| llama3.1:8b | 8B | ~6GB |

> 显存不足时会用 CPU（很慢）。

## 3.2 量化标签

| 标签 | 量化 | 显存 | 质量 |
|------|------|:----:|:----:|
| 默认 | Q4_0/Q4_K_M | 低 | 略降 |
| `:q8_0` | 8bit | 高 | 接近原版 |
| `:fp16` | 16bit | 最高 | 最好 |

### 3.2.1 选型建议

- 显存刚好够用默认 Q4；追求质量且有富余选 q8。
- 中文任务优先 Qwen 系列，推理任务可选 deepseek-r1。

### 3.2.2 边界与反例

- 标称"显存需求"是加载权重的最小值，长上下文还需要额外 KV Cache。
- 显存差一点点就会整模型落 CPU，速度可能慢几十倍。

### 3.2.3 上下文与显存

```
显存 ≈ 权重 + KV Cache
KV Cache 随 num_ctx 与并发线性增长
```

| num_ctx | 额外显存 | 说明 |
|---------|----------|------|
| 2048 | 低 | 短对话 |
| 8192 | 中 | 常规 |
| 32768 | 高 | 长文档，易落 CPU |

# 4. Java 接入

结论先行：Spring AI 提供 Ollama 原生 Starter，也支持 OpenAI 兼容方式，两种都能无缝对接。

## 4.1 Spring AI 配置

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: qwen2.5:7b
```

## 4.2 原生客户端

```java
ChatModel model = OllamaChatModel.builder()
        .baseUrl("http://localhost:11434")
        .modelName("qwen2.5:7b")
        .temperature(0.7)
        .build();
```

## 4.3 两种接入方式对比

| 方式 | 优点 | 缺点 |
|------|------|------|
| Ollama Starter | 功能完整、原生 | 只能连 Ollama |
| OpenAI 兼容 | 可换后端 | 部分高级参数缺失 |

### 4.3.1 边界与坑

- 流式输出在 Ollama 原生协议与 OpenAI 协议下事件格式不同，切换时需调整解析。
- `temperature` 等参数在服务端 Modelfile 中也可设定，请求参数会覆盖服务端默认。

### 4.3.2 流式示例

```java
Flux<String> stream = chatModel.stream(
        new Prompt("用一句话解释什么是向量数据库"));
stream.subscribe(System.out::print);
```

# 5. OpenAI 兼容模式

结论先行：Ollama 自带 `/v1` 兼容端点，可直接复用任何 OpenAI 客户端。

## 5.1 直接调用

```bash
# Ollama 提供 OpenAI 兼容接口
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen2.5:7b","messages":[{"role":"user","content":"你好"}]}'
```

## 5.2 Spring AI 接入

```yaml
spring:
  ai:
    openai:
      base-url: http://localhost:11434/v1
      api-key: dummy
      chat:
        options:
          model: qwen2.5:7b
```

## 5.3 端点对照

| 端点 | 协议 |
|------|------|
| `/api/chat` | Ollama 原生 |
| `/api/generate` | Ollama 原生补全 |
| `/v1/chat/completions` | OpenAI 兼容 |
| `/v1/embeddings` | OpenAI 兼容嵌入 |

### 5.3.1 边界与坑

- 注意 OpenAI 兼容路径需带 `/v1`，而 Ollama 原生不带。
- 兼容层对 `tools`、`response_format` 等高级字段支持滞后，需以实际版本为准。

### 5.3.2 嵌入与工具调用

| 能力 | 端点 | 备注 |
|------|------|------|
| 文本嵌入 | `/v1/embeddings` | 需嵌入模型 |
| 工具调用 | `/v1/chat/completions` | 依赖模型与模板支持 |
| 视觉输入 | `/v1/chat/completions` | 需多模态模型 |

# 6. 自定义模型（Modelfile）

结论先行：Modelfile 是 Ollama 的"Dockerfile"，用于固化基座、参数与系统提示，生成可复用模型。

## 6.1 基本写法

```dockerfile
FROM qwen2.5:7b

PARAMETER temperature 0.3
PARAMETER num_ctx 8192

SYSTEM """
你是一个专业的中文技术助手，回答简洁、准确。
"""
```

```bash
ollama create my-assistant -f Modelfile
ollama run my-assistant
```

> 用 Modelfile 可以定制系统提示与参数。

## 6.2 常用指令

| 指令 | 作用 |
|------|------|
| `FROM` | 指定基座模型或 GGUF 文件 |
| `SYSTEM` | 系统提示词 |
| `PARAMETER` | 推理参数（temperature/num_ctx 等） |
| `TEMPLATE` | 对话模板 |
| `ADAPTER` | 挂载 LoRA 适配器 |

### 6.2.1 参数说明

| 参数 | 说明 | 建议 |
|------|------|------|
| `temperature` | 采样温度 | 0.2~0.7 |
| `num_ctx` | 上下文窗口 | 按显存设 |
| `top_p` | 核采样 | 0.9 |
| `repeat_penalty` | 重复惩罚 | 1.1 |

### 6.2.2 边界与坑

- `num_ctx` 增大直接抬高 KV Cache 显存，设置过大会 OOM 或落 CPU。
- Modelfile 中 `SYSTEM` 只在模板未包含系统位时生效，需确认基座模板。

### 6.2.3 模板定制

```
TEMPLATE """{{ if .System }}<|im_start|>system
{{ .System }}<|im_end|>
{{ end }}<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""
```

模板必须与基座训练格式一致，否则指令遵循能力下降。

# 7. 微调模型部署

结论先行：Ollama 只认 GGUF 格式，微调后的模型需先转换为 GGUF 再导入。

## 7.1 导入流程

```bash
# 导入 GGUF 格式的微调模型
ollama create my-model -f Modelfile

# Modelfile
# FROM ./my-finetuned-model.gguf
```

> Ollama 支持 GGUF 格式，微调后需转换为 GGUF。

## 7.2 转换链路

```
LoRA/全量微调产物
   ↓ merge 到基座（可选）
HF 格式（safetensors）
   ↓ llama.cpp convert_hf_to_gguf.py
GGUF（fp16）
   ↓ 量化（可选）
GGUF（q4_k_m 等）
   ↓ ollama create
Ollama 模型
```

### 7.2.1 挂载 LoRA

```dockerfile
FROM qwen2.5:7b
ADAPTER ./my-lora-adapter
```

### 7.2.2 边界与坑

- 转换脚本版本必须与 llama.cpp 对齐，否则量化报错。
- 分词器（tokenizer）不一致会导致输出乱码，需用原基座词表。
- 量化会损失精度，代码/数学微调模型建议用 q8 或 fp16。

### 7.2.3 全量微调 vs LoRA

| 方式 | 产物 | 导入方式 |
|------|------|----------|
| 全量微调 | 完整权重 | merge 后转 GGUF |
| LoRA | 适配器 | merge 或 `ADAPTER` |

> 未 merge 的 LoRA 需基座与适配器同时存在，且模板一致。

# 8. 适用场景

结论先行：Ollama 面向本地开发、隐私敏感与离线场景；高并发生产应交给 vLLM。

## 8.1 场景匹配

| 适用 | 不适用 |
|------|--------|
| 本地开发 | 高并发生产 |
| 隐私敏感 | 大规模部署 |
| 离线环境 | 需要顶级能力 |
| 原型验证 | 低延迟要求 |
| 个人使用 | 多用户并发 |

## 8.2 决策建议

```
并发 < 5 且单机？
 ├── 是 → Ollama
 └── 否 → 需要高吞吐/多副本？
          ├── 是 → vLLM
          └── 否 → Ollama + 队列
```

### 8.2.1 边界与反例

- 用 Ollama 承载对外 API 且并发一高，请求排队严重、延迟飙升。
- 需要 GPU 集群多副本时，Ollama 缺乏成熟的分布式调度。

### 8.2.2 混合架构

```
开发机：Ollama（快速验证）
        ↓ 接口一致
生产集群：vLLM（高吞吐）
```

统一使用 OpenAI 兼容协议，可在两者间平滑迁移。

# 9. 与 vLLM 对比

结论先行：两者定位不同——Ollama 胜在易用与本地，vLLM 胜在吞吐与生产。

| 维度 | Ollama | vLLM |
|------|:------:|:----:|
| 易用性 | 极高 | 中 |
| 吞吐 | 低 | 高 |
| 并发 | 弱 | 强 |
| 部署 | 单机 | 集群 |
| 适用 | 本地/开发 | 生产 |

## 9.1 技术差异

| 技术点 | Ollama | vLLM |
|--------|:------:|:----:|
| 后端 | llama.cpp | 自研 + CUDA |
| KV Cache | 连续 | PagedAttention |
| 批处理 | 有限 | 连续批处理 |
| 量化 | GGUF | AWQ/GPTQ/FP8 |

## 9.2 选型结论

### 9.2.1 组合使用

开发期用 Ollama 快速验证 Prompt 与 Modelfile，上线时把同一模型切到 vLLM，接口都是 OpenAI 兼容，切换成本低。

### 9.2.2 边界与反例

- 不能简单以"参数量"对比两者性能，后端与批处理策略差异远大于模型本身。
- Ollama 的量化格式与 vLLM 不通用，需分别准备权重。

### 9.2.3 迁移路径

- 保留 Modelfile 与 Prompt，迁移到 vLLM 时行为尽量一致。
- 注意量化格式不同，需重新准备 vLLM 可用的权重。
- 重新压测，不要沿用 Ollama 的并发参数。

# 10. 生产注意事项

结论先行：即便用于生产，也必须解决并发、显存、监控、单点与版本管理五个问题。

| 风险 | 建议 |
|------|------|
| 并发能力弱 | 高并发用 vLLM |
| 显存不足 | 用更小/量化模型 |
| 无监控 | 加监控 |
| 单点 | 多实例 + 负载均衡 |
| 模型管理 | 版本化 |

## 10.1 监控

```
/health            → 存活探针
/api/ps            → 已加载模型
日志中的 load/unload → 冷启动分析
系统层 nvidia-smi  → 显存/利用率
```

## 10.2 上线清单

```
□ 并发评估，超阈值改用 vLLM
□ 显存预留 KV Cache 余量
□ 多实例 + 负载均衡
□ 存活/就绪探针
□ 模型与 Modelfile 版本化
□ 冷启动与 keep-alive 已配置
```

## 10.3 资源隔离

- 与训练任务分离部署，避免抢显存。
- 限制单实例显存，防止 OOM 拖垮整机。
- 多实例前置负载均衡，单点故障可切换。

# 11. 小结

结论先行：Ollama 是本地与隐私场景的首选，一旦进入高并发生产则应交棒给 vLLM。

## 11.1 关键结论

- Ollama 让本地部署大模型极简。
- 一条命令拉取与运行模型。
- 提供 OpenAI 兼容接口，可对接 Spring AI/LangChain4j。

## 11.2 实践要点

- Modelfile 可定制系统提示与参数。
- 适合本地开发与隐私场景，高并发生产用 vLLM。
