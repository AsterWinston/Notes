# 1. ChatModelListener：核心观测入口

结论先行：`ChatModelListener` 是 LangChain4j 可观测性的统一入口。它把模型调用的全生命周期暴露为三个回调，日志、指标、Trace、成本统计都可以挂在这三个点上实现。

## 1.1 生命周期回调

```java
public interface ChatModelListener {
    default void onRequest(ChatModelRequestContext ctx) {}
    default void onResponse(ChatModelResponseContext ctx) {}
    default void onError(ChatModelErrorContext ctx) {}
}
```

```
onRequest ──▶ （模型调用） ──▶ onResponse
                       └────▶ onError
```

一次调用只会走 `onRequest` 后接 `onResponse` 或 `onError` 之一，不会同时触发成功与失败。

## 1.2 上下文对象

### 1.2.1 请求上下文

`ChatModelRequestContext` 携带模型名、`ChatRequest`（含全部消息）以及一个可变的 `attributes()` 映射。`attributes()` 是跨回调传递数据的通道，常用于把 `onRequest` 里创建的 Span 传给 `onResponse`。

### 1.2.2 响应上下文

`ChatModelResponseContext` 携带 `ChatResponse`，可从中读取 `aiMessage()`、`finishReason()` 与 `tokenUsage()`。这是指标与成本统计的主要数据源。

### 1.2.3 错误上下文

`ChatModelErrorContext` 携带异常对象与模型名，用于错误计数、异常上报与 Span 结束。

## 1.3 为什么需要统一入口

模型调用散落在 AI Service、Agent、RAG 流程各处，若每个调用点各自埋点，代码会被观测逻辑污染。`ChatModelListener` 把观测收敛到模型这一层，业务代码保持干净，观测能力随模型自动生效。

| 方案 | 侵入性 | 覆盖度 |
|------|--------|--------|
| 业务代码内埋点 | 高 | 易漏 |
| 代理/网关埋点 | 低 | 看不到内部消息 |
| ChatModelListener | 低 | 全量覆盖 |

# 2. 注册监听器

结论先行：监听器既可以在单个模型上注册，也可以通过 SPI 全局注册；多个监听器按顺序执行，且单个监听器抛出的异常会被隔离，不影响主流程。

## 2.1 单模型注册

```java
ChatModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")
        .listeners(List.of(new LoggingListener(), new MetricsListener()))
        .build();
```

> 多个监听器按顺序执行；异常会被隔离，不影响主流程。

## 2.2 全局注册

通过 Java SPI 机制注册，可对所有模型生效，无需逐处配置：

```
src/main/resources/META-INF/services/
    dev.langchain4j.model.chat.listener.ChatModelListener
```

文件内容为实现类的全限定名，每行一个。适合统一的日志与指标采集。

## 2.3 执行顺序与隔离

| 特性 | 行为 |
|------|------|
| 顺序 | 按注册顺序依次调用 |
| 异常 | 被框架捕获并记录，不中断其他监听器 |
| 线程 | 回调发生在调用线程，需注意阻塞 |

尽管框架会隔离异常，监听器内部仍建议自行 try-catch，避免噪音日志淹没真正的错误。

## 2.4 监听器与内容审核

`onResponse` 也是做内容审核（moderation）的天然位置：拿到模型输出后检查敏感内容，命中则记录并告警。注意审核通常需要额外调用，不要阻塞主线程过久，可异步处理。

## 2.5 注册位置的选择

| 位置 | 优点 | 缺点 |
|------|------|------|
| 单模型 builder | 精准可控 | 需逐处配置 |
| SPI 全局 | 一次生效 | 无法按模型差异化 |
| 两者结合 | 通用采集 + 特殊处理 | 注意重复计数 |

# 3. 日志监听器

结论先行：日志监听器是最容易落地的观测手段，但记录 Prompt 与响应正文时必须脱敏与截断，否则会带来隐私与合规风险。

## 3.1 实现

```java
@Slf4j
public class LoggingListener implements ChatModelListener {

    @Override
    public void onRequest(ChatModelRequestContext ctx) {
        log.info("llm.request model={} messages={}",
                ctx.modelName(),
                ctx.chatRequest().messages().size());
    }

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        ChatResponse response = ctx.chatResponse();
        TokenUsage usage = response.tokenUsage();
        log.info("llm.response finishReason={} inputTokens={} outputTokens={}",
                response.finishReason(),
                usage == null ? null : usage.inputTokenCount(),
                usage == null ? null : usage.outputTokenCount());
    }

    @Override
    public void onError(ChatModelErrorContext ctx) {
        log.error("llm.error", ctx.error());
    }
}
```

## 3.2 脱敏

> 记录 Prompt/响应正文时**必须脱敏**，避免隐私与合规风险。

常见做法：

| 手段 | 说明 |
|------|------|
| 截断 | 只记录前 N 个字符 |
| 掩码 | 手机号、身份证、邮箱替换为占位符 |
| 分级 | 通过开关控制是否记录正文，生产默认关闭 |
| 采样 | 只对少量请求记录完整正文 |

## 3.3 结构化日志

用 JSON 结构化输出，便于日志平台检索与聚合：

```java
log.info("{}", objectMapper.writeValueAsString(Map.of(
        "event", "llm.response",
        "model", ctx.modelName(),
        "inputTokens", usage.inputTokenCount(),
        "outputTokens", usage.outputTokenCount(),
        "traceId", MDC.get("traceId"))));
```

关键字段至少包含 `event`、`model`、`traceId`、Token 数与耗时，便于按 Trace 串联。

# 4. 指标监听器（Micrometer）

结论先行：指标比日志更适合长期趋势与告警。通过 Micrometer 把 Token、错误、延迟等写入时序数据库，就能做看板与预算告警。

## 4.1 实现

```java
public class MetricsListener implements ChatModelListener {

    private final MeterRegistry registry;

    public MetricsListener(MeterRegistry registry) {
        this.registry = registry;
    }

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        TokenUsage usage = ctx.chatResponse().tokenUsage();
        if (usage != null) {
            registry.counter("ai.tokens.input").increment(usage.inputTokenCount());
            registry.counter("ai.tokens.output").increment(usage.outputTokenCount());
        }
    }

    @Override
    public void onError(ChatModelErrorContext ctx) {
        registry.counter("ai.errors",
                "type", ctx.error().getClass().getSimpleName()).increment();
    }
}
```

## 4.2 关键指标

| 指标 | 含义 |
|------|------|
| `ai.tokens.input` | 输入 Token |
| `ai.tokens.output` | 输出 Token |
| `ai.errors` | 调用失败次数 |
| `ai.latency` | 调用耗时 |
| `ai.tool.calls` | 工具调用次数 |
| `ai.tool.errors` | 工具错误数 |

延迟需要在 `onRequest` 记录开始时间戳，`onResponse`/`onError` 计算差值，并借助 `attributes()` 传递。

## 4.3 高基数陷阱

指标标签（tag）的取值组合数决定时序数量。把 `userId`、`traceId` 这类高基数值作为标签，会让时序数据库迅速膨胀。标签应只保留低基数的维度，如 `model`、`tenant`、`status`。

## 4.4 延迟指标实现

延迟需跨 `onRequest` 与 `onResponse` 两个回调，用 `attributes()` 传递开始时间：

```java
@Override
public void onRequest(ChatModelRequestContext ctx) {
    ctx.attributes().put("startNs", System.nanoTime());
}

@Override
public void onResponse(ChatModelResponseContext ctx) {
    Object start = ctx.attributes().get("startNs");
    if (start instanceof Long ns) {
        registry.timer("ai.latency", "model", ctx.modelName())
                .record(System.nanoTime() - ns, TimeUnit.NANOSECONDS);
    }
}
```

计时器（Timer）会同时产出计数、总和与分位数，适合做 P95/P99 延迟看板。

# 5. 工具调用观测

结论先行：工具调用是 Agent 可观测性的重点，因为它是模型与外部系统交互的边界，也是最容易出错、最需要审计的环节。

## 5.1 监听响应中的工具请求

```java
public class ToolCallListener implements ChatModelListener {

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        AiMessage ai = ctx.chatResponse().aiMessage();
        if (ai.hasToolExecutionRequests()) {
            for (ToolExecutionRequest req : ai.toolExecutionRequests()) {
                log.info("tool.call name={} args={}", req.name(), req.arguments());
            }
        }
    }
}
```

> 工具调用是 Agent 可观测性的重点，需记录名称、参数、耗时与结果状态。

## 5.2 工具执行耗时

`ChatModelListener` 只能看到模型"请求调用工具"，看不到工具真正执行的过程与结果。要观测工具执行耗时与状态，需要在工具方法内部或通过包装器埋点：

```java
public class TimedOrderTools {
    private final OrderTools delegate;

    public String getOrder(String id) {
        long start = System.nanoTime();
        try {
            return delegate.getOrder(id);
        } finally {
            registry.timer("ai.tool.latency", "tool", "getOrder")
                    .record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
        }
    }
}
```

## 5.3 工具结果观测

只记录"调用了工具"不够，还要记录结果状态，才能算出工具成功率：

| 字段 | 说明 |
|------|------|
| `tool.name` | 工具名 |
| `tool.arguments` | 参数（脱敏后） |
| `tool.status` | success / error |
| `tool.duration` | 耗时 |

有了这些字段，就能回答"哪个工具最容易失败""哪次调用拖慢了整条链路"。

# 6. 与 OpenTelemetry 集成

结论先行：LangChain4j 的监听器机制可以无缝对接 OpenTelemetry（OTel，开放遥测标准），把每次模型调用变成一个 Span，从而在分布式链路中与其他服务串联。

## 6.1 自定义 Listener 方案

```java
public class OtelListener implements ChatModelListener {

    private final Tracer tracer;

    public OtelListener(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public void onRequest(ChatModelRequestContext ctx) {
        Span span = tracer.spanBuilder("llm.chat")
                .setAttribute("llm.model", ctx.modelName())
                .startSpan();
        ctx.attributes().put("span", span);   // 传递到 onResponse
    }

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        Object span = ctx.attributes().get("span");
        if (span instanceof Span s) {
            TokenUsage usage = ctx.chatResponse().tokenUsage();
            if (usage != null) {
                s.setAttribute("llm.usage.input", usage.inputTokenCount());
                s.setAttribute("llm.usage.output", usage.outputTokenCount());
            }
            s.end();
        }
    }

    @Override
    public void onError(ChatModelErrorContext ctx) {
        Object span = ctx.attributes().get("span");
        if (span instanceof Span s) {
            s.recordException(ctx.error());
            s.end();
        }
    }
}
```

> 也可直接使用社区提供的 OTel / Micrometer 集成模块。

## 6.2 Span 属性约定

遵循 OpenTelemetry 的 GenAI 语义约定（semantic conventions）能让看板与厂商兼容：

| 属性 | 含义 |
|------|------|
| `gen_ai.system` | 供应商标识，如 `openai` |
| `gen_ai.request.model` | 请求模型名 |
| `gen_ai.usage.input_tokens` | 输入 Token |
| `gen_ai.usage.output_tokens` | 输出 Token |

## 6.3 上下文传播

在 Web 服务中，模型调用往往发生在请求线程的深处。要让 OTel 把 `llm.chat` Span 挂到请求的父 Span 下，必须保证 `Context`（上下文）正确传播。若使用异步或线程池，需显式传递上下文，否则 Span 会断链。

# 7. 与观测平台集成

结论先行：选平台取决于你已有的技术栈——已有 OTel 就接 Phoenix/Tempo，需要 LLM 专用看板就接 Langfuse/LangSmith。

## 7.1 平台对照

| 平台 | 集成方式 |
|------|----------|
| Langfuse | 自定义 `ChatModelListener` 上报 |
| LangSmith | 自定义 `ChatModelListener` 上报 |
| Arize Phoenix | OTel 上报 |
| Grafana + Prometheus + Tempo | Micrometer + OTel |

## 7.2 选型建议

| 诉求 | 推荐 |
|------|------|
| 已有 Grafana 体系 | Micrometer + OTel |
| 需要 Prompt 级调试 | Langfuse / LangSmith |
| 需要本地评测 | Arize Phoenix |

## 7.3 数据回流

观测数据不应只用于排障。把线上真实的输入输出回流到评测集，可持续改进 Prompt 与检索策略，形成"观测 → 评测 → 优化"的闭环。

# 8. Agent Trace 设计

结论先行：Agent 的一次任务应形成一条完整 Trace，把记忆读取、每轮 LLM 调用、每次工具调用都作为子 Span，这样才能回答"钱花在哪、卡在哪、错在哪"。

## 8.1 Trace 结构

```
Trace: 用户目标
├── Span: 记忆读取
├── Span: LLM 调用（第 1 轮，决策）
├── Span: 工具调用 getOrder("A123")
│     └── Span: DB 查询
├── Span: LLM 调用（第 2 轮）
├── Span: 工具调用 urgeOrder(...)
└── Span: LLM 调用（第 3 轮，最终答案）
```

关键属性：`traceId`、`memoryId`、`model`、`tokens`、`tool.name`、`tool.status`。

> 详见 `13_Observability/3_agent_trace_and_spans.md`。

## 8.2 关键属性

| 属性 | 用途 |
|------|------|
| `traceId` | 贯穿整条链路 |
| `memoryId` | 关联会话 |
| `model` | 归因模型 |
| `tokens` | 成本归因 |
| `tool.name` / `tool.status` | 工具成功率分析 |

## 8.3 Trace 与日志关联

Trace 与日志必须能互相跳转：日志里带 `traceId`，Trace 里带 `spanId`，排障时从一条错误日志直达完整链路。这也是网关生成并透传 `traceId` 的意义。

# 9. 成本监控

结论先行：Token 与成本必须是一等指标，按租户/用户维度统计并设置预算告警，否则很容易在无人察觉时成本失控。

## 9.1 成本计算

```java
// 按模型/租户维度累计成本
public void onResponse(ChatModelResponseContext ctx) {
    TokenUsage usage = ctx.chatResponse().tokenUsage();
    if (usage == null) return;

    double cost = costCalculator.cost(
            ctx.modelName(),
            usage.inputTokenCount(),
            usage.outputTokenCount());

    registry.counter("ai.cost",
            "model", ctx.modelName(),
            "tenant", TenantContext.current()).increment(cost);
}
```

> Token 与成本必须是一等指标，按租户/用户维度统计并设置预算告警。

## 9.2 预算告警

按"日/月"滚动窗口累计成本，超过阈值时告警甚至熔断。注意成本单价随模型不同而不同，`costCalculator` 需维护一张模型 → 单价的配置表，并在模型升级时同步更新。

## 9.3 成本优化

观测到成本后，常见优化手段：

| 手段 | 效果 |
|------|------|
| 小模型处理简单请求 | 显著降本 |
| 精简 Prompt / 少样本 | 减少输入 Token |
| 缓存重复请求 | 避免重复计费 |
| 限制最大输出长度 | 控制输出 Token |

# 10. 生产注意事项

结论先行：观测系统的三大风险是隐私、性能与治理。监听器本身必须容错，采集必须脱敏，标签必须克制。

## 10.1 隐私与合规

| 风险 | 建议 |
|------|------|
| 记录完整 Prompt 泄露隐私 | 脱敏 + 截断 + 分级开关 |
| 敏感字段入库 | 只存必要字段，其余哈希 |
| 合规审计缺失 | 保留调用审计日志 |

## 10.2 性能与采样

| 风险 | 建议 |
|------|------|
| 监听器抛异常 | 内部 try-catch，隔离失败 |
| 指标高基数 | 不要把 userId 当标签 |
| 采样过全 | 生产按比例采样 + 错误全采样 |

## 10.3 治理

| 风险 | 建议 |
|------|------|
| 无 traceId 贯穿 | 网关生成并透传 |
| 无成本监控 | 按租户/模型统计成本 |
| 无法复盘 | 保留完整轨迹并设保留期 |

## 10.4 数据保留与合规

观测数据本身含敏感信息，需设定保留期与访问控制：

| 数据 | 建议保留期 |
|------|------------|
| 指标 | 长期（已聚合、低敏感） |
| 日志 | 7~30 天 |
| 完整 Prompt/响应 | 尽量短，且脱敏 |
| Trace | 7~14 天 |

保留期应写入数据治理策略，并定期清理，避免观测系统成为新的隐私风险点。

# 11. 小结

- `ChatModelListener` 是 LangChain4j 可观测的统一入口。
- 日志、指标、Trace 都可通过监听器实现。
- 关键指标：Token、成本、延迟、工具错误率。
- 监听器必须容错，日志必须脱敏。
