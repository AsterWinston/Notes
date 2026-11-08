# 1. 概述

Dashboard 是 Kibana 的**可视化组合面板**，将多个图表、表格、指标组合成一个实时监控视图。它是 Kibana 里最接近"产品"的形态：工程师在 Discover 里自助探索，而值班、业务、管理层看的是 Dashboard——一个 URL 打开一屏答案。

架构上 Dashboard 本身只是一个"布局清单"：Saved Object 里存的是面板列表（每个面板引用一个 Lens 图表/Saved Search/Markdown/控件）、每个面板的网格坐标与尺寸、以及 Dashboard 级的 query/filters/时间设置。打开 Dashboard 时，各面板**并行**向 ES 发起自己的查询，全部返回后渲染完成。理解"面板即查询"这个模型，就理解了 Dashboard 性能、联动、导出的一切行为。

```text
┌──────────────────────────────────────────────────────┐
│  Dashboard: Order Monitoring                         │
│  ┌───────────────────────┐ ┌──────────────────────┐  │
│  │  Orders QPS (折线图)   │ │ Revenue (指标)       │  │
│  │     ╱╲   ╱╲    ╱╲     │ │    ¥1,253,680       │  │
│  │    ╱  ╲ ╱  ╲  ╱  ╲    │ │         ▲ 12%        │  │
│  │   ╱    ╳    ╳    ╳    │ │                      │  │
│  │  ╱     ╲    ╲    ╲   │ └──────────────────────┘  │
│  └───────────────────────┘                           │
│  ┌───────────────────────┐ ┌──────────────────────┐  │
│  │ Status Pie (饼图)      │ │ Top Products (表格)   │  │
│  │  60% active            │ │ iPhone   ¥500,000    │  │
│  │  25% pending           │ │ MacBook  ¥300,000    │  │
│  │  15% cancelled         │ │ iPad     ¥200,000    │  │
│  └───────────────────────┘ └──────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

面板类型四种：Lens 可视化（主力）、Saved Search（Discover 保存的搜索，可显示命中数）、Controls（全局过滤控件）、Markdown（文字说明/公告/链接）。一个成熟大屏通常是"顶部 Controls + 指标行 + 趋势区 + 明细区"的纵向结构。

# 2. 创建 Dashboard

```text
方式 1: 从 Discover/Lens 直接保存到 Dashboard
  Discover → 搜索 → Save → "Add to new Dashboard"

方式 2: 新建空 Dashboard
  Dashboard → Create dashboard → Add panel
    → 从 Visualize Library 选择已有图表
    → 或创建新的 Lens 图表
```

两种方式对应两种工作流：**自下而上**（先有分析结论，顺手沉淀成面板，探索期常用）与**自上而下**（先设计信息布局再填充图表，正式监控大屏建议如此，避免"图表垃圾场"）。面板引用的是 Saved Object，同一 Lens 图表可被多个 Dashboard 引用——改一处、处处生效，这是"口径统一"的技术保证；反之把图表"复制"进 Dashboard（embed as new）则会断掉引用关系，适合需要各屏差异化时。

# 3. 面板操作与布局

```text
面板操作按钮 (面板右上角 ...):
  ├── Edit visualization     → 编辑 Lens 图表
  ├── Inspect               → 查看数据/请求/响应
  ├── Maximize              → 全屏展开
  ├── Copy to dashboard     → 复制到其他 Dashboard
  ├── Replace panel         → 替换为其他图表
  └── Remove from dashboard → 删除面板

拖拽: 面板右下角 → 调整大小
拖拽: 面板标题栏 → 移动位置
```

两个高频且实用的操作：**Inspect**（面板级）是排障利器——Data 页看渲染用的中间表、Request 页看实际 DSL 与 `took`、Response 页看原始聚合结果，"哪个面板拖慢了整屏"用各面板的 `took` 一比即知；**Edit visualization** 是"就地改图"入口，但注意进入的编辑页默认是编辑**被引用的源图表**，改动会同步到所有引用它的 Dashboard，需要独立版本时先在 Lens 里 Save a copy。编辑模式下所有面板可拖拽、缩放，栅格对齐，改完必须 Save Dashboard 才落盘——未保存时刷新页面布局丢失是新人常见困惑。

# 4. 全局过滤器与时间联动

Dashboard 的 query、filters、时间范围是**全局属性**，所有面板自动继承：时间选择器改动 → 全部面板重查询；加一个 filter 芯片 → 全部面板 AND 上该条件；顶部 query bar 输入 KQL 同理。这带来"联动"效果——点饼图某个扇区生成的 filter 会同时收窄折线图与表格，这是 Dashboard 区别于"一堆孤立截图"的核心价值。

8.5+ 引入的 **Controls 面板**把过滤能力显性化到界面上：

```text
Dashboard → Create panel → Controls

Options list (下拉选择):
  Field: status.keyword
  Label: Order Status
  效果: Dashboard 顶部出现下拉框 → 选择值 → 所有面板联动过滤

Range slider (范围滑块):
  Field: amount
  Step: 10
  效果: 滑块选择金额范围 → 所有面板联动

Time slider (时间滑块):
  Field: @timestamp
  效果: 时间轴拖拽 → 所有面板时间联动
```

控件生成的条件本质仍是 Dashboard 级 filter，只是从"编辑者预设"变成"查看者自选"。给业务方自助大屏时，用 Options list 控件替代让对方学 KQL，是使用率的分水岭。

> **注意**：filter 芯片对多数据视图/多 pattern 面板可能"部分生效"——某面板的索引里不存在该字段时，条件对该面板无意义（不会报错也不会过滤）。跨视图大屏的联动过滤要尽量落在**所有索引都有的公共字段**（ECS 规范字段）上。

# 5. Drilldown 下钻链接

Drilldown 让 Dashboard 之间"连成网"而非一座座孤岛：在 Lens 图表编辑器里为字段值配置点击行为（8.x 中 Lens 与 Dashboard 设置里均可管理），支持四种目标——跳到另一 Dashboard、跳到 Discover、跳到 Saved Search、打开外部 URL，且可携带当前上下文（filters、query、time range 以变量形式拼接进目标链接）：

```text
设置 Drilldown:
  Lens 编辑 → 保存 → Drilldown → Create
    Type: Go to URL / Go to Dashboard
    传递当前上下文（filters, time range）

使用 Drilldown:
  点击图表中的数据点 → 跳转到目标 Dashboard
  例如: 点击 "cancelled" 饼图块 → 跳转到 "Cancelled Orders" Dashboard
```

典型的信息层级设计是**三层漏斗**：总览大屏（看异常）→ 域下钻屏（看细分）→ Discover/明细（看单条日志）。Drilldown 把这三层串起来后，值班 SOP 可以写成"从 A 屏点红色扇区进 B 屏，再点 trace 进 Discover"，比培训文档有效得多。外部 URL 形态则用于打通 Kibana 之外的系统（工单、CMDB、Grafana）。

> **行业实践**：大屏之间的层级一旦超过 3 层、或出现横向互链（A↔B↔C 网状），用户就会迷路。维护"一屏一职责"的树状结构，配合面包屑与命名规范（见 7 节），比堆功能更重要。

# 6. 分享、嵌入与管理

```text
Dashboard 右上角:
  └── Share → 分享链接 / CSV 导出 / 嵌入 iframe

Dashboard → Edit:
  └── Controls 设置（全局过滤器）

Stack Management → Saved Objects → Dashboard:
  ├── Export → 导出为 NDJSON
  ├── Import → 导入 Dashboard
  └── Copy → 复制到其他 Space
```

Share 里的**永久链接（Permanent link）**把当前 filters/时间序列化进 URL，适合"把这个视图发给同事"；**Short link** 是一次性跳转，适合工单里引用；**Embed** 生成 iframe 代码嵌入内部门户——注意 8.x 嵌入默认仍依赖浏览器登录态，纯 iframe 免登录要用 ES 侧生成的带权限 API Key 方案或反向代理鉴权，不要为了嵌入直接对公网开匿名（8.0 已彻底移除匿名白名单式 embed 的老路）。Reporting 还支持把 Dashboard 定期导出 PDF 邮件分发，适合日报周报场景。

# 7. 导出与导入（跨环境迁移）

Dashboard 依赖 Lens 图表、Saved Search、Data View，迁移必须带上整棵引用树——`includeReferencesDeep` 就是干这件事的：

```bash
# 导出 Dashboard（含依赖的 Lens/Search/Data View）
POST /api/saved_objects/_export
{
  "objects": [{ "type": "dashboard", "id": "my-dashboard" }],
  "includeReferencesDeep": true
}
# 响应为 NDJSON 文件流，包含所有引用对象

# 导入
POST /api/saved_objects/_import?overwrite=true
# 上传 NDJSON 文件（multipart/form-data，字段名 file）
```

导入导出的实操要点：UI 路径（Stack Management → Saved Objects）底层就是这套 API，NDJSON 可用文本编辑器打开检查；跨大版本迁移（7.x→8.x）不能直接导，需先用升级工具或导出时选新版格式，`overwrite=true` 谨慎用（会覆盖目标环境同名对象）；`excludes: ["search","query"]` 参数可只导布局不带当前过滤状态。CI/CD 化管理大屏时，把导出文件放进 Git、按环境跑 `_import`，是 Kibana 配置 IaC 化的最低成本起点。

# 8. 性能与容量控制

Dashboard 卡顿的根因几乎总是"面板数 × 单面板查询代价 × 时间窗口"三者的乘积。逐项控制：

| 问题 | 机制 | 对策 |
|------|------|------|
| 整屏加载慢 | 面板并行发起聚合，ES 协调节点汇总排队 | 面板数 ≤ 15；拆分大屏 |
| 单面板转圈久 | 高基数 terms/date_histogram 聚合代价大 | Inspect 看 took，缩窗口/降桶数 |
| 刷新风暴 | auto-refresh 高频 × 面板数 | 刷新间隔 ≥ 30s 或手动刷新 |
| 深分页/大时间窗 | 扫描文档量线性上涨 | 默认 Last 15m/24h，复盘用绝对时间 |
| 指标面板抖动 | count 类面板每刷新都全量聚合 | 用较粗 interval 或预聚合（Transform） |

面板的 date_histogram 桶数（时间跨度 ÷ interval）是聚合开销的主变量——"7 天窗口 + 1 分钟 interval"是 1 万个桶，必慢；Kibana 默认 Auto 分桶一般合理，手动调 interval 时别调太细。高频刷新的大屏（作战室/大屏电视）建议固定 Absolute 时间 + 定时 reload，或干脆把实时性要求降到 30s 以上。

> **注意**：Inspect 显示的 `took` 只是 ES 执行时间，不含 Kibana 结果整形与浏览器渲染。"took 50ms 但面板转 3 秒"往往是返回体积太大（几万个桶/几万个 terms）——减小桶数与基数永远比调超时配置更有效。

# 9. 大屏组织实践

1. **面板分组**：使用 Controls 面板统一过滤，而非每个面板单独加 filter
2. **命名规范**：Dashboard 命名 `[Team]-[Purpose]-[Env]`，如 `Order-Monitoring-Prod`
3. **时间范围**：Dashboard 设置合理的时间默认值（Last 15m / Last 24h）
4. **Markdown 面板**：用 Markdown 面板添加说明文字、链接、公告
5. **减少面板数**：一个 Dashboard 不超过 10-15 个面板（加载性能）
6. **避免实时刷新过频**：生产 Dashboard 刷新间隔 ≥ 30s

补充组织层面的经验：每个大屏放一个 Markdown "说明书"面板（指标口径、负责人、数据来源），半年后接手的人能省一小时提问；用 **Tags** 与 **Spaces** 管理大屏归属，避免 Saved Objects 列表变成垃圾场；"监控大屏 + 下钻大屏 + 复盘模板"三类分开建设，不要一个屏承担所有角色；大屏改版用导出 NDJSON 进 Git 做版本对比，改坏了可回滚。

# 10. 小结

Dashboard = 布局清单 + 并行查询：每个面板引用一个 Saved Object（Lens/Search/Control/Markdown），共享全局 query、filters 与时间范围，联动由此而来。日常操作记四件事：Controls 面板做查看者可选的联动过滤、Drilldown 串起"总览→下钻→明细"三层漏斗、Inspect 定位慢面板（看 took 与返回体积）、`_export` 带 `includeReferencesDeep` 做跨环境迁移。性能铁律：面板 ≤ 15、默认窗口 ≤ 24h、刷新 ≥ 30s、桶数节制。组织上"一屏一职责 + 命名规范 + Git 化导出"，大屏才能从截图墙变成可持续维护的监控资产。
