# 1. XA 协议概述

XA 是 X/Open DTP（分布式事务处理）参考模型定义的**分布式事务标准**，规定了事务管理器（TM）与资源管理器（RM，即数据库）之间的协调接口：`XA START / END / PREPARE / COMMIT / ROLLBACK / RECOVER`。它把传统 2PC 拆成了可落地的动词表——TM 先让每个 RM 执行 `XA PREPARE`（把事务改到"持久化就绪"状态），全部 PREPARE 成功后再逐一 `XA COMMIT`，任何一个环节失败则统一 `XA ROLLBACK`。

Seata 的 XA 模式（`Branch_Type.XA`）没有自研一套回滚机制，而是**把 2PC 的实现完全委托给数据库**：Seata TC 负责全局事务状态机与提交决策，真正的 prepare/commit/rollback 由 JDBC 的 `javax.transaction.xa.XAResource` 接口驱动数据库原生执行。代价换来的是数据库保证的隔离性，收益是"零 undo_log、强一致、SQL 通吃"。

> **设计考量**：XA 是 1991 年的标准，比 Seata 早三十年。Seata XA 模式的价值不是发明新协议，而是给"数据库已具备的能力"接上现代化的全局事务编排——TM/RM 生命周期管理、xid 传播、故障恢复调度都由 Seata 接管，业务只看到一个 `@GlobalTransactional`。

# 2. XA 两阶段提交时序

```text
Phase 1 (Prepare):
  ┌────────────────────────────────┐
  │ TM → TC: 全局事务 begin         │
  │ RM → TC: 分支注册               │
  │ 业务 SQL 在 XA START...END 间执行│
  │ TC → RM: XA PREPARE            │
  │ RM → TC: PREPARE OK / FAIL     │
  └────────────────────────────────┘

Phase 2 (Commit/Rollback):
  ├── All PREPARE OK → XA COMMIT（可恢复提交）
  └── Any FAIL       → XA ROLLBACK（全链回退）
```

**关键特征**：`XA PREPARE` 之后，事务处于"已持久化就绪、等待最终裁决"状态——修改写入 redo/prepared 事务日志，**数据库行锁持续持有**直到 Phase 2 完成。这与 AT 模式（Phase 1 本地事务直接提交、立刻释放 DB 锁）形成根本差异。

Seata XA 的完整流程：

```text
TM                     TC                   RM (XA DataSource)
  │                     │                        │
  │── global begin ───→ │                        │
  │←──── xid ─────────  │                        │
  │                     │                        │
  │  业务方法执行         │                        │
  │                     │←── branch register ────│
  │                     │                        │── XA START
  │                     │                        │── 业务 SQL
  │                     │                        │── XA END
  │                     │←── branch report(IDLE) ─│
  │                     │                        │
  │── global commit ──→ │                        │
  │                     │── XA PREPARE ─────────→│
  │                     │←─ prepared ────────────│
  │                     │── XA COMMIT ──────────→│
  │                     │←─ committed ───────────│
  │←── result ───────── │                        │
```

注意一个细节：Seata XA 在 Phase 1 **就完成 branch report**，但数据库侧的真正锁释放发生在 Phase 2 的 COMMIT/ROLLBACK。全局事务的 RT = 所有分支的 prepare 时间 + commit 广播时间，锁持有时间直接由 RT 决定——这就是 XA 吞吐上不去的结构性原因。

# 3. Seata XA 集成

## 3.1 配置

```yaml
seata:
  enabled: true
  data-source-proxy-mode: XA    # 关键：代理模式切到 XA
  tx-service-group: order_tx_group

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/order_db?useUnicode=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    # XA 模式必须使用 XA 数据源驱动
```

MySQL 的 XA 数据源需要 `mysqlx` 之外的 XA-capable 驱动配合，Seata XA 代理会反射构造 `com.mysql.jdbc.jdbc2.optional.MysqlXADataSource`（8.x 为 `com.mysql.cj.jdbc.MysqlXADataSource`）；Oracle/PostgreSQL/DB2 自带 XA 实现。

```xml
<!-- 依赖：seata-all 已内置 XA 支持，无需额外驱动包（数据库驱动除外） -->
<dependency>
	<groupId>io.seata</groupId>
	<artifactId>seata-spring-boot-starter</artifactId>
	<version>2.2.0</version>
</dependency>
<dependency>
	<groupId>com.mysql</groupId>
	<artifactId>mysql-connector-j</artifactId>
</dependency>
```

## 3.2 业务代码

```java
// XA 模式无需修改业务代码，与 AT 模式写法完全一致
@GlobalTransactional(rollbackFor = Exception.class)
public void createOrder(Order order) {
    orderMapper.insert(order);                    // 分支1：本地 XA
    accountClient.debit(userId, amount);          // 分支2：RPC → 远程 XA 分支
    inventoryClient.deduct(productId, quantity);  // 分支3：远程 XA 分支
}
```

XA 与 AT 的切换是配置级的（`data-source-proxy-mode`），对业务代码零差异，这让"同一套代码在不同库/不同一致性要求的环境分别跑 AT 与 XA"成为可行运维策略。

## 3.3 与 AT 的资源差异

- AT：需要 `undo_log` 表，Phase 1 本地事务提交即释放 DB 锁。
- XA：不需要 `undo_log` 表，Phase 1 Prepare 后 DB 锁持续持有直到 Phase 2。
- AT 靠 Seata 全局锁防脏写；XA 天然没有"脏写"问题——因为它根本没提前提交，隔离完全由数据库行锁保证。

# 4. XA vs AT 对比

| 维度 | XA 模式 | AT 模式 |
|------|---------|---------|
| **锁持有时间** | Prepare 后持续持锁直到全局裁决（长） | Phase 1 提交即释放本地锁（短） |
| **并发性能** | 低（锁窗口≈全局事务 RT） | 高（锁窗口≈单条 SQL） |
| **一致性级别** | 强一致（隔离由 DB 保证） | 最终一致（默认读未提交） |
| **读隔离** | 数据库默认隔离级别即可 | 需 `@GlobalLock` + FOR UPDATE 补丁 |
| **undo_log 表** | 不需要 | 需要 |
| **脏写风险** | 无（未提交不可见） | 靠全局锁防止 |
| **数据库要求** | 必须支持 XA 协议 + XA 驱动 | 支持本地事务即可 |
| **SQL 支持面** | 所有 SQL（DDL 参与限制除外） | 需可解析的 INSERT/UPDATE/DELETE |
| **故障恢复** | 数据库 prepared 事务 + XA RECOVER | Seata 驱动 undo_log 补偿 |
| **资源占用** | 连接、锁被占用到最后一刻 | 释放快，连接占用短 |

**设计考量**：两者是"隔离性换吞吐"的两端。XA 的全局事务时间被放大为资源占用时间，热点行（高并发扣减同一账户/库存）场景下 XA 几乎必然成为瓶颈；AT 通过提前提交+全局锁把冲突检测从 DB 挪到 TC，牺牲了天然隔离换吞吐。不存在"哪个更先进"，只有"哪段业务需要哪种代价"。

# 5. 全局锁缺失的真相：无脏写但有资源占用

一个容易误读的点：AT 模式讲解里常说"全局锁防止脏写"，于是有人推断 XA 没有全局锁是个缺陷。恰恰相反——**XA 因为本地事务迟迟不提交，根本不存在脏写窗口**，TC 侧因此无需维护 lock_table 行锁记录（Seata XA 分支注册时 lock 相关逻辑被跳过）。但这不代表 XA "更轻"：

```text
XA 的资源账本（每个全局事务窗口内）：
  ├── 数据库行锁     —— 持续到 Phase 2，阻塞其他事务对同行的读写
  ├── 数据库连接     —— XA 连接在 prepare 后不能归还连接池
  ├── 事务日志空间   —— prepared 状态写入 redo，崩溃也要保留
  └── TC 会话状态    —— 全局事务在 TC 内存/存储中的存续时间变长
```

> **注意**：**XA 的"锁"比 AT 的全局锁更重**。AT 的全局锁是 TC 里一条逻辑记录（可快速重试）；XA 的锁是真实的 InnoDB 行锁+连接占用。AT 高并发冲突表现为 lock_table 重试；XA 高并发冲突直接表现为业务 SQL 层面的 `Lock wait timeout exceeded`——冲突会打到用户请求上，这是 XA 压测 TPS 腰斩的根因，也是"资源占用问题"的准确含义。

# 6. MySQL 中的 XA 事务与恢复

脱离 Seata 看数据库原生 XA，语义更清楚：

```sql
-- XA 事务示例
XA START 'xid-001';
UPDATE account SET balance = balance - 50 WHERE user_id = 1;
XA END 'xid-001';
XA PREPARE 'xid-001';
-- 此时数据已持久化就绪但被锁定：
-- 其他事务无法修改/锁定 user_id=1 的行，查询可见性受隔离级别约束

-- Phase 2: 提交或回滚
XA COMMIT 'xid-001';   -- 或 XA ROLLBACK 'xid-001';

-- 崩溃后找回悬挂的 prepared 分支
XA RECOVER CONVERT XID;
```

**MySQL XA 的关键限制**：

- XA 事务与本地事务不能在同一连接混用；autocommit 必须关闭。
- prepared 事务在崩溃恢复后**必须人工或程序 `XA COMMIT / XA ROLLBACK`**，否则会一直持有锁——Seata 的 XA 分支恢复逻辑会按 TC 状态自动处理，但绕过 Seata 手工执行 XA 时这是运维雷区。
- 主从复制：MySQL 单阶段提交 XA 事务的 binlog 与 prepared 日志存在窗口不一致（5.7/8.0 引入 `XA support for recovery` 即 binlog 内记录 XA 事件后才缓解），云数据库（如部分 RDS/PolarDB 版本）对 XA 支持受限，**上线前必须在目标实例实测**。
- 驱动与连接池：HikariCP 需正确配置，XA 连接在 prepare 期间不能回收；连接池 maxLifetime 过短会引发 `connection killed` 类故障。

# 7. 适用场景

- ✓ 需要**强一致**且事务短：账户间调账、跨行/跨库清算类（配合短 RT）。
- ✓ 数据库原生 XA 成熟可用（MySQL 5.7+/8.0、Oracle、PostgreSQL、DB2）。
- ✓ **异构 DB** 场景：同一全局事务里混合 MySQL + Oracle + PostgreSQL——AT 对每种库都要解析 SQL 方言，XA 只要驱动合规即可统一接入，这是 XA 相比 AT 的独门优势。
- ✓ SQL 面复杂到 AT 无法解析：大量存储过程外仍含复杂 UPDATE/函数表达式时，AT 的前后镜像查询可能失效，XA 不关心 SQL 内容。
- ✓ 遗留系统改造：零代码变更 + 零 undo_log 表，只换代理模式。
- ✗ 高并发热点行场景（长时间锁影响吞吐量，压测必现瓶颈）。
- ✗ 长事务（全局 RT 分钟级 → 锁占用分钟级，绝对禁止）。
- ✗ 云数据库对 XA 支持存疑的环境（先实测再选型）。

**使用场景对照（与 AT 的分界）**：同一个下单链路，MySQL 同构库 + 高并发选 AT；跨 MySQL/Oracle 异构 + 中低频强一致选 XA；资金级强一致 + 高并发，两者都不合适，应该走 TCC 业务预留或单库聚合根重构。

# 8. 小结

Seata XA 模式的本质是"**把 Seata 降级成全局事务的编排器，把一致性执行权交还给数据库 2PC**"：不需要 undo_log、没有脏写窗口、隔离性真实、SQL 通吃、异构数据库统一接入；代价是 prepare 后行锁与连接一直占着直到全局裁决，吞吐随全局 RT 线性恶化。

选型口径可以收敛成一句话：**短事务 + 强隔离 + 异构/复杂 SQL 选 XA，热点行与高并发一律回避 XA**。上线前必须完成三件验证——目标实例（尤其云数据库）的 XA 能力实测、prepared 悬挂事务的恢复演练（`XA RECOVER` 与 Seata 恢复任务联调）、以及连接池与超时的锁等待压测基线。把 XA 当成"AT 的强一致备胎"而非性能选项，它的价值才能正确兑现。
