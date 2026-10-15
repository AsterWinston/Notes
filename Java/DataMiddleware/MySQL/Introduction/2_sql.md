# 1. SQL概述

SQL（Structured Query Language，结构化查询语言）是操作关系型数据库的统一标准语言。虽然不同数据库厂商有自己的扩展（如 MySQL 的 LIMIT、Oracle 的 ROWNUM），但 SQL 的核心语法是通用的。

SQL 按功能分为五大类：

| 分类 | 全称 | 典型语句 |
|------|------|---------|
| DDL | Data Definition Language | CREATE、ALTER、DROP、TRUNCATE |
| DML | Data Manipulation Language | INSERT、UPDATE、DELETE |
| DQL | Data Query Language | SELECT |
| DCL | Data Control Language | GRANT、REVOKE |
| TCL | Transaction Control Language | COMMIT、ROLLBACK、SAVEPOINT |

# 2. DDL（数据定义语言）

DDL 用于定义和修改数据库对象（库、表、索引、视图等）的**结构**，不操作数据本身。

## 2.1 数据库操作

```sql
-- 创建数据库（使用推荐字符集）
CREATE DATABASE IF NOT EXISTS mydb
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_0900_ai_ci;

-- 查看所有数据库
SHOW DATABASES;

-- 查看建库语句
SHOW CREATE DATABASE mydb;

-- 切换数据库
USE mydb;

-- 修改数据库字符集
ALTER DATABASE mydb CHARACTER SET utf8mb4;

-- 删除数据库（生产环境慎用！）
DROP DATABASE IF EXISTS mydb;
```

> **设计原则**：数据库名称应简洁有意义，使用小写+下划线命名，与项目名保持一致。建议一个微服务对应一个数据库（Database per Service）。

## 2.2 表操作

### 2.2.1 数据类型概览

| 分类 | 类型 | 说明 |
|------|------|------|
| 整数 | TINYINT / SMALLINT / INT / BIGINT | 1/2/4/8 字节，有符号/无符号 |
| 浮点数 | FLOAT / DOUBLE | 单/双精度浮点，不精确 |
| 定点数 | DECIMAL(M, D) | 精确小数，财务计算推荐 |
| 字符串 | CHAR(N) / VARCHAR(N) | 定长/变长字符串 |
| 大文本 | TEXT / MEDIUMTEXT / LONGTEXT | 长文本存储 |
| 二进制 | BLOB / MEDIUMBLOB / LONGBLOB | 二进制数据（文件、图片） |
| 日期时间 | DATE / TIME / DATETIME / TIMESTAMP | 日期和时间 |
| JSON | JSON | 原生 JSON 存储（5.7.8+） |
| 枚举 | ENUM | 单选枚举值 |

> **CHAR vs VARCHAR**：
> - CHAR(N) 是**定长**，始终占用 N 个字符的空间，适合长度固定的字段（如身份证号、手机号、MD5 值）。读写效率高，无碎片。
> - VARCHAR(N) 是**变长**，实际占用 = 数据长度 + 1~2 字节（长度前缀），适合长度可变的字段（如姓名、邮箱）。更省空间，但 UPDATE 可能导致页分裂。
> - MySQL 5.0.3 之后，N 表示**字符数**而非字节数（VARCHAR(255) 在 utf8mb4 下可存 255 个字符，最多占 255×4 = 1020 字节）。

### 2.2.2 创建表

```sql
CREATE TABLE IF NOT EXISTS user (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  username VARCHAR(50) NOT NULL COMMENT '用户名',
  password VARCHAR(255) NOT NULL COMMENT '密码（加密存储）',
  email VARCHAR(100) DEFAULT NULL COMMENT '邮箱',
  age TINYINT UNSIGNED DEFAULT 0 COMMENT '年龄',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：0禁用，1启用',
  balance DECIMAL(10, 2) NOT NULL DEFAULT 0.00 COMMENT '余额',
  create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (id),
  UNIQUE KEY uk_username (username),
  KEY idx_status (status),
  KEY idx_create_time (create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='用户表';
```

**字段设计要点**：
- **主键**推荐使用 `BIGINT UNSIGNED AUTO_INCREMENT`，性能好、递增有序、空间适中（UUID 是它的 4 倍且随机性导致页分裂）
- **字符串字段**长度要**刚好够用**，不要无脑 VARCHAR(255)，避免索引过宽
- **敏感信息**（密码）必须加密存储，不可明文
- **默认值**尽量提供，避免插入时遗漏
- **NOT NULL** 优先，NULL 会额外占 1 字节标记位且索引和比较逻辑更复杂
- **create_time / update_time** 是标准审计字段，几乎每张业务表都应包含

### 2.2.3 修改表

```sql
-- 添加字段
ALTER TABLE user ADD COLUMN phone VARCHAR(20) DEFAULT NULL COMMENT '手机号' AFTER email;

-- 修改字段类型（注：可能锁表，大表需用 pt-online-schema-change 等工具）
ALTER TABLE user MODIFY COLUMN age SMALLINT UNSIGNED DEFAULT 0;

-- 重命名字段
ALTER TABLE user CHANGE COLUMN phone mobile VARCHAR(20) DEFAULT NULL COMMENT '手机号';

-- 删除字段
ALTER TABLE user DROP COLUMN mobile;

-- 重命名表
ALTER TABLE user RENAME TO user_info;
RENAME TABLE user_info TO user;
```

> **DDL 的代价**：MySQL 5.6 之前，ALTER TABLE 会锁表并复制整张表数据（耗时 + 不可写）。5.6 引入了 **Online DDL**，部分操作可在不锁表的情况下完成。8.0 进一步支持**原子 DDL**和**瞬时 DDL**（如 `ALTER TABLE ... ADD COLUMN` 只改元数据，不重建表）。

### 2.2.4 删除表

```sql
-- 删除表
DROP TABLE IF EXISTS user;

-- 清空表数据（保留表结构），比 DELETE 快（不记录每行日志）
TRUNCATE TABLE user;
```

> **DELETE vs TRUNCATE**：
> - DELETE 是 DML，逐行删除，记录在 binlog（ROW 格式）、undo log 中，可回滚，可带 WHERE 条件，触发器会被触发
> - TRUNCATE 是 DDL（原子 DDL），直接删除并重建表（或释放数据页），速度快，不可回滚（8.0 原子 DDL 下特殊），auto_increment 重置
> - 清空千万行数据用 TRUNCATE，秒级完成；DELETE 可能需要数小时

# 3. DML（数据操作语言）

DML 用于对表中的**数据**进行增删改操作。

## 3.1 INSERT——插入数据

```sql
-- 插入单行（指定字段）
INSERT INTO user (username, password, email, age)
VALUES ('zhangsan', 'encrypted_pwd', 'zhangsan@example.com', 25);

-- 插入多行（批量插入，减少网络往返）
INSERT INTO user (username, password, email, age) VALUES
  ('lisi', 'encrypted_pwd', 'lisi@example.com', 30),
  ('wangwu', 'encrypted_pwd', 'wangwu@example.com', 28);

-- INSERT ... SELECT（从其他表导入）
INSERT INTO user_backup (id, username, create_time)
SELECT id, username, create_time FROM user WHERE status = 1;

-- 插入或更新（唯一键冲突时更新其他列）
INSERT INTO user (id, username, password) VALUES (1, 'zhangsan', 'new_pwd')
ON DUPLICATE KEY UPDATE password = VALUES(password);

-- 替换插入（唯一键冲突时先删除再插入）
REPLACE INTO user (id, username, password) VALUES (1, 'zhangsan', 'new_pwd');
```

> **批量插入优化**：
> - 单次 INSERT 插入多条记录，减少客户端与服务器之间的网络往返次数
> - 手动控制事务（`START TRANSACTION; ... COMMIT;`），避免每条 INSERT 自动提交
> - 主键顺序插入（有序写入减少页分裂）
> - 大批量数据可使用 `LOAD DATA INFILE`，比 INSERT 快 20 倍

## 3.2 UPDATE——更新数据

```sql
-- 基本更新
UPDATE user SET age = 26, email = 'new_email@example.com'
WHERE id = 1;

-- 多表关联更新
UPDATE user u
INNER JOIN order o ON u.id = o.user_id
SET u.balance = u.balance - o.amount
WHERE o.status = 'pending' AND o.id = 100;

-- 使用 LIMIT 控制更新行数
UPDATE user SET status = 0 WHERE status = 1 LIMIT 1000;
```

> **安全红线**：
> - **UPDATE / DELETE 必须带 WHERE 条件**，否则整表更新/删除
> - 生产环境建议开启 `sql_safe_updates` 模式：`SET SQL_SAFE_UPDATES = 1;` ——当没有 WHERE 或 LIMIT 条件时，UPDATE 和 DELETE 会拒绝执行
> - 大表批量更新应分批进行（`LIMIT 1000` + 循环），避免长事务锁表

## 3.3 DELETE——删除数据

```sql
-- 基本删除
DELETE FROM user WHERE id = 1;

-- 多表关联删除
DELETE u, o FROM user u
INNER JOIN order o ON u.id = o.user_id
WHERE u.status = 0;

-- 分批删除大表数据
DELETE FROM user_log WHERE create_time < '2024-01-01' LIMIT 10000;
```

> **软删除 vs 硬删除**：
> - **软删除**（推荐）：添加 `is_deleted` 字段，删除时 `UPDATE ... SET is_deleted = 1`，数据仍在，可恢复
> - **硬删除**：物理删除，数据不可恢复，但节省空间
> - 业务系统通常使用软删除，日志/流水类数据用硬删除

# 4. DQL（数据查询语言）

## 4.1 SELECT完整语法

```sql
SELECT [DISTINCT | ALL] select_expr [, select_expr ...]
FROM table_references
[WHERE where_condition]
[GROUP BY {col_name | expr | position}]
[HAVING where_condition]
[ORDER BY {col_name | expr | position} [ASC | DESC]]
[LIMIT {[offset,] row_count | row_count OFFSET offset}]
```

**SQL 解析和执行顺序**（非常关键！）：

```
FROM  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  ORDER BY  →  LIMIT
```

这意味着：
- WHERE 中不能使用 SELECT 定义的**别名**（因为 WHERE 在 SELECT 之前执行），但 ORDER BY 可以
- 聚合函数只能在 HAVING 中使用，不能在 WHERE 中使用（WHERE 执行时还没分组）

## 4.2 基本查询

```sql
-- 查询所有字段（生产环境避免 SELECT *）
SELECT * FROM user;

-- 查询指定字段
SELECT id, username, create_time FROM user;

-- 字段别名
SELECT id AS '用户ID', username AS '用户名' FROM user;

-- 去重
SELECT DISTINCT status FROM user;

-- 常量列
SELECT id, username, 'VIP用户' AS user_type FROM user;
```

## 4.3 WHERE条件过滤

### 4.3.1 比较运算符

```sql
-- 等于 / 不等于
SELECT * FROM user WHERE age = 25;
SELECT * FROM user WHERE age <> 25;        -- 推荐（SQL 标准）
SELECT * FROM user WHERE age != 25;        -- 非标准但 MySQL 支持

-- 范围查询
SELECT * FROM user WHERE age BETWEEN 20 AND 30;
SELECT * FROM user WHERE id IN (1, 2, 3, 5, 8);

-- NULL 判断（不能用 = 或 !=）
SELECT * FROM user WHERE email IS NULL;
SELECT * FROM user WHERE email IS NOT NULL;
```

> **NULL 陷阱**：NULL 表示"未知"，任何值与 NULL 比较结果都是 NULL（未知），而非 TRUE/FALSE。`NULL = NULL` 返回 NULL，`NULL <> NULL` 也返回 NULL。因此必须使用 `IS NULL` 和 `IS NOT NULL`。ORDER BY 中 NULL 默认排在最前（ASC）或最后（DESC）。

### 4.3.2 逻辑运算符

```sql
-- AND（优先级高于 OR）
SELECT * FROM user WHERE age > 20 AND status = 1;

-- OR
SELECT * FROM user WHERE age < 18 OR age > 60;

-- NOT
SELECT * FROM user WHERE NOT status = 1;
```

> **优先级陷阱**：`WHERE a = 1 OR b = 2 AND c = 3` 等价于 `a = 1 OR (b = 2 AND c = 3)`。为避免歧义，**始终使用括号**明确逻辑。

### 4.3.3 LIKE模糊查询

```sql
-- % 匹配任意字符序列（包括空字符）
SELECT * FROM user WHERE username LIKE '张%';     -- 以"张"开头
SELECT * FROM user WHERE username LIKE '%三';     -- 以"三"结尾
SELECT * FROM user WHERE username LIKE '%三%';    -- 包含"三"

-- _ 匹配任意单个字符
SELECT * FROM user WHERE username LIKE '张_';      -- "张" + 一个任意字符

-- 转义：如果要匹配 % 或 _ 本身
SELECT * FROM user WHERE username LIKE '%\%%';    -- 包含 % 字符
SELECT * FROM user WHERE username LIKE '%\_%' ESCAPE '\';  -- 同上
```

> **LIKE 与索引**：`LIKE '张%'`（前缀匹配）可能使用索引；`LIKE '%三'` 或 `LIKE '%三%'`（中缀/后缀匹配）通常无法使用 B+Tree 索引，会退化为全表扫描。大量模糊搜索场景应使用**全文索引（FULLTEXT）**或搜索引擎（Elasticsearch）。

### 4.3.4 正则表达式（REGEXP）

```sql
-- MySQL 支持 POSIX 正则
SELECT * FROM user WHERE email REGEXP '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$';

-- 常用正则
SELECT * FROM user WHERE username REGEXP '^张';     -- 以"张"开头
SELECT * FROM user WHERE username REGEXP '三$';     -- 以"三"结尾
SELECT * FROM user WHERE phone REGEXP '^1[3-9]\\d{9}$';  -- 手机号格式
```

## 4.4 ORDER BY排序

```sql
-- 单列排序
SELECT * FROM user ORDER BY age DESC;

-- 多列排序（先按 age 升序，age 相同时按 id 降序）
SELECT * FROM user ORDER BY age ASC, id DESC;

-- 使用别名排序
SELECT id, age * 100 AS score FROM user ORDER BY score DESC;

-- 使用字段位置排序（不推荐，可读性差）
SELECT id, username, age FROM user ORDER BY 3 DESC;   -- 按第 3 列（age）排序
```

## 4.5 LIMIT分页

```sql
-- 前 10 条
SELECT * FROM user LIMIT 10;

-- 第 2 页（偏移 10 条，取 10 条）
SELECT * FROM user LIMIT 10, 10;           -- 方式一
SELECT * FROM user LIMIT 10 OFFSET 10;     -- 方式二（推荐，含义更明确）
```

> **深分页问题**：`LIMIT 1000000, 10` 实际会扫描前 1000010 行再丢弃前 1000000 行，数据量越大越慢。优化方案：
> - **游标分页（推荐）**：记录上一页最后一条的 ID，`WHERE id > 1000000 LIMIT 10`
> - **子查询 + 覆盖索引**：`SELECT * FROM t WHERE id >= (SELECT id FROM t ORDER BY id LIMIT 1000000, 1) LIMIT 10`
> - **禁止跳页**：用户体验上，只提供"上一页"/"下一页"而非任意页码跳转

## 4.6 聚合函数

| 函数 | 作用 | 注意 |
|------|------|------|
| COUNT(*) | 统计行数（包含 NULL） | 常用，InnoDB 下尽量用 COUNT(索引列) |
| COUNT(列) | 统计该列非 NULL 值的行数 | NULL 不计入 |
| COUNT(DISTINCT 列) | 统计该列不同值的个数 | NULL 不计入 |
| SUM(列) | 求和 | 忽略 NULL |
| AVG(列) | 平均值 | 忽略 NULL |
| MAX(列) | 最大值 | 可用于字符串、日期 |
| MIN(列) | 最小值 | 可用于字符串、日期 |

```sql
SELECT
  COUNT(*) AS total_users,
  COUNT(DISTINCT status) AS status_types,
  MAX(age) AS max_age,
  MIN(age) AS min_age,
  AVG(age) AS avg_age,
  SUM(balance) AS total_balance
FROM user;
```

> **COUNT 性能**：`COUNT(*)` ≈ `COUNT(1)` ≈ `COUNT(主键)` > `COUNT(字段)`。InnoDB 不会缓存 COUNT(*) 的结果，每次都是实时扫描（MyISAM 会缓存，但 MyISAM 不支持事务，不推荐使用）。

## 4.7 GROUP BY分组

```sql
-- 单列分组
SELECT status, COUNT(*) AS cnt FROM user GROUP BY status;

-- 多列分组
SELECT status, age, COUNT(*) AS cnt FROM user GROUP BY status, age;

-- 配合 WITH ROLLUP（汇总行）
SELECT status, COUNT(*) AS cnt FROM user
GROUP BY status WITH ROLLUP;  -- 最后多一行汇总
```

> **sql_mode 的 ONLY_FULL_GROUP_BY**：
> MySQL 5.7.5+ 默认启用此模式，要求 SELECT 中的非聚合列必须出现在 GROUP BY 中。否则报错。正确写法：
> ```sql
> -- 正确（dept 在 GROUP BY 中）
> SELECT dept, AVG(salary) FROM emp GROUP BY dept;
> -- 错误（name 不在 GROUP BY 中）
> SELECT name, dept, AVG(salary) FROM emp GROUP BY dept;
> ```
> 这能避免因 SQL 语义模糊导致的不确定结果，是 SQL 99 标准的要求。

## 4.8 HAVING过滤

HAVING 用于过滤分组后的结果（WHERE 用于过滤分组前的行）：

```sql
-- WHERE：分组前过滤
-- HAVING：分组后过滤
SELECT status, COUNT(*) AS cnt
FROM user
WHERE age > 18              -- 先过滤行
GROUP BY status
HAVING cnt > 5              -- 再过滤分组
ORDER BY cnt DESC;          -- 最后排序
```

> HAVING 中可以使用聚合函数（COUNT、SUM 等），WHERE 中不行。能用 WHERE 过滤的数据尽量在 WHERE 中处理——先过滤后分组，数据量更小，性能更好。

## 4.9 DISTINCT去重

```sql
-- 单列去重
SELECT DISTINCT status FROM user;

-- 多列去重（组合去重）
SELECT DISTINCT status, age FROM user;
```

> DISTINCT 本质是对结果集做全排序（或 hash）去重，代价较大。如果只需要判断某字段是否有重复值，用 `GROUP BY ... HAVING COUNT(*) > 1` 更合适。

## 4.10 CASE WHEN条件表达式

```sql
-- 简单 CASE 表达式
SELECT username,
  CASE status
    WHEN 0 THEN '禁用'
    WHEN 1 THEN '启用'
    ELSE '未知'
  END AS status_name
FROM user;

-- 搜索 CASE 表达式（更灵活）
SELECT username, age,
  CASE
    WHEN age < 18 THEN '未成年'
    WHEN age BETWEEN 18 AND 60 THEN '成年人'
    WHEN age > 60 THEN '老年人'
    ELSE '未知'
  END AS age_group
FROM user;

-- 在聚合中使用 CASE
SELECT
  COUNT(*) AS total,
  COUNT(CASE WHEN status = 1 THEN 1 END) AS active,
  COUNT(CASE WHEN status = 0 THEN 1 END) AS disabled
FROM user;
```

# 5. DCL（数据控制语言）

DCL 用于管理数据库用户权限。

## 5.1 用户管理

```sql
-- 创建用户（8.0 必须先创建用户再授权，不能 GRANT 隐式创建）
CREATE USER 'app_user'@'%' IDENTIFIED BY 'StrongPass123!';

-- 修改密码
ALTER USER 'app_user'@'%' IDENTIFIED BY 'NewStrongPass456!';

-- 删除用户
DROP USER 'app_user'@'%';

-- 查看所有用户
SELECT host, user FROM mysql.user;
```

> **MySQL 8.0 用户管理变更**：8.0 不再支持 `GRANT ... IDENTIFIED BY` 隐式创建用户。必须显式 `CREATE USER` 后再 `GRANT`。

## 5.2 权限管理

```sql
-- 授予权限
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'app_user'@'%';

-- 授予所有权限（生产环境慎用）
GRANT ALL PRIVILEGES ON mydb.* TO 'app_user'@'%';

-- 查看用户权限
SHOW GRANTS FOR 'app_user'@'%';

-- 撤销权限
REVOKE INSERT ON mydb.* FROM 'app_user'@'%';

-- 刷新权限
FLUSH PRIVILEGES;
```

**常见权限列表**：

| 权限 | 说明 |
|------|------|
| SELECT、INSERT、UPDATE、DELETE | 基础 CRUD 权限 |
| CREATE、ALTER、DROP、INDEX | 结构修改权限 |
| CREATE VIEW、SHOW VIEW | 视图权限 |
| CREATE ROUTINE、EXECUTE | 存储过程权限 |
| GRANT OPTION | 允许用户将自己拥有的权限授予他人 |
| ALL PRIVILEGES | 所有权限（不含 GRANT OPTION） |

**最小权限原则（Principle of Least Privilege）**：
- 应用账号只授予 CRUD 权限，不要给 DDL 权限
- 读写分离时，从库账号只给 SELECT 权限
- CI/CD 部署账号给 ALTER/CREATE 权限，但限制 IP 段
