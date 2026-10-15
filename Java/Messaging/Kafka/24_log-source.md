# 1. LogManager 日志管理器

## 1.1 初始化与加载

```scala
// LogManager.scala

class LogManager(logDirs: Seq[File],
                 initialOfflineDirs: Seq[File],
                 val topicConfigs: Map[String, LogConfig],
                 val defaultConfig: LogConfig,
                 val cleanerConfig: CleanerConfig,
                 ...) extends Logging {

  // 所有日志对象
  private val currentLogs = new Pool[TopicPartition, UnifiedLog]()
  // 磁盘检查点文件
  val recoveryPointCheckpoints: Map[File, OffsetCheckpointFile]
  val logStartOffsetCheckpoints: Map[File, OffsetCheckpointFile]

  def startup(topicNames: Set[String]): Unit = {
    // ① 创建/验证所有日志目录
    createAndValidateLogDirs(logDirs)

    // ② 扫描所有目录，加载已有 Partition 日志
    loadLogs(logDirs)

    // ③ 启动 LogCleaner 线程
    cleaner.startup()

    // ④ 启动后台线程
    if (scheduler != null) {
      scheduler.startup()
      // 定期刷盘
      scheduler.schedule("kafka-log-flusher",
        () => flushDirtyLogs(),
        config.flushIntervalMs, config.flushIntervalMs)
      // 定期写 checkpoint
      scheduler.schedule("kafka-recovery-point-checkpoint",
        () => checkpointRecoveryPointOffsets(),
        ...)
      // 定期删除过期日志
      scheduler.schedule("kafka-retention",
        () => cleanupLogs(),
        config.retentionCheckIntervalMs,
        config.retentionCheckIntervalMs)
    }
  }

  // 加载日志（恢复）
  private def loadLogs(logDirs: Seq[File]): Unit = {
    for (dir <- logDirs) {
      val recoveryPoints = recoveryPointCheckpoints(dir).read()
      val logStartOffsets = logStartOffsetCheckpoints(dir).read()

      // 遍历目录下所有 Partition 子目录
      for (logDir <- dir.listFiles if logDir.isDirectory) {
        val topicPartition = parseTopicPartitionName(logDir.getName)
        // 创建 UnifiedLog
        val log = UnifiedLog(
          dir = logDir,
          config = ...,
          logStartOffset = logStartOffsets.getOrElse(topicPartition, 0L),
          recoveryPoint = recoveryPoints.getOrElse(topicPartition, 0L),
          ...)
        currentLogs.put(topicPartition, log)
      }
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/log/LogManager.scala:69`

# 2. UnifiedLog 核心实现

## 2.1 追加日志 append()

```scala
// UnifiedLog.scala

class UnifiedLog(@volatile var dir: File,
                  @volatile var config: LogConfig,
                  @volatile var logStartOffset: Long,
                  @volatile var recoveryPoint: Long,
                  ...) extends Logging {

  val segments: LogSegments             // Segment 集合
  val leaderEpochCache: LeaderEpochCache // Leader Epoch 缓存
  val producerStateManager: ProducerStateManager // 幂等/事务状态
  var nextOffsetMetadata: LogOffsetMetadata // 下一个 offset
  var highWatermarkMetadata: LogOffsetMetadata // HW

  // ===== appendAsLeader: 作为 Leader 追加 =====
  def appendAsLeader(records: MemoryRecords,
                     leaderEpoch: Int,
                     isFromClient: Boolean = true,
                     interBrokerProtocolVersion: ApiVersion): LogAppendInfo = {
    // ① 验证记录
    validateMessagesAndAssignOffsets(records,
      offsetCounter = logEndOffset,
      now = time.milliseconds(),
      sourceCodec = ...)

    // ② 追加到活跃 Segment
    val appendInfo = append(records, isFromClient, interBrokerProtocolVersion)

    // ③ 更新 LEO
    nextOffsetMetadata = appendInfo.lastOffsetMetadata

    // ④ 如果来自 Producer 且是幂等/事务 → 更新 Producer State
    if (isFromClient) {
      producerStateManager.update(appendInfo)
    }

    appendInfo
  }

  // ===== append: 实际写入 =====
  private def append(records: MemoryRecords,
                     isFromClient: Boolean,
                     interBrokerProtocolVersion: ApiVersion): LogAppendInfo = {
    val appendInfo = analyzeAndValidateRecords(records, isFromClient)

    // ① 获取或创建活跃 Segment
    val segment = maybeRoll(appendInfo.lastOffset, appendInfo.maxTimestamp,
                            appendInfo.records.sizeInBytes, ...)

    // ② 追加记录到 Segment
    segment.append(
      appendInfo.firstOffset,
      appendInfo.maxTimestamp,
      appendInfo.maxTimestamp,  // 如果不是 AppendTime 则为 -1
      records)

    // ③ 更新索引（offset index + time index）
    val lastOffset = appendInfo.lastOffset
    segment.offsetIndex.append(lastOffset,
                                segment.size) // position in .log
    segment.timeIndex.maybeAppend(
      appendInfo.maxTimestampSoFar, lastOffset - segment.baseOffset)

    appendInfo
  }
}
```

> 文件位置: `core/src/main/scala/kafka/log/UnifiedLog.scala:737` (appendAsLeader)

## 2.2 读取日志 read()

```scala
// UnifiedLog.scala

def read(startOffset: Long,
         maxLength: Int,
         isolation: FetchIsolation,
         minOneMessage: Boolean): FetchDataInfo = {

  val endOffsetMetadata = isolation match {
    case FetchIsolation.LOG_END => localLogEndOffset
    case FetchIsolation.HIGH_WATERMARK => highWatermarkMetadata
    case FetchIsolation.TXN_COMMITTED => lastStableOffsetMetadata
  }

  // ① 定位起始 Segment
  val segment = segments.floorSegment(startOffset)
  val adjustedStartOffset = Math.max(startOffset, segment.baseOffset)

  // ② 从 Segment 读取
  val fetchInfo = segment.read(
    adjustedStartOffset,
    maxLength,
    endOffsetMetadata.messageOffset
  )

  // ③ 包装返回（包含 FetchDataInfo + HW + LSO）
  FetchDataInfo(
    endOffsetMetadata,
    fetchInfo
  )
}
```

> 文件位置: `core/src/main/scala/kafka/log/UnifiedLog.scala:1001`

# 3. LogSegment 实现

## 3.1 追加写入

```scala
// LogSegment.scala

class LogSegment(val log: FileRecords,          // .log 文件
                  val lazyOffsetIndex: LazyIndex[OffsetIndex],   // .index 文件
                  val lazyTimeIndex: LazyIndex[TimeIndex],       // .timeindex 文件
                  val baseOffset: Long,
                  ...) extends Logging {

  private var createdTime: Long = time.milliseconds()
  @volatile private var _maxTimestampSoFar: Long = ...

  // ===== append: 追加记录 =====
  def append(firstOffset: Long,
             largestOffset: Long,
             largestTimestamp: Long,
             records: MemoryRecords): Unit = {
    val physicalPosition = log.sizeInBytes() // 当前文件物理位置

    // ① 将记录追加到 .log 文件的末尾
    val appendedBytes = log.append(records)

    // ② 更新最大时间戳
    if (largestTimestamp > _maxTimestampSoFar) {
      _maxTimestampSoFar = largestTimestamp
    }

    // ③ 必要时更新 offset 索引（稀疏索引）
    if (bytesSinceLastIndexEntry > indexIntervalBytes) {
      offsetIndex.append(firstOffset, physicalPosition)
      bytesSinceLastIndexEntry = 0
    }
  }

  // ===== read: 读取记录 =====
  def read(startOffset: Long,
           maxSize: Int,
           maxOffset: Option[Long] = None): FetchDataInfo = {
    // ① 通过索引定位 startOffset 在文件中的物理位置
    val indexEntry = translateOffset(startOffset)
    //    ├── 从 offsetIndex 中找到 ≤ startOffset 的最大条目
    //    ├── indexEntry.offset → 物理位置
    //    └── 返回 (startPosition, 对应的 offset, 对应的物理位置)

    // ② 从 .log 文件的 startPosition 处开始读取
    val fetchInfo = log.read(
      startPosition = indexEntry.position,
      maxSize = maxSize,
      maxOffset = maxOffset
    )

    FetchDataInfo(
      fetchOffsetMetadata = LogOffsetMetadata(startOffset, baseOffset, indexEntry.position),
      records = fetchInfo
    )
  }

  // ===== 定位物理位置 (Offset 查找) =====
  private def translateOffset(offset: Long): OffsetPosition = {
    val mapping = offsetIndex.lookup(offset)
    // 从索引中找到 ≤ offset 的最近一个条目
    // → 返回 (position: 物理文件位置, offset: 对应的逻辑 offset)
    mapping
  }

  // ===== recover: 恢复（检查 CRC） =====
  def recover(producerStateManager: ProducerStateManager,
              leaderEpochCache: Option[LeaderEpochCache] = None): Int = {
    // ① 从文件开头扫描（或从 lastFlushedOffset 开始）
    // ② 逐条检查 CRC
    // ③ 找到最后一个 CRC 正确的消息
    // ④ 删除 CRC 错误的消息之后的数据
    val readBuffer = log.duplicate()
    var validBytes = 0
    var lastOffset = baseOffset

    while (readBuffer.position < readBuffer.limit) {
      val batch = readBuffer.batchIterator.next()
      if (batch.checksum != batch.checksum) {
        // CRC 错误 → 损坏结束
        break
      }
      validBytes = readBuffer.position
      lastOffset = batch.lastOffset
    }

    log.truncateTo(validBytes) // 截断损坏部分
    validBytes
  }
}
```

> 文件位置: `core/src/main/scala/kafka/log/LogSegment.scala:148` (append)

## 3.2 OffsetIndex（偏移量索引）

```scala
// OffsetIndex.scala

class OffsetIndex(@volatile var file: File,
                   val baseOffset: Long,
                   val maxIndexSize: Int = -1,
                   val writable: Boolean = true) extends AbstractIndex {

  // 每个索引项: RelativeOffset(4B) + Position(4B) = 8 bytes
  val entrySize = 8

  // ===== 追加索引项 =====
  def append(offset: Long, position: Int): Unit = {
    val relativeOffset = relativeOffset(offset) // offset - baseOffset
    mmap.putInt(relativeOffset.toInt)
    mmap.putInt(position)
    incrementEntries()
  }

  // ===== 查找 =====
  def lookup(targetOffset: Long): OffsetPosition = {
    // ① 二分查找 mmap（内存映射文件）
    val idx = indexSlotRangeFor(targetOffset,
      IndexSearchType.KEY) // 找 ≤ targetOffset 的最大条目

    // ② 从索引中获取
    val relativeOffset = relativeOffset(mmap, idx)
    val position = physical(mmap, idx)

    OffsetPosition(baseOffset + relativeOffset, position)
  }
}
```

> 文件位置: `core/src/main/scala/kafka/log/OffsetIndex.scala:63`

# 4. LogCleaner 日志清理器

## 4.1 Compact 清理流程

```scala
// LogCleaner.scala

class LogCleaner(initialConfig: CleanerConfig,
                  logDirs: Seq[File],
                  logs: Pool[TopicPartition, UnifiedLog],
                  time: Time) extends Logging {

  private val cleaners = (0 until config.numThreads).map { i =>
    new CleanerThread(i)
  }

  def startup(): Unit = {
    cleaners.foreach(_.start())
  }

  // CleanerThread
  class CleanerThread(threadId: Int) extends Thread {
    override def run(): Unit = {
      while (true) {
        // ① 选择脏率最高的 Log
        val cleanable = grabFilthiestCompactedLog(time)
        if (cleanable == null) {
          Thread.sleep(config.backoffMs)
          break
        }

        // ② 对 Log 执行清理
        cleanLog(cleanable.log)
        // ③ 删除旧 Segment
      }
    }
  }

  // ===== 清理单个 Log =====
  private def cleanLog(log: UnifiedLog): Unit = {
    // ① 获取可清理的 Segment 列表（跳过活跃 Segment）
    val segments = log.cleanableSegments

    // ② 读取所有 Key，记录每个 Key 的最新 Offset
    val offsetMap = groupSegmentsByKeyAndFindMaxTimestamp(segments)

    // ③ 基于 offsetMap 过滤旧值
    //    Key=user1: offsets=[100,200,300] → 只保留 300
    //    Key=null (tombstone): 如果超过 deleteRetentionMs → 记录 tombstone 删除

    // ④ 写入新的干净 Segment
    val cleaned = cleanSegments(log, segments, offsetMap)

    // ⑤ 原子替换
    log.replaceSegments(cleaned, segments)
  }
}
```

> 文件位置: `core/src/main/scala/kafka/log/LogCleaner.scala:91`

# 5. LogSegments 分段管理

```scala
// LogSegments.scala

class LogSegments(val baseDir: File) {
  // 使用 ConcurrentSkipListMap 存储所有 Segment（按 baseOffset 排序）
  private val segments = new ConcurrentSkipListMap[Long, LogSegment]()

  // 查看活跃 Segment
  def activeSegment: LogSegment = segments.lastEntry().getValue

  // 根据 offset 查找对应 Segment
  def floorSegment(offset: Long): LogSegment = {
    val entry = segments.floorEntry(offset)
    if (entry != null) entry.getValue
    else segments.firstEntry().getValue
  }

  // 添加新 Segment
  def add(segment: LogSegment): Unit = {
    segments.put(segment.baseOffset, segment)
  }

  // Delete 策略: 删除过期 Segment 集合
  def deletableSegments(predicate: (LogSegment, Option[LogSegment]) => Boolean): Iterable[LogSegment] = {
    // 从最老的 Segment 开始检查
    // 满足条件: (segment + 下一个segment) 组合通过 predicate 检查
    // 但不能删除活跃 Segment
  }
}
```

> 文件位置: `core/src/main/scala/kafka/log/LogSegments.scala:30`

# 6. 日志恢复流程

## 6.1 Broker 启动恢复

```scala
// UnifiedLog.scala → 构造器中的恢复逻辑

private def setup(): Unit = {
  // ① 扫描 Segment 目录 → 加载所有 .log 文件
  val segmentFiles = dir.listFiles.sortBy(_.getName)
  for (file <- segmentFiles if file.getName.endsWith(".log")) {
    val baseOffset = file.getName.replace(".log", "").toLong
    // 创建 LogSegment
    val segment = LogSegment.open(dir, baseOffset, config, ...)
    segments.add(segment)
  }

  // ② 恢复 Segment（截断损坏部分）
  recoverLog()

  // ③ 恢复 Producer State（幂等/事务）
  producerStateManager.truncateAndReload(...)

  // ④ 设置 LEO 和 HW
  nextOffsetMetadata = ...
  highWatermarkMetadata = ...
}

private def recoverLog(): Unit = {
  // ① 找到最后一个非空的 Segment
  // ② 对其执行 segment.recover()
  //    → 从 lastStableOffset 开始扫描
  //    → 找到最后一个 CRC 正确的消息
  //    → 截断之后的数据
  // ③ 更新 offset 索引
  // ④ 设置 LEO = 最后一条消息 offset + 1
}
```

# 7. 内存映射（MappedByteBuffer）

```scala
// AbstractIndex.scala (OffsetIndex / TimeIndex 的父类)

abstract class AbstractIndex(@volatile var file: File,
                              val baseOffset: Long,
                              val maxIndexSize: Int = -1,
                              val writable: Boolean = true) extends Closeable {

  protected var mmap: MappedByteBuffer
  protected var maxEntries: Int
  protected var entries: Int

  def reset(): Unit = {
    // 删除旧 mmap
    mmap.force()
    mmap = null

    // 创建新 mmap
    mmap = {
      val raf = new RandomAccessFile(file, "rw")
      raf.setLength(roundDownToExactMultiple(fileLength, entrySize))
      val channel = raf.getChannel
      channel.map(FileChannel.MapMode.READ_WRITE, 0, channel.size())
    }
  }
}
```

# 8. Log 恢复与副本截断

```scala
// Partition.scala → 副本同步后的截断

def truncateTo(targetOffset: Long): Future[ReplicaOperationResult] = {
  // ① 根据 Leader Epoch 确定截断位置
  val leaderEpochTruncation = leaderEpochCache.leaderEpochTruncation(targetOffset)

  // ② 截断 Log
  log.truncateTo(targetOffset)
  //    → 找到 ≤ targetOffset 的最大 Segment
  //    → 删除之后的所有 Segment
  //    → 将最后一个 Segment 截断到 targetOffset 位置
}
```

> 文件位置: `core/src/main/scala/kafka/cluster/Partition.scala:432`
