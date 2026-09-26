
# 1. 基础概念

## 1.1 Q1: ZooKeeper 是什么？核心能力有哪些？

**回答要点**：
- 分布式协调服务，为分布式系统提供协调原语
- 核心能力：配置管理、服务发现、分布式锁、集群选主、分布式队列/屏障
- 数据模型：类文件系统的层次 ZNode 树
- 通知机制：Watch 事件驱动
- 一致性协议：ZAB（ZooKeeper Atomic Broadcast）

## 1.2 Q2: ZooKeeper 的 CAP 模型是什么？为什么？

**回答要点**：
- **CP 模型**（一致性 + 分区容错），牺牲可用性（A）
- 过半机制（Quorum）：需要 N/2+1 节点确认
- Leader 选举期间集群不可写（短暂不可用）
- 典型场景：Leader 宕机后 ~10-30s 选举期间写入被拒绝

## 1.3 Q3: ZooKeeper 的节点类型有哪些？

**回答要点**：
- 4 种基本类型：PERSISTENT（持久）、EPHEMERAL（临时）、PERSISTENT_SEQUENTIAL（持久顺序）、EPHEMERAL_SEQUENTIAL（临时顺序）
- 3.5+ 扩展：CONTAINER（容器节点）、PERSISTENT_WITH_TTL、PERSISTENT_SEQUENTIAL_WITH_TTL
- 临时节点不能有子节点，随 Session 消失而删除
- 顺序节点在路径后自动追加 10 位递增序号

## 1.4 Q4: 什么是 Watch 机制？有什么特点？

**回答要点**：
- 客户端在 ZNode 上注册监听器，ZNode 变化时服务端推送通知
- 核心特点：
  1. **一次性触发**（One-time Trigger）：触发后自动删除（3.6+ 支持持久化 Watch）
  2. **轻量级**：通知只告知"变了"，不携带变更数据
  3. **有序性**：通知顺序与变更顺序一致
  4. **异步**：通过独立线程发送，不阻塞主流程

# 2. 协议与一致性

## 2.1 Q5: ZAB 协议是什么？与 Paxos/Raft 有什么异同？

**回答要点**：
- ZAB = ZooKeeper Atomic Broadcast，专为 ZK 主备模型设计的一致性协议
- 两大模式：崩溃恢复 + 原子广播
- 与 Paxos：ZAB 主备固定 Leader（不像 Paxos 允许多 Proposer 竞争），更简洁
- 与 Raft：ZAB 和 Raft 都采用固定 Leader + 日志复制，Raft 更强调可理解性
- ZAB 特点：epoch 机制、Proposal + ACK + Commit 两阶段、截断机制

## 2.2 Q6: ZooKeeper 如何保证数据一致性？

**回答要点**：
1. **全局顺序**：zxid 保证所有操作全局有序
2. **过半提交**：Leader 必须收到过半 ACK 才提交
3. **崩溃恢复**：新 Leader 必须拥有所有已提交事务（鸽巢原理）
4. **截断机制**：丢弃旧 Leader 未提交的事务
5. **epoch 递增**：防止旧 Leader 在恢复后继续处理请求

## 2.3 Q7: 为什么 ZooKeeper 集群推荐奇数个节点？

**回答要点**：
- 容错数 = (N-1)/2
- 3 节点容错 1，4 节点也容错 1（都需 3 票过半）
- 4 节点比 3 节点多消耗一台机器但容错没有提升
- 5 节点容错 2（需 3 票），真正提升了容错能力

# 3. Leader 选举

## 3.1 Q8: ZooKeeper 的 Leader 选举过程是怎样的？

**回答要点**：
1. 所有节点进入 LOOKING 状态
2. 投自己一票（vote id=self, zxid=self.zxid）
3. 与其他节点交换投票
4. PK 比较规则：epoch 大者胜 → zxid 大者胜 → myid 大者胜
5. 更新自己的投票为更优候选者
6. 重复直到某个节点获得过半票数
7. 算法名称：FastLeaderElection（FLE）

## 3.2 Q9: Leader 选举时如何保证选出的 Leader 拥有最新数据？

**回答要点**：
- 投票 PK 的第二优先级是 zxid（最大事务 ID）
- zxid 最大的节点必然包含了所有已提交的事务
- 证明：已提交的事务被过半节点持久化 → 任意过半节点集合必有交集 → 新 Leader（需要过半选票）必然从过半数节点中被选出 → 其中至少一个节点包含所有已提交事务 → 新 Leader 的 zxid 就是最大的

# 4. Session 与临时节点

## 4.1 Q10: ZooKeeper 的 Session 机制是怎样的？

**回答要点**：
- Session 是客户端和服务端之间的逻辑连接
- 关键点：
  1. SessionID 唯一标识，会话超时时间由客户端提议、服务端协商截断
  2. 心跳间隔 = sessionTimeout / 3
  3. 普通请求也可作为隐式心跳
  4. 分桶管理过期：O(1) 复杂度检测过期 Session
  5. Session 过期后：清理临时节点 + 清理 Watch

## 4.2 Q11: 临时节点和持久节点有什么区别？临时节点可以用来做什么？

**回答要点**：
- 持久节点：显式创建和删除，与 Session 无关
- 临时节点：与创建者的 Session 绑定，Session 过期/关闭时自动删除；不能有子节点
- 用途：服务注册（实例下线自动清理）、分布式锁（锁持有者崩溃自动释放）、选主（Leader 崩溃自动摘除）

# 5. 分布式锁

## 5.1 Q12: ZooKeeper 如何实现分布式锁？

**回答要点**：
- 基于 EPHEMERAL_SEQUENTIAL 节点
- 流程：
  1. 所有竞争者创建临时顺序节点
  2. 获取子节点列表，按序号排序
  3. 序号最小的获得锁
  4. 其他竞争者 Watch 前一个节点（链式等待）
- 优势：自动释放（Session 断开）、避免惊群效应、可重入（Curator）

## 5.2 Q13: 为什么 ZK 分布式锁能避免惊群效应？

**回答要点**：
- 所有等待者只 Watch 自己的前一个节点
- 锁释放时，只有紧邻的下一个等待者被唤醒
- 一次只有一个节点被通知去竞争锁
- 对比：所有等待者 Watch 同一节点会导致全部唤醒后竞争（惊群）

# 6. 源码与实现

## 6.1 Q14: ZooKeeper 的请求处理链是怎样的？

**回答要点**：
```
Leader 处理链：
PrepRequestProcessor → ProposalRequestProcessor → CommitProcessor → FinalRequestProcessor

Follower 处理链（写请求）：
FollowerRequestProcessor → [转发给 Leader] → CommitProcessor → FinalRequestProcessor

各处理器职责：
- PrepRequestProcessor：生成 zxid、设置 session
- ProposalRequestProcessor：发起提案、广播给 Follower
- CommitProcessor：等待过半 ACK
- FinalRequestProcessor：应用到 DataTree、触发 Watch、返回响应
```

## 6.2 Q15: ZooKeeper 服务端和客户端使用了什么网络模型？

**回答要点**：
- 3.x：Java NIO（NIOServerCnxnFactory），单线程 Accept + 多线程 Selector
- 3.6+：支持 Netty（NettyServerCnxnFactory），性能更好、更稳定
- 客户端：使用 NIO SocketChannel，SendThread + EventThread 双线程模型

# 7. 实践与运维

## 7.1 Q16: 如何保证 ZooKeeper 的高可用？

**回答要点**：
- 集群部署：至少 3 节点（推荐 5）
- 节点分布：不同物理机/机架
- 独立磁盘：dataLogDir 使用 SSD
- Observer：读扩展 + 跨区域部署
- 监控告警：Leader 存在性、延迟、连接数、磁盘空间
- 自动清理：autopurge 开启

## 7.2 Q17: ZooKeeper 适合存储大量数据吗？为什么？

**回答要点**：
- **不适合**
- 原因：所有数据在内存中、单 ZNode 默认上限 1MB、ZNode 数量过多影响性能
- 建议：只存储元数据/配置（KB 级），大数据用 HDFS/DB/对象存储
- ZNode 数量控制在 10 万级别以下

## 7.3 Q18: ZooKeeper 和 etcd/Consul 如何选择？

**回答要点**：
- ZK：Java 生态系统（Dubbo/Kafka/Hadoop），需要底层协调原语
- etcd：Kubernetes/Go 生态，简单 K-V 存储
- Consul：一站式服务网格，多数据中心，Web UI
- 核心判断：技术栈匹配度 > 功能特性 > 运维复杂度

## 7.4 Q19: 有什么常见的 ZooKeeper 反模式？

**回答要点**：
1. 把 ZK 当数据库（存大量数据）
2. 把 ZK 当消息队列（高频创建删除顺序节点）
3. 全局大锁（吞吐瓶颈）
4. 手动管理 Watch（事件丢失）
5. 偶数节点集群（浪费资源）
6. sessionTimeout 过短（GC 导致误判）
7. 不设置 namespace（多应用数据混乱）

## 7.5 Q20: ZooKeeper 服务端如何处理数据持久化？

**回答要点**：
- 事务日志（Transaction Log）：每条写操作在提交前写盘，顺序追加
- 快照（Snapshot）：事务数达到 snapCount（默认 100K）时，序列化整个 DataTree
- 恢复：加载最新快照 + 回放后续事务日志
- forceSync=yes 保证每次事务都 fsync，确保持久性
