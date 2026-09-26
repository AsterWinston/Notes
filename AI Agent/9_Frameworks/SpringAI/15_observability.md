# 1. 为什么 LLM 应用更需要可观测

**结论先行**：LLM（Large Language Model，大语言模型）应用是非确定性的分布式系统，传统日志与监控不足以定位问题；可观测（Observability）是调试、控本、保稳定的前提。

## 1.1 不确定性与可复现难题

传统服务是确定性的，LLM 应用不是：

| 特性 | 后果 |
|------|------|
| 输出非确定 | 同样输入结果不同，难以复现问题 |
| 调用链长 | 一次请求含多次模型/工具/检索调用 |
| 成本随 Token 变化 | 成本不可预测，需要监控 |
| 延迟波动大 | 受模型与网络影响 |
| 上下文敏感 | 问题可能出在"喂给模型的内容" |

## 1.2 与传统服务的差异

| 维度 | 传统服务 | LLM 应用 |
|------|----------|----------|
| 输出 | 确定 | 概率性 |
| 调用链 | 通常短 | 长且动态分支 |
| 失败模式 | 异常明确 | 静默劣化（答非所问） |
| 成本 | 资源驱动 | Token 驱动 |
| 调试入口 | 断点/日志 | Trace 回放 |

### 1.2.1 静默劣化的危险

最棘手的不是报错，而是"不报错但答错"：检索召回变差、提示被截断、工具返回空结果。这些只能靠 Trace 与指标的组合分析发现。

## 1.3 可观测的收益

- 快速定位：一次请求的每个环节都可回放；
- 成本可控：按租户/用户统计 Token 与费用；
- 质量可评：结合评测集持续回归；
- 风险可查：工具调用与内容审核可审计。

> Spring AI 内置 Observation 机制，天然对接 Micrometer / OpenTelemetry。

# 2. 三大支柱

**结论先行**：可观测由 Trace（追踪）、Metrics（指标）、Logs（日志）三大支柱构成，三者互补而非替代。

```
Trace    一次请求的完整调用链（模型、工具、检索、记忆）
Metrics  延迟、Token、成功率、错误率、成本
Logs     结构化日志（含脱敏后的提示与响应）
```

## 2.1 Trace（追踪）

Trace 描述"一次请求经过了哪些环节、每个环节耗时多少"。它由多个 Span（跨度）组成，Span 之间有父子关系，形成树状调用链。

## 2.2 Metrics（指标）

Metrics 是聚合的数值，用于趋势与告警。它必须低基数（low cardinality），即标签取值数量有限。

## 2.3 Logs（日志）

Logs 是离散的事件记录，信息量最大但检索成本高，适合记录细节与异常。

## 2.4 三者关系

| 支柱 | 粒度 | 用途 | 关键约束 |
|------|------|------|----------|
| Trace | 单请求 | 定位链路瓶颈 | 需 TraceId 贯穿 |
| Metrics | 聚合 | 告警与趋势 | 避免高基数 |
| Logs | 事件 | 细节与取证 | 必须脱敏 |

```
Trace ──(聚合)──▶ Metrics
  │
  └──(关联)──▶ Logs（通过 TraceId 关联）
```

## 2.5 三大支柱的协作流程

**结论先行**：一次故障排查通常遵循"指标发现异常 → Trace 定位环节 → 日志查看细节"的顺序。

```
1) Metrics 告警：ai.chat.duration P99 升高
        │
        ▼
2) Trace 下钻：定位到 retrieval Span 耗时最长
        │
        ▼
3) Logs 取证：该 Span 关联的检索参数与返回条数异常
        │
        ▼
4) 修复与回归：调整 topK / 索引后观察指标回落
```

### 2.5.1 没有 TraceId 的代价

若三支柱之间没有统一的 TraceId，排查会退化为"在日志海里人工比对时间戳"，效率极低。TraceId 是三者协作的粘合剂。

# 3. Spring AI 的 Observation

**结论先行**：Spring AI 通过 Micrometer Observation 为每次模型调用生成观测数据，接入 Actuator 后即可导出指标与追踪。

## 3.1 观测模型

Spring AI 通过 Micrometer Observation 为每次模型调用生成观测数据：

```
observation
├── name: spring.ai.chat.client
├── lowCardinalityKeyValues
│   ├── spring.ai.chat.client.operation = chat
│   └── spring.ai.chat.client.stream = false
└── highCardinalityKeyValues
    ├── spring.ai.chat.client.request.model
    └── spring.ai.chat.client.response.finish_reasons
```

### 3.1.1 低基数与高基数的分工

| 类型 | 是否作指标标签 | 是否入 Trace | 示例 |
|------|----------------|--------------|------|
| 低基数 | 是 | 是 | operation、stream |
| 高基数 | 否 | 是 | model、finish_reasons |

高基数标签若用于指标，会导致时间序列爆炸，务必只放进 Trace。

## 3.2 启用

```yaml
management:
  observations:
    annotations:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

## 3.3 自定义 Observation

```java
@Component
public class LlmObservability {

    private final ObservationRegistry registry;

    public LlmObservability(ObservationRegistry registry) {
        this.registry = registry;
    }

    public <T> T observe(String name, Supplier<T> action) {
        return Observation.createNotStarted(name, registry)
                .lowCardinalityKeyValue("component", "llm")
                .observe(action);
    }
}
```

### 3.3.1 手动埋点场景

当业务逻辑不在 Spring AI 的自动观测范围内（如自研的检索、重排），应手动埋点，保持链路完整。

## 3.4 Advisor 集成

Advisor（顾问）是 Spring AI 的拦截扩展点，适合统一做指标与日志采集：

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(new MetricsAdvisor(meterRegistry),
                new StructuredLoggingAdvisor())
        .build();
```

| Advisor | 职责 | 顺序建议 |
|---------|------|----------|
| 日志 Advisor | 记录请求/响应 | 最外层（order 小） |
| 指标 Advisor | 采集 Token/耗时 | 次外层 |
| 业务 Advisor | 改写提示 | 内层 |

# 4. 关键指标

**结论先行**：LLM 应用的核心指标可归为四类——延迟、Token、成本、错误，围绕这四类建立告警即可覆盖大部分风险。

## 4.1 指标清单

| 指标 | 含义 | 告警建议 |
|------|------|----------|
| `ai.chat.duration` | 模型调用耗时 | P99 超阈值告警 |
| `ai.tokens.prompt` | 输入 Token 数 | 突增告警（防注入/失控） |
| `ai.tokens.completion` | 输出 Token 数 | 突增告警（成本） |
| `ai.cost.total` | 累计成本 | 日预算告警 |
| `ai.tool.calls` | 工具调用次数 | 异常频繁告警 |
| `ai.tool.errors` | 工具错误数 | 错误率告警 |
| `ai.requests.errors` | 调用失败率 | 超阈值告警 |
| `ai.cache.hit_rate` | 缓存命中率 | 过低说明缓存失效 |

## 4.2 用 Advisor 采集 Token 指标

```java
public class MetricsAdvisor implements CallAdvisor {

    private final MeterRegistry registry;

    public MetricsAdvisor(MeterRegistry registry) {
        this.registry = registry;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        Timer.Sample sample = Timer.start(registry);
        try {
            ChatClientResponse resp = chain.nextCall(request);
            Usage usage = resp.chatResponse().getMetadata().getUsage();
            registry.counter("ai.tokens.prompt").increment(usage.getPromptTokens());
            registry.counter("ai.tokens.completion").increment(usage.getCompletionTokens());
            return resp;
        } finally {
            sample.stop(registry.timer("ai.chat.duration"));
        }
    }

    @Override
    public String getName() { return "MetricsAdvisor"; }

    @Override
    public int getOrder() { return 200; }
}
```

### 4.2.1 流式响应的指标采集

流式（stream）响应下，`chatResponse()` 的 Usage 可能在最后一个数据块才可用，需在流结束时汇总，不能在中途读取。

## 4.3 成本核算

```java
public class CostCalculator {

    // 单位：元 / 千 Token，示例值，需按实际价目维护
    private static final double IN_PRICE = 0.008;
    private static final double OUT_PRICE = 0.024;

    public double cost(long prompt, long completion) {
        return prompt / 1000.0 * IN_PRICE + completion / 1000.0 * OUT_PRICE;
    }
}
```

### 4.3.1 成本维度

| 维度 | 用途 | 是否作标签 |
|------|------|------------|
| 租户 | 计费与配额 | 是（有限集合） |
| 功能模块 | 成本归因 | 是 |
| 用户 ID | 个人用量 | 否（高基数） |
| 模型 | 单价差异 | 是 |

# 5. 结构化日志

**结论先行**：日志必须结构化、可检索、可脱敏；LLM 请求/响应日志尤其要控制长度并遮蔽隐私字段。

## 5.1 日志设计

```java
@Slf4j
public class StructuredLoggingAdvisor implements CallAdvisor {

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        String traceId = MDC.get("traceId");
        log.info("llm.request traceId={} messages={}",
                traceId, request.prompt().getInstructions().size());

        ChatClientResponse resp = chain.nextCall(request);

        log.info("llm.response traceId={} answer={}",
                traceId, truncate(resp.chatResponse().getResult().getOutput().getText()));
        return resp;
    }

    private String truncate(String s) {
        return s.length() > 500 ? s.substring(0, 500) + "..." : s;
    }

    @Override
    public String getName() { return "StructuredLoggingAdvisor"; }

    @Override
    public int getOrder() { return 0; }
}
```

## 5.2 日志脱敏

> **日志必须脱敏**：手机号、身份证、密钥、用户隐私一律遮蔽后再记录。

```java
public final class Masker {

    private static final Pattern PHONE =
            Pattern.compile("1[3-9]\\d{9}");
    private static final Pattern ID =
            Pattern.compile("\\d{17}[\\dXx]");

    public static String mask(String text) {
        if (text == null) return null;
        String s = PHONE.matcher(text).replaceAll("1**********");
        return ID.matcher(s).replaceAll("******************");
    }
}
```

## 5.3 日志字段规范

| 字段 | 含义 | 是否必填 |
|------|------|----------|
| traceId | 链路标识 | 必填 |
| conversationId | 会话标识 | 必填 |
| model | 模型名 | 必填 |
| tokens | Token 数 | 必填 |
| latencyMs | 耗时 | 必填 |
| status | 结果状态 | 必填 |

# 6. OpenTelemetry 接入

**结论先行**：通过 Micrometer Tracing 桥接 OpenTelemetry，可把模型、工具、检索调用统一导出为 Trace，在标准平台上查看完整链路。

## 6.1 依赖与配置

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

这样每次请求会生成一条 Trace，模型调用、工具调用、检索都成为其 Span，可在 Jaeger / Tempo 等平台查看完整链路。

## 6.2 Span 设计

| Span | 类型 | 关键属性 |
|------|------|----------|
| chat | CLIENT | model、tokens、finish_reason |
| tool | INTERNAL | tool.name、status |
| retrieval | INTERNAL | topK、doc.count |
| memory | INTERNAL | read/write |

### 6.2.1 Span 嵌套规则

Span 的父子关系应反映真实调用层次：检索与记忆是 chat 的前置，工具调用是 chat 的后续，二者不应同级。

## 6.3 采样策略

| 策略 | 适用 | 说明 |
|------|------|------|
| 概率采样 | 高流量生产 | 如 10% |
| 错误全采样 | 排障 | 失败请求必留 |
| 关键租户全采样 | 重点客户 | 按租户配置 |
| 头部采样 | 低流量 | 全量 |

# 7. Agent Trace 设计

**结论先行**：Agent 的 Trace 应能完整还原"推理-行动"循环，这是排查 Agent 问题的第一手资料。

## 7.1 Trace 结构

```
Trace: 用户提问
├── Span: 记忆读取
├── Span: 模型调用（第 1 轮，决定调用工具）
├── Span: 工具调用 getOrder("A123")
│     └── Span: 数据库查询
├── Span: 模型调用（第 2 轮，生成回答）
└── Span: 记忆写入
```

## 7.2 关键属性

关键属性：`traceId`、`conversationId`、`model`、`tokens`、`cost`、`tool.name`、`tool.result.status`。

| 属性 | 用途 |
|------|------|
| traceId | 跨系统串联 |
| conversationId | 多轮归组 |
| model | 模型归因 |
| tokens / cost | 成本归因 |
| tool.name / status | 工具排障 |

## 7.3 回放与调试

Trace 必须能"回放"：给定 traceId，能还原每轮的输入提示、工具参数与返回、最终输出。这要求日志中保留（脱敏后的）关键内容，而不只是计数。

### 7.3.1 边界与坑

- 只记 Token 不记内容，无法定位"答错"；
- 只记内容不记 Token，无法定位"变贵"；
- 两者都需，且都要脱敏与截断。

> 详见 `13_Observability/3_agent_trace_and_spans.md`。

## 7.4 多轮会话的 Trace 关联

**结论先行**：单次请求的 Trace 无法反映多轮对话的完整质量，需用 conversationId 把多轮 Span 归组分析。

```java
// 伪代码：会话维度归组
String conversationId = session.getId();
// 每轮请求的 Trace 都携带同一 conversationId
// 分析时可据此还原完整多轮交互
```

### 7.4.1 多轮分析维度

| 维度 | 用途 |
|------|------|
| 轮次数 | 发现"越问越差" |
| 累计 Token | 会话级成本 |
| 工具调用序列 | 发现重复无效调用 |
| 用户打断率 | 体验指标 |

### 7.4.2 边界与坑

- 会话标识若用自增 ID，可能泄露业务规模，建议用不可枚举的 UUID；
- 会话数据保留需符合隐私政策，支持删除权。

# 8. 与外部平台集成

**结论先行**：平台选择取决于团队栈与合规要求；Spring AI 通过 OpenTelemetry 或自定义 Advisor 上报数据，接入方式基本一致。

## 8.1 平台对比

| 平台 | 说明 |
|------|------|
| Langfuse | 开源 LLM 可观测平台，支持 Prompt 管理、评测 |
| LangSmith | LangChain 官方平台 |
| Arize Phoenix | 开源 Trace 与评测 |
| Grafana + Tempo + Prometheus | 通用可观测栈 |

## 8.2 接入方式

Spring AI 通过 OpenTelemetry / 自定义 Advisor 上报数据。

```
Spring AI ──Observation──▶ Micrometer ──▶ OTLP ──▶ Collector
                                                      │
                                        ┌─────────────┼─────────────┐
                                        ▼             ▼             ▼
                                     Tempo        Prometheus      Jaeger
```

### 8.2.1 选型建议

| 诉求 | 推荐 |
|------|------|
| 只想看 Trace | Tempo / Jaeger |
| 指标告警为主 | Prometheus + Grafana |
| LLM 专项评测 | Langfuse / Phoenix |
| 已有通用栈 | Grafana 全家桶 |

# 9. 生产注意事项

**结论先行**：可观测本身也有成本与隐私风险，需在采样、脱敏、基数、链路贯通四方面做好治理。

## 9.1 采样与成本

| 风险 | 建议 |
|------|------|
| 采样过全导致成本高 | 生产按比例采样（如 10%）+ 错误全采样 |
| 记录完整 Prompt 泄露隐私 | 脱敏、截断、按级别开关 |

## 9.2 隐私与高基数

| 风险 | 建议 |
|------|------|
| 高基数标签 | 不要把用户 ID 当指标标签 |
| 无成本监控 | 按租户/用户维度统计成本 |

### 9.2.1 高基数反例

```java
// 反例：把用户 ID 作为指标标签，时间序列数量爆炸
registry.counter("ai.requests", "userId", userId).increment();
```

正确做法是把用户维度放进 Trace 或日志，指标只保留有限集合的标签。

## 9.3 全链路贯通

| 风险 | 建议 |
|------|------|
| 无 TraceId 贯穿 | 网关生成 traceId，全链路透传 |

### 9.3.1 治理清单

| 检查项 | 要求 | 状态 |
|--------|------|------|
| TraceId 透传 | 网关生成、跨服务传递 | 必选 |
| 日志脱敏 | 敏感字段遮蔽 | 必选 |
| 指标基数 | 标签取值有限 | 必选 |
| 采样策略 | 概率 + 错误全采 | 建议 |
| 成本看板 | 按租户/模块 | 建议 |
| 回放能力 | 可按 traceId 还原 | 建议 |

# 10. 小结

- Spring AI 的 Observation 让 LLM 调用天然可观测。
- 核心指标：延迟、Token、成本、工具错误率。
- Trace 是调试 Agent 的第一手资料，必须能"回放"。
- 日志与 Trace 都要脱敏，指标要避免高基数。
- 采样、脱敏、基数、链路贯通是可观测治理的四条底线。
