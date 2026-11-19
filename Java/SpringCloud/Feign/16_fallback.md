# 1. 概述

Fallback 是 Feign 客户端调用失败后的**降级兜底机制**。Spring Cloud OpenFeign 支持两种方式：

| 方式 | 说明 | 能否获取异常 |
|------|------|-------------|
| `fallback` | 指定降级类，必须实现 `@FeignClient` 接口 | 否 |
| `fallbackFactory` | 指定降级工厂，内部可获取 `Throwable` | 是 |

## 1.1 先划清本篇与熔断篇的边界

`14_circuit-breaker.md` 讲的是"什么时候该停止尝试"（状态机与统计口径），本篇讲的是"停止之后返回什么"（降级值的设计与执行机制）。一句话关系：**fallback 是熔断器的出口函数**——没有开 `spring.cloud.openfeign.circuitbreaker.enabled`，`@FeignClient` 的 `fallback` 属性被完全忽略（`14_circuit-breaker.md` §2.1 的开关链），异常直接上抛。这个前提制造了本系列最高频的"配了没生效"问题，也是 §4 流程图里"异常必经 CircuitBreaker"的原因。

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

## 2.1 两个结构性细节

- **implements 接口是编译期红利**：提供方接口以后加方法，fallback 类立刻编译失败，逼降级逻辑同步演进——这是它优于反射式兜底（Mock、动态代理）的根因；
- **读与写分治**：示例里 GET 返回兜底值、POST 直接抛 `ServiceDegradedException`，这是降级设计的底线形态——**查询失败可以用假数据救展示，写失败绝不能静默返回"看起来成功"的对象**，否则用户以为下单成功而实际什么都没发生。

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

## 3.1 create(cause) 的执行时机与成本纪律

`FallbackFactory` 是 `org.springframework.cloud.client.circuitbreaker` 包的接口（与 Hystrix/R4j 共用的抽象），`create(Throwable)` **在每次失败发生时**被调用一次——不是启动期工厂。两条纪律由此而来：**工厂方法必须廉价**（拿 Redis、拼大对象都禁止，重活放进返回对象的实例方法里按需做）；**日志打在工厂里等于每个失败请求一条 ERROR**，故障风暴时日志本身就是二次灾害，按异常类型分派后只对该告警的类别记日志（§7），或做限频。另外注意匿名类**放弃了 implements 的编译期红利**——提供方接口加方法后匿名类不会报错，直到那个方法第一次触发降级才 `AbstractMethodError`？不会——接口加方法后匿名类没实现直接编译失败（Java 匿名类同样要求实现全部抽象方法），真正失去的是"IDE 自动补全 + 一处集中维护"的可读性；长期建议用**具名的完整实现类**接住 cause，兼得两者。

# 4. Fallback 执行流程

```text
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

## 4.1 图上三个不显然的推论

1. **cause 是翻译后的异常**：`ErrorDecoder` 在熔断器**内侧**执行，fallback 拿到的是它产出的 `FeignException`/自定义异常——所以 `8_error-handling.md` §9.6 说"翻译集中在 ErrorDecoder"之后，fallback 的分派逻辑才能写得干净（拿到 `BusinessException` 而不是裸状态码）。
2. **OPEN 态不产生新异常**：熔断打开时直接走 fallback，`cause` 是打开那一刻的**最后触发异常**还是 `CallNotPermittedException`（Resilience4j），取决于版本与装饰方式——分派逻辑不要把 `instanceof` 当穷举，保留 default 分支。
3. **重试对 fallback 不可见**：Feign `Retryer` 在熔断内侧跑完全部尝试，fallback 只在**最终失败**时执行一次——降级值不会因"三次尝试各失败一次"而重复生成副作用（这也是 §9 策略允许做缓存写回的前提）。

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

（简化还原。）真实实现有三个细节值得对照：`factory.create(...)` 传入的**不只是 target.name()**——还有 group（`@FeignClient` 的 `path`/contextId 参与派生，对应 `14_circuit-breaker.md` §6.1 的命名问题，即"熔断器按 name+方法 configKey 维度注册"）；fallback 实例**按 configKey 缓存**，`create(cause)` 产出的兜底对象会被方法调用复用，工厂内部无需自缓存；`circuitBreaker.run(supplier, fallbackFunction)` 就是 Spring Cloud CircuitBreaker 的通用签名——**fallback 分支的实现完全在 CB 抽象层，与 Feign 无关**，这解释了为什么同一套 fallback 语义可以从 Hystrix 平移到 Resilience4j。

# 6. fallback vs fallbackFactory 选择

| 场景 | 推荐 |
|------|------|
| 降级逻辑简单，不关心失败原因 | `fallback` |
| 需要根据异常类型做不同处理 | `fallbackFactory` |
| 需要记录异常日志 | `fallbackFactory` |
| 404 返回 null，其他错误抛异常 | `fallbackFactory` |
| 根据错误码返回不同的兜底值 | `fallbackFactory` |

经验判据：**只要出现"所有失败共用一个兜底值"，用 `fallback`；需要区分"资源不存在 / 下游故障 / 业务拒绝"任何一档，用 `fallbackFactory`**。团队内做统一约定更实际——混用两种模式会让"看注解猜降级行为"失效，CR 时也容易漏检（例如有人给写接口配了 `fallback` 却返回了成功对象，`fallbackFactory` 版才带着那条"必须抛异常"的注释）。

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

## 7.1 按 root cause 分派的两个坑

`findRootCause` 向下钻到底的策略与 `instanceof FeignException.NotFound` 这类**顶层类型判断**互相干扰：`NotFound` 通常没有 cause，钻到底还是它自己没问题；但 `RetryableException` 包着 `SocketTimeoutException` 时，顶层看到的是 Retryable、root 看到的是 SocketTimeout——**同一故障按不同起点分派会命中不同分支**，团队要固定一种口径（推荐：先按顶层 Feign 异常族分派，落不进再看 root）。另一坑是**自定义 ErrorDecoder 会重塑分派表**：一旦按 `8_error-handling.md` §3 把 HTTP 错误翻译成 `BusinessException`，这里的 `NotFound/InternalServerError` 分支就永远不会命中，分派条件要跟着改成业务异常的错误码——两套翻译层不同步，就是"降级逻辑改了没生效"的经典成因。

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

## 8.1 两套体系的关系比"选哪个"更重要

启用 `spring-cloud-starter-alibaba-sentinel` 的 Feign 集成后，`@FeignClient` 的 fallback **改由 Sentinel 的熔断体系驱动**（`SentinelFeign` Builder 接管，`14_circuit-breaker.md` 的 r4j 开关和配置整体失效）——两者不是并存可选，而是**互斥的驱动层**，yml 里同时留着两套配置只会误导后人。而 `@SentinelResource` 是**业务方法级**的切面兜底，与 Feign 客户端级 fallback 作用在不同粒度，可以叠加：典型分工是 Feign fallback 处理"这个远程调用挂了返回什么"，`@SentinelResource` 处理"整个本地编排方法（含多次远程调用）超限/异常时怎么短路"。选型的真实判据是**团队主熔断栈是谁**：全司统一 Sentinel 就用 Sentinel 的 Feign 集成；没有历史包袱则按本篇（R4j + Spring 标准抽象）。

# 9. 降级策略设计

## 9.1 策略 1：返回兜底值

```java
return new User(id, "N/A", 0);  // 适合展示场景
```

要求展示层能识别占位值（配合 `14_circuit-breaker.md` §13.7 的来源标记），否则"N/A"会流进导出和报表。

## 9.2 策略 2：返回缓存数据

```java
return redisTemplate.opsForValue().get("user:" + id);
```

命中率是生命线：降级才想起缓存，而平时没人写这个缓存，等于没有降级。正确形态是**读路径旁路写缓存**（正常查询顺手回写 + TTL），或独立的同步任务预热；同时给缓存读设置短超时（Redis 也可能同时挂——共享基础设施的故障相关性要在预案里想过）。

## 9.3 策略 3：抛出业务异常

```java
throw new ServiceDegradedException("用户服务暂不可用，请稍后重试");
```

写操作与核心链路的默认选择：把"做不了"明确告知上游，让上层决定补偿（异步重试队列、人工工单）。它是唯一**不掩盖事实**的降级。

## 9.4 策略 4：静默失败

```java
return null;  // 上游处理 null，前端展示「加载失败」
```

最危险的一档：null 沿调用链扩散，NPE 在离故障点十层深的地方爆炸。要用就必须**在产生 null 的位置记降级事件**（指标 + 日志），且只用于"该字段本来就可空"的语义位。

## 9.5 策略选择矩阵

| | 读操作 | 写操作 |
|------|--------|--------|
| **非核心链路**（推荐位、评价数） | 兜底值/过期缓存 | 静默失败（异步补偿可选） |
| **核心链路**（下单、支付前置查询） | 缓存优先，无缓存抛业务异常 | 抛业务异常，绝不假成功 |

再加两个维度：**是否有异步补偿通道**（有则写操作可以"收下请求、落队列、稍后执行"，把强一致换成交付承诺）；**降级时长**（秒级抖动用缓存无所谓，预期小时级故障要检查缓存 TTL 是否会让数据陈旧得不可接受）。策略本身不复杂，复杂的是**每个接口在上线前都被问过这一组问题**（`25_best-practices.md` 的检查清单化）。

# 10. 最佳实践

1. **降级逻辑必须在 Fallback 类内部闭环** — 不要再调用其他远程服务（降级链路里再发生远程调用 = 故障时的连环放大；若必须调 §9.2 的 Redis，它要走独立的短超时客户端）
2. **写操作降级要谨慎** — POST/PUT/DELETE 失败时，返回 null 可能会误导调用方
3. **FallbackFactory 中不要做耗时操作** — 如查数据库、调 Redis 等可能影响熔断器恢复（工厂廉价原则见 §3.1）
4. **记录降级事件** — 在 FallbackFactory 中记录日志或发监控指标；指标维度按 configKey（`14_circuit-breaker.md` §12），日志注意风暴限频
5. **熔断器开启时所有请求都走 Fallback** — 要确保 Fallback 能应对全量流量（兜底路径平时零流量、故障时 100% 流量，是全系统最没被压测过的代码——降级演练要进故障演习清单）
6. **fallback 与 decode404 选一个管 404** — 同一个"资源不存在"语义别在 ErrorDecoder、fallback 分派、decode404 三处各写一套（§7.1 的翻译层同步问题），团队定一处收口
