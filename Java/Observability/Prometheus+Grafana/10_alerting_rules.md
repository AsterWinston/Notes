
# 1. 告警架构

Prometheus 采用**采集与告警分离**的架构：

```
Prometheus Server                   Alertmanager
+-------------------+               +-------------------+
| Scrape Metrics    |               | Alertmanager      |
| Evaluate Rules    |-- Alert -->   | - Grouping        |
| Alert State       |   HTTP       | - Inhibition      |
| (Pending/Firing)  |               | - Silencing       |
+-------------------+               | - Notification    |
                                    | (Email/Slack/...) |
                                    +-------------------+
```

# 2. 告警规则语法

```yaml
# rules/alerts.yml
groups:
  - name: node_alerts
    interval: 30s       # 规则评估间隔（默认继承 global.evaluation_interval）
    limit: 0            # 限制单个告警的最大时间序列数（0=无限制）
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 80
        for: 5m          # 持续 5 分钟才触发
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Instance {{ $labels.instance }} CPU usage is high"
          description: "CPU usage on {{ $labels.instance }} is {{ $value | humanizePercentage }} for more than 5 minutes"
          runbook_url: "https://wiki.example.com/runbooks/high-cpu"
```

## 2.1 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `alert` | 是 | 告警名称，需唯一 |
| `expr` | 是 | PromQL 表达式，返回非零值时触发告警 |
| `for` | 否 | 持续时间，表达式连续满足该时长才转为 Firing |
| `labels` | 否 | 附加到告警上的标签，可用于路由匹配 |
| `annotations` | 否 | 告警附加信息（摘要、描述、Runbook 链接） |

# 3. 告警状态机

```
Inactive (未触发)
    |
    |-- expr 返回 true
    v
Pending (待处理)
    |
    |-- 持续满足 for 时长
    v
Firing (已触发) --> 发送到 Alertmanager
    |
    |-- expr 返回 false
    v
Inactive (恢复)
```

## 3.1 状态说明

- **Inactive**：表达式结果为 false（或向量为空），正常状态
- **Pending**：表达式结果为 true，但尚未达到 `for` 指定的持续时间
- **Firing**：表达式持续为 true 且超过 `for` 时长，告警被发送给 Alertmanager

# 4. 模板化告警

Prometheus 使用 Go 模板语法，通过 `$labels` 和 `$value` 变量引用告警上下文：

```yaml
- alert: InstanceDown
  expr: up{job="api"} == 0
  for: 1m
  annotations:
    summary: "Instance {{ $labels.instance }} is down"
    description: |
      Job: {{ $labels.job }}
      Instance: {{ $labels.instance }}
      Value: {{ $value }}
      Since: {{ ($value | humanizeTimestamp) }}
    runbook_url: "https://runbooks.example.com/instance-down"

- alert: HighMemoryUsage
  expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
  for: 10m
  annotations:
    summary: "{{ $labels.instance }} memory usage is high"
    description: "Instance {{ $labels.instance }} memory usage is {{ $value | humanizePercentage }}"
```

## 4.1 模板函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `humanize` | 格式化大数字 | `1000000` → `1e+06` |
| `humanizePercentage` | 格式化为百分比 | `0.95` → `95%` |
| `humanizeTimestamp` | 时间戳转可读时间 | `1600000000` → `2020-09-13` |
| `humanizeDuration` | 秒数转可读时长 | `3600` → `1h` |
| `match` | 正则匹配 | `match "pattern" $labels.instance` |
| `title` | 首字母大写 | `title "hello"` → `Hello` |
| `toUpper` / `toLower` | 大小写转换 | |
| `reReplaceAll` | 正则替换 | `reReplaceAll ":[0-9]+" "" $labels.instance` |
| `graphitePrefix` / `graphiteTags` | Graphite 格式转换 | |

# 5. 常用告警规则模式

## 5.1 节点级别告警

```yaml
groups:
  - name: node_alerts
    rules:
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} is down"

      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage > 80% on {{ $labels.instance }}"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage > 90% on {{ $labels.instance }}"

      - alert: DiskUsage
        expr: (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage > 85% on {{ $labels.instance }} mount {{ $labels.mountpoint }}"

      - alert: DiskSpacePredict
        expr: predict_linear(node_filesystem_free_bytes[6h], 4*3600) < 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Disk on {{ $labels.instance }} will fill in 4 hours"

      - alert: DiskIOHigh
        expr: rate(node_disk_io_time_seconds_total[5m]) > 0.9
        for: 10m
        annotations:
          summary: "Disk IO > 90% on {{ $labels.instance }} device {{ $labels.device }}"

      - alert: NetworkErrors
        expr: rate(node_network_receive_errors_total[5m]) / rate(node_network_receive_packets_total[5m]) > 0.01
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Network error rate > 1% on {{ $labels.instance }} interface {{ $labels.device }}"

      - alert: HighLoad
        expr: node_load15 > (count(node_cpu_seconds_total{mode="idle"}) by (instance) * 0.8)
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Load average > 80% of CPU count on {{ $labels.instance }}"
```

## 5.2 服务级别告警

```yaml
groups:
  - name: service_alerts
    rules:
      - alert: ServiceDown
        expr: up{job=~"api|web|worker"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} instance {{ $labels.instance }} is down"

      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job) / sum(rate(http_requests_total[5m])) by (job) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% for job {{ $labels.job }}"

      - alert: HighLatency
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency > 2s for job {{ $labels.job }}"

      - alert: TooManyRestarts
        expr: changes(process_start_time_seconds{job="api"}[15m]) > 3
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} restarted {{ $value }} times in 15 minutes"
```

# 6. 多条件告警

```yaml
- alert: BothHighCPUAndMemory
  expr: >
    (100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 80)
    and
    ((1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 80)
  for: 5m
  annotations:
    summary: "Both CPU and Memory > 80% on {{ $labels.instance }}"
```

# 7. Absent 告警

用于检测指标数据停止上报：

```yaml
- alert: NoData
  expr: absent(up{job="api"}) == 1
  for: 5m
  annotations:
    summary: "No data for job api for 5 minutes"

- alert: JobAbsent
  expr: absent(up{job="critical_service"})
  for: 3m
  annotations:
    summary: "No up metric for critical_service"
    description: "Prometheus cannot find any target for job critical_service"

- alert: MetricAbsent
  expr: absent(node_cpu_seconds_total)
  for: 5m
  annotations:
    summary: "node_cpu_seconds_total metric disappears"
```

# 8. 告警规则调试

## 8.1 在 Prometheus UI 查看

访问 `http://prometheus:9090/alerts` 查看告警规则状态：

- **绿色（Inactive）**：规则正常，未触发
- **黄色（Pending）**：条件满足，等待 `for` 时间
- **红色（Firing）**：已触发，已发送告警

## 8.2 手动调试查询

```promql
# 测试告警表达式是否返回结果
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 80

# 查看规则评估结果
ALERTS{alertname="HighCPUUsage"}
```

## 8.3 验证配置文件

```bash
# 语法检查
promtool check rules rules/alerts.yml

# 单元测试告警规则
promtool test rules rules_test.yml
```

# 9. 告警规则最佳实践

## 9.1 告警命名规范

```
<Component><Condition> 或 <Condition><Component>

Good: HighCPUUsage, InstanceDown, HighErrorRate, DiskSpaceFull
Bad: alert1, cpu_alert, problem_with_server
```

## 9.2 Severity 分级

| 级别 | 响应时间 | 说明 |
|------|----------|------|
| critical | 立即 | 服务不可用、数据丢失 |
| warning | 工作时间 | 资源接近极限、潜在风险 |
| info | 记录 | 通知性信息 |

## 9.3 避免告警风暴

```yaml
# Bad：没有 for 持续时间，抖动的指标会频繁触发
expr: up == 0

# Good：持续一段时间才触发
expr: up == 0
for: 1m
```

## 9.4 包含 Runbook

```yaml
annotations:
  runbook_url: "https://wiki.example.com/runbooks/high-cpu"
```

# 10. promtool

## 10.1 安装

```bash
# prometheus 安装包自带 promtool
go install github.com/prometheus/prometheus/cmd/promtool@latest
```

## 10.2 常用命令

```bash
# 检查规则文件
promtool check rules alerts.yml

# 测试规则
promtool test rules test.yml

# 验证 prometheus 配置
promtool check config prometheus.yml
```
