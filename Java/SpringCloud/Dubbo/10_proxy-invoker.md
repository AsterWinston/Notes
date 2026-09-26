
# 1. Invoker —— 核心领域模型

Invoker 是 Dubbo 中最核心的抽象，代表一个**可执行的对象**。无论是本地调用还是远程调用，对外都统一为 Invoker。

```java
public interface Invoker<T> extends Node {
    Class<T> getInterface();
    Result invoke(Invocation invocation) throws RpcException;
    URL getUrl();
    boolean isAvailable();
    void destroy();
}
```

# 2. Invoker 的分类与层级

```
                             ┌─────────────────┐
                             │  业务接口调用      │
                             └────────┬────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │     Proxy (动态代理)        │
                        │  InvokerInvocationHandler  │
                        └─────────────┬─────────────┘
                                      │ invoke()
                        ┌─────────────┴─────────────┐
                        │    ClusterInvoker          │
                        │  (集群容错 + 路由 + 负载均衡) │
                        └─────────────┬─────────────┘
                                      │ select()
                        ┌─────────────┴─────────────┐
                        │   RegistryDirectory        │
                        │     (Invoker 列表)          │
                        └─────────────┬─────────────┘
                                      │ list()
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
    DubboInvoker              DubboInvoker            DubboInvoker
    (Provider 1)              (Provider 2)            (Provider 3)
              │                       │                       │
    ┌─────────┴─────────┐   ┌─────────┴─────────┐   ┌─────────┴───...
    │  ExchangeClient   │   │  ExchangeClient   │   │
    │  (Netty Client)   │   │  (Netty Client)   │   │
    └───────────────────┘   └───────────────────┘   └─────────────
```

**Provider 端**：

```
Request → DubboExporter → AbstractProxyInvoker
                               │
                               └── invoke() → 反射调用 ServiceImpl
```

# 3. 代理工厂（ProxyFactory）

```java
@SPI("javassist")
public interface ProxyFactory {
    // Provider 端：将 ServiceImpl 包装成 Invoker
    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;

    // Consumer 端：将 Invoker 生成代理对象
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces);

    // 获取无 Invoker 的代理（用于本地存根/回调）
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic);
}
```

内置实现：

| 实现类 | 方式 | 特点 |
|--------|------|------|
| `JavassistProxyFactory` | Javassist 字节码生成 | 默认实现，无反射开销 |
| `JdkProxyFactory` | JDK 动态代理 | 标准 Java API |
| `StubProxyFactoryWrapper` | 包装器 | 自动处理本地存根 Stub |
| `GenericProxyFactory` | 泛化调用 | 泛化调用专用 |

# 4. JavassistProxyFactory —— 默认代理

## 4.1 Provider 端：getInvoker()

```java
public class JavassistProxyFactory extends AbstractProxyFactory {

    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // 获取 Wrapper（Javassist 生成）
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
                // 通过 Wrapper 调用实际方法（无反射！）
                return wrapper.invokeMethod(proxy, methodName,
                    parameterTypes, arguments);
            }
        };
    }
}
```

## 4.2 Wrapper —— 字节码生成

```java
public abstract class Wrapper {
    // Wrapper 缓存：Map<ClassName, Wrapper>
    private static final Map<Class<?>, Wrapper> WRAPPER_MAP = 
        new ConcurrentHashMap<>();

    public static Wrapper getWrapper(Class<?> c) {
        return WRAPPER_MAP.computeIfAbsent(c, Wrapper::makeWrapper);
    }

    private static Wrapper makeWrapper(Class<?> c) {
        // 使用 Javassist 动态生成 Wrapper 子类
        // 生成的类大致如下：
        // 
        // public class Wrapper0 extends Wrapper {
        //     public Object invokeMethod(Object instance, String mn,
        //             Class[] pt, Object[] arg) {
        //         UserService svc = ((UserService)instance);
        //         if ("getUser".equals(mn) && pt.length == 1) {
        //             return svc.getUser(((Long)arg[0]));
        //         }
        //         if ("createUser".equals(mn) && pt.length == 1) {
        //             return svc.createUser(((User)arg[0]));
        //         }
        //         throw new NoSuchMethodException(...);
        //     }
        // }

        ClassGenerator cc = ClassGenerator.newInstance(cl);
        // 遍历所有 public 方法，生成 if-else 分支
        for (Method m : ms) {
            cc.addMethod(/* 生成方法体 */);
        }
        return (Wrapper) cc.toClass().newInstance();
    }

    public abstract Object invokeMethod(Object instance, String methodName,
        Class<?>[] parameterTypes, Object[] arguments) throws Throwable;
}
```

**优点**：避免反射，直接方法调用，性能接近原生调用。

## 4.3 Consumer 端：getProxy()

```java
@Override
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    // 通过 Javassist 生成代理类
    return (T) Proxy.getProxy(interfaces).newInstance(
        new InvokerInvocationHandler(invoker)
    );
}
```

# 5. InvokerInvocationHandler —— 代理拦截

```java
public class InvokerInvocationHandler implements InvocationHandler {
    private final Invoker<?> invoker;
    private final String serviceKey;
    private final URL consumerUrl;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        // 1. Object 的方法（toString/hashCode/equals）不走 RPC
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }

        // 2. 构造 RpcInvocation
        RpcInvocation rpcInvocation = new RpcInvocation(
            method, invoker.getInterface().getName(), args
        );
        
        // 3. 透传消费端的隐式参数
        Map<String, String> attachments = RpcContext.get().getAttachments();
        if (!attachments.isEmpty()) {
            rpcInvocation.addAttachments(attachments);
        }

        // 4. 调用 Invoker（进入 Cluster → LoadBalance → DubboInvoker 链路）
        Result result = invoker.invoke(rpcInvocation);

        // 5. 处理结果（同步/异步）
        if (method.getReturnType() == CompletableFuture.class) {
            // 异步返回
            return ((AsyncRpcResult) result).getResponseFuture();
        }
        
        return result.recreate();  // RpcResult → 业务返回值
    }
}
```

# 6. RpcInvocation —— 调用上下文

```java
public class RpcInvocation implements Invocation, Cloneable {
    private String targetServiceUniqueName;  // group/interface:version
    private String methodName;
    private Class<?>[] parameterTypes;
    private Object[] arguments;
    private Map<String, String> attachments; // 隐式参数
    private Invoker<?> invoker;
    
    // 兼容 Dubbo 2.x 接口级
    private String serviceName;              // org.example.UserService
    private transient Class<?>[] returnTypes;
    private transient Object result;
}
```

# 7. Invoker 树形结构

```
Invoker (根)
├── MockClusterInvoker (如果配置了 mock)
│   └── AbstractClusterInvoker
│       ├── Directory (RegistryDirectory)
│       │   ├── Router Chain (条件路由 → 标签路由 → 脚本路由)
│       │   └── Invoker List
│       │       ├── DubboInvoker(provider1) ← Filter Chain
│       │       ├── DubboInvoker(provider2) ← Filter Chain
│       │       └── DubboInvoker(provider3) ← Filter Chain
│       └── LoadBalance (选择策略)
└── Filter Chain (Consumer 端 Filter)
```

# 8. Invoker 的数据流动

```
Consumer 调用 userService.getUser(1L)
    │
    ▼
InvokerInvocationHandler.invoke()
    │ 构造 RpcInvocation
    ▼
MockClusterInvoker.invoke() [如果有 mock]
    │
    ▼
FailoverClusterInvoker.invoke() [以 failover 为例]
    │
    ├── 1. Directory.list(invocation) → 获取 Invoker 列表
    ├── 2. Router.route(invokers, url, invocation) → 过滤
    ├── 3. LoadBalance.select(invokers, url, invocation) → 选择
    │
    ▼
Filter Chain (Consumer Filter: MonitorFilter → ...)
    │
    ▼
DubboInvoker.invoke()
    │
    ├── ExchangeClient.request(invocation)
    │   └── NettyClient.send(request)
    │
    ▼
─── TCP / Network ───

Provider 端:
    │
    ▼
NettyServer.received(request)
    │
    ▼
DubboProtocol.requestHandler.reply()
    │
    ▼
exporterMap.get(serviceKey).getInvoker().invoke(invocation)
    │
    ▼
Filter Chain (Provider Filter: ExceptionFilter → ...)
    │
    ▼
AbstractProxyInvoker.invoke()
    │
    ▼
Wrapper.invokeMethod(serviceImpl, methodName, paramTypes, args)
    │
    ▼
UserServiceImpl.getUser(1L) → 返回 User
```

# 9. JdkProxyFactory —— JDK 动态代理

```java
public class JdkProxyFactory extends AbstractProxyFactory {

    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // 使用 JDK 反射调用
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                    Class<?>[] parameterTypes, Object[] arguments)
                    throws Throwable {
                Method method = proxy.getClass().getMethod(methodName, 
                    parameterTypes);
                return method.invoke(proxy, arguments); // 反射调用
            }
        };
    }

    @Override
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.newProxyInstance(
            Thread.currentThread().getContextClassLoader(),
            interfaces,
            new InvokerInvocationHandler(invoker)
        );
    }
}
```

切换代理方式：

```yaml
dubbo:
  provider:
    proxy: jdk        # 或 javassist (默认)
```

# 10. 代理性能对比

| 代理方式 | 调用机制 | 性能 | 适用场景 |
|---------|---------|------|---------|
| Javassist | 字节码 if-else | 极快 | 默认推荐 |
| JDK Proxy | 反射 invoke | 较慢 | 需标准 Java API |
| CGLIB | 字节码继承 | 较快 | 已废弃 |

**Javassist 比 JDK 代理快 3-5 倍**，因为避免了反射的 Method.invoke() 开销。
