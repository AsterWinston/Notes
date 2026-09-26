
> 源码版本：RocketMQ 4.9.x
> 核心模块：`client/src/main/java/org/apache/rocketmq/client/`

# 1. DefaultMQPushConsumer 启动流程

## 1.1 类层次结构

```
MQPushConsumer (接口)
    └── DefaultMQPushConsumer (用户 API)
         └── DefaultMQPushConsumerImpl (核心实现)
              └── 内部封装：PullAPIWrapper
```

## 1.2 启动流程源码

```java
// DefaultMQPushConsumerImpl.start()
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            this.serviceState = ServiceState.START_FAILED;

            // 1. 校验配置 (ConsumerGroup 名称、MessageModel 等)
            this.checkConfig();

            // 2. 拷贝订阅关系 (subscribe 时设置的 Topic/Tag)
            this.copySubscription();

            // 3. 获取 MQClientInstance (JVM 单例)
            this.mQClientFactory = MQClientManager.getInstance()
                .getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

            // 4. 设置重平衡策略
            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            // 默认：AllocateMessageQueueAveragely (平均分配)
            // 可选：AllocateMessageQueueAveragelyByCircle (环状分配)
            //       AllocateMessageQueueByConfig (手动分配)
            //       AllocateMessageQueueByMachineRoom (机房就近)
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());

            // 5. 设置消费策略
            this.pullAPIWrapper = new PullAPIWrapper(
                mQClientFactory,
                this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

            // 6. 初始化 OffsetStore (进度存储)
            switch (this.defaultMQPushConsumer.getMessageModel()) {
                case BROADCASTING:
                    this.offsetStore = new LocalFileOffsetStore(
                        this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                    break;
                case CLUSTERING:
                    this.offsetStore = new RemoteBrokerOffsetStore(
                        this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                    break;
            }
            this.offsetStore.load();

            // 7. 根据 MessageListener 类型，创建消费服务
            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                this.consumeOrderly = true;
                this.consumeMessageService = new ConsumeMessageOrderlyService(
                    this, (MessageListenerOrderly) this.getMessageListenerInner());
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                this.consumeOrderly = false;
                this.consumeMessageService = new ConsumeMessageConcurrentlyService(
                    this, (MessageListenerConcurrently) this.getMessageListenerInner());
            }

            // 8. 注册 Consumer 到 MQClientInstance
            boolean registerOK = mQClientFactory.registerConsumer(
                this.defaultMQPushConsumer.getConsumerGroup(), this);

            // 9. 启动 MQClientInstance
            mQClientFactory.start();

            // 10. 向所有 Broker 发送心跳，通知 Consumer 上线
            this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

            // 11. 立即执行一次 Rebalance
            this.mQClientFactory.rebalanceImmediately();

            this.serviceState = ServiceState.RUNNING;
            break;
    }
}
```

# 2. PullMessageService — Push 的本质

## 2.1 "Push" 的真面目

```
*** "Push 消费" 本质上也是 Pull ***

DefaultMQPushConsumer 内部：
  PullMessageService ──→ 不断从 Broker 拉取消息
                          拉取到消息后 → ConsumeMessageService 消费
                          无消息时 → PullRequestHoldService Long Polling

流程：
1. Rebalance 服务分配好 Queue 归属
2. 每个 Queue 生成一个 PullRequest
3. PullMessageService 从 pullRequestQueue 中取出 PullRequest
4. 调用 DefaultMQPushConsumerImpl.pullMessage() 拉取
5. 拉取成功 → 提交给 ConsumeMessageService 处理
6. 处理后再次创建 PullRequest → 放入 pullRequestQueue
```

## 2.2 PullMessageService 源码

```java
public class PullMessageService extends ServiceThread {
    private final LinkedBlockingQueue<PullRequest> pullRequestQueue = 
        new LinkedBlockingQueue<>();
    private final MQClientInstance mQClientFactory;

    @Override
    public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            // 从阻塞队列取出 PullRequest
            PullRequest pullRequest = this.pullRequestQueue.take();
            // 执行拉取
            this.pullMessage(pullRequest);
        }

        log.info(this.getServiceName() + " service end");
    }

    private void pullMessage(final PullRequest pullRequest) {
        final MQConsumerInner consumer = 
            this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
        if (consumer != null) {
            DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
            impl.pullMessage(pullRequest);
        }
    }
}
```

## 2.3 DefaultMQPushConsumerImpl.pullMessage()

```java
public void pullMessage(final PullRequest pullRequest) {
    // 1. 获取 ProcessQueue (当前 Queue 的处理队列)
    final ProcessQueue processQueue = pullRequest.getProcessQueue();

    // 2. 检查 ProcessQueue 是否被丢弃 (Rebalance 导致)
    if (processQueue.isDropped()) {
        return;
    }

    // 3. 检查是否锁定 (顺序消费)
    if (consumeOrderly && !processQueue.isLocked()) {
        // 延迟重试
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
        return;
    }

    // 4. 获取消费偏移量
    long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());

    // 5. 发起 Pull 请求 → Broker
    try {
        PullResult pullResult = this.pullAPIWrapper.pullKernelImpl(
            pullRequest.getMessageQueue(),  // MessageQueue
            subExpression,                  // Tag 过滤表达式
            subscriptionData.getExpressionType(), // TAG / SQL92
            offset,                         // 偏移量
            this.defaultMQPushConsumer.getPullBatchSize(), // 拉取数量
            this.defaultMQPushConsumer.getPullInterval(),  // 拉取间隔
            ...);

        // 6. 处理 Pull 结果
        switch (pullResult.getPullStatus()) {
            case FOUND:
                // 拉取到消息
                // 将消息放入 ProcessQueue
                processQueue.putMessage(pullResult.getMsgFoundList());
                // 提交给 ConsumeMessageService 消费
                this.consumeMessageService.submitConsumeRequest(
                    pullResult.getMsgFoundList(), processQueue, pullRequest.getMessageQueue());
                break;
            case NO_NEW_MSG:
            case NO_MATCHED_MSG:
                // 没有消息 → 稍后再拉（进入 Long Polling）
                pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                this.executePullRequestImmediately(pullRequest);
                break;
            case OFFSET_ILLEGAL:
                // 偏移量非法 → 重新计算
                pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                this.offsetStore.updateOffset(pullRequest.getMessageQueue(), 
                    pullRequest.getNextOffset(), false);
                this.executePullRequestImmediately(pullRequest);
                break;
        }
    } catch (Exception e) {
        // 异常 → 延迟重试
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    }
}
```

# 3. ConsumeMessageConcurrentlyService 并发消费源码

```java
public class ConsumeMessageConcurrentlyService implements ConsumeMessageService {
    // 消费线程池
    private final ThreadPoolExecutor consumeExecutor;

    public ConsumeMessageConcurrentlyService(DefaultMQPushConsumerImpl defaultMQPushConsumerImpl,
            MessageListenerConcurrently messageListener) {
        this.defaultMQPushConsumerImpl = defaultMQPushConsumerImpl;
        this.messageListener = messageListener;

        this.consumeExecutor = new ThreadPoolExecutor(
            this.defaultMQPushConsumer.getConsumeThreadMin(),
            this.defaultMQPushConsumer.getConsumeThreadMax(),
            1000 * 60, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(),
            new ThreadFactoryImpl("ConsumeMessageThread_"));
    }

    @Override
    public void submitConsumeRequest(List<MessageExt> msgs, 
            ProcessQueue processQueue, MessageQueue messageQueue) {
        final int consumeBatchSize = 
            this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
        // 如果 consumeBatchSize <= 消息条数 → 分批次提交
        if (msgs.size() <= consumeBatchSize) {
            ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
            this.consumeExecutor.submit(consumeRequest);
        } else {
            for (int total = 0; total < msgs.size(); ) {
                List<MessageExt> msgThis = msgs.subList(total, 
                    Math.min(total + consumeBatchSize, msgs.size()));
                total += consumeBatchSize;
                ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
                this.consumeExecutor.submit(consumeRequest);
            }
        }
    }
}

// ConsumeRequest — 消费任务
class ConsumeRequest implements Runnable {
    @Override
    public void run() {
        // 1. 校验 ProcessQueue 是否被丢弃
        if (this.processQueue.isDropped()) {
            return;
        }

        // 2. 调用用户注册的 MessageListener
        ConsumeConcurrentlyStatus status = 
            messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);

        // 3. 处理消费结果
        if (!processQueue.isDropped()) {
            processConsumeResult(status, context, this);
        }
    }
}
```

# 4. 消费进度管理 (OffsetStore)

## 4.1 RemoteBrokerOffsetStore（集群模式）

```java
public class RemoteBrokerOffsetStore implements OffsetStore {
    @Override
    public void persist(MessageQueue mq) {
        long offset = this.offsetTable.get(mq);
        // 发送 UPDATE_CONSUMER_OFFSET 请求到 Broker
        UpdateConsumerOffsetRequestHeader requestHeader = 
            new UpdateConsumerOffsetRequestHeader();
        requestHeader.setTopic(mq.getTopic());
        requestHeader.setConsumerGroup(groupName);
        requestHeader.setQueueId(mq.getQueueId());
        requestHeader.setCommitOffset(offset);

        this.mQClientFactory.getMQClientAPIImpl().updateConsumerOffset(brokerAddr, 
            requestHeader, timeoutMillis);
    }
}
```

## 4.2 LocalFileOffsetStore（广播模式）

```java
public class LocalFileOffsetStore implements OffsetStore {
    // 本地文件存储路径: ~/.rocketmq_offsets/{clientId}/{group}/offsets.json
    private final String storePath;

    @Override
    public void persist(MessageQueue mq) {
        // 写入本地 JSON 文件
        String content = JSON.toJSONString(this.offsetTable, true);
        MixAll.string2File(content, storePath);
    }
}
```

# 5. ConsumeMessageOrderlyService 顺序消费源码

```java
public class ConsumeMessageOrderlyService implements ConsumeMessageService {
    // 核心：每个 MessageQueue 在同一时刻只由一个线程消费

    @Override
    public void submitConsumeRequest(List<MessageExt> msgs, 
            ProcessQueue processQueue, MessageQueue messageQueue) {
        // 如果当前 Queue 正在消费中 → 不处理
        if (!processQueue.isLocked()) {
            // 加锁
            processQueue.setLocked(true);
            // 提交消费请求
            ConsumeRequest consumeRequest = new ConsumeRequest(processQueue, messageQueue);
            this.consumeExecutor.submit(consumeRequest);
        }
    }

    // ConsumeRequest 逐条消费
    class ConsumeRequest implements Runnable {
        @Override
        public void run() {
            // 1. 从 ProcessQueue 取一条消息
            MessageExt msg = processQueue.takeMessage();

            // 2. 消费
            ConsumeOrderlyStatus status = messageListener.consumeMessage(
                Collections.singletonList(msg), context);

            // 3. 处理结果
            switch (status) {
                case SUCCESS:
                    // 消费成功，移出 ProcessQueue
                    processQueue.removeMessage(msg);
                    break;
                case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                    // 稍后重试
                    // 当前 Queue 暂停 suspendCurrentQueueTimeMillis
                    break;
            }
        }
    }
}
```

# 6. 完整调用链路

```
DefaultMQPushConsumer.start()
    │
    ├── DefaultMQPushConsumerImpl.start()
    │    ├── 创建 ConsumeMessageService (Concurrent/Orderly)
    │    ├── 创建 OffsetStore (Remote/Local)
    │    └── 启动 MQClientInstance
    │         ├── PullMessageService.start()
    │         └── RebalanceService.start()
    │
    ▼
RebalanceService → 分配 Queue → 创建 PullRequest
    │
    ▼
PullMessageService.pullMessage(PullRequest)
    │
    ▼
DefaultMQPushConsumerImpl.pullMessage()
    │
    ├── pullAPIWrapper.pullKernelImpl()
    │    └── Netty → Broker (PULL_MESSAGE)
    │
    ├── processQueue.putMessage() — 放入处理队列
    ├── consumeMessageService.submitConsumeRequest() — 提交消费
    │    └── ConsumeRequest.run()
    │         └── messageListener.consumeMessage() — 用户回调
    │
    └── processConsumeResult()
         ├── SUCCESS → 更新 Offset → 立即拉下一批
         └── RECONSUME_LATER → 发送回 Broker → 延迟重试
```

# 7. 参考

- [DefaultMQPushConsumerImpl 源码](https://github.com/apache/rocketmq/blob/master/client/src/main/java/org/apache/rocketmq/client/impl/consumer/DefaultMQPushConsumerImpl.java)
- [PullMessageService 源码](https://github.com/apache/rocketmq/blob/master/client/src/main/java/org/apache/rocketmq/client/impl/consumer/PullMessageService.java)
