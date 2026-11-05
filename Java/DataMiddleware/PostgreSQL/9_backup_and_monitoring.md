# 1. 备份的目标：RPO 与 RTO

备份不是"把数据复制一份"这么简单。真正要回答的是两个问题：

- **RPO（Recovery Point Objective）**：最多能接受丢失多长时间的数据？
- **RTO（Recovery Time Objective）**：故障后多长时间内必须恢复服务？

| 方案 | RPO | RTO | 适用 |
|------|-----|-----|------|
| 仅 `pg_dump` 每日一次 | 最多 24 小时 | 小时级 | 小库、开发测试、逻辑迁移 |
| 物理全备 + WAL 归档 | 秒级到分钟级 | 分钟到小时 | 生产标准方案 |
| 流复制 + 定期全备 | 秒级（异步可能丢少量） | 分钟级 | 高可用 + 容灾 |

> **行业实践**：**复制不是备份**。误删表、错误 UPDATE、勒索软件会瞬间同步到所有备库。必须有一套与主库隔离的、可回溯到任意时间点的备份。

# 2. 逻辑备份

逻辑备份把数据库导出成 SQL 或自定义格式，跨版本、跨平台恢复能力强，但速度慢，且**不能做时间点恢复**。

## 2.1 pg_dump 的四种格式

```bash
pg_dump -h 127.0.0.1 -U app_rw -d appdb -Fp -f /backup/appdb.sql      # 纯 SQL
pg_dump -h 127.0.0.1 -U app_rw -d appdb -Fc -f /backup/appdb.dump     # 自定义，可并行恢复
pg_dump -h 127.0.0.1 -U app_rw -d appdb -Fd -j 4 -f /backup/appdb.dir # 目录，可并行导出
```

| 格式 | 并行导出 | 并行恢复 | 压缩 | 选择性恢复 |
|------|---------|---------|------|-----------|
| plain | 否 | 否 | 需外部工具 | 需手工编辑 SQL |
| custom | 否 | 是 | 内置 | 是 |
| directory | 是 | 是 | 内置 | 是 |

## 2.2 一致性与全局对象

`pg_dump` 会获取 `ACCESS SHARE` 锁，并在一个**可重复读快照**中完成导出。因此导出期间**不阻塞普通读写**，但会阻塞 `ALTER TABLE`、`DROP TABLE` 等需要 `ACCESS EXCLUSIVE` 的 DDL；导出的数据是一致快照，不会出现"导到一半表结构变了"。

`pg_dump` **只导出一个数据库**，不包含角色、表空间等全局对象，需要额外执行 `pg_dumpall -g -f /backup/globals.sql`。恢复顺序是先建角色与表空间，再恢复各数据库。

## 2.3 恢复

```bash
# 纯 SQL 用 psql 执行；自定义/目录格式用 pg_restore
psql -h 127.0.0.1 -U postgres -d appdb_new -f /backup/appdb.sql
pg_restore -h 127.0.0.1 -U postgres -d appdb_new -j 4 --clean --if-exists /backup/appdb.dump
pg_restore -d appdb_new -t orders /backup/appdb.dump   # 只恢复某张表
```

`--clean --if-exists` 会先删除同名对象，适合覆盖恢复；恢复到全新库时不需要。

## 2.4 逻辑备份的边界

- **不能做 PITR**：只能恢复到导出那一刻；
- **大库很慢**：导出、传输、恢复都是线性时间；
- **序列值可能不同步**：恢复后检查 `setval`；
- **扩展与统计信息需要额外处理**：`pg_dump` 只生成 `CREATE EXTENSION`，恢复后要 `ANALYZE`。

# 3. 物理备份

物理备份直接复制数据目录与 WAL，恢复速度快，是生产环境的基础。最常用的工具是 `pg_basebackup`。

## 3.1 pg_basebackup

```bash
pg_basebackup -h 10.0.0.11 -p 5432 -U replicator \
  -D /backup/base_20261106 -Fp -Xs -P \
  --checkpoint=fast --manifest-checksums=SHA256
```

| 参数 | 作用 |
|------|------|
| `-Fp` / `-Ft` | 普通文件格式 / tar 格式 |
| `--checkpoint=fast` | 立即触发检查点，缩短备份时间（增加 I/O 峰值） |
| `--manifest-checksums` | 生成校验清单，供 `pg_verifybackup` 使用 |

```bash
pg_verifybackup /backup/base_20261106
```

如果备份期间没有 `-Xs`，也没有配置 WAL 归档，备份会因缺少 WAL 而不可用。**物理备份必须包含从备份开始到结束的完整 WAL**。

## 3.2 低层 API 与增量备份

手工做文件系统快照时，需要先执行 `SELECT pg_backup_start('snapshot_20261106', fast => true);`，对数据目录做 LVM/EBS 快照，再执行 `SELECT * FROM pg_backup_stop();`。快照必须**原子地**覆盖整个 PGDATA，且快照期间 WAL 要单独保留。

PG 17 起支持增量物理备份，依赖 WAL 摘要：

```bash
# 主库开启 summarize_wal = on
pg_basebackup -D /backup/full -Fp -Xs -P
pg_basebackup -D /backup/inc1 -Fp -Xs -P --incremental=/backup/full/backup_manifest
pg_combinebackup /backup/full /backup/inc1 -o /restore/data
```

增量备份能降低备份窗口与存储成本，但恢复链路更复杂，必须定期演练。

# 4. PITR 时间点恢复

**PITR（Point-In-Time Recovery）** 的原理是：先恢复一个基础备份，再重放归档 WAL 到指定时间点。它能把数据库恢复到"误删之前一秒"。

## 4.1 开启 WAL 归档

```ini
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
archive_timeout = 60s
```

| 参数 | 说明 |
|------|------|
| `%p` / `%f` | 待归档 WAL 的完整路径 / 文件名 |
| `test ! -f` | 防止重复覆盖，命令返回 0 才算成功 |
| `archive_timeout` | 强制切换 WAL 段，控制归档延迟 |

```sql
SELECT archived_count, failed_count, last_archived_wal, last_failed_wal FROM pg_stat_archiver;
```

> **警告**：`archive_command` 返回非 0 时，PG 会保留 WAL 并不断重试。归档目录满、权限错误、NFS 挂载失败都会导致 `pg_wal` 无限增长。必须对 `failed_count` 和 `pg_wal` 大小告警。生产环境通常用 `pgBackRest`、`WAL-G` 或云厂商归档服务替代裸 `cp`。

## 4.2 恢复步骤

```bash
pg_ctl -D /data/pgdata stop
mv /data/pgdata /data/pgdata.bak.20261106     # 保留旧目录，不要直接删除
cp -a /backup/base_20261106 /data/pgdata
touch /data/pgdata/recovery.signal
```

```ini
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-11-06 10:30:00+08'
recovery_target_action = 'promote'
```

```bash
pg_ctl -D /data/pgdata start
```

```sql
SELECT pg_is_in_recovery();          -- f 表示恢复完成
```

## 4.3 恢复目标与时间线

| 参数 | 含义 |
|------|------|
| `recovery_target_time` | 恢复到指定时间点 |
| `recovery_target_lsn` | 恢复到指定 LSN |
| `recovery_target_action` | `pause`（默认）、`promote`、`shutdown` |

```sql
-- 做高风险操作前创建还原点，之后可精确回到这里
SELECT pg_create_restore_point('before_migration_20261106');
```

> **注意时区**：`recovery_target_time` 按数据库的 `TimeZone` 解释。跨时区团队协作时，显式带偏移量（如 `+08`）最稳妥。每次恢复并提升，PG 会创建新的**时间线（timeline）**；恢复后继续归档时要确保目录按时间线分开。

# 5. 备份恢复演练

备份的价值只有在恢复成功时才成立。建议每季度至少做一次完整演练，并记录耗时。

```text
□ 备份是否成功？最近一次成功时间？pg_verifybackup 是否通过？
□ 是否包含所需 WAL？能否恢复到目标时间点？
□ 恢复耗时多少？是否满足 RTO？
□ 行数、关键业务表、序列值是否一致？
□ 扩展、角色、权限是否恢复，应用能否跑通核心链路？
```

```sql
SELECT 'orders' AS table_name, count(*) FROM orders
UNION ALL SELECT 'customers', count(*) FROM customers;

-- 检查序列是否落后于最大 ID
SELECT max(id) AS max_id, (SELECT last_value FROM orders_id_seq) AS seq_value FROM orders;

ANALYZE;   -- 恢复后更新统计信息，避免计划劣化
```

# 6. 监控

监控的目标不是"收集所有指标"，而是**在用户感知之前发现异常**。

## 6.1 关键指标

| 类别 | 指标 | 说明 |
|------|------|------|
| 可用性 | 实例能否连接、主备角色 | 必须秒级发现 |
| 连接 | 当前连接数 / `max_connections` | 超过 80% 预警 |
| 缓存 | 共享缓冲区命中率 | OLTP 通常 > 99% |
| 吞吐 / 延迟 | TPS、QPS、p95/p99 耗时 | 突降或突增都要关注 |
| 复制 | `replay_lag`、字节延迟 | 超过阈值影响读写分离 |
| 锁 / 事务 | 等锁会话数、最长事务、`idle in transaction` | 关联业务超时与膨胀 |
| 临时文件 | `temp_bytes` | 突增说明 `work_mem` 不足 |
| 磁盘 / WAL | 数据目录、`pg_wal`、归档目录、归档失败数 | 80% 预警，90% 严重 |
| 回卷 / 备份 | `age(datfrozenxid)`、最近成功备份 | 超过 15 亿必须处理 |

## 6.2 常用查询

```sql
SELECT count(*) AS total,
       count(*) FILTER (WHERE state = 'active') AS active,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx,
       current_setting('max_connections')::int AS max_conn
FROM pg_stat_activity;

SELECT datname, round(sum(blks_hit) * 100.0 / nullif(sum(blks_hit + blks_read), 0), 2) AS hit_pct
FROM pg_stat_database GROUP BY datname;

SELECT pid, now() - xact_start AS duration, state, left(query, 80)
FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY duration DESC LIMIT 10;

SELECT pid, pg_blocking_pids(pid) AS blockers, wait_event FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

SELECT application_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag
FROM pg_stat_replication;
```

## 6.3 采集与告警

生产环境通常用 `postgres_exporter` + Prometheus + Grafana。告警阈值建议：

| 指标 | 警告 | 严重 |
|------|------|------|
| 连接使用率 | > 80% | > 90% |
| 复制延迟（字节 / 时间） | > 64MB / > 30s | > 256MB / > 5min |
| 最长事务时长 | > 5min | > 30min |
| 锁等待会话数 | > 5 | > 20 |
| `pg_wal` 大小 | > 2 × `max_wal_size` | > 磁盘 20% |
| 归档失败次数 | > 0 持续 5min | > 0 持续 30min |
| XID 年龄 | > 10 亿 | > 15 亿 |
| 磁盘使用率 | > 80% | > 90% |
| 最近成功备份 | > 26 小时 | > 48 小时 |

> **告警设计原则**：优先告警"会影响用户"和"会导致不可恢复"的指标。死锁偶尔增长可以观察，归档失败和备份失败必须立刻处理。

# 7. 容量规划

磁盘占用不只是表数据：

```text
总需求 ≈ 数据 + 索引 + TOAST + WAL（max_wal_size + 复制槽保留）
       + 归档（保留策略）+ 备份（全量 + 增量 + 异地副本）
       + 临时文件峰值 + 30% 余量
```

```sql
SELECT pg_size_pretty(pg_database_size('appdb'));

SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total,
       pg_size_pretty(pg_relation_size(relid)) AS heap,
       pg_size_pretty(pg_indexes_size(relid)) AS indexes
FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;
```

内存预算按 `shared_buffers + max_connections × (work_mem × 并发排序数 + 基础开销) + maintenance_work_mem + OS 缓存` 估算。`work_mem` 是每个排序/哈希节点，不是每个连接。结合业务增长预期，至少积累三个月的数据估算未来 12 个月的存储与 IOPS 需求。

# 8. 常见错误与排错

| 现象 | 根因 | 处理 |
|------|------|------|
| `pg_wal` 持续增长 | 归档失败、复制槽滞留 | 查 `pg_stat_archiver` 与 `pg_replication_slots` |
| 恢复后数据只到某个旧时间点 | WAL 归档缺失或不连续 | 检查归档目录与 `restore_command` |
| `recovery_target_time` 不生效 | 时区理解错误、目标早于备份终点 | 显式带时区，确认备份的起始 LSN/时间 |
| `pg_restore` 报扩展不存在 | 目标实例未安装扩展 | 先安装软件包并 `CREATE EXTENSION` |
| 备份成功但无法启动 | 缺少 WAL、权限不对、目录不完整 | 用 `pg_verifybackup`，检查 `pg_controldata` |
| 逻辑备份恢复后主键冲突 | 序列未重置 | `setval` 到最大 ID |
| 恢复后查询计划变差 | 统计信息缺失 | `ANALYZE`，必要时重建索引 |

# 9. 练习与自检

**练习一**：用 `pg_dump -Fd -j 4` 备份一个库，再恢复到新库。对比恢复前后的表行数、索引数量和序列值。

**练习二**：搭建 WAL 归档，创建表并插入数据，记录时间点，然后 `DROP TABLE`，再用 PITR 恢复到删除前，写出完整命令序列。

**练习三**：设计一套监控指标，覆盖连接、复制、WAL、事务、磁盘五类，并为每类给出一个告警阈值和处置动作。

**自检清单**：

- 能区分逻辑备份、物理备份、PITR 的能力边界；
- 能说清 `pg_dump` 的一致性快照机制与锁行为；
- 能独立完成"基础备份 + WAL 归档 + 恢复到指定时间点"的全流程；
- 知道 `archive_command` 失败会导致什么连锁反应；
- 能列出至少 8 个必须监控的 PG 指标。

# 10. 小结

| 主题 | 核心结论 |
|------|---------|
| 备份目标 | 先定 RPO/RTO，再选方案；复制不等于备份 |
| 逻辑备份 | `pg_dump` 灵活、跨版本，但不能 PITR，大库慢 |
| 物理备份 | `pg_basebackup` 快，必须包含完整 WAL，可校验 |
| PITR | 基础备份 + 归档 WAL + 恢复目标，能回到误操作前 |
| 演练 | 不演练的备份等于没有备份，必须验证数据与耗时 |
| 监控 | 连接、延迟、复制、WAL、事务、磁盘、备份七类覆盖主要故障 |

到这里，PG 的单机、复制、备份三条主线已经闭环。最后一篇把这些知识串成一张地图：与 MySQL/Oracle 的对比、高频自检题、生产检查清单，以及整个模块的总结。