# 1. 概述

Sentinel 通过 `SentinelFeign` 和 `@SentinelRestTemplate` 为 Feign 声明式 HTTP 调用提供流量防护。

```text
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

## 1.1 这条路线的本质：给 Feign 换 InvocationHandler

开启集成后 SCA 提供的是**替换掉整个 Feign.Builder** 的实现——代理的 handler 从原生 `FeignInvocationHandler` 换成 `SentinelInvocationHandler`，`SphU.entry` 包住"一次完整的方法调用"。由此推导出后面所有细节，值得先立起来：**① Entry 粒度是"方法调用"而非"HTTP 尝试"**——Feign 内部的 Retryer 循环若存在（本路线已禁用，§5.1）全在 Entry 内完成，熔断统计看到的是最终结果；**② entryType=OUT**——出向资源不参与系统保护口径与入口统计（`6_system_protection.md` §1.1 的消费方镜像）；**③ 与 Spring Cloud CircuitBreaker 路线互斥**：Feign 系列讲的 resilience4j 路线（`circuitbreaker.enabled`，Targeter 换双层代理）与这条 SCA 路线各有一个 fallback 语义，**同开等于两层代理叠两个兜底入口**，行为以谁在更外层为准且统计互盲——二选一（本系列选 Sentinel 就是选这条）。

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

## 2.1 "无需额外依赖"要加一个开关前提

依赖确实两个就够，但"自动配置"**默认不生效**（见 §5 勘误的开关）——条件装配类带的是 `@ConditionalOnProperty(feign.sentinel.enabled)`，开关不拨它就不装。验证是否真接上了，看两点：簇点链路出现 OUT 型 Feign 资源（`GET:http://...` 族）、fallback 在超阈值时被调用；都没有=还是裸 Feign 在跑。

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

## 3.3 SCA 路线的 fallback 语义与 Feign 系列不同（关键差异）

对照记忆：Resilience4j 路线里 `fallback` 管业务异常、`blockHandler` 管熔断拒绝、被拦与出错走不同门（Feign 系列 `16_fallback.md` §2 那张表）。**SCA Sentinel-Feign 路线只有一扇门**：`SentinelInvocationHandler` 对 **BlockException 和一切业务异常统一调 fallback/fallbackFactory**（异常本身仍经 handler 上抛前的处理路径：fallback 缺失时 BlockException 原样抛、业务异常原样抛）。三条后果：**① 没有熔断统计豁免**——Resilience4j 里"降级返回值不计失败"的贴心语义在这里不存在，反正失败已计数、fallback 只是善后，差异不大；**② fallbackFactory 的 cause 分派表更重要**——§3.2 的 instanceof 分流正是本路线下的"blockHandler 替身"，FlowException（这里含义是**下游返回的流控**？注意：consumer 本地给 Feign 资源配流控规则几乎无意义，容量规则权威在提供方 §9.5，`cause` 里出现的高频成员其实是下游 429 翻译来的 FeignException 与本地 DegradeException）；**③ createUser 返回 null 的坑加倍**——本路线 fallback 吞掉一切异常，调用方对"null=写失败"几乎无感知，写接口降级要么返回可判别结构要么在 fallback 里显式 rethrow，Feign 系列 `16_fallback.md` §3 的"降级可观测"纪律在这里没有 blockHandler 那条天然埋点帮忙。

# 4. 资源名规则

Sentinel Feign 自动生成的资源名格式：

```text
HTTP 方法:协议://服务名/路径

示例:
GET:http://user-service/api/user/{id}
POST:http://order-service/api/order
```

资源名包含 HTTP 方法前缀，便于在 Dashboard 中区分同路径的读写操作。

## 4.1 【纠偏】模板名是理想态：以簇点链路现名为准

格式方向正确（`方法:URL`，host 位就是 `@FeignClient name`），但 `/{id}` 这种路径变量在部分 SCA 版本里是**以构建后的真实 URL 记名**（`/api/user/123` 各成资源）——即消费侧资源基数随路径参数爆炸的风险真实存在（`12_gateway_integration.md` §5.1 ① 讲的是网关侧匹配原始 URI，这里是调用侧另一处"路径进名字"的账）。可执行纪律三条：**① 规则 JSON 的 resource 永远从簇点链路复制**（`11_dubbo_integration.md` §3.2 同款，先触发一次调用）；**② 基数观察**：簇点链路里同族资源刷屏时，把容量类规则上移到网关/提供方层，consumer 侧熔断规则只保留少数关键模板名；**③ 自定义收敛**：高频接口改用手写 `SphU.entry("feign:getUser", EntryType.OUT, ...)` 包一层（语义与自动资源名同构，名字归你管）。方法前缀那半句是对的：读写同路径确实靠 GET:/POST: 分开。

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

## 5.1 【纠偏】两处开关/键名校准

**`feign.sentinel.enabled` 的默认值不是 true**：这行条件装配历史默认**不装配**（`@ConditionalOnProperty` 无 matchIfMissing），必须显式开启——"配了 fallback 从没被调过"的第一现场就是忘了这行。Boot3/SCA 2023 一代键名有向 `spring.cloud.*` 家族收敛的迁移运动，**以所用 SCA 版本文档核对当前键**，验证方式用 §2.1 的两条硬证据而不是相信键写对了。**`feign.client.config.*` 在 Boot3 世代已废**：超时这类 Feign 框架键迁到了 `spring.cloud.openfeign.client.config.*`（Feign 系列 `25_best-practices.md` §7.2 的键名迁移清单），旧键写在这里是"配置静默无效、超时维持默认值"的经典来源——本系列跨两代文档时，超时值最终生效与否都到目标实例上实测一次。

## 5.2 本路线下 Retryer 被替换为 NEVER_RETRY

SCA 的 `SentinelFeign.Builder` 装配时把 Feign 原生 Retryer 置为**从不重试**：设计立场是"重试与快速失败矛盾"——让熔断统计看到干净的单次结果、降级路径单一可预期。后果两条：Feign 系列 `9_retry.md` §1.1 的"默认 NEVER_RETRY"在这里成为**强制**（配 `Retryer` Bean 无效，别在集成路线上找重试开关）；要重试能力就交给上一层——LB 层换实例重试（Feign 系列 `13_loadbalancer.md` §10，`spring.cloud.loadbalancer.retry.enabled`，且仅对 GET 等幂等语义安全）或 Resilience4j Retry 路线（那意味着整体走 §1.1 的另一条路）。超时预算公式随之简化：本路线单次调用最坏耗时 ≈ readTimeout（无重试乘数），§9.2 的预算表比 Feign+Retryer 组合干净得多。

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

## 6.1 两条路线的兜底形态差异一句话

RestTemplate 路线与 Feign 路线的降级哲学相反：这里 blockHandler 返回的是一个 **200 壳响应**（`10_spring_cloud_integration.md` §5.1 ④ 的坑），调用方拿到字符串继续跑；Feign 路线则统一进 fallback 返回业务对象。同一下游同时被两种方式调用时，**资源名体系也是两套**（`GET:http://svc/path` 自动名 vs RestTemplate 的 URL 名），规则别互相抄；新建出向调用优先 Feign/声明式，RestTemplate 保护留给存量。

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

## 7.1 【纠偏】统计不需要"标记"，ErrorDecoder 只管异常类型

注释暗示"503 要特殊处理才计入熔断统计"——不成立：本路线 `SentinelInvocationHandler` 对**任何抛出的异常**（含 Default ErrorDecoder 的 FeignException）统一 `entry.setError`，非 2xx 天然计为失败，ErrorDecoder 无权也无需"放行统计"。它真正影响的是两件事：**fallbackFactory 里 cause 的类型**（§3.3 的分派表按它分流——把 503 译成 RetryableException 反而让分派逻辑多认一个没意义的类型，本路线 Retryer 已禁用（§5.2），RetryableException 在这里就只是个普通异常）；以及**异常是否该计**。"404 不计入"的正解不是 ErrorDecoder，是 **`decode404`**：打开后 404 直接走 Decoder 返回 null/空集合，根本不产生异常（Feign 系列 `20_response-mapping.md` §6 的空响应矩阵；代价是"真 404 故障"也被吞进成功统计，Feign 系列 `8_error-handling.md` §7 的权衡在熔断语境加倍成立——下游路由丢失会让你的熔断器永远健康）。真正想"某些状态码不计失败"的进阶解：自定义 ErrorDecoder 返回一个被 `exceptionsToIgnore` 语义排除的类型——本路线没有注解属性可配，只能包 handler 或接受全计。

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

## 8.1 这份 JSON 要过的三道核对

**① 资源名**：`/{id}` 模板名写法须与簇点链路现名逐字一致（§4.1，写错静默失明）；**② count 的双重含义**：第一条 `200` 是**慢调用判定线（ms）**且必须 < §5 的 readTimeout=10000（否则慢调用永不进统计，`5_circuit_breaker.md` §10.1 的隐藏前置）；第二条 `0.3`=异常比例 30%；**③ timeWindow 与超时无关**（§9.2 勘误的主体）——它是 OPEN 时长，与 readTimeout 大小关系不构成任何约束；想约束的是"熔断判定的时间输入"：RT 模式统计的是 exit 结算的端到端耗时，超时先发生则该请求以异常身份离场、走的是比例/数量模式而非慢调用线，两类规则各配时清楚自己统计的是哪个阶段的病。

# 9. 最佳实践

1. **fallback vs fallbackFactory**：需要感知异常类型时使用 `fallbackFactory`
2. **超时设置**：Feign `readTimeout` < Sentinel 熔断 `timeWindow`，确保超时先于熔断
3. **resource 命名**：Sentinel 自动生成的资源名足够清晰，一般不需要自定义
4. **降级返回值**：始终返回有意义的兜底对象，不要返回 `null`
5. **熔断优先于流控**：对 Feign 调用优先配置熔断规则，流控作为补充

## 9.1 第 2 条是全系列最该删掉重写的实践

"readTimeout < timeWindow"比较的是**两个不相干的时间**：readTimeout 决定单次调用何时失败，timeWindow 决定失败后短路多久，二者没有先后约束关系（10s OPEN 配 1s readTimeout 完全正常）。想表达"超时类故障如何参与熔断"的正确不等式组是：**`RT 熔断阈值(count) < readTimeout`**（慢调用统计可得的必要条件，§8.1 ②）；**`readTimeout × 1 ≤ 熔断窗口内允许的线程占用预算`**——超时占线程、被短路不占，熔断的价值恰恰发生在超时之后；预算上本路线单次=timeout（无重试乘数，§5.2），写进 `9_retry.md` §9.1 的乘积不等式时该层系数为 1。第 3 条按 §4.1 的现实打折：自动名"够用"的前提是无路径变量或基数受控，大促接口先做基数检查再谈要不要自定义。第 5 条补一句权威归属：consumer 侧给 Feign 资源配流控的正当场景只剩"自我保护出口线程池"（线程数模式、按本应用容量），下游容量真相永远在提供方一侧——与 `11_dubbo_integration.md` §1.1 的"容量类规则单侧权威"同一条纪律的 HTTP 表达。

# 10. 相关阅读

Feign 侧的代理结构与熔断接线全景在 Feign 系列 `21_source-dynamic-proxy.md`/`14_circuit-breaker.md`（两路线对照的最佳读本）；DegradeRule 字段账在 `5_circuit_breaker.md`；错误码翻译在 Feign 系列 `8_error-handling.md`；origin 传播断点在 `7_authority_rule.md` §2.2；§5.1 键名迁移总表在 Feign 系列 `25_best-practices.md` §7.2。
