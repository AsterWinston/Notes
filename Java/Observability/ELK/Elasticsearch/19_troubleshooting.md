
# 1. 集群状态异常

```bash
# 查看健康
GET /_cluster/health
GET /_cluster/health?level=indices  # 每个索引健康

# Red: 有主分片未分配
# Yellow: 所有主分片 OK，部分副本未分配
# Green: 正常

# 查看未分配分片原因
GET /_cat/shards?v&h=index,shard,state,unassigned.reason

# 常见原因：
# INDEX_CREATED: 刚创建，等待分配
# CLUSTER_RECOVERED: 集群恢复中
# NODE_LEFT: 节点离开，副本未分配
# ALLOCATION_FAILED: 分配失败（磁盘满/分片数超限）
# REALLOCATED_REPLICA: 重新分配副本
```

# 2. Unassigned Shard

```bash
# 1. 查看详细原因
GET /_cluster/allocation/explain
{
  "index": "orders",
  "shard": 0,
  "primary": true
}

# 2. 解决方案：
# - 副本未分配 → 等待自动恢复
# - 磁盘满 → 清理磁盘或扩容
# - 分片超限 → 减少节点分片数或增加节点
# - 节点离线 → 恢复节点或强制重新分配

# 3. 强制分配（慎重！）
POST /_cluster/reroute?retry_failed=true
```

# 3. 写入被拒 (429)

```
# 错误信息：
TOO_MANY_REQUESTS/12/disk usage exceeded flood stage watermark

# 原因：磁盘使用超过 95%
# 解决：
# 1. 清理磁盘
# 2. 提高 watermarks（临时）
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.flood_stage": "97%"
  }
}

# 3. 删除旧索引
DELETE /old-index-*
```

# 4. OOM / GC 频繁

```bash
# 1. 查看 JVM
GET /_nodes/stats/jvm

# 2. 热点线程
GET /_nodes/hot_threads

# 3. 解决方案：
# - 增加堆内存（≤ 32GB）
# - 减少分片数
# - 关闭 fielddata（改用 doc_values）
# - 限制聚合桶数量
# - 增大 GC 阈值
```

# 5. 慢查询定位

```bash
# 1. 开启慢查询日志
PUT /orders/_settings
{
  "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.level": "info"
}

# 2. 查看正在执行的慢查询
GET /_tasks?actions=*search*&detailed=true

# 3. Kill 慢查询
POST /_tasks/task_id:xxx/_cancel

# 4. Profile 分析
GET /orders/_search
{ "profile": true, "query": { ... } }
```

# 6. 数据不一致

```bash
# 1. 检查分片分布
GET /_cat/shards/orders?v

# 2. 强制同步
POST /orders/_stats?level=shards

# 3. 重新分配副本
POST /_cluster/reroute?retry_failed=true

# 4. 必要时重建副本
# 关闭索引 → 修改副本数为 0 → 重新设置副本数
```

# 7. 快速诊断命令

```bash
# 集群状态
GET /_cluster/health
GET /_cat/nodes?v
GET /_cat/indices?v&s=store.size:desc

# 资源
GET /_nodes/stats/jvm,os,process
GET /_cat/thread_pool?v

# 分片
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# 任务
GET /_cat/tasks?v
GET /_tasks?actions=*search*,*write*
```
