etcd 的运维工作可以概括成四件事：**成员别丢、数据能恢复、空间别爆、故障能演练**。它不像无状态服务那样重启就完事，一次错误的成员删除、一次没验证的快照、一次全集群 defrag，都可能让 Kubernetes 直接失去写能力。本篇给出一套可执行的操作清单。

# 1. 成员管理

## 1.1 查看成员

```bash
export ETCDCTL_API=3
ENDPOINTS=https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379
CACERT=/etc/etcd/pki/ca.crt
CERT=/etc/etcd/pki/client.crt
KEY=/etc/etcd/pki/client.key

etcdctl --endpoints=$ENDPOINTS \
  --cacert=$CACERT --cert=$CERT --key=$KEY \
  member list -w table
```

`member list -w table` 输出成员 ID、状态（`started` 表示已加入并同步）、成员名、peer URL、client URL 和是否为 learner。成员名必须唯一，删除/更新时用 ID。

下文用 `etcdctl ...` 省略统一的 `--endpoints` 与证书参数，实际执行时补上即可。

## 1.2 增加成员

```bash
# 先作为不投票的学习节点加入，等同步完成后再 promote
etcdctl ... member add etcd-4 --peer-urls=https://10.0.0.4:2380 --learner
etcdctl ... endpoint status --cluster -w table   # 查看 learner 是否已同步
etcdctl ... member promote <member-id>          # 提升为正式成员
```

命令会打印新节点启动所需的 `ETCD_INITIAL_CLUSTER`、`ETCD_INITIAL_CLUSTER_STATE=existing` 等环境变量，在新节点上按这些值启动即可。

★ **关键认知**：`member add` 一旦成功，集群的 quorum 就按新规模计算。3 节点加第 4 个投票成员时，多数派从 2 变成 3；如果新节点迟迟不启动，集群可能暂时失去写能力。生产上优先用 learner 过渡，确认同步完成再 promote。

## 1.3 删除成员

```bash
etcdctl ... member list -w table      # 先确认要删的 ID
etcdctl ... member remove <member-id> # 从集群移除
```

删除顺序：先把该成员从负载均衡/客户端 endpoints 摘掉，再 `member remove`，然后停止它的 etcd 进程，最后保留或归档 data-dir。删除 Leader 会触发重新选主，短暂不可写是正常的；不要在集群已经不健康时删成员，否则可能直接失去多数派。

## 1.4 替换故障成员

成员彻底损坏时，正确流程是"先删后加"：`member remove <bad-id>`，再用相同或新的 name 执行 `member add etcd-2 --peer-urls=...`，清空该节点的旧数据目录，最后用打印出的 `initial-cluster` 启动。

⚠ **关键认知**：旧 data-dir 里保存的是**旧成员 ID** 和旧集群状态。直接拿它启动，节点会以旧身份尝试连接，出现 `member ... already exists` 或集群 ID 不匹配。替换成员前先备份并移走旧目录。

# 2. 快照备份

## 2.1 手工备份

```bash
BACKUP_DIR=/backup/etcd
mkdir -p $BACKUP_DIR
etcdctl ... snapshot save $BACKUP_DIR/etcd-$(date +%Y%m%d-%H%M%S).db
```

建议从 **Follower** 做备份，避免给 Leader 增加额外 IO。快照是某一 revision 的一致视图，不需要停写。

## 2.2 验证快照

```bash
etcdctl snapshot status $BACKUP_DIR/etcd-20261106-020000.db -w table
```

输出包含快照哈希、对应的全局 revision、key 数量和快照大小。

★ **关键认知**：**没有验证过的备份等于没有备份**。每次备份后至少执行一次 `snapshot status`，定期还要在隔离环境做一次真实恢复演练。只看到文件存在，不代表文件可恢复。

## 2.3 定期备份

```bash
# /etc/cron.d/etcd-backup：每天凌晨 2 点备份
0 2 * * * root /usr/local/bin/etcdctl ... snapshot save /backup/etcd/etcd-$(date +\%Y\%m\%d).db
```

保留策略：本地 7 天、异地 30 天；备份文件不要和 etcd 数据放同一块磁盘；备份目录设置容量告警；定期抽样恢复到测试集群，验证 revision 和 key 数量。

# 3. 快照恢复

## 3.1 恢复流程

恢复是**全集群操作**，不是单节点热修复。假设三个节点全部损坏，从同一份快照重建：

```bash
# 1. 所有节点停止 etcd，移走旧数据并创建空目录
sudo systemctl stop etcd
sudo mv /var/lib/etcd /var/lib/etcd.bak.$(date +%Y%m%d%H%M)
sudo mkdir -p /var/lib/etcd && sudo chown -R etcd:etcd /var/lib/etcd

# 2. 每个节点用同一份快照 restore，只改 --name 与 peer URL
etcdctl snapshot restore /backup/etcd/etcd-20261106-020000.db \
  --name etcd-1 \
  --data-dir /var/lib/etcd \
  --initial-cluster etcd-1=https://10.0.0.1:2380,etcd-2=https://10.0.0.2:2380,etcd-3=https://10.0.0.3:2380 \
  --initial-advertise-peer-urls https://10.0.0.1:2380 \
  --initial-cluster-token etcd-restored-2026

# 3. 启动所有节点
sudo systemctl start etcd
```

★ **关键认知**：`snapshot restore` 会生成**新的成员 ID 和新的集群**，不是把旧集群"原地复活"。因此：

- 所有节点必须用同一份快照恢复，不能一部分用旧数据、一部分用快照。
- 客户端 endpoints 仍可沿用原地址，但集群 ID 已经变化。
- 恢复后 revision 会回到快照点，快照之后的写入会丢失，这是恢复的固有代价。

## 3.2 恢复后验证

```bash
etcdctl ... endpoint health --cluster
etcdctl ... endpoint status --cluster -w table
etcdctl ... member list -w table
etcdctl ... get /registry/ --prefix --limit=10   # 抽查关键 key
```

如果 Kubernetes 使用这份 etcd，还要确认 kube-apiserver 能正常 List-Watch，必要时重启 apiserver 的连接。

## 3.3 常见恢复错误

| 错误 | 原因 | 处理 |
|------|------|------|
| `data-dir is not empty` | 恢复目录里还有旧数据 | 移走旧目录，确保为空 |
| 集群起不来、一直选主 | 三个节点 `initial-cluster` 不一致 | 三份参数必须完全一致 |
| `cluster ID mismatch` | 混用了旧数据与快照 | 全部节点统一重来 |
| 恢复后 key 少了 | 快照时间点较旧 | 属正常，确认备份频率 |

# 4. 压缩与碎片整理

## 4.1 compact：删除历史版本

```bash
REV=$(etcdctl ... endpoint status --write-out=json | jq -r '.[0].Status.header.revision')
etcdctl ... compact "$REV"   # 压缩到当前 revision，之前的历史不可再读
```

自动压缩用启动参数 `auto-compaction-mode: periodic` 与 `auto-compaction-retention: "1h"`；`periodic` 按时间保留，`revision` 按 revision 数量保留。保留时间要大于"最长断线时间"，否则客户端断线久了，Watch 起点会被压缩，只能全量重建。

## 4.2 defrag：回收磁盘空间

compact 只标记历史可删，BoltDB 文件里的空洞不会自动缩小，需要 defrag：

```bash
etcdctl ... --endpoints=https://10.0.0.2:2379 defrag   # 观察 DB SIZE 与 DB SIZE IN USE
```

defrag 期间该成员会阻塞读写，因此一次只对一个成员执行，避开业务高峰；不要在多数成员上同时执行，否则集群会短暂失去 quorum；先在 Follower 上做，最后再考虑 Leader，执行前确认集群健康、备份可用。

## 4.3 NOSPACE 告警处理

当 DB 超过 `--quota-backend-bytes`，etcd 会进入只读模式，写入返回：

```text
etcdserver: mvcc: database space exceeded
```

处理步骤：

```bash
etcdctl ... alarm list                    # 1. 查看告警
REV=$(etcdctl ... endpoint status --write-out=json | jq -r '.[0].Status.header.revision')
etcdctl ... compact "$REV"                # 2. 压缩到当前 revision
etcdctl ... --endpoints=https://10.0.0.2:2379 defrag   # 3. 逐节点碎片整理
etcdctl ... alarm disarm                  # 4. 解除告警，恢复写入
```

★ **关键认知**：**只调大 quota 不能消除 NOSPACE 告警**。告警是持久化的，必须 compact + defrag 释放实际空间后再 `alarm disarm`。如果只是把配额从 2GB 调到 4GB，写可能恢复，但空间问题会在更晚的时候以更严重的方式回来。

# 5. 监控与告警

## 5.1 关键指标

| 指标 | 含义 | 关注点 |
|------|------|--------|
| `etcd_server_has_leader` | 是否有 Leader | 0 表示集群不可写 |
| `etcd_server_leader_changes_seen_total` | Leader 变更次数 | 频繁变更说明不稳定 |
| `etcd_disk_wal_fsync_duration_seconds` | WAL fsync 延迟 | P99 应小于 10ms |
| `etcd_mvcc_db_total_size_in_bytes` | DB 大小 | 与配额比较 |
| `etcd_server_proposals_failed_total` | 提案失败数 | 持续增长说明 quorum 或磁盘问题 |
| `etcd_network_peer_round_trip_time_seconds` | peer RTT | 网络抖动会拖慢写 |

## 5.2 常用 PromQL

```promql
# WAL fsync P99
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
# 存储使用率
etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes
# 最近 15 分钟 Leader 变更
increase(etcd_server_leader_changes_seen_total[15m])
# 提案失败速率
rate(etcd_server_proposals_failed_total[5m])
```

告警阈值建议：无 Leader 持续 1m 为 P0；存储超 90% 持续 10m 为 P0；WAL fsync P99 > 100ms 持续 10m 为 P1；15m 内 Leader 变更 > 3 次为 P1；提案失败速率 > 0 持续 5m 为 P1。

## 5.3 日志里要看什么

```bash
journalctl -u etcd --since "1 hour ago" | grep -Ei \
  "leader changed|slow fdatasync|space exceeded|failed to send|took too long"
```

`leader changed` 偶发正常，频繁要查；`slow fdatasync` 是磁盘慢；`database space exceeded` 是超配额；`failed to send out heartbeat` 是 peer 网络问题；`apply request took too long` 可能是大事务或磁盘问题。

# 6. 容量规划

| 维度 | 建议 |
|------|------|
| 节点数 | 生产用 3 或 5，奇数；不要用 4 |
| DB 大小 | 控制在 4-8GB 以内，超过 8GB 风险明显上升 |
| 磁盘 | SSD/NVMe，fsync P99 < 10ms，独立于日志盘 |
| 网络 | 同地域低延迟，peer RTT 尽量 < 10ms |
| 单 value | KB 级，最大请求默认约 1.5MB |
| Watch 数 | 按客户端数量估算，避免每请求一个 Watch |
| 快照 | 容量足够保存至少 7 天，异地在另一份 |
| 文件描述符 | `LimitNOFILE` 至少 65536 |

容量估算可以从 key 数量、平均 value 大小、历史保留时间三个维度做：

```text
DB 大小 ≈ 当前 key 总大小 + 压缩窗口内的历史版本 + BoltDB 碎片
```

如果每天新增 10 万条、平均 1KB、保留 1 小时历史，增量并不大；真正撑大 DB 的往往是长期不清理的 Kubernetes 事件、Completed Job、旧 Secret 和大量小 key。

# 7. 故障演练

| 演练 | 操作 | 预期 |
|------|------|------|
| 挂 Follower | 停止一个 Follower | 读写正常，无选主 |
| 挂 Leader | 停止 Leader | 数秒内重新选主，客户端重试成功 |
| 失去多数派 | 停止两个节点 | 不再接受写，恢复多数派后自动恢复 |
| 磁盘写满 | 填满数据盘 | 触发 NOSPACE，写入失败，告警触发 |
| 快照恢复 | 在隔离环境恢复 | 数据完整，key 数量与 revision 符合预期 |
| 证书过期 | 使用过期证书连接 | 明确报 x509 错误，替换后恢复 |
| 网络分区 | 隔离一个节点 | 少数派不产生新 Leader，多数派继续服务 |

演练原则：

1. 先在测试环境做，再在生产低峰期做。
2. 每次只动一个节点，确认集群恢复健康后再进行下一步。
3. 演练前必须有一份**已验证**的快照。
4. 记录选主耗时、客户端错误率、恢复步骤，形成 runbook。

★ **关键认知**：etcd 的高可用不是"三个节点都活着"这么简单，而是"任意时刻至少有多数派能通信且磁盘可写"。网络分区、磁盘满、时钟漂移、证书过期，都会让三个进程看起来都在，但集群实际不可用。

# 8. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| 备份文件很小或为 0 | 备份命令失败但 cron 没检查退出码 | 备份脚本加 `set -e` 与 `snapshot status` |
| 恢复后集群选不出主 | `initial-cluster` 或 peer URL 不一致 | 三节点参数逐字对齐 |
| defrag 后客户端超时 | defrag 阻塞了该成员 | 逐节点执行，避开高峰 |
| compact 报错 | revision 超过当前值 | 用 `endpoint status` 取当前 revision |
| 告警一直不消失 | 只 compact 没 defrag，或没 disarm | 三步都做完 |
| 删除成员后写失败 | 删完后剩余节点不足多数派 | 先恢复节点再调整成员 |
| 快照恢复到错误集群 | 用了别的环境快照 | 恢复前核对集群名与 key 前缀 |
| 备份占满磁盘 | 没有保留策略 | 定期清理 + 容量告警 |

⚠ **关键认知**：etcd 的 data-dir 是"集群身份 + 数据"的绑定。任何涉及删除、替换、恢复的操作，第一步都应该是**停机 + 备份 + 移走旧目录**，而不是直接 `rm -rf`。生产事故里最贵的不是磁盘，是那份被误删的 WAL。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 成员 | 加节点优先 learner，删节点确认 quorum |
| 替换 | 先 remove 再加，旧 data-dir 不能复用 |
| 备份 | 从 Follower 做，快照必须验证 |
| 恢复 | 全集群、同一快照、空 data-dir、参数一致 |
| compact | 删除历史版本，保留时间大于最长断线 |
| defrag | 回收磁盘，逐节点执行，避开高峰 |
| NOSPACE | compact + defrag + disarm，三步缺一不可 |
| 监控 | Leader、fsync、DB 大小、提案失败是核心 |
| 演练 | 挂节点、丢多数派、磁盘满、快照恢复都要练 |

下一篇是收尾的面试与总结，用一张知识地图把定位、Raft、API、锁、服务发现和运维串起来，给出高频自检题、生产检查清单，以及 etcd 与 Redis、ZooKeeper 的选型结论。
