
# 1. 概述

ShardingSphere 集成三种分布式事务方案，覆盖不同的一致性需求：

| 模式 | 一致性 | 性能 | 依赖 |
|------|--------|------|------|
| **XA** | 强一致 | 中 | Atomikos / Narayana / Bitronix |
| **Seata AT** | 最终一致 | 高 | Seata Server |
| **BASE** | 最终一致 | 高 | 无（内置） |

# 2. XA 事务

## 2.1 配置

```yaml
rules:
  - !TRANSACTION
    defaultType: XA
    providerType: Atomikos
```

## 2.2 Atomikos 集成

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-xa-core</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-xa-atomikos</artifactId>
</dependency>
```

## 2.3 使用

```java
// JDBC 模式
@Transactional
public void createOrder(Order order) {
    jdbcTemplate.update(
        "INSERT INTO t_order (user_id, status) VALUES (?, ?)",
        order.getUserId(), order.getStatus());
    jdbcTemplate.update(
        "INSERT INTO t_order_item (order_id, product_id) VALUES (?, ?)",
        order.getId(), order.getProductId());
    // 如果两个 INSERT 落在不同数据源 → ShardingSphere 自动升级为 XA
}
```

**XA 执行流程**：

```
1. XA START ──→ ds0.t_order_0   INSERT
2. XA START ──→ ds1.t_order_1   INSERT
3. XA END + XA PREPARE
4. 所有 PREPARE OK → XA COMMIT
   任一 FAIL → XA ROLLBACK
```

# 3. Seata AT 集成

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-base-seata-at</artifactId>
</dependency>
```

```yaml
rules:
  - !TRANSACTION
    defaultType: BASE
    providerType: Seata
```

```properties
# seata.conf
seata.tx-service-group=default_tx_group
seata.registry.type=nacos
seata.registry.nacos.server-addr=localhost:8848
seata.config.type=nacos
seata.config.nacos.server-addr=localhost:8848
```

**Seata AT 在 ShardingSphere 中的工作方式**：

```
@GlobalTransactional
public void createOrder(Order order) {
    // ShardingSphere 内部：
    // 1. Seata 代理数据源
    // 2. 执行 SQL 时自动记录 undo_log
    // 3. 分支注册到 Seata TC
    // 4. 全局提交/回滚
}
```

# 4. XA vs Seata AT 选择

| 维度 | XA | Seata AT |
|------|-----|---------|
| **一致性** | 强一致 | 最终一致 |
| **隔离性** | 读取已提交 | 读取未提交（默认） |
| **依赖** | JTA 实现（Atomikos等） | Seata Server |
| **性能** | 中（长时间锁） | 高 |
| **回滚** | 数据库原生 | undo_log 自动补偿 |
| **跨语言** | 不支持（JDBC 模式） | 不支持（JDBC 模式） |
| **适用场景** | 金融、支付 | 电商、通用 |

# 5. BASE 事务

ShardingSphere 也支持 BASE（基本可用、柔性状态、最终一致）事务，基于 Saga 模式：

```java
// BASE 事务强调最终一致性，允许中间状态
// 适合长事务和无法使用 XA 的场景
```

# 6. Proxy 模式的事务

```
Proxy 模式下，ShardingSphere 作为数据库代理：
  - 客户端发送 BEGIN/COMMIT/ROLLBACK
  - Proxy 内部转换为 XA 事务
  - 对支持 XA 的数据库（MySQL 5.7+, PG 9.5+）透明
```

# 7. 事务隔离级别

```java
// 设置事务隔离级别
connection.setTransactionIsolation(
    Connection.TRANSACTION_READ_COMMITTED);
```

**注意**：跨数据源时，隔离级别由各个后端数据库独立保证。XA 模式下隔离性更好，Seata AT 默认为读未提交。

# 8. 分布式事务最佳实践

1. **优先单分片事务**：WHERE 条件包含分片键 → 单库事务，无需分布式
2. **XA 适用**：严格一致性要求的金融场景
3. **Seata AT 适用**：高并发，可容忍最终一致的电商场景
4. **事务尽量小**：减少锁持有时间，降低死锁风险
5. **超时设置**：合理设置事务超时，避免长时间悬挂
