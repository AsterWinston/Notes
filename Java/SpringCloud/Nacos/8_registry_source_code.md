
# 1. 客户端模块结构

Nacos 客户端（nacos-client）负责与 Nacos 服务端通信，核心职责包括：

```
nacos-client/
├── com.alibaba.nacos.client
│   ├── naming/                        # 命名服务（注册中心）
│   │   ├── NacosNamingService         # 命名服务入口
│   │   ├── NamingProxy                # 通信代理（HTTP/gRPC）
│   │   ├── HostReactor                # 服务实例缓存与订阅
│   │   ├── BeatReactor                # 心跳管理器
│   │   ├── EventDispatcher            # 事件分发器
│   │   ├── cache/
│   │   │   └── DiskCache              # 磁盘缓存
│   │   └── remote/
│   │       └── NamingClientProxy      # 2.x gRPC 代理
│   ├── config/                        # 配置中心（见第 7 章）
│   └── common/
│       ├── ServerListManager          # 服务端地址列表管理
│       └── SecurityProxy              # 安全代理
```

# 2. NacosNamingService 初始化

## 2.1 构造函数

```java
// NacosNamingService.java
public class NacosNamingService implements NamingService {
    
    // 服务端代理（负责 HTTP/gRPC 通信）
    private NamingClientProxy clientProxy;
    
    // 心跳反应器
    private BeatReactor beatReactor;
    
    // 主机反应器（缓存 + 订阅 + 定时拉取）
    private HostReactor hostReactor;
    
    // 事件分发器
    private EventDispatcher eventDispatcher;
    
    // 命名空间
    private String namespace;
    
    public NacosNamingService(Properties properties) throws NacosException {
        // 1. 初始化命名空间
        this.namespace = InitUtils.initNamespaceForNaming(properties);
        
        // 2. 创建服务端代理
        this.clientProxy = new NamingClientProxyDelegate(this.namespace, ...);
        
        // 3. 初始化各个组件
        this.beatReactor = new BeatReactor(this.clientProxy, ...);
        this.hostReactor = new HostReactor(this.clientProxy, ...);
        this.eventDispatcher = new EventDispatcher();
    }
}
```

## 2.2 NamingClientProxyDelegate（通信代理代理）

Nacos 2.x 引入了 `NamingClientProxyDelegate` 作为代理选择层，根据情况选择使用 gRPC 或 HTTP：

```java
// NamingClientProxyDelegate.java
public class NamingClientProxyDelegate implements NamingClientProxy {
    
    private NamingGrpcClientProxy grpcClientProxy;  // gRPC 代理（优先）
    private NamingHttpClientProxy httpClientProxy;  // HTTP 代理（兜底）
    
    @Override
    public void registerService(String serviceName, String groupName, Instance instance) {
        // 优先使用 gRPC，不支持时回退到 HTTP
        if (grpcClientProxy.isEnable()) {
            grpcClientProxy.registerService(serviceName, groupName, instance);
        } else {
            httpClientProxy.registerService(serviceName, groupName, instance);
        }
    }
    
    @Override
    public ServiceInfo queryInstancesOfService(String serviceName, String groupName, 
                                                String clusters, boolean healthyOnly) {
        // 查询实例优先使用 gRPC
        if (grpcClientProxy.isEnable()) {
            return grpcClientProxy.queryInstancesOfService(serviceName, groupName, 
                                                          clusters, healthyOnly);
        }
        return httpClientProxy.queryInstancesOfService(serviceName, groupName, 
                                                       clusters, healthyOnly);
    }
    
    // 服务订阅必须使用 gRPC（Nacos 2.x 核心要求）
    @Override
    public ServiceListResult subscribe(String serviceName, String groupName, String clusters) {
        if (!grpcClientProxy.isEnable()) {
            throw new NacosException("Subscribe is only supported by gRPC in Nacos 2.x");
        }
        return grpcClientProxy.subscribe(serviceName, groupName, clusters);
    }
}
```

# 3. 心跳机制（BeatReactor）

## 3.1 心跳发送流程

```java
// BeatReactor.java
public class BeatReactor implements Closeable {
    
    // 心跳信息 Map: serviceName -> BeatInfo
    private final Map<String, BeatInfo> dom2Beat = new ConcurrentHashMap<>();
    
    // 心跳线程池
    private final ScheduledExecutorService executorService;
    private final NamingProxy serverProxy;
    
    // 心跳间隔（默认 5s）
    public static final long DEFAULT_BEAT_INTERVAL = 5000L;
    
    // 添加心跳信息（注册时调用）
    public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
        String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
        
        // 1. 存储心跳信息
        dom2Beat.put(key, beatInfo);
        
        // 2. 启动心跳定时任务
        BeatTask beatTask = new BeatTask(key, beatInfo);
        executorService.scheduleAtFixedRate(
            beatTask, 
            beatInfo.getPeriod(), // 首次延迟
            beatInfo.getPeriod(), // 定时间隔
            TimeUnit.MILLISECONDS
        );
    }
    
    // 心跳任务
    class BeatTask implements Runnable {
        String serviceName;
        BeatInfo beatInfo;
        
        @Override
        public void run() {
            if (beatInfo.isStopped()) {
                return;
            }
            // 发送心跳请求
            JsonNode result = serverProxy.sendBeat(beatInfo);
            long nextInterval = beatInfo.getPeriod();
            
            if (result != null) {
                // 服务端可能返回新的心跳间隔
                nextInterval = result.get("clientBeatInterval").asLong();
                // 检查是否被标记为不健康
                boolean isLightBeatEnabled = false;
                if (result.has("lightBeatEnabled")) {
                    isLightBeatEnabled = result.get("lightBeatEnabled").asBoolean();
                }
            }
            
            // 更新心跳间隔
            if (nextInterval != beatInfo.getPeriod()) {
                // 重新调度心跳任务
                beatInfo.setPeriod(nextInterval);
                reschedule(serviceName, beatInfo, nextInterval);
            }
        }
    }
}
```

## 3.2 心跳请求格式

```java
// NamingHttpClientProxy.java
public JsonNode sendBeat(BeatInfo beatInfo) {
    Map<String, String> params = new HashMap<>(8);
    params.put("beat", JSON.toJSONString(beatInfo));
    params.put("serviceName", beatInfo.getServiceName());
    params.put("ip", beatInfo.getIp());
    params.put("port", String.valueOf(beatInfo.getPort()));
    params.put("cluster", beatInfo.getCluster());
    params.put("weight", String.valueOf(beatInfo.getWeight()));
    
    // PUT /nacos/v1/ns/instance/beat
    return reqApi(UtilAndComs.nacosUrlBase + "/instance/beat", params, HttpMethod.PUT);
}
```

## 3.3 轻量心跳（Nacos 2.x）

```java
// NamingGrpcClientProxy.java - 轻量心跳
public void sendBeat(BeatInfo beatInfo) {
    // 构建 gRPC 心跳请求
    InstanceRequest request = new InstanceRequest();
    request.setType(InstanceRequestType.INSTANCE_BEAT);  // 心跳类型
    request.setServiceName(beatInfo.getServiceName());
    request.setGroupName(beatInfo.getGroupName());
    request.setIp(beatInfo.getIp());
    request.setPort(beatInfo.getPort());
    request.setCluster(beatInfo.getCluster());
    
    // gRPC 轻量心跳不携带完整实例信息，仅携带标识
    rpcClient.request(request);
}
```

# 4. HostReactor 核心实现

## 4.1 服务信息缓存与订阅

```java
// HostReactor.java
public class HostReactor implements Closeable {
    
    // 服务信息缓存: key(serviceName@@group) → ServiceInfo
    private final Map<String, ServiceInfo> serviceInfoMap;
    
    // 订阅关系: serviceName → Subscriber
    private final Map<String, Set<Subscriber>> subscriberMap;
    
    // 定时拉取线程池
    private final ScheduledExecutorService executor;
    
    // 定时更新任务延迟
    private final long DEFAULT_DELAY = 1000L;       // 首次 1s
    private final long DEFAULT_UPDATE_INTERVAL = 3000L; // 周期 3s
    
    // 主动拉取最新服务列表（非订阅模式）
    public ServiceInfo getServiceInfo(String serviceName, String clusters) {
        String key = ServiceInfo.getKey(serviceName, clusters);
        
        // 1. 先查内存缓存
        ServiceInfo serviceInfo = serviceInfoMap.get(key);
        
        if (serviceInfo == null) {
            // 2. 查磁盘缓存
            serviceInfo = DiskCache.read(serviceName, clusters);
            if (serviceInfo != null) {
                serviceInfoMap.put(key, serviceInfo);
            }
        }
        
        if (serviceInfo == null) {
            // 3. 从服务端拉取（同步）
            serviceInfo = clientProxy.queryList(serviceName, clusters, 0, false);
            serviceInfoMap.put(key, serviceInfo);
        }
        
        // 4. 启动异步更新任务
        scheduleUpdateIfAbsent(serviceName, clusters);
        
        return serviceInfo;
    }
    
    // 服务订阅（核心）
    public ServiceInfo subscribe(String serviceName, String clusters) {
        // 1. 立即拉取一次
        ServiceInfo serviceInfo = clientProxy.queryList(
            serviceName, clusters, pushReceiver.getUdpPort(), false);
        
        // 2. 更新内存缓存
        serviceInfoMap.put(ServiceInfo.getKey(serviceName, clusters), serviceInfo);
        
        // 3. 启动定时更新任务（异步）
        scheduleUpdateIfAbsent(serviceName, clusters);
        
        return serviceInfo;
    }
}
```

## 4.2 定时更新任务

```java
// HostReactor.java - UpdateTask
public class UpdateTask implements Runnable {
    
    final String serviceName;
    final String clusters;
    long lastRefTime = Long.MAX_VALUE;
    
    @Override
    public void run() {
        try {
            String key = ServiceInfo.getKey(serviceName, clusters);
            ServiceInfo cachedService = serviceInfoMap.get(key);
            
            // 1. 检查上次更新时间
            if (cachedService.getLastRefTime() > lastRefTime) {
                // 已经有更新的数据
                return;
            }
            
            // 2. 从服务端查询（增量或全量）
            ServiceInfo serviceInfo = clientProxy.queryList(
                serviceName, clusters, pushReceiver.getUdpPort(), false);
            
            if (serviceInfo != null && serviceInfo.getHosts() != null) {
                // 3. 比较 checksum，决定是否需要更新
                if (!cachedService.getChecksum().equals(serviceInfo.getChecksum())) {
                    // 数据有变化，更新缓存
                    serviceInfoMap.put(key, serviceInfo);
                    
                    // 4. 写入磁盘缓存
                    DiskCache.write(serviceName, clusters, serviceInfo);
                    
                    // 5. 通知订阅者
                    notifySubscribers(serviceName, serviceInfo);
                }
            }
        } catch (Exception e) {
            log.error("Failed to update service info: {}", serviceName, e);
        }
        // 重新调度下一次更新
        executor.schedule(this, DEFAULT_UPDATE_INTERVAL, TimeUnit.MILLISECONDS);
    }
}
```

# 5. 事件通知机制

## 5.1 事件分发器

```java
// EventDispatcher.java
public class EventDispatcher {
    
    // 事件监听器 Map: serviceName -> List<EventListener>
    private final Map<String, List<EventListener>> listenerMap;
    
    // 注册监听器
    public void addListener(String serviceName, String clusters, EventListener listener) {
        String key = ServiceInfo.getKey(serviceName, clusters);
        listenerMap.computeIfAbsent(key, k -> new CopyOnWriteArrayList<>()).add(listener);
    }
    
    // 分发服务变更事件
    public void serviceChanged(String serviceName, List<Instance> instances) {
        String key = ServiceInfo.getKey(serviceName, "");
        List<EventListener> listeners = listenerMap.get(key);
        
        if (listeners != null) {
            NamingEvent event = new NamingEvent(serviceName, instances);
            for (EventListener listener : listeners) {
                // 在线程池中执行监听器回调
                executor.execute(() -> {
                    try {
                        listener.onEvent(event);
                    } catch (Exception e) {
                        log.error("Error notifying listener for service: {}", serviceName, e);
                    }
                });
            }
        }
    }
}
```

## 5.2 UDP 推送接收器（Nacos 1.x）

```java
// PushReceiver.java
public class PushReceiver implements Runnable, Closeable {
    
    private DatagramSocket udpSocket;
    private volatile boolean closed = false;
    
    public PushReceiver(HostReactor hostReactor) {
        try {
            // 随机端口绑定 UDP Socket
            this.udpSocket = new DatagramSocket();
            this.hostReactor = hostReactor;
        } catch (SocketException e) {
            throw new RuntimeException(e);
        }
    }
    
    @Override
    public void run() {
        while (!closed) {
            try {
                byte[] buffer = new byte[UDP_MSS];
                DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
                
                // 阻塞等待 UDP 推送
                udpSocket.receive(packet);
                
                // 解析推送内容（JSON 格式）
                String json = new String(packet.getData(), 0, packet.getLength(), 
                                         StandardCharsets.UTF_8);
                PushPacket pushPacket = JSON.parseObject(json, PushPacket.class);
                
                // 触发服务更新
                String ack = "{\"type\":\"push-ack\",\"lastRefTime\":" + 
                             pushPacket.getLastRefTime() + "}";
                
                // 发送 ACK 确认
                DatagramPacket ackPacket = new DatagramPacket(
                    ack.getBytes(StandardCharsets.UTF_8),
                    ack.length(), packet.getSocketAddress());
                udpSocket.send(ackPacket);
                
                // 异步更新服务列表
                hostReactor.processServiceJson(pushPacket);
                
            } catch (Exception e) {
                log.error("UDP push receiver error", e);
            }
        }
    }
}
```

# 6. 磁盘缓存机制

## 6.1 本地缓存文件

```java
// DiskCache.java
public class DiskCache {
    
    // 缓存目录: ${user.home}/nacos/naming/
    public static final String CACHE_DIR = 
        System.getProperty("user.home") + File.separator + "nacos" + 
        File.separator + "naming" + File.separator;
    
    // 写入缓存
    public static void write(String serviceName, String clusters, ServiceInfo serviceInfo) {
        String key = ServiceInfo.getKey(serviceName, clusters);
        File cacheFile = new File(CACHE_DIR, key);
        
        try {
            // 创建目录
            if (!cacheFile.getParentFile().exists()) {
                cacheFile.getParentFile().mkdirs();
            }
            
            // 将 ServiceInfo 序列化为 JSON 写入文件
            String content = JSON.toJSONString(serviceInfo);
            FileUtils.writeStringToFile(cacheFile, content, StandardCharsets.UTF_8);
            
        } catch (IOException e) {
            log.error("Failed to write disk cache for: {}", key, e);
        }
    }
    
    // 读取缓存
    public static ServiceInfo read(String serviceName, String clusters) {
        String key = ServiceInfo.getKey(serviceName, clusters);
        File cacheFile = new File(CACHE_DIR, key);
        
        if (!cacheFile.exists()) {
            return null;
        }
        
        try {
            String content = FileUtils.readFileToString(cacheFile, StandardCharsets.UTF_8);
            return JSON.parseObject(content, ServiceInfo.class);
        } catch (IOException e) {
            log.error("Failed to read disk cache for: {}", key, e);
            return null;
        }
    }
}
```

# 7. 服务端地址管理

## 7.1 ServerListManager（多服务端容错）

```java
// ServerListManager.java
public class ServerListManager {
    
    // 服务端地址列表（从启动参数解析）
    private List<String> serverList = new ArrayList<>();
    
    // 当前服务端索引
    private volatile int currentIndex = 0;
    
    // 获取下一个服务端地址（简单轮询）
    public String getNextServer() {
        if (serverList.isEmpty()) {
            throw new NacosException("No available server");
        }
        int index = Math.abs(currentIndex.incrementAndGet() % serverList.size());
        return serverList.get(index);
    }
    
    // 将当前失败的服务端放到列表尾部
    public void refreshCurrentServerAddr() {
        // 移除当前失败的服务端，放到队列末尾
        String current = serverList.get(currentIndex.get());
        serverList.remove(current);
        serverList.add(current);
    }
}
```

# 8. 完整客户端注册时序图

```
应用启动
    │
    ▼
NamingService.registerInstance("order-service", instance)
    │
    ▼
NacosNamingService.registerInstance()
    │
    ├─→ NamingClientProxy.registerService()
    │     │
    │     ├─→ NamingGrpcClientProxy.registerService()   ← 2.x 路径
    │     │     │  gRPC 注册 + 心跳注册
    │     │     └─→ rpcClient.request(InstanceRequest)
    │     │
    │     └─→ NamingHttpClientProxy.registerService()   ← 1.x 路径
    │           │  HTTP POST /nacos/v1/ns/instance
    │           └─→ reqApi()
    │
    └─→ BeatReactor.addBeatInfo()
          │
          └─→ executorService.scheduleAtFixedRate(BeatTask, 5s)
                │
                └─→ 每 5s 发送心跳
                      │
                      ├─→ NamingProxy.sendBeat()
                      │      ★ 心跳同时承担续约职责
                      │
                      └─→ 处理服务端返回
                             └─→ 更新心跳间隔 / 标记不健康
```

# 9. 异常处理与重试机制

```java
// NamingHttpClientProxy.java
public String reqApi(String api, Map<String, String> params, String method) 
        throws NacosException {
    
    int retryTimes = 3;  // 最大重试 3 次
    
    for (int i = 0; i < retryTimes; i++) {
        try {
            // 获取当前服务端地址
            String serverAddr = getServerListManager().getCurrentServer();
            
            // 发起 HTTP 请求
            String url = "http://" + serverAddr + api;
            HttpResult result = httpClient.request(url, params, method);
            
            if (result.isSuccess()) {
                return result.getData();
            }
            
        } catch (Exception e) {
            log.warn("Request failed, retry {}/{}", i + 1, retryTimes, e);
            
            if (i == retryTimes - 1) {
                // 最后一次重试失败 → 尝试从磁盘缓存读取
                throw new NacosException(NacosException.SERVER_ERROR, e.getMessage());
            }
            
            // 切换到下一个服务端
            getServerListManager().refreshCurrentServerAddr();
        }
    }
    throw new NacosException(NacosException.SERVER_ERROR, "All servers failed");
}
```
