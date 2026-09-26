
# 1. 概述

在 Spring Cloud 环境下，FeignClient 的创建通常由 `@EnableFeignClients` + `@FeignClient` 自动完成。但在以下场景需要**手动使用 `Feign.builder()` 构建**：

- **不使用 Spring 容器**（纯 Java 应用）
- **动态创建客户端**（运行时决定 target URL）
- **非标准配置**（需要多个不同配置的 Client 实例）
- **特殊 Encoder/Decoder**（如自定义 Protobuf 编码）
- **单元测试**（Mock 或直连测试）

# 2. Feign.Builder API

```java
public abstract class Feign {

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private Contract contract = new Contract.Default();
        private Client client = new Client.Default(null, null);
        private Retryer retryer = new Retryer.Default();
        private Logger logger = new NoOpLogger();
        private Encoder encoder = new Encoder.Default();
        private Decoder decoder = new Decoder.Default();
        private ErrorDecoder errorDecoder = new ErrorDecoder.Default();
        private Request.Options options = new Request.Options();
        private InvocationHandlerFactory invocationHandlerFactory;
        private boolean decode404;
        private boolean dismiss404;
        private List<RequestInterceptor> requestInterceptors;
        private Level logLevel = Level.NONE;
        // ...

        public Builder contract(Contract contract) { ... }
        public Builder client(Client client) { ... }
        public Builder retryer(Retryer retryer) { ... }
        public Builder logger(Logger logger) { ... }
        public Builder encoder(Encoder encoder) { ... }
        public Builder decoder(Decoder decoder) { ... }
        public Builder errorDecoder(ErrorDecoder errorDecoder) { ... }
        public Builder options(Request.Options options) { ... }
        public Builder requestInterceptor(RequestInterceptor interceptor) { ... }
        public Builder requestInterceptors(Iterable<RequestInterceptor> interceptors) { ... }
        public Builder logLevel(Logger.Level logLevel) { ... }
        public Builder decode404() { ... }
        public Builder dismiss404() { ... }
        public Builder invocationHandlerFactory(InvocationHandlerFactory factory) { ... }

        public <T> T target(Class<T> apiType, String url) { ... }
        public <T> T target(Target<T> target) { ... }
    }
}
```

# 3. 基础示例：纯 Java 环境

```java
public class FeignClientExample {

    public static void main(String[] args) {
        UserClient client = Feign.builder()
                .encoder(new JacksonEncoder())
                .decoder(new JacksonDecoder())
                .logger(new Slf4jLogger(UserClient.class))
                .logLevel(Logger.Level.FULL)
                .target(UserClient.class, "http://localhost:8080");

        User user = client.getUser(1L);
        System.out.println(user);
    }
}
```

# 4. Spring 容器中手动构建

```java
@Configuration
public class ManualFeignClientConfig {

    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverters;

    @Bean
    public UserClient userClient() {
        return Feign.builder()
                .encoder(new SpringEncoder(messageConverters))
                .decoder(new SpringDecoder(messageConverters))
                .contract(new SpringMvcContract())
                .requestInterceptor(new AuthInterceptor())
                .retryer(new Retryer.Default(200, 2000, 3))
                .options(new Request.Options(5000, 10000))
                .target(UserClient.class, "http://localhost:8080");
    }
}
```

# 5. 动态 URL Target

```java
public class DynamicUrlClientFactory {

    private final Encoder encoder;
    private final Decoder decoder;

    public UserClient create(String serviceUrl) {
        return Feign.builder()
                .encoder(encoder)
                .decoder(decoder)
                .target(UserClient.class, serviceUrl);
    }
}
```

# 6. 自定义 Target

```java
public class DynamicTarget<T> implements Target<T> {

    private final Class<T> type;
    private final String name;
    private final Supplier<String> urlSupplier;

    public DynamicTarget(Class<T> type, String name, Supplier<String> urlSupplier) {
        this.type = type;
        this.name = name;
        this.urlSupplier = urlSupplier;
    }

    @Override
    public Class<T> type() { return type; }

    @Override
    public String name() { return name; }

    @Override
    public String url() { return urlSupplier.get(); }

    @Override
    public Request apply(RequestTemplate input) {
        // 每次请求都会调用 url()，实现动态路由
        String currentUrl = url();
        input.target(currentUrl);
        return input.request();
    }
}
```

```java
// 从注册中心动态获取 URL
Supplier<String> urlSupplier = () -> {
    Instance instance = nacosDiscovery.getHealthyInstance("user-service");
    return "http://" + instance.getIp() + ":" + instance.getPort();
};

UserClient client = Feign.builder()
        .target(new DynamicTarget<>(UserClient.class, "user-service", urlSupplier));
```

# 7. 构建多配置实例

```java
@Configuration
public class MultiFeignConfig {

    @Bean
    public UserClient userClientV1(ObjectFactory<HttpMessageConverters> converters) {
        return Feign.builder()
                .encoder(new SpringEncoder(converters))
                .decoder(new SpringDecoder(converters))
                .options(new Request.Options(2000, 5000))
                .target(UserClient.class, "http://v1.api.example.com");
    }

    @Bean
    public UserClient userClientV2(ObjectFactory<HttpMessageConverters> converters) {
        return Feign.builder()
                .encoder(new SpringEncoder(converters))
                .decoder(new SpringDecoder(converters))
                .options(new Request.Options(3000, 8000))
                .target(UserClient.class, "http://v2.api.example.com");
    }
}
```

# 8. Feign.Builder 与 Spring 自动配置对比

| 维度 | @FeignClient 自动配置 | Feign.builder() 手动构建 |
|------|---------------------|-------------------------|
| **易用性** | 高，零代码 | 低，需手动组装 |
| **灵活性** | 中，YAML + `@Configuration` | 高，代码级控制 |
| **Spring 集成** | 自动 LoadBalancer、熔断等 | 需手动集成 |
| **动态性** | 静态定义 | 支持运行时动态创建 |
| **测试性** | 需 `@SpringBootTest` | 可直接 `new` 测试 |
| **可观测性** | 自动集成 Micrometer | 需手动集成 |

# 9. 配合 LoadBalancer 的手动构建

```java
@Bean
public UserClient loadBalancedUserClient(
        LoadBalancerClient loadBalancerClient,
        ObjectFactory<HttpMessageConverters> converters) {

    return Feign.builder()
            .encoder(new SpringEncoder(converters))
            .decoder(new SpringDecoder(converters))
            .client(new FeignBlockingLoadBalancerClient(
                    new Client.Default(null, null),
                    loadBalancerClient
            ))
            .target(Target.EmptyTarget.create(UserClient.class, "user-service"));
}
```

# 10. 使用 FeignClientsRegistrar 之外的工厂模式

```java
@Component
public class FeignClientFactory {

    @Autowired
    private ConfigurableApplicationContext context;

    public <T> T createClient(Class<T> clazz, String serviceId) {
        Feign.Builder builder = context.getBean(Feign.Builder.class);
        // 可按需覆盖配置
        return builder
                .options(new Request.Options(3000, 5000))
                .target(clazz, "http://" + serviceId);
    }
}
```

在 `@Configuration` 中暴露 `Feign.Builder`：

```java
@Bean
@Scope("prototype")
public Feign.Builder feignBuilder(/* ... */) {
    return Feign.builder()
            .encoder(new SpringEncoder(messageConverters))
            .decoder(new SpringDecoder(messageConverters))
            .contract(new SpringMvcContract());
}
```

# 11. 单元测试中的应用

```java
@Test
void testUserClient() {
    UserClient client = Feign.builder()
            .client((request, options) -> {
                // 模拟 HTTP 响应
                return Response.builder()
                        .status(200)
                        .headers(Map.of("Content-Type", List.of("application/json")))
                        .body("{\"id\":1,\"name\":\"Test\"}")
                        .request(request)
                        .build();
            })
            .encoder(new JacksonEncoder())
            .decoder(new JacksonDecoder())
            .target(UserClient.class, "http://localhost");

    User user = client.getUser(1L);
    assertEquals("Test", user.getName());
}
```
