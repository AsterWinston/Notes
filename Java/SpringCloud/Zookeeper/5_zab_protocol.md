
# 1. ZAB 协议概述

ZAB（ZooKeeper Atomic Broadcast，ZooKeeper 原子广播协议）是 ZooKeeper 自研的一致性协议，为 ZooKeeper 提供**崩溃恢复**和**原子广播**两大核心能力。

> **设计定位**：与 Paxos、Raft 齐名的分布式一致性协议，但 ZAB 专为 ZooKeeper 的点对多主备模型设计。ZAB 不是 Paxos 的简单实现，而是一个相似的独立协议。

```text
ZAB 协议核心目标：
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  1. 原子广播（Atomic Broadcast）                             │
│     · 将一组消息按完全相同的顺序投递给所有副本                │
│     · 要么所有副本都收到消息，要么一个都没有                  │
│                                                              │
│  2. 崩溃恢复（Crash Recovery）                               │
│     · Leader 崩溃后自动恢复                                  │
│     · 保证不丢失已提交的事务                                  │
│     · 保证不出现"幽灵"状态（被旧 Leader 提交但不应提交的操作）  │
│                                                              │
│  3. 顺序一致性                                               │
│     · 所有副本按相同的全局顺序应用状态变更                    │
│     · 客户端看到的更新顺序与发送顺序一致                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

# 2. ZAB 协议的两大阶段

ZAB 协议运行在两种模式之间交替：

```text
            ┌──────────────┐
            │              │
     ┌─────►│  广播模式     │  ← 正常运行状态
     │      │ (Broadcast) │
     │      │              │
     │      └──────┬───────┘
     │             │
     │      Leader 宕机 或 失去过半支持
     │             │
     │             ▼
     │      ┌──────────────┐
     │      │              │
     └──────│  恢复模式     │  ← 选举 + 数据同步
            │ (Recovery)  │
            │              │
            └──────────────┘
              新 Leader 选出
              + 数据同步完成
```

## 2.1 广播模式（Broadcast）

广播模式是 ZooKeeper 的**正常运行状态**，类似两阶段提交（2PC）的简化版。

```text
ZAB 广播流程（写请求的完整生命周期）：

Client          Leader                    Follower1             Follower2
  │                │                          │                    │
  │──write("/a")──►│                          │                    │
  │                │                          │                    │
  │                │ 1. Leader 收到写请求                              │
  │                │    生成 Proposal                                     │
  │                │    (zxid, path, data)                                     │
  │                │                          │                    │
  │                │ 2. 广播 Proposal  ──────►│                    │
  │                │───────────────────────────────►              │
  │                │                          │                    │
  │                │ 3. Leader 本地持久化      │ 4. 各自持久化       │
  │                │    写入事务日志            │    写入事务日志     │
  │                │                          │                    │
  │                │◄──────── ACK ────────────│                    │
  │                │◄──────────────────────────── ACK ────────────│
  │                │                          │                    │
  │                │ 5. 收到过半 ACK (2/3)                             │
  │                │                          │                    │
  │                │ 6. Leader 先本地 Commit                            │
  │                │    (应用到内存数据库)                                    │
  │                │                          │                    │
  │                │ 7. 广播 COMMIT ────────►│                    │
  │                │───────────────────────────────►              │
  │                │                          │                    │
  │◄───response────│                          │ 8. 各自提交         │
  │                │                          │    应用到内存DB     │
  └────────────────┴──────────────────────────┴────────────────────┘
```

**广播模式的关键特性**：

| 特性 | 说明 |
|------|------|
| **FIFO 通道** | Leader 和每个 Follower 之间维护一个 FIFO 的消息队列 |
| **顺序保证** | 同一个 Proposal 对所有 Follower 按相同顺序投递 |
| **过半提交** | 收到过半 Follower 的 ACK 后，Leader 才提交 |
| **异步广播** | Leader 不需要等待所有 ACK 再发下一个 Proposal，可以流水线处理 |

## 2.2 恢复模式（Recovery）

恢复模式在 Leader 崩溃或重启后触发，由三个子阶段组成：

```text
恢复模式完整流程：

┌──────────────────────────────────────────────────────────┐
│                                                           │
│  阶段 1：Leader 选举（Leader Election）                    │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  · 所有节点进入 LOOKING 状态                          │ │
│  │  · 每个节点投票给自己（或已知的最新节点）              │ │
│  │  · 交换投票信息，选出最新（zxid 最大、myid 最大）     │ │
│  │  · 过半节点达成共识 → 新 Leader 选出                 │ │
│  └─────────────────────────────────────────────────────┘ │
│                           │                               │
│                           ▼                               │
│  阶段 2：发现（Discovery）                                │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  · 新 Leader 收集所有 Follower 的最新 epoch              │
│  │  · 确定当前 epoch（新 Leader 提议 +1）               │ │
│  │  · 新 Leader 收集 Follower 的已接受 Proposal 历史    │ │
│  └─────────────────────────────────────────────────────┘ │
│                           │                               │
│                           ▼                               │
│  阶段 3：同步（Synchronization）                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  · 新 Leader 和 Follower 对比事务日志差异             │ │
│  │  · 将差异数据同步给落后的 Follower                   │ │
│  │  · 截断 Follower 上任何未被过半提交的 Proposal        │ │
│  │  · 所有 Follower 追上 Leader 后 → 进入广播模式       │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

# 3. ZAB 协议的核心保障

## 3.1 已经提交的事务不能丢失

```text
保障机制：
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  场景：旧 Leader 提交了一个事务后立即崩溃                      │
│                                                               │
│  旧 Leader 提交时满足：至少有过半 Follower 收到了 Proposal     │
│  → 任何新选出 Leader 的节点中，至少有一个包含了这个 Proposal  │
│  → 新 Leader 拥有最大 zxid，必然包含了所有已提交的事务        │
│                                                               │
│  数学证明（鸽巢原理）：                                       │
│    集群 N=5 节点，达成共识需要 3 票                            │
│    新 Leader 需要 3 票当选                                     │
│    两个 3 票集合必有交集（至少 1 个节点重叠）                  │
│    → 重叠节点上包含了旧 Leader 提交的事务                      │
│    → 新 Leader 从重叠节点同步到该事务                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

## 3.2 未提交的事务必须丢弃

```text
保障机制：
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  场景：旧 Leader 广播了 Proposal 但未收集到过半 ACK 就崩溃了  │
│                                                               │
│  → 部分 Follower 收到了该 Proposal（持久化到本地）            │
│  → 部分 Follower 没有收到                                     │
│  → 新 Leader 可能没有这个 Proposal                            │
│                                                               │
│  解决方案 — 截断（Truncation）：                               │
│  · 新 Leader 同步时告知 Follower 其 epoch 和最新 zxid        │
│  · Follower 对比自己的事务日志，丢弃任何不匹配的事务          │
│  · "幽灵" Proposal 被彻底清除                                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

## 3.3 顺序一致性保证

```text
顺序一致性保证：
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  1. 可靠提交：                                                │
│     如果一条消息被一个节点提交，它最终会被所有节点提交         │
│                                                               │
│  2. 全序交付：                                               │
│     如果消息 a 在消息 b 之前被提交，则所有节点上 a 都在 b 前  │
│                                                               │
│  3. 因果顺序：                                                │
│     如果消息 a 发生在消息 b 之前（happen-before），            │
│     则 a 在所有节点上的顺序都在 b 之前                        │
│                                                               │
│  4. 前缀属性：                                                │
│     如果节点应用了 tx(n)，则它一定已经应用了 tx(1)...tx(n-1)   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

# 4. epoch 机制

epoch（朝代/任期）是 ZAB 协议中至关重要的概念，用于标识 Leader 任期。

```text
epoch 的作用：
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│  zxid = <epoch (32bit)> + <counter (32bit)>                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  epoch = 0x00000001        counter = 0x00000001           │  │
│  │  表示：第 1 任 Leader 的第 1 个事务                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  每次 Leader 更替，epoch 必须递增：                             │
│    旧 Leader (epoch=1) ──崩溃──► 新 Leader (epoch=2)           │
│                                                                 │
│  epoch 的三重保障：                                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  1. currentEpoch：当前 epoch 值（持久化到磁盘）            │  │
│  │  2. acceptedEpoch：Follower 接收到的 Leader 的 epoch       │  │
│  │  3. 新 Leader 选举时将 currentEpoch + 1 作为新 epoch      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  epoch 变化时的处理：                                           │
│  · Follower 接收到更高 epoch 的 Leader 消息时：                │
│    → 自动更新自己的 currentEpoch                               │
│    → 拒绝任何来自旧 epoch Leader 的消息                        │
│                                                                 │
│  · Leader 发起 Proposal 时会携带 epoch 信息：                  │
│    → Follower 检查 epoch 是否匹配，不匹配则拒绝                │
│    → 防止旧 Leader（网络分区恢复后）继续发起 Proposal 导致脑裂  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

# 5. ZAB 与 Paxos / Raft 对比

| 维度 | ZAB | Paxos | Raft |
|------|-----|-------|------|
| **设计目标** | 主备系统原子广播 | 分布式共识通用算法 | 分布式共识（可理解性优先） |
| **领导者** | 固定 Leader | 多 Proposer（可能冲突） | 固定 Leader |
| **两阶段** | Proposal + Commit | Prepare + Accept | AppendEntries |
| **epoch 机制** | 有（epoch 递增） | 有（ballot number） | 有（term） |
| **Membership 变更** | 需要重启（3.5+ 支持动态配置） | 理论上支持 | 支持 Joint Consensus |
| **日志提交** | 过半 ACK 后 Leader 提交 | 多数派接受即达成 | Leader 提交 + 间接提交 |
| **学习难度** | 中等 | 极难 | 相对容易 |
| **实际使用** | ZooKeeper | Google Chubby、论文 | etcd、Consul、TiKV |
| **论文** | 《Zab: High-performance broadcast》 | 《Paxos Made Simple》 | 《In Search of an Understandable Consensus Algorithm》 |

# 6. ZAB 协议状态机

ZooKeeper 服务器在以下状态之间转换：

```text
          ┌──────────┐
          │  LOOKING  │  ← 初始状态 / Leader 故障后
          └─────┬─────┘
                │
        ┌───────┴────────┐
        ▼                ▼
  ┌──────────┐    ┌──────────────┐
  │ LEADING  │    │  FOLLOWING   │
  │          │    │              │
  │ 收到过半  │    │  接收 Leader  │
  │ 投票后    │    │  消息        │
  └─────┬────┘    └──────┬───────┘
        │                │
        │    Leader 失去过半支持   │
        │                │
        └───────┬────────┘
                │
                ▼
          ┌──────────┐
          │  LOOKING  │  ← 重新选举
          └──────────┘
```

**状态说明**：

| 状态 | 说明 | 行为 |
|------|------|------|
| `LOOKING` | 正在寻找 Leader | 发起/参与选举投票 |
| `LEADING` | 当前节点是 Leader | 处理写请求、发起 Proposal、管理 Follower |
| `FOLLOWING` | 当前节点是 Follower | 转发写请求、处理读请求、ACK Proposal |
| `OBSERVING` | 当前节点是 Observer | 处理读请求、不参与投票 |

# 7. 消息广播的详细流程

## 7.1 Leader 端处理

```java
// 伪代码：Leader 处理写请求的核心流程
class Leader {
    void processWriteRequest(Request request) {
        // 1. 生成 zxid（epoch + counter）
        long zxid = getNextZxid();

        // 2. 创建 Proposal
        Proposal proposal = new Proposal();
        proposal.packet = request;   // 请求数据
        proposal.zxid = zxid;

        // 3. 记录到 outstandingProposals
        outstandingProposals.put(zxid, proposal);

        // 4. 广播 Proposal 给所有 Follower
        for (LearnerHandler follower : followers) {
            follower.queuePacket(proposal);
        }

        // 5. （异步）等待 ACK
        //    当收到过半 ACK 时，回调 processAck()
    }

    void processAck(long zxid) {
        Proposal proposal = outstandingProposals.get(zxid);

        // 同步等待过半 ACK
        if (proposal.tryToCommit()) {
            // 6. Leader 先本地提交
            zk.getZKDatabase().processTxn(
                proposal.packet.getTxnHeader(),
                proposal.packet.getTxn()
            );

            // 7. 广播 COMMIT
            for (LearnerHandler follower : followers) {
                follower.queuePacket(new Commit(zxid));
            }

            // 8. 处理下一个等待提交的 Proposal
            outstandingProposals.remove(zxid);
        }
    }
}
```

## 7.2 Follower 端处理

```java
// 伪代码：Follower 处理 Leader 消息的核心流程
class Follower {
    void processPacket(QuorumPacket packet) {
        switch (packet.getType()) {
            case PROPOSAL:
                // 1. 持久化到事务日志
                zk.getZKDatabase().append(packet);

                // 2. 记录到 pendingTxns
                pendingTxns.add(packet.getZxid());

                // 3. 发送 ACK
                sendAck(packet.getZxid());
                break;

            case COMMIT:
                // 4. 将事务应用到内存数据库
                zk.getZKDatabase().commit(packet.getZxid());

                // 5. 从 pendingTxns 移除
                pendingTxns.remove(packet.getZxid());
                break;
        }
    }
}
```

# 8. 崩溃恢复中的同步策略

新 Leader 选出后，需要与 Follower 同步数据。ZooKeeper 支持三种同步策略：

```text
同步策略选择逻辑（Leader 端）：

┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Leader 获取 Follower 的 lastZxid（最后提交的 zxid）       │
│                                                           │
│  比较 minCommittedLog 和 maxCommittedLog：                │
│                                                           │
│  if (follower.lastZxid > leader.maxCommittedLog) {        │
│      // TRUNC：Follower 领先，需要截断                    │
│      → 发送 TRUNC 指令，让 Follower 回滚                  │
│  }                                                        │
│                                                           │
│  if (follower.lastZxid < leader.minCommittedLog) {        │
│      // SNAP：Follower 太落后，全量同步                   │
│      → 发送 SNAP（序列化当前 DataTree）                    │
│  }                                                        │
│                                                           │
│  else {                                                   │
│      // DIFF：Follower 差距不大，增量同步                  │
│      → 发送 DIFF（仅发送差异的事务日志条目）               │
│  }                                                        │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**三种同步方式对比**：

| 方式 | 触发条件 | 传输内容 | 带宽消耗 | 速度 |
|------|---------|---------|---------|------|
| **DIFF** | Follower 仅落后少量事务 | 差异的事务日志条目 | 低 | 快 |
| **SNAP** | Follower 落后太多 | 完整 DataTree 快照 | 高 | 慢 |
| **TRUNC** | Follower 包含了未提交的事务 | 截断指令 | 极低 | 快 |

# 9. ZAB 协议的配置关联

```properties
# zoo.cfg 中与 ZAB 相关的核心参数

# 基本时间单位（影响超时检测速度）
tickTime=2000

# Follower 连接 Leader 的超时
initLimit=10  # 10 * tickTime = 20s

# Follower 同步 Leader 的超时（同步阶段，非运行时）
syncLimit=5   # 5 * tickTime = 10s

# Leader 可同时处理的 outstanding 请求数
# 增大可提升吞吐，但会增加内存占用和恢复时间
globalOutstandingLimit=1000

# 快照触发阈值（事务数达到该值后生成快照）
# 影响恢复速度：小值 → 恢复快但 I/O 频繁；大值 → 恢复慢
snapCount=100000

# PreAlloc 大小（预分配事务日志文件大小，KB）
# 减少运行时动态扩展文件的开销
preAllocSize=65536  # 64MB

# 事务日志 fsync 策略（影响持久性 vs 性能）
# yes：每次事务都 fsync（最安全，最慢）
# no：让操作系统决定刷新时机（快但可能丢事务）
forceSync=yes
```

# 10. 脑裂与防护

## 10.1 什么是脑裂

脑裂是指集群被网络分区隔断后，两侧都选出了 Leader，导致数据不一致的严重问题。

```text
脑裂场景：
┌────────────────────────────────────────────────┐
│                                                 │
│  3 节点集群正常状态：                             │
│  ┌──────┐   ┌──────┐   ┌──────┐                │
│  │  A   │   │  B   │   │  C   │                │
│  │Leader│   │Foll  │   │Foll  │                │
│  └──────┘   └──────┘   └──────┘                │
│                                                 │
│  网络分区后（考虑不防护的情况）：                  │
│  ┌──────┐           ┌──────┐   ┌──────┐        │
│  │  A   │           │  B   │   │  C   │        │
│  │旧Ldr │   ✗       │Foll  │   │Foll  │        │
│  └──────┘           └──┬───┘   └──┬───┘        │
│                        │          │             │
│                    B 成为新 Leader  C 跟随 B     │
│                                                 │
│  结果：A 认为自己是 Leader，B 也认为自己是 Leader │
│  → 两个 Leader 同时写入 → 数据不一致！           │
└────────────────────────────────────────────────┘
```

## 10.2 过半机制防脑裂

```text
ZooKeeper 的防护机制：

┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. 过半机制（Quorum）                                          │
│     · 选举需要 N/2+1 票                                        │
│     · 提交事务需要 N/2+1 ACK                                   │
│     · 3节点集群分区 → 少数派（1节点）无法当选也无法提交        │
│                                                                 │
│  2. epoch 递增                                                  │
│     · 每次选新 Leader，epoch 递增                               │
│     · 旧 Leader 的 Proposal 会因 epoch 不匹配被拒绝            │
│                                                                 │
│  3. Follower 发现 Leader 失联后进入 LOOKING 状态               │
│     · 在超时后不再接受旧 Leader 的消息                          │
│                                                                 │
│  4. 网络分区恢复后的处理                                        │
│     · 旧 Leader（分区后失去过半支持）发现自己不再是 Leader       │
│     · 降级为 Follower 并同步新 Leader 的数据                    │
│     · 旧 Leader 上未被提交的事务被截断                          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

典型的 ZooKeeper 集群节点数选择：
┌──────────────────────────────────────────────────────────────────┐
│  · 3 节点：容忍 1 个故障（最常用）                                   │
│  · 5 节点：容忍 2 个故障                                          │
│  · 7 节点：容忍 3 个故障（写性能显著下降）                           │
│  · 奇数节点：因为偶数节点不能提升容错能力，反而增加投票开销           │
│    例如：4 节点容忍 1 个故障（强一致需要 3 票），与 3 节点相同       │
└──────────────────────────────────────────────────────────────────┘
```
