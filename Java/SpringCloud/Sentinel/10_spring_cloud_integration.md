
# 1. 概述

Sentinel 作为 Spring Cloud Alibaba 的核心组件，通过 `spring-cloud-starter-alibaba-sentinel` 提供**零配置集成**。主要能力包括：

- **自动配置**：自动启用 Sentinel 拦截器
- **注解支持**：`@SentinelResource` 透明拦截
- **RestTemplate 支持**：`@SentinelRestTemplate` 为 HTTP 调用添加防护
- **动态数据源**：与 Nacos/Apollo 等配置中心集成
- **Actuator 端点**：暴露 Sentinel 运行指标

# 2. 依赖与自动配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

**自动配置类**：

| 配置类 | 职责 |
|--------|------|
| `SentinelWebAutoConfiguration` | 注册 SentinelWebInterceptor，拦截所有 Spring MVC 请求 |
| `SentinelWebFluxAutoConfiguration` | WebFlux 环境的 Sentinel 集成 |
| `SentinelFeignAutoConfiguration` | Feign 集成 |
| `SentinelAutoConfiguration` | 核心 Bean 初始化 |
| `SentinelEndpointAutoConfiguration` | Actuator 端点暴露 |

# 3. 拦截机制

## 3.1 Spring MVC 自动拦截

`SentinelWebInterceptor` 自动将所有 Spring MVC 请求作为 Sentinel 资源保护：

```
HTTP 请求 → DispatcherServlet
  → SentinelWebInterceptor.preHandle()
    → SphU.entry("GET:/api/user/{id}")
      → SlotChain 执行
        → 通过 → Controller 执行
        → 拒绝 → SentinelBlockException → 返回 429
```

**资源名生成规则**：`{HTTP_METHOD}:{URL_PATTERN}`

```java
// 自动生成资源名示例
GET:/api/user/{id}
POST:/api/order
DELETE:/api/product/{id}
```

## 3.2 自定义资源名

```java
@Component
public class MyUrlCleaner implements UrlCleaner {
    @Override
    public String clean(String originUrl) {
        // 统一处理 Restful 风格的 URL
        // /api/user/1 → /api/user/{id}
        // /api/user/2 → /api/user/{id}
        return originUrl.replaceAll("/\\d+", "/{id}")
                        .replaceAll("/[a-f0-9]{32}", "/{uuid}");
    }
}
```

# 4. @SentinelResource 详解

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/user/{id}")
    @SentinelResource(
        value = "getUser",                     // 资源名
        entryType = EntryType.OUT,             // 入口类型（IN=进入/OUT=出口）
        blockHandler = "handleBlock",          // 流控/降级处理方法
        blockHandlerClass = BlockHandler.class,// 指定 blockHandler 所在类
        fallback = "handleFallback",           // 业务异常处理方法
        fallbackClass = FallbackHandler.class, // 指定 fallback 所在类
        defaultFallback = "handleDefault"      // 默认 fallback（单个 Throwable 参数）
    )
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
}
```

## 4.1 blockHandler vs fallback

| 维度 | blockHandler | fallback |
|------|-------------|----------|
| **触发条件** | BlockException（流控/熔断/系统保护） | 业务异常（Throwable） |
| **参数签名** | 原方法参数 + BlockException | 原方法参数 + Throwable |
| **目的** | 处理流量管控拒绝 | 处理业务异常兜底 |
| **是否必须** | 否（不指定则抛 BlockException） | 否（不指定则抛原异常） |

## 4.2 分离 blockHandler 类

```java
public class UserBlockHandler {

    // 方法必须为 static
    public static User handleGetUserBlock(Long id, BlockException ex) {
        log.warn("User query blocked: id={}", id);
        return new User(id, "Blocked", 0);
    }

    public static User handleCreateUserBlock(User user, BlockException ex) {
        log.warn("User creation blocked");
        return null;
    }
}

@SentinelResource(value = "getUser",
    blockHandler = "handleGetUserBlock",
    blockHandlerClass = UserBlockHandler.class)
public User getUser(Long id) { ... }
```

# 5. @SentinelRestTemplate

为 RestTemplate 添加 Sentinel 保护：

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
        return new RestTemplate();
    }
}

public class RestBlockHandler {
    public static SentinelClientHttpResponse handleBlock(
            HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution, BlockException ex) {
        return new SentinelClientHttpResponse("Blocked by Sentinel: " + ex.getMessage());
    }
}
```

# 6. Actuator 端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: sentinel
  endpoint:
    sentinel:
      enabled: true
```

```bash
# 查看 Sentinel 状态
curl http://localhost:8080/actuator/sentinel
```

# 7. 全局异常处理

```java
@RestControllerAdvice
public class SentinelExceptionHandler {

    @ExceptionHandler(BlockException.class)
    @ResponseStatus(HttpStatus.TOO_MANY_REQUESTS)  // 429
    public Result handleBlockException(BlockException ex) {
        return Result.error(429, "请求过于频繁，请稍后再试");
    }

    @ExceptionHandler(DegradeException.class)
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)  // 503
    public Result handleDegradeException(DegradeException ex) {
        return Result.error(503, "服务暂时不可用");
    }
}
```

# 8. 最佳实践

1. **资源命名规范**：`{服务名}:{接口}:{方法}`，如 `userService:/api/user:getUser`
2. **blockHandler 与 fallback 分离**：使用独立类管理，保持 Controller 简洁
3. **URL 统一标准化**：通过 `UrlCleaner` 避免 Restful 参数导致资源爆炸
4. **fallback 始终有值**：降级返回应当是有意义的兜底数据，不是 null
5. **配合 Actuator**：开启 Sentinel 端点，集成 Prometheus 监控
