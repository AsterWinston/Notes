# 1. 集成架构

```text
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

## 1.1 注入点的源码坐标

"自动配置注入"的具体落点在 `FeignClientFactoryBean.loadBalance`（`21_source-dynamic-proxy.md` §3.3 的 `getOptional(context, Client.class)`）：`FeignRibbonClientAutoConfiguration` 的继任者按 classpath 条件装配这个装饰 Client 进**每个客户端的子上下文**。架构上值得注意两点：Feign 核心（`feign-core`）对 LoadBalancer **零依赖**——这个 Client 是 Spring Cloud 集成层的产物，纯 OpenFeign 用不到（`12_feign-builder.md` §9 手动挂的就是它）；`target` 侧的 `HardCodedTarget` 携带服务名（`name`），选址查询用的名字来源在此定死——`@FeignClient(name)` 与 `url` 的语义分岔（有 url 时根本不走 `loadBalance`）也是在这一层做出的。

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

（骨架简化还原。）真实代码有三处与示意不同，每处都对应一个可观察行为：**第 2 步不是裸 `choose(serviceId)`**——它会用请求的 method/headers 构造 `RequestDataContext` 再 `execute(serviceId, request, factory)`，这是 §9 的 hint 与 `13_loadbalancer.md` §5 灰度头过滤的数据通道；**URL 重写不靠字符串拼接**——而是 `new RequestTemplate(originalUri).target(...)` 级别的模板重写并处理 scheme（实例 `isSecure()`/metadata `management.port` 等），`reconstructURI` 的拼接在"实例 URI 自带 path 前缀"（网关式注册）时行为与示意不同；**选不到实例的异常路径要背下来**：`IllegalStateException` 不是 `IOException`，`SynchronousMethodHandler` 的重试 catch 接不住它——"No instances available" **不重试、直达熔断层计为失败**（`14_circuit-breaker.md` §3.2 名单里它属于 RuntimeException 默认计入），注册中心抖动时熔断器被空列表打开是这条路径的连锁反应，`record-exceptions` 是否排除它是个真实的架构决策。

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

## 3.1 两种 null 与一次 block 的代价

返回 null 的两种成因要分开归因：`getInstance(serviceId)` 为 null → **该服务没有负载均衡器上下文**（依赖或条件装配问题，通常启动就有迹象）；`EmptyResponse`/无 server → **选址时实例列表为空**（注册/过滤/namespace 问题，`15_service-discovery.md` §10 决策树的②③分支在此分岔）。两种都会被 §2 转成 `No instances available`，但排查方向一个是查自己的依赖树、一个是查注册中心。另一句要当真的是 `block()`：同步 Feign 路径上业务线程在这里**无超时地等**选址 Mono——默认实现里 supplier 链全是内存/本地缓存操作无所谓，但自定义 supplier 若引入远程调用（§5 的纪律在 `13_loadbalancer.md` §5 说过），这里就是"LB 卡死业务线程"的现场，**block 不带 timeout 是集成层的既定事实，不是你可以依赖的安全网**。

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

```text
LoadBalancerClientFactory
  ├── user-service → ApplicationContext (user-service)
  │     ├── RoundRobinLoadBalancer
  │     └── DiscoveryClientServiceInstanceListSupplier
  │
  └── order-service → ApplicationContext (order-service)
        ├── RandomLoadBalancer
        └── DiscoveryClientServiceInstanceListSupplier
```

## 4.1 与 Feign 子上下文：同一家族的两兄弟

`LoadBalancerClientFactory` 和 `FeignClientFactory`（`4_configuration.md` §1.1）都继承 `NamedContextFactory`，机制同款（懒创建、父上下文可见、`@LoadBalancerClient(configuration=...)` 的注入通道与 `@FeignClient(configuration=...)` 对称），但**是两个独立的容器集**：user-service 的 Feign 上下文里有 Encoder/Retryer，它的 LB 上下文里才有 LoadBalancer/supplier。实践推论三条：给某服务换负载均衡策略要动 LB 侧（`13_loadbalancer.md` §6），在 Feign 配置类里塞 `ReactorLoadBalancer` Bean 是没用的（放错了房子）；两侧的作用域键也**不同名**——Feign 按 contextId/name，LB 按 serviceId（`name`），依赖两者的配置（比如灰度）要保证三个名字一致；图里"每服务一个 RoundRobin/Random 实例"意味着**轮询游标、缓存、统计都是按服务隔离的**，不存在跨服务的全局轮询相位。

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

（示意骨架：真实自动配置靠一组 `@Conditional` 嵌套类挑选 supplier 组合与算法 Bean，不是并列两个 @Bean。）两个读点：zone 偏好版的链序是 发现→**zone 过滤→缓存**（默认版无过滤）——过滤在缓存**内层**，zone 视图随缓存一起 35s 刷新；`getLazyProvider` 把 supplier 注入算法时保持**惰性**，避免每服务上下文循环装配。想加"实例级过滤"（只调 v2 分组）的官方姿势与自定义链的注册方式，`13_loadbalancer.md` §9 已给；本篇关注的是这些都在 LB 子上下文发生，与 Feign 组件的装配互不干涉。

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

## 6.1 轮询实现的三个边界

`position` 是**每服务上下文一个**的 AtomicInteger（§4.1 隔离的落点）：高并发下相位可能跳号（increment 与 get 之间列表长度变化时），分布无碍但"严格逐台轮转"不成立。`Math.abs(position.incrementAndGet())` 有教科书级角落案例：值到达 `Integer.MIN_VALUE` 时 `abs` 仍为负、取模得负下标越界——真实代码用 `Integer.MAX_VALUE &` 位掩规避，示意版留作面试梗。**"忽略权重"是本算法的正式行为**：Nacos 控制台调权重对默认 RoundRobin 完全无效（`15_service-discovery.md` §2.4），要权重生效换 `NacosLoadBalancer`（`13_loadbalancer.md` §4.3），或自建按 metadata.weight 加权的 supplier 视图。

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

## 7.1 失效时机那条要拆穿一半

第 1 条准确：`Flux.cache` 语义下 TTL 过后的**第一个请求**同步触发重新拉取（该请求承担 DiscoveryClient 调用延迟，缓存击穿时刻的 RT 毛刺来源）。第 2 条要纠偏：默认实现**没有**"Nacos 推送直接顶穿缓存"的通路——`DiscoveryClientServiceInstanceListSupplier` 拉的是 DiscoveryClient 的本地视图；Nacos 的订阅推送更新的是 **Nacos 客户端自己的服务缓存**（`NamingService` 内部），要让推送加速 LB 侧感知，需要的是缩短 `spring.cloud.loadbalancer.cache.ttl` 或使用带事件驱动的自定义 supplier。所以端到端生效时间 = 注册中心传播（各家模型，`15_service-discovery.md` §8）+ 本缓存剩余寿命，**两级缓存叠加才是"下线后还能调到"的完整解释**；`5s` 的激进 TTL 会把 DiscoveryClient 拉取频率放大 7 倍，改之前想清楚压的是哪一级。

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

## 8.1 一个值得注意的行为：空列表返回 Flux.empty()

`instances.isEmpty()` 时返回**空 Flux 而不是 `Flux.just(空列表)`**——对上层 §6 的 `.next()` 来说，空 Flux 意味着 Mono 永远没有值，`block()` 返回 null，走的是 §3.1 的"response == null"分支而非 EmptyResponse。链路后果相同（`No instances available`），但读栈找差异时这里能对上号。"与注册中心交互"三条的真实延迟模型差异更大（`15_service-discovery.md` §3.3/§4.3）：Eureka 这条拿的是**已是缓存**的视图（DiscoveryClient 层自带 30s fetch 缓存，再叠 LB 缓存 = 三级）；Nacos 临时实例模式下 `getInstances` 走客户端本地服务表（订阅推送维护，正常不打 OpenAPI——示例的 OpenAPI 说法是无订阅/首次拉取的形态）；Consul 每次真 HTTP 调用，所以它的 LB 缓存 TTL 最不该被调小。

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

Feign 集成的现实通道要说精确：`FeignBlockingLoadBalancerClient` 构造的 `RequestData` 里携带的是**HTTP method + headers**（§2 还原注记），`hint` 属性族主要服务于 `@LoadBalancerClient` 侧的 hint 策略与 RestTemplate/WebClient 路径；Feign 灰度生态的主流做法是把标记**放进 header**，自定义策略从 `ctx.getClientRequest().getHeaders()` 读（`13_loadbalancer.md` §5 的 GrayLoadBalancer 即此路线）——两条通道数据源同一个（RequestTemplate 生成的 headers），选 header 路线的收益是标记天然随请求可观测、可被网关透传。attributes 路线留给"不想污染报文"的内部标记。

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

"自动规避"要加三个限定。其一，RoundRobin 的"换实例"来自游标递增，属**顺带效果**而非故障排除：RandomLoadBalancer 重试可能连击同一台坏实例，实例数 2 时重试命中故障机概率也真实存在——"排除已试实例"的严格语义属于 `retry.enabled` 的 LoadBalancerRetryPolicy（spring-retry 上下文里维护 `EXCLUDED_SERVERS`，`13_loadbalancer.md` §10 的那个开关）或熔断器把坏实例的失败计入统计。其二，本图前提是 **Feign Retryer 已开启**（默认 NEVER_RETRY，一次都试不出来，`9_retry.md` §1.1）。其三，换实例重试只对 IOException 家族可靠，`No instances available` 这类 IllegalStateException 根本不进重试（§2 末尾）。生产上"故障实例快速摘除"的正解仍是三层配合：连接失败重试换实例（次数 1-2）+ 提供方优雅下线（`15` §8.1）+ 注册中心健康剔除，别指望轮询游标替你排雷。

# 11. 完整时序图

```text
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

## 11.1 沿这张图收敛三类故障

**`No instances available for X`**：停在 `choose` 之前的两帧——先看 `getInstance(serviceId)` 有无（§3.1 的装配问题），再看列表为何空（注册/namespace/过滤链，`15` §10 决策树），并注意它不进重试直达熔断（§2）。**选址阶段耗时毛刺**：`缓存未命中` 分支被踩到的请求承担 DiscoveryClient 拉取（§7.1 击穿毛刺），Consul 类每次真 IO 的实现最明显。**URL 改写后 404/连错服务**：检查 `reconstructedUri` 的拼装——实例注册 URI 自带 context-path 或网关前缀（§2 的 scheme/path 注记）时拼接结果会多一层，提供方注册的 IP:PORT 是否可路由（容器网络 NAT 注册错地址，`15` §1.2 第三行）。一图三查，这就是把集成链路画到"哪个方法哪一行"粒度的意义。

## 11.2 Feign 系列阅读地图（批次收尾）

```text
入门   1_introduction → 2_quick-start → 3_annotations
配置   4_configuration → 5_contract → 7_interceptor → 11_timeout → 10_logging
编解码 6_encoder-decoder → 17_compression → 18_file-upload → 20_response-mapping
容错   8_error-handling → 9_retry → 13_loadbalancer → 14_circuit-breaker → 16_fallback
集成   15_service-discovery → 19_http-client-engines → 12_feign-builder
源码   21_source-dynamic-proxy → 22_source-contract-methodhandler
       → 24_source-loadbalancer-integration → 23_source-request-pipeline
实践   25_best-practices → 26_performance-tuning
```
