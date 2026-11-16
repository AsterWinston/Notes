
# 1. 源码模块概览

ZooKeeper 源码采用 Maven 多模块结构。以下分析基于 3.7+ 版本的核心源码。

```text
ZooKeeper 核心包结构：

org.apache.zookeeper
  ├── server                        -- 服务端核心
  │   ├── quorum                    -- 集群相关（QuorumPeer, Election, ZAB）
  │   ├── ZooKeeperServer           -- 服务端主类
  │   ├── RequestProcessor          -- 请求处理链
  │   ├── DataTree                  -- 内存数据树
  │   ├── SessionTrackerImpl        -- Session 管理
  │   ├── NIOServerCnxnFactory      -- NIO 网络层
  │   ├── NettyServerCnxnFactory    -- Netty 网络层（3.6+）
  │   └── WatchManager              -- Watch 管理
  │
  ├── client                        -- 客户端
  │   ├── ZooKeeper                 -- 客户端主类
  │   ├── ClientCnxn                -- 连接管理
  │   └── ZKWatchManager            -- 客户端 Watch 管理
  │
  └── data                          -- 数据协议
      ├── Stat                      -- 状态信息
      └── ACL                       -- 权限
```

# 2. 服务端启动流程

```java
// 服务端启动主流程（QuorumPeerMain -> QuorumPeer.run()）

public class QuorumPeerMain {
    public static void main(String[] args) {
        // 1. 解析配置文件 zoo.cfg
        QuorumPeerConfig config = new QuorumPeerConfig();
        config.parse(args[0]);

        // 2. 创建并启动 QuorumPeer
        QuorumPeerMain main = new QuorumPeerMain();
        main.runFromConfig(config);
    }

    void runFromConfig(QuorumPeerConfig config) {
        // 3. 创建 ServerCnxnFactory（NIO 或 Netty）
        ServerCnxnFactory cnxnFactory =
            ServerCnxnFactory.createFactory();
        cnxnFactory.configure(config.getClientPortAddress(),
            config.getMaxClientCnxns());

        // 4. 创建 QuorumPeer 线程
        QuorumPeer quorumPeer = new QuorumPeer();
        quorumPeer.setClientPortAddress(...);
        // ...

        // 5. 启动 QuorumPeer（进入状态机循环）
        quorumPeer.start();
        cnxnFactory.start();
    }
}
```

```text
QuorumPeer 状态机循环：

QuorumPeer.run():
  while (running) {
    switch (getPeerState()) {
      case LOOKING:
        // 查找 Leader（FastLeaderElection.lookForLeader()）
        Vote vote = electAlgo.lookForLeader();
        if (vote == self) -> setPeerState(LEADING)
        else -> setPeerState(FOLLOWING / OBSERVING)
        break;

      case LEADING:
        // 成为 Leader
        leader = new Leader(this);
        leader.lead();  // 阻塞直到 Leader 退位
        setPeerState(LOOKING);
        break;

      case FOLLOWING:
        // 成为 Follower
        follower = new Follower(this);
        follower.followLeader();  // 阻塞直到失去 Leader
        setPeerState(LOOKING);
        break;

      case OBSERVING:
        observer = new Observer(this);
        observer.observeLeader();
        setPeerState(LOOKING);
        break;
    }
  }
```

# 3. 请求处理链（Request Processor Pipeline）

请求处理链是 ZooKeeper 数据处理的核心，不同角色使用不同的处理链：

## 3.1 Leader 处理链

```java
// Leader.setupRequestProcessors()
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);

    // 提交处理器（等待过半ACK）
    RequestProcessor commitProcessor = new CommitProcessor(
        finalProcessor, nextProcessor);

    // 提案处理器（ZAB协议的核心，广播Proposal给Follower）
    RequestProcessor proposalProcessor = new ProposalRequestProcessor(
        this, commitProcessor);

    // 预处理器（生成zxid，设置session信息）
    RequestProcessor prepProcessor = new PrepRequestProcessor(
        this, proposalProcessor);

    // 设置处理链
    // prepProcessor -> proposalProcessor -> commitProcessor -> finalProcessor
    firstProcessor = new LeaderRequestProcessor(prepProcessor);
}
```

```text
Leader 请求处理流程：
+----------------------------------------------------------------------+
|                                                                       |
|  Client Request                                                       |
|       |                                                               |
|       v                                                               |
|  PrepRequestProcessor                                                 |
|    · 生成 zxid（epoch + counter）                                     |
|    · 关联 Session 信息                                                |
|    · 生成 TxnHeader                                                   |
|       |                                                               |
|       v                                                               |
|  ProposalRequestProcessor（ZAB 发送）                                 |
|    · 构造 Proposal                                                    |
|    · 持久化到本地事务日志                                              |
|    · 广播 Proposal 给所有 Follower                                     |
|       |                                                               |
|       v                                                               |
|  CommitProcessor                                                      |
|    · 等待过半 Follower 的 ACK                                         |
|    · 收集到过半 ACK 后才放行到下一个处理器                              |
|    · 对于读请求直接放行                                                |
|       |                                                               |
|       v                                                               |
|  FinalRequestProcessor                                                |
|    · 应用变更到 DataTree（内存数据库）                                 |
|    · 触发 Watch                                                        |
|    · 构造并返回 Response                                               |
|                                                                       |
+----------------------------------------------------------------------+
```

## 3.2 Follower 处理链

```java
// Follower.setupRequestProcessors()
protected void setupRequestProcessors() {
    RequestProcessor finalProcessor = new FinalRequestProcessor(this);

    RequestProcessor commitProcessor = new CommitProcessor(
        finalProcessor, id);

    // FollowerRequestProcessor：判断请求类型
    //   读请求 -> 直接发给 commitProcessor
    //   写请求 -> 转发给 Leader
    RequestProcessor followerProcessor = new FollowerRequestProcessor(
        this, commitProcessor);

    firstProcessor = followerProcessor;
}
```

# 4. ZAB 协议核心实现

## 4.1 Leader.lead() — Leader 主循环

```java
// Leader.lead() 核心流程（简化）
void lead() throws Exception {
    // 1. 等待过半数 Follower 连接并同步完成
    waitForEpochAck(self.getAcceptedEpoch());

    // 2. 进入消息广播循环
    while (true) {
        synchronized (this) {
            long start = Time.currentElapsedTime();
            long cur = start;
            long end = start + self.tickTime / 2;

            while (cur < end) {
                // 等待新请求到达
                wait(end - cur);
                cur = Time.currentElapsedTime();
            }

            // 检查是否需要发心跳
            if (!shouldPing()) continue;

            // 发送心跳 PING 给所有 Learner（Follower/Observer）
            for (LearnerHandler handler : learners) {
                handler.ping();
            }
        }
    }
}

// Leader.processAck() — 处理 Follower 的 ACK
public synchronized void processAck(long sid, long zxid,
    SocketAddress followerAddr) {
    Proposal proposal = outstandingProposals.get(zxid);
    if (proposal != null) {
        // 记录该 Follower 的 ACK
        proposal.addAck(sid);

        // 检查是否获得过半 ACK
        if (proposal.hasAllQuorums()) {
            // Leader 先本地提交
            zk.getZKDatabase().processTxn(
                proposal.getHeader(),
                proposal.getTxn()
            );

            // 广播 COMMIT 给所有 Follower
            for (LearnerHandler f : followers) {
                f.queuePacket(new QuorumPacket(
                    Leader.COMMIT, zxid, null, null));
            }
            outstandingProposals.remove(zxid);
        }
    }
}
```

## 4.2 LearnerHandler — Follower 通信层

```java
// Leader 为每个 Follower 创建一个 LearnerHandler 线程
class LearnerHandler extends ZooKeeperThread {
    void run() {
        // 1. 接收 Follower 的 FOLLOWERINFO / OBSERVERINFO
        // 2. 发送 LEADERINFO（epoch 信息）
        // 3. 接收 ACKEPOCH（确认 epoch）
        // 4. 确定同步策略（DIFF / SNAP / TRUNC）
        // 5. 发送同步数据
        // 6. 进入主循环
        while (true) {
            QuorumPacket packet = new QuorumPacket();
            ia.readRecord(packet, "packet");

            switch (packet.getType()) {
                case REQUEST:
                    // Follower 转发的客户端写请求
                    Leader.this.propose(packet);
                    break;
                case ACK:
                    // Follower 的 ACK
                    Leader.this.processAck(sid, packet.getZxid());
                    break;
                case PING:
                    // Follower 的心跳
                    break;
            }
        }
    }
}
```

# 5. 内存数据库 DataTree

```java
// org.apache.zookeeper.server.DataTree — 核心内存数据结构
public class DataTree {
    // ZNode 存储：路径 -> DataNode
    private final ConcurrentHashMap<String, DataNode> nodes =
        new ConcurrentHashMap<>();

    // Watch 管理
    private final WatchManager dataWatches;   // 数据变化 Watch
    private final WatchManager childWatches;  // 子节点变化 Watch

    // 临时节点映射：sessionId -> 临时节点路径集合
    private final Map<Long, HashSet<String>> ephemerals =
        new ConcurrentHashMap<>();

    // 处理写事务
    public ProcessTxnResult processTxn(TxnHeader header, Record txn) {
        switch (header.getType()) {
            case OpCode.create:
                return createNode(header, txn);
            case OpCode.delete:
                return deleteNode(header, txn);
            case OpCode.setData:
                return setData(header, txn);
            // ...
        }
    }

    // 创建节点核心逻辑
    public ProcessTxnResult createNode(TxnHeader header,
                                       CreateTxn createTxn) {
        String path = createTxn.getPath();
        String parentPath = path.substring(0, path.lastIndexOf('/'));

        // 1. 检查父节点是否存在
        DataNode parent = nodes.get(parentPath);
        if (parent == null) throw new NoNodeException();

        // 2. 检查节点是否已存在
        if (nodes.containsKey(path)) throw new NodeExistsException();

        // 3. 创建 DataNode
        DataNode node = new DataNode();
        node.data = createTxn.getData();
        node.stat = new StatPersisted();
        // ... 设置各种 stat 属性
        nodes.put(path, node);

        // 4. 更新父节点的子节点列表
        parent.addChild(pathName);

        // 5. 如果是临时节点，记录到 ephemerals
        if (isEphemeral(createTxn.getEphemeralOwner())) {
            ephemerals.computeIfAbsent(
                createTxn.getEphemeralOwner(),
                k -> new HashSet<>()
            ).add(path);
        }

        // 6. 触发 Watch
        dataWatches.triggerWatch(path, EventType.NodeCreated);
        childWatches.triggerWatch(parentPath,
            EventType.NodeChildrenChanged);

        return result;
    }
}
```

# 6. SessionTracker 实现

```java
// SessionTrackerImpl — Session 管理实现
public class SessionTrackerImpl implements SessionTracker {
    // Session 存储：sessionId -> SessionImpl
    private final ConcurrentHashMap<Long, SessionImpl> sessionsById;

    // 分桶过期存储：过期时间点 -> 该时间点过期的 Session 集合
    private final ConcurrentHashMap<Long, SessionSet> sessionSets;

    private long nextExpirationTime;

    // 每个 tickTime 调用一次
    @Override
    public synchronized void run() {
        long currentTime = Time.currentElapsedTime();

        // 处理已到达过期时间的桶
        while (nextExpirationTime <= currentTime) {
            SessionSet set = sessionSets.remove(nextExpirationTime);
            if (set != null) {
                for (SessionImpl s : set.sessions) {
                    // 1. 设置 session 为 closing
                    s.isClosing = true;

                    // 2. 清理该 session 的所有临时节点
                    Set<String> ephemerals = zkDb.getDataTree()
                        .getEphemerals(s.sessionId);
                    for (String path : ephemerals) {
                        zkDb.getDataTree().deleteNode(path,
                            s.sessionId);
                    }

                    // 3. 从 session 表中移除
                    sessionsById.remove(s.sessionId);

                    // 4. 通知客户端并移除 Watch
                    // ...
                }
            }
            nextExpirationTime += expirationInterval;
        }
    }

    // 收到心跳时调用
    @Override
    public synchronized boolean touchSession(
        long sessionId, int timeout) {
        SessionImpl s = sessionsById.get(sessionId);
        if (s == null) return false;

        // 从旧的过期桶中移除
        SessionSet oldSet = sessionSets.get(s.expirationTime);
        if (oldSet != null) oldSet.sessions.remove(s);

        // 重新计算过期时间
        s.expirationTime = nextExpirationTime + timeout;
        // 放入新的过期桶
        sessionSets.computeIfAbsent(s.expirationTime,
            k -> new SessionSet()).sessions.add(s);

        return true;
    }
}
```

# 7. 网络通信层

## 7.1 NIOServerCnxnFactory

```java
// NIO 网络模型的核心组件
public class NIOServerCnxnFactory extends ServerCnxnFactory {
    // Accept 线程：接收新连接
    private AcceptThread acceptThread;

    // Selector 线程池：处理 I/O
    private SelectorThread[] selectorThreads;

    // 启动工厂
    @Override
    public void start() {
        // 1. 创建 SelectorThread 数组
        for (int i = 0; i < numSelectorThreads; i++) {
            selectorThreads[i] = new SelectorThread(i);
        }

        // 2. 启动 AcceptThread
        acceptThread = new AcceptThread();
        acceptThread.start();

        // 3. 配置完成
    }

    // AcceptThread — 接收连接
    class AcceptThread extends Thread {
        public void run() {
            while (running) {
                SocketChannel sc = serverSocket.accept();
                sc.configureBlocking(false);

                // 轮询分配给某个 SelectorThread
                SelectorThread selector = selectorThreads[
                    roundRobinIndex++ % selectorThreads.length ];
                selector.addAcceptedConnection(sc);
            }
        }
    }

    // SelectorThread — 处理 I/O
    class SelectorThread extends Thread {
        private final Selector selector;

        public void run() {
            while (running) {
                selector.select(1000);
                for (SelectionKey key : selector.selectedKeys()) {
                    if (key.isAcceptable()) {
                        // 处理新连接
                        doAccept(key);
                    } else if (key.isReadable()) {
                        // 处理读事件
                        IOWorkRequest work = doRead(key);
                        // 提交给 WorkerService 处理请求
                        workerService.schedule(work);
                    }
                }
            }
        }
    }
}
```

## 7.2 NettyServerCnxnFactory（3.6+）

```java
// Netty 网络模型（3.6+ 推荐）
public class NettyServerCnxnFactory extends ServerCnxnFactory {
    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;

    @Override
    public void start() {
        bossGroup = new NioEventLoopGroup(bossThreads);
        workerGroup = new NioEventLoopGroup(workerThreads);

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    ChannelPipeline pipeline = ch.pipeline();
                    // 添加编解码器
                    pipeline.addLast("decoder", ...);
                    pipeline.addLast("encoder", ...);
                    // 添加业务处理器
                    pipeline.addLast("handler",
                        new NettyServerCnxn(ch, zkServer));
                }
            });

        ChannelFuture future = bootstrap.bind(clientPort);
        future.sync();
    }
}
```

# 8. 选举核心实现

```java
// FastLeaderElection — 快速 Leader 选举核心（已在 6_leader_election.md 详细描述）
// 此处补充关键数据结构

public class FastLeaderElection implements Election {
    // 当前节点的投票
    private volatile Vote currentVote;

    // 收到的投票箱
    private final ConcurrentHashMap<Long, Vote> recvset;

    // 选举连接管理器
    private QuorumCnxManager manager;

    // 选举主循环
    public Vote lookForLeader() throws InterruptedException {
        // 1. 初始化选举逻辑时钟
        logicalclock.incrementAndGet();

        // 2. 投票给自己
        updateProposal(getInitId(), getInitLastLoggedZxid(),
            getPeerEpoch());

        // 3. 发送 Notification 给所有其他节点
        sendNotifications();

        // 4. 循环接收投票
        while (getPeerState() == LOOKING && !stop) {
            Notification n = recvqueue.poll(timeout, MILLISECONDS);

            if (n == null) {
                // 超时，重新发送 Notification
                sendNotifications();
                continue;
            }

            // 5. 判断投票者状态
            switch (n.state) {
                case LOOKING:
                    // PK 比较投票
                    if (totalOrderPredicate(n.leader, n.zxid, ...)) {
                        updateProposal(n.leader, n.zxid, ...);
                        sendNotifications();
                    }
                    break;
                case FOLLOWING:
                case LEADING:
                    // 投票者认为自己是 Follower/Leader
                    // 接收该投票并计数
                    break;
            }

            // 6. 检查是否自己的票数过半
            if (haveEnoughVotes(votes, currentVote)) {
                // 等待一段时间确认无更高 zxid 的节点
                while (true) {
                    Notification n2 = recvqueue.poll(
                        finalizeWait, MILLISECONDS);
                    if (n2 == null) break;
                    if (totalOrderPredicate(n2.leader, n2.zxid, ...)) {
                        // 还有更好的，继续等待
                        updateProposal(n2.leader, ...);
                        continue;
                    }
                }
                // 确认当选
                return currentVote;
            }
        }
    }
}
```

# 9. 源码阅读建议

```text
ZooKeeper 源码阅读路线：

1. 入口：QuorumPeerMain.main() -> QuorumPeer.run()
   理解整体状态机循环

2. 选举：FastLeaderElection.lookForLeader()
   理解选举流程和投票 PK 逻辑

3. ZAB 协议：Leader.lead() + ProposalRequestProcessor
   理解写请求如何被广播和提交

4. 请求处理：PrepRequestProcessor -> ProposalRequestProcessor
   -> CommitProcessor -> FinalRequestProcessor
   理解请求的处理链路

5. 数据存储：DataTree + FileTxnSnapLog
   理解内存数据库和持久化

6. 会话管理：SessionTrackerImpl
   理解分桶过期算法

7. 网络通信：NIOServerCnxnFactory / NettyServerCnxnFactory
   理解网络模型

8. Watch 机制：WatchManager
   理解事件通知
```
