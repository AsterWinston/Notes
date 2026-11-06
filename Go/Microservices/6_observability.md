服务一拆多，排障就从"看一个日志文件"变成"在十个进程里找同一个请求"。可观测性不是上线后再补的监控面板，而是服务设计的一部分：日志回答"发生了什么"，指标回答"整体是否健康"，追踪回答"这一个请求经过了谁"。本篇把三件套和请求 ID 串起来，给出 Go 服务可落地的最小接入。

# 1. 三件套各管什么

## 1.1 日志

日志是**离散事件的记录**，适合回答"这个订单为什么失败"。一条好日志要包含：时间、级别、服务名、追踪 ID、业务主键、错误详情。缺了业务主键，日志就无法和具体请求关联。

## 1.2 指标

指标是**可聚合的数值**，适合回答"错误率是否升高""P99 延迟多少"。它不记录单个请求，只记录计数和分布，所以存储成本低、查询快，适合做告警。

## 1.3 追踪

追踪是**一次请求的完整路径**，适合回答"慢在哪一跳"。

```text
Trace 4bf92f3577b34da6
├── API Gateway          12ms
│   └── order-service    180ms
│       ├── MySQL query   95ms
│       └── inventory RPC 60ms  ← 慢在这里
```

## 1.4 三者协作

| 场景 | 先用 | 再用 | 最后 |
|------|------|------|------|
| 收到告警"错误率高" | 指标定位服务与时间 | 追踪找失败链路 | 日志看具体错误 |
| 用户反馈"下单很慢" | 追踪看耗时分布 | 指标确认是否普遍 | 日志看慢查询参数 |

★ **关键认知**：三件套不是三套独立系统，而是**通过 trace_id 和资源属性互相跳转**。指标上的一个尖刺，应该能一键跳到对应时间段的追踪，再跳到具体日志。设计时先定好关联字段，比堆工具更重要。

# 2. 请求 ID

## 2.1 生成与透传

请求 ID 是贯穿全链路的"身份证"。入口生成，之后每一跳都透传，日志和追踪都带上它。

```go
func ServerRequestID() middleware.Middleware {
	return func(next middleware.Handler) middleware.Handler {
		return func(ctx context.Context, req any) (any, error) {
			md, _ := metadata.FromServerContext(ctx)
			if id := md.Get("x-request-id"); id == "" {
				md = metadata.Metadata{"x-request-id": {uuid.NewString()}}
				ctx = metadata.NewServerContext(ctx, md)
			}
			return next(ctx, req)
		}
	}
}
```

下游调用时用 `metadata.AppendToClientContext(ctx, "x-request-id", id)` 把它写进 gRPC metadata。

## 2.2 日志关联

```go
logger := log.With(log.NewStdLogger(os.Stdout),
	"ts", log.DefaultTimestamp,
	"service", "order-service",
	"trace.id", tracing.TraceID(),
	"span.id", tracing.SpanID(),
)
```

Kratos 的 `tracing.TraceID()` 是动态值，打印时会从 context 里取当前的 trace id。这样同一个请求的所有日志自动带上同一个 ID。

## 2.3 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 入口不生成，依赖上游 | 外部请求没有 ID | 网关统一生成 |
| 透传时覆盖已有 ID | 链路断开 | 有则复用，无则生成 |
| 只传 HTTP 头，不传 gRPC metadata | 跨协议丢失 | 两种协议都处理 |

⚠ **关键认知**：请求 ID 和 trace_id 不是一回事。trace_id 由追踪系统生成，可能因为采样而不存在；请求 ID 必须**100% 存在**，因为它还要用于对账、客服查询和日志检索。高要求场景两个都留。

# 3. 结构化日志

## 3.1 Kratos 日志接口

Kratos 的 `log.Logger` 是接口，可以换实现：

```go
logger := log.With(log.NewStdLogger(os.Stdout),
	"ts", log.DefaultTimestamp, "service", "order-service",
	"trace.id", tracing.TraceID(), "span.id", tracing.SpanID())
```

## 3.2 接入 Zap

```go
func NewLogger() log.Logger {
	cfg := zap.NewProductionConfig()
	cfg.EncoderConfig.TimeKey = "ts"
	cfg.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
	l, _ := cfg.Build()
	return kratoszap.NewLogger(l)
}
```

生产环境输出 JSON，方便 Filebeat/Fluent Bit 采集；本地开发可以用 `zap.NewDevelopment()` 输出彩色文本。

## 3.3 字段规范

| 字段 | 类型 | 说明 |
|------|------|------|
| `ts` | string | ISO8601 时间 |
| `level` | string | debug/info/warn/error |
| `trace_id` | string | 追踪 ID |
| `msg` | string | 简短描述，不拼接变量 |
| `order_id` | number | 业务主键，按需添加 |
| `err` | string | 错误信息 |

★ **关键认知**：日志消息要**固定**，变量放字段。写成 `"order 123 created"` 会让检索和聚合变得困难；写成 `msg="order created", order_id=123`，才能按字段过滤和统计。

# 4. 指标

## 4.1 四类黄金指标

| 指标 | 含义 | 示例 |
|------|------|------|
| 延迟 | 请求耗时分布 | P50/P95/P99 |
| 流量 | 请求量 | QPS |
| 错误 | 失败比例 | 5xx 占比 |
| 饱和度 | 资源使用 | CPU、连接池、队列长度 |

## 4.2 Prometheus 接入

```go
var httpRequests = prometheus.NewCounterVec(
	prometheus.CounterOpts{Name: "http_requests_total"},
	[]string{"service", "method", "path", "code"},
)
var httpDuration = prometheus.NewHistogramVec(
	prometheus.HistogramOpts{
		Name:    "http_request_duration_seconds",
		Buckets: []float64{0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5},
	},
	[]string{"service", "method", "path"},
)

func Metrics(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		httpRequests.WithLabelValues("order", r.Method, r.URL.Path, "200").Inc()
		httpDuration.WithLabelValues("order", r.Method, r.URL.Path).Observe(time.Since(start).Seconds())
	})
}
```

正式代码还要注册变量、记录真实状态码，并用 `http.Handle("/metrics", promhttp.Handler())` 暴露抓取端点。

## 4.3 指标命名

| 规则 | 示例 | 说明 |
|------|------|------|
| 用 snake_case | `order_created_total` | Prometheus 约定 |
| 计数器以 `_total` 结尾 | `http_requests_total` | 语义清晰 |
| 耗时用秒 | `_duration_seconds` | 统一单位 |
| 标签基数要低 | `code="500"` | 不要把用户 ID 当标签 |

⚠ **关键认知**：**标签基数（cardinality）是指标的头号杀手**。把 `user_id`、`order_id`、原始 URL 放进标签，会生成百万级时间序列，直接压垮 Prometheus。路径要用路由模板 `/v1/orders/{id}`，不要用真实路径。

# 5. 链路追踪

## 5.1 Trace 与 Span

```text
Trace（一次请求）
├── Span: GET /v1/orders/123        [0ms ───────────── 210ms]
│   ├── Span: auth.check            [2ms ── 12ms]
│   ├── Span: order.Get             [15ms ──────── 150ms]
│   │   └── Span: mysql.query       [20ms ─────── 110ms]
│   └── Span: inventory.Deduct      [155ms ── 205ms]
```

- **Trace** 由 trace_id 标识，全局唯一。
- **Span** 表示一个工作单元，带 span_id、parent_span_id、开始时间、耗时、属性、状态。
- 父子关系通过 `parent_span_id` 建立，最终拼成一棵树。

## 5.2 OpenTelemetry 初始化

```go
func InitTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
	exp, err := otlptracegrpc.New(ctx,
		otlptracegrpc.WithEndpoint("localhost:4317"),
		otlptracegrpc.WithInsecure())
	if err != nil {
		return nil, err
	}
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exp),
		sdktrace.WithResource(resource.NewWithAttributes(
			semconv.SchemaURL, semconv.ServiceName("order-service"))),
		sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, propagation.Baggage{}))
	return tp, nil
}
```

退出前要 `tp.Shutdown(ctx)`，否则缓冲区里的 span 会丢失。

## 5.3 Kratos 中间件

```go
import "github.com/go-kratos/kratos/v2/middleware/tracing"

srv := http.NewServer(
	http.Middleware(
		recovery.Recovery(),
		tracing.Server(), // 自动创建 span 并注入 trace id
		logging.Server(logger),
	),
)
```

Kratos 的 `tracing.Server()` 使用 OTel 全局 TracerProvider，所以只要初始化时设置了 `otel.SetTracerProvider(tp)`，链路就会自动上报。

手动埋点：

```go
ctx, span := otel.Tracer("order-service").Start(ctx, "OrderUsecase.Create")
defer span.End()
span.SetAttributes(attribute.Int64("order.id", o.ID))
if err := uc.repo.Save(ctx, o); err != nil {
	span.RecordError(err)
	span.SetStatus(codes.Error, err.Error())
	return err
}
```

★ **关键认知**：Span 应该围绕**有意义的业务动作**创建，比如"扣库存""生成订单"，而不是每行代码一个。Span 太细会让链路图变成噪声，太粗又定位不到瓶颈。一般一个 RPC 方法、一个数据库操作、一个外部调用各一个 span。

## 5.4 采样

| 策略 | 做法 | 适用 |
|------|------|------|
| 头部采样 | 入口按比例决定 | 高流量、成本敏感 |
| 尾部采样 | Collector 看完整链路再决定 | 保留错误与慢请求 |
| 全量 | 100% 采集 | 低流量、排障期 |

```yaml
processors:
  tail_sampling:
    policies:
      - {name: errors, type: status_code, status_code: {status_codes: [ERROR]}}
      - {name: slow, type: latency, latency: {threshold_ms: 500}}
      - {name: baseline, type: probabilistic, probabilistic: {sampling_percentage: 5}}
```

⚠ **关键认知**：采样率不是越高越好。全量采集会让存储和网络成本线性上涨，而 1% 的随机采样又可能漏掉罕见错误。生产上常用"错误全采 + 慢请求全采 + 正常请求低比例"，这需要 Collector 的尾部采样，而不是在 SDK 里一刀切。

# 6. 一条链路的完整实践

```text
请求进入网关
  1. 生成 x-request-id（若无），启动 root span
  2. 记录访问日志，指标 +1
  3. 透传 request-id 与 traceparent 到下游
  4. 出错时 span.RecordError，日志带 trace_id
```

排查时的顺序：指标告警 → 找 trace → 看 span 耗时 → 用 trace_id 查日志 → 定位代码。

# 7. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 日志里 trace_id 为空 | 中间件顺序不对 | tracing 放在 logging 之前 |
| 链路断在下游 | 没透传 traceparent | 用 OTel propagator |
| 指标序列爆炸 | 标签用了真实 ID | 用路由模板 |
| 采样后查不到错误 | 头部随机采样 | 尾部采样保留错误 |
| 请求 ID 每跳都变 | 下游重新生成 | 有则复用 |

# 8. 练习

1. 给一个 HTTP 服务加上 `x-request-id` 中间件，调用两个下游，验证三个服务的日志里 ID 一致。
2. 用 Prometheus 记录接口的 QPS、错误率和 P99，配置一条"5xx 超过 1%"的告警规则。
3. 用 Jaeger 或 Tempo 接收 OTel 数据，制造一次慢查询，在链路图里找到最慢的 span。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 日志 | 结构化、固定 msg、变量放字段 |
| 指标 | 延迟/流量/错误/饱和度，标签基数要低 |
| 追踪 | Trace 树定位慢在哪一跳 |
| 关联 | trace_id 串起三件套，request_id 保证 100% 存在 |
| 接入 | OTel SDK + Collector，Kratos 中间件自动埋点 |

下一篇进入稳定性治理，讲超时、重试、熔断、限流、降级如何配合，以及幂等与最终一致性的落地方式。
