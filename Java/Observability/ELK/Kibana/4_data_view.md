# 1. 概述

Data View（8.0 前称为 **Index Pattern**）定义 Kibana 如何识别 ES 索引中的字段，是所有搜索和可视化的基础。它的本质是一份"字段字典 + 索引映射清单"：Kibana 拿它去问 ES 的 field_caps API，把匹配到的索引的 mapping 拉回来、解析成字段列表（名称、类型、是否可聚合、计数），Discover 的字段面板、Lens 的拖拽区、Dashboard 的时间选择器全都依赖这份清单。没有 Data View，ES 里再多的索引对 Kibana 也只是"不可见的数据"。

7.x→8.x 改名不只是换叫法：旧版一个 Index Pattern 只能对应一种通配符标题，且与"时间字段"强绑定（非时序索引要么选无时间字段、要么体验残缺）；Data View 允许**一个视图合并多个不同 pattern**（见 6 节），弱化了对时间字段的强制要求，还支持在视图级别直接挂 Runtime Fields。

```text
ES Index: orders-2024-01, orders-2024-02, ...
    │
    ▼
Data View: orders*
  ├── Time field: created_at
  ├── Fields: order_no(keyword), amount(double), status(keyword), ...
  └── Runtime Fields: day_of_week(script)
```

> **注意**：Data View 与 `logs-*`、`metrics-*` 这类 **Alias / Data Stream 是配合而非替代关系**。推荐让 pattern 指向 alias 或 data stream 名（如 `orders`），由别名层管理物理索引的切换与 rollover——这样 ILM 滚动出新索引时 Data View 完全不用改；若 pattern 直接写死日期后缀，视图就会"断代"。

# 2. 创建 Data View

UI 路径是 Stack Management → Data Views → Create，填三个必选项：名称、索引 pattern、时间字段。自动化/IaC 场景走 Kibana API（注意这是 **Kibana 的 API**，端口 5601，用 Kibana 会话或 API Key 鉴权，不是 ES 的 9200）：

```bash
# Kibana API 创建
POST /api/data_views/data_view
{
  "data_view": {
    "title": "orders-*",
    "name": "Orders",
    "timeFieldName": "created_at"
  }
}
```

创建过程中的两个关键决策：

**时间字段怎么选**：只有 date/date_nanos 类型字段可选。选错或漏选的直接后果是时间过滤器失效——Discover 顶栏变灰、Dashboard 无法按时间联动、折线图 x 轴无法自动分桶。若 mapping 里时间是 `keyword` 存字符串，Kibana 不会提示，只会"静默不可用"，务必确认写入端 mapping 正确（`dynamic: false` 的模板常埋这个雷）。无时间属性的配置类/维表索引可以选 `@timestamp` 之外的"不指定时间字段"（Use no time field），此时视图仅失去时间过滤能力，检索仍可用。

**多 pattern 合并**：一个视图可以同时包含 `app-logs-*` 与 `gateway-logs-*`，8.x 在 Advanced settings 里勾选后，创建界面可输入多个 pattern。前提是各索引同名字段类型一致，否则类型冲突字段会以"冲突字段（conflict）"形式展示且无法聚合。

# 3. 通配符模式

Pattern 用通配符匹配索引集合，规则与 ES 索引名通配符一致（仅 `*`，`*` 可出现在任意位置）：

| Pattern | 匹配 | 说明 |
|---------|------|------|
| `orders-*` | orders-2024-01, orders-2024-02 | 所有 orders 前缀 |
| `logs-2024.01.*` | logs-2024.01.15, logs-2024.01.16 | 指定月份 |
| `*-logs-*` | app-logs-prod, nginx-logs-dev | 包含 logs 的索引 |
| `orders` | orders（精确） | 单个索引 |

三个容易踩的写法：`orders*`（无横线）会额外匹配 `orders-archive`、`orders_test` 等非预期索引，命名约定不一致时宁可用 `orders-*`；`*` 单独作为 pattern 会匹配到 `.kibana*` 等系统索引与所有数据流内部索引，字段面板被 `migration.*`、`deprecation.*` 之类元字段淹没——**永远不要用裸 `*` 建生产视图**；以 `-` 开头的 pattern（如 `-.kibana*`）会被拒绝，系统索引用 dot prefix 命名本身就是在提醒你别碰。

> **行业实践**：pattern 尽量指向 **alias 或 data stream**，把"哪些物理索引属于这个视图"的决定权交给别名管理。典型生产写法是 `orders`（指向滚动别名）而非 `orders-*`；跨服务统一检索时再合并 pattern 或用 `service-logs-*` 这种统一命名规范兜底。

# 4. 字段管理

## 4.1 字段类型、计数与格式化

Data View 的 Fields 页签列出全部字段，每个字段带四类信息，排障时都用得上：

| 信息 | 含义 | 用途 |
|------|------|------|
| **Type** | ES 映射类型（keyword/text/long/date…） | 决定可用操作（范围比较、聚合） |
| **Count** | 最近 24h（可配 `advanced:defaultIndex` 统计窗口）文档中出现次数 | 判断字段覆盖率、是否已停写 |
| **Scripted/RT** | 是否计算字段 | 性能预估（见 4.2） |
| **Metadata** | `esCustomProperty` 等 mapping 自定义属性 | 标记敏感字段、单位 |

字段格式化（Format）是**纯展示层**的改写，只影响 Discover 表格与 Lens tooltip 的呈现，不改变 ES 里的数据：

```text
Data View → 字段列表 → 编辑字段 → Format

常见格式化:
  - Bytes: 数字 → "1.5 MB"
  - Duration: 毫秒 → "1h 30m 25s"
  - Number: 数字 → "1,000,000.00"
  - URL: 字段值 → 可点击链接
  - String: 字段值 → 大写/小写
  - Color: 数值 → 颜色条
```

另一个高频操作是设置字段的**默认显示行为**：`Population layer`（是否默认列进 Discover 表格）、别名（Alias，长字段名缩短）、排序权重。把 `message`、`error` 这类高价值字段设为默认列，Discover 打开即见关键信息。

## 4.2 Runtime Field

Runtime Field 是在 Data View（或 index mapping 的 `runtime` 段）中动态定义的**查询期计算字段**，无需重建索引、无需 reindex：

```text
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

它的执行模型是：查询命中每个文档时，ES 在检索线程上运行 script 并 `emit()` 出虚拟字段值，供过滤/聚合使用。因此 runtime field 的性能代价**与命中集大小成正比**，时间窗口收窄（如 last 1h）时几乎无感，全索引扫描时明显变慢且无法下推优化。字段固定类型建议用 date_nanos/double/keyword，与脚本 emit 的类型严格一致，否则报 type mismatch。

**Runtime Field vs Scripted Field**：

- Runtime Field：8.x 推荐，ES 端执行，可参与查询/聚合/排序，字段列表中立于 mapping 字段统一管理
- Scripted Field：旧版机制，在 Kibana 端执行，仅影响展示不进入可聚合字段集，且拖累所有面板——能迁则迁

## 4.3 常见坑：新字段/新索引不出现

Data View 的字段列表是**缓存**（存在 Saved Object 的 `fields` 属性里），按策略定期刷新（默认 `7_days` 时间窗口内的 `date` 字段计数，自动刷新周期由 `advanced:histogram:interval` 与视图刷新策略决定）。典型症状与解法：

| 症状 | 原因 | 解法 |
|------|------|------|
| 新字段在 Fields 里搜不到 | 视图字段缓存未刷新 | Data View → Refresh field list（右上角） |
| 新索引（如明天日期的）不被匹配 | pattern 静态通配未覆盖 | 检查 pattern；改用 alias/data stream |
| 字段类型显示 text 无法聚合 | mapping 动态推断成 text | 用 `.keyword` 子字段，或修索引模板 |
| 字段计数全空 | 统计窗口内无数据 | 扩大时间范围确认字段确实有值 |
| 同名字段类型冲突 | 多索引合并 pattern 类型不一致 | 统一写入模板或拆分视图 |

> **注意**：Refresh field list 只是重新调 field_caps，**不会改 mapping、不会动数据**，可放心频繁执行；但如果新字段在 ES 里根本还没写入过文档，刷新后也不会出现——先确认数据再怀疑 Kibana。

# 5. 字段过滤（Source filters）

对宽表索引（几百个字段），Kibana 提供了 Source filters：在 Data View 设置里排除不需要在 Discover/Lens 字段面板展示的字段（如 `@version`、`beat.*`、`host.*`、`ecs.*` 等 ECS 公共字段），只影响**界面呈现**：

```text
Data View → Source filters

排除不需要的字段（减少界面噪音；_source 抓取时也更轻）:
  _source
  @version
  beat.*
  host.*
  input.*
```

它减少的是字段列表噪音与文档展开时的 JSON 体积感知，查询 DSL 并不会因此变成 `_source: {...}` 过滤（那是 Lens 的 Select fields 与 ES 侧 stored fields 优化路径），所以别对性能收益抱太高期望——它的本职是**降噪**。

# 6. 多 Data View 策略

视图组织反映团队的数据边界，两种主流策略：

```text
按团队/应用拆分：
  - order-service-*   → Order Team
  - user-service-*    → User Team
  - nginx-*           → Ops Team

统一 Data View（跨应用搜索）：
  - service-logs-*    → 所有服务日志
  - metricbeat-*      → 所有主机指标
```

实践建议：**两者都要有**。按团队拆的视图是日常工作面（字段面板干净、权限好收口、Discover 打开即用）；跨服务统一视图（通常命名为 `all-logs-*`）专门用于故障排查时"一把捞"，代价是字段面板长、需要靠 Source filters 降噪。8.7+ 的 Discover 支持在同一标签页附加多个数据视图并行检索，跨视图排障体验已明显改善。命名上保持与索引命名规范同源（`<团队>-<域>-<环境>-*`），避免视图名与 pattern 脱节。

# 7. API 管理

Data View 是普通 Saved Object（类型名仍叫 `index-pattern`），除 UI 外全部可用 API 操作，便于 IaC（Terraform provider、ansible）与跨环境迁移：

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

用 curl 调这些接口时注意鉴权方式：用户名密码（Basic）只适用于开了 API Key/Basic 认证的环境，更多时候用**浏览器会话 Cookie**（F12 复制）或 Stack Management 生成的 API Key（`Authorization: ApiKey <token>`）。跨环境迁移推荐走 Saved Objects 的 `_export/_import`（type=index-pattern），比裸 API 重建更省事，还能带上格式化与 Runtime Field 配置。

# 8. 小结

Data View 是 Kibana 一切检索与可视化的地基：本质是"索引 pattern + 时间字段 + 字段缓存字典"的 Saved Object。创建时先定三件事——pattern 指向 alias/data stream 而非裸日期后缀、时间字段必须是真正的 date 类型（否则全链路时间功能静默失效）、多 pattern 合并时警惕同名字段类型冲突。字段管理面记住三个高频动作：Refresh field list 解决"新字段不出现"、Format 做展示层美化、Runtime Field 把旧 Scripted Field 的临时计算迁到 ES 端。组织策略上"团队视图 + 全局排障视图"并存最实用，自动化维护走 /api/data_views 或 Saved Objects 导出导入。
