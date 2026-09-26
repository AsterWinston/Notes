# 1.系统数据库

MySQL 安装完成后内置了四个系统数据库：

## 1.1mysql库

存储 MySQL 服务器正常运行所需的**元数据**：

| 内容 | 关键表 |
|------|--------|
| 用户信息 | `user`、`db`、`tables_priv`、`columns_priv` |
| 时区信息 | `time_zone` 系列表 |
| 存储过程/函数 | `proc`、`func` |
| 事件调度器 | `event` |
| 复制信息 | `slave_master_info`、`slave_relay_log_info`（主从复制） |

## 1.2information_schema库

提供对数据库**元数据**的只读访问（符合 SQL 标准），包含数据库、表、字段、索引、权限等所有对象的定义信息：

```sql
-- 查看库中所有表
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb';

-- 查看表的所有字段
SELECT COLUMN_NAME, DATA_TYPE, COLUMN_COMMENT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'user';

-- 查看表的索引
SELECT INDEX_NAME, COLUMN_NAME FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'user';

-- 查看库大小
SELECT
  table_schema AS '数据库',
  ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS '大小(MB)'
FROM information_schema.TABLES
GROUP BY table_schema;
```

> **注意**：InnoDB 的 `information_schema` 统计信息可能是估算值（采样计算），不是精确值。执行 `ANALYZE TABLE` 可更新统计信息。

## 1.3performance_schema库

用于收集数据库**运行时性能数据**，提供底层监控能力。MySQL 5.7 后默认开启。

```sql
-- 查看 SQL 执行统计
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;

-- 查看等待事件
SELECT * FROM performance_schema.events_waits_current;

-- 查看 MDL 锁
SELECT * FROM performance_schema.metadata_locks;

-- 查看数据锁
SELECT * FROM performance_schema.data_locks;

-- 查看线程信息
SELECT * FROM performance_schema.threads;
```

> `performance_schema` 会消耗额外内存（约 1-2 GB 用于大实例），但相比其提供的诊断能力，这代价完全值得。

## 1.4sys库（MySQL 5.7+）

基于 `performance_schema` 和 `information_schema` 的**视图集合**，提供更易读的性能诊断信息：

```sql
-- 查看最耗 IO 的 SQL
SELECT * FROM sys.io_global_by_file_by_bytes;

-- 查看未使用的索引
SELECT * FROM sys.schema_unused_indexes;

-- 查看冗余索引
SELECT * FROM sys.schema_redundant_indexes;

-- 查看内存使用
SELECT * FROM sys.memory_global_total;

-- 查看当前活跃事务
SELECT * FROM sys.innodb_lock_waits;

-- 查看当前执行的 SQL 语句
SELECT * FROM sys.processlist;
```

# 2.常用管理工具

## 2.1mysql——命令行客户端

```bash
# 连接数据库
mysql -u root -p
mysql -u root -p -h 192.168.1.100 -P 3306
mysql -u root -p -D mydb               # 直接进入指定库

# 执行单条 SQL
mysql -u root -p -D mydb -e "SHOW TABLES;"

# 导入 SQL 文件
mysql -u root -p mydb < backup.sql

# 从 SQL 文件恢复（在 mysql 客户端内）
source /path/to/backup.sql;

# 输出格式控制
mysql -u root -p -e "SELECT * FROM user" -E  # 垂直显示
mysql -u root -p -e "SELECT * FROM user" -t  # 表格显示
```

## 2.2mysqladmin——管理工具

```bash
# 查看服务器状态
mysqladmin -u root -p status

# 查看服务器变量
mysqladmin -u root -p variables

# 查看连接数
mysqladmin -u root -p processlist

# 关闭服务器
mysqladmin -u root -p shutdown

# 刷新权限
mysqladmin -u root -p flush-privileges

# 刷新日志
mysqladmin -u root -p flush-logs

# 检查服务器是否存活（常用于健康检查脚本）
mysqladmin -u root -p ping
```

## 2.3mysqlbinlog——二进制日志工具

```bash
# 查看 binlog 内容
mysqlbinlog /var/lib/mysql/binlog.000001

# 输出 SQL 语句（ROW 格式转 SQL）
mysqlbinlog -v /var/lib/mysql/binlog.000001

# 指定起止时间
mysqlbinlog --start-datetime='2026-05-17 10:00:00' \
            --stop-datetime='2026-05-17 11:00:00' \
            /var/lib/mysql/binlog.000001

# 指定偏移量
mysqlbinlog --start-position=1000 --stop-position=2000 /var/lib/mysql/binlog.000001

# 恢复数据（将 binlog 重放）
mysqlbinlog /var/lib/mysql/binlog.000001 | mysql -u root -p
```

## 2.4mysqlshow——显示数据库对象

```bash
# 查看所有数据库
mysqlshow -u root -p

# 查看库中所有表
mysqlshow -u root -p mydb

# 查看表结构
mysqlshow -u root -p mydb user

# 查看表统计
mysqlshow -u root -p --status mydb user
```

## 2.5mysqldump——逻辑备份工具

mysqldump 是 MySQL 最常用的逻辑备份工具，将数据库导出为 SQL 文本文件。

```bash
# 备份单个数据库
mysqldump -u root -p mydb > mydb_backup.sql

# 备份多个数据库
mysqldump -u root -p --databases db1 db2 > backup.sql

# 备份所有数据库
mysqldump -u root -p --all-databases > full_backup.sql

# 只备份表结构（无数据）
mysqldump -u root -p --no-data mydb > schema.sql

# 只备份数据（无表结构）
mysqldump -u root -p --no-create-info mydb > data.sql

# 备份指定表
mysqldump -u root -p mydb user `order` > tables.sql

# 一致性备份（不锁表，仅支持 InnoDB）
mysqldump --single-transaction -u root -p mydb > backup.sql

# 备份存储过程和函数
mysqldump -u root -p --routines --triggers mydb > backup.sql

# 压缩备份
mysqldump -u root -p mydb | gzip > mydb_backup.sql.gz

# 恢复
mysql -u root -p mydb < mydb_backup.sql
```

**mysqldump 备份原理**（InnoDB + `--single-transaction`）：
1. 开启一个 RR 级别的事务
2. 通过 MVCC 获取事务开始时的一致性快照
3. 逐步导出快照中的数据

> **大表备份**：对于 TB 级别的数据库，mysqldump 速度慢且影响性能，应使用物理备份工具（**Percona XtraBackup**、MySQL Enterprise Backup）。

## 2.6mysqlimport / LOAD DATA——数据导入

```sql
-- mysqlimport 命令行工具
mysqlimport -u root -p --local --fields-terminated-by=',' mydb /tmp/user.csv

-- LOAD DATA（功能更强，在 mysql 客户端中使用）
LOAD DATA LOCAL INFILE '/tmp/user.csv'
INTO TABLE user
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS                        -- 跳过 CSV 表头
(id, username, email, @age)
SET age = IF(@age = '', 0, @age);    -- 处理空值
```

# 3.用户与权限管理

## 3.1用户管理

```sql
-- 创建用户（MySQL 8.0 必须显式 CREATE USER）
CREATE USER 'app_user'@'%' IDENTIFIED BY 'StrongPass123!';
CREATE USER 'readonly'@'192.168.1.%' IDENTIFIED BY 'ReadOnly456!';

-- 修改密码
ALTER USER 'app_user'@'%' IDENTIFIED BY 'NewStrongPass789!';

-- 锁定/解锁用户
ALTER USER 'app_user'@'%' ACCOUNT LOCK;
ALTER USER 'app_user'@'%' ACCOUNT UNLOCK;

-- 删除用户
DROP USER 'app_user'@'%';

-- 查看所有用户
SELECT user, host, account_locked FROM mysql.user;

-- 密码过期策略
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;  -- 每 90 天过期
ALTER USER 'app_user'@'%' PASSWORD EXPIRE NEVER;             -- 永不过期
```

## 3.2权限管理

```sql
-- 授予权限
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app_user'@'%';
GRANT SELECT ON mydb.* TO 'readonly'@'192.168.1.%';
GRANT ALL PRIVILEGES ON mydb.* TO 'admin'@'localhost';
GRANT EXECUTE ON mydb.* TO 'app_user'@'%';                  -- 执行存储过程
GRANT CREATE, ALTER, DROP ON mydb.* TO 'dev_user'@'%';      -- DDL 权限

-- 查看权限
SHOW GRANTS FOR 'app_user'@'%';

-- 撤销权限
REVOKE DELETE ON mydb.* FROM 'app_user'@'%';
REVOKE ALL PRIVILEGES ON mydb.* FROM 'app_user'@'%';
```

**权限粒度层级**：

| 层级 | 语法 | 说明 |
|------|------|------|
| 全局 | `GRANT ... ON *.* TO ...` | 所有库 |
| 数据库 | `GRANT ... ON mydb.* TO ...` | 指定库下所有表 |
| 表 | `GRANT ... ON mydb.user TO ...` | 指定表 |
| 列 | `GRANT SELECT(id, name) ON mydb.user TO ...` | 指定表的具体列 |
| 存储过程 | `GRANT EXECUTE ON PROCEDURE mydb.proc TO ...` | 指定存储过程 |

## 3.3角色管理（MySQL 8.0）

MySQL 8.0 引入了**角色（Role）**，类似用户组的权限集合，可简化权限管理：

```sql
-- 创建角色
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- 给角色授权
GRANT SELECT ON mydb.* TO 'app_read';
GRANT INSERT, UPDATE, DELETE ON mydb.* TO 'app_write';
GRANT ALL PRIVILEGES ON mydb.* TO 'app_admin';

-- 将角色赋给用户
GRANT 'app_read' TO 'readonly_user'@'%';
GRANT 'app_read', 'app_write' TO 'app_user'@'%';

-- 激活角色（默认非活跃，需手动激活）
SET DEFAULT ROLE ALL TO 'app_user'@'%';

-- 查看角色
SELECT * FROM mysql.role_edges;
```

# 4.日常监控与维护

## 4.1关键监控指标

```sql
-- 连接数
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';    -- 历史峰值

-- QPS / TPS（需计算差值）
SHOW STATUS LIKE 'Questions';               -- 总查询数
SHOW STATUS LIKE 'Com_commit';              -- 事务提交数
SHOW STATUS LIKE 'Com_rollback';            -- 事务回滚数

-- InnoDB Buffer Pool 命中率
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- 锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 长事务
SELECT trx_id, trx_state, trx_started,
  TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS trx_duration_sec
FROM information_schema.INNODB_TRX
WHERE trx_started < DATE_SUB(NOW(), INTERVAL 60 SECOND)
ORDER BY trx_started;

-- 当前连接列表
SHOW FULL PROCESSLIST;
SELECT * FROM information_schema.PROCESSLIST;
```

## 4.2查看数据库大小

```sql
-- 所有数据库大小
SELECT
  table_schema AS '数据库',
  ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS '大小(MB)'
FROM information_schema.TABLES
GROUP BY table_schema
ORDER BY SUM(data_length + index_length) DESC;

-- 单库下所有表大小
SELECT
  table_name AS '表名',
  ROUND(data_length / 1024 / 1024, 2) AS '数据大小(MB)',
  ROUND(index_length / 1024 / 1024, 2) AS '索引大小(MB)',
  table_rows AS '行数'
FROM information_schema.TABLES
WHERE table_schema = 'mydb'
ORDER BY data_length DESC;
```

## 4.3查看连接状态

```sql
-- 按状态分组统计连接
SELECT state, COUNT(*) AS cnt
FROM information_schema.PROCESSLIST
GROUP BY state
ORDER BY cnt DESC;

-- 查看正在执行的 SQL
SELECT id, user, host, db, command, time, state, info
FROM information_schema.PROCESSLIST
WHERE command != 'Sleep'
ORDER BY time DESC;

-- 杀掉连接
KILL CONNECTION 123;    -- 断开连接
KILL QUERY 123;         -- 仅终止当前查询，保留连接
```

## 4.4表维护

```sql
-- 分析表（更新索引统计信息，优化器决策依据）
ANALYZE TABLE user;

-- 检查表
CHECK TABLE user;

-- 优化表（重建表 + 重组索引，回收碎片空间）
-- 等价于 ALTER TABLE ... ENGINE=InnoDB;
OPTIMIZE TABLE user;

-- 修复表（仅 MyISAM/Archive，InnoDB 不支持）
REPAIR TABLE user;
```

> **定期维护建议**：
> - 低峰期执行 `OPTIMIZE TABLE`（对频繁 DELETE 的表回收碎片）
> - 每周 `ANALYZE TABLE` 更新统计信息，保证优化器决策准确
> - `OPTIMIZE TABLE` 会锁表，大表请用 `pt-online-schema-change`

## 4.5InnoDB状态查询

```sql
-- 查看 InnoDB 引擎详细信息
SHOW ENGINE INNODB STATUS\G;

-- 关键段落说明
-- TRANSACTIONS：当前事务状态
-- FILE I/O：文件 IO 线程信息
-- BUFFER POOL AND MEMORY：缓冲池使用情况
-- ROW OPERATIONS：每秒行操作数
-- LATEST DETECTED DEADLOCK：最近一次死锁信息
```

## 4.6日志管理

```sql
-- 刷新日志（关闭当前日志文件，创建新文件）
FLUSH LOGS;

-- 查看错误日志位置
SHOW VARIABLES LIKE 'log_error';

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';

-- 查看 binlog 列表
SHOW BINARY LOGS;
SHOW MASTER STATUS;

-- 删除 binlog
PURGE BINARY LOGS TO 'binlog.000010';                    -- 删除指定编号之前的
PURGE BINARY LOGS BEFORE '2026-05-01 00:00:00';          -- 删除指定时间之前的
RESET MASTER;                                             -- 删除所有 binlog
```
