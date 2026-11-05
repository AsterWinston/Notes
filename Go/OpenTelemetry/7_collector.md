SDK 直连后端在 demo 里最省事，生产里却很快撞墙：几十个服务各配一套导出地址、后端一换就要全量发版、采样和脱敏散落在每个进程里。Collector 就是把这些横切职责从应用里抽出来的那一层。本篇讲清 Receiver/Processor/Exporter 的数据流、批处理与重试的坑，以及 agent 与 gateway 的部署取舍。

# 1. 为什么需要 Collector

SDK 直连有四个问题：配置分散在几十个仓库、每个 Pod 都向后端建连接、脱敏与路由无处安放、后端抖动会直接拖慢业务进程。Collector 把这些问题集中到一层解决——统一出口、批量与内存保护、属性加工、可靠传输、按信号路由，并暴露自身的接收与丢弃指标。

★ **关键认知**：SDK 的职责是"产生并尽力送出"数据，不是"可靠交付"。它为了不阻塞业务，队列满时会丢数据；Collector 才是那个可以缓冲、重试、持久化、统一治理的中间层。

# 2. 架构与数据流

## 2.1 组件与 pipeline

```text
              ┌──────────────────── Collector ────────────────────┐
 OTLP ───────►│  Receiver ──► Processor ──► Processor ──► Exporter │───► 后端
              └────────────────────────────────────────────────────┘
```

| 组件 | 职责 | 例子 |
|------|------|------|
| Receiver | 接收数据，转成内部格式 | `otlp`、`prometheus`、`filelog` |
| Processor | 加工、过滤、采样 | `batch`、`memory_limiter`、`attributes`、`tail_sampling` |
| Exporter | 把数据发往后端 | `otlp`、`otlphttp`、`prometheus`、`debug` |
| Extension | 提供辅助能力，不碰数据流 | `health_check`、`pprof`、`file_storage` |
| Service | 声明 pipeline 与启用的扩展 | `service.pipelines` |

★ **关键认知**：`service.pipelines` 只描述"哪类信号、用哪些组件、按什么顺序"，真正的并发与重试由内部调度。同一组件写在多条 pipeline 里，每条 pipeline 都会拥有独立实例，参数各自生效。

## 2.2 最小可运行配置

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128
  batch:
    timeout: 5s
    send_batch_size: 8192

exporters:
  debug:
    verbosity: basic

extensions:
  health_check:
    endpoint: 0.0.0.0:13133

service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
```

一条 pipeline 配多个 exporter 时，同一批数据会各发一份。每个 exporter 有自己的队列与重试，一个失败不会阻断另一个；Processor 只执行一次，加工结果被共享。

⚠ **关键认知**：扇出会成倍增加出口流量和序列化开销。调试用的 `debug` exporter 在压测时会把日志打满，确认链路后应立刻移除。

# 3. Receiver

## 3.1 常见 Receiver

| Receiver | 用途 | 备注 |
|----------|------|------|
| `otlp` | 接收 OTLP gRPC/HTTP | 新系统首选 |
| `prometheus` | 按 Prometheus 协议抓取指标 | 拉模型，采第三方 `/metrics` |
| `jaeger`、`zipkin` | 兼容老协议 | 迁移期使用 |
| `filelog`、`hostmetrics` | 采容器日志、主机指标 | agent 模式常用 |

## 3.2 OTLP 是统一入口

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_recv_msg_size_mib: 16
      http:
        endpoint: 0.0.0.0:4318
```

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-agent:4317
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf   # HTTP 用 4318 并显式指定
```

★ **关键认知**：OTLP 的 gRPC 与 HTTP 端口不同、路径不同、TLS 配置也不同。Go SDK 里 `otlptracegrpc` 的 endpoint 写 `host:4317` 不带 scheme，`otlptracehttp` 才写 `http://host:4318`，混用是最常见的连不上原因。

⚠ **关键认知**：OTLP 默认没有鉴权，不要把 4317/4318 直接暴露到公网，否则任何人都能往里灌数据。生产环境要么只监听内网，要么在前面加 mTLS 或网关鉴权。

# 4. Processor

## 4.1 batch：把逐条变批量

```yaml
processors:
  batch:
    timeout: 5s                 # 最长等 5 秒就发，控制延迟上限
    send_batch_size: 8192       # 攒到 8192 条触发发送
    send_batch_max_size: 10000  # 单批硬上限
```

| 参数 | 调大 | 调小 |
|------|------|------|
| `timeout` | 请求少、吞吐高，但延迟变大 | 延迟低，但请求数多 |
| `send_batch_size` | 单批更大、效率高 | 更实时 |

⚠ **关键认知**：`batch` 必须是 pipeline 里最后一个 processor。放在 `memory_limiter` 前面，被拒绝的数据已经占过批缓存；放在脱敏前面，脱敏会逐条处理本可以批量的数据。顺序错了不会报错，只会悄悄变慢。

## 4.2 memory_limiter：先保进程

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512        # 硬上限：超过后拒绝新数据
    spike_limit_mib: 128  # 预期突发；软上限 512-128=384 MiB 用于触发 GC
```

它达到硬上限后直接拒绝新数据，并通过 `otelcol_receiver_refused_*` 暴露。

★ **关键认知**：Collector 内存主要由接收缓冲、批缓存、发送队列三块构成。只配 `batch` 不配 `memory_limiter`，下游变慢时队列会一直涨，最后被 OOM Kill。它必须放在 pipeline 第一位，容器里还应设 `GOMEMLIMIT` 略低于容器 limit。

## 4.3 属性加工与脱敏

```yaml
processors:
  attributes/redact:
    actions:
      - key: http.request.header.authorization
        action: delete                 # 认证头一律不落库
      - key: user.email
        action: hash                   # 保留关联性，但不可还原
      - key: db.statement
        action: update
        value: "[redacted]"            # 强合规场景直接抹掉
```

`insert` 是键不存在才写，`update` 是存在才覆盖，`upsert` 两者兼顾，`hash` 用 SHA-1 替换原值。

★ **关键认知**：脱敏要发生在数据离开进程之后、进入后端之前。在应用里做，每个团队都要实现一遍且容易漏；在 gateway 上统一做，规则只有一份，审计也只看一处。

## 4.4 tail_sampling：按整条链路决定去留

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: baseline
        type: probabilistic
        probabilistic:
          sampling_percentage: 5
```

策略之间是"或"：错误全留、其余按 5% 留。需要保留慢请求时，再加一个 `latency` 策略并设 `threshold_ms`。

⚠ **关键认知**：尾部采样要求同一条 trace 的所有 span 到达同一个 Collector 实例。多副本又不按 trace_id 负载均衡，链路会被拆开，每个实例只看到半条，判定必然出错。常见做法是只在 gateway 做，并用 `loadbalancing` exporter 按 trace_id 路由。

## 4.5 处理器顺序

```text
memory_limiter → 过滤/脱敏 → tail_sampling → batch
```

限流最先，避免后续处理浪费内存；脱敏在采样前，避免敏感数据进入采样缓存；尾部采样需要按 trace 聚合；批处理最后，把最终结果一次性发出。

# 5. Exporter

## 5.1 常见 Exporter

常用 `otlp` / `otlphttp` 发往 Tempo、Jaeger 或商业 APM；`prometheus` 暴露 `/metrics` 供抓取；`prometheusremotewrite` 做远程写；`otlphttp/loki` 写日志；`debug` 只用于排错。

## 5.2 队列与重试

```yaml
exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
```

| 配置 | 含义 | 建议 |
|------|------|------|
| `num_consumers` | 并发发送协程数 | 按后端承受能力调，不是越大越好 |
| `queue_size` | 内存队列长度 | 配合内存上限，别单独放大 |
| `max_elapsed_time` | 超过后放弃 | 必须设，否则坏数据永久占队列 |

★ **关键认知**：重试是放大流量的机制。后端已经过载时，所有 Collector 同时重试只会让它更糟。必须设置 `max_elapsed_time`，并监控 `otelcol_exporter_send_failed_*`；持续失败应告警，而不是无限重试。

需要跨重启保留时，可给 `sending_queue` 配 `storage: file_storage`，并在 `extensions` 里声明磁盘目录。它会增加磁盘 IO，且目录必须独占，通常只在 gateway 或关键链路开启。`debug` 的 `verbosity: detailed` 会打印完整数据，既占磁盘又可能泄露敏感字段，不要长期开启。

# 6. 部署模式

## 6.1 agent 与 gateway

```text
┌──────── Pod A ────────┐
│ app ──► otel-agent    │──┐      ┌───────────────────┐
└───────────────────────┘  ├─────►│ Collector Gateway │
┌──────── Pod B ────────┐  │      │ 采样、脱敏、路由   │
│ app ──► otel-agent    │──┘      └────────┬──────────┘
└───────────────────────┘           ┌───────┼───────┐
                                    ▼       ▼       ▼
                                 Tempo  Prometheus  Loki
```

- **agent**：与应用同节点或同 Pod，通常 DaemonSet。出口固定为 localhost，网络跳数少，可采主机指标和容器日志；缺点是实例多、配置分散，不适合尾部采样。
- **gateway**：独立 Deployment，配置集中，能做尾部采样和统一脱敏，后端只面对少量连接；缺点是多一跳网络，自身成为需要扩容和监控的关键组件。

## 6.2 组合与取舍

| 维度 | 只有 agent | 只有 gateway | agent + gateway |
|------|-----------|-------------|-----------------|
| 配置管理 | 分散 | 集中 | agent 极简，治理在 gateway |
| 尾部采样 | 不适合 | 适合 | 适合 |
| 主机/容器日志 | 方便 | 不方便 | 方便 |
| 适用规模 | 小集群 | 中等集群 | 多团队、大集群 |

★ **关键认知**：默认推荐 agent 收、gateway 治。agent 只做最轻的批处理和资源补充，保证本机出口稳定；需要全局视角的能力放到 gateway。规模很小时可以只留一层，但不要一开始就上两层。

在 Kubernetes 上，agent 用 DaemonSet 读 `/var/log/pods`，gateway 用 Deployment 且副本至少 2。不要用 CPU 做 gateway HPA 的唯一指标：Collector 是 IO 与队列驱动型，CPU 不高时也可能因下游慢而积压，应结合队列长度和拒绝数扩容。

# 7. 排错

## 7.1 定位顺序

```text
1. 应用侧：SDK 有没有报错？OTLP endpoint 对不对？
2. Collector 入口：receiver_refused / accepted 是否增长？
3. pipeline 中间：debug exporter 能否看到数据？字段是否完整？
4. 出口：exporter_send_failed 是否增长？后端是否可达？
5. 后端：写入限流、索引失败、保留策略是否丢数据？
```

★ **关键认知**：排错要从数据产生端往存储端走，每一步只看一个指标。直接去后端搜"为什么没有数据"，等于跳过了最能说明问题的那几跳。

## 7.2 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 应用报 connection refused | 端口或协议写错 | gRPC 用 4317，HTTP 用 4318 |
| Collector 收到但后端没有 | pipeline 没包含对应 exporter | 检查 `service.pipelines` |
| 启动即退出 | 引用了未定义的组件 | 组件必须在对应 section 定义 |
| 数据被大量拒绝 | 触发 `memory_limiter` | 扩容、下调采样、查下游延迟 |
| 指标名多出后缀 | Prometheus 做单位与 `_total` 转换 | 以 `/metrics` 实际输出为准 |
| 日志重复多份 | 多个 pipeline 收同一来源 | 合并 pipeline 或去掉重复 receiver |
| 尾部采样结果不稳定 | 多副本未按 trace_id 负载均衡 | 用 `loadbalancing` exporter |
| 重启后丢一截数据 | 内存队列且未 flush | 开持久化队列或接受丢失 |

## 7.3 自监控指标

重点看四个：`otelcol_receiver_accepted_spans` 突然归零说明入口断了，`otelcol_receiver_refused_spans` 持续大于 0 说明过载，`otelcol_exporter_send_failed_spans` 持续增长要告警，`otelcol_exporter_queue_size` 接近上限说明下游慢。Collector 自己的指标同样通过 `/metrics` 暴露，用另一套 Prometheus 抓取，不要和业务关键告警混在一起。

# 8. 练习与自检

1. 起一个只带 `otlp` receiver 和 `debug` exporter 的 Collector，把第 5 篇的 Go 示例改成 OTLP 导出，确认能收到 span；再把 `verbosity` 从 `basic` 改成 `detailed` 对比输出。
2. 先只放 `batch` 观察发送节奏；再加入 `memory_limiter` 并把 `limit_mib` 调到 64，压测观察 `receiver_refused_spans` 是否上升，理解"拒绝"而不是"阻塞"的行为。
3. 写一份 agent + gateway 配置：agent 只做 `batch` 并导出到 gateway，gateway 做 `attributes` 脱敏和 `tail_sampling`。说明为什么尾部采样不能放在 agent 层。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 定位 | Collector 负责缓冲、加工、重试、路由，SDK 只负责产生 |
| 数据流 | Receiver → Processor → Exporter，pipeline 声明顺序与组件 |
| Processor 顺序 | `memory_limiter` 最前，`batch` 最后 |
| 可靠性 | 队列 + 重试 + 可选持久化，必须设 `max_elapsed_time` |
| 尾部采样 | 整条 trace 必须落同一实例，通常放 gateway |
| 部署 | 默认 agent 收、gateway 治；规模小可单层 |
| 排错 | 从 receiver 指标开始，逐跳看到 exporter 与后端 |

下一篇进入具体框架的埋点：Gin 中间件怎么接、GORM 的 SQL 怎么脱敏、gRPC 的 stats handler 与手动 span 怎么配合，以及上下文在哪一层最容易断。
