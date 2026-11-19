# 1. 概述

## 1.1 服务名 → 实例地址的翻译官

Spring Cloud OpenFeign 默认集成了 **Spring Cloud LoadBalancer**，可以将 `@FeignClient` 中的服务名（`name`）自动解析为实际的服务实例地址，并实现客户端负载均衡。

```text
服务名 (user-service)
    │
    │  Spring Cloud LoadBalancer
    ▼
实例列表 (192.168.1.10:8080, 192.168.1.11:8080, 192.168.1.12:8080)
    │
    │  负载均衡算法
    ▼
选定实例 (192.168.1.11:8080)
```

## 1.2 它在 Feign 管线中的位置

LoadBalancer 不挂在拦截器或 Contract 层，而是**包在 `Client` 外面**（§7 的装饰器）：前面所有环节拼出的 URL host 就是服务名，直到 `client.execute` 才被替换成真实实例地址。这个位置决定了两个性质：选址发生在**每次调用**（不是构建期固化，实例列表热更新因此无需重建代理）；它对重试友好——`Retryer` 触发的下一次尝试会重新走一遍 `execute`，天然可能选中另一个实例（§9 最后一行的机制来源）。

# 2. 版本演进：Ribbon → Spring Cloud LoadBalancer

| | Ribbon | Spring Cloud LoadBalancer |
|------|-------|--------------------------|
| **状态** | 已进入维护模式 | 官方推荐 |
| **Spring Cloud 版本** | Hoxton 及之前 | Hoxton.SR1+ (2020.0+) |
| **响应式支持** | 不支持 | 支持 WebFlux |
| **缓存** | ServerList 缓存 | CachingServiceInstanceListSupplier |
| **API 风格** | 命令式 | 响应式 + 阻塞桥接 |

**迁移注意**（2020.0.x 起）：
- `spring-cloud-starter-netflix-ribbon` → 移除
- `spring-cloud-starter-loadbalancer` → 自动引入（`spring-cloud-starter-openfeign` 已包含）
- `ribbon.xxx` 配置 → `spring.cloud.loadbalancer.xxx`

迁移中最容易丢的是行为默认值：Ribbon 时代"连接失败自动换实例"由 `MaxAutoRetriesNextServer=1` 默认兜着，LoadBalancer 的重试**默认关闭**且需要 spring-retry 在 classpath（§3 的 `retry.enabled`）。升级后"故障实例还在轮询里挨个踩一遍"的投诉，多数就出在这条默认值差异上。

# 3. 核心配置

```yaml
spring:
  cloud:
    loadbalancer:
      # 是否启用
      enabled: true
      # 服务发现集成
      discovery:
        enabled: true
      # 缓存配置
      cache:
        enabled: true
        ttl: 35s          # 实例列表缓存 TTL
        capacity: 256     # 缓存容量
      # 重试配置（需引入 spring-retry）
      retry:
        enabled: false
      # Zone 区域感知
      zone: zone1
```

`retry.enabled` 的键族展开是 `retry.max-attempts`（总尝试，含首次）与 `retry.max-same-service-attempts`/`max-excluded-instances`，语义细节在 §10 排障处换算；`zone` 配的是**本客户端所在区域**，配合实例 metadata `zone` 做同区优先（§9 的 ZonePreference 装饰层），Nacos 环境更常用权重路由（§4.3）而非这个通用键。

# 4. 负载均衡算法

## 4.1 默认：RoundRobin（轮询）

```java
// 默认使用 RoundRobinLoadBalancer，无需额外配置
```

轮询游标是 `AtomicInteger` 取模，实例列表来自**缓存快照**（§9）——列表刷新瞬间游标与新列表的对应关系会重排，表现为短暂的分布不均，属预期现象；想要"同用户固定实例"的会话亲和，轮询给不了，需要自定义策略（§5）或网关层解决。

## 4.2 切换到随机算法

```java
@Configuration
public class LoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name
        );
    }
}
```

## 4.3 切换到 Nacos 加权轮询

```java
@Bean
public ReactorLoadBalancer<ServiceInstance> nacosWeightedLoadBalancer(
        Environment environment,
        LoadBalancerClientFactory loadBalancerClientFactory) {
    String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    return new NacosLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name
    );
}
```

## 4.4 配置类的注册方式与作用域

§4.2/4.3 这类"每服务一套策略"的配置类**不能加 `@Component`**（否则全局生效，同 `4_configuration.md` §5.1 的污染逻辑），正确入口有二：`@LoadBalancerClients(defaultConfiguration = ...)` 全局，或 `@LoadBalancerClient(name = "user-service", configuration = ...)` 按服务（§6）。类里从 `PROPERTY_NAME` 取当前服务名的写法保证了同一配置类可以被多个服务复用而各自独立实例化——每个服务有自己的一份 `ReactorLoadBalancer` 子上下文。

# 5. 自定义负载均衡策略

```java
@Configuration
public class GrayLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> grayLoadBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new GrayRoundRobinLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                name
        );
    }
}

public class GrayRoundRobinLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;
    private final String serviceId;
    private final AtomicInteger position = new AtomicInteger(0);

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = supplierProvider.getIfAvailable();

        // 获取请求上下文中的灰度标记
        RequestDataContext context = (RequestDataContext) request.getContext();
        HttpHeaders headers = context.getClientRequest().getHeaders();
        boolean gray = Boolean.parseBoolean(headers.getFirst("X-Gray"));

        return supplier.get(request).next().map(instances -> {
            // 根据灰度标记过滤实例
            List<ServiceInstance> filtered = instances.stream()
                    .filter(i -> gray == isGrayInstance(i))
                    .collect(Collectors.toList());

            if (filtered.isEmpty()) {
                return new EmptyResponse();
            }

            // 轮询
            int pos = Math.abs(position.incrementAndGet()) % filtered.size();
            return new DefaultResponse(filtered.get(pos));
        });
    }

    private boolean isGrayInstance(ServiceInstance instance) {
        String gray = instance.getMetadata().get("gray");
        return Boolean.parseBoolean(gray);
    }
}
```

三个工程要点：`X-Gray` 头必须由调用方先种下（`7_interceptor.md` 的灰度拦截器从 ThreadLocal/上下文读标记写头——两个组件是配套关系，单独上线任何一个都不生效）；`choose` 的返回是 `Mono`，同步 Feign 路径上 `FeignBlockingLoadBalancerClient` 会阻塞等待，**supplier 卡住会直接吊住业务线程**，自定义实现里不要引入远程调用；`EmptyResponse` 时上游得到"没有可用实例"异常而非 5xx——灰度组实例数为 0 的发布窗口，这正是预期行为，但要确保告警把它和真故障区分开。

# 6. @LoadBalancerClient 注解

```java
@Configuration
@LoadBalancerClient(
    name = "user-service",
    configuration = UserServiceLoadBalancerConfig.class
)
public class LoadBalancerAutoConfig { }
```

声明某个服务的**独立负载均衡策略**，不与其他服务共享。批量声明用 `@LoadBalancerClients`（数组包一层）；`name` 写服务名，`contextId` 属性可给同名多客户端场景指定更细的作用域（与 Feign 侧 `4_configuration.md` §7 的 contextId 语义对齐）。

# 7. FeignBlockingLoadBalancerClient 源码解析

```java
public class FeignBlockingLoadBalancerClient implements Client {

    private final Client delegate;          // 底层 HTTP 引擎
    private final LoadBalancerClient loadBalancerClient;

    @Override
    public Response execute(Request request, Request.Options options) throws IOException {
        // 1. 提取 serviceId（从 URL host 部分）
        URI uri = URI.create(request.url());
        String serviceId = uri.getHost();

        // 2. 通过 LoadBalancer 选择实例
        ServiceInstance instance = loadBalancerClient.choose(serviceId);

        // 3. 重建 URL（替换 host:port）
        String newUrl = request.url().replace(
                uri.getHost() + ":" + uri.getPort(),
                instance.getHost() + ":" + instance.getPort()
        );

        // 4. 委托给真正的 HTTP 客户端执行
        Request newRequest = Request.create(..., newUrl, ...);
        return delegate.execute(newRequest, options);
    }
}
```

**关键流程**：
1. `request.url()` → 提取 host 作为 serviceId
2. `loadBalancerClient.choose(serviceId)` → 选一个实例
3. 将 URL 中的 host:port 替换为实际实例地址
4. 委托 `delegate`（如 `ApacheHttpClient`）执行

（还原度说明：真实实现还会把请求包装成 `RequestDataContext` 传给 `choose`——这正是 §5 里能从 `request.getContext()` 读到灰度头的通道；URL 改写也是对原 RequestTemplate 的 target 替换而非字符串 replace。）流程里埋着两条高频故障的根因：**第 2 步返回 null** → `IllegalStateException: No instances available for user-service`，去查注册中心而不是查 Feign；**第 1 步的 host 即服务名**，所以 `url =` 直连写死后这里的 serviceId 变成真实域名，负载均衡自然失效——与 `4_configuration.md` §7.2 是同一件事的两面。

# 8. 不经过负载均衡的调用

3 种方式直连指定 URL：

**方式 1：@FeignClient 指定 url**

```java
@FeignClient(name = "external-api", url = "https://api.example.com")
public interface ExternalApiClient { }
```

**方式 2：Feign.Builder 直接 target URL**

```java
UserClient client = Feign.builder()
        .target(UserClient.class, "http://192.168.1.100:8080");
```

**方式 3：不引入 spring-cloud-starter-loadbalancer（或排除依赖）**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

方式 3 影响的是**所有**用服务名的客户端，排除后它们全部退化成"host 当域名解析"（UnknownHost 或错误直连）；只为个别第三方 API 直连时，方式 1 是正确答案。另有一个半直连中间态：`url` 支持 SpEL/占位符（`url = "${user-service.url:}"`），配置了走直连、留空回落服务名——测试环境与生产共用一套代码的常用开关，比双份接口优雅。

# 9. LoadBalancer 生命周期

```text
首次请求某个服务
    │
    ├── ServiceInstanceListSupplier 获取实例列表
    │       │
    │       ├── DiscoveryClientServiceInstanceListSupplier (从注册中心获取)
    │       ├── CachingServiceInstanceListSupplier (缓存 N 秒)
    │       └── ZonePreferenceServiceInstanceListSupplier (同 Zone 优先)
    │
    ├── ReactorLoadBalancer.choose() 选择实例
    │
    ├── 请求执行
    │
    └── 失败时可能触发重试（选择下一个实例）
```

supplier 是**装饰器链**：从内到外通常是 发现源 → 缓存 → zone 偏好（→ 自定义过滤），`choose` 拿到的永远是全链加工后的视图。这带来一个实用结论：想加"实例级过滤"（只调 v2 分组、排除维护标记实例），正确做法是再包一层自定义 `ServiceInstanceListSupplier` 而不是改 LoadBalancer 算法——过滤发生在选址之前，语义上也更贴切（"候选集"问题 vs "挑选"问题）。注册中心推送（Nacos）与缓存 TTL（35s）叠加后，实例上下线的端到端生效时间 ≈ 注册中心传播 + 缓存剩余寿命，容量与发布窗口规划按最坏值算。

# 10. 常见问题

**Q：服务下线后还能调通？**

实例列表缓存 TTL 默认 35s，在缓存失效前仍可能路由到已下线的实例。配合重试机制可缓解。

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 5s  # 缩短缓存时间
      retry:
        enabled: true
```

缓解链路要说完整：打到死实例 → 连接失败（IOException 类）→ 重试在**排除已试实例**后换一台。`retry.enabled=true` 要求 classpath 有 spring-retry（`spring-retry` 依赖默认不随 starter 带入，这是"配了没反应"的第一嫌疑）。总尝试换算见下一问。

**Q：三层重试的放大系数怎么算？**

```text
下游实际收到的请求量 ≈ Feign Retryer 次数 × LoadBalancer retry.max-attempts × 上游熔断层 Retry 次数
```

三层都有"换实例/重放"语义，全开时 3×2×2 = 一个入口请求最多 12 次打到下游。默认 LoadBalancer 重试关闭、Feign NEVER_RETRY，等于把放大权交给你显式配置——**只在最内层保留一层自动重试**（通常给连接类失败用 LoadBalancer 换实例，读超时交给业务语义决定），并保证每层次数都写进配置注释（`9_retry.md` §9.1 的预算不等式）。

**Q：如何指定客户端不经过 LoadBalancer？**

```java
@FeignClient(name = "user-service", url = "http://localhost:8080")
```

或设置：

```yaml
spring:
  cloud:
    loadbalancer:
      enabled: false  # 全局禁用
```

全局禁用开关是排障工具不是架构选项：禁用后所有服务名调用退化（§8 方式 3 的后果同形），生产环境按客户端用 `url` 解决个体需求即可。
