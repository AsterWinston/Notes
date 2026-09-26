
> 版本说明：本文档基于 **Spring Boot 3.2.x / 3.3.x**、**Spring AMQP 3.1.x**，全面讲解 Spring Boot 与 RabbitMQ 的集成开发。

---

# 1. 依赖与配置

## 1.1 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<!-- JSON 序列化（可选） -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

## 1.2 application.yml 配置

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: admin123
    virtual-host: /
    # 连接超时
    connection-timeout: 30000
    # 心跳
    requested-heartbeat: 60
    # 发布确认
    publisher-confirm-type: correlated  # SIMPLE / CORRELATED / NONE
    publisher-returns: true
    # 消费者
    listener:
      simple:
        acknowledge-mode: manual       # NONE / AUTO / MANUAL
        prefetch: 1
        concurrency: 3                 # 并发消费者数
        max-concurrency: 10            # 最大并发数
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000ms
          multiplier: 2.0
          max-interval: 10000ms
    # 模板（生产者）
    template:
      retry:
        enabled: true
        max-attempts: 3
```

---

# 2. 核心配置类

```java
package com.example.rabbitmq.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    // ========== 交换机声明 ==========
    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder.directExchange("order.direct")
                .durable(true)
                .build();
    }

    @Bean
    public TopicExchange orderTopicExchange() {
        return ExchangeBuilder.topicExchange("order.topic")
                .durable(true)
                .build();
    }

    @Bean
    public FanoutExchange notificationExchange() {
        return ExchangeBuilder.fanoutExchange("notification.fanout")
                .durable(true)
                .build();
    }

    // ========== 队列声明 ==========
    @Bean
    public Queue orderCreateQueue() {
        return QueueBuilder.durable("order.create.queue")
                .withArgument("x-dead-letter-exchange", "order.dlx")
                .withArgument("x-dead-letter-routing-key", "order.dlx")
                .build();
    }

    @Bean
    public Queue orderSmsQueue() {
        return QueueBuilder.durable("order.sms.queue").build();
    }

    @Bean
    public Queue orderEmailQueue() {
        return QueueBuilder.durable("order.email.queue").build();
    }

    // ========== 死信声明 ==========
    @Bean
    public DirectExchange dlxExchange() {
        return new DirectExchange("order.dlx");
    }

    @Bean
    public Queue dlxQueue() {
        return QueueBuilder.durable("order.dlx.queue").build();
    }

    @Bean
    public Binding dlxBinding() {
        return BindingBuilder.bind(dlxQueue()).to(dlxExchange()).with("order.dlx");
    }

    // ========== 绑定 ==========
    @Bean
    public Binding orderCreateBinding() {
        return BindingBuilder.bind(orderCreateQueue())
                .to(orderExchange()).with("order.create");
    }

    @Bean
    public Binding smsBinding() {
        return BindingBuilder.bind(orderSmsQueue())
                .to(notificationExchange()).with("");
    }

    @Bean
    public Binding emailBinding() {
        return BindingBuilder.bind(orderEmailQueue())
                .to(notificationExchange()).with("");
    }

    // ========== JSON 消息转换器 ==========
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    // ========== RabbitTemplate 配置 ==========
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                          MessageConverter messageConverter) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter);

        // 确认回调（消息到达 Broker）
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                System.out.println("[Confirm] 消息已送达 Broker: " +
                        (correlationData != null ? correlationData.getId() : "null"));
            } else {
                System.err.println("[Confirm] 消息发送失败: " + cause);
            }
        });

        // 回退回调（消息未路由到任何队列）
        template.setReturnsCallback(returned -> {
            System.err.printf("[Return] 消息路由失败: exchange=%s, routingKey=%s, replyCode=%d, replyText=%s%n",
                    returned.getExchange(), returned.getRoutingKey(),
                    returned.getReplyCode(), returned.getReplyText());
        });

        template.setMandatory(true);
        return template;
    }
}
```

---

# 3. RabbitTemplate 发送消息

## 3.1 基本发送

```java
@Service
public class OrderMessageProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    // 发送 JSON 对象（自动序列化）
    public void sendOrderCreated(OrderDTO order) {
        rabbitTemplate.convertAndSend("order.direct", "order.create", order);
    }

    // 带 CorrelationData（用于 Confirm 追踪）
    public void sendWithConfirm(OrderDTO order) {
        CorrelationData correlationData = new CorrelationData(order.getOrderId());
        rabbitTemplate.convertAndSend("order.direct", "order.create",
                order, correlationData);
    }

    // 发送延迟消息（需延迟插件）
    public void sendDelayed(OrderDTO order, long delayMs) {
        rabbitTemplate.convertAndSend("order.delayed", "order.cancel", order,
            msg -> {
                msg.getMessageProperties().setHeader("x-delay", delayMs);
                return msg;
            });
    }

    // 发送带自定义属性的消息
    public void sendWithProperties(OrderDTO order) {
        rabbitTemplate.convertAndSend("order.direct", "order.create", order,
            msg -> {
                MessageProperties props = msg.getMessageProperties();
                props.setMessageId(order.getOrderId());
                props.setPriority(5);
                props.setExpiration("60000");
                return msg;
            });
    }

    // 同步 RPC（sendAndReceive）
    public OrderResult sendAndReceive(OrderDTO order) {
        Object response = rabbitTemplate.convertSendAndReceive(
                "order.direct", "order.rpc", order);
        return (OrderResult) response;
    }
}
```

---

# 4. @RabbitListener 消费消息

```java
@Component
public class OrderMessageConsumer {

    // ========== 自动确认消费 ==========
    @RabbitListener(queues = "order.create.queue")
    public void handleOrderCreate(OrderDTO order) {
        System.out.println("[自动确认] 收到订单: " + order.getOrderId());
        // 处理业务...
    }

    // ========== 手动确认消费 ==========
    @RabbitListener(queues = "order.create.queue", ackMode = "MANUAL")
    public void handleOrderManual(OrderDTO order, Channel channel,
                                   @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) {
        try {
            System.out.println("[手动确认] 处理订单: " + order.getOrderId());
            // 业务处理
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            try {
                // 拒绝并进入死信
                channel.basicNack(deliveryTag, false, false);
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }
    }

    // ========== 获取消息元数据 ==========
    @RabbitListener(queues = "order.create.queue")
    public void handleWithMetadata(OrderDTO order,
                                    @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey,
                                    @Header(AmqpHeaders.RECEIVED_EXCHANGE) String exchange,
                                    Message message) {
        MessageProperties props = message.getMessageProperties();
        System.out.printf("[元数据] exchange=%s, routingKey=%s, messageId=%s, body=%s%n",
                exchange, routingKey, props.getMessageId(), order);
    }

    // ========== 多队列消费 ==========
    @RabbitListener(queues = {"order.sms.queue", "order.email.queue"})
    public void handleNotification(String notification) {
        System.out.println("[通知] " + notification);
    }

    // ========== Topic 模式 ==========
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(value = "order.topic.queue", durable = "true"),
            exchange = @Exchange(value = "order.topic", type = ExchangeTypes.TOPIC),
            key = "order.#"
    ))
    public void handleTopic(OrderDTO order) {
        System.out.println("[Topic] 收到: " + order.getOrderId());
    }
}
```

## 4.1 @RabbitListener 注解参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `queues` | 监听的队列名称数组 | `{"queue1", "queue2"}` |
| `bindings` | 声明式创建交换机/队列/绑定 | `@QueueBinding(...)` |
| `id` | 消费者唯一标识 | `"orderConsumer"` |
| `concurrency` | 并发消费者数 | `"3-10"` |
| `ackMode` | 确认模式 | `"MANUAL"` |
| `autoStartup` | 是否自动启动 | `true` |
| `containerFactory` | 指定 ListenerContainerFactory | `"retryContainerFactory"` |
| `admin` | 声明资源失败时是否报错 | `RabbitAdmin` |

---

# 5. 重试与死信处理

## 5.1 消费者端重试（Spring Retry）

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryOperationsInterceptor retryInterceptor() {
        return RetryInterceptorBuilder.stateless()
                .maxAttempts(3)
                .backOffOptions(1000, 2.0, 10000) // 1s, 2s, 4s, ...
                .recoverer(new RejectAndDontRequeueRecoverer()) // 重试耗尽后拒绝
                .build();
    }

    @Bean
    public SimpleRabbitListenerContainerFactory retryContainerFactory(
            ConnectionFactory connectionFactory,
            RetryOperationsInterceptor retryInterceptor) {
        SimpleRabbitListenerContainerFactory factory =
                new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setAcknowledgeMode(AcknowledgeMode.AUTO);
        factory.setAdviceChain(retryInterceptor);
        return factory;
    }
}

// 使用
@RabbitListener(queues = "order.create.queue",
        containerFactory = "retryContainerFactory")
public void handle(OrderDTO order) {
    // 处理中抛异常会自动重试
}
```

## 5.2 配置死信队列完整方案

```
order.create.queue ──(死信)──→ order.dlx ──→ order.dlx.queue
```

```java
@Configuration
public class DeadLetterConfig {

    // 业务队列
    @Bean
    public Queue businessQueue() {
        return QueueBuilder.durable("order.business.queue")
                .withArgument("x-dead-letter-exchange", "order.dlx")
                .withArgument("x-dead-letter-routing-key", "order.dead")
                .withArgument("x-message-ttl", 30000)
                .build();
    }

    // 死信交换机
    @Bean
    public DirectExchange dlxExchange() {
        return new DirectExchange("order.dlx");
    }

    // 死信队列
    @Bean
    public Queue dlxQueue() {
        return new Queue("order.dlx.queue", true);
    }

    @Bean
    public Binding dlxBinding() {
        return BindingBuilder.bind(dlxQueue()).to(dlxExchange()).with("order.dead");
    }
}

// 死信消费者
@Component
public class DeadLetterHandler {

    @RabbitListener(queues = "order.dlx.queue")
    public void handleDeadMessage(Message message) {
        String body = new String(message.getBody());
        System.err.println("[死信处理] " + body);

        // 可记录到数据库、发送告警...
    }
}
```

---

# 6. 完整示例 — 订单异步处理

## 6.1 DTO

```java
package com.example.rabbitmq.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderDTO implements Serializable {
    private String orderId;
    private String userId;
    private BigDecimal amount;
    private String status;
    private LocalDateTime createTime;
}
```

## 6.2 Service

```java
@Service
@Slf4j
public class OrderService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public void createOrder(OrderDTO order) {
        // 1. 保存订单到数据库
        order.setStatus("CREATED");
        order.setCreateTime(LocalDateTime.now());
        orderRepository.save(order);

        // 2. 发送消息到 RabbitMQ
        CorrelationData correlationData = new CorrelationData(order.getOrderId());
        rabbitTemplate.convertAndSend("order.direct", "order.create",
                order, correlationData);

        log.info("订单创建成功: {}", order.getOrderId());
    }
}
```

## 6.3 Controller

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping
    public Result<String> createOrder(@RequestBody CreateOrderRequest request) {
        OrderDTO order = new OrderDTO();
        order.setOrderId(UUID.randomUUID().toString());
        order.setUserId(request.getUserId());
        order.setAmount(request.getAmount());

        orderService.createOrder(order);
        return Result.success(order.getOrderId());
    }
}
```

## 6.4 Consumer（短信通知）

```java
@Component
@Slf4j
public class OrderNotificationConsumer {

    @RabbitListener(queues = "order.sms.queue")
    @RabbitListener(queues = "order.email.queue")
    public void handleNotification(OrderDTO order) {
        log.info("处理订单通知: orderId={}, userId={}, amount={}",
                order.getOrderId(), order.getUserId(), order.getAmount());
        // 发送短信/邮件...
    }
}
```

---

# 7. 事务与 Confirm 在 Spring 中的使用

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
```

```java
@Service
public class ReliableProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        // Confirm 回调
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            String id = correlationData != null ? correlationData.getId() : "null";
            if (ack) {
                log.info("[Confirm Success] id={}", id);
            } else {
                log.error("[Confirm Failed] id={}, cause={}", id, cause);
                // 持久化到 DB 或重试
            }
        });

        // Return 回调
        rabbitTemplate.setReturnsCallback(returned -> {
            log.error("[Return] msg={}, replyCode={}, replyText={}",
                    new String(returned.getMessage().getBody()),
                    returned.getReplyCode(),
                    returned.getReplyText());
        });
    }

    public void sendReliable(String exchange, String routingKey, Object payload) {
        CorrelationData cd = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.convertAndSend(exchange, routingKey, payload, cd);
    }
}
```

---

# 8. 异步确认（CompletableFuture）

```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    return template;
}

@Service
public class AsyncConfirmService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public CompletableFuture<Boolean> sendAsync(String exchange, String key, Object msg) {
        CorrelationData cd = new CorrelationData(UUID.randomUUID().toString());

        CompletableFuture<Boolean> future = new CompletableFuture<>();
        rabbitTemplate.convertAndSend(exchange, key, msg, cd);

        // Confirm 回调中完成 Future
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (correlationData != null && correlationData.getId().equals(cd.getId())) {
                if (ack) {
                    future.complete(true);
                } else {
                    future.completeExceptionally(new RuntimeException(cause));
                }
            }
        });

        return future;
    }
}
```

---

# 9. 常用配置速查

| 配置项 | 说明 | 建议值 |
|--------|------|--------|
| `publisher-confirm-type` | 发布确认模式 | `correlated` |
| `publisher-returns` | 路由失败回退 | `true` |
| `acknowledge-mode` | 消费确认模式 | `manual` |
| `prefetch` | 预取数量 | `1~250` 按场景 |
| `concurrency` | 并发消费者数 | CPU 核数 |
| `max-concurrency` | 最大并发消费者数 | 按吞吐需求 |
| `requested-heartbeat` | 心跳间隔（秒） | `30~60` |
| `connection-timeout` | 连接超时（毫秒） | `30000` |

---

# 10. Spring AMQP vs 原生客户端

| 维度 | 原生 amqp-client | Spring AMQP |
|------|------------------|-------------|
| 连接管理 | 手动管理 Connection/Channel | CachingConnectionFactory 自动管理 |
| 消息序列化 | 手动 byte[] | MessageConverter 自动序列化 |
| 声明资源 | channel.exchangeDeclare() | @Bean 声明式 / @Exchange/@Queue 注解 |
| 消费消息 | channel.basicConsume() | @RabbitListener 注解驱动 |
| 异常处理 | 手动 try-catch | 声明式 ErrorHandler / Retry |
| 事务/确认 | 手动 confirmSelect + 回调 | publish-confirm-type 配置 |
| 模板操作 | channel.basicPublish() | rabbitTemplate.convertAndSend() |
