
# 1. 注册中心概述

注册中心是 Dubbo 服务治理的枢纽，负责服务地址的注册与发现。Provider 上线时注册服务地址，Consumer 启动时订阅服务地址，Provider 下线时自动注销。

```
Provider ──register──→  Registry Center (Nacos/Zookeeper/Redis)  ←──subscribe── Consumer
                           │
                     ┌─────┴─────┐
                     ▼           ▼
                Fail/Off     Notify Change
                → 自动摘除    → Consumer 更新本地缓存
```

# 2. 注册中心配置

## 2.1 单注册中心

```yaml
dubbo:
  registry:
    address: nacos://127.0.0.1:8848
    parameters:
      namespace: dev
```

## 2.2 多注册中心

```yaml
dubbo:
  registry:
    # 注册中心 1：Nacos 集群
    nacos-registry:
      address: nacos://10.0.0.1:8848
      parameters:
        namespace: prod
    # 注册中心 2：Zookeeper 集群（灾备）
    zk-registry:
      address: zookeeper://10.0.0.2:2181
      default: false
```

Provider 向所有注册中心注册，Consumer 从所有注册中心订阅。

## 2.3 指定服务注册到特定注册中心

```java
@DubboService(registry = "nacos-registry")
public class OrderServiceImpl implements OrderService {
}
```

# 3. Nacos 集成

Nacos（Dynamic Naming and Configuration Service）是 Dubbo 3 推荐的注册中心，支持 AP/CP 模式切换。

## 3.1 依赖

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-registry-nacos</artifactId>
</dependency>
```

## 3.2 配置

```yaml
dubbo:
  registry:
    address: nacos://127.0.0.1:8848?namespace=dev&group=DEFAULT_GROUP
    parameters:
      namespace: dev              # Nacos 命名空间（隔离环境）
      group: DEFAULT_GROUP        # Nacos 分组
  application:
    name: user-service
    # Dubbo 3 默认启用应用级服务发现
    register-mode: instance       # instance=应用级, interface=接口级, all=双注册
```

## 3.3 注册数据结构

**接口级（Dubbo 2.x）**：

```
服务名: providers:org.example.UserService:1.0.0:dev
内容: dubbo://192.168.1.100:20880/org.example.UserService?...
```

**应用级（Dubbo 3.x 默认）**：

```
服务名: instances:user-service
内容: {
  "name": "user-service",
  "id": "192.168.1.100:20880",
  "address": "192.168.1.100",
  "port": 20880,
  "metadata": {
    "dubbo.endpoints": "[{\"port\":20880,\"protocol\":\"dubbo\"}]",
    "dubbo.metadata-service.url": "...",
    "dubbo.metadata.revision": "..."
  }
}
```

应用级发现的优势：单个应用注册一个实例，不与接口数量挂钩，注册中心压力大幅降低。

## 3.4 NacosRegistry 核心源码

```java
public class NacosRegistry extends FailbackRegistry {

    @Override
    public void doRegister(URL url) {
        String serviceName = getServiceName(url);
        String instanceId = getInstanceId(url);
        Instance instance = createInstance(url);
        namingService.registerInstance(serviceName, instance);
    }

    @Override
    public void doSubscribe(URL url, NotifyListener listener) {
        String serviceName = getServiceName(url);
        namingService.subscribe(serviceName, event -> {
            List<Instance> instances = event.getInstances();
            List<URL> urls = instancesToUrls(instances);
            notify(url, listener, urls);
        });
    }

    @Override
    public void doUnregister(URL url) {
        String serviceName = getServiceName(url);
        String instanceId = getInstanceId(url);
        namingService.deregisterInstance(serviceName, instanceId);
    }
}
```

# 4. Zookeeper 集成

## 4.1 依赖

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-registry-zookeeper</artifactId>
</dependency>
```

## 4.2 配置

```yaml
dubbo:
  registry:
    address: zookeeper://127.0.0.1:2181
    timeout: 30000
    parameters:
      group: dubbo
```

## 4.3 ZK 节点结构

```
/dubbo/                                    ← 根节点
  └── org.example.UserService/             ← 服务接口
      ├── providers/                        ← 提供者目录
      │   ├── dubbo://192.168.1.100:20880/...  (临时节点)
      │   └── dubbo://192.168.1.101:20880/...  (临时节点)
      ├── consumers/                        ← 消费者目录
      │   └── consumer://192.168.2.100/...   (临时节点)
      ├── routers/                          ← 路由规则
      │   └── condition://...
      └── configurators/                    ← 动态配置
          └── override://...
```

Provider 在 ZK 上创建**临时节点**，Session 断开后自动删除，实现自动摘除。

## 4.4 ZookeeperRegistry 核心源码

```java
public class ZookeeperRegistry extends FailbackRegistry {

    @Override
    public void doRegister(URL url) {
        zkClient.create(toUrlPath(url), url.getParameters(), 
            CreateMode.EPHEMERAL);  // 临时节点
    }

    @Override
    public void doSubscribe(URL url, NotifyListener listener) {
        // 监听 providers 目录
        List<URL> urls = new ArrayList<>();
        for (String path : toCategoriesPath(url)) {
            List<String> children = zkClient.addChildListener(path, (parent, current) -> {
                // 节点变化 → 刷新并通知
                notify(url, listener, toUrlsWithEmpty(url, path, current));
            });
            urls.addAll(toUrlsWithEmpty(url, path, children));
        }
        notify(url, listener, urls);
    }
}
```

# 5. 注册中心架构

## 5.1 抽象层设计

```
RegistryService (接口)
    └── AbstractRegistry (抽象基类，本地文件缓存)
         └── FailbackRegistry (失败重试机制)
              ├── NacosRegistry
              ├── ZookeeperRegistry
              ├── RedisRegistry
              ├── EtcdRegistry
              └── ...
```

## 5.2 FailbackRegistry —— 失败重试

```java
public abstract class FailbackRegistry extends AbstractRegistry {

    // 注册失败的 URL 集合（定时重试）
    private final Set<URL> failedRegistered = new ConcurrentHashSet<>();
    private final Set<URL> failedUnregistered = new ConcurrentHashSet<>();
    private final Set<URL> failedSubscribed = new ConcurrentHashSet<>();

    public FailbackRegistry(URL url) {
        super(url);
        // 定时任务：每 5 秒重试失败的操作
        retryExecutor.scheduleWithFixedDelay(() -> {
            retry(); // 遍历失败集合并重试
        }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
    }

    @Override
    public void register(URL url) {
        super.register(url);
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            doRegister(url);
        } catch (Exception e) {
            // 注册失败加入重试队列
            failedRegistered.add(url);
        }
    }
}
```

## 5.3 AbstractRegistry —— 本地文件缓存

```java
public abstract class AbstractRegistry implements RegistryService {
    // 磁盘缓存文件路径: ~/.dubbo/dubbo-registry-{app}-{addr}.cache
    private final Properties properties = new Properties();
    private final File file;

    protected void saveProperties(URL url) {
        // 持久化到磁盘，防止注册中心宕机后本地无数据
        properties.setProperty(url.getServiceKey(), URL.encode(url.toFullString()));
        FileLock lock = fileLock.tryLock();
        try {
            properties.store(new FileOutputStream(file), "Dubbo Registry Cache");
        } finally {
            lock.release();
        }
    }

    public void recover() {
        // 启动时先从本地文件恢复
        FileLock lock = fileLock.tryLock();
        try {
            properties.load(new FileInputStream(file));
        } finally {
            lock.release();
        }
        // 恢复为 URL 列表，用于首次调用
    }
}
```

# 6. 应用级 vs 接口级服务发现

| 维度 | 接口级（2.x） | 应用级（3.x） |
|------|-------------|-------------|
| **注册粒度** | 每个接口一个节点 | 每个应用一个节点 |
| **注册中心压力** | 与接口数成正比 | 恒定，一个应用一个实例 |
| **元数据存储** | URL 中携带所有参数 | 元数据中心（Nacos/本地） |
| **跨中心部署** | 复杂（参数变更需重注册） | 简单（元数据独立更新） |
| **兼容性** | Dubbo 2.x | Dubbo 3.x，兼容 2.x |

**应用级发现的核心流程**：

```
Provider 启动
  │
  ├─→ 注册应用实例 (ServiceInstance) → Nacos/ZK
  │      └─ 包含 IP:Port 和元数据地址
  │
  └─→ 发布元数据 (MetadataService)
         └─ 包含: 接口列表、参数定义、方法签名

Consumer 启动
  │
  ├─→ 订阅应用实例 (ServiceInstance) → 获得 IP:Port 列表
  │
  ├─→ 通过元数据服务获取接口定义
  │      └─ MetadataService.getServiceDefinition()
  │
  └─→ 将 IP:Port 转换为 Invoker 对象
```

# 7. 元数据服务（MetadataService）

```java
public interface MetadataService {
    // 获取指定修订版本之后的元数据
    String getMetadataInfo(String revision);
    
    // 获取全量服务定义
    String getServiceDefinition(String interfaceName, 
        String version, String group);
}
```

MetadataService 也是通过 Dubbo RPC 暴露的，Consumer 通过它获取 Provider 的接口列表和详细参数。

# 8. 注册中心高可用

```
Consumer                                                    Provider
   │                                                          │
   │  1. 启动时从注册中心拉取 Provider 列表                     │
   ├──────────────────────────────────────────────────────→  │
   │                                                          │
   │  2. 注册中心宕机                                          │
   │  ├── 本地文件缓存兜底（~/.dubbo/）                         │
   │  ├── 内存中已有 Provider 列表继续使用                       │
   │  └── 新 Provider 无法发现（降级）                          │
   │                                                          │
   │  3. 注册中心恢复                                          │
   │  └── 重连并重新订阅最新 Provider 列表                      │
```

## 8.1 缓存策略

| 缓存 | 位置 | 说明 |
|------|------|------|
| 内存缓存 | Properties | Provider URL 列表 |
| 磁盘缓存 | `~/.dubbo/` 文件 | 持久化缓存，注册中心宕机后兜底 |
| 远程缓存 | 注册中心 | 最新数据源 |

## 8.2 启动加载顺序

```
1. 加载本地磁盘缓存文件
2. 连接注册中心（异步，不阻塞启动）
3. 拉取注册中心最新数据覆盖本地缓存
```

# 9. 注册扩展实现

```java
@SPI("dubbo")
public interface RegistryFactory {
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
}
```

扩展实现示例（自定义注册中心）：

```java
public class CustomRegistryFactory implements RegistryFactory {
    @Override
    public Registry getRegistry(URL url) {
        return new CustomRegistry(url);
    }
}
```

SPI 文件（`META-INF/dubbo/org.apache.dubbo.registry.RegistryFactory`）：

```
custom=com.example.CustomRegistryFactory
```

使用：

```yaml
dubbo:
  registry:
    address: custom://127.0.0.1:9999
```
