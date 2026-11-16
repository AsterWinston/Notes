etcd 是云原生时代最常用的强一致键值存储，Kubernetes 的全部集群状态都放在它里面。很多人会用它存配置、做服务发现、实现分布式锁，却说不清它为什么能保证"读到最新值"，也说不清 Raft 到底在复制什么。本篇从定位讲到 Raft 的角色、选举、日志复制和线性一致性读，为后面的 API 与实战打底。

# 1. etcd 的定位与适用场景

## 1.1 etcd 是什么

etcd 是 CoreOS 发起的开源分布式键值存储，使用 Go 编写，当前主流版本是 3.5.x，客户端 API 为 v3。它对外提供 gRPC 接口，官方命令行工具是 `etcdctl`，Go 客户端是 `go.etcd.io/etcd/client/v3`。

它最常见的身份是 **Kubernetes 的唯一数据源**：Pod、Service、ConfigMap、Secret、Node 等对象最终都以 key-value 的形式写进 etcd。kube-apiserver 本身不持久化业务状态，它把 etcd 当成自己的存储层。

### 1.1.1 四个关键词

理解 etcd 可以先抓住四个关键词：

| 关键词 | 含义 | 带来的能力 |
|--------|------|------------|
| 键值存储 | 扁平 key 空间，value 是字节数组 | 模型简单，适合存小对象 |
| 强一致 | 写成功即多数派持久化，读默认线性一致 | 不会读到旧值 |
| Watch | 客户端订阅 key 变化，服务端推送事件 | 替代轮询，支撑 List-Watch |
| Lease | 给 key 绑定带 TTL 的租约 | 自动过期，支撑服务注册与锁 |

★ **关键认知**：etcd 不是"带 Watch 的 Redis"。它的第一目标是**一致性**，不是吞吐。所有写都要经过 Raft 多数派确认并落盘，因此单次写入通常是毫秒级，适合存元数据，不适合存大 value 或高频计数器。

## 1.2 与 ZooKeeper、Consul 的对比

三者都能做配置中心和服务发现，但定位与模型差别很大：

| 维度 | etcd | ZooKeeper | Consul |
|------|------|-----------|--------|
| 一致性协议 | Raft | ZAB | Raft |
| 数据模型 | 扁平 key + 范围查询 | 树形 znode | KV + 服务目录 |
| 临时节点 | Lease 租约 | 会话临时节点 | 健康检查 + TTL |
| Watch | 按 key/前缀订阅，带 revision | 一次性 watcher，需重新注册 | blocking query |
| API | gRPC/HTTP JSON，v3 | 自有二进制协议 | HTTP/DNS |
| 典型场景 | Kubernetes、元数据、选主 | Hadoop、Kafka 旧版、传统注册中心 | 多数据中心服务发现 |
| 运维复杂度 | 中等 | 较高 | 中等，内置服务网格能力 |

★ **关键认知**：etcd 的 Watch 是**长期流**，并且事件带全局 revision，断线后可以从指定 revision 续订；ZooKeeper 的 watcher 是一次性的，触发后必须重新注册，容易在"重新注册"的窗口里丢事件。这是 Kubernetes 选择 etcd 的重要原因。

## 1.3 什么时候不该用 etcd

| 场景 | 为什么不适合 | 更合适的方案 |
|------|--------------|--------------|
| 缓存、会话 | 写放大、容量小、成本高 | Redis |
| 大文件、日志 | value 建议小于 1.5 MB，库容量有限 | 对象存储、消息队列 |
| 高频计数 | 每次写都要 Raft 复制 | Redis、数据库 |
| 复杂查询 | 只有 key 范围和少量过滤 | MySQL、PostgreSQL |
| 临时任务队列 | 缺少消费组、确认机制 | Kafka、RabbitMQ |

etcd 的默认存储配额是 2 GB，生产上通常调到 4-8 GB 就应止步。它的数据要经过 WAL、BoltDB 和内存索引，库越大，启动、快照和恢复越慢。

# 2. 数据模型：键空间、Revision 与 MVCC

## 2.1 扁平键空间

etcd 没有目录，只有字符串 key。所谓"层级"只是用 `/` 分隔的命名约定：

```text
/services/order/10.0.0.1:8080
/services/user/10.0.0.2:8080
/config/order/timeout
/locks/order/1001
```

前缀查询就是把这些有共同前缀的 key 一次取出来。key 建议统一加前导 `/`，避免出现 `service` 和 `services` 这种边界歧义。

## 2.2 Revision

每次写事务成功，etcd 会把全局 revision 加一。一个 key 至少有三个与版本相关的字段：

| 字段 | 含义 | 用途 |
|------|------|------|
| `CreateRevision` | 该 key 第一次创建时的 revision | 判断 key 是否曾存在、实现公平排队 |
| `ModRevision` | 该 key 最近一次修改的 revision | CAS、乐观并发控制 |
| `Version` | 该 key 被修改的次数，从 1 开始 | 判断是否被改过 |
| `Lease` | 绑定的租约 ID，0 表示没有 | 判断是否会自动过期 |

```go
resp, err := cli.Get(ctx, "/config/order/timeout")
if err != nil {
    return err
}
for _, kv := range resp.Kvs {
    // Revision 是本次读看到的全局版本，不是这个 key 的版本
    fmt.Printf("key=%s value=%s create=%d mod=%d version=%d\n",
        kv.Key, kv.Value, kv.CreateRevision, kv.ModRevision, kv.Version)
}
fmt.Println("读事务的全局 revision:", resp.Header.Revision)
```

★ **关键认知**：`resp.Header.Revision` 是这次读的**全局快照版本**。用 `Get` 拿到一个前缀后，再从这个 revision + 1 开始 Watch，就能保证"先列后听"不丢事件。这是服务发现和配置中心的标准写法。

### 2.2.1 删除也是写

删除 key 不是抹掉历史，而是写入一条 tombstone，revision 继续增加。因此删除后重建同名 key，`CreateRevision` 会变、`Version` 从 1 重新开始；旧 revision 仍能读到删除前的值，直到历史被压缩。Watch 的 DELETE 事件通常只有 key 和 revision，旧值要用 `WithPrevKV()` 获取。

## 2.3 MVCC 与压缩

etcd 用 MVCC（多版本并发控制）保存 key 的历史版本。每次修改都追加一个新版本，旧版本保留，直到执行 compact。

```text
key: /config/order/timeout
  rev 100  value=3s      ← 旧版本
  rev 120  value=5s      ← 旧版本
  rev 150  value=8s      ← 当前版本
```

压缩（compact）会丢弃指定 revision 之前的历史，但不会立刻释放磁盘空间：BoltDB 文件里会留下空洞，需要 defrag 才能真正回收。这部分在运维篇展开。

★ **关键认知**：**压缩影响的是读历史与 Watch 起点，不影响当前值**。压缩之后再用旧 revision 读会得到 `ErrCompacted`，Watch 也会以 `CompactRevision` 取消。客户端必须处理这个错误，重新拉一次全量再续订。

# 3. Raft 共识算法

## 3.1 三种角色

Raft 把节点分成三种角色：

```text
┌──────────────────────────────────────────────┐
│                etcd 集群（3 节点）             │
│                                              │
│  ┌──────────┐   AppendEntries   ┌──────────┐ │
│  │  Leader  │ ────────────────► │ Follower │ │
│  │ 处理写    │                   │ 复制日志  │ │
│  │ 发起提交  │ ◄──────────────── │ 投票      │ │
│  └────┬─────┘      ACK          └──────────┘ │
│       │                                      │
│       │          ┌──────────┐                │
│       └────────► │ Follower │                │
│                  └──────────┘                │
│                                              │
│  Candidate：选举期间临时出现，拉票结束后消失    │
└──────────────────────────────────────────────┘
```

| 角色 | 职责 |
|------|------|
| Leader | 接收所有写请求，追加日志，向 Follower 复制，统计多数派后提交 |
| Follower | 接收 Leader 的日志与心跳，选举时投票；读请求也可以由它服务 |
| Candidate | 选举超时后自增 term，向其他节点拉票 |

正常情况下只有一个 Leader。客户端写请求发给任意节点时，Follower 会把请求转发给 Leader，或者返回 Leader 地址让客户端重试。

## 3.2 领导者选举

etcd 的默认参数是心跳 100 ms、选举超时 1000 ms。一个简化的选举过程：

```text
1. 所有节点初始都是 Follower，各自维护一个随机选举定时器
2. 某个 Follower 先超时，term + 1，变成 Candidate
3. 它向其他节点发送 RequestVote(term, lastLogIndex, lastLogTerm)
4. 其他节点在"没投过票 + 候选人日志不落后"时投赞成票
5. Candidate 拿到多数派选票，成为 Leader
6. Leader 立即发送心跳，压制其他节点的选举定时器
7. 没选上的 Candidate 退回 Follower，等下一轮随机超时
```

### 3.2.1 随机超时的作用

如果所有节点同时超时，就会同时拉票，反复平票。随机化选举超时（etcd 默认在 1000-2000 ms 之间波动）让大概率只有一个节点先醒来，减少选票分裂。

★ **关键认知**：选举能否成功，取决于**多数派是否可达**。3 节点集群挂掉 2 个，剩下的节点既选不出 Leader，也读不到最新值；5 节点集群可以容忍 2 个节点故障。这就是"奇数节点、多数派"的由来。

## 3.3 日志复制

写请求的完整路径如下：

```text
客户端
  │  Put("/a", "1")
  ▼
┌──────────┐
│  Leader  │
│ 1. 追加   │  本地日志: [ ... , (term=5, Put /a=1) ]  ← 未提交
└────┬─────┘
     │ 2. 并行 AppendEntries
     ├──────────────► Follower A：追加成功，返回 ACK
     └──────────────► Follower B：追加成功，返回 ACK
     ▼
3. 收到多数派 ACK（3 节点需要 2 个，含自己）
     ▼
4. 提交日志，应用到状态机（BoltDB），返回客户端成功
     ▼
5. 下一次心跳把 commitIndex 告诉 Follower，Follower 也应用
```

### 3.3.1 Quorum 与容错

| 集群规模 | 多数派 | 可容忍故障 | 说明 |
|----------|--------|------------|------|
| 1 | 1 | 0 | 仅开发测试 |
| 3 | 2 | 1 | 生产最小规模 |
| 5 | 3 | 2 | 推荐的生产规模 |
| 7 | 4 | 3 | 跨机房时考虑，延迟更高 |

偶数节点不会增加容错，只会增加成本。例如 4 节点的多数派仍是 3，和 3 节点一样只能挂 1 个，因此生产上一般用 3 或 5。已提交指日志已被多数派持久化，已应用指写入状态机；Follower 可能短暂落后，因此串行化读不保证最新。

# 4. 线性一致性读

## 4.1 两种读语义

etcd v3 支持两种读：

| 读模式 | 客户端写法 | 语义 | 代价 |
|--------|------------|------|------|
| 线性一致读（默认） | `Get` 不加选项 | 一定读到最新已提交值，等价于所有读写排成一条全序 | 需要与 Leader 确认多数派，延迟高 |
| 串行化读 | `clientv3.WithSerializable()` | 读本节点当前状态，可能是旧值 | 不走 Raft，延迟低，适合容忍陈旧的场景 |

```go
// 线性一致读：默认行为，最安全
resp, err := cli.Get(ctx, "/config/order/timeout")

// 串行化读：从本地状态机读，可能落后；适合监控、批量扫描
resp, err := cli.Get(ctx, "/config/order/timeout", clientv3.WithSerializable())
```

## 4.2 ReadIndex 的过程

线性一致读不是"Leader 直接读内存"那么简单，因为 Leader 可能已经被网络分区隔离，却还以为自己是 Leader。etcd 使用 ReadIndex 思路：

```text
1. 客户端把读请求发给 Leader（或由 Follower 转发）
2. Leader 记录当前 commitIndex，记为 readIndex
3. Leader 向所有 Follower 发一轮心跳，确认自己仍被多数派认可
4. 收到多数派心跳响应后，说明没有更新的 Leader 出现
5. Leader 等待本地状态机 apply 到 readIndex
6. 返回读结果
```

★ **关键认知**：线性一致读的关键不是"读 Leader 内存"，而是"先确认自己仍是合法 Leader，再等到状态机追平读点"。这就是为什么 etcd 的读也可能因为网络抖动而变慢。

## 4.3 写为什么更贵

一次写要经历：网络往返、Raft 日志追加、多数派 fsync、状态机 apply、返回响应。其中最慢的 Follower 决定多数派确认时间，磁盘 fsync 决定 WAL 追加时间。所以 etcd 的写入吞吐通常在每秒几千到几万次，远低于 Redis。把它当"强一致的小数据库"而不是"高速缓存"来用，架构才不会走偏。

# 5. Watch 与事件流

## 5.1 Watch 的推送模型

Watch 是 etcd 的另一个核心能力：客户端订阅 key 或前缀，服务端在变化时推送事件，而不是让客户端轮询。与轮询相比，它实时、低延迟、低 QPS；代价是连接要长期保持，并且必须处理断线和历史压缩。

## 5.2 Watch 与历史压缩

Watch 的起点是一个 revision。如果这个 revision 已经被压缩，服务端无法回放中间事件，只能返回 `required revision has been compacted`：

```text
客户端断线在 rev=100 → 服务端已 compact 到 rev=200
Watch(rev=101) 失败 → 客户端 Get 全量拿到 rev=250 → Watch(rev=251)
```

★ **关键认知**：**Watch 不是消息队列，不保证无限期保留事件**。etcd 只保留压缩点之后的历史。客户端必须实现"Watch 失败 → 全量重列 → 从新 revision 续订"的恢复逻辑，否则一次断线就可能永久丢事件。

# 6. 常见误解与坑

| 误解 | 事实 | 正确做法 |
|------|------|----------|
| etcd 可以当 Redis 用 | 写要过 Raft 和磁盘，吞吐低、容量小 | 只存元数据、配置、锁 |
| 任意节点读都一样新 | 串行化读可能落后 | 需要最新值时用默认线性一致读 |
| Watch 会保留所有历史 | 压缩后旧事件不可回放 | 处理 `ErrCompacted`，全量重列 |
| 3 节点集群挂 2 个还能读 | 失去多数派，无法线性一致读/写 | 恢复多数派后再提供服务 |
| value 可以随便大 | 请求默认上限约 1.5 MB，库容量有限 | value 控制在 KB 级，大对象存外部 |
| 删除 key 就释放空间 | 只写 tombstone，空间要 compact + defrag | 定期压缩与碎片整理 |
| 集群节点越多越好 | 节点越多，写延迟越高 | 生产用 3 或 5 节点 |

⚠ **关键认知**：etcd 的"强一致"只覆盖 etcd 自身的数据。如果你的业务先写 MySQL 再写 etcd，两者之间没有事务，仍然需要补偿、对账或本地消息表。

# 7. 自检题

1. 3 节点 etcd 集群挂掉 1 个 Follower，读写是否正常？再挂掉 Leader 呢？请从 quorum 和选举两个角度解释。
2. 为什么"先 Get 前缀，再 Watch"必须使用 `resp.Header.Revision + 1` 作为 Watch 起点？如果直接用 `WithRev(0)` 会怎样？
3. 一个客户端用 `WithSerializable()` 从 Follower 读配置，另一个客户端刚在 Leader 写入成功。它一定读不到新值吗？为什么？

# 8. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 强一致、小容量、带 Watch 与 Lease 的元数据存储 |
| 数据模型 | 扁平 key，全局 revision，MVCC 保留历史 |
| Raft 角色 | Leader 处理写，Follower 复制与投票，Candidate 拉票 |
| 日志复制 | 多数派持久化后提交，再应用到状态机 |
| 选举 | 随机超时 + term + 多数票，奇数节点更划算 |
| 线性一致读 | 默认读，需要 Leader 确认多数派并等待 apply |
| 串行化读 | 本地读，可能陈旧，适合可容忍延迟的场景 |
| Watch | 按 revision 推送，压缩后必须全量重列 |

下一篇进入部署，讲清单机与三节点集群的启动参数、peer/client 两个端口的区别、集群初始化流程、健康检查，以及最常见的启动失败原因。
