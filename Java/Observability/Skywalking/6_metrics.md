
# 1. 指标类型

SkyWalking 的指标体系回答的是"系统现在健康吗"——Trace 回答单次请求去哪了，指标则把海量请求**压缩成可比较、可聚合、可告警的时序数值**。理解指标先理解三种基本类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| **Counter** | 累加计数器，只增不减，聚合后常转为速率 | 请求总数、错误总数 |
| **Gauge** | 瞬时值，可增可减 | JVM 内存使用量、线程数 |
| **Histogram** | 分布统计，保留各桶计数 | 响应时间分布（P50/P90/P99） |

SkyWalking 的实现细节是：**所有指标最终都按时间桶（Minute/Hour/Day）存储**，百分位值由 OAP 在查询时用桶近似计算。这与 Prometheus 的原始 counter/gauge/histogram 语义不完全对等，跨系统对照时要在脑子里做一次映射。

**设计考量**：

- **为什么不存原始明细而存聚合**：指标的本质是用固定存储换"任意时刻可比"。全量明细交给 Trace/Log 存储，指标层只保留按维度切好的时间序列，才能支撑秒级返回的 Dashboard 与告警
- **精度损失点**：平均值（`service_resp_time`）会掩盖长尾，P99 靠桶近似有误差；对延迟敏感的业务应优先看分位数而不是均值

# 2. 实体模型（Source）与指标命名规范

**OAL 里一切指标都挂在"实体"（Source）上**。SkyWalking 把被观测对象抽象成一套层级实体，指标名 = 实体 + 度量，这就是命名规范的全部秘密：

```text
Service ──────────── 一个逻辑应用（service_name）
  ├── Instance ───── 服务的一个进程实例（主机/Pod）
  ├── Endpoint ───── 服务的一个入口端点（URL/RPC方法）
  └── ServiceRelation ── 服务间调用边（A→B）
派生实体：DatabaseAccess / CacheAccess / MQAccess ── 按调用目标的分类聚合视图
```

命名规范：`{实体}_{度量}`，全小写下划线分隔——`service_resp_time`、`endpoint_cpm`、`service_relation_server_resp_time`（关系类指标带 `server`/`client` 前缀标明视角）。这套约定直接决定了**告警规则和 Dashboard 模板能否被工具自动发现**，自定义指标必须遵守。

**Layer 分层**解决"指标属于哪个技术域"的问题：每个服务注册时声明 layer（Agent 可配置 `agent.application_layer`，或按使用的组件自动推断），Dashboard 按 layer 挂载对应模板——MQ 层显示消费延迟，DATABASE 层显示 SQL 统计，OS_LINUX 层显示 CPU/内存。同一份 `service_cpm` 指标在所有 layer 通用，但展示的仪表盘不同。常见 layer：`GENERAL`（普通应用）、`HTTP`、`RPC`、`DATABASE`、`CACHE`、`MQ`、`K8S`、`MESH`、`OS_LINUX`。**一个服务只能属于一个 layer**，混合网关型应用归入其入口协议对应的层。

> **注意**：实体 ID 采用 Base64 编码的 `service name`（实例再加 instance UUID），**改名等于新建实体**，历史指标与拓扑边全部断档——服务命名要当作 schema 来管理，迁移期用双上报或延长 TTL 兜底。

# 3. 指标聚合层级（L1/L2）

SkyWalking 指标不是"agent 算好直接落库"，而是**两级流式聚合**，这是它区别于"客户端埋点直报 TSDB"模式的核心设计：

```text
Agent 端（不聚合，直接上报 TraceSegment / Meter 原始数据）
  │
  ▼
OAP L1 聚合（内存中，分钟窗口，Worker 级）
  ├── 从 TraceSegment 提取指标（端点耗时、成功与否）
  └── 按 minute 窗口滚动，窗口内增量合并，输出到 L2
      │
      ▼
OAP L2 聚合（降采样，跨实例/跨粒度）
  ├── 多 OAP 实例的同一分钟数据在存储侧合并（Sum/Count/Max 字段还原）
  ├── Downsampling：Minute → Hour → Day
  └── 写入存储后端（ES/BanyanDB 的 metrics 索引）
```

**为什么要两级**：OAP 是多实例无状态部署的，同一个服务同一分钟的数据会被负载均衡切散到不同 OAP 上。L1 在内存做"单机内的增量聚合"（省传输、省存储写放大），L2 在存储层做"跨机合并"（指标写入带 SUM/COUNT/MAX 明细字段，查询或下一个周期再归并）。代价是**指标可见延迟约 1 个分钟窗口**——刚发生的错误要 1–2 分钟后才在曲线和告警里出现，排障时别误判为"数据丢了"。

Histogram/Percentile 类指标在 L1 阶段以桶计数形态合并，桶边界在 OAL 定义时固定，这也是 P99 只能近似的原因。

# 4. OAL — 观测分析语言

OAL（Observability Analysis Language）是 SkyWalking 的指标定义 DSL，文件在 `config/oal/*.oal`，OAP 启动时编译成流处理 Worker。关键认知：**它定义的不是"查询"而是"预计算管道"**——每条 from 语句对应一个常驻内存的聚合 Worker。

```oal
// 服务级指标（9.7 core.oal 的真实写法）
service_resp_time = from(Service.latency).longAvg();
service_sla = from(Service.*).percent(status == true);   // 0-10000 万分比
service_cpm = from(Service.*).cpm();                     // Call Per Minute
service_percentile = from(Service.latency).percentile(10); // p50/p75/p90/p95/p99
service_apdex = from(Service.latency).apdex(name, status);

// 端点级指标
endpoint_resp_time = from(Endpoint.latency).longAvg();
endpoint_cpm = from(Endpoint.*).cpm();
endpoint_sla = from(Endpoint.*).percent(status == true);

// 服务关系（调用边）指标：用 filter 区分 client/server 视角
service_relation_client_cpm = from(ServiceRelation.*).filter(detectPoint == DetectPoint.CLIENT).cpm();
service_relation_server_resp_time = from(ServiceRelation.latency).filter(detectPoint == DetectPoint.SERVER).longAvg();

// 数据库指标（虚拟实体；缓存/消息同理：CacheAccess、MQAccess）
database_access_resp_time = from(DatabaseAccess.latency).longAvg();
database_access_sla = from(DatabaseAccess.*).percent(status == true);

// 实例 JVM 指标（java-agent.oal）
instance_jvm_cpu = from(ServiceInstanceJVMCPU.usePercent).doubleAvg();
instance_jvm_young_gc_count = from(ServiceInstanceJVMGC.count).filter(phase == GCPhase.NEW).sum();
instance_jvm_old_gc_time = from(ServiceInstanceJVMGC.time).filter(phase == GCPhase.OLD).sum();
```

## 4.1 OAL 函数

| 函数 | 说明 | 输出类型 |
|------|------|---------|
| `longAvg()` / `doubleAvg()` | 求均值（延迟用 longAvg） | Double/Long |
| `cpm()` | 每分钟调用次数 | Long |
| `percent(...)` | 满足条件的百分比（SLA，0-10000） | Int |
| `count()` / `sum()` | 计数 / 求和 | Long |
| `max()` / `min()` / `mean()` | 极值与均值 | Double |
| `apdex(name, status)` | 应用性能指数（阈值在 application.yml 配置） | Int(0-100) |
| `percentile(10)` | 10 桶近似百分位（p50/p75/p90/p95/p99） | Multiple |
| `filter(字段 == 值)` | 条件过滤链 | 配合上述函数 |

Apdex 的满意/容忍阈值在 9.x 移出 OAL、由 `application.yml` 的 `core.default.apdex-threshold`（默认 500/2000ms）按 layer 配置，`得分 =（满意 + 容忍/2）/ 总数 × 100`——比单看均值更抗"长尾伪装"，适合做服务分级的横向排名。

## 4.2 自定义 OAL

```oal
// 在 config/oal/custom.oal 中添加（9.x 支持独立文件，不必改 core.oal）
// 依赖 agent 侧用 @Tag 写入 order_event 标签
order_created_count = from(Endpoint.*).filter(name == "/order/create").filter(tag["order_event"] == "created").count();
order_failed_count = from(Endpoint.*).filter(name == "/order/create").filter(tag["order_event"] == "failed").count();
```

注意两点现实约束：**其一**，source 实体名与 tag key 必须在运行时真实存在（tag 由 agent 通过 `@Tag` 写入），拼错不报错、只是永远没数据；**其二**，修改 OAL 后必须重启 OAP 重新编译管道，历史时间桶不会回填。

> **行业实践**：大多数团队只启用核心 OAL 文件（service/instance/endpoint/relation/database/mq），把不需要的分析模块（如 envoy、istio、mqtt）从 `config/oal/` 移除，能明显降低 OAP 内存与 CPU——**每个 Worker 都是有状态的常驻内存对象**，这是自研指标体系不会告诉你的成本账。

# 5. MAL — 指标分析语言

MAL（Meter Analysis Language）是 9.x 面向"外部指标导入"的分析引擎：Prometheus、OTLP metrics、Telegraf 等时序数据进入 OAP 后，由 MAL 规则重映射成 SkyWalking 实体指标，复用同一套 Dashboard 与告警。

```yaml
# config/meter-analyzer-config/jvm.yaml（Micrometer JVM 指标导入）
rules:
  - name: jvm_memory_used_bytes
    mode: TAG
    labelNames: [area, id]     # 保留/参与聚合的标签
    aggFunctions: [sum]        # 多实例同名字段合并函数
    detached: false            # true = 按标签组合拆成独立指标
```

**与 OAL 的分工**：OAL 的输入是"SkyWalking 自己的 Trace/Log/Meter 流"，输出实体指标；MAL 的输入是"第三方时序数据"（典型经 `prometheus-fetcher` 抓取或 OTLP 推送），做清洗、标签合并、单位换算。注意 `detached: true` 会把每个标签组合拆成独立序列，高基数标签会引发**指标爆炸**，MAL 规则里必须先做标签白名单。

# 6. 指标查询 API（GraphQL）

指标查询走 OAP 12800 的 GraphQL（`/graphql`）。`id` 参数是**实体的 Base64 ID**，不是可读名字：

```graphql
# 查询服务级指标：先解析 serviceId，再取值
{
  service:Services {
    getService(name: "my-service") { id, key }
  }
}

{
  readMetricsValues: getLinearIntValues(
    metric: { name: "service_resp_time", id: "my-service.U3Vj..." }
    condition: { start: "2024-01-15 1000", end: "2024-01-15 1100", step: MINUTE }
  ) {
    values { value timestamp }
  }
}
```

| API | 适用 |
|-----|------|
| `getSingleMetricsValues` | 单实体单值趋势（曲线图） |
| `getMultipleMinutesMetrics` | 多值指标（百分位/直方图） |
| `sortMetrics` | TopN 排名（慢服务端点榜） |
| `readLabeledMetricsValues` | 带标签指标（按实例/线程池拆分） |

**时间参数用 UTC、step 必须与指标的聚合粒度一致**：MINUTE 指标查 HOUR 步长可以做，反之（用细步长查粗粒度指标）只能得到空点。UI 的一切图表就是这些查询的封装，排"UI 没数"时直接用 curl 打 GraphQL，可以二分定位是 OAP 无数据还是 UI/前端问题。

# 7. 告警指标基线

指标是告警的燃料。SkyWalking 告警规则（`config/alarm-settings.yml`）直接引用 OAL/MAL 产出的指标名，支持阈值加连续窗口判据：

```yaml
# config/alarm-settings.yml
rules:
  # 服务级：响应时间连续 3 个周期 > 1000ms
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 3
    count: 3
    silence-period: 5
    message: "{name} 响应时间连续超过 1 秒"
  # 成功率跌破 95%
  service_sla_rule:
    metrics-name: service_sla
    op: "<"
    threshold: 9500      # SLA 以万分比存储，9500 = 95%
    period: 2
    count: 1
```

**基线怎么定**才有意义——拍脑袋的统一阈值等于没有阈值。经验做法：先按端点分层跑两周，取 `P99 值 × 1.5` 作为 RT 告警线、`峰值 CPM 的 20% 跌幅`作为流量异常线；核心交易端点收紧、后台任务放宽。另注意 `service_sla` 按**万分比**存储（9500 = 95%），这是告警配置里最隐蔽的坑。

> **行业实践**：告警通道用 webhook 对接 IM/值班平台时，务必利用 `silence-period` 与"恢复通知（scope: Service/Instance/Endpoint 三级）"防止告警风暴；9.x 的告警消息自带 `alarmMessageBufEvents` 聚合，webhook 收到的是一条含多个实体异常的事件而非逐条轰炸。

# 8. 指标存储结构

指标最终按"namespace + 指标名 + 天"组织存储（ES 模式下每个指标一个时间序列索引），形如：

```text
ES 索引结构:
  skywalking_service_resp_time-20240115     (namespace_指标名-日期)
  skywalking_endpoint_cpm-20240115
  skywalking_service_percentile-20240115    (多值/百分位类指标)

  文档示例:
  {
    "entity_id": "my-service",              // 实体 Base64 ID
    "time_bucket": 202401151030,
    "value": 45,
    "size": 128,                       // 桶内样本数（供跨 OAP 实例 L2 合并）
    "value_sum": 5785,                 // 增量合并字段
    "value_max": 320,
    "0": 10, "1": 20, ...              // percentile 类指标的桶计数
  }
```

`size/value_sum/value_max` 三个字段就是 L2 跨实例合并的数学基础——均值指标用 `Σvalue_sum / Σsize` 还原，max 指标取 `max(value_max)`，percentile 类指标以桶计数形态存储（文档尾部的 `0..9` 字段），查询期近似算分位数。理解这个结构后，"为什么多 OAP 实例下曲线依然准确"就不再是玄学。

## 8.1 OAP 自监控（Self Observability）

可观测性平台自己也要被观测。OAP 自监控有两条路：**原生 meter 导出**（内部 Worker 的队列积压、处理延迟等指标可直接查询）；**`prometheus-fetcher`** 反向抓取被监控方的 Prometheus 端点，抓到后交给 MAL（第 5 节）映射为 SkyWalking 实体指标：

```yaml
# config/application.yml —— 抓取模式导入外部指标
prometheus-fetcher:
  enabled: true
  timeout: 3
  forcePrometheusFormat: false
# config/prometheus-fetcher-config/*.yml：声明抓哪些 target、哪些指标
```

> **注意**：判断"OAP 处理不过来"的第一信号是它的 GC 与队列指标（持久化队列长度持续上涨），而不是业务曲线掉点——曲线掉点是果，队列积压才是因。自监控指标本身要能独立于主链路告警（哪怕用另一套 Prometheus 抓 OAP 的 `/metrics`）。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 实体模型 | Service/Instance/Endpoint/Relation + Layer 决定指标挂在哪、用哪套仪表盘 |
| 命名规范 | `{实体}_{度量}`；关系指标带 server/client 视角前缀；改名即断史 |
| 两级聚合 | L1 内存分钟窗口增量聚合，L2 存储跨实例合并；指标有 1–2 分钟可见延迟 |
| OAL/MAL | OAL 编译 Trace 流为预计算 Worker；MAL 导入 Prometheus 等外部指标 |
| 查询 | GraphQL 按 Base64 实体 ID + step 查询；"UI 没数"先 curl 二分 |
| 告警 | 阈值基于两周 P99×1.5 基线；SLA 是万分比；用 silence-period 防风暴 |

指标体系跑起来后，下一篇《7. 日志集成》把日志也拉进同一条 Trace 关联线。
