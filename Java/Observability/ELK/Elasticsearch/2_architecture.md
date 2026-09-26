
# 1. 概念层级

```
Cluster (集群)
  └── Node (节点, 1-N)
        └── Index (索引, 1-N)
              ├── Settings (配置)
              ├── Mapping (字段定义)
              └── Shard (分片)
                    ├── Primary Shard (主分片, 1-N, 写入时定)
                    └── Replica Shard (副本分片, 0-N, 可动态调整)
```

# 2. 节点角色

| 角色 | 配置 | 职责 |
|------|------|------|
| **Master-eligible** | `node.master: true` | 集群管理（索引创建/删除、分片分配） |
| **Data** | `node.data: true` | 数据存储、CRUD、搜索、聚合 |
| **Ingest** | `node.ingest: true` | 预处理管道（Pipeline） |
| **ML** | `node.ml: true` | 机器学习任务 |
| **Coordinating** | 所有角色 false | 请求路由、结果归并（无状态） |
| **Remote Cluster** | `node.remote_cluster_client: true` | 跨集群搜索（CCS） |

```yaml
# 生产推荐节点分工
# Master 节点（3 台，小巧）
node.roles: [master]

# Data 节点
node.roles: [data, ingest]

# Coordinating 节点
node.roles: []         # 所有角色默认 false
```

# 3. 分片模型

```
Index: orders (3 主分片, 1 副本)
┌──────────────────────────────────────────────────────┐
│  Primary 0 (Node A)     Replica 0 (Node B)           │
│  Primary 1 (Node B)     Replica 1 (Node C)           │
│  Primary 2 (Node C)     Replica 2 (Node A)           │
└──────────────────────────────────────────────────────┘

计算公式: shard = hash(_routing) % num_primary_shards
```

**关键规则**：
- 主分片数索引创建时确定，不可修改
- 副本分片数可动态调整
- 主分片和其副本不在同一节点
- 每个分片是一个完整的 Lucene 实例

# 4. 写入流程

```
Client → Node 1 (Coordinating)
  │
  │ 1. 计算路由: shard = hash(_routing) % num_primary_shards
  │
  ├──→ Node 2 (Primary Shard)
  │      │
  │      │ 2. 写入 Lucene（内存 buffer + translog）
  │      │
  │      │ 3. 转发到副本
  │      │
  │      ├──→ Node 3 (Replica Shard)
  │      │     └── 4. 写入副本
  │      │
  │      └── 5. 回复协调节点
  │
  └── 6. 回复 Client
```

## 4.1 Refresh — 数据可搜索

```
Refresh: 内存 Buffer → Lucene Segment（可搜索）

间隔: refresh_interval (默认 1s)
代价: 频繁 refresh 影响写入性能
优化: 批量导入时设置 refresh_interval=-1（导入完成后恢复）
```

## 4.2 Flush — 持久化到磁盘

```
Flush: 内存 Segment → 文件系统缓存 → fsync 磁盘

触发条件:
  - translog 达到 512MB
  - 每 30 分钟自动触发
  - 手动 POST /_flush
```

# 5. 读取流程

```
Client → Node 1 (Coordinating)
  │
  │ 1. Query Phase: 向所有相关分片发送查询
  │    返回: (docId, score, sort values)
  │
  │ 2. 协调节点排序和分页
  │
  │ 3. Fetch Phase: 按排序结果取前 N 个 docId
  │    向对应分片获取完整文档
  │
  └── 4. 返回结果
```

# 6. 段合并（Segment Merge）

Lucene 每 refresh 生成一个 Segment，多个 Segment 需要合并：

```
Segments: [S1] [S2] [S3] [S4] ...
              ↓ merge
         [S12]  [S3]  [S4] ...
              ↓ merge
         [S1234]  ← 一个更大的 Segment

合并策略: tiered (默认), 根据段大小分层合并
合并开销: CPU + IO，在低负载时自动执行
```

# 7. Master 选举

```
# 选举流程
1. Master-eligible 节点 ping 所有节点
2. 收集投票（Vote）
3. 超过半数 (quorum) 投票的节点成为 Master
4. 新 Master 发布集群状态

# 脑裂防护
discovery.seed_hosts: ["node1", "node2", "node3"]
cluster.initial_master_nodes: ["node1", "node2", "node3"]

# 最小 Master 节点数
discovery.zen.minimum_master_nodes = (master_eligible_nodes / 2) + 1
```
