
# 1. EventParser 概述

EventParser 是 Canal Server 的**数据入口模块**，负责与 MySQL Master 建立连接，模拟 MySQL Slave 协议订阅 Binlog，并将 Binlog 事件解析为 Canal 内部的结构化数据 `CanalEntry.Entry`。

```
┌──────────────────────────────────────────────────────────┐
│                     EventParser                           │
│                                                          │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────┐  │
│  │ Binlog 连接   │──>│ Binlog 拉取  │──>│ Entry 转换   │  │
│  │ (协议模拟)    │   │ (流式接收)   │   │ (结构还原)   │  │
│  └──────────────┘   └──────────────┘   └──────┬──────┘  │
│                                                │         │
│                                         ┌──────▼──────┐  │
│                                         │   Sink      │  │
│                                         │ (RingBuffer)│  │
│                                         └─────────────┘  │
└──────────────────────────────────────────────────────────┘
```

# 2. 核心类层次结构

```
CanalEventParser (接口)
    │
    └── AbstractEventParser (抽象基类)
            │
            ├── LocalBinlogEventParser (本地 Binlog 文件解析)
            │
            └── AbstractMysqlEventParser (MySQL 增量解析抽象)
                    │
                    ├── MysqlEventParser (标准 MySQL)
                    ├── GroupEventParser (分库分表合并)
                    └── RdsBinlogEventParserProxy (阿里云 RDS)
```

## 2.1 接口定义

```java
// CanalEventParser — 顶层接口
public interface CanalEventParser extends CanalLifeCycle {
    
    // 启动解析
    @Override
    void start();
    
    // 停止解析
    @Override
    void stop();
    
    // 设置事件过滤器（Sink）
    void setEventSink(CanalEventSink<List<CanalEntry.Entry>> eventSink);
}
```

## 2.2 AbstractEventParser — 解析流程骨架

```java
public abstract class AbstractEventParser<EVENT> 
        extends AbstractCanalLifeCycle 
        implements CanalEventParser<EVENT> {

    protected CanalEventSink<List<CanalEntry.Entry>> eventSink;
    protected CanalHAController haController;
    protected CanalMetaManager metaManager;
    protected CanalAlarmHandler alarmHandler;
    
    // 模板方法：定义解析生命周期
    @Override
    public void start() {
        super.start();
        
        // 1. 初始化
        init();
        
        // 2. 启动 Binlog 拉取线程
        startDumpThread();
        
        // 3. 启动解析线程
        startParseThread();
        
        // 4. 启动心跳检测线程
        startHeartBeatThread();
        
        // 5. 启动表结构快照线程
        startTableMetaThread();
    }
    
    // 模板方法：定义解析流程
    protected void processEvent(EVENT event) {
        // 1. 预处理（过滤、事务边界判断）
        preConsume(event);
        
        // 2. 转换为 CanalEntry
        List<CanalEntry.Entry> entries = parseEvent(event);
        
        // 3. 送入 Sink
        if (entries != null && !entries.isEmpty()) {
            doSink(entries);
        }
    }
    
    // 子类必须实现的抽象方法
    protected abstract void init();
    protected abstract List<CanalEntry.Entry> parseEvent(EVENT event);
    protected abstract void doSink(List<CanalEntry.Entry> entries);
}
```

# 3. MysqlEventParser — 核心实现

## 3.1 类结构

```java
public class MysqlEventParser extends AbstractMysqlEventParser {
    
    // MySQL 连接信息
    private InetSocketAddress masterAddress;
    private String username;
    private String password;
    
    // Binlog dump 参数
    private Long slaveId;
    private String binlogFileName;
    private Long binlogPosition;
    
    // 连接对象
    private MysqlConnection mysqlConnection;
    
    // 事件解析器
    private BinlogEventParser binlogEventParser;
    
    // 当前位点
    private EntryPosition currentPosition;
    
    // 表结构缓存
    private TableMetaCache tableMetaCache;
    
    // ... 其他成员变量
}
```

## 3.2 启动流程

```java
@Override
public void start() {
    super.start();
    
    // Step 1: 创建 MySQL 连接
    mysqlConnection = buildMysqlConnection();
    
    // Step 2: 连接并认证
    mysqlConnection.connect();
    
    // Step 3: 查找起始位点
    EntryPosition startPosition = findStartPosition();
    
    // Step 4: 发送 Binlog Dump 命令
    sendBinlogDump(startPosition);
    
    // Step 5: 启动接收线程，循环读取 Binlog 事件
    startEventReceiveThread();
    
    // Step 6: 启动解析线程，将事件转换为 Entry
    startParseThread();
}
```

## 3.3 findStartPosition — 位点查找策略

位点查找是 EventParser 的一个关键逻辑，决定了从 Binlog 的哪个位置开始订阅：

```java
protected EntryPosition findStartPosition() {
    
    EntryPosition startPosition = new EntryPosition();
    
    // 优先级 1: 从 MetaManager 恢复上次的消费位点
    Position cursor = metaManager.getCursor();
    if (cursor != null && cursor.getIdentity() != null) {
        startPosition = ((LogPosition) cursor).getPostion();
        logger.info("found start position from meta manager: {}", startPosition);
        return startPosition;
    }
    
    // 优先级 2: 使用配置指定的位点
    if (StringUtils.isNotEmpty(defaultConnectionConfig.getDefaultJournalName())) {
        startPosition.setJournalName(defaultConnectionConfig.getDefaultJournalName());
        startPosition.setPosition(defaultConnectionConfig.getDefaultPosition());
        logger.info("found start position from instance config: {}", startPosition);
        return startPosition;
    }
    
    // 优先级 3: 使用配置指定的时间戳
    if (defaultConnectionConfig.getDefaultTimestamp() != null) {
        long timestamp = defaultConnectionConfig.getDefaultTimestamp();
        startPosition = findPositionByTimestamp(timestamp);
        logger.info("found start position by timestamp [{}] : {}", timestamp, startPosition);
        return startPosition;
    }
    
    // 优先级 4: 从最新位置开始（通过 SHOW MASTER STATUS 获取）
    startPosition = findEndPosition(mysqlConnection);
    logger.info("found start position from latest: {}", startPosition);
    return startPosition;
}
```

**位点查找优先级**：
```
MetaManager 位点（HA 切换后恢复）
    ↓ 未找到
Instance 配置指定的 journalName + position
    ↓ 未指定
Instance 配置指定的 timestamp
    ↓ 未指定
MySQL SHOW MASTER STATUS（最新 binlog 位置）
```

## 3.4 Binlog Dump 命令发送

```java
protected void sendBinlogDump(EntryPosition position) {
    
    // 构造 dump 请求
    BinlogDumpCommand command = new BinlogDumpCommand();
    command.setBinlogPosition(position.getPosition());
    command.setBinlogFileName(position.getJournalName());
    command.setSlaveServerId(this.slaveId);
    
    // 如果启用 GTID，使用 GTID 方式 dump
    if (StringUtils.isNotEmpty(position.getGtid())) {
        command.setGtidSet(position.getGtid());
    }
    
    // 发送 COM_BINLOG_DUMP 命令
    mysqlConnection.sendBinlogDump(command);
    
    logger.info("send binlog dump command: {}", command);
}
```

## 3.5 Binlog 事件接收循环

```java
// 事件接收线程的核心循环
protected void dumpEventLoop() throws Exception {
    
    while (running) {
        try {
            // 从 MySQL 连接读取下一个 binlog event
            BinlogEvent event = mysqlConnection.readBinlogEvent(timeout);
            
            if (event == null) {
                // 心跳超时，发送查询保持连接
                mysqlConnection.sendHeartbeat();
                continue;
            }
            
            // 更新当前位点
            updateCurrentPosition(event);
            
            // 将原始事件放入解析队列
            putEventToParseQueue(event);
            
            // 向 MySQL 发送 ACK (半同步复制时使用)
            if (semiSyncEnabled) {
                mysqlConnection.sendSemiSyncAck(event.getLogPos());
            }
            
        } catch (Exception e) {
            handleDumpError(e);
        }
    }
}
```

# 4. Binlog 事件到 CanalEntry 的转换

## 4.1 事件类型映射

```java
// MysqlEventParser 内部的事件类型映射
protected EventType convertEventType(int mysqlEventType) {
    
    switch (mysqlEventType) {
        case LogEvent.WRITE_ROWS_EVENT:
        case LogEvent.WRITE_ROWS_EVENT_V1:
            return EventType.INSERT;
            
        case LogEvent.UPDATE_ROWS_EVENT:
        case LogEvent.UPDATE_ROWS_EVENT_V1:
            return EventType.UPDATE;
            
        case LogEvent.DELETE_ROWS_EVENT:
        case LogEvent.DELETE_ROWS_EVENT_V1:
            return EventType.DELETE;
            
        case LogEvent.QUERY_EVENT:
            return parseQueryEventType(); // 解析 DDL 类型
            
        default:
            return EventType.QUERY;
    }
}
```

## 4.2 行数据转换

```java
// 将 MySQL Row Event 转换为 Canal RowData
protected RowData parseRowData(RowsLogEvent event, TableMapLogEvent tableMap) {
    
    RowData.Builder rowDataBuilder = RowData.newBuilder();
    
    // 获取表结构元数据
    TableMeta tableMeta = getTableMeta(tableMap.getDbName(), tableMap.getTableName());
    
    // 转换前镜像（Before Image）
    if (event instanceof UpdateRowsLogEvent || event instanceof DeleteRowsLogEvent) {
        List<Column> beforeColumns = convertColumns(
            tableMeta, 
            event.getBeforeRows(),
            tableMap.getColumnTypes(),
            tableMap.getColumnNullability()
        );
        rowDataBuilder.addAllBeforeColumns(beforeColumns);
    }
    
    // 转换后镜像（After Image）
    if (event instanceof WriteRowsLogEvent || event instanceof UpdateRowsLogEvent) {
        List<Column> afterColumns = convertColumns(
            tableMeta,
            event.getAfterRows(),
            tableMap.getColumnTypes(),
            tableMap.getColumnNullability()
        );
        rowDataBuilder.addAllAfterColumns(afterColumns);
    }
    
    return rowDataBuilder.build();
}
```

## 4.3 Column 转换详解

```java
protected Column convertColumn(TableMeta tableMeta, int columnIndex,
                               byte[] columnValue, int columnType, boolean isNull) {
    
    Column.Builder columnBuilder = Column.newBuilder();
    
    // 获取列元数据
    TableMeta.ColumnMeta columnMeta = tableMeta.getColumnMeta(columnIndex);
    
    columnBuilder.setIndex(columnIndex);
    columnBuilder.setSqlType(columnType);
    columnBuilder.setName(columnMeta.getName());
    columnBuilder.setIsKey(columnMeta.isKey());
    columnBuilder.setMysqlType(columnMeta.getType());
    
    // 处理 NULL 值
    if (isNull) {
        columnBuilder.setIsNull(true);
        columnBuilder.setValue("");
        return columnBuilder.build();
    }
    
    // 根据 MySQL 类型进行值转换
    String value = convertValueByType(columnType, columnValue, columnMeta);
    columnBuilder.setValue(value);
    columnBuilder.setIsNull(false);
    columnBuilder.setLength(columnValue == null ? 0 : columnValue.length);
    
    return columnBuilder.build();
}
```

## 4.4 数据类型的值转换

```java
// Canal 支持的数据类型转换（部分示例）
protected String convertValueByType(int columnType, byte[] value, 
                                     TableMeta.ColumnMeta columnMeta) {
    switch (columnType) {
        case Types.TINYINT:
        case Types.SMALLINT:
        case Types.INTEGER:
            return String.valueOf(ByteHelper.toInt(value));
            
        case Types.BIGINT:
            if (columnMeta.isUnsigned()) {
                return String.valueOf(ByteHelper.toUnsignedLong(value));
            }
            return String.valueOf(ByteHelper.toLong(value));
            
        case Types.FLOAT:
            return String.valueOf(ByteHelper.toFloat(value));
            
        case Types.DOUBLE:
            return String.valueOf(ByteHelper.toDouble(value));
            
        case Types.DECIMAL:
        case Types.NUMERIC:
            return ByteHelper.toDecimal(value, columnMeta.getPrecision(), 
                                        columnMeta.getScale());
            
        case Types.DATE:
            return ByteHelper.toDate(value);
            
        case Types.TIME:
        case Types.TIME_WITH_TIMEZONE:
            return ByteHelper.toTime(value, columnType);
            
        case Types.TIMESTAMP:
        case Types.TIMESTAMP_WITH_TIMEZONE:
            return ByteHelper.toTimestamp(value, columnType);
            
        case Types.VARCHAR:
        case Types.CHAR:
            return new String(value, columnMeta.getCharset());
            
        case Types.BLOB:
        case Types.LONGBLOB:
        case Types.BINARY:
        case Types.VARBINARY:
            return ByteHelper.toHexString(value);
            
        case Types.BIT:
            return String.valueOf(ByteHelper.toBit(value));
            
        default:
            return new String(value, columnMeta.getCharset());
    }
}
```

# 5. TableMeta 管理

Canal 需要知道表结构才能正确解析 Binlog 中的行数据。TableMeta 负责缓存表结构元数据。

## 5.1 TableMeta 模型

```java
public class TableMeta {
    
    private String schema;          // 库名
    private String table;           // 表名
    private List<ColumnMeta> columns;  // 列信息
    private Map<String, ColumnMeta> columnIndex;  // 列名索引
    
    public static class ColumnMeta {
        private String name;        // 列名
        private int type;           // SQL 类型 (java.sql.Types)
        private String mysqlType;   // MySQL 原始类型名
        private boolean isKey;      // 是否主键
        private boolean isNull;     // 是否可为空
        private String defaultValue;// 默认值
        private int precision;      // 精度
        private int scale;          // 小数位
        private String charset;     // 字符集
    }
}
```

## 5.2 TableMetaCache — 表结构缓存

```java
public class TableMetaCache {
    
    // 缓存: key = schema.table, value = TableMeta
    private Cache<String, TableMeta> cache;
    
    // MySQL 连接（用于查询 INFORMATION_SCHEMA）
    private MysqlConnection metaConnection;
    
    /**
     * 获取表结构
     * 优先从缓存获取，缓存未命中则从 MySQL 查询
     */
    public TableMeta getTableMeta(String schema, String table) {
        String cacheKey = schema + "." + table;
        
        TableMeta meta = cache.getIfPresent(cacheKey);
        if (meta != null) {
            return meta;
        }
        
        // 从 MySQL 查询表结构
        meta = loadTableMetaFromMySQL(schema, table);
        if (meta != null) {
            cache.put(cacheKey, meta);
        }
        
        return meta;
    }
    
    /**
     * 通过 INFORMATION_SCHEMA 查询表结构
     */
    private TableMeta loadTableMetaFromMySQL(String schema, String table) {
        // 查询列信息
        String sql = "SELECT " +
            "COLUMN_NAME, " +
            "DATA_TYPE, " +
            "COLUMN_TYPE, " +
            "COLUMN_KEY, " +
            "IS_NULLABLE, " +
            "COLUMN_DEFAULT, " +
            "NUMERIC_PRECISION, " +
            "NUMERIC_SCALE, " +
            "CHARACTER_SET_NAME, " +
            "ORDINAL_POSITION " +
            "FROM INFORMATION_SCHEMA.COLUMNS " +
            "WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ? " +
            "ORDER BY ORDINAL_POSITION";
        
        // 执行查询并构建 TableMeta
        // ...
    }
}
```

# 6. 心跳检测机制

EventParser 实现了完善的心跳检测，确保与 MySQL 的连接健康。

## 6.1 心跳流程

```java
public class HeartBeatTask implements Runnable {
    
    private static final int HEARTBEAT_INTERVAL = 5000;  // 心跳间隔 5s
    private static final int HEARTBEAT_TIMEOUT = 5000;   // 心跳超时 5s
    private long lastHeartbeatTimestamp;
    
    @Override
    public void run() {
        while (running) {
            try {
                Thread.sleep(HEARTBEAT_INTERVAL);
                
                // 发送心跳
                mysqlConnection.sendHeartbeat();
                
                // 等待心跳返回
                waitingHeartbeatResponse();
                
                // 更新最后心跳时间
                lastHeartbeatTimestamp = System.currentTimeMillis();
                
            } catch (HeartbeatTimeoutException e) {
                // 心跳超时，触发重连
                handleConnectionLost(e);
            }
        }
    }
    
    private void waitingHeartbeatResponse() throws HeartbeatTimeoutException {
        long startTime = System.currentTimeMillis();
        while (running) {
            if (heartbeatResponseReceived.get()) {
                return;
            }
            if (System.currentTimeMillis() - startTime > HEARTBEAT_TIMEOUT) {
                throw new HeartbeatTimeoutException("heartbeat timeout");
            }
            Thread.sleep(100);
        }
    }
}
```

## 6.2 连接异常处理

```java
protected void handleConnectionLost(Exception e) {
    logger.error("connection lost, try to reconnect", e);
    
    // 1. 记录当前位点（保证不丢失数据）
    saveCurrentPosition();
    
    // 2. 尝试重连（带退避策略）
    int retryCount = 0;
    int maxRetries = 5;
    long retryInterval = 1000;  // 1s
    
    while (retryCount < maxRetries) {
        try {
            // 等待
            Thread.sleep(retryInterval * (retryCount + 1));
            
            // 重新连接
            mysqlConnection.reconnect();
            
            // 恢复 Binlog Dump
            EntryPosition resumePosition = findStartPosition();
            sendBinlogDump(resumePosition);
            
            logger.info("reconnect success, resume from {}", resumePosition);
            return;
            
        } catch (Exception reconnectError) {
            retryCount++;
            logger.error("reconnect failed, retry {}/{}", retryCount, maxRetries);
        }
    }
    
    // 重试耗尽，触发告警
    alarmHandler.sendAlarm("canal connection lost and cannot reconnect");
}
```

# 7. 事务边界处理

## 7.1 事务事件识别

```java
protected void identifyTransaction(EVENT event) {
    
    if (event instanceof XidEvent) {
        // XID Event 标志事务提交
        handleTransactionEnd(event);
    } 
    else if (event instanceof QueryEvent) {
        String query = ((QueryEvent) event).getQuery();
        if ("BEGIN".equalsIgnoreCase(query)) {
            handleTransactionBegin(event);
        } else if ("COMMIT".equalsIgnoreCase(query)) {
            handleTransactionEnd(event);
        }
    }
    else if (event instanceof GtidEvent) {
        // GTID Event 也可能标志新事务开始
        handleTransactionBegin(event);
    }
}

protected void handleTransactionBegin(EVENT event) {
    // 创建事务上下文
    Transaction transaction = new Transaction();
    transaction.setBeginPosition(getCurrentPosition(event));
    currentTransaction.set(transaction);
}

protected void handleTransactionEnd(EVENT event) {
    Transaction transaction = currentTransaction.get();
    if (transaction != null) {
        transaction.setEndPosition(getCurrentPosition(event));
        // 将完整的事务数据发送到 Sink
        flushTransaction(transaction);
        currentTransaction.remove();
    }
}
```

# 8. DDL 事件处理

## 8.1 DDL 识别与转换

```java
protected void parseDdlEvent(QueryEvent queryEvent) {
    
    String sql = queryEvent.getQuery();
    
    if (isCreateTable(sql)) {
        // CREATE TABLE：刷新表结构缓存
        handleCreateTable(sql);
    } 
    else if (isAlterTable(sql)) {
        // ALTER TABLE：更新表结构缓存
        handleAlterTable(sql);
    } 
    else if (isDropTable(sql)) {
        // DROP TABLE：移除表结构缓存
        handleDropTable(sql);
    }
    else if (isRenameTable(sql)) {
        // RENAME TABLE：更新表名映射
        handleRenameTable(sql);
    }
    
    // 构造 DDL Entry
    RowChange.Builder builder = RowChange.newBuilder();
    builder.setIsDdl(true);
    builder.setSql(sql);
    builder.setEventType(detectDdlEventType(sql));
    
    // 放入 Sink
    // ...
}

private EventType detectDdlEventType(String sql) {
    String upperSql = sql.toUpperCase().trim();
    if (upperSql.startsWith("CREATE")) return EventType.CREATE;
    if (upperSql.startsWith("ALTER"))  return EventType.ALTER;
    if (upperSql.startsWith("DROP"))   return EventType.ERASE;
    if (upperSql.startsWith("RENAME")) return EventType.RENAME;
    if (upperSql.startsWith("TRUNCATE")) return EventType.TRUNCATE;
    return EventType.QUERY;
}
```

# 9. 阿里云 RDS 适配

`RdsBinlogEventParserProxy` 是针对阿里云 RDS MySQL 的特殊实现：

```java
public class RdsBinlogEventParserProxy extends AbstractMysqlEventParser {
    
    // RDS 特有：需要通过 OSS 拉取 Binlog Backups
    private OSSClient ossClient;
    private String accessKey;
    private String secretKey;
    private String instanceId;
    
    @Override
    protected EntryPosition findStartPosition() {
        // RDS 没有直接的 binlog dump 权限
        // 需要通过 RDS API 获取 binlog 信息
        RdsBinlogInfo binlogInfo = rdsApiClient.describeBinlogFiles(instanceId);
        
        // 从 OSS 下载最新的 binlog 文件
        downloadBinlogFromOSS(binlogInfo.getLatestFile());
        
        return binlogInfo.getLatestPosition();
    }
    
    @Override
    protected void sendBinlogDump(EntryPosition position) {
        // RDS 使用不同的 binlog 订阅方式
        // 通过订阅 RDS 的 binlog 推送管道
        rdsApiClient.subscribeBinlog(position);
    }
}
```

# 10. 性能优化要点

## 10.1 批量处理

```java
// 批量提交到 Sink，减少 RingBuffer 的竞争
protected void doSink(List<CanalEntry.Entry> entries) {
    // 累积到一定数量再批量提交
    batchBuffer.addAll(entries);
    
    if (batchBuffer.size() >= BATCH_SIZE || 
        System.currentTimeMillis() - lastFlushTime > FLUSH_INTERVAL) {
        eventSink.sink(batchBuffer, ...);
        batchBuffer.clear();
        lastFlushTime = System.currentTimeMillis();
    }
}
```

## 10.2 零拷贝

Canal 在解析 Binlog 时尽量使用零拷贝技术，避免不必要的内存分配：

```java
// 直接基于 ByteBuffer 操作，不复制字节数组
protected String parseEncodedString(ByteBuffer buffer, String charset) {
    int length = buffer.getInt();
    if (length == 0) return "";
    
    byte[] bytes = new byte[length]; // 不可避免的复制
    buffer.get(bytes);
    return new String(bytes, charset);
}
```

## 10.3 TSDB 表结构快照

```java
// TSDB (Table Structure Database) 提供表结构的本地快照
// 避免每次解析都需要查询 MySQL INFORMATION_SCHEMA
public class TableMetaTSDB {
    
    // 使用嵌入式 H2 数据库存储表结构
    private DataSource h2DataSource;
    
    // 表结构变更时自动更新 TSDB
    public void applyTableMeta(String schema, String table, 
                                TableMeta meta, String ddlSql) {
        // 对 DDL SQL 做快照解析
        TableMeta newMeta = snapshotParser.parse(ddlSql, meta);
        saveToTSDB(schema, table, newMeta);
        // 更新内存缓存
        metaCache.put(schema, table, newMeta);
    }
}
```
