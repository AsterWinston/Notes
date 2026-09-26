
# 1. 概述

Mycat2 支持 **XA 分布式事务**，在跨分片写入时保证数据一致性。

```
Mycat2 (TM)
    │
    │ XA START
    ├──────────→ db0 ── INSERT orders
    ├──────────→ db1 ── INSERT orders
    │
    │ XA END + XA PREPARE
    ├──────────→ db0 ── PREPARE OK
    ├──────────→ db1 ── PREPARE OK
    │
    │ XA COMMIT (all OK) 或 XA ROLLBACK (any FAIL)
    ├──────────→ db0 ── COMMIT
    └──────────→ db1 ── COMMIT
```

# 2. 两阶段提交流程

```
Phase 1 (Prepare):
  1. Mycat2 向每个参与的 MySQL 发送 XA START
  2. 各 MySQL 执行 SQL
  3. Mycat2 发送 XA END + XA PREPARE
  4. MySQL 将事务写入 binlog (prepare 状态)
  5. 返回 PREPARE OK

Phase 2 (Commit):
  1. 所有 MySQL PREPARE OK → XA COMMIT
  2. 任一 MySQL PREPARE FAIL → XA ROLLBACK

故障恢复：
  MySQL 重启后检查 binlog 中的 prepared 事务
  → 向 Mycat2 查询该 xid 的最终状态
  → COMMIT 或 ROLLBACK
```

# 3. 配置

```json
{
  "schemas": {
    "order_db": {
      "xa": {
        "enabled": true,
        "timeout": 30000,
        "retryCount": 3
      }
    }
  }
}
```

```yaml
# mycat.yml
server:
  xa:
    transactionTimeout: 30000
    retryCount: 3
    recovery:
      enabled: true
      interval: 60000
```

# 4. 事务使用

```sql
-- 显式事务
BEGIN;
INSERT INTO orders(id, user_id, amount) VALUES (1, 100, 50.0);
INSERT INTO order_items(id, order_id, product_id) VALUES (1, 1, 200);
COMMIT;

-- 如果任一 INSERT 失败 → ROLLBACK
-- 如果两个 INSERT 在不同分片 → XA 事务自动介入

ROLLBACK;
```

```java
// JDBC 事务
@Transactional
public void createOrder(Order order) {
    jdbcTemplate.update(
        "INSERT INTO orders(id, user_id, amount) VALUES (?,?,?)",
        order.getId(), order.getUserId(), order.getAmount()
    );
    jdbcTemplate.update(
        "INSERT INTO order_items(id, order_id, product_id) VALUES (?,?,?)",
        item.getId(), order.getId(), item.getProductId()
    );
    // 如果两个 INSERT 路由到不同分片 → Mycat2 自动升级为 XA
}
```

# 5. 单分片 vs 跨分片

```
单分片事务：
  INSERT INTO orders WHERE id = 1
  INSERT INTO order_items WHERE id = 1
  → 两个语句都落在 db0 → MySQL 本地事务即可

跨分片事务：
  INSERT INTO orders WHERE id = 1  → db0
  INSERT INTO orders WHERE id = 2  → db1
  → 跨两个 MySQL 实例 → XA 事务
```

# 6. 事务隔离级别

| 隔离级别 | 支持 | 说明 |
|---------|------|------|
| READ UNCOMMITTED | ✓ | 最低 |
| READ COMMITTED | ✓ | 默认 |
| REPEATABLE READ | ✓ | MySQL 默认 |
| SERIALIZABLE | ✓ | 最高 |

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
-- ... 在 READ COMMITTED 下执行
COMMIT;
```

# 7. 全局表的事务

```sql
-- 全局表的 UPDATE 涉及所有分片
UPDATE dict_province SET name = '新名称' WHERE code = '110000';
-- → Mycat2 开启 XA 事务
-- → 对所有分片的 dict_province 执行 UPDATE
-- → 任一失败 → 全部回滚
```

# 8. XA 恢复机制

```
场景：Mycat2 在 Phase 2 执行过程中宕机

Mycat2 重启后：
  1. 检查本地日志中的 prepared 事务
  2. 连接各 MySQL 查询 XA RECOVER
  3. 找出状态不一致的事务
  4. 根据日志决定 COMMIT 或 ROLLBACK
  5. 完成恢复
```

```sql
-- MySQL 端查询 XA 事务状态
XA RECOVER;

-- 提交或回滚
XA COMMIT 'xid-xxx';
XA ROLLBACK 'xid-xxx';
```

# 9. XA vs Seata AT 在 Mycat2 中的对比

| 维度 | Mycat2 XA | Seata AT（配合 Mycat2） |
|------|----------|------------------------|
| 集成方式 | 内置，无需额外组件 | 需部署 Seata Server |
| 性能 | 中（长时间锁） | 中高（全局锁，更轻量） |
| 一致性 | 强一致 | 最终一致 |
| undo_log | 不需要 | 需要 |
| 适用场景 | 金融级强一致 | 高并发，可容忍最终一致 |

# 10. 注意事项

1. **MySQL 需开启 XA 支持**：`innodb_support_xa = ON`（默认开启）
2. **事务超时控制**：避免长时间持锁
3. **XID 格式**：Mycat2 自动管理，无需手动指定
4. **性能损耗**：XA 事务相比单库事务性能下降约 20-40%
5. **避免大事务**：跨分片大事务会导致锁持有时间过长
