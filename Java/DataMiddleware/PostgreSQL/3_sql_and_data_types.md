# 1. SQL 基础与 psql

PostgreSQL 的 SQL 方言高度贴近标准，很多在 MySQL 里需要绕路的写法，在 PG 里可以直接表达。但它的**类型系统更严格**：不会悄悄把字符串转成数字，也不允许 `GROUP BY` 漏掉非聚合列。这种严格一开始让人不适应，长期看却能挡掉大量隐性 bug。

## 1.1 psql 常用元命令

`psql` 是官方命令行客户端，元命令以反斜杠开头，不经过 SQL 解析器。

| 命令 | 作用 |
|------|------|
| `\l` / `\c appdb` | 列出数据库 / 切换数据库 |
| `\dn` / `\dt` | 列出 schema / 当前 schema 的表 |
| `\d orders` / `\d+ orders` | 查看表结构、索引、约束 / 额外显示大小 |
| `\x` / `\timing` | 纵向显示 / 显示每条 SQL 的耗时 |

## 1.2 最小可运行示例

```sql
-- 建表：identity 主键、精确金额、数组与 JSONB
CREATE TABLE orders (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_no   varchar(32) NOT NULL UNIQUE,
    amount     numeric(12,2) NOT NULL CHECK (amount >= 0),
    tags       text[] DEFAULT '{}',
    detail     jsonb DEFAULT '{}'::jsonb,
    created_at timestamptz NOT NULL DEFAULT now()
);

-- 插入并返回主键；随后按数组与 JSONB 条件查询
INSERT INTO orders (order_no, amount, tags)
VALUES ('SO20261106001', 199.00, ARRAY['vip', 'promo'])
RETURNING id, created_at;

SELECT order_no, tags[1] AS first_tag, detail ->> 'channel' AS channel
FROM orders
WHERE tags @> ARRAY['vip']
  AND created_at >= now() - interval '7 days';
```

`RETURNING` 是 PG 很实用的特性：`INSERT/UPDATE/DELETE` 都能直接返回受影响的行，省掉一次额外查询，也避免"插入后按唯一键再查一遍"的竞态。SQL 分类上，DDL 涵盖 `CREATE/ALTER/DROP`，DML 涵盖增删改，DQL 是 `SELECT`，DCL 管权限，TCL 管事务；PG 的 DDL 可以放进事务并回滚，这是它与 MySQL 的重要差异。

# 2. schema 与 search_path

## 2.1 schema 是什么

在 PG 里，**数据库和模式（schema）是两层不同的命名空间**：一个实例可以有多个数据库，一个数据库可以有多个 schema，schema 下才是表、视图、函数。完整限定名是 `database.schema.table`，但 SQL 通常只能跨 schema，不能跨数据库查询（跨库需要 FDW 或 `dblink`）。

```sql
CREATE SCHEMA reporting;
CREATE TABLE reporting.daily_sales (stat_date date PRIMARY KEY, amount numeric(14,2) NOT NULL);
```

## 2.2 search_path 的解析顺序

当 SQL 写 `SELECT * FROM orders` 而不带 schema 前缀时，PG 按 `search_path` 从左到右查找：

```sql
SHOW search_path;                                   -- 默认 "$user", public
SET search_path TO reporting, public;
ALTER DATABASE appdb SET search_path TO app, public;
```

默认值里的 `"$user"` 表示"与当前用户名同名的 schema"。如果存在 `app_rw` 这个 schema，它会优先于 `public` 被搜索。

```text
search_path = app, public

SELECT * FROM orders
        │
        ├─► app.orders 存在？ ── 是 ──► 使用它
        │
        └─► public.orders 存在？ ── 是 ──► 使用它
                                  └─ 否 ──► ERROR: relation does not exist
```

## 2.3 常见坑

- **同名表遮蔽**：`app.orders` 与 `public.orders` 同时存在时，`search_path` 决定用哪个，迁移期间极易写错数据。
- **函数解析也受影响**：自定义函数与内置函数同名时，可能调用到非预期版本，带来安全风险。
- **`CREATE TABLE` 落在第一个可写 schema**：不写前缀时，表不一定建在 `public`。

> **最佳实践**：生产 SQL 显式写 schema 前缀；`search_path` 只保留必要项；创建函数时用 `SET search_path = pg_catalog, pg_temp` 固化解析环境。

# 3. 数据类型

## 3.1 数值类型

| 类型 | 存储 | 范围/精度 | 适用 |
|------|------|----------|------|
| `integer` | 4 字节 | 约 ±21 亿 | 常规计数、ID |
| `bigint` | 8 字节 | 约 ±9.2e18 | 雪花 ID、大表主键 |
| `numeric(p,s)` | 变长 | 精确十进制 | **金额、利率** |
| `real` / `double precision` | 4/8 字节 | 约 6/15 位有效数字 | 科学计算、坐标 |

PG **没有 `UNSIGNED` 整数**，需要非负约束时用 `CHECK (col >= 0)`。金额不要用 `float`，二进制浮点无法精确表示 0.1，累加后会出现分位误差；`numeric` 是十进制精确类型，代价是运算比整数慢。建表时直接写 `amount numeric(14,2) NOT NULL` 即可。

## 3.2 文本类型

| 类型 | 行为 | 说明 |
|------|------|------|
| `text` | 变长，无长度上限 | **推荐默认选择** |
| `varchar(n)` | 变长，超过 n 报错 | 长度是业务约束，不是性能优化 |

关键认知：**在 PG 中 `text`、`varchar(n)`、`char(n)` 的存储与查询性能没有本质差异**。`varchar(n)` 只是一个长度约束，不要为了"性能"把 `text` 改短。

## 3.3 时间与日期

| 类型 | 含义 | 是否带时区 |
|------|------|-----------|
| `date` / `time` | 日期 / 一天内的时间 | — |
| `timestamp` | 日期 + 时间 | 不带时区，存"墙上时钟" |
| `timestamptz` | 日期 + 时间 | 内部存 UTC，按会话时区显示 |
| `interval` | 时间间隔 | 如 `'1 day 2 hours'` |

**绝大多数业务时间字段应该用 `timestamptz`**，它存绝对时间点，跨时区、跨夏令时都不会错。`timestamp` 适合"营业时间 09:00"这类与地点绑定的墙上时间。

```sql
SET TIME ZONE 'Asia/Shanghai';
SELECT '2026-11-06 12:00:00+00'::timestamptz;   -- 2026-11-06 20:00:00+08
SELECT (created_at AT TIME ZONE 'Asia/Shanghai')::date FROM orders;
```

> **注意**：`now()` 返回 `timestamptz`，`current_date` 依赖会话时区。应用服务器与数据库时区不一致时，"今天"的边界会差 8 小时，统计报表最容易在这里出错。时区转换只在一端做。

## 3.4 布尔与数组

`boolean` 只接受 `true/false`、`t/f`、`yes/no`、`1/0`（以及 `NULL`），布尔字段直接写 `enabled boolean NOT NULL DEFAULT false`，不要用 `char(1)` 存 `Y/N`。

```sql
-- 数组适合"一行内少量、整体读写"的多值属性，下标从 1 开始
CREATE TABLE article (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tags text[] NOT NULL DEFAULT '{}'
);

INSERT INTO article (tags) VALUES (ARRAY['pg', 'database']);
SELECT * FROM article WHERE tags @> ARRAY['pg'];              -- 包含
SELECT * FROM article WHERE tags && ARRAY['pg', 'mysql'];     -- 重叠
SELECT id, unnest(tags) AS tag FROM article;                  -- 展开
```

如果多值需要独立更新、独立查询、独立建索引组合，应拆成关联表；数组更适合整体读写的标签集合。

## 3.5 范围类型

范围类型表示连续区间，天然支持重叠、包含判断，适合时间段、价格区间、IP 段。

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- 同一会议室的时间段不允许重叠
CREATE TABLE booking (
    id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id int NOT NULL,
    period  tstzrange NOT NULL,
    EXCLUDE USING gist (room_id WITH =, period WITH &&)
);

-- 插入重叠区间会直接报错，由数据库保证互斥
INSERT INTO booking (room_id, period) VALUES (1, '[2026-11-06 09:00+08, 11:00+08)');
```

`[` 表示包含边界，`)` 表示不包含。用 `EXCLUDE` 把"业务互斥"交给数据库，比应用层先查后插更可靠——后者在并发下存在竞态。

# 4. 序列与 identity

## 4.1 用 identity 替代 serial

早期 PG 用 `serial` 模拟自增，它本质是语法糖：隐式创建序列，把 `nextval()` 设为默认值。问题是**序列归属关系"尽力而为"**，权限、复制、迁移时容易漏掉。SQL 标准写法是 `GENERATED ... AS IDENTITY`，PG 10 起支持：

```sql
CREATE TABLE new_way (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL
);

-- ALWAYS 模式禁止显式插入；迁移时用 OVERRIDING SYSTEM VALUE 保留原 ID
INSERT INTO new_way (id, name) VALUES (1, 'a');              -- 错误
INSERT INTO new_way (id, name) OVERRIDING SYSTEM VALUE VALUES (1, 'a');

CREATE TABLE new_way2 (
    id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY   -- 允许显式插入
);
```

**新项目一律用 identity，不要用 serial**。它把序列归属写进系统目录，`pg_dump`、权限、`ALTER TABLE` 的行为都更可靠。

## 4.2 跳号与重置

序列的 `nextval()` **不受事务控制**：事务回滚后已取走的号不会还回去；实例崩溃也会丢失缓存中的号。因此自增 ID **必然存在空洞**，不能当"连续编号"使用。

```sql
-- 迁移后重置序列到当前最大 ID，避免主键冲突
SELECT setval(pg_get_serial_sequence('new_way', 'id'),
              COALESCE((SELECT max(id) FROM new_way), 1), true);
```

> **行业实践**：订单号、流水号这类需要"看起来连续"的编号，不要直接用主键。用独立的业务编号生成器（日期 + 分片 + 序列），主键只作技术标识。

# 5. NULL 语义

NULL 表示"未知"，不是 0、不是空字符串。PG 严格遵循三值逻辑：`TRUE`、`FALSE`、`UNKNOWN`。

## 5.1 三值逻辑与排序

```sql
SELECT 1 = NULL;                   -- NULL
SELECT NULL IS NULL;               -- true
SELECT 1 IS DISTINCT FROM NULL;    -- true

-- PG 默认：ASC 时 NULL 排最后，DESC 时 NULL 排最前
SELECT * FROM orders ORDER BY amount ASC NULLS FIRST;
```

`WHERE` 只保留结果为 `TRUE` 的行，`UNKNOWN` 会被过滤。这就是 `WHERE col <> 'x'` 会漏掉 `col IS NULL` 的原因。

## 5.2 约束与索引中的 NULL

| 场景 | 行为 |
|------|------|
| `NOT NULL` | 禁止 NULL |
| `UNIQUE` | **允许多行 NULL**（NULL 彼此不相等） |
| `PRIMARY KEY` | 隐含 `NOT NULL` + `UNIQUE` |
| `CHECK` | 结果为 `UNKNOWN` 时视为通过 |
| B-tree 索引 | 默认存储 NULL，`IS NULL` 可以走索引 |
| 聚合 | `sum/avg/max/min` 忽略 NULL；`count(col)` 也忽略 |

```sql
CREATE TABLE t_unique (email text UNIQUE);
INSERT INTO t_unique VALUES (NULL), (NULL);   -- 允许

-- PG 15+：把 NULL 视为相等，只允许一行 NULL
CREATE TABLE t_unique2 (email text UNIQUE NULLS NOT DISTINCT);
```

## 5.3 处理建议

- **能 `NOT NULL` 就 `NOT NULL`**，把"缺失"建模成显式状态。
- **`NOT IN` 遇到 NULL 会整体失效**：`x NOT IN (1, 2, NULL)` 永远不为真，改用 `NOT EXISTS`。
- **`COALESCE` 只做展示层兜底**，不要用它掩盖数据质量问题。

```sql
-- 错误示范：子查询含 NULL 时 NOT IN 返回空结果
SELECT * FROM orders WHERE customer_id NOT IN (SELECT customer_id FROM blacklist);

-- 正确做法：语义清晰且不受 NULL 影响
SELECT o.* FROM orders o WHERE NOT EXISTS (
    SELECT 1 FROM blacklist b WHERE b.customer_id = o.customer_id
);
```

# 6. 常见错误与排错

| 报错/现象 | 原因 | 处理 |
|-----------|------|------|
| `column must appear in the GROUP BY clause` | 非聚合列未全部出现在 `GROUP BY` | 补全列，或用 `max()/min()` 明确取值 |
| `operator does not exist: integer = text` | 类型不匹配，PG 不做隐式转换 | 显式 `CAST` 或修正参数类型 |
| `relation "xxx" does not exist` | `search_path` 不对或表在别的 schema | `\dt *.*` 查找，补 schema 前缀 |
| `duplicate key value violates unique constraint` | 序列未同步、并发插入、业务重复 | 重置序列，检查唯一键 |
| 查询结果"少了几行" | 条件含 NULL，被三值逻辑过滤 | 用 `IS NULL` 或 `coalesce` 修正 |

```sql
SELECT pg_typeof(now()), pg_typeof(1), pg_typeof('1'::int);
```

# 7. 练习与自检

**练习一**：建 `event_log` 表，包含 `id`（identity）、`event_time`（timestamptz）、`payload`（jsonb）、`tags`（text[]）。插入三条数据，分别用 `->>`、`@>`、`unnest` 查询。

**练习二**：创建 `coupon` 表，要求同一用户同一时间段内只能有一张有效券（提示：`tstzrange` 加 `EXCLUDE`）。

**练习三**：解释两条 SQL 的结果为什么不同，并给出修正写法：

```sql
SELECT count(*) FROM orders WHERE amount <> 100;
SELECT count(*) FROM orders WHERE amount IS DISTINCT FROM 100;
```

**自检清单**：

- 能说清 `timestamp` 与 `timestamptz` 的存储差异，知道业务该选哪个；
- 能解释为什么 `varchar(n)` 不是性能优化；
- 能画出 `search_path` 的解析顺序，知道同名表遮蔽的风险；
- 能解释序列为什么必然跳号，迁移后如何重置；
- 遇到"查询少行"时，第一反应是检查 NULL 而不是怀疑索引。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| psql | 元命令是日常运维的效率入口，`\d`、`\x`、`\timing` 最常用 |
| schema | 数据库下的命名空间，`search_path` 决定无限定名的解析顺序 |
| 数值 | 金额用 `numeric`，PG 无 `UNSIGNED`，非负靠 `CHECK` |
| 文本 | `text` 是默认选择，长度约束不等于性能优化 |
| 时间 | 业务时间优先 `timestamptz`，时区转换只在一端做 |
| 复杂类型 | 数组适合整体读写的多值，范围类型可用 `EXCLUDE` 表达互斥 |
| 自增 | 新项目用 identity，序列必然跳号，不能当业务编号 |
| NULL | 三值逻辑，`NOT IN` 有陷阱，应尽量加 `NOT NULL` |

表结构和类型定好之后，数据就会开始被并发读写。下一篇进入 **MVCC 与事务**，解释 PG 如何用行版本和快照实现隔离，以及为什么 UPDATE 会制造"膨胀"、VACUUM 为什么是 PG 运维的核心议题。