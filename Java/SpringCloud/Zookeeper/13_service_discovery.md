
# 1. 服务注册与发现概述

服务注册与发现是微服务架构中最核心的基础设施之一。ZooKeeper 凭借其**临时节点**和**Watch 机制**，天然适合作为服务注册中心。

```text
服务注册与发现的三个核心问题：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 服务注册：Provider 如何把自己的地址告诉注册中心                   |
|     方案：Provider 在 ZK 上创建临时节点                               |
|                                                                       |
|  2. 服务发现：Consumer 如何找到 Provider                              |
|     方案：Consumer 通过 ZK 获取 Provider 列表 + Watch 动态更新        |
|                                                                       |
|  3. 健康检查：如何感知 Provider 下线                                  |
|     方案：临时节点随 Session 断开自动删除 + Watch 通知 Consumer       |
|                                                                       |
+----------------------------------------------------------------------+
```

# 2. ZooKeeper 注册模型

## 2.1 经典的 Dubbo 注册模型

Dubbo 是使用 ZK 作为注册中心的典型代表，其注册模型非常经典：

```text
Dubbo 在 ZooKeeper 中的注册结构：

/dubbo
  /com.example.OrderService (接口全限定名)
    /providers (持久节点)
      /dubbo%3A%2F%2F192.168.1.10%3A20880%2F... (临时节点，URL 编码)
      /dubbo%3A%2F%2F192.168.1.11%3A20880%2F... (临时节点)
    /consumers (持久节点)
      /consumer%3A%2F%2F192.168.1.20%2F... (临时节点)
    /routers (持久节点)
    /configurators (持久节点)
```

## 2.2 通用注册模型

```text
通用微服务注册模型：

/services (持久节点)
  /order-service (持久节点，服务名)
    /instance-{host}-{port}-{uuid} (临时节点，实例信息)
      data: {
        "name": "order-service",
        "host": "192.168.1.10",
        "port": 8081,
        "metadata": {
          "version": "1.0.0",
          "region": "cn-beijing",
          "weight": 100
        }
      }
    /instance-192.168.1.11-8081-abc123 (临时节点)
    /instance-192.168.1.12-8081-def456 (临时节点)
  /user-service
    /instance-192.168.1.20-8082-ghi789
    /instance-192.168.1.21-8082-jkl012
```

# 3. 服务注册流程

```text
Provider 注册流程（以 Spring Cloud 为例）：

1. Provider 启动
       |
2. 与 ZooKeeper 建立连接（CuratorFramework）
       |
3. 构造 ServiceInstance 对象
   - serviceId: "order-service"
   - host: 本机 IP
   - port: 服务端口
   - metadata: {version, region, ...}
       |
4. 向 ZK 注册（创建临时节点）
   /services/order-service/instance-{id}
       |
5. 注册成功，对外提供服务
       |
6. Provider 进程退出或与 ZK 断连
       |
7. Session 超时 -> 临时节点自动删除
       |
8. Consumer 通过 Watch 感知实例下线
```

## 3.1 Spring Cloud 自动注册

```java
// Spring Cloud 自动注册过程（框架自动完成，无需手动编写）
// 核心类：ZookeeperAutoServiceRegistration
@Configuration
public class AutoRegistrationConfig {
    // 框架自动完成：
    // 1. 应用启动时触发 WebServerInitializedEvent
    // 2. ZookeeperAutoServiceRegistration 监听到该事件
    // 3. 调用 ServiceRegistry.register(registration)
    // 4. 底层通过 Curator 创建临时节点
}
```

## 3.2 手动注册示例

```java
// 使用 Curator 手动注册服务
@Service
public class ServiceRegistrar {
    private final CuratorFramework client;

    public ServiceRegistrar(CuratorFramework client) {
        this.client = client;
    }

    public void register(String serviceName, String host,
                         int port, Map<String, String> metadata)
            throws Exception {
        // 确保服务根路径存在
        String servicePath = "/services/" + serviceName;
        if (client.checkExists().forPath(servicePath) == null) {
            client.create()
                .creatingParentsIfNeeded()
                .forPath(servicePath);
        }

        // 构造实例信息（JSON 格式）
        ServiceInstanceInfo instance = new ServiceInstanceInfo();
        instance.setName(serviceName);
        instance.setHost(host);
        instance.setPort(port);
        instance.setMetadata(metadata);

        String instanceJson = new ObjectMapper()
            .writeValueAsString(instance);

        // 创建临时节点
        String instancePath = servicePath + "/instance-" +
            host + "-" + port;
        client.create()
            .withMode(CreateMode.EPHEMERAL)
            .forPath(instancePath, instanceJson.getBytes());

        System.out.println("Registered: " + instancePath);
    }
}
```

# 4. 服务发现流程

```text
Consumer 发现流程：

1. Consumer 启动
       |
2. 从 ZK 获取指定服务的所有实例
   getChildren("/services/order-service")
       |
3. 注册 Watch 监听实例变化
       |
4. 选择负载均衡策略选择一个实例
       |
5. 发起 RPC 调用
       |
6. 实例下线（临时节点删除）
       |
7. Watch 触发通知 Consumer
       |
8. Consumer 更新本地实例列表
   重新负载均衡
```

## 4.1 Spring Cloud 自动发现

```java
// Spring Cloud DiscoveryClient 自动注入
@Autowired
private DiscoveryClient discoveryClient;

public List<ServiceInstance> getOrderServices() {
    return discoveryClient.getInstances("order-service");
}
```

## 4.2 手动发现示例

```java
// 使用 Curator 手动发现服务
@Component
public class ServiceDiscovery {
    private final CuratorFramework client;
    private final ConcurrentMap<String, List<ServiceInstanceInfo>>
        serviceCache = new ConcurrentHashMap<>();

    public ServiceDiscovery(CuratorFramework client) {
        this.client = client;
    }

    // 发现服务并注册 Watch 监听
    public List<ServiceInstanceInfo> discover(String serviceName)
            throws Exception {
        String servicePath = "/services/" + serviceName;
        List<String> children = client.getChildren().forPath(servicePath);

        List<ServiceInstanceInfo> instances = new ArrayList<>();
        for (String child : children) {
            String path = servicePath + "/" + child;
            byte[] data = client.getData().forPath(path);
            ServiceInstanceInfo instance = parseInstance(data);
            instances.add(instance);
        }

        serviceCache.put(serviceName, instances);
        return instances;
    }

    // 使用 PathChildrenCache 自动监听实例变化
    public void watchService(String serviceName) throws Exception {
        String servicePath = "/services/" + serviceName;
        PathChildrenCache cache = new PathChildrenCache(
            client, servicePath, true);
        cache.getListenable().addListener(
            (client, event) -> {
            switch (event.getType()) {
                case CHILD_ADDED:
                    System.out.println("Instance added: " +
                        event.getData().getPath());
                    break;
                case CHILD_REMOVED:
                    System.out.println("Instance removed: " +
                        event.getData().getPath());
                    break;
                case CHILD_UPDATED:
                    System.out.println("Instance updated: " +
                        event.getData().getPath());
                    break;
            }
            // 更新本地缓存
            refreshLocalCache(serviceName);
        });
        cache.start(StartMode.BUILD_INITIAL_CACHE);
    }
}
```

# 5. 客户端负载均衡

```text
负载均衡策略（客户端侧）：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 简单随机（Random）                                                |
|     · 从实例列表随机选一个                                            |
|     · 实现简单，适用于各实例性能相同的场景                            |
|                                                                       |
|  2. 轮询（Round Robin）                                               |
|     · 按顺序轮流访问                                                  |
|     · 适用于各实例请求处理耗时相似的场景                              |
|                                                                       |
|  3. 加权轮询（Weighted Round Robin）                                  |
|     · 根据实例权重分配请求比例                                        |
|     · 权重信息存储在实例 metadata 中                                  |
|                                                                       |
|  4. 最小连接数（Least Connections）                                   |
|     · 选择当前连接数最少的实例                                        |
|                                                                       |
|  5. 一致性哈希（Consistent Hash）                                     |
|     · 相同请求路由到相同实例（有状态服务）                            |
|                                                                       |
+----------------------------------------------------------------------+
```

```java
// Spring Cloud LoadBalancer 集成
// 自动注入 LoadBalancerClient
@Autowired
private LoadBalancerClient loadBalancerClient;

@Bean
@LoadBalanced  // 启用负载均衡拦截器
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 使用服务名调用（自动负载均衡）
String result = restTemplate.getForObject(
    "http://order-service/api/order/1", String.class);
```

# 6. 健康检查策略

| 方式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **Session 心跳** | 临时节点随 Session 删除 | 无需额外配置，天然支持 | 仅能检测进程级故障 |
| **HTTP 健康检查** | 定期调用 /health 端点 | 能检测应用级故障 | 需额外配置 |
| **TCP 端口探测** | 检测端口是否可达 | 简单 | 无法检测应用假死 |
| **主动上报** | 实例定时上报健康状态 | 实时性好 | 增加网络开销 |

```java
// Spring Cloud + Actuator 健康检查
// 自动集成，配置如下：
// spring.cloud.zookeeper.discovery.health-check-enabled=true
```

# 7. ZK 作为注册中心的优缺点

## 7.1 优点

| 优点 | 说明 |
|------|------|
| **强一致性** | 基于 ZAB 协议，实例列表不会出现不一致 |
| **实时性** | Watch 机制推送变更，延迟低（< 1s） |
| **自动清理** | 临时节点自动删除，天然支持实例下线检测 |
| **高可用** | ZK 集群自身高可用 |
| **生态成熟** | Dubbo、Kafka 等广泛使用，久经考验 |

## 7.2 缺点

| 缺点 | 说明 | 应对措施 |
|------|------|---------|
| **CP 代价** | Leader 选举期间不可用（秒级） | 使用 Observer 提升读可用性 |
| **连接风暴** | 大量 Consumer 同时 Watch 同一个 provider 目录 | 使用 Curator 处理 |
| **不支持多 DC** | ZK 集群不支持跨数据中心 | 使用多套 ZK + 同步机制 |
| **容量限制** | ZK 不适合存储大量节点数据 | 控制单集群实例数 < 10万 |
| **无负载均衡** | 仅为注册中心，负载均衡需客户端实现 | 结合 Ribbon/LoadBalancer |

# 8. ZK 注册模式对比其他注册中心

| 特性 | ZooKeeper | Eureka | Nacos | Consul | etcd |
|------|-----------|--------|-------|--------|------|
| CAP | CP | AP | CP+AP | CP(默认) | CP |
| 健康检查 | Session心跳 | 客户端心跳 | TCP/HTTP | 多模式 | Lease租约 |
| Watch | 一次性触发 | 定时拉取 | 长连接推送 | 长连接推送 | Watch |
| 多DC | 不支持 | 支持 | 支持 | 原生支持 | 不支持 |
| 配置管理 | 需二次开发 | 不支持 | 原生支持 | KV存储 | KV存储 |
| 性能 | 读极高/写中等 | 读高/写高 | 读高/写高 | 读高/写低 | 读高/写高 |
| UI | 无 | 有仪表盘 | 有控制台 | 有WebUI | 无 |

# 9. 最佳实践

```text
ZK 服务注册发现最佳实践：

1. 使用临时节点做实例注册
   - 确保实例下线后自动清理
   - 不要用持久节点注册实例

2. 合理设置 Session 超时
   - 太短：网络抖动误下线
   - 太长：故障检测慢
   - 推荐：15s ~ 30s

3. 控制 ZNode 数据量
   - 实例信息尽量精简（几百字节）
   - 大字段（如健康状态详情）存到其他系统

4. Consumer 侧缓存实例列表
   - 启动时全量拉取
   - 通过 Watch 增量更新
   - 不要每次调用都查 ZK

5. 实例权重信息放在 metadata 中
   - 支持异构部署
   - Consumer 从 metadata 中读取权重

6. 使用 namespace 隔离环境
   - /dev/services 和 /prod/services 分开
   - 通过 namespace 配置实现

7. 保护机制
   - 本地缓存兜底（ZK 不可用时用上一次的列表）
   - 添加 ACL 保护敏感路径
```
