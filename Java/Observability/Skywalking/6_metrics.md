
# 1. 指标类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **Counter** | 累加计数器 | 请求总数、错误总数 |
| **Gauge** | 瞬时值 | JVM 内存使用量、线程数 |
| **Histogram** | 分布统计 | 响应时间分布（P50/P90/P99） |

# 2. 指标聚合层级

```
Agent 端（不聚合，直接上报 TraceSegment）
  │
  ▼
OAP L1 聚合（内存中，秒级）
  ├── 从 TraceSegment 提取指标
  ├── 按分钟窗口聚合
  └── 输出到 L2
      │
      ▼
OAP L2 聚合（存储中，分钟级）
  ├── 从 L1 接收分钟级数据
  ├── 进一步聚合（小时/天）
  └── 写入存储后端
```

# 3. OAL — 观测分析语言

OAL（Observability Analysis Language）是 SkyWalking 的指标定义 DSL：

```oal
// 服务级指标
service_resp_time = from(Service.*).latency();
service_sla = from(Service.*).status(equal=true, isSuccess).percent();
service_cpm = from(Service.*).cpm();       // Call Per Minute
service_apdex = from(Service.*).apdex(500, 2000);

// 端点级指标
endpoint_resp_time = from(Endpoint.*).latency();
endpoint_cpm = from(Endpoint.*).cpm();
endpoint_sla = from(Endpoint.*).status(equal=true, isSuccess).percent();

// 数据库指标
database_access_resp_time = from(DatabaseAccess.*).latency();
database_access_cpm = from(DatabaseAccess.*).cpm();

// 实例指标
instance_jvm_young_gc_count = from(Instance.*).jvm().youngGCCount();
instance_jvm_old_gc_count = from(Instance.*).jvm().oldGCCount();
instance_jvm_memory_max = from(Instance.*).jvm().memoryHeapMax();
```

## 3.1 OAL 函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `latency()` | 平均响应时间 | `service_resp_time` |
| `cpm()` | 每分钟调用次数 | `service_cpm` |
| `percent()` | 百分比（通常用于 SLA） | `service_sla` |
| `sum()` | 求和（Counter） | `service_call_count` |
| `apdex(T, 4T)` | 应用性能指数 | `service_apdex` |
| `filter()` | 条件过滤 | `status(equal=true)` |
| `p99/p95/p90/p75/p50` | 百分位延迟 | `all_p99` |

## 3.2 自定义 OAL

```oal
// 在 config/oal/core.oal 中添加
order_created_count = from(Endpoint.order.create).filter(tag("success") == "true").sum();
order_failed_count = from(Endpoint.order.create).filter(tag("success") == "false").sum();
order_create_resp_time = from(Endpoint.order.create).latency();
```

# 4. MAL — 分钟级聚合语言

MAL（Meter Analysis Language）是 9.x 引入的新引擎，支持更灵活的指标聚合：

```yaml
# config/meter-analyzer-config/spring-sleuth.yaml
metricsRules:
  - name: http_server_requests
    metricsRange: minutes
    filter: >
      meter_name == "http_server_requests_seconds" &&
      tag("uri", "/api/user/**")
    expSuffix: tag("uri")
    exp:
      - count
      - avg
      - max
      - p99
```

# 5. 指标查询

```graphql
# GraphQL 查询服务级指标
{
  getValues: getLinearIntValues(
    metric: {
      name: "service_resp_time"
      id: "my-service"
    }
    duration: {
      start: "2024-01-15-1000"
      end: "2024-01-15-1100"
      step: MINUTE
    }
  ) {
    values {
      value
    }
    label
  }
}
```

# 6. 自监控（Self Observability）

SkyWalking OAP 自身也暴露指标：

```yaml
# config/application.yml
prometheus-fetcher:
  enabled: true
  host: http://localhost:1234
  interval: 15

# OAP 启动参数（启用自监控端口）
# -Dprometheus-fetcher.port=1234
```

**OAP 自监控指标**：
- `mesh_analysis_latency` — 分析延迟
- `mesh_analysis_error_count` — 分析错误数
- `persistence_prepare_latency` — 存储准备延迟

# 7. 指标存储结构

```
ES 索引结构:
  sw_metrics-all-20240115  (按天分索引)
  
  文档示例:
  {
    "service_id": "abc123",
    "entity_id": "service_resp_time",
    "time_bucket": 202401151030,
    "value": 45.2,
    "detail": [10, 20, 45, 100, 200]  (Histogram 分布)
  }
```
