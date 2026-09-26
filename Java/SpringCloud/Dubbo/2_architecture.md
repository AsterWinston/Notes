
# 1. 分层架构总览

Dubbo 采用经典的 10 层架构，从上到下分为三层（Biz）、三层（RPC）、四层（Remoting）：

```
┌──────────────────────────────────────────────────────────────┐
│                     Business Layer (业务层)                    │
├──────────────────────────────────────────────────────────────┤
│  Service Layer      │ 业务接口定义与实现，用户核心关注层          │
│  Config Layer       │ 配置解析：XML、注解、属性、API             │
│  Proxy Layer        │ 动态代理：ProxyFactory 生成透明代理        │
├──────────────────────────────────────────────────────────────┤
│                      RPC Layer (RPC 层)                       │
├──────────────────────────────────────────────────────────────┤
│  Registry Layer     │ 服务注册与发现，封装多种注册中心实现         │
│  Cluster Layer      │ 集群容错：Directory Router LoadBalance    │
│  Monitor Layer      │ 调用统计与监控，RPC 次数/耗时/异常          │
├──────────────────────────────────────────────────────────────┤
│                    Remoting Layer (远程通信层)                  │
├──────────────────────────────────────────────────────────────┤
│  Protocol Layer     │ RPC 协议：DubboProtocol TripleProtocol     │
│  Exchange Layer     │ 信息交换：请求/响应包装，同步→异步转换       │
│  Transport Layer    │ 网络传输：Netty Server/Client 抽象         │
│  Serialize Layer    │ 序列化：Hessian2 FastJSON2 Protobuf Kryo  │
└──────────────────────────────────────────────────────────────┘
```

# 2. 一次完整 RPC 调用的核心流程

```
      Consumer                                     Provider
┌───────────────────┐                        ┌───────────────────┐
│ 1. Proxy          │                        │                   │
│    (动态代理)       │                        │                   │
│       │            │                        │                   │
│       2. Cluster   │                        │                   │
│    (路由+负载均衡)   │                        │                   │
│       │            │                        │                   │
│       3. Protocol  │ ──── 请求(序列化)───→    │ 6. Protocol      │
│    (Dubbo/Triple)  │ ←─── 响应(序列化)───   │    (Dubbo/Triple)  │
│       │            │                        │       │            │
│       4. Exchange  │                        │ 7. Dispatcher     │
│    (请求/响应包装)   │                        │    (线程分发)       │
│       │            │                        │       │            │
│       5. Transport │ ──── TCP/IP 网络 ────→  │ 8. Transport      │
│    (Netty Client)  │ ←─── TCP/IP 网络 ────   │    (Netty Server)  │
│                   │                        │       │            │
│                   │                        │ 9. Exporter       │
│                   │                        │    (本地Invoker)    │
│                   │                        │       │            │
│                   │                        │ 10. Service Impl  │
│                   │                        │    (业务实现)       │
└───────────────────┘                        └───────────────────┘
```

**Consumer 端流程：**

1. **Proxy**：通过动态代理创建服务接口的代理对象，调用接口方法时被拦截
2. **Cluster**：从 Directory 获取可用的 Provider 列表，经过 Router（路由）和 LoadBalance（负载均衡）选出目标节点
3. **Protocol**：根据协议构造 RPC Invocation，调用序列化
4. **Exchange**：将 Invocation 包装成 Request 对象，为 Future 模式做准备
5. **Transport**：Netty Client 将请求发送到目标 Provider

**Provider 端流程：**

6. **Protocol**：根据协议解析请求，反序列化得到 Invocation
7. **Dispatcher**：根据 Dispatcher 策略将请求分发给业务线程池（all/direct/message/connection/execution）
8. **Transport**：Netty Server 接收网络请求
9. **Exporter**：维护本地服务的 Invoker 映射，根据 service key 找到对应的 Invoker
10. **ServiceImpl**：反射调用业务实现类的方法，返回结果

# 3. 核心组件关系

```
            ┌──────────────┐
            │   Service    │
            │  (业务接口)    │
            └──────┬───────┘
                   │
        ┌──────────┴──────────┐
        ▼                      ▼
  ServiceConfig          ReferenceConfig
  (服务提供者配置)          (服务消费者配置)
        │                      │
        ▼                      ▼
  ┌────────────┐        ┌────────────┐
  │  Proxy      │        │  Proxy      │
  │ (Javassist  │        │ (Javassist  │
  │ /JDK Proxy) │        │ /JDK Proxy) │
  └──────┬─────┘        └──────┬─────┘
         │                     │
         ▼                     ▼
  ┌──────────────┐       ┌──────────────────┐
  │  Invoker     │       │  ClusterInvoker    │
  │ (Abstract    │       │  ├── Directory     │
  │  Proxy       │       │  ├── Router        │
  │  Invoker)   │       │  ├── LoadBalance   │
  └──────┬───────┘       │  └── Cluster       │
         │               └────────┬─────────┘
         │                        │
         ▼                        ▼
  ┌──────────────┐       ┌──────────────┐
  │  Exporter    │       │  Invoker     │
  │  (暴露服务)    │       │  (远程调用)    │
  └──────────────┘       └──────┬───────┘
                                │
                         ┌──────┴───────┐
                         ▼              ▼
                    ┌──────────┐ ┌──────────┐
                    │ Exchange │ │ Protocol │
                    │  Client  │ │  Adapter  │
                    └──────────┘ └──────────┘
```

# 4. URL —— 配置总线的核心

Dubbo 用 `org.apache.dubbo.common.URL` 贯穿全部生命周期。它本质上是一个不可变的参数容器：

```java
public final class URL implements Serializable {
    private final String protocol;      // 协议: dubbo/http/...
    private final String username;       // 用户名
    private final String password;       // 密码
    private final String host;           // 主机
    private final int port;              // 端口
    private final String path;           // 接口全限定名
    private final Map<String, String> parameters;  // 所有扩展参数
}
```

典型示例：

```
dubbo://192.168.1.100:20880/org.example.UserService
  ?timeout=3000
  &version=1.0.0
  &group=demo
  &loadbalance=random
  &retries=2
  &application=user-provider
```

**URL 在调用链中的流转**：

- ServiceConfig 导出时封装为 URL → 注册到注册中心
- RegistryDirectory 从注册中心拉取 URL 列表
- Router 根据 URL 中的路由规则进行过滤
- LoadBalance 根据 URL 中的参数选择策略
- 最终 DubboInvoker 将 URL 中包含的地址和参数用于发起网络请求

# 5. Invoker —— 可执行对象的抽象

```java
public interface Invoker<T> {
    Class<T> getInterface();
    Result invoke(Invocation invocation) throws RpcException;
    URL getUrl();
    boolean isAvailable();
    void destroy();
}
```

Invoker 是 Dubbo 中最核心的领域模型，代表一个**可执行的对象**：

- 在 Provider 端：`AbstractProxyInvoker` 封装了 Service 实现类的反射调用
- 在 Consumer 端：`DubboInvoker` 封装了向远程 Provider 发起网络调用的逻辑
- 在 Cluster 层：`ClusterInvoker` 封装了目录服务、路由、负载均衡联合决策后的调用

# 6. Invocation —— 调用参数上下文

```java
public interface Invocation {
    String getTargetServiceUniqueName();
    String getMethodName();
    Class<?>[] getParameterTypes();
    Object[] getArguments();
    Map<String, String> getAttachments();
    Invoker<?> getInvoker();
}
```

一次 RPC 调用的完整信息都存储在 Invocation 中：目标服务名、方法名、参数类型、参数值，以及隐式参数（attachments）用于透传上下文（如 traceId）。

# 7. Result —— 调用结果抽象

```java
public interface Result {
    Object getValue();
    Throwable getException();
    boolean hasException();
    Map<String, String> getAttachments();
}
```

支持同步返回（`getValue()`）和异常返回（`hasException()`），同时携带响应的 attachments。

# 8. 设计模式总结

| 设计模式 | 应用场景 |
|---------|---------|
| **代理模式** | Proxy 层：动态代理实现透明远程调用 |
| **装饰器模式** | Filter 链：层层包装 Invoker |
| **责任链模式** | Filter 链上按顺序传递 Invoker |
| **工厂模式** | ExtensionLoader 通过 SPI 创建扩展实例 |
| **模板方法模式** | AbstractConfig/AbstractProtocol 定义框架骨架 |
| **策略模式** | LoadBalance/Cluster/Router 多种策略可切换 |
| **观察者模式** | NotifyListener 监听注册中心数据变更 |
| **适配器模式** | Protocol 适配不同协议实现 |
