生产者看起来只是"把消息发出去"，但真正的难点在于：发出去的消息会不会丢、会不会重复、顺序对不对、吞吐够不够。Kafka 生产者把这些选择拆成了几个参数——`acks`、分区器、批大小、压缩、重试、幂等。本篇用 kafka-go 把这些参数逐个讲透，并说明哪些能力它没有、什么时候必须换 Sarama。

# 1. 生产者的职责

## 1.1 一条消息的旅程

```text
业务代码
   │ Send(key, value, headers)
   ▼
┌──────────────┐
│  Partition   │  key 哈希 / 轮询 / 按负载
└──────┬───────┘
       ▼
┌──────────────┐
│ Accumulator  │  按分区攒批，达到 BatchSize 或 BatchTimeout 就发
└──────┬───────┘
       ▼
┌──────────────┐
│  Compress    │  gzip / snappy / lz4 / zstd
└──────┬───────┘
       ▼
┌──────────────┐
│   Network    │  发送给分区 leader，按 acks 等待确认
└──────┬───────┘
       ▼
   成功 / 失败 / 超时重试
```

生产者要负责四件事：**选分区、攒批、压缩、处理失败**。业务代码只需要给出 key、value 和 headers，其余都在这条链路里完成。

## 1.2 同步与异步

| 维度 | 同步写入（`Async=false`） | 异步写入（`Async=true`） |
|------|---------------------------|--------------------------|
| 返回时机 | Broker 确认后返回 | 放进缓冲区就返回 |
| 错误感知 | 调用处直接拿到 error | 通过 `Completion` 回调 |
| 吞吐 | 受单次往返延迟影响 | 高，适合高并发 |
| 顺序 | 同一分区内按调用顺序 | 需注意回调并发和重试 |
| 适用 | 订单、支付、必须确认的场景 | 日志、埋点、可容忍少量失败 |

★ **关键认知**：同步和异步不是"可靠性"的区别，而是"错误在哪儿被处理"的区别。`Async=true` 依然可以做到不丢消息，前提是你认真处理 `Completion` 回调里的错误，并在进程退出前 `Close` 把缓冲区 flush 干净。

# 2. 同步写入

## 2.1 最小示例

```go
package main

import (
    "context"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

func main() {
    w := &kafka.Writer{
        Addr:         kafka.TCP("localhost:9092"),
        Topic:        "orders",
        Balancer:     &kafka.Hash{},        // 按 key 哈希，保证同 key 同分区
        RequiredAcks: kafka.RequireAll,     // 等所有 ISR 确认
        MaxAttempts:  5,                    // 可重试错误最多试 5 次
        Compression:  kafka.Snappy,         // 按批压缩
    }
    defer w.Close()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    err := w.WriteMessages(ctx, kafka.Message{
        Key:   []byte("order-1001"),
        Value: []byte(`{"order_id":"1001","amount":99.00}`),
        Headers: []kafka.Header{
            {Key: "event-type", Value: []byte("order.created")},
            {Key: "schema-version", Value: []byte("1")},
        },
    })
    if err != nil {
        log.Fatalf("发送失败: %v", err)
    }
}
```

## 2.2 关键参数

- **`RequiredAcks`**：生产端可靠性的第一道闸门。`RequireAll` 等所有 ISR 写入；`RequireOne` 只等 leader；`RequireNone` 发出去就不管。
- **`MaxAttempts`**：遇到可重试错误（网络抖动、leader 切换、`NotEnoughReplicas`）时的最大尝试次数。注意它是"尝试次数"不是"重试次数"。
- **`WriteBackoffMin` / `WriteBackoffMax`**：重试退避区间，默认 100ms 到 1s。退避太短会在 Broker 故障时形成重试风暴。
- **`Balancer`**：决定分区。`Hash` 按 key，`LeastBytes` 按负载，`RoundRobin` 轮询。

★ **关键认知**：`WriteMessages` 返回 nil 只代表"这一批消息被 Broker 确认了"。它不保证消费者处理成功，也不保证消息只出现一次。生产端能承诺的上限是"写入成功"，端到端语义要靠消费端和业务幂等一起完成。

## 2.3 错误处理

用 `errors.Is` 区分三类错误：`context.DeadlineExceeded` 表示超时，消息可能已写入也可能没有；`context.Canceled` 表示调用方主动取消，通常直接退出；其余错误记录 topic、key 和错误码后决定是否重试。

⚠ **关键认知**：超时是最难处理的错误——Broker 可能已经写入，只是响应没回来。此时盲目重试会产生重复消息。正确做法是给消息带一个业务唯一 ID，消费端按 ID 去重（第 6 篇展开）。

# 3. 异步写入

## 3.1 Async 与 Completion

```go
w := &kafka.Writer{
    Addr:         kafka.TCP("localhost:9092"),
    Topic:        "audit-log",
    Async:        true,
    BatchSize:    500,               // 攒够 500 条再发
    BatchTimeout: 200 * time.Millisecond, // 最多等 200ms
    Completion: func(msgs []kafka.Message, err error) {
        if err != nil {
            // 注意：这里在后台协程执行，不能做耗时操作
            log.Printf("异步发送失败，共 %d 条: %v", len(msgs), err)
            metrics.Inc("kafka_write_error", len(msgs))
        }
    },
}
defer w.Close() // 退出前必须 Close，否则缓冲区里的消息会丢

for _, evt := range events {
    if err := w.WriteMessages(ctx, evt); err != nil {
        log.Printf("入队失败: %v", err) // 缓冲区满或 ctx 取消
    }
}
```

## 3.2 异步的代价

异步写入把"等待 Broker"从业务链路里拿掉了，吞吐会明显上升，但引入三个新问题：

- **错误延迟暴露**：失败发生在回调里，业务调用处已经返回成功。
- **内存占用**：缓冲区积压会吃内存，`WriteMessages` 在缓冲满时会阻塞。
- **退出丢消息**：进程被 kill 前没 `Close`，未 flush 的批次就没了。

★ **关键认知**：`Completion` 回调运行在 Writer 的后台协程里，回调里做数据库写入、远程调用这类耗时操作会拖慢整个发送链路。正确做法是把失败消息投递到本地重试队列或日志，由独立协程处理。

⚠ **关键认知**：异步模式下不要依赖 `WriteMessages` 的返回值判断"发送成功"。它只表示"放进缓冲区成功"。真正的确认在 `Completion` 里，监控必须覆盖回调错误率。

# 4. 分区与顺序

## 4.1 分区键

Kafka 只保证**分区内有序**。需要同一业务实体的消息按顺序处理，就把实体 ID 作为 key：

```go
// 同一订单的创建、支付、取消会落到同一分区，按写入顺序被消费
w.WriteMessages(ctx, kafka.Message{
    Key:   []byte(orderID),
    Value: payload,
})
```

没有 key 时，消息会尽量均匀分布到各分区。Kafka 2.4 之后 Java 客户端默认使用**粘性分区器**（sticky partitioner）：先随机选一个分区，把一批消息都发过去，再换下一个，避免"逐条轮询"导致的小 batch。

| key | 分区行为 | 顺序 |
|-----|----------|------|
| 有 key | `hash(key) % 分区数` | 同 key 有序 |
| 无 key | 轮询或粘性分区 | 不保证 |
| 固定 key | 全部落一个分区 | 有序但热点 |

## 4.2 Balancer 选择

| Balancer | 行为 | 适用 |
|----------|------|------|
| `&kafka.Hash{}` | 按 key 哈希 | 需要同 key 有序 |
| `&kafka.LeastBytes{}` | 选当前字节数最少的分区 | 无 key、追求均衡 |
| `&kafka.RoundRobin{}` | 轮询 | 无 key、简单场景 |
| `&kafka.Murmur2Balancer{}` | 与 Java 客户端一致的哈希 | 多语言共用同一 Topic |

⚠ **关键认知**：不同客户端的分区哈希算法可能不同。Go 服务和 Java 服务都往同一个 Topic 写同一种 key，却用了不同的 Balancer，同一个 key 可能落到不同分区，顺序性就被破坏了。跨语言场景统一用 `Murmur2Balancer`（对应 Java 默认的 murmur2）。

## 4.3 顺序性保证

要真正做到"同一 key 严格有序"，需要同时满足：

1. 同一个 key 始终进同一个分区（分区数不变、Balancer 一致）。
2. 生产端同一时间只允许一个批次在途，或开启幂等生产者。
3. 消费端按分区顺序处理，不要把一个分区的消息并发丢给多个 worker。

```text
分区 0:  [A1][A2][A3]  → 消费者按 offset 顺序处理
分区 1:  [B1][B2]      → 与分区 0 无先后关系
```

★ **关键认知**：Kafka 的顺序是"分区内有序"，不是"消息级有序"。如果业务要求 A 订单和 B 订单之间也有先后，那只能把它们放到同一分区，代价是并行度下降。

# 5. 批处理与压缩

## 5.1 批参数

| 参数 | 作用 | 调大的收益 | 调大的代价 |
|------|------|------------|------------|
| `BatchSize` | 单批最多消息条数 | 摊薄协议开销 | 内存、延迟上升 |
| `BatchBytes` | 单批最大字节数 | 减少请求数 | 大消息更容易超时 |
| `BatchTimeout` | 攒批最长时间 | 提高批大小 | 低流量时延迟变高 |

低延迟场景：`BatchTimeout` 调小（如 10ms），`BatchSize` 适中。高吞吐场景：`BatchTimeout` 调到 100ms 以上，`BatchSize` 调到几百到几千。

## 5.2 压缩算法

| 算法 | 压缩比 | CPU | 延迟 | 建议 |
|------|--------|-----|------|------|
| gzip | 高 | 高 | 高 | 带宽极紧张、CPU 富余 |
| snappy | 中 | 低 | 低 | 通用默认 |
| lz4 | 中 | 很低 | 最低 | 延迟敏感 |
| zstd | 高 | 中 | 中 | 新项目首选 |

压缩发生在 Producer 端，Broker 原样存储压缩后的 batch，Consumer 拉取后解压。也就是说，压缩节省的是**网络和磁盘**，消耗的是**客户端 CPU**。

★ **关键认知**：压缩率依赖"批"。单条消息压缩几乎没有收益，因为每条都要带压缩头。先把批攒起来，再谈压缩。如果发现压缩后体积没降，先检查 `BatchSize` 和 `BatchTimeout` 是不是太小。

大量 100 字节以下的小消息会让固定开销（offset、时间戳、headers、batch 头）占比过高，也会让分区索引和文件段增多、page cache 命中率下降。解决办法是**在业务层合并**：把多条小事件合并成一个批次消息，不要指望调大 Kafka 参数就能解决消息模型本身的问题。

# 6. 重试与幂等

## 6.1 重试参数

```go
w := &kafka.Writer{
    Addr:           kafka.TCP("localhost:9092"),
    Topic:          "orders",
    RequiredAcks:   kafka.RequireAll,
    MaxAttempts:    10,
    WriteBackoffMin: 100 * time.Millisecond,
    WriteBackoffMax: 2 * time.Second,
}
```

可重试错误包括网络超时、leader 切换、副本不足；不可重试错误包括消息过大、Topic 不存在、序列化失败。对所有错误一律重试，只会放大故障。

## 6.2 幂等生产者

Kafka 的幂等生产者通过 **PID（Producer ID）+ 分区级序列号** 去重：

```text
Producer 启动 → 向 Broker 申请 PID
每条消息 → (PID, Partition, Sequence)
Broker 按分区记录最大序列号，重复的序列号直接丢弃
```

开启条件（Java 客户端）：

```text
enable.idempotence=true
acks=all
retries>0
max.in.flight.requests.per.connection<=5
```

幂等生产者能解决"生产端重试导致的重复"，但**不能跨生产者会话去重**：进程重启后 PID 变了，Broker 会当成新生产者。要跨会话，需要事务和 `transactional.id`。

## 6.3 kafka-go 与 Sarama 的差异

| 能力 | kafka-go | Sarama |
|------|----------|--------|
| 同步/异步发送 | 支持 | 支持 |
| 自定义分区 | Balancer | Partitioner |
| 幂等生产者 | 不支持 | `Producer.Idempotent=true` |
| 事务 | 不支持 | 支持 |
| 批处理/压缩 | 支持 | 支持 |

★ **关键认知**：kafka-go 的 `MaxAttempts` 重试**不保证幂等**。超时后重试可能让同一条消息在分区里出现两次。如果你的业务不能容忍重复，要么在消费端做去重，要么换 Sarama 开启幂等生产者，不要指望客户端参数自动解决。

⚠ **关键认知**：幂等生产者只保证"单分区、单会话内不重复"，不保证"业务只执行一次"。真正的事故往往发生在"数据库已提交、offset 未提交"的窗口里，这部分必须靠消费端幂等或事务来兜底。

# 7. 常见错误与坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 吞吐上不去 | `BatchTimeout` 太小、`BatchSize` 太小 | 适当调大批量 |
| 发送延迟高 | `RequiredAcks=all` + ISR 慢 | 查 ISR、磁盘、网络 |
| 消息重复 | 超时重试、无幂等 | 业务去重或开幂等 |
| 顺序错乱 | 重试、多分区、并发消费 | 同 key 同分区 + 单并发处理 |
| `MessageTooLarge` | 超过 Broker `message.max.bytes` | 拆消息或调大限制 |
| 退出丢消息 | 异步模式没 `Close` | 退出前 flush 并等待回调 |

⚠ **关键认知**：Key 设计不当会造成**热点分区**。比如用固定字符串 `"order"` 当 key，所有消息都进一个分区，其他分区闲着，消费并行度也上不去。Key 要选"有业务意义且分布均匀"的字段，比如订单 ID、用户 ID。

# 8. 动手练习

1. 用 `kafka.Hash` 发送 100 条 key 为 `user-1` 到 `user-10` 的消息，用 `kafka-console-consumer --property print.key=true --property print.partition=true` 验证同 key 是否落同一分区。
2. 把 `BatchTimeout` 从 1s 改成 10ms，对比 1000 条消息的发送耗时和 Broker 收到的请求数，再用异步写入观察 `Completion` 回调里的错误。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 同步/异步 | 同步在调用处拿错误，异步在回调里拿 |
| 可靠性 | `RequireAll` + 副本 + `min.insync.replicas` |
| 分区 | 有 key 哈希，无 key 轮询/粘性 |
| 顺序 | 同 key 同分区 + 单批次在途 + 消费单并发 |
| 批处理 | 批越大吞吐越高，延迟也越高 |
| 压缩 | 按批生效，zstd 综合最优 |
| 重试 | 只重试可重试错误，注意重复 |
| 幂等 | kafka-go 不支持，Sarama 支持 |

下一篇进入消费者与消费组，讲清楚分区怎么分配、offset 什么时候提交、rebalance 为什么会发生，以及一个 Go 服务如何优雅退出而不丢正在处理的消息。
