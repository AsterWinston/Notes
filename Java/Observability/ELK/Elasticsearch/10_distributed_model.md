
# 1. 写入流程详解

```
Client → Node 1 (Coordinating)
  │
  │ Step 1: 路由计算
  │   shard = hash(_routing) % num_primary_shards
  │   shard = hash("1") % 3 = 1 → Primary 1 on Node 2
  │
  ├── Step 2: 转发到 Primary Shard (Node 2)
  │     │
  │     │ a. 写入 Translog (同步磁盘)
  │     │ b. 写入 Memory Buffer
  │     │ c. 转发到 Replica (并行)
  │     │
  │     ├──→ Replica on Node 3
  │     │     └── a. 写入 Translog, b. 写入 Memory Buffer
  │     │
  │     └── 等待副本确认 (wait_for_active_shards)
  │
  └── Step 3: 返回 Client (ACK)
```

## 1.1 wait_for_active_shards

```json
PUT /orders/_doc/1?wait_for_active_shards=all
{ "amount": 100.0 }

# all: 等待所有副本确认（最安全，最慢）
# 1: 仅主分片（最快，最不安全）
# quorum: (num_replicas / 2) + 1（默认）
```

# 2. Refresh — 内存到可搜索

```
Memory Buffer (不可搜索)
  │
  │ refresh (默认 1s)
  ▼
Segment (内存中，可搜索)
  │
  │ translog fsync
  ▼
Segment (文件系统缓存)

最终 → fsync → 磁盘
```

```json
# 手动刷新
POST /orders/_refresh

# 批量导入时关闭
PUT /orders/_settings
{ "refresh_interval": "-1" }
```

# 3. Translog — 持久化保障

```
Translog = Write-Ahead Log（类似 MySQL Redo Log）

作用：防止数据丢失
  - 写入 Memory Buffer 前先写 Translog
  - 宕机后从 Translog 恢复

Flush 触发条件：
  - Translog > 512MB
  - 每 30 分钟自动
  - 内存 Buffer 满

Flush 过程：
  1. Memory Buffer → 新 Segment (refresh)
  2. Segment fsync 到磁盘
  3. 清空 Translog
```

```json
# 调整 Translog
PUT /orders/_settings
{
  "translog": {
    "durability": "async",        // 异步刷盘（性能高，风险大）
    "sync_interval": "5s"
  }
}
```

# 4. 读取流程再深入

```
Client → Node 1 (Coordinating)
  │
  │ Step 1: Query Phase
  │   向所有相关分片发送查询
  │   各分片返回: (docId, _score, sort values) — 轻量
  │
  │ Step 2: Query Then Fetch
  │   协调节点归并排序
  │   取 Top N 个 docId
  │
  │ Step 3: Fetch Phase
  │   向对应分片获取完整文档
  │   返回 Client
```

## 4.1 preference — 路由偏好

```json
GET /orders/_search?preference=_local
{ }

# _local: 优先本地分片
# _primary: 仅主分片
# _replica: 仅副本分片
# _shards:2,3: 指定分片
# 自定义字符串: 相同字符串路由到相同分片
```

# 5. 分片分配策略

```json
PUT /orders/_settings
{
  "routing.allocation.include._tier": "hot",
  "routing.allocation.require.box_type": "ssd",
  "routing.allocation.exclude._name": "node-old",
  "routing.allocation.total_shards_per_node": 2
}
```

## 5.1 分片分配感知

```yaml
# elasticsearch.yml
cluster.routing.allocation.awareness.attributes: rack_id
cluster.routing.allocation.awareness.force.rack_id.values: rack1,rack2
```

**效果**：主分片和副本不会分配到同一个 rack，提高可用性。

# 6. 段合并策略

```json
# 强制合并到 1 个 segment（读优化）
POST /orders/_forcemerge?max_num_segments=1

# 查看段信息
GET /_cat/segments/orders?v
```

**合并策略**：`tiered`（默认），根据段大小分层合并。
