
# 1. 高可用架构概述

RocketMQ 通过以下机制保证高可用和数据可靠性：

```
┌──────────────────────────────────────────────────────┐
│                    Producer                          │
│    (sendLatencyFaultEnable + 重试 = 自动切换)         │
└────────────────────────┬─────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
    ┌──────────┐                  ┌──────────┐
    │ Master   │ ←── 数据复制 ──→ │  Slave   │
    │ brokerId=0│    (同步/异步)   │ brokerId=1│
    └──────────┘                  └──────────┘
          │                             │
    ┌─────┴─────┐                  ┌───┴────┐
    │ CommitLog │                  │CommitLog│
    │  (1GB × N)│                  │ (只读)  │
    └───────────┘                  └─────────┘
          │
    ┌─────┴──────┐
    │ 刷盘策略    │
    ├────────────┤
    │ 同步/异步刷盘│
    └────────────┘
```

**可靠性保证层级**：
```
Level 0: Producer 发送重试 → 网络故障容错
Level 1: 同步刷盘 → 单机数据不丢失
Level 2: 主从同步复制 → 机器故障不丢数据
Level 3: DLedger 自动选主 → 自动化故障转移
```

# 2. 主从复制

## 2.1 复制方式对比

| 维度 | 同步复制 (SYNC_MASTER) | 异步复制 (ASYNC_MASTER) |
|------|----------------------|------------------------|
| 写入延迟 | 高（等待 Slave 确认） | 低（不等待） |
| 数据可靠性 | 高（Master/Slave 一致） | 中（可能丢失未同步数据） |
| 故障恢复 | Slave 已同步，自动接管 | Slave 可能落后，需人工处理 |
| 适用场景 | 金融/交易等强一致性场景 | 日志/监控等可容忍丢失场景 |

## 2.2 同步复制流程

```
Producer 发送消息
    │
    ▼
Master 写入 CommitLog
    │
    ├── 1. 写入 PageCache
    ├── 2. 构建报文 (消息内容 + offset + size)
    ├── 3. 发送到 Slave (HAService → HAClient)
    │
    ▼
Slave 接收并写入
    │
    ├── 4. 反序列化报文
    ├── 5. 写入本地 CommitLog
    ├── 6. 确认同步 (ack)
    │
    ▼
Master 收到 ack
    │
    ├── 同步刷盘场景：等待刷盘完成
    └── 返回 SendResult.SEND_OK 给 Producer
```

## 2.3 主从复制协议

```
Master → Slave 通讯协议 (HAService)：

报文格式：
┌──────────────────────────────────────┐
│ Header (8B)                          │
│ ├── slaveMaxOffset (8B) — Slave T 偏移│
├──────────────────────────────────────┤
│ Body (变长)                           │
│ └── CommitLog 增量数据               │
└──────────────────────────────────────┘

Slave 定期上报 maxOffset (当前已同步到的位置)
Master 根据差值 (masterOffset - slaveMaxOffset) 决定复制的数据量
```

## 2.4 消息消费 — Slave 读

```java
// Consumer 默认从 Master 消费
// Broker 配置 slaveReadEnable=false 则不从 Slave 读

// 特殊场景（Master 繁忙） → 建议从 Slave 读取
// 配置 consumeMessageOrderly → 建议从 Slave 读取
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
// Broker 配置 slaveReadEnable=true
```

# 3. DLedger — 自动选主

## 3.1 为什么需要 DLedger？

```
传统主从模式的问题：
1. Master 宕机 → 需要人工介入切换 Slave → Master
2. 切换期间不可写 (服务中断)
3. 需要额外的高可用协调组件

DLedger (Distributed Ledger) 解决：
1. 内嵌于 RocketMQ，无额外依赖
2. 基于 Raft 协议自动选主
3. 自动故障转移，秒级恢复
```

## 3.2 DLedger 架构

```
DLedger Group (Raft Group)
┌────────────────────────────────────────┐
│  Node-0 (Leader)                       │
│    └── 接收写入，复制到 Follower        │
│                                         │
│  Node-1 (Follower)                     │
│    └── 接收复制，准备接管               │
│                                         │
│  Node-2 (Follower)                     │
│    └── 接收复制，准备接管               │
└────────────────────────────────────────┘
```

## 3.3 DLedger 部署配置

```properties
# 每个 Brocker 节点的配置 (dlgedger 模式)
# broker-n0.conf
brokerClusterName = RaftCluster
brokerName = RaftNode00
listenPort = 30911
namesrvAddr = 127.0.0.1:9876
storePathRootDir = /tmp/rmqstore/node00
storePathCommitLog = /tmp/rmqstore/node00/commitlog
enableDLegerCommitLog = true
dLegerGroup = RaftNode00
# DLedger 的节点列表 (n0, n1, n2)
dLegerPeers = n0-localhost:40911;n1-localhost:40912;n2-localhost:40913
dLegerSelfId = n0
sendMessageThreadPoolNums = 16
```

## 3.4 DLedger 存储结构

```
DLedger 将 CommitLog 替换为 DLedgerCommitLog：

CommitLog ──→ DLedgerCommitLog
                ├── DLedgerEntryStore
                │    ├── EntryFile (类似 CommitLog)
                │    └── EntryIndexFile (类似 ConsumeQueue)
                ├── DLedgerRaftStore
                │    └── 存储 Raft 状态
                └── DLedgerStateMachine
                     └── 应用到 CommitLog 状态机
```

## 3.5 选主过程

```
1. 每个节点有随机选举超时 (150ms ~ 300ms)
2. 超时后发起选举 → RequestVote RPC
3. 获得多数票 (> N/2) → 成为 Leader
4. Leader 定期发送心跳 (AppendEntries)
5. Leader 宕机 → 超时 → 重新选举
```

# 4. 刷盘策略

## 4.1 同步刷盘 vs 异步刷盘

```
同步刷盘 (SYNC_FLUSH)：
  写入 PageCache → MappedByteBuffer.force() → 返回成功
  每条消息都确保持久化到磁盘
  
异步刷盘 (ASYNC_FLUSH)：
  写入 PageCache → 立即返回成功 → 后台定时 force() (默认 500ms)
  PageCache 中的数据未被持久化前有丢失风险
```

## 4.2 刷盘 vs 主从复制 —— 组合矩阵

```
                    同步刷盘          异步刷盘
                ┌───────────────┬──────────────┐
  同步复制      │ 最高可靠       │ 高可靠         │
  (SYNC_MASTER) │ 延迟最高       │ 延迟较低       │
                │ 金融/交易      │ 订单核心链路    │
                ├───────────────┼──────────────┤
  异步复制      │ 中可靠         │ 性能优先      │
  (ASYNC_MASTER)│ 延迟较低       │ 延迟最低      │
                │ 不推荐         │ 日志/监控     │
                └───────────────┴──────────────┘
```

# 5. Producer 端可靠性保护

## 5.1 发送失败重试

```java
// 同步发送时自动重试（默认 2 次）
producer.setRetryTimesWhenSendFailed(3);
// 存储失败也重试
producer.setRetryAnotherBrokerWhenNotStoreOK(true);
```

## 5.2 故障延迟容错

```java
producer.setSendLatencyFaultEnable(true);
// 如果 Broker 响应慢/失败 → 暂时跳过 → 避免雪崩
```

## 5.3 多 NameServer 容错

```java
// Client 端轮询 NameServer 列表
producer.setNamesrvAddr("ns1:9876;ns2:9876;ns3:9876");
```

# 6. 数据可靠性最佳实践

| 场景 | 推荐配置 | 说明 |
|------|----------|------|
| 交易订单 | `SYNC_MASTER + SYNC_FLUSH` | 数据零丢失 |
| 用户通知 | `SYNC_MASTER + ASYNC_FLUSH` | 平衡可靠性与性能 |
| 日志采集 | `ASYNC_MASTER + ASYNC_FLUSH` | 吞吐优先 |
| 核心链路 | DLedger (≥3节点) | 自动故障转移 |

# 7. 故障恢复场景

## 7.1 Master 宕机

```
场景: Master 宕机，Slave 存活

生产端：
  Producer 自动重试 → 其他 Broker (如果有多个 Master)
  （单主模式：暂不可写，直到 Master 恢复或人工切换）

消费端：
  Consumer 自动切换到 Slave 继续消费 (如果 slaveReadEnable=true)
  （消费进度由 Consumer 端缓存，恢复后可续消费）

人工恢复：
  1. 启动新 Master (或提升 Slave → Master)
  2. 更新 NameServer 路由
  3. Producer 自动发现新 Master
```

## 7.2 DLedger 模式自动故障转移

```
场景: Leader Node-0 宕机

1. Follower 检测心跳超时 → 发起选举
2. Node-1 获得超半票 → 成为新 Leader
3. Producer 自动发现新 Leader (通过 NameServer / 配置更新)
4. 恢复写入
5. 旧 Leader 恢复 → 自动成为 Follower

总时间: 通常 5~15 秒（取决于选举超时配置）
```

# 8. 参考

- [RocketMQ HA 设计](https://github.com/apache/rocketmq/blob/master/docs/cn/design.md)
- [DLedger GitHub](https://github.com/openmessaging/dledger)
