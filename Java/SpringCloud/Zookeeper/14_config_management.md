
# 1. 配置管理概述

配置管理是分布式系统的核心需求之一。ZooKeeper 的**数据模型**与**Watch 机制**天然适合做配置中心：数据存在 ZNode 中，变更通过 Watch 推送给所有订阅者。

```text
配置中心要解决的问题：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 配置统一管理：所有环境、所有节点的配置集中存储                     |
|  2. 动态更新：配置变更后实时推送给应用，无需重启                       |
|  3. 版本管理：记录配置变更历史，支持回滚                               |
|  4. 灰度发布：部分节点先使用新配置，验证后全量                         |
|  5. 权限控制：敏感配置（密码、密钥）的安全管控                         |
|                                                                       |
+----------------------------------------------------------------------+
```

# 2. 基础原理

```text
ZK 配置管理核心流程：

+--------------------------------------------------------------------------+
|                                                                           |
|  配置管理端                          ZooKeeper                        应用 |
|      |                                  |                               |  |
|      |  1. 创建/更新配置节点              |                               |  |
|      +---- create/setData /config/db --->|                               |  |
|      |                                  |                               |  |
|      |                                  |  2. Watch 触发                |  |
|      |                                  +----- WatchedEvent -------->|  |
|      |                                  |                               |  |
|      |                                  |  3. 拉取最新配置             |  |
|      |                                  |<---- getData /config/db -----|  |
|      |                                  +---- 返回最新数据 ----------->|  |
|      |                                  |                               |  |
|      |                                  |  4. 应用新配置（无需重启）    |  |
|      |                                  |                               |  |
+--------------------------------------------------------------------------+
```

# 3. 配置存储模型

## 3.1 常用 ZNode 结构

```text
配置目录结构（推荐）：

/config (持久节点)
  /application (持久节点，公共配置)
    data: |
      db.url=jdbc:mysql://localhost:3306/mydb
      db.timeout=5000
      redis.host=localhost
      redis.port=6379

  /order-service (持久节点，应用专属配置)
    data: |
      server.port=8081
      order.expire.days=30

  /order-service,dev (持久节点，开发环境覆盖)
    data: |
      db.url=jdbc:mysql://dev-db:3306/mydb

  /order-service,prod (持久节点，生产环境覆盖)
    data: |
      db.url=jdbc:mysql://prod-db:3306/mydb
      db.password=${enc:encrypted_password}
```

## 3.2 配置分层原则

```text
配置优先级（从低到高）：
+----------------------------------------------------------------------+
|                                                                       |
|  1. application（公共配置）                                          |
|     · 所有服务共享的基础配置                                          |
|                                                                       |
|  2. application,{profile}（公共 + 环境）                              |
|     · 按环境区分的公共配置（dev/test/prod）                            |
|                                                                       |
|  3. {service-name}（应用专属配置）                                    |
|     · 每个微服务独有的配置                                            |
|                                                                       |
|  4. {service-name},{profile}（应用 + 环境）                           |
|     · 某应用在某环境的具体配置（最高优先级）                          |
|                                                                       |
+----------------------------------------------------------------------+
```

# 4. Spring Cloud Zookeeper Config

## 4.1 基本配置

```yaml
spring:
  application:
    name: order-service
  cloud:
    zookeeper:
      connect-string: zk1:2181,zk2:2181,zk3:2181
      config:
        enabled: true
        root: /config                    # 配置根路径
        default-context: application     # 默认上下文（应用名）
        profile-separator: ','           # 环境和应用名的分隔符
        watcher:
          enabled: true                  # 启用动态刷新监听
```

## 4.2 配置加载顺序

```text
Spring Cloud Zookeeper 自动加载以下路径的配置：

1. /config/application
2. /config/application,dev          (如果 profile 为 dev)
3. /config/order-service
4. /config/order-service,dev        (如果 profile 为 dev)

后加载的会覆盖先加载的同名配置项

注意：profile-separator 配置会影响路径格式：
  - 默认 ','   : /config/order-service,dev
  - 使用 '::'  : /config/order-service::dev
  - 使用 '-'   : /config/order-service-dev
```

## 4.3 动态刷新

```java
// @RefreshScope — 配置变更时自动刷新 Bean
@RestController
@RefreshScope
public class ConfigDemoController {

    @Value("${db.timeout:5000}")
    private int dbTimeout;

    @Value("${cache.ttl:300}")
    private int cacheTtl;

    @GetMapping("/config")
    public Map<String, Object> showConfig() {
        return Map.of(
            "dbTimeout", dbTimeout,
            "cacheTtl", cacheTtl
        );
    }
}

// 自定义刷新回调
@Component
public class ConfigChangeListener {
    @EventListener(RefreshScopeRefreshedEvent.class)
    public void onRefresh(RefreshScopeRefreshedEvent event) {
        System.out.println("Configuration refreshed at " +
            LocalDateTime.now());
        // 执行自定义重载逻辑
    }
}
```

## 4.4 手动刷新配置

```java
// 通过 Actuator 端点触发刷新
// POST /actuator/refresh

// 编程式刷新
@Autowired
private ContextRefresher contextRefresher;

public void refreshConfig() {
    Set<String> keys = contextRefresher.refresh();
    System.out.println("Refreshed keys: " + keys);
}
```

# 5. Curator 实现配置管理

```java
// 基于 Curator 实现一个简单的配置中心
@Component
public class ZkConfigCenter {
    private final CuratorFramework client;
    private final ConcurrentHashMap<String, String> configCache
        = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, List<ConfigChangeListener>>
        listeners = new ConcurrentHashMap<>();

    public ZkConfigCenter(CuratorFramework client) {
        this.client = client;
    }

    // 获取配置值
    public String getConfig(String key) {
        return configCache.get(key);
    }

    public int getInt(String key, int defaultValue) {
        String val = configCache.get(key);
        return val != null ? Integer.parseInt(val) : defaultValue;
    }

    // 初始化配置并注册监听
    public void initConfig(String configPath) throws Exception {
        // 确保路径存在
        if (client.checkExists().forPath(configPath) == null) {
            client.create().creatingParentsIfNeeded()
                .forPath(configPath);
        }

        // 使用 NodeCache 监听配置变化
        NodeCache nodeCache = new NodeCache(client, configPath);
        nodeCache.getListenable().addListener(() -> {
            ChildData data = nodeCache.getCurrentData();
            if (data != null) {
                String content = new String(data.getData());
                updateLocalCache(content);
                notifyListeners();
            }
        });
        nodeCache.start(true);

        // 初始化本地缓存
        ChildData data = nodeCache.getCurrentData();
        if (data != null) {
            updateLocalCache(new String(data.getData()));
        }
    }

    // 解析 Properties 格式的配置
    private void updateLocalCache(String content) {
        configCache.clear();
        try {
            Properties props = new Properties();
            props.load(new StringReader(content));
            for (String key : props.stringPropertyNames()) {
                configCache.put(key, props.getProperty(key));
            }
        } catch (IOException e) {
            throw new RuntimeException("Config parse error", e);
        }
    }

    // 注册自定义配置变更监听器
    public void addListener(String key, ConfigChangeListener listener) {
        listeners.computeIfAbsent(key, k -> new CopyOnWriteArrayList<>())
            .add(listener);
    }

    private void notifyListeners() {
        listeners.forEach((key, list) -> {
            String newValue = configCache.get(key);
            list.forEach(l -> l.onChange(key, newValue));
        });
    }

    public interface ConfigChangeListener {
        void onChange(String key, String newValue);
    }
}
```

# 6. 分层多节点配置模式

```java
// 高级配置管理：支持多节点分层覆盖
@Component
public class HierarchicalConfigCenter {
    private final CuratorFramework client;
    private final TreeCache treeCache;

    public HierarchicalConfigCenter(CuratorFramework client) {
        this.client = client;
        // 监听整个 /config 子树
        this.treeCache = TreeCache.newBuilder(client, "/config")
            .setCacheData(true)
            .build();
    }

    @PostConstruct
    public void init() throws Exception {
        treeCache.getListenable().addListener((c, event) -> {
            if (event.getType() == TreeCacheEvent.Type.NODE_UPDATED
                || event.getType() == TreeCacheEvent.Type.NODE_ADDED) {
                rebuildConfig();
            }
        });
        treeCache.start();
    }

    // 多层配置合并（后加载的覆盖前面的）
    private Map<String, String> rebuildConfig() {
        Map<String, String> merged = new LinkedHashMap<>();
        String[] layers = {
            "/config/application",
            "/config/application," + getProfile(),
            "/config/" + getServiceName(),
            "/config/" + getServiceName() + "," + getProfile()
        };

        for (String layer : layers) {
            ChildData data = treeCache.getCurrentData(layer);
            if (data != null) {
                Properties props = parseProperties(data.getData());
                props.forEach((k, v) -> merged.put(
                    (String) k, (String) v));
            }
        }
        return merged;
    }

    private Properties parseProperties(byte[] data) {
        Properties props = new Properties();
        try {
            props.load(new StringReader(new String(data)));
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        return props;
    }

    private String getProfile() {
        return System.getProperty("spring.profiles.active", "default");
    }

    private String getServiceName() {
        return System.getProperty("spring.application.name", "unknown");
    }
}
```

# 7. 配置版本管理与回滚

```java
// 简单配置版本记录方案
public class ConfigVersionManager {
    private final CuratorFramework client;
    private static final String VERSION_PATH = "/config-versions";

    // 更新配置前先保存历史版本
    public void updateConfigWithVersion(String configPath,
                                        String newData) throws Exception {
        // 1. 读取当前版本和内容
        Stat stat = new Stat();
        byte[] currentData = client.getData()
            .storingStatIn(stat)
            .forPath(configPath);

        // 2. 保存历史版本
        String historyPath = VERSION_PATH + configPath + "/" +
            System.currentTimeMillis() + "_v" + stat.getVersion();
        client.create()
            .creatingParentsIfNeeded()
            .forPath(historyPath, currentData);

        // 3. 更新配置
        client.setData().forPath(configPath, newData.getBytes());
    }

    // 回滚到上一个版本
    public void rollback(String configPath) throws Exception {
        List<String> versions = client.getChildren()
            .forPath(VERSION_PATH + configPath);
        if (!versions.isEmpty()) {
            // 取最新的历史版本
            String latestVersion = versions.get(versions.size() - 1);
            byte[] historyData = client.getData()
                .forPath(VERSION_PATH + configPath + "/" + latestVersion);

            // 恢复到该版本
            client.setData().forPath(configPath, historyData);
        }
    }

    // 查看版本历史
    public List<String> getVersionHistory(String configPath)
            throws Exception {
        String historyPath = VERSION_PATH + configPath;
        if (client.checkExists().forPath(historyPath) == null) {
            return Collections.emptyList();
        }
        return client.getChildren().forPath(historyPath);
    }
}
```

# 8. 灰度发布

```text
基于版本标签的灰度发布方案：

/config
  /order-service (持久节点，稳定版配置 -> v1.0)
  /order-service-beta (持久节点，灰度版配置 -> v1.1)

灰度策略：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 新建灰度配置节点：/config/order-service-beta                     |
|  2. 在实例 metadata 中打标签：{"release-channel": "beta"}            |
|  3. 灰度实例读取 /config/order-service-beta                          |
|  4. 稳定实例继续读取 /config/order-service                           |
|  5. 验证通过后，将 beta 配置同步到稳定版                              |
|  6. 移除 beta 标签，全量切换                                         |
|                                                                       |
+----------------------------------------------------------------------+
```

# 9. 安全配置管理

```text
密码等敏感配置的处理：

1. 不要在 ZK 中存储明文密码
   +----------------------------------------------------------------+
   |  /config/db                                                    |
   |    data:                                                        |
   |      db.url=...                                                 |
   |      db.username=app_user                                       |
   |      db.password=${enc:AES256:base64encodedciphertext}          |
   +----------------------------------------------------------------+

2. 结合密钥管理服务（KMS）
   +----------------------------------------------------------------+
   |  配置 ZNode 只存储密钥的引用：                                   |
   |    db.password.ref=kms://prod/db-password                        |
   |                                                                  |
   |  应用程序启动时从 KMS 获取实际的密码值                           |
   +----------------------------------------------------------------+

3. 使用 ACL 限制配置访问
   +----------------------------------------------------------------+
   |  /config                                                       |
   |    /public    -> ACL: world:anyone:r                            |
   |    /sensitive -> ACL: digest:admin:...:r                        |
   +----------------------------------------------------------------+
```

# 10. 配置管理最佳实践

```text
配置管理最佳实践：

1. 配置分层设计
   +----------------------------------------------------------------+
   |  公共配置 / 环境配置 / 应用配置 / 版本标签                       |
   |  按优先级顺序合并，支持灵活覆盖                                   |
   +----------------------------------------------------------------+

2. 保持 ZNode 数据精简
   +----------------------------------------------------------------+
   |  每个 ZNode 配置数据控制在 100KB 以内                            |
   |  大型配置文件（如上千键值对）考虑拆分为多个 ZNode               |
   +----------------------------------------------------------------+

3. 使用 Watch 实现动态刷新
   +----------------------------------------------------------------+
   |  配置变更实时推送，配合 @RefreshScope 实现热更新                |
   |  对不支持热更新的 Bean，提供重启标记                            |
   +----------------------------------------------------------------+

4. 配置变更审计
   +----------------------------------------------------------------+
   |  记录每次配置变更的操作人、时间、变更前后内容                    |
   |  可通过 ZK 事务日志 + 应用日志实现                              |
   +----------------------------------------------------------------+

5. 本地缓存兜底
   +----------------------------------------------------------------+
   |  ZK 不可用时，使用本地缓存的配置                                |
   |  定期将 ZK 配置同步到本地文件                                    |
   +----------------------------------------------------------------+

6. 配置规范化
   +----------------------------------------------------------------+
   |  使用标准命名：{app}.{module}.{key}                              |
   |  如：order.db.timeout, order.cache.ttl                           |
   +----------------------------------------------------------------+

7. 变更通知
   +----------------------------------------------------------------+
   |  重要配置变更时，通过消息通知相关人员                            |
   |  配置回滚后也需要通知                                            |
   +----------------------------------------------------------------+
```
