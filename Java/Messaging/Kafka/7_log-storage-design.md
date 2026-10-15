# 1. Kafka 日志存储核心概念

## 1.1 从 Topic 到文件

```
Topic "orders" (3 Partitions, 2 Replicas)
│
├── Partition 0 (Broker 1 = Leader, Broker 2 = Follower)
│   └── /data/kafka-logs/orders-0/
│       ├── 00000000000000000000.log       ← 数据文件（第一个 Segment）
│       ├── 00000000000000000000.index     ← 偏移量索引
│       ├── 00000000000000000000.timeindex ← 时间戳索引
│       ├── 00000000000000000536.log       ← 数据文件（第二个 Segment）
│       ├── 00000000000000000536.index
│       ├── 00000000000000000536.timeindex
│       ├── 00000000000000001024.log       ← 活跃 Segment (Active)
│       ├── 00000000000000001024.index
│       ├── 00000000000000001024.timeindex
│       ├── leader-epoch-checkpoint
│       └── partition.metadata
│
├── Partition 1
│   └── /data/kafka-logs/orders-1/...
│
└── Partition 2
    └── /data/kafka-logs/orders-2/...
```

# 2. Segment（日志分段）

## 2.1 Segment 命名合约

每个 Segment 由三个文件组成，以第一条消息的 offset（20 位数字，不足左侧补零）命名：

| 文件 | 格式 | 内容 |
|------|------|------|
| `.log` | 消息数据文件 | 实际消息内容（二进制） |
| `.index` | 偏移量索引 | Offset → Position 映射（稀疏索引） |
| `.timeindex` | 时间戳索引 | Timestamp → Offset 映射 |

## 2.2 .log 文件内部结构

```
00000000000000001024.log
│
├── Offset=1024, MsgSize=N, RecordBatch
│   ├── BaseOffset: 1024
│   ├── BatchLength: N bytes
│   ├── PartitionLeaderEpoch: 5
│   ├── Magic: 2  (Kafka 0.11+)
│   ├── CRC: 0xABCD1234
│   ├── Attributes: Compression, TimestampType, etc.
│   ├── LastOffsetDelta: 9    (Batch 内含 10 条消息)
│   ├── BaseTimestamp: 1700000000000
│   ├── MaxTimestamp: 1700000000123
│   ├── Producer ID: 12345
│   ├── Producer Epoch: 1
│   ├── BaseSequence: 0
│   ├── Records: [Msg0, Msg1, ..., Msg9]
│   │   ├── Record 0: Length, Attributes, TimestampDelta, OffsetDelta, Key, Value, Headers
│   │   ├── Record 1: ...
│   │   └── Record 9: ...
│   └── ─────────────────────────
│
├── Offset=1034, RecordBatch ...
│
└── ...
```

## 2.3 Message Format 演进

| 版本 (Magic) | Kafka 版本 | 主要变化 |
|-------------|-----------|---------|
| 0 | < 0.10 | 原始消息格式，无 timestamp |
| 1 | 0.10 ~ 0.11 | 添加 Timestamp，引入 Wrapper 格式 |
| 2 | 0.11+ | RecordBatch，支持幂等、事务、压缩改进、Headers |

Magic 2 的 RecordBatch 关键变化：
- 从单条消息 → 批量消息（Batch）
- BaseOffset + OffsetDelta 替代绝对 Offset
- BaseTimestamp + TimestampDelta 替代绝对 Timestamp
- 支持压缩（整个 Batch 一起压缩）

# 3. 索引文件详解

## 3.1 稀疏索引设计

Kafka 使用**稀疏索引**而非密集索引，每隔一定字节数才记录一个索引项。

```
.index 文件结构:
每个条目 8 bytes = RelativeOffset(4B) + Position(4B)

RelativeOffset = Offset - BaseOffset
Position = 消息在 .log 文件中的物理位置

示例:
BaseOffset=1024, .log中位置起始为0

index entry 0: RelativeOffset=0,   Position=0    → Offset=1024 在文件位置 0
index entry 1: RelativeOffset=10,  Position=784  → Offset=1034 在文件位置 784
index entry 2: RelativeOffset=20,  Position=1526 → Offset=1044 在文件位置 1526
...
```

索引间隔由 `log.index.interval.bytes` 控制（默认 4096，即每 4KB 数据增加一个索引项）。

## 3.2 Offset 查找流程

```
查找 offset=1037 的消息:
│
├── ① 二分查找 Segment 列表
│   找到 BaseOffset=1024 ≤ 1037 < BaseOffset=1073 → 定位到 Segment 1024
│
├── ② 二分查找 .index
│   index entry 0: RelativeOffset=0,  Position=0    (≤6, 候选)
│   index entry 1: RelativeOffset=10, Position=784  (>6, 停止)
│
│   找到候选: RelativeOffset=0, Position=0
│   (1037 的相对偏移量 = 1037 - 1024 = 13, 
│    最接近且 ≤13 的索引项是 RelativeOffset=0)
│
├── ③ 从 .log 文件的 Position=0 处顺序扫描
│   读取每条 RecordBatch 的 BaseOffset + LastOffsetDelta
│   直到找到 offset=1037
│
└── ④ 返回消息
```

时间复杂度：
- O(log N) 定位 Segment
- O(log M) 定位 .index 条目
- O(K) 顺序扫描（K 为 index.interval.bytes 内的消息数）

## 3.3 .timeindex 时间戳索引

```
.timeindex 文件结构:
每个条目 12 bytes = Timestamp(8B) + RelativeOffset(4B)

示例:
entry 0: Timestamp=1700000000000, RelativeOffset=0
entry 1: Timestamp=1700000005000, RelativeOffset=10
entry 2: Timestamp=1700000010000, RelativeOffset=20

查找 timestamp=1700000007000:
→ 二分查找 .timeindex → 找到 entry 1 (Timestamp=1700000005000, RelativeOffset=10)
→ 实际 Offset = 1024 + 10 = 1034
→ 再通过 Offset 查找消息
```

# 4. 日志清理策略

## 4.1 清理策略对比

| 策略 | 参数值 | 原理 | 适用场景 |
|------|--------|------|----------|
| Delete（删除） | `delete` | 超过保留时间或大小后删除 Segment | 日志、监控数据 |
| Compact（压缩） | `compact` | 保留每 Key 的最新值，删除旧值 | 缓存、快照、CDC |
| 两者组合 | `compact,delete` | 先 Compact 后 Delete | 需要长期 Key 保留 + 定期清理旧数据 |

## 4.2 Delete 策略

```properties
# Topic 级配置
cleanup.policy=delete

# 保留时间（时间、大小、同时满足任一即可触发删除）
retention.ms=604800000       # 7天（默认）
retention.bytes=-1           # 不限制（按大小）
# 注: retention.ms = max(retention.ms, min.compaction.lag.ms)

# Segment 级配置
segment.ms=604800000         # Segment 最大存活时间
segment.bytes=1073741824     # Segment 最大大小 (1GB, 默认)
```

**Delete 触发条件（满足任一）：**

1. 当前时间 - Segment 最大时间戳 > `retention.ms`
2. Topic 总大小 > `retention.bytes`（从最旧 Segment 开始删）

## 4.3 Compact 策略

```
压缩前:
Key="user_001" → Value="Alice"
Key="user_002" → Value="Bob"
Key="user_001" → Value="Alice_v2"   ← 旧值被压缩
Key="user_003" → Value="Charlie"
Key="user_001" → Value="Alice_v3"   ← 旧值被压缩

压缩后 (Compacted Topic):
Key="user_001" → Value="Alice_v3"   ← 只保留最新
Key="user_002" → Value="Bob"
Key="user_003" → Value="Charlie"

(Key 为 null 的消息 → tombstone → 表示删除该 Key)
```

### 4.3.1 Compact 机制

```
┌─────────────────────────────────────────────────┐
│              Log Compaction 流程                  │
│                                                  │
│   Log                                          │
│   ┌─── Segment (Active) ──────────┐              │
│   │ [...新消息追加中...]          │              │
│   └───────────────────────────────┘              │
│                                      (不被 Clean)│
│   ┌─── Segment (非活跃) ──────────┐              │
│   │ [旧消息，等待压缩]            │     ← 可压缩  │
│   └───────────────────────────────┘              │
│   ┌─── Segment (未清理) ──────────┐              │
│   │ [脏 Segment，有待压缩的陈旧值] │ ← 待处理    │
│   └───────────────────────────────┘              │
│   ┌─── Segment (已清理) ──────────┐              │
│   │ [干净 Segment，仅保留最新值]    │ ← 已完成    │
│   └───────────────────────────────┘              │
│                                                  │
│   Log Cleaner 线程池 (log.cleaner.threads=1)     │
│   ┌──────────────────────────────────────────┐   │
│   │ ① 选脏率最高的 Log                         │   │
│   │ ② 读取 Key，映射到最新 Offset               │   │
│   │ ③ 写入新 Segment（仅含最新值）              │   │
│   │ ④ 原子替换旧 Segment                       │   │
│   └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 4.3.2 Tombstone 消息（墓碑消息）

```java
// 标记 Key 的删除
ProducerRecord<String, byte[]> tombstone = new ProducerRecord<>(
    "compacted-topic",
    "key-to-delete",
    null  // value = null → 墓碑
);
producer.send(tombstone);
```

墓碑消息被保留 `delete.retention.ms`（默认 24h）后才真正被删除，给下游 Consumer 足够时间感知。

## 4.4 Cleaner 相关参数

```properties
# Log Cleaner 启用（默认 true）
log.cleaner.enable=true

# Cleaner 线程数（每个 Broker）
log.cleaner.threads=1

# 每次清理的最大字节数（默认不限制）
log.cleaner.max.compaction.lag.ms=9223372036854775807

# Tombstone 保留时间（墓碑消息的存活时间）
delete.retention.ms=86400000   # 24h

# 脏率阈值（超过才触发清理）
min.cleanable.dirty.ratio=0.5

# 清理间隔
log.cleaner.backoff.ms=15000
```

# 5. 零拷贝（Zero-Copy）

## 5.1 传统 I/O vs sendfile

```
传统 I/O（4 次拷贝 + 4 次上下文切换）:
┌─────────┐  copy 1   ┌─────────┐  copy 2   ┌─────────┐
│  Disk   │ ────────▶ │ Kernel  │ ────────▶ │  User   │
│         │  (DMA)    │ Buffer  │  (CPU)    │  Buffer  │
└─────────┘           └─────────┘           └────┬────┘
                                                  │
                        copy 4        copy 3      ▼
                      ┌─────────┐   ┌─────────┐
         NIC  ◀────── │ Socket  │ ◀─│   App   │
                      │ Buffer  │   │         │
                      └─────────┘   └─────────┘

Zero-Copy sendfile()（2 次拷贝 + 2 次上下文切换）:
┌─────────┐  copy 1   ┌─────────┐
│  Disk   │ ────────▶ │ Kernel  │
│         │  (DMA)    │ Buffer  │ ──┐
└─────────┘           └─────────┘   │
                                    │ copy 2 (DMA, scatter-gather)
                                    ▼
                              ┌─────────┐
                              │   NIC   │
                              └─────────┘
```

## 5.2 Kafka 中的零拷贝

```java
// kafka.server.KafkaApis → FileRecords → sendfile

// Fetch 请求处理 → 从磁盘读取数据 → 通过 sendfile 直接传输给网卡
FileRecords.read(file, position)  // 映射到 FileChannel
  → transferTo(socketChannel, position, count)  // 内核态零拷贝
    → sendfile() / splice()
```

```java
// Java 中的零拷贝 API（Kafka 使用 FileChannel.transferTo）
FileChannel fileChannel = FileChannel.open(Paths.get("file.log"));
SocketChannel socketChannel = SocketChannel.open(); // 已连接到 client

// 直接从 fileChannel 传输到 socketChannel（零拷贝）
fileChannel.transferTo(position, count, socketChannel);
```

# 6. 日志管理（LogManager）

## 6.1 LogManager 核心结构

```scala
class LogManager(logDirs: Seq[File], ...) {
  // 所有日志对象
  private val currentLogs = new Pool[TopicPartition, UnifiedLog]()

  // 后台线程
  private val scheduler = new KafkaScheduler(1) // log.flush.scheduler.interval.ms
  private val cleaner: LogCleaner              // log.cleaner.threads

  // 关键方法
  def getOrCreateLog(tp: TopicPartition, ...): UnifiedLog
  def getLog(tp: TopicPartition): Option[UnifiedLog]
  def startup()    // 启动时加载所有日志
  def shutdown()
}
```

## 6.2 UnifiedLog 核心职责

```scala
class UnifiedLog(...) {
  val dir: File                          // Partition 日志目录
  val config: LogConfig                  // 日志配置
  val segments: LogSegments              // Segment 集合
  val leaderEpochCache: LeaderEpochCache // Leader Epoch 缓存
  val offsetIndex: OffsetIndex           // 当前 Segment 的 offset 索引

  // 核心操作
  def append(records: MemoryRecords, ...): LogAppendInfo
  def read(startOffset: Long, maxLength: Int, ...): FetchDataInfo
  def deleteOldSegments(): Int
}
```

## 6.3 日志目录故障处理

Kafka 支持多个日志目录（`log.dirs` 用逗号分隔）。当某个目录故障时：

```properties
# 多个日志目录
log.dirs=/data/kafka-logs-1,/data/kafka-logs-2,/data/kafka-logs-3

# 是否在磁盘错误时关闭 Broker
log.dir.failure.timeout.ms=30000
```

Kafka 会检测目录的 I/O 错误，并将故障 Partition 标记为 offline，迁移到其他目录或 Broker。

# 7. 时间戳与 Offset 关系

## 7.1 三种时间戳类型

```java
// Producer 端配置
props.put("message.timestamp.type", "CreateTime"); // CreateTime 或 LogAppendTime

// CreateTime: 消息创建时的时间戳（Producer 设置）
// LogAppendTime: 消息追加到 Broker 的时间戳（Broker 覆盖）

// Broker 端 Topic 配置
message.timestamp.type=CreateTime   // 默认
// 或
message.timestamp.type=LogAppendTime
```

## 7.2 按时间查找

```java
// 查找指定时间之后第一条消息的 offset
Map<TopicPartition, Long> timestampsToSearch = new HashMap<>();
timestampsToSearch.put(new TopicPartition("topic", 0), epochMs);

Map<TopicPartition, OffsetAndTimestamp> offsets = consumer
    .offsetsForTimes(timestampsToSearch);

OffsetAndTimestamp result = offsets.get(new TopicPartition("topic", 0));
long offset = result.offset();   // 对应的 offset
long timestamp = result.timestamp(); // 实际的时间戳
```

内部实现：
1. 二分查找 `.timeindex` 文件
2. 找到 Timestamp ≥ 目标时间的第一个条目
3. 返回对应的 Offset

# 8. 日志异常恢复

## 8.1 不完整的 Batch 处理

当 Broker 崩溃时，最后一个 Segment 可能有不完整写入：

```
✅ 完整 Batch: [Header + Records + CRC]  ← CRC 校验通过
❌ 不完整 Batch: [Header + 部分 Records]  ← CRC 校验失败 → 截断
```

恢复时 Kafka 会：
1. 检查最后 Batch 的 CRC
2. 若 CRC 错误 → 截断到上一个完整 Batch
3. 若 CRC 正确 → 保留整个 Batch

## 8.2 多余 Batch 处理

由于 ISR 不一致导致 Leader 比 Follower 多写了部分数据，但新 Leader 选举后需要截断到 HW（High Watermark）：

```
场景：
  Leader 写入了 offset=100,101,102 （但 101,102 只有 Leader 确认）
  Follower 只复制到 offset=100
  Leader 宕机 → Follower 成为新 Leader

恢复：
  旧 Leader 重启后变为 Follower
  检查自己的 LEO > 新 Leader 的 LEO
  Truncate 到 HW=100（删除 offset=101,102）
```

截断时的 `LeaderEpochCache` 用于安全地判断哪些数据需要截断。

# 9. 存储参数速查

```properties
########################
# Segment 相关
########################
log.segment.bytes=1073741824          # Segment 大小 (1GB)
log.segment.ms=604800000              # Segment 滚动时间 (7天)
log.index.size.max.bytes=10485760     # 索引文件最大大小 (10MB)
log.index.interval.bytes=4096         # 索引粒间隔 (4KB)

########################
# 保留策略 (Delete)
########################
log.retention.ms=604800000            # 保留时间 (7天)
log.retention.bytes=-1                # 不按大小限制
log.retention.check.interval.ms=300000 # 检查间隔 (5分钟)

########################
# 清理策略 (Compact)
########################
log.cleaner.enable=true
log.cleaner.threads=1
log.cleaner.backoff.ms=15000
log.cleaner.min.cleanable.ratio=0.5
min.compaction.lag.ms=0
max.compaction.lag.ms=9223372036854775807
delete.retention.ms=86400000          # Tombstone 保留 (24h)

########################
# 刷盘策略
########################
log.flush.interval.messages=9223372036854775807
log.flush.interval.ms=9223372036854775807
log.flush.scheduler.interval.ms=9223372036854775807
```
