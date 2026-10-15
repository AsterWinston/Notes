# 1. Kafka 概述与演进

## 1.1 Kafka 是什么

Apache Kafka 是一个开源的**分布式事件流平台**（Distributed Event Streaming Platform），最初由 LinkedIn 公司开发，2011 年捐赠给 Apache 软件基金会。

Kafka 从诞生之初就不是一个单纯的消息队列，其设计目标包含三个核心能力：

1. **发布与订阅**：像消息队列一样，以高吞吐量发布和订阅记录流
2. **持久化存储**：以容错、持久的方式存储记录流
3. **流式处理**：在记录产生时对其进行实时处理

> 一句话总结：Kafka = 消息引擎系统 + 分布式存储 + 流处理平台

## 1.2 发展历程

| 版本 | 时间 | 关键特性 |
|------|------|----------|
| 0.7.x | 2011 | 首个 Apache 版本，基础发布/订阅 |
| 0.8.x | 2013 | 引入分区、副本机制、Consumer 可以回溯消费 |
| 0.9.x | 2015 | 引入新的 Consumer API、SSL 安全 |
| 0.10.x | 2016 | Kafka Streams、Timestamp 索引、SASL |
| 0.11.x | 2017 | 精确一次语义（Exactly-Once）、幂等生产者、事务 |
| 1.x | 2017 | 消除 ZooKeeper 依赖的准备工作、Controller 改进 |
| 2.x | 2019 | 增量 Cooperative Rebalance、基于 Raft 的共识（KRaft 早期） |
| 3.x | 2022 | KRaft 生产可用、完全移除 ZooKeeper 准备、分层存储预览 |

## 1.3 与其它消息中间件对比

### 1.3.1 Kafka vs RabbitMQ

| 维度 | Kafka | RabbitMQ |
|------|-------|----------|
| 设计哲学 | 分布式日志（Log） | 消息代理（Broker） |
| 吞吐量 | 百万级/秒 | 万级/秒 |
| 消息持久化 | 默认持久化到磁盘 | 支持持久化和暂存 |
| 消费模型 | Pull 模型 | Push 模型 |
| 路由模型 | Topic → Partition → Consumer | Exchange → Binding → Queue |
| 消息回溯 | 支持（按 offset/时间） | 不支持（消费即删除） |
| 消息顺序 | Partition 内有序 | 单个队列有序 |
| 延迟 | 毫秒级 | 微秒级 |
| 协议 | 自定义 TCP 协议 | AMQP 0-9-1 |
| 流处理 | Kafka Streams/KSQL | 无 |

### 1.3.2 Kafka vs RocketMQ

| 维度 | Kafka | RocketMQ |
|------|-------|-----------|
| 注册中心 | ZooKeeper / KRaft | NameServer |
| 事务消息 | 支持 | 支持（原生分布式事务） |
| 延迟消息 | 不支持原生 | 支持（18 个延迟级别） |
| 消息过滤 | 无原生服务端过滤 | 支持 SQL 表达式过滤 |
| 顺序消息 | Partition 内有序 | 全局有序 + 分区有序 |
| 语言实现 | Java + Scala | 纯 Java |
| 生态 | Kafka Streams / Connect | 较弱 |

### 1.3.3 Kafka vs Pulsar

| 维度 | Kafka | Pulsar |
|------|-------|--------|
| 存储计算分离 | Broker 即存储 | 原生分离（BookKeeper） |
| 多租户 | 较弱 | 原生支持 |
| 消息确认 | 连续 Offset | 单条 ACK |
| 消费模型 | 单一消费模型 | Stream + Queue 统一 |
| Geo-Replication | MirrorMaker | 原生支持 |

## 1.4 为什么 Kafka 这么快

Kafka 高性能的核心原因：

### 1.4.1 顺序写入磁盘

Kafka 利用磁盘顺序写入的高性能（600MB/s+）远超随机写入（100KB/s）。消息以追加（append）方式写入日志文件，避免了磁盘随机寻址。

### 1.4.2 页缓存（Page Cache）

Kafka 重度依赖操作系统的页缓存来缓存最近写入和读取的数据，避免在 JVM 中维护对象缓存，从而避免 GC 开销。

### 1.4.3 零拷贝（Zero-Copy）

Kafka 使用 `sendfile()` 系统调用，数据直接从磁盘文件传输到网卡，不经过用户态内存拷贝：

```
传统路径：Disk → Kernel Buffer → User Buffer → Socket Buffer → NIC
零拷贝：Disk → Kernel Buffer → Socket Buffer → NIC (sendfile)
```

### 1.4.4 批量处理

Producer 端：使用 `RecordAccumulator` 将消息按 Partition 批量聚合，压缩后发送。
Consumer 端：使用 `fetch.min.bytes` 和 `fetch.max.wait.ms` 批量拉取。

### 1.4.5 分区并行

Topic 被分割为多个 Partition，生产者和消费者可以在不同分区上并行工作，实现水平扩展。

## 1.5 核心应用场景

### 1.5.1 日志收集与聚合

```
App → FileBeat/Kafka Producer → Kafka Cluster → Logstash/Flume → Elasticsearch
```

### 1.5.2 消息系统（解耦、削峰、异步）

```
Order Service → Kafka → Inventory Service / Payment Service / Notification Service
```

### 1.5.3 流处理

```
User Click Stream → Kafka → Kafka Streams → Real-time Dashboard
```

### 1.5.4 事件溯源（Event Sourcing）

Kafka 作为不可变事件日志，记录所有状态变更事件，系统可通过重放事件恢复任意时刻状态。

### 1.5.5 CDC（Change Data Capture）

```
MySQL Binlog → Debezium (Kafka Connect) → Kafka → Hive/HBase/ClickHouse
```

### 1.5.6 指标与监控

```
Metrics Agent → Kafka → Flink/Spark Streaming → Time-series DB (InfluxDB/Prometheus)
```

## 1.6 Kafka 生态体系

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Ecosystem                          │
├───────────────┬──────────────┬───────────────┬──────────────┤
│  Kafka Core   │   Connect    │    Streams    │   KSQL       │
│  (Broker)     │ (数据集成)    │  (流处理)      │ (流式SQL)    │
├───────────────┼──────────────┼───────────────┼──────────────┤
│  Producer API │  Consumer API │  Admin API   │  Interceptor │
├───────────────┼──────────────┼───────────────┼──────────────┤
│                    MirrorMaker 2 (跨集群复制)                 │
├─────────────────────────────────────────────────────────────┤
│   Schema Registry (Avro/Protobuf) │  Cruise Control (运维)   │
└─────────────────────────────────────────────────────────────┘
```

## 1.7 核心设计思想

### 1.7.1 不可变提交日志（Immutable Commit Log）

Kafka 的核心数据结构是仅追加（append-only）的提交日志。所有消息一旦写入便不可修改，仅能顺序读取。

这种设计带来的好处：
- 顺序 I/O 性能极致
- 简化复制机制（只需复制日志）
- 消费者可回溯到任意位置
- 天然支持多订阅者

### 1.7.2 消费者驱动的拉取模型

Kafka 使用 Pull 模型（Consumer 主动从 Broker 拉取数据），而非 Push 模型。好处：
- Consumer 按自身处理能力控制消费速率
- Broker 无需维护消费者状态
- 支持批量拉取

### 1.7.3 分区是并行的基本单位

Partition 是 Kafka 水平扩展和并行的基本单元。每个 Partition 是一个有序、不可变的消息序列。Partition 数量决定了一个 Topic 的最大并行度。

### 1.7.4 "Dumb Broker, Smart Consumer"

传统模型中，Broker 负责消息路由、投递、确认等复杂逻辑。Kafka 反其道而行之：
- Broker：仅负责存储和顺序分发，不跟踪消费进度
- Consumer：自行管理消费位置（offset），灵活控制

## 1.8 从消息队列到事件流

### 1.8.1 传统消息队列思维的局限

传统 MQ（RabbitMQ / ActiveMQ）将消息视为"任务"——消费完即可丢弃。这导致：
- 消息无法回溯
- 无法支持多消费者独立消费
- 事件无法被重新处理

### 1.8.2 事件流的思维转变

Kafka 将数据视为一个持续的事件流（日志流）：
- 事件是事实的不可变记录
- 多个消费者独立消费同一个日志
- 系统状态 = 事件流的折叠结果

```java
// 事件流思维示例：订单事件
// key: orderId, value: { eventType: CREATED/UPDATED/CANCELLED, ... }
// 多个服务独立消费同一条事件流
//   └── OrderService: 维护订单状态
//   └── InventoryService: 扣减库存
//   └── AnalyticsService: 实时统计
//   └── AuditService: 审计日志
```

## 1.9 本文组织结构

本系列笔记按照从理论到源码、从应用到原理的顺序组织：

- **第 1-3 篇**：概述、架构概念、快速入门（建立全局认知）
- **第 4-5 篇**：Producer 和 Consumer 深度剖析（客户端核心）
- **第 6-10 篇**：Broker 架构、存储、复制、Controller、分区管理（服务端核心）
- **第 11-12 篇**：事务、精确一次、投递语义（高级特性）
- **第 13-14 篇**：Kafka Streams 与 Kafka Connect（生态组件）
- **第 15-17 篇**：安全、监控、性能调优（运维实践）
- **第 18-19 篇**：版本对比、故障排查（进阶知识）
- **第 20 篇**：Spring 集成（开发实践）
- **第 21-26 篇**：源码分析（深度理解）
- **第 27 篇**：最佳实践（总结提炼）
