# 1 概述：从幂等到事务

## 1.1 消息系统的一致性需求

在分布式消息系统中，我们面临三个层次的一致性保障：

| 级别 | 保障范围 | 实现方式 |
|------|---------|---------|
| 幂等（Idempotence） | 单 Partition、单 Producer Session | PID + Sequence Number |
| 事务（Transaction） | 跨多个 Partition/Topic | Transactional ID + 两阶段提交 |
| 精确一次（Exactly-Once） | 端到端（Produce → Consume → Process） | 事务 Producer + 事务 Consumer |

## 1.2 为什么需要事务

```
经典场景：转账

操作:
  ① 从 Topic "account-A" 消费消息（扣款 100 元）
  ② 向 Topic "account-B" 发送消息（加款 100 元）

问题:
  如果 ① 成功后 Consumer 崩溃:
    → "account-A" 已消费但未完成 ②
    → 钱凭空消失！

事务保证:
  beginTransaction()
    消费 A 的消息（offset + 1）
    生产 B 的消息
  commitTransaction()
  → 要么都成功，要么都不生效
```

# 2 幂等生产者深入

## 2.1 PID 与 Sequence Number 机制

```scala
// Broker 端 ProducerStateManager 的核心逻辑

case class ProducerStateEntry(
  producerId: Long,
  producerEpoch: Short,
  currentTxnFirstOffset: Option[Long],
  coordinatorEpoch: Int,
  lastTimestamp: Long,
  lastSeqNum: Int            // 该 (PID, Partition) 最后一条消息的 SeqNum
)

class ProducerStateManager {
  private val producers = mutable.Map[Long, ProducerStateEntry]()

  def validateAppend(
    producerId: Long,          // Producer ID
    producerEpoch: Short,      // Producer Epoch
    sequenceNum: Int,          // 当前消息的 Sequence Number
    partition: TopicPartition
  ): ValidationResult = {

    getEntry(producerId) match {
      case None =>
        // 第一次看到此 Producer → 允许
        Allow

      case Some(entry) =>
        if (producerEpoch < entry.producerEpoch) {
          // 旧的 Epoch → 拒绝（僵尸 Producer）
          Reject
        } else if (sequenceNum <= entry.lastSeqNum) {
          // 重复消息 → 幂等跳过（返回成功但不存储）
          Duplicate
        } else if (sequenceNum > entry.lastSeqNum + 1) {
          // 序列号跳号 → 有消息丢失！
          OutOfOrderSequence
        } else {
          // sequenceNum == entry.lastSeqNum + 1 → 正常
          Allow
        }
    }
  }
}
```

## 2.2 幂等的局限

```
幂等生产者不能解决:

① Producer 重启后 → PID 重新分配 → 旧 PID 的消息被新 PID 写入
   → 可能出现重复（不同的 PID）

② 跨 Topic 原子写入
   → 向 Topic-A 和 Topic-B 各发一条消息
   → A 成功 B 失败 → 数据不一致

③ 跨 Producer-Consumer 原子性
   → 消费消息 + 处理 + 发送结果 → 整体不能保证原子
```

# 3 事务上下文

## 3.1 事务涉及的角色

```
┌──────────────────────────────────────────────────────────┐
│                     Kafka 事务架构                         │
│                                                          │
│  ┌──────────────────┐     ┌──────────────────────┐       │
│  │ Transactional     │     │ TransactionCoordinator│       │
│  │ Producer          │────▶│ (在某 Broker 上)      │       │
│  │ (transactional.id)│     │                      │       │
│  └──────────────────┘     │ ① 分配 PID           │       │
│                           │ ② 管理 Produce Epoch  │       │
│                           │ ③ 协调 Commit/Abort  │       │
│                           │ ④ 写入 __txn_state  │       │
│                           └──────────┬───────────┘       │
│                                      │                   │
│              ┌───────────────────────┼───────────────┐   │
│              ▼                       ▼               ▼   │
│        ┌──────────┐           ┌──────────┐    ┌──────────┐
│        │ Broker 1 │           │ Broker 2 │    │ Broker 3 │
│        │ P0 data  │           │ P1 data  │    │ P2 data  │
│        │+ TxnMkr  │           │+ TxnMkr  │    │+ TxnMkr  │
│        └──────────┘           └──────────┘    └──────────┘
│                                                          │
│  内部 Topic:                                             │
│  ┌─────────────────────────────────────┐                │
│  │ __transaction_state (事务状态日志)    │                │
│  │ Key: Transactional ID               │                │
│  │ Value: TransactionMetadata          │                │
│  │  - producerId, producerEpoch        │                │
│  │  - txnTimeoutMs, txnStatus          │                │
│  │  - txnStartTimestamp                │                │
│  │  - topicPartitions (涉及的分区)      │                │
│  └─────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────┘
```

# 4 事务 API 与示例

## 4.1 Producer 事务 API

```java
public class TransactionalProducerExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", StringSerializer.class);
        props.put("value.serializer", StringSerializer.class);

        // 启用幂等（事务强制要求）
        props.put("enable.idempotence", true);

        // 设置事务 ID
        props.put("transactional.id", "order-service-tx-01");
        // transactional.id 需要全局唯一
        // 多个 Producer 实例使用相同 transactional.id → 被视为同一个逻辑 Producer

        KafkaProducer<String, String> producer = new KafkaProducer<>(props);

        // ① 初始化事务（向 TransactionCoordinator 注册，获取 PID）
        producer.initTransactions();

        try {
            // ② 开始事务
            producer.beginTransaction();

            // ③ 在事务中发送消息
            producer.send(new ProducerRecord<>("orders", "key1", "order-created"));
            producer.send(new ProducerRecord<>("payments", "key1", "payment-pending"));
            producer.send(new ProducerRecord<>("notifications", "key1", "order-alert"));

            // ④ 提交事务
            producer.commitTransaction();
        } catch (ProducerFencedException e) {
            // 被其他相同 transactional.id 的 Producer 隔离（僵尸检测）
            producer.close();
        } catch (KafkaException e) {
            // ⑤ 回滚事务
            producer.abortTransaction();
        }
    }
}
```

## 4.2 Consumer-Process-Produce 模式（EOS 模式）

```java
public class TransactionalConsumeProcessProduce {
    public static void main(String[] args) {
        // === Producer 配置（事务模式）===
        Properties producerProps = new Properties();
        producerProps.put("bootstrap.servers", "localhost:9092");
        producerProps.put("key.serializer", StringSerializer.class);
        producerProps.put("value.serializer", StringSerializer.class);
        producerProps.put("transactional.id", "eos-processor-01");

        KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps);
        producer.initTransactions();

        // === Consumer 配置（只读已提交消息）===
        Properties consumerProps = new Properties();
        consumerProps.put("bootstrap.servers", "localhost:9092");
        consumerProps.put("group.id", "eos-processor-group");
        consumerProps.put("key.deserializer", StringDeserializer.class);
        consumerProps.put("value.deserializer", StringDeserializer.class);
        consumerProps.put("enable.auto.commit", false);
        consumerProps.put("isolation.level", "read_committed");
        // read_committed: 只读取已提交的事务消息（跳过未提交的）

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
        consumer.subscribe(Collections.singletonList("source-topic"));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

            // 每个 poll 作为一个事务处理
            producer.beginTransaction();

            try {
                for (ConsumerRecord<String, String> record : records) {
                    // 处理消息
                    String processedValue = processMessage(record.value());

                    // 将处理结果发送到目标 Topic
                    producer.send(new ProducerRecord<>(
                            "target-topic", record.key(), processedValue));
                }

                // 构建包含 offset 的 Consumer Group Metadata
                Map<TopicPartition, OffsetAndMetadata> offsets =
                    buildOffsetMap(records);

                // 将 Consumer offset 发送到事务中
                producer.sendOffsetsToTransaction(offsets, "eos-processor-group");

                // 提交事务：同时提交 offset 和产出的消息
                producer.commitTransaction();
            } catch (Exception e) {
                // 处理失败 → 回滚（offset 不提交，消息不写入）
                producer.abortTransaction();
                // 回滚后 Consumer 会重新消费这部分消息
            }
        }
    }
}
```

# 5 事务底层原理

## 5.1 两阶段提交协议

```
Kafka 事务使用简化的两阶段提交：

阶段 1: PREPARE_COMMIT（准备阶段）
  TransactionCoordinator:
   ① Producer 发送 EndTxnRequest(COMMIT)
   ② Coordinator 在 __transaction_state 中写入 PREPARE_COMMIT
   ③ Coordinator 向涉及的各 Partition Leader 发送 WriteTxnMarkersRequest
   ④ 各 Leader 在 Partition 日志中写入 COMMIT Marker

阶段 2: COMMITTED（提交完成）
   ⑤ 所有 Partition 的 Marker 写入完成
   ⑥ Coordinator 在 __transaction_state 中写入 COMPLETE_COMMIT
   ⑦ 事务完成

流程图:
┌────────────┐     ① EndTxnRequest     ┌────────────────────┐
│  Producer  │ ────────────────────────▶│ TransactionCoordinator│
│            │                          │                    │
└────────────┘                          │ ② 写入 __txn_state │
                                        │    PREPARE_COMMIT  │
                                        │                    │
                                        │ ③ WriteTxnMarkers │
                                        │    Request          │
                                        │    ┌─────────┐     │
                                        ├───▶│Broker P0│     │
                                        │    └─────────┘     │
                                        │    ┌─────────┐     │
                                        ├───▶│Broker P1│     │
                                        │    └─────────┘     │
                                        │                    │
                                        │ ④ 各 Broker 返回   │
                                        │    成功             │
                                        │                    │
                                        │ ⑤ 写入 __txn_state │
                                        │    COMPLETE_COMMIT │
                                        └────────────────────┘
```

## 5.2 Transaction Marker（事务标记）

事务消息在磁盘上的特殊标记：

```
Partition 日志中的事务消息:
┌────────────────────────────────────────────────────────┐
│ Batch: PID=123, TxnMarker=COMMIT, ...                  │
├────────────────────────────────────────────────────────┤
│ Batch: PID=123, SeqNum=0, TxnData, offset=100          │ ← 事务中的消息
│ Batch: PID=123, SeqNum=1, TxnData, offset=101          │
│ Batch: PID=123, TxnData, offset=102                    │
│ ...                                                    │
│ ──── 事务边界 ────                                      │
│ Batch: PID=123, TxnMarker=COMMIT, offset=103           │ ← 提交标记
│ Batch: PID=456, SeqNum=0, non-txn data, offset=104    │ ← 非事务消息
└────────────────────────────────────────────────────────┴

Consumer 读取行为 (isolation.level=read_committed):
  读取到 offset=102 的事务消息 → 缓存（不放回给用户）
  读取到 offset=103 的 COMMIT Marker → 将缓存的事务消息一并返回
  → 用户只看到已提交的事务消息

  (isolation.level=read_uncommitted) → 不等待 Marker，直接返回所有消息
```

## 5.3 Producer Fencing（僵尸隔离）

```
场景: 两个 Producer 实例使用相同的 transactional.id

Producer A (epoch=0):
  正在执行事务 → 发送消息 P0, P1

时间 ──────────────────────────────────────▶

Producer A 网络分区/假死 ─┤
                           │
Producer B (epoch=1) 启动 ─┤
  → initTransactions()
  → TransactionCoordinator 返回 epoch=1
  → epoch 递增（0 → 1）

Producer A 恢复，尝试继续事务：
  → endTxn(COMMIT, epoch=0)
  → Coordinator 检查: epoch=0 < currentEpoch=1
  → 返回 ProducerFencedException
  → Producer A 无法提交，事务被中止

结果:
  旧 Producer A 无法"复活"并提交可能已过期的数据
  新 Producer B 从头开始
```

# 6 Consumer 的隔离级别

## 6.1 read_uncommitted vs read_committed

```java
// Consumer 配置
props.put("isolation.level", "read_committed");

// read_uncommitted (默认):
//   → 读取所有消息，包括未提交的事务消息
//   → 低延迟，但可能读到"脏数据"

// read_committed:
//   → 仅读取已提交的事务消息
//   → 等待 COMMIT/ABORT Marker 后再返回
//   → 可能增加延迟（需要等待 Marker）
```

### 6.1.1 对 LSO (Last Stable Offset) 的影响

```
Partition 中:

HW (High Watermark) = 所有 ISR 副本都确认的最大 Offset
LSO (Last Stable Offset) = 最后一个已提交的事务消息的 Offset

对于 read_committed Consumer:
  最多消费到 min(HW, LSO)

示例:
  offset=100: msg1 (commit)
  offset=101: msg2 (commit)
  offset=102: txn_msg (未提交)  ← LSO = 102
  offset=103: msg3 (普通消息)
  HW = 104

  read_uncommitted: 可读到 offset=103
  read_committed:   只能读到 offset=101（被 pending 事务阻塞）
```

# 7 事务相关的内部 Topic 和参数

## 7.1 __transaction_state Topic

```properties
# 事务状态日志配置
transaction.state.log.replication.factor=3
transaction.state.log.num.partitions=50
transaction.state.log.min.isr=2
transaction.state.log.segment.bytes=104857600  # 100MB

# Transactional ID 到 Transaction Coordinator 的映射:
# Coordinator = hash(transactional.id) % __transaction_state.partitions
# 该分区的 Leader 所在 Broker 即为 TransactionCoordinator
```

## 7.2 事务超时参数

```properties
# Producer 端
transaction.timeout.ms=60000           # 事务超时（默认 1 分钟）
# Producer 在此时间内必须提交或中止事务，否则 Broker 自动中止

# Consumer 端
isolation.level=read_committed         # 隔离级别

# Broker 端
transactional.id.expiration.ms=604800000  # transactional.id 过期时间 (7天)
# Producer 恢复事务状态的最大时间
```

# 8 Exactly-Once Semantics (EOS) 详解

## 8.1 精确一次的配置要求

```
EOS = 幂等 Producer + 事务 Producer + 事务 Consumer

Producer 端:
  enable.idempotence=true
  transactional.id=<unique-id>
  (自动设置 acks=all, retries=MAX, max.in.flight ≤ 5)

Broker 端:
  transaction.state.log.replication.factor=3
  transaction.state.log.min.isr=2

Consumer 端:
  isolation.level=read_committed
  enable.auto.commit=false
```

## 8.2 EOS 的完整链路

```
┌──────────────────────────────────────────────────────────┐
│            Exactly-Once Semantic 端到端流程               │
│                                                          │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐          │
│  │Producer A │────▶│ Topic X  │────▶│Consumer B│          │
│  │(幂等+事务) │     │          │     │(read_commit)        │
│  └──────────┘     └──────────┘     └────┬─────┘          │
│                                         │                │
│                                   B 在事务中处理:         │
│                                   ┌───▼─────────────────┐ │
│                                   │ beginTransaction()  │ │
│                                   │ consume from X      │ │
│                                   │ produce to Y        │ │
│                                   │ sendOffsets         │ │
│                                   │ commitTransaction() │ │
│                                   └───┬─────────────────┘ │
│                                       │                   │
│                                       ▼                   │
│                                  ┌──────────┐            │
│                                  │ Topic Y  │            │
│                                  └──────────┘            │
│                                                          │
│  完整链路的 Exactly-Once 保证:                            │
│    ① A → X: 幂等生产者 + 事务保证每条消息恰好写入一次       │
│    ② X → B: B 只读取已提交消息 (read_committed)           │
│    ③ B → Y: B 的 offset 提交和消息生产在同一事务中        │
│    → 整个链路要么全部成功，要么全部回滚                     │
└──────────────────────────────────────────────────────────┘
```

## 8.3 EOS 对性能的影响

| 方面 | 影响 | 原因 |
|------|------|------|
| 写延迟 | +10~20% | 需要等待 Transaction Marker 写入 |
| Consumer 延迟 | +数 ms~数百 ms | read_committed 等待 Marker |
| 吞吐 | 轻微下降 | 额外的事务元数据开销 |
| 磁盘 I/O | 增加 | __transaction_state 的写操作 |

# 9 事务最佳实践

## 9.1 使用建议

1. **transactional.id 唯一性**：每个逻辑 Producer 实例使用唯一 ID
2. **事务尽量短**：减少事务持有时长，避免大量消息在一个事务中
3. **合理设置超时**：`transaction.timeout.ms` 不宜过大（避免资源占用）也不宜过小（避免超时中止）
4. **监控事务指标**：
   ```
   kafka.server:type=TransactionCoordinatorMetrics
   kafka.coordinator.transaction:type=Summary
   ```
5. **handle ProducerFencedException**：正确处理僵尸 Producer 隔离

## 9.2 常见问题

```
问题 1: ProducerFencedException
原因: 另一个相同 transactional.id 的 Producer 已注册
解决: 确保 transactional.id 唯一，关闭旧 Producer 实例

问题 2: TimeoutException on commit
原因: 事务超时
解决: 增大 transaction.timeout.ms 或减少事务中消息量

问题 3: read_committed Consumer 读取延迟高
原因: 等待 pending 事务的 Marker
解决: 检查卡住的事务（监控 __transaction_state）

问题 4: 事务状态日志膨胀
原因: 大量 transactional.id
解决: 配置合理的 retention 和 compaction
```
