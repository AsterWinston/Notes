
# 1. 什么是 Exporter

Exporter 是 Prometheus 生态中负责**暴露指标数据的 Agent 程序**。它将目标系统的内部状态转换为 Prometheus 可以采集的 Metrics 格式，实现"任何系统都可以被 Prometheus 监控"。

## 1.1 Exporter 工作原理

```
Target System          Exporter                 Prometheus
+----------+   调用   +-----------+    HTTP    +------------+
|  App     |------->|  Metrics  |<----------|  Scraper   |
|  OS      |   API   |   Endpoint|  GET      |  Pull      |
|  DB      |         | :9100     |  /metrics |  Metrics   |
+----------+         +-----------+           +------------+
```

# 2. Exporter 分类

| 类别 | 实例 | 说明 |
|------|------|------|
| 官方 Exporter | Node Exporter, Blackbox Exporter, Consul Exporter | Prometheus 官方维护 |
| 社区 Exporter | Redis Exporter, MySQL Exporter, PostgreSQL Exporter | 社区维护，CNCF 生态 |
| 集成 Exporter | JMX Exporter, Kafka Exporter | 集成到特定中间件 |
| 自定义 Exporter | 自研应用指标 | 基于 Client Library 开发 |

# 3. 常用 Exporter 列表

| Exporter | 默认端口 | 监控目标 |
|----------|----------|----------|
| node_exporter | 9100 | Linux/Windows 主机指标 |
| blackbox_exporter | 9115 | HTTP/HTTPS/TCP/ICMP 探测 |
| mysqld_exporter | 9104 | MySQL 性能指标 |
| postgres_exporter | 9187 | PostgreSQL 性能指标 |
| redis_exporter | 9121 | Redis 性能指标 |
| mongodb_exporter | 9216 | MongoDB 性能指标 |
| nginx_exporter | 9113 | Nginx 指标（需搭配 stub_status） |
| nginx-lb-exporter | 8080 | Nginx Plus / OpenResty |
| haproxy_exporter | 9101 | HAProxy 指标 |
| kafka_exporter | 9308 | Kafka 指标 |
| jmx_exporter | 任意 | JVM 和 Java 应用指标 |
| elasticsearch_exporter | 9108 | Elasticsearch 指标 |
| consul_exporter | 9107 | Consul 集群指标 |
| etcd_exporter | 9000 | etcd 指标 |

# 4. Exporter 开发

## 4.1 使用 Go Client 开发

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // 定义 Counter 指标
    pingCounter = prometheus.NewCounter(prometheus.CounterOpts{
        Name: "ping_requests_total",
        Help: "Total number of ping requests.",
    })

    // 定义 Gauge 指标
    queueSize = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "queue_size",
        Help: "Current queue size.",
    })

    // 定义 Histogram 指标
    requestDuration = prometheus.NewHistogram(prometheus.HistogramOpts{
        Name:    "request_duration_seconds",
        Help:    "Request duration in seconds.",
        Buckets: prometheus.DefBuckets,
    })
)

func init() {
    prometheus.MustRegister(pingCounter)
    prometheus.MustRegister(queueSize)
    prometheus.MustRegister(requestDuration)
}

func handler(w http.ResponseWriter, r *http.Request) {
    pingCounter.Inc()
    duration := prometheus.NewTimer(requestDuration)
    defer duration.ObserveDuration()

    queueSize.Set(float64(getQueueLen()))
    w.Write([]byte("pong"))
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.HandleFunc("/ping", handler)
    http.ListenAndServe(":8080", nil)
}
```

## 4.2 使用 Python Client 开发

```python
from prometheus_client import start_http_server, Counter, Gauge, Histogram
import random
import time

# 定义指标
requests_total = Counter('http_requests_total', 'Total HTTP Requests',
                         ['method', 'endpoint'])
queue_size = Gauge('queue_size', 'Current queue size')
request_duration = Histogram('request_duration_seconds',
                             'Request duration in seconds',
                             buckets=[0.1, 0.5, 1.0, 2.5, 5.0])

def process_request():
    with request_duration.time():
        requests_total.labels(method='GET', endpoint='/api').inc()
        queue_size.set(random.randint(0, 100))
        time.sleep(random.random())

if __name__ == '__main__':
    start_http_server(8080)
    while True:
        process_request()
        time.sleep(1)
```

## 4.3 使用 Java Client 开发

```java
import io.prometheus.client.Counter;
import io.prometheus.client.Gauge;
import io.prometheus.client.Histogram;
import io.prometheus.client.exporter.HTTPServer;

public class MyExporter {
    static final Counter requests = Counter.build()
        .name("http_requests_total")
        .help("Total HTTP requests.")
        .labelNames("method", "endpoint")
        .register();

    static final Gauge queueSize = Gauge.build()
        .name("queue_size")
        .help("Current queue size.")
        .register();

    static final Histogram requestDuration = Histogram.build()
        .name("request_duration_seconds")
        .help("Request duration.")
        .buckets(0.1, 0.5, 1.0, 2.5, 5.0)
        .register();

    public static void main(String[] args) throws Exception {
        new HTTPServer(8080);
        // ... application code
        requests.labels("GET", "/api").inc();
        queueSize.set(42);
    }
}
```

# 5. Exporter 最佳实践

## 5.1 指标命名规范

```go
// Good
app_requests_total{method="GET"}
app_request_duration_seconds
app_db_connections_active

// Bad
app.requests.total      // 使用点号而非下划线
app_requests            // 缺少单位
num_of_connections      // 使用驼峰
app_request_duration    // 缺少单位 seconds
```

## 5.2 单位规范

| 度量 | 单位后缀 | 示例 |
|------|----------|------|
| 时间 | `_seconds` | `request_duration_seconds` |
| 大小 | `_bytes` | `memory_usage_bytes` |
| 计数 | `_total` | `requests_total` |
| 比率 | `_ratio` | `cpu_usage_ratio` |
| 温度 | `_celsius` | `cpu_temperature_celsius` |

## 5.3 标签设计

```go
// Good：有限、有意义的标签
requests_total{method="GET", endpoint="/api", status="200"}

// Bad：高基数标签
requests_total{user_id="12345", session_id="abc-def", ip="10.0.0.1"}

// Bad：标签包含动态值
requests_total{path="/api/v1/users/12345"}  // path 中带 user_id
```

## 5.4 性能考虑

- **避免高基数**：标签值的组合数应控制在 100,000 以内
- **合理设置采集间隔**：根据指标变化频率调整 `scrape_interval`
- **使用 Summary 替代 Histogram** 仅在不需要跨实例聚合时
- **Histogram Bucket 越少越好**：通常 10-15 个 bucket 足够
- **避免指标值频繁变化**：Gauge 值不应每毫秒变化

# 6. Exporter 安全

## 6.1 启用认证

```yaml
scrape_configs:
  - job_name: 'secured'
    metrics_path: /metrics
    static_configs:
      - targets: ['localhost:8080']
    basic_auth:
      username: 'admin'
      password: 'password'
```

## 6.2 限定监听地址

```bash
# 只监听内网接口
./node_exporter --web.listen-address=192.168.1.1:9100
```

## 6.3 禁止远程访问

```bash
# 只监听本地地址
./node_exporter --web.listen-address=127.0.0.1:9100
```
