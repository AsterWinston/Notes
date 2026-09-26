
# 1. 概述

在多机房/多可用区（Multi-AZ）部署场景中，Zone 亲和路由器优先将请求发送到**与调用方在同一 Zone** 的服务实例，以减少跨 Zone 的网络延迟和带宽成本。

```
┌─────────────────── Zone SH ───────────────────┐
│  调用方 (order-service)                        │
│     │                                          │
│     ├────→ user-service-1 (SH, 低延迟 1ms)    │
│     │                                          │
│     └────→ user-service-2 (SH, 低延迟 1ms)    │
└───────────────────────────────────────────────┘

┌────────────────── Zone BJ ───────────────────┐
│  user-service-3 (BJ, 高延迟 30ms)  ← 不会被优先选择│
└──────────────────────────────────────────────┘
```

# 2. ZonePreferenceServiceInstanceListSupplier

## 2.1 源码解析

```java
public class ZonePreferenceServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final String zone;

    public ZonePreferenceServiceInstanceListSupplier(
            ServiceInstanceListSupplier delegate, String zone) {
        super(delegate);
        this.zone = zone;
    }

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .collectList()
                .flatMapMany(allInstances -> {
                    // ① 按 zone 分组
                    List<ServiceInstance> sameZone = allInstances.stream()
                            .filter(instance -> {
                                String instanceZone = instance.getMetadata()
                                        .getOrDefault("zone", "");
                                return zone.equalsIgnoreCase(instanceZone);
                            })
                            .collect(Collectors.toList());

                    // ② 有同 Zone 实例 → 仅返回同 Zone 实例
                    if (!sameZone.isEmpty()) {
                        return Flux.fromIterable(sameZone);
                    }

                    // ③ 无同 Zone 实例 → 降级返回全部实例
                    return Flux.fromIterable(allInstances);
                });
    }
}
```

## 2.2 决策逻辑

```
同 Zone 实例 > 0 ？
  │
  ├── 是 → 仅路由到同 Zone 实例（完全隔离）
  │
  └── 否 → 降级路由到所有实例（跨 Zone）
```

# 3. 配置方式

## 3.1 YAML 配置

```yaml
spring:
  cloud:
    loadbalancer:
      zone: SH                    # 当前调用方所在的 Zone

    nacos:
      discovery:
        metadata:
          zone: SH                # 当前实例注册到 Nacos 时声明的 Zone
```

## 3.2 Builder 链式配置

```java
@Bean
public ServiceInstanceListSupplier zoneAwareSupplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()
            .withZonePreference()        // 开启 Zone 感知
            .withCaching()
            .build(context);
}
```

## 3.3 各注册中心的 Zone 配置

**Nacos**：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: SH           # Nacos 使用 cluster 概念（等价于 Zone）
        metadata:
          zone: SH
```

**Eureka**：

```yaml
eureka:
  instance:
    metadata-map:
      zone: SH
  client:
    availability-zones:
      region-1: SH,BJ,GZ
```

**Consul**：

```yaml
spring:
  cloud:
    consul:
      discovery:
        tags:
          - zone=SH
```

# 4. 多 Zone 优先级路由

当同 Zone 没有实例时，可以实现按 Zone 优先级依次尝试：

```java
public class MultiZonePrioritySupplier
        extends DelegatingServiceInstanceListSupplier {

    // Zone 优先级列表：当前 Zone 排第一
    private final List<String> zonePriority;

    public MultiZonePrioritySupplier(
            ServiceInstanceListSupplier delegate,
            List<String> zonePriority) {
        super(delegate);
        this.zonePriority = zonePriority;
    }

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .collectList()
                .flatMapMany(allInstances -> {
                    for (String zone : zonePriority) {
                        List<ServiceInstance> matched = allInstances.stream()
                                .filter(i -> zone.equals(
                                        i.getMetadata().get("zone")))
                                .collect(Collectors.toList());
                        if (!matched.isEmpty()) {
                            return Flux.fromIterable(matched);
                        }
                    }
                    return Flux.fromIterable(allInstances);
                });
    }
}
```

```yaml
# 配置多级 Zone 优先级
app:
  zone-priority: SH,BJ,GZ    # 优先 SH → 其次 BJ → 最后 GZ
```

# 5. Nacos 集群优先路由

Nacos 的 `cluster-name` 可以与 LoadBalancer 的 Zone 机制协同：

```java
@Bean
public ReactorLoadBalancer<ServiceInstance> nacosClusterBalancer(
        Environment environment,
        LoadBalancerClientFactory factory) {
    String name = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);

    // NacosLoadBalancer 内置了同集群优先逻辑
    return new NacosLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
}
```

**Nacos 集群优先 + Zone 感知的完整配置**：

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        cluster-name: SH        # Nacos 原生集群概念

    loadbalancer:
      zone: SH                  # Spring LoadBalancer Zone 概念
      nacos:
        enabled: true           # 启用 NacosLoadBalancer
```

# 6. 跨 Zone 降级策略

## 6.1 百分比降级

```java
public class GradualZoneFallbackSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final String zone;
    private final double fallbackRatio;  // 跨 Zone 流量比例（0~1）

    @Override
    public Flux<ServiceInstance> get() {
        return getDelegate().get()
                .collectList()
                .flatMapMany(instances -> {
                    List<ServiceInstance> sameZone = instances.stream()
                            .filter(i -> zone.equals(i.getMetadata().get("zone")))
                            .collect(Collectors.toList());
                    List<ServiceInstance> otherZone = instances.stream()
                            .filter(i -> !zone.equals(i.getMetadata().get("zone")))
                            .collect(Collectors.toList());

                    // 即使同 Zone 有实例，也分配 fallbackRatio 比例给跨 Zone
                    List<ServiceInstance> result = new ArrayList<>(sameZone);
                    if (!otherZone.isEmpty()) {
                        int crossZoneCount = (int) (result.size() * fallbackRatio);
                        result.addAll(otherZone.subList(0,
                                Math.min(crossZoneCount, otherZone.size())));
                    }
                    return Flux.fromIterable(result);
                });
    }
}
```

## 6.2 动态降级开关

```java
@Component
public class ZoneFallbackController {

    private final AtomicBoolean sameZoneOnly = new AtomicBoolean(true);

    public boolean isSameZoneOnly() {
        return sameZoneOnly.get();
    }

    public void enableCrossZone() {
        sameZoneOnly.set(false);  // 同 Zone 故障 → 允许跨 Zone
    }

    public void disableCrossZone() {
        sameZoneOnly.set(true);   // 同 Zone 恢复 → 回到仅同 Zone
    }
}
```

# 7. 同 Zone 亲和 + 健康检查

```java
@Bean
public ServiceInstanceListSupplier productionSupplier(
        ConfigurableApplicationContext context) {
    return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()           // ① 从注册中心获取
            .withZonePreference()            // ② 同 Zone 优先过滤
            .withHealthChecks()              // ③ 健康检查过滤（避免路由到已故障的同 Zone 实例）
            .withCaching()                   // ④ 缓存
            .build(context);
}
```

**请求链路**：

```
注册中心返回: [SH-1, SH-2, BJ-1, BJ-2, GZ-1]
    │
    ▼ ZonePreference(zone=SH)
过滤后: [SH-1, SH-2]        ← 仅同 Zone
    │
    ▼ HealthCheck
过滤后: [SH-1]               ← SH-2 不健康被剔除
    │
    ▼ RoundRobinLoadBalancer.choose()
选中: SH-1
```

# 8. 最佳实践

| 规则 | 说明 |
|------|------|
| **Zone 名统一** | 确保各服务注册时使用相同的 Zone 命名 |
| **健康检查** | Zone 过滤后务必加健康检查，避免故障放大 |
| **降级策略** | 同 Zone 全挂时自动 fallback 到其他 Zone |
| **缓存 TTL** | 跨 Zone 部署时 TTL 不宜过长，便于 Zone 切换 |
| **K8s 适配** | 可将 K8s 的 Topology Zone 标签映射为 LoadBalancer Zone |
