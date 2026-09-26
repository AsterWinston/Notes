
# 1. 服务治理概述

服务治理是 Dubbo 的核心价值之一，涵盖服务的全生命周期管理，确保微服务架构的稳定性、可用性和可维护性。

```
┌──────────────────────────────────────────────────┐
│                  服务治理矩阵                       │
├──────────────┬──────────────┬───────────────────┤
│   运行时治理   │   静态治理    │   运维管理          │
├──────────────┼──────────────┼───────────────────┤
│ 动态配置      │ 服务分组      │ 健康检查            │
│ 流量管控      │ 服务版本      │ 优雅上下线           │
│ 灰度发布      │ 接口契约      │ 限流降级            │
│ 路由规则      │ 服务鉴权      │ 监控告警            │
│ 权重调整      │ 依赖管理      │ 全链路压测           │
└──────────────┴──────────────┴───────────────────┘
```

# 2. 动态配置（配置规则）

动态配置通过注册中心下发覆盖提供者或消费者的参数，无需重启即可生效。

## 2.1 配置规则下发流程

```
Dubbo Admin (管理后台)
    │ 创建配置规则
    ▼
注册中心 /configurators 节点
    │
    ▼
Provider RegistryDirectory.notify()
    │ 解析为 OverrideConfigurator
    ▼
覆盖 Provider URL 中的参数
    │
    ▼
生效 → 影响 Consumer 负载均衡、路由等
```

## 2.2 动态配置示例

```yaml
# 动态调整 Provider 权重（不重启）
configVersion: v2.0
scope: service
key: org.example.UserService
enabled: true
configs:
  - side: provider
    addresses: [192.168.1.100:20880]
    parameters:
      weight: 200          # 调整权重
      timeout: 5000        # 调整超时
      loadbalance: leastactive  # 调整负载均衡策略
```

## 2.3 OverrideConfigurator 源码

```java
public class OverrideConfigurator extends AbstractConfigurator {

    @Override
    public List<Invoker<T>> configure(List<Invoker<T>> invokers, 
            Invocation invocation) {
        // 遍历 Invoker 列表，根据规则覆盖 URL 参数
        for (int i = 0; i < invokers.size(); i++) {
            Invoker<T> invoker = invokers.get(i);
            URL url = invoker.getUrl();
            
            if (addresses.contains(url.getAddress())) {
                // 创建新 URL，覆盖参数
                URL newUrl = url.addParameters(parameters);
                invokers.set(i, createNewInvoker(newUrl));
            }
        }
        return invokers;
    }
}
```

# 3. 流量管控

## 3.1 TPS 限流

```java
@DubboService(
    tps = 1000,                    // 服务级 TPS
    methods = {
        @Method(name = "getUser", tps = 5000),
        @Method(name = "createUser", tps = 100)
    }
)
public class UserServiceImpl implements UserService {
}
```

源码实现：

```java
@Activate(group = PROVIDER, value = TPS_LIMIT_RATE_KEY)
public class TpsLimitFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) {
        int rate = invoker.getUrl().getMethodParameter(
            invocation.getMethodName(), TPS_LIMIT_RATE_KEY, -1);
        
        if (rate > 0) {
            // 使用 StatItem 实现滑动窗口计数器
            StatItem statItem = stats.get(serviceKey);
            if (statItem == null) {
                stats.putIfAbsent(serviceKey, 
                    new StatItem(serviceKey, rate, interval));
                statItem = stats.get(serviceKey);
            }
            
            if (!statItem.isAllowable()) {
                throw new RpcException("TPS limit exceeded: " + rate);
            }
        }
        return invoker.invoke(invocation);
    }
}
```

## 3.2 并发控制

```java
@DubboService(
    executes = 200,                // 服务级最大并发
    actives = 100,                 // 消费端并发控制
    methods = {
        @Method(name = "heavyProcess", executes = 50)
    }
)
```

- `executes`：Provider 端单服务最大并发数
- `actives`：Consumer 端单服务最大并发数（ActiveLimitFilter）

## 3.3 连接控制

```yaml
dubbo:
  protocol:
    accepts: 500            # Provider 最大接受连接数
  consumer:
    connections: 2          # 每个 Consumer 到每个 Provider 的连接数
```

# 4. 优雅上下线

## 4.1 优雅上线（预热）

```yaml
dubbo:
  provider:
    warmup: 120000          # 预热时间 ms（默认 10 分钟）
    weight: 100             # 最终权重
```

预热期间权重从 0 线性增长到设定值，避免刚启动的服务承载全量流量。

```java
// 预热权重计算
static int calculateWarmupWeight(int uptime, int warmup, int weight) {
    int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
    return ww < 1 ? 1 : (ww > weight ? weight : ww);
}
```

## 4.2 优雅下线

```yaml
dubbo:
  provider:
    shutdown-wait-time: 10000    # 关闭后等待 ms
```

**下线流程**：

```
1. Kill 信号/K8s preStop → Spring ApplicationContext.close()
2. DubboShutdownHook
   ├── 从注册中心注销服务
   ├── 发送 ReadOnly 事件（Consumer 感知剔除）
   ├── 等待 pending 请求处理完成（最多 shutdown-wait-time ms）
   └── 关闭 Netty Server
```

```java
public class DubboShutdownHook extends Thread {

    @Override
    public void run() {
        // 1. 从注册中心注销
        destroyAllRegistries();
        
        // 2. 注销所有 Protocol
        destroyProtocols();
        
        // 3. 等待请求处理完成
        if (shutdownWaitTime > 0) {
            Thread.sleep(shutdownWaitTime);
        }
    }
}
```

## 4.3 K8s 预停止钩子

```yaml
# Kubernetes Deployment
spec:
  template:
    spec:
      containers:
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "curl -X POST http://localhost:22222/offline && sleep 10"]
```

Dubbo QOS 命令：

```bash
# 应用下线（注册中心取消注册 + 拒绝新请求）
curl -X POST http://localhost:22222/offline

# 查看当前状态
curl http://localhost:22222/live
```

# 5. 服务降级

## 5.1 Mock 降级

```java
// 自动降级（调用失败后）
@DubboReference(mock = "return null")
private UserService userService;

// 强制降级（不走远程调用）
@DubboReference(mock = "force:return null")
private UserService userService;

// 方法级 Mock
@DubboReference(mock = "com.example.mock.UserServiceMock")
private UserService userService;
```

## 5.2 源码

```java
public class MockClusterInvoker<T> implements ClusterInvoker<T> {
    
    @Override
    public Result invoke(Invocation invocation) {
        String mockValue = getUrl().getParameter("mock");
        
        if (isForceMock(mockValue)) {
            // 强制 Mock，不走远程
            return doMockInvoke(invocation, mockValue);
        }
        
        try {
            return invoker.invoke(invocation);
        } catch (RpcException e) {
            return doMockInvoke(invocation, mockValue);
        }
    }

    private Result doMockInvoke(Invocation invocation, String mockValue) {
        if ("return null".equals(mockValue)) {
            return new RpcResult(null);
        }
        if ("return empty".equals(mockValue)) {
            return new RpcResult(getEmptyResult(invocation));
        }
        if (mockValue.startsWith("throw")) {
            throw new RpcException(mockValue);
        }
        // Mock 实现类
        Object mockService = ExtensionLoader.getExtensionLoader(MockService.class)
            .getExtension(mockValue);
        return mockService.invoke(invocation);
    }
}
```

# 6. 本地存根（Stub）

```java
// 本地存根：Consumer 端执行部分本地逻辑
public class UserServiceStub implements UserService {
    private final UserService userService;

    public UserServiceStub(UserService userService) {
        this.userService = userService;
    }

    @Override
    public User getUser(Long id) {
        // 前置处理
        if (id == null || id <= 0) {
            return null;
        }
        
        // 远程调用
        User user = userService.getUser(id);
        
        // 后置处理
        if (user == null) {
            throw new RuntimeException("User not found: " + id);
        }
        return user;
    }
}
```

```java
@DubboReference(stub = "org.example.stub.UserServiceStub")
private UserService userService;
```

# 7. 参数回调

```java
// 回调接口
public interface CallbackListener {
    void changed(String msg);
}

// 服务接口
public interface CallbackService {
    void addListener(String key, CallbackListener listener);
}

// Consumer 端
@DubboReference
private CallbackService callbackService;

public void registerListener() {
    callbackService.addListener("myKey", msg -> {
        System.out.println("Callback received: " + msg);
    });
    // Provider 端可以通过 listener.changed() 回调 Consumer
}
```

```yaml
# Provider 需要配置允许的回调连接数
dubbo:
  provider:
    callbacks: 1000
```

# 8. 事件通知

```java
// oninvoke / onreturn / onthrow
public interface NotifyService {
    void oninvoke(String method, Object[] args);
    void onreturn(Object result, String method, Object[] args);
    void onthrow(Throwable ex, String method, Object[] args);
}

@DubboReference(
    oninvoke = "notifyService.oninvoke",
    onreturn = "notifyService.onreturn",
    onthrow = "notifyService.onthrow"
)
private UserService userService;
```

# 9. 结果缓存

```java
@DubboReference(cache = "lru")
// cache 选项: lru (LRU 缓存)、threadlocal (线程本地缓存)、jcache (JSR107)
private UserService userService;
```

LRU 缓存实现：

```java
public class LruCache implements Cache {
    private final Map<Object, Object> store;

    public LruCache(URL url) {
        int max = url.getParameter("cache.size", 1000);
        this.store = new LinkedHashMap<Object, Object>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
                return size() > max;
            }
        };
    }

    @Override
    public void put(Object key, Object value) {
        store.put(key, value);
    }

    @Override
    public Object get(Object key) {
        return store.get(key);
    }
}
```

# 10. 服务分组与版本

```java
// 版本隔离
@DubboService(version = "1.0.0")  // Provider v1
@DubboService(version = "2.0.0")  // Provider v2

@DubboReference(version = "1.0.0") // Consumer 指定版本
private UserService userServiceV1;

// 分组隔离
@DubboService(group = "read")      // 读服务组
@DubboService(group = "write")     // 写服务组

@DubboReference(group = "read")
private UserService readService;
```

# 11. 多协议、多注册中心

```yaml
dubbo:
  registries:
    beijing:
      address: nacos://10.0.1.1:8848
    shanghai:
      address: nacos://10.0.2.1:8848
      default: false

  protocols:
    dubbo:
      name: dubbo
      port: 20880
    tri:
      name: tri
      port: 50051

  provider:
    registry: beijing,shanghai    # 向两个注册中心注册
    protocol: dubbo,tri            # 通过两个协议暴露
```

```java
@DubboService(registry = "beijing", protocol = "dubbo")
public class BeijingServiceImpl implements UserService {}
```
