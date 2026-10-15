# 1. 架构总览

## 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Kafka 集群架构                                │
│                                                                     │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐                     │
│  │ Producer A│   │ Producer B│   │ Producer C│  ...                 │
│  └─────┬─────┘   └─────┬─────┘   └─────┬─────┘                     │
│        │ push(msg,topic)│               │                           │
│        ▼                ▼               ▼                           │
│  ┌─────────────────────────────────────────────────┐                │
│  │                  Kafka Cluster                   │                │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │                │
│  │  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │       │                │
│  │  │ P0=Leader│  │ P1=Leader│  │ P2=Leader│       │                │
│  │  │ P1=Folwr │  │ P2=Folwr │  │ P0=Folwr │       │                │
│  │  └──────────┘  └──────────┘  └──────────┘       │                │
│  └─────────────────────────────────────────────────┘                │
│        │ pull(offset)  │               │                            │
│        ▼               ▼               ▼                            │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐                     │
│  │Consumer G1│   │Consumer G2│   │Consumer G3│  ...                 │
│  └───────────┘   └───────────┘   └───────────┘                     │
│                                                                     │
│  ┌─────────────────────────────────────────────────┐                │
│  │              ZooKeeper (或 KRaft)                │                │
│  │   /brokers/ids, /brokers/topics, /controller    │                │
│  └─────────────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────────┘
```

## 1.2 Topic（主题）

Topic 是 Kafka 中消息的逻辑分类单元，类似于数据库中的表或文件系统中的文件夹。

```
Topic "order-events"
├── Partition 0: [msg0, msg1, msg2, msg3, msg4, ...]
├── Partition 1: [msg0, msg1, msg2, msg3, ...]
└── Partition 2: [msg0, msg1, msg2, msg3, msg4, msg5, ...]
```

### 1.2.1 关键特性

- **逻辑概念**：Topic 仅仅是一个逻辑名称，实际数据存储在 Partition 中
- **全局唯一**：在一个 Kafka 集群中 Topic 名称必须唯一
- **多订阅者**：一个 Topic 可以被多个 Consumer Group 同时消费
- **无限制**：Topic 的数量理论上没有上限（受限于 ZooKeeper 节点数）

### 1.2.2 Topic 命名建议

```
<业务域>.<数据类型>.<环境>

例如：
inventory.stock-change.prod
payment.transaction.dev
user.behavior-tracking.prod
```

## 1.3 Partition（分区）

Partition 是 Kafka 存储和并行的基本单位，每个 Partition 是一个有序、不可变的消息序列。

### 1.3.1 本质

Partition 在磁盘上体现为一个目录，包含多个 Segment 文件：

```
/tmp/kafka-logs/
└── my-topic-0/          ← Partition 0 目录
    ├── 00000000000000000000.log      ← 数据文件
    ├── 00000000000000000000.index    ← 偏移量索引
    ├── 00000000000000000000.timeindex← 时间戳索引
    ├── 00000000000000000536.log
    ├── 00000000000000000536.index
    └── 00000000000000000536.timeindex
```

### 1.3.2 分区数量的影响

| 分区数 | 并发度 | 单分区吞吐 | 故障恢复 | 元数据开销 |
|--------|--------|-----------|----------|-----------|
| 过少 | 低 | 高 | 慢 | 小 |
| 适中 | 中 | 中 | 中 | 中 |
| 过多 | 高 | 低 | 中 | 大 |

### 1.3.3 分区数规划公式

```
期望吞吐量 / 单分区吞吐量 = 分区数

示例：
目标：100MB/s，单分区：10MB/s
所需分区数 ≈ 10

考虑到增长：实际配置 2~3 倍 → 20~30 分区
```

经验值：
- `T/P = Max(期望吞吐/单分区吞吐, 期望Consumer线程数)`
- 一个 Topic 的分区数通常不超过 Broker 数量 × 集群分区上限 / Topic 数量

## 1.4 Offset（偏移量）

Offset 是消息在 Partition 中唯一的顺序编号，从 0 开始递增。

### 1.4.1 Offset 的三重含义

```
"order-events" Partition 0
┌─────┬─────┬─────┬─────┬──────────┬─────┐
│ 100 │ 101 │ 102 │ 103 │   104    │ 105 │
├─────┼─────┼─────┼─────┼──────────┼─────┤
│     │     │     │     │          │     │
│     │     LEO   │     │          │     │
│     │           │     │          │     │
│     Consumer    │     │          │     │
│     Offset (已消费)   │          │     │
│                      │          │     │
│               Log End Offset   │     │
│               (最后写入位置)     │     │
│                                │     │
│                          HW (High Watermark)
│                          (已提交的位置)
└────────────────────────────────────────┘
```

- **Consumer Offset**：Consumer 已消费到的位置（每个 Consumer Group 维护）
- **Log End Offset (LEO)**：Partition 中最后一条消息的 offset + 1
- **High Watermark (HW)**：ISR 中所有副本都已确认的 offset，消费者最多消费到此处

### 1.4.2 Offset 与时间戳的对应

Kafka 支持按时间戳查找 offset：

```java
// 查找指定时间之后的第一条消息的 offset
Map<TopicPartition, OffsetAndTimestamp> offsets = consumer
    .offsetsForTimes(Collections.singletonMap(
        new TopicPartition("topic", 0),
        System.currentTimeMillis() - 3600000  // 1 小时前
    ));
```

## 1.5 Broker（代理节点）

Broker 是 Kafka 集群中的单个服务节点，负责接收、存储和转发消息。

### 1.5.1 Broker 的核心职责

```
Broker 节点
├── SocketServer（网络层）
│   ├── Acceptor Thread（接收连接）
│   ├── Processor Threads（读写网络 I/O）
│   └── RequestChannel（请求队列）
├── KafkaApis（请求处理层）
│   ├── handleProduceRequest
│   ├── handleFetchRequest
│   ├── handleOffsetCommitRequest
│   └── ...（共 50+ 种请求类型）
├── ReplicaManager（副本管理层）
├── LogManager（日志管理层）
├── GroupCoordinator（消费者组协调器）
└── TransactionCoordinator（事务协调器）
```

### 1.5.2 Broker ID

每个 Broker 有唯一整数 ID，可以是手动配置的 `broker.id` 或自动生成。

## 1.6 集群与副本

### 1.6.1 Leader 与 Follower

每个 Partition 在多个 Broker 上有副本，其中一个为 Leader，其余为 Follower：

```
Topic "orders", 3 Partitions, Replication Factor = 3

Broker 1         Broker 2         Broker 3
┌─────────┐     ┌─────────┐     ┌─────────┐
│ P0=L    │     │ P2=L    │     │ P1=L    │
│ P1=F    │     │ P1=F    │     │ P0=F    │
│ P2=F    │     │ P0=F    │     │ P2=F    │
└─────────┘     └─────────┘     └─────────┘

L = Leader（读写）
F = Follower（仅同步复制）
```

### 1.6.2 ISR（In-Sync Replicas）

ISR 是与 Leader 保持同步的副本集合，包含 Leader 本身。当 Follower 落后超过 `replica.lag.time.max.ms`（默认 30s），会被移出 ISR。

```
初始: ISR = [Broker1, Broker2, Broker3]
Broker2 挂掉 40 秒后: ISR = [Broker1, Broker3]
Broker2 恢复追赶: ISR = [Broker1, Broker2, Broker3]
```

### 1.6.3 Leader 选举

- Kafka Controller 负责选举新 Leader
- 优先从 **AR（Assigned Replicas）中的 ISR** 中选择
- 若设置了 `unclean.leader.election.enable=false`（默认），当 ISR 为空时不允许选举

## 1.7 Consumer Group（消费者组）

Consumer Group 是 Kafka 实现消息广播和单播的关键机制。

### 1.7.1 基于 Group 的消费模型

```
Topic "orders" (3 Partitions)
│
├── P0 ─────▶ Consumer Group "order-service"
│    ├── P1 ──▶ Consumer A (消费 P0)
│    └── P2 ──▶ Consumer B (消费 P1, P2)
│
└── P0 ─────▶ Consumer Group "analytics" (独立消费)
     ├── P1 ──▶ Consumer C (独立消费 P0, P1, P2)
     └── P2
```

规则：
- 同一个 Group 内，一个 Partition 只能被一个 Consumer 消费
- 不同 Group 之间消费完全独立，每个 Group 维护自己的 Offset
- Consumer 数量 > Partition 数量时，多余 Consumer 闲置

### 1.7.2 Group 内部协调

```
Consumer Group "order-service"
│
├── Consumer A (memberId=A)
│   └── 分配: [P0]
│
├── Consumer B (memberId=B, Group Leader)
│   └── 分配: [P1, P2]
│   └── 负责: 执行 Partition 分配策略
│
└── GroupCoordinator (运行在某 Broker 上)
    └── 负责: 管理组成员、接收 Offset 提交、触发 Rebalance
```

## 1.8 Rebalance（重平衡）

Rebalance 是 Consumer Group 内成员变化或订阅 Topic 分区变化时，重新分配分区的过程。

### 1.8.1 触发条件

1. **Group 成员变更**：Consumer 加入/离开/崩溃
2. **订阅 Topic 分区数变化**：动态增加了分区
3. **订阅 Topic 模式变化**：Consumer 通过正则匹配，新 Topic 匹配到模式

### 1.8.2 Rebalance 协议

Kafka 支持两种 Rebalance 协议：

**Eager Protocol（默认 < 2.4）**：
- 发生 Rebalance 时所有 Consumer 停止消费（Stop-The-World）
- 所有 Consumer 重新加入 Group
- 缺点：全局暂停，影响较大

```
阶段: ① 所有 Consumer 停止消费 → ② 全部重新加入 → ③ 重新分配 → ④ 恢复消费
```

**Cooperative Protocol（>= 2.4, 增量式）**：
- Consumer 不会全部停止，仅涉及变动的分区重新分配
- 分配分批进行，逐批恢复消费

```
阶段: ① 新 Consumer 加入 → ② 仅释放/分配部分分区 → ③ 其余 Consumer 继续消费
```

### 1.8.3 Partition 分配策略

| 策略 | 类名 | 描述 |
|------|------|------|
| Range | `RangeAssignor` | 按 Topic 逐一分配，Partition 连续分配给 Consumer |
| RoundRobin | `RoundRobinAssignor` | 所有 Topic 的 Partition 排序后轮询分配 |
| Sticky | `StickyAssignor` | 尽量保持上次分配结果，减少分区迁移 |
| Cooperative Sticky | `CooperativeStickyAssignor` | 基于 Sticky 的增量分配（Cooperative Rebalance） |

## 1.9 Segment（日志分段）

每个 Partition 由多个 Segment 文件构成，Segment 是日志分段的最小单位。

```
Partition 0
├── 00000000000000000000.log  (大小: 1GB)
│   ├── offset=0,  msg: "hello"
│   ├── offset=1,  msg: "world"
│   └── offset=2,  msg: "kafka"
│
├── 00000000000000000003.log  (大小: 1GB)
│   ├── offset=3,  msg: "foo"
│   └── offset=4,  msg: "bar"
│
└── 00000000000000000005.log  (活跃 Segment，正在写入)
    └── offset=5,  msg: "baz"  ← 当前写入位置
```

### 1.9.1 Segment 命名规则

Segment 文件名 = 该 Segment 中第一条消息的 Offset（20 位数字，不足前面补零）

### 1.9.2 Segment 滚动条件

满足以下任一条件时创建新 Segment：
- 当前 Segment 大小达到 `log.segment.bytes`（默认 1GB）
- 当前 Segment 存在时间达到 `log.roll.ms`（默认 7 天）
- 当前 Segment 的索引或时间索引满了
- Append 消息时 Offset 与索引间距过大

## 1.10 总结

Kafka 的架构围绕以下设计原则展开：

- **分布式日志**：数据以不可变的提交日志形式组织
- **分区并行**：通过 Partition 实现水平扩展
- **副本容错**：通过 Leader-Follower 模型实现高可用
- **消费者组**：通过 Group 机制实现广播和单播的统一
- **偏移量自管理**：Consumer 自行管理消费进度，Broker 保持简单
