
# 1. 监控体系架构

```
┌──────────────────────────────────────────────────────┐
│                     监控数据流                          │
├──────────┬──────────┬──────────┬────────────────────┤
│  Provider │ Consumer │ Registry │                     │
│  ┌──────┐ │ ┌──────┐ │ ┌──────┐ │                     │
│  │Metrics│ │ │Metrics│ │ │ 状态  │ │                     │
│  └──┬───┘ │ └──┬───┘ │ └──┬───┘ │                     │
│     │     │    │     │    │     │                     │
│     └─────┼────┼─────┼────┘     │                     │
│           │    │     │          │                     │
│           ▼    ▼     ▼          │                     │
│     ┌──────────────────┐       │                     │
│     │  监控中心/Admin    │       │                     │
│     │  ┌───────────────┐│       │                     │
│     │  │Dubbo Admin    ││       │                     │
│     │  │  (控制台)      ││       │                     │
│     │  ├───────────────┤│       │                     │
│     │  │Prometheus     ││       │                     │
│     │  │  + Grafana    ││       │                     │
│     │  ├───────────────┤│       │                     │
│     │  │Tracing (追踪)  ││       │                     │
│     │  └───────────────┘│       │                     │
│     └──────────────────┘       │                     │
└──────────────────────────────────────────────────────┘
```

# 2. Dubbo Admin

## 2.1 功能

Dubbo Admin 是官方提供的可视化控制台，支持：

- 服务查询：按应用/接口/地址搜索服务
- 服务治理：权重调整、路由规则、动态配置
- 实例管理：Provider/Consumer 列表、健康状态
- 流量管理：条件路由、标签路由、灰度发布
- 监控统计：调用次数、成功率、RT 分布

## 2.2 部署

```bash
# Docker 部署
docker run -p 38080:38080 \
  -e admin.registry.address=nacos://127.0.0.1:8848 \
  -e admin.config-center=nacos://127.0.0.1:8848 \
  apache/dubbo-admin:latest
```

访问：`http://localhost:38080`

## 2.3 Provider 端配置

```yaml
dubbo:
  application:
    qos-enable: true      # 开启 QOS (Admin 依赖)
    qos-port: 22222
    qos-accept-foreign-ip: true
```

# 3. Metrics（指标监控）

## 3.1 指标维度

| 指标 | 说明 | 聚合维度 |
|------|------|---------|
| `dubbo.provider.requests.total` | Provider 请求总数 | application, interface, method |
| `dubbo.provider.requests.succeed` | Provider 成功请求数 | application, interface, method |
| `dubbo.provider.requests.failed` | Provider 失败请求数 | application, interface, method |
| `dubbo.provider.requests.processing` | Provider 处理中请求数 | application, interface, method |
| `dubbo.provider.requests.latency` | Provider 请求延迟 | application, interface, method |
| `dubbo.consumer.requests.total` | Consumer 请求总数 | application, interface, method |
| `dubbo.consumer.requests.succeed` | Consumer 成功请求数 | application, interface, method |
| `dubbo.consumer.requests.latency` | Consumer 请求延迟 | application, interface, method |
| `dubbo.threadpool.active` | 线程池活跃线程数 | application, protocol |
| `dubbo.threadpool.queue` | 线程池队列长度 | application, protocol |

## 3.2 指标采集源码

```java
public class MetricsCollector {
    private final Map<String, MethodMetric> methodMetrics = new ConcurrentHashMap<>();

    // MonitorFilter 中调用
    public void increment(String application, String service, 
            String method, boolean success, long elapsed) {
        
        MethodMetric metric = methodMetrics.computeIfAbsent(
            service + "." + method, 
            k -> new MethodMetric(application, service, method));
        
        metric.total.incrementAndGet();
        if (success) {
            metric.succeed.incrementAndGet();
        } else {
            metric.failed.incrementAndGet();
        }
        metric.lastElapsed.set(elapsed);
        metric.totalElapsed.addAndGet(elapsed);
    }
}
```

## 3.3 Prometheus 集成

```yaml
dubbo:
  metrics:
    protocol: prometheus
    port: 9090
    enable-jvm: true
```

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-metrics-prometheus</artifactId>
</dependency>
```

Prometheus 配置 (`prometheus.yml`)：

```yaml
scrape_configs:
  - job_name: 'dubbo'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['localhost:9090']
```

Grafana Dashboard 导入 ID: `Dubbo Metrics Dashboard`。

# 4. Tracing（链路追踪）

## 4.1 实现原理

Dubbo 通过 Filter 机制在请求的 attachments 中透传 traceId/spanId：

```java
@Activate(group = {PROVIDER, CONSUMER})
public class TracingFilter implements Filter {
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        // Consumer 端：创建 Span
        Span span = tracer.buildSpan(invocation.getMethodName());
        
        // 将 traceId/spanId 写入 attachments
        invocation.setAttachment("traceId", span.getTraceId());
        invocation.setAttachment("spanId", span.getSpanId());
        
        try (Scope scope = span.start()) {
            return invoker.invoke(invocation);
        } finally {
            span.finish();
        }
    }
}
```

## 4.2 集成 SkyWalking

```yaml
dubbo:
  tracing:
    enabled: true
    sampling:
      probability: 0.1
```

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
</dependency>
```

## 4.3 集成 Zipkin

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-tracing-brave</artifactId>
</dependency>
```

```yaml
dubbo:
  tracing:
    enabled: true
    exporter: zipkin
    endpoint: http://localhost:9411/api/v2/spans
```

# 5. 健康检查

## 5.1 Dubbo QOS 健康检查

```bash
# 检查 Provider 是否存活
curl http://localhost:22222/live

# 返回: true/false
```

## 5.2 Spring Boot Actuator 集成

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
```

```bash
curl http://localhost:8081/actuator/health

# 返回:
# {
#   "status": "UP",
#   "components": {
#     "dubbo": {
#       "status": "UP",
#       "details": {
#         "registry": "connected",
#         "provider": {
#           "count": 2,
#           "services": ["UserService", "OrderService"]
#         }
#       }
#     }
#   }
# }
```

## 5.3 K8s 健康检查

```yaml
livenessProbe:
  httpGet:
    path: /live
    port: 22222
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8081
  initialDelaySeconds: 10
  periodSeconds: 5
```

# 6. 日志与审计

## 6.1 访问日志

```yaml
dubbo:
  provider:
    accesslog: /tmp/dubbo-access.log    # Provider 访问日志
```

或通过注解：

```java
@DubboService(accesslog = "/logs/user-service.log")
public class UserServiceImpl implements UserService {}
```

日志格式：

```
[2024-01-15 10:30:00] 192.168.1.100:20880 -> org.example.UserService:getUser(Long) ["1"] 
  -> {"id":1,"name":"user-1","age":25}, cost 5ms
```

## 6.2 自定义日志 Filter

```java
@Activate(group = {PROVIDER, CONSUMER})
public class AuditLogFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        long start = System.currentTimeMillis();
        String traceId = MDC.get("traceId");
        
        try {
            Result result = invoker.invoke(invocation);
            long elapsed = System.currentTimeMillis() - start;
            
            // 写入审计日志
            auditLogger.info("traceId={} service={} method={} elapsed={} status=SUCCESS",
                traceId, invoker.getUrl().getServiceInterface(),
                invocation.getMethodName(), elapsed);
            
            return result;
        } catch (Exception e) {
            long elapsed = System.currentTimeMillis() - start;
            auditLogger.error("traceId={} service={} method={} elapsed={} status=FAIL error={}",
                traceId, invoker.getUrl().getServiceInterface(),
                invocation.getMethodName(), elapsed, e.getMessage());
            throw e;
        }
    }
}
```

# 7. 告警规则（Prometheus + AlertManager）

```yaml
groups:
  - name: dubbo-alerts
    rules:
      - alert: DubboHighErrorRate
        expr: |
          rate(dubbo_provider_requests_failed[5m]) 
          / rate(dubbo_provider_requests_total[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Dubbo error rate > 1% for {{ $labels.service }}"

      - alert: DubboThreadPoolExhausted
        expr: dubbo_threadpool_active / dubbo_threadpool_max > 0.9
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Thread pool usage > 90% on {{ $labels.instance }}"
```

# 8. 常用排查命令

```bash
# 查看 Dubbo 服务状态（QOS）
curl http://localhost:22222/ls

# 查看服务详情
curl http://localhost:22222/info

# 手动上下线
curl -X POST http://localhost:22222/online
curl -X POST http://localhost:22222/offline

# 查看线程池状态
curl http://localhost:22222/threadpool

# 查看 JVM 信息
curl http://localhost:22222/jvm

# 查看注册中心状态
curl http://localhost:22222/registry
```
