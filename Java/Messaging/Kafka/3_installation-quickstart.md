# 1 环境要求

## 1.1 基础环境

| 组件 | 版本要求 | 说明 |
|------|---------|------|
| JDK | 8+（Kafka 3.x 需要 Java 11+） | OpenJDK / Oracle JDK |
| OS | Linux / macOS / Windows | 生产环境建议 Linux |
| 内存 | Broker 建议 6GB+ | 默认 JVM Heap 1GB |
| 磁盘 | SSD 推荐 | 顺序 I/O 为主 |

## 1.2 Kafka 与 ZooKeeper 版本关系

| Kafka 版本 | 依赖 ZooKeeper | KRaft 支持 |
|-----------|---------------|-----------|
| ≤ 2.8 | 必须 | 早期预览 |
| 2.8 - 3.2 | 推荐（可 KRaft 预览） | 预览 |
| 3.3+ | 可选（KRaft 推荐） | 生产可用 |

# 2 单机安装（ZooKeeper 模式）

## 2.1 下载与解压

```bash
# 下载 Kafka（以 3.6.0 为例）
wget https://dlcdn.apache.org/kafka/3.6.0/kafka_2.13-3.6.0.tgz

# 解压
tar -xzf kafka_2.13-3.6.0.tgz
cd kafka_2.13-3.6.0
```

## 2.2 启动 ZooKeeper

```bash
# 使用 Kafka 自带的 ZooKeeper（生产环境应独立部署）
bin/zookeeper-server-start.sh config/zookeeper.properties
```

## 2.3 修改 Broker 配置

编辑 `config/server.properties`：

```properties
# Broker 唯一 ID
broker.id=0

# 监听地址
listeners=PLAINTEXT://localhost:9092

# 日志存储目录
log.dirs=/tmp/kafka-logs

# ZooKeeper 连接地址
zookeeper.connect=localhost:2181

# 默认分区数和副本数
num.partitions=3
default.replication.factor=1

# 日志保留策略
log.retention.hours=168
log.segment.bytes=1073741824
```

## 2.4 启动 Broker

```bash
# 后台启动
bin/kafka-server-start.sh -daemon config/server.properties

# 前台启动（便于观察日志）
bin/kafka-server-start.sh config/server.properties

# 查看是否启动成功
jps | grep Kafka
```

## 2.5 基本操作

```bash
# 1. 创建 Topic
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
    --create --topic quickstart-events \
    --partitions 3 --replication-factor 1

# 2. 查看 Topic 列表
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# 3. 查看 Topic 详情
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --topic quickstart-events

# 输出示例：
# Topic: quickstart-events  PartitionCount: 3  ReplicationFactor: 1
#   Topic: quickstart-events  Partition: 0  Leader: 0  Replicas: 0  Isr: 0
#   Topic: quickstart-events  Partition: 1  Leader: 0  Replicas: 0  Isr: 0
#   Topic: quickstart-events  Partition: 2  Leader: 0  Replicas: 0  Isr: 0

# 4. 发送消息（Console Producer）
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 \
    --topic quickstart-events

> hello kafka
> this is a test message
> ^C

# 5. 消费消息（Console Consumer）
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic quickstart-events --from-beginning

# 6. 查看 Consumer Group 信息
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group my-group
```

# 3 单机安装（KRaft 模式，Kafka 3.3+）

KRaft 模式不需要 ZooKeeper，Kafka 使用自身的 Raft 共识协议管理元数据。

## 3.1 生成集群 ID

```bash
KAFKA_CLUSTER_ID=$(bin/kafka-storage.sh random-uuid)
echo $KAFKA_CLUSTER_ID
```

## 3.2 格式化存储

```bash
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
```

## 3.3 修改 KRaft 配置

编辑 `config/kraft/server.properties`：

```properties
# 使用 KRaft 模式
process.roles=broker,controller

# Controller 连接地址
controller.quorum.voters=1@localhost:9093

# 监听地址
listeners=PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT

# 日志目录
log.dirs=/tmp/kraft-logs

# Controller 用的元数据目录（与 log.dirs 不同）
metadata.log.dir=/tmp/kraft-metadata-logs
```

## 3.4 启动

```bash
bin/kafka-server-start.sh -daemon config/kraft/server.properties
```

# 4 集群安装

## 4.1 ZooKeeper 集群模式拓扑

```
┌──────────────────┐
│   ZooKeeper 集群  │
│  zk1:2181        │
│  zk2:2181        │
│  zk3:2181        │
└────────┬─────────┘
         │
    ┌────┴──────────────┐
    │                   │
    ▼                   ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│Broker 0 │    │Broker 1 │    │Broker 2 │
│:9092    │    │:9093    │    │:9094    │
└─────────┘    └─────────┘    └─────────┘
```

## 4.2 Broker 配置文件

**Broker 0（`config/server-0.properties`）**：

```properties
broker.id=0
listeners=PLAINTEXT://0.0.0.0:9092
log.dirs=/tmp/kafka-logs-0
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```

**Broker 1（`config/server-1.properties`）**：

```properties
broker.id=1
listeners=PLAINTEXT://0.0.0.0:9093
log.dirs=/tmp/kafka-logs-1
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```

**Broker 2（`config/server-2.properties`）**：
类似配置，`broker.id=2`，`listeners=PLAINTEXT://0.0.0.0:9094`。

## 4.3 启动集群

```bash
# 分别启动各 Broker
bin/kafka-server-start.sh -daemon config/server-0.properties
bin/kafka-server-start.sh -daemon config/server-1.properties
bin/kafka-server-start.sh -daemon config/server-2.properties

# 验证
bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092 | head
```

## 4.4 创建 Topic（3 副本）

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
    --create --topic test-topic \
    --partitions 3 --replication-factor 3 \
    --config min.insync.replicas=2 \
    --config retention.ms=604800000
```

# 5 Java 客户端快速入门

## 5.1 Maven 依赖

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.6.0</version>
</dependency>
```

## 5.2 Producer 示例

```java
import org.apache.kafka.clients.producer.*;
import java.util.Properties;
import java.util.concurrent.Future;

public class QuickStartProducer {
    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                "org.apache.kafka.common.serialization.StringSerializer");
        // 开启幂等（默认 true for 3.0+）
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        // 等待所有 ISR 确认
        props.put(ProducerConfig.ACKS_CONFIG, "all");

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
            for (int i = 0; i < 100; i++) {
                ProducerRecord<String, String> record = new ProducerRecord<>(
                        "quickstart-events",
                        "key-" + i,
                        "message-" + i
                );

                producer.send(record, (metadata, exception) -> {
                    if (exception != null) {
                        exception.printStackTrace();
                    } else {
                        System.out.printf("Sent to topic=%s partition=%d offset=%d%n",
                                metadata.topic(), metadata.partition(), metadata.offset());
                    }
                });
            }

            // 确保所有消息发送完成
            producer.flush();
        }
    }
}
```

## 5.3 Consumer 示例

```java
import org.apache.kafka.clients.consumer.*;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class QuickStartConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "quickstart-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                "org.apache.kafka.common.serialization.StringDeserializer");
        // 从头开始消费（当没有已提交 offset 时）
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        // 关闭自动提交
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        // 使用 Cooperative Rebalance
        props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
                "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("quickstart-events"));

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("offset=%d, key=%s, value=%s, partition=%d%n",
                            record.offset(), record.key(), record.value(), record.partition());
                }

                // 手动提交 offset
                consumer.commitSync();
            }
        }
    }
}
```

# 6 Admin Client 操作

```java
import org.apache.kafka.clients.admin.*;
import java.util.*;

public class QuickStartAdmin {
    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        try (AdminClient admin = AdminClient.create(props)) {
            // 创建 Topic
            NewTopic newTopic = new NewTopic("admin-topic", 3, (short) 1);
            admin.createTopics(Collections.singleton(newTopic)).all().get();

            // 列出 Topic
            ListTopicsResult topics = admin.listTopics();
            topics.names().get().forEach(System.out::println);

            // 描述 Topic
            DescribeTopicsResult describe = admin.describeTopics(
                    Collections.singleton("admin-topic"));
            describe.all().get().forEach((name, desc) ->
                System.out.printf("Topic: %s, Partitions: %d%n",
                        name, desc.partitions().size())
            );

            // 列出 Consumer Group
            admin.listConsumerGroups().all().get().forEach(g ->
                System.out.printf("Group: %s%n", g.groupId())
            );

            // 删除 Topic
            admin.deleteTopics(Collections.singleton("admin-topic")).all().get();
        }
    }
}
```

# 7 Kafka 脚本工具速查

```bash
###############################
# Topic 管理
###############################

# 创建 Topic
kafka-topics.sh --bootstrap-server localhost:9092 \
    --create --topic my-topic --partitions 6 --replication-factor 3

# 列出 Topic
kafka-topics.sh --bootstrap-server localhost:9092 --list

# 查看 Topic 详情
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic my-topic

# 增加分区（不支持减少）
kafka-topics.sh --bootstrap-server localhost:9092 \
    --alter --topic my-topic --partitions 12

# 修改配置
kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type topics --entity-name my-topic \
    --alter --add-config retention.ms=86400000

# 删除 Topic
kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic my-topic

###############################
# Consumer Group 管理
###############################

# 列出所有 Consumer Group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# 查看 Group 详情
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group my-group

# 重置 Offset（到最早）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --group my-group --topic my-topic --reset-offsets --to-earliest --execute

# 重置 Offset（到最新）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --group my-group --topic my-topic --reset-offsets --to-latest --execute

# 按时间重置 Offset
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --group my-group --topic my-topic \
    --reset-offsets --to-datetime 2024-01-01T00:00:00.000 --execute

# 删除 Consumer Group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --delete --group my-group

###############################
# 性能测试
###############################

# Producer 性能测试
kafka-producer-perf-test.sh \
    --topic perf-test \
    --num-records 1000000 \
    --record-size 1024 \
    --throughput -1 \
    --producer-props bootstrap.servers=localhost:9092 \
        acks=1 batch.size=16384 linger.ms=5

# Consumer 性能测试
kafka-consumer-perf-test.sh \
    --topic perf-test \
    --messages 1000000 \
    --bootstrap-server localhost:9092

###############################
# 查看日志
###############################

# 查看 Segment 文件内容
kafka-dump-log.sh --files /tmp/kafka-logs/my-topic-0/00000000000000000000.log \
    --print-data-log

# 查看索引文件
kafka-dump-log.sh --files /tmp/kafka-logs/my-topic-0/00000000000000000000.index
```

# 8 Docker 部署

## 8.1 docker-compose.yml（ZooKeeper 模式）

```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

## 8.2 docker-compose.yml（KRaft 模式）

```yaml
version: '3'
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      CLUSTER_ID: 'ciWo7IWazngRchmPES6q5A'
```

```bash
# 启动
docker-compose up -d

# 进入容器
docker exec -it <kafka-container-id> bash
```
