
# 1. Leader 选举概述

ZooKeeper 的 Leader 选举是其一致性的核心保障。当集群启动或当前 Leader 宕机时，所有节点通过**快速 Leader 选举（Fast Leader Election）**算法选出一个新的 Leader。

```text
选举触发时机：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  1. 集群初始化启动（所有节点处于 LOOKING 状态）        │
│  2. Leader 宕机或网络分区（Follower 失去 Leader）      │
│  3. Leader 主动退位（如动态重配置时）                  │
│                                                       │
│  选举前提：节点进入 LOOKING 状态                       │
│  选举完成条件：过半数节点对同一节点投票               │
│                                                       │
└──────────────────────────────────────────────────────┘
```

# 2. 选举算法发展史

| 版本 | 算法 | 特点 |
|------|------|------|
| 3.3.0 之前 | LeaderElection（UDP 广播） | 简单但效率低、易消息丢失 |
| 3.3.0 ~ 3.4.x | AuthFastLeaderElection | 基于 TCP 的改进版 |
| 3.4.0+ | **FastLeaderElection（FLE）** | 当前标准算法，基于 TCP |
| 3.6.0+ | FLE 持续优化 | 支持多地址绑定等 |

> **当前版本统一使用 FastLeaderElection**，以下内容围绕 FLE 展开。

# 3. FastLeaderElection 核心数据结构

## 3.1 Vote（投票）

```java
// org.apache.zookeeper.server.quorum.Vote
public class Vote {
    private final long id;       // 被推荐的 Leader 的 myid
    private final long zxid;     // 被推荐 Leader 的最新事务 ID
    private final long peerEpoch;// 被推荐 Leader 的 epoch
    private final long electionEpoch; // 当前选举轮次
    private final ServerState state;  // 投票者当前状态
}
```

## 3.2 Notification（通知消息）

```java
// 在选举网络中交换的消息格式
public class Notification {
    long leader;        // 推荐 Leader 的 myid
    long zxid;          // 推荐 Leader 的 zxid
    long electionEpoch; // 选举轮次
    long peerEpoch;     // 推荐 Leader 的 epoch
    ServerState state;  // 推荐者当前状态（LOOKING/FOLLOWING/LEADING）
    long sid;           // 发送者的 myid
}
```

## 3.3 选举相关配置

```text
选举的通信通道：
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  QuorumCnxManager（连接管理器）                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                                                      │ │
│  │  Listener：监听选举端口（默认 3888），接收其他节点    │ │
│  │  Sender：  主动向其他节点发送选举消息                  │ │
│  │                                                      │ │
│  │  连接策略：myid 小的主动连接 myid 大的                 │ │
│  │  目的：避免每个节点对之间出现两条重复连接              │ │
│  │                                                      │ │
│  │  示例（3 节点）：                                     │ │
│  │    Server1 (id=1) → 连接 → Server2 (id=2)            │ │
│  │    Server1 (id=1) → 连接 → Server3 (id=3)            │ │
│  │    Server2 (id=2) → 连接 → Server3 (id=3)            │ │
│  │                                                      │ │
│  │  结果：形成一个全连接的有向图                          │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  SendWorker / RecvWorker：每个连接的两个工作线程          │
│    · SendWorker：负责发送选票                            │
│    · RecvWorker：负责接收选票                            │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

# 4. 选举流程详解

## 4.1 整体流程

```text
快速 Leader 选举完整流程：

Step 1: 节点进入 LOOKING 状态
        │
Step 2: 投自己一票
        │  · vote(id=self, zxid=self.zxid, epoch=self.currentEpoch)
        │
Step 3: 将自己的投票发送给集群中所有其他节点
        │  · 通过 QuorumCnxManager 发送 Notification
        │
Step 4: 收集其他节点的投票
        │  · 循环 receive 其他节点的 Notification
        │
Step 5: 比较收到的投票和自己的投票
        │  · 优先比较 electionEpoch（轮次）
        │  · 再比较 zxid（数据新旧）
        │  · 最后比较 myid（打破平局）
        │
Step 6: 更新自己的投票为"更好的"候选人
        │  · 重新发送更新后的投票
        │
Step 7: 统计投票
        │  · 如果某个候选人获得过半票数 → 该候选人成为 Leader
        │  · 自己的票数过半 → 自己成为 Leader
        │  · 别人的票数过半 → 跟随该候选人
        │
Step 8: 退出 LOOKING 状态
        │  · 进入 LEADING 或 FOLLOWING 状态
```

## 4.2 投票比较规则

```text
选举 PK 规则（用于比较两个候选者谁更优）：

compareVote(vote1, vote2):
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  // 第一优先级：electionEpoch（选举轮次）                      │
│  if (vote1.electionEpoch > vote2.electionEpoch)                │
│      → vote1 更好                                            │
│                                                               │
│  // 第二优先级：zxid（最大事务 ID，反映数据新旧程度）          │
│  else if (vote1.electionEpoch == vote2.electionEpoch) {       │
│      if (vote1.zxid > vote2.zxid)                             │
│          → vote1 更好                                         │
│  }                                                            │
│                                                               │
│  // 第三优先级：myid（节点 ID，打破平局）                      │
│  else if (vote1.zxid == vote2.zxid) {                        │
│      if (vote1.id > vote2.id)                                 │
│          → vote1 更好                                         │
│  }                                                            │
│                                                               │
│  总结：electionEpoch > zxid > myid                             │
│  确保选出数据最新（zxid 最大）的节点作为 Leader                │
└──────────────────────────────────────────────────────────────┘
```

## 4.3 选举收敛过程示例

```text
示例：3 节点集群启动选举过程

初始状态（启动后各自投自己）：
┌────────────┐    ┌────────────┐    ┌────────────┐
│ Server 1   │    │ Server 2   │    │ Server 3   │
│ myid=1     │    │ myid=2     │    │ myid=3     │
│ zxid=100   │    │ zxid=200   │    │ zxid=50    │
│ vote:1     │    │ vote:2     │    │ vote:3     │
└────────────┘    └────────────┘    └────────────┘

Round 1: 交换投票
┌─────────────────────────────────────────────────────────┐
│  Server1 收到 Server2 的投票：vote(2, zxid=200)          │
│    PK：200 > 100 → Server1 改投 Server2                 │
│  Server1 收到 Server3 的投票：vote(3, zxid=50)           │
│    PK：200 > 50 → Server1 坚持投 Server2                │
│                                                          │
│  Server2 收到 Server1 的投票：vote(1, zxid=100)          │
│    PK：200 > 100 → Server2 坚持投自己                    │
│  Server2 收到 Server3 的投票：vote(3, zxid=50)           │
│    PK：200 > 50 → Server2 坚持投自己                     │
│                                                          │
│  Server3 收到 Server1 的投票：vote(1, zxid=100)          │
│    PK：100 > 50 → Server3 改投 Server1                  │
│  Server3 收到 Server2 的投票：vote(2, zxid=200)          │
│    PK：200 > 100 → Server3 改投 Server2                  │
└─────────────────────────────────────────────────────────┘

Round 2: 再次发送更新后的投票
┌─────────────────────────────────────────────────────────┐
│  Server1 发送：vote(2, zxid=200)                         │
│  Server2 发送：vote(2, zxid=200)                         │
│  Server3 发送：vote(2, zxid=200)                         │
│                                                          │
│  统计：Server2 获得 3 票（超过半数 3/2+1=2）             │
│  → Server2 当选为 Leader ✓                               │
└─────────────────────────────────────────────────────────┘
```

# 5. 选举中的特殊场景

## 5.1 Leader 宕机后重新选举

```text
场景说明：
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  集群正常运行：Server2 是 Leader，Server1/3 是 Follower   │
│  Server2 突然宕机                                         │
│                                                           │
│  触发机制：                                               │
│    · Server1 和 Server3 发现与 Leader 失联                │
│    · 在 syncLimit * tickTime 时间内未收到 Leader 心跳     │
│    · 进入 LOOKING 状态                                    │
│                                                           │
│  选举流程：                                               │
│    · Server1：zxid=230（最新） → 自荐                    │
│    · Server3：zxid=150（较旧） → 自荐                     │
│    · PK 结果：230 > 150 → Server1 当选为 Leader          │
│                                                           │
│  关键保证：                                               │
│    · Server2 可能在宕机前提交了 zxid=240 的事务           │
│    · Server1 的 zxid=230，包含了 zxid=240 吗？            │
│    · 是的！过半提交意味着至少有一个 Follower 收到了      │
│    · 该 Follower 参与选举，新 Leader 必然拥有该事务       │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

## 5.2 网络分区下的选举

```text
3 节点集群发生网络分区：

情况 1：少数派分区（1 个节点被隔离）
┌──────────────────────────────────────────────────────┐
│  ┌──────────┐           ┌──────────┐  ┌──────────┐  │
│  │ Server 1 │           │ Server 2 │  │ Server 3 │  │
│  │  Follower│     ✗     │  Leader  │  │  Follower│  │
│  └──────────┘           └──────────┘  └──────────┘  │
│                                                       │
│  Server1 进入 LOOKING 状态                             │
│    → 期望获得 2 票                                      │
│    → 实际只有自己 1 票                                  │
│    → 永远无法当选                                       │
│    → 一直保持 LOOKING 状态直到网络恢复                  │
│                                                       │
│  Server2/Server3 保持多数派                             │
│    → 继续正常工作                                      │
└──────────────────────────────────────────────────────┘

情况 2：均分分区（可能出现脑裂）
┌──────────────────────────────────────────────────────┐
│  ┌──────────┐          ┌──────────┐  ┌──────────┐   │
│  │ Server 1 │          │ Server 2 │  │ Server 3 │   │
│  │ Leader   │     ✗    │          │  │          │   │
│  └──────────┘          └──────────┘  └──────────┘   │
│                                                       │
│  Server1 独自一个分区                                  │
│    → 失去过半 Follower → 退位为 LOOKING               │
│    → 无法获得过半选票 → 无法当选                       │
│                                                       │
│  Server2/Server3 形成新分区                            │
│    → 发现 Leader 失联 → 进入 LOOKING                  │
│    → 二者互投 → 获得 2 票 → 选出新 Leader             │
│                                                       │
│  关键：Server1 失去过半支持后主动退位                   │
│        两个分区都不能接受写请求（Server1 退位）        │
│        避免了脑裂                                       │
└──────────────────────────────────────────────────────┘
```

## 5.3 新节点加入后的选举

```text
场景：3 节点集群正常运行，加入第 4 个节点

错误的认知：新节点（zxid=0）会竞争 Leader
实际行为：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  新节点 Server4 (myid=4, zxid=0) 启动后：             │
│    · 进入 LOOKING 状态发送投票 vote(self, zxid=0)     │
│    · 收到现有节点的投票（zxid 很大）                   │
│    · PK 结果：大的 zxid > 0                           │
│    · 改为投现有 Leader                                │
│    · 最终以 Follower 身份加入（或 Observer）          │
│                                                       │
│  Leader 不会改变，集群正常运行                           │
└──────────────────────────────────────────────────────┘
```

# 6. 源码核心流程分析

## 6.1 FastLeaderElection 核心方法

```java
// org.apache.zookeeper.server.quorum.FastLeaderElection
public class FastLeaderElection implements Election {
    // 当前的投票（投给谁）
    private volatile Vote currentVote;

    // 收到的投票箱：key = 投票者 sid
    private final ConcurrentHashMap<Long, Vote> recvset;

    // 核心选举循环
    public Vote lookForLeader() throws InterruptedException {
        // 1. 更新选举轮次（logicalclock++）
        // 2. 投自己一票，并广播
        // 3. 循环接收其他节点的投票
        // 4. PK 比较，更新投票
        // 5. 检查是否有节点获得过半票数
        // 6. 如果自己的过半数 → return (自己作为 Leader)
        // 7. 如果别人的过半数 → return (关注别人作为 Follower)
    }

    // 判断是否有节点获得过半票数
    private boolean termPredicate(
        HashMap<Long, Vote> votes,
        Vote vote
    ) {
        // 统计投给 vote.id 的节点数
        count = 统计 vote.id 的数量
        return count > self.getVotingView().size() / 2;
    }

    // 投票比较
    protected boolean totalOrderPredicate(
        long newId, long newZxid, long newEpoch,
        long curId, long curZxid, long curEpoch
    ) {
        // 返回 true 表示 new 的候选者优于 cur 的候选者
        // 比较顺序：electionEpoch > zxid > myid
    }
}
```

## 6.2 QuorumPeer 状态机

```java
// org.apache.zookeeper.server.quorum.QuorumPeer（精简）
public class QuorumPeer extends ZooKeeperThread
    implements QuorumStats.Provider {

    // 服务器当前状态
    public enum ServerState {
        LOOKING, FOLLOWING, LEADING, OBSERVING
    }

    private ServerState state = ServerState.LOOKING;

    @Override
    public void run() {
        while (running) {
            switch (getPeerState()) {
                case LOOKING:
                    // 执行 Leader 选举
                    Vote current = electAlgo.lookForLeader();
                    if (current.getId() == getId()) {
                        setPeerState(ServerState.LEADING);
                    } else {
                        setPeerState(ServerState.FOLLOWING);
                    }
                    break;

                case LEADING:
                    // 执行 Leader 逻辑
                    leader.lead();
                    // 如果 lead() 返回，说明 Leader 退位
                    setPeerState(ServerState.LOOKING);
                    break;

                case FOLLOWING:
                    // 执行 Follower 逻辑
                    follower.followLeader();
                    // 如果 followLeader() 返回，说明失去 Leader
                    setPeerState(ServerState.LOOKING);
                    break;

                case OBSERVING:
                    observer.observeLeader();
                    setPeerState(ServerState.LOOKING);
                    break;
            }
        }
    }
}
```

# 7. Observer 的选举参与

Observer 在选举中的特殊行为：

```text
Observer 与 Leader 选举的关系：

┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Observer 参与了 Leader 选举的投票！                       │
│  （但不参与事务投票，这是两个不同的投票场景）               │
│                                                           │
│  为什么 Observer 要参与选举投票？                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                                                      │ │
│  │  1. 加快选举收敛：更多投票节点可以更快形成共识        │ │
│  │                                                      │ │
│  │  2. 信息传递：Observer 可能拥有最新的数据（zxid），  │ │
│  │     作为"最大的 zxid"的信息来源                      │ │
│  │                                                      │ │
│  │  3. 选票计数：选举投票不计入 Observer 的票数         │ │
│  │     （votingView 不包含 Observer）                   │ │
│  │    过半票数仅基于 Participant 节点计算               │ │
│  │                                                      │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
│  配置示例：                                               │
│    server.4=zk4.example.com:2888:3888:observer             │
│                                                           │
│  Observer 在选举中发送自己的投票给所有节点                 │
│  接收其他节点的投票并更新自己的投票                        │
│  但在统计"过半票数"时，Observer 的投票不计数               │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

# 8. 选举性能与优化

## 8.1 选举耗时分析

```text
选举耗时构成：
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  1. 检测 Leader 故障时间                                  │
│     · Follower 发现 Leader 失联：syncLimit * tickTime    │
│     · 默认：5 * 2000ms = 10s                             │
│                                                           │
│  2. 选举消息交换时间                                      │
│     · 网络延迟 × 消息轮次（通常 1~2 轮）                  │
│     · 通常：< 200ms                                       │
│                                                           │
│  3. 数据同步时间                                          │
│     · DIFF 模式：几秒（取决于差异事务数）                  │
│     · SNAP 模式：几十秒到几分钟（取决于数据量）            │
│                                                           │
│  优化方向：                                               │
│    · 减小 syncLimit 缩短故障检测时间（但可能误判）        │
│    · 使用 SSD 加快事务日志 I/O                            │
│    · 合理设置 snapCount 避免全量同步                      │
│    · 确保网络延迟低且稳定                                  │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

## 8.2 选举调优参数

```properties
# 影响选举速度和稳定性的配置

# 缩短故障检测时间（更激进地检测 Leader 故障）
# 优点：选举更快；缺点：网络抖动时可能误判
syncLimit=3      # 3 * 2000ms = 6s

# 基本 tick 时间（心跳间隔的最小单位）
# 减小 tickTime 可加快检测，但增加 CPU 开销
tickTime=2000

# Follower 初始连接超时（不影响运行时故障检测）
initLimit=10

# 动态重配置相关（ZooKeeper 3.5+）
reconfigEnabled=true
```

# 9. 选举相关故障排查

| 症状 | 可能原因 | 排查方法 |
|------|---------|---------|
| 集群一直处于选举状态 | 网络分区、防火墙、myid 冲突 | 检查选举端口（3888）连通性；检查 myid 文件 |
| 某个节点反复加入退出 | 磁盘 I/O 太慢、事务日志损坏 | 检查磁盘性能；清理损坏的事务日志 |
| Leader 频繁切换 | 网络不稳定、GC 导致心跳超时 | 检查网络延迟；优化 JVM GC 参数 |
| 新节点无法加入 | 配置不一致、版本不兼容 | 比对 zoo.cfg 配置；检查 ZK 版本 |
| 选举消息丢失 | UDP 被拦截（老版本） | 确认使用 TCP（FastLeaderElection 使用 TCP） |

# 10. 动态重配置（ZooKeeper 3.5+）

ZooKeeper 3.5 引入了动态重配置功能，允许在不重启集群的情况下变更集群成员：

```text
动态重配置支持的操作：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  · 新增节点（Follower/Observer）                      │
│  · 移除节点                                          │
│  · 替换节点（原子地进行 remove + add）               │
│                                                       │
│  实现方式：                                           │
│    1. 通过 ZooKeeper 自身的 ZNode 存储配置            │
│       /zookeeper/config                               │
│                                                       │
│    2. reconfig 命令（CLI 或 API）                     │
│       reconfig -add server.4=host:2888:3888:participant│
│       reconfig -remove 4                              │
│                                                       │
│    3. 新节点以 non-voting follower 身份加入           │
│       数据同步完成后自动转为 voting follower          │
│                                                       │
│  关键特性：                                           │
│    · 原子变更：配置变更是原子操作                     │
│    · 滚动升级：逐台重启支持                            │
│    · 动态扩展：无需计划停机                           │
│                                                       │
└──────────────────────────────────────────────────────┘
```
