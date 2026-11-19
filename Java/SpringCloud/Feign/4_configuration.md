# 1. 配置体系概览

## 1.1 四个配置层级

Spring Cloud OpenFeign 支持多级配置：

```text
全局默认配置
  ├── 指定 FeignClient 名称配置
  │     └── 指定 contextId 配置（部分版本支持）
  └── 自定义 Configuration 类配置
        └── @FeignClient(configuration = ...)
```

理解这套体系的关键是一个容易被忽略的事实：**每个 @FeignClient 拥有自己独立的"子配置上下文"**。`FeignClientFactoryBean` 构建代理时，会把 `default` 的 yml 配置、当前 name/contextId 的 yml 配置、`configuration` 注解指定的类，依次注册为一个临时上下文的配置源，再从中按类型解析 `Contract`、`Encoder`、`Client` 等组件。所谓"多级覆盖"，本质就是同一个类型在子上下文中存在多个候选时的解析优先级问题。

# 2. application.yml 全局配置

```yaml
spring:
  cloud:
    openfeign:
      # 全局客户端配置
      client:
        config:
          default:                          # default 表示全局默认
            connect-timeout: 5000           # 连接超时 (ms)
            read-timeout: 10000             # 读取超时 (ms)
            logger-level: BASIC             # 日志级别: NONE/BASIC/HEADERS/FULL
            retryer: com.example.MyRetryer  # 重试器类全限定名
            error-decoder: com.example.MyErrorDecoder
            request-interceptors:
              - com.example.AuthInterceptor
              - com.example.TraceInterceptor
            decode404: false                # 404 时是否解码响应体
            encoder: com.example.MyEncoder
            decoder: com.example.MyDecoder
            contract: feign.Contract$Default

          # 指定服务名配置（覆盖 default）
          user-service:
            connect-timeout: 2000
            read-timeout: 5000
            logger-level: FULL

          # 如果同一个服务有多个 FeignClient，用 contextId 区分
          user-service-api-v2:
            connect-timeout: 3000

      # 压缩配置
      compression:
        request:
          enabled: true
          mime-types: application/json,application/xml
          min-request-size: 2048            # 最小压缩阈值 (bytes)
        response:
          enabled: true
          useGzipDecoder: true              # 使用 Gzip 解码器

      # OkHttp / HttpClient 启用
      httpclient:
        enabled: true                        # 启用 Apache HttpClient 5
        max-connections: 200
        max-connections-per-route: 50
        time-to-live: 900                   # 连接存活时间 (s)
        time-to-live-unit: seconds
      okhttp:
        enabled: false

      # 熔断器（已废弃，使用 spring.cloud.circuitbreaker.resilience4j 替代）
      circuitbreaker:
        enabled: true
```

按作用范围把这些配置分三类记：**按客户端的**（`client.config.<name>` 下的超时/日志/组件类）、**全局开关型的**（`compression.*`、`httpclient.*`、`okhttp.*`、`circuitbreaker.*`，所有客户端共享）、**已迁移的**（`circuitbreaker.*` 在新版本中由 `spring.cloud.loadbalancer`/resilience4j 自身配置接管，旧键仅作兼容）。

# 3. 指定 FeignClient 名称配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-service:                     # name/value 的值
            connect-timeout: 3000
            read-timeout: 6000
          product-service:
            connect-timeout: 5000
```

## 3.1 配置键到底匹配谁

配置 key 与 `@FeignClient(name = "xxx")` 中的 `name` 一致。更精确的规则是：**先按 contextId 查找，找不到再按 name 查找**。没有指定 contextId 时二者相同，所以"用 name 当 key"在单客户端场景总是成立；一旦出现同服务多客户端，只有 contextId 这一份配置能生效——这是 §7 存在的理由。

# 4. 自定义配置类

```java
public class UserFeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }

    @Bean
    public RequestInterceptor authInterceptor() {
        return template -> {
            String token = AuthContext.getToken();
            if (token != null) {
                template.header("Authorization", "Bearer " + token);
            }
        };
    }

    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(100, 1000, 3);
    }
}
```

```java
@FeignClient(name = "user-service", configuration = UserFeignConfig.class)
public interface UserClient { }
```

`configuration` 指定类的价值在于它能表达 yml 表达不了的东西：**有状态/有依赖的组件实例**（拦截器要注入 Token 管理器、Decoder 要注入 ObjectMapper 定制规则）。yml 里的 `retryer: com.example.MyRetryer` 只能按无参构造反射创建。两种入口的选择标准：纯参数→yml；需要协作对象→配置类。

# 5. 自定义配置类的注意事项

**配置类不要被 Spring 组件扫描到**，否则会成为全局配置：

```java
// ↓ 不能加 @Configuration，否则会被全局扫描
// ↓ 不要放在 @SpringBootApplication 所在包的子包下
public class UserFeignConfig {
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

有两种方式避免：

1. **移到不被扫描的包**（如 `com.example.feign.config` 放在 `@ComponentScan` 之外）
2. **使用 `excludeFilters`**：

```java
@Configuration
@ComponentScan(basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.REGEX,
        pattern = "com\\.example\\..*\\.FeignConfig"
    ))
```

## 5.1 为什么一旦扫到就"全局污染"

被主容器扫到的 `@Configuration`，其 `@Bean` 会注册进**主上下文**；而客户端子上下文解析组件时以主上下文为 parent——所有客户端都能向上找到这个 Bean，"专属配置"自然升级成了全局配置。典型的连锁症状：给 A 服务加的鉴权拦截器开始往 B、C 服务的请求里塞同样的头。规则背下来即可：**按服务的配置类必须对组件扫描不可见**；只有确定对所有客户端都成立的组件（全局 TraceId 注入、统一 ObjectMapper）才放主上下文。

# 6. 代码配置 vs YAML 配置优先级

```text
@FeignClient(configuration = ...) @Bean 配置  (最高)
    ↑ 覆盖
指定服务名 YAML 配置 (spring.cloud.openfeign.client.config.user-service)
    ↑ 覆盖
default YAML 配置 (spring.cloud.openfeign.client.config.default)
    ↑ 覆盖
Spring Boot 自动配置默认值  (最低)
```

优先级成立的原因回到 §1.1：yml 各层最终也被翻译成配置源注册进同一个子上下文，显式 `configuration` 的 Bean 定义先于 yml 派生的定义参与按类型解析。**两个例外**要知道：`connect-timeout`/`read-timeout` 没有对应的组件 Bean 形态，只走 yml（注解侧无入口，不参与竞争）；`Retryer` 若 yml 与 @Bean 同时出现，某些 4.x 小版本的表现取决于 Bean 注册顺序，建议**同一组件只用一种入口**，不要依赖这个覆盖细节。

# 7. contextId 的作用

当同一个服务有多个 `@FeignClient` 接口时：

```java
@FeignClient(name = "user-service", contextId = "user-client-v1")
public interface UserClientV1 { }

@FeignClient(name = "user-service", contextId = "user-client-v2")
public interface UserClientV2 { }
```

相应的 YAML 配置：

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          user-client-v1:
            connect-timeout: 3000
          user-client-v2:
            connect-timeout: 5000
```

如果不指定 contextId，默认使用 name 的值。

## 7.1 name 与 contextId 的分工

两个属性服务于完全不同的机制，混用是高频困惑源：

| 属性 | 职责 |
|------|------|
| `name`（或 `value`） | 负载均衡目标：作为服务名去注册中心发现实例；SpEL/占位符在此解析 |
| `contextId` | 配置键（§3.1）、Spring Bean 名称、熔断器分组与 fallback 关联的标识 |

同服务多接口（如按业务拆分的 v1/v2、读写分离的两个客户端）时，**不指定 contextId 会直接 Bean 定义冲突**（两个 FactoryBean 都想注册同名单元），报错信息里的 `bean with that name has already been defined` 说的就是这件事；此时正确做法不是打开 `allow-bean-definition-overriding`，而是给每个接口起唯一 contextId。

## 7.2 url 属性与配置体系的关系

`@FeignClient(url = "http://localhost:8080")` 直连模式会**绕过 name 的负载均衡语义**（`LoadBalancerFeignClient`/`FeignBlockingLoadBalancerClient` 不再参与实例选择），但 `client.config.<contextId>` 的超时等配置依然生效。灰度测试里"配了超时怎么还不生效"的问题，多半是把 url 写死在了注解上而配置键仍按服务名查找——先确认自己走的是哪条路。

# 8. 全局熔断配置（Resilience4j）

```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true

resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10000ms
        permitted-number-of-calls-in-half-open-state: 5
        automatic-transition-from-open-to-half-open-enabled: true
  timelimiter:
    configs:
      default:
        timeout-duration: 5s
```

两个与配置键名相关的实践点：熔断器实例名默认由 `FeignClientName#methodKey` 拼装，方法签名里含 `$`、`/` 等字符会让 Resilience4j 的分组配置难以命中，`alphanumeric-ids.enabled: true` 可以换成纯字母数字 ID（见 §9 末行）；`timelimiter` 的 5s 与 Feign 的 read-timeout 是**叠加关系而非替代**——谁先触发谁产生异常，建议让 Feign 超时略小于 TimeLimiter，保证拿到的异常类型是可预期的 `SocketTimeoutException` 家族（细节见 `14_circuit-breaker.md`）。

# 9. 常用配置速查表

| 配置项路径 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `spring.cloud.openfeign.client.config.<name>.connect-timeout` | int | 10000 | 连接超时(ms) |
| `spring.cloud.openfeign.client.config.<name>.read-timeout` | int | 60000 | 读取超时(ms) |
| `spring.cloud.openfeign.client.config.<name>.logger-level` | enum | NONE | FULL/BASIC/HEADERS/NONE |
| `spring.cloud.openfeign.client.config.<name>.decode404` | boolean | false | 404 不抛异常 |
| `spring.cloud.openfeign.client.config.<name>.dismiss404` | boolean | false | 404 返回 null（需配合 decode404） |
| `spring.cloud.openfeign.compression.request.enabled` | boolean | false | 请求 Gzip 压缩 |
| `spring.cloud.openfeign.compression.response.enabled` | boolean | false | 响应 Gzip 解压 |
| `spring.cloud.openfeign.httpclient.enabled` | boolean | true | 启用 Apache HttpClient 5 |
| `spring.cloud.openfeign.okhttp.enabled` | boolean | false | 启用 OkHttp |
| `spring.cloud.openfeign.circuitbreaker.enabled` | boolean | false | 启用熔断器 |
| `spring.cloud.openfeign.circuitbreaker.alphanumeric-ids.enabled` | boolean | false | 熔断器名称只含字母数字 |

速查表里最常被误用的是 `decode404`/`dismiss404` 组合：`decode404: true` 让 404 进入 Decoder（配合返回 `Optional`/空对象表达"查无此记录"的正常语义）；`dismiss404` 直接返回 null。二者都是为了把"资源不存在"从异常降级成业务返回值——但如果下游用 404 表达"路由都没配"，这套配置会把真实故障吞成"没找到"，启用前必须和提供方确认 404 语义（延伸阅读：`8_error-handling.md`）。
