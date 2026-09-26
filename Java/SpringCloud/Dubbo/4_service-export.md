
# 1. 什么是服务导出

服务导出是将一个 Java 对象（Service 实现类）变成 Dubbo 框架可以调用的远程服务的全过程。导出完成后，其他应用可以通过 RPC 调用该服务。

```
ServiceImpl (业务实现)
    │
    ▼
ServiceConfig (配置载体)
    │
    ▼
doExport() (导出入口)
    │
    ├──→ doExportUrls()          (URL 组装)
    │       │
    │       ▼
    │   loadRegistries()           (获取注册中心地址)
    │       │
    │       ▼
    │   遍历每个协议，遍历每个注册中心
    │       │
    │       ├──→ doExportUrlsFor1Protocol()
    │       │       │
    │       │       ├──→ 组装 URL
    │       │       ├──→ proxyFactory.getInvoker(ref, interface, url)
    │       │       └──→ protocol.export(invoker)
    │       │               │
    │       │               ├── RegistryProtocol
    │       │               │   ├── doLocalExport (本地暴露)
    │       │               │   │   └── DubboProtocol.export()
    │       │               │   │       ├── openServer (启动 Netty Server)
    │       │               │   │       └── exporterMap.put(key, exporter)
    │       │               │   │
    │       │               │   └── register (注册到注册中心)
    │       │               │       └── registry.register(url)
    │       │               │
    │       │               └── 返回 Exporter
```

# 2. @DubboService 注解

```java
@DubboService(
    version = "1.0.0",
    group = "dev",
    timeout = 3000,
    retries = 0,
    weight = 100,
    actives = 100,
    executes = 200,
    loadbalance = "random",
    cluster = "failover",
    protocol = "dubbo",
    registry = "nacosRegistry",
    delay = -1,
    validation = "true",
    parameters = {
        "key1", "value1",
        "key2", "value2"
    }
)
public class UserServiceImpl implements UserService {
    // ...
}
```

**关键属性**：

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `version` | "" | 服务版本，用于区分接口相同但版本不同的服务 |
| `group` | "" | 服务分组，同一接口不同分组之间调用隔离 |
| `timeout` | 1000 | 远程调用超时(ms) |
| `retries` | 2 | 失败重试次数（不含第一次） |
| `loadbalance` | "random" | 负载均衡策略 |
| `cluster` | "failover" | 集群容错策略 |
| `delay` | -1 | 延迟注册时间(ms)，-1 表示等 Spring 容器初始化完后延迟导出 |
| `weight` | 100 | 权重 |

# 3. 服务导出入口：ServiceConfig.doExport()

```java
public class ServiceConfig<T> extends AbstractServiceConfig {
    
    public synchronized void export() {
        // 防重复导出
        if (exported) { return; }
        exported = true;

        // 1. 刷新配置（补充默认值、从配置中心获取等）
        refresh();

        // 2. 检查并设置默认值
        checkAndUpdateSubConfigs();

        // 3. 导出服务
        doExport();
    }

    private void doExport() {
        // 组装所有协议地址（如: dubbo://192.168.1.100:20880）
        List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this);

        // 遍历每个注册协议，以 RegistryProtocol 为入口
        for (ProtocolConfig protocolConfig : protocols) {
            // 每个注册中心 + 每个协议 组合导出一组 URL
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }

    private void doExportUrlsFor1Protocol(
            ProtocolConfig protocolConfig, List<URL> registryURLs) {

        // 4. 构造服务 URL
        URL url = buildServiceUrl(protocolConfig, ...);
        
        // 5. 通过代理工厂将 ref(ServiceImpl) 包装成 Invoker
        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, interfaceClass, url);
        
        // 6. 删除节点后，重新导出暴露 Invoker (同样会触发注册）
        DelegateProviderMetaDataInvoker wrapperInvoker = 
            new DelegateProviderMetaDataInvoker(invoker, this);

        // 7. 通过协议层导出（RegistryProtocol 为入口）
        Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
        exporters.add(exporter);
    }
}
```

**核心步骤**：
1. `loadRegistries()` — 从配置中获取注册中心 URL 列表
2. `buildServiceUrl()` — 构造 Dubbo 服务的完整 URL（包含所有参数）
3. `PROXY_FACTORY.getInvoker()` — 创建代理包装 Invoker
4. `PROTOCOL.export()` — 通过协议层暴露服务

# 4. 代理工厂：JavassistProxyFactory.getInvoker()

```java
public class JavassistProxyFactory extends AbstractProxyFactory {

    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // 如果目标对象已经是 Wrapper 的子类，直接包装
        final Wrapper wrapper = Wrapper.getWrapper(
            proxy.getClass().getName().indexOf('$') < 0 
                ? proxy.getClass() 
                : type
        );

        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                    Class<?>[] parameterTypes, Object[] arguments)
                    throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, 
                    parameterTypes, arguments);
            }
        };
    }
}
```

`Wrapper.getWrapper()` 使用 Javassist 动态生成包装类，避免反射调用，提升性能。生成的类大致如下：

```java
// 运行期生成的 Wrapper 子类伪代码
public class Wrapper0 extends Wrapper {
    public Object invokeMethod(Object instance, String methodName,
            Class[] paramTypes, Object[] args) {
        UserService service = ((UserService) instance);
        if ("getUser".equals(methodName) && paramTypes.length == 1) {
            return service.getUser((Long) args[0]);
        }
        if ("createUser".equals(methodName) && paramTypes.length == 1) {
            return service.createUser((User) args[0]);
        }
        // ...
    }
}
```

# 5. 协议层导出：RegistryProtocol.export()

```java
public class RegistryProtocol implements Protocol {

    @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) {
        // 1. 获取注册中心 URL
        URL registryUrl = getRegistryUrl(originInvoker);

        // 2. 获取服务提供者 URL（去掉注册中心前缀）
        URL providerUrl = getProviderUrl(originInvoker);

        // 3. 本地暴露（启动 Netty Server，绑定端口）
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // 4. 向注册中心注册
        final Registry registry = getRegistry(registryUrl);
        final URL registeredUrl = getUrlToRegistry(providerUrl, registryUrl);
        registry.register(registeredUrl);

        // 5. 注册销毁回调（取消注册、关闭本地服务）
        return new DestroyableExporter<>(exporter, originInvoker, 
            registeredUrl, registry);
    }
}
```

# 6. DubboProtocol.export() —— 本地暴露

```java
public class DubboProtocol extends AbstractProtocol {

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) {
        String key = serviceKey(invoker.getUrl()); // group/interface:version

        // 1. 创建 DubboExporter
        DubboExporter<T> exporter = new DubboExporter<>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        // 2. 启动/获取 Netty Server
        openServer(invoker.getUrl());

        return exporter;
    }

    private void openServer(URL url) {
        String key = url.getAddress(); // host:port

        // Netty Server 以 host:port 为 key 进行缓存，复用连接
        ProtocolServer server = serverMap.get(key);
        if (server == null) {
            synchronized (this) {
                server = serverMap.get(key);
                if (server == null) {
                    serverMap.put(key, createServer(url));
                }
            }
        }
        server.reset(url); // 重置连接参数（超时、线程池等）
    }
}
```

**关键点**：
- 同一 `host:port` 上的所有 Dubbo 服务共享同一个 Netty Server
- 服务以 `group/interface:version` 为 key 存储在 `exporterMap` 中
- 请求到达时通过 key 从 `exporterMap` 中找到对应的 `DubboExporter` 来处理

# 7. DubboExporter 请求处理

```java
public class DubboExporter<T> extends AbstractExporter<T> {
    private final String key;
    private final Map<String, Exporter<?>> exporterMap;

    public DubboExporter(Invoker<T> invoker, String key, 
            Map<String, Exporter<?>> exporterMap) {
        super(invoker);
        this.key = key;
        this.exporterMap = exporterMap;
    }

    @Override
    public void unexport() {
        // 取消暴露时从 map 中移除
        super.unexport();
        exporterMap.remove(key);
    }
}
```

当 Netty Server 收到请求后，通过请求中的 service key 在 `exporterMap` 中查找对应的 `DubboExporter`，再通过其内部的 Invoker 代理调用业务方法。

# 8. 服务导出的线程模型

服务导出过程是**同步**的，在 Spring 容器刷新时在**主线程**中执行：

```
Spring ApplicationContext.refresh()
    └── DubboServiceBean.init()
        └── ServiceConfig.export()
            ├── 本地暴露 (绑定端口、启动 Netty Server)
            └── 注册中心注册 (网络 IO)
```

# 9. 延迟导出

```java
@DubboService(delay = 5000)  // 启动后延迟 5 秒导出
// 或
@DubboService(delay = -1)    // 等 Spring 容器完全初始化后导出
```

延迟导出的实现是通过 `DelayExportProvider` 利用 `ScheduledExecutorService` 延迟执行导出逻辑：

```java
public void export() {
    if (shouldDelay()) {
        DELAY_EXPORT_EXECUTOR.schedule(this::doExport, 
            getDelay(), TimeUnit.MILLISECONDS);
    } else {
        doExport();
    }
}
```

延迟导出适用于：需要预热缓存、依赖外部资源初始化等场景。

# 10. 导出流程时序图

```
ServiceConfig   ProxyFactory   RegistryProtocol   DubboProtocol   Registry   Netty
    │                │               │                  │              │         │
    ├─export()──────→│               │                  │              │         │
    │                │               │                  │              │         │
    │←─Invoker───────┤               │                  │              │         │
    │                │               │                  │              │         │
    ├───────────────→│──────────────→│                  │              │         │
    │                │   export()    │                  │              │         │
    │                │               ├─doLocalExport()─→│              │         │
    │                │               │                  ├─openServer()─│─────→  │
    │                │               │                  │              │  bind()│
    │                │               │  ←─Exporter──────┤              │ ←──────┤
    │                │               │                  │              │         │
    │                │               ├─registry.register()───────────→│         │
    │                │               │                  │              │         │
    │  ←─Exporter────├───────────────┤                  │              │         │
```
