
# 1. 查询优化

## 1.1 Filter 物化

```json
# Filter 上下文自动缓存
GET /orders/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "active" } }   // filter 自动缓存
      ]
    }
  }
}
```

## 1.2 Profile API

```json
GET /orders/_search
{
  "profile": true,
  "query": { "match": { "title": "iPhone" } }
}

# 返回各阶段耗时: query / rewrite / create_weight / build_scorer
```

# 2. 缓存体系

| 缓存类型 | 缓存内容 | 失效条件 |
|---------|---------|---------|
| **Node Query Cache** | Filter 查询结果 | Segment 变更 |
| **Shard Request Cache** | 整个搜索请求结果 | Refresh |
| **Field Data Cache** | 排序/聚合字段值 | Segment 变更 |

```json
# 查看缓存统计
GET /_nodes/stats/indices/query_cache
GET /_nodes/stats/indices/fielddata
GET /_nodes/stats/indices/request_cache
```

# 3. 排序优化

```json
# 关闭评分（不需要相关性时）
GET /orders/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "active" } }
      ]
    }
  },
  "sort": [{ "created_at": "desc" }]
}
```

**效果**：filter 上下文不计算 `_score`，性能大幅提升。

# 4. 深分页解决方案

## 4.1 from+size（默认，有限制）

```json
GET /orders/_search
{
  "from": 10000,
  "size": 20
}
# 限制: max_result_window = 10000
```

## 4.2 Scroll（遍历全量）

```json
# 创建 scroll
GET /orders/_search?scroll=5m
{ "size": 1000 }

# 后续翻页
GET /_search/scroll
{ "scroll": "5m", "scroll_id": "xxx" }

# 清理
DELETE /_search/scroll
{ "scroll_id": "xxx" }
```

## 4.3 Search After（推荐，实时）

```json
GET /orders/_search
{
  "size": 20,
  "query": { "match": { "status": "active" } },
  "sort": [{ "created_at": "asc" }, { "_id": "asc" }],
  "search_after": [1705302000000, "order_id_xxx"]
}
```

## 4.4 PIT (Point In Time)（8.x 推荐）

```json
# 创建 PIT
POST /orders/_pit?keep_alive=5m
// { "id": "pit_xxx" }

# 使用 PIT 搜索
GET /_search
{
  "size": 20,
  "pit": { "id": "pit_xxx", "keep_alive": "5m" },
  "sort": [{ "@timestamp": "asc" }],
  "search_after": [1705302000000]
}

# 清理
DELETE /_pit
{ "id": "pit_xxx" }
```

# 5. 分页方案对比

| 方案 | 适用场景 | 实时性 | 深度分页 |
|------|---------|--------|---------|
| from+size | 浅分页 | 实时 | ✗ ≤10000 |
| scroll | 全量导出 | 快照 | ✓ |
| search_after | 实时翻页 | 实时 | ✓ |
| PIT | 8.x 实时搜索 | 实时 | ✓ |

# 6. Search-as-you-type

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": {
        "type": "search_as_you_type"
      }
    }
  }
}

# 自动生成子字段: title / title._2gram / title._3gram / title._index_prefix
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "iph",
      "type": "bool_prefix",
      "fields": ["title", "title._2gram", "title._3gram"]
    }
  }
}
```

# 7. 索引排序

```json
PUT /orders
{
  "settings": {
    "index": {
      "sort.field": "created_at",
      "sort.order": "desc"
    }
  }
}
```

**效果**：索引时已排序，搜索时无需全局排序（类似 MySQL 聚簇索引）。
