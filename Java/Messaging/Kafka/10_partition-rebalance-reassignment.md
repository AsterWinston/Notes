# 1 分区重平衡概述

## 1.1 什么时候需要分区重平衡

分区重平衡（Partition Reassignment）是指将 Partition 的副本在不同 Broker 之间迁移的过程。

触发场景：

1. **新增 Broker**：集群扩容，将已有 Partition 的副本迁移过去
2. **下线 Broker**：计划性下线 Broker 前，将其上的 Leader 和副本迁出
3. **负载均衡**：某 Broker 上 Leader 数过多，导致负载不均
4. **磁盘容量不均**：某 Broker 磁盘快满，需迁移部分数据
5. **机架故障迁移**：某个 Rack 需要维护

## 1.2 重平衡的范围

```
┌─────────────────────────────────────────────────────────────┐
│              Partition Reassignment 操作粒度                 │
│                                                             │
│  Topic 级别:                                                │
│    → 迁移指定 Topic 的所有 Partition                          │
│                                                             │
│  Partition 级别:                                            │
│    → 迁移指定的个别 Partition                                 │
│                                                             │
│  Broker 级别:                                               │
│    → 迁移某 Broker 上的所有 Partition                         │
│                                                             │
│  集群级别:                                                    │
│    → 对整个集群重新分配（Cruise Control / kafka-reassign）     │
└─────────────────────────────────────────────────────────────┘
```

# 2 kafka-reassign-partitions 工具

## 2.1 生成迁移计划

### 2.1.1 迁移指定 Topic

```bash
# 1. 创建 topics-to-move.json
cat > topics-to-move.json << EOF
{
  "topics": [
    {"topic": "my-topic"}
  ],
  "version": 1
}
EOF

# 2. 生成迁移计划（不执行）
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --topics-to-move-json-file topics-to-move.json \
    --broker-list "0,1,2,3" \
    --generate

# 输出:
# Current partition replica assignment
# {"version":1,"partitions":[...]}   ← 当前分配
#
# Proposed partition reassignment configuration
# {"version":1,"partitions":[...]}   ← 建议分配
```

### 2.1.2 手动制定迁移计划

```json
{
  "version": 1,
  "partitions": [
    {
      "topic": "my-topic",
      "partition": 0,
      "replicas": [2, 3, 4]     // 目标: 从 [0, 1, 2] 迁移到 [2, 3, 4]
    },
    {
      "topic": "my-topic",
      "partition": 1,
      "replicas": [3, 4, 0]
    }
  ]
}
```

## 2.2 执行迁移

```bash
# 执行迁移计划
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --reassignment-json-file reassignment.json \
    --execute

# 查看迁移进度
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --reassignment-json-file reassignment.json \
    --verify

# 输出:
# Status of partition reassignment:
# Reassignment of partition my-topic-0 is still in progress
# Reassignment of partition my-topic-1 is complete

# 取消正在进行的迁移（Kafka 3.7+）
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --cancel --reassignment-json-file reassignment.json
```

# 3 副本迁移底层流程

## 3.1 迁移三阶段

```
Partition P0 副本从 [0,1,2] 迁移到 [2,3,4]:

阶段 1: 添加新副本
  Replicas: [0, 1, 2] → [0, 1, 2, 3, 4]
  ├── Controller 向 Broker 3 发出 LeaderAndIsr: P0 创建为 Follower
  ├── Controller 向 Broker 4 发出 LeaderAndIsr: P0 创建为 Follower
  ├── Broker 3,4 的 ReplicaFetcher 开始从 Leader(Broker 0) 同步数据
  └── ISR 可能暂时缩小（新副本同步中）

阶段 2: 等待新副本追上
  ├── 等待 Broker 3,4 的 LEO 追上 Leader 的 LEO
  ├── ISR 扩展: [0,1,2] → [0,1,2,3,4]
  └── 新副本完成同步

阶段 3: 移除旧副本
  Replicas: [0, 1, 2, 3, 4] → [2, 3, 4]
  ├── Controller 向 Broker 0,1 发出 StopReplica: 删除 P0 副本
  ├── Broker 0,1 删除对应的 Log Segment 文件
  └── 迁移完成
```

## 3.2 迁移时的数据一致性

```
迁移过程中如何保证消息不丢？

场景: P0 从 [0,1,2] → [2,3,4]，Leader 还是 Broker 0

① 新副本(Broker 3,4) 从 Leader 拉取全量数据
   FETCH offset=0 → fetch 所有历史数据
   复制过程中 Leader 继续接收新消息 → Follower 持续追赶

② 当新副本追上 Leader LEO
   ISR 包含新副本 → Leader 写入的每条消息都会同步到新副本

③ 如果迁移过程中 Leader 变更
   新 Leader (如 Broker 2) 继续接收消息
   旧 Leader (Broker 0) 的未同步数据 → 通过 LeaderEpoch 截断
```

# 4 迁移限速（Throttle）

## 4.1 为什么需要限速

大量数据迁移会占用大量网络带宽和磁盘 I/O，影响正常业务读写。Kafka 提供迁移限速机制。

```bash
# 先创建迁移计划（带限速）
cat > reassignment.json << EOF
{
  "version": 1,
  "partitions": [
    {"topic": "topic-a", "partition": 0, "replicas": [2, 3]}
  ]
}
EOF

# 执行迁移（不限速默认无限制）
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --reassignment-json-file reassignment.json \
    --throttle 52428800 \
    --execute
# throttle=52428800 bytes/s = 50MB/s
```

## 4.2 限速配置层级

```properties
# 1. Broker 级别动态配置（对通过该 Broker 的所有迁移生效）
# Leader 端复制带宽限制
leader.replication.throttled.rate=104857600  # 100MB/s
# Follower 端复制带宽限制
follower.replication.throttled.rate=104857600

# 2. kafka-reassign-partitions --throttle（仅对该迁移任务生效）

# 3. 动态修改限速
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
    --alter --add-config 'leader.replication.throttled.rate=104857600' \
    --entity-type brokers --entity-name 0
```

# 5 分区再平衡算法

## 5.1 均衡目标

理想的均衡状态：
- 每个 Broker 上的 Partition 数量尽量均匀
- 每个 Broker 上的 Leader Partition 数量尽量均匀
- 尽量满足 Rack-Aware 分布

## 5.2 分配算法概述

```scala
// kafka.admin.AdminUtils.assignReplicasToBrokers
// 默认分配算法（Round-Robin + Rack-Aware）

def assignReplicasToBrokers(
  brokerMetadatas: Seq[BrokerMetadata],
  nPartitions: Int,
  replicationFactor: Int,
  fixedStartIndex: Int = -1,
  startPartitionId: Int = -1
): Map[Int, Seq[Int]] = {

  // 1. 随机确定起始 Broker 索引
  val startIndex = if (fixedStartIndex >= 0) fixedStartIndex
                   else Random.nextInt(brokerMetadatas.size)

  // 2. 为每个 Partition 分配副本
  val ret = mutable.Map[Int, List[Int]]()
  for (i <- 0 until nPartitions) {
    val shift = i + startIndex
    val replicaList = for (j <- 0 until replicationFactor) yield {
      // 分配第 j 个副本（跳转到不同 Broker）
      brokerMetadatas((shift + j) % brokerMetadatas.size).id
    }
    ret.put(i, replicaList.toList)
  }
  ret.toMap
}
```

## 5.3 Cruise Control 自动负载均衡

Cruise Control 是 LinkedIn 开源的 Kafka 集群自动运维系统：

```
Cruise Control 通过分析以下指标进行自动 Rebalance:
  - CPU 利用率
  - 网络入/出吞吐
  - 磁盘使用率
  - Leader Partition 分布
  - Replica 分布
  - Topic Partition 大小

典型操作:
  ① add_broker: 新增 Broker 后自动迁移
  ② remove_broker: 下线 Broker 前自动迁出
  ③ rebalance: 根据负载指标自动再平衡
  ④ fix_offline_replicas: 修复离线副本
```

# 6 Consumer Group Rebalance

## 6.1 Consumer Group Rebalance 与 Partition Reassignment 的区别

| 维度 | Consumer Rebalance | Partition Reassignment |
|------|-------------------|----------------------|
| 对象 | Consumer Group 内 Consumer 与 Partition 的映射 | Partition 副本在 Broker 间的分布 |
| 触发者 | GroupCoordinator | Controller |
| 影响 | Consumer 消费暂停 | Broker 间数据复制 |
| 数据迁移 | 无（仅 Offset 变化） | 有（Segment 文件复制） |
| 数据量 | 极小 | 很大 |

## 6.2 Rebalance 抑制机制

```java
// 避免频繁触发 Rebalance
props.put("group.initial.rebalance.delay.ms", 3000);
// 首次 Rebalance 延迟，防止 Consumer 启动顺序引起的多次 Rebalance

// Kafka 2.3+:
// 静态 Group Membership（Static Membership）
props.put("group.instance.id", "consumer-instance-1");
// 临时离开（如滚动重启）→ 不会触发 Rebalance
// session.timeout.ms 内回来 → 保留之前的分配
```

### 6.2.1 Static Membership 工作原理

```
场景: Consumer 滚动重启

传统模式（无 Static Membership）:
  Consumer 关闭 → Coordinator 感知 → Rebalance ①
  Consumer 重启 → JoinGroup → Rebalance ②
  (总共 2 次 Rebalance, 所有 Consumer 暂停 2 次)

Static Membership 模式:
  Consumer 关闭 → Coordinator 感知 → 等待 session.timeout.ms
  Consumer 重启（携带 group.instance.id）→ Coordinator 识别 →
  分配不变，不触发 Rebalance
  (0 次 Rebalance!)
```

# 7 常见操作与排查

## 7.1 查看当前 Partition 分布

```bash
# 查看所有 Topic 的 Partition 分布
kafka-topics.sh --bootstrap-server localhost:9092 --describe

# 查看特定 Topic 的分配
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --topic my-topic

# 输出:
# Topic: my-topic  Partition: 0  Leader: 0
#   Replicas: 0,1,2  Isr: 0,1,2

# 统计每个 Broker 上的 Leader 数量
kafka-topics.sh --bootstrap-server localhost:9092 --describe \
    | grep "Leader:" | awk '{print $NF}' | sort | uniq -c
```

## 7.2 手动 Preferred Leader Election

```bash
# 全集群执行（将 Leader 迁回 Preferred Leader）
kafka-leader-election.sh --bootstrap-server localhost:9092 \
    --election-type preferred --all-topic-partitions

# 指定 Partition
kafka-leader-election.sh --bootstrap-server localhost:9092 \
    --election-type preferred \
    --topic my-topic --partition 0
```

## 7.3 检查 Under-Replicated Partitions

```bash
# 检查哪些 Partition 副本不足
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --under-replicated-partitions

# 检查 ISR 状态
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --under-min-isr-partitions

# 监控指标
# kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
# 值应该持续为 0
```

## 7.4 迁移失败排查

```
常见迁移失败原因:

① 磁盘空间不足
  检查目标 Broker 磁盘: df -h /data/kafka-logs

② 复制速度太慢
  增大限速或检查网络

③ 目标 Broker 故障
  kafka-broker-api-versions.sh --bootstrap-server <target-broker>

④ Partition 分配冲突
  确保同一个 Partition 的多个副本不在同一个 Broker

⑤ ZooKeeper 超时
  增大 zookeeper.connection.timeout.ms
```

# 8 最佳实践

## 8.1 迁移操作建议

1. **低峰期执行**：迁移会产生额外的网络和磁盘开销
2. **设置合理限速**：一般设为链路带宽的 20~30%
3. **分批进行**：不要一次性迁移所有 Partition，分批验证
4. **监控指标**：
   - `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions`
   - `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Fetch`
5. **迁移前备份**：记录当前 Partition 分布（用于回滚）
6. **逐步下线**：下线 Broker 前先执行迁移，确认迁移完成后再停服务

## 8.2 Partition 数调整原则

```properties
# 增加分区（可以动态执行）
kafka-topics.sh --bootstrap-server localhost:9092 \
    --alter --topic my-topic --partitions 20

# 减少分区（不支持直接减少！）
# 替代方案:
#   ① 创建新 Topic（更少分区数）
#   ② 将数据从旧 Topic 迁移到新 Topic
#   ③ 切换 Consumer 到新 Topic
#   ④ 删除旧 Topic
```

注意：Partition 数只能增加不能减少。设计初期应合理评估，避免后期无法缩减。
