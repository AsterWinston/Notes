分库分表之后，"一个业务动作写两个库"从行内事务变成了跨库事务——这是分片方案最贵的代价。ShardingSphere 的立场是：**不把分布式事务做进内核，而是统一抽象后接入成熟实现**（本地事务、Atomikos/Narayana 的 XA、Seata 的 BASE），由 `!TRANSACTION` 规则声明默认类型。本篇给出 5.5.x 的事务类型全景、XA 与 Seata AT 的配置与执行机制、Seata AT 在分片场景的具体坑、以及"能不用分布式事务就不用"的最终一致设计。

# 1. 事务类型全景

ShardingSphere 5.x 官方事务类型为 **LOCAL / XA / BASE** 三种（源码 `TransactionType` 枚举只有这三个值）：

| 类型 | 一致性 | 协议/机制 | 依赖 | 性能 |
|------|--------|----------|------|------|
| **LOCAL** | 最终一致（不保证跨库原子） | 各分库各自提交，无协调 | 无 | 最高 |
| **XA** | 强一致 | 两阶段提交（2PC） | Atomikos / Narayana / Bitronix（JTA 实现） | 中，锁持有时间长 |
| **BASE** | 最终一致 | 补偿回滚 | Seata Server | 高 |

> **注意**：部分中文资料提到的 "HBL" 并不是 ShardingSphere 的事务类型——源码 `TransactionType` 枚举自始至终只有 `LOCAL, XA, BASE` 三个值。与之相近的真实历史概念是 Sharding-JDBC 1.x–3.x 时代的 **BEST_EFFORTS**（尽力而为型：逐库提交、失败仅记日志不回滚），它在 4.x 事务重构中被移除；"HBL" 多半是对这类"尽力而为"叫法的以讹传讹。看教程先对版本。

三种类型的共同底座是 **`TransactionManager` SPI + 连接组（ConnectionGroup）**：无论哪种类型，ShardingSphere 把一次事务涉及的所有物理连接收进事务管理器统一管理；差别只在提交策略——LOCAL 逐连接 `commit()`，XA 走 `xaStart/xaEnd/xaPrepare/xaCommit` 两步，BASE 把分支事务注册给 Seata TC 由其驱动全局提交或反向补偿。

# 2. XA 事务

## 2.1 配置

```yaml
rules:
  - !TRANSACTION
    defaultType: XA
    providerType: Atomikos     # Atomikos / Narayana / Bitronix
```

Maven 依赖（JDBC 形态按需引入其一）：

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-xa-core</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-xa-atomikos</artifactId>
</dependency>
<!-- 或 shardingsphere-transaction-xa-narayana -->
```

## 2.2 使用与执行流程

```java
@Transactional
public void createOrder(Order order) {
    // 两条 INSERT 若落不同分库，自动升级为跨库事务
    jdbcTemplate.update("INSERT INTO t_order (user_id, status) VALUES (?, ?)",
            order.getUserId(), order.getStatus());
    jdbcTemplate.update("INSERT INTO t_order_item (order_id, product_id) VALUES (?, ?)",
            order.getId(), order.getProductId());
}
```

XA 的执行时序（两阶段提交）：

```
1. XA START ──→ ds0.t_order_0   INSERT
2. XA START ──→ ds1.t_order_1   INSERT
3. 本地业务完成 → 各连接 XA END + XA PREPARE
4. 全部 PREPARE 成功 → XA COMMIT
   任一 PREPARE 失败 → XA ROLLBACK（全链回滚）
```

对业务仍是 `@Transactional` 语义，ShardingSphere 把 JDBC 连接包装成 XAConnection 交给 JTA 管理器协调。**只有单分片时 XA 不生效**：ShardingSphere 检测事务是否真的跨库，单库直接降级本地事务，避免无谓 2PC——这是它比"全部走 JTA"的传统方案性能好的关键细节。

> **注意**：XA 的强一致建立在**数据库支持 XA 协议**上（MySQL 5.7+/8.0 支持，但**事务型表引擎必须 InnoDB**；云厂商某些中间件/代理层会阉割 XA 语句，上线前用 `XA PREPARE` 冒烟验证）。XA 第二阶段提交前锁不释放，跨分片 XA 事务把 N 个库的行锁寿命拉长到全局提交为止，**高并发下热点行死锁概率显著上升**。

# 3. BASE 事务：Seata AT 集成

## 3.1 配置

```yaml
rules:
  - !TRANSACTION
    defaultType: BASE
    providerType: Seata
```

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-base-seata-at</artifactId>
</dependency>
```

Seata 客户端参数（`seata.conf`，ShardingSphere 会读取）：

```properties
seata.tx-service-group=default_tx_group
seata.service.vgroup-mapping.default_tx_group=default
seata.service.grouplist.default=127.0.0.1:8091
seata.registry.type=nacos
seata.registry.nacos.server-addr=localhost:8848
seata.config.type=nacos
seata.config.nacos.server-addr=localhost:8848
```

## 3.2 工作方式

```
@GlobalTransactional
public void createOrder(Order order) {
    // ShardingSphere + Seata 内部链路：
    // 1. Seata 代理数据源（RootContext 携带 XID）
    // 2. 执行 SQL 前记录前镜像、后镜像 → undo_log
    // 3. 各分支注册到 Seata TC
    // 4. 全局提交：删 undo_log；全局回滚：按 undo_log 反向补偿
}
```

**Seata AT 在分片场景的坑**，逐个说：

1. **undo_log 表必须在每个真实分库存在**：Seata 的补偿日志按库建表，分片环境意味着 `demo_ds_0..n` 每个物理库都要建 `undo_log`，漏一个库、回滚到该分支时报"undo_log not found"。
2. **业务表要有主键/唯一索引**：AT 模式靠主键定位前后镜像行，分片主键若用数据库自增+步长要小心镜像解析；SNOWFLAKE 主键场景无障碍。
3. **全局锁与热点**：AT 默认在 TC 上加全局锁防脏写，秒杀类热点行更新会退化成全局锁排队，性能不如"库存分桶"设计。
4. **隔离级别默认读未提交（全局视角）**：分支本地提交后全局未提交前，其他分支能读到"中间态"数据；需要读已提交要开 AT 的 `global` 锁或 SELECT FOR UPDATE，性能再折损。
5. **ShardingSphere 与 Seata 的 SQL 解析重叠**：Seata 也要解析 SQL 生成镜像，双层改写偶发版本兼容问题（尤其 Seata 大版本升级），**升级任一方都要回归跨库回滚用例**。

> **设计争议**：既然 ShardingSphere 有 XA，为什么还要接 Seata？2PC 把锁寿命交给最慢分片，互联网写场景吞吐损失不可接受；AT 用"先提交+补偿"换性能，代价是中间态可见与 undo_log 的写放大（一次 UPDATE 至少多两行镜像 I/O）。金融账务类选 XA/TCC，电商交易类选 AT/Saga，不存在通吃。

# 4. LOCAL 事务：多数人的正确答案

LOCAL 是默认类型，也是最常被滥用的类型——需要澄清它的真实语义：

```yaml
rules:
  - !TRANSACTION
    defaultType: LOCAL
```

- 跨分片 INSERT 多库时，LOCAL **不保证原子性**：每个分库各自 commit，中途失败就是部分成功。
- 但 **单分片事务完全可靠**：ShardingSphere 把单库内的语句归入同一物理连接，普通 ACID 不受影响。

**设计考量**：

- **路由收敛优先于事务方案**：让 95% 以上的事务落在单分片内（分片键同源、绑定表），LOCAL 就够；为剩余 5% 引入 XA/Seata 的复杂度、故障面、性能税通常不值。
- 真正的跨库动作（下单扣库存、转账）优先走**业务消息模型**（第 6 节），分布式事务方案只作为兜底或低频场景。

# 5. Proxy 模式下的事务

Proxy 形态下客户端发的是标准 MySQL 协议事务语句，转换由服务端完成：

```
客户端                          Proxy                        后端库
  BEGIN     ────►  开启事务上下文（LOCAL/XA）
  INSERT×N  ────►  按规则路由多分片执行
  COMMIT    ────►  LOCAL: 逐连接 commit
                   XA:   xaPrepare → xaCommit
  ROLLBACK  ────►  反向执行
```

- `SET autocommit`、隔离级别、只读事务等连接级语义由 Proxy 维护并透传到后端；**跨库时隔离级别由各后端库独立保证**，ShardingSphere 不伪造全局快照。
- Proxy 上 Seata AT 有独立可选插件（server 端代理，客户端无需嵌 Seata），XA 需在 Proxy 侧添加 Narayana 依赖包。
- 客户端长事务 + Proxy 后端连接组：一个 BEGIN 占住 N 个后端连接直到 COMMIT，Proxy 连接池容量要按事务并发数而非 QPS 估算。

# 6. 最终一致设计：事务之外

成熟的分片架构里，"跨库强一致事务"出现的次数远少于预期，常见替代：

| 模式 | 机制 | 典型场景 |
|------|------|---------|
| **本地消息表** | 业务动作+消息同库落表，投递重试 | 下单成功后发通知/积分 |
| **事务消息**（RocketMQ half message） | MQ 预写+回查 | 库存扣减、订单状态广播 |
| **Saga 编排** | 正向链+补偿链 | 长流程（履约、售后） |
| **TCC**（Seata TCC） | Try/Confirm/Cancel 手工实现 | 账务、券核销 |

```
下单事务（单分片 LOCAL）
  │  同事务写：order + 本地消息表
  ▼
投递服务扫消息表 → MQ → 库存/积分服务各自消费（幂等）
失败重试 + 对账任务兜底
```

**设计考量**：

- 判断标准不是"要不要一致性"而是"**多快要求一致 + 中间态能不能被业务吸收**"：能展示"处理中"状态的就用消息/Saga，秒级都不允许错的就地 XA。
- **对账是最终一致的最后一道防线**：任何消息方案都必须有日终对账把漏网差错修正，否则"最终一致"只是"最终没人知道"。

# 7. 事务隔离级别与超时

```java
// 显式设置隔离级别（跨库时逐连接下发）
connection.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
```

- **XA 模式下各库隔离级别仍独立生效**，2PC 只保证原子性，不保证跨库快照；RR 隔离下跨库范围锁的间隙不一致问题依然存在。
- `!TRANSACTION` 的 `timeoutSeconds` 与 Seata 的 `timeout` 要按链路最慢分支留量，**超时悬挂**（XA PREPARE 成功但协调者挂了）需要启动期恢复扫描，Atomikos 日志/Narayana 恢复模块要持久化在稳定盘。

# 8. 最佳实践

1. **优先单分片事务**：WHERE 带分片键 → 单库 ACID，无需分布式方案（这是分片键设计的回报）。
2. **XA 适用**：强一致、低并发、跨库写少的场景（金融清结算）。
3. **Seata AT 适用**：高并发、可容忍毫秒级中间态的电商交易。
4. **BASE 之外先想消息**：通知、积分、风控走最终一致，不占交易链路。
5. **事务尽量小**：跨库事务中不做 RPC、不加分布式锁，缩短锁持有时间。
6. **超时与悬挂预案**：设合理 timeout，压测覆盖"协调者宕机 + 恢复"用例。

# 9. 小结

| 方案 | 一致性 | 一句话选择理由 |
|------|--------|---------------|
| LOCAL | 单分片强、跨分片无保证 | 95% 事务应收敛到单分片，默认选它 |
| XA | 强一致 | 低频跨库写 + 不能错账；锁寿命代价大 |
| BASE（Seata AT） | 最终一致 | 高频跨库写可容忍中间态；undo_log 每分库必备 |
| 消息/Saga/TCC | 最终一致 | 跨服务长流程的事实标准，必须配对账 |

下一篇（9_distsql）看 Proxy 管理面的语言：DistSQL 如何在线完成这一切配置与变更。
