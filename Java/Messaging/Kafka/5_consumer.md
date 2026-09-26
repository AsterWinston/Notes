# 1 Consumer 总览

## 1.1 Consumer 与 Consumer Group 关系

```
┌────────────────────────────────────────────────────────────┐
│                  Consumer Group 模型                        │
│                                                            │
│  Topic "orders" (6 Partitions)                             │
│  ├── P0 ─┐                                                 │
│  ├── P1 ─┤                                                 │
│  ├── P2 ─┼──→ Consumer Group "order-service"               │
│  ├── P3 ─┤     ├── Consumer A: P0, P1                      │
│  ├── P4 ─┤     ├── Consumer B: P2, P3                      │
│  └── P5 ─┘     └── Consumer C: P4, P5                      │
│                                                            │
│  同一个 Group: 消息被均分消费（队列模式）                      │
│                                                            │
│  ─────────────────────────────────────────────────────────→│
│                                                            │
│  所有分区 ──→ Consumer Group "analytics"                   │
│               └── Consumer D: P0~P5（独立消费所有分区）       │
│                                                            │
│  不同 Group: 各自独立消费（发布/订阅模式）                     │
└────────────────────────────────────────────────────────────┘
```

## 1.2 Consumer 核心参数速查

```java
Properties props = new Properties();

// === 必选参数 ===
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.deserializer", StringDeserializer.class);
props.put("value.deserializer", StringDeserializer.class);
props.put("group.id", "my-consumer-group");

// === Offset 管理 ===
props.put("enable.auto.commit", false);              // 关闭自动提交
props.put("auto.offset.reset", "earliest");          // earliest/latest/none
props.put("auto.commit.interval.ms", 5000);

// === 拉取控制 ===
props.put("fetch.min.bytes", 1);                     // 最少拉取字节
props.put("fetch.max.wait.ms", 500);                 // 最大等待时间
props.put("fetch.max.bytes", 52428800);              // 单次拉取最大字节(50MB)
props.put("max.partition.fetch.bytes", 1048576);     // 单分区拉取最大字节(1MB)
props.put("max.poll.records", 500);                  // 单次 poll 最大记录数

// === Rebalance 与心跳 ===
props.put("session.timeout.ms", 45000);              // Session 超时
props.put("heartbeat.interval.ms", 3000);            // 心跳间隔
props.put("max.poll.interval.ms", 300000);           // poll 间隔上限

// === 分配策略 ===
props.put("partition.assignment.strategy",
        Collections.singletonList(CooperativeStickyAssignor.class));

// === 隔离级别（事务消息） ===
props.put("isolation.level", "read_committed");      // read_uncommitted/read_committed
```

# 2 消费者组协调协议

## 2.1 Consumer Group 状态机

```
                    ┌──────────┐
          Consumer  │  Empty   │  所有成员离开
          加入      └────┬─────┘
                    │        ▲
                    ▼        │  最后成员离开
              ┌──────────┐   │
              │ Preparing│   │
              │ Rebalance│───┘
              └────┬─────┘
                   │ Rebalance 完成
                   ▼
              ┌──────────┐
              │  Stable  │ ← 正常消费状态
              └────┬─────┘
                   │ 成员加入/离开/超时
                   ▼
              ┌──────────┐
              │ Preparing│
              │ Rebalance│ ──→ ...
              └──────────┘
```

### 2.1.1 各状态说明

| 状态 | 含义 |
|------|------|
| Empty | Group 无活跃成员，无 offset 提交或已过期 |
| PreparingRebalance | 有成员加入或离开，等待所有成员回复 JoinGroup |
| CompletingAwaitingAssignment | Leader 已分配，等待所有成员 SyncGroup |
| Stable | Group 稳定，Consumer 正常消费 |
| Dead | Group 无活跃成员且 metadata 清理 |

## 2.2 JoinGroup 协议（Eager 模式）

```
Coordinator                    Consumer A (Leader)        Consumer B
    │                              │                        │
    │ ◄─ ① Heartbeat 超时          │                        │
    │                              │                        │
    │ ──② JoinGroup Request───────▶│                        │
    │                              │                        │
    │ ◄──③ JoinGroup Request────────────────────────────────│
    │                              │                        │
    │ ──④ JoinGroup Response──────▶│                        │
    │    (member list, leader=A)   │                        │
    │                              │                        │
    │ ──④ JoinGroup Response───────────────────────────────▶│
    │                              │                        │
    │                              │ ⑤ Leader 执行分配策略    │
    │                              │   分配: A→[P0,P1]       │
    │                              │         B→[P2,P3]       │
    │                              │                        │
    │ ◄──⑥ SyncGroup Request──────│                        │
    │    (partition assignment)    │                        │
    │                              │                        │
    │ ◄──⑥ SyncGroup Request────────────────────────────────│
    │                              │                        │
    │ ──⑦ SyncGroup Response─────▶│                        │
    │    A 的分配: [P0, P1]        │                        │
    │                              │                        │
    │ ──⑦ SyncGroup Response───────────────────────────────▶│
    │    B 的分配: [P2, P3]        │                        │
    │                              │                        │
    │ ★ Rebalance 完成，开始消费     │                        │
```

## 2.3 Cooperative（增量式）Rebalance 协议（Kafka 2.4+）

```
与 Eager 的关键区别：

Eager Revocation:
  JoinGroup Request 携带 "所有当前拥有的分区"
  → Coordinator 要求全部释放 → 重新完全分配 → Stop-The-World

Cooperative Revocation:
  Consumer 分批次逐步释放分区:
  ① 第一轮: Consumer 释放不需要的分区，Consumer A 继续消费 P0, P1
  ② 第二轮: 将释放的分区分配给新加入的 Consumer C
  ③ Consumer C 开始消费 → Rebalance 完成后恢复所有消费

好处: 减少"暂停消费"的时间窗口
```

### 2.3.1 开启 Cooperative Rebalance

```java
// 方式一：单个策略
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
        CooperativeStickyAssignor.class.getName());

// 方式二：多个策略（逐个尝试，选择第一个支持的）
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
        "org.apache.kafka.clients.consumer.CooperativeStickyAssignor," +
        "org.apache.kafka.clients.consumer.RangeAssignor");
```

# 3 分区分配策略详解

## 3.1 RangeAssignor（范围分配）

默认策略，按 Topic 逐一分配。对每个 Topic，分区连续分配给各个 Consumer。

```
Topic "A": 7 Partitions, Group: 3 Consumers

排序后分配:
Consumer1: P0, P1, P2  (7/3=2, 余1 → 多拿1个)
Consumer2: P3, P4
Consumer3: P5, P6

缺点: 当订阅多个 Topic 时，前面的 Consumer 容易负载不均
```

```java
// RangeAssignor.java（核心逻辑）
Map<String, List<TopicPartition>> assign(
        Map<String, Integer> partitionsPerTopic,  // {"A": 7, "B": 5}
        Map<String, Subscription> subscriptions) { // {C1, C2, C3}

    for (Map.Entry<String, Integer> topicEntry : partitionsPerTopic.entrySet()) {
        String topic = topicEntry.getKey();
        int numPartitions = topicEntry.getValue();
        List<String> consumers = sorted(subscriptions.keySet());

        int numPartitionsPerConsumer = numPartitions / consumers.size();
        int remainder = numPartitions % consumers.size();

        int start = 0;
        for (int i = 0; i < consumers.size(); i++) {
            int end = start + numPartitionsPerConsumer + (i < remainder ? 1 : 0);
            assignment.get(consumers.get(i)).addAll(
                partitions.subList(start, end)); // 连续分配
            start = end;
        }
    }
}
```

## 3.2 RoundRobinAssignor（轮询分配）

将所有 Topic 的所有 Partition 排序后在 Consumer 间轮询。

```
Topic "A": P0, P1, P2
Topic "B": P0, P1

所有分区排序: A-P0, A-P1, A-P2, B-P0, B-P1

Consumer1: A-P0, A-P2, B-P1
Consumer2: A-P1, B-P0

优点: 更均匀
缺点: Rebalance 时重新全部分配，分区迁移量大
```

## 3.3 StickyAssignor（粘性分配）

```
Rebalance 前后对比:

Rebalance 前:
C1: P0, P1, P2, P3
C2: P4, P5, P6

C3 加入后 → Sticky 分配:
C1: P0, P1            ← 保留了 P0, P1 (Sticky!)
C2: P5, P6            ← 保留了 P5, P6
C3: P2, P3, P4        ← 新分配的

对比 Range/RoundRobin 会全部打乱重分配:
C1: P0, P1, P2
C2: P3, P4
C3: P5, P6            ← 全部重新分配，迁移量大
```

## 3.4 CooperativeStickyAssignor（增量粘性分配）

基于 Sticky 的策略 + Cooperative 协议。分配时只迁移所需的少量分区，且分批完成。

```
迁移过程（分批次）:

第 1 批:
  → 释放 P2, P3 从 C1
  → C1 继续消费 P0, P1

第 2 批:
  → 将 P2, P3 分配给 C3
  → C3 开始消费

结果: C1 在整个过程中只短暂停止 P2, P3，P0, P1 持续消费
```

# 4 Offset 管理机制

## 4.1 Offset 提交的三种方式

### 4.1.1 自动提交

```java
props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000);

// 问题: poll() 内部每 5s 自动提交一次
// 如果在提交后、处理前发生崩溃 → 消息丢失
```

### 4.1.2 同步提交

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    processRecords(records);
    try {
        consumer.commitSync(); // 阻塞直到提交成功或报错
    } catch (CommitFailedException e) {
        // 处理提交失败（如 Rebalance 时）
    }
}
```

### 4.1.3 异步提交

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    processRecords(records);
    consumer.commitAsync((offsets, exception) -> {
        if (exception != null) {
            log.error("Offset commit failed: {}", offsets, exception);
        }
    });
    // 无阻塞，吞吐高，但失败可能被后续成功覆盖
}
```

### 4.1.4 混合提交（推荐）

```java
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
        processRecords(records);
        consumer.commitAsync(); // 正常情况异步提交

        // 每 1000 条同步提交一次（确保进度落盘）
        if (count++ % 1000 == 0) {
            consumer.commitSync();
        }
    }
} finally {
    // Consumer 关闭前最后一次同步提交，确保关闭前不丢进度
    try {
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```

### 4.1.5 精确 Offset 提交

```java
// 逐条处理 + 逐条提交（保证每一条都提交）
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
for (ConsumerRecord<String, String> record : records) {
    processRecord(record);
    offsets.put(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1) // offset + 1 是下一条的起始位置
    );
    consumer.commitSync(offsets);
}
```

## 4.2 Offset 存储位置

### 4.2.1 演进历史

| Kafka 版本 | Offset 存储位置 |
|-----------|----------------|
| < 0.9 | ZooKeeper (`/consumers/{group}/offsets/{topic}/{partition}`) |
| >= 0.9 | 内部 Topic `__consumer_offsets`（默认 50 分区） |

### 4.2.2 `__consumer_offsets` 内部结构

```
Key: GroupMetadataKey
  - groupId: String
  - topic: String
  - partition: int

Value: OffsetAndMetadata
  - offset: long
  - metadata: String
  - commitTimestamp: long
  - leaderEpoch: int (Kafka 2.1+)
```

## 4.3 Offset 重置策略

```java
// 当没有已提交 offset 时，使用此策略
props.put("auto.offset.reset", "earliest"); // 从最早的 offset 开始

// 三种可选值:
//   earliest → 从最早的消息开始消费
//   latest   → 从最新的消息开始消费（默认）
//   none     → 无已提交 offset 时抛出异常
```

# 5 Consumer 心跳与会话管理

## 5.1 心跳线程

Kafka Consumer 有两类关键线程：

```
KafkaConsumer 线程模型
│
├── 用户线程（User Thread）
│   └── consumer.poll() → 拉取 + 处理
│
├── 心跳线程（Heartbeat Thread）
│   ├── 向 GroupCoordinator 发送 Heartbeat
│   ├── 心跳间隔: heartbeat.interval.ms（默认 3s）
│   ├── Session 超时: session.timeout.ms（默认 45s）
│   └── 若 session.timeout.ms 内未收到心跳 → 触发 Rebalance
│
└── 后台线程
    └── Coordinator 发现、Metadata 刷新
```

### 5.1.1 关键超时时间线

```
时间轴 ────────────────────────────────────────────────────▶

│◀──── heartbeat.interval.ms ────▶│  ← 每隔 3s 发送心跳
│                                  │
│◀─────────── session.timeout.ms ────────────▶│ ← 45s 内未收到心跳 → Rebalance
│                                              │
│◀── max.poll.interval.ms ──────────────────────────────────▶│
│                    ← poll 间隔超过此值 → Consumer 被认为"失联" → Rebalance
```

## 5.2 max.poll.interval.ms 的陷阱

```java
// 危险示例: 消息处理时间 > max.poll.interval.ms
props.put("max.poll.interval.ms", 300000); // 5 分钟

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, String> record : records) {
        // 单条消息处理需要 10 分钟（超过了 5 分钟限制！）
        heavyProcess(record);
    }
    // Consumer 无法按时 poll → Coordinator 认为失联 → 触发 Rebalance
}
```

**解决方案：**

```java
// 方案 1: 增大 max.poll.interval.ms
props.put("max.poll.interval.ms", 600000); // 10 分钟

// 方案 2: 减小 max.poll.records
props.put("max.poll.records", 10); // 每次只拉 10 条

// 方案 3: poll 后异步处理（暂停分区后再恢复）
List<TopicPartition> partitions = records.partitions();
consumer.pause(partitions); // 暂停拉取新消息
asyncProcess(records, () -> {
    consumer.resume(partitions); // 处理完后恢复
});
```

# 6 Consumer 拉取流程

## 6.1 Fetcher 拉取模型

```
KafkaConsumer.poll(timeout)
│
├── ① ensureCoordinatorReady() → 确保 Coordinator 连接就绪
├── ② 如果处于 Rebalance → 参与 Rebalance
├── ③ updateAssignmentMetadataIfNeeded() → 更新分区分配
│
└── ④ pollForFetches(timer)
    │
    ├── Fetcher.sendFetches()
    │   ├── 收集所有已分配分区的 fetch 请求
    │   ├── 按 Broker 聚合请求
    │   └── 发送 FetchRequest
    │
    └── Fetcher.collectFetch()
        ├── 从 completedFetches 中收集完成的响应
        ├── 反序列化消息
        ├── 放入 ConsumerRecords（用户在 poll() 中获取）
        └── 更新 consumed offset 到 SubscriptionState
```

## 6.2 Fetch 参数协同机制

```
fetch.min.bytes = 1                 ← 最少字节数。达不到则等待
fetch.max.wait.ms = 500             ← 等待多久（ms）
fetch.max.bytes = 52428800          ← 总拉取上限(50MB)
max.partition.fetch.bytes = 1048576 ← 单分区上限(1MB)
max.poll.records = 500              ← 单次 poll 返回上限(条)

Broker 端行为:
  - 如果消息字节数 >= fetch.min.bytes → 立刻返回
  - 否则等待最多 fetch.max.wait.ms 如果有新消息累积 → 返回
  - 等待超时仍不足 → 返回空或已有消息

Consumer 端行为:
  - poll(timeout): timeout 和 fetch.max.wait.ms 的较大者控制总等待
```

# 7 再均衡监听器（Rebalance Listener）

## 7.1 ConsumerRebalanceListener 接口

```java
public interface ConsumerRebalanceListener {
    // Rebalance 开始前调用（当前 Consumer 即将失去分区所有权时）
    void onPartitionsRevoked(Collection<TopicPartition> partitions);

    // Rebalance 完成后调用（新分配的分区就绪时）
    void onPartitionsAssigned(Collection<TopicPartition> partitions);

    // Cooperative: 撤销前调用（允许延迟撤销）
    default void onPartitionsLost(Collection<TopicPartition> partitions) {}
}
```

## 7.2 典型使用场景

```java
public class OffsetBackedConsumer {
    private Map<TopicPartition, OffsetAndMetadata> partitionOffsets = new ConcurrentHashMap<>();

    public void consume() {
        consumer.subscribe(topics, new ConsumerRebalanceListener() {
            @Override
            public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
                // Rebalance 前：最后一次提交 offset
                // 确保"已处理完但未提交"的消息不会丢失
                Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
                for (TopicPartition tp : partitions) {
                    OffsetAndMetadata offset = partitionOffsets.get(tp);
                    if (offset != null) {
                        offsets.put(tp, offset);
                    }
                }
                consumer.commitSync(offsets);
                log.info("Rebalance: committed offsets for {} partitions", partitions.size());
            }

            @Override
            public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
                // Rebalance 后：获取新分配分区的最新 offset，从断点恢复
                for (TopicPartition tp : partitions) {
                    OffsetAndMetadata committed = consumer.committed(tp);
                    if (committed != null) {
                        consumer.seek(tp, committed.offset());
                        log.info("Rebalance: seeking {} to offset {}", tp, committed.offset());
                    }
                }
            }
        });
    }
}
```

## 7.3 CooperativeRebalanceListener（Kafka 2.4+）

```java
// 对于 Cooperative Rebalance，推荐使用 CooperativeStickyAssignor
// 配合使用更精确的回调
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Eager Rebalance: 全部释放前
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Rebalance 后，包括增量和全量分配
    }

    @Override
    public void onPartitionsLost(Collection<TopicPartition> partitions) {
        // Cooperative: C 崩溃/离开时强制释放
        // 不需要手动 commit offset
        log.warn("Lost partitions: {}", partitions);
    }
});
```

# 8 消费语义保障

## 8.1 至少一次（At-Least-Once）

```java
// 先处理再提交 → 至少一次
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, String> record : records) {
        process(record); // 处理可能成功，随后提交失败
    }
    consumer.commitSync();
    // 问题: 处理成功但提交失败 → 重启后重复消费
}
```

## 8.2 最多一次（At-Most-Once）

```java
// 先提交再处理 → 最多一次
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    consumer.commitSync(); // 先提交 offset
    for (ConsumerRecord<String, String> record : records) {
        process(record); // 处理可能失败 → 消息丢失
    }
}
```

## 8.3 精确一次（Exactly-Once）——消费者侧

```java
// 将业务处理 + offset 提交封装在同一个事务中
// 例如: 消费 Kafka 消息 → 写数据库 → 提交 Kafka offset
// 这三个操作必须在同一个事务中

// 伪代码（使用 Spring Kafka）:
@KafkaListener(topics = "orders")
@Transactional // 数据库事务
public void onMessage(ConsumerRecord<String, String> record) {
    // 1. 处理消息并写入数据库
    orderRepository.save(convertToOrder(record));

    // 2. Kafka offset 在事务提交时自动提交
    // Spring Kafka 的 KafkaMessageListenerContainer
    // 通过 KafkaTransactionManager 协调 Kafka 事务和 DB 事务
}
```

# 9 Consumer 性能调优

## 9.1 参数调优指南

```java
// === 高吞吐 Consumer 配置模板 ===
Properties props = new Properties();
props.put("enable.auto.commit", false);
props.put("fetch.min.bytes", 1048576);               // 1MB（批量拉取）
props.put("fetch.max.wait.ms", 500);
props.put("max.partition.fetch.bytes", 10485760);    // 10MB
props.put("max.poll.records", 1000);
props.put("receive.buffer.bytes", 65536);            // TCP 接收缓冲
props.put("max.poll.interval.ms", 600000);           // 处理时间长的话增大
props.put("session.timeout.ms", 60000);              // 适当增大避免误触发 Rebalance
props.put("heartbeat.interval.ms", 10000);

// === 使用分区分配策略 ===
props.put("partition.assignment.strategy",
        Collections.singletonList(CooperativeStickyAssignor.class));
```

## 9.2 多线程消费模型

### 9.2.1 模型一：每个 Consumer 一个线程（简单）

```java
// 每个线程维护一个 KafkaConsumer 实例
// 优点: 简单，线程安全自动保证
// 缺点: 线程数固定，扩展性受限

ExecutorService executor = Executors.newFixedThreadPool(numConsumers);
for (int i = 0; i < numConsumers; i++) {
    executor.submit(() -> {
        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(topics);
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
                for (ConsumerRecord<String, String> record : records) {
                    process(record);
                }
            }
        }
    });
}
```

### 9.2.2 模型二：消费与处理分离（推荐）

```java
// Consumer 线程: 仅负责 poll
// Worker 线程池: 负责消息处理

int numWorkers = Runtime.getRuntime().availableProcessors();
ExecutorService workers = Executors.newFixedThreadPool(numWorkers);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    List<TopicPartition> pausedPartitions = records.partitions();

    // 暂停拉取（防止处理期间再拉取导致 offset 混乱）
    consumer.pause(pausedPartitions);

    // 异步处理
    CompletableFuture<Void> processingFuture = CompletableFuture.runAsync(() -> {
        for (ConsumerRecord<String, String> record : records) {
            process(record);
        }
    }, workers).thenRun(() -> {
        consumer.resume(pausedPartitions); // 处理完后恢复
        consumer.commitAsync();
    });
}
```

### 9.2.3 模型三：按分区并行

```java
// 每个 Partition 独占一个线程处理
// 保证分区内消息有序处理

Map<TopicPartition, BlockingQueue<ConsumerRecord<String, String>>> queues = new HashMap<>();

// 调度线程: poll → 按分区分发到对应队列
// 工作线程: 每个分区一个线程，从队列取消息处理
```

## 9.3 常见 Consumer 异常

| 异常 | 原因 | 解决 |
|------|------|------|
| `CommitFailedException` | Rebalance 期间提交 | 增大 `max.poll.interval.ms` / 在 Rebalance Listener 中提交 |
| `WakeupException` | `consumer.wakeup()` 被调用 | 主动结束消费循环 |
| `OffsetOutOfRangeException` | Offset 不存在（已删除） | 重置 offset 到 earliest/latest |
| `AuthenticationException` | SASL 认证失败 | 检查 security 配置 |
| `TopicAuthorizationException` | 无权限 | 申请 ACL 权限 |
| `RecordDeserializationException` | 反序列化失败 | `ErrorHandlingDeserializer` / 跳过 |
| `KafkaException` (not coordinator) | Coordinator 不可达 | 检查 Coordinator Broker 状态 |
