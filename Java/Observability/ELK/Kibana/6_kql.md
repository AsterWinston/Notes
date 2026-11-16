# 1. 概述

KQL（Kibana Query Language）是 Kibana 自研的查询语法，目标是**用最小语法覆盖日志检索的绝大多数场景**，把用户从 ES Query DSL（以及 Lucene 语法）的学习成本里解放出来。它最初是为 Beats 日志查询设计（2016 年），如今是 Discover、Dashboard 过滤、告警 Rule、Lens 数据视图的默认输入语言。

KQL 不是 ES DSL 的"别名"，而是运行在 Kibana 前端/服务端的**独立解析器**产出的中间表达式，再编译为 Query DSL：

```text
KQL 查询 → Kibana 自动转换为 ES Query DSL → 发送到 ES
```

以 `status:active AND amount > 100` 为例，编译产物大致是：

```json
{
  "bool": {
    "must": [
      { "term": { "status": "active" } },
      { "range": { "amount": { "gt": 100 } } }
    ]
  }
}
```

在 Discover 的 Inspect → Request 里随时能看到这个翻译结果——这也是理解 KQL"到底查了什么"的唯一权威途径（KQL 对 text 字段自动走 match、对 keyword 走 term 等隐式规则，只读语法本身猜不到）。

三条语法设计原则：**字段名与值之间用冒号不用等号**；**布尔运算符大写 AND/OR/NOT**（小写会被当普通词）；**没有正则、没有裸通配开头、没有字段权重**——这些"没有"是刻意的，复杂需求留给 Lucene 语法兜底。

# 2. 基础语法

## 2.1 字段查询与值匹配

```kql
# 字段查询
status:active                           # 精确匹配
response:200                            # 数字精确匹配
message:"Error occurred"                # 带空格的短语（引号）
user.name:john                          # 嵌套字段（点号）
```

字段匹配的实际语义比字面复杂一层，值得单独记住：`status:active` 对 **keyword 字段编译为 term**（整词精确匹配，大小写敏感）；对 **text 字段编译为 match**（按分词器切词后匹配，`message:"error timeout"` 会同时要求两个词相邻短语）。因此"为什么我查 `status:Active` 没结果"大概率是 keyword 大小写问题，而"查 `message:error` 命中一堆不相关的行"则是 text 分词 + match 语义的正常表现。

值包含空格、冒号或特殊字符时必须加双引号（`"..."`），引号内空格不再分词。布尔值字段直接写 `true`/`false`（`is_admin:true`），日期字段可用日期字符串或 `now-` 表达式（见 3 节）。

## 2.2 通配符

```kql
# 通配符
user*                                  # 匹配 user, users, username
response:40*                           # 匹配 400, 401, 403, 404
message:err*                           # 匹配 error, err, errors
```

KQL 通配符只有 `*`（匹配零个或多个字符），且**只允许出现在值的尾部（前缀匹配）**或整体作为值（`message:err*`）；`*user*`、`*user` 这类中缀/后缀通配在语法上合法但性能是灾难级的（见 8 节）。注意 `*` 只对 keyword 类字段有意义——text 字段本身已分词，对它做通配既慢又违背直觉。

## 2.3 逻辑运算与分组

```kql
status:active AND amount > 100                 # 且
status:cancelled OR status:pending             # 或
NOT status:deleted                             # 非
(status:active OR status:pending) AND amount > 100  # 括号分组
```

优先级 `NOT > AND > OR`，但**永远显式加括号**——复杂条件不写括号不仅易错，Kibana 的 UI 解析与高亮也可能误导肉眼。多个值属于同一字段时，用 `in` 风格的简写更清晰且避免括号地狱：

```kql
status:("active" OR "pending" OR "shipped")    # 多值枚举
response:200 OR response:201 OR response:204   # 冗长写法（不推荐）
```

# 3. 范围查询

数值与日期都支持比较运算符 `>` `>=` `<` `<=`：

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

日期表达式支持 ES 的数学语法：`now`、`now-7d`、`now-1h/M`（取整到月）、绝对值配合时区偏移（`"2024-01-15T00:00:00+08:00"`）。实际排障中"按 KQL 查时间"用得远少于**顶部时间选择器**——时间过滤器永远优先，因为它能利用索引层按时间分文件/分 shard 的物理裁剪，KQL 里的日期范围只是普通 range 查询。两者同时存在时是 AND 关系。

> **注意**：对 date 字段做 `>` 比较时值必须是合法日期表达式，写成 `@timestamp > 2024-01-15`（无引号）会被解析器吞掉或报错——日期字符串要加引号，这是 KQL 与 Lucene 行为不同的易错点之一。

# 4. 存在性查询

判断字段"有值/无值"有两种写法，语义并不相同，容易用错：

```kql
# 字段存在
description:*                           # 字段存在且值匹配任意（含空串场景看 ES 版本）
_exists_:phone                          # 字段存在（旧式/兼容写法）

# 字段不存在
NOT description:*                       # 字段无值
NOT _exists_:phone                      # 字段不存在
```

`field:*` 在 KQL 里编译为 `exists` 查询（ES 的 exists 对 text 字段会借助 `_field_` 存在性判断），而 `_exists_:` 前缀是历史兼容写法，官方文档现在更推荐 `field:*` 表达"存在"。要过滤**空串**（字段存在但值为 `""`），exists 类写法都不管用——keyword 字段用 `field:""`，text 字段则因分词后空串无 term，需要回 ES 层用脚本或 wildcard 兜底，KQL 表达不了这种细节。

# 5. 嵌套字段查询

ES 的 object 与 nested 是两种完全不同的存储模型，KQL 对二者的写法也不同，这是 KQL 语法里最容易踩出**语义错误**的一节：

```kql
# object 嵌套：点号路径直达
address.city:Beijing                   # { "address": { "city": "Beijing" } }

# nested 类型：用 { } 包裹子条件，组内字段必须同属一个子文档
items:{ name:iPhone AND quantity > 1 }
```

**object（普通 JSON 对象）**在 ES 里被扁平化为 `address.city` 这样的叶子字段存储，`items.name:iPhone AND items.quantity:2` 并不保证 name 和 quantity 来自同一个数组元素——数组跨元素"串味"是经典数据质量陷阱。**nested 类型**才保证子文档独立索引，KQL 用花括号语法把条件约束进同一子文档。如果你的 mapping 里数组是 object（写入端默认动态推断几乎都是 object），`{ }` 语法一样能写，但 ES 端只能按扁平逻辑处理，起不到 nested 的隔离作用——语法救不了 mapping。

# 6. 特殊字符转义

KQL 的双引号字符串内，特殊字符需用反斜杠转义（保留字符：`\` `(` `)` `:` `<` `>` `"` `*` `?` 等按上下文生效）：

```kql
message:"C:\\Program Files"             # 反斜杠
message:"price: $100"                   # 引号内冒号无需转义
message:"50% off"                       # 普通字符直接写
```

比转义更实用的建议：**值里有歧义就加引号**。`message:"C:\Program Files"`（quoted 内反斜杠在部分小版本仍需 `\\`，以 Inspect 编译结果为准）这类 Windows 路径、含冒号的日志片段、含空格的枚举值，一律整体加引号。URL 值同理：`url:"https://example.com/a?b=1&c=2"` 用引号包住即可，不要手动逐字符转义——Inspect 看 DSL 是检验转义是否正确的最终手段。

# 7. KQL vs Lucene

Discover 搜索框左下角可切换语法模式（默认 KQL，Settings 可改全局默认），两种语言的对照：

| 维度 | KQL | Lucene |
|------|-----|--------|
| 语法 | 现代、简洁 | 传统、强大 |
| 比较操作 | `>` `<` `>=` `<=` | `[a TO b]` 范围 |
| 逻辑 | `AND` `OR` `NOT` | `AND` `OR` `NOT` / `&&` `\|\|` |
| 模糊匹配 | 前缀 `field*` | `field~`（编辑距离） |
| 正则 | 不支持 | `/regex/` |
| 裸词查询 | 不支持（必须 `field:value`） | 支持（默认字段/全字段） |
| 字段权重 | 不支持 | `field^2` |
| 推荐 | **默认，新用户** | 高级用户兜底 |

```kql
# KQL
status:active AND amount > 100

# Lucene 等价写法
status:active AND amount:{100 TO *}
```

选择判据：**能用 KQL 就用 KQL**——Lucene 语法对 text 字段的默认分词行为与 KQL 不完全一致、裸词易写出隐式全字段匹配（性能不可控），且 Lucene 模式缺少字段名/值的深度自动补全。KQL 表达不了的典型需求：模糊纠错（`name~2`）、正则（`message:/timeout-\d+/`）、单字符通配 `?`、对多字段做 boost 排序时，才切 Lucene。

> **设计考量**：Kibana 为什么自研语言而不是直接暴露 Lucene/DSL？KQL 是"窄接口"哲学：语法可解析性优先于表达力，保证补全、高亮、跨版本翻译行为可控；Lucene 表达力更强但用户行为不可预测（一个裸 `*` 就能打挂集群），所以保留为显式切换的高级选项而非默认。

# 8. 查询性能建议

KQL 写法的性能差异本质是**它编译成什么 DSL** 的差异，对照如下：

```kql
# ✓ 好：字段前缀查询
user*                                   # 走倒排索引 term 前缀扩展（有限代价）

# ✗ 差：全模糊查询
message:*user*                          # 词典全扫描，极慢

# ✓ 好：精确匹配
status:active                           # keyword term，O(1) 级别定位

# ✗ 差：text 字段短词
description:a                           # 高频词，命中集爆炸 + 相关性排序开销

# ✓ 好：范围 + 时间限定缩小数据量
amount > 100 AND created_at > now-7d    # 先用时间过滤裁剪分片
```

性能红线与经验：

- **通配符永远只往右放**：`err*` 可接受，`*rror`、`*err*` 会让 ES 遍历整个 term 词典；万不得已要"包含"语义时，优先考虑写入端把该字段建成 `ngram`/`wildcard` 类型索引（8.x 的 `wildcard` 字段类型就是为这种需求设计的），而不是在查询端硬扛。
- **term 精确优于 match 分词**：对 text 字段查"完整短语"务必加引号（编译为 `match_phrase`），不加引号会退化为多 term 的 OR 语义，命中面大且排序开销高。
- **NOT 不省资源**：`NOT status:200` 在 ES 层是"全量减 200"，扫描面比正向查询更大；高基数字段上用 `field:(A OR B)` 枚举正向值永远优于排除法。
- **时间窗口是最强的性能开关**：range 与 exists 类查询的代价和扫描文档数成正比，先把时间收窄再谈语法优化。
- 复杂过滤写不出性能时，用 Inspect → Request 看 DSL 的 `took` 与命中的 indices，比凭感觉改语法可靠得多。

# 9. 自动补全

Kibana 的 KQL 自动补全功能：

```text
1. 输入 "sta"            → 补全字段名: status, status_code
2. 输入 "status:"        → 补全字段值: active, pending, cancelled
3. 输入 "status:ac"      → 补全: active
```

依赖：

- ES 的 **field caps API**：聚合多索引的字段名与类型（Data View 字段列表同源）
- ES 的 **terms aggregation**：拉取字段 Top 值作为候选（`history` 设置缓存最近使用）

补全列表"转圈很久才出来"或"少了预期值"不是 KQL 问题：前者是 top values 聚合慢（大索引/高基数），后者可能是该值不在统计窗口内、或字段是 text 分词后无法枚举原值。理解补全机制后，知道"值补全只覆盖 Top N 而非字典全量"，就不会误以为补全没有 = 数据没有。

# 10. 小结

KQL 以最小语法覆盖日志检索主流场景：`field:value`、`>`/`>=` 比较、`AND/OR/NOT` + 括号、尾部通配 `*`、`field:*` 存在性，编译结果用 Inspect 随时可查。语义层面记住三个隐式规则——keyword 走 term、text 走 match、日期字符串要加引号；nested 用 `{ }` 语法但隔离语义取决于 mapping。与 Lucene 的分工：KQL 默认，正则/模糊/裸词/boost 才切 Lucene。性能心法四句：时间窗口先行、通配只往右、短语加引号、NOT 不如正向枚举。
