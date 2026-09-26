# 1. 知识点
- **NameServer 核心架构**：RocketMQ 的注册中心，无状态节点，所有 Broker 向其注册 Topic 路由信息，Producer 和 Consumer 从 NameServer 获取路由。NameServer 之间不通信，集群部署多台相互独立。
- **消息模型（Topic / Queue / Message）**：Topic 是消息的逻辑分类标签，Queue 是物理分区（分区内严格有序），Message 是实际传输的消息体，Broker 存储消息，NameServer 管理路由。
- **普通消息**：最基础的消息类型，Producer 发送后 Consumer 异步消费，不保证顺序。
- **顺序消息**：全局顺序（单 Queue 实现，所有消息发往同一 Queue）和局部顺序（同一 businessKey 的消息发往同一 Queue，通过 MessageQueueSelector 实现）。
- **延时消息**：支持 18 个延迟级别（1s/5s/10s/30s/1m/2m/3m/4m/5m/6m/7m/8m/9m/10m/20m/30m/1h/2h），通过 `DelayTimeLevel` 设置，Broker 根据延迟级别暂存消息到期后投递。
- **事务消息**：采用半消息（Half Message）机制，Producer 先发送 prepare 消息到 HALF_TOPIC，执行本地事务后根据结果 Commit 或 Rollback。Broker 定期回查 `TransactionListener.checkLocalTransaction()` 获取本地事务最终状态。
- **TransactionListener**：包含 `executeLocalTransaction()`（执行本地事务）和 `checkLocalTransaction()`（回查本地事务结果），回查次数有限（默认 15 次），超时未确认的消息触发回查。
- **同步发送**：Producer 发送消息后阻塞等待 Broker 返回结果，可靠性高但延迟较大。
- **异步发送**：Producer 发送消息后立即返回，通过回调函数 `SendCallback` 处理结果，吞吐高。
- **单向发送（OneWay）**：Producer 发送消息后不等待任何响应，可靠性最低但吞吐最高，适用于日志等场景。
- **消息模板（MessageBuilder）**：RocketMQ 提供的消息构建工具，链式调用设置 Topic、Tag、Key、Body 等属性。
- **集群消费（ClusterMessageModel）**：默认模式，同一 Consumer Group 内的消费者共享消费消息，一条消息只被组内一个消费者消费（点对点）。
- **广播消费（Broadcasting）**：同一 Consumer Group 内所有消费者都消费全量消息，通过 `MessageModel.BROADCASTING` 设置。
- **Consumer 类型**：PullConsumer（拉模式，主动从 Broker 拉消息）和 PushConsumer（推模式，底层封装 Pull 实现长轮询，Broker 有新消息时主动推送）。
- **消费进度 Offset**：由 `OffsetStore` 管理，Broadcast 模式存本地文件，Cluster 模式存 Broker，Consumer 重启后可从上次位置继续消费。
- **消费组负载均衡**：同一 Consumer Group 内的 Queue 分配给组内 Consumer，策略包括平均分配、轮询、AllocateMessageQueueAveragely 等。扩缩容触发 Rebalance（20s 无人认领的 Queue 重新分配）。
- **消息重试**：消费失败后 `retryTimes +1`，消息发回重试队列 `%RETRY%+consumerGroup`，延迟级别随 `reconsumeTimes` 指数增加。
- **死信队列（DLQ）**：重试超过最大次数（默认 16 次）的消息进入死信队列 `%DLQ%+consumerGroup`，需人工处理。
- **消息过滤**：支持 TAG 过滤（Producer 设置 tags，Consumer 通过 `MessageSelector.byTag()` 筛选）、SQL92 属性过滤（需在 Broker 配置 `enablePropertyFilter=true`）、Filter Server 模式。
- **事务消息两阶段提交+回查**：阶段一发送半消息（对 Consumer 不可见），阶段二执行本地事务后提交或回滚。超时未确认则 Broker 回查 Producer 获取本地事务结果。
- **消息轨迹（MsgTrace）**：通过 `msgTraceEnable=true` 开启，记录生产时间、存储时间、消费时间等关键节点信息，写入 `RMQ_SYS_TRACE_TOPIC`。
- **高可用（Dledger Raft）**：基于 Raft 协议的主从自动故障转移，替代传统 Master-Slave 手动切换，保证数据强一致性。
- **主从同步**：Master 将消息同步到 Slave，支持同步双写和异步复制两种方式。
- **同步刷盘**：消息写入 CommitLog 后强制刷盘到磁盘再返回成功，可靠性高但性能低。
- **异步刷盘**：消息写入 PageCache 后立即返回成功，由操作系统异步刷盘，性能高但可能丢失消息。
- **NameServer 集群**：无状态、多节点独立部署，Broker 向所有 NameServer 注册，Producer/Consumer 随机连接一个 NameServer 获取路由。
- **消息幂等**：通过业务唯一标识 `msgKey` 或 `BizId` 做去重，可用数据库唯一索引或 Redis Key 判断消息是否已消费。
- **RocketMQ vs Kafka**：RocketMQ 原生支持 Tag 过滤、消息优先级、延时消息、事务消息；消息堆积能力更强；国内生态更好。Kafka 基于日志追加，适合大数据流处理。
- **存储模型**：CommitLog 顺序写（所有 Topic 的消息顺序写入同一 CommitLog 文件）+ ConsumeQueue（按 Topic-Queue 建立索引，存储消息在 CommitLog 中的偏移量）。
- **零拷贝**：Consumer 拉取消息时通过 MappedByteBuffer 和 `FileRegion.transferTo()` 实现零拷贝，数据直接从 PageCache 传输到网卡。
- **消息堆积**：RocketMQ 磁盘容量大时堆积能力强，消息过期默认 72 小时，支持异步批量拉取处理堆积。
- **ACL 权限控制**：支持对 Topic、Consumer Group 的访问控制，通过 `plain_acl.yml` 配置文件配置 AccessKey/SecretKey。
- **批量发送**：支持批量消息发送，将多条消息打包成一个请求发送，提高吞吐量。

# 2. 面试题

## 2.1 RocketMQ 的 NameServer 和 ZooKeeper 有什么区别？
NameServer 是无状态的轻量级注册中心，节点之间不通信，所有 Broker 向每个 NameServer 注册路由信息。ZooKeeper 是强一致性的 CP 系统，节点间通过 ZAB 协议同步。NameServer 设计更简单，CAP 倾向于 AP（高可用），即使部分节点不可用也不影响整体服务。RocketMQ 选择自研 NameServer 而非 ZooKeeper 的原因是为了降低依赖和运维复杂度，且消息队列场景对一致性要求不如协调服务严格。

## 2.2 RocketMQ 如何实现顺序消息？
全局顺序消息：所有消息发送到同一个 Queue，消费者单线程顺序消费。局部顺序消息：将同一业务标识（如订单 ID）的消息通过 `MessageQueueSelector` 哈希到同一个 Queue，Consumer 使用 `MessageListenerOrderly` 单线程消费该 Queue，加锁保证同一 Queue 同一时刻只有一个线程消费，从而实现局部有序。

## 2.3 RocketMQ 事务消息的实现原理是什么？
采用两阶段提交+回查机制：
1. Producer 发送半消息到 `RMQ_SYS_TRANS_HALF_TOPIC`，此时消息对 Consumer 不可见。
2. Broker 返回半消息发送成功，Producer 执行本地事务。
3. Producer 根据本地事务结果向 Broker 发送 Commit 或 Rollback 请求。
4. 若 Broker 长时间未收到确认（如 Producer 宕机），则定时回查 Producer 的 `checkLocalTransaction()` 方法获取本地事务状态。
5. 回查有次数限制（默认 15 次），超过次数仍未确认则标记为 Rollback。

## 2.4 RocketMQ 的延时消息是如何实现的？
Producer 发送消息时设置 `DelayTimeLevel`（1~18 对应不同延迟时间），Broker 收到延时消息后将其暂存到内部延时主题 `SCHEDULE_TOPIC_XXXX` 对应延迟级别的 Queue 中。Broker 内部有一个定时任务（TimerTask）扫描到期消息，将其从延时 Queue 取出投递到原始目标 Topic 供 Consumer 消费。延迟级别与时间的映射关系是预设的，不支持任意时间。

## 2.5 RocketMQ 的集群消费和广播消费有什么区别？
集群消费（默认）：同一 Consumer Group 内所有消费者分摊消费 Topic 的所有消息，每条消息只被组内一个消费者处理，消费进度保存在 Broker。广播消费：同一 Consumer Group 内每个消费者都消费全量消息，消费进度保存在消费者本地文件。通过 `MessageModel.CLUSTERING` 或 `MessageModel.BROADCASTING` 设置。

## 2.6 消息消费失败后 RocketMQ 如何处理重试？
Consumer 消费失败返回 `RECONSUME_LATER` 后，Broker 将消息的 `retryTimes +1`，并把消息发回重试队列 `%RETRY%+ConsumerGroup`。重试消息的延迟时间随 `reconsumeTimes` 指数递增（10s→30s→1m→2m→3m→...→2h）。当重试次数超过最大限制（默认 16 次），消息进入死信队列 `%DLQ%+ConsumerGroup`。

## 2.7 RocketMQ 如何保证消息幂等？
1. 业务层面：使用全局唯一业务标识（如订单号 + 操作类型）作为 `msgKey`。
2. 数据库去重表：消费前将 `msgKey` 插入去重表（唯一索引），插入成功则消费，失败则跳过。
3. Redis 去重：以 `msgKey` 为 Key 存入 Redis，通过 `SETNX` 判断是否已处理，设置合理过期时间。
4. RocketMQ 自身不保证幂等，因为重试、网络等原因可能导致消息重复投递。

## 2.8 RocketMQ 的 PushConsumer 底层实现原理是什么？
PushConsumer 底层实际上是 Pull 模式。Consumer 内部有一个 `PullRequestService` 线程，向 Broker 发起长轮询请求（默认超时 30s），Broker 持有请求直到有新消息或超时。收到消息后交由 `ConsumeMessageService` 线程池消费。长轮询减少了空轮询的网络开销，同时保持了消息的实时性。

## 2.9 RocketMQ 和 Kafka 的对比？
| 维度 | RocketMQ | Kafka |
|------|----------|-------|
| 存储模型 | CommitLog + ConsumeQueue | Partition + Segment 文件 |
| 消息过滤 | 原生支持 Tag/SQL92 | 不支持，需客户端过滤 |
| 延时消息 | 原生支持 18 个级别 | 需自行实现 |
| 事务消息 | 原生支持半消息 | 支持（幂等+事务） |
| 顺序消息 | 局部顺序 + 全局顺序 | 分区内有序 |
| 消息堆积 | 磁盘容量决定，堆积能力强 | 同样磁盘容量决定 |
| 依赖组件 | NameServer（自研） | ZooKeeper / KRaft |
| 适用场景 | 业务消息、电商、金融 | 日志收集、流处理、大数据管道 |
| 生态 | 国内生态好，阿里系 | 国际生态好，大数据集成 |
| 语言 | Java | Scala/Java |

## 2.10 什么是死信队列？如何处理死信？
死信队列（DLQ）是 RocketMQ 中存放无法被正常消费的消息的特殊队列，命名格式为 `%DLQ%+ConsumerGroup`。消息在重试超过最大次数（默认 16 次）后进入死信队列。处理方式：1) 通过控制台或 API 查看死信消息内容；2) 修复消费逻辑后，通过控制台重新发送死信消息回原 Topic；3) 配置告警机制，监控死信队列消息数量。

## 2.11 RocketMQ 的同步刷盘和异步刷盘有什么区别？
同步刷盘：消息写入 CommitLog 后，调用 `force()` 强制将数据从 PageCache 刷入磁盘，成功后才返回 Producer ACK。保证消息不丢失但吞吐低。异步刷盘：消息写入 PageCache 后立即返回 ACK，由操作系统异步刷盘（默认 500ms 一次）。吞吐高但 Broker 宕机可能丢失最近 500ms 的消息。通过 `flushDiskType` 参数配置。

## 2.12 RocketMQ 的负载均衡机制是怎样的？
Producer 端负载均衡：发送消息时通过 `MessageQueueSelector` 选择目标 Queue，默认轮询。Consumer 端负载均衡：Consumer Group 内的消费者实例定期（默认 20s）从 Broker 获取当前活跃的 Queue 和 Consumer 列表，通过分配策略（默认平均分配 `AllocateMessageQueueAveragely`）重新分配 Queue。当消费者加入或离开时触发 Rebalance，20s 无人认领的 Queue 会重新分配。

## 2.13 RocketMQ 如何实现高可用？
1. **主从架构**：Master 负责读写，Slave 只读（Consumer 可消费 Slave），Master 宕机后 Producer 无法写入但 Consumer 仍可从 Slave 消费。
2. **Dledger（Raft）**：基于 Raft 协议的主从自动故障转移，Master 宕机后自动选举新 Master，实现自动切换。
3. **NameServer 集群**：多节点无状态，任一不可用不影响整体。

## 2.14 消息轨迹（MsgTrace）记录哪些信息？如何使用？
消息轨迹记录一条消息从生产到消费的全链路关键节点：生产时间、存储时间、消费时间、消费状态、重试次数等。使用方式：
- 生产者开启：`producer.setMsgTraceEnable(true)`。
- 消费者开启：`consumer.setMsgTraceEnable(true)`。
- 轨迹数据存储在系统 Topic `RMQ_SYS_TRACE_TOPIC` 中，需部署独立的 Trace 消费者处理。

## 2.15 RocketMQ 的零拷贝是如何实现的？
RocketMQ 使用 Java NIO 的 `MappedByteBuffer`（内存映射文件）将 CommitLog 和 ConsumeQueue 文件映射到虚拟内存，读写操作直接操作内存，避免用户态和内核态的拷贝。Consumer 拉取消息时，通过 `FileRegion.transferTo()` 直接将 PageCache 中的数据通过 DMA 传输到网卡，实现零拷贝（数据不经过用户态）。

## 2.16 RocketMQ 的存储模型（CommitLog + ConsumeQueue）是什么？
RocketMQ 采用**单一 CommitLog + 多个 ConsumeQueue** 的存储架构：

**CommitLog**（消息主体）：
- **全局唯一**：所有 Topic 的消息按写入顺序追加到同一个 CommitLog 文件（顺序写，超高吞吐）。
- 文件大小默认 1GB，写满后创建新文件。
- 每条消息包含：消息长度、MAGIC CODE、主题/队列 ID、消息体、存储时间等。

**ConsumeQueue**（消费索引）：
- **按 Topic + QueueId 组织**，每个 ConsumeQueue 对应一个 Topic 的一个 Queue。
- 每条 ConsumeQueue 记录固定 20 字节（Offset + Size + Tags HashCode）。
- Consumer 先查 ConsumeQueue 获取 Offset → 再去 CommitLog 按 Offset 读取完整消息。
- 作用：相当于 CommitLog 的**稀疏索引**，避免 Consumer 全量扫描 CommitLog。

**存储流程**：
1. Producer 发送消息 → Broker 将消息追加写入 CommitLog → 返回写好的 Offset。
2. 异步构建线程（ReputMessageService）解析 CommitLog，将 Offset/Size/TagHashCode 分发写入各个 ConsumeQueue。
3. Consumer 拉取消息时，先定位 ConsumeQueue 获取 Offset，再去 CommitLog 随机读取。

**优势**：单一日志文件配合顺序写入，最大化磁盘吞吐（类似 Kafka 的 Partition）；ConsumeQueue 轻量级索引，写入快，查询快。

## 2.17 RocketMQ 消费端长轮询机制
RocketMQ Consumer 采用**长轮询（Long Polling）**实现类似 Push 的消费效果：

**PushConsumer 的真面目**：RocketMQ 的 PushConsumer 底层是**长轮询 Pull 模式**（push 只是上层封装，底层每次都发 Pull 请求）。

**长轮询流程**：
1. Consumer 发送 Pull 请求到 Broker（带超时参数）。
2. Broker 收到 Pull 请求：
   - 如果目标 Queue 有可用消息（且未达到 Consumer 堆积上限）→ 立即返回消息。
   - 如果**没有可用消息** → Broker 通过 `PullRequestHoldService` 挂起该请求，暂时不响应。
3. Broker 持续等待：当有新消息到达或被挂起的请求超时（默认 15s）时，Broker 给 Consumer 返回响应（消息或空）。
4. Consumer 收到响应 → 处理消息 → 继续发下一次 Pull 请求。

**长轮询 vs 短轮询**：
| 维度 | 短轮询 | 长轮询（RocketMQ） |
|------|-------|-------------------|
| 请求方式 | 客户端定时每隔 N 秒发请求 | 客户端发一次请求，服务器"hold"住直到有数据或超时 |
| 延时 | 最多 N 秒延迟 | 几乎实时 |
| 资源开销 | 大量无效请求浪费带宽 | 有效利用网络，无效请求少 |
| 实现 | 客户端轮询 | Broker 端暂挂请求（HoldService） |

长轮询巧妙在客户端实现了"Push 推送"的实时性，同时保持了 Pull 模式的可控性（消费速率由 Consumer 控制）。

## 2.18 RocketMQ 5.x 有哪些重大变化？
| 特性 | 说明 |
|------|------|
| **Proxy 代理模式** | 新增弹性 Proxy 组件，Broker 不直接暴露给客户端，架构分为 gRPC 代理层 + Broker 模块。支持多协议（Remoting/gRPC）多语言客户端，灰度切换流量 |
| **Pop 消费模式** | 替代传统长轮询 Pull，客户端一个 Pop 请求即可获取多个 Topic/多个 Queue 的消息，简化消费逻辑，支持 Serverless 弹性扩缩 |
| **轻量级 SDK + gRPC 协议** | 客户端可以使用 gRPC 调用（替代 Heavy Remoting），支持 Go/Python/C++ 等非 Java 语言原生接入 |
| **Container/Slave Acting Master** | Slave 可以承担读取请求，减轻 Master 负载 |
| **Metrics 体系** | 基于 OpenTelemetry 的统一指标输出，可对接 Prometheus |
| **ACL 增强** | 更精细的权限控制，支持动态更新 |
| **批量消息的优化** | 更好的批量发送和吞吐 |

**Proxy 模式架构**：
```
Producer/Consumer → [gRPC/Remoting] → Proxy（无状态） → Broker
```

Proxy 层对 Broker 屏蔽真实客户端地址，可实现无损升级、限流分组、租户隔离。5.x 是 RocketMQ 架构上的最大一次升级，目标是对齐云原生的弹性、多语言和高可用需求。

## 2.19 消息大小限制是什么？超大消息如何处理？
**默认限制**：RocketMQ 对所有消息（无论单向、同步、异步）限制**最大 4MB**（`maxMessageSize` 参数可配置，默认 4MB）。

**超大消息处理方案**：
1. **消息属性绕过**（消息本身小）：Java 对象无需完整传入消息体，只需传关键字段 ID，Consumer 收到后去数据库/Redis 查询完整数据。这是最推荐的方式。
2. **压缩后发送**：针对文本类消息，发送前用 GZIP/ZSTD 压缩，接收端解压。可压缩 5~10 倍。
3. **文件存储 + 引用传参**：将大文件存入 MinIO/OSS/COS，消息体中只传下载 URL + 文件元信息。
4. **拆分 + 聚合**：将大消息拆为多个小消息（共享 MessageId），Consumer 端判断是否收齐后聚合处理。需自行维护拆分的批次 ID、重组逻辑，通常不推荐。

**注意**：超大消息不应被放入 MQ——MQ 的设计原意是传递**事件的 ID 和上下文**，实际数据应该存在于数据库或文件存储中。Kafka 和 RabbitMQ 也有类似限制（Kafka 默认 1MB，RabbitMQ 无硬限制但大消息严重影响性能）。
