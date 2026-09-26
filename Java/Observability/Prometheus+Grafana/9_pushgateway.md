
# 1. 什么是 Pushgateway

Pushgateway 是 Prometheus 生态中的一个**辅助组件**，允许临时或批处理任务通过**推送（Push）** 方式向 Prometheus 暴露指标。它作为指标的中转站：任务将指标推送到 Pushgateway，Prometheus 再从 Pushgateway 拉取。

## 1.1 架构

```
+----------------+     Push (HTTP)    +----------------+     Pull (HTTP)    +----------------+
| Batch Job      |------------------>|  Pushgateway   |<------------------|  Prometheus    |
| Cron Job       |   POST /metrics   |  :9091         |   GET /metrics    |  :9090         |
| Short-lived    |                   |                 |                   |                |
| Process        |                   +----------------+                   +----------------+
```

# 2. 适用场景

| 场景 | 说明 | 示例 |
|------|------|------|
| 批处理任务 | 任务执行完成后上报结果 | 数据 ETL 任务成功/失败计数 |
| 定时任务 | Cron Job 执行状态 | 备份任务持续时间 |
| 短生命周期任务 | 任务运行时间极短，来不及被 Pull | CI/CD 构建状态 |
| 网络隔离环境 | 目标网络不允许 Prometheus 直接访问 | DMZ 区域的指标 |

# 3. 不适用场景

| 场景 | 问题 | 推荐方案 |
|------|------|----------|
| 长期运行服务 | Pushgateway 不提供超时删除，旧指标残留 | 使用 Exporter Pull 方式 |
| 高可用指标 | Pushgateway 是单点 | 部署多个 Pushgateway 或使用 Exporter |
| 大量指标 | Pushgateway 将所有指标存于内存 | 使用 Exporter 分散指标 |

# 4. 安装与部署

## 4.1 二进制

```bash
wget https://github.com/prometheus/pushgateway/releases/download/v1.9.0/pushgateway-1.9.0.linux-amd64.tar.gz
tar xvf pushgateway-1.9.0.linux-amd64.tar.gz
cd pushgateway-1.9.0.linux-amd64
./pushgateway
```

## 4.2 Docker

```bash
docker run -d -p 9091:9091 --name pushgateway prom/pushgateway:v1.9.0
```

# 5. 推送指标

## 5.1 基本格式

```bash
# 推送单个指标
echo "my_batch_job_completed_total 1" | curl --data-binary @- http://localhost:9091/metrics/job/my_job

# 推送带标签的指标
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/my_job/instance/server1
# HELP batch_job_duration_seconds Batch job duration
# TYPE batch_job_duration_seconds gauge
batch_job_duration_seconds 42.5
batch_job_records_processed_total 1000
EOF
```

## 5.2 编程方式推送

**Python：**

```python
import requests
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

registry = CollectorRegistry()
g = Gauge('job_last_success_unixtime', 'Last time job succeeded', registry=registry)
g.set_to_current_time()

push_to_gateway('localhost:9091', job='my_batch_job', registry=registry)
```

**Go：**

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/push"
)

func main() {
    completionTime := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "db_backup_last_completion_timestamp_seconds",
        Help: "The last time DB backup completed.",
    })
    completionTime.SetToCurrentTime()

    pusher := push.New("http://localhost:9091", "db_backup").
        Collector(completionTime).
        Grouping("instance", "server1")

    if err := pusher.Add(); err != nil {
        panic(err)
    }
}
```

# 6. 分组机制

Pushgateway 使用 **Job + Grouping Labels** 来组织指标：

```
/metrics/job/<JOB>{/<LABEL_NAME>/<LABEL_VALUE>}
```

```bash
# 同一 job 的不同分组
/metrics/job/my_job/instance/server1
/metrics/job/my_job/instance/server2
/metrics/job/my_job/region/us-east-1
```

- 不同分组的指标互不干扰
- 相同分组的新推送会覆盖旧指标
- 分组标签也会附加到时间序列上

# 7. Prometheus 采集配置

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true  # 保留 Pushgateway 中已有的 job/instance 标签
    static_configs:
      - targets:
        - 'localhost:9091'
    metric_relabel_configs:
      # 过滤掉 pushgateway 的辅助指标
      - source_labels: [__name__]
        regex: 'pushgateway_.*'
        action: drop
```

## 7.1 honor_labels 的作用

- `true`：保留 Pushgateway 推送时设置的 job/instance 标签，覆盖 Prometheus 添加的
- `false`（默认）：用 Prometheus 的 job/instance 覆盖推送的值

建议对 Pushgateway 设置为 `true`，因为推送方设置的 job/instance 标识更准确。

# 8. 删除指标

## 8.1 API 方式

```bash
# 删除整个 Job
curl -X DELETE http://localhost:9091/metrics/job/my_job

# 删除 Job 的特定分组
curl -X DELETE http://localhost:9091/metrics/job/my_job/instance/server1
```

## 8.2 自动过期策略

Pushgateway **不提供内置的指标过期机制**，需要自行管理：

```bash
#!/bin/bash
# 清理超过 1 小时的旧指标
curl -s http://localhost:9091/metrics | grep -oP 'job="[^"]*"' | sort -u | while read job; do
    job_name=$(echo $job | cut -d'"' -f2)
    curl -s -X DELETE "http://localhost:9091/metrics/job/$job_name"
done
```

# 9. 最佳实践

## 9.1 始终设置 Instance 标签

```bash
# Bad：没有 instance 标签，多个推送无法区分
http://pushgateway:9091/metrics/job/my_job

# Good：使用 instance 区分不同的推送源
http://pushgateway:9091/metrics/job/my_job/instance/host1
```

## 9.2 推送时包含时间信息

```bash
# 推送 Unix 时间戳，便于监控任务是否按时执行
echo "my_cron_job_last_success_timestamp $(date +%s)" | \
  curl --data-binary @- http://pushgateway:9091/metrics/job/my_cron_job
```

## 9.3 配合 Alertmanager 做"指标消失"告警

```yaml
groups:
  - name: pushgateway_alerts
    rules:
      - alert: PushJobAbsent
        expr: absent(my_cron_job_last_success_timestamp{job="pushgateway"})
        for: 10m
        annotations:
          summary: "Cron job my_cron_job has not reported in 10 minutes"
```

## 9.4 每次推送完整状态

```bash
# 推送时覆盖所有指标，避免残留旧值
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/etl/instance/node1
# HELP etl_rows_processed Total rows processed
# TYPE etl_rows_processed counter
etl_rows_processed 5000
# HELP etl_errors_total Total ETL errors
# TYPE etl_errors_total counter
etl_errors_total 0
EOF
```

# 10. 限制与注意事项

## 10.1 单点故障

Pushgateway 是**有状态**的，重启后数据丢失。建议：

```yaml
# 持久化存储
docker run -d \
  -p 9091:9091 \
  -v pushgateway_data:/data \
  prom/pushgateway:v1.9.0 \
  --persistence.file=/data/pushgateway.data \
  --persistence.interval=5m
```

## 10.2 指标残留

Pushgateway 不会自动清理过期指标，旧指标会持续暴露。必须通过 DELETE API 或固定过期策略清理。

## 10.3 不能作为 Exporter 替代

Pushgateway 的定位是**临时中转**，不应将长期运行的服务改为 Push 模式。长期运行的服务应使用 Exporter Pull 方式。
