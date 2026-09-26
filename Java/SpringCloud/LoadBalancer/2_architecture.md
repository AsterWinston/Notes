
# 1. 整体架构

Spring Cloud LoadBalancer 采用**分层可组合**架构设计，核心分为四层：

```
┌─────────────────────────────────────────────────────────┐
│                    集成层 (Integration)                  │
│  LoadBalancerInterceptor  │  ReactorLoadBalancerExch...  │
│  (RestTemplate)           │  (WebClient)                │
│  FeignBlockingLoadBalancerClient  │  ReactiveLoadBal...  │
│  (Feign)                          │  (Gateway)          │
├─────────────────────────────────────────────────────────┤
│                  负载均衡层 (Balancer)                    │
│  ReactorLoadBalancer<T>                                  │
│   ├── RoundRobinLoadBalancer    (轮询)                   │
│   ├── RandomLoadBalancer        (随机)                   │
│   └── [Custom] Weighted / ConsistentHash / ...          │
├─────────────────────────────────────────────────────────┤
│                 实例供应层 (Supplier)                     │
│  ServiceInstanceListSupplier (函数式接口)                 │
│   ├── DiscoveryClientServiceInstanceListSupplier         │
│   ├── CachingServiceInstanceListSupplier                 │
│   ├── ZonePreferenceServiceInstanceListSupplier          │
│   ├── HealthCheckServiceInstanceListSupplier             │
│   └── SameInstancePreferenceServiceInstanceListSupplier  │
├─────────────────────────────────────────────────────────┤
│                 配置管理层 (Configuration)                │
│  LoadBalancerClientFactory  │  @LoadBalancerClient       │
│  LoadBalancerClients        │  @LoadBalancerClients      │
└─────────────────────────────────────────────────────────┘
```

# 2. 核心接口

## 2.1 ReactorLoadBalancer<T> — 负载均衡器

负载均衡算法的顶层抽象，基于 Reactor：

```java
public interface ReactorLoadBalancer<T> {

    // 根据请求上下文选择实例
    Mono<Response<T>> choose(Request request);

    // 无上下文选择（默认实现调用 choose(Request)）
    default Mono<Response<T>> choose() {
        return choose(REACTOR_SERVICE_INSTANCE_REQUEST);
    }
}
```

## 2.2 ServiceInstanceListSupplier — 实例供应器

函数式接口，负责提供实例列表，支持链式装饰：

```java
public interface ServiceInstanceListSupplier
        extends Supplier<Flux<ServiceInstance>> {

    // 返回服务 ID
    String getServiceId();

    // 提供带有请求上下文的实例列表
    default Flux<ServiceInstance> get(Request request) {
        return get();
    }

    // 提供全量实例列表
    @Override
    Flux<ServiceInstance> get();
}
```

## 2.3 LoadBalancerClientFactory — 客户端工厂

为每个 `serviceId` 创建独立的 Spring 子容器：

```java
public class LoadBalancerClientFactory
        extends NamedContextFactory<LoadBalancerClientSpecification> {

    public static final String NAMESPACE = "loadbalancer";
    public static final String PROPERTY_NAME = "loadbalancer.client.name";

    public LoadBalancerClientFactory() {
        super(LoadBalancerClientConfiguration.class, NAMESPACE, PROPERTY_NAME);
    }
}
```

## 2.4 ReactorServiceInstanceLoadBalancer — 服务实例负载均衡

组合了 `ReactorLoadBalancer` 和 `ServiceInstanceListSupplier`：

```java
public interface ReactorServiceInstanceLoadBalancer
        extends ReactorLoadBalancer<ServiceInstance> {
}
```

# 3. ServiceInstanceListSupplier 装饰器链

`ServiceInstanceListSupplier` 采用**装饰器模式**，通过链式组合实现多种功能：

```
DiscoveryClientServiceInstanceListSupplier     ← 从注册中心获取原始列表
    │              (每次调用触发服务发现)
    ▼
CachingServiceInstanceListSupplier             ← 缓存 N 秒
    │              (避免每次都查注册中心)
    ▼
ZonePreferenceServiceInstanceListSupplier      ← 同 Zone 优先
    │              (多可用区亲和路由)
    ▼
HealthCheckServiceInstanceListSupplier         ← 健康检查过滤
    │              (剔除不健康实例)
    ▼
SameInstancePreferenceServiceInstanceListSupplier  ← 会话粘滞
    │              (尽量路由到上次的实例)
    ▼
...自定义 Supplier...
    │
    ▼
ReactorLoadBalancer.choose()                   ← 负载均衡算法
```

## 3.1 默认 Supplier 链配置

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnDiscoveryEnabled
public class LoadBalancerClientConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()              // ① 从注册中心获取
                .withCaching()                      // ② 开启缓存
                .withZonePreference()               // ③ 同 Zone 优先
                .withHealthChecks()                 // ④ 健康检查过滤
                .build(context);
    }
}
```

# 4. 核心请求模型

## 4.1 Request<T> — 请求对象

```java
public interface Request<T> {

    // 请求上下文（如 ClientRequest）
    T getContext();

    // 透传 Hint（给 LoadBalancerLifecycle 使用）
    default Map<String, Object> getHints() {
        return Collections.emptyMap();
    }
}

public class DefaultRequest<T> implements Request<T> {
    private final T context;

    public DefaultRequest(T context) {
        this.context = context;
    }

    @Override
    public T getContext() {
        return context;
    }
}
```

## 4.2 Response<T> — 响应对象

```java
public class DefaultResponse implements Response<ServiceInstance> {
    private final ServiceInstance server;

    public DefaultResponse(ServiceInstance server) {
        this.server = server;
    }

    @Override
    public ServiceInstance getServer() {
        return server;
    }

    @Override
    public boolean hasServer() {
        return server != null;
    }
}

public class EmptyResponse extends DefaultResponse {
    public EmptyResponse() {
        super(null);
    }
}
```

# 5. ServiceInstance — 服务实例抽象

```java
public interface ServiceInstance {

    // 实例唯一ID（默认 host:port）
    default String getInstanceId() {
        return null;
    }

    // 所属服务名
    String getServiceId();

    // IP / 主机名
    String getHost();

    // 端口
    int getPort();

    // 是否 HTTPS
    boolean isSecure();

    // 完整 URI
    URI getUri();

    // 元数据（如 version, weight, zone, gray 等）
    Map<String, String> getMetadata();

    // 默认实现
    default String getScheme() {
        return isSecure() ? "https" : "http";
    }
}
```

# 6. LoadBalancerLifecycle — 生命周期回调

```java
public interface LoadBalancerLifecycle<RC, RES, T> {

    // 是否支持该请求/响应类型
    default boolean supports(Class<RC> requestContextClass,
                             Class<RES> responseClass,
                             Class<T> serverTypeClass) {
        return true;
    }

    // 选择实例前回调
    void onStart(Request<RC> request);

    // 选择实例完成回调（成功）
    void onComplete(CompletionContext<RES, T, RC> completionContext);

    // 选择实例完成回调（失败）
    default void onComplete(CompletionContext<RES, T, RC> completionContext) {
        // 默认委托给另一个回调
    }
}
```

**生命周期触发顺序**：

```
① onStart(request)        ← 开始选择
② choose(request)         ← 执行负载均衡
③ onComplete(context)     ← 选择完成
```

# 7. LoadBalancerClientFactory 子容器模型

```java
// LoadBalancerClientFactory 继承自 NamedContextFactory
// 为每个服务创建独立的 ApplicationContext 子容器

public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
        implements DisposableBean, ApplicationContextAware {

    // 每个 name 对应一个 AnnotationConfigApplicationContext
    private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();

    // 获取指定 name 的 Bean
    public <T> T getInstance(String name, Class<T> type) { ... }

    // 获取 Lazy Provider（按需创建）
    public <T> ObjectProvider<T> getLazyProvider(String name, Class<T> type) { ... }
}
```

**容器隔离示意图**：

```
LoadBalancerClientFactory（父容器）
  │
  ├── "user-service" 子容器
  │     ├── ServiceInstanceListSupplier (user-service 专用)
  │     ├── ReactorLoadBalancer<ServiceInstance> (RoundRobin)
  │     └── LoadBalancerClientConfiguration
  │
  ├── "order-service" 子容器
  │     ├── ServiceInstanceListSupplier (order-service 专用)
  │     ├── ReactorLoadBalancer<ServiceInstance> (Random)
  │     └── LoadBalancerClientConfiguration
  │
  └── "default" 子容器（默认配置）
```

# 8. 设计模式总结

| 模式 | 应用 | 说明 |
|------|------|------|
| **装饰器** | `ServiceInstanceListSupplier` 链 | 缓存、Zone、健康检查叠加 |
| **策略** | `ReactorLoadBalancer` 实现 | 轮询/随机/自定义算法 |
| **工厂方法** | `LoadBalancerClientFactory` | 创建服务级子容器 |
| **模板方法** | `NamedContextFactory` | 子容器生命周期管理 |
| **观察者** | `LoadBalancerLifecycle` | 选择完成回调 |

# 9. 响应式与阻塞双通道

```
┌──────────────────────────────────────────────────────┐
│                  响应式（Reactive）                    │
│                                                       │
│  WebClient / Gateway                                 │
│       │                                              │
│       ▼                                              │
│  ReactorLoadBalancer.choose()                        │
│       │                                              │
│       ▼                                              │
│  Mono<Response<ServiceInstance>>                     │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                  阻塞（Blocking）                     │
│                                                       │
│  RestTemplate / Feign                                │
│       │                                              │
│       ▼                                              │
│  LoadBalancerClient.choose(String serviceId)         │
│       │                                              │
│       ▼                                              │
│  ServiceInstance  (内部调用 choose().block())         │
└──────────────────────────────────────────────────────┘
```

核心阻塞桥接类 `BlockingLoadBalancerClient`：

```java
public class BlockingLoadBalancerClient implements LoadBalancerClient {

    private final LoadBalancerClientFactory loadBalancerClientFactory;

    @Override
    public ServiceInstance choose(String serviceId) {
        // 获取响应式 LoadBalancer，阻塞调用
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
                loadBalancerClientFactory.getInstance(serviceId,
                        ReactorServiceInstanceLoadBalancer.class);
        if (loadBalancer == null) {
            loadBalancer = loadBalancerClientFactory.getInstance(serviceId,
                    ReactorLoadBalancer.class);
        }
        Response<ServiceInstance> response = Mono.from(loadBalancer.choose())
                .block();
        return response != null ? response.getServer() : null;
    }

    @Override
    public <T> ServiceInstance choose(String serviceId, Request<T> request) {
        ReactorLoadBalancer<ServiceInstance> loadBalancer = ...;
        Response<ServiceInstance> response = Mono.from(
                loadBalancer.choose(request)).block();
        return response != null ? response.getServer() : null;
    }
}
```
