# 1.事务概述

事务（Transaction）是一组操作的集合，它是一个**不可分割的工作单位**。事务会把所有操作作为一个整体一起向系统提交或撤销操作请求，即这些操作要么**全部成功**，要么**全部失败**。

**经典转账场景**：A 向 B 转账 100 元，包含两步操作：
1. A 账户扣 100 元
2. B 账户加 100 元

如果步骤 1 成功但步骤 2 失败，A 的钱被扣了但 B 没收到——这在金融系统中是不能接受的。事务机制确保两步要么都成功（COMMIT），要么都回滚（ROLLBACK）。

## 1.1为什么需要事务

- **保证数据一致性**：防止中间状态（部分成功、部分失败）被持久化
- **并发控制**：多个用户同时操作同一数据时，保证隔离性
- **故障恢复**：系统崩溃后能恢复到一致状态

# 2.事务操作

## 2.1自动提交模式

MySQL 默认是自动提交模式——每条 SQL 语句都是一个独立的事务，执行后自动提交：

```sql
-- 查看自动提交状态（1 = ON，0 = OFF）
SELECT @@autocommit;

-- 关闭自动提交（后续所有操作需手动 COMMIT 或 ROLLBACK）
SET @@autocommit = 0;

-- 开启自动提交（恢复默认）
SET @@autocommit = 1;
```

> **JDBC / Spring 事务**：当使用 Spring 的 `@Transactional` 时，框架会自动将 autocommit 设为 false，并在方法结束时根据是否抛出异常决定 COMMIT 或 ROLLBACK。这与手动管理事务是同一机制。

## 2.2显式事务控制

### 方式一：修改 autocommit 模式

```sql
SET @@autocommit = 0;   -- 关闭自动提交

UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 未提交
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 未提交

COMMIT;     -- 提交：所有修改生效
-- 或
ROLLBACK;   -- 回滚：所有修改撤销

SET @@autocommit = 1;   -- 恢复
```

### 方式二：显式开启事务（推荐）

```sql
START TRANSACTION;  -- 或 BEGIN;

UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;

COMMIT;   -- 或 ROLLBACK;
```

> 两种方式的区别：`START TRANSACTION` 可以在 autocommit=1 的模式下显式控制事务，而不需要改变全局/会话的 autocommit 设置。**推荐使用此方式**。

## 2.3保存点（SAVEPOINT）

保存点允许在事务中创建回滚标记，实现**部分回滚**：

```sql
START TRANSACTION;

UPDATE account SET balance = balance - 100 WHERE id = 1;
SAVEPOINT sp1;                                    -- 设置保存点

UPDATE account SET balance = balance + 100 WHERE id = 2;
-- 发现 id=2 的操作有问题
ROLLBACK TO sp1;                                   -- 回滚到 sp1，只保留第一步
-- 此时可以 COMMIT 第一步，或继续其他操作
COMMIT;
```

> 保存点常用于复杂的存储过程中：多个步骤中某个步骤失败时，回滚到上一个保存点而非整个事务。

## 2.4事务自动提交的触发条件

以下操作会**隐式提交**当前事务（即使你没有显式 COMMIT）：

- 执行 DDL 语句（CREATE、ALTER、DROP、TRUNCATE 等）
- 执行 DCL 语句（GRANT、REVOKE）
- 执行锁定语句（LOCK TABLES、UNLOCK TABLES）
- 开始一个新事务（START TRANSACTION 或 BEGIN）
- 从 MySQL 断开连接
- 执行 `SET AUTOCOMMIT = 1`（仅当之前设置为 0 时）

> **特别注意**：`TRUNCATE` 在 MySQL 8.0 之前会隐式提交，8.0 的原子 DDL 特性下 TRUNCATE 遵循事务语义。但为兼容性考虑，不建议在事务中混用 DDL 和 DML。

# 3.事务四大特性（ACID）

ACID 是衡量事务正确性的四个基本要素：

| 特性 | 全称 | 含义 | MySQL InnoDB 实现机制 |
|------|------|------|----------------------|
| **A** | Atomicity（原子性） | 事务是不可分割的最小操作单元，要么全部成功，要么全部失败 | undo log（回滚日志） |
| **C** | Consistency（一致性） | 事务完成时，必须使所有数据都保持一致状态 | redo log + undo log + 锁 |
| **I** | Isolation（隔离性） | 并发事务之间相互隔离，互不干扰 | MVCC + 锁（行锁、间隙锁） |
| **D** | Durability（持久性） | 事务一旦提交，对数据的改变就是永久的 | redo log（重做日志） |

**ACID 之间的关系**：
- 原子性是基础：不完整的操作不会留下痕迹
- 隔离性是手段：不同事务之间不互相干扰
- 持久性是保障：提交后的数据不丢失
- 一致性是最终目标：A + I + D 共同保证 C

# 4.并发事务带来的问题

当多个事务同时运行且操作同一数据时，可能引发以下问题：

| 问题 | 描述 | 示例 |
|------|------|------|
| **脏读（Dirty Read）** | 一个事务读到另一个事务**尚未提交**的数据 | 事务 B 修改了余额但未提交，事务 A 读到了修改后的值，然后 B 回滚了——A 读到的就是脏数据 |
| **不可重复读（Non-Repeatable Read）** | 一个事务内，两次读取**同一行**数据，结果不同（被其他事务 UPDATE 并提交了） | 事务 A 第一次读余额为 100，事务 B 此时更新为 200 并提交，事务 A 第二次读变成了 200 |
| **幻读（Phantom Read）** | 一个事务内，两次查询**同一条件**，结果集行数不同（其他事务 INSERT 了符合条件的行并提交） | 事务 A 查询 dept_id=1 的员工有 5 人，事务 B 此时插入了一个 dept_id=1 的员工并提交，事务 A 再次查询变成了 6 人 |

**不可重复读 vs 幻读**：
- 不可重复读：关注的是**同一行的内容被修改**（UPDATE）
- 幻读：关注的是**多了或少了几行**（INSERT / DELETE）

# 5.事务隔离级别

隔离级别由上到下依次增强，但性能也依次降低（并发能力下降）：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 加锁机制 |
|---------|------|-----------|------|---------|
| **READ UNCOMMITTED** | ✓ | ✓ | ✓ | 几乎不加锁 |
| **READ COMMITTED** | ✗ | ✓ | ✓ | 行锁（读到已提交数据） |
| **REPEATABLE READ（InnoDB默认）** | ✗ | ✗ | 部分解决 | 快照读（MVCC） + 临键锁 |
| **SERIALIZABLE** | ✗ | ✗ | ✗ | 表锁/范围锁，所有读加共享锁 |

> **为什么 InnoDB 的 REPEATABLE READ 能部分解决幻读？**
> InnoDB 在 RR 级别下使用**临键锁（Next-Key Lock）**，不仅锁定匹配的行，还锁定行之间的间隙（Gap Lock），防止其他事务插入符合条件的行。但对于 `SELECT ... FOR UPDATE` 的当前读，仍需小心幻读。

## 5.1查看和设置隔离级别

```sql
-- 查看当前会话隔离级别
SELECT @@TRANSACTION_ISOLATION;                 -- MySQL 8.0
SELECT @@tx_isolation;                          -- MySQL 5.7（已废弃）

-- 查看全局隔离级别
SELECT @@GLOBAL.TRANSACTION_ISOLATION;

-- 设置当前会话隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置全局隔离级别（重启后失效，永久生效需改 my.cnf）
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- my.cnf 配置
-- transaction_isolation = READ-COMMITTED
```

## 5.2各隔离级别实战演示

### READ-UNCOMMITTED（读未提交）

```sql
-- 会话 A                   -- 会话 B
SET SESSION tx_isolation='READ-UNCOMMITTED';
START TRANSACTION;          START TRANSACTION;
SELECT balance FROM account WHERE id=1;  -- 1000
                            UPDATE account SET balance=500 WHERE id=1;  -- 未提交
SELECT balance FROM account WHERE id=1;  -- 500（脏读！）
                            ROLLBACK;
SELECT balance FROM account WHERE id=1;  -- 1000（又变了）
COMMIT;
```

### READ-COMMITTED（读已提交）

```sql
-- 会话 A                   -- 会话 B
SET SESSION tx_isolation='READ-COMMITTED';
START TRANSACTION;
SELECT balance FROM account WHERE id=1;  -- 1000
                            UPDATE account SET balance=1500 WHERE id=1;
                            COMMIT;
SELECT balance FROM account WHERE id=1;  -- 1500（不可重复读！）
COMMIT;
```

### REPEATABLE-READ（可重复读）

```sql
-- 会话 A                   -- 会话 B
SET SESSION tx_isolation='REPEATABLE-READ';
START TRANSACTION;
SELECT balance FROM account WHERE id=1;  -- 1500
                            UPDATE account SET balance=2000 WHERE id=1;
                            COMMIT;
SELECT balance FROM account WHERE id=1;  -- 1500（重复读！不受 B 影响）
COMMIT;
SELECT balance FROM account WHERE id=1;  -- 2000（提交后可看到最新值）
```

## 5.3隔离级别的工程选择

| 隔离级别 | 适用场景 |
|---------|---------|
| **READ COMMITTED** | 互联网高并发业务（淘宝、京东等大多使用 RC，配合 binlog ROW 格式） |
| **REPEATABLE READ** | MySQL InnoDB 默认，适合对一致性要求较高的 OLTP 场景 |
| **SERIALIZABLE** | 极少使用，并发性能极差 |

> **为什么互联网公司倾向使用 RC 而非 RR？**
> 1. RC 不加 Gap Lock，锁冲突少，并发度高
> 2. RC 下 binlog 必须使用 ROW 格式（STATEMENT 格式在 RC 下不安全），ROW 格式本身也是互联网的主流选择
> 3. RR 的 Gap Lock 可能带来死锁风险，排查更困难
> 4. 互联网业务的最终一致性通常通过补偿机制（MQ、对账）保证，不依赖数据库的事务隔离

# 6.事务日志简介

事务的 ACID 特性依赖于 InnoDB 的两种日志：

| 日志 | 作用 | 存储位置 | 实现特性 |
|------|------|---------|---------|
| **redo log** | 记录数据页的物理修改（"做了什么修改"） | 磁盘（ib_logfile0, ib_logfile1） | 持久性（D） |
| **undo log** | 记录数据修改前的逻辑状态（"修改前是什么"） | undo 表空间 | 原子性（A） + MVCC |

- **redo log**：WAL（Write-Ahead Log）机制。修改先写 redo log，再异步刷脏页。崩溃恢复时通过 redo log 重放已提交但未刷盘的事务。
- **undo log**：记录旧版本数据。回滚时根据 undo log 恢复；MVCC 中通过 undo log 版本链支持快照读。

> 详细原理参考：[12_InnoDB_engine.md](12_InnoDB_engine.md) 和 [1_log.md](Operation&Maintenance/1_log.md)。

# 7.MVCC简介

**MVCC（Multi-Version Concurrency Control，多版本并发控制）**是 InnoDB 实现非锁定读（快照读）的核心机制，使得读操作不阻塞写操作，写操作不阻塞读操作。

**核心思想**：每一行数据维护多个版本，读操作读取一个"快照"（某个时间点的数据），而不是当前最新的数据，避免了读写锁冲突。

**实现要素**：

- **隐藏字段**：每行数据有 `DB_TRX_ID`（最后修改的事务 ID）、`DB_ROLL_PTR`（回滚指针，指向上一个版本）
- **undo log 版本链**：通过 `DB_ROLL_PTR` 串联起同一行的不同版本
- **Read View**：快照读时生成，记录当前活跃事务 ID 集合，用于判断哪些版本对当前事务可见

**读操作分类**：

| 读类型 | 说明 | 示例 |
|--------|------|------|
| **快照读（Snapshot Read）** | 读取数据的可见版本（基于 MVCC） | `SELECT * FROM user WHERE id = 1;` |
| **当前读（Current Read）** | 读取数据的最新版本并加锁 | `SELECT ... FOR UPDATE`、`UPDATE`、`DELETE` |

> 详细 MVCC 原理参考：[12_InnoDB_engine.md](12_InnoDB_engine.md)。

# 8.事务最佳实践

- **事务尽量短**：事务开启后尽快提交，避免长事务锁表、undo log 膨胀
- **避免事务中的用户交互**：不要在事务中等待用户输入、RPC 调用、发邮件等
- **合理设置隔离级别**：高并发场景考虑 RC，强一致性场景使用 RR
- **避免大事务**：大量数据更新分批进行，如逐批 UPDATE + COMMIT
- **注意事务嵌套**：`START TRANSACTION` 嵌套会隐式提交前一个事务，Spring 的事务传播行为有更精细的控制
- **监控长事务**：
  ```sql
  -- 查看当前活跃事务（运行时间）
  SELECT * FROM information_schema.INNODB_TRX
  WHERE trx_started < DATE_SUB(NOW(), INTERVAL 60 SECOND);
  ```
