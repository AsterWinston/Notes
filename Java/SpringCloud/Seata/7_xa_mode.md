
# 1. XA 协议概述

XA 是 X/Open DTP 定义的**分布式事务处理标准**，由事务管理器（TM）和资源管理器（RM）之间通过两阶段提交协议协调。

Seata 的 XA 模式利用数据库原生 XA 能力，无需 Seata 额外维护 undo_log。

# 2. XA 两阶段提交

```
Phase 1 (Prepare):
  ┌────────────────────────────────┐
  │ TM → 各 RM: XA START           │
  │ RM → TM:   执行 SQL, XA END     │
  │ TM → 各 RM: XA PREPARE         │
  │ RM → TM:   PREPARE OK / FAIL   │
  └────────────────────────────────┘

Phase 2 (Commit/Rollback):
  ├── All PREPARE OK → XA COMMIT
  └── Any FAIL → XA ROLLBACK
```

**关键特征**：Prepare 阶段后，数据库锁**持续持有**直到 Phase 2 完成。这与 AT 模式（Phase 1 提交即释放锁）完全不同。

# 3. Seata XA 集成

## 3.1 配置

```yaml
seata:
  data-source-proxy-mode: XA    # 使用 XA 模式

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/order_db?useUnicode=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    # XA 模式必须使用支持 XA 的驱动
```

## 3.2 业务代码

```java
// XA 模式无需修改业务代码，与 AT 模式完全一致
@GlobalTransactional
public void createOrder(Order order) {
    orderMapper.insert(order);                  // 分支1
    accountClient.debit(userId, amount);        // 分支2 (RPC → 远程分支)
    inventoryClient.deduct(productId, quantity);// 分支3
}
```

**与 AT 的区别**：
- AT：需要 `undo_log` 表，Phase 1 提交释放 DB 锁
- XA：不需要 `undo_log` 表，Phase 1 Prepare 后 DB 锁持续持有直到 Phase 2

# 4. XA 模式执行流程

```
Seata XA 流程：
                                                    
TM                     TC                  RM (MySQL)
 │                      │                    │
 │── begin(xid) ────→   │                    │
 │                      │                    │
 │                      │  执行 SQL           │
 │                      │ ←──XA START──────── │
 │                      │ ←──XA END────────── │
 │                      │ ←──XA PREPARE────── │
 │                      │ ←──PREPARE OK────── │
 │                      │                    │
 │── commit(xid) ──→    │                    │
 │                      │ ←──XA COMMIT─────── │
 │                      │ ←──COMMIT OK─────── │
```

# 5. XA vs AT 对比

| 维度 | XA 模式 | AT 模式 |
|------|---------|---------|
| **锁持有时间** | Prepare 后持续持锁（长） | Phase 1 提交即释放（短） |
| **并发性能** | 低（长时间锁） | 高（锁释放快） |
| **一致性** | 强一致 | 最终一致（读未提交） |
| **undo_log 表** | 不需要 | 需要 |
| **数据库要求** | 支持 XA 协议 | 支持本地事务即可 |
| **跨数据库类型** | 有限（依赖 XA 驱动） | 更灵活 |
| **SQL 支持** | 所有 SQL | 支持 INSERT/UPDATE/DELETE |
| **故障恢复** | 数据库 XA 恢复 | Seata undo_log 补偿 |

# 6. MySQL 中的 XA 事务

```sql
-- XA 事务示例
XA START 'xid-001';
UPDATE account SET balance = balance - 50 WHERE user_id = 1;
XA END 'xid-001';
XA PREPARE 'xid-001';
-- 此时数据已锁定，但未最终提交
-- 其他事务无法修改 user_id=1 的记录

-- Phase 2: 提交或回滚
XA COMMIT 'xid-001';   -- 或 XA ROLLBACK 'xid-001';
```

**MySQL XA 的限制**：
- XA 事务与本地事务不能混合
- 主从复制时，XA 事务的 binlog 在主库 COMMIT 后才写入
- 部分云数据库（如阿里云 RDS）对 XA 支持有限

# 7. 适用场景

- ✓ 需要**强一致性**的金融/支付场景
- ✓ 数据库原生支持 XA（MySQL 5.7+/Oracle/PostgreSQL）
- ✓ 不存在跨数据库类型的异构场景
- ✗ 高并发场景（长时间锁影响吞吐量）
- ✗ 期望最终一致、高性能的场景（用 AT 或 TCC）
- ✗ 涉及不支持 XA 的数据源
