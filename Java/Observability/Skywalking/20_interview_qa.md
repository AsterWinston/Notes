本篇两部分：第一部分是可扫读的高频考点清单；第二部分 12 题按面试真实问法展开，覆盖架构三组件、字节码增强原理、sw8 传播协议、L1/L2 聚合、采样机制、存储选型、断链排查、性能开销、与 Zipkin/Jaeger 差异、与 OpenTelemetry 关系等必考项。

# 1. 知识点

- **架构三组件** — Agent(探针)→OAP Server(分析平台)→Storage(存储)+UI; Agent 采集上报 gRPC, OAP 做接收/分析/聚合/写入四段管道且无状态可水平扩展, 存储是唯一有状态组件(ES/MySQL/BanyanDB/H2), UI 走 GraphQL 查询 OAP
- **探针矩阵** — 自动插桩探针(Java/.NET/Go/Node.js/PHP)、SDK 手动埋(apm-toolkit-trace)、eBPF(Rover)、Service Mesh(Envoy ALS); "字节码增强零侵入"是 Java Agent 的卖点
- **数据模型** — Trace→Segment(单进程内片段)→Span; Span 三类型 Entry(接收外部调用)/Exit(发起外部调用)/Local(进程内方法); Segment 间靠 refs 引用关联(记父 segmentId+spanId)
- **sw8 协议** — HTTP header `sw8` 共 8 段以 `-` 分隔: sample(0/1), traceId, parentSegmentId, parentSpanId, service, instance, endpoint, targetAddress; sample 与 parentSpanId 为明文数字, 其余字符串字段均 Base64; 整体默认限长 2k; 扩展头 sw8-x 携带跳过分析标志与发送时间戳(MQ 传输延迟计算); Dubbo 走 attachment、MQ 走消息 header 同构传播
- **Agent 原理** — premain + java.lang.instrument 注册 ClassFileTransformer, ByteBuddy 在类加载期按 skywalking-plugin.def 的增强定义(类匹配: byName/注解/父类接口)插入拦截器(before/after/onException); Agent 类走独立 ClassLoader 与应用隔离; 上下文栈存 ThreadLocal, 跨线程靠注解/包装器传递
- **上报机制** — Span 攒成 Segment 进内存 ring buffer channel, 满则溢写磁盘 buffer, gRPC 异步批量上报; 队列满丢数据但不阻塞业务线程(异步化是低开销的根基)
- **L1/L2 聚合** — L1 在内存按分钟窗口聚合流式数据(OAL 引擎产出 service_cpm/resp_time/sla 等), L2 把分钟结果降采样成 Hour/Day(downsampling 配置); 落存储的是聚合结果+原始记录两类
- **OAL** — 指标定义 DSL, 形如 `service_resp_time = from(Service.*).latency()`, 编译期生成聚合逻辑; 改指标口径不动代码
- **采样两级** — Agent 侧 sample_n_per_3_secs(每 3 秒限 Trace 条数, -1 全采); OAP 侧 trace-sampling-policy rate 精度 1/10000, 10000=全采(默认), 可按服务覆盖并配 duration 保留慢 Trace, forceSampleErrorSegment 保错误 Trace; 排查"Trace 变少"两级都要查
- **存储选型** — BanyanDB(SkyWalking 原生时序库, 9.x+ 官方主推, LSM 写优化省内存)/ES(大规模成熟, 注意映射与分片治理)/MySQL(日 Trace 千万内小团队)/H2(演示); 数据按天分索引, TTL 分 recordDataTTL(Trace 3-7 天)与 metricsDataTTL(指标 30 天)
- **告警** — alarm-settings.yml, 内存时间窗队列按分钟检查; 9.0~9.5 写 metrics-name+op+threshold+count, 9.6+ 改 MQE 表达式(如 avg(service_resp_time/1000)>1)+silence-period 防刷屏; 通知走 hooks 体系(webhook/slack/dingtalk/feishu, is-default 兜底), 消息体含 scope/name/ruleName/tags 结构化字段
- **拓扑** — STAM 流式拓扑分析, 从每条 Trace 的 Exit/Entry Span 配对提取边与边指标, 流量驱动非注册驱动; 客户端/服务端视角指标分离, 差值大指向网络/中间件层
- **断链根因** — 网关/LB 不透传 sw8 header; 异步线程池上下文丢失(@TraceCrossThread/RunnableWrapper 修); MQ 未走标准插件; 未插桩的自研 RPC; Agent/OAP 版本不兼容; sw8 超 2k 限制被中间件截断
- **性能开销** — 正常 3%~8% CPU; Span 创建微秒级, Segment 序列化异步; 反模式: 循环打 @Trace、trace_sql_parameters 全开、大 payload 进 tag; 容器环境 Agent 额外吃 50~150MB RSS, limits 要留头寸
- **与 OTel 关系** — OTel 是采集层标准(SDK+OTLP 协议)不做后端; OAP 提供 receiver-otel 收 OTLP metrics/logs/trace, Java Agent 也可导出 OTLP; 趋势=埋点与后端解耦, SkyWalking 深度能力(sw8/拓扑/OAL)仍自有
- **服务命名纪律** — agent.service_name 不含动态量(K8s 用工作负载名), 环境用 agent.namespace 隔离, 实例名用 POD_NAME; 多服务共用一个名字=拓扑假自环头号原因

# 2. 面试题

## 2.1 讲讲 SkyWalking 的整体架构和三大核心组件？

SkyWalking 是三层架构: 探针层 Agent 负责无侵入采集 Trace/JVM/日志/指标, 通过 gRPC(或 HTTP/Kafka 中转)上报; 平台层 OAP Server 是核心, 内部是一条四段管道——Receiver 接收反序列化、Analyzer 把 Segment 解析成 Trace 并识别服务/端点/实例、Aggregator 用 OAL 引擎做 L1 内存聚合与 L2 降采样、Exporter 批量写存储, OAP 本身无状态、靠集群模式(zookeeper/k8s/etcd 等)水平扩展; 数据层 Storage 存聚合指标与原始记录, UI 通过 GraphQL 查询 OAP。追问必答点: OAP 无状态意味着接收与聚合可以随便加副本, 系统天花板完全落在存储写入能力上, 容量规划先规划存储再规划 OAP。

## 2.2 SkyWalking Java Agent 是怎么做到无侵入监控的？

利用 JVM 的 java.lang.instrument 机制: 启动参数 -javaagent 触发 MANIFEST 里声明的 premain, Agent 向 Instrumentation 注册 ClassFileTransformer; 之后每个类加载都经过 Transformer, 按 skywalking-plugin.def 注册的增强定义匹配目标类(byName、类注解、方法注解、父类/接口匹配四种策略), 用 ByteBuddy 在目标方法前后织入拦截器代码——beforeMethod 里创建 Span(如 Entry Span 解析 sw8 头)、afterMethod 里停止 Span 并记状态、handleMethodException 里打错误标记, 全程不改一行业务代码、不重新编译。两个细节加分: Agent 自己的类用独立 ClassLoader 加载避免与应用冲突; 上下文栈挂在 ThreadLocal 上, 所以跨线程池/CompletableFuture 时需要 toolkit 的 @TraceCrossThread 或包装器显式传递, 这也是最常见的断链原因之一。

## 2.3 TraceId 如何跨服务传播？sw8 协议头里都有什么？

HTTP 场景由上游 Exit Span 注入 header `sw8`, 下游 Entry Span 解析并写入 Segment 的 refs(记录父 traceId/segmentId/spanId), 从而把进程内的多个 Segment 拼成一棵全局 Trace 树。sw8 值以 `-` 分隔共 8 段: 采样标志(0/1)、traceId、父 segmentId、父 spanId、父服务名、父实例名、父端点名、目标地址——其中字符串字段 Base64 编码, 整体默认限长 2k。另有扩展头 sw8-x(跳过分析标志、客户端发送时间戳, MQ 场景用后者算传输延迟)。Dubbo 走 RpcInvocation attachment、gRPC 走 metadata、Kafka/RocketMQ 走消息 header, 承载的都是同一组字段。面试常追问"sw8 和 W3C traceparent 什么区别": sw8 携带服务/实例/端点身份使 OAP 分析端不用二次关联即可构建拓扑, 代价是 header 更大、非 HTTP 中间件截断风险更高——透传链路上每一跳(Nginx/网关/LB)都要确认不重写、不裁剪这两个头。

## 2.4 SkyWalking 的指标是怎么算出来的？说说 L1/L2 聚合与 OAL？

SkyWalking 不把每条 Trace 存了再聚合查询, 而是在 OAP 内存中两级流式聚合: L1 在分钟窗口内把 Trace/日志/JVM 数据流式聚合成分钟级指标(如 service_resp_time、service_sla、endpoint_cpm), L2 再把分钟结果降采样成小时/天粒度(downsampling 配置, 这也是 UI 长时间范围看趋势数据更稀的原因)。指标提取逻辑由 OAL(Observability Analysis Language)定义, 形如 `service_resp_time = from(Service.*).latency()` 的类 SQL 脚本在编译期生成 Java 聚合代码——改口径是改配置不是改代码。落存储的因此只有两类: 聚合指标(小, TTL 30 天级)与原始 Trace 段(大, TTL 3~7 天), 这个分层正是 SkyWalking 比"裸 ES 存全量明细"省资源的关键。

## 2.5 高流量场景 SkyWalking 如何控制数据量？采样在哪一层做？

两级旋钮: Agent 侧 agent.sample_n_per_3_secs 限制每 3 秒开始采样的 Trace 数(默认 -1 全采), 在生成端就少造数据; OAP 侧 trace-sampling-policy-settings.yml 的 rate(精度 1/10000, 默认 10000 即全采)可按服务覆盖, 并配 duration 阈值让慢 Trace 无条件保留, forceSampleErrorSegment(默认开)保证错误 Trace 不丢。生产策略是分层降采: 入口网关与核心链路全采, 高 QPS 内部服务降 rate, 决策尽量集中在 OAP(改配置不重启业务应用)。必须说清代价: 降采样会稀释拓扑边权统计与告警数据源, 所以官方默认全采、靠慢/错强制保留兜底。另外 sw8 首段采样标志沿链传播, 上游不采下游就不采, 排查"整条链缺失"先查这两级配置而不是怀疑丢包。

## 2.6 SkyWalking 支持哪些存储？生产上怎么选型？

官方支持 H2(内嵌, 演示专用)、MySQL/PostgreSQL、Elasticsearch/OpenSearch、BanyanDB(SkyWalking 自研的观测数据时序库, 9.x 后官方主推)、以及边缘化的 Infinispan。选型按规模与组织技能走: 日 Trace 千万条以内且有 DBA, MySQL 够用但要自建按天分表清理; 大规模且团队有 ES 运维经验, ES 成熟, 但要治理分片数、TTL 和映射(高基数端点会引爆 fielddata); 新项目默认 BanyanDB——它按 SkyWalking 数据模型(时序指标+段流记录)原生设计, LSM 写优化、按时间分片整块过期、资源占用显著低于 ES。加分点: 存储直接决定查询能力上限(ES 支持 tag 检索与聚合、MySQL 只能索引点查), 且 recordDataTTL/metricsDataTTL 必须一开始就设(Trace 3~7 天、指标 30 天起步), 不设 TTL 等于给磁盘埋雷。

## 2.7 线上出现 Trace 断链，你的排查思路是什么？

先定位断点再定根因: 拿到上下游两条 Segment 对比 refs 字段——refs 为空说明上游没传出去, 有但指向不存在的实体说明中间件改写了上下文。然后按四类高频根因逐一排除: ①传播层切断, Nginx/网关/LB 没透传 sw8 header, 断点两侧抓 header(网关日志加 $http_sw8)对比即可实锤; ②异步丢失, 线程池提交任务没带上下文, 用 @TraceCrossThread 或 RunnableWrapper 修复; ③未插桩组件, 自研 RPC 或少见 MQ 需手工 inject/extract 补桥; ④版本/配置, 两端 Agent 大版本跨度导致协议不兼容, 或 sw8 超长被截断。注意区分断链与采样丢失: 后者是整个 Segment 都不存在(查两级采样配置), 前者是两侧都有但中间缺桥。经验: 断链绝大多数是治理问题不是 bug, 网关透传与线程池规范要写进接入 checklist。

## 2.8 SkyWalking 对业务性能的影响有多大？开销主要在哪？

官方口径正常负载 CPU 开销 3%~8%, 内存增加几十到一百多 MB。开销构成: 类加载期增强是一次性成本(启动变慢、Metaspace 上升); 运行期 Span 创建/结束是 ThreadLocal+对象池操作, 微秒级; 大头在 Segment 组装、序列化与 gRPC 上报——全部异步化(独立上报线程+内存 channel+磁盘 buffer 溢出保护), 队列满时丢观测数据而不阻塞业务线程, 这是"探针故障不拖垮业务"的设计底线。面试反转点: 开销异常通常是配置反模式而非 Agent 本身——循环体内打 @Trace 撑爆单 Segment、trace_sql_parameters 开启导致 Span 巨大、无 ignore 路径让健康检查淹没上报带宽。测量方法用同负载带/不带 Agent 对比, 别用 RSS 增长直接定罪(混淆业务对象增长)。

## 2.9 SkyWalking 和 Zipkin、Jaeger 有什么本质区别？

范畴不同: Zipkin/Jaeger 是分布式追踪系统, 核心交付是 Trace 的采集-存储-查询-可视化; SkyWalking 是一体化 APM, Trace 只是入口数据, 在采集之上还有 OAL 流式聚合引擎(指标体系)、STAM 拓扑分析、alarm 告警、日志关联与 Profiling 代码级诊断, UI 完整闭环。技术上三点差异: 传播协议上 Zipkin 用 B3/Jaeger 用 W3C traceparent, 只带 ID, 而 sw8 携带服务/实例/端点身份, OAP 不需要二次元数据关联即可实时构建拓扑与指标; 采集上 Zipkin/Jaeger 基本要求集成客户端库, SkyWalking 字节码增强零改动(并有 Go/eBPF/Mesh ALS 矩阵); 部署上 Zipkin 单 jar 最轻, SkyWalking 需要 OAP+存储, 运维最重但功能上限最高。选型一句话: 只需要 trace 选 Zipkin/Jaeger 够用, 要 APM 全家桶且 Java 为主选 SkyWalking; 同时提醒面试官今天还有 OpenTelemetry 这条线(见下题)。

## 2.10 OpenTelemetry 会取代 SkyWalking 吗？两者是什么关系？

不会取代, 因为层次不同: OTel 统一的是采集层——各语言 SDK、自动插桩、W3C 传播标准与 OTLP 传输协议, 明确不做聚合、拓扑、告警、UI 这些后端能力, 所以"用了 OTel"仍需回答"数据进谁的存储、用谁的仪表盘"。SkyWalking 的应对是双轨: OAP 提供 receiver-otel 接收 OTLP 三类信号(metrics/logs/traces, 各版本支持深度不同, 指标侧靠 MAL 规则映射成原生仪表盘最成熟), SkyWalking Java Agent 也能把数据导出给 OTLP 后端, 形成"埋点与后端解耦"。趋势判断: 新项目埋点优先 OTel SDK 对冲供应商锁定, 但深度诊断、拓扑与告警仍由后端(SkyWalking/商业 APM)提供; 在 SkyWalking 生态里, sw8 探针路线与 OTLP 路线可混用, 代价是两套传播头在边界服务处会断, 迁移要统一节奏。

## 2.11 OAP 如何部署高可用？扛不住量了怎么扩？

OAP 无状态, 部署 2 节点以上注册进同一集群(cluster 模块支持 zookeeper/k8s/etcd/consul/nacos 发现), Agent 端 gRPC 长连接负载均衡到多个节点, 节点故障自动切换重连; 前端用 LB/LVS/K8s Service 暴露 11800(gRPC)/12800(HTTP+GraphQL), UI 无状态可随意扩容, 全系统唯一的有状态点是存储, 高可用靠存储集群自身(ES 3 节点、BanyanDB 集群)与副本机制。扩容顺序有讲究: 先看瓶颈画像——存储 bulk 拒绝写则先扩存储(ES 加节点/调 bulkActions/concurrentRequests), OAP CPU 饱和或队列(buffer)常态积压才加 OAP 节点; 单机 8C16G 的经验吞吐约 1~2 万 Segment/s, 超过后比堆机器更优的是水平拆分: 用 core.role=Receiver/Aggregator 做角色分离, 或按数据类型(Trace+Log 一集群、Metrics 一集群)/业务域/环境拆多个 OAP 集群, 故障隔离收益通常大于容量收益。收尾一句: 观测平台自身必须被观测, telemetry 模块开 Prometheus 导出(队列积压/GC/写失败三类告警必配), 否则失明最被动。

## 2.12 你们生产上接入 SkyWalking 踩过什么坑？（开放题）

答三到四个有细节的坑比背概念有效, 参考弹药: ①服务命名——多个不相关服务共用一个 agent.service_name 导致拓扑假自环, 根治是把命名规范(域-应用, 环境进 namespace)写进 CI 准入; ②K8s 注入——initContainer 拷贝目录层级与 -javaagent 路径差一层, Agent 静默不加载, 验收要看 agent 日志 banner+UI 双证; ③端点爆炸——RESTful 数字路径未归一化, 一个接口上万端点, ES 聚合超时+指标膨胀, 用 @PathVariable 模板/归组规则修复; ④升级顺序——先升 Agent 后升 OAP 导致新字段被老 OAP 丢弃, 从此固化"先 OAP 后 Agent"; ⑤大 payload 反模式——往 tag 塞请求体导致 Segment 变 MB 级, 改为"tag 放摘要, 大内容进日志用 TraceId 关联"。每题结尾补一句"现在我们的规范是……", 体现治理闭环而不是单纯踩坑史。
