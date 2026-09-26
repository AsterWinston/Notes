
# 1. 应用级服务发现

## 1.1 问题背景

Dubbo 2.x 的**接口级发现**将每个接口都作为一个独立的服务注册到注册中心，导致：

- **注册中心压力大**：接口数量 × 实例数量 = 海量节点
- **推送风暴**：单个 Provider 变更触发大量 Consumer 重新计算
- **元数据膨胀**：每个 URL 都携带完整的服务参数

```
# Dubbo 2.x 注册中心节点结构
/dubbo/com.example.UserService/providers/192.168.1.100:20880
/dubbo/com.example.OrderService/providers/192.168.1.100:20880
/dubbo/com.example.ProductService/providers/192.168.1.100:20880
/dubbo/com.example.PaymentService/providers/192.168.1.100:20880
# ... 100 个接口 = 100 个节点

# 如果有 1000 个 Provider，100 个接口
# = 100,000 个注册节点！
```

## 1.2 解决方案

Dubbo 3 默认使用**应用级发现**：以应用为粒度注册，元数据独立存储。

```
# Dubbo 3 注册中心节点结构
/services/app-a/192.168.1.100:20880  # 仅一个节点

# 接口映射关系存储在元数据中心
MetadataService: { services: ["UserService", "OrderService", ...] }
```

**数据分离**：

| 数据 | 存储位置 | 内容 |
|------|---------|------|
| 实例信息 | 注册中心 (Nacos/ZK) | IP、Port、状态 |
| 接口列表 | 元数据中心 | 应用提供了哪些接口 |
| 接口详情 | 元数据中心 | 方法签名、参数类型 |
| 服务参数 | 元数据中心 | 超时、重试、负载均衡等 |

## 1.3 配置

```yaml
dubbo:
  application:
    register-mode: instance    # instance(应用级) / interface(接口级) / all(双注册)
    metadata-type: remote      # remote(元数据中心) / local(本地文件)
  metadata-report:
    address: nacos://127.0.0.1:8848
```

# 2. Triple 协议

详见 [第 20 章](20_triple-protocol.md)。核心亮点：

- 基于 HTTP/2，支持多路复用
- 兼容 gRPC，双向互调
- 支持 Unary / Server Stream / Client Stream / Bi-di Stream
- 支持浏览器直接访问（HTTP/1.1 降级）

# 3. Service Mesh 支持

## 3.1 Proxyless Mesh

Dubbo 3 支持两种 Mesh 模式：

```
Sidecar Mesh (传统):
  App ←→ Envoy ←→ Network ←→ Envoy ←→ App

Proxyless Mesh (Dubbo 3):
  App ←→ Network ←→ App（Dubbo 直接接入控制面）
```

**Proxyless 优势**：
- 无 Sidecar 开销（CPU/内存/延迟）
- 减少一跳网络延迟
- Dubbo 原生治理能力与 xDS 协议融合

## 3.2 xDS 协议支持

```yaml
dubbo:
  application:
    mesh-enable: true
  registry:
    address: mesh://istiod.istio-system.svc:15012
```

Dubbo 通过 xDS 协议与 Istio/Pilot 控制面通信，接收服务发现和路由规则。

## 3.3 配置

```java
// xDS 配置监听器
public class XdsRegistry extends FailbackRegistry {
    
    @Override
    public void doSubscribe(URL url, NotifyListener listener) {
        // 连接 Istio Pilot → 订阅 xDS 资源
        xdsClient.watchEndpoints(
            url.getServiceInterface(),
            new EndpointWatcher() {
                @Override
                public void onUpdate(List<Endpoint> endpoints) {
                    List<URL> urls = toDubboURLs(endpoints);
                    notify(url, listener, urls);
                }
            }
        );
    }
}
```

# 4. 内置云原生可观测性

## 4.1 指标标准化

Dubbo 3 内置 Metrics API，兼容 Micrometer，可对接 Prometheus / Grafana：

```yaml
dubbo:
  metrics:
    protocol: prometheus
    port: 9090
    enable-jvm: true
    enable-threadpool: true
```

## 4.2 链路追踪

```yaml
dubbo:
  tracing:
    enabled: true
    sampling:
      probability: 0.1         # 采样率 10%
    propagation:
      type: w3c                 # W3C TraceContext 标准
```

# 5. 虚拟线程支持（JDK 21）

Dubbo 3.3+ 原生支持 JDK 21 虚拟线程：

```yaml
dubbo:
  protocol:
    threadpool: virtual       # 使用虚拟线程（Project Loom）
```

```java
// 编程式
ThreadPool virtualThreadPool = new VirtualThreadPool();
DubboBootstrap.getInstance()
    .protocol(new ProtocolConfig("dubbo", 20880)
        .setThreadpool(virtualThreadPool));
```

# 6. 统一路由规则（新）

## 6.1 旧版路由 vs 新版路由

| 特性 | 旧 Router | 新 StateRouter |
|------|----------|----------------|
| 路由粒度 | 接口级 | 应用级 |
| 状态感知 | 无 | 健康/不健康/半健康 |
| 性能 | 每次全量过滤 | 缓存优化 |
| 扩展性 | 有限 | 基于 RouterSnapshot |
| 配置方式 | ConditionRouter/TagRouter | 统一路由规则 DSL |

## 6.2 新路由规则 DSL

```yaml
apiVersion: v3
kind: VirtualService
metadata:
  name: user-service-route
spec:
  hosts:
    - org.example.UserService
  dubbo:
    - route:
        - destination:
            host: user-service
            subset: v2
          weight: 10
        - destination:
            host: user-service
            subset: v1
          weight: 90
```

# 7. 元数据上报与 Revision 机制

```java
// Provider 启动时计算元数据 Revision
public class MetadataInfo {
    private String app;
    private Map<String, ServiceInfo> services;  // 接口→方法签名映射
    private Map<String, String> params;          // 服务参数
    private String revision;                     // 以上内容的 hash
    
    public String calculateRevision() {
        return MD5Utils.md5(
            JSON.toJSONString(this.services) +
            JSON.toJSONString(this.params)
        );
    }
}
```

Consumer 根据 Revision 判断是否需要重新获取元数据：

```
Consumer 启动
  ├── 从注册中心获取 Provider 实例列表
  ├── 检查本地缓存的 Provider Revision
  │   └── Revision 一致 → 跳过元数据拉取
  │   └── Revision 不同 → 调用 MetadataService 获取最新接口定义
  └── 构建 Invoker 代理
```

# 8. 多实例负载均衡增强

Dubbo 3 支持**跨实例自适应负载均衡**（AdaptiveLoadBalance）：

```java
public class AdaptiveLoadBalance extends AbstractLoadBalance {
    
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers,
            URL url, Invocation invocation) {
        
        // 1. 采集 Provider 节点的实时指标
        //    - 响应时间 (滑动窗口)
        //    - 并发请求数
        //    - 错误率
        //    - CPU/内存 (通过 QOS 获取)
        
        // 2. 多维度评分
        for (Invoker<T> invoker : invokers) {
            double score = evaluate(invoker);
            scores.put(invoker, score);
        }
        
        // 3. 选择得分最高的 Provider
        return selectBest(scores);
    }
}
```

# 9. Dubbo 3 迁移指南

## 9.1 从 Dubbo 2.x 迁移

1. **升级依赖**：`dubbo.version` → `3.3.x`
2. **应用级发现过渡**：先设置为 `register-mode: all`（双注册），平稳后再切为 `instance`
3. **协议**：新增 Triple 协议（可与 Dubbo 协议并存）
4. **序列化**：检查自定义序列化兼容性
5. **Filter/SPI**：检查 @Activate group 是否正确

```yaml
# 平滑迁移配置
dubbo:
  application:
    register-mode: all             # 阶段1: 双注册（兼容 Dubbo 2.x Consumer）
    # register-mode: instance       # 阶段2: 纯应用级
  protocol:
    name: dubbo                    # 阶段1: 保留 Dubbo 协议
    # 阶段2: name: tri               # 逐步切换到 Triple
```

## 9.2 JDK 版本升级

```
Dubbo 3.0.x → JDK 8+
Dubbo 3.1.x → JDK 8/17
Dubbo 3.2.x → JDK 17+ (推荐)
Dubbo 3.3.x → JDK 17/21 (支持虚拟线程)
```

# 10. 版本特性速查

| 版本 | 核心新特性 |
|------|-----------|
| 3.0.0 | 应用级服务发现、Triple 协议、重构路由 |
| 3.1.0 | Service Mesh、Proxyless、xDS |
| 3.2.0 | 统一路由规则、StateRouter、自适应负载均衡 |
| 3.3.0 | JDK 21 虚拟线程、Spring Boot 3 原生支持、Reactive |
