
> 源码版本：RocketMQ 4.9.x
> 核心模块：`namesrv/src/main/java/org/apache/rocketmq/namesrv/`

# 1. NameServer 启动流程

## 1.1 NamesrvStartup

```java
// namesrv/src/main/java/.../NamesrvStartup.java
public class NamesrvStartup {
    public static void main(String[] args) {
        main0(args);
    }

    public static NamesrvController main0(String[] args) {
        // 1. 创建 Netty 配置
        final NamesrvConfig namesrvConfig = new NamesrvConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        nettyServerConfig.setListenPort(9876);  // 默认端口

        // 2. 解析命令行参数 (-c 配置文件)
        CommandLine commandLine = CommandLine.parse(args);
        if (commandLine.hasOption('c')) {
            String file = commandLine.getOptionValue('c');
            InputStream in = new BufferedInputStream(new FileInputStream(file));
            Properties properties = new Properties();
            properties.load(in);
            MixAll.properties2Object(properties, namesrvConfig);
            MixAll.properties2Object(properties, nettyServerConfig);
        }

        // 3. 创建 NamesrvController
        final NamesrvController controller = new NamesrvController(
            namesrvConfig, nettyServerConfig);

        // 4. 初始化 + 启动
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }

        // 5. 注册 JVM 关闭钩子
        Runtime.getRuntime().addShutdownHook(new Thread(controller::shutdown));

        controller.start();

        return controller;
    }
}
```

## 1.2 NamesrvController.initialize()

```java
public boolean initialize() {
    // 1. 加载 KV 配置 (namesrv/kvConfig.json)
    this.kvConfigManager.load();

    // 2. 创建 Netty 服务器 (监听 9876)
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, 
        this.brokerHousekeepingService);

    // 3. 注册请求处理器
    this.registerProcessor();

    // 4. 启动定时任务 (10s 一次扫描不活跃 Broker)
    this.scheduledExecutorService.scheduleAtFixedRate(
        () -> {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }, 5, 10, TimeUnit.SECONDS);

    // 5. 启动另一个定时任务 (10min 一次打印 KV config)
    this.scheduledExecutorService.scheduleAtFixedRate(
        () -> {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }, 1, 10, TimeUnit.MINUTES);

    // 6. 开启 TLS（如果配置了）
    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
        // ...
    }

    return true;
}
```

# 2. 请求处理器注册

```java
// NamesrvController.registerProcessor()
private void registerProcessor() {
    if (namesrvConfig.isClusterTest()) {
        // 测试环境：直接注册
        this.remotingServer.registerDefaultProcessor(
            new ClusterTestRequestProcessor(this, 
                adminBrokerProcessor.getProduct()), this.defaultProcessorExecutor);
    } else {
        // 生产环境
        NettyRequestProcessor defaultProcessor = new DefaultRequestProcessor(this);

        // 1. 注册 KV Config 操作
        this.remotingServer.registerProcessor(
            RequestCode.PUT_KV_CONFIG, defaultProcessor, ...);
        this.remotingServer.registerProcessor(
            RequestCode.GET_KV_CONFIG, defaultProcessor, ...);
        this.remotingServer.registerProcessor(
            RequestCode.DELETE_KV_CONFIG, defaultProcessor, ...);

        // 2. 注册路由注册 (Broker 使用)
        this.remotingServer.registerProcessor(
            RequestCode.REGISTER_BROKER, defaultProcessor, ...);

        // 3. 取消注册
        this.remotingServer.registerProcessor(
            RequestCode.UNREGISTER_BROKER, defaultProcessor, ...);

        // 4. 路由查询 (Producer/Consumer 使用)
        this.remotingServer.registerProcessor(
            RequestCode.GET_ROUTEINFO_BY_TOPIC, defaultProcessor, ...);

        // 5. 获取 Broker 集群信息
        this.remotingServer.registerProcessor(
            RequestCode.GET_BROKER_CLUSTER_INFO, defaultProcessor, ...);

        // 6. 获取所有 Topic 列表
        this.remotingServer.registerProcessor(
            RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER, defaultProcessor, ...);

        // 7. 获取 Topic 的历史信息
        this.remotingServer.registerProcessor(
            RequestCode.GET_TOPICS_BY_CLUSTER, defaultProcessor, ...);

        // 8. 删除 Topic
        this.remotingServer.registerProcessor(
            RequestCode.DELETE_TOPIC_IN_NAMESRV, defaultProcessor, ...);

        // 9. 获取系统 Topic 列表
        this.remotingServer.registerProcessor(
            RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS, defaultProcessor, ...);

        // 10. Broker 心跳
        this.remotingServer.registerProcessor(
            RequestCode.BROKER_HEARTBEAT, defaultProcessor, ...);

        // 11. 单元化相关
        this.remotingServer.registerProcessor(
            RequestCode.GET_UNIT_TOPIC_LIST, defaultProcessor, ...);
    }
}
```

# 3. DefaultRequestProcessor 核心处理

```java
public class DefaultRequestProcessor implements NettyRequestProcessor {
    private final RouteInfoManager routeInfoManager =
        namesrvController.getRouteInfoManager();

    @Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
            RemotingCommand request) throws RemotingCommandException {
        
        switch (request.getCode()) {
            case RequestCode.PUT_KV_CONFIG:
                return this.putKVConfig(ctx, request);
            case RequestCode.GET_KV_CONFIG:
                return this.getKVConfig(ctx, request);
            case RequestCode.REGISTER_BROKER:
                // ★ 核心：Broker 注册
                return this.registerBroker(ctx, request);
            case RequestCode.UNREGISTER_BROKER:
                return this.unregisterBroker(ctx, request);
            case RequestCode.GET_ROUTEINFO_BY_TOPIC:
                // ★ 核心：Topic 路由查询
                return this.getRouteInfoByTopic(ctx, request);
            case RequestCode.GET_BROKER_CLUSTER_INFO:
                return this.getBrokerClusterInfo(ctx, request);
            // ... 更多 case
            default:
                break;
        }
        return null;
    }
}
```

# 4. RouteInfoManager 核心源码

## 4.1 数据结构

```java
public class RouteInfoManager {
    private static final InternalLogger log = 
        InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

    // 读写锁（保证线程安全）
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    // Topic → QueueData 列表
    // Key: Topic 名称
    // Value: List<QueueData> (每个 Broker 一个 QueueData)
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;

    // BrokerName → BrokerData
    // Key: BrokerName (eg. "broker-a")
    // Value: BrokerData {cluster, brokerAddrs: {0→ip1, 1→ip2}}
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;

    // ClusterName → BrokerName 集合
    private final HashMap<String/* clusterName */, Set<String>> clusterAddrTable;

    // BrokerAddr → BrokerLiveInfo (存活信息)
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;

    // BrokerAddr → FilterServer 地址列表
    private final HashMap<String/* brokerAddr */, List<String>> filterServerTable;
}
```

## 4.2 registerBroker — 注册 Broker

```java
// RouteInfoManager.registerBroker() 
// (完整版见 06_nameserver.md 4.2 节)

public RegisterBrokerResult registerBroker(
        final String clusterName, final String brokerAddr,
        final String brokerName, final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList, final Channel channel) {

    RegisterBrokerResult result = new RegisterBrokerResult();

    try {
        // 加写锁
        this.lock.writeLock().lockInterruptibly();

        // 1. 更新 clusterAddrTable
        Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
        if (null == brokerNames) {
            brokerNames = new HashSet<>();
            this.clusterAddrTable.put(clusterName, brokerNames);
        }
        brokerNames.add(brokerName);

        // 2. 更新 brokerAddrTable
        boolean registerFirst = false;
        BrokerData brokerData = this.brokerAddrTable.get(brokerName);
        if (null == brokerData) {
            registerFirst = true;
            brokerData = new BrokerData(clusterName, brokerName, 
                new HashMap<Long, String>());
            this.brokerAddrTable.put(brokerName, brokerData);
        }
        String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
        registerFirst = registerFirst || (null == oldAddr);

        // 3. 更新 topicQueueTable
        // Master (brokerId == 0) 才更新 Topic 配置
        if (null != topicConfigWrapper && MixAll.MASTER_ID == brokerId) {
            if (this.isBrokerTopicConfigChanged(brokerAddr, 
                    topicConfigWrapper.getDataVersion()) || registerFirst) {
                ConcurrentMap<String, TopicConfig> tcTable =
                    topicConfigWrapper.getTopicConfigTable();
                if (tcTable != null) {
                    for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                        this.createAndUpdateQueueData(brokerName, entry.getValue());
                    }
                }
            }
        }

        // 4. 更新 brokerLiveTable (心跳刷新)
        BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
            new BrokerLiveInfo(
                System.currentTimeMillis(),          // 最后心跳时间
                topicConfigWrapper.getDataVersion(), // Broker 数据版本
                channel,                             // Netty 通道
                haServerAddr));                      // HA 服务地址

        // 5. 返回 Master 地址和 HA 地址给 Broker
        result.setHaServerAddr(haServerAddr);
        result.setMasterAddr(brokerData.getBrokerAddrs().get(MixAll.MASTER_ID));

    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    } finally {
        this.lock.writeLock().unlock();
    }

    return result;
}
```

## 4.3 getRouteInfoByTopic — 获取 Topic 路由

```java
// RouteInfoManager.pickupTopicRouteData()
public TopicRouteData pickupTopicRouteData(final String topic) {
    TopicRouteData topicRouteData = new TopicRouteData();

    boolean foundQueueData = false;
    boolean foundBrokerData = false;
    Set<String> brokerNameSet = new HashSet<>();

    List<BrokerData> brokerDataList = new LinkedList<>();
    List<QueueData> queueDataList = new LinkedList<>();

    try {
        // 加读锁
        this.lock.readLock().lockInterruptibly();

        // 1. 获取 QueueData
        List<QueueData> queueDataList = this.topicQueueTable.get(topic);
        if (queueDataList != null) {
            queueDataList.addAll(queueDataList);
        }

        // 2. 获取 BrokerData
        for (QueueData qd : queueDataList) {
            String brokerName = qd.getBrokerName();
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (brokerData != null) {
                brokerDataList.add(new BrokerData(brokerData));
            }
        }
    } finally {
        this.lock.readLock().unlock();
    }

    // 3. 构建返回结果
    topicRouteData.setQueueDatas(queueDataList);
    topicRouteData.setBrokerDatas(brokerDataList);
    return topicRouteData;
}
```

## 4.4 scanNotActiveBroker — 扫描不活跃 Broker

```java
// RouteInfoManager.scanNotActiveBroker() — 定时任务 10s 一次
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = 
        this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        // 超过 120 秒没有心跳 → 判定为不活跃
        if ((System.currentTimeMillis() - last) > BROKER_CHANNEL_EXPIRED_TIME) {
            log.info("The broker channel expired, {}:{}", 
                next.getKey(), last);
            // 关闭 Netty Channel
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
            it.remove();  // 从存活表中移除
        }
    }
}

// Channel 断开后的清理
public void onChannelDestroy(String remoteAddr, Channel channel) {
    String brokerAddrFound = null;

    // 1. 从 brokerLiveTable 移除
    if (channel != null) {
        for (Entry<String, BrokerLiveInfo> entry : this.brokerLiveTable.entrySet()) {
            if (entry.getValue().getChannel() == channel) {
                brokerAddrFound = entry.getKey();
                break;
            }
        }
    }

    if (null == brokerAddrFound) {
        brokerAddrFound = remoteAddr;
    }

    // 2. 从 brokerAddrTable 移除 Broker 地址
    if (brokerAddrFound != null && brokerAddrFound.length() > 0) {
        try {
            this.lock.writeLock().lockInterruptibly();
            // 从 brokerLiveTable 移除
            this.brokerLiveTable.remove(brokerAddrFound);
            // 从 filterServerTable 移除
            this.filterServerTable.remove(brokerAddrFound);

            // 从 brokerAddrTable 中移除对应的 Broker 地址
            String brokerNameFound = null;
            boolean removeBrokerName = false;
            Iterator<Entry<String, BrokerData>> itBrokerAddrTable =
                this.brokerAddrTable.entrySet().iterator();
            while (itBrokerAddrTable.hasNext()) {
                Entry<String, BrokerData> entry = itBrokerAddrTable.next();
                Iterator<Entry<Long, String>> itBrokerAddrs = 
                    entry.getValue().getBrokerAddrs().entrySet().iterator();
                while (itBrokerAddrs.hasNext()) {
                    Entry<Long, String> eea = itBrokerAddrs.next();
                    if (brokerAddrFound.equals(eea.getValue())) {
                        itBrokerAddrs.remove();
                        brokerNameFound = entry.getKey();
                        break;
                    }
                }
            }

            // 3. 从 clusterAddrTable 移除
            // ... (移除 Broker 名下的 BrokerSet)
        } finally {
            this.lock.writeLock().unlock();
        }
    }
}
```

# 5. 数据模型

## 5.1 QueueData

```java
public class QueueData {
    private String brokerName;       // Broker 名称
    private int readQueueNums;       // 读队列数量
    private int writeQueueNums;      // 写队列数量
    private int perm;                // 权限 (2=写, 4=读, 6=读写)
    private int topicSyncFlag;       // Topic 同步标志
}
```

## 5.2 BrokerData

```java
public class BrokerData {
    private String cluster;                      // 集群名
    private String brokerName;                   // Broker 名
    private HashMap<Long/* brokerId */, String/* brokerAddr */> brokerAddrs;
}
```

## 5.3 TopicRouteData

```java
public class TopicRouteData {
    private String orderTopicConf;          // 顺序 Topic 配置
    private List<QueueData> queueDatas;     // QueueData 列表
    private List<BrokerData> brokerDatas;   // BrokerData 列表
}
```

# 6. 参考

- [NamesrvController 源码](https://github.com/apache/rocketmq/blob/master/namesrv/src/main/java/org/apache/rocketmq/namesrv/NamesrvController.java)
- [RouteInfoManager 源码](https://github.com/apache/rocketmq/blob/master/namesrv/src/main/java/org/apache/rocketmq/namesrv/routeinfo/RouteInfoManager.java)
