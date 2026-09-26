
# 1. Broker 概述

Broker 是 RocketMQ 的核心组件，承担消息**存储、投递、查询**及**高可用保障**等职责。

## 1.1 启动入口

```java
// broker/src/main/java/org/apache/rocketmq/broker/BrokerStartup.java
public static void main(String[] args) {
    start(createBrokerController(args));
}
public static BrokerController createBrokerController(String[] args) {
    // 1. 解析命令行参数 (-c broker.conf -n namesrv:9876)
    // 2. 创建 BrokerConfig, NettyServerConfig, NettyClientConfig...
    // 3. 混合配置文件与命令行参数
    // 4. 创建 BrokerController
}
```

## 1.2 Broker 角色

```
                      Master (brokerId=0)
                     /        |        \
               (读写)     (读写)     (写服务)
                  /          |          \
        Producer ─┤     Consumer ─┤    管理工具
                  \          |          /
                   \         |         /
            Slave (brokerId>0): 只读备份
```

| 角色 | brokerId | brokerRole | 说明 |
|------|----------|------------|------|
| Master | 0 | SYNC_MASTER / ASYNC_MASTER | 读写 |
| Slave | >=1 | SLAVE | 只读（或根据配置） |
| DLedger 选主 | N/A | - | 自动选主 |

# 2. BrokerController 核心模块

```
BrokerController
├── BrokerConfig                     — 配置管理
├── NettyServerConfig                — 网络服务端配置
├── NettyClientConfig                — 网络客户端配置
├── MessageStore                     — 核心存储模块
│    └── DefaultMessageStore
│         ├── CommitLog              — 消息存储文件
│         ├── ConsumeQueue           — 消费队列
│         ├── IndexFile              — 索引文件
│         ├── FlushConsumeQueueService — ConsumeQueue 刷盘
│         ├── ReputMessageService    — 消息转发（构建 ConsumeQueue）
│         ├── HAService              — 主从同步
│         ├── CleanCommitLogService  — 过期文件清理
│         └── CleanConsumeQueueService — 过期文件清理
├── RemotingServer                   — Netty 网络服务
├── ClientManageProcessor            — 客户端管理请求处理
│    (heartbeat, unregister, checkClient)
├── ConsumerManageProcessor          — 消费者管理请求处理
│    (getConsumerList, consumerManage)
├── SendMessageProcessor             — 消息发送请求处理
│    (sendMessage, sendBatch, consumerSendMsgBack)
├── PullMessageProcessor             — 消息拉取请求处理
│    (pullMessage, queryMessage, viewMessage)
├── QueryMessageProcessor            — 消息查询请求处理
├── AdminBrokerProcessor             — 管理命令请求处理
├── ReplyMessageProcessor            — 应答消息处理（RocketMQ 4.7+）
├── SubscriptionGroupManager         — 消费组订阅关系管理
├── ConsumerOffsetManager            — 消费进度管理
├── TopicConfigManager               — Topic 配置管理
├── ConsumerFilterManager            — 消费者过滤管理
├── BrokerStatsManager               — 统计信息管理
├── ScheduledMessageService          — 延迟消息服务
├── TransactionalMessageService      — 事务消息服务
├── ClientHousekeepingService        — 客户端连接管理
├── PullRequestHoldService           — Pull 长轮询 (Long Polling)
├── BrokerOuterAPI                   — Broker 与其他组件通信
├── RegisterBrokerTask               — 向 NameServer 注册定时任务
├── ScheduledExecutorService         — 定时任务线程池
└── ...                              — 其他
```

# 3. Broker 消息处理架构

```
                     Netty Server (10911)
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
        SendMessage  PullMessage  AdminBroker
        Processor    Processor    Processor
               │            │            │
               └────────────┼────────────┘
                            ▼
                   BusinessHandler
                            │
                    ┌───────┴───────┐
                    ▼               ▼
             PutMessage        GetMessage
                    │               │
                    └───────┬───────┘
                            ▼
                    MessageStore
                 (CommitLog / ConsumeQueue)
```

## 3.1 请求处理线程模型

```
Netty Worker Group (已设置线程名称前缀)
    ↓
    ├── Default → 工作线程池 (SendMessageThreadPoolNums=默认 1 个线程池)
    ├── Async → 异步处理
    └── OneWay → 单向处理
```

```java
// BrokerController.registerProcessor() — 注册处理器
this.remotingServer.registerProcessor(
    RequestCode.SEND_MESSAGE,           // 请求码
    this.sendMessageProcessor,          // 处理器
    this.sendMessageExecutor            // 处理线程池
);
// 通过线程池隔离不同请求类型，避免相互影响
```

## 3.2 线程池配置

| 线程池 | 用途 | 默认线程数 |
|--------|------|-----------|
| `sendMessageExecutor` | 处理 SEND_MESSAGE 请求 | 1 |
| `pullMessageExecutor` | 处理 PULL_MESSAGE 请求 | 16 + brokerId |
| `queryMessageExecutor` | 处理 QUERY_MESSAGE 请求 | 8 |
| `adminBrokerExecutor` | 处理管理员请求 | 16 |
| `consumerManageExecutor` | 处理消费者管理 | 16 |

# 4. 通信协议

## 4.1 请求/响应头

```java
// 请求头
SendMessageRequestHeader {
    String producerGroup;
    String topic;
    String defaultTopic;
    Integer defaultTopicQueueNums;
    Integer queueId;
    Integer sysFlag;
    Long bornTimestamp;
    Integer flag;
    String properties;
    Integer reconsumeTimes;
    boolean unitMode;
    boolean batch;
    Integer maxReconsumeTimes;
}
```

## 4.2 请求码一览

```java
public class RequestCode {
    // 消息发送
    public static final int SEND_MESSAGE = 10;
    public static final int SEND_MESSAGE_V2 = 310;
    public static final int SEND_BATCH_MESSAGE = 320;

    // 消息拉取
    public static final int PULL_MESSAGE = 11;
    public static final int LITE_PULL_MESSAGE = 361;

    // 消费者
    public static final int QUERY_CONSUMER_OFFSET = 14;
    public static final int UPDATE_CONSUMER_OFFSET = 15;
    public static final int CONSUMER_SEND_MSG_BACK = 36;

    // 心跳
    public static final int HEART_BEAT = 34;
    public static final int UNREGISTER_CLIENT = 35;

    // 注册
    public static final int REGISTER_BROKER = 103;

    // 查询
    public static final int GET_ROUTEINFO_BY_TOPIC = 105;

    // ... 更多
}
```

# 5. 延迟消息调度 (ScheduledMessageService)

```
延迟消息投递流程：
Producer 发送消息 (delayLevel=3)
    │
    ▼
Broker 接收消息 ──→ 写入 CommitLog
    │
    ├── 替换 originTopic → SCHEDULE_TOPIC_XXXX
    ├── 替换 originQueueId → delayLevel - 1
    └── 存储原始信息到 properties (REAL_TOPIC, REAL_QID...)
    │
    ▼
ScheduledMessageService 定时扫描
    │
    ├── Level 1: 1s    → TimerTask
    ├── Level 2: 5s    → TimerTask
    ├── Level 3: 10s   → TimerTask
    ├── ...
    └── Level 18: 2h   → TimerTask
    │
    ▼
到期 → 恢复原始 Topic/QueueId → 重新写入 CommitLog
    │
    ▼
Consumer 正常消费
```

# 6. 事务消息管理 (TransactionalMessageService)

```
事务消息流程：
1. Producer 发送 Half Message
2. Broker 存储消息 (标记 TRANSACTION_NOT_TYPE 或 TRANSACTION_PREPARED_TYPE)
3. Producer 执行本地事务
4. Producer 提交 Commit / Rollback
5. Broker 根据结果处理：
   ├── Commit → 消息对 Consumer 可见
   └── Rollback → 逻辑删除
6. 超时未提交 → Broker 发起回查 → Producer checkLocalTransaction()
```

# 7. Broker 配置体系

## 7.1 核心配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `brokerClusterName` | DefaultCluster | 集群名称 |
| `brokerName` | - | Broker 名称，主从相同 |
| `brokerId` | 0 | 0=Master，>0=Slave |
| `brokerRole` | ASYNC_MASTER | Broker 角色 |
| `flushDiskType` | ASYNC_FLUSH | 刷盘策略 |
| `namesrvAddr` | - | NameServer 地址 |
| `listenPort` | 10911 | Broker 服务端口（Netty） |
| `haListenPort` | 10912 | HA 服务端口 |
| `storePathRootDir` | ~/store | 存储根目录 |
| `storePathCommitLog` | ~/store/commitlog | CommitLog 目录 |
| `mapedFileSizeCommitLog` | 1GB | 单个 CommitLog 文件大小 |
| `fileReservedTime` | 72h | 文件保留时间 |
| `deleteWhen` | 04 (凌晨4点) | 定时删除时间 |
| `maxMessageSize` | 4MB | 消息最大大小 |
| `autoCreateTopicEnable` | true | 自动创建 Topic |
| `autoCreateSubscriptionGroup` | true | 自动创建消费组 |
| `useTLS` | false | 是否启用 TLS |
| `aclEnable` | false | 是否启用 ACL |

# 8. 参考

- [Broker 启动类 — BrokerStartup](https://github.com/apache/rocketmq/blob/master/broker/src/main/java/org/apache/rocketmq/broker/BrokerStartup.java)
- [BrokerController](https://github.com/apache/rocketmq/blob/master/broker/src/main/java/org/apache/rocketmq/broker/BrokerController.java)
