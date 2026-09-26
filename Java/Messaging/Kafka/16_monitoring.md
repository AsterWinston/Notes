# 1 监控概览

## 1.1 监控层次

```
┌──────────────────────────────────────────────────────────────┐
│                    Kafka 监控金字塔                            │
│                                                              │
│           ┌─────────────────────────────┐                    │
│           │  ④ 业务监控: 延迟、吞吐、数据量  │                    │
│           │     自定义 Metrics            │                    │
│           └─────────────┬───────────────┘                    │
│                         │                                    │
│           ┌─────────────▼───────────────┐                    │
│           │  ③ 消费监控: Lag, 提交, 重平衡  │                    │
│           │     Consumer Group Metrics   │                    │
│           └─────────────┬───────────────┘                    │
│                         │                                    │
│           ┌─────────────▼───────────────┐                    │
│           │  ② Broker 监控: 请求、网络、磁盘│                    │
│           │     Broker/Controller Metrics │                    │
│           └─────────────┬───────────────┘                    │
│                         │                                    │
│           ┌─────────────▼───────────────┐                    │
│           │  ① 系统监控: CPU, 内存, 磁盘, 网络│                 │
│           │     OS-Level Metrics          │                  │
│           └─────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────┘
```

# 2 JMX 指标体系

## 2.1 关键 Broker 指标

### 2.1.1 消息流指标

```bash
# 每分钟消息入站速率 (messages/s)
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=*
# OneMinuteRate, FiveMinuteRate, FifteenMinuteRate, MeanRate

# 每分钟字节入站速率 (bytes/s)
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=*

# 每分钟字节出站速率
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec,topic=*

# 每分钟消息入站速率
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=my-topic
```

### 2.1.2 请求指标

```bash
# 请求总数
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Fetch
# Mean, Max, 50thPercentile, 75thPercentile, 95thPercentile, 99thPercentile

# 请求队列大小
kafka.network:type=RequestChannel,name=RequestQueueSize

# 网络处理器空闲率
kafka.network:type=SocketServer,name=NetworkProcessorAvgIdlePercent
# 值 < 0.3 说明网络线程不足

# I/O 线程空闲率
kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
# 值 < 0.3 说明 I/O 线程不足
```

### 2.1.3 副本与 ISR 指标

```bash
# 未同步副本数（应始终为 0）
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions

# ISR 收缩速率
kafka.server:type=ReplicaManager,name=IsrShrinksPerSec

# ISR 扩展速率
kafka.server:type=ReplicaManager,name=IsrExpandsPerSec
```

### 2.1.4 日志与磁盘指标

```bash
# Offline 日志目录数（应始终为 0）
kafka.log:type=LogManager,name=OfflineLogDirectoryCount

# 日志刷盘速率
kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs

# Cleaner 清理速率（Compact 策略）
kafka.log:type=LogCleaner,name=cleaner-recopy-percent
```

### 2.1.5 Controller 指标

```bash
# 活跃 Controller 计数（始终为 1）
kafka.controller:type=KafkaController,name=ActiveControllerCount

# Controller 选举速率（频繁选举有问题）
kafka.controller:type=ControllerStats,name=LeaderElectionRateAndTimeMs

# 当前离线分区数
kafka.controller:type=KafkaController,name=OfflinePartitionsCount
```

## 2.2 关键 Consumer Group 指标

```bash
# Consumer Lag（最重要！）
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*,topic=*,partition=*
# records-lag-max: 最大 Lag
# records-lag: 当前 Lag

# Consumer 消费速率
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*
# records-consumed-rate: 消费速率(条/s)
# bytes-consumed-rate: 消费速率(bytes/s)

# Fetch 延迟
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*
# fetch-latency-avg, fetch-latency-max

# Rebalance 统计
kafka.consumer:type=consumer-coordinator-metrics,client-id=*
# rebalance-total, rebalance-rate-per-hour
```

## 2.3 关键 Producer 指标

```bash
# 发送速率
kafka.producer:type=producer-metrics,client-id=*
# record-send-rate, byte-send-rate

# 发送延迟
kafka.producer:type=producer-metrics,client-id=*
# request-latency-avg, request-latency-max

# 缓冲区使用
kafka.producer:type=producer-metrics,client-id=*
# buffer-available-bytes: 剩余缓冲区
# buffer-total-bytes: 总缓冲区

# 错误率
kafka.producer:type=producer-metrics,client-id=*
# record-error-rate, record-retry-rate
```

# 3 常用命令行监控工具

## 3.1 kafka-consumer-groups（查看消费状况）

```bash
# 列出所有 Consumer Group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# 查看指定 Group 的消费状态（最重要！）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group my-group

# 输出:
# GROUP       TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# my-group    my-topic  0          1050            1500            450
# my-group    my-topic  1          1100            1100            0
# my-group    my-topic  2          980             2000            1020
#
# LAG = LOG-END-OFFSET - CURRENT-OFFSET
# Consumer 落后 450 条消息

# 查看所有有 Lag 的 Group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --all-groups --state |

grep "LAG" -A 1000  # 简化查看
```

## 3.2 kafka-topics（查看 Topic 状态）

```bash
# 查看所有 Topic
kafka-topics.sh --bootstrap-server localhost:9092 --list

# 查看特定 Topic 详情
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --topic my-topic

# 查看 ISR 不足的 Partition
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --under-replicated-partitions

# 查看 ISR 不满足 min.insync.replicas 的 Partition
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --under-min-isr-partitions

# 查看不可用 Partition
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --unavailable-partitions
```

## 3.3 kafka-log-dirs（查看磁盘使用）

```bash
# 查看各 Broker 磁盘使用详情
kafka-log-dirs.sh --bootstrap-server localhost:9092 \
    --describe --broker-list 0,1,2

# 输出: 每个分区在磁盘上的大小
# topic:my-topic  partition:0  size:1073741824  offset-lag:0
```

# 4 指标采集方案

## 4.1 Prometheus + JMX Exporter

```bash
# 1. 下载 JMX Exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar

# 2. 配置 kafka.yml (JMX Exporter 配置)
```

```yaml
# kafka.yml
lowercaseOutputName: true
lowercaseOutputLabelNames: true

rules:
  # Broker 消息流
  - pattern: kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec, topic=(.+)><>OneMinuteRate
    name: kafka_messages_in_per_sec
    labels:
      topic: "$1"

  - pattern: kafka.server<type=BrokerTopicMetrics, name=BytesInPerSec, topic=(.+)><>OneMinuteRate
    name: kafka_bytes_in_per_sec
    labels:
      topic: "$1"

  - pattern: kafka.server<type=BrokerTopicMetrics, name=BytesOutPerSec, topic=(.+)><>OneMinuteRate
    name: kafka_bytes_out_per_sec
    labels:
      topic: "$1"

  # 未同步副本（最重要！）
  - pattern: kafka.server<type=ReplicaManager, name=UnderReplicatedPartitions><>Value
    name: kafka_under_replicated_partitions

  # 活跃 Controller
  - pattern: kafka.controller<type=KafkaController, name=ActiveControllerCount><>Value
    name: kafka_active_controller_count

  # Consumer Group Lag
  - pattern: kafka.consumer<type=consumer-fetch-manager-metrics, client-id=(.+), topic=(.+), partition=(.+)><>(records-lag-max|records-lag)
    name: kafka_consumer_lag
    labels:
      client_id: "$1"
      topic: "$2"
      partition: "$3"

  # 网络处理器空闲率
  - pattern: kafka.network<type=SocketServer, name=NetworkProcessorAvgIdlePercent><>Value
    name: kafka_network_processor_idle

  # 请求处理空闲率
  - pattern: kafka.server<type=KafkaRequestHandlerPool, name=RequestHandlerAvgIdlePercent><>Value
    name: kafka_request_handler_idle
```

```bash
# 3. 启动 Broker 时加载 JMX Exporter
export KAFKA_OPTS="\
    -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=9091:/opt/jmx_exporter/kafka.yml"
bin/kafka-server-start.sh config/server.properties
```

## 4.2 Datadog Agent

```yaml
# datadog-agent/conf.d/kafka.d/conf.yaml
init_config:
instances:
  - host: localhost
    port: 9999
    name: kafka_broker_1
    tags:
      - env:production
      - kafka_cluster:main
```

# 5 外部监控工具

## 5.1 Burrow（Consumer Lag 监控）

Burrow 是 LinkedIn 开源的 Kafka Consumer Lag 检测工具，能评估 Consumer 状态而非仅看 Lag 数值。

```
Burrow 的 Consumer Status 评估:
  - OK: 消费正常
  - WARNING: Lag 在增长但未跨过分区边界
  - ERROR: Lag 已跨越分区边界，或 Consumer 停止消费
  - NOTFOUND: Consumer Group 未找到
```

```bash
# 配置 burrow.cfg
[general]
cluster=local

[cluster.local]
class-name=kafka
servers=broker1:9092,broker2:9092
zookeeper=zk1:2181,zk2:2181/kafka

[httpserver.default]
port=8000

# 启动
./burrow --config-dir /etc/burrow
```

## 5.2 Cruise Control（集群平衡与监控）

```bash
# 启动 Cruise Control
bin/cruise-control-start.sh config/cruisecontrol.properties

# API 查询副本分布
curl "http://localhost:9090/kafkacruisecontrol/partition_load?resource=CPU"

# 执行自动 Rebalance
curl -X POST "http://localhost:9090/kafkacruisecontrol/rebalance?dryrun=false"
```

## 5.3 Kafka Manager (CMAK)

```
功能:
  - 可视化查看 Topic 清单和详情
  - 创建/删除 Topic
  - 查看 Consumer Group 状态和 Lag
  - 分区 Reassignment 操作
  - Broker 列表和监控
  - 集群级别的指标仪表盘
```

# 6 告警规则

## 6.1 关键告警（Prometheus AlertManager 示例）

```yaml
groups:
  - name: kafka-alerts
    rules:
      # 1. 未同步副本
      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_under_replicated_partitions > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Kafka has under-replicated partitions"
          description: "{{ $value }} partitions are under-replicated on {{ $labels.broker }}"

      # 2. ISO 副本不足
      - alert: KafkaUnderMinISR
        expr: kafka_under_min_isr_partitions > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Partition ISR below min.insync.replicas"

      # 3. Consumer Lag 过大
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_lag{records_lag_max!=""} > 10000
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Consumer lag is > 10000 for topic {{ $labels.topic }}:{{ $labels.partition }}"

      # 4. Consumer Group 离线
      - alert: KafkaConsumerGroupOffline
        expr: kafka_consumergroup_members == 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Consumer group {{ $labels.group }} has no active members"

      # 5. Broker 宕机
      - alert: KafkaBrokerDown
        expr: kafka_server_replicafetchermanager_maxlag > 0 and absent(kafka_controller_active_controller_count)
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka broker may be down"

      # 6. 没有活跃 Controller
      - alert: KafkaNoActiveController
        expr: kafka_active_controller_count != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No active Kafka controller"

      # 7. 请求处理延迟过高
      - alert: KafkaHighRequestLatency
        expr: kafka_request_latency_ms{quantile="0.99"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "99th percentile request latency is above 1s"
```

## 6.2 监控检查清单

| 检查项 | 正常值 | 告警阈值 | 频率 |
|--------|--------|---------|------|
| UnderReplicatedPartitions | 0 | > 0 | 每分钟 |
| ActiveControllerCount | 1 | != 1 | 每分钟 |
| Consumer Lag | 视业务而定 | > 10000 或持续增长 | 每分钟 |
| OfflinePartitionsCount | 0 | > 0 | 每分钟 |
| NetworkProcessorIdle | > 0.3 | < 0.2 | 每 5 分钟 |
| RequestHandlerIdle | > 0.3 | < 0.2 | 每 5 分钟 |
| 磁盘使用率 | < 70% | > 80% | 每 5 分钟 |
| OS CPU | < 70% | > 85% | 每 5 分钟 |
| GC 暂停时间 | < 100ms | > 500ms | 每 5 分钟 |

# 7 运维命令速查

```bash
###############################
# Broker 运维
###############################

# 关闭 Broker（优雅关闭，等所有 Partition 迁移）
kafka-server-stop.sh

# 关闭 Broker（从集群中移除前）
# 1. 先迁移所有 Partition 到其他 Broker
# 2. 再停止 Broker
# 3. 最后从 ZK 移除 /brokers/ids/{brokerId}

###############################
# ZK/KRaft 元数据检查
###############################

# ZK 模式
zookeeper-shell.sh localhost:2181 ls /brokers/ids
zookeeper-shell.sh localhost:2181 get /controller

# KRaft 模式
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 \
    describe --status

###############################
# 日志段检查
###############################

# 查看 Partition 的日志文件
ls -lh /var/lib/kafka/data/my-topic-0/

# 解析日志文件内容
kafka-dump-log.sh --files /var/lib/kafka/data/my-topic-0/00000000000000000000.log
kafka-dump-log.sh --files /var/lib/kafka/data/my-topic-0/00000000000000000000.index
kafka-dump-log.sh --files /var/lib/kafka/data/my-topic-0/00000000000000000000.timeindex

###############################
# 查看 Broker 版本
###############################
kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```
