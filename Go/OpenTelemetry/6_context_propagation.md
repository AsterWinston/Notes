分布式链路最容易出现的问题不是"span 不会建"，而是"链子断了"：每个服务都有数据，后端里却是一条条孤立的短线。根因几乎都在传播——`ctx` 没传、header 没注入、客户端没装埋点。本篇把 Context、W3C TraceContext、Baggage 和 HTTP/gRPC 透传讲成一条完整的链路。

# 1. Context 是传播的载体

## 1.1 context.Context 回顾

`context.Context` 在 Go 里承载三件事：

- 取消信号（`Done()`）。
- 超时/截止时间（`Deadline()`）。
- 请求级键值（`Value(key)`）。

OTel 没有另造一套上下文，而是把 `Span` 和 `Baggage` 放进 `context.Context` 的 value 里。因此**传 ctx 就是传链路**。

## 1.2 OTel 在 Context 里放了什么

| 内容 | 写入 | 读取 |
|------|------|------|
| 当前 Span | `tracer.Start` 返回的新 ctx | `trace.SpanFromContext(ctx)` |
| SpanContext | 随 Span 一起 | `trace.SpanContextFromContext(ctx)` |
| Baggage | `baggage.ContextWithBaggage` | `baggage.FromContext(ctx)` |

```go
// Start 把新 span 放进 ctx；不接住返回值，父子关系就断了
ctx, span := tracer.Start(ctx, "handle")
defer span.End()

sc := trace.SpanContextFromContext(ctx)
fmt.Println(sc.TraceID(), sc.SpanID(), sc.IsSampled())
```

## 1.3 两条传递规则

```go
// 规则一：函数第一个参数是 ctx，且原样往下传
func (s *Service) Create(ctx context.Context, req Req) error {
	return s.repo.Insert(ctx, req) // 不要在这里换成 context.Background()
}

// 规则二：需要派生时用 WithCancel/WithTimeout/WithValue，保留父 ctx
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()
```

★ **关键认知**：链路断掉的头号原因就是**在某一层把 ctx 换成了 `context.Background()`**。日志、超时、追踪全都会断，而且往往在 code review 里看不出来。

# 2. W3C TraceContext

## 2.1 traceparent 的格式

OTel 默认使用 W3C TraceContext 标准，HTTP header 名是 `traceparent`：

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                │
             │  │                                │                └─ flags：01 表示已采样
             │  │                                └─ 当前 span_id（16 位十六进制）
             │  └─ trace_id（32 位十六进制）
             └─ version：00
```

| 部分 | 长度 | 说明 |
|------|------|------|
| version | 2 位十六进制 | 当前为 `00` |
| trace-id | 32 位小写十六进制 | 全链路不变 |
| parent-id | 16 位小写十六进制 | 即调用方的 span_id，每跳变化 |
| trace-flags | 2 位十六进制 | 最低位是 sampled 标志 |

★ **关键认知**：跨服务时 **trace_id 不变，span_id 每跳都变**。下游的 `parent-id` 就是上游的 `span-id`，后端靠这个字段把父子关系拼起来。

## 2.2 tracestate

`tracestate` 用来携带厂商扩展信息，格式是逗号分隔的键值对：

```text
tracestate: congo=t61rcWkgMzE,rojo=00f067aa0ba902b7
```

规则要点：

- 最多 32 个成员。
- 键和值有字符集限制，不能随意塞 JSON。
- 中间件应**原样透传**，不要解析后重写。
- 各厂商的键有优先级，通常最新的放最前。

一般业务不需要直接操作 `tracestate`，SDK 会自动传递。只有在对接特定 APM 的扩展字段时才需要关注。

## 2.3 采样标志

`trace-flags=01` 表示这条 trace 已被采样，下游应继续记录：

```text
网关决定采样 → traceparent 里 flags=01
   │
   ├─ 订单服务读到 01，ParentBased 直接记录
   └─ 库存服务读到 01，同样记录
```

如果 flags 是 `00`，下游仍会生成 SpanContext 并继续传播，但 `IsRecording()` 为 false，不会导出 span。这样既保留了链路结构，又节省了存储。

## 2.4 合法性校验

解析时以下情况会被判为非法，通常直接丢弃并按"无父 span"处理：

| 情况 | 结果 |
|------|------|
| trace-id 全为 0 | 非法 |
| parent-id 全为 0 | 非法 |
| version 为 `ff` | 非法 |
| 长度不足、含非十六进制字符 | 非法 |
| version 大于 `00` 且长度更长 | 解析前 55 位，忽略扩展字段 |

⚠ **关键认知**：不要自己用字符串拼接 `traceparent`。格式、大小写、全零校验、版本兼容都有细节，统一交给 `propagation.TraceContext{}`。

# 3. Baggage

## 3.1 Baggage 是什么

Baggage 是随请求传播的**业务键值对**，适合放租户 ID、灰度标签、调用来源这类跨服务都要用的信息：

```text
baggage: tenant=acme,region=cn,canary=true
```

它和 Span 属性的区别：

| 维度 | Baggage | Span Attributes |
|------|---------|-----------------|
| 传播范围 | 跨进程、跨服务 | 只属于当前 Span |
| 是否自动透传 | 是，随 header 走 | 否 |
| 存储 | 每个服务都能读到 | 只存在当前 span |
| 用途 | 业务上下文传递 | 观测与检索 |

## 3.2 API 用法

```go
// 写入
bag, err := baggage.Parse("tenant=acme,region=cn")
if err != nil {
	return err
}
ctx = baggage.ContextWithBaggage(ctx, bag)

// 读取
bag = baggage.FromContext(ctx)
tenant := bag.Member("tenant").Value() // "acme"

// 动态添加
bag, _ = bag.SetMember(baggage.NewMember("canary", "true"))
ctx = baggage.ContextWithBaggage(ctx, bag)
```

传播器要同时注册 TraceContext 和 Baggage：

```go
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
	propagation.TraceContext{},
	propagation.Baggage{},
))
```

★ **关键认知**：Baggage 不会自动变成 Span 属性。想让后端能按租户筛选链路，需要在创建 span 时显式读取并 `SetAttributes`，否则它只是"在 ctx 里路过"。

## 3.3 限制与安全

| 风险 | 说明 | 建议 |
|------|------|------|
| 头部膨胀 | 每个请求都带，跨服务累积 | 只放必要字段，控制大小 |
| 敏感信息泄露 | 中间件、代理、日志都能看到 | 不放 token、密码、身份证 |
| 被伪造 | 客户端可自行设置 header | 不用于鉴权，服务端要重新校验 |
| 规范限制 | 建议总大小不超过 8KB | 超限可能被截断或拒绝 |

⚠ **关键认知**：Baggage 是**明文传输**的，且客户端可以随意伪造。它适合做灰度标记、租户提示，绝不能当作可信身份或权限凭据。

# 4. HTTP 透传

## 4.1 服务端：otelhttp

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

handler := otelhttp.NewHandler(mux, "order-api",
	otelhttp.WithTracerProvider(tp),
	otelhttp.WithPropagators(propagator), // 默认用全局 propagator
)
http.ListenAndServe(":8080", handler)
```

`otelhttp.NewHandler` 会自动：

1. 从请求头 Extract 出父 SpanContext。
2. 创建 `SpanKindServer` 的 span。
3. 把新 ctx 放进 `r.Context()`。
4. 记录状态码、方法、路由等属性。
5. 请求结束时 `End`。

Handler 内部拿到的 `r.Context()` 已经带 span：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	span := trace.SpanFromContext(ctx)
	span.SetAttributes(attribute.String("order.id", r.URL.Query().Get("id")))
}
```

## 4.2 客户端：otelhttp.Transport

```go
client := &http.Client{
	Transport: otelhttp.NewTransport(http.DefaultTransport),
}

// 必须用带 ctx 的构造函数，否则请求没有父 span 可继承
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
	return err
}
resp, err := client.Do(req)
```

`otelhttp.NewTransport` 会自动 Inject `traceparent`、`tracestate` 和 `baggage`，并创建 `SpanKindClient` 的 span。

⚠ **关键认知**：只包装 Transport 还不够，**请求必须用 `NewRequestWithContext` 创建**。用 `http.NewRequest` 再 `req = req.WithContext(ctx)` 也可以，但漏掉这一步，客户端 span 就会变成新的根。

## 4.3 手动 Inject / Extract

没有自动埋点的场景（消息队列、自定义协议）需要手动传播：

```go
// 发送端：把 ctx 里的 trace 信息写进 carrier
carrier := propagation.MapCarrier{}
propagator.Inject(ctx, carrier)
// carrier 里会有 traceparent / tracestate / baggage

// 接收端：从 carrier 还原 ctx
ctx = propagator.Extract(context.Background(), carrier)
```

HTTP 场景的 carrier 是 `propagation.HeaderCarrier`，底层就是 `http.Header`：

```go
propagator.Inject(ctx, propagation.HeaderCarrier(req.Header))
ctx = propagator.Extract(r.Context(), propagation.HeaderCarrier(r.Header))
```

★ **关键认知**：`Inject` 和 `Extract` 必须使用**同一个 propagator 配置**。发送端注入了 Baggage，接收端只注册了 TraceContext，Baggage 就会静默丢失。

# 5. gRPC 透传

## 5.1 StatsHandler（推荐）

新版本 otelgrpc 用 stats handler，挂在连接层：

```go
// 服务端
srv := grpc.NewServer(
	grpc.StatsHandler(otelgrpc.NewServerHandler()),
	grpc.ChainUnaryInterceptor(Recovery(), Auth()),
)

// 客户端
conn, err := grpc.NewClient(addr,
	grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

stats handler 会在每个 RPC 上创建 span，并自动从 metadata 中提取/注入 `traceparent` 与 `baggage`。

## 5.2 metadata 是 gRPC 的 header

gRPC 的透传载体是 `metadata.MD`，本质是 `map[string][]string`，键会被转成小写：

```go
md, ok := metadata.FromIncomingContext(ctx)
if ok {
	// traceparent 在 metadata 里的键是小写的
	vals := md.Get("traceparent")
	_ = vals
}
```

不要在业务代码里手动拼 `traceparent`。要么用 stats handler，要么用 propagator 配合自定义 carrier：

```go
type mdCarrier metadata.MD

func (c mdCarrier) Get(key string) string {
	if v := metadata.MD(c).Get(key); len(v) > 0 {
		return v[0]
	}
	return ""
}
func (c mdCarrier) Set(key, value string) { metadata.MD(c).Set(key, value) }
func (c mdCarrier) Keys() []string {
	keys := make([]string, 0, len(c))
	for k := range c {
		keys = append(keys, k)
	}
	return keys
}
```

## 5.3 流式调用

流式 RPC 的 span 由 stats handler 统一处理，和 Unary 一样自动传播。手写拦截器时要注意：流式拦截器拿不到独立 ctx，必须通过 `ss.Context()` 取，并在包装 `ServerStream` 时覆写 `Context()`（参考 gRPC 拦截器那篇）。

⚠ **关键认知**：**stats handler 和 otelgrpc 拦截器不要同时注册**。两者都会创建 span，结果是每条 RPC 出现重复的 Server/Client span，后端统计直接翻倍。

# 6. 跨服务串联

## 6.1 一条正确的链路

```text
浏览器 / 客户端
  │ traceparent: 00-T1-S1-01
  ▼
┌─────────────┐
│ gateway      │  Extract 得到 parent=S1，创建 S2，trace_id=T1
│              │  出站 Inject：00-T1-S2-01
└──────┬──────┘
       │ HTTP
       ▼
┌─────────────┐
│ order        │  Extract 得到 parent=S2，创建 S3，trace_id=T1
│              │  出站 Inject：00-T1-S3-01
└──────┬──────┘
       │ gRPC
       ▼
┌─────────────┐
│ inventory    │  Extract 得到 parent=S3，创建 S4，trace_id=T1
└─────────────┘

所有服务 trace_id 都是 T1，span_id 各不相同。
```

## 6.2 断链原因速查

| 断链位置 | 典型写法 | 修复 |
|---------|---------|------|
| Handler 内部 | `ctx := context.Background()` | 用 `r.Context()` 或 `c.Request.Context()` |
| 异步 goroutine | `go func() { use(context.Background()) }()` | 把 ctx 作为参数传进 goroutine |
| HTTP 客户端 | 直接用 `http.DefaultClient` | 包装 `otelhttp.NewTransport` |
| 请求构造 | `http.NewRequest` 没带 ctx | `http.NewRequestWithContext` |
| gRPC 客户端 | 没挂 stats handler | `grpc.WithStatsHandler` |
| 未注册传播器 | 只 `SetTracerProvider` | 同时 `SetTextMapPropagator` |
| 反向代理 | 白名单过滤了 `traceparent` | 显式放行 trace 相关 header |
| 消息队列 | 消息头没带 trace 信息 | Producer Inject、Consumer Extract |
| 强制新根 | 误用 `trace.WithNewRoot()` | 删除，或确认确实要新链路 |
| Gin 中间件顺序 | `r.Use(otelgin...)` 写在路由注册之后 | 在注册路由前 `Use` |

★ **关键认知**：排查断链要**从上游往下游查**：先确认上游出站请求头里有没有 `traceparent`，再确认下游入站有没有 Extract 成功，最后看下游出的请求有没有继续 Inject。只在后端看"为什么没有父子关系"，很难定位是哪一跳丢的。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 每个服务一条独立 trace | header 没传或没提取 | 检查出入站埋点和 propagator |
| 下游 span 的 trace_id 变了 | 下游误用 `WithNewRoot` 或新建了 trace | 去掉强制新根 |
| 日志有 trace_id 但链路对不上 | 日志 ctx 和请求 ctx 不是同一个 | 统一用请求 ctx |
| gRPC 有 span 但无父子关系 | 只注册了拦截器没处理 metadata | 用 `otelgrpc.NewServerHandler` |
| Baggage 丢失 | 只注册了 TraceContext | 加 `propagation.Baggage{}` |
| 跨网关后链路断 | 代理剥离/重写 header | 配置 header 透传白名单 |
| 同一条 RPC 出现两个 span | stats handler 与拦截器重复 | 二选一 |
| 采样标志不一致 | 下游独立采样器比例不同 | 统一 `ParentBased` |

# 8. 练习与自检

1. 起两个 HTTP 服务：A 用 `otelhttp.NewTransport` 调用 B，B 用 `otelhttp.NewHandler` 接收。用 curl 带一个手工构造的 `traceparent` 请求 A，验证 B 的 span 是否复用同一个 trace_id。
2. 在 A 调用 B 之前加一行 `ctx = context.Background()`，观察链路变化；再改成把 ctx 放进 goroutine 参数，确认恢复。
3. 用 `baggage.Parse("tenant=acme,canary=true")` 写入 ctx，经 HTTP 传到 B，在 B 中读取并 `SetAttributes`，验证 Baggage 能跨服务但不自动成为 span 属性。

# 9. 小结

| 主题 | 要点 |
|------|------|
| Context | 传 ctx 就是传链路，禁止中途换成 Background |
| traceparent | `version-traceid-spanid-flags`，trace_id 全程不变 |
| tracestate | 厂商扩展，原样透传，不要重写 |
| Baggage | 明文业务键值，可伪造，不放敏感信息 |
| HTTP | 服务端 `otelhttp.NewHandler`，客户端 `otelhttp.NewTransport` + 带 ctx 的请求 |
| gRPC | 用 stats handler，避免和拦截器重复埋点 |
| 排错 | 从上游出站 header 开始，逐跳确认 Inject/Extract |

下一篇进入 Collector：为什么生产环境不该让 SDK 直连后端、Receiver/Processor/Exporter 各自负责什么、agent 与 gateway 怎么部署，以及批处理、队列、重试和 tail sampling 的坑。
