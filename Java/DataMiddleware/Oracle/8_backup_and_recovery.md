# 1. 备份基础

数据库备份要回答两个问题：**能恢复到哪个时间点（RPO）**，以及**恢复要花多久（RTO）**。Oracle 的备份体系围绕这两个指标设计，核心工具是 **RMAN（Recovery Manager）**，辅以 Data Pump 做逻辑备份。

先区分两组容易混淆的概念：

| 概念 | 含义 |
|------|------|
| 物理备份 | 备份数据文件、控制文件、归档日志等物理文件 |
| 逻辑备份 | 导出表、schema 的数据与结构（`expdp`/`impdp`） |
| 一致性备份 | 数据库关闭或处于一致状态时备份，可直接恢复 |
| 非一致性备份 | 数据库打开时备份，必须配合归档日志做恢复 |
| 还原（Restore） | 把备份文件复制回原位置 |
| 恢复（Recover） | 在还原的基础上应用 redo/归档日志，把数据推进到目标时间点 |

> **注意**：`RESTORE` 和 `RECOVER` 是两个动作。只 `RESTORE` 不 `RECOVER`，数据库停在备份时刻，缺失备份之后的变更；只 `RECOVER` 不 `RESTORE`，没有文件可恢复。生产事故里"恢复了但数据不对"，多半是漏了 `RECOVER` 或归档日志不全。

## 1.1 备份方式对比

| 方式 | 工具 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| 冷备 | 操作系统复制 | 简单、可靠 | 需停机 | 小库、维护窗口 |
| 热备 | RMAN | 在线、增量、校验 | 需归档模式 | 生产主力 |
| 逻辑备份 | Data Pump | 可跨版本/平台、按对象 | 慢、不能做 PITR | 迁移、单表恢复 |
| 快照备份 | 存储快照 | 秒级、对库无感 | 依赖存储、需一致性保障 | 大型库、云环境 |

# 2. 归档模式

**归档模式（ARCHIVELOG）** 是 RMAN 热备和 PITR 的前提。开启后，在线重做日志写满时，ARCn 进程会把它复制到归档目录，形成连续的日志链。没有归档日志，数据库只能恢复到最近一次一致性备份点。

## 2.1 查看与开启

```sql
-- 查看归档状态与归档目的地
ARCHIVE LOG LIST;
SHOW PARAMETER log_archive_dest_1;
-- 开启归档：先停库到 mount，改模式，再打开
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

## 2.2 归档日志的管理

归档日志会持续增长，必须配合删除策略，否则会撑爆 `FRA`（Fast Recovery Area），导致数据库挂起（`ORA-00257: archiver error`）。

```sql
-- 查看 FRA 占用；删除已备份归档用 RMAN 的 DELETE ARCHIVELOG 命令
SELECT * FROM v$recovery_area_usage;
```

> **警告**：绝对不要用操作系统命令直接 `rm` 归档日志。RMAN 的元数据会与实际文件不一致，后续备份/恢复会报错。删除归档必须通过 RMAN，或至少先 `CROSSCHECK` 再 `DELETE EXPIRED`。

## 2.3 归档与非归档的取舍

NOARCHIVELOG 不能热备、不能 PITR，性能无额外影响，适合测试库和只读库；ARCHIVELOG 支持热备与 PITR，代价是写归档的额外 I/O，生产库必须开启。

> **行业实践**：生产库一律开归档，并配置 `FRA` 与归档删除策略。测试库如果数据可重建，可以不开归档以省空间；但一旦测试库承载了不可复现的数据，也应开归档。

# 3. RMAN

RMAN 是 Oracle 官方备份工具，直接与数据库实例和存储交互，支持增量备份、压缩、加密、块级校验、自动管理备份集。

## 3.1 连接方式

```bash
# 本地连接（操作系统认证）与远程连接
rman target /
rman target sys/Oracle123@//192.168.1.100:1521/orcl
```

RMAN 的备份信息默认记录在**目标库的控制文件**里，长期使用建议配置**恢复目录（Recovery Catalog）**，把备份元数据放到独立数据库，避免控制文件丢失后备份信息也没了。

## 3.2 常用配置

```text
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
RMAN> CONFIGURE BACKUP OPTIMIZATION ON;
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 4;
RMAN> CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 2 TIMES TO DISK;
```

`RETENTION POLICY` 决定保留窗口，`BACKUP OPTIMIZATION` 跳过已备份且未变化的文件，`CONTROLFILE AUTOBACKUP` 每次备份后自动备份控制文件与参数文件，`PARALLELISM` 控制并行通道数，`ARCHIVELOG DELETION POLICY` 决定归档日志何时可删。

## 3.3 备份命令

```text
-- 全库备份 + 归档日志，并删除旧归档
RMAN> BACKUP DATABASE PLUS ARCHIVELOG DELETE INPUT;
-- 备份指定表空间、控制文件与参数文件
RMAN> BACKUP TABLESPACE users;
RMAN> BACKUP CURRENT CONTROLFILE;
RMAN> BACKUP SPFILE;
-- 压缩备份，并校验是否可恢复
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
RMAN> RESTORE DATABASE VALIDATE;
```

## 3.4 查看与清理

```text
RMAN> LIST BACKUP SUMMARY;      -- 列出备份
RMAN> REPORT OBSOLETE;          -- 按保留策略报告过期备份
RMAN> DELETE OBSOLETE;          -- 删除过期备份
RMAN> CROSSCHECK BACKUP;        -- 比对元数据与实际文件
RMAN> DELETE EXPIRED BACKUP;    -- 清理已不存在的文件记录
```

> **行业实践**：备份脚本要包含 `CROSSCHECK` 与 `DELETE OBSOLETE`，否则备份目录会越来越大。但删除前务必确认保留策略与归档删除策略，不要为了省空间把恢复窗口内的备份删掉。

# 4. 全备与增量

## 4.1 备份级别

| 级别 | 含义 | 依赖 |
|------|------|------|
| Level 0 | 增量备份的基线，等价于全备 | 无 |
| Level 1 Cumulative | 累积增量，备份上次 Level 0 以来的所有变化 | 最近一次 Level 0 |
| Level 1 Differential | 差异增量，备份上次任意级别备份以来的变化 | 最近一次 Level 0 或 Level 1 |

```text
-- 每周一次 Level 0 基线
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;

-- 每天一次差异增量
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;

-- 累积增量
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;
```

## 4.2 块变更跟踪

增量备份默认要扫描所有数据块，判断是否变化。数据量大时很慢。**块变更跟踪（Block Change Tracking）** 用一个位图文件记录自上次备份以来哪些块变了，让增量备份只读变化的块。

```sql
-- 开启块变更跟踪，并查看状态
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
USING FILE '/u01/app/oracle/bct/change_tracking.ctf';
SELECT status, filename, bytes FROM v$block_change_tracking;
```

> **注意**：块变更跟踪会带来少量写开销，但通常远小于全表扫描的代价。数据量超过几百 GB 的库，开增量备份前应先开启 BCT。

## 4.3 全备 vs 增量

| 维度 | 全备 | 增量 |
|------|------|------|
| 备份时间 | 长 | 短 |
| 恢复时间 | 短（只需全备） | 长（全备 + 多个增量） |
| 存储占用 | 大 | 小 |
| 适用 | 周基线 | 日常 |

> **设计考量**：增量不是"越频繁越好"。增量越多，恢复时要依次应用多个备份集，RTO 变长。常见策略是"周全备 + 日增量 + 归档日志"，并定期做**合并（merge）** 或重做全备，控制恢复链长度。

# 5. PITR

**PITR（Point-In-Time Recovery，时间点恢复）** 是把数据库恢复到过去某个时刻，典型场景是"误删数据""误更新""批量脚本跑错"。

## 5.1 原理

PITR 依赖三样东西：

1. **基线备份**：全备或 Level 0。
2. **归档日志**：覆盖从基线到目标时间点的所有变更。
3. **目标时间点**：一个时间、SCN 或日志序列号。

```text
全备（周一 00:00）──归档日志──▶ 目标时间（周三 14:30）
                                    │
                          应用到此为止，停止恢复
```

## 5.2 操作步骤

```text
-- 关闭数据库并启动到 mount
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
-- 设定恢复目标时间，执行还原与恢复
RMAN> RUN {
  SET UNTIL TIME "TO_DATE('2026-11-06 14:30:00', 'YYYY-MM-DD HH24:MI:SS')";
  RESTORE DATABASE;
  RECOVER DATABASE;
}
-- 先以只读方式打开验证数据，无误后重置日志打开
RMAN> ALTER DATABASE OPEN READ ONLY;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

> **注意**：`OPEN RESETLOGS` 会重置日志序列，之后必须立即做一次全备。恢复前建议先做**数据库克隆**或在测试库验证，不要直接在生产库上反复试。

## 5.3 基于 SCN 与序列号的恢复

```text
-- 恢复到指定 SCN 或归档日志序列
RMAN> RUN { SET UNTIL SCN 123456789; RESTORE DATABASE; RECOVER DATABASE; }
RMAN> RUN { SET UNTIL SEQUENCE 1000 THREAD 1; RESTORE DATABASE; RECOVER DATABASE; }
```

时间点恢复最怕**目标时间点附近有长事务**。一个事务在 14:00 开始、14:40 提交，恢复到 14:30 会把这个事务整体回滚，结果可能与预期不符。恢复前要确认目标时间点附近的事务边界。

## 5.4 表级时间点恢复

如果只想恢复一张表，可以用 **表级 PITR**（12c 起支持），不必恢复整个库：

```text
RMAN> RECOVER TABLE app_user.orders
      UNTIL TIME "TO_DATE('2026-11-06 14:30:00', 'YYYY-MM-DD HH24:MI:SS')"
      AUXILIARY DESTINATION '/u01/aux';
```

RMAN 会在辅助实例中恢复，再用 Data Pump 把表导入原库，对生产影响比整库恢复小得多。

# 6. 恢复演练

**备份的价值只有在恢复成功时才成立**。很多团队有备份，但从没验证过能不能恢复，真出事时才发现归档缺失、备份损坏、恢复脚本跑不通。

## 6.1 演练清单

演练要验证：`RESTORE ... VALIDATE` 能通过、`V$ARCHIVE_GAP` 无缺口、测试库能完整走通 restore + recover、PITR 恢复到指定时间点数据符合预期、Data Guard 切换可执行、文档与联系人是最新的。

```sql
SELECT * FROM v$archive_gap;   -- 检查归档日志缺口
```

## 6.2 常见恢复错误

| 错误 | 原因 | 处理 |
|------|------|------|
| `ORA-01194: 文件需要更多恢复` | 归档日志不全或未应用 | 补齐归档，继续 `RECOVER` |
| `ORA-00279: 需要归档日志` | 恢复链断裂 | 从备份或归档目录找对应序列 |
| `RMAN-06023: 找不到备份` | 备份被删或元数据不一致 | `CROSSCHECK`，从恢复目录查 |
| `ORA-01547/01555` | undo 不足或快照过旧 | 增大 undo，缩短事务 |

> **行业实践**：恢复演练应至少每季度一次，并记录 RTO 实际值。演练要在**隔离环境**做，避免影响生产。演练报告要包含"从发起恢复到业务可用"的完整时间线，而不是只记录 RMAN 命令执行时间。

# 7. 备份策略

## 7.1 3-2-1 原则

- **3** 份数据：生产数据 + 至少 2 份备份。
- **2** 种介质：磁盘 + 磁带/对象存储。
- **1** 份异地：防止站点级灾难。

## 7.2 典型策略

| 库类型 | 备份策略 | 保留 | 说明 |
|--------|---------|------|------|
| 核心 OLTP | 周日全备 + 每日增量 + 归档实时备份 | 30 天 | 配合 Data Guard |
| 一般业务 | 每周全备 + 每日增量 | 14 天 | 单机 + 定期演练 |
| 报表库 | 每周全备 | 7 天 | 数据可重建，RPO 要求低 |
| 开发测试 | 按需逻辑备份 | 7 天 | `expdp` 即可 |

```text
-- 周日全备，周一到周六增量
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG TAG 'WEEKLY_FULL';
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG TAG 'DAILY_INCR';
```

## 7.3 备份窗口与性能

备份会消耗 I/O 和 CPU，尤其是全备。控制手段包括：限制 RMAN 通道数与 `PARALLELISM`、使用压缩备份集、在业务低峰执行、用 Data Guard 备库做备份，以及用增量 + 块变更跟踪缩短备份时间。

> **注意**：压缩不是万能的。`BASIC` 压缩 CPU 开销大，`LOW`/`MEDIUM` 更均衡。CPU 紧张的库不要盲目开高压缩级别，否则备份期间业务 SQL 会被拖慢。

## 7.4 逻辑备份的定位

`expdp`/`impdp` 适合**按 schema、按表迁移**，以及小范围数据修复。它不能替代物理备份做 PITR，因为逻辑备份没有时间点概念，且导出期间的数据一致性依赖 `FLASHBACK_SCN` 或 `CONSISTENT=Y`。

```bash
# 导出与导入整个 schema
expdp app_user/App123456@orclpdb1 schemas=APP_USER \
  directory=DATA_PUMP_DIR dumpfile=app_user_%U.dmp parallel=4 compression=all
impdp app_user/App123456@orclpdb1 schemas=APP_USER \
  directory=DATA_PUMP_DIR dumpfile=app_user_%U.dmp remap_schema=APP_USER:APP_USER_NEW
```

# 8. 练习

- **练习一**：在测试库开启归档模式，执行一次 `BACKUP DATABASE PLUS ARCHIVELOG`，用 `LIST BACKUP SUMMARY` 与 `REPORT OBSOLETE` 查看备份与过期情况。
- **练习二**：模拟误删一张表的数据，用 RMAN 表级 PITR 恢复到删除前的时间点，记录命令与验证结果，并解释为什么整库 PITR 不适合单表场景。
- **练习三**：设计一个核心 OLTP 库的备份策略，给出全备/增量频率、保留天数、归档删除策略、演练周期，并估算 RPO 与 RTO。

# 9. 小结

| 主题 | 核心结论 |
|------|---------|
| 备份分类 | 物理/逻辑、一致/非一致、还原与恢复是两回事 |
| 归档模式 | 热备与 PITR 的前提，生产库必须开启 |
| RMAN | 官方备份工具，支持增量、压缩、校验、自动管理 |
| 全备与增量 | Level 0 是基线，差异/累积增量按需选，BCT 加速增量 |
| PITR | 基线 + 归档 + 目标时间点，恢复后 `OPEN RESETLOGS` 并立即全备 |
| 演练 | 备份成功不等于能恢复，必须定期在隔离环境验证 |
| 策略 | 3-2-1 原则，按 RPO/RTO 设计频率与保留 |

到这里，Oracle 的核心机制已经走完一遍。最后一篇回到工程实践：**从 MySQL 迁移到 Oracle** 时，语法、分页、自增、日期、数据类型有哪些坑，迁移步骤怎么排，以及一组可以自测的面试题。
