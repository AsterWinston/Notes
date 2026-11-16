日志是最古老的可观测信号，也是最容易失控的一个：一条 `fmt.Printf` 就能上线，但当天日志量涨到每天几 TB 时，成本和检索都会变成灾难。OTel Logs 想解决的不是"怎么打日志"，而是**怎么让日志结构化、带上链路上下文、并和其他信号走同一条采集管道**。

# 1. 从字符串日志到结构化日志

## 1.1 字符串日志的问题

```go
log.Printf("user %s created order %s cost %dms", userID, orderID, cost)
```

这行日志对人友好，对机器不友好：

- 想按 `order_id` 检索，只能正则匹配。
- 字段顺序一变，解析规则就废。
- 无法稳定提取耗时做聚合。
- 多语言、多服务下格式各不相同。

结构化日志把字段独立出来：

```json
{"time":"2026-11-06T10:00:00Z","level":"INFO","msg":"order created","user_id":"u-42","order_id":"A-1001","cost_ms":86}
```

## 1.2 log/slog 最小示例

Go 1.21 起标准库自带 `log/slog`，是 OTel 桥接的首选：

```go
func main() {
	// JSON handler 便于采集；开发期可换 TextHandler 提高可读性
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo, // 生产可调到 Warn，直接减少日志量
	}))
	logger.Info("order created", "order_id", "A-1001", "cost_ms", 86)
}
```

输出字段与入参一一对应：`{"level":"INFO","msg":"order created","order_id":"A-1001","cost_ms":86}`。

★ **关键认知**：结构化日志的核心不是 JSON 本身，而是**字段名稳定**。字段名是查询和告警的契约，一旦上线就不要随意改名；需要新增字段时只加不改。

# 2. OTel Logs 模型

## 2.1 LogRecord 的字段

一条 OTel 日志记录（LogRecord）包含：

| 字段 | 含义 | 备注 |
|------|------|------|
| Timestamp | 事件发生时间 | 业务代码产生日志的时刻 |
| ObservedTimestamp | 被采集系统看到的时间 | 用于发现时钟漂移和积压 |
| SeverityNumber | 数字级别（1-24） | TRACE/FATAL 等映射到固定区间 |
| SeverityText | 文本级别 | `INFO`、`ERROR` |
| Body | 日志正文 | 可以是字符串或结构化值 |
| Attributes | 结构化字段 | `order_id`、`http.route` 等 |
| Resource | 来源身份 | `service.name`、`host.name` |
| InstrumentationScope | 产生日志的库/模块 | 桥接器名称 |
| TraceID / SpanID | 关联的链路 | 有 ctx 时自动填充 |

★ **关键认知**：OTel Logs 不是"另一套日志文件"，而是**日志的传输模型**。你可以继续用 slog/zap 写日志，桥接器负责把一条日志翻译成 LogRecord，再通过 OTLP 送走。

## 2.2 LoggerProvider 与 Bridge

```text
业务代码 → slog/zap → 桥接器 → OTel Logs API
                              → LoggerProvider（Resource + Processor）
                              → OTLP → Collector / 后端
```

| 角色 | 作用 |
|------|------|
| LoggerProvider | 类似 TracerProvider，持有 Resource 和 Processor |
| Logger | 按 scope 划分的日志入口 |
| Bridge | 把第三方日志库的记录转成 OTel LogRecord |
| Processor | Simple / Batch，决定何时导出 |
| Exporter | OTLP gRPC/HTTP，或 stdout 调试 |

## 2.3 信号成熟度

Traces 和 Metrics 在 Go SDK 1.x 早期就稳定了，Logs 更晚：大约从 1.34 起 Logs SDK 才进入稳定阶段，日志桥接包也经历了多轮 API 调整。落地前先确认三件事：

- `go.mod` 里 `otel/sdk/log` 的版本。
- 桥接包（如 `contrib/bridges/otelslog`）是否与 SDK 版本匹配。
- 后端是否支持 OTLP Logs，而不是只支持自家的日志协议。

# 3. 与 Trace 关联

## 3.1 InfoContext 自动注入

使用支持 OTel 的 slog handler 时，只要传了 `ctx`，桥接器就会自动提取 trace_id 和 span_id：

```go
func (s *OrderService) Create(ctx context.Context, req CreateReq) error {
	// 关键：用 InfoContext 而不是 Info，ctx 里的 span 才会被读到
	slog.InfoContext(ctx, "开始创建订单", "order_id", req.OrderID)

	if err := s.repo.Insert(ctx, req); err != nil {
		slog.ErrorContext(ctx, "订单写入失败", "order_id", req.OrderID, "error", err)
		return err
	}
	return nil
}
```

导出的 LogRecord 会多出 `TraceID: 4bf92f3577b34da6a3ce929d0e0e4736` 和 `SpanID: 00f067aa0ba902b7`。

⚠ **关键认知**：`logger.Info(...)` 和 `logger.InfoContext(ctx, ...)` 只差一个参数，但前者**不会带 trace_id**。日志里没有 trace_id，链路和日志就只能靠时间戳猜，排障效率会差很多。

## 3.2 没有桥接时手动关联

如果日志走的是文件 + Promtail/Fluent Bit，而不是 OTLP，也可以手动把 ID 写进字段：

```go
sc := trace.SpanContextFromContext(ctx)
if sc.IsValid() {
	slog.Info("order created", "order_id", orderID,
		"trace_id", sc.TraceID().String(),
		"span_id", sc.SpanID().String())
}
```

采集端再把这些字段解析出来。注意：**trace_id 不要做成 Loki 的标签**，它是高基数字段，会让索引爆炸；应作为结构化字段或 metadata 存储，查询时用过滤表达式。

## 3.3 统一 request id 与 trace id

有些老系统已经有自己的 `request_id`，不必强行替换：

| 标识 | 作用 | 建议 |
|------|------|------|
| `trace_id` | 跨服务链路主键 | 必须有，由 OTel 生成/传播 |
| `span_id` | 当前操作节点 | 由 SDK 自动写 |
| `request_id` | 业务侧单次请求标识 | 可保留，和 trace_id 并存 |
| `user_id` | 业务主体 | 视合规要求，必要时脱敏 |

★ **关键认知**：`request_id` 是业务语义，`trace_id` 是观测语义，两者不是一回事。可以共存，但不要用 `request_id` 替代 `trace_id`，否则跨服务链路仍然串不起来。

# 4. 日志桥接

## 4.1 otelslog 桥接

```go
exporter, err := otlploggrpc.New(ctx) // 地址由 OTEL_EXPORTER_OTLP_ENDPOINT 控制
if err != nil {
	log.Fatal(err)
}
provider := sdklog.NewLoggerProvider(
	sdklog.WithResource(res),
	sdklog.WithProcessor(sdklog.NewBatchProcessor(exporter)), // 生产用 Batch
)
defer provider.Shutdown(ctx) // 生产要加超时

slog.SetDefault(slog.New(otelslog.NewHandler("order-service",
	otelslog.WithLoggerProvider(provider),
)))
```

之后所有通过 `slog.Default()` 打的日志都会进入 OTel 管道。

## 4.2 其他日志库

| 日志库 | 桥接包 | 说明 |
|--------|--------|------|
| `log/slog` | `contrib/bridges/otelslog` | 标准库，新项目首选 |
| `zap` | `contrib/bridges/otelzap` | 高性能，注意 Core 组合 |
| `zerolog` | `contrib/bridges/otelzerolog` | 零分配，字段模型不同 |
| `logrus` | `contrib/bridges/otellogrus` | 老项目常见，性能一般 |

桥接的共同点：把第三方库的字段、级别、时间戳翻译成 LogRecord；差别在字段命名和 hook 机制。选型时优先考虑团队现有栈，不要为了 OTel 换日志库。

## 4.3 双写与迁移

迁移期常见"既写 stdout 又走 OTLP"。如果同时挂两个 handler，要清楚**每条日志会被写两次**；迁移完成后及时去掉重复出口，否则成本直接翻倍。

⚠ **关键认知**：日志双写不是"多一层保险"，而是**双倍存储和双倍网络**。保留 stdout 用于容器日志采集时，就不要再用 OTLP 送同一份日志。

# 5. 采样与成本

## 5.1 先算一笔账

假设 1 万 QPS、每请求 5 条日志、每条 800 字节：

每天日志量 = `10000 × 5 × 800 × 86400 ≈ 3.46 TB`，每月约 104 TB；即使压缩 8 倍仍有十几 TB。日志成本通常比 Trace 高一个数量级，**采样和字段裁剪是必做项**。

## 5.2 分层采样

| 策略 | 做法 | 适用 |
|------|------|------|
| 级别过滤 | 生产只开 INFO，DEBUG 按需动态开启 | 所有服务 |
| 按请求采样 | 用 trace 的采样结果决定是否打成功日志 | 高 QPS 服务 |
| 错误全留 | ERROR/FATAL 不采样 | 所有服务 |
| 按路由过滤 | 健康检查、探针日志直接丢弃 | 所有服务 |
| 尾部采样 | Collector 按 trace 结果决定保留 | 需要完整链路的场景 |

按 trace 采样可以保证"一条链路要么日志完整，要么都没有"，避免只留下半条链路的碎片：

```go
if trace.SpanContextFromContext(ctx).IsSampled() {
	slog.InfoContext(ctx, "order created", "order_id", orderID)
}
```

★ **关键认知**：日志采样要和 Trace 采样**对齐**。Trace 采了 10%、日志却全量，等于用最贵的方式记录了一批没人看的成功请求；反过来 Trace 全采、日志只留 10%，链路里又缺少关键细节。

## 5.3 敏感信息

日志平台通常访问范围更广，敏感字段必须在**产生端**处理：

| 字段 | 处理 |
|------|------|
| 密码、token、cookie | 绝不记录 |
| 手机号、身份证、银行卡 | 脱敏或哈希 |
| 完整请求体/响应体 | 默认不记录，按需开关 |
| SQL 参数值 | 只记录参数化语句 |
| 内部 IP、密钥 | 按合规要求脱敏 |

⚠ **关键认知**：不要指望后端做脱敏。日志一旦发出，就已经离开了你的信任边界；脱敏必须在写日志的那一刻完成。

# 6. 最小可运行示例

先用 stdout 验证桥接是否工作，再换 OTLP：

```go
package main

import (
	"context"
	"log/slog"
	"time"

	"go.opentelemetry.io/contrib/bridges/otelslog"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdoutlog"
	"go.opentelemetry.io/otel/sdk/resource"
	sdklog "go.opentelemetry.io/otel/sdk/log"
	semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func main() {
	ctx := context.Background()
	exp, _ := stdoutlog.New(stdoutlog.WithPrettyPrint())
	res, _ := resource.Merge(resource.Default(), resource.NewWithAttributes(
		semconv.SchemaURL, semconv.ServiceName("logs-demo")))

	provider := sdklog.NewLoggerProvider(
		sdklog.WithResource(res),
		sdklog.WithProcessor(sdklog.NewBatchProcessor(exp)),
	)
	defer func() {
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
		defer cancel()
		_ = provider.Shutdown(shutdownCtx) // 关闭时 flush，最后一批日志才不丢
	}()

	slog.SetDefault(slog.New(otelslog.NewHandler("logs-demo",
		otelslog.WithLoggerProvider(provider))))

	// 建一个 span，验证日志能自动带上 trace_id
	ctx, span := otel.Tracer("logs-demo").Start(ctx, "create-order")
	defer span.End()
	slog.InfoContext(ctx, "订单创建成功", "order_id", "A-1001")
}
```

```bash
go get go.opentelemetry.io/otel/sdk/log@latest
go get go.opentelemetry.io/contrib/bridges/otelslog@latest
go get go.opentelemetry.io/otel/exporters/stdout/stdoutlog@latest
go run .
```

输出里应能看到 `TraceID` 和 `SpanID` 与刚才的 span 一致。如果 `TraceID` 为空，先检查是不是误用了 `logger.Info`。

# 7. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 日志没有 trace_id | 用了不带 ctx 的日志方法 | 改 `InfoContext`/`ErrorContext` |
| OTel 里一条日志都没有 | 桥接器没挂、Provider 没导出、进程提前退出 | 检查 handler、exporter、`Shutdown` |
| trace_id 成了 Loki 标签 | 把高基数字段当索引 | 改为结构化字段，查询时过滤 |
| 日志量突然翻倍 | stdout 与 OTLP 双写 | 只保留一条出口 |
| 日志延迟很大 | Batch 队列积压或后端限流 | 看队列指标，扩容或降采样 |
| 关键日志缺失 | 采样把 ERROR 也采掉了 | 错误日志强制保留 |
| 信息不够排障 | 只打了消息，没打业务字段 | 补 `order_id`、`request_id`、耗时 |
| 磁盘被日志打满 | 热路径循环里打 INFO | 降级日志、限流、改采样 |

# 8. 练习与自检

1. 把一段使用 `log.Printf` 的代码改成 `slog` + `otelslog`，要求每条日志都带 `trace_id`，并验证同一条链路下的多条日志 trace_id 一致。
2. 给一个 1 万 QPS 的服务估算日志成本：每请求 3 条、每条 600 字节、保留 30 天。再计算按 trace 10% 采样后的成本。
3. 设计一套日志字段规范，至少包含：时间、级别、服务名、trace_id、span_id、业务主键、错误信息。说明哪些字段不能做索引标签。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 结构化 | 字段名稳定，JSON 便于采集，避免拼接字符串 |
| LogRecord | Timestamp、Severity、Body、Attributes、TraceID/SpanID、Resource |
| 关联 | 用 `InfoContext`，trace_id 由 ctx 自动注入 |
| 桥接 | otelslog/otelzap 等把现有日志接入 OTel，不必换库 |
| 成本 | 日志最贵，级别过滤 + trace 对齐采样 + 字段裁剪 |
| 安全 | 脱敏在产生端完成，密码/token 绝不落日志 |

下一篇进入 SDK 初始化与 Resource：Provider 应该在哪创建、服务名和环境怎么注入、BatchSpanProcessor 的队列和超时怎么配、采样器怎么选，以及进程退出时怎样优雅关闭而不丢最后一批数据。
