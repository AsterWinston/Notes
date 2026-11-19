# 1. 依赖引入

## 1.1 Maven 与 Gradle

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

## 1.2 版本对齐用 BOM，不要手写版本号

三个 starter 都不写 `<version>`，版本统一由 Spring Cloud BOM 管理：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.x</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

OpenFeign 与 Spring Boot 的版本是**强配对**关系（2022.x 起要求 Boot 3），手工混搭是新手期最常见的启动失败来源：症状要么是 `ClassNotFoundException: jakarta.*`（Boot3 配了旧线），要么是自动配置类找不到（新 starter 配了旧 BOM）。

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

## 2.1 @EnableFeignClients 的两个关键作用

`@EnableFeignClients` 有两个关键作用：
1. 启用 `@FeignClient` 注解扫描
2. 为每个 `@FeignClient` 接口创建 JDK 动态代理并注册到 Spring 容器

## 2.2 注册的本质：FeignClientFactoryBean

准确说，容器里注册的并不是代理对象本身，而是每个接口对应的 `FeignClientFactoryBean`（携带 name/contextId/configuration 等注解元信息）。Bean 被真正获取时才由工厂调用 Feign 的 `Feign.Builder` 构建代理——这解释了为什么"代理构建发生在应用启动之后、第一次使用时"，也是 `21_source-dynamic-proxy.md` 的起点。

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

## 3.1 提供方不需要任何配合改动

服务提供方是普通 `@RestController`，它不知道 Feign 的存在。契约的"一致性"完全靠调用方接口与提供方路由手工对齐——这也是 Feign 与 gRPC/proto 方案的本质区别：**Feign 没有编译期的跨服务契约校验**，提供方改了路径，调用方要到运行时才 404。团队规模大时建议在提供方导出 OpenAPI、调用方代码生成客户端接口，或至少用契约测试兜底（参见 `25_best-practices.md`）。

## 3.2 URL 的组合规则

最终请求地址 = 服务实例基地址 + 类级 `path` + 方法级映射路径。`path = "/api/users"` 是接口级的公共前缀，方法上只写差异部分；它等价于每个方法都带上该前缀，但不必重复书写。构造器注入（而非字段注入 `@Autowired`）让 `UserClient` 在单测里可以直接替换成手写的桩实现——声明式接口的另一项隐性收益就是**天然可 mock**。

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

注意 `<name>` 处默认填 `@FeignClient` 的 **contextId（缺省时即 name）**，而不是接口全限定名；多个客户端指向同一个服务时只有 contextId 能区分配置。配置项全解见 `4_configuration.md`，两个超时的真实作用点在 `11_timeout-connection.md`。

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

## 5.1 选择哪种扫描方式

多模块工程里 `basePackages` 最常用；`basePackageClasses` 以类锚定包，重构改名不断链；`clients` 显式枚举，适合**测试切片**（`@SpringBootTest` + `@EnableFeignClients(clients = UserClient.class)` 只注册需要的客户端）。若客户端接口放在第三方 jar 且包路径不在扫描范围内，启动日志不会有任何报错，表现为"注入时 NoSuchBeanDefinitionException"——先检查扫描范围再看别的。

# 6. 调用链路全貌

```text
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

对照本系列源码篇，这张图的每一行都有独立深潜：代理分发 → `21_source-dynamic-proxy.md`；Contract/MethodHandler 的构建期与运行期分工 → `22_source-contract-methodhandler.md`；从模板到响应体的管线细节 → `23_source-request-pipeline.md`；LoadBalancer 如何嵌入 Client → `24_source-loadbalancer-integration.md`。

# 7. 完整示例项目结构

```text
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

结构上的一条经验：**把"按服务的配置类"与"全局配置类"分开放**，并保证按服务的配置类不在组件扫描路径内（原因见 `4_configuration.md` §5），否则会出现"只想给 user-service 加鉴权头，结果全服务生效"的作用域事故。

# 8. 注意事项

1. **`@PathVariable` 必须指定 value**：`@PathVariable("id")`，不能省略（Spring 4.3+ 在 Controller 中可以省略参数名，但 Feign 接口中必须显式指定）
2. **`@RequestParam` 必须指定 value**：`@RequestParam("name") String name`
3. **`@RequestBody` 只允许一个**：一个方法只能有一个 `@RequestBody` 参数
4. **无参 GET 不能标记 `@RequestBody`**：否则 Feign 会用 POST 发送
5. **复杂对象作为 GET 参数**：使用 `@SpringQueryMap` 将对象展开为 query parameters

前三条的根本原因相同：省略参数名依赖字节码保留方法参数名（`-parameters` 编译选项），而 `SpringMvcContract` 解析接口方法时无法保证拿到——Controller 类由 Spring 编译体系处理可以侥幸生效，Feign 接口一旦定义在依赖包里就会静默解析成错误的占位符名，表现为路径里出现 `{id}` 字面量或 404。**显式写 value 是零成本消除此类环境差异的方式**。第 4、5 条则源于 Feign 的编码模型：请求体参数与查询参数是两条互斥的组装通道，GET 场景的参数只能从 URL 侧表达，`@SpringQueryMap` 就是官方的"对象摊平成 query"桥接（嵌套对象默认不递归展开，深结构需要自定义 `Contract` 或手工拆参）。

# 9. 首跑排错清单

第一次接入时问题几乎都落在以下五类，按顺序排查最快：

| 症状 | 高概率原因 |
|------|------------|
| `UnknownHostException: user-service` | 未引入 LoadBalancer 依赖，服务名没有解析成实例地址 |
| `No servers available` / 503 性质错误 | 注册中心里没有 healthy 实例，或服务名大小写/环境不一致 |
| 注入时 `NoSuchBeanDefinitionException` | 接口不在 `@EnableFeignClients` 扫描范围（§5.1） |
| 对方收到 404，本地路径肉眼一致 | 类级 `path` 重复前缀，或提供方 `@RequestMapping` 拼法不同（§3.2） |
| `RetryableException` 伴随 read-timeout | 下游慢，默认读超时 60s 仍被触发，或连接被中间设备静默断开（§4、`11_timeout-connection.md`） |
