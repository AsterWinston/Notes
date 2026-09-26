
# 1. 概述

Fallback 是 Feign 客户端调用失败后的**降级兜底机制**。Spring Cloud OpenFeign 支持两种方式：

| 方式 | 说明 | 能否获取异常 |
|------|------|-------------|
| `fallback` | 指定降级类，必须实现 `@FeignClient` 接口 | 否 |
| `fallbackFactory` | 指定降级工厂，内部可获取 `Throwable` | 是 |

# 2. Fallback 基本用法

```java
@FeignClient(name = "user-service", fallback = UserClientFallback.class)
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable Long id);

    @PostMapping("/api/users")
    User createUser(@RequestBody User user);
}
```

```java
@Component
public class UserClientFallback implements UserClient {

    @Override
    public User getUser(Long id) {
        // 返回兜底数据
        return new User(id, "N/A", 0);
    }

    @Override
    public User createUser(User user) {
        // 写操作降级：抛业务异常或返回 null
        throw new ServiceDegradedException("用户服务不可用，创建用户失败");
    }
}
```

# 3. FallbackFactory 基本用法

```java
@FeignClient(name = "user-service",
             fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable Long id);
}
```

```java
@Component
@Slf4j
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        // 记录异常详情
        log.error("UserClient 调用异常", cause);

        return new UserClient() {
            @Override
            public User getUser(Long id) {
                // 根据异常类型做不同处理
                if (cause instanceof FeignException.NotFound) {
                    return null;
                }
                if (cause instanceof RetryableException) {
                    return getFromCache(id);  // 临时故障，用缓存
                }
                return new User(id, "N/A", 0);
            }

            private User getFromCache(Long id) {
                // 从本地缓存获取
                return cacheManager.get("user:" + id, User.class);
            }
        };
    }
}
```

# 4. Fallback 执行流程

```
Feign 调用
    │
    ├── 正常 → Decoder.decode() → 返回业务对象
    │
    └── 异常
        │
        ├── CircuitBreaker OPEN → 直接走 fallback
        │
        └── 其他异常 (IOException / FeignException / TimeoutException)
              │
              ├── ErrorDecoder.decode() → 抛出异常
              │
              ├── CircuitBreaker 记录失败
              │
              └── FeignCircuitBreakerInvocationHandler 捕获
                    │
                    └── fallback / fallbackFactory.create(cause).method(args)
```

# 5. 源码：FeignCircuitBreakerInvocationHandler

```java
class FeignCircuitBreakerInvocationHandler implements InvocationHandler {

    private final CircuitBreakerFactory factory;
    private final FeignClientFactoryBean feign;
    private final Target<?> target;
    private final FallbackFactory<?> fallbackFactory;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. 创建 CircuitBreaker
        CircuitBreaker circuitBreaker = factory.create(target.name());

        // 2. 创建 fallback
        Supplier<Object> fallback = () -> {
            Object fallbackInstance = fallbackFactory.create(...);
            return method.invoke(fallbackInstance, args);
        };

        // 3. 用 circuitBreaker 包装 Feign 调用
        return circuitBreaker.run(
                () -> {
                    // 实际 Feign 调用
                    return feignInstanceMethod.invoke(feignTarget, args);
                },
                throwable -> fallback.get()
        );
    }
}
```

# 6. fallback vs fallbackFactory 选择

| 场景 | 推荐 |
|------|------|
| 降级逻辑简单，不关心失败原因 | `fallback` |
| 需要根据异常类型做不同处理 | `fallbackFactory` |
| 需要记录异常日志 | `fallbackFactory` |
| 404 返回 null，其他错误抛异常 | `fallbackFactory` |
| 根据错误码返回不同的兜底值 | `fallbackFactory` |

# 7. 异常类型分类处理

```java
@Component
public class SmartFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {

        // 拆解异常链
        Throwable root = findRootCause(cause);

        if (root instanceof SocketTimeoutException) {
            return readTimeoutFallback();
        }
        if (root instanceof ConnectException) {
            return connectionRefusedFallback();
        }
        if (root instanceof FeignException.NotFound) {
            return notFoundFallback();
        }
        if (root instanceof FeignException.InternalServerError) {
            return serverErrorFallback();
        }

        return defaultFallback(root);
    }

    private UserClient readTimeoutFallback() {
        return id -> getFromDistributedCache(id);
    }

    private UserClient connectionRefusedFallback() {
        return id -> { throw new ServiceUnavailableException("用户服务不可达"); };
    }

    private Throwable findRootCause(Throwable t) {
        Throwable root = t;
        while (root.getCause() != null && root.getCause() != root) {
            root = root.getCause();
        }
        return root;
    }
}
```

# 8. Fallback 与 @SentinelResource 对比

Spring Cloud Alibaba Sentinel 也提供降级支持，可以与 Feign 结合：

```java
@FeignClient(name = "user-service",
             fallback = UserClientFallback.class)
public interface UserClient {
    // Feign 层面 fallback
}
```

```java
// 也可以使用 Sentinel 的 fallback
@SentinelResource(value = "getUser",
                  fallback = "getUserFallback",
                  fallbackClass = UserSentinelFallback.class)
public User getUser(Long id) {
    return userClient.getUser(id);
}
```

**建议**：优先使用 Spring Cloud Circuit Breaker + Resilience4j 的 fallback，与 Feign 集成更紧密。

# 9. 降级策略设计

## 9.1 策略 1：返回兜底值

```java
return new User(id, "N/A", 0);  // 适合展示场景
```

## 9.2 策略 2：返回缓存数据

```java
return redisTemplate.opsForValue().get("user:" + id);
```

## 9.3 策略 3：抛出业务异常

```java
throw new ServiceDegradedException("用户服务暂不可用，请稍后重试");
```

## 9.4 策略 4：静默失败

```java
return null;  // 上游处理 null，前端展示「加载失败」
```

# 10. 最佳实践

1. **降级逻辑必须在 Fallback 类内部闭环** — 不要再调用其他远程服务
2. **写操作降级要谨慎** — POST/PUT/DELETE 失败时，返回 null 可能会误导调用方
3. **FallbackFactory 中不要做耗时操作** — 如查数据库、调 Redis 等可能影响熔断器恢复
4. **记录降级事件** — 在 FallbackFactory 中记录日志或发监控指标
5. **熔断器开启时所有请求都走 Fallback** — 要确保 Fallback 能应对全量流量
