
# 1. @LoadBalancerClient 注解

## 1.1 注解定义

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(LoadBalancerClientConfigurationRegistrar.class)
public @interface LoadBalancerClient {

    // 服务名（必填）
    @AliasFor("name")
    String value() default "";

    @AliasFor("value")
    String name() default "";

    // 配置类（必填）
    Class<?>[] configuration() default {};
}
```

## 1.2 @LoadBalancerClients 批量配置

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(LoadBalancerClientConfigurationRegistrar.class)
public @interface LoadBalancerClients {

    // 多个服务配置
    LoadBalancerClient[] value() default {};

    // 默认配置（所有服务共享）
    Class<?>[] defaultConfiguration() default {};
}
```

# 2. 服务级别自定义负载均衡

## 2.1 不同服务使用不同算法

```java
@Configuration
@LoadBalancerClients({
    @LoadBalancerClient(name = "user-service",
            configuration = UserServiceConfig.class),
    @LoadBalancerClient(name = "order-service",
            configuration = OrderServiceConfig.class),
    @LoadBalancerClient(name = "payment-service",
            configuration = PaymentServiceConfig.class)
})
@LoadBalancerClients(defaultConfiguration = DefaultConfig.class)
public class LoadBalancerGlobalConfig {
}

// user-service：轮询 + 长缓存 + 健康检查
@Configuration(proxyBeanMethods = false)
class UserServiceConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> userServiceBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }

    @Bean
    public ServiceInstanceListSupplier userServiceSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withCaching()               // 缓存 60s（对稳定服务）
                .withHealthChecks()
                .build(context);
    }
}

// order-service：随机
@Configuration(proxyBeanMethods = false)
class OrderServiceConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> orderServiceBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}

// payment-service：Nacos 加权
@Configuration(proxyBeanMethods = false)
class PaymentServiceConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> paymentServiceBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new NacosLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}

// 默认配置：轮询 + Zone 感知
@Configuration(proxyBeanMethods = false)
class DefaultConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> defaultBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }

    @Bean
    public ServiceInstanceListSupplier defaultSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withCaching()
                .withZonePreference()
                .build(context);
    }
}
```

# 3. 自定义 ServiceInstanceListSupplier

## 3.1 基于版本的实例过滤

```java
public class VersionBasedServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final String targetVersion;

    public VersionBasedServiceInstanceListSupplier(
            ServiceInstanceListSupplier delegate, String targetVersion) {
        super(delegate);
        this.targetVersion = targetVersion;
    }

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .filter(instance -> {
                    String version = instance.getMetadata().get("version");
                    return targetVersion.equals(version);
                });
    }
}

// 使用
@Bean
public ServiceInstanceListSupplier versionSupplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withBase(original -> new VersionBasedServiceInstanceListSupplier(
                    original, "v2"))
            .build(context);
}
```

## 3.2 健康标记过滤

```java
public class HealthStatusServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .filter(this::isHealthy);
    }

    private boolean isHealthy(ServiceInstance instance) {
        String status = instance.getMetadata().getOrDefault("health", "UP");
        return "UP".equalsIgnoreCase(status);
    }
}
```

# 4. 自定义 LoadBalancerLifecycle

```java
@Component
public class MetricsLoadBalancerLifecycle
        implements LoadBalancerLifecycle<RequestDataContext, Object, ServiceInstance> {

    private final MeterRegistry meterRegistry;

    @Override
    public void onStart(Request<RequestDataContext> request) {
        // 开始选择：记录时间戳
        request.getHints().put("startTime", System.currentTimeMillis());
    }

    @Override
    public void onComplete(
            CompletionContext<Object, ServiceInstance, RequestDataContext> context) {
        Long startTime = (Long) context.getClientRequest()
                .getHints().get("startTime");

        if (startTime != null) {
            long duration = System.currentTimeMillis() - startTime;
            meterRegistry.timer("loadbalancer.choose.duration")
                    .record(duration, TimeUnit.MILLISECONDS);
        }

        ServiceInstance instance = context.getClientResponse() != null
                ? (ServiceInstance) context.getClientResponse().getServer() : null;

        if (instance != null) {
            meterRegistry.counter("loadbalancer.choose.count",
                    "serviceId", instance.getServiceId(),
                    "host", instance.getHost()).increment();
        }
    }
}
```

# 5. 配置隔离与分包策略

推荐的项目包结构：

```
com.example.config
  ├── loadbalancer/
  │   ├── config/                     ← 不被 @ComponentScan 扫描
  │   │   ├── UserServiceLBConfig.java
  │   │   ├── OrderServiceLBConfig.java
  │   │   └── DefaultLBConfig.java
  │   ├── supplier/
  │   │   ├── GraySupplier.java
  │   │   └── VersionSupplier.java
  │   ├── algorithm/
  │   │   ├── ConsistentHashBalancer.java
  │   │   └── WeightedBalancer.java
  │   └── lifecycle/
  │       └── MetricsLifecycle.java
  └── LoadBalancerGlobalConfig.java   ← 汇总 @LoadBalancerClients
```

关键规则：

1. `@LoadBalancerClient(configuration=...)` 引用的配置类**不能被 `@ComponentScan` 扫描到**
2. 将这些配置类放在独立的包中，不与主配置类同包
3. `@LoadBalancerClients(defaultConfiguration=...)` 可以指定全局默认配置

# 6. 动态配置更新

## 6.1 运行时切换负载均衡策略

```java
@Component
public class DynamicLoadBalancerManager {

    private final LoadBalancerClientFactory factory;
    private final Map<String, AtomicReference<ReactorLoadBalancer<ServiceInstance>>>
            customBalancers = new ConcurrentHashMap<>();

    // 监听 Nacos 配置变更，动态替换策略
    @NacosConfigListener(dataId = "loadbalancer-strategy")
    public void onConfigChange(String config) {
        JSONObject json = JSON.parseObject(config);
        String serviceId = json.getString("serviceId");
        String strategy = json.getString("strategy");

        ReactorLoadBalancer<ServiceInstance> newBalancer = createBalancer(serviceId, strategy);
        customBalancers.computeIfAbsent(serviceId,
                k -> new AtomicReference<>()).set(newBalancer);
    }

    private ReactorLoadBalancer<ServiceInstance> createBalancer(
            String serviceId, String strategy) {
        ObjectProvider<ServiceInstanceListSupplier> provider =
                factory.getLazyProvider(serviceId, ServiceInstanceListSupplier.class);

        switch (strategy) {
            case "random": return new RandomLoadBalancer(provider, serviceId);
            case "roundrobin": return new RoundRobinLoadBalancer(provider, serviceId);
            default: throw new IllegalArgumentException("Unknown strategy: " + strategy);
        }
    }
}
```

## 6.2 动态调整权重

```java
@RestController
@RequestMapping("/admin/lb")
public class LoadBalancerAdminController {

    @Autowired
    private NacosNamingService nacosNamingService;

    @PostMapping("/{serviceName}/{ip}/{port}/weight")
    public String updateWeight(
            @PathVariable String serviceName,
            @PathVariable String ip,
            @PathVariable int port,
            @RequestParam double weight) throws NacosException {
        nacosNamingService.updateInstance(serviceName, ip, port, weight);
        return "ok";
    }

    @PostMapping("/{serviceName}/{ip}/{port}/enable")
    public String enable(@PathVariable String serviceName,
            @PathVariable String ip,
            @PathVariable int port,
            @RequestParam boolean enabled) throws NacosException {
        nacosNamingService.updateInstance(serviceName, ip, port, enabled);
        return "ok";
    }
}
```

# 7. 条件化配置

```java
@Configuration
public class EnvironmentAwareLoadBalancerConfig {

    @Bean
    @ConditionalOnProperty(name = "app.loadbalancer.strategy", havingValue = "random")
    public ReactorLoadBalancer<ServiceInstance> randomBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }

    @Bean
    @ConditionalOnMissingBean(ReactorLoadBalancer.class)
    public ReactorLoadBalancer<ServiceInstance> defaultBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

# 8. 扩展点总结

| 扩展点 | 接口/类 | 说明 |
|--------|---------|------|
| 负载均衡算法 | `ReactorLoadBalancer` / `ReactorServiceInstanceLoadBalancer` | 自定义选择策略 |
| 实例供应器 | `ServiceInstanceListSupplier` | 服务发现 + 过滤 + 缓存链 |
| 生命周期回调 | `LoadBalancerLifecycle` | 选择前后钩子（监控/日志） |
| 请求上下文 | `Request<T>` / `RequestDataContext` | 传递请求级别 Hint |
| URI 重建 | `LoadBalancerUriTools` | 自定义 URI 拼接逻辑 |
| 配置隔离 | `@LoadBalancerClient` / `@LoadBalancerClients` | 服务级配置 |
