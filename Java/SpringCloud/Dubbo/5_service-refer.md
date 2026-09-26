
# 1. 什么是服务引用

服务引用是 Consumer 端获取远程服务代理对象的过程。引用完成后，Consumer 可以像调用本地方法一样调用远程服务。

```
@DubboReference                                                   远程 Provider
UserService userService;   ←── 注入代理对象 ──→  RPC 调用 ──→  UserServiceImpl
        │
        ▼
ReferenceConfig (配置载体)
        │
        ▼
doRefer() (引用入口)
        │
        ├──→ createProxy()
        │       │
        │       ├──→ 构造注册中心 URL
        │       ├──→ RegistryProtocol.refer(url)
        │       │       │
        │       │       ├── 从注册中心获取 Invoker → RegistryDirectory
        │       │       └── doRefer() → ClusterInvoker
        │       │
        │       └──→ proxyFactory.getProxy(invoker)
```

# 2. @DubboReference 注解

```java
@DubboReference(
    version = "1.0.0",
    group = "dev",
    timeout = 3000,
    retries = 2,
    loadbalance = "random",
    cluster = "failover",
    check = false,               // 启动时不检查 Provider 可用性
    mock = "return null",        // 服务降级：直接返回 null
    stub = "org.example.stub.UserServiceStub",  // 本地存根
    async = false,               // 是否异步调用
    cache = "lru",               // 结果缓存
    validation = "true",
    parameters = {"key1", "value1"}
)
private UserService userService;
```

**关键属性**：

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `check` | true | 启动时是否检查 Provider 存在 |
| `mock` | "" | 服务降级时的 mock 类或返回值 |
| `stub` | "" | 本地存根（在远程调用前后执行本地逻辑） |
| `async` | false | 是否返回异步接口 |
| `cache` | "" | 结果缓存策略（lru/threadlocal/jcache） |
| `url` | "" | 直连 Provider 地址（绕过注册中心） |

# 3. 服务引用入口：ReferenceConfig.get()

```java
public class ReferenceConfig<T> extends AbstractReferenceConfig {

    public synchronized T get() {
        if (ref == null) {
            init();
        }
        return ref;
    }

    private void init() {
        // 1. 检查配置参数
        checkAndUpdateSubConfigs();

        // 2. 组装服务相关消费者 Map
        Map<String, String> map = new HashMap<>();
        appendParameters(map, application);
        appendParameters(map, consumer);
        appendParameters(map, this);

        // 3. 构造注册中心 URL
        List<URL> urls = ConfigValidationUtils.loadRegistries(this);

        // 4. 创建代理对象
        ref = createProxy(urls, map);
    }
}
```

# 4. createProxy() —— 创建服务代理

```java
private T createProxy(List<URL> registryURLs, Map<String, String> map) {
    // 1. 如果用户指定了 url（直连模式），直接构造 Invoker
    if (StringUtils.isNotEmpty(url)) {
        // 直连模式：绕过注册中心
        Invoker<?> invoker = refProtocol.refer(interfaceClass, url);
        return (T) PROXY_FACTORY.getProxy(invoker);
    }

    // 2. 否则走注册中心模式
    List<Invoker<?>> invokers = new ArrayList<>();
    Invoker<?> invoker = null;

    for (URL registryURL : registryURLs) {
        // 每个注册中心生成一个 ClusterInvoker
        invoker = REF_PROTOCOL.refer(interfaceClass, registryURL);
        invokers.add(invoker);
    }

    if (invokers.size() == 1) {
        invoker = invokers.get(0);
    } else {
        // 多注册中心：通过 StaticDirectory 聚合
        invoker = CLUSTER.join(new StaticDirectory(invokers));
    }

    // 3. 通过代理工厂创建 JDK/Javassist 动态代理
    return (T) PROXY_FACTORY.getProxy(invoker);
}
```

**核心逻辑**：
- 直连模式：`url` 直接指定 Provider 地址，绕过注册中心
- 注册中心模式：由 RegistryProtocol 处理，集成 Directory + Router + LoadBalance
- 多注册中心：用 StaticDirectory 聚合多个 Invoker

# 5. RegistryProtocol.refer() —— 注册中心引用

```java
public class RegistryProtocol implements Protocol {

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) {
        // 1. 获取注册中心 URL
        URL registryUrl = getRegistryUrl(url);

        // 2. 创建 RegistryDirectory
        //    负责从注册中心订阅 Provider 列表并维护 Invoker 缓存
        Registry registry = getRegistry(registryUrl);
        RegistryDirectory<T> directory = new RegistryDirectory<>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);

        // 3. 订阅 Provider 列表
        //    收到变更通知时，Directory 会刷新内部 Invoker 列表
        directory.subscribe(toSubscribeUrl(url));

        // 4. 通过 cluster.join(directory) 构建 ClusterInvoker
        //    将 Directory、Router、LoadBalance 组合在一起
        return doRefer(CLUSTER, registry, type, url);
    }
}
```

# 6. RegistryDirectory —— 动态 Invoker 目录

```java
public class RegistryDirectory<T> extends AbstractDirectory<T>
        implements NotifyListener {

    // URL → Invoker 映射，根据注册中心推送的 URL 动态维护
    private volatile Map<String, Invoker<T>> urlInvokerMap = new HashMap<>();

    @Override
    public synchronized void notify(List<URL> urls) {
        Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(urls);
        // CAS 更新缓存
        this.urlInvokerMap = newUrlInvokerMap;
        // 销毁被移除的 Invoker
        destroyUnusedInvokers(oldMap, newUrlInvokerMap);
    }

    @Override
    public List<Invoker<T>> doList(Invocation invocation) {
        // 根据方法级配置筛选（如方法级超时覆盖）
        return new ArrayList<>(urlInvokerMap.values());
    }
}
```

**关键流程**：
1. 首次 refer 时从注册中心拉取全量 Provider URL 列表
2. 注册 RegistryDirectory 为 Listener，监听节点变更
3. Provider 上下线时，`notify()` 被回调，更新 `urlInvokerMap`
4. `doList()` 提供给 Router/LoadBalance 获取当前可用的 Invoker 列表

# 7. Cluster 层：ClusterInvoker

```java
// Cluster.join(directory) 返回 ClusterInvoker
public interface Cluster {
    <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) 
        throws RpcException;
}

// 以 FailoverCluster 为例
public class FailoverCluster implements Cluster {
    @Override
    public <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) {
        return new FailoverClusterInvoker<>(directory);
    }
}
```

ClusterInvoker 的调用链（以 FailoverClusterInvoker 为例）：

```java
public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {

    @Override
    public Result doInvoke(Invocation invocation, 
            List<Invoker<T>> invokers, LoadBalance loadbalance) {
        
        List<Invoker<T>> copyInvokers = invokers;
        int retries = getUrl().getMethodParameter(invocation.getMethodName(), 
            "retries", 2) + 1; // 含第一次调用

        RpcException lastException = null;
        for (int i = 0; i < retries; i++) {
            // 重新获取列表（可能发生变更）
            checkInvokers(copyInvokers, invocation);
            // 通过 Router + LoadBalance 选择 Invoker
            Invoker<T> invoker = this.select(loadbalance, invocation, 
                copyInvokers, invoked);
            try {
                Result result = invoker.invoke(invocation);
                return result;
            } catch (RpcException e) {
                lastException = e;
                // 收集错误信息，继续重试
            }
        }
        throw lastException;
    }
}
```

# 8. 代理层：JavassistProxyFactory.getProxy()

```java
public class JavassistProxyFactory extends AbstractProxyFactory {

    @Override
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        // 使用 Javassist 生成代理类
        return (T) Proxy.getProxy(interfaces).newInstance(
            new InvokerInvocationHandler(invoker)
        );
    }
}
```

`InvokerInvocationHandler` 将代理方法调用转发到 Invoker：

```java
public class InvokerInvocationHandler implements InvocationHandler {
    private final Invoker<?> invoker;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) 
            throws Throwable {
        // Object 类的方法（toString/hashCode/equals）直接调用
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
        }

        // 构造 RPC Invocation
        RpcInvocation rpcInvocation = new RpcInvocation(
            method, invoker.getInterface().getName(), args);

        // 通过 Invoker 调用远程服务
        return invoker.invoke(rpcInvocation).recreate();
    }
}
```

# 9. check 属性与启动依赖

```java
@DubboReference(check = false)
private UserService userService;
```

`check=true`（默认）时，Consumer 启动会检查是否有可用的 Provider，不存在则启动失败：

```java
public class RegistryDirectory<T> extends AbstractDirectory<T> {

    private void checkInvokers(List<Invoker<T>> invokers, Invocation invocation) {
        if (CollectionUtils.isEmpty(invokers)) {
            throw new RpcException("Failed to invoke ... No provider available");
        }
    }
}
```

# 10. 直连模式

绕过注册中心，直接指定 Provider 地址：

```java
@DubboReference(url = "dubbo://192.168.1.100:20880")
private UserService userService;
```

或通过配置：

```yaml
dubbo:
  consumer:
    url: dubbo://192.168.1.100:20880
```

用于调试、测试或静态环境。实现上走 `直连 Invoker` → `DubboProtocol.refer()` → Netty Client 直接发起调用。

# 11. 引用流程时序图

```
ReferenceConfig  RegistryProtocol  RegistryDirectory  Registry  DubboProtocol  ProxyFactory
    │                 │                  │                 │           │               │
    ├─refer()────────→│                  │                 │           │               │
    │                 │                  │                 │           │               │
    │                 ├─new()───────────→│                 │           │               │
    │                 │  RegistryDirect. │                 │           │               │
    │                 │                  │                 │           │               │
    │                 │                  ├─subscribe()────→│           │               │
    │                 │                  │                 │           │               │
    │                 │                  │←─notify(URLs)───┤           │               │
    │                 │                  │  (异步回调)       │           │               │
    │                 │                  │                 │           │               │
    │                 │                  ├─toInvokers(URLs)│           │               │
    │                 │                  │       │         │           │               │
    │                 │                  │       └─────────│─────refer─→│               │
    │                 │                  │                 │           │               │
    │                 │ ←─ClusterInvoker─┤                 │           │               │
    │                 │                  │                 │           │               │
    │ ←─ClusterInvoker┤                  │                 │           │               │
    │                 │                  │                 │           │               │
    ├───────────────────────────────────────────────────────────────────→               │
    │                                                           getProxy(invoker)      │
    │ ←─Proxy Object──────────────────────────────────────────────────────────────────┤
```
