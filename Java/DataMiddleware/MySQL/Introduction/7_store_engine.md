# 1.存储引擎概述

存储引擎（Storage Engine）是 MySQL 中负责**数据的存储和提取**的组件。MySQL 采用**插件式存储引擎架构**——不同的引擎实现了统一的接口，可以灵活切换。

**核心设计思想**：将 SQL 处理层与数据存储层解耦，使 MySQL 能适应不同的业务场景（OLTP、OLAP、归档等）。

```sql
-- 查看 MySQL 支持的存储引擎
SHOW ENGINES;

-- 查看当前默认存储引擎
SHOW VARIABLES LIKE 'default_storage_engine';

-- 查看某张表使用的存储引擎
SHOW TABLE STATUS LIKE 'user';
SHOW CREATE TABLE user;
```

# 2.常用存储引擎对比

## 2.1总览对比

| 特性 | InnoDB | MyISAM | Memory | Archive | CSV |
|------|--------|--------|--------|---------|-----|
| **事务（ACID）** | ✓ | ✗ | ✗ | ✗ | ✗ |
| **行级锁** | ✓ | ✗（表锁） | ✗（表锁） | ✗ | ✗ |
| **外键** | ✓ | ✗ | ✗ | ✗ | ✗ |
| **MVCC** | ✓ | ✗ | ✗ | ✗ | ✗ |
| **崩溃恢复** | ✓（redo log） | ✗（修复表） | ✗（重启丢数据） | ✗ | ✗ |
| **索引结构** | B+Tree（聚簇索引） | B+Tree（非聚簇） | Hash / B+Tree | ✗ | ✗ |
| **全文索引** | ✓（5.6+） | ✓ | ✗ | ✗ | ✗ |
| **数据压缩** | 页压缩 | 压缩表 | ✗ | ✓（高压缩比） | ✗ |
| **存储限制** | 64TB | 256TB（默认） | 受内存限制 | 无 | 受文件系统限制 |
| **适用场景** | 通用 OLTP | 已淘汰（遗留系统） | 临时表、缓存 | 日志归档 | 数据交换 |
| **MySQL 默认** | 5.5 起为默认 | 5.5 之前为默认 | — | — | — |

## 2.2InnoDB——核心引擎

InnoDB 是 MySQL 的默认引擎和**首选引擎**，适用于绝大多数 OLTP 场景。

### 核心特性

- **支持事务（ACID）**：通过 redo log、undo log、锁、MVCC 实现完整的 ACID
- **行级锁**：最小锁粒度，高并发下锁冲突最少
- **MVCC**：非锁定读，读写不互相阻塞
- **聚簇索引**：主键索引的叶子节点存储完整行数据，查询效率高
- **外键支持**：保证参照完整性
- **崩溃恢复**：基于 redo log 实现，系统崩溃后自动恢复
- **热备份**：支持在线热备份（mysqldump --single-transaction 或 XtraBackup）
- **自适应哈希索引**：自动为热点数据建立 hash 索引，加速等值查询
- **Change Buffer**：缓存非唯一二级索引的修改，减少磁盘 IO
- **Buffer Pool**：内存缓冲池，缓存数据和索引，命中率高

### 底层存储结构

InnoDB 数据按**表空间（Tablespace）**→ **段（Segment）**→ **区（Extent，1MB）**→ **页（Page，16KB）**→ **行（Row）**的层级组织，详细参见 [12_InnoDB_engine.md](12_InnoDB_engine.md)。

### 适用场景

- 绝大多数 OLTP 系统（电商、社交、金融等）
- 需要事务支持的业务
- 高并发读写场景
- 需要崩溃恢复能力

### 不适用场景

- 纯只读的海量数据分析（OLAP，可选 ClickHouse 等列存引擎）
- 简单日志记录（可选 Archive 引擎，高压缩比更省空间）

> InnoDB 详细介绍参见 [12_InnoDB_engine.md](12_InnoDB_engine.md)。

## 2.3MyISAM——遗留引擎

MyISAM 是 MySQL 5.5 之前的默认引擎，现已基本被 InnoDB 取代。**新项目不应使用 MyISAM**。

### 核心特性

- **不支持事务**：没有 COMMIT / ROLLBACK
- **表级锁**：读写互斥，并发度低
- **读写并发不阻塞**：写入时会锁全表，读操作被阻塞
- **不支持外键**
- **崩溃后需修复**：非正常关闭后，MyISAM 表可能损坏，需用 `REPAIR TABLE` 修复
- **支持全文索引**：InnoDB 在 5.6 后也支持
- **存储三个文件**：`.frm`（表结构）、`.MYD`（数据）、`.MYI`（索引）

### 性能特点

- **查询快**：在纯 SELECT 场景下，MyISAM 比 InnoDB 稍快（无事务开销）
- **COUNT(*) 极快**：MyISAM 维护了行数计数器（但 WHERE 条件下仍需扫描）
- **压缩表**：`myisampack` 压缩后只读，空间节省 50%~70%

### 遗留使用场景（不推荐）

- 纯只读的历史数据表
- 日志型只需追加的流水表
- 数据仓库的维度表

> **为什么不推荐 MyISAM？**
> 1. 无事务，数据一致性无法保证
> 2. 表级锁，并发写入性能极差
> 3. 崩溃恢复能力弱，数据容易损坏
> 4. InnoDB 已全面超越 MyISAM，包括全文索引（5.6+）、空间索引（5.7+）
> 5. MySQL 8.0 已不再为 MyISAM 开发新功能

## 2.4Memory（HEAP）——内存引擎

数据全部存储在内存中，重启后数据丢失。

### 核心特性

- **极快读写**：数据在内存，无磁盘 IO
- **只支持表级锁**
- **默认使用 HASH 索引**：等值查询极快；范围查询不如 B+Tree
- **VARCHAR 字段实际存储为 CHAR**（浪费空间）
- **不支持 TEXT / BLOB 类型**

### 适用场景

- 临时表（`CREATE TEMPORARY TABLE ... ENGINE=Memory`）
- 会话状态缓存（如在线用户、验证码）
- 缓存查询结果（但现在更多用 Redis）

### 注意事项

- 重启后数据丢失
- 单个表最大 16MB（受 `max_heap_table_size` 限制）
- 不适合存储重要数据

## 2.5Archive——归档引擎

专为数据归档和日志存储设计的引擎。

### 核心特性

- **高压缩比**：通过 zlib 压缩，通常节省 50%~80% 空间
- **仅支持 INSERT 和 SELECT**：不支持 UPDATE、DELETE、索引
- **行级锁**：INSERT 时只锁插入行
- **无事务**

### 适用场景

- 日志流水归档（访问日志、操作日志）
- 审计数据（几乎不修改，偶尔查询）

## 2.6CSV——数据交换引擎

将表数据存储为 CSV 文件。

### 核心特性

- 数据以**纯文本 CSV** 格式存储，可直接用文本编辑器查看或 Excel 打开
- 支持 SELECT、INSERT
- 不支持索引、事务、NULL（所有列 NOT NULL）

### 适用场景

- MySQL 与 Excel / 其他程序之间的数据导入导出
- 元数据配置（简单的 CSV 配置表）

## 2.7其他引擎简介

| 引擎 | 说明 |
|------|------|
| **NDB / NDBCLUSTER** | MySQL Cluster 分布式集群引擎，数据自动分片，高可用 |
| **Blackhole** | 黑洞引擎，写入的数据被丢弃（类似 /dev/null），binlog 正常记录，用于中继复制 |
| **Federated** | 联邦引擎，访问远程 MySQL 服务器上的表（类似 Oracle 的 dblink），已废弃 |
| **Merge** | 将多个 MyISAM 表合并为一，用于分表场景（InnoDB 不适用） |
| **TokuDB** | 第三方引擎，高压缩比 + Fractal Tree 索引，适合大数据写入（已被 Percona 收购，8.0 不支持） |

# 3.存储引擎的架构原理

## 3.1插件式架构

MySQL 通过定义统一的**Handler API**接口（在 `sql/handler.h` 中定义），使不同存储引擎可以实现以下标准接口：

- `write_row()`、`update_row()`、`delete_row()`：行操作
- `index_read()`、`index_next()`：索引读取
- `rnd_init()`、`rnd_next()`：全表扫描

SQL 层不需要关心底层是哪种引擎，只需调用 Handler API。这就是"插件式架构"的核心——**面向接口编程**。

## 3.2如何选择存储引擎

| 场景 | 推荐引擎 |
|------|---------|
| 需要事务、高并发 | **InnoDB** |
| 纯查询、不需要事务 | InnoDB（MyISAM 已不推荐） |
| 临时表 / 缓存 | **Memory** |
| 日志归档 | **Archive** |
| 数据交换 | **CSV** |

> **核心原则**：除非有明确理由，**一律使用 InnoDB**。InnoDB 是唯一支持事务的引擎，也是 Oracle/MySQL 官方未来持续投入的核心引擎。

# 4.存储引擎相关SQL

```sql
-- 建表时指定存储引擎
CREATE TABLE user (
  id BIGINT PRIMARY KEY
) ENGINE=InnoDB;

-- 修改表的存储引擎（会锁表并重建！大表需使用 pt-online-schema-change）
ALTER TABLE user ENGINE=InnoDB;

-- 修改默认存储引擎（永久生效需配置 my.cnf）
SET default_storage_engine=InnoDB;

-- 批量生成 ALTER 语句（将指定库中所有表转为 InnoDB）
SELECT CONCAT('ALTER TABLE ', table_name, ' ENGINE=InnoDB;')
FROM information_schema.tables
WHERE table_schema = 'mydb' AND engine != 'InnoDB';
```
