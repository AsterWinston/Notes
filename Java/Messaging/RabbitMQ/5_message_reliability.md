
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，讲解消息从生产到消费全链路的可靠性保障机制及代码实现。

---

# 1. 消息可靠性全景

```
┌─────────┐           ┌───────────────┐           ┌─────────┐
│Producer │── ① ──→  │   RabbitMQ    │── ③ ──→  │Consumer │
└─────────┘           └───────────────┘           └─────────┘
     ⑦                     ⑥                        ⑤
     ① Producer → Broker: Publisher Confirm / Transaction
     ② Broker: 消息持久化写入磁盘
     ③ Broker → Consumer: 消息投递
     ④ Consumer: 业务处理
     ⑤ Consumer → Broker: Consumer ACK
     ⑥ Broker: 删除消息
     ⑦ Producer ← Broker: Confirm 成功 / 失败回调
```

**消息丢失的三个阶段**：

| 阶段 | 丢失场景 | 解决方案 |
|------|----------|----------|
| **① 生产阶段** | 网络故障、Broker 宕机 | Publisher Confirm / Transaction |
| **② Broker 阶段** | Broker 宕机、磁盘损坏 | 消息持久化 + 镜像队列 / 仲裁队列 |
| **③ 消费阶段** | Consumer 宕机前未处理完 | 手动 ACK（autoAck=false） |

---

# 2. Consumer ACK（消费者确认）

## 2.1 确认模式

| 模式 | 机制 | 可靠性 | 性能 |
|------|------|:---:|:---:|
| **自动确认（autoAck=true）** | Broker 发出消息后立即删除 | 低 | 高 |
| **手动确认（autoAck=false）** | Consumer 显式发送 basicAck 后才删除 | **高** | 中 |
| **手动 Nack** | 拒绝消息，可 requeue 或丢弃 | 高 | 中 |

## 2.2 手动 ACK / NACK 详解

```java
Channel channel = connection.createChannel();
channel.basicQos(1); // 每次只取一条
channel.basicConsume(QUEUE_NAME, false, // autoAck=false
    (consumerTag, delivery) -> {
        long deliveryTag = delivery.getEnvelope().getDeliveryTag();
        try {
            // 业务处理
            processMessage(delivery.getBody());

            // 成功 — 确认单条消息
            channel.basicAck(deliveryTag, false);
        } catch (BusinessException e) {
            // 业务异常 — 拒绝且不重回队列（进入死信）
            channel.basicNack(deliveryTag, false, false);
        } catch (Exception e) {
            // 临时异常 — 拒绝并重回队列（重试）
            channel.basicNack(deliveryTag, false, true);
        }
    },
    consumerTag -> {}
);
```

## 2.3 ACK 方法对比

| 方法 | 多消息 | 是否 Requeue | 场景 |
|------|:---:|:---:|------|
| `basicAck(deliveryTag, multiple)` | 可选 | N/A（确认） | 成功处理 |
| `basicNack(deliveryTag, multiple, requeue)` | 可选 | 可选 | 失败处理 |
| `basicReject(deliveryTag, requeue)` | 否 | 可选 | 单条拒绝 |

---

# 3. 消息持久化（Persistence）

RabbitMQ 的重启分为两种：

| 重启类型 | 说明 | 数据影响 |
|----------|------|----------|
| **应用重启** (`rabbitmqctl stop_app` + `start_app`) | 只重启 RabbitMQ 应用，Erlang VM 保持运行 | 如果进程终止，未刷新到磁盘的数据可能丢失 |
| **节点重启** (OS 重启/`rabbitmqctl shutdown`) | 完全停止 RabbitMQ 和 Erlang VM | 所有持久化数据写入磁盘后安全停止 |

## 3.1 三个层次的持久化

```
消息持久化需要三层同时满足：

1. Exchange 持久化  ─→  exchangeDeclare(name, type, durable=true)
2. Queue 持久化      ─→  queueDeclare(name, durable=true, ...)
3. Message 持久化    ─→  delivery-mode=2 (MessageProperties.PERSISTENT_TEXT_PLAIN)
```

```java
// 完整的三层持久化
channel.exchangeDeclare("order.exchange", BuiltinExchangeType.DIRECT, true);       // 1
channel.queueDeclare("order.queue", true, false, false, null);                     // 2
channel.queueBind("order.queue", "order.exchange", "order.created");

channel.basicPublish("order.exchange", "order.created",
        MessageProperties.PERSISTENT_TEXT_PLAIN,   // 3: delivery-mode=2
        message.getBytes(StandardCharsets.UTF_8));
```

> **注意**：持久化不保证消息立即写入磁盘。RabbitMQ 有一个短暂的间隔（几百毫秒）将消息刷新到磁盘。对于极高可靠场景，应使用 **Quorum Queue** 或配合 **Publisher Confirm**。

---

# 4. Publisher Confirm（发布者确认）

Publisher Confirm 是 **轻量级** 的异步确认机制，Producer 发送消息后 Broker 返回确认结果。

```
Producer                                Broker
   │                                       │
   │──── basicPublish + Confirm ──────────→│
   │                                       │ 持久化写入
   │←──── basicAck (成功) ─────────────────│
   │  或                                  │
   │←──── basicNack (失败，重新发送) ───────│
```

## 4.1 三种确认模式

| 模式 | 机制 | 性能 | 适用场景 |
|------|------|:---:|------|
| **单条确认** | 发一条等一条 | 低 | 极低并发，每条消息都很重要 |
| **批量确认** | 发一批等一批 | 中 | 中等并发 |
| **异步确认** | 发送 + 异步回调 | **高** | 高并发生产（推荐） |

## 4.2 异步 Publisher Confirm（推荐）

```java
public class PublisherConfirmAsync {
    private static final String EXCHANGE = "confirm.exchange";
    private static final String QUEUE = "confirm.queue";

    public static void main(String[] args) throws Exception {
        try (Connection connection = RabbitMQUtil.getConnection();
             Channel channel = connection.createChannel()) {

            channel.exchangeDeclare(EXCHANGE, BuiltinExchangeType.DIRECT, true);
            channel.queueDeclare(QUEUE, true, false, false, null);
            channel.queueBind(QUEUE, EXCHANGE, "confirm.key");

            // 开启 Publisher Confirm
            channel.confirmSelect();

            // 记录未确认的消息
            ConcurrentNavigableMap<Long, String> outstandingConfirms =
                    new ConcurrentSkipListMap<>();

            // 成功回调
            channel.addConfirmListener(
                (deliveryTag, multiple) -> {
                    System.out.println("[确认成功] deliveryTag=" + deliveryTag);
                    if (multiple) {
                        // 批量确认：清理 <= deliveryTag 的所有记录
                        ConcurrentNavigableMap<Long, String> confirmed =
                                outstandingConfirms.headMap(deliveryTag, true);
                        confirmed.clear();
                    } else {
                        outstandingConfirms.remove(deliveryTag);
                    }
                },
                (deliveryTag, multiple) -> {
                    System.out.println("[确认失败] deliveryTag=" + deliveryTag + " 需重发");
                    if (multiple) {
                        ConcurrentNavigableMap<Long, String> nacked =
                                outstandingConfirms.headMap(deliveryTag, true);
                        for (Map.Entry<Long, String> entry : nacked.entrySet()) {
                            // 重发失败的消息
                            retrySend(channel, EXCHANGE, "confirm.key", entry.getValue());
                        }
                        nacked.clear();
                    } else {
                        String body = outstandingConfirms.get(deliveryTag);
                        if (body != null) {
                            retrySend(channel, EXCHANGE, "confirm.key", body);
                            outstandingConfirms.remove(deliveryTag);
                        }
                    }
                }
            );

            // 发送消息并记录
            for (int i = 0; i < 100; i++) {
                String msg = "Message-" + i;
                long nextSeqNo = channel.getNextPublishSeqNo();
                outstandingConfirms.put(nextSeqNo, msg);
                channel.basicPublish(EXCHANGE, "confirm.key",
                        MessageProperties.PERSISTENT_TEXT_PLAIN,
                        msg.getBytes(StandardCharsets.UTF_8));
            }

            // 等待所有确认
            Thread.sleep(5000);
        }
    }

    private static void retrySend(Channel channel, String exchange,
                                   String routingKey, String body) {
        try {
            channel.basicPublish(exchange, routingKey,
                    null, body.getBytes(StandardCharsets.UTF_8));
            System.out.println("[重发] " + body);
        } catch (Exception e) {
            System.err.println("[重发失败] " + body + " " + e.getMessage());
        }
    }
}
```

## 4.3 Spring Boot 配置 Publisher Confirm

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated  # 确认模式
    publisher-returns: true             # 路由失败回退
```

---

# 5. 事务（Transactions）

RabbitMQ 支持 AMQP 事务（`txSelect` / `txCommit` / `txRollback`），但 **性能极差（吞吐量下降 250 倍）**，不推荐在生产中使用。

```java
// 不推荐：事务模式
channel.txSelect();
try {
    channel.basicPublish(EXCHANGE, ROUTING_KEY, null, msg.getBytes());
    channel.txCommit();
} catch (Exception e) {
    channel.txRollback();
}
```

> **强烈推荐使用 Publisher Confirm 替代事务**，性能提升显著且同样可靠。

---

# 6. 路由失败处理 — Mandatory 与 ReturnCallback

当消息发送到 Exchange 但没有任何 Queue 可以路由时：

| 设置 | 行为 |
|------|------|
| `mandatory=true` | 消息**退回给 Producer**（需要 SetReturnCallback 处理） |
| `mandatory=false`（默认） | 消息**直接丢弃** |

```java
// 设置 ReturnCallback
channel.addReturnListener((replyCode, replyText, exchange, routingKey, properties, body) -> {
    String msg = new String(body, StandardCharsets.UTF_8);
    System.out.println("[路由失败] " + msg + " → replyText=" + replyText);
    // 记录到数据库 / 日志 / 重试
});

// 发送时设置 mandatory=true
channel.basicPublish(EXCHANGE, ROUTING_KEY, true, null, msg.getBytes());
```

## 6.1 Spring Boot 中的 ReturnCallback

```java
@Configuration
public class RabbitConfig implements RabbitTemplate.ReturnsCallback {

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMandatory(true);
        template.setReturnsCallback(this);
        return template;
    }

    @Override
    public void returnedMessage(ReturnedMessage returned) {
        System.out.printf("[路由失败] msg=%s, replyCode=%d, replyText=%s, exchange=%s, routingKey=%s%n",
                new String(returned.getMessage().getBody()),
                returned.getReplyCode(),
                returned.getReplyText(),
                returned.getExchange(),
                returned.getRoutingKey());
    }
}
```

---

# 7. 高可靠方案设计

## 7.1 设计原则

```
                        ┌──────────────────────┐
                        │  持久化 Exchange      │
                        │  持久化 Queue         │
                        │  持久化 Message       │
                        │  ├─ Publisher Confirm │
                        │  ├─ Consumer Manual ACK│
                        │  └─ Mandatory+ReturnCall│
                        └──────────────────────┘
```

## 7.2 完整的高可靠 Producer 示例

```java
public class ReliableProducer {
    private final Channel channel;
    private final ConcurrentNavigableMap<Long, MessageRecord> pending =
            new ConcurrentSkipListMap<>();

    public ReliableProducer(Channel channel) throws IOException {
        this.channel = channel;
        this.channel.confirmSelect(); // 启用确认
    }

    public void send(String exchange, String routingKey, String message) throws IOException {
        long seqNo = channel.getNextPublishSeqNo();

        // 记录消息（含重试次数）
        MessageRecord record = new MessageRecord(exchange, routingKey, message, seqNo);
        pending.put(seqNo, record);

        // 使用持久化属性，mandatory=true
        channel.basicPublish(exchange, routingKey, true,
                MessageProperties.PERSISTENT_TEXT_PLAIN,
                message.getBytes(StandardCharsets.UTF_8));
    }

    public void handleConfirm(long deliveryTag, boolean multiple, boolean success) {
        // 成功：移除记录
        if (success) {
            if (multiple) {
                pending.headMap(deliveryTag, true).clear();
            } else {
                pending.remove(deliveryTag);
            }
        } else {
            // 失败：重试（最多 3 次）
            MessageRecord record = pending.get(deliveryTag);
            if (record != null && record.retryCount < 3) {
                record.retryCount++;
                // 重新发送...
            } else {
                // 超过重试次数，记录到死信 DB
                saveToDeadLetterDB(record);
            }
        }
    }

    private static class MessageRecord {
        String exchange, routingKey, body;
        long seqNo;
        int retryCount = 0;

        MessageRecord(String e, String r, String b, long s) {
            exchange = e; routingKey = r; body = b; seqNo = s;
        }
    }

    private void saveToDeadLetterDB(MessageRecord record) {
        // 记录到数据库或日志，后续人工处理
        System.err.println("[FAILED] " + record.body);
    }
}
```

## 7.3 三种可靠性级别对比

| 级别 | 措施 | 可靠性 | 吞吐量 |
|------|------|:---:|:---:|
| **尽力而为** | 无特殊措施 | 低 | 最高 |
| **中等可靠** | autoAck=false + 消息持久化 | 中 | 高 |
| **极高可靠** | Confirm + ACK + 持久化 + Quorum Queue + 本地消息表 | 极高 | 中 |

## 7.4 本地消息表（最终一致性方案）

```
┌──────────────────────────────────────────────────────┐
│                 业务数据库                             │
│  ┌──────────┐  本地事务写入   ┌──────────┐            │
│  │ 业务表   │ ←────────────→ │ 消息表    │            │
│  └──────────┘                └──────────┘            │
│                                    ↓                  │
│                           定时任务扫描未发送消息        │
│                                    ↓                  │
│                              RabbitMQ                 │
└──────────────────────────────────────────────────────┘
```

```sql
-- 消息表结构
CREATE TABLE event_message (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id VARCHAR(64) NOT NULL UNIQUE,
    exchange VARCHAR(128) NOT NULL,
    routing_key VARCHAR(128) NOT NULL,
    payload TEXT NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'PENDING',  -- PENDING / SENT / FAILED
    retry_count INT DEFAULT 0,
    max_retry INT DEFAULT 5,
    next_retry_time DATETIME,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

# 8. 消息可靠性总结

```
         第1层                  第2层                  第3层
    ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
    │ 确认机制     │       │ 持久化       │       │ 冗余         │
    │             │       │             │       │             │
    │ Confirm     │       │ Delivery    │       │ Mirror/     │
    │ ACK/NACK    │  ───→ │ mode=2     │  ───→ │ Quorum      │
    │ Mandatory   │       │ Durable     │       │ 队列        │
    │             │       │ Queue/Exch  │       │             │
    └─────────────┘       └─────────────┘       └─────────────┘
         ↓                      ↓                     ↓
    确认消息到达           确认写入磁盘           确认多节点复制
```

> **面试要点**：RabbitMQ 不丢消息需要三重保障——生产者 Confirm + Broker 持久化 + 消费者手动 ACK。在集群环境下，使用 Quorum Queue 可进一步保证数据在多数节点确认后才返回成功。
