消费者比生产者更容易出问题：消息重复了、顺序乱了、rebalance 一直转、服务重启后 offset 丢了。这些现象大多不是 Kafka 的 bug，而是消费组协议和 offset 提交时机没理解透。本篇从 Consumer Group 的分配机制讲起，用 kafka-go 写出一个可靠的消费循环，再把优雅退出和并发处理的边界讲清楚。

# 1. Consumer 与 Consumer Group

## 1.1 两种消费模式

Kafka 消费者有两种用法：

| 模式 | kafka-go 配置 | 分区分配 | 适用 |
|------|---------------|----------|------|
| 手动指定分区 | 设 `Topic` + `Partition`，不设 `GroupID` | 自己决定 | 精确控制、单分区任务 |
| 消费组订阅 | 设 `GroupID` + `Topic` | Broker 协调器分配 | 绝大多数业务消费 |

消费组模式是主流：多个实例用同一个 `GroupID`，Kafka 自动把分区分给它们，某个实例挂了会自动把分区转给其他实例。

## 1.2 消费组的工作方式

```text
Topic: orders（3 个分区）
┌─────────────┬─────────────┬─────────────┐
│ Partition 0 │ Partition 1 │ Partition 2 │
└──────┬──────┴──────┬──────┴──────┬──────┘
       │             │             │
       ▼             ▼             ▼
   ┌───────────────────────────────────┐
   │      Group: order-worker          │
   │  Consumer A   Consumer B   Consumer C │
   │     P0            P1           P2  │
   └───────────────────────────────────┘

另一个 Group: analytics-worker
   │ 独立消费 P0、P1、P2 的全量消息
   ▼
```

规则很简单：

- **同组内分摊**：一个分区在同一时刻只会分给组内的一个消费者。
- **组间独立**：不同 `GroupID` 各自维护 offset，互不影响。
- **消费者多于分区会空闲**：3 个分区最多支撑 3 个并行消费者，第 4 个分不到分区。
- **消费者少于分区会兼职**：1 个消费者可以同时消费多个分区。

★ **关键认知**：消费组的并行度上限是**分区数**，不是消费者数。想提高消费能力，先看分区数够不够；盲目加 Pod 只会让多出来的实例空转，还会增加 rebalance 频率。

# 2. 消费组协议

## 2.1 JoinGroup / SyncGroup / Heartbeat

```text
Consumer                     Group Coordinator（某个 Broker）
   │ 1. FindCoordinator ────────────────▶ │
   │ ◀──────────── coordinator 地址 ───── │
   │ 2. JoinGroup（订阅哪些 Topic） ─────▶ │
   │ ◀──────────── 成员列表 + 谁是 leader ─│
   │ 3. SyncGroup（leader 提交分配方案） ─▶ │
   │ ◀──────────── 分到的分区列表 ──────── │
   │ 4. Heartbeat（周期心跳） ───────────▶ │
   │ 5. LeaveGroup（优雅退出时） ────────▶ │
```

- **Coordinator**：由 `group.id` 哈希选出的 Broker，负责管理消费组成员和 offset。
- **JoinGroup**：成员上报自己订阅的 Topic；Coordinator 选一个成员作为 leader。
- **SyncGroup**：leader 计算分区分配方案，Coordinator 下发给所有成员。
- **Heartbeat**：成员定期报活，超过 `session.timeout.ms` 没心跳就被踢出组。

三个超时参数要分清：

| 参数 | 含义 | kafka-go 字段 |
|------|------|---------------|
| session timeout | 多久没心跳判定成员死亡 | `SessionTimeout` |
| heartbeat interval | 心跳间隔 | `HeartbeatInterval` |
| rebalance timeout | 等待所有成员重新加入的最长时间 | `RebalanceTimeout` |

## 2.2 分区分配策略

| 策略 | 思路 | 优点 | 缺点 |
|------|------|------|------|
| Range | 按 Topic 排序，连续切分 | 实现简单，同 key 稳定 | 多 Topic 时可能不均 |
| RoundRobin | 所有分区轮流发 | 整体均衡 | rebalance 时迁移多 |
| Sticky | 尽量均衡且少迁移 | 减少 rebalance 抖动 | 实现复杂 |
| CooperativeSticky | 增量式撤销/分配 | 不全量停消费 | 需要客户端和服务端支持 |

Kafka 2.4 之后推荐 **CooperativeSticky**：rebalance 时只撤销需要迁移的分区，其他分区继续消费，避免"整组停摆"。kafka-go 默认用 Range，可以显式配置 `GroupBalancers`。

★ **关键认知**：分配策略只决定"谁拿哪些分区"，不决定"消息给谁"。消息始终属于分区，消费者只是分区当前的负责人。理解这一点，就不会问"为什么同一个 key 有时被 A 处理、有时被 B 处理"——rebalance 后分区换人了，但同 key 仍然在同一分区，只是负责人变了。

## 2.3 Rebalance 的触发条件

- 消费者加入或离开（扩容、缩容、滚动发布）。
- 消费者心跳超时，被 Coordinator 踢出。
- 消费者处理太慢，超过 `max.poll.interval.ms`（Java 客户端）。
- Topic 的分区数发生变化。
- 订阅的 Topic 列表变化。
- Coordinator 所在 Broker 故障，重新选主。

rebalance 期间，分区会短暂不可消费。Eager 协议要求所有消费者先放弃全部分区，再重新分配；Cooperative 协议只迁移必要的分区。

⚠ **关键认知**：滚动发布时如果每个实例启动后立刻加入组，会连续触发多次 rebalance。可以用**静态成员**（`group.instance.id`）让实例在重启窗口内保留身份，或者放慢发布节奏。kafka-go 对静态成员的支持有限，滚动发布要留足 `RebalanceTimeout`。

# 3. kafka-go 消费者实战

## 3.1 最小消费组示例

```go
package main

import (
    "context"
    "errors"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

func main() {
    r := kafka.NewReader(kafka.ReaderConfig{
        Brokers:           []string{"localhost:9092"},
        GroupID:           "order-worker",
        Topic:             "orders",
        MinBytes:          10e3,             // 至少 10KB 才返回，减少空转
        MaxBytes:          10e6,             // 单次最多拉 10MB
        MaxWait:           time.Second,      // 没数据时最多等 1 秒
        StartOffset:       kafka.FirstOffset, // 新组从最早开始读
        SessionTimeout:    30 * time.Second,
        HeartbeatInterval: 3 * time.Second,
        RebalanceTimeout:  60 * time.Second,
        IsolationLevel:    kafka.ReadCommitted, // 只读已提交的事务消息
    })
    defer r.Close()

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    for {
        m, err := r.FetchMessage(ctx) // 只拉取，不提交
        if err != nil {
            if errors.Is(err, context.Canceled) {
                break
            }
            log.Printf("拉取失败: %v", err)
            continue
        }

        if err := process(m); err != nil {
            log.Printf("处理失败 partition=%d offset=%d: %v", m.Partition, m.Offset, err)
            continue // 不提交，下次重新拉取
        }

        // 业务成功后再提交，保证至少一次
        if err := r.CommitMessages(ctx, m); err != nil {
            log.Printf("提交失败 partition=%d offset=%d: %v", m.Partition, m.Offset, err)
        }
    }
}

func process(m kafka.Message) error {
    log.Printf("处理 key=%s value=%s", m.Key, m.Value)
    return nil
}
```

## 3.2 逐段说明

- **`GroupID`**：消费组标识。同一个组共享分区和 offset，不同组互不影响。
- **`StartOffset`**：组内没有提交记录时的起点。kafka-go 默认 `FirstOffset`（从最早读），Java 客户端默认 `latest`（只读新消息），跨语言时要注意。
- **`MinBytes` / `MaxBytes` / `MaxWait`**：拉取批量与等待策略。吞吐优先调大 `MinBytes`，延迟优先调小 `MaxWait`。
- **`IsolationLevel`**：`ReadCommitted` 会跳过未提交的事务消息；不涉及事务时用默认的 `ReadUncommitted` 即可。
- **`FetchMessage`**：只拉取，不提交 offset。要自动提交就用 `ReadMessage`，但那样很难保证"处理成功再提交"。

## 3.3 手动提交 offset

`CommitMessages(ctx, m)` 提交的是 `m.Offset + 1`，表示"这个位置之前的消息都处理完了"。

```go
// 逐条提交：简单、可靠，但每条一次网络往返
r.CommitMessages(ctx, m)

// 批量提交：处理完一批后提交最后一条
if len(batch) > 0 {
    if err := r.CommitMessages(ctx, batch[len(batch)-1]); err != nil {
        log.Printf("批量提交失败: %v", err)
    }
}
```

★ **关键认知**：offset 提交是**按分区**的。提交 offset=101 表示该分区 0 到 100 都已完成，不表示其他分区也到了这个位置。kafka-go 会按消息所属分区分别提交，不需要你手动分组。

⚠ **关键认知**：批量提交时，如果批次里前面的消息处理失败、后面的成功，提交最后一条会跳过失败的消息。必须保证**提交位置之前的消息全部成功**，否则就要记录失败位置并回退，或者改成逐条提交。

需要同时消费多个 Topic 时，把 `Topic` 换成 `GroupTopics: []string{"orders", "order-events"}`，它们会一起参与分区分配。处理逻辑差别很大的消息建议拆成不同消费组，避免互相拖慢。

# 4. 优雅退出

## 4.1 信号处理

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

for {
    m, err := r.FetchMessage(ctx)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            break // 收到退出信号
        }
        log.Printf("拉取失败: %v", err)
        continue
    }
    // 处理消息……
}
```

`signal.NotifyContext` 会在收到 SIGINT/SIGTERM 时取消 ctx。`FetchMessage` 随之返回 `context.Canceled`，循环退出。

## 4.2 关闭顺序

```text
1. 收到退出信号，停止拉取新消息
2. 等待正在处理的消息完成
3. 提交已完成消息的 offset
4. 调用 Reader.Close()，触发 LeaveGroup
5. 关闭数据库、HTTP 服务等下游资源
```

顺序不能反：先关下游再提交 offset，提交可能失败；先 `Close` 再提交，Reader 已经不能用了。

## 4.3 处理中的消息怎么办

收到退出信号时，`ctx` 已经被取消，继续用它做提交会立即返回 `context.Canceled`。正确做法是给收尾操作一个独立的超时 ctx：

```go
commitCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
if err := r.CommitMessages(commitCtx, m); err != nil {
    log.Printf("退出时提交失败: %v", err) // 下次启动会重复消费，靠幂等兜底
}
```

Go 1.21+ 也可以用 `context.WithoutCancel(ctx)` 保留原 ctx 的值但去掉取消信号，再叠加超时。

★ **关键认知**：优雅退出只能保证"尽量提交"，不能保证"一定不重复"。进程被 `kill -9`、机器断电、提交请求在网络上丢失，都会让消息重新投递。消费逻辑必须幂等，这是第 6 篇的主题。

# 5. 消费速率与背压

## 5.1 Fetch 参数

| 参数 | 默认 | 调大的效果 | 调小的效果 |
|------|------|------------|------------|
| `MinBytes` | 1 | 批更大、请求更少、延迟更高 | 低延迟、请求更多 |
| `MaxBytes` | 1MB | 单次拉更多、内存更高 | 内存低、吞吐可能下降 |
| `MaxWait` | 10s | 没数据时等更久、长轮询更省 | 空转更多、延迟更低 |

低延迟场景可以设 `MinBytes=1`、`MaxWait=100ms`；高吞吐场景设 `MinBytes=1MB`、`MaxWait=500ms` 到 1s。

## 5.2 处理并发

`Reader` 不是并发安全的，读只能由一个 goroutine 负责。要并发处理，可以"单读多处理"：读循环把 `FetchMessage` 拿到的消息投递到一个带缓冲的 channel，固定数量的 worker 从 channel 里取消息处理，退出时先关 channel 再等 `WaitGroup`。这套模式本身不难，难的是提交 offset。

⚠ **关键认知**：并发处理会打乱提交顺序。如果 worker 各自提交 offset，后处理完的消息可能先提交，导致前面未完成的消息被跳过。正确做法是每个分区维护一个"已完成 offset 水位"，只提交连续完成的部分；或者按分区串行处理，牺牲一部分并行度。

## 5.3 不要阻塞心跳

Kafka 的成员存活靠心跳维持。kafka-go 在后台发送心跳，但如果你长时间不调用 `FetchMessage`，本地队列会堆积，lag 会持续上涨。Java 客户端还有 `max.poll.interval.ms`：两次 poll 间隔太长会被踢出组。kafka-go 没有同名参数，但同样不要把消费循环卡在慢查询或大文件处理上。

★ **关键认知**：消费慢和消费挂是两件事。消费慢表现为 lag 上涨但成员还在；消费挂表现为心跳超时、分区被转走、日志里反复 rebalance。前者要扩容或优化处理逻辑，后者要查 GC、死锁和下游超时。

# 6. 常见错误与坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 消费不到消息 | 新组把 `StartOffset` 设成了 `LastOffset` | 改回 `FirstOffset` |
| 重复消费 | 处理成功但提交失败/进程崩溃 | 业务幂等 |
| 消息被跳过 | 并发处理 + 乱序提交 | 按分区维护提交水位 |
| 一直 rebalance | 处理太慢、心跳超时、频繁扩缩容 | 调大超时、减少发布抖动 |
| 消费者空闲 | 消费者数大于分区数 | 扩分区或减实例 |
| `GroupCoordinatorNotAvailable` | 协调器切换 | 重试并观察 `__consumer_offsets` |
| 提交 offset 报错 | ctx 已取消、组在 rebalance | 用独立 ctx 重试 |

⚠ **关键认知**：`Reader` 不是并发安全的，但 `Writer` 是。不要因为 Writer 能共用，就想当然地让多个 goroutine 同时 `ReadMessage`。读循环只能有一个，需要并行就把消息分发给 worker。

# 7. 动手练习

1. 起两个使用同一 `GroupID` 的消费者，观察分区分配；再停掉一个，观察剩余消费者接管全部分区需要多久。
2. 在处理函数里 `sleep 5s`，把 `SessionTimeout` 调到 3s，观察 rebalance 日志和消息重复情况；再用 `FetchMessage` + 手动提交验证 SIGTERM 后正在处理的消息能完成并提交。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 消费组 | 同组分摊、组间独立 |
| 并行度 | 上限等于分区数 |
| 协议 | JoinGroup、SyncGroup、Heartbeat |
| 分配策略 | Range 简单，CooperativeSticky 少抖动 |
| 提交 | `ReadMessage` 自动提交，可靠场景用 `FetchMessage` |
| 优雅退出 | 停拉取、等处理、独立 ctx 提交、Close |
| 并发 | Reader 单协程读，worker 处理，注意提交水位 |
| 背压 | 调 Fetch 参数、扩分区、优化处理逻辑 |

下一篇专门讲 offset 与 rebalance：三种 offset 的语义、自动提交和手动提交的取舍、rebalance 期间消息到底会不会丢、消费 lag 怎么算和怎么处理。
