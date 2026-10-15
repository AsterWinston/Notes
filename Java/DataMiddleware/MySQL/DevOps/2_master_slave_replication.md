# 1. 主从复制概述

主从复制是指将**主库（Master）**的 DDL 和 DML 操作通过二进制日志（binlog）传到**从库（Slave）**，从库对这些日志重新执行（重放），从而使从库和主库的数据保持同步。

## 1.1 复制的优点

| 优点 | 说明 |
|------|------|
| **高可用（HA）** | 主库故障时快速切换到从库，减少停机时间 |
| **读写分离** | 主库负责写，从库负责读，分散负载 |
| **数据备份** | 在从库执行备份，不影响主库服务 |
| **灾难恢复** | 异地机房部署从库，防止机房级故障 |
| **报表/分析分离** | 复杂查询在从库执行，避免影响 OLTP 业务 |
| **在线 DDL 辅助** | 先在从库执行 DDL 变更，确认无误后切换 |

## 1.2 复制类型

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| **异步复制** | 主库提交后立即返回，不等待从库确认 | 性能要求高的场景（MySQL 默认） |
| **半同步复制** | 主库提交后等待至少一个从库确认收到 binlog | 数据一致性要求较高的场景 |
| **组复制（MGR）** | 基于 Paxos 协议的多数派复制 | 强一致性、自动故障转移 |

# 2. 主从复制原理

## 2.1 复制三步骤

```
┌──────────────┐    Binlog     ┌──────────────┐
│   Master     │ ───────────→  │   Slave      │
│              │               │              │
│ 1. Binlog    │               │ 2. Relay Log │
│    Dump      │               │    (I/O线程)  │
└──────────────┘               │              │
                               │ 3. SQL线程   │
                               │    重放      │
                               └──────────────┘
```

**步骤一：主库写入 binlog**
- 主库事务提交时，将数据变更以事件（Event）形式记录到 binlog
- `sync_binlog = 1` 确保 binlog 写入磁盘

**步骤二：从库 I/O 线程读取 binlog**
- 从库启动 I/O 线程，与主库建立连接
- 主库启动 **Binlog Dump 线程**，将 binlog 内容发给从库
- 从库 I/O 线程将接收的 binlog 内容写入 **中继日志（Relay Log）**

**步骤三：从库 SQL 线程重放**
- 从库 SQL 线程读取 Relay Log 中的事件并重放执行
- 完成数据同步

## 2.2 关键线程

| 线程 | 位置 | 职责 |
|------|------|------|
| **Binlog Dump 线程** | 主库 | 读取 binlog 并发送给从库 |
| **I/O 线程** | 从库 | 接收 binlog 并写入 Relay Log |
| **SQL 线程** | 从库 | 读取 Relay Log 并重放执行（MySQL 5.6+ 支持多线程） |
| **Coordinator 线程** | 从库 | 多线程复制时协调 Worker 线程分配 |

> **MySQL 5.6 之前**，从库 SQL 线程是单线程，主库的并发写入在从库串行重放，在大写量场景下容易产生复制延迟。**MySQL 5.6 引入基于 Schema（库）的并行复制**，**5.7 引入基于组提交（LOGICAL_CLOCK）的并行复制**，大幅减少延迟。**8.0 支持基于 WRITESET 的并行复制。**

## 2.3 复制格式与数据一致性

**STATEMENT 格式**：记录 SQL 语句。风险：不确定性函数（`NOW()`、`UUID()`、`RAND()`）导致主从不一致。

**ROW 格式**：记录每行变更。安全可靠，是生产环境唯一推荐。

> ROW 格式 + `binlog_row_image = FULL`（完整记录）能保证最大安全性，但日志量大。可用 `MINIMAL` 减少日志量（仅记录变更列的前/后值）。

# 3. 搭建主从复制

## 3.1 环境准备

| 角色 | IP | MySQL 版本 |
|------|------|-----------|
| 主库 | 192.168.1.100 | 8.0 |
| 从库 | 192.168.1.101 | 8.0 |

**前提**：
- 两台服务器 MySQL 版本建议一致（至少主库版本 ≤ 从库版本）
- 主库和从库的 `server-id` 必须不同
- 主库 binlog 已开启

## 3.2 主库配置

```sql
-- my.cnf 增加以下配置
[mysqld]
server-id = 1                    # 唯一标识（必须唯一）
log_bin = /var/lib/mysql/mysql-bin  # 开启 binlog
binlog_format = ROW              # ROW 格式

-- 创建复制专用账号
CREATE USER 'repl'@'%' IDENTIFIED BY 'ReplPass123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- 记录当前 binlog 位置（锁定主库片刻，或使用 --single-transaction）
FLUSH TABLES WITH READ LOCK;

-- 查看主库状态，记录文件名和位置
SHOW MASTER STATUS;
-- +------------------+----------+
-- | File             | Position |
-- +------------------+----------+
-- | mysql-bin.000001 |     1234 |
-- +------------------+----------+

-- 备份数据（在另一个终端）
-- mysqldump --single-transaction -u root -p --all-databases > backup.sql

UNLOCK TABLES;
```

## 3.3 从库配置

```sql
-- my.cnf 增加以下配置
[mysqld]
server-id = 2

-- 导入主库备份
-- mysql -u root -p < backup.sql

-- 配置复制源
CHANGE REPLICATION SOURCE TO       -- MySQL 8.0.23+ 语法
  SOURCE_HOST = '192.168.1.100',
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'ReplPass123!',
  SOURCE_LOG_FILE = 'mysql-bin.000001',
  SOURCE_LOG_POS = 1234;

-- 或使用旧语法（MySQL 8.0.23 之前）
CHANGE MASTER TO
  MASTER_HOST = '192.168.1.100',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'ReplPass123!',
  MASTER_LOG_FILE = 'mysql-bin.000001',
  MASTER_LOG_POS = 1234;

-- 启动复制
START REPLICA;        -- MySQL 8.0.22+
START SLAVE;          -- 旧语法

-- 查看复制状态
SHOW REPLICA STATUS\G;
-- 关键字段：
--   Replica_IO_Running: Yes   ← I/O 线程正常
--   Replica_SQL_Running: Yes  ← SQL 线程正常
--   Seconds_Behind_Source: 0  ← 复制延迟（0 = 无延迟）
```

## 3.4 使用GTID搭建（推荐）

GTID（Global Transaction Identifier，全局事务标识符）让每个事务有全局唯一 ID，简化主从切换和故障恢复。

```sql
-- 主库 my.cnf
gtid_mode = ON
enforce_gtid_consistency = ON

-- 从库 my.cnf（同上配置）

-- 从库配置复制（GTID 模式，无需指定 binlog 文件和位置）
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '192.168.1.100',
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'ReplPass123!',
  SOURCE_AUTO_POSITION = 1;          -- 自动定位

-- 启动复制
START REPLICA;
```

> **GTID 优势**：主从切换时不需要手动指定 binlog 文件和位置，系统自动定位。配合 Orchestrator 等工具可实现自动化故障转移。

# 4. 复制监控

```sql
-- 查看从库状态
SHOW REPLICA STATUS\G;

-- 关键监控指标
-- Seconds_Behind_Source：延迟秒数（0 表示无延迟）
-- Replica_IO_Running：IO 线程状态
-- Replica_SQL_Running：SQL 线程状态
-- Last_IO_Error / Last_SQL_Error：最近错误信息

-- 查看主库的从库连接
SHOW PROCESSLIST;  -- 查找 Binlog Dump 线程
SHOW REPLICAS;      -- MySQL 8.0.22+
```

**重要**：`Seconds_Behind_Source` 为 0 并不代表完全无延迟。它只表示 SQL 线程正在处理的 Relay Log 事件的时间戳是刚刚的。但如果 I/O 线程传输慢或网络抖动，仍可能有延迟。

# 5. 常见复制问题

## 5.1 主从延迟

### 5.1.1 原因
- 从库服务器性能弱于主库
- 从库负载高（大量慢查询占用资源）
- 主库写并发高，从库单线程重放跟不上
- 网络带宽/延迟
- 大事务（一个事务包含百万行变更，从库重放耗时）

### 5.1.2 解决方案

| 方案 | 说明 |
|------|------|
| **并行复制** | 5.6 基于 Schema + 5.7 基于 LOGICAL_CLOCK → 8.0 WRITESET |
| **提升从库硬件** | 使用与主库相同或更好的配置 |
| **分库** | 将不同库复制到不同从库，分散负载 |
| **金丝雀发布** | 应用层对延迟敏感的读切换到主库 |
| **半同步复制** | 降低延迟容忍度 |

```sql
-- 设置并行复制（从库）
STOP REPLICA;
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';   -- 8.0 推荐 WRITESET
SET GLOBAL replica_parallel_workers = 4;              -- Worker 线程数
START REPLICA;

-- 查看并行复制状态
SELECT * FROM performance_schema.replication_applier_status_by_worker;
```

## 5.2 主从不一致

### 5.2.1 原因
- binlog 为 STATEMENT 格式且 SQL 含不确定性函数
- 从库有人为直接写入
- 从库在执行 DDL 时出错跳过
- 恢复时 binlog 位置错误

### 5.2.2 修复工具

- **pt-table-checksum**（Percona Toolkit）：对比主从数据差异
- **pt-table-sync**：修复不一致的数据

## 5.3 复制中断

```sql
-- 查看错误
SHOW REPLICA STATUS\G;  -- 查看 Last_SQL_Error

-- 常见错误码
-- 1062：Duplicate entry → 主键冲突（从库可能被人为写入了数据）
-- 1032：Can't find record → 从库找不到要更新/删除的行
-- 1146：Table doesn't exist → 从库缺少表

-- 跳过错误（慎用！可能导致数据不一致）
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START REPLICA;
```

# 6. 半同步复制

异步复制下，主库提交后不管从库是否收到 binlog，若主库宕机且 binlog 未传到从库，则数据丢失。**半同步复制**要求至少一个从库收到 binlog（不要求从库完成重放）后，主库才返回成功。

```sql
-- 主库和从库都安装插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- 主库
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 等待超时后降级为异步

-- 从库
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
RESTART REPLICA IO_THREAD;  -- 重新连接主库
```

> **半同步复制 vs 异步复制**：半同步复制的性能低于异步复制（等待从库确认），但数据安全性更高。金融系统通常使用半同步或多副本强一致方案。

# 7. 主从切换

当主库故障时，需要将从库提升为新主库：

```bash
# 1. 确认主库不可用
# 2. 在候选从库上执行
STOP REPLICA;
RESET REPLICA ALL;       -- 清除复制配置

# 3. 将其他从库指向新主库
CHANGE MASTER TO MASTER_HOST = '192.168.1.101', ...;
START SLAVE;

# 4. 应用层切换连接地址
```

> **手动切换风险高**。生产环境推荐使用 **Orchestrator**、**MHA** 或 **MySQL Group Replication** 实现自动故障转移。

# 8. 双主复制（主-主）

两台服务器互为主从，均可读写：

```
Master A ←──→ Master B
  (写)          (写)
```

**优点**：两台服务器都可用于写入，无单点瓶颈。

**问题**：
- **冲突解决**：两台同时修改同一行数据产生冲突
- **自增 ID 冲突**：需配置 `auto_increment_increment` 和 `auto_increment_offset`

```sql
-- Master A
auto_increment_increment = 2    -- 步长
auto_increment_offset = 1       -- 偏移（A：1, 3, 5, 7...）

-- Master B
auto_increment_increment = 2
auto_increment_offset = 2       -- （B：2, 4, 6, 8...）
```

> **双主复制风险大**，不推荐作为常规方案。如需多写，建议使用 MySQL Group Replication 或 NDB Cluster。
