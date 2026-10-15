# 1. Controller 概述

## 1.1 Controller 是什么

Controller 是 Kafka 集群中的**大脑**，负责管理所有 Partition 和副本的状态。每个集群同时只有一个活跃的 Controller（Leader Controller），其余为备用（Standby）。

```
┌──────────────────────────────────────────────────────────────┐
│                    Controller 职责                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 1. Broker 管理                                           │ │
│  │    ├── 监听 /brokers/ids → Broker 上/下线                 │ │
│  │    ├── 处理 Broker 下线后该 Broker 上的 Partition 重新分配 │ │
│  │    └── 管理 Preferred Leader Election                    │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │ 2. Partition 管理                                        │ │
│  │    ├── 处理 Topic 创建/删除（Partition 创建/分配）         │ │
│  │    ├── 管理 Partition Reassignment（分区副本迁移）         │ │
│  │    └── 新增 Partition 的分配（Topic 扩容分区数）            │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │ 3. Leader 选举                                            │ │
│  │    ├── Partition Leader 选举                              │ │
│  │    ├── ISR 变更通知                                       │ │
│  │    └── Leader And Isr 请求下发                             │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │ 4. 元数据广播                                             │ │
│  │    ├── 向所有 Broker 发送 UpdateMetadata 请求               │ │
│  │    └── 维护整个集群的 Metadata Cache                       │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

# 2. Controller 选举

## 2.1 ZooKeeper 模式下的选举

```
Controller 选举基于 ZooKeeper 的临时节点 + Watch 机制：

┌─────────────────────────────────────────────────────┐
│              ZooKeeper 中 Controller 相关节点          │
│                                                     │
│  /controller                                        │
│  ├── { "version": 1,                               │
│  │     "brokerid": 0,    ← 当前 Controller          │
│  │     "timestamp": "..." }                         │
│  │                                                  │
│  └── 临时节点 (EPHEMERAL)                            │
│      当 Controller Broker 宕机 → 节点消失             │
│      其他 Broker 的 Watcher 被触发 → 抢注册             │
│      最先成功创建 /controller 的 Broker 成为新 Controller│
└─────────────────────────────────────────────────────┘
```

### 2.1.1 选举流程

```
① 所有 Broker 启动时 → 尝试在 ZooKeeper 创建 /controller 临时节点
② 第一个成功创建的 Broker → 成为 Controller
③ 失败的 Broker → 在 /controller 上注册 Watcher
④ Controller Broker 宕机 → /controller 消失 → 触发 Watcher
⑤ 其他 Broker 重新争抢 → 新的 Controller 产生
⑥ 新 Controller 从 ZooKeeper 读取最新的元数据 → 接管全局管理

Controller Epoch (controller_epoch):
  - 每次 Controller 变更 → epoch + 1
  - 其他 Broker 收到带 epoch 的请求时 → 检查 epoch 是否最新
  - 旧 Controller 发送的请求（epoch 过期）→ 被拒绝
```

## 2.2 KRaft 模式下的选举

KRaft 模式下 Controller 不再依赖 ZooKeeper，而是通过 Raft 共识协议选举：

```
┌───────────────────────────────────────────────────┐
│            KRaft (Kafka Raft)                      │
│                                                   │
│  Quorum Voters: {1, 2, 3}                        │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Node 1   │  │ Node 2   │  │ Node 3   │        │
│  │ Leader   │  │ Follower │  │ Follower │        │
│  │(Controller)│ │          │  │          │        │
│  └──────────┘  └──────────┘  └──────────┘        │
│                                                   │
│  Metadata Topic: __cluster_metadata               │
│  └── 使用 Raft 日志存储元数据（替代 ZooKeeper）       │
└───────────────────────────────────────────────────┘
```

# 3. Controller 核心状态机

## 3.1 Partition 状态机

```
Partition State Machine:

           ┌──────────┐
           │NonExistent│  ← Partition 不存在
           └────┬─────┘
                │ 创建 Topic / 新增 Partition
                ▼
           ┌──────────┐
           │ NewPartition│
           └────┬─────┘
                │ Leader 选举完成
                ▼
           ┌──────────┐
      ┌───▶│  Online  │  ← 正常服务状态
      │    └────┬─────┘
      │         │ 分区删除
      │         ▼
      │    ┌──────────┐
      │    │  Offline  │  ← Leader 不可用
      │    └────┬─────┘
      │         │ 重新选举 Leader
      └─────────┘
```

## 3.2 Replica 状态机

```
Replica State Machine:

           ┌──────────┐
           │  NewReplica│
           └────┬─────┘
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
   ┌────────┐┌────────┐┌────────┐
   │Online  ││Offline ││Replica │
   │Replica ││Replica ││Deletion│
   │        ││        ││Started │
   └────────┘└────────┘└────┬───┘
                            │ 删除完成
                            ▼
                       ┌──────────┐
                       │NonExistent│
                       │Replica    │
                       └──────────┘

OnlineReplica 子状态:
  ├── Leader    (负责读写)
  └── Follower  (从 Leader 同步)
```

# 4. Leader 选举算法

## 4.1 选举优先策略

```scala
// PartitionLeaderSelector
// 按优先级顺序选择新 Leader:

def selectLeader(partition: Partition): Int = {
  val isr = partition.inSyncReplicas
  val leaderReplicas = partition.leaderReplicas  // 当前 Leader 副本

  if (isr.contains(leaderReplicas.head)) {
    // 优先 1: 当前 Leader 仍在 ISR 中（续任）
    leaderReplicas.head
  } else {
    // 优先 2: ISR 中的任意副本
    isr.head
  }
  // 优先 3 (如果 unclean.leader.election.enable=true):
  //   从 AR（Assigned Replicas）中选择
}

// 同等条件下优先选择在 ISR 中存活时间最长的副本
```

## 4.2 LeaderAndIsr 请求

Controller 选举出 Leader 后，通过 `LeaderAndIsr` 请求将结果下发到各 Broker：

```scala
case class LeaderAndIsrRequest(
  controllerId: Int,
  controllerEpoch: Int,
  brokerEpoch: Long,
  partitionStates: Map[TopicPartition, LeaderAndIsrPartitionState]
)

// LeaderAndIsrPartitionState:
//   topic: String
//   partition: Int
//   controllerEpoch: Int
//   leader: Int            ← 新 Leader 的 Broker ID
//   leaderEpoch: Int       ← Leader Epoch（递增）
//   isr: List[Int]         ← 当前 ISR 列表
//   partitionEpoch: Int    ← Partition Epoch
//   replicas: List[Int]    ← 所有副本列表
//   addingReplicas: List[Int]
//   removingReplicas: List[Int]
```

### 4.2.1 Broker 处理 LeaderAndIsr 请求

```
Broker 收到 LeaderAndIsr 请求:

① 检查 controllerEpoch（拒绝旧 Controller 的请求）
② 对于每个 Partition State:
   ├── 自己是新 Leader → 初始化 Leader Partition
   │   ├── startHighWatermarkCheckPoint
   │   └── maybeIncrementLeaderHW
   └── 自己是 Follower → 初始化 Follower Partition
       └── 注册到 ReplicaFetcherThread（从新 Leader 拉取数据）
③ 完成 → 响应 Controller
```

# 5. 元数据传播

## 5.1 UpdateMetadata 请求

Controller 周期性地（或元数据变更时）向所有 Broker 发送 `UpdateMetadata` 请求：

```scala
case class UpdateMetadataRequest(
  controllerId: Int,
  controllerEpoch: Int,
  brokerEpoch: Long,
  partitionStates: Map[TopicPartition, UpdateMetadataPartitionState],
  liveBrokers: Seq[UpdateMetadataBroker]
)

// Controller 中的触发时机:
// ① Broker 上线/下线
// ② Topic 创建/删除
// ③ Partition 变更（Leader 切换、ISR 变更）
// ④ 手动触发
```

## 5.2 Metadata 请求

Producer 和 Consumer 通过 `Metadata` 请求获取集群元数据：

```java
// Producer 首次发送或元数据过期时，会发送 Metadata 请求
// 请求:
MetadataRequest(topics=["my-topic"], allowAutoTopicCreation=false)

// 响应:
MetadataResponse {
  brokers: [{id: 0, host: "broker1", port: 9092}, ...]
  topicMetadata: [{
    topic: "my-topic",
    partitions: [{
      partition: 0,
      leader: 0,              // Leader 所在 Broker
      replicas: [0, 1, 2],   // 所有副本
      isr: [0, 1]            // ISR
    }, ...]
  }]
}
```

### 5.2.1 Metadata 缓存刷新

```java
// Producer 端
props.put("metadata.max.age.ms", 300000); // 默认 5 分钟刷新一次

// Consumer 端同样可以配置

// 元数据强制刷新的触发条件:
// ① metadata.max.age.ms 到期
// ② 发生 UNKNOWN_TOPIC_OR_PARTITION 错误
// ③ 发送消息时发现 Leader 不可用
```

# 6. ZK 路径结构（Controller 相关）

## 6.1 关键 ZK 节点

```
ZooKeeper 中 Kafka 的节点结构:

/kafka（或自定义的 chroot）
│
├── /controller
│   └── {"brokerid":0, "timestamp":"...", "version":1}
│       (临时节点, Controller 选举用)
│
├── /controller_epoch
│   └── 42  (Controller Epoch, 每次 Controller 变更 +1)
│
├── /brokers
│   ├── /ids
│   │   ├── /0 → {"host":"broker1","port":9092,...}
│   │   ├── /1 → {"host":"broker2","port":9093,...}
│   │   └── /2 → ...
│   │
│   ├── /topics
│   │   └── /my-topic
│   │       └── /partitions
│   │           ├── /0/state → {"leader":0,"isr":[0,1,2],...}
│   │           ├── /1/state → {"leader":1,"isr":[1,2],...}
│   │           └── /2/state → ...
│   │
│   └── /seqid  (Broker ID 自动生成用)
│
├── /config
│   ├── /topics/my-topic → {"config":"..."}
│   ├── /brokers/0 → {"config":"..."}
│   └── /changes/config_changes  (配置变更通知)
│
├── /admin
│   ├── /delete_topics          (待删除 Topic)
│   └── /reassign_partitions    (待重新分配的 Partition)
│
├── /consumers (已废弃, 旧版 Consumer 使用)
│
└── /isr_change_notification    (ISR 变更通知)
```

## 6.2 Controller 对 ZK 节点的监听

```scala
// KafkaController 需要监听的 ZK 路径
class KafkaController {
  // 1. Broker 变更
  zkClient.subscribeBrokerChangeListener(brokerChangeHandler)

  // 2. Topic 变更
  zkClient.subscribeTopicChangeListener(topicChangeHandler)

  // 3. Partition 重组（Admin 操作）
  zkClient.subscribePartitionReassignment(partitionReassignmentHandler)

  // 4. Preferred Leader Election
  zkClient.subscribePreferredReplicaElection(preferredReplicaElectionHandler)

  // 5. ISR 变更
  zkClient.subscribeIsrChangeNotification(isrChangeNotificationHandler)

  // 6. 日志目录变更
  zkClient.subscribeLogDirEventNotification(logDirEventNotificationHandler)
}
```

# 7. KRaft 模式详解（Kafka 3.3+）

## 7.1 为什么从 ZooKeeper 迁移到 KRaft

| 问题 | ZooKeeper 模式 | KRaft 模式 |
|------|--------------|-----------|
| 元数据延迟 | ZK 写入 + Controller 感知 = 两跳 | 单跳（Raft Leader 直接处理） |
| 分区数上限 | ZK 能力瓶颈（~20 万分区/集群） | 理论百万分区 |
| 运维复杂度 | 需要运维 ZooKeeper 集群 | 纯 Kafka 运维 |
| 故障恢复 | 依赖 ZK session 超时（~18s） | Raft 选举（更快） |
| Controller 切换 | ZK 节点消失 + 重新注册（秒级） | Raft 选举（毫秒级） |

## 7.2 KRaft 架构

```
┌───────────────────────────────────────────────────────────┐
│                    KRaft 架构                              │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Node 1     │  │   Node 2     │  │   Node 3     │     │
│  │ broker       │  │ broker       │  │ broker       │     │
│  │ controller   │  │ controller   │  │ controller   │     │
│  │ (Active)     │  │ (Standby)    │  │ (Standby)    │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                 │                 │              │
│         └────────┬────────┴────────┬────────┘              │
│                  │    Raft 共识     │                       │
│                  ▼                 ▼                       │
│         ┌──────────────────────────────────┐               │
│         │  __cluster_metadata              │               │
│         │  (存储元数据的 Raft 日志)          │               │
│         │  ├── Broker 注册信息              │               │
│         │  ├── Topic 配置                  │               │
│         │  └── Partition 信息              │               │
│         └──────────────────────────────────┘               │
│                                                           │
│  对比 ZK 模式:                                             │
│  之前的 ZK 节点数据 → 存储在 __cluster_metadata Topic       │
│  Raft Leader = Active Controller                          │
└───────────────────────────────────────────────────────────┘
```

### 7.2.1 KRaft 角色配置

```properties
# Node 1: 既是 Broker 又是 Controller 且为 Voter
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@node1:9093,2@node2:9093,3@node3:9093

# Node 2: 同上
process.roles=broker,controller
node.id=2
controller.quorum.voters=1@node1:9093,2@node2:9093,3@node3:9093

# Node 3: 同上
```

### 7.2.2 KRaft 元数据快照

```
__cluster_metadata Topic:
  ├── Segment 0: [metadata record 1, record 2, ...]
  ├── Segment 1: [record n, record n+1, ...]
  └── ...

  每条 Metadata Record:
    - TopicRecord (Topic 创建/更新)
    - PartitionRecord (Partition 创建/状态变更)
    - BrokerRegistrationRecord (Broker 注册)
    - ConfigRecord (配置变更)
    - FenceBrokerRecord (隔离故障 Broker)
    - RemoveTopicRecord (Topic 删除)
    - ...
```

# 8. Controller 故障处理

## 8.1 Controller 切换

```
Controller Broker 0 宕机:
│
├── ZK 模式:
│   ① /controller 临时节点消失（session timeout ~18s）
│   ② 其他 Broker 的 Watcher 被触发
│   ③ 争抢创建 /controller 节点
│   ④ 新 Controller 出现 → 从 ZK 提取全量元数据
│   ⑤ 发送 LeaderAndIsr / UpdateMetadata → 恢复集群管理
│
└── KRaft 模式:
    ① Raft Leader 心跳超时
    ② Follower 发起选举
    ③ 新 Raft Leader 产生 → 同时成为 Controller
    ④ 直接拥有最新的元数据（Raft 日志一致）
    ⑤ 零数据恢复 → 更快接管
```

## 8.2 Split Brain（脑裂）问题

```
ZK 模式下如何防止脑裂:

Controller Epoch 机制:
  - 每次新 Controller 获取 ZK /controller_epoch 并 +1
  - 新 Controller 发送的请求附带新的 controllerEpoch
  - 其他 Broker 收到请求时检查：
    if (request.controllerEpoch < localEpoch) {
      reject("Controller epoch is too old")  // 拒绝旧 Controller
    }

  结果: 即使旧 Controller 以为自己是 Controller
       它发出的请求因 epoch 过期被所有 Broker 拒绝
       避免脑裂导致的数据不一致
```

# 9. 常见问题

## 9.1 Controller 选举慢（ZooKeeper 模式）

```
症状: Controller 宕机后，集群长时间无响应

原因:
  - ZooKeeper Session 超时过大 (zookeeper.session.timeout.ms=18000)
  - 新 Controller 加载全量元数据耗时长（Partition 数量多）

排查:
  # 查看 Controller 状态
  bin/zookeeper-shell.sh localhost:2181 get /controller

  # 查看 Controller 日志
  grep "became leader" server.log
```

## 9.2 Controller 重启导致集群不可用

```
原因: 新 Controller 初始化时，需要从 ZK 拉取所有 Topic 和 Partition 信息
      并处理每一个 Partition 的 Leader 选举
      如果 Topic/Partition 数量很大，这个过程可能需要数分钟

缓解:
  - 使用 KRaft 模式（原生快照恢复）
  - 减少 Partition 总数（合并不必要的 Topic）
  - 适当增大 zookeeper.session.timeout.ms
```
