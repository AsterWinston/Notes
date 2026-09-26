
# 1. 文档模型

```
Index: orders
  └── _doc (type, 7.x 后固定)
        └── Document (JSON)
              ├── _id: 文档唯一标识
              ├── _source: 原始 JSON
              ├── _version: 乐观锁版本
              ├── _seq_no: 序列号（8.x 推荐替代 version）
              └── _primary_term: 主分片任期
```

# 2. 创建文档

```json
# 指定 ID
PUT /orders/_doc/1
{ "order_no": "ORD001", "amount": 100.0 }

# 自动生成 ID
POST /orders/_doc
{ "order_no": "ORD002", "amount": 200.0 }
```

# 3. 查询文档

```json
# 按 ID 查询
GET /orders/_doc/1

# 按 ID 批量查询
GET /orders/_mget
{ "ids": ["1", "2", "3"] }

# 仅返回部分字段
GET /orders/_doc/1?_source=order_no,amount
```

# 4. 更新文档

```json
# 全量更新
PUT /orders/_doc/1
{ "order_no": "ORD001", "amount": 150.0, "status": "paid" }

# 部分更新
POST /orders/_update/1
{
  "doc": { "status": "paid" }
}

# Script 更新
POST /orders/_update/1
{
  "script": {
    "source": "ctx._source.amount += params.increment",
    "params": { "increment": 50 }
  }
}
```

**内部原理**：ES 不支持原地更新，部分更新 = GET → 修改 → 重新索引（标记旧版本删除，段合并时清理）。

# 5. 删除文档

```json
DELETE /orders/_doc/1

# 删除后，文档被标记为 deleted，段合并时才物理删除
```

# 6. 乐观锁

```json
# 使用 _version（旧）
PUT /orders/_doc/1?version=5
{ "amount": 200.0 }
# → 409 Conflict if version mismatch

# 使用 _seq_no + _primary_term（8.x 推荐）
PUT /orders/_doc/1?if_seq_no=5&if_primary_term=1
{ "amount": 200.0 }

# 查看当前 seq_no
GET /orders/_doc/1
# { "_seq_no": 5, "_primary_term": 1 }
```

# 7. Bulk API

```json
POST /_bulk
{ "index": { "_index": "orders", "_id": "1" } }
{ "order_no": "ORD001", "amount": 100.0 }
{ "create": { "_index": "orders", "_id": "2" } }
{ "order_no": "ORD002", "amount": 200.0 }
{ "update": { "_index": "orders", "_id": "1" } }
{ "doc": { "amount": 150.0 } }
{ "delete": { "_index": "orders", "_id": "3" } }
```

## 7.1 Bulk 最佳实践

```bash
# 1. 每批 5-15MB（不是文档数）
# 2. 单批文档数 500-1000
# 3. 多线程并行提交
# 4. 检查 bulk response 中的 errors

curl -X POST "localhost:9200/orders/_bulk" \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @bulk.json
```

# 8. Refresh 控制

```json
# 插入后立即刷新（单文档）
PUT /orders/_doc/1?refresh=true

# 批量导入时关闭刷新
PUT /orders/_settings
{ "refresh_interval": -1 }
# 导入完成恢复
PUT /orders/_settings
{ "refresh_interval": "30s" }
```

# 9. Routing 手动路由

```json
# 指定 routing（相同 routing 值路由到同一分片）
PUT /orders/_doc/1?routing=user_123
{ "order_no": "ORD001" }

# 查询时也必须指定 routing
GET /orders/_search?routing=user_123
{ "query": { "match_all": {} } }
```
