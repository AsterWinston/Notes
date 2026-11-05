# 1. 知识地图

学完前九篇，PG 的知识点容易散成"参数、索引、事务、复制"几堆。这一篇先把它们串成一张图，再用对比和自检题把边界钉牢。

## 1.1 一张图串起十篇

```text
                        ┌──────────────────────────┐
                        │  应用 / 连接池 / 代理      │
                        │  HikariCP、PgBouncer      │
                        └────────────┬─────────────┘
                                     │ SQL
                                     ▼
┌──────────────────────────────────────────────────────────────┐
│                        查询生命周期                            │
│  解析 → 重写 → 计划（统计信息）→ 执行（索引 / JOIN / 排序）      │
└───────────────┬──────────────────────────────┬───────────────┘
                │                              │
                ▼                              ▼
      ┌──────────────────┐          ┌────────────────────┐
      │ 事务与 MVCC       │          │ 索引与优化器         │
      │ 快照、行版本、锁   │          │ B-tree/GIN/GiST/BRIN │
      │ VACUUM、回卷       │          │ EXPLAIN、统计信息     │
      └────────┬─────────┘          └─────────┬──────────┘
               │                              │
               └──────────────┬───────────────┘
                              ▼
                   ┌────────────────────┐
                   │ 存储与 WAL          │
                   │ shared_buffers      │
                   │ pg_wal、检查点       │
                   └─────────┬──────────┘
                             │
          ┌──────────────────┼───────────────────┐
          ▼                  ▼                   ▼
   ┌────────────┐    ┌──────────────┐    ┌──────────────┐
   │ 复制与高可用 │    │ 备份与 PITR   │    │ 监控与容量    │
   │ 流复制/逻辑  │    │ pg_dump/base  │    │ 指标、告警    │
   │ Patroni     │    │ WAL 归档      │    │ 增长预测      │
   └────────────┘    └──────────────┘    └──────────────┘
```

如果只能记住一条主线，就记这条：**PG 用 WAL 保证持久化，用多版本堆表实现 MVCC，用 VACUUM 回收旧版本，用索引和统计信息决定查询路径，用复制与备份对抗故障**。其余知识点都挂在这条主线上。

## 1.2 学习路径建议

| 阶段 | 目标 | 对应篇目 |
|------|------|---------|
| 入门 | 能装、能连、能写 SQL | 1、2、3 |
| 进阶 | 理解事务、索引、执行计划 | 4、5 |
| 实战 | 会选类型、会用 JSONB 与全文检索、会装扩展 | 3、6、7 |
| 生产 | 能设计高可用、能做备份恢复、能监控告警 | 8、9 |
| 复盘 | 能对比选型、能回答原理题 | 10 |

# 2. 与 MySQL、Oracle 的对比

## 2.1 进程与存储架构

| 维度 | PostgreSQL | MySQL（InnoDB） | Oracle |
|------|-----------|----------------|--------|
| 连接模型 | 一连接一进程 | 一连接一线程 | 专用/共享服务器进程 |
| 表组织 | 堆表，所有索引指向 `ctid` | 主键聚簇索引 | 堆表 + 可选 IOT |
| 多版本存放 | 堆表内保留旧元组 | undo log 版本链 | undo 段 |
| 旧版本清理 | VACUUM / autovacuum | purge 线程 | UNDO 表空间 + 自动管理 |
| 日志 | WAL（物理） | redo log + binlog | redo log + archive log |
| DDL 事务 | 完全支持 | 8.0 原子 DDL，语义有限 | 支持（隐式提交较多） |
| 扩展 | `CREATE EXTENSION` 原生插件 | 插件/分支受限 | 组件与选项包 |

## 2.2 事务与并发控制

| 维度 | PostgreSQL | MySQL | Oracle |
|------|-----------|-------|--------|
| 默认隔离级别 | READ COMMITTED | REPEATABLE READ | READ COMMITTED |
| RC 快照粒度 | 每条语句 | 每条语句 | 每条语句 |
| RR 快照粒度 | 整个事务 | 整个事务（快照读） | 不支持标准 RR |
| 幻读防护 | RR 快照即可；Serializable 用 SSI | RR 用 Next-Key Lock | Serializable 用锁 |
| 写冲突处理 | RR/Serializable 报错并重试 | 锁等待/死锁检测 | 锁等待/死锁检测 |
| 长事务影响 | 阻碍 VACUUM，导致膨胀与回卷风险 | undo 膨胀、purge 延迟 | undo 表空间增长 |

## 2.3 索引与优化器

| 维度 | PostgreSQL | MySQL | Oracle |
|------|-----------|-------|--------|
| 默认索引 | B-tree | B+Tree | B-tree |
| 聚簇索引 | 无（`CLUSTER` 仅一次性重排） | 主键即聚簇 | 堆表/IOT |
| 覆盖索引 | `INCLUDE` + Index Only Scan | 二级索引含主键 | 索引组织/覆盖 |
| 多值索引 | GIN（数组、JSONB、全文） | 有限 | 有限 |
| 空间索引 | GiST/SP-GiST + PostGIS | R-Tree | Spatial |
| 大表顺序扫描 | BRIN 可用 | 无对应 | 无对应 |
| 优化器 | 基于代价，统计信息丰富 | 基于代价 | 基于代价 |
| 强制索引 | 无 `FORCE INDEX`，可调 `enable_*` | `FORCE INDEX` | Hint |

> **行业实践**：从 MySQL 迁到 PG，最常见的性能回退不是"PG 更慢"，而是**索引假设没变**。MySQL 里主键查询只走一棵树，PG 里是"索引 → 堆表"两步；MySQL 的 `LIKE '%x%'` 靠覆盖索引硬扛，PG 需要 `pg_trgm` 或全文检索。

## 2.4 数据类型与扩展能力

| 能力 | PostgreSQL | MySQL | Oracle |
|------|-----------|-------|--------|
| JSON | JSONB（可索引、可更新） | JSON（8.0 支持部分索引） | JSON（21c 增强） |
| 数组 | 原生数组类型 + GIN | 无 | 集合类型（PL/SQL） |
| 范围 | 原生范围类型 + 排他约束 | 无 | 无 |
| 自定义类型 | 原生支持 | 有限 | 对象类型 |
| 全文检索 | 内置 + 中文分词扩展 | 内置全文索引 | Oracle Text |
| 向量 | pgvector 扩展 | 有限/第三方 | 23ai 向量 |
| 地理 | PostGIS（事实标准） | 基础空间类型 | Oracle Spatial |

## 2.5 复制、备份与运维

| 维度 | PostgreSQL | MySQL | Oracle |
|------|-----------|-------|--------|
| 物理复制 | WAL 流复制 | binlog 复制 | Data Guard |
| 逻辑复制 | 原生 publication/subscription | 基于 binlog 的第三方/原生 | GoldenGate |
| 同步复制 | `synchronous_standby_names` | 半同步/组复制 | Data Guard SYNC |
| 时间点恢复 | 基础备份 + WAL 归档 | 全备 + binlog | RMAN + archive log |
| 高可用 | Patroni/repmgr + DCS | MHA/Orchestrator/组复制 | RAC/Data Guard |
| 连接数成本 | 高（进程模型） | 中（线程模型） | 中高 |

## 2.6 选型建议

**优先 PostgreSQL 的场景**：

- 需要复杂查询、窗口函数、CTE、递归；
- 数据类型复杂：JSONB、数组、范围、地理、向量；
- 需要事务性 DDL、强约束、扩展能力；
- 团队愿意接受 VACUUM、连接池等运维概念。

**优先 MySQL 的场景**：

- 团队与生态已经围绕 MySQL 建设；
- 读多写少、简单主键查询为主；
- 需要大量现成的云托管与中间件生态；
- 对进程模型带来的连接成本敏感。

**优先 Oracle 的场景**：

- 已有 Oracle 资产与 DBA 团队；
- 需要 RAC 级别的共享存储高可用；
- 商业支持与合规要求明确。

> **设计考量**：数据库选型很少是纯技术比较，更多是"团队能力 + 生态 + 成本"的综合决策。PG 的功能优势只有在团队真正理解 MVCC、VACUUM、连接池之后才能兑现。

# 3. 高频自检题

## 3.1 为什么 PostgreSQL 的 UPDATE 会造成表膨胀？

因为 PG 的 UPDATE 不是原地修改，而是插入一条新的行版本，并把旧版本的 `xmax` 标记为当前事务。旧版本仍然占用页面空间，直到 VACUUM 确认没有活跃快照需要它，才把空间标记为可复用。普通 VACUUM 不归还磁盘给操作系统，所以频繁更新的表会持续变大。解决办法是调优 autovacuum、控制长事务、必要时用 `pg_repack` 在线重建。

## 3.2 为什么 PostgreSQL 的 REPEATABLE READ 能避免幻读，却仍可能写偏斜？

PG 的 RR 在事务开始时取一个快照，整个事务都看这个快照，因此范围查询两次结果一致，不会出现标准意义上的幻读。但两个并发事务可能各自读到不同的行、各自写入不同的行，彼此的写不在对方的快照里，最终破坏跨行约束——这就是写偏斜。要彻底避免，需要 SERIALIZABLE（SSI），并配套捕获 `40001` 的重试逻辑。

## 3.3 VACUUM、VACUUM FULL、pg_repack 有什么区别？

普通 VACUUM 扫描表、清理死元组、更新可见性映射、推进 freeze，不阻塞读写，但不缩小文件。VACUUM FULL 重写整张表并把空间归还操作系统，期间持有 ACCESS EXCLUSIVE 锁，会阻塞所有访问。pg_repack 通过触发器和临时表在线重建，几乎不阻塞，是生产环境回收大表空间的首选。

## 3.4 为什么 PostgreSQL 必须使用连接池？

PG 是一连接一进程，每个连接都要独立的内存地址空间和进程调度开销。连接数开到几千会导致内存紧张、上下文切换频繁，甚至触发 OOM。连接池把大量客户端复用成少量真实连接，既降低数据库压力，也提供限流能力。应用侧用 HikariCP，数据库侧可用 PgBouncer 的事务模式进一步复用。

## 3.5 B-tree、GIN、GiST、BRIN 分别适合什么场景？

B-tree 是默认选择，适合等值、范围、排序、前缀匹配。GIN 是倒排索引，适合数组包含、JSONB 键、全文检索、`pg_trgm`。GiST 适合几何、范围、最近邻和排他约束，可能是有损索引。BRIN 只记录数据块范围的摘要，体积极小，适合物理顺序与查询条件高度相关的大表，比如按时间写入的日志。选错类型不会报错，只会让索引形同虚设。

## 3.6 同步复制与异步复制如何取舍？

异步复制不等待备库确认，写入延迟低，但主库故障会丢失尚未传送的事务。同步复制在主库返回成功前等待备库确认，能保证 RPO 接近 0，但同步备库故障时主库写入会阻塞。生产常用"一主两备 + `ANY 1`"的 quorum 模式，在数据安全与可用性之间取平衡；`synchronous_commit` 还可以细分为 `remote_write`、`on`、`remote_apply`。

## 3.7 pg_dump 能用于时间点恢复吗？

不能。`pg_dump` 导出的是某个一致性快照下的逻辑数据，只能恢复到导出那一刻，无法重放之后的变更。PITR 需要"物理基础备份 + 连续的 WAL 归档"，通过 `restore_command` 和 `recovery_target_time` 重放到指定时间点。逻辑备份适合迁移、单表恢复和小库备份，不能替代 PITR。

## 3.8 哪些写法会导致索引失效？

常见的有：对索引列做函数或运算（`WHERE year(created_at) = 2026`）、隐式类型转换（字符串列与数字比较）、`LIKE '%x%'` 前缀通配、联合索引违反最左前缀、`OR` 连接了无索引列、统计信息过期导致优化器选错、以及用 `->>` 取 JSONB 字段却没有建表达式索引。排查时先看 `EXPLAIN` 的 `key` 与 `rows`，再检查统计信息和条件写法。

## 3.9 什么是事务 ID 回卷？如何预防？

PG 的事务 ID 是 32 位，约 42 亿个，用完会绕回。若不处理，旧事务会被误判为未来事务，数据可见性错乱。VACUUM 会把足够旧的行版本 freeze，使其不再参与 ID 比较。预防手段是监控 `age(datfrozenxid)`、避免长事务阻塞 autovacuum、对频繁更新的表单独调优 autovacuum，必要时手工 `VACUUM FREEZE`。

## 3.10 JSONB 与 JSON 的差别是什么？

`json` 以原始文本存储，保留键顺序、空格和重复键，写入快但查询每次都要解析，且不支持 GIN 索引。`jsonb` 在写入时解析成二进制，去重、不保留顺序，支持 `@>`、`?` 等操作符和 GIN 索引，查询与修改都更高效。业务字段应选 `jsonb`，只有需要逐字节存档原始报文时才用 `json`。

## 3.11 主库宕机后备库如何接管？脑裂如何避免？

接管由 Patroni/repmgr 这类组件完成：通过 DCS 的 leader 租约判断主库失效，确认集群仍有 quorum 后，选出数据最完整的备库提升为新主库，其他备库重新指向它，最后更新 HAProxy/VIP/DNS 让客户端重连。脑裂的根因是旧主库仍在接受写入，避免手段是 fencing——通过 watchdog、STONITH、网络隔离或云 API 强制关机，确保旧主库在选举前停止服务。

# 4. 生产检查清单

## 4.1 建模与 SQL

- [ ] 主键使用 `bigint identity` 或 UUID，避免 `int` 耗尽；
- [ ] 金额用 `numeric`，时间用 `timestamptz`，状态用 `text + CHECK` 或字典表；
- [ ] 核心字段加 `NOT NULL`，业务规则用 `CHECK`/`UNIQUE` 落到数据库；
- [ ] 避免 `SELECT *`，为高频查询设计覆盖索引；
- [ ] 大表按时间或租户分区，提前规划归档策略；
- [ ] JSONB 只放扩展属性，核心字段仍然独立建列。

## 4.2 参数与资源

- [ ] `shared_buffers` 约为内存 25%，修改后重启验证；
- [ ] `work_mem` 按"每操作"估算，避免并发排序打爆内存；
- [ ] `max_connections` 与连接池总预算匹配，不盲目调大；
- [ ] `wal_level`、`max_wal_size`、`checkpoint_timeout` 按写入量调整；
- [ ] `password_encryption = scram-sha-256`，禁用 `trust` 认证；
- [ ] `idle_in_transaction_session_timeout`、`statement_timeout`、`lock_timeout` 全部设置。

## 4.3 事务与清理

- [ ] 事务尽量短，不在事务内做 RPC、发消息、等用户输入；
- [ ] 监控长事务与 `idle in transaction`，设置自动断开；
- [ ] autovacuum 全局开启，对高频更新大表单独调小 scale factor；
- [ ] 监控 `n_dead_tup`、表大小、`age(datfrozenxid)`；
- [ ] DDL 前设置 `lock_timeout`，大表加索引用 `CONCURRENTLY`；
- [ ] 避免无索引的批量 `UPDATE/DELETE`，分批提交。

## 4.4 高可用与备份

- [ ] 至少一主一备，跨可用区部署；
- [ ] 同步复制策略明确，知道故障时写入会阻塞还是丢数据；
- [ ] 复制槽有监控与上限，防止 WAL 撑爆磁盘；
- [ ] 每日全备 + 持续 WAL 归档，备份与主库物理隔离；
- [ ] 每季度做一次恢复演练，记录 RTO 与数据校验结果；
- [ ] 高可用组件带 fencing，避免脑裂；
- [ ] 客户端通过 VIP/HAProxy 连接，不在配置里写死主库 IP。

## 4.5 监控与应急

```sql
-- 上线前十分钟快检
SELECT count(*) AS conns,
       count(*) FILTER (WHERE state = 'active') AS active,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_tx
FROM pg_stat_activity;

SELECT datname, age(datfrozenxid) AS xid_age FROM pg_database ORDER BY 2 DESC;
SELECT application_name, state, replay_lag FROM pg_stat_replication;
SELECT failed_count, last_failed_wal FROM pg_stat_archiver;
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

- [ ] 可用性、连接数、复制延迟、归档失败、磁盘使用率均有告警；
- [ ] 慢查询有 `pg_stat_statements` 与 `auto_explain` 支撑；
- [ ] 明确故障分级与联系人，备份恢复有 runbook；
- [ ] 定期 review Top SQL 与索引使用率，清理无用索引；
- [ ] 版本在社区支持周期内，小版本及时跟进。

## 4.6 常见错误速查

| 错误做法 | 后果 | 正确做法 |
|---------|------|---------|
| 用超级用户跑应用 | 一次 SQL 注入或误操作影响整个实例 | 最小权限账号 + `pg_hba.conf` 限制来源 |
| 事务里做 RPC、等用户输入 | 长事务钉住 `xmin`，膨胀与回卷风险 | 缩短事务，设置 `idle_in_transaction_session_timeout` |
| 大表直接 `VACUUM FULL` | 长时间 ACCESS EXCLUSIVE 锁，业务全阻塞 | 用 `pg_repack` 或维护窗口 |
| 只做流复制不做备份 | 误删/错误更新瞬间同步到备库，无法回溯 | 独立备份 + WAL 归档 + 定期恢复演练 |
| 用 `pg_dump` 当 PITR | 只能恢复到导出那一刻 | 物理基础备份 + WAL 归档 |
| 盲目调大 `max_connections` | 内存与进程调度压力失控 | 连接池 + 合理预算 |
| 建了索引不 `ANALYZE` | 统计信息失准，优化器选错计划 | 数据变化后 `ANALYZE`，必要时扩展统计 |

# 5. 综合练习

**练习一**：给你一张每天写入 500 万行、保留 90 天的日志表，设计存储方案。要求说明：是否分区、用什么索引、如何归档、autovacuum 如何配置、磁盘如何估算。

**练习二**：一个业务反馈"每天凌晨报表查询变慢，白天正常"。请列出至少五条排查路径，并说明每条路径对应的视图或命令。

**练习三**：设计一个跨机房的高可用方案，要求 RPO 接近 0、RTO 小于 5 分钟。说明复制模式、quorum 配置、fencing 手段、客户端路由与故障切换后的数据校验步骤。

# 6. 模块总结

## 6.1 十篇回顾

| 篇目 | 核心问题 | 一句话结论 |
|------|---------|-----------|
| 1 | PG 是什么、怎么运转 | 进程模型 + 共享内存 + WAL，功能完整但连接昂贵 |
| 2 | 怎么装、怎么配 | 配置分实例/会话，认证看 `pg_hba.conf`，权限按层检查 |
| 3 | 怎么写 SQL、怎么选类型 | 类型严格、schema 分层、NULL 三值逻辑、identity 自增 |
| 4 | 事务与并发怎么实现 | 堆表多版本 + 快照，VACUUM 是运维核心 |
| 5 | 怎么让查询变快 | 索引类型匹配查询形态，统计信息决定计划 |
| 6 | 半结构化与文本怎么查 | JSONB + GIN，全文检索有中文分词边界 |
| 7 | 能力怎么扩展 | 扩展按库安装，版本与备份要单独验证 |
| 8 | 怎么做高可用 | 流复制 + DCS + fencing + 路由，缺一不可 |
| 9 | 怎么不丢数据 | 备份 + WAL 归档 + PITR，演练才算数 |
| 10 | 怎么串起来 | 用 RPO/RTO 与业务边界做取舍 |

## 6.2 继续深入的方向

- **源码与内部机制**：`src/backend/access/heap/` 的可见性判断、`src/backend/executor/` 的 Volcano 模型；
- **性能工程**：`pg_stat_statements` + `auto_explain` + `pg_stat_kcache` 的组合分析；
- **分布式**：Citus、CockroachDB 与 PG 分片方案的取舍；
- **云原生**：Operator、Serverless PG、存储计算分离架构；
- **数据平台**：CDC（`wal2json`/Debezium）、FDW、逻辑复制到数仓。

# 7. 小结

PostgreSQL 的学习曲线主要不在 SQL 语法，而在**几个反直觉的机制**：更新会制造新版本、旧版本要靠 VACUUM 清理、连接是进程而不是线程、所有索引都要回表、DDL 也能回滚。理解这些机制之后，参数调优、索引设计、高可用方案都会变得有据可依。

这个模块从架构讲到运维，覆盖了单机使用、查询优化、半结构化数据、扩展、复制、备份与监控。真正的掌握不靠背结论，而靠一次次在实例上验证：改一个参数看 `EXPLAIN` 的变化，制造一次死锁看日志，做一次 PITR 看时间线。把这些动作做过一遍，面试题和生产故障都会变成熟悉的场景。