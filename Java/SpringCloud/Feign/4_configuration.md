
# 1. 配置体系概览

Spring Cloud OpenFeign 支持多级配置：

```
全局默认配置
  ├── 指定 FeignClient 名称配置
  │     └── 指定 contextId 配置（部分版本支持）
  └── 自定义 Configuration 类配置
        └── @FeignClient(configuration = ...)
```

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
        enabled: true                        # 启用 Apache HttpClient
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

配置 key 与 `@FeignClient(name = "xxx")` 中的 `name` 一致。

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

# 6. 代码配置 vs YAML 配置优先级

```
@FeignClient(configuration = ...) @Bean 配置  (最高)
    ↑ 覆盖
指定服务名 YAML 配置 (spring.cloud.openfeign.client.config.user-service)
    ↑ 覆盖
default YAML 配置 (spring.cloud.openfeign.client.config.default)
    ↑ 覆盖
Spring Boot 自动配置默认值  (最低)
```

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
