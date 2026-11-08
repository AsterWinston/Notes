# 1. 可观测性集成目标

## 1.1 为什么要集成监控

**k6 指标只能回答"压出来的表现是什么"，回答不了"为什么是这个表现"——后者必须靠被测系统的可观测性数据**。当 p95 延迟从 80ms 爬到 800ms 时，`http_req_duration` 曲线本身不携带任何归因信息：是应用线程池满了、数据库连接池耗尽、GC 停顿，还是下游依赖抖动？定位这些问题需要把 k6 的负载曲线与**被测系统的应用指标（JVM/线程池/连接池）、中间件指标（DB 慢查询、MQ 堆积）、基础设施指标（CPU/内存/网络/磁盘）**放在同一条时间轴上对照。更进一步，压测期间的异常请求要能下钻到单次调用的 trace，才能把"错误率 2%"变成"这 2% 全部命中了同一个慢 SQL"。

因此可观测性集成的目标可以拆成三层：**指标层**——k6 侧指标进入时序库，与 SUT（System Under Test，被测系统）指标同屏；**日志层**——压测时间窗内的 SUT 日志可按标签/时间检索；**链路层**——压测请求携带追踪头，SUT 的 trace 后端能区分"压测流量"与"真实流量"并单独分析。三层齐备，压测才从"打分"升级为"诊断"。

## 1.2 典型集成路径

**主流路径是"k6 → Prometheus 生态 → Grafana 统一展示"**。v0.5x 的 k6 已把 Prometheus Remote Write 输出内置进主程序（`experimental-prometheus-rw`，不再需要 xk6 扩展），指标写入 Prometheus/Mimir/VictoriaMetrics/Thanos 任意兼容后端；SUT 自身的指标通常也已经在 Prometheus 里，两边天然同源，Grafana 一个数据源查全部。时序上 k6 会为每次运行生成唯一的 `test_run_id`，所有指标带上该标签，多次压测互不污染，也支持"本次 vs 基线"的对比查询。

```text
┌─────────┐  Remote Write   ┌──────────────────┐   ┌─────────┐
│   k6    │ ──────────────► │ Prometheus/Mimir  │◄──┤  SUT    │
│ (施压端) │  k6_* 指标       │  /VictoriaMetrics │   │ 应用指标 │
└─────────┘                 └────────┬─────────┘   └─────────┘
     │ traceparent 头                │ PromQL
     ▼                               ▼
┌─────────┐   OTLP/Jaeger   ┌──────────────────┐
│   SUT   │ ──────────────► │     Grafana       │
│ 链路埋点 │                  │ 压测面板+trace下钻 │
└─────────┘                 └──────────────────┘
```

老一代路径是 k6 → InfluxDB → Grafana（见第 2 节），在 InfluxDB v1 输出被移除后已不再是首选；另一条路径是 Grafana Cloud k6——云端执行的测试**自动**把指标写入 Grafana Cloud 的 k6 数据源，零配置获得趋势分析与历史对比（见第 15 篇）。

# 2. InfluxDB 集成

## 2.1 基本配置

**InfluxDB 曾是 k6 最早的可视化搭档，但内置输出只支持 InfluxDB v1，已在 k6 v0.34 移除**——老资料里的这条命令在新版本会直接报 `unknown output: influxdb`：

```bash
# 旧版 k6（<= v0.33）的写法，v0.5x 已不可用
K6_OUT=influxdb=http://localhost:8086/k6 k6 run script.js
```

v0.5x 上仍要把 k6 指标写进 InfluxDB，有三条替代路径：用社区扩展 **xk6-output-influxdb**（支持 v2 的 token/organization/bucket 认证，`xk6 build` 编入后 `--out influxdb=...`）；用 k6 的 **JSON 输出 + Telegraf** 的 file input 转写 InfluxDB（多一跳但零扩展）；或者干脆切到 Prometheus Remote Write——如果 Grafana 是最终消费端，v1/v2 的 InfluxDB 与 Prometheus 系在能力上没有本质差别，而后者是 k6 官方投入的方向。

## 2.2 使用建议

无论走哪条路径，InfluxDB 集成的两条老经验至今有效：**控制标签维度，避免基数爆炸**——InfluxDB 的 series 基数 = 各标签取值的笛卡尔积，把 `user_id`、未参数化的 `url` 这类高基数值打进标签，series 数量会失控，写入放大且查询变慢；**为每次测试添加 run_id 标签**——`k6 run --tag run_id=$CI_PIPELINE_ID script.js`，这样多次运行的数据共存一个 bucket 也能精确筛选与对比，测试结束后按 run_id 清理也简单。这两条同样适用于 Prometheus RW（见 3.2 与 6.2），本质是时序数据库的共性约束。

# 3. Prometheus Remote Write

## 3.1 基本配置

**Prometheus Remote Write 是 v0.5x 内置的输出扩展（名字仍带 experimental 前缀），配置全部通过 `K6_PROMETHEUS_RW_*` 环境变量完成**，比老扩展版少了二进制构建环节：

```bash
k6 run script.js \
  --out experimental-prometheus-rw \
  -e K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write \
  -e K6_PROMETHEUS_RW_TREND_STATS="p(90),p(95),p(99),avg,max" \
  -e K6_PROMETHEUS_RW_PUSH_INTERVAL=5s \
  -e K6_PROMETHEUS_RW_STALE_MARKERS=true
```

关键配置项的语义：`SERVER_URL` 是 Remote Write 端点（Prometheus 需开启 `--web.enable-remote-write-receiver`，Mimir/VM/Grafana Cloud 原生支持）；`TREND_STATS` 决定 trend 类指标（如 http_req_duration）预聚合出哪些统计量——**分位数在 k6 侧算好再推**，默认只有 `p(90),p(95)`，要 p99 必须显式加；`PUSH_INTERVAL` 是批量推送周期（默认 5s）；`STALE_MARKERS` 在测试结束时写入 Prometheus 的 stale marker，让曲线在测试停止处干净截断而不是拖一条平线。对接 Grafana Cloud 时端点与 Basic Auth（实例用户名 + API token）经 `K6_PROMETHEUS_RW_SERVER_URL` 与对应认证变量传入。

## 3.2 注意事项

**写入侧最需要注意的是推送失败的表现与背压**：Remote Write 端点不通时 k6 会记录输出错误日志但**继续压测**——指标静默丢失而测试照常出 summary，事后才发现"这次白压了"，因此压测前的连通性冒烟（30 秒小流量跑一次、去 Prometheus 查 `k6_vus` 是否出现）应固化为流程。采样频率方面，k6 推送的是**预聚合统计量而非原始样本**，存储量与 `TREND_STATS` 的个数、标签维度成正比，与 RPS 基本无关——这是 RW 输出相对 JSON 逐样本输出的最大优势，长时 soak 测试也不会撑爆磁盘。指标命名保持与团队既有规范一致：k6 推送的指标统一带 `k6_` 前缀，不要试图在 k6 侧改名，转换交给 Prometheus 的 relabel。

## 3.3 指标命名与形态

推送后的指标按类型呈现不同形态，写 PromQL 前需要知道这张映射表：

| k6 指标类型 | Prometheus 中的形态 | 示例 |
|------------|--------------------|------|
| Counter | `_total` 后缀 | `k6_http_reqs_total`、`k6_iterations_total` |
| Rate | `_rate` 后缀（0~1 的 gauge） | `k6_http_req_failed_rate` |
| Trend | 按 TREND_STATS 展开多条 | `k6_http_req_duration_p95`、`k6_http_req_duration_avg` |
| Gauge | 原名 | `k6_vus`、`k6_vus_max` |
| 元信息 | info gauge，标签携带元数据 | `k6_test_info`（含 test_run_id、脚本名等） |

所有序列都带 `test_run_id` 标签（另有 `service_name`、`scenario` 等），按次筛选与跨次对比都以它为主键。查询单次压测的 p95 曲线：

```text
k6_http_req_duration_p95{test_run_id="当前ID"}
```

RPS 用 counter 求速率：

```text
sum(rate(k6_http_reqs_total{test_run_id="当前ID"}[1m]))
```

# 4. Grafana 展示

## 4.1 面板设计

**面板以业务流程为主线组织，同屏呈现"负载侧 + 延迟 + 错误 + 资源"四要素**。一个经过验证的 k6 压测 dashboard 布局：第一行放全局态势——`k6_vus`（当前并发）、RPS、全局 p95、错误率四个 stat 面板，配 `test_run_id` 模板变量做下拉切换；第二行放负载侧明细——各 scenario 的迭代速率与 `dropped_iterations`（ dropped 非零意味着施压端已跟不上，数据不可信，见第 15 篇）；第三行放延迟分布——p90/p95/p99 叠线，按 `api` 标签拆分出关键接口各自的曲线；第四行放 SUT 资源——应用 CPU/JVM 堆/GC 时间、DB 连接池活跃数、下游 P99。Grafana 官方提供 k6 dashboard（ID 18030 等社区模板）可直接导入再按业务裁剪，比从零画省一半时间。

## 4.2 关联分析

**关联分析的操作定义是：把压测日志、SUT 日志、trace、指标对齐到同一条时间线，从 k6 侧的异常时刻出发反查系统侧发生了什么**。流程上分四步：第一步在 k6 曲线上锚定异常窗口（错误率抬头、p99 突刺的精确时刻）；第二步用该时间窗过滤 SUT 日志（前提是日志系统按时间可检索，压测标签如 `x-load-test: run-42` 请求头贯穿到日志 MDC 里最好）；第三步从 trace 后端按同一时间窗 + 压测标记筛出慢 trace，看单次调用的耗时分解落在哪个 span；第四步回到指标面板验证假设（如"DB 连接池在异常窗口打满"应对应 `hikaricp_connections_active == max`）。**压测前统一所有系统的时钟源（NTP）是这条链路的前提**——施压机与 SUT 时钟偏差几秒，时间窗对齐就全盘失效。

## 4.3 压测期间同步观测被测系统

**压测不是"发起负载然后等 summary"，而是全程盯守 SUT 的观测面**。值班式压测的最低配置：压测执行者屏幕上同时开三块——k6 实时曲线（Grafana 或 `k6 run --out web-dashboard` 的内置实时面板，v0.5x 可用）、SUT 资源面板、日志实时流。这样做的价值在于捕捉**只存在于过程中的信号**：延迟随时间的缓慢爬升（内存泄漏/连接泄漏）、周期性的 p99 毛刺（GC 或定时任务）、错误率与 VU 爬坡的精确同步关系（容量拐点的定位）。end-of-test summary 把这些全部平均掉，所以"summary 达标"不能替代过程观测，尤其对 soak 测试。

# 5. 分布式追踪与 OpenTelemetry

## 5.1 追踪头注入

**让 SUT 的 trace 后端把压测流量串成完整链路，需要 k6 在每个请求上注入 W3C Trace Context 的 `traceparent` 头**。手工方式是自己生成 trace-id/span-id：

```javascript
import http from "k6/http";
import { randomString } from "k6/crypto";

// W3C traceparent 格式：version-trace_id-parent_id-flags
// 00-<32位hex>-<16位hex>-01
function makeTraceparent() {
  const traceId = randomString(32, "abcdef0123456789");
  const spanId = randomString(16, "abcdef0123456789");
  return `00-${traceId}-${spanId}-01`;
}

export default function () {
  http.get("https://shop.example.com/api/cart", {
    headers: {
      traceparent: makeTraceparent(),        // 每次请求独立 trace
      "x-load-test": "run-42",               // 压测流量标记，便于后端过滤
    },
  });
}
```

更工程化的方式是使用 k6 的实验性追踪模块 `k6/experimental/tracing`，用 `instrumentHTTP` 给 http 模块的所有请求统一织入追踪头，免去每处手写：

```javascript
import { instrumentHTTP } from "k6/experimental/tracing";

// 对 http 模块全局注入：此后所有 http.get/post 自动带 traceparent
instrumentHTTP({
  headers: {
    "x-load-test": "run-42",
  },
});
```

注入之后，SUT 侧只要框架支持 W3C 上下文传播（Spring Boot + Micrometer Tracing、OpenTelemetry Java Agent 均默认支持），每个压测请求在 SUT 内部形成的 span 树都会挂到 k6 给出的 trace-id 下。在 Tempo/Jaeger 里按 `x-load-test` 属性或时间窗过滤，就能只看压测流量的链路，分析慢 span 的分布。

> **注意**：`traceparent` 的 flags 位若置为采样（`-01`），SUT 的采样器大概率全采压测流量——这对分析是好事，但要评估 trace 后端的写入容量；反过来如果 SUT 采样率很低，压测期间可能恰好一条 trace 都没留，排障时才发现。压测前应与平台方约定：**压测窗口内对带压测标记的流量提高采样率**。

## 5.2 与 OpenTelemetry 结合的思路

**k6 与 OTel 的结合目前是"链路先行、指标绕行"**。链路侧：如 5.1 所述，W3C traceparent 是 OTel 的默认传播格式，k6 注入的头与 OTel SDK 无缝衔接，这条路径成熟可用。指标侧：k6 没有内置 OTLP 输出，两种绕行方案——其一，Prometheus RW 写入 OTel Collector 的 Prometheus receiver（或经 remote write 兼容层），让 k6 指标进入 OTel 管道与 SUT 的 OTLP 指标汇合；其二，用社区扩展 xk6-output-opentelemetry 直接以 OTLP 协议推送 k6 指标。日志侧 k6 没有结构化日志输出的官方通道，惯例是把 run_id/时间窗写进测试报告，日志系统按窗口关联。

三支柱对齐后，Grafana 这类支持 exemplar 的面板可以把指标曲线上的点直接跳转到对应 trace（`k6_http_req_duration_p95` 的毛刺 → 点击 exemplar → Tempo 里的具体调用链），这是压测排障体验的天花板形态；但它依赖 SUT 侧指标带 exemplar 上报，改造成本不低，通常只对核心链路做。

# 6. 常见问题

## 6.1 指标丢失

**指标丢失的排查顺序：端点连通性 → 认证 → k6 输出错误日志 → 写入端容量**。先用 curl 直接 POST 一个样本到 Remote Write 端点验证网络与认证（Grafana Cloud 的 Basic Auth 用户名/密码写错是高频原因，症状是 k6 日志里 401）；再看 k6 stderr——输出扩展的写入失败会打印 `output` 相关错误但**不会中断测试**，压测机日志必须留存；写入端方面，Prometheus 未开 `--web.enable-remote-write-receiver` 时返回 404，VM/Mimir 的租户头缺失返回 4xx，这些都在 k6 日志有对应记录。最后一类"假丢失"：查询时时间范围或 `test_run_id` 过滤不对——k6 指标的时间戳是施压机时间，施压机时钟漂移会让数据"落在未来或过去"，查不到先查钟。

## 6.2 标签基数过高

**基数爆炸的元凶排序：未参数化的 URL > 用户级标签 > 状态码/方法组合**。REST 路径 `/api/users/12345` 每个 ID 一条序列；把 `user_id` 打进 tags 则每个虚拟用户一条；两者叠加时一次万级 VU 的压测能写出千万级序列，直接打爆 Prometheus 的 head block。治理手段：请求统一带 `name` 标签参数化为 `/api/users/:id`（k6 的 `http_req_duration{name="/api/users/:id"}` 会聚合）；用户维度信息只进日志/JSON 归档，**绝不进时序标签**；输出前用扩展的标签过滤配置或 Prometheus relabel 做白名单——只保留 `scenario`、`name`、`status`、`test_run_id` 等有限维度。判断现状是否健康：`count({__name__=~"k6_.*"})` 看单次运行的序列总量，正常在千级以内。

> **行业实践**：把"压测流量的可观测性接入"做成平台化能力而非每次手工配置——统一的 Grafana 压测 dashboard（模板变量选 test_run_id）、统一的压测标记头规范（`x-load-test`）、SUT 侧对压测标记流量自动提升采样率、指标入库统一走带标签白名单的接入层。做到这一步后，任何团队跑压测都自动获得三支柱对齐的排障能力，而不是每次重新发明。

# 7. 小结

可观测性集成的目标是把 k6 的负载数据与 SUT 的指标/日志/链路**对齐到同一条时间线**，让压测从"打分"升级为"诊断"。指标路径以内置的 Prometheus Remote Write 输出为主（`K6_PROMETHEUS_RW_*` 配置，`k6_` 前缀指标，`test_run_id` 标签做单次筛选），InfluxDB v1 输出已移除、v2 需扩展或绕行。Grafana 面板按业务流程组织、同屏四要素，压测全程盯守实时曲线而非只等 summary。链路侧用 W3C `traceparent` 头（手写或 `k6/experimental/tracing` 的 `instrumentHTTP`）+ 压测标记头打通 SUT trace；指标侧经 Prometheus 管道或 xk6-output-opentelemetry 汇入 OTel。两大高频坑：写入失败不中断测试导致的静默丢失（压测前冒烟验证连通性），与高基数标签（URL 参数化、用户维度不进时序库）。
