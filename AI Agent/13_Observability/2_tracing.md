# 1. Trace 的核心概念

结论先行：Trace（追踪）是一次完整请求的调用链，Span（跨度）是链路上的一个操作。理解"Trace 是一棵树、Span 是树上的节点"是掌握分布式追踪的第一步。

## 1.1 Trace 与 Span 的定义

结论先行：Trace 由唯一 `traceId` 标识，Span 由唯一 `spanId` 标识，并通过 `parentSpanId` 表达父子关系。

机制拆解：一次用户请求从入口到出口经过多个服务与组件，每经过一个组件就产生一个 Span；所有 Span 共享同一 `traceId`，从而在存储中被还原成一棵树。

```
Trace   一次完整请求的调用链（有唯一 traceId）
  └── Span  链路上的一个操作（有唯一 spanId + parentSpanId）
        └── Span
```

### 1.1.1 Trace 的边界

机制拆解：Trace 的边界通常是"一次外部触发"，例如一次 HTTP 请求、一条消息消费或一次定时任务触发。子任务若由同一触发派生，应挂到同一 Trace；若为独立触发，应生成新 Trace 并用链接（Link）关联。

边界与坑：把异步独立任务强行并入同一 Trace，会让链路过长且语义混乱；反之拆分过细会丢失因果。

### 1.1.2 Span 的唯一标识

机制拆解：`traceId` 通常为 16 字节（32 位十六进制），`spanId` 为 8 字节（16 位十六进制）。ID 需全局唯一且不可预测。

```java
public record SpanId(String traceId, String spanId, String parentSpanId) {
    public boolean isRoot() {
        return parentSpanId == null || parentSpanId.isEmpty();
    }
}
```

## 1.2 父子关系与树形结构

结论先行：父子关系决定链路结构，错误的父子关系会导致链路断裂或错挂。

机制拆解：根 Span 的 `parentSpanId` 为空；其余 Span 的父 ID 指向产生它的当前 Span。

```
Trace (traceId=abc)
├── Span A (spanId=1, parent=null)
│   ├── Span B (spanId=2, parent=1)
│   └── Span C (spanId=3, parent=1)
└── Span D (spanId=4, parent=null)
```

边界与坑：`parent=null` 出现在非根位置，说明上下文丢失，链路被拆成多条独立 Trace。

## 1.3 概念对照

结论先行：Trace、Span、Event、Attribute、Link 各有明确职责，不要混用。

机制拆解：Event 表示 Span 内的瞬时事件（如日志），Attribute 表示键值属性，Link 表示跨 Trace 的因果引用。

| 概念 | 职责 | 示例 |
|------|------|------|
| Trace | 一次请求的完整链 | 用户提问请求 |
| Span | 一次操作 | `llm.chat` |
| Event | Span 内瞬时事件 | 重试、缓存未命中 |
| Attribute | 键值属性 | `llm.model=gpt-4o` |
| Link | 跨 Trace 引用 | 批处理关联上游 |

# 2. Span 的内容

结论先行：Span 由标识、时间、类型、状态、属性与事件组成，字段设计决定后续查询能力。

## 2.1 Span 数据结构

结论先行：用不可变记录承载 Span，保证线程安全与可重放。

机制拆解：`kind` 表示角色，`status` 表示结果，`attributes` 承载业务语义，`events` 记录时间点事件。

```java
public record Span(
        String traceId,
        String spanId,
        String parentSpanId,
        String name,
        Instant startTime,
        Duration duration,
        SpanKind kind,                    // INTERNAL / CLIENT / SERVER
        SpanStatus status,                // OK / ERROR / UNSET
        Map<String, Object> attributes,
        List<SpanEvent> events
) {}
```

## 2.2 SpanKind 与 SpanStatus

结论先行：Kind 描述"我是什么角色"，Status 描述"我做成功了没有"，二者不可混淆。

### 2.2.1 SpanKind 语义

机制拆解：`SERVER` 表示接收请求，`CLIENT` 表示发起调用，`INTERNAL` 表示内部操作，`PRODUCER`/`CONSUMER` 表示消息收发。

| Kind | 含义 | 示例 |
|------|------|------|
| INTERNAL | 内部操作 | 提示词组装 |
| CLIENT | 发起调用 | 调用模型 API |
| SERVER | 接收请求 | 接收用户请求 |
| PRODUCER | 发送消息 | 发布事件 |
| CONSUMER | 消费消息 | 消费任务 |

### 2.2.2 SpanStatus 语义

机制拆解：默认 `UNSET`；成功设 `OK`；失败设 `ERROR` 并记录异常。未显式设置会导致查询"状态=ERROR"时漏掉失败。

```java
span.setStatus(StatusCode.OK);

try {
    doWork();
} catch (Exception e) {
    span.recordException(e);
    span.setStatus(StatusCode.ERROR, e.getMessage());
    throw e;
}
```

边界与坑：仅 `recordException` 而不设 `ERROR`，在部分后端中不会被标记为失败。

## 2.3 SpanEvent 与 Attributes

结论先行：Event 记录"发生了什么时刻"，Attribute 记录"当时是什么状态"。

机制拆解：Event 带时间戳，适合重试、缓存命中等瞬时信息；Attribute 适合模型名、Token 数等稳定语义。

```java
span.addEvent("retry", Map.of("attempt", 2));
span.setAttribute("llm.model", "gpt-4o");
```

边界与坑：属性值过大（如完整文档）会显著增加存储与上报成本。

# 3. 上下文传播

结论先行：Trace 上下文必须跨线程、跨服务、跨消息传播，否则链路断裂。上下文传播是分布式追踪的生命线。

## 3.1 传播机制

结论先行：上下文以"当前作用域（Context）"的形式在调用栈中流动，跨边界时需序列化到载体。

机制拆解：进程内通过 `Context.current()` 与 `makeCurrent()` 维护；跨进程通过注入/提取（inject/extract）写入载体。

```
HTTP Header: traceparent: 00-{traceId}-{spanId}-01
（W3C Trace Context 标准）
```

## 3.2 传播载体

结论先行：不同传输方式使用不同载体，但语义一致。

### 3.2.1 HTTP 传播

机制拆解：使用 W3C `traceparent` 头，也可兼容 `b3` 等旧标准。客户端注入，服务端提取。

```java
// 服务间透传
webClient.get()
        .uri(url)
        .header("traceparent", currentTraceparent())
        .retrieve()
        .bodyToMono(String.class);
```

### 3.2.2 MQ 与异步传播

机制拆解：消息发送时把上下文写入消息头，消费时提取；线程池需在提交任务时捕获上下文。

| 场景 | 传播方式 |
|------|----------|
| HTTP | W3C traceparent header |
| MQ | 消息 header |
| 异步线程 | 手动传递上下文 |
| 响应式 | Reactor Context |

边界与坑：线程池复用线程会导致上下文"串号"，必须在任务开始时设置、结束时清理。

## 3.3 上下文丢失的典型场景

结论先行：异步、线程池、响应式、消息是四大高发场景。

机制拆解：凡是"换线程"或"跨进程"的地方，当前 Context 都不会自动跟随。

边界与坑：丢失后 Span 变成新的根，导致 Trace 被拆散，排查时"少了半条链"。

生产建议：在框架层统一封装传播逻辑，禁止业务代码自行拼装。

# 4. 手动埋点

结论先行：当自动埋点覆盖不到时，用手动埋点补齐；核心是"创建 Span、设为当前、记录属性与异常、结束 Span"。

## 4.1 埋点辅助类

结论先行：封装一个通用的 `traced` 方法，统一处理异常与结束，避免重复代码。

机制拆解：使用 try-with-resources 管理作用域，保证异常时也能结束 Span。

```java
public class TraceHelper {

    private final Tracer tracer;

    public <T> T traced(String name, Map<String, Object> attributes,
                        Supplier<T> action) {
        Span span = tracer.spanBuilder(name).startSpan();

        attributes.forEach((k, v) -> span.setAttribute(k, String.valueOf(v)));

        try (Scope scope = span.makeCurrent()) {
            T result = action.get();
            span.setStatus(StatusCode.OK);
            return result;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

## 4.2 属性设置与异常记录

结论先行：属性应在 Span 创建后尽快设置，异常必须显式记录。

机制拆解：属性越早设置，越能覆盖后续失败场景；异常记录包含类型、消息与堆栈。

```java
Map<String, Object> attrs = Map.of(
        "llm.model", model,
        "agent.step", step);
traceHelper.traced("llm.chat", attrs, () -> callModel(input));
```

## 4.3 埋点边界

结论先行：不是所有方法都需要埋点，只对"跨边界、耗时长、易失败"的操作埋点。

机制拆解：模型调用、工具调用、检索、DB、外部 HTTP 是重点；纯内存计算通常无需独立 Span。

边界与坑：过度埋点会产生海量 Span，增加存储与查询负担。

生产建议：为埋点制定规范清单，评审时检查覆盖与粒度。

# 5. 异步场景的上下文

结论先行：异步是上下文丢失的第一大来源，必须显式传递 Context。

## 5.1 线程池场景

结论先行：提交任务时捕获当前 Context，在任务内包装执行。

机制拆解：`Context.wrap()` 把当前上下文绑定到 Runnable/Supplier。

```java
// 手动传递上下文到异步线程
public CompletableFuture<String> callAsync(String input) {
    Context context = Context.current();

    return CompletableFuture.supplyAsync(
            context.wrap(() -> doCall(input)), executor);
}
```

边界与坑：若在任务内部才调用 `Context.current()`，得到的是线程池线程的空上下文。

## 5.2 响应式场景

结论先行：Reactor 的 Context 与 OpenTelemetry 的 Context 是两套体系，需要桥接。

机制拆解：把 OTel Context 放入 Reactor Context，并在操作符中取出。

```java
return Mono.deferContextual(ctx -> {
    Context otelContext = ctx.get(Context.class);
    return Mono.fromCallable(() -> doCall(input));
});
```

边界与坑：`subscribeOn`/`publishOn` 切换线程时，Reactor Context 会跟随，但 OTel Context 不会自动跟随，需显式恢复。

## 5.3 虚拟线程场景（JDK 21）

结论先行：JDK 21 虚拟线程同样不自动继承 OTel Context，但 InheritableThreadLocal 在虚拟线程下行为不同，需谨慎。

机制拆解：优先使用显式 Context 传递，避免依赖线程本地存储。

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Context ctx = Context.current();
    executor.submit(ctx.wrap(() -> doCall(input)));
}
```

边界与坑：虚拟线程数量巨大，若每个都绑定上下文且不清理，内存压力上升。

生产建议：统一通过 `wrap` 传递，禁止依赖 InheritableThreadLocal 传递追踪上下文。

# 6. 父子关系

结论先行：正确的父子关系让链路成为一棵树，错误的关系导致链路断裂。

## 6.1 正确与错误的父子关系

结论先行：子 Span 必须在父 Span 为"当前"时创建。

机制拆解：`makeCurrent()` 把 Span 设为当前，后续创建的 Span 自动成为其子。

```
正确：
Span A (模型调用)
└── Span B (工具调用)     parent = A

错误（上下文丢失）：
Span A (模型调用)          ← 独立 trace
Span B (工具调用)          ← 独立 trace，无法关联
```

## 6.2 上下文丢失的后果

结论先行：链路断裂会让根因定位从"看一棵树"退化为"拼多条独立记录"。

机制拆解：每个丢失点的 Span 都成为新根，`traceId` 不同，无法关联。

边界与坑：如果入口未透传 `traceId`，下游即使正确传播也拿不到上游 ID。

## 6.3 修复策略

结论先行：在边界处显式传递并恢复上下文，配合自动化测试验证。

```java
Span child = tracer.spanBuilder("tool.queryOrder")
        .setParent(parentContext)
        .startSpan();
try (Scope ignored = child.makeCurrent()) {
    doTool();
} finally {
    child.end();
}
```

> **上下文丢失是分布式追踪最常见的坑**：异步、线程池、响应式都要显式传递。

生产建议：编写"链路完整性"测试，断言 Span 数量与父子关系。

# 7. Trace 的采样

结论先行：采样在成本与可诊断性之间取舍，生产实践是"正常按比例、错误全采、慢请求全采"。

## 7.1 采样策略对比

结论先行：头部采样一致性好但看不到结果，尾部采样可按结果采但需要缓冲。

机制拆解：头部采样在 Trace 起点决定，所有服务继承决策；尾部采样收集完整 Trace 后按规则决定。

| 采样策略 | 说明 |
|----------|------|
| 头部采样 | 入口决定，一致性好 |
| 尾部采样 | 完成后决定，可按结果采 |
| 比例采样 | 简单，可能漏错误 |
| 错误全采 | 保错误，正常抽样 |

## 7.2 采样实现

结论先行：用 `parentBased` 保证一致性，并叠加错误强制采样。

### 7.2.1 头部采样

机制拆解：入口按比例决定，子 Span 继承父决策。

```java
// 头部采样：入口决定
Sampler sampler = Sampler.parentBased(
        Sampler.traceIdRatioBased(0.1));      // 10%
```

### 7.2.2 错误与慢请求强制采样

机制拆解：在尾部采样器或上报环节判断错误、超时，强制保留。

```java
// 错误必采
if (isError) {
    forceSample();
}
```

边界与坑：头部采样时错误尚不可知，纯头部方案无法做到错误全采。

## 7.3 采样与错误保留

结论先行：采样决策一旦做出，同一 Trace 内必须一致，否则链路残缺。

机制拆解：把采样标记写入 `traceparent` 的 flags 位，下游继承。

边界与坑：不同服务采样率不一致会导致链路"半截有、半截无"。

生产建议：错误率、P99 延迟、成本三类信号触发强制采样。

# 8. Trace 的存储

结论先行：存储选型取决于规模、查询需求与现有生态，保留期需与成本权衡。

## 8.1 存储选型

结论先行：LLM 专用平台适合语义化 Span 与评测，通用后端适合指标与告警联动。

| 存储 | 特点 |
|------|------|
| Jaeger | 开源，UI 好 |
| Tempo | Grafana 生态 |
| Zipkin | 轻量 |
| Langfuse | LLM 专用 |

## 8.2 保留期与成本

结论先行：保留期越长成本越高，应按价值分层。

机制拆解：正常 Trace 保留 7~30 天，错误与高价值 Trace 可保留更久。

| 类型 | 建议保留期 | 理由 |
|------|------------|------|
| 正常请求 | 7~30 天 | 成本可控 |
| 错误请求 | 90 天+ | 复盘需要 |
| 采样 Trace | 7 天 | 已抽样 |
| 评测样本 | 长期 | 回归基准 |

## 8.3 索引与查询性能

结论先行：合理的索引决定查询速度，属性设计影响可检索性。

机制拆解：`traceId`、服务名、时间、状态应建索引；自由属性按需建索引。

边界与坑：为所有属性建索引会拖慢写入并增加存储。

生产建议：只对高频查询属性建索引，其余用全文检索兜底。

# 9. Trace 的查询

结论先行：查询维度决定排障效率，最常用的是按 `traceId`、按服务时间、按属性过滤。

## 9.1 查询维度

结论先行：不同维度服务不同场景。

| 查询维度 | 用途 |
|----------|------|
| traceId | 追查单次请求 |
| 服务 + 时间 | 查某时间段 |
| 属性过滤 | 找特定模型/工具 |
| 耗时 > X | 找慢请求 |
| 状态 = ERROR | 找失败 |

## 9.2 典型排查流程

结论先行：从"错误与慢请求"入手，逐层下钻到根因 Span。

机制拆解：先按状态或耗时筛选，再打开 Trace 看最长 Span，最后看该 Span 的输入属性。

```yaml
query:
  filters:
    - field: status
      op: eq
      value: ERROR
    - field: llm.model
      op: eq
      value: gpt-4o
  sort:
    - field: duration
      order: desc
```

边界与坑：只看总耗时不看子 Span，容易误判瓶颈所在。

生产建议：为高频排查场景保存查询模板。

# 10. 生产注意事项

结论先行：分布式追踪的失败多源于上下文丢失、采样失当与敏感泄露。

## 10.1 常见风险

结论先行：风险覆盖传播、采样、存储与安全四方面。

| 风险 | 建议 |
|------|------|
| 上下文丢失 | 显式传递（异步/响应式） |
| 无 traceId 贯穿 | 网关生成并透传 |
| 全量采样 | 比例 + 错误全采 |
| 属性含敏感信息 | 脱敏 |
| 保留期过长 | 按需设置 |
| 无错误标记 | 显式设置 status |

## 10.2 检查清单

结论先行：上线前逐项确认链路完整、采样正确、数据安全。

| 检查项 | 通过标准 |
|--------|----------|
| traceId 贯穿 | 全链路一致 |
| 异步传递 | 已 wrap/桥接 |
| 采样一致 | 同 Trace 同决策 |
| 错误标记 | status=ERROR 已设 |
| 敏感属性 | 已脱敏 |
| 保留策略 | 分层设置 |

边界与坑：清单无人复核会腐化，建议纳入 CI。

生产建议：为关键链路编写端到端追踪测试。

# 11. 小结

- Trace 是调用链，Span 是链路上的操作。
- 上下文传播是关键，异步/响应式/线程池/消息场景易丢失。
- 采样用"比例 + 错误全采"，同一 Trace 决策必须一致。
- 服务间用 W3C traceparent 标准透传。
- Trace 存储与保留期按成本与需求分层权衡。
- 查询从错误与慢请求入手，逐层下钻到根因 Span。
