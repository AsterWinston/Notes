
# 1. 字段类型总览

| 类型 | 说明 | 示例 |
|------|------|------|
| `text` | 全文检索字段（分词） | 商品描述 |
| `keyword` | 精确值字段（不分词） | 订单号、状态 |
| `long/float/double` | 数值 | 金额、数量 |
| `date` | 日期 | 创建时间 |
| `boolean` | 布尔 | 是否删除 |
| `object` | 嵌套 JSON 对象 | 地址 |
| `nested` | 独立索引的数组对象 | 订单明细 |
| `join` | 父子关联 | 订单+订单项 |
| `geo_point` | 地理坐标 | 经纬度 |
| `ip` | IP 地址 | 客户端 IP |
| `binary` | Base64 编码的二进制 | 文件内容 |
| `range` | 范围类型 | 价格区间 |

# 2. Dynamic Mapping

ES 根据 JSON 自动推断字段类型：

```json
POST /test/_doc
{
  "name": "Alice",          // → text + keyword 子字段
  "age": 25,                 // → long
  "active": true,            // → boolean
  "created": "2024-01-01",  // → date
  "amount": 99.9             // → float
}
```

## 2.1 Dynamic Template

```json
PUT /orders
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keyword": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      {
        "integers_as_integer": {
          "match_mapping_type": "long",
          "mapping": { "type": "integer" }
        }
      }
    ]
  }
}
```

# 3. 关键字段参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `index` | 是否索引（可搜索） | true |
| `doc_values` | 是否支持排序/聚合 | true |
| `norms` | 是否存储评分因子 | text: true, keyword: false |
| `store` | 是否单独存储 | false |
| `ignore_above` | keyword 超过此长度不索引 | 2147483647 |
| `null_value` | NULL 替代值 | null |
| `copy_to` | 复制到其他字段 | — |
| `coerce` | 类型强制转换 | true |

```json
{
  "email": {
    "type": "keyword",
    "ignore_above": 256     // 超过 256 字符不索引
  },
  "title": {
    "type": "text",
    "norms": false          // 不需要评分时关闭，节省空间
  },
  "all_fields": {
    "type": "text",
    "copy_to": "full_text"   // 聚合搜索
  }
}
```

# 4. Nested 类型

```json
PUT /orders
{
  "mappings": {
    "properties": {
      "items": {
        "type": "nested",    // 每个 item 独立索引
        "properties": {
          "name": { "type": "keyword" },
          "quantity": { "type": "integer" }
        }
      }
    }
  }
}

# 查询必须用 nested query
GET /orders/_search
{
  "query": {
    "nested": {
      "path": "items",
      "query": {
        "bool": {
          "must": [
            { "term": { "items.name": "iPhone" } },
            { "range": { "items.quantity": { "gt": 1 } } }
          ]
        }
      }
    }
  }
}
```

**Nested vs Object**：`object` 会把数组扁平化导致跨对象匹配；`nested` 每个元素独立文档。

# 5. Runtime Field

```json
PUT /orders/_mapping
{
  "runtime": {
    "day_of_week": {
      "type": "keyword",
      "script": {
        "source": "emit(doc['created_at'].value.dayOfWeek())"
      }
    }
  }
}
```

**优势**：无需重建索引即可添加字段，节省存储，但查询性能低于索引字段。

# 6. Mapping Explosion 防护

```yaml
# elasticsearch.yml
indices.mapping.total_fields.limit: 1000       # 索引最大字段数
indices.mapping.depth.limit: 20                # 最大嵌套深度
indices.mapping.nested_fields.limit: 50        # 最大 nested 字段数
indices.mapping.nested_objects.limit: 10000    # 单个文档最大 nested 对象数
```
