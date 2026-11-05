可观测性不是"多打日志"，而是当故障没按预期出现时，你仍然能靠系统留下的数据把问题定位出来。OpenTelemetry（简称 OTel）把这套采集能力做成了厂商中立的标准。本篇先建立全局地图：三支柱各自回答什么问题、OTel 生态由哪些部分组成、它和 Prometheus/Jaeger 的边界在哪里。

# 1. 可观测性三支柱

## 1.1 三种信号回答不同问题

初学者常把可观测性等同于"日志多打一点"。实际上，三类信号的分工非常明确：

| 信号 | 回答的问题 | 典型数据 | 成本量级 |
|------|-----------|---------|---------|
| Metrics（指标） | 系统整体现在健康吗？趋势如何？ | QPS、延迟分位数、错误率、队列长度 | 最低，聚合后存储 |
| Traces（链路） | 这一次请求慢在哪一跳？ | trace_id、span、耗时、调用关系 | 中，按请求量增长 |
| Logs（日志） | 这一跳具体发生了什么？ | 时间、级别、消息、字段 | 最高，按条数增长 |

★ **关键认知**：三者不是互相替代，而是一条**从粗到细的排查路径**——指标先发现"有问题"，链路定位"哪一跳有问题"，日志确认"这一跳为什么有问题"。只做其中一层，排障时总会在某个环节卡住。

## 1.2 一个请求在三种信号里的投影

```text
用户请求 GET /orders/A-1001
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ Metrics                                              │
│  http.server.requests{method=GET,status=200} += 1    │
│  http.server.duration 记录 86ms                       │
│  db.client.connections{state=used} = 7               │
└─────────────────────────────────────────────────────┘
        │ 同一个请求
        ▼
┌─────────────────────────────────────────────────────┐
│ Trace（trace_id=4bf92f35...）                        │
│  [gateway 86ms]                                      │
│    ├── [order-service 71ms]                          │
│    │     ├── [redis 3ms]                             │
│    │     └── [postgres 52ms]  ← 慢在这里              │
│    └── [user-service 9ms]                            │
└─────────────────────────────────────────────────────┘
        │ 同一个 span
        ▼
┌─────────────────────────────────────────────────────┐
│ Logs                                                 │
│  {"level":"warn","trace_id":"4bf92f35...",           │
│   "span_id":"00f067aa...","msg":"slow query",        │
│   "sql":"SELECT ... WHERE id=$1","cost_ms":52}       │
└─────────────────────────────────────────────────────┘
```

## 1.3 关联键：trace_id 与 span_id

三种信号能串起来，靠的是共享标识：

- `trace_id`：一次分布式请求的全局 ID，32 位十六进制。
- `span_id`：链路中一个操作节点的 ID，16 位十六进制。
- Exemplar：指标样本上附带的一个 trace_id，用于从延迟直方图直接跳到某条慢请求。

★ **关键认知**：**trace_id 是可观测性的主键**。日志里没有 trace_id，就只能按时间去猜；指标上没有 exemplar，就只能知道"慢了"却找不到"哪一次慢"。

# 2. OpenTelemetry 生态

## 2.1 定位：标准，不是后端

OpenTelemetry 只负责**产生、采集、加工、传输**遥测数据，不负责长期存储和可视化。它由 CNCF 托管，目标是让埋点代码与后端解耦：

```text
业务代码 ──► OTel API/SDK ──► OTLP ──► Collector ──► 任意后端
                                                    ├─ Jaeger / Tempo
                                                    ├─ Prometheus
                                                    ├─ Loki
                                                    └─ 商业 APM
```

★ **关键认知**：换后端时只改 Collector 的 exporter 配置，不改业务代码——这是 OTel 相对厂商私有 Agent 最大的价值。反过来，**OTel 本身不能替你看图排障**，它必须落在一个后端上。

## 2.2 生态组成

| 组件 | 作用 | 典型包/项目 |
|------|------|------------|
| API | 稳定的埋点接口，库作者只依赖它 | `go.opentelemetry.io/otel` |
| SDK | API 的具体实现，负责采样、批处理、导出 | `go.opentelemetry.io/otel/sdk/*` |
| Contrib | 官方维护的框架自动埋点与桥接 | `otelhttp`、`otelgin`、`otelgrpc` |
| OTLP | 统一传输协议，支持 gRPC 与 HTTP | 4317（gRPC）、4318（HTTP） |
| Collector | 独立的接收、处理、导出进程 | `otelcol`、`otelcol-contrib` |
| Semantic Conventions | 属性命名的统一约定 | `service.name`、`http.request.method` |
| Operator | Kubernetes 上管理 Collector 的控制器 | `opentelemetry-operator` |

## 2.3 数据流与部署形态

```text
┌──────────────┐   OTLP    ┌──────────────────┐   OTLP    ┌──────────────┐
│ 应用进程      │ ────────► │ Collector Agent  │ ────────► │ 后端存储      │
│ API + SDK    │           │ 每节点/每 Pod     │           │ Jaeger/Tempo │
└──────────────┘           └──────────────────┘           └──────────────┘
                                   │
                                   ▼
                           ┌──────────────────┐
                           │ Collector Gateway│  统一采样、脱敏、路由
                           └──────────────────┘
```

小规模可以直接 SDK 导出到后端；生产环境通常加一层 Collector，用于批处理、重试、脱敏和统一出口。

## 2.4 版本与成熟度

| 信号 | Go SDK 状态 | 说明 |
|------|------------|------|
| Traces | 稳定 | API 与 SDK 都已 1.x |
| Metrics | 稳定 | 同步/异步仪器齐全 |
| Logs | 1.34 起稳定 | 早期 1.x 版本需按实验性 API 对待 |
| Profiles | Alpha | 持续剖析，暂不作为主链路 |

本篇以 OpenTelemetry Go SDK 1.x 与 Collector 为准。升级前先看 `go.mod` 里的实际版本，`semconv` 和日志桥接包在不同小版本间有过 API 调整。

# 3. 三种信号如何协作

## 3.1 排障顺序

```text
1. 看指标：错误率突增 / P99 延迟抬升 / 队列积压
        │
        ▼
2. 看链路：筛出错误或慢的 trace，找到最慢 span
        │
        ▼
3. 看日志：用 trace_id 过滤，确认异常堆栈与业务字段
        │
        ▼
4. 回到指标：修复后观察同一条曲线是否恢复
```

⚠ **关键认知**：不要一上来就 grep 日志。日志量最大、噪声最多，先靠指标缩小时间窗、靠链路缩小服务范围，日志只查那几条相关记录，效率差一个数量级。

## 3.2 Exemplar 把指标和链路钉在一起

直方图只告诉你"P99 是 800ms"，exemplar 则在这个桶里附带一个具体 trace_id：

```text
http_server_duration_bucket{le="1.0"} 9821 # {trace_id="4bf92f35..."} 0.86
```

在 Grafana 里点 exemplar 就能跳到那条慢请求的链路。它依赖 SDK 记录 exemplar、后端支持存储，两端都开启才有用。

# 4. 与 Prometheus / Jaeger / Loki 的边界

## 4.1 各系统本质

| 系统 | 本质 | 负责 | 不负责 |
|------|------|------|--------|
| OpenTelemetry | 采集标准 + SDK + 传输 | 埋点、加工、导出 | 存储、查询、告警 |
| Prometheus | 指标时序数据库 | 抓取/存储指标、PromQL、告警 | 分布式链路、日志正文 |
| Jaeger | 链路后端与 UI | 存储/查询 trace、服务拓扑 | 指标长期聚合、日志全文 |
| Tempo | 链路后端 | 低成本对象存储 trace、TraceQL | 指标聚合 |
| Loki | 日志后端 | 按标签索引日志、LogQL | 高基数标签、链路拓扑 |
| Grafana | 可视化 | 统一查询面板、跨信号跳转 | 数据采集与存储 |

## 4.2 容易混淆的三件事

1. **OTel 不是 Prometheus 的替代品**。Prometheus 是拉模型（pull），OTel 默认是推模型（push OTLP）。Collector 可以暴露 `/metrics` 让 Prometheus 来抓，也可以把数据 remote write 过去。
2. **OTel 不是 Jaeger 的替代品**。Jaeger 1.35+ 原生接收 OTLP，Jaeger v2 更是直接基于 Collector 架构；OTel 负责把 span 送过去，Jaeger 负责存和查。
3. **Collector 不是数据库**。它内存里只有队列和批缓存，进程重启未导出的数据可能丢失，不能当存储层用。

★ **关键认知**：把这几个系统想成"生产端标准 + 传输管道 + 专用存储"三层，选型就不会乱：**采集统一用 OTel，存储按信号选专用后端，展示统一到 Grafana**。

# 5. 最小可运行示例

先只接 Trace，把一条 span 打到标准输出，确认整条链路通了再加后端：

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func main() {
	ctx := context.Background()

	// 1. 建 exporter：开发期先用 stdout，确认有数据再换 OTLP
	exp, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		log.Fatal(err)
	}

	// 2. 合并默认资源与自定义服务名，后端才不至于显示 unknown_service
	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceName("hello-otel"),
		),
	)
	if err != nil {
		log.Fatal(err)
	}

	// 3. 建 provider：批处理 + 资源 + 采样策略
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exp),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(1.0))),
	)
	defer func() {
		// 4. 退出前 flush，否则最后一批 span 可能来不及导出
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		_ = tp.Shutdown(shutdownCtx)
	}()
	otel.SetTracerProvider(tp)

	// 5. 业务埋点：Start 必须配对 End
	tracer := otel.Tracer("hello-otel")
	_, span := tracer.Start(ctx, "handle-request")
	span.SetAttributes(attribute.String("order.id", "A-1001"))
	span.AddEvent("库存校验完成")
	span.End()

	fmt.Println("done")
}
```

```bash
go get go.opentelemetry.io/otel@latest
go get go.opentelemetry.io/otel/sdk@latest
go get go.opentelemetry.io/otel/exporters/stdout/stdouttrace@latest
go run .
```

输出大致是：

```text
{
	"Name": "handle-request",
	"SpanContext": {"TraceID": "9f2c...", "SpanID": "1a3b...", "TraceFlags": "01"},
	"Parent": {"TraceID": "0000...", "SpanID": "0000..."},
	"SpanKind": 1,
	"StartTime": "2026-11-06T10:00:00.000Z",
	"EndTime": "2026-11-06T10:00:00.001Z",
	"Attributes": [{"Key": "order.id", "Value": {"Type": "STRING", "Value": "A-1001"}}],
	"Events": [{"Name": "库存校验完成", "Time": "..."}]
}
```

★ **关键认知**：先用 stdout 验证"有没有数据、服务名对不对、属性有没有丢"，再接 OTLP。很多人一上来就配后端，结果连 span 有没有产生都不知道，排查方向会完全跑偏。

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 后端显示 `unknown_service` | 没设置 `service.name` | Resource 里加 `semconv.ServiceName` |
| 一条 span 都没有 | 没注册全局 provider，或进程提前退出 | `otel.SetTracerProvider` + `Shutdown` flush |
| 只有入口 span，下游没有 | 没有透传 context 或没装客户端埋点 | 见第 6 篇传播章节 |
| 指标名和预期不一致 | Prometheus 导出器会做 `_total`、单位后缀转换 | 以 `/metrics` 实际输出为准 |
| 日志里没有 trace_id | 打日志时没传 ctx，或没用桥接 | 用 `InfoContext` 或日志桥接 |
| 采集了但没人看 | 缺少面板、告警和排障流程 | 指标先建 RED/USE 看板 |
| 成本突然上涨 | 高基数属性或全量日志 | 采样、删标签、降日志级别 |

# 7. 练习与自检

1. 把上面的示例改成 OTLP gRPC 导出，指向本地 Collector 的 `localhost:4317`，用 `otelcol --config` 起一个只打印到 debug exporter 的实例，确认 Collector 能收到。
2. 给示例加一个子 span，并故意不调用 `span.End()`，观察 stdout 是否还能看到它，理解"未结束的 span 不会导出"。
3. 用一句话分别说明：指标、链路、日志各自最适合回答什么问题；再说明 trace_id 在三者之间起什么作用。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 三支柱 | 指标看趋势、链路看路径、日志看细节 |
| 关联键 | trace_id 是主键，span_id 定位节点，exemplar 连接指标与链路 |
| OTel 定位 | 采集与传输标准，不负责存储和展示 |
| 生态 | API、SDK、Contrib、Collector、OTLP、semconv |
| 边界 | Prometheus 存指标、Jaeger/Tempo 存链路、Loki 存日志 |
| 落地顺序 | 先 stdout 验证，再 OTLP，再接后端 |

下一篇进入 Trace 与 Span，把 `TracerProvider`、`Tracer`、`Span` 三者的关系、父子上下文、属性事件状态，以及 Span 从创建到导出的完整生命周期讲清楚。
