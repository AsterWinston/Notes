
> 源码版本：RocketMQ 4.9.x
> 核心模块：`broker/` 和 `client/`

# 1. 消息过滤整体架构

```
过滤流程:
Consumer 订阅 (Topic + 表达式)
    │
    ├── Broker 端过滤 (ConsumeQueue Tag Hash)
    │    └── 过滤不匹配的消息 → 不读取 CommitLog → 不传输
    │
    ├── Broker 端 SQL92 属性过滤
    │    └── 按消息属性做精确过滤
    │
    └── Consumer 端二次过滤
         └── 防止 Tag Hash 冲突的兜底过滤
```

# 2. Broker 端 Tag 过滤源码

## 2.1 ExpressionMessageFilter

```java
// broker/src/main/java/.../filter/ExpressionMessageFilter.java
public class ExpressionMessageFilter {

    // 基于 ConsumeQueue 的过滤 (高效、只读 20 字节)
    public boolean isMatchedByConsumeQueue(Long tagsCode, 
            ConsumeQueueExt.CqExtUnit cqExtUnit) {
        if (null == subscriptionData) {
            return true;
        }
        if (subscriptionData.isClassFilterMode()) {
            return true;  // 类过滤模式：不在 Broker 过滤，全部返回
        }
        // 空订阅 → 全部通过
        if (null == tagsCode || subscriptionData.getTagsSet().isEmpty()) {
            return true;
        }
        // Tag 匹配 (基于 HashCode)
        return subscriptionData.getTagsSet().contains(tagsCode.toString());
    }

    // 基于 CommitLog 的过滤 (精确、读完整消息)
    public boolean isMatchedByCommitLog(ByteBuffer msgBuffer, 
            Map<String, String> properties) {
        if (subscriptionData == null) {
            return true;
        }
        // Tag 精确匹配
        String tags = properties.get(MessageConst.PROPERTY_TAGS);
        if (tags != null && !subscriptionData.getTagsSet().isEmpty()) {
            if (subscriptionData.getTagsSet().contains(tags)) {
                return true;
            }
        }
        // SQL92 过滤
        if (subscriptionData.isSqlType()) {
            // 转交 ConsumerFilterManager 处理
            return false;
        }
        return false;
    }
}
```

## 2.2 过滤在 PullMessageProcessor 中的应用

```java
// PullMessageProcessor 中拉取消息的过程

// 1. 获取 ConsumeQueue 数据
// DefaultMessageStore.getMessage()
GetMessageResult getMessage(final String group, final String topic,
        final int queueId, final long offset, final int maxMsgNums,
        final MessageFilter messageFilter) {

    // 1. 获取 ConsumeQueue
    ConsumeQueue consumeQueue = findConsumeQueue(topic, queueId);
    long currentOffset = offset;
    long maxOffset = consumeQueue.getMaxOffsetInQueue();

    // 2. 遍历 ConsumeQueue 条目
    for (; currentOffset < maxOffset && getMessageResult.getBufferTotalSize() == 0; 
            currentOffset++) {
        // 读取 ConsumeQueue 条目 (20B)
        SelectMappedBufferResult bufferConsumeQueue = 
            consumeQueue.getIndexBuffer(currentOffset);
        long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();   // CommitLog offset
        int sizePy = bufferConsumeQueue.getByteBuffer().getInt();      // msg size
        long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();  // tags code

        // 2. Broker 端过滤
        if (messageFilter != null 
                && !messageFilter.isMatchedByConsumeQueue(tagsCode, null)) {
            // 不匹配 → 跳过，不读 CommitLog
            continue;
        }

        // 3. 读取 CommitLog 中的数据
        SelectMappedBufferResult selectResult = 
            commitLog.getMessage(offsetPy, sizePy);
        if (selectResult != null) {
            // 4. CommitLog 级过滤 (防 Tag Hash 冲突，或 SQL92 过滤)
            if (messageFilter != null 
                    && !messageFilter.isMatchedByCommitLog(
                        selectResult.getByteBuffer().slice(), null)) {
                selectResult.release();
                continue;
            }
            // 5. 加入到结果
            getMessageResult.addMessage(selectResult);
        }
    }

    return getMessageResult;
}
```

## 2.3 过滤前后对比

```
无过滤 (全量消费):
  Consumer 收到 10000 条消息
  Consumer 端自行过滤掉 9500 条
  ↓ 浪费 95% 带宽和处理能力

有 Tag 过滤:
  Broker 基于 ConsumeQueue(20B) 过滤掉 9500 条
  ↓ 只传输 500 条

有 SQL92 过滤:
  Broker 基于属性精确过滤
  ↓ 返回最精准的结果集
```

# 3. ConsumerFilterManager (SQL92 过滤管理)

```java
// broker/src/main/java/.../filter/ConsumerFilterManager.java
public class ConsumerFilterManager {

    // 注册/更新过滤数据
    public boolean registerFilter(String topic, String consumerGroup,
            String expression, String expressionType, long clientVersion) {
        ConsumerFilterData filterData = new ConsumerFilterData();
        filterData.setTopic(topic);
        filterData.setConsumerGroup(consumerGroup);
        filterData.setExpression(expression);       // SQL92 表达式
        filterData.setExpressionType(expressionType);// "TAG" or "SQL92"
        filterData.setClientVersion(clientVersion);

        // 如果表达式类型是 SQL92 → 编译表达式
        if (ExpressionType.SQL92.equals(expressionType)) {
            // 编译 → SqlFilter 对象
            filterData.setCompiledExpression(compile(expression));
        }

        // 持久化到文件
        // ...
    }

    // 消息匹配
    public boolean match(String topic, String consumerGroup,
            Map<String, String> properties) {
        ConsumerFilterData filterData = get(topic, consumerGroup);
        if (filterData == null) {
            return true;
        }

        if (ExpressionType.TAG.equals(filterData.getExpressionType())) {
            // Tag 过滤
            String tags = properties.get(MessageConst.PROPERTY_TAGS);
            return filterData.getExpression().contains(tags);
        } else if (ExpressionType.SQL92.equals(filterData.getExpressionType())) {
            // SQL92 过滤 — 基于表达式树计算
            return filterData.getCompiledExpression().evaluate(properties);
        }
        return true;
    }

    // 持久化 Consumer 过滤数据到文件
    public boolean persist() {
        // 写入：store/config/consumerFilter.json
        String content = JSON.toJSONString(this.filterDataByTopic);
        MixAll.string2File(content, filePath);
    }
}
```

# 4. 消息重试 (Consumer 端)

## 4.1 Consumer 发回重试消息

```java
// Consumer 端：消费失败时，发送回 Broker 进行重试
// ConsumeMessageConcurrentlyService.processConsumeResult()

public void processConsumeResult(
        final ConsumeConcurrentlyStatus status,
        final ConsumeConcurrentlyContext context,
        final ConsumeRequest consumeRequest) {

    switch (status) {
        case CONSUME_SUCCESS:
            // 消费成功 → 从 ProcessQueue 移除
            break;
        case RECONSUME_LATER:
            // 消费失败 → 发回 Broker 重试
            for (MessageExt msg : msgs) {
                // 获取重试 Topic: %RETRY%consumerGroup
                String retryTopic = MixAll.getRetryTopic(msg.getGroup());
                // 发送 MQClientAPIImpl.consumerSendMessageBack()
                this.defaultMQPushConsumerImpl.sendMessageBack(msg, delayLevel);
            }
            break;
    }
}

// DefaultMQPushConsumerImpl.sendMessageBack()
public void sendMessageBack(MessageExt msg, int delayLevel) {
    // 调用 MQClientAPIImpl.consumerSendMessageBack()
    this.mQClientFactory.getMQClientAPIImpl().consumerSendMessageBack(
        msg.getBornHost(), msg.getBrokerName(), msg, 
        this.defaultMQPushConsumer.getConsumerGroup(), delayLevel, 
        5000, maxConsumeRetryTimes);
}
```

## 4.2 Broker 接收重试消息

```java
// SendMessageProcessor.consumerSendMsgBack()

private RemotingCommand consumerSendMsgBack(ChannelHandlerContext ctx, 
        RemotingCommand request) {
    
    // 1. 获取消息的偏移量等
    long offset = requestHeader.getOffset();
    String group = requestHeader.getConsumerGroup();
    int delayLevel = requestHeader.getDelayLevel();

    // 2. 获取原始消息
    MessageExt msgExt = this.brokerController.getMessageStore()
        .lookMessageByOffset(offset);

    // 3. 获取重试 Topic: %RETRY% + consumerGroup
    String retryTopic = MixAll.getRetryTopic(group);

    // 4. 构建重试消息
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(retryTopic);
    msgInner.setBody(msgExt.getBody());
    msgInner.setFlag(msgExt.getFlag());
    msgInner.setProperties(msgExt.getProperties());

    // 5. 设置重试次数 +1
    int reconsumeTimes = requestHeader.getReconsumeTimes() + 1;
    msgInner.setReconsumeTimes(reconsumeTimes);

    // 6. 设置延迟级别
    msgInner.setDelayTimeLevel(delayLevel);

    // 7. 检查是否超过最大重试次数
    if (reconsumeTimes > maxConsumeRetryTimes) {
        // 改用死信 Topic
        String newTopic = MixAll.getDLQTopic(group);
        msgInner.setTopic(newTopic);
    }

    // 8. 存储重试消息
    PutMessageResult putMessageResult = 
        this.brokerController.getMessageStore().putMessage(msgInner);

    // 9. 返回结果
    return response;
}
```

# 5. 死信队列 (DLQ) 处理

```java
// 死信 Topic 的命名规则
// %DLQ% + consumerGroup

// 死信产生时机 (Broker 端 consumerSendMsgBack)
if (reconsumeTimes > maxConsumeRetryTimes || delayLevel < 0) {
    newTopic = MixAll.getDLQTopic(group);
}

// 死信消息的特征
MessageExt msg;
msg.getTopic();             // %DLQ%order_consumer_group
msg.getReconsumeTimes();    // 16 (或更多)
msg.getProperty(MessageConst.PROPERTY_ORIGIN_MESSAGE_ID);  // 原始消息ID
```

# 6. 消费进度与重试的关系

```java
// Consumer 端的进度提交

// 正常情况下：
// Consumer 拉取 10 条消息，消费 10 条
// 所有成功 → 提交 offset 到 Broker

// 如果其中 3 条 RECONSUME_LATER：
// 1. 3 条被发回 Broker（进入重试 Topic）
// 2. Consumer 提交 offset (跳过了这 3 条)
// 3. 后续重试 Topic 中的消息被消费 → 单独追踪
// 4. 原始 Queue 中的 offset 已提交 → 不影响正常消费
```

# 7. 重试消息的存储路线

```
消费失败 RECONSUME_LATER
    │
    ▼
Consumer.sendMessageBack(msg, delayLevel)
    │
    ▼
Broker.consumerSendMsgBack()
    │
    ├── Topic: %RETRY%consumerGroup
    ├── reconsumeTimes++
    │
    ▼
CommitLog 写入 (延迟消息处理)
    │
    ├── 设置 delayLevel
    ├── 替换为 SCHEDULE_TOPIC
    │
    ▼
ScheduledMessageService 定时扫描
    │
    ▼
到期 → 恢复原始 %RETRY% 消息 → Consumer 重新消费
    │
    ├── 重试 16 次成功 → 正常完成
    └── 重试 16 次仍失败 → Topic 转为 %DLQ%consumerGroup
```

# 8. 参考

- [ExpressionMessageFilter 源码](https://github.com/apache/rocketmq/blob/master/broker/src/main/java/org/apache/rocketmq/broker/filter/ExpressionMessageFilter.java)
- [SendMessageProcessor 源码](https://github.com/apache/rocketmq/blob/master/broker/src/main/java/org/apache/rocketmq/broker/processor/SendMessageProcessor.java)
