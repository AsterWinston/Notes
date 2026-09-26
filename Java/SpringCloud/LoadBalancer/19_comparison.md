
# 1. 架构对比

## 1.1 Ribbon 架构

```
┌─────────────────────────────────────────────────┐
│                    Ribbon                        │
│                                                  │
│  ILoadBalancer ←── 核心接口                     │
│       │                                         │
│       ├── BaseLoadBalancer                      │
│       │     ├── ServerList ← 服务列表           │
│       │     ├── IRule ← 负载均衡规则            │
│       │     ├── IPing ← 健康检查                │
│       │     └── ServerListFilter ← 过滤         │
│       │                                         │
│       └── ZoneAwareLoadBalancer                 │
│             └── 多区域感知                      │
└─────────────────────────────────────────────────┘
```

## 1.2 LoadBalancer 架构

```
┌─────────────────────────────────────────────────┐
│              Spring Cloud LoadBalancer           │
│                                                  │
│  ReactorLoadBalancer<T> ←── 核心接口（响应式）  │
│       │                                         │
│       ├── ServiceInstanceListSupplier ← 实例供应│
│       │     ├── DiscoveryClient 集成            │
│       │     ├── Caching 装饰器                  │
│       │     ├── ZonePreference 装饰器           │
│       │     └── HealthCheck 装饰器              │
│       │                                         │
│       ├── LoadBalancerClientFactory              │
│       │     └── 服务级子容器隔离                │
│       │                                         │
│       └── LoadBalancerLifecycle ← 生命周期钩子  │
└─────────────────────────────────────────────────┘
```

# 2. 功能对比

| 功能 | Ribbon | Spring Cloud LoadBalancer |
|------|--------|---------------------------|
| **负载均衡算法** | 轮询/随机/加权/可用性过滤/区域感知等 | 轮询/随机（内置），自定义扩展 |
| **响应式支持** | 不支持 | 原生 Reactor 支持 |
| **配置方式** | `ribbon.*` 属性 + `@RibbonClient` | `spring.cloud.loadbalancer.*` + `@LoadBalancerClient` |
| **服务发现集成** | 自动集成 Eureka | 通过 `DiscoveryClient` 抽象集成所有注册中心 |
| **缓存** | ServerList 缓存 | `CachingServiceInstanceListSupplier` |
| **健康检查** | `IPing` 接口 | `HealthCheckServiceInstanceListSupplier` |
| **Zone 感知** | `ZoneAwareLoadBalancer` | `ZonePreferenceServiceInstanceListSupplier` |
| **重试** | 内置（`RetryHandler`） | Spring Retry 集成 |
| **服务级配置** | `@RibbonClient(name="xxx")` | `@LoadBalancerClient(name="xxx")` |
| **生命周期** | 不开放 | `LoadBalancerLifecycle` 钩子 |
| **静态服务器列表** | `listOfServers` 属性 | `SimpleDiscoveryClient` |
| **粘滞会话** | 内置 `StickySession` | `SameInstancePreferenceServiceInstanceListSupplier` |

# 3. 性能对比

| 维度 | Ribbon | LoadBalancer |
|------|--------|-------------|
| **线程模型** | 阻塞（每个请求占用线程） | 响应式（事件循环） |
| **并发能力** | 受限于线程池大小 | 受限于 CPU & 内存 |
| **连接开销** | 高（线程切换） | 低（无阻塞） |
| **吞吐量（估算）** | ~5k QPS/实例 | ~20k+ QPS/实例 |
| **GC 压力** | 较高（线程栈内存） | 较低 |

# 4. 配置迁移对照

## 4.1 负载均衡策略

```java
// Ribbon
// application.yml
user-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule

// 或代码
@RibbonClient(name = "user-service", configuration = RandomRuleConfig.class)
public class RandomRuleConfig {
    @Bean
    public IRule ribbonRule() {
        return new RandomRule();
    }
}

// ─── 迁移到 ───

// Spring Cloud LoadBalancer
@LoadBalancerClient(name = "user-service", configuration = RandomLBConfig.class)
public class RandomLBConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

## 4.2 超时配置

```
Ribbon:
  ribbon.ConnectTimeout=3000
  ribbon.ReadTimeout=5000

LoadBalancer:
  spring.cloud.loadbalancer.rest.connections.connect-timeout=3000
  spring.cloud.loadbalancer.rest.connections.read-timeout=5000
  （或在 RestTemplate/WebClient 中直接配置）
```

## 4.3 重试配置

```yaml
# Ribbon
ribbon:
  MaxAutoRetries: 1
  MaxAutoRetriesNextServer: 2
  OkToRetryOnAllOperations: false

# LoadBalancer
spring:
  cloud:
    loadbalancer:
      retry:
        max-retries-on-same-service-instance: 1
        max-retries-on-next-service-instance: 2
        retry-on-all-operations: false
```

# 5. 依赖迁移

```xml
<!-- Ribbon（旧） -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>

<!-- LoadBalancer（新） -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<!-- 常见迁移：OpenFeign 中 Ribbon 已被 LB 替代 -->
<!-- spring-cloud-starter-openfeign 已自动包含 spring-cloud-starter-loadbalancer -->
```

# 6. 代码迁移

## 6.1 RestTemplate 用法

```java
// Ribbon 时代 / LoadBalancer 时代（声明方式不变）
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 使用方式完全相同
restTemplate.getForObject("http://user-service/api/users/1", User.class);
```

## 6.2 自定义负载均衡规则

```java
// Ribbon
public class MyRibbonRule extends AbstractLoadBalancerRule {
    @Override
    public Server choose(Object key) {
        // 自定义逻辑
        return ...;
    }
}

// LoadBalancer
public class MyLoadBalancer implements ReactorServiceInstanceLoadBalancer {
    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // 自定义逻辑
        return ...;
    }
}
```

# 7. 迁移检查清单

| 检查项 | 操作 |
|--------|------|
| **移除 Ribbon 依赖** | 排除 `spring-cloud-starter-netflix-ribbon` |
| **更新 Spring Cloud 版本** | 升级到 2020.0.x 或更高 |
| **更新配置前缀** | `ribbon.*` → `spring.cloud.loadbalancer.*` |
| **替换 @RibbonClient** | 改用 `@LoadBalancerClient` / `@LoadBalancerClients` |
| **替换自定义 IRule** | 实现 `ReactorLoadBalancer<ServiceInstance>` |
| **替换自定义 IPing** | 使用 `HealthCheckServiceInstanceListSupplier` |
| **替换 ServerListFilter** | 使用自定义 `ServiceInstanceListSupplier` 装饰器 |
| **关闭 Feign 重试** | `feign.client.config.default.retryer=NEVER_RETRY` |
| **开启 LB 重试** | `spring.cloud.loadbalancer.retry.enabled=true`（引入 spring-retry） |
| **更新 RestTemplate 超时** | 通过 `RestTemplateBuilder` 配置 |
| **测试验证** | 灰度验证负载均衡行为 |

# 8. 兼容模式

Spring Cloud Hoxton.SR1 ~ Hoxton.SR12 同时支持 Ribbon 和 LoadBalancer：

```yaml
# 明确使用 LoadBalancer（关闭 Ribbon）
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false    # 关闭 Ribbon
```

Spring Cloud 2020.0.x 起，Ribbon 已经从依赖中移除：

```xml
<!-- 如果仍想使用 Ribbon（不推荐），需手动添加 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

# 9. LoadBalancer 不足与弥补

| 不足 | 原因 | 弥补方式 |
|------|------|---------|
| **内置算法少** | 仅 RoundRobin / Random | 自行实现 `ReactorLoadBalancer` |
| **无权重原生支持** | 需依赖注册中心 | 使用 NacosLoadBalancer |
| **无连接数感知** | 不跟踪连接数 | 自行实现 `LoadBalancerLifecycle` |
| **无请求延迟感知** | 不跟踪实例 RT | 自行记录 Metrics + 自定义算法 |
| **静态列表仅开发用** | 不支持生产静态配置 | 通过 `SimpleDiscoveryClient` 实现 |
