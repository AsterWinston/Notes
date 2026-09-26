
# 1. 自动配置入口

```java
// spring.factories / org.springframework.boot.autoconfigure.AutoConfiguration.imports
org.springframework.cloud.loadbalancer.config.LoadBalancerAutoConfiguration
```

## 1.1 LoadBalancerAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerProperties.class)
public class LoadBalancerAutoConfiguration {

    // ① 收集所有 @LoadBalanced 标注的 RestTemplate
    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

    // ② 收集所有 @LoadBalanced 标注的 WebClient.Builder
    @LoadBalanced
    @Autowired(required = false)
    private List<WebClient.Builder> webClientBuilders = Collections.emptyList();

    // ③ 智能初始化：在所有 Bean 初始化后定制 RestTemplate/WebClient
    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
            ObjectProvider<List<RestTemplateCustomizer>> customizers) {
        return () -> restTemplates.forEach(restTemplate ->
                customizers.getIfAvailable().forEach(
                        customizer -> customizer.customize(restTemplate)));
    }

    @Bean
    public SmartInitializingSingleton loadBalancedWebClientInitializer(
            ObjectProvider<List<WebClientCustomizer>> customizers) {
        return () -> webClientBuilders.forEach(builder ->
                customizers.getIfAvailable().forEach(
                        customizer -> customizer.customize(builder)));
    }

    // ④ RestTemplate 定制器：注入 LoadBalancerInterceptor
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingClass(
            "org.springframework.retry.support.RetryTemplate")
    static class LoadBalancerInterceptorConfig {

        @Bean
        public LoadBalancerInterceptor loadBalancerInterceptor(
                LoadBalancerClient loadBalancerClient,
                LoadBalancerRequestFactory requestFactory) {
            return new LoadBalancerInterceptor(loadBalancerClient,
                    requestFactory);
        }

        @Bean
        @ConditionalOnMissingBean
        public RestTemplateCustomizer restTemplateCustomizer(
                LoadBalancerInterceptor interceptor) {
            return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(interceptor);
                restTemplate.setInterceptors(list);
            };
        }
    }

    // ⑤ WebClient 定制器：注入 ReactorLoadBalancerExchangeFilterFunction
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(name = "org.springframework.web.reactive.function.client.WebClient")
    static class LoadBalancerWebClientConfig {

        @Bean
        @ConditionalOnMissingBean
        public ReactorLoadBalancerExchangeFilterFunction
                reactorLoadBalancerExchangeFilterFunction(
                        LoadBalancerClientFactory clientFactory,
                        LoadBalancerProperties properties) {
            return new ReactorLoadBalancerExchangeFilterFunction(
                    clientFactory, properties);
        }

        @Bean
        @ConditionalOnMissingBean
        public WebClientCustomizer webClientCustomizer(
                ReactorLoadBalancerExchangeFilterFunction filterFunction) {
            return builder -> builder.filter(filterFunction);
        }
    }
}
```

# 2. LoadBalancerClient 自动配置

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(LoadBalancerClient.class)
public class BlockingLoadBalancerClientAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerClient blockingLoadBalancerClient(
            LoadBalancerClientFactory loadBalancerClientFactory) {
        // 创建阻塞式负载均衡客户端
        return new BlockingLoadBalancerClient(loadBalancerClientFactory);
    }

    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient);
    }
}
```

# 3. 默认客户端配置

```java
@Configuration(proxyBeanMethods = false)
public class LoadBalancerClientConfiguration {

    // ① 默认使用轮询
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

    // ② 默认 ServiceInstanceListSupplier 链
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnDiscoveryEnabled
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withCaching()
                .withZonePreference()
                .withHealthChecks()
                .build(context);
    }
}
```

# 4. NamedContextFactory 核心机制

```java
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
        implements DisposableBean, ApplicationContextAware {

    // 每个 name → 一个独立的 ApplicationContext 子容器
    private final Map<String, AnnotationConfigApplicationContext> contexts =
            new ConcurrentHashMap<>();

    private ApplicationContext parent;
    private final String propertyName;
    private final String propertySourceName;

    // 获取指定 name 的 Bean
    public <T> T getInstance(String name, Class<T> type) {
        AnnotationConfigApplicationContext context = getContext(name);
        try {
            return context.getBean(type);
        } catch (NoSuchBeanDefinitionException e) {
            return null;
        }
    }

    // 惰性获取（延迟创建子容器）
    public <T> ObjectProvider<T> getLazyProvider(String name, Class<T> type) {
        return new LazyProvider<>(() -> getInstance(name, type));
    }

    // 创建/获取子容器
    protected AnnotationConfigApplicationContext getContext(String name) {
        if (!this.contexts.containsKey(name)) {
            synchronized (this.contexts) {
                if (!this.contexts.containsKey(name)) {
                    this.contexts.put(name, createContext(name));
                }
            }
        }
        return this.contexts.get(name);
    }

    // 创建子容器
    protected AnnotationConfigApplicationContext createContext(String name) {
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext();
        context.setParent(this.parent);
        context.register(getConfigurations(name));
        context.refresh();

        // 设置子容器的 Environment 属性
        context.getEnvironment().getPropertySources()
                .addFirst(new MapPropertySource(this.propertySourceName,
                        Collections.singletonMap(this.propertyName, name)));

        return context;
    }
}
```

# 5. LoadBalancerClientFactory 继承结构

```
ApplicationContextAware
        │
        ▼
DisposableBean
        │
        ▼
NamedContextFactory<LoadBalancerClientSpecification>
        │  (继承)
        ▼
LoadBalancerClientFactory

关键参数:
  - NAMESPACE = "loadbalancer"
  - PROPERTY_NAME = "loadbalancer.client.name"
  - 默认配置类 = LoadBalancerClientConfiguration.class
  - 每个 serviceId 创建独立子容器
```

# 6. @LoadBalancerClient 注解处理

```java
public class LoadBalancerClientConfigurationRegistrar
        implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {

        // 解析 @LoadBalancerClient 注解
        Map<String, Object> attrs = metadata.getAnnotationAttributes(
                LoadBalancerClient.class.getName());

        String name = (String) attrs.get("name");
        Class<?>[] configurationClasses = (Class<?>[]) attrs.get("configuration");

        // 注册 LoadBalancerClientSpecification Bean
        // 每个 specification 包含 name → configuration 映射
        BeanDefinitionBuilder builder = BeanDefinitionBuilder
                .genericBeanDefinition(LoadBalancerClientSpecification.class);
        builder.addConstructorArgValue(name);
        builder.addConstructorArgValue(configurationClasses);
        registry.registerBeanDefinition(
                "loadBalancerClientSpecification." + name,
                builder.getBeanDefinition());
    }
}
```

# 7. ServiceInstanceListSupplier.Builder 链构建

```java
public static final class Builder {

    private final Creator creator;

    private Builder(ApplicationContext context) {
        this.creator = context::getBean;
    }

    // 使用基础 Supplier 包装
    public Builder withBase(
            Function<ServiceInstanceListSupplier,
                    ServiceInstanceListSupplier> baseCreator) {
        this.baseCreator = baseCreator;
        return this;
    }

    // 叠加 DiscoveryClient
    public Builder withDiscoveryClient() {
        this.discoveryClientSupplierCreator =
                context -> new DiscoveryClientServiceInstanceListSupplier(
                        context.getBean(DiscoveryClient.class),
                        context.getEnvironment());
        return this;
    }

    // 叠加缓存
    public Builder withCaching() {
        this.cachingCreator = delegate -> {
            LoadBalancerProperties props = context.getBean(...);
            return new CachingServiceInstanceListSupplier(
                    delegate,
                    props.getCache().getTtl(),
                    props.getCache().getCapacity());
        };
        return this;
    }

    // 叠加 Zone 优先
    public Builder withZonePreference() {
        this.zonePreferenceCreator = delegate -> {
            LoadBalancerProperties props = context.getBean(...);
            return new ZonePreferenceServiceInstanceListSupplier(
                    delegate, props.getZone());
        };
        return this;
    }

    // 叠加健康检查
    public Builder withHealthChecks() {
        this.healthCheckCreator = delegate ->
                new HealthCheckServiceInstanceListSupplier(
                        delegate, ...);
        return this;
    }

    // 构建最终链
    public ServiceInstanceListSupplier build(
            ConfigurableApplicationContext context) {
        ServiceInstanceListSupplier supplier = baseCreator.apply(null);
        // 逐层包装...
        return supplier;
    }
}
```

# 8. 自动配置条件注解

```java
// 服务发现是否启用
@ConditionalOnDiscoveryEnabled

// 检查 DiscoveryClient 是否可用
@ConditionalOnBean(DiscoveryClient.class)

// 检查 Spring Retry 是否可用
@ConditionalOnClass(RetryTemplate.class)

// 检查是否为 Feign 环境
@ConditionalOnClass(Feign.class)

// 检查是否为 Gateway 环境
@ConditionalOnClass(name = "org.springframework.cloud.gateway.filter.GlobalFilter")
```

# 9. 配置加载优先级

```
① spring.cloud.loadbalancer.configurations.{serviceId}.xxx  (服务级)
    ↓ 覆盖
② @LoadBalancerClient(name="xxx", configuration=...)          (注解级)
    ↓ 覆盖
③ @LoadBalancerClients(defaultConfiguration=...)              (全局级)
    ↓ 覆盖
④ LoadBalancerClientConfiguration                            (默认)
```
