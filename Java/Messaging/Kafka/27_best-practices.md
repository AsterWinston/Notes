# 1 设计原则

## 1.1 Partition 规划

### 1.1.1 分区数计算

```
分区数 = Max(
    targetThroughput / singlePartitionThroughput,
    consumerConcurrency
)
```

| 维度 | 建议 |
|------|------|
| 最小分区数 | ≥ Consumer Group 中 Consumer 实例数 |
| 最大分区数 | 单 Broker 不超过 4000 个分区，整个集群不超过 200000 个分区 |
| 单分区吞吐 | 通常 10-50 MB/s（依配置而异），保守估算取 10 MB/s |
| 增长预留 | 为未来增长预留 2-3 倍空间 |

### 1.1.2 分区数过多的危害

- 分区元数据更多 → Controller 恢复慢、ZooKeeper 压力大
- 每个分区至少需要 2 个文件句柄（.log + .index）→ 文件句柄耗尽
- 无 Key 消息会退化为多个小 Batch → 网络 I/O 开销增加

### 1.1.3 最佳实践

```
1. 先估算再验证，避免一开始就创建过多分区
2. 使用 kafka-producer-perf-test.sh 测量单分区吞吐
3. 监控 kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
4. 分区数增加后不可减少，谨慎操作
```

## 1.2 Topic 设计

```
命名规范: <domain>.<entity>.<event_type>

示例:
  order.order.created
  payment.transaction.completed
  user.profile.updated
  system.metrics.cpu
  product.inventory.changed
```

### 1.2.1 Topic 配置模板

```properties
# 普通日志 / 监控 Topic
cleanup.policy=delete
retention.ms=604800000           # 7 天
segment.ms=86400000              # 1 天
segment.bytes=1073741824         # 1GB
compression.type=producer        # 不覆盖 Producer 的压缩

# 关键业务 Topic
cleanup.policy=delete
min.insync.replicas=2            # 至少 2 个 ISR
retention.ms=2592000000          # 30 天
replication.factor=3

# 变更日志 / CDC / 快照 Topic
cleanup.policy=compact
min.compaction.lag.ms=86400000   # 24 小时
delete.retention.ms=86400000     # tombstone 保留 24 小时
replication.factor=3
```

# 2 生产者最佳实践

## 2.1 可靠性保障清单

```java
// ✅ 生产级 Producer 配置
Properties props = new Properties();
props.put("acks", "all");                       // ① 等待所有 ISR 确认
props.put("enable.idempotence", true);           // ② 开启幂等
props.put("max.in.flight.requests.per.connection", 5); // ③ 并发度
props.put("retries", Integer.MAX_VALUE);         // ④ 无限重试（配合幂等）
props.put("delivery.timeout.ms", 120000);        // ⑤ 2 分钟总超时
props.put("compression.type", "lz4");            // ⑥ 启用压缩
props.put("batch.size", 131072);                 // ⑦ 128KB 批量
props.put("linger.ms", 10);                      // ⑧ 10ms 额外等待
```

## 2.2 消息 Key 设计

```
消息 Key 的作用:
  ① 分区路由: hash(key) % partitions → 决定消息落在哪个分区
  ② 消息去重: 幂等去重的依据
  ③ 日志 Compact: 按 Key 保留最新值
  ④ 顺序保证: 相同 Key → 同一分区 → 有序

设计原则:
  - 需要有序的消息 → 使用相同 Key
  - 需要均匀分布 → Key 多样性要足够高
  - 避免使用递增 ID 作为 Key → 可能导致热点分区
```

## 2.3 异常处理

```java
public void sendWithProperErrorHandling(String topic, String key, String value) {
    ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, value);

    producer.send(record, (metadata, exception) -> {
        if (exception != null) {
            if (exception instanceof RetriableException) {
                // 可重试异常 → Kafka 已自动重试 → 这里剩余的是 delivery.timeout.ms 超时
                log.error("Retriable error after all retries: {}", exception.getMessage());
                // 入死信队列 / 落库 / 告警
                deadLetterQueue.add(record);
            } else {
                // 不可重试异常 → RecordTooLargeException, SerializationException 等
                log.error("Non-retriable error: {}", exception.getMessage(), exception);
                // 修复问题后重发
            }
        } else {
            log.debug("Sent to {}-{} @ {}", metadata.topic(), metadata.partition(), metadata.offset());
        }
    });
}
```

# 3 消费者最佳实践

## 3.1 消费可靠性保障

```java
// ✅ 生产级 Consumer 配置
Properties props = new Properties();
props.put("enable.auto.commit", false);                   // ① 手动提交
props.put("auto.offset.reset", "earliest");               // ② 最早的 offset
props.put("isolation.level", "read_committed");           // ③ 只读已提交
props.put("max.poll.records", 500);                       // ④ 适度批量
props.put("session.timeout.ms", 60000);                   // ⑤ 60s session
props.put("heartbeat.interval.ms", 10000);                // ⑥ 10s 心跳
props.put("max.poll.interval.ms", 600000);                // ⑦ 10min 处理时间
props.put("partition.assignment.strategy",
    CooperativeStickyAssignor.class.getName());            // ⑧ 增量分配
```

## 3.2 去重模式

```java
@Component
public class IdempotentConsumer {
    // 方案 A: 数据库唯一键去重
    @KafkaListener(topics = "orders")
    @Transactional
    public void consumeWithDb(ConsumerRecord<String, String> record) {
        try {
            orderRepository.insert(new ProcessedMessage(record.key(), record.value()));
        } catch (DuplicateKeyException e) {
            log.warn("Duplicate message: {}", record.key());
            return; // 已处理 → 跳过
        }
        process(record);
    }

    // 方案 B: Redis 去重
    @KafkaListener(topics = "orders")
    public void consumeWithRedis(ConsumerRecord<String, String> record) {
        String dedupKey = "processed:" + record.key();
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(dedupKey, "1", Duration.ofHours(24));
        if (Boolean.FALSE.equals(isNew)) {
            return; // 已处理
        }
        process(record);
    }
}
```

# 4 Broker 配置最佳实践

## 4.1 生产级 Broker 配置

```properties
##############################
# 基础配置
##############################
broker.id=0
log.dirs=/data1/kafka-logs,/data2/kafka-logs  # 多磁盘目录
num.partitions=12                              # 默认分区数
default.replication.factor=3                   # 默认副本数

##############################
# 可靠性
##############################
min.insync.replicas=2                          # 最小 ISR
unclean.leader.election.enable=false           # 不允许非 ISR 选举
auto.leader.rebalance.enable=true              # 自动 Leader 平衡

##############################
# 网络 & I/O
##############################
num.network.threads=8
num.io.threads=16
num.replica.fetchers=4

##############################
# 日志管理
##############################
log.retention.ms=604800000                     # 7 天
log.segment.bytes=1073741824                   # 1GB
log.retention.check.interval.ms=300000         # 5 分钟

##############################
# ZK / KRaft (选其一)
##############################
# ZK 模式
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka
zookeeper.session.timeout.ms=18000

# KRaft 模式 (3.3+)
process.roles=broker,controller
controller.quorum.voters=1@node1:9093,2@node2:9093,3@node3:9093
controller.listener.names=CONTROLLER

##############################
# 内部 Topic
##############################
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

##############################
# 压缩
##############################
compression.type=producer                      # 保留 Producer 的压缩设置
```

# 5 运维最佳实践

## 5.1 部署清单

| 项目 | 建议 |
|------|------|
| JDK | JDK 17+（G1 GC，默认） |
| JVM Heap | 6-64 GB（Xms = Xmx） |
| OS | Linux (ext4/xfs, noatime 挂载) |
| 网络 | 10 Gbps between Brokers |
| 磁盘 | SSD, 独立挂载, 不共享 |
| 监控 | Prometheus + Grafana + AlertaManager |
| 备份 | MirrorMaker 2 / 跨机房复制 |

## 5.2 日常巡检

```bash
#!/bin/bash
# kafka-daily-check.sh

BS="broker1:9092,broker2:9092,broker3:9092"

echo "=== 1. Broker 状态 ==="
kafka-broker-api-versions.sh --bootstrap-server $BS 2>&1 | grep "broker"

echo "=== 2. Under-Replicated Partitions ==="
kafka-topics.sh --bootstrap-server $BS --describe --under-replicated-partitions
echo "(应为空)"

echo "=== 3. Controller Status ==="
# ZK 模式: zookeeper-shell.sh zk1:2181 get /controller | grep brokerid
# KRaft: kafka-metadata-quorum.sh --bootstrap-server $BS describe --status

echo "=== 4. Consumer Group Lag (Top 10) ==="
kafka-consumer-groups.sh --bootstrap-server $BS --list | while read group; do
    kafka-consumer-groups.sh --bootstrap-server $BS --describe --group "$group" 2>/dev/null
done | grep -v "^\s*$" | sort -t',' -k6 -rn | head -20

echo "=== 5. Offline Partitions ==="
kafka-topics.sh --bootstrap-server $BS --describe --unavailable-partitions
echo "(应为空)"

echo "=== 6. 磁盘使用 ==="
kafka-log-dirs.sh --bootstrap-server $BS --describe --broker-list 0,1,2
```

## 5.3 滚动升级规范

```
1. 逐台 Broker 升级（一次一台）
2. 每台升级前确认集群 ISR 恢复
3. 观察至少 10 分钟后再升级下一台
4. 先升级 Controller Broker
5. 升级过程中保持 inter.broker.protocol.version = old
6. 全部 Broker 升级完成后统一升级协议版本
7. 生产环境不要升级到最新版（等一个补丁版本）
```

# 6 容量规划

## 6.1 估算公式

```
磁盘容量 = 日数据量 × retention_days × replication_factor
          / compression_ratio

示例:
  日数据量: 1TB
  保留天数: 7 天
  副本数: 3
  压缩比: 0.7 (30% 压缩)

  磁盘 = 1TB × 7 × 3 / 0.7 = 30TB

  + 10% 系统开销 = 33TB
  分散到 3 台 Broker = 11TB / 台
```

## 6.2 Broker 资源参考

| Producer 吞吐 | 建议 Broker 配置 |
|-------------|----------------|
| < 10 MB/s | 4C/16G/200GB SSD |
| 10-50 MB/s | 8C/32G/500GB SSD |
| 50-200 MB/s | 16C/64G/2TB SSD |
| > 200 MB/s | 32C/128G/4TB NVMe × 多目录 |

# 7 安全最佳实践

```properties
# 最少配置的安全生产集群
listeners=SASL_SSL://0.0.0.0:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false
```

```
ACL 模板:
  Producer: 需要 Topic WRITE + (Cluster IDEMPOTENT_WRITE, 如果开启幂等)
  Consumer: 需要 Topic READ + Group READ
  Admin:    需要 Cluster CREATE + Topic ALTER_CONFIGS
  Monitor:  需要 Topic DESCRIBE + Group DESCRIBE
```

# 8 监控告警

```
关键告警规则:
  1. UnderReplicatedPartitions > 0 → CRITICAL
  2. ActiveControllerCount != 1 → CRITICAL
  3. Consumer Lag > 阈值 → WARNING
  4. OfflinePartitionsCount > 0 → CRITICAL
  5. UnderMinIsrPartitionCount > 0 → WARNING
  6. 磁盘使用率 > 80% → WARNING
  7. NetworkProcessorIdle < 0.2 → WARNING
  8. RequestHandlerIdle < 0.2 → WARNING
```

# 9 面试 / 系统设计高频点

## 9.1 高频原理题

| 问题 | 核心要点 |
|------|---------|
| Kafka 为什么快？ | 顺序写 + Page Cache + 零拷贝 + 批量 + 分区并行 |
| 消息不丢失怎么保证？ | acks=all + min.insync.replicas=2 + 多副本 + 非 Unclean Election |
| 消息有序怎么保证？ | 相同 Key → 相同 Partition + max.in.flight=1 或幂等 |
| Rebalance 怎么优化？ | Cooperative Sticky + Static Membership + 合理超时 |
| Exactly-Once 怎么实现？ | 幂等(PID+SeqNum) + 事务(TransactionalID) + Read Committed |
| ISR 是什么？ | In-Sync Replicas，与 Leader 保持同步的副本，受 replica.lag.time.max.ms 控制 |
| HW 和 LEO 是什么？ | LEO=已写入的最后位置+1，HW=所有 ISR 都确认的位置 |
| KRaft 和 ZK 区别？ | KRaft 自管理元数据，消除 ZK 依赖，秒级 Controller 切换 |

## 9.2 高频设计题

| 场景 | 方案 |
|------|------|
| 海量日志采集 | FileBeat → Kafka → Flink/Logstash → ES（ack=0, 压缩=lz4） |
| 订单系统削峰 | 订单服务 → Kafka → 订单处理服务（acks=all, min.isr=2） |
| 实时统计 UV/PV | 点击流 → Kafka → Kafka Streams（窗口聚合）→ 监控大盘 |
| CDC 数据同步 | Debezium → Kafka → JDBC Sink Connector → 数仓 |
| 跨机房复制 | MirrorMaker 2，异步复制，允许秒级延迟 |

## 9.3 快速参考卡

```
Kafka 版本选择:
  新项目 → 3.6+ KRaft 模式
  存量 → ≥ 2.8，计划迁移

Producer 核心 5 参数:
  acks, enable.idempotence, compression.type, batch.size, linger.ms

Consumer 核心 5 参数:
  enable.auto.commit, auto.offset.reset, max.poll.records,
  session.timeout.ms, isolation.level

Broker 核心 5 参数:
  log.dirs, num.io.threads, min.insync.replicas,
  unclean.leader.election.enable, num.replica.fetchers

故障排查 3 板斧:
  ① kafka-topics --describe --under-replicated
  ② kafka-consumer-groups --describe --group
  ③ kafka-broker-api-versions
```
