# 1. etcd 概述与定位

## 1.1 etcd 是什么

etcd 是 CoreOS 开发的开源分布式键值存储系统，使用 Go 语言编写。它是 Kubernetes 集群的**唯一数据源（Single Source of Truth）**——所有集群状态数据（Pod、Service、ConfigMap、Node 等）都以 key-value 形式存储在 etcd 中。

etcd 在 CNCF 的毕业时间甚至早于 Kubernetes（2018 年 11 月毕业），是云原生生态的基础设施级项目。

### 1.1.1 Kubernetes 中存储在 etcd 的数据

```
/registry/
├── pods/
│   ├── default/nginx-deployment-7d5c8f-abc/
│   ├── production/api-server-xyz/
│   └── ...
├── services/
│   ├── default/kubernetes/
│   ├── production/api-service/
│   └── ...
├── deployments/
├── configmaps/
├── secrets/
├── namespaces/
├── nodes/
├── persistentvolumes/
├── serviceaccounts/
├── roles/
├── rolebindings/
├── clusterroles/
└── ...
```

## 1.2 etcd 在 Kubernetes 中的关键角色

### 1.2.1 Watch 机制的基础

Kubernetes 的控制循环（List-Watch）机制完全依赖 etcd 的 Watch 功能。当一个控制器需要监听 Pod 变化时：

```
Controller → API Server (Watch) → etcd (Watch)

etcd 的每次 key 变更都会产生一个事件，通过 Watch 流推送给所有订阅者
```

如果没有 etcd 的 Watch 能力，Kubernetes 控制器只能通过轮询来检测变化——在大规模集群中这将导致 API Server 压力剧增。

### 1.2.2 乐观并发控制

etcd 的 `ModRevision`（全局递增版本号）是 Kubernetes 乐观并发控制的基础：

```yaml
# 每个 Kubernetes 对象都有一个 resourceVersion
metadata:
  resourceVersion: "12345"
```

当控制器需要更新一个对象时，携带当前的 `resourceVersion`。如果 etcd 中的实际版本已经变化（被其他控制器先修改了），更新会被拒绝（409 Conflict），控制器需要重新获取最新版本后重试。

# 2. Raft 共识算法

## 2.1 为什么需要 Raft

分布式系统中，多个节点如何就"某个值是什么"达成一致，是一个经典问题。etcd 使用 Raft 算法（2013 年提出，比 Paxos 更易理解）来解决这个问题。

Raft 将一致性问题分解为三个子问题：
1. **领导者选举（Leader Election）**：集群选出一个 Leader，所有写操作通过 Leader 进行
2. **日志复制（Log Replication）**：Leader 将日志条目复制到所有 Follower
3. **安全性（Safety）**：确保任意时刻最多有一个 Leader，且已提交的日志不会被覆盖

## 2.2 etcd 集群中的角色

```
┌──────────────────────────────────────────────┐
│                etcd Cluster (3 节点)           │
│                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Leader  │  │ Follower │  │ Follower │   │
│  │          │◄─┤          │◄─┤          │   │
│  │ 接收写入  │  │ 复制日志  │  │ 复制日志  │   │
│  │ 发起提交  │  │ 投票     │  │ 投票     │   │
│  └──────────┘  └──────────┘  └──────────┘   │
│                                                │
│  读写行为：                                     │
│  - 写操作：只能通过 Leader（强一致性）            │
│  - 读操作：可以从任意节点（默认线性一致性读）       │
└──────────────────────────────────────────────┘
```

## 2.3 Raft 的写操作流程

```
1. 客户端向 Leader 发送写请求
         │
         ▼
2. Leader 将请求追加到本地日志（未提交）
         │
         ▼
3. Leader 并行向所有 Follower 发送 AppendEntries RPC
         │
         ▼
4. Follower 将日志条目追加到本地日志，回复 ACK
         │
         ▼
5. Leader 收到多数派（Quorum = N/2+1）的 ACK
         │
         ▼
6. Leader 提交日志（应用到状态机），回复客户端成功
         │
         ▼
7. Leader 在下一次心跳中告知 Follower 提交进度
```

**Quorum（法定人数）**：对于 3 节点集群，需要至少 2 个节点（3/2+1=2）存活才能正常写入。这就是 N=2F+1 的含义——3 个节点容忍 1 个故障。

## 2.4 领导者选举

```
初始状态（所有节点都是 Follower）
         │
    ┌────▼────┐
    │ 随机超时  │  每个 Follower 有一个随机选举超时（150-300ms）
    └────┬────┘
         │
    ┌────▼────┐
    │ 发起选举  │  第一个超时的节点变为 Candidate，增加 term
    └────┬────┘
         │
    ┌────▼────┐
    │ 请求投票  │  Candidate 向其他节点发送 RequestVote RPC
    └────┬────┘
         │
    ┌────▼────┐
    │  选举结果  │
    ├──────────┤
    │ 获得多数票 → 成为 Leader，开始发送心跳
    │ 票数不足  → 回到 Follower，等待下一次超时
    │ 平票      → 进入下一轮选举（term + 1）
    └──────────┘
```

# 3. etcd 数据模型与存储

## 3.1 物理存储结构

etcd 使用 BoltDB（一个 Go 语言编写的嵌入式 B+ 树数据库）作为持久化存储引擎。数据目录结构：

```
/var/lib/etcd/
├── member/
│   ├── snap/
│   │   ├── 0000000000000001-0000000000001000.snap  (快照文件)
│   │   └── ...
│   └── wal/
│       ├── 0000000000001001-0000000000002000.wal   (WAL 日志文件)
│       └── ...
```

**WAL（Write-Ahead Log）**：所有写操作先记录到 WAL 文件，然后才应用到 BoltDB。这是 etcd 保证数据持久性的关键。

**Snapshot（快照）**：etcd 定期生成快照，压缩 WAL 日志。快照是 BoltDB 的完整备份。

## 3.2 Key-Value 与 MVCC

etcd 使用 MVCC（多版本并发控制）存储数据：

```
Key: /registry/pods/default/nginx
  ├── Revision 100: {apiVersion: v1, kind: Pod, ...} (创建)
  ├── Revision 150: {apiVersion: v1, kind: Pod, status: Running, ...} (状态更新)
  └── Revision 200: {apiVersion: v1, kind: Pod, deletionTimestamp: ...} (删除标记)

每个 Revision 是一个全局递增的版本号
可以查询任意历史版本的记录
```

MVCC 使得 etcd 能够：
- 支持 Watch（从某个 Revision 开始监听变更）
- 支持历史版本查询
- 支持事务（CAS: Compare-And-Swap 基于 ModRevision）

## 3.3 存储空间管理

```bash
# 查看 etcd 存储空间
etcdctl endpoint status --cluster -w table

# 输出示例：
# +--------------------------+-----------------+---------+---------+--------+
# |        ENDPOINT          |      VERSION    | DB SIZE | IS LEADER| RAFT TERM |
# +--------------------------+-----------------+---------+---------+--------+
# | https://10.0.0.1:2379    | 3.5.14          | 256 MB  | true     | 42      |
# | https://10.0.0.2:2379    | 3.5.14          | 254 MB  | false    | 42      |
# +--------------------------+-----------------+---------+---------+--------+
```

### 3.3.1 碎片整理

长期运行的集群中，etcd 存在磁盘碎片（BoltDB 的 B+ 树空洞）：

```bash
# 检查碎片情况
etcdctl endpoint status --cluster -w table

# 执行碎片整理（对 Leader 执行会触发选举）
etcdctl defrag --endpoints=https://10.0.0.2:2379

# 逐个节点执行（注意避开 Leader）
for endpoint in https://10.0.0.1:2379 https://10.0.0.2:2379 https://10.0.0.3:2379; do
  etcdctl defrag --endpoints=$endpoint
done
```

## 3.4 存储配额与告警

```bash
# 检查存储配额
etcdctl endpoint status --write-out=json

# etcd 默认有 2GB 的存储上限
# 超过上限后，etcd 进入只读模式（所有写操作被拒绝）

# 调整配额（通过启动参数）
# --quota-backend-bytes=4294967296  (4GB)

# 触发空间告警阈值
# --experimental-warning-apply-duration=100ms
```

**为什么 etcd 有存储上限？** 因为 etcd 的数据完全保存在内存中（通过 mmap 映射 BoltDB 文件）。2GB 的 BoltDB 文件在内存中的映射区域会占用 2GB 虚拟内存。过大的存储会导致内存压力、启动缓慢、备份恢复耗时。

# 4. etcd 运维实战

## 4.1 备份与恢复

### 4.1.1 快照备份

```bash
# 创建快照
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 验证快照
etcdctl snapshot status /backup/etcd-snapshot-20240115.db --write-out=table

# 查看快照中的 Key 数量
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | abc123.. |   50000  |    12345   |   256 MB   |
# +----------+----------+------------+------------+
```

### 4.1.2 从快照恢复

```bash
# 1. 在所有 etcd 节点停止 etcd
systemctl stop etcd

# 2. 在所有节点清理现有数据
mv /var/lib/etcd /var/lib/etcd.bak

# 3. 在每个节点恢复快照
etcdctl snapshot restore /backup/etcd-snapshot-20240115.db \
  --name etcd-1 \
  --initial-cluster etcd-1=https://10.0.0.1:2380,etcd-2=https://10.0.0.2:2380,etcd-3=https://10.0.0.3:2380 \
  --initial-advertise-peer-urls https://10.0.0.1:2380 \
  --data-dir /var/lib/etcd

# 4. 重启 etcd
systemctl start etcd
```

### 4.1.3 定期备份 CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
spec:
  schedule: "0 2 * * *"        # 每天凌晨 2 点
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: bitnami/etcd:3.5
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
                --endpoints=https://etcd:2379 \
                --cacert=/certs/ca.crt \
                --cert=/certs/etcd-client.crt \
                --key=/certs/etcd-client.key
            volumeMounts:
            - name: backup
              mountPath: /backup
            - name: certs
              mountPath: /certs
              readOnly: true
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
          - name: certs
            secret:
              secretName: etcd-client-certs
          restartPolicy: OnFailure
```

## 4.2 监控 etcd

### 4.2.1 关键 PromQL 指标

```promql
# etcd Leader 变更（最近 1 小时）
changes(etcd_server_leader_changes_seen_total[1h])

# Raft 提案延迟（P99）
histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))

# WAL 同步延迟（P99）
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# 存储使用率
(etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes) * 100

# 活跃的 Watch 数量
sum(etcd_debugging_mvcc_watcher_total)

# RPC 速率
rate(grpc_server_handled_total[5m])
```

### 4.2.2 关键告警规则

```yaml
groups:
- name: etcd
  rules:
  - alert: EtcdHighFsyncDuration
    expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.5
    for: 10m
    annotations:
      summary: "etcd WAL fsync 延迟 > 500ms"
      
  - alert: EtcdLeaderChanges
    expr: rate(etcd_server_leader_changes_seen_total[15m]) > 0
    annotations:
      summary: "etcd Leader 最近 15 分钟内发生变更"
      
  - alert: EtcdDatabaseQuotaExceeded
    expr: (etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes) > 0.9
    annotations:
      summary: "etcd 存储超过配额的 90%"
```

## 4.3 性能调优

### 4.3.1 磁盘要求

etcd 的 Raft 协议要求日志条目在返回客户端之前必须被持久化到磁盘（fsync）。因此磁盘延迟直接影响 etcd 的写入性能：

```
推荐配置：
- SSD（NVMe 更好），IOPS ≥ 5000
- 独立磁盘，不与 Kubernetes 日志/容器数据共享 I/O
- fio 测试：fsync 延迟 P99 < 10ms

# 磁盘测试命令
fio --rw=write --ioengine=sync --fdatasync=1 --directory=/var/lib/etcd \
    --size=22m --bs=2300 --name=mytest
```

### 4.3.2 关键启动参数

```bash
etcd \
  --name=etcd-1 \
  --data-dir=/var/lib/etcd \
  --snapshot-count=10000 \                # 每 10000 次写生成快照
  --heartbeat-interval=100 \              # Leader 心跳间隔 (ms)
  --election-timeout=1000 \               # 选举超时 (ms)
  --quota-backend-bytes=8589934592 \      # 存储配额 8GB (生产建议 4-8GB)
  --max-request-bytes=1572864 \           # 最大请求大小 1.5MB
  --auto-compaction-mode=periodic \       # 自动压缩模式
  --auto-compaction-retention=1h \        # 保留 1 小时的历史版本
  --max-txn-ops=1000                      # 事务中最大操作数
```

### 4.3.3 自动压缩

etcd 的 MVCC 会积累历史版本，必须定期压缩（compact）以释放空间：

```bash
# 手动压缩到指定 Revision
etcdctl compact 50000

# 自动压缩（通过启动参数）
# --auto-compaction-mode=periodic
# --auto-compaction-retention=1h

# 查看当前 Revision（通过 Prometheus 指标）
# etcd_mvcc_db_total_size_in_bytes
```

# 5. etcd 在 Kubernetes 中的配置

## 5.1 kube-apiserver 与 etcd 的连接

```bash
# kube-apiserver 的启动参数
kube-apiserver \
  --etcd-servers=https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379 \
  --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt \
  --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt \
  --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key \
  --etcd-prefix=/registry \                    # etcd key 前缀
  --etcd-compaction-interval=5m \              # 压缩间隔
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml  # 静态加密
```

## 5.2 Secret 静态加密

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}   # 兜底：允许读取未加密的老数据
```

启用静态加密后，Secret 在 etcd 中以加密形式存储。执行以下命令加密已存在的 Secret：

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

# 6. 常见问题诊断

## 6.1 etcd 性能问题

```bash
# 症状：etcd 请求延迟高

# 检查
etcdctl endpoint health --cluster -w table

# 常见原因：
# 1. 磁盘慢（HDD 而非 SSD）
# 2. 网络延迟高
# 3. etcd 数据文件过大（需要压缩）
# 4. 大量 Watch 订阅

# 解决：
# - 使用 SSD
# - 碎片整理：etcdctl defrag
# - 自动压缩：--auto-compaction-mode=periodic
# - 减少不必要的 Watch
```

## 6.2 存储空间耗尽

```bash
# 症状：写操作失败，etcd 进入只读模式

# 立即操作：
# 1. 执行 compact
rev=$(etcdctl endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
etcdctl compact $rev

# 2. 碎片整理
etcdctl defrag --endpoints=<endpoint>

# 3. 删除不必要的 Kubernetes 资源（大量无用的 Secret、ConfigMap、Completed Job）
```

## 6.3 etcd 集群脑裂

脑裂极少发生（Raft 协议保证了安全性），但如果出现（如网络分区 + 超时 → 两个 Leader）：

```bash
# 检查每个节点的 Leader 状态
etcdctl endpoint status --cluster -w table

# 如果发现两个 Leader：
# 1. 停止所有 etcd 节点
# 2. 从最新快照恢复（见 4.1）
# 3. 重启所有节点
```
