
# 1. 概述

Stack Management 是 Kibana 的**管理控制台**，提供对 Elastic Stack 的配置和管理能力。

```
Stack Management 分类:
  ├── Elasticsearch
  │     ├── Index Management
  │     ├── Index Templates
  │     ├── Index Lifecycle Policies (ILM)
  │     └── Watcher (Advanced Alerting)
  ├── Kibana
  │     ├── Data Views
  │     ├── Saved Objects
  │     ├── Spaces
  │     ├── Advanced Settings
  │     └── Tags
  ├── Security
  │     ├── Users / Roles / API Keys
  │     └── Role Mappings
  └── Stack
        ├── License Management
        └── Upgrade Assistant
```

# 2. Index Management

```
Stack Management → Index Management

Indices:
  - 查看所有索引列表、大小、文档数、状态
  - 打开/关闭/删除/强制合并索引
  - 查看 Index Settings 和 Mappings

Manage Index:
  - Edit settings: 动态修改 refresh_interval、replicas
  - Force merge: 合并段以减少存储
```

# 3. Index Templates

```
Stack Management → Index Templates

创建 Template:
  Name: order-template
  Index pattern: orders-*
  Priority: 100
  Settings: number_of_shards=3, number_of_replicas=1
  Mappings: { "order_no": {"type": "keyword"}, "amount": {"type": "double"} }
  Aliases: orders-alias
```

# 4. ILM 策略

```
Stack Management → Index Lifecycle Policies

创建 ILM Policy: "logs-policy"
  Hot phase:
    Rollover: max_size=50GB OR max_age=30d
  Warm phase (min_age=3d):
    Force merge: max_num_segments=1
    Shrink: number_of_shards=1
  Cold phase (min_age=30d):
    Searchable snapshot
  Delete phase (min_age=90d):
    Delete
```

# 5. Saved Objects

```
Stack Management → Saved Objects

按类型查看/管理:
  - Dashboards / Visualizations / Maps
  - Index Patterns / Searches / Tags

操作:
  - Export: 导出为 NDJSON（含所有依赖）
  - Import: 从 NDJSON 导入
  - Copy to Space: 跨 Space 复制
  - Delete: 删除（含依赖检查）
```

# 6. Advanced Settings

```
Stack Management → Advanced Settings

常用设置:
  dateFormat: "YYYY-MM-DD HH:mm:ss"
  defaultIndex: "logs-*"
  csv:maxSizeBytes: 10485760 (10MB 导出上限)
  discover:sampleSize: 500
  doc_table:hideTimeColumn: false
  histogram:maxBars: 100
  state:storeInSessionStorage: false
  theme:darkMode (Dark Theme)
```

# 7. Tags

```
Tags 给 Saved Objects 分类:

Dashboard "Order Monitor" → Tags: "orders", "production"
Dashboard "User Monitor"   → Tags: "users", "production"
Lens "Revenue Chart"      → Tags: "orders", "revenue"

搜索: Tags:"orders" → 所有标记为 orders 的对象
```

# 8. Watcher (X-Pack)

Watcher 提供**脚本化告警**，比 Alerting 更灵活但更复杂：

```json
PUT _watcher/watch/order_alert
{
  "trigger": { "schedule": { "interval": "5m" } },
  "input": {
    "search": {
      "request": {
        "indices": ["orders-*"],
        "body": {
          "query": { "range": { "amount": { "lt": 100 } } }
        }
      }
    }
  },
  "condition": { "compare": { "ctx.payload.hits.total.value": { "gt": 10 } } },
  "actions": {
    "email_admin": {
      "email": {
        "to": "admin@example.com",
        "subject": "Low amount orders detected"
      }
    }
  }
}
```
