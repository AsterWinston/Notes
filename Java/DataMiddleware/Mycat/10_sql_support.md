# 1. SQL 支持总览：中间件不是数据库

**理解 Mycat SQL 能力的正确心智模型是：它是"SQL 改写器 + 结果归并器"，不是查询引擎**。一条 SQL 进来，Mycat 解析语法树 → 按分片规则计算目标节点 → 把逻辑表名改写为物理表名、把 SQL 拆成各分片可独立执行的子 SQL 下发 → 把多个结果集按语义合并返回。凡是"合并语义"能定义的（求和、归并排序、去重拼接）就能支持；凡是语义只在单机执行器里存在（窗口函数的复杂帧、存储过程中的流程控制）就无法支持。

```text
SQL 请求
  │
  ▼
解析层（1.6：RumpParser + Druid AST；2.x：Calcite Parser）
  ├── 简单语句（能整句下发） ────────────→ 原样/改表名后下推
  ├── 可拆分聚合（SUM/COUNT/GROUP BY） ──→ 改写为下推 + 二次聚合
  ├── 可归并排序（ORDER BY/LIMIT） ──────→ 改写为下推 + 堆归并
  └── 无法拆分的语法 ────────────────────→ 中间件级执行或报错
```

**1.6 与 2.x 的能力分界**要先说清：**Mycat 1.6 的 SQL 支持以"简单直白"为设计目标**，跨库 JOIN、子查询、聚合函数大多只做保守改写或明确拒绝，复杂语句的报错信息也偏晦涩（"UnSupport SQL"是社区看到最多的一行日志）；**2.x 引入 Calcite 后，谓词下推、列裁剪、聚合分解在正确性与覆盖面上一档**，接近 ShardingSphere 水准。以 1.6 为准绳的存量系统中，"SQL 是否被支持"必须在上线前逐条验证，而不是按 MySQL 直觉推断。

# 2. 完全支持的 SQL

## 2.1 单分片路由的 CRUD

**只要 WHERE 中携带分片键、或 INSERT 的 VALUES 中包含分片键，下列语句全部等价于单库执行**：

```sql
-- SELECT：分片键精确路由，原句下发
SELECT id, name FROM users WHERE user_id = 7;

-- INSERT：分片键在 VALUES 里，Mycat 计算路由后整句下发
INSERT INTO users(id, name, age) VALUES (1, 'Alice', 25);

-- UPDATE / DELETE：带分片键
UPDATE users SET age = 26 WHERE user_id = 7;
DELETE FROM users WHERE user_id = 7;

-- REPLACE 同理
REPLACE INTO users(id, name, age) VALUES (1, 'Alice', 26);
```

## 2.2 广播型查询与聚合下推

不带分片键时语句被发往所有分片，Mycat 负责结果合并。聚合的支持规则是**"能否分解为局部聚合 + 全局再聚合"**：

| 函数/子句 | 支持 | 拆分与合并方式 |
|-----------|------|---------------|
| `COUNT(*)` | 完全 | 各分片 COUNT → 全局求和 |
| `SUM(col)` | 完全 | 各分片 SUM → 全局求和 |
| `MAX/MIN(col)` | 完全 | 各分片取极值 → 全局取极值 |
| `AVG(col)` | 完全（改写） | 内部改写为 `SUM/COUNT` 下发，全局重算 AVG |
| `GROUP BY col` | 完全（内存压力） | 各分片分组 → 中间件按组键再聚合（全组驻留内存） |
| `GROUP_CONCAT` | 部分 | 去重语义与截断长度（`group_concat_max_len`）不可靠，慎用 |
| `HAVING` | 部分 | 依赖 GROUP BY 改写后再过滤，复杂 HAVING 报错 |
| `DISTINCT` | 完全（代价高） | 全分片去重集合拉到中间件合并，内存/临时文件开销随数据量涨 |

```sql
-- AVG 的真实下发形态（1.6 改写规则）
SELECT city, AVG(age) FROM users GROUP BY city;
-- 发往每个分片：SELECT city, AVG(age), COUNT(age) FROM users GROUP BY city
-- 中间件用 SUM=AVG×COUNT 重算全局 AVG
```

## 2.3 排序、分页与 limit

**ORDER BY 采用"各分片各取前 OFFSET+LIMIT 条 → 归并排序 → 截取目标窗口"的改写**，正确性没问题，代价是分片数 ×（OFFSET+LIMIT）行的拉取量：

```sql
-- 下发改写：LIMIT 10 OFFSET 100 → 每分片 LIMIT 110
SELECT * FROM users ORDER BY age DESC LIMIT 10 OFFSET 100;

-- 游标式分页（推荐替代深分页）：
SELECT * FROM users WHERE age > 30 ORDER BY age LIMIT 10;
```

排序键不含在 SELECT 列表时（`ORDER BY` 非投影列），1.6 会**自动把排序列追加进下发 SQL 的 SELECT 列**，归并完成后从结果集中再删除——这导致"客户端拿到列数与 SELECT 书写不一致"类怪现象的排查思路：先 EXPLAIN 看下发列。

# 3. 有限支持：需要理解代价的 SQL

```sql
-- JOIN：仅在 ER 对齐/全局表/同分片键时可下推为本地 JOIN（第 8 篇）
SELECT u.name, o.amount FROM users u JOIN orders o ON u.id = o.user_id;

-- 子查询：IN/EXISTS 若内层表的分片键与外层关联，改写为广播半连接；
-- 关联子查询（correlated）大概率报 UnSupport
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE amount > 100);

-- UNION / UNION ALL：各分片各执行后合并去重，支持但内存敏感
SELECT id FROM users WHERE city = 'BJ'
UNION ALL
SELECT id FROM users WHERE city = 'SH';
```

**跨库 JOIN 的处理梯度**（1.6 路由器的真实决策顺序）：

```text
JOIN 两表：
  1. 两表 ER 对齐（parent 关系）        → 下推，单分片本地 JOIN ✅
  2. 一表为全局表                      → 下推到各分片，分片表×全局表本地 JOIN ✅
  3. 分片键等价（WHERE 已限定同一分片）  → 下推到该分片本地 JOIN ✅
  4. 其余                              → 中间件级 JOIN：驱动表全量拉回，
                                         按被驱动表分片键分批并行查询，
                                         内存拼接（无报错，但可能 OOM）⚠️
```

> **注意**：第 4 档是"静默的性能陷阱"——语句执行成功、结果正确、耗时指数级增长。**上线前用 `EXPLAIN` 检查 JOIN 的落地形态**，输出中出现多个节点且执行计划带 `JOIN` 中间算子的，必须回到第 1~3 档重构数据分布。

**函数与表达式的支持规则**：WHERE 里的普通函数（`UPPER(name)='ALICE'`、`DATE(create_time)='2024-01-01'`）会原样下推，**但包裹了分片键的函数会让分片键"不可见"**——`WHERE MOD(user_id, 10) = 7` 无法算出分片值，退化为全分片广播。"分片键裸奔在 WHERE 里"是 Mycat SQL 编写第一军规。

# 4. 不支持的语法清单

**以 1.6.7.x 为基准的不支持列表**（2.x 已补齐一部分，见备注）：

| 语法 | 1.6 | 2.x | 替代方案 |
|------|-----|-----|---------|
| 存储过程 `CALL proc()` | ✗ | ✗ | 逻辑上移应用层 |
| 触发器 `CREATE TRIGGER` | ✗（透传单节点，无全局语义） | ✗ | 应用层/消息 |
| 视图 `CREATE VIEW`（逻辑视图） | ✗ | 部分（透传） | 全局表 + 改写查询 |
| 自定义函数 UDF | ✗ | ✗ | Java 层计算 |
| `LOAD DATA INFILE` | ✗ | ✗ | 应用批量 INSERT |
| `LOCK TABLES` / `UNLOCK` | ✗ | ✗ | 用事务替代 |
| `GET_LOCK()` 命名锁 | ✗ | ✗ | Redis/ZK 分布式锁 |
| `SELECT ... INTO OUTFILE` | ✗ | ✗ | 导出工具 |
| 多表 `UPDATE t1,t2`/`DELETE t1 FROM ...`（跨分片） | ✗ | 部分 | 拆成两语句 |
| 跨库分布式 DDL（改表结构同步所有分片） | ✗（需逐库执行） | ✓（可广播） | 自研 DDL 脚本 |
| 窗口函数 `OVER()` | ✗ | 部分 | 应用层排序计算 |
| 事务内 SAVEPOINT | ✗ | ✗ | 业务拆分 |
| `SET` 会话变量透传 | 部分（白名单） | 部分 | 避免依赖 |
| 临时表 `CREATE TEMPORARY TABLE` | ✗ | ✗ | 中间结果走 Redis |

```sql
-- 错误示范：跨分片多表删除
DELETE u, o FROM users u JOIN orders o ON u.id = o.user_id WHERE u.id = 1;

-- 正确做法：拆成两条单表语句
DELETE FROM orders WHERE user_id = 1;
DELETE FROM users WHERE id = 1;
```

**DDL 的特殊说明**：1.6 的 `CREATE TABLE`/`ALTER TABLE` 只发到默认节点，**不会自动在全部署分片上执行**——所有分片建表必须靠外部脚本（foreach 分片实例执行）或运维平台完成。这是"MySQL 习惯用 Mycat"的最大行为差异，新人几乎必踩：`ALTER TABLE` 加列在 dn1 成功、dn2 未执行，随后路由到 dn2 的写入报"Unknown column"。

# 5. SQL 直通模式（透传）

**对确认不需要路由与改写、且性能敏感的语句，Mycat 1.6.7.4+ 支持 hint 级透传**：把整条 SQL 原样发给某个指定节点，跳过解析/改写/归并流水线：

```sql
-- 强制走指定数据节点，SQL 原样下发不解析
/*!mycat:datanode=1*/ SELECT COUNT(*) FROM big_log_table WHERE dt='2024-06-01';

-- 透传模式开关（会话级）：后续 SQL 只路由不改写（谨慎使用）
/*!mycat:sql=SELECT 1*/ SET @mycat.sql.transparent = true;

-- schema.xml 上 dbType="dataSource" 时，整个 schema 退化为"连接池代理"
-- 类似 dbproxy，完全不做 SQL 语义处理
```

**设计考量**：

- **透传省掉解析与归并开销**，对单节点大结果集流式查询收益明显
- **透传放弃了保护**：SQL 错误不再被提前发现、结果不再归并、读写分离按 balance 而非语句语义——只用于"明确知道自己在干什么"的批量导出类语句
- MySQL 协议兼容性调优（`SET NAMES`、`SELECT @@version` 等）部分依赖透传语义，驱动连接 Mycat 报奇怪的元数据错误时，先检查会话变量透传配置

# 6. 执行计划与诊断

```sql
-- EXPLAIN：查看 SQL 的节点路由与改写形态
EXPLAIN SELECT * FROM users WHERE user_id = 1;
-- +------+---------------------------------------------+
-- | NODE | DATA                                        |
-- +------+---------------------------------------------+
-- | dn1  | SELECT * FROM users WHERE user_id = 1       |
-- +------+---------------------------------------------+

EXPLAIN SELECT * FROM orders ORDER BY create_time DESC LIMIT 10;
-- 输出多个节点 + Merge 算子 → 全分片扫描 + 归并排序

-- 9066 管理端口观察慢 SQL 全貌
SHOW @@SQL;
SHOW @@SQLSUMMARY;
SHOW @@TIME.PATTERN;   -- SQL 模板按节点聚合统计（1.6.7.x）
```

**排障套路**：先 `EXPLAIN` 看路由面（广播了没有）、再 `SHOW @@SQL` 看执行面（卡在哪个节点）、最后对比后端真实执行计划 `EXPLAIN`（MySQL 自己的）。**Mycat 的慢九成来自全分片扫描，而不是网络归并**。

# 7. 兼容性最佳实践

1. **WHERE 永远带分片键裸值**：`user_id = ?` 而非 `func(user_id) = ?`；报表类查询走独立的 T+1 数仓，不硬扛 OLAP 广播
2. **LIMIT 与 OFFSET 控制窗口**：跨分片实际拉取量 = 分片数 × (OFFSET+LIMIT)，超过百行的深分页改游标式
3. **JOIN 三选一**：ER 对齐、全局表、或干脆在写入时冗余字段——避免中间件级 JOIN 进入生产路径
4. **DDL 纳入发布流程**：所有分片同步执行并校验 schema 一致性（pt-online-schema-change 逐节点跑），禁止依赖 Mycat "帮你广播"
5. **聚合尽量在单分片内完成**：报表口径按分片键分组统计，让"全局聚合"只发生在数据聚合服务而非交易库
6. **SQL 测试基线**：把业务 TOP 200 SQL 在预发环境用真实分片数跑一遍 `EXPLAIN` + 正确性比对，比任何语法文档都可靠

> **行业实践**：使用 Mycat 的团队普遍沉淀一份"MySQL 方言限制清单"给业务开发（禁存储过程、禁临时表、禁跨分片多表 DELETE、分片键禁止函数包裹），配合 SQL 审核平台（如 Archery）在上线卡点强制执行——**Mycat 的 SQL 支持边界不是"知道就好"的知识，而是必须有工具拦截的工程约束**。

# 8. 小结

**Mycat 的 SQL 支持面由"可拆分性"决定**：带分片键的单语句 CRUD 完全透明；聚合函数遵循"局部聚合 + 全局再聚合"改写（AVG 拆成 SUM/COUNT 是代表案例）；ORDER BY/DISTINCT/GROUP BY 支持但代价随分片数线性增长；JOIN 依赖 ER 表、全局表或分片键对齐下推，否则落入静默的中间件级 JOIN；存储过程、触发器、临时表、命名锁、跨分片多表 DML 在 1.6 上明确不支持，DDL 更需人工广播到全部分片。2.x 借助 Calcite 把下推与改写做得更完备，但边界依然存在——**用 Mycat 的工程正确姿势是：把 TOP SQL 全部 EXPLAIN 一遍验证路由形态，把禁写语法沉淀进审核平台，让"中间件不是数据库"成为团队的共同常识**。
