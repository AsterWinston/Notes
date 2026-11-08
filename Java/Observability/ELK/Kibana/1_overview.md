# 1. 概述

Kibana 是 Elastic Stack 的**可视化窗口**，提供对 Elasticsearch 数据的搜索、分析和图表展示能力。2013 年与 Elasticsearch 一同开源，是"ELK"中的"K"。但今天再看 Kibana，把它的角色仍限定为"ES 的前端图表工具"已经不准确了：8.x 的 Kibana 是整个 Elastic 技术栈的**统一操作入口**——日志检索、指标监控、APM 链路分析、安全告警、索引生命周期管理，全部从这个 Web 应用进入，ES 反而退居为"存储与计算引擎"。

理解 Kibana 定位的关键一点：**它自身不持久化任何业务数据**。Kibana 里看到的每一张图表、每一条查询结果都实时来自 Elasticsearch；Kibana 自己产生的唯一数据是"配置元数据"（Saved Objects：仪表盘定义、查询、告警规则等），而这些元数据同样存储在 ES 的 `.kibana` 系统索引里。因此 Kibana 是一个**无状态服务**，可以随意横向扩容、随时重启，不丢任何数据——这一点决定了它的部署模型和高可用方案与 ES 完全不同。

```text
┌──────────────────────────────────────────────────────┐
│                   Kibana                              │
│                                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ Discover │ │Dashboard │ │  Lens    │ ← 数据可视化  │
│  │ (探索)    │ │ (仪表盘)  │ │ (拖拽分析)│             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ Alerting │ │    ML    │ │   APM    │ ← 高级功能   │
│  │ (告警)    │ │ (机器学习)│ │ (性能监控)│             │
│  └──────────┘ └──────────┘ └──────────┘             │
│                                                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │  Spaces  │ │  Fleet   │ │ Security │ ← 管理       │
│  │ (多租户)  │ │ (采集管理)│ │  (SIEM)  │             │
│  └──────────┘ └──────────┘ └──────────┘             │
└──────────────────────────────────────────────────────┘
```

> **设计考量**：Kibana 为什么把配置也存进 ES 而不是自带的 SQLite/文件？换来的是配置数据天然获得 ES 的备份（snapshot）、权限（security）与复制（CCR）能力，Saved Object 的导入导出本质就是一次 ES 文档搬运；代价是 Kibana 与 ES 版本强耦合——Kibana 版本必须与 ES 主版本一致，升级时 Saved Objects 要做 schema 迁移。

# 2. 核心能力与功能地图

Kibana 的能力可以按**数据可视化、管理、告警、安全**四大块来记忆，这也是官方 Main Menu 的分组逻辑。逐块拆解如下。

## 2.1 数据可视化块

数据探索与图表呈现是 Kibana 的立身之本：**Discover** 面向"明细行"，逐条查看原始文档，是日志排障的主战场；**Lens** 面向"聚合分析"，拖拽字段即得图表，是 8.x 可视化构建的绝对主力；**Dashboard** 把多个图表与过滤器组织成监控大屏；**Canvas** 与 **Maps** 分别覆盖演示级报表和地理可视化。四者的分工与适用场景在后续章节（4-9 篇笔记）分别展开。

## 2.2 管理块

Stack Management 是"用 Kibana 管理 ES 集群"的入口集合：索引与别名管理（Index Management）、索引生命周期策略（ILM）、快照与恢复（SLM）、告警与连接器、用户与角色（Security）、多租户空间（Spaces）、以及 7.x 之后统一采集体系的 **Fleet**（管理 Elastic Agent 与各类 Integrations 的数据接入）。Java 后端团队最常接触的是其中的 Data View、ILM 与用户权限三块。

## 2.3 告警与自动化块

Kibana Alerting 采用 **Rule + Connector** 模型：Rule 定义"查什么、多久查一次、什么条件触发"，Connector 定义"触发后通知到哪里"（Webhook、邮件、Slack、PagerDuty、索引写入等）。它与传统 Watcher 并行存在——Watcher 是运行在 ES 侧、用 Watch DSL 描述的重量级方案，Kibana Rules 则是 UI 化、统一的轻量方案，8.x 新项目一律推荐后者。

## 2.4 安全与可观测块

Security 应用提供 SIEM 能力（检测规则、告警调查、Cases 工单、Timeline 时间线），ML 应用提供异常检测与数据可视化分析，APM/Infrastructure/Uptime 则合并为 Observability 统一面板。这两块与 Elasticsearch 内的数据（审计日志、endpoint 事件、APM span）深度绑定，属于"Kibana 把 ES 从搜索引擎变成分析平台"的增量价值。

| 能力 | 说明 | 工具 |
|------|------|------|
| **数据探索** | 交互式搜索和过滤 ES 数据 | Discover |
| **可视化** | 图表展示（柱状/折线/饼图/地图） | Lens / Visualize |
| **仪表盘** | 多面板组合的实时监控大屏 | Dashboard |
| **告警** | 基于查询结果的规则告警 | Alerting |
| **机器学习** | 异常检测、分类、回归 | Machine Learning |
| **安全分析** | SIEM 安全信息事件管理 | Security |
| **APM** | 应用性能监控 UI | APM / Observability |
| **管理** | 索引管理、ILM、权限控制 | Stack Management |

# 3. 版本与 License 层级

Kibana 的功能与订阅层级强相关，选型时必须先确认 License。8.x 的默认发行版是 **Basic（免费）**：Discover、Dashboard、Lens、Dev Tools、Fleet、APM UI、Synthetics 等核心能力全部免费可用；**Gold** 主要增加 SSO（SAML/OIDC/LDAP）、自定义角色与 Watcher；**Platinum** 解锁 Machine Learning、字段/文档级权限、CCR 跨集群复制；**Enterprise** 增加 Endpoint 防护（Elastic Defend）、SLO、高级 AI 能力。对照如下：

| 功能 | Basic | Gold | Platinum | Enterprise |
|------|:-----:|:----:|:--------:|:----------:|
| Discover/Dashboard/Lens | ✓ | ✓ | ✓ | ✓ |
| Fleet 与 Elastic Agent 接入 | ✓ | ✓ | ✓ | ✓ |
| Kibana Alerting（常用规则/连接器） | ✓ | ✓ | ✓ | ✓ |
| Watcher（ES 侧告警 DSL） |  | ✓ | ✓ | ✓ |
| SSO（SAML/OIDC/LDAP）、自定义角色 |  | ✓ | ✓ | ✓ |
| Machine Learning 异常检测 |  |  | ✓ | ✓ |
| 字段级/文档级权限（DLS/FLS） |  |  | ✓ | ✓ |
| CCR 跨集群复制 |  |  | ✓ | ✓ |
| Elastic Defend 端点安全 |  |  |  | ✓ |
| SLO、AI Assistant 等高级能力 |  |  |  | ✓ |

许可证层面还有两个时间点要记住：自 7.14 起 Elasticsearch 与 Kibana 源码以 **SSPL + Elastic License 双许可**发布（不是 OSI 意义的开源）；自 8.16 起又追加了 **AGPLv3** 选项，重新回到"严格意义上的开源许可证"。

> **注意**：上表只列高频差异项，且 Elastic 各小版本偶有调整（历史上 Spaces、部分 Security 能力曾从高阶层级下放为免费）。立项前务必以官网 elastic.co/subscriptions 的对照页为准，尤其要确认你们用到的 ML、DLS/FLS、SSO 属于哪一层级，避免上线后才发现 License 覆盖不了。

# 4. Kibana vs Grafana

| 维度 | Kibana | Grafana |
|------|--------|---------|
| **数据源** | 仅 Elasticsearch | ES/Prometheus/MySQL/InfluxDB 等 |
| **可视化** | Lens 拖拽、Canvas 大屏 | 面板丰富 |
| **告警** | 内置（ES Query/Index Threshold） | 内置（多数据源） |
| **学习曲线** | 低（与 ES 深度绑定） | 中 |
| **ML/APM/SIEM** | 内置 | 需额外插件 |
| **开源** | ✓（Elastic License） | ✓（AGPL） |

**结论**：仅使用 ES → Kibana；多数据源 → Grafana；两者可互补（Grafana 连 ES，Kibana 做 ML/APM）。展开一层判断依据：Kibana 的核心优势在于它是"懂 ES 语义"的前端——字段类型推断、时间字段联动、KQL、ILM/快照管理都只有原生 Kibana 能做，日志检索体验也明显优于 Grafana 的 Explore 视图；Grafana 的优势则是 datasource 插件生态和轻量的时序看板。团队如果指标在 Prometheus、日志在 ES，常见落地是"指标看板给 Grafana，日志检索与排障留给 Kibana"，两边各放一个 ES datasource 做交叉跳转（exemplar/derived field 链接）。

> **行业实践**：以 ES 为唯一存储栈的中小团队直接全家 Kibana；已有 Prometheus/Loki/Tempo 体系的云原生团队，指标大盘用 Grafana、日志与 APM 深度分析用 Kibana 是务实的折中，关键是把两边的跳转链接打通，而不是强行统一到一套工具。

# 5. Kibana 8.x 新特性

| 特性 | 说明 |
|------|------|
| **Lens** | 拖拽式可视化，替代原 Visualize |
| **Dashboard 增强** | Controls 面板过滤器、Drilldown 下钻 |
| **Alerting 框架** | 统一 Rule + Connector 模型 |
| **Fleet** | 统一采集管理（Elastic Agent） |
| **Security SIEM** | 检测规则、Timeline、Cases |
| **Observability** | APM + Infrastructure + Uptime 统一面板 |

对运维影响最大的隐性变化是：**8.0 起 Security 与 TLS 默认开启**，Kibana 不能再像 7.x 那样裸 HTTP 直连无认证的 ES，首次启动就要处理 enrollment token 或 kibana_system 账号（详见 3_installation 篇）。此外 8.11 引入了查询语言 **ES|QL**（管道式分析语言，可在 Console 与后续版本的数据视图中使用），8.7+ 的 Discover 支持多数据视图 Tab 对比检索，都代表了"从看板工具向分析平台"演进的方向。

# 6. 核心设计理念

1. **数据驱动**：所有功能围绕 Elasticsearch 数据索引展开。Kibana 没有自己的查询引擎，Discover 的过滤、Lens 的聚合最终都翻译成 ES Query DSL 交给 ES 执行；这也意味着图表性能问题的排查路径永远是"先看面板生成的 DSL 是否高效"（Inspect → Query）。
2. **Saved Objects**：一切配置（Dashboard/Visualize/Index Pattern）都是可导出的 Saved Object。设计收益是配置即数据：版本化、Git 化管理、跨环境迁移都只是导出 NDJSON；代价是配置迁移与 Kibana 大版本绑定，7.x 导出直接导入 8.x 需要字段兼容。
3. **Spaces 隔离**：多租户空间隔离，不同团队独立工作。Space 同时提供资源隔离（仪表盘、Data View 互不可见）与权限边界（按空间分配角色），比"一个大 Home 页全员混用"可维护得多。
4. **KQL 查询**：自研查询语言，降低 ES Query DSL 学习成本。KQL 的哲学是"覆盖 90% 检索场景的最小语法"，需要正则、模糊、嵌套布尔时再切 Lucene 语法兜底。
5. **插件化**：功能模块可按需开启（ML/APM/SIEM 等）。Kibana 内核（Core）本身只提供路由、Saved Objects、安全与通知等基础设施，上层全部应用皆为插件，`xpack.*.enabled: false` 即可关闭不需要的模块以缩减内存与攻击面。

# 7. 适用场景

```text
✓ 适用：
  - ES 数据的可视化探索和搜索
  - 实时业务监控大屏
  - 日志集中检索分析
  - APM 应用性能监控
  - 安全事件分析（SIEM）
  - 多团队数据隔离（Spaces）

✗ 不适用：
  - 需要连接多个异构数据源（用 Grafana）
  - 独立的 BI 报表（用 Metabase/Superset）
```

两个"不适用"值得补充判断细节：BI 场景缺的不是图表而是**语义层**——指标口径定义、行级权限、自助报表治理，这些是 Metabase/Superset 的主场，Kibana 的 Lens 公式只解决临时口径；多数据源场景则是 Kibana 架构层面的限制——它的整个字段模型建立在 ES mapping 之上，没有为外部数据源设计通用抽象，不要试图用 ES 做联邦查询再上 Kibana。

> **注意**：Kibana 与 ES 的版本必须**主版本一致**且 ES 不低于 Kibana 补丁版本，混用大版本（如 8.11 Kibana + 7.17 ES）官方不支持，会出现 Saved Objects 迁移失败或 API 不兼容，升级时永远"先 ES 后 Kibana"。

# 8. 小结

Kibana 在 Elastic Stack 中承担"统一操作入口"角色：无状态、不存业务数据、配置全部以 Saved Objects 形式落在 ES 的 `.kibana` 索引。功能地图按数据可视化（Discover/Lens/Dashboard/Canvas/Maps）、管理（Stack Management/Fleet/Spaces）、告警（Rule+Connector/Watcher）、安全与可观测（SIEM/ML/APM）四大块理解。选型三条硬判断：数据全在 ES 选 Kibana，多源看板选 Grafana，深度 BI 报表别勉强 Kibana；License 层级决定 ML、DLS/FLS、SSO 是否可用，务必对照官方订阅页；8.x 默认安全开启与版本强耦合是落地前必须规划的两个工程前提。
