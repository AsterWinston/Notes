
# 1. Ribbon 在 Zuul 中的角色

Ribbon 是一个客户端负载均衡器，在 Zuul 中用于 **serviceId 路由**场景下选择具体的后端服务实例。配合 Eureka 服务发现，构成了 Zuul 中最核心的路由能力。

```
客户端 → Zuul 网关 → Ribbon 负载均衡器 → 后端服务实例列表
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                实例 8081  实例 8082  实例 8083
```

# 2. RibbonRoutingFilter 源码分析

## 2.1 完整执行链路

```java
@Component
public class RibbonRoutingFilter extends ZuulFilter {
    private final RibbonCommandFactory<?> ribbonCommandFactory;

    @Override
    public String filterType() { return "route"; }

    @Override
    public int filterOrder() { return 10; }

    @Override
    public boolean shouldFilter() { ... }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();

        // 1. 创建 RibbonCommand（结合 Hystrix + Ribbon）
        RibbonCommand command = this.ribbonCommandFactory.create(ctx);

        // 2. 执行命令
        ClientHttpResponse response = command.execute();

        // 3. 将响应写入上下文
        ctx.setResponseDataStream(response.getBody());
        ctx.setResponseStatusCode(response.getRawStatusCode());
        // ...
        return null;
    }
}
```

## 2.2 RibbonCommandFactory

```java
public class HttpClientRibbonCommandFactory
        implements RibbonCommandFactory<AbstractRibbonCommand> {

    private final SpringClientFactory clientFactory;
    private final ZuulProperties zuulProperties;

    @Override
    public HttpClientRibbonCommand create(SetRequestContext requestContext) {
        // 从 RequestContext 获取 serviceId
        String serviceId = requestContext.getServiceId();

        if (zuulProperties.getRibbonIsolationStrategy() == SEMAPHORE) {
            // 信号量隔离：在当前线程执行 Ribbon 调用
            return new HttpClientRibbonCommand(
                serviceId, verb, uri, retryable, headers, params);
        } else {
            // 线程隔离：在 Hystrix 线程池中执行
            return new HttpClientRibbonCommand(
                serviceId, verb, uri, retryable, headers, params);
        }
    }
}
```

# 3. Ribbon 负载均衡策略

## 3.1 内置负载均衡规则

| 规则类 | 策略 | 说明 |
|--------|------|------|
| `RoundRobinRule` | 轮询 | 按顺序循环分配到每个实例 |
| `RandomRule` | 随机 | 随机选择实例 |
| `WeightedResponseTimeRule` | 加权响应时间 | 根据实例响应时间动态计算权重 |
| `BestAvailableRule` | 最空闲 | 选择并发请求最少的实例 |
| `RetryRule` | 重试 | 在配置的时间段内重试选择 |
| `AvailabilityFilteringRule` | 可用性过滤 | 过滤掉断路器打开或连接过多的实例 |
| `ZoneAvoidanceRule`（默认） | 区域感知 | 综合判断实例所在 Zone 的可用性 |

## 3.2 默认行为：ZoneAvoidanceRule

Spring Cloud Zuul 默认使用 `ZoneAvoidanceRule`：

```java
public class ZoneAvoidanceRule extends PredicateBasedRule {
    // 使用 CompositePredicate，组合两个判断条件
    // 1. ZoneAvoidancePredicate: 根据 Zone 的错误率选择
    // 2. AvailabilityPredicate: 排除断路器跳闸的实例
    private CompositePredicate compositePredicate;

    @Override
    public Server choose(Object key) {
        ILoadBalancer lb = getLoadBalancer();
        // 先过滤可用实例，再从过滤结果中选择
        List<Server> available = lb.getReachableServers();
        // 从过滤后的实例中轮询选择
        return ...;
    }
}
```

## 3.3 自定义负载均衡策略

```yaml
# 全局配置
ribbon:
  NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule

# 按 serviceId 配置（推荐）
user-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule

order-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.BestAvailableRule
```

如需更复杂的策略（如基于请求头路由、灰度发布），可实现自定义 IRule：

```java
@Component
public class HeaderBasedRule extends AbstractLoadBalancerRule {

    @Override
    public Server choose(Object key) {
        RequestContext ctx = RequestContext.getCurrentContext();
        String version = ctx.getRequest().getHeader("X-API-Version");

        ILoadBalancer lb = getLoadBalancer();
        List<Server> reachableServers = lb.getReachableServers();

        // 根据版本号选择对应的实例
        for (Server server : reachableServers) {
            if (server.getMetadata().get("version").equals(version)) {
                return server;
            }
        }

        // 兜底：轮询
        return new RoundRobinRule().choose(lb, key);
    }

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {}
}
```

# 4. Ribbon 重试机制

## 4.1 重试配置

```yaml
zuul:
  retryable: true  # 全局开启重试

user-service:
  ribbon:
    # 当前实例重试次数（不包括首次）
    MaxAutoRetries: 1

    # 重试下一个实例的最大次数（不包括第一个）
    MaxAutoRetriesNextServer: 2

    # 是否所有操作都可重试（默认 false，GET 可重试，POST 不可）
    OkToRetryOnAllOperations: false

    # 重试时选择实例的间隔（ms）
    ServerListRefreshInterval: 5000

    # 连接超时（ms）
    ConnectTimeout: 2000

    # 读取超时（ms）
    ReadTimeout: 5000
```

## 4.2 重试流程

```
第 1 次请求 → 实例 A → 超时（重试）
第 2 次请求 → 实例 A → 超时（MaxAutoRetries=1，切换到下个实例）
第 3 次请求 → 实例 B → 超时（MaxAutoRetriesNextServer=2，继续）
第 4 次请求 → 实例 C → 成功 ✅
```

重试机制依赖 `spring-retry` 库，需要额外引入：

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

## 4.3 重试源码链路

```
RibbonRoutingFilter.run()
    │
    ▼
RibbonCommand.execute() (Hystrix Command)
    │
    ▼
AbstractLoadBalancerAwareClient.executeWithLoadBalancer()
    │
    ▼
RetryTemplate (spring-retry)
    │
    ├── 正常返回 → 结束
    │
    └── 异常 → 根据 retryPolicy 判断是否重试
         │
         ├── 可重试 → 选择下一个实例 → 重新执行
         │
         └── 超出重试次数 → 抛出异常 → Hystrix.fallback
```

# 5. 连接池与超时配置

## 5.1 HttpClient 连接池

Zuul 的 HTTP 请求通过 Apache HttpClient 发起。连接池参数对性能影响巨大：

```yaml
user-service:
  ribbon:
    # 连接池配置
    MaxTotalConnections: 500          # 最大总连接数
    MaxConnectionsPerHost: 200        # 每个 Host 最大连接数
    PoolMaxThreads: 200              # 连接池最大线程数

    # 超时配置
    ConnectTimeout: 2000             # 建立连接超时（ms）
    ReadTimeout: 10000               # 读取数据超时（ms）
    SocketTimeout: 10000             # Socket 超时（ms）

    # 连接存活与重试
    ConnectionPoolCleanerTaskEnabled: true
    ConnIdleEvictTimeMilliSeconds: 3600000  # 空闲连接回收时间（ms）
    GZipPayload: true                # 启用 GZip 压缩
```

## 5.2 connectionRequestTimeout

`connectionRequestTimeout` 是常被忽略但很重要的参数——从连接池获取连接的超时时间：

```yaml
user-service:
  ribbon:
    connection-request-timeout: 3000  # 从连接池获取连接的超时
```

当连接池满时可以设置此参数，避免请求无限等待连接。

# 6. SpringClientFactory 原理

`SpringClientFactory` 是 Spring Cloud Netflix 为每个 Ribbon 客户端创建独立 Spring 容器（ApplicationContext）的工厂：

```java
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {
    // 为每个 serviceId 创建独立的 ApplicationContext
    // 每个 context 可以有自己的 Ribbon 配置

    public SpringClientFactory() {
        // 指定默认配置类
        super(RibbonClientConfiguration.class, "ribbon", "ribbon.client.name");
    }

    // 获取指定 serviceId 的 LoadBalancer
    public ILoadBalancer getLoadBalancer(String name) {
        return getInstance(name, ILoadBalancer.class);
    }

    // 获取指定 serviceId 的 ServerList
    public ServerList<?> getServerList(String name) {
        return getInstance(name, ServerList.class);
    }
}
```

`NamedContextFactory` 的设计让每个服务拥有独立的配置空间：

```
SpringClientFactory
    │
    ├── "user-service"  → ApplicationContext(user-service)
    │                        ├── ILoadBalancer
    │                        ├── IClientConfig
    │                        └── ServerList
    │
    ├── "order-service" → ApplicationContext(order-service)
    │                        ├── ILoadBalancer
    │                        ├── IClientConfig
    │                        └── ServerList
    │
    └── "default"       → ApplicationContext(default)
                             └── RibbonClientConfiguration
```

# 7. 服务实例选择流程

```
RibbonRoutingFilter.run()
    │
    ▼
AbstractRibbonCommand (Hystrix)
    │
    ▼
LoadBalancerContext.getServerFromLoadBalancer()
    │
    ▼
ZoneAwareLoadBalancer.chooseServer(key)
    │
    ├── 1. 获取所有 Zone 及其实例
    │      ┌──────────┐  ┌──────────┐
    │      │ Zone-A:  │  │ Zone-B:  │
    │      │ 例 1     │  │ 例 3     │
    │      │ 例 2     │  │          │
    │      └──────────┘  └──────────┘
    │
    ├── 2. ZoneAvoidancePredicate 过滤
    │      排除断路器跳闸的实例和错误率高的 Zone
    │
    ├── 3. 从候选 Zone 中选择
    │      IRule.choose(loadBalancer, key)
    │      └── 默认: ZoneAvoidanceRule
    │
    └── 4. 返回选中的 Server
```

# 8. IRule 接口与自定义策略

```java
public interface IRule {
    // 从负载均衡器中选择一个服务实例
    Server choose(Object key);

    // 设置负载均衡器引用
    void setLoadBalancer(ILoadBalancer lb);

    // 获取负载均衡器引用
    ILoadBalancer getLoadBalancer();
}
```

## 8.1 实现基于元数据的路由规则

```java
public class MetadataAwareRule extends ZoneAvoidanceRule {

    @Override
    public Server choose(Object key) {
        RequestContext ctx = RequestContext.getCurrentContext();
        String env = ctx.getRequest().getHeader("X-Deploy-Env");

        if (env == null) {
            return super.choose(key);  // 兜底
        }

        ILoadBalancer lb = getLoadBalancer();
        List<Server> servers = lb.getReachableServers();

        // 根据环境标签（dev/test/prod）选择实例
        List<Server> matchedServers = servers.stream()
            .filter(s -> env.equals(s.getMetadata().get("env")))
            .collect(Collectors.toList());

        if (matchedServers.isEmpty()) {
            return super.choose(key);
        }

        // 从匹配的实例中轮询
        return new RoundRobinRule().choose(
            new BaseLoadBalancer(matchedServers), key);
    }
}
```

# 9. 常见 Ribbon 配置示例

## 9.1 综合生产配置

```yaml
spring:
  application:
    name: zuul-gateway

zuul:
  retryable: true
  routes:
    user-service:
      path: /user/**
      serviceId: user-service

# 全局 Ribbon 默认配置
ribbon:
  eager-load:
    enabled: true  # 启动时预加载，避免首次请求慢
  ReadTimeout: 6000
  ConnectTimeout: 3000
  MaxAutoRetries: 1
  MaxAutoRetriesNextServer: 2
  OkToRetryOnAllOperations: false

# 按 serviceId 精细化配置
user-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
    NIWSServerListClassName: com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
    ReadTimeout: 5000
    ConnectTimeout: 2000
    MaxAutoRetries: 0
    MaxAutoRetriesNextServer: 1
    ServerListRefreshInterval: 10000
```

## 9.2 饥饿加载

```yaml
ribbon:
  eager-load:
    enabled: true
    clients:
      - user-service
      - order-service
      - product-service
```

饥饿加载在应用启动时预初始化 Ribbon 客户端，避免首次请求的"冷启动"延迟。

## 9.3 不使用 Eureka 的 Ribbon 配置

```yaml
ribbon:
  eureka:
    enabled: false

user-service:
  ribbon:
    listOfServers: 192.168.1.10:8080,192.168.1.11:8080,192.168.1.12:8080
    ServerListRefreshInterval: 5000
```

适用于没有服务注册中心但需要负载均衡的场景。
