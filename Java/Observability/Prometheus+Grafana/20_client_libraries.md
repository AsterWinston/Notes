
# 1. 概述

Prometheus Client Libraries（客户端库）是官方提供的 SDK，用于在应用程序中**自定义暴露指标**。它们遵循统一的指标暴露格式，支持四种核心 Metric 类型。

## 1.1 官方支持的客户端库

| 语言 | 仓库 | 版本 |
|------|------|------|
| Go | `prometheus/client_golang` | v1.x |
| Java | `prometheus/client_java` | 0.x / 1.x |
| Python | `prometheus/client_python` | 0.x |
| Ruby | `prometheus/client_ruby` | 0.x |
| Rust | `prometheus/client_rust` | 0.x |
| .NET | `prometheus-net` | 社区维护 |

# 2. Go Client

## 2.1 安装

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
```

## 2.2 指标定义与注册

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // promauto 自动注册
    RequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "app_requests_total",
            Help: "Total number of requests.",
            ConstLabels: prometheus.Labels{"service": "my-app"},
        },
        []string{"method", "path", "status"},
    )

    RequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "app_request_duration_seconds",
            Help:    "Request duration in seconds.",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )

    ActiveRequests = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "app_active_requests",
            Help: "Number of currently active requests.",
        },
    )

    RequestSize = promauto.NewSummary(
        prometheus.SummaryOpts{
            Name:       "app_request_size_bytes",
            Help:       "Request size in bytes.",
            Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
        },
    )
)
```

## 2.3 HTTP 处理

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
    // 注册 /metrics 端点
    http.Handle("/metrics", promhttp.Handler())

    // 带自定义选项
    http.Handle("/metrics", promhttp.HandlerFor(
        prometheus.DefaultGatherer,
        promhttp.HandlerOpts{
            EnableOpenMetrics: true,
            MaxRequestsInFlight: 10,
            Timeout: 10 * time.Second,
        },
    ))

    http.ListenAndServe(":8080", nil)
}
```

## 2.4 Go Runtime 指标

```go
import "github.com/prometheus/client_golang/prometheus/collectors"

registry := prometheus.NewRegistry()
registry.MustRegister(collectors.NewGoCollector())
registry.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))
```

# 3. Java Client

## 3.1 安装

```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient</artifactId>
    <version>0.16.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_hotspot</artifactId>
    <version>0.16.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_httpserver</artifactId>
    <version>0.16.0</version>
</dependency>
```

## 3.2 指标定义

```java
import io.prometheus.client.Counter;
import io.prometheus.client.Gauge;
import io.prometheus.client.Histogram;
import io.prometheus.client.Summary;

public class AppMetrics {
    private static final Counter requests = Counter.build()
        .name("app_requests_total")
        .help("Total requests.")
        .labelNames("method", "path", "status")
        .register();

    private static final Gauge activeSessions = Gauge.build()
        .name("app_active_sessions")
        .help("Active sessions.")
        .register();

    private static final Histogram requestDuration = Histogram.build()
        .name("app_request_duration_seconds")
        .help("Request duration.")
        .labelNames("method", "path")
        .buckets(0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10)
        .register();

    private static final Summary responseSize = Summary.build()
        .name("app_response_size_bytes")
        .help("Response size.")
        .quantile(0.5, 0.05)
        .quantile(0.9, 0.01)
        .quantile(0.99, 0.001)
        .register();
}
```

## 3.3 JVM 指标

```java
import io.prometheus.client.hotspot.DefaultExports;
DefaultExports.initialize();

// 自动注册的 JVM 指标：
// jvm_memory_bytes_used, jvm_memory_bytes_max, jvm_memory_pool_bytes_*
// jvm_gc_collection_seconds, jvm_threads_current, jvm_classes_loaded
// jvm_info, jvm_version_info
```

## 3.4 HTTP Server

```java
import io.prometheus.client.exporter.HTTPServer;

public class MetricsServer {
    public static void main(String[] args) throws Exception {
        new HTTPServer(8080);
        // 应用逻辑...
    }
}
```

## 3.5 Java Client 1.x (新版本)

```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>prometheus-metrics-core</artifactId>
    <version>1.3.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>prometheus-metrics-instrumentation-jvm</artifactId>
    <version>1.3.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>prometheus-metrics-exporter-httpserver</artifactId>
    <version>1.3.0</version>
</dependency>
```

```java
import io.prometheus.metrics.core.metrics.Counter;
import io.prometheus.metrics.core.metrics.Gauge;
import io.prometheus.metrics.core.metrics.Histogram;
import io.prometheus.metrics.exporter.httpserver.HTTPServer;

Counter counter = Counter.builder()
    .name("app_requests_total")
    .help("Total requests")
    .labelNames("method", "path")
    .register();

counter.labelValues("GET", "/api").inc();
counter.labelValues("POST", "/api").inc(5);
```

# 4. Python Client

## 4.1 安装

```bash
pip install prometheus-client
```

## 4.2 指标定义

```python
from prometheus_client import Counter, Gauge, Histogram, Summary, start_http_server

# Counter
requests = Counter('app_requests_total', 'Total requests',
                   ['method', 'path', 'status'])

# Gauge
in_flight = Gauge('app_in_flight_requests', 'In-flight requests')

# Histogram
duration = Histogram('app_request_duration_seconds', 'Request duration',
                     buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10])

# Summary
response_size = Summary('app_response_size_bytes', 'Response size')

# 使用
requests.labels(method='GET', path='/api', status='200').inc()
in_flight.inc()
in_flight.dec()
duration.observe(0.25)
response_size.observe(1024)
```

## 4.3 多进程模式

```python
from prometheus_client import multi_process

# Gunicorn 等多进程模式
# 需要指定 Promeheus 多进程目录
import os
os.environ['PROMETHEUS_MULTIPROC_DIR'] = '/tmp/prometheus_multiproc'

from prometheus_client import Counter, Gauge, Histogram, Summary, generate_latest, CollectorRegistry, multiprocess

# 在 master 进程中
registry = CollectorRegistry()
multiprocess.MultiProcessCollector(registry)

# 在应用代码中
from prometheus_client import Counter
c = Counter('my_counter', 'Description')
c.inc()
```

## 4.4 Flask 集成

```python
from flask import Flask, request
from prometheus_client import Counter, Histogram, generate_latest
import time

app = Flask(__name__)

requests_total = Counter('http_requests_total', 'Total requests',
                         ['method', 'endpoint', 'http_status'])
request_duration = Histogram('http_request_duration_seconds', 'Duration',
                             ['method', 'endpoint'])

@app.before_request
def start_timer():
    request._start_time = time.time()

@app.after_request
def record_metrics(response):
    request_duration.labels(
        method=request.method,
        endpoint=request.path
    ).observe(time.time() - request._start_time)

    requests_total.labels(
        method=request.method,
        endpoint=request.path,
        http_status=response.status_code
    ).inc()
    return response

@app.route('/metrics')
def metrics():
    return generate_latest()
```

# 5. 自定义 Collector

## 5.1 Go

```go
type MyCollector struct {
    desc   *prometheus.Desc
    labels []string
}

func NewMyCollector() *MyCollector {
    return &MyCollector{
        desc: prometheus.NewDesc(
            "my_custom_metric",
            "My custom metric",
            []string{"label1"},
            nil,
        ),
    }
}

func (c *MyCollector) Describe(ch chan<- *prometheus.Desc) {
    ch <- c.desc
}

func (c *MyCollector) Collect(ch chan<- prometheus.Metric) {
    value := getMetricValue()
    ch <- prometheus.MustNewConstMetric(
        c.desc,
        prometheus.GaugeValue,
        value,
        "label_value",
    )
}
```

## 5.2 Java

```java
import io.prometheus.client.Collector;
import java.util.Arrays;
import java.util.List;

public class MyCollector extends Collector {
    @Override
    public List<MetricFamilySamples> collect() {
        List<MetricFamilySamples> samples = new ArrayList<>();
        samples.add(new MetricFamilySamples(
            "my_custom_metric",
            Type.GAUGE,
            "My custom metric",
            Arrays.asList(new Sample(
                "my_custom_metric",
                Arrays.asList("label1"),
                Arrays.asList("value1"),
                getMetricValue()
            ))
        ));
        return samples;
    }
}
```

# 6. 指标暴露格式

所有客户端库都遵循统一的 **OpenMetrics 格式**：

```
# HELP app_requests_total Total number of requests.
# TYPE app_requests_total counter
app_requests_total{method="GET",path="/api",status="200"} 1027
app_requests_total{method="POST",path="/api",status="201"} 45
# HELP app_request_duration_seconds Request duration.
# TYPE app_request_duration_seconds histogram
app_request_duration_seconds_bucket{method="GET",le="0.005"} 100
app_request_duration_seconds_bucket{method="GET",le="0.01"} 200
app_request_duration_seconds_bucket{method="GET",le="+Inf"} 1000
app_request_duration_seconds_sum{method="GET"} 450.5
app_request_duration_seconds_count{method="GET"} 1000
```

# 7. 最佳实践

## 7.1 标签使用

```go
// Good: 有限且有意义的标签
RequestsTotal.WithLabelValues("GET", "/api", "200").Inc()

// Bad: 包含高基数标签（请求 ID、用户 ID 等）
RequestsTotal.WithLabelValues(requestID, userID, ip).Inc()

// Bad: 标签值动态且无限
RequestsTotal.WithLabelValues(fmt.Sprintf("/api/users/%d", userID)).Inc()
```

## 7.2 性能

```go
// Bad: 每次请求创建标签（性能差）
counter.WithLabelValues(r.Method, r.URL.Path).Inc()

// Good: 批量注册常见路径
var pathLabels = map[string]string{
    "/api/users":  "users",
    "/api/orders": "orders",
}
if label, ok := pathLabels[r.URL.Path]; ok {
    counter.WithLabelValues(r.Method, label).Inc()
}
```

## 7.3 指标注册

- 使用 `promauto` 自动注册，避免忘记注册导致采集不到数据
- 使用 `MustRegister` 确保启动时失败而不是运行时静默失败
- 避免重复注册（同一指标名注册两次会 panic）
