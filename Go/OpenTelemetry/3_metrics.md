Metrics 是可观测性里成本最低、收益最快的一层：它用聚合换掉了明细，让你能用很小的存储回答"系统现在怎么样"。但指标也是最容易被写坏的——Counter 记了会下降的量、直方图单位写错、标签里塞了 user_id，都会让后端数据变成噪声。本篇从仪器语义讲到指标设计，把 Go SDK 的 Metrics 用法一次讲透。

# 1. 指标模型

## 1.1 MeterProvider、Meter、Instrument

三层关系和 Trace 类似：

| 对象 | 职责 | 创建频率 |
|------|------|---------|
| MeterProvider | 持有 Resource、Reader、View、聚合配置 | 进程级一次 |
| Meter | 按模块划分的指标工厂 | 包级一次 |
| Instrument | 具体的指标（Counter/Histogram 等） | 初始化时一次 |

初始化时先建 `MeterProvider` 并注册为全局，再用 `otel.Meter("order-service")` 取工厂；`Meter` 的名字会体现在 instrumentation scope 上。

★ **关键认知**：**Instrument 必须复用，不能每个请求创建**。同名 instrument 用不同描述重复创建会返回错误；即使参数一致，也没有必要每次分配。把 instrument 放在包级变量或依赖注入的 `Metrics` 结构体里。

## 1.2 同步与异步

| 类型 | 数据来源 | 调用方式 | 适合 |
|------|---------|---------|------|
| 同步（Sync） | 业务代码主动上报 | `Add` / `Record` | 请求计数、耗时、消息大小 |
| 异步（Async） | SDK 周期回调采集 | 注册 callback | 队列深度、连接数、缓存命中率 |

同步仪器在事件发生时调用；异步仪器由 SDK 每隔一个采集周期调用你的回调，读取当前值。异步回调必须**快速、无阻塞、无外部慢调用**，否则会拖慢整个采集周期。

## 1.3 最小可运行示例

```go
package main

import (
	"context"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/stdout/stdoutmetric"
	"go.opentelemetry.io/otel/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
	semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func main() {
	ctx := context.Background()
	exp, _ := stdoutmetric.New(stdoutmetric.WithPrettyPrint())
	res, _ := resource.Merge(resource.Default(), resource.NewWithAttributes(
		semconv.SchemaURL, semconv.ServiceName("metrics-demo")))

	// 周期读取器：每 2 秒把当前聚合结果导出一次
	reader := sdkmetric.NewPeriodicReader(exp, sdkmetric.WithInterval(2*time.Second))
	mp := sdkmetric.NewMeterProvider(
		sdkmetric.WithReader(reader),
		sdkmetric.WithResource(res),
	)
	defer mp.Shutdown(ctx) // 生产环境要给 Shutdown 加超时
	otel.SetMeterProvider(mp)

	meter := otel.Meter("metrics-demo")
	requests, _ := meter.Int64Counter("http.server.requests", metric.WithUnit("{request}"))
	duration, _ := meter.Float64Histogram("http.server.duration", metric.WithUnit("s"))

	attrs := metric.WithAttributes(
		attribute.String("http.request.method", "GET"),
		attribute.Int("http.response.status_code", 200),
	)
	requests.Add(ctx, 1, attrs)        // 计数 +1
	duration.Record(ctx, 0.086, attrs) // 记录一次 86ms
	time.Sleep(3 * time.Second)        // 等周期导出器打一次快照
}
```

```bash
go get go.opentelemetry.io/otel/sdk/metric@latest
go get go.opentelemetry.io/otel/exporters/stdout/stdoutmetric@latest
go run .
```

★ **关键认知**：指标是**聚合值**，不是每一次调用的流水。Counter 最终导出的是"累计到多少"，Histogram 导出的是"各个桶里各有多少次"，中间的具体值不会保留。想要明细，用 Trace 或日志。

# 2. 四种常用仪器

## 2.1 Counter：只增不减

```go
requests, _ := meter.Int64Counter("http.server.requests")
requests.Add(ctx, 1, metric.WithAttributes(
	attribute.String("http.route", "/orders/:id"),
))
```

语义：单调递增。适合请求数、错误数、发送字节数、任务完成数。

⚠ **关键认知**：Counter **不能减**。用 Counter 记录"当前在线人数"或"队列长度"，一旦数值下降，后端会把它当成计数器重置或数据异常，算出来的速率完全失真。

## 2.2 UpDownCounter：可增可减

```go
inFlight, _ := meter.Int64UpDownCounter("http.server.active_requests")
inFlight.Add(ctx, 1)          // 请求进入
defer inFlight.Add(ctx, -1)   // 请求结束
```

语义：非单调的累计量，适合并发请求数、等待中的任务数、连接池占用。

★ **关键认知**：UpDownCounter 记录的是"变化量"，后端对一段时间求和就能得到区间内的净变化。它适合"进程生命周期内持续累加/累减"的量，但不适合随时可被外部重置的瞬时值——那类用 Gauge。

## 2.3 Histogram：分布与分位数

```go
start := time.Now()
handle(ctx)
duration.Record(ctx, time.Since(start).Seconds(), metric.WithAttributes(
	attribute.String("http.route", "/orders/:id"),
))
```

Histogram 导出的是桶计数（bucket counts）、总和（sum）和样本数（count）。分位数（P50/P95/P99）由后端根据桶边界近似计算。

⚠ **关键认知**：Histogram 的桶边界决定了分位数精度。用默认桶记录一个 5ms 到 2s 的接口，P99 可能全部落进最后一个桶，等于没测。要用 View 按业务延迟分布定制边界（见 3.3）。

## 2.4 Gauge：瞬时值

```go
queueDepth, _ := meter.Int64Gauge("worker.queue.depth")
queueDepth.Record(ctx, int64(len(queue)), metric.WithAttributes(
	attribute.String("queue", "email"),
))
```

Gauge 记录"调用这一刻的值"，后端通常保留最后一次采样，适合内存占用、温度、当前队列深度。

Go SDK 的同步 Gauge 是在 1.x 中后期补齐的，不同小版本可用性不一致；如果当前版本没有 `Int64Gauge`，用异步 `ObservableGauge`（在回调里 `Observe` 当前值）或 UpDownCounter 过渡。

## 2.5 选型对照

| 仪器 | 单调 | 记录方式 | 典型场景 | 后端看到什么 |
|------|------|---------|---------|-------------|
| Counter | 是 | `Add(+n)` | 请求数、错误数、字节数 | 累计总量 / 速率 |
| UpDownCounter | 否 | `Add(±n)` | 并发数、在途任务 | 净变化 / 当前值 |
| Histogram | 否 | `Record(value)` | 延迟、请求体大小 | 桶计数、sum、count |
| Gauge | 否 | `Record(value)` | 队列深度、内存、温度 | 最后一次采样值 |

★ **关键认知**：选仪器的第一原则是问"这个量会不会下降、是否需要分布"。会下降且只关心当前值 → Gauge；会下降但关心累计净变化 → UpDownCounter；只增 → Counter；关心分位数 → Histogram。

# 3. 属性、聚合与 View

## 3.1 属性就是维度

调用 `Add` / `Record` 时传的每组属性，在 Prometheus 里就是一条时间序列。属性的取值集合必须**有界且可控**：

| 属性 | 是否适合做指标标签 | 原因 |
|------|-------------------|------|
| `http.request.method` | 适合 | 取值就那几个 |
| `http.route` | 适合 | 路由模板，数量固定 |
| `http.response.status_code` | 适合 | 几十个 |
| `url.path` | 不适合 | 带 ID，几乎无限 |
| `user.id` | 绝对不适合 | 基数爆炸 |
| `trace_id` | 绝对不适合 | 每条请求都不同 |

## 3.2 聚合与时间性

SDK 在本地先做聚合，再按周期导出。两种时间性（temporality）：

| 时间性 | 含义 | 常见后端 |
|--------|------|---------|
| Cumulative | 从进程启动到现在的累计值 | Prometheus |
| Delta | 只包含本周期内的增量 | 部分 OTLP 后端 |

OTLP 导出器默认多为 cumulative；Prometheus 导出器只支持 cumulative。如果后端要求 delta，通过环境变量或 exporter 选项切换，**不要自己在下游再求差**，容易在重启和补点时算错。

## 3.3 View 定制直方图桶

```go
view := sdkmetric.NewView(
	sdkmetric.Instrument{Name: "http.server.duration"},
	sdkmetric.Stream{
		Aggregation: sdkmetric.AggregationExplicitBucketHistogram{
			// 单位是秒，边界按接口真实延迟分布选择
			Boundaries: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10},
		},
	},
)

mp := sdkmetric.NewMeterProvider(
	sdkmetric.WithReader(reader),
	sdkmetric.WithView(view),
)
```

★ **关键认知**：桶边界要围绕 SLO 设计。如果 SLO 是"P99 < 300ms"，边界必须包含 `0.3` 附近的值（如 `0.25`、`0.3`、`0.5`），否则后端只能告诉你"在 0.25 到 0.5 之间"，无法判断有没有达标。

View 还能改名字、删属性、改聚合方式，例如把 `http.server.requests` 重命名为 `http_requests`，或在 View 里按属性过滤以降低基数。

# 4. 导出

## 4.1 OTLP 周期导出

生产环境最常见的方式：

```go
exp, _ := otlpmetricgrpc.New(ctx,
	otlpmetricgrpc.WithEndpoint("localhost:4317"),
	otlpmetricgrpc.WithInsecure(), // 内网无 TLS 时；公网必须开 TLS
)
reader := sdkmetric.NewPeriodicReader(exp,
	sdkmetric.WithInterval(15*time.Second), // 默认约 60s，观察用可调短
)
mp := sdkmetric.NewMeterProvider(
	sdkmetric.WithReader(reader),
	sdkmetric.WithResource(res),
)
defer mp.Shutdown(ctx)
otel.SetMeterProvider(mp)
```

周期导出的间隔决定了指标的时间分辨率。调得太短会增加网络和后端写入压力，调得太长会让告警延迟变大，通常 10-30 秒是折中值。

## 4.2 Prometheus 拉取

如果后端是 Prometheus，可以用 `otelprom.New()` 创建 Prometheus exporter，把它作为 Reader 传给 `MeterProvider`，再用 `promhttp.Handler()` 暴露 `/metrics`。它默认注册到 `prometheus.DefaultRegisterer`，不需要周期导出器。

★ **关键认知**：Prometheus exporter 是一个 **Reader，不是周期导出器**。每次 Prometheus 来抓 `/metrics` 时才收集，因此不存在"导出间隔"；它只支持 cumulative 时间性。SDK 直连 Prometheus 只适合单实例或开发环境，多实例生产建议走 Collector 汇总。

## 4.3 命名与单位

OTel 的指标命名用点号，Prometheus 导出后会被转换：

```text
OTel 名称                  Prometheus 实际名称（示例）
http.server.requests   →   http_server_requests_total
http.server.duration   →   http_server_duration_seconds
worker.queue.depth     →   worker_queue_depth
```

规则由导出器版本决定，**以 `/metrics` 实际输出为准**，不要凭记忆写告警表达式。命名建议：

- 用语义约定里的标准名，不要自创 `my_service_req_cnt`。
- 单位用 UCUM：秒 `s`、字节 `By`、请求 `{request}`。
- Counter 名称用复数名词（`requests`），不要带 `_total`，导出器会加。
- 名字里不要重复服务名，服务名已经在 Resource 里。

# 5. 指标设计

## 5.1 RED 与 USE

两套经典方法论，覆盖大部分服务：

| 方法 | 关注 | 指标 |
|------|------|------|
| RED | 请求型服务 | Rate（QPS）、Errors（错误率）、Duration（延迟） |
| USE | 资源型组件 | Utilization（使用率）、Saturation（饱和度）、Errors（错误数） |

一个 HTTP 服务的最小指标集：`http.server.requests`（Counter）、`http.server.duration`（Histogram）、`http.server.active_requests`（UpDownCounter），再加数据库连接数和 Goroutine 数两个 ObservableGauge。

## 5.2 基数控制

时间序列数量 ≈ 各标签取值数的乘积。比如 `method(5) × route(80) × status(20) = 8000` 条序列，再乘上 15s 采集、30 天保留，存储和查询都会明显变重。

⚠ **关键认知**：指标后端的崩溃往往不是流量大，而是**标签基数失控**。上线前用"标签取值能不能枚举"做判断：不能枚举的，一律不进指标。

## 5.3 指标不是明细

不要试图用指标回答"用户 A 的第 3 单为什么慢"：

- 指标没有单次请求的上下文。
- 高基数标签会让聚合失去意义。
- 明细应该交给 Trace 和日志。

★ **关键认知**：指标负责**发现异常和衡量趋势**，Trace 负责**定位单次异常**，日志负责**解释异常细节**。用指标存明细，等于用卡车运一粒米。

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 数值只增不减 | 用 Counter 记录可下降的量 | 换 UpDownCounter 或 Gauge |
| P99 全是最大值 | 直方图桶边界不适合业务延迟 | 用 View 定制边界 |
| Prometheus 查不到指标 | 名称被改写、标签名变化 | 看 `/metrics` 原始输出 |
| 指标重复/报错 | 同名 instrument 描述不一致或每请求创建 | 初始化时创建一次并复用 |
| 延迟单位错乱 | `Record` 传了毫秒，单位写 `s` | 统一用秒，或单位与数值严格对应 |
| 采集回调拖慢进程 | 异步回调里查数据库/发 HTTP | 回调只读内存中的原子值 |
| 内存持续上涨 | 标签基数爆炸 | 删除高基数标签，加 View 过滤 |
| 导出报错但业务无感 | 忽略了 instrument 创建错误和导出错误 | 检查 `err`，设置 `otel.SetErrorHandler` |

# 7. 练习与自检

1. 给一个 HTTP 接口补齐 RED 三件套：请求计数、延迟直方图、在途请求数。用 View 把延迟桶边界改成适合 `P99 < 300ms` 的一组值。
2. 用 `Int64ObservableGauge` 上报 Goroutine 数量和数据库连接池使用数，说明为什么这两个值不适合用 Counter。
3. 估算一个指标的时间序列数量：`method` 5 种、`route` 60 种、`status` 12 种、`region` 4 种。如果再加一个 `user_id`，会发生什么？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 模型 | MeterProvider → Meter → Instrument，instrument 复用 |
| 同步/异步 | 事件驱动用同步，瞬时状态用异步 |
| 四种仪器 | Counter 只增、UpDownCounter 可减、Histogram 看分布、Gauge 看瞬时 |
| 属性 | 必须有界，禁止 user_id、trace_id、原始 URL |
| View | 定制桶边界和指标名，按 SLO 设计 |
| 导出 | OTLP 周期推、Prometheus 拉，注意时间性与命名转换 |
| 设计 | RED/USE 起步，先控制基数再谈丰富度 |

下一篇进入 Logs：结构化日志怎么写、OTel Logs 的 LogRecord 模型、如何让日志自动带上 trace_id，以及在高吞吐场景下怎么采样才不至于把日志账单打穿。
