Kafka 上线之后，开发关注的"消息发没发出去"会变成运维关注的"集群还活着吗、lag 为什么涨、磁盘还剩多少、谁能读这个 Topic"。这些问题靠翻代码解决不了，得靠指标、容量估算和安全配置。本篇把 Broker 监控、lag 排查、磁盘规划、分区估算和 TLS/SASL/ACL 串成一套可落地的运维清单。

# 1. 运维的第一张图

## 1.1 集群健康

```text
┌─────────────────────────────────────────────────────┐
│                     Kafka Cluster                   │
│                                                     │
│  Controller（KRaft 选举，集群唯一）                  │
│      │                                              │
│      ├── Broker 1  ── 磁盘 / 网络 / 请求队列          │
│      ├── Broker 2  ── 磁盘 / 网络 / 请求队列          │
│      └── Broker 3  ── 磁盘 / 网络 / 请求队列          │
│                                                     │
│  Topic: orders                                      │
│    P0: leader=B1, ISR={B1,B2,B3}                    │
│    P1: leader=B2, ISR={B2,B3}                       │
│    P2: leader=B3, ISR={B1,B3}                       │
└─────────────────────────────────────────────────────┘
```

运维要先看三件事：

- **控制面**：Controller 是否唯一、选举是否频繁。
- **数据面**：有没有离线分区、ISR 是否稳定、请求是否排队。
- **容量**：磁盘、网络、文件句柄是否接近上限。

## 1.2 指标分层

| 层级 | 关注点 | 典型指标 |
|------|--------|----------|
| Broker | 进程与资源 | CPU、GC、磁盘、网络、请求队列 |
| Topic/Partition | 副本与分布 | ISR、leader 分布、分区大小 |
| Consumer Group | 消费进度 | lag、提交频率、rebalance 次数 |
| Client | 客户端健康 | 发送错误率、重试、请求延迟 |

★ **关键认知**：监控要覆盖**四个黄金信号**：延迟、流量、错误、饱和度。Kafka 的延迟看请求耗时和 lag，流量看 BytesIn/Out 和 MessagesIn，错误看失败请求和 ISR 收缩，饱和度看磁盘、网络、请求队列和文件句柄。

# 2. 监控指标

## 2.1 Broker 指标

| 指标 | 含义 | 告警建议 |
|------|------|----------|
| `ActiveControllerCount` | 当前 Controller 数量 | 集群总和必须为 1 |
| `OfflinePartitionsCount` | 没有 leader 的分区数 | `>0` 立即告警 |
| `UnderReplicatedPartitions` | 副本不足的分区数 | 持续 `>0` 告警 |
| `IsrShrinksPerSec` | ISR 收缩速率 | 持续大于 0 要查 |
| `RequestHandlerAvgIdlePercent` | 请求处理线程空闲率 | 低于 20% 说明过载 |
| 磁盘使用率 | `log.dirs` 剩余空间 | 70% 预警，85% 严重 |

这些指标通过 JMX 暴露，生产上通常用 JMX Exporter 转成 Prometheus 格式，再配 Grafana 面板。

⚠ **关键认知**：`UnderReplicatedPartitions` 短暂大于 0 不一定是故障，滚动重启、网络抖动都会造成短暂收缩。真正要关注的是**持续不恢复**，以及伴随 `OfflinePartitionsCount > 0` 的情况。

## 2.2 Topic 与分区指标

- **分区数**：增长是否和业务规划一致。
- **Leader 分布**：是否集中在少数 Broker。
- **ISR 大小**：是否长期小于副本数。
- **分区大小**：是否有明显倾斜。
- **High Watermark / LEO**：判断写入和同步进度。

```bash
  # 查看 Topic 的分区、leader、ISR
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders

  # 查看所有 Topic 的分布
kafka-topics.sh --bootstrap-server localhost:9092 --describe
```

## 2.3 客户端指标

kafka-go 的 `Writer.Stats()` 和 `Reader.Stats()` 会暴露发送/拉取次数、错误数、重试、延迟、队列长度等。生产上至少采集：

| 指标 | 含义 |
|------|------|
| 发送错误率 | `Errors / Writes` |
| 重试次数 | 网络或 Broker 问题的信号 |
| 请求延迟 p99 | 客户端视角的尾延迟 |
| 队列长度 | 异步写入是否积压 |
| Reader lag | 消费进度 |
| 提交失败次数 | offset 是否稳定推进 |

## 2.4 lag 监控

lag 是最重要的业务指标：

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-worker
```

输出里关注 `CURRENT-OFFSET`、`LOG-END-OFFSET`、`LAG`。生产上不要只看"lag 是否大于阈值"，还要看：

- **趋势**：持续上涨还是周期性波动。
- **时间 lag**：当前时间减去最后一条消费消息的时间戳，比消息条数更能反映业务延迟。
- **分组维度**：按消费组、Topic、分区分别看，避免平均值掩盖问题。

常用工具：Burrow（专门算 lag 和消费健康度）、Kafka Exporter（Prometheus）、Cruise Control（分区均衡和容量）。

★ **关键认知**：lag 的绝对值没有统一标准。100 万条对日志管道可能正常，对支付系统就是事故。告警阈值要结合"单条处理耗时 × lag"估算业务延迟，而不是照搬别人的数字。

# 3. 磁盘与容量

## 3.1 磁盘占用估算

```text
每天原始字节 = 每秒消息数 × 平均消息大小 × 86400
总占用 = 每天原始字节 × 保留天数 × 副本数 × (1 + 索引/段开销)
```

示例：每秒 2,000 条、平均 1KB、保留 7 天、副本 3：

```text
每天 = 2000 × 1KB × 86400 ≈ 165 GB
7 天 × 3 副本 ≈ 3.5 TB
加 20% 余量 ≈ 4.2 TB
```

压缩会降低实际占用，但不要按"压缩比 5 倍"做满打满算的规划；业务峰值、副本追赶、Segment 未回收都会让实际值高于估算。

## 3.2 保留策略

| 配置 | 作用 |
|------|------|
| `retention.ms` | 按时间保留，默认 7 天 |
| `retention.bytes` | 按分区大小保留 |
| `cleanup.policy=delete` | 按保留策略删除 |
| `cleanup.policy=compact` | 按 key 保留最新值 |

`compact` 适合"键值状态"类 Topic，比如用户配置、订单最新状态。它不会因为时间删除旧消息，而是保留每个 key 的最新值，必须配合 `min.cleanable.dirty.ratio` 和 `delete.retention.ms` 使用。

## 3.3 磁盘满的后果

磁盘满不是"写入变慢"，而是直接故障：

- Broker 写日志失败，分区可能下线。
- 生产者收到 `KafkaStorageException` 或超时。
- Controller 可能触发 leader 切换，引发连锁反应。
- 如果 `log.dirs` 配了多块盘，一块满可能影响该盘上的所有分区。

预防手段：70% 预警、80% 开始清理、保留策略留足余量、监控 Segment 增长速率、必要时启用分层存储（Kafka 3.6+ 逐步成熟）。

⚠ **关键认知**：不要用 `rm` 直接删 Kafka 日志目录里的文件。Kafka 自己管理 Segment 和索引，手工删除会导致索引错乱、offset 越界，甚至数据不可读。要清理就调保留策略或使用管理工具。

# 4. 分区规划

## 4.1 分区数量的影响

| 影响 | 说明 |
|------|------|
| 并行度 | 消费并行度上限等于分区数 |
| 元数据 | 分区越多，Controller 和客户端元数据越大 |
| Rebalance | 分区越多，分配和迁移越慢 |
| 故障恢复 | leader 选举和副本同步更耗时 |

## 4.2 估算方法

```text
分区数 = max(
  生产峰值吞吐 / 单分区写入能力,
  消费峰值吞吐 / 单分区消费能力
) × 余量系数
```

经验值：单分区写入按 5-10MB/s、消费按 10-20MB/s 保守估算，余量系数取 2。比如峰值写入 50MB/s，按 10MB/s 一个分区算，至少 5 个分区，乘 2 后取 10 个。

还要考虑：

- 单 Broker 的分区总数控制在 2,000-4,000 以内（视硬件和版本）。
- 消费者实例数不能超过分区数，规划时要和部署副本数对齐。
- 分区数一旦增加不能减少，宁可初期多留一点余量。

## 4.3 扩容分区

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic orders --partitions 12
```

扩容只影响新消息的分区映射，已有消息不会迁移。但 `hash(key) % 分区数` 的结果会变，同一个 key 的新消息可能落到不同分区，顺序性会受影响。

★ **关键认知**：分区扩容是**单向操作**，不能减回去。扩容前要确认：消费端能否容忍 key 重分布、下游是否依赖分区内顺序、监控和容量是否跟得上。能提前规划就不要等到 lag 爆炸再扩。

# 5. 安全

## 5.1 传输加密 TLS

Broker 端配置证书和监听协议：

```text
listeners=SASL_SSL://:9093
ssl.keystore.location=/etc/kafka/server.keystore.jks
ssl.truststore.location=/etc/kafka/server.truststore.jks
```

客户端需要信任 Broker 的 CA；如果开启双向 TLS，还要提供客户端证书。

## 5.2 认证 SASL

| 机制 | 特点 | 适用 |
|------|------|------|
| `PLAIN` | 用户名密码，必须配合 TLS | 内网简单场景 |
| `SCRAM-SHA-256/512` | 挑战响应，密码不裸传 | 生产推荐 |
| `GSSAPI` | Kerberos | 已有 Kerberos 体系 |
| `OAUTHBEARER` | Token | 云环境、统一身份 |

Broker 要开启对应的 `sasl.enabled.mechanisms`，并配置 JAAS。监听协议通常是 `SASL_SSL`，不要把 `SASL_PLAINTEXT` 暴露到公网。

## 5.3 授权 ACL

ACL 由 principal、资源、操作三部分组成：

```bash
  # 允许 order-service 写 orders Topic
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:order-service \
  --operation Write --topic orders

  # 允许 order-worker 读 orders 并提交 offset
kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:order-worker \
  --operation Read --topic orders \
  --group order-worker
```

常见操作：`Read`、`Write`、`Describe`、`Create`、`Delete`、`Alter`。生产上按"最小权限"分配，不要给业务账号 `ClusterAction` 或 `All`。

## 5.4 Go 客户端配置

kafka-go 通过 `Dialer`（Reader）和 `Transport`（Writer）配置 TLS/SASL：

```go
mech, err := scram.Mechanism(scram.SHA256, "order-service", "password")
if err != nil {
    log.Fatal(err) // 参数错误或算法不支持
}

dialer := &kafka.Dialer{
    Timeout:       10 * time.Second,
    DualStack:     true,
    SASLMechanism: mech,
    TLS:           &tls.Config{MinVersion: tls.VersionTLS12},
}

r := kafka.NewReader(kafka.ReaderConfig{
    Brokers: []string{"broker-1:9093", "broker-2:9093"},
    GroupID: "order-worker",
    Topic:   "orders",
    Dialer:  dialer,
})

w := &kafka.Writer{
    Addr:  kafka.TCP("broker-1:9093", "broker-2:9093"),
    Topic: "orders",
    Transport: &kafka.Transport{
        SASL: mech,
        TLS:  &tls.Config{MinVersion: tls.VersionTLS12},
    },
}
```

★ **关键认知**：TLS 解决"传输不被窃听"，SASL 解决"你是谁"，ACL 解决"你能做什么"。三者是三层不同的东西，不能互相替代。只开 TLS 不代表有权限控制，只配 ACL 也不代表密码不会被抓包。

# 6. 常见故障与处理

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `OfflinePartitionsCount > 0` | Broker 下线、磁盘故障 | 先恢复 Broker，再查副本 |
| ISR 持续收缩 | 磁盘慢、网络差、GC 长 | 查 IO、网络、JVM |
| 磁盘写满 | 保留策略过长、流量突增 | 调 retention、扩容、清理 |
| `REQUEST_TIMED_OUT` | Broker 过载、网络延迟 | 看请求队列和 p99 |
| `NOT_ENOUGH_REPLICAS` | ISR 小于 `min.insync.replicas` | 恢复副本或调低 minISR |
| `OFFSET_OUT_OF_RANGE` | offset 超出保留范围 | 重置 offset 到 earliest/latest |
| rebalance 风暴 | 频繁扩缩容、处理超时 | 放慢发布、调大超时 |

⚠ **关键认知**：故障处理的第一原则是"先止损，再定位"。lag 暴涨时先扩消费者或限流生产，保住业务；磁盘快满时先调短 retention 或扩容，避免 Broker 挂掉。不要在故障现场做大规模分区重分配。

# 7. 动手练习

1. 用 `kafka-topics.sh --describe` 和一个 JMX 指标（如 `UnderReplicatedPartitions`）搭一个最小监控，模拟停掉一个 Broker，观察指标变化。
2. 按本篇公式估算一个"每秒 5,000 条、每条 2KB、保留 3 天、副本 3"的 Topic 需要多少磁盘，再用实际 `du` 结果对比；然后开启 SCRAM-SHA-256，用 kafka-go 的 `Dialer` 连接并验证只读账号写入会被拒绝。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 健康检查 | Controller 唯一、无离线分区、ISR 稳定 |
| 指标 | 延迟、流量、错误、饱和度四类 |
| lag | 看趋势和时间延迟，不只看绝对值 |
| 磁盘 | 按吞吐 × 保留 × 副本估算，留 20% 余量 |
| 保留 | delete/compact 按业务选，别手工删文件 |
| 分区 | 只增不减，扩容会改变 key 映射 |
| 安全 | TLS 加密、SASL 认证、ACL 授权，三层配合 |
| 故障 | 先止损再定位，避免故障时重分配 |

下一篇是模块收尾：把消息设计、重试与死信、顺序性、容量规划和上线检查清单整理成一套可执行的最佳实践，并用自检题检验前面八篇的理解。
