# 1. Service Mesh 概述

## 1.1 微服务架构中的通信挑战

当应用从单体演进到微服务后，服务间的通信（东西向流量）变得极其复杂。以下问题在 Kubernetes 原生能力中难以优雅解决：

| 挑战 | 传统方案 | 问题 |
|------|---------|------|
| **服务发现与负载均衡** | kube-proxy + Service | 仅 L4 负载均衡，不支持流量权重、重试 |
| **超时与重试** | 应用代码实现 | 每个语言/框架都要实现一遍 |
| **熔断与限流** | Hystrix/Resilience4j | 应用代码侵入性强，多语言难以统一 |
| **金丝雀发布** | Deployment + Service Label | 流量比例不精确，缺乏自动回滚 |
| **mTLS** | 应用代码配置 TLS | 证书管理复杂，缺乏自动轮换 |
| **链路追踪** | 应用 SDK 嵌入 | 每个服务需要集成，改造成本高 |
| **可观测性** | 各服务独立上报 | 缺乏统一的流量指标和拓扑视图 |

Service Mesh 的核心理念是：**将这些通信关注点从应用代码中剥离，下沉到独立的 Sidecar 代理中，以平台层能力统一提供。**

## 1.2 Service Mesh 的核心架构

```
没有 Service Mesh:
  Service A ────直接调用────→ Service B
  (代码中处理重试、超时、TLS、...)

有 Service Mesh (Sidecar 模式):
  
  ┌─────────────────┐      ┌─────────────────┐
  │   Service A      │      │   Service B      │
  │   (业务代码)       │      │   (业务代码)       │
  │   localhost:8080  │      │   localhost:8080  │
  └────────┬─────────┘      └────────┬─────────┘
           │ localhost                 ▲ localhost
           ▼                           │
  ┌─────────────────┐      ┌─────────────────┐
  │  Envoy Sidecar   │ mTLS │  Envoy Sidecar   │
  │  (代理容器)       │──────│  (代理容器)       │
  │                  │      │                  │
  │ - 负载均衡        │      │ - 负载均衡         │
  │ - 超时/重试       │      │ - 超时/重试        │
  │ - 熔断限流        │      │ - 熔断限流         │
  │ - mTLS           │      │ - mTLS            │
  │ - 指标/追踪       │      │ - 指标/追踪        │
  └─────────────────┘      └─────────────────┘
  
  应用代码只需: curl http://service-b:8080
  Envoy 自动处理: 重试、超时、mTLS、指标上报、追踪
```

### 1.2.1 数据面 (Data Plane) vs 控制面 (Control Plane)

```
┌──────────────────────────────────────────────┐
│              Control Plane (控制面)            │
│                                              │
│  ┌──────────────────────────────────┐        │
│  │         Istiod / Linkerd CP       │        │
│  │  - 服务发现 (Watch K8s API)       │        │
│  │  - 证书管理 (CA, 自动签发mTLS证书) │        │
│  │  - 配置分发 (xDS / gRPC)          │        │
│  └──────────────┬───────────────────┘        │
│                 │ 配置下发                    │
└─────────────────┼────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌────────┐  ┌────────┐  ┌────────┐
│ Envoy  │  │ Envoy  │  │ Envoy  │   ← Data Plane
│ Sidecar│  │ Sidecar│  │ Sidecar│     (数据面)
└────────┘  └────────┘  └────────┘
   Pod A       Pod B       Pod C
```

**数据面**：以 Sidecar 形式运行在每个 Pod 中，负责实际处理流量（转发、加密、限流、上报指标等）。Istio 默认使用 Envoy，Linkerd 使用自研的 linkerd2-proxy。

**控制面**：负责配置管理、服务发现、证书管理。将路由规则、安全策略等配置推送给数据面。

# 2. Istio

## 2.1 Istio 架构

Istio 是目前使用最广泛的 Service Mesh 实现，CNCF 毕业项目。其核心组件（v1.5+ 简化为单体的 istiod）：

```
┌─────────────────────────────────────────────────┐
│                     istiod                        │
│                                                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐     │
│  │ Service   │ │ Security │ │ Configuration │     │
│  │ Discovery │ │ (CA/CSR) │ │ (Galley +     │     │
│  │ (Pilot)   │ │          │ │  Webhook)     │     │
│  └──────────┘ └──────────┘ └──────────────┘     │
│                                                    │
└───────────────────────┬───────────────────────────┘
                        │ xDS (Discovery Service)
                        ▼
              ┌────────────────────┐
              │  Envoy (每Pod一个)    │
              │  - L4/L7 代理       │
              │  - mTLS            │
              │  - Telemetry       │
              └────────────────────┘
```

### 2.1.1 Envoy 的核心能力

- **L4/L7 代理**：TCP/HTTP/gRPC 协议的全功能代理
- **服务发现**：通过 xDS 动态获取后端服务列表
- **负载均衡**：轮询、最少连接、一致性哈希、权重
- **HTTP 路由**：基于 Host/Path/Header 的精确路由
- **健康检查**：主动健康检查 + 被动异常检测
- **熔断（Circuit Breaking）**：连接池限制、异常检测
- **mTLS**：自动双向 TLS 加密通信
- **遥测**：L7 指标（请求率、延迟、错误率）+ 分布式追踪
- **可扩展性**：Lua 过滤器、WASM 插件

## 2.2 Istio 安装

```bash
# 下载 Istio CLI
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20

# 安装（使用 demo profile）
istioctl install --set profile=demo -y

# 启用 Sidecar 自动注入
kubectl label namespace default istio-injection=enabled

# 查看组件
kubectl get pods -n istio-system
# NAME                                    READY   STATUS
# istiod-xxxxxxxx-xxxxx                   1/1     Running
# istio-ingressgateway-xxxxxxxx-xxxxx     1/1     Running
# istio-egressgateway-xxxxxxxx-xxxxx      1/1     Running
```

### 2.2.1 Sidecar 注入原理

Istio 通过 Mutating Admission Webhook 自动在 Pod 中注入 Envoy Sidecar：

```
用户创建 Pod
     │
     ▼
Mutating Webhook 拦截
     │
     ▼
检查 Namespace 标签 istio-injection=enabled
     │
     ▼
在 Pod Spec 中添加:
  - istio-init InitContainer (配置 iptables 规则)
  - istio-proxy Sidecar Container (Envoy 代理)
     │
     ▼
Pod 启动后，所有出入流量被 iptables 规则劫持到 Envoy
```

注入后的 Pod 结构：

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    sidecar.istio.io/status: '{"version":"..."}'
spec:
  initContainers:
  - name: istio-init
    image: istio/proxyv2:1.20
    # 配置 iptables 规则：将 Pod 流量劫持到 Envoy
  containers:
  - name: app
    image: myapp:v1
  - name: istio-proxy            # Envoy Sidecar
    image: istio/proxyv2:1.20
```

## 2.3 VirtualService 与 DestinationRule

### 2.3.1 VirtualService：路由规则

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews                           # 目标 Service 名称
  gateways:
  - my-gateway                        # 可选：绑定到 Gateway
  http:
  # 规则一：金丝雀发布（按权重分流）
  - route:
    - destination:
        host: reviews
        subset: v1                    # 90% 流量到 v1
      weight: 90
    - destination:
        host: reviews
        subset: v2                    # 10% 流量到 v2
      weight: 10

  # 规则二：基于 Header 的路由
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2                    # jason 用户始终访问 v2

  # 规则三：超时和重试
  - route:
    - destination:
        host: reviews
        subset: v1
    timeout: 5s                       # 请求超时 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: connect-failure,refused-stream,5xx

  # 规则四：故障注入（测试弹性）
  - match:
    - headers:
        end-user:
          exact: test-user
    fault:
      delay:
        percentage:
          value: 50                   # 50% 的请求
        fixedDelay: 5s                # 延迟 5 秒
      abort:
        percentage:
          value: 10                   # 10% 的请求
        httpStatus: 500               # 返回 500 错误
    route:
    - destination:
        host: reviews
        subset: v1
```

### 2.3.2 DestinationRule：后端策略

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN             # 负载均衡算法
    connectionPool:                  # 连接池配置
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:                # 异常检测（熔断）
      consecutive5xxErrors: 5        # 连续 5 次 5xx 则踢出
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    tls:
      mode: ISTIO_MUTUAL             # mTLS
  subsets:                           # 定义子集
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v2
    labels:
      version: v2
```

## 2.4 Gateway：入口网关

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway            # 使用 Istio Ingress Gateway Pod
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "bookinfo.example.com"
    - "*.example.com"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "api.example.com"
    tls:
      mode: SIMPLE                   # SIMPLE/MUTUAL/PASSTHROUGH
      credentialName: api-tls-cert   # Kubernetes Secret 名称
```

Istio Gateway 是 Ingress 的升级替代品。它不仅支持 HTTP/HTTPS，还支持 TCP、gRPC、mTLS 等协议。

## 2.5 安全：mTLS 与 AuthorizationPolicy

### 2.5.1 自动 mTLS

```yaml
# 启用全 Mesh 范围的 mTLS（严格模式）
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT                     # STRICT / PERMISSIVE / DISABLE
```

### 2.5.2 细粒度授权策略

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: reviews-access
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW                      # ALLOW / DENY / CUSTOM
  rules:
  # 规则一：允许来自 productpage 的 GET 请求
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/productpage"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/reviews/*"]

  # 规则二：允许来自 monitoring namespace 的 Prometheus
  - from:
    - source:
        namespaces: ["monitoring"]
    to:
    - operation:
        paths: ["/metrics"]

  # 规则三：拒绝 POST/DELETE（安全强化）
  - to:
    - operation:
        methods: ["POST", "PUT", "DELETE"]
    when:
    - key: request.auth.claims[iss]
      notValues: ["accounts.google.com"]
```

## 2.6 Istio 可观测性

### 2.6.1 Kiali：服务拓扑可视化

```bash
# 打开 Kiali Dashboard
istioctl dashboard kiali
# 显示服务拓扑图、流量速率、错误率、延迟分布
```

### 2.6.2 Jaeger：分布式追踪

Istio 自动为所有 HTTP/gRPC 请求注入 Trace Headers（`x-b3-traceid` 等），无需应用代码改动即可获得调用链追踪。

### 2.6.3 Grafana + Prometheus

Istio 自动暴露 Envoy 指标，可通过 Prometheus 抓取并导入 Grafana Dashboard：

```
关键指标：
istio_requests_total                    # 请求总数
istio_request_duration_milliseconds     # 请求延迟
istio_request_bytes                     # 请求大小
istio_response_bytes                    # 响应大小
istio_tcp_received_bytes_total          # TCP 接收字节
istio_tcp_sent_bytes_total              # TCP 发送字节
```

# 3. Linkerd

## 3.1 Linkerd vs Istio

Linkerd 是 CNCF 的另一个毕业级 Service Mesh，设计哲学与 Istio 截然不同：

| 维度 | Istio | Linkerd |
|------|-------|---------|
| **数据面** | Envoy（C++，功能丰富） | linkerd2-proxy（Rust，轻量级） |
| **控制面** | Go，功能全面 | Go + Rust，极度简化 |
| **资源消耗** | 中等 | 极低（每个 Sidecar ~10MB） |
| **安装复杂度** | 中 | 低（一条命令） |
| **功能深度** | 非常丰富 | 保持精简 |
| **适用场景** | 大型企业、多集群 | 中小规模、简洁优先 |
| **学习曲线** | 陡峭 | 平缓 |

### 3.1.1 Linkerd 核心特性

- **零配置 mTLS**：默认启用，自动轮换 24 小时证书
- **轻量级代理**：用 Rust 编写的 micro-proxy，~10MB 内存，< 1ms P99 延迟增量
- **零代码侵入**：不需要修改一行应用代码
- **完善的 Dashboard**：内置可视化界面和 Grafana 集成

## 3.2 Linkerd 安装与使用

```bash
# 安装 CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# 安装控制面
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# 查看状态
linkerd check
linkerd dashboard

# 注入 Sidecar（自动或手动）
linkerd inject deployment.yaml | kubectl apply -f -
# 或
kubectl annotate namespace default linkerd.io/inject=enabled
```

### 3.2.1 Linkerd 特有的调试工具

```bash
# 实时查看请求流
linkerd viz tap deployment/my-app

# 查看服务拓扑
linkerd viz stat deployment
linkerd viz routes deployment/my-app --to deployment/backend

# 查看边缘（Entry/Exit 流量）
linkerd viz edges deployment -n default

# 性能分析
linkerd viz profile deployment/my-app --tap deploy/my-app
```

# 4. 生产实践与选型

## 4.1 何时使用 Service Mesh

```
使用 Service Mesh 的明确信号:
├── 你需要跨服务统一实施 mTLS
├── 你需要金丝雀发布（精确的流量权重控制）
├── 你需要应用无侵入的分布式追踪
├── 你需要在平台层统一实施超时/重试/熔断
├── 你需要 L7 可观测性（HTTP/gRPC 指标）
└── 你的组织有 10+ 微服务且有专业的平台团队

暂不需要 Service Mesh 的信号:
├── 只有 3-5 个服务，金丝雀可用 Deployment + Service Label 实现
├── 团队规模小，没有专职的 K8s 运维人员
├── 应用已通过 SDK 完整实现了重试/熔断/追踪
└── 对性能极度敏感，不接受 Sidecar 的延迟增量（通常 < 1-2ms）
```

## 4.2 Sidecar 资源与性能

Sidecar 会消耗额外资源，需要合理规划：

```yaml
# Istio Envoy Sidecar 资源建议
# 生产环境
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 2000m
    memory: 512Mi

# 轻量流量
resources:
  requests:
    cpu: 10m
    memory: 64Mi
```

**延迟开销**：Envoy Sidecar 通常增加 P99 延迟 2-5ms（取决于配置复杂度）。Linkerd 的 micro-proxy 通常 < 1ms。

## 4.3 Service Mesh 与 API Gateway 的边界

```
外部流量（南北向）
     │
     ▼
┌─────────────────┐
│  API Gateway     │  ← Ingress Controller / Istio Gateway / Kong / APISIX
│  认证 / 限流 /   │     (管理外部→内部流量)
│  路由 / 协议转换  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Service Mesh    │  ← Istio / Linkerd
│  mTLS / 追踪 /   │     (管理服务间流量)
│  重试 / 熔断     │
└─────────────────┘
```

两者职责互补：
- API Gateway 处理**南北向**流量（认证、限流、路由、协议转换）
- Service Mesh 处理**东西向**流量（mTLS、追踪、重试、熔断）

部分网关（如 Istio Gateway、Kong、Envoy Gateway）可以同时承担两者角色，但分离部署通常更清晰。
