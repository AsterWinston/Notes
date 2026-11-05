# 1. 索引基础

索引的本质是**用额外的写入成本和存储空间，换取查询时的定位效率**。PG 有一个必须记住的结构特征：**所有索引都是二级索引**，叶子节点存的是行在堆表中的物理位置 `ctid`，而不是整行数据。所谓"聚簇索引"在 PG 里并不存在，`CLUSTER` 只是按某个索引顺序重写一次表，之后新插入的数据不会自动维持顺序。

```text
B-tree 索引                     堆表（heap）
┌──────────────┐   ctid   ┌──────────────────┐
│ key=100 ─────┼─────────▶│ (0,1) id=100 ... │
│ key=200 ─────┼────┐     │ (0,2) id=150 ... │
│ key=300 ─────┼──┐ └────▶│ (1,1) id=200 ... │
└──────────────┘  └──────▶│ ...              │
                          └──────────────────┘
```

这解释了两个现象：主键查询也可能访问两个位置（索引 + 堆表），除非走 **Index Only Scan**；索引不含完整行，`SELECT *` 几乎必然回表。

| 收益 | 代价 |
|------|------|
| 等值/范围查询降为对数级定位 | 每个索引额外占用磁盘与共享缓冲区 |
| 索引有序，可加速 `ORDER BY`、`MIN/MAX` | 每次写入都要维护索引 |
| 唯一索引把业务约束交给数据库 | 更新索引列会放大写放大，并破坏 HOT 更新 |

> **设计原则**：索引应由**查询模式**驱动，而不是由"字段看起来重要"驱动。先用 `pg_stat_statements` 找出高频、高耗时的 SQL，再针对性建索引。

# 2. 索引类型

## 2.1 B-tree

B-tree 是默认类型，覆盖 90% 的场景。它支持等值、范围、前缀 `LIKE 'abc%'`（在 `C` collation 或 `text_pattern_ops` 下）、`ORDER BY`、`IS NULL`。

```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_customer_created ON orders (customer_id, created_at DESC);
CREATE UNIQUE INDEX uk_users_email ON users (lower(email));
```

B-tree 不适合数组包含、JSONB 键存在、全文检索、几何包含，这些要用 GIN 或 GiST。

## 2.2 GIN

**GIN（Generalized Inverted Index，通用倒排索引）** 为"一个值里包含多个元素"的场景设计：数组、JSONB、`tsvector`、`pg_trgm`。它把每个元素映射到包含它的行，查询 `@>`、`?`、`@@` 时非常快。

```sql
CREATE INDEX idx_article_tags ON article USING gin (tags);
SELECT * FROM article WHERE tags @> ARRAY['pg'];
CREATE INDEX idx_orders_detail ON orders USING gin (detail);
SELECT * FROM orders WHERE detail @> '{"channel": "app"}';
```

GIN 写入成本较高，因为一个值会拆成多个索引项。默认开启 `fastupdate`，新条目先进入待处理列表，由后台批量合并；这提升写入吞吐，但让刚插入的数据在合并前查询变慢。

## 2.3 GiST

**GiST（Generalized Search Tree）** 是一套可扩展的平衡树框架，适合空间、范围、相似性类数据：几何类型、范围类型、`EXCLUDE` 约束、最近邻搜索。

```sql
CREATE INDEX idx_booking_period ON booking USING gist (period);
CREATE INDEX idx_shop_geom ON shop USING gist (geom);

SELECT id, name FROM shop ORDER BY geom <-> ST_Point(121.47, 31.23) LIMIT 10;
```

GiST 索引可能是**有损的（lossy）**：索引只给候选集，还需回表精确判断，`EXPLAIN` 里会看到 `Rows Removed by Index Recheck`。

## 2.4 BRIN

**BRIN（Block Range Index）** 不索引每一行，而是按连续数据块记录 min/max 摘要，体积极小，适合**物理顺序与查询条件高度相关**的大表，典型是时间序列。

```sql
CREATE INDEX idx_logs_created_brin ON logs USING brin (created_at) WITH (pages_per_range = 32);
```

随机顺序插入时，BRIN 几乎退化为全表扫描。用 `pg_stats.correlation` 判断相关性，接近 1 说明效果好。Hash 索引只支持等值查询，SP-GiST 适合非平衡空间分区，实际使用面都较窄。

## 2.5 类型选型

| 场景 | 首选 | 备选 |
|------|------|------|
| 主键、外键、时间范围、排序 | B-tree | — |
| 数组包含、JSONB 键、全文检索 | GIN | GiST（全文） |
| 几何、范围、最近邻、排他约束 | GiST | SP-GiST |
| 超大时间序列表、物理有序 | BRIN | B-tree（分区表） |

# 3. 索引设计

## 3.1 联合索引的列顺序

联合索引 `(a, b, c)` 可以服务 `a`、`(a,b)`、`(a,b,c)` 的查询，但不能直接服务只查 `b` 的查询。顺序通常按：**等值条件列最前 → 区分度高的列靠前 → 范围条件列放最后 → 让索引顺序匹配 `ORDER BY`**。

```sql
CREATE INDEX idx_orders_tenant_status_created ON orders (tenant_id, status, created_at DESC);
```

## 3.2 部分索引与表达式索引

部分索引只索引满足条件的行，体积小、维护成本低，适合"热数据少、冷数据多"。

```sql
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';

-- 查询必须包含同样的条件，优化器才会用这个索引
SELECT * FROM orders
WHERE status = 'pending' AND created_at < now() - interval '1 hour';
```

当查询对列做了函数运算，普通索引会失效，可以让索引直接存表达式结果：

```sql
CREATE INDEX idx_users_email_lower ON users (lower(email));
SELECT * FROM users WHERE lower(email) = 'alice@example.com';
```

> **注意**：表达式索引要求查询表达式与索引定义**完全一致**，`lower(email)` 和 `email::text` 不会被视为同一个表达式。

## 3.3 覆盖索引与并发建索引

PG 11 起支持 `INCLUDE`，把非键列附加到索引叶子节点：

```sql
CREATE INDEX idx_orders_customer_include ON orders (customer_id) INCLUDE (amount, created_at);
SELECT amount, created_at FROM orders WHERE customer_id = 1001;
```

但 **Index Only Scan 还依赖可见性映射**：只有对应数据页被标记为"全部可见"时，PG 才敢跳过堆表。`UPDATE/DELETE` 会清除标记，`VACUUM` 会重新设置，所以刚大批量更新过的表可能退化为 `Index Scan`。

普通 `CREATE INDEX` 持有 `SHARE` 锁，阻塞写入。生产大表应使用 `CONCURRENTLY`：

```sql
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
```

并发建索引失败会留下 `INVALID` 索引，查询不会使用它，但会拖慢写入。用 `SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;` 找出并 `DROP INDEX CONCURRENTLY` 后重建。

# 4. EXPLAIN 与执行计划

## 4.1 EXPLAIN 与 EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 1001;
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders WHERE customer_id = 1001;
```

| 选项 | 作用 |
|------|------|
| `ANALYZE` | 实际执行，给出真实时间与行数 |
| `BUFFERS` | 显示共享缓冲区命中/读盘块数 |
| `FORMAT JSON` / `WAL` | 便于程序解析 / 显示产生的 WAL 量 |

> **警告**：`EXPLAIN ANALYZE` 会真正执行 SQL。对 `UPDATE/DELETE` 使用时要包在事务里并回滚，否则数据会被改掉。

## 4.2 扫描与连接节点

| 节点 | 含义 | 何时出现 |
|------|------|---------|
| `Seq Scan` | 顺序扫描整表 | 无可用索引或优化器认为全表更快 |
| `Index Scan` | 走索引并回表 | 选择性好的等值/范围查询 |
| `Index Only Scan` | 只读索引，不回表 | 覆盖索引 + 可见性映射干净 |
| `Bitmap Heap Scan` | 先收集候选页，再按页批量回表 | 中等选择性、多条件组合 |
| `Nested Loop` | 外表小、内表连接键有索引 | 外表估算错误时会放大随机 I/O |
| `Hash Join` | 中大表等值连接 | `work_mem` 不足时哈希表落盘 |
| `Merge Join` | 两侧已按连接键排序 | 依赖排序或索引顺序 |

## 4.3 读懂代价与缓冲区

```text
Hash Join  (cost=1.09..2.20 rows=1 width=40) (actual time=0.030..0.032 rows=1 loops=1)
  Buffers: shared hit=3
  ->  Seq Scan on orders o  (cost=0.00..1.05 rows=5 width=24) (actual time=0.010..0.011 rows=5 loops=1)
```

阅读顺序是**从内到外、从下到上**：`cost=启动代价..总代价` 是抽象单位，不是毫秒；`rows` 是每个 loop 的估算行数；`loops` 大于 1 时总时间约等于 `actual time × loops`；`Buffers: shared hit` 是缓存命中，`read` 是读盘。估算 `rows` 与实际相差一个数量级，通常意味着统计信息不准。

实战先看三个问题：**哪一步 actual rows 最大**（优化重点）、**估算准不准**（计划是否选错）、**读盘多不多**（缓存与索引是否足够）。

# 5. 统计信息与慢查询

## 5.1 ANALYZE

优化器基于统计信息估算选择性与行数。统计信息由 `ANALYZE` 收集，autovacuum 也会自动触发。大批量导入或数据分布剧变后，应手动执行：

```sql
ANALYZE orders;

SELECT attname, n_distinct, most_common_vals, correlation
FROM pg_stats WHERE tablename = 'orders';
```

`default_statistics_target` 默认 100，决定直方图与 MCV 的采样粒度。分布极不均匀的列可以单独提高：`ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;`，然后重新 `ANALYZE`。

单列统计无法表达**列间相关性**。例如 `city` 与 `district` 强相关，优化器会默认独立，严重低估组合条件的行数。PG 10 起支持扩展统计：

```sql
CREATE STATISTICS st_orders_city_district (dependencies, ndistinct)
ON city, district FROM orders;
ANALYZE orders;
```

`pg_stats.correlation` 表示列值与物理行序的相关程度，接近 1 或 -1 说明物理有序、范围扫描 I/O 连续；接近 0 说明物理随机，优化器可能转向全表扫描。它也是判断 BRIN 是否有效的关键指标。

## 5.2 日志与 pg_stat_statements

```ini
log_min_duration_statement = 500ms   # 记录超过 500ms 的语句
log_checkpoints = on
log_lock_waits = on
```

`pg_stat_statements` 按归一化 SQL 聚合执行次数、耗时、行数、缓冲区访问，是定位慢查询的第一入口：

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT queryid, calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       rows, shared_blks_hit, shared_blks_read, left(query, 100) AS sql
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

> **PG 13 起字段改名**：`total_time` 改为 `total_exec_time`，`mean_time` 改为 `mean_exec_time`。老监控脚本升级时要同步改。需要抓具体执行计划时，可临时开启 `auto_explain`，把慢 SQL 的计划写进日志。

# 6. 连接池

## 6.1 为什么 PG 必须用连接池

PG 是一连接一进程，每个连接都要占用独立内存、参与进程调度，`max_connections` 开到几千会让上下文切换和内存压力失控。应用侧连接池解决"复用连接"，数据库侧连接池（PgBouncer）解决"把大量客户端复用成少量真实连接"。

```text
应用实例 × N
   │ 每实例 HikariCP 10 个连接
   ▼
PgBouncer（事务级复用）
   │ 只维持 50 个真实连接
   ▼
PostgreSQL（max_connections = 100）
```

## 6.2 PgBouncer 的三种模式

| 模式 | 复用粒度 | 兼容性 | 适用 |
|------|---------|--------|------|
| session | 客户端连接期间独占一个服务端连接 | 最好 | 兼容性优先 |
| transaction | 每个事务归还连接 | 较好 | **大多数 OLTP 应用** |

事务模式下，会话级状态（临时表、`SET`、`LISTEN/NOTIFY`、会话级 advisory lock）不能跨事务保持。应用依赖这些特性时，要么用会话模式，要么把状态显式存起来。

## 6.3 HikariCP 配置

HikariCP 的关键参数是 `maximum-pool-size`（按数据库连接预算分配）、`connection-timeout`（获取连接超时，快速失败）、`max-lifetime`（小于服务端或中间件的连接寿命）和 `keepalive-time`（定期探活，防止被防火墙断开）。连接串里显式加 `sslmode=require` 与 `ApplicationName`，后者会出现在 `pg_stat_activity` 里，便于排查。

> **行业实践**：连接池总预算按 `max_connections × 0.8 / 应用实例数` 估算，再留出运维与复制连接。盲目把 `maximum-pool-size` 设成 100，只会在故障时把数据库压垮——连接池的意义是**限流**，不是无限扩容。

# 7. 常见错误与排错

| 现象 | 根因 | 处理 |
|------|------|------|
| 建了索引仍走 Seq Scan | 选择性差、统计过期、条件对列做了运算 | `ANALYZE`，改写条件，或建表达式索引 |
| Index Only Scan 仍有很多 heap fetch | 可见性映射未更新 | `VACUUM` 表，减少频繁更新 |
| 索引存在但用不上 | 联合索引列顺序不匹配、类型不一致 | 调整列顺序，统一参数类型 |
| `CREATE INDEX CONCURRENTLY` 卡住 | 有长事务持有旧快照 | 先清理长事务再重试 |
| 查询突然变慢，计划从 Index 变 Seq | 数据量增长或统计信息变化 | 对比 `EXPLAIN`，更新统计，评估分区 |
| 连接数被打满 | 连接池过大、连接泄漏 | 查 `pg_stat_activity`，修正池大小与超时 |

```sql
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size,
       idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes WHERE relname = 'orders'
ORDER BY pg_relation_size(indexrelid) DESC;
```

# 8. 练习与自检

**练习一**：建一张 100 万行订单表，分别测试 `WHERE customer_id = ?`、`WHERE status = ?`、`WHERE created_at > ?` 在无索引、B-tree、BRIN 下的计划与耗时，记录差异并解释原因。

**练习二**：为 `WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC LIMIT 20` 设计联合索引，用 `EXPLAIN (ANALYZE, BUFFERS)` 验证它消除了排序节点。

**练习三**：用 `pg_stat_statements` 找出总耗时最高的三条 SQL，分析 `shared_blks_read` 与 `calls`，判断优化方向是加索引、改 SQL 还是加缓存。

**自检清单**：

- 能说清 PG 为什么没有聚簇索引，以及 `ctid` 在索引中的作用；
- 能根据查询形态在 B-tree、GIN、GiST、BRIN 之间选择；
- 能读懂 `EXPLAIN` 的 cost、rows、actual time、loops、buffers；
- 知道 Index Only Scan 依赖可见性映射，`VACUUM` 会影响它。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| 索引组织 | 全部是二级索引，叶子存 `ctid`，回表是常态 |
| 类型选型 | B-tree 通用，GIN 管多值与全文，GiST 管空间与范围，BRIN 管大而有序列 |
| 设计手法 | 联合索引看列序，部分索引省空间，表达式索引救函数条件，INCLUDE 做覆盖 |
| EXPLAIN | 先看最大行数节点，再看估算偏差，最后看缓冲区读盘 |
| 统计信息 | 估算决定计划，`ANALYZE`、统计目标、扩展统计是三大手段 |
| 连接池 | PG 进程模型决定连接昂贵，应用池 + PgBouncer 是标准组合 |

索引解决的是"结构化数据怎么查得快"。但现代业务还有大量半结构化数据与文本搜索需求：订单扩展字段、商品多语言描述、日志关键词检索。下一篇进入 **JSONB 与全文检索**，看看 PG 能在不引入 Elasticsearch 的情况下把这类需求做到什么程度，边界又在哪里。