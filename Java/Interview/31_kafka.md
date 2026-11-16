本篇覆盖 Kafka 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：架构与存储、可靠性与一致性、消费模型、生态与运维、选型与对比。RocketMQ 的机制与对比见 13_rocketmq.md，RabbitMQ 的协议与路由见 37_rabbitmq.md，日志采集链路见 14_canal.md 与 24_elastic_search.md，消息驱动的分布式事务见 21_seata.md。

# 1. 架构与存储

## 1.1 集群与元数据

### 1.1.1 Kafka 的核心架构是什么？ZooKeeper 与 KRaft 差在哪？4.0 有什么变化？

**核心组件（一张图记忆）**：

```text
Producer ──▶ Broker 集群（Topic → Partition → Replica）
                │  每个 Partition 一个 Leader + N 个 Follower
                ▼
Consumer Group（组内一个 Partition 只给一个 Consumer）

元数据层（旧）：ZooKeeper（/brokers、/controller、/consumers...）
元数据层（新）：KRaft —— 控制器 Quorum 自己维护一份元数据日志
```

**KRaft 的动机与机制**：ZooKeeper 是外部依赖（要单独运维、元数据双写、Controller 故障恢复慢、分区数上不去）；KRaft 把**元数据也当成一个日志主题**（`__cluster_metadata`），由 **3/5 个 Controller 组成 Raft Quorum** 复制，**Active Controller 处理写、Standby 随时接管**——收益：部署少一个组件、Controller 切换秒级、**支持百万级分区**、元数据变更可回放。

**版本线（2026 口径）**：2.8 KRaft 早期预览 → 3.3 新集群可生产用 → 3.5 ZooKeeper 标记废弃 → **4.0（2025 年）彻底移除 ZooKeeper，仅支持 KRaft**；同时抬高了 Java 版本要求（broker/工具需 Java 17，客户端按 release notes），并带来**新一代消费者协议（KIP-848）GA、Queues for Kafka（KIP-932，队列语义）早期可用**等变化。**面试口径**："新集群直接 KRaft；存量 ZK 集群按版本迁移工具（3.6+ 的 ZK→KRaft 迁移）滚动切换，4.0 之后没有 ZK 选项。"

**Topics/Partitions 的语义**：Topic 是逻辑分类，**Partition 是并行与有序的基本单位**（同一 Partition 内严格有序，跨 Partition 无序）；**Partition 数决定最大消费并行度**（组内消费者多于分区就有人空闲）；**副本数（RF）决定冗余**（RF=3 是生产常态）。

**易错**：把 Broker 当"主从"（Kafka 每个 Partition 各有 Leader，**没有全局主从**）；以为 KRaft 用 ZooKeeper 选 Controller（KRaft 是自选举，与 ZK 无关）；"4.0 必须 Java 17"说成"客户端也必须"（按官方矩阵）。

### 1.1.2 Controller 与 Group Coordinator 各管什么？Leader 选举怎么发生？

**两个"协调者"别混**：

| 角色 | 职责 | 位置 |
|------|------|------|
| **Controller** | **集群级元数据**：Partition Leader/ISR 变更、副本分配、Broker 上下线处理；把元数据下发给所有 Broker | 旧版：ZK 抢 `/controller` 临时节点；KRaft：Quorum 中的 Active Controller |
| **Group Coordinator** | **消费组级**：管理 `JoinGroup/SyncGroup/Heartbeat/OffsetCommit`，负责 Rebalance 与位点存储 | 某个 Broker（按 `group.id` 哈希到 `__consumer_offsets` 的某个分区） |

**Partition Leader 选举（故障时）**：Controller 发现 Leader 所在 Broker 下线 → **从 ISR 中选一个副本**（优先 `preferred leader`）→ 更新元数据并通知相关 Broker；**若 ISR 为空**：`unclean.leader.election.enable=true` 允许从 OSR（落后副本）选 Leader（**保可用、可能丢数据**），`false` 则**该分区不可用直到 ISR 恢复**（保一致）——这是"可用性 vs 一致性"在 Kafka 里最直白的开关（2.1.2 展开）。

**Controller 故障**：旧版 ZK 临时节点消失 → 其他 Broker 抢注册（**恢复期可能秒级~分钟级**，分区多时更慢）；KRaft 的 Standby Controller 通过 Raft 直接接管（**秒级**，且元数据已同步）。**Controller 抖动**是常见故障：日志里频繁 `controller epoch` 变化、分区 Leader 反复切换——查 ZK 会话超时/网络/KRaft quorum 健康（4.2.3）。

**易错**：把 Group Coordinator 当 Controller（一个管消费组、一个管集群）；以为"Leader 挂了从所有副本里选"（默认只从 ISR 选）；忽略 `preferred leader` 回迁（扩容/重启后 Leader 分布不均，`kafka-preferred-replica-election` 或自动回迁）。

## 1.2 存储模型

### 1.2.1 Topic/Partition/Segment/Offset 的结构？稀疏索引怎么查消息？

**三层结构**：

```text
Topic（逻辑）
 └─ Partition（有序日志，offset 从 0 递增）
     └─ Segment（物理文件，按大小/时间滚动，默认 1GB）
         ├─ 00000000000000000000.log      数据（消息批次）
         ├─ 00000000000000000000.index    offset → 物理位置（稀疏）
         ├─ 00000000000000000000.timeindex 时间戳 → offset（稀疏）
         └─ leader-epoch-checkpoint        Leader 纪元（防截断错乱）
```

**写入**：消息**追加**到当前活跃 Segment 的末尾（顺序写），写满（`segment.bytes`）或到期（`segment.ms`）就滚动新文件；**读取**：给定 offset，先**二分查找 `.index`**（稀疏索引，默认每 4KB 记一条）定位到"不大于目标 offset 的最近索引项"→ 从该物理位置**顺序扫描** `.log` 找到目标——**索引稀疏换内存，扫描量有界**。

**Offset 的语义**：Partition 内单调递增（不是全局唯一）；**消费位点是"下一条要读的 offset"**；`__consumer_offsets` 里存的是"组 → 分区 → 已提交位点"。**日志保留**：按时间（`retention.ms`，默认 7 天）或大小（`retention.bytes`）删除**整段**（不是删单条）——所以"消息过期"是段级行为，**没消费完就过期=真的丢**（3.2.2 的堆积账）。

**易错**：以为 offset 全局唯一（每个分区各自从 0 开始）；以为删除是逐条（段级删除）；把 `.index` 当"全量索引"（稀疏，靠扫描补齐）；忽略 `leader-epoch-checkpoint`（Leader 切换后日志截断的正确性依赖它）。

### 1.2.2 Kafka 为什么吞吐高？顺序写、PageCache、零拷贝、批量、压缩各贡献多少？

**六层叠加（答题按"从磁盘到网络"的顺序讲）**：

1. **顺序追加写**：所有消息 append 到 Segment 末尾——**顺序 IO 的吞吐远高于随机 IO**（尤其 HDD；SSD 也受益于减少写放大）；对比"B 树随机更新"的数据库，Kafka 把"写"做成了日志流；
2. **PageCache（OS 页缓存）**：Kafka **不自己管理缓存**（JVM 堆保持较小），读写都走内核 page cache——**热数据直接命中内存**、写先落缓存由 OS 异步刷盘（`log.flush.*` 默认不强制每次 fsync，靠副本而不是单机刷盘保证可靠，2.1.2）；
3. **零拷贝（sendfile）**：Consumer 拉取时，数据从 **PageCache 直接 DMA 到网卡**，省掉"内核→用户态→内核"的两次拷贝与上下文切换；**前提**：未压缩/无需转换、命中 page cache、走普通 fetch 路径（**SSL 加密会破坏零拷贝**，要用户态加密）；
4. **批量**：Producer 端 `batch.size`（默认 16KB）+ `linger.ms`（默认 0，生产常调 5~20ms）攒批发送；Broker 按批存储；Consumer 批量拉取（`fetch.min.bytes`/`max.partition.fetch.bytes`）——**每请求的固定开销被摊薄**；
5. **压缩**：Producer 端 gzip/snappy/lz4/zstd（**Broker 原样存、不解压**，Consumer 解压）——省网络与磁盘，代价是 CPU；`compression.type=producer` 默认；
6. **分区并行 + 水平扩展**：多 Partition 分布在多 Broker，读写天然并行；加 Broker/分区扩容吞吐（4.1.2）。

**"为什么不用 JVM 堆缓存"（追问）**：堆内缓存会**吃 GC**（大堆停顿）、且数据从 page cache 还要再拷一次到堆；Kafka 选择"**把内存交给 OS**"，JVM 只做网络与索引——这也是 Broker 堆不用很大的原因（30_linux.md 的页缓存账）。

**易错**：把"零拷贝"说成"完全不拷贝"（DMA 仍有拷贝，省的是 CPU 拷贝与上下文切换）；以为"刷盘策略=每次 fsync"（默认靠副本 + OS 刷盘）；压缩算法选 zstd 就一定好（CPU 与兼容性要压测）；`linger.ms` 调大提升吞吐但增加延迟（业务权衡）。

### 1.2.3 日志清理策略 delete 与 compact 有什么区别？分层存储是什么？

**delete（默认）**：按时间/大小**删除整个旧 Segment**——适合"事件流/日志"（历史不需要逐条保留）；`retention.ms`/`retention.bytes`、`segment.ms` 控制粒度；**注意**：段内有未消费消息也会被删（3.2.2 的堆积风险）。

**compact（压实）**：**按 Key 保留最新值**，旧值在后台合并时清除——适合"变更日志/状态快照"（如数据库 CDC、Kafka Streams 的 state store changelog、配置分发）；**墓碑（tombstone）**：value 为 null 的消息表示"删除该 key"，保留 `delete.retention.ms` 后清除；**compact 是后台异步的**（不保证实时）、**不保证 offset 连续**（中间会有空洞，消费端要容忍）。

**两者可组合**（`cleanup.policy=compact,delete`：既压实又按时间删）。

**分层存储（Tiered Storage，KIP-405）**：把**冷 Segment 下沉到对象存储**（S3/OSS），本地只留索引与热数据——解决"为了历史数据留大量 Broker 磁盘"的成本问题；版本节奏：3.6 早期预览、3.9/4.0 进入可用（以官方为准）；**收益**：存算分离、扩容不用搬历史、保留期可以拉长；**代价**：冷读延迟（要回源对象存储）、运维多一个依赖（25_minio.md 可作为自建对象存储层）。

**易错**：把 compact 当"去重所有重复消息"（只按 key 保留最新，且是后台异步）；以为 compact 后 offset 连续（会有空洞）；用 compact 存"需要全量历史"的事件（应该 delete + 下游归档）；开了分层存储但没配 retention（本地照样涨）。

# 2. 可靠性与一致性

## 2.1 副本与 ISR

### 2.1.1 副本、ISR、HW/LEO、Leader Epoch 怎么协作？

**四个概念一条链**：
- **AR（Assigned Replicas）**：分区的全部副本（如 3 个）；
- **ISR（In-Sync Replicas）**：**与 Leader 保持同步**的副本集合——Follower 的拉取延迟不超过 `replica.lag.time.max.ms`（默认 **30s**）就在 ISR 里，超时被踢到 **OSR**，追上后回归（`AR = ISR + OSR`）；
- **LEO（Log End Offset）**：每个副本"下一条要写入的 offset"（副本各自的进度）；
- **HW（High Watermark）**：**ISR 中所有副本 LEO 的最小值**——**消费者只能读到 offset < HW 的消息**（保证"读到的消息至少在 ISR 里都存在"，不会因 Leader 切换而消失）。

**协作流程**：Producer 写入 Leader（追加到 LEO）→ Follower 发 Fetch 请求拉取 → Leader 返回数据 + 当前 HW → Follower 写入并更新自己的 LEO/HW → **Leader 的 HW 随 ISR 最小 LEO 推进** → Consumer 只能消费到 HW 之前。

**Leader Epoch（解决 HW 截断的历史坑）**：Leader 切换时，旧 Leader 可能"多写了一段但未进 HW"，新 Leader 若简单按 HW 截断会**丢已提交数据/错乱**——Kafka 给每次 Leader 任期一个 **epoch（纪元）**，Follower 用 `(epoch, startOffset)` 判断"哪些段是自己独有的、需要截断"，实现**安全截断**。

**两个推论（答题深度）**：① **ISR 是"动态的可靠性预算"**——ISR 缩到 1 时，`acks=all` 也只等 Leader 自己（配合 `min.insync.replicas` 才能拒绝写入，2.1.2）；② **HW 推进有延迟**——Follower 没跟上时 Consumer 读不到最新消息（"生产成功但消费看不到"先看 ISR/HW，不一定是 bug）。

### 2.1.2 acks、min.insync.replicas、unclean 选举怎么组合才不丢消息？

**三个参数的分工**：

| 参数 | 位置 | 作用 |
|------|------|------|
| **acks** | Producer | 等谁确认：`0` 不等、`1` 等 Leader、`all/-1` 等 **ISR 全部** |
| **min.insync.replicas** | Broker/Topic | **ISR 少于该值时，`acks=all` 的写入直接失败**（`NotEnoughReplicas`）——防止"ISR 只剩 Leader 还假装成功" |
| **unclean.leader.election.enable** | Broker | ISR 为空时是否允许从 OSR 选 Leader（**true=保可用可能丢数据；false=保一致牺牲可用**） |

**推荐组合（生产标配）**：`RF=3` + `acks=all` + `min.insync.replicas=2` + `unclean=false` + `enable.idempotence=true`——含义：**容忍 1 个 Broker 挂而不丢已确认消息**；若挂 2 个（ISR 只剩 1 < min.insync=2），写入**快速失败**（而不是悄悄接受后丢），等副本恢复。

**消息丢失的完整清单（按环节）**：
1. **Producer**：`acks=0/1` + Leader 在复制前崩溃 → 丢；**重试 + 幂等**解决"重试导致重复"但不解决"没写进去"（要靠 acks/min.insync）；
2. **Broker**：`RF=1` 或 `acks=1` + Leader 崩溃；**unclean 选举**从落后副本上位（丢掉它没同步的部分）；
3. **刷盘**：Kafka **默认不每条 fsync**（靠副本冗余）——**所有副本同时断电**（机房掉电）仍可能丢 page cache 里的数据；金融级要评估"副本跨机架 + 定期刷盘/下层存储"；
4. **Consumer**：**先提交 offset 后处理**（处理失败也"消费过了"）、自动提交窗口内崩溃（重复而非丢失）、位点重置到末尾（主动跳过）。

**答题收口**："**Kafka 的可靠性是'副本 + acks + min.insync'三件套的乘积**——只配 `acks=all` 不配 `min.insync.replicas`，在 ISR 缩水时会静默降级；只配副本不配 `unclean=false`，故障时可能'可用但数据倒退'。"

## 2.2 投递语义

### 2.2.1 三种投递语义是什么？幂等 Producer 与事务怎么实现 Exactly Once？

**三种语义**：
- **At most once**：先提交位点/不等确认——**可能丢、不重复**（日志采集可接受）；
- **At least once**（默认）：处理完再提交——**不丢、可能重复**（业务主流，靠消费端幂等兜底）；
- **Exactly once**：端到端"不丢不重"——Kafka 的 EOS 靠**幂等 Producer + 事务 + read_committed**。

**幂等 Producer（解决"重试导致的重复"）**：`enable.idempotence=true`（**3.0+ 默认开启**）→ Producer 获得 **PID**，每个 Partition 维护**单调 Sequence Number**；Broker 按 `(PID, Partition, Seq)` 去重、拒绝乱序——**同一条消息重试多次只写一次**；约束：`acks=all`、`max.in.flight ≤ 5`（在此范围内仍保序）。

**事务（解决"跨分区原子写 + 消费位点原子提交"）**：
```java
producer.initTransactions();                 // 向 Transaction Coordinator 注册 transactional.id
producer.beginTransaction();
producer.send(record1); producer.send(record2);   // 跨分区
producer.sendOffsetsToTransaction(offsets, groupId);  // 位点也纳入事务（消费-处理-生产）
producer.commitTransaction();                // 或 abortTransaction()
```
**机制**：**Transaction Coordinator**（按 `transactional.id` 哈希到 `__transaction_state` 的某个分区）协调两阶段；消息带事务标记，分区里写 **COMMIT/ABORT marker**；Consumer 配 **`isolation.level=read_committed`** 时：**只读到 LSO（Last Stable Offset）之前、且已提交的消息**，未提交/已中止的被跳过。

**边界（成熟度）**：Kafka 事务只保证 **"Kafka 内部跨分区"** 的原子性——**消费端写数据库/调外部接口不在事务里**（要用 outbox/幂等，34_distributed_theory.md）；EOS 有性能开销；`read_committed` 会增加读延迟（等 LSO）。

**易错**：以为 `enable.idempotence=true` 就端到端 Exactly Once（只是 Producer→Broker 段）；事务 Producer 忘了 `transactional.id`（重启后无法 fence 旧实例）；Consumer 用 `read_uncommitted` 却期望不读脏（读得到未提交/已中止）；把事务当"分布式事务"（不覆盖外部系统）。

### 2.2.2 消息重复与乱序从哪来？顺序性怎么保证？

**重复的四个来源**：① **Producer 重试**（消息已写入但 ACK 丢 → 重发；幂等 Producer 解决）；② **Consumer 处理成功但提交位点前崩溃**（Rebalance 后从旧位点重放；业务幂等解决）；③ **Rebalance 本身**（分区换人，位点未提交部分重放）；④ **人为重放**（位点重置/回溯，3.2.2）。

**乱序的四个来源**：① **Producer 重试 + 多 in-flight**（没有幂等时，重试批次可能后到；`max.in.flight=1` 或开幂等解决）；② **跨分区**（Kafka 天生只保分区内有序）；③ **扩分区**（key 的哈希模数变化，部分 key 换分区，过渡期同一 key 可能落两个分区——2.2.2 的"扩分区代价"）；④ **消费端并发处理**（多线程处理同一分区、异步回调乱序提交）。

**顺序性保证的四步**：
1. **同一业务键 → 同一 Partition**：Producer 用 **key**（订单号/用户 ID），默认分区器 `murmur2(key) % partitions`；**null key 用粘性分区器**（2.4+，攒批到同一分区，不保证 key 语义）；
2. **Producer 端保序**：开幂等（默认）或 `max.in.flight=1`；`retries>0`；
3. **Consumer 端单分区单线程**（或按 key 哈希到固定 worker，key 内串行）；**提交位点按"连续完成的最小 offset"**（别乱序提交）；
4. **扩分区要停写/双写过渡**（同 13_rocketmq.md 的队列扩容纪律）。

**业务幂等的四种实现**（与 13_rocketmq.md 3.2.3 同源）：唯一索引（最强）、状态机（订单状态天然幂等）、去重表（同事务）、Redis SETNX（有窗口，配合 TTL）。

**收口**："**Kafka 只承诺'分区内有序 + at-least-once'**——端到端的'不重不乱'是**生产端幂等 + 消费端幂等 + key 路由 + 位点纪律**四件事共同完成的，不是某个开关。"

# 3. 消费模型

## 3.1 消费者组

### 3.1.1 Consumer Group 怎么工作？Offset 存在哪、怎么提交？

**组内语义**：同一 `group.id` 的消费者**竞争消费** Topic 的所有 Partition——**一个 Partition 同一时刻只属于组内一个 Consumer**（所以消费者数 > 分区数时有人空闲；消费者数 < 分区数时有人多担）。

**协调者（Group Coordinator）**：按 `group.id` 哈希到 **`__consumer_offsets`** 的某个分区，所在 Broker 就是该组的 Coordinator——负责 `JoinGroup`（组员加入/选组长）、`SyncGroup`（下发分配方案）、`Heartbeat`（保活）、`OffsetCommit/Fetch`（位点读写）。

**位点存储**：`__consumer_offsets` 是**内部 compact Topic**（默认 50 分区）——key 是 `(group, topic, partition)`，value 是 offset + 元数据；**compact 保证每个 key 只留最新位点**（1.2.3）。**位点提交**：
- **自动提交**（`enable.auto.commit=true`，默认 5s）：**可能重复**（处理完但还没到提交周期就崩溃）——"至少一次"的默认实现；
- **手动同步**（`commitSync()`）：阻塞、可靠、可重试；
- **手动异步**（`commitAsync()`）：不阻塞、失败不自动重试（回调里处理）——**常用"异步 + 定期同步兜底"**；
- **提交时机**：**处理完业务再提交**（先提交后处理 = 可能丢，2.1.2）。

**易错**：消费者数超过分区数（多余实例空转）；自动提交配"处理很慢"（重复窗口大）；`commitSync` 在 poll 循环外/异常路径没执行（重复）；Consumer 换组（`group.id` 变了）从零/最新开始（`auto.offset.reset` 的语义：无位点时的起点，`earliest/latest`）。

### 3.1.2 Rebalance 的触发、流程与分配策略？怎么减少 Rebalance？

**触发条件**：组成员加入/离开（发布/扩容/崩溃）、订阅的 Topic/分区变化、**心跳超时**（`session.timeout.ms`）、**处理超时**（`max.poll.interval.ms` 内没 poll）、Coordinator 变更。

**流程（Eager 协议）**：Coordinator 收到 JoinGroup → 选 Group Leader（第一个加入的）→ Leader 按分配策略算方案 → SyncGroup 下发 → **所有消费者 Revoke 全部分区、暂停消费**（Stop-the-world）→ 重新分配 → 恢复。**Cooperative 协议**：只 Revoke **需要迁移的分区**，分多轮增量调整（**不全局暂停**），大幅降低 Rebalance 抖动。

**四种分配策略**：

| 策略 | 逻辑 | 特点 |
|------|------|------|
| **Range**（默认） | 按 Topic 的分区序号范围平均分 | 多 Topic 时容易倾斜（每个 Topic 都从头分） |
| **RoundRobin** | 所有 Topic 的分区汇总后轮询 | 更均衡；订阅不一致时可能错配 |
| **Sticky** | 均衡 + **尽量保留原分配** | 减少 Rebalance 迁移量 |
| **CooperativeSticky** | Sticky + **增量式 Rebalance** | 生产推荐（配合 `cooperative-sticky` 协议） |

**减少 Rebalance 的六条**：① **静态成员**（`group.instance.id`：重启不触发 Rebalance，在 session 超时内回来即可）；② **合理 session.timeout**（别太短，网络抖动误判）；③ **处理时间 < max.poll.interval**（调大或减少 `max.poll.records`）；④ **发布滚动而非全停**（一次只重启一个实例）；⑤ **CooperativeSticky**；⑥ **4.0 的新消费者协议（KIP-848）**——服务端主导分配、Rebalance 更快更平滑（新版本加分项）。

**易错**：把 `session.timeout` 当"处理超时"（心跳线程独立，处理慢是 `max.poll.interval` 的事，3.2.1）；Rebalance 期间**重复消费**（位点未提交）与**消费停顿**（Eager）是两笔账；分配策略在客户端配置，升级/换策略要全组一致。

## 3.2 消费治理

### 3.2.1 session.timeout、heartbeat、max.poll.interval 各管什么？怎么配？

**三个时间的三层职责（高频混淆）**：

| 参数 | 默认 | 管什么 | 配错的后果 |
|------|------|--------|-----------|
| **session.timeout.ms** | 45s（3.x；旧版 10s） | **组员存活**：超过没心跳，Coordinator 认为它死了并踢出组 | 太短：网络抖动/GC 停顿就误踢（频繁 Rebalance）；太长：真挂了要等很久才重分配 |
| **heartbeat.interval.ms** | 3s | 心跳频率（通常为 session 的 1/3） | 太频繁增加 Coordinator 压力；太慢来不及续命 |
| **max.poll.interval.ms** | 5min | **两次 poll 的最大间隔**：处理太慢、超过它，消费者被踢（即使心跳正常） | 太小：长处理任务被踢（重复消费）；太大：卡死要等很久才被发现 |
| **max.poll.records** | 500 | 一次 poll 返回的最大条数 | 太大：单批处理时间长（超 poll interval）；太小：吞吐低 |

**关键认知**：**心跳由独立线程发送**——所以"进程活着、心跳正常"不等于"消费在推进"；**处理慢到超过 `max.poll.interval` 才会被踢**（这是"消费者明明在跑却被 Rebalance"的第一嫌疑）。**配法**：处理耗时 P99 × 批量条数 < `max.poll.interval`（留 2~3 倍余量）；session 用默认 45s 一般够，跨机房/GC 长的场景适当加大。

**长处理任务的三种解**：减小 `max.poll.records`（批小、频繁 poll）；把耗时逻辑异步化（poll 线程只入队，**但要自己控制队列积压与位点提交**）；调大 `max.poll.interval`（最省事，但故障发现变慢）。

**易错**：只调 session 不调 poll interval（处理慢照样被踢）；消费线程里 `sleep`/同步调用外部（把 poll 卡住）；异步处理后**立刻提交位点**（消息还在队列里没处理完，崩了丢）；`max.poll.records` 调到几万（单批处理超时）。

### 3.2.2 Consumer Lag 怎么监控与处理？堆积了怎么扩容？

**Lag 的定义**：`Lag = 分区 LEO - 消费组已提交 offset`——**每个分区分别算，再求和/取最大**（只看总和会掩盖"个别分区积压"）。**工具**：`kafka-consumer-groups.sh --describe`（CURRENT-OFFSET/LOG-END-OFFSET/LAG）、JMX（`records-lag-max`）、Burrow、Kafka Exporter + Prometheus/Grafana、Cruise Control。

**Lag 增长的四种分型**：
1. **消费能力不足**（生产 TPS > 消费 TPS）：加消费者（**上限=分区数**，3.1.1）、优化单条处理（批量/异步/慢 SQL）、扩分区（有代价，4.1.2）；
2. **消费卡住**（个别分区 lag 飙升）：毒消息（处理必失败，反复重试）、下游依赖挂、锁等待——看消费者线程栈与错误日志；
3. **Rebalance 抖动**（3.1.2/3.2.1）：消费频繁暂停，lag 锯齿状；
4. **生产突增**（上游活动/重放）：限流/削峰，先保消费。

**扩容的正确顺序**：**先优化单条处理，再加实例**（加实例受分区数限制；分区多于实例时先加实例）→ 还不够则**扩分区**（注意 key 顺序，4.1.2）→ 再不够考虑**堆积中转/分流**（类似 13_rocketmq.md 3.3.2 的搬迁方案）。

**必须监控的两条**：**lag 的增长斜率**（比绝对值更早报警）与**保留期余量**（lag 追不上 + 消息快过期 = 真丢，1.2.1）——"堆积不怕、过期才怕"是这题的收口句（13_rocketmq.md 同款）。

# 4. 生态与运维

## 4.1 生态

### 4.1.1 Kafka Connect、Streams、MirrorMaker 各解决什么？

**Kafka Connect（数据进出管道）**：把外部系统与 Kafka 之间的**同步**做成"配置化连接器"——**Source Connector**（MySQL/CDC/文件/日志 → Kafka）、**Sink Connector**（Kafka → ES/HDFS/S3/DB）；运行在 **Worker**（standalone/distributed）上，用 REST API 管理；**Converters**（JSON/Avro/Protobuf）与 **SMT（单消息转换）** 做轻量加工；位点/配置/状态存内部 Topic（`connect-configs/offsets/status`）。**定位**：**"搬数据"不要自己写消费者**——CDC 场景（Debezium）是它的主场（14_canal.md 的同族能力）。

**Kafka Streams（流处理库）**：**嵌入式库**（不是集群），应用里定义 **Topology**（Source → Processor → Sink）；提供 `KStream/KTable/GlobalKTable`、窗口（滚动/滑动/会话）、Join、聚合；**状态存储用 RocksDB + changelog Topic**（故障恢复靠回放 changelog）；支持 **Exactly-Once v2**（事务）；**Interactive Queries** 可查本地状态。**定位**：轻量流处理（无需 Flink 集群），Java 应用内做实时聚合/ enrichment。

**MirrorMaker 2（跨集群复制）**：基于 Connect 的**多数据中心复制**——复制 Topic 数据、**消费组位点**、ACL/配置（可配）；支持**主备（active-passive）与双活（active-active）**；**注意**：异步复制（RPO>0）、双活要防"循环复制"与冲突（同 key 双写）、位点翻译的语义（消费者切集群后从哪继续）。**同类**：Confluent Replicator（商业）、云厂商的跨区复制。

**其他生态一句话**：Schema Registry（模式管理，Avro/Protobuf 兼容性——**生产强烈建议**，否则上下游字段演进全靠默契）；ksqlDB（SQL 流处理，Confluent）；Cruise Control（自动均衡/扩缩容）；Kafka Exporter（Prometheus 指标）。

### 4.1.2 分区数怎么定？扩分区有什么代价？

**定量口径（三步）**：
1. **按吞吐**：单分区可承载的吞吐（生产+消费，经验值**几 MB/s ~ 几十 MB/s**，随压缩/批大小/硬件变化）→ `分区数 ≈ 峰值吞吐 / 单分区吞吐`；
2. **按并行度**：**分区数 ≥ 消费组最大并行度**（消费者数上限=分区数，3.1.1）——按"未来 1~2 年峰值消费能力"定；
3. **按约束**：单 Broker 分区总数有上限（文件句柄/内存/元数据，经验上**每 Broker 数千**要警惕；KRaft 抬高了天花板但硬件仍有边界）；**分区不是越多越好**——每分区都有 Leader 选举、文件、内存与 Rebalance 成本。

**扩分区的四个代价（必答）**：
1. **key 的顺序被打破**：分区器是 `hash(key) % 分区数`，**模数变了 → 部分 key 换分区**，同一 key 的旧消息在新旧分区各一段——**有序业务必须停写/双写过渡**（13_rocketmq.md 的队列扩容同款）；
2. **Rebalance**：分区变化触发消费组重分配（3.1.2）；
3. **不能减**：Kafka **不支持减少分区**（只能新建 Topic + 迁移数据）；
4. **端到端延迟/资源**：更多分区=更多小批次、更多文件与元数据、Leader 切换更慢。

**规划口径**："**按终局容量一次定足、宁多勿少但要克制**"——分区多带来的运维成本是持续的，**先按 2~3 年峰值算，再打折**；顺序敏感的业务把"扩分区"当变更管理（评审+窗口）。

## 4.2 运维

### 4.2.1 集群扩容与分区迁移怎么做？

**扩容（加 Broker）**：
1. 新节点**同版本、同配置**、唯一 `broker.id`（KRaft 为 `node.id`），配好 `broker.rack`（机架感知，2.1.1 的故障域）、监听地址、Controller Quorum 连接；
2. 启动后加入集群（**新 Broker 不会自动接收存量分区**——只是"有容量了"）；
3. **分区迁移**：`kafka-reassign-partitions.sh --generate` 生成计划（指定 Topic 与目标 Broker）→ 检查 JSON → `--execute`（可加 `--throttle` 限速）→ `--verify` 确认；
4. **迁移期间**：副本追数据会吃网络/磁盘，**低峰执行 + 限流**；观察 **UnderReplicatedPartitions** 与 ISR；
5. **下线旧 Broker**：先把它的副本迁走（或 `kafka-leader-election` 迁 Leader）→ 确认无分区依赖 → 停节点。

**自动化**：**Cruise Control** 做"均衡/扩缩容/故障转移"的自动计划（比手工 JSON 稳）；云托管（MSK 等）提供一键扩缩。**KRaft 特有**：Controller Quorum 的增删用 `kafka-metadata-quorum`（动态重配置，版本支持以文档为准）。

**易错**：只加 Broker 不迁移（新节点闲置、老节点仍满）；迁移不限速（打满网络，影响生产）；忘了 `broker.rack`（副本全在同一机架，机架故障全丢）；迁移与滚动升级/大促叠加（变更冲突）。

### 4.2.2 安全（SASL/SSL/ACL）与配额怎么配？

**传输安全**：`listeners=SASL_SSL://...`（生产标配）——**SSL** 加密（keystore/truststore、证书轮换）、**SASL** 认证（`PLAIN` 仅测试；`SCRAM-SHA-256/512` 用户名密码；`GSSAPI` Kerberos；`OAUTHBEARER` 云环境）。客户端配 `security.protocol`、`sasl.mechanism`、`sasl.jaas.config`。

**授权（ACL）**：启用 `authorizer.class.name`，用 `kafka-acls.sh` 管理——**资源**（Topic/Group/Cluster/TransactionalId）、**操作**（Read/Write/Create/Delete/Alter/Describe）、**主体**（`User:alice`）；原则：**按应用最小授权**（生产者只 Write 自己的 Topic、消费者只 Read 需要的 Topic + 自己的 Group）、**禁止通配 `*`**、`super.users` 只给运维。**注意**：ACL 变更要评估对现有客户端的影响（先 Describe 再收紧）。

**配额（防单租户打爆集群）**：`producer_byte_rate`/`consumer_byte_rate`/`request_percentage`，按 **user + client-id** 限速（`kafka-configs.sh --alter`）；超限是**延迟而不是拒绝**（客户端被限流），监控 quota 指标。**多租户**场景还要配合 Topic 命名/资源隔离/独立集群。

**其余**：**静态加密**（磁盘/云盘加密）、**审计**（谁读了哪个 Topic）、**凭证轮换**（SCRAM 密码/证书）、**JMX 端口**别裸奔（30_linux.md 的安全基线同款）。

### 4.2.3 常见故障怎么排查？（Controller 抖动、ISR 缩水、磁盘满、请求超时）

| 症状 | 根因线索 | 排查动作 |
|------|---------|---------|
| **UnderReplicatedPartitions > 0** | Broker 慢/磁盘满/网络分区/GC 停顿，Follower 追不上（超 30s 出 ISR） | 看 Broker 磁盘 IO/网络/GC；`kafka-topics --describe` 看 ISR；恢复后自动回 ISR |
| **OfflinePartitions > 0** | 该分区所有副本都不可用（Broker 全挂/磁盘坏） | 恢复 Broker；评估 `unclean` 选举（丢数据风险）；从副本/备份恢复 |
| **Controller 抖动/频繁重选** | ZK 会话超时（旧版）、网络抖动、KRaft quorum 失联、GC 长停顿 | 看 controller 日志与 epoch；ZK/KRaft 健康；网络与时钟（30_linux.md 的 NTP） |
| **磁盘满** | retention 太长/流量涨/未开分层存储 | 调 `retention.ms/bytes`、清理、扩盘、分层存储（1.2.3）；**别直接删 `.log` 文件**（索引与数据不一致） |
| **请求超时/队列满** | 网络线程/IO 线程打满、请求队列满、页缓存不足 | 看 `RequestChannel` 队列、磁盘 await、网络；扩 Broker/调线程/限流 |
| **消费 lag 飙升** | 3.2.2 的四种分型 | 先看是"消费慢/卡住/抖动/生产突增" |
| **消息过大被拒** | `max.message.bytes`/`max.request.size`/`replica.fetch.max.bytes` 不一致 | 统一调大（有上限）或改用"声明检查"（大消息存对象存储，消息只带引用，25_minio.md） |
| **OffsetOutOfRange** | 位点对应的消息已过期删除/位点被重置 | 按 `auto.offset.reset` 或手工重置；核对 retention 与 lag（3.2.2） |

**排障顺序**：先看**集群级指标**（URP/离线分区/Controller/磁盘/网络）→ 再定位到**具体 Broker/分区**（日志、磁盘、GC）→ 最后看**客户端**（生产/消费错误码、Rebalance）。**保留现场**：Broker 日志、JMX 快照、`kafka-dump-log`（看分段内容）。

# 5. 选型与对比

## 5.1 对比

### 5.1.1 Kafka、RocketMQ、RabbitMQ 怎么选？

| 维度 | Kafka | RocketMQ | RabbitMQ |
|------|-------|----------|----------|
| 定位 | **日志/流/大数据管道**，吞吐优先 | **业务消息**（电商/金融），功能全 | **低延迟/复杂路由**，AMQP 生态 |
| 吞吐 | **百万级**（批+顺序写+零拷贝） | 十万级 | 万级（单队列） |
| 延迟 | 毫秒级（批处理换吞吐） | 毫秒级 | **微秒~毫秒级**（最低） |
| 堆积能力 | **极强**（磁盘顺序写，专为堆积设计） | 强（CommitLog） | 弱（堆积影响性能/内存） |
| 延迟消息 | 无原生（外部调度/4.0 前后生态方案） | **原生 18 级** | 插件（delayed message exchange） |
| 事务消息 | 事务（Kafka 内部 EOS）+ 外部靠 outbox | **原生半消息+回查** | 弱（Tx 性能差） |
| 消息过滤 | 无服务端过滤（客户端/Topic 划分） | **Tag/SQL92** | **Routing Key/Headers**（灵活） |
| 顺序 | 分区内有序 | 分区/全局有序 | 单队列有序 |
| 运维 | 中（KRaft 后简化；调优需经验） | 中（NameServer 轻） | 低（管理 UI 友好） |
| 典型 | 日志采集、埋点、CDC、流计算 | 订单/支付/核心链路 | 企业内部集成、RPC 异步化、任务队列 |

**选型句**："**日志/流/大数据 → Kafka；业务消息要事务/延时/过滤 → RocketMQ；低延迟复杂路由/中小规模 → RabbitMQ**"——但先问"团队已有的中间件栈是什么"（多养一套 MQ 的成本常大于技术差异）。

### 5.1.2 Kafka 不适合什么场景？

**六个"别用 Kafka"信号**：
1. **需要原生延时/定时消息**（订单 30 分钟关单）——Kafka 没有（要外部调度器/时间轮，13_rocketmq.md 2.2.1 的对比）；
2. **需要服务端按内容过滤**（Tag/SQL）——Kafka 只能按 Topic/Partition 分，过滤在客户端（浪费带宽）；
3. **需要优先级队列/每消息 TTL**——Kafka 是统一日志，没有优先级；
4. **单条低延迟 + 复杂路由**（RPC 异步化、工作队列）——RabbitMQ 更合适；
5. **超大消息**（默认 1MB 限制）——用"声明检查"（消息带对象存储引用，25_minio.md）而不是硬调大；
6. **小规模/低流量/强事务业务**——运维 Kafka 的复杂度不划算（业务消息用 RocketMQ/云 MQ；数据库事务别指望 Kafka 覆盖外部系统，2.2.1 的边界）。

**补充**：**请求-响应/点对点**不是 Kafka 的主场（消费组是"发布订阅+负载均衡"，不是"给某个消费者"）；**全局严格顺序**只能单分区（吞吐受限）。

## 5.2 面试收口

### 5.2.1 设计一个"订单事件总线"怎么用 Kafka 落地？

**需求**：下单/支付/取消等事件，供库存、积分、风控、通知等多个下游消费；要求不丢、可重放、下游互不影响。

**设计七步（按顺序答）**：
1. **Topic 与 Key**：`order-events`（按事件类型可拆 Topic，如 `order-created/paid/cancelled`），**key = orderId**（同一订单的事件落同一分区 → 有序）；
2. **可靠性配置**：`RF=3` + `min.insync.replicas=2` + `acks=all` + 幂等 Producer（2.1.2/2.2.1）；`unclean=false`；
3. **事件写入与 DB 的一致性**：**本地事务 + Outbox 表**（同库事务写业务表与 outbox）→ CDC（Debezium/Canal，14_canal.md）或独立投递器发 Kafka——**别在业务事务里直接 `send()`**（DB 回滚了消息已发/发失败 DB 已提交）；
4. **消费组隔离**：每个下游一个 `group.id`（库存组/积分组/通知组）——**互不影响位点**；组内按分区并行；
5. **消费幂等与重试**：业务唯一键/状态机防重；失败进**重试 Topic（分级延迟）+ DLQ**（13_rocketmq.md 3.2 的语义在 Kafka 靠自建 Topic 实现——Kafka 没有原生重试队列）；
6. **可观测与治理**：lag 监控（3.2.2）、URP/离线分区告警（4.2.3）、Schema Registry 管事件演进（4.1.1）、ACL 按组授权（4.2.2）；
7. **保留与重放**：`retention` 覆盖"最长故障恢复窗口"（如 7~14 天）；新下游上线从 `earliest` 重放/或用快照+增量。

**两个必须主动说的边界**：**Kafka 事务不覆盖下游 DB**（EOS 只在 Kafka 内部，2.2.1）——"扣库存"的幂等要在库存服务自己做；**双活/异地**用 MirrorMaker 2（异步、RPO>0），别把跨城当同步复制（4.1.1）。

### 5.2.2 2026 年 Kafka 的版本与生态现状？

**版本线**：**4.0（2025 年）是分水岭**——**彻底移除 ZooKeeper（KRaft only）**、Java 版本要求抬高（broker/工具 Java 17，客户端以 release notes 为准）、**新消费者协议 KIP-848 GA**（服务端主导分配、Rebalance 更快）、**Queues for Kafka（KIP-932）早期可用**（共享组、队列语义，向 RabbitMQ/RocketMQ 的"工作队列"靠拢）、**分层存储（KIP-405）进入可用**（冷数据下沉对象存储）。3.x 仍在维护，存量 ZK 集群按官方迁移路径切 KRaft。

**生态格局**：**Apache Kafka 核心仍是 Apache 2.0**；商业发行（Confluent Platform/Cloud）提供 Schema Registry、ksqlDB、Connectors、托管；云上 **AWS MSK / Azure Event Hubs（Kafka 兼容）/ 阿里云消息队列**；**S3 原生/存算分离的 Kafka 兼容产品**（WarpStream/Bufstream/Redpanda 等）在"低成本长保留"场景分流——面试报"Kafka 协议成为事实标准，实现开始多元化"即有时效感。

**收口**："Kafka 的主线没变——**高吞吐、可重放、生态厚**；4.0 把历史包袱（ZK）卸掉、把消费者协议和存储分层补齐——**选型看的是'你的场景是否吃吞吐与重放'**，不是版本号（5.1.2）。"