"Kafka 能保证 exactly-once 吗？"这个问题有一半是营销，一半是工程。Kafka 的事务确实能让"消费 - 处理 - 生产"在 Kafka 内部原子完成，但它管不到你的数据库、缓存和第三方接口。真实系统里，端到端的"恰好一次"通常等于"至少一次投递 + 业务幂等"。本篇把三种语义、幂等生产者、事务边界和去重表讲清楚，让你知道哪些能靠 Kafka，哪些必须自己兜。

# 1. 三种投递语义

## 1.1 At-most-once

```text
拉取消息 → 提交 offset → 处理业务
                │
                └── 崩溃：offset 已提交，业务没做 → 消息丢失
```

最多一次：消息可能丢，但不会重复。实现方式是"先提交、后处理"。

适用场景：可以丢的日志、指标、低价值通知。业务消息一般不能用。

## 1.2 At-least-once

```text
拉取消息 → 处理业务 → 提交 offset
                │
                └── 崩溃：业务已做，offset 没提交 → 消息重复
```

至少一次：消息不会丢，但可能重复。实现方式是"处理成功后再提交 offset"。

这是 Kafka 消费端的默认语义，也是绝大多数业务系统的实际选择。代价是消费逻辑必须能承受重复。

## 1.3 Exactly-once

"恰好一次"要同时解决"生产端重试不重复"和"消费端处理结果与 offset 原子提交"两个问题。Kafka 的方案是幂等生产者（PID + 分区序列号，Broker 去重）、事务（多分区写入和 offset 提交放进同一个事务）和 `read_committed`（消费者只看已提交事务的消息）三者配合。

★ **关键认知**：Kafka 的 exactly-once 是**流处理语义**，不是"业务只执行一次"。它保证"Kafka 内的读写和 offset 原子"，不保证你调用的数据库、HTTP 接口只执行一次。跨系统仍然要靠幂等。

## 1.4 对比表

| 语义 | 丢消息 | 重复 | 实现成本 | 典型场景 |
|------|--------|------|----------|----------|
| At-most-once | 可能 | 不会 | 低 | 日志、指标 |
| At-least-once | 不会 | 可能 | 中 | 订单、支付、业务事件 |
| Exactly-once（Kafka 内） | 不会 | 不会 | 高 | 流处理、CDC 管道 |
| 端到端恰好一次 | 不会 | 不会 | 很高 | 需要幂等 sink 配合 |

# 2. 生产者侧可靠性

## 2.1 acks 与 min.insync.replicas

| 配置 | 行为 | 风险 |
|------|------|------|
| `acks=0` | 发出即成功 | Broker 没收到也不知道 |
| `acks=1` | leader 写入即成功 | leader 崩溃可能丢 |
| `acks=all` | 所有 ISR 写入才成功 | 延迟高，但最可靠 |

`acks=all` 并不自动等于"写到了多数副本"。如果 ISR 只剩 leader，它也只写了一份。要防丢，需要三者同时满足：

```text
replication.factor >= 3
min.insync.replicas >= 2
acks = all
```

这样即使挂掉一个 Broker，还剩两个副本；如果可用副本少于 2，生产会直接失败（`NotEnoughReplicas`），而不是静默写入单副本。

★ **关键认知**：`min.insync.replicas` 是**Topic 级别**的配置，`acks` 是**生产者级别**的配置。两者必须配合，缺一不可。只设 `acks=all` 而 `min.insync.replicas=1`，在只剩 leader 时仍然会"成功"。

## 2.2 幂等生产者

幂等生产者给每个生产者分配一个 **PID（Producer ID）**，并为每个分区维护递增的序列号：

```text
Producer 启动 → 向 Broker 申请 PID
发送消息 → (PID, Partition, Sequence=1,2,3...)
Broker 按分区记录最大序列号
收到重复序列号 → 直接丢弃，返回成功
```

Java 客户端通过 `enable.idempotence=true` 开启，并自动要求 `acks=all`、`retries>0`、`max.in.flight.requests.per.connection<=5`。Sarama 的等价配置：

```go
cfg := sarama.NewConfig()
cfg.Producer.RequiredAcks = sarama.WaitForAll
cfg.Producer.Retry.Max = 5
cfg.Producer.Idempotent = true
cfg.Net.MaxOpenRequests = 1 // 幂等模式下 Sarama 要求单请求在途
cfg.Producer.Return.Successes = true
```

幂等生产者能解决"生产端重试导致的重复"，但有边界：

- 只保证**单分区、单生产者会话**内不重复。
- 进程重启后 PID 变化，Broker 会当成新生产者。
- 不保证跨 Topic、跨分区的事务原子性。

## 2.3 事务

事务在幂等的基础上增加了 `transactional.id` 和事务协调器：`initTransactions` 申请 PID 并提升 epoch、fencing 旧实例；`beginTransaction` 后发送消息；`sendOffsetsToTransaction` 把消费 offset 纳入事务；最后 `commitTransaction` 或 `abortTransaction` 原子提交或回滚。消费者设置 `isolation.level=read_committed` 后，只能看到已提交事务的消息。

★ **关键认知**：事务的原子范围只包括**Kafka 内部的写入和 offset 提交**。它不能把"写 MySQL"纳入同一个事务。所谓"Kafka 事务 + 数据库"的 exactly-once，实际做法是让数据库写入幂等，或者用 Outbox 模式。

## 2.4 kafka-go 的限制

kafka-go 支持常规生产和消费，但**不支持幂等生产者和事务**；Sarama 两者都支持。如果业务必须用 Kafka 事务，kafka-go 不是合适的选择。但大多数 Go 业务服务并不需要事务，用"至少一次 + 消费端幂等"更简单，也更容易排查。

# 3. 消费者侧幂等

## 3.1 为什么消费端必须幂等

即使生产端开了幂等、消费端用了事务，下面这些情况仍会产生重复：

- 业务处理成功，提交 offset 前进程崩溃。
- rebalance 把未提交的分区转给新消费者。
- 生产端超时重试（未开幂等时）。
- 人为重置 offset 回放历史。

所以幂等不是"可选优化"，而是**至少一次语义下的必需品**。

## 3.2 去重表

最通用的方案是在业务库里建一张去重表，把"插入去重记录"和"执行业务写"放在同一个本地事务里：

```sql
CREATE TABLE processed_message (
    consumer_group VARCHAR(64) NOT NULL,
    message_id     VARCHAR(64) NOT NULL,
    processed_at   TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (consumer_group, message_id)
);
```

```go
func handle(ctx context.Context, db *sql.DB, group string, m kafka.Message) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback() // Commit 后再 Rollback 是空操作

    // 去重记录和业务写必须在同一事务
    res, err := tx.ExecContext(ctx,
        `INSERT INTO processed_message (consumer_group, message_id)
         VALUES ($1, $2) ON CONFLICT DO NOTHING`,
        group, string(m.Key))
    if err != nil {
        return err
    }
    if n, _ := res.RowsAffected(); n == 0 {
        return nil // 已处理过
    }
    if err := doBusiness(ctx, tx, m); err != nil {
        return err // 回滚会撤销去重记录
    }
    return tx.Commit()
}
```

★ **关键认知**：去重记录必须和业务写在同一事务里。先插去重记录、再单独写业务，中间崩溃会留下"标记已处理但业务没做"的脏数据；反过来先写业务、再插标记，重复消息会再次执行业务。只有同一事务才能保证两者同生共死。

## 3.3 状态机与唯一约束

不是所有业务都适合建去重表。有状态流转的业务可以让操作天然幂等，比如 `UPDATE inventory SET reserved = reserved + 1 WHERE order_id = $1 AND status = 'PENDING'`，行数为 0 说明已处理。插入型业务可以加唯一约束（如 `UNIQUE (order_id)`），重复插入捕获冲突后当作已处理。

| 方案 | 优点 | 缺点 |
|------|------|------|
| 去重表 | 通用、可审计 | 多一张表，写入放大 |
| 状态机 | 无额外表 | 只适合有状态流转的业务 |
| 唯一约束 | 简单、数据库保证 | 只适合插入型业务 |
| Redis SETNX | 快 | 持久化和原子性弱 |

## 3.4 Redis 去重的边界

```go
ok, _ := rdb.SetNX(ctx, "processed:order:"+orderID, "1", 24*time.Hour).Result()
if !ok {
    return nil // 已处理
}
if err := doBusiness(ctx); err != nil {
    rdb.Del(ctx, "processed:order:"+orderID) // 失败撤销标记
    return err
}
```

⚠ **关键认知**：Redis 去重不是事务性的。`SetNX` 成功、业务失败、进程崩溃在 `Del` 之前，标记会残留，导致消息被误判为已处理。Redis 只能当"快速挡板"，最终一致性要靠数据库唯一约束或去重表兜底。

## 3.5 幂等键设计

幂等键要用业务唯一 ID（订单号、支付流水号、事件 ID），不要用 offset。它必须在重试、重放、迁移后保持稳定，长度可控，并且带上消费组维度，避免不同组互相影响。

# 4. 事务边界

## 4.1 本地事务与消息

最常见的错误是"双写"：数据库 `COMMIT` 之后再 `kafka.Send`，如果发送失败，数据库有订单、Kafka 没事件；反过来先发 Kafka 再写库，则可能"事件发了、订单没落库"。两个系统没有共同事务，任何顺序都有窗口。

## 4.2 事务消息

Kafka 事务能解决"Kafka 内部多分区写入 + offset"的原子性，典型模式是 consume-transform-produce：消费 input、处理、生产 output、`sendOffsetsToTransaction` 提交 input offset、`commitTransaction`。这样 output 的写入和 input 的 offset 要么都成功，要么都回滚，但仍然不包含外部数据库。

## 4.3 Outbox 模式

跨系统最实用的方案是 **Outbox（发件箱）**：

```text
同一本地事务：INSERT INTO orders ...  +  INSERT INTO outbox(event_id, payload, status='NEW')
        │
        ▼
独立发布者（轮询或 Debezium CDC）：读 outbox → 发 Kafka → 标记 SENT
```

优点：业务写和事件记录在同一个数据库事务里，不会出现"订单成功、事件丢失"。缺点：发布是至少一次，消费端仍要幂等。

★ **关键认知**：Outbox 把"分布式事务"降级成了"本地事务 + 至少一次投递 + 消费幂等"。这不是偷懒，而是工程上更可控的选择。真正的跨系统两阶段提交成本高、可用性差，大多数业务不值得。

## 4.4 取舍

| 方案 | 一致性 | 复杂度 | 适用 |
|------|--------|--------|------|
| 先写库再发消息 | 弱，可能丢事件 | 低 | 可容忍不一致 |
| 先发消息再写库 | 弱，可能发空事件 | 低 | 事件可补偿 |
| Kafka 事务 | Kafka 内原子 | 中 | 流处理管道 |
| Outbox | 最终一致 | 中 | 业务事件发布 |
| 两阶段提交 | 强一致 | 很高 | 极少使用 |

# 5. 端到端 exactly-once

## 5.1 Kafka 内部链路

如果整条链路都在 Kafka 内（消费 → 转换 → 再写入 Kafka），可以用 `read_committed` 消费 + 事务性生产 + `sendOffsetsToTransaction` 做到 exactly-once。这依赖事务性生产者的 `transactional.id`、Broker 的事务协调器和消费者的 `read_committed`。Kafka Streams 把这套封装好了，手写客户端要小心 epoch fencing 和超时。

## 5.2 跨系统链路

一旦写出 Kafka，进入数据库、缓存、HTTP 接口，就不存在"Kafka 保证 exactly-once"这回事。可行的是：

- **幂等 sink**：数据库唯一约束、`UPSERT`、状态机。
- **去重表**：记录已处理的 message ID。
- **可重放**：出错时能从 offset 重放，且重复执行无副作用。

## 5.3 工程建议

1. 默认选**至少一次 + 消费幂等**，不要一上来就上事务。
2. 生产端 `acks=all`、`min.insync.replicas>=2`，先把"不丢"做好。
3. 幂等键用业务 ID，不要用 offset 或随机 UUID。
4. 跨系统发布用 Outbox，不要双写。
5. 真需要 Kafka 事务时，评估客户端支持（kafka-go 不支持）和运维成本。

# 6. 常见错误与坑

| 现象 | 原因 | 处理 |
|------|------|------|
| 重复消费导致重复扣款 | 消费不幂等 | 去重表/状态机/唯一约束 |
| 开了事务仍有重复 | 事务只覆盖 Kafka | 外部系统做幂等 |
| 幂等生产者重启后重复 | PID 变了 | 用业务幂等兜底 |
| 去重表有记录但业务没做 | 去重和业务不同事务 | 放同一事务 |
| 消息发了但订单没落库 | 双写顺序问题 | Outbox |
| `read_committed` 读不到数据 | 事务未提交/已回滚 | 检查事务边界 |
| 去重表无限增长 | 没有清理策略 | 按时间分区、定期归档 |
| Redis 标记残留 | SetNX 后业务失败 | 失败撤销 + 数据库兜底 |

⚠ **关键认知**：去重表会随着消息量增长。生产上要按时间分区或定期归档，否则它会变成新的性能瓶颈。保留时间要覆盖"消息可能被重放的最长窗口"。

# 7. 动手练习

1. 写一个"处理成功但提交 offset 前 panic"的消费者，重启后观察重复消息，再用 `(consumer_group, message_id)` 去重表消除重复。
2. 把去重记录和业务写拆成两个事务，模拟中间崩溃，观察"标记存在但业务没做"的脏数据。
3. 用 Outbox 模式实现"下单 + 发事件"：本地事务写 orders 和 outbox，另一个 goroutine 轮询 outbox 发 Kafka，验证数据库回滚时不会发出事件。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 三种语义 | 至多一次会丢，至少一次会重，恰好一次最难 |
| 生产端防丢 | `acks=all` + `min.insync.replicas>=2` + RF>=3 |
| 幂等生产者 | PID + 序列号，单分区单会话有效 |
| 事务 | 只覆盖 Kafka 读写和 offset，不含外部系统 |
| 消费幂等 | 去重表/状态机/唯一约束，和业务同一事务 |
| Outbox | 本地事务 + 至少一次发布 + 消费幂等 |
| 端到端 | 不要追求字面 exactly-once，追求"重复无副作用" |

下一篇进入 Schema 与序列化，讲 JSON、Protobuf、Avro 的取舍，Schema Registry 如何管理版本，以及消息字段增删时怎样做到向前向后兼容。
