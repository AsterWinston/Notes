
# 1. 知识点

- **管道模型** — Input Plugin → Queue → Filter Chain → Output Plugin; Queue 是解耦点, 队列满阻塞 Input 形成背压, 压力沿 Filebeat/Kafka 向上传导
- **线程模型** — Input 线程 + N 个 filter worker(pipeline.workers, 默认=核数); worker 是"批"的并行单位, 批内串行、批间并行; batch.size=125 默认, batch.delay=凑批最大等待 ms
- **Persistent Queue** — queue.type: persisted; 事件序列化追加写 Page File(64MB/页), Checkpoint 记录已确认水位线(默认每 1024 次写或 1000ms); 重启从 checkpoint 恢复; at-least-once
- **DLQ** — dead_letter_queue.enable: true; Output 永久失败(如 mapping 冲突 400)的事件落盘; 依赖 PQ 开启; 读回用 input { dead_letter_queue { commit_offsets => true } }; PQ 防进程死, DLQ 防数据坏
- **Grok vs Dissect** — grok=正则宏语言(可回溯, 慢), dissect=分隔符定位(快 5-10x, 要求格式固定); grok 必须 ^ 锚定, 多候选写进一个 match 数组命中即停
- **Filebeat/Logstash/Kafka 分工** — Filebeat 端侧轻量采集(registry 续传); Kafka 削峰+多消费+回放; Logstash 中心重加工(模板/ILM/DLQ 与 ES 联动最深)
- **8.x 关键变化** — Java Execution Engine(核心插件 Ruby→Java, JRuby 宿主与 DSL 不变); 默认 ECS 兼容; 默认开启安全; ssl_enabled/ssl_verification_mode 取代旧 ssl/cacert 参数
- **_node/stats 定位法** — 差分 events.in/out 找背离管道; queue.events_count 持续涨=下游慢; plugins.filters[].events.each 指认热点 filter; hot_threads API 抓线程栈
- **ES 429 应对** — es_rejected_execution_exception 瓶颈在 ES: 加 Logstash 并发是反模式; ES 侧调 refresh_interval/副本/分片, LS 侧靠 output retry+PQ/Kafka 缓冲
- **数据不丢四件套** — Filebeat registry / Kafka offset(上游), PQ(在途), output retry+DLQ(写出), ES _id 幂等(防重放重复)

# 2. 面试题

## 2.1 Logstash 是什么？在 ELK 中扮演什么角色？

Logstash 是 ELK 中的**数据收集和处理引擎**，承担 ETL 角色：Input 从多种来源采集（Beats、Kafka、JDBC、文件、Syslog），Filter 做解析与丰富（Grok 切字段、GeoIP 补地理、Date 对齐时间戳、Mutate 清洗），Output 写入目标（ES、Kafka、S3）。它运行在 JVM 上、用声明式 Ruby DSL 描述管道，是 Elastic Stack 里唯一"重加工"的位置——Filebeat 把日志廉价搬出来，ES 负责存查，**"把一条 Nginx access log 变成可检索的结构化文档"归 Logstash**。要补一句边界：日志已结构化、不需加工时它常被跳过（Filebeat 直连 ES + ingest pipeline），L 是为复杂度服务的。

## 2.2 Logstash 的管道模型和线程模型是怎样的？

```text
Input Plugin → Queue → Filter Chain → Output Plugin

Input: 从数据源采集 (Beats/Kafka/File/JDBC)
Queue: Memory Queue 或 Persistent Queue 缓冲
Filter: 解析和转换 (Grok/Mutate/GeoIP/Date)
Output: 写入目标 (ES/Kafka/File/S3)
```

线程层面：**Input 插件自带线程**把事件写入 Queue；**`pipeline.workers` 个 worker 线程**各自从队列取一批（`pipeline.batch.size` 条，最多等 `pipeline.batch.delay` 毫秒凑批），批内事件顺序流过 filter 链、批尾等 Output 确认——**worker 是"批的并行单位"，同批内串行、批间并行**，因此批内不保序，需要严格有序用 `pipeline.ordered: true`。workers 默认=CPU 核数，IO-bound 可超配（线程多在等 ES），CPU-bound 超核数只是上下文切换损耗。一个进程可跑多条管道（pipelines.yml），各管道独立 workers/queue/reload 开关，但**共享同一个 JVM 堆**——拆管道隔离的是算力和队列，不是内存。

## 2.3 Grok 和 Dissect 的区别？Grok 为什么慢，怎么优化？

| 维度 | Grok | Dissect |
|------|------|---------|
| 原理 | 正则表达式（命名模式库拼装） | 分隔符定位（按字面标记切位置） |
| 性能 | 中（回溯开销） | 高（5-10x faster） |
| 适用 | 复杂/不规则日志 | 分隔符明确的日志 |

Grok 的慢不在"正则引擎慢"，而在**模式展开后的规模**：`%{COMBINEDAPACHELOG}` 是几十条子模式的合并，匹配失败要回溯整串，无锚定时还会在每个位置试一次。优化四板斧：`^` 锚定（收益最大）；多个候选模式合进**一个** match 数组（顺序即优先级、命中即停），别串行叠多个 grok 块；少用 `%{GREEDYDATA}` 兜底；格式固定的日志直接换 Dissect。选择线是"格式有没有契约"——自家服务用 Dissect，第三方漂移格式用锚定 Grok，还能 dissect 打底 + tag_on_failure 交 grok 兜底。

## 2.4 Persistent Queue 是如何工作的？和 DLQ 是什么关系？

PQ 把 Input 与 Filter 之间的缓冲从 JVM 堆搬到磁盘：事件**序列化后追加写入 Page File**（默认 64MB 一页），worker 从最老的页按批消费，Output 确认后由 **Checkpoint 推进"已完整处理"的序列号水位线**（默认每 1024 次写入或 1000ms 触发，先到先算）；崩溃重启从位点继续，未确认批次**重放**。语义是 **at-least-once**：丢失窗口=最后一个 checkpoint 周期，重复窗口=重放批次，下游需幂等（ES 写 `_id`）。容量由 `queue.max_bytes` 封顶，写满**阻塞 Input 形成背压**而非丢弃。与 DLQ 的分工一句话：**PQ 防"进程死"（在途事件丢），DLQ 防"数据坏"（写不进 ES 的毒数据被扔）**——mapping 冲突这类 400 永久失败重试无望，落盘 DLQ 让管道继续走，事后用 `input { dead_letter_queue {} }` 修复重放；且 DLQ 依赖 `queue.type: persisted` 才生效。

## 2.5 如何解析 Nginx 日志？

```ruby
input { beats { port => 5044 } }

filter {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  date { match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ] }
  geoip { source => "clientip" }
  mutate { remove_field => ["message", "timestamp"] }
}

output { elasticsearch { hosts => ["http://es:9200"] } }
```

四步固定套路：**grok 切字段 → date 把日志时间提升为 `@timestamp`（不配的话 ES 里全是摄取时间，排障时间线全错）→ geoip 富化 → mutate 清理**。实践加分点：grok 加 `^` 锚定、`_grokparsefailure` 时保留原文进隔离索引；自定义 log_format 就写 patterns 目录自定义模式；8.x 生产要带 TLS+认证写 ES；上线前 `--config.test_and_exit` 校验。

> **注意**：Nginx 若用 JSON access log 格式，这整段 grok/geoip 前的解析可换成 `json { source => "message" }`，甚至让 Filebeat 侧直接解析、Logstash 只做富化——解析下沉得越早越省 CPU。

## 2.6 Logstash 性能优化怎么做？

1)先用 `_node/stats` 定位瓶颈段而不是盲调参数——CPU 100% 且队列堆积=filter 慢，CPU 空闲但堆积=Output 慢；2)Grok 锚定、候选合并、可换 Dissect、删 ruby filter；3)workers=核数起步（IO-bound 至 2 倍），batch.size 提到 250 并按 `batch×workers×事件大小` 核算堆；4)ES output 开 `http_compression`、关 `sniffing`、`manage_template => false`；5)JVM：G1、`-Xms=-Xmx`（别超 31g 压缩指针悬崖）；6)生产 PQ+DLQ、SSD 放队列。反面要点也要说：batch/workers 不是越大越好（内存风险、ES 429），瓶颈在 ES 时加并发只会制造更多拒绝。

## 2.7 Logstash 与 Filebeat、Kafka 分别怎么分工？

- Filebeat：Go 写的端侧 agent（30-50MB），只读文件+registry 续传+转发，几乎不加工；日志已结构化时可直接发 ES，省掉 L
- Logstash：JVM 中心 ETL，重加工（Grok/GeoIP/聚合）与 ES 生态联动最深（模板、ILM、DLQ），不该装在每台业务机
- Kafka：补 Logstash 两个短板——**缓冲容量**（PQ 受单机磁盘限制，Kafka 分区 TB 级削峰）与**多消费**（同一份日志同时喂 ES、风控流、数仓，可回放）；Logstash 消费端按分区数横向扩容，实例数≤分区数

推荐链路：Filebeat →（量大或需回放再插 Kafka）→ Logstash → ES。追问"为什么不全装 Logstash"答：N 台机器 × 1GB JVM 的成本与业务机侵入性；追问"什么时候别上 Kafka"答：万级 eps 以内、单消费方，registry+PQ 两段持久化已够，别为仪式感加集群。

> **行业实践**：审计/计费类链路"数据不丢"靠三段接力——Filebeat registry（端）、Kafka offset（中）、Logstash PQ+DLQ（心），再配 ES `_id` 幂等（防重放）；缺任何一段都只能保证"尽力而为"。

## 2.8 Logstash 如何保证数据不丢失？

按链路分段说责任：1)**上游段**——Filebeat registry（确认交付前偏移不前进）或 Kafka offset（交付后仍可回放）；2)**在途段**——Persistent Queue，进程崩溃/重启后从 checkpoint 恢复未处理事件；3)**写出段**——elasticsearch output 指数退避重试，永久失败进 **DLQ** 落盘事后重放，不因毒数据丢整批；4)**恢复段**——Kafka Input 的 consumer group offset 管理。要主动讲代价：整套组合是 **at-least-once 而非 exactly-once**，重放必产生重复，ES 写入要用业务键 `_id` 幂等；PQ/DLQ 磁盘本身是新单点，合规场景保留"重放上游原始日志"预案。

## 2.9 什么场景用 Pipeline-to-Pipeline？

Pipeline-to-Pipeline 允许同一 Logstash 进程内管道间零序列化传数据（源管道 `output { pipeline { send_to } }`，目标管道 `input { pipeline { address } }`）：1)同一机器上隔离不同类型数据处理（Nginx/App 独立管道、独立 workers）；2)预处理→分类→多下游（`send_to` 多管道名自动负载均衡，多个 pipeline output 块则广播复制）；3)资源隔离（高优先管道独立 worker 配额）。判断线是**隔离需求在进程内还是跨进程**：同进程内省掉网络+序列化；但跨机器、要独立扩缩容与故障域时，老实拆部署走 Kafka。限制要报出来：address 全局唯一、链路不持久化（各管道自己的 PQ 仍在）、无环检测、geoip 等资源按管道各加载一份。

## 2.10 Logstash vs Fluentd/Fluent Bit/Vector 怎么选？

| 场景 | 推荐 |
|------|------|
| ELK 完整生态，复杂 ETL | Logstash |
| 通用数据管道，多输出目标 | Fluentd |
| K8s 日志采集，边缘设备 | Fluent Bit |
| 极致性能，Rust 技术栈 | Vector |

差异一句话：**面向的目标系统广度与资源模型**。Logstash 200+ 插件围绕 Elastic 最深但 JVM 500MB+ 起步、重加工吞吐 2K-8K eps；Fluentd 1000+ 社区插件、按 chunk 的 buffer/retry 语义更精细、S3/GCS/Mongo 等输出更全；Fluent Bit ~5MB 做端侧；Vector 用 Rust+VRL 把"采集+加工"合到一个高吞吐工具里。混合架构常见：边缘 Fluent Bit/Filebeat → 中心 Logstash，各取所长。加分点是提团队栈匹配——配置 DSL/插件语言没人懂，指标再漂亮也会在事故之夜还债。

## 2.11 ES 返回 429 拒绝写入，Logstash 侧怎么应对？

先定性：**429（`es_rejected_execution_exception`）是 ES 写入线程池队列满，瓶颈在 ES 不在 Logstash**，加大 workers/batch 只会制造更多拒绝。处置顺序：1)确认 elasticsearch output 的指数退避重试在工作（`bulk_requests.retries` 在涨说明没丢，只是在等），靠 PQ/Kafka 吸收背压，**不要**关重试硬灌；2)ES 侧体检——refresh_interval 调大（灌数据期 30s/-1）、副本临时降 0、查热点 shard、确认 disk watermark 没触发索引只读锁、mapping 是否字段爆炸；3)架构侧回 Logstash 收字段（删 agent/ecs 冗余树减 bulk 体积）。恢复判据：retries 增量回落为 0、`queue.events_count` 开始下降。

## 2.12 线上 Logstash 变慢、队列持续堆积，怎么排查？

按"**先分段、再看插件、后看资源**"：1)`curl localhost:9600/_node/stats/pipelines` 取两次差分，确认哪条管道 `events.in` 与 `events.out` 背离、`queue.events_count` 在涨；2)看 `plugins.filters[].events.each` 找单事件耗时最高的 filter——多数事故是新加的低质 grok 或误用 ruby；3)each 都不高但吞吐低，看 outputs 的 bulk 延迟与 retries，转 ES 侧排查（429/慢查询/red）；4)CPU 100% 用 `_node/hot_threads?human=true` 抓线程栈落到具体插件；CPU 空闲看 `_node/stats/jvm` 的 GC 时间与 heap——锯齿陡峭说明堆小了或 batch 过大；5)每次只改一个变量压测验证（generator 灌样本、差分算 eps）。收尾查数据面：`events.filtered` 与 in 是否一致（没被意外过滤）、条件分支是否永远为假导致全量走兜底。

> **注意**：排查全程只读指标不改参数——"看到 CPU 有余量就加 workers"是最常见的二次事故来源；改参数后的验证必须单变量 + 差分对比，否则无法归因。
