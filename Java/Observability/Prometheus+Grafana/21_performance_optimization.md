
# 1. 性能瓶颈分析

Prometheus 的性能瓶颈通常出现在以下环节：

```
Scrape → WAL → Head Block → Compaction → Query
  │        │        │           │           │
采集耗时  写盘速度  内存占用  CPU/IO    查询延迟
```

## 1.1 关键性能指标

```promql
# 采集延迟
rate(prometheus_target_interval_length_seconds[5m])

# 采集失败数
rate(prometheus_target_scrapes_exceeded_sample_limit_total[5m])
rate(prometheus_target_scrapes_sample_duplicate_drop_total[5m])

# 摄入延迟
prometheus_tsdb_head_min_time
prometheus_tsdb_head_max_time

# 查询性能
prometheus_engine_queries_concurrent_max
rate(prometheus_engine_query_duration_seconds[5m])

# 存储性能
rate(prometheus_tsdb_compactions_total[5m])
prometheus_tsdb_storage_blocks_bytes
```

# 2. 采集优化

## 2.1 调整采集频率

```yaml
global:
  scrape_interval: 30s      # 默认 15s，非关键指标可调大为 30-60s
  scrape_timeout: 10s

scrape_configs:
  - job_name: 'slow-targets'
    scrape_interval: 60s    # 变化慢的指标（如磁盘使用率）
    scrape_timeout: 30s     # 采集慢的目标需要更长超时

  - job_name: 'fast-targets'
    scrape_interval: 10s    # 高频变化的指标（如 QPS）
    scrape_timeout: 5s
```

## 2.2 限制采集指标数

```yaml
scrape_configs:
  - job_name: 'node'
    # 默认限制 0（无限制）
    sample_limit: 10000       # 每个目标最多采集 10000 个样本
    target_limit: 100         # 该 job 最多 100 个目标
    label_limit: 50           # 每个指标最多 50 个标签
    label_name_length_limit: 100  # 标签名最大长度
    label_value_length_limit: 100 # 标签值最大长度
```

## 2.3 使用 Honor Labels 控制

```yaml
scrape_configs:
  - job_name: 'apps'
    honor_labels: true     # 保留应用设置的标签（避免 Prometheus 覆盖）
    honor_timestamps: true # 使用应用提供的时间戳而非采集时间
```

## 2.4 过滤不必要指标

```yaml
scrape_configs:
  - job_name: 'node'
    metric_relabel_configs:
      # 删除不需要的指标
      - source_labels: [__name__]
        regex: 'node_disk_(io_time|reads|writes)_weighted_seconds_total'
        action: drop
      # 只保留需要的指标
      - source_labels: [__name__]
        regex: 'node_(cpu|memory|network|filesystem)_.*'
        action: keep
      # 删除不必要的标签
      - regex: 'device|fstype'
        action: labeldrop
```

# 3. PromQL 优化

## 3.1 避免全表扫描

```promql
# Bad：扫描所有时间序列
http_requests_total

# Good：使用标签过滤缩小范围
http_requests_total{job="api", namespace="prod", method="GET"}
```

## 3.2 提前聚合

```promql
# Bad：先运算再聚合（运算量 = 序列数 × 操作数）
rate(http_requests_total[5m]) / sum(rate(http_requests_total[5m])) by (job)

# Good：先聚合再运算（运算量 = job 数 × 操作数）
rate(http_requests_total[5m]) / on() group_left sum(rate(http_requests_total[5m]))
```

## 3.3 合理使用 Recording Rule

```yaml
# rules/recording.yml
groups:
  - name: heavy_queries
    interval: 1m
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
      - record: instance:cpu_usage:avg5m
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)
```

## 3.4 避免过大的时间窗口

```promql
# Bad：超大时间窗口（需要扫描大量数据）
rate(http_requests_total[7d])

# Good：合理时间窗口
rate(http_requests_total[5m])

# 长期趋势使用 Recording Rule
```

## 3.5 优化 Histogram 查询

```promql
# Bad：每次查询重新计算 P99
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))

# Good：使用 Recording Rule 预计算
# 在 rules 中：
# - record: job:request_duration:p99
#   expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))

# 然后直接查询
job:request_duration:p99
```

# 4. 存储优化

## 4.1 调整数据保留

```bash
--storage.tsdb.retention.time=15d   # 数据保留时间（默认 15d）
--storage.tsdb.retention.size=100GB # 数据保留大小
```

## 4.2 优化 Block 合并

```bash
--storage.tsdb.min-block-duration=2h   # 最小 Block 时长
--storage.tsdb.max-block-duration=48h  # 最大 Block 时长（更长的 Block 减少 compaction 频率）
```

## 4.3 WAL 优化

```bash
--storage.tsdb.wal-compression        # 启用 WAL 压缩（减少磁盘 IO）
--storage.tsdb.wal-segment-size=128MB # WAL 段大小
```

## 4.4 内存优化

```bash
# 限制内存中的序列数
--storage.tsdb.max-exemplars=10000

# 内存池大小
--storage.tsdb.mmap-chunks-in-head=1000000

# 降低 Go GC 压力
GOGC=50 prometheus ...
```

# 5. 查询引擎优化

```bash
--query.timeout=2m              # 查询超时
--query.max-concurrency=20      # 最大并发查询数
--query.max-samples=50000000    # 单查询最大样本数
```

# 6. 硬件规划

## 6.1 容量估算

| 指标数 | 样本数/秒 | 采集间隔 | CPU | 内存 | 磁盘（15天） |
|--------|----------|----------|-----|------|------------|
| 10 万 | 6,667 | 15s | 2核 | 8GB | 30GB |
| 50 万 | 33,333 | 15s | 4核 | 16GB | 150GB |
| 100 万 | 66,667 | 15s | 8核 | 32GB | 300GB |
| 500 万 | 333,333 | 15s | 16核 | 64GB+ | 1.5TB |

## 6.2 公式

```
样本数/秒 = 时间序列数 / scrape_interval
每天样本数 = 时间序列数 × (86400 / scrape_interval)
磁盘空间(15天) = 每天样本数 × 1.37 bytes × 15
```

## 6.3 磁盘选型

- 推荐 **SSD**（NVMe > SATA SSD > HDD）
- TSDB 需要低延迟随机写入
- 避免 NFS 等网络存储

# 7. Prometheus 自身监控

## 7.1 性能 Dashboard

```promql
# 采集目标状态
up

# 内存使用
process_resident_memory_bytes{job="prometheus"}

# CPU 使用
rate(process_cpu_seconds_total{job="prometheus"}[5m])

# Goroutine 数量
go_goroutines{job="prometheus"}

# TSDB 状态
prometheus_tsdb_head_series           # Head Block 中的序列数
prometheus_tsdb_head_chunks           # Head Block 中的 Chunk 数
prometheus_tsdb_head_samples_appended_total  # 写入样本数

# 查询性能
prometheus_engine_queries_total       # 查询总数
prometheus_engine_query_duration_seconds  # 查询延迟
```

## 7.2 告警规则

```yaml
groups:
  - name: prometheus_self
    rules:
      - alert: PrometheusHighMemory
        expr: process_resident_memory_bytes{job="prometheus"} > 32e9
        for: 5m
        annotations:
          summary: "Prometheus memory > 32GB"

      - alert: PrometheusHighSeriesCount
        expr: prometheus_tsdb_head_series > 1000000
        for: 5m
        annotations:
          summary: "Prometheus > 1M time series"

      - alert: PrometheusTargetMissing
        expr: up{job="prometheus"} == 0
        for: 1m
        annotations:
          summary: "Prometheus self is down"

      - alert: PrometheusRuleEvaluationLatency
        expr: rate(prometheus_evaluator_duration_seconds[5m]) > 0.9
        for: 5m
        annotations:
          summary: "Rule evaluation latency > 90% of interval"
```

# 8. 分布式方案

当单机 Prometheus 达到性能上限时：

| 方案 | 说明 | 适用场景 |
|------|------|----------|
| 分片（Sharding） | 按 Job/Tenant 拆分到多个 Prometheus | 大规模指标采集 |
| 联邦（Federation） | 分层采集，全局聚合 | 多数据中心 |
| Thanos | 全局查询 + 长期存储 | 需要全局视图+长期存储 |
| VictoriaMetrics | 高性能替代，资源占用低 | 高基数/高性能需求 |

## 8.1 分片示例

```yaml
# Prometheus A（采集前半段实例）
scrape_configs:
  - job_name: 'node'
    relabel_configs:
      - source_labels: [__address__]
        regex: '10\.0\.[0-9]+\.[0-9]+:9100'
        action: keep

# Prometheus B（采集后半段实例）
scrape_configs:
  - job_name: 'node'
    relabel_configs:
      - source_labels: [__address__]
        regex: '10\.1\.[0-9]+\.[0-9]+:9100'
        action: keep
```
