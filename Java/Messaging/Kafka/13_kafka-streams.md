# 1 Kafka Streams 概述

## 1.1 什么是 Kafka Streams

Kafka Streams 是 Apache Kafka 官方提供的**流处理库**，允许开发者以 Java/Scala 编写流处理应用，不依赖外部集群（无 Spark/Flink 式的资源管理器）。

```
┌─────────────────────────────────────────────────────┐
│            Kafka Streams 在生态中的位置               │
│                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │ Producer │───▶│  Kafka   │───▶│ Consumer │       │
│  └──────────┘    │          │    └──────────┘       │
│                  └────┬─────┘                       │
│                       │                             │
│            ┌──────────▼──────────┐                  │
│            │   Kafka Streams     │                  │
│            │   (流处理应用)        │                  │
│            │                     │                  │
│            │  Input Topic(s)     │                  │
│            │       │             │                  │
│            │   Topology (DAG)    │                  │
│            │       │             │                  │
│            │  Output Topic(s)    │                  │
│            └─────────────────────┘                  │
│                                                     │
│  对比如:                                            │
│  - Spark Streaming/Kafka                   │
│  - Flink                                    │
│  Kafka Streams 不需要独立集群，作为库内嵌运行│
└─────────────────────────────────────────────────────┘
```

## 1.2 核心特点

| 特性 | 说明 |
|------|------|
| 内嵌库 | 无外部依赖，作为 Java 库引入 |
| 弹性伸缩 | 启动多个相同 `application.id` 的实例 → 自动负载均衡 |
| 状态存储 | RocksDB 本地状态存储 + Kafka Changelog 容错 |
| 精确一次 | 支持 EOS 语义（需配置） |
| 事件时间 | 支持事件时间窗口和乱序处理 |
| DSL + Processor API | 高层次流式 DSL + 底层 Processor API |

# 2 核心概念

## 2.1 Topology（拓扑）

Topology 是一个**有向无环图（DAG）**，定义了流处理的逻辑：

```
Topology 示例: WordCount

Source: "text-input" Topic
    │
    ▼
Processor: flatMapValues (split words)
    │
    ▼
Processor: groupByKey
    │
    ▼
Processor: count (windowed)
    │
    ▼
Sink: "word-count-output" Topic
```

## 2.2 KStream vs KTable vs GlobalKTable

| 抽象 | 本质 | 更新方式 | 示例 |
|------|------|---------|------|
| KStream | 无界数据流（append-only） | 每个记录独立 | 点击流、日志 |
| KTable | 变更日志（changelog） | 按 Key 更新（UPSERT） | 用户信息、配置 |
| GlobalKTable | 全量数据，广播到所有 Task | 按 Key 更新 | 静态维度表 |

```
KStream (流):
  Key:user1 → "click"   ← 独立事件
  Key:user2 → "view"
  Key:user1 → "purchase"

KTable (表):
  Key:user1 → {"name": "Alice", "age": 25}    ← 初始状态
  Key:user1 → {"name": "Alice", "age": 26}    ← 更新（覆盖旧值）

GlobalKTable:
  ┌─────────────────────────────────────┐
  │ 所有实例都拥有全量数据副本            │
  │ 用于流-表 Join 时维度表的数据           │
  └─────────────────────────────────────┘
```

## 2.3 StreamThread 与 StreamTask

```
Streams 应用 (Application ID: "my-streams-app")
│
├── StreamThread 1 (线程)
│   ├── Task 0_0 (处理 Partition 0)
│   │   ├── Topology: source → map → aggregate → sink
│   │   └── StateStore: local RocksDB for Task 0_0
│   └── Task 0_1 (处理 Partition 1)
│       └── ...
│
├── StreamThread 2 (线程)
│   ├── Task 1_2 (处理 Partition 2)
│   └── Task 1_3 (处理 Partition 3)
│
线程数 = num.stream.threads (配置)
Task 数 = max(Topic 分区数)
每个 Task 独占一个线程处理
```

# 3 核心 API 详解

## 3.1 Streams DSL

### 3.1.1 无状态操作

```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, String> stream = builder.stream("input-topic");

// filter: 过滤
stream.filter((key, value) -> value.length() > 10)
      .to("filtered-topic");

// filterNot: 反向过滤
stream.filterNot((key, value) -> value.contains("spam"))
      .to("clean-topic");

// map: 转换 Key 和 Value
stream.map((key, value) -> KeyValue.pair(key.toUpperCase(), value.toLowerCase()))
      .to("mapped-topic");

// mapValues: 仅转换 Value
stream.mapValues(value -> value.toUpperCase())
      .to("upper-topic");

// flatMap: 一对多转换
stream.flatMap((key, value) ->
    Arrays.stream(value.split(" "))
          .map(word -> KeyValue.pair(key, word))
          .collect(Collectors.toList()))
      .to("words-topic");

// flatMapValues: 仅对 Value 一对多
stream.flatMapValues(value ->
    Arrays.asList(value.split(" ")))
      .to("words-v2-topic");

// branch: 分流（多个输出）
Map<String, KStream<String, String>> branches = stream.split(Named.as("type-"))
    .branch((key, value) -> value.startsWith("ERROR"),
            Branched.as("error"))
    .branch((key, value) -> value.startsWith("WARN"),
            Branched.as("warn"))
    .defaultBranch(Branched.as("info"));

branches.get("type-error").to("error-topic");
branches.get("type-warn").to("warn-topic");
branches.get("type-info").to("info-topic");

// merge: 合并流
KStream<String, String> stream1 = builder.stream("topic-1");
KStream<String, String> stream2 = builder.stream("topic-2");
stream1.merge(stream2).to("merged-topic");

// peek: 旁路操作（不影响流）
stream.peek((key, value) -> log.info("Processing: {} -> {}", key, value))
      .to("output-topic");
```

### 3.1.2 有状态操作

```java
// groupBy + 聚合
KGroupedStream<String, String> grouped = stream
    .groupBy((key, value) -> value.split(":")[0]); // 重新选择分组键

// count: 计数
KTable<String, Long> countTable = grouped.count();

// aggregate: 自定义聚合
KTable<String, Long> sumTable = grouped.aggregate(
    () -> 0L,                                          // 初始化
    (key, value, aggregate) -> aggregate + parse(value), // 加法器
    Materialized.as("sum-store")                       // 存储名
);

// reduce: 规约
KTable<String, String> reduced = grouped.reduce(
    (oldValue, newValue) -> oldValue + "," + newValue
);
```

### 3.1.3 窗口操作

```java
import java.time.Duration;

// 1. 滚动窗口 (Tumbling Window)
// 固定大小，不重叠
// 窗口: [0,5) [5,10) [10,15) ...
TimeWindows tumblingWindow = TimeWindows
    .ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofSeconds(30));

KTable<Windowed<String>, Long> tumblingCount = stream
    .groupByKey()
    .windowedBy(tumblingWindow)
    .count();

// 2. 跳跃窗口 (Hopping Window)
// 固定大小，有重叠
// 窗口大小=10分钟, 步长=5分钟
// 窗口: [0,10) [5,15) [10,20) ...
TimeWindows hoppingWindow = TimeWindows
    .ofSizeAndGrace(Duration.ofMinutes(10), Duration.ofSeconds(30))
    .advanceBy(Duration.ofMinutes(5));

KTable<Windowed<String>, Long> hoppingCount = stream
    .groupByKey()
    .windowedBy(hoppingWindow)
    .count();

// 3. 滑动窗口 (Sliding Window)
// 按时间差定义的窗口，基于 Join 使用
SlidingWindows slidingWindow = SlidingWindows
    .ofTimeDifferenceAndGrace(Duration.ofMinutes(5), Duration.ofSeconds(30));

// 4. 会话窗口 (Session Window)
// 基于活动间隔动态合并
SessionWindows sessionWindow = SessionWindows
    .ofInactivityGapAndGrace(Duration.ofMinutes(5), Duration.ofSeconds(30));

KTable<Windowed<String>, Long> sessionCount = stream
    .groupByKey()
    .windowedBy(sessionWindow)
    .count();
```

### 3.1.4 Join 操作

```java
// 1. Stream-Stream Join（窗口 Join）
KStream<String, String> impressions = builder.stream("impressions");
KStream<String, String> clicks = builder.stream("clicks");

KStream<String, String> joined = impressions.join(
    clicks,
    (impressionValue, clickValue) -> impressionValue + "|" + clickValue,
    JoinWindows.ofTimeDifferenceAndGrace(
        Duration.ofMinutes(5), Duration.ofSeconds(30))
);

// 2. Stream-Table Join（流丰富）
KStream<String, String> purchases = builder.stream("purchases");
KTable<String, String> userProfiles = builder.table("user-profiles");

KStream<String, String> enrichedPurchases = purchases.join(
    userProfiles,
    (purchase, profile) -> purchase + "|" + profile
);

// 3. Stream-GlobalKTable Join（维度表 Join）
GlobalKTable<String, String> productCatalog = builder.globalTable("products");

KStream<String, String> orders = builder.stream("orders");
KStream<String, String> enrichedOrders = orders.join(
    productCatalog,
    (orderKey, order) -> orderKey,                // 从订单中提取关联 Key
    (order, product) -> order + "|" + product     // 合并
);

// 4. Table-Table Join
KTable<String, Long> leftTable = builder.table("left");
KTable<String, String> rightTable = builder.table("right");

KTable<String, String> joinedTable = leftTable.join(
    rightTable,
    (leftValue, rightValue) -> leftValue + "-" + rightValue
);
```

# 4 状态存储（State Store）

## 4.1 本地状态 + Changelog

```
┌────────────────────────────────────────────────────┐
│            状态存储架构                             │
│                                                    │
│  Stream Task (处理 Partition 0)                      │
│  ┌──────────────────┐     ┌──────────────────────┐ │
│  │  RocksDB (本地)   │────▶│  Changelog Topic     │ │
│  │  Key → Value      │     │  (容错复制)           │ │
│  └──────────────────┘     └──────────────────────┘ │
│                                                    │
│  正常处理: 读写本地 RocksDB → 高性能                │
│  故障恢复: 从 Changelog Topic 重建本地状态          │
│  滚动升级: 新实例从 Changelog 恢复状态              │
└────────────────────────────────────────────────────┘
```

## 4.2 状态存储类型

```java
// 1. Key-Value Store
KeyValueBytesStoreSupplier storeSupplier =
    Stores.persistentKeyValueStore("my-store");

KTable<String, Long> table = builder.table(
    "input-topic",
    Materialized.<String, Long>as(storeSupplier)
        .withKeySerde(Serdes.String())
        .withValueSerde(Serdes.Long())
);

// 2. Window Store
WindowBytesStoreSupplier windowStoreSupplier =
    Stores.persistentWindowStore("window-store",
        Duration.ofMinutes(5),
        Duration.ofMinutes(1),
        false);

// 3. Session Store
SessionBytesStoreSupplier sessionStoreSupplier =
    Stores.persistentSessionStore("session-store",
        Duration.ofMinutes(5));

// 4. 全局 Store (GlobalKTable)
GlobalKTable<String, UserProfile> globalTable = builder.globalTable(
    "user-profiles",
    Materialized.<String, UserProfile>as(
        Stores.persistentKeyValueStore("global-user-store"))
        .withKeySerde(Serdes.String())
        .withValueSerde(new UserProfileSerde())
);
```

## 4.3 交互式查询（Interactive Queries）

```java
// 从 Streams 应用中查询本地状态（暴露为 API）
REST API → Streams 应用

@GetMapping("/user/{userId}/count")
public Long getUserCount(@PathVariable String userId) {
    // 1. 查找负责该 Key 的实例
    KafkaStreams streams = ...;
    HostStoreInfo hostInfo = streams.queryMetadataForKey(
        "count-store", userId, Serdes.String().serializer());

    if (hostInfo.equals(LOCAL_HOST)) {
        // 2a. 本地查询
        ReadOnlyKeyValueStore<String, Long> store =
            streams.store(StoreQueryParameters.fromNameAndType(
                "count-store", QueryableStoreTypes.keyValueStore()));
        return store.get(userId);
    } else {
        // 2b. 转发到远程实例
        return restTemplate.getForObject(
            "http://{host}:{port}/user/{userId}/count",
            Long.class, hostInfo.host(), hostInfo.port(), userId);
    }
}
```

# 5 精确一次配置（EOS）

```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "eos-streams-app");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

// 启用 Exactly-Once
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG,
        StreamsConfig.EXACTLY_ONCE_V2);
// 或: EXACTLY_ONCE (旧版, 0.11+), AT_LEAST_ONCE (默认)

// 必选配置（服务端）
// Broker: transaction.state.log.replication.factor=3
// Broker: transaction.state.log.min.isr=2
// Consumer: isolation.level=read_committed (使用 enableEos 时自动设置)
```

# 6 Processor API（底层 API）

```java
public class WordCountProcessor implements Processor<String, String> {
    private ProcessorContext context;
    private KeyValueStore<String, Long> kvStore;

    @Override
    public void init(ProcessorContext context) {
        this.context = context;
        this.kvStore = context.getStateStore("Counts");
        // 定期提交（Punctuator）
        this.context.schedule(Duration.ofSeconds(1),
            PunctuationType.STREAM_TIME, this::punctuate);
    }

    @Override
    public void process(Record<String, String> record) {
        String[] words = record.value().split(" ");
        for (String word : words) {
            Long oldValue = kvStore.get(word);
            Long newValue = (oldValue == null) ? 1L : oldValue + 1;
            kvStore.put(word, newValue);
        }
    }

    private void punctuate(long timestamp) {
        try (KeyValueIterator<String, Long> iter = kvStore.all()) {
            while (iter.hasNext()) {
                KeyValue<String, Long> entry = iter.next();
                context.forward(new Record<>(entry.key, entry.value, timestamp));
            }
        }
    }

    @Override
    public void close() {}
}
```

# 7 核心配置

```java
Properties props = new Properties();

// === 必选配置 ===
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "my-streams-app");
// application.id 也是 Consumer Group ID 和内部 Topic 前缀
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "broker1:9092");

// === 并行度 ===
props.put(StreamsConfig.NUM_STREAM_THREADS_CONFIG, 3);
// 每个实例的 Stream 线程数
// 总并行度 = 实例数 × num.stream.threads

// === 状态存储 ===
props.put(StreamsConfig.STATE_DIR_CONFIG, "/var/lib/kafka-streams");
// 本地状态存储目录（RocksDB 文件）
props.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG,
        CustomRocksDBConfig.class);
// RocksDB 高级配置

// === 缓冲和缓存 ===
props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 10 * 1024 * 1024L);
// 10MB: 聚合结果的缓存（减少下游 Topic 写入）
props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 30000);
// 提交间隔: 30s（频率与处理的 at-least-once 次数相关）

// === 容错 ===
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG,
        StreamsConfig.EXACTLY_ONCE_V2);
// 精确一次语义
```

# 8 常见问题与调优

## 8.1 状态存储恢复

```
问题: 启动慢，需要从 Changelog Topic 重建 State Store
原因: 
  - 本地 RocksDB 目录丢失
  - application.id 对应的 Changelog Topic 数据量大

优化:
  ① 合理设置 Segment 大小（减少恢复时的文件数）
  ② 使用 Standby Replicas (减少故障转移时间)
    props.put(StreamsConfig.NUM_STANDBY_REPLICAS_CONFIG, 1);
  ③ 增大 num.stream.threads → 每个 Task 的 State Store 更小
```

## 8.2 反压

```
问题: 源 Topic 写入速度 > 处理速度
表现: Consumer Lag 持续增大

优化:
  ① 增加实例数（水平扩展）
  ② 增加 num.stream.threads（垂直扩展）
  ③ 增加源 Topic 分区数
  ④ 优化处理逻辑（减少 I/O, 减少序列化开销）
```

## 8.3 数据倾斜

```
问题: 某些 Partition 数据量远大于其他 → 处理不均衡

优化:
  ① 选择更合理的消息 Key（让数据更均匀分布）
  ② 增加分区数（让数据更分散）
  ③ 在聚合前做 pre-aggregation（本地预聚合）
```
