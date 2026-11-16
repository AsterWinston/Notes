
> 源码版本：RocketMQ 4.9.x
> 核心模块：`broker/src/main/java/org/apache/rocketmq/broker/`

# 1. 启动入口 `BrokerStartup`

```java
// broker/src/main/java/.../BrokerStartup.java
public class BrokerStartup {
    public static void main(String[] args) {
        start(createBrokerController(args));
    }

    public static BrokerController createBrokerController(String[] args) {
        System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, 
            Integer.toString(MQVersion.CURRENT_VERSION));

        // 1. 解析命令行参数
        // ./mqbroker -c broker.conf -n namesrv:9876
        //    -c → 配置文件
        //    -n → NameServer 地址
        //    -p → 打印所有配置
        CommandLine commandLine = CommandLine.parse(args);
        String configFile = commandLine.getOptionValue('c');
        String namesrvAddr = commandLine.getOptionValue('n');

        // 2. 创建配置对象
        final BrokerConfig brokerConfig = new BrokerConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        final NettyClientConfig nettyClientConfig = new NettyClientConfig();
        final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();

        // 3. 加载配置文件（覆盖默认值）
        if (configFile != null && !configFile.isEmpty()) {
            InputStream in = new BufferedInputStream(new FileInputStream(configFile));
            Properties p = new Properties();
            p.load(in);
            // 将 properties 的值合并到 BrokerConfig/NettyServerConfig 等
            MixAll.properties2Object(p, brokerConfig);
            MixAll.properties2Object(p, nettyServerConfig);
            MixAll.properties2Object(p, nettyClientConfig);
            MixAll.properties2Object(p, messageStoreConfig);
        }

        // 4. 命令行参数再次覆盖（优先级最高）
        if (commandLine.hasOption('n')) {
            brokerConfig.setNamesrvAddr(namesrvAddr);
        }

        // 5. 创建 BrokerController
        final BrokerController controller = new BrokerController(
            brokerConfig, nettyServerConfig, nettyClientConfig, messageStoreConfig);

        controller.getConfiguration().registerConfig(properties);

        return controller;
    }

    public static void start(BrokerController controller) {
        controller.start();
        String tip = "The broker[" + controller.getBrokerConfig().getBrokerName() 
            + "] boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        System.out.printf("%s%n", tip);
        System.in.read();  // 阻塞主线程，防止退出
    }
}
```

# 2. BrokerController.start() 核心启动流程

```java
// BrokerController.start()
public void start() throws Exception {
    // ========== 阶段 1: 存储层初始化 ==========

    // 1. 创建 MessageStore
    if (this.messageStoreConfig.isEnableDLegerCommitLog()) {
        // DLedger 模式
        this.messageStore = new DLedgerCommitLog(this.brokerConfig);
    } else {
        // 传统模式
        this.messageStore = new DefaultMessageStore(
            this.messageStoreConfig, this.brokerStatsManager, 
            this.messageArrivingListener, this.brokerConfig);
    }

    // 2. 加载存储文件 (恢复 CommitLog, ConsumeQueue, IndexFile)
    boolean loadResult = this.messageStore.load();

    // ========== 阶段 2: 网络层启动 ==========

    // 3. 启动 Netty Remoting Server (端口 10911)
    this.remotingServer.start();

    // 4. 启动 Netty Remoting Client (用于 Broker 之间的通信)
    this.fastRemotingServer.start();

    // ========== 阶段 3: 核心服务启动 ==========

    // 5. 启动 Broker 对外 API (连接 NameServer)
    this.brokerOuterAPI.start();

    // 6. 注册业务处理器 (Processor)
    this.registerProcessor();

    // 7. 启动定时任务
    //   - 向 NameServer 注册 (30s)
    //   - 持久化消费进度 (5s)
    //   - 持久化 Topic 配置 (10s)
    //   - 清理过期文件 (deleteWhen)
    //   - 检查事务消息 (60s)
    this.scheduledExecutorService.scheduleAtFixedRate(...);

    // 8. 启动事务消息检查服务
    this.transactionalMessageCheckService.start();

    // ========== 阶段 4: 消息存储服务启动 ==========

    // 9. 启动消息存储相关服务
    this.getMessageStore().start();
    //   - FlushConsumeQueueService (刷盘)
    //   - ReputMessageService (构建 ConsumeQueue/IndexFile)
    //   - HAService (主从复制)
    //   - CleanCommitLogService / CleanConsumeQueueService (清理)
    //   - ScheduledMessageService (延迟消息)

    // ========== 阶段 5: 拉取服务启动 ==========

    // 10. 启动 Long Polling 服务
    this.pullRequestHoldService.start();

    // 11. 启动 Client 心跳检查服务
    this.clientHousekeepingService.start();

    // 12. 启动 FilterServer 管理线程
    this.filterServerManager.start();

    // 13. 注册 Broker 关闭钩子
    Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
        @Override
        public void run() {
            BrokerController.this.shutdown();
        }
    }));
}
```

# 3. registerProcessor() 处理器注册

```java
public void registerProcessor() {
    // 多种请求类型，分配到不同的 Processor + 不同的 Executor

    // 1. 消息发送 (单一请求码)
    SendMessageProcessor sendProcessor = new SendMessageProcessor(this);
    this.remotingServer.registerProcessor(
        RequestCode.SEND_MESSAGE, sendProcessor, this.sendMessageExecutor);
    this.remotingServer.registerProcessor(
        RequestCode.SEND_MESSAGE_V2, sendProcessor, this.sendMessageExecutor);
    this.remotingServer.registerProcessor(
        RequestCode.SEND_BATCH_MESSAGE, sendProcessor, this.sendMessageExecutor);

    // 2. 消息拉取 (单一请求码)
    PullMessageProcessor pullProcessor = new PullMessageProcessor(this);
    this.remotingServer.registerProcessor(
        RequestCode.PULL_MESSAGE, pullProcessor, this.pullMessageExecutor);

    // 3. 消费进度更新
    ConsumerManageProcessor consumerManageProcessor = new ConsumerManageProcessor(this);
    this.remotingServer.registerProcessor(
        RequestCode.UPDATE_CONSUMER_OFFSET, consumerManageProcessor, ...);

    // 4. 心跳 (多个请求码)
    ClientManageProcessor clientManageProcessor = new ClientManageProcessor(this);
    this.remotingServer.registerProcessor(
        RequestCode.HEART_BEAT, clientManageProcessor, ...);
    this.remotingServer.registerProcessor(
        RequestCode.UNREGISTER_CLIENT, clientManageProcessor, ...);

    // 5. 管理命令 (AdminBrokerProcessor)
    AdminBrokerProcessor adminProcessor = new AdminBrokerProcessor(this);
    // ... 注册多个请求码

    // 6. 消息查询
    QueryMessageProcessor queryProcessor = new QueryMessageProcessor(this);
    // ...

    // 7. 事务消息
    EndTransactionProcessor endTransactionProcessor = new EndTransactionProcessor(this);
    // ...

    // 8. 回复消息 (4.7+)
    ReplyMessageProcessor replyMessageProcessor = new ReplyMessageProcessor(this);
    // ...
}
```

# 4. SendMessageProcessor — 消息处理

```java
// broker/src/main/java/.../SendMessageProcessor.java
public class SendMessageProcessor extends AbstractSendMessageProcessor {

    @Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx, 
            RemotingCommand request) throws RemotingCommandException {
        switch (request.getCode()) {
            case RequestCode.SEND_MESSAGE:
            case RequestCode.SEND_MESSAGE_V2:
                return this.sendMessage(ctx, request, new DefaultSendMessageCallback());
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                return this.consumerSendMsgBack(ctx, request);
            default:
                break;
        }
        return null;
    }
}
```

## 4.1 消息存储入口

```java
// AbstractSendMessageProcessor.sendMessage()
public RemotingCommand sendMessage(ChannelHandlerContext ctx, 
        RemotingCommand request, SendMessageCallback callback) {

    // 1. 解析请求头
    SendMessageRequestHeader requestHeader = parseRequestHeader(request);

    // 2. 构建消息对象 MessageExtBrokerInner
    //    - Topic, Tag, Body, Properties
    //    - bornTimestamp, bornHost, storeTimestamp, storeHost
    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(requestHeader.getTopic());
    msgInner.setQueueId(requestHeader.getQueueId());
    // ... 设置其他字段

    // 3. 事务消息处理
    if (requestHeader.getSysFlag() & MessageSysFlag.TRANSACTION_PREPARED_TYPE) {
        // Half Message
        msgInner.setPreparedTransactionOffset(msgInner.getQueueOffset());
        msgInner.setSysFlag(msgInner.getSysFlag() | MessageSysFlag.TRANSACTION_PREPARED_TYPE);
    }

    // 4. 调用 MessageStore.putMessage() 存储消息
    PutMessageResult putMessageResult = this.messageStore.putMessage(msgInner);

    // 5. 构建响应
    return handlePutMessageResult(putMessageResult, response, request, msgInner, ...);
}
```

# 5. PullMessageProcessor — 消息拉取

```java
// PullMessageProcessor.processRequest()
private RemotingCommand processRequest(Channel channel, RemotingCommand request, 
        boolean brokerAllowSuspend) throws RemotingCommandException {

    // 1. 解析请求头
    PullMessageRequestHeader requestHeader = parseRequestHeader(request);

    // 2. 获取订阅关系
    SubscriptionGroupConfig subscriptionGroupConfig = 
        this.brokerController.getSubscriptionGroupManager()
            .findSubscriptionGroupConfig(requestHeader.getConsumerGroup());

    // 3. 获取过滤数据
    ConsumerFilterData filterData = 
        this.brokerController.getConsumerFilterManager()
            .get(requestHeader.getTopic(), requestHeader.getConsumerGroup());

    // 4. 从 MessageStore 获取消息
    final GetMessageResult getMessageResult = 
        this.brokerController.getMessageStore().getMessage(
            requestHeader.getConsumerGroup(),
            requestHeader.getTopic(),
            requestHeader.getQueueId(),
            requestHeader.getQueueOffset(),
            requestHeader.getMaxMsgNums(),
            filterData
        );

    // 5. 构建响应
    if (getMessageResult.getStatus() == GetMessageStatus.FOUND) {
        response.setCode(ResponseCode.SUCCESS);
        // 编码消息内容
    } else if (getMessageResult.getStatus() == GetMessageStatus.NO_MATCHED_MESSAGE) {
        response.setCode(ResponseCode.PULL_NOT_FOUND);
    } else if (getMessageResult.getStatus() == GetMessageStatus.NO_MESSAGE_IN_QUEUE) {
        // 没有消息 → 进入 Long Polling
        if (brokerAllowSuspend && hasSubscriptionFlag) {
            return this.brokerController.getPullRequestHoldService()
                .suspendPullRequest(topic, queueId, pullRequest);
        }
    }
    return response;
}
```

# 6. 启动后的定时任务

```java
// BrokerController.start() 中的定时任务注册
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        // 1. 向所有 NameServer 注册 (每 30s，可调)
        BrokerController.this.registerBrokerAll(true, false, 
            brokerConfig.isForceRegister());
    }
}, BROKER_INITIAL_DELAY, BROKER_REGISTER_INTERVAL, TimeUnit.MILLISECONDS);

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        // 2. 持久化消费进度 (每 5s)
        BrokerController.this.consumerOffsetManager.persist();
    }
}, 1000 * 10, BROKER_PERSIST_CONSUMER_OFFSET_INTERVAL, TimeUnit.MILLISECONDS);

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        // 3. 持久化 Topic 配置 (每 10s)
        BrokerController.this.topicConfigManager.persist();
    }
}, 1000 * 10, BROKER_PERSIST_TOPIC_CONFIG_INTERVAL, TimeUnit.MILLISECONDS);

this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        // 4. 清理过期文件 (凌晨 deleteWhen 时间)
        BrokerController.this.getMessageStore().cleanExpiredFiles();
    }
}, 1000 * 60, BROKER_CLEAN_EXPIRED_FILES_INTERVAL, TimeUnit.MILLISECONDS);
```

# 7. MessageStore 初始化

```java
// DefaultMessageStore.load()
public boolean load() {
    // 1. 加载 CommitLog (恢复 MappedFileQueue)
    boolean result = this.commitLog.load();

    // 2. 加载 ConsumeQueue (遍历所有 Topic/QueueId)
    result = result && this.loadConsumeQueue();

    // 3. 加载 IndexFile
    result = result && this.loadIndexFile();

    // 4. 恢复状态 (内存中的偏移量、计数器等)
    result = result && this.recover();

    return result;
}
```

# 8. 参考

- [BrokerStartup 源码](https://github.com/apache/rocketmq/blob/master/broker/src/main/java/org/apache/rocketmq/broker/BrokerStartup.java)
- [BrokerController 源码](https://github.com/apache/rocketmq/blob/master/broker/src/main/java/org/apache/rocketmq/broker/BrokerController.java)
