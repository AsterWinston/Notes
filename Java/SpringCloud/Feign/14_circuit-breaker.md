# 1. 概述

## 1.1 状态机模型

Spring Cloud OpenFeign 从**2020.0.x 版本**开始，将熔断支持从 Hystrix 迁移到 **Resilience4j**。当 Feign 调用失败率达到阈值时，熔断器打开，直接走降级逻辑，避免雪崩效应。

```text
正常状态 (CLOSED): 请求正常通过 → 统计失败率
    │
    │ 失败率 >= 阈值 (如 50%)
    ▼
熔断状态 (OPEN): 直接走 fallback → 等待冷却时间
    │
    │ 冷却时间到
    ▼
半开状态 (HALF_OPEN): 允许少量请求通过
    │
    ├── 成功 → 恢复到 CLOSED
    └── 失败 → 重新进入 OPEN
```

## 1.2 熔断在调用栈的哪一层

Feign 的熔断不是 `Client` 层组件，而是**代理层装饰器**：`FeignClientFactoryBean` 构建方法调用时会把整个 `SynchronousMethodHandler`（含重试、编解码）包进 CircuitBreaker。层级关系值得背下来，它决定每个异常/指标的归属：

```text
业务调用 → 熔断装饰器(CircuitBreaker + TimeLimiter + Bulkhead + fallback)
              └→ Feign 方法执行(Contract → 拦截器 → Retryer 重试 → Client → LoadBalancer)
```

由此可推出三条口径：Feign 内部的重试对熔断**不可见**（熔断只数最终结果）；fallback 捕获的是"整次方法调用"的失败而非单次网络尝试；熔断统计的 RT 含重试累计耗时。

# 2. 依赖引入

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<!-- 如果需要响应式 API -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-reactor</artifactId>
</dependency>
```

## 2.1 熔断生效的完整开关链

```text
依赖(本§) → spring.cloud.openfeign.circuitbreaker.enabled=true(§3)
         → fallback/fallbackFactory 属性(§4/§5)
```

三段缺一都没有降级能力，且**都失败得静默**：缺依赖时 enabled 配了也无处生效；只配 fallback 不开 enabled，`@FeignClient` 的 fallback 属性被直接忽略（编译过、启动过、线上炸——新手事故率最高的一环）。验收方法不是看配置，而是制造一次失败看是否走到 fallback 分支。

# 3. 基本配置

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
        alphanumeric-ids:
          enabled: true          # Group 名只允许字母数字

resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED   # 滑动窗口类型
        sliding-window-size: 10            # 窗口大小
        minimum-number-of-calls: 5         # 最小调用次数才开始统计
        failure-rate-threshold: 50         # 失败率阈值(%)
        wait-duration-in-open-state: 10000ms      # OPEN 持续时间
        permitted-number-of-calls-in-half-open-state: 5   # HALF_OPEN 允许请求数
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:                 # 记录为失败的异常
          - org.springframework.web.client.HttpServerErrorException
          - java.util.concurrent.TimeoutException
          - java.io.IOException
        ignore-exceptions:                 # 忽略的异常（不计入失败统计）
          - com.example.exception.BusinessException
    instances:
      user-service:
        base-config: default
        sliding-window-size: 20
        wait-duration-in-open-state: 15000ms

  timelimiter:
    configs:
      default:
        timeout-duration: 5s               # 调用超时熔断
        cancel-running-future: true        # 超时后取消任务
    instances:
      user-service:
        timeout-duration: 3s
```

## 3.1 统计口径三要素

**滑窗类型**：COUNT_BASED 按最近 N 次调用统计，低峰期几个失败就能触发（窗口小、样本敏感）；TIME_BASED 按最近 N 秒，流量平稳场景更可控。**minimum-number-of-calls** 是小流量保护：窗口凑不满最小样本前不算失败率，防止"1/1=100% 就熔断"。**slow-call-rate**（示例未列）是第三种触发维度：RT 超过 `slow-call-duration-threshold` 的调用按失败计——对"下游没报错但变慢"的故障，它比异常统计更早救场，建议与 failure-rate 同时启用。

## 3.2 record/ignore 异常名单的真实默认值

不配这两个名单时，Resilience4j 默认把**所有 RuntimeException 计为失败**——意味着 Feign 的 4xx（`FeignClientException`，多半是参数/权限问题）也会计数。示例显式列名单是对的，但列法要注意包含关系：Feign 侧常见应计入的是 `IOException`（网络层，status=-1 的 RetryableException 最终形态）与 `FeignServerException`（5xx），应排除的是业务翻译异常与 `FeignClientException` 家族——`8_error-handling.md` §9.7 讲的正是这套口径对齐；`TimeoutException`（TimeLimiter 产物，§7）必须计入，否则超时不驱动熔断。

# 4. FeignClient 使用 Fallback

```java
@FeignClient(name = "user-service",
             fallback = UserClientFallback.class)
public interface UserClient {

    @GetMapping("/users/{id}")
    User getUser(@PathVariable Long id);
}
```

```java
@Component
public class UserClientFallback implements UserClient {

    @Override
    public User getUser(Long id) {
        // 降级逻辑：返回缓存数据或兜底值
        return new User(id, "N/A", 0);
    }
}
```

Spring Cloud OpenFeign 会自动将 fallback 与 Resilience4j 熔断器绑定。

## 4.1 fallback 实例的查找规则

`fallback = X.class` 声明的是**类型**，运行期从 Spring 容器取**实例**（按类型 findBean）。由此产生两条使用规则：fallback 类必须注册为 Bean（示例的 `@Component`），漏注册的症状是启动不报错、降级时 `IllegalStateException: Fallback instance not found`；同接口多个 fallback 实现时按类型取会歧义，此时应改用 `fallbackFactory`（§5）——工厂模式由你显式 `new` 出 fallback 实例，容器查找问题消失。"implements UserClient 返回兜底对象"的写法自带一项编译红利：提供方接口加方法，fallback 立刻编译失败，倒逼降级逻辑同步演进，这是它优于反射 Mock 的结构性原因。

# 5. FallbackFactory（获取异常）

```java
@FeignClient(name = "user-service",
             fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient { }
```

```java
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        return new UserClient() {
            @Override
            public User getUser(Long id) {
                log.error("调用 user-service 失败, userId={}", id, cause);
                if (cause instanceof FeignException.NotFound) {
                    return null;
                }
                return new User(id, "N/A", 0);
            }
        };
    }
}
```

## 5.1 按异常类型分派降级策略

拿到 `cause` 之后，fallback 才可能"分类救火"：`FeignException.NotFound`（资源本来就没有）返回 null 不打错误日志；`RetryableException`/IOException（下游故障）返回缓存兜底并告警；`BusinessException`（§3.2 名单里的透传）原样上抛**而不是降级**——把业务规则错误"降"成默认值会掩盖真实失败，这是分派逻辑里最重要的一条红线（`create` 返回的对象里重新 `throw cause` 即可）。另外 `create(Throwable)` 每次失败调用一次，实现必须廉价；把取缓存等重操作做进工厂方法等于给故障链路再压一遍。

# 6. 配置作用于全局/特定服务

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        failure-rate-threshold: 50
    instances:
      user-service:           # 对应 @FeignClient name
        failure-rate-threshold: 30
        sliding-window-size: 20
      order-service:
        failure-rate-threshold: 40
```

## 6.1 instances 键名必须对上真实熔断器名

Feign 场景的熔断器实例名不是裸服务名，而是由 **contextId + 方法签名（configKey）** 派生（默认形如 `user-service-com.example.UserClient#getUser(Long)` 的结构，这正是 `alphanumeric-ids` 开关存在的原因——把特殊字符规整成可配置的 ID）。`instances.<key>` 写不中真实名就会"看似配置了实则走 default"。两个落地做法：上线前经 actuator `/circuitbreakers` 端点抄一次真实注册名（§12 同源）；或干脆不追求方法级差异，统一用 `configs.default` + `group`/tag 维度管理。§6 示例的服务名写法在部分版本恰好可作 group 匹配，**别靠运气，核对真实名**。

# 7. CircuitBreaker + TimeLimiter 协同

```text
Feign 调用
    │
    ├── TimeLimiter  (超时控制)
    │       └── 超时后抛出 TimeoutException → 触发 CircuitBreaker
    │
    ├── CircuitBreaker  (状态机)
    │       ├── CLOSED → 放行请求
    │       └── OPEN → 直接 fallback
    │
    └── Fallback / FallbackFactory
```

**重要**：TimeLimiter 超时时间应 **小于** Feign readTimeout，否则 Feign 的读取超时可能先触发，导致 TimeLimiter 无法生效。

```yaml
spring.cloud.openfeign.client.config.user-service.read-timeout: 8000  # 8s
resilience4j.timelimiter.instances.user-service.timeout-duration: 5s   # 5s < 8s ✓
```

## 7.1 谁先超时决定你拿到什么异常

两个计时器是叠加关系，**谁短谁先响**，响法不同：TimeLimiter 先响 → 业务拿到 `TimeoutException`（r4j 包装），HTTP 底层调用仍在跑（`cancel-running-future` 只是尽力取消，连接要等 read-timeout 才真正归还池）；read-timeout 先响 → 拿到 Feign 的 IOException 家族，`TimeoutException` 名单形同虚设，若统计名单里只配了 TimeoutException，这类失败竟不进熔断统计。示例取 5s < 8s 的取向是"让熔断层统一拿到超时口径"；但按 `11_timeout-connection.md` §7 的连接占用视角，"read-timeout 略小、让真实网络异常先行"也成立——两种都有道理，**必须选定一种并让异常名单配套**，最坏的是两边参数互不知情。

# 8. Bulkhead（信号量隔离）

```yaml
resilience4j:
  bulkhead:
    configs:
      default:
        max-concurrent-calls: 20              # 最大并发数
        max-wait-duration: 500ms             # 等待超时
    instances:
      user-service:
        max-concurrent-calls: 10
```

**SemaphoreBulkhead**（信号量隔离）是 Spring Cloud + Feign 场景的默认选择（因为异步线程池隔离与 WebMVC 冲突）。

并发闸门的容量要与上下游对齐：`max-concurrent-calls` 小于 Tomcat 线程数时，超出部分要么等 500ms 要么抛 `MaxWaitDurationExceededException` 进 fallback——这是"一个慢下游拖死整个服务线程池"的标准解法（资源隔离的 Feign 形态）。参数换算基准：某客户端并发 ≈ 入口 QPS × 下游 P99 RT（利特尔法则），算出 20 就配 25，别用默认拍脑袋。线程池隔离（ThreadPoolBulkhead）只适合显式异步的调用点，在同步 MVC 链路里反而多一次线程切换。

# 9. 自定义 CircuitBreaker 工厂

```java
@Configuration
public class FeignCircuitBreakerConfig {

    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> {
            return new Resilience4JConfigBuilder(id)
                    .circuitBreakerConfig(CircuitBreakerConfig.custom()
                            .slidingWindowType(SlidingWindowType.COUNT_BASED)
                            .slidingWindowSize(10)
                            .failureRateThreshold(50)
                            .build())
                    .timeLimiterConfig(TimeLimiterConfig.custom()
                            .timeoutDuration(Duration.ofSeconds(5))
                            .build())
                    .build();
        });
    }
}
```

## 9.1 什么时候用代码配置而不是 yml

`configureDefault`/`configure(...).withCircuitBreakerConfig` 的优先级高于 resilience4j 的 yml 装配，适合三类需求：配置值要从外部系统动态计算（如按租户加载）、要把 CB+TL+Bulkhead 三种配置**原子地**放在一起防止键名漂移、或 yml 表达不了的谓词（按返回值判定失败）。代价是绕开了配置中心的可视化，团队要统一口径：同一参数只允许一边配，代码优先时 yml 对应段删干净，否则"改了 yml 怎么没生效"又是一轮排查。

# 10. 手动创建 CircuitBreaker FeignClient

```java
@Bean
public UserClient userClient(
        Resilience4JCircuitBreakerFactory circuitBreakerFactory,
        ObjectFactory<HttpMessageConverters> converters) {

    CircuitBreaker circuitBreaker = circuitBreakerFactory.create("user-service");

    UserClient directClient = Feign.builder()
            .encoder(new SpringEncoder(converters))
            .decoder(new SpringDecoder(converters))
            .target(UserClient.class, "http://user-service");

    // 包装为带熔断的代理
    return FeignCircuitBreaker.builder()
            .factory(circuitBreakerFactory)
            .target(directClient, UserClient.class);
}
```

此段为示意伪码：自动配置路线实际由 `FeignClientFactoryBean` 里的 `FeignCircuitBreakerTargeter` 完成同样包装（机制与 §1.2 层级图一致），`FeignCircuitBreaker` 的 API 在 4.x 中主要供框架内部使用。真正需要这段手工组装的场景罕见（比如给非 `@FeignClient` 手搓的 builder 客户端补熔断），且此时 `12_feign-builder.md` §9 的提醒同样适用——手动路线每补一个能力都要还一份债（示例里 LoadBalancer 就又缺席了）。

# 11. Hystrix 迁移指南

| Hystrix | Resilience4j 替代 |
|---------|------------------|
| `hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds` | `resilience4j.timelimiter.configs.default.timeout-duration` |
| `hystrix.command.default.circuitBreaker.requestVolumeThreshold` | `resilience4j.circuitbreaker.configs.default.sliding-window-size` |
| `hystrix.command.default.circuitBreaker.errorThresholdPercentage` | `resilience4j.circuitbreaker.configs.default.failure-rate-threshold` |
| `hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds` | `resilience4j.circuitbreaker.configs.default.wait-duration-in-open-state` |
| 线程池隔离 | `resilience4j.bulkhead` (信号量) 或 `resilience4j-threadpoolbulkhead` |

迁移的行为差异比键名映射更需要检查清单：Hystrix 的熔断粒度默认按"命令"（≈ 方法），与 §6.1 的命名问题同源；Hystrix 超时包含队列+执行全程，R4j 的 TimeLimiter 只管调用本身；最隐蔽的一条是 **Hystrix 默认对超时"线程池内快速失败"，而 R4j 同步链路的取消是尽力而为**（§7.1），迁移后"超时了但线程没立刻回来"属预期行为变化，容量参数要重算而不是照搬。

# 12. 熔断指标监控

Resilience4j 内置 Micrometer 支持，可直接接入 Prometheus + Grafana：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}

resilience4j:
  circuitbreaker:
    metrics:
      legacy:
        enabled: false
      enable-micrometer: true
```

可观测的核心指标：
- `resilience4j_circuitbreaker_state` — 熔断器状态 (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
- `resilience4j_circuitbreaker_calls_seconds` — 调用耗时
- `resilience4j_circuitbreaker_failure_rate` — 失败率
- `resilience4j_circuitbreaker_slow_call_rate` — 慢调用率

`name` 标签就是 §6.1 说的真实熔断器名——监控面板建好那天，顺手把标签值抄下来作为 instances 配置键的唯一事实来源，两个问题一起解决。最有告警价值的是 `_state`（OPEN 即事件）与 `not_permitted_calls_total`（被闸门挡下的量，Bulkhead/熔断生效的直接证据）；`_failure_rate` 适合做趋势看板，不建议直接当告警条件——它受窗口样本量影响，毛刺很多。

# 13. 最佳实践

1. **TimeLimiter 超时 < Feign readTimeout** — 让熔断器超时先于底层 HTTP 超时（两种方向都可，选定后异常名单配套，见 §7.1）
2. **降级逻辑要轻量** — Fallback 中避免再调用其他 Feign 接口
3. **配置 `record-exceptions`** — 明确哪些异常应计入失败
4. **配合 Bulkhead** — 在高并发场景限制并发数，防止资源耗尽
5. **HALF_OPEN 允许多个请求** — `permitted-number-of-calls-in-half-open-state` 不宜过小
6. **fallback 必须有覆盖测试** — 给每个带降级的客户端留一条"制造失败断言降级值"的集成用例，降级逻辑只在故障里执行，没有测试的 fallback 等于没有 fallback；
7. **降级值要能自证** — 兜底对象里带上来源标记（如 `User(id, "N/A", 0, fromCache=true)` 或专门的降级字段），让下游业务能区分"真数据"和"救火数据"，静默的默认值会在报表和风控里制造二次事故。
