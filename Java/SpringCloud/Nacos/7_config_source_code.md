
# 1. 源码模块概览

配置中心的源码主要集中在以下模块：

```
nacos-config/               # 配置中心服务端
  ├── ConfigController       # 配置 REST API 控制器
  ├── LongPollingService     # 长轮询服务
  ├── DumpService            # 全量持久化服务
  ├── CacheItem              # 配置缓存条目
  └── ConfigChangePublisher  # 配置变更事件发布

nacos-client/               # 配置中心客户端
  ├── ConfigService          # 配置服务接口
  ├── NacosConfigService     # 配置服务实现
  ├── ClientWorker           # 客户端工作线程（长轮询核心）
  ├── ServerHttpAgent        # HTTP 通信代理
  └── CacheData              # 客户端缓存数据对象
```

# 2. 客户端初始化流程

## 2.1 NacosConfigService 构造函数

```java
// NacosConfigService.java
public class NacosConfigService implements ConfigService {
    
    private ClientWorker worker;        // 工作线程，负责拉取和监听配置
    private ServerHttpAgent agent;      // HTTP 通信代理
    private String namespace;           // 命名空间
    private ConfigFilterChainManager configFilterChainManager; // 过滤器链
    
    public NacosConfigService(Properties properties) throws NacosException {
        String serverAddr = properties.getProperty(PropertyKeyConst.SERVER_ADDR);
        
        // 1. 创建 HTTP 通信代理
        this.agent = new ServerHttpAgent(serverAddr);
        
        // 2. 创建客户端工作线程
        this.worker = new ClientWorker(this.agent, properties);
        
        // 3. 初始化命名空间
        this.namespace = properties.getProperty(PropertyKeyConst.NAMESPACE);
        
        // 4. 初始化过滤器链
        this.configFilterChainManager = new ConfigFilterChainManager(properties);
    }
}
```

## 2.2 ClientWorker 初始化

```java
// ClientWorker.java
public class ClientWorker implements Closeable {
    
    // 配置缓存 Map: dataId → CacheData
    private final AtomicReference<Map<String, CacheData>> cacheMap;
    
    // 定时任务执行器
    private final ScheduledExecutorService executor;
    private final ScheduledExecutorService executorService;
    
    // 长轮询线程（核心线程）
    final ConfigRpcTransportClient agent;
    
    public ClientWorker(ServerHttpAgent httpAgent, Properties properties) {
        this.agent = new ConfigRpcTransportClient(properties, ...);
        
        // 初始化缓存 Map
        cacheMap = new AtomicReference<>(new HashMap<>());
        
        // 定时任务线程池（核心线程数 = CPU 核数）
        executor = Executors.newScheduledThreadPool(2, ...);
        executorService = Executors.newScheduledThreadPool(
            Runtime.getRuntime().availableProcessors(), ...);
        
        // 启动长轮询检查任务（检查超时）
        executor.scheduleWithFixedDelay(() -> {
            checkConfigInfo();
        }, 0L, 2L, TimeUnit.SECONDS);
        
        // 启动长轮询主任务（批量监听）
        executorService.execute(new LongPollingRunnable());
    }
}
```

# 3. 配置拉取流程

## 3.1 首次拉取

```java
// NacosConfigService.java
@Override
public String getConfig(String dataId, String group, long timeoutMs) 
        throws NacosException {
    
    // 1. 先从本地缓存获取
    String content = getLocalConfig(dataId, group);
    if (content != null) {
        return content;
    }
    
    // 2. 缓存未命中，从服务端拉取
    return getConfigInner(namespace, dataId, group, timeoutMs);
}

private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) {
    // 1. 先检查本地快照
    String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
    if (content != null) {
        return content;
    }
    
    try {
        // 2. 构造请求
        String[] ct = worker.getServerConfig(dataId, group, tenant, timeoutMs);
        
        // 3. 将结果缓存到本地
        CacheData cache = worker.addCacheDataIfAbsent(dataId, group);
        cache.setContent(ct[0]);
        
        // 4. 写入本地快照
        LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, ct[0]);
        
        return ct[0];
    } catch (NacosException e) {
        // 5. 服务端不可用 → 读取本地快照兜底
        return LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
    }
}
```

## 3.2 CacheData 数据结构

```java
// CacheData.java
public class CacheData {
    
    public final String dataId;      // 配置 ID
    public final String group;       // 分组
    private volatile String content; // 配置内容
    private String md5;              // 内容 MD5
    private volatile boolean isInitializing = true;
    
    // 配置监听器列表
    private final CopyOnWriteArrayList<ManagerListenerWrap> listeners;
    
    // 配置内容是否已被使用
    private volatile boolean isConsistentWithServer = true;
    
    // 用于监听变更的 MD5 值（上次从服务端获取的）
    private volatile String lastModifiedTs;
    
    public CacheData(String dataId, String group) {
        this.dataId = dataId;
        this.group = group;
        this.listeners = new CopyOnWriteArrayList<>();
        this.md5 = getMD5String(content);
    }
    
    // 检查配置是否变更
    public void checkListenerMd5() {
        for (ManagerListenerWrap wrap : listeners) {
            String currentMd5 = getMD5String(content);
            if (!currentMd5.equals(wrap.lastCallMd5)) {
                wrap.lastCallMd5 = currentMd5;
                // 通知监听器
                wrap.invoke(content, currentMd5);
            }
        }
    }
}
```

# 4. 长轮询机制（核心）

## 4.1 Nacos 1.x 长轮询模型

Nacos 1.x 配置中心使用 **HTTP 长轮询（Long Polling）** 机制替代传统的短轮询或 WebSocket：

```
Client                                          Server
  │                                                │
  │ POST /nacos/v1/cs/configs/listener             │
  │ Headers: Long-Pulling-Timeout=30000            │
  │ Body: dataId^1group^1md5^1␁dataId^2group ^2md5^2│
  │───────────────────────────────────────────────>│
  │                                                │
  │                                                │ 检查 MD5 是否变化
  │                                                │──────┐
  │                                                │<─────┘
  │                                                │
  │                                                │ 如果没有变化: 挂起请求 29.5s
  │                                                │ 如果有变化: 立即返回变化项
  │                                                │
  │                                                │ 挂起到期仍无变化: 返回空
  │<───────────────────────────────────────────────│
  │                                                │
  │ 收到响应后，立即发起下一次长轮询                  │
  │───────────────────────────────────────────────>│
```

## 4.2 客户端长轮询实现

```java
// ClientWorker.java - LongPollingRunnable
class LongPollingRunnable implements Runnable {
    
    @Override
    public void run() {
        while (!closed) {
            try {
                // 1. 收集所有需要监听的配置项（dataId, group, md5）
                List<CacheData> cacheDatas = new ArrayList<>();
                for (CacheData cacheData : cacheMap.get().values()) {
                    if (cacheData.isUseLocalConfigInfo()) {
                        cacheDatas.add(cacheData);
                    }
                }
                
                if (cacheDatas.isEmpty()) {
                    Thread.sleep(100L);
                    continue;
                }
                
                // 2. 构建批量监听请求
                List<String> inInitializingCacheList = new ArrayList<>();
                StringBuilder sb = new StringBuilder();
                for (CacheData cacheData : cacheDatas) {
                    // 格式: dataId^2group^2md5^2tenant^1
                    sb.append(cacheData.dataId).append(WORD_SEPARATOR)
                      .append(cacheData.group).append(WORD_SEPARATOR)
                      .append(cacheData.md5).append(WORD_SEPARATOR)
                      .append(tenant).append(LINE_SEPARATOR);
                    
                    if (cacheData.isInitializing()) {
                        inInitializingCacheList.add(
                            GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, tenant));
                    }
                }
                
                // 3. 发送长轮询请求（默认超时 30s）
                List<String> changedGroups = checkUpdateDataIds(
                    sb.toString(), inInitializingCacheList);
                
                // 4. 处理变更的配置项
                for (String groupKey : changedGroups) {
                    String[] key = GroupKey.parseKey(groupKey);
                    String dataId = key[0];
                    String group = key[1];
                    String tenant = key[2];
                    
                    // 从服务端拉取最新配置内容
                    String content = getServerConfig(dataId, group, tenant, 3000L);
                    
                    // 更新本地缓存
                    CacheData cache = cacheMap.get().get(GroupKey.getKeyTenant(dataId, group, tenant));
                    cache.setContent(content);
                    
                    // 触发监听器回调
                    cache.checkListenerMd5();
                }
                
            } catch (Exception e) {
                log.error("LongPolling error", e);
                // 异常后等待一段时间再重试
                Thread.sleep(100L);
            }
        }
    }
}
```

## 4.3 服务端长轮询实现

```java
// LongPollingService.java
public class LongPollingService extends AbstractEventListener {
    
    // 长轮询超时（减去 500ms 以留出处理时间）
    private static final int LONG_POLLING_TIMEOUT = 29500;
    
    // 客户端长轮询任务队列（key = 配置组签名）
    final Queue<ClientLongPolling> allSubs = new ConcurrentLinkedQueue<>();
    
    // 处理客户端长轮询请求
    public String doPolling(HttpServletRequest request, HttpServletResponse response,
                             Map<String, String> clientMd5Map, 
                             List<String> changedGroups) {
        
        // 1. 检查是否有配置变更
        List<String> changedGroupKeys = checkUpdateDataIds(clientMd5Map);
        
        if (!changedGroupKeys.isEmpty()) {
            // 有变更，立即返回
            generateResponse(request, response, changedGroupKeys);
            return HttpServletResponse.SC_OK + "";
        }
        
        // 2. 没有变更，挂起请求（异步化）
        final ClientLongPolling clientLongPolling = new ClientLongPolling(
            request, response, clientMd5Map);
        
        // 3. 添加到轮询队列
        allSubs.add(clientLongPolling);
        
        // 4. 设置超时异步返回
        ConfigExecutor.executeLongPolling(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(LONG_POLLING_TIMEOUT);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                // 超时后，检查是否已被处理（配置可能中途变更了）
                boolean hasHandled = clientLongPolling.isHandled();
                if (!hasHandled) {
                    // 仍未处理 → 返回空（表示没有变更）
                    clientLongPolling.sendResponse(Collections.emptyList());
                    allSubs.remove(clientLongPolling);
                }
            }
        });
        
        // 返回异步标志
        return HttpServletResponse.SC_OK + "";
    }
    
    // 配置变更事件处理（关键回调）
    @Override
    public void onEvent(Event event) {
        if (event instanceof LocalDataChangeEvent) {
            LocalDataChangeEvent evt = (LocalDataChangeEvent) event;
            
            // 遍历所有挂起的长轮询客户端
            for (ClientLongPolling clientSub : allSubs) {
                // 检查这个变更是否影响了该客户端订阅的配置
                if (clientSub.clientMd5Map.containsKey(evt.groupKey)) {
                    // 移除该订阅并立即返回变更
                    allSubs.remove(clientSub);
                    clientSub.sendResponse(Arrays.asList(evt.groupKey));
                }
            }
        }
    }
}
```

# 5. Nacos 2.x gRPC 配置推送

## 5.1 ConfigRpcTransportClient

Nacos 2.x 使用 gRPC 双向流替代了 HTTP 长轮询：

```java
// ConfigRpcTransportClient.java
public class ConfigRpcTransportClient extends ConfigTransportClient {
    
    private RpcClient rpcClient;
    private ConfigChangeNotifyRequestHandler handler;
    
    // 订阅配置
    @Override
    public void executeConfigListen() {
        // 1. 构建批量监听请求
        ConfigBatchListenRequest request = new ConfigBatchListenRequest();
        for (CacheData cacheData : cacheMap) {
            request.addConfigListenContext(cacheData.dataId, 
                cacheData.group, cacheData.md5, cacheData.tenant);
        }
        
        // 2. 发送 gRPC 请求
        rpcClient.request(request, new RequestCallBack<ConfigChangeBatchListenResponse>() {
            @Override
            public void onResponse(ConfigChangeBatchListenResponse response) {
                for (ConfigContext configContext : response.getChangedConfigs()) {
                    // 有变更的配置 → 触发拉取最新内容
                    String content = agent.queryConfig(
                        configContext.getDataId(), 
                        configContext.getGroup(), ...);
                    
                    CacheData cacheData = getCacheData(configContext);
                    cacheData.setContent(content);
                    cacheData.checkListenerMd5();
                }
            }
        });
    }
    
    // 注册 gRPC 服务端推送处理器
    @Override
    public void startInternal() {
        // 注册服务端主动推送的回调
        rpcClient.registerServerRequestHandler(
            new ConfigChangeNotifyRequestHandler() {
                @Override
                public ConfigChangeNotifyResponse handle(
                        ConfigChangeNotifyRequest request, RequestMeta meta) {
                    
                    // 收到服务端推送的配置变更通知
                    String dataId = request.getDataId();
                    String group = request.getGroup();
                    
                    // 异步拉取最新配置
                    executor.execute(() -> {
                        String content = queryConfig(dataId, group, ...);
                        CacheData cacheData = getCacheData(dataId, group);
                        if (cacheData != null) {
                            cacheData.setContent(content);
                            cacheData.checkListenerMd5();
                        }
                    });
                    
                    return new ConfigChangeNotifyResponse();
                }
            });
    }
}
```

# 6. 配置持久化（服务端）

## 6.1 DumpService

```java
// DumpService.java
@Component
public class DumpService {
    
    @Autowired
    private PersistService persistService;
    
    // 全量持久化（启动时和定时执行）
    @PostConstruct
    public void init() {
        // 启动时从数据库加载全部配置到内存
        dumpAllToDisk();
        
        // 定期全量持久化到磁盘（默认 6h）
        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                dumpAllToDisk();
            }
        };
        Timer timer = new Timer("DumpTimer", true);
        timer.scheduleAtFixedRate(task, DUMP_INTERVAL, DUMP_INTERVAL);
    }
    
    // 全量持久化
    private void dumpAllToDisk() {
        // 1. 从数据库读取所有配置
        Page<ConfigInfo> configInfos = persistService.findAllConfigInfo();
        
        // 2. 写入磁盘文件
        for (ConfigInfo configInfo : configInfos) {
            String path = getDumpPath(configInfo.getDataId(), 
                configInfo.getGroup(), configInfo.getTenant());
            FileUtils.writeStringToFile(new File(path), configInfo.getContent(), "UTF-8");
        }
    }
    
    // 增量持久化（配置变更时触发）
    public void dump(String dataId, String group, String tenant, 
                     String content, String type) {
        // 1. 写入数据库
        persistService.insertOrUpdate(dataId, group, tenant, content, type);
        
        // 2. 写入磁盘
        String path = getDumpPath(dataId, group, tenant);
        FileUtils.writeStringToFile(new File(path), content, "UTF-8");
        
        // 3. 发布数据变更事件（通知长轮询客户端）
        NotifyCenter.publishEvent(
            new LocalDataChangeEvent(GroupKey.getKeyTenant(dataId, group, tenant)));
    }
}
```

## 6.2 数据库表结构

```sql
-- 配置信息主表
CREATE TABLE config_info (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    data_id VARCHAR(255) NOT NULL,
    group_id VARCHAR(128) NOT NULL,
    tenant_id VARCHAR(128) DEFAULT '' NOT NULL,
    app_name VARCHAR(128),
    content LONGTEXT NOT NULL,
    md5 VARCHAR(32),
    type VARCHAR(64),
    c_desc VARCHAR(256),
    c_use VARCHAR(64),
    effect VARCHAR(64),
    encrypted_data_key TEXT,
    gmt_create DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
    gmt_modified DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE KEY uk_configinfodatagrouptenant (data_id, group_id, tenant_id)
);

-- 配置历史版本表
CREATE TABLE his_config_info (
    id BIGINT PRIMARY KEY,
    nid BIGINT AUTO_INCREMENT UNIQUE,
    data_id VARCHAR(255),
    group_id VARCHAR(128),
    tenant_id VARCHAR(128),
    content LONGTEXT,
    md5 VARCHAR(32),
    op_type CHAR(10),      -- D:删除, U:更新, I:插入
    gmt_create DATETIME,
    gmt_modified DATETIME
);
```

# 7. Spring 配置加载机制

## 7.1 NacosPropertySourceLocator

Spring Cloud Alibaba Nacos 通过实现 `PropertySourceLocator` 接口将 Nacos 配置加载到 Spring Environment：

```java
// NacosPropertySourceLocator.java
@Order(0)
public class NacosPropertySourceLocator implements PropertySourceLocator {
    
    @Override
    public PropertySource<?> locate(Environment env) {
        NacosConfigProperties nacosConfigProperties = 
            Binder.get(env).bind(NacosConfigProperties.PREFIX, 
                NacosConfigProperties.class).orElse(new NacosConfigProperties());
        
        CompositePropertySource composite = new CompositePropertySource("NACOS");
        
        // 1. 加载共享配置（最低优先级）
        loadSharedConfiguration(composite, nacosConfigProperties, env);
        
        // 2. 加载扩展配置
        loadExtConfiguration(composite, nacosConfigProperties, env);
        
        // 3. 加载应用自身配置（最高优先级）
        loadApplicationConfiguration(composite, nacosConfigProperties, env);
        
        return composite;
    }
    
    private void loadApplicationConfiguration(CompositePropertySource composite,
                                              NacosConfigProperties properties,
                                              Environment environment) {
        // 构建 Data ID 和 Group
        String dataId = properties.getPrefix() + "-" + 
                       environment.getActiveProfiles()[0] + "." + 
                       properties.getFileExtension();
        String group = properties.getGroup();
        String namespace = properties.getNamespace();
        
        // 从 Nacos 获取配置
        String config = configService.getConfig(dataId, group, 5000);
        
        // 生成 PropertySource 并添加
        NacosPropertySource source = new NacosPropertySource(
            dataId, group, config, properties.getFileExtension());
        composite.addPropertySource(source);
    }
}
```

## 7.2 配置刷新机制

```java
// NacosContextRefresher.java
public class NacosContextRefresher 
        implements ApplicationListener<ApplicationReadyEvent> {
    
    private final ContextRefresher contextRefresher;
    
    // 注册 Nacos 配置监听器
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        registerNacosListenersForApplications();
    }
    
    private void registerNacosListenersForApplications() {
        for (NacosPropertySource source : nacosPropertySourceRepository.getAll()) {
            // 为每个 Nacos 配置源注册监听器
            configService.addListener(source.getDataId(), source.getGroup(), 
                new AbstractListener() {
                    @Override
                    public void receiveConfigInfo(String configInfo) {
                        // 配置变更后刷新 Spring Context
                        Set<String> refreshKeys = contextRefresher.refresh();
                        log.info("Refreshed keys: {}", refreshKeys);
                    }
                });
        }
    }
}
```

# 8. 配置管理关键流程图

```
                         Nacos 配置中心完整流程
                               
┌──────────────┐     注册配置监听        ┌──────────────────┐
│  Application │─────────────────────────>│ NacosConfigService│
│  Startup     │                          │ (Client)          │
└──────┬───────┘                          └────────┬─────────┘
       │                                           │
       │ 加载 bootstrap.yml                        │ addListener(dataId, group)
       │ (NacosPropertySourceLocator)              │
       │──────────────────────────────────────────>│
       │                                           │
       │                                           │ getConfig() 首次拉取
       │                                           │────────────────────>│
       │                                           │                     │
       │                                           │<── 返回配置内容 ─────│
       │<── 返回 PropertySource ───────────────────│                     │
       │                                           │                     │
       │ 初始化 ContextRefresher                    │                     │
       │ (注册配置变更监听器)                       │ 长轮询监听           │
       │──────────────────────────────────────────>│──────────┐          │
       │                                           │<─────────┘          │
       │                                           │                     │
       │                              配置变更通知 │                     │
       │                                           │<────────────────────│ (Nacos Server)
       │                                           │                     │
       │                                           │ 拉取最新配置         │
       │                                           │────────────────────>│
       │                                           │<── 返回新配置 ──────│
       │                                           │                     │
       │ 发布 RefreshEvent                         │                     │
       │<──────────────────────────────────────────│                     │
       │                                           │                     │
       │ ContextRefresher.refresh()                │                     │
       │ @RefreshScope Bean 重建                    │                     │
       │ 新的 @Value 值生效                         │                     │
```

# 9. 性能与容错设计

## 9.1 MD5 比对机制

配置变更是通过 MD5 值对比来判断的，避免传输完整配置内容：

```java
// 服务端
String md5 = MD5.getInstance().getMD5String(content);

// 客户端
boolean isChanged = !clientMd5.equals(serverMd5);
```

## 9.2 异步全量拉取

配置变更后不是立即更新，而是通过事件驱动异步拉取：

```java
// 收到配置变更通知 → 异步队列 → 拉取最新配置 → 更新缓存 → 通知监听器
NotifyCenter.publishEvent(event);
// → ConfigChangePublisher 异步处理
// → ConfigExecutor.execute()
// → 拉取配置 & 更新 CacheData
```

## 9.3 快照容错

配置中心客户端三层容错保护：

| 层级 | 机制 | 说明 |
|------|------|------|
| 层级 1 | Nacos Server | 正常情况，从服务端获取最新配置 |
| 层级 2 | 本地快照 | 服务端不可用时，使用本地磁盘缓存的配置 |
| 层级 3 | 应用默认值 | 无任何缓存时，使用代码中的默认值 |
