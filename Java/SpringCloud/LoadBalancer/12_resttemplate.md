
# 1. 概述

Spring Cloud LoadBalancer 通过 `@LoadBalanced` 注解和 `LoadBalancerInterceptor` 拦截 `RestTemplate` 的请求，自动将服务名替换为实际实例地址。

# 2. @LoadBalanced 注解原理

## 2.1 标记与收集

```java
// ① 声明负载均衡 RestTemplate
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

## 2.2 自动配置收集

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
public class LoadBalancerAutoConfiguration {

    // 收集所有标注 @LoadBalanced 的 RestTemplate
    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

    // 收集所有标注 @LoadBalanced 的 WebClient.Builder
    @LoadBalanced
    @Autowired(required = false)
    private List<WebClient.Builder> webClientBuilders = Collections.emptyList();

    // 为每个 RestTemplate 添加拦截器
    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializer(
            ObjectProvider<List<RestTemplateCustomizer>> customizers) {
        return () -> {
            for (RestTemplate restTemplate : restTemplates) {
                for (RestTemplateCustomizer customizer : customizers.getIfAvailable()) {
                    customizer.customize(restTemplate);
                }
            }
        };
    }

    // 注入 LoadBalancerInterceptor
    @Bean
    @ConditionalOnMissingBean
    public RestTemplateCustomizer loadBalancerRestTemplateCustomizer(
            LoadBalancerInterceptor interceptor) {
        return restTemplate -> {
            List<ClientHttpRequestInterceptor> interceptors = new ArrayList<>(
                    restTemplate.getInterceptors());
            interceptors.add(interceptor);
            restTemplate.setInterceptors(interceptors);
        };
    }
}
```

# 3. LoadBalancerInterceptor 源码

```java
public class LoadBalancerInterceptor
        implements ClientHttpRequestInterceptor {

    private final LoadBalancerClient loadBalancerClient;
    private final LoadBalancerRequestFactory requestFactory;

    @Override
    public ClientHttpResponse intercept(
            HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        URI originalUri = request.getURI();
        String serviceId = originalUri.getHost();

        // 使用 LoadBalancerClient 选择实例并执行请求
        return this.loadBalancerClient.execute(serviceId,
                this.requestFactory.createRequest(request, body, execution));
    }
}
```

## 3.1 LoadBalancerClient.execute() 流程

```java
public class BlockingLoadBalancerClient implements LoadBalancerClient {

    @Override
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
            throws IOException {

        // ① 选择实例
        ServiceInstance instance = choose(serviceId);

        if (instance == null) {
            throw new IllegalStateException(
                    "No instances available for " + serviceId);
        }

        try {
            // ② 用真实地址执行请求
            return request.apply(instance);
        } catch (IOException e) {
            throw e;
        } catch (Exception e) {
            throw new IOException(e);
        }
    }
}
```

## 3.2 RestTemplate 请求流程

```
restTemplate.getForObject("http://user-service/api/users/1", ...)
    │
    ▼
InterceptingClientHttpRequest.execute()
    │
    ▼ 触发拦截器链
LoadBalancerInterceptor.intercept(request, body, execution)
    │
    ├── 提取 serviceId = "user-service"
    │
    ├── loadBalancerClient.choose("user-service")
    │     │
    │     ├── ReactorLoadBalancer.choose().block()
    │     └── → ServiceInstance{host: 192.168.1.10, port: 8080}
    │
    ├── 重建 URI: http://192.168.1.10:8080/api/users/1
    │
    ├── execution.execute(newRequest, body)  ← 实际 HTTP 调用
    │
    └── 返回 ClientHttpResponse
```

# 4. 拦截器位置

```java
// LoadBalancerInterceptor 被添加到 RestTemplate 的拦截器列表末尾
// 建议它的 order 值较大，以便在其他拦截器之后执行

// RestTemplate 拦截器执行顺序：
// ① 自定义日志拦截器 (order = 1)
// ② 自定义认证拦截器 (order = 2)
// ③ LoadBalancerInterceptor (order = 999)  ← URI 替换发生在这里
// ④ 实际 HTTP 请求
```

# 5. URI 重建机制

```java
public class LoadBalancerUriTools {

    public static URI reconstructURI(
            ServiceInstance instance, URI original) {
        // 原 URI: http://user-service/api/users/1
        // 实例:   ServiceInstance{host: 192.168.1.10, port: 8080}
        // 结果:   http://192.168.1.10:8080/api/users/1

        return UriComponentsBuilder.fromUri(original)
                .host(instance.getHost())
                .port(instance.getPort())
                .build()
                .toUri();
    }

    // HTTPS 支持
    public static URI reconstructURI(ServiceInstance instance, URI original,
            boolean secure) {
        String scheme = secure ? "https" : original.getScheme();
        return UriComponentsBuilder.fromUri(original)
                .scheme(scheme)
                .host(instance.getHost())
                .port(instance.getPort())
                .build()
                .toUri();
    }
}
```

# 6. 非负载均衡的 RestTemplate

```java
@Configuration
public class RestTemplateConfig {

    // LB RestTemplate：用于微服务间调用
    @Bean
    @LoadBalanced
    public RestTemplate loadBalancedRestTemplate() {
        return new RestTemplate();
    }

    // 普通 RestTemplate：用于外部 API 调用
    @Bean
    public RestTemplate plainRestTemplate() {
        return new RestTemplateBuilder()
                .setConnectTimeout(Duration.ofSeconds(5))
                .setReadTimeout(Duration.ofSeconds(10))
                .build();
    }
}

// 使用
@Service
public class OrderService {

    @Autowired
    private RestTemplate loadBalancedRestTemplate;  // 微服务间调用

    @Autowired
    @Qualifier("plainRestTemplate")
    private RestTemplate plainRestTemplate;          // 外部 API 调用

    public void callUserService() {
        // 自动负载均衡
        String result = loadBalancedRestTemplate
                .getForObject("http://user-service/api/users/1", String.class);
    }

    public void callExternalApi() {
        // 直接 HTTP 调用
        String result = plainRestTemplate
                .getForObject("https://api.external.com/data", String.class);
    }
}
```

# 7. 连接池配置

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
            .setConnectTimeout(Duration.ofSeconds(3))
            .setReadTimeout(Duration.ofSeconds(5))
            .build();
}
```

```yaml
# 底层 HTTP 客户端连接池（推荐 Apache HttpClient）
spring:
  cloud:
    loadbalancer:
      resttemplate:
        connections:
          max-total: 200          # 最大连接数
          max-per-route: 50       # 每个路由最大连接数
          ttl: 30s                # 连接存活时间
```

# 8. LoadBalancerRequestFactory

```java
public class LoadBalancerRequestFactory {

    private final LoadBalancerClient loadBalancerClient;

    public LoadBalancerRequest<ClientHttpResponse> createRequest(
            HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) {

        // 返回一个 LoadBalancerRequest，apply() 被调用时用实际实例地址执行请求
        return instance -> {
            HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance);
            return execution.execute(serviceRequest, body);
        };
    }
}

// ServiceRequestWrapper：将 URI 中的 host:port 替换为实际实例
public class ServiceRequestWrapper extends HttpRequestWrapper {

    private final ServiceInstance instance;

    @Override
    public URI getURI() {
        return LoadBalancerUriTools.reconstructURI(
                this.instance,
                getRequest().getURI());
    }
}
```

# 9. 线程安全考虑

```java
// RestTemplate 是线程安全的（通常在单例中共享）
// @LoadBalanced 不会改变这一特性

// 但注意：不要在 RestTemplate 的拦截器中使用 ThreadLocal
// LoadBalancerInterceptor 是无状态的，可以安全共享
```

# 10. 迁移指南：从 Ribbon 到 LoadBalancer

```java
// 旧 Ribbon 配置
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
// 配套：
// ribbon.ConnectTimeout=3000
// ribbon.ReadTimeout=5000
// user-service.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule

// 新 LoadBalancer 配置（RestTemplate Bean 声明不变）
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
// 配套：
// spring.cloud.loadbalancer.configurations.user-service.strategy=random
```
