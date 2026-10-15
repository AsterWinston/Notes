# 1. 副本机制概述

## 1.1 副本拓扑

```
Topic "orders", Partitions=3, Replication Factor=3

           Broker 1              Broker 2              Broker 3
        ┌──────────┐          ┌──────────┐          ┌──────────┐
        │ P0=Leader│          │ P2=Leader│          │ P1=Leader│
        │ P1=Folwr │          │ P1=Folwr │          │ P0=Folwr │
        │ P2=Folwr │          │ P0=Folwr │          │ P2=Folwr │
        └──────────┘          └──────────┘          └──────────┘

        副本集合:
        P0: Replicas={1,2,3}, Leader=1, ISR={1,3}
        P1: Replicas={1,2,3}, Leader=3, ISR={3,2}
        P2: Replicas={1,2,3}, Leader=2, ISR={2,1,3}
```

## 1.2 核心概念对照表

| 概念 | 英文 | 说明 |
|------|------|------|
| AR | Assigned Replicas | 所有分配的副本（包括 Leader 和 Follower） |
| ISR | In-Sync Replicas | 与 Leader 保持同步的副本集合 |
| OSR | Out-of-Sync Replicas | 不在 ISR 中的副本（AR - ISR） |
| LEO | Log End Offset | 每个副本最后一条消息的 Offset + 1 |
| HW | High Watermark | ISR 中最小的 LEO，消费者最多消费到此处 |
| Leader | Leader 副本 | 负责读写，同步数据给 Follower |
| Follower | Follower 副本 | 被动从 Leader 同步数据 |

# 2. Leader 与 Follower 同步机制

## 2.1 同步过程

```
时间 ──────────────────────────────────────────────────▶

Leader (Broker 1, P0)               Follower (Broker 2, P0)
│                                    │
│ ① Producer 发送消息到 Leader       │
│    appendRecords → LEO=102        │
│                                    │
│ ② Follower 定期发送 FetchRequest  │
│    (fetch_offset=100)             │◄──────────────┐
│                                    │               │
│ ◄── Leader 返回 FetchResponse ────│               │
│     records=[offset=100,101]       │               │
│     leader_hw=99                   │               │
│                                    │               │
│                                    │ ③ Follower    │
│                                    │    appendRecords│
│                                    │    → LEO=102   │
│                                    │    → HW=min(LEOs)│
│                                    │      = min(102,102)
│                                    │      = 102      │
│                                    │               │
│ ④ Follower 下一轮 Fetch            │               │
│    携带自己 LEO=102 ──────────────▶│               │
│                                    │               │
│ ⑤ Leader 更新 HW                   │               │
│    HW = min(ISR 内所有副本的 LEO)    │               │
│    = min(102, 102) = 102           │               │
│                                    │               │
│ ⑥ 下一轮 FetchResponse 返回 HW=102 │               │
│    ──────────────────────────────▶ │               │
│                                    │ Follower 感知  │
│                                    │ HW 推进到 102   │
│                                    │               │
└────────────────────────────────────┴───────────────┘
```

## 2.2 HW 与 LEO 的更新机制

```
场景: 单 Leader (broker 0), 单 Follower (broker 1)

第 1 步: Leader 收到消息 offset=100
  broker 0: LEO=101, HW=100
  broker 1: LEO=100, HW=100

第 2 步: Follower 拉取 offset=100
  broker 0: LEO=101, HW=100          ← Leader 不知道 Follower 已成功
  broker 1: LEO=100 → 101, HW=100    ← Follower 更新了 LEO

第 3 步: Follower 成功写入后发下一轮 Fetch
  携带 fetch_offset=101 (告知 Leader 自己的 LEO)

第 4 步: Leader 收到 Follower 的 LEO=101
  更新 Remote Replica LEO=101
  HW = min(Leader LEO=101, Remote LEO=101) = 101

  broker 0: LEO=101, HW=100 → 101
  broker 1: 等待下一轮 FetchResponse 中获知 HW=101
```

关键：
- **Follower** 通过 **FetchResponse** 感知 HW
- **Leader** 通过 **FetchRequest 中携带的 Follower LEO** 感知 Follower 进度

# 3. ISR 机制详解

## 3.1 ISR 收缩与扩展

```
ISR 收缩条件 (shrink):
  Follower 超过 replica.lag.time.max.ms 未拉取消息
  或 Follower LEO 落后太多 (replica.lag.max.messages, 已废弃)

ISR 扩展条件 (expand):
  Follower 重新追赶上 Leader 且连续
  replica.lag.time.max.ms 时间内在持续拉取
```

### 3.1.1 收缩流程

```
时间线:
t=0s:    ISR = [0, 1, 2],  Follower 2 正常同步
t=10s:   Follower 2 宕机（不再发送 FetchRequest）
t=40s:   Leader 发现 Follower 2 超过 30s 未同步
         → ISR = [0, 1] （收缩）
t=80s:   Follower 2 恢复
t=100s:  Follower 2 持续拉取并追赶，且 30s 内未落后
         → ISR = [0, 1, 2] （扩展）
```

### 3.1.2 ISR 持久化

ISR 变更后需要持久化到 ZooKeeper：

```
/brokers/topics/{topicName}/partitions/{partitionId}/state

{
  "controller_epoch": 42,
  "leader": 0,
  "version": 1,
  "leader_epoch": 15,
  "isr": [0, 1, 2]
}
```

## 3.2 关键参数

```properties
# ISR 收缩阈值
# Follower 超过此时间未拉取 → 移出 ISR
replica.lag.time.max.ms=30000  # 默认 30s

# 注意: replica.lag.max.messages 已废弃（Kafka 0.9+）
# 原因: 按消息数判断在吞吐波动时不准确
#       高峰期（10万条/s）vs 低谷期（100条/s）
#       相同的消息数差距在实际时间上完全不同
```

# 4. Leader Epoch 机制

## 4.1 问题背景：HW 截断的不安全性

```
场景: acks=1, min.insync.replicas=1

第 1 步: Leader A 写入 offset=100,101 → LEO_A=102, HW_A=100
第 2 步: Follower B 拉取 → LEO_B=101 (只复制了 offset=100)
第 3 步: Leader A 宕机
第 4 步: Follower B 当选为新 Leader → HW_B=101

问题: Leader A 上有 offset=101 这条仅自己确认的消息
      新 Leader B 上没有 offset=101

A 重启后恢复为 Follower:
  A 检查自己的 LEO=102 > 新 Leader B 的 LEO=101
  按 HW 截断: Truncate to HW=101
  但 A 的 HW=100 < B 的 HW=101
  → A 截断到 101 (以 B 的 HW 为准)
  → offset=101 被保留（仅在 A 上）
  ❌ offset=101 实际是"脏数据"（未被多数确认）

  如果 B 未收到过 offset=101:
    A 截断到 101 保留了不应该有的数据
```

## 4.2 Leader Epoch 解决方案

Leader Epoch 是一个单调递增的整数，每次 Leader 变更时递增。通过 (LeaderEpoch, Offset) 绑定来安全截断。

```
LeaderEpochCache 记录每次 Leader 变更时的起始 Offset:

Leader Epoch=0: start_offset=0
Leader Epoch=1: start_offset=100  (Leader 变更)
Leader Epoch=2: start_offset=200  (Leader 变更)

恢复时:
  A 向新 Leader B 请求自己最近 Leader Epoch 的结束 Offset
  请求: OffsetsForLeaderEpochRequest(epoch=2)
  响应: end_offset=200

  A 的 LEO=250, 仅截断到 200
  截掉 200~250 之间的消息（这些消息属于旧 Leader Epoch，新 Leader 可能没有）
```

## 4.3 LeaderEpochFile 存储

```scala
// leader-epoch-checkpoint 文件内容
0    0      // Epoch 0, Start Offset 0
1    100    // Epoch 1, Start Offset 100
2    200    // Epoch 2, Start Offset 200
3    210    // Epoch 3, Start Offset 210

class LeaderEpochCache {
  // epoch → (start offset, end offset)
  val epochs: TreeMap[Int, EpochEntry]
}
```

# 5. Unclean Leader Election

## 5.1 概念

当 ISR 中所有副本都宕机时，是否允许从 OSR（非同步副本）中选举 Leader。

```
场景:
  ISR = [0, 1]
  Follower 1 宕机 30s+ → ISR = [0]
  Leader 0 也宕机 → ISR 为空

  unclean.leader.election.enable=true (默认 false):
    → 从 OSR (副本 1) 中选举 Leader
    → 可能丢失一些消息（1 的 LEO < 0 的 HW）
    → 但服务可用

  unclean.leader.election.enable=false (推荐):
    → 不选举，等待 ISR 中副本恢复
    → 服务暂时不可用
    → 不会丢消息
```

```properties
unclean.leader.election.enable=false  # 推荐
```

# 6. 分区副本分配策略

## 6.1 默认分配算法

Kafka 为每个 Partition 分配副本时，遵循以下原则：

1. **均匀分布**：Partition 均匀分布到不同 Broker
2. **机架感知**：副本尽量分布在不同 Rack

```
假设: 3 个 Broker, 6 个 Partition, RF=3

Step 1: 随机选 Broke 的起始索引（例如 random = broker2）
Step 2: 分配副本

P0: Leader=2, Replicas=[2, 0, 1]  (分配: broker2, broker0, broker1)
P1: Leader=0, Replicas=[0, 1, 2]
P2: Leader=1, Replicas=[1, 2, 0]
P3: Leader=2, Replicas=[2, 1, 0]
P4: Leader=0, Replicas=[0, 2, 1]
P5: Leader=1, Replicas=[1, 0, 2]

结果: 每个 Broker 上有 2 个 Leader Partition
```

## 6.2 Rack-Aware 分配（Kafka 2.5+）

```properties
# 每个 Broker 配置所属机架
broker.rack=rack-1
```

```
数据中心:
  Rack-1: Broker 0, Broker 1
  Rack-2: Broker 2, Broker 3
  Rack-3: Broker 4, Broker 5

Topic, 3 Partitions, RF=3, Rack-Aware:

P0: Replicas=[0(R1), 2(R2), 4(R3)]  ← 副本在不同机架
P1: Replicas=[2(R2), 4(R3), 1(R1)]
P2: Replicas=[4(R3), 1(R1), 3(R2)]

Rack 故障 (Rack-1 断电):
  P0: Leader → 变为 Broker 2 (Rack-2)，Broker 4 (Rack-3) 也有副本 ← 不丢数据
```

# 7. 副本 Fetch 机制

## 7.1 ReplicaFetcher 线程

每个 Follower Broker 会启动多个 ReplicaFetcher 线程，向 Leader Broker 发起 FetchRequest：

```scala
class ReplicaFetcherThread(
  name: String,
  sourceBroker: BrokerEndPoint, // Leader Broker
  brokerConfig: KafkaConfig,
  replicaMgr: ReplicaManager
) {
  override def doWork(): Unit = {
    // 1. 构建 FetchRequest（包含所有需要从该 Leader 同步的 Partition 列表）
    val fetchRequest = buildFetchRequest(partitionMap)

    // 2. 发送 FetchRequest 到 Leader Broker
    val fetchResponse = fetchFromLeader(fetchRequest)

    // 3. 处理 FetchResponse
    processFetchResponse(fetchResponse)
    //    └── 追加消息到本地 Log
    //    └── 更新 HW
    //    └── 推进 LEO
  }
}
```

## 7.2 复制相关参数

```properties
##############################
# Follower 拉取参数
##############################

# Follower 向 Leader 拉取的副本 ID
# -1 表示普通 Consumer, >0 表示 Follower Broker
# (Follower 拉取优先级可能更高)

# Follower 拉取最小/最大字节数
replica.fetch.min.bytes=1
replica.fetch.max.bytes=1048576       # 1MB

# Follower 拉取请求超时
replica.fetch.wait.max.ms=500

# Follower 拉取时 Leader 的响应最大字节
replica.fetch.response.max.bytes=10485760  # 10MB

# 每个 Fetcher 线程的拉取间隔
replica.socket.receive.buffer.bytes=65536

##############################
# 复制限速（防止复制占用太多带宽）
##############################

# Leader 端复制限速
leader.replication.throttled.rate=-1   # 默认不限制
# Follower 端复制限速
follower.replication.throttled.rate=-1

# 动态配置示例:
kafka-configs.sh --alter --add-config \
    leader.replication.throttled.rate=104857600 \  # 100MB/s
    --entity-type brokers --entity-name 0
```

# 8. 分区 Leader 选择

## 8.1 Preferred Leader Election

每个 Partition 的副本列表的第一个副本被称为 **Preferred Leader**。

```
P0: Replicas=[0, 1, 2]  → Preferred Leader = Broker 0
```

当 Preferred Leader 恢复后，可以将 Leader 迁回：

```bash
# 对所有 Topic 的 Partition 执行 Preferred Leader 选举
kafka-leader-election.sh --bootstrap-server localhost:9092 \
    --election-type preferred --all-topic-partitions

# 对指定 Partition 执行
kafka-leader-election.sh --bootstrap-server localhost:9092 \
    --election-type preferred \
    --topic my-topic --partition 0
```

### 8.1.1 自动 Preferred Leader 选举

```properties
# 自动首选 Leader 选举（默认开启）
auto.leader.rebalance.enable=true

# Leader 不平衡比率（超过此值触发自动选举）
leader.imbalance.per.broker.percentage=10

# 检查间隔
leader.imbalance.check.interval.seconds=300
```

# 9. 常见复制问题

## 9.1 ISR 频繁收缩/扩展（ISR 抖动）

```
症状:
  [2024-01-01 10:00:00] ISR shrink: [0,1,2] → [0,1]
  [2024-01-01 10:00:05] ISR expand: [0,1] → [0,1,2]
  [2024-01-01 10:00:10] ISR shrink: [0,1,2] → [0,1]
  (反复抖动)

原因:
  - Follower Broker 内存不足 → GC 暂停 → 拉取超时
  - 网络抖动
  - 磁盘 I/O 过载导致 Page Cache 命中率低

解决:
  - 增大 replica.lag.time.max.ms (如 60000)
  - 增加 num.replica.fetchers (增加拉取线程)
  - 优化 Broker JVM 参数
```

## 9.2 Under-Replicated Partitions

```
症状: kafka-topics.sh --describe 显示 "Isr: 1,2" (缺少 3)
      持续存在 Under-Replicated Partition

原因:
  - Broker 宕机未恢复
  - 副本同步慢
  - 磁盘故障

排查:
  # 查看 ISR 不完整的 Partition
  kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions

  # 查看原因
  kafka-metadata-quorum.sh --describe --status
```
