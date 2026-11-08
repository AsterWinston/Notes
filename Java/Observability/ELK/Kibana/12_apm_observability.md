# 1. Observability 总览

Kibana Observability 提供统一的**应用性能监控（APM）、基础设施监控和用户监控**界面，其底层思想是**三大支柱数据（Metrics、Logs、Traces）同存于 ES、用统一的时间与关联字段串起来**——这正是 ELK 方案相对"每类信号一个专用系统"的核心卖点：从一条慢 trace 直接跳到同时段的 pod 日志与主机指标，不需要在多个系统间手工对时间戳。

```text
Observability 三大支柱:
  ├── APM           → 应用性能（延迟/错误/事务）    traces
  ├── Infrastructure → 主机/容器/K8s 指标           metrics
  ├── Uptime        → 服务可用性检测                probes
  ├── User Experience → 真实用户监控（RUM）         browser traces
  └── SLO           → 目标达成度与错误预算          derived
```

8.x 把"APM"与"Infra"入口合并为 Observability 菜单，并引入 **Entity（实体）概念**：每个服务、主机、pod 都是一个 entity，logs/metrics/traces 都带 `service.name`、`host.name`、`k8s.pod.name` 等 ECS 关联字段，实体视图按这些字段自动聚合。看板子时如果某服务的日志关联不上指标，九成是采集端没把 ECS 字段打全（比如自研日志缺 `service.name`），**关联字段是统一可观测的第一等公民**。

> **注意**：APM 数据默认写入 `traces-apm*`、`metrics-apm*`、`logs-apm*` 等以 **OTel 风格数据集（data stream）** 组织的索引（8.7+ 的 intake v2 协议），与 7.x 时代的 `apm-*` 单索引结构不同。老集群升级后新旧索引并存，Data View 要覆盖 `traces-apm*` 与 `apm-*` 两套模式，否则历史 trace 查不到。

# 2. APM

## 2.1 数据模型：Transaction 与 Span

APM 的核心概念是 **trace（一次端到端请求）**，由 **transaction（入口工作单元）** 和 **span（transaction 内部的子工作）** 组成：transaction 代表请求进入服务的第一个动作（HTTP 请求、消息消费、定时任务），span 代表其内部的细分耗时（数据库查询、缓存读写、下游 RPC），**error 是挂在 trace 上的异常事件**。这套模型与 OpenTelemetry 的 trace/span 语义一一对应，OTel SDK 发来的 OTLP 数据可以被 APM Server 直接接收。

```text
trace 结构 (一次下单请求):
  transaction: POST /api/order        120ms   ← 服务入口
    ├── span: Controller              5ms
    ├── span: Service                 10ms
    │     └── span: JDBC select       80ms    ← 瓶颈
    ├── span: Redis get               3ms
    └── span: exit call → inventory   20ms
```

数据落到 ES 后，一次请求产生多条文档：transaction 文档（`traces-apm*`，含 duration、result、timestamp）、若干 span 文档（含 `parent_id` 指向 transaction 或其他 span）、error 文档（`errors.apm*`），**trace 本身不是存储实体，而是靠 `trace.id` 字段在查询时聚合出来的**——这决定了深挖单条 trace 永远从 Discover 按 `trace.id` 查，而不是有什么"trace 索引"。采样上，8.x 支持 **tail-based sampling**（在中央 Collectors 上按条件保留慢/错 trace，其余丢弃），无采样时高吞吐服务的 span 量非常可观，**容量规划按"每 transaction 5–20 span"估算文档数**。

## 2.2 Agents 语言矩阵

Java 生态接入以 Elastic APM Java Agent 最成熟（字节码增强，零代码改动），各语言 agent 能力有差异：

| 语言/运行时 | Agent | 框架自动覆盖 | 分布式追踪 | 备注 |
|-------------|-------|--------------|-----------|------|
| Java 8+ | elastic-apm-agent | Spring WebMVC/Boot、JAX-RS、JDBC、Kafka、ES 客户端 | ✓ | `-javaagent` 挂载即可，最成熟 |
| Go | apmgo (v2) | net/http、gin、gRPC、database/sql | ✓ | 需少量插桩代码 |
| Node.js | elastic-apm-node | Express、Fastify、http、pg/mysql | ✓ | |
| Python | elastic-apm-py | Django、Flask、FastAPI、requests | ✓ | |
| .NET | ElasticApmAgent | ASP.NET Core、HttpClient、EF | ✓ | |
| PHP / Ruby | 官方 agent | 常用框架 | ✓ | 迭代慢于 Java/Go |
| iOS / Android / Browser | RUM agents | 页面/点击/网络请求 | ✓ | 归 User Experience |
| 任意语言 | OTel SDK / Collector | 全生态 instrumentation | ✓ | OTLP 直连 APM Server |

Java 服务的典型接入配置，agent 版本必须与 stack 大版本对齐：

```bash
# JVM 启动参数（agent jar 与 apm-server 同大版本）
java -javaagent:/opt/elastic-apm-agent-8.11.0.jar \
     -Delastic-apm.service_name=order-service \
     -Delastic-apm.server_urls=https://apm.internal:8200 \
     -Delastic-apm.secret_token=<enrollment-token> \
     -Delastic-apm.transaction_sample_rate=0.2 \
     -Delastic-apm.enable_span_metrics=true \
     -jar order-service.jar
```

> **行业实践**：新项目优先评估 **OpenTelemetry SDK + Collector 出口到 Elastic**（OTLP），把厂商锁定降到最低；存量 Java 微服务用 Elastic Java Agent 成本最低——一个启动参数就有完整链路。两者可混用，traceparent 头互认，链路不断。

## 2.3 Service Map

Service Map 由 APM Server 根据 span 中的 `destination.service.resource` 字段**自动推断服务依赖拓扑**，无需手工配置——前提是框架被 agent 覆盖（HTTP client、JDBC、Kafka 等调用会带上目的地信息），**自研的裸 Socket 调用不会出现在图上**。节点颜色实时反映延迟与错误率（红=有错误、黄=慢），点击节点直接进入该服务的 Trances/Errors/Metrics 视图。

```text
APM → Services → 选择 Service → Service Map

自动生成服务依赖拓扑:
  ┌────────┐     ┌────────┐
  │Gateway │────→│Order   │──→ MySQL
  └────────┘     │Service │──→ Redis
                 └────────┘──→ Kafka
```

图边缘的箭头方向表示调用方向，**图上看不出来的依赖 = 采集覆盖不到的调用**，这是审计 agent 覆盖率的快捷入口。

## 2.4 Transactions — 事务

Transactions 视图是**按事务类型聚合的性能大盘**，看的是分布而非单条：

```text
APM → Services → order-service → Transactions

Transaction 列表:
  Name                Avg. Duration  Throughput  Error %
  GET /api/order/{id}    120ms         50 tpm      0.2%
  POST /api/order         85ms         20 tpm      0.0%

点击 Transaction → 查看:
  ├── Timeline (Waterfall): Span 耗时瀑布图
  │     ├── Span: Controller (5ms)
  │     ├── Span: Service (10ms)
  │     │     └── Span: JDBC select (80ms)  ← 瓶颈
  │     └── Span: Redis get (3ms)
  ├── Metadata: HTTP Headers, User Agent
  └── Labels: 自定义标签
```

排查从这页开始是标准动作：**先按 "Average duration / P95" 排序找到最慢事务类型，再进入样本列表挑一条 trace 看瀑布图定位瓶颈 span**。注意 `enable_span_metrics` 打开后每个 span 目标（数据库、外部服务）会自动生成服务依赖指标（`Service Dependencies` 页），回答"order-service 的延迟里 MySQL 占多少"这类归因问题。

## 2.5 Errors

Errors 按异常签名聚合，点开一个错误组能看到堆栈、发生时间分布、受影响事务与请求上下文：

```text
APM → Services → order-service → Errors

Error 列表:
  Type                  Count  Message
  NullPointerException  12     at com.example.UserService.getUser
  TimeoutException       5     Connection timeout to redis

点击 Error → 查看:
  ├── Stack Trace
  ├── Occurrences (发生时间分布)
  ├── Affected Transactions
  └── Context (请求参数 / 用户信息)
```

**设计考量**：错误上下文默认开启但敏感——`transaction.context.request.body`、用户信息可能含 PII，Java agent 用 `sanitize_field_names`（默认屏蔽 password、token、*secret* 等）和 `central_config` 控制，**接入评审时要把"错误上报里到底带不带用户数据"当作合规检查项**，而不是事后补救。

# 3. 与 SkyWalking / OpenTelemetry 对比

| 维度 | Elastic APM (Kibana 8.x) | SkyWalking | OpenTelemetry (SDK+Collector) |
|------|--------------------------|------------|-------------------------------|
| 定位 | 可观测套件中的 traces 层，与 logs/metrics 同栈 | 独立 APM 平台，自带 UI/存储 | 采集标准与框架，本身不含后端 |
| 数据模型 | trace/transaction/span，OTel 语义兼容 | trace/segment/span | trace/span 标准定义 |
| 存储 | ES（复用现有集群与检索能力） | 自建存储（BanyanDB/H2/ES 可选） | 取决于导出后端 |
| Java 探针 | 字节码增强，开箱成熟 | 字节码增强，国内生态最全 | 社区 instrumentation 极广 |
| 分析能力 | Discover/Lens/SLO/告警全家桶 | 拓扑/热力图/告警内置 | 无（靠后端） |
| 与日志关联 | 天然同库，trace_id 一键跳日志 | 需自建日志系统配合 | 取决于后端 |
| 采样 | head-based + tail-based sampling | agent 端限流 | SDK/Collector 灵活策略 |
| 厂商锁定 | 低（可 OTLP 进出） | 中（私有协议为主，支持 OTLP 接入） | 最低（标准本身） |
| 适合 | 已重度使用 ELK 的团队 | 需要独立完整 APM 且不想动 ELK | 多后端、长期标准化战略 |

> **行业实践**：已经为日志/指标买了 ES 团队的，Elastic APM 的边际成本最低（一套存储、一套权限、一套告警）；没有 ELK 基础、纯中文生态微服务，SkyWalking 的开箱拓扑与告警更省心；无论选哪个，**新服务用 OTel SDK 埋点、后端可换**是最抗风险的姿势——Elastic APM Server 原生接收 OTLP，SkyWalking 也有 OTLP receiver，前端埋点标准与后端选择可以解耦。

# 4. Infrastructure

Infrastructure 视图基于 Elastic Agent 的 system/kubernetes integration 上报的指标，本质是 **按 ECS 实体聚合的指标大盘**：

```text
Observability → Infrastructure

Hosts:
  ├── CPU Usage (折线图)
  ├── Memory Usage (折线图)
  ├── Disk Usage (折线图)
  └── Network Traffic (折线图)

Containers (K8s):
  ├── Pod CPU / Memory
  ├── Pod Status (Running / Failed)
  └── Pod Logs (关联到 Discover)
```

每个实体行的三个快捷入口是日常运维主路径：**Metrics（跳转自动生成的实体指标页）、Logs（按 host.name/pod 字段过滤的 Discover）、Split（按任意维度拆分面板）**。K8s 场景的实用技巧是 Inventory 视图按 namespace/cluster 分组后开 **Inventory 规则**——"每个 deployment 一个监控单元"，新增服务自动纳入，不用手工维护看板清单。

# 5. Uptime

Uptime（Synthetics/检查）从外部主动探测服务可用性，8.x 中由 **Synthetics 心跳脚本**（基于 Playwright）与轻量 TCP/HTTP check 组成：

```text
Observability → Uptime

Monitor 列表:
  Name             URL                  Status    P50      P99
  API Health       /api/health          Up        45ms     120ms
  Homepage         /home               Up        230ms    800ms

点击 Monitor → Duration trends → TSL/SSL 证书到期提醒
```

> **注意**：Synthetics 的浏览器心跳需要 **Elastic Agent + 专用 Fleet policy**（journey 存为 Saved Object、在私有 location 的 agent 上执行），不是 Kibana 进程内置能力；纯 HTTP 探测也要规划好探测点与生产流量的地理差异，单点探测的"可用性"仅代表一个视角。证书到期监控顺带在这里做，比另搭脚本省心。

# 6. User Experience (RUM)

真实用户监控从浏览器/移动端上报，与后端 APM 通过 **trace 传播自动打通**（前端 transaction 与后端 transaction 同属一条 trace，端到端瀑布图直接展示"白屏时间 vs 接口时间"的比例）：

```text
真实用户监控（需集成 APM RUM Agent）:

Observability → User Experience
  ├── Page Load Duration (P50/P75/P90/P95)
  ├── Core Web Vitals: LCP / FID / CLS
  ├── Breakdown (每个步骤的耗时: TTFB/DOM/资源加载)
  └── Browser/OS/Geo 分布
```

Web Vitals 三个指标的定位不同：**LCP 查渲染慢（图片/首屏接口）、FID/INP 查交互卡（JS 主线程阻塞）、CLS 查布局抖动（异步资源无占位）**，User Experience 页按指标劣化直接下钻到具体页面与浏览器版本组合，适合前端团队自主巡检。

# 7. SLO 管理

SLO 功能（Enterprise License）把"服务达标与否"从人眼看图变成**可度量的错误预算账本**：定义 SLO 的历史窗口（30 天滚动）、目标（99.5%）、指标来源（可以是 APM 事务指标、原始索引查询、或 Uptime monitor），系统持续计算 SLI 并展示预算燃烧。

```text
Observability → SLO

创建 SLO:
  - Service: order-service
  - Indicator: Latency < 500ms
  - Objective: 99.5% in 30-day rolling window

Dashboard:
  SLI value:
  ┌─────────────────────────────────────────────────┐
  │  ████████████████████████████████████████░░ 99.2% │
  │  ─────────────────────────────────────────────── │
  │         Target: 99.5%         Budget: -0.3%      │
  └─────────────────────────────────────────────────┘
```

**设计考量**：SLO 规则（burn rate alerting）比单点阈值告警更符合 SRE 工作法——**快燃烧**（短窗口消耗大比例预算，如 1 小时烧掉 2%）叫值班，**慢燃烧**（72 小时烧掉 5%）开工单，两条规则组合（Google SRE Workbook 的多窗口多速率法）在 Alerting 的 SLO 规则类型里直接可配。

> **行业实践**：SLO 的价值不在监控而在**决策约束**——错误预算耗尽就冻结高风险发布，这需要把 SLO 状态接进发布流程（CI/CD gate 读 SLO API）。只做看板不接入流程，SLO 会退化成又一块没人看的仪表盘。另外窗口别贪大：30 天滚动 + 自然月双口径最常见，季度窗口的预算对日常决策太迟钝。

# 8. 小结

- **数据模型**：transaction/span/error 以 `trace.id` 逻辑聚合存于 OTel 风格 data stream，慢请求排查的标准路径是 Transactions 排序 → 样本 trace 瀑布图 → 瓶颈 span。
- **接入选型**：存量 Java 用 Elastic agent 一个启动参数起步，新项目走 OTel SDK/OTLP 保标准；RUM 与后端 trace 自动贯通是 ELK 相对独立 APM 的招牌能力。
- **与 SkyWalking/OTel 的分工**：有 ELK 底座选 Elastic APM 边际成本最低，独立 APM 需求选 SkyWalking，埋点层统一用 OTel 降低锁定。
- **SLO 是出口**：trace/metrics/logs 最终服务于"能不能发布、烧了多少预算"的判断，SLO + 燃烧率告警把这套判断自动化，前提是 ECS 关联字段先打全。
