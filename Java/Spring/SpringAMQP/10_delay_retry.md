# 1.延迟队列概述

## 1.1什么是延迟队列
延迟队列是一种特殊的消息队列，消息被发送后不会立即被消费，而是在经过指定的延迟时间后才会被路由到目标队列。

## 1.2延迟队列的实现方式
RabbitMQ本身不直接支持延迟队列功能，需要通过以下方式实现：

| 实现方式 | 原理 | 优点 | 缺点 |
|---------|------|------|------|
| TTL + 死信队列 | 消息到期后进入死信队列 | 无需额外插件 | TTL粒度不灵活，队列级别TTL限制 |
| 延迟插件 | RabbitMQ官方插件rabbitmq_delayed_message_exchange | 灵活性高，每条消息可指定不同延迟 | 需要安装额外插件 |
| 第二种方式(每级延迟一个队列) | 为不同延迟时间创建多个死信队列 | 无需插件 | 配置复杂，粒度有限 |

# 2.TTL + 死信队列实现

## 2.1TTL（Time To Live）详解
TTL是消息或队列的存活时间。RabbitMQ支持两种TTL设置：

### 2.1.1队列级别TTL
在队列声明时设置，此队列中的所有消息都遵循相同的TTL：

```java
@Bean
public Queue delayQueue() {
    return QueueBuilder.durable("delay.30s.queue")
        .ttl(30000)  // 队列中所有消息30秒后过期
        .deadLetterExchange("target.exchange")
        .deadLetterRoutingKey("target.routing")
        .build();
}
```

### 2.1.2消息级别TTL
在发送消息时设置，每条消息可以有不同的TTL：

```java
rabbitTemplate.convertAndSend("delay.exchange", "delay.routing", order,
    msg -> {
        msg.getMessageProperties().setExpiration("30000");  // 30秒
        return msg;
    });
```

### 2.1.3TTL的重要特性
> **关键限制**：如果同时设置了队列TTL和消息TTL，以**较小的值**为准。消息过期时会被**从队首开始移除**（非严格按过期时间），这意味着队首的消息TTL较长时，会阻塞后面已过期的消息移出队列。

这导致了消息级别TTL的一个重要问题：
```
Queue: [消息A(TTL=5分钟), 消息B(TTL=30秒), 消息C(TTL=30秒)]
       ↑ 队首                                   ↑ 已过期
       
消息B和C已经过期，但因为消息A还在队首等待，B和C无法被移出队列。
直到消息A过期或被消费，B和C才会被移出。
```

## 2.2多级延迟队列实现
解决TTL粒度问题：为不同的延迟时间创建不同的队列。

```java
@Configuration
public class MultiLevelDelayConfig {

    // 延迟时间常量（毫秒）
    public static final long DELAY_5S = 5 * 1000;
    public static final long DELAY_10S = 10 * 1000;
    public static final long DELAY_30S = 30 * 1000;
    public static final long DELAY_1M = 60 * 1000;
    public static final long DELAY_5M = 5 * 60 * 1000;
    public static final long DELAY_30M = 30 * 60 * 1000;

    // ========== 目标交换机（延迟后的消息真正到达的交换机） ==========
    @Bean
    public DirectExchange targetExchange() {
        return new DirectExchange("order.target.exchange", true, false);
    }

    @Bean
    public Queue targetQueue() {
        return new Queue("order.target.queue", true);
    }

    @Bean
    public Binding targetBinding() {
        return BindingBuilder.bind(targetQueue())
                .to(targetExchange())
                .with("order.target");
    }

    // ========== 各级延迟队列 ==========

    @Bean
    public Queue delay5sQueue() {
        return QueueBuilder.durable("delay.5s.queue")
                .ttl((int) DELAY_5S)
                .deadLetterExchange("order.target.exchange")
                .deadLetterRoutingKey("order.target")
                .build();
    }

    @Bean
    public Queue delay30sQueue() {
        return QueueBuilder.durable("delay.30s.queue")
                .ttl((int) DELAY_30S)
                .deadLetterExchange("order.target.exchange")
                .deadLetterRoutingKey("order.target")
                .build();
    }

    @Bean
    public Queue delay5mQueue() {
        return QueueBuilder.durable("delay.5m.queue")
                .ttl((int) DELAY_5M)
                .deadLetterExchange("order.target.exchange")
                .deadLetterRoutingKey("order.target")
                .build();
    }
}
```

发送延迟消息：
```java
@Service
public class DelayMessageService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送延迟消息（通过死信队列）
     */
    public void sendDelayMessage(Object message, long delayMillis) {
        String delayQueue = getDelayQueue(delayMillis);
        rabbitTemplate.convertAndSend(delayQueue, message);
    }

    /**
     * 根据延迟时间选择对应的延迟队列
     */
    private String getDelayQueue(long delayMillis) {
        if (delayMillis <= 5000) return "delay.5s.queue";
        if (delayMillis <= 30000) return "delay.30s.queue";
        if (delayMillis <= 300000) return "delay.5m.queue";
        throw new IllegalArgumentException("不支持的延迟时间: " + delayMillis);
    }
}
```

# 3.延迟插件实现（推荐）

## 3.1延迟插件介绍
`rabbitmq_delayed_message_exchange`是RabbitMQ官方团队开发的插件（Community Plugin），它允许声明延迟交换机，每条消息可以指定独立的延迟时间。

## 3.2安装延迟插件
```bash
# 1. 下载插件（版本需与RabbitMQ版本匹配）
# https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases

# 2. 将插件文件放入RabbitMQ的plugins目录
# cp rabbitmq_delayed_message_exchange-3.13.0.ez \
#    /usr/lib/rabbitmq/plugins/

# 3. 启用插件
# rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 4. 重启RabbitMQ
# systemctl restart rabbitmq-server
```

## 3.3Spring AMQP配置延迟交换机
```java
@Configuration
public class DelayedPluginConfig {

    @Bean
    public Exchange delayedExchange() {
        // 声明一个延迟交换机（本质上是x-delayed-type交换机）
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");  // 内部实际使用的交换机类型

        return new CustomExchange(
            "order.delayed.exchange",   // 交换机名称
            "x-delayed-message",         // 延迟交换机类型（固定值）
            true,                        // 持久化
            false,                       // 不自动删除
            args
        );
    }

    @Bean
    public Queue delayedQueue() {
        return new Queue("order.delayed.queue", true);
    }

    @Bean
    public Binding delayedBinding() {
        return BindingBuilder.bind(delayedQueue())
                .to(delayedExchange())
                .with("order.delayed")
                .noargs();  // 延迟交换机不需要RoutingKey匹配参数
    }
}
```

## 3.4使用延迟插件发送消息
```java
@Service
public class DelayedMessageService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送延迟消息（使用延迟插件）
     */
    public void sendDelayedMessage(Order order, long delayMillis) {
        rabbitTemplate.convertAndSend(
            "order.delayed.exchange",
            "order.delayed",
            order,
            msg -> {
                // 关键：在消息头部设置 x-delay（毫秒）
                msg.getMessageProperties().setHeader("x-delay", delayMillis);
                return msg;
            }
        );
    }

    /**
     * 下单后30分钟检查支付状态
     */
    public void schedulePaymentCheck(Order order) {
        long delay = 30 * 60 * 1000;  // 30分钟
        sendDelayedMessage(new PaymentCheckTask(order.getId()), delay);
    }
}

// 消费延迟消息
@Component
public class PaymentCheckConsumer {

    @RabbitListener(queues = "order.delayed.queue")
    public void checkPayment(PaymentCheckTask task) {
        Order order = orderService.findById(task.getOrderId());
        if (order != null && order.isUnpaid()) {
            orderService.cancelOrder(order.getId(), "支付超时");
        }
    }
}
```

## 3.5延迟插件的优缺点
优点：
- 灵活性高：每条消息可以指定不同的延迟时间
- 无需创建多个队列：一个交换机+一个队列即可
- 配置简单

缺点：
- 需要额外安装插件
- 延迟消息存储在Mnesia表中，有性能限制（不适合大量消息堆积）
- 延迟消息在Broker重启时会丢失（不持久化到磁盘）

# 4.消息重试机制

## 4.1两种重试层级

### 4.1.1Spring Retry（应用层重试）
在消费者方法抛出异常时，Spring AMQP在应用内部进行重试（不涉及Broker）：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
          multiplier: 2.0
          max-interval: 10000
```

Spring Retry的重试逻辑：
```
第1次调用 → 失败
等待1秒
第2次调用 → 失败
等待2秒（1*2）
第3次调用 → 失败
→ 抛出AmqpRejectAndDontRequeueException → 消息被Nack(requeue=false) → 进入死信
```

### 4.1.2Broker层重试（延迟+死信）
当应用层重试全部失败后，消息进入死信队列，可以在死信队列的消费者中实现更"重量级"的重试逻辑：

```java
@Component
public class DeadLetterRetryConsumer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @RabbitListener(queues = "dlq.order.queue")
    public void retryDeadLetter(Order order, Message message, Channel channel) throws Exception {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();

        // 获取重试次数
        Integer retryCount = message.getMessageProperties()
                .getHeader("x-retry-count");
        if (retryCount == null) retryCount = 0;

        if (retryCount >= 3) {
            // 超过最大重试次数，记录到数据库，人工处理
            log.error("消息重试耗尽: {}", order);
            failureRepository.save(new MessageFailure(order.getId(), 
                new String(message.getBody())));
            channel.basicAck(deliveryTag, false);
            return;
        }

        try {
            // 重新发送到业务队列（带重试计数）
            rabbitTemplate.convertAndSend("order.exchange", "order.create", order,
                msg -> {
                    msg.getMessageProperties().setHeader("x-retry-count", retryCount + 1);
                    msg.getMessageProperties().setExpiration(
                        String.valueOf(calculateRetryDelay(retryCount + 1)));
                    return msg;
                }
            );

            // 确认死信消息
            channel.basicAck(deliveryTag, false);

        } catch (Exception e) {
            // 发送失败，保留在死信队列
            channel.basicNack(deliveryTag, false, true);
        }
    }

    /**
     * 计算重试延迟（指数退避）
     */
    private long calculateRetryDelay(int retryCount) {
        long baseDelay = 10000;  // 10秒
        return baseDelay * (long) Math.pow(2, retryCount - 1);
        // 第1次重试: 10秒
        // 第2次重试: 20秒
        // 第3次重试: 40秒
    }
}
```

## 4.2指数退避策略
为了尽可能不丢失消息且不过度重试，推荐使用指数退避（Exponential Backoff）：

```java
public class RetryDelayCalculator {

    /**
     * 计算指数退避延迟
     * @param attempt 第几次重试（从1开始）
     * @param baseDelay 基础延迟（毫秒）
     * @param maxDelay 最大延迟（毫秒）
     */
    public static long calculate(int attempt, long baseDelay, long maxDelay) {
        long delay = baseDelay * (long) Math.pow(2, attempt - 1);
        return Math.min(delay, maxDelay);
    }

    // 示例：
    // attempt=1: baseDelay * 1   = 10s
    // attempt=2: baseDelay * 2   = 20s
    // attempt=3: baseDelay * 4   = 40s
    // attempt=4: baseDelay * 8   = 80s
    // attempt=5: baseDelay * 16  = 160s → 超过maxDelay → 取maxDelay
}
```

# 5.延迟队列的应用场景

## 5.1订单超时取消
```java
// 场景：下单后30分钟内未支付，自动取消订单
@Service
public class OrderTimeoutService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void scheduleOrderTimeoutCheck(String orderId) {
        OrderTimeoutMessage msg = new OrderTimeoutMessage(orderId);
        rabbitTemplate.convertAndSend(
            "order.delayed.exchange",
            "order.timeout",
            msg,
            message -> {
                // 30分钟后投递
                message.getMessageProperties().setHeader("x-delay", 30 * 60 * 1000);
                return message;
            }
        );
    }
}

@Component
public class OrderTimeoutConsumer {

    @RabbitListener(queues = "order.timeout.queue")
    public void handleOrderTimeout(OrderTimeoutMessage msg) {
        Order order = orderRepository.findById(msg.getOrderId());
        if (order != null && OrderStatus.UNPAID.equals(order.getStatus())) {
            orderService.cancel(order.getId(), "超时未支付");
            // 释放库存
            inventoryService.releaseStock(order.getItems());
        }
    }
}
```

## 5.2定时任务调度
```java
// 场景：每天凌晨3点执行报表生成
// 不推荐用MQ做定时任务（应该用XXL-JOB、Quartz等），这里仅供参考

// 每天计算下一轮执行时间并发送延迟消息
public void scheduleReportTask() {
    long delayTo3AM = calculateTimeToNext3AM();
    rabbitTemplate.convertAndSend("task.delayed.exchange", "task.report",
        new ReportTask("daily-sales"),
        msg -> {
            msg.getMessageProperties().setHeader("x-delay", delayTo3AM);
            return msg;
        }
    );
}
```

## 5.3验证码过期
```java
// 场景：发送验证码后5分钟过期
public void sendVerificationCode(String phone, String code) {
    // 1. 保存验证码到Redis（5分钟过期）
    redisTemplate.opsForValue().set("sms:" + phone, code, 5, TimeUnit.MINUTES);

    // 2. 发送延迟消息，5分钟后清除验证码记录
    rabbitTemplate.convertAndSend("sms.delayed.exchange", "sms.expire",
        new SmsExpireEvent(phone),
        msg -> {
            msg.getMessageProperties().setHeader("x-delay", 5 * 60 * 1000);
            return msg;
        }
    );
}
```

## 5.4重试场景：不可靠的外部服务调用
```java
// 场景：调用第三方支付接口失败后，延迟重试
public void callExternalPayment(Order order) {
    try {
        paymentGateway.pay(order);
    } catch (PaymentException e) {
        // 失败后延迟30秒重试
        rabbitTemplate.convertAndSend("payment.retry.exchange", "payment.retry",
            new PaymentRetryTask(order.getId()),
            msg -> {
                msg.getMessageProperties().setHeader("x-delay", 30000);
                return msg;
            }
        );
    }
}
```

# 6.延迟队列的选型建议

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 简单延迟需求（固定延迟时间） | TTL + 死信队列 | 无需额外插件，稳定可靠 |
| 灵活延迟需求（每条消息不同延迟） | 延迟插件 | 实现简单，配置灵活 |
| 大批量延迟消息 | TTL + 死信队列 | 延迟插件有性能瓶颈 |
| 高可靠性延迟（Broker重启不丢） | TTL + 死信队列 | 持久化支持更好 |
| 秒级延迟精确度 | 延迟插件 | TTL方案受队列队首阻塞影响 |
| 可选的替代方案 | Redis延迟队列 / 定时任务 | 当RabbitMQ延迟队列无法满足需求时 |

# 7.生产者重试（发送重试）

## 7.1RabbitTemplate发送重试
```yaml
spring:
  rabbitmq:
    template:
      retry:
        enabled: true
        max-attempts: 3
        initial-interval: 1000
        multiplier: 2.0
        max-interval: 10000
```

## 7.2自定义RetryTemplate
```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);

    RetryTemplate retryTemplate = new RetryTemplate();

    // 配置退避策略
    ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
    backOffPolicy.setInitialInterval(1000);
    backOffPolicy.setMultiplier(2.0);
    backOffPolicy.setMaxInterval(10000);
    retryTemplate.setBackOffPolicy(backOffPolicy);

    // 配置重试策略
    SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
    retryPolicy.setMaxAttempts(3);
    retryTemplate.setRetryPolicy(retryPolicy);

    template.setRetryTemplate(retryTemplate);
    return template;
}
```

# 8.对比总结

| 机制 | 实现方式 | 适用场景 | 复杂度 |
|------|---------|---------|--------|
| TTL + 死信 | 队列TTL + DLX | 固定延迟、批量消息 | 中 |
| 延迟插件 | x-delayed-message exchange | 灵活延迟 | 低 |
| Spring Retry | 应用层重试 | 临时性错误、网络抖动 | 低 |
| 死信重试 | DLQ + 重发 | 持久性错误恢复 | 高 |
| 指数退避 | 动态延迟计算 | 外部服务恢复 | 中 |
