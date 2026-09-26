
# 1. 节点角色配置

```yaml
# Master-only 节点（小巧，仅管理）
node.roles: [master]
# 资源: 2C4G，不存数据

# Data 节点（存储 + 计算）
node.roles: [data, ingest]
# 资源: 8C32G+，SSD

# Coordinating-only 节点（无状态路由）
node.roles: []
# 资源: 4C8G，无磁盘需求

# ML 节点（机器学习）
node.roles: [ml]
# 资源: 8C16G+，GPU 可选
```

## 1.1 生产推荐拓扑

```
3 Master Nodes (2C4G)  → 集群管理
3+ Data Nodes (8C32G+) → 数据存储 + 搜索
2+ Coordinating Nodes (4C8G) → 负载均衡入口
  × (X-Pack License) ML Node
```

# 2. Hot-Warm-Cold 冷热架构

```yaml
# Hot 节点（SSD，高性能）
node.attr.tier: hot
node.attr.box_type: ssd

# Warm 节点（HDD，大容量）
node.attr.tier: warm
node.attr.box_type: hdd

# Cold 节点（对象存储/归档）
node.attr.tier: cold
```

```json
# 索引分配
PUT /logs-2024.01.15
{
  "settings": {
    "routing.allocation.include._tier": "hot",
    "routing.allocation.require.box_type": "ssd"
  }
}
```

# 3. ILM — 索引生命周期管理

```json
PUT /_ilm/policy/logs_policy
{
  "phases": {
    "hot": {
      "actions": {
        "rollover": {
          "max_size": "50GB",
          "max_age": "1d"
        }
      }
    },
    "warm": {
      "min_age": "3d",
      "actions": {
        "allocate": { "require": { "data": "warm" } },
        "forcemerge": { "max_num_segments": 1 },
        "shrink": { "number_of_shards": 1 }
      }
    },
    "cold": {
      "min_age": "30d",
      "actions": {
        "allocate": { "require": { "data": "cold" } }
      }
    },
    "delete": {
      "min_age": "90d",
      "actions": { "delete": {} }
    }
  }
}

# 关联到模板
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}
```

# 4. Rollover

```json
# 创建带别名的索引
PUT /logs-000001
{
  "aliases": { "logs": { "is_write_index": true } }
}

# 自动滚动
POST /logs/_rollover/logs-000002
{
  "conditions": {
    "max_age": "1d",
    "max_docs": 10000000,
    "max_size": "50gb"
  }
}
```

# 5. 分片管理

```json
# 查看分片分配
GET /_cat/shards?v
GET /_cluster/allocation/explain

# 手动移动分片
POST /_cluster/reroute
{
  "commands": [
    {
      "move": {
        "index": "orders", "shard": 0,
        "from_node": "node-old", "to_node": "node-new"
      }
    }
  ]
}

# 修改副本数
PUT /orders/_settings
{ "number_of_replicas": 2 }
```

## 5.1 分片数公式

```
单分片推荐大小: 10-50GB（搜索）/ 50-100GB（日志）
分片数 ≈ 总数据量 / 单分片大小

示例:
  每日 100GB 日志，保留 7 天 = 700GB
  单分片 50GB → 约 14 个主分片
  每个节点 ≤ 600 分片（总主+副）
```

# 6. 集群健康

```bash
# 查看健康
GET /_cluster/health

# green  — 所有分片已分配
# yellow — 所有主分片已分配，部分副本未分配
# red   — 有主分片未分配

# 等待健康
GET /_cluster/health?wait_for_status=yellow&timeout=50s

# 查看未分配分片
GET /_cat/shards?v&h=index,shard,state,unassigned.reason
```

# 7. 水印控制

```yaml
# elasticsearch.yml
cluster.routing.allocation.disk.watermark.low: 85%       # 不分配新分片
cluster.routing.allocation.disk.watermark.high: 90%      # 开始迁移分片
cluster.routing.allocation.disk.watermark.flood_stage: 95% # 只读
```

```json
# 动态修改
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "80%"
  }
}
```
