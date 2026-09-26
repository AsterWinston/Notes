
# 1. Canal Client 概述

Canal Client 是 Canal 官方提供的 Java 客户端 SDK，用于从 Canal Server 订阅和消费增量数据。它封装了与 Canal Server 的通信协议，提供了简洁的编程接口。

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.7</version>
</dependency>
```

# 2. 连接器类型

## 2.1 CanalConnector 接口

```java
public interface CanalConnector {
    
    // 连接到 Canal Server
    void connect() throws CanalClientException;
    
    // 断开连接
    void disconnect() throws CanalClientException;
    
    // 检查连接状态
    boolean checkValid() throws CanalClientException;
    
    // 订阅过滤规则（如 ".*\\..*" 订阅所有表）
    void subscribe(String filter) throws CanalClientException;
    
    // 取消订阅
    void unsubscribe() throws CanalClientException;
    
    // ========== 数据获取方式 ==========
    
    // 获取数据（自动 ACK）
    Message get(int batchSize) throws CanalClientException;
    
    // 获取数据（指定超时，自动 ACK）
    Message get(int batchSize, Long timeout, TimeUnit unit) throws CanalClientException;
    
    // 获取数据（不自动 ACK，需要手动 ack/rollback）
    Message getWithoutAck(int batchSize) throws CanalClientException;
    
    // 获取数据（不自动 ACK，指定超时）
    Message getWithoutAck(int batchSize, Long timeout, TimeUnit unit) 
            throws CanalClientException;
    
    // ========== ACK 和 Rollback ==========
    
    // 确认消费
    void ack(long batchId) throws CanalClientException;
    
    // 回滚（重新消费）
    void rollback(long batchId) throws CanalClientException;
    
    // 回滚到上次 ACK 的位置
    void rollback() throws CanalClientException;
}
```

## 2.2 三种连接模式

```java
// 1. 单机模式：直接连接单个 Canal Server
CanalConnector connector = CanalConnectors.newSingleConnector(
    new InetSocketAddress("127.0.0.1", 11111),
    "example",           // destination（实例名）
    "",                  // username（默认空）
    ""                   // password（默认空）
);

// 2. 集群模式：通过 ZooKeeper 自动发现 Canal Server
CanalConnector connector = CanalConnectors.newClusterConnector(
    "127.0.0.1:2181",    // ZooKeeper 地址
    "example",           // destination
    "",                  // username
    ""                   // password
);

// 3. 自定义模式：可自定义 HA 策略
CanalConnector connector = CanalConnectors.newClusterConnector(
    "127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183",
    "example",
    "username",
    "password"
);
```

## 2.3 集群模式的工作原理

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Canal Server │    │ Canal Server │    │ Canal Server │
│   Node-1     │    │   Node-2     │    │   Node-3     │
│  (Running)   │    │  (Standby)   │    │  (Standby)   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │  ZooKeeper  │
                    │   Cluster   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │Canal Client │
                    │(自动发现)    │
                    └─────────────┘
```

Client 通过 ZK 获取当前 Running 的 Server，自动连接。当 Server 发生切换时，Client 自动重连到新 Server。

# 3. 消费模式详解

## 3.1 自动 ACK 模式

最简单的消费模式，获取数据后自动确认消费：

```java
public class AutoAckConsumer {
    
    private CanalConnector connector;
    
    public void start() {
        connector.connect();
        connector.subscribe("test\\..*");
        connector.rollback();
        
        while (running) {
            // 获取数据，自动 ACK
            Message message = connector.get(1000, 1L, TimeUnit.SECONDS);
            
            if (message.getId() == -1 || message.getEntries().isEmpty()) {
                Thread.sleep(100);
                continue;
            }
            
            // 处理数据
            processEntries(message.getEntries());
            
            // 不需要显式 ACK，get() 方法内部已自动 ACK
        }
    }
}
```

**特点**：
- 优点：简单，不易出错
- 缺点：如果在处理过程中异常，数据无法回滚重试
- 适用场景：对数据一致性要求不高的场景

## 3.2 手动 ACK 模式（推荐）

```java
public class ManualAckConsumer {
    
    private CanalConnector connector;
    private long lastBatchId = -1;
    
    public void start() {
        connector.connect();
        connector.subscribe(".*\\..*");
        connector.rollback();  // 回滚到上次 ACK 位置
        
        while (running) {
            try {
                // 获取数据（手动 ACK）
                Message message = connector.getWithoutAck(1000, 5L, TimeUnit.SECONDS);
                long batchId = message.getId();
                
                if (batchId == -1 || message.getEntries().isEmpty()) {
                    Thread.sleep(200);
                    continue;
                }
                
                // 处理数据
                processBatch(message.getEntries());
                
                // 处理成功 → ACK
                connector.ack(batchId);
                lastBatchId = batchId;
                
            } catch (Exception e) {
                logger.error("process error, rollback", e);
                // 处理失败 → Rollback
                connector.rollback();
                // 下次 getWithoutAck 将重新获取这批数据
            }
        }
    }
}
```

**特点**：
- 优点：处理失败可回滚，保证数据不丢失
- 缺点：需要手动管理 ACK/Rollback
- 适用场景：生产环境推荐，需要保证数据可靠性的场景

## 3.3 Batch Ack 回调模式

```java
public class BatchAckConsumer {
    
    private CanalConnector connector;
    
    public void start() {
        connector.connect();
        connector.subscribe(".*\\..*");
        connector.rollback();
        
        while (running) {
            Message message = connector.getWithoutAck(1000, 5L, TimeUnit.SECONDS);
            
            if (message.getId() == -1 || message.getEntries().isEmpty()) {
                Thread.sleep(200);
                continue;
            }
            
            // 批量处理每一行数据
            for (Entry entry : message.getEntries()) {
                try {
                    if (entry.getEntryType() == EntryType.ROWDATA) {
                        processRow(entry);
                    }
                } catch (Exception e) {
                    logger.error("process row error", e);
                    // 单行失败也可以继续（取决于业务需求）
                }
            }
            
            // 全部行处理完后统一 ACK
            connector.ack(message.getId());
        }
    }
}
```

# 4. 订阅过滤规则

## 4.1 过滤语法

```java
// 订阅所有库的所有表
connector.subscribe(".*\\..*");

// 只订阅 test 库
connector.subscribe("test\\..*");

// 只订阅 test 库的 user 和 order 表
connector.subscribe("test\\.user,test\\.order");

// 使用正则分组
connector.subscribe("(test|prod)\\..*");

// 订阅以 t_ 开头的表
connector.subscribe(".*\\.t_.*");

// 动态修改订阅规则（不需要重连）
connector.subscribe("prod\\.order,prod\\.payment");
```

## 4.2 过滤切换时的注意事项

```java
public class FilterSwitchHandler {
    
    private CanalConnector connector;
    
    /**
     * 安全切换过滤规则
     * 先 unsubscribe，再 subscribe
     */
    public void switchFilter(String newFilter) {
        try {
            // 1. 取消当前订阅
            connector.unsubscribe();
            
            // 2. 等待当前批次处理完
            drainRemaining();
            
            // 3. 订阅新规则
            connector.subscribe(newFilter);
            
            logger.info("filter switched to: {}", newFilter);
            
        } catch (Exception e) {
            logger.error("filter switch failed", e);
        }
    }
    
    private void drainRemaining() {
        // 消费完 unsubscribe 之前的剩余数据
        Message msg;
        do {
            msg = connector.getWithoutAck(100);
            if (msg.getId() != -1 && !msg.getEntries().isEmpty()) {
                connector.ack(msg.getId());
            }
        } while (msg.getId() != -1);
    }
}
```

# 5. Message 与 Entry 解析

## 5.1 数据结构

```
Message
├── long id                    # Batch ID（ACK 时使用）
├── List<Entry> entries        # 事件列表
│   └── Entry
│       ├── Header header      # 事件头
│       │   ├── logfileName    # binlog 文件名
│       │   ├── logfileOffset  # binlog 偏移量
│       │   ├── schemaName     # 库名
│       │   ├── tableName      # 表名
│       │   ├── eventType      # INSERT/UPDATE/DELETE/...
│       │   ├── executeTime    # 执行时间
│       │   └── gtid           # GTID
│       ├── EntryType entryType# ROWDATA/TRANSACTIONBEGIN/TRANSACTIONEND/HEARTBEAT
│       └── byte[] storeValue  # Protobuf 序列化的 RowChange
└── boolean raw                # 是否是原始数据
```

## 5.2 完整的解析示例

```java
public class MessageParser {
    
    public static void parseAndPrint(Message message) {
        
        List<Entry> entries = message.getEntries();
        
        for (Entry entry : entries) {
            
            // 忽略事务标记
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN
                    || entry.getEntryType() == EntryType.TRANSACTIONEND) {
                continue;
            }
            
            // 处理心跳
            if (entry.getEntryType() == EntryType.HEARTBEAT) {
                handleHeartbeat(entry);
                continue;
            }
            
            // 处理行数据
            if (entry.getEntryType() == EntryType.ROWDATA) {
                try {
                    RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
                    processRowChange(entry.getHeader(), rowChange);
                } catch (InvalidProtocolBufferException e) {
                    throw new RuntimeException("parse row change failed", e);
                }
            }
        }
    }
    
    private static void processRowChange(Header header, RowChange rowChange) {
        EventType eventType = rowChange.getEventType();
        
        // 处理 DDL
        if (rowChange.getIsDdl()) {
            System.out.printf("DDL: %s.%s %s%n", 
                header.getSchemaName(), header.getTableName(), rowChange.getSql());
            return;
        }
        
        // 处理 DML
        System.out.printf("--- %s.%s [%s] ---%n",
            header.getSchemaName(), header.getTableName(), eventType);
        
        for (RowData rowData : rowChange.getRowDatasList()) {
            
            switch (eventType) {
                case INSERT:
                    System.out.print(" INSERT: ");
                    printColumns(rowData.getAfterColumnsList());
                    break;
                    
                case UPDATE:
                    System.out.print(" UPDATE before: ");
                    printColumns(rowData.getBeforeColumnsList());
                    System.out.print(" UPDATE after:  ");
                    printColumns(rowData.getAfterColumnsList());
                    break;
                    
                case DELETE:
                    System.out.print(" DELETE: ");
                    printColumns(rowData.getBeforeColumnsList());
                    break;
                    
                default:
                    break;
            }
        }
    }
    
    private static void printColumns(List<Column> columns) {
        Map<String, String> map = columns.stream()
            .filter(c -> c.getUpdated() || c.getIsKey())
            .collect(Collectors.toMap(
                Column::getName,
                c -> c.getIsNull() ? "NULL" : c.getValue(),
                (a, b) -> b,
                LinkedHashMap::new
            ));
        System.out.println(JsonUtils.toJson(map));
    }
    
    private static void handleHeartbeat(Entry entry) {
        // 心跳时间戳在 executeTime 中
        long ts = entry.getHeader().getExecuteTime();
        logger.debug("heartbeat at {}", ts);
    }
}
```

# 6. 连接管理与重连

## 6.1 自动重连封装

```java
public class CanalClientTemplate {
    
    private CanalConnector connector;
    private String destination;
    private String filter;
    private volatile boolean running = false;
    
    // 重连配置
    private int maxRetries = 3;
    private long retryInterval = 3000;  // 3s
    
    public CanalClientTemplate(String zkServers, String destination, String filter) {
        this.destination = destination;
        this.filter = filter;
        this.connector = CanalConnectors.newClusterConnector(zkServers, destination, "", "");
    }
    
    /**
     * 启动消费（自动重连）
     */
    public void start(CanalEntryHandler handler) {
        running = true;
        
        while (running) {
            try {
                connectAndConsume(handler);
            } catch (Exception e) {
                logger.error("consume error, will retry", e);
                sleep(retryInterval);
                
                // 重建连接器
                reconnect();
            }
        }
    }
    
    private void connectAndConsume(CanalEntryHandler handler) throws InterruptedException {
        connector.connect();
        connector.subscribe(filter);
        connector.rollback();
        
        logger.info("canal client connected, filter: {}", filter);
        
        while (running) {
            Message message = connector.getWithoutAck(1000, 5L, TimeUnit.SECONDS);
            long batchId = message.getId();
            
            if (batchId == -1 || message.getEntries().isEmpty()) {
                continue;
            }
            
            try {
                for (Entry entry : message.getEntries()) {
                    if (entry.getEntryType() == EntryType.ROWDATA) {
                        handler.handle(entry);
                    }
                }
                connector.ack(batchId);
            } catch (Exception e) {
                connector.rollback(batchId);
                throw e;  // 触发外层重连
            }
        }
    }
    
    private void reconnect() {
        try {
            connector.disconnect();
        } catch (Exception ignored) {}
        connector = CanalConnectors.newClusterConnector(
            zkServers, destination, "", "");
    }
    
    public void stop() {
        running = false;
        connector.disconnect();
    }
}
```

## 6.2 Spring Boot 集成示例

```java
@Component
public class CanalClientRunner implements ApplicationRunner {
    
    @Value("${canal.zkServers:127.0.0.1:2181}")
    private String zkServers;
    
    @Value("${canal.destination:example}")
    private String destination;
    
    @Value("${canal.filter:.*\\..*}")
    private String filter;
    
    private CanalConnector connector;
    private volatile boolean running;
    
    @Override
    public void run(ApplicationArguments args) {
        running = true;
        
        new Thread(this::consume, "canal-consumer").start();
    }
    
    private void consume() {
        connector = CanalConnectors.newClusterConnector(zkServers, destination, "", "");
        
        while (running) {
            try {
                connector.connect();
                connector.subscribe(filter);
                connector.rollback();
                
                while (running) {
                    Message message = connector.getWithoutAck(1000, 5L, TimeUnit.SECONDS);
                    if (message.getId() != -1 && !message.getEntries().isEmpty()) {
                        processMessage(message);
                        connector.ack(message.getId());
                    }
                }
            } catch (Exception e) {
                log.error("canal consume error", e);
                sleep(3000);
            } finally {
                try { connector.disconnect(); } catch (Exception ignored) {}
            }
        }
    }
    
    @PreDestroy
    public void shutdown() {
        running = false;
        if (connector != null) connector.disconnect();
    }
}
```

# 7. 批量处理与性能优化

## 7.1 批量写入示例

```java
public class BatchProcessor {
    
    private static final int BATCH_SIZE = 500;
    
    /**
     * 按表名分组，批量写入
     */
    public void batchProcess(Message message) {
        
        // 按表名分组
        Map<String, List<RowChangeData>> groupedByTable = new HashMap<>();
        
        for (Entry entry : message.getEntries()) {
            if (entry.getEntryType() != EntryType.ROWDATA) continue;
            
            String key = entry.getHeader().getSchemaName() + "." 
                        + entry.getHeader().getTableName();
            
            RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
            
            for (RowData rowData : rowChange.getRowDatasList()) {
                RowChangeData data = new RowChangeData();
                data.setEventType(rowChange.getEventType());
                data.setColumns(convertColumns(rowData));
                
                groupedByTable.computeIfAbsent(key, k -> new ArrayList<>()).add(data);
            }
        }
        
        // 批量处理每张表的数据
        for (Map.Entry<String, List<RowChangeData>> tableEntry : groupedByTable.entrySet()) {
            String tableName = tableEntry.getKey();
            List<RowChangeData> rows = tableEntry.getValue();
            
            // 分小批次处理（避免单次写入过大）
            for (int i = 0; i < rows.size(); i += BATCH_SIZE) {
                int end = Math.min(i + BATCH_SIZE, rows.size());
                List<RowChangeData> batch = rows.subList(i, end);
                
                processBatch(tableName, batch);
            }
        }
    }
}
```

# 8. 多线程消费

```java
public class MultiThreadConsumer {
    
    private CanalConnector connector;
    private ExecutorService executor;
    
    // 每个线程独立消费
    private static final int THREAD_COUNT = 4;
    
    public void start() {
        connector.connect();
        connector.subscribe(".*\\..*");
        connector.rollback();
        
        executor = Executors.newFixedThreadPool(THREAD_COUNT);
        
        for (int i = 0; i < THREAD_COUNT; i++) {
            executor.submit(new ConsumerTask());
        }
    }
    
    class ConsumerTask implements Runnable {
        @Override
        public void run() {
            while (running) {
                // 注意：多线程共用同一个 connector 需要同步
                synchronized (connector) {
                    try {
                        Message message = connector.getWithoutAck(1000);
                        
                        if (message.getId() == -1 || message.getEntries().isEmpty()) {
                            continue;
                        }
                        
                        // 多线程并行处理 Entry
                        message.getEntries().parallelStream()
                            .filter(e -> e.getEntryType() == EntryType.ROWDATA)
                            .forEach(e -> processEntry(e));
                        
                        connector.ack(message.getId());
                        
                    } catch (Exception e) {
                        connector.rollback();
                    }
                }
            }
        }
    }
}
```

> **注意**：多线程消费时，同一个 Connector 的 get/ack/rollback 操作必须加锁同步。更推荐的方式是单线程 Get，多线程处理。

# 9. 异常处理最佳实践

```java
public class RobustConsumer {
    
    private CanalConnector connector;
    private MessageBatcher batcher;  // 消息缓存器
    
    public void consume() {
        while (running) {
            Message message = null;
            
            try {
                message = connector.getWithoutAck(1000);
                
                if (message.getId() == -1) continue;
                
                // 1. 幂等性检查
                if (isDuplicated(message.getId())) {
                    connector.ack(message.getId());
                    continue;
                }
                
                // 2. 处理数据
                processEntries(message.getEntries());
                
                // 3. 标记已处理
                markProcessed(message.getId());
                
                // 4. ACK
                connector.ack(message.getId());
                
            } catch (SerializationException e) {
                // Protobuf 反序列化异常：跳过脏数据
                logger.error("invalid entry, skip", e);
                if (message != null) connector.ack(message.getId());
                
            } catch (BusinessException e) {
                // 业务异常：回滚重试
                logger.error("business error, rollback", e);
                if (message != null) connector.rollback(message.getId());
                sleep(1000);
                
            } catch (NetworkException e) {
                // 网络异常：重连
                logger.error("network error, reconnect", e);
                reconnect();
                
            } catch (Exception e) {
                // 未知异常：记录并继续
                logger.error("unknown error", e);
                if (message != null) connector.ack(message.getId());
            }
        }
    }
}
```

# 10. Canal Client 配置参数

```java
// 通过 Properties 配置 Client
public class CanalClientConfig {
    
    // 连接超时（毫秒）
    private int connectTimeout = 10000;
    
    // 获取数据超时（毫秒）
    private long getTimeout = 5000;
    
    // 每次获取的批量大小
    private int batchSize = 1000;
    
    // 订阅的过滤规则
    private String subscribeFilter = ".*\\..*";
    
    // 重试次数
    private int retryCount = 3;
    
    // 重试间隔（毫秒）
    private long retryInterval = 5000;
    
    // 空闲检测间隔（毫秒）
    private long idleCheckInterval = 60000 * 30;  // 30分钟
    
    // 是否扁平化消息格式
    private boolean flatMessage = true;
    
    public CanalConnector buildConnector() {
        CanalConnector connector = CanalConnectors.newClusterConnector(
            zkServers, destination, username, password
        );
        return connector;
    }
}
```

# 11. CanalConnector 实现类解析

```java
// SimpleCanalConnector — 单机 TCP 连接实现
public class SimpleCanalConnector implements CanalConnector {
    
    private SocketAddress address;
    private SocketChannel channel;
    
    // 序列化使用的 Protobuf 编解码器
    private PacketCodec codec;
    
    @Override
    public Message getWithoutAck(int batchSize, Long timeout, TimeUnit unit) {
        // 构造 Get 请求
        Get get = Get.newBuilder()
            .setDestination(destination)
            .setClientId(clientId)
            .setBatchSize(batchSize)
            .setTimeout(timeout)
            .setUnit(unit.ordinal())
            .setAutoAck(false)
            .build();
        
        // 发送请求并接收响应
        Packet packet = codec.encode(get);
        channel.write(packet);
        
        Packet response = channel.read();
        Messages messages = codec.decode(response);
        
        return convertToMessage(messages);
    }
}

// ClusterCanalConnector — ZK 集群连接实现
public class ClusterCanalConnector implements CanalConnector {
    
    private CuratorFramework zkClient;
    private CanalConnector currentConnector;
    private String currentServer;
    
    /**
     * 从 ZK 获取当前 Running 的 Server 并连接
     */
    @Override
    public void connect() {
        // 从 ZK 获取 running 节点
        String path = "/otter/canal/destinations/" + destination + "/running";
        String runningServer = new String(zkClient.getData().forPath(path));
        
        // 构建连接器
        currentServer = runningServer;
        currentConnector = CanalConnectors.newSingleConnector(
            parseAddress(runningServer), destination, username, password
        );
        currentConnector.connect();
        
        // 监听 running 节点变化
        watchRunningNode();
    }
    
    private void watchRunningNode() {
        // 当 running 节点发生变化时，自动切换连接
        String path = "/otter/canal/destinations/" + destination + "/running";
        zkClient.getData().usingWatcher((CuratorWatcher) event -> {
            if (event.getType() == Watcher.Event.EventType.NodeDataChanged) {
                reconnect();
            }
        }).forPath(path);
    }
}
```
