Redis 有两套消息能力：Pub/Sub 是"广播即忘"，Stream 是"可持久化的日志 + 消费组"。很多团队用 Pub/Sub 做业务消息，结果消费者重启期间的消息全丢了；也有人用 List 当队列，消费者崩溃后任务就没了。本篇把三种方案放在一起对比，重点讲清 Stream 的消费组、ACK、Pending 和堆积处理。

# 1. Pub/Sub：发布订阅

## 1.1 基本模型

```text
       ┌─────────┐
       │ Publisher│
       └────┬────┘
            │ PUBLISH channel msg
            ▼
     ┌──────────────┐
     │    Redis     │  没有订阅者时，消息直接丢弃
     └──┬────┬────┬─┘
        │    │    │
        ▼    ▼    ▼
      Sub1 Sub2 Sub3   同一频道的所有订阅者都收到
```

```go
// 订阅
sub := rdb.Subscribe(ctx, "news")
defer sub.Close()

ch := sub.Channel() // 返回一个 Go channel
for msg := range ch {
    fmt.Printf("频道=%s 内容=%s\n", msg.Channel, msg.Payload)
}

// 发布（另一个客户端/协程）
rdb.Publish(ctx, "news", "hello")
```

★ **关键认知**：Pub/Sub 是**即时广播**，消息不落盘、不持久化。发布时如果没有在线订阅者，消息就永久丢失。它适合"配置变更通知""缓存失效广播"这类**丢了也能恢复**的场景，不适合订单、支付这类业务消息。

## 1.2 模式订阅

```go
// 订阅所有 news. 开头的频道
sub := rdb.PSubscribe(ctx, "news.*")
```

```bash
PSUBSCRIBE news.*    # 通配符订阅
PUBSUB CHANNELS      # 查看当前活跃频道
PUBSUB NUMSUB news   # 查看某频道订阅数
```

⚠ **关键认知**：Pub/Sub 的订阅连接是**独占**的，不能在这个连接上执行普通命令。go-redis 会为订阅单独建连，所以连接池里的连接数要预留订阅所需的数量。

# 2. Stream：持久化消息

## 2.1 结构

Stream 是一个**只追加的日志**，每条消息有唯一 ID（格式 `时间戳-序号`），消息写入后不会因为被消费而删除。

```text
Stream: orders
┌──────────────────────────────────────────────┐
│ 1700000000000-0  {order_id:1001, amount:99}  │
│ 1700000000001-0  {order_id:1002, amount:50}  │
│ 1700000000002-0  {order_id:1003, amount:20}  │
└──────────────────────────────────────────────┘
        │
        ├── 消费组 g1 ── consumer A 处理前两条
        └── 消费组 g2 ── consumer B 处理全部（各自独立）
```

```go
// 生产
id, err := rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "orders",
    Values: map[string]interface{}{"order_id": 1001, "amount": 99},
}).Result()
```

常用命令：

| 命令 | 作用 |
|------|------|
| `XADD` | 追加消息 |
| `XLEN` | 消息总数 |
| `XRANGE` | 按 ID 范围读取 |
| `XDEL` | 删除指定消息 |
| `XTRIM` | 裁剪旧消息 |
| `XGROUP CREATE` | 创建消费组 |
| `XREADGROUP` | 消费组读取 |
| `XACK` | 确认消息 |
| `XPENDING` | 查看未确认消息 |
| `XCLAIM` | 接管超时消息 |

## 2.2 消费组与 ACK

消费组的语义是"**同组内分摊、组间独立**"：组 g1 的消息只会被它的某个消费者处理；组 g2 可以独立消费同一批消息。

```go
// 创建消费组，0 表示从最开始读，$ 表示只读新消息
err := rdb.XGroupCreateMkStream(ctx, "orders", "g1", "0").Err()
if err != nil && !strings.Contains(err.Error(), "BUSYGROUP") {
    return err // 组已存在不算错误
}

for {
    res, err := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
        Group:    "g1",
        Consumer: "consumer-1", // 消费者名，重启后应保持稳定
        Streams:  []string{"orders", ">"},
        Count:    10,
        Block:    2 * time.Second,
    }).Result()
    if errors.Is(err, redis.Nil) {
        continue // 没有新消息
    }
    if err != nil {
        return err
    }

    for _, msg := range res[0].Messages {
        if err := handle(msg); err != nil {
            continue // 不 ACK，留在 Pending，等待重试
        }
        rdb.XAck(ctx, "orders", "g1", msg.ID) // 处理成功才确认
    }
}
```

★ **关键认知**：`>` 表示"只读**从未投递过**的新消息"。如果要重新处理自己 Pending 里的消息，需要用具体 ID（通常是 `0`）读取：`XREADGROUP GROUP g1 c1 STREAMS orders 0`。很多"消息丢了"的错觉，其实是消费者只读 `>`，从没回看 Pending。

## 2.3 Pending 列表

消息被投递给某个消费者后，在 `XACK` 之前一直留在 Pending（PEL，Pending Entries List）里。

```bash
XPENDING orders g1              # 汇总：多少条、最小/最大 ID、各消费者数量
XPENDING orders g1 - + 10       # 明细：哪条消息、被谁拿走、投递了几次
```

```go
pending, err := rdb.XPendingExt(ctx, &redis.XPendingExtArgs{
    Stream: "orders",
    Group:  "g1",
    Start:  "-",
    End:    "+",
    Count:  100,
}).Result()

for _, p := range pending {
    // p.Idle 是消息被投递后闲置的时长
    if p.Idle > time.Minute {
        // 超过 1 分钟没 ACK，可以被其他消费者接管
    }
}
```

★ **关键认知**：Pending 是 Stream 可靠性的核心。它记录了"消息已投递但未确认"，消费者崩溃后消息不会消失，只是挂在 Pending 里。必须有**另一个人**来处理这些消息，否则它们永远不会被消费。

## 2.4 接管超时消息：XCLAIM

```bash
# 把闲置超过 60 秒的消息转交给 consumer-2
XCLAIM orders g1 consumer-2 60000 <消息ID>
```

```go
claimed, err := rdb.XClaim(ctx, &redis.XClaimArgs{
    Stream:   "orders",
    Group:    "g1",
    Consumer: "consumer-2",
    MinIdle:  time.Minute, // 只接管闲置超过 1 分钟的消息
    Messages: []string{"1700000000000-0"},
}).Result()
```

生产上通常配一个"巡检协程"，周期性扫描 Pending，把超时消息重新投递或告警。

⚠ **关键认知**：`XCLAIM` 会把消息的投递次数加 1。如果某条消息反复处理失败，投递次数会一直涨，最终可能拖垮系统。要设置最大重试次数，超过后转入死信 Stream 或人工处理。

# 3. 消息堆积

## 3.1 为什么会堆积

| 原因 | 表现 |
|------|------|
| 生产速度 > 消费速度 | `XLEN` 持续增长 |
| 消费者卡死 | Pending 数量持续增长 |
| 消费者没 ACK | Pending 里有大量旧消息 |
| 消费者组过多 | 每个组都要保留自己的进度 |

## 3.2 处理手段

**扩容消费者**：同一消费组内可以起多个消费者，消息会自动分摊：

```go
// 每个实例用不同的消费者名
consumerName := fmt.Sprintf("consumer-%s", hostname)
```

**限制 Stream 长度**：用 `MAXLEN` 近似裁剪，防止无限增长：

```go
rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "orders",
    MaxLen: 100000,       // 大约保留 10 万条
    Approx: true,         // ~ 近似裁剪，性能更好
    Values: map[string]interface{}{"order_id": 1001},
})
```

```bash
XTRIM orders MAXLEN ~ 100000
```

★ **关键认知**：`MAXLEN` 会删除**最旧**的消息，可能把还没消费的消息删掉。只有当"旧消息可以丢"时才用裁剪；如果消息必须处理完，就靠扩容消费者和监控 Pending，而不是裁剪。

## 3.3 STREAM 与 List 的取舍

| 维度 | List | Stream |
|------|------|--------|
| 持久化 | 有，但弹出即删 | 有，消费不删除 |
| 消费组 | 不支持 | 支持 |
| ACK | 无 | 有 |
| 消息回溯 | 不能 | 可以按 ID 读 |
| 内存 | 出队即释放 | 需手动裁剪 |
| 复杂度 | 低 | 中 |

结论：**临时任务用 List，业务消息用 Stream**。List 的 `BRPOP` 简单好用，但它无法回答"这条消息处理成功了吗"。

# 4. 可靠消息方案

## 4.1 至少一次投递

```text
生产者 XADD
   │
   ▼
Stream ──XREADGROUP──▶ 消费者处理
   │                      │
   │                      ├── 成功 ──XACK──▶ 移出 Pending
   │                      │
   │                      └── 失败/崩溃 ──▶ 留在 Pending
   │                                            │
   └──────── 巡检协程 XCLAIM 超时消息 ◀──────────┘
```

★ **关键认知**：这套模型是**至少一次（at-least-once）**，不是恰好一次。因为"处理成功但 ACK 前崩溃"会导致消息被重新投递。要实现业务上的"恰好一次"，必须让处理逻辑**幂等**：用唯一业务 ID 做去重，或让数据库写操作带上状态条件。

## 4.2 幂等消费示例

```go
func handleOrder(ctx context.Context, msg redis.XMessage) error {
    orderID := msg.Values["order_id"].(string)

    // 用 SetNX 做去重标记，key 带业务 ID
    ok, err := rdb.SetNX(ctx, "processed:order:"+orderID, "1", 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !ok {
        return nil // 已经处理过，直接返回成功，稍后 ACK
    }

    // 真正的业务处理
    if err := db.CreateOrder(ctx, orderID); err != nil {
        rdb.Del(ctx, "processed:order:"+orderID) // 失败要撤销标记，允许重试
        return err
    }
    return nil
}
```

⚠ **关键认知**：去重标记要在**业务成功后再保留**，业务失败时要删掉，否则重试会被误判为"已处理"。更严谨的做法是把去重和业务写操作放在同一个数据库事务里。

## 4.3 死信处理

```go
const maxDeliveries = 5

for _, p := range pending {
    if p.RetryCount >= maxDeliveries {
        // 超过重试上限，转入死信 Stream，人工排查
        rdb.XAdd(ctx, &redis.XAddArgs{
            Stream: "orders:dead",
            Values: map[string]interface{}{"id": p.ID, "reason": "too many retries"},
        })
        rdb.XAck(ctx, "orders", "g1", p.ID) // 从原组移除，避免无限循环
    }
}
```

# 5. Pub/Sub vs Stream vs List

| 维度 | Pub/Sub | List | Stream |
|------|---------|------|--------|
| 消息持久化 | 否 | 是 | 是 |
| 离线消息 | 丢失 | 保留 | 保留 |
| 多消费者分摊 | 否（广播） | 是 | 是（消费组） |
| ACK 确认 | 无 | 无 | 有 |
| 回溯历史 | 不能 | 不能 | 能 |
| 适用场景 | 通知、广播 | 简单任务队列 | 业务消息、事件流 |

★ **关键认知**：选型的核心问题是"**消息丢了能不能接受**"。能接受 → Pub/Sub 最轻；不能接受但不需要 ACK → List；需要"处理确认 + 重试 + 回溯" → Stream。不要用 Pub/Sub 承载订单，也不要为了简单用 List 承载支付。

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 订阅者收不到消息 | 发布时订阅者不在线 | 改用 Stream 持久化 |
| 消息重复消费 | 处理成功但 ACK 前崩溃 | 业务幂等 |
| Pending 一直增长 | 消费者没 ACK 或处理卡住 | 查 `XPENDING`，修消费者 |
| `NOGROUP` | 消费组不存在 | `XGROUP CREATE` 先建组 |
| `BUSYGROUP` | 组已存在重复创建 | 忽略该错误 |
| Stream 内存暴涨 | 没裁剪 | `MAXLEN ~` 或 `XTRIM` |
| 新消费者读不到旧消息 | 用 `>` 只读新消息 | 用 `0` 读 Pending 或从头读 |
| 消费者名重复 | 多实例用了同一个名字 | 用 hostname/pod 名区分 |

⚠ **关键认知**：多个实例共用同一个消费者名会造成"消息被其中一个实例拿走，另一个以为自己也该处理"的混乱。消费者名要**每个实例唯一**，并且**重启后保持不变**，这样才能继续处理自己的 Pending。

# 7. 动手练习

1. 用 Pub/Sub 起一个订阅者，验证"订阅前发布的消息收不到"。
2. 创建 Stream 和消费组，故意处理失败不 ACK，用 `XPENDING` 观察消息挂在 Pending 里。
3. 用 `XCLAIM` 把超时消息转交给另一个消费者，验证消息能被重新处理。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Pub/Sub | 广播即忘，不持久化，适合通知 |
| Stream | 只追加日志，支持消费组和 ACK |
| 消费组 | 组内分摊、组间独立 |
| Pending | 已投递未确认，消费者崩溃后的兜底 |
| XCLAIM | 接管超时消息，注意重试上限 |
| 可靠投递 | 至少一次 + 业务幂等 + 死信 |
| 堆积 | 扩容消费者、监控 Pending、必要时裁剪 |

下一篇进入集群与哨兵，讲主从复制、Sentinel 故障转移、Cluster 槽位、MOVED/ASK 重定向和热点 Key 的处理。