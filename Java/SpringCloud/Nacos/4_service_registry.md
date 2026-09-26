
# 1. 服务注册概述

服务注册是微服务架构中最基础的能力。服务提供者启动时，将自己的网络地址（IP:Port）及其他元数据注册到注册中心，以便服务消费者能够发现和调用。

# 2. 注册模型

## 2.1 服务分级模型

```
Namespace (命名空间 / 租户)
  │
  ├── Group (分组，默认 DEFAULT_GROUP)
  │     │
  │     └── Service (服务，由 serviceName 唯一标识)
  │           │
  │           ├── Cluster (集群，如 DEFAULT / SH / BJ)
  │           │     │
  │           │     ├── Instance (实例 1: 192.168.1.1:8080)
  │           │     ├── Instance (实例 2: 192.168.1.2:8080)
  │           │     └── Instance (实例 3: 192.168.1.3:8080)
  │           │
  │           └── Cluster (集群 V2)
  │                 └── Instance ...
  │
  └── Group (自定义分组)
        └── ...
```

## 2.2 实例类型

| 特性 | 临时实例 (Ephemeral) | 持久实例 (Persistent) |
|------|---------------------|----------------------|
| 心跳方式 | 客户端主动上报心跳 | 服务端主动探测 |
| 剔除策略 | 心跳超时自动剔除 | 不会自动剔除，标记不健康 |
| 一致协议 | Distro (AP) | Raft (CP) |
| 典型场景 | 微服务应用实例 | DNS 记录、DB 实例 |
| 生命周期 | 与客户端进程绑定 | 独立于客户端进程 |

# 3. Nacos 服务注册 API

## 3.1 REST API

```bash
# 注册服务实例
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service&ip=192.168.1.100&port=8080'

# 注册持久实例
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service&ip=192.168.1.100&port=8080&ephemeral=false'
```

## 3.2 Java SDK 注册

```java
// 方式一：Nacos SDK 直接注册
NacosNamingService naming = NacosNamingServiceFactory.createNamingService("127.0.0.1:8848");

Instance instance = new Instance();
instance.setIp("192.168.1.100");
instance.setPort(8080);
instance.setWeight(1.0);
instance.setClusterName("DEFAULT");
instance.setMetadata(new HashMap<String, String>() {{
    put("version", "v1.0.0");
    put("region", "shanghai");
}});

naming.registerInstance("order-service", instance);

// 方式二：使用 Builder 模式
naming.registerInstance("order-service", "192.168.1.100", 8080, "DEFAULT");

// 方式三：带分组的注册
naming.registerInstance("order-service", "MY_GROUP", instance);
```

## 3.3 gRPC 注册（Nacos 2.x）

```java
// 2.x 客户端默认使用 gRPC 通信
// 注册流程在底层自动处理，API 保持不变
NacosNamingService naming = NacosNamingServiceFactory.createNamingService(properties);
naming.registerInstance("order-service", "192.168.1.100", 8080);
// 底层实际通过 BiRequestStream 将注册请求发送到服务端
```

# 4. Spring Cloud 集成

## 4.1 依赖配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.0.5.0</version>
</dependency>
```

## 4.2 配置文件

```yaml
spring:
  application:
    name: order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev           # 命名空间（填写 namespace ID）
        group: DEFAULT_GROUP     # 分组
        cluster-name: SH         # 集群名称
        ephemeral: true          # 临时实例（默认）
        weight: 10               # 权重 (1-100)
        metadata:                # 扩展元数据
          version: v1.0.0
          region: shanghai
        ip: 192.168.1.100        # 指定注册 IP（多网卡时有用）
        port: 8080               # 指定注册端口
        enabled: true            # 是否启用服务发现
```

## 4.3 启动类配置

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

# 5. 注册流程源码解析

## 5.1 客户端注册流程

```java
// NacosServiceRegistry.java (spring-cloud-alibaba-nacos-discovery)
@Override
public void register(Registration registration) {
    // 1. 获取实例ID
    String serviceId = registration.getServiceId();
    String group = nacosDiscoveryProperties.getGroup();

    // 2. 构建 Nacos Instance
    Instance instance = getNacosInstanceFromRegistration(registration);

    // 3. 调用 namingService 注册
    try {
        namingService().registerInstance(serviceId, group, instance);
    } catch (Exception e) {
        throw new RuntimeException("Failed to register service at Nacos", e);
    }
    log.info("nacos registry, {} {} registered.", group, serviceId);
}
```

## 5.2 NacosNamingService 核心注册逻辑

```java
// NacosNamingService.java (nacos-client)
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) 
        throws NacosException {
    // 1. 生成唯一 ID（用于心跳）
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
    if (instance.isEphemeral()) {
        // 2. 临时实例：通过心跳续约
        BeatInfo beatInfo = beatReactor.buildBeatInfo(groupedServiceName, instance);
        // 3. 第一步先发送注册请求
        serverProxy.registerService(groupedServiceName, groupName, instance);
        // 4. 启动心跳定时任务
        beatReactor.addBeatInfo(groupedServiceName, beatInfo);
    } else {
        // 持久实例：只注册，不需要心跳
        serverProxy.registerService(groupedServiceName, groupName, instance);
    }
}
```

## 5.3 服务端注册接收

```java
// InstanceController.java (nacos-naming)
@CanDistro
@PostMapping
@Secured(action = ActionTypes.WRITE)
public String register(HttpServletRequest request) throws Exception {
    // 1. 解析请求参数
    String namespaceId = WebUtils.optional(request, "namespaceId", Constants.DEFAULT_NAMESPACE_ID);
    String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    
    // 2. 构建实例对象
    Instance instance = parseInstance(request);
    
    // 3. 调用服务管理器注册
    serviceManager.registerInstance(namespaceId, serviceName, instance);
    
    return "ok";
}
```

## 5.4 ServiceManager 注册处理

```java
// ServiceManager.java (nacos-naming)
public void registerInstance(String namespaceId, String serviceName, Instance instance) {
    // 1. 创建空服务（如果不存在）
    createEmptyService(namespaceId, serviceName, instance.isEphemeral());
    
    // 2. 获取服务对象
    Service service = getService(namespaceId, serviceName);
    
    // 3. 添加实例到服务
    addInstance(namespaceId, serviceName, instance.isEphemeral(), instance);
    
    // 4. 发布实例变更事件
    NotifyCenter.publishEvent(new InstanceChangeEvent(serviceName, namespaceId));
}

public void addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips) {
    // 构建 Key（namespace + service）
    String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);
    
    // 获取已存在的实例
    Service service = getService(namespaceId, serviceName);
    
    synchronized (service) {
        List<Instance> instanceList = new ArrayList<>();
        
        // 将 IP 列表添加到实例表
        for (Instance ip : ips) {
            // 检查是否已存在
            Instance oldInstance = service.getInstance(ip);
            if (oldInstance != null) {
                // 更新已有实例
                oldInstance.copy(ip);
            } else {
                // 新增实例
                instanceList.add(ip);
            }
        }
        service.addInstances(instanceList);
    }
    
    // 通过一致性协议同步到其他节点
    consistencyService.put(key, service);
}
```

# 6. 注册流程时序图

```
ClientApp          NacosClient        NacosServer        OtherNodes
   │                    │                   │                 │
   │  registerInstance  │                   │                 │
   │───────────────────>│                   │                 │
   │                    │                   │                 │
   │                    │ POST /nacos/v1/ns/instance          │
   │                    │───────────────────>│                 │
   │                    │                   │                 │
   │                    │                   │ 写入内存注册表  │
   │                    │                   │──────┐          │
   │                    │                   │<─────┘          │
   │                    │                   │                 │
   │                    │                   │ Distro 异步同步  │
   │                    │                   │────────────────>│
   │                    │                   │                 │
   │                    │                   │ 发布变更事件    │
   │                    │                   │──────┐          │
   │                    │                   │<─────┘          │
   │                    │                   │                 │
   │                    │     200 OK        │                 │
   │                    │<──────────────────│                 │
   │                    │                   │                 │
   │  注册成功          │                   │                 │
   │<────────────────── │                   │                 │
   │                    │                   │                 │
   │                    │  心跳 (5s 间隔)    │                 │
   │                    │───────────────────>│                 │
```

# 7. 注册的关键设计

## 7.1 双注册策略（心跳 + 初次注册）

Nacos 客户端将注册和心跳视为两个独立的操作：

1. **注册请求**：客户端启动时发送一次完整注册请求
2. **心跳请求**：定时发送轻量级心跳包（仅包含 IP:Port）

心跳不仅是健康检查手段，也承担着**续约注册**的职责——服务端如果意外丢失某个实例的数据（如宕机恢复），可通过心跳信息重建实例。

## 7.2 幂等性保证

注册操作是幂等的——同一个实例多次注册不会产生重复数据。通过 `instanceId`（默认由 IP:Port 和 Cluster 组合）作为唯一标识：

```java
// InstanceIdGenerator.java
public String generateInstanceId(Instance instance) {
    // 默认：IP#Port#ClusterName#ServiceName
    return instance.getIp() + "#" + instance.getPort() 
        + "#" + instance.getClusterName() + "#" + instance.getServiceName();
}
```

## 7.3 注册表内存数据结构

```java
// ServiceManager 的核心数据结构
Map<String, Service> serviceMap = new ConcurrentHashMap<>();
// Key: com.alibaba.nacos.naming.iplist.ephemeral.{namespaceId}##{serviceName}
//     com.alibaba.nacos.naming.iplist.{namespaceId}##{serviceName}

// Service 内部结构
public class Service {
    private Map<String, Cluster> clusterMap = new ConcurrentHashMap<>();
}

// Cluster 内部结构
public class Cluster {
    // 持久实例（CP 模式）
    private Set<Instance> persistentInstances = new HashSet<>();
    // 临时实例（AP 模式）
    private Set<Instance> ephemeralInstances = new HashSet<>();
}
```

# 8. 注册参数详解

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| serviceName | String | 是 | - | 服务名称，全局唯一标识 |
| ip | String | 是 | 自动获取 | 实例 IP 地址 |
| port | int | 是 | - | 实例端口 |
| namespaceId | String | 否 | public | 命名空间 ID |
| groupName | String | 否 | DEFAULT_GROUP | 分组名称 |
| clusterName | String | 否 | DEFAULT | 集群名称 |
| weight | double | 否 | 1.0 | 权重 (1-100) |
| healthy | boolean | 否 | true | 健康状态 |
| enabled | boolean | 否 | true | 是否接收请求 |
| ephemeral | boolean | 否 | true | 是否临时实例 |
| metadata | Map | 否 | {} | 自定义元数据 |

# 9. 注册最佳实践

## 9.1 合理设置权重

```java
// 根据服务器配置差异化设置权重
// 4C8G 机器权重 10，2C4G 机器权重 5
instance.setWeight(10.0);
```

## 9.2 利用元数据做流量治理

```java
// 版本标识
metadata.put("version", "v2.0.0");
// 机房/地域标识
metadata.put("zone", "zone-a");
metadata.put("region", "shanghai");
// 自定义标签
metadata.put("canary", "true"); // 金丝雀标识
```

## 9.3 优雅上下线

```java
@Component
public class GracefulShutdown {
    
    @Autowired
    private NacosAutoServiceRegistration serviceRegistration;
    
    @PreDestroy
    public void destroy() {
        // 1. 先从 Nacos 摘除自己（不再接收新请求）
        serviceRegistration.stop();
        
        // 2. 等待存量请求处理完毕
        try {
            Thread.sleep(15000); // 15s 排水时间
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // 3. 关闭 Spring 容器
    }
}
```

## 9.4 多网卡环境指定 IP

```yaml
spring:
  cloud:
    nacos:
      discovery:
        ip: ${MY_POD_IP:}    # K8s Pod IP
        network-interface: eth0  # 指定网卡
```
