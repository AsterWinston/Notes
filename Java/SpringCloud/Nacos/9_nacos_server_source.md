
# 1. 服务端启动流程

## 1.1 Nacos 应用入口

```java
// Nacos.java
@SpringBootApplication(scanBasePackages = "com.alibaba.nacos")
@ServletComponentScan
@EnableScheduling
public class Nacos {
    
    public static void main(String[] args) {
        SpringApplication.run(Nacos.class, args);
    }
}
```

## 1.2 启动时核心初始化

```java
// NacosNamingService.java (服务端模块)
@Service
public class NacosNamingService {
    
    @PostConstruct
    public void init() {
        // 1. 初始化服务管理器
        serviceManager.init();
        
        // 2. 初始化一致性协议层
        consistencyService.init();
        
        // 3. 初始化集群管理器
        clusterManager.init();
        
        // 4. 初始化健康检查处理器
        healthCheckProcessor.init();
        
        // 5. 初始化推送服务
        pushService.init();
        
        // 6. 从数据库恢复服务数据
        loadFromStorage();
    }
}
```

# 2. 服务管理器（ServiceManager）

## 2.1 核心数据结构

```java
// ServiceManager.java
@Component
public class ServiceManager implements RecordListener<Service> {
    
    // 服务注册表（内存）: namespace -> serviceMap
    private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
    
    // 一致性服务
    @Autowired
    private ConsistencyService consistencyService;
    
    // Distro 一致性服务
    @Autowired
    private DistroConsistencyServiceImpl distroConsistencyService;
    
    // 服务初始化
    public void init() {
        // 注册 Distro 数据监听器
        distroConsistencyService.listen(KeyBuilder.SERVICE_META_KEY_PREFIX, this);
        
        // 定时执行服务心跳检测
        GlobalExecutor.schedule(new ServiceReporter(), 60000, TimeUnit.MILLISECONDS);
    }
    
    /**
     * 获取或创建服务
     */
    public Service getService(String namespaceId, String serviceName) {
        return serviceMap
            .computeIfAbsent(namespaceId, k -> new ConcurrentHashMap<>())
            .get(serviceName);
    }
    
    /**
     * 创建空服务（懒初始化）
     */
    public void createEmptyService(String namespaceId, String serviceName, 
                                    boolean ephemeral) {
        Service service = getService(namespaceId, serviceName);
        if (service != null) {
            return;
        }
        
        Service newService = new Service(serviceName);
        newService.setNamespaceId(namespaceId);
        newService.setEphemeral(ephemeral);
        
        // 初始化服务集群
        Cluster cluster = new Cluster(DEFAULT_CLUSTER_NAME, newService);
        cluster.setHealthChecker(new TcpSuperSenseProcessor());
        newService.addCluster(cluster);
        
        // 写入内存注册表
        serviceMap.get(namespaceId).put(serviceName, newService);
    }
    
    /**
     * 注册实例
     */
    public void registerInstance(String namespaceId, String serviceName, 
                                  Instance instance) {
        createEmptyService(namespaceId, serviceName, instance.isEphemeral());
        
        Service service = getService(namespaceId, serviceName);
        
        // 将实例添加到服务中
        addInstance(namespaceId, serviceName, instance.isEphemeral(), 
                   instance.getClusterName(), instance);
        
        // 发布服务变更事件（触发集群同步和推送）
        getPushService().serviceChanged(service);
    }
}
```

## 2.2 实例添加流程

```java
// ServiceManager.java
public void addInstance(String namespaceId, String serviceName, 
                        boolean ephemeral, String clusterName, Instance... ips) {
    
    String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);
    
    Service service = getService(namespaceId, serviceName);
    
    synchronized (service) {
        List<Instance> updatedInstances = new ArrayList<>();
        
        for (Instance ip : ips) {
            // 检查是否已存在
            Instance oldInstance = service.getAllInstance(ip);
            
            if (oldInstance != null) {
                // 更新已有实例
                oldInstance.copy(ip);
            } else {
                // 新增实例
                if (ip.isEphemeral()) {
                    // 临时实例 → 添加到集群的临时实例集合
                    service.getClusterMap().get(clusterName)
                           .getEphemeralInstances().add(ip);
                } else {
                    // 持久实例 → 通过一致性协议写入
                    service.getClusterMap().get(clusterName)
                           .getPersistentInstances().add(ip);
                }
                updatedInstances.add(ip);
            }
        }
        
        // 更新服务最后修改时间
        service.setLastModifiedMillis(System.currentTimeMillis());
        
        // 通过一致性协议同步到其他节点
        consistencyService.put(key, service);
        
        // 发布实例变更事件
        NotifyCenter.publishEvent(
            new InstanceChangeEvent(namespaceId, serviceName, updatedInstances));
    }
}
```

# 3. 一致性协议层

## 3.1 协议选择策略

```java
// ConsistencyService.java (接口)
public interface ConsistencyService {
    
    void put(String key, Record value);
    
    void remove(String key);
    
    Datum get(String key);
    
    void listen(String key, RecordListener listener);
}

// DelegateConsistencyServiceImpl.java (代理实现)
@Service("consistencyDelegate")
public class DelegateConsistencyServiceImpl implements ConsistencyService {
    
    @Autowired
    private PersistentConsistencyService persistentConsistencyService;  // 持久实例 (CP)
    
    @Autowired
    private EphemeralConsistencyService ephemeralConsistencyService;    // 临时实例 (AP)
    
    @Override
    public void put(String key, Record value) {
        // 根据 Key 判断使用哪种协议
        mapConsistencyService(key).put(key, value);
    }
    
    private ConsistencyService mapConsistencyService(String key) {
        // KeyBuilder 生成的 Key 包含 ephemeral 标识
        if (KeyBuilder.matchEphemeralKey(key)) {
            return ephemeralConsistencyService;  // AP: Distro 协议
        }
        return persistentConsistencyService;     // CP: Raft 协议
    }
}
```

## 3.2 Distro 协议（AP 模式）

```java
// DistroConsistencyServiceImpl.java
@Service("distroConsistencyService")
public class DistroConsistencyServiceImpl implements EphemeralConsistencyService {
    
    // 本节点数据存储: key → Datum (服务实例数据)
    private final Map<String, Datum> dataStore = new ConcurrentHashMap<>();
    
    // Distro 任务引擎
    @Autowired
    private DistroTaskEngineHolder distroTaskEngineHolder;
    
    // 服务端成员管理器（获取其他节点信息）
    @Autowired
    private ServerMemberManager memberManager;
    
    @Override
    public void put(String key, Record value) {
        // 1. 写入本节点内存
        onPut(key, value);
        
        // 2. 异步同步到所有其他节点
        distroProtocol.sync(new DistroKey(key), DataOperation.ADD, 1000L);
    }
    
    public void onPut(String key, Record value) {
        if (KeyBuilder.matchEphemeralInstanceListKey(key)) {
            // 临时实例列表
            Datum datum = new Datum();
            datum.key = key;
            datum.value = value;
            dataStore.put(key, datum);
            
            // 通知本地监听器（如 PushService）
            notifyListener(key, DataOperation.CHANGE);
        }
    }
    
    // Distro 数据同步（发送到其他节点）
    public void sync(DistroKey distroKey, DataOperation action, long delay) {
        for (Member member : memberManager.allMembersWithoutSelf()) {
            DistroTask task = new DistroTask();
            task.setAction(action);
            task.setKeys(Collections.singletonList(distroKey));
            
            // 提交异步同步任务
            distroTaskEngineHolder.getDelayTaskEngine()
                .addTask(member.getAddress(), task, delay);
        }
    }
}
```

## 3.3 Raft 协议（CP 模式）

```java
// RaftConsistencyServiceImpl.java
@Service("raftConsistencyService")
public class RaftConsistencyServiceImpl implements PersistentConsistencyService {
    
    @Autowired
    private RaftCore raftCore;
    
    // 数据存储: group → Raft 组信息
    private final Map<String, RaftGroupInfo> raftGroupInfoMap = new ConcurrentHashMap<>();
    
    @Override
    public void put(String key, Record value) {
        try {
            // 1. 根据 Key 确定 Raft Group
            String groupKey = GroupKey.getGroupKey(key);
            RaftGroupInfo groupInfo = raftGroupInfoMap.computeIfAbsent(groupKey, 
                k -> new RaftGroupInfo(raftCore));
            
            // 2. 写入 Raft 日志（通过 JRaft 引擎）
            raftCore.signalPublish(groupInfo.getRaftGroup(), key, value);
            
        } catch (Exception e) {
            throw new ConsistencyException("Raft put failed", e);
        }
    }
    
    @Override
    public Datum get(String key) {
        // 强一致性读（通过 Raft read-index）
        return raftCore.getDatum(key);
    }
}
```

# 4. 服务变更推送

## 4.1 PushService 核心实现

```java
// PushService.java
@Component
public class PushService implements ApplicationListener<ServiceChangeEvent> {
    
    // 订阅者 Map: serviceName → clientId → Subscriber
    private final Map<String, Map<String, PushClient>> clientMap = new ConcurrentHashMap<>();
    
    // gRPC 推送管理器（Nacos 2.x）
    @Autowired
    private GrpcPushService grpcPushService;
    
    // UDP 推送管理器（Nacos 1.x）
    @Autowired
    private UdpPushService udpPushService;
    
    // 监听服务变更事件
    @Override
    public void onApplicationEvent(ServiceChangeEvent event) {
        Service service = event.getService();
        String serviceName = service.getName();
        String namespaceId = service.getNamespaceId();
        
        // 向订阅了该服务的所有客户端推送变更
        Map<String, PushClient> clients = clientMap.get(
            ServiceInfo.getKey(serviceName, ""));
        
        if (clients != null) {
            for (PushClient client : clients.values()) {
                // 构建推送数据
                ServiceInfo serviceInfo = buildServiceInfo(service, client);
                
                // 通过 gRPC 推送（优先）
                if (client.isGrpc()) {
                    grpcPushService.push(client, serviceInfo);
                } else {
                    // 通过 UDP 推送（1.x 兼容）
                    udpPushService.push(client, serviceInfo);
                }
            }
        }
    }
    
    // 构建推送的服务信息
    private ServiceInfo buildServiceInfo(Service service, PushClient client) {
        ServiceInfo serviceInfo = new ServiceInfo();
        serviceInfo.setName(service.getName());
        serviceInfo.setGroupName(service.getGroupName());
        serviceInfo.setClusters(client.getClusters());
        
        List<Instance> hosts = new ArrayList<>();
        for (Cluster cluster : service.getClusterMap().values()) {
            // 根据 client 的集群过滤
            if (client.getClusters().contains(cluster.getName())) {
                hosts.addAll(cluster.allIPs(
                    client.isHealthyOnly()  // 是否只要健康实例
                ));
            }
        }
        serviceInfo.setHosts(hosts);
        serviceInfo.setChecksum(calculateChecksum(hosts));
        serviceInfo.setLastRefTime(System.currentTimeMillis());
        
        return serviceInfo;
    }
}
```

## 4.2 gRPC 推送（Nacos 2.x）

```java
// GrpcPushService.java
@Component
public class GrpcPushService {
    
    @Autowired
    private ConnectionManager connectionManager;  // gRPC 连接管理器
    
    public void push(PushClient client, ServiceInfo serviceInfo) {
        // 构建 NotifySubscriberRequest
        NotifySubscriberRequest request = new NotifySubscriberRequest();
        request.setServiceName(serviceInfo.getName());
        request.setGroupName(serviceInfo.getGroupName());
        request.setNamespace(client.getNamespaceId());
        request.setServiceInfo(convertToGrpcServiceInfo(serviceInfo));
        
        // 通过 gRPC 双向流推送
        Connection connection = connectionManager.getConnection(client.getConnectionId());
        if (connection != null) {
            connection.request(request, new AbstractRequestCallBack() {
                @Override
                public void onResponse(Object response) {
                    // 推送成功
                }
                
                @Override
                public void onException(Throwable e) {
                    log.error("gRPC push failed for client: {}", client, e);
                }
            });
        }
    }
}
```

# 5. 配置中心服务端

## 5.1 ConfigController 核心 API

```java
// ConfigController.java
@RestController
@RequestMapping(Constants.CONFIG_CONTROLLER_PATH)  // /v1/cs/configs
public class ConfigController {
    
    @Autowired
    private ConfigServletInner inner;  // 配置核心处理逻辑
    
    // 获取配置
    @GetMapping
    public void getConfig(HttpServletRequest request, HttpServletResponse response) {
        String dataId = request.getParameter("dataId");
        String group = request.getParameter("group");
        String tenant = request.getParameter("tenant");
        
        String config = inner.doGetConfig(request, response, dataId, group, tenant, ...);
        response.getWriter().write(config);
    }
    
    // 发布/更新配置
    @PostMapping
    public Boolean publishConfig(HttpServletRequest request, HttpServletResponse response) {
        String dataId = request.getParameter("dataId");
        String group = request.getParameter("group");
        String tenant = request.getParameter("tenant");
        String content = request.getParameter("content");
        String type = request.getParameter("type");
        
        return inner.doPublishConfig(dataId, group, tenant, content, type, ...);
    }
    
    // 删除配置
    @DeleteMapping
    public Boolean deleteConfig(HttpServletRequest request, HttpServletResponse response) {
        String dataId = request.getParameter("dataId");
        String group = request.getParameter("group");
        String tenant = request.getParameter("tenant");
        
        return inner.doDeleteConfig(dataId, group, tenant, ...);
    }
}
```

## 5.2 ConfigServletInner 配置发布核心

```java
// ConfigServletInner.java
@Service
public class ConfigServletInner {
    
    @Autowired
    private PersistService persistService;
    
    // 发布配置
    public boolean doPublishConfig(String dataId, String group, String tenant,
                                    String content, String type, String configTags) {
        // 1. 计算 MD5
        String md5 = MD5Utils.md5Hex(content, "UTF-8");
        
        // 2. 持久化到数据库
        persistService.insertOrUpdate(tenant, dataId, group, content, type, md5);
        
        // 3. 写入磁盘 dump 文件
        DumpService.dump(dataId, group, tenant, content, md5, type);
        
        // 4. 发布本地数据变更事件（触发长轮询或 gRPC 通知）
        NotifyCenter.publishEvent(
            new LocalDataChangeEvent(GroupKey.getKeyTenant(dataId, group, tenant)));
        
        return true;
    }
    
    // 获取配置
    public String doGetConfig(HttpServletRequest request, HttpServletResponse response,
                               String dataId, String group, String tenant) {
        // 1. 检查是否有 Beta 配置（灰度发布）
        String betaIps = request.getParameter("betaIps");
        if (StringUtils.isNotBlank(betaIps)) {
            // 获取 Beta 配置
            return persistService.findConfigInfo4Beta(dataId, group, tenant);
        }
        
        // 2. 获取正式配置
        ConfigInfo configInfo = persistService.findConfigInfo(dataId, group, tenant);
        return configInfo != null ? configInfo.getContent() : "";
    }
}
```

# 6. 集群节点管理

## 6.1 ServerMemberManager

```java
// ServerMemberManager.java
@Component
public class ServerMemberManager implements ApplicationListener<WebServerInitializedEvent> {
    
    // 集群成员列表
    private volatile List<Member> memberList = new ArrayList<>();
    
    // 自身成员信息
    private volatile Member self;
    
    // 初始化
    @PostConstruct
    public void init() {
        // 1. 读取 cluster.conf 文件获取集群节点列表
        List<String> serverList = readClusterConf();
        
        // 2. 初始化成员信息
        for (String server : serverList) {
            Member member = new Member();
            member.setAddress(server);
            memberList.add(member);
        }
        
        // 3. 确定自身成员
        self = determineSelfMember();
        
        // 4. 启动心跳检测（检测其他节点健康状态）
        GlobalExecutor.schedule(new MemberHealthChecker(), 5000, 3000, TimeUnit.MILLISECONDS);
    }
    
    // 心跳检测
    class MemberHealthChecker implements Runnable {
        @Override
        public void run() {
            for (Member member : memberList) {
                if (member.equals(self)) continue;
                
                // 向其他节点发送心跳
                try {
                    boolean healthy = healthCheck(member);
                    member.setState(healthy ? MemberState.UP : MemberState.DOWN);
                } catch (Exception e) {
                    member.setState(MemberState.DOWN);
                }
            }
        }
    }
}
```

# 7. 请求处理过滤器链

```java
// Nacos 请求处理过滤器链
@Component
public class NacosRequestFilterChain {
    
    // TpsControlRequestFilter     → 限流控制
    // AuthRequestFilter           → 鉴权检查
    // TrafficReviseFilter         → 流量修正
    // DistroFilter                → Distro 路由（ap 请求转发到对应节点）
    // TimingFilter                → 性能统计
    
    @Autowired
    private List<AbstractRequestFilter> filters;
    
    public void doFilter(ServletRequest request, ServletResponse response) {
        FilterChain chain = new DefaultFilterChain(filters);
        chain.doFilter(request, response);
    }
}
```

# 8. 服务端关键线程池

| 线程池名称 | 核心线程数 | 用途 |
|-----------|-----------|------|
| Naming-HealthCheck | CPU 核数 | 持久实例健康检查 |
| Naming-Push | CPU 核数 | 服务变更推送 |
| Naming-Distro | CPU 核数 | Distro 数据同步 |
| Naming-Raft | 1/节点 | Raft 日志复制 |
| Config-LongPolling | 动态 | 配置长轮询处理 |
| Config-Notify | CPU 核数 | 配置变更通知 |

```java
// GlobalExecutor.java
public class GlobalExecutor {
    
    // 健康检查线程池
    public static final ScheduledExecutorService healthCheckExecutor =
        ExecutorFactory.Managed.newScheduledExecutorService(
            "Naming-HealthCheck", 
            Runtime.getRuntime().availableProcessors());
    
    // 推送线程池
    public static final ExecutorService pushExecutor =
        ExecutorFactory.Managed.newExecutorService(
            "Naming-Push", 
            Runtime.getRuntime().availableProcessors(), 
            1024, 0, false);
    
    // Distro 同步线程池
    public static final ExecutorService distroExecutor =
        ExecutorFactory.Managed.newExecutorService(
            "Naming-Distro",
            Runtime.getRuntime().availableProcessors(),
            1024, 0, false);
}
```
