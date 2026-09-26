
# 1. 版本演进路线

```
Nacos 1.x (2018-2021)                Nacos 2.x (2021-至今)
┌────────────────────────┐          ┌──────────────────────────┐
│ HTTP REST + UDP         │   →      │ gRPC Bi-directional      │
│ 长轮询配置通知           │   →      │ gRPC Server Push         │
│ Derby/MySQL 存储         │   →      │ 插件化存储              │
│ 单一鉴权模式             │   →      │ 插件化鉴权 (SPI)        │
│ 临时实例仅 AP 模式       │   →      │ 不变                    │
│ 服务端间 HTTP 同步       │   →      │ gRPC Stream 同步        │
│ 无事件中心               │   →      │ 事件机制 (NotifyCenter) │
│ 批量注册支持弱           │   →      │ 批量注册 API            │
└────────────────────────┘          └──────────────────────────┘
```

# 2. gRPC 通信层升级（最核心变化）

## 2.1 为什么升级到 gRPC

Nacos 1.x 的 HTTP + UDP 通信架构存在以下问题：

| 问题 | 1.x 表现 | 2.x 解决方案 |
|------|---------|-------------|
| 推送延迟高 | UDP 不可靠，存在丢包 | gRPC 双向流，可靠推送 |
| 连接开销大 | 每次 HTTP 短连接 | gRPC 长连接复用 |
| 配置通知效率低 | HTTP 长轮询，30s 超时 | gRPC Server Push，实时通知 |
| 服务端间同步效率低 | HTTP 请求-响应模式 | gRPC Stream，批量处理 |

## 2.2 gRPC 连接模型

```
                    Nacos 2.x gRPC 连接架构

┌─────────────────────────────────────────────────────────────┐
│                      Nacos Client                           │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Config Client │  │Naming Client │  │ gRPC Stream  │      │
│  │ SDK          │  │ SDK          │  │ Connection   │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         └─────────────────┼─────────────────┘               │
│                           │                                  │
│                   ┌───────▼────────┐                        │
│                   │  RpcClient     │                        │
│                   │  (Single      │                        │
│                   │   Connection) │                        │
│                   └───────┬────────┘                        │
└───────────────────────────┼──────────────────────────────────┘
                            │
                    gRPC Bi-directional Stream
                    (双向流，单 TCP 长连接)
                            │
┌───────────────────────────┼──────────────────────────────────┐
│                      Nacos Server                            │
│                   ┌───────▼────────┐                        │
│                   │ Connection     │                        │
│                   │ Manager        │                        │
│                   └───────┬────────┘                        │
│         ┌─────────────────┼─────────────────┐               │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐      │
│  │Config Push  │  │Naming Push  │  │Server Check  │      │
│  │Service      │  │Service      │  │Service       │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

## 2.3 gRPC 请求类型

```java
// Nacos 2.x gRPC 支持的请求类型

// 服务注册/发现
public enum InstanceRequestType {
    INSTANCE_REGISTER,    // 注册实例
    INSTANCE_DEREGISTER,  // 注销实例
    INSTANCE_QUERY,       // 查询实例
    INSTANCE_BEAT,        // 心跳
    INSTANCE_SUBSCRIBE,   // 订阅
    INSTANCE_UNSUBSCRIBE  // 取消订阅
}

// 配置管理
public enum ConfigRequestType {
    CONFIG_QUERY,         // 查询配置
    CONFIG_PUBLISH,       // 发布配置
    CONFIG_REMOVE,        // 删除配置
    CONFIG_LISTEN,        // 监听配置变更
    CONFIG_NOTIFY         // 主动推送变更
}

// 系统检查
public enum ServerRequestType {
    SERVER_CHECK,         // 服务端探测客户端
    CLIENT_DETECTION,     // 客户端回应探测
    CONNECTION_SETUP,     // 连接建立
    CONNECTION_RESET      // 连接重建
}
```

## 2.4 gRPC vs HTTP 性能对比

| 指标 | Nacos 1.x (HTTP) | Nacos 2.x (gRPC) | 提升 |
|------|-----------------|-------------------|------|
| 配置推送延迟 | 3s ~ 30s | < 100ms | **30倍+** |
| 服务变更推送 | 1s ~ 3s (UDP) | < 100ms | **10倍+** |
| 单连接内存消耗 | 1MB+ | 0.5MB | 50% |
| 每秒注册 TPS | ~2000 | ~5000 | 2.5倍 |
| 每秒查询 QPS | ~5000 | ~15000 | 3倍 |
| 客户端连接数（万级） | 不稳定 | 稳定支持 | 显著提升 |

# 3. 插件化存储

## 3.1 存储插件架构

Nacos 2.x 将存储层设计为插件化，通过 SPI 机制支持自定义存储：

```java
// 存储 SPI 接口
public interface DataSourcePluginService {
    String getDataSourceType();
    DataSource getDataSource();
}

// MySQL 插件（内置实现）
public class MysqlDataSourcePluginImpl implements DataSourcePluginService {
    @Override
    public String getDataSourceType() {
        return "mysql";
    }
    
    @Override
    public DataSource getDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(EnvUtil.getProperty("db.url.0"));
        ds.setUsername(EnvUtil.getProperty("db.user.0"));
        ds.setPassword(EnvUtil.getProperty("db.password.0"));
        return ds;
    }
}

// Derby 插件（内置实现，用于单机模式）
public class DerbyDataSourcePluginImpl implements DataSourcePluginService {
    // ...
}

// 自定义存储插件示例（如接入 PostgreSQL）
public class PostgreDataSourcePluginImpl implements DataSourcePluginService {
    @Override
    public String getDataSourceType() {
        return "postgresql";
    }
    
    @Override
    public DataSource getDataSource() {
        // PostgreSQL 数据源配置
    }
}
```

## 3.2 SPI 配置

```
resources/
└── META-INF/
    └── services/
        └── com.alibaba.nacos.plugin.datasource.spi.DataSourcePluginService
            内容: com.example.postgre.PostgreDataSourcePluginImpl
```

# 4. 批量注册

## 4.1 批量注册 API

```bash
# 批量注册多个实例
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance/batch' \
  -H 'Content-Type: application/json' \
  -d '{
    "serviceName": "order-service",
    "groupName": "DEFAULT_GROUP",
    "instances": [
      {"ip": "192.168.1.1", "port": 8080, "weight": 10},
      {"ip": "192.168.1.2", "port": 8080, "weight": 10},
      {"ip": "192.168.1.3", "port": 8080, "weight": 10}
    ],
    "metadata": {"version": "v2.0.0"}
  }'
```

## 4.2 批量注册源码

```java
// InstanceController.java
@PostMapping("/batch")
public String batchRegister(@RequestBody String requestBody) {
    BatchInstanceRequest request = JSON.parseObject(requestBody, 
        BatchInstanceRequest.class);
    
    List<Instance> instances = request.getInstances();
    String serviceName = request.getServiceName();
    String groupName = request.getGroupName();
    
    // 批量注册
    for (Instance instance : instances) {
        instance.setServiceName(serviceName);
        instance.setEphemeral(true);
        serviceManager.registerInstance(namespaceId, serviceName, instance);
    }
    
    // 只触发一次推送事件（批量优化）
    Service service = serviceManager.getService(namespaceId, serviceName);
    pushService.serviceChanged(service);
    
    return "ok";
}
```

# 5. 事件机制增强

## 5.1 NotifyCenter 事件中心

```java
// Nacos 2.x 引入统一的事件通知中心
public class NotifyCenter {
    
    // 事件发布
    public static boolean publishEvent(Event event) {
        return publisher.publish(event);
    }
    
    // 注册事件订阅者
    public static void registerSubscriber(Subscriber consumer) {
        publisher.addSubscriber(consumer);
    }
}

// 配置变更事件
NotifyCenter.publishEvent(new LocalDataChangeEvent(groupKey));

// 服务变更事件
NotifyCenter.publishEvent(new ServiceChangeEvent(service));

// 实例变更事件
NotifyCenter.publishEvent(new InstanceChangeEvent(namespaceId, serviceName));
```

## 5.2 事件驱动的服务推送

```java
// PushService 监听 ServiceChangeEvent
@Component
public class PushService extends Subscriber<ServiceChangeEvent> {
    
    @Override
    public void onEvent(ServiceChangeEvent event) {
        Service service = event.getService();
        
        // 获取订阅了该服务的所有客户端
        Map<String, PushClient> clients = clientMap.get(service.getName());
        
        // 通过 gRPC 批量推送变更
        for (PushClient client : clients.values()) {
            NotifySubscriberRequest request = buildNotifyRequest(service, client);
            connectionManager.getConnection(client.getConnectionId())
                .request(request);
        }
    }
    
    @Override
    public Class<? extends Event> subscribeType() {
        return ServiceChangeEvent.class;
    }
}
```

# 6. 灰度升级 Nacos 1.x → 2.x

## 6.1 端口变化

Nacos 2.x 新增 gRPC 端口（基于主端口偏移）：

```
主端口: 8848 (HTTP)
  + 1000 = 9848: gRPC 客户端请求服务端端口
  - 1000 = 7848: gRPC 客户端连接端口 (内置 RpcServer)
  + 1001 = 9849: gRPC 集群通信端口
```

## 6.2 升级检查清单

| 检查项 | 说明 | 操作 |
|--------|------|------|
| 防火墙 | 新端口（7848/9848/9849）是否开放 | 开放新端口 |
| 客户端 SDK | 是否升级到 nacos-client 2.x | 升级依赖 |
| Spring Cloud 版本 | 是否对应的 Alibaba 版本 | 参考兼容性矩阵 |
| JVM 版本 | 推荐 JDK 8 / 11 | 确认版本 |
| 配置兼容性 | application.properties 是否有废弃配置 | 检查配置 |

## 6.3 客户端升级

```xml
<!-- 升级客户端 SDk -->
<dependency>
    <groupId>com.alibaba.nacos</groupId>
    <artifactId>nacos-client</artifactId>
    <version>2.2.3</version>
</dependency>
```

```yaml
# 客户端配置新增（服务端地址需包含 gRPC 端口）
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  # 主端口即可，SDK 自动偏移
```

# 7. Nacos 生态扩展

## 7.1 多语言 SDK

| 语言 | SDK | 版本 | 仓库 |
|------|-----|------|------|
| Java | nacos-client | 2.2.3 | github.com/alibaba/nacos |
| Go | nacos-sdk-go | 2.2.0 | github.com/nacos-group/nacos-sdk-go |
| Python | nacos-sdk-python | 1.0.1 | github.com/nacos-group/nacos-sdk-python |
| C++ | nacos-sdk-cpp | 1.1.0 | github.com/nacos-group/nacos-sdk-cpp |
| C# | nacos-sdk-csharp | 1.3.5 | github.com/nacos-group/nacos-sdk-csharp |
| Node.js | nacos | 2.7.3 | npmjs.com/package/nacos |
| Rust | nacos-sdk-rust | 0.3.0 | github.com/nacos-group/nacos-sdk-rust |

## 7.2 Istio 集成

Nacos 提供了 Istio MCP Server，允许将 Nacos 的服务注册数据同步到 Istio 控制平面：

```yaml
# Nacos → Istio 服务同步
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: nacos-external-service
spec:
  hosts:
    - "order-service.nacos"
  location: MESH_EXTERNAL
  ports:
    - number: 8080
      name: http
      protocol: HTTP
  resolution: DNS
```

# 8. Nacos 3.x 展望

社区规划中的 Nacos 3.x 方向：

1. **控制面与数据面分离**：向 Service Mesh 架构演进
2. **xDS 协议支持**：原生支持 Envoy 的 xDS 标准协议
3. **多租户增强**：更细粒度的租户隔离和配额管理
4. **存储引擎独立**：支持更多的后端存储（如 TiDB、Elasticsearch）
5. **AI 驱动的容量规划**：基于历史数据预测服务实例数变化
6. **WASM 插件生态**：支持 WebAssembly 插件扩展
