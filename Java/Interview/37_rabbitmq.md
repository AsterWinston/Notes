本篇覆盖 RabbitMQ 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：模型与路由、可靠性与消费、集群与高可用、Spring 与运维、选型与实践。RocketMQ 的机制与对比见 13_rocketmq.md，Kafka 的日志/分区模型见 31_kafka.md，会话与 JWT/CSRF 等 Web 安全题见 4_java_web.md、11_spring_security.md（旧文串入本片的同类题已归位删除）。

# 1. 模型与路由

## 1.1 AMQP 核心模型

### 1.1.1 RabbitMQ 的核心组件与消息流转是什么？

**AMQP 0-9-1 模型（五层）**：

```text
Producer → Connection（TCP）→ Channel（虚拟连接）
   → Exchange（交换机，按 RoutingKey + Binding 路由）
       → Queue（队列，存消息）
           → Consumer（Channel 上消费，ACK 后删除）
```

| 组件 | 作用 | 关键点 |
|------|------|--------|
| **vhost** | 顶层逻辑隔离单元 | 独立 Exchange/Queue/Binding/权限；多应用/多环境隔离（`/prod`、`/dev`） |
| **Connection** | TCP 长连接 | 建立成本高（握手/TLS），**一个连接多 Channel 复用** |
| **Channel** | 轻量虚拟连接 | 所有 AMQP 操作在 Channel 上；**非线程安全**（1.1.2） |
| **Exchange** | 路由（不存消息） | 类型 + Binding 决定去哪；**没有匹配队列时消息默认丢弃**（除非 mandatory/备用交换机） |
| **Queue** | 存储与投递 | 持久化/镜像/Quorum/Stream 等类型；消费者 ACK 后才删除 |
| **Binding** | Exchange↔Queue 的规则 | 含 BindingKey（Direct/Topic 匹配用）；Headers 用参数匹配 |
| **RoutingKey** | 消息的路由键 | 生产者发送时指定；与 BindingKey 按 Exchange 类型匹配 |

**消息流转（发布/订阅版）**：生产者 `basicPublish(exchange, routingKey, props, body)` → Exchange 查 Binding → 投递到一个/多个 Queue → 消费者 `basicConsume`（push）或 `basicGet`（pull，少用）→ 业务处理 → `basicAck` → Broker 删除。

**四个易错**：① **Exchange 不存消息**（没路由到队列就丢了，要 `mandatory=true` 收 Return 或配 **Alternate Exchange**）；② **默认交换机**（名称为空，Direct）——**每个 Queue 自动以队列名绑定**，`basicPublish("", "queueName", ...)` 就是"直接发队列"；③ **vhost 隔离**不是"逻辑分组"那么轻——权限/配额/插件都按 vhost 生效；④ **Queue 的声明参数必须一致**（已存在但参数不同 → `406 PRECONDITION_FAILED`，4.2.1）。

### 1.1.2 为什么用 Channel 而不是 TCP 连接？Channel 的线程安全与坑？

**Channel 的本质**：**同一条 TCP 连接上的多路复用"会话"**——每个 Channel 有独立 ID、独立的投递/确认上下文；**创建/销毁几乎零成本**（不握手、不分配内核 socket）。

**为什么不用"一操作一 TCP"**：TCP 连接=fd + 内核缓冲 + 握手/TLS 成本；RabbitMQ 单机连接数有限（数万级），**Channel 单连接可开数万**；且 Channel 共享连接的带宽与心跳，系统调用更少。

**三个必须知道的点**：
1. **Channel 不是线程安全的**：多线程共用一个 Channel 会**交错帧/确认错乱**（`channel closed`/`unknown delivery tag`）——**每个线程用自己的 Channel**（Spring 的 `CachingConnectionFactory` 帮你缓存/隔离）；
2. **Channel 异常会关闭整条连接上的该 Channel**（不影响其他 Channel，但**该 Channel 上未 ACK 的消息会重新入队**）；
3. **Channel ID 有上限**（协议 2 字节，约 65535/连接），且**长生命周期 Channel 不要无限创建**（泄漏）。

**Spring AMQP 的实践**：`CachingConnectionFactory` 缓存 Channel（按模式：CHANNEL/CONNECTION），`RabbitTemplate` 自动借还；`@RabbitListener` 的容器按并发数创建 Channel；**不要在业务代码里 `new Channel` 后不关**。

**易错**：把 Channel 当线程安全（并发发送/确认错乱）；一个 Channel 上混用"发布+消费+事务"（语义混乱）；Channel 关闭后继续用（`AlreadyClosedException`）；连接数打满（每请求新建 Connection）。

## 1.2 交换机与路由

### 1.2.1 四种 Exchange 类型怎么选？默认/备用交换机是什么？

| 类型 | 路由规则 | 典型场景 |
|------|---------|---------|
| **Direct** | RoutingKey **完全等于** BindingKey | 单播/精确路由（订单状态通知、按类型分发） |
| **Fanout** | **忽略 RoutingKey**，广播到所有绑定队列 | 广播（配置刷新、缓存失效、日志分发） |
| **Topic** | `*` 匹配**一个词**、`#` 匹配**零或多个词**（`.` 分隔） | 灵活订阅（`order.created.*`、`order.#`） |
| **Headers** | 按消息 **Headers 的 KV** 匹配（`x-match=all/any`） | 多维度条件路由（少用，性能不如 Topic） |

**默认交换机（Default Exchange）**：名称为空字符串、类型 Direct、**每个 Queue 自动以队列名绑定**——`basicPublish("", "order.queue", ...)` 直接投递，省去声明 Binding（开发方便，但**隐式耦合队列名**，生产更推荐显式 Exchange）。

**备用交换机（Alternate Exchange, AE）**：Exchange 声明时配 `alternate-exchange`——**消息无法路由到任何队列时**转到 AE（通常 Fanout 到"未路由队列"），避免静默丢失；与 `mandatory=true`（Return 给生产者）互补：**AE 兜底存储，Return 通知生产者**。

**选型句**："**精确单播 Direct、广播 Fanout、灵活订阅 Topic、Headers 只在多条件且不在乎性能时用**——能用 Topic 表达就别用 Headers。"

### 1.2.2 如何用 Topic、Headers、Consistent Hash 做灵活路由？

**Topic 的组合（最常用）**：
- `order.created` + Binding `order.*` → 匹配"订单创建"（一个词）；
- `order.#` → 匹配 `order`、`order.created`、`order.created.paid`（零或多词）；
- **注意**：`*` 不匹配零词、`#` 可匹配零词；**词数/顺序要对**（`order.*` 不匹配 `order.created.paid`）。

**Headers 的用法**：Binding 参数 `x-match=all`（全部 KV 匹配）或 `any`（任一匹配）——适合"按业务属性（region/type/priority）组合"；**代价**：Broker 要逐条比较 headers，**吞吐低于 Topic**。

**Consistent Hash Exchange（插件）**：按 **RoutingKey 的哈希**把消息**均匀分布到多个绑定队列**——**同一 RoutingKey 永远去同一队列**（顺序性 + 负载均衡），常用于"按订单号/用户 ID 分区消费"；**注意**：扩缩队列会改变映射（顺序过渡要停写/双写）。

**其他路由能力**：**Exchange-to-Exchange Binding**（交换机级联/聚合）；**Federation**（跨集群的 Exchange/Queue 联邦，3.1.2）；**Shovel**（搬运动态源→目标）。

**易错**：Topic 的 `#` 与 `*` 混用错（`order.#.paid` 合法但语义要想清）；Headers 的 `x-match` 默认是 `all`（不是 any）；Consistent Hash 扩队列破坏"同 key 同队列"（要预案）；用 Fanout 做"按类型过滤"（下游全收再丢，浪费）。

# 2. 可靠性与消费

## 2.1 消息不丢

### 2.1.1 如何保证消息不丢？生产、存储、消费三端各做什么？

**全链路（三端 + 一个"路由"暗坑）**：

| 环节 | 风险 | 手段 |
|------|------|------|
| **生产者→Broker** | 网络断/ Broker 没收到 | **Publisher Confirm**（异步 ack/nack）+ **重试** + **本地消息表/Outbox**（3.1.1 的最终一致） |
| **路由** | Exchange 没匹配到 Queue（静默丢） | **`mandatory=true` + Return 回调**、**Alternate Exchange**、声明检查 |
| **Broker 存储** | 重启/节点挂 | **Exchange/Queue durable** + **消息 DeliveryMode=2** + **Quorum Queue**（Raft 复制） |
| **Broker→消费者** | 消费者崩溃/处理失败 | **手动 ACK（处理完再确认）** + **prefetch** + 失败重试/死信 |
| **消费者业务** | 处理一半崩溃/重复 | **幂等**（2.2.2）+ 本地事务/状态机 |

**四个"必须一起成立"的点（面试官爱追问"这样就一定不丢吗"）**：
1. **Confirm 的语义**：Broker **把消息落到目标队列**（持久消息在 Quorum 队列要多数派提交）后才 ack——**Confirm 成功≠消费者处理成功**，它只保证"到达 Broker 的队列"；
2. **持久化的三层**：Exchange durable、Queue durable、Message persistent——**缺一层，重启就丢**（比如 Queue 非持久，重启后队列没了）；
3. **Quorum Queue 的复制**：Raft 多数派确认才算写入（3.1.1）——**经典队列（4.0 起无镜像）单节点故障会丢**；
4. **消费者 ACK**：**自动 ACK（autoAck=true）在收到消息时就确认**，处理崩溃=丢——**必须手动 ACK + 处理成功后确认**。

**收口**："**不丢 = 生产确认 + 持久化/复制 + 消费确认** 三段闭环；只做一段（比如只开持久化）都只是'心理安慰'。"

### 2.1.2 Confirm、Return、持久化、Quorum 队列怎么配合？

**四个机制的职责（别混）**：

| 机制 | 方向 | 解决什么 |
|------|------|---------|
| **Publisher Confirm** | Broker → 生产者 | 消息**是否到达 Broker/队列**（ack/nack） |
| **Return（mandatory）** | Broker → 生产者 | 消息**无法路由**（没有匹配队列）时退回 |
| **持久化（durable + persistent）** | Broker 内部 | Broker **重启**后消息还在 |
| **Quorum Queue（Raft）** | 集群副本 | **节点故障**时消息不丢（多数派提交） |

**推荐组合（生产标配）**：
```text
① 声明 durable Exchange/Queue（Quorum 类型）
② 消息 DeliveryMode=2；生产者开 Confirm（异步回调，nack/超时重发）
③ mandatory=true + ReturnCallback（未路由告警，不静默丢）
④ 消费者 manual ACK + prefetch（按处理能力）+ 幂等
⑤ 失败重试（有限次）→ 死信队列（2.2.3）
```
**Spring Boot 配置要点**：`spring.rabbitmq.publisher-confirm-type=correlated`、`publisher-returns=true`、`template.mandatory=true`、`listener.simple.acknowledge-mode=manual`、`prefetch`、`retry`（4.1.2）。

**追问"Confirm 了一定不丢吗"**：**不**——它只保证"到队列"；**消费者处理失败/重复**仍要靠 ACK+幂等；**Broker 磁盘坏/全部副本挂**仍可能丢（要备份/跨集群复制）。"金融级"要 Quorum + 同步刷盘 + 对账，不是单一开关。

**易错**：只开 Confirm 不开持久化（重启丢）；`mandatory` 没配 Return 回调（未路由静默丢）；Quorum 队列用 `transient` 消息（不支持/语义不符）；生产者重试无幂等（重复）。

## 2.2 消费语义

### 2.2.1 ACK/NACK/Reject、prefetch、自动 ACK 的坑？

**三种确认**：
- `basicAck(deliveryTag, multiple)`：确认（`multiple=true` 确认该 tag 及之前所有）；
- `basicNack(deliveryTag, multiple, requeue)`：拒绝（可批量；`requeue=true` 重新入队，false 进死信/丢弃）；
- `basicReject(deliveryTag, requeue)`：拒绝单条（不能批量）。

**自动 ACK 的坑**：`autoAck=true` 在**消息发出时**就确认——**业务没处理完/消费者崩溃=消息永久丢失**；只适合"丢了无所谓"（日志/监控）。**生产一律手动 ACK**，且**处理成功后**再 ack。

**prefetch（basicQos）**：限制**每个消费者未 ACK 的消息数**——`prefetch=1` 最稳（处理完才拉下一条，天然负载均衡/顺序），但吞吐低；**太大**=消息堆在消费者内存（OOM/延迟）；**太小**=消费者空转。**经验**：`prefetch ≈ 单条处理耗时 × 目标吞吐 / 消费者数`，再压测调（常见 10~100）。

**Nack/Reject 的坑**：`requeue=true` 会把消息**放回队头**（可能立刻重投给同一消费者 → **无限重试/毒消息**）——要配**重试上限 + 死信**（2.2.3）；`requeue=false` 且没配 DLX = **消息直接丢**；**ACK 的 deliveryTag 必须来自当前 Channel**（跨 Channel/重复 ACK → `unknown delivery tag` 并关闭 Channel）。

**Channel 关闭/连接断**：该 Channel 上**未 ACK 的消息自动重新入队**（redelivered=true）——这是"至少一次"的来源，也是重复消费的来源（2.2.2）。

**易错**：业务异常也 ACK（丢消息）；`requeue=true` 无上限（毒消息打满 CPU）；prefetch 设 0（不限，内存爆）；在异步线程 ACK（Channel 非线程安全/已关闭）。

### 2.2.2 重复消费与顺序性怎么保证？

**重复的四个来源**：生产者重试（Confirm 超时但实际到了）、Broker 重投（未 ACK 连接断）、消费者处理成功但 ACK 丢失、人为重放。**RabbitMQ 是 at-least-once**——**消费端必须幂等**。

**幂等四招（与 13_rocketmq.md 3.2.3 同源）**：**唯一索引**（消息 ID/业务键，与业务同事务）、**状态机**（订单只允许"待支付→已支付"）、**去重表**（`consumed_msg(msg_id)` 唯一）、**Redis SETNX**（性能高、与业务不同事务，只做削量）。**消息 ID 要生产端生成**（UUID/雪花），别用 RabbitMQ 的 deliveryTag（重投会变）。

**顺序性（RabbitMQ 的弱项，要诚实）**：
- **单 Queue + 单 Consumer + prefetch=1 + 手动 ACK**：严格 FIFO；
- **多 Consumer**：轮询分发 + 处理速度不同 = 乱序；**`x-single-active-consumer=true`**（单活跃消费者，故障切换）可保序；
- **Consistent Hash Exchange**：按 key 分到不同队列，**同 key 同队列**（key 内有序、key 间并行）；
- **重试/requeue**：会把消息放回队头/其他消费者，**破坏顺序**——顺序消息的失败要"挂起/人工"，不要 requeue；
- **Stream**：append-only + offset 消费，天然有序（3.1.1），适合"回放/顺序读"。

**收口**："**RabbitMQ 保证'队列内 FIFO 投递'，不保证'多消费者下的处理顺序'**——要顺序就单消费者/单活跃/按 key 分区；要全局顺序，选 RocketMQ 顺序消息或 Kafka 单分区（13_rocketmq.md 2.1.1、31_kafka.md 2.2.2）。"

### 2.2.3 死信队列、重试、延迟队列怎么设计？

**死信的触发条件（四类）**：① `basicReject/Nack` 且 `requeue=false`；② **消息/队列 TTL 过期**；③ **队列达到 `x-max-length`/`x-max-length-bytes`**（挤出队头）；④ **Quorum 队列超过 `x-delivery-limit`**（投递次数上限）。

**死信链路的声明**：队列参数 `x-dead-letter-exchange`（DLX）+ `x-dead-letter-routing-key`（可选）——死信被**重新发布**到 DLX，再路由到 DLQ；**DLQ 自己不要再配 DLX**（否则死信循环）；DLQ 要有人工/自动处理与告警。

**重试的三种做法**：
1. **本地重试**（Spring Retry，`spring.rabbitmq.listener.simple.retry`）：不重投 Broker，简单；进程崩溃则丢"重试上下文"（消息还在 unacked，会重投）；
2. **重试队列（TTL + DLX）**：失败 → 发到 `retry.5s/30s/...`（各自 TTL）→ 过期死信回业务队列——**延迟可配、与业务解耦**；要**有限次数**（消息头记 retryCount）；
3. **延迟插件**（`rabbitmq_delayed_message_exchange`）：Exchange 类型 `x-delayed-message` + 消息头 `x-delay`（毫秒）——**每条消息独立延迟**，最灵活；**注意**：插件把延迟消息存在**交换机内部**（不是队列持久化路径），节点故障/集群切换的可靠性要评估（生产关键路径要压测/兜底）。

**延迟队列的经典坑（TTL+DLQ 方案）**：**队列 TTL 是"队头过期"语义**——队头一条 30 分钟的消息会**堵住后面 5 分钟的消息**（短延迟被长延迟阻塞）；解法：**按延迟档位分队列**（5s/30s/1m 各一个）或**用延迟插件**（2.2.3 方式 3）。

**优先级队列**：`x-max-priority`（推荐 0~10）——**只有堆积时才有意义**（Broker 先投高优先级）；不要设 255（内存/性能）。

**易错**：DLQ 没监控（死信静默堆积）；重试无上限（毒消息循环）；requeue=true 当重试（顺序/风暴）；延迟插件当"绝对可靠"（节点故障评估）；DLX 配成自己（死循环）。

# 3. 集群与高可用

## 3.1 集群形态

### 3.1.1 普通集群、镜像队列、Quorum 队列、Stream 的区别与演进？

| 形态 | 数据 | 高可用 | 现状 |
|------|------|--------|------|
| **普通集群（classic queue）** | 元数据全节点共享；**队列数据只在声明节点** | 节点挂=该队列不可用（无副本） | 仍支持（单节点队列），**4.0 起移除经典镜像** |
| **镜像队列（mirrored）** | Master + Slave 全量复制 | Master 挂自动提升 | **4.0 移除**（历史方案，别再选） |
| **Quorum 队列（3.8+）** | **Raft 多数派复制**（3/5 节点） | Leader 挂自动选主，强一致 | **官方推荐的持久化队列**（默认类型可配） |
| **Stream（3.9+）** | **append-only 日志**，按 offset 消费、可回放 | 复制（Raft）、大吞吐 | 日志/回放/多消费者独立位点；Super Stream 分区 |

**Quorum 队列的要点**：**durable + 复制 + at-least-once**；**不支持** exclusive/非持久/优先级（部分版本）/`x-max-length` 的某些语义（以版本文档为准）；**`x-delivery-limit`** 防毒消息；**性能低于经典队列**（Raft 写多数派），适合"可靠性优先"的业务队列。**4.x 默认队列类型**可通过 policy/vhost 设为 quorum（新项目建议默认 quorum）。

**Stream 的定位**：**非破坏性消费**（读完不删，按 retention 保留）、**多消费者各自 offset**、**可回放**——适合"事件日志/审计/多下游独立消费"；与 Kafka 的模型相近，但生态/工具不同（31_kafka.md 的对比可借鉴）。

**选型句**："**业务队列要可靠 → Quorum；日志/回放/多订阅 → Stream；经典队列只用于临时/低价值**（且单节点无副本）——镜像队列是历史，4.0 后不存在。"

### 3.1.2 集群怎么组、脑裂怎么处理、Federation/Shovel 怎么用？

**集群组成**：节点用 **Erlang cookie**（同一集群必须一致）+ `rabbitmqctl join_cluster`；**元数据（vhost/exchange/queue/binding/user）全节点复制**；**经典队列数据在声明节点**（其他节点只转发）；**Quorum/Stream 数据按 Raft 复制**；**至少 3 节点**（Quorum 多数派/可用性）。

**节点类型**：**Disk 节点**（元数据落盘，至少一个）/ **RAM 节点**（元数据内存，重启丢失，只做性能补充）——生产**全 Disk 或多数 Disk**。

**脑裂（partition）处理**：`cluster_partition_handling`——`pause_minority`（少数派暂停，推荐）、`autoheal`（自动恢复，可能丢消息）、`ignore`（不处理，风险）；**Quorum 队列自己用 Raft 处理分区**（多数派可写），但**元数据/经典队列仍受集群分区策略影响**；**网络分区恢复后要核对队列/消息**。

**Federation / Shovel（跨集群/跨机房）**：**Federation**（Exchange/Queue 联邦）——上游 Exchange 把消息"转发"到下游（按需、跨 WAN、松耦合）；**Shovel**——一个"搬运工"把源队列的消息搬到目标（可靠、可配 prefetch）；**适用**：多机房/多集群/云边协同；**注意**：都不是"同步复制"（异步、可能重复/延迟），业务要幂等。

**易错**：2 节点集群当高可用（Quorum 需要多数派，2 节点挂 1 个就不可写）；RAM 节点当主（重启丢元数据）；跨城组一个集群（延迟/分区，应该 Federation/Shovel）；分区策略选 `ignore` 后不处理（双写/脑裂）。

## 3.2 容量与流控

### 3.2.1 内存/磁盘告警与流控（credit flow）怎么工作？

**两层保护**：
1. **资源告警（alarm）**：**内存**（`vm_memory_high_watermark`，默认 0.4=40% RAM）或**磁盘**（`disk_free_limit`，默认 50MB 或内存大小）触发 → **阻塞所有发布连接**（publisher 被 block，直到恢复）——"消息发不进去/连接卡住"先看 `rabbitmqctl status` 的 alarms；
2. **Credit Flow（信用流控）**：**队列进程**给上游（连接/通道）发"信用"——队列处理不过来时**减少信用**，让发布者减速/阻塞（防止队列进程内存暴涨）——这是**进程级背压**，与 alarm 互补。

**内存去哪了**：消息（未 ACK/堆积）、队列索引、连接/Channel、Erlang 进程；**分页（paging）**：内存压力下把消息**换到磁盘**（性能下降，但避免 OOM）——`rabbitmqctl status` 的 `paging` 指标；**4.0 起 lazy queue 模式被移除**（经典队列的分页行为变化，以文档为准）。

**调优方向**：**prefetch 控制消费者侧**（2.2.1）；**队列长度上限 + DLX**（防无限堆积）；**Quorum 队列的 Raft 开销**（3.1.1）；**连接/Channel 复用**（1.1.2）；**磁盘 IO**（持久化/Quorum 写）；**Erlang 调度器/CPU**。

**易错**：只看队列长度不看 **unacked**（消费者拿了不 ACK，内存涨）；alarm 触发后以为是网络问题（发布被 block）；磁盘满导致**只读/阻塞**（30_linux.md 2.2.3 的磁盘治理）；无队列长度上限（堆积打爆内存）。

### 3.2.2 消息堆积怎么排查与处理？

**先分型（和 13_rocketmq.md 3.3.2 同框架）**：**生产突增**（消费 TPS 正常但入队暴涨）、**消费变慢**（单条处理慢/下游慢/锁）、**消费者挂/不足**（consumers 数下降、unacked 高）、**毒消息**（反复重投，CPU 高但队列不降）。

**排查动作**：管理 UI/`rabbitmqctl list_queues name messages messages_unacknowledged consumers`；看 **publish/deliver/ack 速率**与 **unacked**；消费者线程栈/日志（慢在哪）；下游（DB/HTTP）指标；**Channel/连接数**（是否被流控/阻塞）。

**处理手段（按顺序）**：① **修消费**（优化 SQL/批量/异步/加缓存）；② **加消费者**（prefetch 合理、别超过下游能力）；③ **队列分片**（Consistent Hash/Topic 拆队列，1.2.2）；④ **降级/丢弃**（非关键消息 TTL/限量，2.2.3）；⑤ **迁移/中转**（临时消费者搬到新队列/新集群，类似 RocketMQ 3.3.2 的搬迁）；⑥ **限流生产**（保护下游）。

**预防**：**队列长度/age 告警**（比绝对值看斜率）、**消费端压测**、**prefetch/并发**按处理能力配、**DLQ 监控**、**容量水位**（磁盘/内存 alarm 阈值）。

**易错**：无脑加消费者（下游被打垮）；忽略 unacked（消费者"假活"）；毒消息无限 requeue（2.2.1）；只看队列不看磁盘（消息落盘但 IO 满）。

# 4. Spring 与运维

## 4.1 Spring AMQP

### 4.1.1 Spring AMQP 的核心组件与关键配置？

**四件套**：
- **`CachingConnectionFactory`**：管理 Connection/Channel 缓存（`cache-mode: CHANNEL/CONNECTION`），**不要每次 new**；
- **`RabbitTemplate`**：发送（convertAndSend）、接收（receiveAndConvert）、**Confirm/Return 回调**；
- **`@RabbitListener` + 容器工厂**：消费（`SimpleRabbitListenerContainerFactory`：并发、prefetch、ack 模式、重试）；
- **`RabbitAdmin`**：自动声明 Exchange/Queue/Binding（`@Bean Queue/Exchange/Binding` 或注解）。

**最小配置**：
```yaml
spring:
  rabbitmq:
    host: mq.internal
    port: 5672
    virtual-host: /order
    publisher-confirm-type: correlated   # 开启 Confirm
    publisher-returns: true              # 开启 Return
    template:
      mandatory: true                    # 未路由要退回
    listener:
      simple:
        acknowledge-mode: manual         # 手动 ACK
        prefetch: 20
        concurrency: 4
        max-concurrency: 16
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1s
          multiplier: 2
```

```java
@RabbitListener(queues = "order.created.queue")
public void onCreated(OrderCreatedEvent e, Channel channel,
                      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try {
        orderService.handle(e);              // 业务（幂等）
        channel.basicAck(tag, false);        // 成功才 ACK
    } catch (BizRetryableException ex) {
        channel.basicNack(tag, false, false); // 不重回队列 → 走 DLX/重试
    }
}
```

**消息转换**：默认 SimpleMessageConverter（Java 序列化，**不推荐**）；生产用 **`Jackson2JsonMessageConverter`**（跨语言/可读）；**类名白名单**（`trustedPackages`，防反序列化漏洞）。

**并发与顺序**：`concurrency/max-concurrency` 提高吞吐，但**同一队列多消费者=乱序**（2.2.2）；要顺序就 `concurrency=1` + `prefetch=1` + `x-single-active-consumer`。

**易错**：`acknowledge-mode` 默认 AUTO（Spring 在方法正常返回后 ACK，异常时 nack/requeue）——**AUTO 不是"自动 ACK"**，但异常路径的行为要清楚（默认 requeue 可能毒消息循环）；`@RabbitListener` 方法抛异常 + 重试配置不当（无限重投）；没配 `RabbitAdmin` 但依赖自动声明（队列不存在）。

### 4.1.2 生产者的 Confirm/Return 与消费者的重试/DLQ 怎么接？

**生产者侧（可靠发送）**：
```java
rabbitTemplate.setConfirmCallback((correlation, ack, cause) -> {
    if (!ack) { /* 记录/重发/告警 */ }
});
rabbitTemplate.setReturnsCallback(returned -> {
    /* 未路由：记录 + 告警（mandatory=true 才触发） */
});
```
**要点**：Confirm 是**异步**的（别在发送线程里阻塞等）；**CorrelationData** 带业务 ID（消息表/日志关联）；**nack/超时**要重发（幂等）+ **本地消息表**兜底（DB 事务 + 定时重投，3.1.1）。

**消费者侧（重试 + DLQ）**：
```java
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("order.created.queue")
        .deadLetterExchange("order.dlx")
        .deadLetterRoutingKey("order.failed")
        .build();
}
@Bean
public RetryOperationsInterceptor retryInterceptor(MessageRecoverer recoverer) {
    return RetryInterceptorBuilder.stateless()
        .maxAttempts(3)
        .backOffOptions(1000, 2.0, 10000)
        .recoverer(recoverer)          // 超过次数 → RepublishMessageRecoverer 到 DLX
        .build();
}
```
**三种重试位置**：**容器内重试**（Spring Retry，不重投 Broker，简单）；**重试队列**（TTL+DLX，延迟重投，2.2.3）；**延迟插件**（每条消息自定义延迟）。**超过次数 → DLQ + 告警 + 人工**，不要无限重试。

**易错**：Confirm 回调里做重活（阻塞 IO 线程）；Return 没配 mandatory（不触发）；重试与 requeue 叠加（次数翻倍）；DLQ 没消费者/没告警（死信静默堆积）；重试不幂等（重复副作用）。

## 4.2 运维

### 4.2.1 监控指标、管理插件、常见错误码怎么排查？

**监控四组**：**队列**（depth、unacked、consumers、publish/deliver/ack 速率、age）；**连接/Channel**（数量、blocked、churn）；**资源**（内存/磁盘 alarm、fd、Erlang 进程、GC）；**集群**（分区、Quorum leader、节点健康）。**工具**：Management UI、HTTP API（`/api/queues`）、`rabbitmqctl`（`list_queues`/`status`/`environment`）、**Prometheus 插件**（`/metrics`）+ Grafana、`rabbitmq-diagnostics`。

**常见错误码/现象**：

| 现象 | 根因 | 处置 |
|------|------|------|
| `404 NOT_FOUND` | 队列/Exchange 不存在（未声明/名字错/vhost 错） | 核对声明与 vhost；`RabbitAdmin` 自动声明 |
| `406 PRECONDITION_FAILED` | 声明参数与已存在的不一致（durable/类型/参数） | **删了重建**或统一声明（别在生产随便删） |
| `403 ACCESS_REFUSED` | 用户无 vhost 权限（configure/write/read） | `rabbitmqctl set_permissions` |
| 连接被拒/超时 | 端口/TLS/防火墙/`guest` 只能本机 | 查监听、证书、安全组；建专用用户 |
| 发布被 **blocked** | 内存/磁盘 alarm、credit flow | 扩容/清盘/降速（3.2.1） |
| `unknown delivery tag` | ACK 的 tag 不属于该 Channel/重复 ACK | 检查线程/Channel 复用；别跨线程 ACK |
| Channel 频繁关闭 | 参数错误/大消息/客户端 bug | 看 Broker 日志的 `channel error` |
| 队列不消费 | 消费者挂了/prefetch 0/中毒消息 | 看 consumers/unacked + 消费者日志 |

**排查顺序**：**管理 UI/API 看队列与连接** → **Broker 日志**（`/var/log/rabbitmq`）→ **客户端日志/线程栈** → **系统层**（CPU/内存/磁盘/网络，30_linux.md 2.1–2.3）。

### 4.2.2 版本演进（3.x→4.x）与升级要注意什么？

**关键节点**：**3.8** Quorum 队列；**3.9** Streams；**3.12/3.13** 逐步弃用经典镜像/懒队列；**4.0（2024）**——**移除经典镜像队列（classic mirroring）**、**移除 lazy queue 模式**、要求更新的 Erlang/OTP、AMQP 1.0 与 MQTT 增强、Quorum 成为高可用默认方向；**4.1（2025）** 继续在 Quorum/Stream/K8s Operator 上演进（具体特性以官方 release notes 为准）。

**升级检查清单**：① **镜像队列必须先迁到 Quorum**（4.0 不再支持，升级即失 HA）；② **lazy/transient 队列的语义变化**（消息持久化/分页行为）；③ **插件兼容**（延迟插件/Consistent Hash/Shovel/Federation 的版本匹配）；④ **Erlang/OTP 版本**；⑤ **滚动升级顺序**（Quorum 队列可滚动；经典队列在节点重启时不可用）；⑥ **客户端库兼容**（amqp-client/Spring AMQP 版本）；⑦ **管理 UI/API 变化**（自动化脚本回归）。

**收口**："RabbitMQ 的升级不是'换个包'——**先盘点队列类型（镜像/经典/Quorum/Stream）与插件**，再按'先迁 Quorum、再升 4.x'的顺序做；**升级前做故障演练**（杀节点/分区），否则'升完发现没 HA'是典型事故。"

# 5. 选型与实践

## 5.1 选型

### 5.1.1 RabbitMQ vs Kafka vs RocketMQ 怎么选？

| 维度 | RabbitMQ | RocketMQ | Kafka |
|------|----------|----------|-------|
| 定位 | **低延迟/复杂路由**（AMQP） | **业务消息**（事务/延迟/重试） | **日志/流/大数据**（高吞吐） |
| 吞吐 | 万级（单机） | 十万级 | **百万级** |
| 延迟 | **微秒~毫秒（最低）** | 毫秒 | 毫秒（批处理） |
| 路由 | **Exchange/Binding 最灵活** | Topic/Tag | Topic/Partition |
| 延迟消息 | 插件/TTL+DLX | **原生多级** | 无原生 |
| 事务消息 | 弱（Confirm 不等于事务） | **原生半消息** | 事务（Kafka 内部 EOS） |
| 回溯/堆积 | 消费即删（Stream 可回放） | 支持按时间回溯 | **按 offset 回溯，堆积最强** |
| 顺序 | 单队列/单活跃消费者 | 分区/全局顺序 | 单分区有序 |
| 适用 | 企业内部集成、RPC 异步化、工作队列、复杂路由 | 电商/金融核心链路 | 日志/埋点/CDC/流计算 |

**选型三问**：① **吞吐/堆积量级**（百万级→Kafka）；② **是否需要事务消息/原生延迟/服务端过滤**（要→RocketMQ）；③ **是否需要复杂路由/极低延迟/AMQP 生态**（要→RabbitMQ）。**再加一问**："团队已经在运维哪套？"——多养一套 MQ 的成本常大于技术差异。

### 5.1.2 RabbitMQ 不适合什么场景？

**六个"别用"信号**：
1. **海量日志/埋点/流处理**（百万 TPS、长保留、回放）→ Kafka（31_kafka.md）；
2. **需要按 offset 回溯/重放历史**（经典队列消费即删）→ Kafka/Stream（Stream 可，但生态不如 Kafka）；
3. **原生事务消息/任意延迟/服务端 Tag 过滤** → RocketMQ（13_rocketmq.md）；
4. **超大规模堆积**（数亿消息长期堆积）→ Kafka（磁盘顺序写/分区）；
5. **全局严格顺序 + 高吞吐**（RabbitMQ 单队列单消费者吞吐有限）→ RocketMQ 顺序消息/Kafka 单分区；
6. **多活双写/跨城强一致**（Federation/Shovel 是异步）→ 业务层对账/专门的多活方案。

**反过来，"适合 RabbitMQ"的画像**：**消息量万级、延迟敏感、路由复杂（多条件/多下游）、需要工作队列/优先级/RPC 模式**——它是"业务消息的瑞士军刀"，不是"大数据的管道"。

## 5.2 实践

### 5.2.1 设计一个"订单超时关单"的延迟方案？

**需求**：下单后 30 分钟未支付 → 自动关单 + 释放库存；**要幂等**（支付成功与关单并发）、**可取消**（支付后不再关）、**可观测**。

**四种方案对比**：

| 方案 | 机制 | 优点 | 缺点 |
|------|------|------|------|
| **延迟插件** | `x-delayed-message` + `x-delay=1800000` | 每条独立延迟、实现简单 | 插件依赖；延迟消息存储可靠性要评估（2.2.3） |
| **TTL + DLX（分档）** | 30m 队列 TTL → 死信到关单队列 | 原生、无插件 | **队头阻塞**（不同延迟要分队列）；TTL 精度 |
| **定时扫表** | DB/Redis ZSet 存"到期时间"，定时任务扫描 | 精确、可补偿、可观测 | 轮询延迟/扫表压力（分片/索引） |
| **MQ 延迟消息（RocketMQ）** | 原生延迟级别 | 简单可靠 | 换 MQ 依赖（13_rocketmq.md 2.2.1） |

**推荐组合（生产稳妥）**：**延迟消息触发 + 定时扫表兜底**——延迟消息到点查订单状态（未支付才关），扫表补偿"消息丢/延迟不准"；**幂等**：关单前 `UPDATE order SET status='CLOSED' WHERE id=? AND status='UNPAID'`（影响行数=0 则已支付/已关，直接返回）；**释放库存**用幂等补偿（预扣记录状态机）；**可观测**：延迟消息量、关单成功率、异常单告警。

**易错**：延迟消息当"精确闹钟"（秒级误差/重复投递，处理要幂等）；关单与支付并发（状态机/乐观锁，16_mysql.md 3.2.4）；只靠消息不扫表（丢了就永远不关）；释放库存不幂等（重复回补）。

### 5.2.2 设计一个可靠的消息消费服务（含重试/幂等/监控）？

**六层设计**：
1. **接入**：手动 ACK + 合理 prefetch + 并发（顺序业务用单消费者/单活跃）；消息转换用 JSON + 白名单；
2. **幂等**：**业务唯一键**（消息 ID/订单号）唯一索引或状态机；去重表与业务**同事务**；Redis 只做削量；
3. **事务边界**：**业务 DB 提交后再 ACK**（先 ACK 后提交=丢；先提交后 ACK=重复，靠幂等）；本地事务与"发下一跳消息"用 **Outbox**（3.1.1）；
4. **重试**：有限次数 + 退避（本地/重试队列）；**毒消息**（`x-delivery-limit`/重试上限）→ **DLQ + 告警 + 人工工作台**；
5. **可观测**：每条消息带 **traceId/messageId**（跨生产/消费/下游），指标（消费 TPS、失败率、重试数、DLQ 深度、P99 处理时长），日志结构化；
6. **运维**：**优雅停机**（停止拉取 → 处理完在途 → 关闭，K8s preStop/terminationGracePeriod，32_kubernetes.md 4.2.2）、**容量/堆积告警**（3.2.2）、**重放工具**（DLQ/历史消息按条件重投，幂等前提）。

**答题收口**："**可靠消费 = '至少一次投递 + 幂等处理 + 有限重试 + 死信兜底 + 可观测'**——MQ 只负责'送到'，'处理对'是消费服务的工程责任（13_rocketmq.md 3.2.3 的同一条结论）。"