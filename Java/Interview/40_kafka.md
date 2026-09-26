# 1. 知识点
- **核心架构**：Kafka 集群由多个 Broker 组成，旧版依赖 ZooKeeper 管理元数据（选举 Controller、维护集群成员信息），新版（3.7.0+）使用 KRaft 自管理。Controller 负责 Partition Leader 选举和集群元数据管理。Producer 生产消息，Consumer 消费消息，Consumer Group 实现负载消费。
- **Topic（主题）**：消息的逻辑分类，可跨多个 Broker 存储，一个 Topic 包含多个 Partition。
- **Partition（分区）**：Topic 的物理存储单元，分区内消息严格有序，分区是 Kafka 并行读写的基本单位。每个分区是一个有序的、不可变的消息序列。
- **Log Segment（日志分段）**：每个 Partition 由多个 Segment 文件组成（.log 数据文件、.index 时间索引、.timeindex 时间索引），有利于文件的清理和快速定位。
- **Replica（副本）**：每个 Partition 有多个副本，包含一个 Leader 和多个 Follower。只有 Leader 处理读写请求，Follower 从 Leader 同步数据。
- **ISR（In-Sync Replicas）**：与 Leader 保持同步的副本集合。Follower 延迟超过 `replica.lag.time.max.ms`（默认 30s）会被移出 ISR，追上后重新加入。
- **OSR（Out-of-Sync Replicas）**：已落后 Leader 太多、被移出 ISR 的副本集合。
- **AR（Assigned Replicas）**：分区的全部副本集合（AR = ISR + OSR）。
- **HW（High Watermark）**：高水位，消费者最多能消费到的消息位置（ISR 中最小的 LEO），确保已同步的消息才对消费者可见。
- **LEO（Log End Offset）**：日志末端偏移量，即下一条即将写入消息的 Offset。
- **acks 机制**：`acks=0`（不等待确认，最快但可能丢消息）、`acks=1`（Leader 写入即返回，不保证副本同步）、`acks=-1/all`（等待 ISR 中所有副本同步，最强保证）。
- **幂等性**：`enable.idempotence=true`，Producer 自动为每条消息分配 Producer ID 和 Sequence Number，Broker 去重保证同一 Partition 内 Exactly Once。
- **事务**：Kafka 支持跨分区原子写入，通过 `initTransactions()` / `beginTransaction()` / `commitTransaction()` 实现。
- **Consumer Group**：同一 Group 内消费者竞争消费 Partition，一个 Partition 只能被组内一个消费者消费（即组内消费者数量不应超过 Partition 数量，否则会有消费者空闲）。
- **Offset 提交**：自动提交（`enable.auto.commit=true`，默认 5s 一次）和手动提交（同步 `commitSync()` 阻塞、异步 `commitAsync()` 非阻塞）。Offset 存储在 `__consumer_offsets` 内部 Topic 中。
- **分区分配策略**：RangeAssignor（按单个 Topic 的 Partition 平均分配给消费者）、RoundRobinAssignor（跨所有 Topic 轮询分配）、StickyAssignor（粘性分配，尽量维持原有分配关系）、CooperativeStickyAssignor（渐进式 Rebalance，减少暂停时间）。
- **Rebalance**：触发条件包括：消费者加入/离开 Group、Topic Partition 增加、消费者超时（`session.timeout.ms` 默认 45s 无心跳）。Rebalance 期间所有消费者暂停消费。
- **Commit Log 存储**：Kafka 采用顺序追加写入日志分段文件（.log + .index + .timeindex），利用 PageCache 缓存，通过 `sendfile` 系统调用实现零拷贝。
- **稀疏索引**：Kafka 的 `.index` 文件使用稀疏索引，每写入一定量数据才记录一个索引条目。查找消息时通过二分查找定位到最近的索引位置，再顺序扫描找到目标 Offset。
- **顺序写 vs 随机写**：磁盘顺序写的性能可与内存随机写媲美（因为减少了磁头寻道时间），Kafka 利用顺序写实现高吞吐。
- **高吞吐原因**：顺序写磁盘、批量发送（`batch.size` / `linger.ms`）、消息压缩（gzip/snappy/lz4/zstd）、PageCache、零拷贝（sendfile）。
- **刷盘策略**：Kafka 依赖操作系统周期性刷盘（`log.flush.interval.ms`、`log.flush.interval.messages`），不强制每次写入刷盘，利用 PageCache 实现高性能。
- **副本同步**：Follower 主动从 Leader 拉取数据（Fetch Request），同步进度通过 HW 和 LEO 跟踪。Follower 延迟过大被移入 OSR，赶上后重新加入 ISR。
- **Controller 选举**：旧版依赖 ZooKeeper 的 `/controller` 临时节点争抢注册实现选举，Controller 监听 ZK 中 Broker 变化并重新分配 Partition Leader。新版 KRaft 使用 Raft 协议选举 Controller。
- **Leader 选举**：Partition Leader 失效后，Controller 从 ISR 中选择新的 Leader。若 ISR 为空，`unclean.leader.election.enable` 为 true 时可从 OSR 选 Leader（保证可用性但可能丢失数据），为 false 则等待 ISR 恢复（保证一致性）。
- **Exactly Once 语义**：幂等性（Producer 去重）+ 事务（跨分区原子写入）+ 消费者隔离级别 `read_committed`（只读取已提交的事务消息），实现端到端精确一次。
- **消息有序性**：同一 Partition 内按写入顺序严格有序，相同 Key 的消息哈希到同一 Partition 保证局部有序。
- **集群扩缩**：新增 Broker 后，通过 `kafka-reassign-partitions.sh` 生成迁移计划 JSON，执行 `--execute` 将 Partition 迁移到新 Broker，完成后 `--verify` 确认。
- **Consumer Lag 监控**：通过 `kafka-consumer-groups.sh --describe` 查看消费延迟（LAG），使用 JMX 指标暴露进行监控告警。
- **适用场景**：日志收集（ELK/Kafka+ELK）、消息系统（解耦/削峰填谷）、流处理（Kafka Streams/Flink）、事件溯源、大数据管道（Hadoop/Spark 数据源）。
- **KRaft（Kafka Raft）**：Kafka 3.5+ 移除 ZooKeeper 依赖，使用 Raft 协议自管理元数据，Controller 同时管理元数据日志，简化部署和运维。
- **日志清理策略**：`log.cleanup.policy` 支持 delete（按时间或大小删除旧 Segment）和 compact（保留每个 Key 最新值，用于变更日志场景）。
- **消息压缩**：支持 gzip、snappy、lz4、zstd 压缩算法，Producer 端压缩后发送，Broker 原样存储不解压，Consumer 端解压。
- **Kafka Connect**：用于连接外部系统的框架，Source Connector 从外部系统导入数据到 Kafka，Sink Connector 从 Kafka 导出数据到外部系统。
- **Kafka Streams**：轻量级流处理库，提供 DSL API 进行流式聚合、过滤、Join 等操作，无需额外集群。

# 2. 面试题

## 2.1 Kafka 为什么这么快（高吞吐）？
1. **顺序写磁盘**：Kafka 采用日志追加方式写入，避免了磁盘随机寻道的开销，顺序写性能极高。
2. **PageCache**：充分利用操作系统的 PageCache，将数据缓存在内存中，读写优先命中内存。
3. **零拷贝（sendfile）**：Consumer 拉取数据时使用 Linux 的 `sendfile` 系统调用，数据直接从 PageCache 传输到网卡，不走用户态，减少拷贝次数和上下文切换。
4. **批量处理**：Producer 端批量发送（`batch.size` / `linger.ms`），Consumer 端批量拉取，减少网络往返。
5. **消息压缩**：支持 gzip/snappy/lz4/zstd 压缩，减少网络带宽和磁盘占用。
6. **分区并行**：Topic 多 Partition 实现并行读写，水平扩展能力强。

## 2.2 Kafka 的 ISR 机制是什么？
ISR（In-Sync Replicas）是与 Leader 保持同步的副本集合。Follower 从 Leader 拉取数据，若在 `replica.lag.time.max.ms`（默认 30s）内完成同步，则保持在 ISR 中。若延迟超过该时间，则被移入 OSR（Out-of-Sync）。只有当消息被 ISR 中所有副本同步后，Leader 的 HW 才会推进。`acks=-1` 时，消息需 ISR 内所有 Follower 确认后才返回成功。ISR 机制在可用性和一致性之间取得平衡。

## 2.3 Kafka 的消息丢失和重复消费问题如何处理？
**消息丢失**：
- Producer 端：`acks=-1` + `retries` + `enable.idempotence=true`。
- Broker 端：`replication.factor >= 3`，`min.insync.replicas >= 2`。
- Consumer 端：关闭自动提交 Offset，手动提交（处理完消息后再提交）。

**重复消费**：
- 开启幂等性（`enable.idempotence=true`），Broker 基于 Producer ID + Sequence Number 去重。
- 消费者端实现业务幂等（数据库唯一索引、Redis 去重等）。
- 使用事务保证 Exactly Once。

## 2.4 Kafka 的 Rebalance 是什么？触发条件和流程是怎样的？
Rebalance 是 Consumer Group 内重新分配 Partition 的过程。触发条件：
1. Consumer Group 内有消费者加入或离开。
2. Topic 的 Partition 数增加。
3. 消费者长时间未发送心跳（超过 `session.timeout.ms`）。

流程：
1. Group Coordinator 检测到触发条件，发起 JoinGroup 请求。
2. 所有消费者重新发送 JoinGroup 请求，Coordinator 选举出一个 Group Leader。
3. Group Leader 根据分配策略（Range/RoundRobin/Sticky）生成分配方案。
4. Coordinator 将分配方案同步给所有消费者（SyncGroup）。

Rebalance 期间消费者暂停消费，因此要尽量避免频繁 Rebalance。

## 2.5 Kafka 的事务是如何实现的？
Kafka 事务保证跨分区写入的原子性和 Exactly Once 语义。
1. Producer 调用 `initTransactions()` 向事务协调器注册，获得 transactional.id。
2. 调用 `beginTransaction()` 开始事务。
3. 发送消息时，消息被标记为事务中。
4. 调用 `commitTransaction()`：Producer 向事务协调器发送提交请求，协调器将事务状态写入 `__transaction_state` 主题，并标记消息为已提交。
5. Consumer 设置为 `isolation.level=read_committed`，只消费已提交的事务消息，过滤掉未提交或已中止的事务消息。

## 2.6 Kafka 的 Controller 是做什么的？如何选举？
Controller 是 Kafka 集群中的管理节点，负责：
1. Partition Leader 选举。
2. 监听 Broker 上线/下线，触发 ISR 变更和 Leader 重选。
3. 向所有 Broker 分发元数据信息。
4. 分区副本的分配和再分配。

选举机制：
- 旧版（ZooKeeper）：所有 Broker 竞争创建 `/controller` 临时节点，先创建成功的成为 Controller。Controller 宕机后临时节点被删除，其他 Broker 重新竞争。
- 新版（KRaft）：使用 Raft 协议选举 Quorum Controller，Leader Controller 负责元数据管理。

## 2.7 Kafka 的 HW 和 LEO 是什么？它们的关系如何？
- **LEO（Log End Offset）**：每个副本日志中下一条待写入消息的 Offset，即当前日志末尾位置。
- **HW（High Watermark）**：消费者可见消息的最高 Offset，取 ISR 中所有副本 LEO 的最小值。
- **关系**：Leader 维护 ISR 中所有 Follower 的 LEO 值，HW = min(Leader LEO, Follower1 LEO, Follower2 LEO, ...)。只有 Offset < HW 的消息才对 Consumer 可见，确保只有已同步到 ISR 的消息才能被消费。Follower 同步数据时，Leader 返回当前 HW，Follower 也更新自己的 HW。

## 2.8 Kafka 的消息有序性如何保证？
Kafka 在 Partition 级别保证消息有序性：同一 Partition 内的消息按写入顺序严格有序（FIFO），但不同 Partition 之间没有顺序保证。生产者的写入顺序通过 `max.in.flight.requests.per.connection=1` 配合 `enable.idempotence=true` 来保证（Kafka 3.0+ `enable.idempotence` 默认为 true，`max.in.flight.requests` 可设置为 5）。

## 2.9 Kafka 和 RocketMQ 的对比？
| 维度 | Kafka | RocketMQ |
|------|-------|----------|
| 架构 | Broker + ZK/KRaft | Broker + NameServer |
| 存储模型 | Partition + Segment 文件 | CommitLog + ConsumeQueue |
| 消息过滤 | 无（客户端过滤） | 原生 Tag/SQL92 过滤 |
| 延时消息 | 无原生支持 | 原生 18 级延时消息 |
| 事务消息 | 支持（幂等+事务） | 原生事务消息（半消息） |
| 消息优先级 | 不支持 | 不支持（社区有讨论） |
| 消息堆积 | 磁盘容量决定 | 同样，但 RocketMQ 优化更好 |
| 流处理 | Kafka Streams、KSQL | RocketMQ Streams（较弱） |
| 适用场景 | 日志/流/大数据管道 | 业务消息、电商、金融 |
| 语言 | Scala/Java | Java |
| 生态 | 国际生态强、大数据集成 | 国内生态强（阿里系） |
| 部署依赖 | ZooKeeper（旧）/ KRaft（新） | NameServer（轻量） |

## 2.10 Kafka 如何监控 Consumer Lag？
1. **命令行**：`kafka-consumer-groups.sh --bootstrap-server <broker> --group <group> --describe`，查看 CURRENT-OFFSET 与 LOG-END-OFFSET 的差值即 LAG。
2. **JMX 指标**：Kafka Broker 和 Consumer 暴露 JMX MBean，通过 Prometheus + Grafana 采集和可视化。
3. **Burrow**：LinkedIn 开源的 Kafka Consumer Lag 检查工具，支持多集群和告警。
4. **Kafka Manager / Cruise Control**：提供 Lag 监控和集群管理界面。

## 2.11 Kafka 的分区分配策略有哪些？
- **RangeAssignor**（默认）：对每个 Topic 的 Partition 按序号范围平均分配给 Consumer，可能导致分配不均（Consumer 多的 Topic 分配倾斜）。
- **RoundRobinAssignor**：将所有 Topic 的 Partition 汇总后轮询分配给 Consumer，分配更均衡。
- **StickyAssignor**：在分配均衡的基础上尽量维持原有分配关系，减少 Rebalance 时的迁移量。
- **CooperativeStickyAssignor**：渐进式 Sticky 策略，支持 Cooperative Rebalance（不暂停所有 Consumer，逐轮调整分配）。

## 2.12 Kafka 的零拷贝是如何实现的？
Kafka 使用 Linux 的 `sendfile()` 系统调用实现零拷贝（Zero Copy）。当 Consumer 拉取数据时，数据从磁盘读取到 PageCache，然后通过 `sendfile()` 直接将数据从 PageCache DMA 拷贝到网卡缓冲区，避免了传统方式中"磁盘→内核缓冲区→用户缓冲区→Socket 缓冲区→网卡"的多次拷贝和上下文切换。

## 2.13 什么情况下会发生 Kafka 消息重复？如何解决？
消息重复的可能原因：
1. Producer 因网络超时重试，但消息实际已写入成功。
2. Consumer 处理完消息但在提交 Offset 前崩溃，Rebalance 后其他消费者从旧 Offset 开始重新消费。

解决方案：
1. **Producer 幂等性**：`enable.idempotence=true`，Broker 基于 Producer ID + Sequence Number 自动去重。
2. **事务**：跨分区原子写入 + Consumer `read_committed`。
3. **消费者业务幂等**：数据库唯一索引、Redis 去重 Key、消息日志表等。
4. **手动提交 Offset**：处理完消息后手动调用 `commitSync()` 提交。

## 2.14 Kafka 集群如何扩容？Partition 如何迁移？
1. **新增 Broker**：部署新 Broker 实例，配置相同的 `broker.id`、`zookeeper.connect`。
2. **数据迁移（Reassign）**：使用 `kafka-reassign-partitions.sh` 将部分 Partition 迁移到新 Broker。
   - 生成迁移计划 JSON（`--generate`），指定要迁移的 Topic 和目标 Broker 列表。
   - 执行迁移（`--execute`），Kafka 在后台逐步迁移数据。
   - 验证迁移进度（`--verify`），确认所有 Partition 迁移完成。
3. 迁移过程中原 Broker 仍正常服务，对生产消费影响较小。注意替换会消耗集群 I/O 和网络带宽，建议低峰期操作。

## 2.15 RabbitMQ、Kafka和RocketMQ有什么区别？如何选型？
| 维度 | RabbitMQ | Kafka | RocketMQ |
|------|----------|-------|----------|
| 语言 | Erlang | Scala/Java | Java |
| 吞吐量 | 万级 | 百万级 | 十万级 |
| 延时 | 微秒级（最低） | 毫秒级 | 毫秒级 |
| 消息可靠性 | 高（主从镜像队列） | 高（ISR副本+acks=-1） | 高（同步刷盘+主从） |
| 消费模式 | Push（AMQP） | Pull（长轮询） | Pull（长轮询）+Push |
| 消息过滤 | 基于Routing Key | 无（客户端过滤） | 原生Tag+SQL92过滤 |
| 延时消息 | 插件（rabbitmq_delayed_message_exchange） | 无原生支持 | 原生18级延时消息 |
| 事务消息 | 较弱（Tx机制性能差） | 支持（幂等+事务） | 原生事务消息（半消息） |
| 消息堆积 | 差（积压影响性能） | 强（磁盘容量决定，天然擅长堆积） | 强（磁盘容量决定，优化更好） |
| 顺序消息 | 单队列有序 | 单Partition有序 | 分区有序+全局有序 |
| 重试机制 | 自动重试+死信队列 | 需自行实现 | 原生支持重试+死信 |
| 适用场景 | 业务消息（AMQP标准） | 日志/流处理/大数据管道 | 电商/金融/核心业务 |
| 生态 | AMQP标准、管理UI完善 | 大数据生态强 | 国内生态强（阿里系） |
| 运维成本 | 低（管理UI友好） | 中（旧版需ZK，新版KRaft简化） | 中（NameServer轻量） |

**选型建议**：
- **RabbitMQ**：适合中小规模业务消息，AMQP标准协议，延迟最低，管理界面友好。适合企业内部系统集成、RPC调用等场景。
- **Kafka**：适合海量日志收集、流处理、大数据管道。高吞吐是核心优势，但无原生延时消息/消息过滤。适合ELK日志收集、用户行为埋点、实时流计算。
- **RocketMQ**：适合核心业务消息（电商下单、支付），原生支持事务消息和延时消息，针对电商/金融场景优化。国内Java生态首选。

**一句话选型**：业务消息小规模选RabbitMQ，大数据日志流处理选Kafka，电商/金融核心业务选RocketMQ。
