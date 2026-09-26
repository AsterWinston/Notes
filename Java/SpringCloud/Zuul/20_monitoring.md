
# 1. 监控体系概览

Zuul 的监控体系覆盖网关的四个核心维度：

```
┌──────────────────────────────────────────────┐
│                  Zuul 监控                    │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 路由信息  │  │ 过滤器信息│  │ 调用链路  │   │
│  │ /routes  │  │ /filters │  │ Sleuth    │   │
│  └──────────┘  └──────────┘  └──────────┘   │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 健康检查  │  │ Hystrix  │  │ 性能指标  │   │
│  │ /health  │  │ 熔断指标 │  │ Micrometer│   │
│  └──────────┘  └──────────┘  └──────────┘   │
└──────────────────────────────────────────────┘
```

# 2. 内置端点

## 2.1 /actuator/routes

```yaml
management:
  endpoints:
    web:
      exposure:
        include: routes
```

```bash
# 获取所有路由信息
curl http://localhost:8080/actuator/routes

# 获取路由详情（包含每个路由的配置信息）
curl http://localhost:8080/actuator/routes/details
```

响应示例：

```json
{
    "/user/**": "user-service",
    "/order/**": "order-service",
    "/product/**": "product-service",
    "/external/**": "http://external-api.example.com:8080"
}
```

详情模式：

```json
{
    "user-service": {
        "id": "user-service",
        "fullPath": "/user/**",
        "location": "user-service",
        "prefix": "/user",
        "prefixStripped": true,
        "sensitiveHeaders": ["Cookie", "Set-Cookie", "Authorization"],
        "customSensitiveHeaders": false,
        "retryable": false
    }
}
```

## 2.2 /actuator/filters

```java
@Bean
public FilterConfigurationEndpoint filterConfigurationEndpoint() {
    return new FilterConfigurationEndpoint();
}
```

```bash
# 获取所有过滤器及类型
curl http://localhost:8080/actuator/filters

# 按类型筛选
curl http://localhost:8080/actuator/filters/pre
curl http://localhost:8080/actuator/filters/route
curl http://localhost:8080/actuator/filters/post
curl http://localhost:8080/actuator/filters/error
```

响应示例：

```json
{
    "pre": [
        {"class": "ServletDetectionFilter", "order": -3, "disabled": false},
        {"class": "Servlet30WrapperFilter", "order": -2, "disabled": false},
        {"class": "FormBodyWrapperFilter", "order": -1, "disabled": false},
        {"class": "DebugFilter", "order": 1, "disabled": false},
        {"class": "PreDecorationFilter", "order": 5, "disabled": false}
    ],
    "route": [
        {"class": "RibbonRoutingFilter", "order": 10, "disabled": false},
        {"class": "SimpleHostRoutingFilter", "order": 100, "disabled": false},
        {"class": "SendForwardFilter", "order": 500, "disabled": false}
    ],
    "post": [
        {"class": "SendResponseFilter", "order": 1000, "disabled": false}
    ],
    "error": [
        {"class": "SendErrorFilter", "order": 0, "disabled": false}
    ]
}
```

## 2.3 /actuator/health

```yaml
management:
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true  # 支持 Kubernetes liveness/readiness 探针
```

# 3. Hystrix 指标监控

## 3.1 开启 Hystrix Stream

```yaml
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

```bash
# Hystrix 实时指标流（SSE）
curl http://localhost:8080/actuator/hystrix.stream
```

配合 Hystrix Dashboard 可视化：

```bash
# 启动 Dashboard
java -jar hystrix-dashboard-1.5.18.jar

# 在 Dashboard 中监控流地址
# http://zuul-gateway:8080/actuator/hystrix.stream
```

## 3.2 Turbine 聚合监控

多实例环境下，通过 Turbine 聚合多个 Zuul 网关的 Hystrix 数据：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableTurbine
public class TurbineApplication {
    public static void main(String[] args) {
        SpringApplication.run(TurbineApplication.class, args);
    }
}
```

```yaml
turbine:
  app-config: zuul-gateway  # 要监控的应用名
  cluster-name-expression: new String("default")
  combine-host-port: true
```

## 3.3 Hystrix 关键指标

| 指标 | 含义 |
|------|------|
| `rollingCountSuccess` | 时间窗口内成功请求数 |
| `rollingCountFailure` | 时间窗口内失败请求数 |
| `rollingCountTimeout` | 时间窗口内超时请求数 |
| `rollingCountShortCircuited` | 时间窗口内短路拒绝数 |
| `rollingCountThreadPoolRejected` | 线程池拒绝数 |
| `currentConcurrentExecutionCount` | 当前并发请求数 |
| `latencyExecute_mean` | 平均执行延迟 |
| `latencyExecute_p99` | 99 分位延迟 |
| `isCircuitBreakerOpen` | 熔断器是否打开 |

# 4. Micrometer 指标集成

Spring Boot 2.x 使用 Micrometer 作为指标 (Metrics) 的标准门面，Zuul 通过 Micrometer 暴露指标：

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

## 4.1 自定义 Zuul 指标

```java
@Component
public class ZuulMetricsFilter extends ZuulFilter {

    private final MeterRegistry meterRegistry;
    private final Counter requestCounter;
    private final Timer requestTimer;

    public ZuulMetricsFilter(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;

        // 请求计数器（按 serviceId 分组）
        this.requestCounter = Counter.builder("zuul.requests.total")
            .description("Zuul 请求总数")
            .register(meterRegistry);

        // 请求计时器
        this.requestTimer = Timer.builder("zuul.requests.duration")
            .description("Zuul 请求耗时")
            .register(meterRegistry);
    }

    @Override
    public String filterType() { return "post"; }

    @Override
    public int filterOrder() { return 1100; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 记录请求计数
        String serviceId = (String) ctx.get("serviceId");
        requestCounter.increment();

        // 记录请求耗时
        Long startTime = (Long) ctx.get("startTime");
        if (startTime != null) {
            requestTimer.record(
                System.currentTimeMillis() - startTime,
                TimeUnit.MILLISECONDS);
        }

        // 按状态码和 serviceId 分组统计
        int statusCode = ctx.getResponseStatusCode();
        Counter.builder("zuul.responses")
            .tag("serviceId", serviceId != null ? serviceId : "unknown")
            .tag("status", String.valueOf(statusCode))
            .register(meterRegistry)
            .increment();

        return null;
    }
}
```

导出到 Prometheus 的指标示例：

```
# HELP zuul_requests_total Zuul 请求总数
# TYPE zuul_requests_total counter
zuul_requests_total 12345

# HELP zuul_requests_duration_seconds Zuul 请求耗时
# TYPE zuul_requests_duration_seconds summary
zuul_requests_duration_seconds_count 12345
zuul_requests_duration_seconds_sum 123.45
zuul_requests_duration_seconds{quantile="0.5"} 0.05
zuul_requests_duration_seconds{quantile="0.95"} 0.2
zuul_requests_duration_seconds{quantile="0.99"} 0.5

# HELP zuul_responses
# TYPE zuul_responses counter
zuul_responses{serviceId="user-service",status="200"} 5000
zuul_responses{serviceId="user-service",status="500"} 10
zuul_responses{serviceId="order-service",status="200"} 3000
```

# 5. 链路追踪（Sleuth + Zipkin）

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

```yaml
spring:
  sleuth:
    sampler:
      probability: 0.1  # 10% 采样率
  zipkin:
    base-url: http://zipkin-server:9411
```

Sleuth 会自动在 Zuul 网关的线程上下文中注入 TraceId 和 SpanId，并传递到后端服务。

## 5.1 自定义 TraceId 注入

```java
@Component
public class TraceIdFilter extends ZuulFilter {

    @Override
    public String filterType() { return "pre"; }

    @Override
    public int filterOrder() { return 3; }

    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 获取 Sleuth 生成的 TraceId
        String traceId = MDC.get("traceId");
        if (traceId != null) {
            // 注入到请求头，传递给后端服务
            ctx.addZuulRequestHeader("X-B3-TraceId", traceId);
            // 注入到响应头，方便客户端获取
            ctx.addZuulResponseHeader("X-Trace-Id", traceId);
        }

        return null;
    }
}
```

# 6. 日志与审计

## 6.1 访问日志

```java
@Component
public class AccessLogFilter extends ZuulFilter {

    private static final Logger ACCESS_LOG =
        LoggerFactory.getLogger("ACCESS_LOG");

    @Override
    public String filterType() { return "post"; }
    @Override
    public int filterOrder() { return 1100; }
    @Override
    public boolean shouldFilter() { return true; }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        Long startTime = (Long) ctx.get("startTime");
        long duration = startTime != null
            ? System.currentTimeMillis() - startTime : -1;

        // 标准 Nginx 风格访问日志
        ACCESS_LOG.info("{} {} {} {} {}ms",
            request.getRemoteAddr(),
            request.getMethod(),
            request.getRequestURI(),
            ctx.getResponseStatusCode(),
            duration);

        return null;
    }
}
```

## 6.2 Logback 配置

```xml
<!-- 访问日志单独输出 -->
<appender name="ACCESS_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/access.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>logs/access.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %msg%n</pattern>
    </encoder>
</appender>

<logger name="ACCESS_LOG" level="INFO" additivity="false">
    <appender-ref ref="ACCESS_FILE"/>
</logger>
```

# 7. 自定义 Actuator 端点

```java
@Component
@Endpoint(id = "zuul-stats")
public class ZuulStatsEndpoint {

    @ReadOperation
    public Map<String, Object> stats() {
        Map<String, Object> stats = new LinkedHashMap<>();
        stats.put("registeredFilters",
            FilterRegistry.instance().getAllFilters().size());
        stats.put("hystrixCircuitBreakers",
            getOpenCircuitBreakers());
        return stats;
    }

    private List<String> getOpenCircuitBreakers() {
        // 获取所有打开的熔断器
        // 通过 HystrixCommandMetrics 获取
        List<String> open = new ArrayList<>();
        for (HystrixCommandMetrics metrics :
                HystrixCommandMetrics.getInstances()) {
            if (metrics.getProperties()
                    .circuitBreakerEnabled().get()
                    && metrics.getCircuitBreaker().isOpen()) {
                open.add(metrics.getCommandKey().name());
            }
        }
        return open;
    }
}
```

# 8. Grafana + Prometheus 监控大盘

推荐的 Grafana 面板指标：

| 面板 | 指标 | 描述 |
|------|------|------|
| 请求 QPS | `zuul_requests_total` | 每秒请求数 |
| 请求延迟 | `zuul_requests_duration_seconds` | P50/P95/P99 延迟 |
| 响应状态码分布 | `zuul_responses{status=~".*"}` | 各状态码占比 |
| 熔断器状态 | `hystrix_circuit_breaker_open` | 各服务熔断状态 |
| 线程池状态 | `hystrix_thread_pool_*` | 活跃/排队/拒绝线程数 |
| JVM 指标 | `jvm_*` | 堆内存、GC 等 |
| 按服务分组 QPS | 带 serviceId 标签的计数器 | 每服务的请求分布 |
