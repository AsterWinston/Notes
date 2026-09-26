
> 源码版本：RocketMQ 4.9.x
> 核心模块：`store/src/main/java/org/apache/rocketmq/store/`

# 1. ConsumeQueue 是什么

ConsumeQueue 是 CommitLog 的**消费索引**，为每个 `{Topic, QueueId}` 建立独立的队列文件。

```
为什么需要 ConsumeQueue？

CommitLog 中所有 Topic 的消息混在一起，Consumer 要找到 
"Topic-A, Queue-3" 的消息需要遍历整个 CommitLog → 太慢

ConsumeQueue 解决：
- 按 Topic/Queue 维度，存储 20 字节的索引条目
- Consumer 先读 ConsumeQueue (20B)，再按偏移量读 CommitLog
```

# 2. ConsumeQueue 文件格式

## 2.1 每个条目 = 20 字节

```
ConsumeQueue 条目结构 (CqUnit)：
┌──────────────────────────────────────┐
│ CommitLog Offset (8B)  物理偏移量    │
├──────────────────────────────────────┤
│ Message Size (4B)      消息体大小    │
├──────────────────────────────────────┤
│ Tags Code (8B)         Tag 哈希码    │
└──────────────────────────────────────┘
合计：20 字节
```

## 2.2 存储路径

```
~/store/consumequeue/
    ├── Topic-A/
    │    ├── 0/
    │    │    ├── 00000000000000000000  (第1个文件)
    │    │    └── 00000000000000600000  (第2个文件，offset=600000)
    │    ├── 1/
    │    └── 2/
    └── Topic-B/
         └── 0/
```

**单文件大小** = 30万条 × 20B = 6MB (默认)
**文件名** = 文件中第一条记录的物理偏移量

# 3. ConsumeQueue 构建 — ReputMessageService

## 3.1 ReputMessageService 核心逻辑

```java
// store/.../DefaultMessageStore.java 内部类
class ReputMessageService extends ServiceThread {
    
    // 当前重放到的 CommitLog 偏移量
    private volatile long reputFromOffset = 0;

    @Override
    public void run() {
        DefaultMessageStore.log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            try {
                Thread.sleep(1);
                this.doReput();
            } catch (Exception e) {
                DefaultMessageStore.log.warn(this.getServiceName() + " has exception. ", e);
            }
        }

        DefaultMessageStore.log.info(this.getServiceName() + " service end");
    }

    private void doReput() {
        if (this.reputFromOffset < DefaultMessageStore.this.commitLog.getMinOffset()) {
            // 最小偏移量已变化，重新设定 reputFromOffset
            this.reputFromOffset = DefaultMessageStore.this.commitLog.getMinOffset();
        }

        // 1. 从 CommitLog 获取 reputFromOffset 往后的数据
        SelectMappedBufferResult result = DefaultMessageStore.this.commitLog
            .getData(reputFromOffset);
        if (result == null) {
            return;
        }

        try {
            // 2. 从 reputFromOffset 开始循环读取 CommitLog 中的消息
            int readSize = 0;
            while (readSize < result.getSize() && doNext) {
                // 3. 解析一条消息 → DispatchRequest
                DispatchRequest dispatchRequest = DefaultMessageStore.this.commitLog
                    .checkMessageAndReturnSize(result.getByteBuffer(), false, false);

                if (dispatchRequest.isSuccess()) {
                    int size = dispatchRequest.getMsgSize();

                    // 4. 分发：同时更新 ConsumeQueue 和 IndexFile
                    DefaultMessageStore.this.doDispatch(dispatchRequest);

                    if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig()
                           .getBrokerRole() || DefaultMessageStore.this.getMessageStoreConfig()
                           .isEnableDLegerCommitLog()) {
                        // Leader (Master/DLedger) 分发
                        this.reputFromOffset += size;
                        readSize += size;
                    } else {
                        // Follower 也分发 (同步数据)
                        this.reputFromOffset += size;
                        readSize += size;
                    }

                    // 通知消息到达监听器
                    DefaultMessageStore.this.messageArrivingListener
                        .arriving(dispatchRequest.getTopic(),
                            dispatchRequest.getQueueId(),
                            dispatchRequest.getConsumeQueueOffset() + 1,
                            dispatchRequest.getTagsCode(),
                            dispatchRequest.getStoreTimestamp(),
                            ...);
                } else {
                    break;
                }
            }
        } finally {
            result.release();
        }
    }
}
```

## 3.2 doDispatch — 分发到 ConsumeQueue 和 IndexFile

```java
// DefaultMessageStore.doDispatch()
public void doDispatch(DispatchRequest req) {
    // 1. 写入 ConsumeQueue
    for (CommitLogDispatcher dispatcher : this.dispatcherList) {
        dispatcher.dispatch(req);
    }
}

// CommitLogDispatcherBuildConsumeQueue (构建 ConsumeQueue)
class CommitLogDispatcherBuildConsumeQueue implements CommitLogDispatcher {
    @Override
    public void dispatch(DispatchRequest request) {
        int queueId = request.getQueueId() < 0 
            ? request.getQueueId() : request.getQueueId();
        
        // 根据 Topic 找到 ConsumeQueue
        ConsumeQueue cq = findConsumeQueue(request.getTopic(), queueId);
        
        // 追加条目：20B = commitLogOffset(8B) + msgSize(4B) + tagsCode(8B)
        cq.putMessagePositionInfoWrapper(request);
    }
}

// CommitLogDispatcherBuildIndex (构建 IndexFile)
class CommitLogDispatcherBuildIndex implements CommitLogDispatcher {
    @Override
    public void dispatch(DispatchRequest request) {
        if (DefaultMessageStore.this.messageStoreConfig.isMessageIndexEnable()) {
            DefaultMessageStore.this.indexService.buildIndex(request);
        }
    }
}
```

## 3.3 ConsumeQueue.putMessagePositionInfoWrapper()

```java
// ConsumeQueue 类
public void putMessagePositionInfoWrapper(DispatchRequest request) {
    final int maxRetries = 30;  // 最多重试 30 次
    boolean canWrite = this.defaultMessageStore.getRunningFlags()
        .isCQWriteable();

    for (int i = 0; i < maxRetries && canWrite; i++) {
        long writeOffset = this.byteBufferIndex * CQ_STORE_UNIT_SIZE;
        // 获取或创建 MappedFile
        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile(writeOffset);

        // 追加 20B 条目
        boolean result = mappedFile.appendMessage(
            CQ_STORE_UNIT_SIZE,  // 每条 20B
            new ConsumeQueueAppendMessageCallback() {
                @Override
                public AppendMessageResult doAppend(ByteBuffer byteBuffer) {
                    byteBuffer.putLong(request.getCommitLogOffset());  // 8B
                    byteBuffer.putInt(request.getMsgSize());           // 4B
                    byteBuffer.putLong(request.getTagsCode());         // 8B
                    return new AppendMessageResult(AppendMessageStatus.PUT_OK);
                }
            });
        if (result) {
            break;
        }
    }
}
```

# 4. IndexFile 构建源码

## 4.1 IndexService.buildIndex()

```java
// IndexService — 索引服务
public void buildIndex(DispatchRequest req) {
    // 1. 获取或创建 IndexFile (最后一个文件)
    IndexFile indexFile = retryGetAndCreateIndexFile();

    // 2. 生成唯一 Key 的索引
    // msg Keys = "ORDER_ID_xxx GLOBAL_ID_yyy" (空格分隔)
    if (req.getUniqKey() != null) {
        indexFile.putKey(req.getUniqKey(), req.getCommitLogOffset(), 
            req.getStoreTimestamp());
    }

    // 3. 生成业务 Key 的索引
    // 用户自定义的 msg.setKeys("business_key")
    String[] keys = req.getKeys().split(KEY_SEPARATOR); // 按空格分割
    for (String key : keys) {
        if (key != null && key.length() > 0) {
            indexFile.putKey(key, req.getCommitLogOffset(), 
                req.getStoreTimestamp());
        }
    }
}
```

## 4.2 IndexFile.putKey()

```java
// IndexFile 类
public boolean putKey(String key, long phyOffset, long storeTimestamp) {
    // 1. 检查是否需要创建新 IndexFile
    if (this.indexHeader.getIndexCount() >= this.hashSlotNum 
            || this.indexHeader.getIndexCount() >= Integer.MAX_VALUE) {
        return false;  // 满了
    }

    // 2. 计算 Key 的 hash
    int keyHash = indexKeyHashMethod(key);

    // 3. 计算 hash slot 位置
    int slotPos = keyHash % this.hashSlotNum;  // 默认 500万

    // 4. 计算新的 index count
    int absIndexPos = this.indexHeader.getIndexCount();
    int newIndexCount = absIndexPos + 1;

    // 5. 写入 Index 条目 (20B)
    int indexPos = IndexHeader.INDEX_HEADER_SIZE                  // 40B header
                 + this.hashSlotNum * hashSlotSize                // hash slot 表
                 + absIndexPos * indexSize;                       // 新条目的位置

    // 每条记录格式:
    // keyHash(4B) + phyOffset(8B) + timeDiff(4B) + prevIndexNo(4B) = 20B
    this.mappedByteBuffer.putInt(indexPos, keyHash);
    this.mappedByteBuffer.putLong(indexPos + 4, phyOffset);
    this.mappedByteBuffer.putInt(indexPos + 12, 
        (int) (storeTimestamp - this.indexHeader.getBeginTimestamp()));
    this.mappedByteBuffer.putInt(indexPos + 16, 
        hashSlotTable[slotPos]);  // 指向前一条

    // 6. 更新 hash slot 指向新条目
    hashSlotTable[slotPos] = absIndexPos;

    // 7. 更新 Header 数据
    if (newIndexCount <= 1) {
        this.indexHeader.setBeginTimestamp(storeTimestamp);
    }
    if (this.indexHeader.getEndTimestamp() < storeTimestamp) {
        this.indexHeader.setEndTimestamp(storeTimestamp);
    }
    this.indexHeader.setIndexCount(newIndexCount);
    this.indexHeader.setBeginPhyOffset(phyOffset);
    this.indexHeader.setEndPhyOffset(phyOffset);

    return true;
}
```

## 4.3 查询逻辑

```java
// IndexFile 查询
public void selectPhyOffset(List<Long> phyOffsets, String key, int maxNum, 
        long begin, long end, boolean lock) {
    
    // 1. 校验时间范围
    if (begin > this.indexHeader.getEndTimestamp() 
            || end < this.indexHeader.getBeginTimestamp()) {
        return; // 不在索引范围内
    }

    // 2. 计算 key hash
    int keyHash = indexKeyHashMethod(key);
    // 3. 定位 hash slot
    int slotPos = keyHash % this.hashSlotNum;
    int slotValue = this.mappedByteBuffer.getInt(slotPos);

    // 4. 遍历链表
    int timeDiff;
    for (int i = slotValue; i >= 0; ) {
        int entryPos = IndexHeader.INDEX_HEADER_SIZE 
                     + this.hashSlotNum * hashSlotSize 
                     + i * indexSize;

        int entryKeyHash = this.mappedByteBuffer.getInt(entryPos);
        long entryPhyOffset = this.mappedByteBuffer.getLong(entryPos + 4);
        int entryTimeDiff = this.mappedByteBuffer.getInt(entryPos + 12);
        long entryStoreTimestamp = this.indexHeader.getBeginTimestamp() + entryTimeDiff;
        int entryPrevIndexNo = this.mappedByteBuffer.getInt(entryPos + 16);

        // 5. 匹配条件
        if (entryKeyHash == keyHash          // key hash 匹配
                && entryStoreTimestamp >= begin   // 时间 >= 开始
                && entryStoreTimestamp <= end) {  // 时间 <= 结束
            phyOffsets.add(entryPhyOffset);
            if (phyOffsets.size() >= maxNum) {
                break;
            }
        }

        // 6. 下一个链表节点
        i = entryPrevIndexNo;  // 0 表示链表结束
    }
}
```

# 5. ConsumeQueue 刷盘

```java
// FlushConsumeQueueService
// 定时将 ConsumeQueue 的 PageCache 刷到磁盘
class FlushConsumeQueueService extends ServiceThread {
    @Override
    public void run() {
        while (!this.isStopped()) {
            // 每 1s 触发一次刷盘
            int interval = DefaultMessageStore.this.getMessageStoreConfig()
                .getFlushIntervalConsumeQueue();
            this.waitForRunning(interval);

            // 刷盘
            DefaultMessageStore.this.flushConsumeQueue();
        }
    }
}

// DefaultMessageStore.flushConsumeQueue()
private void flushConsumeQueue() {
    for (ConsumeQueue cq : this.consumeQueueTable.values()) {
        cq.flush(0);  // 至少刷 0 页 (无论多少都刷)
    }
}
```

# 6. IndexFile 文件管理

```java
public class IndexFile {
    // IndexFile 结构:
    // ┌────────────────────────────────────┐
    // │ IndexHeader (40B)                   │
    // │  - beginTimestamp  (8B)             │
    // │  - endTimestamp    (8B)             │
    // │  - beginPhyOffset  (8B)             │
    // │  - endPhyOffset    (8B)             │
    // │  - hashSlotCount   (4B) = 5000000   │
    // │  - indexCount      (4B)             │
    // ├────────────────────────────────────┤
    // │ Hash Slot 表 (5000000 × 4B = 20MB)  │
    // │  slot[i] = latest entry index no    │
    // ├────────────────────────────────────┤
    // │ Index 条目 (每个 20B)               │
    // │  keyHash(4B) + phyOffset(8B)        │
    // │  + timeDiff(4B) + prevIndexNo(4B)  │
    // └────────────────────────────────────┘
    
    // 单个 IndexFile 最大 = 40 + 5000000×4 + 2000×20 = 约 400MB
    // (其中 indexCount 可达 5000000 条)
}
```

# 7. 参考

- [DefaultMessageStore — ReputMessageService](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/DefaultMessageStore.java)
- [ConsumeQueue 源码](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/ConsumeQueue.java)
- [IndexService 源码](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/index/IndexService.java)
