
# 1. SQL 支持总览

Mycat2 基于 Apache Calcite 进行 SQL 解析，兼容 MySQL 8.0 大部分语法。

```
SQL 请求
  │
  ▼
Calcite 解析器
  ├── DDL → Mycat2 分析处理（CREATE/ALTER/DROP TABLE）
  ├── DML → 路由到后端 MySQL
  └── 不支持的语法 → 返回错误
```

# 2. 完全支持的 SQL

## 2.1 DML

```sql
-- SELECT
SELECT * FROM users WHERE id = 1;
SELECT id, name FROM users WHERE age > 18 ORDER BY id LIMIT 10;
SELECT COUNT(*), AVG(age) FROM users GROUP BY city;

-- INSERT
INSERT INTO users(id, name, age) VALUES (1, 'Alice', 25);
INSERT INTO users SELECT * FROM tmp_users;  -- 部分支持

-- UPDATE
UPDATE users SET age = 26 WHERE id = 1;

-- DELETE
DELETE FROM users WHERE id = 1;

-- REPLACE
REPLACE INTO users(id, name, age) VALUES (1, 'Alice', 26);
```

## 2.2 聚合函数

| 函数 | 支持 | 跨分片处理 |
|------|------|-----------|
| `COUNT(*)` | ✓ | 各分片求和 |
| `SUM(col)` | ✓ | 各分片求和 |
| `AVG(col)` | ✓ | SUM/COUNT 二次计算 |
| `MAX(col)` | ✓ | 各分片取最大 |
| `MIN(col)` | ✓ | 各分片取最小 |
| `GROUP_CONCAT` | 部分 | 不推荐跨分片使用 |

## 2.3 排序与分页

```sql
-- ORDER BY — 支持（跨分片归并排序）
SELECT * FROM users ORDER BY age DESC LIMIT 10;

-- 跨分片分页的处理方式：
-- 1. 从每个分片取 OFFSET + LIMIT 条
-- 2. 归并排序
-- 3. 截取 OFFSET 到 OFFSET + LIMIT
```

# 3. 有限支持 / 需注意的 SQL

```sql
-- JOIN（需要 ER 表或全局表支持）
SELECT u.name, o.amount
FROM users u JOIN orders o ON u.id = o.user_id;
-- 如果 users 和 orders 不是 ER 表 → 性能差

-- 子查询
SELECT * FROM users WHERE id IN (
    SELECT user_id FROM orders WHERE amount > 100
);
-- 部分子查询支持，复杂的可能报错

-- UNION
SELECT id FROM users WHERE city = 'Beijing'
UNION
SELECT id FROM users WHERE city = 'Shanghai';
-- 支持，每个分片独立执行后归并

-- DISTINCT
SELECT DISTINCT city FROM users;
-- 跨分片去重，性能随分片数下降
```

# 4. 不支持的 SQL

```sql
-- 存储过程
CALL my_procedure(1, 'test');

-- 触发器
CREATE TRIGGER ...

-- 视图
CREATE VIEW ...

-- 用户自定义函数（UDF）
SELECT my_custom_function(id) FROM users;

-- 表级别的 LOCK/UNLOCK
LOCK TABLES users WRITE;

-- LOAD DATA INFILE
LOAD DATA INFILE '/tmp/data.csv' INTO TABLE users;

-- 多表 UPDATE/DELETE（跨分片）
DELETE u, o FROM users u JOIN orders o ON u.id = o.user_id;
```

# 5. Calcite 查询优化

## 5.1 优化过程

```
原始 SQL
  │
  ▼
Calcite SQL Parser → SqlNode（AST）
  │
  ▼
Validator → 校验表名、列名、类型
  │
  ▼
Optimizer (RBO + CBO)
  ├── 谓词下推（Filter Pushdown）
  ├── 列裁剪（Project Prune）
  ├── 常量折叠
  └── JOIN 重排序
  │
  ▼
生成执行计划 → RelNode
```

## 5.2 谓词下推

```sql
-- 原始 SQL
SELECT * FROM orders WHERE id = 1 AND amount > 100;

-- 优化后（WHERE 条件下推到 MySQL）
-- 发送到后端的 SQL:
SELECT * FROM orders WHERE id = 1 AND amount > 100;
```

## 5.3 列裁剪

```sql
-- 原始 SQL
SELECT id, name FROM users WHERE age > 18;

-- 优化后：只查询需要的列（减少网络传输）
SELECT id, name, age FROM users WHERE age > 18;
-- Mycat2 合并结果时丢弃 age 列
```

# 6. 执行计划查看

```sql
-- 查看 SQL 的执行计划
EXPLAIN SELECT * FROM users WHERE id = 1;

-- 返回示例：
-- +-------+--------+-----------------+
-- | node  | type   | sql             |
-- +-------+--------+-----------------+
-- | dn0   | SELECT | SELECT * FROM   |
-- |       |        | users WHERE id=1|
-- +-------+--------+-----------------+
```

# 7. 跨分片聚合性能指标

```
测试环境：4 分片，单表 100w 行

| SQL 类型                 | 单分片 | 跨 4 分片 | 说明       |
|-------------------------|--------|----------|-----------|
| 精确查询 WHERE id = 1   | 2ms    | 2ms      | 无影响     |
| COUNT(*)               | 5ms    | 15ms     | 归并求和   |
| ORDER BY LIMIT 10      | 8ms    | 35ms     | 归并排序   |
| GROUP BY + AVG         | 20ms   | 80ms     | 二次聚合   |
| DISTINCT               | 15ms   | 60ms     | 归并去重   |
```

# 8. SQL 兼容性最佳实践

1. **WHERE 条件始终包含分片键**：避免全分片扫描
2. **LIMIT 不宜过大**：跨分片取数 = 分片数 × LIMIT
3. **避免跨分片 DISTINCT**：性能随分片数线性下降
4. **复杂 JOIN 考虑 ER 表或全局表**：简化分片路由
5. **大事务拆小**：减少 XA 事务持锁时间
