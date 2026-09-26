
# 1. 集群层概述

集群层（Cluster Layer）是 Dubbo 最核心的一层，负责将多个服务提供者组织为一个逻辑集群，并提供：

- **服务目录（Directory）**：维护 Provider 列表
- **路由（Router）**：按规则过滤 Provider 列表
- **负载均衡（LoadBalance）**：从过滤后的列表中选择一个 Provider
- **集群容错（Cluster）**：调用失败后的重试/容错策略

```
Consumer 调用 userService.getUser(1L)
    │
    ▼
MockClusterWrapper (如果配置了 mock)
    │
    ▼
ClusterInvoker（如 FailoverClusterInvoker）
    │
    ├── 1. Directory.list(invocation)      // 获取 Invoker 列表
    │       └── RegistryDirectory           // 从注册中心维护
    │
    ├── 2. Router.route(invokers, url, invocation)  // 路由过滤
    │       ├── ConditionRouter             // 条件路由
    │       ├── TagRouter                   // 标签路由
    │       ├── ServiceRouter               // 服务路由
    │       └── AppRouter                   // 应用路由
    │
    ├── 3. LoadBalance.select(invokers, url, invocation) // 负载均衡
    │       ├── RandomLoadBalance
    │       ├── RoundRobinLoadBalance
    │       ├── LeastActiveLoadBalance
    │       ├── ConsistentHashLoadBalance
    │       └── ShortestResponseLoadBalance
    │
    └── 4. 执行调用，处理失败重试（容错策略）
```

# 2. Directory —— 服务目录

## 2.1 接口定义

```java
public interface Directory<T> extends Node {
    Class<T> getInterface();
    List<Invoker<T>> list(Invocation invocation) throws RpcException;
    @Override
    void destroy();
}
```

## 2.2 实现体系

```
Directory (接口)
├── AbstractDirectory
│   ├── StaticDirectory         (静态目录：直连模式下固定 Invoker 列表)
│   └── RegistryDirectory       (动态目录：订阅注册中心，自动更新)
└── ServiceDiscoveryRegistryDirectory
```

## 2.3 RegistryDirectory —— 核心实现

```java
public class RegistryDirectory<T> extends AbstractDirectory<T> 
        implements NotifyListener {
    
    // key = providerUrl.toFullString(), value = Invoker
    private volatile Map<String, Invoker<T>> urlInvokerMap = new HashMap<>();
    
    // 注册中心监听器列表
    private List<Configurator> configurators = new ArrayList<>();
    private volatile List<Router> routers = new ArrayList<>();

    @Override
    public synchronized void notify(List<URL> urls) {
        // 分类：configurators、routers、providers
        List<URL> invokerUrls = new ArrayList<>();
        List<URL> routerUrls = new ArrayList<>();
        List<URL> configuratorUrls = new ArrayList<>();
        
        for (URL url : urls) {
            String category = url.getParameter("category");
            if ("providers".equals(category)) {
                invokerUrls.add(url);
            } else if ("routers".equals(category)) {
                routerUrls.add(url);
            } else if ("configurators".equals(category)) {
                configuratorUrls.add(url);
            }
        }
        
        // 更新路由器（路由规则变更时生效）
        if (!routerUrls.isEmpty()) {
            this.routers = toRouters(routerUrls);
        }
        
        // 更新配置规则
        if (!configuratorUrls.isEmpty()) {
            this.configurators = toConfigurators(configuratorUrls);
        }
        
        // 更新 Invoker 列表
        refreshInvoker(invokerUrls);
    }

    private void refreshInvoker(List<URL> invokerUrls) {
        // 将 URL 转换为 Invoker（通过 DubboProtocol.refer）
        Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);
        
        // CAS 替换
        Map<String, Invoker<T>> oldMap = this.urlInvokerMap;
        this.urlInvokerMap = newUrlInvokerMap;
        
        // 销毁被下线的 Invoker
        destroyUnusedInvokers(oldMap, newUrlInvokerMap);
    }

    @Override
    public List<Invoker<T>> doList(Invocation invocation) {
        // 应用 Configurator（动态配置覆盖 URL 参数）
        List<Invoker<T>> invokers = new ArrayList<>(urlInvokerMap.values());
        if (!configurators.isEmpty()) {
            for (Configurator configurator : configurators) {
                invokers = configurator.configure(invokers, invocation);
            }
        }
        return invokers;
    }
}
```

# 3. Cluster —— 集群容错策略

## 3.1 接口定义

```java
@SPI(FailoverCluster.NAME)  // 默认 failover
public interface Cluster {
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) 
        throws RpcException;
    
    static Invoker<T> getInvoker(Directory<T> directory, String cluster) {
        return ExtensionLoader.getExtensionLoader(Cluster.class)
            .getExtension(cluster).join(directory, true);
    }
}
```

## 3.2 内置集群策略

| 策略 | 类名 | 说明 |
|------|------|------|
| **failover** | `FailoverCluster` | 失败自动切换，重试其他服务器（默认） |
| **failfast** | `FailfastCluster` | 快速失败，只发起一次调用 |
| **failsafe** | `FailsafeCluster` | 失败安全，异常时忽略（仅记录日志） |
| **failback** | `FailbackCluster` | 失败自动恢复，后台定时重试 |
| **forking** | `ForkingCluster` | 并行调用多个服务器，取第一个成功返回 |
| **broadcast** | `BroadcastCluster` | 广播调用所有提供者，任一失败即失败 |
| **available** | `AvailableCluster` | 遍历可用的 Invoker，直接调用第一个 |
| **mergeable** | `MergeableCluster` | 分组聚合，合并多个提供者返回结果 |

## 3.3 FailoverCluster —— 失败重试

```java
public class FailoverCluster implements Cluster {
    public static final String NAME = "failover";

    @Override
    public <T> Invoker<T> join(Directory<T> directory, 
            boolean buildFilterChain) {
        return new FailoverClusterInvoker<>(directory);
    }
}

public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {

    @Override
    public Result doInvoke(Invocation invocation, 
            List<Invoker<T>> invokers, LoadBalance loadbalance) {
        
        List<Invoker<T>> copyInvokers = invokers;
        int retries = getUrl().getMethodParameter(
            invocation.getMethodName(), "retries", 2) + 1;
        
        if (copyInvokers.size() == 0) {
            throw new RpcException("No provider available");
        }

        RpcException lastException = null;
        List<Invoker<T>> invoked = new ArrayList<>(copyInvokers.size());
        Set<String> providers = new HashSet<>();
        
        for (int i = 0; i < retries; i++) {
            // 重试前重新获取 Invoker 列表（可能已变更）
            if (i > 0) {
                checkInvokers(copyInvokers, invocation);
                copyInvokers = directory.list(invocation);
                checkInvokers(copyInvokers, invocation);
            }
            
            // 通过 Router + LoadBalance 选择
            Invoker<T> invoker = select(loadbalance, invocation, 
                copyInvokers, invoked);
            invoked.add(invoker);
            providers.add(invoker.getUrl().getAddress());
            
            try {
                Result result = invoker.invoke(invocation);
                
                // 重试成功后打印警告日志（说明之前失败了）
                if (i > 0) {
                    logger.warn("Retry success after {} times. " + 
                        "Tried: {}, now: {}", i, providers, 
                        invoker.getUrl().getAddress());
                }
                return result;
                
            } catch (RpcException e) {
                lastException = e;
                // 业务异常不重试（直接抛出）
                if (e.isBiz()) {
                    throw e;
                }
                // continue 重试
            } catch (Throwable e) {
                lastException = new RpcException(e);
            }
        }
        
        // 所有重试都失败
        throw new RpcException(
            "Failed to invoke after " + retries + " retries. " +
            "Tried providers: " + providers, lastException);
    }
}
```

## 3.4 FailfastCluster —— 快速失败

```java
public class FailfastClusterInvoker<T> extends AbstractClusterInvoker<T> {

    @Override
    public Result doInvoke(Invocation invocation, 
            List<Invoker<T>> invokers, LoadBalance loadbalance) {
        
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        
        try {
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            if (e instanceof RpcException && ((RpcException) e).isBiz()) {
                throw (RpcException) e;
            }
            throw new RpcException("Failfast invoke providers " + 
                invoker.getUrl() + " error", e);
        }
    }
}
```

## 3.5 FailsafeCluster —— 失败安全

```java
public class FailsafeClusterInvoker<T> extends AbstractClusterInvoker<T> {

    @Override
    public Result doInvoke(Invocation invocation, 
            List<Invoker<T>> invokers, LoadBalance loadbalance) {
        
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failsafe ignore exception: " + e.getMessage(), e);
            return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation);
        }
    }
}
```

## 3.6 ForkingCluster —— 并行调用

```java
public class ForkingClusterInvoker<T> extends AbstractClusterInvoker<T> {

    @Override
    public Result doInvoke(Invocation invocation, 
            List<Invoker<T>> invokers, LoadBalance loadbalance) {
        
        checkInvokers(invokers, invocation);
        
        // forks 参数：并行调用数（默认 2）
        int forks = getUrl().getParameter("forks", 2);
        int timeout = getUrl().getParameter("timeout", 1000);
        
        // 并行调用 forks 个 Invoker
        final List<Invoker<T>> selected = new ArrayList<>();
        for (int i = 0; i < Math.min(forks, invokers.size()); i++) {
            Invoker<T> invoker = select(loadbalance, invocation, 
                invokers, selected);
            selected.add(invoker);
        }

        // 线程池并行执行
        ExecutorService executor = Executors.newFixedThreadPool(forks);
        final BlockingQueue<Object> ref = new LinkedBlockingQueue<>();
        
        for (final Invoker<T> invoker : selected) {
            executor.execute(() -> {
                try {
                    Result result = invoker.invoke(invocation);
                    ref.offer(result);
                } catch (Throwable e) {
                    ref.offer(e);
                }
            });
        }
        
        // 取第一个成功返回的结果
        try {
            Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
            if (ret instanceof Throwable) {
                throw (Throwable) ret;
            }
            return (Result) ret;
        } catch (InterruptedException e) {
            throw new RpcException("Forking invoke timeout");
        } finally {
            executor.shutdownNow();
        }
    }
}
```

# 4. AbstractClusterInvoker —— 公共基类

```java
public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    protected final Directory<T> directory;

    // 公共的 select 方法：调用 Router + LoadBalance
    protected Invoker<T> select(LoadBalance loadbalance, 
            Invocation invocation,
            List<Invoker<T>> invokers, 
            List<Invoker<T>> selected) {
        
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        
        // 1. 从 Directory 获取 Router Chain
        List<Router> routers = directory.getRouters();
        
        // 2. Router 过滤
        if (!routers.isEmpty()) {
            for (Router router : routers) {
                invokers = router.route(invokers, 
                    getConsumerUrl(), invocation);
                if (invokers.isEmpty()) {
                    break;
                }
            }
        }
        
        // 3. LoadBalance 选择
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
        
        return loadbalance.select(invokers, getUrl(), invocation);
    }
}
```

# 5. 配置使用

```yaml
dubbo:
  consumer:
    cluster: failover       # 集群容错策略
    retries: 2               # Failover 重试次数（不含首次）
```

```java
@DubboReference(cluster = "failfast")
private UserService userService;

@DubboReference(cluster = "failover", retries = 3)
private OrderService orderService;

@DubboReference(cluster = "forking", parameters = {"forks", "3"})
private PaymentService paymentService;
```

# 6. 集群策略选择指南

| 场景 | 推荐策略 | 原因 |
|------|---------|------|
| 幂等读操作 | failover (默认) | 失败重试无副作用 |
| 非幂等写操作 | failfast | 避免重试导致重复写入 |
| 不重要辅助逻辑（日志等） | failsafe | 失败不影响主流程 |
| 最终一致性操作 | failback | 后台定时重试 |
| 必须成功但时效要求高 | forking | 并行调用取最快 |
| 通知所有实例 | broadcast | 缓存更新、配置下发 |
| 需要结果合并 | mergeable | 分组聚合 |

# 7. MockClusterWrapper —— 服务降级

```java
public class MockClusterWrapper implements Cluster {

    @Override
    public <T> Invoker<T> join(Directory<T> directory, 
            boolean buildFilterChain) {
        return new MockClusterInvoker<>(directory, 
            this.cluster.join(directory, buildFilterChain));
    }
}

public class MockClusterInvoker<T> implements ClusterInvoker<T> {
    private final Invoker<T> invoker; // 真实 ClusterInvoker
    
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result result;
        try {
            result = invoker.invoke(invocation);
        } catch (RpcException e) {
            if (e.isForbidded()) {
                // 配置了 mock 降级
                String mockValue = getUrl().getParameter("mock");
                return doMockInvoke(invocation, mockValue);
            }
            throw e;
        }
        return result;
    }
}
```

配置示例：

```java
// 服务降级：失败时返回 null
@DubboReference(mock = "return null")
private UserService userService;

// 服务降级：失败时返回 Mock 实现类
@DubboReference(mock = "com.example.mock.UserServiceMock")
private UserService userService;

// 强制降级：不调用远程，直接 mock
// mock=force:return+null
```
