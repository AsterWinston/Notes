# 1. InnoDB逻辑存储结构

InnoDB 按以下层级组织数据存储：

| 层级 | 名称 | 说明 |
|------|------|------|
| **表空间（Tablespace）** | 最高层 | 一个 MySQL 实例可有多个表空间，存储记录、索引等数据 |
| **段（Segment）** | 逻辑分区 | 数据段（B+Tree 叶子节点）、索引段（非叶子节点）、回滚段 |
| **区（Extent）** | 分配单元 | 固定 1MB（64 个连续页，每页 16KB），保证页的连续性 |
| **页（Page）** | 最小 IO 单元 | 默认 16KB，每次从磁盘申请 4~5 个区（保证页连续性） |
| **行（Row）** | 最小存储单元 | 每行数据按行格式（COMPACT / DYNAMIC 等）存储 |

```
表空间 (Tablespace)
  └── 段 (Segment)
        └── 区 (Extent) = 64 × 页
              └── 页 (Page) = 16KB
                    └── 行 (Row)
```

> **为什么需要"区"这一级？**
> 如果每次按页分配（零散分配），多次分配后数据页在磁盘上会变得碎片化（不连续），导致随机 IO 增加。按区（1MB 连续空间）预分配，可保证 B+Tree 叶子节点在磁盘上尽可能连续，提升顺序 IO 性能。

# 2. InnoDB架构

InnoDB 架构分为**内存结构**和**磁盘结构**两大部分，两者通过后台线程协调工作。

## 2.1 内存架构

### 2.1.1 Buffer Pool（缓冲池）

缓冲池是 InnoDB 中**最重要的内存区域**，缓存磁盘上的数据页和索引页，减少磁盘 IO。

**工作机制**：
1. 执行查询时，先检查 Buffer Pool 中是否有目标数据页
2. 若有（命中），直接从内存读取，性能极高
3. 若无（未命中），从磁盘加载页到 Buffer Pool，再读取
4. 修改数据时，先修改 Buffer Pool 中的页，标记为"脏页（Dirty Page）"，随后异步刷新到磁盘

**页管理**：通过链表管理三种状态的页：

| 页类型 | 说明 |
|--------|------|
| **Free Page** | 空闲页，未被使用 |
| **Clean Page** | 已使用，数据与磁盘一致 |
| **Dirty Page** | 脏页，数据已被修改但未刷入磁盘 |

**Buffer Pool 大小建议**：
- 专用数据库服务器：物理内存的 **50%~70%**
- 共享服务器：物理内存的 25%~40%
- 通过 `innodb_buffer_pool_size` 设置（MySQL 8.0 支持在线调整）

```sql
-- 查看当前大小
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- 查看命中率（越高越好，理想 > 99%）
SHOW STATUS LIKE 'innodb_buffer_pool_read%';
-- (Innodb_buffer_pool_read_requests - Innodb_buffer_pool_reads) / Innodb_buffer_pool_read_requests
```

### 2.1.2 Change Buffer（变更缓冲区）

Change Buffer 缓存**非唯一二级索引**的修改操作（INSERT、UPDATE、DELETE），延迟合并，减少磁盘随机 IO。

**工作原理**：
1. 当修改非唯一二级索引页时，若该索引页**不在 Buffer Pool 中**
2. 不直接从磁盘加载该页，而是将修改缓存在 Change Buffer 中
3. 当该索引页后续被读取或 MySQL 空闲时，将 Change Buffer 中的修改合并到索引页

**为什么只针对非唯一二级索引？**
唯一索引的插入需要检查唯一性，必须读取索引页才能判断，无法延迟。

```sql
-- 查看 Change Buffer 状态
SHOW VARIABLES LIKE 'innodb_change_buffering';
-- 可选值：inserts / deletes / purges / changes（推荐 changes = 全部） / none / all
```

> Change Buffer 对于**写密集型**且**二级索引较多**的场景效果显著（如日志系统的索引）。读密集型场景效果不大。

### 2.1.3 Adaptive Hash Index（自适应哈希索引）

InnoDB 自动监控 B+Tree 索引页的查询模式，如果发现某些数据被频繁等值查询，会在 Buffer Pool 中为热点数据建立 Hash 索引，加速等值查询。

- 完全自动，无需人工干预
- 可开关：`SET GLOBAL innodb_adaptive_hash_index = ON/OFF;`
- 在高并发等值查询场景效果好（如 `WHERE id = xxx`）

### 2.1.4 Log Buffer（日志缓冲区）

缓存要写入磁盘的 redo log 数据，默认 16MB。大事务（批量 INSERT/UPDATE）时增加日志缓冲区可减少磁盘 IO。

```sql
SHOW VARIABLES LIKE 'innodb_log_buffer_size';  -- 缓冲区大小
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';  -- 刷盘策略
```

## 2.2 磁盘结构

### 2.2.1 System Tablespace（系统表空间）

存储 Change Buffer 的数据。可通过 `innodb_data_file_path` 配置：

```
[mysqld]
innodb_data_file_path = ibdata1:128M:autoextend:max:10G
```

> MySQL 5.x 中系统表空间还包含 InnoDB 数据字典和 undo log，8.0 中这些已移到独立表空间。

### 2.2.2 File-Per-Table Tablespaces（独立表空间）

每张 InnoDB 表对应一个 `.ibd` 文件，包含该表的数据和索引。通过 `innodb_file_per_table` 控制（默认 ON，推荐）。

**优点**：
- 单表空间可以独立管理（`ALTER TABLE ... DISCARD/IMPORT TABLESPACE`）
- TRUNCATE / DROP 释放空间
- 便于运维：可单独备份、恢复一张表

### 2.2.3 General Tablespaces（通用表空间）

通过 `CREATE TABLESPACE` 创建，可指定多个表存储在同一表空间中：

```sql
CREATE TABLESPACE ts1 ADD DATAFILE 'ts1.ibd' ENGINE=InnoDB;
CREATE TABLE t1 (id INT) TABLESPACE ts1;
```

适合需要将多张相关表放置在同一物理文件的场景。

### 2.2.4 Undo Tablespaces（撤销表空间）

存储 undo log（回滚日志），用于事务回滚和 MVCC。8.0 默认自动创建 2 个 undo 表空间（每个 16MB）。

```sql
SHOW VARIABLES LIKE 'innodb_undo_tablespaces';
```

### 2.2.5 Temporary Tablespaces（临时表空间）

存储用户创建的**临时表**及内部排序时产生的**临时数据**。

### 2.2.6 Doublewrite Buffer（双写缓冲区）

双写机制是 InnoDB 防止**部分页写入（Partial Page Write）**导致数据损坏的重要特性。

**场景**：InnoDB 的一个页是 16KB，但操作系统和磁盘的原子写入单位通常是 4KB（一个扇区）。如果写入过程中断电，可能导致页中只有部分数据被写入（部分写），该页损坏且无法恢复。

**方案**：
1. 脏页刷盘前，先将页的完整副本写入双写缓冲区（磁盘上的连续区域，`ib_16384_0.dblwr`）
2. 双写缓冲区写入完成后，再将页写入实际数据文件
3. 如果步骤 2 发生崩溃，恢复时从双写缓冲区获取完整副本修复

> 双写缓冲区有性能开销（每次刷页多一次写操作）。但 SSD 的原子写能力提升后，部分文件系统（如 ext4 + `data=ordered`）已能保证 16KB 原子写入。

### 2.2.7 Redo Log（重做日志）

Redo Log 记录事务对数据页的**物理修改**，是实现**持久性（Durability）**的关键。

```sql
SHOW VARIABLES LIKE 'innodb_log_file_size';    -- 每个日志文件大小
SHOW VARIABLES LIKE 'innodb_log_files_in_group';  -- 日志文件数量
```

> 详细原理见[第 3 节事务管理](#3-事务管理)。

## 2.3 后台线程

| 线程 | 职责 |
|------|------|
| **Master Thread** | 核心调度线程，负责脏页刷新、合并插入缓存、undo 页回收等 |
| **IO Thread** | 使用异步 IO（AIO）处理 IO 请求回调：4 个 Read、4 个 Write、1 个 Log、1 个 Insert Buffer |
| **Purge Thread** | 清理已提交事务不再需要的 undo log |
| **Page Cleaner Thread** | 协助 Master Thread 刷新脏页，减轻阻塞 |

```sql
-- 查看 InnoDB 引擎状态（包括线程信息）
SHOW ENGINE INNODB STATUS\G;
```

# 3. 事务管理

InnoDB 通过以下机制实现 ACID：

| 特性 | 实现机制 |
|------|---------|
| **原子性（Atomicity）** | undo log（回滚日志） |
| **一致性（Consistency）** | redo log + undo log + 锁（三者共同保证） |
| **隔离性（Isolation）** | MVCC + 锁（行锁、间隙锁、临键锁） |
| **持久性（Durability）** | redo log（重做日志） |

## 3.1 Redo Log

### 3.1.1 WAL（Write-Ahead Log）机制

InnoDB 对数据页的修改遵循"先写日志，再写磁盘"的 WAL 原则：

1. 事务修改 Buffer Pool 中的数据页
2. 在修改数据页之前，先将修改记录写入 Log Buffer（内存中的 redo log 缓冲区）
3. 事务提交时，Log Buffer 中的 redo log 刷入磁盘
4. 被修改的数据页（脏页）随后异步刷新到磁盘

**为什么需要 WAL？**
- 顺序写 Log 比随机写数据页快几个数量级
- 崩溃恢复时，通过 redo log 重放已提交但未刷盘的事务，保证持久性

### 3.1.2 刷盘策略（innodb_flush_log_at_trx_commit）

| 值 | 策略 | 性能 | 安全性 |
|----|------|------|--------|
| **0** | 每秒将 Log Buffer 写入磁盘并 fsync | 最快 | MySQL 崩溃可能丢失 1 秒数据 |
| **1**（默认） | 每次事务提交都写入磁盘并 fsync | 最慢 | 最安全，不丢数据 |
| **2** | 每次事务提交写 Log 到 OS 缓存，每秒 fsync | 中等 | MySQL 崩溃不丢数据，OS 崩溃丢 1 秒 |

```sql
-- 查看当前配置
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
```

> **推荐**：金融等高安全性系统使用 `1`；互联网系统可在主库使用 `1`，从库使用 `2` 以提高复制性能。

### 3.1.3 逻辑结构

Redo Log 是**循环写入**的固定大小文件组。当写满时触发 Checkpoint，将脏页刷新到磁盘，释放 redo log 空间。

```
[ ib_logfile0 ] → [ ib_logfile1 ] → [ ib_logfile0 ] → ...
```

## 3.2 Undo Log

Undo Log 记录数据**修改前**的逻辑信息，实现**原子性**和**MVCC**。

### 3.2.1 作用

1. **事务回滚**：回滚时根据 undo log 将数据恢复到修改前的状态（与 redo log 的物理记录不同，undo log 是逻辑记录）
   - INSERT → undo log 记录 DELETE
   - DELETE → undo log 记录 INSERT
   - UPDATE → undo log 记录反向 UPDATE
2. **MVCC 快照读**：通过 undo log 版本链，快照读能够读取到历史版本的数据

### 3.2.2 Undo Log 版本链

```
┌──────────────────────────────────────────────────┐
│ 当前行: name='C' (trx_id=300)                     │
│   DB_ROLL_PTR ──────────────────────────┐         │
├─────────────────────────────────────────┤         │
│ 上一版本: name='B' (trx_id=200)          │         │
│   DB_ROLL_PTR ──────────────────┐       │ ←───────┘
├─────────────────────────────────┤       │
│ 更早版本: name='A' (trx_id=100)  │       │ ←───────┘
│   DB_ROLL_PTR = NULL             │
└─────────────────────────────────┘
```

不同事务或相同事务对同一行记录进行修改，undo log 以链表串联各版本。最新版本在链头，最旧版本在链尾。

# 4. MVCC（多版本并发控制）

## 4.1 当前读与快照读

| 读类型 | 读取内容 | 是否加锁 | 示例 |
|--------|---------|---------|------|
| **快照读（Snapshot Read）** | 数据的某个历史版本 | 不加锁 | 普通 `SELECT` |
| **当前读（Current Read）** | 数据的最新版本 | 加锁 | `SELECT ... FOR UPDATE`、`INSERT`、`UPDATE`、`DELETE` |

```
快照读 = 无锁非阻塞，基于 MVCC 的 ReadView + undo log 版本链
当前读 = 保证读到最新数据并防止并发修改
```

**不同隔离级别下的快照读行为**：

| 隔离级别 | 快照读生成时机 |
|---------|--------------|
| **READ COMMITTED** | 每次 SELECT 都生成新的 ReadView |
| **REPEATABLE READ** | 只在事务中第一次 SELECT 时生成 ReadView，后续复用 |
| **SERIALIZABLE** | 快照读退化为当前读（自动加 `LOCK IN SHARE MODE`） |

## 4.2 MVCC实现要素

### 4.2.1 三个隐藏字段

InnoDB 每行数据有三个隐藏字段（你可能看不到，但它们始终存在）：

| 字段 | 长度 | 说明 |
|------|------|------|
| **DB_TRX_ID** | 6 字节 | 最近修改（INSERT / UPDATE）该行的事务 ID |
| **DB_ROLL_PTR** | 7 字节 | 回滚指针，指向 undo log 中的上一版本（版本链指针） |
| **DB_ROW_ID** | 6 字节 | 隐藏主键，当没有显式主键也没有非空唯一索引时自动生成 |

```sql
-- 不可见，但可通过间接方式感知其存在
-- 如：SELECT * FROM table WHERE DB_TRX_ID > xxx;  -- 无法直接查询
```

### 4.2.2 Undo Log 版本链

每次修改一行数据，undo log 中会生成一条记录，通过 `DB_ROLL_PTR` 串联成版本链。版本链记录了该行数据的完整演变历史。

### 4.2.3 ReadView（读视图）

ReadView 是快照读 SQL 执行时 MVCC 提取数据的依据，记录并维护系统当前**活跃的（未提交的）事务 ID**。

**四个核心字段**：

| 字段 | 说明 |
|------|------|
| `m_ids` | 当前活跃事务（未提交）的事务 ID 集合 |
| `min_trx_id` | `m_ids` 中最小的事务 ID |
| `max_trx_id` | 预分配的下一个事务 ID（当前最大事务 ID + 1） |
| `creator_trx_id` | 创建该 ReadView 的事务 ID |

**版本可见性判断规则**：

对于 undo log 版本链中的某个版本，其事务 ID 记为 `trx_id`：

| 条件 | 含义 | 可见性 |
|------|------|--------|
| `trx_id == creator_trx_id` | 当前事务自己修改的 | ✓ **可见** |
| `trx_id < min_trx_id` | 生成 ReadView 前已提交 | ✓ **可见** |
| `trx_id >= max_trx_id` | 生成 ReadView 后才开始的事务 | ✗ **不可见** |
| `min_trx_id <= trx_id < max_trx_id` | 在 ReadView 创建时可能活跃 | 若 `trx_id` 在 `m_ids` 中 → 未提交（不可见）；不在 → 已提交（可见） |

**版本链遍历过程**：
1. 从当前最新版本开始，逐一检查版本链中每一行的 `DB_TRX_ID`
2. 找到第一个满足可见性条件的版本，作为快照读的结果返回
3. 如果遍历到链尾（最旧版本），说明所有版本都不可见，返回空

## 4.3 RC vs RR 的 MVCC 差异

同一行数据，同一时刻，RC 和 RR 可能读到不同结果，原因在于 ReadView 的**生成时机不同**：

### 4.3.1 READ COMMITTED

**每次 SELECT 都生成新的 ReadView**，所以能读到其他事务已提交的修改。

```
事务 A：                        事务 B：
START TRANSACTION;              START TRANSACTION;
SELECT name → '张三'(快照1)      UPDATE SET name='李四';
                                COMMIT;
SELECT name → '李四'(快照2)      -- 生成新 ReadView，看到 B 的提交
COMMIT;
```

### 4.3.2 REPEATABLE READ

**仅在第一次 SELECT 时生成 ReadView，后续复用**，所以保证可重复读。

```
事务 A：                        事务 B：
START TRANSACTION;              START TRANSACTION;
SELECT name → '张三'(快照唯一)    UPDATE SET name='李四';
                                COMMIT;
SELECT name → '张三'(复用快照)    -- 不生成新 ReadView，仍看到旧数据
COMMIT;
```
