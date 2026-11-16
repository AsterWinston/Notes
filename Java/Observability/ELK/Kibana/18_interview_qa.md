# 1. 基础题

## 1.1 Kibana 是什么？在 ELK 中的角色？

Kibana 是 Elastic Stack 的可视化和管理工具，本质是一个**无状态的 Node.js 应用**：它对 ES 做数据面代理（Discover/看板查询全部经它转发，浏览器不直连 ES），同时承载整个平台的管理面。在 ELK 中它负责四件事——**可视化**（Dashboard、Lens 图表）、**搜索**（Discover 交互式数据探索）、**管理**（Index Management、ILM、权限、Fleet）、**高级功能**（ML、APM、SIEM）。所有配置类资产（看板、搜索、Data View、规则）都以 Saved Objects 形式持久化在 ES 的 `.kibana` 索引里，所以 Kibana 自身没有需要备份的本地状态，水平扩展只需多起实例挂负载均衡。

## 1.2 Kibana 和 Grafana 的区别？

| 维度 | Kibana | Grafana |
|------|--------|---------|
| 数据源 | 仅 ES | 多数据源（ES/Prometheus/MySQL等） |
| ML/APM/SIEM | 内置 | 需插件 |
| 社区 | Elastic | 独立开源 |

一句话结论：**纯 ES 环境用 Kibana，多数据源场景用 Grafana，两者可互补**。面试加分点是讲出互补的常见形态——指标侧 Prometheus + Grafana，日志/trace 侧 ES + Kibana，看板间用链接跳转；但当组织把日志、指标、trace 都收敛进 ES（ECS + data stream），Kibana 的跨信号关联（trace 跳日志、指标拆维度）是 Grafana 给不了的一体化体验。

## 1.3 Data View 是什么？

Data View（原 Index Pattern）定义 Kibana 如何识别 ES 索引中的字段，是 Discover 和 Dashboard 的数据源基础。通过通配符 `orders-*` 匹配多个索引，支持 Runtime Field 动态计算字段。要补三点深度：其一，它不是视图也不存数据，只是一份 **pattern + 时间字段 + 字段元数据的 Saved Object**，字段列表定期从 mapping 刷新，新字段"消失"通常是没点 Refresh；其二，**time field 决定了 KQL 能否用时间范围、看板能否按时间聚合**，日志类必须设、维表类留空；其三，Runtime Field 在查询时计算、灵活但拖慢检索，稳定口径应下沉到 ingest pipeline 变成实体字段。

## 1.4 Lens 是什么？相比老的 Visualize 好在哪？

Lens 是 8.x 默认的**拖拽式可视化构建器**：把字段拖到图表区，自动推断合适的聚合与图表类型，切换折线/柱状/饼图/指标卡不用重写配置。相比老 Visualize 的优势：**数据模型层复用（一个 Lens 可混合多个 Data View 的指标做对比图）、表达式驱动（底层是 ES|QL/公式栏，能写 `count() / time() * 60` 这类派生指标）、自动处理桶与排序**。定位上记一句就够：Lens 管图表，Dashboard 管编排，Canvas 管像素级排版汇报页，Maps 管地理——面试里被问"8.x 还该不该学老的 Vega/heatmap 类型"，答案是常规需求全用 Lens，Vega 只留给定制交互。

# 2. 原理题

## 2.1 Saved Objects 是什么？存储在哪里？

Saved Objects 是 Kibana 所有配置的统一持久化模型（Dashboard/Visualization/Data View 等），存储在 ES 的 `.kibana` 系统索引中，8.x 实际索引名带版本号（如 `.kibana_8.11.0_001`）。支持导出/导入、跨 Space 复制、版本迁移。原理层补三点：每个对象有 `type/id/attributes/references` 四要素，**引用关系（references）显式建模**——Dashboard 引用哪些 panels 和 search，这决定了删除检查与"Include related objects"导出的行为；Space 隔离靠 `namespaces` 字段，同一对象 ID 可在不同 Space 独立存在；升级时 Task Manager 跑迁移脚本（migrations）把旧结构升上来，所以大版本升级后第一次启动最慢、迁移完旧索引转只读。

## 2.2 KQL 和 Lucene 语法的区别？

| 维度 | KQL | Lucene |
|------|-----|--------|
| 设计 | 现代、简洁 | 传统、强大 |
| 范围 | `amount > 100` | `amount:{100 TO *}` |
| 正则 | ✗ | ✓ |
| 推荐 | 新用户默认 | 高级用户 |

Kibana 默认使用 KQL，可在搜索栏切换至 Lucene。展开讲差异根源：**KQL 是为"日志检索"设计的简洁语法，`field:value` 精确匹配（需 keyword 类型）、天然 AND、显式 OR/NOT**，不会因字段默认打全text分词误伤；**Lucene 直通 ES query_string，支持正则、`~` 模糊、`^` 加权、范围、通配符**，代价是默认 OR 语义导致查询"过宽"的经典陷阱（`foo bar` 在 Lucene 下是命中其一即可）。实战建议：团队统一 KQL，复杂狩猎临时切 Lucene；注意 KQL 的 `field:value` 对 text 字段只命中**完整词项**，模糊匹配要靠 `~`（编辑距离）或换 Lucene 正则。

## 2.3 Kibana 的 Spaces 隔离原理？

每个 Space 在 `.kibana` 索引中以 `space:{id}`（namespaces 字段）区分存储，不同 Space 之间的 Saved Objects 完全隔离。通过 RBAC 控制用户对 Space 的访问权限。必须讲清边界：**Space 隔离的是"配置与功能入口"（application privilege + disabledFeatures），不隔离数据本身**——数据可见性由 Role 的 index privileges（含 DLS/FLS）决定；只建 Space 不配角色，A 团队的用户照样能在 Discover 里查 B 团队索引，这是最常见的面试陷阱与生产事故。

## 2.4 Kibana 架构上如何与 ES 交互？鉴权怎么传递？

浏览器请求先到 Kibana（默认 5601），Kibana 用自己的服务账号 `kibana_system` 连接 ES 完成**基础设施操作**（读写 .kibana、任务索引），但**数据查询按登录用户身份透传**——ES 侧对最终用户做 authorization（Run As 机制），所以 Dev Tools 里执行 `DELETE /logs-*` 实际消耗的是当前用户的索引权限。这带来两个结论：多实例 Kibana 天然可水平扩展（无本地状态，LB 轮询即可，Session 存于 server-side storage）；后台任务（告警、报表、Watch 触发）由 **Task Manager** 基于 `.kibana_task_manager` 索引做分布式租约调度，多实例互不重复执行。

## 2.5 Alerting 和 Watcher 的区别？该用哪个？

Watcher 是 **ES 端**的老告警机制：watch 定义在 ES 里、由 master 节点上的 Watcher 服务执行，输入/条件/动作全用脚本表达，灵活但只有 JSON 一处维护、无 UI 工作流、Platinum 起步。Kibana Alerting 是 **8.x 推荐框架**：Rule（求值）→ Alert（状态机，含 Ack/Recovered/Closed）→ Connector（统一管理的投递配置、加密凭据），由 Task Manager 在多 Kibana 实例间分布式调度，UI/API 全接管。选型口径：**新告警一律 Alerting，Watcher 只作存量兼容或深度脚本定制**；两者并存时最容易出现同一条件双重告警，迁移期要把 Alerting 事件写回索引与 watch history 对账再逐个下线。

# 3. 实践题

## 3.1 如何创建一个监控订单量的 Dashboard？

标准流程六步：1）Stack Management 创建 Data View `orders-*`，指定 time field `created_at`；2）Lens 建 QPS 折线图——X 轴拖 `@timestamp`（histogram interval），Y 轴 `Count of records`；3）Lens 建指标卡——拖 `SUM(amount)` 并配同比对比；4）Lens 建饼图/横向柱——维度 `status`（keyword 类型字段直接用，别拖 .keyword 到 text 上）；5）Dashboard 依次 add panels、排布并加 Markdown 说明口径与 owner；6）Controls 添加 `category` 下拉过滤、设默认时间窗 Last 24h。加分回答是治理视角：面板 ≤15、命名规范、评审归档、慢面板用 Inspect 看 DSL 回 ES 侧优化。

## 3.2 如何设置错误日志告警？

1）Stack Management → Rules → Create rule → **Elasticsearch query** 类型；2）索引选 `logs-*`，Query 写 `level:ERROR`；3）Threshold：count > 10（窗口 last 5 min）；4）Check every 1 min（窗口与频率独立，注意恢复延迟=窗口长度）；5）预先建好 Slack Webhook Connector 并在规则里引用，消息模板带 `{{context.value}}` 与 `{{context.link}}` 深链；6）配 Throttle 防重复轰炸，开启恢复通知。加分点：区分维度时改用 Index threshold + Group by `service.name`；阈值参考历史 p95/同比而不是拍脑袋；规则先影子运行两周观察误报率再进值班通道。

## 3.3 如何给不同团队隔离数据和 Dashboard？

1）创建 Spaces：Team-A、Team-B；2）各 Space 建独立 Data View 指向各自索引；3）RBAC 创建角色（如 `team_a_viewer`）：**三层权限都要配**——cluster 给 `monitor`、indices 限 `order-*,product-*` 的 read、application 在 `space:team-a` 资源上给 `kibana_all_read`/feature read；4）用户或 AD 组通过 role mapping 绑角色；5）各团队在自己 Space 建看板。关键补一句：只配 application privilege 不配 index privilege 会 403，只配 index 不配 application 会功能不可见；跨租户强隔离（同一索引按行过滤）要上 DLS/FLS，Space 本身不是数据边界。

## 3.4 License 分层怎么影响功能选型？

| 层级 | 代表功能 |
|------|---------|
| Basic（免费） | 核心 Kibana、Alerting 基础规则、Fleet/Agent 采集、看板/报表（部分） |
| Gold | 告警增强等少量功能（基本被 Platinum 覆盖，价值有限） |
| Platinum | Watcher、ML 异常检测、DLS/FLS、SIEM 检测引擎、Remote Cluster |
| Enterprise | SLO 功能、Elastic Security Endpoint/风险评分、机器学习推理（部分）、文件级恢复 |

选型逻辑：日志检索+看板+基础告警 Basic 就够；要 **ML 异常检测、Watcher、字段/文档级权限、SIEM 任一项，直接评估 Platinum**（30 天 trial 自签即可开启）；SLO、端点安全、UEBA 到 Enterprise。面试常见追问"trial 到期怎么办"——功能对象进入只读/禁用而非删除，换正式 license 即恢复，但**依赖 ML 规则的告警会停跑**，生产必须提前续期。

# 4. 架构题

## 4.1 Kibana 如何处理高并发？

无状态设计是根基：**Kibana 多实例水平扩展（LB 分发）**，配置全部在 `.kibana`、会话存服务端会话索引，任何实例可服务任何用户。高并发的真正瓶颈在 ES 侧而非 Kibana——所以调优抓手有三个：**ES 连接池**（`elasticsearch.maxSockets` 控制并发上限，配合 requestTimeout/shardTimeout 快速失败，绝不靠调大超时掩盖慢查询）；**Task Manager** 把告警/报表等后台负载与在线请求分离、多实例分摊；看板侧控制并发查询数（面板数 ≤15、避免自动刷新过密），高频访问的聚合结果考虑用 transform/rollup 预计算，把"每次现算"变成"读现成"。

## 4.2 Fleet 和传统 Beat 有什么区别？

Fleet 是 Kibana 的统一采集管理平台，通过 Elastic Agent 一站式管理所有数据采集；传统 Beat 需要手动在每台机器上部署和配置多个独立的 Beat（Filebeat/Metricbeat 等）。Fleet 提供 UI 集中管理、一键升级、策略下发。架构上多说三层：**Agent（采集）→ Fleet Server（策略下发/数据中转/ES 凭据收敛）→ ES**，通信全走 HTTPS 出网方向，注册靠 enrollment token（可复用 token 支撑 K8s/弹性扩缩容自注册）。口径记两句：**新场景一律 Agent，存量 Beat 不强制迁移、长期共存**；managed 模式下机器本地配置会被 policy 覆盖，"真相在 Kibana 不在机器"。

## 4.3 APM 的数据模型是什么？trace 存在哪？

APM 以 **transaction（服务入口工作单元）+ span（内部子调用）+ error** 构成 trace 模型，语义上与 OpenTelemetry 完全兼容（APM Server 可直接收 OTLP）。存储上 **trace 不是实体**：8.x intake v2 协议把数据写入 `traces-apm*`、`metrics-apm*`、`logs-apm*` 等 data stream，靠 `trace.id` 字段查询时聚合还原链路，所以深挖单条 trace 就是按 trace.id 查文档。Service Map 由 span 里的 destination 元数据自动推断，裸 Socket 调用不会被发现——**图上缺的依赖等于采集没覆盖的调用**，这也是审计 agent 覆盖率的方法。

# 5. 排障题

## 5.1 Kibana 看板加载慢甚至报 400/超时，怎么排查？

按链路顺序切四刀：1）先分清**平台慢还是查询慢**——`GET /api/status` 与 Kibana 日志看有无 ES 连接/线程池报错，同一查询在 Dev Tools 直发对比 took；2）打开面板 **Inspect → Request**，拿到真实 DSL，看范围：时间窗是否过大、是否全模糊 `*kw*`、terms 聚合是否打在 `user_id` 这类高基数字段上（协调节点内存与 doc_count_error 双高）、深分页 search_after 是否被滥用；3）ES 侧体检：慢日志（`GET _nodes/:node/name/...settings` 打开 search slowlog）定位具体查询，检查索引时间字段裁剪是否生效、分片是否过多过小、老索引有无 force merge/降冷；4）结论口径：**Kibana 的超时参数只是掩盖问题，慢的根源九成在查询写法与数据模型**，能加时间过滤加时间过滤、能预计算用 transform、高基数聚合改 composite 或派生低基数字段。

## 5.2 告警没有触发/重复触发，怎么定位？

沉默的告警系统比误报更危险，排查分层：1）**规则层**——Stack Management 看规则是否 Active/报错（规则执行失败本身进 Warning 状态，要检查 `.kibana-alert_logs*` 与规则 Metrics 中的失败计数），确认 check interval 与时间窗设置（窗口=1 分钟但每小时查一次，就会漏）；2）**Task Manager 层**——`.kibana_task_manager` 任务积压或多实例时钟漂移会造成延迟触发；3）**触发逻辑层**——重复触发多半是 Throttle 未配、或 Groups by 维度使每分组各自告警（属正常设计），而"该报没报"常见于查询本身写错（KQL 对 text 字段 `level:ERROR` 大小写分词陷阱）；4）**投递层**——Connector 单独测试（UI 里的 test 按钮/Server Log connector），凭据错、出网被拦、`kbn-xsrf` 缺失都在这层。维护窗口的 Snooze 状态忘了取消也是经典原因，先查 `_find?tags` 与 mute 状态再翻日志。

## 5.3 Discover 里字段/新数据看不到，可能是什么原因？

最高频原因是 **Data View 字段缓存过期**——新加的列来自 mapping 刷新，Stack Management → Data Views → Refresh field list 即恢复；其余按链路：采集侧（Fleet Agent offline、pipeline `_simulate` 解析失败进死信队列）、索引侧（写入成功但索引 pattern 不匹配 Data View、日期映射成秒级时间戳导致时间过滤全落窗外——`index.mapping.ignore_malformed` 与时区配置是重灾区）、权限侧（角色 index privilege 不含该索引时字段列表直接为空而非报错）、以及 refresh_interval 调大后近实时可见性延迟。排查动线建议自下而上：先确认 ES 里有文档（`GET idx/_count`），再看 mapping 与时间字段，最后回 Kibana 刷新字段列表。

# 6. 小结

本篇覆盖 Kibana 面试的四类问题：**概念题**（角色定位、Data View/Lens/Saved Objects、与 Grafana 边界）、**原理题**（.kibana 存储与 references、Spaces 隔离边界、KQL/Lucene 语义差异、鉴权透传与 Task Manager、Alerting vs Watcher）、**实践题**（看板六步、告警六步、三层 RBAC、License 分层选型）、**架构与排障题**（无状态扩展、Fleet 三层架构、APM 数据模型、看板慢/告警失灵/字段消失三条完整排查链）。答题的通用套路是**先给一句话定位，再补机制边界与坑**——"Space 不隔离数据""超时参数只是掩盖慢查询""图上缺的依赖就是没采到的调用"这类判断句，才是区分用过与理解的信号。
