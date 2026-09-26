
# 1. 整体架构

Prometheus 采用**分层模块化架构**，核心模块包括：

```
                        +---------------------+
                        |   Service Discovery |
                        | (K8s/Consul/DNS/...)|
                        +----------+----------+
                                   |
                                   v
+-----------------+     +----------+----------+     +------------------+
|   Targets       |---->|   Prometheus Server  |---->|   Alertmanager   |
| (Exporters/     |     |                      |     +------------------+
|  Applications)  |     |  +----------------+  |
|  Pushgateway    |<--->|  |   TSDB (存储)   |  |
+-----------------+     |  +----------------+  |
                        |                      |
                        |  +----------------+  |
                        |  |  PromQL Engine  |  |
                        |  +----------------+  |
                        |          |           |
                        +----------+-----------+
                                   |
                                   v
                        +---------------------+
                        |   HTTP API / UI     |
                        +---------------------+
                                   |
                                   v
                        +---------------------+
                        |   Grafana           |
                        +---------------------+
```

## 1.1 各模块职责

| 模块 | 职责 |
|------|------|
| Scraper | 按配置定期从 Target 拉取指标数据 |
| Storage | 接收 Scraper 推送的样本数据，写入 TSDB |
| PromQL Engine | 解析并执行查询，支持即时查询和范围查询 |
| Service Discovery | 动态发现并更新采集目标列表 |
| Alerting | 评估告警规则，将触发告警发送给 Alertmanager |
| HTTP API | 提供 RESTful 接口供外部查询和管理 |
| Web UI | 内置的简易查询和状态查看界面 |

# 2. 数据模型

## 2.1 时间序列（Time Series）

Prometheus 所有数据都存为**时间序列**，每个时间序列由以下标识唯一确定：

```
<metric_name>{<label_name>=<label_value>, ...}
```

示例：

```
http_requests_total{method="GET", handler="/api/v1/users", status="200"}
```

时间序列存储的是随时间变化的一系列样本数据点：

```
^
|  1.2  1.5  1.8  2.1  2.3
|   |    |    |    |    |
+---------------------------->  time
```

## 2.2 Metric Name（指标名）

- 标识一类度量指标的通用特征
- 遵循正则：`[a-zA-Z_:][a-zA-Z0-9_:]*`
- 冒号预留给 Recording Rule 使用，不建议在采集端使用

## 2.3 Labels（标签）

- Key/Value 对，标识时间序列的特定维度
- Label Key 遵循正则：`[a-zA-Z_][a-zA-Z0-9_]*`
- 以 `__` 开头的 Label 为内部保留标签（如 `__name__`、`__scheme__`、`__address__`）
- 标签使数据具有**多维性**，可进行灵活聚合和筛选

## 2.4 Samples（样本）

每个时间序列点包含：

| 字段 | 说明 |
|------|------|
| Timestamp | Unix 时间戳，毫秒精度 |
| Value | float64 浮点数 |

## 2.5 标签设计原则

```yaml
# Good: 有意义的维度
http_requests_total{method="GET", endpoint="/api", status="200"}

# Bad: 包含高基数标签（如 user_id、ip、email）
http_requests_total{user_id="12345", ip="10.0.0.1", email="user@example.com"}
```

# 3. Metric 类型

## 3.1 Counter（计数器）

- 只增不减（除非重置）
- 适用于：请求计数、错误计数、任务完成数

```promql
# 总请求数
http_requests_total

# 每秒请求速率
rate(http_requests_total[5m])

# 一定时间内的增量
increase(http_requests_total[1h])
```

## 3.2 Gauge（仪表盘）

- 可增可减，反映当前状态
- 适用于：内存使用量、CPU 温度、并发连接数

```promql
# 当前内存使用
node_memory_MemAvailable_bytes

# 求平均值
avg(node_memory_MemAvailable_bytes{instance="localhost:9100"})

# 使用差值
delta(node_memory_MemAvailable_bytes[5m])
```

## 3.3 Histogram（直方图）

- 对观测值进行分桶统计，支持计算分位数
- 包含：
  - `<basename>_bucket{le="<upper inclusive bound>"}`：累计桶计数
  - `<basename>_sum`：观测值总和
  - `<basename>_count`：观测值总次数

```promql
# 请求延迟的 P99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# 平均请求延迟
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

## 3.4 Summary（摘要）

- 与 Histogram 类似，但在客户端计算分位数
- 包含：
  - `<basename>{quantile="<φ>"}`：预计算的分位数
  - `<basename>_sum`：观测值总和
  - `<basename>_count`：观测值总次数

## 3.5 Histogram vs Summary

| 特性 | Histogram | Summary |
|------|-----------|---------|
| 分位数计算位置 | 服务端（PromQL） | 客户端 |
| 聚合能力 | 可跨实例聚合 | 不可聚合 |
| 配置灵活性 | 可自定义 Bucket | 预定义 Quantile |
| 资源开销 | 较低（客户端不做计算） | 较高（客户端计算分位数） |

# 4. Job 与 Instance

- **Instance**：一个可被抓取的端点，通常对应单个进程
- **Job**：一组执行相同功能的 Instance 的集合

```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets:
        - '192.168.1.1:9100'   # instance 1
        - '192.168.1.2:9100'   # instance 2
```

Prometheus 会自动添加标签：
- `job`：对应 `job_name`
- `instance`：对应 `<host>:<port>`

# 5. Pull 模型详解

## 5.1 Pull 优势

1. **更容易检测故障**：Pull 失败意味着目标不可达或已宕机
2. **更容易控制采集频率**：在 Server 端统一配置
3. **天然防过载**：Server 控制并发连接数
4. **更安全**：只需要 Server 到 Target 的单向网络可达

## 5.2 Pull 流程

```
1. 配置加载 ──> 2. 目标发现 ──> 3. HTTP GET /metrics
                                     │
                                     v
                              4. 解析响应
                                     │
                                     v
                            5. 数据清洗与验证
                                     │
                                     v
                              6. TSDB 写入
```

## 5.3 自定义 Headers

当通过代理采集或需要认证时：

```yaml
scrape_configs:
  - job_name: 'authenticated'
    authorization:
      credentials: 'bearer_token'
    params:
      name: ['value']
```

# 6. 指标暴露格式

Prometheus 使用**文本格式**暴露指标，基于 OpenMetrics 标准：

```
# HELP http_requests_total The total number of HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="post",code="200"} 1027 1395066363000
http_requests_total{method="post",code="400"} 3 1395066363000
```

格式说明：
- `# HELP`：指标说明（可选）
- `# TYPE`：指标类型（可选，但建议提供）
- 数据行：`metric_name{labels} value [timestamp]`
- 时间戳可选（省略时由 Server 自动添加采集时间）

# 7. 存储模型概述

Prometheus 2.x 使用自研的 TSDB 引擎：

```
/data/
  └── 01EM6Q6A1YPX4G9X6R6XG9X6R6/
      ├── chunks/
      │   └── 000001
      ├── index
      ├── meta.json
      └── tombstones
```

- 每 2 小时生成一个 Block（数据块）
- Block 不可变，最终被合并压缩
- WAL 用于故障恢复
- 设计受 Go 团队伯克利数据库（LevelDB）和 Facebook Gorilla TSDB 启发

（详细存储机制见 `13_local_storage.md`）

# 8. 高可用方案

## 8.1 基本高可用

最简单的方案是运行两个相同的 Prometheus 实例采集相同目标：

```
             +---> Prometheus A
Targets -----+
             +---> Prometheus B
```

## 8.2 联邦集群

适用于多级部署（如多个数据中心）：

```
Global Prometheus <--- Region Prometheus <--- Zone Prometheus <--- Targets
```

## 8.3 Thanos / VictoriaMetrics

适用于需要长期存储和全局视图的场景，详情见 `15_thanos.md`。
