
# 1. AT 模式原理

AT（Automatic Transaction）模式是 Seata 的**默认模式**，对业务代码**零侵入**，通过代理数据源自动生成 undo_log 实现回滚。

```
Phase 1 (执行 + 记录 undo_log):
  ┌──────────────────────────────────┐
  │ 1. DataSourceProxy 拦截 SQL      │
  │ 2. 执行前：查询前置镜像 (before)   │
  │ 3. 执行业务 SQL                   │
  │ 4. 执行后：查询后置镜像 (after)    │
  │ 5. 生成 undo_log (before + after)│
  │ 6. 分支注册到 TC                  │
  │ 7. 本地事务提交                    │
  └──────────────────────────────────┘

Phase 2 (提交 或 回滚):
  ├── Commit: TC 通知 → 异步删除 undo_log
  └── Rollback: TC 通知 → 根据 undo_log 反向补偿
```

# 2. undo_log 自动补偿机制

## 2.1 前置镜像与后置镜像

```sql
-- 原始 SQL
UPDATE account SET balance = balance - 50 WHERE user_id = 1;

-- 执行前：查询前置镜像 (before image)
SELECT id, user_id, balance FROM account WHERE user_id = 1;
-- → {id: 1, user_id: 1, balance: 100}

-- 执行业务 SQL（balance: 100 → 50）

-- 执行后：查询后置镜像 (after image)
SELECT id, user_id, balance FROM account WHERE user_id = 1;
-- → {id: 1, user_id: 1, balance: 50}

-- undo_log 记录
{
  "xid": "192.168.1.100:8091:2039718144",
  "branchId": 2039718145,
  "beforeImage": { "rows": [ { "fields": [ {"name":"id","value":1}, {"name":"balance","value":100} ] } ] },
  "afterImage":  { "rows": [ { "fields": [ {"name":"id","value":1}, {"name":"balance","value":50}  ] } ] },
  "sqlType": "UPDATE"
}
```

## 2.2 反向补偿 SQL

```sql
-- 回滚时根据 undo_log 生成反向 SQL
-- before balance=100, after balance=50
-- → 反向 SQL: UPDATE account SET balance = 100 WHERE user_id = 1

-- 带条件校验（防止脏写）:
UPDATE account SET balance = 100
WHERE user_id = 1
  AND balance = 50;  -- 校验 after image，防止被其他事务修改
```

# 3. DataSourceProxy — 数据源代理

```java
// Seata 自动代理数据源
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource(DataSourceProperties properties) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(properties.getUrl());
        dataSource.setUsername(properties.getUsername());
        dataSource.setPassword(properties.getPassword());

        // Seata 包装
        return new DataSourceProxy(dataSource);
    }
}

// Spring Boot 自动配置简化：
// seata.data-source-proxy-mode=AT
```

## 3.1 SQL 拦截流程

```java
// DataSourceProxy 的核心：PreparedStatementProxy
public class PreparedStatementProxy extends AbstractPreparedStatementProxy {

    @Override
    public int executeUpdate() throws SQLException {
        // 1. 识别 SQL 类型（INSERT/UPDATE/DELETE）
        SQLRecognizer recognizer = SQLVisitorFactory.get(targetSQL, dbType);
        
        // 2. 执行前置查询（SELECT FOR UPDATE + 生成 before image）
        TableRecords beforeImage = executor.beforeImage(recognizer);
        
        // 3. 执行业务 SQL
        int result = statement.executeUpdate();
        
        // 4. 执行后置查询（生成 after image）
        TableRecords afterImage = executor.afterImage(recognizer, beforeImage);
        
        // 5. 构建 undo_log
        undoLogManager.buildUndoItem(beforeImage, afterImage, recognizer);
        
        return result;
    }
}
```

# 4. 全局锁 — 写隔离

AT 模式在 Phase 1 本地事务提交后释放了数据库锁，这可能导致并发事务之间的**脏写**问题。Seata 通过**全局锁**解决：

```
事务A                       事务B
  │                           │
  │ UPDATE account SET        │
  │   balance = balance - 50  │
  │ WHERE user_id = 1         │
  │   (本地事务提交)            │
  │                           │
  │ 获取全局锁                  │
  │ (lock_table 插入记录)      │
  │                           │
  │                           │ UPDATE account SET
  │                           │   balance = balance - 30
  │                           │ WHERE user_id = 1
  │                           │   (本地事务提交)
  │                           │
  │                           │ 尝试获取全局锁 → 等待
  │                           │   (已被事务A持有)
  │                           │
  │ 全局提交 → 释放全局锁       │
  │                           │
  │                           │ 获取全局锁 → 执行
```

**全局锁存储**：TC Server 的 `lock_table`：

```sql
CREATE TABLE `lock_table` (
  `row_key` VARCHAR(128) NOT NULL,    -- 行键（如 "account:1"）
  `xid`     VARCHAR(128) NOT NULL,    -- 持有锁的事务
  `branch_id` BIGINT NOT NULL,
  PRIMARY KEY (`row_key`)
);
```

# 5. 读隔离

Seata AT 模式默认是**读未提交**（Read Uncommitted）隔离级别。因为 Phase 1 本地事务已提交，其他事务可以读到未决全局事务的数据。

如需**读已提交**，可使用 `SELECT FOR UPDATE` 语句，Seata 会代理为全局锁检查：

```java
// 读已提交：代理为 SELECT FOR UPDATE + 全局锁校验
@GlobalLock   // 仅加锁，不参与全局事务
@Select("SELECT * FROM account WHERE user_id = #{userId} FOR UPDATE")
Account selectForUpdate(Long userId);
```

# 6. AT 模式优缺点

| 优点 | 缺点 |
|------|------|
| 对业务零侵入，开启代理即可 | 需要数据库支持本地事务 |
| 自动回滚，无需手动补偿 | 依赖 undo_log 表 |
| 性能较好（本地事务 + 异步提交） | 默认读未提交 |
| 适用于绝大多数业务场景 | 不支持跨数据库类型的 INSERT...SELECT |

# 7. 适用场景

- ✓ 基于关系型数据库的传统业务（订单、库存、账户）
- ✓ 对性能有一定要求的场景
- ✓ 不想修改业务代码的存量系统改造
- ✗ 涉及 NoSQL/消息队列等非数据库资源
- ✗ 需要强一致读的场景（需额外配置 `@GlobalLock`）
