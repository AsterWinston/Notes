# 1. 模块定位

Agent 的调用链长、非确定性高，出了问题"看不见"是最大痛点。本模块讲如何用 Trace / Metrics / Logs 让 Agent 变得可观测、可调试。

## 1.1 为什么需要可观测性

普通服务出问题，看堆栈和日志基本能定位。Agent 系统不同：

- 一次请求会经过多轮模型调用、检索与工具调用；
- 输出是概率采样的结果，同样输入可能不同输出；
- 失败表现往往只是"答案不对"，看不出发生在哪一环。

没有可观测性，调试就退化为猜测。可观测性把"看不见"变成"可回放、可归因、可量化"。

### 1.1.1 三大支柱的分工

| 支柱 | 回答的问题 | Agent 场景重点 |
|------|------------|----------------|
| Trace（追踪） | 这一步做了什么 | 推理 / 工具 / 检索 Span |
| Metrics（指标） | 整体是否健康 | 延迟、Token、成功率 |
| Logs（日志） | 当时发生了什么 | 结构化 + 脱敏 |

### 1.1.2 与普通服务观测的差异

| 维度 | 普通服务 | Agent 系统 |
|------|----------|------------|
| 调用单位 | 接口 / SQL | 模型 / 工具 / 检索 |
| 成本维度 | CPU / 内存 | Token / 费用 |
| 输出特性 | 确定 | 概率采样 |
| 语义 | 状态码 | 语义正确性 |

## 1.2 本模块解决什么问题

| 问题 | 本模块的答案 |
|------|--------------|
| 不知道模型看到了什么 | 结构化记录 Prompt 与上下文 |
| 不知道慢在哪一步 | Trace / Span 记录各步耗时 |
| 不知道花了多少钱 | Token 与成本作为一等指标 |
| 不知道质量为何下降 | 观测数据 + 评测联动 |
| 线上问题无法复现 | 基于 traceId 的完整回放 |

# 2. 学习目标

## 2.1 能力目标

- 理解 LLM 应用可观测性的特殊性（Token、成本、多步调用）。
- 掌握 Agent 的 Trace 与 Span 设计。
- 能用 Micrometer / OpenTelemetry / Langfuse 等落地。
- 能基于观测数据定位性能与质量问题。

## 2.2 前置知识与关联能力

| 前置能力 | 说明 | 关联模块 |
|----------|------|----------|
| Spring Boot | 依赖注入、Actuator | `9_Frameworks` |
| LLM 调用基础 | Prompt、Token、流式 | `4_LLMEngineering` |
| RAG 基础 | 检索、分块、Embedding | `6_RAG` |
| 微服务基础 | 追踪、日志、指标 | 通用后端 |

## 2.3 学习产出

学完后应能独立完成：

- 为一次 Agent 请求建立完整 Trace，贯穿模型、工具与检索；
- 暴露关键指标并配置成本告警；
- 在故障时用 traceId 回放定位根因；
- 把观测数据回流为评测用例。

# 3. 文件清单

## 3.1 文件与大纲

| 文件 | 大纲 |
|------|------|
| `1_observability_overview.md` | 三大支柱（Trace/Metrics/Logs）在 LLM 场景的落地 |
| `2_tracing.md` | Trace 基础、调用链、Span 模型与上下文传播 |
| `3_agent_trace_and_spans.md` | Agent 专用 Trace：推理步骤、工具调用、记忆操作 |
| `4_logging.md` | 结构化日志、Prompt/响应脱敏记录、日志分级 |
| `5_metrics.md` | 关键指标：延迟、Token、成功率、工具错误率 |
| `6_opentelemetry.md` | OpenTelemetry 语义约定与 Java 接入 |
| `7_langfuse_langsmith.md` | Langfuse / LangSmith 等平台接入与用法 |
| `8_micrometer.md` | Micrometer + Spring Boot Actuator 监控 LLM 应用 |
| `9_cost_and_token_monitoring.md` | 成本与 Token 监控、预算告警 |
| `10_debugging_agents.md` | Agent 调试方法论：复现、回放、根因分析 |

## 3.2 文件间关系

```
概览（1）
   │
   ├── 追踪主线：2 → 3 ─┐
   ├── 记录主线：4      │
   ├── 指标主线：5      ├── 平台与标准：6 → 7 → 8
   └────────────────────┘         │
                                  ▼
                          成本（9）→ 调试（10）
```

## 3.3 阅读建议

- 初次学习：按文件序号顺序阅读；
- 只做落地：优先读 3（Agent Trace）、8（Micrometer）、9（成本）；
- 只排故障：直接读 10（调试方法论）。

# 4. 学习顺序与依赖

## 4.1 推荐学习路径

```
概览（1） → Trace（2→3） → Logs/Metrics（4→5） → 平台与标准（6→8）
      → 成本（9） → 调试（10）
```

先建立"三大支柱"的整体认知，再深入 Trace（Agent 场景最核心），随后补齐日志与指标，最后落到平台接入、成本治理与调试实战。

## 4.2 上下游依赖

- 上游依赖：`9_Frameworks`、`4_LLMEngineering`。
- 下游影响：`12_Evaluation`（数据来源）、`14_ProductionEngineering`（运维）。

可观测性输出的轨迹数据，是评测集的重要来源；而生产工程的监控告警，则直接建立在本模块的指标之上。

## 4.3 学习节奏建议

| 阶段 | 内容 | 建议占比 |
|------|------|----------|
| 认知 | 概览 + Trace 概念 | 30% |
| 动手 | 接入 OTel / Micrometer | 40% |
| 治理 | 成本与告警 | 20% |
| 实战 | 调试与回放 | 10% |

# 5. 最小可观测示例

本模块所有内容都围绕一件事：**让每一步都可被记录、聚合与回放**。下面给出三个最小可运行的骨架，帮助建立直观印象。

## 5.1 Trace 埋点最小示例

```java
public record Span(
        String name,        // Span 名：retrieve / plan / tool / generate
        long startNanos,
        long endNanos,
        Map<String, String> tags
) {
    public long durationMs() {
        return (endNanos - startNanos) / 1_000_000;
    }
}
```

```java
public class SimpleTracer {

    // 一次请求 = 一条 Trace，多个步骤 = 多个 Span
    public Span start(String name) {
        return new Span(name, System.nanoTime(), 0, new HashMap<>());
    }

    public void end(Span span) {
        span.endNanos();
        log.info("span={} durationMs={} tags={}",
                span.name(), span.durationMs(), span.tags());
    }
}
```

## 5.2 指标暴露最小示例

```java
@Component
public class LlmMetrics {

    private final Counter calls;
    private final Timer latency;
    private final Counter tokens;

    public LlmMetrics(MeterRegistry registry) {
        this.calls = Counter.builder("llm.calls")
                .description("模型调用次数")
                .register(registry);
        this.latency = Timer.builder("llm.latency")
                .description("模型调用耗时")
                .register(registry);
        this.tokens = Counter.builder("llm.tokens")
                .description("Token 消耗总量")
                .register(registry);
    }

    public void record(long elapsedMs, int usedTokens) {
        calls.increment();
        latency.record(elapsedMs, TimeUnit.MILLISECONDS);
        tokens.increment(usedTokens);
    }
}
```

## 5.3 日志最小示例

```yaml
# logback-spring.xml 片段：输出结构化日志并带上 traceId
logging:
  pattern:
    console: "%d{ISO8601} [%thread] %-5level traceId=%X{traceId} %logger{36} - %msg%n"
```

```java
// 在入口处把 traceId 放入 MDC，全链路日志自动携带
MDC.put("traceId", traceId);
try {
    handle(request);
} finally {
    MDC.remove("traceId");
}
```

## 5.4 三者协同

```
一次 Agent 请求
   │
   ├─ Trace：记录每步的输入/输出/耗时
   ├─ Metrics：聚合成功率、延迟、Token
   └─ Logs：记录异常与关键事件
```

| 排查需求 | 首选支柱 |
|----------|----------|
| 哪一步慢 | Trace |
| 整体是否恶化 | Metrics |
| 具体报错内容 | Logs |
| 成本是否超预算 | Metrics |

三者并非割裂：Trace 提供下钻到单次请求的能力，Metrics 提供整体健康视图，Logs 提供事件细节。实践中通常先用 Metrics 发现异常，再用 Trace 定位环节，最后用 Logs 确认细节。

# 6. 关键知识点速览

## 6.1 核心概念

- Agent 的一次请求 = 一条 Trace，多个 LLM/工具调用 = 多个 Span。
- Token 与成本必须作为一等指标监控。
- Trace 上下文传播是多 Agent / 多服务场景的难点。
- 可观测性不是事后补，而是设计时就要埋点。
- 调试 Agent 的关键是"能回放完整轨迹"。

## 6.2 三大支柱对照

| 支柱 | 回答的问题 | Agent 场景重点 |
|------|------------|----------------|
| Trace | 这一步做了什么 | 推理 / 工具 / 检索 Span |
| Metrics | 整体是否健康 | 延迟、Token、成功率 |
| Logs | 当时发生了什么 | 结构化 + 脱敏 |

## 6.3 常见误区

| 误区 | 纠正 |
|------|------|
| 只记日志不建 Trace | 日志无法还原调用链 |
| 不记录 Prompt 版本 | 无法复现与对比 |
| Token 不作为指标 | 成本失控且无预警 |
| 事后补埋点 | 缺失关键历史数据 |
| 全量明文记录 | 泄露敏感信息、存储膨胀 |
