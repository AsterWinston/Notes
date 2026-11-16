埋点做完只是开始：数据最终要落到能查、能看图、能告警的后端上。Jaeger 与 Tempo 存链路，Prometheus 存指标，Loki 存日志，Grafana 把它们拼成一次排障动作。本篇讲清四类后端的接法、容量怎么估、成本怎么压，最后用一组自检题收束整个模块。

# 1. 后端与信号的对应关系

| 信号 | 常用后端 | 查询语言 | 核心取舍 |
|------|---------|---------|---------|
| Traces | Jaeger、Tempo | Jaeger UI、TraceQL | Jaeger 功能全，Tempo 成本低 |
| Metrics | Prometheus、Mimir、VictoriaMetrics | PromQL | 本地盘简单，长期存储要远端 |
| Logs | Loki、Elasticsearch | LogQL、KQL | Loki 省成本，全文检索弱 |
| 展示 | Grafana | 统一面板 | 跨信号跳转的关键 |

```text
┌─────────┐   ┌─────────┐   ┌─────────┐
│  app A  │   │  app B  │   │  app C  │
└────┬────┘   └────┬────┘   └────┬────┘
     └─────────────┼─────────────┘
                   ▼
          ┌─────────────────┐
          │    Collector    │  采样、脱敏、路由
          └───┬────┬────┬───┘
              ▼    ▼    ▼
          Tempo  Prometheus  Loki
              └────┬────┘
                   ▼
             ┌──────────┐
             │ Grafana  │  指标 → 链路 → 日志
             └──────────┘
```

★ **关键认知**：后端按信号分工，但关联键必须统一。Grafana 能从指标 exemplar 跳到 trace，再从 trace 跳到同 trace_id 的日志，靠的是三端都保留 `trace_id`。任何一端丢了它，跨信号排障就退化成手工复制时间戳。

# 2. Jaeger 对接

## 2.1 用 OTLP 接收

Jaeger 1.35 起原生支持 OTLP，Jaeger v2 直接基于 Collector 构建，不需要再用已废弃的 Jaeger thrift exporter：

```yaml
exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
    sending_queue:
      enabled: true
      queue_size: 5000
    retry_on_failure:
      enabled: true
      max_elapsed_time: 300s
```

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/jaeger]
```

应用侧仍然只配 Collector 地址，感知不到后端是 Jaeger 还是 Tempo，这正是 OTLP 的价值。

## 2.2 存储与保留

生产必须用持久化后端：Badger 适合单机小规模，Cassandra 适合大规模写入，Elasticsearch 适合需要全文检索但成本最高；`memory` 只用于本地演示。

⚠ **关键认知**：Jaeger 的默认 all-in-one 镜像使用内存存储，只适合看一条链路长什么样。生产必须显式配置持久化存储与保留时间，否则一次重启就会让所有历史链路消失。

# 3. Tempo 对接

## 3.1 为什么选 Tempo

Tempo 把 trace 块直接写进对象存储（S3、GCS、OSS、MinIO），索引很轻，成本随对象存储价格走。代价是查询依赖 TraceQL，且对标签设计更敏感。

| 维度 | Jaeger | Tempo |
|------|--------|-------|
| 存储 | Cassandra/ES/Badger | 对象存储 |
| 成本 | 中到高 | 低 |
| 查询 | Jaeger UI 搜索 | TraceQL |
| 适合 | 需要成熟 UI 与检索 | 大规模、成本敏感 |

## 3.2 Collector 配置

```yaml
exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
    sending_queue:
      enabled: true
      queue_size: 10000
    retry_on_failure:
      enabled: true
      max_elapsed_time: 300s
```

★ **关键认知**：Tempo 的查询性能很大程度取决于属性是否适合筛选。`service.name`、`deployment.environment.name`、`http.route` 这类低基数属性要保留；用户 ID、订单号放进属性不会拖慢写入，但会让 TraceQL 被迫扫描大量块。

# 4. Prometheus 对接

## 4.1 推与拉的桥接

OTel 默认推（OTLP），Prometheus 默认拉（scrape），两种接法：

| 方式 | 数据方向 | Collector 组件 | 适用 |
|------|---------|---------------|------|
| 暴露 `/metrics` | Prometheus 来抓 | `prometheus` exporter | 单集群、传统 Prometheus |
| 远程写 | Collector 推过去 | `prometheusremotewrite` exporter | 多集群汇聚、托管 Prometheus |

## 4.2 暴露端点供抓取

```yaml
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
    resource_to_telemetry_conversion:
      enabled: true   # 把 service.name 等资源属性变成指标标签

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

```yaml
scrape_configs:
  - job_name: otel-gateway
    static_configs:
      - targets: ["otel-gateway:8889"]
```

远程写则把 exporter 换成 `prometheusremotewrite`，`endpoint` 指向 `/api/v1/write`，适合把多个集群的 Collector 汇聚到长期存储，但要注意鉴权、乱序样本和重复样本。

## 4.3 命名与时间性

| 现象 | 原因 | 说明 |
|------|------|------|
| 多了 `_seconds` | OTel 带单位，Prometheus 加后缀 | 以 `/metrics` 实际输出为准 |
| Counter 多了 `_total` | Prometheus 约定 | `http.server.requests` 变成 `http_server_requests_total` |
| 点号变下划线 | Prometheus 命名限制 | `http.server.duration` → `http_server_duration` |
| 曲线锯齿或归零 | 时间性不匹配 | OTLP delta 与 cumulative 要对齐 |

⚠ **关键认知**：不要拿 OTel 里的指标名直接写 PromQL。名字、后缀、标签都可能变，先打开 `/metrics` 看真实输出，再写查询和看板。delta 时间性可用 `deltatocumulative` processor（contrib）转换，或让 SDK 直接输出 cumulative。

# 5. Loki 对接

## 5.1 OTLP 日志写入

```yaml
exporters:
  otlphttp/loki:
    endpoint: http://loki:3100/otlp
    tls:
      insecure: true

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/loki]
```

应用侧按第 4 篇的方式把日志桥接到 OTLP，Collector 只做转发与脱敏，不再额外解析文本。

## 5.2 标签与高基数

| 字段 | 是否做标签 | 原因 |
|------|-----------|------|
| `service.name` | 是 | 低基数，最常用筛选 |
| `deployment.environment.name` | 是 | 低基数 |
| `level` | 可以 | 通常只有 5 到 6 个值 |
| `trace_id`、`user_id`、`order_id` | 否 | 高基数，时间线会爆炸 |
| `http.route` | 谨慎 | 路由数量可控时可做 |

★ **关键认知**：`trace_id` 不是标签，而是结构化元数据或正文字段。把它做成 Loki 标签，等于给每条请求建一条索引，几天内就能把索引打爆。正确做法是用 Grafana 的 derived field 从日志正文提取 trace_id，再跳转到 Tempo。

# 6. 性能与容量

## 6.1 先算数据量

```text
span 速率 = 请求 QPS × 每请求 span 数
每日字节 = span 速率 × 平均 span 大小 × 86400
```

以 2000 QPS、每请求 8 个 span、平均 500 B 计算：span 速率 16000/s，原始日增约 691 GB，10% 采样后约 69 GB。日志按"每请求条数 × 单条大小"估，指标按活跃时间线数量估。

⚠ **关键认知**：容量规划最常见的错误是只算请求量、不算每请求 span 数。一次 N+1 查询就能把 8 个 span 变成 80 个，日增直接翻十倍。先优化埋点结构，再谈扩存储。

## 6.2 采样与容量

| 策略 | 保留什么 | 成本 | 风险 |
|------|---------|------|------|
| 头部 10% | 均匀样本 | 低 | 错误链路可能被丢 |
| 尾部：错误 + 慢 + 5% 基线 | 异常 + 均匀样本 | 中 | 需要缓存整条 trace |
| 全量 | 所有链路 | 高 | 存储与查询都吃紧 |

生产常用入口 10% 头部采样加 gateway 尾部补采错误与慢请求。注意头部已丢弃的 trace，尾部采样救不回来；需要错误全留时，入口采样率不能压得太低。

| 资源 | 估算方式 | 注意事项 |
|------|---------|---------|
| CPU | 按 span/s 压测，单核大致 1 到 3 万 span/s | processor 越多越慢 |
| 内存 | 队列长度 × 单条大小 × 2 到 3 倍安全系数 | 必须配 `memory_limiter` |
| 网络 | 入口与出口流量之和 | 扇出会成倍增加 |
| 磁盘 | 仅持久化队列需要 | 预留队列上限的 2 倍 |

★ **关键认知**：Collector 的扩容信号不是 CPU，而是队列长度与拒绝计数。下游变慢时 CPU 可能很低，但队列已经堆满、数据正在被丢弃，基于 CPU 的 HPA 在此时不会扩容。

## 6.3 成本控制

| 动作 | 收益 | 代价 |
|------|------|------|
| 降低头部采样率 | 线性降存储 | 长尾问题更难复现 |
| 尾部只留错误与慢请求 | 保留关键链路 | gateway 内存与复杂度上升 |
| 删除无用属性 | 减小单条体积 | 可能丢失检索维度 |

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| Collector 有数据但后端没有 | pipeline 缺 exporter 或地址错 | 检查 `service.pipelines` 与 endpoint |
| Prometheus 查不到指标 | 命名、后缀或时间性不符 | 以 `/metrics` 实际输出为准 |
| 链路能查但日志关联不上 | 日志没带 trace_id | 用日志桥接或补字段 |
| Loki 写入被拒 | 标签基数过高 | 移除 trace_id、user_id 等标签 |
| 采样后错误链路缺失 | 入口头部采样已丢弃 | 提高入口采样率或前置尾部采样 |

# 8. 高频自检题

## 8.1 概念与选型

**1. OTel、Collector、Jaeger 三者的边界是什么？**

OTel 是采集标准与 SDK，负责产生和传输；Collector 是加工与转发管道；Jaeger 是存储与查询后端。Collector 不是数据库，重启可能丢未导出的数据。

**2. 为什么生产环境通常不让 SDK 直连后端？**

配置分散、连接数放大、缺少统一脱敏与采样、后端故障会直接影响应用。Collector 把这些问题集中到一层解决。

**3. 头部采样与尾部采样的本质区别？**

头部在 span 创建时决定，成本低但看不到结果；尾部等整条 trace 结束后按错误、延迟等条件决定，需要缓存整条链路，且同一 trace 必须落在同一实例。

**4. agent 与 gateway 怎么选？**

agent 贴近应用，适合采主机指标和容器日志；gateway 集中治理，适合尾部采样、脱敏和统一出口。默认组合是 agent 收、gateway 治。

## 8.2 排障与容量

**5. 应用说发了数据，后端却没有，按什么顺序查？**

应用 SDK 错误 → Collector receiver 接收指标 → debug exporter 是否可见 → exporter 失败与队列指标 → 后端写入与保留策略。逐跳确认，不要直接搜后端。

**6. `memory_limiter` 为什么必须放第一位？**

它按进程内存拒绝新数据。放在其他 processor 后面，数据已经占用过批缓存或采样缓存，限流失去保护意义。

**7. Prometheus 里查不到 OTel 的指标名，常见原因？**

单位后缀、`_total`、点号转下划线、资源属性是否转成标签、时间性是否匹配。以 `/metrics` 实际输出为准。

**8. 为什么不能把 trace_id 做成 Loki 标签？**

Loki 为每个标签组合建索引，trace_id 每条请求都不同，会导致索引与时间线爆炸。应放正文或结构化元数据，用 derived field 跳转。

## 8.3 动手练习

1. 用 Docker Compose 起 Collector + Tempo + Grafana，把第 5 篇的示例指向 Collector，确认能在 Tempo 里按 `service.name` 查到链路。
2. 给同一个 Collector 增加 `prometheus` exporter，配置 Prometheus 抓取 `/metrics`，观察 `http_server_request_duration_seconds` 的名字与标签变化。
3. 把 Loki 的 `trace_id` 从标签改成正文或结构化元数据，用 Grafana derived field 建立从 trace 到日志的跳转，记录改造前后的索引大小。

# 9. 模块总结

| 篇目 | 核心结论 |
|------|---------|
| 1 概览与信号 | 指标看趋势、链路看路径、日志看细节，trace_id 是主键 |
| 2 Trace 与 Span | `Start` 返回的新 ctx 必须继续传，`End` 必须配对 |
| 3 Metrics | 同步与异步仪器分工，标签基数决定成本 |
| 4 Logs | 日志要结构化并带 trace_id，桥接优于手工拼字段 |
| 5 SDK 与 Resource | 服务身份显式设置，Batch 队列会静默丢弃，关闭要 flush |
| 6 上下文传播 | trace_id 全程不变，span_id 每跳变化，Inject/Extract 同配置 |
| 7 Collector | 限流最前、批处理最后，尾部采样放 gateway |
| 8 框架埋点 | 自动埋点搭骨架，手动 span 补语义，SQL 三处脱敏 |
| 9 后端与容量 | 按信号选后端，按 span 速率估容量，按队列扩容 |

# 10. 小结

| 主题 | 要点 |
|------|------|
| 后端分工 | Jaeger/Tempo 存链路，Prometheus 存指标，Loki 存日志 |
| Jaeger | 用 OTLP 接收，生产必须换持久化存储 |
| Tempo | 对象存储降成本，属性设计决定查询体验 |
| Prometheus | 拉用 `prometheus` exporter，推用 remote write |
| Loki | 只给低基数维度建标签，trace_id 放正文 |
| 容量 | 先算每请求 span 数，再算日增与采样后体积 |

本模块到此收束：从三支柱的概念、SDK 埋点、上下文传播，到 Collector 治理与后端落地，已经覆盖一条遥测数据从产生到可查询的完整路径。后续若继续深入，下一个方向是 Profiles（持续剖析）与 eBPF 无侵入采集，它们解决的是"代码没埋点但想知道 CPU 花在哪"的问题。
