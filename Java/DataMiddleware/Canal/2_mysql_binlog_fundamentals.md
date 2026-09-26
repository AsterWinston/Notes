
# 1. 什么是 Binary Log

Binary Log（二进制日志，简称 binlog）是 MySQL Server 层维护的一种二进制日志，记录了对数据库执行更改的所有操作（DML、DDL）。它主要用于：

- **主从复制（Replication）**：Master 将 binlog 发送给 Slave，Slave 重放以实现数据同步
- **数据恢复（Point-in-Time Recovery）**：通过 binlog 将数据库恢复到指定时间点
- **CDC（Change Data Capture）**：第三方工具（如 Canal）通过解析 binlog 捕获数据变更

```
┌─────────────────────────────────────────┐
│              MySQL Server                │
│                                          │
│  ┌──────────┐      ┌──────────────┐     │
│  │  Client  │ ───> │   执行 SQL    │     │
│  │  请求    │      │              │     │
│  └──────────┘      └──────┬───────┘     │
│                           │              │
│                    ┌──────▼───────┐     │
│                    │  Storage     │     │
│                    │  Engine      │     │
│                    │ (InnoDB)     │     │
│                    └──────────────┘     │
│                           │              │
│                    ┌──────▼───────┐     │
│                    │  Redo Log    │     │
│                    │  (物理日志)  │     │
│                    └──────────────┘     │
│                           │              │
│                    ┌──────▼───────┐     │
│                    │  Binlog      │     │
│                    │  (逻辑日志)  │     │
│                    └──────────────┘     │
└─────────────────────────────────────────┘
```

## 1.1 Binlog vs Redo Log vs Undo Log

| 特性 | Binlog | Redo Log | Undo Log |
|------|--------|----------|----------|
| 层级 | Server 层 | InnoDB 引擎层 | InnoDB 引擎层 |
| 记录内容 | 逻辑日志（SQL 或行变更） | 物理日志（页修改） | 逻辑日志（回滚段） |
| 用途 | 复制、恢复 | 崩溃恢复 | 事务回滚、MVCC |
| 写入方式 | 追加写 | 循环写 | 随机写 |
| 文件管理 | `mysql-bin.000001` | `ib_logfile0/1` | 回滚段（表空间内） |

# 2. Binlog 的三种格式

MySQL 支持三种 binlog 格式，通过 `binlog_format` 参数控制。

## 2.1 STATEMENT 格式

记录每一条修改数据的 SQL 语句原文。

```sql
-- 设置 binlog 格式
SET GLOBAL binlog_format = 'STATEMENT';

-- 执行一条 UPDATE 语句
UPDATE users SET status = 1 WHERE id > 100;
```

**Binlog 中记录的内容**：
```
# at 1234
#230101 12:00:00 server id 1  end_log_pos 1300  Query
SET TIMESTAMP=1672545600;
UPDATE users SET status = 1 WHERE id > 100;
```

**优点**：日志量小，节省磁盘空间和网络带宽。

**缺点**：
- 某些含有 `LIMIT` 或 `NOW()` 等非确定性函数的 SQL 可能导致主从不一致
- 基于索引的数据修改可能导致 Slave 执行计划不同
- Canal 解析复杂，需要做 SQL 语法解析

## 2.2 ROW 格式（推荐）

记录每一行数据被修改的详情，包含修改前后的数据镜像。

```sql
SET GLOBAL binlog_format = 'ROW';

UPDATE users SET status = 1 WHERE id > 100;
-- 假设影响 3 行数据
```

**Binlog 中记录的内容**（逻辑表示）：
```
Table: test.users
Event: UPDATE_ROWS_EVENT
Rows:
  - Before: {id: 101, name: "Alice", status: 0}
    After:  {id: 101, name: "Alice", status: 1}
  - Before: {id: 102, name: "Bob", status: 0}
    After:  {id: 102, name: "Bob", status: 1}
  - Before: {id: 103, name: "Charlie", status: 0}
    After:  {id: 103, name: "Charlie", status: 1}
```

**优点**：
- 数据一致性高，不会出现主从不一致
- Canal 可以直接获取到行级变更，无需 SQL 解析
- 支持精确的回滚和数据恢复

**缺点**：
- 日志量比 STATEMENT 大（尤其是批量操作时）
- 不支持查看原始 SQL 语句

## 2.3 MIXED 格式

MySQL 自动选择 STATEMENT 或 ROW 格式。大多数情况下使用 STATEMENT，当可能出现数据不一致时自动切换为 ROW。

```sql
SET GLOBAL binlog_format = 'MIXED';
```

**切换规则**：
- 使用 `UUID()`、`NOW()`、`USER()` 等不确定函数时 → 切换为 ROW
- 使用 `INSERT ... SELECT` 时 → 切换为 ROW
- 使用用户自定义函数（UDF）时 → 切换为 ROW
- 使用 `LOAD_FILE()` 时 → 切换为 ROW

## 2.4 三种格式对比

| 维度 | STATEMENT | ROW | MIXED |
|------|-----------|-----|-------|
| 日志大小 | 小 | 大（批量操作时尤其明显） | 中等 |
| 数据一致性 | 可能不一致 | 完全一致 | 完全一致 |
| DDL 支持 | 支持 | 支持 | 支持 |
| Canal 解析 | 复杂（SQL解析） | 简单（直接读取） | 混合 |
| 审计能力 | 可以看到原 SQL | 可以看到行级变更 | 混合 |
| Canal 推荐度 | ❌ 不推荐 | ✅ 强烈推荐 | ⚠️ 可用 |

> **重要**：Canal 要求 MySQL 的 `binlog_format` 必须设置为 **ROW**。这是 Canal 正常工作的前提条件。

# 3. Binlog 事件类型

## 3.1 DML 事件（数据变更）

| 事件类型 | MySQL 值 | 说明 |
|----------|---------|------|
| WRITE_ROWS_EVENT | 30 | INSERT 操作的行级记录（v1） |
| UPDATE_ROWS_EVENT | 31 | UPDATE 操作的行级记录（v1） |
| DELETE_ROWS_EVENT | 32 | DELETE 操作的行级记录（v1） |
| WRITE_ROWS_EVENT_V1 | 19 | INSERT（旧版） |
| UPDATE_ROWS_EVENT_V1 | 20 | UPDATE（旧版） |
| DELETE_ROWS_EVENT_V1 | 21 | DELETE（旧版） |

## 3.2 DDL 事件（表结构变更）

| 事件类型 | MySQL 值 | 说明 |
|----------|---------|------|
| QUERY_EVENT | 2 | 包含 DDL SQL 语句 |
| CREATE_FILE_EVENT | 12 | LOAD DATA 操作 |

## 3.3 控制事件

| 事件类型 | MySQL 值 | 说明 |
|----------|---------|------|
| FORMAT_DESCRIPTION_EVENT | 15 | Binlog 文件头，记录格式信息 |
| ROTATE_EVENT | 4 | Binlog 文件切换 |
| GTID_LOG_EVENT | 33 | GTID 模式专用 |
| ANONYMOUS_GTID_LOG_EVENT | 34 | 匿名 GTID |
| PREVIOUS_GTIDS_LOG_EVENT | 35 | 文件中所有 GTID 集合 |
| XID_EVENT | 16 | 事务提交标识 |
| TABLE_MAP_EVENT | 17 | 表结构映射信息 |
| HEARTBEAT_LOG_EVENT | 27 | 心跳事件（Master 空推送） |

## 3.4 Canal 事件类型映射

Canal 将 MySQL binlog 事件抽象为统一的 Entry 类型：

```java
public enum EntryType {
    TRANSACTIONBEGIN,   // 事务开始
    TRANSACTIONEND,     // 事务结束
    ROWDATA,            // 行数据变更
    HEARTBEAT,          // 心跳
    GTIDLOG,            // GTID 日志
}
```

```java
public enum EventType {
    INSERT,     // 对应 WRITE_ROWS_EVENT
    UPDATE,     // 对应 UPDATE_ROWS_EVENT
    DELETE,     // 对应 DELETE_ROWS_EVENT
    CREATE,     // 对应 DDL CREATE
    ALTER,      // 对应 DDL ALTER
    DROP,       // 对应 DDL DROP
    TRUNCATE,   // 对应 DDL TRUNCATE
    RENAME,     // 对应 DDL RENAME
    CINDEX,     // CREATE INDEX
    DINDEX,     // DROP INDEX
    QUERY,      // 其他 DDL
    ERASE,     // 对应 DDL DROP TABLE
}
```

# 4. Binlog 文件结构

## 4.1 文件命名

```bash
# 查看当前 binlog 文件
mysql> SHOW BINARY LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       154 |
| mysql-bin.000002 |    1048576|
| mysql-bin.000003 |    524288 |
+------------------+-----------+

# 查看当前正在使用的 binlog 文件
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 | 524288   |              |                  |
+------------------+----------+--------------+------------------+
```

Binlog 文件由以下三部分组成：

## 4.2 文件结构图解

```
┌──────────────────────────────────────────────────────────────┐
│  mysql-bin.000001                                            │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Magic Number (4 bytes): 0xFE 0x62 0x69 0x6E          │   │
│  │ (即 "\xFEbin")                                        │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Format Description Event (binlog版本信息)            │   │
│  │  - binlog_version: 4                                  │   │
│  │  - mysql_version: 8.0.33                              │   │
│  │  - event_type_header_length                          │   │
│  │  - checksum_algorithm                                │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Previous GTIDs Event                                  │   │
│  │  (记录前一个binlog文件包含的所有GTID)                 │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ GTID Event (可选的GTID信息)                           │   │
│  │  - source_id: server-uuid                            │   │
│  │  - transaction_id: 1-N                                │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Query Event (事务BEGIN)                               │   │
│  │  - query: "BEGIN"                                    │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Table Map Event (表结构元数据)                       │   │
│  │  - table_id: 108                                     │   │
│  │  - database: "test"                                  │   │
│  │  - table: "users"                                    │   │
│  │  - column_count: 5                                   │   │
│  │  - column_types: [LONG, VARCHAR, ...]                │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Row Event (具体的数据变更)                             │   │
│  │  - type: UPDATE_ROWS_EVENT                            │   │
│  │  - rows: [before_image, after_image] * N             │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ XID Event (事务提交)                                  │   │
│  └───────────────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Rotate Event (下个binlog文件信息)                     │   │
│  │  - next_file: mysql-bin.000002                        │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

## 4.3 Event Header 结构

每个 binlog event 都有一个公共的事件头（Event Header）：

```c
// MySQL Binlog Event Header 结构（v4）
struct EventHeader {
    uint32_t timestamp;      // 事件发生时间戳
    uint8_t  event_type;     // 事件类型（见 2.3 节）
    uint32_t server_id;      // 产生该事件的 Server ID
    uint32_t event_size;     // 事件总长度（包含 header）
    uint32_t log_pos;        // 下一个事件在 binlog 中的位置
    uint16_t flags;          // 事件标志
};
```

## 4.4 使用 mysqlbinlog 查看

```bash
# 使用 mysqlbinlog 工具查看 binlog 内容
mysqlbinlog --verbose --base64-output=DECODE-ROWS mysql-bin.000003

# 输出示例
### UPDATE `test`.`users`
### WHERE
###   @1=101
###   @2='Alice'
###   @3=0
### SET
###   @1=101
###   @2='Alice'
###   @3=1
# at 45678
```

# 5. MySQL 主从复制原理

## 5.1 传统复制流程

MySQL 主从复制是 Canal 工作模式的直接参考对象：

```
┌──────────────┐                        ┌──────────────┐
│    Master    │                        │    Slave     │
│              │                        │              │
│ ┌──────────┐ │   ① Binlog Dump       │ ┌──────────┐ │
│ │ Binlog   │─┼─────────────────────> │ │ I/O      │ │
│ │ Thread   │ │   (push events)       │ │ Thread   │ │
│ └──────────┘ │                        │ └────┬─────┘ │
│              │                        │      │write  │
└──────────────┘                        │ ┌────▼─────┐ │
                                        │ │ Relay    │ │
                                        │ │ Log      │ │
                                        │ └────┬─────┘ │
                                        │      │apply  │
                                        │ ┌────▼─────┐ │
                                        │ │ SQL      │ │
                                        │ │ Thread   │ │
                                        │ └──────────┘ │
                                        └──────────────┘

Canal 模拟的是 Slave 的角色，但只执行 ① 步（接收 binlog），
不会执行 I/O Thread 写入 Relay Log 和 SQL Thread 回放。
```

## 5.2 复制步骤详解

**步骤 ① — Binlog Dump（Master → Slave I/O Thread）**

这是 Canal 完整模拟的部分：

1. Slave 发起连接请求
2. Master 为每个 Slave 创建一个 Binlog Dump Thread
3. Slave 通过 `COM_REGISTER_SLAVE` 命令注册自己
4. Slave 通过 `COM_BINLOG_DUMP` 命令请求 binlog 数据
5. Master 持续推送 binlog event 给 Slave

```
-- Slave 连接时，MySQL Master 可以看到：
mysql> SHOW PROCESSLIST;
+-----+------+-----------+------+-------------+------+------------------------+
| Id  | User | Host      | db   | Command     | Time | State                  |
+-----+------+-----------+------+-------------+------+------------------------+
| 10  | repl | slave:3306| NULL | Binlog Dump | 3600 | Master has sent all... |
| 11  | canl| canal:11111|NULL| Binlog Dump | 3600 | Master has sent all... |
+-----+------+-----------+------+-------------+------+------------------------+
```

**步骤 ② — Write Relay Log（Slave I/O Thread）**

Slave 的 I/O Thread 将收到的 binlog 写入本地 Relay Log：
```
Relay Log 格式与 Binlog 完全一致，只是文件名不同（relay-bin.xxxxxx）
```

**步骤 ③ — Apply Relay Log（Slave SQL Thread）**

Slave 的 SQL Thread 从 Relay Log 读取事件并重放到本地数据库。

## 5.3 复制模式

MySQL 支持三种复制模式：

**1. 基于 Binlog 文件 + 位点的复制**

```sql
-- 在 Slave 上执行
CHANGE MASTER TO
    MASTER_HOST = '192.168.1.100',
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'password',
    MASTER_LOG_FILE = 'mysql-bin.000003',  -- binlog 文件名
    MASTER_LOG_POS = 524288;               -- binlog 位点
```

**问题**：当 Master 发生故障，Slave 需要手动找到新 Master 的 binlog 位点，操作复杂且容易出错。

**2. 基于 GTID 的复制（推荐）**

GTID（Global Transaction Identifier）为每个事务分配全局唯一标识：

```
GTID = server_uuid : transaction_id

示例：3E11FA47-71CA-11E1-9E33-C80AA9429562:1-100
     └─────────── server_uuid ──────────┘ └─ 事务序号 ──┘
```

```sql
-- 在 Slave 上执行（GTID 模式）
CHANGE MASTER TO
    MASTER_HOST = '192.168.1.100',
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'password',
    MASTER_AUTO_POSITION = 1;  -- 使用 GTID 自动定位
```

**GTID 的优势**：
- Slave 切换 Master 时无需指定 binlog 文件+位点
- 自动跳过已执行的事务，避免重复
- 故障恢复更简单可靠

**Canal 对 GTID 的支持**：
- 完全支持 GTID 模式
- 配置 `canal.instance.gtidon=true` 启用
- 在 HA 切换时自动使用 GTID 定位

# 6. Binlog 相关配置

## 6.1 MySQL 配置文件（my.cnf）

```ini
[mysqld]

# ========== Canal 必需的配置 ==========

# 1. Server ID（主从复制中每台机器必须唯一）
server-id = 1

# 2. 开启 binlog
log-bin = mysql-bin

# 3. Binlog 格式必须为 ROW
binlog_format = ROW

# 4. Binlog 行映像模式（推荐 FULL，记录完整的前后镜像）
binlog_row_image = FULL

# ========== 推荐配置 ==========

# 5. 每N次事务提交后刷盘（1 = 每次刷盘，最安全；0 = 由OS控制）
sync_binlog = 1

# 6. 单个 binlog 文件大小
max_binlog_size = 512M

# 7. Binlog 过期时间（秒），0 = 不过期
binlog_expire_logs_seconds = 604800  # 7天

# 8. Binlog 缓存大小
binlog_cache_size = 32768

# 9. 每事务最大 binlog 缓存
max_binlog_cache_size = 4G

# 10. Binlog 校验和（建议开启）
binlog_checksum = CRC32

# 11. GTID 模式（可选但推荐用于高可用）
gtid_mode = ON
enforce_gtid_consistency = ON
```

## 6.2 查看 Binlog 配置

```sql
-- 查看 binlog 相关配置
mysql> SHOW VARIABLES LIKE '%binlog%';
+--------------------------------------------+----------------------+
| Variable_name                              | Value                |
+--------------------------------------------+----------------------+
| binlog_format                              | ROW                  |
| binlog_row_image                           | FULL                 |
| binlog_row_metadata                        | MINIMAL              |
| log_bin                                    | ON                   |
| max_binlog_size                            | 536870912            |
| sync_binlog                                | 1                    |
+--------------------------------------------+----------------------+

-- 查看当前 binlog 状态
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set      |
+------------------+----------+--------------+------------------+------------------------+
| mysql-bin.000003 | 524288   |              |                  | uuid:1-150             |
+------------------+----------+--------------+------------------+------------------------+
```

## 6.3 创建 Canal 用户

Canal 需要一个具有相应权限的 MySQL 用户：

```sql
-- 创建 Canal 专用用户
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal_password';

-- 授予权限
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';

-- 刷新权限
FLUSH PRIVILEGES;

-- 验证权限
SHOW GRANTS FOR 'canal'@'%';
```

**权限说明**：

| 权限 | 用途 |
|------|------|
| `SELECT` | 查询表结构元数据 |
| `REPLICATION SLAVE` | 允许连接到 Master 获取 binlog（核心权限） |
| `REPLICATION CLIENT` | 允许执行 `SHOW MASTER STATUS` 和 `SHOW SLAVE STATUS` |

## 6.4 阿里云 RDS 特别配置

对于阿里云 RDS MySQL，不需要创建 Canal 用户：

```sql
-- 阿里云 RDS 直接使用高权限账号，执行以下命令授予复制权限
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'your_rds_user'@'%';
```

# 7. Binlog 与事务的关系

## 7.1 事务在 Binlog 中的表示

```
一个完整的事务在 Binlog 中由以下 event 组成：

[GTID Event]                 -- GTID 标识（可选）
[QUERY Event: "BEGIN"]       -- 事务开始
[TABLE_MAP Event]            -- 表元数据映射
[ROWS Event 1]               -- 第1个 DML 操作
[ROWS Event 2]               -- 第2个 DML 操作
...
[XID Event]                  -- 事务提交
```

**示例** — 一个 UPDATE 语句影响多行：

```
mysql> BEGIN;
mysql> UPDATE users SET status = 1 WHERE id = 1;
mysql> UPDATE orders SET status = 'PAID' WHERE user_id = 1;
mysql> COMMIT;

Binlog 中记录：
┌─────────────────┐
│ GTID_LOG_EVENT  │  uuid:42
├─────────────────┤
│ QUERY: "BEGIN"  │
├─────────────────┤
│ TABLE_MAP users │
├─────────────────┤
│ UPDATE_ROWS     │  id=1, status: 0->1
├─────────────────┤
│ TABLE_MAP orders│
├─────────────────┤
│ UPDATE_ROWS     │  user_id=1, status: PENDING->PAID
├─────────────────┤
│ XID (COMMIT)    │
└─────────────────┘
```

## 7.2 Binlog 组提交（Group Commit）

MySQL 5.6+ 支持 Binlog 组提交，将多个事务的 Binlog 一次刷盘，提升写入性能：

```
┌──────────────────────────────────────┐
│  事务1 ──┐                           │
│          ├──> 组提交队列              │
│  事务2 ──┤      │                    │
│          │   ┌──▼────┐  fsync        │
│  事务3 ──┘   │ Binlog├────────> 磁盘  │
│              └───────┘               │
│  事务4 ──────────┘                   │
└──────────────────────────────────────┘
```

对于 Canal 而言，组提交意味着可能一次收到多个事务的 event 数据，需要在消费端正确处理事务边界。

# 8. Binlog 相关工具

## 8.1 mysqlbinlog

MySQL 官方提供的 binlog 解析工具：

```bash
# 查看 binlog 文件内容
mysqlbinlog mysql-bin.000003

# 指定位置范围查看
mysqlbinlog --start-position=100 --stop-position=2000 mysql-bin.000003

# 指定时间范围查看
mysqlbinlog --start-datetime="2024-01-01 10:00:00" \
            --stop-datetime="2024-01-01 10:05:00" \
            mysql-bin.000003

# 以 ROW 格式解码显示（显示伪 SQL）
mysqlbinlog --verbose --base64-output=DECODE-ROWS mysql-bin.000003

# 按数据库过滤
mysqlbinlog --database=test_db mysql-bin.000003
```

## 8.2 MySQL Binlog Dump 命令

```sql
-- 手动发起 binlog dump 请求（需要 root 权限）
mysql> SHOW BINLOG EVENTS IN 'mysql-bin.000003' FROM 100 LIMIT 10;
+------------------+-----+----------------+-----------+-------------+-----------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info      |
+------------------+-----+----------------+-----------+-------------+-----------+
| mysql-bin.000003 | 100 | Format_desc    | 1         | 154         | Server... |
| mysql-bin.000003 | 154 | Previous_gtids | 1         | 195         |           |
| mysql-bin.000003 | 195 | Gtid           | 1         | 260         | SET @@... |
| mysql-bin.000003 | 260 | Query          | 1         | 331         | BEGIN     |
| mysql-bin.000003 | 331 | Table_map      | 1         | 385         | test.users|
| mysql-bin.000003 | 385 | Update_rows    | 1         | 456         | table_id..|
| mysql-bin.000003 | 456 | Xid            | 1         | 487         | COMMIT... |
+------------------+-----+----------------+-----------+-------------+-----------+
```

# 9. 为什么 Canal 需要 ROW 格式的 Binlog

Canal 的核心价值在于**精确捕获每一行数据的变更**。只有 ROW 格式的 binlog 才能提供：

1. **字段级变更信息**：知道每一行每个字段的修改前值和修改后值
2. **完整的行镜像**：`binlog_row_image=FULL` 确保获取所有列的值（包括未修改的列）
3. **避免 SQL 解析**：不需要解析复杂的 SQL 语法，直接读取二进制行数据
4. **数据一致性**：ROW 格式天然保证主从一致，不会因 SQL 执行上下文差异导致问题

```java
// Canal 解析 ROW 格式 Binlog 后得到的结构化数据
{
  "database": "test",
  "table": "users",
  "type": "UPDATE",
  "ts": 1672545600000,
  "data": [{
    "id": "1",
    "name": "Alice",
    "status": "1",      // 修改后的值
    "email": "alice@example.com"
  }],
  "old": [{
    "status": "0"       // 修改前的值（仅变更字段，取决于配置）
  }]
}
```

如果使用 STATEMENT 格式，Canal 只能收到 `UPDATE users SET status = 1 WHERE id = 1` 这条 SQL 语句，需要自行做 SQL 解析才能还原出具体变更了什么数据，难度大且不可靠。
