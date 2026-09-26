# 1. 为什么用 LLM 专用平台

**结论先行**：通用可观测栈（Jaeger、Prometheus、Grafana）能回答"服务是否慢、是否报错"，却难以回答"模型看到了什么提示、答得对不对、这次调用花了多少钱"。LLM 专用平台（LLMOps Platform）正是为后一类问题而生。

## 1.1 通用可观测栈的盲区

分布式追踪与指标系统以"服务/接口"为核心抽象，而 LLM 应用的核心抽象是"一次推理"。两者的错位导致：

| 需求 | 通用平台 | LLM 平台 |
|------|:--------:|:--------:|
| 查看完整提示 | 难 | 原生 |
| Token 与成本 | 需自建 | 内置 |
| 工具调用可视化 | 难 | 内置 |
| 评测集成 | 无 | 内置 |
| Prompt 管理 | 无 | 内置 |

通用平台要展示提示，只能把提示塞进 span 的 attribute（属性），随后遇到三个问题：属性体积上限、隐私合规、检索困难。

## 1.2 LLM 专用平台补足的能力

### 1.2.1 提示级可见性

一次模型调用被拆成输入（提示、系统消息、工具定义）、输出（补全、工具调用、结束原因）与元数据（模型、参数、Token）。平台以结构化方式存储，支持按内容检索与逐条回放。

### 1.2.2 成本与 Token 归因

平台内置各厂商定价表，自动把 Token 换算为成本，并支持按用户、会话、租户、标签分组聚合。这是自建方案最容易低估的工作量——定价表需要持续维护。

### 1.2.3 评测闭环

线上 Trace 可一键沉淀为数据集（Dataset），再接入自动评测器（Evaluator）或人工标注（Annotation），形成"线上问题 → 离线评测 → 回归验证"的闭环。

## 1.3 选型坐标

| 维度 | 自托管开源 | SaaS |
|------|-----------|------|
| 数据主权 | 高 | 依赖厂商 |
| 运维成本 | 高 | 低 |
| 上手速度 | 慢 | 快 |
| 生态集成 | 需自行打通 | 开箱即用 |
| 成本模型 | 服务器固定成本 | 按量/席位 |

选择时先明确：数据能否出境、团队是否有运维能力、是否需要深度定制。

# 2. Langfuse

**开源、可自托管**，功能覆盖 Trace / Prompt 管理 / 评测 / 数据集。

| 功能 | 说明 |
|------|------|
| Trace | 完整调用链（含提示与响应） |
| Sessions | 会话视图 |
| Prompt Management | 提示版本管理 |
| Datasets & Evaluation | 评测集与自动评测 |
| Scores | 人工/自动打分 |
| Cost Tracking | Token 与成本 |

## 2.1 核心数据模型

### 2.1.1 Trace / Observation / Span

Langfuse 的层次为：`Trace`（一次端到端请求）→ `Observation`（一次操作）。Observation 又分为 `SPAN`（有耗时的普通步骤）、`GENERATION`（一次模型调用，含 Token 与成本）、`EVENT`（瞬时事件）。

```
Trace
 ├── SPAN  rag.retrieval
 ├── SPAN  tool.search
 └── GENERATION  chat (model, tokens, cost)
```

### 2.1.2 Session 与 User

`sessionId` 把多轮对话串成会话视图；`userId` 用于按用户聚合成本与质量。两者都是可选的字符串标识，建议使用不可逆的哈希而非明文账号。

### 2.1.3 Score 与 Dataset

`Score` 可来自人工、模型或规则（如"是否有害"、"是否命中事实"）；`Dataset` 是带预期输出的样本集合，用于回归评测。

## 2.2 部署

### 2.2.1 Docker Compose 自托管

```bash
# Docker Compose 自托管
docker compose up -d
# 访问 http://localhost:3000
```

默认 Compose 会拉起 Web、Postgres、ClickHouse、Redis、MinIO 等组件。生产环境务必替换默认密钥并启用 HTTPS。

### 2.2.2 生产部署要点

| 组件 | 作用 | 注意 |
|------|------|------|
| Postgres | 元数据 | 定期备份 |
| ClickHouse | Trace 存储 | 磁盘容量与保留期 |
| Redis | 队列/缓存 | 高可用 |
| MinIO/S3 | 大对象 | 生命周期策略 |

建议将 Trace 存储与业务数据库物理隔离，避免可观测负载影响主业务。

## 2.3 Java 集成思路

Langfuse 提供 HTTP API 与 OTel 接入。前者直接、可控；后者复用现有链路，适合已经用 OTel 的系统。

### 2.3.1 HTTP API 上报

```java
@Component
public class LangfuseReporter {

    private final LangfuseClient client;

    public void reportTrace(String traceId, String userId,
                            String input, String output,
                            Usage usage, BigDecimal cost) {
        client.trace(TraceRequest.builder()
                .id(traceId)
                .userId(userId)
                .input(input)
                .output(output)
                .metadata(Map.of(
                        "inputTokens", usage.getPromptTokens(),
                        "outputTokens", usage.getCompletionTokens(),
                        "cost", cost))
                .build());
    }
}
```

### 2.3.2 OTLP 上报

或通过 OTLP 上报，复用 OTel 链路：

```yaml
otel:
  exporter:
    otlp:
      endpoint: http://langfuse:4318
```

Langfuse 会识别 OTel 的 GenAI 语义约定（Semantic Conventions），把 `gen_ai.*` 属性映射为模型名、Token 等字段。

### 2.3.3 异步与批量

上报绝不能阻塞主流程。推荐做法：

```java
// 伪代码：有界队列 + 后台批量发送
public class AsyncTraceReporter {

    private final BlockingQueue<LlmTracePayload> queue =
            new ArrayBlockingQueue<>(10_000);

    public void submit(LlmTracePayload payload) {
        if (!queue.offer(payload)) {
            dropped.increment();
            return;
        }
    }

    @Scheduled(fixedDelay = 1000)
    void flush() {
        List<LlmTracePayload> batch = new ArrayList<>();
        queue.drainTo(batch, 200);
        if (!batch.isEmpty()) {
            client.batch(batch);
        }
    }
}
```

队列必须有界，溢出时丢弃并计数，绝不能因上报拖垮业务。

# 3. LangSmith

LangChain 官方平台，SaaS 为主。

| 功能 | 说明 |
|------|------|
| Trace | 调用链可视化 |
| Evaluation | 评测与对比 |
| Datasets | 测试集管理 |
| Monitoring | 生产监控 |

## 3.1 定位与能力

LangSmith 与 LangChain/LangGraph 生态耦合最深，若技术栈以 Python LangChain 为主，体验最顺滑。其评测（Evaluation）能力较成熟，支持多评分器与实验对比。

## 3.2 Java 接入方式

### 3.2.1 OTel 接入

```java
// 通过 OTel 或 HTTP API 上报
// LangSmith 主要面向 LangChain 生态
```

Java 端无官方一等 SDK，通常通过标准 OTLP 导出，或直接调用其 REST API。

### 3.2.2 HTTP API 上报

```java
// 伪代码：把 Langfuse 风格载荷映射为 LangSmith Run
public void reportRun(LlmTracePayload p) {
    RunCreate run = RunCreate.builder()
            .id(p.traceId())
            .name(p.name())
            .runType("llm")
            .startTime(p.startTime())
            .endTime(p.endTime())
            .inputs(Map.of("prompt", p.input()))
            .outputs(Map.of("completion", p.output()))
            .extra(Map.of("tokens", p.inputTokens() + p.outputTokens()))
            .build();
    langsmithClient.createRun(run);
}
```

## 3.3 与 Langfuse 对比

| 维度 | Langfuse | LangSmith |
|------|----------|-----------|
| 开源 | 是 | 否 |
| 自托管 | 支持 | 有限 |
| 生态 | 语言无关 | LangChain 优先 |
| 评测 | 内置 | 强 |
| 数据主权 | 高 | 依赖厂商 |

若需要数据主权与自托管，Langfuse 更合适；若深度使用 LangChain 且接受 SaaS，LangSmith 更省心。

# 4. 自建轻量方案

若不想引入平台，可用 OTel + Grafana 自建：

```
应用 → OTel Collector → Tempo（Trace）
                      → Prometheus（Metrics）
                      → Loki（Logs）
                              │
                              ▼
                          Grafana（统一看板）
```

## 4.1 架构

应用侧统一使用 OpenTelemetry SDK 埋点，通过 OTLP 发给 Collector；Collector 按信号（signal）分流到后端。Trace 中通过 span attribute 携带提示摘要与 Token。

## 4.2 组件职责

| 组件 | 作用 |
|------|------|
| Tempo | Trace 存储与查询 |
| Prometheus | 指标 |
| Loki | 日志 |
| Grafana | 可视化与告警 |

## 4.3 取舍

> 自建灵活但工作量大；LLM 专用平台开箱即用但需接受其数据模型。

自建的额外工作包括：提示查看 UI、成本定价表、评测框架、数据集管理。这些正是专用平台的核心价值。

# 5. 上报内容设计

**结论先行**：上报载荷应覆盖"谁、何时、用什么模型、花了多少、看到什么、结果如何"，同时默认不含原始敏感文本。

## 5.1 上报载荷

```java
public record LlmTracePayload(
        String traceId,
        String sessionId,
        String userId,
        String tenantId,
        String name,
        Instant startTime,
        Instant endTime,
        String model,
        int inputTokens,
        int outputTokens,
        BigDecimal cost,
        String input,              // 脱敏后
        String output,             // 脱敏后
        List<ToolCallTrace> tools,
        String status,
        String errorMessage
) {}
```

| 字段 | 用途 | 是否必需 |
|------|------|:--------:|
| traceId | 与日志/指标关联 | 是 |
| sessionId | 会话回放 | 否 |
| userId/tenantId | 归因与计费 | 否 |
| model | 选型与成本 | 是 |
| inputTokens/outputTokens | 成本计算 | 是 |
| cost | 直接聚合 | 是 |
| input/output | 调试与评测 | 否（脱敏） |
| tools | 工具链路 | 否 |
| status/errorMessage | 失败分析 | 是 |

## 5.2 脱敏

### 5.2.1 脱敏策略

| 策略 | 说明 | 适用 |
|------|------|------|
| 全量哈希 | 不可逆 | 仅统计 |
| 正则掩码 | 保留结构 | 手机号/邮箱 |
| 实体识别 | NER 替换 | 姓名/地址 |
| 截断 | 保留前缀 | 长文本 |
| 丢弃 | 不采集 | 高敏 |

### 5.2.2 实现

```java
@Component
public class Redactor {

    private static final Pattern PHONE =
            Pattern.compile("1[3-9]\\d{9}");
    private static final Pattern EMAIL =
            Pattern.compile("[\\w.+-]+@[\\w-]+\\.[\\w.]+");

    public String redact(String text) {
        if (text == null) {
            return null;
        }
        String masked = PHONE.matcher(text).replaceAll("***");
        masked = EMAIL.matcher(masked).replaceAll("***");
        return masked.length() > 4000 ? masked.substring(0, 4000) : masked;
    }
}
```

> **上报前必须脱敏**，即使平台是自托管的。

## 5.3 采样与批量

| 场景 | 策略 |
|------|------|
| 错误请求 | 全量上报 |
| 正常请求 | 按比例采样（如 10%） |
| 高价值会话 | 白名单全量 |
| 高峰 | 降低采样率 |

采样要在业务侧完成，避免无效网络开销；同时保证错误与慢请求优先保留。

## 5.4 载荷演进

上报载荷是跨团队契约，字段一旦被看板或计费依赖就不能随意删除。建议：

| 变更类型 | 兼容性 | 处理方式 |
|----------|:------:|----------|
| 新增可选字段 | 兼容 | 直接上线 |
| 字段改名 | 破坏 | 双写过渡 |
| 删除字段 | 破坏 | 先停用再删 |
| 语义变更 | 破坏 | 新增字段 |

在载荷中预留 `schemaVersion`，便于平台侧做兼容解析。

# 6. 平台能力的典型用法

## 6.1 查看失败案例

```
按 status=ERROR 过滤 → 查看完整提示与响应
→ 定位是检索没命中还是模型答错
```

## 6.2 成本分析

```
按 model / tenant / user 分组 → 找出高消耗来源
```

### 6.2.1 分组维度

| 分组 | 回答的问题 |
|------|-----------|
| model | 是否该换小模型 |
| tenant | 哪个客户在烧钱 |
| user | 是否有人滥用 |
| feature | 哪个功能成本高 |

### 6.2.2 预算联动

成本看板应与预算系统打通：当某租户接近阈值时自动预警，超限时触发降级或拒绝。

## 6.3 Prompt 对比

```
A/B 两组 Trace 对比：成功率、Token、延迟、人工评分
```

## 6.4 数据集回流

```
把线上失败案例加入数据集 → 用于评测与回归
```

### 6.4.1 样本构造

回流样本应包含输入、期望输出与失败原因标签；同一失败模式保留少量代表性样本即可，避免数据集被同类样本淹没。

# 7. 生产注意事项

## 7.1 风险清单

| 风险 | 建议 |
|------|------|
| 上报原始 PII | 必须脱敏 |
| 平台单点 | 自托管集群 / 降级 |
| 上报阻塞主流程 | 异步上报 |
| 全量上报 | 采样 |
| 无保留策略 | 设置数据保留期 |
| 数据出境 | 用自托管/境内 |

## 7.2 数据治理

- 明确保留期，Trace 与原始文本分离存储。
- 对用户提供数据删除通道（被遗忘权）。
- 审计谁可以查看原始提示。

## 7.3 可用性与降级

```java
// 伪代码：上报失败不影响业务
try {
    reporter.report(payload);
} catch (Exception e) {
    log.debug("trace 上报失败，已忽略", e);
    metrics.counter("trace.report.failure").increment();
}
```

上报链路应视为"尽力而为"的旁路，任何异常都不得向上传播。

## 7.4 成本与容量

全量上报在高并发下会产生可观的存储与网络成本。上线前应估算：

| 变量 | 说明 |
|------|------|
| QPS | 每秒模型调用量 |
| 平均载荷大小 | 脱敏后字节数 |
| 采样率 | 实际上报比例 |
| 保留期 | 存储天数 |

存储量约等于 `QPS × 载荷大小 × 采样率 × 86400 × 保留天数`。据此决定自托管集群的磁盘规格。

# 8. 小结

- LLM 专用平台提供提示查看、Token 成本、工具可视化、评测等能力。
- Langfuse 开源可自托管，功能全面；LangSmith 与 LangChain 生态耦合深。
- 可通过 OTel/HTTP 从 Java 上报，上报链路必须异步且有界。
- 自建方案用 OTel + Grafana 栈，灵活但工作量大。
- 上报前必须脱敏，且异步上报不阻塞主流程。
- 采样、保留期与数据主权是上线前必须明确的三件事。
