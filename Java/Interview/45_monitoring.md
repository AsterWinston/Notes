# 1. 知识点
- **三大支柱**：Logs（日志记录离散事件）、Metrics（度量随时间聚合数值趋势）、Traces（分布式链路跟踪单个请求端到端流转），三者相互关联补全全局视图
- **日志(Logs)**：记录应用运行时的离散事件、错误详情、调试信息，适合排查特定问题
- **指标(Metrics)**：聚合的数值数据，如 QPS、CPU 使用率、错误率、P99 延迟，适合看趋势和告警
- **分布式链路跟踪(Traces)**：追踪一个请求在分布式系统中经过哪些服务/方法，耗时如何分布，适合定位性能瓶颈
- **三者关联**：Logs 中注入 traceId 关联 Traces，Traces 聚合生成 Metrics，形成完整可观测性地图
- **Prometheus**：CNCF 毕业的开源监控系统，时序数据库(TSDB)，Pull 模式主动拉取指标
- **时序数据库(TSDB)**：Prometheus 自带的时序数据库，按时间序列存储数据（指标名 + 标签键值对），高效压缩和查询
- **Pull 拉取模式**：Prometheus Server 周期性地主动从 Target 拉取 Metrics 数据
- **scrape**：Prometheus 一次拉取指标的操作，scrape_interval 默认 15 秒
- **target**：被 Prometheus 抓取的监控目标，可以是应用实例（暴露 /metrics 端点）或 Exporter
- **/metrics HTTP端点**：目标应用通过 HTTP 接口以文本格式暴露 Prometheus 指标数据
- **Prometheus架构**：Server（抓取+存储）→ Exporter（暴露指标）→ Alertmanager（告警管理）→ Grafana（可视化）
- **Exporter采集器**：将被监控目标的数据转换为 Prometheus 格式并暴露，如 node_exporter（主机指标）、mysql_exporter（数据库指标）、JMX Exporter（JVM 指标）
- **Alertmanager**：Prometheus 生态的告警管理器，处理报警（分组、抑制、静默、路由），发送通知到邮件/钉钉/飞书/Webhook
- **PushGateway**：允许短期任务（Job/CronJob）主动推送指标（Push 方式），Prometheus 再从 PushGateway 拉取
- **Grafana**：开源数据可视化和仪表板工具，从多种数据源获取数据生成图表
- **Dashboard面板**：Grafana 中的仪表面板，可按 Row、Panel 方式展示多个图表和指标
- **Data Source**：Grafana 的数据来源，支持 Prometheus、Elasticsearch、InfluxDB、MySQL 等多种数据库
- **Alert告警规则**：Grafana 可基于面板数据定义告警规则，触发后通过通知渠道告警，但主要告警推荐用 Prometheus Alertmanager
- **ELK/S**：Elasticsearch + Logstash/Filebeat + Kibana，广泛使用的日志收集分析展示技术栈
- **Elasticsearch**：分布式搜索和分析引擎，存储日志并提供全文检索和聚合
- **Logstash**：服务器端日志处理管道，接收/转换/发送日志数据，功能强但资源消耗较高
- **Filebeat**：轻量级日志采集器，部署在应用服务器上读取日志文件，发送到 Logstash/ES/Kafka
- **Kibana**：基于 ES 的可视化和分析界面，探索日志、创建可视化、设置告警
- **日志收集架构(Filebeat→Kafka→Logstash→ES→Kibana)**：Filebeat 采集日志 → Kafka 缓冲削峰 → Logstash 解析清洗 → Elasticsearch 存储索引 → Kibana 查询可视化
- **Fluentd**：开源数据收集器，统一数据采集和消费，支持 500+ 插件，替代 Logstash 的选择之一
- **SkyWalking**：Apache 开源的应用性能监控(APM)系统，分布式链路追踪和可观测性平台
- **Java Agent探针**：SkyWalking 基于字节码增强(ByteBuddy)，无侵入式自动探针，应用无需修改代码
- **服务拓扑图**：SkyWalking 自动发现服务间调用关系，生成拓扑图，直观展示服务依赖和健康状态
- **调用链路追踪**：SkyWalking 提供方法级别的细粒度调用链，包含执行耗时、参数、SQL
- **慢SQL检测**：自动识别执行时间超过阈值的 SQL 语句，帮助定位数据库性能瓶颈
- **Trace+Log联合**：SkyWalking 可将 Trace ID 注入日志(通过 gRPC 上报日志或集成 logback)，实现 Trace 和 Log 一键跳转
- **链路追踪原理**：一个完整请求链路通过 TraceId + Span 组织，核心要素 TraceId/SpanId/ParentSpanId
- **TraceId**：一次完整请求在整个分布式调用链中全局唯一的标识，贯穿所有调用
- **SpanId**：一次调用的唯一标识，每个 Span 代表链路上的一个独立工作单元（如一次 RPC 调用）
- **ParentSpanId**：父 Span 的 ID，标识调用的层级关系和调用链构造
- **MDC传递traceId**：在 logback/log4j 中使用 MDC(correlation map)存储 traceId，子线程/跨服务可以打印 %X{traceId} 打通日志与链路
- **Spring Cloud Sleuth**：Spring Cloud 的分布式链路跟踪方案，已停更官方建议迁移到 Micrometer Tracing（Micrometer Tracing 是 Sleuth 的继任者）
- **Micrometer Tracing**：Spring 生态新集成标准，提供观测门面(facade)，支持 Brave+Zipkin 或 OpenTelemetry+Jaeger 后端
- **告警体系**：定义规则、通知渠道、分级策略，保障问题及时响应和不被打扰
- **Prometheus告警规则**：使用 PromQL 定义 Recording Rules 和 Alerting Rules
- **告警通知**：邮件、钉钉机器人、飞书机器人、Webhook（接入自建或第三方通知）
- **告警分级**：P0(紧急)立即处理 → P1(严重)30分内 → P2(一般)1小时 → P3(提示)次日
- **告警降噪/收敛**：同一问题连续触发不重复通知，merge 成一条聚合告警，避免告警疲劳
- **告警升级(Alert Escalation)**：低级别告警超时未处理自动升级为高级别通知，扩大通知范围
- **JVM指标监控**：通过 Prometheus JMX Exporter 或 Spring Boot Actuator + Micrometer 暴露 JVM 指标（堆内存、GC、线程数、类加载等）
- **Actuator + Micrometer**：Spring Boot 应用添加 micrometer-registry-prometheus 依赖，自动暴露 `/actuator/prometheus` 端点
- **APM对比**：Pinpoint（Naver，韩国Naver出品粒度极细）、CAT（美团开源实时监控系统）、SkyWalking（Apache项目、生态最活跃）、Zipkin（最早开源调用链工具、Twitter）
- **SLI(Service Level Indicator)**：服务水平指标，衡量服务质量的量化值，如可用性99.9%、P99延迟<200ms
- **SLO(Service Level Objective)**：服务水平目标，基于 SLI 设定的目标值，如月度可用性≥99.95%
- **SLA(Service Level Agreement)**：服务等级协议，与用户签订的法律/合同承诺，达不到可能涉及赔偿
- **MDC (Mapped Diagnostic Context)**: 日志框架(Logback/Log4j2)提供的线程级诊断上下文，基于ThreadLocal存储traceId/userId等。在Filter/拦截器中设置MDC，后续所有日志自动携带(%X{traceId}模式)。分布式链路追踪的基础
- **全局链路跟踪流程**: (1) 前端Axios请求拦截器生成traceId发送至Nginx; (2) Gateway检查Header中是否有traceId，若无则生成并加入请求头; (3) 后端TraceIdFilter检查traceId，生成/获取后放入MDC; (4) 调用链路中所有服务日志自动携带同一traceId; (5) ELK收集日志，Kibana可视化完整调用链
- **慢接口追踪**: (1) 日志记录: 在关键节点记录请求参数、响应时间、SQL执行时间; (2) SkyWalking/分布式链路追踪: 跟踪完整调用链和每段方法性能指标; (3) Arthas trace: 实时追踪方法调用路径和各步耗时

# 2. 面试题

## 2.1 什么是可观测性的三大支柱？
- **Logs（日志）**：记录应用产生的事件信息，时间点明确，回答"发生了什么"，适合排障
- **Metrics（指标）**：某段时间内聚合的数值测量，回答"趋势如何"，适合看全局健康、设置告警
- **Traces（链路追踪）**：记录单个请求完整的调用过程和耗时分解，回答"为什么慢"

三者结合形成全栈可观测性：问题告警（Metics）→ 查看调用链（Trace）→ 查看具体日志详情（Log），TraceId 将三者串联。

## 2.2 Prometheus 的架构是怎样的？Pull 模式和 Push 模式有什么区别？
**架构**：
Prometheus Server(抓取+存储) → Service Discovery(自动发现 Target) → Exporter/JMX/应用(暴露 /metrics) → PromQL查询 → Grafana(可视化) / Alertmanager(告警)

**Pull vs Push**：
- **Pull（拉取，Prometheus 默认）**：Server 定时主动去 Target 拉数据。优点：不需要应用主动上报、Server 控制抓取频率、易检测目标是否存活；缺点：需服务发现、短生命周期任务数据丢失
- **Push（推送）**：应用主动推送数据到 PushGateway/Server。适合短生命周期任务（Job/CronJob）、网络隔离（无法被 Prometheus 访问到）

Prometheus 通过 PushGateway 变相支持 Push 模式。

## 2.3 Prometheus 的四种数据类型是什么？
1. **Counter（计数器）**：只增不减的累计计量（如请求总数、错误总数），可通过 rate() 求 QPS
2. **Gauge（仪表盘）**：可增可减的瞬时值（如 CPU 使用率、内存、连接数、温度）
3. **Histogram（直方图）**：对数据分段统计分布，自动提供分位数（如 P50/P90/P99 延迟、请求体大小分布）
4. **Summary（摘要）**：直接在客户端计算分位数，比 Histogram 精度高但不可聚合（多实例无法合成为汇总分位数）

常用选择：普通分布用 Histogram，需要精确分位数且单实例不聚合的场景用 Summary。

## 2.4 常用的 PromQL 查询有哪些？
```promql
# HTTP 请求 QPS（每秒请求数）
rate(http_requests_total[5m])

# CPU 使用率
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用百分比
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# P99 延迟
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 错误率
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# 空闲磁盘空间百分比
node_filesystem_avail_bytes / node_filesystem_size_bytes * 100 < 20
```

## 2.5 ELK 日志收集的整体架构是怎样的？
```
应用服务器(Filebeat 轻量采集) → Kafka(削峰缓冲) → Logstash(清洗解析增强) → Elasticsearch(存储索引) → Kibana(可视化查询)
```

- **Filebeat**：部署在各应用服务器上，轻量级（约 30MB 内存），读日志文件 → 发送
- **Kafka**：高吞吐消息队列，削峰填谷，写入速率平滑到下游、避免 Logstash 被打爆
- **Logstash**：重量级 ETL 工具，将日志解析为结构化 JSON，富化（GeoIP、UserAgent）、过滤
- **Elasticsearch**：存储结构化日志，高效搜索和聚合
- **Kibana**：在 ES 基础上提供仪表板和搜索 UI

简化方案可去掉 Kafka 和 Logstash，直接 Filebeat → ES → Kibana，或使用 Fluentd 替代 Logstash。

## 2.6 Filebeat 和 Logstash 的区别是什么？
| 维度 | Filebeat | Logstash |
|------|----------|----------|
| 定位 | 轻量日志采集器 | 重量服务端管道 |
| 资源占用 | 低（约30MB内存） | 高（约500MB+ JVM） |
| 功能 | 简单收割/转发 | 复杂ETL解析/富化 |
| 典型位置 | 每台应用服务器部署 | 集中式或分层 |
| 适用 | 采集日志发送 | 解析日志、数据清洗加工 |

最佳实践：Filebeat 采集 + Logstash 加工，各取所长。

## 2.7 SkyWalking 相比 Zipkin 有什么优势？
| 维度 | SkyWalking | Zipkin |
|------|-----------|--------|
| 侵入性 | Java Agent 无侵入探针 | 需代码集成 Brave SDK |
| 语言支持 | Java/.NET/Go/Node.js/Python | 多语言 SDK |
| 可视化 | 内置丰富的仪表板 UI | 基本 UI，依赖外部 |
| 复杂度分析 | 自动生成服务拓扑、慢 SQL | 较简单的调用链 |
| 存储后端 | ES/H2/MySQL/BanyanDB | ES/MySQL/Cassandra |
| 性能开销 | 较低（agent 层） | 中等 |
| 生态融合 | Trace+Log+Metrics 统一 | 仅 Trace |

SkyWalking 适合 Java 技术栈追求无侵入开箱即用，Zipkin 适合多语言轻量集成。

## 2.8 链路追踪的核心原理是什么？
任务链路追踪的核心是一棵树结构。当一个请求到达：
1. 生成一个全局唯一的 **TraceId**，标识整个请求链路
2. 每进入一个新服务或关键操作，生成一个 **SpanId**，表示一个独立的工作单元
3. 当前 Span 记录 **ParentSpanId**（上层调用的 SpanId），构建父子关系
4. 所有 Span 携带相同的 TraceId，传递到下一个被调用方

跨线程和跨服务的传递机制：通过 HTTP Header（如 `b3: xxxx`）或 RPC Metadata 将 TraceId 和 SpanId 注入到下游服务。SkyWalking 使用 gRPC 传输 Trace 到后端 Collector。

## 2.9 如何将 TraceId 注入到日志中？
```java
// Java 日志示例 - logback MDC
import org.slf4j.MDC;

// 在 Filter 或 Interceptor 中
String traceId = request.getHeader("X-Trace-Id"); // 从 SkyWalking 获取
if (traceId == null) traceId = UUID.randomUUID().toString();
MDC.put("traceId", traceId);

// logback-spring.xml
<pattern>%d %p [%X{traceId}] %t %c - %m%n</pattern>
```
所有日志输出自动带上 `[traceId:xxxx]`，便于在日志系统（ELK/Kibana）中基于 TraceId 搜索同一请求的所有日志。

## 2.10 如何设计一个合理的告警体系？
1. **告警规则**：基于真实的 SLA/SLI 设定阈值，使用 PrometheusRule 定义，避免太多低价值告警
2. **告警分级**：
   - P0（紧急）：服务完全不可用、数据丢失，立即电话/短信通知
   - P1（严重）：核心功能受损但不完全宕机，30 分钟内处理
   - P2（警告）：性能下降、磁盘将满，1 小时内处理
   - P3（通知）：预发布/审核信息，工作时间处理
3. **告警抑制/降噪**：同一设备/实例的相同告警聚合为一条，不重复通知
4. **告警收敛**：先发一次通知，若问题持续再逐渐升级
5. **告警升级**：30 分钟未处理 P1 → 提升为 P0 通知更广人群
6. **通知渠道**：邮件 + 钉钉/飞书/微信群机器人 + on-call 电话轮转
7. **告警治理**：定期回顾告警，清理假阳性告警、大盘不可用告警优化

## 2.11 什么是 SLI、SLO、SLA？它们的关系是什么？
- **SLI（服务等级指标）**：量化服务质量的可测量指标，如"99.9% 请求在 200ms 内完成"、"可用性 99.95%"
- **SLO（服务等级目标）**：基于 SLI 设定的服务目标值，例如"99.9%的月度可用性"
- **SLA（服务等级协议）**：向用户/客户承诺的合同条款，基于 SLO，达不成可能赔偿/退款

关系：SLI(底层) → SLO(目标) → SLA(合同)。例如：SLI = 测量到 99.99%可用性，SLO = 目标不低于 99.9%，SLA = 承诺客户 99.9%，若低于需赔付。

## 2.12 如何用 Prometheus 监控 JVM 指标？
**方案一：Spring Boot Actuator + Micrometer（推荐）**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
Actuator 自动暴露 `/actuator/prometheus` 端点，包含 JVM 指标（堆内存、GC、线程数等）。

**方案二：JMX Exporter**
下载 JMX Exporter JAR + 编写 config.yml，作为 Java Agent 启动：
```bash
java -javaagent:jmx_prometheus_javaagent-0.20.0.jar=8080:config.yml -jar app.jar
```

**Prometheus 配置抓取**：
```yaml
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
```

## 2.13 日志采集方案有哪些？（ELK 的替代方案）
1. **ELK Stack**：Filebeat + Elasticsearch + Logstash + Kibana，行业标准
2. **EFK Stack**：Fluentd 替代 Logstash，Fluentd 轻量且插件化
3. **Loki + Grafana**：轻量级日志方案，Loki 是 Prometheus 风格的日志聚合系统，完美集成 Grafana
4. **PLG Stack**：Promtail(采集) + Loki(存储) + Grafana(展示)，资源占用远低于 ELK
5. **SkyWalking + ES**：SkyWalking 自带的日志上报能力，与 Trace 关联展示
6. **阿里云 SLS / 腾讯云 CLS**：云厂商托管日志服务，免运维

## 2.14 什么是MDC？在分布式链路追踪中如何使用？
MDC(Mapped Diagnostic Context)是Logback/Log4j2提供的线程级诊断上下文，基于ThreadLocal实现，用于在当前线程的日志中自动携带上下文信息(如traceId、userId、requestId等)。

**使用方式**：
1. 在Filter或拦截器中从请求头获取或生成traceId
2. `MDC.put("traceId", traceId)` 将traceId放入MDC
3. 在logback.xml中配置pattern：`[%X{traceId}]`，后续所有日志自动携带traceId
4. 请求结束时调用`MDC.clear()`防止内存泄漏

**分布式链路追踪中的应用**：跨服务传递时，从上游服务请求头(如`X-Trace-Id`)中提取traceId并放入MDC，使得同一TraceId串联整个调用链路的所有日志。与SkyWalking等APM工具配合，将TraceId注入MDC后，可在Kibana中通过TraceId一键搜索请求的所有日志，实现Log与Trace的联动。

## 2.15 全局链路跟踪的完整流程是怎样的？
**完整流程**：
1. **前端生成**：Axios请求拦截器使用UUID生成traceId，通过Header(`X-Trace-Id`)随请求发送至Nginx
2. **Gateway检查**：Spring Cloud Gateway全局过滤器检查请求Header中是否存在traceId，若不存在则生成新的traceId并加入请求头(防止前端未传的场景)
3. **后端TraceIdFilter**：在各微服务中，自定义TraceIdFilter作为第一个Filter执行，检查请求Header中的traceId，若存在则直接获取，不存在则生成；将traceId放入MDC(`MDC.put("traceId", traceId)`)
4. **日志自动携带**：通过logback配置`%X{traceId}`模式，调用链路中所有服务的日志自动携带同一traceId
5. **ELK收集展示**：Filebeat采集日志→Kafka缓冲→Logstash解析→ES存储→Kibana可视化。在Kibana中通过traceId可搜索完整调用链路的所有日志

**跨服务传递**：使用Feign拦截器/Interceptor在发起远程调用时自动将traceId注入请求头，下游服务TraceIdFilter提取后放入MDC，保证全链路traceId一致。

## 2.16 慢接口怎么追踪定位？
**三种方式逐层深入**：

1. **日志记录**：
   - AOP切面记录Controller方法的请求参数和响应时间
   - MyBatis拦截器记录SQL执行时间
   - 通过日志可快速发现哪个方法/哪条SQL耗时最长

2. **SkyWalking分布式链路追踪**：
   - 无侵入Java Agent自动采集方法级调用链
   - 展示完整调用链拓扑图和每个Span的耗时
   - 自动检测慢SQL(超过阈值的SQL语句高亮)
   - 服务拓扑图展示各服务的响应时间，快速定位是哪个下游服务慢

3. **Arthas在线诊断**：
   - `trace`命令：实时追踪方法调用路径和各步耗时(`trace com.xxx.Service methodName`)
   - `watch`命令：观察方法入参和返回值
   - `monitor`命令：统计方法调用次数、成功率、平均耗时
   - 优势：无需重启，生产环境实时诊断
