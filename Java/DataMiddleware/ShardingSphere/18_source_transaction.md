# 1. 事务管理器架构

分片环境里一个逻辑事务可能横跨 N 个物理连接，事务引擎要解决的问题是：**把这些物理事务收拢成一个有原子语义的整体**。ShardingSphere 把事务能力抽象成 SPI，三种实现对应三种一致性承诺：

```java
// ShardingSphere 事务管理器 SPI
public interface ShardingSphereTransactionManager extends AutoCloseable {

    // 初始化
    void init(DatabaseType databaseType,
              Collection<ResourceTransactionManager> managers);

    // 开启事务
    void begin();

    // 提交
    void commit() throws TransactionException;

    // 回滚
    void rollback() throws TransactionException;
}
```

5.5.x 的 `TransactionType` 枚举三值：**LOCAL（默认，单数据源本地事务 / 多数据源尽力而为）、XA（强一致，2PC）、BASE（柔性事务，当前实现为 Seata AT）**。全局规则由 `!TRANSACTION` 配置（`defaultType: XA`），Proxy 上可 `ALTER TRANSACTION RULE` 在线切默认类型；连接级可通过 `ConnectionTransaction` API 或 Hint 临时指定。**注意默认值是 LOCAL**：多库事务不显式声明 XA/BASE 时并不会自动升级为强一致，这是"用了 ShardingSphere 就以为有分布式事务"的最大误解。

## 1.1 与 Spring 事务的分工

一个高频混淆点要一次讲清：`@Transactional` 走的是 Spring 的 `DataSourceTransactionManager`，它只认识**一个** DataSource（即 ShardingSphereDataSource），管理的是"逻辑连接"的开启/提交时机；真正的多物理连接原子性由 ShardingSphere 内部事务管理器决定。**两层职责不重叠也不互相替代**——Spring 负责传播行为与切面，SS 负责物理层 2PC/柔性协议。配 XA 时如果项目同时挂了 Spring 的 JTA 管理（`spring-boot-starter-jta-atomikos`），会出现两个 TM 争抢 XAResource 注册，只能保留一层（13 篇 6.2 同类问题）。

# 2. XA 事务管理器

```java
public class XAShardingSphereTransactionManager
        implements ShardingSphereTransactionManager {

    private final XATransactionManager xaManager;  // Atomikos / Narayana

    @Override
    public void begin() {
        // 1. 获取所有参与的数据源
        Collection<String> xaDataSources = getParticipatedDataSources();

        // 2. 为每个数据源注册 XAResource
        for (String dsName : xaDataSources) {
            XADataSource xaDataSource = getXADataSource(dsName);
            XAResource xaResource = xaDataSource.getXAConnection()
                .getXAResource();

            // 3. 登记到 JTA 事务管理器
            xaManager.getTransaction().enlistResource(xaResource);
        }
    }

    @Override
    public void commit() throws TransactionException {
        try {
            xaManager.getTransaction().commit();
        } catch (Exception e) {
            throw new TransactionException(e);
        }
    }
}
```

实现层面有四处关键细节。其一，**XA 模式下数据源必须换成 XADataSource 实现**（`com.mysql.cj.jdbc.MysqlXADataSource` 等），YAML 里 `dataSourceClassName` 写 XA 类而非 HikariDataSource——普通池化 DataSource 拿不到 `XAResource`，事务静默退化为 LOCAL。其二，5.x 内置 Atomikos 与 Narayana 两个 SPI（Bitronix 已移除），Proxy 用 Narayana 需手动放依赖包。其三，**XA 下连接与事务一一绑定**：事务期间一个数据源固定占用一条 XA 连接，跨 8 库事务 = 8 条连接被锁到事务结束，池容量规划按"并发事务数 × 平均参与库数"重算。其四，prepare 阶段的阻塞锁：MySQL XA 的 prepare 即写 binlog 并持锁直到 commit，**长事务在 XA 下的锁持有时间是普通事务的灾难放大器**，事务瘦身优先级高于一切调参。

## 2.1 XA 的两阶段时序

```text
应用            SS事务管理器        ds0            ds1
 │ begin()          │                │              │
 │─────────────────▶│ xa start(xid0) │              │
 │                  │───────────────▶│              │
 │                  │ xa start(xid1) │              │
 │                  │────────────────┼─────────────▶│
 │ DML(路由到ds0/ds1)│ 各自 xa end/execute           │
 │ commit()         │                │              │
 │─────────────────▶│ xa prepare(xid0)│             │
 │                  │───────────────▶│ 持锁待命      │
 │                  │ xa prepare(xid1)│             │
 │                  │────────────────┼─────────────▶│
 │                  │ 两者均OK → xa commit(xid0/1)   │
 │                  │ 任一失败 → xa rollback 全部     │
```

# 3. 事务类型自动判定

```java
// ShardingSphere 内部的事务判断逻辑
public class TransactionContext {

    private TransactionType transactionType;

    public void determineTransactionType(
            RouteContext routeContext) {

        // 1. 如果 SQL 只涉及一个数据源 → 本地事务
        if (routeContext.getRouteUnits().size() == 1) {
            transactionType = TransactionType.LOCAL;
            return;
        }

        // 2. 如果涉及多个数据源 → XA 事务
        if (routeContext.getRouteUnits().size() > 1) {
            transactionType = TransactionType.XA;
            return;
        }
    }
}
```

精确语义是"**按当前事务内累计触达的数据源数**判定，而非单条 SQL 的路由数"：同一个 Connection 事务里，第一条 SQL 命中 ds0 时物理层就开启 ds0 本地事务，第二条命中 ds1 时，**已开始的本地事务无法原地升级为 XA**（xid 要注入已在途的连接）——ShardingSphere 的做法是把"事务升级"约束在事务开始阶段：显式声明 XA（规则默认或连接级指定）则每个首次触达的库都按 XA 协议起；声明 LOCAL 则多库就是多段独立提交。**最佳纪律是把跨库写拆到事务开始前就确定参与者**，或干脆让业务保证单事务单分片（分片键即租户/用户维度的红利）。

# 4. Seata AT 集成

```java
public class SeataATShardingSphereTransactionManager
        implements ShardingSphereTransactionManager {

    @Override
    public void begin() {
        // Seata 的 begin 由 GlobalTransactionScanner 处理
        // ShardingSphere 只需要确保数据源被 Seata 代理
        // DataSourceProxy 自动接管
    }

    @Override
    public void commit() throws TransactionException {
        // Seata TM 驱动提交，ShardingSphere 不干预
    }

    @Override
    public void rollback() throws TransactionException {
        // Seata TM 驱动回滚
    }
}
```

Seata 集成核心：ShardingSphere 的数据源被 Seata `DataSourceProxy` 包装，所有 SQL 自动记录 undo_log。

## 4.1 代理顺序：谁包谁

**正确嵌套是 Seata 在外、ShardingSphere 在内**：`Seata DataSourceProxy → ShardingSphereDataSource → 物理 XA/普通 DataSource`。业务拿到的连接是 Seata 代理连接，Seata 拦截 SQL 前后快照写 undo_log；ShardingSphere 在其内部分片路由。反过来包（SS 在外）时 Seata 看不到逻辑 SQL 的物理执行结果集，快照失效。**代码路径要点**：JDBC 模式在 Spring Boot 里用 `seata-spring-boot-starter` + `DataSourceProxy` 手工/自动包装 SS DataSource，并关闭 SS 自己的 `!TRANSACTION` XA 规则（两者互斥）；Seata 侧需排除对 `undo_log` 表的代理，且 undo_log 表要在**每个物理分片库**各建一份。Proxy 模式 5.5 提供 optional-plugins 的 Seata AT 接入，异构语言应用也能吃到柔性事务。

## 4.2 AT 与 XA 的语义差

| 维度 | XA (2PC) | Seata AT |
|------|----------|----------|
| 隔离性 | 真 ACID（prepared 持锁） | 默认读未提交全局，靠**全局锁**防脏写 |
| 锁持有 | 到 commit 全程 | 本地锁即放，全局锁只在 commit 校验瞬间 |
| 回滚依据 | 数据库引擎 | undo_log 反向 SQL |
| 悬挂/残留 | prepared 悬挂需恢复 | 分支注册失败留脏数据需补偿 |
| 吞吐损耗 | 高（约 -50%） | 中（快照+全局锁校验） |
| 要求 | 库支持 XA | 每库 undo_log 表、主键/唯一键必需 |

AT 的"脏读窗口"最易被业务忽略：其他全局事务能看到尚未全局提交的中间态（本地已提交）。金融扣减类链路要么用 XA，要么 AT + `@GlobalTransactional(isolation=READ_COMMITTED)`（行锁轮询代价高），要么业务上幂等重试兜底。

# 5. 事务拦截流程

```java
// ShardingSphere-JDBC 事务拦截
public class ShardingSphereConnection extends AbstractConnectionAdapter {

    @Override
    public void setAutoCommit(boolean autoCommit) throws SQLException {
        if (!autoCommit) {
            // 开启事务
            transactionManager.begin();
        }
        super.setAutoCommit(autoCommit);
    }

    @Override
    public void commit() throws SQLException {
        try {
            // 1. 先提交各个物理连接
            super.commit();
            // 2. 再调事务管理器提交（XA COMMIT）
            transactionManager.commit();
        } catch (Exception e) {
            transactionManager.rollback();
            throw e;
        }
    }
}
```

`ShardingSphereConnection` 在构造时并不持有物理连接，**连接是懒获取的**：首条 SQL 路由确定后才对涉及的每个数据源 getConnection 并登记进事务上下文。这带来一个 Spring 集成细节：`@Transactional` 开启时（setAutoCommit(false)）如果还没有任何 SQL 执行，事务上下文的参与者集合是空的，begin 只做类型标记；因此**事务类型必须在第一条跨库 SQL 之前确定**，与 3 节结论呼应。`close()` 时未提交的事务按 JDBC 规范由物理层各自回滚，但 XA prepared 残留不在该保证内——应用崩溃恢复靠 6 节。

# 6. XA 恢复（Crash Recovery）

```java
public class XARecovery {

    public void recover() {
        // 1. 查询所有 XADataSource 中的 prepared 事务
        for (XADataSource ds : xaDataSources) {
            XAResource xaResource = ds.getXAConnection()
                .getXAResource();
            Xid[] preparedXids = xaResource.recover(
                XAResource.TMSTARTRSCAN | XAResource.TMENDRSCAN);

            // 2. 对每个 prepared 事务决定 COMMIT 或 ROLLBACK
            for (Xid xid : preparedXids) {
                if (transactionLog.exists(xid)) {
                    // 有提交日志 → COMMIT
                    xaResource.commit(xid, false);
                } else {
                    // 无提交日志 → ROLLBACK
                    xaResource.rollback(xid);
                }
            }
        }
    }
}
```

**恢复的依据是 TM 的事务日志**：Atomikos 落 `logs/recovery`（tmlog）文件、Narayana 有 journal 目录，日志必须放持久化磁盘并随部署保留，容器化环境挂空 tmpfs 等于自毁恢复能力。数据库侧 prepared 事务会**无限期持有行锁**（MySQL 不超时释放），所以运维视角的悬挂处置顺序是：查 `XA RECOVER CONVERT XID` → 对照 TM 日志/应用状态判定 → 人工 `XA COMMIT/ROLLBACK 'gtrid','formatid'` → 解锁下游。xid 三元组里的 branch qualifier 在 SS 内部由连接序号生成，跨实例排查时以 gtrid（全局事务号）为线索检索各分支。生产必须配 `xa-lock-timeout`/`xa-timeout`（Narayana/Atomikos 均有对应属性），让 prepared 事务到点自动回收，别让 DBA 半夜手动清锁。

> **注意**：MySQL 5.7 之前 XA 不写 binlog（GTID 环境有提交丢失风险），5.7+ 才默认 `innodb_support_xa` + binlog 持久化 prepared；异构组合里 PG 的 prepared xact 上限 `max_prepared_transactions=0`（默认禁 XA）会直接报错，上线前逐库检查该参数（14 篇 4.2）。

# 7. LOCAL + 最终一致

不强求 2PC 的场景，主流解法是"本地事务 + 异步补偿"：

```java
// 本地消息表模式：分片 A 的业务与消息同片同库，本地事务保证原子
@Transactional
public void createOrder(Order order) {
    orderMapper.insert(order);                    // ds0 本地事务
    msgMapper.insert(TransferMsg.toInventory(order)); // 同库消息表
}

// 投递服务轮询消息表 → MQ → 库存服务消费（幂等表防重）→ 回执更新状态
```

ShardingSphere 在这套方案里的角色是**保证"业务写 + 消息写"落在同一分片同一本地事务**（消息表按同分片键冗余 order 的 user_id），之后的一致性由 MQ + 对账闭环完成，不需要 SS 的事务引擎参与。选型阶梯：单分片事务（最优，白拿 ACID）> XA（少量强一致跨片，如资金）> Seata AT（高并发跨片、可容忍读窗口）> 本地消息表/RocketMQ 事务消息（跨服务、接受秒级最终一致）。**阶梯越往下越依赖对账与幂等基建**，没有对账体系就上柔性事务等于裸奔。

# 8. 性能对比

```text
测试：2 个数据源，各 10 万行，批量 INSERT 100 条

| 模式       | TPS   | AVG RT |
|-----------|--------|--------|
| 本地事务   | 5000   | 20ms   |
| XA (Atomikos) | 2500 | 40ms |
| Seata AT   | 3800 | 26ms   |
```

数字背后的结构性原因：LOCAL 只有一次网络往返的提交；XA 多一轮 prepare fsync + 二阶段协调往返，且锁持有翻倍直接压缩并发度；AT 多了前后镜像查询与 undo_log 写入，但本地锁早释放换来并发度。经验配比——**跨库事务占比 <1% 的系统，收益大头来自把事务改成单库（路由优化），而不是在 XA 和 AT 之间精挑**：先用 sql-show/PREVIEW 审计参与者数量，用分片键设计把跨库事务消灭在建模阶段，才是这一篇真正的结论。

> **行业实践**：支付/库存等强一致域用"单分片键收敛事务边界 + XA 仅兜底极少数跨域操作"；营销/积分等最终一致域走消息表；Seata 在同时已有 SS 与微服务框架的技术栈里要防"双 TM"（Seata 全局事务 + SS XA 套娃），规约是**一个事务边界内只允许一种分布式协议**。

# 9. 小结

1. **默认类型是 LOCAL**：跨库不声明就不强一致，"用了 SS 就有分布式事务"是最大误解
2. **XA 三纪律**：数据源必须 XADataSource、事务期一库一连接（连接账重算）、TM 日志持久化
3. **Seata 嵌 SS 在外**，关闭 SS 自身 XA 规则，undo_log 逐分片建表；AT 有全局读未提交窗口
4. **事务类型在开始前定**：单片化（分片键建模）> 本地消息表/事务消息 > AT > XA，按一致性等级阶梯选
5. **prepared 悬挂是 XA 的运维税**：xa-timeout + 前向/反向恢复流程必须预先演练