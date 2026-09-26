
# 1. 常见问题速查

| 现象 | 可能原因 | 检查项 |
|------|---------|--------|
| No instances available | 注册中心无实例 | 检查提供者是否注册成功 |
| 路由到下线实例 | 缓存未过期 | 缩短 TTL / 清除缓存 |
| 负载均衡不生效 | 未加 @LoadBalanced | 检查 Bean 声明 |
| 所有请求打到一个实例 | RoundRobin position 溢出 | 检查实例是否都在同一个 Zone |
| 频繁超时 | 实例性能不足 | 检查下游 RT / 调整超时 |
| Feign 调用失败 | LoadBalancer 未启用 | 检查依赖是否完整 |
| WebClient 不生效 | 未用 @LoadBalanced Builder | 注入 WebClient.Builder |
| 内存增长 | 子容器不释放 | 检查服务数量 / 缓存容量 |

# 2. 实例不可用排查

## 2.1 错误信息

```
java.lang.IllegalStateException: No instances available for user-service
```

## 2.2 排查步骤

```
① 检查提供者是否注册成功
   curl http://localhost:8848/nacos/v1/ns/instance/list?serviceName=user-service

② 检查消费者配置
   spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
   spring.cloud.loadbalancer.enabled=true

③ 检查缓存是否过期
   curl http://localhost:8080/actuator/loadbalancer-caches

④ 检查 HealthCheck 是否过滤了所有实例
   spring.cloud.loadbalancer.health-check.interval=25s
   手动检查 /actuator/health: curl http://192.168.1.10:8080/actuator/health

⑤ 检查 Zone 配置是否匹配
   spring.cloud.loadbalancer.zone=SH
   实例的 metadata.zone 是否与当前 Zone 一致
```

## 2.3 诊断端点

```bash
# 查看缓存状态
curl http://localhost:8080/actuator/loadbalancer-caches

# 清除指定服务缓存
curl -X DELETE http://localhost:8080/actuator/loadbalancer-caches/user-service

# 查看健康状态
curl http://localhost:8080/actuator/health
```

# 3. 负载均衡不生效

## 3.1 RestTemplate 未加 @LoadBalanced

```java
// ❌ 错误
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();           // 没有 @LoadBalanced
}

// ✅ 正确
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

## 3.2 WebClient 用法错误

```java
// ❌ 错误：直接 new WebClient
WebClient client = WebClient.create();
client.get().uri("http://user-service/api/users/1")...  // 不经过 LB

// ✅ 正确：注入带 @LoadBalanced 的 Builder
@Autowired
private WebClient.Builder webClientBuilder;

webClientBuilder.build()
    .get().uri("http://user-service/api/users/1")...
```

## 3.3 Feign 未指定 serviceId

```java
// ❌ 错误：指定 url 会绕过 LoadBalancer
@FeignClient(name = "user-service", url = "http://localhost:8081")

// ✅ 正确
@FeignClient(name = "user-service")
```

# 4. 缓存相关问题

## 4.1 路由到已下线实例

```yaml
# 原因：缓存 TTL 过长
spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 35s  # 下线后最多 35s 仍可能路由到已下线实例
```

**解决**：
1. 缩短 TTL：`ttl: 5s`
2. 配置健康检查过滤
3. 主动清除缓存：

```bash
curl -X DELETE http://localhost:8080/actuator/loadbalancer-caches/user-service
```

## 4.2 缓存不更新

```java
// 可能原因：registerHealthCheck 未配置
// 检查是否有新的实例注册但缓存未刷新

@Bean
public ServiceInstanceListSupplier supplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withCaching()   // TTL 到后才刷新
            .withHealthChecks()
            .build(context);
}
```

# 5. 配置冲突排查

## 5.1 子容器配置覆盖问题

```java
// 问题：@LoadBalancerClient 配置类被 @ComponentScan 扫描到
// 导致该配置对所有服务生效，而非指定服务

// ❌ 错误：放在主包下被扫描
package com.example.config;
@Configuration          // ← 不该加 @Configuration
public class UserServiceLBConfig { }

// ✅ 正确：独立包，不加 @Configuration
package com.example.lb.config;
// 不加 @ComponentScan 的关键注解
public class UserServiceLBConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> balancer(...) { ... }
}

// 由 @LoadBalancerClient 引用
@LoadBalancerClient(name = "user-service",
        configuration = UserServiceLBConfig.class)
```

## 5.2 Ribbon 残余依赖

```bash
# 检查是否还有 Ribbon 依赖
mvn dependency:tree | grep ribbon

# 如有，排除
<exclusion>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</exclusion>
```

# 6. 重试排查

## 6.1 重试不生效

```yaml
# 检查是否引入了 spring-retry
# 仅配置不会生效，需要显式添加依赖

# pom.xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>

# application.yml
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
```

## 6.2 POST 请求被重复执行

```yaml
# 默认不重试 POST/PUT/DELETE
# 如果开启了 retry-on-all-operations，可能导致重复创建资源

spring:
  cloud:
    loadbalancer:
      retry:
        retry-on-all-operations: false   # 确保是 false
```

# 7. 版本兼容性问题

| 问题 | 版本 | 解决 |
|------|------|------|
| `ClassNotFoundException: ReactorLoadBalancer` | Spring Cloud < 2020.0 | 升级到 2020.0.x+ |
| `NoClassDefFoundError: DiscoveryClient` | 缺少注册中心依赖 | 添加 Nacos/Eureka 依赖 |
| `NoSuchMethodError` | 版本混用 | 统一 Spring Cloud BOM |
| WebClient 注入失败 | Spring Boot < 2.4 | 升级 Spring Boot |
| Actuator endpoint 404 | Spring Boot 2.x | 配置 `management.endpoints.web.exposure.include` |

# 8. 日志排查

```yaml
logging:
  level:
    org.springframework.cloud.loadbalancer: DEBUG   # 核心日志
    org.springframework.cloud.client.discovery: DEBUG  # 服务发现日志
    com.alibaba.nacos.client: DEBUG                  # Nacos 客户端日志
```

关键日志输出：

```
# 服务发现
DEBUG o.s.c.d.DiscoveryClient - Getting instances for user-service
DEBUG o.s.c.d.DiscoveryClient - Found 3 instances

# 缓存
DEBUG o.s.c.l.cache.CachingServiceInstanceListSupplier - Cache miss for user-service
DEBUG o.s.c.l.cache.CachingServiceInstanceListSupplier - Caching 3 instances for user-service

# 选择
DEBUG o.s.c.l.core.RoundRobinLoadBalancer - Selected instance 192.168.1.10:8080

# 拦截
DEBUG o.s.c.l.interceptor.LoadBalancerInterceptor - Intercepted request to user-service
```

# 9. 终极排查方案

```java
@Component
public class LoadBalancerDebugger {

    @EventListener
    public void onServiceInstanceRegistered(
            @SuppressWarnings("rawtypes") InstanceRegisteredEvent event) {
        log.info("Instance registered: {} = {}",
                event.getSource().getClass().getSimpleName(),
                event.getSource());
    }

    @Bean
    public LoadBalancerLifecycle debugLifecycle() {
        return new LoadBalancerLifecycle<
                RequestDataContext, Object, ServiceInstance>() {

            @Override
            public void onStart(Request<RequestDataContext> request) {
                log.info("LB choose start: hints={}",
                        request.getHints());
            }

            @Override
            public void onComplete(
                    CompletionContext<Object, ServiceInstance,
                            RequestDataContext> context) {
                if (context.getStatus() == CompletionContext.Status.SUCCESS) {
                    ServiceInstance instance = context.getClientResponse()
                            .getServer();
                    log.info("LB choose success: {}:{}",
                            instance.getHost(), instance.getPort());
                } else {
                    log.error("LB choose failed", context.getThrowable());
                }
            }
        };
    }
}
```
