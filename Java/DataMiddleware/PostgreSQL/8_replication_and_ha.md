# 1. 复制的基本原理

单机数据库的可用性上限就是一台机器的可用性。要扛住硬件故障、机房故障和计划内维护，必须让数据存在多份，并且能在主库失效时接管。PostgreSQL 提供两条复制路线：

- **物理复制**：按字节复制 WAL，备库与主库的数据文件完全一致；
- **逻辑复制**：解析 WAL 中的行级变更，按表复制到另一个实例，允许跨版本、跨 schema、选择性复制。

| 维度 | 物理复制 | 逻辑复制 |
|------|---------|---------|
| 复制单位 | WAL 字节流 | 表的行变更 |
| 主备版本 | 必须一致（小版本建议对齐） | 可跨主版本 |
| DDL / 序列 | 自动同步 | **不同步** |

## 1.1 物理复制的数据流

```text
主库 backend 修改数据
   │
   ▼
WAL buffer ──▶ pg_wal（主库落盘）
   │
   ├──▶ walsender 读取 WAL ──网络──▶ 备库 walreceiver
   │                                      │
   │                                      ▼
   │                                 pg_wal（备库落盘）
   │                                      │
   │                                      ▼
   │                                 startup 重放 ──▶ 备库数据文件
   ▼
客户端提交返回
```

备库重放的是**物理 WAL 记录**，因此主备的操作系统、架构、PostgreSQL 主版本必须一致。

## 1.2 复制槽

复制槽解决"备库暂时断线时，主库不要把还没传过去的 WAL 删掉"的问题。

```sql
SELECT * FROM pg_create_physical_replication_slot('standby1_slot');
SELECT slot_name, slot_type, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```

复制槽是双刃剑：能防止 WAL 被过早清理，但**备库长期离线时 WAL 会无限堆积直到磁盘写满**。PG 13 起可以设置 `max_slot_wal_keep_size = 10GB`，超过后槽失效，备库需要重建。

> **行业实践**：生产必须监控 `pg_replication_slots.retained` 和 `pg_wal` 目录大小。WAL 撑爆磁盘是 PG 集群最常见的故障之一，根因往往是一个废弃的复制槽。

# 2. 流复制

## 2.1 主库配置

```ini
listen_addresses = '*'
wal_level = replica                 # 逻辑复制才需要改为 logical
max_wal_senders = 10
wal_keep_size = 1GB                 # 没有复制槽时的兜底保留量
hot_standby = on                    # 备库需要
```

```text
# pg_hba.conf：允许备库网段以复制用户连接
host    replication    replicator    10.0.0.0/8    scram-sha-256
```

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'ReplPass!';
```

## 2.2 用 pg_basebackup 搭建备库

```bash
# -Fp 普通文件、-Xs 流式传输 WAL、-P 显示进度、-R 自动写恢复配置
pg_basebackup -h 10.0.0.11 -p 5432 -U replicator \
  -D /data/pgdata -Fp -Xs -P -R
```

PG 12 之后不再有 `recovery.conf`，恢复配置写在 `postgresql.conf` / `postgresql.auto.conf`，是否进入恢复模式由 `standby.signal` 文件决定。`-R` 会自动生成 `primary_conninfo`，备库启动后可用下面的语句确认：

```sql
SELECT pg_is_in_recovery();          -- t
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
       now() - pg_last_xact_replay_timestamp() AS replay_delay;
```

## 2.3 同步复制与异步复制

默认是**异步复制**：主库提交时只保证本地 WAL 落盘，不等备库确认，主库宕机时尚未传送到备库的事务会丢失。要避免丢数据，需要同步复制：

```ini
synchronous_standby_names = 'standby1'   # 等待至少 1 个备库确认
synchronous_commit = on
```

| `synchronous_commit` | 主库返回成功前等待什么 | 数据丢失风险 |
|---------------------|----------------------|-------------|
| `off` | 不等待本地 WAL 落盘 | 最大 |
| `local` | 本地 WAL 落盘 | 主库崩溃不丢 |
| `on`（默认） | 备库 WAL 已落盘 | 同步备库全挂时可能阻塞 |
| `remote_apply` | 备库已重放，可被查询看到 | 延迟最大，读一致性最好 |

```ini
# 多备库：按顺序取前 2 个可用同步备库；或任意 1 个确认（quorum）
synchronous_standby_names = 'FIRST 2 (standby1, standby2)'
synchronous_standby_names = 'ANY 1 (standby1, standby2, standby3)'
```

> **重要权衡**：同步复制把"提交延迟"和"数据安全"绑在一起。同步备库故障时主库写入会**卡住**，直到超时或配置变更。生产常用"一主两备 + `ANY 1`"，既保证至少一份副本，又避免单备库拖垮写入。

## 2.4 复制监控

```sql
SELECT application_name, state, sync_state,
       sent_lsn, replay_lsn, replay_lag
FROM pg_stat_replication;
```

`state = streaming` 表示正常；`replay_lag` 最贴近"备库数据落后多久"；`replay_lsn` 是备库已重放的位置。

# 3. 逻辑复制

逻辑复制不要求主备物理一致，适合**跨版本升级、按表同步、多主写入汇聚、CDC 到外部系统**。

## 3.1 发布与订阅

```ini
# 发布端
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

```sql
-- 发布端：发布 orders 与 customers
CREATE PUBLICATION app_pub FOR TABLE orders, customers;

-- 订阅端：表结构必须预先存在
CREATE SUBSCRIPTION app_sub
CONNECTION 'host=10.0.0.11 port=5432 dbname=appdb user=replicator password=ReplPass!'
PUBLICATION app_pub;

-- 表结构变更后刷新发布定义
ALTER PUBLICATION app_pub ADD TABLE new_table;
ALTER SUBSCRIPTION app_sub REFRESH PUBLICATION;
```

```sql
SELECT subname, subenabled, subslotname FROM pg_subscription;
SELECT srrelid::regclass, srsubstate, srsublsn FROM pg_subscription_rel;
SELECT * FROM pg_stat_subscription;
```

## 3.2 逻辑复制的限制

| 不复制的内容 | 说明 |
|-------------|------|
| DDL | 表结构变更需要在订阅端手动执行 |
| 序列 | `serial/identity` 当前值不同步，切换时需重置 |
| 大对象 / 全局对象 | `lo` 不支持；角色、权限、表空间、扩展需单独处理 |

订阅端遇到主键冲突、约束违反、行不存在时，apply worker 会报错并停止。可以配置 `ALTER SUBSCRIPTION app_sub SET (disable_on_error = true);` 让订阅出错时禁用而不是无限重试，排查看 `pg_stat_subscription_stats` 与订阅端日志。

> **注意**：逻辑复制**不支持双向写入同一张表**。两个方向同时写会产生冲突甚至循环复制。多活方案需要应用层路由或冲突解决机制。

# 4. 高可用架构

流复制解决"数据有几份"，高可用还要解决"主库挂了谁来接管、客户端怎么找到新主库、旧主库恢复后怎么办"。

## 4.1 组件分工

| 组件 | 职责 |
|------|------|
| PostgreSQL 流复制 | 数据同步 |
| DCS（etcd/Consul/ZooKeeper） | 保存集群状态、领导者锁、配置 |
| Patroni / repmgr | 选主、故障检测、自动切换、重建备库 |
| HAProxy / VIP / DNS | 客户端流量路由 |
| 监控与告警 | 发现延迟、切换失败、脑裂 |

## 4.2 Patroni 架构

```text
              ┌─────────────────────────────┐
              │   etcd / Consul / ZK (DCS)  │
              │   保存 leader key、集群配置   │
              └───────┬──────────┬──────────┘
                      │          │
        ┌─────────────┘          └────────────┐
        ▼                                     ▼
┌─────────────────┐                  ┌─────────────────┐
│  Patroni (PG A) │                  │  Patroni (PG B) │
│  leader         │◀── 流复制 WAL ──▶│  replica        │
└────────┬────────┘                  └────────┬────────┘
         ▲                                    ▲
         └──────────  HAProxy  ◀──── 应用连接 ──┘
```

Patroni 的核心机制：每个节点通过 DCS 竞争 **leader key**，持有租约的节点成为主库；主库定期续租，租约过期后备库发起选举；切换前尽量做 **fencing**，确保旧主库不再接受写入；通过 REST API 暴露状态，HAProxy 据此路由流量。

Patroni 的配置核心是 DCS 地址、`ttl`、`loop_wait` 与 `synchronous_mode`，再配上 `postgresql.data_dir` 与监听地址即可，完整字段见官方文档。

```bash
patronictl -c /etc/patroni.yml list
patronictl -c /etc/patroni.yml switchover --master pg-a --candidate pg-b
patronictl -c /etc/patroni.yml failover
```

`repmgr` 是更轻量的方案：`repmgr standby clone` 克隆备库、`repmgr standby register` 注册、`repmgr cluster show` 查看、`repmgr standby promote` 提升。它同样需要共享元数据（通常是主库中的 `repmgr` schema），但对 DCS、fencing、配置管理的支持弱一些。

## 4.3 故障转移与脑裂

```text
1. 监控/Patroni 发现主库心跳超时
        ▼
2. 确认 leader key 租约过期，集群仍有 quorum
        ▼
3. fencing：确保旧主库停止写入（STONITH/watchdog/网络隔离）
        ▼
4. 选出数据最完整的备库，提升为新主库
        ▼
5. 其他备库重新指向新主库，更新 HAProxy/VIP/DNS
        ▼
6. 旧主库恢复后作为新备库重建（pg_rewind 或 pg_basebackup）
```

**第 3 步是关键**。如果旧主库只是网络分区、实际仍在接受写入，就会出现**脑裂（split brain）**：两个节点都认为自己是主库，数据分叉。fencing 的手段包括电源隔离（IPMI/云 API 关机）、网络隔离（交换机端口、安全组）、watchdog 强制重启，以及 DCS 的 quorum 机制保证少数派无法选主。

> **行业实践**：不要用"脚本检测 + 直接 promote"自建高可用。没有 fencing 和 quorum 的自动切换，比不切换更危险。优先使用 Patroni 这类经过验证的方案，或直接使用云托管的多可用区实例。

# 5. 读写分离与一致性边界

## 5.1 延迟从哪里来

```text
主库提交 → WAL 本地落盘 → walsender 发送
        → 网络 → walreceiver 接收 → 备库落盘
        → startup 重放 → 可被查询看到
```

网络带宽不足或跨地域、备库磁盘慢或 CPU 饱和、主库写入突增、备库长查询触发 `hot_standby_feedback`，都会拉长延迟。主库看字节延迟，备库看 `now() - pg_last_xact_replay_timestamp()`。

## 5.2 一致性的几种边界

| 需求 | 方案 | 代价 |
|------|------|------|
| 最终一致，允许秒级延迟 | 直接读异步备库 | 最简单，可能读到旧数据 |
| 读到自己刚写的数据 | 写后一段时间读主库，或按业务键粘滞主库 | 实现简单，主库压力增加 |
| 强一致读 | 读主库，或 `remote_apply` 后读同步备库 | 延迟增加，备库故障影响写入 |
| 精确因果一致 | 记录写入 LSN，读备库前等待 `replay_lsn >= LSN` | 需要应用或代理支持 |

## 5.3 备库查询冲突

备库重放时可能需要清理旧版本，而备库上的长查询还在读这些版本。PG 等待一段时间后**强制取消备库查询**，报 `canceling statement due to conflict with recovery`。可以调 `max_standby_streaming_delay`，或开 `hot_standby_feedback` 让主库保留备库还在读的旧版本。

> **取舍**：`hot_standby_feedback = on` 能显著减少查询取消，但会让主库的 VACUUM 无法清理备库还需要的老版本，长事务在备库上跑得越久，主库膨胀越严重。报表类长查询更适合放到专用只读副本。

# 6. 常见错误与排错

| 现象 | 根因 | 处理 |
|------|------|------|
| 备库长时间 `catchup` 不进入 `streaming` | 网络、认证、WAL 缺失 | 查备库日志，确认 `primary_conninfo` 与 `pg_hba.conf` |
| `requested WAL segment has already been removed` | 备库落后太多，主库已清理 WAL | 用复制槽或增大 `wal_keep_size`，必要时重建备库 |
| `pg_wal` 持续增长 | 复制槽未推进、归档失败 | 查 `pg_replication_slots` 与 `archive_command` |
| 同步备库故障导致主库写入卡住 | `synchronous_standby_names` 指向不可用节点 | 调整同步列表或降级为异步 |
| 逻辑复制停止 | 主键冲突、约束违反、表结构不一致 | 修复数据后 `ALTER SUBSCRIPTION ... ENABLE` |
| 切换后应用连到旧主库 | VIP/DNS/HAProxy 未更新 | 检查路由层，缩短 TTL |
| 切换后数据分叉 | 缺少 fencing，旧主库仍在写 | 启用 watchdog/STONITH，检查 DCS quorum |
| 备库查询频繁被取消 | 重放冲突 | 调 `max_standby_streaming_delay` 或开 `hot_standby_feedback` |

```sql
SELECT pid, application_name, client_addr, state, sync_state, replay_lag FROM pg_stat_replication;
```

# 7. 练习与自检

**练习一**：用两个 Docker 容器搭建一主一备的流复制，用 `pg_basebackup -R` 初始化备库，并在主库插入数据后验证备库能查到。

**练习二**：切换为同步模式（`ANY 1`），停掉备库，观察主库写入行为与恢复时间。思考生产环境如何避免这种阻塞。

**练习三**：用逻辑复制把一张表从实例 A 同步到实例 B，故意在 B 上制造主键冲突，观察订阅状态与日志，然后修复并恢复订阅。

**自检清单**：

- 能画出物理复制的 WAL 流转路径，区分 `sent/write/flush/replay` 四个 LSN；
- 能解释同步复制各档位的丢数据风险与阻塞风险；
- 知道复制槽的作用与 WAL 堆积风险；
- 能列出逻辑复制不支持的至少四项内容。

# 8. 小结

| 主题 | 核心结论 |
|------|---------|
| 物理复制 | 复制 WAL 字节流，主备版本与架构必须一致 |
| 复制槽 | 防止 WAL 被提前清理，但长期离线会撑爆磁盘 |
| 同步复制 | `remote_apply` 一致性最强，但备库故障会阻塞写入 |
| 逻辑复制 | 表级、跨版本，但 DDL/序列/大对象不同步 |
| 高可用 | Patroni/repmgr + DCS + fencing + 路由层，缺一不可 |
| 读写分离 | 延迟不可避免，按业务选择最终一致、读己之写或强一致 |

有了复制，数据就不再只存在一台机器上。但"有副本"不等于"能恢复"——误删数据、错误更新会瞬间同步到所有备库，复制帮不上忙。下一篇进入**备份与监控**，讲清逻辑备份、物理备份、PITR 的完整链路，以及如何用指标和告警把故障挡在用户感知之前。