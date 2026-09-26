
# 1. 服务发现概述

服务发现是指服务消费者如何获知服务提供者的网络位置（IP:Port）。Nacos 支持两种服务发现模式：

1. **主动拉取（Pull）**：客户端定时从服务端拉取全量/增量服务列表
2. **被动推送（Push）**：服务端在服务实例发生变化时主动推送给订阅者

# 2. 服务发现 API

## 2.1 REST API

```bash
# 查询服务实例列表
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=order-service'

# 查询健康实例
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=order-service&healthyOnly=true'

# 订阅服务
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=order-service&clusters=SH'
```

## 2.2 Java SDK

```java
// 1. 获取全部实例
NacosNamingService naming = NacosNamingServiceFactory.createNamingService("127.0.0.1:8848");
List<Instance> instances = naming.getAllInstances("order-service");

// 2. 获取健康实例（带分组和集群）
List<Instance> healthyInstances = naming.selectInstances("order-service", "DEFAULT_GROUP", true);

// 3. 仅获取健康的单个实例（内置负载均衡）
Instance instance = naming.selectOneHealthyInstance("order-service");

// 4. 订阅服务变化
naming.subscribe("order-service", new EventListener() {
    @Override
    public void onEvent(Event event) {
        NamingEvent namingEvent = (NamingEvent) event;
        System.out.println("Service changed: " + namingEvent.getServiceName());
        System.out.println("Instances: " + namingEvent.getInstances());
    }
});

// 5. 取消订阅
naming.unsubscribe("order-service", eventListener);
```

## 2.3 Spring Cloud 集成

```java
@RestController
public class DiscoveryController {

    @Autowired
    private DiscoveryClient discoveryClient;

    @GetMapping("/services")
    public List<String> getServices() {
        return discoveryClient.getServices();
    }

    @GetMapping("/instances/{serviceId}")
    public List<ServiceInstance> getInstances(@PathVariable String serviceId) {
        return discoveryClient.getInstances(serviceId);
    }
}
```

# 3. 服务发现机制详解

## 3.1 整体发现流程

```
┌────────────┐                              ┌──────────────┐
│  Consumer  │                              │ Nacos Server │
│  App       │                              │              │
│            │                              │              │
│  ┌───────┐ │  1. subscribe(serviceName)   │ ┌──────────┐ │
│  │ Sub   │ │─────────────────────────────>│ │ Service  │ │
│  │ scriber│ │                              │ │ Registry │ │
│  └───────┘ │  2. 返回当前实例列表         │ └──────────┘ │
│            │<─────────────────────────────│              │
│  ┌───────┐ │                              │ ┌──────────┐ │
│  │ Cache │ │  3. 缓存到本地               │ │ Push     │ │
│  │       │ │                              │ │ Service  │ │
│  └───────┘ │                              │ └──────────┘ │
│            │  4. 实例变更时推送/长轮询通知  │              │
│            │<─────────────────────────────│              │
│            │                              │              │
│            │  5. 拉取最新实例列表          │              │
│            │─────────────────────────────>│              │
│            │<─────────────────────────────│              │
└────────────┘                              └──────────────┘
```

## 3.2 客户端订阅流程（源码分析）

```java
// NacosNamingService.java
@Override
public void subscribe(String serviceName, String groupName, List<String> clusters, 
                       EventListener listener) throws NacosException {
    // 1. 创建订阅信息
    String serviceNameWithGroup = NamingUtils.getGroupedName(serviceName, groupName);
    
    // 2. 注册事件监听器
    eventDispatcher.addListener(host, serviceNameWithGroup, listener);
    
    // 3. 第一步先拉取一次全量实例
    serverProxy.queryList(serviceNameWithGroup, clusters, 0, false);
    
    // 4. 启动定时拉取任务（默认 3s 间隔）
    hostReactor.subscribe(serviceNameWithGroup, clusters);
}
```

## 3.3 服务端订阅处理

```java
// SubscribeManager.java
public void subscribe(Service service, Subscriber subscriber) {
    // 1. 记录订阅关系
    subscriberMap.computeIfAbsent(service, k -> new ConcurrentHashSet<>()).add(subscriber);
    
    // 2. 将订阅者的 UDP 端口记录到 PushService
    pushService.addClient(subscriber);
    
    // 3. 立即推送一次当前服务实例列表
    pushService.push(subscriber, service);
}
```

# 4. 实例变更推送机制

## 4.1 Nacos 1.x 推送模型（UDP + HTTP）

```
Nacos Server                         Consumer Client
     │                                      │
     │  InstanceChanged                    │
     │──────┐                              │
     │<─────┘                              │
     │                                      │
     │  UDP Push (端口: 随机)              │
     │─────────────────────────────────────>│ 收到 UDP 通知
     │                                      │
     │  HTTP GET /instance/list            │
     │<─────────────────────────────────────│ 拉取最新数据
     │  返回最新实例列表                    │
     │─────────────────────────────────────>│
     │                                      │ 更新本地缓存
```

## 4.2 Nacos 2.x 推送模型（gRPC Stream）

```
Nacos Server                         Consumer Client
     │                                      │
     │  InstanceChanged                     │
     │──────┐                              │
     │<─────┘                              │
     │                                      │
     │  gRPC Bi-directional Stream          │
     │  NotifySubscriberRequest             │
     │─────────────────────────────────────>│ 收到变更通知
     │                                      │
     │  gRPC 增量/全量数据                  │ 更新本地缓存
     │<─────────────────────────────────────│
```

# 5. 本地缓存机制

## 5.1 缓存设计

Nacos 客户端在本地维护了一份服务实例的缓存，确保即使在 Nacos 服务端不可用的情况下，客户端仍然可以使用缓存数据继续服务发现。

```java
// HostReactor.java
public class HostReactor {
    
    // 核心缓存结构
    private Map<String, ServiceInfo> serviceInfoMap;
    
    // 定时更新任务（延迟 1s，周期 3s）
    private ScheduledExecutorService executor;
    
    // 故障转移缓存（写入磁盘）
    private String cacheDir;
    
    public HostReactor(NamingProxy serverProxy) {
        this.serverProxy = serverProxy;
        this.serviceInfoMap = new ConcurrentHashMap<>();
        this.executor = new ScheduledThreadPoolExecutor(2, ...);
        this.cacheDir = System.getProperty("user.home") + "/nacos/naming/";
    }
}
```

## 5.2 数据失效与保护机制

```java
// 获取服务信息（带失效保护）
public ServiceInfo getServiceInfo(String serviceName, String clusters) {
    String key = ServiceInfo.getKey(serviceName, clusters);
    ServiceInfo serviceInfo = serviceInfoMap.get(key);
    
    if (serviceInfo == null) {
        // 1. 内存缓存未命中 → 读取磁盘缓存
        serviceInfo = readDiskCache(key);
        if (serviceInfo != null) {
            serviceInfoMap.put(key, serviceInfo);
        }
    }
    
    if (serviceInfo == null) {
        // 2. 磁盘缓存也未命中 → 从服务端拉取
        serviceInfo = updateServiceNow(serviceName, clusters);
    }
    
    // 3. 定时异步更新
    executor.schedule(new UpdateTask(serviceName, clusters), 0, TimeUnit.SECONDS);
    
    return serviceInfo;
}

// 磁盘缓存读取
private ServiceInfo readDiskCache(String key) {
    File cacheFile = new File(cacheDir, key);
    if (cacheFile.exists()) {
        String content = FileUtils.readFileToString(cacheFile, "UTF-8");
        return JSON.parseObject(content, ServiceInfo.class);
    }
    return null;
}
```

## 5.3 缓存文件结构

```
~/
└── nacos/
    └── naming/
        ├── order-service@@DEFAULT_GROUP@@SH
        ├── user-service@@DEFAULT_GROUP@@DEFAULT
        └── payment-service@@CUSTOM_GROUP@@BJ
```

每个文件内容为 JSON 格式的 `ServiceInfo`：

```json
{
  "name": "DEFAULT_GROUP@@order-service",
  "clusters": "SH",
  "cacheMillis": 10000,
  "hosts": [
    {
      "instanceId": "192.168.1.1#8080#SH#order-service",
      "ip": "192.168.1.1",
      "port": 8080,
      "weight": 1.0,
      "healthy": true,
      "enabled": true,
      "ephemeral": true,
      "clusterName": "SH",
      "metadata": {}
    }
  ],
  "lastRefTime": 1700000000000,
  "checksum": "abc123def456",
  "allIPs": false,
  "reachProtectionThreshold": false
}
```

# 6. 负载均衡集成

## 6.1 Nacos + Spring Cloud LoadBalancer

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    loadbalancer:
      nacos:
        enabled: true    # 启用 Nacos 权重负载均衡
```

```java
// 权重负载均衡策略
public class NacosLoadBalancer implements ReactorServiceInstanceLoadBalancer {
    
    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
        
        // 根据 Nacos 权重过滤和选择
        List<ServiceInstance> weightedInstances = instances.stream()
            .filter(this::isHealthy)
            .sorted((a, b) -> Double.compare(getWeight(b), getWeight(a)))
            .collect(Collectors.toList());
        
        // 加权随机算法
        double totalWeight = weightedInstances.stream()
            .mapToDouble(this::getWeight).sum();
        double random = ThreadLocalRandom.current().nextDouble(totalWeight);
        
        for (ServiceInstance instance : weightedInstances) {
            random -= getWeight(instance);
            if (random <= 0) {
                return Mono.just(new DefaultResponse(instance));
            }
        }
        return Mono.empty();
    }
    
    private double getWeight(ServiceInstance instance) {
        return Double.parseDouble(instance.getMetadata()
            .getOrDefault("nacos.weight", "1.0"));
    }
}
```

## 6.2 集群优先调用

```yaml
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: SH   # 标记自己属于 SH 集群
```

```java
// 同集群优先的负载均衡
@Bean
public ReactorLoadBalancer<ServiceInstance> loadBalancer(
        Environment environment, LoadBalancerClientFactory factory) {
    
    String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    
    return new NacosLoadBalancer(factory.getLazyProvider(name, ServiceInstanceListSupplier.class), 
        name,
        // 集群优先
        new NacosLoadBalancerConfiguration.ClusterFirstRule());
}
```

# 7. 服务发现性能优化

## 7.1 订阅 vs 定时拉取

| 方式 | 实时性 | 资源消耗 | 适用场景 |
|------|--------|---------|---------|
| 订阅推送 | 高（秒级） | 长连接 | 核心业务服务 |
| 定时拉取 | 中（3s 延迟） | 低 | 非核心服务 |
| 按需拉取 | 低（调用时） | 最低 | 低频调用服务 |

## 7.2 增量更新优化

Nacos 2.x 支持增量更新，服务端只推送变化的实例，大幅减少网络数据传输。

```java
// NamingPushRequestHandler.java (服务端)
@Override
public void handle(Service service, Service changedService) {
    // 计算增量
    List<Instance> addedList = new ArrayList<>();
    List<Instance> removedList = new ArrayList<>();
    
    for (Instance i : changedService.allIPs()) {
        if (!service.allIPs().contains(i)) {
            addedList.add(i); // 新增实例
        }
    }
    
    for (Instance i : service.allIPs()) {
        if (!changedService.allIPs().contains(i)) {
            removedList.add(i); // 移除实例
        }
    }
    
    // 推送增量变更
    NotifyCenter.publishEvent(new InstanceDeltaChangeEvent(addedList, removedList));
}
```

# 8. 故障转移与容灾

## 8.1 故障转移流程

```java
// HostReactor.java
private ServiceInfo getServiceInfo0(String serviceName, String clusters) {
    try {
        // 1. 尝试从服务端获取
        ServiceInfo serviceInfo = serverProxy.queryList(serviceName, clusters, ...);
        if (serviceInfo != null) {
            return serviceInfo;
        }
    } catch (NacosException e) {
        log.error("Failed to query service from Nacos Server", e);
    }
    
    // 2. 服务端不可用 → 读取本地磁盘缓存
    ServiceInfo cachedInfo = readDiskCache(key);
    if (cachedInfo != null) {
        log.warn("Using local disk cache for service: {}", serviceName);
        return cachedInfo;
    }
    
    // 3. 无任何缓存 → 返回空实例列表（不抛异常，保证可用性）
    return new ServiceInfo(serviceName, clusters);
}
```

## 8.2 保护阈值

当健康实例数占总实例数的比例低于保护阈值时，Nacos 会将所有实例（包括不健康实例）返回给客户端，防止因网络波动导致大规模剔除问题。

```properties
# 设置保护阈值（默认 0，即不启用）
nacos.naming.protect.threshold=0.0
```

```java
// Service.java
public boolean reachProtectionThreshold() {
    return (healthyInstanceCount / totalInstanceCount) < protectThreshold;
}
```

# 9. 全链路服务发现示例

```java
@Service
public class OrderService {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Autowired
    private RestTemplate restTemplate;

    public String createOrder() {
        // 1. 服务发现（支持集群优先 + 权重）
        List<ServiceInstance> instances = discoveryClient
            .getInstances("user-service");
        
        if (instances.isEmpty()) {
            throw new RuntimeException("No available user-service instance");
        }

        // 2. 负载均衡选择实例
        ServiceInstance instance = loadBalance(instances);

        // 3. 发起调用
        String url = "http://" + instance.getHost() + ":" + instance.getPort() + "/user/info";
        return restTemplate.getForObject(url, String.class);
    }

    private ServiceInstance loadBalance(List<ServiceInstance> instances) {
        // 简单轮询实现
        instances.sort(Comparator.comparing(ServiceInstance::getInstanceId));
        int index = Math.abs(counter.incrementAndGet() % instances.size());
        return instances.get(index);
    }
}
```
