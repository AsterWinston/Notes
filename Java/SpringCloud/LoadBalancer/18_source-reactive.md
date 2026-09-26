
# 1. ReactorLoadBalancerExchangeFilterFunction

WebClient 集成中的核心响应式过滤器：

```java
public class ReactorLoadBalancerExchangeFilterFunction
        implements ExchangeFilterFunction {

    private final LoadBalancerClientFactory loadBalancerClientFactory;
    private final LoadBalancerProperties properties;

    @Override
    public Mono<ClientResponse> filter(
            ClientRequest clientRequest, ExchangeFunction next) {

        URI originalUrl = clientRequest.url();
        String serviceId = originalUrl.getHost();

        // ① 非负载均衡请求直接透传
        if (serviceId == null) {
            return next.exchange(clientRequest);
        }

        // ② 构建 Hint
        RequestDataContext context = new RequestDataContext(
                new RequestData(clientRequest),
                getHint(serviceId, Collections.emptyMap()));

        // ③ 选择实例
        return choose(serviceId, context)
                .flatMap(response -> {
                    if (response == null || !response.hasServer()) {
                        return Mono.error(
                                new ServiceInstanceNotFoundException(serviceId));
                    }

                    ServiceInstance instance = response.getServer();

                    // ④ 重建 URI
                    URI newUrl = LoadBalancerUriTools.reconstructURI(
                            instance, originalUrl);

                    // ⑤ 创建新请求并继续
                    ClientRequest newRequest = ClientRequest.from(clientRequest)
                            .url(newUrl)
                            .build();

                    return next.exchange(newRequest);
                });
    }

    Mono<Response<ServiceInstance>> choose(
            String serviceId, RequestDataContext context) {

        // 获取响应式负载均衡器
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
                loadBalancerClientFactory.getInstance(serviceId,
                        ReactorServiceInstanceLoadBalancer.class);

        if (loadBalancer == null) {
            return Mono.error(new IllegalStateException(
                    "No loadbalancer available for " + serviceId));
        }

        // 异步选择
        return loadBalancer.choose(new DefaultRequest<>(context));
    }
}
```

# 2. ReactiveLoadBalancerClientFilter（Gateway）

```java
public class ReactiveLoadBalancerClientFilter
        implements GlobalFilter, Ordered {

    public static final int LOAD_BALANCER_CLIENT_FILTER_ORDER = 10100;

    private final LoadBalancerClientFactory clientFactory;
    private final LoadBalancerProperties properties;

    @Override
    public int getOrder() {
        return LOAD_BALANCER_CLIENT_FILTER_ORDER;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
            GatewayFilterChain chain) {

        URI url = exchange.getAttribute(
                ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
        String schemePrefix = exchange.getAttribute(
                ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR);

        // 非 lb 协议：跳过
        if (url == null || (!"lb".equals(url.getScheme())
                && !"lb".equals(schemePrefix))) {
            return chain.filter(exchange);
        }

        String serviceId = url.getHost();

        // 收集生命周期回调
        Set<LoadBalancerLifecycle> supportedLifecycleProcessors =
                LoadBalancerLifecycleValidator
                        .getSupportedLifecycleProcessors(
                                clientFactory.getInstances(serviceId,
                                        LoadBalancerLifecycle.class),
                                RequestDataContext.class,
                                ResponseData.class,
                                ServiceInstance.class);

        // 构建请求上下文
        RequestDataContext requestContext = new RequestDataContext(
                new RequestData(exchange.getRequest()), getHint(serviceId));

        // 触发 onStart 回调
        supportedLifecycleProcessors.forEach(
                lifecycle -> lifecycle.onStart(
                        new DefaultRequest<>(requestContext)));

        // 执行负载均衡
        return choose(serviceId, requestContext)
                .flatMap(response -> {
                    if (!response.hasServer()) {
                        return Mono.error(NotFoundException.create(
                                LoadBalancerProperties.class,
                                "Unable to find instance for " + serviceId));
                    }

                    ServiceInstance instance = response.getServer();

                    // 重建 URI
                    URI requestUrl = reconstructURI(instance, url);
                    exchange.getAttributes().put(
                            ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR,
                            requestUrl);

                    // HTTPS 处理
                    if (instance.isSecure()) {
                        exchange.getAttributes().put(
                                ServerWebExchangeUtils.GATEWAY_SCHEME_PREFIX_ATTR,
                                "https");
                    }

                    // 触发 onComplete 回调
                    supportedLifecycleProcessors.forEach(lifecycle ->
                            lifecycle.onComplete(
                                    CompletionContext.success(...)));

                    return chain.filter(exchange);
                })
                .onErrorResume(throwable -> {
                    // 触发 onComplete 失败回调
                    supportedLifecycleProcessors.forEach(lifecycle ->
                            lifecycle.onComplete(
                                    CompletionContext.failed(throwable, ...)));
                    return Mono.error(throwable);
                });
    }
}
```

# 3. 响应式与非响应式对比

## 3.1 RestTemplate（阻塞）路径

```java
// 线程模型：Tomcat worker 线程
restTemplate.getForObject("http://user-service/api/users/1", User.class)
    │
    ├── LoadBalancerInterceptor.intercept()           [同一线程]
    ├── BlockingLoadBalancerClient.choose()
    │     └── Mono.from(loadBalancer.choose()).block() [阻塞等待]
    ├── ApacheHttpClient.execute()                     [同一线程，阻塞 I/O]
    ├── 反序列化响应                                    [同一线程]
    └── 返回 User                                      [同一线程]
```

## 3.2 WebClient（响应式）路径

```java
// 线程模型：Reactor Netty event loop
webClient.get().uri("http://user-service/api/users/1").retrieve()
    .bodyToMono(User.class)
    │
    ├── ReactorLBExchangeFilterFunction.filter()  [EventLoop 线程 A]
    │     ├── choose() → Mono<Response<...>>      [异步，不阻塞]
    │     └── flatMap → next.exchange()           [异步调度]
    ├── Reactor Netty HTTP 请求                    [EventLoop 线程 B]
    ├── 反序列化响应                               [EventLoop 线程 C]
    └── 返回 Mono<User>                            [异步]
```

# 4. 背压（Backpressure）支持

```java
// ServiceInstanceListSupplier 返回的是 Flux<ServiceInstance>
// 支持响应式背压机制

// 自定义背压感知的 Supplier
public class BackpressureAwareSupplier
        extends DelegatingServiceInstanceListSupplier {

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                // 限制下游请求速率（每秒最多 100 次选择）
                .limitRate(100)
                // 缓冲区最多 32 个实例
                .onBackpressureBuffer(32);
    }
}
```

# 5. 错误信号传播

```java
// 响应式链中的错误处理
public Mono<User> getUser(Long id) {
    return webClient.get()
            .uri("http://user-service/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            // ① 选择不到实例
            .onErrorMap(ServiceInstanceNotFoundException.class,
                    e -> new ServiceUnavailableException("user-service"))
            // ② 实例连接失败
            .onErrorMap(ConnectException.class,
                    e -> new ServiceUnavailableException("user-service"))
            // ③ HTTP 4xx/5xx
            .onStatus(HttpStatus::isError, response -> {
                if (response.statusCode() == HttpStatus.SERVICE_UNAVAILABLE) {
                    return Mono.error(new ServiceUnavailableException(...));
                }
                return Mono.error(new RuntimeException(...));
            })
            // ④ 超时
            .timeout(Duration.ofSeconds(5))
            // ⑤ 降级
            .onErrorResume(e -> Mono.just(User.FALLBACK));
}
```

# 6. Reactor 调度器（Scheduler）

```java
// LoadBalancer 内部的调度器使用

// ① Supplier 获取（可能阻塞 → 用 boundedElastic）
public class DiscoveryClientServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    @Override
    public Flux<ServiceInstance> get() {
        return Mono.fromCallable(() -> {
                    // 阻塞调用：查注册中心
                    return discoveryClient.getInstances(getServiceId());
                })
                .subscribeOn(Schedulers.boundedElastic())  // 切换线程池
                .flatMapMany(Flux::fromIterable);
    }
}

// ② 健康检查（异步 HTTP）
public class HealthCheckServiceInstanceListSupplier {

    private Mono<Boolean> checkHealth(ServiceInstance instance) {
        return WebClient.create()
                .get()
                .uri(instance.getUri() + "/actuator/health")
                .retrieve()
                .toBodilessEntity()
                .map(r -> r.getStatusCode().is2xxSuccessful())
                // 不需要 boundedElastic — WebClient 本身是异步的
                .onErrorReturn(false);
    }
}
```

# 7. 选择超时处理

```java
// 为 choose() 操作设置超时
public Mono<Response<ServiceInstance>> chooseWithTimeout(
        ReactorLoadBalancer<ServiceInstance> balancer,
        Request request, Duration timeout) {

    return Mono.from(balancer.choose(request))
            .timeout(timeout)
            .onErrorMap(TimeoutException.class, e ->
                    new RuntimeException("LoadBalancer choose timeout"));
}

// 使用
ReactorLoadBalancer<ServiceInstance> balancer = ...;
Mono<Response<ServiceInstance>> choice = chooseWithTimeout(
        balancer, request, Duration.ofSeconds(3));
```

# 8. 并发选择

```java
// 同时从多个 LoadBalancer 选择（如不同版本的服务）
public Mono<Tuple2<ServiceInstance, ServiceInstance>> chooseBoth(
        String serviceIdV1, String serviceIdV2,
        LoadBalancerClientFactory factory) {

    ReactorLoadBalancer<ServiceInstance> lb1 =
            factory.getInstance(serviceIdV1, ...);
    ReactorLoadBalancer<ServiceInstance> lb2 =
            factory.getInstance(serviceIdV2, ...);

    return Mono.zip(
            Mono.from(lb1.choose()),
            Mono.from(lb2.choose())
    );
}
```

# 9. 响应式链最佳实践

```java
@Service
public class ReactiveOrderService {

    private final WebClient webClient;
    private final LoadBalancerClientFactory lbFactory;

    public Mono<Order> createOrder(CreateOrderRequest req) {
        String serviceId = "order-service";

        return Mono.fromCallable(() -> {
                    ReactorLoadBalancer<ServiceInstance> lb =
                            lbFactory.getInstance(serviceId, ...);
                    return lb;
                })
                .flatMap(lb -> Mono.from(lb.choose(
                        new DefaultRequest<>(buildContext(req)))))
                .timeout(Duration.ofSeconds(3))
                .switchIfEmpty(Mono.error(
                        new ServiceInstanceNotFoundException(serviceId)))
                .flatMap(response -> {
                    ServiceInstance instance = response.getServer();
                    return webClient.post()
                            .uri(instance.getUri() + "/api/orders")
                            .bodyValue(req)
                            .retrieve()
                            .bodyToMono(Order.class);
                })
                .onErrorResume(e -> {
                    log.error("Failed to create order", e);
                    return Mono.just(Order.FALLBACK);
                });
    }
}
```
