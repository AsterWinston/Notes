
# 1. 健康检查概述

Nacos 需要准确判断每个服务实例的健康状态，确保服务消费者只调用健康的实例。根据实例类型不同，采用了两套完全不同的健康检查策略。

# 2. 临时实例 vs 持久实例健康检查对比

| 特性 | 临时实例（Ephemeral） | 持久实例（Persistent） |
|------|----------------------|----------------------|
| 检查方式 | 客户端主动上报心跳 | 服务端主动探测 |
| 剔除策略 | 心跳超时自动摘除 | 标记不健康，不摘除 |
| 协议支持 | HTTP / gRPC 心跳 | TCP / HTTP / MySQL 探测 |
| 恢复机制 | 重新上报心跳恢复 | 探测通过自动恢复 |
| 典型场景 | 微服务应用实例 | 数据库、中间件、遗留系统 |
| 一致性 | Distro（AP） | Raft（CP） |

# 3. 临时实例健康检查

## 3.1 客户端心跳流程

```
Client App                    Nacos Server
    │                               │
    │  (1) 服务注册                  │
    │──────────────────────────────>│ 记录 lastBeat = now
    │                               │ instance.healthy = true
    │                               │
    │  (2) 心跳 (每 5s)            │
    │  BeatInfo {                   │
    │    ip: 192.168.1.1            │ 更新 lastBeat = now
    │    port: 8080                 │
    │    serviceName: order-service │
    │  }                            │
    │──────────────────────────────>│
    │                               │
    │  (3) 服务端检查                │
    │     lastBeat > 15s?           │──────┐
    │                               │<─────┘
    │                               │
    │   若超时: instance.healthy = false
    │   若超过 30s: 从注册表中剔除
    │                               │
    │  (4) 推送实例变更 (剔除通知)  │
    │<──────────────────────────────│
```

## 3.2 服务端心跳接收

```java
// InstanceController.java (服务端)
@CanDistro
@PutMapping("/beat")
public ObjectNode beat(HttpServletRequest request) {
    ObjectNode result = JacksonUtils.createEmptyJsonNode();
    
    // 1. 解析心跳请求
    String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    String ip = WebUtils.required(request, "ip");
    int port = Integer.parseInt(WebUtils.required(request, "port"));
    String cluster = WebUtils.required(request, "cluster");
    String beatStr = WebUtils.required(request, "beat");
    BeatInfo beatInfo = JSON.parseObject(beatStr, BeatInfo.class);
    
    // 2. 处理心跳
    ClientBeatIntervalInterval interval = serviceManager
        .handleBeat(namespaceId, serviceName, ip, port, cluster, 
                    beatInfo, result);
    
    // 3. 返回心跳间隔（服务端可能调整）
    result.put("clientBeatInterval", interval.getValue());
    result.put("code", HttpServletResponse.SC_OK);
    
    return result;
}
```

## 3.3 ServiceManager 心跳处理

```java
// ServiceManager.java
public ClientBeatIntervalInterval handleBeat(String namespaceId, String serviceName,
    String ip, int port, String cluster, BeatInfo beatInfo, ObjectNode result) {
    
    // 1. 获取服务
    Service service = getService(namespaceId, serviceName);
    if (service == null) {
        // 服务不存在，尝试通过心跳恢复注册
        createEmptyService(namespaceId, serviceName, true);
        service = getService(namespaceId, serviceName);
    }
    
    // 2. 获取集群
    Cluster clusterObj = service.getClusterMap().get(cluster);
    
    // 3. 查找实例
    Instance instance = clusterObj.getEphemeralInstance(ip, port);
    
    if (instance == null) {
        // 实例不存在（可能服务端重启后丢失），通过心跳重建
        instance = new Instance();
        instance.setIp(ip);
        instance.setPort(port);
        instance.setClusterName(cluster);
        instance.setEphemeral(true);
        instance.setHealthy(true);
        instance.setEnabled(true);
        
        clusterObj.getEphemeralInstances().add(instance);
        
        // 发布新实例注册事件
        NotifyCenter.publishEvent(new InstanceChangeEvent(namespaceId, serviceName));
        
        result.put("code", HttpServletResponse.SC_CREATED); // 201 Created by heartbeat
    }
    
    // 4. 更新最后心跳时间
    instance.setLastBeat(System.currentTimeMillis());
    
    // 5. 如果实例之前被标记为不健康，恢复健康
    if (!instance.isHealthy()) {
        instance.setHealthy(true);
        result.put("code", HttpServletResponse.SC_RESET_CONTENT); // 205 Health restored
    }
    
    // 6. 更新心跳间隔
    long clientBeatInterval = instance.getInstanceHeartBeatInterval();
    if (beatInfo.getPeriod() != clientBeatInterval) {
        instance.setInstanceHeartBeatInterval(beatInfo.getPeriod());
    }
    
    return new ClientBeatIntervalInterval(clientBeatInterval);
}
```

## 3.4 过期实例自动剔除

```java
// ServiceManager.java
// 定时任务：检查并剔除心跳超时的临时实例
@Component
public class ExpiredInstanceCleaner implements Runnable {
    
    @Override
    public void run() {
        for (Map.Entry<String, Map<String, Service>> namespaceEntry : serviceMap.entrySet()) {
            for (Map.Entry<String, Service> serviceEntry : namespaceEntry.getValue().entrySet()) {
                Service service = serviceEntry.getValue();
                if (!service.isEphemeral()) continue; // 只处理临时实例服务
                
                for (Cluster cluster : service.getClusterMap().values()) {
                    List<Instance> expiredList = new ArrayList<>();
                    
                    for (Instance instance : cluster.getEphemeralInstances()) {
                        long now = System.currentTimeMillis();
                        long timeout = instance.getInstanceHeartBeatInterval() * 3;
                        
                        // 心跳超时判定
                        if (now - instance.getLastBeat() > timeout) {
                            expiredList.add(instance);
                        }
                    }
                    
                    // 移除过期实例
                    if (!expiredList.isEmpty()) {
                        cluster.getEphemeralInstances().removeAll(expiredList);
                        service.setLastModifiedMillis(System.currentTimeMillis());
                        
                        // 推送变更通知
                        pushService.serviceChanged(service);
                    }
                }
            }
        }
    }
}

// 配置定时执行（默认每 5s 执行一次）
// GlobalExecutor.schedule(new ExpiredInstanceCleaner(), 5000, 5000, MILLISECONDS);
```

# 4. 持久实例健康检查

## 4.1 健康检查模式

持久实例支持多种健康检查模式：

```java
// 健康检查类型枚举
public enum HealthCheckMode {
    NONE,    // 不检查
    TCP,     // TCP 端口探测
    HTTP,    // HTTP 健康检查接口
    MYSQL    // MySQL 连接检查
}
```

## 4.2 服务端探测流程

```java
// HealthCheckProcessor.java (抽象处理器)
public abstract class HealthCheckProcessor {
    
    // 健康检查线程池
    protected final ScheduledExecutorService executor;
    
    // 核心健康检查逻辑
    public abstract void process(HealthCheckTask task);
    
    // 默认的检查任务调度
    public void schedule(HealthCheckTask task) {
        executor.scheduleWithFixedDelay(() -> {
            try {
                process(task);
            } catch (Exception e) {
                log.error("Health check failed for task: {}", task, e);
            }
        }, task.getCheckRtInterval(), task.getCheckRtInterval(), TimeUnit.MILLISECONDS);
    }
}
```

## 4.3 TCP 健康检查实现

```java
// TcpSuperSenseProcessor.java
@Component
public class TcpSuperSenseProcessor extends HealthCheckProcessor {
    
    // Socket 连接超时
    private static final int TCP_SOCKET_TIMEOUT = 3000; // 3s
    
    // 失败容忍次数（连续失败多少次才标记不健康）
    private static final int DEFAULT_FAIL_TOLERANCE = 3;
    
    @Override
    public void process(HealthCheckTask task) {
        Instance instance = task.getInstance();
        Cluster cluster = task.getCluster();
        
        try (Socket socket = new Socket()) {
            // TCP 连接探测
            socket.connect(
                new InetSocketAddress(instance.getIp(), instance.getPort()), 
                TCP_SOCKET_TIMEOUT);
            
            // 连接成功 → 标记健康
            if (!instance.isHealthy()) {
                instance.setHealthy(true);
                // 发布健康恢复事件
                NotifyCenter.publishEvent(new InstanceHealthChangedEvent(instance, true));
            }
            
            // 重置失败次数
            task.setFailCount(0);
            
        } catch (Exception e) {
            // 连接失败
            task.incrementFailCount();
            
            int tolerance = getFailTolerance(cluster);
            if (task.getFailCount() >= tolerance) {
                // 连续失败达到阈值 → 标记不健康
                instance.setHealthy(false);
                // 发布健康恶化事件
                NotifyCenter.publishEvent(new InstanceHealthChangedEvent(instance, false));
            }
        }
    }
    
    private int getFailTolerance(Cluster cluster) {
        HealthChecker checker = cluster.getHealthChecker();
        // 可通过健康检查器配置
        return checker != null ? checker.getFailCountThreshold() : DEFAULT_FAIL_TOLERANCE;
    }
}
```

## 4.4 HTTP 健康检查实现

```java
// HttpHealthCheckProcessor.java
public class HttpHealthCheckProcessor extends HealthCheckProcessor {
    
    @Override
    public void process(HealthCheckTask task) {
        Instance instance = task.getInstance();
        
        // 获取健康检查路径（从实例元数据中获取）
        String healthCheckPath = instance.getMetadata()
            .getOrDefault("healthCheckPath", "/actuator/health");
        String healthCheckUrl = "http://" + instance.getIp() + ":" 
            + instance.getPort() + healthCheckPath;
        
        try {
            // HTTP GET 请求
            HttpURLConnection connection = 
                (HttpURLConnection) new URL(healthCheckUrl).openConnection();
            connection.setConnectTimeout(3000);
            connection.setReadTimeout(3000);
            
            int responseCode = connection.getResponseCode();
            
            if (responseCode == 200) {
                instance.setHealthy(true);
                task.setFailCount(0);
            } else {
                handleFailure(task, instance);
            }
        } catch (Exception e) {
            handleFailure(task, instance);
        }
    }
    
    private void handleFailure(HealthCheckTask task, Instance instance) {
        task.incrementFailCount();
        if (task.getFailCount() >= 3) {
            instance.setHealthy(false);
        }
    }
}
```

# 5. 保护阈值机制

## 5.1 保护阈值原理

保护阈值（Protect Threshold）是一个介于 0 和 1 之间的浮点数。当健康实例占比低于该阈值时，触发保护机制：

```java
// Service.java
public class Service {
    
    private float protectThreshold = 0.0F; // 保护阈值（默认 0 即不启用）
    
    public boolean reachProtectionThreshold() {
        int totalCount = allIPs().size();
        if (totalCount == 0) return false;
        
        int healthyCount = healthyIPs().size();
        float healthyRatio = (float) healthyCount / totalCount;
        
        return healthyRatio < protectThreshold;
    }
    
    public List<Instance> allIPs(boolean healthyOnly) {
        if (healthyOnly) {
            // 如果触发保护阈值且 healthyOnly=true
            // → 返回所有实例（包括不健康的），防止大规模雪崩
            if (reachProtectionThreshold()) {
                return allIPs(); // 忽略健康状态，全部返回
            }
            return healthyIPs();
        }
        return allIPs();
    }
    
    public List<Instance> allIPs() {
        List<Instance> allInstances = new ArrayList<>();
        for (Cluster cluster : clusterMap.values()) {
            allInstances.addAll(cluster.allIPs());
        }
        return allInstances;
    }
}
```

## 5.2 保护阈值应用场景

假设服务有 10 个实例，保护阈值设为 0.6：

```
正常时:
  健康实例: 9  →  比例 = 0.9 → 只返回健康实例

波动时:
  健康实例: 5  →  比例 = 0.5 < 0.6 → 触发保护 → 返回全部 10 个实例
  （即使有 5 个实例心跳超时，仍返回全部实例，容错网络波动）
```

# 6. gRPC 健康检查（Nacos 2.x）

## 6.1 连接级别健康检查

Nacos 2.x 中，客户端与服务器之间建立了 gRPC 长连接。服务端通过检查 gRPC 连接状态来判断客户端是否存活：

```java
// ConnectionManager.java
@Component
public class ConnectionManager {
    
    // 活跃连接 Map: connectionId -> Connection
    private final Map<String, Connection> connections = new ConcurrentHashMap<>();
    
    // gRPC 连接建立的 callback
    @Override
    public boolean connectAck(ConnectAckRequest request) {
        String connectionId = request.getConnectionId();
        
        Connection connection = new Connection();
        connection.setConnectionId(connectionId);
        connection.setLabels(request.getLabels());
        connection.setLastActiveTime(System.currentTimeMillis());
        
        connections.put(connectionId, connection);
        
        // 注册该连接关联的服务实例
        registerClientInfo(connection, request);
        
        return true;
    }
    
    // 定期检测连接活跃性
    @Scheduled(fixedRate = 3000)
    public void checkConnectionHealth() {
        long now = System.currentTimeMillis();
        
        for (Connection connection : connections.values()) {
            // 超过 20s 没有活跃 → 判定为失活
            if (now - connection.getLastActiveTime() > 20000) {
                // 连接失效 → 摘除该客户端注册的所有服务实例
                unregisterClientServices(connection);
                connections.remove(connection.getConnectionId());
            }
        }
    }
}
```

## 6.2 ServerCheckRequest 保持活跃

```java
// GrpcClient.java (客户端侧)
// 定期响应服务端的 ServerCheckRequest
rpcClient.registerServerRequestHandler((request, meta) -> {
    if (request instanceof ServerCheckRequest) {
        ServerCheckResponse response = new ServerCheckResponse();
        response.setConnectionId(connectionId);
        return response;
    }
    return null;
});
```

# 7. 健康检查配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `nacos.naming.healthCheck.enabled` | true | 是否启用健康检查 |
| `nacos.naming.healthCheck.interval` | 5000ms | 健康检查间隔（临时实例心跳） |
| `nacos.naming.healthCheck.timeout` | 15000ms | 心跳超时时间（超时 3 次剔除） |
| `nacos.naming.healthCheck.failCount` | 3 | 持久实例失败容忍次数 |
| `nacos.naming.protect.threshold` | 0.0 | 保护阈值（0 为不启用） |

# 8. 自定义健康检查

```java
// 实现自定义健康检查处理器
@Component
public class CustomHealthCheckProcessor extends HealthCheckProcessor {
    
    @Override
    public void process(HealthCheckTask task) {
        Instance instance = task.getInstance();
        
        // 自定义检查逻辑（如：检查 Kafka 消费延迟）
        try {
            boolean isHealthy = performCustomCheck(instance);
            instance.setHealthy(isHealthy);
        } catch (Exception e) {
            instance.setHealthy(false);
        }
    }
    
    private boolean performCustomCheck(Instance instance) {
        // 自定义健康检查实现
        return true;
    }
}
```

## 8.1 实例级别自定义健康检查路径

```java
Instance instance = new Instance();
instance.setIp("192.168.1.1");
instance.setPort(8080);
instance.getMetadata().put("healthCheckPath", "/health/custom");
// 支持自定义健康检查协议
instance.getMetadata().put("healthCheckProtocol", "redis-ping");
```
