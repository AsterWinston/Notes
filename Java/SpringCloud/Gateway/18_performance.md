
# 1. Netty 调优

## 1.1 线程模型配置

Spring Cloud Gateway 基于 Reactor Netty，默认 EventLoop 线程数为 CPU 核心数 × 2。

```yaml
# Netty 线程模型配置
spring:
  cloud:
    gateway:
      httpclient:
        # HTTP 客户端线程配置
        pool:
          type: fixed                          # 固定连接池
          max-connections: 500                 # 最大连接数
          acquire-timeout: 45000               # 获取连接超时
          max-idle-time: 3000                  # 最大空闲时间
          max-life-time: 60000                 # 连接最大生命周期
          eviction-interval: 3000              # 清理检查间隔
          metrics: true                        # 连接池指标

      httpserver:
        # HTTP 服务端配置
        max-initial-line-length: 4096
        max-header-size: 16384
```

## 1.2 Netty 原生参数

```java
@Configuration
public class NettyConfiguration {

    @Bean
    public ReactorResourceFactory reactorResourceFactory() {
        ReactorResourceFactory factory = new ReactorResourceFactory();

        // 自定义 EventLoop 线程数
        factory.setLoopResources(LoopResources.create(
            "gateway-eventloop",
            Runtime.getRuntime().availableProcessors(),  // boss 线程
            Runtime.getRuntime().availableProcessors() * 2,  // work 线程
            true));

        // 自定义连接池
        factory.setConnectionProvider(
            ConnectionProvider.builder("gateway-pool")
                .maxConnections(500)
                .pendingAcquireTimeout(Duration.ofSeconds(45))
                .maxIdleTime(Duration.ofSeconds(3))
                .maxLifeTime(Duration.ofSeconds(60))
                .evictInBackground(Duration.ofSeconds(3))
                .metrics(true)
                .build());

        return factory;
    }

    @Bean
    public HttpClient httpClient(ReactorResourceFactory resourceFactory) {
        return HttpClient.create(resourceFactory.getConnectionProvider())
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
            .option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.SO_REUSEADDR, true)
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(10)));
    }
}
```

## 1.3 EventLoop 阻塞检测

```yaml
# 开启 EventLoop 阻塞检测（调试用）
reactor:
  netty:
    workGroup:
      eventLoop:
        block-detection: true
        block-detection-threshold: 5000ms
```

# 2. 连接池优化

## 2.1 连接池类型选择

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          # ELASTIC：弹性池（默认）。按需创建连接，空闲 60s 后回收
          #        优点：连接数自动伸缩；缺点：高并发时连接数失控
          # FIXED：固定池。限制最大连接数，超出排队等待
          #        优点：连接数可控；缺点：需合理配置 max-connections
          type: fixed
          # FIXED 池专用参数
          max-connections: 500           # 建议值 = 后端实例数 × 每个实例的连接数
          acquire-timeout: 45000         # 等待获取连接的超时时间
```

## 2.2 连接池参数计算

```java
// 连接池参数计算示例
int backendInstances = 3;        // 后端服务实例数
int connectionsPerInstance = 100; // 每实例连接数
int maxConnections = backendInstances * connectionsPerInstance;
// maxConnections = 300

// 根据预估 QPS 和响应时间计算
double expectedQps = 10000;       // 预期 QPS
double avgResponseTime = 0.05;    // 平均响应时间（秒）
int requiredConnections = (int) Math.ceil(expectedQps * avgResponseTime);
// requiredConnections = 500
```

## 2.3 连接泄漏检测

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          # 启用连接泄漏检测
          metrics: true

# 开启 Netty 资源泄漏检测（JVM 参数）
# -Dio.netty.leakDetectionLevel=paranoid
# Level: disabled / simple / advanced / paranoid
```

# 3. HTTP 客户端调优

## 3.1 全局配置

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        # 超时配置
        connect-timeout: 5000
        response-timeout: 10s

        # 请求压缩
        compression: true

        # Header 大小限制
        max-header-size: 16384
        max-initial-line-length: 4096

        # SSL 配置
        ssl:
          handshake-timeout-millis: 10000
          use-insecure-trust-manager: false

        # 代理配置
        proxy:
          host: proxy.example.com
          port: 8080
          non-proxy-hosts:
            - localhost
            - 10.*
```

## 3.2 基于路由的配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: high_performance_route
          uri: http://fast-backend:8080
          predicates:
            - Path=/api/fast/**
          metadata:
            response-timeout: 1000       # 更短的超时

        - id: batch_route
          uri: http://batch-backend:8080
          predicates:
            - Path=/api/batch/**
          metadata:
            response-timeout: 30000      # 批量操作较长超时
            connect-timeout: 10000
```

# 4. 请求体与内存管理

## 4.1 请求体大小限制

```yaml
spring:
  codec:
    max-in-memory-size: 5MB           # 内存中最大缓冲区大小

spring:
  cloud:
    gateway:
      routes:
        - id: file_upload
          uri: http://upload-service
          predicates:
            - Path=/upload/**
          filters:
            - name: RequestSize
              args:
                maxSize: 10MB          # 限制上传大小
```

## 4.2 合理使用缓存

```java
// 避免不必要的请求体缓存
@Component
public class SelectiveCacheFilter implements GlobalFilter, Ordered {

    private static final Set<String> CACHE_REQUIRED_PATHS = Set.of(
        "/api/write", "/api/validate");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // 只在需要缓存请求体的路径上启用
        if (CACHE_REQUIRED_PATHS.stream().anyMatch(path::startsWith)) {
            return ServerWebExchangeUtils.cacheRequestBody(exchange,
                request -> chain.filter(
                    exchange.mutate().request(request).build()));
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return OrderedGatewayFilter.HIGHEST_PRECEDENCE;
    }
}
```

# 5. 路由性能优化

## 5.1 路由匹配效率

```yaml
spring:
  cloud:
    gateway:
      routes:
        # 将高频路由放在前面（order 小的优先匹配）
        - id: high_freq_route
          order: 0
          predicates:
            - Path=/api/high-frequency/**
          uri: http://fast-backend

        - id: low_freq_route
          order: 100
          predicates:
            - Path=/api/low-frequency/**
          uri: http://slow-backend
```

## 5.2 断言执行顺序

```yaml
# 将区分度高的断言放在前面，减少不必要的断言执行
predicates:
  - Method=GET                    # 先判断方法（开销小）
  - Query=version, v1              # 再判断参数
  - Header=X-Region, us-east       # 最后判断 Header
  - Path=/api/**                   # 路径放最后（PathMatcher 相对较重）
```

## 5.3 路由数量控制

过多的路由会导致匹配性能下降：

```java
@Component
public class RouteMetricsCollector {

    private final RouteLocator routeLocator;
    private final MeterRegistry meterRegistry;

    @EventListener(RefreshRoutesEvent.class)
    public void reportRouteCount() {
        routeLocator.getRoutes()
            .count()
            .subscribe(count -> {
                // 路由数量指标
                Gauge.builder("gateway.routes.count", () -> count)
                    .description("Total number of routes")
                    .register(meterRegistry);

                // 路由数量超过阈值时告警
                if (count > 1000) {
                    log.warn("Route count exceeds 1000: {}", count);
                }
            });
    }
}
```

# 6. 编译时优化

## 6.1 AOT 编译

Spring Cloud Gateway 4.x 支持 AOT（Ahead-of-Time）编译，减少启动时间和内存占用：

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <builder>paketobuildpacks/builder:tiny</builder>
            <env>
                <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
                <BP_NATIVE_IMAGE_BUILD_ARGUMENTS>
                    --initialize-at-build-time=org.springframework.cloud.gateway
                </BP_NATIVE_IMAGE_BUILD_ARGUMENTS>
            </env>
        </image>
    </configuration>
</plugin>
```

## 6.2 编译时注意事项

```yaml
# AOT 编译时需要显式配置负载均衡客户端
spring:
  cloud:
    loadbalancer:
      eager-load:
        clients: user-service,order-service
```

# 7. 缓存策略

## 7.1 LocalResponseCache 过滤器

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: cached_route
          uri: http://backend
          predicates:
            - Path=/api/static/**
          filters:
            - name: LocalResponseCache
              args:
                timeToLive: 1m          # 缓存存活时间
                size: 5MB               # 缓存容量
```

## 7.2 路由元数据缓存

```java
@Component
public class RouteMetadataCache {

    private final Cache<String, Object> metadataCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .recordStats()
        .build();

    public Object getMetadata(String routeId, String key) {
        String cacheKey = routeId + ":" + key;
        return metadataCache.get(cacheKey, k -> loadMetadata(routeId, key));
    }

    private Object loadMetadata(String routeId, String key) {
        // 从外部存储加载
        return null;
    }
}
```

# 8. 安全加固

## 8.1 最小化暴露

```yaml
# 关闭不必要的端点
management:
  endpoints:
    web:
      exposure:
        exclude: "*"
      include: health,info
```

## 8.2 请求验证

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - name: RequestSize
          args:
            maxSize: 5MB
        - RemoveRequestHeader=Cookie
        - SecureHeaders=
```

# 9. 灰度发布与蓝绿部署

## 9.1 权重路由

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: canary_v1
          uri: http://v1.example.com
          predicates:
            - Path=/api/**
            - Weight=production, 90
        - id: canary_v2
          uri: http://v2.example.com
          predicates:
            - Path=/api/**
            - Weight=production, 10
```

## 9.2 Header 路由

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: stable
          uri: lb://stable-service
          predicates:
            - Path=/api/**
            - Header=X-Version, stable

        - id: canary
          uri: lb://canary-service
          predicates:
            - Path=/api/**
            - Header=X-Version, canary
```

# 10. 性能基准测试

## 10.1 测试工具

```bash
# 使用 wrk 或 hey 进行压测
wrk -t10 -c100 -d30s --latency http://localhost:8080/api/test

# 使用 JMeter 进行场景测试
# 使用 Gatling 进行代码化压测
```

## 10.2 关键性能指标

| 指标 | 期望值 | 告警阈值 |
|------|--------|----------|
| QPS | 取决于硬件 | 低于期望值 80% |
| P50 延迟 | < 10ms | > 100ms |
| P99 延迟 | < 50ms | > 500ms |
| 错误率 | < 0.1% | > 1% |
| CPU 使用率 | < 70% | > 85% |
| 内存使用率 | < 80% | > 90% |
| 连接池使用率 | < 70% | > 90% |

## 10.3 常见瓶颈与解决方案

| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| QPS 上不去 | CPU 瓶颈 | 增加实例数、优化过滤器逻辑 |
| 延迟高 | 后端慢 | 配置超时、熔断、优化后端 |
| 内存溢出 | 请求体缓存 | 限制请求体大小、减少缓存 |
| 连接超时 | 连接池耗尽 | 增大连接池、优化后端响应速度 |
| 线程阻塞 | 阻塞操作 | 替换为异步操作、使用 `subscribeOn` |

# 11. 生产部署清单

## 11.1 JVM 参数

```bash
# 生产环境 JVM 参数
java -Xms2g -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -XX:+ParallelRefProcEnabled \
     -XX:+UseStringDeduplication \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/gateway/heapdump.hprof \
     -Dio.netty.leakDetectionLevel=simple \
     -jar gateway.jar
```

## 11.2 配置清单

```yaml
# 生产环境推荐配置
server:
  port: 8080
  compression:
    enabled: true
    mime-types: text/html,text/xml,text/plain,application/json
    min-response-size: 1024

spring:
  cloud:
    gateway:
      httpclient:
        pool:
          type: fixed
          max-connections: 500
          acquire-timeout: 45000
          max-idle-time: 3000
        connect-timeout: 5000
        response-timeout: 10s

      metrics:
        enabled: true

  codec:
    max-in-memory-size: 2MB

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

## 11.3 容器化部署

```dockerfile
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app
COPY target/gateway.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
spec:
  replicas: 3
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
```
