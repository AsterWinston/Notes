
# 1. 概述

Sentinel 通过 Dubbo Filter SPI 机制透明集成 Dubbo，对 Provider 和 Consumer 两侧均提供流量防护能力。

```
Consumer                                Provider
┌──────────────────────┐               ┌──────────────────────┐
│  SentinelDubboConsumer│              │  SentinelDubboProvider│
│  Filter               │              │  Filter               │
│  ┌──────────────────┐ │   RPC Call   │  ┌──────────────────┐ │
│  │ 流量控制          │ │ ──────────→ │  │ 流量控制          │ │
│  │ 熔断降级          │ │ ←────────── │  │ 系统保护          │ │
│  │ 统计监控          │ │              │  │ 统计监控          │ │
│  └──────────────────┘ │              │  └──────────────────┘ │
└──────────────────────┘               └──────────────────────┘
```

# 2. 依赖引入

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<!-- Sentinel Dubbo Adapter -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-sentinel-spring-cloud-starter</artifactId>
    <version>${dubbo.version}</version>
</dependency>
```

# 3. Filter 注册机制

Sentinel 通过 Dubbo SPI 自动注册 Filter：

```
META-INF/dubbo/org.apache.dubbo.rpc.Filter:

sentinel.dubbo.provider=com.alibaba.csp.sentinel.adapter.dubbo3.SentinelDubboProviderFilter
sentinel.dubbo.consumer=com.alibaba.csp.sentinel.adapter.dubbo3.SentinelDubboConsumerFilter
```

**Provider 端资源名**：`{interfaceName}:{methodName}({paramTypes})`

```java
// 资源名示例
com.example.UserService:getUser(Long)
com.example.OrderService:createOrder(Order, Long)
```

**Consumer 端资源名**：`{interfaceName}:{methodName}({paramTypes})`

# 4. Provider 端限流

## 4.1 配置方式

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow
```

## 4.2 通过 Dashboard 配置

在 Dashboard 中针对资源名 `com.example.UserService:getUser(Long)` 配置流控规则。

## 4.3 通过 @DubboService 配置

```java
@DubboService(parameters = "sentinel.flow.rules=[{\"resource\":\"" +
    "com.example.UserService:getUser(Long)\",\"grade\":1,\"count\":100}]")
public class UserServiceImpl implements UserService { ... }
```

# 5. Consumer 端熔断

Consumer 端 Sentinel Filter 对 RPC 调用进行熔断降级：

```java
@DubboService
public class OrderServiceImpl implements OrderService {

    @DubboReference(
        timeout = 3000,
        retries = 0,  // 开启 Sentinel 熔断后建议关闭 Dubbo 自身重试
        parameters = "sentinel.enabled=true"
    )
    private UserService userService;

    @Override
    public OrderResult createOrder(Order order) {
        try {
            // RPC 调用，自动受 Sentinel Dubbo Consumer Filter 保护
            User user = userService.getUser(order.getUserId());
            // ... 业务逻辑
        } catch (RpcException e) {
            // 熔断或流控拒绝
            return OrderResult.fallback(order);
        }
    }
}
```

# 6. 嵌套调用与 Context

Dubbo 调用链中的 Context 传递：

```java
// Consumer 端
ContextUtil.enter("order-context", "order-service");
try (Entry entry = SphU.entry("orderEntry")) {
    // 调用 UserService（Sentinel Dubbo Filter 自动创建嵌套 Entry）
    User user = userService.getUser(userId);
}
ContextUtil.exit();
```

**注意**：Dubbo Filter 中的 Entry 会在 Filter 调用链中自动管理，不需要手动 release。

# 7. Filter 执行顺序

```java
// Consumer 端 Filter 链
ConsumerContextFilter → SentinelDubboConsumerFilter → ... → Transport

// Provider 端 Filter 链
Transport → ... → SentinelDubboProviderFilter → Business Method
```

Sentinel Filter 应尽可能靠近业务方法，确保其他 Filter（如 Context、Trace）在 Sentinel 判断之前执行。

# 8. 配置示例

```yaml
dubbo:
  application:
    name: order-service
  registry:
    address: nacos://localhost:8848

spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
        port: 8719
      eager: true
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            rule-type: flow
        degrade:
          nacos:
            server-addr: localhost:8848
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            rule-type: degrade
```

# 9. 最佳实践

1. **Dubbo retries=0**：启用 Sentinel 熔断后，关闭 Dubbo 自身的重试机制
2. **资源粒度**：接口 + 方法粒度即可，过细会带来管理负担
3. **Provider 侧流控**：保护 Provider 不被 Consumer 海量请求击垮
4. **Consumer 侧熔断**：保护 Consumer 线程不被慢 Provider 阻塞
5. **异常透传**：`BlockException` 会自动转换为 `RpcException`，Consumer 可捕获处理
