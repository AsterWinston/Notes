
# 1. 概述

Sentinel 通过 `SentinelFeign` 和 `@SentinelRestTemplate` 为 Feign 声明式 HTTP 调用提供流量防护。

```
Feign Client (声明式接口)
  │
  ├── SentinelFeign.Builder
  │     └── SentinelInvocationHandler
  │           ├── 请求前：SphU.entry() → SlotChain
  │           ├── 请求中：Feign 调用
  │           └── 请求后：统计 + 降级处理
  │
  └── 被拦截 → BlockException → fallback / fallbackFactory
```

# 2. 依赖引入

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**注意**：Sentinel Feign 集成由 `SentinelFeignAutoConfiguration` 自动配置，无需额外依赖。

# 3. 基本使用

## 3.1 fallback 方式

```java
// 1. 定义 Feign 接口
@FeignClient(
    name = "user-service",
    fallback = UserClientFallback.class   // 降级处理类
)
public interface UserClient {

    @GetMapping("/api/user/{id}")
    User getUser(@PathVariable Long id);

    @PostMapping("/api/user")
    User createUser(@RequestBody User user);
}

// 2. fallback 实现
@Component
public class UserClientFallback implements UserClient {

    @Override
    public User getUser(Long id) {
        // 返回兜底数据
        return new User(id, "降级用户", 0);
    }

    @Override
    public User createUser(User user) {
        // 降级处理：记录到数据库待重试
        log.warn("创建用户被降级: {}", user);
        return null;
    }
}
```

## 3.2 fallbackFactory 方式

```java
@FeignClient(
    name = "user-service",
    fallbackFactory = UserClientFallbackFactory.class  // 可获取异常
)
public interface UserClient { ... }

@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        // 记录异常信息
        log.error("Feign 调用失败: service=user-service, cause={}", cause.getMessage());

        if (cause instanceof FlowException) {
            // 流控拒绝
            return id -> new User(id, "系统繁忙", 0);
        }
        if (cause instanceof DegradeException) {
            // 熔断拒绝
            return id -> new User(id, "服务熔断中", 0);
        }
        // 其他异常
        return id -> { throw new RuntimeException(cause); };
    }
}
```

# 4. 资源名规则

Sentinel Feign 自动生成的资源名格式：

```
HTTP 方法:协议://服务名/路径

示例:
GET:http://user-service/api/user/{id}
POST:http://order-service/api/order
```

资源名包含 HTTP 方法前缀，便于在 Dashboard 中区分同路径的读写操作。

# 5. 配置项

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      eager: true
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow
        degrade:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            rule-type: degrade

# Feign 配置
feign:
  sentinel:
    enabled: true          # 启用 Sentinel 集成（默认 true）
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 10000
```

# 6. @SentinelRestTemplate

对于非 Feign 的 RestTemplate 调用：

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @SentinelRestTemplate(
        blockHandler = "handleBlock",
        blockHandlerClass = RestBlockHandler.class,
        fallback = "handleFallback",
        fallbackClass = RestFallbackHandler.class
    )
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .readTimeout(Duration.ofSeconds(10))
            .build();
    }
}

// blockHandler 示例
public class RestBlockHandler {
    public static SentinelClientHttpResponse handleBlock(
            HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution, BlockException ex) {
        return new SentinelClientHttpResponse(
            "{\"code\":429,\"message\":\"流控限制\"}");
    }
}
```

# 7. 错误解码与熔断联动

Feign 的 `ErrorDecoder` 与 Sentinel 熔断可以协同工作：

```java
@Component
public class SentinelAwareErrorDecoder implements ErrorDecoder {

    private final ErrorDecoder defaultDecoder = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        // 503 → 标记为异常（Sentinel 统计）
        // 404 → 不计入异常（业务正常情况）
        if (response.status() == 503) {
            // 返回的异常会被 Sentinel Feign Filter 统计
            return new RetryableException(
                response.status(),
                "Service Unavailable",
                response.request().httpMethod(),
                null,
                response.request()
            );
        }
        return defaultDecoder.decode(methodKey, response);
    }
}
```

# 8. 熔断降级规则示例

```json
// Nacos 配置：${app}-degrade-rules
[
  {
    "resource": "GET:http://user-service/api/user/{id}",
    "grade": 0,
    "count": 200,
    "timeWindow": 10,
    "minRequestAmount": 5,
    "slowRatioThreshold": 0.5
  },
  {
    "resource": "POST:http://order-service/api/order",
    "grade": 1,
    "count": 0.3,
    "timeWindow": 15,
    "minRequestAmount": 10
  }
]
```

# 9. 最佳实践

1. **fallback vs fallbackFactory**：需要感知异常类型时使用 `fallbackFactory`
2. **超时设置**：Feign `readTimeout` < Sentinel 熔断 `timeWindow`，确保超时先于熔断
3. **resource 命名**：Sentinel 自动生成的资源名足够清晰，一般不需要自定义
4. **降级返回值**：始终返回有意义的兜底对象，不要返回 `null`
5. **熔断优先于流控**：对 Feign 调用优先配置熔断规则，流控作为补充
