
# 1. 概述

## 1.1 什么是 JMS

JMS（Java Message Service，Java 消息服务）是 JavaEE 的**异步消息中间件标准规范**（JSR-914），提供统一的发送/接收消息 API，屏蔽具体消息中间件差异。

```
┌──────────┐   send   ┌──────────────┐   receive  ┌──────────┐
│ 生产者    │ ───────► │  消息提供者    │ ─────────► │ 消费者    │
│ Producer │          │  Broker       │            │ Consumer │
└──────────┘          └──────────────┘            └──────────┘
```

## 1.2 JMS 与消息中间件的关系

| 概念 | 说明 |
|------|------|
| JMS API | 规范接口（`jakarta.jms.*`） |
| JMS Provider | 具体实现：ActiveMQ、RabbitMQ（AMQP 但提供 JMS 支持）、IBM MQ、OpenMQ 等 |
| 与 Kafka 关系 | Kafka 不是 JMS 实现（无 JMS API），使用自有协议 |

> 具体 MQ 对比见 `Messaging/` 目录（Kafka、RabbitMQ、RocketMQ）。

## 1.3 为什么用消息

- **异步解耦**：生产/消费无需同时在线。
- **削峰填谷**：瞬时高流量由队列缓冲。
- **可靠性**：消息持久化，失败可重试。
- 典型场景：订单确认通知、日志收集、任务分发、事件驱动。

# 2. JMS 核心概念

## 2.1 对象模型

```
ConnectionFactory
      │ createConnection()
      ▼
Connection
      │ createSession()
      ▼
Session
      ├── createProducer() ──► MessageProducer
      ├── createConsumer() ──► MessageConsumer
      ├── createQueue() / createTopic() ──► Destination
      └── createMessage()... ──► Message
```

| 对象 | 作用 |
|------|------|
| `ConnectionFactory` | 创建连接（由容器管理/通过 JNDI 获取） |
| `Connection` | JMS 连接（物理连接） |
| `Session` | 会话（事务边界、消息创建） |
| `MessageProducer` | 消息生产者 |
| `MessageConsumer` | 消息消费者 |
| `Destination` | 目的地（Queue 或 Topic） |
| `Message` | 消息对象 |

## 2.2 两种消息模型

| 模型 | 目的地 | 特点 |
|------|--------|------|
| **点对点（P2P）** | Queue | 一条消息只有一个消费者；消费者可离线，消息保留直到被消费 |
| **发布/订阅（Pub/Sub）** | Topic | 一条消息广播给所有订阅者；订阅者需要在线或有持久订阅 |

```
P2P：  P ──► [Queue] ──► C1（独占）
                          C2 不会收到

Pub/Sub： P ──► [Topic] ──► C1, C2, C3（都收到）
```

# 3. 消息类型

| 接口 | 内容 |
|------|------|
| `TextMessage` | 文本字符串 |
| `ObjectMessage` | 可序列化 Java 对象 |
| `MapMessage` | 键值对集合 |
| `BytesMessage` | 字节流 |
| `StreamMessage` | Java 原始类型流 |
| `Message` | 无负载，仅属性 |

```java
TextMessage msg = session.createTextMessage("hello");
msg.setStringProperty("type", "ORDER");
msg.setIntProperty("userId", 1001);
```

# 4. 发送消息（生产者）

## 4.1 容器环境中使用

```java
@Stateless
public class OrderService {

    @Resource(mappedName = "java:comp/env/jms/ConnectionFactory")
    private ConnectionFactory factory;

    @Resource(mappedName = "java:comp/env/jms/OrderQueue")
    private Queue orderQueue;

    public void placeOrder(Order order) {
        try (Connection conn = factory.createConnection();
             JMSContext context = conn.createContext()) {

            JMSProducer producer = context.createProducer();
            TextMessage msg = context.createTextMessage(
                    "{\"orderId\":" + order.getId() + "}");
            producer.send(orderQueue, msg);
        } catch (JMSException e) {
            throw new RuntimeException(e);
        }
    }
}
```

- JMS 2.0 引入简化的 `JMSContext`，替代 `Session`/`Connection` 组合，推荐使用。

## 4.2 发送属性设置

```java
producer.setDeliveryMode(DeliveryMode.PERSISTENT);   // 持久化
producer.setPriority(5);
producer.setTimeToLive(60000);                       // 存活 60 秒
producer.setJMSReplyTo(replyQueue);                  // 指定回复队列
producer.setJMSCorrelationID(msgId);                 // 关联 ID
```

# 5. 接收消息（消费者）

## 5.1 同步接收

```java
try (Connection conn = factory.createConnection();
     JMSContext context = conn.createContext()) {
    conn.start();
    JMSConsumer consumer = context.createConsumer(orderQueue);
    TextMessage msg = (TextMessage) consumer.receive(5000);  // 阻塞最多5秒
    if (msg != null) {
        System.out.println(msg.getText());
    }
}
```

## 5.2 异步监听（推荐）

```java
@MessageDriven(
    activationConfig = {
        @ActivationConfigProperty(propertyName = "destinationType",
                propertyValue = "jakarta.jms.Queue"),
        @ActivationConfigProperty(propertyName = "destinationLookup",
                propertyValue = "java:app/jms/orderQueue")
    }
)
public class OrderConsumer implements MessageListener {
    @Override
    public void onMessage(Message message) {
        try {
            if (message instanceof TextMessage tm) {
                System.out.println("收到：" + tm.getText());
            }
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
```

- MDB 是 JavaEE 标准的异步消息消费者，容器管理并发与事务。

# 6. 会话与事务

## 6.1 会话的三种模式

| 模式 | 说明 |
|------|------|
| `AUTO_ACKNOWLEDGE` | 接收后自动确认 |
| `CLIENT_ACKNOWLEDGE` | 客户端手动 `acknowledge()` |
| `DUPS_OK_ACKNOWLEDGE` | 允许重复确认，性能好 |

## 6.2 事务性会话

```java
try (JMSContext context = conn.createContext(true)) {   // true = 事务会话
    context.createProducer().send(queue, msg1);
    context.createProducer().send(queue, msg2);
    context.commit();        // 全部提交
    // context.rollback();   // 全部回滚
}
```

## 6.3 JTA + JMS 全局事务

- MDB 的 `onMessage` 默认参与容器事务：**消息接收 + 业务操作 + 数据库更新在同一事务**。
- 业务处理失败抛异常 → 消息被回滚，可重新投递（`Redelivered` 标记）。

# 7. 消息可靠性机制

| 机制 | 说明 |
|------|------|
| 持久化（PERSISTENT） | 消息写入持久化存储，Broker 重启不丢 |
| 确认机制（Ack） | 消费确认后才算处理成功 |
| 重投递 | 消费者异常时消息重新投递（避免无限重投设置上限） |
| 死信队列（DLQ） | 重投递次数超限的消息进入死信队列 |
| 消息过期（TTL） | 过期消息不被消费 |

# 8. 与具体 MQ 的对照

| 特性 | JMS（ActiveMQ） | RabbitMQ | Kafka | RocketMQ |
|------|-----------------|----------|-------|----------|
| 协议 | JMS | AMQP | 自有协议 | 自有协议 |
| 模型 | Queue/Topic | Exchange/Queue | Topic/Partition | Topic/Queue |
| 消息顺序 | 支持 | 单队列有序 | 分区内有序 | 支持 |
| 持久化 | 支持 | 支持 | 支持（日志） | 支持 |
| 吞吐 | 中 | 中 | 极高 | 高 |
| 典型场景 | 企业集成 | 微服务解耦 | 日志/流处理 | 电商/金融 |

# 9. 常见问题排查清单

| 问题 | 排查方向 |
|------|----------|
| 消息丢失 | 确认 `DeliveryMode.PERSISTENT`、消费者 ack 设置、Broker 持久化 |
| 消息重复消费 | 消费者幂等设计（幂等键/去重表） |
| 消息阻塞 | 检查消费者并发数、连接数、消息积压 |
| `JMSException` 连接失败 | 检查 ConnectionFactory、Broker 地址、认证 |
| MDB 未消费 | 检查 `destinationLookup`、MDB 是否部署、队列是否存在 |
| 事务不一致 | 确认 JTA 是否包含 JMS 与数据库资源 |
