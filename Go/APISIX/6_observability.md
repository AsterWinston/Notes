网关是流量的汇聚点，也是排障时最先被怀疑的对象。日志回答"这个请求发生了什么"，指标回答"整体是否健康"，追踪回答"慢在哪一跳"。三件套配齐，很多问题不需要登机器猜。

# 1. 可观测性三件套

| 维度 | 回答的问题 | 粒度 | 成本 | 典型工具 |
|------|------------|------|------|----------|
| 日志 | 这个请求具体发生了什么 | 单请求 | 高 | ELK、Loki |
| 指标 | 整体趋势和异常 | 聚合 | 低 | Prometheus、Grafana |
| 追踪 | 跨服务调用链慢在哪 | 单链路 | 中 | Jaeger、Tempo |

指标告诉你"5xx 涨了"，日志告诉你"哪条路由、哪个上游、什么错误"，追踪告诉你"是网关慢还是上游慢"。

★ **关键认知**：网关的指标要能**下钻到路由和上游**。只看全局 QPS 和 5xx 比例，故障时无法判断是哪个服务、哪台机器。APISIX 的 Prometheus 指标默认带 `route`、`service`、`consumer`、`node` 标签，抓取时要保留。

# 2. 访问日志

## 2.1 格式

```yaml
nginx_config:
  http:
    enable_access_log: true
    access_log: logs/access.log
    access_log_format: >-
      $remote_addr "$request" $status $body_bytes_sent
      rt=$request_time urt=$upstream_response_time
      route=$route_id consumer=$consumer_name
      upstream=$upstream_addr ustatus=$upstream_status
      req_id=$request_id
    access_log_format_escape: default
```

`$request_time` 是网关看到的总耗时，`$upstream_response_time` 是上游耗时，两者之差大致是网关自身开销。

## 2.2 常用变量

| 变量 | 含义 |
|------|------|
| `$request_time` | 请求总耗时，秒 |
| `$upstream_response_time` | 上游响应耗时，秒 |
| `$upstream_addr` | 实际转发的上游地址 |
| `$upstream_status` | 上游返回的状态码 |
| `$route_id` | 命中的路由 ID |
| `$service_id` | 命中的 Service ID |
| `$consumer_name` | 认证后的 Consumer 名 |
| `$request_id` | 请求 ID，便于串联日志 |
| `$balancer_ip` | 负载均衡选中的节点 IP |

有了 `$route_id` 和 `$consumer_name`，日志平台就能按路由、按调用方聚合，不用再靠 URI 正则去猜。

## 2.3 结构化日志插件

| 插件 | 去向 | 适用 |
|------|------|------|
| `http-logger` | HTTP 接口 | 自建日志服务 |
| `kafka-logger` | Kafka | 大流量、异步解耦 |
| `file-logger` | 本地文件 | 配合 Filebeat |
| `syslog` | Syslog | 传统运维体系 |

```json
{
  "http-logger": {
    "uri": "http://log-collector:8080/logs",
    "batch_max_size": 100,
    "inactive_timeout": 5,
    "max_retry_count": 3,
    "include_req_body": false
  }
}
```

`batch_max_size` 控制攒批，`inactive_timeout` 控制多久没新日志就强制发送。

⚠ **关键认知**：`include_req_body: true` 可能把密码、Token、身份证号写进日志。默认必须关闭；确需排查时临时开启并限定路由，排查完立刻关掉。请求头里的 `Authorization`、`Cookie` 同理。

# 3. Prometheus 指标

## 3.1 开启

先在路由或 Global Rule 上挂 `prometheus` 插件，再配置导出地址：

```bash
curl -X PUT "http://127.0.0.1:9180/apisix/admin/plugin_metadata/prometheus" \
  -H "X-API-KEY: $ADMIN_KEY" -d '{
  "export_uri": "/apisix/prometheus/metrics",
  "export_addr": { "ip": "0.0.0.0", "port": 9091 },
  "metric_prefix": "apisix_"
}'
```

```yaml
scrape_configs:
  - job_name: apisix
    metrics_path: /apisix/prometheus/metrics
    static_configs:
      - targets: ["10.0.0.10:9091", "10.0.0.11:9091"]
```

## 3.2 关键指标

| 指标 | 类型 | 含义 |
|------|------|------|
| `apisix_http_status` | counter | 按状态码、路由、上游统计的请求数 |
| `apisix_http_latency` | histogram | 请求延迟，`type` 区分 request/upstream/apisix |
| `apisix_bandwidth` | counter | 入向和出向字节数 |
| `apisix_nginx_http_current_connections` | gauge | 当前连接数，按状态区分 |
| `apisix_node_info` | gauge | 节点信息，值为 1 |
| `apisix_etcd_reachable` | gauge | etcd 是否可达 |
| `apisix_etcd_modify_indexes` | gauge | etcd 配置修改版本 |

`apisix_http_latency` 的三个 `type` 是排查"慢在哪"的核心：`request` 是客户端到网关的总耗时，`upstream` 是网关到上游，`apisix` 是网关自身开销，约等于 `request - upstream`。

## 3.3 PromQL 示例

```promql
# 全局 5xx 比例
sum(rate(apisix_http_status{code=~"5.."}[5m]))
/
sum(rate(apisix_http_status[5m]))

# 全局 P99 请求延迟
histogram_quantile(
  0.99,
  sum(rate(apisix_http_latency_bucket{type="request"}[5m])) by (le)
)

# 每条路由的 QPS
sum(rate(apisix_http_status[1m])) by (route)

# 网关自身开销 P99
histogram_quantile(
  0.99,
  sum(rate(apisix_http_latency_bucket{type="apisix"}[5m])) by (le)
)
```

如果 `apisix` 类型的延迟突然升高，而上游延迟没变，问题通常在网关：插件太重、共享内存打满、worker 不够、DNS 解析慢。

★ **关键认知**：先看 `request` 是否超标，再看 `upstream` 是否占了大头，最后用 `apisix` 判断是不是网关自身的问题。三个 `type` 要一起看，不能只盯总延迟。

# 4. 链路追踪

## 4.1 trace context

APISIX 支持 W3C Trace Context，通过 `traceparent` 在服务之间传递：

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                  │
             │  │                                │                  └─ 采样标志
             │  │                                └─ 父 Span ID
             │  └─ Trace ID
             └─ 版本
```

## 4.2 opentelemetry 配置

```yaml
plugin_attr:
  opentelemetry:
    trace_id_source: x-request-id
    resource:
      service.name: APISIX
    collector:
      address: "otel-collector:4318"
      request_timeout: 3
    batch_span_processor:
      drop_on_queue_full: false
      max_queue_size: 1024
      max_export_batch_size: 512
```

```json
{
  "opentelemetry": {
    "sampler": {
      "name": "trace_id_ratio",
      "options": { "fraction": "0.01" }
    },
    "additional_attributes": ["http.method", "http.path"]
  }
}
```

## 4.3 采样策略

| 策略 | 含义 | 适用 |
|------|------|------|
| `always_on` | 全采样 | 开发、压测、短时排障 |
| `always_off` | 不采样 | 关闭追踪 |
| `trace_id_ratio` | 按比例采样 | 生产常用 1%-10% |
| `parent_base` | 跟随上游的采样决定 | 已有统一采样策略 |

⚠ **关键认知**：采样率 1% 意味着 99% 的请求没有完整链路，故障时可能找不到出问题的那条。生产建议"低比例常规采样 + 错误请求全采样"（在 Collector 侧做尾部采样），而不是简单调高全局比例。

# 5. Admin API 与 Control API

## 5.1 常用端点

| 方法与路径 | 作用 |
|------------|------|
| `GET /apisix/admin/routes` | 列出路由，支持分页 |
| `GET /apisix/admin/routes/{id}` | 读取单条路由 |
| `GET /apisix/admin/plugins/list` | 已加载插件列表 |
| `GET /apisix/admin/plugins/{name}` | 插件 schema |
| `GET /apisix/admin/schema/{resource}` | 资源 schema |
| `GET /apisix/admin/migrate/export` | 导出全量配置 |
| `POST /apisix/admin/migrate/import` | 导入配置 |
| `PUT /apisix/admin/plugins/reload` | 重载插件代码 |

排查配置问题时，第一步永远是"读回配置"，而不是看提交记录：

```bash
curl -s "http://127.0.0.1:9180/apisix/admin/routes/10" \
  -H "X-API-KEY: $ADMIN_KEY" | jq .
```

## 5.2 Control API

Control API 默认只监听 `127.0.0.1:9090`，适合本机排障：

```bash
curl -s http://127.0.0.1:9090/v1/healthcheck | jq .       # 网关自身健康
curl -s http://127.0.0.1:9090/v1/healthcheck/100 | jq .   # 上游健康状态
curl -s http://127.0.0.1:9090/v1/discovery/nacos/dump | jq .
```

`/v1/healthcheck` 返回的是 APISIX 自身状态，不是上游状态。上游状态要用带 `upstream_id` 的端点。

★ **关键认知**：Admin API 写入成功只代表 etcd 落盘成功，不代表所有数据面已经生效。配置同步是异步的，跨机房可能延迟几百毫秒到几秒。刚改完就验证时，先等一秒或观察 `apisix_etcd_modify_indexes`。

# 6. Dashboard

APISIX Dashboard 是独立进程，默认监听 9000，需要 etcd 地址和 Admin API 密钥。适合可视化查看路由、上游、Consumer、SSL，以及手工创建和修改配置。

它不适合作为唯一配置来源：Dashboard 的变更不会自动进 Git，无法审计和回滚；批量操作还是走 API 或 GitOps；图表粒度粗，不能替代 Prometheus + Grafana。

⚠ **关键认知**：Dashboard 只是 Admin API 的客户端。多人同时在生产 Dashboard 上改配置，很容易互相覆盖。生产应把配置纳入 Git，通过 CI 调 Admin API 发布，Dashboard 只用于只读查看和应急。

# 7. 告警设计

| 指标 | 告警条件示例 | 说明 |
|------|--------------|------|
| 延迟 | P99 > 1s 持续 5 分钟 | 按业务 SLO 定 |
| 流量 | QPS 突降 50% | 可能是上游或 DNS 故障 |
| 错误 | 5xx 比例 > 5% 持续 5 分钟 | 最核心的告警 |
| 饱和度 | 连接数 > 80% 上限 | 需要扩容或限流 |
| etcd | `apisix_etcd_reachable == 0` | 配置无法更新 |
| 证书 | 距过期 < 14 天 | 避免 HTTPS 中断 |

```yaml
groups:
  - name: apisix
    rules:
      - alert: ApisixHigh5xxRate
        expr: |
          sum(rate(apisix_http_status{code=~"5.."}[5m]))
          /
          sum(rate(apisix_http_status[5m])) > 0.05
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "APISIX 5xx 比例超过 5%"

      - alert: ApisixEtcdUnreachable
        expr: apisix_etcd_reachable == 0
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "APISIX 无法连接 etcd"
```

★ **关键认知**：告警要按"用户能感知的指标"设计，而不是按"机器指标"设计。CPU 80% 不一定影响用户，5xx 5% 一定影响。机器指标可以作为容量预警，但不应该成为半夜叫醒人的主要来源。

# 8. 排错手册

| 状态码 / 现象 | 含义 | 排查顺序 |
|---------------|------|----------|
| 502 | 网关连不上上游，或上游返回非法响应 | 看 `$upstream_addr`、`$upstream_status`，再查端口和健康检查 |
| 504 | 网关等上游超时 | 看 `$upstream_response_time`，对比 `read` 超时 |
| 499 | 客户端提前断开 | 看客户端超时和网关排队时间 |
| 503 | 无健康节点，或被限流/熔断 | 看健康检查、限流日志、`api-breaker` |
| 指标 9091 抓不到 | 插件没挂、导出服务没开 | 挂插件，检查 `plugin_metadata` |
| 追踪没有 Span | 采样率为 0、Collector 不可达 | 检查 sampler 和 `plugin_attr.opentelemetry` |
| 日志没有 `$route_id` | 日志格式没加变量 | 修改 `access_log_format` 并 reload |

配置不生效时按顺序查：路由是否命中（看 `$route_id`）→ 插件是否加载（`plugins/list`）→ 挂载层级是否被覆盖 → 优先级是否被抢先 → 是否用错了模式（传统模式读 etcd，Standalone 读本地 YAML）。

# 9. 练习

1. 在访问日志格式里加入 `$route_id`、`$consumer_name`、`$request_id`，用 `curl` 发几个请求，确认字段都有值。
2. 开启 Prometheus 插件，用 PromQL 查出"每条路由的 5xx 比例"，并按比例从高到低排序。
3. 配置 OpenTelemetry 插件，把采样率设为 1%，用 Jaeger 查看一条完整链路，确认网关 Span 和上游 Span 是否连在一起。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 日志 | 加 `route_id`、`consumer_name`、`request_id`，结构化后送远端 |
| 指标 | `http_status`、`http_latency`、`bandwidth`、连接数、etcd 可达性 |
| 延迟拆解 | `request = upstream + apisix`，先定位慢在哪一段 |
| 追踪 | W3C `traceparent`，生产用比例采样 + 错误全采样 |
| Admin API | 读回配置、查 schema、导出备份 |
| Control API | 本机健康检查、上游健康状态、服务发现实例 |
| Dashboard | 只是 Admin API 的客户端，不能替代 GitOps 和监控 |
| 告警 | 按用户感知的 5xx、延迟、饱和度设计 |

下一篇进入动态配置与 etcd，讲清配置的 key 结构、Watch 同步机制、灰度发布、版本回滚，以及如何用 Go 直接读取 APISIX 的配置来搭控制面。
