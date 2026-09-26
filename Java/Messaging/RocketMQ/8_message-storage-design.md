
# 1. 存储整体架构

RocketMQ 存储模块是整个系统的核心，基于**文件系统**（非数据库），采用**顺序写 + 随机读**的设计。

```
                              store/
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
           commitlog/     consumequeue/     index/
         (消息主存储)     (消费队列索引)    (消息索引)
          
  Topic-Order:
    消息1 → CommitLog 全局唯一、顺序追加
    消息2 → ConsumeQueue 消费索引(20B/条)、随机读取
    消息3 → IndexFile 哈希索引(Key/时间查询)
```

## 1.1 存储三大核心文件

| 文件类型 | 位置 | 格式 | 用途 |
|----------|------|------|------|
| **CommitLog** | `~/store/commitlog/` | 消息原始存储 | 所有 Topic 的消息都写入同一个 CommitLog（顺序写） |
| **ConsumeQueue** | `~/store/consumequeue/{topic}/{queueId}/` | 20 字节定长记录 | 按 Topic-Queue 维度索引 CommitLog，用于消费 |
| **IndexFile** | `~/store/index/` | 哈希 + 链表 | 按 Key/时间范围快速定位消息 |

## 1.2 为什么顺序写 + 随机读？

```
磁盘 I/O 特性：
┌──────────────────────────────────────────────────────────┐
│  顺序写 (Sequential Write) ≈ 600MB/s                    │
│  随机写 (Random Write)     ≈ 100KB/s                    │
│  顺序读 ≈ 随机读 (SSD 上差异不大)                        │
│  差距：顺序写比随机写快约 6000 倍                       │
└──────────────────────────────────────────────────────────┘

RocketMQ 的设计：
- 所有消息写入同一个 CommitLog → 顺序写 (高吞吐)
- 消费时通过 ConsumeQueue 定位 → 随机读 (可接受)
- 利用 PageCache + MappedByteBuffer → 减少磁盘 I/O
```

# 2. 内存映射文件 (MappedFile)

## 2.1 为什么使用内存映射？

```
传统 I/O 流程:
用户态 Buffer → 内核态 PageCache → 磁盘 (2次拷贝)

MappedByteBuffer 流程:
逻辑地址 ──(MMU)──→ 物理地址 (直接访问 PageCache，0次拷贝)
```

## 2.2 MappedFile 数据结构

```java
public class MappedFile extends ReferenceResource {
    // 文件大小 (默认 CommitLog 1GB, ConsumeQueue 约 600w*20B ≈ 120MB)
    private int fileSize;

    // 文件通道
    private FileChannel fileChannel;

    // 内存映射缓冲区 (写入消息的缓冲区)
    private MappedByteBuffer mappedByteBuffer;

    // 写入位置 (当前已写入的位置)
    private final AtomicInteger wrotePosition = new AtomicInteger(0);

    // 提交位置 (已 flush 到磁盘的位置)
    private final AtomicInteger committedPosition = new AtomicInteger(0);

    // 刷盘位置 (已 flush 到磁盘的位置，同 committedPosition)
    private final AtomicInteger flushedPosition = new AtomicInteger(0);

    // 文件名 (物理偏移量，如 00000000000000000000)
    private String fileName;

    // 文件起始偏移量
    private long fileFromOffset;
}
```

## 2.3 文件命名规则

```
CommitLog 文件名 = 文件起始偏移量 (20位，左补0)
例如：00000000000000000000  (第一个文件，偏移量 0)
     00000000001073741824  (第二个文件，偏移量 1073741824 = 1GB)
```

## 2.4 MappedFile 三种写入方式

| 方式 | 方法 | 刷盘保证 | 性能 | 说明 |
|------|------|----------|------|------|
| 直接写入 | `mappedByteBuffer.put()` | 否 | 最高 | PageCache 层 |
| Commit | `commit()` | `MappedByteBuffer.force()` | 中 | 数据从 PageCache 刷到磁盘 |
| 异步刷盘 | `fluch()` | 异步 | 高 | 后台定时刷盘 |

# 3. CommitLog 设计

## 3.1 存储格式

```
每条消息的存储格式：

┌────────────────────────────────────────────────┐
│  TOTAL_SIZE (4B)         总长度                │
├────────────────────────────────────────────────┤
│  MAGIC_CODE (4B)         魔数                  │
├────────────────────────────────────────────────┤
│  BODY_CRC (4B)           消息体 CRC 校验码      │
├────────────────────────────────────────────────┤
│  QUEUE_ID (4B)           MessageQueue ID       │
├────────────────────────────────────────────────┤
│  FLAG (4B)               标志位                │
├────────────────────────────────────────────────┤
│  QUEUE_OFFSET (8B)       ConsumeQueue 中的偏移  │
├────────────────────────────────────────────────┤
│  PHYSICAL_OFFSET (8B)    CommitLog 物理偏移量   │
├────────────────────────────────────────────────┤
│  SYS_FLAG (4B)           系统标志位            │
│  BORN_TIMESTAMP (8B)     消息产生时间戳         │
│  BORN_HOST (8B)          消息产生主机           │
│  STORE_TIMESTAMP (8B)    消息存储时间戳         │
│  STORE_HOST_ADDRESS (8B) 消息存储主机           │
│  RECONSUME_TIMES (4B)    重试次数              │
│  Prepared Transaction Offset (8B) 事务          │
├────────────────────────────────────────────────┤
│  BODY_SIZE (4B)          消息体大小            │
│  BODY (N)                消息体                │
├────────────────────────────────────────────────┤
│  TOPIC_SIZE (1B)         Topic 长度            │
│  TOPIC (N)               Topic 名称            │
├────────────────────────────────────────────────┤
│  PROPERTIES_SIZE (2B)    属性长度              │
│  PROPERTIES (N)          扩展属性              │
└────────────────────────────────────────────────┘
```

## 3.2 MappedFileQueue

```java
public class MappedFileQueue {
    // 存储路径
    private final String storePath;

    // 单个 MappedFile 大小
    private final int mappedFileSize;

    // 所有 MappedFile 列表（CopyOnWriteArrayList 保证线程安全）
    private final CopyOnWriteArrayList<MappedFile> mappedFiles = 
        new CopyOnWriteArrayList<>();

    // 当前写入的 MappedFile
    // flushedWhere / committedWhere 等指针
}
```

## 3.3 写入流程

```java
// CommitLog.putMessage()
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    // 1. 设置存储时间
    msg.setStoreTimestamp(System.currentTimeMillis());

    // 2. 序列化消息
    // （编码为上述 CommitLog 格式）

    // 3. 获取最后一个 MappedFile
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

    // 4. 追加写入
    result = mappedFile.appendMessage(msg, this.appendMessageCallback);

    // 5. 提交
    // handleDiskFlush(result, msg) — 处理刷盘
    // handleHA(result, msg) — 处理 HA 同步

    return result;
}
```

# 4. ConsumeQueue 设计

## 4.1 为什么需要 ConsumeQueue？

```
CommitLog 是全局唯一的，所有 Topic 混在一起写入

问题：Consumer 如何找到 "Topic-A, Queue-3" 的消息？

解决：ConsumeQueue 按 {Topic, QueueId} 维度建立索引
     每个 ConsumeQueue 条目 = 20 字节
     Consumer 只需顺序读取 ConsumeQueue，再按偏移量随机读 CommitLog
```

## 4.2 20 字节条目格式

```
ConsumeQueue 条目（每条 20 字节）：
┌──────────────────────┬──────────────────────┬────────────┬───────────┐
│ CommitLog Offset (8B)│ Message Size (4B)    │ Tag HashCode (8B)        │
└──────────────────────┴──────────────────────┴────────────┴───────────┘
        偏移量               消息体大小                Tag 哈希码
```

**读取流程**：
```
1. 根据 ConsumeQueue 的 offset 计算文件位置：
   pos = offset * 20

2. 读取 20 字节，得到 { commitLogOffset, msgSize, tagsCode }

3. 根据 commitLogOffset 定位 CommitLog，读取 msgSize 字节

4. 根据 tagsCode 做 Tag 过滤（Consumer 端过滤）
```

## 4.3 构建时机 — ReputMessageService

```java
public class ReputMessageService extends ServiceThread {
    @Override
    public void run() {
        while (!this.isStopped()) {
            // 1. 从 CommitLog 的 reputFromOffset 位置开始读取
            SelectMappedBufferResult result = 
                DefaultMessageStore.this.commitLog.getData(reputFromOffset);

            if (result != null) {
                // 2. 解析消息：读取 DispatchRequest
                DispatchRequest dispatchRequest = 
                    DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(...);

                // 3. 分发到 ConsumeQueue 和 IndexFile
                DefaultMessageStore.this.doDispatch(dispatchRequest);

                // 4. 推进 reputFromOffset
                this.reputFromOffset += size;
            }
        }
    }
}
```

# 5. IndexFile 设计

## 5.1 用途

按 **消息 Key** 或 **时间范围** 快速定位消息（适用于`msg.setKeys("orderId_123")` 后的查询）。

## 5.2 文件格式

```
IndexFile 结构：
┌─────────────────────────────────────────────┐
│  IndexHeader (40B)                           │
│  ┌─────────────────────────────────────────┐ │
│  │ beginTimestamp   (8B)                   │ │
│  │ endTimestamp     (8B)                   │ │
│  │ beginPhyOffset   (8B)                   │ │
│  │ endPhyOffset     (8B)                   │ │
│  │ hashSlotCount    (4B)                   │ │
│  │ indexCount       (4B)                   │ │
│  └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│  Hash Slot 表 (hashSlotCount × 4B = 500w×4B) │
│  slot[i] = 最近一条 Index 条目的 indexNo     │
├─────────────────────────────────────────────┤
│  Index 条目表 (indexCount × 20B)            │
│  ┌─────────────────────────────────────────┐ │
│  │ keyHash     (4B)                        │ │
│  │ phyOffset   (8B)                        │ │
│  │ timeDiff    (4B)                        │ │
│  │ prevIndexNo (4B)                        │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## 5.3 查询流程

```
1. 计算 key 的 Hash 值 → keyHash
2. keyHash % hashSlotCount → 找到 hashSlot[keyHash]
3. 遍历链表查询（nextItem = IndexEntry.prevIndexNo）
4. 根据 phyOffset 定位 CommitLog，读取消息
```

# 6. 刷盘机制

## 6.1 两种策略对比

| 策略 | 配置值 | 流程 | 可靠性 | 性能 |
|------|--------|------|--------|------|
| **同步刷盘** | SYNC_FLUSH | 写入 PageCache → 等待 force() 完成 → 返回成功 | 高（断电不丢） | 低 |
| **异步刷盘** | ASYNC_FLUSH | 写入 PageCache → 立即返回成功 → 后台定时 force() | 中（丢 PageCache 数据） | 高 |

## 6.2 同步刷盘

```java
// GroupCommitService (同步刷盘)
// 使用 CountDownLatch 等待刷盘完成

// CommitLog.RUN — 处理刷盘
public void handleDiskFlush(AppendMessageResult result, MessageExt msg) {
    if (FlushDiskType.SYNC_FLUSH == flushDiskType) {
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
        GroupCommitRequest request = new GroupCommitRequest(
            result.getWroteOffset() + result.getWroteBytes());
        service.putRequest(request);
        // 等待刷盘完成
        request.waitForFlush(timeout);
    }
}
```

## 6.3 异步刷盘

```java
// FlushRealTimeService (异步刷盘)
// 定时执行 MappedByteBuffer.force()

// CommitLog.RUN — 处理刷盘
public void handleDiskFlush(AppendMessageResult result, MessageExt msg) {
    if (FlushDiskType.ASYNC_FLUSH == flushDiskType) {
        // 唤醒 FlushRealTimeService
        this.flushCommitLogService.wakeup();
    }
}
```

# 7. 过期文件清理

```
文件保留时间：fileReservedTime (默认 72 小时)
清理触发时间：deleteWhen (默认凌晨 4 点)

清理条件：
1. 文件最后修改时间 < (当前时间 - fileReservedTime)
2. 文件不在当前读写范围内
3. 磁盘空间不足时强制清理（cleanImmediately）
```

# 8. 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `storePathRootDir` | `~/store` | 存储根目录 |
| `storePathCommitLog` | `~/store/commitlog` | CommitLog 存储路径 |
| `mapedFileSizeCommitLog` | 1GB | CommitLog 单文件大小 |
| `mapedFileSizeConsumeQueue` | 30w×20B ≈ 6MB | ConsumeQueue 单文件大小 |
| `flushDiskType` | ASYNC_FLUSH | 刷盘策略 |
| `syncFlushTimeout` | 5s | 同步刷盘超时 |
| `flushIntervalCommitLog` | 500ms | 异步刷盘间隔 |
| `flushLeastPagesWhenWarmMapedFile` | 4096 页(16MB) | 最少刷盘页数 |
| `commitCommitLogInterval` | 200ms | CommitLog 提交间隔 |
| `fileReservedTime` | 72h | 文件保留时间 |

# 9. 参考

- [RocketMQ 存储设计文档](https://github.com/apache/rocketmq/blob/master/docs/cn/design.md)
- [DefaultMessageStore 源码](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/DefaultMessageStore.java)
