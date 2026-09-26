
# 1. 概述

Ingest Pipeline 允许在索引前对文档进行**预处理**（字段转换、丰富、过滤），无需依赖 Logstash。

```
Document → Ingest Pipeline → Index
              │
              ├── Set Processor（设置字段）
              ├── Grok Processor（正则解析）
              ├── Date Processor（日期转换）
              ├── Script Processor（Painless 脚本）
              └── Remove Processor（删除字段）
```

# 2. 创建 Pipeline

```json
PUT /_ingest/pipeline/order_pipeline
{
  "description": "订单数据预处理",
  "processors": [
    {
      "set": {
        "field": "processed_at",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "convert": {
        "field": "amount",
        "type": "double"
      }
    },
    {
      "script": {
        "source": """
          ctx.total = ctx.amount * ctx.quantity;
          ctx.year = ctx.created_at.substring(0, 4);
        """
      }
    },
    {
      "remove": {
        "field": ["temp_field", "internal_notes"],
        "ignore_missing": true
      }
    }
  ]
}

# 使用 Pipeline
PUT /orders/_doc/1?pipeline=order_pipeline
{ "order_no": "ORD001", "amount": "100.0", "quantity": 3, "created_at": "2024-01-15" }
```

# 3. Processors

## 3.1 Set — 设置字段

```json
{
  "set": {
    "field": "env",
    "value": "production"
  }
}
```

## 3.2 Grok — 正则解析

```json
{
  "grok": {
    "field": "message",
    "patterns": ["%{IP:client_ip} %{WORD:method} %{URIPATHPARAM:url} %{NUMBER:status:int} %{NUMBER:response_time:float}"]
  }
}

# 输入: "192.168.1.1 GET /api/user 200 25.5"
# 输出: { client_ip: "192.168.1.1", method: "GET", url: "/api/user", status: 200, response_time: 25.5 }
```

## 3.3 Date — 日期转换

```json
{
  "date": {
    "field": "created_at",
    "formats": ["yyyy-MM-dd HH:mm:ss", "ISO8601"],
    "target_field": "@timestamp"
  }
}
```

## 3.4 Script — Painless 脚本

```json
{
  "script": {
    "source": """
      if (ctx.amount >= 10000) {
        ctx.risk = "high";
      } else {
        ctx.risk = "normal";
      }
    """
  }
}
```

## 3.5 Pipeline — 嵌套管道

```json
{
  "pipeline": {
    "name": "another_pipeline"
  }
}
```

# 4. 条件执行

```json
{
  "processors": [
    {
      "set": {
        "if": "ctx.amount >= 10000",
        "field": "review_required",
        "value": true
      }
    },
    {
      "set": {
        "if": "ctx.status == 'rejected'",
        "field": "reject_reason",
        "value": "manual",
        "on_failure": [
          {
            "set": {
              "field": "error",
              "value": "failed to process rejected order"
            }
          }
        ]
      }
    }
  ]
}
```

# 5. Pipeline 调试

```json
# 模拟处理（不实际索引）
POST /_ingest/pipeline/order_pipeline/_simulate
{
  "docs": [
    {
      "_source": { "order_no": "ORD001", "amount": "100.0", "quantity": 3 }
    }
  ]
}
```

# 6. vs Logstash

| 维度 | Ingest Pipeline | Logstash |
|------|----------------|----------|
| 部署 | ES 内置 | 独立部署 |
| 性能 | 高（ES 节点执行） | 中（独立进程） |
| 功能 | 基础处理 | 丰富插件生态 |
| 外部依赖 | ✗ | JDBC/Redis/Kafka 输入 |
| 适用场景 | 简单预处理 | 复杂 ETL |
