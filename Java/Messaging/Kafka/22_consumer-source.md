# 1 源码分析总览

## 1.1 Consumer 核心类图

```
KafkaConsumer
│
├── ConsumerCoordinator (Group 协调)
│   ├── JoinGroup / SyncGroup / Heartbeat
│   ├── PartitionAssignor (Range / RoundRobin / Sticky / CooperativeSticky)
│   └── SubscriptionState (分区状态管理)
│
├── Fetcher (消息拉取)
│   ├── 构建 FetchRequest
│   ├── 发送请求 / 接收响应
│   └── 反序列化 / 放入 ConsumerRecords
│
├── ConsumerNetworkClient (异步请求管理)
│   ├── 发送请求 + 注册 Future
│   └── poll: 轮询网络事件
│
├── SubscriptionState (分区/offset/分配状态)
│   ├── assignment: 已分配的 Partition 集合
│   ├── position(tp): 下一个消费的 offset
│   └── committed(tp): 已提交的 offset
│
├── ConsumerMetadata (元数据)
│   └── 集群信息 + Coordinator 位置
│
└── Deserializers (反序列化)
    └── Key / Value 反序列化器
```

# 2 KafkaConsumer 初始化

## 2.1 构造器分析

```java
// KafkaConsumer.java

public class KafkaConsumer<K, V> implements Consumer<K, V> {
    private final String clientId;
    private final ConsumerCoordinator coordinator;
    private final Fetcher<K, V> fetcher;
    private final ConsumerNetworkClient client;
    private final SubscriptionState subscriptions;
    private final ConsumerMetadata metadata;
    private final Deserializer<K> keyDeserializer;
    private final Deserializer<V> valueDeserializer;
    private final ConsumerInterceptors<K, V> interceptors;
    private final Time time;
    private final long retryBackoffMs;

    KafkaConsumer(ConsumerConfig config,
                  Deserializer<K> keyDeserializer,
                  Deserializer<V> valueDeserializer) {
        // ① 初始化时间
        this.time = Time.SYSTEM;
        this.clientId = config.getString(ConsumerConfig.CLIENT_ID_CONFIG);

        // ② 反序列化器
        this.keyDeserializer = keyDeserializer;
        this.valueDeserializer = valueDeserializer;

        // ③ 拦截器
        this.interceptors = new ConsumerInterceptors<>(interceptorList);

        // ④ 核心状态管理
        GroupRebalanceConfig groupRebalanceConfig = new GroupRebalanceConfig(
            config,
            GroupRebalanceConfig.ProtocolType.CONSUMER);

        this.subscriptions = new SubscriptionState(
            new LogContext(),
            OffsetResetStrategy.valueOf(
                config.getString(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG).toUpperCase()));

        // ⑤ 元数据管理
        this.metadata = new ConsumerMetadata(
            retryBackoffMs,
            config.getLong(ConsumerConfig.METADATA_MAX_AGE_CONFIG),
            !config.getBoolean(ConsumerConfig.EXCLUDE_INTERNAL_TOPICS_CONFIG),
            ...);

        // ⑥ 网络客户端
        this.client = new ConsumerNetworkClient(
            logContext,
            kafkaClient,      // NetworkClient
            metadata,
            time,
            retryBackoffMs,
            config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG),
            heartbeatIntervalMs);

        // ⑦ Fetcher (消息拉取器)
        this.fetcher = new Fetcher<>(
            logContext,
            this.client,
            fetchMinBytes,
            fetchMaxWaitMs,
            fetchMaxBytes,
            maxPartitionFetchBytes,
            maxPollRecords,
            checkCrcs,
            this.keyDeserializer,
            this.valueDeserializer,
            this.metadata,
            this.subscriptions,
            this.metrics,
            ...);

        // ⑧ GroupCoordinator (消费者组协调)
        this.coordinator = new ConsumerCoordinator(
            groupRebalanceConfig,
            logContext,
            this.client,
            assignors,                // PartitionAssignor 列表
            this.metadata,
            this.subscriptions,
            metrics,
            ...);
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/KafkaConsumer.java:190`

# 3 poll() 方法核心链路

## 3.1 KafkaConsumer.poll()

```java
// KafkaConsumer.java

@Override
public ConsumerRecords<K, V> poll(final Duration timeout) {
    return poll(time.timer(timeout), true);
}

private ConsumerRecords<K, V> poll(final Timer timer, final boolean includeMetadataInTimeout) {
    acquireAndEnsureOpen();
    try {
        // 保存当前消费位置（用于拦截器对比）
        this.subscriptions.allConsumed();

        // ====== 阶段 1: 确保 Coordinator 就绪 ======
        if (!this.subscriptions.hasAutoAssignedPartitions()) {
            if (includeMetadataInTimeout) {
                // Coordinator 发现（如果 group.id 非空）
                if (!this.coordinator.poll(timer)) {
                    return ConsumerRecords.empty();
                }
            }
        }

        // ====== 阶段 2: 处理 Rebalance（如果需要） ======
        if (this.subscriptions.hasAutoAssignedPartitions()) {
            // 检查是否需要重新加入 Group
            if (this.coordinator.needRejoin()) {
                if (!this.coordinator.poll(timer)) {
                    return ConsumerRecords.empty();
                }
            }
            // 等待 Rebalance 完成
            this.coordinator.ensureActiveGroup(waitTimer);
        }

        // ====== 阶段 3: 更新元数据（如果分区分配已变更） ======
        if (this.subscriptions.hasAutoAssignedPartitions()) {
            this.coordinator.poll(timer);
        }

        // ====== 阶段 4: 拉取消息 ======
        // ① 构建并发送 Fetch 请求
        Map<TopicPartition, List<ConsumerRecord<K, V>>> records =
            pollForFetches(timer);

        // ② 包装为 ConsumerRecords
        ConsumerRecords<K, V> consumerRecords =
            new ConsumerRecords<>(records);

        // ====== 阶段 5: 自动提交 Offset（如果启用） ======
        if (!this.subscriptions.hasAutoAssignedPartitions()) {
            maybeAutoCommitOffsetsAsync(timer.currentTimeMs());
        }

        // ====== 阶段 6: 拦截器后处理 ======
        return this.interceptors.onConsume(consumerRecords);
    } finally {
        release();
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/KafkaConsumer.java:1210`

## 3.2 pollForFetches() 拉取实现

```java
// KafkaConsumer.java

private Map<TopicPartition, List<ConsumerRecord<K, V>>> pollForFetches(Timer timer) {
    long pollTimeout = Math.min(this.coordinator.timeToNextPoll(timer.currentTimeMs()),
                                timer.remainingMs());

    // ① 如果已有可用数据 → 立即返回
    final Map<TopicPartition, List<ConsumerRecord<K, V>>> records =
        fetcher.fetchedRecords();
    if (!records.isEmpty()) {
        return records;
    }

    // ② 构建并发送 Fetch 请求
    fetcher.sendFetches();

    // ③ 轮询网络 I/O（等待响应）
    //    阻塞在 poll() 上，直到：
    //    - 有 fetch.min.bytes 数据到达
    //    - fetch.max.wait.ms 超时
    //    - poll timeout 到期
    client.poll(pollTimeout, timer.currentTimeMs(), () -> {
        return !fetcher.hasAvailableFetches();
    });

    // ④ 收集并返回已完成的 FetchResponse
    //    将数据从 Fetcher 缓冲区移动到 ConsumerRecords
    return fetcher.fetchedRecords();
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/KafkaConsumer.java:1533`

# 4 ConsumerCoordinator 协调器源码

## 4.1 加入 Group 的主流程

```java
// ConsumerCoordinator.java

public boolean poll(Timer timer) {
    maybeUpdateSubscriptionMetadata();

    invokeCompletedOffsetCommitCallbacks();

    // ① 周期性心跳发送
    if (subscriptions.partitionsAutoAssigned()) {
        pollHeartbeat(timer.currentTimeMs());
    }

    // ② 自动提交 offset（如果需要）
    maybeAutoCommitOffsetsAsync(timer.currentTimeMs());

    // ③ 如果需要 Rejoin → 执行 Rebalance
    if (subscriptions.hasAutoAssignedPartitions() && needRejoin()) {
        if (rejoinNeededOrPending()) {
            if (!ensureActiveGroup(waitTimer)) {
                return false;
            }
        }
    }

    return true;
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/ConsumerCoordinator.java:357`

## 4.2 Rebalance 完整流程

```java
// AbstractCoordinator.java (ConsumerCoordinator 的父类)

boolean joinGroupIfNeeded(final Timer timer) {
    while (rejoinNeededOrPending()) {
        // === 步骤 1: 检查是否需要寻找 Coordinator ===
        if (!ensureCoordinatorReady(timer)) {
            return false;
        }

        // === 步骤 2: 如果需要 → 让出所有分区 (Cooperative Rebalance) ===
        if (needsJoinPrepare) {
            needsJoinPrepare = false;
            onJoinPrepare(generation.generationId, generation.memberId);
            // ConsumerCoordinator.onJoinPrepare →
            //   ① 调用 RebalanceListener.onPartitionsRevoked() 或 onPartitionsLost()
            //   ② 如果是 Eager: 撤销所有分区
            //   ③ 如果是 Cooperative: 仅撤销需要迁移的分区
        }

        // === 步骤 3: 发送 JoinGroup 请求 ===
        final RequestFuture<ByteBuffer> future = sendJoinGroup();
        client.poll(future, timer);
        //   JoinGroupRequest 包含:
        //   - groupId
        //   - sessionTimeoutMs
        //   - memberId (首次为空)
        //   - protocolType: "consumer"
        //   - groupProtocols: [RangeAssignor, CooperativeStickyAssignor, ...]

        if (!future.isDone()) {
            return false;
        }

        // === 步骤 4: 检查 JoinGroup 响应 ===
        JoinGroupResponse joinResponse = ...;
        if (joinResponse == JoinGroupResponse.UNKNOWN_MEMBER_ID) {
            // Member ID 过期（可能 Coordinator 重启）
            resetAndReboot();
            continue;
        }

        // === 步骤 5: 如果当前 Consumer 是 Group Leader → 执行分区分配 ===
        if (joinResponse.isLeader()) {
            Map<String, List<TopicPartition>> assignment =
                onLeaderElected(joinResponse.generationId(),
                                joinResponse.memberId(),
                                joinResponse.members());
            //   ① 获取所有 Consumer 的订阅信息
            //   ② 调用 PartitionAssignor.assign() 执行分配算法
            //   ③ 返回分配结果

            // === 步骤 6: 发送 SyncGroup 请求 ===
            sendSyncGroup(joinResponse.generationId(),
                          joinResponse.memberId(),
                          assignment);
            //   Leader 将分配结果发给 Coordinator
        } else {
            // === 步骤 6 (非 Leader): 发送空分配给 Coordinator ===
            sendSyncGroup(joinResponse.generationId(),
                          joinResponse.memberId(),
                          Collections.emptyMap());
        }

        // === 步骤 7: 处理 SyncGroup 响应 ===
        //   获取自己的分区分配
        //   调用 onJoinComplete() →
        //     onPartitionsAssigned(assignedPartitions)
        //     更新 subscriptionState
    }
    return true;
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/AbstractCoordinator.java:499`

# 5 Fetcher 拉取器源码

## 5.1 sendFetches() 构建请求

```java
// Fetcher.java

public synchronized void sendFetches() {
    // ① 获取所有需要拉取的分区（已分配且未暂停）
    Map<Node, Map<TopicPartition, FetchRequest.PartitionData>> fetchDataMap =
        prepareFetchRequests();

    // ② 遍历每个 Broker，构建并发送 FetchRequest
    for (Map.Entry<Node, Map<TopicPartition, FetchRequest.PartitionData>> entry
            : fetchDataMap.entrySet()) {
        Node node = entry.getKey();
        Map<TopicPartition, FetchRequest.PartitionData> fetchData = entry.getValue();

        // 构建 FetchRequest
        FetchRequest.Builder requestBuilder = FetchRequest.Builder
            .forConsumer(maxVersion, fetchMaxWaitMs, minBytes, fetchData)
            .setMaxBytes(maxBytes)
            .isolationLevel(isolationLevel);

        // 发送异步请求
        client.send(node, requestBuilder)
            .compose(new RequestFutureAdapter<ClientResponse, Void>() {
                @Override
                public void onSuccess(ClientResponse response, ...) {
                    handleFetchResponse(response, fetchData);
                }
            });
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/Fetcher.java:205`

## 5.2 prepareFetchRequests() 关键逻辑

```java
// Fetcher.java

private Map<Node, Map<TopicPartition, FetchRequest.PartitionData>> prepareFetchRequests() {
    Map<Node, Map<TopicPartition, FetchRequest.PartitionData>> fetchDataMap =
        new LinkedHashMap<>();

    // 获取所有已分配的分区
    for (TopicPartition tp : subscriptions.fetchablePartitions()) {
        // ① 检查该分区是否已有未完成的 Fetch 请求
        if (hasInFlightRequest(tp)) continue;

        // ② 获取 fetch offset
        long fetchOffset = subscriptions.position(tp); // 当前消费位置
        int leaderEpoch = subscriptions.leaderEpoch(tp);
        int maxBytes = maxPartitionFetchBytes;

        // ③ 添加到 Broker 对应的 fetchData
        Node node = metadata.fetch().leaderFor(tp);
        if (node == null) {
            metadata.requestUpdate(); // 元数据过期 → 异步刷新
        } else {
            fetchDataMap.computeIfAbsent(node, k -> new LinkedHashMap<>())
                .put(tp, new FetchRequest.PartitionData(
                    fetchOffset, // ← 从这个 offset 开始拉取
                    leaderEpoch,
                    maxBytes
                ));
        }
    }
    return fetchDataMap;
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/Fetcher.java:711`

## 5.3 handleFetchResponse() 处理响应

```java
// Fetcher.java

private void handleFetchResponse(ClientResponse resp,
                                  Map<TopicPartition, FetchRequest.PartitionData> fetchData) {
    FetchResponse<Records> response = (FetchResponse<Records>) resp.responseBody();

    for (Map.Entry<TopicPartition, FetchResponse.PartitionData<Records>> entry
            : response.responseData().entrySet()) {
        TopicPartition tp = entry.getKey();
        FetchResponse.PartitionData<Records> partitionData = entry.getValue();

        // ① 错误处理
        Errors error = partitionData.error;
        if (error == Errors.OFFSET_OUT_OF_RANGE) {
            // offset 不在当前日志中（已被删除）
            // 处理: 重置 offset 到 earliest/latest
            handleOffsetOutOfRange(tp, partitionData);
        } else if (error == Errors.NOT_LEADER_OR_FOLLOWER) {
            // Leader 变更 → 刷新元数据
            metadata.requestUpdate();
        } else if (error == Errors.NONE) {
            // ② 正常处理 → 放入 completedFetches 队列
            completedFetches.add(new CompletedFetch(
                tp,
                partitionData,
                metrics,
                // 记录 fetchOffset, partitionData.highWatermark, partitionData.logStartOffset
                ...
            ));
        }
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/Fetcher.java:480`

## 5.4 fetchedRecords() 解析数据

```java
// Fetcher.java

public Map<TopicPartition, List<ConsumerRecord<K, V>>> fetchedRecords() {
    Map<TopicPartition, List<ConsumerRecord<K, V>>> records = new LinkedHashMap<>();

    int recordsRemaining = maxPollRecords; // max.poll.records
    while (recordsRemaining > 0) {
        // ① 从 completedFetches 中取一个已完成的分区响应
        CompletedFetch completedFetch = completedFetches.peek();
        if (completedFetch == null) break;

        // ② 解析下一批 Record
        RecordBatch batch = completedFetch.nextFetchedRecordBatch();
        if (batch == null) {
            // 该分区已完成解析，出队 → 更新 position
            completedFetches.poll();
            subscriptions.position(tp, completedFetch.nextFetchOffset);
            continue;
        }

        // ③ 检查隔离级别（事务消息过滤）
        if (isolationLevel == IsolationLevel.READ_COMMITTED
                && !batch.isTransactional()) {
            continue; // 等待 COMMIT/ABORT marker
        }

        // ④ 反序列化并添加到结果集
        for (Record record : batch) {
            ConsumerRecord<K, V> consumerRecord = parseRecord(tp, batch, record);
            records.computeIfAbsent(tp, k -> new ArrayList<>()).add(consumerRecord);
            recordsRemaining--;
        }
    }
    return records;
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/Fetcher.java:375`

# 6 SubscriptionState 状态管理

```java
// SubscriptionState.java

public class SubscriptionState {
    // 枚举: 分区状态
    //   AWAITING_RESET: 等待重置 offset
    //   FETCHING: 正在拉取
    //   SUSPENDED: 暂停（被 pause）
    //   REVOKED: 被撤销

    // 关键数据结构
    private final Map<TopicPartition, TopicPartitionState> assignment;
    // assignment.get(tp) → {
    //   position,          // 下一个消费的 offset
    //   consumed,          // poll 前的 position（用于拦截器对比）
    //   committed,         // 已提交的 offset
    //   highWatermark,     // HW
    //   logStartOffset,    // Log Start Offset
    //   leaderEpoch,       // Leader Epoch
    // }

    // 获取允许从 earliest/latest/none 生效的 offset 重置策略
    private final OffsetResetStrategy defaultResetStrategy;

    public long position(TopicPartition tp) {
        TopicPartitionState state = assignment.get(tp);
        if (state != null) return state.position;
        else return getResetStrategy(tp) == OffsetResetStrategy.NONE
            ? fetchCommitted(tp).offset  // 从已提交的 offset 开始
            : getPartitionResetOffset(tp); // earliest/latest
    }
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/SubscriptionState.java:86`

# 7 心跳与偏移提交

## 7.1 心跳周期

```java
// AbstractCoordinator.java

public void pollHeartbeat(long now) {
    if (coordinatorUnknown()) return;

    long timeSinceLastHeartbeat = now - lastHeartbeatSend;

    if (timeSinceLastHeartbeat > heartbeatIntervalMs) {
        // 需要发送心跳
        sendHeartbeatRequest();
        lastHeartbeatSend = now;
    }
}

private synchronized void sendHeartbeatRequest() {
    HeartbeatRequest.Builder requestBuilder = new HeartbeatRequest.Builder(
        groupId,
        generation.generationId,
        generation.memberId);

    client.send(coordinator, requestBuilder)
        .compose(heartbeatResponse -> {
            Errors error = heartbeatResponse.error();
            if (error == Errors.NONE) {
                // 心跳成功 → 重置 session 超时
                heartbeat.lastHeartbeatReceive = time.milliseconds();
            } else if (error == Errors.COORDINATOR_LOAD_IN_PROGRESS) {
                coordinatorDead();
            } else {
                // ... 其他错误
            }
            return null;
        });
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/AbstractCoordinator.java:773`

## 7.2 Offsets 提交

```java
// ConsumerCoordinator.java

public RequestFuture<Void> commitOffsetsAsync(
        final Map<TopicPartition, OffsetAndMetadata> offsets,
        final OffsetCommitCallback callback) {
    // 构建 OffsetCommitRequest
    OffsetCommitRequest.Builder builder = new OffsetCommitRequest.Builder(
        groupId,
        offsets
    );

    return client.send(coordinator, builder)
        .compose(response -> {
            for (Map.Entry<TopicPartition, Errors> entry
                    : response.errors().entrySet()) {
                if (entry.getValue() == Errors.NONE) {
                    // 更新本地 tracking 的 committed offset
                    subscriptions.committed(entry.getKey(),
                                            offsets.get(entry.getKey()));
                }
            }
            callback.onComplete(offsets, null);
            return null;
        });
}
```

> 文件位置: `clients/src/main/java/org/apache/kafka/clients/consumer/internals/ConsumerCoordinator.java:376`
