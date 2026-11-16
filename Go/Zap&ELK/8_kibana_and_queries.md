Kibana 是 ELK 的门面，也是绝大多数人真正"用"的部分。但很多人只会点开 Discover 搜关键词，遇到"字段搜不到""聚合数字不对""时间差 8 小时"就卡住。这些问题大多不在 Kibana，而在 Data View 的时间字段、KQL 语法和 ES 的 mapping。本篇把这几个高频卡点讲透，并给出可直接套用的查询写法。

# 1. Kibana 的定位

## 1.1 它做了什么

Kibana 本身**不存数据**，它是一个前端 + 后端代理：

```text
浏览器
  │  查询、可视化操作
  ▼
┌──────────────────────────────────────────┐
│ Kibana                                    │
│  ├─ 把界面操作翻译成 ES Query DSL          │
│  ├─ 调用 Elasticsearch 的 _search / _msearch│
│  ├─ 保存 Dashboard、Data View、告警规则     │
│  └─ 校验用户权限（Space / Role）           │
└──────────────────┬───────────────────────┘
                   ▼
            Elasticsearch
```

所以 Kibana 慢，通常是 ES 慢；Kibana 搜不到，通常是 mapping 或时间范围的问题。

## 1.2 8.x 的关键变化

| 变化 | 说明 |
|------|------|
| 索引模式改名 Data View | 概念相同，入口在 Stack Management |
| 默认开启安全 | 需要账号密码，`kibana_system` 服务账号 |
| Lens 成为默认可视化 | 替代旧的 Visualize，拖拽式 |
| 告警内置 | 不再需要单独的 Elasticsearch Watcher 插件 |
| 默认 ECS 字段 | 与 Filebeat/Logstash 的字段对齐 |

★ **关键认知**：**Kibana 只是查询界面，它不改变数据**。你在界面上做的所有筛选，最终都会变成 ES 的 `bool` 查询。理解这一点，遇到性能问题就知道该去看 ES 的慢查询日志，而不是反复刷新页面。

# 2. Data View（索引模式）

## 2.1 创建与时间字段

Data View 告诉 Kibana"查哪些索引、用哪个字段当时间轴"：

```text
Stack Management → Data Views → Create data view

Name:        logs-order-api
Index pattern: logs-order-api-*
Time field:  @timestamp
```

| 配置 | 作用 | 常见错误 |
|------|------|----------|
| Index pattern | 匹配一个或多个索引 | 写成具体日期，跨天查不到 |
| Time field | 时间过滤的基准 | 选错字段，时间范围失效 |
| Allow hidden | 是否包含隐藏索引 | 看不到 `.ds-*` 数据流后备索引 |

## 2.2 通配符与多索引

```text
logs-*                 所有以 logs- 开头的索引
logs-order-*           订单服务
logs-*-2026.11.*       2026 年 11 月的所有日志
logs-order-*,logs-pay-* 逗号分隔多个模式
```

⚠ **关键认知**：Data View 的 pattern 是**在查询时展开的**，不是创建时固定。所以 `logs-order-*` 会自动包含明天新建的索引，不需要每天改配置。反过来说，如果 pattern 写成了 `logs-order-2026.11.06`，第二天就查不到新数据了。

## 2.3 数据流与隐藏索引

8.x 用 data stream 后，实际索引名是 `.ds-logs-order-2026.11.06-000001`，以点开头，默认隐藏。Data View 指向 data stream 名即可，不需要关心底层索引：

```text
Data View: logs-order  →  匹配 data stream "logs-order"
                              └─ 实际索引 .ds-logs-order-2026.11.06-000001
```

# 3. KQL

## 3.1 基础语法

KQL（Kibana Query Language）是 Kibana 默认的查询语言，比 Lucene 更接近自然语言：

```text
# 单个字段精确匹配
level: "error"

# 全文搜索（在默认字段里找）
timeout

# 字段存在
error: *

# 字段不存在
not error: *
```

| 写法 | 含义 |
|------|------|
| `field: value` | 字段等于值（字符串可不加引号） |
| `field: "value with space"` | 含空格必须加引号 |
| `field: value*` | 前缀通配 |
| `field: *value*` | 包含（注意性能） |
| `field: *` | 字段存在 |
| `field >= 100` | 数值/日期比较 |

## 3.2 布尔与分组

```text
# 与：默认就是 and
service.name: "order-api" and level: "error"

# 或
level: "error" or level: "warn"

# 非
not level: "debug"

# 分组：优先级用括号
(service.name: "order-api" or service.name: "pay-api") and level: "error"
```

## 3.3 范围与特殊值

```text
# 数值范围
duration_ms >= 500 and duration_ms < 2000

# 时间范围（通常用右上角时间选择器，不写在 KQL 里）
@timestamp >= "2026-11-06T00:00:00.000+08:00"

# 多值（用 or）
level: ("error" or "fatal")

# 嵌套字段用点号
service.name: "order-api"
```

## 3.4 KQL、Lucene 与 DSL 的对比

| 语言 | 使用场景 | 示例 |
|------|----------|------|
| KQL | Kibana 界面默认 | `level: "error" and duration_ms > 500` |
| Lucene | Kibana 里切到 Lucene 语法 | `level:error AND duration_ms:[500 TO *]` |
| Query DSL | 代码调用 ES API | `{"bool":{"must":[...]}}` |

★ **关键认知**：**KQL 的 `field: value` 对 `text` 字段是分词匹配，对 `keyword` 字段是精确匹配**。同样是 `level: "error"`，如果 `level` 是 `text`，可能匹配到 `error_code` 这种被分词的词；如果是 `keyword`，才是真正的等值。字段类型决定了查询语义，这是最容易误解的地方。

# 4. 字段映射

## 4.1 为什么字段搜不到

排查顺序：

```text
1. Data View 的时间范围对吗？          → 右上角时间选择器
2. 索引里真的有这条文档吗？            → Discover 里去掉所有过滤
3. 字段在 mapping 里存在吗？           → GET index/_mapping
4. 字段是 text 还是 keyword？          → 决定用 match 还是 term
5. 字段名写对了吗？嵌套用点号          → service.name 不是 service_name
```

## 4.2 text 与 keyword

| 类型 | 是否分词 | 适合 | 查询方式 |
|------|----------|------|----------|
| `text` | 是 | 日志正文、消息 | 全文搜索、match |
| `keyword` | 否 | ID、状态、服务名 | 精确匹配、聚合、排序 |
| `date` | - | 时间 | 范围查询 |
| `long`/`double` | - | 数值 | 范围、聚合 |
| `ip` | - | IP 地址 | CIDR 查询 |

```json
// 查看字段类型
GET logs-order-*/_mapping/field/level
```

```json
// 典型返回：level 是 keyword，可以精确匹配和聚合
{
  "logs-order-2026.11.06": {
    "mappings": {
      "level": { "full_name": "level", "mapping": { "type": "keyword" } }
    }
  }
}
```

⚠ **关键认知**：`text` 字段**不能直接聚合**。你会看到 Kibana 提示 "field is not aggregatable"，或者聚合结果为空。解决办法是在 mapping 里加 `fields.keyword` 子字段（`"type": "keyword", "ignore_above": 256`），然后用 `level.keyword` 聚合。

```json
"level": {
  "type": "text",
  "fields": {
    "keyword": { "type": "keyword", "ignore_above": 256 }
  }
}
```

## 4.3 动态映射的坑

如果没定义模板，ES 会按第一条数据猜类型：

| 第一条值 | 推断类型 | 后续问题 |
|----------|----------|----------|
| `"123"` | text | 想按数字范围查，查不了 |
| `123` | long | 后续出现 `"123"` 字符串，bulk 失败 |
| `"2026-11-06"` | date | 格式不符的日志被拒绝 |
| `null` | 无 | 字段不出现，后续类型随机 |

★ **关键认知**：**日志索引必须显式定义 mapping**。动态映射的"猜测"一旦落定就不能改（只能 reindex），而日志字段的类型往往在第一条样本里看不出来。生产模板至少要把 ID、级别、服务名固定成 `keyword`。

# 5. 可视化

## 5.1 Discover

Discover 是排障主战场，几个必备操作：

| 操作 | 用途 |
|------|------|
| 时间选择器 | 缩小范围，避免扫全量 |
| 字段列表 | 点字段看 Top 5 值分布 |
| 添加列 | 只看关心的字段，减少噪音 |
| 保存搜索 | 把常用查询存下来复用 |
| 分享短链接 | 把现场发给同事，带上时间范围 |

## 5.2 Lens 可视化

Lens 是 8.x 的默认可视化工具，拖字段即可出图：

| 图表 | 适合 | 配置要点 |
|------|------|----------|
| Bar | 按时间/维度计数 | X 轴时间，Y 轴 Count |
| Line | 趋势 | 加时间间隔（如 1m） |
| Table | Top N | 按 `service.name` 分组 |
| Metric | 单值 | 错误率、P95 |
| Heatmap | 分布 | 时间 × 延迟区间 |

```text
典型错误率面板：
  Filter: level: "error" or level: "fatal"
  X 轴: @timestamp（自动间隔）
  Y 轴: Count
  对比: 总请求数（另一个 layer）
```

## 5.3 Dashboard

Dashboard 的组织原则：

1. **第一屏看健康**：错误率、QPS、P95 延迟、告警数。
2. **第二屏看服务**：按 `service.name` 拆分的错误 Top N。
3. **第三屏看细节**：慢请求、异常堆栈、特定 request_id 搜索入口。
4. **每个面板都带时间范围**，并且用同一个时间选择器联动。

★ **关键认知**：Dashboard 不是越多越好。**一个团队维护一个"入口 Dashboard" + 每个服务一个"详情 Dashboard"** 就够了。面板超过 20 个，加载会变慢，也没人看得过来。

# 6. 检索排错

## 6.1 一条 request_id 的排查流程

```text
1. 用户报障，提供 request_id = r-abc123
2. Kibana 搜索：request_id: "r-abc123"
3. 时间范围放宽到用户操作前后 10 分钟
4. 按 @timestamp 升序，看请求的完整生命周期
5. 找到第一条 error，看 error_type / error / operation
6. 如果是下游服务，拿 trace_id 去 APM 看调用链
7. 如果是数据问题，用业务主键（order_id）扩大搜索
```

## 6.2 常用查询模板

```text
# 某服务最近 15 分钟的错误
service.name: "order-api" and level: ("error" or "fatal")

# 慢请求（超过 1 秒）
duration_ms >= 1000

# 某个用户的全部操作
user_id: "1001"

# 排除健康检查噪音
not path: "/healthz" and not path: "/metrics"

# 某类异常
error_type: "db_timeout" or error.message: *timeout*

# 按请求 ID 精确追踪（keyword 字段）
request_id: "r-abc123"

# 特定时间段的发布后错误
level: "error" and service.version: "v1.2.3"
```

⚠ **关键认知**：`field: *timeout*` 这种**前后通配符无法使用索引**，会退化成全量扫描。数据量大时先用 `error_type` 这类 keyword 字段缩小范围，再对少量结果做文本搜索。

## 6.3 时间对不上

最常见的原因是时区：

| 现象 | 原因 | 修复 |
|------|------|------|
| 日志时间比实际晚 8 小时 | 采集时按 UTC 解析 | Logstash `date` 加 `timezone` |
| Kibana 显示时间不对 | 浏览器时区设置 | 右上角切换时区 |
| 时间范围查不到 | `@timestamp` 字段选错 | Data View 里改 Time field |
| 日志里的时间和 `@timestamp` 不一致 | 用了采集时间而非日志时间 | 用 `date` filter 覆盖 |

★ **关键认知**：**永远以 `@timestamp` 为准**，并保证它是日志产生的时间，而不是 Filebeat 读到的时间。两者在正常情况只差毫秒，但在积压、重放、容器重启时会差几分钟甚至几小时，直接影响排障。

# 7. 权限

## 7.1 Space

Space 是逻辑隔离，把不同团队的 Dashboard、Data View、保存的搜索分开：

```text
Space: order-team
  ├─ Data View: logs-order-*
  ├─ Dashboard: 订单服务健康
  └─ 成员: order-devs

Space: pay-team
  ├─ Data View: logs-pay-*
  └─ Dashboard: 支付服务健康
```

## 7.2 Role

Role 控制"能看哪些索引、能做哪些操作"：

```json
PUT _security/role/logs_reader
{
  "indices": [
    {
      "names": ["logs-order-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": {
        "grant": ["@timestamp", "level", "message", "service.*", "request_id"],
        "except": ["user.phone", "user.email"]
      }
    }
  ],
  "kibana": [
    {
      "base": ["read"],
      "feature": {
        "dashboard": ["read"],
        "discover": ["read"]
      },
      "spaces": ["order-team"]
    }
  ]
}
```

## 7.3 最小权限原则

| 角色 | 权限 | 说明 |
|------|------|------|
| 应用写入 | `create_doc` on `logs-app-*` | 只能写，不能读 |
| 开发排查 | `read` + 字段脱敏 | 不能看手机号、身份证 |
| SRE | `read` + `manage` ILM | 能调保留策略 |
| 管理员 | `all` | 严格限制人数 |

★ **关键认知**：**日志系统里往往藏着比业务库更全的数据**——请求体、错误栈、用户标识都可能落盘。所以权限要按"最小必要"给，字段级安全（`field_security`）不是可选项，而是合规底线。

# 8. 常见错误

| 问题 | 现象 | 修复 |
|------|------|------|
| Data View 写成具体日期 | 第二天查不到 | 用 `logs-app-*` |
| 时间字段选成 `log.file.path` | 时间过滤失效 | 选 `@timestamp` |
| 用 `text` 字段聚合 | 结果为空 | 用 `.keyword` 子字段 |
| KQL 里写 SQL 语法 | 语法错误 | 用 `and/or/not`，不是 `AND/OR` |
| 前后通配符查大索引 | 超时 | 先用 keyword 缩小范围 |
| 时区没配 | 时间差 8 小时 | `date` filter + 浏览器时区 |
| 给所有人 `superuser` | 数据泄露风险 | 按团队建 Role + Space |
| Dashboard 面板过多 | 加载慢 | 拆分，控制在 20 个以内 |

# 9. 练习与自检

1. 写出三条 KQL：① 查 `order-api` 最近所有 `error` 和 `fatal`；② 查 `duration_ms` 在 500 到 2000 之间的请求；③ 排除 `/healthz` 路径且 `user_id` 存在的日志。
2. 一个字段在 Discover 里能看到值，但做 Terms 聚合时提示不可聚合。请解释原因并给出两种修复方式。
3. 用户说"日志时间晚了 8 小时"，请列出从应用到 Kibana 的完整排查路径，并指出最可能出问题的一环。

# 10. 小结

| 主题 | 要点 |
|------|------|
| Kibana | 前端 + 代理，不存数据 |
| Data View | 索引 pattern + 时间字段，pattern 用通配 |
| KQL | `field: value`，布尔用 `and/or/not` |
| 字段类型 | `text` 分词，`keyword` 精确，聚合用 keyword |
| mapping | 显式定义，动态映射会猜错且难改 |
| 可视化 | Discover 排障，Lens 出图，Dashboard 分层 |
| 检索 | 先用 keyword 缩小范围，再文本搜索 |
| 权限 | Space 隔离 + Role 最小权限 + 字段脱敏 |

下一篇进入生产实践，把前面八篇串起来：日志链路如何端到端设计、成本怎么控制、安全合规怎么落地、告警怎么配、故障怎么排查，最后给出整个模块的总结。
