
# 1. 概述

KQL（Kibana Query Language）是 Kibana 自研的查询语法，简化 ES Query DSL 的学习成本。

```
KQL 查询 → Kibana 自动转换为 ES Query DSL → 发送到 ES
```

# 2. 基础语法

```kql
# 字段查询
status:active                           # 精确匹配
response:200                            # 数字精确匹配
message:"Error occurred"                # 带空格的短语（引号）
user.name:john                          # 嵌套字段（点号）

# 通配符
user*                                   # 匹配 user, users, username
response:40*                            # 匹配 400, 401, 403, 404
message:err*                            # 匹配 error, err, errors

# 逻辑运算符
status:active AND amount > 100          # 且
status:cancelled OR status:pending      # 或
NOT status:deleted                      # 非
(status:active OR status:pending) AND amount > 100  # 括号分组
```

# 3. 范围查询

```kql
# 数值范围
amount > 100
amount >= 100
amount < 500
amount <= 500
amount >= 100 AND amount <= 500        # 区间

# 日期范围
@timestamp > "2024-01-15"
@timestamp < "now-7d"
created_at >= "2024-01-01" AND created_at <= "2024-01-31"
```

# 4. 存在性查询

```kql
# 字段存在
description:*                           # description 字段不为空
_exists_:phone                          # phone 字段存在

# 字段不存在
NOT description:*                       # description 为空
NOT _exists_:phone                      # phone 不存在
```

# 5. 嵌套字段查询

```kql
# object 嵌套
address.city:Beijing                   # { "address": { "city": "Beijing" } }

# nested 查询（items 是 nested 类型）
items:{ name:iPhone AND quantity > 1 }
```

# 6. 特殊字符转义

```kql
# 需要转义: \ ( ) : < > " * ?
message:"C:\\Program Files"             # 反斜杠
message:"price:\$100"                   # 冒号
url:"https\\://example.com"             # URL 中的冒号和斜杠
```

# 7. KQL vs Lucene

| 维度 | KQL | Lucene |
|------|-----|--------|
| 语法 | 现代、简洁 | 传统、强大 |
| 比较操作 | `>` `<` `>=` `<=` | `[a TO b]` 范围 |
| 逻辑 | `AND` `OR` `NOT` | `AND` `OR` `NOT` / `&&` `\|\|` |
| 模糊匹配 | 前缀 `field*` | `field~` (编辑距离) |
| 正则 | 不支持 | `/regex/` |
| 推荐 | **默认，新用户** | 高级用户 |

```kql
# KQL
status:active AND amount > 100

# Lucene
status:active AND amount:{100 TO *}
```

# 8. 自动补全

```
Kibana 的 KQL 自动补全功能：
  1. 输入 "sta" → 补全字段名: status, status_code
  2. 输入 "status:" → 补全字段值: active, pending, cancelled
  3. 输入 "status:ac" → 补全: active

依赖：
  - ES 的 field caps API（字段信息）
  - ES 的 terms aggregation（Top 值）
```

# 9. 查询性能建议

```kql
# ✓ 好：字段前缀查询
user*                                   # 利用倒排索引

# ✗ 差：全模糊查询
*user*                                  # 只能全扫描，极慢

# ✓ 好：精确匹配
status:active                           # 走 keyword 索引

# ✗ 差：text 字段短词（分词开销）
description:a                           # 匹配所有含 "a" 的文档

# ✓ 好：范围限制
amount > 100 AND created_at > now-7d    # 缩小数据量
```
