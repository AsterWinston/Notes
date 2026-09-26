
# 1. 数据流转定位

在 Canal 的数据管道中，EventSink 和 EventStore 位于 Parser 和 Client 之间：

```
Parser ──> [EventSink] ──> [EventStore] ──> Client
  │             │               │              │
  │       过滤/转换/聚合     暂存/持久化      拉取消费
  │                             │
  │                    ┌────────▼────────┐
  │                    │   RingBuffer    │
  │                    │ (Disruptor)     │
  │                    └─────────────────┘
```

# 2. EventSink — 事件过滤器

## 2.1 接口定义

```java
public interface CanalEventSink<T> extends CanalLifeCycle {
    
    /**
     * 过滤/下沉一个事件
     * @param event   原始事件对象
     * @param entry   解析后的 Entry
     * @param filterTableNames  过滤的表名列表
     * @return true 表示事件被接受，false 表示被过滤掉
     */
    boolean sink(T event, CanalEntry.Entry entry, String... filterTableNames) 
            throws CanalSinkException, InterruptedException;
}
```

## 2.2 EntryEventSink — 核心实现

```java
public class EntryEventSink implements CanalEventSink<List<CanalEntry.Entry>> {

    // Disruptor RingBuffer（与 Store 共享）
    private Disruptor<Event<CanalEntry.Entry>> disruptor;
    
    // 表名过滤规则
    private AviaterRegexFilter filter;
    private AviaterRegexFilter blackFilter;
    
    // 事务缓冲区
    private ThreadLocal<List<CanalEntry.Entry>> transactionBuffer;
    
    @Override
    public boolean sink(List<CanalEntry.Entry> entries, CanalEntry.Entry entry,
                        String... filterTableNames) {
        
        // 1. 空事件检查
        if (entry == null) {
            return false;
        }
        
        // 2. 事务开始/结束事件直接通过
        if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN ||
            entry.getEntryType() == EntryType.TRANSACTIONEND) {
            return doSink(entries);
        }
        
        // 3. 心跳事件直接通过
        if (entry.getEntryType() == EntryType.HEARTBEAT) {
            return doSink(entries);
        }
        
        // 4. 表级过滤
        if (!filterTable(entry)) {
            return false;
        }
        
        // 5. 黑名单过滤
        if (isInBlackList(entry)) {
            return false;
        }
        
        // 6. 放入事务缓冲区（保证事务完整性）
        transactionBuffer.get().add(entry);
        
        return true;
    }
    
    /**
     * 表过滤逻辑
     */
    private boolean filterTable(CanalEntry.Entry entry) {
        if (filter == null) return true;
        
        String schemaName = entry.getHeader().getSchemaName();
        String tableName = entry.getHeader().getTableName();
        
        // 构建匹配键: schema.table
        String fullName = schemaName + "." + tableName;
        
        return filter.filter(fullName);
    }
    
    /**
     * 黑名单检查
     */
    private boolean isInBlackList(CanalEntry.Entry entry) {
        if (blackFilter == null) return false;
        
        String schemaName = entry.getHeader().getSchemaName();
        String tableName = entry.getHeader().getTableName();
        String fullName = schemaName + "." + tableName;
        
        return blackFilter.filter(fullName);
    }
}
```

## 2.3 事务边界完整性保证

EventSink 的一个重要职责是保证事务的完整性——同一个事务内的所有 DML 事件必须一起提交到 Store：

```java
@Override
public boolean sink(List<CanalEntry.Entry> entries, CanalEntry.Entry entry,
                    String... filterTableNames) {
    
    switch (entry.getEntryType()) {
        case TRANSACTIONBEGIN:
            // 开始新事务
            transactionBuffer.set(new ArrayList<>());
            entries.add(entry);
            return true;
            
        case ROWDATA:
            // 将 DML 事件暂存到事务缓冲区
            List<CanalEntry.Entry> buffer = transactionBuffer.get();
            if (buffer != null) {
                buffer.add(entry);
            }
            return true;  // 不立即提交，等待事务结束
            
        case TRANSACTIONEND:
            // 事务结束，将缓冲的事件批量提交
            List<CanalEntry.Entry> buffer = transactionBuffer.get();
            if (buffer != null && !buffer.isEmpty()) {
                entries.addAll(buffer);
            }
            entries.add(entry);
            transactionBuffer.remove();  // 清理
            return doSink(entries);
            
        default:
            return doSink(entries);
    }
}

private boolean doSink(List<CanalEntry.Entry> entries) {
    if (entries == null || entries.isEmpty()) {
        return true;
    }
    
    // 通过 Disruptor 发布事件到 RingBuffer
    RingBuffer<Event<CanalEntry.Entry>> ringBuffer = disruptor.getRingBuffer();
    long sequence = ringBuffer.next();
    try {
        Event<CanalEntry.Entry> event = ringBuffer.get(sequence);
        event.setData(new ArrayList<>(entries));
    } finally {
        ringBuffer.publish(sequence);
    }
    
    return true;
}
```

## 2.4 AviaterRegexFilter — 正则过滤器

```java
public class AviaterRegexFilter {
    
    private Pattern pattern;
    private String regex;
    
    public AviaterRegexFilter(String regex) {
        this.regex = regex;
        this.pattern = Pattern.compile(regex, Pattern.CASE_INSENSITIVE);
    }
    
    /**
     * 过滤匹配
     * @param value 格式: schema.table
     * @return true 表示匹配成功，事件通过
     */
    public boolean filter(String value) {
        if (value == null || value.isEmpty()) {
            return false;
        }
        return pattern.matcher(value).matches();
    }
}
```

**常用过滤配置示例**：

```properties
# 订阅所有库的所有表
canal.instance.filter.regex = .*\\..*

# 只订阅 test 库
canal.instance.filter.regex = test\\..*

# 只订阅 test 库的 user 和 order 表
canal.instance.filter.regex = test\\.user,test\\.order

# 订阅多个库
canal.instance.filter.regex = test\\..*,prod\\..*

# 黑名单：排除特定表
canal.instance.filter.black.regex = test\\.temp_table,.*\\.backup_.*
```

# 3. EventStore — 事件存储

## 3.1 接口定义

```java
public interface CanalEventStore<T> extends CanalLifeCycle {
    
    /**
     * 存储事件
     */
    void put(List<T> data) throws InterruptedException, CanalStoreException;
    
    /**
     * 获取事件（带回调）
     */
    Events<T> get(BatchAckCallback callback, PositionRange range, 
                  int batchSize, long timeout, TimeUnit unit) 
            throws InterruptedException, CanalStoreException;
    
    /**
     * 获取事件（不带回调，使用位点范围）
     */
    Events<T> get(BatchAckCallback callback, PositionRange range,
                  int batchSize, TimeUnit unit, long timeout, boolean isRaw) 
            throws InterruptedException, CanalStoreException;
    
    /**
     * 确认消费完成，更新消费位点
     */
    void ack(Position position) throws CanalStoreException;
    
    /**
     * 回滚到指定位点（消费异常时使用）
     */
    void rollback(Position position) throws CanalStoreException;
    
    /**
     * 获取当前消费的位点信息（用于重复数据判断）
     */
    Position getLatestAckPosition(PositionRange range) throws CanalStoreException;
}
```

## 3.2 MemoryEventStoreWithBuffer — 基于 Disruptor 的内存存储

这是 Canal 默认的 Store 实现，核心设计如下：

```java
public class MemoryEventStoreWithBuffer extends AbstractCanalStoreScavenge
        implements CanalEventStore<CanalEntry.Entry>, CanalStoreScavenge {

    // RingBuffer 默认大小
    private static final int BUFFER_SIZE = 16384;   // 16 * 1024
    
    // 每个槽位的内存单元大小（字节）
    private static final int BUFFER_MEM_UNIT = 1024;  // 1KB
    
    // Disruptor 实例
    private Disruptor<Event<CanalEntry.Entry>> disruptor;
    
    // RingBuffer
    private RingBuffer<Event<CanalEntry.Entry>> ringBuffer;
    
    // 消费序列号
    private Sequence consumerSequence;
    
    @Override
    public void start() {
        super.start();
        
        // 创建 Disruptor
        disruptor = new Disruptor<>(
            new CanalEventFactory(),
            bufferSize,
            new NamedThreadFactory("store-disruptor"),
            producerType,   // SINGLE 或 MULTI
            waitStrategy    // BlockingWaitStrategy / SleepingWaitStrategy
        );
        
        // 注册消费者处理器
        disruptor.handleEventsWith(new StoreEventHandler());
        
        // 启动 Disruptor
        ringBuffer = disruptor.start();
    }
    
    @Override
    public void stop() {
        disruptor.shutdown();
        super.stop();
    }
}
```

## 3.3 RingBuffer 数据结构

```
┌───────────────────────────────────────────────────────────────┐
│                    Disruptor RingBuffer                       │
│                      (bufferSize = 16384)                     │
│                                                               │
│   ┌─────┬─────┬─────┬─────┬─────┬─────┬──── ─────┬─────┐    │
│   │  0  │  1  │  2  │ ... │  N  │ N+1 │   ...    │16383│    │
│   └─────┴─────┴─────┴─────┴─────┴─────┴──── ─────┴─────┘    │
│     ▲                                   ▲                    │
│     │                                   │                    │
│   Producer                          Consumer                 │
│   Sequence                          Sequence                 │
│   (Put位置)                         (Get位置)                │
│                                                               │
│   可用数据量 = Producer Sequence - Consumer Sequence         │
│   可用空间   = BUFFER_SIZE - 可用数据量                      │
└───────────────────────────────────────────────────────────────┘
```

## 3.4 CanalEventFactory — 事件对象创建

```java
public class CanalEventFactory implements EventFactory<Event<CanalEntry.Entry>> {
    
    @Override
    public Event<CanalEntry.Entry> newInstance() {
        return new CanalEvent();
    }
}

public class CanalEvent implements Event<CanalEntry.Entry> {
    
    // 事件数据（每个槽位的数据对象会被重用）
    private List<CanalEntry.Entry> entries = new ArrayList<>();
    
    // Disruptor 的序列号
    private long sequence;
    
    @Override
    public void clear() {
        // 对象复用：清空但不释放，减少 GC
        entries.clear();
    }
    
    @Override
    public List<CanalEntry.Entry> getData() {
        return entries;
    }
    
    @Override
    public void setData(List<CanalEntry.Entry> data) {
        this.entries = data;
    }
}
```

## 3.5 Put — 写入事件

```java
@Override
public void put(List<CanalEntry.Entry> data) throws InterruptedException {
    
    if (data == null || data.isEmpty()) {
        return;
    }
    
    // 获取 RingBuffer 的下一个可用序号
    long next = ringBuffer.next();
    
    try {
        // 获取槽位
        Event<CanalEntry.Entry> event = ringBuffer.get(next);
        
        // 设置数据
        event.setData(data);
        
    } finally {
        // 发布事件，消费者可以消费
        ringBuffer.publish(next);
    }
}
```

## 3.6 Get — 消费事件（核心逻辑）

这是 EventStore 最复杂的部分，涉及消费位点管理、ACK 回调和超时控制：

```java
@Override
public Events<CanalEntry.Entry> get(BatchAckCallback callback, 
                                     PositionRange range,
                                     int batchSize, long timeout, TimeUnit unit) 
        throws InterruptedException, CanalStoreException {
    
    List<CanalEntry.Entry> result = new ArrayList<>(batchSize);
    long startTime = System.currentTimeMillis();
    long timeoutMillis = unit.toMillis(timeout);
    
    // 获取当前可用的最大序列号
    long availableSequence = ringBuffer.getCursor();
    long nextSequence = consumerSequence.get() + 1;
    
    // 循环读取，直到获取足够的数据或超时
    while (result.size() < batchSize) {
        
        if (nextSequence <= availableSequence) {
            // 有可用数据
            Event<CanalEntry.Entry> event = ringBuffer.get(nextSequence);
            List<CanalEntry.Entry> entries = event.getData();
            
            if (entries != null && !entries.isEmpty()) {
                result.addAll(entries);
            }
            
            nextSequence++;
            
        } else {
            // 无可用数据
            if (result.isEmpty()) {
                // 未获取到任何数据，检查超时
                long elapsed = System.currentTimeMillis() - startTime;
                if (elapsed > timeoutMillis) {
                    break;  // 超时返回空
                }
                
                // 等待新数据
                waitForData(timeoutMillis - elapsed);
                
                // 重新获取可用序列号
                availableSequence = ringBuffer.getCursor();
            } else {
                // 已获取部分数据，达到 batchSize 或超时则返回
                if (result.size() >= batchSize) {
                    break;
                }
                long elapsed = System.currentTimeMillis() - startTime;
                if (elapsed > timeoutMillis) {
                    break;
                }
            }
        }
    }
    
    // 更新消费序列号
    if (!result.isEmpty()) {
        consumerSequence.set(nextSequence - 1);
    }
    
    // 构造返回结果
    return new Events<>(result, range);
}
```

## 3.7 ACK — 消费确认

```java
@Override
public void ack(Position position) throws CanalStoreException {
    
    if (position == null) {
        return;
    }
    
    // 更新 ACK 位点（持久化到 MetaManager）
    metaManager.updateCursor(position);
    latestAckPosition = position;
    
    // 清理已确认的事件数据（释放内存）
    cleanUntil(position);
    
    logger.debug("ack position: {}", position);
}

/**
 * 清理直到指定位点的事件
 */
private void cleanUntil(Position position) {
    // 将已确认的 Entry 从 RingBuffer 中清理
    long cleanSequence = convertPositionToSequence(position);
    
    for (long i = consumerSequence.get() + 1; i <= cleanSequence; i++) {
        Event<CanalEntry.Entry> event = ringBuffer.get(i);
        event.clear();  // 释放数据，供 GC 回收
    }
}
```

## 3.8 Rollback — 消费回滚

```java
@Override
public void rollback(Position position) throws CanalStoreException {
    
    // 重置消费序列号到指定位点
    long rollbackSequence = convertPositionToSequence(position);
    consumerSequence.set(rollbackSequence);
    
    logger.info("rollback to position: {}, sequence: {}", position, rollbackSequence);
}
```

# 4. FileEventStore — 文件持久化存储

当内存不足以缓存事件时，可以使用 FileEventStore 将事件持久化到磁盘：

```java
public class FileEventStore implements CanalEventStore<CanalEntry.Entry> {
    
    // 文件存储目录
    private String storeDir;
    
    // 当前写入文件
    private RandomAccessFile writeFile;
    
    // 当前读取文件
    private RandomAccessFile readFile;
    
    // 文件索引（位点 → 文件偏移量的映射）
    private TreeMap<Long, FilePosition> positionIndex;
    
    @Override
    public void put(List<CanalEntry.Entry> data) throws CanalStoreException {
        
        // 序列化数据
        byte[] serialized = Serializer.serialize(data);
        
        // 写入文件
        writeFile.seek(writeFile.length());
        writeFile.writeInt(serialized.length);
        writeFile.write(serialized);
        
        // 更新索引
        long currentPosition = writeFile.getFilePointer();
        positionIndex.put(currentPosition, new FilePosition(fileName, currentPosition));
    }
    
    @Override
    public Events<CanalEntry.Entry> get(BatchAckCallback callback,
                                         PositionRange range,
                                         int batchSize, long timeout, TimeUnit unit) {
        // 从文件中按位点读取数据
        // ...
    }
}
```

# 5. MixedEventStore — 混合存储

MixedEventStore 结合了内存和文件存储的优点：

```java
public class MixedEventStore implements CanalEventStore<CanalEntry.Entry> {
    
    // 内存存储（热数据）
    private MemoryEventStoreWithBuffer memoryStore;
    
    // 文件存储（冷数据）
    private FileEventStore fileStore;
    
    // 切换阈值
    private long memoryHighWatermark = 80;  // 内存使用 80% 时开始写入文件
    private long memoryLowWatermark = 20;   // 内存使用降到 20% 时停止写文件
    
    @Override
    public void put(List<CanalEntry.Entry> data) throws CanalStoreException {
        
        if (memoryStore.isFull(memoryHighWatermark)) {
            // 内存使用率超过高水位，写入文件
            fileStore.put(data);
        } else {
            // 内存充足，写入内存
            memoryStore.put(data);
        }
    }
    
    @Override
    public Events<CanalEntry.Entry> get(BatchAckCallback callback,
                                         PositionRange range,
                                         int batchSize, long timeout, TimeUnit unit) {
        // 优先从内存读取
        Events<CanalEntry.Entry> events = memoryStore.get(callback, range, batchSize, 0, unit);
        
        if (events.isEmpty()) {
            // 内存中没有数据，从文件读取
            events = fileStore.get(callback, range, batchSize, timeout, unit);
        }
        
        return events;
    }
}
```

# 6. Disruptor 配置优化

## 6.1 WaitStrategy 选择

Disruptor 提供了多种等待策略，对性能影响很大：

| 策略 | CPU 占用 | 延迟 | 适用场景 |
|------|---------|------|---------|
| `BlockingWaitStrategy` | 低 | 较高 | 对延迟不敏感的默认选择 |
| `SleepingWaitStrategy` | 中等 | 中等 | 平衡 CPU 和延迟 |
| `YieldingWaitStrategy` | 高 | 低 | 低延迟场景（需要更多 CPU 核） |
| `BusySpinWaitStrategy` | 极高 | 极低 | 对延迟极度敏感（多核 CPU） |

```java
// Canal 默认配置
// 如果 RingBuffer 有数据 → YieldingWaitStrategy（低延迟）
// 如果 RingBuffer 无数据 → BlockingWaitStrategy（省 CPU）
WaitStrategy waitStrategy = new PhasedBackoffWaitStrategy(
    1, 1, TimeUnit.MILLISECONDS,
    new YieldingWaitStrategy(),
    new BlockingWaitStrategy()
);
```

## 6.2 内存配置

```properties
# conf/canal.properties

# RingBuffer 大小
canal.instance.memory.buffer.size = 16384

# 每个槽位的内存单元（字节）
canal.instance.memory.buffer.memunit = 1024

# 内存中可缓存的最大 Entry 数
canal.instance.memory.batch.mode = MEMSIZE

# 是否为 dml 事件返回 -1 的 batchId（用于不支持 ack 的场景）
canal.instance.get.without.ack.ignore = true
```

# 7. PositionRange — 位点范围

`PositionRange` 定义了每次 Get 操作的位点范围，用于防止重复消费：

```java
public class PositionRange {
    
    private EntryPosition start;    // 起始位点（包含）
    private EntryPosition end;      // 结束位点（不包含）
    
    // 构造函数
    // start = latestAckPosition (上次 ACK 的位点)
    // end   = currentPosition (当前的写入位点)
}
```

**位点范围的作用**：

```
写指针（Producer）→→→→→→→→→→→→→→→→→
                  │  范围1   │  范围2   │
                  │  (已ACK) │(可消费)  │
读指针（Consumer）→→→→→→→→→→→→→→→→→
                  ↑          ↑
               lastAck   currentPos
               
PositionRange = {start: lastAck, end: currentPos}
```

# 8. ACK 与 Rollback 机制

## 8.1 正常消费流程

```java
// Client 端的标准消费流程
CanalConnector connector = CanalConnectors.newSingleConnector(...);
connector.connect();
connector.subscribe(".*\\..*");

long batchId = -1;
while (running) {
    // 1. 获取一批数据（不自动 ACK）
    Message message = connector.getWithoutAck(1000);
    batchId = message.getId();
    
    // 2. 处理数据
    try {
        processEntries(message.getEntries());
        // 3. 处理成功 → ACK
        connector.ack(batchId);
    } catch (Exception e) {
        // 4. 处理失败 → Rollback
        connector.rollback(batchId);
        // 下次获取时会重新获取这一批数据
    }
}
```

## 8.2 Server 端的 ACK 处理

```java
// CanalServerWithNetty 中的 ACK handler
public void ack(ClientIdentity clientIdentity, long batchId) {
    
    // 1. 根据 batchId 查找对应的位点
    Position position = batchIdToPositionMap.get(batchId);
    if (position == null) {
        throw new CanalServerException("unknown batchId: " + batchId);
    }
    
    // 2. 更新消费者位点
    CanalInstance instance = getCanalInstance(clientIdentity.getDestination());
    instance.getEventStore().ack(position);
    
    // 3. 清理 batchId 映射
    batchIdToPositionMap.remove(batchId);
    
    logger.debug("client {} ack batchId: {}, position: {}", 
                 clientIdentity, batchId, position);
}
```

## 8.3 位点持久化

```java
// MetaManager 中的位点持久化
public void updateCursor(Position position) {
    
    LogPosition logPosition = (LogPosition) position;
    EntryPosition entryPosition = logPosition.getPostion();
    
    // 1. 更新内存中的位点
    this.cursor = logPosition;
    
    // 2. 持久化到 ZooKeeper（HA 模式）
    if (zkClient != null) {
        String path = ZKPaths.CANAL_DESTINATION_ROOT + "/" + destination + "/cursor";
        byte[] data = JsonUtils.marshalToString(entryPosition).getBytes();
        zkClient.setData(path, data);
    }
    
    // 3. 持久化到本地文件（单机模式）
    if (localFile != null) {
        writePositionToFile(entryPosition);
    }
}
```

# 9. Get 请求超时与轮询

## 9.1 Client 轮询模式

```java
public class CanalClientPoller implements Runnable {
    
    private CanalConnector connector;
    private long timeout = 1000;  // 1s
    private int batchSize = 1000;
    
    @Override
    public void run() {
        while (running) {
            try {
                // 阻塞获取（有数据立即返回，无数据等待 timeout）
                Message message = connector.getWithoutAck(batchSize, timeout, TimeUnit.MILLISECONDS);
                
                if (message.getId() != -1 && !message.getEntries().isEmpty()) {
                    processMessage(message);
                    connector.ack(message.getId());
                }
                
                // 如果 message.getId() == -1 表示超时无数据，自动进行下一次轮询
                
            } catch (Exception e) {
                handleException(e);
            }
        }
    }
}
```

## 9.2 Server 端超时机制

```java
// Server 端在 Netty Handler 中处理 Get 请求
public void handleGet(Channel channel, GetMessage getMessage) {
    
    CanalInstance instance = getCanalInstance(getMessage.getDestination());
    CanalEventStore<CanalEntry.Entry> store = instance.getEventStore();
    
    // 构造位点范围
    PositionRange range = new PositionRange();
    range.setStart(getMessage.getStartPosition());
    range.setEnd(getMessage.getEndPosition());
    
    // 从 Store 获取数据
    Events<CanalEntry.Entry> events = store.get(
        new BatchAckCallback() {
            @Override
            public void onSuccess(Events<Entry> events) {
                // 数据获取成功，返回给客户端
                sendMessage(channel, events);
            }
        },
        range,
        getMessage.getBatchSize(),
        getMessage.getTimeout(),
        getMessage.getUnit()
    );
}
```

# 10. 内存回收与防溢出

Canal Store 实现了内存保护机制，防止因为 Client 消费过慢导致内存溢出：

```java
// MemoryEventStoreWithBuffer 中的内存保护
public class MemoryEventStoreWithBuffer {
    
    // 内存限制（默认 1GB）
    private long memoryLimit = 1024 * 1024 * 1024;  // 1GB
    
    // 当前内存使用量
    private AtomicLong memoryUsed = new AtomicLong(0);
    
    @Override
    public void put(List<CanalEntry.Entry> data) throws InterruptedException {
        
        // 计算数据大小
        long dataSize = calculateDataSize(data);
        
        // 等待直到有足够的内存
        while (memoryUsed.get() + dataSize > memoryLimit) {
            if (!running) {
                throw new CanalStoreException("store is not running");
            }
            // 阻塞等待消费者释放内存
            Thread.sleep(100);
        }
        
        memoryUsed.addAndGet(dataSize);
        
        // 写入 RingBuffer
        // ...
    }
    
    @Override
    public void ack(Position position) {
        // ACK 后释放内存
        long dataSize = calculatePositionDataSize(position);
        memoryUsed.addAndGet(-dataSize);
        
        // ...
    }
}
```
