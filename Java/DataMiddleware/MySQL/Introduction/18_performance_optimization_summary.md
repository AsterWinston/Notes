# 1. 性能优化概览

数据库性能优化是一个系统工程，需要从多个层面递进分析：

```
客户端/连接池 → SQL 语句 → 索引 → 表结构 → MySQL 配置 → 硬件/OS
    ↑                                                          ↑
  最易见效                                                   最昂贵
```

**优化原则**：

1. **先测量后优化**：用慢查询日志/EXPLAIN 找到真正的瓶颈，不要凭感觉
2. **收益递减定律**：从成本低、收益高的点开始（如索引），逐步下探
3. **全局视角**：优化了一个查询可能导致另一个变慢，需持续监控

# 2. 慢SQL排查流程

## 2.1 标准化排查流程

```
第 1 步：确认是否真的慢 → 开启慢查询日志
       ↓
第 2 步：找出 TOP N 慢 SQL → mysqldumpslow / pt-query-digest
       ↓
第 3 步：分析慢的原因 → EXPLAIN + SHOW PROFILE
       ↓
第 4 步：制定优化方案 → 加索引 / 改 JOIN / 拆分 SQL / 改架构
       ↓
第 5 步：实施并验证 → 再次 EXPLAIN + 业务验证
       ↓
第 6 步：持续监控 → 对比优化前后慢查询变化
```

## 2.2 关键检查命令

```sql
-- 1. 当前执行的所有 SQL
SHOW FULL PROCESSLIST;

-- 2. 锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 3. 长事务
SELECT * FROM information_schema.INNODB_TRX
WHERE trx_started < DATE_SUB(NOW(), INTERVAL 60 SECOND);

-- 4. 未使用/冗余索引
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_redundant_indexes;

-- 5. 表统计信息
SELECT
  table_name,
  table_rows,
  ROUND(data_length / 1024 / 1024, 2) AS data_mb,
  ROUND(index_length / 1024 / 1024, 2) AS index_mb
FROM information_schema.TABLES
WHERE table_schema = 'mydb'
ORDER BY (data_length + index_length) DESC;

-- 6. InnoDB 缓冲池命中率
SHOW STATUS LIKE 'innodb_buffer_pool_read%';
-- Buffer Pool 命中率 = (requests - reads) / requests
-- 理想：> 99%

-- 7. 连接状态
SHOW STATUS LIKE 'Threads%';
```

# 3. 常见慢查询原因与解决

## 3.1 全表扫描（type=ALL）

```sql
-- 问题
SELECT * FROM user WHERE email = 'test@example.com';  -- email 无索引

-- 解决：创建索引
CREATE INDEX idx_email ON user(email);
```

## 3.2 索引失效

| 场景 | SQL 示例 | 解决 |
|------|---------|------|
| 隐式类型转换 | `WHERE phone = 13800138000`（phone 是 VARCHAR） | 加引号：`WHERE phone = '13800138000'` |
| 函数操作 | `WHERE YEAR(create_time) = 2026` | 用范围：`WHERE create_time >= '2026-01-01' AND create_time < '2027-01-01'` |
| 前置模糊 | `WHERE name LIKE '%三'` | 全文索引 或 ES |
| 违反最左前缀 | `WHERE age = 20`（联合索引是 name, age） | 调整索引列顺序或统一查询模式 |
| OR 两边非全索引 | `WHERE id = 1 OR email = 'xxx'`（email 无索引） | UNION 改写或建索引 |
| NOT IN / != | `WHERE status != 1` | 改用 `WHERE status IN (0, 2, 3)` |

## 3.3 深分页

```sql
-- 问题
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;

-- 解决：游标分页
SELECT * FROM user WHERE id > 1000000 ORDER BY id LIMIT 10;
```

## 3.4 JOIN无索引

```sql
-- 问题：被驱动表的连接字段无索引
EXPLAIN SELECT * FROM emp e JOIN dept d ON e.dept_id = d.id;
-- 检查 Extra 中是否有 Using join buffer → 说明被驱动表无索引

-- 解决：为被驱动表的 JOIN 字段加索引
CREATE INDEX idx_emp_dept ON emp(dept_id);
```

## 3.5 Using filesort

```sql
-- 问题：排序未利用索引
SELECT * FROM user ORDER BY age;

-- 解决：为排序字段建索引
CREATE INDEX idx_age ON user(age);
```

## 3.6 Using temporary

```sql
-- 问题：GROUP BY 产生临时表
SELECT dept_id, COUNT(*) FROM emp GROUP BY dept_id;

-- 解决：为 GROUP BY 字段建索引
CREATE INDEX idx_dept ON emp(dept_id);
```

## 3.7 SELECT *

```sql
-- 问题：返回不必要的数据
SELECT * FROM user WHERE status = 1;

-- 解决：只取需要的列（利用覆盖索引）
SELECT id, username FROM user WHERE status = 1;  -- 若 idx_status 覆盖
```

## 3.8 大事务

```sql
-- 问题：一次事务处理大量数据
START TRANSACTION;
UPDATE user SET status = 0 WHERE status = 1;  -- 100 万行
COMMIT;

-- 解决：分批处理
UPDATE user SET status = 0 WHERE status = 1 ORDER BY id LIMIT 1000;
-- 循环执行，每批单独 COMMIT
```

# 4. 索引优化检查清单

| 检查项 | 命令/方法 |
|--------|----------|
| 表是否有主键 | `SHOW CREATE TABLE` |
| WHERE 字段是否有索引 | `EXPLAIN` 查看 key 列 |
| JOIN 字段是否有索引 | `EXPLAIN` 查看 Extra 是否有 Using join buffer |
| ORDER BY 是否走索引 | `EXPLAIN` 查看 Extra 是否有 Using filesort |
| GROUP BY 是否走索引 | `EXPLAIN` 查看 Extra 是否有 Using temporary |
| 索引是否被使用 | `sys.schema_unused_indexes` |
| 是否有冗余索引 | `sys.schema_redundant_indexes` |
| 覆盖索引利用 | `EXPLAIN` Extra 是否为 Using index |

# 5. SQL编写规范（性能视角）

```sql
-- ✓ IN 的值不宜过多（通常 < 1000）
SELECT * FROM user WHERE id IN (1, 2, ..., 999);  -- OK
-- ✗ IN 5000 个值 → 考虑 JOIN 临时表

-- ✓ WHERE 中能用 = 不用 !=
WHERE status = 1            -- 可能用索引
WHERE status != 1           -- 不太可能用索引

-- ✓ WHERE 中能用 IN 不用 OR
WHERE id IN (1, 2, 3)       -- 可能用索引
WHERE id = 1 OR id = 2 OR id = 3  -- 可能用不上

-- ✓ JOIN 中用小表驱动大表
-- 优化器通常自动处理，但 LEFT JOIN 需要人工注意

-- ✓ 使用 UNION ALL 代替 UNION（如果不需要去重）
SELECT id FROM t1 UNION ALL SELECT id FROM t2;  -- 更快

-- ✓ 分页用游标而非 OFFSET
WHERE id > last_id LIMIT 10  -- 快
LIMIT 1000000, 10             -- 慢

-- ✓ 批量写入而非逐条 INSERT
INSERT INTO t VALUES (1,a), (2,b), (3,c);  -- 快
```

# 6. 参数调优速查

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `innodb_buffer_pool_size` | 物理内存 50%~70% | 最重要参数 |
| `innodb_flush_log_at_trx_commit` | 1（安全）或 2（性能） | 持久性 vs 性能 |
| `sync_binlog` | 1 | binlog 每次刷新到磁盘 |
| `innodb_log_file_size` | min(1G, 25% × buffer_pool) | redo log 大小 |
| `max_connections` | 500~1000 | 最大连接数 |
| `thread_cache_size` | 50~100 | 线程缓存 |
| `table_open_cache` | 2000~4000 | 表的文件描述符缓存 |
| `tmp_table_size` | 64M~256M | 内存临时表大小 |
| `sort_buffer_size` | 256K~4M（按需） | 排序缓冲区，按连接分配 |
| `join_buffer_size` | 256K~4M（按需） | JOIN 缓冲区 |

# 7. 性能基准测试

```sql
-- 简单测试：全表扫描
SELECT BENCHMARK(10000000, (SELECT COUNT(*) FROM user));

-- sysbench 压力测试（推荐）
-- 准备数据
sysbench oltp_read_write --mysql-host=localhost --mysql-user=root --mysql-password=root123 \
  --mysql-db=mydb --tables=10 --table-size=100000 prepare

-- 运行测试
sysbench oltp_read_write --mysql-host=localhost --mysql-user=root --mysql-password=root123 \
  --mysql-db=mydb --tables=10 --table-size=100000 --threads=16 --time=60 run

-- 清理
sysbench oltp_read_write --mysql-host=localhost --mysql-user=root --mysql-password=root123 \
  --mysql-db=mydb --tables=10 cleanup
```

# 8. 紧急故障处理

| 故障现象 | 应急处理 |
|---------|---------|
| 数据库响应极慢 | `SHOW FULL PROCESSLIST` 找出慢 SQL → `KILL QUERY <id>` |
| 连接数打满 | 临时扩大 `max_connections` + KILL 无效 Sleep 连接 |
| 主从延迟极大 | 降低 `long_query_time` 找出从库慢 SQL → KILL 慢 SQL |
| 磁盘即将满 | 清理 binlog + 慢查询日志 + 无用的表 |
| CPU 100% | 找高频 SQL（`performance_schema` 或 `pt-query-digest`），分析索引 |
| 死锁 | `SHOW ENGINE INNODB STATUS\G` 查看 LATEST DETECTED DEADLOCK → 按日志修改代码的锁顺序 |
| 表损坏（MyISAM） | `REPAIR TABLE xxx;` |
| 误删数据 | 立即停止所有写入 → 从备份 + binlog 恢复 |
