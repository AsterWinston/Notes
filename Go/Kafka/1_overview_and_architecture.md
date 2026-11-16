Kafka 常被叫作"消息队列"，但它真正的定位是**分布式提交日志**：消息写进去不会因为被消费而消失，消费者只是沿着日志往前移动一个游标。这个认知差异决定了很多设计——能不能回溯、能不能多个服务各读一遍、磁盘该留多久、分区该怎么规划。本篇从定位讲到 Broker、Topic、Partition、Replica 和 Offset，把 Kafka 的骨架一次搭清楚，为后面写 Go 客户端打好地基。

# 1. Kafka 的定位

## 1.1 从消息队列到事件流平台

Apache Kafka 是一个**分布式事件流平台**，最初由 LinkedIn 开发，2011 年进入 Apache 基金会。它同时承担三件事：

- **发布订阅**：生产者写事件，消费者按自己的节奏读。
- **持久化存储**：事件按保留策略落盘，可重复读取。
- **流式处理**：Kafka Streams、Flink、Spark 直接在流上做计算。

传统消息队列把消息当成"待办任务"，消费完就删除；Kafka 把消息当成"发生的事实"，追加到日志末尾，谁想读就自己维护读到的位置。

| 场景 | 典型链路 |
|------|----------|
| 业务解耦 | 订单服务 → Kafka → 库存、积分、通知 |
| 日志聚合 | 应用 → Kafka → Elasticsearch、ClickHouse |
| 指标与埋点 | SDK → Kafka → Flink → 实时大屏 |
| CDC | MySQL Binlog → Debezium → Kafka → 数仓 |
| 事件溯源 | 所有状态变更写 Kafka，重放恢复状态 |

★ **关键认知**：Kafka 的核心数据结构是**只追加的提交日志（commit log）**。写入永远是 append，读取永远是按 offset 顺序扫描。消费不是"取走并删除"，而是"记住我读到哪了"。理解这一点，后面所有"为什么能回溯""为什么能多组消费""为什么顺序只在分区内"都顺理成章。

⚠ **关键认知**：把 Kafka 当 RabbitMQ 用，最容易犯的错是"消费完就该没了"。Kafka 的消息由**时间或大小保留策略**删除，不由消费行为删除。如果业务真的要求"处理完立即清理"，要么缩短 retention，要么接受"消息仍可被重读"，并在消费端做幂等。

## 1.2 核心概念全景

```text
                    ┌─────────────────────────────────────────┐
                    │              Kafka Cluster              │
                    │                                         │
  Producer ────────▶│  Broker 1        Broker 2        Broker 3│
      │             │  ┌─────────┐    ┌─────────┐    ┌─────────┐│
      │             │  │Topic A  │    │Topic A  │    │Topic A  ││
      │             │  │ P0(L)   │    │ P1(L)   │    │ P2(L)   ││
      │             │  │ P1(F)   │    │ P2(F)   │    │ P0(F)   ││
      │             │  └─────────┘    └─────────┘    └─────────┘│
      │             └───────────────┬─────────────────────────────┘
      │                             │
      │                    ┌────────▼────────┐
      └───────────────────▶│ Consumer Group  │
                           │  C1  C2  C3     │
                           └─────────────────┘
```

### 1.2.1 Broker

**Broker** 就是一台 Kafka 服务进程。一个集群由多个 Broker 组成，每个 Broker 有唯一 ID。客户端只需要配置几个 `bootstrap.servers`，连上任意一个就能拿到整个集群的元数据。

集群里有一个 **Controller**，负责分区 leader 选举、副本状态变更、Topic 创建删除等管理动作。Kafka 3.x 的主流部署方式是 **KRaft**：用 Raft 协议在 Broker 内部选主，元数据存放在 `__cluster_metadata` 日志里，不再依赖 ZooKeeper。ZooKeeper 模式在 3.x 仍可用，但 4.0 已移除。

### 1.2.2 Topic 与 Partition

**Topic** 是逻辑上的消息分类，比如 `orders`、`payments`、`user-events`。Topic 在物理上被切成若干个 **Partition**，每个 Partition 是一个有序、不可变的日志文件序列。

```text
Topic: orders
┌──────────────────────────────────────────────────────┐
│ Partition 0: [0][1][2][3][4][5] →  新消息追加到末尾   │
│ Partition 1: [0][1][2][3]                           │
│ Partition 2: [0][1][2][3][4][5][6][7]               │
└──────────────────────────────────────────────────────┘
```

- **分区内有序**：同一个 Partition 内，消息按写入顺序排列。
- **分区间无序**：不同 Partition 之间没有全局顺序。
- **分区是并行单位**：一个 Topic 的最大消费并行度等于分区数（同一消费组内）。
- **offset 是分区内的编号**：每个 Partition 从 0 开始独立编号。

★ **关键认知**：Kafka 只保证**单分区有序**，不保证 Topic 全局有序。需要"同一订单的事件按顺序处理"，就把订单 ID 作为 key，让同一订单的消息落到同一分区；需要全局有序，只能把 Topic 设成单分区，代价是失去水平扩展能力。

⚠ **关键认知**：分区数不是越多越好。分区越多，Broker 需要维护的文件句柄、副本任务、元数据越大，leader 选举和 rebalance 也越慢。分区数要在"峰值吞吐 / 单分区吞吐"和"运维成本"之间取平衡，第 8 篇会给出估算方法。

### 1.2.3 Replica 与 ISR

每个 Partition 有若干个**副本（Replica）**，副本数就是 replication factor。其中一个是 **leader**，负责处理读写；其余是 **follower**，只向 leader 拉取数据做同步。

- **ISR（In-Sync Replicas）**：与 leader 保持同步的副本集合，包含 leader 自己。
- **HW（High Watermark）**：ISR 中所有副本都已经同步到的位置，消费者只能读到 HW 之前的数据。
- **LEO（Log End Offset）**：每个副本日志末尾的下一个位置。

```text
Leader:   [0][1][2][3][4][5][6][7]   LEO = 8
Follower: [0][1][2][3][4][5]         LEO = 6
HW = 6   ← 消费者最多读到 offset 5
```

当 follower 落后太多或失联，会被踢出 ISR。leader 挂掉时，Controller 从 ISR 中选一个新 leader。如果允许**非 ISR 副本**当选（`unclean.leader.election.enable=true`），可能丢失数据，但能提高可用性。

★ **关键认知**：`acks=all` 只表示"所有 ISR 都写入成功"，并不等于"所有副本都写入成功"。如果 ISR 只剩 leader 一个，`acks=all` 也只写了一份。真正防丢要同时满足：`acks=all`、`min.insync.replicas>=2`、replication factor>=3。

### 1.2.4 Offset

Offset 是消息在分区内的位置编号。需要区分三个概念：

| 名称 | 含义 |
|------|------|
| Log End Offset（LEO） | 分区当前末尾的下一个位置 |
| High Watermark（HW） | 消费者可见的最大位置 |
| Consumer Offset | 某个消费组在某个分区上"下一条要读"的位置 |

消费组提交的 offset 保存在内部 Topic `__consumer_offsets` 里。它和消息本身的保留策略相互独立：消息可能因为过期被删了，但 offset 记录还在；反过来，offset 记录过期了，消费组就会退回 `auto.offset.reset` 策略。

⚠ **关键认知**：提交 offset 表示"这条之前（不含这条）都处理完了"。如果处理完消息、提交前崩溃，重启后会从旧 offset 重读，产生重复消费。这就是 Kafka 默认的**至少一次**语义，第 5、6 篇会展开。

# 2. 集群架构与读写路径

## 2.1 整体架构

```text
┌────────────┐        ┌──────────────────────────────────┐
│  Producer  │───────▶│           Kafka Cluster          │
└────────────┘        │  ┌────────┐  ┌────────┐  ┌─────┐ │
                      │  │Broker 1│  │Broker 2│  │ ... │ │
┌────────────┐        │  └───┬────┘  └───┬────┘  └─────┘ │
│  Consumer  │◀───────│      │           │               │
└────────────┘        │  ┌───▼───────────▼────┐          │
                      │  │  Controller（KRaft）│          │
┌────────────┐        │  └────────────────────┘          │
│ Admin/CLI  │───────▶│                                  │
└────────────┘        └──────────────────────────────────┘
```

客户端不直接和所有 Broker 打交道。第一次连接 bootstrap 节点拿到元数据后，生产者只向目标分区的 leader 写，消费者只从目标分区的 leader 读。follower 的职责是复制，不承担客户端读请求（除非配置了 follower fetching）。

## 2.2 生产写入路径

```text
Producer
   │ 1. 拉取元数据，找到目标分区 leader
   ▼
Partitioner ── key 哈希 / 轮询 / 粘性 ──▶ 选定 Partition
   │ 2. 写入客户端缓冲区，按分区攒批
   ▼
Batch（可压缩）
   │ 3. 发送 ProduceRequest 给 leader
   ▼
Leader Broker
   │ 4. 追加到本地日志
   │ 5. follower 拉取并写入
   ▼
  6. 按 acks 决定何时返回成功
```

`acks` 有三个取值：

| acks | 行为 | 可靠性 | 延迟 |
|------|------|--------|------|
| `0` | 发出即成功，不等 Broker | 最低，可能静默丢失 | 最低 |
| `1` | leader 写本地日志即成功 | 中，leader 挂可能丢 | 低 |
| `all`（`-1`） | 所有 ISR 写入才成功 | 最高 | 较高 |

## 2.3 消费读取路径

消费者以 **Pull** 方式主动拉取，而不是 Broker 推送。每次 `FetchRequest` 带上"我要从哪个 offset 开始、最多给我多少字节"。

```text
Consumer                          Leader Broker
   │ ── Fetch(partition, offset) ──────▶ │
   │                                    │ 从 offset 顺序读日志段
   │ ◀── RecordBatch（可能含多条） ────── │
   │                                    │
   │ 处理消息                            │
   │ ── OffsetCommit(group, offset+1) ─▶ │ 写入 __consumer_offsets
```

Pull 模型的好处是消费者能按自己的处理能力控制速率，Broker 也不用为每个消费者维护推送状态。代价是"没有消息时也要轮询"，Kafka 用 `fetch.max.wait.ms` 和 `fetch.min.bytes` 做了长轮询优化。

★ **关键认知**：Kafka 的 Broker 是"哑"的——它只负责存日志和返回数据，不跟踪谁读到了哪。消费进度由客户端提交到 `__consumer_offsets`。这套"瘦 Broker、胖客户端"的设计让 Broker 很容易水平扩展，也把重试、幂等、顺序这些责任交给了使用方。

## 2.4 副本同步与 HW

follower 持续向 leader 发送 `FetchRequest`，leader 返回数据后 follower 追加到本地日志。leader 维护每个 follower 的 LEO，取最小值作为 HW。

```text
时间 →
Leader   : 0 1 2 3 4 5 6 7
Follower : 0 1 2 3 4 5
HW       : 0 1 2 3 4 5 6   ← 取 ISR 最小 LEO
```

消费者只能读到 HW 之前的数据，这样即使发生 leader 切换，新 leader 也不会把"尚未同步到多数副本"的消息暴露给消费者，避免读后消失。

⚠ **关键认知**：Kafka 3.x 默认使用 **leader epoch** 处理副本截断，避免旧版本"靠 HW 截断"导致的日志不一致。运维上如果看到 `OfflinePartitionsCount > 0` 或 ISR 频繁收缩，优先查 Broker 磁盘、网络和 GC，而不是盲目调大超时。

# 3. 为什么 Kafka 快

## 3.1 顺序写与页缓存

机械硬盘随机写只有几百 IOPS，顺序写却能达到数百 MB/s。Kafka 只追加日志，天然是顺序写；读取也尽量顺序扫描，配合操作系统页缓存（page cache）命中热数据。它不在 JVM 堆里缓存消息，而是依赖页缓存，所以堆内存小、GC 压力低，缓存还能被多个进程共享。

★ **关键认知**：Kafka 的高吞吐不是靠"内存数据库"，而是靠**顺序 I/O + 页缓存 + 批量**。这也解释了为什么 Kafka 不适合"大量小消息、随机读"的场景——批量摊不开，顺序优势就没了。

## 3.2 零拷贝

传统读取要经过四次拷贝：

```text
磁盘 → 内核缓冲区 → 用户缓冲区 → Socket 缓冲区 → 网卡
```

Kafka 使用 `sendfile`，数据从页缓存直接送到网卡：

```text
磁盘 → 页缓存 → Socket 缓冲区 → 网卡
```

省掉两次 CPU 拷贝和两次上下文切换。消费者读历史数据时收益最明显。

## 3.3 批量与压缩

生产者把同一分区的消息攒成 `RecordBatch` 再发送，Broker 按批存储，消费者按批拉取。批越大，摊到每条消息的协议开销和系统调用越少。

压缩也是按批做的。常见算法有 gzip、snappy、lz4、zstd：gzip 压缩比高但 CPU 贵；lz4 延迟最低；zstd 在压缩比和 CPU 之间比较均衡，新项目优先考虑。具体选型在第 3 篇展开。

## 3.4 分区并行

一个 Topic 的多个分区分布在多个 Broker 上，生产者和消费者可以同时读写不同分区。吞吐不够就加分区、加消费者，这是 Kafka 水平扩展的基本手段。

⚠ **关键认知**：加分区能提高并行度，但会破坏"同 key 落同一分区"的稳定性——分区数变化后，`hash(key) % 分区数` 的结果会变，历史 key 可能落到新分区，顺序性受影响。分区扩容要选在低峰期，并接受短期内的重分布。

# 4. 与 RabbitMQ 的对比与选型

## 4.1 模型差异

```text
RabbitMQ：
Producer → Exchange ──binding──▶ Queue → Consumer
                                     │
                                  消费后删除

Kafka：
Producer → Topic/Partition（只追加日志）
                ├── Consumer Group A（各自维护 offset）
                └── Consumer Group B（独立读同一份数据）
```

RabbitMQ 的核心是**路由**：exchange 根据 routing key 把消息投递到一个或多个 queue，消费后消息从 queue 删除。Kafka 的核心是**日志**：消息按分区追加，谁读谁记 offset，读不删除。

## 4.2 对比表

| 维度 | Kafka | RabbitMQ |
|------|-------|----------|
| 设计定位 | 分布式事件流平台 | 通用消息代理 |
| 吞吐量 | 十万到百万级/秒 | 万级/秒 |
| 消费模型 | Pull | Push（也有 Pull） |
| 消息删除 | 按保留策略 | 消费确认后删除 |
| 回溯消费 | 支持按 offset 或时间 | 不支持 |
| 多订阅者 | 多个消费组独立消费 | 需要多个 queue 绑定 |
| 顺序性 | 分区内有序 | 单 queue 有序 |
| 延迟 | 毫秒级 | 微秒到毫秒级 |
| 路由能力 | 较弱，靠 key 和分区 | 强，exchange 类型丰富 |
| 延迟消息 | 无原生支持 | 插件支持 |
| 事务 | 支持 EOS | 支持，但语义不同 |
| 运维复杂度 | 较高，分区/副本/磁盘要规划 | 相对简单 |

## 4.3 选型建议

- **要吞吐、要回溯、要流处理、要多团队重复消费同一份事件** → Kafka。
- **要复杂路由、优先级队列、延迟消息、单条确认** → RabbitMQ。
- **两者都有** → 常见做法是 Kafka 做事件总线，RabbitMQ 做任务分发，不要让一个组件承担所有语义。

★ **关键认知**：选型的核心不是"谁更快"，而是"消息的生命周期模型是否匹配业务"。Kafka 假设消息会被多个人、在不同时间读；RabbitMQ 假设消息是一条待处理的任务，处理完就该消失。模型选错，后面会用大量补丁去模拟对方的能力。

# 5. 常见误解与坑

| 误解 | 实际情况 |
|------|----------|
| "Kafka 消费完消息就删了" | 由 retention 删除，消费不影响 |
| "Topic 内全局有序" | 只有分区内有序 |
| "副本数等于写入成功数" | `acks=all` 只等 ISR，ISR 可能只剩一个 |
| "offset 是全局编号" | 每个分区独立编号 |
| "消费者越多越快" | 同组内超过分区数的消费者会空闲 |
| "分区越多吞吐越高" | 超过一定数量后运维成本和延迟上升 |
| "Kafka 只能做消息队列" | 还能做存储、流处理、CDC 总线 |

⚠ **关键认知**：`auto.create.topics.enable=true` 在生产环境通常要关掉。客户端拼错 Topic 名时，Broker 会自动建一个默认分区数的 Topic，看起来"没报错"，实际上消息写进了一个没人消费的新 Topic。

# 6. 动手练习

1. 用 Docker 启动一个单节点 KRaft Kafka，创建 3 分区、1 副本的 `demo` Topic，用 `kafka-console-producer` 写入 10 条带 key 的消息，再用 `kafka-console-consumer --property print.key=true` 观察分区分布。
2. 把副本数改成 3（需要三个 Broker），停掉一个 Broker，观察 `kafka-topics.sh --describe` 里 ISR 的变化，以及生产是否仍能成功。
3. 用同一个消费组启动两个消费者，再启动第三个，观察哪个消费者分不到分区；然后用不同 `group.id` 启动，验证两组能各自读到全量消息。

# 7. 小结

| 概念 | 要点 |
|------|------|
| Kafka 定位 | 分布式提交日志，不是消费即删的队列 |
| Broker | 服务进程，Controller 负责管理 |
| Topic/Partition | 逻辑分类与物理分片，分区内有序 |
| Replica/ISR | leader 读写，follower 同步，ISR 决定可靠性 |
| Offset | 分区内位置，消费组提交到 `__consumer_offsets` |
| HW | 消费者可见边界，取 ISR 最小 LEO |
| 高性能 | 顺序写、页缓存、零拷贝、批量、分区并行 |
| 对比 RabbitMQ | 日志模型 vs 路由模型，按消息生命周期选型 |

下一篇进入 Go 客户端实战，从安装 `segmentio/kafka-go` 开始，写出第一个可运行的生产者和消费者，并把连接、配置、关闭这几件小事做对。
