# 1. 概述

## 1.1 需要手动构建的场景

在 Spring Cloud 环境下，FeignClient 的创建通常由 `@EnableFeignClients` + `@FeignClient` 自动完成。但在以下场景需要**手动使用 `Feign.builder()` 构建**：

- **不使用 Spring 容器**（纯 Java 应用）
- **动态创建客户端**（运行时决定 target URL）
- **非标准配置**（需要多个不同配置的 Client 实例）
- **特殊 Encoder/Decoder**（如自定义 Protobuf 编码）
- **单元测试**（Mock 或直连测试）

## 1.2 一条决策线

`@FeignClient` 的能力边界是"编译期确定的静态契约 + 容器装配的组件"。builder 路线换来的是运行期自由度，代价是**自动配置给的三样东西全部要手动还回去**：SpringMvcContract（§4.1 的坑）、LoadBalancer/服务发现集成（§9）、熔断与可观测（§8 对比表最后一列）。判据很简单：需要"每次运行才知道目标"或"完全脱离 Spring"才用 builder；只是配置不同，优先走 `configuration` 指定类（`4_configuration.md`）。

# 2. Feign.Builder API

## 2.1 组件与默认值

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

这张默认值清单解释了一个经典困惑：**原生 OpenFeign 默认重试 5 次，Spring Cloud 里默认不重试**——`FeignClientsConfiguration` 把 Retryer 覆盖成了 `NEVER_RETRY`（`9_retry.md` §1.1）。同理 `Contract.Default`/`Encoder.Default`（toString 级）与 Spring 环境的 `SpringMvcContract`/`SpringEncoder` 差异都在这一层发生：手动构建时"行为突然变原始"，根源是 builder 的出厂默认，而不是配置丢了。

## 2.2 构建产物的粒度

`target()` 之前所有组件都只是"计划"，`target()` 触发 Contract 解析、代理生成——所以**同一个 builder 配置重复 target 同一接口会产出两个独立代理实例**（各自一份 MethodMetadata）。动态创建场景（§5）应缓存构建结果，把 builder 当"配置模板"而非"每请求工厂"；每请求新建代理会退化成持续解析注解的 CPU 消耗点。

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

示例同时换掉了编解码与 Logger 两处默认，这就是纯 Java 集成 Jackson 的最小闭环；`new Slf4jLogger(UserClient.class)` 的传参决定了 SLF4J logger 名归属（`10_logging.md` §5.1：不传参时挂在 `feign.Logger`，配 DEBUG 的位置也随之不同）。

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

## 4.1 最容易漏的三个组件

这条路本质是"在 Spring 里借用容器资源，但绕开 Feign 自动配置"，漏配都发生在静默处：

- **Contract**（本例已配）：不写 `contract(new SpringMvcContract())`，接口上的 `@GetMapping` 会被 `Contract.Default` 无视，启动即抛 `Method ... not annotated with HTTP method`——报错直白，但新人常误以为注解没生效去查 Spring MVC；
- **Client**：默认 `Client.Default` 意味着没有连接池（`11_timeout-connection.md` §5.3），生产手动构建至少要换成配了池的 Apache/OkHttp；
- **熔断/降级**：`fallback`/`fallbackFactory` 属性对这条路线**完全无效**（那是 `FeignClientFactoryBean` 包装 `Capability`/装饰器的产物），想要熔断需手动套 Resilience4j 装饰器，一般到这一步说明更应该回到 `@FeignClient`。

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

多租户"每客户独立域名"是典型需求。注意 §2.2 的缓存纪律在这里升级为性能关键：按 URL 维度 `ConcurrentHashMap.computeIfAbsent` 缓存代理实例，而不是每次 create——Contract 解析按"代理实例 × 方法数"付费，租户数 × 调用数会让它变成热点。URL 变更（客户换域名）则是反向问题：缓存要能失效重建，给 create 加一个版本号键即可。

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
        String currentUrl = urlSupplier.get();
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

`apply` 每请求执行一次，是"动态性"的全部来源；`name()` 则参与日志 configKey 前缀与拦截器可读上下文，固定成服务名即可。这个模式等于手写了一个迷你负载均衡（注释里从 `EmptyTarget` 换成 `DynamicTarget` 的 `url()` 也每次调用 `url()`，机制相同），但只有"选址"没有"治理"——实例健康剔除、重试换实例、区域路由这些 LoadBalancer 都替你做了，能用 §9 就不要手搓 §6；手动 DynamicTarget 更适合"路由规则本身特殊"（按租户分片、按权重灰度到指定域名）的场景。

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

## 7.1 同接口多实例必须解决注入歧义

同一类型两个 Bean，按类型注入直接 `NoUniqueBeanDefinitionException`。三种收口：Bean 方法上 `@Qualifier("userClientV1")` 并在注入点同名标注；一个加 `@Primary`（默认走 v1，v2 显式 Qualifier）；或干脆拆两个接口（`UserClientV1`/`UserClientV2`）——接口即语义，拆法对代码可读性最友好，也是 `@FeignClient` 时代 `contextId` 想解决的同一问题的两种答案。

# 8. Feign.Builder 与 Spring 自动配置对比

| 维度 | @FeignClient 自动配置 | Feign.builder() 手动构建 |
|------|---------------------|-------------------------|
| **易用性** | 高，零代码 | 低，需手动组装 |
| **灵活性** | 中，YAML + `@Configuration` | 高，代码级控制 |
| **Spring 集成** | 自动 LoadBalancer、熔断等 | 需手动集成 |
| **动态性** | 静态定义 | 支持运行时动态创建 |
| **测试性** | 需 `@SpringBootTest` | 可直接 `new` 测试 |
| **可观测性** | 自动集成 Micrometer | 需手动集成 |

表里"灵活性 vs 集成度"的交换比，源码视角的真相是：`@FeignClient` 路线里 `FeignClientFactoryBean` 做的正是 §4 你手写的全部事情，只不过它按子上下文装配（组件可被 yml/@Bean 覆盖）并额外织入 LoadBalancer、熔断装饰器与 Micrometer 能力。**自动配置 = builder + 集成胶水 + 上下文优先级机制**，理解这一句话，两条路线的所有差异都能自行推导。

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

## 9.1 手动补齐自动配置核心一环

这个包装关系就是 `24_source-loadbalancer-integration.md` 的主线：`FeignBlockingLoadBalancerClient` 把"服务名 → 实例选址"做完后，把真实 URL 塞回 Request 交给 delegate（内层 `Client.Default`）执行。两点细节：`EmptyTarget` 的 name 必须与注册中心服务名一致（它决定查谁），url 占位无意义；内层 delegate 换成 Apache/OkHttp 实现才享受连接池——负载均衡只管选址，不管网络。到这里你已经手动复刻了自动配置约九成功能，这也反过来印证 §1.2：**需要 LoadBalancer 就别用 builder**，这段代码的存在意义是理解机制，不是生产范式。

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

## 10.1 这个工厂成立的前提与隐患

`context.getBean(Feign.Builder.class)` 只在你**主容器显式定义了** `feignBuilder` Bean 时成立——自动配置的那个 `prototype` Builder 存在于每个客户端的**子上下文**里，主容器拿不到（§2 的"作用域"机制见 `4_configuration.md` §1.1）。而即便拿到了全局 Builder，它已被注入固定的拦截器列表与编解码配置，再按调用 `target()` 出的客户端共享这些组件，"按 serviceId 差异化配置"会落空。这个模式的正确形态是：工厂接收组件参数、内部自己 `Feign.builder()` 组装并缓存结果（§5），只从容器借**零件**（converters、interceptors Bean），不借**半成品**。

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

lambda 替换 `Client` 是"假 HTTP"的最小方案——真实走完 Contract→Encoder→Decoder，只把网络层掐掉，适合验证序列化与路由拼装。按测试目标选择层次：只测**使用方业务逻辑**时 mock `UserClient` 接口本身（Mockito）更快也更干净；测**契约/编码/错误映射**才需要这种带 builder 的假传输；测**与服务端真实行为一致性**则上 MockWebServer/WireMock 起本地 HTTP 服务，连 URL 都真。三层成本递增、覆盖递增，CR 里值得按此判断测试写在了正确的层。
