# 1.错误日志（Error Log）

错误日志记录了 MySQL 启动和停止时以及运行过程中发生的错误信息。当数据库出现故障导致无法正常使用时，**应首先查看此日志**。

## 1.1基本特性

- **默认开启**，无法关闭
- 默认文件名：`mysqld.log`（Linux `/var/log/mysqld.log`，Ubuntu `/var/log/mysql/error.log`）
- 记录内容：启动/关闭事件、运行错误、未处理的异常、复制错误等

## 1.2查看与配置

```sql
-- 查看错误日志位置
SHOW VARIABLES LIKE 'log_error';

-- 查看错误日志详细级别（MySQL 8.0）
SHOW VARIABLES LIKE 'log_error_verbosity';
-- 1 = 仅错误，2 = 错误 + 警告，3 = 错误 + 警告 + 备注（默认）
```

**my.cnf 配置**：

```
[mysqld]
log_error = /var/log/mysql/mysqld.log
log_error_verbosity = 3
```

> `log_error_verbosity = 3` 在生产环境推荐，提供最完整的诊断信息，方便排查问题。

## 1.3关键错误类型

| 错误类型 | 示例 | 解决思路 |
|---------|------|---------|
| 启动失败 | `InnoDB: Unable to lock ./ibdata1` | 检查是否有另一个 mysqld 进程 |
| 磁盘满 | `Error writing file '/tmp/xxx' (Errcode: 28 - No space left on device)` | 清理磁盘空间 |
| 连接数满 | `Too many connections` | 增加 max_connections 或检查连接泄露 |
| 表损坏 | `Incorrect key file for table 'xxx'` | `REPAIR TABLE`（MyISAM）或从备份恢复 |

# 2.二进制日志（Binary Log / Binlog）

Binlog 记录了所有 DDL 和 DML 操作（但不包括 SELECT 和 SHOW），是 MySQL 中**最重要**的日志之一。

## 2.1作用

| 作用 | 说明 |
|------|------|
| **数据恢复** | 数据库误删数据后，通过回放 binlog 恢复到指定时间点 |
| **主从复制** | 主库将 binlog 传给从库，从库重放实现数据同步 |
| **数据审计** | 配合 binlog 解析工具，追溯数据变更历史 |
| **数据同步** | Canal、Maxwell 等工具监听 binlog 实现 MySQL → Kafka/ES 同步 |

## 2.2配置

MySQL 8.0 默认开启 binlog（`log_bin = ON`）。生产环境务必确认已开启。

```
[mysqld]
# 开启 binlog（必须指定文件名前缀）
log_bin = /var/lib/mysql/binlog/mysql-bin

# binlog 格式：STATEMENT / ROW / MIXED
binlog_format = ROW

# 单个 binlog 文件最大大小（超过后自动切换新文件）
max_binlog_size = 104857600    # 100MB

# binlog 过期自动清理时间（秒）（MySQL 8.0），默认 30 天
binlog_expire_logs_seconds = 2592000

# 每次事务提交时刷 binlog 到磁盘（1 = 刷新，0 = 不刷新）
sync_binlog = 1

# 记录 binlog 的行映像（full = 完整记录，minimal = 仅记录变更列）
binlog_row_image = FULL
```

## 2.3三种格式对比

| 格式 | 记录内容 | 优点 | 缺点 | 使用场景 |
|------|---------|------|------|---------|
| **STATEMENT** | SQL 语句原文 | 日志量小、可读性好 | 某些函数可能导致主从不一致（如 `UUID()`、`NOW()`） | 已基本淘汰 |
| **ROW**（默认） | 每行数据的变更详情 | 安全、精确，不会出现主从不一致 | 日志量大（UPDATE 全表 1000 万行 → 1000 万条记录） | **生产环境首选** |
| **MIXED** | STATEMENT + ROW 混合 | 大部分用 STATEMENT，特殊用 ROW | 复杂场景下仍有不确定性 | 过渡方案 |

> **推荐一律使用 ROW 格式**。虽然 ROW 日志量大，但磁盘廉价 + 配合 `binlog_row_image = MINIMAL` 可显著减少日志量（仅记录变更列的前后值）。ROW 格式是安全可靠的基础。

## 2.4查看Binlog

### 命令行查看

```bash
# 查看 binlog 文件列表
mysql -u root -p -e "SHOW BINARY LOGS;"

# 查看当前正在写的 binlog
mysql -u root -p -e "SHOW MASTER STATUS;"

# 查看 binlog 事件
mysqlbinlog /var/lib/mysql/binlog/mysql-bin.000001

# 查看 ROW 格式 binlog 的伪 SQL（-v 显示，-vv 更详细）
mysqlbinlog -vv /var/lib/mysql/binlog/mysql-bin.000001

# 按时间范围查看
mysqlbinlog --start-datetime='2026-05-17 10:00:00' \
            --stop-datetime='2026-05-17 11:00:00' \
            /var/lib/mysql/binlog/mysql-bin.000001

# 按偏移量查看
mysqlbinlog --start-position=1234 --stop-position=5678 \
            /var/lib/mysql/binlog/mysql-bin.000001
```

### SQL中查看

```sql
-- 查看 binlog 文件列表
SHOW BINARY LOGS;

-- 查看当前 binlog 位置
SHOW MASTER STATUS;

-- 查看 binlog 事件
SHOW BINLOG EVENTS IN 'mysql-bin.000001';
SHOW BINLOG EVENTS IN 'mysql-bin.000001' FROM 1234 LIMIT 10;
```

## 2.5基于Binlog的数据恢复

```bash
# 全量恢复 + binlog 增量恢复
# 1. 恢复上周日的全量备份
mysql -u root -p mydb < full_backup_20260511.sql

# 2. 回放全量备份之后的所有 binlog
mysqlbinlog /var/lib/mysql/binlog/mysql-bin.000010 \
            /var/lib/mysql/binlog/mysql-bin.000011 \
            /var/lib/mysql/binlog/mysql-bin.000012 | mysql -u root -p

# 3. 恢复到指定时间点（如误操作发生在 14:00，恢复到 13:59:59）
mysqlbinlog --stop-datetime='2026-05-17 13:59:59' \
            /var/lib/mysql/binlog/mysql-bin.000012 | mysql -u root -p
```

> **恢复前务必先备份当前状态**，防止恢复操作覆盖有用数据。

## 2.6Binlog管理

```sql
-- 删除 binlog：删除指定编号之前的
PURGE BINARY LOGS TO 'mysql-bin.000020';

-- 删除指定时间之前的
PURGE BINARY LOGS BEFORE '2026-05-10 00:00:00';

-- 删除所有 binlog（主从复制环境慎用！）
RESET MASTER;

-- 刷新日志（关闭当前 binlog，开启新文件）
FLUSH LOGS;
```

> **不要手动 `rm` binlog 文件**，必须使用 `PURGE BINARY LOGS` 安全删除。

# 3.查询日志（General Query Log）

查询日志记录客户端的**所有操作语句**（包括 SELECT），是 binlog 的补集。但日志量极大，默认关闭。

```sql
-- 查看状态和位置
SHOW VARIABLES LIKE '%general_log%';

-- 临时开启
SET GLOBAL general_log = 1;

-- 永久开启（my.cnf）
-- general_log = 1
-- general_log_file = /var/lib/mysql/mysql-query.log
```

> **使用建议**：查询日志体积增长极快，仅在**短期调试**时开启，用完立即关闭。生产环境不要长期开启。

# 4.慢查询日志（Slow Query Log）

慢查询日志记录所有执行时间超过阈值的 SQL，是**性能优化最常用的日志**。

## 4.1配置

```sql
-- 查看状态
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- 临时开启
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;  -- 阈值设为 1 秒

-- 永久配置（my.cnf）
```

```
[mysqld]
# 开启慢查询
slow_query_log = 1
# 慢查询阈值（秒，可精确到微秒：0.001 = 1ms）
long_query_time = 1
# 未使用索引的查询也记录
log_queries_not_using_indexes = 1
# 慢管理语句也记录（如 ALTER TABLE）
log_slow_admin_statements = 1
# 慢查询日志文件路径
slow_query_log_file = /var/lib/mysql/mysql-slow.log
# 记录慢查询的最小扫描行数（配合 long_query_time）
min_examined_row_limit = 1000
```

## 4.2慢查询分析工具

### mysqldumpslow

```bash
# 获取访问次数最多的 10 条 SQL
mysqldumpslow -s c -t 10 /var/lib/mysql/mysql-slow.log

# 获取耗时最长的 10 条 SQL
mysqldumpslow -s t -t 10 /var/lib/mysql/mysql-slow.log

# 获取平均耗时最长的 10 条
mysqldumpslow -s at -t 10 /var/lib/mysql/mysql-slow.log

# 获取锁等待时间最长的 10 条
mysqldumpslow -s l -t 10 /var/lib/mysql/mysql-slow.log

# 按指定模式过滤（如只分析涉及 user 表的 SQL）
mysqldumpslow -s t -t 10 -g 'user' /var/lib/mysql/mysql-slow.log
```

> mysqldumpslow 是 MySQL 自带的慢查询分析工具。更适合生产级分析的是 **pt-query-digest**（Percona Toolkit）。

### pt-query-digest

```bash
# 分析慢查询日志
pt-query-digest /var/lib/mysql/mysql-slow.log > slow_report.txt

# 分析指定时间范围的 binlog
pt-query-digest --type binlog /var/lib/mysql/binlog/mysql-bin.000010 > report.txt

# 实时分析（tcpdump 抓取）
pt-query-digest --type tcpdump /path/to/tcpdump.out
```

> `pt-query-digest` 会输出：各 SQL 的执行次数、总耗时、平均耗时、锁时间、行扫描数等详细统计，按耗时排序。

## 4.3日志分析流程

```
1. 开启慢查询日志
       ↓
2. 运行一段时间（至少 1 天，覆盖业务高低峰）
       ↓
3. 运行 pt-query-digest 或 mysqldumpslow 分析
       ↓
4. 找出 TOP N 慢 SQL
       ↓
5. EXPLAIN + 索引优化 + SQL 改写
       ↓
6. 验证优化效果（继续分析下一周期的日志）
       ↓
7. 优化后关闭或调高 long_query_time
```

# 5.Redo Log vs Undo Log vs Binlog

| 日志 | 层面 | 记录内容 | 作用 |
|------|------|---------|------|
| **Redo Log** | InnoDB 引擎层 | 数据页的物理修改（"页 X 偏移量 Y 改成了 Z"） | 崩溃恢复，保证持久性 |
| **Undo Log** | InnoDB 引擎层 | 数据修改前的逻辑状态（"改了什么"） | 事务回滚 + MVCC 版本链 |
| **Binlog** | MySQL Server 层 | SQL 语句或行变更（逻辑日志） | 主从复制、数据恢复 |

**写入顺序**（WAL）：

```
1. 修改 Buffer Pool 中的数据页
2. 写 Undo Log（记录修改前的值）
3. 写 Redo Log（记录修改后的值，先写 Log Buffer）
4. 修改 Buffer Pool 中的页（标记为脏页）
5. 事务提交：Redo Log 刷盘（innodb_flush_log_at_trx_commit=1）
6. 写 Binlog
7. Binlog 刷盘（sync_binlog=1）
8. 返回客户端"成功"
9. 后续：脏页异步刷盘（Checkpoint）
```

> **两阶段提交**：Redo Log 和 Binlog 通过两阶段提交保证一致性。阶段一：写 Redo Log（Prepare 状态），阶段二：写 Binlog 后 Redo Log 改为 Commit 状态。崩溃恢复时以此判断事务是否已提交。
