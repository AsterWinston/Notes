
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，涵盖优先级队列、惰性队列、仲裁队列、备用交换机、流量控制、消费者预取等高级特性。

---

# 1. 优先级队列（Priority Queue）

## 1.1 概念

默认队列先进先出（FIFO）。启用优先级后，消息按**优先级从高到低**出队。

```
普通队列（FIFO）:     H → M → L → 出队: H → M → L
优先级队列:           H(m), L(m), M(m) → 出队: H → M → L
```

## 1.2 声明与使用

```java
Map<String, Object> args = new HashMap<>();
args.put("x-max-priority", 10);  // 最大优先级为 10（0~10）
channel.queueDeclare("priority.queue", true, false, false, args);

// 发送不同优先级的消息
AMQP.BasicProperties highPriority = new AMQP.BasicProperties.Builder()
        .priority(10).build();
AMQP.BasicProperties lowPriority = new AMQP.BasicProperties.Builder()
        .priority(1).build();

channel.basicPublish("", "priority.queue", highPriority, "VIP消息".getBytes());
channel.basicPublish("", "priority.queue", lowPriority, "普通消息".getBytes());
```

## 1.3 性能影响

| 维度 | 普通队列 | 优先级队列 |
|------|:---:|:---:|
| 入队性能 | 高 | 略降低（需要排序插入位置） |
| 出队性能 | 高 | 降低（需要扫描优先级） |
| 内存消耗 | 低 | 增加（内部使用多个子队列） |
| 建议优先级范围 | N/A | 0~5 性能较好，0~10 可用，不建议更大 |

> **注意**：如果消费者足够快，优先级的优势不明显（队列不堆积时，所有消息立即被消费）。

---

# 2. 惰性队列（Lazy Queue）

## 2.1 概念

普通队列的消息尽可能存储在**内存**中，内存不足时换出到磁盘。惰性队列的消息**尽可能存储在磁盘**中，仅在消费时加载到内存。

```
普通队列：消息 → 内存（不足→磁盘）
惰性队列：消息 → 磁盘（消费时→内存）
```

## 2.2 声明

```java
Map<String, Object> args = new HashMap<>();
args.put("x-queue-mode", "lazy");  // 3.6~3.7 方式
// 或使用新方式（推荐）
args.put("x-queue-type", "classic");  // 经典队列已支持 lazy 参数

channel.queueDeclare("lazy.queue", true, false, false, args);
```

## 2.3 适用场景

| 场景 | 说明 |
|------|------|
| **消息堆积** | 内存资源有限，需要大量消息堆积在磁盘 |
| **不稳定消费者** | 消费速度波动大，担心消息占据过多内存触发流控 |
| **大消息** | 消息体较大，不适合长期在内存 |

## 2.4 Lazy Queue vs Normal Queue

| 维度 | 普通队列 | 惰性队列 |
|------|----------|----------|
| 消息存储 | 内存为主 | 磁盘为主 |
| 内存消耗 | 较高 | 较低 |
| 消费延迟 | 低（内存命中） | 较高（磁盘读取） |
| 吞吐量 | 高 | 中 |
| 适用场景 | 实时消费、消息不堆积 | 大堆积量、内存资源紧张 |

---

# 3. 仲裁队列（Quorum Queue）

## 3.1 概念

Quorum Queue 是 RabbitMQ 3.8 引入的**基于 Raft 共识协议**的持久化、高可用队列类型，是 Mirror Queue 的现代替代方案。

```
                    Quorum Queue (3 Nodes)
            ┌─────────────────────────────────┐
            │  ┌────────┐            Leader   │
            │  │ Node 1 │  ◄── Leader        │
            │  └───┬────┘                    │
            │      │ Sync (Raft)              │
            │  ┌───▼────┐       Follower     │
            │  │ Node 2 │                    │
            │  └───┬────┘                    │
            │      │ Sync (Raft)              │
            │  ┌───▼────┐       Follower     │
            │  │ Node 3 │                    │
            │  └────────┘                    │
            └─────────────────────────────────┘
    写入：Leader 接收写入 → 复制到多数 Follower → 确认写入
```

## 3.2 声明

```java
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
channel.queueDeclare("order.quorum.queue", true, false, false, args);
```

## 3.3 Quorum Queue 特性

| 特性 | 说明 |
|------|------|
| **数据安全** | 基于 Raft，写入需多数节点确认才能成功 |
| **自动 Leader 选举** | Leader 宕机自动选举新 Leader |
| **消息去重** | 支持消息去重（需唯一 message-id） |
| **死信支持** | 支持 DLX |
| **局限性** | 不支持 TTL（消息级别）、不支持优先级、不支持 Exclusive |

## 3.4 Quorum Queue vs Mirror Queue

| 维度 | Quorum Queue (3.8+) | Mirror Queue (传统) |
|------|---------------------|---------------------|
| 一致性协议 | **Raft**（强一致） | 非共识（异步镜像） |
| 写入确认 | 多数派确认才返回 | 可能异步复制 |
| 故障恢复 | 自动 Leader 选举 | 最旧镜像被提升（可能丢数据） |
| 成员变更 | 支持动态添加删除 | 需修改 Policy |
| 性能 | 中（Raft 同步有开销） | 高（可异步） |
| 官方推荐 | ✅ 推荐 | ⚠️ 3.13 起标记为不推荐 |

---

# 4. 备用交换机（Alternate Exchange）

## 4.1 概念

当主交换机无法路由消息（无匹配 Queue）时，消息转发到备用交换机，而不是直接丢弃。

```
┌───────────┐  无法路由  ┌────────────────┐
│ 主交换机   │──────────→│ 备用交换机(AE)  │
│(order.dir)│            │ (order.ae)     │
└───────────┘            └──────┬─────────┘
                                │
                         ┌──────▼──────┐
                         │ order.ae.q  │
                         └─────────────┘
```

## 4.2 代码实现

```java
// 声明备用交换机参数
Map<String, Object> exchangeArgs = new HashMap<>();
exchangeArgs.put("alternate-exchange", "order.unroutable");

// 声明主交换机（带 AE）
channel.exchangeDeclare("order.direct", BuiltinExchangeType.DIRECT, true, false, exchangeArgs);

// 声明备用交换机（Fanout，接收所有未路由消息）
channel.exchangeDeclare("order.unroutable", BuiltinExchangeType.FANOUT, true);

// 声明备用队列
channel.queueDeclare("order.unroutable.queue", true, false, false, null);
channel.queueBind("order.unroutable.queue", "order.unroutable", "");
```

## 4.3 AE vs ReturnCallback

| 方案 | 方式 | 适用场景 |
|------|------|----------|
| **ReturnCallback** | Producer 侧处理，消息退回发送方 | 需要 Producer 感知并重发 |
| **Alternate Exchange** | Broker 侧处理，自动转发到 AE | Producer 无感知，Broker 统一兜底 |

---

# 5. 流量控制（Flow Control）

RabbitMQ 使用**基于信用的流量控制机制**，防止生产者和消费者速度不匹配导致内存耗尽或积压过大。

## 5.1 Producer 侧流量控制

| 机制 | 触发条件 | 效果 |
|------|----------|------|
| 内存告警（Memory Alarm） | 内存使用超过 `vm_memory_high_watermark`（默认 0.4 = 40%） | 阻塞所有 Producer 的发布请求 |
| 磁盘告警（Disk Alarm） | 磁盘可用空间低于 `disk_free_limit` | 阻塞所有 Producer 的发布请求 |

```ini
# rabbitmq.conf 中的流控配置
vm_memory_high_watermark.relative = 0.4  # 内存使用 40% 时触发流控
disk_free_limit.absolute = 2GB           # 磁盘剩余空间 2GB 时告警
```

## 5.2 Consumer 侧流量控制 — Prefetch

```java
// 设置 Consumer Prefetch：最多同时处理多少条未确认消息
channel.basicQos(100);  // 最多 100 条未 ACK 消息

// 高级用法：按字节数限制
channel.basicQos(0, 64000, false);  // prefetchSize=0(不限制), prefetchCount=0, global=false
```

| Prefetch 值 | 行为 | 适用场景 |
|:---:|------|----------|
| **1** | 每次发一条，ACK 后再发 | 任务耗时不均，公平分发 |
| **100~300** | 批量发送 | 低延迟、高速消费 |
| **0**（默认） | 无限制 | 不在乎消息堆积但可能撑死消费者 |

---

# 6. 消费端顺序性

## 6.1 RabbitMQ 能保证消息顺序吗？

| 条件 | 是否有序 | 说明 |
|------|:---:|------|
| 单 Producer + 单 Queue + 单 Consumer | ✅ 有序 | 最简单的有序消费 |
| 单 Producer + 单 Queue + 多 Consumer | ❌ 无序 | 多消费者并发消费，顺序不可控 |
| 多 Producer + 单 Queue + 单 Consumer | ❌ 无序 | 多 Producer 发送速度不同 |
| 有重试/requeue | ❌ 无序 | requeue 的消息可能排在后面 |
| 优先级队列 | ❌ 无序 | 设计上打破 FIFO |

## 6.2 保证顺序的方案

```
方案一：全局有序
  → 单 Producer + 单 Queue + 单 Consumer + autoAck=false
  → 缺点：无法水平扩展，吞吐量低

方案二：分区有序（推荐）
  → 按业务 Key（如 orderId 哈希）分区
  → 相同 Key → 同一 Queue → 同一 Consumer
  → 全局无序，分区内有序

┌──────────┐
│ Producer │
│ order 001│──→ Queue_A {001, 004, 007} ──→ Consumer A
│ order 002│──→ Queue_B {002, 005, 008} ──→ Consumer B
│ order 003│──→ Queue_C {003, 006, 009} ──→ Consumer C
└──────────┘
```

---

# 7. 单活跃消费者（Single Active Consumer）

RabbitMQ 3.11+ 支持 **单活跃消费者** 模式：同一队列多个消费者中，只有一个处于活跃状态，其余为热备。

```java
Map<String, Object> args = new HashMap<>();
args.put("x-single-active-consumer", true);
channel.queueDeclare("sac.queue", true, false, false, args);
```

| 特性 | 说明 |
|------|------|
| **活跃消费者** | 只有一个消费者在消费 |
| **自动切换** | 活跃消费者宕机/取消 → 自动切换到另一个消费者 |
| **适用场景** | 需要顺序消费 + 高可用的场景 |

---

# 8. 消息去重（Message Deduplication）

Quorum Queue 和 Stream 支持消息去重，通过 **x-message-deduplication** 参数和唯一 **message-id** 实现。

```java
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
// 注意：deduplication 是 Quorum Queue 的全局配置，不需要额外参数

channel.queueDeclare("dedup.queue", true, false, false, args);

// 发送时设置唯一 message-id
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
        .messageId(UUID.randomUUID().toString())
        .build();

channel.basicPublish("", "dedup.queue", props, msg.getBytes());

// 再次发送相同 messageId 的消息 → RabbitMQ 忽略该消息
channel.basicPublish("", "dedup.queue", props, msg.getBytes()); // 被去重
```

---

# 9. 队列长度限制与溢出策略

```java
Map<String, Object> args = new HashMap<>();
args.put("x-max-length", 10000);              // 最多 10000 条消息
args.put("x-max-length-bytes", 104857600);     // 最多 100MB
args.put("x-overflow", "drop-head");           // 溢出策略

channel.queueDeclare("limited.queue", true, false, false, args);
```

| 溢出策略 | 行为 |
|----------|------|
| `drop-head`（默认） | 删除队列头部最早的消息，丢弃或进入死信 |
| `reject-publish` | 拒绝新的发布（生产者收到 basic.nack） |
| `reject-publish-dlx` | 拒绝新的发布 + 加入死信队列 |

---

# 10. 各高级特性速查表

| 特性 | 配置方式 | RabbitMQ 版本 | 推荐度 |
|------|----------|:---:|:---:|
| 优先级队列 | `x-max-priority=N` | 3.5+ | ⭐⭐⭐ |
| 惰性队列 | `x-queue-mode=lazy` | 3.6+ | ⭐⭐⭐⭐ |
| 仲裁队列 | `x-queue-type=quorum` | 3.8+ | ⭐⭐⭐⭐⭐ |
| 备用交换机 | `alternate-exchange` | 3.0+ | ⭐⭐⭐⭐ |
| 消费者预取 | `basicQos(N)` | 全部 | ⭐⭐⭐⭐⭐ |
| 单活跃消费者 | `x-single-active-consumer` | 3.11+ | ⭐⭐⭐ |
| 消息去重 | message-id (Quorum/Stream) | 3.8+ | ⭐⭐ |
| 队列溢出 | `x-overflow` | 3.0+ | ⭐⭐⭐ |
