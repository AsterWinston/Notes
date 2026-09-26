# 1. Agent Trace 的独特之处

结论先行：普通服务的一次请求是一条浅链（深度 3~5），而 Agent（智能体）的一次请求是"多轮循环 + 多次模型/工具/检索调用"的深链（深度 10+）。这决定了 Agent 的追踪必须按步骤埋点，且要能表达循环与决策。

## 1.1 与普通服务的差异

结论先行：Agent Trace 在深度、动态性和语义三个维度都远超普通服务。

机制拆解：普通服务的调用图在编译期基本确定；Agent 的调用图在运行时由模型决策动态生成，每一轮的下一步取决于上一轮结果。

```
普通服务：一次请求 = 一条链（深度 3~5）
Agent：   一次请求 = 多轮循环 + 多次模型/工具/检索调用（深度 10+）
```

### 1.1.1 深度与广度

机制拆解：深度来自多轮"思考-行动"循环，广度来自并行工具与并行检索。

边界与坑：深度过大时，Trace 可视化会非常宽，需要折叠与聚合。

### 1.1.2 动态性与循环

机制拆解：同一目标可能走不同路径，也可能陷入重复调用同一工具的循环。

```java
public record AgentStep(
        int index,
        String decision,          // tool_call / final_answer
        String toolName,
        String toolArgsHash
) {}
```

边界与坑：若不记录步号，无法判断"是否重复调用同一工具"。

## 1.2 一次 Agent 请求的完整链路

结论先行：从记忆读取到记忆写入，中间穿插多轮模型与工具调用，构成完整因果链。

机制拆解：每一轮模型调用产生决策，决策触发工具，工具结果回填后再进入下一轮。

```
Trace: 用户目标 "查订单 A123 物流并催单"
├── Span: 记忆读取
├── Span: LLM 调用 #1（决策：调用 queryOrder）
├── Span: Tool queryOrder("A123")
│   └── Span: DB 查询
├── Span: LLM 调用 #2（决策：调用 queryLogistics）
├── Span: Tool queryLogistics("SF123")
│   └── Span: HTTP 调用
├── Span: LLM 调用 #3（决策：调用 urgeOrder）
├── Span: Tool urgeOrder("SF123")
├── Span: LLM 调用 #4（生成最终答案）
└── Span: 记忆写入
```

## 1.3 为什么需要步骤级埋点

结论先行：只有步骤级埋点，才能回答"第几步开始偏离、哪次工具返回错误"。

机制拆解：Agent 的失败往往是"某一步工具返回异常，但模型仍继续"；若只记录最终答案，无法定位。

边界与坑：步骤级埋点会增加 Span 数量，需要配合采样与属性裁剪。

生产建议：为每步记录决策类型、工具名与结果状态，形成可回放序列。

# 2. Span 命名规范

结论先行：统一命名规范是 Agent Trace 可查询、可聚合的前提。命名应体现"类别.对象"，避免自由文本。

## 2.1 命名约定

结论先行：Agent 执行、单步、模型、工具、检索、记忆各有固定前缀。

机制拆解：前缀用于分类聚合，后缀用于具体对象；步号用 `#n` 后缀。

| Span | 命名 | 示例 |
|------|------|------|
| Agent 执行 | `agent.run` | `agent.run` |
| Agent 单步 | `agent.step` | `agent.step#2` |
| 模型调用 | `llm.chat` | `llm.chat` |
| 工具调用 | `tool.{name}` | `tool.queryOrder` |
| 检索 | `retrieval` | `retrieval.vector` |
| 记忆 | `memory.read` / `memory.write` | `memory.read` |

## 2.2 命名反例

结论先行：把动态内容写进 Span 名会导致基数爆炸。

机制拆解：`name` 通常是查询维度，若包含用户 ID、订单号，则无法聚合。

```java
// 错误：名称含动态参数
tracer.spanBuilder("tool.queryOrder(A123)").startSpan();

// 正确：名称固定，参数放属性
Span span = tracer.spanBuilder("tool.queryOrder").startSpan();
span.setAttribute("tool.args.id", "A123");
```

边界与坑：动态名称会让 Trace 列表出现海量不同名称，无法统计。

## 2.3 命名与查询

结论先行：规范命名让"按工具统计成功率"等查询成为可能。

机制拆解：查询时用前缀匹配（如 `tool.*`）聚合所有工具调用。

| 查询 | 用途 |
|------|------|
| `name = llm.chat` | 统计模型调用 |
| `name LIKE tool.%` | 统计所有工具 |
| `name = agent.step#n` | 定位第 n 步 |

生产建议：把命名规范写入团队文档并做代码评审。

# 3. 关键属性

结论先行：属性是 Agent Trace 的信息载体，应覆盖 Agent 级、模型级、工具级与检索级四类。

## 3.1 Agent 级属性

结论先行：记录 Agent 身份、版本、目标、状态与步数，用于归因与对比。

机制拆解：版本用于 A/B 对比，目标需截断以控体积，步数用于识别循环。

```java
// Agent 级
span.setAttribute("agent.id", agentId);
span.setAttribute("agent.version", version);
span.setAttribute("agent.goal", truncate(goal, 200));
span.setAttribute("agent.status", status);
span.setAttribute("agent.steps", totalSteps);
```

## 3.2 模型调用属性

结论先行：模型属性用于成本、性能与决策分析。

### 3.2.1 Token 与结束原因

机制拆解：Token 用于计费，`finish_reason` 用于判断截断与工具触发。

```java
// 模型调用
span.setAttribute("llm.model", model);
span.setAttribute("llm.usage.input_tokens", inputTokens);
span.setAttribute("llm.usage.output_tokens", outputTokens);
span.setAttribute("llm.finish_reason", finishReason);
```

### 3.2.2 决策与工具调用

机制拆解：记录本轮是否产生工具调用及工具名，用于判断决策是否符合预期。

```java
span.setAttribute("llm.tool_calls", toolCalls.size());
span.setAttribute("llm.decision", decision);
```

边界与坑：若只记录最终文本，无法区分"模型没调用工具"与"工具调用失败"。

## 3.3 工具与检索属性

结论先行：工具记录名称、参数（脱敏）、状态与耗时；检索记录查询、命中数与最高分。

机制拆解：参数脱敏后记录，便于复现但不泄露；检索分数用于诊断召回。

```java
// 工具调用
span.setAttribute("tool.name", toolName);
span.setAttribute("tool.args", maskArgs(args));
span.setAttribute("tool.status", status);
span.setAttribute("tool.duration_ms", duration);

// 检索
span.setAttribute("retrieval.query", truncate(query, 200));
span.setAttribute("retrieval.hits", hits.size());
span.setAttribute("retrieval.top_score", topScore);
```

边界与坑：完整工具参数可能含个人信息，必须脱敏；检索原文不应整体写入属性。

生产建议：建立属性白名单，禁止未登记属性入库。

# 4. 用 Advisor 埋点（Spring AI）

结论先行：Spring AI 的 Advisor（顾问）机制可在不侵入业务代码的情况下自动埋点。

## 4.1 Advisor 机制

结论先行：Advisor 环绕在 ChatClient 调用前后，可统一创建 Span、记录 Token 与异常。

机制拆解：`CallAdvisor` 处理同步调用，`StreamAdvisor` 处理流式调用；`getOrder()` 决定执行顺序。

## 4.2 完整实现

结论先行：在调用前创建 Span，调用后记录用量，异常时记录并结束。

```java
public class AgentTraceAdvisor implements CallAdvisor, StreamAdvisor {

    private final Tracer tracer;

    public AgentTraceAdvisor(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        Span span = tracer.spanBuilder("llm.chat").startSpan();
        span.setAttribute("llm.messages", request.prompt().getInstructions().size());

        try (Scope scope = span.makeCurrent()) {
            ChatClientResponse response = chain.nextCall(request);

            Usage usage = response.chatResponse().getMetadata().getUsage();
            if (usage != null) {
                span.setAttribute("llm.usage.input_tokens", usage.getPromptTokens());
                span.setAttribute("llm.usage.output_tokens", usage.getCompletionTokens());
            }
            span.setStatus(StatusCode.OK);
            return response;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }

    @Override
    public Flux<ChatClientResponse> adviseStream(ChatClientRequest request,
                                                 StreamAdvisorChain chain) {
        Span span = tracer.spanBuilder("llm.chat.stream").startSpan();
        return chain.nextStream(request)
                .doOnComplete(span::end)
                .doOnError(e -> {
                    span.recordException(e);
                    span.setStatus(StatusCode.ERROR);
                    span.end();
                });
    }

    @Override
    public String getName() { return "AgentTraceAdvisor"; }

    @Override
    public int getOrder() { return 250; }
}
```

## 4.3 边界与顺序

结论先行：Advisor 顺序影响 Span 嵌套关系，需与限流、重试等 Advisor 协调。

机制拆解：外层 Advisor 的 Span 会成为内层的父 Span；重试 Advisor 会多次调用模型，需合并或标记重试。

边界与坑：流式场景中若未处理取消（cancel），Span 可能永不结束。

生产建议：为流式路径增加超时与取消钩子，确保 Span 一定结束。

# 5. 用监听器埋点（LangChain4j）

结论先行：LangChain4j 通过 `ChatModelListener` 在请求、响应、错误三个时点回调，适合自动埋点。

## 5.1 监听器机制

结论先行：以 requestId 关联请求与响应，用并发 Map 保存进行中的 Span。

机制拆解：`onRequest` 创建 Span 并暂存；`onResponse` 记录用量并结束；`onError` 记录异常并结束。

## 5.2 完整实现

结论先行：用 `ConcurrentHashMap` 保存进行中的 Span，避免并发串号。

```java
public class TracingChatModelListener implements ChatModelListener {

    private final Tracer tracer;
    private final Map<String, Span> spans = new ConcurrentHashMap<>();

    @Override
    public void onRequest(ChatModelRequestContext ctx) {
        Span span = tracer.spanBuilder("llm.chat")
                .setAttribute("llm.model", ctx.modelName())
                .setAttribute("llm.messages", ctx.chatRequest().messages().size())
                .startSpan();

        spans.put(ctx.attributes().get("requestId").toString(), span);
    }

    @Override
    public void onResponse(ChatModelResponseContext ctx) {
        Span span = spans.remove(ctx.attributes().get("requestId").toString());
        if (span == null) return;

        TokenUsage usage = ctx.chatResponse().tokenUsage();
        if (usage != null) {
            span.setAttribute("llm.usage.input_tokens", usage.inputTokenCount());
            span.setAttribute("llm.usage.output_tokens", usage.outputTokenCount());
        }
        span.setStatus(StatusCode.OK);
        span.end();
    }

    @Override
    public void onError(ChatModelErrorContext ctx) {
        Span span = spans.remove(ctx.attributes().get("requestId").toString());
        if (span == null) return;

        span.recordException(ctx.error());
        span.setStatus(StatusCode.ERROR);
        span.end();
    }
}
```

## 5.3 并发与内存

结论先行：进行中的 Span 映射必须线程安全，且要有清理机制防止泄漏。

机制拆解：正常情况下响应或错误回调会移除条目；若回调缺失，需定时清理过期条目。

```java
// 兜底清理：超过阈值的未完成 Span 强制结束
scheduledExecutor.scheduleAtFixedRate(this::evictStale, 1, 1, TimeUnit.MINUTES);
```

边界与坑：requestId 缺失或重复会导致 Span 错配；流式响应的用量可能在结束时才可得。

生产建议：为监听器增加指标（进行中数量、清理次数）以便观测自身。

# 6. Trace 回放

结论先行：Trace 的价值在于能完整回放一次执行，把 Span 序列还原为可读的步骤列表。

## 6.1 回放的价值

结论先行：回放让"事后调试"成为可能，尤其适合非确定性的 Agent 行为。

机制拆解：按时间排序 Span，按名称映射为语义步骤（模型步、工具步、通用步）。

## 6.2 回放实现

结论先行：用模式匹配把 Span 转为步骤对象。

```java
public ExecutionReplay replay(String traceId) {
    List<Span> spans = traceStore.findByTraceId(traceId);

    return new ExecutionReplay(spans.stream()
            .sorted(Comparator.comparing(Span::startTime))
            .map(span -> switch (span.name()) {
                case "llm.chat" -> LlmStep.of(
                        span.attribute("llm.model"),
                        span.attribute("llm.usage.input_tokens"),
                        span.attribute("llm.usage.output_tokens"));
                case String n when n.startsWith("tool.") -> ToolStep.of(
                        span.attribute("tool.name"),
                        span.attribute("tool.args"),
                        span.attribute("tool.status"));
                default -> GenericStep.of(span.name(), span.duration());
            })
            .toList());
}
```

## 6.3 回放边界

结论先行：回放依赖保存了足够属性；属性缺失则回放不完整。

机制拆解：需要保存输入摘要、工具参数（脱敏）、结果状态与用量。

边界与坑：若采样导致部分 Span 缺失，回放会出现"断点"。

生产建议：对错误 Trace 全量保存，确保回放完整。

# 7. 常见问题定位

结论先行：大多数 Agent 问题都能在 Trace 中找到对应线索，关键是把症状映射到查询。

## 7.1 症状与线索

结论先行：慢、错、不调用、死循环、成本高、报错六类症状各有线索。

| 症状 | Trace 中的线索 |
|------|----------------|
| 回答慢 | 哪个 Span 耗时最长 |
| 答错 | 检索命中了什么、模型输入是什么 |
| 工具没调用 | LLM 决策 Span 有无 tool_call |
| 死循环 | 工具调用 Span 是否重复 |
| 成本高 | 各模型调用 Token 之和 |
| 报错 | ERROR Span 与异常信息 |

## 7.2 排查流程

结论先行：从顶层 Span 耗时排序开始，逐层下钻到异常或最长 Span。

机制拆解：先看 `agent.run` 总耗时与步数，再看每轮 `llm.chat` 与 `tool.*` 的耗时与状态。

```yaml
inspect:
  steps:
    - total_duration: agent.run
    - slowest_spans: top(10)
    - error_spans: status == ERROR
    - repeated_tools: group_by(tool.name) > threshold
```

边界与坑：只看总耗时可能误判；瓶颈常在某个外部工具或检索。

## 7.3 典型案例

结论先行：以"工具没被调用"为例，逐层验证决策链。

机制拆解：检查系统提示是否描述工具、模型是否支持工具调用、`finish_reason` 是否为 `tool_calls`、参数是否解析成功。

| 检查点 | 预期 |
|--------|------|
| 工具是否注册 | 已注册且描述清晰 |
| finish_reason | tool_calls |
| 参数解析 | 成功且合法 |
| 工具执行 | SUCCESS |

生产建议：把高频案例固化为排查手册。

# 8. 多 Agent Trace

结论先行：多 Agent（如 Supervisor + 子 Agent）场景下，必须保证子 Agent 的 Span 挂到同一父 Trace 上，否则链路割裂。

## 8.1 层级结构

结论先行：Supervisor 决策派发，子 Agent 执行，最后汇总。

机制拆解：每个子 Agent 是一个子 Span 子树，其模型与工具调用挂在其下。

```
Trace: 复杂任务
├── Span: Supervisor 决策
├── Span: Agent A 执行（独立子 Trace）
│   ├── Span: LLM
│   └── Span: Tool
├── Span: Agent B 执行
│   └── Span: LLM
└── Span: Supervisor 汇总
```

## 8.2 上下文传递

结论先行：派发子 Agent 时必须传递当前 Context，子 Agent 内部才能正确挂载。

机制拆解：把父 Span 的 Context 作为子 Agent 的父上下文，跨进程时通过 `traceparent` 传递。

```java
Context parent = Context.current();
Context child = parent.with(span.getSpanContext());
```

> 多 Agent 场景要保证子 Agent 的 Span 挂到父 Trace 上。

## 8.3 边界与坑

结论先行：并行子 Agent、跨服务子 Agent、独立触发子 Agent 的处理方式不同。

机制拆解：并行子 Agent 共享父 Span；跨服务需透传；独立触发应用 Link 而非父子。

边界与坑：子 Agent 各自生成新 `traceId`，导致无法与主任务关联。

生产建议：统一由调度器注入父上下文，禁止子 Agent 自行生成根 Span。

# 9. 生产注意事项

结论先行：Agent Trace 的落地风险集中在埋点缺失、敏感泄露、上下文丢失与回放能力不足。

## 9.1 常见风险

结论先行：逐项对照，优先解决"无 Agent 级 Span"与"无回放能力"。

| 风险 | 建议 |
|------|------|
| 无 Agent 级 Span | 显式埋点 |
| 属性含敏感信息 | 脱敏/截断 |
| 工具参数全记录 | 脱敏后记录 |
| 上下文丢失 | 显式传递 |
| 无 traceId 贯穿 | 全链路透传 |
| 无回放能力 | 保存完整 Span |

## 9.2 检查清单

结论先行：上线前确认命名、属性、回放与多 Agent 关联均达标。

| 检查项 | 通过标准 |
|--------|----------|
| Span 命名 | 符合规范前缀 |
| Agent 属性 | 身份/版本/步数齐备 |
| Token 用量 | 已记录 |
| 工具参数 | 已脱敏 |
| 回放 | 错误 Trace 可完整回放 |
| 多 Agent | 挂到同一 Trace |

边界与坑：清单需随 Agent 架构演进更新，否则会漏检新链路。

生产建议：把检查项纳入 CI 与发布门禁。

# 10. 小结

- Agent Trace 比普通服务深得多，需按步骤埋点。
- 规范 Span 命名与关键属性，动态内容放属性而非名称。
- 用 Advisor/Listener 自动埋点，注意并发安全与 Span 清理。
- Trace 的价值在于完整回放，错误 Trace 应全量保存。
- 多 Agent 场景要保证 Span 挂到同一 Trace。
- 排查从耗时与状态入手，逐层下钻到根因 Span。
