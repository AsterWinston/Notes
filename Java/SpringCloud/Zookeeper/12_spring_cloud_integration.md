
# 1. Spring Cloud Zookeeper 概述

Spring Cloud Zookeeper 是 Spring Cloud 生态中集成 Apache ZooKeeper 的官方子项目，提供**服务注册与发现**和**配置管理**两大核心能力。

```text
Spring Cloud Zookeeper 在生态中的位置：
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  Spring Cloud 抽象层                                      │
│  ┌──────────┐  ┌────────────┐  ┌──────────────────┐     │
│  │Discovery │  │   Config   │  │ LoadBalancer    │     │
│  │(Service  │  │(Property  │  │(@LoadBalanced   │     │
│  │ Discovery)│  │ Source)   │  │ RestTemplate)  │     │
│  └────┬─────┘  └─────┬──────┘  └────────┬─────────┘     │
│       │              │                 │                  │
│       └──────────────┼─────────────────┘                  │
│                      │                                    │
│              ┌───────┴────────┐                           │
│              │ Spring Cloud   │                           │
│              │   Zookeeper    │                           │
│              └───────┬────────┘                           │
│                      │                                    │
│              ┌───────┴────────┐                           │
│              │ Apache Curator │                           │
│              └───────┬────────┘                           │
│                      │                                    │
│              ┌───────┴────────┐                           │
│              │   ZooKeeper    │                           │
│              └────────────────┘                           │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

# 2. 依赖引入

```xml
<!-- Maven -->
<properties>
    <spring-cloud.version>2023.0.0</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring Cloud Zookeeper Discovery（服务发现） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    </dependency>

    <!-- Spring Cloud Zookeeper Config（配置管理） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-config</artifactId>
    </dependency>

    <!-- Spring Cloud LoadBalancer（客户端负载均衡） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
</dependencies>
```

```groovy
// Gradle
implementation 'org.springframework.cloud:spring-cloud-starter-zookeeper-discovery'
implementation 'org.springframework.cloud:spring-cloud-starter-zookeeper-config'
implementation 'org.springframework.cloud:spring-cloud-starter-loadbalancer'
```

# 3. 自动配置原理

```text
Spring Cloud Zookeeper 自动配置类：
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  ZookeeperDiscoveryClientConfiguration                   │
│    @ConditionalOnClass({CuratorFramework.class,           │
│      ZookeeperDiscoveryProperties.class})                 │
│    · 自动创建 CuratorFramework Bean                      │
│    · 自动创建 ZookeeperDiscoveryClient                   │
│    · 自动创建 ZookeeperServiceWatch                      │
│    · 自动创建 ZookeeperAutoServiceRegistration           │
│                                                           │
│  ZookeeperConfigBootstrapConfiguration                   │
│    @ConditionalOnClass({CuratorFramework.class,           │
│      ZookeeperConfigProperties.class})                    │
│    · 自动创建 ZookeeperPropertySourceLocator             │
│    · 将 Zookeeper 配置源注入到 Environment               │
│    · 支持动态刷新（@RefreshScope）                        │
│                                                           │
│  ZookeeperLoadBalancerConfiguration                      │
│    @ConditionalOnClass(LoadBalancerClient.class)          │
│    · 自动创建 ZookeeperServiceInstanceListSupplier       │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

## 3.1 自动注册流程

```text
服务注册流程（应用启动时）：
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  1. Spring Boot 应用启动                                     │
│  2. CuratorFramework Bean 初始化（连接 ZK）                   │
│  3. ZookeeperAutoServiceRegistration.onApplicationEvent()    │
│     ├─ 从配置获取 serviceId, port, instanceId 等             │
│     ├─ 构造 ZookeeperRegistration                            │
│     └─ 调用 ServiceRegistry.register()                      │
│  4. Curator 在 ZK 上创建临时节点：                           │
│     /services/${spring.application.name}/                   │
│       ${instanceId} → {"name":"...", "host":"...", ...}     │
│  5. 应用关闭时自动删除临时节点（Session 断开）               │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

# 4. 服务注册与配置

## 4.1 服务提供者配置

```yaml
# application.yml — 服务提供者
spring:
  application:
    name: order-service            # 服务名（将作为 ZK 中注册的路径）
  cloud:
    zookeeper:
      connect-string: localhost:2181
      discovery:
        enabled: true
        root: /services            # 服务注册根路径
        register: true             # 是否注册（默认 true）
        instance-id: ${spring.application.name}:${server.port}
        metadata:
          version: 1.0
          region: cn-beijing
      config:
        enabled: true
        root: /config              # 配置根路径
        default-context: application
        profile-separator: '::'
        watcher:
          enabled: true            # 启用配置动态刷新

server:
  port: 8081
```

## 4.2 服务消费者配置

```yaml
# application.yml — 服务消费者
spring:
  application:
    name: gateway-service
  cloud:
    zookeeper:
      connect-string: localhost:2181
      discovery:
        enabled: true
        register: false            # 消费者可以不注册
        root: /services
    loadbalancer:
      zookeeper:
        enabled: true              # 启用 ZK 负载均衡

server:
  port: 8080
```

## 4.3 ZK 节点结构

```text
ZooKeeper 中的服务注册结构：

/services
  ├── /order-service
  │   ├── /order-service:8081    (临时节点，存储实例元数据)
  │   │   └── data: {
  │   │       "name": "order-service",
  │   │       "id": "order-service:8081",
  │   │       "address": "192.168.1.10",
  │   │       "port": 8081,
  │   │       "sslPort": null,
  │   │       "payload": {
  │   │           "@class": "...ZookeeperInstance",
  │   │           "id": "...",
  │   │           "name": "order-service",
  │   │           "metadata": {"version": "1.0"}
  │   │       },
  │   │       "registrationTimeUTC": 1705300000000,
  │   │       "serviceType": "DYNAMIC",
  │   │       "uriSpec": {
  │   │           "parts": [{"value": "http", "variable": false},
  │   │                      {"value": "192.168.1.10", "variable": false},
  │   │                      {"value": "8081", "variable": false}]
  │   │       }
  │   │   }
  │   └── /order-service:8082    (另一个实例)
  ├── /user-service
  │   └── /user-service:8083
  └── /pay-service
```

# 5. 配置管理

## 5.1 ZK 中的配置存储

```text
ZooKeeper 配置目录结构：

/config
  ├── /application                  ← 公共配置（所有应用共享）
  │   └── data: "db.timeout=5000"
  ├── /application,dev              ← 公共 + dev 环境
  │   └── data: "db.timeout=3000"
  ├── /order-service                ← order-service 特有配置
  │   └── data: "server.port=8081"
  ├── /order-service,dev            ← order-service + dev 环境
  │   └── data: "server.port=8082"
  └── /order-service::dev           ← 使用 :: 分隔符（profile-separator）
```

## 5.2 动态配置刷新

```java
// 使用 @RefreshScope 实现动态配置刷新
@RestController
@RefreshScope  // 配置变更时自动刷新此 Bean
public class ConfigController {

    @Value("${db.timeout:5000}")
    private int dbTimeout;

    @Value("${cache.ttl:300}")
    private int cacheTtl;

    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of(
            "dbTimeout", dbTimeout,
            "cacheTtl", cacheTtl
        );
    }
}
```

```yaml
# 配置刷新相关
spring:
  cloud:
    zookeeper:
      config:
        watcher:
          enabled: true    # 启用配置变更监听
```

# 6. 调用配置

```java
// 使用 @LoadBalanced RestTemplate
@Configuration
public class RestTemplateConfig {
    @Bean
    @LoadBalanced  // 启用客户端负载均衡
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@RestController
public class OrderController {
    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/order")
    public String placeOrder() {
        // 使用服务名而非 IP:Port 调用
        String result = restTemplate.getForObject(
            "http://user-service/user/info",
            String.class
        );
        return "Order placed. User info: " + result;
    }
}
```

```java
// 使用 Spring Cloud OpenFeign
@FeignClient(name = "user-service")  // 服务名，自动从 ZK 获取实例
public interface UserServiceClient {
    @GetMapping("/user/info")
    UserInfo getUserInfo(@RequestParam("id") Long id);
}
```

# 7. 健康检查

## 7.1 Actuator 健康检查集成

```yaml
# 引入 Actuator 后自动启用健康检查端点
spring:
  cloud:
    zookeeper:
      discovery:
        health-check-enabled: true
        health-check-interval: 15s  # 健康检查间隔

# Actuator 端点
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      probes:
        enabled: true
```

```text
健康检查机制：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  1. 服务启动时自动注册到 ZK                           │
│  2. ZookeeperServiceWatch 定期检查 /actuator/health   │
│  3. 若实例不健康：                                     │
│     · 将实例状态标记为 OUT_OF_SERVICE                 │
│     · 消费者不再路由到该实例                          │
│  4. 若实例恢复健康：                                   │
│     · 自动恢复路由                                    │
│                                                       │
│  注意：                                               │
│    · ZK 本身不做健康检查（仅 Session 心跳）           │
│    · Spring Cloud Zookeeper 在应用层做扩展检查         │
│                                                       │
└──────────────────────────────────────────────────────┘
```

# 8. 高级配置项

```yaml
spring:
  cloud:
    zookeeper:
      # ============== 连接配置 ==============
      connect-string: zk1:2181,zk2:2181,zk3:2181
      block-until-connected-wait: 10s  # 等待连接成功的超时
      block-until-connected-unit: SECONDS
      connection-timeout: 15s
      session-timeout: 30s
      max-retries: 5
      max-sleep-ms: 5000
      base-sleep-time-ms: 1000

      # ============== Discovery 配置 ==============
      discovery:
        enabled: true
        root: /services
        register: true
        instance-id: ${spring.application.name}:${server.port}
        instance-host: ${spring.cloud.client.ip-address}
        instance-port: ${server.port}
        instance-ssl-port: null
        uri-spec: http://{address}:{port}
        metadata:
          management.port: ${management.server.port:8080}
          version: ${info.version:unknown}

      # ============== Config 配置 ==============
      config:
        enabled: true
        root: /config
        default-context: application
        profile-separator: ','
        watcher:
          enabled: true
          delay: 1000  # 事件防抖延迟

      # ============== 依赖关闭顺序 ==============
      dependencies:
        enabled: true
        dependency-configurations:
          - path: /services
            required: true    # 依赖服务是否必须存在
            load-order: 1     # 依赖加载顺序
            wait-time: 60s    # 等待时间
```

# 9. 关键 Bean 注入

```java
// 手动注入 CuratorFramework
@Autowired
private CuratorFramework curatorFramework;

// DiscoveryClient（Spring Cloud 抽象）
@Autowired
private DiscoveryClient discoveryClient;

@GetMapping("/services")
public List<String> getServices() {
    return discoveryClient.getServices();
}

@GetMapping("/instances")
public List<ServiceInstance> getInstances(
    @RequestParam String serviceName) {
    return discoveryClient.getInstances(serviceName);
}

// ZookeeperDiscoveryProperties
@Autowired
private ZookeeperDiscoveryProperties discoveryProperties;

// ZookeeperServiceRegistry（手动注册/注销）
@Autowired
private ZookeeperServiceRegistry serviceRegistry;
```

# 10. 监控与运维

```yaml
# 暴露 Actuator 端点用于监控
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

```text
关键监控指标：
┌──────────────────────────────────────────────────────┐
│                                                       │
│  · zookeeper_discovery_instances：注册的实例数        │
│  · curator_connection_state：连接状态                 │
│  · curator_session_state：Session 状态                │
│  · curator_cache_size：Cache 大小                     │
│  · curator_pending_tasks：待处理任务数                │
│  · curator_operations_total：操作统计                 │
│                                                       │
│  健康检查端点：                                       │
│    GET /actuator/health                               │
│    返回：zookeeper 连接状态 + discovery 状态           │
│                                                       │
└──────────────────────────────────────────────────────┘
```

# 11. 与 Nacos / Eureka 对比

| 特性 | Spring Cloud ZK | Spring Cloud Nacos | Spring Cloud Eureka |
|------|----------------|-------------------|-------------------|
| **一致性模型** | CP | CP + AP | AP |
| **健康检查** | Session 心跳 + Actuator | 主动探测 | 客户端心跳 |
| **动态配置** | 支持（需额外配置） | 原生支持 | 不支持 |
| **CAP 代价** | Leader 选举期不可用 | AP 模式无，CP 模式短暂不可用 | 无（但数据不一致） |
| **多数据中心** | 不支持 | 支持 | 支持 |
| **Web UI** | 无自带（需第三方） | 自带 | 自带 |
| **Spring Cloud 集成** | 良好 | 原生（Alibaba） | 原生 |
| **流行度** | 中（Dubbo 广泛使用） | 高 | 中低（停维） |
