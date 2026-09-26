
# 1. 概述

Data View（8.x 前称为 Index Pattern）定义 Kibana 如何识别 ES 索引中的字段，是所有搜索和可视化的基础。

```
ES Index: orders-2024-01, orders-2024-02, ...
    │
    ▼
Data View: orders*
  ├── Time field: created_at
  ├── Fields: order_no(keyword), amount(double), status(keyword), ...
  └── Runtime Fields: day_of_week(script)
```

# 2. 创建 Data View

```
Stack Management → Data Views → Create data view
  Name: orders
  Index pattern: orders-*
  Time field: @timestamp

# API 创建
POST /api/data_views/data_view
{
  "data_view": {
    "title": "orders-*",
    "name": "Orders",
    "timeFieldName": "created_at"
  }
}
```

# 3. 通配符模式

| Pattern | 匹配 | 说明 |
|---------|------|------|
| `orders-*` | orders-2024-01, orders-2024-02 | 所有 orders 前缀 |
| `logs-2024.01.*` | logs-2024.01.15, logs-2024.01.16 | 指定月份 |
| `*-logs-*` | app-logs-prod, nginx-logs-dev | 包含 logs 的索引 |
| `orders` | orders（精确） | 单个索引 |

# 4. 字段管理

## 4.1 字段格式化

```
Data View → 字段列表 → 编辑字段 → Format

常见格式化:
  - Bytes: 数字 → "1.5 MB"
  - Duration: 毫秒 → "1h 30m 25s"
  - Number: 数字 → "1,000,000.00"
  - URL: 字段值 → 可点击链接
  - String: 字段值 → 大写/小写
  - Color: 数值 → 颜色条
```

## 4.2 Runtime Field

```
在 Kibana 中动态定义计算字段（无需重建索引）:

Data View → Add field → 选择 Runtime

# 示例: 计算利润
Name: profit
Type: Double
Script: emit(doc['price'].value - doc['cost'].value)

# 示例: 提取星期几
Name: day_of_week
Type: Keyword
Script: emit(doc['created_at'].value.dayOfWeekEnum().toString())
```

**Runtime Field vs Scripted Field**：
- Runtime Field：8.x 推荐，性能更好，在 ES 端执行
- Scripted Field：旧版，在 Kibana 端执行，性能差

# 5. 字段过滤器

```
Data View → Source filters

排除不需要的字段（减少带宽和内存）:
  _source
  @version
  beat.*
  host.*
  input.*
```

# 6. 多 Data View 策略

```
按团队/应用拆分：
  - order-service-*   → Order Team
  - user-service-*    → User Team
  - nginx-*           → Ops Team

统一 Data View（跨应用搜索）：
  - service-logs-*    → 所有服务日志
  - metricbeat-*      → 所有主机指标
```

# 7. API 管理

```bash
# 列表
GET /api/data_views

# 获取
GET /api/data_views/data_view/<id>

# 更新
POST /api/data_views/data_view/<id>
{ "data_view": { "title": "orders-new-*" } }

# 删除
DELETE /api/data_views/data_view/<id>

# 字段列表缓存刷新
POST /api/data_views/data_view/<id>/refresh_fields
```
