
# 1. 概述

应用监控指的是将**应用程序的内部状态**（请求量、延迟、错误率、资源使用等）暴露为 Prometheus 指标，实现应用级别的可观测性。Prometheus 提供多种语言的 Client Library，支持 Java、Go、Python、Ruby、Rust 等。

# 2. Java 应用监控

## 2.1 依赖引入

```xml
<!-- Maven -->
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
    <artifactId>simpleclient_servlet</artifactId>
    <version>0.16.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_httpserver</artifactId>
    <version>0.16.0</version>
</dependency>
```

```groovy
// Gradle
implementation 'io.prometheus:simpleclient:0.16.0'
implementation 'io.prometheus:simpleclient_hotspot:0.16.0'
implementation 'io.prometheus:simpleclient_servlet:0.16.0'
```

## 2.2 基本使用

```java
import io.prometheus.client.Counter;
import io.prometheus.client.Gauge;
import io.prometheus.client.Histogram;
import io.prometheus.client.Summary;
import io.prometheus.client.exporter.HTTPServer;

public class AppMetrics {
    // Counter
    static final Counter requests = Counter.build()
        .name("app_http_requests_total")
        .help("Total HTTP requests.")
        .labelNames("method", "path", "status")
        .register();

    // Gauge
    static final Gauge activeSessions = Gauge.build()
        .name("app_active_sessions")
        .help("Current active sessions.")
        .register();

    // Histogram
    static final Histogram requestDuration = Histogram.build()
        .name("app_request_duration_seconds")
        .help("Request duration.")
        .labelNames("method", "path")
        .buckets(0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10)
        .register();

    // Summary
    static final Summary responseSize = Summary.build()
        .name("app_response_size_bytes")
        .help("Response size.")
        .quantile(0.5, 0.05)
        .quantile(0.9, 0.01)
        .quantile(0.99, 0.001)
        .register();
}
```

## 2.3 JVM 监控

```java
// 自动注册 JVM 指标
import io.prometheus.client.hotspot.DefaultExports;
DefaultExports.initialize();

// 暴露的 JVM 指标：
// jvm_memory_bytes_used{area="heap|nonheap"}
// jvm_gc_collection_seconds_sum{gc="G1 Young Generation"}
// jvm_threads_current
// jvm_classes_loaded
// jvm_info
```

## 2.4 Spring Boot 集成

Spring Boot Actuator 已原生支持 Prometheus：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: my-app
```

## 2.5 Spring Boot 自定义指标

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Component;

@Component
public class CustomMetrics {
    private final Counter orderCounter;
    private final Timer orderTimer;

    public CustomMetrics(MeterRegistry registry) {
        orderCounter = Counter.builder("app_orders_created_total")
            .description("Total orders created")
            .tag("status", "pending")
            .register(registry);

        orderTimer = Timer.builder("app_order_processing_seconds")
            .description("Order processing time")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public void recordOrder(String status) {
        orderCounter.increment();
        orderCounter.tag("status", status).increment();
    }

    public Timer.Sample startTimer() {
        return Timer.start();
    }

    public void stopTimer(Timer.Sample sample) {
        sample.stop(orderTimer);
    }
}
```

# 3. Go 应用监控

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests.",
        },
        []string{"method", "path", "status"},
    )

    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration.",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )

    activeRequests = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_active_requests",
            Help: "Current active requests.",
        },
    )
)

func main() {
    // 注册 Go runtime 指标
    // go_memstats_alloc_bytes, go_goroutines, go_gc_duration_seconds 等
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

## 3.1 Go 中间件模式

```go
func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        activeRequests.Inc()
        defer activeRequests.Dec()

        timer := prometheus.NewTimer(requestDuration.WithLabelValues(r.Method, r.URL.Path))
        defer timer.ObserveDuration()

        next.ServeHTTP(w, r)
        requestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
    })
}
```

# 4. Python 应用监控

```python
from prometheus_client import start_http_server, Counter, Gauge, Histogram, Summary
import time
import random

# 定义指标
requests = Counter('app_requests_total', 'Total requests', ['method', 'endpoint'])
in_flight = Gauge('app_in_flight_requests', 'In-flight requests')
duration = Histogram('app_request_duration_seconds', 'Request duration',
                     buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5])

@duration.time()
def handle_request():
    in_flight.inc()
    try:
        requests.labels(method='GET', endpoint='/api').inc()
        time.sleep(random.random())
    finally:
        in_flight.dec()

if __name__ == '__main__':
    start_http_server(8000)
    while True:
        handle_request()
        time.sleep(0.1)
```

## 4.1 Flask 集成

```python
from flask import Flask, request
from prometheus_client import Counter, Histogram, generate_latest
import time

app = Flask(__name__)

requests_total = Counter('http_requests_total', 'Total requests',
                         ['method', 'endpoint', 'status'])
request_duration = Histogram('http_request_duration_seconds', 'Request duration',
                             ['method', 'endpoint'])

@app.before_request
def before_request():
    request.start_time = time.time()

@app.after_request
def after_request(response):
    request_duration.labels(
        method=request.method,
        endpoint=request.path
    ).observe(time.time() - request.start_time)

    requests_total.labels(
        method=request.method,
        endpoint=request.path,
        status=response.status_code
    ).inc()
    return response

@app.route('/metrics')
def metrics():
    return generate_latest()
```

# 5. JMX Exporter（Java 中间件监控）

JMX Exporter 通过 Java Agent 方式嵌入目标 JVM 应用，将 JMX MBeans 暴露为 Prometheus 指标。

## 5.1 配置

```yaml
# jmx_exporter_config.yml
startDelaySeconds: 0
ssl: false
rules:
  - pattern: 'java.lang<type=Memory><>HeapMemoryUsage'
    name: jvm_memory_heap_bytes
    type: GAUGE
    labels:
      area: heap
  - pattern: 'java.lang<type=Threading><>ThreadCount'
    name: jvm_threads_current
    type: GAUGE
  - pattern: 'Catalina<type=GlobalRequestProcessor, name=\"(\w+)\"><>(\w+)'
    name: tomcat_$2_total
    type: COUNTER
    labels:
      worker: "$1"
  - pattern: 'org.apache.tomcat<type=ThreadPool, name=\"(\w+)\"><>(\w+)'
    name: tomcat_threadpool_$2
    type: GAUGE
    labels:
      pool: "$1"
```

## 5.2 启动

```bash
java -javaagent:jmx_prometheus_javaagent.jar=8080:jmx_exporter_config.yml \
     -jar myapp.jar
```

# 6. 应用监控指标设计

## 6.1 RED 方法

| 指标 | 类型 | RED 对应 |
|------|------|----------|
| `http_requests_total` | Counter | Rate（请求速率） |
| `http_request_duration_seconds` | Histogram | Errors（错误率） |
| `http_requests_errors_total` | Counter | Duration（延迟） |

## 6.2 USE 方法（资源型应用）

| 指标 | 类型 | USE 对应 |
|------|------|----------|
| `db_connections_utilization` | Gauge | Utilization |
| `queue_size` | Gauge | Saturation |
| `error_count` | Counter | Errors |

## 6.3 典型应用指标集合

```java
// Web 服务
http_requests_total{method, path, status}
http_request_duration_seconds{method, path}
http_request_size_bytes{method, path}
http_response_size_bytes{method, path}
http_active_requests

// 数据库
db_queries_total{query, status}
db_query_duration_seconds{query}
db_connections_active
db_connections_idle
db_connections_max

// 消息队列
messages_published_total{topic}
messages_consumed_total{topic, consumer}
messages_pending{topic}
consumer_lag{topic, partition}

// 缓存
cache_hits_total{cache}
cache_misses_total{cache}
cache_size_bytes{cache}
```
