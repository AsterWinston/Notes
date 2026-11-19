# 1. 概述

## 1.1 核心链路

Spring Cloud OpenFeign 与**服务注册中心**（Nacos、Eureka、Consul、Zookeeper）深度集成。核心链路：

```text
@FeignClient(name = "user-service")
    │
    │  1. 从注册中心获取实例列表
    ▼
DiscoveryClient (NacosDiscoveryClient / EurekaDiscoveryClient / ...)
    │
    │  2. 传递实例列表给 LoadBalancer
    ▼
ServiceInstanceListSupplier (DiscoveryClientServiceInstanceListSupplier)
    │
    │  3. LoadBalancer 选择实例
    ▼
FeignBlockingLoadBalancerClient
    │
    │  4. 发送 HTTP 到实际 IP:Port
    ▼
目标服务实例
```

## 1.2 一个容易建立错的模型：Feign 不认识注册中心

严格说，**Feign 核心代码里没有任何注册中心的痕迹**：它只负责把服务名拼进 URL host（`13_loadbalancer.md` §7 的 `execute` 里才做 host 替换），"发现"发生在 LoadBalancer 的 supplier 链里，"拉取实例列表"发生在 DiscoveryClient 里。这条 `Feign → LoadBalancer → DiscoveryClient → 注册中心 SDK` 的三层分工解释了三件事：

| 现象 | 归因层 |
|------|--------|
| Feign 报 `No instances available` | LoadBalancer 层（supplier 返回空列表） |
| 控制台能看到实例但本地拿不到 | DiscoveryClient/缓存层（§8 的缓存与订阅问题） |
| 拿到列表但实例地址不可达 | 注册层（provider 注册的 IP 就是错的，如多网卡/Docker NAT） |

排障时按层收敛，比从异常栈倒推快得多（§10 有完整决策树）。

# 2. Nacos 集成

## 2.1 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

## 2.2 配置

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        namespace: dev              # 命名空间（隔离环境）
        group: DEFAULT_GROUP        # 分组
        cluster-name: beijing       # 集群名（同集群优先调用）
        weight: 10                  # 权重（配合 NacosLoadBalancer）
        ephemeral: true             # 是否临时实例
        metadata:
          version: v2.0             # 自定义元数据（灰度/版本路由）
```

## 2.3 使用

```java
@FeignClient(name = "user-service")  // name = 注册中心中的服务名
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable Long id);
}
```

无需任何额外代码，Feign 自动通过 Nacos 发现 `user-service` 的实例。

## 2.4 每个配置项在"调用侧"的真实含义

这些键大多是**注册属性**，但每一条都会反过来影响别人怎么调到你，也要理解"你调别人时它怎么起作用"：

- `namespace`：服务发现的**硬隔离边界**——消费方与提供方必须在同一 namespace 才能互相发现。"本地调通、上 K8s 后 No instances available"的高频根因就是两边 namespace 不一致（且 Nacos 控制台默认只显示 public，看不到别的命名空间里的注册，误导排查）。
- `group`：同 namespace 内的二级分组，DiscoveryClient 查询默认按 `DEFAULT_GROUP`；提供方注册在自定义 group 时，消费方需要额外处理才能发现，团队约定**用 namespace 隔离环境、group 隔离业务域**，不要拿 group 当环境用。
- `cluster-name`：同集群优先路由在 Nacos 侧由 `NacosLoadBalancer`/权重策略消费 metadata `zone`/cluster 信息实现（§3 的 Eureka zone、`13_loadbalancer.md` §3 的 `spring.cloud.loadbalancer.zone` 是同一问题的两套落地）。
- `weight`：只有负载均衡器按权重选址时才有效——Spring Cloud LoadBalancer 默认轮询会**忽略权重**，必须显式换成 `NacosLoadBalancer`（`13_loadbalancer.md` §4.3 那段 Bean 就是为此存在）。"权重设了没用"先查这个。
- `ephemeral`：临时实例走客户端心跳（实例挂了 15s 标 unhealthy、30s 剔除），感知快；持久实例由服务端主动探测，生命周期由注册中心管理。Feign 调用链默认期望对端是临时实例语义。

# 3. Eureka 集成

## 3.1 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

## 3.2 配置

```yaml
spring:
  application:
    name: order-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    metadata-map:
      zone: zone1
      version: v2.0
```

## 3.3 Eureka 的三层缓存延迟模型

Eureka 的发现链路天生带三级延迟，这是它"下线感知 90s"口碑的来源（对比表见 §8）：提供方注销/心跳丢失要等 **server 端剔除任务（默认 30s 周期，且要过 90s 过期阈值）** → server 注册表变更要等 **responseCache 编译刷新（30s）** → 客户端要等 **registry fetch（默认 30s）**。三段串起来，最坏情况一个新调用方拿到"死亡实例列表"的时间以分钟计。两个直接后果：`prefer-ip-address: true` 必须开（否则走主机名解析，容器网络里经常解错）；故障演练里"杀掉 pod 后流量多久归零"在 Eureka 上的答案和 Nacos 完全不同，容量预案要按各自延迟模型写。客户端 registry 缓存与 LoadBalancer 的 35s supplier 缓存（§6）**还要再叠一层**——缩短哪一层都治标，真正的兜底是重试换实例（`13_loadbalancer.md` §10）。

# 4. Consul 集成

## 4.1 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

## 4.2 配置

```yaml
spring:
  application:
    name: order-service
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
        instance-id: ${spring.application.name}:${random.value}
        health-check-path: /actuator/health
        health-check-interval: 15s
        tags:
          - version=v2.0
```

## 4.3 Consul 模型的差异点

Consul 是**服务端主动健康检查**（agent 定期 HTTP 探活 `/actuator/health`），下线感知及时（§8 表）；代价是 `health-check-interval` 期间内一次探活失败就会把实例**摘除注册表**，而 Spring Boot 的 health 端点很敏感——DB 连接池抖动、磁盘水位、甚至某个 `HealthIndicator` 超时，都会让一个**业务上完全健康的实例被全局摘除**。生产上普遍做法是给它单独配一条只反映"本进程可用性"的检查路径（Boot 支持 `management.endpoint.health.group.liveness` 等分组，Consul 集成配置指向该组），别让依赖组件的复合健康度决定流量去向。`instance-id` 里的 `random.value` 解决同机多实例覆盖注册问题，K8s 里更常用 `${POD_NAME}` 保证重启后幂等。

# 5. DiscoveryClient 接口

```java
public interface DiscoveryClient {

    String description();

    // 获取所有服务实例
    List<ServiceInstance> getInstances(String serviceId);

    // 获取所有注册的服务名
    List<String> getServices();

    // 从注册中心获取服务实例（响应式，LoadBalancer 使用）
    default Flux<ServiceInstance> getInstancesReactive(String serviceId) {
        return Flux.fromIterable(getInstances(serviceId));
    }
}
```

各注册中心的实现：

| 注册中心 | DiscoveryClient 实现 |
|----------|---------------------|
| Nacos | `NacosDiscoveryClient` |
| Eureka | `EurekaDiscoveryClient` |
| Consul | `ConsulDiscoveryClient` |
| Zookeeper | `ZookeeperDiscoveryClient` |

两个接口方法的运维价值被低估：actuator 的 `/actuator/discoveryclient`（或用一段启动自检代码调 `getInstances("user-service")` 打日志）是区分"注册中心没有"还是"我这层没拿到"的最快手段——`getInstances` 返回空而控制台有实例，问题锁定在 namespace/订阅/缓存；两边都空，问题在提供方注册（网络、凭据、启动参数）。另外多数实现内部带自己的缓存（Nacos 全量推送订阅、Eureka fetchInterval），所以 `DiscoveryClient.getInstances` 拿到的本来就**不是实时视图**，而是"最近一次同步的快照"——§1.2 的排障表第一行、第二行的分界正在这层。

# 6. ServiceInstanceListSupplier 链

```text

Request
  │
  ▼
DiscoveryClientServiceInstanceListSupplier   (从注册中心获取实例)
  │
  ▼
CachingServiceInstanceListSupplier           (缓存实例列表)
  │
  ▼
ZonePreferenceServiceInstanceListSupplier    (同 Zone 过滤)
  │
  ▼
HealthCheckServiceInstanceListSupplier       (健康检查过滤，部分注册中心)
  │
  ▼
ReactorLoadBalancer.choose()
```

## 6.1 自定义 Supplier 链

```java
@Configuration
public class CustomSupplierConfig {

    @Bean
    public ServiceInstanceListSupplier discoveryClientSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withCaching()
                .withZonePreference()
                .withHealthChecks()
                .withHints()       // 基于 Hint 过滤（灰度场景）
                .build(context);
    }
}
```

## 6.2 链条语义与顺序敏感点

builder 的调用顺序就是装饰顺序，几个键的语义要精确理解：**缓存层的位置**决定 zone/健康过滤是在"缓存快照"还是"实时列表"上做——放在 withCaching 之后意味着过滤器每 35s 才随缓存刷新重算一次；`withHealthChecks()` 是**调用方主动探活**（对每个实例打配置的 health 路径），Nacos/Eureka 这类自带健康状态的注册中心不需要它（重复探测还放大流量），它服务于"注册中心不健康但进程活着"的裸 DiscoveryClient 场景；`withHints()` 让 `@LoadBalancerHint` 或请求上下文里的 hint 参与过滤，是 §7 灰度的轻量替代。此配置类同样**不能加 `@Component`** 全局扫（作用域规则见 `13_loadbalancer.md` §4.4）。

# 7. 灰度发布 / 多版本路由

通过注册中心的自定义元数据 + 自定义 LoadBalancer 实现：

```java
@Component
public class GrayRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 传入版本标记
        String version = getRequestVersion();
        if (version != null) {
            template.header("X-Service-Version", version);
        }
    }
}
```

```java
public class VersionBasedLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // 从 RequestDataContext 读取 Hint
        RequestDataContext ctx = (RequestDataContext) request.getContext();
        String version = ctx.getClientRequest().getHeaders()
                .getFirst("X-Service-Version");

        return supplier.get(request).next().map(instances -> {
            List<ServiceInstance> matched = instances.stream()
                    .filter(i -> version == null ||
                            version.equals(i.getMetadata().get("version")))
                    .collect(Collectors.toList());
            if (matched.isEmpty()) matched = instances; // fallback
            int idx = ThreadLocalRandom.current().nextInt(matched.size());
            return new DefaultResponse(matched.get(idx));
        });
    }
}
```

这套"标记头 → 元数据过滤"的最小实现有四个必须在生产补齐的点：① `matched.isEmpty()` 时**回落全量实例**是宽松策略——灰度请求被投进正式环境，在版本不兼容的场景下等于灰度事故，严格的租户/灰度体系这里应抛错或按预案降级，回落方向要和团队明确约定；② 标记头必须由 `7_interceptor.md` 的全链路染色机制种下（含跨服务传递、MQ 透传），单点实现只管一跳；③ 灰度实例数少（1-2 台）时随机选址可能反复打同一台，配熔断观察；④ 更主流的落地是现成组件——Nacos 的 metadata + 权重路由、Spring Cloud Gateway 的灰度路由或 ServiceMesh 层，Feign 侧自研负载均衡只作为理解机制的参考实现（`13_loadbalancer.md` §5 的 GrayLoadBalancer 是同一思路的另一半：过滤在 supplier 层做还是 choose 层做）。

# 8. 注册中心健康检查与下线

| 注册中心 | 检查机制 | 下线感知延迟 |
|----------|----------|-------------|
| Nacos | 临时实例心跳 (5s) + 持久实例主动探测 | 15s 内 |
| Eureka | 客户端心跳 (30s) + 自我保护 | 90s（默认） |
| Consul | Agent 健康检查 | 及时 |

**影响**：服务下线后，在 `CachingServiceInstanceListSupplier` 缓存过期前仍可能路由到已下线实例。建议：

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        ttl: 5s     # 缩短缓存，加速感知
      retry:
        enabled: true
        retry-on-all-operations: false  # 仅 GET 重试
    nacos:
      discovery:
        watch-delay: 1000  # Nacos Watch 延迟
```

## 8.1 优雅下线把"感知延迟"问题变成"发布纪律"

上表算的是**被动感知**（进程死了靠心跳超时），而实际发布走的是**主动注销**：正常停机时实例先在注册中心标记不健康/注销，调用方仍可能因三层缓存（§3.3 模型）把最后几个请求打到正在关闭的进程上。标准闭环是 `readiness 摘流 → 等一个缓存周期（≥ LB ttl + registry 推送）→ 再 stop 进程`，配合提供方 `server.shutdown: graceful` 处理已接收的请求。`retry-on-all-operations: false` 体现了发布窗口的重试边界：换实例重试只放行 GET，写操作交给幂等设计而不是交给运气（`9_retry.md` §8.1）。

# 9. 多注册中心

```java
@Configuration
public class MultiRegistryLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> multiRegistryBalancer(
            Environment env,
            LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);

        // 从 Nacos 获取实例
        ServiceInstanceListSupplier nacosSupplier = nacosSupplier(name);
        // 从 Eureka 获取实例
        ServiceInstanceListSupplier eurekaSupplier = eurekaSupplier(name);

        // 合并两个注册中心的实例
        ServiceInstanceListSupplier merged = new MergedSupplier(nacosSupplier, eurekaSupplier);

        return new RoundRobinLoadBalancer(
                ObjectProvider.of(merged), name
        );
    }
}
```

多注册中心最常见于**迁移过渡期**（Eureka→Nacos 双注册双订阅）。合并要注意两点：一是去重——同一实例在两边都注册时 host:port 相同，轮询不去重会让它获得双倍流量权重；二是 `spring.cloud.loadbalancer.discovery.enabled` 这类全局开关在多 `DiscoveryClient` Bean 并存时的选取规则（按类型注入会 NoUniqueBeanDefinition，需要 `@Primary` 或自己包 MergedSupplier，如上）。迁移完成的判据也在这里：调用方的合并视图里 **Nacos 来源占比 100%** 持续一周，才能拆掉旧注册中心。

# 10. 常见问题

**Q：服务名能直接解析但一直调不通？**

检查注册中心的实例是否健康（`status: UP` / `healthy: true`）。在 Nacos 控制台查看服务详情。

**Q：@FeignClient name 与服务提供方的 `spring.application.name` 不一致？**

必须完全一致，Feign 通过 name 从 DiscoveryClient 查找服务实例。

**Q：如何指定调用特定实例（调试场景）？**

在 `@FeignClient` 中指定 `url` 跳过注册中心：

```java
@FeignClient(name = "user-service", url = "http://192.168.1.10:8080")
```

**Q：`No instances available for xxx` 的完整决策树？**

按 §1.2 的层次从上往下排：① `curl` 调用方的 actuator/自测代码看 `DiscoveryClient.getInstances` 是否为空——空则进入 ②；非空则是 LoadBalancer 过滤层问题（zone 配置错把全部实例滤掉、自定义 supplier 链有 bug），检查 §6 配置。② 的分支：控制台看提供方**是否注册成功**（没注册查启动日志与 server-addr/namespace/凭据）；注册成功但你拿不到，核对**两边 namespace/group 是否一致**（最常见）、本服务是否真的引入了对应的 discovery starter（依赖里有 openfeign 没有 nacos-discovery 时，DiscoveryClient 会退化或找不到实现）。③ 间歇性"有时通有时不通"多半不是发现问题而是选址命中死实例，回到 §8 的缓存/重试模型。把这条树贴进故障模板，能把"查注册中心还是查配置"从小时级压到分钟级。
