
# 1. NameServer 概述

NameServer 是 RocketMQ 的**轻量级路由注册中心**，职责单一：
- **路由注册**：Broker 启动时向所有 NameServer 注册 Topic 路由信息
- **路由发现**：Producer/Consumer 从 NameServer 获取路由表
- **Broker 健康检测**：通过心跳机制剔除故障 Broker

## 1.1 设计哲学

```
┌────────────────────────────────────┐
│  为什么不用 ZooKeeper？             │
│                                    │
│  1. 功能精简：只需路由发现，不需要  │
│     ZK 的发布/订阅、选举等功能      │
│  2. CAP 权衡：优先 AP（可用性）     │
│     优于 CP（一致性）               │
│  3. 部署简单：单机即可运行，无需    │
│     多节点共识协议                   │
│  4. 无状态：NameServer 之间不通信， │
│     每台是完整独立的路由表           │
└────────────────────────────────────┘
```

# 2. NameServer 架构

```
                 NamesrvController
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   RouteInfoManager  KVConfigManager  NettyRemotingServer
        │                                │
   ┌────┴────┐                    ┌─────┴─────┐
   │topicQueue│                    │ Port: 9876│
   │brokerAddr│                    │ Netty Server│
   │brokerLive│                    │ 请求处理    │
   │filterSvr │                    └───────────┘
   └─────────┘
```

# 3. 路由信息管理 (RouteInfoManager)

## 3.1 核心数据结构

```java
public class RouteInfoManager {
    // Topic → QueueData 列表（Topic 的路由信息）
    // Key: Topic 名称
    // Value: {BrokerName, readQueueNums, writeQueueNums, perm...}
    private final HashMap<String, List<QueueData>> topicQueueTable;

    // BrokerName → BrokerData（Broker 的地址信息）
    // Key: BrokerName (如 broker-a)
    // Value: {cluster, brokerAddrs: {0→ip1:port1, 1→ip2:port2}}
    private final HashMap<String, BrokerData> brokerAddrTable;

    // ClusterName → BrokerName 列表
    private final HashMap<String, Set<String>> clusterAddrTable;

    // Broker 地址 → 存活信息
    private final HashMap<String, BrokerLiveInfo> brokerLiveTable;

    // Broker 地址 → FilterServer 地址
    private final HashMap<String, List<String>> filterServerTable;
}
```

## 3.2 BrokerLiveInfo

```java
class BrokerLiveInfo {
    long lastUpdateTimestamp;      // 最后心跳时间
    DataVersion dataVersion;       // 数据版本号
    Channel channel;               // Netty 连接通道
    String haServerAddr;           // HA 服务地址
}
```

# 4. 路由注册流程

```
Broker 启动
    │
    ├─ 每 30s (registerBrokerInterval) 向所有 NameServer 发送心跳
    │   请求：REQUEST_CODE.REGISTER_BROKER (103)
    │   携带信息：ClusterName, BrokerName, BrokerAddr, BrokerId, TopicConfig...
    │
    ▼
NameServer 处理 (DefaultRequestProcessor.registerBroker())
    │
    ├─ 1. 加写锁 (this.lock.writeLock().lock())
    │   2. 更新 clusterAddrTable
    │   3. 更新 brokerAddrTable
    │   4. 更新 topicQueueTable (TopicConfig 下所有 Topic)
    │   5. 更新 brokerLiveTable (lastUpdateTimestamp = now)
    │   6. 解写锁
    │
    ▼
Broker 收到响应 → 继续下一次心跳周期
```

## 4.1 心跳注册请求详解

```java
// Broker 端发送心跳 (BrokerOuterAPI.registerBroker())
RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
requestHeader.setBrokerName(this.brokerConfig.getBrokerName());
requestHeader.setBrokerAddr(this.brokerConfig.getBrokerIP1() + ":" + address);
requestHeader.setBrokerId(this.brokerConfig.getBrokerId());
requestHeader.setClusterName(this.brokerConfig.getBrokerClusterName());
requestHeader.setHaServerAddr(haServerAddr);
requestHeader.setCompressed(false);

// 请求体 (RegisterBrokerBody)
// → TopicConfigSerializeWrapper (包含所有 Topic 的 TopicConfig)
RegisterBrokerBody requestBody = new RegisterBrokerBody();
requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
```

## 4.2 NameServer 处理请求核心逻辑

```java
// RouteInfoManager.registerBroker() 核心代码简化
public RegisterBrokerResult registerBroker(
        String clusterName, String brokerAddr, String brokerName,
        long brokerId, String haServerAddr, 
        TopicConfigSerializeWrapper topicConfigWrapper, ...) {

    this.lock.writeLock().lockInterruptibly();

    // 更新 brokerAddrTable
    Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
    if (brokerNames == null) {
        brokerNames = new HashSet<>();
        this.clusterAddrTable.put(clusterName, brokerNames);
    }
    brokerNames.add(brokerName);

    // 标记是否为首次注册
    boolean registerFirst = false;
    BrokerData brokerData = this.brokerAddrTable.get(brokerName);
    if (brokerData == null) {
        registerFirst = true;
        brokerData = new BrokerData(clusterName, brokerName, new HashMap<>());
        this.brokerAddrTable.put(brokerName, brokerData);
    }

    // 更新 Broker 地址
    String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
    registerFirst = registerFirst || (oldAddr == null);

    // 更新 Topic 路由信息
    if (topicConfigWrapper != null && brokerId == MixAll.MASTER_ID) {
        // Master 才更新 Topic 路由
        if (this.isBrokerTopicConfigChanged(...) || registerFirst) {
            ConcurrentMap<String, TopicConfig> tcTable = topicConfigWrapper.getTopicConfigTable();
            for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                // 为每个 Topic 创建 QueueData
                this.createAndUpdateQueueData(brokerName, entry.getValue());
            }
        }
    }

    // 更新 Broker 存活信息
    BrokerLiveInfo prevBrokerLive = this.brokerLiveTable.put(brokerAddr,
        new BrokerLiveInfo(System.currentTimeMillis(), 
            topicConfigWrapper.getDataVersion(), channel, haServerAddr));

    this.lock.writeLock().unlock();

    // 返回 NameServer 版本的 MasterAddr 和 HaServerAddr
    return new RegisterBrokerResult(haServerAddr, masterAddr);
}
```

# 5. 路由发现

## 5.1 Producer/Consumer 获取路由

```java
// 客户端定时任务：每 30s 从 NameServer 拉取路由
// MQClientInstance.updateTopicRouteInfoFromNameServer()

// 请求：REQUEST_CODE.GET_ROUTEINFO_BY_TOPIC (105)
TopicRouteData routeData = mQClientAPIImpl
    .getTopicRouteInfoFromNameServer(topic, timeout);

// TopicRouteData 结构
public class TopicRouteData {
    List<QueueData> queueDatas;      // QueueData 列表
    List<BrokerData> brokerDatas;    // Broker 地址列表
    // ...
}
```

## 5.2 路由缓存与更新

```
TopicRouteInfo 在 Client 端缓存

更新时机：
1. 定时更新（每 30s）
2. 发送消息时发现路由变化（Broker 返回 Topic Not Exist）
3. 连接断开时
```

# 6. 心跳机制与故障剔除

## 6.1 Broker 心跳

```
Broker ──(每30s)──→ NameServer (注册/心跳)
                      │
                      └── 更新 brokerLiveTable.lastUpdateTimestamp
```

## 6.2 故障检测

```java
// NameServer 每 10s 扫描 brokerLiveTable
// NamesrvController.initialize() → this.scheduledExecutorService.scheduleAtFixedRate()

// 扫描逻辑
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = 
        this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> entry = it.next();
        long last = entry.getValue().getLastUpdateTimestamp();
        // 超过 120s 未收到心跳 → 剔除
        if ((System.currentTimeMillis() - last) > BROKER_CHANNEL_EXPIRED_TIME) {
            this.onChannelDestroy(entry.getKey(), entry.getValue().getChannel());
            it.remove();
        }
    }
}
```

**关键常量**：
- `BROKER_CHANNEL_EXPIRED_TIME` = 120s（120 秒无心跳则剔除）
- 扫描间隔 = 10s
- 心跳间隔 = 30s

# 7. 高可用设计

## 7.1 NameServer 集群

```
NameServer-1        NameServer-2        NameServer-3
(独立)              (独立)              (独立)
    ↑                   ↑                   ↑
    └── Broker 同时注册 ─┴── 全部三台 ───────┘
    └── Client 连接任意一台即可 ──────────┘
```

- NameServer 之间**不通信**，每一台是独立的
- Broker 向**所有 NameServer** 注册
- Client 连接**任意一台**即可，支持配置多个地址
- 一致性靠 Broker 定期注册保证

## 7.2 客户端多 NameServer 配置

```java
// 多个 NameServer 地址用分号分割
producer.setNamesrvAddr("192.168.1.1:9876;192.168.1.2:9876");

// 或设置系统属性
System.setProperty("rocketmq.namesrv.addr", "ns1:9876;ns2:9876");
```

# 8. 最佳实践

1. **至少部署 2 台** NameServer，避免单点故障
2. Broker 配置 `namesrvAddr` 为全部 NameServer 地址
3. 重启 NameServer **不会丢失数据**（Broker 会自动重新注册）
4. 监控 NameServer 的内存和 CPU，通常资源消耗很低
5. 如果 NameServer 全部宕机，**已有连接不受影响**，但无法感知路由变更

# 9. 参考

- [RocketMQ 源码 — NamesrvController](https://github.com/apache/rocketmq/blob/master/namesrv/src/main/java/org/apache/rocketmq/namesrv/NamesrvController.java)
