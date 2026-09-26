
# 1. Consumer 概述

Consumer 负责从 Broker 拉取消息并进行业务消费。RocketMQ 提供两种消费模式：**Push**（被动接收）和 **Pull**（主动拉取）。

## 1.1 消费模式对比

| 维度 | Push 消费 (DefaultMQPushConsumer) | Pull 消费 (DefaultLitePullConsumer) |
|------|----------------------------------|-------------------------------------|
| 控制方式 | Broker 推送，Client 被动接收 | Client 主动拉取 |
| 实现原理 | 内部封装 Pull，长轮询模拟 Push | 应用自主调用 pull() |
| 易用性 | 高，只需注册 MessageListener | 低，需自己控制拉取和提交 |
| 流量控制 | 框架层控制 | 应用层完全自主控制 |
| 适用场景 | 大部分业务场景 | 严格流量控制、批量处理场景 |

## 1.2 消费消息模式

**集群消费（Clustering） — 默认
**：

```
Producer ─→ Topic (8个MessageQueue)
              ├── Q0 ─→ Consumer 1 (Group: myGroup)
              ├── Q1 ─→ Consumer 1
              ├── Q2 ─→ Consumer 2
              ├── Q3 ─→ Consumer 2
              ├── Q4 ─→ Consumer 3
              ├── Q5 ─→ Consumer 3
              ├── Q6 ─→ Consumer 4
              └── Q7 ─→ Consumer 4
每条消息只被组内一个实例消费，水平扩展能力随实例增加
```

**广播消费（Broadcasting）
**：

```
Producer ─→ Topic (8个MessageQueue)
              ├── Q0 ─→ Consumer 1, Consumer 2, Consumer 3
              ├── Q1 ─→ Consumer 1, Consumer 2, Consumer 3
              ...
              └── Q7 ─→ Consumer 1, Consumer 2, Consumer 3
每条消息被组内所有实例消费，适合配置刷新等场景
```

# 2. Push Consumer 深入

## 2.1 基本使用

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");
consumer.setNamesrvAddr("127.0.0.1:9876");
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
consumer.subscribe("TopicTest", "TagA || TagB");
consumer.setMessageModel(MessageModel.CLUSTERING); // 集群模式（默认）

consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        for (MessageExt msg : msgs) {
            process(msg);
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});

consumer.start();
```

## 2.2 消费起始位置

```java
public enum ConsumeFromWhere {
    CONSUME_FROM_LAST_OFFSET,       // 从上次消费位置继续
    @Deprecated
    CONSUME_FROM_LAST_OFFSET_AND_FROM_MIN_WHEN_BOOT_FIRST,  // 已废弃
    @Deprecated
    CONSUME_FROM_MIN_OFFSET,        // 已废弃
    @Deprecated
    CONSUME_FROM_MAX_OFFSET,        // 已废弃
    CONSUME_FROM_FIRST_OFFSET,      // 从队列最前面开始
    CONSUME_FROM_TIMESTAMP,         // 从指定时间点开始
}
```

**注意**：`CONSUME_FROM_LAST_OFFSET` 仅在 Consumer Group 首次消费时生效。已有消费进度的 Group 会从 OffsetStore 记录的位点继续。

## 2.3 并发消费 vs 顺序消费

```java
// 并发消费（无序，吞吐高）
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        // 可多线程并发处理
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});

// 顺序消费（有序，吞吐低）
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeOrderlyContext context) {
        // 同一队列消息串行处理
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

## 2.4 消费进度提交

| OffsetStore 类型 | 存储位置 | 适用模式 |
|-------------------|----------|----------|
| `RemoteBrokerOffsetStore` | Broker 内存 + 文件 | 集群消费 |
| `LocalFileOffsetStore` | Consumer 本地文件 | 广播消费 |

```java
// 集群消费偏移量存储文件（默认路径）
// ~/.rocketmq_offsets/{clientId}/{consumerGroup}/offsets.json
```

# 3. Pull Consumer (DefaultLitePullConsumer)

## 3.1 基本使用

```java
DefaultLitePullConsumer consumer = new DefaultLitePullConsumer("pull_group");
consumer.setNamesrvAddr("127.0.0.1:9876");
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
consumer.subscribe("TopicTest", "*");
consumer.setAutoCommit(false);  // 手动提交偏移量
consumer.start();

while (true) {
    List<MessageExt> msgs = consumer.poll(1000);
    for (MessageExt msg : msgs) {
        System.out.println(new String(msg.getBody()));
    }
    consumer.commitSync();  // 手动提交偏移量
}
```

## 3.2 核心配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `pollTimeoutMillis` | 5000ms | 拉取超时时间 |
| `pullBatchSize` | 10 | 每次拉取的最大消息数 |
| `pullThreadNums` | 20 | 拉取线程数 |
| `autoCommit` | true | 是否自动提交偏移量 |

# 4. 消费负载均衡 (Rebalance)

## 4.1 触发时机

1. Consumer 实例增减（上线/下线）
2. Topic 的 MessageQueue 数量变化
3. 定时触发（默认 20s）

## 4.2 Rebalance 算法

```
1. 获取 Consumer Group 下所有 Consumer 实例 (cidAll)
2. 获取 Topic 下所有 MessageQueue (mqAll)
3. 排序 cidAll 和 mqAll
4. 平均分配 mqAll 给每个 Consumer
   分配结果: cid_1 → [Q0, Q1, Q2], cid_2 → [Q3, Q4, Q5], cid_3 → [Q6, Q7]
```

**Requeue 的危害与规避**：
- Rebalance 期间暂停消费，可能造成短暂消息堆积
- 建议稳定环境，避免频繁 Consumer 增减
- 可使用粘性分配策略减少不必要的迁移

# 5. 消费幂等保证

```java
// 方案 1：使用消息 Key 去重
String msgKey = msgExt.getMsgId() + "_" + msgExt.getTopic();
if (redis.setIfAbsent(msgKey, "1", 72, TimeUnit.HOURS)) {
    process(msgExt);  // 业务处理
}

// 方案 2：数据库唯一索引
// INSERT INTO processed_msgs (msg_id) VALUES (#{msgId}) ON DUPLICATE KEY UPDATE...

// 方案 3：业务状态机
// 状态从 PENDING → PROCESSING → DONE，只有 PENDING 状态才处理
```

# 6. 流量控制

## 6.1 Consumer 端控制

```java
// 并发消费 + 线程池控制
consumer.setConsumeThreadMin(20);    // 最小消费线程数
consumer.setConsumeThreadMax(64);    // 最大消费线程数
consumer.setConsumeMessageBatchMaxSize(32); // 每批最大消息数
consumer.setPullBatchSize(32);       // 推送时每批消息数

// 消费速度控制（顺序消费专用）
consumer.setMaxReconsumeTimes(16);   // 最大重试次数
```

## 6.2 端到端流量控制

```
速率控制器 = 滑动窗口 + 令牌桶

Producer 端：限制发送速率
Broker 端：限制接收/投递速率
Consumer 端：限制拉取/消费速率
```

# 7. 关键配置参数速查

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `consumeThreadMin` | 20 | 最小消费线程数 |
| `consumeThreadMax` | 64 | 最大消费线程数 |
| `consumeMessageBatchMaxSize` | 1 | 每次回调最大消息数（并发模式可设>1） |
| `pullBatchSize` | 32 | 每次拉取消息数 |
| `pullInterval` | 0ms | 拉取间隔 |
| `maxReconsumeTimes` | 16 | 消费失败最大重试次数 |
| `consumeTimeout` | 15min | 单条消费超时（顺序模式） |
| `messageModel` | CLUSTERING | 消费模式 |
| `consumeFromWhere` | CONSUME_FROM_LAST_OFFSET | 起始消费位置 |
| `allocateMessageQueueStrategy` | 平均分配 | 分配策略 |
| `rebalanceInterval` | 20s | 重平衡间隔 |

# 8. 最佳实践

1. **消费幂等**：始终做好幂等处理，RocketMQ 不保证精确一次
2. **异常捕获**：消费逻辑内部 try-catch，不要依赖框架重试处理所有异常
3. **不要阻塞**：并发消费模式下不要长时间阻塞消费线程
4. **谨慎使用广播**：广播消费不支持消费进度服务端存储
5. **合理配比线程数**：消费线程数 ≈ (MQ数 / Consumer数) × 并发系数
6. **关注消费延迟**：监控 `DIFF`（未被消费的消息数）

# 9. 参考

- [Consumer — RocketMQ 官方文档](https://rocketmq.apache.org/docs/4.x/consumer/01intro/)
