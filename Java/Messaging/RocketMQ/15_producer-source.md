
> 源码版本：RocketMQ 4.9.x
> 核心模块：`client/src/main/java/org/apache/rocketmq/client/`

# 1. Producer 启动流程源码解析

## 1.1 入口：`DefaultMQProducer.start()`

```java
// client/src/main/java/.../DefaultMQProducer.java
public class DefaultMQProducer extends ClientConfig implements MQProducer {
    
    @Override
    public void start() throws MQClientException {
        // 1. 设置 Producer Group 为生产者组名
        this.setProducerGroup(this.defaultMQProducerImpl.getDefaultMQProducer().getProducerGroup());
        // 2. 委派给 DefaultMQProducerImpl 执行
        this.defaultMQProducerImpl.start();
    }
}
```

## 1.2 核心启动：`DefaultMQProducerImpl.start()`

```java
// DefaultMQProducerImpl.start()
public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            this.serviceState = ServiceState.START_FAILED;

            // 1. 校验 Producer Group 名称
            this.checkConfig();

            // 2. 修改 instanceName 为进程 ID (防止单 JVM 多实例冲突)
            if (!this.defaultMQProducer.getProducerGroup()
                  .equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }

            // 3. 获取 MQClientInstance (JVM 级别单例)
            this.mQClientFactory = MQClientManager.getInstance()
                .getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);

            // 4. 注册 Producer 到 MQClientInstance
            boolean registerOK = mQClientFactory.registerProducer(
                this.defaultMQProducer.getProducerGroup(), this);

            // 5. 启动 MQClientInstance (如果未启动)
            if (startFactory) {
                mQClientFactory.start();
            }

            this.serviceState = ServiceState.RUNNING;
            break;
        // ... 其他状态处理
    }
}
```

## 1.3 MQClientInstance：JVM 级 Client 工厂

```java
// MQClientInstance.start() — 核心启动逻辑
public void start() throws MQClientException {
    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;

                // 1. 根据 namesrvAddr 获取 NameServer 地址列表
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }

                // 2. 启动 Netty Client
                this.mQClientAPIImpl.start();  // Netty Bootstrap 连接

                // 3. 启动各种定时任务
                this.startScheduledTask();

                // 4. 启动拉取消息服务 (Consumer 相关)
                this.pullMessageService.start();

                // 5. 启动 Rebalance 服务 (Consumer 相关)
                this.rebalanceService.start();

                // 6. 启动默认 Producer (用于发送内部消息)
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);

                this.serviceState = ServiceState.RUNNING;
                break;
        }
    }
}
```

## 1.4 定时任务详情

```java
private void startScheduledTask() {
    // 1. 每 2 分钟：从 NameServer 拉取 NameServer 地址列表
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
    }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);

    // 2. 每 30 秒：更新 Topic 路由信息
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        MQClientInstance.this.updateTopicRouteInfoFromNameServer();
    }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);

    // 3. 每 30 秒：向所有 Broker 发送心跳
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        MQClientInstance.this.cleanOfflineBroker();
        MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
    }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);

    // 4. 每 5 秒：持久化消费进度 (Consumer 相关)
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        MQClientInstance.this.persistAllConsumerOffset();
    }, 1000 * 10, 5000, TimeUnit.MILLISECONDS);

    // 5. 每 1 分钟：调整消费线程池大小
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        MQClientInstance.this.adjustThreadPool();
    }, 1, 1, TimeUnit.MINUTES);
}
```

# 2. 消息发送链路源码解析

## 2.1 入口：`DefaultMQProducerImpl.sendDefaultImpl()`

```java
// 同步发送入口
SendResult sendDefaultImpl(Message msg, CommunicationMode communicationMode,
        SendCallback sendCallback, long timeout) {

    // 1. 验证消息合法性
    Validators.checkMessage(msg, this.defaultMQProducer);

    // 2. 获取 Topic 路由信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());

    // 3. 同步发送循环（含重试）
    int timesTotal = 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed();
    for (; times < timesTotal; times++) {
        // 4. 选择 MessageQueue
        MessageQueue mq = topicPublishInfo.selectOneMessageQueue(lastBrokerName);

        // 5. 发送核心逻辑
        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, 
            topicPublishInfo, timeout);

        // 6. 处理发送结果
        switch (communicationMode) {
            case ASYNC:
                return null;
            case ONEWAY:
                return null;
            case SYNC:
                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                    // 重试逻辑...
                }
        }
    }
}
```

## 2.2 路由查找：`tryToFindTopicPublishInfo()`

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 1. 从本地缓存获取
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);

    // 2. 如果缓存为空或路由无效 → 向 NameServer 拉取
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    // 3. 如果仍然没有 → 尝试使用 DefaultTopic 路由
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(
            topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

## 2.3 核心发送：`sendKernelImpl()`

```java
private SendResult sendKernelImpl(Message msg, MessageQueue mq,
        CommunicationMode communicationMode, SendCallback sendCallback,
        TopicPublishInfo topicPublishInfo, long timeout) {

    // 1. 根据 MessageQueue 获取 Broker 地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(
        mq.getBrokerName());
    if (null == brokerAddr) {
        // 从 NameServer 获取
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }

    // 2. 消息体压缩（大于 compressMsgBodyOverHowmuch = 4KB 时）
    if (this.tryToCompressMessage(msg)) {
        // 设置压缩标记
        msg.setSysFlag(msgFlag | MessageSysFlag.COMPRESSED_FLAG);
    }

    // 3. 检查是否有事务属性 (TRANSACTION_PREPARED_TYPE)
    // ...

    // 4. 构建发送请求头
    SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
    requestHeader.setProducerGroup(...);
    requestHeader.setTopic(msg.getTopic());
    requestHeader.setQueueId(mq.getQueueId());
    // ... 其他字段

    // 5. 根据通信模式发送
    switch (communicationMode) {
        case ASYNC:
            RemotingCommand request = buildRequest(requestHeader, msg);
            this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                brokerAddr, brokerName, request, timeout, 
                this.defaultMQProducer.getSendMsgTimeout(), 
                communicationMode, sendCallback, ...);
            break;
        case ONEWAY:
            // ...
        case SYNC:
            RemotingCommand response = this.mQClientFactory.getMQClientAPIImpl()
                .sendMessage(brokerAddr, brokerName, msg, requestHeader, 
                    timeout, communicationMode, ...);
            return this.sendMessageResponse(brokerAddr, topic, msg, response);
    }
}
```

## 2.4 消息压缩：`tryToCompressMessage()`

```java
private boolean tryToCompressMessage(final Message msg) {
    if (msg instanceof MessageBatch) {
        // 批量消息暂不压缩
        return false;
    }
    byte[] body = msg.getBody();
    if (body != null) {
        if (body.length >= this.defaultMQProducer.getCompressMsgBodyOverHowmuch()) {
            byte[] compressedBody = UtilAll.compress(body);
            if (compressedBody != null && compressedBody.length < body.length) {
                msg.setBody(compressedBody);
                return true;
            }
        }
    }
    return false;
}
```

# 3. 消息发送完整调用链路

```
DefaultMQProducer.send(msg)
    │
    ▼
DefaultMQProducerImpl.send(msg, timeout)
    │
    ▼
DefaultMQProducerImpl.sendDefaultImpl(msg, communicationMode, callback, timeout)
    │
    ├── Validators.checkMessage(msg)  — 消息校验
    ├── tryToFindTopicPublishInfo(topic)  — 获取路由
    │    ├── topicPublishInfoTable.get(topic)  — 本地缓存
    │    └── updateTopicRouteInfoFromNameServer(topic)  — NameServer 拉取
    │
    ├── selectOneMessageQueue(lastBrokerName)  — 选择队列
    │    └── 轮询 + 故障延迟机制
    │
    └── sendKernelImpl(msg, mq, ...)  — 核心发送
         ├── findBrokerAddressInPublish(brokerName)  — 查 Broker 地址
         ├── compressMessage(msg)  — 消息压缩
         ├── 构建 RemotingCommand (请求报文)
         ├── MQClientAPIImpl.sendMessage()  — Netty 网络发送
         │    ├── invokeSync() / invokeAsync() / invokeOneway()
         │    └── NettyRemotingClient.invokeSyncImpl()
         │         ├── 序列化 RemotingCommand
         │         ├── Channel.writeAndFlush()
         │         └── ResponseFuture 等待响应
         └── processSendResponse()  — 解析响应
```

# 4. 关键设计模式与要点

## 4.1 MQClientInstance 单例

```java
// 同一 JVM 的同一 ClientId 共享一个 MQClientInstance
// 工厂ID = IP@InstanceName@UnitName
// 避免重复创建网络连接、线程池等昂贵资源
public class MQClientManager {
    private static MQClientManager instance = new MQClientManager();
    private ConcurrentMap<String, MQClientInstance> factoryTable = ...;
}
```

## 4.2 消息唯一 ID 生成

```java
// MessageClientIDSetter.createUniqID()
// 格式：IP(hex) + PID(hex) + ClassLoaderID(hex) + seq(number)
// 例如：0A6F2B08070D33B2421800000000
// 长度 16 字节 = 128 位

// 进程内自增序列号保证唯一性
private static final int[] SEQUENCE_ONCE_ARR = new int[1024];
```

## 4.3 故障延迟容错

```java
// LatencyFaultTolerance — 核心数据结构
// faultItemTable: {brokerName → FaultItem}
FaultItem {
    String name;           // Broker 名称
    volatile long currentLatency;       // 当前延迟
    volatile long startTimestamp;       // 故障开始时间
}

// 故障周期：
// 如果是 550ms 延迟 → 30000ms 不允许发送
// 如果是 1000ms 延迟 → 60000ms 不允许发送
// ...
// 解决：避免反复向不可用 Broker 发送请求（雪崩）
```

# 5. 参考

- [DefaultMQProducerImpl 源码](https://github.com/apache/rocketmq/blob/master/client/src/main/java/org/apache/rocketmq/client/impl/producer/DefaultMQProducerImpl.java)
- [MQClientInstance 源码](https://github.com/apache/rocketmq/blob/master/client/src/main/java/org/apache/rocketmq/client/impl/factory/MQClientInstance.java)
