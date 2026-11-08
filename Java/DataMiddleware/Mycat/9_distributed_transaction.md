# 1. 概述：分片事务的困难

**事务的 ACID 建立在"单一数据库实例的锁与日志"之上，数据一跨库，原子性就失去了统一提交点**。在 Mycat 里一个前端事务可能涉及多个 dataNode：

```text
BEGIN;
  INSERT INTO orders(user_id,...) VALUES (1,...);    → dn1
  INSERT INTO orders(user_id,...) VALUES (2,...);    → dn2
  UPDATE account SET balance=balance-1 WHERE uid=1;  → dn3
COMMIT;
```

单库模式下这三条在同一个 MySQL 里由 redo log + 组提交保证原子；分片后它们分属三个物理实例，`COMMIT` 要变成"三次网络提交"——**只要任何一次失败，就可能出现"部分成功"的脏状态**。分布式事务要解决的正是这个"要么全成、要么全败"的跨节点协调问题。

Mycat 处理事务的方式按版本与配置分成几档，先建立全景：

| 模式 | 版本 | 一致性 | 引入时间 | 生产使用度 |
|------|------|--------|---------|-----------|
| 默认本地事务（各节点各自提交，不协调） | 1.6 | 无原子保证 | 初始 | **绝大多数生产在用** |
| XATX（强一致 2PC） | 1.6.7.4+ | 强一致 | 后期 | 少 |
| XAAUTO | 1.6.7.4+ | 强一致 | 后期 | 少 |
| 原生 XA + Calcite | 2.x | 强一致 | 2019 | 增长中 |
| 借助 Seata AT/TCC | 任意 | 最终/补偿一致 | 生态 | 主流强一致方案 |

> **注意**：本节先把最容易误解的一条讲透——**Mycat 1.6 默认模式下，跨节点事务根本不是原子事务**。应用写 `BEGIN...COMMIT`，Mycat 只是把 COMMIT 翻译成"给每个参与节点各发一次 COMMIT"，如果第二个节点提交失败，第一个节点的写入不会回滚。这不是 bug，是 1.6 早期"性能优先、把一致性交给业务"的设计选择，也正因如此，业界用 Mycat 时几乎都靠"单分片事务 + 业务补偿"绕开跨库事务，而不是依赖中间件。

# 2. 默认模式：连接级事务的真相

**Mycat 的事务边界默认是"按数据节点分段的本地事务"**。一个前端连接开启事务后，Mycat 为每个被涉及的 dataNode 各借一条后端连接并 `SET autocommit=0`，事务状态在每条后端连接上独立维护：

```text
前端 BEGIN
   │
   ├── 首次写 dn1 → 借 dn1 后端连接 → SET autocommit=0
   ├── 首次写 dn2 → 借 dn2 后端连接 → SET autocommit=0
   │
前端 COMMIT
   ├── 向 dn1 后端连接发 COMMIT
   └── 向 dn2 后端连接发 COMMIT   ← 两者相互独立，无协调！
```

这带来三个直接后果：

1. **单分片事务是安全且高效的**——所有语句落同一节点，退化为普通 MySQL 事务，Mycat 完全透明。这是"MySQL 能做到的 Mycat 也能做到"的唯一范围
2. **跨分片写不保证原子**——某节点 COMMIT 失败时，其余节点已提交的数据不会回滚，产生不一致
3. **事务期间连接被绑定**——事务不结束，借出的后端连接不能归还，跨节点事务会同时占用多个节点的连接，并发高时放大连接池压力

**设计考量**：

- **绝大多数业务把"跨分片写"设计成"单分片写"**：用 ER 分片把订单主表与明细表绑进同一分片，一个下单事务天然落在一个 dataNode，直接享受本地事务的完整 ACID
- **确实要跨节点的场景**（转账扣两个不同用户的库）走应用层最终一致：本地消息表 / 事务消息 / TCC，而不是把希望寄托在中间件的隐式协调

# 3. XATX：基于 2PC 的强一致模式

## 3.1 XA 协议与 2PC 流程

**MySQL 从 5.5 起原生支持 XA 事务接口**（`XA PREPARE`/`XA COMMIT` 语句），底层由 InnoDB 把 prepare 状态持久化到 binlog，具备崩溃恢复能力。Mycat 的 XATX 模式就是让中间件扮演 XA 协调者（TM），把跨节点事务改写成标准 2PC：

```text
Phase 1 (Prepare):
  1. Mycat(TM) 向每个参与的 MySQL(RM) 下发 XA START 'xid' → 执行各分片 SQL
  2. 执行完毕发 XA END + XA PREPARE
  3. 各 MySQL 将事务写入 binlog（prepare 状态），返回 PREPARE OK

Phase 2 (Commit):
  4. 全部 PREPARE OK → Mycat 逐个发 XA COMMIT 'xid'
  5. 任一 PREPARE FAIL → Mycat 逐个发 XA ROLLBACK 'xid'

故障恢复：
  MySQL 重启后 XA RECOVER 能列出处于 prepared 的悬挂事务
  → 协调者（Mycat 集群，经 Atomix/ZK 共享事务日志）决定最终 COMMIT/ROLLBACK
```

## 3.2 配置与使用

```xml
<!-- schema.xml 的 dataHost 上声明 XA 心跳检查（配合 1.6.7.4+） -->
<dataHost name="host1" maxCon="1000" minCon="10" balance="0"
          writeType="0" dbType="mysql" dbDriver="native"
          switchType="1" slaveThreshold="100" use_xa_for_check="1">
  <heartbeat>select 1</heartbeat>
  <writeHost host="w1" url="192.168.1.10:3306" user="mycat" password="mycat123"/>
</dataHost>
```

```sql
-- 会话级开启跨节点 XA 事务（xatx 模式）
SET autocommit = 0;
BEGIN;
INSERT INTO orders(id, user_id) VALUES (1, 100);   -- dn1
INSERT INTO orders(id, user_id) VALUES (2, 101);   -- dn2
COMMIT;   -- 此时走 XA 两阶段提交，两个分片原子生效

-- XAAUTO：不显式 BEGIN，Mycat 检测到语句跨节点时自动升级为 XA
SET xatx_commit_level = 3;
```

`xatxCommitLevel` 的语义（server.xml 全局属性，也可会话级覆盖）：

| 值 | 含义 |
|----|------|
| 0 | 关闭 XA（默认，本地事务模式） |
| 1 | 仅全局表 DML 使用 XA |
| 2 | 跨库（跨 dataHost）事务使用 XA |
| 3 | 全部跨节点事务使用 XA |

## 3.3 为什么生产上很少开 XA

**性能与可用性两笔账都算不过来**：

1. **2PC 是同步阻塞协议**：Phase 1 要等所有 RM prepare 完成，最慢节点决定整体延迟；prepare 后到 commit 之间，所有节点的行锁与 XA 快照被"冻结持有"，锁持有时间约为本地事务的 2~3 倍，高并发下锁等待级联放大，官方社区基准普遍给出 **XA 相比本地事务吞吐下降 30%~60%**
2. **悬挂事务依赖协调者活着**：Mycat 在 Phase 2 中途宕机，prepared 事务要等它（或它的集群副本）恢复后裁决；单点部署 Mycat 时这就是一个"数据卡在中间态"的运维噩梦，这也是 1.6.7.4 把 XATX 与 Atomix 集群绑定发布的原因——**XA 模式事实上要求 Mycat 集群化部署**
3. **MySQL XA 的历史 bug 面**：主从切换后 XA 状态丢失、`innodb_support_xa` 与 GTID 组合的若干边界问题在 5.7 时代时有报告，DBA 群体对线上开 XA 普遍持保留态度

> **设计争议**：Mycat 团队在 1.6 后期补上 XATX，被社区解读为"补齐 PPT 能力"而非"生产可用承诺"——同期阿里 DTS/PolarDB-X 已明确把强一致跨库事务放存储层做，ShardingSphere 走"本地事务优先 + Seata 集成"路线。分片中间件自己在代理层做 XA 的时代，基本随着 2019 年前后"业务单分片化 + 专门的分布式事务框架"共识的形成而终结。

# 4. 与 Seata 的关系

**Seata 是生态位互补而非竞品**：Mycat 管路由，Seata 管跨服务/跨库事务协调。集成形态上，Seata AT 模式要求数据源被代理为 `DataSourceProxy`——应用直接连 Mycat 时，Seata 拦截不了 SQL，因此常见组合有两种：

```text
组合 A（推荐）：Seata AT + 各分片 MySQL 直连数据源
  应用(ShardingSphere-JDBC/手工路由) → 分片 MySQL
  Seata TC 集群协调全局事务，undo_log 补偿回滚
  —— Mycat 不参与事务层

组合 B：应用 → Mycat(单分片事务) → MySQL
  跨分片需求出现时，用业务改造规避：
  1) ER 缠绕让事务单分片化
  2) 拆事务 + 本地消息表/事务消息最终一致
```

| 维度 | Mycat XATX | Seata AT |
|------|-----------|----------|
| 一致性 | 强一致（2PC） | 最终一致（补偿，全局锁读隔离） |
| 锁持有 | prepare 期间全节点持锁，长 | 仅更新行全局锁，短 |
| 额外组件 | 无（Atomix 内置于 Mycat 集群） | Seata Server（TC）+ undo_log 表 |
| 性能损耗 | 30%~60% | 10%~20% |
| 应用改造 | 零（透明） | 数据源需代理，同库需装 undo_log |
| 适用 | 低频强一致写 | 高频微服务跨库写 |

> **行业实践**：2026 年国内栈的主流答案高度一致——**分片层（Mycat/ShardingSphere）负责让单分片事务覆盖 90% 场景，剩下 10% 交给 Seata AT/TCC 或消息最终一致**；把 Mycat XATX 当成"开箱即用的分布式事务"直接上线的团队，绝大多数在压测阶段就退回了默认模式。

# 5. 隔离级别与常见坑

1.6 把前端会话的隔离级别透传给所有涉及的后端节点：

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Mycat 向本会话借出的每条后端连接同步执行该 SET
```

**三个高频坑**：

1. **事务中途涉及新节点**：事务开始时只写了 dn1，后来一条 UPDATE 打到 dn2——dn2 的连接是在事务中途才 `SET autocommit=0` 的，它的快照起点比事务名义起点晚，RR 下的可重复读语义在跨节点视角并不成立（默认模式与 XA 模式都有此现象，程度不同）
2. **XA 模式要求所有 RM 同一事务分支超时**：`xaRecoverSleepDelaySeconds`/`xaTransactionTimeout` 设置不当会造成"Mycat 已放弃、MySQL 侧 prepared 永久悬挂"，必须配合巡检脚本定期 `XA RECOVER CONVERT XID` 清理
3. **SAVEPOINT 跨节点无效**：中间件层无法把回滚点同步到多后端连接，1.6 直接不支持事务内 SAVEPOINT

```sql
-- 悬挂 XA 事务的巡检示例（后端 MySQL 执行）
XA RECOVER CONVERT XID;
-- 确认协调者已放弃的分支后手工收敛：
XA COMMIT 'xid-001';
-- 或
XA ROLLBACK 'xid-002';
```

# 6. 小结

**Mycat 分布式事务的现实分层是三层**：默认模式给"单分片事务 + 跨分片尽力提交"，透明但不原子，这是绝大多数生产系统的实际选择；XATX/XAAUTO（1.6.7.4+，需集群部署）补上标准 2PC 强一致，代价是 30%~60% 吞吐损耗、长锁持有与悬挂事务运维负担；跨服务场景则交给 Seata AT/TCC 或消息最终一致，Mycat 退回路由层。**架构决策的正确顺序**：先用 ER 分片把事务压进单分片 → 压不进去的用补偿/消息做最终一致 → 真正需要强一致的金融级链路才考虑 XA，并把中间件集群高可用当作前置条件。指望中间件透明解决一切跨库事务，在 2026 年依然是不成立的幻想。