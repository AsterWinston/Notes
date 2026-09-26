
# 1. 什么是 PromQL

PromQL（Prometheus Query Language）是 Prometheus 内置的**函数式查询语言**，支持即时查询、范围查询、聚合运算和数学计算。它的设计受 Go 模板和函数式编程语言启发，强调组合性和表达能力。

# 2. 数据类型

PromQL 有四种核心数据类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| Instant Vector | 同一时刻的一组时间序列 | `http_requests_total` |
| Range Vector | 一段时间内的一组时间序列 | `http_requests_total[5m]` |
| Scalar | 单个浮点数值 | `1.0`, `3.14` |
| String | 字符串 | `"hello"` |

## 2.1 Instant Vector

```promql
# 返回当前时刻所有 http_requests_total 时间序列的值
http_requests_total

# 筛选特定条件
http_requests_total{job="api-server", method="GET"}
```

## 2.2 Range Vector

```promql
# 返回过去 5 分钟内 http_requests_total 的所有样本点
http_requests_total[5m]
```

## 2.3 时间范围单位

| 单位 | 含义 |
|------|------|
| `s` | 秒 |
| `m` | 分钟 |
| `h` | 小时 |
| `d` | 天 |
| `w` | 周 |
| `y` | 年 |

# 3. Metric Selector（指标选择器）

## 3.1 完全匹配

```promql
# 精确匹配（=）
http_requests_total{method="GET"}

# 不匹配（!=）
http_requests_total{method!="POST"}

# 正则匹配（=~）
http_requests_total{method=~"GET|POST"}

# 正则不匹配（!~）
http_requests_total{method!~"PUT|DELETE"}
```

## 3.2 多标签过滤

```promql
# 多个标签条件取 AND（默认）
http_requests_total{job="api", method="GET", status="200"}

# 空标签值
http_requests_total{job=""}

# 匹配任意值（用 .+ 正则）
http_requests_total{job=~".+"}
```

## 3.3 标签存在性判断

```promql
# 存在 job 标签
http_requests_total{job=~".*"}

# 不存在 job 标签（即 job 标签为空）
http_requests_total{job=""}
```

# 4. Offset（时间偏移）

```promql
# 查询 1 小时前的数据
http_requests_total offset 1h

# 与范围选择器结合
http_requests_total[5m] offset 1d
```

# 5. @ Modifier（指定时间戳）

```promql
# 查询指定时间戳的数据
http_requests_total @ 1600000000

# 与 range 结合
http_requests_total[5m] @ 1600000000
```

# 6. 操作符

## 6.1 算术操作符

| 操作符 | 含义 |
|--------|------|
| `+` | 加法 |
| `-` | 减法 |
| `*` | 乘法 |
| `/` | 除法 |
| `%` | 取模 |
| `^` | 幂运算 |

```promql
# CPU 使用率
(1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 计算比例
rate(http_requests_total{status="500"}[5m]) / rate(http_requests_total[5m]) * 100
```

## 6.2 比较操作符

| 操作符 | 含义 |
|--------|------|
| `==` | 等于 |
| `!=` | 不等于 |
| `<` | 小于 |
| `>` | 大于 |
| `<=` | 小于等于 |
| `>=` | 大于等于 |

```promql
# 返回 CPU 使用率大于 80% 的实例
(1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) > 80

# 使用 bool 修饰符返回 0 或 1
http_requests_total > bool 1000
```

## 6.3 逻辑/集合操作符

| 操作符 | 含义 |
|--------|------|
| `and` | 交集（取左侧与右侧匹配的序列） |
| `or` | 并集（取左侧所有序列，加上右侧未匹配的） |
| `unless` | 差集（取左侧与右侧不匹配的序列） |

```promql
# 内存使用率高 且 CPU 使用率也高的实例
node_memory_MemAvailable_bytes < 1e9 and node_cpu_seconds_total > 0.8

# 合并两组指标
rate(http_requests_total{status="200"}[5m]) or rate(http_requests_total{status="404"}[5m])

# 排除特定条件
node_memory_MemAvailable_bytes unless {instance="localhost:9100"}
```

# 7. 向量匹配（Vector Matching）

## 7.1 One-to-One（一对一）

```promql
# 相同标签的序列进行运算
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# 使用 on 指定匹配标签
node_memory_MemAvailable_bytes / on(instance) node_memory_MemTotal_bytes

# 使用 ignoring 忽略标签
node_memory_MemAvailable_bytes / ignoring(job) node_memory_MemTotal_bytes
```

## 7.2 Many-to-One / One-to-Many（多对一 / 一对多）

```promql
# group_left：左侧为多，右侧为一
# 将 group 标签从右侧复制到左侧的结果中
rate(http_requests_total[5m]) / on(job) group_left group
  sum(rate(http_requests_total[5m])) by (job)

# group_right：右侧为多，左侧为一
sum(rate(http_requests_total[5m])) by (job) / on(job) group_right
  rate(http_requests_total[5m])
```

# 8. 聚合操作

| 操作 | 含义 |
|------|------|
| `sum` | 求和 |
| `min` | 最小值 |
| `max` | 最大值 |
| `avg` | 平均值 |
| `stddev` | 标准差 |
| `stdvar` | 方差 |
| `count` | 计数 |
| `count_values` | 对每个唯一值计数 |
| `bottomk` | 最小的 K 个值 |
| `topk` | 最大的 K 个值 |
| `quantile` | 分位数（0 ≤ φ ≤ 1） |
| `group` | 分组 |

## 8.1 基本用法

```promql
# 按 job 分组求和
sum by (job) (rate(http_requests_total[5m]))

# 不保留任何标签（全局聚合）
sum(rate(http_requests_total[5m]))

# 排除指定标签
sum without (instance) (rate(http_requests_total[5m]))
```

## 8.2 常用聚合模式

```promql
# 每个 job 的前 3 个请求最多的端点
topk by (job) (3, rate(http_requests_total[5m]))

# 每个 endpoint 的 P95 延迟
quantile by (endpoint) (0.95, rate(http_request_duration_seconds[5m]))

# 各状态码的请求数分布
count_values by (job) ("status_code", http_requests_total)
```

# 9. 二元表达式优先级

从高到低：

```
^
*, /, %
+, -
==, !=, <, <=, >, >=
and, unless
or
```

# 10. 常见查询模式

## 10.1 速率查询

```promql
# 每秒请求数（推荐）
rate(http_requests_total[5m])

# 每秒请求数（适用于低频计数器）
irate(http_requests_total[5m])

# 一定时间内的增量（适用于 Counter）
increase(http_requests_total[1h])
```

## 10.2 资源使用率

```promql
# CPU 使用率（排除 idle 模式）
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 磁盘使用率
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

## 10.3 可用性

```promql
# 服务可用性（成功率）
sum(rate(http_requests_total{status=~"2..|3.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# Up 状态
up{job="api-server"} == 1
```

# 11. 查询最佳实践

## 11.1 避免高基数查询

```promql
# Bad：可能导致 OOM
http_requests_total

# Good：先聚合再查询
sum(rate(http_requests_total[5m])) by (job)
```

## 11.2 使用 Recording Rule 预计算

```promql
# 对于频繁使用的复杂查询，在 rules 中预计算
record: job:http_requests_total:rate5m
expr: sum(rate(http_requests_total[5m])) by (job)
```

## 11.3 注意时间窗口选择

```promql
# 太短（抖动大）
rate(http_requests_total[30s])

# 太长（延迟高）
rate(http_requests_total[1d])

# 推荐：4 倍于采集间隔
# 如果 scrape_interval = 15s，推荐 [1m] 或 [5m]
rate(http_requests_total[1m])
```
