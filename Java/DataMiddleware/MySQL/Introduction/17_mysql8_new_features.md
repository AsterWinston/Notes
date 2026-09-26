# 1.MySQL 8.0 重大架构变更

## 1.1原子DDL（Atomic DDL）

DDL 操作（CREATE/ALTER/DROP TABLE 等）实现原子性——要么成功，要么自动回滚，不会留下不一致的中间状态。

```sql
-- 8.0：即使中途崩溃，不会出现"文件已创建但元数据未记录"等不一致
CREATE TABLE t1 (...) ENGINE=InnoDB;  -- 原子操作

-- 批量 DDL：全部成功或全部回滚
DROP TABLE t1, t2;  -- 原子操作（5.7 可能部分成功）
```

**实现基础**：InnoDB 将数据字典（元数据）存储到事务性表中（`mysql.ibd`），DDL 和 DML 在同一事务语义下。

## 1.2事务性数据字典

MySQL 5.7 及之前的元数据存储在 `.frm`、`.par`、`.trg` 等文件中，容易产生不一致。MySQL 8.0 将所有元数据迁移到 **InnoDB 事务表**中：

| 改进 | 5.7 | 8.0 |
|------|-----|-----|
| 元数据存储 | 文件系统（.frm + .opt） | InnoDB 表（mysql.ibd） |
| 崩溃一致性 | 可能不一致 | 事务保证（原子 DDL） |
| 信息查看 | `information_schema` 查询慢 | 缓存 + 无文件 I/O，更快 |

> **升级影响**：从 5.7 升级到 8.0 时，旧的 `.frm` 文件会被迁移到数据字典表。

## 1.3移除查询缓存（Query Cache）

MySQL 8.0 彻底移除了查询缓存（Query Cache）：

- QC 的全局锁在高并发下是严重瓶颈
- 任何 UPDATE 都会使该表的所有 QC 条目失效
- 实测 QC 对大多数 OLTP 场景**弊大于利**

**替代方案**：Redis 等外部缓存 + 应用层二级缓存。

## 1.4默认字符集改为 utf8mb4

MySQL 8.0 默认字符集从 `latin1` 改为 `utf8mb4`，排序规则为 `utf8mb4_0900_ai_ci`。

# 2.新增核心SQL功能

## 2.1窗口函数（Window Function）

窗口函数是 8.0 最受期待的功能，解决了"既要分组统计又要保留明细行"的痛点。

```sql
-- ROW_NUMBER：排序编号
SELECT name, dept, salary,
  ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
FROM employee;

-- RANK / DENSE_RANK
SELECT name, score,
  RANK() OVER (ORDER BY score DESC) AS `rank`,
  DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM exam;

-- LAG / LEAD：上下行对比
SELECT date, revenue,
  LAG(revenue, 1) OVER (ORDER BY date) AS prev_revenue,
  revenue - LAG(revenue, 1) OVER (ORDER BY date) AS diff
FROM sales;

-- 累计求和
SELECT date, amount,
  SUM(amount) OVER (ORDER BY date) AS cumulative
FROM transactions;

-- NTILE：分桶
SELECT name, salary, NTILE(4) OVER (ORDER BY salary DESC) AS bucket
FROM employee;
```

## 2.2公用表表达式（CTE）

CTE（Common Table Expression）使用 `WITH` 子句定义临时结果集，使复杂查询更易读：

### 普通CTE

```sql
WITH dept_stats AS (
  SELECT dept_id, AVG(salary) AS avg_sal, COUNT(*) AS cnt
  FROM emp GROUP BY dept_id
)
SELECT e.name, e.salary, d.avg_sal
FROM emp e
JOIN dept_stats d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal;
```

### 递归CTE

递归 CTE 是 8.0 的重磅功能，可优雅地处理**树形/层级数据**：

```sql
-- 查询 id=1 的所有子节点（树形结构）
WITH RECURSIVE category_tree AS (
  -- 基础查询：根节点
  SELECT id, name, parent_id, 0 AS level
  FROM category WHERE id = 1

  UNION ALL

  -- 递归查询：子节点
  SELECT c.id, c.name, c.parent_id, ct.level + 1
  FROM category c
  JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY level, id;
```

**递归 CTE 替代**：MySQL 5.7 的树形查询通常需要存储过程循环或多次 SQL 查询，递归 CTE 简化了大量的代码复杂度。

## 2.3降序索引（Descending Index）

MySQL 8.0 真正支持降序索引——索引可以声明 `DESC`，使 `ORDER BY col1 ASC, col2 DESC` 这类查询能利用索引顺序直接返回结果，无需额外排序（filesort）。

```sql
-- 8.0：真正支持降序索引
CREATE INDEX idx_age_salary ON emp(age ASC, salary DESC);

-- 查询直接利用索引排序
SELECT * FROM emp ORDER BY age ASC, salary DESC;
-- Extra: Using index（而非 Using filesort）
```

> MySQL 5.7 语法上支持 `DESC` 但实际被忽略，索引始终以 ASC 存储。

## 2.4不可见索引（Invisible Index）

将索引设为不可见（优化器不会使用），用于评估索引重要性：

```sql
-- 不想删除索引但想测试删除后的性能影响
ALTER TABLE user ALTER INDEX idx_status INVISIBLE;

-- 验证没有该索引时查询是否受影响
-- 如果确认无影响，再安全删除

-- 恢复可见
ALTER TABLE user ALTER INDEX idx_status VISIBLE;

-- 查看不可见索引
SELECT TABLE_NAME, INDEX_NAME FROM information_schema.STATISTICS WHERE is_visible = 'NO';
```

> **使用场景**：生产环境安全删除索引——先设为 INVISIBLE 运行几天，确认性能无退化后再 DROP。

## 2.5函数索引（Functional Index / 表达式索引）

对**表达式的计算结果**建立索引，解决"WHERE 条件中有函数导致索引失效"的问题：

```sql
-- 问题：YEAR() 函数导致索引失效
SELECT * FROM user WHERE YEAR(create_time) = 2026;  -- 全表扫描

-- 8.0 方案：创建函数索引
CREATE INDEX idx_year_create_time ON user ((YEAR(create_time)));

-- 查询自动使用函数索引
SELECT * FROM user WHERE YEAR(create_time) = 2026;  -- 索引扫描

-- 更多示例
CREATE INDEX idx_upper_name ON user ((UPPER(name)));
CREATE INDEX idx_month_salary ON emp ((MONTH(hire_date)), (salary * 12));
```

> 函数索引的表达式必须用**括号括起来** `((expr))`，且表达式必须是确定性的（相同输入始终相同输出）。

## 2.6角色管理（Roles）

MySQL 8.0 引入角色（Role），简化权限管理：

```sql
CREATE ROLE 'app_read', 'app_write', 'app_admin';

GRANT SELECT ON mydb.* TO 'app_read';
GRANT INSERT, UPDATE, DELETE ON mydb.* TO 'app_write';

GRANT 'app_read', 'app_write' TO 'dev_user'@'%';
SET DEFAULT ROLE ALL TO 'dev_user'@'%';
```

## 2.7资源组（Resource Group）

用于给不同的查询分配不同的 CPU 资源：

```sql
CREATE RESOURCE GROUP r1 TYPE=USER VCPU=0-1;
CREATE RESOURCE GROUP r2 TYPE=USER VCPU=2-3;

SET RESOURCE GROUP r1 FOR THREAD 123;
```

## 2.8直方图（Histogram）

8.0 支持创建列值的统计直方图，帮助优化器更准确地估算选择率：

```sql
-- 创建直方图
ANALYZE TABLE user UPDATE HISTOGRAM ON age, status;

-- 查看直方图
SELECT * FROM information_schema.COLUMN_STATISTICS;
```

> 对于数据分布不均匀的列（如 90% 的数据 status=1），直方图能大幅提升执行计划准确性。

## 2.9双密码（Dual Password）

8.0 支持同一用户有两个密码，用于密码平滑轮换（零停机切换）：

```sql
ALTER USER 'app_user'@'%' IDENTIFY BY 'new_password' RETAIN CURRENT PASSWORD;
-- 此时旧密码和新密码都能登录

-- 确认所有应用已更新密码后，清除旧密码
ALTER USER 'app_user'@'%' DISCARD OLD PASSWORD;
```

# 3.性能相关改进

## 3.1InnoDB 改进

| 改进 | 说明 |
|------|------|
| **撤消表空间自动截断** | DROP 大表时更快（后台清理 undo log） |
| **redo log 优化** | 更好的并发写入性能 |
| **缓冲池状态导出** | 重启后可保留缓冲池热数据 |
| **无锁 ALTER TABLE** | 更多 ALTER 操作支持 Instant DDL |

```sql
-- Instant DDL：ALTER TABLE 仅修改元数据，秒级完成
ALTER TABLE user ADD COLUMN phone VARCHAR(20) DEFAULT NULL, ALGORITHM=INSTANT;

-- 查看 DDL 支持的算法
SELECT * FROM information_schema.INNODB_INSTANT_ADD_COLUMN;
```

## 3.2复制增强

| 改进 | 说明 |
|------|------|
| **基于 WRITESET 的并行复制** | 比 LOGICAL_CLOCK 更细粒度的并行 |
| **克隆插件** | 通过 `CLONE INSTANCE` 快速同步从库 |
| **MySQL Group Replication 改进** | 更好的流控、故障检测 |

```sql
-- 克隆插件：快速搭建从库（无需 xtrabackup）
CLONE INSTANCE FROM 'repl'@'192.168.1.100':3306 IDENTIFIED BY 'password';
```

## 3.3优化器改进

- **CBO 改进**：更好的成本估算模型
- **anti-join / semi-join 优化**：对 `NOT IN` / `EXISTS` 更高效的执行方式
- **哈希 JOIN**：8.0.18+ 支持哈希 JOIN（等值 JOIN 快于 Nested Loop）
- **Skip Scan**：8.0.13+ 支持跳跃扫描（联合索引中跳过最左列也能用索引）

```sql
-- Skip Scan 示例
CREATE INDEX idx_status_age ON user(status, age);

-- 传统：status 不在 WHERE 中 → 索引失效
-- 8.0：Skip Scan 可能仍用索引（分别扫描每个 status 值对应的 age >= 20 部分）
SELECT * FROM user WHERE age >= 20;
```

# 4.安全相关改进

| 改进 | 说明 |
|------|------|
| **caching_sha2_password** | 默认认证插件（替代 mysql_native_password） |
| **弃用 mysql_native_password** | 8.4 将移除（8.0 中仍可配置使用） |
| **OpenSSL** | 支持 TLS 1.3 |
| **审计日志增强** | 更细粒度的审计规则 |
| **密码策略增强** | `validate_password` 组件加强密码复杂度 |

# 5.开发相关改进

| 改进 | 说明 |
|------|------|
| **JSON 增强** | `JSON_TABLE()` 将 JSON 数组转为关系表、`->>` 操作符、多值索引 |
| **REGEXP 增强** | 正则表达式支持更多函数（`REGEXP_INSTR`、`REGEXP_REPLACE` 等） |
| **UUID 函数** | `UUID_TO_BIN()` 支持时间重排参数，生成有序 UUID |
| **RANKING 窗口函数** | `RANK`、`DENSE_RANK`、`ROW_NUMBER`、`NTILE` |
| **CHECK 约束** | 8.0.16+ 真正支持 CHECK 约束 |
| **在线修改 `innodb_buffer_pool_size`** | 无需重启 |

# 6.MySQL 8.0 升级建议

| 项目 | 建议 |
|------|------|
| **升级路径** | 5.7 → 8.0 直接升级（5.6 需先升 5.7） |
| **兼容性检查** | 运行 `mysqlcheck -u root -p --check-upgrade` |
| **SQL_MODE 变化** | `ONLY_FULL_GROUP_BY` 更严格，需检查 GROUP BY 语句 |
| **认证插件** | 8.0 默认 `caching_sha2_password`，老旧 JDBC 驱动可能报错 |
| **字符集** | 旧库升级后建议 `ALTER TABLE` 改为 `utf8mb4` |
| **Query Cache** | 移除 QC，确认系统无依赖 |
| **数据字典** | 不再有 `.frm` 文件，备份和迁移方式变化 |

> MySQL 9.0 已发布（2024），8.0 将在 2026 年 4 月结束主要支持（EOL），建议持续关注升级路径。
