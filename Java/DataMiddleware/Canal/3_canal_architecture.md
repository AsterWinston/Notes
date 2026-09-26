
# 1. 整体架构概览

Canal 的架构设计遵循**管道-过滤器（Pipeline-Filter）**模式，将 Binlog 的订阅、解析、存储、消费划分为独立的处理阶段：

```
┌───────────────────────────────────────────────────────────────────────┐
│                          MySQL Master                                 │
│                     (binlog_format=ROW)                               │
└──────────────────────────────┬────────────────────────────────────────┘
                               │ Binlog Stream (MySQL Replication Protocol)
                               ▼
┌───────────────────────────────────────────────────────────────────────┐
│                          Canal Server                                 │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Canal Instance                             │    │
│  │  (每个 instance 对应一个 MySQL 数据源)                       │    │
│  │                                                               │    │
│  │  ┌─────────┐    ┌─────────┐    ┌───────────┐    ┌────────┐  │    │
│  │  │ Parser  │───>│  Sink   │───>│   Store   │───>│ Server │  │    │
│  │  │ (解析器)│    │(数据过滤)│    │(数据存储) │    │(服务层)│  │    │
│  │  └─────────┘    └─────────┘    └───────────┘    └────────┘  │    │
│  │       │                                               │      │    │
│  └───────┼───────────────────────────────────────────────┼──────┘    │
│          │                                               │           │
│  ┌───────┼───────────── Canal MQ Connector ──────────────┼──────┐    │
│  │       │                                               │      │    │
│  │  ┌────▼────┐  ┌──────────┐  ┌──────────┐  ┌──────────▼───┐ │    │
│  │  │  Kafka  │  │ RocketMQ │  │ RabbitMQ │  │   TCP Server │ │    │
│  │  │Connector│  │Connector │  │Connector │  │  (Netty)     │ │    │
│  │  └────┬────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘ │    │
│  └───────┼────────────┼─────────────┼─────────────┼──────────┘    │
└──────────┼────────────┼─────────────┼─────────────┼───────────────┘
           │            │             │             │
    ┌──────▼──┐   ┌────▼───┐   ┌────▼───┐   ┌─────▼──────┐
    │  Kafka  │   │RocketMQ│   │RabbitMQ│   │Canal Client│
    │ Cluster │   │Cluster │   │Cluster │   │ (Consumer) │
    └─────────┘   └────────┘   └────────┘   └────────────┘
```

# 2. 核心设计理念

## 2.1 管道-过滤器模式

Canal 将 Binlog 处理流程抽象为一条管道，每个阶段都是独立的过滤器：

```
Input: Binlog Event Stream
  │
  ├── [Parser]    → 从 MySQL Master 拉取 Binlog，解析为结构化事件
  │     │
  ├── [Sink]      → 对事件进行过滤、转换、聚合
  │     │
  ├── [Store]     → 将事件持久化存储（内存 RingBuffer 或文件）
  │     │
  └── [Server]    → 通过 Netty 对外提供事件订阅服务
         │
Output: Structured Data Change Event
```

每个阶段职责单一、可独立替换。例如，Store 可以选用内存、文件、甚至第三方存储实现。

## 2.2 多实例隔离

一个 Canal Server 可以同时管理多个 Canal Instance，每个 Instance 对应一个独立的 MySQL 数据源：

```
┌────────────────────────────────────────────┐
│             Canal Server                   │
│                                            │
│  ┌──────────────┐  ┌──────────────┐       │
│  │ Instance A   │  │ Instance B   │       │
│  │ (MySQL-订单库)│  │ (MySQL-用户库)│       │
│  │              │  │              │       │
│  │ Parser ─┐    │  │ Parser ─┐    │       │
│  │ Sink    │    │  │ Sink    │    │       │
│  │ Store   │    │  │ Store   │    │       │
│  │ MetaMgr ┘    │  │ MetaMgr ┘    │       │
│  └──────────────┘  └──────────────┘       │
└────────────────────────────────────────────┘
```

每个 Instance 独立维护自己的 Parser、Sink、Store 和位点信息，互不影响。

## 2.3 事件驱动模型

Canal 采用**事件驱动**的编程模型，通过 Disruptor（高性能无锁队列）在各阶段之间传递事件：

```
┌──────────┐  RingBuffer  ┌──────────┐  RingBuffer  ┌──────────┐
│  Parser  │─────────────>│   Sink   │─────────────>│  Store   │
│ (Producer)              │(Consumer/ │             │ (Consumer)
│                         │ Producer) │             │
└──────────┘              └──────────┘             └─────┬────┘
                                                        │
                                                  Netty Server
                                                        │
                                                  ┌─────▼──────┐
                                                  │   Client   │
                                                  │ (Consumer) │
                                                  └────────────┘
```

# 3. 核心组件详解

## 3.1 CanalInstance — 顶层抽象

`CanalInstance` 是 Canal 最顶层的运行单位，代表一个完整的数据订阅实例：

```java
public interface CanalInstance {
    // 启动实例
    void start();
    // 停止实例
    void stop();
    // 是否运行中
    boolean isStart();
    // 获取事件解析器
    CanalEventParser getEventParser();
    // 获取事件过滤器
    CanalEventSink<List<CanalEntry.Entry>> getEventSink();
    // 获取事件存储
    CanalEventStore<CanalEntry.Entry> getEventStore();
    // 获取元数据管理器
    CanalMetaManager getMetaManager();
    // 获取告警处理器
    CanalAlarmHandler getAlarmHandler();
    // 获取MQ生产者（如果配置了MQ模式）
    CanalMQProducer getCanalMQProducer();
}
```

## 3.2 EventParser — 事件解析器

EventParser 是 Canal 的数据入口，负责与 MySQL 建立连接并拉取 Binlog：

```java
public interface CanalEventParser {
    // 启动解析器
    void start();
    // 停止解析器
    void stop();
}
```

核心实现类 `AbstractEventParser` 的职责：
1. 构造 MySQL Slave 伪装连接
2. 执行 `COM_REGISTER_SLAVE` 和 `COM_BINLOG_DUMP`
3. 接收 Binlog 事件流
4. 解析 Binlog 事件为 `CanalEntry.Entry`
5. 将解析结果推入 Sink 的 RingBuffer

## 3.3 EventSink — 事件过滤器/转换器

EventSink 位于 Parser 和 Store 之间，负责过滤和转换事件：

```java
public interface CanalEventSink<T> {
    // 过滤事件流
    boolean sink(T event, CanalEntry.Entry entry, String... filterTableNames);
    // 启动
    void start();
    // 停止
    void stop();
}
```

主要功能：
- **表/库过滤**：`canal.instance.filter.regex=.*\\..*`
- **黑名单过滤**：`canal.instance.filter.black.regex=`
- **事务边界处理**：合并同一事务内的多个 Event
- **事件类型过滤**：只订阅 INSERT/UPDATE/DELETE
- **数据转换**：字段值格式化、编码转换

## 3.4 EventStore — 事件存储

EventStore 负责暂存解析后的事件，等待 Client 消费：

```java
public interface CanalEventStore<T> extends CanalLifeCycle {
    // 存储事件
    void put(List<T> data);
    // 获取事件（指定超时时间）
    Events<T> get(BatchAckCallback callback, PositionRange range, int batchSize, long timeout);
    // 获取事件（按传输格式）
    Events<T> get(BatchAckCallback callback, PositionRange range, int batchSize,
                  TimeUnit unit, long timeout, boolean isRaw);
    // ACK 确认
    void ack(Position position);
    // 回滚到指定位置
    void rollback(Position position);
}
```

**存储实现**：

| 实现类 | 说明 |
|--------|------|
| `MemoryEventStoreWithBuffer` | 基于 Disruptor RingBuffer 的内存存储（默认） |
| `FileEventStore` | 基于文件系统的持久化存储 |
| `MixedEventStore` | 内存+文件混合存储（内存满时写入文件） |

**MemoryEventStoreWithBuffer 的核心设计**：

```
┌──────────────────────────────────────────────────────────┐
│                  Disruptor RingBuffer                     │
│                                                          │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐     │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │...│N-1│     │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘     │
│    ▲                         ▲                           │
│    │                         │                           │
│  Put Sequence            Get Sequence                    │
│  (写入位置)              (消费位置)                       │
│                                                          │
│  bufferSize = 16384 (默认)                               │
│  bufferMemUnit = 1024 (默认 1KB/entry)                   │
│  put/wait/interval = 1000ms                              │
└──────────────────────────────────────────────────────────┘
```

## 3.5 MetaManager — 元数据管理器

MetaManager 负责持久化 Canal 的消费位点信息：

```java
public interface CanalMetaManager {
    // 获取当前消费位点
    Position getCursor();
    // 更新消费位点
    void updateCursor(Position position);
}
```

**存储的位点信息**：

```java
// 位点信息接口
public interface LogPosition extends Position {
    LogIdentity getIdentity();  // SlaveId + SourceAddress
    EntryPosition getPostion(); // Binlog 文件名 + 偏移量
}

public class EntryPosition implements Position {
    private String journalName;  // binlog 文件名，如 mysql-bin.000003
    private Long   position;     // binlog 偏移量
    private Long   timestamp;    // 位点时间戳
    private String gtid;         // GTID 集合
}
```

## 3.6 Server — 事件投递服务

Canal Server 将 Store 中的事件通过不同方式投递出去：

**TCP 模式（默认）**：
- 基于 Netty 构建 TCP Server
- Client 通过 Canal Client SDK 连接
- 支持订阅/取消订阅、ACK/ROLLBACK

**MQ 模式**：
- Kafka Producer：将事件序列化为 JSON/Protobuf 写入 Kafka
- RocketMQ Producer：投递到 RocketMQ Topic
- RabbitMQ Producer：投递到 RabbitMQ Queue

# 4. 数据流转全景

## 4.1 单条数据变更的完整旅程

以一条 `UPDATE` 语句为例，追踪数据在 Canal 中的完整流转：

```
1. MySQL Master 执行 UPDATE users SET name='Bob' WHERE id=1
                                    │
2. 写入 Binlog (ROW格式)           │
                                    ▼
   ┌──────────────────────────────────────────────┐
   │ TABLE_MAP: test.users (id, name, age)        │
   │ UPDATE_ROWS: id=1, name: 'Alice' -> 'Bob'    │
   │ XID: COMMIT                                  │
   └──────────────────────────────────────────────┘
                                    │
3. Canal Parser 拉取 Binlog        │
                                    ▼
   ┌──────────────────────────────────────────────┐
   │ CanalEntry.Entry {                           │
   │   Header {                                   │
   │     logfileName: "mysql-bin.000003"           │
   │     logfileOffset: 12345                      │
   │     schemaName: "test"                        │
   │     tableName: "users"                        │
   │     eventType: UPDATE                         │
   │     executeTime: 1672545600000                 │
   │   }                                           │
   │   EntryType: ROWDATA                          │
   │   StoreValue: RowChange (serialized proto)    │
   │ }                                             │
   └──────────────────────────────────────────────┘
                                    │
4. 放入 Sink RingBuffer            │
                                    ▼
   ┌──────────────────────────────────────────────┐
   │ Sink 执行过滤:                                │
   │ - 检查 table regex 匹配 ✓                     │
   │ - 检查 black list  ✓                          │
   │ - 转换数据类型                                │
   └──────────────────────────────────────────────┘
                                    │
5. 放入 Store RingBuffer           │
                                    ▼
   ┌──────────────────────────────────────────────┐
   │ Store 存储:                                   │
   │ - 写入 Disruptor RingBuffer                   │
   │ - 更新位点信息                                │
   │ - 等待 Client 消费                            │
   └──────────────────────────────────────────────┘
                                    │
6. Client 拉取 (getWithoutAck)       │
                  │                 │
   ┌──────────────▼─────────────────┐
   │ Canal Client 消费:                │
   │ - 反序列化 protobuf               │
   │ - 业务处理（更新缓存/同步ES/...  │
   │ - ACK 确认                       │
   └────────────────────────────────┘
```

## 4.2 数据序列化链路

```
MySQL Binlog Event (binary)
        │
        ▼ (dbsync 驱动解析)
CanalEntry.Entry (Java Object in memory)
        │
        ├──> Protobuf 序列化 ──> byte[] ──> Store (RingBuffer)
        │                                      │
        │                                Netty TCP
        │                                      │
        │                              ┌───────▼────────┐
        │                              │ Canal Client   │
        │                              │ 反序列化 Entry  │
        │                              └────────────────┘
        │
        └──> JSON 序列化 ──> String ──> Kafka / RocketMQ / RabbitMQ
```

# 5. 线程模型

## 5.1 Canal Server 线程架构

```
┌─────────────────────────────────────────────────────────────────┐
│                       Canal Server JVM                           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  EventParser 线程组                       │    │
│  │                                                          │    │
│  │  [parseThread]      [dumpThread]      [heartbeatThread]  │    │
│  │  - 解析 binlog      - 拉取 binlog      - 心跳检测         │    │
│  │  - 构造 Entry       - 建立连接         - 连接保活         │    │
│  │  - 写入SinkBuffer   - 流式接收                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│             ┌────────────────▼─────────────────┐                │
│             │       SinkStage (Disruptor)       │                │
│             │       [sinkThread-1..N]           │                │
│             │       单线程处理（默认）          │                │
│             └────────────────┬─────────────────┘                │
│                              │                                   │
│             ┌────────────────▼─────────────────┐                │
│             │      StoreStage (Disruptor)       │                │
│             │      [storeThread-1..N]           │                │
│             │      单线程处理（默认）           │                │
│             └────────────────┬─────────────────┘                │
│                              │                                   │
│  ┌───────────────────────────▼────────────────────────────┐     │
│  │                 Netty Server 线程组                      │     │
│  │                                                          │     │
│  │  [bossGroup-1]        [workerGroup-N]                    │     │
│  │  - 接受客户端连接      - 处理 Client 请求                 │     │
│  │  - 分发连接            - 序列化/发送事件                  │     │
│  └─────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

## 5.2 线程数配置

```properties
# canal.properties

# Parser 线程数（处理 binlog dump 和解析）
canal.instance.parser.threads = 1

# Sink 线程数（事件过滤处理）
canal.instance.sink.threads = 1

# Store 线程数（事件存储）
canal.instance.store.threads = 1

# Netty Boss 线程数（接受连接）
canal.port.boss.threads = 1

# Netty Worker 线程数（处理请求）
canal.port.worker.threads = 4
```

# 6. 关键数据结构

## 6.1 CanalEntry.Entry

Canal 中所有事件的核心载体：

```protobuf
message Entry {
    // 事件头
    Header header = 1;
    // 事件类型：ROWDATA / TRANSACTIONBEGIN / TRANSACTIONEND / HEARTBEAT / GTIDLOG
    EntryType entryType = 2;
    // 序列化后的具体内容（如 RowChange）
    bytes storeValue = 3;
}

message Header {
    int32 version = 1;
    string logfileName = 2;      // binlog 文件名
    int64 logfileOffset = 3;    // binlog 文件偏移量
    int64 serverId = 4;         // MySQL server id
    string serverenCode = 5;    // 编码
    int64 executeTime = 6;      // 事件执行时间戳
    string sourceType = 7;      // 数据源类型（MYSQL/ORACLE）
    string schemaName = 8;      // 库名
    string tableName = 9;       // 表名
    int64 eventLength = 10;     // 事件长度
    EventType eventType = 11;   // INSERT/UPDATE/DELETE/CREATE/ALTER...
    Pair props = 12;            // 扩展属性
    string gtid = 13;           // GTID 字符串
}
```

## 6.2 RowChange

DML 操作的行级变更数据：

```protobuf
message RowChange {
    int64 tableId = 1;          // 表 ID
    EventType eventType = 2;    // INSERT/UPDATE/DELETE
    bool isDdl = 10;            // 是否是 DDL
    string sql = 11;            // DDL 的 SQL 语句
    repeated RowData rowDatas = 12;  // 变更的行数据
    string ddlSchemaName = 13;  // DDL 的库名
    RowChange_Props props = 14; // 扩展属性
}

message RowData {
    repeated Column beforeColumns = 1;  // 变更前数据
    repeated Column afterColumns = 2;   // 变更后数据
    RowChange_Props props = 3;          // 行级扩展属性
}

message Column {
    int32 index = 1;        // 列索引
    int32 sqlType = 2;      // SQL 类型（对应 java.sql.Types）
    string name = 3;        // 列名
    bool isKey = 4;         // 是否主键
    bool updated = 5;       // 是否被更新
    bool isNull = 6;        // 列值是否为 NULL
    string value = 7;       // 列值（字符串形式）
    int32 length = 8;       // 列值原始长度
    string mysqlType = 9;   // MySQL 类型名称
}
```

## 6.3 数据示例

INSERT 操作：

```json
{
  "header": {
    "schemaName": "test",
    "tableName": "users",
    "eventType": "INSERT",
    "executeTime": 1672545600000
  },
  "entryType": "ROWDATA",
  "rowChange": {
    "eventType": "INSERT",
    "rowDatas": [{
      "afterColumns": [
        {"index": 0, "name": "id", "value": "1", "isKey": true},
        {"index": 1, "name": "name", "value": "Alice"},
        {"index": 2, "name": "email", "value": "alice@example.com"}
      ]
    }]
  }
}
```

UPDATE 操作：

```json
{
  "header": {
    "schemaName": "test",
    "tableName": "users",
    "eventType": "UPDATE"
  },
  "entryType": "ROWDATA",
  "rowChange": {
    "eventType": "UPDATE",
    "rowDatas": [{
      "beforeColumns": [
        {"index": 0, "name": "id", "value": "1", "isKey": true},
        {"index": 1, "name": "name", "value": "Alice"},
        {"index": 2, "name": "email", "value": "alice@example.com"}
      ],
      "afterColumns": [
        {"index": 0, "name": "id", "value": "1", "isKey": true},
        {"index": 1, "name": "name", "value": "Bob"},
        {"index": 2, "name": "email", "value": "bob@example.com"}
      ]
    }]
  }
}
```

DELETE 操作：

```json
{
  "header": {
    "schemaName": "test",
    "tableName": "users",
    "eventType": "DELETE"
  },
  "entryType": "ROWDATA",
  "rowChange": {
    "eventType": "DELETE",
    "rowDatas": [{
      "beforeColumns": [
        {"index": 0, "name": "id", "value": "1", "isKey": true}
      ]
    }]
  }
}
```

# 7. 配置体系

## 7.1 两层配置结构

```
canal.properties        ← Server 级别配置（全局）
    │
    └── instance/
        └── example/
            └── instance.properties  ← Instance 级别配置（单数据源）
```

**canal.properties** — Server 全局配置：

```properties
# Canal Server 监听端口
canal.port = 11111

# 多实例管理模式
canal.instance.global.mode = spring

# 实例列表（逗号分隔）
canal.destinations = example, example2

# ZooKeeper 地址（HA 模式）
canal.zkServers = 127.0.0.1:2181

# 数据持久化方式
canal.instance.global.spring.xml = classpath:spring/file-instance.xml

# MQ 相关配置
canal.serverMode = tcp  # tcp / kafka / rocketmq / rabbitmq
```

**instance.properties** — Instance 配置：

```properties
# MySQL 连接信息
canal.instance.master.address = 127.0.0.1:3306
canal.instance.dbUsername = canal
canal.instance.dbPassword = canal

# 过滤规则
canal.instance.filter.regex = .*\\..*
canal.instance.filter.black.regex =

# 位点信息
canal.instance.master.journal.name =
canal.instance.master.position =

# GTID 模式
canal.instance.gtidon = false

# TSDB（表结构快照）
canal.instance.tsdb.enable = true
```

## 7.2 配置优先级

```
命令行参数 (-Dxxx=yyy)
    │
    ▼
环境变量 (CANAL_XXX=yyy)
    │
    ▼
instance.properties
    │
    ▼
canal.properties
    │
    ▼
默认值（代码中的 @Value 注解默认值）
```

# 8. 设计模式总结

Canal 架构中应用了多种经典设计模式：

| 设计模式 | 应用场景 |
|----------|---------|
| **管道-过滤器** | Parser→Sink→Store→Server 的数据流转 |
| **观察者模式** | Client 订阅 Instance 事件 |
| **策略模式** | Store 的多种存储策略（Memory/File/Mixed） |
| **模板方法** | AbstractEventParser 定义解析流程骨架 |
| **工厂模式** | CanalInstance 的创建和组装 |
| **单例模式** | MetaManager、AlarmHandler 等全局组件 |
| **适配器模式** | Canal Adapter 将数据适配到不同目标 |
| **生产者-消费者** | RingBuffer 中的事件生产和消费 |
