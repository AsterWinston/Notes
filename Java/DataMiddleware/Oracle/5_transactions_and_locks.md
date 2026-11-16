# 1. 事务基础

事务是一组要么全部成功、要么全部失败的数据库操作。Oracle 的事务与 MySQL 最大的不同在于：**事务由第一条 DML 隐式开始，而不是 `BEGIN`**。在 Oracle 里执行 `INSERT` 时事务已经打开，`BEGIN` 只是 PL/SQL 块的开始，不是事务开始。

```sql
-- Oracle 没有 "START TRANSACTION"
INSERT INTO orders (id, amount) VALUES (1, 100);  -- 事务从这里开始
UPDATE account SET balance = balance - 100 WHERE id = 1;

COMMIT;    -- 提交，变更持久化
-- 或
ROLLBACK;  -- 回滚，撤销全部未提交变更
```

## 1.1 ACID 在 Oracle 中的落点

| 特性 | Oracle 的实现 |
|------|--------------|
| 原子性（Atomicity） | undo 段记录前镜像，回滚时反向应用 |
| 一致性（Consistency） | 约束、触发器、应用逻辑共同保证 |
| 隔离性（Isolation） | undo + SCN 实现读一致性，锁实现写隔离 |
| 持久性（Durability） | LGWR 保证 redo 落盘，提交即持久 |

关键点是**读一致性（Read Consistency）**：Oracle 用 undo 段构造出"语句开始时刻"的数据快照，所以查询不会被未提交的写阻塞，也看不到未提交的数据。查询过程中即使别人改了数据，当前语句看到的仍是同一个 SCN 下的版本。

## 1.2 SCN 与提交

**SCN（System Change Number）** 是 Oracle 内部单调递增的逻辑时钟。每个事务提交时会分配一个 SCN，数据块头部记录着"最后一次修改它的 SCN"。查询时，Oracle 只展示 SCN 小于等于当前快照 SCN 的变更，这就是读一致性的底层机制。

```sql
-- 查看当前 SCN
SELECT current_scn FROM v$database;

-- 查看会话的事务信息
SELECT xid, status, start_scn, commit_scn
FROM v$transaction;
```

> **注意**：`COMMIT` 只保证 redo 落盘，不保证数据块已写回数据文件。数据块由 DBWn 延迟写。这正是 Oracle 提交快的原因，也是崩溃后需要实例恢复的原因。

## 1.3 保存点

```sql
SAVEPOINT sp1;
UPDATE account SET balance = balance - 100 WHERE id = 1;

SAVEPOINT sp2;
UPDATE account SET balance = balance + 100 WHERE id = 2;

ROLLBACK TO sp2;   -- 只撤销 sp2 之后的操作
COMMIT;            -- 提交 sp2 之前的操作
```

保存点适合"部分回滚"的场景，比如批量处理中某一条失败但不想丢掉前面已处理的成果。但要注意，保存点不能跨 `COMMIT` 使用，提交后所有保存点失效。

# 2. 隔离级别

SQL 标准定义了四个隔离级别，Oracle 只实现了其中两个层级，但通过 undo 机制达到了接近标准的效果。

| 隔离级别 | Oracle 支持 | 脏读 | 不可重复读 | 幻读 |
|---------|------------|------|-----------|------|
| READ UNCOMMITTED | 不支持 | — | — | — |
| **READ COMMITTED**（默认） | 支持 | 不会 | **可能** | **可能** |
| REPEATABLE READ | 不单独提供 | 不会 | 不会 | 不会 |
| **SERIALIZABLE** | 支持 | 不会 | 不会 | 不会 |

## 2.1 READ COMMITTED

默认级别，**每条语句**开始时生成一个新快照。同一事务内两次查询之间，如果别人提交了修改，第二次能看到新值，这就是**不可重复读**。

```sql
-- 会话 A
SELECT amount FROM orders WHERE id = 1;   -- 100

-- 会话 B（此时提交了修改）
UPDATE orders SET amount = 200 WHERE id = 1;
COMMIT;

-- 会话 A 再查
SELECT amount FROM orders WHERE id = 1;   -- 200（READ COMMITTED 下会变）
```

## 2.2 SERIALIZABLE

`SET TRANSACTION ISOLATION LEVEL SERIALIZABLE` 让**整个事务**共用一个快照，事务内多次查询结果一致，相当于"可重复读 + 防幻读"。但 Oracle 的实现方式不是加锁，而是**快照隔离 + 冲突检测**：如果当前事务要修改的行在快照之后被别的事务改过，就报 `ORA-08177: can't serialize access for this transaction`。

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM account WHERE id = 1;
-- 期间别人改了这行并提交
UPDATE account SET balance = balance - 100 WHERE id = 1;
-- ORA-08177，需要回滚重试
```

> **行业实践**：绝大多数 OLTP 业务用默认的 READ COMMITTED，靠应用层的行锁（`SELECT FOR UPDATE`）或乐观锁（版本号）解决并发。SERIALIZABLE 用在高一致性要求的账务核对场景，且**必须配套重试逻辑**，否则把随机失败暴露给用户。

## 2.3 快照过旧：ORA-01555

长事务在 SERIALIZABLE 或长查询在 READ COMMITTED 下，都需要读取旧版本数据。如果 undo 表空间不够、旧版本已被覆盖，就会报 `ORA-01555: snapshot too old`。

典型触发是"大批量游标循环 + 期间大量 DML"：事务拖得越久，需要的旧版本越多。处理思路是缩短事务、增大 undo 表空间与 `UNDO_RETENTION`、把大查询拆成按主键范围的分批查询。

# 3. 锁机制

Oracle 的锁是**自动管理**的，不需要像 MySQL 那样显式 `LOCK TABLES`。它的设计哲学是"**尽量少锁、锁尽量短**"，并且**不做锁升级**——一万行行锁还是行锁，不会升级成表锁。

## 3.1 锁的粒度

| 级别 | 说明 | 典型场景 |
|------|------|---------|
| 行级锁（TX） | 锁住被 DML 修改的行 | `UPDATE`、`DELETE`、`SELECT FOR UPDATE` |
| 表级锁（TM） | 表示事务在该表上有活动，分多种模式 | DML 时自动加，防止 DDL 冲突 |
| 字典锁 | 保护对象定义 | DDL 期间 |

日常最需要理解的是 **TX 锁**：执行 `UPDATE` 时，Oracle 在该行上加一把排他锁，直到事务提交或回滚才释放。**行锁不阻塞普通查询**，只阻塞其他事务对同一行的写。

```sql
-- 会话 A
UPDATE account SET balance = 900 WHERE id = 1;   -- 持有 id=1 的行锁

-- 会话 B：会一直等待，直到 A 提交或回滚
UPDATE account SET balance = 1000 WHERE id = 1;

-- 会话 B 可以设置不等待
UPDATE account SET balance = 1000 WHERE id = 1 NOWAIT;
-- 或等待 5 秒后放弃
UPDATE account SET balance = 1000 WHERE id = 1 WAIT 5;
```

## 3.2 SELECT FOR UPDATE

普通 `SELECT` 不加锁，靠 undo 读快照。如果"先查再改"需要防止别人插队，就要用 `SELECT ... FOR UPDATE` 锁住目标行。

```sql
-- 悲观锁：查到即锁定，其他事务改不了
SELECT * FROM account WHERE id = 1 FOR UPDATE;

-- 不等待，锁不到立刻报 ORA-00054
SELECT * FROM account WHERE id = 1 FOR UPDATE NOWAIT;

-- 等待 3 秒
SELECT * FROM account WHERE id = 1 FOR UPDATE WAIT 3;

-- 跳过被锁定的行，适合多消费者抢任务
SELECT * FROM job WHERE status = 'PENDING'
FOR UPDATE SKIP LOCKED;
```

| 写法 | 行为 | 适用 |
|------|------|------|
| `FOR UPDATE` | 无限等待 | 默认悲观锁 |
| `FOR UPDATE NOWAIT` | 锁不到立即失败 | 快速失败，避免请求堆积 |
| `FOR UPDATE WAIT n` | 等 n 秒后失败 | 有超时预期的场景 |
| `FOR UPDATE SKIP LOCKED` | 跳过被锁行 | 任务队列、批量取数 |

> **行业实践**：`SKIP LOCKED` 是 Oracle 做"任务队列"的利器。多个 worker 同时 `SELECT ... FOR UPDATE SKIP LOCKED`，每个拿到不同的行，不会互相等待。这比用 `UPDATE ... WHERE rownum <= n` 抢任务更可靠。

## 3.3 锁的查看

```sql
-- 当前被锁对象与会话
SELECT s.sid, s.username, o.object_name, l.locked_mode
FROM v$locked_object l
JOIN dba_objects o ON l.object_id = o.object_id
JOIN v$session s ON l.session_id = s.sid;
-- 谁阻塞了谁
SELECT sid, blocking_session, event, seconds_in_wait
FROM v$session WHERE blocking_session IS NOT NULL;
```

## 3.4 锁等待与阻塞

Oracle 没有 `lock_timeout` 这种会话级参数，但可以用 `FOR UPDATE WAIT n` 或在应用层设置语句超时。排查阻塞的标准路径是：用 `V$SESSION.BLOCKING_SESSION` 找到阻塞者，再看它执行的 SQL 和持锁时长；`V$LOCK` 里 `block = 1` 或 `request > 0` 的行就是锁等待明细。

```sql
SELECT s.sid, s.sql_id, s.event FROM v$session s
WHERE s.sid = (SELECT blocking_session FROM v$session WHERE blocking_session IS NOT NULL);
```

# 4. 死锁

死锁是两个或多个事务互相持有对方需要的锁，形成环。Oracle 会自动检测，检测到后**回滚其中一个事务**并报 `ORA-00060`，另一个继续执行。

```text
事务 A：锁住行 1 ──等待──▶ 行 2
                              ▲
                              │
事务 B：锁住行 2 ──等待──▶ 行 1
（A 等 B 释放行 2，B 等 A 释放行 1，形成环）
```

复现方法很简单：两个会话先各更新一行，再反向更新对方那一行，后执行的一方就会触发死锁检测。Oracle 在告警日志和跟踪文件里写入两个会话的 SQL、持有的锁与等待的锁。

## 4.1 死锁的排查

```sql
-- 告警日志位置；再在 trace 目录搜索 ORA-00060
SELECT value FROM v$diag_info WHERE name = 'Diag Trace';
```

## 4.2 避免死锁的手段

- **固定加锁顺序**：所有事务按相同顺序访问资源，比如先按 `id` 升序更新账户。
- **缩短事务**：尽快提交，不要把用户交互、远程调用放在事务里。
- **补齐索引**：`UPDATE` 没有合适索引会锁住扫描到的所有行，放大冲突面。
- **使用 `NOWAIT`/`WAIT n`**：快速失败，把重试交给应用层。
- **捕获 `ORA-00060` 并重试**：应用必须能处理死锁，而不是把错误直接抛给用户。

> **注意**：死锁不是"数据库坏了"，而是并发访问顺序不一致的必然结果。Oracle 主动回滚一个事务是**保护机制**，正确的应对是重试，而不是去调大什么参数。

# 5. AUTONOMOUS_TRANSACTION

**自治事务（Autonomous Transaction）** 用 `PRAGMA AUTONOMOUS_TRANSACTION` 声明，它开启一个**独立于调用方的事务**：内部 `COMMIT`/`ROLLBACK` 不影响主事务，主事务回滚也不影响它。

```sql
CREATE OR REPLACE PROCEDURE sp_write_log(p_msg VARCHAR2) AS
    PRAGMA AUTONOMOUS_TRANSACTION;   -- 声明自治事务
BEGIN
    INSERT INTO audit_log (msg, created_at) VALUES (p_msg, SYSDATE);
    COMMIT;                          -- 只提交日志，不提交主事务
END;
/
```

## 5.1 典型用途

| 场景 | 说明 |
|------|------|
| 错误日志 | 主事务回滚时，日志仍要保留 |
| 审计记录 | 记录"谁尝试了什么"，无论成败 |
| 独立计数器 | 不影响主业务事务的序列式计数 |
| 动态 DDL | 某些运维过程需要在独立事务中执行 |

## 5.2 使用规则与坑

自治事务有几条硬性规则：内部**必须显式 `COMMIT` 或 `ROLLBACK`**，否则结束时抛 `ORA-06519`；它**看不到主事务未提交的修改**；它提交后主事务再回滚，其变更**不会回滚**；它还会额外占用 undo 和锁资源，滥用会导致事务数超限或日志膨胀。

> **警告**：自治事务不是"随便提交"的许可证。它破坏了事务的原子性边界，用错会导致"主业务回滚了，但日志/计数/关联数据留下了"的不一致。只在**明确需要脱离主事务**的场景使用，并在代码注释里写清原因。

# 6. 常见误用

## 6.1 在循环里提交

典型错误是在 `FOR` 循环里逐行 `INSERT` 并 `COMMIT`。这会带来大量日志同步等待，更严重的是**破坏原子性**：如果中途失败，前面已提交的数据无法回滚，业务处于半完成状态。正确做法是用 `FORALL` 批量插入后统一提交，或按批次提交并记录批次边界。

## 6.2 长事务与 undo 膨胀

一个事务跑了半小时，undo 表空间会持续增长，其他会话可能遇到 `ORA-01555`，DDL 也会被阻塞。生产环境应限制事务时长，批量任务按批次提交。

## 6.3 忘记提交

Oracle 不像 MySQL 默认自动提交（`autocommit=true`）。JDBC 默认是手动提交，如果代码里只执行 `INSERT` 却没 `commit()`，连接关闭时事务回滚，数据就"凭空消失"。

```java
// 错误示范：执行了 INSERT 却没有 conn.commit()，连接归还后事务回滚
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement("INSERT INTO t VALUES (?)")) {
    ps.setInt(1, 1);
    ps.executeUpdate();
}
// 正确做法：显式 conn.commit()，或使用 Spring 的 @Transactional
```

## 6.4 用 `SELECT` 做"检查再插入"

两个事务同时"先查不存在再插入"，会双双插入成功，产生重复数据。要么加唯一约束，要么用 `MERGE`，要么 `SELECT ... FOR UPDATE` 锁住范围。

```sql
-- 推荐：用 MERGE 做幂等写入
MERGE INTO account a
USING (SELECT 1 AS id FROM dual) s
ON (a.id = s.id)
WHEN MATCHED THEN UPDATE SET a.balance = a.balance + 100
WHEN NOT MATCHED THEN INSERT (id, balance) VALUES (1, 100);
```

## 6.5 误以为行锁会升级

Oracle **不会**把大量行锁升级为表锁，这减少了意外阻塞，但也意味着一个更新百万行的事务会持有百万把行锁，undo 和锁资源开销都很大。大批量更新应拆批并控制事务大小。

# 7. 练习

- **练习一**：开两个 `sqlplus` 会话，在 READ COMMITTED 下复现"不可重复读"，再改为 SERIALIZABLE 复现 `ORA-08177`，记录两次报错并解释差异。
- **练习二**：制造一次死锁（两个事务反向更新两行），从告警日志中找到 `ORA-00060` 的详细信息，再改写成固定顺序验证不再死锁。
- **练习三**：写一个带 `PRAGMA AUTONOMOUS_TRANSACTION` 的日志过程，在主事务回滚后验证日志仍存在；再去掉内部 `COMMIT`，观察 `ORA-06519`。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 事务开始 | Oracle 由第一条 DML 隐式开始，没有 `START TRANSACTION` |
| 读一致性 | undo + SCN 构造快照，读不阻塞写，写不阻塞读 |
| 隔离级别 | 默认 READ COMMITTED；SERIALIZABLE 靠快照 + 冲突检测，需重试 |
| 锁 | 行锁（TX）+ 表锁（TM），不做锁升级，普通查询不加锁 |
| `FOR UPDATE` | 悲观锁，可用 `NOWAIT`/`WAIT n`/`SKIP LOCKED` 控制等待 |
| 死锁 | 自动检测并回滚一方，靠固定顺序和短事务预防，应用要重试 |
| 自治事务 | 独立提交，看不到主事务未提交数据，必须显式 COMMIT/ROLLBACK |
| 误用 | 循环提交、忘记提交、检查再插入、长事务是四大高频坑 |

事务与锁保证了并发下的正确性，但"正确"不等于"快"。下一篇进入**索引与执行计划**，讲清 B-tree、位图、函数索引的选型，怎么读 `EXPLAIN PLAN` 和 `DBMS_XPLAN`，统计信息为什么会让同一条 SQL 时快时慢，以及 hint 该不该用。
