# 1. 为什么需要 MVCC

数据库要同时服务很多事务。如果"读"必须等"写"提交、"写"必须等"读"结束，吞吐会迅速塌陷。**MVCC（Multi-Version Concurrency Control，多版本并发控制）** 的思路是：不覆盖旧数据，保留多个版本，让读操作看一个固定快照，写操作产生新版本，双方互不阻塞。

PG 的 MVCC 有一个鲜明特征：**多版本直接存放在堆表里**。一次 `UPDATE` 并不是原地修改，而是插入一条全新的行版本，再把旧版本标记为"已删除"。这带来两个直接后果：读操作永远不需要等待写操作；旧版本不会自动消失，必须由 **VACUUM** 回收，否则表会持续膨胀。

# 2. 行版本与系统列

## 2.1 每行都带的隐藏列

| 系统列 | 含义 |
|--------|------|
| `xmin` | 创建该行版本的事务 ID |
| `xmax` | 删除或锁定该行版本的事务 ID，0 表示未被删除/锁定 |
| `ctid` | 该版本的物理位置 `(块号, 行内偏移)` |

```sql
SELECT id, name, xmin, xmax, ctid FROM demo_user;
```

`ctid` 不是稳定的行标识：`UPDATE` 之后新版本通常在另一个位置，`ctid` 会变。**不要用 `ctid` 当主键**，它只在"同一时刻定位物理行"时有意义。

## 2.2 更新和删除的物理过程

```text
UPDATE：插入 A'（xmin=200）并把旧行 A 的 xmax 写成 200
        ┌──────────────┐        ┌──────────────┐
        │ A  xmin=100  │        │ A' xmin=200  │
        │    xmax=200  │        │    xmax=0    │
        └──────────────┘        └──────────────┘
DELETE：A' 的 xmax 写成 300，空间仍占用，等待 VACUUM
```

所以 **`UPDATE` = 插入新版本 + 标记旧版本删除**，写放大明显；**`DELETE` 不立即释放空间**，只是打标记；旧版本对"开始得比删除事务早"的快照仍然可见。

PG 有一个优化叫 **HOT（Heap-Only Tuple）更新**：如果更新没有改动任何被索引的列，且新版本能放进同一个数据页，就不需要更新索引项。HOT 能显著降低 UPDATE 的索引维护成本，因此**给频繁更新的列建索引要谨慎**。

## 2.3 死元组与表膨胀

被标记删除、且对所有活跃快照都不可见的版本，称为**死元组（dead tuple）**。它们占据页面空间，导致表文件只增不减、索引残留无效条目、缓存命中率下降。这就是**表膨胀（bloat）**：不报错，只让性能缓慢劣化。

# 3. 快照与可见性

## 3.1 事务 ID 与回卷

PG 的事务 ID 是 32 位整数，可用范围约 42 亿个，用完会绕回 0——**事务 ID 回卷（wraparound）**。若不处理，旧事务会被误认为未来事务，可见性彻底错乱。解决办法是 **freeze**：VACUUM 把足够旧的行版本标记为"对所有事务可见"，不再参与 ID 比较。

`autovacuum_freeze_max_age` 默认是 2 亿，到达后会自动触发 freeze。生产中不能只看默认值，建议在 10 亿左右告警、15 亿以上紧急处理；接近 20 亿时数据库会拒绝分配新事务 ID，必须提前干预。

```sql
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database ORDER BY xid_age DESC;   -- 10 亿告警，15 亿紧急处理，接近 20 亿拒绝写入
```

## 3.2 快照的结构

| 字段 | 含义 |
|------|------|
| `xmin` | 所有活跃事务中最小的事务 ID，比它小的都已结束 |
| `xmax` | 下一个尚未分配的事务 ID，比它大的都还没开始 |
| `xip` | 快照生成时仍在进行中的事务 ID 列表 |

```sql
SELECT pg_current_snapshot();      -- 例如 1000:1005:1002,1003
SELECT pg_snapshot_xmin(pg_current_snapshot()),
       pg_snapshot_xip(pg_current_snapshot());
```

## 3.3 可见性判断

```text
1. xmin 事务是否已提交？
   ├─ 否（进行中/已回滚） → 不可见
   └─ 是 → 继续
2. xmin 是否 >= 快照 xmax，或在 xip 列表中？
   ├─ 是 → 在快照之后才开始/尚未结束 → 不可见
   └─ 否 → 继续
3. xmax 是否为 0？
   ├─ 是 → 可见
   └─ 否 → xmax 已提交则不可见；未提交/已回滚则仍可见
```

为了加速判断，PG 会在元组头部写入 **hint bits**，把"事务已提交/已回滚"缓存在行上，避免每次都查 `pg_xact`。

# 4. 事务隔离级别

## 4.1 PG 的实现特点

| 隔离级别 | PG 的快照粒度 | 脏读 | 不可重复读 | 幻读 | 写偏斜 |
|---------|--------------|------|-----------|------|--------|
| READ COMMITTED（默认） | **每条语句**一个新快照 | 不会 | 可能 | 可能 | 可能 |
| REPEATABLE READ | **整个事务**一个快照 | 不会 | 不会 | **不会** | 可能 |
| SERIALIZABLE | 事务快照 + SSI 冲突检测 | 不会 | 不会 | 不会 | 不会 |

两个关键点：**PG 没有真正的读未提交**，`READ UNCOMMITTED` 会被静默提升为 RC；**PG 的 RR 已经能防幻读**，因为整个事务看同一个快照。但 RR 仍允许**写偏斜（write skew）**：两个事务各读一部分、各写另一部分，彼此看不到对方的写，最终破坏跨行约束。

```sql
SHOW transaction_isolation;
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

## 4.2 RC 与 RR 的差异

```sql
-- 会话 A                                     -- 会话 B
BEGIN;                                        BEGIN;
SELECT balance FROM account WHERE id = 1;     -- 1000
                                              UPDATE account SET balance = 1500 WHERE id = 1;
                                              COMMIT;
SELECT balance FROM account WHERE id = 1;
-- RC：返回 1500（每条语句新快照，不可重复读）
-- RR：返回 1000（事务级快照，可重复读）
COMMIT;
```

RC 下 `UPDATE` 遇到并发修改会**重新检查最新版本**（EvalPlanQual）；RR 则直接报 `could not serialize access due to concurrent update`，需要应用重试。

## 4.3 SERIALIZABLE 与 SSI

PG 的 SERIALIZABLE 基于 **SSI（Serializable Snapshot Isolation）**，不靠大范围加锁，而是记录事务间的读写依赖，检测"串行执行结果是否会不同"。发现危险依赖环就中止其中一个：

```text
ERROR: could not serialize access due to read/write dependencies among transactions
HINT: The transaction might succeed if retried.
```

**这不是 bug，而是 SSI 的正常工作方式**。应用必须捕获 SQLSTATE `40001` 和 `40P01` 并自动重试。没有重试逻辑却开了 SERIALIZABLE，等于把随机失败暴露给用户。

> **行业实践**：多数互联网业务用 READ COMMITTED 加应用层幂等/补偿；金融账务等强一致场景才用 SERIALIZABLE，并配套重试框架。

# 5. VACUUM

## 5.1 VACUUM 做什么

1. 扫描表，把死元组占用的空间标记为**可复用**；
2. 更新**可见性映射（visibility map）**，让 Index Only Scan 能跳过堆表；
3. **冻结**足够旧的行版本，防止事务 ID 回卷。

普通 `VACUUM` **不归还磁盘空间给操作系统**，只让空间可被后续插入复用。

## 5.2 autovacuum 的触发条件

```text
触发 VACUUM：dead_tuples > autovacuum_vacuum_threshold
            + autovacuum_vacuum_scale_factor × reltuples（默认 50 + 0.2 × 行数）
```

一张 100 万行的表，死元组超过约 20 万才会触发。对大表来说太宽松，更新频繁的大表应单独调小：

```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.01
);
```

```sql
-- 查看 autovacuum 活动
SELECT pid, relid::regclass, phase, heap_blks_total, heap_blks_scanned
FROM pg_stat_progress_vacuum;

-- 死元组与最近清理时间
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;
```

## 5.3 膨胀检测

```sql
SELECT relname, n_live_tup, n_dead_tup,
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

需要精确值时安装 `pgstattuple`，执行 `SELECT * FROM pgstattuple('orders');` 查看实际死元组比例与空闲空间。

## 5.4 VACUUM、VACUUM FULL 与 pg_repack

| 方式 | 是否阻塞读写 | 是否归还磁盘 | 适用 |
|------|-------------|-------------|------|
| `VACUUM` | 不阻塞 | 否，只标记可复用 | 日常维护 |
| `VACUUM FULL` | **阻塞**（ACCESS EXCLUSIVE） | 是，重写整表 | 严重膨胀且可停机 |
| `pg_repack` | 几乎不阻塞 | 是 | 在线重建大表 |
| `CLUSTER` | 阻塞 | 是，按索引重排 | 需要物理顺序与索引一致 |

> **警告**：生产环境不要随意 `VACUUM FULL` 大表。它会持有排他锁直到重写完成，期间所有查询和写入都会阻塞。优先用 `pg_repack`，或安排在维护窗口。

## 5.5 长事务为什么危险

长事务（尤其 `idle in transaction`）会持有旧快照，导致比它更旧的死元组**不能被清理**，`xmin` 被钉住、freeze 无法推进，并长期持锁阻塞 DDL。

```sql
-- 超过 5 分钟的事务
SELECT pid, now() - xact_start AS duration, state, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL AND now() - xact_start > interval '5 minutes'
ORDER BY duration DESC;
```

建议在实例级设置 `idle_in_transaction_session_timeout = 60s`、`statement_timeout = 30s`、`lock_timeout = 5s`，从源头限制长事务与无限等锁。

# 6. 锁

## 6.1 表级锁

| 锁模式 | 典型语句 | 冲突对象 |
|--------|---------|---------|
| ACCESS SHARE | `SELECT` | 仅 ACCESS EXCLUSIVE |
| ROW SHARE | `SELECT FOR UPDATE` | EXCLUSIVE、ACCESS EXCLUSIVE |
| ROW EXCLUSIVE | `INSERT/UPDATE/DELETE` | SHARE 及更强 |
| SHARE UPDATE EXCLUSIVE | `VACUUM`、并发建索引 | 自身及更强 |
| EXCLUSIVE | 阻塞所有写，允许普通读 | 除 ACCESS SHARE 外 |
| ACCESS EXCLUSIVE | `DROP`、`TRUNCATE`、`VACUUM FULL`、多数 `ALTER` | 全部 |

实用规则：**普通 DML 只拿 ROW EXCLUSIVE，普通 SELECT 只拿 ACCESS SHARE，两者不冲突**；真正容易卡住业务的是 `ALTER TABLE` 的 ACCESS EXCLUSIVE。

```sql
-- DDL 前设置等锁超时，拿不到锁就失败，而不是排队堵住后面所有查询
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN remark text;
```

## 6.2 行级锁

```sql
SELECT * FROM account WHERE id = 1 FOR UPDATE;         -- 排他，阻止更新/删除
SELECT * FROM account WHERE id = 1 FOR NO KEY UPDATE;  -- 较弱，不阻止外键检查
SELECT * FROM account WHERE id = 1 FOR SHARE;          -- 共享，阻止写入

-- 任务队列：跳过别人正在处理的行
SELECT * FROM job WHERE status = 'pending'
ORDER BY id LIMIT 10 FOR UPDATE SKIP LOCKED;
```

`FOR UPDATE SKIP LOCKED` 是多消费者抢任务的经典写法：每个 worker 跳过被锁定的行，不会互相等待。

## 6.3 锁等待排查

```sql
SELECT pid, wait_event_type, wait_event,
       pg_blocking_pids(pid) AS blocked_by, state, query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

需要看具体锁对象时查询 `pg_locks`，过滤 `granted = false`，再按 `pid` 关联 `pg_stat_activity`。

## 6.4 死锁

两个事务各自持有对方需要的锁就形成死锁。`deadlock_timeout` 默认 1 秒：等锁超过后做死锁检测，发现环就中止其中一个，报 `deadlock detected`。

避免手段：**固定加锁顺序**、缩短事务、补齐索引（避免 `UPDATE` 锁定远超预期的行）、设置 `lock_timeout`、捕获 `40P01` 并重试。

# 7. 常见错误与排错

| 现象 | 根因 | 处理 |
|------|------|------|
| 表越来越大、查询变慢 | 死元组堆积、autovacuum 跟不上 | 查 `n_dead_tup`，调参或手动 VACUUM |
| `database is not accepting commands to avoid wraparound` | 事务 ID 年龄逼近上限 | 立即 `VACUUM FREEZE`，排查长事务 |
| `canceling statement due to lock timeout` | 等锁超过 `lock_timeout` | 查 `pg_blocking_pids`，优化 DDL 时机 |
| `could not serialize access due to concurrent update` | RR 下并发改同一行 | 重试事务，或评估降为 RC |
| 备库报 `conflict with recovery` | 主库 VACUUM 清理了备库还在读的版本 | 调 `max_standby_streaming_delay`，或把长查询移走 |

```sql
SELECT pid, state, xact_start, wait_event_type, left(query, 80) FROM pg_stat_activity
WHERE xact_start IS NOT NULL ORDER BY xact_start;
SELECT datname, deadlocks, conflicts, temp_files FROM pg_stat_database;
```

# 8. 练习与自检

**练习一**：开两个 psql 会话，分别在 RC 与 RR 下复现"不可重复读"，记录两次结果并解释差异。

**练习二**：制造一次死锁（两个事务反向更新两行），从日志和 `pg_stat_database.deadlocks` 确认它，再改写成固定顺序验证不再死锁。

**练习三**：创建 10 万行的表，循环更新全部行三次，观察 `n_dead_tup`、表文件大小与 VACUUM 前后的变化。解释为什么 VACUUM 后文件大小没变。

**自检清单**：

- 能画出一次 UPDATE 产生的两个行版本，并标出 `xmin/xmax`；
- 能解释 RC 与 RR 的快照粒度差异，知道 PG 的 RR 已能防幻读；
- 能说清 VACUUM 的三个作用，以及 `VACUUM FULL` 的代价；
- 能用 `pg_blocking_pids` 定位锁等待，用 `pg_stat_activity` 找长事务；
- 知道 `FOR UPDATE SKIP LOCKED` 的适用场景。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| MVCC | 堆表保留多版本，读不阻塞写，旧版本靠 VACUUM 回收 |
| 系统列 | `xmin/xmax` 决定可见性，`ctid` 是物理位置且会变 |
| 快照 | RC 每语句一个快照，RR 每事务一个，Serializable 用 SSI |
| 回卷 | 32 位事务 ID 会绕回，必须靠 freeze 防护 |
| VACUUM | 标记空间可复用、更新可见性映射、冻结旧事务；不缩文件 |
| 长事务 | 钉住 xmin、阻碍清理、持锁不放，是 PG 运维头号敌人 |
| 锁 | DML 之间基本不冲突，DDL 的 ACCESS EXCLUSIVE 才是阻塞源 |

事务与 MVCC 决定了"数据怎么写、怎么读、怎么回收"。接下来要回答另一个问题：**怎么让查询变快**。下一篇进入索引与优化，从 B-tree、GIN、GiST、BRIN 的选型讲到 `EXPLAIN` 的读法，再到连接池为什么在 PG 里几乎是必需品。