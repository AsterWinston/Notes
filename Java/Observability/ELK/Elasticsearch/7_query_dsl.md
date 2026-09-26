
# 1. 查询上下文 vs 过滤上下文

| 上下文 | 用途 | 评分 | 缓存 |
|--------|------|------|------|
| **Query** | 相关性搜索 | ✓ 计算 _score | ✗ |
| **Filter** | 精确筛选 | ✗ 无评分 | ✓ 自动缓存 |

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "iPhone" } }        // 查询上下文，评分
      ],
      "filter": [
        { "term": { "status": "active" } },        // 过滤上下文，无评分，缓存
        { "range": { "price": { "gte": 1000 } } }
      ]
    }
  }
}
```

# 2. 全文查询

```json
# match — 分词后匹配
{ "match": { "title": "iPhone 15 Pro" } }

# match_phrase — 短语匹配（保持顺序）
{ "match_phrase": { "title": "iPhone 15" } }

# match_phrase_prefix — 前缀短语（搜索建议）
{ "match_phrase_prefix": { "title": "iPh" } }

# multi_match — 多字段查询
{
  "multi_match": {
    "query": "iPhone",
    "fields": ["title^3", "description"]  // title 权重 3 倍
  }
}

# query_string — 类 Lucene 语法
{ "query_string": { "query": "title:iPhone AND price:[1000 TO 2000]" } }
```

# 3. 词项查询

```json
# term — 精确匹配（keyword 类型或 text.keyword 子字段）
{ "term": { "status": "active" } }

# terms — 多值匹配
{ "terms": { "category": ["电子产品", "图书"] } }

# range — 范围查询
{
  "range": {
    "price": {
      "gte": 1000,
      "lte": 5000,
      "boost": 2.0
    }
  }
}

# exists — 字段存在
{ "exists": { "field": "description" } }

# prefix — 前缀查询
{ "prefix": { "order_no": "ORD" } }
```

# 4. Bool 复合查询

```json
{
  "query": {
    "bool": {
      "must": [ ... ],           // AND, 参与评分
      "should": [                // OR, 参与评分
        { "match": { "title": "iPhone" } },
        { "match": { "title": "MacBook" } }
      ],
      "must_not": [ ... ],       // NOT, filter 上下文
      "filter": [ ... ],         // filter 上下文（不评分，缓存）
      "minimum_should_match": 1  // should 至少匹配 N 个
    }
  }
}
```

# 5. 相关性评分

## 5.1 BM25（默认）

```
score(q, d) = Σ IDF(qi) × TF(qi, d) / (TF(qi, d) + k1 × (1 - b + b × |d| / avgDL))

IDF: 逆文档频率（词越稀有权重越高）
TF: 词频（饱和控制，k1=1.2）
|d|: 文档长度（长度归一化，b=0.75）
```

## 5.2 explain API

```json
GET /products/_search
{
  "query": { "match": { "title": "iPhone" } },
  "explain": true
}
```

# 6. Search Template

```json
# 保存模板
POST /_scripts/product_search
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": { "{{field}}": "{{value}}" }
      }
    }
  }
}

# 使用模板
GET /products/_search/template
{
  "id": "product_search",
  "params": { "field": "title", "value": "iPhone" }
}
```

# 7. 搜索参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `size` | 每页数量 | 10 |
| `from` | 偏移量 | 0 |
| `_source` | 返回字段 | 全部 |
| `sort` | 排序 | `_score` |
| `timeout` | 超时 | 无 |
| `track_total_hits` | 精确总数 | true |

```json
GET /products/_search
{
  "size": 20,
  "from": 0,
  "_source": ["title", "price"],
  "sort": [
    { "price": "asc" },
    { "_score": "desc" }
  ],
  "timeout": "2s"
}
```
