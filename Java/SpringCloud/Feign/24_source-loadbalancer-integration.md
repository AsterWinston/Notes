
# 1. 集成架构

```
Feign.Builder
    │
    ├── client(FeignBlockingLoadBalancerClient)  ← 装饰底层 HTTP 引擎
    │       ├── delegate: Client (ApacheHttpClient / OkHttpClient / Default)
    │       └── loadBalancerClient: LoadBalancerClient
    │
    └── target(Target)
            └── HardCodedTarget / EmptyTarget
```

Spring Cloud 的自动配置将 `FeignBlockingLoadBalancerClient` 作为 `Client` 注入到 `Feign.Builder` 中。

# 2. FeignBlockingLoadBalancerClient 源码

```java
public class FeignBlockingLoadBalancerClient implements Client {

    private final Client delegate;
    private final LoadBalancerClient loadBalancerClient;

    public FeignBlockingLoadBalancerClient(Client delegate,
                                            LoadBalancerClient loadBalancerClient) {
        this.delegate = delegate;
        this.loadBalancerClient = loadBalancerClient;
    }

    @Override
    public Response execute(Request request, Request.Options options)
            throws IOException {
        // 1. 解析原始 URL 获取 serviceId
        URI originalUri = URI.create(request.url());
        String serviceId = originalUri.getHost();
        if (serviceId == null) {
            throw new IllegalStateException("Request URL not contain a valid hostname: "
                    + request.url());
        }

        // 2. 通过 LoadBalancerClient 选择一个服务实例
        ServiceInstance instance = loadBalancerClient.choose(serviceId);
        if (instance == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }

        // 3. 构造新的 URL（将 serviceId 替换为实际的 host:port）
        URI reconstructedUri = reconstructURI(instance, originalUri);
        Request newRequest = buildRequest(request, reconstructedUri.toString());

        // 4. 委托给底层 HTTP 客户端执行
        return delegate.execute(newRequest, options);
    }

    private URI reconstructURI(ServiceInstance instance, URI original) {
        // 从原始 URL 中提取 path + query
        String path = original.getRawPath();
        String query = original.getRawQuery();

        // 使用实例的 host 和 port 重新构造 URI
        return URI.create(instance.getUri().toString()
                + (path != null ? path : "")
                + (query != null ? "?" + query : ""));
    }

    private Request buildRequest(Request request, String url) {
        // 创建新的 Request，替换 URL
        return Request.create(
                request.httpMethod(),
                url,
                request.headers(),
                request.body(),
                request.charset(),
                request.requestTemplate());
    }
}
```

# 3. LoadBalancerClient.choose() 内部流程

```java
// Spring Cloud LoadBalancer 实现：
public class BlockingLoadBalancerClient implements LoadBalancerClient {

    private final LoadBalancerClientFactory loadBalancerClientFactory;

    @Override
    public ServiceInstance choose(String serviceId) {
        return choose(serviceId, Request.DEFAULT);
    }

    @Override
    public ServiceInstance choose(String serviceId, Request<RequestDataContext> request) {
        // 1. 从 LoadBalancerClientFactory 获取对应 serviceId 的 ReactorLoadBalancer
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
                loadBalancerClientFactory.getInstance(serviceId);

        if (loadBalancer == null) {
            return null;
        }

        // 2. 响应式调用 → 阻塞等待结果
        Response<ServiceInstance> response = loadBalancer.choose(request).block();

        if (response == null) {
            return null;
        }
        if (response.hasServer()) {
            return response.getServer();
        }
        return null;
    }
}
```

# 4. LoadBalancerClientFactory 配置隔离

```java
public class LoadBalancerClientFactory extends NamedContextFactory<LoadBalancerClientSpecification> {

    public LoadBalancerClientFactory() {
        super(LoadBalancerClientSpecification.class, "loadbalancer", "loadbalancer.client.name");
    }

    // 根据 serviceId 获取专属的 ReactorLoadBalancer
    public ReactorLoadBalancer<ServiceInstance> getInstance(String serviceId) {
        return getContext(serviceId).getInstance(serviceId, ReactorLoadBalancer.class);
    }

    // 根据 serviceId 获取专属的 ServiceInstanceListSupplier
    public ServiceInstanceListSupplier getInstance(String serviceId,
                                                    Class<ServiceInstanceListSupplier> type) {
        return getContext(serviceId).getBean(type);
    }
}
```

**NamedContextFactory** 原理：为每个 serviceId 创建独立的 Spring ApplicationContext，实现配置隔离。

```
LoadBalancerClientFactory
  ├── user-service → ApplicationContext (user-service)
  │     ├── RoundRobinLoadBalancer
  │     └── DiscoveryClientServiceInstanceListSupplier
  │
  └── order-service → ApplicationContext (order-service)
        ├── RandomLoadBalancer
        └── DiscoveryClientServiceInstanceListSupplier
```

# 5. ServiceInstanceListSupplier 链

```java
// 默认 Supplier 链构建（LoadBalancerClientConfiguration）：
@Configuration
public class LoadBalancerClientConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()              // 1. 从 DiscoveryClient 获取实例
                .withCaching()                      // 2. 缓存实例列表
                .build(context);
    }

    // Zone 感知 Supplier（如果有配置）
    @Bean
    @ConditionalOnBean(ReactiveDiscoveryClient.class)
    @ConditionalOnMissingBean
    public ServiceInstanceListSupplier zonePreferenceServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withZonePreference()
                .withCaching()
                .build(context);
    }

    // 默认负载均衡算法
    @Bean
    @ConditionalOnMissingBean
    public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(
                LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(name,
                        ServiceInstanceListSupplier.class),
                name);
    }
}
```

# 6. RoundRobinLoadBalancer 源码

```java
public class RoundRobinLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    private final AtomicInteger position;
    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
    private final String serviceId;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
                .getIfAvailable(NoopServiceInstanceListSupplier::new);

        return supplier.get(request)
                .next()                              // 获取第一波实例列表
                .map(instances -> {
                    if (instances.isEmpty()) {
                        return new EmptyResponse();
                    }
                    // 轮询选一个（忽略权重）
                    int pos = Math.abs(position.incrementAndGet());
                    ServiceInstance instance = instances.get(pos % instances.size());
                    return new DefaultResponse(instance);
                });
    }
}
```

# 7. CachingServiceInstanceListSupplier 缓存机制

```java
public class CachingServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final Flux<List<ServiceInstance>> serviceInstances;

    public CachingServiceInstanceListSupplier(
            ServiceInstanceListSupplier delegate,
            LoadBalancerCacheManager cacheManager) {
        super(delegate);

        // 当 delegate 推送新列表时，更新缓存
        this.serviceInstances = delegate.get()
                .cache(cacheManager.getCache(serviceId), ttl);   // 默认 35s
    }

    @Override
    public Flux<List<ServiceInstance>> get() {
        return serviceInstances;
    }
}
```

**缓存失效时机**：
1. TTL 到期（默认 35s）→ 自动失效，下一次请求触发刷新
2. 注册中心事件推送（如 Nacos 服务变更通知）→ 由 `DiscoveryClientServiceInstanceListSupplier` 主动推送

# 8. DiscoveryClientServiceInstanceListSupplier

```java
public class DiscoveryClientServiceInstanceListSupplier
        implements ServiceInstanceListSupplier {

    private final DiscoveryClient discoveryClient;
    private final String serviceId;

    @Override
    public Flux<List<ServiceInstance>> get(Request request) {
        return Flux.defer(() -> {
            // 通过 DiscoveryClient 获取实例列表
            List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
            if (instances.isEmpty()) {
                return Flux.empty();
            }
            return Flux.just(instances);
        });
    }
}
```

**与注册中心交互**：
- Nacos：`NacosDiscoveryClient.getInstances()` → 调用 Nacos OpenAPI `/nacos/v1/ns/instance/list`
- Eureka：`EurekaDiscoveryClient.getInstances()` → 从本地缓存的 Applications 获取
- Consul：`ConsulDiscoveryClient.getInstances()` → 调用 Consul HTTP API

# 9. 请求级别的 Hint 传递

LoadBalancer 支持通过 `Request` 的 context 传递**负载均衡提示（Hint）**：

```java
// FeignBlockingLoadBalancerClient 中可以构建带 Hint 的 Request：
RequestDataContext context = new RequestDataContext(
        new RequestData(
                httpMethod,
                originalUri,
                headers,
                // hint: 如灰度标记
                Map.of("hint", hintKey)
        )
);

Request<RequestDataContext> lbRequest = new DefaultRequest<>(context);
ServiceInstance instance = loadBalancerClient.choose(serviceId, lbRequest);
```

在自定义 LoadBalancer 中读取 Hint：

```java
public class HintBasedLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        RequestDataContext ctx = (RequestDataContext) request.getContext();
        Map<String, Object> hints = ctx.getClientRequest().getAttributes();
        String version = (String) hints.get("version");
        // 根据 version hint 过滤实例...
    }
}
```

# 10. 重试与负载均衡的交互

```java
// 当一次调用失败后，重试的逻辑：
// SynchronousMethodHandler (while-true 循环中)
//   ├── 第 1 次: client.execute() → FeignBlockingLoadBalancerClient
//   │     └── choose(serviceId) → 选中实例 A
//   │     └── delegate.execute(A, ...) → IOException (连接超时)
//   │     └── 抛出 RetryableException
//   ├── 重试: retryer.continueOrPropagate(e)
//   ├── 第 2 次: client.execute()
//   │     └── choose(serviceId) → 选中实例 B  ← 负载均衡会选择不同实例！
//   │     └── delegate.execute(B, ...) → 成功
//   └── 返回结果
```

**关键**：每次重试都是一次**全新的** `FeignBlockingLoadBalancerClient.execute()` 调用，因此会**重新执行 LoadBalancer.choose()**，自动规避故障实例。

# 11. 完整时序图

```
Feign 调用
  │
  ▼
FeignBlockingLoadBalancerClient.execute(request, options)
  │
  ├── URI originalUri = request.url()  → "http://user-service/api/users/1"
  │
  ├── String serviceId = "user-service"
  │
  ├── LoadBalancerClient.choose("user-service")
  │     │
  │     ├── ReactorLoadBalancer.choose(request)
  │     │     │
  │     │     ├── ServiceInstanceListSupplier.get(request)
  │     │     │     ├── CachingServiceInstanceListSupplier (缓存命中? 直接返回)
  │     │     │     │   └── 缓存未命中 → DiscoveryClientServiceInstanceListSupplier
  │     │     │     │         └── DiscoveryClient.getInstances("user-service")
  │     │     │     │               └── [192.168.1.10:8080, 192.168.1.11:8080]
  │     │     │     └── 返回 List<ServiceInstance>
  │     │     │
  │     │     └── RoundRobinLoadBalancer.getServiceInstance()
  │     │           └── position.incrementAndGet() % instances.size()
  │     │                 → 选中 192.168.1.11:8080
  │     │
  │     └── .block()  → 阻塞获取结果
  │
  ├── reconstructedUri = "http://192.168.1.11:8080/api/users/1"
  │
  ├── newRequest = Request.create(GET, "http://192.168.1.11:8080/api/users/1", ...)
  │
  ├── delegate.execute(newRequest, options)  → ApacheHttpClient / OkHttpClient
  │     │
  │     ├── httpClient.execute(apacheRequest)
  │     ├── socket.send(bytes)
  │     ├── socket.receive(bytes)
  │     └── return Response(200, headers, body)
  │
  └── return response
```
