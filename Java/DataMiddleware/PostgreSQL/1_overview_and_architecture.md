# 1. PostgreSQL 的定位

PostgreSQL（常简称 PG）是一个**开源的对象-关系型数据库管理系统**，起源于 1986 年加州大学伯克利分校的 POSTGRES 项目，1996 年正式更名为 PostgreSQL。它由全球社区维护，不受任何单一商业公司控制，许可证为宽松的 PostgreSQL License（类 BSD），允许闭源商用、修改和再分发。

在很多技术选型的讨论里，PG 的标签是"**功能完整、贴近 SQL 标准、扩展性强**"。它把一些在其他数据库中属于"企业版增值项"的能力直接放进了核心：事务性 DDL、窗口函数、通用表表达式（CTE）、JSONB、数组、范围类型、全文检索、物化视图、逻辑复制、并行查询、外部数据包装器（FDW）。这也意味着它的学习曲线比 MySQL 稍陡，很多能力需要主动了解才会用。

## 1.1 版本与生态

PostgreSQL 每年发布一个大版本，社区通常维护最近 5 个主版本。写作时间点（2026 年）下生产环境常见的是 14 到 18。

| 版本 | 发布时间 | 值得关注的变化 |
|------|---------|---------------|
| 13 | 2020 | 索引去重、并行 VACUUM、可信扩展（trusted extension） |
| 14 | 2021 | 多范围类型、lz4 TOAST 压缩、libpq 管道模式 |
| 15 | 2022 | MERGE、统计信息移入共享内存、`NULLS NOT DISTINCT`、public schema 权限收紧 |
| 16 | 2023 | 逻辑复制支持从备库发布、并行查询改进 |
| 17 | 2024 | `JSON_TABLE`、增量物理备份、流式 I/O、`MAINTAIN` 权限 |
| 18 | 2025 | 异步 I/O 子系统、uuidv7、虚拟生成列、OAuth 2.0 认证 |

选择版本的原则很简单：**新项目直接上当前稳定版，老项目至少保证还在社区支持周期内**。PG 的小版本升级（如 17.2 到 17.5）只修 bug 和安全问题，二进制兼容，通常只需替换二进制并重启，风险很低，建议跟随。

## 1.2 最小可运行示例

先用 Docker 起一个实例，确认整条链路可用：

```bash
# 启动 PostgreSQL 17，数据卷持久化，端口映射到宿主机
docker run -d --name pg-dev \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:17
```

```bash
# 进入容器内的 psql 交互终端
docker exec -it pg-dev psql -U postgres
```

```sql
-- 建表、写入、查询，验证实例、权限、存储都正常
CREATE TABLE demo_user (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL
);
INSERT INTO demo_user (name) VALUES ('alice'), ('bob');
SELECT * FROM demo_user ORDER BY id;
```

如果这三步都能跑通，说明实例已经可用。后面所有机制讨论，最终都要落到这类可观测的行为上。

## 1.3 与 MySQL 的核心差异

| 维度 | PostgreSQL | MySQL（InnoDB） |
|------|-----------|----------------|
| 连接模型 | 一连接一**进程**（process-per-connection） | 一连接一**线程**（thread-per-connection） |
| 默认隔离级别 | READ COMMITTED | REPEATABLE READ |
| MVCC 实现 | 堆表中保留多版本元组，旧版本由 VACUUM 清理 | undo log 版本链 + 回滚指针 |
| 索引组织 | 所有索引都是**二级索引**，指向堆表 ctid，无聚簇索引 | 主键即聚簇索引，二级索引存主键值 |
| 事务性 DDL | 完全支持，`ROLLBACK` 可撤销 `CREATE/ALTER/DROP` | 8.0 起原子 DDL，但语义仍不等同 |
| 复制基础 | 物理 WAL 流复制 + 逻辑复制（解码 WAL） | binlog 复制 |
| Schema 概念 | 数据库下的命名空间，一个库可有多 schema | Schema 等同于 Database |
| 标识符大小写 | 不加引号统一折叠为**小写** | 取决于 `lower_case_table_names` 与平台 |
| 扩展能力 | 原生 `CREATE EXTENSION` 加载插件 | 插件体系受限，多数靠官方分支 |
| 数据类型 | 数组、范围、JSONB、网络、几何、自定义类型 | JSON、枚举、空间类型（有限） |
| 计数与统计 | 统计信息收集器、`pg_stat_*` 视图丰富 | `SHOW STATUS`、`performance_schema` |

**这些差异会直接改变应用写法**。例如 PG 里一张表被频繁 UPDATE，磁盘上会留下大量死元组，必须依赖 autovacuum 回收；而 MySQL 的 undo 由 purge 线程处理，表现形态不同。再如 PG 没有聚簇索引，主键查询也要走"索引 → 堆表"两步，因此 `SELECT *` 的代价结构与 MySQL 不一样。

> **行业实践**：从 MySQL 迁移到 PG 时，最容易踩的三个坑是——大小写敏感（未加引号的 `User` 会变成 `user`）、`GROUP BY` 严格性（PG 要求非聚合列都出现在 `GROUP BY` 中）、以及 UPDATE 密集表必须调优 autovacuum。

# 2. 进程架构

PostgreSQL 采用**多进程**模型，而不是多线程。主进程 `postmaster` 负责监听端口、接受连接、派生后端进程，并在子进程异常退出时负责清理和整体重启。这种设计的好处是进程间隔离彻底，单个后端崩溃通常不会拖垮整个实例；代价是每个连接都有独立地址空间，内存开销比线程模型大。

```text
                     ┌──────────────────────────────┐
                     │        postmaster            │
                     │  监听 5432、派生/回收子进程   │
                     └──────────────┬───────────────┘
        ┌───────────────┬───────────┼───────────┬───────────────┐
        ▼               ▼           ▼           ▼               ▼
 ┌────────────┐  ┌────────────┐  ┌────────┐  ┌──────────┐  ┌──────────┐
 │ backend 1  │  │ backend 2  │  │ bgwriter│  │checkpointer│ │ walwriter│
 │ 客户端会话  │  │ 客户端会话  │  │ 后台写盘 │  │  检查点   │  │  WAL 刷盘 │
 └────────────┘  └────────────┘  └────────┘  └──────────┘  └──────────┘
        │               │
        └───────┬───────┘
                ▼
      ┌───────────────────┐        ┌────────────────────┐
      │   共享内存         │◀──────▶│  数据文件 / WAL 文件 │
      │ shared_buffers 等  │        │  PGDATA            │
      └───────────────────┘        └────────────────────┘
```

## 2.1 postmaster 的职责

`postmaster` 本身不执行 SQL。它做四件事：

- **监听连接**：绑定 `listen_addresses` 与 `port`，接受 TCP 或 Unix socket 连接。
- **认证分流**：把连接交给新的 backend，由 backend 读取 `pg_hba.conf` 完成认证。
- **进程管理**：某个 backend 因段错误等原因崩溃时，postmaster 会终止所有其他进程并进入恢复流程，保证共享内存不被污染。
- **信号处理**：接收 `SIGHUP` 重读配置、`SIGTERM` 智能关闭、`SIGQUIT` 立即关闭。

这也解释了一个常见现象：**PG 无法只重启某一个后端**。任何后端崩溃都会导致整个实例重启并做崩溃恢复，所以扩展或自定义函数里的段错误杀伤力很大。

## 2.2 后端进程（backend）

每个客户端连接对应一个 backend 进程，生命周期与连接一致。它负责解析 SQL、生成计划、执行、返回结果，并持有会话级的私有内存（排序区、临时表缓存等）。连接断开后进程退出，私有内存随之释放。

因为进程比线程重，PG 的**连接数不能像 MySQL 那样随便开到几千**。每个连接即使空闲也要占用几 MB 到几十 MB 内存，且进程切换成本高。生产环境的正确做法是前置连接池（PgBouncer 或应用侧 HikariCP），把真实连接数控制在 `max_connections` 的合理范围内。

## 2.3 辅助进程

| 进程 | 作用 | 关键配置 |
|------|------|---------|
| `checkpointer` | 周期性地把脏页刷盘并记录检查点，缩短崩溃恢复时间 | `checkpoint_timeout`、`max_wal_size` |
| `bgwriter` | 平时渐进地把共享缓冲区脏页写出，平滑 I/O | `bgwriter_delay`、`bgwriter_lru_maxpages` |
| `walwriter` | 异步把 WAL 缓冲区刷到 `pg_wal` | `wal_writer_delay` |
| `autovacuum launcher` | 调度 autovacuum worker 回收死元组、更新统计信息 | `autovacuum_*` |
| `archiver` | 把已完成的 WAL 段复制到归档目录，支撑 PITR | `archive_mode`、`archive_command` |
| `walsender` | 向备库或逻辑订阅端发送 WAL | `max_wal_senders` |
| `walreceiver` | 备库侧接收 WAL（备库上的进程） | — |
| `logical replication launcher` | 启动逻辑复制 worker | `max_logical_replication_workers` |
| `parallel worker` | 并行查询的工作进程 | `max_parallel_workers` |

这些进程的名字可以直接在 `ps` 或 `pg_stat_activity` 的 `backend_type` 字段里看到。排查"CPU 为什么高"时，先分清是业务 backend、autovacuum 还是 checkpointer，方向完全不同。

# 3. 共享内存

共享内存是 PG 所有 backend 交换数据的地方。它由 postmaster 在启动时一次性申请，大小在运行期不能动态扩展（少数参数可以，但 `shared_buffers` 不行）。

## 3.1 shared_buffers

**`shared_buffers` 是 PG 自己的页缓存**，缓存 8KB 的数据页。读写路径是：backend 先在共享缓冲区里找页，找不到再从磁盘读入；修改也在缓冲区里完成，脏页由 bgwriter 或 checkpointer 刷盘。

经典建议是设置为物理内存的 **25%**。这个数字不是教条，而是因为 PG 还依赖操作系统的页缓存（page cache）做第二层缓存。设得过大反而会挤占 OS 缓存，让 WAL 读写、临时文件受影响。云数据库通常由厂商预调，不要盲目照搬本地经验。

```sql
-- 查看当前值（单位是 8KB 页）
SHOW shared_buffers;

-- 查看缓冲区命中率，OLTP 系统通常应高于 99%
SELECT
    sum(blks_hit) * 100.0 / nullif(sum(blks_hit + blks_read), 0) AS hit_ratio
FROM pg_stat_database;
```

## 3.2 WAL 缓冲区与其他区域

| 区域 | 作用 | 相关参数 |
|------|------|---------|
| WAL buffer | 暂存待写入 WAL 的记录，由 walwriter 刷盘 | `wal_buffers`（默认 -1，自动取 shared_buffers 的 1/32，上限 16MB） |
| CLOG / pg_xact buffer | 缓存事务提交状态 | 由 PG 内部管理 |
| Lock table | 存放锁对象与等待队列 | `max_locks_per_transaction` |
| Shared catalog | 系统目录的共享缓存 | 随连接数增长 |
| Statistics | PG 15 起统计信息放在共享内存 | `pg_stat_*` 的数据来源 |

> **注意**：`shared_buffers` 修改后必须**重启**实例，`SIGHUP` 不够。改之前先确认机器内存、连接数和 OS 缓存余量，否则容易触发 OOM Killer。

## 3.3 每个 backend 的私有内存

与共享内存相对的是会话私有内存，主要受这些参数控制：

- **`work_mem`**：单个排序、哈希、聚合操作可用的内存。注意它是"每个操作"而非"每个连接"，一条 SQL 有多个排序节点就会用多份。这是最容易被打爆的参数。
- **`maintenance_work_mem`**：VACUUM、CREATE INDEX、ALTER TABLE 等维护操作的内存，全局有上限但单进程可用较大值。
- **`temp_buffers`**：会话访问临时表时的私有缓冲。
- **`catalog_cache`**：系统目录缓存，随访问的表数量增长。

一个粗略的估算：`max_connections × (work_mem × 并发排序数 + 基础开销)` 才是峰值内存。很多"PG 内存突然涨满"的事故，都是把 `work_mem` 当成每连接上限来算导致的。

# 4. WAL 与崩溃恢复

## 4.1 为什么需要 WAL

PG 修改数据时并不是直接把 8KB 页写回磁盘，而是遵循 **WAL（Write-Ahead Logging，预写式日志）**：

1. 在共享缓冲区中修改数据页；
2. 生成描述这次修改的 WAL 记录，写入 WAL buffer；
3. 事务提交时，确保该事务的 WAL 记录已经 `fsync` 到磁盘；
4. 数据页在之后的某个时刻由后台进程异步刷盘。

这样做的核心收益是**把随机写变成顺序写**。提交时只需保证几字节到几 KB 的 WAL 落盘，而不必等待分散在磁盘各处的数据页。崩溃后，用 WAL 重放"已提交但数据页未刷盘"的修改，就能恢复到一致状态。

## 4.2 检查点与全页写

检查点（checkpoint）的作用是记录一个"从此刻起，之前的修改都已经落盘"的位置，从而让崩溃恢复只需从最近的检查点开始重放。检查点越频繁，恢复越快，但 I/O 压力越大；间隔越长，恢复越慢。

这里有一个关键机制：**全页写（full page write）**。检查点之后，某个数据页第一次被修改时，PG 会把**整个 8KB 页**的镜像写进 WAL。原因是操作系统写磁盘时可能只写了半个页（torn page），如果只记录行级变更，恢复时基于一个残缺的页重放会得到错误结果。全页写用空间换正确性，`full_page_writes` 默认开启，不建议关闭。

## 4.3 LSN 与 WAL 文件

每条 WAL 记录都有一个 **LSN（Log Sequence Number）**，本质是字节偏移量，格式如 `0/16B3748`。它单调递增，可以用来：

- 判断主备延迟（备库重放到的 LSN 与主库当前 LSN 的差距）；
- 指定 PITR 的恢复目标；
- 在 `pg_stat_replication` 中观察复制进度。

WAL 文件存放在 `PGDATA/pg_wal`，默认每段 16MB。文件数量由 `min_wal_size` 和 `max_wal_size` 控制，不是越多越好——WAL 堆积往往意味着归档失败或复制槽未推进。

# 5. 一条查询的生命周期

```text
客户端
  │  1. TCP 连接 + 启动包
  ▼
postmaster ──派生──▶ backend
  │  2. 读取 pg_hba.conf 认证
  ▼
Parser（词法/语法分析）──▶ 原始语法树
  ▼
Analyzer（语义分析，查系统目录）──▶ 查询树
  ▼
Rewriter（展开视图、规则）
  ▼
Planner / Optimizer（基于代价选择路径）──▶ 执行计划
  ▼
Executor（按节点拉取元组）
  │  3. 读：Buffer Manager 命中 shared_buffers 或读盘
  │     写：修改缓冲区 + 生成 WAL
  ▼
返回结果集（libpq 协议）
```

几个容易忽略的点：

- **解析与计划都发生在 backend 内**，没有 MySQL 那种全局查询缓存。PG 的计划缓存是按预备语句（prepared statement）做的，通用计划与定制计划会随着执行次数切换。
- **权限检查在分析阶段完成**，所以一条 SQL 引用了几十张表，仅解析就会产生大量系统目录访问。
- **执行器是 Volcano 模型**（逐行拉取），PG 11 之后加入了 JIT 编译和并行执行来缓解大查询的 CPU 开销。

# 6. 常见误区与排错

**误区一：把 PG 当成"另一个 MySQL"来调参。** `shared_buffers`、`work_mem`、`max_connections` 三个参数的含义与 MySQL 的 `innodb_buffer_pool_size`、`sort_buffer_size`、`max_connections` 并不一一对应，照搬数值经常出问题。

**误区二：认为主键查询一定只读一次磁盘。** PG 没有聚簇索引，主键索引的叶子节点存的是 `ctid`，还要再访问堆表。只有**索引-only 扫描**（`Index Only Scan`）且可见性映射干净时，才能避免第二次访问。

**误区三：看到 `pg_wal` 很大就直接删文件。** 手工删除 WAL 会导致实例无法恢复甚至彻底损坏。正确做法是排查归档失败、复制槽滞留、`max_wal_size` 设置，必要时用 `pg_archivecleanup` 清理已归档的旧段。

**排查入口**：

```sql
-- 当前所有进程在做什么（含辅助进程）
SELECT pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity
ORDER BY backend_start;

-- 数据库级累计统计，看回滚率、死锁、临时文件
SELECT datname, xact_commit, xact_rollback, deadlocks, temp_files, temp_bytes
FROM pg_stat_database
WHERE datname = current_database();

-- 检查点与后台写盘情况
SELECT * FROM pg_stat_bgwriter;
```

```bash
# 看实例日志里最近的错误与慢查询
tail -n 200 /var/lib/postgresql/data/log/postgresql-*.log
```

# 7. 练习与自检

**练习一**：启动一个 PG 容器，用 `psql` 查出当前实例的 `shared_buffers`、`work_mem`、`max_connections`，并计算"如果 200 个连接同时各做一个排序"的潜在内存上限。

**练习二**：在 `pg_stat_activity` 中找出 `backend_type` 为 `autovacuum worker` 的进程，说明它为什么会出现，以及它与普通 backend 的区别。

**练习三**：用自己的话解释：为什么 PG 关闭 `full_page_writes` 可能提升性能，却在生产环境几乎不被推荐？

**自检清单**：

- 能画出 postmaster、backend、辅助进程与共享内存的关系图；
- 能说清 WAL 的写入顺序与检查点的作用；
- 能解释 PG 与 MySQL 在索引组织、MVCC、连接模型上的三点本质差异。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 定位 | 功能完整、标准兼容、扩展性强的开源对象-关系数据库 |
| 进程模型 | postmaster + 每连接一个 backend + 一组辅助进程，进程隔离好但连接成本高 |
| 内存 | `shared_buffers` 是页缓存，`work_mem` 是每操作私有内存，二者不能混算 |
| 持久化 | WAL 先写日志再刷数据页，检查点缩短恢复时间，全页写防止 torn page |
| 与 MySQL | 无聚簇索引、MVCC 靠堆表多版本 + VACUUM、DDL 事务化、Schema 是命名空间 |

理解了进程与 WAL 这两条主线，后面的事务、索引、复制、备份都会顺很多。下一篇从**安装与配置**入手，把这些组件真正跑起来，并搞清楚 `postgresql.conf` 与 `pg_hba.conf` 的分工。