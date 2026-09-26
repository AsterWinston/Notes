
> 源码版本：RocketMQ 4.9.x
> 核心模块：`store/src/main/java/org/apache/rocketmq/store/`

# 1. CommitLog 类层次结构

```
CommitLog (接口)
    └── CommitLog (默认实现)
         ├── MappedFileQueue           — 管理 MappedFile 列表
         │    └── CopyOnWriteArrayList<MappedFile>
         │
         ├── FlushCommitLogService     — 刷盘服务
         │    ├── GroupCommitService   (SYNC_FLUSH)
         │    └── FlushRealTimeService (ASYNC_FLUSH)
         │
         ├── AppendMessageCallback     — 消息追加回调
         │
         └── ReputMessageService       — 消息重放/构建 ConsumeQueue
```

# 2. CommitLog 初始化

```java
// CommitLog 构造函数
public CommitLog(final DefaultMessageStore defaultMessageStore) {
    // 1. 设置存储路径 (如: ~/store/commitlog)
    String storePath = defaultMessageStore.getMessageStoreConfig()
        .getStorePathCommitLog();
    // 2. 设置单个 MappedFile 大小 (默认 1GB)
    int mappedFileSize = defaultMessageStore.getMessageStoreConfig()
        .getMapedFileSizeCommitLog();
    // 3. 创建 MappedFileQueue
    this.mappedFileQueue = new MappedFileQueue(storePath, mappedFileSize, 
        defaultMessageStore.getAllocateMappedFileService());

    // 4. 初始化刷盘服务
    if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig()
            .getFlushDiskType()) {
        this.flushCommitLogService = new GroupCommitService();
    } else {
        this.flushCommitLogService = new FlushRealTimeService();
    }

    // 5. 初始化追加回调
    this.appendMessageCallback = new DefaultAppendMessageCallback(
        defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());

    // 6. 初始化 ReputMessageService (消息重放)
    this.reputMessageService = new ReputMessageService();
}
```

# 3. putMessage() — 消息写入核心

## 3.1 入口方法

```java
// CommitLog.putMessage()
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    // 1. 设置存储时间戳
    msg.setStoreTimestamp(System.currentTimeMillis());
    msg.setBodyCRC(UtilAll.crc32(msg.getBody()));

    // 2. 存储延迟消息
    // 如果消息有 DELAY 属性 → 先做延迟消息处理
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
            || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
        // 延迟消息处理
        if (msg.getDelayTimeLevel() > 0) {
            // 修改 Topic → SCHEDULE_TOPIC_XXXX
            // 修改 QueueId → delayLevel - 1
            // 保存原始 Topic/QueueId → Properties
            String topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
            int queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, 
                String.valueOf(msg.getQueueId()));
            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
    }

    // 3. 获取最后一个 MappedFile
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
    // 如果为空或已写满 → 创建新的 MappedFile
    if (null == mappedFile || mappedFile.isFull()) {
        mappedFile = this.mappedFileQueue.createMappedFile(createFileCallback);
    }

    // 4. 追加到 MappedFile（采用自旋重试）
    long elapsedTimeInLock = 0;
    PutMessageResult putMessageResult = null;

    // 5. 获取 putMessageLock（默认使用 ReentrantLock for 自旋）
    putMessageLock.lock();
    try {
        // 继续追加写入
        putMessageResult = mappedFile.appendMessage(msg, this.appendMessageCallback);
    } finally {
        putMessageLock.unlock();
    }

    // 6. 处理刷盘
    if (putMessageResult.getPutMessageStatus() == PutMessageStatus.PUT_OK) {
        // 提交消息（MappedByteBuffer.commit()）
        handleMessageCommit(putMessageResult, msg);
        // 刷盘
        handleDiskFlush(putMessageResult, msg);
        // HA 同步
        handleHA(putMessageResult, msg, tranType);
    }

    return putMessageResult;
}
```

## 3.2 为什么用 putMessageLock

```java
// 注释原文：
// "Serialize message appends to get an accurately written sequence per file."
// 翻译：串行化消息追加，确保文件中精确的顺序写入。

// 如果不加锁：
// Thread-1: write 开始 → 获取 wrotePosition
// Thread-2: write 开始 → 获取 wrotePosition (可能与 Thread-1 相同!)
// → 导致两个线程写入同一位置 → 数据损坏

// 加锁目的：保证写入的串行性，保持文件内消息的严格顺序
// 性能影响：锁粒度在单个文件级别，高并发场景下仍有很好性能
```

# 4. DefaultAppendMessageCallback — 消息编码

```java
class DefaultAppendMessageCallback implements AppendMessageCallback {
    // 最大消息体大小: 4MB (可配置)
    private final int maxMessageSize;

    @Override
    public AppendMessageResult doAppend(long fileFromOffset, 
            ByteBuffer byteBuffer, int maxBlank, MessageExtBrokerInner msgInner) {

        // 当前 ByteBuffer 写指针
        long wroteOffset = fileFromOffset + byteBuffer.position();

        // 1. 计算固定长度
        // sysFlag + bornHost + bornTimestamp + storeHost + storeTimestamp... 
        // = 固定长度 (104 bytes, 不同版本略有差异)
        byte[] propertiesData = MessageDecoder.messageProperties2Bytes(msgInner.getProperties());
        int propertiesLength = propertiesData == null ? 0 : propertiesData.length;
        final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
        final int topicLength = topicData.length;
        final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;
        final int msgLen = calMsgLength(msgInner.getSysFlag(), bodyLength, 
            topicLength, propertiesLength);

        // 2. 检查空间是否足够
        if (msgLen > maxBlank) { // 不够 → 填充空白，返回 END_OF_FILE
            this.msgStoreItemMemory.flip();
            byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
            return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, ...);
        }

        // 3. 写入消息总长度 (4B)
        byteBuffer.putInt(msgLen);

        // 4. 写入 Magic Code (4B) — 魔数校验
        byteBuffer.putInt(MessageDecoder.MESSAGE_MAGIC_CODE);

        // 5. 写入 CRC (4B)
        byteBuffer.putInt(msgInner.getBodyCRC());

        // 6. 写入 QueueId (4B)
        byteBuffer.putInt(msgInner.getQueueId());

        // 7. 写入 Flag (4B)
        byteBuffer.putInt(msgInner.getFlag());

        // 8. 写入 QueueOffset (8B)
        byteBuffer.putLong(msgInner.getQueueOffset());

        // 9. 写入 PhysicalOffset (8B) — 当前物理偏移量
        byteBuffer.putLong(fileFromOffset + byteBuffer.position());

        // 10. 写入 SystemFlag (4B)
        byteBuffer.putInt(msgInner.getSysFlag());

        // 11. 写入各种时间戳和主机地址
        byteBuffer.putLong(msgInner.getBornTimestamp());
        byteBuffer.put(hostBytes);       // BornHost
        byteBuffer.putLong(msgInner.getStoreTimestamp());
        byteBuffer.put(hostBytes);       // StoreHost
        byteBuffer.putInt(msgInner.getReconsumeTimes());

        // 12. 写入事务偏移量 (8B)
        byteBuffer.putLong(msgInner.getPreparedTransactionOffset());

        // 13. 写入消息体大小和内容
        byteBuffer.putInt(bodyLength);
        if (bodyLength > 0) {
            byteBuffer.put(msgInner.getBody());
        }

        // 14. 写入 Topic 大小和内容
        byteBuffer.put((byte) topicLength);
        byteBuffer.put(topicData);

        // 15. 写入属性大小和内容
        byteBuffer.putShort((short) propertiesLength);
        if (propertiesLength > 0) {
            byteBuffer.put(propertiesData);
        }

        // 返回写入结果
        return new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, 
            msgLen, msgId, msgInner.getStoreTimestamp(), ...);
    }
}
```

# 5. 刷盘机制源码

## 5.1 同步刷盘：GroupCommitService

```java
class GroupCommitService extends FlushCommitLogService {
    // 请求队列 (信号量同步)
    private volatile List<GroupCommitRequest> requestsWrite = new ArrayList<>();
    private volatile List<GroupCommitRequest> requestsRead = new ArrayList<>();

    // 等待刷盘的请求
    static class GroupCommitRequest {
        long nextOffset;                 // 等待刷盘的偏移量
        CountDownLatch countDownLatch;   // 用于阻塞等待
    }

    @Override
    public void run() {
        while (!this.isStopped()) {
            // 1. 交换读写队列 (Double Buffering)
            swapRequests();

            // 2. 执行 force() 刷盘
            for (GroupCommitRequest req : requestsRead) {
                // 执行 MappedFile.flush()
                CommitLog.this.mappedFileQueue.getFlushedWhere();
                flushedWhere = CommitLog.this.mappedFileQueue.flush(flushPages);
                if (flushedWhere >= req.getNextOffset()) {
                    // 刷盘完成，唤醒等待线程
                    req.getCountDownLatch().countDown();
                }
            }

            // 3. 等待新请求或超时
            waitForRunning(10);
        }
    }
}

// CommitLog 中处理刷盘
public void handleDiskFlush(AppendMessageResult result, MessageExt msg) {
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig()
            .getFlushDiskType()) {
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;

        // 创建刷盘请求
        GroupCommitRequest request = new GroupCommitRequest(
            result.getWroteOffset() + result.getWroteBytes());
        service.putRequest(request);

        // 等待刷盘完成 (最多 5s)
        request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig()
            .getSyncFlushTimeout());
    }
}
```

## 5.2 异步刷盘：FlushRealTimeService

```java
class FlushRealTimeService extends FlushCommitLogService {
    @Override
    public void run() {
        while (!this.isStopped()) {
            // 1. 判断是否到达刷盘间隔 (默认 500ms)
            boolean flushCommitLogTimed = CommitLog.this
                .isAbleToFlush(flushCommitLogInterval);

            // 2. 判断是否有足够的数据需要刷盘 (至少 4 pages → 16KB)
            boolean flushCommitLogPages = CommitLog.this
                .isAbleToFlushByPages(flushCommitLogLeastPages);

            // 3. 执行 force()
            if (flushCommitLogTimed || flushCommitLogPages) {
                CommitLog.this.mappedFileQueue.flush(flushCommitLogLeastPages);
            }

            // 4. 等待
            waitForRunning(CommitLog.this.getCommitLogInterval());
        }
    }
}
```

# 6. 消息恢复 (Recover)

## 6.1 恢复场景

```
Broker 异常关闭后重启 → 需要恢复 CommitLog & ConsumeQueue

恢复流程:
1. 从最后一个 CommitLog 文件恢复
2. 逆序遍历文件 (从文件尾向前)
3. 找到最后一条完整消息 → 确定最终偏移量
4. 重建 ConsumeQueue (通过 ReputMessageService)
```

## 6.2 recoverNormally() 源码

```java
// DefaultMessageStore.recoverNormally()
private void recoverNormally(long maxPhyOffsetOfConsumeQueue) {
    // 1. 从最后一个 MappedFile 开始恢复 CommitLog
    List<MappedFile> mappedFiles = this.commitLog.getMappedFileQueue()
        .getMappedFiles();
    int index = mappedFiles.size() - 3;
    if (index < 0) index = 0;

    MappedFile mappedFile = mappedFiles.get(index);
    ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
    long processOffset = mappedFile.getFileFromOffset();

    // 2. 逐条解析消息，找到最后一条有效消息
    long mappedFileOffset = 0;
    while (true) {
        // 读取消息
        DispatchRequest req = this.commitLog.checkMessageAndReturnSize(
            byteBuffer, true, true);
        if (req.isSuccess()) {
            // 更新 processOffset
            mappedFileOffset += req.getMsgSize();
            processOffset += req.getMsgSize();
        } else {
            break;
        }
    }

    // 3. 截断 CommitLog（如果有多余数据）
    long maxPhysicalOffsetInCommitLog = processOffset;
    if (maxPhysicalOffsetInCommitLog < mappedFile.flushedWhere()) {
        // 截断
        mappedFile.setFlushedPosition(maxPhysicalOffsetInCommitLog);
    }

    // 4. 比较 ConsumeQueue 最大偏移量
    // 如果 CommitLog 偏移量 > ConsumeQueue 偏移量，需要重建 ConsumeQueue
    if (maxPhysicalOffsetInCommitLog > maxPhyOffsetOfConsumeQueue) {
        // 不能有错，要求重构建
        // 从 consumequeue 最大偏移开始重新转发
        this.commitLog.setReputFromOffset(maxPhyOffsetOfConsumeQueue);
    }
}
```

# 7. 完整写入调用链路

```
Producer.send()
    │
    ▼
Broker: SendMessageProcessor.processRequest()
    │
    ▼
SendMessageProcessor.asyncPutMessage()
    │
    ▼
DefaultMessageStore.putMessage(msgInner)
    │
    ▼
CommitLog.putMessage(msgInner)
    │
    ├── 延迟消息处理 (修改 Topic/QueueId)
    ├── 获取最后一个 MappedFile
    │
    ├── [获取 putMessageLock] ── 串行化写入
    │    │
    │    ▼
    │    mappedFile.appendMessage(msgInner, callback)
    │    │    │
    │    │    ▼
    │    │    DefaultAppendMessageCallback.doAppend()
    │    │    │
    │    │    ├── 计算消息大小
    │    │    ├── 编码消息为二进制 (按 CommitLog 格式)
    │    │    ├── ByteBuffer.put(header + body)
    │    │    └── 返回 wroteOffset, msgLen
    │    │
    │    ├── handleMessageCommit() — commit 到磁盘 (可选)
    │    ├── handleDiskFlush() — 刷盘
    │    │    ├── SYNC_FLUSH → GroupCommitService.putRequest()
    │    │    └── ASYNC_FLUSH → FlushRealTimeService.wakeup()
    │    └── handleHA() — 主从复制
    │
    └── [释放 putMessageLock]
    │
    ▼
返回 PutMessageResult 给调用方
```

# 8. 参考

- [CommitLog 源码](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/CommitLog.java)
- [MappedFile 源码](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/MappedFile.java)
