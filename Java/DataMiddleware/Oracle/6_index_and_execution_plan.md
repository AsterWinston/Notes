# 1. 索引基础

索引是用**额外的存储和写入开销**换取查询速度的数据结构。Oracle 的索引以 B-tree 为绝对主力，另外提供位图、函数、反向键、域索引等变体。理解索引的第一步，不是背类型，而是理解**代价**：每次 `INSERT`、`UPDATE`、`DELETE` 都要维护索引，索引越多，写越慢；索引选错，优化器可能放着全表扫描不用，反而绕远路。

Oracle 的索引是**独立于表的段（Segment）**，有自己独立的表空间和存储参数。这与 MySQL InnoDB"二级索引存主键值"、PostgreSQL"索引指向 ctid"都不同：Oracle 的 B-tree 叶子节点存的是**索引列值 + ROWID**，通过 ROWID 再回表取整行。

```text
       根块 Root
      ┌────┴────┐
   分支块      分支块
   ┌─┴─┐      ┌─┴─┐
  叶子  叶子   叶子  叶子   ← 每个叶子存 key + ROWID
```

> **设计考量**：ROWID 是行的物理地址，Oracle 通过它直接定位数据块和行槽位，回表是一次精确的随机读。这也意味着**行迁移（row migration）会让 ROWID 变化**，导致索引需要重建，这是 Oracle 特有的运维点。

# 2. B-tree 索引

B-tree 是默认索引类型，适合**高基数（distinct 值多）**的列，比如主键、订单号、用户 id、时间戳。

## 2.1 创建与基本用法

```sql
-- 普通 B-tree 索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- 唯一索引
CREATE UNIQUE INDEX uk_orders_no ON orders(order_no);
-- 复合索引：注意列顺序
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

索引信息可在 `USER_INDEXES` 查看索引类型与状态，列顺序在 `USER_IND_COLUMNS` 中按 `COLUMN_POSITION` 排列。

## 2.2 复合索引与最左前缀

复合索引 `(user_id, status)` 能高效支持 `WHERE user_id = ?` 和 `WHERE user_id = ? AND status = ?`，但**不能**高效支持只有 `WHERE status = ?` 的查询，因为 `status` 不是最左列。这被称为**最左前缀原则**，与 MySQL 一致。

> **行业实践**：复合索引的列顺序按"等值条件在前、范围条件在后、区分度高的在前"来排。比如 `WHERE user_id = ? AND created_at > ?`，应建 `(user_id, created_at)`；如果反过来建 `(created_at, user_id)`，范围条件会截断后续列的使用。

## 2.3 索引失效的常见原因

| 原因 | 例子 | 处理 |
|------|------|------|
| 列上做函数运算 | `WHERE UPPER(name) = 'X'` | 建函数索引，或改写条件 |
| 隐式类型转换 | `WHERE id = '100'`（id 是 NUMBER） | 参数类型与列一致 |
| 前导通配符 | `WHERE name LIKE '%abc'` | 用全文索引或反向索引 |
| 对 NULL 判断 | `WHERE col IS NULL` | 单列索引不存全 NULL 键，复合索引可包含 |
| 联合索引非最左列 | 见上 | 调整索引或补索引 |
| 统计信息过期 | 优化器误判 | 收集统计信息 |

> **注意**：Oracle 的 B-tree 索引**默认不存储全为 NULL 的键**。所以 `WHERE col IS NULL` 用不上普通单列索引。如果这类查询很频繁，可以建 `(col, 常量)` 的复合索引，让 NULL 也能被索引覆盖。

# 3. 位图索引

位图索引（Bitmap Index）为每个 distinct 值维护一个**位图**，每一位对应一行。比如 `status` 有 NEW/PAID/DONE 三个值，就存三串位图，多条件查询时按位与/或，效率极高。它适合**低基数**列，比如性别、状态、地区、标志位。

```sql
CREATE BITMAP INDEX idx_orders_status_bm ON orders(status);
```

| 维度 | B-tree | 位图 |
|------|--------|------|
| 适用基数 | 高（> 1000 个不同值） | 低（< 100 个不同值） |
| 并发 DML | 行级，冲突小 | **容易锁冲突**，不适合高并发写 |
| 存储 | 相对大 | 压缩后小 |
| 典型场景 | OLTP | 数据仓库、报表、星型模型 |

> **警告**：位图索引在 OLTP 高并发写入下是灾难。一个 `UPDATE` 可能锁住整个位图段，导致大量会话互相阻塞。**只读或批量加载的报表库**才用位图索引，业务库的状态字段不要建位图。

# 4. 函数索引

当查询条件里对列做了函数运算，普通索引失效。函数索引（Function-Based Index）把**函数计算结果**作为索引键存起来。

```sql
-- 查询：忽略大小写匹配
SELECT * FROM users WHERE UPPER(email) = 'A@B.COM';

-- 普通索引失效，建函数索引
CREATE INDEX idx_users_upper_email ON users(UPPER(email));

-- 现在能走索引
SELECT * FROM users WHERE UPPER(email) = 'A@B.COM';
```

函数索引的使用有前提：

- 查询里的表达式必须与索引定义**完全一致**（包括大小写、空格）。
- 函数必须是**确定性（deterministic）**的，`SYSDATE`、`DBMS_RANDOM` 这类不能用。
- 需要 `QUERY REWRITE` 权限（某些场景）。

> **设计考量**：函数索引能救急，但更推荐**从源头避免在列上做函数**。比如把 `created_at` 的范围查询改写成 `>= 当天零点 AND < 次日零点`，就能用普通索引，且不依赖函数索引的维护成本。

# 5. 其他索引类型

| 类型 | 用途 | 说明 |
|------|------|------|
| 反向键索引（Reverse Key） | 缓解 RAC 或序列主键的叶子块争用 | `CREATE INDEX ... REVERSE`，范围查询失效 |
| 降序索引（Descending） | 复合索引中某列降序 | `CREATE INDEX ... (a, b DESC)`，11g 起真正生效 |
| 不可见索引（Invisible） | 灰度验证索引是否被使用 | `ALTER INDEX ... INVISIBLE`，优化器默认忽略 |
| 本地/全局分区索引 | 分区表上的索引 | 本地索引与分区一一对应，易维护 |
| 域索引（Domain） | 全文、空间、JSON 等 | 由插件或选件提供 |

不可见索引可以用 `CREATE INDEX ... INVISIBLE` 创建、用 `ALTER INDEX ... VISIBLE` 恢复；反向键索引用 `CREATE INDEX ... REVERSE`，把键值反向存储以打散叶子块争用，但范围查询会失效。

# 6. 执行计划

执行计划是优化器给出的"怎么执行这条 SQL"的说明书。读懂它，是 SQL 优化的核心能力。

## 6.1 生成执行计划

```sql
-- 估算计划，不真正执行
EXPLAIN PLAN FOR SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- 实际执行过的计划，最准确
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id', NULL, 'ALLSTATS LAST'));
-- SQL*Plus 的 AUTOTRACE
SET AUTOTRACE ON EXPLAIN STATISTICS
```

## 6.2 常见访问路径

| 操作 | 含义 | 何时出现 |
|------|------|---------|
| `TABLE ACCESS FULL` | 全表扫描 | 无可用索引、或优化器认为全表更快 |
| `TABLE ACCESS BY INDEX ROWID` | 通过 ROWID 回表 | 索引命中后取整行 |
| `INDEX UNIQUE SCAN` | 唯一索引等值查找 | 主键/唯一键等值查询 |
| `INDEX RANGE SCAN` | 索引范围扫描 | 范围条件、非唯一等值 |
| `INDEX FULL SCAN` | 索引全扫描（有序） | 需要按索引顺序读全部 |
| `INDEX FAST FULL SCAN` | 索引快速全扫描（无序，多块读） | 只查索引列，可并行 |
| `INDEX SKIP SCAN` | 跳过索引前导列 | 复合索引非最左列，统计信息有前导列 distinct 值少 |
| `NESTED LOOPS` | 嵌套循环连接 | 小结果集驱动大表 |
| `HASH JOIN` | 哈希连接 | 大表与大表等值连接 |
| `MERGE JOIN` | 排序合并连接 | 非等值连接或已排序数据 |

## 6.3 读一个执行计划

```text
| Id | Operation                    | Name            | Rows | Cost |
|  0 | SELECT STATEMENT             |                 |   10 |    4 |
|  1 |  TABLE ACCESS BY INDEX ROWID | ORDERS          |   10 |    4 |
|* 2 |   INDEX RANGE SCAN           | IDX_ORDERS_USER |   10 |    2 |
```

读计划的关键是**从内到外、从上到下**看缩进：`Id=2` 的索引范围扫描先执行，得到 ROWID；`Id=1` 用 ROWID 回表；`Id=0` 返回结果。

重点看三列：

- **Rows**：优化器估算的行数。如果与实际相差几个数量级，说明统计信息有问题。
- **Cost**：估算代价，相对值，不是毫秒。
- **`*` 标记**：表示该行有谓词条件（Predicate Information），要看是否发生了隐式转换。

```sql
-- 查看谓词信息，确认条件是否按预期下推
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST +PREDICATE'));
```

## 6.4 实际行数与估算行数

`DBMS_XPLAN` 的 `ALLSTATS LAST` 会显示 `A-Rows`（实际行数）与 `E-Rows`（估算行数）。两者差距大，通常是统计信息过期或列之间有关联（比如 `city` 和 `province` 强相关，优化器按独立假设估算，严重偏离）。

> **行业实践**：优化 SQL 时，先看 `A-Rows` 与 `E-Rows` 是否吻合，再看访问路径是否合理，最后才考虑加 hint。顺序反了，就会陷入"凭感觉加 hint"的泥潭。

# 7. 统计信息

优化器是**基于代价（CBO）**的，代价估算依赖统计信息。统计信息过期，执行计划就可能突然变差，这是 Oracle 生产环境"同一条 SQL 昨天还快、今天就慢"的最常见原因。

## 7.1 收集统计信息

```sql
-- 收集单表统计信息，同时收集索引统计
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname    => 'APP_USER',
        tabname    => 'ORDERS',
        cascade    => TRUE,
        method_opt => 'FOR ALL COLUMNS SIZE AUTO'
    );
END;
/
-- 查看最后收集时间
SELECT table_name, last_analyzed, num_rows FROM user_tables WHERE table_name = 'ORDERS';
```

## 7.2 直方图

当某列数据分布**严重倾斜**（比如 90% 的订单状态是 `DONE`，只有 1% 是 `NEW`），优化器如果只知道"有 3 个 distinct 值"，就会按平均值估算，导致计划选错。**直方图（Histogram）**记录列的分布细节。

```sql
-- 查看列的直方图信息
SELECT column_name, num_distinct, histogram, num_buckets
FROM user_tab_col_statistics
WHERE table_name = 'ORDERS';
```

`method_opt => 'FOR ALL COLUMNS SIZE AUTO'` 让 Oracle 自动判断哪些列需要直方图。手动指定 `SIZE 254` 可以强制建最多 254 个桶的直方图，但不要对所有列都建，会增加收集开销和硬解析成本。

## 7.3 统计信息为什么会"坏"

- **大批量导入后没收集**：优化器还以为表只有几千行。
- **数据分布剧变**：新增了一个占比 50% 的状态值，旧直方图失效。
- **绑定变量窥探（Bind Peeking）**：第一次执行时窥探绑定值生成计划，后续复用该计划，但值分布不同就失真。12c 之后的**自适应游标共享（ACS）**会为不同绑定值生成多个计划缓解这个问题。
- **手动改过统计信息**：有人用 `DBMS_STATS.SET_TABLE_STATS` 写死了行数，导致后续全部失真。

> **注意**：不要随意用 `DBMS_STATS.SET_*_STATS` 伪造统计信息。短期看能让计划"听话"，长期会让优化器彻底失去判断力。要用 hint 就在 SQL 里局部用，不要污染全局统计。

# 8. Hint

Hint 是写在 SQL 注释里的优化器指令，格式 `/*+ HINT_NAME(...) */`。它的定位是"**在优化器判断错误时的最后手段**"，不是日常优化工具。

```sql
-- 强制走某个索引
SELECT /*+ INDEX(orders idx_orders_user_id) */ *
FROM orders WHERE user_id = 1;

-- 强制全表扫描
SELECT /*+ FULL(orders) */ * FROM orders WHERE status = 'NEW';

-- 指定连接顺序与方法
SELECT /*+ LEADING(a b) USE_NL(b) INDEX(b idx_b) */ *
FROM a JOIN b ON a.id = b.a_id;

-- 并行
SELECT /*+ PARALLEL(orders 4) */ COUNT(*) FROM orders;
```

| Hint | 作用 | 风险 |
|------|------|------|
| `INDEX(t idx)` | 强制用索引 | 索引被删或数据变化后计划更差 |
| `FULL(t)` | 强制全表扫描 | 大表全扫可能拖垮 I/O |
| `LEADING(a b)` | 指定连接顺序 | 依赖表别名，改 SQL 易失效 |
| `USE_NL` / `USE_HASH` | 指定连接算法 | 选错算法性能差一个数量级 |
| `PARALLEL` | 并行执行 | 占满 CPU，影响其他会话 |

> **行业实践**：Hint 应该作为"临时止血"或"关键 SQL 固化计划"的手段，并且要**记录原因、定期复核**。更稳妥的做法是用 **SQL Plan Baseline**（计划基线）锁定经过验证的执行计划，让优化器在基线内选择，而不是硬编码 hint。hint 写错（表别名不匹配、索引名拼错）会被静默忽略，这也是排查时的常见坑。

# 9. SQL 优化

## 9.1 优化顺序

1. **确认问题 SQL**：从 `V$SQL` 按 `ELAPSED_TIME`、`BUFFER_GETS` 排序找 Top SQL。
2. **看执行计划**：`DBMS_XPLAN.DISPLAY_CURSOR` 看实际计划与实际行数。
3. **判断瓶颈**：是全表扫描、回表太多、连接顺序错、还是排序/哈希落盘。
4. **改 SQL 或加索引**：优先改 SQL 和补索引，最后才用 hint。
5. **验证**：对比改前改后的逻辑读、物理读、耗时。

```sql
-- 找消耗逻辑读最多的 SQL
SELECT sql_id, executions, buffer_gets, disk_reads,
       ROUND(elapsed_time / 1e6, 2) AS elapsed_sec, sql_text
FROM v$sql
ORDER BY buffer_gets DESC
FETCH FIRST 10 ROWS ONLY;
```

## 9.2 常见优化手段

- **只查需要的列**：避免 `SELECT *`，减少回表和网络传输；如果索引已覆盖所有列，可走索引快速全扫描。
- **避免在列上做函数/运算**：改写条件让索引可用。
- **分页用游标**：深分页用 `WHERE id > :last_id` 而非大 `OFFSET`。
- **批量代替逐行**：`FORALL`、`INSERT ... SELECT`、`MERGE`。
- **减少排序**：排序很贵，能在索引里预排序就利用索引顺序。
- **分区裁剪**：分区表查询带上分区键，避免全分区扫描。

## 9.3 一个覆盖索引的例子

```sql
-- 查询只需要 user_id 和 status
SELECT user_id, status FROM orders WHERE user_id = 1;

-- 建复合索引，查询可只读索引不回表
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

执行计划里会出现 `INDEX RANGE SCAN`，且没有 `TABLE ACCESS BY INDEX ROWID`，这就是**索引覆盖（Covering Index）**，逻辑读显著下降。

# 10. 练习

**练习一**：建一张 10 万行的 `orders` 表，分别在 `user_id` 上建普通索引、在 `status` 上建位图索引，执行 `WHERE user_id = ?` 与 `WHERE status = ?`，用 `DBMS_XPLAN` 对比访问路径，并解释为什么位图索引不适合高频更新。

**练习二**：写一条 `WHERE UPPER(email) = ?` 的查询，先看它走全表扫描，再建函数索引，确认计划变为索引扫描；然后改写成 `WHERE email = LOWER(?)`，验证普通索引也能生效，说明两种方案的成本差异。

**练习三**：用 `DBMS_XPLAN.DISPLAY_CURSOR(..., 'ALLSTATS LAST')` 观察一条多表连接 SQL 的 `E-Rows` 与 `A-Rows`，找出一处估算偏差超过 10 倍的地方，说明可能是统计信息问题还是列相关性问题。

# 11. 小结

| 主题 | 核心结论 |
|------|---------|
| 索引本质 | 用写开销换读性能，叶子节点存 key + ROWID |
| B-tree | 适合高基数列，复合索引遵循最左前缀 |
| 位图 | 适合低基数、只读报表，OLTP 高并发写慎用 |
| 函数索引 | 表达式必须与查询完全一致，函数需确定性 |
| 执行计划 | 从内到外读缩进，重点看 Rows、Cost、谓词 |
| 统计信息 | CBO 的基础，过期会导致计划突变，直方图解决倾斜 |
| Hint | 最后手段，易失效，优先用计划基线固化 |
| 优化顺序 | 找 SQL → 看计划 → 判瓶颈 → 改 SQL/索引 → 验证 |

索引与执行计划解决的是"单条 SQL 快不快"，而**分区与高可用**解决的是"数据量大了、机器坏了怎么办"。下一篇进入分区表、RAC、Data Guard，讲清它们各自的适用边界，以及为什么"上 RAC"不等于"高可用就万事大吉"。
