
# 1. 核心类图

```
┌──────────────────────────────────────────┐
│        ReactiveLoadBalancer<T>            │
│  + choose(Request): Mono<Response<T>>    │
│  + choose(): Mono<Response<T>>           │
└──────────────┬───────────────────────────┘
               │ 实现
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌─────────┐ ┌──────────┐ ┌────────────┐
│RoundRobin│ │ Random   │ │ Custom     │
│Balancer  │ │Balancer  │ │Balancer    │
└─────────┘ └──────────┘ └────────────┘
    │          │
    ▼          ▼
┌──────────────────────────────────────────┐
│  ReactorServiceInstanceLoadBalancer       │
│  (extends ReactorLoadBalancer<ServiceInstance>)│
└──────────────────────────────────────────┘
```

# 2. RoundRobinLoadBalancer 完整源码

```java
public class RoundRobinLoadBalancer
        implements ReactorServiceInstanceLoadBalancer {

    private static final Log log =
            LogFactory.getLog(RoundRobinLoadBalancer.class);

    // 原子自增计数器 — 线程安全的轮询
    final AtomicInteger position;

    // 惰性提供 ServiceInstanceListSupplier
    final ObjectProvider<ServiceInstanceListSupplier>
            serviceInstanceListSupplierProvider;

    final String serviceId;

    public RoundRobinLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplierProvider,
            String serviceId) {
        this(supplierProvider, serviceId, new Random().nextInt(1000));
    }

    public RoundRobinLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplierProvider,
            String serviceId, int seedPosition) {
        this.serviceInstanceListSupplierProvider = supplierProvider;
        this.serviceId = serviceId;
        this.position = new AtomicInteger(seedPosition);
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // ① 获取 ServiceInstanceListSupplier
        ServiceInstanceListSupplier supplier =
                serviceInstanceListSupplierProvider
                        .getIfAvailable(NoopServiceInstanceListSupplier::new);

        // ② 获取实例列表（Flux → 取第一个 List）
        return supplier.get(request)
                .next()
                .map(serviceInstances ->
                        processInstanceResponse(supplier, serviceInstances));
    }

    private Response<ServiceInstance> processInstanceResponse(
            ServiceInstanceListSupplier supplier,
            List<ServiceInstance> serviceInstances) {

        // ③ 执行轮询选择
        Response<ServiceInstance> response =
                getInstanceResponse(serviceInstances);

        // ④ 如果 Supplier 支持回调，通知选中结果
        if (supplier instanceof SelectedInstanceCallback) {
            ((SelectedInstanceCallback) supplier)
                    .selectedServiceInstance(response.getServer());
        }
        return response;
    }

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {

        if (instances.isEmpty()) {
            if (log.isWarnEnabled()) {
                log.warn("No servers available for service: " + serviceId);
            }
            return new EmptyResponse();
        }

        // ⑤ 核心算法：原子自增 + 取模
        int pos = Math.abs(this.position.incrementAndGet());
        ServiceInstance instance = instances.get(pos % instances.size());

        return new DefaultResponse(instance);
    }
}
```

# 3. 原子操作的线程安全分析

```java
// AtomicInteger.incrementAndGet() → CAS 无锁自增
// 当 position 溢出时：
//   Integer.MAX_VALUE + 1 = Integer.MIN_VALUE (Java 溢出不抛异常)
//   Math.abs(Integer.MIN_VALUE) = Integer.MIN_VALUE (负数)
//   取模后可能得到负数索引

// 解决方案：使用 Math.abs + 与运算确保非负
// 实际源码中使用 Math.abs()，但 Java 绝对值对 MIN_VALUE 无效
// 业界最佳实践：
int pos = (int) (System.nanoTime() & 0x7FFFFFFF);  // 或使用更安全的取模
```

# 4. RandomLoadBalancer 完整源码

```java
public class RandomLoadBalancer
        implements ReactorServiceInstanceLoadBalancer {

    private static final Log log =
            LogFactory.getLog(RandomLoadBalancer.class);

    private final ObjectProvider<ServiceInstanceListSupplier>
            serviceInstanceListSupplierProvider;
    private final String serviceId;
    private final ThreadLocalRandom random;

    public RandomLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplierProvider,
            String serviceId) {
        this.serviceInstanceListSupplierProvider = supplierProvider;
        this.serviceId = serviceId;
        this.random = ThreadLocalRandom.current();
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                serviceInstanceListSupplierProvider
                        .getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request)
                .next()
                .map(instances -> getInstanceResponse(instances));
    }

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }

        // 使用 ThreadLocalRandom 避免竞争
        int index = random.nextInt(instances.size());
        ServiceInstance instance = instances.get(index);
        return new DefaultResponse(instance);
    }
}
```

# 5. ServiceInstanceListSupplier 装饰器链

## 5.1 DelegatingServiceInstanceListSupplier 基类

```java
public class DelegatingServiceInstanceListSupplier
        implements ServiceInstanceListSupplier {

    private final ServiceInstanceListSupplier delegate;

    public DelegatingServiceInstanceListSupplier(
            ServiceInstanceListSupplier delegate) {
        this.delegate = delegate;
    }

    protected ServiceInstanceListSupplier getDelegate() {
        return delegate;
    }

    @Override
    public String getServiceId() {
        return delegate.getServiceId();
    }

    @Override
    public Flux<ServiceInstance> get() {
        return delegate.get();
    }

    @Override
    public Flux<ServiceInstance> get(Request request) {
        return delegate.get(request);
    }
}
```

## 5.2 装饰器链执行流程

```java
// 链式调用等效于嵌套调用：
new CachingServiceInstanceListSupplier(
    new ZonePreferenceServiceInstanceListSupplier(
        new HealthCheckServiceInstanceListSupplier(
            new DiscoveryClientServiceInstanceListSupplier(
                discoveryClient, environment
            )
        , healthCheckInterval)
    , zone)
, cacheTtl, cacheCapacity)

// get() 调用顺序：
CachingSupplier.get()
  → 检查缓存 → 未命中
  → ZonePreferenceSupplier.get()
    → HealthCheckSupplier.get()
      → DiscoveryClientSupplier.get()
        → discoveryClient.getInstances(serviceId)  ← 实际查注册中心
      ← 返回全部实例 [A,B,C,D]
    ← 健康检查过滤：[A,B,C]  (D 不健康)
  ← Zone 过滤：[A,B]  (C 不在当前 Zone)
← 写入缓存 [A,B]
```

# 6. BlockingLoadBalancerClient 源码

```java
public class BlockingLoadBalancerClient
        implements LoadBalancerClient {

    private final LoadBalancerClientFactory loadBalancerClientFactory;

    @Override
    public ServiceInstance choose(String serviceId) {
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
                loadBalancerClientFactory.getInstance(serviceId,
                        ReactorServiceInstanceLoadBalancer.class);
        if (loadBalancer == null) {
            loadBalancer = loadBalancerClientFactory.getInstance(
                    serviceId, ReactorLoadBalancer.class);
        }
        if (loadBalancer == null) {
            throw new IllegalStateException(
                    "No load balancer available for " + serviceId);
        }

        // 响应式 → 阻塞桥接
        Response<ServiceInstance> loadBalancerResponse =
                Mono.from(loadBalancer.choose()).block();

        if (loadBalancerResponse == null) {
            throw new IllegalStateException(
                    "LoadBalancer returned null for " + serviceId);
        }
        return loadBalancerResponse.getServer();
    }

    @Override
    public <T> ServiceInstance choose(String serviceId,
            Request<T> request) {
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
                loadBalancerClientFactory.getInstance(serviceId,
                        ReactorServiceInstanceLoadBalancer.class);
        if (loadBalancer == null) {
            loadBalancer = loadBalancerClientFactory.getInstance(
                    serviceId, ReactorLoadBalancer.class);
        }

        Response<ServiceInstance> loadBalancerResponse =
                Mono.from(loadBalancer.choose(request)).block();

        return loadBalancerResponse != null
                ? loadBalancerResponse.getServer() : null;
    }

    @Override
    public <T> T execute(String serviceId,
            LoadBalancerRequest<T> request) throws IOException {
        ServiceInstance instance = choose(serviceId);
        if (instance == null) {
            throw new IllegalStateException(
                    "No instances available for " + serviceId);
        }
        return request.apply(instance);
    }
}
```

# 7. LoadBalancerUriTools — URI 重建

```java
public final class LoadBalancerUriTools {

    public static URI reconstructURI(
            ServiceInstance serviceInstance, URI original) {

        return UriComponentsBuilder.fromUri(original)
                .host(serviceInstance.getHost())
                .port(serviceInstance.getPort())
                .build()
                .toUri();
    }

    // 保留原始 URI 的 scheme（http/https）
    // 替换 host 和 port

    // 示例：
    // 输入 URI:      lb://user-service/api/users/1
    // ServiceInstance: {host: 192.168.1.10, port: 8080}
    // 输出 URI:      lb://192.168.1.10:8080/api/users/1
    // ↑ 注意：输出仍然保留了 lb:// scheme
    // Gateway 的 ReactiveLoadBalancerClientFilter 会再次替换为 http://
}
```

# 8. CompletionContext — 生命周期上下文

```java
public final class CompletionContext<RES, T, RC> {

    public enum Status {
        SUCCESS,        // 选择成功
        FAILED          // 选择失败
    }

    private final Status status;
    private final Request<RC> clientRequest;
    private final Response<T> clientResponse;
    private final Throwable throwable;

    // 构建成功上下文
    public static <RES, T, RC> CompletionContext<RES, T, RC> success(
            Request<RC> request, Response<T> response, Status status) {
        return new CompletionContext<>(status, request, response, null);
    }

    // 构建失败上下文
    public static <RES, T, RC> CompletionContext<RES, T, RC> failed(
            Throwable throwable, Request<RC> request) {
        return new CompletionContext<>(Status.FAILED, request, null, throwable);
    }
}
```

# 9. 性能关键路径分析

```
一次负载均衡调用的关键路径：

① choose() 入口                                      ~0ns
② ServiceInstanceListSupplier.get()                   ~10μs (缓存命中)
   └── 缓存未命中 → DiscoveryClient.getInstances()    ~5-50ms
③ instances.collectList()                             ~10μs
④ getInstanceResponse()                               ~5μs
   └── position.incrementAndGet() (CAS)               ~50ns
   └── instances.get(pos % size)                      ~10ns
⑤ 返回 DefaultResponse                                ~10ns

总延迟（缓存命中）: ~10-20μs
总延迟（缓存未命中）: ~5-50ms（取决于注册中心响应）
```
