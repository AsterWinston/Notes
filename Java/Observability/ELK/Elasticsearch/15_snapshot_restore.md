
# 1. Snapshot Repository

```json
# 注册文件系统仓库
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups/es_backup",
    "compress": true,
    "chunk_size": "1g",
    "max_snapshot_bytes_per_sec": "50mb",
    "max_restore_bytes_per_sec": "100mb"
  }
}

# 注册 S3 仓库
PUT /_snapshot/s3_backup
{
  "type": "s3",
  "settings": {
    "bucket": "es-backups",
    "region": "us-east-1",
    "base_path": "snapshots"
  }
}
```

# 2. 创建快照

```json
# 创建快照
PUT /_snapshot/my_backup/snapshot_20240115?wait_for_completion=true
{
  "indices": "orders,products",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

# 3. 查看快照

```json
# 所有快照
GET /_snapshot/my_backup/_all

# 单个快照详情
GET /_snapshot/my_backup/snapshot_20240115

# 快照状态
GET /_snapshot/my_backup/snapshot_20240115/_status
```

# 4. 恢复快照

```json
# 恢复全部
POST /_snapshot/my_backup/snapshot_20240115/_restore

# 恢复指定索引
POST /_snapshot/my_backup/snapshot_20240115/_restore
{
  "indices": "orders",
  "rename_pattern": "orders",
  "rename_replacement": "orders_restored",
  "include_global_state": false
}
```

# 5. SLM — 自动快照

```json
# 创建 SLM 策略
PUT /_slm/policy/nightly_backup
{
  "schedule": "0 30 2 * * ?",      // 每天 2:30AM
  "name": "<nightly-{now/d}>",
  "repository": "my_backup",
  "config": {
    "indices": ["orders", "products"],
    "ignore_unavailable": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 30
  }
}
```

# 6. 增量快照原理

```
Snapshot 1: [A, B, C]     → 全量
Snapshot 2: [A, B, C, D]  → 仅新增 D（增量）
Snapshot 3: [B, C, D, E]  → 删除 A，新增 E

原理：ES 对比每个段的 hash，只备份变化的段。
```

# 7. 跨集群复制 (CCR)

```json
# 在 Follower 集群上
PUT /orders-ccr/_ccr/follow
{
  "remote_cluster": "remote_cluster_name",
  "leader_index": "orders",
  "max_read_request_operation_count": 5000,
  "max_retry_delay": "60s"
}

# 暂停/恢复
POST /orders-ccr/_ccr/pause_follow
POST /orders-ccr/_ccr/resume_follow
```

# 8. Reindex — 数据迁移

```json
POST /_reindex
{
  "source": {
    "index": "orders_old"
  },
  "dest": {
    "index": "orders_new"
  },
  "script": {
    "source": "ctx._source.amount = ctx._source.amount * 1.1"
  }
}

# 跨集群 reindex
POST /_reindex
{
  "source": {
    "remote": {
      "host": "http://old-cluster:9200"
    },
    "index": "orders"
  },
  "dest": {
    "index": "orders"
  }
}
```
