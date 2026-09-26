
# 1. 概述

Canal 支持将解析后的增量数据直接投递到 Kafka，这是生产环境中最常用的架构。相比 TCP 模式，Kafka 模式具有以下优势：

- **解耦**：生产者和消费者完全解耦
- **持久化**：Kafka 天然持久化，消息可回溯
- **多消费者**：多个消费组可独立消费同一份数据
- **吞吐量**：Kafka 的高吞吐特性适合大数据量场景

```
┌──────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│  MySQL   │───>│ Canal Server │───>│    Kafka     │───>│   Consumers  │
│          │    │ (Producer)   │    │   Cluster    │    │ (下游业务)    │
└──────────┘    └──────────────┘    └─────────────┘    └──────────────┘
                                                              │
                                              ┌───────────────┼───────────┐
                                              │               │           │
                                        ┌─────▼────┐   ┌─────▼────┐ ┌────▼─────┐
                                        │  Flink   │   │  Canal   │ │  Custom  │
                                        │  消费    │   │ Adapter  │ │  Service │
                                        └──────────┘   └──────────┘ └──────────┘
```

# 2. Canal 端配置

## 2.1 canal.properties 核心配置

```properties
#################################################
#########    Canal Server 模式    ##############
#################################################

# 切换为 Kafka 模式
canal.serverMode = kafka

#################################################
#########    Kafka 连接配置    #################
#################################################

# Kafka 集群地址（多个用逗号分隔）
kafka.bootstrap.servers = 192.168.1.10:9092,192.168.1.11:9092,192.168.1.12:9092

# Kafka 生产者确认模式
kafka.acks = all                     # 0 / 1 / all

# 消息压缩方式
kafka.compression.type = snappy      # none / gzip / snappy / lz4 / zstd

# 重试次数
kafka.retries = 3

# 批量发送配置
kafka.batch.size = 16384             # 16KB
kafka.linger.ms = 100                # 100ms 内攒够 batch.size 就发送
kafka.buffer.memory = 33554432       # 32MB 发送缓冲区

# 序列化类
kafka.kerberos.enable = false
kafka.kerberos.krb5.file = ""
kafka.kerberos.jaas.file = ""

# Kafka Producer 其他配置
kafka.max.request.size = 1048576     # 1MB
kafka.max.in.flight.requests.per.connection = 1
kafka.retry.backoff.ms = 100

# SASL认证（如果需要）
# kafka.sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required username="xxx" password="xxx";
# kafka.sasl.mechanism = PLAIN
# kafka.security.protocol = SASL_PLAINTEXT
```

## 2.2 Instance 级别配置

```properties
# instance.properties — Kafka 相关配置

# Kafka Topic 名称（动态 Topic 路由时使用）
canal.mq.topic = example

# Topic 的动态路由规则
# 为空则使用全局 topic，不为空则按规则投递到不同 topic
# 格式：schema.table:topic1,schema2.table2:topic2
canal.mq.dynamicTopic = .*\\.user:user_topic,.*\\.order:order_topic

# 分区策略配置
# 格式：schema.table:hashColumn1,hashColumn2
# 示例：test.user:id,test.order:user_id
canal.mq.partitionsNum = 3
canal.mq.partitionHash = test\\.user:id,test\\.order:order_id

# 消息序列化方式
canal.mq.flatMessage = true         # 使用扁平化消息（推荐）
                                     # false: Protobuf 格式

# 数据库层面的动态 Topic（按库名路由）
canal.mq.databaseTopic = test_db:test_topic,prod_db:prod_topic
```

# 3. Kafka 消息格式

## 3.1 FlatMessage JSON 格式（推荐）

```json
{
  "id": 1,
  "database": "test",
  "table": "users",
  "pkNames": ["id"],
  "isDdl": false,
  "type": "UPDATE",
  "es": 1672545600000,
  "ts": 1672545600005,
  "sql": "",
  "sqlType": {
    "id": "4",
    "name": "12",
    "email": "12", 
    "status": "-6"
  },
  "mysqlType": {
    "id": "bigint(20)",
    "name": "varchar(50)",
    "email": "varchar(100)",
    "status": "tinyint(4)"
  },
  "data": [
    {"id": "1", "name": "Alice", "email": "alice@example.com", "status": "0"}
  ],
  "old": [
    {"status": "1"}
  ]
}
```

## 3.2 Protobuf 格式

```json
{
  "destination": "example",
  "batchId": 1001,
  "entries": [
    {
      "header": {
        "logfileName": "mysql-bin.000003",
        "logfileOffset": 456789,
        "schemaName": "test",
        "tableName": "users", 
        "eventType": "UPDATE",
        "executeTime": 1672545600000,
        "gtid": "uuid:1-150"
      },
      "entryType": "ROWDATA",
      "storeValue": "<base64 encoded RowChange proto>"
    }
  ]
}
```

## 3.3 DDL 消息格式

```json
{
  "id": 100,
  "database": "test",
  "table": "users",
  "pkNames": [],
  "isDdl": true,
  "type": "ALTER",
  "es": 1672545600000,
  "ts": 1672545600005,
  "sql": "ALTER TABLE users ADD COLUMN phone VARCHAR(20)",
  "sqlType": {},
  "mysqlType": {},
  "data": [],
  "old": []
}
```

# 4. Topic 路由策略

## 4.1 静态 Topic（全局统一）

```properties
# 所有表的数据都投递到同一个 Topic
canal.mq.topic = canal_sync_data
```

## 4.2 动态 Topic（按表路由）

```properties
# 不同表投递到不同 Topic
canal.mq.dynamicTopic = db1\\.user:user_topic,db1\\.order:order_topic,.*\\..*:default_topic
```

## 4.3 动态 Topic 实现

```java
public class CanalKafkaProducer {
    
    /**
     * 根据消息内容动态选择 Topic
     */
    private String resolveTopic(CanalEntry.Entry entry) {
        
        // 1. 检查动态 Topic 规则
        if (StringUtils.isNotEmpty(dynamicTopic)) {
            String schemaName = entry.getHeader().getSchemaName();
            String tableName = entry.getHeader().getTableName();
            String fullName = schemaName + "." + tableName;
            
            // 解析动态 Topic 规则
            String[] rules = dynamicTopic.split(",");
            for (String rule : rules) {
                String[] parts = rule.split(":");
                if (parts.length == 2) {
                    String pattern = parts[0].trim();
                    String topic = parts[1].trim();
                    
                    if (fullName.matches(pattern)) {
                        return topic;
                    }
                }
            }
        }
        
        // 2. 检查按库路由
        if (StringUtils.isNotEmpty(databaseTopic)) {
            String schemaName = entry.getHeader().getSchemaName();
            String[] rules = databaseTopic.split(",");
            for (String rule : rules) {
                String[] parts = rule.split(":");
                if (parts[0].trim().equals(schemaName)) {
                    return parts[1].trim();
                }
            }
        }
        
        // 3. 回退到默认 Topic
        return defaultTopic;
    }
}
```

# 5. 分区策略

## 5.1 分区配置

```properties
# 分区数量
canal.mq.partitionsNum = 3

# 分区哈希规则
# 格式：schema.table:hashColumn1,hashColumn2
canal.mq.partitionHash = test\\.user:id,test\\.order:user_id
```

## 5.2 分区实现

```java
public class CanalKafkaPartitioner {
    
    private Map<String, List<String>> partitionHashConfig = new HashMap<>();
    private int partitionsNum;
    
    /**
     * 计算消息应该投递到哪个分区
     */
    public int partition(CanalEntry.Entry entry) {
        
        String schemaName = entry.getHeader().getSchemaName();
        String tableName = entry.getHeader().getTableName();
        String fullName = schemaName + "." + tableName;
        
        List<String> hashColumns = partitionHashConfig.get(fullName);
        
        if (hashColumns == null) {
            // 没有配置哈希列，随机分区
            return ThreadLocalRandom.current().nextInt(partitionsNum);
        }
        
        try {
            RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
            
            if (rowChange.getRowDatasCount() == 0) {
                return 0;
            }
            
            RowData rowData = rowChange.getRowDatas(0);
            
            // 从 after 或 before 列中提取哈希值
            StringBuilder hashValue = new StringBuilder();
            for (Column column : rowData.getAfterColumnsList()) {
                if (hashColumns.contains(column.getName())) {
                    hashValue.append(column.getValue());
                }
            }
            
            // 如果 after 没有，尝试 before（DELETE 场景）
            if (hashValue.length() == 0) {
                for (Column column : rowData.getBeforeColumnsList()) {
                    if (hashColumns.contains(column.getName())) {
                        hashValue.append(column.getValue());
                    }
                }
            }
            
            // 计算分区
            int hash = Math.abs(hashValue.toString().hashCode());
            return hash % partitionsNum;
            
        } catch (Exception e) {
            logger.warn("partition calculation failed, use random", e);
            return ThreadLocalRandom.current().nextInt(partitionsNum);
        }
    }
}
```

## 5.3 分区策略的重要性

```
正确分区 → 相同主键的数据进入同一分区 → 顺序消费，数据一致

错误分区 → 相同主键的 UPDATE 和 DELETE 进入不同分区 → 并发消费可能乱序

示例：
INSERT user #1 → Partition 0  (id=1 → hash=0)
UPDATE user #1 → Partition 0  (id=1 → hash=0)  ✓ 正确，在同一个 Partition
DELETE user #1 → Partition 0  (id=1 → hash=0)  ✓ 正确

INSERT order #1 → Partition 1
UPDATE order #1 → Partition 1  ✓ 正确，单分区内有序
```

# 6. Kafka 生产者详解

```java
public class CanalKafkaProducer implements CanalMQProducer {
    
    private KafkaProducer<String, String> producer;
    
    @Override
    public void init(Properties properties) {
        
        Properties kafkaProps = new Properties();
        
        // 基本配置
        kafkaProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, 
            properties.getProperty("kafka.bootstrap.servers"));
        kafkaProps.put(ProducerConfig.ACKS_CONFIG, 
            properties.getProperty("kafka.acks", "all"));
        kafkaProps.put(ProducerConfig.RETRIES_CONFIG, 
            Integer.parseInt(properties.getProperty("kafka.retries", "3")));
        
        // 序列化
        kafkaProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class.getName());
        kafkaProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class.getName());
        
        // 批量配置
        kafkaProps.put(ProducerConfig.BATCH_SIZE_CONFIG,
            Integer.parseInt(properties.getProperty("kafka.batch.size", "16384")));
        kafkaProps.put(ProducerConfig.LINGER_MS_CONFIG,
            Integer.parseInt(properties.getProperty("kafka.linger.ms", "100")));
        kafkaProps.put(ProducerConfig.BUFFER_MEMORY_CONFIG,
            Integer.parseInt(properties.getProperty("kafka.buffer.memory", "33554432")));
        
        // 压缩
        kafkaProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG,
            properties.getProperty("kafka.compression.type", "none"));
        
        // 幂等性（保证 exactly-once）
        kafkaProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // 最大请求大小
        kafkaProps.put(ProducerConfig.MAX_REQUEST_SIZE_CONFIG,
            Integer.parseInt(properties.getProperty("kafka.max.request.size", "1048576")));
        
        this.producer = new KafkaProducer<>(kafkaProps);
    }
    
    @Override
    public void send(CanalMQData data, Callback callback) {
        
        String topic = resolveTopic(data.getEntry());
        Integer partition = calculatePartition(data.getEntry());
        String key = buildMessageKey(data.getEntry());
        String value = buildMessageValue(data);
        
        ProducerRecord<String, String> record;
        if (partition != null) {
            record = new ProducerRecord<>(topic, partition, null, key, value);
        } else {
            record = new ProducerRecord<>(topic, null, key, value);
        }
        
        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                logger.error("kafka send failed: {}", exception.getMessage());
                callback.onFailure(exception);
            } else {
                callback.onSuccess(metadata);
            }
        });
    }
    
    /**
     * 构建 Kafka 消息 Key
     * Key 相同 → 进入同一分区（用于排序保证）
     */
    private String buildMessageKey(CanalEntry.Entry entry) {
        try {
            RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
            if (rowChange.getRowDatasCount() == 0) return null;
            
            RowData rowData = rowChange.getRowDatas(0);
            List<Column> columns = rowData.getAfterColumnsList();
            
            if (columns.isEmpty()) {
                columns = rowData.getBeforeColumnsList();
            }
            
            // 用主键作为 Key
            StringBuilder key = new StringBuilder();
            key.append(entry.getHeader().getSchemaName())
               .append(".")
               .append(entry.getHeader().getTableName())
               .append(":");
            
            for (Column col : columns) {
                if (col.getIsKey()) {
                    key.append(col.getValue()).append("_");
                }
            }
            
            return key.toString();
        } catch (Exception e) {
            return null;
        }
    }
}
```

# 7. 消费者示例

## 7.1 Java Kafka Consumer

```java
public class CanalKafkaConsumer {
    
    public static void main(String[] args) {
        
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "canal-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("canal_sync_data"));
        
        while (running) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            
            for (ConsumerRecord<String, String> record : records) {
                try {
                    // 解析 FlatMessage JSON
                    FlatMessage flatMessage = JSON.parseObject(
                        record.value(), FlatMessage.class);
                    
                    System.out.printf("topic=%s partition=%d offset=%d key=%s%n",
                        record.topic(), record.partition(), record.offset(), record.key());
                    System.out.printf("db=%s table=%s type=%s%n",
                        flatMessage.getDatabase(), flatMessage.getTable(), 
                        flatMessage.getType());
                    
                    // 处理数据
                    processFlatMessage(flatMessage);
                    
                } catch (Exception e) {
                    logger.error("process message error, partition={}, offset={}",
                        record.partition(), record.offset(), e);
                    // 记录错误但不抛异常（继续处理后续消息）
                }
            }
            
            // 手动提交 offset
            consumer.commitSync();
        }
        
        consumer.close();
    }
}
```

## 7.2 Flink SQL 消费 Canal-Kafka 数据

```sql
-- Flink SQL: 注册 Canal-Kafka Source
CREATE TABLE canal_source (
    id BIGINT,
    database STRING,
    `table` STRING,
    type STRING,
    es BIGINT,
    ts BIGINT,
    data MAP<STRING, STRING>,
    old MAP<STRING, STRING>
) WITH (
    'connector' = 'kafka',
    'topic' = 'canal_sync_data',
    'properties.bootstrap.servers' = 'localhost:9092',
    'properties.group.id' = 'flink-canal-group',
    'format' = 'json',
    'scan.startup.mode' = 'latest-offset'
);

-- 过滤并处理数据
SELECT
    database,
    `table`,
    type,
    data['id'] AS primary_key,
    data['name'] AS name,
    data['status'] AS status
FROM canal_source
WHERE `table` = 'users' AND type = 'UPDATE';
```

# 8. Canal Kafka 安全配置

## 8.1 SASL 认证

```properties
# SASL/PLAIN 认证
kafka.security.protocol = SASL_PLAINTEXT
kafka.sasl.mechanism = PLAIN
kafka.sasl.jaas.config = org.apache.kafka.common.security.plain.PlainLoginModule required username="canal" password="canal_secret";
```

## 8.2 Kerberos 认证

```properties
kafka.kerberos.enable = true
kafka.kerberos.krb5.file = /etc/krb5.conf
kafka.kerberos.jaas.file = /etc/canal/kafka_jaas.conf

# kafka_jaas.conf 内容：
# KafkaClient {
#     com.sun.security.auth.module.Krb5LoginModule required
#     useKeyTab=true
#     storeKey=true
#     keyTab="/etc/canal/canal.keytab"
#     principal="canal/hostname@REALM";
# };
```

# 9. 性能调优

## 9.1 Canal 端优化

```properties
# 生产环境推荐配置
kafka.acks = 1                         # 折衷考虑可靠性和性能
kafka.compression.type = lz4           # LZ4 平衡速度与压缩率
kafka.batch.size = 131072              # 128KB
kafka.linger.ms = 50                   # 50ms
kafka.buffer.memory = 67108864         # 64MB
kafka.max.in.flight.requests.per.connection = 5
kafka.max.request.size = 5242880       # 5MB
```

## 9.2 Kafka Broker 端建议

```properties
# server.properties — Broker 端优化建议
num.partitions = 6                     # 根据 TPS 设置合适的分区数
log.retention.hours = 168              # 保留 7 天
log.segment.bytes = 1073741824         # 1GB 一个段
compression.type = producer            # 使用生产者压缩
unclean.leader.election.enable = false # 禁止脏 Leader 选举
min.insync.replicas = 2               # 至少 2 个副本确认
```
