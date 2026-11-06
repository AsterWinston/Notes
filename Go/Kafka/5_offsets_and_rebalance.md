Offset 是 Kafka 消费者唯一需要持久化的状态。它看起来只是一个数字，却同时决定了三件事：消息会不会重复、会不会丢、消费积压有多少。再加上 rebalance 会在任意时刻把分区从一个人手里转给另一个人，offset 的提交时机就成了可靠性的分水岭。本篇把 offset 语义、自动/手动提交、rebalance 过程和 lag 排查串起来讲。

# 1. Offset 的语义

## 1.1 三种 offset

| 名称 | 位置 | 含义 |
|------|------|------|
| Log End Offset（LEO） | Broker | 分区日志末尾的下一个位置 |
| High Watermark（HW） | Broker | 消费者可见的最大位置 |
| Current Position | Consumer | 下一条要拉取的位置 |
| Committed Offset | `__consumer_offsets` | 消费组已确认处理到的位置 |

一条消息的 offset 是它在分区里的绝对编号。消费者拉取到 offset=100 的消息后，本地位置推进到 101；提交 offset=101，表示"100 及之前的消息都处理完了"。

```text
分区日志：
offset:  95  96  97  98  99  100  101  102
         ├──────── 已提交 ────────┤
                                  ▲
                          committed offset = 101
                                  当前拉到 100，position = 101
```

## 1.2 __consumer_offsets

消费组的 offset 保存在内部 Topic `__consumer_offsets` 里。消息 key 是 `(group, topic, partition)`，value 是 offset、metadata、提交时间等信息；默认 50 个分区，副本数由 `offsets.topic.replication.factor` 控制。

★ **关键认知**：offset 记录和消息本身是**两套生命周期**。消息按 `retention.ms` 过期删除，offset 按 `offsets.retention.minutes` 过期（Kafka 3.x 默认 7 天）。如果消费组停了很久，offset 记录先过期，再次启动时就会走 `auto.offset.reset`（Java 客户端）或 `StartOffset`（kafka-go），可能从最早或最新重新开始。

## 1.3 提交的是"下一条"

这是最容易写错的地方：提交的 offset 是**下一条要读的位置**，不是"最后处理的那条"。处理 offset=100 的消息后应提交 101，表示 100 及之前都已完成。kafka-go 的 `CommitMessages(ctx, m)` 会自动按 `m.Offset + 1` 提交；如果你自己拼 `kafka.Message` 或直接调底层 API，忘记加一就会导致**重复消费最后一条**。

⚠ **关键认知**：把 offset 理解成"已经处理到的位置"而不是"下一条"，在重置 offset 时很容易差一。`--to-offset 100` 表示从 100 开始读，不是读到 100 为止。

# 2. 自动提交与手动提交

## 2.1 自动提交

Java 客户端的自动提交由 `enable.auto.commit=true` 和 `auto.commit.interval.ms` 控制。它并不是"处理完一条就提交"，而是在 `poll()` 时把**上一轮 poll 返回的最后位置**提交上去。也就是说，消息被拉取后、业务还没处理完，offset 可能已经提交了。kafka-go 的 `ReadMessage` 也是类似行为：返回消息后按 `CommitInterval` 自动提交，处理函数崩溃时消息可能不会再投递。

## 2.2 手动提交

要做"处理成功后再提交"，必须手动控制：

```go
for {
    m, err := r.FetchMessage(ctx) // 只拉取
    if err != nil {
        break
    }
    if err := process(m); err != nil {
        continue // 不提交，下次重新拉取
    }
    if err := r.CommitMessages(ctx, m); err != nil {
        log.Printf("提交失败: %v", err) // 提交失败会导致重复消费
    }
}
```

自动提交和手动提交的取舍：

| 维度 | 自动提交 | 手动提交 |
|------|----------|----------|
| 代码复杂度 | 低 | 中 |
| 可靠性 | 可能丢消息 | 至少一次 |
| 重复概率 | 低 | 较高 |
| 适用 | 日志、可丢的指标 | 订单、支付、业务事件 |

## 2.3 提交时机与投递语义

```text
至多一次（at-most-once）：
  拉取 → 提交 offset → 处理
  崩溃时：消息已标记完成，但业务没做 → 丢

至少一次（at-least-once）：
  拉取 → 处理 → 提交 offset
  崩溃时：业务可能已做，offset 没提交 → 重复

精确一次（exactly-once）：
  处理结果和 offset 在同一个事务里提交
  崩溃时：要么都成功，要么都回滚
```

★ **关键认知**：提交时机是投递语义的**唯一决定因素**，比 `acks` 更直接。`acks=all` 只保证消息进了 Kafka，不保证业务处理了；提交 offset 才代表业务进度。两者要分开看。

## 2.4 提交失败与重试

提交 offset 也可能失败：Coordinator 正在切换、组在 rebalance、网络超时。失败后不要静默忽略，否则下次会重复消费。

```go
for attempt := 0; attempt < 3; attempt++ {
    err := r.CommitMessages(ctx, m)
    if err == nil {
        break
    }
    if errors.Is(err, context.Canceled) {
        break // 退出中，交给幂等兜底
    }
    time.Sleep(time.Duration(attempt+1) * 100 * time.Millisecond)
}
```

⚠ **关键认知**：提交失败后继续处理下一条，会让"未提交范围"越来越大。一旦进程崩溃，这一大段消息都会重放。提交失败应该降级为"停止拉取、先重试提交"，或者接受重复并确保幂等。

# 3. Rebalance 深入

## 3.1 触发条件

```text
B 崩溃 / 退出 / 心跳超时
  → Coordinator 进入 PreparingRebalance
  → 所有成员 JoinGroup，leader 重新计算分配
  → SyncGroup 下发结果，A 接管 P2
```

常见触发原因：

- 消费者加入或离开（扩容、缩容、滚动发布）。
- 心跳超时，成员被判定死亡。
- 处理时间超过 `max.poll.interval.ms`（Java 客户端）。
- Topic 分区数变化。
- 订阅的 Topic 列表变化。
- Coordinator 故障转移。

## 3.2 Eager 与 Cooperative

| 协议 | 行为 | 影响 |
|------|------|------|
| Eager | 所有成员放弃全部分区，再重新分配 | 整组短暂停消费 |
| Cooperative | 只撤销需要迁移的分区 | 大部分分区持续消费 |

Eager 的 rebalance 是"全停全启"：即使只有一个消费者加入，所有人也要先交出分区。Cooperative 把一次大 rebalance 拆成多轮小调整，减少停顿，但实现更复杂，需要客户端支持。

★ **关键认知**：rebalance 不是"消息重新分配"，而是"分区负责人重新分配"。消息一直躺在分区日志里，不会因为 rebalance 移动。新负责人从**已提交 offset** 继续读，所以未提交的部分会被重复处理。

## 3.3 Static Membership

滚动发布时，每个实例重启都会触发一次 rebalance。静态成员（`group.instance.id`）在 `session.timeout.ms` 窗口内保留身份，窗口内回来不会触发；kafka-go 对此支持有限，Go 服务更实际的做法是放慢发布节奏、调大 `RebalanceTimeout`，并确保消费逻辑能容忍重复。

## 3.4 Rebalance 期间的消息

```text
A 处理 offset=100 时 P0 被转给 B
  → A 没来得及提交
  → B 从 committed offset=100 开始读
  → offset=100 被重新处理，产生重复
```

三种典型情况：

- **提交成功、处理成功**：新负责人从下一条开始，不重复。
- **处理成功、提交失败**：新负责人重读，重复。
- **处理到一半被撤销**：消息未提交，新负责人重读，重复。

⚠ **关键认知**：Kafka 不会等消费者处理完再 rebalance。Eager 协议下，分区被撤销后继续处理也没有意义，因为提交会失败。消费逻辑要能处理"同一消息被处理两次"，不能假设"分到我手里就只处理一次"。

# 4. 重复消费与丢失

## 4.1 重复的根因

| 根因 | 说明 |
|------|------|
| 处理成功、提交失败 | 最常见，尤其是 rebalance 或网络抖动 |
| 进程崩溃 | 提交前的处理全部重放 |
| 生产者重试 | 超时后重发，Broker 写入两条 |
| 自动提交早于处理 | 拉取后立即提交，处理失败也重读不到 |
| offset 重置 | 人为回退到更早位置 |
| 多消费组重复 | 不同组各自消费全量，不是"重复"而是设计 |

## 4.2 丢失的根因

| 根因 | 说明 |
|------|------|
| 先提交后处理 | 崩溃时消息已标记完成 |
| 异步处理未确认就提交 | worker 失败但 offset 已推进 |
| `auto.offset.reset=latest` | 新组跳过历史消息 |
| retention 过期 | 消息还没消费就被删除 |
| `acks=1` + leader 崩溃 | 生产端写入丢失 |
| 非 ISR 选举 | 落后副本当选 leader，丢数据 |

## 4.3 排查路径

```text
消息少了？
  ├── 生产端：acks 是否 all？ISR 是否健康？
  ├── 保留：消息是否已过期删除？
  ├── 消费端：是否先提交后处理？
  ├── 起点：auto.offset.reset 是否 latest？
  └── 过滤：业务代码是否静默丢弃？

消息多了？
  ├── 是否处理成功但提交失败？
  ├── 是否发生过 rebalance？
  ├── 生产者是否超时重试？
  └── 是否人为重置过 offset？
```

★ **关键认知**：排查重复时，先看**提交日志**和**rebalance 日志**，再看业务代码。绝大多数重复不是 Kafka "发了两条"，而是同一条被消费了两次。生产端的重复通常伴随超时和重试记录，消费端的重复通常伴随 rebalance 或提交失败。

# 5. 消费滞后

## 5.1 lag 的计算

```text
lag = 分区 LEO - 消费组已提交 offset
```

Broker 端的 lag 基于**已提交 offset**。如果消费者拉取了但还没提交，Broker 看到的 lag 会偏大；消费者本地的 lag 基于 current position，会更接近真实进度。

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-worker
```

输出中的关键列：

| 列 | 含义 |
|----|------|
| `CURRENT-OFFSET` | 已提交的 offset |
| `LOG-END-OFFSET` | 分区当前 LEO |
| `LAG` | 两者之差 |
| `CONSUMER-ID` | 当前负责该分区的消费者 |

## 5.2 lag 增长的排查

| 现象 | 可能原因 | 排查 |
|------|----------|------|
| 所有分区 lag 均匀上涨 | 生产量超过消费能力 | 看生产 QPS、消费耗时 |
| 单个分区 lag 高 | 热点 key、分区倾斜 | 看 key 分布 |
| lag 高但消费者 CPU 低 | 卡在下游、锁、GC | 看 pprof、下游延迟 |
| lag 周期性归零又上涨 | 批量任务、定时高峰 | 看业务流量模型 |
| lag 突然暴涨 | 消费者掉线、rebalance | 看成员数和 rebalance 日志 |
| 提交 offset 不前进 | 处理失败被跳过、提交报错 | 看错误日志 |

## 5.3 处理积压

处理积压的顺序应该是：

1. **确认瓶颈**：是消费逻辑慢、下游慢，还是分区不够。
2. **加消费者**：最多加到分区数，超过部分会空闲。
3. **加分区**：能提高并行度，但会改变 key 的分区映射，影响顺序性。
4. **优化处理**：批量写数据库、并发调用下游、减少同步锁。
5. **限流生产**：极端情况下先保住核心链路。
6. **重置 offset**：只用于"确认可以跳过"的场景，且必须先停掉所有消费者。

```bash
  # 停掉所有消费者后，把 offset 重置到最早
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-worker --topic orders \
  --reset-offsets --to-earliest --execute
```

⚠ **关键认知**：重置 offset 是不可逆操作。`--to-latest` 会跳过所有积压，`--to-earliest` 会重放所有历史。执行前必须确认消费组没有活跃成员，否则命令会被拒绝，或者和正在运行的消费者互相覆盖。

★ **关键认知**：lag 只是症状，不是根因。持续上涨的 lag 说明"消费速率 < 生产速率"，要么提高消费速率，要么降低生产速率。靠调大 `MaxBytes` 或增加消费者，只能在分区数和下游能力允许的范围内起效。

# 6. 常见错误与坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 重启后重复消费 | 提交失败或 rebalance | 业务幂等 |
| 重启后漏消息 | 先提交后处理 | 改成处理完再提交 |
| 新消费组读不到历史 | `auto.offset.reset=latest` | 设 earliest 或显式重置 |
| 提交报 `RebalanceInProgress` | 组正在重新分配 | 重试，必要时暂停拉取 |
| offset 记录消失 | `offsets.retention.minutes` 过期 | 重置 offset 或重建组 |
| lag 一直很高 | 分区数不够/消费慢 | 扩分区、优化处理 |
| 手动重置失败 | 还有活跃消费者 | 先停消费再执行 |
| 同 key 顺序乱 | rebalance 后换了负责人 | 幂等 + 同 key 同分区 |

⚠ **关键认知**：`auto.offset.reset` 只在"没有已提交 offset"时生效。组里已有提交记录，改这个参数不会让消费从头开始；要重置必须用 `kafka-consumer-groups.sh --reset-offsets`。

# 7. 动手练习

1. 用 `FetchMessage` 处理消息但不提交，重启消费者，观察同一条消息是否重复出现；再用 `ReadMessage` 对比行为。
2. 起两个消费者，停掉一个，观察 rebalance 前后 `kafka-consumer-groups.sh --describe` 里 `CONSUMER-ID` 的变化和 lag 的短暂上升。
3. 人为把 offset 重置到最早，观察重复消费范围，并用业务唯一 ID 做一次去重验证。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Offset | 分区内位置，提交的是下一条 |
| `__consumer_offsets` | 保存消费组进度，有独立保留期 |
| 自动提交 | 简单但可能丢，适合可丢消息 |
| 手动提交 | 处理成功后提交，至少一次 |
| Rebalance | 分区负责人变化，未提交消息会重复 |
| Eager/Cooperative | 全停全启 vs 增量迁移 |
| lag | LEO - committed offset，先找瓶颈再扩容 |
| 重置 offset | 危险操作，必须先停消费者 |

下一篇进入投递语义与幂等，把"至少一次"如何落到业务代码讲清楚：去重表怎么设计、唯一约束怎么加、事务边界画在哪里，以及 Kafka 事务到底能保证什么、不能保证什么。
