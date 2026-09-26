
# 1. Producer 概述

Producer 是消息生产者，负责将业务消息发送到 Broker。RocketMQ 提供了高度可配置的生产者能力，支持多种发送模式和容灾机制。

## 1.1 类层次结构

```
MQProducer (接口)
  └── DefaultMQProducer (默认实现)
       ├── TransactionMQProducer (事务消息生产者)
       └── 自定义 Producer
```

## 1.2 Producer Group（生产组）

- 一组相同角色的 Producer 实例组成的逻辑分组
- **作用**：事务消息回查时，由同一 Group 内的任意 Producer 处理
- **命名建议**：`{业务名}_{系统名}`，如 `order_pay_system`

# 2. 消息发送方式

## 2.1 同步发送 (Sync Send)

```java
DefaultMQProducer producer = new DefaultMQProducer("sync_group");
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();

Message msg = new Message("TopicTest", "TagA", "Hello Sync".getBytes());
SendResult result = producer.send(msg);
System.out.println(result);
// SendResult [sendStatus=SEND_OK, msgId=..., messageQueue=..., queueOffset=...]
```

**流程**：
```
Producer 发送请求 → Netty I/O 线程 → Broker 处理 → 返回响应
                 └── 调用线程阻塞等待 ──────────────→ 收到响应唤醒
```

**SendResult 关键字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `sendStatus` | SendStatus | SEND_OK / FLUSH_DISK_TIMEOUT / FLUSH_SLAVE_TIMEOUT |
| `msgId` | String | 消息全局唯一 ID（Producer 生成） |
| `offsetMsgId` | String | Broker 生成的消息 ID |
| `messageQueue` | MessageQueue | 消息路由到的队列 |
| `queueOffset` | long | 消息在队列中的偏移量 |

## 2.2 异步发送 (Async Send)

```java
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult result) {
        System.out.println("发送成功: " + result);
    }

    @Override
    public void onException(Throwable e) {
        System.err.println("发送失败: " + e.getMessage());
    }
});
// 调用线程不阻塞，立即返回
Thread.sleep(3000); // 保证回调线程可以执行完
```

**注意要点**：
- 异步发送无返回值，通过回调感知结果
- 回调在 Netty I/O 线程中执行，不要做耗时操作
- 应用关闭前需等待所有异步回调完成

## 2.3 单向发送 (Oneway Send)

```java
producer.sendOneway(msg);
// 不关心发送结果，适用于日志采集等可丢失场景
```

**特点**：
- 不等待 Broker 响应，最低延迟
- 不保证消息一定到达，无重试
- 适合日志、监控数据等可靠性要求不高的场景

## 2.4 三种方式对比

| 方式 | 可靠性 | 延迟 | 适用场景 |
|------|--------|------|----------|
| 同步 | 高（知道结果） | 高（等待响应） | 重要业务消息 |
| 异步 | 高（回调感知） | 中 | 对延迟敏感的重要消息 |
| 单向 | 低（不确认） | 最低 | 日志、监控采集 |

# 3. 消息路由与队列选择

## 3.1 路由流程

```
发送消息
    │
    ├── 1. 从 NameServer 获取 Topic 路由信息（TopicRouteData）
    │      └── 缓存路由，定时更新（默认 30s）
    │
    ├── 2. 选择 MessageQueue
    │      ├── 不指定 → 默认轮询所有队列
    │      └── 指定 MessageQueueSelector → 按业务规则选择
    │
    ├── 3. 确定 Broker
    │      └── 从 MessageQueue 关联的 Broker 中选择
    │
    └── 4. 发送到 Broker
```

## 3.2 默认队列选择算法

```java
// TopicPublishInfo.selectOneMessageQueue()
public MessageQueue selectOneMessageQueue(String lastBrokerName) {
    // 故障延迟机制：上次发送失败的 Broker 延迟重试
    if (lastBrokerName != null) {
        int index = this.sendWhichQueue.incrementAndGet();
        // 轮询所有队列
        int pos = Math.abs(index) % this.messageQueueList.size();
        return this.messageQueueList.get(pos);
    }
    // 正常情况：轮询
    int index = this.sendWhichQueue.incrementAndGet();
    int pos = Math.abs(index) % this.messageQueueList.size();
    return this.messageQueueList.get(pos);
}
```

## 3.3 自定义队列选择（顺序消息的关键）

```java
// 按订单ID选择队列，保证同一个订单的消息进入同一队列（分区有序）
SendResult result = producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Long orderId = (Long) arg;
        int index = (int) (orderId % mqs.size());
        return mqs.get(index);
    }
}, orderId);
```

# 4. 发送重试机制

## 4.1 同步发送重试

```
第1次发送 → 失败
    ├── 等待 retryAnotherBrokerWhenNotStoreOK
    ├── 第2次重试 → 失败
    ├── 第3次重试 → 成功（默认最多重试2次，共3次发送）
    └── 仍失败 → 抛 MQClientException
```

```java
// 重试配置
producer.setRetryTimesWhenSendFailed(3);        // 同步发送重试次数，默认 2
producer.setRetryTimesWhenSendAsyncFailed(3);   // 异步发送重试次数，默认 2
producer.setRetryAnotherBrokerWhenNotStoreOK(true); // Broker 不可用时换其他 Broker
```

## 4.2 故障延迟机制 (LatencyFaultTolerance)

```java
// 默认关闭，开启后避免反复重试不可用 Broker
producer.setSendLatencyFaultEnable(true);
```

**原理**：
```
Broker 响应时间 > 阈值 → 标记为不可用 → 延迟重试（按延迟级别递增）
550ms → 30s 不可用
1000ms → 60s 不可用
2000ms → 120s 不可用
3000ms → 180s 不可用
...
```

# 5. 批量发送

```java
List<Message> messages = new ArrayList<>();
messages.add(new Message("TopicTest", "TagA", "Order-001".getBytes()));
messages.add(new Message("TopicTest", "TagA", "Order-002".getBytes()));
messages.add(new Message("TopicTest", "TagA", "Order-003".getBytes()));

SendResult result = producer.send(messages);
```

**限制条件**：
- 批量消息必须有相同的 Topic
- 批量消息不能有延迟消息属性
- 总大小不能超过 4MB（`maxMessageSize`）
- 建议单批次不超过 1MB

**超过大小限制的方案（拆分发送）**：

```java
public class ListSplitter {
    public List<List<Message>> split(List<Message> messages, int maxSize) {
        List<List<Message>> result = new ArrayList<>();
        List<Message> current = new ArrayList<>();
        int currentSize = 0;
        for (Message msg : messages) {
            int msgSize = msg.getBody().length + 20; // 20 为日志开销
            if (currentSize + msgSize > maxSize) {
                result.add(current);
                current = new ArrayList<>();
                currentSize = 0;
            }
            current.add(msg);
            currentSize += msgSize;
        }
        if (!current.isEmpty()) result.add(current);
        return result;
    }
}
```

# 6. 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `sendMsgTimeout` | 3000ms | 发送消息超时时间 |
| `compressMsgBodyOverHowmuch` | 4096 (4KB) | 超过此大小的消息体自动压缩 |
| `retryTimesWhenSendFailed` | 2 | 同步发送重试次数 |
| `maxMessageSize` | 4MB | 最大消息大小 |
| `retryAnotherBrokerWhenNotStoreOK` | false | 失败时尝试其他 Broker |
| `sendLatencyFaultEnable` | false | 开启故障延迟机制 |
| `pollNameServerInterval` | 30000ms | 从 NameServer 拉取路由间隔 |
| `heartbeatBrokerInterval` | 30000ms | 向 Broker 发送心跳间隔 |

# 7. 最佳实践

1. **单实例复用**：一个 Producer 实例可发送多个 Topic，避免反复创建
2. **合理使用异步**：非核心链路用异步发送，提升吞吐
3. **消息 Key 设计**：`msg.setKeys("orderId_123")` 方便后续按 Key 查询
4. **业务方记录 sendResult**：记录 `offsetMsgId` 用于问题追踪
5. **故障延迟开启**：高可靠场景建议 `setSendLatencyFaultEnable(true)`
6. **关闭时等待**：`producer.shutdown()` 等待异步回调完成

# 8. 参考

- [Producer — RocketMQ 官方文档](https://rocketmq.apache.org/docs/4.x/producer/01intro/)
