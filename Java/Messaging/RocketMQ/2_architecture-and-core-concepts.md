
# 1. 整体架构

```
                          NameServer 集群
                    /           |           \
              (路由注册)    (路由注册)    (路由注册)
                  /             |             \
            Producer ───→   Broker(M/S)  ←─── Consumer
              │               ┌────────┐        │
              │               │ CommitLog│       │
              └─── 消息发送 ─→│ConsumeQ │ ─→ 消息消费
                              │IndexFile│
                              └────────┘
```

## 1.1 四大核心角色

| 角色 | 职责 | 集群方式 | 有无状态 |
|------|------|----------|----------|
| **NameServer** | 路由注册与发现、Broker 心跳管理 | 无状态集群（多节点独立） | 无 |
| **Broker** | 消息存储、投递、主从复制 | 主从/多副本（DLedger） | 有 |
| **Producer** | 消息生产、发送到 Broker | 无状态集群 | 无 |
| **Consumer** | 消息消费、偏移量管理 | 集群/广播 | 消费进度有状态 |

## 1.2 核心通信流程

```
1. Broker 启动，向所有 NameServer 注册 Topic 路由信息
2. Producer 从 NameServer 获取 Topic 路由信息，建立长连接
3. Producer 发送消息到 Broker Master
4. Broker Master 存储消息，同步/异步复制到 Slave
5. Consumer 从 NameServer 获取路由，连接 Broker 消费
6. Consumer 定时上报消费进度到 Broker
```

# 2. 消息模型核心概念

## 2.1 Topic（主题）

- 消息的一级分类，逻辑上的消息集合
- 例如：`order_topic`、`log_topic`
- 一个 Topic 可配置多个 MessageQueue

## 2.2 Tag（标签）

- 消息的二级分类，Topic 下的子类型
- 一个消息只能有一个 Tag
- Consumer 可通过 Tag 订阅过滤：`topicA || tagA`

## 2.3 MessageQueue（消息队列）

- 消息的物理分区，最小存储/消费单元
- 一个 Topic 下有多个 MessageQueue（默认 4 读 8 写）
- MessageQueue = BrokerName + QueueId
- **作用**：并行写入、有序消费、负载均衡

```
Topic: order_topic
├── MessageQueue-0  [Broker-a, qid=0]
├── MessageQueue-1  [Broker-a, qid=1]
├── MessageQueue-2  [Broker-a, qid=2]
├── MessageQueue-3  [Broker-a, qid=3]
├── MessageQueue-4  [Broker-b, qid=0]
├── MessageQueue-5  [Broker-b, qid=1]
├── MessageQueue-6  [Broker-b, qid=2]
└── MessageQueue-7  [Broker-b, qid=3]
```

## 2.4 ConsumerGroup（消费组）

- 多个 Consumer 实例组成的逻辑组
- **集群模式（Clustering）**：同一组的 Consumer 平均分摊队列，每条消息只被组内一个实例消费
- **广播模式（Broadcasting）**：同一组的每个 Consumer 都消费全部消息

## 2.5 Offset（偏移量）

```
类型          存储位置         描述
─────────────────────────────────────
QueueOffset   消息在队列中位置  消息唯一标识
CommitOffset  Consumer 上报     消费进度
MaxOffset     队列最大偏移量    最新消息位置
```

# 3. 消息生命周期

```
[Producer]
    │
    ▼
[选择 MessageQueue] ←── NameServer 路由表
    │
    ▼
[Broker Master 接收]
    │
    ├──→ [CommitLog 写入] ──→ [同步/异步复制] ──→ [Broker Slave]
    │
    └──→ [ConsumeQueue 构建] ──→ [IndexFile 构建]
    │
    ▼
[Consumer 拉取]
    │ (Long Polling)
    ▼
[消费处理]
    │
    ├── 成功 → 提交 Offset → Broker
    └── 失败 → 重试消费 → 重试次数达上限 → 死信队列
```

# 4. 网络通信模型

RocketMQ 4.x 使用自研的 **Netty Remoting** 层：

```
┌──────────────────────────────────────┐
│          Remoting Protocol           │
├──────────┬───────────────────────────┤
│ 协议编码  │ json / rocketmq (自定义二进制) │
│ 传输层    │ Netty (TCP)                │
│ 序列化    │ FastJson / 自定义           │
│ 请求模型  │ 同步 / 异步 / OneWay        │
│ 编解码    │ 总长度(4B) + Header长度(4B) │
│           │ + Header + Body            │
└──────────┴───────────────────────────┘
```

# 5. 关键配置参数速览

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `brokerRole` | ASYNC_MASTER | 主从角色 |
| `flushDiskType` | ASYNC_FLUSH | 刷盘类型 |
| `defaultTopicQueueNums` | 8 | 写队列数量 |
| `listenPort` | 10911 | Broker 端口 |
| `namesrvAddr` | - | NameServer 地址 |
| `maxMessageSize` | 4MB | 最大消息体 |
| `fileReservedTime` | 72h | 文件保留时间 |

# 6. 参考

- [Architecture — RocketMQ 官方架构文档](https://rocketmq.apache.org/docs/rmq-arc/)
