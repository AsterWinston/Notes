
# 1. 监控体系概述

Spring Cloud Gateway 的可观测性包括三个支柱：指标（Metrics）、日志（Logging）、追踪（Tracing）。

```
可观测性
    ├── Metrics（指标）
    │       ├── 请求量、成功率、延迟
    │       ├── 系统资源（连接池、线程）
    │       └── 业务指标（限流、熔断状态）
    │
    ├── Logging（日志）
    │       ├── 请求访问日志
    │       ├── 错误日志
    │       └── 调试日志
    │
    └── Tracing（追踪）
            ├── 分布式链路追踪（Sleuth/Zipkin）
            ├── 请求全链路耗时分析
            └── 服务依赖关系
```

# 2. GatewayMetricsFilter 源码分析

## 2.1 核心实现

```java
public class GatewayMetricsFilter implements GlobalFilter, Ordered {

    private final MeterRegistry meterRegistry;
    private final GatewayMetricsProperties properties;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 记录请求开始
        exchange.getAttributes().put(GATEWAY_METRICS_START_TIME, System.nanoTime());

        return chain.filter(exchange).then(Mono.defer(() -> {
            Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);

            // 构建指标标签
            List<Tag> tags = new ArrayList<>();
            tags.add(Tag.of("routeId", route != null ? route.getId() : "unknown"));
            tags.add(Tag.of("routeUri", route != null
                ? route.getUri().toString() : "unknown"));
            tags.add(Tag.of("outcome", getOutcome(exchange)));
            tags.add(Tag.of("status", getStatus(exchange)));
            tags.add(Tag.of("httpStatusCode",
                String.valueOf(getHttpStatusCode(exchange))));
            tags.add(Tag.of("httpMethod",
                exchange.getRequest().getMethod().name()));

            // 路径标签（可选，启用后可能导致高基数）
            if (properties.isTagsPathEnabled()) {
                tags.add(Tag.of("path",
                    exchange.getRequest().getPath().value()));
            }

            // 记录耗时
            Long startTime = exchange.getAttribute(
                GATEWAY_METRICS_START_TIME);
            if (startTime != null) {
                long elapsed = System.nanoTime() - startTime;
                // 注册 Timer 指标
                Timer.SAMPLE sample = Timer.start(meterRegistry);
                sample.stop(Timer.builder("spring.cloud.gateway.requests")
                    .tags(tags)
                    .description("Gateway requests")
                    .register(meterRegistry));
            }

            return Mono.empty();
        }));
    }

    @Override
    public int getOrder() {
        return 0;  // 在过滤器链中间执行
    }
}
```

# 3. Metrics 指标

## 3.1 配置

```yaml
spring:
  cloud:
    gateway:
      metrics:
        enabled: true
        tags:
          path:
            enabled: true    # 是否包含路径标签（谨慎在高基数场景使用）

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

## 3.2 核心指标

| 指标名 | 类型 | 标签 | 说明 |
|--------|------|------|------|
| `spring.cloud.gateway.requests` | Timer | routeId, routeUri, outcome, status, httpStatusCode, httpMethod, path | 请求耗时分布 |
| `gateway.requests.count` | Counter | routeId | 请求计数 |
| `gateway.requests.success` | Counter | routeId | 成功计数 |
| `gateway.requests.failure` | Counter | routeId | 失败计数 |
| `resilience4j.circuitbreaker.state` | Gauge | name | 断路器状态 |
| `resilience4j.circuitbreaker.calls` | Counter | name, kind | 调用计数 |
| `resilience4j.ratelimiter.available` | Gauge | name | 限流器可用令牌 |

## 3.3 自定义指标

```java
@Component
public class CustomGatewayMetrics {

    private final MeterRegistry meterRegistry;
    private final Counter requestCounter;
    private final DistributionSummary responseSizeSummary;

    public CustomGatewayMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;

        // 自定义计数器
        this.requestCounter = Counter.builder("gateway.custom.requests")
            .description("Total custom gateway requests")
            .tag("application", SpringContextHolder.getApplicationName())
            .register(meterRegistry);

        // 自定义分布摘要
        this.responseSizeSummary = DistributionSummary.builder("gateway.response.size")
            .description("Response size distribution")
            .baseUnit("bytes")
            .publisherPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
    }

    public void recordRequest(String routeId, HttpStatus status, long duration, long responseSize) {
        requestCounter.increment();

        Timer.builder("gateway.custom.request.duration")
            .tags("routeId", routeId, "status", String.valueOf(status.value()))
            .register(meterRegistry)
            .record(Duration.ofMillis(duration));

        responseSizeSummary.record(responseSize);
    }
}

// 在自定义过滤器中记录指标
@Component
public class MetricsCollectingFilter implements GlobalFilter, Ordered {
    private final CustomGatewayMetrics metrics;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        String routeId = route != null ? route.getId() : "unknown";
        long start = System.currentTimeMillis();

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long duration = System.currentTimeMillis() - start;
            HttpStatus status = exchange.getResponse().getStatusCode();
            // 估算响应大小
            long responseSize = estimateResponseSize(exchange);
            metrics.recordRequest(routeId, status, duration, responseSize);
        }));
    }

    @Override
    public int getOrder() {
        return OrderedGatewayFilter.LOWEST_PRECEDENCE;
    }
}
```

# 4. Prometheus + Grafana

## 4.1 Prometheus 配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-cloud-gateway'
    scrape_interval: 10s
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - 'localhost:8080'
```

## 4.2 Grafana Dashboard

推荐监控面板指标：

```
1. 请求总览
   - 总请求速率（QPS）
   - 各路由请求速率
   - 5xx / 4xx 比例（折线图）

2. 延迟分析
   - P50 / P95 / P99 延迟（热图或折线图）
   - 各路由平均延迟

3. 熔断器状态
   - 各断路器状态（状态机图）
   - 熔断打开次数

4. 限流情况
   - 被限流请求数
   - 限流器令牌使用率

5. 系统资源
   - 连接池使用率
   - EventLoop 线程状态
```

## 4.3 PromQL 查询示例

```promql
# 总 QPS
rate(spring_cloud_gateway_requests_seconds_count[1m])

# P99 延迟
histogram_quantile(0.99,
  rate(spring_cloud_gateway_requests_seconds_bucket[1m]))

# 各路由请求量
sum by (routeId) (
  rate(spring_cloud_gateway_requests_seconds_count[1m])
)

# 错误率
sum(rate(spring_cloud_gateway_requests_seconds_count{
  outcome!="SUCCESSFUL"}[1m]))
/ sum(rate(spring_cloud_gateway_requests_seconds_count[1m]))
* 100

# 限流数
rate(gateway_requests_denied_total[1m])
```

# 5. 分布式链路追踪

## 5.1 Spring Cloud Sleuth 集成

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

```yaml
spring:
  sleuth:
    enabled: true
    sampler:
      probability: 1.0           # 采样率（生产环境建议 0.1~0.5）
    web:
      skip-pattern: /actuator/.*  # 跳过 Actuator 端点
    reactor:
      enabled: true               # 开启 Reactor 上下文传播
```

## 5.2 Micrometer Tracing（替代 Sleuth）

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: gateway-service

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"

management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0

  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

## 5.3 自定义 Span

```java
@Component
public class TracingGlobalFilter implements GlobalFilter, Ordered {

    private final Tracer tracer;

    public TracingGlobalFilter(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 创建自定义 Span
        Span span = tracer.nextSpan().name("custom-gateway-filter").start();

        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            span.tag("routeId", extractRouteId(exchange));
            span.tag("method", exchange.getRequest().getMethod().name());
            span.tag("path", exchange.getRequest().getPath().value());

            return chain.filter(exchange)
                .doFinally(signalType -> {
                    span.tag("status",
                        String.valueOf(exchange.getResponse().getStatusCode()));
                    span.finish();
                });
        }
    }

    @Override
    public int getOrder() {
        return OrderedGatewayFilter.HIGHEST_PRECEDENCE + 100;
    }
}
```

# 6. 日志监控

## 6.1 访问日志

```yaml
logging:
  level:
    org.springframework.cloud.gateway: INFO
    reactor.netty: INFO
    # 开启请求体日志（调试用）
    org.springframework.cloud.gateway.filter.ratelimit: DEBUG

# 开启 Netty 访问日志（需在 JVM 参数中开启）
# -Dreactor.netty.http.server.accessLogEnabled=true
```

## 6.2 自定义访问日志格式

```java
@Component
@Order(-150)
public class AccessLogGlobalFilter implements GlobalFilter {

    private static final Logger accessLog = LoggerFactory.getLogger("ACCESS_LOG");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long elapsed = System.currentTimeMillis() - startTime;

            // 收集日志信息
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();
            Route route = exchange.getAttribute(
                ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);

            // 构建结构化日志
            Map<String, Object> logData = new LinkedHashMap<>();
            logData.put("timestamp", Instant.now().toString());
            logData.put("method", request.getMethod().name());
            logData.put("path", request.getPath().value());
            logData.put("query", request.getURI().getRawQuery());
            logData.put("status", response.getStatusCode());
            logData.put("duration", elapsed + "ms");
            logData.put("route", route != null ? route.getId() : "unknown");
            logData.put("ip", request.getRemoteAddress()
                .getAddress().getHostAddress());
            logData.put("userAgent", request.getHeaders()
                .getFirst("User-Agent"));

            accessLog.info("{}", new ObjectMapper().writeValueAsString(logData));
        }));
    }
}
```

## 6.3 日志配置示例（logback-spring.xml）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>

    <!-- 访问日志 -->
    <appender name="ACCESS_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/access.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/access.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>

    <logger name="ACCESS_LOG" level="INFO" additivity="false">
        <appender-ref ref="ACCESS_LOG"/>
    </logger>
</configuration>
```

# 7. 健康检查

## 7.1 Actuator 配置

```yaml
management:
  endpoint:
    health:
      show-details: when-authorized       # always / never / when-authorized
      probes:
        enabled: true                     # Kubernetes 探针支持
  health:
    defaults:
      enabled: true
    redis:
      enabled: true                       # Redis 健康检查
    circuitbreakers:
      enabled: true                       # 断路器健康检查
```

## 7.2 自定义健康指标

```java
@Component
public class GatewayHealthIndicator implements HealthIndicator {

    private final RouteLocator routeLocator;
    private final AtomicInteger healthyRouteCount = new AtomicInteger(0);

    @Override
    public Health health() {
        int routeCount = healthyRouteCount.get();
        return Health.up()
            .withDetail("routes", routeCount)
            .withDetail("status", routeCount > 0 ? "UP" : "NO_ROUTES")
            .build();
    }

    @EventListener
    public void onRefreshRoutesEvent(RefreshRoutesEvent event) {
        routeLocator.getRoutes()
            .count()
            .subscribe(healthyRouteCount::set);
    }
}
```

# 8. 最佳实践

## 8.1 监控配置清单

```yaml
# 生产环境推荐配置
spring:
  cloud:
    gateway:
      metrics:
        enabled: true
        tags:
          path:
            enabled: false      # 高基数场景禁用路径标签

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,gateway
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      instance: ${spring.cloud.client.hostname}:${spring.application.instance_id:${random.value}}
```

## 8.2 监控告警规则

```yaml
# Prometheus AlertManager 告警规则示例
groups:
  - name: gateway-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(spring_cloud_gateway_requests_seconds_count{
            outcome!="SUCCESSFUL"}[5m]))
          / sum(rate(spring_cloud_gateway_requests_seconds_count[5m]))
          > 0.05
        for: 5m
        annotations:
          summary: "Gateway error rate > 5%"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(spring_cloud_gateway_requests_seconds_bucket[5m]))
          > 2
        for: 5m
        annotations:
          summary: "Gateway P95 latency > 2s"

      - alert: CircuitBreakerOpen
        expr: resilience4j_circuitbreaker_state{state="open"} > 0
        for: 1m
        annotations:
          summary: "Circuit breaker is open"
```

## 8.3 注意事项

1. **采样率**：全量采样（1.0）会增加系统开销，生产环境建议 0.1~0.5
2. **基数控制**：路径标签（path tag）可能导致高基数问题，谨慎启用
3. **日志轮转**：访问日志配置合理的轮转策略（按天或按大小）
4. **告警阈值**：根据业务特点设置合理的告警阈值，避免告警风暴
5. **存储规划**：Metrics 和 Tracing 数据需要规划存储周期和容量
