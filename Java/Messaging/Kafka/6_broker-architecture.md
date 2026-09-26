# 1 Broker 整体架构

## 1.1 Broker 内部组件拓扑

```
┌──────────────────────────────────────────────────────────────────┐
│                         Kafka Broker                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    SocketServer（网络层）                  │   │
│  │                                                          │   │
│  │   Acceptor Thread ──▶ Processor Threads (CPU 核数)       │   │
│  │       │                      │                           │   │
│  │   接受新连接              读写 NIO Selector               │   │
│  │       │                      │                           │   │
│  │       └──────────┬───────────┘                           │   │
│  │                  ▼                                       │   │
│  │          RequestChannel (RequestQueue + ResponseQueue)   │   │
│  └──────────────────┬───────────────────────────────────────┘   │
│                     │                                            │
│  ┌──────────────────▼───────────────────────────────────────┐   │
│  │                  KafkaApis（请求处理层）                   │   │
│  │                                                          │   │
│  │   RequestHandler Threads (num.io.threads)                │   │
│  │   ├── handleProduceRequest          (PRODUCE)            │   │
│  │   ├── handleFetchRequest            (FETCH)              │   │
│  │   ├── handleListOffsetsRequest      (LIST_OFFSETS)       │   │
│  │   ├── handleOffsetCommitRequest     (OFFSET_COMMIT)      │   │
│  │   ├── handleOffsetFetchRequest      (OFFSET_FETCH)       │   │
│  │   ├── handleJoinGroupRequest        (JOIN_GROUP)         │   │
│  │   ├── handleSyncGroupRequest        (SYNC_GROUP)         │   │
│  │   ├── handleHeartbeatRequest        (HEARTBEAT)          │   │
│  │   ├── handleLeaderAndIsrRequest     (LEADER_AND_ISR)     │   │
│  │   ├── handleUpdateMetadataRequest   (UPDATE_METADATA)    │   │
│  │   └── ... 50+ 种请求类型                                │   │
│  └──────────────────┬───────────────────────────────────────┘   │
│                     │                                            │
│  ┌──────────────────▼───────────────────────────────────────┐   │
│  │              核心管理组件                                  │   │
│  │                                                          │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌──────────────┐ │   │
│  │  │ReplicaManager │  │  LogManager   │  │   KafkaZkC   │ │   │
│  │  │ 副本管理器      │  │ 日志管理器    │  │ lient(ZK)    │ │   │
│  │  └───────────────┘  └───────────────┘  └──────────────┘ │   │
│  │                                                          │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌──────────────┐ │   │
│  │  │GroupCoordinator│  │Transaction    │  │  QuotaManagers│   │
│  │  │ 消费者组协调器  │  │Coordinator    │  │ 配额管理器    │ │   │
│  │  └───────────────┘  └───────────────┘  └──────────────┘ │   │
│  │                                                          │   │
│  │  ┌───────────────┐  ┌───────────────────────────────┐   │   │
│  │  │KafkaController │  │  Purgatory（延迟操作缓存）     │   │   │
│  │  │ 控制器（仅一台） │  │  DelayedProduce/DelayedFetch │   │   │
│  │  └───────────────┘  └───────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

# 2 SocketServer 网络层

## 2.1 Reactor 多线程模型

Kafka 的网络层采用 **1 Acceptor + N Processor** 的 Reactor 模型：

```
                         Acceptor Thread (1个)
                              │
                   (接受 TCP 连接，分配)
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
  Processor 0           Processor 1           Processor 2
  (NIO Selector)        (NIO Selector)        (NIO Selector)
       │                      │                      │
       │ 读取 RequestChannel   │                      │
       │   RequestQueue       │                      │
       ▼                      ▼                      ▼
  ┌───────────────────────────────────────────────────┐
  │              RequestChannel                       │
  │  RequestQueue (ArrayBlockingQueue per Processor)  │
  │  ResponseQueue (ArrayBlockingQueue per Processor) │
  └────────────────────┬──────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   RequestHandler 0  RequestHandler 1  ...  (num.io.threads)
   (KafkaApis 业务处理)
```

## 2.2 线程分工

| 线程类型 | 数量 | 职责 |
|---------|------|------|
| Acceptor | 1 | 接受新 TCP 连接，分配到 Processor |
| Processor | `num.network.threads` | NIO Selector 轮询，读写网络 I/O，将请求放入 RequestChannel |
| RequestHandler | `num.io.threads` | 从 RequestChannel 取出请求，调用 KafkaApis 处理，写回 ResponseQueue |
| KafkaScheduler | 线程池 | 后台定时任务（日志清理、ISR 检查等） |

## 2.3 Processor 核心逻辑

```scala
// Processor.run() 核心逻辑（简化）
override def run(): Unit = {
  while (isRunning) {
    // 1. 处理新连接（从 Acceptor 移交过来的）
    configureNewConnections()

    // 2. 处理 NIO Selector 事件
    selector.poll(300)

    // 3. 读取完成的请求放入 RequestChannel
    //    每个连接从 SocketChannel 读 → 反序列化 → RequestChannel.RequestQueue
    //    包含: ApiKeys (PRODUCE/FETCH/...), RequestHeader, RequestBody

    // 4. 从 ResponseChannel 取响应写回
    //    将 Response 序列化后写回 SocketChannel
  }
}
```

## 2.4 关键参数

```properties
# 网络线程数（Processor 数）
num.network.threads=3

# I/O 线程数（RequestHandler 数）
num.io.threads=8

# 每个连接的最大请求队列
queued.max.requests=500

# Socket 发送/接收缓冲区
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400

# 每个 IP 的最大连接数
max.connections.per.ip=2147483647

# 最大连接数
max.connections=2147483647
```

# 3 KafkaApis 请求路由

## 3.1 请求处理入口

```scala
// KafkaApis.handle() 是所有请求的入口
def handle(request: RequestChannel.Request): Unit = {
  try {
    request.header.apiKey match {
      case ApiKeys.PRODUCE          => handleProduceRequest(request)
      case ApiKeys.FETCH            => handleFetchRequest(request)
      case ApiKeys.LIST_OFFSETS     => handleListOffsetRequest(request)
      case ApiKeys.METADATA         => handleTopicMetadataRequest(request)
      case ApiKeys.OFFSET_COMMIT    => handleOffsetCommitRequest(request)
      case ApiKeys.OFFSET_FETCH     => handleOffsetFetchRequest(request)
      case ApiKeys.FIND_COORDINATOR => handleFindCoordinatorRequest(request)
      case ApiKeys.JOIN_GROUP       => handleJoinGroupRequest(request)
      case ApiKeys.SYNC_GROUP       => handleSyncGroupRequest(request)
      case ApiKeys.HEARTBEAT        => handleHeartbeatRequest(request)
      case ApiKeys.LEADER_AND_ISR   => handleLeaderAndIsrRequest(request)
      case ApiKeys.STOP_REPLICA     => handleStopReplicaRequest(request)
      case ApiKeys.UPDATE_METADATA  => handleUpdateMetadataRequest(request)
      case ApiKeys.INIT_PRODUCER_ID => handleInitProducerIdRequest(request)
      case ApiKeys.ADD_PARTITIONS_TO_TXN  => handleAddPartitionsToTxnRequest(request)
      case ApiKeys.ADD_OFFSETS_TO_TXN     => handleAddOffsetsToTxnRequest(request)
      case ApiKeys.END_TXN                => handleEndTxnRequest(request)
      case ApiKeys.WRITE_TXN_MARKERS      => handleWriteTxnMarkersRequest(request)
      // ... 50+ API Keys
    }
  } catch {
    case e: Throwable => requestHelper.handleError(request, e)
  }
}
```

## 3.2 PRODUCE 请求处理链路

```scala
// API Version 3+ (支持事务和精确一次)
def handleProduceRequest(request: RequestChannel.Request): Unit = {
  val produceRequest = request.body[ProduceRequest]

  // 1. 筛选出本 Broker 是 Leader 的分区
  val authorizedTopics = // ACL 校验
  val (existingAndAuthorized, nonExisting) = produceRequest.partitionRecordsOrFail.partition {
    case (tp, _) => metadataCache.contains(tp.topic) && authorizedTopics.contains(tp.topic)
  }

  // 2. 追加到 Log
  replicaManager.appendRecords(
    timeout = produceRequest.timeout.toInt,
    requiredAcks = produceRequest.acks,  // 0, 1, -1(all)
    internalTopicsAllowed = ...,
    isFromClient = true,
    entriesPerPartition = existingAndAuthorized,
    responseCallback = (status) => {
      // 3. 构建响应
      //   ├── acks=0 → 立刻响应（无等待）
      //   ├── acks=1 → Leader 写入后响应
      //   └── acks=all → 等待 ISR 全部完成（Purgatory）
    }
  )
}
```

## 3.3 FETCH 请求处理链路

```scala
def handleFetchRequest(request: RequestChannel.Request): Unit = {
  val fetchRequest = request.body[FetchRequest]

  // 1. 获取每个分区的 fetch offset 和最大字节数
  val fetchData = fetchRequest.fetchData.asScala

  // 2. 从 Log 中读取数据
  replicaManager.fetchMessages(
    timeout = fetchRequest.maxWait,
    replicaId = fetchRequest.replicaId,
    fetchMinBytes = fetchRequest.minBytes,
    fetchMaxBytes = fetchRequest.maxBytes,
    versionId = fetchRequest.versionId,
    fetchInfos = fetchData,
    responseCallback = (responseData) => {
      // 3. 构建 FetchResponse
      //   ├── 优先返回缓存数据（满足 fetchMinBytes 即返回）
      //   ├── 数据不足时进入 Purgatory 等待（最多 maxWait ms）
      //   └── 超时/有足够数据 → 返回响应
    }
  )
}
```

# 4 Purgatory（延迟操作机制）

## 4.1 设计目的

Purgatory 是 Kafka 用于处理"需要等待条件满足"的延迟操作的机制。典型场景：

| 场景 | 等待条件 | 超时 |
|------|---------|------|
| DelayedProduce (acks=all) | 所有 ISR 完成复制 | `request.timeout.ms` |
| DelayedFetch (fetch.min.bytes) | 累积到足够数据 | `fetch.max.wait.ms` |
| DelayedJoin | 所有成员完成 JoinGroup | session 超时 |
| DelayedTxnMarker | Transaction Marker 完成 | 事务超时 |

## 4.2 Purgatory 工作原理

```
┌─────────────────────────────────────────────────────┐
│                  Purgatory（炼狱）                    │
│                                                     │
│   Watcher List（监视线程）+ Timer（定时器）           │
│                                                     │
│   ① 操作到来 → 检查条件是否满足                       │
│       ├── 满足 → 立刻执行回调                         │
│       └── 不满足 → 加入 Purgatory 等待               │
│                                                     │
│   ② 监视事件发生 → 检查是否满足条件                    │
│       ├── 满足 → 从 Purgatory 移除，执行回调           │
│       └── 不满足 → 继续等待                           │
│                                                     │
│   ③ 超时 → 从 Purgatory 移除，执行超时回调             │
│                                                     │
│   ┌──────────────────────────────┐                  │
│   │  DelayedOperationPurgatory   │                  │
│   │                              │                  │
│   │  ┌────────────────────────┐  │                  │
│   │  │ Watchers (Map)         │  │                  │
│   │  │ Key → List<DelayedOp>  │  │                  │
│   │  │ "topic-partition"      │  │                  │
│   │  └────────────────────────┘  │                  │
│   │                              │                  │
│   │  ┌────────────────────────┐  │                  │
│   │  │ Timer (SystemTimer)    │  │                  │
│   │  │ 按超时时间排序           │  │                  │
│   │  └────────────────────────┘  │                  │
│   └──────────────────────────────┘                  │
└─────────────────────────────────────────────────────┘
```

## 4.3 DelayedProduce 示例

```scala
// acks=all 时，Produce 请求成为 DelayedProduce
class DelayedProduce(delayMs: Long,
                     produceMetadata: ProduceMetadata,
                     replicaManager: ReplicaManager,
                     responseCallback: Map[TopicPartition, PartitionResponse] => Unit)
  extends DelayedOperation(delayMs) {

  // 条件检查：所有 ISR 副本是否已完成复制
  override def tryComplete(): Boolean = {
    produceMetadata.partitionStatus.forall { case (tp, status) =>
      status.acksPending && // ISR 全部 ACK
      status.error == Errors.NONE
    }
  }

  // 超时处理
  override def onExpiration(): Unit = {
    produceMetadata.partitionStatus.foreach { case (tp, status) =>
      if (!status.acksPending) {
        status.error = Errors.REQUEST_TIMED_OUT
      }
    }
    responseCallback(produceMetadata.partitionStatus.asJava)
  }
}
```

# 5 Broker 启动流程

## 5.1 KafkaServer 启动时序

```
KafkaServer.startup()
│
├── ① 初始化 Metrics 指标系统
│
├── ② 初始化 ZooKeeper / KRaft 客户端
│   └── 注册 Broker ID
│
├── ③ 启动 LogManager
│   ├── 加载所有磁盘上的 Log Segment
│   ├── 启动 Log Cleaner 线程（Compact 策略）
│   ├── 启动 Background 线程（Flush / Checkpoint）
│   └── 恢复 HighWatermark (Truncate to HW)
│
├── ④ 启动 SocketServer
│   ├── 创建 Acceptor + Processor 线程
│   └── 绑定端口，开始接收连接
│
├── ⑤ 启动 ReplicaManager
│   ├── 初始化 ReplicaFetcher 线程
│   └── ISR 管理启动
│
├── ⑥ 启动 KafkaApis（RequestHandler 线程）
│
├── ⑦ 启动 KafkaController（如本机是 Controller）
│   ├── 监听 /controller ZooKeeper 节点
│   ├── 注册 Controller 选举回调
│   └── ControllerContext 初始化
│
├── ⑧ 启动 GroupCoordinator
│   ├── 加载 __consumer_offsets 中的 Group 信息
│   └── 开启 Offset 过期清理
│
├── ⑨ 启动 TransactionCoordinator
│   └── 加载 __transaction_state 中的事务状态
│
├── ⑩ 启动 AutoTopicCreationManager（如启用）
│
├── ⑪ 启动 DynamicConfigManager（动态配置监听）
│
└── ⑫ 向 ZooKeeper 注册 /brokers/ids/{brokerId}
    └── 发布 epoch, listeners, JMX port 等元数据
```

## 5.2 Broker 关闭流程

```
KafkaServer.shutdown()
│
├── ① 前导操作
│   ├── Controller Move（如本机是 Controller）
│   └── 停止接收新连接
│
├── ② 关闭 SocketServer & KafkaApis
│
├── ③ 关闭 ReplicaManager（停止 Fetcher 线程）
│
├── ④ 关闭 GroupCoordinator / TransactionCoordinator
│
├── ⑤ Flush Log 到磁盘并关闭 LogManager
│
└── ⑥ 从 ZooKeeper 注销 /brokers/ids/{brokerId}
```

# 6 日志刷盘（Flush）策略

## 6.1 Kafka 的刷盘哲学

Kafka **不**依赖 `fsync` 来保证持久性，而是依赖**多副本复制**。原因：

```
fsync 的代价：
  同步刷盘 → 每次写入都等待磁盘完成 → 吞吐急剧下降

Kafka 的替代方案：
  ① 写入 Page Cache（OS 管理，内存级别性能）
  ② 依赖多副本复制保底（另一台 Broker 有副本）
  ③ Page Cache 由 OS 后台刷盘
```

## 6.2 刷盘配置

```properties
# 每多少条消息触发 fsync（默认 Long.MAX_VALUE，即不强制刷盘）
log.flush.interval.messages=9223372036854775807

# 每多少毫秒触发 fsync（默认 Long.MAX_VALUE）
log.flush.interval.ms=9223372036854775807

# 刷盘调度间隔（检查是否需要刷盘）
log.flush.scheduler.interval.ms=9223372036854775807

# Linux 内核参数调优（推荐）
# vm.dirty_background_ratio = 5
# vm.dirty_ratio = 10
```

# 7 请求配额（Quota）管理

## 7.1 配额类型

```properties
# ① 网络带宽配额
# 限制每个 Client ID 的生产/消费带宽
# 动态配置（不需要重启）
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config producer_byte_rate=1048576 \
    --entity-type clients --entity-name clientA
# 限制 clientA 生产速率为 1MB/s

bin/kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config consumer_byte_rate=2097152 \
    --entity-type clients --entity-name clientA
# 限制 clientA 消费速率为 2MB/s

# ② 请求速率配额
# 限制每个 Client ID 或 User 的请求频率
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config request_percentage=50 \
    --entity-type users --entity-name user1
# user1 最多使用 I/O 线程和网络线程时间的 50%
```

## 7.2 令牌桶算法

Kafka 使用 **Token Bucket** 算法实现限流：

```
令牌桶: 容量=burst 值，填充速率=quota 值

正常: 桶中有足够令牌 → 请求通过
突发: 允许短期超过配额（burst），桶中有积攒的令牌
持续超配额: 令牌耗尽 → 请求被延迟（throttle_time_ms）
```

# 8 内部 Topic 一览

Kafka 使用内部 Topic 存储元数据：

| 内部 Topic | 分区数 | 用途 |
|-----------|--------|------|
| `__consumer_offsets` | 默认 50 | 存储 Consumer Group 的 Offset |
| `__transaction_state` | 默认 50 | 存储事务状态 |
| `__cluster_metadata` (KRaft) | 默认 1 | KRaft 模式下的元数据日志 |

```properties
# 内部 Topic 的副本和分区配置
offsets.topic.replication.factor=3
offsets.topic.num.partitions=50
transaction.state.log.replication.factor=3
transaction.state.log.num.partitions=50
transaction.state.log.min.isr=2
```

# 9 动态配置

Kafka Broker 支持部分配置的动态修改（无需重启）：

```bash
# 查看所有动态配置
kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type brokers --entity-name 0 --describe

# 修改某 Broker 的动态配置
kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config leader.replication.throttled.rate=104857600 \
    --entity-type brokers --entity-name 0

# 修改 Topic 的动态配置
kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config retention.ms=43200000 \
    --entity-type topics --entity-name my-topic
```

常用可动态修改的配置：

| 配置项 | 说明 |
|-------|------|
| `log.retention.ms` | 日志保留时间 |
| `min.insync.replicas` | 最小 ISR 数 |
| `unclean.leader.election.enable` | 是否允许非 ISR 副本选为 Leader |
| `compression.type` | 压缩类型 |
| `leader.replication.throttled.rate` | Leader 复制限速 |
| `follower.replication.throttled.rate` | Follower 复制限速 |
