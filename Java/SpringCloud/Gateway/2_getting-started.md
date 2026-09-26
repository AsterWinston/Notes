
# 1. 项目搭建

## 1.1 Maven 依赖配置

```xml
<!-- Spring Cloud Gateway 核心依赖 -->
<dependencies>
    <!-- 网关核心（必须） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <!-- 服务发现（按需） -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- Actuator 监控（推荐） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- 熔断降级（按需） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
    </dependency>

    <!-- Redis 限流（按需） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 1.2 Gradle 依赖配置

```groovy
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
    implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-reactor-resilience4j'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.0"
    }
}
```

## 1.3 启动类

```java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

**注意**：不要在启动类上加 `@EnableDiscoveryClient` —— 当 `spring-cloud-starter-alibaba-nacos-discovery` 在 classpath 中时，Spring Boot 会自动配置。

# 2. 第一个 Hello World 路由

## 2.1 目标

将所有 `/hello/**` 的请求转发到 `http://httpbin.org`。

## 2.2 YAML 配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: hello_route
          uri: http://httpbin.org
          predicates:
            - Path=/hello/**
          filters:
            - StripPrefix=1
```

**配置说明**：
- `id`：路由唯一标识，任意字符串
- `uri`：目标服务地址，支持 `http://`、`https://`、`lb://`、`ws://`、`wss://`、`forward://` 协议
- `predicates`：断言列表，请求必须满足所有条件（AND 关系）才能匹配该路由
- `filters`：过滤器列表，对请求/响应进行处理
- `StripPrefix=1`：剥离访问路径的前 1 段。请求 `/hello/get` 转发为 `/get`

## 2.3 验证

```bash
# 启动网关后访问
curl http://localhost:8080/hello/get

# 实际转发到 http://httpbin.org/get
# 返回 httpbin.org 的响应内容
```

# 3. 核心配置项详解

## 3.1 基础配置

```yaml
spring:
  cloud:
    gateway:
      # 路由定义
      routes:
        - id: route_id
          uri: http://example.com
          predicates:
            - Path=/example/**
          filters:
            - AddRequestHeader=X-Custom, Value
          order: 0
          metadata:
            key: value

      # 默认过滤器（对所有路由生效）
      default-filters:
        - AddResponseHeader=X-Gateway, Spring-Cloud-Gateway

      # HandlerMapping 顺序（默认 1）
      handler-mapping:
        order: 1

      # 启用路由去重（默认 true）
      remove-non-specific-handler-mapping: true

      # 转发请求头的 X-Forwarded-* 处理
      x-forwarded:
        enabled: true           # 是否启用转发头处理
        proto-enabled: true     # 是否添加 X-Forwarded-Proto
        host-enabled: true      # 是否添加 X-Forwarded-Host
        port-enabled: true      # 是否添加 X-Forwarded-Port
        for-enabled: true       # 是否添加 X-Forwarded-For
        for-append: true        # 是否追加（而非替换）

      # 发现服务自动路由（需配合服务发现组件）
      discovery:
        locator:
          enabled: false
          lower-case-service-id: true
          filters:
            - StripPrefix=1
```

## 3.2 HTTP 客户端配置

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 5000           # 连接超时（ms）
        response-timeout: 10s           # 响应超时（Duration）
        max-header-size: 16384          # 最大 Header 大小
        max-initial-line-length: 4096   # 初始行最大长度
        wiretap: false                  # 启用 Netty 级别的日志
        compression: false              # 启用请求压缩

        # SSL 配置
        ssl:
          handshake-timeout-millis: 10000
          close-notify-flush-timeout-millis: 3000
          close-notify-read-timeout-millis: 0
          use-insecure-trust-manager: false   # 是否信任所有证书（测试用）

        # 连接池配置
        pool:
          type: fixed                    # elastic（弹性，默认）/ fixed（固定）
          name: proxy
          max-connections: 500           # 仅 fixed 类型
          acquire-timeout: 45000
          max-idle-time: 3000
          max-life-time: 60000
          eviction-interval: 0           # 0 表示不启用定期清理

        # WebSocket 配置
        websocket:
          max-frame-payload-length: 65536
          proxy-ping: true
```

## 3.3 HTTPS 服务端配置

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:gateway.jks
    key-store-password: changeit
    key-alias: gateway
    key-store-type: JKS

spring:
  cloud:
    gateway:
      httpclient:
        ssl:
          use-insecure-trust-manager: false
          trusted-x509-certificates:
            - classpath:ca.crt
```

## 3.4 路由元数据（Metadata）配置

元数据可用于传递附加信息，在自定义过滤器或谓词中使用：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: metadata_route
          uri: http://example.com
          predicates:
            - Path=/metadata/**
          metadata:
            response-timeout: 200
            connect-timeout: 200
            custom-key: custom-value
```

在自定义过滤器中读取元数据：

```java
public class MetadataReadingFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route != null) {
            Object timeout = route.getMetadata().get("response-timeout");
            if (timeout instanceof Number) {
                // 使用自定义超时
            }
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1; // 优先执行
    }
}
```

# 4. Actuator 监控接入

## 4.1 配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: gateway,health,info
  endpoint:
    gateway:
      enabled: true
```

## 4.2 可用端点

| 端点 | 方法 | 描述 |
|------|------|------|
| `/actuator/gateway/routes` | GET | 获取所有已定义的路由 |
| `/actuator/gateway/routes/{id}` | GET | 获取指定路由的详情 |
| `/actuator/gateway/routes/{id}` | POST | 新增/更新路由定义 |
| `/actuator/gateway/routes/{id}` | DELETE | 删除路由定义 |
| `/actuator/gateway/refresh` | POST | 刷新路由缓存 |
| `/actuator/gateway/globalfilters` | GET | 列出所有全局过滤器 |
| `/actuator/gateway/routefilters` | GET | 列出所有过滤器工厂 |
| `/actuator/gateway/routedefinitions` | GET | 获取路由定义（非展开的 Route） |

## 4.3 常用操作

```bash
# 查看所有路由
curl http://localhost:8080/actuator/gateway/routes

# 查看特定路由详情
curl http://localhost:8080/actuator/gateway/routes/hello_route

# 新增路由
curl -X POST http://localhost:8080/actuator/gateway/routes/new_route \
  -H "Content-Type: application/json" \
  -d '{
    "predicates": [{"name": "Path", "args": {"pattern": "/new/**"}}],
    "filters": [],
    "uri": "http://localhost:8081",
    "order": 0
  }'

# 刷新路由
curl -X POST http://localhost:8080/actuator/gateway/refresh

# 删除路由
curl -X DELETE http://localhost:8080/actuator/gateway/routes/hello_route
```

# 5. 日志配置

```yaml
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
    org.springframework.web.reactive: DEBUG
    reactor.netty: DEBUG
    # 限流相关日志
    org.springframework.cloud.gateway.filter.ratelimit: DEBUG
```

# 6. 常见问题与最佳实践

## 6.1 排除不需要的自动配置

Spring Cloud Gateway 基于 Spring WebFlux，需要排除 Spring MVC：

```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
      - org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration
```

或者在启动类上显式排除：

```java
@SpringBootApplication(exclude = {
    WebMvcAutoConfiguration.class,
    SecurityFilterAutoConfiguration.class
})
```

## 6.2 端口与上下文路径

```yaml
server:
  port: 8080
  # 网关不建议设置 context-path，否则所有路由都需要处理前缀
  # servlet:
  #   context-path: /gateway
```

## 6.3 编码配置

```yaml
spring:
  http:
    encoding:
      charset: UTF-8
      enabled: true
      force: true
```

## 6.4 验证启动成功

```bash
# 启动后检查
curl http://localhost:8080/actuator/health

# 返回 {"status": "UP"}
```

## 6.5 路由优先级

当多个路由可以匹配同一请求时，`order` 值越小优先级越高：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: specific_route
          uri: http://specific-service
          predicates:
            - Path=/api/v2/**
          order: 0   # 高优先级

        - id: general_route
          uri: http://general-service
          predicates:
            - Path=/api/**
          order: 1   # 低优先级（当 v2 不匹配时生效）
```

**注意**：未设置 `order` 时默认为 0，同 order 的路由按定义顺序匹配。
