# 1. 死信队列概述

## 1.1 什么是死信（Dead Letter）
在RabbitMQ中，**死信（Dead Letter）**是指那些因为某些原因无法被正常消费的消息。这些消息不会被直接丢弃，而是可以被重新路由到专门的交换机（死信交换机）和队列（死信队列）中进行特殊处理。

## 1.2 消息变成死信的条件
一条消息在满足以下**任一条件**时会变成死信：

| 条件 | 说明 | 适用场景 |
|------|------|---------|
| 消息被拒绝（Nack/Reject）且requeue=false | 消费者明确拒绝且不让消息重新入队 | 业务逻辑判断消息无效 |
| 消息过期（TTL到期） | 消息在队列中存活时间超过了TTL | 超时订单取消、验证码过期 |
| 队列达到最大长度 | 队列中的消息数量超过max-length限制 | 流量控制，防止队列无限增长 |

## 1.3 死信队列的工作流程
```
正常流程:
  生产者 → Exchange → Queue → Consumer（正常消费、Ack）

死信流程:
  生产者 → Exchange → 业务Queue（消息变死信）
                            ↓
                     Dead Letter Exchange（DLX）
                            ↓
                     死信Queue（DLQ）
                            ↓
                     死信消费者（特殊处理）
```

# 2. 死信队列的配置

## 2.1 声明死信Exchange和死信Queue
```java
@Configuration
public class DeadLetterConfig {

    // ========== 死信基础设施 ==========

    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange("dlx.order.exchange", true, false);
    }

    @Bean
    public Queue deadLetterQueue() {
        return new Queue("dlq.order.queue", true);
    }

    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
                .to(deadLetterExchange())
                .with("dlx.order.dead");
    }

    // ========== 业务队列（配置了死信交换机） ==========

    @Bean
    public Queue orderQueue() {
        Map<String, Object> args = new HashMap<>();
        // 核心配置：指定死信交换机
        args.put("x-dead-letter-exchange", "dlx.order.exchange");
        // 指定死信路由键（可选，不指定则使用原消息的RoutingKey）
        args.put("x-dead-letter-routing-key", "dlx.order.dead");
        // 消息TTL（毫秒）
        args.put("x-message-ttl", 30000);
        // 队列最大长度
        args.put("x-max-length", 10000);

        return new Queue("order.queue", true, false, false, args);
    }

    // 或使用QueueBuilder
    @Bean
    public Queue orderQueue2() {
        return QueueBuilder.durable("order.queue")
                .deadLetterExchange("dlx.order.exchange")
                .deadLetterRoutingKey("dlx.order.dead")
                .ttl(30000)
                .maxLength(10000L)
                .build();
    }
}
```

## 2.2 @QueueBinding声明（可选方式）
```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(
        value = "order.queue",
        arguments = {
            @Argument(name = "x-dead-letter-exchange", value = "dlx.order.exchange"),
            @Argument(name = "x-dead-letter-routing-key", value = "dlx.order.dead"),
            @Argument(name = "x-message-ttl", value = "30000", type = "java.lang.Integer")
        }
    ),
    exchange = @Exchange(value = "order.exchange", type = ExchangeTypes.TOPIC),
    key = "order.create"
))
public void handleOrder(Order order) {
    // 处理订单
}
```

# 3. 死信消息的处理

## 3.1 死信消费者
```java
@Component
public class DeadLetterConsumer {

    @RabbitListener(queues = "dlq.order.queue")
    public void handleDeadLetter(Order order, Message message,
                                  @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey) {
        // 判断死信原因
        String reason = getDeathReason(message);

        System.out.println("收到死信消息: " + order);
        System.out.println("死信原因: " + reason);
        System.out.println("原RoutingKey: " + routingKey);

        switch (reason) {
            case "expired":
                handleExpiredOrder(order);
                break;
            case "rejected":
                handleRejectedOrder(order);
                break;
            case "maxlen":
                handleOverflowOrder(order);
                break;
            default:
                handleUnknownDeadLetter(order);
        }
    }

    /**
     * 从消息头部解析死信原因
     */
    private String getDeathReason(Message message) {
        // RabbitMQ在消息变成死信时会在header中添加x-death信息
        List<Map<String, Object>> deathHeaders = message.getMessageProperties()
                .getXDeathHeader();
        if (deathHeaders != null && !deathHeaders.isEmpty()) {
            Map<String, Object> death = deathHeaders.get(0);
            return (String) death.get("reason");
        }
        return "unknown";
    }

    private void handleExpiredOrder(Order order) {
        // 处理过期订单（如自动取消）
        orderService.cancelOrder(order.getId(), "订单超时未支付");
    }

    private void handleRejectedOrder(Order order) {
        // 处理被拒绝的订单（如记录到数据库，人工处理）
        orderService.markAsDeadLetter(order.getId());
    }

    private void handleOverflowOrder(Order order) {
        // 处理因队列溢出而死信的消息
        alertService.sendAlert("订单队列溢出，消息进入死信");
    }

    private void handleUnknownDeadLetter(Order order) {
        // 未知原因，记录日志
        log.warn("未知原因的死信消息: {}", order);
    }
}
```

## 3.2 x-death头部信息
当消息变成死信时，RabbitMQ会在消息头部添加`x-death`信息，记录了消息的"死亡"过程：

```java
// x-death 头部结构示例
[
    {
        "reason": "expired",      // 死信原因：expired/rejected/maxlen
        "count": 1,              // 成为死信的次数
        "exchange": "order.exchange",  // 原Exchange
        "queue": "order.queue",       // 原Queue
        "routing-keys": ["order.create"],  // 原RoutingKey
        "time": 1715678900,            // 死信发生时间
        "original-expiration": "30000" // 原始TTL
    }
]
```

## 3.3 防止死信循环
如果死信队列的消费者又拒绝消息（requeue=false），消息可能再次成为死信，形成循环。

解决方案：
1. **死信队列不配置死信交换机**（最简单）
2. **记录x-death.count**，超过阈值后丢弃
3. **将最终失败的消息存入数据库**，不再作为死信重新路由

```java
@Component
public class DeadLetterConsumer {

    @RabbitListener(queues = "dlq.order.queue")
    public void handleDeadLetter(Order order, Message message, Channel channel) throws Exception {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        
        // 获取死信次数
        List<Map<String, Object>> deathHeaders = 
            message.getMessageProperties().getXDeathHeader();
        int deathCount = deathHeaders != null ? deathHeaders.size() : 0;
        
        if (deathCount > 3) {
            // 死信超过3次，存入数据库，不再重新入队
            log.error("消息死信次数超过阈值: {}", order);
            deadLetterRepository.save(order, message);
            channel.basicAck(deliveryTag, false);
            return;
        }
        
        try {
            // 尝试重新处理
            orderService.compensate(order);
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            // 处理失败，拒绝（进入死信循环的下一次路由）
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

# 4. 死信队列的应用场景

## 4.1 延迟订单取消
```java
// 场景：下单后30分钟内未支付，自动取消订单

@Configuration
public class OrderTimeoutConfig {

    @Bean
    public Queue orderTimeoutQueue() {
        return QueueBuilder.durable("order.timeout.queue")
                .ttl(30 * 60 * 1000)  // 30分钟TTL
                .deadLetterExchange("order.exchange")
                .deadLetterRoutingKey("order.cancel")
                .build();
    }
}

// 下单时发送延迟消息
@Service
public class OrderService {

    public void createOrder(Order order) {
        // 1. 保存订单
        orderRepository.save(order);
        
        // 2. 发送延迟消息，30分钟后检查支付状态
        rabbitTemplate.convertAndSend("", "order.timeout.queue", 
            new OrderTimeoutMessage(order.getId()));
    }

    // 3. 处理"订单取消"消息（即30分钟后被投递到死信队列的消息）
    @RabbitListener(queues = "order.timeout.queue")
    public void checkOrderPayStatus(OrderTimeoutMessage message) {
        // 此方法不会被调用，因为消息会在TTL到期后进入死信
        // 这里只是为了声明绑定
    }

    @RabbitListener(queues = "order.cancel.queue")
    public void handleOrderCancel(OrderTimeoutMessage message) {
        Order order = orderRepository.findById(message.getOrderId());
        if (order.getStatus() == OrderStatus.UNPAID) {
            // 30分钟了还没支付，取消订单
            orderService.cancelOrder(order.getId(), "超时未支付");
        }
    }
}
```

> **注意**：更灵活的实现请参考"延迟队列与消息重试"章节。

## 4.2 消息拒绝处理
```java
// 消费者明确拒绝无效消息，不重试
@RabbitListener(queues = "order.queue")
public void handleOrder(Order order, Channel channel, Message message) throws Exception {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    
    if (order.getAmount() == null || order.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
        System.out.println("无效订单（金额为空或小于等于0），进入死信");
        channel.basicReject(deliveryTag, false);  // requeue=false → 进入死信
        return;
    }
    
    // 正常处理
    processOrder(order);
    channel.basicAck(deliveryTag, false);
}
```

## 4.3 队列溢出保护
```java
// 配置队列最大长度，超出后消息进入死信
@Bean
public Queue orderQueue() {
    return QueueBuilder.durable("order.queue")
        .maxLength(10000L)  // 最多10000条消息
        .overflow(QueueBuilder.Overflow.rejectPublish) // 超出后拒绝新消息
        .deadLetterExchange("dlx.order.exchange")
        .deadLetterRoutingKey("dlx.order.overflow")
        .build();
}
```

## 4.4 消息生命周期追踪
死信队列可以用来追踪消息的完整生命周期：

```
业务Queue → 死信Exchange → 死信Queue → 死信消费者:
  - 记录到数据库
  - 发送告警
  - 人工处理
  - 定时任务自动重试
```

# 5. 死信队列的多个场景整合

## 5.1 为不同死信原因设置不同处理
```java
@Configuration
public class MultiDLXConfig {

    // 过期死信交换机
    @Bean
    public DirectExchange expireDLX() {
        return new DirectExchange("dlx.expire.exchange", true, false);
    }

    @Bean
    public Queue expireDLQueue() {
        return new Queue("dlq.expire.queue", true);
    }

    // 拒绝死信交换机
    @Bean
    public DirectExchange rejectDLX() {
        return new DirectExchange("dlx.reject.exchange", true, false);
    }

    @Bean
    public Queue rejectDLQueue() {
        return new Queue("dlq.reject.queue", true);
    }

    // 溢出死信交换机
    @Bean
    public DirectExchange overflowDLX() {
        return new DirectExchange("dlx.overflow.exchange", true, false);
    }

    @Bean
    public Queue overflowDLQueue() {
        return new Queue("dlq.overflow.queue", true);
    }
}
```

# 6. 死信队列的监控与运维

## 6.1 监控指标
```java
@Component
public class DeadLetterMonitor {

    @Autowired
    private AmqpAdmin amqpAdmin;

    @Scheduled(fixedDelay = 30000) // 每30秒检查一次
    public void monitorDeadLetterQueue() {
        QueueInformation info = amqpAdmin.getQueueInfo("dlq.order.queue");
        if (info != null && info.getMessageCount() > 100) {
            // 死信队列堆积超过100条，告警
            alertService.sendAlert("死信队列堆积: " + info.getMessageCount());
        }
    }
}
```

## 6.2 管理界面的死信信息
在RabbitMQ管理界面中，Queue详情页会显示死信相关信息：
- `DLX`：关联的死信交换机
- `DLK`：关联的死信路由键
- 消息详情的`x-death`头部可以查看消息的死亡过程

## 6.3 死信队列的清理
```java
@Component
public class DeadLetterCleanup {

    @Autowired
    private AmqpAdmin amqpAdmin;

    /**
     * 定期清理死信队列（保留最近N条）
     */
    @Scheduled(cron = "0 0 3 * * ?")  // 每天凌晨3点
    public void cleanupOldDeadLetters() {
        // 注意：RabbitMQ不支持直接按条件删除消息
        // 通常的做法是：
        // 1. 消费死信消息
        // 2. 判断是否需要保留
        // 3. Ack掉需要清理的消息
    }
    
    /**
     * 手动清空死信队列（危险操作，仅限紧急情况）
     */
    public void emergencyPurge() {
        amqpAdmin.purgeQueue("dlq.order.queue", true);
    }
}
```
