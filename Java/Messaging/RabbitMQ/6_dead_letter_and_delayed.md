
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，涵盖死信队列（DLX）、消息/队列 TTL、延迟消息插件、重试策略设计。

---

# 1. 死信队列（Dead Letter Exchange / DLX）

## 1.1 什么是死信

消息在以下三种情况下会变为**死信（Dead Letter）**：

| 条件 | 说明 |
|------|------|
| **消息被拒绝** | `basicReject` / `basicNack` 且 `requeue=false` |
| **消息 TTL 过期** | 消息在队列中存活时间超过 TTL |
| **队列满** | 队列达到最大长度/容量，溢出策略为 `reject-publish` |

## 1.2 死信处理流程

```
┌────────────────────────────────────────────┐
│                  Broker                     │
│                                            │
│  ┌──────────┐          ┌──────────────┐    │
│  │ 业务队列  │─────────→│  死信交换机   │    │
│  │ (order)  │ 死信消息  │ (dlx.exch)   │    │
│  └──────────┘          └──────┬───────┘    │
│                               │            │
│                        ┌──────▼───────┐    │
│                        │  死信队列     │    │
│                        │ (dlx.order)  │    │
│                        └──────────────┘    │
└────────────────────────────────────────────┘
```

## 1.3 代码实现

```java
public class DeadLetterSetup {
    private static final String BUSINESS_EXCHANGE = "order.business";
    private static final String BUSINESS_QUEUE = "order.business.queue";
    private static final String DLX_EXCHANGE = "order.dlx";
    private static final String DLX_QUEUE = "order.dlx.queue";

    public static void setup(Channel channel) throws IOException {
        // 1. 声明死信交换机
        channel.exchangeDeclare(DLX_EXCHANGE, BuiltinExchangeType.DIRECT, true);

        // 2. 声明死信队列
        channel.queueDeclare(DLX_QUEUE, true, false, false, null);
        channel.queueBind(DLX_QUEUE, DLX_EXCHANGE, "order.dlx");

        // 3. 声明业务交换机
        channel.exchangeDeclare(BUSINESS_EXCHANGE, BuiltinExchangeType.DIRECT, true);

        // 4. 声明业务队列 — 通过参数绑定死信
        Map<String, Object> args = new HashMap<>();
        args.put("x-dead-letter-exchange", DLX_EXCHANGE);        // 死信交换机
        args.put("x-dead-letter-routing-key", "order.dlx");     // 死信路由键
        args.put("x-message-ttl", 30000);                       // 消息 TTL 30秒
        args.put("x-max-length", 10000);                        // 最大消息数

        channel.queueDeclare(BUSINESS_QUEUE, true, false, false, args);
        channel.queueBind(BUSINESS_QUEUE, BUSINESS_EXCHANGE, "order.created");
    }
}
```

## 1.4 死信消费者（监控/告警）

```java
public class DeadLetterConsumer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.basicConsume("order.dlx.queue", false,
            (consumerTag, delivery) -> {
                String body = new String(delivery.getBody(), StandardCharsets.UTF_8);
                String originalExchange = delivery.getEnvelope().getExchange();
                String originalRoutingKey = delivery.getEnvelope().getRoutingKey();

                // 从消息头获取原始信息和死亡原因
                Map<String, Object> headers = delivery.getProperties().getHeaders();
                String deathReason = "";
                if (headers != null && headers.containsKey("x-death")) {
                    List<Map<String, Object>> deaths =
                            (List<Map<String, Object>>) headers.get("x-death");
                    if (deaths != null && !deaths.isEmpty()) {
                        deathReason = String.valueOf(deaths.get(0).get("reason"));
                    }
                }

                System.err.printf("[死信告警] body=%s, reason=%s, origExchange=%s, origKey=%s%n",
                        body, deathReason, originalExchange, originalRoutingKey);

                // 记录数据库 / 发报警
                channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            },
            consumerTag -> {}
        );
    }
}
```

## 1.5 `x-death` 头信息

| 字段 | 说明 |
|------|------|
| `reason` | 死亡原因：`rejected` / `expired` / `maxlen` |
| `queue` | 原始队列名称 |
| `exchange` | 原始交换机 |
| `routing-keys` | 原始路由键 |
| `count` | 第几次死亡（可防止循环死信） |
| `time` | 死亡时间戳 |

---

# 2. TTL（Time To Live）

## 2.1 消息 TTL vs 队列 TTL

| 方式 | 设置位置 | 适用范围 | 过期后 |
|------|----------|----------|--------|
| **消息 TTL** | 单条消息属性 | 该条消息 | 变为死信（需配置 DLX） |
| **队列 TTL** | 队列参数 | 队列中所有消息 | 变为死信（需配置 DLX） |
| **队列过期** | 队列参数 `x-expires` | 整个队列 | 空闲超时后自动删除 |

## 2.2 消息级别 TTL

```java
// 方式1：发送时设置 TTL（针对单条消息）
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
        .expiration("60000")  // 60秒后过期，单位毫秒
        .build();
channel.basicPublish(EXCHANGE, ROUTING_KEY, props, msg.getBytes());
```

## 2.3 队列级别 TTL

```java
// 方式2：队列声明时设置 TTL（队列中所有消息统一 TTL）
Map<String, Object> args = new HashMap<>();
args.put("x-message-ttl", 60000);  // 该队列所有消息 60 秒后过期

channel.queueDeclare("ttl.queue", true, false, false, args);
```

> **TTL 与队列优先级的规则**：如果同时设置了消息 TTL 和队列 TTL，以**较小的值**生效。

## 2.4 队列过期

```java
// 队列在 1 小时内无消费者且无消息时自动删除
args.put("x-expires", 3600000);
channel.queueDeclare("auto.delete.queue", true, false, false, args);
```

---

# 3. 延迟消息（Delayed Message Plugin）

RabbitMQ 原生不支持延迟消息，需安装社区插件 `rabbitmq-delayed-message-exchange`。

## 3.1 插件安装

```bash
# Docker 方式安装（含插件）
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.13-management

# 下载插件 .ez 文件并放入 plugins 目录
# 或使用预装镜像
docker pull heidiks/rabbitmq-delayed-message-exchange:3.13.2-management

# 启用插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

## 3.2 工作原理

```
┌───────────────────────────────────────────┐
│          delayed.exchange                 │
│          (x-delayed-type: direct)         │
│                                           │
│  消息 + x-delay 头 → 延迟存储 → 到期后路由 │
└───────────────────────────────────────────┘
```

## 3.3 Java 代码

```java
public class DelayedMessageExample {
    private static final String DELAYED_EXCHANGE = "order.delayed";
    private static final String DELAYED_QUEUE = "order.delayed.queue";
    private static final String DELAYED_ROUTING_KEY = "order.delay";

    public static void main(String[] args) throws Exception {
        try (Connection connection = RabbitMQUtil.getConnection();
             Channel channel = connection.createChannel()) {

            // 声明延迟交换机（类型为 x-delayed-message）
            Map<String, Object> exchangeArgs = new HashMap<>();
            exchangeArgs.put("x-delayed-type", "direct");  // 底层实际路由类型
            channel.exchangeDeclare(DELAYED_EXCHANGE, "x-delayed-message",
                    true, false, exchangeArgs);

            // 声明队列并绑定
            channel.queueDeclare(DELAYED_QUEUE, true, false, false, null);
            channel.queueBind(DELAYED_QUEUE, DELAYED_EXCHANGE, DELAYED_ROUTING_KEY);

            // 发送延迟消息
            String msg1 = "30分钟后取消订单";
            sendDelayed(channel, DELAYED_EXCHANGE, DELAYED_ROUTING_KEY,
                    msg1, 30 * 60 * 1000); // 30 分钟

            String msg2 = "5秒后发送验证码";
            sendDelayed(channel, DELAYED_EXCHANGE, DELAYED_ROUTING_KEY,
                    msg2, 5000); // 5 秒
        }
    }

    private static void sendDelayed(Channel channel, String exchange,
             String routingKey, String message, long delayMs) throws IOException {
        Map<String, Object> headers = new HashMap<>();
        headers.put("x-delay", delayMs);

        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
                .headers(headers)
                .build();

        channel.basicPublish(exchange, routingKey, props,
                message.getBytes(StandardCharsets.UTF_8));

        System.out.printf("[Delayed] %s → delay=%dms%n", message, delayMs);
    }
}
```

## 3.4 TTL+DLX 实现延迟 vs 延迟插件对比

| 维度 | TTL + DLX | 延迟插件 |
|------|-----------|----------|
| **灵活性** | 需为不同延迟时间创建不同队列 | 每条消息可单独设置延迟时间 |
| **性能** | 队列内消息按时间排序需消耗 | 内部使用 Mnesia 表，效率更高 |
| **精确度** | 毫秒级 | 毫秒级 |
| **复杂度** | 高（多个队列+TLL+DLX 配置） | 低 |
| **适用性** | 少量固定延迟时间 | 多种不同延迟时间的任务 |

---

# 4. 重试策略与指数退避

## 4.1 重试架构

```
┌──────────┐  Nack+requeue  ┌──────────────┐
│ 消费者    │──────────────→│  Working Queue│
│ (处理)    │               └──────┬───────┘
└──────────┘                      │ 重试N次后
                                  ↓
                         ┌──────────────┐
                         │  重试队列     │
                         │ (TTL=30s)    │
                         └──────┬───────┘
                                │ TTL 过期
                                ↓
                         ┌──────────────┐
                         │  DLE → 死信   │
                         │  或回到Work   │
                         └──────────────┘
```

## 4.2 指数退避实现

```java
public class RetryConsumer {
    private static final String WORK_QUEUE = "work.queue";
    private static final String RETRY_QUEUE = "retry.queue";
    private static final int MAX_RETRY = 5;

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        setupRetryInfrastructure(connection);

        Channel channel = connection.createChannel();
        channel.basicQos(1);

        channel.basicConsume(WORK_QUEUE, false,
            (consumerTag, delivery) -> {
                String body = new String(delivery.getBody(), StandardCharsets.UTF_8);
                int retryCount = getRetryCount(delivery.getProperties());
                long deliveryTag = delivery.getEnvelope().getDeliveryTag();

                try {
                    processMessage(body);
                    channel.basicAck(deliveryTag, false);
                    System.out.println("[处理成功] " + body);
                } catch (Exception e) {
                    if (retryCount >= MAX_RETRY) {
                        // 超过最大重试次数 → Nack + 不requeue → 进入 DLX
                        channel.basicNack(deliveryTag, false, false);
                        System.err.println("[放弃重试] " + body + " 已进入死信");
                    } else {
                        // 未超过重试上限 → 发送到重试队列
                        long delay = (long) Math.pow(2, retryCount) * 1000; // 2^N 秒
                        sendToRetryQueue(channel, delivery.getProperties(),
                                delivery.getBody(), "work.exchange", "work.key", retryCount + 1, delay);

                        // 确认原消息（因为已发送到重试队列）
                        channel.basicAck(deliveryTag, false);
                        System.out.printf("[重试] %s → 第%d次重试，延迟%dms%n",
                                body, retryCount + 1, delay);
                    }
                }
            },
            consumerTag -> {}
        );
    }

    private static void sendToRetryQueue(Channel channel, AMQP.BasicProperties origProps,
            byte[] body, String origExchange, String origKey, int retryCount, long delay) throws IOException {
        Map<String, Object> headers = new HashMap<>();
        headers.put("x-original-exchange", origExchange);
        headers.put("x-original-routing-key", origKey);
        headers.put("x-retry-count", retryCount);

        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
                .headers(headers)
                .expiration(String.valueOf(delay)) // 延迟时间
                .build();

        channel.basicPublish("", "retry.queue", props, body);
    }

    private static int getRetryCount(AMQP.BasicProperties props) {
        if (props == null || props.getHeaders() == null) return 0;
        Object count = props.getHeaders().get("x-retry-count");
        return count != null ? (int) count : 0;
    }

    private static void processMessage(String body) throws Exception {
        // 模拟业务处理
        if (Math.random() > 0.7) throw new Exception("模拟处理失败");
    }

    private static void setupRetryInfrastructure(Connection connection) throws Exception {
        // 省略详细代码：创建 work.exchange, retry.queue(TTL=?), retry.dlx 等
    }
}
```

## 4.3 Spring 中的重试配置

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryOperationsInterceptor retryInterceptor(RabbitTemplate rabbitTemplate) {
        return RetryInterceptorBuilder.stateless()
                .maxAttempts(5)
                .backOffOptions(1000, 2.0, 60000) // 初始1s，乘数2.0，最大60s
                .build();
    }

    @Bean
    public SimpleRabbitListenerContainerFactory retryContainerFactory(
            ConnectionFactory connectionFactory, RetryOperationsInterceptor retryInterceptor) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        factory.setAdviceChain(retryInterceptor);
        return factory;
    }
}
```

---

# 5. 实战场景

## 5.1 场景一：订单超时取消

```
下单 → 发送 "order.created" 消息（x-delay=30min）
     → 30分钟后消费者收到消息
     → 查询订单状态
        ├─ 已支付 → 丢弃（ACK）
        └─ 未支付 → 关闭订单
```

```java
// 生产者：下单后发送延迟消息
public void createOrder(Order order) {
    // 1. 写入数据库
    orderRepository.save(order);

    // 2. 发送延迟取消消息
    Map<String, Object> headers = new HashMap<>();
    headers.put("x-delay", 30 * 60 * 1000); // 30 分钟
    rabbitTemplate.convertAndSend("order.delayed", "order.cancel",
            new CancelOrderEvent(order.getId()),
            msg -> {
                msg.getMessageProperties().getHeaders().putAll(headers);
                return msg;
            });
}

// 消费者：检查订单并取消
@RabbitListener(queues = "order.cancel.queue")
public void handleCancel(@Payload CancelOrderEvent event) {
    Order order = orderRepository.findById(event.getOrderId());
    if (order != null && order.getStatus() == OrderStatus.UNPAID) {
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
        // 释放库存
    }
}
```

## 5.2 场景二：短信重试

```
发送短信 → 失败
  → 第1次重试（2秒后）
  → 第2次重试（4秒后）
  → 第3次重试（8秒后）
  → 超过重试上限 → 记录死信 → 告警通知运维
```
