SDK 初始化看起来只是几行 `NewTracerProvider`，但它决定了后面所有数据的质量：服务名错了，后端里全是 `unknown_service`；采样器配错，链路只剩半截；关闭顺序错了，最后一批 span 直接丢在内存里。本篇把 Resource、BatchSpanProcessor、Sampler 和优雅关闭串成一套可复用的初始化模板。

# 1. 初始化顺序

## 1.1 正确的启动顺序

```text
读取配置 → 构建 Resource → 创建 Exporter → 创建 Processor
→ 创建 Provider → 注册全局 → 启动服务
收到退出信号 → 停服务 → Shutdown Provider → 退出进程
```

★ **关键认知**：初始化是**一次性**的，必须发生在业务开始之前。Provider 注册得晚，早期请求就会落在 noop 实现上，直接没有数据；把 Provider 创建放进请求处理函数，则是资源泄漏。

## 1.2 为什么 Resource 最先确定

Resource 描述"这些数据是谁产生的"，会附加到每一条 span、metric、log 上。它必须在创建 Provider 之前定好，因为 Provider 创建后 Resource 就固定了。

## 1.3 全局注册与显式注入

两种用法：

```go
otel.SetTracerProvider(tp) // 方式一：注册全局，自动埋点库会直接使用
handler := otelhttp.NewHandler(mux, "api", otelhttp.WithTracerProvider(tp)) // 方式二：显式注入
```

业务代码通常用全局方式，测试里用显式注入避免互相污染。注意：**全局只能设置一次有意义的值**，重复 `SetTracerProvider` 会覆盖，但已经取出的 `Tracer` 行为取决于实现，不要依赖这种时序。

# 2. Resource

## 2.1 默认 Resource

`resource.Default()` 会带上 SDK、进程、主机等基础信息，并读取部分环境变量；如果没有设置服务名，`service.name` 的默认值是 `unknown_service:可执行文件名`。

后端里出现大量 `unknown_service`，通常就是只用了默认 Resource，没显式设置服务名。

## 2.2 常用属性

| 属性 | 常量/键 | 示例 |
|------|---------|------|
| 服务名 | `semconv.ServiceName` | `order-service` |
| 服务版本 | `semconv.ServiceVersion` | `1.4.2` |
| 部署环境 | `deployment.environment.name` | `prod`、`staging` |
| 服务命名空间 | `service.namespace` | `ecommerce` |
| 主机名 / 容器 ID | `host.name` / `container.id` | 自动探测 |

```go
res, err := resource.Merge(resource.Default(), resource.NewWithAttributes(
	semconv.SchemaURL,
	semconv.ServiceName("order-service"),
	semconv.ServiceVersion("1.4.2"),
	attribute.String("deployment.environment.name", "prod"),
))
```

`semconv` 不同版本之间常量名有调整，例如环境属性在旧版本里叫 `deployment.environment`。升级 SDK 时留意编译错误，必要时先用 `attribute.String("deployment.environment.name", ...)` 这类显式键过渡。

## 2.3 环境变量注入

生产环境推荐由部署平台注入，代码只保留兜底：

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: order-service
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: deployment.environment.name=prod,service.version=1.4.2
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://otel-collector:4317
  - name: OTEL_TRACES_SAMPLER
    value: parentbased_traceidratio
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "0.1"
```

```go
// WithFromEnv 读取 OTEL_SERVICE_NAME 与 OTEL_RESOURCE_ATTRIBUTES
res, err := resource.New(ctx,
	resource.WithFromEnv(),
	resource.WithTelemetrySDK(),
	resource.WithProcess(),
	resource.WithHost(),
)
```

★ **关键认知**：服务名、环境、版本这类"部署相关"信息应该由环境变量提供，代码里只写默认值。同一个镜像在 staging 和 prod 跑，靠环境变量区分，而不是打两个镜像。

## 2.4 Merge 的覆盖规则

`resource.Merge(base, override)` 中，`override` 会覆盖 `base` 的同名属性。常见组合是 `resource.Default()` 提供 SDK/进程/主机信息，`NewWithAttributes` 提供服务名、版本、环境。

⚠ **关键认知**：`resource.New` 只包含你显式给的检测器结果，不会自动带 `telemetry.sdk.*`。要么用 `resource.Merge(resource.Default(), ...)`，要么把 `WithTelemetrySDK()`、`WithProcess()` 等检测器都列上，否则后端会缺少排查所需的运行环境信息。

# 3. SpanProcessor 与 BatchSpanProcessor

## 3.1 Simple 与 Batch

| 处理器 | 行为 | 适用 |
|--------|------|------|
| SimpleSpanProcessor | 每个 span 结束时同步导出 | 调试、低流量、测试 |
| BatchSpanProcessor | 进队列，后台批量导出 | 生产环境默认 |

生产用 `sdktrace.WithBatcher(exporter)`；本地调试可以临时换成 `sdktrace.WithSyncer(exporter)`，立即看到每一条。

⚠ **关键认知**：`WithSyncer` 会在业务 goroutine 里同步调用 exporter。后端抖动或网络慢时，**业务请求会跟着变慢**。它只适合本地调试，不要带到生产。

## 3.2 BatchSpanProcessor 的关键参数

```go
bsp := sdktrace.NewBatchSpanProcessor(exporter,
	sdktrace.WithBatchTimeout(5*time.Second),       // 最长等多久就发一批
	sdktrace.WithMaxQueueSize(8192),                // 内存里最多排多少个 span
	sdktrace.WithMaxExportBatchSize(1024),          // 单批最多多少个
	sdktrace.WithExportTimeout(10*time.Second),     // 单次导出超时
)
```

| 参数 | 默认值 | 作用 | 调大/调小的代价 |
|------|--------|------|----------------|
| `MaxQueueSize` | 2048 | 等待导出的 span 上限 | 调大抗突发，但内存更高 |
| `BatchTimeout` | 5s | 批次最长等待时间 | 调小延迟低，但请求数多 |
| `MaxExportBatchSize` | 512 | 每批最大 span 数 | 调大吞吐高，但单次超时风险高 |
| `ExportTimeout` | 30s | 单次导出超时 | 太长会堵队列，太短会频繁失败 |

对应的环境变量：`OTEL_BSP_MAX_QUEUE_SIZE`、`OTEL_BSP_SCHEDULE_DELAY`、`OTEL_BSP_MAX_EXPORT_BATCH_SIZE`、`OTEL_BSP_EXPORT_TIMEOUT`。

## 3.3 队列溢出会怎样

```text
业务产生 span → 入队 → 队列满 → 直接丢弃（不阻塞业务）→ exporter 看不到
```

★ **关键认知**：BatchSpanProcessor 的设计目标是**不拖慢业务**，所以队列满时它选择丢数据，而不是阻塞。丢数据默认不会让业务报错，也不一定会打日志——必须主动监控队列和导出错误，否则你会以为"链路都正常"，其实只是没采到。

```go
// 导出失败、队列异常等情况会走这里；不设置就可能完全静默
otel.SetErrorHandler(otel.ErrorHandlerFunc(func(err error) {
	slog.Error("otel sdk error", "err", err)
}))
```

# 4. 采样器

## 4.1 内置采样器

| 采样器 | 行为 | 适用 |
|--------|------|------|
| `AlwaysSample` | 全采 | 开发、低流量、关键链路 |
| `NeverSample` | 全不采 | 临时关闭某个服务 |
| `TraceIDRatioBased(r)` | 按 trace_id 哈希采 r 比例 | 均匀降量 |
| `ParentBased(root)` | 有父按父，无父按 root 策略 | 生产最常用 |

```go
sdktrace.WithSampler(sdktrace.ParentBased(
	sdktrace.TraceIDRatioBased(0.1), // 新链路采 10%
))
```

`TraceIDRatioBased` 是确定性的：同一个 trace_id 在任何服务、任何实例上算出的结果一致。这意味着即使不用 `ParentBased`，只要所有服务的比例相同，链路也能保持一致。

## 4.2 为什么子 Span 必须遵循父决策

```text
网关采 10% → traceflags=01 → 订单服务继承"采"
库存服务若独立配 0.5 → 可能算出"不采" → 链路断成半截
```

★ **关键认知**：生产环境用 `ParentBased`，让下游**无条件继承上游的采样标志**。这样网关决定采 10%，整条链路就是完整 10%；否则下游改比例、改采样器，都会制造"部分 span 缺失"的诡异链路。

## 4.3 头部采样与尾部采样

| 方式 | 决策时机 | 优点 | 缺点 |
|------|---------|------|------|
| 头部采样 | Span 创建时 | 开销最低、实现简单 | 无法根据结果决定，可能丢错误链路 |
| 尾部采样 | 整条 trace 结束后 | 可按错误/延迟保留 | 需要 Collector 缓存整条链路 |

SDK 里的采样器都是头部采样。想实现"错误全留、慢请求全留"，要在 Collector 的 `tail_sampling` 里做，第 7 篇会展开。

# 5. 优雅关闭

## 5.1 关闭顺序

```text
收到 SIGTERM
   │
   ▼
1. 停止接收新请求（HTTP Server Shutdown / gRPC GracefulStop）
   │
   ▼
2. 等待在途请求结束（带超时）
   │
   ▼
3. Shutdown TracerProvider / MeterProvider / LoggerProvider
   │  flush 队列中的 span/metric/log
   ▼
4. 进程退出
```

⚠ **关键认知**：顺序不能反。先关 Provider，正在处理的请求还会产生 span，这些 span 会进入已经关闭的处理器被丢弃；先停服务再关 Provider，才能保证最后一批数据完整导出。

## 5.2 信号处理

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

// 先让 HTTP 服务停止接收新请求，并等待在途请求
if err := srv.Shutdown(shutdownCtx); err != nil {
	slog.Error("http shutdown", "err", err)
}

// 再关闭 Provider，flush 最后一批遥测数据
if err := tp.Shutdown(shutdownCtx); err != nil {
	slog.Error("otel shutdown", "err", err)
}
```

## 5.3 Shutdown 的超时

```go
// 错误示范：没有超时，后端不可达时可能一直卡住
defer tp.Shutdown(context.Background())

// 正确做法：给一个明确的 flush 窗口
defer func() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	_ = tp.Shutdown(ctx)
}()
```

`Shutdown` 会停止接收新 span，并尝试把队列里的数据导出。后端不可达时，它会等到超时；没有超时保护，进程可能迟迟退不掉，Kubernetes 最后只能 SIGKILL。

⚠ **关键认知**：`log.Fatal` / `os.Exit` 会**跳过 defer**，写在 `main` 里的 `defer tp.Shutdown(...)` 根本不会执行。要么在退出前显式调用，要么用 `run() error` 模式把退出逻辑收敛到一处。

# 6. 完整初始化模板

```go
type Config struct {
	ServiceName   string
	Version       string
	Env           string
	CollectorAddr string
	SampleRatio   float64
}

func setupOTel(ctx context.Context, cfg Config) (func(context.Context) error, error) {
	// 1. Resource：服务身份 + 默认运行环境信息
	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceName(cfg.ServiceName),
			semconv.ServiceVersion(cfg.Version),
			attribute.String("deployment.environment.name", cfg.Env),
		),
	)
	if err != nil {
		return nil, err
	}

	// 2. Exporter：生产用 OTLP gRPC；内网无 TLS 才用 WithInsecure
	exp, err := otlptracegrpc.New(ctx,
		otlptracegrpc.WithEndpoint(cfg.CollectorAddr),
		otlptracegrpc.WithInsecure(),
	)
	if err != nil {
		return nil, err
	}

	// 3. BatchSpanProcessor：按流量调整队列和批量
	bsp := sdktrace.NewBatchSpanProcessor(exp,
		sdktrace.WithBatchTimeout(5*time.Second),
		sdktrace.WithMaxQueueSize(8192),
		sdktrace.WithMaxExportBatchSize(1024),
		sdktrace.WithExportTimeout(10*time.Second),
	)

	// 4. TracerProvider：Resource + 采样器 + 处理器
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.ParentBased(
			sdktrace.TraceIDRatioBased(cfg.SampleRatio),
		)),
		sdktrace.WithSpanProcessor(bsp),
	)

	// 5. 注册全局：自动埋点库依赖这两个设置
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{},
		propagation.Baggage{},
	))
	otel.SetErrorHandler(otel.ErrorHandlerFunc(func(err error) {
		slog.Error("otel sdk error", "err", err)
	}))

	// 6. 返回关闭函数：调用方负责在停服务之后执行
	return tp.Shutdown, nil
}
```

Metrics 和 Logs 的初始化结构完全一样，只是把 `TracerProvider` 换成 `MeterProvider` / `LoggerProvider`，并各自持有自己的 Reader/Processor。关闭时按 Logs → Metrics → Traces 或相反顺序都可以，关键是**每个 Provider 都要 Shutdown**。

# 7. 常见配置错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 后端全是 `unknown_service` | 没设 `service.name` | Resource 显式设置或用 `OTEL_SERVICE_NAME` |
| 生产链路只有 10% 完整 | 下游采样器和上游不一致 | 统一 `ParentBased` |
| 偶发大量链路缺失 | BSP 队列太小，突发被丢 | 调大 `MaxQueueSize`，监控导出错误 |
| 进程退出慢或卡死 | Shutdown 无超时，后端不可达 | 加 `context.WithTimeout` |
| 最后一批数据丢失 | 先关 Provider 或 `log.Fatal` 跳过 defer | 调整顺序，避免 `os.Exit` |
| 导出全部失败但业务正常 | 没设 `otel.SetErrorHandler` | 注册错误处理器并打日志 |
| gRPC 连不上 Collector | endpoint 写成 `http://host:4317` | gRPC 用 `host:4317`，HTTP 才带 scheme |
| 本地能导出、线上不能 | 代码写死 localhost | 用 `OTEL_EXPORTER_OTLP_ENDPOINT` 注入 |
| 内存持续上涨 | 每个请求新建 Provider/Exporter | 启动时创建一次，全局复用 |

# 8. 练习与自检

1. 用环境变量配置一个服务：`OTEL_SERVICE_NAME=payment-service`、`OTEL_RESOURCE_ATTRIBUTES=deployment.environment.name=staging`、`OTEL_TRACES_SAMPLER=parentbased_traceidratio`、`OTEL_TRACES_SAMPLER_ARG=0.2`，代码里不写死这些值，验证 Resource 和采样是否生效。
2. 把 BSP 的 `MaxQueueSize` 调到 1、`BatchTimeout` 调到 1 分钟，制造一批 span，观察丢数据和延迟导出，理解两个参数的作用。
3. 写一个带 `signal.NotifyContext` 的 `main`，模拟收到 SIGTERM：先 `srv.Shutdown`，再 `tp.Shutdown`，用 Collector 的 debug exporter 确认最后一批 span 是否到达。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 初始化 | 配置 → Resource → Exporter → Processor → Provider → 全局注册 |
| Resource | 服务名/版本/环境必须显式设置，否则 unknown_service |
| 环境变量 | 部署相关信息由环境注入，代码只留兜底 |
| BSP | 生产用 Batch，队列满会静默丢弃，必须监控 |
| 采样 | 生产用 `ParentBased`，下游继承上游决策 |
| 关闭 | 先停服务，再 Shutdown Provider，且必须带超时 |
| 错误处理 | `otel.SetErrorHandler` 让导出失败可见 |

下一篇进入 Context 与传播：W3C TraceContext 的 `traceparent` 到底长什么样、Baggage 怎么用、HTTP 和 gRPC 如何自动透传，以及跨服务链路断开时应该按什么顺序排查。
