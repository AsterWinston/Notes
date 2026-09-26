# 1.AMQP事务概述

## 1.1什么是AMQP事务
AMQP协议提供了事务机制，允许生产者将多条消息的发送操作放入一个事务中，**要么全部成功，要么全部失败**。这与数据库的事务概念类似，但实现方式不同。

AMQP事务的工作流程：
```
1. txSelect()    → 将Channel设置为事务模式
2. basicPublish() → 发送消息1（暂存）
3. basicPublish() → 发送消息2（暂存）
4. txCommit()    → 提交：所有消息被真正投递到Broker
   或
4. txRollback()  → 回滚：所有消息被丢弃
```

## 1.2事务的代价
AMQP事务会**严重降低性能**（吞吐量可能下降100-250倍），因为：
- 每条消息的发送都需要等待Broker的事务确认
- 事务模式下的Channel是同步的，不能并发发送
- Broker需要额外处理事务的提交和回滚逻辑

因此，**在生产环境中一般不推荐使用AMQP事务**，而是使用Publisher Confirm机制替代。

## 1.3事务 vs Publisher Confirm
| 对比维度 | AMQP事务 | Publisher Confirm |
|---------|---------|-------------------|
| 性能 | 极低（吞吐量下降100-250倍） | 较高（吞吐量下降10-20%） |
| 可靠性 | 同步确认，更可靠 | 异步确认，可能少量丢失 |
| 资源占用 | 高（Channel独占，同步等待） | 低（异步，不阻塞） |
| 复杂度 | 低（类似数据库事务） | 中（需要处理回调、CorrelationData） |
| 推荐度 | 不推荐 | 推荐 |

# 2.Spring AMQP中的事务配置

## 2.1开启事务支持
```java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    
    // 启用事务支持
    template.setChannelTransacted(true);
    
    return template;
}
```

或通过注解：
```yaml
spring:
  rabbitmq:
    template:
      channel-transacted: true
```

## 2.2使用@Transactional管理事务
Spring AMQP可以与Spring事务管理集成，通过`@Transactional`注解声明式管理RabbitMQ事务：

```java
@Service
public class TransactionalMessageService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private RabbitTransactionManager rabbitTransactionManager;

    /**
     * 事务中发送多条消息：要么全部成功，要么全部失败
     */
    @Transactional(transactionManager = "rabbitTransactionManager")
    public void sendWithTransaction(Order order, Payment payment) {
        // 消息1
        rabbitTemplate.convertAndSend("order.exchange", "order.create", order);
        
        // 消息2
        rabbitTemplate.convertAndSend("payment.exchange", "payment.create", payment);
        
        // 如果在此抛出异常，两条消息都会被回滚（即都不会发送到Broker）
    }
}
```

## 2.3配置RabbitTransactionManager
```java
@Bean
public RabbitTransactionManager rabbitTransactionManager(
        ConnectionFactory connectionFactory) {
    return new RabbitTransactionManager(connectionFactory);
}
```

## 2.4使用TransactionTemplate编程式事务
```java
@Service
public class ProgrammaticTxService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private RabbitTransactionManager transactionManager;

    public void sendMessagesInTransaction(Order order) {
        TransactionTemplate txTemplate = new TransactionTemplate(transactionManager);
        
        txTemplate.execute(status -> {
            try {
                rabbitTemplate.convertAndSend("order.exchange", "order.create", order);
                rabbitTemplate.convertAndSend("inventory.exchange", "inventory.deduct", 
                    order.getItems());
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

# 3.分布式事务：数据库 + MQ

## 3.1问题场景
常见的分布式事务场景：

```
用户下单 → 写入数据库（订单表） + 发送MQ消息（通知库存系统）
```

如何保证数据库写入和MQ发送两个操作的一致性？如果在数据库写入成功后MQ发送失败（或反过来），就会出现数据不一致。

## 3.2解决方案

### 3.2.1半消息 + 事务回查（RocketMQ原生支持）
RocketMQ原生支持事务消息，RabbitMQ没有这种能力。可以用以下方案替代。

### 3.2.2本地消息表（最经典的方案）
```
1. 在数据库中创建一个"消息发送表"（message_log）
2. 同一事务中操作：
   - 插入业务数据（订单）
   - 插入消息记录（message_log，状态=PENDING）
3. 事务提交后，执行业务逻辑
4. 定时任务扫描 message_log，将状态=PENDING的消息发送到MQ
5. 发送成功：更新状态=SUCCESS
6. 发送失败：记录失败次数，超过阈值后告警
```

```java
@Service
public class OrderServiceWithMessageTable {

    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private MessageLogRepository messageLogRepository;
    
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private PlatformTransactionManager transactionManager;

    /**
     * 下单（使用本地消息表保证数据库和MQ的一致性）
     */
    @Transactional
    public void createOrder(Order order) throws JsonProcessingException {
        // 1. 生成消息ID
        String messageId = UUID.randomUUID().toString();
        
        // 2. 在同一事务中：保存订单 + 保存消息记录
        order.setStatus(OrderStatus.CREATED);
        orderRepository.save(order);
        
        MessageLog messageLog = new MessageLog();
        messageLog.setMessageId(messageId);
        messageLog.setExchange("order.exchange");
        messageLog.setRoutingKey("order.create");
        messageLog.setMessageBody(objectMapper.writeValueAsString(order));
        messageLog.setStatus(MessageLogStatus.PENDING);
        messageLog.setCreateTime(LocalDateTime.now());
        messageLogRepository.save(messageLog);
        
        // 3. 事务提交后发送消息
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    try {
                        sendAndConfirm(messageLog);
                    } catch (Exception e) {
                        log.error("消息发送失败: {}", messageId, e);
                    }
                }
            }
        );
    }

    /**
     * 发送消息并使用Publisher Confirm确认
     */
    private void sendAndConfirm(MessageLog messageLog) {
        CorrelationData correlationData = new CorrelationData(messageLog.getMessageId());
        correlationData.getFuture().addCallback(
            result -> {
                if (result.isAck()) {
                    messageLogRepository.updateStatus(messageLog.getMessageId(), 
                        MessageLogStatus.SUCCESS);
                } else {
                    messageLogRepository.updateStatus(messageLog.getMessageId(), 
                        MessageLogStatus.FAILED);
                }
            },
            ex -> {
                messageLogRepository.updateStatus(messageLog.getMessageId(), 
                    MessageLogStatus.FAILED);
            }
        );
        
        rabbitTemplate.convertAndSend(
            messageLog.getExchange(),
            messageLog.getRoutingKey(),
            messageLog.getMessageBody(),
            correlationData
        );
    }
}
```

### 3.2.3定时补偿任务
```java
@Component
public class MessageCompensationTask {

    @Autowired
    private MessageLogRepository messageLogRepository;
    
    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 每分钟扫描一次，重发失败消息
     */
    @Scheduled(fixedDelay = 60000)
    public void resendPendingMessages() {
        // 查询状态为PENDING且超过1分钟的消息
        List<MessageLog> pendingMessages = messageLogRepository
            .findByStatusAndCreateTimeBefore(
                MessageLogStatus.PENDING,
                LocalDateTime.now().minusMinutes(1)
            );
        
        for (MessageLog log : pendingMessages) {
            // 检查重试次数
            if (log.getRetryCount() >= 5) {
                messageLogRepository.updateStatus(
                    log.getMessageId(), MessageLogStatus.FINAL_FAILED);
                alertService.sendAlert("消息投递失败，已达最大重试次数: " + log.getMessageId());
                continue;
            }
            
            try {
                CorrelationData data = new CorrelationData(log.getMessageId());
                rabbitTemplate.convertAndSend(
                    log.getExchange(), log.getRoutingKey(), 
                    log.getMessageBody(), data);
                messageLogRepository.incrementRetryCount(log.getMessageId());
            } catch (Exception e) {
                log.error("重发消息失败: {}", log.getMessageId(), e);
            }
        }
    }
}
```

# 4.RabbitMQ事务的底层原理

## 4.1AMQP事务命令
AMQP协议定义了以下事务相关命令：

| AMQP命令 | 说明 |
|---------|------|
| tx.select | 将当前Channel设置为事务模式 |
| tx.commit | 提交当前事务 |
| tx.rollback | 回滚当前事务 |

## 4.2事务的底层流程
```
Client (Producer)                    Broker (RabbitMQ)
      |                                    |
      |------- tx.select() -------------->|  Channel进入事务模式
      |                                    |
      |------- basic.publish(msg1) ------>|  msg1暂存（不真正投递）
      |------- basic.publish(msg2) ------>|  msg2暂存（不真正投递）
      |                                    |
      |------- tx.commit() -------------->|  将msg1、msg2真正投递到Exchange
      |                                    |
      |<------ tx.commit-ok --------------|  确认提交成功
```

## 4.3事务的局限性
1. **性能极差**：同步等待，无法批量确认
2. **不支持消费者事务**：AMQP事务只在生产者端有意义，消费者的事务通常用数据库事务代替
3. **不支持跨Channel事务**：事务范围仅限于单个Channel
4. **与高可用不兼容**：事务模式下无法使用Publisher Confirm

# 5.可靠消息方案的演进

## 5.1方案对比
| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|------|--------|------|--------|---------|
| AMQP事务 | 强一致 | 极低 | 低 | 几乎不推荐 |
| Publisher Confirm | 弱一致 | 高 | 中 | 一般业务 |
| 本地消息表 | 最终一致 | 中 | 中 | 核心业务 |
| 事务消息（RocketMQ）| 最终一致 | 较高 | 中 | 核心业务 |
| 最大努力通知 | 弱一致 | 高 | 低 | 非核心通知 |

## 5.2推荐方案
对于大多数Spring AMQP + RabbitMQ的分布式场景，推荐使用**本地消息表 + Publisher Confirm + 定时补偿**方案：
- 满足最终一致性要求
- 性能较好
- 实现复杂度适中
- 不依赖特定MQ特性（可以在不同MQ之间迁移）
