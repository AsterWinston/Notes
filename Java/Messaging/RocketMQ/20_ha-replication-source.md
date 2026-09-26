
> 源码版本：RocketMQ 4.9.x
> 核心模块：`store/src/main/java/org/apache/rocketmq/store/ha/`

# 1. HA 模块概述

```
HA 模块负责 Broker Master → Slave 之间的数据同步

核心组件:
┌───────────────────────────────────────────────────┐
│  Master 端                                        │
│  ├── HAService            — HA 服务主入口          │
│  ├── AcceptSocketService  — 接受 Slave 连接        │
│  ├── GroupTransferService — 同步双写等待           │
│  └── HAClient             — 对每台 Slave 的链接    │
│                              (实际类在 HAClient)   │
└───────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────┐
│  Slave 端                                         │
│  ├── HAService            — HA 服务主入口          │
│  └── HAClient             — 连接 Master 同步数据   │
└───────────────────────────────────────────────────┘
```

# 2. HAService 初始化与启动

```java
// DefaultMessageStore.start() 中创建 HAService
if (!messageStoreConfig.isEnableDLegerCommitLog()) {
    this.haService = new HAService(this);
}

// HAService 构造函数
public HAService(final DefaultMessageStore defaultMessageStore) {
    this.defaultMessageStore = defaultMessageStore;
    
    // Master 端组件
    this.acceptSocketService = new AcceptSocketService(
        messageStoreConfig.getHaListenPort());  // 默认 10912
    
    this.groupTransferService = new GroupTransferService();

    // Slave 端组件
    this.haClient = new HAClient();
}

// HAService.start()
public void start() throws Exception {
    // 1. Master 启动 AcceptSocketService（接受 Slave 连接请求）
    this.acceptSocketService.beginAccept();
    this.acceptSocketService.start();

    // 2. 启动 GroupTransferService (Master 同步等待)
    this.groupTransferService.start();

    // 3. Slave 启动 HAClient (Slave 连接 Master 同步数据)
    this.haClient.start();
}
```

# 3. Master 端源码解析

## 3.1 AcceptSocketService — 接受 Slave 连接

```java
class AcceptSocketService extends ServiceThread {
    private final SocketAddress socketAddressListen;
    private ServerSocketChannel serverSocketChannel;
    private Selector selector;

    @Override
    public void run() {
        log.info("HAServer AcceptSocketService begin");

        while (!this.isStopped()) {
            // 1. 监听连接的 Selector
            this.selector.select(1000);
            Set<SelectionKey> selected = this.selector.selectedKeys();

            for (SelectionKey key : selected) {
                if (key.isAcceptable()) {
                    // 2. 接受 Slave 连接
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel sc = server.accept();

                    // 3. 为每个 Slave 创建一对读写线程
                    //    - HAConnection (管理连接)
                    //    - WriteSocketService (Master → Slave 写数据)
                    //    - ReadSocketService (Master ← Slave 读 ACK)
                    HAConnection conn = new HAConnection(HAService.this, sc);
                    conn.start();
                    HAService.this.addConnection(conn);
                }
            }
            selected.clear();
        }
    }
}
```

## 3.2 HAConnection 与 WriteSocketService

```java
// HAConnection — 每个 Slave 连接的管理器
public class HAConnection {
    private final HAService haService;
    private final SocketChannel socketChannel;
    private final WriteSocketService writeSocketService;  // 向 Slave 写数据
    private final ReadSocketService readSocketService;    // 从 Slave 读 ACK

    public HAConnection(HAService haService, SocketChannel socketChannel) {
        this.haService = haService;
        this.socketChannel = socketChannel;
        this.writeSocketService = new WriteSocketService(socketChannel);
        this.readSocketService = new ReadSocketService(socketChannel, this);
    }
}

// ================== WriteSocketService (Master → Slave 写数据) ==================

class WriteSocketService extends ServiceThread {
    private final SocketChannel socketChannel;

    @Override
    public void run() {
        log.info("WriteSocketService start");

        while (!this.isStopped()) {
            // 1. 当前 Slave 已经同步到了哪个偏移量
            // slaveRequestOffset — 来自 Slave 通报的 ACK (ReadSocketService 解析)
            long slaveRequestOffset = HAConnection.this.slaveRequestOffset;

            // 2. Master 当前的 CommitLog 最大偏移量
            long masterMaxOffset = defaultMessageStore.getCommitLog().getMaxOffset();

            // 3. 计算需要同步的数据量
            long diff = masterMaxOffset - slaveRequestOffset;

            if (diff > 0) {
                // ===== 有数据需要同步 =====

                // 获取数据
                SelectMappedBufferResult selectResult = defaultMessageStore
                    .getCommitLog().getData(slaveRequestOffset);
                if (selectResult != null) {
                    int size = Math.min(maxTransferSizeByByte, selectResult.getSize());
                    long nextOffset = slaveRequestOffset + size;

                    // 构造 HA 报文 Header:
                    // ┌─────────────────────────────────────┐
                    // │ slaveMaxOffset (8B) — Slave 已知偏移│
                    // │ bodySize (4B)      — 数据体大小     │
                    // └─────────────────────────────────────┘
                    // 后面紧跟: CommitLog 数据

                    ByteBuffer header = ByteBuffer.allocate(8);
                    header.putLong(slaveRequestOffset);
                    header.flip();

                    // 写 Header
                    this.socketChannel.write(header);

                    // 写 Data
                    this.dispatchReadRequest(pos, size);
                    transferSize += size;

                    // 更新 nextTransferFromWhere (Master 记录的 Slave 当前偏移)
                    HAConnection.this.nextTransferFromWhere = nextOffset;
                }
            } else {
                // ===== 没有数据 =====
                // 等待一会
                synchronized (this) {
                    this.wait(Math.min(intervalOnNoData, diff));
                }
            }
        }
    }
}

// ================== ReadSocketService (Master ← Slave 读 ACK) ==================

class ReadSocketService extends ServiceThread {
    private final SocketChannel socketChannel;

    @Override
    public void run() {
        while (!this.isStopped()) {
            // 读取 Slave 发来的 ACK
            // 从 socket 读取 8 字节 → slaveMaxOffset
            // Slave 报告它已经同步到了哪个偏移量
            ByteBuffer byteBuffer = ByteBuffer.allocate(READ_MAX_BUFFER_SIZE);
            int readSize = this.socketChannel.read(byteBuffer);
            if (readSize > 0) {
                byteBuffer.flip();
                long slaveMaxOffset = byteBuffer.getLong();  // 8B
                HAConnection.this.slaveRequestOffset = slaveMaxOffset;

                // 通知 GroupTransferService (同步写等待这个偏移量)
                HAService.this.notifyTransferSome(slaveMaxOffset);
            }
        }
    }
}
```

## 3.3 GroupTransferService — 同步复制等待

```java
// Master: 同步复制时使用 (GroupCommit → CountDownLatch 等待 Slave ACK)

class GroupTransferService extends ServiceThread {
    // 等待列表
    private final List<CommitLog.GroupCommitRequest> requestsWrite = ...;
    private final List<CommitLog.GroupCommitRequest> requestsRead = ...;

    @Override
    public void run() {
        while (!this.isStopped()) {
            // 1. 交换读写队列 (Double Buffering)
            synchronized (this.requestsWrite) {
                // 交换
            }

            // 2. 逐个检查 Slave 是否已经同步到请求的偏移量
            for (CommitLog.GroupCommitRequest req : requestsRead) {
                // 所有 Slaves 中最大的同步偏移量
                long maxSlaveOffset = HAService.this.pushSlaveMaxOffset();

                // 如果 Slave 的同步位置 >= 请求的偏移量 → 同步完成
                if (maxSlaveOffset >= req.getNextOffset()) {
                    req.getCountDownLatch().countDown();  // 唤醒等待线程
                    toRemove.add(req);
                }
            }

            // 3. 如果全部完成，则等待；否则快速循环
            if (toRemove.isEmpty()) {
                this.waitForRunning(10);
            } else {
                requestsRead.removeAll(toRemove);
            }
        }
    }
}
```

# 4. Slave 端源码解析

## 4.1 HAClient — Slave 端同步

```java
// Slave 端模拟 HA 请求的逻辑
// 连接到 Master，持续读取数据

class HAClient extends ServiceThread {
    // Master 地址
    private final String masterAddress;
    private SocketChannel socketChannel;

    // Slave 当前 CommitLog 最大偏移量
    private long currentOffset = 0;

    @Override
    public void run() {
        log.info("HAClient started");

        while (!this.isStopped()) {
            // 1. 连接 Master (如果未连接)
            if (connectMaster()) {

                // 2. 通知 Master 当前 Slave 的偏移量
                // Slave → Master 发一个自己当前的 maxOffset
                if (this.isTimeToReportOffset()) {
                    reportSlaveMaxOffset(this.currentReportedOffset);
                    this.lastReportTimestamp = System.currentTimeMillis();
                }

                // 3. 从 Master 读取数据
                // 读取 8 字节 header = 下一个偏移量
                // 读取 4 字节 bodyLength
                // 读取 body
                ByteBuffer headerBuffer = ByteBuffer.allocate(8);
                socketChannel.read(headerBuffer);
                headerBuffer.flip();
                long masterMaxOffset = headerBuffer.getLong();

                // 4. 读取 CommitLog 数据
                // ...
                // 5. 写入 Slave 本地的 CommitLog
                // defaultMessageStore.getCommitLog().appendData(body)
                currentOffset = masterMaxOffset + bodyLength;
            }
        }
    }
}
```

# 5. CommitLog 中的 HA 流程

```java
// CommitLog.handleHA() — 消息写入后的 HA 处理
public void handleHA(AppendMessageResult result, MessageExtBrokerInner msg, 
        int tranType) {
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
            || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {

        // 同步复制模式——需要在返回 Producer 前等待 Slave 确认
        if (BrokerRole.SYNC_MASTER == this.defaultMessageStore
                .getMessageStoreConfig().getBrokerRole()) {

            HAService haService = this.defaultMessageStore.getHaService();
            if (haService != null) {
                // 构建同步请求
                GroupCommitRequest request = new GroupCommitRequest(
                    result.getWroteOffset() + result.getWroteBytes());
                
                // 放入等待队列
                haService.putRequest(request);

                // 等待 Slave ACK (最多等 5s)
                request.waitForFlush(this.defaultMessageStore
                    .getMessageStoreConfig().getSyncFlushTimeout());
            }
        }
        // 异步复制模式——不等待
        else {
            // 通知 HAService 已有新数据（后台传输）
            if (haService != null) {
                haService.notifyNewData();
            }
        }
    }
}
```

# 6. HA 协议格式

```
Master → Slave 数据推送：
┌──────────────────────────────────────┐
│ Header (12B)                         │
│  ├── slaveMaxOffset (8B)             │
│  └── bodySize (4B)                   │
├──────────────────────────────────────┤
│ Body (bodySize bytes)                │
│  └── CommitLog 增量数据              │
└──────────────────────────────────────┘

Slave → Master ACK：
┌──────────────────────────────────────┐
│ slaveMaxOffset (8B)                  │
└──────────────────────────────────────┘
```

# 7. 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `haListenPort` | 10912 | HA 服务端口 |
| `haSendHeartbeatInterval` | 5000ms | 心跳间隔 |
| `haHousekeepingInterval` | 20000ms | 清理间隔 |
| `haTransferBatchSize` | 32768 (32KB) | 每次传输最大字节数 |
| `haMasterAddress` | - | Slave 指定 Master 地址 |

# 8. 参考

- [HAService 源码](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/ha/HAService.java)
- [HAConnection 源码](https://github.com/apache/rocketmq/blob/master/store/src/main/java/org/apache/rocketmq/store/ha/HAConnection.java)
