
# 1. 环境准备

| 组件 | 版本要求 |
|------|---------|
| JDK | 17+（推荐 21） |
| Spring Boot | 3.2+ |
| Spring Cloud | 2023.0.x |
| Nacos Server | 2.3+（或其他注册中心） |

# 2. 依赖引入

## 2.1 RestTemplate + LoadBalancer

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2023.0.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring Cloud LoadBalancer（核心） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>

    <!-- Nacos 服务发现 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

## 2.2 WebClient + LoadBalancer（响应式）

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- WebFlux（包含 WebClient） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
</dependencies>
```

## 2.3 Feign + LoadBalancer

```xml
<dependencies>
    <!-- Feign 自动引入 spring-cloud-starter-loadbalancer -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

# 3. 配置文件

## 3.1 服务提供者（user-service）

```yaml
server:
  port: 8081

spring:
  application:
    name: user-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev
        metadata:
          version: v1
          weight: 10
```

## 3.2 服务消费者（order-service）

```yaml
server:
  port: 8080

spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev

    loadbalancer:
      enabled: true                    # 启用负载均衡
      discovery:
        enabled: true                  # 启用服务发现集成
      cache:
        enabled: true
        ttl: 35s                       # 实例列表缓存 TTL
        capacity: 256                  # 最大缓存条目
      retry:
        enabled: false                 # 重试（需引入 spring-retry）
      zone: zone1                      # 区域
```

# 4. 第一个负载均衡示例

## 4.1 RestTemplate 方式

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced   // ← 关键注解，标记该 RestTemplate 需要负载均衡
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id) {
        // 使用服务名替代 IP:Port
        String url = "http://user-service/api/users/" + id;
        return restTemplate.getForObject(url, String.class);
    }
}
```

## 4.2 WebClient 方式（响应式）

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced   // ← 标记该 WebClient.Builder 需要负载均衡
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private WebClient.Builder webClientBuilder;

    @GetMapping("/user/{id}")
    public Mono<String> getUser(@PathVariable Long id) {
        return webClientBuilder.build()
                .get()
                .uri("http://user-service/api/users/" + id)
                .retrieve()
                .bodyToMono(String.class);
    }
}
```

## 4.3 Feign 方式

```java
@FeignClient(name = "user-service")   // 服务名，自动通过 LoadBalancer 解析
public interface UserClient {

    @GetMapping("/api/users/{id}")
    String getUser(@PathVariable("id") Long id);
}

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private UserClient userClient;

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id) {
        return userClient.getUser(id);
    }
}
```

## 4.4 手动编程方式

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id) {
        // 手动选择实例
        ServiceInstance instance = loadBalancerClient.choose("user-service");
        if (instance == null) {
            throw new IllegalStateException("No available instance");
        }

        String url = instance.getUri() + "/api/users/" + id;
        return new RestTemplate().getForObject(url, String.class);
    }
}
```

# 5. 启动与验证

## 5.1 启动服务

```bash
# 1. 启动 Nacos Server
startup.cmd -m standalone

# 2. 启动两个 user-service 实例（不同端口）
java -jar user-service.jar --server.port=8081
java -jar user-service.jar --server.port=8082

# 3. 启动 order-service
java -jar order-service.jar --server.port=8080
```

## 5.2 验证负载均衡

```bash
# 多次调用，观察 user-service 是否轮询
curl http://localhost:8080/api/orders/user/1
curl http://localhost:8080/api/orders/user/1
curl http://localhost:8080/api/orders/user/1
```

在 user-service 的两个实例控制台中，可以看到请求被轮询分发。

# 6. 配置自定义负载均衡策略

```java
@Configuration
public class CustomLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {

        String name = environment.getProperty(
                LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(
                        name, ServiceInstanceListSupplier.class),
                name);
    }
}

// 为特定服务应用该策略
@Configuration
@LoadBalancerClient(
    name = "user-service",
    configuration = CustomLoadBalancerConfig.class
)
public class UserServiceLoadBalancerConfig {
}
```

# 7. @LoadBalanced 原理速查

`@LoadBalanced` 是一个标记注解，配合 `LoadBalancerAutoConfiguration` 实现拦截：

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {
}
```

自动配置会收集所有标注了 `@LoadBalanced` 的 `RestTemplate`/`WebClient.Builder`，然后：

1. **RestTemplate**：通过 `LoadBalancerInterceptor` 拦截 `InterceptingClientHttpRequest`
2. **WebClient**：通过 `ReactorLoadBalancerExchangeFilterFunction` 过滤请求

# 8. 常见问题速查

| 问题 | 原因 | 解决 |
|------|------|------|
| 启动报错 No instances available | 注册中心无可用实例 | 确保 provider 注册成功 |
| 负载均衡不生效 | 未加 `@LoadBalanced` | 给 RestTemplate/WebClient.Builder 加上 |
| 仍然路由到下线实例 | 缓存未失效 | 缩短 `spring.cloud.loadbalancer.cache.ttl` |
| Feign 不生效 | 未配置 url 或 serviceId | `@FeignClient(name="service-name")` |
| WebClient 不生效 | 未使用 Builder | 注入 `WebClient.Builder` 并带 `@LoadBalanced` |
