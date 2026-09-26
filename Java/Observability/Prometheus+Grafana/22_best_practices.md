
# 1. 指标命名规范

## 1.1 通用规范

```
<namespace>_<subsystem>_<unit>[_<suffix>]
```

| 部分 | 说明 | 示例 |
|------|------|------|
| namespace | 命名空间（应用名） | `prometheus`、`node`、`http` |
| subsystem | 子系统名 | `memory`、`network`、`requests` |
| unit | 单位 | `bytes`、`seconds`、`total` |
| suffix | 后缀（类型相关） | `_total`、`_count`、`_sum`、`_bucket` |

## 1.2 命名规则

```promql
# Counter：以 _total 结尾
http_requests_total
node_cpu_seconds_total
prometheus_tsdb_compactions_total

# Gauge：无特殊后缀，通常是当前值的描述
node_memory_MemAvailable_bytes
go_goroutines
http_active_requests

# Histogram：包含 _bucket/_sum/_count
http_request_duration_seconds_bucket{le="0.1"}
http_request_duration_seconds_sum
http_request_duration_seconds_count

# Summary：包含 quantile/_sum/_count
http_request_size_bytes{quantile="0.95"}
http_request_size_bytes_sum
http_request_size_bytes_count
```

## 1.3 命名禁忌

```promql
# Bad：使用大写字母
MyAppRequestsTotal

# Bad：使用中划线
my-app-requests-total

# Bad：使用驼峰
myAppRequestsTotal

# Bad：带单位前缀（不是单位后缀）
http_seconds_request_duration

# Bad：使用复数
requests_total

# Good：单数形式
request_total

# Good：合理使用下划线分段
http_request_duration_seconds_total
node_disk_reads_completed_total
```

# 2. 标签设计规范

## 2.1 好的标签

```promql
# 有意义的业务维度
http_requests_total{method="GET", path="/api/users", status="200"}

# 环境标识
node_cpu_seconds_total{env="production", region="us-east-1"}

# 版本信息
app_info{version="1.2.3", commit="abc123", branch="main"}
```

## 2.2 避免高基数标签

```promql
# Bad：高基数标签
http_requests_total{user_id="12345"}               # 用户 ID 无限增长
http_requests_total{session_id="abc-def"}           # Session ID 大量临时值
http_requests_total{email="user@example.com"}       # 邮件地址唯一性高
http_requests_total{ip="10.0.0.1"}                 # IP 地址变化频繁
http_requests_total{path="/api/v1/users/12345"}    # Path 中包含动态 ID

# Good：降基数后的标签
http_requests_total{user_tier="premium"}            # 用户等级（有限取值）
http_requests_total{path_template="/api/v1/users/:id"}  # 使用路径模板
http_requests_total{country="CN"}                   # 地域维度（有限取值）
```

## 2.3 标签值约束

```promql
# 合理的标签值数量
job: < 100 个
instance: < 1000 个
method: < 10 个（GET/POST/PUT/DELETE...）
status: < 100 个

# 需要警惕的情况
# 1. 用户 ID、请求 ID、Session ID 等唯一标识
# 2. URL 路径包含参数
# 3. 时间戳
# 4. 随机数
# 5. IP + Port 组合
```

## 2.4 标签数量控制

```promql
# 每个指标建议不超过 10 个标签
# 每个时间序列的所有标签值组合不应超过 100,000

# Good：5 个标签，基数可控
app_requests_total{job, instance, method, path, status}

# Bad：10+ 个标签，组合爆炸
app_requests_total{job, instance, method, path, status, user_id, session, ip, device, browser, os, version}
```

# 3. Metric 类型选择

| 场景 | 推荐类型 | 原因 |
|------|----------|------|
| 请求计数 | Counter | 只增不减，可用 rate() 算 QPS |
| 当前在线用户 | Gauge | 可增可减，反映当前状态 |
| 请求延迟 | Histogram | 需要聚合计算分位数 |
| 响应大小 | Summary | 不需要跨实例聚合的分位数 |
| 任务持续时间 | Histogram | 需要跨实例聚合 |
| 温度/水位 | Gauge | 反映即时状态 |
| 错误计数 | Counter | 只增不减 |
| 队列长度 | Gauge | 反映即时状态 |

# 4. 容器与 K8s 环境最佳实践

## 4.1 Pod 标签传递

```yaml
# 将 K8s 标签注入指标
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app
  - source_labels: [__meta_kubernetes_pod_label_version]
    target_label: version
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
```

## 4.2 限制容器指标采集范围

```yaml
metric_relabel_configs:
  # 过滤无用容器指标
  - source_labels: [container]
    regex: 'POD|'    # 排除 pause 容器和空容器
    action: drop
```

## 4.3 合理设置资源限制

```yaml
# kube-prometheus-stack values
prometheus:
  resources:
    requests:
      memory: 8Gi
      cpu: 2
    limits:
      memory: 16Gi
      cpu: 4
```

# 5. 告警规则最佳实践

## 5.1 告警分级

| 级别 | 定义 | 响应 | 示例 |
|------|------|------|------|
| P0 (Critical) | 服务完全不可用 | 立即响应，5 分钟 SLA | Service Down, Data Loss |
| P1 (Warning) | 服务降级或即将不可用 | 工作时间 30 分钟响应 | CPU > 90%, Disk > 85% |
| P2 (Info) | 非紧急通知 | 记录日志，次日处理 | 版本更新, 证书即将过期 |

## 5.2 避免告警疲劳

```yaml
# Bad：没有 for 持续时间，抖动会频繁触发
- alert: HighCPU
  expr: cpu_usage > 80

# Good：持续一段时间才触发
- alert: HighCPU
  expr: cpu_usage > 80
  for: 10m

# Bad：没有聚合，每个实例单独告警
- alert: HighCPU
  expr: cpu_usage > 80

# Good：聚合后减少告警数量
- alert: ManyHighCPU
  expr: count(cpu_usage > 80 by (instance)) > 10
```

## 5.3 告警信息可操作性

```yaml
- alert: HighErrorRate
  expr: error_ratio > 0.05
  annotations:
    summary: "Error rate is {{ $value | humanizePercentage }}"
    description: |
      Job: {{ $labels.job }}
      Current error rate: {{ $value | humanizePercentage }}
      Threshold: 5%
    runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
    dashboard_url: "https://grafana.example.com/d/abc123"
```

# 6. Recording Rule 最佳实践

```yaml
# 频繁使用的聚合查询
- record: job:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m])) by (job)

# 开销大的计算
- record: instance:cpu_usage:avg5m
  expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# Dashboard 中高频使用的分位数
- record: job:request_duration:p99
  expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))
```

# 7. Grafana Dashboard 设计

## 7.1 布局原则

1. **黄金信号**：每个 Dashboard 展示 RED/USE 指标
2. **一致性**：同类型指标使用相同的颜色方案（红色=错误，绿色=正常）
3. **时间范围**：默认展示 1-6 小时，支持快速切换
4. **模板变量**：使用变量实现交互式过滤

## 7.2 常用 Dashboard 模板

```json
{
  "title": "Service Overview",
  "panels": [
    { "title": "Requests Rate", "type": "timeseries", "target": "rate(...)" },
    { "title": "Error Rate",    "type": "timeseries", "target": "rate(...)" },
    { "title": "Latency P99",   "type": "timeseries", "target": "histogram_quantile(...)" },
    { "title": "CPU Usage",     "type": "gauge",      "target": "cpu_usage" }
  ]
}
```

# 8. 运维最佳实践

## 8.1 配置管理

- 使用版本控制管理 Prometheus 配置
- 使用 promtool 验证配置变更
- 配置变更后执行 `/-/reload` 热加载

## 8.2 备份策略

- 定期备份 TSDB 数据（通过快照 API）
- 备份 Alertmanager 静默配置
- 备份 Grafana Dashboard JSON

## 8.3 安全

- 使用反向代理（Nginx/Envoy）终止 TLS
- 配置 Basic Auth 或 OAuth 保护 Prometheus UI
- 限制 Prometheus 网络访问范围
- 定期轮换 API Token 和证书
