
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，Java 原生客户端 **amqp-client 5.21.0**，完整演示 RabbitMQ 六种核心消息模式的代码实现。

---

# 1. 模式概览

| 模式 | Exchange 类型 | 核心特点 | 典型场景 |
|------|:---:|------|----------|
| **Hello World** | Default | 最简单的点对点 | 入门示例 |
| **Work Queues** | Default | 多消费者竞争消费，任务分发 | 耗时任务处理（图片处理、批量发送） |
| **Publish/Subscribe** | Fanout | 一条消息广播到所有绑定队列 | 配置刷新通知、日志广播 |
| **Routing** | Direct | 按 RoutingKey 精确路由 | 错误级别日志分发 |
| **Topics** | Topic | 通配符匹配，灵活路由 | 消息多条件分发 |
| **RPC** | Default + Callback | 双向通信，请求-响应模式 | 跨语言服务调用 |

---

# 2. 准备工作 — 工具类

```java
package com.example.rabbitmq.util;

import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitMQUtil {

    private static final String HOST = "localhost";
    private static final int PORT = 5672;
    private static final String USERNAME = "admin";
    private static final String PASSWORD = "admin123";
    private static final String VHOST = "/";

    private static volatile ConnectionFactory factory;

    public static ConnectionFactory getFactory() {
        if (factory == null) {
            synchronized (RabbitMQUtil.class) {
                if (factory == null) {
                    factory = new ConnectionFactory();
                    factory.setHost(HOST);
                    factory.setPort(PORT);
                    factory.setUsername(USERNAME);
                    factory.setPassword(PASSWORD);
                    factory.setVirtualHost(VHOST);
                    factory.setAutomaticRecoveryEnabled(true);
                    factory.setNetworkRecoveryInterval(3000);
                }
            }
        }
        return factory;
    }

    public static Connection getConnection() throws IOException, TimeoutException {
        return getFactory().newConnection();
    }
}
```

---

# 3. Hello World — 最简单的点对点

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Producer │────→│    Queue     │────→│ Consumer │
└──────────┘     └──────────────┘     └──────────┘
```

## 3.1 生产者

```java
import com.example.rabbitmq.util.RabbitMQUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

public class HelloWorldProducer {

    private static final String QUEUE_NAME = "hello.queue";

    public static void main(String[] args) throws Exception {
        try (Connection connection = RabbitMQUtil.getConnection();
             Channel channel = connection.createChannel()) {

            // 声明队列
            channel.queueDeclare(QUEUE_NAME, true, false, false, null);

            // 发送消息
            String message = "Hello RabbitMQ! " + System.currentTimeMillis();
            channel.basicPublish("", QUEUE_NAME, null,
                    message.getBytes(java.nio.charset.StandardCharsets.UTF_8));

            System.out.println("[x] Sent: " + message);
        }
    }
}
```

## 3.2 消费者

```java
import com.example.rabbitmq.util.RabbitMQUtil;
import com.rabbitmq.client.*;

public class HelloWorldConsumer {

    private static final String QUEUE_NAME = "hello.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), java.nio.charset.StandardCharsets.UTF_8);
            System.out.println("[x] Received: " + message);
        };

        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> {});
    }
}
```

> `basicPublish("", QUEUE_NAME, ...)` 中的空字符串 `""` 表示使用 **Default Exchange**，每个队列自动以队列名为 RoutingKey 绑定到 Default Exchange。

---

# 4. Work Queues — 任务分发

```
                        ┌─────────────┐
┌──────────┐     ┌────→│  Consumer 1  │  (Worker 1)
│ Producer │────→│ Q   ├─────────────┤
└──────────┘     └────→│  Consumer 2  │  (Worker 2)
                        └─────────────┘
```

## 4.1 轮询分发（Round-Robin Dispatch）

默认情况下，RabbitMQ 将消息**轮询分发给所有消费者**（你一个我一个），不考虑消费者当前负载。

```java
// 消费者 1
public class Worker1 {
    private static final String QUEUE_NAME = "task.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println("[Worker1] Received: " + message);
            try {
                // 模拟处理耗时
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("[Worker1] Done: " + message);
        };

        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> {});
    }
}
```

## 4.2 公平分发（Fair Dispatch）

设置 `basicQos(1)`（Consumer Prefetch = 1），告诉 RabbitMQ 在收到前一条消息的 ACK 前不要给该消费者发送新消息，实现**能者多劳**。

```java
public class FairWorker {
    private static final String QUEUE_NAME = "task.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);

        // 关键：设置 prefetchCount = 1，同时关闭 autoAck
        channel.basicQos(1);

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            System.out.println("[Worker] Processing: " + message);
            try {
                Thread.sleep(2000); // 模拟耗时处理
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                // 手动确认
                channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
                System.out.println("[Worker] Done: " + message);
            }
        };

        // autoAck = false
        channel.basicConsume(QUEUE_NAME, false, deliverCallback, consumerTag -> {});
    }
}
```

## 4.3 消息确认（ACK）与持久化

```java
// 生产者 — 发送持久化消息
import com.rabbitmq.client.MessageProperties;

String message = "Persistent Task Message";
channel.basicPublish("", QUEUE_NAME,
        MessageProperties.PERSISTENT_TEXT_PLAIN,  // delivery-mode=2
        message.getBytes(StandardCharsets.UTF_8));

// 消费者 — 手动确认
channel.basicConsume(QUEUE_NAME, false,  // autoAck=false
    (consumerTag, delivery) -> {
        try {
            processTask(delivery.getBody());
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        } catch (Exception e) {
            // 处理失败：Nack + requeue=false 进入死信队列
            channel.basicNack(delivery.getEnvelope().getDeliveryTag(), false, false);
        }
    },
    consumerTag -> {}
);
```

---

# 5. Publish/Subscribe — 发布订阅（Fanout）

```
                          Fanout Exchange
                        ┌────────────────┐
┌──────────┐            │                │──→ [sms.queue]
│ Producer │───fanout──→│                │──→ [email.queue]
└──────────┘            │                │──→ [push.queue]
                        └────────────────┘
```

## 5.1 生产者

```java
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

public class FanoutProducer {
    private static final String EXCHANGE_NAME = "notification.fanout";

    public static void main(String[] args) throws Exception {
        try (Connection connection = RabbitMQUtil.getConnection();
             Channel channel = connection.createChannel()) {

            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);

            String message = "系统升级通知: 今晚 22:00～02:00 维护";
            channel.basicPublish(EXCHANGE_NAME, "", null,
                    message.getBytes(StandardCharsets.UTF_8));

            System.out.println("[x] Sent: " + message);
        }
    }
}
```

## 5.2 消费者（短信服务）

```java
public class SmsConsumer {
    private static final String EXCHANGE_NAME = "notification.fanout";
    private static final String QUEUE_NAME = "sms.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        channel.basicConsume(QUEUE_NAME, true,
            (consumerTag, delivery) -> {
                String msg = new String(delivery.getBody(), StandardCharsets.UTF_8);
                System.out.println("[SMS] 发送短信: " + msg);
            },
            consumerTag -> {}
        );
    }
}
```

## 5.3 消费者（邮件服务）

```java
public class EmailConsumer {
    private static final String EXCHANGE_NAME = "notification.fanout";
    private static final String QUEUE_NAME = "email.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");

        channel.basicConsume(QUEUE_NAME, true,
            (consumerTag, delivery) -> {
                String msg = new String(delivery.getBody(), StandardCharsets.UTF_8);
                System.out.println("[Email] 发送邮件: " + msg);
            },
            consumerTag -> {}
        );
    }
}
```

---

# 6. Routing — 路由模式（Direct）

```
                          Direct Exchange
                        ┌────────────────┐
┌──────────┐            │                │─ "order.create" → [order.create.queue]
│ Producer │──routing──→│                │─ "order.cancel" → [order.cancel.queue]
└──────────┘            │                │─ "order.refund" → [order.refund.queue]
                        └────────────────┘
```

## 6.1 生产者

```java
public class DirectProducer {
    private static final String EXCHANGE_NAME = "order.direct";

    public static void main(String[] args) throws Exception {
        try (Connection connection = RabbitMQUtil.getConnection();
             Channel channel = connection.createChannel()) {

            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT, true);

            // 发送创建订单消息
            String createMsg = "{\"orderId\": 1001, \"action\": \"create\"}";
            channel.basicPublish(EXCHANGE_NAME, "order.create", null,
                    createMsg.getBytes(StandardCharsets.UTF_8));
            System.out.println("[x] Sent 'order.create': " + createMsg);

            // 发送取消订单消息
            String cancelMsg = "{\"orderId\": 1002, \"action\": \"cancel\"}";
            channel.basicPublish(EXCHANGE_NAME, "order.cancel", null,
                    cancelMsg.getBytes(StandardCharsets.UTF_8));
            System.out.println("[x] Sent 'order.cancel': " + cancelMsg);

            // 发送退款消息
            String refundMsg = "{\"orderId\": 1003, \"action\": \"refund\"}";
            channel.basicPublish(EXCHANGE_NAME, "order.refund", null,
                    refundMsg.getBytes(StandardCharsets.UTF_8));
            System.out.println("[x] Sent 'order.refund': " + refundMsg);
        }
    }
}
```

## 6.2 消费者（三个队列分别消费不同的 routing key）

```java
public class OrderCreateConsumer {
    private static final String EXCHANGE_NAME = "order.direct";
    private static final String QUEUE_NAME = "order.create.queue";
    private static final String ROUTING_KEY = "order.create";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT, true);
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, ROUTING_KEY);

        channel.basicConsume(QUEUE_NAME, true,
            (consumerTag, delivery) -> {
                String msg = new String(delivery.getBody(), StandardCharsets.UTF_8);
                System.out.println("[创建订单服务] 处理: " + msg);
            },
            consumerTag -> {}
        );
    }
}
```

## 6.3 一个队列绑定多个 RoutingKey

```java
// 一个队列同时接收 order.create 和 order.cancel 消息
channel.queueBind("order.multi.queue", "order.direct", "order.create");
channel.queueBind("order.multi.queue", "order.direct", "order.cancel");
```

---

# 7. Topics — 主题模式（Topic）

```
                          Topic Exchange
                        ┌────────────────┐
   "order.create.cn"    │                │─ "order.#"        → [order.all.queue]
 ──────────────────────→│                │─ "order.create.*" → [order.create.queue]
   "order.cancel.us"    │                │─ "order.*.us"     → [order.us.queue]
 ──────────────────────→│                │
                        └────────────────┘
```

## 7.1 生产者

```java
public class TopicProducer {
    private static final String EXCHANGE_NAME = "order.topic";

    public static void main(String[] args) throws Exception {
        try (Connection connection = RabbitMQUtil.getConnection();
             Channel channel = connection.createChannel()) {

            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);

            // 中国区创建订单
            channel.basicPublish(EXCHANGE_NAME, "order.create.cn", null,
                    "{\"order\":\"create\",\"region\":\"cn\"}".getBytes(StandardCharsets.UTF_8));

            // 美国区取消订单
            channel.basicPublish(EXCHANGE_NAME, "order.cancel.us", null,
                    "{\"order\":\"cancel\",\"region\":\"us\"}".getBytes(StandardCharsets.UTF_8));

            // 中国区退款
            channel.basicPublish(EXCHANGE_NAME, "order.refund.cn", null,
                    "{\"order\":\"refund\",\"region\":\"cn\"}".getBytes(StandardCharsets.UTF_8));

            System.out.println("[x] All messages sent");
        }
    }
}
```

## 7.2 消费者

```java
public class AllOrderConsumer {
    private static final String EXCHANGE_NAME = "order.topic";
    private static final String QUEUE_NAME = "order.all.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        // 匹配所有 order 开头的消息
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "order.#");

        channel.basicConsume(QUEUE_NAME, true,
            (consumerTag, delivery) -> {
                String msg = new String(delivery.getBody(), StandardCharsets.UTF_8);
                System.out.println("[全部订单] " +
                    delivery.getEnvelope().getRoutingKey() + " → " + msg);
            },
            consumerTag -> {}
        );
    }
}

public class CnOrderConsumer {
    private static final String EXCHANGE_NAME = "order.topic";
    private static final String QUEUE_NAME = "order.cn.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        // 匹配所有中国区的消息
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "order.*.cn");

        channel.basicConsume(QUEUE_NAME, true,
            (consumerTag, delivery) -> {
                String msg = new String(delivery.getBody(), StandardCharsets.UTF_8);
                System.out.println("[中国区订单] " +
                    delivery.getEnvelope().getRoutingKey() + " → " + msg);
            },
            consumerTag -> {}
        );
    }
}
```

## 7.3 Topic 匹配示例

| RoutingKey | `order.*.cn` | `order.#` | `order.create.*` | `#.us` |
|------------|:---:|:---:|:---:|:---:|
| `order.create.cn` | ✓ | ✓ | ✓ | ✗ |
| `order.cancel.us` | ✗ | ✓ | ✗ | ✓ |
| `order.refund.cn.east` | ✗ | ✓ | ✗ | ✗ |
| `payment.pay.us` | ✗ | ✗ | ✗ | ✓ |

---

# 8. RPC — 远程过程调用

```
┌──────────┐  request ┌──────────────┐  request ┌──────────┐
│  Client  │─────────→│ request.queue│─────────→│  Server  │
│          │          └──────────────┘          │          │
│          │  reply  ┌──────────────┐  reply   │          │
│  (等待)  │←────────│ reply.queue  │←─────────│  (处理)  │
└──────────┘          └──────────────┘          └──────────┘
```

**核心机制**：通过 `correlationId` 关联请求与响应，`replyTo` 指定回调队列。

## 8.1 RPC 服务端

```java
import com.rabbitmq.client.*;

public class RPCServer {
    private static final String RPC_QUEUE_NAME = "rpc.queue";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(RPC_QUEUE_NAME, true, false, false, null);
        channel.basicQos(1);
        System.out.println("[RPC Server] 等待请求...");

        channel.basicConsume(RPC_QUEUE_NAME, false,
            (consumerTag, delivery) -> {
                String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
                System.out.println("[RPC Server] 收到请求: " + message);

                // 计算响应
                String response = "Fibonacci(" + message + ") = " + fib(Integer.parseInt(message));

                // 发送响应到 replyTo 队列
                AMQP.BasicProperties replyProps = new AMQP.BasicProperties.Builder()
                        .correlationId(delivery.getProperties().getCorrelationId())
                        .build();
                channel.basicPublish("",
                        delivery.getProperties().getReplyTo(),
                        replyProps,
                        response.getBytes(StandardCharsets.UTF_8));

                channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            },
            consumerTag -> {}
        );
    }

    private static int fib(int n) {
        if (n <= 1) return n;
        return fib(n - 1) + fib(n - 2);
    }
}
```

## 8.2 RPC 客户端

```java
import com.rabbitmq.client.*;
import java.io.IOException;
import java.util.UUID;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class RPCClient implements AutoCloseable {
    private static final String RPC_QUEUE_NAME = "rpc.queue";
    private Connection connection;
    private Channel channel;

    public RPCClient() throws Exception {
        connection = RabbitMQUtil.getConnection();
        channel = connection.createChannel();
    }

    public String call(String message) throws Exception {
        final String corrId = UUID.randomUUID().toString();

        // 创建临时回调队列
        String replyQueueName = channel.queueDeclare().getQueue();
        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
                .correlationId(corrId)
                .replyTo(replyQueueName)
                .build();

        channel.basicPublish("", RPC_QUEUE_NAME, props,
                message.getBytes(StandardCharsets.UTF_8));

        // 使用 BlockingQueue 同步等待响应
        final BlockingQueue<String> response = new ArrayBlockingQueue<>(1);
        String ctag = channel.basicConsume(replyQueueName, true,
            (consumerTag, delivery) -> {
                if (delivery.getProperties().getCorrelationId().equals(corrId)) {
                    response.offer(new String(delivery.getBody(), StandardCharsets.UTF_8));
                }
            },
            consumerTag -> {}
        );

        String result = response.take();
        channel.basicCancel(ctag);
        return result;
    }

    @Override
    public void close() throws Exception {
        if (connection != null) connection.close();
    }

    public static void main(String[] args) throws Exception {
        try (RPCClient client = new RPCClient()) {
            for (int i = 0; i < 10; i++) {
                String resp = client.call(String.valueOf(i));
                System.out.println("[RPC Client] " + resp);
            }
        }
    }
}
```

---

# 9. 模式选择决策

```
需要多个消费者同时接收同一条消息？
  ├─ 是 → Pub/Sub（Fanout Exchange）
  └─ 否 → 继续判断

需要按条件路由到不同的队列？
  ├─ 是 → 继续判断
  │   ├─ 精确匹配 → Routing（Direct Exchange）
  │   ├─ 通配符匹配 → Topics（Topic Exchange）
  │   └─ Header 匹配 → Headers Exchange
  └─ 否 → 继续判断

需要请求-响应双向通信？
  ├─ 是 → RPC
  └─ 否 → Hello World（点对点）
```
