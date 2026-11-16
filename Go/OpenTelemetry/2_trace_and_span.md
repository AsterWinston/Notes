Trace 是 OTel 里最直观也最容易写错的信号。看似只是 `Start` 和 `End` 两个调用，真正决定链路质量的却是三件事：**Context 有没有正确往下传、父子关系有没有建立、Span 有没有在正确的时机结束**。本篇把 `TracerProvider`、`Tracer`、`Span` 三者的职责拆开，再走一遍 Span 的完整生命周期。

# 1. 三个核心对象

## 1.1 TracerProvider

`TracerProvider` 是进程级的"工厂 + 配置中心"，它持有：

- Resource：这个进程是谁（服务名、版本、环境）。
- Sampler：哪些 trace 需要记录。
- SpanProcessor：Span 结束后交给谁、怎么批量导出。
- SpanLimits：属性、事件、链接的数量上限。
- IDGenerator：trace_id / span_id 的生成策略。

```go
tp := sdktrace.NewTracerProvider(
	sdktrace.WithResource(res), // 服务身份
	sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
	sdktrace.WithBatcher(exporter), // 导出方式
)
otel.SetTracerProvider(tp) // 注册为全局，库才能拿到
```

★ **关键认知**：`TracerProvider` **一个进程只建一个**，在 `main` 启动时创建、退出时 `Shutdown`。每个请求新建一个 provider 会导致 exporter、后台 goroutine、连接全部泄漏，这是最典型的误用之一。

## 1.2 Tracer

`Tracer` 是从 provider 取出的埋点入口，带一个 `name`（通常是包名或模块名）和可选版本：

```go
tracer := otel.Tracer("order-service")          // 推荐：用全局 provider
// 或显式从某个 provider 取
tracer := tp.Tracer("order-service", trace.WithInstrumentationVersion("1.4.2"))
```

`Tracer` 本身很轻，创建开销可忽略，通常在包级变量或 `init` 里取一次。`name` 会体现在后端的 instrumentation scope 上，用来区分"这段 span 是哪个库/模块产生的"。

## 1.3 Span

一个 Span 表示**一次有开始和结束的操作**，它包含：

| 字段 | 含义 | 是否可变 |
|------|------|---------|
| SpanContext | trace_id、span_id、采样标志、tracestate | 不可变 |
| Name | 操作名，如 `GET /orders/:id` | 结束后不可改 |
| SpanKind | Internal/Server/Client/Producer/Consumer | 创建时定 |
| Attributes | 结构化键值对 | 结束前可增改 |
| Events | 带时间戳的事件列表 | 结束前可追加 |
| Status | Unset/Ok/Error + 描述 | 结束前可设 |
| StartTime/EndTime | 起止时间 | End 时定 |

★ **关键认知**：`SpanContext` 是"身份证"，即使采样丢弃、Span 不记录，它仍然有效并继续向下游传播；`Span` 是"录像机"，只有被采样时 `IsRecording()` 才为 true，属性和事件才会真正保存。

# 2. 创建 Span

## 2.1 最小示例

```go
func createOrder(ctx context.Context) error {
	ctx, span := otel.Tracer("order-service").Start(ctx, "create-order")
	defer span.End() // 必须配对，defer 能兜住 panic 路径

	span.SetAttributes(attribute.String("order.channel", "app"))
	return doCreate(ctx) // 用新 ctx，子操作才会挂在当前 span 下
}
```

★ **关键认知**：`Start` 返回的 `ctx` 不是可选项。**后续所有子调用必须使用这个新 ctx**，否则子 span 会挂到旧父级甚至变成新的根，链路直接断掉。

## 2.2 Start 的常用选项

```go
ctx, span := tracer.Start(ctx, "GET /orders/:id",
	trace.WithSpanKind(trace.SpanKindServer), // 标明这是服务端入口
	trace.WithAttributes(attribute.String("http.request.method", "GET")),
	trace.WithLinks(trace.Link{ // 关联另一条 trace，而不是建立父子关系
		SpanContext: trace.SpanContextFromContext(batchCtx),
	}),
)
```

| 选项 | 用途 | 注意 |
|------|------|------|
| `WithSpanKind` | 标记节点角色 | 默认 Internal，服务端/客户端务必显式设置 |
| `WithAttributes` | 创建时写属性 | 适合入口就能拿到的信息 |
| `WithLinks` | 关联其他 SpanContext | 批处理、消息消费、异步汇合 |
| `WithNewRoot` | 忽略当前 ctx 的父 span | 定时任务、消息溯源要谨慎使用 |
| `WithTimestamp` | 自定义开始时间 | 补录历史事件时才用 |

## 2.3 SpanKind 怎么选

| 类型 | 场景 | 例子 |
|------|------|------|
| `Server` | 接收请求的服务端 | HTTP handler、gRPC 服务方法 |
| `Client` | 主动发起调用 | HTTP client、gRPC client、DB 查询 |
| `Producer` | 发送消息 | Kafka producer |
| `Consumer` | 消费消息 | Kafka consumer |
| `Internal` | 进程内纯计算 | 参数校验、业务编排 |

⚠ **关键认知**：所有 span 都用默认的 Internal，后端仍能画出调用树，但**服务拓扑图会缺失**——拓扑依赖 Server/Client 的配对关系。入口和出口的 Kind 不是可选项。

# 3. 属性、事件与状态

## 3.1 Attributes

属性是 Span 上的结构化元数据，适合放**低基数、可过滤、可聚合**的字段：

```go
span.SetAttributes(
	attribute.String("order.id", "A-1001"), // 业务主键，别拿去做指标标签
	attribute.Int64("order.amount_cents", 9900),
)
```

常用类型：`String`、`Bool`、`Int`、`Int64`、`Float64`、`StringSlice`、`BoolSlice`、`Int64Slice`、`Float64Slice`。值还可以来自 `attribute.Key("custom.key").String("v")`。

⚠ **关键认知**：不要把用户 ID、手机号、完整 URL、原始 SQL 值直接塞进属性。它们既是高基数（拖垮后端索引），又可能违反隐私合规。需要定位时用 `order.id` 这类业务主键，且确认后端访问权限。

## 3.2 Events

事件是 Span 时间线上的一个标记，适合记录"发生了什么"，而不是"当前是什么"：

```go
span.AddEvent("库存扣减完成", trace.WithAttributes(
	attribute.String("sku", "S-100"),
	attribute.Int("quantity", 2),
))

span.AddEvent("等待支付回调")
```

事件和日志的区别：

| 维度 | Span Event | 日志 |
|------|-----------|------|
| 归属 | 必须依附一个 Span | 独立记录 |
| 生命周期 | 随 Span 导出，Span 丢弃则事件也没了 | 独立存储 |
| 容量 | 有数量上限，默认约 128 条 | 取决于日志系统 |
| 用途 | 关键流程节点、异常时间点 | 完整运行细节、审计 |

★ **关键认知**：Span Event **不是日志的替代品**。把每个请求的每行调试信息都写成 Event，会迅速撞上事件上限，还会把链路体积撑大。事件只放"排障时想知道时间点"的关键节点。

## 3.3 Status 与 RecordError

```go
if err := charge(ctx); err != nil {
	span.RecordError(err, trace.WithStackTrace(true)) // 记录异常事件与堆栈
	span.SetStatus(codes.Error, "扣款失败")            // 显式标记 span 失败
	return err
}
```

三个状态：

- `codes.Unset`：默认值，表示"没有特别标记"。
- `codes.Ok`：明确成功。一般不需要手动设置，除非要覆盖库的默认行为。
- `codes.Error`：失败，第二个参数是给**人看**的描述，不要塞敏感信息。

⚠ **关键认知**：`RecordError` **不会自动把 Span 标成 Error**，它只是追加一个 exception 事件。反过来，`SetStatus(codes.Error, ...)` 也不会自动记录错误详情。两者要配合使用，且**必须在 `span.End()` 之前调用**——End 之后所有写入都会被丢弃。

# 4. 父子关系

## 4.1 隐式父 Span

OTel 不在 API 上暴露 `parent` 参数，父子关系完全由 `ctx` 决定：

```go
ctx, parent := tracer.Start(ctx, "create-order")
defer parent.End()

// 只要传入的是 parent 返回的 ctx，child 的父就是 parent
childCtx, child := tracer.Start(ctx, "validate-order")
defer child.End()

// 反例：用 context.Background() 会切断父子关系
_, orphan := tracer.Start(context.Background(), "validate-order")
defer orphan.End()
```

★ **关键认知**：Span 的父子关系是**上下文传递的副产品**。你在函数签名里传 `ctx`，不只是为了超时和取消，也是在传链路。任何一次"图省事用 `context.Background()`"都会制造一条断链。

## 4.2 Link：不是父子，是关联

批处理、消息消费这类场景，一条 span 可能和多个上游 trace 有关，但不应该成为某个上游的子节点：

```go
ctx, span := tracer.Start(ctx, "consume-batch",
	trace.WithLinks(
		trace.Link{SpanContext: trace.SpanContextFromContext(msg1Ctx)},
		trace.Link{SpanContext: trace.SpanContextFromContext(msg2Ctx)},
	),
)
defer span.End()
```

Link 只建立"相关"关系，不影响父子树和耗时统计。消息队列里通常把 producer 的 SpanContext 放进消息头，consumer 消费时用 Link 关联。

## 4.3 并发子 Span

Span 的方法是并发安全的，但并发子任务要注意两点：**传 ctx、等子任务结束再结束父 span**：

```go
ctx, span := tracer.Start(ctx, "fanout")
defer span.End()

var wg sync.WaitGroup
for _, shard := range shards {
	wg.Add(1)
	go func(ctx context.Context, shard string) {
		defer wg.Done()
		_, child := tracer.Start(ctx, "query-shard")
		defer child.End()
		queryShard(ctx, shard)
	}(ctx, shard) // ctx 必须作为参数传入，不能靠闭包捕获
}
wg.Wait() // 等所有子任务结束，父 span 才 End
```

⚠ **关键认知**：不要在 goroutine 里直接捕获外层 `ctx` 变量而不作为参数传递——快速迭代中很容易在循环里踩到闭包变量问题。更重要的是，**父 span 先 End、子 span 后结束**会让链路时间线重叠，后端可能把子 span 判为异常节点。

## 4.4 跨进程的父子关系

跨进程时，父 SpanContext 通过 HTTP header 或 gRPC metadata 传递，由 propagator 注入和提取，子进程 `Start` 时从 ctx 里读到它。这部分在第 6 篇展开，这里只记住结论：**跨进程父子关系靠传播，不靠代码里手动指定**。

# 5. Span 生命周期

## 5.1 从 Start 到 End

```text
tracer.Start(ctx, name, opts...)
  │
  ├─ 1. 从 ctx 找父 SpanContext
  ├─ 2. 调 Sampler 决定是否记录
  ├─ 3. 生成 TraceID / SpanID（或继承父 TraceID）
  ├─ 4. 构造 recording 或 non-recording Span
  └─ 5. 把新 Span 放进新 ctx 并返回
        │
        ▼
SetAttributes / AddEvent / SetStatus / RecordError
        │
        ▼
span.End()
  ├─ 1. 写 EndTime
  ├─ 2. 标记 Span 不可再写
  └─ 3. 交给 SpanProcessor
         ├─ Simple：当前 goroutine 立即导出
         └─ Batch：进队列，后台批量导出
```

## 5.2 非录制 Span

```go
_, span := tracer.Start(ctx, "sampled-out")
span.SetAttributes(attribute.String("k", "v")) // 没采样时无效果，直接丢弃
span.End()
```

即使 `IsRecording()` 为 false，`span.SpanContext()` 依然有有效的 trace_id 和 span_id，也会继续传播。这正是"采样率 10%，但下游仍能串成完整链路"的原因。

## 5.3 数量限制

SDK 对 Span 有默认上限，防止单条链路把内存打爆：

| 限制 | 默认量级 | 环境变量 |
|------|---------|---------|
| 属性数 | 128 | `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` |
| 事件数 | 128 | `OTEL_SPAN_EVENT_COUNT_LIMIT` |
| 链接数 | 128 | `OTEL_SPAN_LINK_COUNT_LIMIT` |
| 属性值长度 | 不限制 | `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` |

⚠ **关键认知**：属性值长度默认不限制，意味着一个误传的大 JSON 或 base64 图片会原样进入链路，既贵又慢。生产环境建议显式设置 `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT`，并在代码层做截断。

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 子 Span 变成独立链路 | 用了 `context.Background()` 或旧 ctx | 始终把 `Start` 返回的 ctx 往下传 |
| Span 没有导出 | 忘记 `span.End()`，或进程提前退出 | `defer span.End()` + provider `Shutdown` |
| 状态改了没生效 | 在 `End()` 之后调用 | 所有写入放在 End 之前 |
| 错误标记了但没堆栈 | 只调了 `SetStatus` | 补 `RecordError(err, trace.WithStackTrace(true))` |
| 后端拓扑图缺边 | SpanKind 全是 Internal | 入口设 Server，出口设 Client |
| 事件丢失 | 事件数超过上限 | 只保留关键节点，详细内容走日志 |
| 链路体积异常大 | 属性里塞了大对象或高基数标签 | 限制长度、删除无用小字段 |
| 父子时间线重叠 | 并发子任务没 Wait | `WaitGroup` 等全部子任务结束再 End |

# 7. 练习与自检

1. 写一个三层的调用链（handler → service → repository），每层创建一个 span，用 stdout exporter 观察 `Parent.SpanID` 是否逐层对应。然后故意在 service 层改用 `context.Background()`，对比链路变化。
2. 给 `create-order` 增加一个失败分支：调用 `RecordError` 但不调用 `SetStatus`，观察后端或 stdout 中 Span 的状态；再补上 `SetStatus(codes.Error, ...)` 对比。
3. 启动两个并发子 span，一个正常 `End`，一个延迟 2 秒再 `End`，说明为什么父 span 应该等所有子任务结束。

# 8. 小结

| 主题 | 要点 |
|------|------|
| TracerProvider | 进程级单例，持有 Resource、Sampler、Processor |
| Tracer | 轻量入口，按模块命名，取一次即可复用 |
| Span | 一次操作的记录，SpanContext 不可变、其余字段结束前可写 |
| 父子关系 | 由 ctx 决定，`Start` 返回的新 ctx 必须继续往下传 |
| Link | 关联而非父子，适合批处理和消息消费 |
| 生命周期 | 采样在 Start，写入在 End 前，导出在 End 后 |
| 硬性要求 | `defer span.End()`、服务端/客户端设置 SpanKind、属性脱敏 |

下一篇进入 Metrics，讲清 Counter、UpDownCounter、Histogram、Gauge 的语义差异、属性与聚合、View 定制桶，以及如何设计一套不会把后端打爆的指标。
