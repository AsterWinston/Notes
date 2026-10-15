# 1. 消息接收概览

Spring AMQP为消息接收提供了两种主要方式：
- **Push模式（推荐）**：通过`@RabbitListener`注解，Broker主动推送消息给消费者
- **Pull模式**：通过`RabbitTemplate.receive()`手动拉取消息

本章重点介绍Push模式，因为这是Spring AMQP最推荐、最常用的消费方式。

# 2. @RabbitListener详解

## 2.1 基本用法
```java
@Component
public class OrderMessageListener {

    /**
     * 最简单的用法：监听指定队列
     */
    @RabbitListener(queues = "order.create.queue")
    public void handleOrderCreate(Order order) {
        System.out.println("收到订单创建消息: " + order);
    }
}
```

**工作原理**：应用启动后，`RabbitListenerAnnotationBeanPostProcessor`会扫描所有`@RabbitListener`注解的方法，为每个方法创建一个`MessageListenerContainer`，自动管理连接、线程、消息分发。

## 2.2 注解属性详解
```java
@RabbitListener(
    id = "orderListener",                           // 监听器唯一ID（可选，默认自动生成）
    containerFactory = "rabbitListenerContainerFactory", // 容器工厂Bean名（可选）
    queues = {"order.create.queue", "order.cancel.queue"}, // 监听队列列表
    bindings = @QueueBinding(                        // 动态声明绑定（与queues二选一）
        value = @Queue(value = "my.queue", durable = "true"),
        exchange = @Exchange(value = "my.exchange", type = ExchangeTypes.TOPIC),
        key = "order.#"
    ),
    concurrency = "5-10",                            // 消费者线程数范围
    autoStartup = "true",                            // 是否自动启动
    priority = "10",                                 // 消费者优先级
    exclusive = false,                               // 是否排他消费者
    admin = "rabbitAdmin",                           // RabbitAdmin Bean名
    ackMode = "AUTO",                                // 确认模式
    returnExceptions = "false",                      // 异常是否返回给调用方
    errorHandler = "myErrorHandler"                  // 错误处理器Bean名
)
public void handleMessage(Order order) {
    // 处理消息
}
```

## 2.3 监听单个队列
```java
@Component
public class SingleQueueListener {

    @RabbitListener(queues = "email.notify.queue")
    public void handleEmailNotification(EmailNotification notification) {
        // 处理邮件通知
        emailService.send(notification);
    }
}
```

## 2.4 监听多个队列
```java
@Component
public class MultiQueueListener {

    /**
     * 同一个方法监听多个队列
     */
    @RabbitListener(queues = {"order.create.queue", "order.cancel.queue"})
    public void handleOrderMessage(Order order) {
        // 通过判断消息类型或其他属性来区分来源
        processOrder(order);
    }

    private void processOrder(Order order) {
        // 统一处理
    }
}
```

## 2.5 不同队列不同方法
```java
@Component
public class SeparateQueueListener {

    @RabbitListener(queues = "order.create.queue")
    public void handleOrderCreate(Order order) {
        System.out.println("订单创建: " + order);
    }

    @RabbitListener(queues = "order.cancel.queue")
    public void handleOrderCancel(Order order) {
        System.out.println("订单取消: " + order);
    }

    @RabbitListener(queues = "order.refund.queue")
    public void handleOrderRefund(RefundRequest refund) {
        System.out.println("退款请求: " + refund);
    }
}
```

# 3. @RabbitListener与@QueueBinding

## 3.1 动态声明绑定
使用`@QueueBinding`可以在方法上直接声明Queue、Exchange和Binding，无需在单独的配置类中定义：

```java
@Component
public class DeclarativeListener {

    @RabbitListener(bindings = @QueueBinding(
        value = @Queue(
            value = "order.create.queue",
            durable = "true",
            arguments = {
                @Argument(name = "x-dead-letter-exchange", value = "dlx.exchange"),
                @Argument(name = "x-dead-letter-routing-key", value = "dlx.order.create")
            }
        ),
        exchange = @Exchange(value = "order.topic.exchange", type = ExchangeTypes.TOPIC),
        key = {"order.create", "order.new"}
    ))
    public void handleOrder(Order order) {
        System.out.println("收到订单: " + order);
    }
}
```

## 3.2 @QueueBinding的优缺点
优点：
- 代码更紧凑，业务逻辑和队列声明在一起
- 适合Demo和简单项目

缺点：
- 配置分散在多个Listener中，不便于集中管理
- 多个Listener引用同一个Exchange/Queue时会产生重复声明
- 不符合关注点分离原则

**建议**：
- 生产项目中，将Exchange、Queue、Binding的定义放在独立的`@Configuration`类中
- `@RabbitListener`只使用`queues`属性引用已定义好的队列名称常量
- `@QueueBinding`适用于Demo、原型项目

# 4. 消息参数绑定

## 4.1 支持的参数类型
`@RabbitListener`方法可以接收以下类型的参数：

```java
@RabbitListener(queues = "order.queue")
public void handleMessage(

    // 1. 消息体（自动反序列化）
    Order order,                              // 消息体转换为Order对象

    // 2. Message对象（Spring AMQP封装）
    Message message,                          // 完整的Spring AMQP Message

    // 3. 原生amqp-client消息对象
    com.rabbitmq.client.Message amqpMessage,  // 原生AMQP消息（需要设置setMessageConverter）

    // 4. Channel对象（用于手动确认）
    Channel channel,                          // AMQP Channel

    // 5. @Header注解：获取单个消息头
    @Header("x-user-id") String userId,      // 获取自定义头部

    // 6. @Headers注解：获取所有消息头
    @Headers Map<String, Object> headers,    // 所有消息头

    // 7. MessageHeaders（Spring消息头）
    MessageHeaders messageHeaders            // Spring消息头封装

) {
    // ...
}
```

## 4.2 详细示例
```java
@Component
public class AdvancedParameterListener {

    @RabbitListener(queues = "order.queue")
    public void handleMessage(
            Order order,                              // 消息体
            Message message,                          // Spring AMQP Message
            Channel channel,                          // AMQP Channel
            @Header("x-trace-id") String traceId,     // 自定义Header
            @Header(AmqpHeaders.CONSUMER_QUEUE) String queueName,  // 来源队列
            @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey // 来源路由键
    ) {
        System.out.println("消息体: " + order);
        System.out.println("Trace ID: " + traceId);
        System.out.println("来源队列: " + queueName);
        System.out.println("路由键: " + routingKey);
        System.out.println("Message ID: " + message.getMessageProperties().getMessageId());
    }
}
```

## 4.3 @Header详解
```java
// 获取单个Header（默认值）
@RabbitListener(queues = "order.queue")
public void handle(
        @Header(value = "x-retry-count", defaultValue = "0") int retryCount) {
    System.out.println("重试次数: " + retryCount);
}

// 获取AmqpHeaders中的标准头
@Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag;
@Header(AmqpHeaders.CONSUMER_TAG) String consumerTag;
@Header(AmqpHeaders.CONSUMER_QUEUE) String queueName;
@Header(AmqpHeaders.RECEIVED_EXCHANGE) String exchangeName;
@Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey;
@Header(AmqpHeaders.REDELIVERED) boolean redelivered;
```

## 4.4 参数获取消息体类型
当方法的第一个参数类型与消息体转换后的类型不匹配时，`MessageConverter`会尝试进行类型转换。如果转换失败，将抛出`MessageConversionException`。

```java
// 消息体是JSON: {"name": "张三", "age": 25}
@RabbitListener(queues = "user.queue")
public void handleUser(User user) {
    // MessageConverter会自动将JSON反序列化为User对象
}

// 消息体是纯文本
@RabbitListener(queues = "notify.queue")
public void handleText(String text) {
    // 纯文本消息，直接作为String
}

// 消息体是二进制
@RabbitListener(queues = "file.queue")
public void handleBytes(byte[] bytes) {
    // 二进制消息
}
```

# 5. 手动确认（Manual Ack）

## 5.1 确认模式概述
AMQP协议定义了三种消费者确认模式：

| 模式 | 说明 | Spring AMQP配置 |
|------|------|----------------|
| none | 自动Ack，Broker发送消息后立即认为已被消费 | `AcknowledgeMode.NONE` |
| auto | 自动Ack，方法正常返回则Ack，异常则Nack（可配置重新入队） | `AcknowledgeMode.AUTO`（默认） |
| manual | 手动Ack，需要显式调用channel.basicAck/Nack | `AcknowledgeMode.MANUAL` |

## 5.2 auto模式（默认）
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto  # 默认
        default-requeue-rejected: false  # Nack后不重新入队
```

在auto模式下：
- 方法**正常返回** → 自动Ack，消息从队列中删除
- 方法**抛出异常** → 自动Nack → 如果`default-requeue-rejected=true`则重新入队，否则丢弃或进入死信队列

## 5.3 manual模式（手动确认）
当需要精确控制消息确认时机时，使用手动确认模式：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual
```

### 5.3.1 基本用法
```java
import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class ManualAckListener {

    @RabbitListener(queues = "order.queue")
    public void handleOrder(Order order, Message message, Channel channel) throws Exception {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();

        try {
            // 处理业务逻辑
            orderService.process(order);

            // 确认消息（deliveryTag, 是否批量确认）
            channel.basicAck(deliveryTag, false);
            System.out.println("消息确认成功: deliveryTag=" + deliveryTag);

        } catch (Exception e) {
            System.out.println("消息处理失败: " + e.getMessage());

            // 拒绝消息（deliveryTag, 是否批量拒绝, 是否重新入队）
            channel.basicNack(deliveryTag, false, true);  // 重新入队
            // 或
            // channel.basicReject(deliveryTag, true);    // 单条拒绝（是否重新入队）
        }
    }
}
```

### 5.3.2 basicAck vs basicNack vs basicReject
| 方法 | 说明 | 签名 |
|------|------|------|
| basicAck | 确认消息 | (deliveryTag, multiple) |
| basicNack | 拒绝消息 | (deliveryTag, multiple, requeue) |
| basicReject | 拒绝单条消息 | (deliveryTag, requeue) |

- `multiple=true`：确认/拒绝deliveryTag之前的所有消息（包括当前）
- `requeue=true`：消息重新放回队首（可能导致无限循环）

### 5.3.3 手动确认的最佳实践
```java
@RabbitListener(queues = "critical.order.queue")
public void handleCriticalOrder(Order order, Message message, Channel channel) throws Exception {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();

    try {
        // 1. 幂等性检查（根据业务ID判断是否已处理）
        String orderId = order.getId();
        if (isAlreadyProcessed(orderId)) {
            channel.basicAck(deliveryTag, false);  // 已处理，直接确认
            return;
        }

        // 2. 执行业务逻辑
        orderService.process(order);

        // 3. 成功后确认
        channel.basicAck(deliveryTag, false);

    } catch (OrderNotFoundException e) {
        // 业务异常（无效消息）：不要重新入队，避免死循环
        System.out.println("无效订单: " + e.getMessage());
        channel.basicNack(deliveryTag, false, false); // requeue=false

    } catch (Exception e) {
        // 服务端异常（如数据库连接断开）：重新入队
        System.out.println("处理失败，重新入队: " + e.getMessage());
        channel.basicNack(deliveryTag, false, true);  // requeue=true
    }
}
```

## 5.4 none模式
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: none
```

`none`模式下，消息一旦发出即视为已消费，不管消费者是否处理成功。这是**不可靠的**，消息可能丢失，一般不推荐使用。

# 6. 消费重试机制

## 6.1 消费者重试的两种实现
Spring AMQP提供了两种消费重试机制：

1. **Spring Retry（应用层重试）**：在消费者方法内部重试，对Broker无感知
2. **RabbitMQ特性（Broker层重试）**：通过死信队列+延迟队列实现

## 6.2 Spring Retry（应用层重试）
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true           # 启用重试
          max-attempts: 3         # 最大重试次数（包含第一次）
          initial-interval: 2000  # 初始重试间隔（毫秒）
          multiplier: 2.0        # 乘数（每次间隔乘以此值）
          max-interval: 10000    # 最大重试间隔
```

工作原理：
```
第1次尝试 → 失败（等待initial-interval，如2秒）
第2次重试 → 失败（等待initial-interval * multiplier，如4秒）
第3次重试 → 失败 → 抛出AmqpRejectAndDontRequeueException → 消息被丢弃或进入死信
```

## 6.3 重试的注意事项
1. **幂等性**：重试意味着同一消息可能被多次处理，业务逻辑必须保证幂等性
2. **重试耗尽策略**：重试全部失败后，需要定义消息的最终去向
3. **避免重新入队导致无限循环**：正确配置`default-requeue-rejected=false`

## 6.4 重试耗尽后的处理策略
```java
@Configuration
public class RetryConfig {

    @Bean
    public MessageRecoverer messageRecoverer(RabbitTemplate rabbitTemplate) {
        // 方案一：重试耗尽后将消息发送到死信队列
        return new RepublishMessageRecoverer(
            rabbitTemplate,
            "dlx.exchange",       // 死信交换机
            "dlx.order.routing"   // 死信路由键
        );
        
        // 方案二：记录日志后丢弃（不推荐生产环境）
        // return new RejectAndDontRequeueRecoverer();
    }
}

// 配置完整的重试策略
@Bean
public RetryOperationsInterceptor retryOperationsInterceptor(
        RabbitTemplate rabbitTemplate) {
    return RetryInterceptorBuilder.stateless()
        .maxAttempts(5)
        .backOffOptions(2000, 2.0, 10000)  // 初始间隔、乘数、最大间隔
        .recoverer(new RepublishMessageRecoverer(
            rabbitTemplate, "dlx.exchange", "dlx.retry.routing"))
        .build();
}
```

# 7. 消费者并发控制

## 7.1 配置消费者线程数
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        concurrency: 3       # 最小消费者线程数
        max-concurrency: 10  # 最大消费者线程数
        prefetch: 100        # 每个消费者预取消息数
```

## 7.2 Preview
```java
@RabbitListener(queues = "high.throughput.queue", concurrency = "5-20")
public void handleMessage(Message message) {
    // 此监听器的消费者线程数在5到20之间动态调整
}
```

## 7.3 并发配置建议
| 场景 | concurrency | prefetch | 说明 |
|------|------------|----------|------|
| 低延迟、匀速场景 | 1-3 | 250 | 普通业务 |
| 高吞吐场景 | 10-20 | 100 | 需要快速处理大量消息 |
| 密集IO场景 | CPU核数*2 | 50 | 需要等待IO（DB/网络） |
| 顺序处理场景 | 1 | 1 | 严格保证顺序 |
| 削峰填谷 | 1-10 | 500 | 灵活伸缩 |

# 8. 消费者优先级

## 8.1 消费者优先级机制
RabbitMQ支持消费者优先级，具有更高优先级的消费者在消息分发时会被优先考虑。这在需要多个消费者一起工作，但希望某些消费者优先处理消息时很有用（如主备消费者模式）。

```java
@RabbitListener(queues = "order.queue", priority = "10")
public void highPriorityConsumer(Order order) {
    // 优先级10的消费者
}

@RabbitListener(queues = "order.queue", priority = "5")
public void normalPriorityConsumer(Order order) {
    // 优先级5的消费者
}
```

## 8.2 排他消费者
```java
@RabbitListener(queues = "order.queue", exclusive = true)
public void exclusiveConsumer(Order order) {
    // 排他消费者：此队列只能通过这个消费者消费
    // 如果已有其他消费者，会报错
}
```

# 9. 条件消费与消息过滤

## 9.1 应用层过滤
在Listener方法内部进行条件判断：

```java
@RabbitListener(queues = "order.queue")
public void handleOrder(Order order) {
    // 应用层过滤：只处理创建类型的订单
    if (!"CREATE".equals(order.getType())) {
        return; // 不处理此消息（auto模式会自动Ack）
    }
    // 处理订单创建
    orderService.process(order);
}
```

## 9.2 消息过滤
```java
// 在@RabbitListener中指定MessageConverter来做过滤
// 或者使用CompositeMessageConverter
```

实际上RabbitMQ推荐的做法是：**在发送端就通过Exchange/Binding将消息路由到正确的队列**，而不是在消费端做大量过滤。

# 10. 异常处理

## 10.1 全局错误处理器
```java
@Component
public class CustomErrorHandler implements RabbitListenerErrorHandler {

    @Override
    public Object handleError(Message message, Message<?> springMessage,
                              ListenerExecutionFailedException exception) {
        // 记录错误日志
        System.out.println("消息处理失败: " + exception.getMessage());
        System.out.println("消息体: " + new String((byte[]) message.getPayload()));
        
        // 可以执行补偿逻辑
        // 返回null表示丢弃消息
        return null;
    }
}

// 在@RabbitListener中使用
@RabbitListener(queues = "order.queue", errorHandler = "customErrorHandler")
public void handleOrder(Order order) {
    // 处理逻辑
}
```

## 10.2 @RabbitListener的异常返回
```java
@RabbitListener(queues = "order.queue", returnExceptions = "true")
public void handleOrder(Order order) {
    // returnExceptions=true意味着异常会被返回给容器
    // 通常配合errorHandler使用
}
```

## 10.3 常见异常处理模式
```java
@RabbitListener(queues = "order.queue")
public void handleOrder(Order order, Message message, Channel channel) throws Exception {
    try {
        // 业务逻辑
        orderService.process(order);
    } catch (BusinessException e) {
        // 业务异常：拒绝且不重入队
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
        // 记录失败信息到数据库供后续处理
        logFailure(order.getId(), e.getMessage());
    } catch (DatabaseException e) {
        // 数据库异常：重新入队等待恢复
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
    }
}
```

# 11. 消息生命周期监听

## 11.1 RabbitListenerEndpointRegistry
可以通过`RabbitListenerEndpointRegistry`来管理监听器的生命周期：

```java
@Component
public class ListenerManager {

    @Autowired
    private RabbitListenerEndpointRegistry registry;

    // 启动指定监听器
    public void startListener(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        if (container != null) {
            container.start();
        }
    }

    // 停止指定监听器
    public void stopListener(String listenerId) {
        MessageListenerContainer container = registry.getListenerContainer(listenerId);
        if (container != null) {
            container.stop();
        }
    }

    // 获取所有监听器容器
    public Collection<MessageListenerContainer> getAllContainers() {
        return registry.getListenerContainers();
    }
}
```

> 注意：需要在`@RabbitListener`中指定`id`属性才能通过Registry获取特定的监听器容器。

## 11.2 启动和停止控制
```java
@RabbitListener(queues = "order.queue", id = "orderListener", autoStartup = "false")
public void handleOrder(Order order) {
    // autoStartup=false 表示不自动启动，需要手动启动
}
```
