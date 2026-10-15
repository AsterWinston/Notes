# 1. 版本演进总览

## 1.1 版本号命名规则

```
Kafka 版本格式: kafka_<Scala版本>-<Kafka版本>

例如: kafka_2.13-3.6.0
  - Scala 版本: 2.13
  - Kafka 版本: 3.6.0

注意: kafka_2.12 和 kafka_2.13 只是编译的 Scala 版本不同
      Kafka 功能相同，不要混淆
```

# 2. 关键版本特性对比

## 2.1 0.10.x (2016) — 流处理与时间

| 特性 | 描述 |
|------|------|
| **Kafka Streams** | 首次引入流处理 API，成为流平台 |
| **Timestamp** | 消息增加时间戳（CreateTime / LogAppendTime） |
| **Time Index** | 支持按时间查找 offset |
| **SASL** | 新增 SASL 认证机制 |
| **Rack-Aware** | 副本分配机房感知 |
| **Quota** | 生产和消费配额限制 |
| **消息格式 v1** | 新增 Timestamp 字段 |

```java
// 0.10 新增: 按时间查找
Map<TopicPartition, Long> timestamps = new HashMap<>();
timestamps.put(new TopicPartition("topic", 0), 1460000000000L);
consumer.offsetsForTimes(timestamps);
```

## 2.2 0.11.x (2017) — 事务与精确一次

| 特性 | 描述 |
|------|------|
| **幂等生产者** | PID + Sequence Number 去重 |
| **事务** | 跨分区原子写入 |
| **Exactly-Once** | 端到端精确一次语义 |
| **Headers** | 消息头（支持自定义元数据） |
| **消息格式 v2** | RecordBatch 格式，支持幂等和事务 |
| **JMX 改进** | 增加事务相关指标 |

```java
// 0.11 新增: Headers
ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");
record.headers().add("trace-id", "abc123".getBytes(UTF_8));
record.headers().add("source", "order-service".getBytes(UTF_8));
```

## 2.3 1.x (2017-2018) — 稳定性与 KRaft 奠基

| 特性 | 描述 |
|------|------|
| **Controller 改进** | 减少 Controller 故障恢复时间 |
| **JBOD 改进** | 多磁盘目录日志平衡 |
| **监测指标增强** | 更丰富的 Broker/Consumer 指标 |
| **KIP-98** | 引入 `default.replication.factor` 等 Topic 默认配置 |
| **KRaft 早期** | 开始移除 ZooKeeper 依赖的内部工作 |

```properties
# 1.x 支持更多动态配置
# Broker 级别: log.retention.ms, min.insync.replicas
# 可通过 kafka-configs.sh 动态修改
```

## 2.4 2.x (2019-2022) — 增量平衡与生产可用

| 特性 | 描述 |
|------|------|
| **Incremental Rebalance** | Cooperative Sticky Assignor |
| **静态组员** | `group.instance.id` 避免重启触发 Rebalance |
| **KRaft 预览** | ZooKeeper-Less Kafka 预览 |
| **MirrorMaker 2** | 跨集群复制工具改进 |
| **Tiered Storage** | 分层存储预览 |
| **KIP-500** | 移除 ZooKeeper 路线图的里程碑 |
| **Leader Election 改进** | 更快的 Leader 选举 |

```java
// 2.3+ 增量 Rebalance
props.put("partition.assignment.strategy",
    CooperativeStickyAssignor.class.getName());

// 2.3+ 静态组员
props.put("group.instance.id", "my-consumer-1");
```

## 2.5 3.x (2022+) — KRaft 生产就绪

| 特性 | 描述 |
|------|------|
| **KRaft 生产可用** | 3.3+ 完全替代 ZooKeeper |
| **快照恢复** | Raft 快照，Controller 故障秒级恢复 |
| **独立 Controller** | Broker 和 Controller 可分离部署 |
| **ZooKeeper 弃用** | 3.5 开始移除 ZooKeeper 模式 |
| **Tiered Storage** | 3.6 分层存储预览 |
| **Jumbo Batch** | 更大的批量支持 |
| **性能优化** | 网络层、日志层的多项改进 |

```properties
# 3.3+ KRaft 配置
process.roles=broker,controller
controller.quorum.voters=1@node1:9093,2@node2:9093,3@node3:9093
controller.listener.names=CONTROLLER
```

# 3. Producer 版本特性

## 3.1 Producer 参数演进

| 参数 | 首次引入 | 版本 | 说明 |
|------|---------|------|------|
| `enable.idempotence` | 0.11 | → 3.0+ 默认 true |
| `transactional.id` | 0.11 | 事务支持 |
| `delivery.timeout.ms` | 2.1 | 替代 retries + linger.ms |
| `compression.type=zstd` | 2.0 | 新的压缩算法 |
| `Sticky Partitioner` | 2.4 | 粘性分区（默认） |

```java
// 2.1+ 使用 delivery.timeout.ms
// 旧方式:
props.put("retries", 10);
props.put("max.in.flight.requests.per.connection", 5);

// 新方式:
props.put("delivery.timeout.ms", 120000);
// 内部自动处理重试，不需要手动指定 retries
```

# 4. Consumer 版本特性

## 4.1 Consumer Group 机制演进

| 版本 | 机制 | 变化 |
|------|------|------|
| < 0.9 | Offset 存在 ZK | MQ 式旧版 Consumer |
| 0.9 | 新 Consumer API | Offset 存在 `__consumer_offsets` |
| 0.10.1 | Rebalance 超时扩展 | max.poll.interval.ms 引入 |
| 2.3 | Static Membership | group.instance.id |
| 2.4 | Cooperative Rebalance | 增量式 Rebalance |
| 2.8 | Enforce Rebalance | 修正 Rebalance 协议 |

## 4.2 分配策略演进

| Kafka 版本 | 可用策略 |
|-----------|---------|
| 0.9 | Range（默认） |
| 0.10 | Range, RoundRobin |
| 0.11 | Range, RoundRobin, Sticky |
| 2.4+ | Range, RoundRobin, Sticky, CooperativeSticky（默认 2.5+） |

# 5. Broker 版本特性

## 5.1 日志格式兼容

| 消息格式 | Kafka 版本 | 关键能力 |
|---------|-----------|---------|
| v0 | < 0.10 | 无 timestamp |
| v1 | 0.10 | Timestamp 支持 |
| v2 | 0.11+ | RecordBatch、幂等、事务、Headers |

```properties
# 版本兼容性配置 (Broker 端)
# 设置 Topic 的兼容性
log.message.format.version=2.8

# 设置 Broker 间通信版本
inter.broker.protocol.version=2.8

# 说明: 当升级 Kafka 集群时，这两个参数用于确保兼容性
# 推荐升级流程:
# ① 升级所有 Broker 的二进制
# ② 保持 inter.broker.protocol.version = 旧版本
# ③ 验证集群正常
# ④ 更新 inter.broker.protocol.version = 新版本
# ⑤ 更新 log.message.format.version = 新版本
```

# 6. 安全特性演进

| 版本 | 新增安全特性 |
|------|------------|
| 0.9 | SSL, SASL/PLAIN, SASL/GSSAPI(Kerberos) |
| 0.10 | ACL 支持 |
| 0.11 | Delegation Token |
| 2.0 | SASL/SCRAM, 动态 ACL 更新 |
| 2.1 | SASL/OAUTHBEARER |
| 3.2 | 增强的 PrincipalBuilder |

# 7. 运维特性演进

## 7.1 动态配置改进

| 版本 | 新增动态配置能力 |
|------|---------------|
| 1.0 | Topic 级别: `retention.ms`, `min.insync.replicas` |
| 1.1 | Broker 级别: `leader.replication.throttled.rate` |
| 2.0 | 更细粒度的配额控制 |
| 2.7 | `log.message.timestamp.difference.max.ms` |
| 3.4 | 更丰富的动态 Broker 配置 |

## 7.2 管理工具升级

| 版本 | 工具变化 |
|------|---------|
| 2.2 | `--bootstrap-server` 替代 `--zookeeper` |
| 2.5 | AdminClient API 完善 |
| 2.8 | `kafka-reassign-partitions.sh` 重构 |
| 3.0 | ZK 相关工具开始标记 deprecated |
| 3.6 | KRaft 原生元数据管理工具 |

# 8. 协议 API 版本对比

## 8.1 关键 API Key 版本

| API Key | 首次引入 | 关键变化 |
|---------|---------|---------|
| PRODUCE (0) | 0.8 | v3: 事务支持, v7: ZStd 压缩 |
| FETCH (1) | 0.8 | v4: 支持消息 v2, v7: 增量 Fetch |
| METADATA (3) | 0.8 | v5: Rack-aware, v9: 支持 KRaft |
| JOIN_GROUP (11) | 0.9 | v5: Static Membership, v6: Cooperative |
| FIND_COORDINATOR (10) | 0.9 | v3: 支持多个 Coordinator 类型 |
| INIT_PRODUCER_ID (22) | 0.11 | 幂等和事务的基础 |
| ADD_PARTITIONS_TO_TXN (24) | 0.11 | 事务支持 |
| WRITE_TXN_MARKERS (27) | 0.11 | 事务提交 Marker |

```bash
# 查看 Broker 支持的 API 版本
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# 输出:
# broker1:9092 (id: 0 rack: null) -> (
#     Produce(0): 0 to 9 [usable: 9],
#     Fetch(1): 0 to 13 [usable: 13],
#     ...
# )
```

# 9. KRaft vs ZooKeeper 模式完整对比

| 维度 | ZooKeeper 模式 | KRaft 模式 |
|------|--------------|-----------|
| 外部依赖 | ZooKeeper 集群 | 无 |
| Controller 选举 | ZK 临时节点 | Raft 共识 |
| 元数据存储 | ZK 节点 | `__cluster_metadata` Topic |
| 选举延迟 | ~18s (ZK session 超时) | 秒级 |
| 故障恢复 | 全量读 ZK + 重建 | Raft 快照 |
| 分区数上限 | ~20 万 (ZK 限制) | 百万级 |
| 部署复杂度 | ZK + Kafka 两套集群 | 仅 Kafka |
| 生产可用 | 3.0 稳定 | 3.3+ 生产可用 |
| 未来支持 | 3.5 开始弃用, 4.0 移除 | 主流推荐 |

```bash
# KRaft 模式迁移步骤 (3.3+)
# ① 部署 KRaft Controller 集群
# ② 迁移 ZK 元数据到 KRaft
# ③ 将 Broker 切换为 KRaft 模式
# ④ 停止 ZooKeeper 集群

# 工具: kafka-metadata-quorum.sh
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 \
    describe --status
```

# 10. 版本升级建议

## 10.1 升级路线

```
从老版本迁移推荐路线:

0.10.x → 0.11.x → 1.1.x → 2.0.x → 2.4.x → 2.8.x → 3.3+ (KRaft)
                          ↑
                     (2.4 是关键版本, CooperativeRebalance)
                          ↑
                    (2.8 是 KRaft 预览的起点)

当前 (2024) 推荐版本:
  新项目: 3.6+ (KRaft 模式)
  存量项目: 3.3+ (计划迁移到 KRaft)
  保守: 2.8+ (稳定但已 End-of-Life)
```

## 10.2 集群升级步骤（滚动升级）

```bash
# 假设当前 2.8 → 目标 3.6

# ① 备份所有配置和数据
tar -czf /backup/kafka-config-$(date +%Y%m%d).tar.gz config/

# ② 设置 inter.broker.protocol.version 为当前版本
# server.properties:
inter.broker.protocol.version=2.8
log.message.format.version=2.8

# ③ 逐个升级 Broker 二进制文件（滚动重启）
# 先升级 Controller, 再升级其他 Broker
# 每个 Broker 重启后确认 ISR 恢复正常

# ④ 所有 Broker 重启完成后，更新协议版本
# server.properties:
inter.broker.protocol.version=3.6
# 逐个滚动重启

# ⑤ 更新消息格式版本（可选，仅新 Topic 生效）
# 或使用 kafka-topics.sh --alter 修改已有 Topic
log.message.format.version=3.6
# 逐个滚动重启

# ⑥ 验证升级完成
kafka-broker-api-versions.sh --bootstrap-server localhost:9092
kafka-topics.sh --bootstrap-server localhost:9092 --describe
```
