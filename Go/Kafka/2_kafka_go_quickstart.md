Go 里操作 Kafka 有两个主流选择：`segmentio/kafka-go` 和 `IBM/sarama`。前者 API 简洁、纯 Go、不用 CGO，适合大多数业务服务；后者功能更全，事务、幂等生产者、Admin 能力更完整。本篇先用 kafka-go 跑通"装环境、写消息、读消息、正确关闭"这条最小闭环，把新手最容易踩的隐式提交和连接配置讲清楚。

# 1. 客户端选型与环境

## 1.1 kafka-go 与 Sarama 的差异

| 维度 | kafka-go | Sarama |
|------|----------|--------|
| 维护方 | Segment | IBM |
| API 风格 | 高层封装，Writer/Reader | 偏底层，SyncProducer/AsyncProducer |
| 依赖 | 纯 Go，无 CGO | 纯 Go |
| 事务 | 不支持 | 支持 |
| 幂等生产者 | 不支持 | 支持 |
| 消费组 | 支持 | 支持 |
| 自定义分区 | Balancer 接口 | Partitioner 接口 |
| 学习成本 | 低 | 中高 |
| 适合 | 常规生产消费、快速落地 | 需要 EOS、精细控制协议 |

★ **关键认知**：选 kafka-go 不是"功能少所以简单"，而是它的抽象刚好覆盖大多数业务：`Writer` 负责攒批、重试、分区，`Reader` 负责拉取、消费组、提交 offset。只有当你要用 Kafka 事务做端到端 exactly-once，或者需要精细控制每个请求时，才值得换 Sarama。

## 1.2 版本约定

本文示例基于 Kafka 3.x（KRaft 模式）、Go 1.22+、`github.com/segmentio/kafka-go v0.4.x`。新代码建议直接用结构体字面量 `&kafka.Writer{...}`，网上不少 `kafka.NewWriter(kafka.WriterConfig{...})` 的旧写法已经过时。

# 2. 本地环境与安装

## 2.1 用 Docker 启动 Kafka

单节点 KRaft 模式足够开发使用：

```bash
docker run -d --name kafka -p 9092:9092 apache/kafka:3.7.0
```

官方镜像默认就是单节点 KRaft 配置，`localhost:9092` 可直接使用。生产环境当然要显式配置监听地址、副本因子和 `__consumer_offsets` 的副本数。

创建一个 3 分区的 Topic：

```bash
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic hello --partitions 3 --replication-factor 1
```

确认创建成功：

```bash
docker exec -it kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --describe --topic hello
```

⚠ **关键认知**：`ADVERTISED_LISTENERS` 必须是客户端能访问到的地址。很多"本地能连、容器里连不上"的问题，都是 Broker 返回了容器内网 IP 或 `localhost`，而客户端在另一个网络命名空间里。

## 2.2 安装 kafka-go

```bash
mkdir kafka-go-demo && cd kafka-go-demo
go mod init example.com/kafka-demo
go get github.com/segmentio/kafka-go@v0.4.49
```

国内网络慢可以先执行 `go env -w GOPROXY=https://goproxy.cn,direct`。

# 3. 第一个生产者

## 3.1 最小可运行示例

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

func main() {
    // Writer 可以复用，内部维护连接和按分区的缓冲区
    w := &kafka.Writer{
        Addr:         kafka.TCP("localhost:9092"), // 只需给一个入口地址
        Topic:        "hello",
        Balancer:     &kafka.LeastBytes{},          // 按分区当前负载选分区
        RequiredAcks: kafka.RequireAll,             // 等所有 ISR 确认
        Async:        false,                        // 同步写，返回时已确认
    }
    defer w.Close() // 关闭时会 flush 未发送的批次

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    err := w.WriteMessages(ctx, kafka.Message{
        Key:   []byte("user-1"), // 有 key 时默认按 key 哈希到固定分区
        Value: []byte(`{"id":1,"name":"alice"}`),
        Headers: []kafka.Header{
            {Key: "trace-id", Value: []byte("abc-123")},
        },
    })
    if err != nil {
        log.Fatalf("写入失败: %v", err) // 包含连接、超时、副本不足等错误
    }

    fmt.Println("写入成功")
}
```

## 3.2 逐段说明

- **`Addr`**：kafka-go 的 `Writer` 用 `Addr` 而不是 `Brokers`，传 `kafka.TCP(...)` 即可。它会自动发现整个集群。
- **`Topic`**：写入的目标 Topic。Topic 不存在时是否自动创建，取决于 Broker 的 `auto.create.topics.enable` 和 Writer 的 `AllowAutoTopicCreation`。
- **`Balancer`**：决定消息进哪个分区。`LeastBytes` 看分区负载，`Hash` 按 key 哈希，`RoundRobin` 轮询。需要同 key 有序时用 `Hash`。
- **`RequiredAcks`**：`RequireAll` 对应 `acks=all`，可靠性最高；`RequireOne` 对应 `acks=1`；`RequireNone` 对应 `acks=0`。
- **`Async`**：`false` 时 `WriteMessages` 阻塞到 Broker 确认；`true` 时立即返回，结果通过 `Completion` 回调通知。
- **`context`**：控制本次写入的总时长。即使 Writer 内部有重试，也会在 ctx 取消时尽快返回。

★ **关键认知**：`Writer` 是**并发安全**的，应该在整个进程里复用一份，而不是每次发消息都 `NewWriter`。每次新建都会重新建连、重新申请缓冲区，高频场景下连接数会迅速膨胀。

## 3.3 关闭与错误处理

```go
defer func() {
    if err := w.Close(); err != nil {
        log.Printf("关闭 Writer 失败: %v", err) // 关闭失败通常意味着有批次没发出去
    }
}()
```

`Close` 会停止后台协程并 flush 缓冲区。顺序上应该先停止接收新消息，再调用 `Close`，否则可能边关边写。

⚠ **关键认知**：`WriteMessages` 返回 nil 只代表"Broker 确认了"，不代表下游消费者处理成功。生产端的可靠性上限由 `acks` 和副本配置决定，端到端的可靠性还要看消费端提交 offset 的时机。

# 4. 第一个消费者

## 4.1 最小可运行示例

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "time"

    "github.com/segmentio/kafka-go"
)

func main() {
    r := kafka.NewReader(kafka.ReaderConfig{
        Brokers:  []string{"localhost:9092"},
        GroupID:  "hello-group", // 有 GroupID 才会走消费组和自动分配分区
        Topic:    "hello",
        MinBytes: 1,             // 至少凑够 1 字节就返回
        MaxBytes: 10e6,          // 单次最多拉 10MB
        MaxWait:  time.Second,   // 没数据时最多等 1 秒
    })
    defer r.Close()

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    for {
        m, err := r.ReadMessage(ctx)
        if err != nil {
            if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
                break // 正常退出
            }
            log.Printf("读取失败: %v", err)
            continue
        }

        fmt.Printf("partition=%d offset=%d key=%s value=%s\n",
            m.Partition, m.Offset, m.Key, m.Value)
    }
}
```

## 4.2 逐段说明

- **`Brokers`**：bootstrap 地址列表，给一个或多个都行。
- **`GroupID`**：消费组 ID。同一个组内的多个 Reader 会分摊分区；不同组各自读到全量消息。
- **`Topic`**：要消费的 Topic。一个 Reader 也可以配 `GroupTopics` 消费多个 Topic，但此时不能再设 `Topic`。
- **`MinBytes` / `MaxBytes` / `MaxWait`**：控制单次 Fetch 的批量大小和等待时间。吞吐优先就调大 `MinBytes`，延迟优先就调小 `MaxWait`。
- **`ReadMessage`**：拉取下一条消息，并**自动提交**这条消息的 offset。

## 4.3 ReadMessage 的隐式提交

这是 kafka-go 新手最容易踩的坑。

★ **关键认知**：`ReadMessage` 不是"只读"，它在返回消息后会按 `CommitInterval` 提交 offset。默认 `CommitInterval=0`，表示每读一条就同步提交。也就是说，如果你在 `ReadMessage` 返回之后才做业务处理，处理到一半崩溃，offset 可能已经提交了，这条消息就再也不会被消费——这是**至多一次**语义。

要做"处理成功后再提交"，必须换成 `FetchMessage` + `CommitMessages`：

```go
for {
    // 只拉取，不提交
    m, err := r.FetchMessage(ctx)
    if err != nil {
        break
    }

    if err := handle(m); err != nil {
        log.Printf("处理失败，不提交 offset: %v", err)
        continue // 重启或重试后会重新拉到这条消息
    }

    // 业务成功后再提交
    if err := r.CommitMessages(ctx, m); err != nil {
        log.Printf("提交 offset 失败: %v", err)
        continue
    }
}
```

⚠ **关键认知**：`CommitMessages` 提交的是"这条消息的 offset + 1"，表示这条已经处理完。批量提交时传最后一条消息即可，它会一次性提交到该位置。不要在处理失败时调用提交，否则就是主动丢消息。

# 5. 常用配置速查

## 5.1 Writer 关键字段

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `Addr` | 必填 | Broker 地址，`kafka.TCP(...)` |
| `Topic` | 必填 | 目标 Topic |
| `Balancer` | `RoundRobin` | 分区选择策略 |
| `RequiredAcks` | `RequireNone` | 建议显式设成 `RequireAll` |
| `Async` | `false` | 是否异步发送 |
| `BatchSize` | 100 | 单批最多多少条 |
| `BatchTimeout` | 1s | 攒批最长时间 |
| `Compression` | 无 | `Gzip`、`Snappy`、`Lz4`、`Zstd` |
| `MaxAttempts` | 10 | 失败重试次数 |
| `WriteBackoffMin` | 100ms | 重试初始退避 |

## 5.2 Reader 关键字段

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `Brokers` | 必填 | bootstrap 地址 |
| `GroupID` | 空 | 为空则不使用消费组 |
| `Topic` | 与 `GroupTopics` 二选一 | 单个 Topic |
| `StartOffset` | `FirstOffset` | 无提交记录时从哪读 |
| `MinBytes` | 1 | 单次 Fetch 最小字节 |
| `MaxBytes` | 1MB | 单次 Fetch 最大字节 |
| `MaxWait` | 10s | 无数据时最长等待 |
| `CommitInterval` | 0 | 0 表示每读完一条就提交 |
| `SessionTimeout` | 30s | 消费组会话超时 |

## 5.3 Context 与超时

```go
// 单次写入最多等 5 秒
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
err := w.WriteMessages(ctx, msg)

// 消费循环用一个可取消的 ctx，退出时 cancel
ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer cancel()
for {
    if _, err := r.ReadMessage(ctx); err != nil {
        break // ctx 取消后 ReadMessage 会返回
    }
}
```

★ **关键认知**：`ReadMessage` 会阻塞等待消息，`MaxWait` 是它单次 Fetch 的等待上限，不是整个调用的超时。真正控制生命周期的是传入的 `context`。优雅退出时先 `cancel()`，再 `Close()` Reader，让正在处理的批次有机会收尾。

# 6. 常见错误与排查

| 现象 | 常见原因 | 处理 |
|------|----------|------|
| `connection refused` | Broker 没起、端口不对 | 先 `kafka-topics.sh` 验证 |
| `UnknownTopicOrPartition` | Topic 不存在 | 先建 Topic，或确认自动创建 |
| `context deadline exceeded` | 超时太短、Broker 慢 | 看 Broker 负载和网络 |
| 读不到历史消息 | 显式设了 `LastOffset`，或 Java 客户端默认 latest | 设成 `FirstOffset`，或确认 offset 已提交 |
| offset 提交失败 | 组在 rebalance、coordinator 切换 | 重试并观察日志 |

⚠ **关键认知**：`Reader` **不是并发安全的**。不要多个 goroutine 同时调用 `ReadMessage`；需要并发处理时，让一个 goroutine 负责读，再把消息投递给 worker pool。`Writer` 则相反，可以放心被多个 goroutine 共用。

# 7. 动手练习

1. 跑通上面的生产者，用 `kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic hello --from-beginning` 验证消息确实写进去了。
2. 把 `GroupID` 改成两个不同的值，各起一个消费者，观察两组都能收到同一条消息；再改成相同值起两个消费者，观察分区被分摊。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 客户端选型 | kafka-go 简洁，Sarama 功能全 |
| 安装 | `go get github.com/segmentio/kafka-go` |
| Writer | 复用一份，`RequireAll` 提高可靠性 |
| Reader | 有 `GroupID` 才走消费组 |
| 隐式提交 | `ReadMessage` 会自动提交，至少一次要用 `FetchMessage` |
| 关闭 | 先停写入/取消 ctx，再 `Close` |
| 并发 | Writer 安全，Reader 不安全 |

下一篇深入生产者，讲同步与异步的取舍、分区键怎么选、批处理和压缩怎么配、重试与幂等生产者到底能解决什么问题。
