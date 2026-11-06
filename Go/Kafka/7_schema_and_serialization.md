生产者往消息里塞一段 JSON，消费者按自己的结构体反序列化——这是最常见的 Kafka 用法，也是最容易在版本升级时炸掉的地方。加个字段、改个类型、把 `int` 换成 `int64`，上下游没对齐就会解析失败或静默丢字段。本篇讲 JSON、Protobuf、Avro 的取舍，Schema Registry 如何管理版本，以及消息演进时哪些改动安全、哪些必须停机。

# 1. 为什么需要 Schema

## 1.1 隐式契约的问题

没有 Schema 时，生产者和消费者之间只有"口头约定"：

```go
// 生产者 v1
type OrderCreated struct {
    OrderID string `json:"order_id"`
    Amount  int64  `json:"amount"`
}

// 消费者仍按 v1 解析，生产者却悄悄改了字段名
type OrderCreated struct {
    OrderID string `json:"orderId"` // 字段名变了，消费者拿到空值
}
```

常见事故：

- 生产者把 `amount` 从整数改成小数，消费者解析失败。
- 生产者删了一个字段，消费者还在用，拿到零值继续跑。
- 两个团队对同一个 Topic 的理解不一致，直到上线才发现。
- 没有文档，新人只能靠猜。

## 1.2 Schema 的职责

Schema 做三件事：

- **定义结构**：字段名、类型、是否必填、默认值。
- **约束演进**：哪些改动兼容，哪些不兼容，由工具检查。
- **生成代码**：从 Schema 生成 Go 结构体，减少手写错误。

★ **关键认知**：Schema 的价值不在"序列化更快"，而在**契约可验证**。它让"生产者改字段"从口头沟通变成一次兼容性检查，问题在发布前暴露，而不是在凌晨的告警里暴露。

# 2. 三种序列化格式

## 2.1 JSON

Go 标准库 `encoding/json` 就能用，调试友好，跨语言无门槛：

```go
type OrderCreated struct {
    OrderID  string `json:"order_id"`
    Amount   int64  `json:"amount"`
    Currency string `json:"currency,omitempty"`
}

payload, err := json.Marshal(OrderCreated{
    OrderID:  "1001",
    Amount:   9900,
    Currency: "CNY",
})
```

优点：可读、工具多、无需代码生成。缺点：体积大、类型弱、数字精度容易出问题。最典型的坑是用 `map[string]interface{}` 接收，JSON 数字默认变成 `float64`，大整数会丢精度；要用结构体或 `json.Decoder.UseNumber()`。

## 2.2 Protobuf

先写 `.proto`：

```protobuf
syntax = "proto3";

package events;

message OrderCreated {
  string order_id = 1;
  int64  amount   = 2;
  string currency = 3;
}
```

用 `protoc --go_out=. --go_opt=paths=source_relative order.proto` 生成代码后序列化：

```go
msg := &events.OrderCreated{
    OrderId:  "1001",
    Amount:   9900,
    Currency: "CNY",
}
payload, err := proto.Marshal(msg)

var out events.OrderCreated
err = proto.Unmarshal(payload, &out)
```

★ **关键认知**：Protobuf 的兼容性靠**字段编号**，不靠字段名。编号一旦发布就不能复用，删除字段要用 `reserved 4, 5;` 和 `reserved "old_field";` 标记。改字段名是安全的，改编号等于换了一个字段。

## 2.3 Avro

Avro 用 JSON 描述 Schema，数据本身是紧凑的二进制：

```json
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "events",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "amount", "type": "long"},
    {"name": "currency", "type": "string", "default": "CNY"}
  ]
}
```

Avro 的最大特点是**读时解析**：数据文件或消息里带 writer schema，消费者用 reader schema 解析，两者按规则做字段映射。这让它非常适合 Schema Registry 和长期演进。

优点：体积小、Schema 自描述、演进规则成熟。缺点：二进制不可读、Go 生态不如 Protobuf 丰富、需要 Registry 配合。

## 2.4 对比表

| 维度 | JSON | Protobuf | Avro |
|------|------|----------|------|
| 可读性 | 好 | 差 | 差 |
| 体积 | 大 | 小 | 最小 |
| Schema | 可选/外部 | `.proto` | JSON Schema |
| 兼容性检查 | 弱 | 靠字段编号 | 内置规则 |
| 代码生成 | 不需要 | 需要 | 需要 |
| Go 支持 | 标准库 | 官方 | 第三方 |
| 适合 | 调试、简单事件 | RPC、强类型事件 | 数据管道、Registry |

选型建议：内部服务、字段稳定、追求性能 → Protobuf；数据湖、CDC、长期演进 → Avro；快速验证、日志类事件 → JSON。

# 3. Schema Registry

## 3.1 架构

```text
Producer → 向 Registry 注册/查询 schema → 序列化 magic + ID + payload → Kafka
Consumer → 读到 magic + ID → 按 ID 查 Registry → 反序列化
```

Registry 本身是一个 HTTP 服务，存的是"Subject → 版本 → Schema"的映射。Kafka 只存序列化后的字节，不参与 Schema 校验。

## 3.2 Subject 与版本

默认的 Subject 命名策略是 `TopicNameStrategy`：

- value 的 Subject 是 `<topic>-value`。
- key 的 Subject 是 `<topic>-key`。

同一个 Subject 下可以有多个版本，每个 Schema 有一个全局唯一的 ID。比如 `orders-value` 的 v1、v2、v3 分别对应 schema ID 101、102、103。生产者发送时带的是 **ID**，不是版本号。

## 3.3 兼容性模式

| 模式 | 含义 | 升级顺序 |
|------|------|----------|
| `BACKWARD` | 新 Schema 能读旧数据 | 先升级消费者，再升级生产者 |
| `FORWARD` | 旧 Schema 能读新数据 | 先升级生产者，再升级消费者 |
| `FULL` | 双向兼容 | 任意顺序，但改动限制最严 |
| `NONE` | 不检查 | 危险，只用于临时环境 |

★ **关键认知**：兼容性模式决定的是"**谁能读谁**"，不是"哪个版本更新"。`BACKWARD` 允许你删字段、加带默认值的字段，因为新消费者能读旧数据；`FORWARD` 允许你加字段，因为旧消费者能忽略新字段。选错模式，升级顺序就会反过来。

## 3.4 线格式

Confluent 系的序列化格式通常是 1 字节 magic（`0x00`）+ 4 字节大端 schema ID + 序列化 payload。Go 里可以这样拆：

```go
func splitPayload(b []byte) (int, []byte, error) {
    if len(b) < 5 || b[0] != 0 {
        return 0, nil, errors.New("非法的 Schema Registry 消息")
    }
    return int(binary.BigEndian.Uint32(b[1:5])), b[5:], nil
}

func wrapPayload(id int, payload []byte) []byte {
    buf := make([]byte, 5+len(payload))
    binary.BigEndian.PutUint32(buf[1:5], uint32(id))
    copy(buf[5:], payload)
    return buf
}
```

Protobuf 的 Confluent 格式还会在 ID 后面加 message index 数组，解析时要按对应规范处理。

# 4. Go 实践

## 4.1 JSON 封装

```go
type Envelope struct {
    EventID    string          `json:"event_id"`
    EventType  string          `json:"event_type"`
    Version    int             `json:"version"`
    OccurredAt time.Time       `json:"occurred_at"`
    Data       json.RawMessage `json:"data"` // 延迟解析具体事件
}
```

`json.RawMessage` 让信封和业务数据分开解析，方便按 `event_type` 路由到不同的结构体；解析后再校验 `EventID`、`EventType` 非空。

## 4.2 Protobuf 示例

```go
payload, err := proto.Marshal(&events.OrderCreated{OrderId: "1001", Amount: 9900})

var out events.OrderCreated
err = proto.Unmarshal(payload, &out) // 消费者侧

// 发送时把 schema 版本放进 header，便于排查
w.WriteMessages(ctx, kafka.Message{
    Key:   []byte(out.OrderId),
    Value: payload,
    Headers: []kafka.Header{
        {Key: "content-type", Value: []byte("application/x-protobuf")},
        {Key: "schema-id", Value: []byte("101")},
    },
})
```

## 4.3 Registry 客户端

生产上建议用成熟客户端，比如 `github.com/riferrei/srclient`，或者 Confluent 官方客户端。核心流程只有三步：

```text
1. 启动时把 schema 注册到 Registry，拿到 ID（或查询已有 ID）
2. 发送时先写 magic + ID，再写 payload
3. 消费时读 ID，从本地缓存或 Registry 取 schema，再反序列化
```

Registry 不可用时，消费者应该用本地缓存的 Schema 继续工作，而不是直接崩溃。缓存要按 ID 存，因为 ID 是全局稳定的。

⚠ **关键认知**：不要让每条消息都同步请求 Registry。Schema ID 到 Schema 的映射变化很少，应该在进程内缓存；否则 Registry 会成为整个链路的新瓶颈，也会把它的故障放大到所有消费者。

# 5. 消息演进

## 5.1 兼容的变更

| 变更 | JSON | Protobuf | Avro |
|------|------|----------|------|
| 加可选字段 | 兼容（消费者忽略未知） | 兼容（新编号） | 兼容（带默认值） |
| 删可选字段 | 可能兼容 | 兼容（保留编号） | 兼容（有默认值） |
| 改字段名 | 不兼容 | 兼容（编号不变） | 不兼容（除非 alias） |
| `int` → `int64` | 可能丢精度 | 部分兼容 | 兼容（promotion） |
| 加必填字段 | 不兼容 | 不兼容 | 不兼容（无默认值） |
| 改字段类型 | 不兼容 | 视 wire type 而定 | 多数不兼容 |

## 5.2 不兼容的变更

- 删除必填字段，或增加没有默认值的必填字段。
- 修改字段类型（`string` → `int`）。
- 重命名字段但不提供 alias。
- 复用 Protobuf 已删除的字段编号。
- 改变枚举值的含义。

⚠ **关键认知**：JSON 的"兼容"是**运行时宽容**，不是编译期保证。多一个字段通常没事，少一个字段可能拿到零值继续跑，直到业务出错。不要因为"JSON 不会报错"就跳过兼容性评审。

## 5.3 版本化策略

1. **Schema Registry 管版本**：用 `BACKWARD` 或 `FULL`，让不兼容改动静默失败在 CI。
2. **消息带版本**：header 里放 `schema-id` 或 `version`，消费端可以按版本分支处理。
3. **双写过渡**：新旧字段同时写一段时间，消费者先读新字段、回退旧字段，确认后再删旧字段。
4. **不要复用字段编号/名称**：Protobuf 用 `reserved`，Avro 用 alias，JSON 保留旧字段一段时间。
5. **消费者要容忍未知字段**：Go 的 `json.Unmarshal` 默认忽略未知字段，Protobuf 也会跳过未知编号，这是演进的基础。

# 6. 常见错误与坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 大整数变成科学计数法 | JSON 默认解成 `float64` | 用结构体或 `UseNumber` |
| 时间格式不统一 | 字符串/RFC3339/时间戳混用 | 约定 RFC3339 或毫秒 |
| 字段名改了消费者拿空值 | JSON 字段名不匹配 | 用 alias/双写过渡 |
| Protobuf 反序列化错乱 | 字段编号被复用 | 删除字段加 `reserved` |
| Avro 读旧数据失败 | 新字段没有默认值 | 加字段必须给 default |
| Registry 挂了全链路失败 | 每条消息同步查 Registry | 本地缓存 Schema |
| Subject 不匹配 | 自定义命名策略 | 统一 `TopicNameStrategy` |
| 消息体过大 | Schema 冗余/嵌套过深 | 拆分事件、压缩 |

★ **关键认知**：Schema 演进的第一原则是"**只加不改**"。加字段、加可选值、加新事件类型通常安全；改类型、改语义、删字段风险最高。真要做不兼容变更，就发新 Topic 或新事件类型，让新旧链路并行一段时间。

# 7. 动手练习

1. 用 JSON 发送一个包含 `int64` 大整数的消息，分别用 `map[string]interface{}` 和结构体反序列化，观察精度差异。
2. 写一个 `.proto`，生成 Go 代码，发送并消费一条 Protobuf 消息；再删掉一个字段并用 `reserved` 保留编号，验证旧消费者仍能解析。
3. 用 `wrapPayload` / `splitPayload` 手动实现 Schema Registry 的 5 字节头，模拟"生产者写 ID、消费者按 ID 取 Schema"的完整链路。

# 8. 小结

| 主题 | 要点 |
|------|------|
| Schema 的作用 | 定义契约、约束演进、生成代码 |
| JSON | 可读、通用，但类型弱、体积大 |
| Protobuf | 紧凑、强类型，靠字段编号兼容 |
| Avro | 体积最小、演进规则成熟，适合 Registry |
| Registry | 存 Subject/版本，消息里带 schema ID |
| 兼容模式 | BACKWARD/FORWARD/FULL 决定升级顺序 |
| 演进原则 | 只加不改，删字段先保留编号 |
| Go 实践 | 缓存 Schema，别每条消息查 Registry |

下一篇进入运维、监控与安全，讲 Broker 该看哪些指标、lag 怎么监控、磁盘怎么估算、分区怎么规划，以及 TLS、SASL、ACL 在 Go 客户端里怎么配。
