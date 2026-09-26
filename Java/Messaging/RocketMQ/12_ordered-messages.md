
# 1. 顺序消息概述

顺序消息保证消息的消费顺序与发送顺序一致。RocketMQ 支持两种顺序级别：

| 类型 | 描述 | 实现方式 | 性能 |
|------|------|----------|------|
| **全局有序** | 整个 Topic 的消息按顺序消费 | 1个 Topic = 1个 Queue | 最低（单点瓶颈） |
| **分区有序** | 同一分区(Queue)内消息有序 | MessageQueueSelector 绑定 | 高（多 Queue 并行） |

## 1.1 为什么消息会乱序？

```
原因1: 多 MessageQueue 并行发送
  ┌─ Queue-0 → 消息1, 消息4
  └─ Queue-1 → 消息2, 消息3
  到达顺序可能为: 1, 2, 3, 4 而非 1, 2, 3, 4

原因2: 发送重试
  消息1 失败重试 → 消息2 已成功，消息1 后到

原因3: Consumer 多线程并发消费
  同一 Queue 采用线程池并发消费
```

# 2. 分区有序实现

## 2.1 Producer 端：绑定同一 Queue

```java
// 使用 MessageQueueSelector 将同一业务标识的消息发送到同一 Queue
SendResult result = producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Long orderId = (Long) arg;  // 以订单ID作为路由键
        int index = (int) (orderId % mqs.size());
        return mqs.get(index);
    }
}, orderId);
```

## 2.2 Consumer 端：顺序监听器

```java
DefaultMQPushConsumer consumer = 
    new DefaultMQPushConsumer("order_consumer_group");
consumer.setNamesrvAddr("127.0.0.1:9876");
consumer.subscribe("OrderTopic", "*");

// 使用 MessageListenerOrderly（而非 MessageListenerConcurrently）
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeOrderlyContext context) {
        // 同一 Queue 的消息会**串行**进入此方法
        // context.setAutoCommit(false); // 可手动控制提交
        for (MessageExt msg : msgs) {
            System.out.printf("Thread=%s, QueueId=%d, Body=%s%n",
                Thread.currentThread().getName(),
                msg.getQueueId(),
                new String(msg.getBody()));
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});

consumer.start();
```

## 2.3 顺序消费的线程模型

```
MessageListenerConcurrently (并发)：
  Queue-0 → Thread-1 → msg1
          → Thread-2 → msg2
          → Thread-3 → msg3
  可并发执行，不保证顺序

MessageListenerOrderly (顺序)：
  Queue-0 → Thread-A → msg1 → msg2 → msg3
  同一 Queue 由同一线程串行执行，保证顺序
```

# 3. 完整示例：订单处理

## 3.1 Producer

```java
public class OrderedProducer {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("orderly_producer");
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.start();

        String[] orders = {"创建订单", "支付订单", "发货", "完成"};
        for (long orderId = 1; orderId <= 5; orderId++) {
            for (int i = 0; i < orders.length; i++) {
                Message msg = new Message("OrderTopic", "OrderTag",
                    (orders[i] + " - orderId=" + orderId).getBytes());

                // 同一 orderId 路由到同一 Queue
                SendResult result = producer.send(msg, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> mqs, 
                            Message msg, Object arg) {
                        Long id = (Long) arg;
                        int index = (int) (id % mqs.size());
                        return mqs.get(index);
                    }
                }, orderId);

                System.out.printf("发送: orderId=%d, step=%s, queue=%d, offset=%d%n",
                    orderId, orders[i], result.getMessageQueue().getQueueId(),
                    result.getQueueOffset());
            }
        }
        producer.shutdown();
    }
}
```

## 3.2 Consumer

```java
public class OrderedConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("orderly_consumer");
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.subscribe("OrderTopic", "*");
        consumer.setConsumeThreadMin(1);
        consumer.setConsumeThreadMax(1);  // 单线程消费，确保顺序

        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(
                    List<MessageExt> msgs, ConsumeOrderlyContext context) {
                context.setAutoCommit(true);
                for (MessageExt msg : msgs) {
                    System.out.printf("消费: QueueId=%d, %s%n",
                        msg.getQueueId(), new String(msg.getBody()));
                }
                return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                // 消费失败时挂起当前队列，后续消息不会跳过
            }
        });

        consumer.start();
    }
}
```

## 3.3 输出结果

```
# Producer:
发送: orderId=1, step=创建订单, queue=0
发送: orderId=1, step=支付订单, queue=0
发送: orderId=1, step=发货,     queue=0
发送: orderId=1, step=完成,     queue=0

# Consumer (Queue-0 的消息严格按顺序消费):
消费: QueueId=0, 创建订单 - orderId=1
消费: QueueId=0, 支付订单 - orderId=1
消费: QueueId=0, 发货 - orderId=1
消费: QueueId=0, 完成 - orderId=1
```

# 4. 顺序消费源码解析

## 4.1 ConsumeMessageOrderlyService

```java
// Consumer 端处理顺序消费的核心类
public class ConsumeMessageOrderlyService implements ConsumeMessageService {

    // 这个线程池的核心特点：对每个 MessageQueue 都分配独立线程
    // 核心约束：同一 MessageQueue 的消息在同一时刻只被一个线程消费

    // 消费请求表：{MessageQueue → ConsumeRequest}
    // 如果某个 Queue 正在消费中，则不投递新请求
}
```

## 4.2 锁机制

```java
// 顺序消费需要 Broker 端分布式锁

// 1. Consumer 消费前需要获取锁
// 请求：REQUEST_CODE_LOCK_BATCH_MQ
this.brokerController.getRebalanceLockManager().tryLockBatch(lockOKMQSet);

// 2. 消费完成后释放锁
// 请求：REQUEST_CODE_UNLOCK_BATCH_MQ
this.brokerController.getRebalanceLockManager().unlockBatch(unlockMQSet);
```

**锁的作用**：
- 确保在 Rebalance 期间，消息不被重复分配到不同 Consumer
- 防止同一条消息被多个 Consumer 同时消费

# 5. ConsumeOrderlyStatus 状态说明

```java
public enum ConsumeOrderlyStatus {
    SUCCESS,                          // 消费成功，继续下一条
    SUSPEND_CURRENT_QUEUE_A_MOMENT,   // 稍后重试（暂停当前 Queue 一段时间）
    ROLLBACK,                         // 回滚（已废弃）
    COMMIT,                           // 提交（已废弃）
}
```

## 5.1 SUSPEND_CURRENT_QUEUE_A_MOMENT 机制

```
消费失败 → 返回 SUSPEND_CURRENT_QUEUE_A_MOMENT
    │
    ├── 当前 Queue 的消费暂停 (suspendCurrentQueueTimeMillis, 默认 1s)
    ├── 消息重新提交消费
    └── 重试次数超过 maxReconsumeTimes → 进入死信队列
```

# 6. 全局有序实现

```java
// 全局有序 = 1个 Topic 只有 1个 WriteQueue
// 方法1: Broker 创建 Topic 时指定 queueNum=1
// TopicConfig:
topicConfig.setReadQueueNums(1);
topicConfig.setWriteQueueNums(1);

// 方法2: Producer/Consumer 照常使用，无需特殊处理
// (因为只有一个 Queue，消息自然有序)
```

**全局有序的弊端**：
- 单 Queue 吞吐量极低
- 无法水平扩展 Consumer
- 不推荐生产环境使用

# 7. 顺序消息最佳实践

1. **使用分区有序**：99% 场景只需分区有序
2. **路由键选择**：订单ID、用户ID 等天然唯一标识
3. **消费者单线程**：`consumeThreadMax=1` 保证顺序
4. **避免耗时操作**：单线程 + 耗时操作 = 消费延迟放大
5. **异常处理**：返回 `SUSPEND_CURRENT_QUEUE_A_MOMENT` 而非吞掉异常
6. **MessageQueue 数量合理**：不少于 Consumer 实例数，避免部分 Consumer 空闲

# 8. 参考

- [RocketMQ 顺序消息](https://rocketmq.apache.org/docs/4.x/producer/03message1/)
