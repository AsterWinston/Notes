
# 1. 主流 APM 全景

选型讨论的前提是先分清两类东西：**分布式追踪系统**（Zipkin/Jaeger，核心是 trace 采集与查询）与**一体化 APM 平台**（SkyWalking/Pinpoint，trace 之外还有指标聚合、告警、代码级诊断）；而 **OpenTelemetry 不是这两类中的任何一类**——它是采集标准与 SDK/Collector 体系，本身不提供存储、聚合与 UI，需要配一个后端。把 OTel 与其余四者放进同一张"竞品表"是选型文章里最常见的范畴错误，正确姿势是把 OTel 当成**所有方案共同的上层趋势**来看（第 6 节）。

## 1.1 五方对比表

| 维度 | SkyWalking | Zipkin | Jaeger | Pinpoint | OpenTelemetry |
|------|-----------|--------|--------|----------|---------------|
| **定位** | 一体化 APM | 分布式追踪 | 分布式追踪 | 一体化 APM（代码级） | 采集标准 + SDK/Collector |
| **开源治理** | Apache 顶级项目 | 独立 OSS（Linux 基金会挂靠） | CNCF 毕业 | Naver 开源（Apache-2.0） | CNCF（OpenTracing+Census 合并） |
| **传输协议** | gRPC/HTTP（可接 Kafka） | HTTP/Thrift/Kafka | gRPC/HTTP（v2 基于 OTel Collector） | gRPC | OTLP（gRPC/HTTP） |
| **上下文传播** | sw8 / sw8-correlation | B3/W3C | W3C traceparent / uber-header | 私有头（Pinpoint-TraceID 等） | W3C tracecontext / B3（可配） |
| **存储** | BanyanDB/ES/MySQL/H2 | MySQL/Cassandra/ES | ES/Cassandra/Badger | HBase/PostgreSQL | 无（交给后端） |
| **Metrics** | ✓ OAL 引擎内置 | ✗ | 有限（依赖 OTel/指标导出） | ✓ 内置 | ✓ SDK 信号标准 |
| **日志关联** | ✓ LAL + TraceId | ✗ | 有限（ES 关联） | ✓ 日志上送 | ✓ Logs 信号标准 |
| **告警** | ✓ alarm 模块 | ✗ | ✗（交给外部） | ✓ 有限 | ✗（交给后端） |
| **代码级诊断** | ✓ Profiling（线程快照/eBPF） | ✗ | ✗ | ✓ 方法级调用栈（强项） | ✗（SDK 层，非诊断） |
| **服务拓扑** | ✓ STAM 流式分析 | 有限（依赖聚合） | ✓ | ✓ | ✗ |
| **多语言** | Java/.NET/Go/Python/Node/PHP/eBPF | 各语言客户端 | 多语言 SDK（转 OTel） | Java 为主（PHP 等实验性） | Java/Go/Python/JS/.NET 等全覆盖 |
| **Agent 侵入方式** | 字节码增强（零代码） | 需集成客户端库 | SDK 埋点（或借 OTel auto-instrument） | 字节码增强（更激进） | SDK/auto-instrument agent |
| **UI/Dashboard** | 内置且完整 | 极简 | 简洁 | 内置（重 Inspector） | 无 |
| **性能开销** | 低（3%~8%） | 低 | 低 | 中偏高（增强粒度大） | 取决于 SDK 后端 |

一句话画像：**Zipkin 是追踪界的"最小可用"，Jaeger 是云原生生态的追踪标准件，Pinpoint 是 Java 深度诊断的极客，SkyWalking 是开箱即用的全栈 APM，OTel 是把前三者的采集层统一掉的那股潮水。**

# 2. 场景推荐

## 2.1 纯 Java 微服务 → SkyWalking / Pinpoint

```text
SkyWalking 优势：
  - Apache 顶级项目，社区活跃，中文资料与国内生产案例密度最高
  - 内置 Metrics(OAL) + 告警 + Profiling + 日志关联，开箱即全栈
  - 字节码增强零侵入，Spring Cloud/Dubbo/ShardingSphere 等国内主流框架覆盖全
  - 存储可插拔（BanyanDB/ES/MySQL）

Pinpoint 优势：
  - 方法级 Inspector：调用栈、参数、返回值、SQL 绑定值可直接回放
  - 字节码增强粒度更细，排查疑难逻辑时信息量惊人
  - 缺点：HBase 运维门槛、增强开销相对偏高、非 Java 生态弱、UI 交互年代感
```

两者取舍点在于"信息深度 vs 运维成本"：多数 Java 团队要的是拓扑 + 慢 Trace + 告警闭环，SkyWalking 够用且轻；确实需要逐方法看参数返回值的调试级需求（且养得起 HBase），Pinpoint 不可替代。

## 2.2 多语言混合 → SkyWalking / Jaeger / OTel+后端

```text
SkyWalking 优势：
  - 探针矩阵广（.NET/Go/Python/Node/PHP/eBPF），协议统一，一套后端一种 UI
  - 各语言数据在同一拓扑里自动关联（sw8 互通）

Jaeger 优势：
  - CNCF 毕业，K8s Operator/生态集成成熟，轻量部署一条命令
  - 与 Prometheus/Grafana 组合补齐指标侧
  - 缺点：APM 能力薄（聚合、告警、诊断都要外挂拼）

OTel 路线：
  - 各语言统一 OTel SDK 埋点/自动插桩，数据导给任意 OTLP 后端
  - 避免"每加一门语言换一套探针体系"，是当前新项目的默认起手式
```

## 2.3 K8s + Service Mesh → Istio + SkyWalking

```text
Istio Envoy 的 ALS → SkyWalking OAP：
  优点：零注入（Sidecar 层面采集），全集群拓扑与 L4/L7 指标当天铺开
  缺点：不覆盖应用内部 DB/Redis/方法调用，深度诊断仍需补 Agent 或 OTel SDK
  现状：Ambient 模式削弱 ALS 路径，Mesh 场景也在向 OTLP 靠拢（见 13 篇）
```

## 2.4 商业 SaaS → Datadog 类

商业方案的优势是开箱即用与 AI 分析，劣势是按数据量计费在大规模下成本陡增、数据出境/私有化合规障碍。国内类似定位的是各家云厂商 APM。选它与否基本取决于预算与合规，不属于本文技术主线。

# 3. 选型矩阵

```text
团队技术栈?
  ├── 纯 Java，要全栈 APM           → SkyWalking / Pinpoint
  ├── Java + Go/Python/Node 混合     → SkyWalking，或全量 OTel SDK + 后端
  └── 只要追踪、指标靠 Prometheus    → Jaeger / Zipkin

是否需要 Metrics + 告警 + Profiling 一体?
  ├── 是，且想少运维               → SkyWalking
  └── 否，DIY 拼装文化浓           → Jaeger + Prometheus + Grafana

是否 K8s + Service Mesh?
  ├── 是                           → SkyWalking ALS / Jaeger(Istio 默认集成)
  └── 否                           → 任一均可

运维能力与存量栈?
  ├── 有 ES 经验                   → SkyWalking(ES 后端) / Jaeger(ES)
  ├── 有 HBase 经验                → Pinpoint
  ├── 有 Cassandra 经验            → Jaeger
  └── 都不想再养新组件             → SkyWalking + BanyanDB

合规与预算?
  ├── 必须私有化                   → 开源阵营全选（注意各自 license 都是 Apache-2.0）
  └── 预算充足、要 SLA 兜底        → 商业 SaaS / 云上托管
```

# 4. 从 Zipkin/Jaeger 迁移

SkyWalking OAP 提供旁路接收能力，可以让旧系统"边跑边迁"，不必一刀切：

```yaml
# config/application.yml —— 开启 Zipkin Receiver，兼容既有 B3 客户端
receiver-zipkin:
  selector: ${SW_RECEIVER_ZIPKIN:default}   # 默认关闭(-)，改为 default 开启
  default:
    # Zipkin 兼容采集：应用无需改代码，zipkin sender 指过来即可
    enableHttpCollector: true
    restHost: 0.0.0.0
    restPort: 9411          # zipkin 标准端口，DNS 一换即切流
    sampleRate: 10000       # 1/10000 精度
    # 也可 enableKafkaCollector 从既有 zipkin kafka topic 消费
```

迁移四步：**第一步**只开接收端不动应用，把 zipkin 数据双指向 SkyWalking（zipkin reporter 配两个 collector 或走 Kafka 分流）验证数据完整；**第二步**Java 应用逐个挂 sw Agent 获得深度数据；**第三步**对比一个账期内的拓扑/指标口径，修正命名与 ignore 规则；**第四步**保留旧系统数据到 TTL 过期后下线。Jaeger 侧当前版本没有专用 receiver，走 **OTLP 路线**（Jaeger 数据经 OTel Collector export 到 OAP，或 SDK 直接换 OTLP）。注意迁移期两套 trace 体系传播头不同（B3 vs sw8），跨体系调用会在边界处断开——**混跑期要接受"半截链路"**，把边界服务优先迁完再谈全局拓扑。

# 5. 自研 vs 开源

很多团队第一反应是"观测数据反正是自己产生的，自研一套更贴合"。把成本拆开看结论就清楚了：

| 维度 | 自研 | 开源(SkyWalking 类) |
|------|------|---------------------|
| 采集层 | 字节码增强/agent 分发是大坑（JDK 版本、框架兼容矩阵） | 现成插件生态 |
| 分析层 | 流式聚合、拓扑推断(STAM 类)、降采样策略需长期打磨 | OAL/MAL 引擎现成且可定制 |
| 存储层 | 时序模型设计 + TTL + 索引治理全部自建 | 可插拔后端 |
| 人力 | 常态 2~3 人团队起步，断人即腐化 | 部署 + 规则维护 |
| 贴合度 | 上限高（可与内部发布/工单系统深度联动） | 靠扩展点（receiver、插件、GraphQL 前端） |
| 风险 | 人员流动 = 平台归零 | 社区方向不由己，需评估活跃度 |

> **行业实践**：主流折中是"**采集与分析用开源，增值层自研**"——Agent/OAP 直接用 SkyWalking 或 OTel SDK，自研只花在租户化 UI、告警路由、与内部 CMDB/工单/发布系统集成。历史上纯自研 APM 存活超过五年的团队极少，原因大同小异：观测平台的竞争力在于兼容矩阵的持续维护，这是一项没有终点的苦役，不是三个月能交付的功能。

**APM 选型的组织因素**常被低估，实际权重不低于技术对比：团队的存储运维基因（ES/MySQL/HBase 哪个有人）、探针治理的推进手段（有没有平台团队能强制统一命名）、语言分布的中长期规划（新增 Go 服务是否已成事实）、中文社区支持（SkyWalking 在这点上国内几乎无对手）、以及**谁来背 7×24 的值班**——开源方案选一个团队内有人愿意当 owner 的，比选"参数表最优"的重要得多。

# 6. OpenTelemetry 趋势与 SkyWalking 的 OTel 兼容路线

**OTel 已是事实标准的采集层**：OpenTracing 与 OpenCensus 合并后，CNCF 内热度仅次于 K8s；框架厂商（Spring Boot 3 的 micrometer-tracing、各 RPC 库）开始原生暴露 OTel API；Jaeger v2 干脆重建在 OTel Collector 之上。**趋势的判断是：供应商锁定的风险被 OTel 对冲掉了——埋点（SDK/传播标准）与后端（存储/聚合/UI）解耦，换后端不动业务代码。**

SkyWalking 对这一趋势的应对是"采集入口对齐 OTLP，深度能力保持自有"：

```yaml
# OAP 侧 receiver-otel：接收 OTLP 指标与日志（9.x 起逐步扩展）
receiver-otel:
  default:
    enabledHandlers: "otlp-metrics,otlp-logs"   # 开启 otlp-traces 可接收 OTLP Trace(早期转 Zipkin 格式经 zipkin query 查看, 新版本支持原生 trace 分析)
    enabledOtelMetricsRules: "k8s/*,mysql/*,redis/*,kafka/*"
```

兼容矩阵的现实程度要诚实评估：**OTLP metrics 的 handler 规则覆盖最广**（K8s/node exporter、主流数据库中间件的 metric 都有 MAL 规则映射成 SkyWalking 指标与仪表盘）；**OTLP trace** 可入库与基本分析，但 sw8 语境下的深度特性（Forced Tracking、部分组件级关联）依赖 SkyWalking 自家探针；**传播标准**上 SkyWalking 仍以 sw8 为主，与 W3C traceparent 的互通借助 OTel Collector/网桥转换或混合部署时统一埋点。反过来，SkyWalking Java Agent 也提供把 trace 导出到 OTLP、以及 OpenTracing/OTel SDK 兼容层的通道——"SkyWalking 探针 + OTel 后端组件"与"OTel SDK + SkyWalking 后端"两个方向都有官方路径。

> **注意**：不要因为"OTel 是标准"就得出"SkyWalking 过时"的结论——OTel 只统一了数据怎么产生与传输，数据进来之后的**聚合建模、拓扑推断、告警、Profiling、UI 这些 APM 的本体**，OTel 明确不做，仍需一个后端。真正的架构题是"埋点层选 OTel 还是厂商探针，后端选谁"，两问要分开答。

# 7. 最佳实践

1. **先定埋点策略再选后端**：新系统默认"OTel SDK 优先、厂商探针补充"，保证后端可换
2. **纯 Java + 要全栈 APM**：SkyWalking 综合成本最低；要方法级回放且能养 HBase 再考虑 Pinpoint
3. **只要 trace**：Jaeger/Zipkin 轻部署即可，别为用不上的 Metrics 引擎付运维税
4. **Mesh/K8s 优先铺 ALS 或 eBPF 底账**，核心链路再补探针，深度与覆盖分两步走
5. **迁移用旁路接收**（zipkin receiver/OTLP），双跑一个账期再下线旧系统
6. **自研止于增值层**：采集、聚合、存储用开源，自研聚焦租户 UI 与内部系统联动

# 8. 小结

| 问题 | 答案 |
|------|------|
| SW 与 Zipkin/Jaeger 的本质差异 | 一体化 APM（trace+metrics+alarm+profiling）vs 纯追踪；SW 存储可插拔且自带聚合引擎 |
| SW 与 Pinpoint 的差异 | 都是字节码增强，Pinpoint 深到方法参数但依赖 HBase、开销偏高；SW 广到多语言/eBPF、开箱全栈 |
| OTel 的角色 | 采集层事实标准，不提供 APM 本体；SW 的路线是 OTLP 入口兼容 + 自有探针深度并行 |
| 选型公式 | 语言分布决定探针矩阵，运维基因决定存储，值班文化决定自研与否，OTel 对冲锁定风险 |

最后一篇把本篇及全系列的高频考点整理成面试问答速查。
