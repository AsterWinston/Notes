
# 1. 接口设计

## 1.1 统一响应体包装

不要直接返回异构的服务端原始 JSON，通过自定义 Decoder 剥离统一响应体：

```java
// 服务端统一返回
{
  "code": 200,
  "message": "success",
  "data": { "id": 1, "name": "John" }
}

// ❌ 错误：Feign 接口暴露了包装结构
@GetMapping("/users/{id}")
ApiResponse<User> getUser(@PathVariable Long id);

// ✓ 正确：Decoder 剥离包装，Feign 接口只关心业务对象
@GetMapping("/users/{id}")
User getUser(@PathVariable Long id);
```

## 1.2 避免循环依赖

```java
// ❌ 危险：服务 A 调 B，B 调 A
// order-service → user-service → order-service

// ✓ 方案 1：消息队列解耦
// ✓ 方案 2：提取公共服务
// ✓ 方案 3：考虑非 Feign 的异步 HTTP 调用
```

## 1.3 接口粒度适中

```java
// ❌ 过于细粒度（N+1 问题）
User user = userClient.getUser(userId);
Order order = orderClient.getOrder(orderId);
Product product = productClient.getProduct(order.getProductId());

// ✓ 批量查询
UserDto dto = userClient.getUserWithOrders(userId);

// ✓ 或使用 CompletableFuture 并行调用
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> userClient.getUser(userId));
CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));
CompletableFuture.allOf(userFuture, orderFuture).join();
```

# 2. 配置管理

## 2.1 配置类隔离

```java
// ❌ 错误：配置类被全局扫描到
@Configuration
public class UserFeignConfig {
    @Bean
    public RequestInterceptor authInterceptor() { ... }
}

// ✓ 正确：不标记 @Configuration，单独指定
public class UserFeignConfig {
    @Bean
    public RequestInterceptor authInterceptor() { ... }
}

// 在 @FeignClient 中引用
@FeignClient(name = "user-service", configuration = UserFeignConfig.class)
```

## 2.2 超时配置原则

```
connectTimeout < circuitBreaker.timeoutDuration < readTimeout
  (3s)        < (5s)                            < (10s)
```

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 3000    # 连接超时：3s
            read-timeout: 10000      # 读取超时：10s

resilience4j:
  timelimiter:
    configs:
      default:
        timeout-duration: 5s         # 熔断器超时：5s（介于两者之间）
```

## 2.3 不同环境不同配置

```yaml
# application-dev.yml
spring.cloud.openfeign.client.config.default.logger-level: FULL

# application-prod.yml
spring.cloud.openfeign.client.config.default.logger-level: BASIC
```

# 3. 异常处理

## 3.1 捕获 FeignException

```java
// ✓ 全局异常处理器
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(FeignException.class)
    public ResponseEntity<?> handleFeignException(FeignException e) {
        log.error("Feign call failed: {}", e.getMessage());
        if (e.status() == -1) {
            // 连接失败（网络不通、超时等）
            return ResponseEntity.status(503)
                    .body(Map.of("message", "Service unavailable"));
        }
        return ResponseEntity.status(e.status())
                .body(Map.of("message", "Upstream error"));
    }
}
```

## 3.2 ErrorDecoder 中不要吞掉异常

```java
// ❌ 错误：在 ErrorDecoder 中捕获了所有异常返回 null
public class BadErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            // ...
        } catch (Exception e) {
            return null;  // ❌ 这会导致调用方拿到 null，误以为成功
        }
    }
}

// ✓ 正确：解析失败回退到默认处理
public class GoodErrorDecoder implements ErrorDecoder {
    private final ErrorDecoder defaultDecoder = new ErrorDecoder.Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            return parseBusinessError(response);
        } catch (Exception e) {
            return defaultDecoder.decode(methodKey, response);
        }
    }
}
```

# 4. Feign 客户端线程安全

## 4.1 Feign 客户端是线程安全的

```java
// ✓ 可以安全地在多个线程中共享同一个 Feign 客户端
@Service
public class OrderService {
    private final UserClient userClient;  // 单例，线程安全

    // 多线程环境下安全使用
    public void processOrders(List<Order> orders) {
        orders.parallelStream().forEach(order -> {
            User user = userClient.getUser(order.getUserId());  // 安全
        });
    }
}
```

## 4.2 Interceptor 的线程安全

```java
// ❌ 危险：拦截器持有可变状态
@Component
public class UnsafeInterceptor implements RequestInterceptor {
    private String currentToken;  // 线程不安全！

    @Override
    public void apply(RequestTemplate template) {
        template.header("Authorization", currentToken);
    }
}

// ✓ 正确：通过 ThreadLocal 或请求上下文传递
@Component
public class SafeInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String token = AuthContextHolder.getToken();  // ThreadLocal
        if (token != null) {
            template.header("Authorization", "Bearer " + token);
        }
    }
}
```

# 5. 性能陷阱

## 5.1 避免在循环中重复调用

```java
// ❌ 错误：N 次远程调用
List<User> users = new ArrayList<>();
for (Long id : userIds) {
    users.add(userClient.getUser(id));  // N 次 HTTP 调用
}

// ✓ 正确：批量接口
@GetMapping("/users")
List<User> getUsersByIds(@RequestParam("ids") List<Long> ids);

// ✓ 或并发调用
List<CompletableFuture<User>> futures = userIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> userClient.getUser(id)))
    .collect(Collectors.toList());
List<User> users = futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

## 5.2 不要用 Feign 传输大文件

```java
// ❌ 不适合：
@GetMapping("/files/{id}")
byte[] downloadLargeFile(@PathVariable String id);  // 100MB 全部加载到内存

// ✓ 使用 feign.Response：
@GetMapping("/files/{id}")
feign.Response download(@PathVariable String id);   // 流式读取

// ✓ 或直接用 OSS/CDN 直传
```

# 6. 日志与可观测性

## 6.1 日志敏感信息脱敏

```java
public class SensitiveDataMaskedLogger extends feign.Logger {
    @Override
    protected void log(String configKey, String format, Object... args) {
        String message = String.format(format, args);
        // 脱敏
        message = message.replaceAll("(password=\")[^\"]+", "$1****");
        message = message.replaceAll("(token=)[^& ]+", "$1****");
        log.debug(methodTag(configKey) + message);
    }
}
```

## 6.2 生产环境日志级别

```yaml
# 生产环境推荐
spring.cloud.openfeign.client.config.default.logger-level: BASIC
logging.level.com.example.feign: DEBUG  # 仅记录请求方法和状态码
```

# 7. 版本兼容性

## 7.1 接口向下兼容

```java
// ✓ 新增可选字段 → 不影响老客户端
@PostMapping("/users")
User create(@RequestBody User user);
// User 新增字段：private String nickName;  // 老客户端不传即为 null

// ✓ 新增可选参数
@GetMapping("/users")
List<User> search(@RequestParam(required = false, defaultValue = "1") Integer page);

// ❌ 删除必填字段或修改返回值结构 → 破坏兼容性
```

## 7.2 Feign 版本与 Spring Cloud 版本对应

| Spring Cloud | Spring Boot | OpenFeign |
|-------------|-------------|-----------|
| 2023.0.x | 3.2.x | 13.x |
| 2022.0.x | 3.0.x ~ 3.1.x | 12.x |
| 2021.0.x | 2.6.x ~ 2.7.x | 11.x |
| 2020.0.x | 2.4.x ~ 2.5.x | 11.x |
| Hoxton | 2.2.x ~ 2.3.x | 10.x |

# 8. 常见陷阱速查表

| 陷阱 | 现象 | 解决方案 |
|------|------|----------|
| `@PathVariable` 没写 value | `IllegalStateException` | 显式指定 `@PathVariable("id")` |
| `@RequestParam` 没写 value | 编译/运行时报错 | 显式指定 `@RequestParam("name")` |
| 配置类被全局扫描 | 全局配置污染 | 去掉 `@Configuration`，通过 `configuration=` 指定 |
| `readTimeout` < 熔断器超时 | 熔断器不生效 | 熔断器超时应 < `readTimeout` |
| GET 请求带 `@RequestBody` | 实际发 POST 请求 | GET 改用 `@RequestParam` 或 `@SpringQueryMap` |
| 循环调用 | StackOverflow / 超时 | 消息队列解耦、提取公共服务 |
| 大结果集不设超时 | 线程堆积 | 设置合理 `read-timeout`，分页查询 |
| `HttpURLConnection` 默认无连接池 | 生产环境性能差 | 切换到 OkHttp 或 Apache HttpClient |
| Fallback 中调远程服务 | 雪崩 | Fallback 逻辑必须本地闭环 |
| `FeignException.status() = -1` | 连接失败 | 单独处理，开启重试 + 熔断 |
