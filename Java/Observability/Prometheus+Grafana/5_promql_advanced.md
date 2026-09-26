
# 1. 函数详解

## 1.1 Counter 相关函数

### 1.1.1 rate()

计算 Counter 在指定时间窗口内的**每秒平均增长率**，是最常用的 PromQL 函数。它自动处理 Counter 重置（如进程重启导致的归零）。

```promql
# 每秒 HTTP 请求数
rate(http_requests_total[5m])
```

原理：取时间窗口内的第一个和最后一个值，除以时间差。若中间有重置（值变小），则累加所有片段的差值。

### 1.1.2 irate()

计算 Counter 在时间窗口内的**瞬时增长率**，取最后两个数据点计算。对突发流量更敏感，但曲线毛刺较多。

```promql
# 瞬时速率，适合短时间窗口的尖峰检测
irate(http_requests_total[5m])
```

| 特性 | rate | irate |
|------|------|-------|
| 计算方式 | 窗口内首尾差值 / 时间 | 窗口内最后两个点差值 / 间隔 |
| 曲线平滑度 | 平滑 | 毛刺多 |
| 适用场景 | 长期趋势、告警 | 短时尖峰检测 |
| 性能 | 更好（计算量少） | 稍差（需要提取最后两个点） |

### 1.1.3 increase()

计算 Counter 在时间窗口内的**总增量**，等价于 `rate() * 时间窗口`。

```promql
# 过去 1 小时的总请求数
increase(http_requests_total[1h])
```

### 1.1.4 resets()

返回 Counter 在时间窗口内的**重置次数**。可用于检测进程重启频率。

```promql
# 检测进程重启
resets(process_start_time_seconds[1h]) > 0
```

## 1.2 Gauge 相关函数

### 1.2.1 delta()

计算 Gauge 在时间窗口内**首尾差值**。

```promql
# 5 分钟内温度变化
delta(temperature_celsius[5m])
```

### 1.2.2 idelta()

计算 Gauge 在时间窗口内**最后两个样本的差值**，类似 irate 但用于 Gauge。

```promql
# 瞬时变化量
idelta(temperature_celsius[5m])
```

### 1.2.3 predict_linear()

基于线性回归预测 Gauge 在指定时间后的值。适用于磁盘满预测、资源耗尽预测等。

```promql
# 预测 4 小时后磁盘是否满
predict_linear(node_filesystem_free_bytes[1h], 4 * 3600) < 0
```

### 1.2.4 deriv()

计算 Gauge 在时间窗口内的**每秒变化率**（基于线性回归）。

```promql
# 每秒温度变化率
deriv(temperature_celsius[1h])
```

## 1.3 时间与变换函数

### 1.3.1 changes()

返回时间序列在时间窗口内的**值变化次数**。

```promql
# 过去 1 小时内状态变化次数
changes(node_memory_MemAvailable_bytes[1h])
```

### 1.3.2 rate() 的 Counter 重置正确处理

Prometheus 的 rate() 和 increase() 能正确处理 Counter 重置的原理：

```
样本点: 5 -> 10 -> 15 -> 3 -> 8 -> 12
                               ^重置点（进程重启）
rate 计算：(10-5) + (15-10) + (8-0) + (12-3)  / 总时间
          = 27 / 总时间
```

注意：重置后的 Counter 值从 0 开始（实际是 3），但 Prometheus 假设重置后从 0 开始，使用 0 作为前一个值，所以实际增量是 8（8-0）而不是 5（8-3）。

### 1.3.3 clamp_min() / clamp_max()

限制值的范围。

```promql
# 将值限制在 0 到 100 之间
clamp_max(clamp_min(cpu_usage, 0), 100)
```

### 1.3.4 round()

四舍五入到指定小数位。

```promql
# 四舍五入到整数
round(cpu_usage)

# 保留 2 位小数
round(cpu_usage, 0.01)
```

## 1.4 排序函数

```promql
# 升序排序
sort(rate(http_requests_total[5m]))

# 降序排序
sort_desc(rate(http_requests_total[5m]))
```

# 2. 直方图与分位数

## 2.1 Histogram 结构

Histogram 指标由三部分组成：

```
# HELP http_request_duration_seconds Request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 10
http_request_duration_seconds_bucket{le="0.01"} 20
http_request_duration_seconds_bucket{le="0.025"} 50
http_request_duration_seconds_bucket{le="0.05"} 80
http_request_duration_seconds_bucket{le="0.1"} 100
http_request_duration_seconds_bucket{le="0.25"} 150
http_request_duration_seconds_bucket{le="0.5"} 200
http_request_duration_seconds_bucket{le="1"} 250
http_request_duration_seconds_bucket{le="2.5"} 300
http_request_duration_seconds_bucket{le="5"} 350
http_request_duration_seconds_bucket{le="+Inf"} 400
http_request_duration_seconds_sum 450.5
http_request_duration_seconds_count 400
```

## 2.2 histogram_quantile()

计算指定分位数，需要结合 rate() 使用以得到有效值：

```promql
# P99 延迟（推荐方式）
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 按 path 分组的 P95
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path))
```

## 2.3 计算原理

`histogram_quantile()` 使用线性插值法计算分位数：

```
假设分位数 φ，累计计数为 C，总计数为 N
目标 rank = φ * N

找到 bucket 使 le_bucket_low < rank ≤ le_bucket_high
估值 = le_bucket_low + (le_bucket_high - le_bucket_low) * (rank - count_low) / (count_high - count_low)
```

## 2.4 Bucket 设计要点

```yaml
# Good：在典型值附近更密集的 bucket
le: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, +Inf]

# Bad：bucket 太少导致分位数不准
le: [0.1, 1, 10, +Inf]

# Bad：bucket 太密集浪费资源
le: [0.001, 0.002, 0.003, ..., 10]
```

## 2.5 平均延迟

```promql
# 平均请求延迟
rate(http_request_duration_seconds_sum[5m]) /
rate(http_request_duration_seconds_count[5m])
```

# 3. 子查询（Subquery）

子查询允许对 Range Vector 先进行一步计算，再将结果作为范围数据输入到外层查询。

## 3.1 语法

```
<instant_query>[<range>:<resolution>]
```

## 3.2 示例

```promql
# 每 5 分钟计算过去 5m 的 rate，再将结果作为 1h 范围进行 max 聚合
max_over_time(
  rate(http_requests_total[5m])[1h:5m]
)

# 计算过去一天中，每 15 分钟的最大 CPU 使用率
max_over_time(
  (100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance))[1d:15m]
)
```

## 3.3 应用场景

1. **降采样**：将高分辨率数据降采样用于长期分析
2. **计算跨时间窗口的聚合**：如"过去 1 小时内每 5 分钟的最大值"
3. **告警规则复杂化**：如检测持续异常而非瞬时异常

# 4. HTTP API

## 4.1 即时查询

```bash
GET /api/v1/query?query=http_requests_total&time=2024-01-01T00:00:00Z
```

```bash
curl 'http://localhost:9090/api/v1/query?query=up'
```

## 4.2 范围查询

```bash
GET /api/v1/query_range?query=rate(http_requests_total[5m])&start=...&end=...&step=15s
```

```bash
curl 'http://localhost:9090/api/v1/query_range?\
  query=rate(http_requests_total[5m])&\
  start=2024-01-01T00:00:00Z&\
  end=2024-01-01T01:00:00Z&\
  step=15s'
```

## 4.3 元数据 API

```bash
# 列出所有指标名
GET /api/v1/label/__name__/values

# 列出标签值
GET /api/v1/label/job/values

# 序列匹配
GET /api/v1/series?match[]=up&match[]=http_requests_total

# 目标状态
GET /api/v1/targets
GET /api/v1/targets?state=down

# 规则
GET /api/v1/rules
GET /api/v1/alerts
```

## 4.4 管理 API

```bash
# 热加载配置
POST /-/reload

# 优雅退出
POST /-/quit

# 创建快照（需启用 admin-api）
POST /api/v1/admin/tsdb/snapshot

# 删除指标（需启用 admin-api）
POST /api/v1/admin/tsdb/delete_series?match[]=up
```

# 5. 查询性能优化

## 5.1 避免全表扫描

```promql
# Bad：所有时间序列扫描
http_requests_total

# Good：使用标签过滤缩小范围
http_requests_total{job="api", namespace="prod"}
```

## 5.2 优先聚合再运算

```promql
# Bad：先运算再聚合（运算次数 = 序列数）
rate(http_requests_total[5m]) / sum(rate(http_requests_total[5m])) by (job)

# Good：先聚合再运算（运算次数 = job 数）
rate(http_requests_total[5m]) / on() group_left sum(rate(http_requests_total[5m]))
```

## 5.3 合理使用 Recording Rule

```promql
# 在 prometheus.yml 中配置
rule_files:
  - "rules/recording.yml"
```

```yaml
# rules/recording.yml
groups:
  - name: http_recording_rules
    interval: 1m
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

## 5.4 时间窗口选择

```promql
# 太短（数据点少，rate 不稳定）
rate(http_requests_total[30s])

# 太长（延迟高，反应慢）
rate(http_requests_total[1d])

# 推荐：4-20 倍于 scrape_interval
# scrape_interval=15s → [1m] 或 [5m]
```

# 6. 高级查询模式

## 6.1 异常检测

```promql
# 基于过去 7 天同比检测异常
rate(http_requests_total[5m])
  / (rate(http_requests_total[5m] offset 1w)) - 1 > 0.2
```

## 6.2 多条件告警

```promql
# CPU > 80% AND 内存 > 80%
(
  100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 80
)
and
(
  (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 80
)
```

## 6.3 空值处理

```promql
# 用 0 替换缺失值
rate(http_requests_total[5m]) or vector(0)

# 忽略 NaN
rate(http_requests_total[5m]) > 0
```
