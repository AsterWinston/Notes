
# 1. 概述

Discover 是 Kibana 的**交互式数据探索工具**，提供对 ES 数据的实时搜索、筛选和预览能力。

```
┌─────────────────────────────────────────────────────┐
│                    Discover                           │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  KQL Search Bar: status:active AND amount > 100│  │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  ┌─────────────┐  ┌──────────────────────────┐     │
│  │ Time Filter  │  │ Fields Panel (左侧)      │     │
│  │ (右上)       │  │  - Selected Fields        │     │
│  │              │  │  - Available Fields       │     │
│  └─────────────┘  │  - Popular Fields          │     │
│                   └──────────────────────────┘     │
│  ┌──────────────────────────────────────────────┐   │
│  │         Document Table (中央)                  │   │
│  │  ┌──────┬─────────┬────────┬────────────────┐ │   │
│  │  │ Time │ order_no│ amount │ status         │ │   │
│  │  ├──────┼─────────┼────────┼────────────────┤ │   │
│  │  │ 10:00│ ORD001  │ 100.0  │ active [>]     │ │   │
│  │  │ 10:01│ ORD002  │ 200.0  │ pending [>]    │ │   │
│  │  └──────┴─────────┴────────┴────────────────┘ │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

# 2. 搜索与筛选

```kql
# KQL 搜索
status:active                          # 精确匹配
amount > 100                           # 范围
customer_full_name: "John Doe"         # 带空格的值
user*                                   # 通配符
status:(active OR pending)              # 布尔
NOT status:cancelled                    # 排除
```

## 2.1 添加过滤器

```
方式 1: 点击文档表格中的字段值 → "+" 添加
方式 2: Search Bar 左侧 → Add filter → 选择字段和操作符
方式 3: Fields Panel → 字段名 → 选择值

过滤器操作符:
  is / is not / is one of / is between
  exists / does not exist
```

# 3. 字段操作

```
Fields Panel:
  - 点击 ★ 添加到 Selected Fields
  - 点击字段名 → 查看 Top 5 值分布
  - 拖拽字段到表格排序

文档展开:
  点击文档左侧 ">" → 查看完整 JSON
  - Table 视图（表格）
  - JSON 视图（原始 JSON）
```

# 4. 时间过滤器

```
时间选择器（右上）：
  - Quick Select: Last 15 minutes / 1 hour / 7 days
  - Absolute: 2024-01-15 10:00 → 2024-01-15 18:00
  - Relative: 从现在往前 24 小时

时间过滤器的使用:
  - 影响所有面板（Dashboard 全局）
  - 点击时间线柱状图 → 快速放大某时间段
  - 拖拽时间线 → 选择范围
```

# 5. 保存搜索

```
Discover 右上角 → Save
  Title: Active Orders over $100
  Tags: orders, important

加载已保存的搜索:
  Discover → Open → 选择已保存的搜索
```

# 6. 搜索会话

```
8.x 新增：Search Sessions

Discover → 搜索结果 → Save session
  - 暂停搜索 → 恢复后继续
  - 后台运行长时间搜索 → 稍后查看结果
  - 适合大数据量的聚合查询

Management → Search Sessions → 查看/管理会话
```

# 7. 数据导出

```
Discover → Share → CSV Reports
  生成 CSV 文件下载

注意:
  - 最多导出 10,000 条（默认配置）
  - 可通过 Advanced Settings 调整 csv:maxSizeBytes
```

# 8. Discover 最佳实践

1. **限制字段数**：Selected Fields 仅添加需要的列（避免全量 _source 渲染）
2. **使用 saved search**：常用查询保存为 saved search，Dashboard 可直接引用
3. **时间范围先行**：先缩小时间范围再搜索（减少 ES 负载）
4. **字段别名**：Data View 中给长字段名设置别名
5. **避免 `*` 全模糊**：KQL 中 `field*` 使用前缀而非全模糊
