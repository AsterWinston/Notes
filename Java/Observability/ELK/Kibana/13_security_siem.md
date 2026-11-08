# 1. SIEM 概述

Kibana Security（产品名 **Elastic Security**）提供**安全信息事件管理（SIEM）**能力，包括威胁检测、调查和响应，它同时是 Elastic 的 **SIEM 与 Endpoint 安全（EDR）** 平台——前者靠对日志/网络流量/身份事件的检测规则，后者靠 Elastic Defend agent 采集进程、网络、文件行为。与传统 SIEM（Splunk ES、QRadar）相比，Elastic Security 的差异化在于**检测规则本身是 Saved Object，可与 Kibana Alerting、Cases、SOAR 式自动化无缝拼装**，而不是割裂的独立模块。

```text
Security 功能:
  ├── Dashboards        → 概览大盘（攻击概览/MITRE 覆盖）
  ├── Investigate       → Alerts / Timeline / Explore
  ├── Detection Rules   → 自动检测威胁
  ├── Hosts / Network   → 主机与网络分析
  ├── Cases             → 安全案例管理
  ├── Hypotheses        → EQL 威胁搜索
  └── Saved Queries     → 可复用查询片段
```

一切功能建立在 **ECS（Elastic Common Schema）** 之上：日志接入时先把字段映射到 ECS 标准字段（`source.ip`、`process.parent.name`、`user.name`、`event.category`），检测规则、威胁狩猎、MITRE 映射**全部按 ECS 字段编写**。这带来一条铁律：**数据接入不映射 ECS，Security 能力等于零**——原始 sysmon 日志不转换的话，内置规则一条都命中不了，这是 SIEM 落地项目失败率最高的一环。

> **注意**：Elastic Security 的 SIEM 特性需要 **Platinum License**（Endpoint 安全属 Enterprise）。Basic License 下只能当"带检测规则手动跑的 Discover"用，且**内置检测规则（prebuilt rules）的数量与更新频率随 License 层级受限**。试用评估时务必用 trial license，别用 basic 环境测完下结论。

# 2. Detection Rules

检测规则引擎是 Security 的心脏：规则以固定周期对 ECS 化数据跑查询，命中即生成 **Alert（带 MITRE 技术号、严重度、原始事件、实体列表）**。理解它的关键是分清**规则类型**与**执行方式**（Single 逐事件 / Threshold 按阈值聚合计数）。

## 2.1 规则类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **Query** | 基于 ES/KQL/Lucene 查询检测 | 10 分钟内 SSH 失败 > 20 次 |
| **EQL** | 按事件**序列/顺序**匹配（威胁狩猎语言） | 先执行 powershell → 后网络连接 |
| **Machine Learning** | ML Job 异常分触发 | 网络流量异常 |
| **Threshold** | 指标阈值 | 请求数 > 10000/s |
| **Event Correlation** | 多规则/多事件关联 | A 告警 → 5min 内 B 告警 |
| **Indicator Match** | 与威胁情报列表匹配 | IP 命中恶意 IP 清单 |
| **New Terms** | 字段出现从未见过的值 | 从未见过的进程名 |

其中 **EQL 与 New Terms/Indicator Match 是 SIEM 特有的高价值类型**：EQL 表达"攻击链的先后顺序"，New Terms 抓"第一次出现即可疑"（新进程、新外联域名），Indicator Match 把外部 IOC 情报喂进检测。

## 2.2 创建规则

手工规则从查询起步，配合严重度、MITRE 技术、Investigation Guide 和阈值：

```text
Security → Rules → Create new rule

Type: Query
Index: winlogbeat-*
Query: event.code:4625 AND winlog.event_data.Status:0xC000006A
Threshold: count > 20 (for last 10 minutes)

Name: "Brute Force Login Attempt"
Severity: Critical
MITRE: T1110.001 (Brute Force: Password Guessing)
```

上例用 Windows 安全事件 4625（登录失败）+ 状态码 `0xC000006A`（密码错误，排除掉账号锁定/禁用等其他失败原因），10 分钟内同主机失败超 20 次即告警——**状态码过滤是关键**，不加它会把大量"账号不存在/已锁定"的正常运维失败也统计进去，直接造成误报。更实用的是**优先启用 prebuilt rules**（数千条官方规则，随版本更新）：

```bash
# 批量启用官方规则（按 Tag/OS 过滤）
POST /api/detection_engine/rules/_bulk_action
{
  "action": "enable",
  "query": { "bool": { "must": [
    { "term": { "ruleset.type": "default" } },
    { "term": { "os": "Windows" } }
  ] } }
}
```

**设计考量**：

- **先跑 prebuilt 再自建**：官方规则覆盖了绝大多数 ATT&CK 常见技术且持续维护，自建规则应聚焦官方没有的业务专有场景（内部系统访问模式）。
- **Fidelity 与 Severity 分开用**：Severity 表示"如果为真有多严重"，Rule Fidelity（Low/Medium/High）表示"这条规则多准"，两者组合决定是否进值班通道，比单一阈值灵活。
- **每条规则带 Investigation Guide**：用 Markdown 写明"命中后先看什么、查哪个字段、常见误报原因"，把分析师经验固化进规则。

> **行业实践**：新集群上线检测规则采用**分批启用**——先开 T1059（命令执行）、T1078（有效账户滥用）等高置信技术，稳定两周降噪后再逐步扩大覆盖面。一次性全开数千条 prebuilt rule，SOC 第一天就会被告警淹没，之后团队对整个平台的信任度很难重建。

# 3. Timeline 调查

Timeline 是**跨数据源的事件时间线编排画布**，把散落在不同索引、不同查询的证据按时间轴拼到一起，是深度调查的核心工作台：

```text
Security → Timelines → Create new

Timeline 允许手动探索和分析事件:

1. 添加查询
   Index: winlogbeat-*
   Query: host.name:"web-server-01"

2. 查看事件时间线
   ┌──────────────────────────────────────┐
   │ 10:00:01  用户 admin 登录成功         │
   │ 10:01:15  用户 root SSH 登录失败      │
   │ 10:01:16  用户 root SSH 登录失败       │
   │ 10:01:17  用户 root SSH 登录失败       │
   │ 10:02:00  用户 root SSH 登录成功       │
   └──────────────────────────────────────┘

3. 保存 Timeline → 附加到 Case
```

时间线支持**拖拽告警、原始文档进画布**，标注 pin（在时间点打标记），配置字段别名与"可疑点"高亮。从告警直接"Add to timeline"是标准操作流：告警只是入口，把相关主机的进程创建、网络连接、登录事件拉进同一时间线看**先后因果**，才能判断这是不是真实攻击链。

# 4. 威胁狩猎：EQL 与 KQL

狩猎（Hunt）是**主动找异常**而非等规则告警，工具是 **Threat Hunt（EQL 查询）**与 Explore/KQL。EQL 是专为端点事件时序设计的查询语言，核心能力是**序列匹配**：

```kql
// KQL: 快速过滤（狩猎起点，找可疑进程）
process.name : "powershell.exe" and process.args : ("-enc", "FromBase64String")
```

```text
// EQL: 序列匹配（先创建计划任务 → 随后该任务进程启动）
sequence by host.id with maxspan=5m
  [process where event.action == "creation" and process.args : "schtasks.exe"]
  [process where event.action == "start" and process.parent.name : "svchost.exe"]
```

EQL 与 KQL/Lucene 的分工要清楚：**KQL/Lucene 做集合过滤与聚合，EQL 做有顺序、有时间跨度的模式匹配**。持久化后门、横向移动的"先 A 后 B"特征，用普通查询几乎无法表达，EQL 一条序列搞定：

| 能力 | KQL/Lucene | EQL |
|------|-----------|-----|
| 字段匹配/布尔 | ✓ | ✓ |
| 聚合计数 | ✓（靠规则 Threshold） | ✗ |
| 事件顺序/时间跨度 | ✗ | ✓（sequence + maxspan） |
| 同一实体关联（by host.id） | ✗ | ✓ |
| 学习成本 | 低 | 中 |

> **注意**：EQL 序列规则**对数据完整性极敏感**——如果前一个事件（如进程创建）因采集缺失没入库，序列就永远凑不齐。攻击者常用的正是"删日志/绕过采集"，因此 EQL 检测要与"数据源健康监控"配套，**采集断流本身应该是一条高危告警**。

# 5. Cases 案例管理

Cases 把"一次安全事件的调查全过程"做成可协作、可追踪的工单对象，与 Alerting/Connector/Jira/ServiceNow 打通：

```text
Security → Cases → Create case

Case: "Web Server Compromised"
  ├── Description: 描述安全事件
  ├── Assignee: 分配给安全分析师
  ├── Tags: bruteforce, ssh, production
  ├── Timeline: 关联调查时间线
  ├── Comments: 团队协作讨论
  ├── Attachments / Evidence (把告警、文档挂为证据)
  └── Connectors: 自动创建 Jira Ticket / ServiceNow Incident
```

告警可**一键转 Case**并带上所有关联告警与证据；多条相似告警可**归并到同一 Case**（防止同一主机被规则重复告警产生 N 个工单）。Case 状态（Open/In Progress/Closed）与优先级构成简单的响应工作流，配合 **Connector 双向同步**把安全流程并入组织既有的 ITSM 体系。

# 6. MITRE ATT&CK 映射

Elastic Security 原生以 **MITRE ATT&CK 框架组织检测覆盖**：每条规则关联一个或多个 Technique ID，Dashboards 提供 **MITRE ATT&CK Matrix 覆盖视图**（哪些技术有规则、告警密度如何），Timeline/告警详情也标注技术号。

```text
MITRE 覆盖矩阵（节选，单元格数字 = 该技术的告警数）
            Initial Access  Execution   Persistence   Priv. Escalation
T1566 Phish      3    | T1059 Script   12   | T1547 RunKey  1  | T1548  ...
T1190 Exploit    1    | T1021 RSI      4    | T1053 Task    2  |
        └── 空白技术 = 检测盲区 → 优先补齐对应数据源与规则
```

价值在于把"我有多少条规则"这种虚荣指标，换成"**我覆盖了攻击生命周期的哪几格、哪几格是空的**"这种可行动指标。空白格通常指向**数据源缺口**而非规则缺口——比如没有 DNS 日志就填不了 T1071（应用层协议）那一格。

> **行业实践**：用覆盖矩阵做**季度检测能力评审**——按业务最可能遭受的攻击面（对外 Web 服务重 Initial Access/Execution，内网重 Lateral Movement/Exfiltration）圈定优先级，逐格补齐数据源与规则，并统计每格的告警数与真阳性率。这张矩阵是给管理层汇报"安全投入产出"最直观的一张图。

# 7. Hosts 与 Network 面板

Hosts/Network 面板把端点与流量数据做成**可下钻的资产视图**（数据来自 Elastic Defend agent 与网络 integration）：

```text
Security → Hosts

Host 列表:
  Host Name         IP           Alerts  Risk Score
  web-server-01    10.0.1.10    3       75 (High)
  db-server-01     10.0.1.20    0       10 (Low)

点击 Host → 查看:
  ├── Authentications (登录活动: 成功/失败/来源地)
  ├── Uncommon Processes (异常进程)
  ├── Anomalies (ML 检测的异常)
  └── Events (原始事件列表)
```

```text
Security → Network

网络流量分析:
  ├── DNS 查询统计（可疑域名、DGA、隧道）
  ├── HTTP 请求分析
  ├── TLS 证书信息（自签、异常签发者）
  ├── Flow 记录（来源/目标 IP + 端口，横向移动视图）
  └── External IP 流量（识别出站到可疑 IP / 数据外传）
```

Hosts 的 **Risk Score（风险评分，需 Enterprise + 风险打分规则）** 按"实体近期告警的严重度与新鲜度"加权，把"该先查哪台机器"变成排序问题。**Network 页的 External 视图专治数据外传**：出站目的地不在白名单、目标端口异常、单会话流量异常大，三条线交叉最能抓 C2 与 exfil。

# 8. 告警集成与自动化响应

Detection Rules 触发后进入统一的 **Alerts 工作流**，与 Kibana Alerting/Case/Connector 打通，实现从"检测 → 分诊 → 响应"闭环：

```text
Security → Alerts

Detection Rules 触发 → 生成 Alert:
  - Alert Table: 所有告警列表（按严重度/时间/状态排序）
  - Alert Details: 关联事件详情 + 实体 + MITRE + 上下文
  - Alert Actions: 发送到 Connector（Slack/Jira/SOAR webhook）
  - Alert Status: Open → Acknowledged → Closed (带处置注解)
```

自动化（Workflows/Connector）承担 **Tier-0 分诊与轻量响应**：命中低危规则自动补全富化（查资产归属、查威胁情报）、自动关单误报模式、自动触发隔离主机（调 EDR API）等，把分析师从重复劳动里解放出来。

> **注意**：自动响应动作（隔离、封 IP）**必须先在"仅告警不执行"模式下观察一段时间**，并设置**回滚与白名单护栏**——一条写歪的检测规则配上自动隔离，可能一夜之间把生产主机批量下线，这是 SIEM 自动化最惨烈的翻车姿势。高危动作保留人工确认（在 Case 里点按钮触发），低危动作才放手自动化。

# 9. 与专用 SOC 平台的分工

| 维度 | Elastic Security | Splunk Enterprise Security / QRadar | 专用 SOAR (Phantom/XSOAR) |
|------|------------------|--------------------------------------|----------------------------|
| 强项 | 检测 + 调查 + 端点一体，数据同库、检索快、TCO 低 | 成熟 SIEM 内容生态、合规报表 | 复杂跨系统响应编排 |
| 数据源适配 | ECS 映射，官方集成多 | 富 Add-on 生态最全 | 靠 App 集成 |
| UEBA/用户画像 | 靠 ML 模块（需 Platinum） | 内置 UEBA 较成熟 | 一般不含 |
| 响应编排 | Connector + 简单 workflow | 需外接 SOAR | 核心能力 |
| 适合 | 已用 ELK / 重检索与端点的团队 | 大型合规驱动 SOC | 已有 SIEM 补响应 |

Elastic Security 的定位是**"检测 + 调查 + 端点"一体化平台**，在中小规模 SOC、以及"把安全并入现有可观测栈"的场景里性价比突出；当组织已有独立 SOAR 或强合规报表需求时，常见架构是 **Elastic 做检测与数据底座 + 外部 SOAR/工单做编排与合规**，两者靠 Case Connector 与 webhook 衔接，而不是用 Elastic 硬顶 SOAR 的编排能力。

# 10. 小结

- **地基是 ECS**：日志不映射 ECS，检测规则、MITRE 覆盖、Hosts/Network 全部空转；SIEM 项目八成失败在数据接入而非规则本身。
- **规则引擎**：query/threshold 管频率型、EQL 管序列型、New Terms/Indicator 管"首次出现/情报命中"，优先启用并分批放开 prebuilt rules，自建规则聚焦业务专有场景并附调查指南。
- **调查闭环**：Timeline 拼因果、Cases 管协作、风险评分排序、Workflows 做 Tier-0 自动化——告警只是入口，不是终点。
- **组织方式**：以 MITRE ATT&CK 矩阵衡量覆盖与盲区，空白格多半指向数据源缺口；自动响应务必先观察再放手、保留人工确认与回滚护栏。
