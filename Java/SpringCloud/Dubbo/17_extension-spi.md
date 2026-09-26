
# 1. SPI 概述

SPI（Service Provider Interface）是 Dubbo 微内核 + 全插件架构的基石。几乎所有功能模块都通过 SPI 实现可插拔，包括 Protocol、Filter、LoadBalance、Registry 等。

```
┌─────────────────────────────────────┐
│         Dubbo Core (Microkernel)     │
│  ExtensionLoader (SPI 引擎)           │
│                                     │
│  ┌────────┐ ┌────────┐ ┌────────┐  │
│  │Protocol│ │ Filter │ │Registry│  │
│  │  SPI   │ │  SPI   │ │  SPI   │  │
│  └────────┘ └────────┘ └────────┘  │
│  ┌────────┐ ┌────────┐ ┌────────┐  │
│  │LoadBal │ │ Router │ │Serialize│  │
│  │  SPI   │ │  SPI   │ │   SPI  │  │
│  └────────┘ └────────┘ └────────┘  │
└─────────────────────────────────────┘
```

# 2. Java SPI vs Dubbo SPI

| 特性 | Java SPI | Dubbo SPI |
|------|---------|-----------|
| 配置文件位置 | `META-INF/services/` | `META-INF/dubbo/` |
| 全量加载 | 是（全部实现类都加载） | 否（按需加载） |
| 默认值 | 不支持 | `@SPI("default")` 指定 |
| IoC 注入 | 不支持 | 支持扩展点注入 |
| AOP 包装 | 不支持 | 支持 Wrapper 类自动包装 |
| 自适应扩展 | 不支持 | `@Adaptive` 动态生成代理 |
| 条件激活 | 不支持 | `@Activate` 按条件激活 |
| 激活排序 | 不支持 | `@Activate(order = N)` |

# 3. Dubbo SPI 三大注解

## 3.1 @SPI —— 标记扩展点接口

```java
@SPI("hessian2")  // 指定默认实现名称
public interface Serialization {
    byte getContentTypeId();
    String getContentType();
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
}
```

## 3.2 @Adaptive —— 自适应扩展

```java
// 标记在类上：手动实现自适应逻辑
@Adaptive
public class AdaptiveLoadBalance implements LoadBalance {
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers,
            URL url, Invocation invocation) {
        // 从 URL 中获取 loadbalance 参数，动态选择实现
        String lbKey = url.getParameter("loadbalance", "random");
        LoadBalance lb = ExtensionLoader.getExtensionLoader(LoadBalance.class)
            .getExtension(lbKey);
        return lb.select(invokers, url, invocation);
    }
}

// 标记在方法上：Dubbo 自动生成 Adaptive 类
@SPI("random")
public interface LoadBalance {
    @Adaptive("loadbalance")  // 从 URL 的 loadbalance 参数获取扩展名
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url,
        Invocation invocation) throws RpcException;
}
```

自动生成的 Adaptive 类代码（伪代码）：

```java
// 编译期自动生成 LoadBalance$Adaptive
public class LoadBalance$Adaptive implements LoadBalance {
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers,
            URL url, Invocation invocation) {
        // 从 URL 参数中获取扩展名
        String extName = url.getParameter("loadbalance", "random");
        // 通过 ExtensionLoader 获取对应实现
        LoadBalance extension = ExtensionLoader
            .getExtensionLoader(LoadBalance.class)
            .getExtension(extName);
        // 委托给具体实现
        return extension.select(invokers, url, invocation);
    }
}
```

## 3.3 @Activate —— 条件激活

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface Activate {
    String[] group() default {};    // provider / consumer
    String[] value() default {};    // URL 参数条件 (如: {"validation", "true"})
    int order() default 0;          // 排序值，越小越靠前
    String[] before() default {};    // 排在指定扩展之前
    String[] after() default {};     // 排在指定扩展之后
}
```

**使用示例**：

```java
@Activate(group = {PROVIDER, CONSUMER}, order = -10000)
public class MonitorFilter implements Filter {
    // 在 Provider 和 Consumer 端都激活，最先执行
}

@Activate(group = {PROVIDER}, value = {"validation", "true"})
public class ValidationFilter implements Filter {
    // 只在 Provider 端激活，且 URL 中 validation=true 才激活
}

@Activate(group = {PROVIDER}, order = -1)
public class ExceptionFilter implements Filter {
    // Provider 端激活，倒数第二个执行
}
```

# 4. ExtensionLoader —— SPI 引擎

ExtensionLoader 是 Dubbo SPI 的核心加载器，负责扩展点的发现、加载、IoC、AOP。

```java
public class ExtensionLoader<T> {
    // 扩展点缓存
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> 
        EXTENSION_LOADERS = new ConcurrentHashMap<>();
    private static final ConcurrentMap<Class<?>, Object> 
        EXTENSION_INSTANCES = new ConcurrentHashMap<>();
    
    // 当前扩展点类型
    private final Class<?> type;
    
    // 缓存
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();
    private final ConcurrentHashMap<String, Holder<Object>> cachedInstances = 
        new ConcurrentHashMap<>();
    private final Holder<Object> cachedAdaptiveInstance = new Holder<>();
    private Set<Class<?>> cachedWrapperClasses;
    
    private String cachedDefaultName;
}
```

## 4.1 获取扩展

```java
// 获取默认扩展（@SPI 注解指定的默认值）
LoadBalance lb = ExtensionLoader.getExtensionLoader(LoadBalance.class)
    .getDefaultExtension();

// 获取指定扩展
LoadBalance randomLB = ExtensionLoader.getExtensionLoader(LoadBalance.class)
    .getExtension("random");

// 获取自适应扩展
LoadBalance adaptiveLB = ExtensionLoader.getExtensionLoader(LoadBalance.class)
    .getAdaptiveExtension();
```

## 4.2 getExtension() 源码

```java
public T getExtension(String name) {
    Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}

private T createExtension(String name) {
    // 1. 获取扩展类
    Class<?> clazz = getExtensionClasses().get(name);
    
    // 2. 从缓存获取实例
    T instance = (T) EXTENSION_INSTANCES.get(clazz);
    if (instance == null) {
        instance = (T) clazz.newInstance();
        EXTENSION_INSTANCES.putIfAbsent(clazz, instance);
    }
    
    // 3. IoC 注入（注入依赖的其他扩展点）
    injectExtension(instance);
    
    // 4. AOP 包装（Wrapper 类包装原始实例）
    Set<Class<?>> wrapperClasses = cachedWrapperClasses;
    if (CollectionUtils.isNotEmpty(wrapperClasses)) {
        for (Class<?> wrapperClass : wrapperClasses) {
            instance = injectExtension(
                wrapperClass.getConstructor(type).newInstance(instance));
        }
    }
    
    return instance;
}
```

## 4.3 加载扩展类

```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

private Map<String, Class<?>> loadExtensionClasses() {
    Map<String, Class<?>> extensionClasses = new HashMap<>();
    
    // 1. 加载 @SPI 注解的默认值
    cacheDefaultExtensionName();
    
    // 2. 扫描三个目录的配置文件
    //    META-INF/dubbo/internal/     (内置扩展)
    //    META-INF/dubbo/              (用户扩展)
    //    META-INF/services/           (兼容 Java SPI)
    for (String dir : strategies) {
        String fileName = dir + type.getName();
        loadDirectory(extensionClasses, fileName);
    }
    
    return extensionClasses;
}

private void loadDirectory(Map<String, Class<?>> extensionClasses,
        String dir) {
    String fileName = dir + type.getName();
    Enumeration<URL> urls;
    ClassLoader classLoader = findClassLoader();
    urls = classLoader.getResources(fileName);
    
    while (urls.hasMoreElements()) {
        URL resourceURL = urls.nextElement();
        // 解析 key=value 配置
        loadResource(extensionClasses, classLoader, resourceURL);
    }
}
```

# 5. IoC 注入

```java
private T injectExtension(T instance) {
    for (Method method : instance.getClass().getMethods()) {
        // 查找 setter 方法
        if (isSetter(method)) {
            Class<?> pt = method.getParameterTypes()[0];
            
            // 获取属性名（setXxx → xxx）
            String property = method.getName().substring(3);
            property = property.substring(0, 1).toLowerCase() 
                + property.substring(1);
            
            // 尝试从 ExtensionLoader 注入或从 Spring 容器获取
            Object object = objectFactory.getExtension(pt, property);
            
            if (object != null) {
                method.invoke(instance, object);
            }
        }
    }
    return instance;
}
```

**示例**：

```java
public class MyProtocol implements Protocol {
    // ExtensionLoader 会自动注入 ProxyFactory 的自适应扩展
    private ProxyFactory proxyFactory;

    public void setProxyFactory(ProxyFactory proxyFactory) {
        this.proxyFactory = proxyFactory;
    }
}
```

# 6. AOP 包装（Wrapper）

Wrapper 是一种特殊的扩展类，其构造函数接受扩展点接口作为参数：

```java
// Wrapper 类实现——ProtocolFilterWrapper
// 不是直接在配置文件中声明，而是 ExtensionLoader 自动识别
public class ProtocolFilterWrapper implements Protocol {
    private final Protocol protocol; // 被包装的对象

    public ProtocolFilterWrapper(Protocol protocol) {
        // 构造函数接受 Protocol 参数 → 被识别为 Wrapper
        this.protocol = protocol;
    }

    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) {
        if (REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
        }
        // 在 export 前后插入 Filter 链构建逻辑
        return protocol.export(buildInvokerChain(invoker, 
            SERVICE_FILTER_KEY, PROVIDER));
    }

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) {
        if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            return protocol.refer(type, url);
        }
        return buildInvokerChain(protocol.refer(type, url), 
            REFERENCE_FILTER_KEY, CONSUMER);
    }
}
```

**Wrapper 包装链**：

```
原始 Protocol 实例
    │
    ▼
ProtocolFilterWrapper (包装 Filter 逻辑)
    │
    ▼
ProtocolListenerWrapper (包装 Listener 逻辑)
    │
    ▼
QosProtocolWrapper (包装 QOS 逻辑)
    │
    ▼
最终暴露给用户的 Protocol
```

# 7. 扩展配置文件格式

文件位置：`META-INF/dubbo/org.apache.dubbo.rpc.Protocol`

```
# 扩展名=全限定类名
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
registry=org.apache.dubbo.registry.integration.RegistryProtocol
rest=org.apache.dubbo.rpc.protocol.rest.RestProtocol
tri=org.apache.dubbo.rpc.protocol.tri.TripleProtocol
```

# 8. 自定义 SPI 扩展完整流程

## 8.1 定义扩展接口

```java
@SPI("default")
public interface RateLimiter {
    boolean acquire(String key, int permits);
}
```

## 8.2 实现扩展

```java
// 默认实现
public class DefaultRateLimiter implements RateLimiter {
    @Override
    public boolean acquire(String key, int permits) {
        return true;  // 不限流
    }
}

// 令牌桶实现
public class TokenBucketRateLimiter implements RateLimiter {
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Override
    public boolean acquire(String key, int permits) {
        return limiters.computeIfAbsent(key,
            k -> RateLimiter.create(permits)).tryAcquire();
    }
}
```

## 8.3 注册扩展

`META-INF/dubbo/com.example.RateLimiter`：

```
default=com.example.DefaultRateLimiter
tokenbucket=com.example.TokenBucketRateLimiter
```

## 8.4 使用扩展

```java
RateLimiter limiter = ExtensionLoader.getExtensionLoader(RateLimiter.class)
    .getExtension("tokenbucket");
limiter.acquire("userService.getUser", 100);
```

或通过 Dubbo URL 参数动态切换：

```java
@DubboService(parameters = {"rateLimiter", "tokenbucket"})
public class UserServiceImpl implements UserService {}
```

# 9. Activate 排序详解

```java
// order 越小越先执行
@Activate(group = PROVIDER, order = -30000)
public class ClassLoaderFilter {}    // 第一个执行

@Activate(group = PROVIDER, order = -10000)
public class MonitorFilter {}        // 第二个执行

@Activate(group = PROVIDER, order = -8000)
public class ValidationFilter {}     // 第三个执行

@Activate(group = PROVIDER, order = -5000)
public class ExecuteLimitFilter {}   // 第四个执行

@Activate(group = PROVIDER, order = -1)
public class ExceptionFilter {}      // 倒数第二个

@Activate(group = PROVIDER, order = 1)
public class TraceFilter {}          // 最后一个
```

获取激活扩展时：

```java
List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class)
    .getActivateExtension(url, "filter", "provider");
// 自动按 order 排序
```

# 10. 三目录策略

| 目录 | 用途 | 示例 |
|------|------|------|
| `META-INF/dubbo/internal/` | Dubbo 内置扩展 | 框架自带实现 |
| `META-INF/dubbo/` | 用户自定义扩展 | 业务方实现 |
| `META-INF/services/` | 兼容 Java SPI | 与 JDK ServiceLoader 互通 |

加载优先级：`internal > dubbo > services`，同名 key 先加载的优先。
