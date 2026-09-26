
# 1. 依赖引入

**Maven**：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**Gradle**：

```gradle
implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
implementation 'org.springframework.boot:spring-boot-starter-web'
```

# 2. 启用 Feign

```java
@SpringBootApplication
@EnableFeignClients
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```

`@EnableFeignClients` 有两个关键作用：
1. 启用 `@FeignClient` 注解扫描
2. 为每个 `@FeignClient` 接口创建 JDK 动态代理并注册到 Spring 容器

# 3. 定义第一个 Feign 客户端

**服务提供方（user-service）**：

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

**服务调用方（order-service）**：

```java
@FeignClient(name = "user-service", path = "/api/users")
public interface UserClient {

    @GetMapping("/{id}")
    User getUser(@PathVariable("id") Long id);

    @PostMapping
    User createUser(@RequestBody User user);
}
```

```java
@Service
public class OrderService {

    private final UserClient userClient;

    public OrderService(UserClient userClient) {
        this.userClient = userClient;
    }

    public OrderDto createOrder(OrderRequest request) {
        User user = userClient.getUser(request.getUserId());
        return buildOrder(user, request);
    }
}
```

# 4. 配置速览

**application.yml**：

```yaml
spring:
  application:
    name: order-service
  cloud:
    openfeign:
      client:
        config:
          user-service:
            connect-timeout: 5000
            read-timeout: 5000
            logger-level: FULL

logging:
  level:
    com.example.order.feign.UserClient: DEBUG
```

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `spring.cloud.openfeign.client.config.<name>.connect-timeout` | 连接超时 (ms) | 10000 |
| `spring.cloud.openfeign.client.config.<name>.read-timeout` | 读取超时 (ms) | 60000 |
| `spring.cloud.openfeign.client.config.<name>.logger-level` | 日志级别 | NONE |
| `spring.cloud.openfeign.client.config.<name>.error-decoder` | 自定义 ErrorDecoder | ErrorDecoder.Default |
| `spring.cloud.openfeign.client.config.<name>.retryer` | 重试器 | Retryer.NEVER_RETRY |

# 5. 扫描路径配置

默认扫描 `@SpringBootApplication` 所在包及其子包。可通过以下方式指定：

```java
// 指定扫描包
@EnableFeignClients(basePackages = "com.example.feign")

// 指定扫描类
@EnableFeignClients(basePackageClasses = {UserClient.class, OrderClient.class})

// 指定扫描的客户端接口类
@EnableFeignClients(clients = {UserClient.class})
```

# 6. 调用链路全貌

```
OrderService
  │
  └─→ userClient.getUser(1L)
        │
        ▼
    JDK Proxy (FeignInvocationHandler)
        │
        ▼
    MethodHandler.invoke(args)
        │
        ├── RequestInterceptor.apply()      (添加请求头/鉴权)
        ├── Contract → RequestTemplate      (URL/参数/Header 组装)
        ├── Encoder.encode()                (请求体序列化)
        ├── Client.execute()                (执行 HTTP 调用)
        │       └── LoadBalancer → 服务实例选择
        │               └── Apache HttpClient / OkHttp
        ├── Decoder.decode()                (响应体反序列化)
        └── ErrorDecoder.decode()           (错误时解码异常)
        │
        ▼
    返回 User 对象
```

# 7. 完整示例项目结构

```
order-service/
├── pom.xml
├── src/main/java/com/example/order/
│   ├── OrderApplication.java              ← @EnableFeignClients
│   ├── config/
│   │   └── FeignConfig.java               ← 全局 Feign 配置
│   ├── feign/
│   │   ├── UserClient.java                ← @FeignClient 接口
│   │   └── UserClientFallback.java        ← 降级处理
│   ├── service/
│   │   └── OrderService.java              ← 业务服务
│   └── model/
│       ├── User.java
│       └── Order.java
└── src/main/resources/
    └── application.yml
```

# 8. 注意事项

1. **`@PathVariable` 必须指定 value**：`@PathVariable("id")`，不能省略（Spring 4.3+ 在 Controller 中可以省略参数名，但 Feign 接口中必须显式指定）
2. **`@RequestParam` 必须指定 value**：`@RequestParam("name") String name`
3. **`@RequestBody` 只允许一个**：一个方法只能有一个 `@RequestBody` 参数
4. **无参 GET 不能标记 `@RequestBody`**：否则 Feign 会用 POST 发送
5. **复杂对象作为 GET 参数**：使用 `@SpringQueryMap` 将对象展开为 query parameters
