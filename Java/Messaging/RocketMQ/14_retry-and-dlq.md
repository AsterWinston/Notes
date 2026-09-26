
# 1. 消息重试概述

当 Consumer 消费消息失败时，RocketMQ 支持**自动重试**，重试次数达上限后将消息转入**死信队列（DLQ, Dead Letter Queue）**。

```
消费失败处理流程：

消息消费失败
    │
    ├── 重试消费 (最多 16 次)
    │    Level 1: 10s 后重试
    │    Level 2: 30s 后重试
    │    Level 3: 1m 后重试
    │    Level 4: 2m 后重试
    │    Level 5: 3m 后重试
    │    ...
    │    Level 16: 2h 后重试
    │
    └── 16 次后仍失败 → 进入 Dead Letter Queue
```

# 2. 消费重试机制

## 2.1 并发消费的重试

```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        try {
            process(msgs);
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        } catch (Exception e) {
            // 返回 RECONSUME_LATER → 消息会重试
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            // 注意：如果消费失败但没有返回 RECONSUME_LATER（比如吞了异常），
            // 框架认为消费成功，不会重试
        }
    }
});
```

## 2.2 顺序消费的重试

```java
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeOrderlyContext context) {
        try {
            process(msgs);
            return ConsumeOrderlyStatus.SUCCESS;
        } catch (Exception e) {
            // 返回 SUSPEND_CURRENT_QUEUE_A_MOMENT → 当前队列挂起一段时间
            return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
        }
    }
});
```

## 2.3 重试时间间隔

```java
// 默认重试延迟级别（与延迟消息共用延迟级别）
// SubscriptionGroupConfig.retryQueueNums
// 重试间隔:
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
```

# 3. Consumer 发送重试消息的流程

```
消费失败 (RECONSUME_LATER)
    │
    ▼
Consumer → Broker (ConsumerSendMsgBack)
    │
    ├── 1. 增加消息的 reconsumeTimes
    ├── 2. 构建新的消息
    │    ├── Topic = %RETRY%{consumerGroup} (重试 Topic)
    │    ├── QueueId = 上次消费的 QueueId
    │    ├── 根据 reconsumeTimes 计算 delayLevel
    │    └── 写入 CommitLog
    │
    ▼
延迟消息机制处理 (ScheduledMessageService)
    │
    ├── 等待 delayLevel 对应的时间
    └── 到期后重新投递
    │
    ▼
Consumer 重新消费（reconsumeTimes 已增加）
```

## 3.1 重试 Topic 命名

```
重试消息的 Topic 格式: %RETRY%{consumerGroup}
例如: %RETRY%order_consumer_group
```

每个 Consumer Group 对应一个重试 Topic，重试消息在该 Topic 下管理。

# 4. 死信队列 (DLQ)

## 4.1 触发条件

```
进入死信队列的条件：
消息的 reconsumeTimes >= maxReconsumeTimes (默认 16)
```

## 4.2 死信 Topic 命名

```
死信 Topic 格式: %DLQ%{consumerGroup}
例如: %DLQ%order_consumer_group
```

## 4.3 死信消息的特性

```java
// 死信消息的属性
MessageExt msg = ...;
msg.getTopic();         // %DLQ%order_consumer_group
msg.getReconsumeTimes(); // 16 (或更多)
msg.getProperty(MessageConst.PROPERTY_ORIGIN_MESSAGE_ID);  // 原始消息 ID
msg.getProperty(MessageConst.PROPERTY_ORIGIN_TOPIC);        // 原始 Topic
```

## 4.4 死信消息处理

```java
// 方案 1：单独消费死信队列
DefaultMQPushConsumer dlqConsumer = new DefaultMQPushConsumer("dlq_handler");
dlqConsumer.subscribe("%DLQ%order_consumer_group", "*");
dlqConsumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        for (MessageExt msg : msgs) {
            // 记录到数据库
            String originMsgId = msg.getProperty(MessageConst.PROPERTY_ORIGIN_MESSAGE_ID);
            String realTopic = msg.getProperty(MessageConst.PROPERTY_ORIGIN_TOPIC);
            // 人工处理或告警
            log.error("死信消息: originTopic={}, originMsgId={}, body={}",
                realTopic, originMsgId, new String(msg.getBody()));
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
dlqConsumer.start();

// 方案 2：通过 Admin/GUI 手动重发死信消息
```

# 5. 消费幂等保证

由于重试机制可能导致同一条消息被多次消费，必须实现**消费幂等**：

## 5.1 方案一：数据库唯一索引

```java
@Transactional
public void consumeMessage(MessageExt msg) {
    String msgId = msg.getMsgId();
    try {
        // 插入消费记录表（msg_id 有唯一索引）
        consumeRecordMapper.insert(new ConsumeRecord(msgId, msg.getTopic()));
        // 执行业务逻辑
        processBusiness(msg);
    } catch (DuplicateKeyException e) {
        // 已消费过，忽略
        log.warn("消息重复: {}", msgId);
    }
}
```

## 5.2 方案二：Redis 去重

```java
public void consumeMessage(MessageExt msg) {
    String msgKey = "msg:consumed:" + msg.getMsgId();
    // SET NX EX: 不存在才设置，设置过期时间
    Boolean success = redisTemplate.opsForValue()
        .setIfAbsent(msgKey, "1", 72, TimeUnit.HOURS);
    if (Boolean.TRUE.equals(success)) {
        processBusiness(msg);
    } else {
        log.warn("消息已消费: {}", msg.getMsgId());
    }
}
```

## 5.3 方案三：业务状态机

```java
// 订单状态：PENDING → PROCESSING → DONE
public void consumeMessage(MessageExt msg) {
    String orderId = msg.getKeys();
    // SELECT * FROM orders WHERE order_id = #{orderId} AND status = 'PENDING'
    // UPDATE orders SET status = 'PROCESSING' WHERE ... AND status = 'PENDING'
    int rows = orderMapper.updateStatus(orderId, "PENDING", "PROCESSING");
    if (rows > 0) {
        processBusiness(msg);
        orderMapper.updateStatus(orderId, "PROCESSING", "DONE");
    }
}
```

# 6. 关键配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `maxReconsumeTimes` | 16 | 最大重试次数 |
| `retryQueueNums` | 1 | 重试队列数量 |
| `consumeMessageBatchMaxSize` | 1 | 并发模式下每次消费最大消息数 |
| `consumeTimeout` | 15min | 顺序消费超时时间 |

```java
// 消费者端配置
consumer.setMaxReconsumeTimes(5);  // 减少重试次数，加速死信
```

# 7. Retry 与 DLQ Topic 的自动创建

```
首次重试时，Broker 自动创建：
- %RETRY%{consumerGroup} (retryTopic)
  配置:
  - ReadQueueNums = 1
  - WriteQueueNums = 1
  - Perm = 6 (可读可写)

首次进入死信时，Broker 自动创建：
- %DLQ%{consumerGroup} (deadLetterTopic)
  配置:
  - ReadQueueNums = 1
  - WriteQueueNums = 1
  - Perm = 6
```

# 8. 最佳实践

1. **catch 异常并返回 RECONSUME_LATER**：不要吞掉异常
2. **设置合理的 maxReconsumeTimes**：根据业务调整（交易类 3-5 次，通知类 10-16 次）
3. **死信监控告警**：死信队列有消息时及时告警
4. **幂等是必须的**：重试 = 重复消费
5. **区分可恢复和不可恢复的错误**：
   - 可恢复（超时、网络抖动）→ RECONSUME_LATER
   - 不可恢复（数据格式错误）→ 记录日志，返回 SUCCESS 避免死循环
6. **重试间隔匹配业务**：配置合适的 `messageDelayLevel`，避免过于密集

# 9. 参考

- [RocketMQ 消费重试](https://rocketmq.apache.org/docs/4.x/consumer/03message2/)
