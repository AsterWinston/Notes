# 1. 知识点
- **数据模型**：类似文件系统的层级命名空间，ZNode 是数据节点（可存数据和子节点），路径用 `/` 分隔。ZK 的数据全部存储在内存中，保证了高吞吐低延迟。
- **节点类型**：
  - **持久节点（PERSISTENT）**：创建后持久存在，除非手动删除。
  - **临时节点（EPHEMERAL）**：与创建者 Session 绑定，Session 失效后自动删除。
  - **持久顺序节点（PERSISTENT_SEQUENTIAL）**：持久节点 + ZK 自动在名称后加自增序号。
  - **临时顺序节点（EPHEMERAL_SEQUENTIAL）**：临时节点 + 顺序属性，是分布式锁实现的基础。
- **Session（会话）**：客户端连接 ZK 时创建 Session，有超时时间（`sessionTimeout`）。通过心跳保活，超时未收到心跳则 Session 失效，释放该 Session 下的临时节点和 Watcher。
- **Watcher 机制**：客户端向服务端注册 Watcher 监听 ZNode 变化。触发一次后失效（One-time Trigger），需重新注册。支持 `getData()`（数据变化）、`exists()`（创建/删除）、`getChildren()`（子节点变化）。Watcher 通知异步发送给客户端。
- **ZAB 协议（ZooKeeper Atomic Broadcast）**：ZK 的一致性协议，保证集群数据一致。包含两种模式：
  - **恢复模式（崩溃恢复）**：集群启动或 Leader 宕机后，选举出新 Leader，将 Follower 数据同步到一致状态。
  - **广播模式（消息广播）**：Leader 处理写请求，使用两阶段提交（类似）转发 Proposal 给 Follower，超过半数 Follower ACK 后提交。
- **节点角色**：
  - **Leader**：唯一，处理所有写请求，发起 Proposal 并提交。负责集群管理、给 Follower 分配全局唯一 ZXID。
  - **Follower**：多台，投票参与 Leader 选举，转发写请求给 Leader，可处理读请求。
  - **Observer**：可选节点，不参与投票和选举，只处理读请求。目的是提高读吞吐量，不降低写性能。
- **Leader 选举**：基于 ZXID（ZooKeeper Transaction ID）和 myid。规则：ZXID 大的优先（数据最新），相同 ZXID 时 myid 大的优先。节点间投票交换信息，达到过半票数后该节点成为 Leader。选举过程中节点状态：LOOKING → LEADING 或 FOLLOWING。
- **ZXID**：全局唯一事务 ID，64 位，高 32 位是 epoch（朝代/Leader 周期），低 32 位是递增事务计数器。每选一次 Leader，epoch 递增。
- **使用场景**：
  - **配置管理**：应用将公共配置写入 ZK 节点，服务启动时读取并在节点变更时 Watcher 通知动态刷新。
  - **服务注册与发现**：服务提供者创建临时节点注册，消费者通过 Watcher 获取最新服务列表。
  - **命名服务**：通过 ZK 的顺序节点生成全局唯一 ID（类雪花 ID）。
  - **分布式锁**：通过临时顺序节点 + Watcher 前一个节点实现。
  - **Master 选举**：利用临时节点和顺序节点抢注，序号最小的成为 Master。
  - **分布式队列**：利用顺序节点的 FIFO 特性。
  - **Barrier（屏障）**：等待所有节点到达后继续执行。
- **Curator 框架**：Apache 提供的 ZooKeeper 高级客户端，提供：流畅 API 风格、连接状态监听器 `ConnectionStateListener`、重试策略（`ExponentialBackoffRetry` / `RetryNTimes` 等）、递归创建节点（`creatingParentsIfNeeded`）、ACL 权限管理、Leader 选举器（`LeaderSelector`）、分布式锁（`InterProcessMutex` 可重入锁）、分布式计数器、读写锁等。
- **ZooKeeper 保证 CP**：满足 CAP 定理中的 CP（一致性 + 分区容错性）。当半数以上节点宕机导致无法选举 Leader 时，ZK 集群进入不可用状态。
- **分布式锁原理**：所有客户端在同一 `/lock` 路径下创建临时顺序节点，序号最小的节点获得锁，其他节点 Watch 前一个节点。释放时删除节点，触发后一个节点的 Watcher，通知它竞争成功。
- **ZooKeeper 不适合服务发现大规模节点**：所有服务节点大量临时节点心跳导致 ZK 负载过高。ZK 写入压力大且 CP 当分区时写不可用，影响注册。
- **与 etcd 对比**：
  - etcd 使用 Raft 协议（易理解和实现），ZooKeeper 使用 ZAB 协议。
  - etcd 更适合 K8s 和云原生场景（gRPC 接口支持多路复用）。
  - ZooKeeper 在 Java 生态更成熟（Dubbo/Druid/Kafka(旧版)/Hadoop 依赖 ZK）。
  - etcd 支持 MVCC（多版本并发控制）和 Watch 历史版本，ZK 不支持。
  - etcd 社区由 CNCF 维护，ZooKeeper 由 Apache 维护。
- **ACL 权限**：ZooKeeper 支持 ACL（访问控制列表），权限类型有 CREATE、READ、WRITE、DELETE、ADMIN。可通过 `scheme:id:permissions` 格式设置。
- **数据版本**：每个 ZNode 有三个版本号：`version`（数据版本）、`cversion`（子节点版本）、`aversion`（ACL 版本），用于乐观锁 CAS 操作。
- **Quorum（过半机制）**：ZK 集群强制要求 2n+1 台节点，写请求需超过半数节点确认才可提交。允许少于半数节点宕机仍然可用。
- **内存存储**：ZK 数据全部在内存中，单个 ZNode 最大 1MB（默认）。设计上不适合存储大容量数据，定位为协调服务。

# 2. 面试题

## 2.1 ZooKeeper 的 ZAB 协议是什么？
ZAB（ZooKeeper Atomic Broadcast）是 ZK 的核心一致性协议，包含两种模式：
1. **崩溃恢复模式**：集群启动或 Leader 宕机后进入。选举出 ZXID 最大的节点作为新 Leader（确保数据最新），Leader 将所有 Follower 的数据同步到一致状态后再进入广播模式。
2. **消息广播模式**：Leader 接收写请求，将操作封装为 Proposal 广播给所有 Follower。Follower 收到 Proposal 后记录日志并返回 ACK。Leader 收到过半 Follower ACK 后提交该 Proposal，并通知所有 Follower 提交。与两阶段提交类似，但 ZAB 添加了崩溃恢复的保证，确保数据一致性。

## 2.2 ZooKeeper 的 Watcher 机制是什么？有什么特点？
Watcher 是 ZK 的事件监听机制。客户端通过 `getData()`、`exists()`、`getChildren()` 三个操作注册 Watcher 监听 ZNode 变化。当 ZNode 发生相应变化时，服务端异步发送事件通知客户端。

特点：
- **一次性触发**：Watcher 触发一次后立即失效，如需持续监听则客户端收到通知后重新注册。
- **异步通知**：Watcher 事件异步发送，不影响 ZK 正常操作。
- **顺序性**：Watcher 事件按服务端变更顺序串行发送给客户端。
- **轻量级**：Watcher 通知只告知事件类型（NodeCreated/NodeDeleted/NodeDataChanged/NodeChildrenChanged）和路径，不包含变更后的数据。

## 2.3 ZooKeeper 如何实现分布式锁？
1. 所有竞争锁的客户端在 ZK 的指定 `/lock` 路径下创建**临时顺序节点**（如 `/lock/lock-000000001`、`/lock/lock-000000002`）。
2. 客户端获取 `/lock` 路径下所有子节点，找出序号最小的节点。
3. 如果自己创建的节点序号最小，则获得锁。
4. 如果不是最小，则找到比自己序号小的前一个节点，调用 `exists()` 注册 Watcher 监听其删除事件。
5. 当前一个节点被删除（锁释放），Watcher 通知到等待的客户端，客户端重新判断自己是否序号最小。
6. 获取锁的客户端执行完操作后删除自己的临时顺序节点，触发下一个等待者竞争。

优势：释放锁自动（Session 断开后临时节点自动删除），避免死锁。

## 2.4 ZooKeeper 的 Leader 选举过程是怎样的？
1. 所有节点初始状态为 LOOKING，多节点自增 ZXID 后投自己一票且发送投票给其他节点。
2. 节点收到投票后，比较投票信息。比较规则：优先比较 ZXID（越大越新），相同则比较 myid（越大优先）。
3. 如果收到投票中的节点比自己的投票更优，改为支持该节点并重新广播投票。票箱中统计自己和收到的投票数。
4. 某节点获得超过半数选票时，状态变为 LEADING，成为 Leader。其余节点状态变为 FOLLOWING。
5. Leader 开始与 Follower 同步数据（以 Leader 为准），同步完成后进入广播模式提供服务。

Leader 故障时：Follower 检测到与 Leader 心跳超时，所有节点状态变回 LOOKING 重新选举。

## 2.5 ZooKeeper 的节点类型有哪些？临时节点有什么用？
- **PERSISTENT（持久节点）**：创建后一直保存，不随连接断开消失。
- **EPHEMERAL（临时节点）**：创建者 Session 有效时存在，连接断开 Session 失效后自动删除。服务注册的基石：服务提供者创建临时节点注册，宕机后节点自动删除通知消费者。
- **PERSISTENT_SEQUENTIAL（持久顺序节点）**：持久节点 + 自动后缀序列号。用于分布式队列、全局唯一 ID 生成。
- **EPHEMERAL_SEQUENTIAL（临时顺序节点）**：临时节点 + 顺序号。分布式锁的基础实现节点。

## 2.6 ZooKeeper 的 Session 机制是什么？
客户端首次连接 ZK 创建 Session，服务端分配一个唯一 `sessionId` 和 `sessionTimeout`。客户端在 `sessionTimeout` 间隔内通过 Ping/Request 发送心跳。服务端超过 `sessionTimeout` 没有收到心跳，将 Session 置为 Expired，并触发清理该 Session 持有的临时节点和 Watcher。如果客户端连接断开又在 timeout 内重连则 Session 继续有效。

## 2.7 为什么 ZooKeeper 集群推荐奇数台节点？
因为 ZK 的过半确认机制（Quorum）要求写请求需要超过半数节点确认才能提交，集群容错能力为：允许 `(N-1)/2` 台节点宕机。偶数节点（如 4 台）也仅允许 1 台宕机（4 台过半需 3 台），与 3 台相比没有提升容错能力反而增加成本，因此奇数部署性价比更高。

## 2.8 ZooKeeper 和 etcd 的区别是什么？
| 维度 | ZooKeeper | etcd |
|------|-----------|------|
| 一致性协议 | ZAB | Raft |
| 数据模型 | ZNode 树形结构 | Key-Value 扁平空间 |
| API 风格 | 原生 Java/C 客户端 | gRPC + RESTful API |
| Watch 机制 | 一次性 Watcher | 持续 Watch（Watch 支持历史版本） |
| MVCC | 不支持 | 支持（多版本并发控制） |
| 适用场景 | Hadoop/Kafka(旧版)/Dubbo/HBase | Kubernetes/云原生/Prometheus |
| 性能 | 读性能高（全内存） | 读写均衡 |
| 社区 | Apache | CNCF |
| 语言 | Java | Go |

## 2.9 Curator 框架提供了什么能力？如何实现分布式锁？
Curator 是 ZooKeeper 的增强客户端，提供：
- **流畅 Builder API**：链式调用风格。
- **ConnectionStateListener**：连接状态变化通知。
- **重试策略**：`ExponentialBackoffRetry`、`RetryNTimes`、`RetryForever` 等。
- **分布式锁**：`InterProcessMutex`（可重入锁）、`InterProcessReadWriteLock`（读写锁）。
- **Leader 选举**：`LeaderSelector`，节点被选为 Leader 触发回调，宕机后自动重新选举。
- **分布式计数器/信号量**。

`InterProcessMutex` 分布式锁原理：内部基于临时顺序节点，创建节点→获取所有子节点排序→判自己是否最小→是则获取锁（内部用 CountDownLatch 阻塞），不是则 Watch 前一个节点并等待。

## 2.10 ZooKeeper 为什么不适合作为服务发现中心？
1. **CP 特性**：当超过半数节点宕机后 ZK 不可用，服务发现场景优先可用性（AP），短暂不可用导致服务间调用失败。
2. **心跳负载高**：大量服务节点（几千/万级）频繁创建临时节点心跳，ZK 负载压力大。
3. **写入压力**：服务上下线产生大量写入，ZK 为 Leader 单写，吞吐瓶颈。
4. **Watcher 一次性**：频繁注册 Watcher 增加网络开销。

替代方案：Nacos（AP+CP 切换）、Eureka（AP）、Consul（CP/AP）。

## 2.11 ZooKeeper 保证的是 CAP 中哪两个？
ZooKeeper 保证 **CP**（一致性和分区容错性）。发生网络分区时，少于半数节点的分区无法选举 Leader 会拒绝服务，保证数据一致性。牺牲了一定的可用性（A）。

## 2.12 什么是 ZooKeeper 的 ZXID？
ZXID（ZooKeeper Transaction ID）是 ZK 的事务全局唯一 ID，64 位长整型。高 32 位是 epoch（Leader 任期编号，每次选举新 Leader 时递增），低 32 位是事务计数器（Leader 任期内递增）。ZXID 用于判断数据的新旧版本，也是 Leader 选举的核心排序依据（优先比较 ZXID 的大小）。
