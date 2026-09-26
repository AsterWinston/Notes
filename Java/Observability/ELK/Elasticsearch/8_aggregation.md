
# 1. 聚合三大类

```
Aggregations
  ├── Bucket (桶聚合)     → 分组
  ├── Metric (指标聚合)    → 统计
  ├── Pipeline (管道聚合)  → 对聚合结果再聚合
  └── Matrix (矩阵聚合)    → 多字段协方差
```

# 2. Bucket 聚合

## 2.1 Terms — 按值分组

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": {
        "field": "status",
        "size": 10,               // 返回 top 10
        "order": { "_count": "desc" }
      }
    }
  }
}
```

## 2.2 Date Histogram — 时间直方图

```json
{
  "aggs": {
    "orders_over_time": {
      "date_histogram": {
        "field": "created_at",
        "fixed_interval": "1d",        // 固定间隔
        "calendar_interval": "month",  // 日历间隔
        "min_doc_count": 1,
        "extended_bounds": {
          "min": "2024-01-01",
          "max": "2024-01-31"
        }
      }
    }
  }
}
```

## 2.3 Range — 范围分桶

```json
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "amount",
        "ranges": [
          { "to": 100 },
          { "from": 100, "to": 500 },
          { "from": 500, "to": 1000 },
          { "from": 1000 }
        ]
      }
    }
  }
}
```

## 2.4 Histogram — 数值直方图

```json
{
  "aggs": {
    "price_histogram": {
      "histogram": {
        "field": "amount",
        "interval": 100
      }
    }
  }
}
```

# 3. Metric 聚合

```json
{
  "aggs": {
    "avg_price": { "avg": { "field": "amount" } },
    "max_price": { "max": { "field": "amount" } },
    "min_price": { "min": { "field": "amount" } },
    "sum_amount": { "sum": { "field": "amount" } },
    "stats": { "stats": { "field": "amount" } },
    "extended_stats": { "extended_stats": { "field": "amount" } },
    "cardinality": {                // 近似去重计数
      "cardinality": {
        "field": "user_id",
        "precision_threshold": 40000
      }
    },
    "percentiles": {                // 百分位
      "percentiles": {
        "field": "amount",
        "percents": [50, 95, 99]
      }
    }
  }
}
```

# 4. 嵌套聚合

```json
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {                                     // 子聚合
        "avg_price": { "avg": { "field": "price" } },
        "top_products": {                            // 顶级命中
          "top_hits": {
            "size": 3,
            "_source": ["title", "price"],
            "sort": [{ "price": "desc" }]
          }
        }
      }
    }
  }
}
```

# 5. Pipeline 聚合

```json
{
  "aggs": {
    "orders_per_month": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month"
      },
      "aggs": {
        "total": { "sum": { "field": "amount" } }
      }
    },
    "avg_monthly_revenue": {          // Pipeline: 基于父聚合
      "avg_bucket": {
        "buckets_path": "orders_per_month>total"
      }
    },
    "revenue_derivative": {           // Pipeline: 环比增长
      "derivative": {
        "buckets_path": "orders_per_month>total"
      }
    }
  }
}
```

# 6. 聚合执行原理

```
聚合执行流程：
  1. 协调节点解析聚合请求
  2. 分发到各分片并行计算
  3. 各分片返回局部聚合结果
  4. 协调节点合并返回最终结果

注意：
  - terms 聚合可能有误差（每个分片返回 top N）
  - cardinality 是近似值（HyperLogLog++ 算法）
  - 聚合在 doc_values 上执行（列式存储）
```

# 7. 聚合优化

```json
# 1. 限制桶数量
"terms": { "field": "status", "size": 10, "shard_size": 100 }

# 2. 使用 filter 缩小范围
{
  "query": { "range": { "created_at": { "gte": "2024-01-01" } } },
  "aggs": { ... }
}

# 3. execution_hint
"terms": { "field": "status", "execution_hint": "map" }
# map: 小 cardinality；global_ordinals: 大 cardinality（默认）
```
