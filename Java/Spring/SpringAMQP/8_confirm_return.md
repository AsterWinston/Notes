# 1.消息可靠性概述

在分布式系统中，消息的可靠传递是核心问题。一条消息从生产者到消费者的完整旅程中，可能在多个环节丢失：

```
生产者 → [网络] → RabbitMQ Broker → [网络] → 消费者
  ↓                  ↓                    ↓
 发送阶段            存储阶段             消费阶段
```

RabbitMQ和Spring AMQP提供了一套完整的可靠性保障机制：

| 环节 | 机制 | 说明 |
|------|------|------|
| 发送阶段 | Publisher Confirm | 确认消息已到达Broker的Exchange |
| 路由阶段 | Return Callback | 消息无法路由到Queue时的回退通知 |
| 存储阶段 | 消息持久化 | 消息存入磁盘防止Broker宕机丢失 |
| 消费阶段 | Consumer Ack | 消费者确认消息已正确处理 |

# 2.Publisher Confirm（发送确认）

## 2.1什么是Publisher Confirm
Publisher Confirm（发布者确认）是RabbitMQ的一种可靠性机制：生产者发送消息后，Broker会异步返回一个确认（Ack或Nack），告知消息是否被成功接收。

AMQP协议中确认的粒度有两种：

| 类型 | 说明 | Spring AMQP配置 |
|------|------|----------------|
| SIMPLE | 所有消息共享一个确认标记，任意Nack都会中断Channel | `ConfirmType.SIMPLE` |
| CORRELATED | 每条消息有独立的确认，可精确定位哪条消息失败 | `ConfirmType.CORRELATED`（推荐） |

## 2.2底层AMQP原理
在AMQP协议层面，Publisher Confirm通过以下机制实现：

```
1. 生产者调用 channel.confirmSelect() 将Channel设置为Confirm模式
2. 发送消息时，Broker为每条消息分配一个 deliveryTag（递增序号）
3. Broker接收消息后，异步发送 basic.ack(deliveryTag, multiple) 或 basic.nack(deliveryTag, multiple)
4. 生产者通过 ConfirmListener 接收确认结果
```

## 2.3Spring AMQP配置

### 2.3.1开启Publisher Confirm
```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated  # 开启关联确认
```

或通过代码配置：
```java
@Bean
public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory factory = new CachingConnectionFactory();
    // ...基本配置...
    factory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
    return factory;
}
```

ConfirmType的三个可选值：
- `NONE`：不开启确认（默认）
- `SIMPLE`：简单确认模式（已过时，不推荐）
- `CORRELATED`：关联确认模式（推荐）

### 2.3.2实现ConfirmCallback
```java
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

@Component
public class MyConfirmCallback implements RabbitTemplate.ConfirmCallback {

    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        if (ack) {
            System.out.println("消息确认成功: " + correlationData);
        } else {
            System.out.println("消息确认失败: " + correlationData + ", 原因: " + cause);
            // 失败处理：记录日志、告警、重发等
        }
    }
}

// 在配置中设置ConfirmCallback
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                      MyConfirmCallback confirmCallback) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    template.setConfirmCallback(confirmCallback);
    return template;
}
```

### 2.3.3完整示例
```java
@Service
public class ReliableMessageSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送消息并等待确认
     */
    public void sendWithConfirm(Order order) {
        // 创建CorrelationData，用于关联发送和确认
        CorrelationData correlationData = new CorrelationData(order.getId());
        correlationData.getFuture().thenAccept(result -> {
            if (result.isAck()) {
                System.out.println("消息确认成功: " + order.getId());
            } else {
                System.out.println("消息确认失败: " + order.getId() + ", 原因: " + result.getReason());
            }
        });

        rabbitTemplate.convertAndSend(
            "order.exchange",
            "order.create",
            order,
            correlationData
        );
    }
}
```

## 2.4ConfirmCallback和ReturnsCallback的关系
两个回调的区别：

| 回调 | 触发条件 | 含义 |
|------|---------|------|
| ConfirmCallback | 消息到达Exchange（无论是否路由成功） | Broker确认收到消息 |
| ReturnsCallback | 消息到达Exchange但无法路由到Queue | 消息不可路由 |

```java
// 同时设置两个回调
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);

    // ConfirmCallback: 消息是否到达Exchange
    template.setConfirmCallback((correlationData, ack, cause) -> {
        if (ack) {
            System.out.println("消息已到达Exchange");
        } else {
            System.out.println("消息未到达Exchange: " + cause);
        }
    });

    // ReturnsCallback: 消息是否成功路由到Queue（需配合mandatory=true）
    template.setReturnsCallback(returned -> {
        System.out.println("消息无法路由: " + returned.getMessage());
        System.out.println("  ReplyCode: " + returned.getReplyCode());
        System.out.println("  ReplyText: " + returned.getReplyText());
        System.out.println("  Exchange: " + returned.getExchange());
        System.out.println("  RoutingKey: " + returned.getRoutingKey());
    });

    return template;
}
```

## 2.5ConfirmCallback的注意事项
1. **异步性**：ConfirmCallback是异步执行的，不要在回调中执行耗时操作
2. **多次回调**：如果`multiple=true`，一条Ack可能确认多条消息
3. **性能影响**：开启Confirm模式会降低吞吐量（约10-20%），需要在可靠性和性能之间权衡
4. **Channel独占**：启用Confirm的Channel上所有发布的消息都会被确认

# 3.Return Callback（回退回调）

## 3.1什么是Return Callback
当消息成功到达Exchange但**无法路由到任何Queue**时（如Routing Key没有匹配的Binding），如果设置了`mandatory=true`，Broker会将消息返回给生产者。

## 3.2开启Return Callback
```yaml
spring:
  rabbitmq:
    publisher-returns: true
    template:
      mandatory: true
```

或代码配置：
```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    
    // 开启强制投递：不可路由时回调
    template.setMandatory(true);
    
    // 设置ReturnCallback（Spring AMQP 3.x推荐使用setReturnsCallback）
    template.setReturnsCallback(returned -> {
        Message message = returned.getMessage();
        System.out.println("消息投递失败，无法路由到Queue:");
        System.out.println("  Exchange: " + returned.getExchange());
        System.out.println("  RoutingKey: " + returned.getRoutingKey());
        System.out.println("  ReplyText: " + returned.getReplyText());
        System.out.println("  MessageBody: " + new String(message.getBody()));
    });
    
    return template;
}
```

## 3.3Return Callback的触发场景
```java
// 场景：Exchange上没有任何Binding匹配此RoutingKey
rabbitTemplate.convertAndSend("order.exchange", "invalid.key", order);
// → 触发 ReturnsCallback

// 场景：Fanout Exchange，RoutingKey无意义也不会触发Return
// Fanout Exchange会忽略RoutingKey，消息广播到所有绑定队列，不会触发Return

// 场景：mandatory=false（默认值）
// 消息无法路由时被Broker静默丢弃，不触发任何回调
```

## 3.4Return Callback的注意事项
1. 只有`mandatory=true`时才会触发
2. 如果设置了备用交换机（Alternate Exchange），消息会进入备用交换机，不会触发Return Callback
3. 返回的消息保留了原始的所有属性（RoutingKey、Headers等）

# 4.可靠消息发送的完整方案

## 4.1消息落库+定时任务补偿
这是工业界常用的方案，确保消息100%投递成功：

```
1. 发送消息前：将消息记录存入数据库（状态=待发送）
2. 发送消息：使用Publisher Confirm确认
3. 发送成功：更新数据库状态=已发送
4. 定时任务：扫描状态为"待发送"且超过N分钟的消息，重新发送
```

### 4.1.1实现示例
```java
@Service
public class ReliableMessageService {

    @Autowired
    private RabbitTemplate rabbitTemplate;
    @Autowired
    private MessageLogRepository messageLogRepository;

    @Transactional
    public void sendReliableMessage(String exchange, String routingKey, Object message) {
        // 1. 生成消息ID
        String messageId = UUID.randomUUID().toString();
        
        // 2. 消息落库（与业务操作在同一事务中）
        MessageLog log = new MessageLog();
        log.setMessageId(messageId);
        log.setExchange(exchange);
        log.setRoutingKey(routingKey);
        log.setMessageBody(JSON.toJSONString(message));
        log.setStatus("PENDING");  // 待确认
        log.setCreateTime(new Date());
        messageLogRepository.save(log);
        
        // 3. 发送消息
        CorrelationData correlationData = new CorrelationData(messageId);
        correlationData.getFuture().addCallback(
            result -> {
                // 确认成功
                messageLogRepository.updateStatus(messageId, "SUCCESS");
            },
            ex -> {
                // 确认失败
                messageLogRepository.updateStatus(messageId, "FAILED");
            }
        );
        
        rabbitTemplate.convertAndSend(exchange, routingKey, message, correlationData);
    }
}
```

### 4.1.2定时补偿任务
```java
@Component
public class MessageResendTask {

    @Autowired
    private MessageLogRepository messageLogRepository;
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Scheduled(fixedDelay = 60000) // 每分钟执行一次
    public void resendFailedMessages() {
        // 查询状态为PENDING且超过1分钟的消息
        List<MessageLog> pendingMessages = messageLogRepository
            .findByStatusAndCreateTimeBefore("PENDING", 
                new Date(System.currentTimeMillis() - 60000));
        
        for (MessageLog log : pendingMessages) {
            try {
                CorrelationData data = new CorrelationData(log.getMessageId());
                rabbitTemplate.convertAndSend(
                    log.getExchange(),
                    log.getRoutingKey(),
                    log.getMessageBody(),
                    data
                );
                
                // 超过最大重试次数则标记为最终失败
                if (log.getRetryCount() >= 3) {
                    messageLogRepository.updateStatus(log.getMessageId(), "FINAL_FAILED");
                } else {
                    messageLogRepository.incrementRetryCount(log.getMessageId());
                }
            } catch (Exception e) {
                // 记录异常
            }
        }
    }
}
```

## 4.2消息落库的简化版（内存关联）
如果业务允许少量消息丢失，可以使用内存中的Map暂存未确认消息，而不需要数据库。

```java
@Component
public class InMemoryConfirmManager {

    private final ConcurrentHashMap<String, MessageRecord> pendingMessages = new ConcurrentHashMap<>();
    
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (correlationData != null) {
                String messageId = correlationData.getId();
                if (ack) {
                    pendingMessages.remove(messageId);
                    System.out.println("消息确认成功: " + messageId);
                } else {
                    MessageRecord record = pendingMessages.get(messageId);
                    if (record != null && record.getRetryCount() < 3) {
                        // 重发
                        resend(record);
                    }
                }
            }
        });
    }
    
    private void resend(MessageRecord record) {
        record.incrementRetryCount();
        CorrelationData data = new CorrelationData(record.getMessageId());
        rabbitTemplate.convertAndSend(record.getExchange(), 
            record.getRoutingKey(), record.getBody(), data);
    }
}
```

# 5.消费者确认（Consumer Ack）

## 5.1确认机制回顾
消费者确认已在"消息接收详解"章节中详细讲解。这里是快速回顾：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto   # none / auto / manual
        default-requeue-rejected: true  # Nack后是否重新入队
```

## 5.2消费端的可靠性保障
```
1. auto模式（默认）：方法正常返回 = Ack；抛出异常 = Nack
2. manual模式：显式调用 channel.basicAck / channel.basicNack
3. 重试机制：在消费端配置重试策略，容忍临时性错误
4. 死信队列：重试耗尽后的消息进入死信队列，人工处理
```

# 6.端到端的消息可靠性

## 6.1消息可靠性的完整链路
要保证消息不丢失，需要从端到端的全链路考虑：

```
1. 生产者端：
   - 消息落库（业务操作 + 消息记录在同一事务中）
   - Publisher Confirm（确认Broker已接收）
   - Return Callback（捕获无法路由的消息）
   - 定时补偿重发（处理未确认的消息）

2. Broker端：
   - Exchange持久化
   - Queue持久化
   - 消息持久化（deliveryMode=PERSISTENT）
   - 集群/镜像/仲裁队列（防止单点故障）
   - 惰性队列（防止内存溢出）

3. 消费者端：
   - Consumer Ack（确认已正确处理）
   - 消费重试（容忍临时故障）
   - 死信队列（处理最终失败的消息）
   - 幂等处理（防止重复消费）
```

## 6.2消息丢失的排查

| 丢失环节 | 典型现象 | 排查方法 |
|---------|---------|---------|
| 生产者到Broker | 消息根本没进入Broker | 检查ConfirmCallback，查看生产端日志 |
| Broker存储 | Broker重启后消息丢失 | 检查Exchange/Queue是否持久化，消息deliveryMode |
| 路由阶段 | 消息进Broker但没进Queue | 检查ReturnCallback，确认Binding配置 |
| Broker到消费者 | 消息在Queue中但消费者没收到 | 检查消费者连接状态、prefetch设置 |
| 消费处理 | 消费者收到但处理失败 | 检查异常处理、Ack模式、死信队列 |

# 7.消息可靠性最佳实践

## 7.1核心业务消息
对于核心业务（如订单、支付），推荐：
- 消息落库 + 定时补偿 + Publisher Confirm
- Queue设置为持久化，消息持久化
- 消费端manual Ack，业务处理成功后才确认

## 7.2非核心业务消息
对于非核心业务（如日志、通知），可以适当降低可靠性要求：
- Publisher Confirm（异步确认）
- 合理设置TTL，过期自动清理
- Auto Ack模式（性能优先）

## 7.3配置清单
```yaml
spring:
  rabbitmq:
    # 发送端
    publisher-confirm-type: correlated
    publisher-returns: true
    template:
      mandatory: true
      retry:
        enabled: true
        max-attempts: 3
    
    # 消费端
    listener:
      simple:
        acknowledge-mode: manual    # 核心业务用manual
        # acknowledge-mode: auto    # 非核心业务用auto
        prefetch: 1                # 确保公平分发
        retry:
          enabled: true
          max-attempts: 5
          initial-interval: 1000
          multiplier: 2.0
          max-interval: 10000
```
