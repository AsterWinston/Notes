# 1. 概述

Stack Management 是 Kibana 的**管理控制台**，提供对 Elastic Stack 的配置和管理能力——它是平台管理员的"总装配间"：从 ES 索引生命周期、到 Kibana 自身的对象/空间/设置、到安全主体与许可证，全部从这里进出。它与 Dev Tools 的分工要划清：**Stack Management 是带校验、带依赖检查、带审计的结构化操作面；Dev Tools 是直发 REST 请求的原始通道**。日常变更走 UI（有提示、防手滑），批量与自动化走 API（可编排、可回滚），二者操作的底层是同一套 REST 接口——UI 里每个操作都能在 Dev Tools 里用请求复现，排查"UI 为什么失败"时打开 Network 面板反查真实请求是管理员基本功。

```text
Stack Management 分类:
  ├── Elasticsearch
  │     ├── Index Management
  │     ├── Index Templates
  │     ├── Index Lifecycle Policies (ILM)
  │     └── Watcher (Advanced Alerting)
  ├── Kibana
  │     ├── Data Views
  │     ├── Saved Objects
  │     ├── Spaces
  │     ├── Advanced Settings
  │     └── Tags
  ├── Security
  │     ├── Users / Roles / API Keys
  │     └── Role Mappings
  └── Stack
        ├── License Management
        └── Upgrade Assistant
```

# 2. Dev Tools Console

Dev Tools 是开发提效第一入口，Console 的本质是**带语法高亮、自动补全、变量注入的 REST 客户端**，请求直接由 Kibana 以当前用户身份代理转发 ES。核心能力按使用频率排：

- **自动补全**：键入 `GET _cat` 后 Ctrl+Space，索引名、API 路径、DSL 字段全部联想；
- **多请求顺序执行**：光标所在请求按 Ctrl+Enter（Win/Linux）或 ⌘+Enter（Mac）单发，多个请求体间用空行分隔依次执行；
- **别名/变量与复制为 cURL**：右键请求 → Copy as cURL，把 Console 里的操作变成脚本或工单附件；
- **Console 专属语法**：`###` 注释分块、行内 `//` 注释、`DELETE` 简写为 `D`、`POST` 简写为 `P`（可配置 `editor.shortcut` 类参数，团队统一后极省手速）。

```bash
# Console 中直接执行（无需任何鉴权配置，以登录身份代理）
GET _cluster/health
PUT orders-2026.11
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1 }
}
POST orders-2026.11/_search
{
  "size": 5,
  "query": { "match": { "status": "active" } },
  "aggs": { "by_day": { "date_histogram": { "field": "created_at", "calendar_interval": "day" } } }
}
# 简写别名: D = DELETE, P = POST, G = GET, PU = PUT, PO = POST(保留)
```

Kibana 自身的内部 API 也能在这里调（`GET /api/spaces/space`、`POST /api/alerting/rules/_find`），但这类接口**非稳定契约，版本升级可能变化**——生产自动化优先用 `_security`、`_ilm`、`_reindex` 等受支持的 ES API，或注明支持范围的 Kibana API。

> **注意**：Console 里执行 `DELETE /logs-*` 这类批量删除**没有二次确认**，与 UI 行为完全不同——写错 pattern 的代价是灾难性的。团队规范应约定：破坏性操作一律先跑对应的 GET 预览命中范围，确认无误再改方法名。

# 3. Index Management

Stack Management → Index Management 提供对索引、数据流、别名的可视化管理，三个页签对应三类对象：

```text
Stack Management → Index Management

Indices:
  - 查看所有索引列表、大小、文档数、状态
  - 打开/关闭/删除/强制合并索引
  - 查看 Index Settings 和 Mappings

Data streams:  ← 8.x 主流写入形态（集成默认产出）
  - 查看 backing indices、generation、匹配模板

Aliases:  ← 读写分离与零停机 reindex 的枢纽
  - is_write_index 切换、别名下索引增删
```

UI 上常用操作的真实代价各不相同：**Edit settings 里只有带"dynamic"标记的设置（如 `refresh_interval`、`number_of_replicas`）能热改**，分片数必须 reindex/split；Force merge 对仍在写入的索引是反模式（只该对 ILM 已进入 warm/cold 的只读索引做，把段合并到 1–5 个换查询性能与存储）；**Shrink/Split** 要求索引先置为只读再操作。

> **注意**：批量操作入口（Actions → Edit settings / Manage index）允许一次选中多个索引，**选中范围以 pattern 表示时要警惕 `logs-*` 命中的比预期多**——执行前用右侧 "Show affected" 或先在 Console 用 `GET /logs-*/_settings?include_defaults=false` 预览。

# 4. Index Templates 与 ILM

## 4.1 Index Templates

Index Template（composable 模型）决定**新索引出生时的样子**，priority 与 `data_stream` 标记是关键：

```text
Stack Management → Index Templates

创建 Template:
  Name: order-template
  Index pattern: orders-*
  Priority: 100
  Settings: number_of_shards=3, number_of_replicas=1
  Mappings: { "order_no": {"type": "keyword"}, "amount": {"type": "double"} }
  Aliases: orders-alias
```

8.x 的集成生态依赖 **index template + ILM policy + ingest pipeline** 三件套定义 data stream，手工模板若也设 `data_stream`，则匹配的索引必须是 data stream backing index。模板优先级冲突（多个 pattern 命中同一索引）按 priority 大者胜，`GET _index_template/order-template` 的 `composed_of`/`priority` 字段排查问题一目了然。

## 4.2 ILM 策略

ILM 界面把策略编排成四个 phase 的可视化时间轴，滚动与降冷全托管：

```text
Stack Management → Index Lifecycle Policies

创建 ILM Policy: "logs-policy"
  Hot phase:
    Rollover: max_size=50GB OR max_docs=500M OR max_age=30d
  Warm phase (min_age=3d):
    Force merge: max_num_segments=1
    Shrink: number_of_shards=1
  Cold phase (min_age=30d):
    Searchable snapshot   ← 数据移入快照存储，查询仍可达
  Delete phase (min_age=90d):
    Delete
```

**设计考量**：

- **rollover 条件用 size 为主、age 为兜底**：只设 max_age 会让写入不均的索引大小悬殊；只设 max_size 则可能一个巨型索引永远不滚——两者组合。
- **shrink 的前提**：主分片数必须能整除目标数，且 shrink 前索引需要被 force merge 否则迁移很慢；Warm 的 min_age 要给足查询热度时间，别过早降冷影响 Discover 体验。
- **Cold 的 searchable snapshot 是存储成本分水岭**：适合"查但极少查"的合规留存段，恢复依赖快照仓库健康。

ILM 报错（action 卡 failed）八成源于**索引状态不满足动作前提**（未只读、分片不整除、allocation 不满足），用 `POST /orders-*/_ilm/move` 手工修状态是危险动作，优先修条件让策略自然重试。

# 5. 快照与恢复 UI

Stack Management → Snapshot & Restore 把 `_snapshot` API 的注册仓库、SLM（快照生命周期）、恢复三件事做成表单：

```json
PUT /_slm/policy/daily-logs
{
  "schedule": "0 30 1 * * ?",
  "name": "<logs-{now/d}>",
  "repository": "repo-s3",
  "config": { "indices": ["logs-*", ".kibana*"] },
  "retention": { "expire_after": "30d", "min_count": 5, "max_count": 50 }
}
```

UI 中 **Restore snapshot 向导**控制粒度到"恢复哪些索引/是否重命名/是否跳过未分配副本"——`wait_for_completion` 长任务在界面走后台进度条，比 curl 友好得多。`.kibana*` 索引必须纳入快照计划，否则"数据都在、配置全丢"。

# 6. Upgrade Assistant

Upgrade Assistant 是升级前的**合规扫描清单**：逐项检查废弃设置、旧索引模板、过期 API 使用（`_get_all` 类弃用端点）、ML 模型兼容性、Saved Objects 结构迁移，给出"可安全升级 / 需处理项"两态。正确姿势是**升级前一周打开它处理完所有 yellow/red 项**——8.x 每次大版本升级最耗时的往往不是安装，而是清理历史遗留：比如 7.x 时代的手工模板升级为 composable、`_doc` 里被弃用的字段格式。

# 7. Saved Objects 与 Advanced Settings

## 7.1 Saved Objects

```text
Stack Management → Saved Objects

按类型查看/管理:
  - Dashboards / Visualizations / Maps
  - Index Patterns / Searches / Tags

操作:
  - Export: 导出为 NDJSON（含所有依赖）
  - Import: 从 NDJSON 导入
  - Copy to Space: 跨 Space 复制
  - Delete: 删除（含依赖检查）
```

导出/导入是环境间搬运看板的**唯一推荐通道**：勾选 "Include related objects" 让引用的 search/lens/data view 一并带走；冲突策略（Overwrite/Create new）在大批量迁移时必须明确选择，默认行为容易把目标环境的同名对象悄悄覆盖。

## 7.2 Advanced Settings

```text
Stack Management → Advanced Settings

常用设置:
  dateFormat: "YYYY-MM-DD HH:mm:ss"
  defaultIndex: "logs-*"
  csv:maxSizeBytes: 10485760 (10MB 导出上限)
  discover:sampleSize: 500
  doc_table:hideTimeColumn: false
  histogram:maxBars: 100
  state:storeInSessionStorage: false
  theme:darkMode (Dark Theme)
```

Advanced Settings 以 `config` 类型 Saved Object 持久化，**改动即时全实例生效**（存 ES，非单机内存）。`state:storeInSessionStorage: true` 在 URL 超长（看板 filters 多）时值得开启，代价是分享链接失效；`dateFormat:tz` 设为浏览器时区还是 UTC，是全团队观感一致性的源头，应作为平台级约定而非个人偏好。

# 8. 安全主体与许可证

Security 页管理用户/角色/API Key/Role Mapping（详见 Spaces 与 RBAC 篇），日常最高频的是 **API Keys 页签盘点**：每个 Key 显示归属用户、创建时间、最后使用时间——**长期未使用的 Key 是隐形后门**，季度清理。License 页显示当前档位、过期时间与功能开关，trial 期间创建的功能对象在 trial 过期后**进入只读**，扩容/续期决策提前看这里。

# 9. 开发提效技巧汇总

- **Console 请求库**：团队把常用运维查询（健康检查、ILM 状态、慢日志开关）沉淀成共享 .txt 片段，新人直接用。
- **Deep Link**：Discover/看板 URL 本身就是完整上下文（含 KQL、时间窗、filters），`Share → Generate link` 出短链进工单。
- **Dev Tools 里调 Kibana API**：配合 `kbn-xsrf: true` 头测试告警连接器、批量建规则，避免反复点 UI。
- **Inspect**：每个面板右键 Inspect → Request/Response，看真实 DSL 与耗时，是调慢查询的起点。
- **导入导出做"看板即代码"**：NDJSON 入 Git，CI 调 `POST /api/saved_objects/_import?overwrite=true`，评审流管看板变更。

> **行业实践**：把"改平台配置必走 Stack Management + 留工单"写进变更流程，Dev Tools 的破坏性操作限定在只读排查用途。Kibana 的管理面没有 ES 那样细的审计日志，**行为审计靠 `.kibana-event-log-*` 安全事件 + 快照兜底**：任何批量删除/导入前，先手动打一个含 `.kibana*` 的快照，成本几秒、回滚保命。

# 10. Watcher (X-Pack)

Watcher 提供 **ES 端的脚本化告警**（Platinum+），比 Kibana Alerting 出现更早、更贴近引擎但更复杂，8.x 定位是存量兼容：

```json
PUT _watcher/watch/order_alert
{
  "trigger": { "schedule": { "interval": "5m" } },
  "input": {
    "search": {
      "request": {
        "indices": ["orders-*"],
        "body": {
          "query": { "range": { "amount": { "lt": 100 } } }
        }
      }
    }
  },
  "condition": { "compare": { "ctx.payload.hits.total.value": { "gt": 10 } } },
  "actions": {
    "email_admin": {
      "email": {
        "to": "admin@example.com",
        "subject": "Low amount orders detected"
      }
    }
  }
}
```

新告警一律建议用 Kibana Alerting（对比表见告警篇），此处仅保留运维入口：`GET _watcher/watch/_execute` 手工试跑、`_stats` 看执行历史、停用 watch 防重复告警。

# 11. 小结

- **两个操作面的分工**：Stack Management 是带校验与依赖检查的结构化管理面，Dev Tools 是原始 REST 通道——批量自动化与排障（看真实请求/Inspect 面板）都离不开后者，破坏性请求则收进 UI 管控。
- **索引管理三件套**：template 决定出生、ILM 决定生命周期、快照/SLM 决定生后可救；force merge/shrink 只在只读段做，rollover 用 size+age 组合。
- **Saved Objects 是平台资产**：NDJSON 导出 + overwrite 策略做"看板即代码"，`.kibana*` 必须进快照计划，升级前一周用 Upgrade Assistant 清完黄红灯。
- **管理功能的隐形价值**：API Key 最后使用时间、Advanced Settings 的全局即时生效、Watcher 的只读兼容——多数事故源于"改动没人知道"，把变更入口收敛并留痕比功能本身更重要。
