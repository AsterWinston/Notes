# 1 源码分析总览

## 1.1 源码环境准备

```bash
# Kafka 源码仓库
git clone https://github.com/apache/kafka.git
cd kafka
git checkout 3.6.0  # 或你分析的版本

# 项目结构
# clients/        → Producer, Consumer, AdminClient (Java)
# server/         → Broker 核心 (Scala)
# connect/        → Kafka Connect
# streams/        → Kafka Streams
# core/           → 核心数据结构 (Java)

# 导入 IDEA
# gradle idea
```

## 1.2 Producer 核心类图

```
KafkaProducer
│
├── Partitioner (分区器)
│   └── DefaultPartitioner
│
├── Serializer (序列化器)
│   └── StringSerializer / ByteArraySerializer
│
├── ProducerInterceptor (拦截器链)
│
├── RecordAccumulator (消息缓冲区)
│   ├── ConcurrentMap<TopicPartition, Deque<ProducerBatch>>
│   └── BufferPool (内存池)
│
├── Sender (发送线程)
│   └── NetworkClient
│       ├── Selector (NIO 网络 I/O)
│       ├── Metadata (集群元数据)
│       └── InFlightRequests (等待确认的请求)
│
└── TransactionManager (事务管理, 可选)
```

# 2 KafkaProducer 初始化

## 2.1 构造器分析

```java
// org.apache.kafka.clients.producer.KafkaProducer

public class KafkaProducer<K, V> implements Producer<K, V> {
    private final Partitioner partitioner;
    private final int maxRequestSize;
    private final long totalMemorySize;
    private final Metadata metadata;
    private final RecordAccumulator accumulator;
    private final Sender sender;
    private final Thread ioThread;
    private final CompressionType compressionType;
    private final Transactions transactionManager;
    private final Time time;
    private final ApiVersions apiVersions;

    public KafkaProducer(Properties properties) {
        this(new ProducerConfig(properties), null, null);
    }

    public KafkaProducer(Properties properties,
                         Serializer<K> keySerializer,
                         Serializer<V> valueSerializer) {
        this(new ProducerConfig(
                ProducerConfig.addSerializerToConfig(properties, keySerializer, valueSerializer)),
             keySerializer,
             valueSerializer);
    }

    // 主构造器
    KafkaProducer(ProducerConfig config,
                  Serializer<K> keySerializer,
                  Serializer<V> valueSerializer,
                  ProducerMetadata metadata,    // metadata wrapper
                  KafkaClient kafkaClient,      // NetworkClient
                  ProducerInterceptors<K, V> interceptors,
                  Time time) {

        // ① 初始化配置参数
        ProducerMetrics metrics = new ProducerMetrics(config);
        this.partitioner = config.getConfiguredInstance(
            ProducerConfig.PARTITIONER_CLASS_CONFIG, Partitioner.class);
        this.compressionType = CompressionType.forName(config.getString(
            ProducerConfig.COMPRESSION_TYPE_CONFIG));

        // ② 序列化器
        this.keySerializer = keySerializer != null
            ? keySerializer
            : config.getConfiguredInstance(
                ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, Serializer.class);
        this.valueSerializer = valueSerializer != null
            ? valueSerializer
            : config.getConfiguredInstance(
                ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, Serializer.class);

        // ③ 拦截器
        List<ProducerInterceptor<K, V>> interceptorList = // ...
        this.interceptors = new ProducerInterceptors<>(interceptorList);

        // ④ 元数据管理
        this.metadata = metadata != null
            ? metadata
            : new ProducerMetadata(
                retryBackoffMs,
                config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),
                ...);

        // ⑤ 核心缓冲区
        this.accumulator = new RecordAccumulator(
            config.getInt(ProducerConfig.BATCH_SIZE_CONFIG),
            this.totalMemorySize,
            // ...
            this.compressionType,
            this.time);

        // ⑥ NetworkClient
        NetworkClient networkClient = kafkaClient != null
            ? (NetworkClient) kafkaClient
            : new NetworkClient(
                selector,
                this.metadata,
                clientId,
                maxInFlightRequestsPerConnection,
                // ...
                config.getInt(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG),
                this.time);

        // ⑦ Sender 线程
        this.sender = newSender(logContext, kafkaClient, this.metadata);
        String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;
        this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
        this.ioThread.start();  // ← 启动 Sender 线程
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java:120`

# 3 send() 方法核心链路

## 3.1 KafkaProducer.send()

```java
// KafkaProducer.java

@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
    return send(record, null); // null callback → 异步无回调
}

@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // ① 拦截器预处理
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);

    return doSend(interceptedRecord, callback);
}
```

## 3.2 doSend() 完整链路

```java
// KafkaProducer.java

private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        throwIfProducerClosed();

        // === 阶段 1: 等待并获取集群元数据 ===
        long nowMs = time.milliseconds();
        ClusterAndWaitTime clusterAndWaitTime =
            waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);

        // === 阶段 2: 序列化 Key 和 Value ===
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(),
                record.headers(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("...");
        }

        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(),
                record.headers(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("...");
        }

        // === 阶段 3: 计算目标分区 ===
        int partition = partition(record, serializedKey, serializedValue,
            clusterAndWaitTime.cluster);
        tp = new TopicPartition(record.topic(), partition);

        setReadOnly(record.headers());

        Header[] headers = record.headers().toArray();

        // === 阶段 4: 估算消息大小并校验 ===
        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(
            apiVersions.maxUsableProduceMagic(),
            compressionType,
            serializedKey,
            serializedValue,
            headers);
        ensureValidRecordSize(serializedSize);

        // === 阶段 5: 追加到 RecordAccumulator ===
        //     如果 Batch 满了 → 立即唤醒 Sender 线程
        RecordAccumulator.RecordAppendResult result =
            accumulator.append(
                tp,                          // TopicPartition
                timestamp,                   // 时间戳
                serializedKey,
                serializedValue,
                headers,
                interceptCallback,           // 包装了拦截器的 callback
                clusterAndWaitTime.remainingMs,
                nowMs);

        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender...");
            this.sender.wakeup(); // ← 唤醒 Sender
        }

        // === 阶段 6: 返回 Future ===
        return result.future;
    } catch (/* ... */) {
        // 异常处理 → 拦截器 onAcknowledgement + callback.onCompletion(error)
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java:457` (doSend 方法)

## 3.3 partition() 方法

```java
// KafkaProducer.java

private int partition(ProducerRecord<K, V> record,
                      byte[] serializedKey,
                      byte[] serializedValue,
                      Cluster cluster) {
    Integer partition = record.partition();
    // 如果 ProducerRecord 中指定了分区 → 直接使用
    return partition != null
        ? partition
        : partitioner.partition(
            record.topic(), record.key(), serializedKey,
            record.value(), serializedValue, cluster);
}

// DefaultPartitioner.partition
public int partition(String topic, Object key, byte[] keyBytes,
                     Object value, byte[] valueBytes,
                     Cluster cluster, int numPartitions) {
    if (keyBytes == null) {
        // 粘性分区 (Kafka 2.4+)
        return stickyPartitionCache.partition(topic, cluster);
    } else {
        // hash(key) % numPartitions
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java:70`

# 4 RecordAccumulator 内部机制

## 4.1 追加逻辑 append()

```java
// RecordAccumulator.java

public RecordAppendResult append(TopicPartition tp,
                                  long timestamp,
                                  byte[] key,
                                  byte[] value,
                                  Header[] headers,
                                  Callback callback,
                                  long maxTimeToBlock,
                                  long nowMs) throws InterruptedException {

    ByteBuffer buffer = null;
    if (headers == null) headers = Record.EMPTY_HEADERS;

    try {
        // ① 获取或创建该分区对应的 Deque<ProducerBatch>
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);

        synchronized (dq) {
            // if (closed) throw new KafkaException("...");

            // ② 尝试追加到已有的未满 Batch
            RecordAppendResult appendResult = tryAppend(timestamp, key, value,
                headers, callback, dq, nowMs);
            if (appendResult != null) {
                return appendResult; // 追加成功
            }
        }

        // ③ 创建新的 Batch（需要分配内存）
        byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
        int size = Math.max(this.batchSize,
            AbstractRecords.estimateSizeInBytesUpperBound(
                maxUsableMagic, compressionType, key, value, headers));

        // ④ 从 BufferPool 分配空间
        buffer = free.allocate(size, maxTimeToBlock); // 可能阻塞
        //    ├── free.availableMemory < size  → 等待 Sender 发送释放空间
        //    └── 等待超时 → TimeoutException

        synchronized (dq) {
            RecordAppendResult appendResult = tryAppend(timestamp, key, value,
                headers, callback, dq, nowMs);
            if (appendResult != null) {
                // 在等待内存期间，有其他线程创建了新 Batch 并追加成功
                free.deallocate(buffer);
                return appendResult;
            }

            // ⑤ 创建 MemoryRecordsBuilder → ProducerBatch → 追加到 Deque
            MemoryRecordsBuilder recordsBuilder =
                recordsBuilder(buffer, maxUsableMagic, compressionType, ...);
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, nowMs);
            FutureRecordMetadata future = Objects.requireNonNull(
                batch.tryAppend(timestamp, key, value, headers, callback, nowMs));

            dq.addLast(batch);
            incomplete.add(batch); // 等待确认的 Batch 集合

            buffer = null;
            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(),
                                         true);
        }
    } finally {
        if (buffer != null) free.deallocate(buffer);
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java:186`

## 4.2 BufferPool 设计

```java
// BufferPool.java

public class BufferPool {
    private final long totalMemory;           // buffer.memory
    private final int poolableSize;           // batch.size
    private final ReentrantLock lock;
    private final Deque<ByteBuffer> free;     // 池化的空闲 ByteBuffer
    private long nonPooledAvailableMemory;    // 未池化的可用内存
    private final Condition moreMemory;       // 等待通知

    public ByteBuffer allocate(int size, long maxTimeToBlockMs)
            throws InterruptedException {
        if (size > totalMemory) {
            throw new IllegalArgumentException(
                "Attempt to allocate " + size + " bytes, but buffer.memory is " + totalMemory);
        }

        ByteBuffer buffer = null;
        this.lock.lock();

        try {
            if (size == poolableSize && !this.free.isEmpty()) {
                // ① 从池中取一个空闲的 ByteBuffer
                return this.free.pollFirst();
            }

            int freeListSize = freeSize() * this.poolableSize;
            if (this.nonPooledAvailableMemory + freeListSize >= size) {
                // ② 从非池化内存中分配
                freeUp(size);
                this.nonPooledAvailableMemory -= size;
            } else {
                // ③ 内存不足 → 阻塞等待
                int accumulated = 0;
                long remainingMs = maxTimeToBlockMs;
                while (accumulated < size) {
                    long startMs = time.milliseconds();
                    if (!this.moreMemory.await(remainingMs, TimeUnit.MILLISECONDS)) {
                        throw new TimeoutException("Failed to allocate memory within...");
                    }
                    long elapsed = time.milliseconds() - startMs;
                    remainingMs = Math.max(0, remainingMs - elapsed);
                    accumulated += freeUp(size - accumulated);
                }
                this.nonPooledAvailableMemory -= size;
            }
        } finally {
            this.lock.unlock();
        }

        return buffer;
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/internals/BufferPool.java:93`

# 5 Sender 线程核心逻辑

## 5.1 Sender.run()

```java
// Sender.java (简化)

public class Sender implements Runnable {
    private final KafkaClient client;         // NetworkClient
    private final RecordAccumulator accumulator;
    private final Metadata metadata;
    private final TransactionManager transactionManager;

    @Override
    public void run() {
        log.debug("Starting Kafka producer I/O thread.");

        long now;
        while (running) {
            try {
                now = time.milliseconds();
                // === 阶段 1: 从 RecordAccumulator 取出就绪的 Batch ===
                long pollTimeout = sendProducerData(now);

                // === 阶段 2: 执行网络 I/O ===
                client.poll(pollTimeout, now);
                //    ├── 发送所有就绪的请求（Produce / Metadata / ...）
                //    ├── 接收响应
                //    └── 触发 Callback (completeResponses)
            } catch (Exception e) {
                log.error("Uncaught error in kafka producer I/O thread: ", e);
            }
        }

        log.debug("Kafka producer I/O thread completed.");
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/internals/Sender.java:131`

## 5.2 sendProducerData() -> 准备数据

```java
// Sender.java

private long sendProducerData(long now) {
    Cluster cluster = metadata.fetch();
    // ① 获取准备就绪的节点（Leader 可用的 Broker）
    RecordAccumulator.ReadyCheckResult result =
        this.accumulator.ready(cluster, now);

    // ② 对于 Leader 不可用的节点 → 请求更新元数据
    if (!result.unknownLeaderTopics.isEmpty()) {
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic, now);
        this.metadata.requestUpdateForNewTopics();
    }

    // ③ 对于未就绪的节点 → 检查是否有连接并初始化
    Iterator<Node> iter = result.readyNodes.iterator();
    while (iter.hasNext()) {
        Node node = iter.next();
        if (!this.client.ready(node, now)) {
            iter.remove();
        }
    }

    // ④ 为每个就绪的 Broker 创建 ProduceRequest
    //    → accumulator.drain(cluster, result.readyNodes, maxRequestSize, now)
    //    → 每个 Broker 一组 TopicPartition Batch
    Map<Integer, List<ProducerBatch>> batches =
        this.accumulator.drain(cluster, result.readyNodes,
                               this.maxRequestSize, now);

    addToInflightBatches(batches);

    // ⑤ 对于需要 leader 确认的 → 检查 ISR 状态
    if (guaranteeMessageOrder) {
        for (List<ProducerBatch> batchList : batches.values()) {
            for (ProducerBatch batch : batchList) {
                batch.resetRetry(); // 重置重试状态
            }
        }
    }

    // ⑥ 发送 Produce Request 到对应 Broker
    sendProduceRequests(batches, now);

    return pollTimeout;
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/internals/Sender.java:412`

## 5.3 sendProduceRequests() -> 构建并发送

```java
// Sender.java

private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {
    for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet()) {
        List<ProducerBatch> batches = entry.getValue();
        // 构建 ProduceRequest (包含消息体、acks、timeout等)
        Map<TopicPartition, MemoryRecords> produceRecordsByPartition =
            new HashMap<>(batches.size());
        for (ProducerBatch batch : batches) {
            produceRecordsByPartition.put(batch.topicPartition, batch.records());
        }

        // 构建请求并发送
        ProduceRequest.Builder requestBuilder = ProduceRequest
            .forMagic(minMagic, acks, timeout, produceRecordsByPartition, transactionalId);
        RequestCompletionHandler callback = response -> {
            handleProduceResponse(response, batches, now);
        };

        String nodeId = Integer.toString(entry.getKey());
        ClientRequest clientRequest = client.newClientRequest(
            nodeId, requestBuilder, now, true, callback);
        client.send(clientRequest, now);
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/internals/Sender.java:597`

# 6 幂等与事务源码分析

## 6.1 幂等 Sequence Number 管理

```java
// ProducerBatch.java

public void setProducerState(ProducerIdAndEpoch producerIdAndEpoch, int baseSequence,
                             boolean isTransactional) {
    recordsBuilder.setProducerState(producerIdAndEpoch.producerId,
                                    producerIdAndEpoch.epoch,
                                    baseSequence,
                                    isTransactional);
}

// MemoryRecordsBuilder.java

public void setProducerState(long producerId, short producerEpoch, int baseSequence,
                              boolean isTransactional) {
    // 分配 Sequence Number: baseSequence, baseSequence+1, ..., baseSequence+N-1
    this.producerId = producerId;
    this.producerEpoch = producerEpoch;
    this.baseSequence = baseSequence;
    this.isTransactional = isTransactional;
}
```

## 6.2 TransactionManager 事务流程

```java
// TransactionManager.java

public class TransactionManager {
    private final String transactionalId;
    private long producerId;
    private short producerEpoch;
    private TransactionState currentState;
    private final Map<TopicPartition, OffsetAndMetadata> pendingOffsets;

    public synchronized void beginTransaction() {
        ensureTransactional();
        maybeFailWithError(); // 检查之前的事务错误
        transitionTo(TransactionState.IN_TRANSACTION);
    }

    public synchronized void beginCommit() {
        ensureTransactional();
        if (currentState != TransactionState.IN_TRANSACTION) {
            throw new KafkaException("Cannot commit transaction...");
        }
        transitionTo(TransactionState.COMMITTING_TRANSACTION);
    }

    public synchronized void beginAbort() {
        ensureTransactional();
        if (currentState == TransactionState.ABORTABLE_ERROR) {
            transitionTo(TransactionState.ABORTING_TRANSACTION);
        }
    }

    // 事务中的 sendOffsets
    public synchronized void addOffsetsToTxn(
            Map<TopicPartition, OffsetAndMetadata> offsets,
            ConsumerGroupMetadata groupMetadata) {
        ensureTransactional();
        pendingOffsets.putAll(offsets);
        // ... 发送 AddOffsetsToTxnRequest
    }

    // Producer Fencing 检测
    public synchronized void maybeTransitionToErrorState(RuntimeException e) {
        if (e instanceof ProducerFencedException) {
            transitionTo(TransactionState.FATAL_ERROR);
        }
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/producer/internals/TransactionManager.java:120`

# 7 关键并发设计分析

## 7.1 生产者线程模型

```
┌─────────────────────────────────────────────────────────────┐
│                     Producer 线程模型                         │
│                                                             │
│  用户线程 (N 个)                Sender 线程 (1 个)            │
│  ┌──────────────┐            ┌──────────────────────┐       │
│  │ Thread-1     │            │    Sender.run()       │       │
│  │ producer.send│──append───▶│    ┌────────────────┐ │       │
│  └──────────────┘            │    │① drain()       │ │       │
│                              │    │  从Accumulator  │ │       │
│  ┌──────────────┐            │    │  取就绪Batch     │ │       │
│  │ Thread-2     │──append───▶│    ├────────────────┤ │       │
│  │ producer.send│            │    │② sendRequests()│ │       │
│  └──────────────┘            │    │  通过NetworkClient│ │       │
│                              │    │  发送到Broker    │ │       │
│  ┌──────────────┐            │    ├────────────────┤ │       │
│  │ Thread-3     │──append───▶│    │③ handleResponses│ │      │
│  │ producer.send│            │    │  触发Callback    │ │       │
│  └──────────────┘            │    └────────────────┘ │       │
│                              └──────────────────────┘       │
│                                                             │
│  RecordAccumulator 内部同步:                                 │
│  - ConcurrentMap<TopicPartition, Deque<ProducerBatch>>      │
│  - 每个 Deque 使用 synchronized(dq) 保护                    │
│  - BufferPool 使用 ReentrantLock + Condition 保护           │
└─────────────────────────────────────────────────────────────┘
```

## 7.2 关键并发要点

1. **RecordAccumulator.append()** → 使用 `synchronized(dq)` 保护每个 Partition 的双端队列
2. **BufferPool.allocate()** → 使用 `ReentrantLock + Condition.await()` 实现生产者-消费者等待
3. **Sender.drain()** → 从 accumulator 取出 Batch，然后 accumulator 释放内存，通过 `moreMemory.signalAll()` 唤醒等待的 append 线程
4. **响应处理** → response 处理在 Sender 线程的 `client.poll()` 中完成，通过 `Callback` 通知用户线程
