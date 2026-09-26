# 1 Kafka Connect 概述

## 1.1 定位与价值

Kafka Connect 是 Apache Kafka 提供的数据集成框架，用于**将外部系统数据导入/导出 Kafka**，目标是"零编码数据集成"。

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kafka Connect 定位                           │
│                                                                 │
│  外部系统                   Kafka                    外部系统    │
│  ┌──────────┐   Source    ┌──────┐    Sink     ┌──────────┐    │
│  │ MySQL    │───────────▶│      │───────────▶│ HDFS     │    │
│  └──────────┘             │      │             └──────────┘    │
│  ┌──────────┐             │Kafka │             ┌──────────┐    │
│  │ MongoDB  │───────────▶│      │───────────▶│ Elasticsearch│   │
│  └──────────┘             └──────┘             └──────────┘    │
│  ┌──────────┐                                ┌──────────┐    │
│  │ REST API │───────────▶                    │ S3       │    │
│  └──────────┘                                └──────────┘    │
│                                                                 │
│  对比传统:                                                       │
│  传统 ETL: 每个数据源写一个 Adapter → 开发成本高, 维护困难       │
│  Kafka Connect: 配置驱动 → 开箱即用, 统一管理                    │
└─────────────────────────────────────────────────────────────────┘
```

## 1.2 核心概念

| 概念 | 说明 |
|------|------|
| Connector | 逻辑作业，定义数据从哪里来/到哪里去 |
| Task | 实际执行数据复制的单元，一个 Connector 可生成多个 Task |
| Worker | 运行 Connector 和 Task 的 JVM 进程 |
| Source Connector | 从外部系统拉取数据写入 Kafka |
| Sink Connector | 从 Kafka 读取数据写入外部系统 |
| Transform | 消息转换（单条消息转换 SMT） |
| Converter | 数据序列化/反序列化 |

# 2 Connect 架构

## 2.1 分布式模式架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Connect 分布式模式                          │
│                                                              │
│  Connect Cluster (Group ID: connect-cluster)                 │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │   Worker 1       │  │   Worker 2       │  │  Worker 3  │ │
│  │                  │  │                  │  │            │ │
│  │ Connector 1      │  │ Connector 1      │  │ Connector 2│ │
│  │ ├── Task 0       │  │ ├── Task 2       │  │ ├── Task 0 │ │
│  │ └── Task 1       │  │ └── Task 3       │  │            │ │
│  │                  │  │                  │  │            │ │
│  │ Connector 3      │  │                  │  │            │ │
│  │ └── Task 0       │  │                  │  │            │ │
│  └──────────────────┘  └──────────────────┘  └────────────┘ │
│                                                              │
│  REST API (端口 8083):                                       │
│  GET    /connectors                   列出所有 Connector     │
│  POST   /connectors                   创建 Connector         │
│  GET    /connectors/{name}            查看 Connector 详情    │
│  PUT    /connectors/{name}/config     更新 Connector 配置    │
│  DELETE /connectors/{name}            删除 Connector         │
│  GET    /connectors/{name}/status     查看 Connector 状态    │
│  POST   /connectors/{name}/restart    重启 Connector         │
│  GET    /connectors/{name}/tasks      查看 Task 列表          │
│  GET    /connectors/{name}/tasks/{id}/status 查看 Task 状态   │
└──────────────────────────────────────────────────────────────┘
```

## 2.2 Worker 内部结构

```
Worker 进程
│
├── REST Server (端口 8083)
│
├── Herder (协调器)
│   ├── DistributedHerder (分布式模式)
│   │   ├── 通过 Kafka Topic 存储配置、Offset、状态
│   │   ├── 均衡 Connector/Task 分配到各 Worker
│   │   └── 监听 Worker 加入/离开
│   └── StandaloneHerder (单机模式)
│
├── Connector 实例
│   └── Task 实例 (多个)
│       ├── SourceTask: poll() → 拉取数据 → send to Kafka
│       └── SinkTask: put() → 从 Kafka 接收 → 写入外部系统
│
└── Offset 存储 (Kafka Topic: connect-offsets)
    记录每个 Source Connector 的消费进度
```

# 3 配置与部署

## 3.1 Worker 配置（分布式模式）

```properties
# config/connect-distributed.properties

# Kafka 集群地址
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092

# Connect Group ID（多 Worker 加入同一 Group）
group.id=connect-cluster

# Key/Value 转换器
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter

# 内部 Topic 配置
config.storage.topic=connect-configs          # Connector 配置存储
offset.storage.topic=connect-offsets           # Source Connector Offset
status.storage.topic=connect-status            # Connector/Task 状态
config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

# 插件路径（Connector JAR 存放位置）
plugin.path=/usr/share/java,/opt/connectors

# REST API
rest.port=8083
rest.advertised.host.name=worker1.example.com
```

## 3.2 启动

```bash
# 分布式模式
bin/connect-distributed.sh config/connect-distributed.properties &

# 检查是否启动
curl http://localhost:8083/
# {"version":"3.6.0","commit":"...","kafka_cluster_id":"..."}
```

# 4 Source Connector 详解

## 4.1 示例：FileStream Source Connector

```bash
# 创建 FileStream Source Connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "local-file-source",
    "config": {
        "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
        "tasks.max": "1",
        "file": "/var/log/app.log",
        "topic": "app-logs"
    }
}'
```

## 4.2 Source Connector 分批读取机制（偏移量管理）

```java
// SourceTask 核心方法
public abstract class SourceTask {
    // ① 启动时调用
    public void start(Map<String, String> props);

    // ② 定期调用（poll 循环）
    // 返回一批 SourceRecord
    public abstract List<SourceRecord> poll() throws InterruptedException;

    // ③ 提交 offset（由 Connect 框架管理）
    // offset 提交到 connect-offsets Topic
    public void commitRecord(SourceRecord record, RecordMetadata metadata);
}
```

### 4.2.1 SourceRecord 与 SourcePartition

```java
// 每条 SourceRecord 携带 SourcePartition 和 SourceOffset
// 用于容错恢复（重启后从上次位置继续）

// 示例: JDBC Source Connector
SourcePartition = Map.of("table", "users")           // 标识数据源分区
SourceOffset   = Map.of("timestamp", 1700000000000L)  // 源端的 offset

// 恢复: Connect 框架从 connect-offsets 读取上次 offset
// 传给 SourceTask 作为起始位置
```

## 4.3 常用 Source Connector

| Connector | 来源 | 用途 |
|-----------|------|------|
| JDBC Source | 关系型数据库 | 全量/增量同步表数据到 Kafka |
| Debezium MySQL | MySQL Binlog | CDC, 实时捕获数据变更 |
| Debezium PostgreSQL | PG WAL | CDC |
| MongoDB Source | MongoDB Oplog | CDC |
| FileStream Source | 本地文件 | 日志文件导入（开发测试用） |
| JMS Source | JMS 队列 | 从 JMS 导入消息 |

### 4.3.1 Debezium MySQL Connector 配置示例

```json
{
    "name": "mysql-inventory-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "mysql.example.com",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "dbz",
        "database.server.id": "184054",
        "database.server.name": "dbserver1",
        "database.include.list": "inventory",
        "table.include.list": "inventory.customers,inventory.orders",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.history.kafka.topic": "schema-changes.inventory",
        "include.schema.changes": "true"
    }
}
```

# 5 Sink Connector 详解

## 5.1 示例：FileStream Sink Connector

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "local-file-sink",
    "config": {
        "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
        "tasks.max": "1",
        "file": "/var/log/exported-logs.txt",
        "topics": "app-logs"
    }
}'
```

## 5.2 SinkTask 核心逻辑

```java
public abstract class SinkTask {
    // ① 启动
    public void start(Map<String, String> props);

    // ② 消费并写入
    public abstract void put(Collection<SinkRecord> records);

    // ③ 刷出缓冲区
    public void flush(Map<TopicPartition, OffsetAndMetadata> offsets);

    // ④ 预提交（Kafka 2.6+）
    //    在 offset 提交前调用（可以做预检查或刷盘）
    public Map<TopicPartition, OffsetAndMetadata> preCommit(
        Map<TopicPartition, OffsetAndMetadata> offsets);
}
```

## 5.3 常用 Sink Connector

| Connector | 目标 | 用途 |
|-----------|------|------|
| S3 Sink | Amazon S3 | 数据存档到 S3 |
| HDFS Sink | HDFS | 数据入湖 |
| Elasticsearch Sink | Elasticsearch | 日志/数据索引 |
| JDBC Sink | 关系型数据库 | 写入数据库 |
| MongoDB Sink | MongoDB | 文档存储 |
| HTTP Sink | HTTP API | 通过 HTTP 推送到外部 |

### 5.3.1 Elasticsearch Sink 配置示例

```json
{
    "name": "elasticsearch-sink",
    "config": {
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "tasks.max": "2",
        "topics": "app-logs",
        "connection.url": "http://elasticsearch:9200",
        "type.name": "_doc",
        "key.ignore": "false",
        "schema.ignore": "true",
        "behavior.on.null.values": "delete",
        "batch.size": "2000",
        "linger.ms": "1000",
        "max.buffered.records": "20000",
        "flush.timeout.ms": "10000",
        "max.retries": "5",
        "retry.backoff.ms": "1000"
    }
}
```

# 6 Transform (SMT - Single Message Transform)

## 6.1 内置 Transform

```json
{
    "name": "my-connector",
    "config": {
        "connector.class": "...",
        "transforms": "renameTopic,insertSource,extractField",

        // ① ReplaceField: 字段增删改
        "transforms.extractField.type":
            "org.apache.kafka.connect.transforms.ReplaceField$Value",
        "transforms.extractField.whitelist": "id,title,content",

        // ② ExtractField: 只保留指定字段（Kafka 0.11.5+）
        "transforms.extractField.type":
            "org.apache.kafka.connect.transforms.ExtractField$Value",
        "transforms.extractField.field": "payload",

        // ③ InsertField: 插入静态字段
        "transforms.insertSource.type":
            "org.apache.kafka.connect.transforms.InsertField$Value",
        "transforms.insertSource.static.field": "source",
        "transforms.insertSource.static.value": "kafka-connect",

        // ④ ReplaceField: 重命名 Topic
        "transforms.renameTopic.type":
            "org.apache.kafka.connect.transforms.ReplaceField$Key",
        "transforms.renameTopic.renames": "topic:custom_topic",

        // ⑤ RegexRouter: 按正则路由到不同 Topic
        "transforms.route.type":
            "org.apache.kafka.connect.transforms.RegexRouter",
        "transforms.route.regex": "prefix-(.*)",
        "transforms.route.replacement": "$1-routed",

        // ⑥ TimestampConverter: 时间戳格式转换
        "transforms.tsConverter.type":
            "org.apache.kafka.connect.transforms.TimestampConverter$Value",
        "transforms.tsConverter.field": "timestamp",
        "transforms.tsConverter.format": "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'",
        "transforms.tsConverter.target.type": "string"
    }
}
```

## 6.2 自定义 Transform

```java
public class UppercaseTransform<R extends ConnectRecord<R>>
        implements Transformation<R> {

    @Override
    public R apply(R record) {
        String value = (String) record.value();
        String upperValue = value.toUpperCase();
        return record.newRecord(
            record.topic(),
            record.kafkaPartition(),
            record.keySchema(),
            record.key(),
            record.valueSchema(),
            upperValue,
            record.timestamp()
        );
    }

    @Override
    public ConfigDef config() {
        return new ConfigDef();
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

# 7 错误处理与死信队列

## 7.1 Sink Connector 的错误处理

```json
{
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dlq-my-connector",
    "errors.deadletterqueue.topic.replication.factor": 3,
    "errors.deadletterqueue.context.headers.enable": true,
    "errors.log.enable": true,
    "errors.log.include.messages": true
}
```

| `errors.tolerance` | 行为 |
|-------------------|------|
| `none` (默认) | 遇到错误立即失败，停止 Task |
| `all` | 所有错误被忽略或写入死信队列 |
| `all` (单作业失败) | 如果 SinkTask.put() 中某条记录失败，其余继续 |

## 7.2 死信队列（DLQ）记录格式

```
DLQ Topic 中每条记录:
  Key: 原始消息的 Key
  Value: 原始消息的 Value
  Headers:
    __connect.errors.topic:         "original-topic"
    __connect.errors.partition:     "0"
    __connect.errors.offset:        "12345"
    __connect.errors.connector.name:"my-connector"
    __connect.errors.task.id:       "0"
    __connect.errors.stage:         "VALUE_CONVERTER" 或 "TRANSFORMATION"
    __connect.errors.class.name:    "com.example.MyException"
    __connect.errors.exception.message: "Error processing..."
```

# 8 核心参数速查

## 8.1 Connector 级配置

```json
{
    "connector.class": "全限定类名",
    "tasks.max": "最大 Task 数",
    "topics": "逗号分隔的 Topic 列表 (Sink 用)",
    "topics.regex": "正则匹配的 Topic 列表 (Sink 用)",
    "key.converter": "Key 转换器",
    "value.converter": "Value 转换器",
    "header.converter": "Header 转换器",
    "transforms": "转换链",
    "errors.tolerance": "none|all",
    "errors.deadletterqueue.topic.name": "死信队列 Topic",
    "errors.retry.timeout": "重试超时ms",
    "errors.retry.delay.max.ms": "最大重试延迟ms"
}
```

## 8.2 Worker 级配置

```properties
# 任务分配
tasks.max=1                    # 默认单 Connector 最大 Task 数
offset.flush.interval.ms=60000 # Offset 刷盘间隔
offset.flush.timeout.ms=5000   # Offset 刷盘超时

# 内部 Topic 配置
config.storage.replication.factor=3
offset.storage.replication.factor=3
status.storage.replication.factor=3

# Admin 操作
connectors.creation.enable=true  # 允许通过 REST 创建 Connector
admin.listeners=https://0.0.0.0:8084 # Admin API 监听地址 (Kafka 3.4+)
```

# 9 常见运维操作

```bash
# 查看所有 Connector
curl -s http://localhost:8083/connectors | jq

# 查看 Connector 状态
curl -s http://localhost:8083/connectors/my-connector/status | jq

# 重启 Connector
curl -X POST http://localhost:8083/connectors/my-connector/restart

# 重启 Task
curl -X POST http://localhost:8083/connectors/my-connector/tasks/0/restart

# 暂停 Connector
curl -X PUT http://localhost:8083/connectors/my-connector/pause

# 恢复 Connector
curl -X PUT http://localhost:8083/connectors/my-connector/resume

# 更新 Connector 配置（PUT = 更新/创建）
curl -X PUT http://localhost:8083/connectors/my-connector/config \
  -H "Content-Type: application/json" \
  -d '{"connector.class": "...", ...}'

# 删除 Connector
curl -X DELETE http://localhost:8083/connectors/my-connector
```
