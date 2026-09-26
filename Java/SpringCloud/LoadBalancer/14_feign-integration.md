
# 1. 概述

Spring Cloud OpenFeign 默认集成了 Spring Cloud LoadBalancer，`@FeignClient` 注解中的服务名会自动通过 LoadBalancer 解析为实际的服务实例地址。

```java
@FeignClient(name = "user-service")   // name 即 serviceId
public interface UserClient {

    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);
}
```

# 2. 自动装配原理

## 2.1 依赖关系

```
spring-cloud-starter-openfeign
    │
    ├── spring-cloud-openfeign-core
    └── spring-cloud-starter-loadbalancer  ← 自动引入
          │
          ├── spring-cloud-loadbalancer
          └── spring-cloud-starter (caffeine cache)
```

## 2.2 自动配置

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({Feign.class, LoadBalancerClient.class})
@ConditionalOnBean(LoadBalancerClient.class)
@AutoConfigureBefore(FeignAutoConfiguration.class)
public class FeignLoadBalancerAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public Client feignClient(LoadBalancerClient loadBalancerClient) {
        // 将 LoadBalancerClient 包装为 Feign 的 Client 实现
        return new FeignBlockingLoadBalancerClient(
                new Client.Default(null, null),    // 默认 HTTP 客户端
                loadBalancerClient);
    }
}
```

# 3. FeignBlockingLoadBalancerClient 源码

```java
public class FeignBlockingLoadBalancerClient implements Client {

    private final Client delegate;               // 底层 HTTP 客户端
    private final LoadBalancerClient loadBalancerClient;
    private final LoadBalancerProperties properties;

    @Override
    public Response execute(Request request, Request.Options options)
            throws IOException {

        URI uri = URI.create(request.url());
        String serviceId = uri.getHost();

        // ① 非服务名 URL（直连）→ 直接透传
        if (serviceId == null || isNotServiceName(serviceId)) {
            return delegate.execute(request, options);
        }

        // ② 带 Hints 的负载均衡选择
        Set<LoadBalancerLifecycle> lifecycles = getLifecycles(serviceId);
        RequestData requestData = new RequestData(request);

        // ③ 调用 LoadBalancerClient.choose() 选择实例
        ServiceInstance instance = loadBalancerClient.choose(
                serviceId, new DefaultRequest<>(requestData));

        if (instance == null) {
            throw new IllegalStateException(
                    "No instances available for " + serviceId);
        }

        // ④ 重建 URL：将 serviceId 替换为实际 host:port
        String newUrl = rebuildUrl(request.url(), instance);

        // ⑤ 重建请求对象
        Request newRequest = Request.create(
                request.httpMethod(),
                newUrl,
                request.headers(),
                request.body(),
                request.charset(),
                request.requestTemplate());

        // ⑥ 委托给真正的 HTTP 客户端执行
        return delegate.execute(newRequest, options);
    }

    private String rebuildUrl(String originalUrl, ServiceInstance instance) {
        // http://user-service/api/users/1
        // → http://192.168.1.10:8080/api/users/1
        URI original = URI.create(originalUrl);
        return LoadBalancerUriTools.reconstructURI(instance, original).toString();
    }
}
```

# 4. 底层 HTTP 客户端选择

`FeignBlockingLoadBalancerClient` 的 `delegate` 决定了实际的 HTTP 引擎：

```yaml
# Apache HttpClient 5（推荐）
feign:
  httpclient:
    hc5:
      enabled: true
    connection-timeout: 3000
    connection-timer-repeat: 3000

# 或 OkHttp
feign:
  okhttp:
    enabled: true

# 或 Java 11 HttpClient
feign:
  httpclient:
    jdk11:
      enabled: true
```

对应的 Client 实现：

```java
// Apache HttpClient 5
FeignBlockingLoadBalancerClient(
    new ApacheHttp5Client(httpClientBuilder.build()),
    loadBalancerClient
)

// OkHttp
FeignBlockingLoadBalancerClient(
    new OkHttpClient(okHttpClient),
    loadBalancerClient
)
```

# 5. @FeignClient 的 url 属性

```java
// 直连模式：绕过 LoadBalancer
@FeignClient(name = "user-service", url = "http://192.168.1.100:8080")
public interface UserClient { }

// 负载均衡模式（默认）
@FeignClient(name = "user-service")        // 通过 LoadBalancer 解析
public interface UserClient { }

// 混合模式：指定 url 时优先级最高
@FeignClient(name = "user-service", url = "${user-service.url:}")
public interface UserClient { }
// 如果配置了 user-service.url，用直连；否则走 LB
```

# 6. 自定义 FeignClient 配置

```java
@FeignClient(
    name = "user-service",
    configuration = UserServiceFeignConfig.class,
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient { }
```

```java
@Configuration
public class UserServiceFeignConfig {

    @Bean
    public Retryer feignRetryer() {
        // 关闭 Feign 自带重试，交给 LoadBalancer
        return Retryer.NEVER_RETRY;
    }

    @Bean
    public RequestInterceptor traceInterceptor() {
        return template -> {
            template.header("X-Trace-Id",
                    MDC.get("traceId"));
        };
    }

    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.HEADERS;
    }
}
```

# 7. Feign + LoadBalancer 完整请求链路

```
OrderService.orderClient.getUser(1L)
    │
    ▼
Feign 动态代理 → MethodHandler.invoke()
    │
    ▼
SynchronousMethodHandler.executeAndDecode()
    │
    ▼
FeignBlockingLoadBalancerClient.execute(request)
    │
    ├── 解析 URL: http://user-service/api/users/1
    ├── serviceId = "user-service"
    │
    ├── loadBalancerClient.choose("user-service")
    │     │
    │     ├── CachingServiceInstanceListSupplier.get()
    │     ├── ZonePreferenceServiceInstanceListSupplier.get()
    │     ├── ReactorLoadBalancer.choose().block()
    │     └── → ServiceInstance{192.168.1.11:8080}
    │
    ├── 重建 URL: http://192.168.1.11:8080/api/users/1
    │
    └── delegate.execute(newRequest)
          │
          └── ApacheHttp5Client / OkHttpClient 发送 HTTP 请求
```

# 8. LoadBalancer + Sentinel 组合

```java
@FeignClient(
    name = "user-service",
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    @SentinelResource(value = "getUser", fallback = "getUserFallback")
    User getUser(@PathVariable("id") Long id);
}
```

```yaml
feign:
  sentinel:
    enabled: true          # 启用 Feign Sentinel 集成
  circuitbreaker:
    enabled: true
```

**熔断 + 负载均衡的协同**：

```
请求 user-service
  │
  ▼ LoadBalancer 选择实例
  ├── 实例 A 被 Sentinel 熔断 → 降级返回 fallback
  ├── 实例 B 正常 → 返回结果
  └── 实例 C 正常 → 返回结果
```

# 9. 排除 LoadBalancer 依赖

如果不想使用负载均衡：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

或者禁用：

```yaml
spring:
  cloud:
    loadbalancer:
      enabled: false
```

# 10. 最佳实践

```java
@FeignClient(
    name = "user-service",
    configuration = UserServiceFeignConfig.class,
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);
}

// 配置类（不要被 @ComponentScan 扫描到）
@Configuration(proxyBeanMethods = false)
class UserServiceFeignConfig {

    @Bean
    public Retryer feignRetryer() {
        return Retryer.NEVER_RETRY; // 重试交给 LoadBalancer 层
    }

    @Bean
    public RequestInterceptor requestInterceptor() {
        return template -> {
            template.header("X-Service-Name", "order-service");
            template.header("X-Request-Id",
                    UUID.randomUUID().toString());
        };
    }

    @Bean
    public Decoder feignDecoder() {
        return new JacksonDecoder();
    }

    @Bean
    public ErrorDecoder feignErrorDecoder() {
        return new CustomErrorDecoder();
    }
}

// 降级工厂
@Component
class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        return id -> {
            log.error("User service unavailable for id: {}", id, cause);
            return User.FALLBACK;
        };
    }
}
```
