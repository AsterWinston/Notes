# 1.索引概述

索引（Index）是帮助 MySQL **高效获取数据**的**数据结构**（有序）。在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，使得可以在这些数据结构上实现高级查找算法。

**类比**：就像一本字典的"拼音索引"或"部首索引"。如果没有索引，查一个字需要从第一页开始逐页翻找（全表扫描）。有了索引，你可以直接定位到对应页码。

## 1.1索引的优缺点

| 优点 | 缺点 |
|------|------|
| **提高查询效率**：将 O(N) 的全表扫描降到 O(log N) | **占用磁盘空间**：索引本身是独立的数据结构，需要额外存储 |
| **降低排序成本**：索引本身有序，ORDER BY 可直接使用索引顺序 | **降低写入性能**：INSERT/UPDATE/DELETE 时需要同时维护索引结构 |
| **加速分组和连接**：GROUP BY 和 JOIN 可利用索引避免临时表 | **并非越多越好**：过多的索引导致优化器选择困难，且占用内存（Buffer Pool） |

> **核心权衡**：索引的本质是**以空间换时间**、**以写入性能换查询性能**。因此索引设计需根据实际查询模式精确选择，而不是盲目添加。

## 1.2没有索引时的问题

```sql
-- 假设 user 表有 100 万行，email 没有索引
SELECT * FROM user WHERE email = 'user@example.com';
```

没有索引时，MySQL 执行器会调用 InnoDB 引擎 API，从第一页开始逐页读取、逐行比较，直到找到目标行。这就是**全表扫描**，100 万行数据可能需要数秒甚至十几秒。

# 2.索引结构

## 2.1B+Tree（MySQL 中最核心的索引结构）

### 为什么是 B+Tree？

| 数据结构 | 为什么不用 |
|---------|-----------|
| **二叉搜索树** | 顺序插入退化为链表，O(N) |
| **红黑树（二叉平衡树）** | 树高度大（100 万数据 ~20 层），磁盘 IO 次数多 |
| **Hash 表** | 不支持范围查询，键冲突多时性能差 |
| **B-Tree** | 非叶子节点也存数据，每个节点能存的键更少，树更高 |
| **B+Tree** | 所有数据在叶子节点，非叶子节点只存键，单节点能存更多键，树更矮；叶子节点形成有序链表，支持范围查询 |

### B+Tree 的结构特性

1. **所有数据存储在叶子节点**，非叶子节点只存储索引键（不存数据），一个 16KB 的页能存更多键
2. **叶子节点之间通过双向链表连接**，形成有序序列，支持高效的范围查询和排序
3. **树的高度极低**：一个 16KB 页、假设键为 BIGINT（8 字节）+ 指针 6 字节，大约能存 1170 个键。3 层 B+Tree 能存约 `1170³ ≈ 16 亿` 行数据

**磁盘 IO 次数**：查询一行数据，只需要进行与**树高度相同次数**的磁盘 IO。通常 3~4 层就够用，每次 IO 读一页 16KB。

### InnoDB 中 B+Tree 的两层含义

| 索引类型 | 叶子节点存储内容 | 特点 |
|---------|----------------|------|
| **聚簇索引（Clustered Index）** | **完整行数据** | 必须有且仅有一个；主键索引即聚簇索引 |
| **二级索引（Secondary Index）** | **主键值** | 可以有多个；查找需回表（拿到主键后再到聚簇索引查完整行） |

> **回表查询**：通过二级索引找到主键值，再回到聚簇索引查完整行数据的过程。回表降低性能，应尽量使用覆盖索引避免。

## 2.2Hash索引

- 底层基于哈希表实现，等值查询 O(1) 极快
- **不支持范围查询**（`>`、`<`、`BETWEEN`）、不支持排序
- 在 Memory 引擎中默认使用，InnoDB 的自适应哈希索引（AHI）自动管理
- 适合场景：等值查询，无排序和范围需求

## 2.3全文索引（Full-Text Index）

- 通过**倒排索引（Inverted Index）**实现文本中的关键词快速查找
- 支持 `MATCH ... AGAINST` 语法
- InnoDB 从 5.6 开始支持
- 适合场景：文章搜索、商品描述关键词搜索（但大量文本搜索推荐用 Elasticsearch）

## 2.4空间索引（R-Tree）

- 用于地理空间数据类型（GEOMETRY、POINT、LINESTRING、POLYGON）
- MyISAM 原生支持，InnoDB 5.7+ 支持
- 适合场景：LBS（附近的人、附近的门店）等地理空间查询

# 3.索引分类

## 3.1按功能分类

| 类型 | 关键字 | 说明 |
|------|--------|------|
| **主键索引** | PRIMARY KEY | 自动创建，唯一且非空，一个表只有一个 |
| **唯一索引** | UNIQUE | 索引列的值必须唯一（允许 NULL），一个表可有多个 |
| **普通索引** | INDEX / KEY | 无唯一性限制，用于加速查询 |
| **全文索引** | FULLTEXT | 用于全文搜索 |
| **前缀索引** | INDEX(col(N)) | 只索引字符串的前 N 个字符 |
| **联合索引** | INDEX(a, b, c) | 多列组合成一个索引 |

## 3.2按存储形式分类（InnoDB）

| 类型 | 叶子节点存储 | 数量限制 |
|------|------------|---------|
| **聚簇索引** | 完整行数据 | 有且仅有一个 |
| **二级索引** | 主键值（需回表） | 可有多个 |

## 3.3按列数量分类

- **单列索引**：只包含一列
- **联合索引（复合索引）**：包含多列，有**最左前缀**特性

# 4.索引语法

```sql
-- 创建索引
CREATE [UNIQUE | FULLTEXT] INDEX index_name ON table_name (col1 [ASC | DESC], col2, ...);
ALTER TABLE table_name ADD INDEX index_name (col1, col2, ...);

-- 查看索引
SHOW INDEX FROM table_name;

-- 删除索引
DROP INDEX index_name ON table_name;
ALTER TABLE table_name DROP INDEX index_name;
```

> **索引命名规范**：`idx_表名_字段名`（普通索引）、`uk_表名_字段名`（唯一索引）、`pk_表名`（主键）。如 `idx_user_email`、`uk_user_phone`。

# 5.SQL性能分析工具

## 5.1查看SQL执行频次

```sql
-- 查看全局 SQL 执行次数（自数据库启动累计）
SHOW GLOBAL STATUS LIKE 'Com_______';  -- 7 个下划线

-- 重点关注
-- Com_select：查询次数
-- Com_insert/Com_update/Com_delete：写入次数
-- 如果 Com_select >> Com_insert，说明是读密集型，索引优化空间大
```

## 5.2慢查询日志

慢查询日志记录执行时间超过阈值的 SQL，是定位性能瓶颈的首要工具。

```sql
-- 查看慢查询开关
SHOW VARIABLES LIKE 'slow_query_log';

-- 查看慢查询阈值（默认 10 秒）
SHOW VARIABLES LIKE 'long_query_time';

-- 临时开启慢查询
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 2;

-- 查看慢查询日志文件位置
SHOW VARIABLES LIKE 'slow_query_log_file';
```

**使用 mysqldumpslow 分析慢查询**：

```bash
# 获取执行次数最多的 10 条 SQL
mysqldumpslow -s c -t 10 /var/lib/mysql/slow.log

# 获取耗时最长的 10 条 SQL
mysqldumpslow -s t -t 10 /var/lib/mysql/slow.log

# 按照总锁时间排序
mysqldumpslow -s l -t 10 /var/lib/mysql/slow.log
```

## 5.3PROFILE分析

SHOW PROFILE 可以分析单条 SQL 在各个阶段的耗时分布。

```sql
-- 查看 profiling 是否开启
SELECT @@profiling;

-- 开启 profiling（会话级）
SET profiling = 1;

-- 执行待分析 SQL
SELECT * FROM user WHERE email = 'test@example.com';

-- 查看 SQL 的 Query ID
SHOW PROFILES;

-- 查看某条 SQL 各阶段的耗时
SHOW PROFILE FOR QUERY 1;

-- 查看 CPU、IO 等详细指标
SHOW PROFILE CPU, BLOCK IO FOR QUERY 1;
```

**耗时阶段关注**：
- `Sending data`：时间最长 = 可能存在全表扫描、大数据量传输
- `Creating tmp table`：排序或分组时创建临时表
- `Sorting result`：排序操作

## 5.4EXPLAIN执行计划

EXPLAIN 是最常用的 SQL 分析工具，展示优化器选择的执行计划。

```sql
EXPLAIN SELECT * FROM user WHERE email = 'test@example.com';
```

### EXPLAIN 核心字段解读

| 字段 | 含义 | 关注要点 |
|------|------|---------|
| **id** | SELECT 标识符 | id 越大越先执行；id 相同从上到下执行 |
| **select_type** | SELECT 类型 | SIMPLE：简单查询；PRIMARY：主查询；SUBQUERY：子查询；DERIVED：派生表（FROM 子查询）；UNION：UNION 中的查询 |
| **table** | 访问的表 | — |
| **partitions** | 匹配的分区 | 分区表时关注 |
| **type** | 访问类型（**最重要！**） | 性能从好到差：NULL > system > const > eq_ref > ref > range > index > ALL |
| **possible_keys** | 可能使用的索引 | 如果有但未使用，说明索引失效 |
| **key** | 实际使用的索引 | NULL = 未使用索引 |
| **key_len** | 使用的索引字节数 | 越短效率越高；联合索引时判断用到哪些列 |
| **ref** | 与索引比较的列/常量 | const 表示常量比较 |
| **rows** | 预估扫描行数 | 越小越好（只是估算，不一定准） |
| **filtered** | 按条件过滤后的百分比 | 越大越好 |
| **Extra** | 额外信息（**非常重要！**） | 见下方 |

### type 访问类型详解

| type | 含义 | 示例 |
|------|------|------|
| **NULL** | 不用访问表和索引 | `SELECT 1`、`SELECT NOW()` |
| **system** | 表只有一行 | 系统表 |
| **const** | 主键或唯一索引等值匹配 | `WHERE id = 1` |
| **eq_ref** | JOIN 时被驱动表用主键/唯一索引匹配 | `JOIN ON b.id = a.b_id` |
| **ref** | 非唯一索引等值匹配 | `WHERE email = 'xxx'` |
| **range** | 索引范围扫描 | `WHERE id > 100`、`BETWEEN`、`IN`、`LIKE 'X%'` |
| **index** | 全索引扫描（遍历整个索引） | 仅扫描索引可获取所需数据 |
| **ALL** | 全表扫描 | **必须避免** |

> `range` 及以上都是可接受的，`index` 和 `ALL` 需要优化（除非预期扫描全表且无可避免）。

### Extra 关键信息

| Extra 值 | 说明 | 应对 |
|-----------|------|------|
| **Using where** | 使用了 WHERE 过滤 | 正常（如果 type=ALL 则需加索引） |
| **Using index** | 覆盖索引（不需回表） | **最佳**，查询的列全在索引中 |
| **Using index condition** | 索引下推（ICP） | 5.6+ 特性，减少回表次数 |
| **Using temporary** | 使用了临时表 | **需要优化**（常见于 GROUP BY、DISTINCT、UNION） |
| **Using filesort** | 额外排序操作 | **需要优化**（未利用索引排序） |
| **Using join buffer** | JOIN 缓冲区（被驱动表无索引） | 为被驱动表 JOIN 字段加索引 |
| **Select tables optimized away** | 优化器直接返回聚合结果 | 最佳（如 `MIN(id)`、`MAX(id)`） |

# 6.索引使用规则

## 6.1最左前缀法则

**联合索引遵循最左前缀法则**：查询必须从索引的最左列开始，且不跳过索引中的列。跳过的列之后的索引会失效。

```sql
-- 创建联合索引
CREATE INDEX idx_user_name_age_status ON user(name, age, status);

-- ✓ 走索引（从 name 开始，连续）
SELECT * FROM user WHERE name = '张三' AND age = 20 AND status = 1;
SELECT * FROM user WHERE name = '张三' AND age = 20;
SELECT * FROM user WHERE name = '张三';                       -- 仅 name 走索引

-- ✗ 不走索引（没有 name）
SELECT * FROM user WHERE age = 20;
SELECT * FROM user WHERE status = 1;

-- ✓ 部分走索引（name 生效，但跳过 age，status 部分不生效）
SELECT * FROM user WHERE name = '张三' AND status = 1;
-- key_len 为 name 字段的长度，说明只有 name 走了索引

-- ✓ 都很走索引（MySQL 优化器会自动调整条件顺序）
SELECT * FROM user WHERE age = 20 AND name = '张三';  -- 优化器调整为先 name 后 age
```

> **联合索引的列顺序设计至关重要**。应将**查询频率高、区分度高**的列放在前面。

## 6.2范围查询

联合索引中，出现范围查询（`>`、`<`、`>=`、`<=`），**范围查询右侧的列索引失效**：

```sql
-- idx_user_name_age_status(name, age, status)
-- 只用到了 name 和 age，status 没用到
SELECT * FROM user WHERE name = '张三' AND age > 20 AND status = 1;

-- 如果使用 >=，部分情况下 status 可能仍然有效（取决于优化器）
SELECT * FROM user WHERE name = '张三' AND age >= 20 AND status = 1;
```

## 6.3索引列运算

```sql
-- ✗ 索引失效（对索引列进行了函数/运算）
SELECT * FROM user WHERE YEAR(create_time) = 2026;
SELECT * FROM user WHERE LEFT(phone, 3) = '138';
SELECT * FROM user WHERE age + 10 = 30;

-- ✓ 优化写法（让条件直接作用在索引列上）
SELECT * FROM user WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01';
SELECT * FROM user WHERE phone LIKE '138%';
SELECT * FROM user WHERE age = 20;
```

## 6.4字符串不加引号

字符串类型字段查询时，不加引号会导致**隐式类型转换**，索引失效：

```sql
-- ✗ phone 是 VARCHAR 类型，未加引号，索引失效
SELECT * FROM user WHERE phone = 13800138000;

-- ✓ 正确
SELECT * FROM user WHERE phone = '13800138000';
```

## 6.5模糊查询

```sql
-- ✓ 尾部模糊：索引不失效（前缀匹配）
SELECT * FROM user WHERE name LIKE '张%';

-- ✗ 头部/中间模糊：索引失效
SELECT * FROM user WHERE name LIKE '%三';
SELECT * FROM user WHERE name LIKE '%三%';
```

> 如果频繁需要 `%keyword%` 搜索，应使用**全文索引**（`MATCH ... AGAINST`）或接入 Elasticsearch。

## 6.6OR连接条件

用 OR 分割的条件，如果 OR 前的列有索引而后面的列没有，则涉及的索引都不会被用到：

```sql
-- id 有索引，age 没有索引 → 索引失效
SELECT * FROM user WHERE id = 1 OR age = 20;

-- 优化：给 age 也加索引，或者改写为 UNION
SELECT * FROM user WHERE id = 1
UNION
SELECT * FROM user WHERE age = 20;
```

## 6.7数据分布影响

MySQL 优化器会根据数据的**统计信息**，评估使用索引和全表扫描的代价。如果评估结果认为全表扫描更快，则不会使用索引：

```sql
-- 如果表中 80% 的数据 status = 1，优化器可能选择全表扫描
SELECT * FROM user WHERE status = 1;
```

> 这是**基于成本的优化器（CBO）**的正常行为。避免在**区分度低**的列上建索引（如性别、状态字段，区分度太低则索引效果差）。

## 6.8SQL提示（Index Hint）

SQL 提示是在语句中加入人为建议，指导优化器选择或忽略索引：

```sql
-- 建议使用指定索引
SELECT * FROM user USE INDEX(idx_name) WHERE name = '张三';

-- 忽略指定索引
SELECT * FROM user IGNORE INDEX(idx_name) WHERE name = '张三';

-- 强制使用指定索引
SELECT * FROM user FORCE INDEX(idx_name) WHERE name = '张三';
```

> **USE vs FORCE**：优化器可能不接受 USE 的建议，但 FORCE 强制使用（除非无法使用）。一般情况下不建议使用 Index Hint，让优化器自行决策；只在性能调优时短期使用。

## 6.9覆盖索引（Covering Index）

覆盖索引指查询所需的所有列都能在索引中找到，**不需要回表**查询。这是索引优化的终极目标之一。

```sql
-- idx_user_name_age(name, age)
-- ✓ 覆盖索引：所需列 name, age 都在索引中
SELECT name, age FROM user WHERE name = '张三';

-- ✗ 需要回表：email 不在索引中
SELECT name, email FROM user WHERE name = '张三';
```

**覆盖索引优势**：
- 减少一次回表的磁盘 IO（聚簇索引查找）
- 二级索引通常比聚簇索引小，更容易缓存到 Buffer Pool
- 查询性能可能提升数倍

> **避免 SELECT \*** 的另一个重要原因：`SELECT *` 几乎必然导致回表（除非聚簇索引），无法利用覆盖索引。

## 6.10前缀索引

当字符串字段很长（如 URL、文章标题），完整索引会占用大量空间。可只取前缀建立索引：

```sql
-- 前 10 个字符建立前缀索引
CREATE INDEX idx_url ON page(url(10));

-- 计算前缀的选择性（不重复比例）
SELECT
  COUNT(DISTINCT url) / COUNT(*) AS full_selectivity,
  COUNT(DISTINCT LEFT(url, 10)) / COUNT(*) AS prefix10_selectivity,
  COUNT(DISTINCT LEFT(url, 20)) / COUNT(*) AS prefix20_selectivity
FROM page;

-- 选择性越接近 1，索引效果越好
```

**前缀索引的局限性**：
- 无法利用前缀索引做 ORDER BY 和 GROUP BY
- 无法做覆盖索引（因为前缀索引不存储完整的列值）

## 6.11索引下推（ICP, Index Condition Pushdown）

MySQL 5.6 引入的优化特性。在联合索引查询时，将部分 WHERE 条件的过滤下推到存储引擎层，减少回表次数。

```sql
-- idx_user_name_age(name, age)
SELECT * FROM user WHERE name LIKE '张%' AND age = 20;

-- 无 ICP：先按 name 前缀匹配查出所有行，回表后再过滤 age
-- 有 ICP：在索引中同时过滤 name 和 age，匹配的行才回表
```

**Extra 中 `Using index condition` 表示 ICP 生效**。

# 7.索引设计原则

## 7.1何时建立索引

- **数据量较大的表**（百万级以上）
- **频繁出现在 WHERE、ORDER BY、GROUP BY 中的列**
- **区分度高（cardinality 高）的列**：如用户 ID、手机号、订单号
- **JOIN 的连接字段**（被驱动表字段需要索引）
- **外键字段**（自动建或手动建索引）

## 7.2何时不建立索引

- **数据量小的表**（几千行以下，全表扫描也很快）
- **频繁更新的列**（索引维护成本高）
- **区分度低的列**：如性别、类型（枚举值很少）
- **不参与查询条件的列**（建了也用不到）
- **TEXT / BLOB 大字段**（可用前缀索引替代）
- **计算列**（如 `WHERE age + 10 > 30`）

## 7.3联合索引设计指南

1. **将查询频率高、区分度高、等值匹配的列放在前面**
2. **范围查询的列尽量放最后**（减少右侧列失效）
3. **尽量用联合索引替代多个单列索引**：节省索引空间 + 支持覆盖索引
4. **考虑查询模式**：如果常查 `(a)` 和 `(a, b)`，建 `(a, b)` 联合索引即可覆盖两种场景
5. **多表 JOIN 尽量让被驱动表的连接键有索引**

# 8.索引失效场景汇总

| 场景 | 解决方案 |
|------|---------|
| 违反最左前缀法则 | 调整联合索引的列顺序，从最左列开始查询 |
| 索引列使用函数/运算 | 将计算移到等号右边 |
| 字符串不加引号（隐式类型转换） | 始终加引号 |
| LIKE 前面加 % | 改为全文索引或搜索引擎 |
| OR 连接非索引列 | 建索引或改写 UNION |
| 不等于（`!=`、`<>`、`NOT IN`） | 尽量用 IN 替代 |
| IS NULL / IS NOT NULL | 索引列不允许 NULL |
| 优化器评估不使用索引 | 使用 FORCE INDEX 或调整统计信息 |

# 9.等价排查SQL

```sql
-- 查看表上的索引
SHOW INDEX FROM table_name;

-- 查看索引的基数（区分度）
SELECT index_name, cardinality
FROM information_schema.statistics
WHERE table_name = 'user';

-- 未使用的索引（MySQL 8.0 可通过 sys schema 查看）
SELECT * FROM sys.schema_unused_indexes;

-- 冗余索引（某个索引是另一个索引的最左前缀）
SELECT * FROM sys.schema_redundant_indexes;

-- 强制更新统计信息
ANALYZE TABLE table_name;
```
