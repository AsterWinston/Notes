# 1 消息投递语义概述

## 1.1 分布式系统的经典"三元组"

在分布式消息系统中，消息投递有三种语义：

| 语义 | 含义 | 消息可能结果 |
|------|------|-------------|
| At-Most-Once（最多一次） | 消息最多被投递一次，可能丢失 | 0 次 或 1 次 |
| At-Least-Once（至少一次） | 消息至少被投递一次，可能重复 | 1 次 或 N 次 |
| Exactly-Once（精确一次） | 消息恰好被投递一次 | 恰好 1 次 |

## 1.2 Kafka 各语义的支撑能力

```
┌──────────────────────────────────────────────────────────────┐
│                Kafka 投递语义全景图                            │
│                                                              │
│  Producer → Broker                                           │
│  ┌─────────────┬──────────────────────────────────────────┐  │
│  │ At-Most-Once│ acks=0, retries=0                        │  │
│  │ At-Least-Once│ acks=1/all, retries>0                   │  │
│  │ Exactly-Once │ 幂等 Producer + 事务 Producer            │  │
│  └─────────────┴──────────────────────────────────────────┘  │
│                                                              │
│  Broker → Consumer                                           │
│  ┌─────────────┬──────────────────────────────────────────┐  │
│  │ At-Most-Once│ 先提交 offset, 再处理消息                 │  │
│  │ At-Least-Once│ 先处理消息, 再提交 offset                │  │
│  │ Exactly-Once │ 事务 Consumer (offset提交+业务处理原子化) │  │
│  └─────────────┴──────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

# 2 Producer 端语义实现

## 2.1 At-Most-Once（最多一次）

```java
// 配置：不等确认，不重试
Properties props = new Properties();
props.put("acks", "0");        // 不等待 Broker 确认
props.put("retries", 0);       // 发送失败不重试

// 结果：
//   Producer 发送后不关心是否成功
//   如果 Broker 宕机或网络超时 → 消息丢失
//   使用场景：日志收集、不重要的监控指标
```

### 2.1.1 At-Most-Once 的工作流

```
Producer.send(msg)
    │
    ├── 成功 → 消息可能到达 Broker
    │
    └── 失败 → 消息丢弃（无重试）
        
结果: 消息最多被处理 1 次
```

## 2.2 At-Least-Once（至少一次）

```java
// 配置：等待确认，自动重试
Properties props = new Properties();
props.put("acks", "all");               // 等待所有 ISR 确认
props.put("retries", Integer.MAX_VALUE);// 失败无限重试
props.put("enable.idempotence", false); // 不开启幂等

// 结果：
//   Broker 已写入但 ACK 网络超时 → Producer 重试 → 消息重复
//   Broker 宕机前写入但未响应 → Producer 重试 → 消息重复
//   使用场景：大部分业务场景
```

### 2.2.1 At-Least-Once 可能产生重复的场景

```
场景 1: ACK 网络超时
  Producer→Broker:  写入 msg (offset=100) ✓ Broker 写入成功
  Broker→Producer:  ACK 由于网络丢失 ✗ Producer 没收到
  Producer→Broker:  重试发送 msg → Broker 写入 offset=101
  → 消息重复！

场景 2: Leader 切换
  Producer→Leader:  写入 msg (offset=100) → Leader 宕机
  Producer→新 Leader: 重试发送 msg → 新 Leader 写入 offset=100
  → 消息可能重复（取决于 Leader 切换时的截断机制）

场景 3: Producer retries 与 in-flight requests
  Batch1 发送 → 超时 → Producer 重试
  但 Batch1 实际已经写入，只是 ACK 未返回
  → 消息重复写入
```

## 2.3 Exactly-Once（精确一次）——Producer 端

```java
// 配置：幂等 + 事务
Properties props = new Properties();
props.put("acks", "all");               // 自动设置
props.put("enable.idempotence", true);   // 开启幂等
// 幂等模式内建：
//   ① PID + Sequence Number 去重
//   ② retries=MAX
//   ③ max.in.flight.requests ≤ 5
//   ④ 不涉及事务 ID → 仅单 Partition + 单 Session 保证

// 跨 Partition 的事务 EOS:
props.put("transactional.id", "my-unique-id"); // 事务模式
producer.initTransactions();
// 保证跨多个 Partition 和跨 Producer Session 的 Exactly-Once
```

# 3 Consumer 端语义实现

## 3.1 At-Most-Once（最多一次）

```java
// 先提交 offset，再处理消息
props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    // offset 在 poll 时自动提交（内部异步）

    for (ConsumerRecord<String, String> record : records) {
        process(record); // 处理过程中可能崩溃
    }
    // 问题: poll 时就已提交了上一批的 offset
    //       如果当前批处理中崩溃 → 这批消息丢失
}

// 或: 手动先提交后处理
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    consumer.commitSync(); // 先提交
    for (ConsumerRecord<String, String> record : records) {
        process(record); // 处理失败 → 消息丢失（offset 已提交）
    }
}
```

## 3.2 At-Least-Once（至少一次）

```java
// 先处理消息，后提交 offset（推荐）
props.put("enable.auto.commit", false);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

    for (ConsumerRecord<String, String> record : records) {
        process(record); // 处理可能成功
    }

    consumer.commitSync(); // 处理完成后提交
    // 问题: 处理成功但提交失败 → 重启后重复消费这批消息
}
```

### 3.2.1 At-Least-Once 的重复场景

```
场景 1: 提交失败
  poll() → 获得 [offset:100-109]
  process([100-109]) → 全部处理成功
  commitSync() → 网络超时，提交失败
  Consumer 重启 → 从 offset=100 重新消费 → 重复！

场景 2: Rebalance 触发
  Consumer A 正在处理 [offset:100-109]
  Rebalance 触发 → Consumer A 失去分区
  在 onPartitionsRevoked 中提交 → 只提交了 100-104
  Consumer B 接替 → 从 105 开始消费
  → offset:105-109 可能被 A 和 B 都处理 → 部分重复

场景 3: 异步提交失败静默覆盖
  poll batch 1 → 处理 → commitAsync (fire and forget)
  poll batch 2 → 处理 → commitAsync
  batch 2 提交成功，但 batch 1 提交失败（被后续成功覆盖）
  Consumer 重启 → batch 1 重复消费
```

## 3.3 Exactly-Once（精确一次）——Consumer 端

### 3.3.1 方案一：事务 Consumer（推荐）

```java
// 消费 + 处理 + 生产 在一个事务中完成

producer.initTransactions();
consumer.subscribe(Collections.singletonList("source"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

    producer.beginTransaction();
    try {
        for (ConsumerRecord<String, String> record : records) {
            String result = process(record);
            producer.send(new ProducerRecord<>("target", record.key(), result));
        }

        // 将 offset 加入事务
        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
        for (TopicPartition tp : records.partitions()) {
            long lastOffset = // 该分区最后一条消息的 offset + 1
            offsets.put(tp, new OffsetAndMetadata(lastOffset));
        }
        producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata().groupId());

        // 原子提交：offset 提交 + 消息生产
        producer.commitTransaction();
    } catch (Exception e) {
        producer.abortTransaction();
    }
}
```

### 3.3.2 方案二：幂等处理 + At-Least-Once 消费

```java
// 不依赖 Kafka 事务，通过业务幂等实现去重

// 每条消息带唯一 ID：
// Producer 端: message key = UUID.randomUUID()
// Consumer 端: 用数据库去重

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, String> record : records) {
        String msgId = record.key(); // 唯一消息 ID

        // 数据库实现去重
        if (processedMessageRepository.existsByMsgId(msgId)) {
            continue; // 已处理，跳过
        }
        // 处理消息
        process(record);
        // 记录已处理
        processedMessageRepository.save(msgId);
    }
    consumer.commitSync();
}
```

### 3.3.3 方案三：最终一致性表 + 本地事务

```java
// 在业务数据库中建"消费记录表"
// 消费消息 → 写业务数据 + 写消费记录 → 本地数据库事务保证原子

@Transactional
public void consume(ConsumerRecord<String, String> record) {
    String msgId = record.key();

    // 检查去重
    if (consumerRecordDao.exists(msgId)) {
        return;
    }

    // 执行业务逻辑
    businessService.process(record);

    // 记录消费（同一数据库事务）
    consumerRecordDao.save(new ConsumerRecordEntity(msgId, record.value()));
}

// 定期同步提交 Kafka offset：
// 找出数据库中连续消费的最大 offset → 提交到 Kafka
// 即使提交不精确（提交少了 → At-Least-Once, 靠去重补偿）
```

# 4 各语义对比总结

## 4.1 实现复杂度与适用场景

```
At-Most-Once → 最简单
  场景: 日志、Metrics、非关键数据
  代价: 可能丢消息
  配置: acks=0, enable.auto.commit=true (自动提交)

At-Least-Once → 中等
  场景: 大部分业务（异步解耦、削峰）
  代价: 可能重复（需要消费者处理幂等）
  配置: acks=all, enable.auto.commit=false (手动提交)

Exactly-Once → 复杂
  场景: 金融交易、订单、支付、库存
  代价: 性能开销、开发复杂度
  配置: enable.idempotence=true + transactional.id + read_committed
```

## 4.2 端到端 Exactly-Once 的完整要求

```
实现端到端 Exactly-Once 需要的条件:

① Producer → Kafka: Exactly-Once (幂等 + 事务)
② Kafka 存储: 多副本 + ISR
③ Kafka → Consumer: 事务 Consumer + read_committed
④ Consumer → 业务系统: 事务性输出（需要输出系统支持事务）
   例如: 数据库事务, Kafka 事务 Consumer-Producer

四个环节缺一不可。
仅对 Kafka 内部做 Exactly-Once 是不够的:
  Kafka offset 提交了 ✓
  但数据库写入失败了 ✗
  → Consumer 重启后从下一个 offset 开始
  → 漏处理了这条消息！
```

## 4.3 各语义间的权衡

```
                        可靠性
                          ▲
                          │         ┌──────────────┐
                          │         │ Exactly-Once │  ← 金融级
                          │         │ (高成本)      │
                          │         └──────────────┘
                          │
                          │    ┌──────────────────┐
                          │    │  At-Least-Once   │  ← 业务级
                          │    │  (中等成本)       │
                          │    └──────────────────┘
                          │
                          │ ┌─────────────────────┐
                          │ │   At-Most-Once      │  ← 非关键
                          │ │   (低成本)           │
                          │ └─────────────────────┘
                          │
                          └──────────────────────────────▶ 性能/吞吐
```

# 5 实现 Exactly-Once 的外部系统方案

## 5.1 Kafka + 数据库（使用数据库事务）

```java
@Service
public class ExactlyOnceConsumerService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @KafkaListener(topics = "input", groupId = "eos-group")
    @Transactional // 数据库事务
    public void onMessage(ConsumerRecord<String, String> record) {
        String messageKey = record.key();

        // 1. 幂等性检查（数据库唯一键）
        try {
            jdbcTemplate.update(
                "INSERT INTO processed_messages (msg_id, created_at) VALUES (?, ?)",
                messageKey, Instant.now()
            );
        } catch (DuplicateKeyException e) {
            // 已处理过，跳过
            return;
        }

        // 2. 执行业务逻辑
        // 3. 发送结果（使用 Kafka 事务同步 offset）
        // Spring Kafka + @Transactional → Kafka offset 在事务提交时自动同步
    }
}
```

## 5.2 使用 Flink 实现 EOS

```java
// Flink 的 TwoPhaseCommitSinkFunction 可以实现
// Flink Kafka Consumer → 处理 → Flink Kafka Producer
// Flink 使用 Checkpoint 机制保证精确一次

// 配置:
env.enableCheckpointing(60000);
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
```

# 6 幂等性设计模式

## 6.1 业务幂等常用方法

| 方法 | 描述 | 适用场景 |
|------|------|---------|
| 数据库唯一约束 | 利用主键/唯一键去重 | CRUD 操作 |
| Token/去重表 | 独立去重表存已处理 ID | 通用场景 |
| 版本号 | 更新时检查版本号 | 数据更新 |
| 状态机 | 利用状态转换幂等 | 订单/工单流转 |
| Redis SETNX | 分布式锁 + 缓存 ID | 高并发场景 |

```java
// 状态机幂等示例：
// 订单状态转换: PENDING → PROCESSING → COMPLETED

public void handleOrderEvent(OrderEvent event) {
    Order order = orderRepo.findById(event.getOrderId());

    switch (event.getType()) {
        case PAYMENT_RECEIVED:
            // 仅从 PENDING 转到 PROCESSING，重复调用无效
            if (order.getStatus() == OrderStatus.PENDING) {
                order.setStatus(OrderStatus.PROCESSING);
                orderRepo.save(order);
            }
            break;
        case DELIVERY_COMPLETED:
            // 仅从 PROCESSING 转到 COMPLETED
            if (order.getStatus() == OrderStatus.PROCESSING) {
                order.setStatus(OrderStatus.COMPLETED);
                orderRepo.save(order);
            }
            break;
    }
}
```
