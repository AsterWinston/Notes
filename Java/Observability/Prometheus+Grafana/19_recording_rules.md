
# 1. 什么是 Recording Rules

Recording Rules（记录规则）是 Prometheus 提供的一种**预计算机制**，可以将复杂的 PromQL 表达式定期计算并写入新的时间序列。预计算的结果就像普通指标一样可以被查询。

## 1.1 核心价值

- **加速查询**：避免每次 Dashboard 加载时都重新计算复杂表达式
- **减少资源消耗**：降低 PromQL 引擎的计算压力
- **简化查询**：为复杂计算提供简洁的指标名
- **降采样**：高分辨率数据聚合为低分辨率数据供长期保留

## 1.2 工作原理

```
原始数据                          Recording Rule                   预计算结果
http_requests_total ──┐                                          job:http_requests:rate5m
                      │     sum(rate(http_requests_total[5m]))        10:00  500
                      ├──>  by (job)                          ──>    10:05  520
http_requests_total   │                                            10:10  480
                      └─>  (每 60s 执行一次)                           ...
```

# 2. 配置语法

```yaml
# rules/recording.yml
groups:
  - name: http_recording_rules
    interval: 1m           # 评估间隔（默认继承 global.evaluation_interval）
    limit: 0               # 最大时间序列数（0=无限制）
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
        labels:
          aggregated: "true"   # 可选：附加标签

      - record: instance:http_request_duration:p99
        expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, instance))
        labels:
          quantile: "p99"
```

## 2.1 命名规范

Prometheus 官方推荐的 Recording Rule 命名格式：

```
<level>:<metric_name>:<operation>
```

| 级别 | 聚合粒度 | 示例 |
|------|----------|------|
| `job` | 按 job 聚合 | `job:http_requests_total:rate5m` |
| `instance` | 按 instance 聚合 | `instance:cpu_usage:avg5m` |
| `path` | 按 path 聚合 | `path:request_duration:p99` |
| `cluster` | 按集群聚合 | `cluster:node_cpu:avg5m` |

# 3. 常用 Recording Rule 模式

## 3.1 速率预计算

```yaml
groups:
  - name: rate_recording
    interval: 1m
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)

      - record: instance:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (instance, job)

      - record: job:http_errors:rate5m
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)

      - record: job:error_ratio:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
```

## 3.2 资源使用率预计算

```yaml
groups:
  - name: resource_recording
    interval: 1m
    rules:
      - record: instance:cpu_usage:avg5m
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

      - record: instance:memory_usage:percent
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

      - record: instance:disk_usage:percent
        expr: (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

      - record: instance:load:per_cpu
        expr: node_load15 / count(node_cpu_seconds_total{mode="idle"}) by (instance)
```

## 3.3 延迟预计算

```yaml
groups:
  - name: latency_recording
    interval: 5m
    rules:
      - record: job:request_duration:p50
        expr: histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))

      - record: job:request_duration:p95
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))

      - record: job:request_duration:p99
        expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))

      - record: job:request_duration:avg
        expr: sum(rate(http_request_duration_seconds_sum[5m])) by (job) / sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

## 3.4 Kubernetes 资源预计算

```yaml
groups:
  - name: kubernetes_recording
    interval: 1m
    rules:
      - record: namespace:pod_cpu_usage:avg5m
        expr: sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace)

      - record: namespace:pod_cpu_request:total
        expr: sum(kube_pod_container_resource_requests{resource="cpu"}) by (namespace)

      - record: namespace:pod_memory_usage:sum
        expr: sum(container_memory_working_set_bytes{container!=""}) by (namespace)

      - record: namespace:pod_memory_request:total
        expr: sum(kube_pod_container_resource_requests{resource="memory"}) by (namespace)

      - record: node:cpu_request_ratio
        expr: sum(kube_pod_container_resource_requests{resource="cpu"}) by (node) / sum(kube_node_status_allocatable{resource="cpu"}) by (node)
```

# 4. 性能优化实践

## 4.1 减少计算频率

对于变化慢的指标，评估间隔可以设得更长：

```yaml
groups:
  - name: slow_changing_metrics
    interval: 15m          # 资源使用率变化慢，15 分钟评估一次
    rules:
      - record: instance:disk_usage:percent
        expr: (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

## 4.2 降低原始数据分辨率

```yaml
# 将高分辨率数据降采样为低分辨率
groups:
  - name: downsampling
    interval: 5m
    rules:
      - record: instance:cpu_usage:5m_max
        expr: max_over_time(instance:cpu_usage:avg5m[1h])    # 1h 窗口的最大值
      - record: instance:cpu_usage:5m_avg
        expr: avg_over_time(instance:cpu_usage:avg5m[1h])     # 1h 窗口的平均值
```

## 4.3 链式预计算

```yaml
# 先计算基础指标
- record: instance:http_requests:rate5m
  expr: rate(http_requests_total[5m])

# 再基于基础指标聚合
- record: job:http_requests:rate5m
  expr: sum(instance:http_requests:rate5m) by (job)

# 多层次聚合
- record: cluster:http_requests:rate5m
  expr: sum(job:http_requests:rate5m) by (cluster)
```

# 5. 存储影响

Recording Rule 结果会像普通指标一样被存储，占用磁盘空间。需要权衡：

## 5.1 评估

```yaml
# 原始查询（每次查询时实时计算）
sum(rate(http_requests_total[5m])) by (job)
# 计算量：每个查询周期重复执行

# Recording Rule（预计算后存储）
job:http_requests:rate5m
# 计算量：一次写入，多次读取
# 存储量：额外的磁盘空间
```

## 5.2 存储开销计算

```
每时间序列的存储：
- 原始数据：1.37 bytes/sample × 86400 samples/day × N 序列
- Recording Rule：1.37 bytes/sample × 86400 samples/day × M 序列

M 通常远小于 N（聚合后序列数减少）
```

# 6. 与告警规则配合

```yaml
groups:
  - name: recording_for_alerts
    interval: 1m
    rules:
      - record: instance:cpu_usage:avg5m
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

  - name: alerts_using_recording_rules
    rules:
      - alert: HighCPUUsage
        expr: instance:cpu_usage:avg5m > 80
        for: 10m
```

# 7. 最佳实践

## 7.1 DO

```yaml
# 复杂的跨序列计算
- record: job:error_ratio:rate5m
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
    /
    sum(rate(http_requests_total[5m])) by (job)

# 频繁使用的聚合
- record: instance:cpu_usage:avg5m
  expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# Dashboard 中高频查询
- record: job:request_duration:p99
  expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))
```

## 7.2 DON'T

```yaml
# 避免简单的 1:1 映射（没有节省计算）
- record: instance:up
  expr: up  # 直接使用 up 即可

# 避免过于细粒度的预计算（存储成本 > 计算收益）
- record: instance:http_requests:rate15s
  expr: rate(http_requests_total[15s])  # 15s 窗口太短

# 避免一次性预计算所有可能的聚合
- record: instance:memory_total:bytes
  expr: node_memory_MemTotal_bytes  # 原始指标本身很简单
```
