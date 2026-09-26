
# 1. 延迟消息概述

RocketMQ 支持**18 个固定的延迟级别**，让消息在指定时间后才被消费者可见。

```
应用场景：
- 订单超时未支付自动关闭 (30分钟后)
- 会议开始前提醒 (5分钟前)
- 定时推送通知 (第二天早上8点)
- 延迟重试 (失败后10s再试)
```

## 1.1 延迟消息 vs 定时消息

| 类型 | 精度 | 实现方式 | RocketMQ 支持 |
|------|------|----------|---------------|
| **延迟消息** | 固定级别 | `delayLevel` 指定延迟等级 | 原生支持 |
| **定时消息** | 任意时间 | 指定投递时间戳 | 5.0 支持（TimerMessage） |

# 2. 延迟级别定义

```properties
# broker.conf — messageDelayLevel
messageDelayLevel = 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

| Level | 延迟时间 | Level | 延迟时间 |
|-------|----------|-------|----------|
| 1 | 1s | 10 | 6m |
| 2 | 5s | 11 | 7m |
| 3 | 10s | 12 | 8m |
| 4 | 30s | 13 | 9m |
| 5 | 1m | 14 | 10m |
| 6 | 2m | 15 | 20m |
| 7 | 3m | 16 | 30m |
| 8 | 4m | 17 | 1h |
| 9 | 5m | 18 | 2h |

**注意**：默认不支持自定义延迟时间，也不支持 "第3秒" 这样的任意时间点（5.0 以前）。

# 3. 使用示例

## 3.1 Producer 发送延迟消息

```java
public class DelayProducer {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("delay_producer");
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.start();

        Message msg = new Message("DelayTopic", "TagA",
            "这是一条延迟消息".getBytes());
        // 设置延迟级别为 3 (对应 10s 延迟)
        msg.setDelayTimeLevel(3);

        Producer.send(msg);

        System.out.println("发送时间: " + new Date());
        producer.shutdown();
    }
}
```

## 3.2 Consumer 消费延迟消息

```java
// Consumer 代码无任何特殊处理，透明消费延迟消息
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("delay_consumer");
consumer.setNamesrvAddr("127.0.0.1:9876");
consumer.subscribe("DelayTopic", "*");
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        for (MessageExt msg : msgs) {
            System.out.println("收到消息时间: " + new Date() 
                + ", 消息内容: " + new String(msg.getBody()));
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
consumer.start();
```

## 3.3 运行结果

```
# Producer 输出：
发送时间: Mon Jan 01 10:00:00 CST 2024

# Consumer 输出 (大约10s后):
收到消息时间: Mon Jan 01 10:00:10 CST 2024, 消息内容: 这是一条延迟消息
```

# 4. 延迟消息实现原理

## 4.1 整体流程

```
Producer 发送延迟消息 (msg.setDelayTimeLevel(3) → 10s)
    │
    ▼
Broker CommitLog 写入
    │
    ├── 将消息的 Topic 改为 SCHEDULE_TOPIC_XXXX
    ├── 将消息的 QueueId 改为 delayLevel - 1 (即 2)
    ├── 将原始 Topic/QueueId 存储在 properties 中
    │    → REAL_TOPIC    = "DelayTopic"
    │    → REAL_QID      = "0"
    └── 提交存储
    │
    ▼
ScheduledMessageService 定时扫描
    │
    ├── Timer 定时器定时扫描 SCHEDULE_TOPIC_XXXX 的各 Queue
    ├── 根据 delayLevel 计算投递时间
    ├── delayTime = storeTimestamp + delayLevel对应的时间
    │
    ▼
到期消息处理
    │
    ├── 校验 delayTime ≤ now
    ├── 恢复原始 Topic 和 QueueId
    ├── 清理延迟消息属性
    └── 重新写入 CommitLog（作为普通消息）
    │
    ▼
Consumer 正常消费
```

## 4.2 延迟消息存储

```
SCHEDULE_TOPIC_XXXX = 包含了 18 个 Queue（对应 18 个延迟级别）
  ├── Queue-0 → delayLevel 1 的消息
  ├── Queue-1 → delayLevel 2 的消息
  ├── Queue-2 → delayLevel 3 的消息
  ├── ...
  └── Queue-17 → delayLevel 18 的消息
```

## 4.3 ScheduledMessageService 核心逻辑

```java
public class ScheduledMessageService extends ConfigManager {
    // 定时器 Timer，每个延迟级别一个 TimerTask
    // Timer 的间隔 = 对应级别的延迟时间
    private final ConcurrentHashMap<Integer /* level */, Long /* delay time in millis */> 
        delayLevelTable = new ConcurrentHashMap<>();

    @Override
    public void init() {
        // 解析 messageDelayLevel 配置
        // 为每个级别注册定时任务
        for (Map.Entry<Integer, Long> entry : delayLevelTable.entrySet()) {
            Integer level = entry.getKey();
            Long timeDelay = entry.getValue();
            // 为每个延迟级别创建一个 DeliverDelayedMessageTimerTask
        }
    }

    // 核心调度类
    class DeliverDelayedMessageTimerTask implements Runnable {
        @Override
        public void run() {
            // 1. 获取对应级别的 ConsumeQueue (SCHEDULE_TOPIC_XXXX, queueId=level-1)
            // 2. 从 offset 开始扫描
            // 3. 计算每条的 deliveryTime
            // 4. 到期 → 重新写入 CommitLog
            // 5. 未到期 → break（后面的消息也未到期）
        }
    }
}
```

## 4.4 关键算法 — 消息投递时间判断

```java
// ScheduledMessageService 中的投递时间计算
// ConsumeQueue 记录: {commitLogOffset(8B), msgSize(4B), tagsCode(8B)}
// tagsCode = 存储了消息的 deliveryTimestamp（实际投递时间）
// storeTimestamp = 消息存入 CommitLog 的时间

// 计算投递时间
Long deliveryTime = storeTimestamp + delayLevelTable.get(delayLevel);
// deliveryTime 被编码存储在 ConsumeQueue 的 tagsCode 字段中

// 判断是否到期：
if (deliveryTime <= System.currentTimeMillis()) {
    // 到期 → 投递
    deliver(messageExtInner);
} else {
    // 未到期 → break
    break;
}
```

# 5. 自定义延迟级别

```properties
# broker.conf
messageDelayLevel = 1s 3s 5s 10s 30s 1m 2m 3m 5m 10m 15m 20m 30m 1h 2h 3h 6h 12h
```

**注意**：
- 仍然是 18 个级别（数量固定）
- 每个级别的具体时间可自定义
- 所有 Broker 配置必须一致
- 修改后需重启 Broker

# 6. RocketMQ 5.0 定时消息

RocketMQ 5.0 引入了更灵活的 **TimerMessage** 机制：

```java
// 5.0 支持任意时间点的延迟
msg.setDeliveryTimestamp(System.currentTimeMillis() + 30 * 60 * 1000);
// 30 分钟后投递（精确到毫秒）
```

与 4.x 的区别：
- 可以指定任意延迟时间，不再限于 18 个级别
- 使用时间轮算法 (TimingWheel) 替代固定级别
- 精度更高，但实现更复杂

# 7. 注意事项

1. **延迟级别从 1 开始**，0 表示非延迟消息
2. **不支持任意延迟**（4.x）：需要使用 18 个固定级别之一
3. **性能开销**：延迟消息需要两次写入 CommitLog（第一次写入 SCHEDULE_TOPIC，第二次投递）
4. **延迟时间** = storeTimestamp + delayLevel对应时间，非 Producer 发送时间
5. **高延迟级别**可能积累大量消息，注意磁盘和内存
6. **避免大量短期延迟**：如 1s 延迟的消息量过大，会增加扫描开销

# 8. 参考

- [RocketMQ 延迟消息](https://rocketmq.apache.org/docs/4.x/producer/04message2/)
- [ScheduledMessageService 源码](https://github.com/apache/rocketmq/blob/master/broker/src/main/java/org/apache/rocketmq/broker/schedule/ScheduledMessageService.java)
