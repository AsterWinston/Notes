
# 1. OAP 的性能模型

OAP 是一条典型的**流式处理管道**：gRPC/HTTP 接收 → 反序列化 → 分析（Segment 解析、记录转换）→ L1/L2 聚合 → 批量写存储。它的吞吐上限不由单点决定，而由**四个池子的容量与队列**共同决定：gRPC 接收线程池、分析/聚合 Worker 池、超时与流聚合窗口、存储执行池（bulk 队列）。调优的思路不是背参数，而是先定位瓶颈在哪个池子——CPU 打满在哪个线程组、哪条队列在堆积、存储 bulk 是否拒写。

OAP 的一个关键设计是**内存优先、批量落盘**：绝大多数聚合在内存窗口完成，只有聚合结果和原始记录（Trace/Log）才进存储，所以**存储写入往往是第一瓶颈**，其次是分析线程的 CPU。理解了这一点，下面所有参数都是在"喂饱管道、别在队列处堵死"之间做平衡。

> **注意**：9.x/10.x 的 `application.yml` 与 6.x/7.x 时代相比变化很大——早期笔记里流传的 `maxAnalyzeCount`、`bufferPath/bufferFileSize` 等键在现版本中**已不存在或不在此位置**；本篇示例以 9.x/10.x 实际键名为准，历史键会明确标注。

# 2. JVM 参数

```bash
# OAP Server JVM 调优（9.x 要求 JDK 11/17，10.x 支持 Java 11/17/21）
# bin/oapService.sh 中设置 JAVA_OPTS
JAVA_OPTS="-Xms8g -Xmx8g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:InitiatingHeapOccupancyPercent=45 \
  -XX:+DisableExplicitGC"
```

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `-Xms/-Xmx` | 相等，4g~8g 起步 | 聚合缓冲与 bulk 队列都在堆内，堆大小决定能扛多深的队列 |
| GC 器 | G1GC（大堆可 ZGC） | OAP 是吞吐型负载，G1 兼顾；ZGC 换更低停顿但吃 CPU |
| `MaxGCPauseMillis` | 200 | 别追求过低，吞吐优先 |
| `InitiatingHeapOccupancyPercent` | 45 | 提前并发标记，避免聚合高峰触发并发标记不及时 |

> **注意**：OAP 生产事故里最典型的雪崩链路是 **Full GC 停顿 → gRPC 超时 → Agent 重连风暴 → 数据洪峰再次压垮**。`-Xms=-Xmx`、关闭显式 GC、IHOP 留提前量，本质都是在防止聚合窗口堆积触发老年代急涨；堆不是越大越好，超过约 32g 应优先考虑角色/集群拆分而不是继续加堆。

# 3. 接收线程与 buffer 队列

```yaml
# config/application.yml（9.x/10.x 实际键名节选）
core:
  default:
    gRPCHost: 0.0.0.0
    gRPCPort: 11800
    gRPCThreadPoolSize: -1                    # -1=自动(约 CPU 核数相关)，可显式调大
    maxConcurrentCallsPerConnection: 0        # 0=不限制；显式设置可防单连接打满线程池
    maxMessageSize: 52428800                  # 9.x 默认 10MB，10.x 默认提到 50MB
    restMaxThreads: 200                       # HTTP(12800) Jetty 线程上限
```

接收侧之后是**两级 buffer**：内存 ring buffer channel → 磁盘 buffer（`oap-server/buffer/` 目录），内存队列打满后溢写磁盘，磁盘也满则丢弃并打 overflow 日志。9.x/10.x 中 channel 大小与磁盘上限大多**按 JVM 内存自动计算**（`SW_CORE_GRPC_POOL_QUEUE_SIZE` 等键可覆盖，早期版本曾直接暴露 bufferPath 类键，历史笔记里的写法不再适用）。关键观测信号在日志里：

```text
# OAP 日志出现即说明该环节处理能力不足（类名随版本有差异）
WARN 2026-11-08 10:00:00,123 - ... buffer is full
WARN 2026-11-08 10:00:01,456 - ConsumerThread - Execute Batch Consumer failure
```

**设计考量**：加大内存队列能吸收瞬时洪峰（发布、告警风暴时的数据尖刺），但**治标不治本**——队列越深，数据可见延迟越大，OOM 风险越高。正确顺序是：先看是不是存储写不动（第 5 节），再看分析/聚合线程是否 CPU 饱和，最后才考虑队列深度。

# 4. 分析 Worker 与 L1/L2 聚合并发

各分析管道以 **Worker 实例**并行处理，实例数默认与 **CPU 核数挂钩**（约 `availableProcessors` 级别，版本与模块略有差异），9.x/10.x 的多数 receiver 不再暴露 `workerPoolSize` 这类显式键；真正影响聚合性能的是 `core.default` 里的窗口参数：

```yaml
core:
  default:
    # L1 内存窗口 flush 到 L2 的周期（毫秒）
    l1FlushPeriod: 500
    # L2 降采样粒度：在分钟级指标之上产出的层级
    downsampling:
      - Hour
      - Day
    # 存储会话超时（毫秒）：等待补齐数据的窗口
    storageSessionTimeout: 70000
    # 内存数据持久化周期（秒）
    persistentPeriod: 25
    # 准备写存储数据的线程数
    prepareThreads: 2
```

**L1 聚合在内存中把流式数据聚合到分钟窗口，L2 把分钟结果降采样成小时/天**（`downsampling` 决定层级）。聚合层级越多、窗口越大，内存占用越高但落盘记录越少；`prepareThreads` 是聚合结果转存储实体的并行度，数据量大时是最常被调的旋钮之一（默认 2 明显偏小）。压测时优先看两条曲线：worker 线程 CPU 饱和度、队列积压长度——分析产出速率高于存储消费速率时，加 worker 只会加深队列。

> **注意**：`prepareThreads`/`l1FlushPeriod` 的调整都会改变内存驻留量，动之前先确认堆余量；火焰图（OAP 自身可用 async-profiler 或 `receiver-profile` 抓）比猜参数高效得多。

# 5. 存储批次与写入吞吐

存储侧性能几乎完全由 **bulk 批量策略**决定，以 ES 为例（9.x/10.x 实际键名）：

```yaml
storage:
  selector: elasticsearch
  elasticsearch:
    clusterNodes: es1:9200,es2:9200,es3:9200
    bulkActions: 5000          # 单批条数
    batchOfBytes: 10485760     # 单批字节上限(10MB)，先到先刷
    flushInterval: 5           # 兜底刷新间隔(秒)
    concurrentRequests: 2      # 同时在途的 bulk 请求数
    indexShardsNumber: 2       # 新索引分片数
    indexReplicasNumber: 1     # 副本数
    superDatasetIndexShardsFactor: 5   # Trace 这类超大数据集的分片放大系数
```

三个参数互相制约：`bulkActions` 决定单批大小（批越大吞吐越高、单次延迟与失败代价越大），`concurrentRequests` 决定在途批次数（**真正的写入并发 ≈ bulk 线程 × 在途请求**），`flushInterval` 保证低流量时数据不滞留。MySQL/PostgreSQL 侧对应的是 `maxSizeOfBatchSql`（批量 SQL 上限）与 `asyncBatchPersistentPoolSize`（异步批量写入线程，默认 4）；BanyanDB 是 `maxBulkSize`/`flushInterval`/`concurrentWriteThreads`（默认 15）加 `metricsShardsNumber`/`superDatasetShardsFactor` 分片系数。

定位写入瓶颈的信号：OAP 日志 `Execute Batch Consumer failure`、ES 线程池 `write`/`bulk` rejected、`429 EsRejectedExecutionException`——出现任一信号，**优先扩存储而不是扩 OAP**。

# 6. 单机吞吐参考量级

以下是社区公开压测与生产经验的**量级参考**（非承诺值，受 Trace 复杂度、日志量、采样率、存储性能影响很大）：

```text
测试环境：OAP 单节点 8C16G，ES 3 节点(4C16G)，Java Agent 全采样

| Agent 实例 | 每秒 Segment | OAP CPU | 备注                   |
|-----------|-------------|---------|------------------------|
| 10        | ~1000       | 20%     | 轻松                   |
| 50        | ~5000       | 45%     | 队列基本不积压         |
| 100       | ~10000      | 70%     | GC 频率开始上升        |
| 200       | ~20000      | 90%+    | 需拆分/扩容，ES 压力显现|
```

经验口径：**单 OAP 节点（8C16G、ES 后端）大约稳在每秒 1~2 万 Segment、几千 QPS 的 Trace 流**；再往上靠水平扩展。日志按条计，量级更高但单条 CPU 成本低；JVM/Meter 指标体量小、频率恒定，通常不是瓶颈项。

# 7. 水平拆分：角色分离与按 workload 拆集群

OAP 无状态可以随便加副本，但 Mixed 角色下**每个节点都跑完整管道**（接收 + L1 + L2），副本加多了每节点内存浪费、聚合窗口也翻倍。9.x/10.x 提供的正式拆分手段有两层：

**第一层是角色分离**：`core.role` 支持 `Mixed/Receiver/Aggregator`——Receiver 角色只做接收与 L1 聚合，把分钟级结果转发给 Aggregator 角色节点做 L2 降采样与写存储：

```yaml
core:
  default:
    role: Receiver        # 接收组：扛 gRPC 与 L1
    # Aggregator 组: role: Aggregator，专职 L2 降采样 + 存储写入
```

**第二层是按 workload 拆多个 OAP 集群**：

```text
                    ┌──────────────────────────────┐
 Agent/SDK ───────▶ │  OAP 集群 A: Trace + Log     │──▶ ES 集群 A
 (按类型路由)        └──────────────────────────────┘
                    ┌──────────────────────────────┐
 Metrics/JVM ─────▶ │  OAP 集群 B: Metrics 专用     │──▶ BanyanDB / ES B
                    └──────────────────────────────┘
```

拆分维度常见三种：**按数据类型**（trace 吃 IO 与内存队列，metrics 吃 CPU 聚合，资源曲线完全不同）、**按业务域**（大部门各一套，故障隔离 + TTL 独立）、**按环境**（prod/staging 隔离，防压测污染容量）。拆分后 Agent 侧按服务指向不同 OAP 地址即可；UI 一套只连一个 GraphQL 集群，**全局视图需要另行规划**——这是拆分方案最容易被低估的成本。

> **行业实践**：拆分的第一动机通常是**故障隔离**——某服务 debug 全开造成的日志洪峰，不应挤掉核心交易 Trace 的处理能力；第二才是容量。先按数据类型拆一刀，几乎总能解决 80% 的相互拖累。

# 8. 监控 OAP 自身

观测系统自己必须有观测，否则链路失明时最被动。SkyWalking 内置 **telemetry 模块**，默认导出 Prometheus 指标：

```yaml
telemetry:
  selector: ${SW_TELEMETRY:prometheus}   # 10.x 默认开启
  prometheus:
    host: 0.0.0.0
    port: 1234                            # 独立于 12800 的指标端口
```

```bash
# 抓取 self-o11y 指标：队列积压、GC、各 worker 耗时
curl -s http://oap:1234/metrics | grep skywalking_oap
```

三类信号必须进告警：**队列积压**（`skywalking_oap_*_buffer/queue` 指标持续增长 = 容量告警第一信号）、**GC 停顿与老年代占用**、**存储 bulk 失败/拒绝计数**（配合 ES `_cluster/health` 与 write rejected）。UI 侧也有现成的 OAP 自监控仪表盘模板（`ui-initialized-templates` 中的 OAP 面板），接上 self-o11y 数据即可直接看。

> **注意**：self-observability 数据默认**不回流存储**（走 Prometheus 导出），不要画蛇添足让 Agent 把 OAP 当业务应用全采样——Trace 洪峰会自我放大；小集群可接受，大集群建议 self-o11y 与业务监控分离的抓取链路。

# 9. 压测与调优清单

```text
□ JVM: -Xms=-Xmx=8g, G1GC, IHOP=45, 关显式 GC, JDK 版本与 OAP 大版本匹配
□ gRPC: gRPCThreadPoolSize=-1 起步; maxConcurrentCallsPerConnection 限流防单连接独占
□ 聚合: prepareThreads 按核数上调; 确认 downsampling 层级是否真的需要 Day
□ 存储: bulkActions=5000, flushInterval=5s, concurrentRequests≥2; MySQL 调 asyncBatchPersistentPoolSize
□ ES: 单分片 10~50GB 规划 shard; write 线程池 rejected=0 为健康线
□ self-o11y: telemetry prometheus 端口 1234, 队列/GC/bulk 失败三类告警必配
□ 拆分阈值: 单机 CPU 持续>70% 或队列常态积压 → 先 role 分离/按数据类型拆集群
□ 日志级别: rootLevel=WARN（OAP logs 本身也吃磁盘 IO）
```

# 10. 小结

| 环节 | 核心参数（9.x/10.x 实际键名） | 瓶颈信号 |
|------|------------------------------|---------|
| JVM | `-Xms=-Xmx`、G1、IHOP | Full GC 频繁、停顿>1s |
| 接收 | `gRPCThreadPoolSize`、`maxConcurrentCallsPerConnection` | buffer full 日志、gRPC 超时 |
| 分析/聚合 | `l1FlushPeriod`、`prepareThreads`、`downsampling`、`core.role` | worker CPU 饱和、队列积压 |
| 存储 | `bulkActions`、`flushInterval`、`concurrentRequests`、`batchOfBytes` | 429 rejected、batch failure |
| 架构 | Receiver/Aggregator 角色分离 + 按 workload 拆集群 | 单机持续>70%、故障相互拖累 |

调优次序记住一句话：**先保存储消费能力，再喂饱分析管道，队列只是缓冲垫不是引擎**；容量过阈值用拆分换扩展，比无限堆单机更可持续。
