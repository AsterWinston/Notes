# 1 ReplicaManager 副本管理核心

## 1.1 架构总览

```scala
ReplicaManager
│
├── Pool[TopicPartition, Partition]       // 所有本地 Partition 副本
│
├── ReplicaFetcherManager                 // Follower 拉取管理
│   └── ReplicaFetcherThread (多个线程)   // 每个源 Broker 一个线程
│       └── 向 Leader Broker 发送 FETCH 请求
│           └── 处理 FETCH 响应 → 将数据追加到本地日志
│
├── DelayedProducePurgatory               // Produce 延迟操作 (acks=all)
├── DelayedFetchPurgatory                 // Fetch 延迟操作 (fetch.min.bytes)
│
└── ISR 管理
    ├── isrChangeSet                      // ISR 变更集合
    └── Scheduler (定期 ISR 过期检查)
```

# 2 Partition 副本对象

## 2.1 核心结构

```scala
// Partition.scala

class Partition(val topicPartition: TopicPartition,
                 val replicaLagTimeMaxMs: Long,
                 interBrokerProtocolVersion: ApiVersion,
                 localBrokerId: Int,
                 time: Time,
                 stateStore: PartitionStateStore,
                 delayedOperations: DelayedOperations,
                 metadataCache: MetadataCache,
                 val logManager: LogManager) extends Logging {

  // 分区状态（Leader / Follower / Offline）
  private var leaderEpochStartOffsetOpt: Option[Long] = None
  @volatile var leaderEpoch: Int = -1
  @volatile var partitionEpoch: Int = -1
  @volatile var isrState: IsrChangeListener = CommittedIsr
  @volatile private var log: Option[UnifiedLog] = None
  @volatile var leaderLogReadResult: LogReadResult = LogReadResult.UnknownLogReadResult

  // 远程副本状态
  val remoteReplicasMap = new Pool[Int, Replica]  // Broker ID → Replica

  // ===== 创建 Leader Partition =====
  def createLeader(isr: List[Int],
                    isNew: Boolean,
                    leaderEpoch: Int,
                    highWatermarkCheckpoints: OffsetCheckpointFile): Boolean = {
    // ① 获取 Log
    val log = logManager.getOrCreateLog(topicPartition, isNew)

    // ② 构建 ISR 集合（所有副本）
    val currentIsr = (isr :+ localBrokerId).distinct
    // 创建远程 Replica 对象
    isr.foreach { brokerId =>
      if (brokerId != localBrokerId) {
        val replica = new Replica(brokerId, topicPartition)
        remoteReplicasMap.put(brokerId, replica)
      }
    }

    // ③ 标记当前 Broker 为 Leader
    leaderEpoch = leaderEpoch
    isrState = CommittedIsr

    // ④ 推进 HW → 读取 checkpoint 中的恢复点
    log.initializeHighWatermark(...)

    true
  }

  // ===== 创建 Follower Partition =====
  def createFollower(isNew: Boolean,
                      leaderId: Int,
                      leaderEpoch: Int,
                      highWatermarkCheckpoints: OffsetCheckpointFile): Boolean = {
    // ① 获取 Log
    val log = logManager.getOrCreateLog(topicPartition, isNew)

    // ② 标记为 Follower（不维护远程副本）
    leaderEpoch = leaderEpoch

    // ③ 记录 Leader 的 Broker ID → 用于 Fetcher 线程定位
    //    ReplicaFetcherManager 据此知道从哪个 Broker 拉取
    true
  }
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Partition.scala:149`

## 2.2 追加记录（双路径）

```scala
// Partition.scala

def appendRecordsToFollowerOrLeader(records: MemoryRecords,
                                     isFromClient: Boolean,
                                     requiredAcks: Short): LogAppendInfo = {
  if (isLeader) {
    // Leader 路径: 写入本地并等待 ISR 同步
    appendRecordsToLeader(records, isFromClient, requiredAcks)
  } else {
    // Follower 路径: 仅写入本地，不等待 ISR
    //   这种场景发生在 Follower 收到 FetchResponse 后
    //   或旧 Leader 降级后仍在写入时（将被 Controller 纠正）
    appendRecordsToFollower(records, isFromClient)
  }
}

private def appendRecordsToLeader(records: MemoryRecords,
                                   isFromClient: Boolean,
                                   requiredAcks: Short): LogAppendInfo = {
  val appendInfo = log.get.appendAsLeader(records,
    leaderEpoch = this.leaderEpoch,
    isFromClient)

  // 推进 HW
  maybeIncrementLeaderHW(appendInfo.lastOffset)

  appendInfo
}

private def appendRecordsToFollower(records: MemoryRecords,
                                     isFromClient: Boolean): LogAppendInfo = {
  val appendInfo = log.get.appendAsFollower(records)

  // Follower 也需要推进 HW（从 FetchResponse 中的 HW 信息获取）
  appendInfo
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Partition.scala:427`

# 3 Leader 同步流程

## 3.1 ReplicaFetcherManager 初始化

```scala
// ReplicaManager.scala

class ReplicaManager(...) {
  // Follower 拉取管理
  val replicaFetcherManager = createReplicaFetcherManager(
    metrics, time, threadNamePrefix, replicationQuota)

  protected def createReplicaFetcherManager(...): ReplicaFetcherManager = {
    new ReplicaFetcherManager(config, stats, time, threadNamePrefix, quotaManager)
  }
}

// ReplicaFetcherManager.scala

class ReplicaFetcherManager(brokerConfig: KafkaConfig,
                             replicaManager: ReplicaManager,
                             metrics: Metrics,
                             time: Time,
                             threadNamePrefix: Option[String],
                             quotaManager: ReplicationQuotaManager)
  extends AbstractFetcherManager {

  // 为每个源 Broker 创建 Fetcher 线程
  override def createFetcherThread(fetcherId: Int,
                                    sourceBroker: BrokerEndPoint): AbstractFetcherThread = {
    val prefix = threadNamePrefix.map(tp => s"$tp:").getOrElse("")
    new ReplicaFetcherThread(
      s"${prefix}ReplicaFetcherThread-$fetcherId-${sourceBroker.id}",
      sourceBroker,
      brokerConfig,
      failedPartitions,
      replicaManager,
      metrics,
      time,
      quotaManager)
  }
}
```

> 文件位置: `core/src/main/scala/kafka/server/ReplicaFetcherManager.scala:42`

## 3.2 ReplicaFetcherThread 核心循环

```scala
// ReplicaFetcherThread.scala

class ReplicaFetcherThread(name: String,
                            sourceBroker: BrokerEndPoint,
                            brokerConfig: KafkaConfig,
                            failedPartitions: FailedPartitions,
                            replicaMgr: ReplicaManager,
                            metrics: Metrics,
                            time: Time,
                            quotaManager: ReplicationQuotaManager)
  extends AbstractFetcherThread(name, clientId, sourceBroker, failedPartitions, ...) {

  // ===== 构建 FETCH 请求 =====
  override def buildFetchRequest(
      partitionMap: Map[TopicPartition, PartitionFetchState]): ResultWithPartitions[FetchRequest.Builder] = {

    // ① 收集所有需要从该 Leader 拉取的 Partition
    val fetchData = partitionMap.map { case (tp, fetchState) =>
      tp -> new FetchRequest.PartitionData(
        fetchState.fetchOffset,      // ← 从 Follower 的 LEO 开始拉取
        fetchState.currentLeaderEpoch,
        fetchState.maxBytes
      )
    }

    // ② 构建 FetchRequest
    val requestBuilder = FetchRequest.Builder
      .forReplica(
        ApiKeys.FETCH.latestVersion(),
        replicaId = brokerConfig.brokerId,
        maxWait = replicaFetchWaitMaxMs,  // replica.fetch.wait.max.ms
        minBytes = replicaFetchMinBytes,   // replica.fetch.min.bytes
        maxBytes = replicaFetchMaxBytes,   // replica.fetch.max.bytes
        fetchData.toMap
      )
      .setMaxBytes(replicaFetchResponseMaxBytes)

    ResultWithPartitions(requestBuilder, Set.empty)
  }

  // ===== 处理 FETCH 响应 =====
  override def processPartitionData(
      topicPartition: TopicPartition,
      fetchOffset: Long,
      partitionData: FetchPartitionData): Option[LogAppendInfo] = {

    val partition = replicaMgr.getPartitionOrException(topicPartition)
    val records = partitionData.records.asInstanceOf[MemoryRecords]

    // ① 追加到本地日志
    val appendInfo = partition.appendRecordsToFollowerOrLeader(
      records,
      isFromClient = false, // ← 来自 Follower 而不是 Producer
      requiredAcks = 0      // ← Follower 不等待 ISR
    )

    // ② 返回追加信息（包含新的 LEO）
    Some(appendInfo)
  }

  // ===== 构建新 fetchOffset（推送到下一轮拉取）=====
  override def buildFetchOffset(partition: TopicPartition,
                                 currentFetchState: PartitionFetchState,
                                 appendInfo: LogAppendInfo): PartitionFetchState = {
    // 将 fetchOffset 更新为 appendInfo.lastOffset + 1
    PartitionFetchState(
      topicId = ...,
      fetchOffset = appendInfo.lastOffset + 1,
      currentLeaderEpoch = ...,
      delay = ...,
      state = Truncating
    )
  }
}
```

> 文件位置: `core/src/main/scala/kafka/server/ReplicaFetcherThread.scala:73`

# 4 HW 推进机制

## 4.1 Leader 端 HW 计算

```scala
// Partition.scala

def maybeIncrementLeaderHW(newOffsets: LogAppendInfo): Unit = {
  val logEndOffset = newOffsets.lastOffset
  // ① 收集 ISR 中所有副本的 LEO
  val allReplicaLEOs = {
    remoteReplicasMap.values.map(_.logEndOffset) :+ logEndOffset
  }

  // ② HW = ISR 中最小的 LEO
  val newHighWatermark = allReplicaLEOs.min

  // ③ 推进 HW
  if (newHighWatermark > highWatermark) {
    log.get.updateHighWatermark(newHighWatermark)
    highWatermark = newHighWatermark
  }
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Partition.scala:578`

## 4.2 Follower 端 HW 更新

```scala
// Follower 在收到 FetchResponse 时更新 HW

def updateHighWatermark(followerFetchOffsetMetadata: LogOffsetMetadata,
                         followerStartOffset: LogOffsetMetadata): Unit = {
  // ① Follower 在 FetchResponse 中得到 Leader 当前的 HW
  //    将其传给 followerFetchOffsetMetadata

  // ② HW = min(Leader 返回的 HW, Follower 本地的 LEO)
  val newHighWatermark = Math.min(followerFetchOffsetMetadata.messageOffset,
                                   logEndOffset)

  // ③ 更新本地 HW
  log.get.updateHighWatermark(newHighWatermark)
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Partition.scala:613`

# 5 ISR 管理

## 5.1 ISR 过期检查

```scala
// ReplicaManager.scala

def maybeShrinkIsr(replicaMaxLagTimeMs: Long): Unit = {
  allPartitions.foreach { case (tp, partition) =>
    if (partition.isLeader) {
      partition.maybeShrinkIsr(replicaMaxLagTimeMs)
    }
  }
}

// Partition.scala

def maybeShrinkIsr(replicaMaxLagTimeMs: Long): Unit = {
  val leaderLogEndOffset = localLog.get.logEndOffset

  for (replica <- remoteReplicasMap.values) {
    // 判断 Follower 是否落后
    val isFollowerOutOfSync =
      (leaderLogEndOffset > replica.logEndOffset) &&   // Follower LEO < Leader LEO
      (time.milliseconds() - replica.lastCaughtUpTimeMs > replicaMaxLagTimeMs)
    // Follower 落后于 Leader 且超过 replicaMaxLagTimeMs

    if (isFollowerOutOfSync) {
      // ISR 收缩
      shrinkIsr(replica.brokerId)
    } else if (replica.logEndOffset >= leaderLogEndOffset) {
      // Follower 追上 → 更新 lastCaughtUpTime
      replica.updateLastCaughtUpTimeMs(time.milliseconds())
      // ISR 扩展
      maybeExpandIsr(replica.brokerId)
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Partition.scala:712`

## 5.2 ISR 变更传播

```scala
// 当 ISR 发生变化时的传播链:

Partition.maybeShrinkIsr / maybeExpandIsr
  │
  ├── ① 更新本地 ISR 集合
  │
  ├── ② 写入 ZK /brokers/topics/{topic}/partitions/{partition}/state
  │     或 KRaft 元数据日志
  │
  ├── ③ Controller 通过 ZK Watcher 感知 ISR 变更
  │     → 更新 MetadataCache
  │
  └── ④ Controller 发送 UpdateMetadata 请求到所有 Broker
        → 各 Broker 更新本地 MetadataCache
        → Producer/Consumer 的 Metadata Refresh 获知新 ISR
```

# 6 Follower 的异常恢复

## 6.1 Truncate 逻辑（Follower 截断）

```scala
// Partition.scala → Follower 收到 LeaderAndIsr 后的截断

def maybeReplaceCurrentWithFutureReplica(): Boolean = {
  // ① 检查 Follower 的 LEO 是否大于 Leader 的 LEO
  if (log.get.logEndOffset > leaderLogEndOffset) {
    // ② Follower 比 Leader 数据多 → 需要截断
    val truncationOffset = leaderEpochCache.leaderEpochTruncation(
      leaderLogEndOffset, leaderEpoch)

    // ③ 安全截断（基于 Leader Epoch）
    log.get.truncateTo(truncationOffset)
  }
  // ④ 否则直接追赶
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Partition.scala:478`

## 6.2 Replica 状态管理

```scala
// Replica.scala

class Replica(val brokerId: Int,
               val topicPartition: TopicPartition) {

  @volatile var logEndOffset: Long = LogOffsetMetadata.UnknownOffset
  @volatile var lastCaughtUpTimeMs: Long = 0L
  @volatile var lastFetchTimeMs: Long = 0L
  @volatile var highWatermark: LogOffsetMetadata = LogOffsetMetadata.UnknownOffsetMetadata
  @volatile var leaderEpoch: Int = -1
  @volatile var isSyncReplica: Boolean = false

  def updateLogEndOffset(newLogEndOffset: Long): Unit = {
    if (newLogEndOffset > logEndOffset) {
      logEndOffset = newLogEndOffset
      if (isSyncReplica) {
        lastCaughtUpTimeMs = time.milliseconds()
      }
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Replica.scala:30`

# 7 复制相关的时序

```
完整 Leader → Follower 同步时序:

Leader (Broker 0)                  Follower (Broker 1)
     │                                    │
     │  Producer 发送 msg (.send)          │
     │ ─────────────────────────────────▶ │
     │                                    │
     │ ① appendAsLeader(records)           │
     │    → LEO: 100 → 101                │
     │    → HW: 99 (未推进)               │
     │                                    │
     │ ② acks=all → DelayedProduce        │
     │   (在 Purgatory 中等待所有 ISR)      │
     │                                    │
     │             ③ ReplicaFetcherThread   │
     │                sendFetchRequests()  │
     │  ◀───────── FetchRequest ─────────│
     │                (fetch_offset=100)   │
     │                                    │
     │ ④ 从 Log 读取数据                   │
     │    read(fetchOffset=100             │
     │         maxBytes=N)                │
     │                                    │
     │ ───────── FetchResponse ─────────▶ │
     │    data: [msg@100],               │
     │    highWatermark: 99 (旧)          │
     │                                    │
     │                              ⑤ appendAsFollower
     │                                 → LEO: 100 → 101
     │                                    │
     │             ⑥ 下一轮 FetchRequest ──│
     │                (fetch_offset=101)   │
     │                (broker1 通知 broker0│
     │                 自己的 LEO=101)      │
     │                                    │
     │ ⑦ 更新 remote replica LEO=101       │
     │    重新计算 HW:                     │
     │    HW = min(Leader LEO=101,        │
     │              Follower LEO=101)     │
     │      = 101                          │
     │    → Purgatory 中的 DelayedProduce  │
     │      条件满足 → 完成               │
     │                                    │
     │ ── ProduceResponse(ACK) ──▶ Producer│
     │                                    │
     │             ⑧ 下一轮 FetchResponse  │
     │                highWatermark: 101   │
     │ ────────────────────────────────▶  │
     │                             ⑨ Follower│
     │                                更新 HW=101 │
     │                                (通过 LeaderEpochCache │
     │                                 防护数据安全) │
```

# 8 复制相关关键参数源码映射

```scala
// KafkaConfig.scala

val ReplicaLagTimeMaxMs = 30000          // replica.lag.time.max.ms
val ReplicaFetchMinBytes = 1             // replica.fetch.min.bytes
val ReplicaFetchMaxBytes = 1048576       // replica.fetch.max.bytes
val ReplicaFetchWaitMaxMs = 500          // replica.fetch.wait.max.ms
val ReplicaFetchBackoffMs = 1000         // replica.fetch.backoff.ms
val NumReplicaFetchers = 1               // num.replica.fetchers
val ReplicaHighWatermarkCheckpointIntervalMs = 5000  // hw 检查间隔
val ReplicaSocketTimeoutMs = 30000       // replica.socket.timeout.ms
val ReplicaSocketReceiveBufferBytes = 65536
```
