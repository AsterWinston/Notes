# 1 Producer 总览

## 1.1 Producer 在 Kafka 生态中的角色

```
┌──────────────────────────────────────────────────────┐
│                   Producer 核心流程                   │
│                                                      │
│  send()                                              │
│    │                                                 │
│    ▼                                                 │
│  ┌────────────────┐                                  │
│  │  Interceptors  │ ← ProducerInterceptor 拦截链      │
│  └───────┬────────┘                                  │
│          ▼                                           │
│  ┌────────────────┐                                  │
│  │   Serializer   │ ← Key/Value 序列化器              │
│  └───────┬────────┘                                  │
│          ▼                                           │
│  ┌────────────────┐                                  │
│  │  Partitioner   │ ← 选择目标分区                     │
│  └───────┬────────┘                                  │
│          ▼                                           │
│  ┌─────────────────────────────────┐                 │
│  │       RecordAccumulator         │ ← 按分区缓冲消息  │
│  │  ┌──────────┐  ┌──────────┐     │                 │
│  │  │ Batch P0 │  │ Batch P1 │ ... │                 │
│  │  └──────────┘  └──────────┘     │                 │
│  └───────────────┬─────────────────┘                 │
│                  ▼                                   │
│  ┌─────────────────────────────────┐                 │
│  │          Sender Thread          │ ← 异步发送线程   │
│  │    (NetworkClient + Selector)   │                 │
│  └───────────────┬─────────────────┘                 │
│                  ▼                                   │
│              Kafka Broker                            │
└──────────────────────────────────────────────────────┘
```

## 1.2 Producer 核心参数速查

```java
Properties props = new Properties();

// === 必选参数 ===
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// === 可靠性参数 ===
props.put("acks", "all");                          // 0 / 1 / all(-1)
props.put("enable.idempotence", true);              // 幂等生产者
props.put("max.in.flight.requests.per.connection", 5); // 幂等时 ≤5
props.put("retries", Integer.MAX_VALUE);            // 重试次数
props.put("delivery.timeout.ms", 120000);           // 投递超时

// === 性能参数 ===
props.put("batch.size", 16384);                     // 批次大小 (bytes)
props.put("linger.ms", 0);                          // 批次等待时间
props.put("buffer.memory", 33554432);               // RecordAccumulator 缓冲区
props.put("compression.type", "snappy");            // 压缩类型: none/gzip/snappy/lz4/zstd
props.put("max.request.size", 1048576);             // 单次请求最大大小

// === 幂等 & 事务 ===
props.put("transactional.id", "my-tx-id");          // 事务 ID（开启事务时设置）
```

# 2 消息发送流程详解

## 2.1 send() 方法两种形式

```java
// 形式一：即发即忘（Fire-and-Forget）
producer.send(record);

// 形式二：同步发送
Future<RecordMetadata> future = producer.send(record);
RecordMetadata metadata = future.get(); // 阻塞等待结果

// 形式三：异步回调
producer.send(record, new Callback() {
    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        if (exception != null) {
            // 发送失败处理
        } else {
            // 成功处理
        }
    }
});
```

## 2.2 完整发送链路

```java
// KafkaProducer.doSend() 的核心逻辑（简化版）
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    // 1. 拦截器前置处理
    ProducerRecord<K, V> interceptedRecord = record;
    for (ProducerInterceptor<K, V> interceptor : interceptors) {
        interceptedRecord = interceptor.onSend(interceptedRecord);
    }

    // 2. 等待获取集群元数据（Topic 的分区信息等）
    ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(
        record.topic(), record.partition(), maxBlockTimeMs);
    long remainingWaitMs = clusterAndWaitTime.remainingWaitMs;
    Cluster cluster = clusterAndWaitTime.cluster;

    // 3. 序列化 Key 和 Value
    byte[] serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
    byte[] serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());

    // 4. 计算目标分区
    int partition = partition(record, serializedKey, serializedValue, cluster);

    // 5. 计算总大小并校验
    int serializedSize = Records.LOG_OVERHEAD + Record.recordSize(serializedKey, serializedValue);
    ensureValidRecordSize(serializedSize);

    // 6. 追加到 RecordAccumulator
    RecordAccumulator.RecordAppendResult result = accumulator.append(
        tp, timestamp, serializedKey, serializedValue, headers,
        interceptCallback, remainingWaitMs, now);

    // 7. 如果 Batch 满了或创建了新 Batch，唤醒 Sender
    if (result.batchIsFull || result.newBatchCreated) {
        sender.wakeup();
    }

    return result.future;
}
```

# 3 分区机制（Partitioner）

## 3.1 默认分区策略

`DefaultPartitioner` 的分配逻辑：

```java
// org.apache.kafka.clients.producer.internals.DefaultPartitioner
public int partition(String topic, Object key, byte[] keyBytes,
                     Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();

    if (keyBytes == null) {
        // 无 Key：Sticky 分区（2.4+）
        // 将消息随机选一个分区后粘住，下一批消息继续发往同一分区
        // 直到 Batch 满了或 linger.ms 时间到再切换
        Integer stickyPartition = stickyPartitionCache.partition(topic, cluster);
        return stickyPartition;
    } else {
        // 有 Key：hash(key) % numPartitions
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}
```

### 3.1.1 Sticky 分区（粘性分区）带来的改进

**旧版行为（2.4 之前）**：无 Key 时每条消息随机轮询分区
- 导致大量小 Batch，网络请求频繁

**新版行为（2.4+）**：粘性分区
- 将消息"粘"在一个分区上，直到 Batch 被发走
- 减少请求数量，提升吞吐

```
旧版轮询：
Msg1→P0  Msg2→P1  Msg3→P2  （3 个网络请求）

新版粘性：
Msg1→P0, Msg2→P0, Msg3→P0  → 同一 Batch → 1 个网络请求
Msg4→P1, Msg5→P1             → 同一 Batch → 1 个网络请求
```

## 3.2 自定义分区器

```java
public class TenantPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        // 从 Key 中提取租户 ID
        String keyStr = (String) key; // key = "tenant_001:order_123"
        String tenantId = keyStr.split(":")[0];
        int tenantHash = Math.abs(tenantId.hashCode());
        int numPartitions = cluster.partitionCountForTopic(topic);
        return tenantHash % numPartitions;
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}

// 使用时指定
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG,
        "com.example.TenantPartitioner");
```

# 4 序列化器（Serializer）

## 4.1 内置序列化器

| 序列化器 | 对应 Java 类型 |
|---------|---------------|
| `StringSerializer` | `String` |
| `IntegerSerializer` | `Integer` |
| `LongSerializer` | `Long` |
| `DoubleSerializer` | `Double` |
| `ByteArraySerializer` | `byte[]` |
| `ByteBufferSerializer` | `java.nio.ByteBuffer` |
| `BytesSerializer` | `org.apache.kafka.common.utils.Bytes` |
| `UUIDSerializer` | `java.util.UUID` |
| `VoidSerializer` | `Void` (null) |

## 4.2 Avro 序列化（Schema Registry 集成）

```xml
<dependency>
    <groupId>io.confluent</groupId>
    <artifactId>kafka-avro-serializer</artifactId>
    <version>7.5.0</version>
</dependency>
```

```java
// Avro 序列化
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
        io.confluent.kafka.serializers.KafkaAvroSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
        io.confluent.kafka.serializers.KafkaAvroSerializer.class);
props.put("schema.registry.url", "http://localhost:8081");
```

## 4.3 Protobuf 序列化（Spring Kafka 推荐）

```java
// 自定义 Protobuf 序列化器
public class ProtobufSerializer<T extends Message>
        implements Serializer<T> {

    @Override
    public byte[] serialize(String topic, T data) {
        return data != null ? data.toByteArray() : null;
    }
}
```

# 5 拦截器（Interceptor）

## 5.1 ProducerInterceptor 接口

```java
public interface ProducerInterceptor<K, V> {
    // 发送前拦截（onSend 是 send 的 on）
    ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);

    // 发送后回调（ACK 返回后）
    void onAcknowledgement(RecordMetadata metadata, Exception exception);

    // 生命周期
    void configure(Map<String, ?> configs);
    void close();
}
```

## 5.2 自定义拦截器示例：发送计时 + 审计

```java
public class AuditInterceptor implements ProducerInterceptor<String, String> {
    private long sentCount;
    private long errorCount;

    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        // 添加审计 header
        record.headers().add("trace-id", UUID.randomUUID().toString().getBytes(UTF_8));
        record.headers().add("timestamp", String.valueOf(System.currentTimeMillis()).getBytes(UTF_8));
        sentCount++;
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (exception != null) {
            errorCount++;
        }
    }

    @Override
    public void close() {
        System.out.printf("Sent: %d, Errors: %d%n", sentCount, errorCount);
    }

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

# 6 RecordAccumulator 缓冲区机制

## 6.1 数据结构

```
RecordAccumulator
│
├── ConcurrentMap<TopicPartition, Deque<ProducerBatch>>
│   │
│   ├── TopicPartition("orders", 0) → Deque
│   │   ├── ProducerBatch (未满, 正在追加)
│   │   └── ProducerBatch (已满, 等待发送)
│   │
│   ├── TopicPartition("orders", 1) → Deque
│   │   └── ProducerBatch (未满)
│   │
│   └── ...
│
└── BufferPool (free.availableMemory)
    ├── 池化 16KB 的 ByteBuffer 列表
    └── 当池耗尽时从 buffer.memory 分配
```

## 6.2 关键参数协同关系

```
消息到达速度  vs  发送速度
     │              │
     ▼              ▼
buffer.memory   batch.size + linger.ms
(总内存上限)    (触发发送的条件)

当 buffer.memory 满时 → send() 阻塞
max.block.ms 超时 → 抛出 TimeoutException
```

### 6.2.1 参数调节策略

| 场景 | batch.size | linger.ms | buffer.memory |
|------|-----------|-----------|--------------|
| 高吞吐低延迟 | 增大 | 增大（10~100ms） | 增大 |
| 低延迟优先 | 减小 | 0 | 适中 |
| 消息大小不均匀 | 基于消息大小估算 | 较小 | 足够大 |
| 网络带宽有限 | 适中 | 适中 | 适中 |

# 7 Sender 线程与网络模型

## 7.1 Sender 线程工作流程

```java
// Sender.run() 核心逻辑（简化）
void run() {
    while (running) {
        // 1. 从 RecordAccumulator 获取准备就绪的 Batch
        long pollTimeout = sendProducerData(now);
        //    ├── 根据 Batch 信息确定哪个 Broker 的哪个 Partition
        //    ├── 检查 leader 是否可用
        //    ├── 将 Batch 转换为 ProduceRequest
        //    └── 添加到 NetworkClient 的 InFlightRequests

        // 2. 执行实际的网络 I/O（由 NetworkClient 驱动）
        client.poll(pollTimeout, now);
        //    ├── 发送 ProduceRequest
        //    ├── 接收 ProduceResponse
        //    └── 触发 Callback
    }
}
```

## 7.2 NetworkClient 与 Selector

Kafka 的网络层基于 Java NIO Selector 实现：

```
Sender Thread
│
├── NetworkClient
│   ├── Selector（管理所有到 Broker 的 TCP 连接）
│   │   ├── KafkaChannel (Broker 1)
│   │   ├── KafkaChannel (Broker 2)
│   │   └── KafkaChannel (Broker 3)
│   │
│   ├── InFlightRequests (每个 Broker 一个队列)
│   │   └── max.in.flight.requests.per.connection 控制
│   │
│   └── Metadata（缓存集群元数据）
│       ├── Cluster (所有 Topic 的分区信息)
│       └── 定期刷新 (metadata.max.age.ms)
│
└── 发送缓冲 / 接收缓冲
```

# 8 ACK 机制与消息可靠性

## 8.1 ACK 三种级别

```
acks=0：
  Producer 不等待任何确认
  ┌──────┐
  │ Prod │────msg───▶ Broker（写 page cache）
  └──────┘
  立即可发送下一条

acks=1：
  Producer 等待 Leader 确认
  ┌──────┐     ┌──────────┐
  │ Prod │────▶│ Leader   │（写 page cache 后响应）
  └──────┘     └──────────┘
  等待 Leader ACK       │
                        ▼
                  ┌──────────┐
                  │ Follower │（异步复制）
                  └──────────┘

acks=all (或 acks=-1)：
  Producer 等待所有 ISR 确认
  ┌──────┐     ┌──────────┐
  │ Prod │────▶│ Leader   │（写 page cache）
  └──────┘     └────┬─────┘
                     │ ① Leader 写入本地
                     │ ② Follower 拉取并确认
                     │ ③ Leader 收到所有 ISR 确认
                     ▼
              Leader 响应 ACK 给 Producer
```

### 8.1.1 选择指南

| acks | 可靠性 | 延迟 | 适用场景 |
|------|--------|------|----------|
| 0 | 可能丢失 | 极低 | 日志收集、丢失可接受 |
| 1 | 可能丢失（Leader 宕机） | 低 | 一般场景，允许少量丢失 |
| all | 不丢失（配合 min.insync.replicas） | 较高 | 金融、订单、支付 |

## 8.2 min.insync.replicas 的作用

`min.insync.replicas` 指定了在 acks=all 时，最少需要多少 ISR 副本确认写入。

```
假设: replication.factor=3, min.insync.replicas=2

正常状态 (ISR=3):
  Leader + 2 Followers → 写入成功 ✓

一个 Follower 故障 (ISR=2):
  Leader + 1 Follower → 写入成功 ✓（满足 min.insync.replicas=2）

两个 Follower 故障 (ISR=1):
  Leader 仅自己 → 写入失败 ✗ → NotEnoughReplicasException
```

推荐的配置组合：

```
replication.factor = 3
min.insync.replicas = 2
acks = all
→ 可容忍 1 个 Broker 故障而不会丢失消息或拒绝写入
```

# 9 消息重试机制

## 9.1 重试参数

```java
props.put("retries", Integer.MAX_VALUE);          // 最大重试次数
props.put("retry.backoff.ms", 100);               // 重试间隔
props.put("delivery.timeout.ms", 120000);         // 投放超时（含重试总时间）
props.put("request.timeout.ms", 30000);           // 单次请求超时
```

### 9.1.1 delivery.timeout.ms 与旧版参数的关系（Kafka 2.1+）

```
旧版: retries + max.in.flight.requests.per.connection
新版: delivery.timeout.ms = 120000

delivery.timeout.ms 是消息从 send() 到 ACK/失败的总时间上限
内部自动拆分：
  send 超时 = delivery.timeout.ms - linger.ms - request.timeout.ms
```

## 9.2 重试可能导致的消息乱序

```
max.in.flight.requests.per.connection = 5

时间线 ──────────────────────────────▶

Batch1 (offset=0..9)  → 发送 → ✗ 失败 → 重试
Batch2 (offset=10..19) → 发送 → ✓ 成功（先写入）
Batch1 重试成功（后写入）

结果: Partition 中消息顺序: [batch2, batch1] ← 乱序！
```

### 9.2.1 解决方案

**方案一：幂等生产者（推荐）**

```java
props.put("enable.idempotence", true);
// 内部自动保证:
// 1. max.in.flight.requests.per.connection ≤ 5
// 2. retries = Integer.MAX_VALUE
// 3. acks = all
// → 重试不引起乱序（通过 Producer ID + Sequence Number 去重）
```

**方案二：同步发送**

```java
props.put("max.in.flight.requests.per.connection", 1);
// 同时只允许一个未确认请求 → 天然有序 → 但严重影响吞吐
```

# 10 幂等生产者（Idempotent Producer）

## 10.1 核心原理

幂等生产者通过为每条消息分配唯一的 **Producer ID (PID) + Sequence Number** 来确保同一批次不会重复写入：

```
Producer 初始化:
  → Broker 返回一个全局唯一的 Producer ID (PID)

每次发送消息:
  → 附带 PID + SeqNum (0, 1, 2, ...)
  → Broker 检查 (PID, Partition) 的 SeqNum:
      - 如果 SeqNum == lastSeqNum + 1 → 正常追加
      - 如果 SeqNum <= lastSeqNum     → 重复消息，丢弃
      - 如果 SeqNum > lastSeqNum + 1  → 有消息丢失，报错 OutOfOrderSequenceException
```

## 10.2 开启幂等

```java
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// Kafka 3.0+ 默认值就是 true
```

开启后自动设置：
- `acks=all`
- `retries=Integer.MAX_VALUE`
- `max.in.flight.requests.per.connection ≤ 5`

## 10.3 幂等的作用域

- **仅对单个 Producer Session 有效**（Producer 重启后 PID 变化）
- **仅对单个 Partition 有效**
- 不能跨 Topic，不能跨 Partition

## 10.4 幂等实现源码关键点

```java
// ProducerStateManager（Broker 端）
public boolean validateSequence(int partition, long seqNum, long lastSeqNum) {
    // 检查序列号是否为连续递增
    if (seqNum <= lastSeqNum) {
        // 重复或过期 → 丢弃（幂等保证）
        return false;
    }
    if (seqNum > lastSeqNum + 1) {
        // 序列号跳号 → 消息丢失，抛出 OutOfOrderSequenceException
        throw new OutOfOrderSequenceException(
            "Out of order sequence number for producer at offset in partition");
    }
    return true;
}
```

# 11 Producer 事务

## 11.1 事务 vs 幂等

| 特性 | 幂等 | 事务 |
|------|------|------|
| 保证范围 | 单个 Partition | 跨多个 Partition / Topic |
| Producer 重启后 | PID 变，不再保证 | Transactional ID 不变，保证 |
| 使用复杂度 | 只需一个参数 | 需要 Transactional ID + API |
| 典型场景 | 无 Key 的简单消息 | 跨 Topic 原子写入 |

```java
// 事务 Producer 示例
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-tx-producer");
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // 事务强制开启

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// 1. 初始化事务
producer.initTransactions();

try {
    // 2. 开启事务
    producer.beginTransaction();

    // 3. 在事务中发送消息
    producer.send(new ProducerRecord<>("orders", "order-123", "created"));
    producer.send(new ProducerRecord<>("payments", "order-123", "pending"));

    // 4. 提交事务
    producer.commitTransaction();
} catch (Exception e) {
    // 5. 回滚事务
    producer.abortTransaction();
}
```

## 11.2 事务底层原理

```
┌───────────────────────────────────────────────────────┐
│                   事务流程                              │
│                                                       │
│  1. Producer 向 TransactionCoordinator 注册            │
│     (根据 transactional.id 的 hash 选择 Coordinator)   │
│                                                       │
│  2. Coordinator 分配 Producer Epoch（单调递增）         │
│     用于跨 Session 去重（僵尸生产者检测）               │
│                                                       │
│  3. beginTransaction → 标记事务开始                    │
│                                                       │
│  4. 发送消息 → 消息标记为事务消息                       │
│     ┌──────────────────────────┐                      │
│     │ Transactional Marker     │                      │
│     │ (COMMIT / ABORT)         │                      │
│     │ 写入 __transaction_state │                      │
│     └──────────────────────────┘                      │
│                                                       │
│  5. commitTransaction → Coordinator 写入 PREPARE_COMMIT│
│     → 写入 TxnMarker 到各 Partition → COMMITTED        │
│                                                       │
│  6. Consumer 读取时根据 Marker 决定是否返回消息          │
│     (read_committed / read_uncommitted)                │
└───────────────────────────────────────────────────────┘
```

# 12 消息压缩

## 12.1 压缩算法对比

| 算法 | 压缩率 | 速度 | CPU 开销 | 适用 |
|------|--------|------|----------|------|
| none | 0% | 最快 | 最小 | 消息已压缩 / 网络带宽充足 |
| gzip | 最高 | 最慢 | 最高 | 带宽受限，延迟不敏感 |
| snappy | 中等 | 快 | 中等 | 延迟与吞吐平衡（常用） |
| lz4 | 中等 | 最快 | 低 | 低延迟高吞吐（推荐） |
| zstd | 高 | 快 | 中 | 新版推荐 |

```java
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
```

## 12.2 压缩发生在哪个层级

```
发送端压缩:
  Producer → (snappy/lz4) → Broker（原样存储）→ Consumer（原样传输，解压在 Consumer 端）

如果使用 topic-level compression:
  消息到达 Broker 后也可以配置 Broker 端再压缩
```

# 13 Producer 性能调优

## 13.1 核心参数速查

```java
// === 高吞吐配置模板 ===
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.serializer", StringSerializer.class);
props.put("value.serializer", StringSerializer.class);

props.put("acks", "1");                              // 或 all（根据需要）
props.put("enable.idempotence", true);
props.put("compression.type", "lz4");
props.put("batch.size", 131072);                     // 128KB
props.put("linger.ms", 10);                          // 10ms 聚合
props.put("buffer.memory", 67108864);                // 64MB
props.put("max.request.size", 10485760);             // 10MB
props.put("max.in.flight.requests.per.connection", 5);
props.put("partitioner.class", DefaultPartitioner.class);
```

## 13.2 常见 Producer 异常

| 异常 | 原因 | 解决 |
|------|------|------|
| `TimeoutException` | 发送超时 | 增大 `delivery.timeout.ms` / 检查网络 |
| `RecordTooLargeException` | 消息超过大小限制 | 增大 `max.request.size` / 拆分消息 |
| `BufferExhaustedException` | RecordAccumulator 满 | 增大 `buffer.memory` / 检查 Sender 线程 |
| `NotEnoughReplicasException` | ISR 不足 | 减少 `min.insync.replicas` / 恢复 Broker |
| `NetworkException` | 网络故障 | 检查网络 / 增大 `retries` |
| `OutOfOrderSequenceException` | 序列号跳号 | 重启 Producer / 检查事务配置 |
| `ProducerFencedException` | 僵尸 Producer 被隔离 | 检查 `transactional.id` 唯一性 |
