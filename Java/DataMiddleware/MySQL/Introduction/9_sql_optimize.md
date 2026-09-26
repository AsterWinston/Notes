# 1.插入数据优化

## 1.1批量插入

```sql
-- ✗ 逐条插入（网络往返次数多，开销大）
INSERT INTO user (name, email) VALUES ('张三', 'a@a.com');
INSERT INTO user (name, email) VALUES ('李四', 'b@b.com');
INSERT INTO user (name, email) VALUES ('王五', 'c@c.com');

-- ✓ 批量插入（减少网络往返，推荐）
INSERT INTO user (name, email) VALUES
  ('张三', 'a@a.com'),
  ('李四', 'b@b.com'),
  ('王五', 'c@c.com');
```

**建议**：单次批量插入 500~1000 条，不是越多越好（内存和事务大小有限制）。

## 1.2手动提交事务

```sql
-- ✗ 每次 INSERT 自动提交（1000 次 COMMIT = 1000 次 IO）
SET autocommit = 1;
-- 循环 INSERT ...

-- ✓ 手动控制事务（1000 条一个事务一次 COMMIT）
START TRANSACTION;
INSERT INTO user (...) VALUES (...), (...), ...;  -- 1000 条
COMMIT;
```

## 1.3主键顺序插入

InnoDB 按主键顺序组织数据。顺序插入时数据页紧密排列，写入性能最高。随机插入（如 UUID 主键）会导致频繁的页分裂，性能急剧下降。

```sql
-- ✓ 顺序插入（性能好）
INSERT INTO user (id, name) VALUES (1, 'a'), (2, 'b'), (3, 'c');

-- ✗ 乱序插入（性能差，页分裂）
INSERT INTO user (id, name) VALUES (3, 'c'), (1, 'a'), (2, 'b');
```

## 1.4大批量数据使用LOAD DATA

当需要导入百万级数据时，`LOAD DATA INFILE` 比 INSERT 快 20 倍以上：

```sql
-- 开启本地文件加载
SET GLOBAL local_infile = 1;

-- 加载 CSV 文件
LOAD DATA LOCAL INFILE '/tmp/user_data.csv'
INTO TABLE user
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(name, email, age)
SET create_time = NOW();
```

`LOAD DATA` 快的原因：
- 批量加载机制绕过 SQL 解析，直接写入数据页
- 跳过 undo log（对于部分操作），减少日志量
- 内部批量 INSERT 到引擎层

> `LOAD DATA` 在 MySQL 8.0.19+ 支持 `LOAD DATA CONCURRENT`，可并发执行。

# 2.主键优化

## 2.1数据组织方式

InnoDB 表的数据按主键顺序组织存放（索引组织表，IOT）。主键 B+Tree 的叶子节点即为数据行。

## 2.2页分裂与页合并

### 页分裂（Page Split）

当新插入的行在已满的数据页中需要按主键顺序放在中间位置时，InnoDB 会：

1. 创建新页
2. 将原页中约 50% 的数据移到新页
3. 更新两个页之间的双向链表
4. 更新父节点（索引页）中的指针

**页分裂代价**：增加磁盘空间碎片、写入 IO 增加、索引树平衡调整。

### 页合并（Page Merge）

当删除记录后，页中有效数据占比低于 `MERGE_THRESHOLD`（默认 50%，可配置），InnoDB 会尝试合并相邻页以优化空间。

```sql
-- 设置页合并阈值（创建表时）
CREATE TABLE user (
  id BIGINT PRIMARY KEY
) COMMENT 'MERGE_THRESHOLD=40';  -- 40% 时就合并
```

## 2.3主键设计原则

| 原则 | 原因 |
|------|------|
| **尽量降低主键长度** | 主键越小，二级索引的叶子节点越小（存主键值），B+Tree 层数越低 |
| **尽量使用自增主键** | AUTO_INCREMENT 保证顺序插入，避免页分裂 |
| **避免 UUID 或自然主键** | 随机值导致页分裂、占用空间大（36 字符 vs 8 字节） |
| **避免修改主键** | 主键修改需要移动行数据 + 更新所有二级索引 |

> **分布式 ID 策略**：如果必须分布式主键，使用雪花算法（Snowflake）的变体，保证趋势递增，比纯 UUID 效果好得多。

# 3.ORDER BY优化

## 3.1排序方式

MySQL 有两种获取排序数据的方式：

| 方式 | 说明 | Extra 显示 |
|------|------|-----------|
| **Using index** | 利用有序索引直接返回，不需要额外排序 | `Using index` |
| **Using filesort** | 扫描数据后在 sort_buffer 中排序后返回 | `Using filesort` |

> **"filesort" 不一定写磁盘**：如果排序数据量小于 `sort_buffer_size`（默认 256KB），在内存中排序；超过则在磁盘临时文件中排序。

## 3.2优化策略

1. **为 ORDER BY 字段建索引**（联合索引遵循最左前缀）
2. **使用覆盖索引**：让 ORDER BY 列 + SELECT 列都能在索引中找到
3. **注意排序一致性**：多字段排序时，ASC/DESC 方向应与索引排序方向一致

```sql
-- 创建索引
CREATE INDEX idx_user_age_name ON user(age ASC, name ASC);

-- ✓ 索引排序（与索引顺序一致）
SELECT name FROM user ORDER BY age, name;

-- ✗ 文件排序（方向不一致）
SELECT name FROM user ORDER BY age ASC, name DESC;

-- ✓ 方向一致的写法（MySQL 8.0 支持降序索引，index (age asc, name desc)）
CREATE INDEX idx_user_age_name_desc ON user(age ASC, name DESC);
```

## 3.3增大排序缓冲区

如果无法避免 filesort，大数据量时可调整缓冲区：

```sql
-- 查看当前值（默认 256KB）
SHOW VARIABLES LIKE 'sort_buffer_size';

-- 会话级调整（仅对当前连接有效）
SET sort_buffer_size = 4194304;  -- 4MB
```

> 不要全局设置过大——sort_buffer_size 是**每个排序会话**都分配的，过大会导致内存不足。

# 4.GROUP BY优化

GROUP BY 也遵循最左前缀法则，通过索引提高效率：

```sql
CREATE INDEX idx_dept_salary ON emp(dept_id, salary);

-- ✓ 使用索引分组（最左列匹配）
SELECT dept_id, COUNT(*) FROM emp GROUP BY dept_id;

-- ✗ 文件排序 + 临时表（不满足最左前缀）
SELECT salary, COUNT(*) FROM emp GROUP BY salary;
```

**优化技巧**：
- WHERE 条件先过滤再分组（减小分组数据量）
- 避免在 GROUP BY 上使用函数
- 如果无法使用索引分组，增大 `sort_buffer_size` 和 `tmp_table_size`

# 5.LIMIT优化

## 5.1深分页问题

`LIMIT offset, count` 当 offset 很大时，需要扫描前 offset + count 行再丢弃前 offset 行，性能极差：

```sql
-- ✗ 深分页（需扫描 1000010 行，丢弃前 1000000 行）
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;
```

## 5.2优化方案

### 方案一：基于主键的游标分页（最佳）

```sql
-- 记录上一页最后一条的 id，用于下一页
SELECT * FROM user WHERE id > 1000000 ORDER BY id LIMIT 10;
```

> 优点：每次只扫描 10 行，性能恒定。限制：不能跳页，只能上一页/下一页。

### 方案二：子查询 + 覆盖索引

```sql
-- 先通过覆盖索引查出主键，再回表（减少回表的数据量）
SELECT * FROM user u
INNER JOIN (
  SELECT id FROM user ORDER BY id LIMIT 1000000, 10
) t ON u.id = t.id;
```

### 方案三：限制最大页数

在业务上限制用户只能翻前 100 页（如电商商品仅展示前 2000 个），超过提示使用更精确的搜索条件。

# 6.COUNT优化

## 6.1COUNT的几种写法

```sql
SELECT COUNT(*)      FROM user;  -- 统计所有行（包括 NULL）
SELECT COUNT(1)      FROM user;  -- 同上，MySQL 优化为 COUNT(*)
SELECT COUNT(id)     FROM user;  -- 统计 id 不为 NULL 的行
SELECT COUNT(email)  FROM user;  -- 统计 email 不为 NULL 的行
```

**性能排序**：`COUNT(*)` ≈ `COUNT(1)` ≈ `COUNT(主键)` > `COUNT(字段)`

> InnoDB 不维护行数计数器（因为 MVCC，不同事务看到的不同行数），每次 COUNT(*) 都需要索引扫描。MyISAM 虽然有计数器但无事务，不推荐用。

## 6.2大表COUNT优化

- **使用 EXPLAIN 的 rows 估算**（不精确，但对展示"约 xxx 条"足够）
- **维护计数表**：通过触发器或应用代码维护统计值
- **使用 Redis 计数器**：业务层维护计数，定期与数据库校对
- **使用 information_schema 的近似值**：
  ```sql
  SELECT table_rows FROM information_schema.tables
  WHERE table_schema = 'mydb' AND table_name = 'user';
  ```
  （仅 InnoDB 估算值，有 10%~50% 误差）

# 7.UPDATE优化

## 7.1避免行锁升级为表锁

```sql
-- ✗ 根据非索引列更新，行锁升级为表锁
UPDATE user SET status = 0 WHERE name = '张三';

-- ✓ 根据索引列更新，仅锁对应行
UPDATE user SET status = 0 WHERE id = 1;
```

**原理**：InnoDB 的行锁是加在索引上的，如果不通过索引过滤，InnoDB 无法精确锁定目标行，只能锁整张表。

## 7.2大表分批更新

```sql
-- 分批更新，每批 1000 行，避免长事务
UPDATE user SET status = 0 WHERE status = 1 AND id > 0 ORDER BY id LIMIT 1000;
-- 循环执行直到 affected_rows = 0
COMMIT;
```

# 8.表结构优化

## 8.1字段设计

| 原则 | 说明 |
|------|------|
| **选用最小的数据类型** | INT > BIGINT，TINYINT > INT，能用 SMALLINT 不用 INT |
| **NOT NULL 优先** | NULL 额外占 1 字节 + 逻辑复杂 |
| **VARCHAR 长度恰到好处** | VARCHAR(50) 而非 VARCHAR(255) |
| **避免 TEXT/BLOB** | 大字段影响索引，可单独拆表或用外部存储 |
| **DECIMAL 存储金额** | FLOAT/DOUBLE 有精度丢失风险 |

## 8.2范式与反范式

- **OLTP 系统**：遵循 3NF，减少数据冗余，保证一性致
- **OLAP / 报表系统**：适当反范式（冗余字段），以空间换时间
- **实际案例**：订单表冗余 `user_name`、`product_name`，避免每次查询 JOIN 用户表和商品表

## 8.3垂直拆分

| 拆分方式 | 原则 |
|---------|------|
| **常用字段 vs 低频字段** | 将 TEXT/BLOB、低频字段拆分到扩展表 |
| **热点字段集中** | 热数据字段在同一张表，方便缓存 |

# 9.查询缓存相关

MySQL 8.0 已**彻底移除查询缓存（Query Cache）**。其问题在于：

- 高并发下 QC 的全局锁是严重的性能瓶颈
- 表上任何 UPDATE 都会使该表的所有 QC 条目失效
- 收益低：只有完全相同的 SQL（连空格和大小写都一致）才能命中，ORM 生成的 SQL 很难命中

> **替代方案**：使用 Redis 等外部缓存，或应用层二级缓存（MyBatis / Hibernate 缓存）。

# 10.硬件与系统层面优化

- **使用 SSD**：磁盘 IO 是数据库最大瓶颈，SSD 比 HDD 快 10~100 倍
- **增加内存**：更大的 Buffer Pool 缓存更多数据（建议为物理内存的 50%~70%）
- **CPU 核心数**：MySQL 一个连接一线程，多核能提高并发处理能力
- **文件系统选择**：ext4 / XFS（推荐 XFS，对大数据文件处理更优）
- **I/O 调度**：使用 `noop` 或 `deadline` 调度（SSD），而非 `cfq`
- **RAID 策略**：RAID 10 性能和可靠性兼顾
