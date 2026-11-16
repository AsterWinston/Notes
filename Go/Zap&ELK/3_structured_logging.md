上一篇把 Zap 的组件拆开了，但组件只解决"怎么打"，真正决定日志好不好用的，是"打什么"。字段设计、请求 ID、错误表达、业务事件命名、与 Trace 的关联，这五件事决定了你半夜排障时是五分钟定位还是五小时猜谜。本篇给出可直接照抄的字段模型和落地代码。

# 1. 结构化日志的字段模型

## 1.1 三层字段

一条高质量的日志，字段可以分成三层，来源和生命周期都不同：

```text
┌─────────────────────────────────────────────────────┐
│ 第一层：固定元数据（进程级，全局不变）                 │
│   service、env、version、host、instance              │
├─────────────────────────────────────────────────────┤
│ 第二层：请求上下文（请求级，一次请求内不变）           │
│   request_id、trace_id、user_id、tenant_id、route    │
├─────────────────────────────────────────────────────┤
│ 第三层：业务字段（调用点级，每条日志不同）             │
│   order_id、sku、amount、duration_ms、error          │
└─────────────────────────────────────────────────────┘
```

三层的构造方式也不同：

```go
// 第一层：进程启动时构造一次
base, _ := zap.NewProduction()
base = base.With(
    zap.String("service", "order-api"),
    zap.String("env", os.Getenv("APP_ENV")),
    zap.String("version", buildVersion), // 由 -ldflags 注入
    zap.String("host", hostname),
)

// 第二层：每个请求构造一次
reqLogger := base.With(
    zap.String("request_id", requestID),
    zap.String("trace_id", traceID),
    zap.String("route", c.FullPath()),
)

// 第三层：每条日志按需追加
reqLogger.Info("order created",
    zap.String("order_id", order.ID),
    zap.Int64("amount_cent", order.Amount),
)
```

★ **关键认知**：三层字段的价值在于**避免重复和遗漏**。第一层保证每条日志都能回答"哪个服务、哪个版本"；第二层保证同一请求的日志能被串起来；第三层才是这次排障真正的线索。混在一起写，就会出现"每条日志都手写 service 名，结果有一条写错了"。

## 1.2 字段设计的第一原则

字段名和值的约定，建议直接对齐 **ECS（Elastic Common Schema）**，这样 Filebeat、Logstash 和 Kibana 的现成仪表盘可以直接复用：

| 语义 | ECS 字段 | 类型 | 说明 |
|------|----------|------|------|
| 服务名 | `service.name` | keyword | 聚合、过滤的主维度 |
| 环境 | `service.environment` | keyword | prod/staging/dev |
| 版本 | `service.version` | keyword | 定位发布引入的问题 |
| 日志级别 | `log.level` | keyword | 告警规则依赖它 |
| 时间 | `@timestamp` | date | ES 的默认时间字段 |
| 消息 | `message` | text | 人类可读摘要 |
| 请求 ID | `trace.id` 或自定义 `request_id` | keyword | 关键词，必须不分词 |
| 链路 ID | `trace.id` | keyword | 与 APM 对齐 |
| 用户 | `user.id` | keyword | 注意脱敏 |
| 错误 | `error.message` / `error.type` | text/keyword | 分别用于阅读和聚合 |

⚠ **关键认知**：**ID 类字段一定要是 `keyword`，不能是 `text`**。`text` 会被分词，`request_id: "abc-123"` 可能被切成 `abc` 和 `123`，聚合时一个请求会裂成多份。这个坑在 Kibana 里表现为"明明有这条日志，按 ID 聚合却对不上"。

# 2. 请求 ID

## 2.1 生成与透传

请求 ID 的规则只有三条，但每一条都不能破：

1. **入口生成**：如果上游带了 `X-Request-ID` 就复用，没带才生成。
2. **全链路透传**：HTTP 放 Header，gRPC 放 metadata，消息队列放消息头。
3. **落到每条日志**：不是只打一条"请求开始"，而是这个请求的每条日志都带。

```text
客户端 ──X-Request-ID: r-abc──► 网关 ──透传──► order-api ──透传──► user-api
                                     │              │                │
                                     ▼              ▼                ▼
                                 日志带 r-abc    日志带 r-abc     日志带 r-abc
```

## 2.2 Go 中的落地

```go
const headerRequestID = "X-Request-ID"

func RequestIDMiddleware(base *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        rid := c.GetHeader(headerRequestID)
        if rid == "" {
            rid = uuid.NewString() // 上游没带就自己生成
        }

        // 把 request_id 写回响应头，方便前端和用户报障时提供
        c.Header(headerRequestID, rid)

        // 关键：把带 request_id 的 logger 放进 context，供下游取用
        logger := base.With(zap.String("request_id", rid))
        ctx := context.WithValue(c.Request.Context(), loggerKey{}, logger)
        c.Request = c.Request.WithContext(ctx)

        c.Next()
    }
}
```

```go
// 下游任意一层都能取到同一个 logger
func (s *OrderService) Create(ctx context.Context, req CreateReq) (*Order, error) {
    logger := LoggerFrom(ctx)
    logger.Info("creating order", zap.Int("item_count", len(req.Items)))

    order, err := s.repo.Insert(ctx, req)
    if err != nil {
        logger.Error("insert order failed", zap.Error(err))
        return nil, err
    }

    logger.Info("order created", zap.String("order_id", order.ID))
    return order, nil
}
```

★ **关键认知**：**不要在每一层重新生成 request_id**。曾经见过 Service 层用 `uuid.NewString()` 覆盖了入口的值，结果同一个请求在日志里出现两个 ID，链路直接断掉。ID 只能从入口往下传，任何一层都不该自己造。

## 2.3 跨服务透传

HTTP 客户端：

```go
func NewHTTPClient(base *zap.Logger) *http.Client {
    return &http.Client{
        Transport: roundTripperFunc(func(req *http.Request) (*http.Response, error) {
            // 从 context 取 request_id 和 trace_id，写进下游请求头
            if rid := RequestIDFrom(req.Context()); rid != "" {
                req.Header.Set(headerRequestID, rid)
            }
            if tid := TraceIDFrom(req.Context()); tid != "" {
                req.Header.Set("traceparent", tid) // W3C Trace Context
            }
            return http.DefaultTransport.RoundTrip(req)
        }),
    }
}
```

gRPC 用 `metadata.NewOutgoingContext`，消息队列把 ID 放进消息属性。原则一样：**只要跨进程，就要带过去**。

# 3. 错误日志

## 3.1 最小字段集

一条能用来排障的错误日志，至少要有这些字段：

```go
logger.Error("create order failed",
    zap.String("error_type", "db_write"),      // 错误分类，便于聚合告警
    zap.String("error_code", "ORDER_DUPLICATE"), // 业务码，前端/运营能对上
    zap.Error(err),                            // 原始错误文本
    zap.String("order_id", orderID),           // 定位具体对象
    zap.String("user_id", userID),             // 定位用户
    zap.String("operation", "OrderRepo.Insert"), // 哪一步失败
)
```

| 字段 | 回答的问题 |
|------|------------|
| `error_type` | 哪一类错误？数据库、网络、校验？ |
| `error_code` | 业务上是什么原因？ |
| `error` | 原始错误说了什么？ |
| `operation` | 失败发生在哪个函数/步骤？ |
| 业务主键 | 涉及哪条数据、哪个用户？ |

## 3.2 业务错误与系统错误

| 维度 | 业务错误 | 系统错误 |
|------|----------|----------|
| 例子 | 余额不足、订单已存在 | 数据库连不上、空指针 |
| 是否预期 | 预期内 | 预期外 |
| 日志级别 | Info / Warn | Error |
| 是否告警 | 通常不告警，看趋势 | 需要告警 |
| 是否返回详情 | 可以返回业务原因 | 只返回 request_id |
| 是否带堆栈 | 不需要 | 需要 |

★ **关键认知**：业务错误用 `Warn` 而不是 `Error`，不是"轻视"，而是**保护告警通道**。如果"用户输错密码"也打 Error，Error 的基数会跟着流量波动，真正的系统故障就被噪声盖住了。

## 3.3 错误链与堆栈

Go 的错误可以层层包装，日志里要保留"从哪来、为什么"：

```go
if err := repo.Insert(ctx, o); err != nil {
    // %w 保留错误链，errors.Is / errors.As 才能往上判断
    return fmt.Errorf("insert order %s: %w", o.ID, err)
}
```

```go
// 顶层统一记录，只在这里打一次堆栈
logger.Error("create order failed",
    zap.NamedError("cause", err),
    zap.String("order_id", oid),
)
```

⚠ **关键认知**：同一条错误不要在每一层都 `Error` 一遍。三层各打一次，日志里就是三条重复的 Error，告警也会被放大三倍。正确做法是**底层包装、顶层记录**：中间层用 `fmt.Errorf("...: %w", err)`，只在能处理或能终结的地方打日志。

# 4. 业务事件

## 4.1 事件命名

业务事件是"发生了什么"的稳定契约，命名要能被人和机器同时理解：

| 好的命名 | 差的命名 | 原因 |
|----------|----------|------|
| `order_created` | `create order success` | 带空格和自然语言，难聚合 |
| `payment_failed` | `pay error!!!` | 有标点、有情绪 |
| `user_login_succeeded` | `login ok` | 语义完整、可枚举 |
| `inventory_deducted` | `deduct` | 缺少对象 |

约定：**小写 + 下划线 + 动词过去式**，并在代码里用常量收敛，避免手滑写错：

```go
const (
    EventOrderCreated     = "order_created"
    EventPaymentFailed    = "payment_failed"
    EventInventoryDeducted = "inventory_deducted"
)

logger.Info("order created",
    zap.String("event", EventOrderCreated),
    zap.String("order_id", o.ID),
    zap.Int64("amount_cent", o.Amount),
)
```

## 4.2 业务事件与技术日志

两者要分开看：

| 类型 | 目的 | 例子 | 保留期 |
|------|------|------|--------|
| 技术日志 | 排障 | 连接池耗尽、SQL 慢查询 | 7-30 天 |
| 业务事件 | 分析、审计 | 下单、支付、退款 | 180 天以上 |
| 审计日志 | 合规 | 谁改了配置、谁导出了数据 | 1-3 年，不可篡改 |

★ **关键认知**：业务事件不要埋在技术日志里靠正则捞。它们应该**单独一个索引或至少单独一个 `event` 字段**，否则做业务分析时要扫描全部技术日志，成本和准确性都很差。

## 4.3 一个完整的订单事件

```go
func (s *OrderService) Create(ctx context.Context, req CreateReq) (*Order, error) {
    logger := LoggerFrom(ctx)

    o, err := s.buildOrder(req)
    if err != nil {
        logger.Warn("order validation failed",
            zap.String("event", "order_validation_failed"),
            zap.String("reason", err.Error()),
            zap.Int("item_count", len(req.Items)),
        )
        return nil, err
    }

    if err := s.repo.Insert(ctx, o); err != nil {
        logger.Error("order persist failed",
            zap.String("event", "order_persist_failed"),
            zap.String("error_type", "db_write"),
            zap.String("order_id", o.ID),
            zap.Error(err),
        )
        return nil, err
    }

    // 成功事件：字段面向分析，不放堆栈、不放 SQL
    logger.Info("order created",
        zap.String("event", EventOrderCreated),
        zap.String("order_id", o.ID),
        zap.String("user_id", o.UserID),
        zap.Int64("amount_cent", o.Amount),
        zap.Int("item_count", len(o.Items)),
        zap.String("payment_method", o.PaymentMethod),
    )
    return o, nil
}
```

# 5. 日志与 Trace 关联

## 5.1 三个 ID 的关系

```text
trace_id：一次完整调用链（跨服务），如一次下单从网关到库存
  └─ span_id：链路上的一个操作（一次 RPC、一次 SQL）
       └─ request_id：一次入口请求，通常与根 span 一一对应
```

| ID | 作用域 | 谁生成 | 用途 |
|----|--------|--------|------|
| `trace_id` | 跨服务全链路 | 链路入口（网关/客户端） | 在 APM 里看完整调用树 |
| `span_id` | 单个操作 | 每个服务/SDK | 定位具体哪一跳慢 |
| `request_id` | 单次入口请求 | 网关或服务自身 | 用户报障时快速捞日志 |

★ **关键认知**：`request_id` 和 `trace_id` 不必是同一个值，但**必须能互相映射**。最省事的做法是入口同时生成/接收两者，并都写进日志；这样用户给你 request_id，你能在 APM 里反查 trace，也能在日志里看细节。

## 5.2 与 OpenTelemetry 集成

如果已经接入 OTel，可以从 `context` 里取当前 span 的 trace/span ID：

```go
import (
    "go.opentelemetry.io/otel/trace"
    "go.uber.org/zap/zapcore"
)

// 实现 zapcore.Core 的包装，自动给每条日志补 trace_id / span_id
type traceCore struct{ zapcore.Core }

func (c traceCore) With(fields []zapcore.Field) zapcore.Core {
    return traceCore{c.Core.With(fields)}
}

func (c traceCore) Check(e zapcore.Entry, ce *zapcore.CheckedEntry) *zapcore.CheckedEntry {
    if c.Enabled(e.Level) {
        return ce.AddCore(e, c)
    }
    return ce
}

func (c traceCore) Write(e zapcore.Entry, fields []zapcore.Field) error {
    // 这里拿不到 context，实际项目通常用 context 感知的 logger 或 OTel 的 log bridge
    return c.Core.Write(e, fields)
}

// 更简单的做法：在请求中间件里把 trace_id 预置进 logger
func TraceLogger(base *zap.Logger, ctx context.Context) *zap.Logger {
    sc := trace.SpanContextFromContext(ctx)
    if !sc.IsValid() {
        return base
    }
    return base.With(
        zap.String("trace_id", sc.TraceID().String()),
        zap.String("span_id", sc.SpanID().String()),
    )
}
```

⚠ **关键认知**：Zap 的 `Write` 拿不到 `context.Context`，所以"自动注入 trace_id"要么在请求边界预置字段，要么用 OTel 官方的日志桥接（`otelslog` / `go.opentelemetry.io/contrib/bridges`）。不要试图在 Core 里用全局变量取当前 span，那在并发下会串号。

## 5.3 日志、指标、链路的分工

| 信号 | 回答的问题 | 成本 | 保留 |
|------|------------|------|------|
| 指标 Metrics | 系统整体健康吗？错误率多少？ | 低 | 长 |
| 链路 Traces | 这个慢请求慢在哪一跳？ | 中 | 中 |
| 日志 Logs | 这个请求具体发生了什么？ | 高 | 短 |

★ **关键认知**：排障的正确顺序是**指标发现异常 → 链路定位到服务 → 日志看清细节**。指望日志承担全部职责，会导致日志量爆炸；只靠指标又看不到原因。三者靠 `trace_id` 串起来，才是完整可观测性。

# 6. 常见错误

| 问题 | 后果 | 正确做法 |
|------|------|----------|
| ID 字段用 `text` 类型 | 聚合结果错误 | 映射成 `keyword` |
| 每层重复打同一条 Error | 告警放大、日志冗余 | 底层包装、顶层记录 |
| 每条日志手写 service 名 | 漏写、写错 | 启动时 `With` 预置 |
| 业务事件靠正则从技术日志捞 | 成本高、易漏 | 单独 `event` 字段或索引 |
| request_id 中间层重新生成 | 链路断裂 | 只从入口透传 |
| 错误只写 `"failed"` | 无法定位 | 带 `error_type`、`error`、业务主键 |
| 打印完整请求体 | 泄露密码、Token | 显式挑选字段 |

# 7. 练习与自检

1. 设计一个 `payment` 服务的日志字段表，区分固定元数据、请求上下文和业务字段三层，并说明每个字段的类型（keyword/text/number/date）。
2. 下面这段代码有两个问题，请指出并修复：
   ```go
   func (s *Svc) Do(ctx context.Context) error {
       logger := zap.L().With(zap.String("request_id", uuid.NewString()))
       err := s.repo.Do(ctx)
       if err != nil {
           logger.Error("failed")
           return err
       }
       return nil
   }
   ```
3. 用户只提供了 `request_id`，你如何在 Kibana 和 APM 之间来回跳转？请写出需要的字段和查询。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 三层字段 | 元数据、请求上下文、业务字段分开构造 |
| 字段规范 | 对齐 ECS，ID 一律 keyword |
| 请求 ID | 入口生成、全链路透传、每条日志都带 |
| 错误日志 | 分类 + 业务码 + 原始错误 + 业务主键 |
| 业务事件 | 小写动词过去式，常量收敛 |
| Trace 关联 | trace_id 预置进 logger，与 request_id 可互查 |
| 排障顺序 | 指标 → 链路 → 日志 |

下一篇进入 `SugaredLogger`、采样、Hook 和动态级别，讲清什么时候该用糖、什么时候必须用强类型 API，以及采样和 Hook 在真实项目里的典型用法与误用。
