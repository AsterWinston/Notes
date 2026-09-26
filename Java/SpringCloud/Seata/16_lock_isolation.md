
# 1. 为什么需要全局锁

AT 模式在 Phase 1 本地事务提交后**释放了数据库行锁**。这意味着在 Phase 2 执行之前，其他事务可以修改同一行数据，导致**脏写**。

```
TxA (全局事务)                     TxB (普通事务)
  │                                  │
  │ UPDATE balance: 100 → 50        │
  │ (本地事务提交，释放行锁)          │
  │                                  │
  │                                  │ UPDATE balance: 50 → 10
  │                                  │ (正常执行！因为行锁已释放)
  │                                  │
  │ Phase 2: ROLLBACK               │
  │ undo_log 反向 SQL:              │
  │   UPDATE balance SET 100        │
  │   WHERE balance = 50            │
  │                                 │
  │ ✗ 0 rows affected!             │
  │   因为 balance 已被 TxB 改为 10  │
  │   回滚失败！                     │
```

Seata 通过**全局锁**防止这种问题。

# 2. 全局锁机制

```
Phase 1 结束后，TM 先获取全局锁，再提交本地事务：

TxA (全局事务)                     TC Server (lock_table)
  │                                  │
  │ Phase 1: INSERT order            │
  │ Phase 1: UPDATE balance          │
  │                                  │
  │ 申请全局锁                        │
  │ ──── acquireLock(xid, rowKey)──→  │
  │ ←──── LOCK_ACQUIRED ────────────  │
  │                                  │ INSERT INTO lock_table
  │ 本地事务提交                      │   (row_key='account:1',
  │                                  │    xid='TxA_xid')
  │                                  │
  │                                  │
TxB (另一个全局事务)                  │
  │ Phase 1: UPDATE balance          │
  │ 申请全局锁                        │
  │ ──── acquireLock(xid, rowKey)──→  │
  │ ←──── LOCK_WAIT ──────────────── │ 检查到已存在锁
  │ (等待/超时/回滚)                   │
  │                                  │
  │                                  │
TxA: Phase 2 COMMIT                  │
  │ 释放全局锁                        │
  │                                  │ DELETE FROM lock_table
  │                                  │   WHERE xid='TxA_xid'
  │                                  │
TxB: 获得全局锁 → 执行                 │
```

# 3. lock_table 结构

```sql
CREATE TABLE `lock_table` (
  `row_key`        VARCHAR(128) NOT NULL,
  `xid`            VARCHAR(128) NOT NULL,
  `transaction_id` BIGINT,
  `branch_id`      BIGINT       NOT NULL,
  `resource_id`    VARCHAR(256),
  `table_name`     VARCHAR(32),
  `pk`             VARCHAR(36),
  `status`         TINYINT      NOT NULL DEFAULT 0 COMMENT '0:locked,1:rollbacking',
  `gmt_create`     DATETIME,
  `gmt_modified`   DATETIME,
  PRIMARY KEY (`row_key`),
  KEY `idx_xid` (`xid`)
) ENGINE=InnoDB;
```

**row_key 格式**：`{resourceId}^^^{tableName}^{pk}`

示例：`jdbc:mysql://localhost:3306/order_db^^^account^1`

# 4. 全局锁获取源码

```java
// ConnectionProxy 提交时获取全局锁
private String buildLockKey(ConnectionProxy connectionProxy) {
    StringBuilder lockKeys = new StringBuilder();
    for (SQLUndoLog undoLog : connectionProxy.getUndoItems()) {
        // 从 afterImage 提取主键
        TableRecords afterImage = undoLog.getAfterImage();
        for (Row row : afterImage.getRows()) {
            // 格式: tableName;pkValue1,pkValue2;pkName1,pkValue1
            lockKeys.append(undoLog.getTableName()).append(";");
            for (Field field : row.primaryKeys()) {
                lockKeys.append(field.getValue()).append(",");
            }
            // ... 完整锁键构建
        }
    }
    return lockKeys.toString();
}

// 向 TC 注册分支（携带锁键）
private Long registerBranch(String lockKeys) {
    return DefaultResourceManager.get().branchRegister(
        BranchType.AT,
        getResourceId(),
        null,    // clientId
        xid,
        null,    // applicationData
        lockKeys // ← 全局锁键
    );
}
```

# 5. 全局锁超时与重试

```java
// RM 端锁获取配置
seata:
  client:
    lock:
      retry-interval: 10          # 重试间隔(ms)
      retry-times: 30             # 重试次数（共 300ms）
      retry-policy-branch-rollback-on-conflict: true

// 锁获取失败的处理
// 1. 重试 retry-times 次，每次间隔 retry-interval ms
// 2. 全部失败 → 本地事务回滚 → 通知 TC 分支失败
// 3. TC 触发全局回滚
```

# 6. 读隔离级别

## 6.1 默认：读未提交

AT 模式下，Phase 1 本地事务提交后数据对外可见，其他事务可以读到未决全局事务的数据：

```sql
-- TxA: Phase 1
UPDATE account SET balance = balance - 50 WHERE user_id = 1;
-- 本地事务提交，balance = 50（但全局事务未提交）

-- TxB: 读
SELECT balance FROM account WHERE user_id = 1;
-- → 50（读到了未决事务的中间值）
```

**适用场景**：大多数业务场景能接受。

## 6.2 SELECT FOR UPDATE + @GlobalLock

需要读已提交时，使用 `@GlobalLock` 注解 + `SELECT FOR UPDATE`：

```java
@GlobalLock  // 仅获取全局锁，不参与全局事务
@Select("SELECT balance FROM account WHERE user_id = #{userId} FOR UPDATE")
Account selectForUpdate(Long userId);
```

**执行流程**：
1. 执行 SQL 前，检查 `lock_table` 是否有该行的全局锁
2. 若有锁 → 等待或失败
3. 若无锁 → 执行查询（DB 保证当前读）

## 6.3 读隔离对比

| 方式 | 隔离级别 | 性能 | 使用复杂度 |
|------|---------|------|-----------|
| 默认读 | 读未提交 | 高 | 零配置 |
| @GlobalLock + FOR UPDATE | 读已提交 | 中（需获取全局锁） | 注解即可 |
| 全局事务读 | 读已提交 | 低（需全局锁） | 参与全局事务 |

# 7. 死锁检测

```java
// TC Server 检测全局锁死锁
public class LockManager {
    
    @Scheduled(fixedDelay = 1000)
    public void detectDeadLock() {
        // 1. 查询 lock_table 中等待超过阈值的锁
        List<LockRow> timeoutLocks = lockStore.queryTimeoutLocks();
        
        // 2. 分析是否存在环形等待
        for (LockRow lock : timeoutLocks) {
            if (isDeadLock(lock)) {
                // 3. 选择 young xid 回滚（打破死锁）
                String youngXid = selectYoungXid(lock);
                GlobalSession session = sessionHolder
                    .findGlobalSession(youngXid);
                if (session != null) {
                    core.rollback(youngXid);
                }
            }
        }
    }
}
```

# 8. 全局锁 vs 数据库锁

| 维度 | 全局锁（Seata） | 数据库行锁 |
|------|---------------|-----------|
| 范围 | 跨数据库 | 单数据库 |
| 持有时间 | Phase 1 提交 → Phase 2 完成 | 事务内 |
| 实现 | TC lock_table | InnoDB 行锁 |
| 粒度 | 行级 | 行级 |
| 超时 | 可配置 | 数据库 innodb_lock_wait_timeout |
