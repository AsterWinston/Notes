
# 1. 集成架构

Spring Cloud LoadBalancer 通过 `DiscoveryClient` 抽象与注册中心解耦：

```
┌──────────────────────────────────────────────────────┐
│              Spring Cloud LoadBalancer               │
│                                                       │
│   DiscoveryClientServiceInstanceListSupplier          │
│           │                                          │
│           ▼                                          │
│      DiscoveryClient (接口抽象)                       │
│           │                                          │
│     ┌─────┼─────────────┬──────────────┐             │
│     ▼     ▼             ▼              ▼             │
│  Nacos   Eureka       Consul         Zookeeper      │
│  Client  Client       Client         Client          │
└──────────────────────────────────────────────────────┘
```

# 2. Nacos 集成

## 2.1 依赖与配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848      # Nacos 服务器地址
        namespace: dev                     # 命名空间
        group: DEFAULT_GROUP               # 分组
        cluster-name: SH                   # 集群名（用于就近路由）
        weight: 10                         # 权重（1-100）
        metadata:                          # 扩展元数据
          version: v2
          gray: "true"
          zone: zone1
        # 健康检查
        enabled: true
        # 心跳配置
        heart-beat-interval: 5000          # 心跳间隔（ms）
        heart-beat-timeout: 15000          # 心跳超时（ms）
        ip-delete-timeout: 30000           # IP 删除超时（ms）
    loadbalancer:
      nacos:
        enabled: true                      # 启用 Nacos 加权负载均衡
```

## 2.2 NacosDiscoveryClient 源码

```java
public class NacosDiscoveryClient implements DiscoveryClient {

    private final NacosServiceDiscovery nacosServiceDiscovery;

    @Override
    public List<ServiceInstance> getInstances(String serviceId) {
        try {
            return nacosServiceDiscovery.getInstances(serviceId);
        } catch (NacosException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public List<String> getServices() {
        try {
            return nacosServiceDiscovery.getServices();
        } catch (NacosException e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 2.3 Nacos 集群优先路由

```java
// NacosLoadBalancer 默认支持同集群优先
// 配置方式：
@Bean
public ReactorLoadBalancer<ServiceInstance> nacosLoadBalancer(
        Environment environment,
        LoadBalancerClientFactory factory) {
    String name = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);
    return new NacosLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
}
```

**多集群路由规则**：

```
当前服务 cluster = SH
  │
  ├── 候选列表中有 SH 集群实例 → 仅路由到 SH（加权）
  │
  └── 候选列表中没有 SH 集群实例 → 路由到所有集群实例
```

# 3. Eureka 集成

## 3.1 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    metadata-map:
      zone: zone1
      weight: 5

spring:
  cloud:
    loadbalancer:
      discovery:
        enabled: true
      cache:
        ttl: 30s
```

## 3.2 Eureka 实例元数据

```java
// 通过 Eureka 的 metadataMap 传递实例属性
eureka:
  instance:
    metadata-map:
      version: v2.0
      weight: "10"
      group: canary
      dc: shanghai
```

在 LoadBalancer 中读取元数据：

```java
ServiceInstance instance = ...;
String version = instance.getMetadata().get("version");
String group = instance.getMetadata().get("group");
```

# 4. Consul 集成

## 4.1 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        service-name: ${spring.application.name}
        instance-id: ${spring.application.name}:${server.port}
        prefer-ip-address: true
        health-check-path: /actuator/health
        health-check-interval: 10s
        tags:
          - version=v2
          - zone=zone1
```

# 5. Zookeeper 集成

## 5.1 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    zookeeper:
      connect-string: localhost:2181
      discovery:
        root: /services
        instance-id: ${spring.application.name}:${server.port}
```

# 6. 静态实例配置（无注册中心）

在开发/测试环境中可以不依赖注册中心：

```yaml
spring:
  cloud:
    loadbalancer:
      configurations: {}
    discovery:
      client:
        simple:
          instances:
            user-service:
              - uri: http://localhost:8081
                metadata:
                  weight: 10
              - uri: http://localhost:8082
                metadata:
                  weight: 5
```

对应实现类：

```java
public class SimpleDiscoveryClient implements DiscoveryClient {

    private final SimpleDiscoveryProperties simpleDiscoveryProperties;

    @Override
    public List<ServiceInstance> getInstances(String serviceId) {
        return this.simpleDiscoveryProperties.getInstances(serviceId)
                .stream()
                .map(DefaultServiceInstance::new)
                .collect(Collectors.toList());
    }
}
```

# 7. DiscoveryClientServiceInstanceListSupplier 源码

```java
public class DiscoveryClientServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final DiscoveryClient discoveryClient;
    private final Environment environment;

    @Override
    public Flux<ServiceInstance> get() {
        // 获取当前服务名（从 Environment 读取 loadbalancer.client.name）
        String serviceId = environment.getProperty(
                LoadBalancerClientFactory.PROPERTY_NAME);

        // 调用 DiscoveryClient.getInstances
        List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);

        if (instances.isEmpty()) {
            log.warn("No instances available for {}", serviceId);
        }

        return Flux.fromIterable(instances);
    }
}
```

# 8. 服务发现客户端对比

| 特性 | Nacos | Eureka | Consul | Zookeeper |
|------|-------|--------|--------|-----------|
| **CAP** | AP + CP（可切换） | AP | CP | CP |
| **健康检查** | TCP/HTTP/MySQL | 心跳 | HTTP/TCP/Script | 心跳 |
| **元数据** | ✓ | ✓ | ✓（Tags） | ✓ |
| **权重支持** | ✓（原生） | ✗（需通过 metadata） | ✗ | ✗ |
| **集群路由** | ✓（原生） | ✗ | ✗ | ✗ |
| **响应式** | ✓ | ✗ | ✗ | ✗ |
| **配置中心** | ✓（内置） | ✗ | ✓（KV） | ✓（弱） |

# 9. 集成最佳实践

```yaml
# 推荐配置：生产级 Nacos + LoadBalancer
spring:
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:prod}
        group: DEFAULT_GROUP
        metadata:
          version: ${APP_VERSION}
          region: ${APP_REGION}

    loadbalancer:
      nacos:
        enabled: true
      cache:
        enabled: true
        ttl: 10s                        # 生产环境缩短缓存
      health-check:
        interval: 10s
        refetch-instances-interval: 25s
```
