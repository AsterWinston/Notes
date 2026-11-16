# 1. 概述

Discover 是 Kibana 的**交互式数据探索工具**，提供对 ES 数据的实时搜索、筛选和预览能力。它在整个 Kibana 工作流里的定位是"看明细"：Dashboard/Lens 回答"趋势和占比是什么"，Discover 回答"具体是哪些文档、原始内容长什么样"。日志排障场景下，工程师 80% 的停留时间都在 Discover——时间过滤缩小窗口、KQL 精确命中、展开文档看完整 JSON、保存查询复用，这套动作是日常肌肉记忆。

与相邻功能的分工值得先讲清楚：**Discover vs Lens**——Discover 返回原始文档列表（带 `_source`），Lens 返回聚合结果，一个看"行"一个看"统计"；**Discover vs Dev Tools**——Dev Tools 手写 Query DSL 调试索引，Discover 面向无 DSL 经验的值班同学；**Discover vs Search Profiler**——前者日常检索，后者分析查询为什么慢。三者共同构成"检索-分析-调优"闭环。

```text
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
│  └─────────────┘  │  - Popular Fields         │     │
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

# 2. 界面三要素与阅读顺序

一次 Discover 查询结果由三块信息组成，按"先宏观后微观"的顺序阅读效率最高：

**顶部文档计数 + 时间直方图**：显示当前时间窗口内命中总数，以及按 interval 自动分桶的柱状图（计数随 KQL 过滤实时联动）。直方图的作用不是精确统计，而是**形状**——尖刺意味着突发（错误洪峰、流量尖峰），断崖意味着数据中断（采集中断、索引切换出故障），均匀则一切正常。点击某根柱子、或拖拽选取一段区间，立即把时间范围缩到那里，这是从"宏观异常"下钻"微观文档"的标准动作。

**左侧字段面板**：分 Selected（当前显示列）、Available（可添加列）、Stored（仅存在于 `_source`、mapping 里没建索引的字段，不能过滤排序）、Runtime 四组。点击字段名查看该字段的 **Top 5 值分布与占比**——这相当于对当前过滤结果做了一次即时 terms 聚合，是判断"过滤是否命中预期维度"的快速手段；Popular 区自动列出出现频率最高的字段。

**中央文档表格**：按时间倒序分页展示命中文档，每页 500 条（`documents:pageSize` 可调），翻页到深页（如第 200 页）会触发 ES 的 from+size 深分页限制（1 万条），此时应缩小过滤而非继续翻页。每行左侧 `>` 展开单文档详情，`<`/`>` 前后翻条，行内字段值点击即添加/移除过滤条件（见 2.1）。

> **注意**：文档计数默认只精确到 1 万（ES `track_total_hits` 默认行为，Kibana 对大索引会显示 `10,000+`）。不要拿 Discover 的总数做精确统计——计数超 1 万时它本身就不精确；需要准确聚合数字请回到 Lens/Dashboard。

## 2.1 搜索与筛选

检索语法用 KQL（详见 6_kql 篇），常用形态：

```kql
status:active                          # 精确匹配
amount > 100                           # 范围
customer_full_name: "John Doe"         # 带空格的值
user*                                  # 通配符
status:(active OR pending)             # 布尔
NOT status:cancelled                   # 排除
```

过滤条件除了 KQL 文本，还可通过**过滤芯片（filters）**叠加，芯片以 KQL 之外的结构化条件存在，与 KQL 文本 AND 合并：

```text
方式 1: 点击文档表格中的字段值 → "+" 添加
方式 2: Search Bar 左侧 → Add filter → 选择字段和操作符
方式 3: Fields Panel → 字段名 → 选择值

过滤器操作符:
  is / is not / is one of / is between
  exists / does not exist
```

每个芯片可单独 disable（灰掉保留）/ enable / 删除，排障时常用手法是"挂一串 is not 芯片逐层排除"，比反复改 KQL 文本更直观。

## 2.2 列设置与文档展开

Selected Fields 控制表格列：点字段旁 ★/加号加入，拖拽调列序，点击列头排序（仅当前页生效——这是常见误解，Discover 的排序是"页内排序"，要全局按字段排序得靠时间字段或改用 _source 导出）。文档展开有两种视图：**Table**（字段→值树形浏览，支持按路径过滤字段）与 **JSON**（原始 `_source`，保留换行与嵌套结构，看多行日志原文首选 JSON 视图）。`_source` 里未 mapping 的字段只能在 JSON 视图看到——这正好用来验证写入端 mapping 模板是否漏字段。

# 3. 时间过滤器

时间过滤器（右上）是 Discover 的"第一生产力"，用法要点：

```text
时间选择器（右上）：
  - Quick Select: Last 15 minutes / 1 hour / 7 days
  - Absolute: 2024-01-15 10:00 → 2024-01-15 18:00
  - Relative: 从现在往前 24 小时

时间过滤器的使用:
  - 影响所有面板（Dashboard 全局）
  - 点击时间线柱状图 → 快速放大某时间段
  - 拖拽时间线 → 选择范围
```

Relative 与 Absolute 的语义差异要记牢：**Relative 每次刷新/进入页面都重新计算**（"最近 15 分钟"永远是滚动窗口），适合实时监控；**Absolute 固定起止时刻**，适合复盘已知故障窗口，且绝对时间可分享链接给他人复现完全相同的视图。Auto-refresh 下拉可设 5s/10s/30s… 轮询新数据，排障时开 30s 即可，不要常开 5s 给 ES 制造无意义压力。直方图上再拖拽可"套娃"缩窗，配合浏览器后退键逐层回退，这是从"全天"下钻到"某个分钟"的标准节奏。

# 4. 保存搜索

重复使用的查询保存为 Saved Search（Saved Object 类型 `search`），它把 KQL + filters + 列配置 + 时间设置打包成一个可引用对象：

```text
Discover 右上角 → Save
  Title: Active Orders over $100
  Tags: orders, important

加载已保存的搜索:
  Discover → Open → 选择已保存的搜索
```

保存搜索的最大价值在**复用**：Dashboard 可以把 Saved Search 作为一个面板类型直接嵌入（显示命中数与表格），告警 Rule 也可以用它作为查询源——"保存一次查询，界面、大屏、告警三处引用"，避免口径漂移。命名与打 Tags 是团队资产管理的开始，没有 Tags 的几百条搜索对象半年后就是黑盒。

# 5. 搜索会话（Search Sessions）

8.x 新增：Search Sessions。它把"一次检索"变成可管理的持久化会话：

```text
8.x 新增：Search Sessions

Discover → 搜索结果 → Save session
  - 暂停搜索 → 恢复后继续
  - 后台运行长时间搜索 → 稍后查看结果
  - 适合大数据量的聚合查询

Management → Search Sessions → 查看/管理会话
```

背后的演进是 ES 8.x 的**可恢复检索（recoverable search）**：大结果集不再依赖单次同步 HTTP 响应返回，Kibana 持有会话游标，翻页、断线重连、后台续跑都基于它，同时规避了传统 scroll 上下文占用资源的问题。运维侧记住：会话有生命周期（`search:timeout` / `search:mappingsTimeout` 等 Advanced Settings 控制），到期自动清理；会话记录存在 `.kibana_search_sessions` 相关对象里，排查"谁在跑大查询"时这里是线索。

# 6. 数据导出

Discover 的 Share → CSV Reports 把当前过滤结果导出为 CSV（离线交给 Excel/分析工具）：

```text
Discover → Share → CSV Reports
  生成 CSV 文件下载

注意:
  - 最多导出 10,000 条（默认配置，来自 kibana:csv:maxRows）
  - 可通过 Advanced Settings 调整 csv:maxRows / csv:maxBytes 上限
```

导出文件走 Kibana Reporting 引擎异步生成（与 PDF 同一套），大导出在 Management → Reporting 里跟踪进度。超过 1 万行的需求不要挑战这个上限——那是 ES 快照/reindex 到临时索引再 `_search` 分页或 ESQL `SAVE TO` 的活；CSV 导出定位是"看样本"，不是"搬数据"。

> **行业实践**：给值班同学的 Discover 使用规范比给权限管控更管用——先缩时间再搜索、字段列别贪多（≤8 列）、导出前确认数据脱敏（日志里的手机号/token 会原样进 CSV）。生产日志导出属于敏感操作，建议在角色层面控制 `csv` 导出权限（privileges 里 Discover 的 export 权限项）。

# 7. 排障实战：原始日志过滤流程

一段线上故障（下单接口报错率上升），Discover 的标准排查流程五步走：

```text
1. 选视图定窗口：Data View = app-logs-*，时间 Absolute 覆盖故障时段
2. 看直方图形状：定位错误尖刺的精确分钟 → 点击柱子缩窗
3. 逐层过滤：level:ERROR → service:order-* → NOT url:"/health" 去噪音
4. 字段分布辅助：点 message / status 字段看 Top 值，锁定集中出现的异常类名/状态码
5. 展开样本：JSON 视图看完整堆栈与 trace_id → 跳转 APM 或保存搜索/分享链接给协作方
```

关键习惯：**每一步过滤后先确认计数与分布符合预期再下钻**，避免"过滤器拼错→0 命中→以为没日志"的误判；`trace_id` 是跨服务串联的钥匙，从一条错误日志反查全链路其他服务日志，正是 Discover 多视图 Tab（8.7+）的典型用法。

# 8. Discover 最佳实践

1. **限制字段数**：Selected Fields 仅添加需要的列（避免全量 _source 渲染）
2. **使用 saved search**：常用查询保存为 saved search，Dashboard 可直接引用
3. **时间范围先行**：先缩小时间范围再搜索（减少 ES 负载）
4. **字段别名**：Data View 中给长字段名设置别名
5. **避免 `*` 全模糊**：KQL 中 `field*` 使用前缀而非全模糊

补两条与列表现代相关的：善用 `@` 与 `:` 触发的自动补全减少拼写成本（依赖 field_caps 与 top values 缓存，首次补全有延迟属正常）；以及用 Discover 顶部的 **Inspect → Request/Response** 查看本次检索实际发往 ES 的 DSL 与耗时——当界面"转圈很久"时，先看 DSL 的 `took` 与命中索引列表，能立刻分清是查询写宽了还是集群慢了。

# 9. 小结

Discover 是"看明细"的工作面：顶部计数与直方图给宏观形状（尖刺=突发、断崖=中断），左侧字段面板做即时分布，中央表格逐条看原文。使用顺序永远"时间先行→KQL/芯片过滤→字段分布定位维度→展开 JSON 看堆栈→保存搜索/分享链接"。保存的搜索是团队资产，Dashboard 与告警都能引用以统一口径；Search Sessions 让大检索与翻页在 8.x 变成持久化会话，CSV 导出只用于取样本（1 万行默认上限，别当 ETL 用）。性能红线就三条：先缩窗口、少选列、不用全模糊通配。
