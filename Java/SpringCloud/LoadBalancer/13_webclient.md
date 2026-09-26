
# 1. 概述

Spring Cloud LoadBalancer 通过 `ReactorLoadBalancerExchangeFilterFunction` 为响应式 `WebClient` 提供负载均衡能力，天然支持异步非阻塞。

# 2. 配置方式

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class UserService {

    @Autowired
    private WebClient.Builder webClientBuilder;

    public Mono<User> getUser(Long id) {
        return webClientBuilder.build()
                .get()
                .uri("http://user-service/api/users/" + id)
                //  ↑ 使用服务名，LoadBalancer 自动解析
                .retrieve()
                .bodyToMono(User.class);
    }
}
```

# 3. ReactorLoadBalancerExchangeFilterFunction

## 3.1 源码解析

```java
public class ReactorLoadBalancerExchangeFilterFunction
        implements ExchangeFilterFunction {

    private final LoadBalancerClientFactory loadBalancerClientFactory;
    private final LoadBalancerProperties properties;

    @Override
    public Mono<ClientResponse> filter(
            ClientRequest request, ExchangeFunction next) {

        URI originalUrl = request.url();
        String serviceId = originalUrl.getHost();

        // ① 非 lb:// 协议的请求直接透传
        if (serviceId == null) {
            return next.exchange(request);
        }

        // ② 构建 Hint（传递请求元数据）
        Map<String, Object> hints = new HashMap<>();
        hints.put(LoadBalancerConstants.REQUEST_CONTEXT_KEY,
                new RequestDataContext(
                        new RequestData(request), hints));

        // ③ 执行负载均衡选择
        return choose(serviceId, hints)
                .flatMap(response -> {
                    if (!response.hasServer()) {
                        return Mono.error(new IllegalStateException(
                                "No instances available for " + serviceId));
                    }

                    ServiceInstance instance = response.getServer();

                    // ④ 重建 URL
                    URI newUrl = LoadBalancerUriTools.reconstructURI(
                            instance, originalUrl);

                    // ⑤ 创建新请求并继续过滤器链
                    ClientRequest newRequest = ClientRequest.from(request)
                            .url(newUrl)
                            .build();

                    return next.exchange(newRequest);
                });
    }

    private Mono<Response<ServiceInstance>> choose(
            String serviceId, Map<String, Object> hints) {

        // 获取该服务的 ReactorLoadBalancer
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
                loadBalancerClientFactory.getInstance(serviceId,
                        ReactorServiceInstanceLoadBalancer.class);

        if (loadBalancer == null) {
            return Mono.error(new IllegalStateException(
                    "No load balancer for " + serviceId));
        }

        // 异步选择实例
        return loadBalancer.choose(new DefaultRequest<>(hints));
    }
}
```

## 3.2 请求流程图

```
WebClient.get().uri("http://user-service/api/users/1")
    │
    ▼
ExchangeFilterFunction 链
    │
    ├── LoggingFilter
    ├── AuthFilter
    └── ReactorLoadBalancerExchangeFilterFunction  ← 负载均衡
          │
          ├── 提取 serviceId = "user-service"
          │
          ├── Mono<Response<ServiceInstance>> choose()
          │     ├── ServiceInstanceListSupplier.get()
          │     │     └── DiscoveryClient.getInstances("user-service")
          │     └── RoundRobinLoadBalancer.choose()
          │           └── → 192.168.1.11:8080
          │
          ├── rebuild URI → http://192.168.1.11:8080/api/users/1
          │
          └── next.exchange(newRequest)  ← 实际 HTTP 请求
```

# 4. 自定义 ExchangeFilterFunction

```java
// 在 LoadBalancer 之前添加自定义过滤器
@Bean
public ExchangeFilterFunction customHeaderFilter() {
    return (request, next) -> {
        ClientRequest modified = ClientRequest.from(request)
                .header("X-Request-Id", UUID.randomUUID().toString())
                .header("X-Start-Time", String.valueOf(System.currentTimeMillis()))
                .build();
        return next.exchange(modified);
    };
}

@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder(
        ExchangeFilterFunction customHeaderFilter) {
    return WebClient.builder()
            .filter(customHeaderFilter) // 自定义过滤器
            .filter(ExchangeFilterFunctions
                    .basicAuthentication("user", "pass")); // 认证过滤器
}
```

# 5. 超时配置

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            .responseTimeout(Duration.ofSeconds(5));

    return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient));
}
```

```yaml
# 配置方式
spring:
  cloud:
    loadbalancer:
      httpclient:
        connect-timeout: 3000
        response-timeout: 5s
```

# 6. 连接池配置

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    ConnectionProvider provider = ConnectionProvider.builder("lb-pool")
            .maxConnections(500)              // 最大连接数
            .pendingAcquireMaxCount(1000)     // 等待队列最大长度
            .pendingAcquireTimeout(Duration.ofSeconds(45))
            .maxIdleTime(Duration.ofSeconds(30))
            .maxLifeTime(Duration.ofSeconds(60))
            .build();

    HttpClient httpClient = HttpClient.create(provider)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);

    return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient));
}
```

# 7. 错误处理

```java
@Service
public class UserService {

    @Autowired
    private WebClient.Builder webClientBuilder;

    public Mono<User> getUser(Long id) {
        return webClientBuilder.build()
                .get()
                .uri("http://user-service/api/users/{id}", id)
                .retrieve()
                .onStatus(status -> status.is5xxServerError(),
                        response -> Mono.error(new RuntimeException(
                                "User service error: " + response.statusCode())))
                .onStatus(status -> status.value() == 404,
                        response -> Mono.error(new UserNotFoundException(id)))
                .bodyToMono(User.class)
                .timeout(Duration.ofSeconds(5))
                .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                        .filter(throwable -> throwable instanceof ConnectException))
                .onErrorResume(e -> {
                    log.error("Failed to get user: {}", id, e);
                    return Mono.just(User.FALLBACK);
                });
    }
}
```

# 8. 无阻塞的负载均衡调用

```java
// WebClient + LoadBalancer 全程异步非阻塞
// 与传统 RestTemplate 阻塞模型对比：

// 阻塞（线程等待）：
ServiceInstance instance = loadBalancerClient.choose(...); // 阻塞
String result = restTemplate.getForObject(...);             // 阻塞

// 非阻塞（事件驱动）：
Mono<Response<ServiceInstance>> choice = loadBalancer.choose(...); // 不阻塞
Mono<String> result = webClient.get().uri(...).retrieve()...;      // 不阻塞

// 两者协作：非阻塞链
return loadBalancer.choose(request)
        .flatMap(instance -> webClient.get()
                .uri(instance.getUri())
                .retrieve()
                .bodyToMono(String.class));
```

# 9. 与 Gateway 的对比

| 特性 | WebClient + LB | Gateway + LB |
|------|---------------|-------------|
| **角色** | 服务间调用 | 入口网关 |
| **协议** | `http://service-id` | `lb://service-id` |
| **LB 实现** | `ReactorLoadBalancerExchangeFilterFunction` | `ReactiveLoadBalancerClientFilter` |
| **响应式** | ✓ | ✓ |
| **服务发现** | 通过 Supplier 链 | 通过 Supplier 链 |

# 10. 最佳实践

```java
@Configuration
public class ProductionWebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        ConnectionProvider provider = ConnectionProvider.builder("lb-pool")
                .maxConnections(500)
                .maxIdleTime(Duration.ofSeconds(30))
                .build();

        HttpClient httpClient = HttpClient.create(provider)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                .responseTimeout(Duration.ofSeconds(5))
                .doOnConnected(conn ->
                        conn.addHandlerLast(new ReadTimeoutHandler(5, TimeUnit.SECONDS))
                            .addHandlerLast(new WriteTimeoutHandler(5, TimeUnit.SECONDS)));

        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .filter(ExchangeFilterFunctions
                        .basicAuthentication("service", "secret"))
                .defaultHeader("X-Source", "order-service");
    }
}
```
