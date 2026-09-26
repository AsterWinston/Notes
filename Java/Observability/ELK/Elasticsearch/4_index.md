
# 1. 索引概念

```
Index (索引) = 一个独立的"数据库"
  ├── Settings: 分片数、副本数、分析器配置
  ├── Mappings: 字段类型定义
  └── Documents: 实际数据（存储在 Shard 中）
```

# 2. 创建索引

```json
PUT /orders
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "30s",
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "order_no": { "type": "keyword" },
      "user_name": { "type": "text", "analyzer": "my_analyzer" },
      "amount": { "type": "double" },
      "status": { "type": "keyword" },
      "created_at": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss" }
    }
  },
  "aliases": {
    "order_alias": {}
  }
}
```

# 3. 索引 Setting

## 3.1 静态 Setting（索引创建时定）

```json
{
  "number_of_shards": 3,              // 主分片数（不可改）
  "codec": "best_compression"          // 压缩算法
}
```

## 3.2 动态 Setting（可在线修改）

```json
PUT /orders/_settings
{
  "number_of_replicas": 2,             // 副本数
  "refresh_interval": "10s",           // refresh 间隔
  "max_result_window": 20000,          // from+size 最大值
  "max_terms_count": 65536             // terms 查询最大词项数
}
```

# 4. 索引别名

```json
# 创建别名
POST /_aliases
{
  "actions": [
    { "add": { "index": "orders-2024-01", "alias": "orders" } }
  ]
}

# 切换别名（原子操作）
POST /_aliases
{
  "actions": [
    { "remove": { "index": "orders-2024-01", "alias": "orders" } },
    { "add": { "index": "orders-2024-02", "alias": "orders" } }
  ]
}
```

**别名优势**：无缝切换索引、零停机。

# 5. 索引模板

```json
PUT /_index_template/order_template
{
  "index_patterns": ["orders-*"],
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "order_no": { "type": "keyword" },
        "amount": { "type": "double" },
        "created_at": {
          "type": "date",
          "format": "yyyy-MM-dd HH:mm:ss"
        }
      }
    },
    "aliases": {
      "orders-search": {}
    }
  }
}

# 创建索引时自动应用模板
PUT /orders-2024-02-01   # 匹配 orders-* 模式
```

# 6. Data Stream（8.x 推荐）

```json
# 创建 Index Template
PUT /_index_template/logs_template
{
  "index_patterns": ["my-logs*"],
  "data_stream": {},
  "template": { ... }
}

# 创建 Data Stream
PUT /_data_stream/my-logs

# 写入（自动路由到最新 backing index）
POST /my-logs/_doc
{ "message": "log entry" }
```

# 7. 索引操作

```bash
# 查看
GET /_cat/indices?v
GET /orders/_settings
GET /orders/_mapping

# 删除
DELETE /orders

# 关闭/打开（释放内存）
POST /orders/_close
POST /orders/_open

# 刷新/强制合并
POST /orders/_refresh
POST /orders/_forcemerge?max_num_segments=1

# 清空索引
POST /orders/_delete_by_query
{ "query": { "match_all": {} } }
```
