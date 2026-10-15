# 1. 为什么需要Ingress

Kubernetes 的 Service 资源提供了 Pod 的网络访问能力，但 Service 本质上是四层（L4）负载均衡。当我们需要将集群内的服务暴露给外部用户时，Service 的两种主要方式——NodePort 和 LoadBalancer——在实际生产环境中暴露出诸多局限性。Ingress 正是为了解决这些问题而设计的七层（L7）负载均衡入口资源。

## 1.1 NodePort和LoadBalancer的局限性

### 1.1.1 NodePort 模式：端口爆炸与运维噩梦

NodePort 是 Kubernetes 最早期的外部访问方案。创建一个类型为 NodePort 的 Service 后，Kubernetes 会在集群中**每一个节点**上打开一个相同的端口（范围通常为 30000-32767），外部流量通过 `<任意节点IP>:<NodePort>` 即可访问到后端的 Pod。

这个方案在小型集群或开发环境中尚可使用，但在生产环境中存在严重问题：

**端口爆炸问题**：每个需要外部访问的服务都要占用一个独立端口。假设你有 50 个微服务需要对外暴露，你就需要 50 个不同的 NodePort。而且这些端口在所有节点上都统一打开，端口冲突的风险随着服务数量线性增长。端口范围只有 2768 个（30000-32767），虽然理论上够用，但管理几百个无规律的数字端口本身就是运维灾难。

**缺乏七层路由能力**：NodePort 不理解 HTTP 协议。你无法基于域名（Host）、URL 路径（Path）或请求头来做路由转发。`http://api.example.com/users` 和 `http://api.example.com/orders` 无法路由到不同的后端服务——它们只能通过不同的端口来区分，这意味着你要么给每个路径分配一个独立的 NodePort（极不优雅），要么在应用层再做一层路由（增加架构复杂度）。

**TLS 终结无处安放**：NodePort 本身不处理 TLS。如果你需要 HTTPS，要么在每个应用的 Pod 里配置证书（证书管理分散且繁复），要么在 NodePort 前面再挂一个 Nginx/HAProxy（那就回到了 Ingress 要解决的问题本身）。

**节点 IP 暴露与高可用问题**：用户需要知道至少一个节点的 IP 才能访问服务。一旦该节点宕机，流量不会自动切换到其他节点——除非你在前面再搭建一个外部负载均衡器。但这又回到了"需要在前面加东西"的循环。

```yaml
# NodePort Service 示例：端口管理混乱的缩影
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: NodePort
  selector:
    app: user-service
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30001
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: NodePort
  selector:
    app: order-service
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30002
---
# 50 个服务 = 50 个 NodePort = 运维灾难
```

### 1.1.2 LoadBalancer 模式：成本与粒度的困境

LoadBalancer 类型的 Service 会调用云厂商的 API 自动创建一个外部负载均衡器（如 AWS NLB/ALB、GCP Cloud Load Balancer、Azure Load Balancer），并将流量转发到对应 Service 的后端 Pod。

这个模式比 NodePort 优雅得多，但实际上在生产环境中存在更严重的架构问题：

**每个 Service 创建一个独立 LB 的成本爆炸**：在云环境中，每个 LoadBalancer 类型的 Service 都会在你的云账号下创建一个真实的负载均衡器实例。假设你部署了 50 个服务，每个都配了 `type: LoadBalancer`，那么云厂商的账单上将出现 50 个负载均衡器的费用。以 AWS 为例，每个 ALB 每月的基础费用约为 $20+，50 个就是 $1000+/月——这还不包括流量费和可用区费用。从架构角度看，这也是一种巨大的资源浪费：这些 LB 都是独立的控制面和数据面，无法共享。

**缺乏七层路由与流量管理**：和 NodePort 一样，基础的 LoadBalancer Service（对应云厂商的四层 LB，如 AWS NLB）只工作在 TCP/UDP 层，不理解 HTTP。你无法实现以下需求：
- 基于 URL 路径的路由（`/api/v1/*` → v1 服务，`/api/v2/*` → v2 服务）
- 基于域名的虚拟主机（`api.example.com` 和 `admin.example.com` 指向不同服务）
- 金丝雀发布（按权重 90/10 分流流量）
- 请求头匹配路由
- 统一限流、认证、跨域处理

**TLS 证书管理分散**：每个需要 HTTPS 的 Service 都要单独配置证书。证书的申请、续期、部署变成了每个服务的独立任务，无法集中管理。

**云厂商绑定**：LoadBalancer Service 的行为完全取决于云厂商的实现。在 AWS 上创建的 Service 配置了 `service.beta.kubernetes.io/aws-load-balancer-*` 注解，迁移到 GCP 或 Azure 时这些注解全部失效，需要重写。这让你的 YAML 文件与特定云厂商深度耦合。

```yaml
# LoadBalancer 模式：一个服务一个 LB，成本爆炸
apiVersion: v1
kind: Service
metadata:
  name: user-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: user-service
  ports:
    - port: 443
      targetPort: 8080
---
# 50 个这样的 Service = 50 个 LB = 每月 $1000+ 账单
```

### 1.1.3 两种模式的共同缺陷总结

| 缺陷 | NodePort | LoadBalancer |
|------|----------|--------------|
| 七层路由（URL/域名） | 不支持 | 不支持 |
| TLS 集中管理 | 不支持 | 不支持（除非用云厂商 ALB 注解） |
| 单入口 | 无（直连节点） | 无（每个服务一个入口） |
| 成本 | 低 | 极高 |
| 流量控制（限流/认证） | 不支持 | 不支持 |
| 金丝雀发布 | 无法原生实现 | 无法原生实现 |
| 跨命名空间路由 | 不支持 | 不支持 |
| 可观测性统一 | 困难 | 困难 |

## 1.2 Ingress解决什么问题

Ingress 是 Kubernetes 在 1.1 版本引入的资源类型（GA 于 1.19），它提供了一种**七层 HTTP/HTTPS 路由**的标准 API。Ingress 的设计哲学是"用一份配置，解决所有入口问题"。

### 1.2.1 统一入口（Single Entry Point）

Ingress 最核心的价值是将集群的外部入口收敛为**一个（或少数几个）**点。所有域名的流量都先到达 Ingress Controller 的 LoadBalancer（只需一个 LB），然后由 Ingress 规则根据域名和路径将流量分发到集群内部不同的 Service。

```
互联网 → [唯一的 LB] → [Ingress Controller Pod(s)] → Ingress 规则分发
                                                       ├→ api.example.com/users   → user-service
                                                       ├→ api.example.com/orders  → order-service
                                                       ├→ admin.example.com       → admin-service
                                                       └→ *.example.com (default) → frontend-service
```

**成本革命**：从 N 个 LoadBalancer 缩减为 1 个（或少量），每月云成本从上千美元降至几十美元。这就是 Ingress 在成本维度的最大价值。

**架构简化**：DNS 只需要将通配符域名 `*.example.com` 指向一个 LB 地址，所有服务的 DNS 管理从此变得透明。新增服务不再需要新增 DNS 记录或 LB 实例，只需写一个 Ingress 规则。

### 1.2.2 七层路由（L7 Routing）

Ingress 工作在 OSI 第七层（应用层），理解 HTTP 协议。这意味着它可以：

**基于域名的虚拟主机（Name-based Virtual Hosting）**：

```
api.example.com → user-service
admin.example.com → admin-service
blog.example.com → blog-service
```

所有域名共用同一个 IP（LB IP），Ingress Controller 根据 HTTP 请求头中的 `Host` 字段将请求转发到对应的后端 Service。这类似于 Nginx 的 `server_name` 指令。

**基于 URL 路径的路由（Path-based Routing）**：

```
api.example.com/users       → user-service
api.example.com/orders      → order-service
api.example.com/products    → product-service
api.example.com/payments    → payment-service
```

同一个域名下的不同 URL 路径可以路由到完全不同的 Service。这在一个大域名下聚合多个微服务时非常实用——前端只需调用一个域名，Ingress 在入口层完成服务拆分。

**路由的组合与优先级**：可以同时使用域名和路径进行路由匹配，而且支持最长路径匹配原则。比如：

```yaml
# /api/ 下的请求先匹配 v2，剩余走 v1
- host: api.example.com
  http:
    paths:
      - path: /api/v2/orders
        backend:
          service:
            name: order-v2-service
            port:
              number: 80
      - path: /api/
        backend:
          service:
            name: api-gateway-service
            port:
              number: 80
```

### 1.2.3 TLS/SSL 终结（TLS Termination）

TLS 终结是指 HTTPS 请求的加密在 Ingress 层被解开（解密），之后以 HTTP 明文形式转发到内部 Service。这是 Ingress 的另一个核心价值：

**集中式证书管理**：所有域名的 TLS 证书都存放在 Kubernetes Secret 中，由 Ingress 统一引用。证书的申请、续期、部署全部集中化。运维人员不需要登录每个 Pod 去更新证书，也不需要在每个应用的配置里指定证书路径。

**内部通信简化**：因为 TLS 在入口层已经被终结，集群内部的 Pod 之间可以使用 HTTP 通信，无需每个服务都配置证书和 HTTPS。在信任的集群网络内，这种模式大幅降低了运维复杂度。当然对于零信任架构，你可能需要 mTLS（Istio/Linkerd 提供的 Service Mesh 层面解决），但这已经超出了 Ingress 的职责范围。

**性能优化**：TLS 握手和加解密是 CPU 密集操作。在 Ingress 层集中处理后端可以：
- 使用专用硬件（如 AWS Nitro）或优化的 SSL 库（如 BoringSSL）
- 复用 SSL Session，减少握手次数
- 集中配置 TLS 版本和密码套件，确保安全合规

### 1.2.4 其他核心能力

**名称解析与服务发现**：Ingress 引用的 Service 名称会被自动解析。当 Service 的 Endpoints 发生变化（Pod 增删、滚动更新）时，Ingress Controller 通过 Watch 机制自动感知并更新后端列表，无需人工介入。

**统一的横切关注点**：通过 Ingress Controller 的注解（Annotations）或中间件（Middleware），可以在入口层统一处理：
- 限流（Rate Limiting）：按 IP、用户、接口维度
- 认证（Authentication）：Basic Auth、OAuth2 Proxy、JWT 验证
- 跨域（CORS）：统一添加 CORS 响应头
- 重定向（Rewrite/Redirect）：URL 重写、HTTP→HTTPS 重定向
- 请求/响应修改：添加/删除请求头、修改响应体
- WAF（Web Application Firewall）：ModSecurity、OpenAppSec 集成
- 链路追踪：注入 Trace ID（如 Jaeger/Zipkin）

**金丝雀发布与 A/B 测试**：通过在 Ingress 层配置权重规则，可以将一部分流量（如 10%）转发到新版本服务，实现无感知的金丝雀发布。当新版本验证通过后，逐步增加权重直至 100%。

**与 Service Mesh 的边界**：很多人会困惑 Ingress 和 Service Mesh（如 Istio）的关系。简单说：
- Ingress 负责**南北向流量（外部→集群）**
- Service Mesh 负责**东西向流量（服务间通信）**

两者职责互补而非替代。许多生产环境同时使用 Ingress（入口层）和 Service Mesh（服务间），后者提供 mTLS、细粒度流量控制、延迟注入等更高级的能力。

# 2. Ingress架构

理解了 Ingress 解决了什么问题之后，我们来深入探讨 Ingress 的架构设计。Kubernetes Ingress 的设计采用了一种很经典的"关注点分离"模式：API 规范与实现分离。

## 2.1 Ingress Controller + Ingress Resource 分离设计

### 2.1.1 设计哲学：标准 API + 可替换实现

Kubernetes 社区在设计 Ingress 时面临一个核心矛盾：如何定义一个统一的入口 API，又不限定具体的代理实现？解决方案是延续 Kubernetes 一贯的模式——定义标准的资源对象（Ingress Resource），由不同的实现方（Ingress Controller）来消费和解释。

```
┌─────────────────────────────────────────────────────┐
│                    标准 API 层                        │
│  Ingress Resource  IngressClass  Secret              │
│  (通用、可移植、与实现无关)                             │
├─────────────────────────────────────────────────────┤
│                    实现层                             │
│  Nginx Ingress Controller  Traefik  HAProxy  Kong    │
│  (各自的配置语法、注解、插件、特性)                       │
└─────────────────────────────────────────────────────┘
```

这种设计的优势：
1. **用户只需学习一套 API**：无论底层是 Nginx、Traefik 还是 HAProxy，用户书写的 Ingress YAML 格式一致。
2. **Controller 可替换**：从 Nginx 迁移到 Traefik 时，基本的 Ingress 规则 YAML 无需大改（注解除外），只需更换部署的 Controller。
3. **生态繁荣**：任何实现了 Ingress 规范的代理都可以成为 Controller，形成了活跃的社区生态。
4. **厂商中立**：不绑死任何一个负载均衡器实现。

### 2.1.2 Ingress Resource：声明式规则定义

Ingress Resource 是一个标准的 Kubernetes API 对象（`apiVersion: networking.k8s.io/v1`），由用户定义"流量应该如何路由"。它关心的是 **WHAT**——什么请求匹配什么规则并转发到哪里。

一个 Ingress Resource 的核心内容包括：
- `spec.ingressClassName`：指定使用哪个 Ingress Controller 处理这条规则
- `spec.rules`：路由规则列表，包含域名、路径、后端 Service
- `spec.tls`：TLS 证书配置
- `spec.defaultBackend`：无规则匹配时的默认后端

Ingress Resource 本身**不会处理任何流量**。它只是存放在 etcd 中的一份配置数据，等待 Ingress Controller 来读取并转化为实际的代理配置。

### 2.1.3 Ingress Controller：配置驱动的代理实例

Ingress Controller 是一个运行在集群中的 Pod（通常以 Deployment 方式部署），它负责：

1. **Watch Ingress 资源**：持续监听 Ingress、Service、Endpoints、Secret 等资源的变化
2. **翻译配置**：将 Ingress 规则翻译成底层代理的配置语法（如 nginx.conf）
3. **热加载**：将新配置应用到底层代理，使其生效
4. **处理流量**：作为实际的代理服务器，接收外部流量并按规则转发

常见的 Ingress Controller 实现：
- **Nginx Ingress Controller**（Kubernetes 社区维护）：基于 OpenResty/Nginx，最成熟
- **NGINX Ingress Controller**（NGINX Inc/F5）：商业版，性能更强
- **Traefik**：Go 实现，支持自动服务发现，配置热更新无需 reload
- **HAProxy Ingress**：基于 HAProxy，性能极高
- **Kong Ingress Controller**：基于 OpenResty，内置 API 网关能力
- **APISIX Ingress Controller**：基于 Apache APISIX，云原生 API 网关
- **Contour**（VMware）：基于 Envoy，支持 Gateway API
- **Istio Ingress Gateway**：Service Mesh 一体化方案

### 2.1.4 分离设计的工程实践

这种设计在工程上有几个精妙之处：

**关注点分离（Separation of Concerns）**：
- 应用开发者只关心 Ingress 规则（将请求路由到我的服务），不需要知道代理的实现细节。
- 平台工程师负责选型和部署 Ingress Controller，配置全局策略（限流、认证、WAF）。
- 两者通过标准 API 解耦，互不干扰。

**配置漂移防护**：Kubernetes 的控制循环（Control Loop）保证了期望状态与实际状态的一致性。当有人手动修改了 Nginx 的配置文件时，Ingress Controller 会在下一个同步周期将其修正回 Ingress 资源定义的期望状态。这避免了"服务器上手动改了一行配置，但没人知道、没有文档"的运维噩梦。

**多 Controller 共存**：一个集群可以部署多个不同类型的 Ingress Controller。通过 IngressClass 资源（v1.18+），可以明确指定哪个 Ingress Resource 由哪个 Controller 处理：

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx

---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: traefik
spec:
  controller: traefik.io/ingress-controller

---
# 使用 Nginx Controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com

---
# 使用 Traefik Controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-ingress
spec:
  ingressClassName: traefik
  rules:
    - host: admin.example.com
```

这允许同一集群内的不同服务使用不同的入口代理——比如外部 API 用性能更强的 Nginx，内部管理界面用带 Web UI 的 Traefik。

## 2.2 请求处理流程

理解一个 HTTP 请求从进入集群到到达 Pod 的完整链路，是排查入口问题的基本功。

### 2.2.1 完整请求链路

```
                          ┌───────────────────────────────────┐
                          │            Kubernetes 集群          │
                          │                                    │
  互联网                   │  ┌──────────────────────────────┐  │
  ───────┐                │  │    Ingress Controller Pod     │  │
         │                │  │  ┌─────────────────────────┐  │  │
         ▼                │  │  │   Nginx/Traefik/HAProxy │  │  │
  ┌─────────────┐  ①     │  │  │   (代理进程)              │  │  │
  │  外部 LB     │────────┼─▶│  │   port: 80/443           │  │  │
  │ (云厂商/自建)│        │  │  └───────────┬─────────────┘  │  │
  └─────────────┘        │  │              │  ③              │  │
                         │  │              │ 按 Ingress      │  │
                         │  │              │ 规则匹配        │  │
                         │  │              │                │  │
                         │  └──────────────┼────────────────┘  │
                         │                 │                   │
                         │    ┌────────────▼────────────────┐  │
                         │    │  Service (ClusterIP)         │  │
                         │    │  10.96.0.x:8080              │  │
                         │    └────────────┬────────────────┘  │
                         │                 │  ④ kube-proxy     │
                         │                 │    iptables/ipvs  │
                         │    ┌────────────▼────────────────┐  │
                         │    │          Pod                  │  │
                         │    │   10.244.1.x:8080            │  │
                         │    │   (应用容器)                    │  │
                         │    └─────────────────────────────┘  │
                         └───────────────────────────────────┘
```

**步骤详解**：

① **外部流量到达 LB**：客户端 DNS 解析域名得到外部 LB 的 IP，HTTP 请求到达 LB。LB 将流量转发到 Ingress Controller Pod 所在的节点。LB 的类型取决于部署方式：
- 云环境：通常是云厂商的 L4 LB（如 AWS NLB），通过 `type: LoadBalancer` 的 Service 自动创建
- 自建机房：可以是 MetalLB（BGP 模式）、或者直接用 NodePort + 外部硬件 LB
- 轻量场景：直接用 NodePort，用户通过 `http://node-ip:30080` 访问

② **请求进入 Ingress Controller Pod**：Ingress Controller 的 Service 将流量转发到 Pod（通常通过 kube-proxy 的 iptables/ipvs 规则）。Controller Pod 内的代理进程（如 Nginx）监听 80/443 端口接收请求。

③ **Ingress 规则匹配**：代理进程根据请求的 Host 头和 URL 路径匹配 Ingress 规则，找到对应的后端 Service 和端口：
- 先匹配 Host（域名），找不到匹配则使用 defaultBackend
- 再匹配 Path（路径），使用最长前缀匹配原则
- 确定最终的 Service 名称和端口号

④ **转发到 Service → Pod**：代理进程向 Service 的 ClusterIP 发起 HTTP 请求（上游调用）。kube-proxy 通过 iptables 或 ipvs 规则将 ClusterIP 的流量随机（或按算法）转发到后端 Pod。这里需要注意：
- Ingress Controller 转发到 Service 的 ClusterIP 时，使用的是 HTTP 协议
- 由于 kube-proxy 也做了 SNAT，Pod 看到的请求源 IP 是 Node 的 IP，而非客户端真实 IP（需要配置 `externalTrafficPolicy: Local` 或使用 PROXY Protocol 来保留真实 IP）

### 2.2.2 性能与可靠性考量

**多副本高可用**：Ingress Controller 通常以 Deployment 方式部署多个副本，分布在不同节点上。外部 LB 通过健康检查（通常是 `/healthz` 端点）探测每个副本的状态，自动摘除不健康的副本。这样即使单个 Pod 或节点故障，流量也能自动切换到健康的副本。

**连接复用**：Ingress Controller 到后端 Pod 的连接通常使用 HTTP Keep-Alive 连接池技术，避免频繁建立 TCP 连接的开销。Nginx 的 `upstream keepalive` 指令可以配置保持的空闲连接数。

**请求缓冲**：代理层会缓冲请求体和响应体，保护慢速客户端和慢速后端互相影响。对于文件上传场景（大请求体），需要适当增大 `proxy-body-size` 和 `proxy-buffer-size`。

**优雅关闭**：Ingress Controller Pod 在收到 SIGTERM 信号（滚动更新或缩容时）后，应该：
1. 等待 LB 将自身摘除（通过 `terminationGracePeriodSeconds` 和 preStop hook）
2. 处理完已在处理的请求（`worker_shutdown_timeout`）
3. 关闭监听端口
4. 退出进程

## 2.3 Ingress Controller工作原理

这是理解 Ingress 的核心——Controller 如何从静态的 YAML 配置变成真正处理流量的代理规则。

### 2.3.1 Watch 机制：Kubernetes 的实时事件驱动

Ingress Controller 不轮询 API Server，而是通过 Kubernetes 的 **List-Watch** 机制订阅资源变更事件。Controller 启动时：

1. **List（全量同步）**：调用 API Server 的 List 接口，获取集群中所有相关的 Ingress、Service、Endpoints、Secret 等资源的当前状态，构建初始的内存模型。
2. **Watch（增量同步）**：在 List 返回的 resourceVersion 基础上，建立一个长连接 Watch。API Server 会将后续的所有变更事件（ADDED / MODIFIED / DELETED）推送给 Controller。

```go
// 伪代码：Controller 的 Watch 逻辑
func (c *Controller) Run(stopCh <-chan struct{}) {
    // List 全量同步
    ingresses, _ := c.client.NetworkingV1().Ingresses("").List(ctx, metav1.ListOptions{})
    for _, ing := range ingresses.Items {
        c.syncIngress(&ing)
    }

    // Watch 增量事件
    watcher, _ := c.client.NetworkingV1().Ingresses("").Watch(ctx, metav1.ListOptions{
        ResourceVersion: ingresses.ResourceVersion,
    })
    for event := range watcher.ResultChan() {
        switch event.Type {
        case watch.Added, watch.Modified:
            c.syncIngress(event.Object.(*v1.Ingress))
        case watch.Deleted:
            c.deleteIngress(event.Object.(*v1.Ingress))
        }
    }
}
```

**监听的资源类型**：

| 资源 | 用途 | 变更时触发 |
|------|------|-----------|
| Ingress | 路由规则的核心配置 | 重新生成 nginx.conf 或路由表 |
| Service | 获取后端 Pod 的端口信息 | 更新 upstream 的端口 |
| Endpoints/EndpointSlice | 获取后端 Pod 的实际 IP 列表 | 更新 upstream 的 IP 列表 |
| Secret | TLS 证书内容 | 重新加载证书文件 |
| ConfigMap | Controller 自身的全局配置 | 更新全局代理配置 |
| IngressClass | 确定哪些 Ingress 由本 Controller 处理 | 添加或移除对某些 Ingress 的监听 |

### 2.3.2 配置生成与 Reload

Controller 监听到变更后，不会立即逐条修改配置，而是**重新生成完整的配置文件**并触发代理进程重新加载。这个过程因 Controller 实现而异：

**Nginx Ingress Controller 的方式**（最经典）：

```
Ingress 变更事件
      │
      ▼
┌─────────────────────┐
│ 读取所有 Ingress     │
│ 读取所有 Service     │
│ 读取所有 Endpoints   │
│ 读取 TLS Secret      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 构建内部模型         │
│ (Go 数据结构)        │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 用 Go Template       │
│ 渲染 nginx.conf      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 写入 /etc/nginx/     │
│ nginx.conf           │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ nginx -s reload      │
│ (优雅重载配置)        │
└─────────────────────┘
```

**Nginx Reload 的代价**：`nginx -s reload` 并非零成本。它会导致：
- 新 Worker 进程启动，旧 Worker 进程在处理完已有请求后退出
- 短暂的资源峰值（新旧进程并存期间，内存和 CPU 翻倍）
- 在高并发下可能导致连接抖动

因此 Nginx Ingress Controller 做了大量优化：
- **合并变更**：在短时间内（默认 200ms）的多次变更合并为一次 reload
- **Endpoints 变更免 reload**：使用 Lua 脚本（OpenResty）动态更新 upstream 的 IP 列表，不触发主配置 reload
- **动态证书加载**：通过 Lua + LuaJIT 实现证书的动态加载

**Traefik 的方式**（无 reload 设计）：

Traefik 是一个完全无 reload 的 Ingress Controller。它使用 Go 原生的 `net/http` 和自定义的路由管理器，所有配置变更通过 Provider 机制直接更新内存中的路由树，整个过程无需重启任何监听器，不丢失任何连接。

```
Ingress 变更 → Traefik Provider → 更新内存路由树 → 立即生效（零中断）
```

这是 Traefik 相比 Nginx Ingress Controller 最大的架构优势之一。但代价是 Traefik 的纯 Go 实现的极限吞吐量通常不如 Nginx C 实现，特别在大量并发连接场景下。

**HAProxy Ingress 的方式**：类似 Nginx，生成 HAProxy 配置文件后 reload。但 HAProxy 支持更优雅的 reload（`-sf` 和 `-st` 机制），连接丢失更少。

### 2.3.3 配置同步的竞态与一致性

一个值得注意的问题：**Controller 内部状态与 API Server 的最终一致性**。

由于 Watch 事件的异步性和批量处理机制，Controller 的内存模型可能与 API Server 的当前状态存在短暂的不一致（通常在亚秒级）。这种不一致会导致：
- 临时将流量转发到已删除的 Pod（直到 Endpoints Watch 事件到达并触发了配置更新）
- 新创建的 Ingress 规则在几秒后才能生效

缓解措施：
- Nginx Ingress Controller 的 `--sync-period` 参数（默认 0，关闭定期全量同步）可设置为 30s 进行定期全量同步
- Endpoints 变更通常<1s 内完成配置更新
- 在滚动更新时使用 Pod 的 `terminationGracePeriodSeconds` 和 preStop hook 提供足够的缓冲时间

### 2.3.4 多 Controller 的事件过滤

当集群中有多个 Ingress Controller 时，每个 Controller 只需要处理属于自己的 Ingress 资源。过滤机制有两种：

**v1.18 前的 annotation 模式**（已废弃）：
```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
```

**v1.18+ 的 IngressClass 模式**（推荐）：
```yaml
spec:
  ingressClassName: nginx
```

Controller 通过 IngressClass 的 `spec.controller` 字段匹配。例如 Nginx Ingress Controller 只处理 `spec.controller: k8s.io/ingress-nginx` 的 IngressClass 所关联的 Ingress。

如果 IngressClass 设置了 `spec.parameters`，还支持更细粒度的配置——比如指定一个 ConfigMap 来覆盖默认的 Controller 全局配置。

# 3. Ingress YAML详解

这一章我们逐字段拆解 Ingress YAML 的每一个配置项。只有理解了每个字段的语义和背后的设计意图，才能在生产环境中写出正确且高效的 Ingress 规则。

## 3.1 完整YAML结构

先看一个生产级别的完整 Ingress YAML，包含所有核心字段：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway-ingress
  namespace: production
  labels:
    app: api-gateway
    team: platform
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/limit-connections: "100"
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-404-service
      port:
        number: 80
  tls:
    - hosts:
        - api.example.com
        - admin.example.com
      secretName: example-com-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-panel-service
                port:
                  number: 80
```

### 3.1.1 API 版本演进

Ingress 的 API 版本经历了三个主要阶段：

| API 版本 | Kubernetes 版本 | 状态 | 主要变化 |
|----------|----------------|------|---------|
| `extensions/v1beta1` | 1.1 - 1.18 | 废弃 | 最早的版本，在 extensions API Group 下 |
| `networking.k8s.io/v1beta1` | 1.14 - 1.21 | 废弃 | 移到 networking API Group，引入 pathType |
| `networking.k8s.io/v1` | 1.19+ | GA（稳定） | pathType 变为必填，ingressClassName 替代 annotation |

**迁移注意事项**：
- `v1` 版本中 `service.name` 和 `service.port.name/number` 是必填的
- `backend` 结构变化较大：从旧版的 `serviceName` + `servicePort`（字符串）变为结构化的 `service.name` + `service.port.number`
- `pathType` 在 `v1` 中为必填字段
- 使用 `kubectl convert` 可以自动转换旧的 YAML

### 3.1.2 metadata 字段

- **name + namespace**：唯一标识一个 Ingress。Ingress 是命名空间级资源，只能引用同一命名空间内的 Service。
- **labels**：用于组织和筛选。例如 `kubectl get ingress -l team=platform`。
- **annotations**：Ingress 最重要的扩展机制。不同 Ingress Controller 通过注解提供各自的扩展功能（重写、限流、认证等）。

## 3.2 rules: host/http/paths/pathType

`spec.rules` 是 Ingress 的核心，定义了一条条路由匹配规则。

### 3.2.1 host 字段：域名匹配

`host` 是一个可选的完全限定域名（FQDN）。它的匹配规则：

- **精确匹配**：`host: api.example.com` 只匹配请求头中 `Host: api.example.com` 的请求。
- **通配符匹配**：`host: "*.example.com"` 匹配 `foo.example.com`、`bar.example.com` 但不匹配 `example.com` 或 `foo.bar.example.com`（只支持一级通配符）。
- **省略 host**：不指定 host 的规则匹配所有域名的请求。通常用于集群内部的简单路由，或作为兜底规则。

```yaml
rules:
  - host: "*.example.com"
    http:
      paths:
        - ...
  - host: api.example.com
    http:
      paths:
        - ...
  - http:
      paths:
        - ...
```

**Host 优先级的实现细节**：在底层 Nginx 配置中，精确匹配的 `server_name` 优先级高于通配符前缀匹配，高于通配符后缀匹配，高于正则匹配。Kubernetes Ingress 规范没有明确定义 host 优先级，但主流 Controller 都会遵循"精确优先于通配符"的原则。

### 3.2.2 http.paths 数组：路径匹配

每个 `host` 下可以定义多条 `paths`，每条路径对应一个后端 Service。路径匹配遵循**最长前缀匹配（Longest Prefix Match）**原则。

```yaml
rules:
  - host: api.example.com
    http:
      paths:
        - path: /api/v2/orders
          pathType: Prefix
          backend:
            service:
              name: order-v2-service
              port:
                number: 80
        - path: /api/v2/
          pathType: Prefix
          backend:
            service:
              name: v2-gateway-service
              port:
                number: 80
        - path: /api/
          pathType: Prefix
          backend:
            service:
              name: v1-gateway-service
              port:
                number: 80
```

**最长前缀匹配的含义**：请求 `/api/v2/orders/123` 会匹配到第一个规则（`/api/v2/orders`），因为它是最长的匹配前缀。请求 `/api/v2/users` 会匹配到第二个规则。请求 `/api/v1/data` 会匹配到第三个规则。

这与大多数 Web 服务器的路由匹配行为一致（Nginx 的 `location`、Apache 的 `RewriteRule`）。

### 3.2.3 backend 字段：后端服务引用

`backend` 定义了匹配成功后流量转发到的目标：

```yaml
backend:
  service:
    name: user-service
    port:
      number: 8080
      # 或使用命名端口：
      # name: http
```

`backend` 也支持 `resource` 类型（用于非 Service 后端，如静态资源），但 `service` 是最常用、最推荐的方式。

## 3.3 pathType (Exact/Prefix/ImplementationSpecific)

`pathType` 是 `networking.k8s.io/v1` 中的必填字段，它定义了路径匹配的语义。有三种取值：

### 3.3.1 Exact：精确匹配

路径必须与请求 URL 的路径完全一致（包括大小写）。`/foo` 只匹配 `/foo`，不匹配 `/foo/`、`/foo/bar` 或 `/Foo`。

```yaml
- path: /healthz
  pathType: Exact
  backend:
    service:
      name: health-check-service
      port:
        number: 8080
```

**适用场景**：精确的 API 端点、健康检查路径、Webhook 回调路径。

### 3.3.2 Prefix：前缀匹配

路径匹配所有以指定前缀开头的请求。`/foo` 匹配 `/foo`、`/foo/`、`/foo/bar`、`/foo/bar/baz` 等。

```yaml
- path: /api/
  pathType: Prefix
  backend:
    service:
      name: api-service
      port:
        number: 80
```

**Prefix 的实现细节**：在 Nginx 的底层实现中，`pathType: Prefix` 被翻译为 `location` 块的前缀匹配。对于 `/api/`，实际生成的配置是：

```nginx
location /api/ {
    proxy_pass http://upstream_balancer;
}
```

注意：如果前缀不以 `/` 结尾，如 `/api`，它也会匹配 `/apiary` 这样的路径（这是 Nginx 原生行为）。因此在生产实践中，推荐路径始终以 `/` 结尾或具体到完整的路径段。

### 3.3.3 ImplementationSpecific：由 Controller 决定

匹配行为完全由 Ingress Controller 的实现决定。这给 Controller 提供了最大的灵活性，但代价是可移植性下降——同一个 Ingress YAML 在 Nginx 和 Traefik 下的行为可能不同。

```yaml
- path: /api
  pathType: ImplementationSpecific
  backend:
    service:
      name: api-service
      port:
        number: 80
```

**典型实现差异**：
- Nginx Ingress Controller：`ImplementationSpecific` 的行为等同于 `Prefix`
- Traefik：可能使用 PathPrefix 规则（Traefik 原生路由语法）
- 不同 Controller 可能在路径规范化、大小写敏感性上存在差异

**生产建议**：优先使用 `Prefix` 和 `Exact`，它们的行为定义明确且可移植。只有在明确知道特定 Controller 的行为并需要利用其特性时，才使用 `ImplementationSpecific`。

## 3.4 defaultBackend (默认后端)

`defaultBackend` 是一个可选的全局兜底后端。当请求的域名和路径都不匹配 `rules` 中的任何规则时，请求被转发到 `defaultBackend`。

### 3.4.1 典型用法

**自定义 404 页面**：

```yaml
spec:
  defaultBackend:
    service:
      name: custom-404-service
      port:
        number: 80
```

这个 Service 指向一个简单应用（如一个 Nginx 容器服务静态资源），返回统一的错误页面。

**全局默认路由**：在一些微服务架构中，使用 API Gateway 作为默认后端，由它来做更复杂的路由决策。

**不设置 defaultBackend**：如果省略，Ingress Controller 通常返回其自身的默认响应（Nginx 返回 404，Traefik 返回 404 页面）。

### 3.4.2 与 rules 的配合

`defaultBackend` 不与任何特定的 host 绑定。它的触发条件是：请求的域名不在任何规则的 host 列表中，或路径不匹配任何规则的 paths。

```yaml
spec:
  defaultBackend:
    service:
      name: catch-all-service
      port:
        number: 80
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80

# 请求 admin.example.com → 命中 defaultBackend（没有 host 规则）
# 请求 api.example.com/orders → 命中 defaultBackend（有 host 但无 path 规则）
# 请求 api.example.com/users → 命中 user-service
```

## 3.5 TLS配置 (hosts/secretName)

`spec.tls` 是一个数组，每条 TLS 配置指定一组域名和对应的 TLS 证书。一个 Ingress 可以配置多组 TLS（比如不同域名使用不同证书）。

### 3.5.1 基本结构

```yaml
spec:
  tls:
    - hosts:
        - api.example.com
        - admin.example.com
      secretName: example-com-tls
    - hosts:
        - another-domain.com
      secretName: another-domain-tls
```

### 3.5.2 Secret 格式要求

TLS Secret 必须是 `kubernetes.io/tls` 类型，包含两个键：
- `tls.crt`：证书文件（PEM 格式，包含从叶子到根的完整证书链）
- `tls.key`：私钥文件（PEM 格式，RSA 或 ECDSA）

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-com-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...
  tls.key: LS0tLS1CRUdJTi...
```

创建方式：

```shell
kubectl create secret tls example-com-tls \
  --cert=fullchain.pem \
  --key=privkey.pem \
  -n production
```

验证证书：

```shell
kubectl get secret example-com-tls -n production -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

### 3.5.3 TLS 匹配规则

Ingress Controller 使用 **SNI（Server Name Indication）**来选择合适的证书。

**SNI 工作原理**：TLS 握手的第一个 ClientHello 报文中包含客户端要访问的域名（SNI 扩展）。服务器根据这个域名选择对应的证书。这使得同一 IP 地址和端口可以服务多个域名的 HTTPS 请求。

**注意事项**：
- 如果 SNI 中的域名不匹配任何 TLS 配置，Ingress Controller 的行为取决于实现：Nginx 返回默认证书（如果配置了），Traefik 返回自签名默认证书
- 旧版浏览器或客户端（如 Java 1.6）可能不支持 SNI，它们将始终获取默认证书
- Let's Encrypt 等 CA 颁发的证书通常是 SAN 证书，一份证书包含多个域名，因此 `hosts` 数组可以包含多个域名引用同一 Secret

### 3.5.4 通配符证书

支持通配符域名（如 `*.example.com`），但 TLS 证书本身需要是由 CA 颁发的通配符证书：

```yaml
spec:
  tls:
    - hosts:
        - "*.example.com"
      secretName: wildcard-example-tls
```

Let's Encrypt 的通配符证书只能通过 DNS-01 Challenge 获取（详见第 6 章）。

### 3.5.5 TLS 与 HTTP 共存

同一个 host 下，可以同时配置 HTTP 和 HTTPS。大多数生产环境会配置 HTTP→HTTPS 重定向：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

## 3.6 IngressClass

IngressClass 在 Kubernetes 1.18 中引入（`networking.k8s.io/v1`），是一个独立的集群级资源。它取代了旧的 `kubernetes.io/ingress.class` annotation 方式。

### 3.6.1 为什么需要 IngressClass

旧方式的问题：

```yaml
# 旧方式：annotation 指定
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
```

- **缺乏标准化**：只是字符串匹配，没有结构化的参数传递
- **无法验证**：没有 Webhook 验证机制来确保 Controller 存在
- **安全风险**：任何用户都可以写任意 annotation，可能错误地将流量路由到错误的 Controller

### 3.6.2 IngressClass 资源结构

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  labels:
    app.kubernetes.io/component: ingress-controller
spec:
  controller: k8s.io/ingress-nginx
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: nginx-config
```

- `spec.controller`：Controller 的唯一标识字符串。Nginx Ingress Controller 使用 `k8s.io/ingress-nginx`，Traefik 使用 `traefik.io/ingress-controller`。
- `spec.parameters`：可引用其他资源（如 ConfigMap 或自定义资源）来传递 Controller 特定的配置。

### 3.6.3 默认 IngressClass

可以在 IngressClass 上设置 annotation 使其成为集群的默认 IngressClass：

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

# 4. 注解 (Annotations) 体系

注解是 Ingress 最强大的扩展机制，也是各 Controller 差异最大的地方。标准 Ingress API 只定义了基本的路由规则，而注解允许 Controller 暴露其特有功能——从 URL 重写到安全策略，从性能调优到流量管理。

## 4.1 Nginx Ingress常用注解

Nginx Ingress Controller（社区版）提供了超过 150 个注解。以下按功能分类介绍最常用的 30+ 个注解。

### 4.1.1 URL 重写与重定向

```yaml
# 重写目标路径：将匹配的路径替换为指定值
nginx.ingress.kubernetes.io/rewrite-target: "/$2"
# 配合正则捕获组使用
# 例如：path: /api/v1(/|$)(.*) → rewrite-target: /api/v2/$2
```

**rewrite-target 详解**：这是最常用也最容易出错的注解。它将匹配到的路径重写为指定的目标路径。使用正则捕获组时，注意路径末尾的 `(/|$)(.*)` 模式，确保 `/api/v1` 和 `/api/v1/users` 都能正确匹配。

```yaml
# 示例：将 /api/v1/* 重写为 /api/v2/* 并转发到 v2-service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-rewrite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: "/api/v2/$2"
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: v2-service
                port:
                  number: 80
```

```yaml
# 永久重定向（301）
nginx.ingress.kubernetes.io/permanent-redirect: "https://new-domain.com"

# 临时重定向（302）
nginx.ingress.kubernetes.io/temporal-redirect: "https://maintenance.example.com"

# 强制 HTTPS 重定向
nginx.ingress.kubernetes.io/ssl-redirect: "true"
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

# 从路径中移除指定前缀后转发
nginx.ingress.kubernetes.io/rewrite-target: "/"
# 配合 path: /app(/|$)(.*) 可以把 /app/users 变成 /users 转发
```

### 4.1.2 跨域（CORS）

```yaml
# 启用 CORS
nginx.ingress.kubernetes.io/enable-cors: "true"

# 允许的源（多个用逗号分隔）
nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com, https://admin.example.com"

# 允许的 HTTP 方法
nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, PATCH, OPTIONS"

# 允许的请求头
nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type, X-Requested-With"

# 预检请求的缓存时间（秒）
nginx.ingress.kubernetes.io/cors-max-age: "3600"

# 是否允许携带凭证（Cookie）
nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
```

### 4.1.3 认证与安全

```yaml
# Basic Auth 认证
nginx.ingress.kubernetes.io/auth-type: basic
nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

# OAuth2 外部认证
nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com/oauth2/auth"
nginx.ingress.kubernetes.io/auth-signin: "https://auth.example.com/oauth2/sign_in"

# IP 白名单
nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8, 172.16.0.0/12"

# ModSecurity WAF
nginx.ingress.kubernetes.io/enable-modsecurity: "true"
nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"

# Client Certificate 认证（mTLS）
nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
nginx.ingress.kubernetes.io/auth-tls-verify-depth: "2"

# 隐藏 Nginx 版本号（安全最佳实践）
nginx.ingress.kubernetes.io/server-snippet: |
  server_tokens off;
  add_header X-Frame-Options "DENY" always;
```

### 4.1.4 限流与速率控制

```yaml
# 限制单个 IP 的并发连接数
nginx.ingress.kubernetes.io/limit-connections: "20"

# 限制单个 IP 每秒请求数
nginx.ingress.kubernetes.io/limit-rps: "100"

# 突发允许
nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"

# 白名单（不受限流影响）
nginx.ingress.kubernetes.io/limit-whitelist: "10.0.0.0/8, 127.0.0.1"
```

### 4.1.5 请求/响应大小与超时

```yaml
# 最大请求体大小（文件上传等场景）
nginx.ingress.kubernetes.io/proxy-body-size: "100m"

# 代理缓冲区大小（用于存储响应头）
nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"

# 禁用响应缓冲（适用于 SSE/WebSocket 场景）
nginx.ingress.kubernetes.io/proxy-buffering: "off"

# 与后端建立连接的超时（秒）
nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"

# 从后端读取响应的超时（秒）
nginx.ingress.kubernetes.io/proxy-read-timeout: "120"

# 向后端发送请求的超时（秒）
nginx.ingress.kubernetes.io/proxy-send-timeout: "120"

# upstream keepalive 空闲连接数
nginx.ingress.kubernetes.io/upstream-keepalive-connections: "32"
```

### 4.1.6 负载均衡与 Session 保持

```yaml
# 负载均衡算法
nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr"

# Session 保持（粘性会话）
nginx.ingress.kubernetes.io/affinity: "cookie"
nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
nginx.ingress.kubernetes.io/session-cookie-path: "/"
nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
nginx.ingress.kubernetes.io/session-cookie-samesite: "Lax"
nginx.ingress.kubernetes.io/session-cookie-httponly: "true"
```

### 4.1.7 WebSocket 与长连接

```yaml
# WebSocket 支持（默认已启用，但需要调整超时）
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
# SSE 需要关闭缓冲
nginx.ingress.kubernetes.io/proxy-buffering: "off"
```

### 4.1.8 金丝雀发布

```yaml
# 启用金丝雀模式
nginx.ingress.kubernetes.io/canary: "true"
# 基于权重
nginx.ingress.kubernetes.io/canary-weight: "10"
# 基于请求头
nginx.ingress.kubernetes.io/canary-by-header: "x-canary"
# 基于 Cookie
nginx.ingress.kubernetes.io/canary-by-cookie: "canary_user"
```

### 4.1.9 其他常用注解

```yaml
# 后端协议（HTTPS/gRPC 后端）
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

# 是否验证后端证书
nginx.ingress.kubernetes.io/secure-backends: "true"

# 自定义错误页面
nginx.ingress.kubernetes.io/custom-http-errors: "404,500,502,503,504"

# 自定义 Nginx 配置片段
nginx.ingress.kubernetes.io/configuration-snippet: |
  more_set_headers "X-Custom: value";

# gRPC 支持
nginx.ingress.kubernetes.io/backend-protocol: "GRPCS"
```

## 4.2 不同Ingress Controller的注解差异

注解的最大问题是**不可移植**——每个 Controller 有自己的注解体系。下面列出主要 Controller 对应同一功能的注解对比：

### 4.2.1 功能对比表

| 功能 | Nginx Ingress | Traefik | HAProxy Ingress | Kong |
|------|--------------|---------|-----------------|------|
| URL 重写 | `rewrite-target` | `rewrite-target` 或 Middleware CRD | ConfigMap | `konghq.com/rewrite` |
| 限流 | `limit-rps` | Middleware CRD | `haproxy.org/rate-limit` | `konghq.com/rate-limiting` 插件 |
| CORS | `enable-cors` | Middleware CRD | `haproxy.org/cors` | `konghq.com/cors` 插件 |
| Basic Auth | `auth-type: basic` | Middleware CRD | `haproxy.org/auth-type` | `konghq.com/basic-auth` 插件 |
| 粘性会话 | `affinity: cookie` | `affinity` | `haproxy.org/affinity` | `konghq.com/session` 插件 |
| 白名单 | `whitelist-source-range` | Middleware CRD | `haproxy.org/whitelist` | `konghq.com/ip-restriction` 插件 |

### 4.2.2 Traefik 的 Middleware CRD

Traefik 在很大程度上减少了注解的使用，而是引入了 **Middleware CRD**（自定义资源）。这使得配置更结构化、可验证，也更可复用。

```yaml
# Traefik 中间件示例：限流中间件
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    average: 100
    burst: 50
---
# Ingress 引用中间件
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: "default-rate-limit@kubernetescrd"
spec:
  ...
```

Middleware 可以被多个 Ingress 共享引用，这解决了注解无法复用的问题——在 Nginx 中，你需要在每个 Ingress 的注解中重复声明相同的限流配置；而在 Traefik 中，定义一次 Middleware 即可被所有路由引用。

### 4.2.3 注解带来的可移植性问题

注解的不可移植性是 Ingress 最大的设计缺陷之一。这正是 Gateway API（第 7 章）试图解决的核心问题——通过结构化的、不依赖特定实现的 API 来表达入口策略。

**迁移建议**：如果你预见到未来可能更换 Ingress Controller：
1. 尽量使用 Ingress 标准字段（host、path、pathType）实现基本路由
2. 对扩展功能尽量保持简洁，避免过度依赖特定 Controller 的注解
3. 考虑使用 Gateway API 替代 Ingress，它的可移植性更好
4. 如果必须使用大量注解，编写迁移脚本或使用 Helm 模板来管理
# 5. 甯歌Ingress Controller

閫夋嫨涓€涓€傚悎浣犵殑 Ingress Controller 鏄叆鍙ｆ灦鏋勮璁＄殑鍏抽敭鍐崇瓥銆傛瘡涓?Controller 鏈夎嚜宸辩殑璁捐鍝插銆佹€ц兘鐗瑰緛鍜岄€傜敤鍦烘櫙銆?
## 5.1 Nginx Ingress Controller

Nginx Ingress Controller 鏄?Kubernetes 鐢熸€佷腑浣跨敤鏈€骞挎硾鐨?Ingress Controller锛岀敱 Kubernetes 绀惧尯缁存姢锛圙itHub 浠撳簱 `kubernetes/ingress-nginx`锛夈€?
### 5.1.1 鏋舵瀯锛氬熀浜?OpenResty

绀惧尯鐗?Nginx Ingress Controller 鐨勬牳蹇冧笉鏄函 Nginx锛岃€屾槸 **OpenResty**锛堜竴涓泦鎴愪簡 LuaJIT 鐨?Nginx 鍙戣鐗堬級銆傝繖涓璁￠€夋嫨闈炲父鍏抽敭锛屽洜涓哄畠瑙ｅ喅浜嗙函 Nginx 鐨勫嚑澶х棝鐐癸細

**涓轰粈涔堢敤 Lua锛?*
- **鍔ㄦ€?upstream 鏇存柊**锛氱函 Nginx 淇敼 upstream 闇€瑕?`nginx -s reload`锛孡ua 鑴氭湰鍙互鐩存帴淇敼鍐呭瓨涓殑 upstream 鍒楄〃锛岄浂涓柇
- **鍔ㄦ€佽瘉涔﹀姞杞?*锛歍LS 璇佷功鍙樻洿鏃朵笉闇€瑕?reload锛孡ua 鑴氭湰鍙互浠庢枃浠剁郴缁熷姩鎬佸姞杞芥柊璇佷功
- **杩愯鏃堕€昏緫瀹氬埗**锛氬彲浠ュ湪璇锋眰澶勭悊鐨勫悇涓樁娈垫彃鍏?Lua 浠ｇ爜
- **鐩戞帶鎸囨爣鏆撮湶**锛氶€氳繃 Lua 鏀堕泦璇锋眰绾у埆鐨勬寚鏍囧苟鏆撮湶涓?Prometheus 鏍煎紡

```
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?         Nginx Ingress Controller       鈹?鈹?                                        鈹?鈹? 鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹? 鈹?鈹? 鈹?     Go 鎺у埗骞抽潰                    鈹? 鈹?鈹? 鈹? - Watch K8s API                  鈹? 鈹?鈹? 鈹? - 鐢熸垚 nginx.conf 妯℃澘            鈹? 鈹?鈹? 鈹? - 绠＄悊 reload 鐢熷懡鍛ㄦ湡            鈹? 鈹?鈹? 鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹? 鈹?鈹?               鈹?                        鈹?鈹? 鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈻尖攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹? 鈹?鈹? 鈹?   OpenResty (Nginx + LuaJIT)     鈹? 鈹?鈹? 鈹? - HTTP/HTTPS 澶勭悊                鈹? 鈹?鈹? 鈹? - Lua 鎻掍欢锛堝姩鎬?upstream 绛夛級     鈹? 鈹?鈹? 鈹? - Prometheus 鎸囨爣                 鈹? 鈹?鈹? 鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹? 鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

### 5.1.2 绀惧尯鐗?vs NGINX INC 鐗?
| 鐗规€?| 绀惧尯鐗堬紙kubernetes/ingress-nginx锛?| NGINX INC 鐗堬紙nginxinc/kubernetes-ingress锛?|
|------|----------------------------------|------------------------------------------|
| 缁存姢鏂?| Kubernetes 绀惧尯 | F5 / NGINX Inc |
| 浠ｇ悊寮曟搸 | OpenResty (Nginx + LuaJIT) | NGINX Plus锛堝晢涓氾級/ Nginx OSS |
| 璁稿彲璇?| Apache 2.0 | Apache 2.0 (OSS) / 鍟嗕笟 (Plus) |
| 鍔ㄦ€侀噸杞?| Lua 鑴氭湰锛堥儴鍒嗗厤 reload锛?| NGINX Plus API锛堝叏鍔ㄦ€侊級 |
| Gateway API 鏀寔 | 瀹為獙鎬?| 閫氳繃 NGINX Gateway Fabric |
| 鍟嗕笟鏀寔 | 鏃?| 鏈夛紙F5 鎻愪緵锛?|

**閫夊瀷寤鸿**锛?- 缁濆ぇ澶氭暟鍦烘櫙閫夋嫨**绀惧尯鐗?*鍗冲彲锛屽畠鍔熻兘瀹屽杽銆佺ぞ鍖烘椿璺冦€佹枃妗ｄ赴瀵?- 濡傛灉闇€瑕佸晢涓氭敮鎸併€佹洿楂樼殑鎬ц兘澶╄姳鏉裤€佹垨 NGINX Plus 鐨勯珮绾у姛鑳斤紝閫夋嫨 NGINX INC 鐗?
### 5.1.3 閮ㄧ讲鏂瑰紡

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
    spec:
      serviceAccountName: ingress-nginx
      containers:
        - name: controller
          image: registry.k8s.io/ingress-nginx/controller:v1.9.0
          args:
            - /nginx-ingress-controller
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
            - --election-id=ingress-controller-leader
            - --controller-class=k8s.io/ingress-nginx
            - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          readinessProbe:
            httpGet:
              path: /healthz
              port: 10254
            initialDelaySeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 10254
            initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
  selector:
    app.kubernetes.io/name: ingress-nginx
```

**鍏抽敭閮ㄧ讲鍙傛暟瑙ｉ噴**锛?- `--election-id`锛氬鍓湰涔嬮棿浣跨敤 Leader Election 鏈哄埗閫変妇涓?Controller
- `--publish-service`锛氭寚瀹氬澶栧彂甯冪殑 Service锛岀敤浜庡皢 Service 鐨?External IP 鍐欏叆 Ingress 鐨?`status.loadBalancer` 瀛楁
- `--controller-class`锛氱敤浜庡尮閰?IngressClass 鐨?`spec.controller`

### 5.1.4 閰嶇疆鏂囦欢锛圕onfigMap锛?
Nginx Ingress Controller 鐨勫ぇ閮ㄥ垎鍏ㄥ眬琛屼负閫氳繃 ConfigMap锛?
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  proxy-body-size: "50m"
  proxy-connect-timeout: "15"
  proxy-read-timeout: "600"
  proxy-send-timeout: "600"
  proxy-buffer-size: "16k"
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
  ssl-prefer-server-ciphers: "true"
  load-balance: "round_robin"
  worker-processes: "4"
  max-worker-connections: "65536"
  limit-req-status-code: "429"
  hsts: "true"
  hsts-max-age: "63072000"
  use-http2: "true"
  server-tokens: "false"
  use-gzip: "true"
```

## 5.2 Traefik

Traefik 鏄?Containous 鍏徃寮€鍙戠殑浜戝師鐢熻竟缂樿矾鐢卞櫒锛屽畬鍏ㄤ娇鐢?Go 璇█缂栧啓銆?
### 5.2.1 鏍稿績鐗规€?
**闆?reload 鏋舵瀯**锛氳繖鏄?Traefik 鏈€澶х殑鍗栫偣銆傚畠鐨勮矾鐢辫〃瀹屽叏鍦ㄥ唴瀛樹腑缁存姢锛岄厤缃彉鏇存椂鐩存帴淇敼鍐呭瓨涓殑璺敱鏍戯紝鏃犻渶閲嶅惎鎴?reload 杩涚▼銆傝繖鎰忓懗鐫€闆惰繛鎺ヤ腑鏂拰闆惰祫婧愬嘲鍊笺€?
**鑷姩鏈嶅姟鍙戠幇**锛歍raefik 鍘熺敓鏀寔澶氱 Provider锛欿ubernetes Ingress銆並ubernetes CRD锛圛ngressRoute锛夈€丏ocker銆丆onsul / etcd銆丄WS ECS 绛夈€?
**涓棿浠堕摼**锛歍raefik 寮曞叆浜嗕腑闂翠欢锛圡iddleware CRD锛夌殑姒傚康銆傛瘡涓腑闂翠欢鏄竴涓嫭绔嬬殑鍔熻兘鍗曞厓锛屽彲浠ヨ涓嶅悓鐨勮矾鐢辩粍鍚堝紩鐢ㄥ拰澶嶇敤銆?
```
璇锋眰 鈫?EntryPoint(80/443) 鈫?Router(Host+Path鍖归厤) 鈫?Middleware(闄愭祦鈫掕璇佲啋CORS鈫掗噸鍐? 鈫?Service(鍚庣)
```

**鍐呯疆浠〃鐩?*锛歍raefik 鎻愪緵浜嗕竴涓?Web UI锛屽彲浠ュ彲瑙嗗寲鎵€鏈夎矾鐢便€佹湇鍔°€佷腑闂翠欢鍜屽仴搴风姸鎬併€?
### 5.2.2 閮ㄧ讲涓庨厤缃?
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
  namespace: traefik
spec:
  replicas: 2
  selector:
    matchLabels:
      app: traefik
  template:
    spec:
      serviceAccountName: traefik
      containers:
        - name: traefik
          image: traefik:v2.10
          args:
            - --api.insecure=true
            - --api.dashboard=true
            - --providers.kubernetesingress
            - --entrypoints.web.address=:80
            - --entrypoints.websecure.address=:443
            - --metrics.prometheus=true
          ports:
            - name: web
              containerPort: 80
            - name: websecure
              containerPort: 443
            - name: admin
              containerPort: 8080
```

### 5.2.3 涓棿浠剁ず渚?
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
spec:
  rateLimit:
    average: 100
    burst: 50
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: retry
spec:
  retry:
    attempts: 3
    initialInterval: 100ms
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: api-route
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`api.example.com`) && PathPrefix(`/users`)
      middlewares:
        - name: rate-limit
        - name: retry
      services:
        - name: user-service
          port: 80
  tls:
    secretName: example-com-tls
```

### 5.2.4 Traefik 鐨勫急鐐?
- **鏋侀檺鍚炲悙閲?*锛氱函 Go 瀹炵幇鐨勫悶鍚愰噺鍦ㄦ瀬楂樺苟鍙戝満鏅紙10涓? 骞跺彂杩炴帴锛夊彲鑳戒笉濡?Nginx C 瀹炵幇
- **鍐呭瓨鍗犵敤**锛氬唴瀛樹腑鐨勮矾鐢辫〃闇€瑕佸崰鐢ㄨ緝澶氬唴瀛?- **绀惧尯鐢熸€?*锛氭敞瑙ｅ拰閰嶇疆绀轰緥涓嶅 Nginx Ingress 涓板瘜
- **璋冭瘯澶嶆潅搴?*锛氬綋閰嶇疆鍑洪棶棰樻椂锛孏o 绋嬪簭鐨勮皟璇曟瘮 Nginx 閰嶇疆鏂囦欢鐨勯€愯鎺ㄧ悊鏇翠緷璧栨棩蹇?
## 5.3 HAProxy Ingress

HAProxy Ingress Controller 灏嗘垚鐔熺殑 HAProxy 璐熻浇鍧囪　鍣ㄥ甫鍏ヤ簡 Kubernetes 涓栫晫銆?
### 5.3.1 鏍稿績浼樺娍

- **鏋侀珮鐨?L4/L7 鎬ц兘**锛欻AProxy 鏄?C 瀹炵幇銆佷簨浠堕┍鍔ㄧ殑鍗曡繘绋嬫ā鍨?- **涓板瘜鐨勮礋杞藉潎琛＄畻娉?*锛歊ound-robin銆丩east connections銆丼ource IP hash銆乁RI hash 绛?- **瀹屾暣鐨勫仴搴锋鏌?*锛氫富鍔ㄥ仴搴锋鏌ワ紙TCP/HTTP/SSL锛夋瘮 Nginx 鐨勮鍔ㄦ帰娴嬫洿鍙婃椂
- **绋冲畾鐨勫唴瀛樺崰鐢?*锛氬唴瀛樹娇鐢ㄩ潪甯稿彲棰勬祴
- **浼橀泤鐨?reload**锛欻AProxy 鐨?`-sf` 妯″紡鍦?reload 鏃朵繚鎸佹棫杩涚▼鐩村埌杩炴帴澶勭悊瀹屾瘯

### 5.3.2 閰嶇疆鏂瑰紡

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-ingress
data:
  ssl-ciphers: "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"
  ssl-options: "no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets"
  timeout-client: "50s"
  timeout-server: "50s"
  balance-algorithm: "roundrobin"
```

## 5.4 Kong Ingress Controller

Kong 鏄竴涓熀浜?OpenResty 鐨?API 缃戝叧锛屽畠鐨?Ingress Controller 灏?API 缃戝叧鍜?Ingress 鐨勫姛鑳界粺涓€璧锋潵銆?
### 5.4.1 鏍稿績浼樺娍

**API 缃戝叧 + Ingress 涓€浣撳寲**锛欿ong 涓嶄粎鏄祦閲忓叆鍙ｏ紝杩樻槸涓€涓畬鏁寸殑 API 绠＄悊骞冲彴锛?- 涓板瘜鐨勬彃浠剁敓鎬侊紙100+ 鎻掍欢锛?- API Key 绠＄悊鍜岃璇?- 娑堣垂鑰咃紙Consumer锛夊拰 ACL 绠＄悊
- 寮€鍙戣€呴棬鎴?
**鎻掍欢绯荤粺**锛?
```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
plugin: rate-limiting
config:
  minute: 100
  policy: local
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    konghq.com/plugins: rate-limiting
spec:
  ...
```

**澹版槑寮忛厤缃?*锛欿ong 鏀寔 DB-less 妯″紡锛屾墍鏈夐厤缃粠 Kubernetes CRD 澹版槑寮忕鐞嗭紝鏃犻渶澶栭儴鏁版嵁搴撱€?
## 5.5 APISIX Ingress Controller

Apache APISIX 鏄?Apache 杞欢鍩洪噾浼氱殑椤剁骇椤圭洰銆?
### 5.5.1 鏍稿績鐗规€?
- **鍏ㄥ姩鎬侀厤缃?*锛氶€氳繃 etcd 瀛樺偍閰嶇疆锛屾彃浠跺拰搴旂敤閰嶇疆瀹炴椂鐢熸晥锛屾棤闇€ reload
- **涓板瘜鎻掍欢**锛?0+ 鍐呯疆鎻掍欢锛屾敮鎸佺儹鍔犺浇
- **澶氬崗璁敮鎸?*锛欻TTP銆乬RPC銆丏ubbo銆丮QTT 绛?
```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpbin-route
spec:
  http:
    - name: rule1
      match:
        hosts:
          - api.example.com
        paths:
          - /users/*
      backends:
        - serviceName: user-service
          servicePort: 80
      plugins:
        - name: limit-count
          enable: true
          config:
            count: 100
            time_window: 60
```

## 5.6 鍚凜ontroller瀵规瘮涓庨€夊瀷

### 5.6.1 缁煎悎瀵规瘮琛?
| 缁村害 | Nginx Ingress | Traefik | HAProxy | Kong | APISIX |
|------|--------------|---------|---------|------|--------|
| 瀹炵幇璇█ | Go + Nginx(C) + Lua | Go | C | OpenResty(Lua) | Nginx(C) + Lua |
| Reload 寮€閿€ | 閮ㄥ垎鍏?reload | 瀹屽叏鍏?reload | 浼橀泤 reload | 鍔ㄦ€?| 瀹屽叏鍏?reload |
| 鎬ц兘锛堝悶鍚愶級 | 鏋侀珮 | 楂?| 鏋侀珮 | 鏋侀珮 | 鏋侀珮 |
| K8s 闆嗘垚搴?| 鏈€楂?| 楂?| 涓?| 楂?| 楂?|
| 绀惧尯瑙勬ā | 鏈€澶?| 澶?| 涓?| 涓?| 蹇€熷闀?|
| 閰嶇疆鏂瑰紡 | Annotations + ConfigMap | CRD + Annotations | ConfigMap | CRD + Annotations | CRD |
| 鍐呯疆 Web UI | 鍚?| 鏄?| 鍚?| 鏄?| 鏄?|
| 涓棿浠?鎻掍欢 | Annotation 鐗囨 | Middleware CRD | ConfigMap snippet | KongPlugin CRD | Plugin CRD |
| gRPC 鏀寔 | 鏄?| 鏄?| 鏄?| 鏄?| 鏄?|
| TCP/UDP 浠ｇ悊 | 鏄?| 鏄?| 鏄?| 鍚?| 鏄?|
| 瀛︿範鏇茬嚎 | 涓瓑 | 浣?| 楂?| 涓珮 | 涓瓑 |
| 鍟嗕笟鏀寔 | 鏃?| Traefik Enterprise | HAProxy Enterprise | Kong Konnect | API7.ai |

### 5.6.2 閫夊瀷鍐崇瓥鏍?
```
闇€瑕?API 缃戝叧鍔熻兘锛圓PI Key 绠＄悊銆佸紑鍙戣€呴棬鎴凤級锛?鈹溾攢 鏄?鈫?Kong 鎴?APISIX
鈹斺攢 鍚?鈫?缁х画
    闇€瑕侀浂 reload銆佽嚜鍔ㄦ湇鍔″彂鐜般€佸彲瑙嗗寲绠＄悊锛?    鈹溾攢 鏄?鈫?Traefik
    鈹斺攢 鍚?鈫?缁х画
         杩芥眰鏋佽嚧鎬ц兘銆佸凡鏈?HAProxy 杩愮淮缁忛獙锛?         鈹溾攢 鏄?鈫?HAProxy Ingress
         鈹斺攢 鍚?鈫?Nginx Ingress Controller锛堟渶绋冲Ε鐨勯€夋嫨锛?```

**涓€鑸帹鑽?*锛?- **榛樿閫夋嫨**锛歂ginx Ingress Controller锛堢ぞ鍖虹増锛夆€斺€旀枃妗ｆ渶鍏ㄣ€佺ぞ鍖烘渶澶с€佺ǔ瀹氭€ф渶濂?- **涓皬鍥㈤槦/寮€鍙戝弸濂?*锛歍raefik
- **宸叉湁 API 缃戝叧闇€姹?*锛欿ong 鎴?APISIX
- **楂樻祦閲?鎬ц兘鍏抽敭**锛欻AProxy Ingress


# 6.TLS/HTTPS

TLS 鏄幇浠?Web 鐨勫熀纭€瀹夊叏鍔犲瘑灞傘€傚湪 Kubernetes 涓紝绠＄悊 TLS 璇佷功娑夊強 Secret 瀛樺偍銆佽嚜鍔ㄧ鍙戙€佺画鏈熴€侀厤缃瓑澶氫釜鏂归潰銆?
## 6.1 TLS Secret绠＄悊

### 6.1.1 鎵嬪姩鍒涘缓 TLS Secret

```shell
# 浣跨敤 OpenSSL 鐢熸垚鑷鍚嶈瘉涔︼紙浠呯敤浜庢祴璇曪級
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=api.example.com/O=MyOrg"

# 鍒涘缓 Kubernetes Secret
kubectl create secret tls example-tls \
  --key tls.key \
  --cert tls.crt \
  -n production
```

### 6.1.2 璇佷功閾剧殑閲嶈鎬?
TLS Secret 涓殑 `tls.crt` 蹇呴』鍖呭惈瀹屾暣鐨勮瘉涔﹂摼锛堜粠鏈嶅姟鍣ㄨ瘉涔﹀埌涓棿璇佷功鍒版牴璇佷功锛夈€備笉瀹屾暣鐨勮瘉涔﹂摼浼氬鑷撮儴鍒嗗鎴风鏃犳硶寤虹珛 TLS 杩炴帴銆?
```shell
# 妫€鏌ヨ瘉涔﹂摼鏄惁瀹屾暣
kubectl get secret example-tls -n production -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl crl2pkcs7 -nocrl -certfile /dev/stdin | openssl pkcs7 -print_certs -text -noout | grep "Subject:"
```

### 6.1.3 Secret 鐨勫懡鍚嶇┖闂撮殧绂?
TLS Secret 蹇呴』涓庡紩鐢ㄥ畠鐨?Ingress 鍦ㄥ悓涓€涓?namespace 涓嬨€傝繖鏄?Kubernetes 鐨勫畨鍏ㄨ璁♀€斺€旈伩鍏嶈法鍛藉悕绌洪棿娉勯湶璇佷功绉侀挜銆傚鏋滈渶瑕佸湪澶氫釜鍛藉悕绌洪棿浣跨敤鍚屼竴涓瘉涔︼紝闇€瑕佸湪姣忎釜鍛藉悕绌洪棿鍒涘缓涓€浠?Secret锛?
```shell
kubectl get secret example-tls -n production -o yaml \
  | sed 's/namespace: production/namespace: staging/' \
  | kubectl apply -f -
```

### 6.1.4 璇佷功鏍煎紡瑕佹眰

- **PEM 鏍煎紡**锛欱ase64 缂栫爜鐨?DER 璇佷功锛屼互 `-----BEGIN CERTIFICATE-----` 寮€澶?- **绉侀挜涓嶈兘鏈夊瘑鐮佷繚鎶?*锛欼ngress Controller 鏃犳硶鍦?reload 鏃惰緭鍏ュ瘑鐮?- **鎺ㄨ崘鐨勫瘑閽ョ畻娉?*锛欵CDSA P-256 浼樹簬 RSA 2048锛堟€ц兘鏇村ソ銆佸瘑閽ユ洿鐭級

```shell
# 鐢熸垚 ECDSA 绉侀挜鍜岃瘉涔?openssl ecparam -genkey -name prime256v1 -out tls.key
openssl req -new -sha256 -key tls.key -out csr.csr -subj "/CN=api.example.com"
openssl x509 -req -in csr.csr -signkey tls.key -out tls.crt -days 365
```

## 6.2 鑷姩璇佷功绠＄悊 (cert-manager + Let's Encrypt)

鎵嬪姩绠＄悊璇佷功锛堢壒鍒槸璇佷功鍒版湡缁湡锛夋槸杩愮淮鐨勫櫓姊︺€俢ert-manager 鏄?Kubernetes 鐢熸€佷腑鏍囧噯鐨勮瘉涔﹁嚜鍔ㄥ寲瑙ｅ喅鏂规銆?
### 6.2.1 Let's Encrypt 绠€浠?
Let's Encrypt 鏄竴涓厤璐广€佽嚜鍔ㄥ寲銆佸紑鏀剧殑 CA銆傚叾鏍稿績鍗忚鏄?**ACME锛圓utomatic Certificate Management Environment锛?*锛岄€氳繃鑷姩鍖栫殑 Challenge 楠岃瘉鍩熷悕鎵€鏈夋潈鍚庨鍙戣瘉涔︺€?
鐗圭偣锛氬厤璐广€佹湁鏁堟湡 90 澶╋紙榧撳姳鑷姩鍖栫画鏈燂級銆佹敮鎸侀€氶厤绗﹁瘉涔︼紙浠?DNS-01锛夈€侀€熺巼闄愬埗锛堟瘡涓煙鍚嶆瘡鍛ㄦ渶澶?50 涓瘉涔︼級銆?
### 6.2.2 cert-manager 瀹夎

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0 \
  --set installCRDs=true
```

### 6.2.3 閰嶇疆 Issuer 鍜?Certificate

鍦?cert-manager 涓紝鍒涘缓璇佷功鍒嗕负涓ゆ锛氬厛瀹氫箟 Issuer锛堥鍙戣€咃級锛屽啀瀹氫箟 Certificate锛堣瘉涔﹁姹傦級銆?
```yaml
# Step 1: 鍒涘缓 ClusterIssuer锛堥泦缇ょ骇鍒級
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
# Step 2: 鍒涘缓 Certificate锛堣瘉涔﹁姹傦級
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: production
spec:
  dnsNames:
    - api.example.com
    - admin.example.com
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
    group: cert-manager.io
  duration: 2160h
  renewBefore: 360h
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
```

### 6.2.4 鍦?Ingress 涓娇鐢?cert-manager

cert-manager 鎻愪緵浜?Ingress Shim 鏈哄埗锛屽彧闇€鍦?Ingress 涓婃坊鍔犳敞瑙ｅ嵆鍙嚜鍔ㄥ垱寤?Certificate 璧勬簮锛?
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    cert-manager.io/private-key-algorithm: "ECDSA"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: example-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

**Ingress Shim 鐨勫伐浣滄祦绋?*锛?1. cert-manager 妫€娴嬪埌甯︽湁娉ㄨВ鐨?Ingress
2. 鑷姩鍒涘缓 Certificate 璧勬簮锛堝寘鍚?Ingress 涓?`tls` 瀹氫箟鐨勫煙鍚嶏級
3. Certificate 璧勬簮瑙﹀彂 Challenge 姹傝В杩囩▼
4. 楠岃瘉閫氳繃鍚庯紝cert-manager 灏嗚瘉涔﹀啓鍏ユ寚瀹氱殑 Secret
5. Ingress Controller 妫€娴嬪埌 Secret 鏇存柊锛屽姞杞芥柊璇佷功

## 6.3 cert-manager鏋舵瀯

### 6.3.1 鏍稿績璧勬簮鍏崇郴

```
Issuer / ClusterIssuer    (璋侀鍙戣瘉涔?
        鈹?        鈻?   Certificate            (瑕佷粈涔堣瘉涔?
        鈹?        鈻? CertificateRequest       (CSR 璇锋眰)
        鈹?        鈻?      Order               (ACME 璁㈠崟)
        鈹?        鈻?    Challenge              (鍩熷悕楠岃瘉)
```

### 6.3.2 Issuer vs ClusterIssuer

- **Issuer**锛氬懡鍚嶇┖闂寸骇鍒€傚彧鑳介鍙戝悓涓€鍛藉悕绌洪棿鍐呯殑璇佷功銆傞€傜敤浜庡绉熸埛鐜锛屾瘡涓洟闃熺鐞嗚嚜宸辩殑 Issuer銆?- **ClusterIssuer**锛氶泦缇ょ骇鍒€傚彲浠ヨ鎵€鏈夊懡鍚嶇┖闂寸殑 Certificate 寮曠敤銆傞€傜敤浜庡叏灞€鍏变韩鐨勮瘉涔﹂鍙戣€呫€?
```yaml
# Issuer锛堝懡鍚嶇┖闂寸骇鍒級
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: team-a
spec:
  ca:
    secretName: ca-key-pair

---
# ClusterIssuer锛堥泦缇ょ骇鍒級
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
```

### 6.3.3 Certificate 鐢熷懡鍛ㄦ湡

```
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?  Issued     鈹傗攢鈹€鈫?璇佷功宸查鍙戜笖鏈夋晥
鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹?       鈹?鍒版湡鍓?renewBefore 澶╄Е鍙戠画鏈?       鈻?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹? Renewing    鈹傗攢鈹€鈫?寮€濮嬬画鏈熸祦绋?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹?       鈹?鏂拌瘉涔﹁幏鍙栨垚鍔?       鈻?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?  Issued     鈹傗攢鈹€鈫?鏂拌瘉涔︾敓鏁?(Secret 鏇存柊)
鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?       鈹?濡傛灉缁湡澶辫触
       鈻?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹? Expired     鈹傗攢鈹€鈫?璇佷功杩囨湡锛堢敓浜т簨鏁咃紒搴旀湁鍛婅锛?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

cert-manager 浼氬湪 `renewBefore` 鏃堕棿锛堥粯璁よ瘉涔﹀埌鏈熷墠 30 澶╋級瑙﹀彂缁湡銆傜画鏈熻繃绋嬩腑锛孖ngress Controller 閫氳繃 Watch 鏈哄埗妫€娴嬪埌 Secret 鏇存柊骞惰嚜鍔ㄥ姞杞芥柊璇佷功鈥斺€旀暣涓繃绋嬪畬鍏ㄨ嚜鍔紝鏃犻渶浜哄伐浠嬪叆銆?
## 6.4 HTTP-01鍜孌NS-01 Challenge

ACME 鍗忚閫氳繃 Challenge 鏉ラ獙璇佷綘瀵瑰煙鍚嶇殑鎵€鏈夋潈銆備袱绉嶆渶甯哥敤鐨?Challenge 绫诲瀷鏄?HTTP-01 鍜?DNS-01銆?
### 6.4.1 HTTP-01 Challenge

Let's Encrypt 鏈嶅姟鍣ㄥ悜 `http://<domain>/.well-known/acme-challenge/<token>` 鍙戦€佷竴涓?HTTP GET 璇锋眰锛屾湡寰呰繑鍥炰竴涓壒瀹氱殑瀵嗛挜鍊笺€?
```
Let's Encrypt 鏈嶅姟鍣?        鈹?        鈹?1. "璇疯瘉鏄庝綘鎷ユ湁 api.example.com"
        鈹?   鈫?璁块棶 http://api.example.com/.well-known/acme-challenge/xxxx
        鈹?        鈻?   鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?   鈹? Ingress Controller     鈹?鈫?cert-manager 鑷姩鍒涘缓涓存椂 Ingress 瑙勫垯
   鈹? solver Pod 杩斿洖 token   鈹?   鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?        鈹?        鈹?2. 姝ｇ‘鍝嶅簲 鈫?楠岃瘉閫氳繃
        鈻?   Let's Encrypt 棰佸彂璇佷功
```

**HTTP-01 鐨勪紭缂虹偣**锛?
| 浼樼偣 | 缂虹偣 |
|------|------|
| 閰嶇疆绠€鍗曪紝cert-manager 鑷姩澶勭悊 | 鏈嶅姟蹇呴』鍦ㄥ叕缃戝彲杈撅紙80 绔彛锛?|
| 涓嶉渶瑕?DNS API 鏉冮檺 | 涓嶆敮鎸侀€氶厤绗﹁瘉涔?|
| 鍗虫椂楠岃瘉锛岄€熷害蹇?| 濡傛湁 WAF/CDN 鍙兘闇€棰濆閰嶇疆 |

```yaml
spec:
  acme:
    solvers:
      - http01:
          ingress:
            class: nginx
```

### 6.4.2 DNS-01 Challenge

Let's Encrypt 瑕佹眰浣犲湪鍩熷悕鐨?DNS 璁板綍涓坊鍔犱竴鏉＄壒瀹氱殑 TXT 璁板綍銆?
```
Let's Encrypt 鏈嶅姟鍣?        鈹?        鈹?1. "璇峰湪 DNS 涓坊鍔?TXT 璁板綍"
        鈹?   鈫?_acme-challenge.api.example.com TXT = "gfj9Xq..."
        鈹?        鈻?   鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?   鈹? cert-manager           鈹?鈫?璋冪敤 DNS Provider API 鑷姩娣诲姞/鍒犻櫎璁板綍
   鈹? 鏀寔: Route53,         鈹?   鈹? CloudDNS, Cloudflare   鈹?   鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?        鈹?        鈹?2. DNS 璁板綍楠岃瘉閫氳繃 鈫?绛夊緟 DNS 浼犳挱锛?0s-2min锛?        鈻?   Let's Encrypt 棰佸彂璇佷功
```

**DNS-01 鐨勪紭缂虹偣**锛?
| 浼樼偣 | 缂虹偣 |
|------|------|
| 鏀寔閫氶厤绗﹁瘉涔︼紙`*.example.com`锛?| 闇€瑕?DNS Provider API 鏉冮檺 |
| 鏈嶅姟涓嶉渶瑕佸叕缃戝彲杈?| DNS 璁板綍浼犳挱鏈夊欢杩?|
| 閫傚悎鍐呯綉鐜 | 閰嶇疆姣?HTTP-01 澶嶆潅 |

```yaml
# DNS-01 Solver 閰嶇疆锛圓WS Route53 绀轰緥锛?spec:
  acme:
    solvers:
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: ZXXXXXXXXXXXXX
```

### 6.4.3 Challenge 閫夊瀷鎸囧崡

```
鏄惁闇€瑕侀€氶厤绗﹁瘉涔︼紵
鈹溾攢 鏄?鈫?DNS-01
鈹斺攢 鍚?鈫?缁х画
    鏈嶅姟鏄惁鍙互鍦ㄥ叕缃戦€氳繃 80 绔彛璁块棶锛?    鈹溾攢 鏄?鈫?HTTP-01锛堟洿绠€鍗曪級
    鈹斺攢 鍚?鈫?DNS-01
```

## 6.5 TLS鐗堟湰涓庡瘑鐮佸浠堕厤缃?
### 6.5.1 瀹夊叏鍩虹嚎

鐜颁唬 TLS 閰嶇疆搴旂鐢ㄥ凡琚敾鐮存垨涓嶅畨鍏ㄧ殑鍗忚鍜岀畻娉曪細

```yaml
# Nginx Ingress 鐨?TLS 閰嶇疆锛圕onfigMap 绾у埆锛?data:
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305"
  ssl-prefer-server-ciphers: "true"
  hsts: "true"
  hsts-max-age: "31536000"
  hsts-include-subdomains: "true"
```

### 6.5.2 TLS 1.3 鐨勪紭鍔?
TLS 1.3锛圧FC 8446锛夌浉姣?TLS 1.2 鏈夋樉钁楁敼杩涳細
- **鎻℃墜浠?2-RTT 鍑忓皯涓?1-RTT**锛堥娆¤繛鎺ワ級鎴?**0-RTT**锛堜細璇濇仮澶嶏級锛屽噺灏戜簡 50%-75% 鐨勬彙鎵嬪欢杩?- **绉婚櫎涓嶅畨鍏ㄧ畻娉?*锛氫笉鍐嶆敮鎸?RSA 瀵嗛挜浜ゆ崲銆丆BC 妯″紡銆丷C4銆丮D5銆丼HA-1 绛?- **鍓嶅悜瀹夊叏鎬т负寮哄埗**锛氭墍鏈夊瘑閽ヤ氦鎹㈤兘蹇呴』浣跨敤 ECDHE 鎴?DHE
- **鍔犲瘑鑼冨洿鎵╁ぇ**锛歍LS 1.3 鍔犲瘑浜嗘洿澶氭彙鎵嬫秷鎭紙鍖呮嫭璇佷功锛夛紝鍑忓皯淇℃伅娉勯湶

### 6.5.3 瀵嗙爜濂椾欢瑙ｆ瀽

| 瀵嗙爜濂椾欢 | 瀵嗛挜浜ゆ崲 | 璁よ瘉 | 鍔犲瘑 | 鍝堝笇 |
|---------|---------|------|------|------|
| ECDHE-RSA-AES128-GCM-SHA256 | ECDHE | RSA | AES-128-GCM | SHA-256 |
| ECDHE-ECDSA-AES256-GCM-SHA384 | ECDHE | ECDSA | AES-256-GCM | SHA-384 |
| ECDHE-RSA-CHACHA20-POLY1305 | ECDHE | RSA | ChaCha20 | Poly1305 |

- **ECDHE**锛氭き鍦嗘洸绾?Diffie-Hellman 涓存椂瀵嗛挜浜ゆ崲锛屾彁渚涘墠鍚戝畨鍏ㄦ€?- **AES-GCM / CHACHA20-POLY1305**锛氳璇佸姞瀵嗭紙AEAD锛夌畻娉曪紝鏁堢巼楂樹笖瀹夊叏

### 6.5.4 Mozilla SSL Configuration Generator

鎺ㄨ崘浣跨敤 Mozilla 鐨?SSL Configuration Generator 鏉ヨ幏鍙栫幇浠?TLS 閰嶇疆銆備笁涓骇鍒細
- **Modern**锛氫粎 TLS 1.3锛屽吋瀹规€ф渶浣?- **Intermediate**锛歍LS 1.2 鍜?1.3锛屾帹鑽愮敓浜т娇鐢?- **Old**锛歍LS 1.0+锛屽吋瀹硅€佽澶?

# 7. Gateway API

Gateway API 鏄?Kubernetes SIG-Network 绀惧尯鎺ㄥ嚭鐨勪笅涓€浠ｅ叆鍙?API锛屾棬鍦ㄦ浛浠ｅ拰瓒呰秺 Ingress銆傚畠涓嶄粎浠呮槸銆孖ngress v2銆嶏紝鑰屾槸涓€娆″鍏ュ彛娴侀噺绠＄悊妯″瀷鐨勫叏闈㈤噸鏂版€濊€冦€?
## 7.1 Ingress鐨勫眬闄愭€?
鍦ㄦ繁鍏?Gateway API 涔嬪墠锛屽厛鐞嗚В瀹冭瑙ｅ喅鐨勫叿浣撻棶棰樸€?
### 7.1.1 娉ㄨВ娣蜂贡锛圓nnotation Hell锛?
Ingress 鐨勬牳蹇冮棶棰樻槸鏍囧噯瀛楁澶皯锛屽嚑涔庢墍鏈夌殑鎵╁睍鍔熻兘閮戒緷璧栨敞瑙ｏ紙Annotations锛夈€傝繖瀵艰嚧锛?
- **Controller 寮虹粦瀹?*锛歚nginx.ingress.kubernetes.io/rewrite-target` 鎸埌 Traefik 灏卞け鏁堬紝闇€瑕佹敼涓?`traefik.ingress.kubernetes.io/rewrite-target`
- **缂轰箯楠岃瘉**锛氭敞瑙ｆ槸鑷敱鏂囨湰锛坄map[string]string`锛夛紝娌℃湁 schema 楠岃瘉銆傛嫾閿欎竴涓敞瑙ｅ悕锛孠ubernetes API Server 涓嶄細鎶ラ敊鈥斺€斾綘鍙兘鍦ㄨ姹傝繑鍥?404 鎴?502 涔嬪悗鎵嶅彂鐜?- **鏃犳硶澶嶇敤**锛氬湪姣忎釜 Ingress 鐨勬敞瑙ｉ噷閲嶅閰嶇疆鐩稿悓鐨勯檺娴佸弬鏁版槸甯告€侊紝DRY 鍘熷垯琚牬鍧?- **瀹夊叏椋庨櫓**锛氬ぇ閮ㄥ垎 Ingress Controller 鏀寔 `configuration-snippet` / `server-snippet` 娉ㄨВ锛屽厑璁哥敤鎴锋敞鍏ュ師濮?Nginx 閰嶇疆銆傝繖缁欎簡寮€鍙戣€呰繃楂樼殑鏉冮檺鈥斺€斾竴涓敊璇殑 `location` 鍧楀彲鑳藉奖鍝嶆暣涓叆鍙ｇ殑绋冲畾鎬?
### 7.1.2 缂轰箯瑙掕壊鍒嗙锛圠ack of Role-Oriented Design锛?
鍦ㄤ竴涓粍缁囦腑锛屼笉鍚岃鑹插鍏ュ彛閰嶇疆鏈変笉鍚岀殑瑙嗚鍜屾潈闄愰渶姹傦細

| 瑙掕壊 | 鍏虫敞鐐?| Ingress 鑳藉惁鏀寔锛?|
|------|--------|------------------|
| 闆嗙兢绠＄悊鍛?| LB 绫诲瀷銆佺鍙ｃ€乀LS 鍏ㄥ眬绛栫暐 | 閮ㄥ垎锛圕onfigMap 鍏ㄥ眬閰嶇疆锛?|
| 搴旂敤寮€鍙戣€?| 璺敱瑙勫垯锛坔ost + path 鈫?Service锛?| 鏄紙Ingress 鏍稿績鍔熻兘锛?|
| 瀹夊叏鍥㈤槦 | TLS 鐗堟湰銆佸瘑鐮佸浠躲€乄AF 绛栫暐 | 鍚︼紙渚靛叆搴旂敤 Ingress 鐨勬敞瑙ｏ級 |
| 骞冲彴/SRE | 闄愭祦銆侀噾涓濋泙銆佹祦閲忛暅鍍?| 鍚︼紙闇€淇敼搴旂敤 Ingress 鐨勬敞瑙ｏ級 |

鍦?Ingress 妯″瀷涓紝鎵€鏈夎繖浜涜亴璐ｉ兘琚帇缂╄繘鍚屼竴涓?Ingress YAML 鏂囦欢鐨勫悓涓€涓?`.metadata.annotations` 鍧椼€傝繖瀹屽叏杩濊儗浜?DevSecOps 鐨勮鑹插垎宸ュ師鍒欍€?
### 7.1.3 浠呴檺 HTTP/HTTPS锛圠7 Only锛?
鏍囧噯 Ingress 鍙敮鎸?HTTP 鍜?HTTPS 鍗忚銆傚浜?TCP銆乁DP銆乬RPC 绛夊崗璁殑鍏ュ彛绠＄悊锛岄渶瑕佸€熷姪锛?- Nginx Ingress Controller 鐨?`tcp-services` / `udp-services` ConfigMap
- Traefik 鐨?`IngressRouteTCP` / `IngressRouteUDP` CRD

杩欎簺鏂规閮芥槸 Controller 涓撴湁鐨勶紝娌℃湁鏍囧噯鍖栥€?
### 7.1.4 涓嶈兘璺ㄥ懡鍚嶇┖闂村叡浜?
Ingress 鍙兘寮曠敤鍚屼竴鍛藉悕绌洪棿鐨?Service銆傚鏋滃悗绔?Service 鍦?`namespace-a`锛岃€屼綘鎯冲湪 `namespace-b` 鍒涘缓 Ingress锛屼綘闇€瑕侊細
- 鍒涘缓 ExternalName Service 鍋氳法鍛藉悕绌洪棿寮曠敤锛坔ack锛?- 鎴栬€呭湪姣忎釜鍛藉悕绌洪棿鍒涘缓閲嶅鐨?Ingress 瑙勫垯锛堢淮鎶ゅ櫓姊︼級

### 7.1.5 缂轰箯楂樼骇娴侀噺绠＄悊

Ingress 鍘熺敓涓嶆敮鎸侊細
- 娴侀噺鍒嗗壊锛堟寜鏉冮噸鍒嗗彂鍒板涓?Service锛?- 娴侀噺闀滃儚锛堝鍒朵竴浠芥祦閲忓埌瑙傛祴鏈嶅姟锛?- 璇锋眰澶村尮閰嶈矾鐢?- 璇锋眰/鍝嶅簲淇敼锛圚eader 澧炲垹鏀癸級
- 鍚庣鍋ュ悍妫€鏌ョ瓥鐣ュ畾鍒?
杩欎簺鑳藉姏鍙兘閫氳繃鍚?Controller 鐨勪笓鏈夋敞瑙ｅ疄鐜帮紝涓嶅叿澶囧彲绉绘鎬с€?
## 7.2 Gateway API璁捐鐞嗗康

### 7.2.1 闈㈠悜瑙掕壊锛圧ole-Oriented锛?
杩欐槸 Gateway API 鏈€閲嶈鐨勮璁″垱鏂般€傚畠灏嗗叆鍙ｉ厤缃媶鍒嗕负澶氫釜鐙珛鐨勮祫婧愬璞★紝姣忎釜瑙掕壊璐熻矗鑷繁鐨勯儴鍒嗭細

```
Cluster Operator (闆嗙兢绠＄悊鍛?
        鈹?        绠＄悊 GatewayClass + Gateway
        鈹?        鈻?Application Developer (搴旂敤寮€鍙戣€?
        鈹?        绠＄悊 HTTPRoute (缁戝畾鍒?Gateway)
        鈹?        鈻?    涓よ€呯嫭绔嬶紝浜掍笉骞叉壈
```

- **GatewayClass**锛氱敱鍩虹璁炬柦鎻愪緵鍟嗙鐞嗭紙浜戝巶鍟嗐€侀泦缇ょ鐞嗗憳锛夈€傚畾涔夊彲鐢ㄧ殑 Gateway 瀹炵幇绫诲瀷锛堝 `nginx-gateway`銆乣istio`銆乣contour`锛?- **Gateway**锛氱敱闆嗙兢绠＄悊鍛樼鐞嗐€傚畾涔夊叿浣撶殑鍏ュ彛鐐癸紙鐩戝惉绔彛銆佸崗璁€乀LS 绛栫暐銆佸厑璁哥殑鍛藉悕绌洪棿锛?- **HTTPRoute / TLSRoute 绛?*锛氱敱搴旂敤寮€鍙戣€呯鐞嗐€傚畾涔夎矾鐢辫鍒欙紙鍩熷悕銆佽矾寰勩€佸悗绔?Service銆佹祦閲忔潈閲嶇瓑锛?
```yaml
# 闆嗙兢绠＄悊鍛樺垱寤?Gateway锛堝熀纭€璁炬柦灞傦級
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: infra
spec:
  gatewayClassName: nginx-gateway
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway: allowed
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: example-tls
            namespace: infra
      allowedRoutes:
        namespaces:
          from: All

---
# 搴旂敤寮€鍙戣€呭垱寤?HTTPRoute锛堝簲鐢ㄥ眰锛?apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-routes
  namespace: app-team-a
spec:
  parentRefs:
    - name: production-gateway
      namespace: infra          # 鍙互寮曠敤鍏朵粬鍛藉悕绌洪棿鐨?Gateway锛?  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /users
      backendRefs:
        - name: user-service
          port: 80
```

### 7.2.2 鍙Щ妞嶏紙Portable锛?
Gateway API 鑷村姏浜庢爣鍑嗗寲鎵€鏈夊叆鍙ｉ厤缃紝璁╁悓涓€浠借矾鐢遍厤缃彲浠ュ湪涓嶅悓瀹炵幇涔嬮棿杩佺Щ鑰屼笉闇€瑕佷慨鏀广€傛爣鍑嗗瓧娈电殑璇箟缁忚繃涓ユ牸瀹氫箟锛岃涓哄湪涓嶅悓瀹炵幇涔嬮棿淇濇寔涓€鑷淬€?
### 7.2.3 鍙墿灞曪紙Extensible锛?
Gateway API 浣跨敤 Kubernetes 鎯敤鐨?API 妯″紡锛圕RD + API Groups锛夋潵鏀寔鎵╁睍锛?- **Policy Attachment**锛氬皢绛栫暐锛堥檺娴併€丆ORS銆佽璇侊級浣滀负鐙珛璧勬簮闄勫姞鍒?Gateway 鎴?Route锛岃€岄潪娉ㄨВ
- **BackendRef 鎵╁睍**锛氫笉浠呮敮鎸?Service锛岃繕鍙互寮曠敤浠绘剰 Kubernetes 璧勬簮浣滀负鍚庣
- **Route 绫诲瀷鎵╁睍**锛氶櫎浜?HTTPRoute锛岃繕瀹氫箟浜?TLSRoute銆乀CPRoute銆乁DPRoute銆丟RPCRoute

## 7.3 鏍稿績璧勬簮妯″瀷 (GatewayClass/Gateway/HTTPRoute)

### 7.3.1 GatewayClass

GatewayClass 鐩稿綋浜?IngressClass锛屽畾涔変簡鍙敤鐨?Gateway 瀹炵幇绫诲瀷銆傚畠閫氬父鏄熀纭€璁炬柦鎻愪緵鑰呭垱寤哄拰绠＄悊鐨勩€?
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
  description: "NGINX Gateway Fabric - production grade"
  parametersRef:
    group: gateway.nginx.org
    kind: NginxGateway
    name: nginx-gateway-config
```

GatewayClass 瀹氫箟浜嗭細
- **鎺у埗鍣ㄥ悕绉?*锛氬敮涓€鏍囪瘑涓€涓?Gateway 瀹炵幇
- **鍙傛暟寮曠敤**锛氭寚鍚戝疄鐜扮壒瀹氱殑閰嶇疆璧勬簮锛堝 ConfigMap銆丆RD锛?- **鐘舵€?*锛氭爣璁?GatewayClass 鏄惁琚帴鍙楋紙Accepted锛?
### 7.3.2 Gateway

Gateway 鏄敱闆嗙兢绠＄悊鍛樺畾涔夌殑鍏ュ彛鍩虹璁炬柦銆傛瘡涓?Gateway 浠ｈ〃涓€涓叿浣撶殑娴侀噺鍏ュ彛鐐广€?
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: infra
spec:
  gatewayClassName: nginx-gateway
  addresses:
    - type: IPAddress
      value: 203.0.113.10
  listeners:
    - name: http-listener
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              shared-gateway: "true"
    - name: https-listener
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: production-tls-secret
      allowedRoutes:
        namespaces:
          from: Same
    - name: tcp-listener
      port: 5432
      protocol: TCP
      allowedRoutes:
        namespaces:
          from: All
```

**Listener 璇﹁В**锛?- `protocol`锛氭敮鎸?HTTP銆丠TTPS銆乀LS銆乀CP銆乁DP
- `tls.mode`锛?  - `Terminate`锛歍LS 缁堢粨鍦?Gateway锛屽悗绔娇鐢?HTTP
  - `Passthrough`锛歍LS 绌块€忓埌鍚庣 Pod锛岀敱搴旂敤鑷繁澶勭悊
- `allowedRoutes`锛氭帶鍒跺摢浜涘懡鍚嶇┖闂寸殑 Route 鍙互缁戝畾鍒版 Listener
  - `Same`锛氬彧鍏佽 Gateway 鎵€鍦ㄥ懡鍚嶇┖闂?  - `All`锛氬厑璁告墍鏈夊懡鍚嶇┖闂?  - `Selector`锛氬彧鍏佽甯︾壒瀹?label 鐨勫懡鍚嶇┖闂?
### 7.3.3 HTTPRoute

HTTPRoute 鏄簲鐢ㄥ紑鍙戣€呭垱寤虹殑璺敱瑙勫垯锛屽畠灏?HTTP/HTTPS 璇锋眰鏄犲皠鍒板悗绔湇鍔°€?
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-routes
  namespace: app-team-a
spec:
  parentRefs:
    - name: production-gateway
      namespace: infra
      sectionName: https-listener
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /users
          headers:
            - name: x-version
              value: v2
          queryParams:
            - name: debug
              value: "true"
          method: GET
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: x-custom-header
                value: "added-by-gateway"
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            set:
              - name: x-response-time
                value: "cached"
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v2
      backendRefs:
        - name: user-service-v2
          port: 80
          weight: 90
        - name: user-service-v2-canary
          port: 80
          weight: 10
```

## 7.4 HTTPRoute璇﹁В

### 7.4.1 璺敱鍖归厤瑙勫垯锛圡atches锛?
HTTPRoute 鏀寔涓板瘜鐨勫尮閰嶆潯浠讹紝涓旀敮鎸佸鏉′欢 AND 缁勫悎锛?
```yaml
rules:
  - matches:
      # 璺緞鍖归厤
      - path:
          type: Exact          # 绮剧‘鍖归厤
          value: /healthz
      - path:
          type: PathPrefix     # 鍓嶇紑鍖归厤锛堟渶甯哥敤锛?          value: /api/v1/
      - path:
          type: RegularExpression  # 姝ｅ垯鍖归厤
          value: "^/api/v[0-9]+/.*"

      # 璇锋眰澶村尮閰?      - headers:
          - name: Content-Type
            value: application/json    # 绮剧‘鍖归厤

      # 鏌ヨ鍙傛暟鍖归厤
      - queryParams:
          - name: api-version
            value: "1"
          - name: debug
            value: ""              # 鍙鏌ュ瓨鍦紝涓嶆鏌ュ€?
      # HTTP 鏂规硶鍖归厤
      - method: GET
      - method: POST

      # 缁勫悎绀轰緥
      - path:
          type: PathPrefix
          value: /webhooks
        method: POST
        headers:
          - name: X-Hub-Signature
```

**鍖归厤浼樺厛绾?*锛?1. 绮剧‘鍖归厤锛圗xact锛変紭鍏堜簬鍓嶇紑鍖归厤锛圥athPrefix锛変紭鍏堜簬姝ｅ垯鍖归厤
2. 鍦ㄥ悓涓€浼樺厛绾у唴锛岃緝闀跨殑鍖归厤浼樺厛
3. 濡傛灉澶氫釜瑙勫垯鍚屾椂鍖归厤锛岀涓€涓鍒欑敓鏁?
### 7.4.2 杩囨护鍣紙Filters锛?
杩囨护鍣ㄥ湪璇锋眰鍖归厤鎴愬姛鍚庛€佽浆鍙戝埌鍚庣涔嬪墠鎵ц銆?
**RequestHeaderModifier**锛?
```yaml
filters:
  - type: RequestHeaderModifier
    requestHeaderModifier:
      set:
        - name: X-Custom-Header
          value: "set-value"
      add:
        - name: X-Forwarded-For
          value: "added-value"
      remove:
        - X-Internal-Token
```

**ResponseHeaderModifier**锛?
```yaml
filters:
  - type: ResponseHeaderModifier
    responseHeaderModifier:
      set:
        - name: X-Response-Time
          value: "custom"
```

**RequestRedirect**锛?
```yaml
filters:
  - type: RequestRedirect
    requestRedirect:
      scheme: https
      hostname: new.example.com
      statusCode: 301
```

**URLRewrite**锛?
```yaml
filters:
  - type: URLRewrite
    urlRewrite:
      hostname: internal-api.example.com
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /v2
```

**RequestMirror**锛?
```yaml
filters:
  - type: RequestMirror
    requestMirror:
      backendRef:
        name: traffic-analyzer-service
        port: 80
```

### 7.4.3 娴侀噺鍒嗗壊锛圔ackendRefs + Weight锛?
Gateway API 鍘熺敓鏀寔鎸夋潈閲嶅皢娴侀噺鍒嗗彂鍒板涓悗绔紝鏃犻渶浠讳綍娉ㄨВ锛?
```yaml
backendRefs:
  - name: user-service-stable
    port: 80
    weight: 90
  - name: user-service-canary
    port: 80
    weight: 10
```

鏉冮噸鎬诲拰涓嶉渶瑕佺瓑浜?100銆俙weight: 1` 鍜?`weight: 1` 绛変环浜庡悇 50%銆?
## 7.5 鍏朵粬Route绫诲瀷

### 7.5.1 TLSRoute

鐢ㄤ簬 TCP + TLS 鐨勬祦閲忚矾鐢憋紝鍙互瀹炵幇 SNI 璺敱锛?
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: tls-routes
spec:
  parentRefs:
    - name: production-gateway
      sectionName: tls-listener
  hostnames:
    - "*.example.com"
  rules:
    - backendRefs:
        - name: tls-backend-service
          port: 443
```

### 7.5.2 TCPRoute

绾洓灞?TCP 娴侀噺璺敱锛?
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: postgres-route
spec:
  parentRefs:
    - name: production-gateway
      sectionName: postgres-listener
  rules:
    - backendRefs:
        - name: postgres-service
          port: 5432
```

### 7.5.3 UDPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: UDPRoute
metadata:
  name: dns-route
spec:
  parentRefs:
    - name: production-gateway
      sectionName: dns-listener
  rules:
    - backendRefs:
        - name: coredns-service
          port: 53
```

### 7.5.4 GRPCRoute

gRPC 涓撶敤鐨勮矾鐢憋紝鏀寔鍩轰簬 gRPC 鏂规硶鐨勮矾鐢憋細

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: GRPCRoute
metadata:
  name: grpc-routes
spec:
  parentRefs:
    - name: production-gateway
  hostnames:
    - grpc.example.com
  rules:
    - matches:
        - method:
            service: user.UserService
            method: GetUser
      backendRefs:
        - name: user-grpc-service
          port: 50051
    - matches:
        - method:
            service: order.OrderService
      backendRefs:
        - name: order-grpc-service
          port: 50052
```

## 7.6 Gateway API vs Ingress 瀵规瘮

### 7.6.1 鍏ㄩ潰瀵规瘮

| 缁村害 | Ingress | Gateway API |
|------|---------|-------------|
| API 鐗堟湰 | `networking.k8s.io/v1` (GA) | `gateway.networking.k8s.io/v1` (GA) |
| 鍗忚鏀寔 | HTTP, HTTPS | HTTP, HTTPS, TLS, TCP, UDP, gRPC |
| 瑙掕壊鍒嗙 | 鏃狅紙鎵€鏈夐厤缃湪鍚屼竴涓祫婧愶級 | GatewayClass/Gateway/脳Route 涓夊眰鍒嗙 |
| 鎵╁睍鏈哄埗 | Annotations锛堣嚜鐢辨枃鏈級 | Policy Attachment锛堢粨鏋勫寲 CRD锛?|
| 璺ㄥ懡鍚嶇┖闂磋矾鐢?| 涓嶆敮鎸?| 鏀寔 |
| 娴侀噺鍒嗗壊 | 闇€娉ㄨВ | 鍘熺敓鏀寔 weight |
| 娴侀噺闀滃儚 | 闇€娉ㄨВ | 鍘熺敓鏀寔 RequestMirror filter |
| 璇锋眰澶翠慨鏀?| 闇€娉ㄨВ | 鍘熺敓鏀寔 RequestHeaderModifier |
| URL 閲嶅啓 | 闇€娉ㄨВ | 鍘熺敓鏀寔 URLRewrite filter |
| 閲嶅畾鍚?| 闇€娉ㄨВ | 鍘熺敓鏀寔 RequestRedirect filter |
| 澶村尮閰嶈矾鐢?| 闇€娉ㄨВ | 鍘熺敓鏀寔 headers match |
| 鏌ヨ鍙傛暟璺敱 | 涓嶆敮鎸?| 鍘熺敓鏀寔 queryParams match |
| 鏂规硶璺敱 | 涓嶆敮鎸?| 鍘熺敓鏀寔 method match |
| 鍙Щ妞嶆€?| 浣庯紙娉ㄨВ缁戝畾瀹炵幇锛?| 楂橈紙鏍囧噯瀛楁璺ㄥ疄鐜颁竴鑷达級 |

### 7.6.2 杩佺Щ绛栫暐

涓嶈鎬ヤ簬浠?Ingress 鍏ㄩ儴杩佺Щ鍒?Gateway API銆傛帹鑽愮殑娓愯繘寮忚縼绉荤瓥鐣ワ細

**绗竴闃舵锛氬苟瀛?*锛氫繚鎸佺幇鏈?Ingress 涓嶅彉锛屾柊鏈嶅姟浣跨敤 Gateway API銆備袱涓郴缁熷彲浠ュ叡瀛樹簬鍚屼竴闆嗙兢锛堝崰鐢ㄤ笉鍚岀鍙ｏ級銆?
**绗簩闃舵锛氳繃娓?*锛氬皢鎴愮啛鐨?Ingress 鈫?Gateway API 杞崲銆備娇鐢?`kubectl ingress2gateway` 宸ュ叿鑷姩杞崲銆?
**绗笁闃舵锛氬畬鍏ㄨ縼绉?*锛氭墍鏈夊叆鍙ｆ祦閲忓垏鎹㈠埌 Gateway API锛岀Щ闄ゆ棫鐨?Ingress 璧勬簮鍜?Controller銆?
## 7.7 Gateway API鐢熸€佷笌瀹炵幇

### 7.7.1 涓昏瀹炵幇

| 瀹炵幇 | 缁存姢鏂?| 鐗圭偣 |
|------|--------|------|
| **NGINX Gateway Fabric** | F5/NGINX Inc | NGINX Plus 鏁版嵁闈紝鍟嗕笟鍖栨敮鎸?|
| **Envoy Gateway** | Envoy 绀惧尯 | 鍩轰簬 Envoy Proxy锛孠ubernetes-native |
| **Istio** | Google/Solo.io | 鍐呯疆 Gateway API 鏀寔锛孲ervice Mesh + Ingress 涓€浣撳寲 |
| **Contour** | VMware | 鍩轰簬 Envoy锛屾棭鏈?Gateway API 閲囩敤鑰?|
| **Cilium** | Isovalent/Cisco | eBPF-based锛岄珮鎬ц兘 |
| **HAProxy Ingress** | HAProxy Technologies | 鍩轰簬 HAProxy锛屾€ц兘鍗撹秺 |
| **Traefik** | Traefik Labs | 姝ｅ湪閫傞厤 Gateway API |
| **APISIX Ingress** | Apache | 鍩轰簬 APISIX 缃戝叧 |

### 7.7.2 褰撳墠鐘舵€佷笌閫夊瀷

鎴嚦 2024 骞达紝Gateway API v1.0 宸茬粡 GA锛屽悇瀹炵幇鐨勬垚鐔熷害涓嶄竴锛?
- **Istio** 鍜?**Contour** 瀵?Gateway API 鐨勬敮鎸佹渶涓烘垚鐔?- **NGINX Gateway Fabric** 鍜?**Envoy Gateway** 姝ｅ湪蹇€熻拷璧?- **Traefik** 鍜?**HAProxy** 鐨勯儴鍒嗗疄鐜颁粛澶勪簬瀹為獙闃舵

**鐢熶骇鐜寤鸿**锛?- 濡傛灉浣跨敤 Service Mesh锛堝 Istio锛夛紝鐩存帴浣跨敤 Gateway API
- 濡傛灉浣跨敤 Envoy 鐢熸€侊紙Contour锛夛紝Gateway API 鏄閫?- 濡傛灉鐩墠浣跨敤 Nginx Ingress 涓旂ǔ瀹氳繍琛岋紝鍙户缁娇鐢ㄥ苟鍏虫敞 NGINX Gateway Fabric 杩涘睍
- 鏂板缓椤圭洰灏介噺浣跨敤 Gateway API锛屼负鏈潵鐨勬爣鍑嗗寲鍜屽彲绉绘鎬у仛鍑嗗


# 8. 閲戜笣闆€鍙戝竷涓庢祦閲忕鐞?
閲戜笣闆€鍙戝竷锛圕anary Deployment锛夋槸涓€绉嶅皢灏戦噺鐢熶骇娴侀噺寮曞鍒版柊鐗堟湰搴旂敤鐨勫彂甯冪瓥鐣ワ紝鏈€鏃╁緱鍚嶄簬銆岀叅鐭夸腑鐨勯噾涓濋泙銆嶁€斺€旂敤灏戞暟鏁忔劅涓綋鏉ユ帰娴嬪嵄闄┿€傚湪 Kubernetes 涓紝閲戜笣闆€鍙戝竷鍙互鍦ㄥ涓眰闈㈠疄鐜帮細Ingress/鍏ュ彛灞傘€丼ervice Mesh 灞傘€佷互鍙婂簲鐢ㄩ儴缃插眰銆?
## 8.1 Ingress灞傚疄鐜伴噾涓濋泙

### 8.1.1 Nginx Ingress Canary 娉ㄨВ

Nginx Ingress Controller 閫氳繃涓€缁勪笓闂ㄧ殑 `canary` 娉ㄨВ瀹炵幇閲戜笣闆€鍙戝竷銆傚熀鏈ā寮忔槸锛氬垱寤轰竴涓笌涓?Ingress 瀵瑰簲鐨勩€岄噾涓濋泙 Ingress銆嶏紝閫氳繃娉ㄨВ鏍囪瘑鍏堕噾涓濋泙韬唤銆?
```yaml
# 涓?Ingress锛堢ǔ瀹氱増鏈級
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-service
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service-stable
                port:
                  number: 80

---
# 閲戜笣闆€ Ingress锛堟柊鐗堟湰锛?apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-service-canary
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-service-canary
                port:
                  number: 80
```

**宸ヤ綔鍘熺悊**锛歂ginx Ingress Controller 鍦ㄧ敓鎴?`nginx.conf` 鏃讹紝妫€娴嬪埌涓や釜鍏变韩鐩稿悓 host 鍜?path 鐨?Ingress锛屽叾涓竴涓甫鏈?`canary: "true"`銆傚畠灏嗗畠浠悎骞朵负涓€涓?`location` 鍧楋紝鍐呴儴浣跨敤 Nginx 鐨?`split_clients` 鎴?`if` 鎸囦护鎸夋潈閲嶅垎鍙戞祦閲忋€?
### 8.1.2 Canary 娴侀噺鍒嗗彂绛栫暐

**鎸夋潈閲嶅垎娴?*锛堟渶甯哥敤锛夛細

```yaml
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-weight: "10"    # 鏁存暟 0-100
```

**鎸夎姹傚ご鍒嗘祦**锛?
```yaml
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-by-header: "x-canary"
nginx.ingress.kubernetes.io/canary-by-header-value: "always"
```

**鎸?Cookie 鍒嗘祦**锛?
```yaml
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-by-cookie: "canary_user"
```

**缁勫悎绛栫暐**锛氫笉鍚岀瓥鐣ュ彲浠ョ粍鍚堜娇鐢紙浼樺厛绾э細鏉冮噸 < 璇锋眰澶?< Cookie锛夛紝浣嗗悓涓€鏃堕棿鍙簲鍚敤涓€绉嶃€?
### 8.1.3 Ingress Canary 鐨勫眬闄愭€?
- **鍙兘娴侀噺鍒嗗壊**锛氫笉鑳藉疄鐜版洿澶嶆潅鐨勫彂甯冪瓥鐣ワ紙濡傚熀浜庡欢杩熺殑鑷姩鍥炴粴锛?- **涓?Deployment 鏃犺€﹀悎**锛欼ngress 灞傜殑閲戜笣闆€涓嶄笌 Kubernetes Deployment 鐨勬粴鍔ㄦ洿鏂扮瓥鐣ヨ仈鍔紝闇€瑕侀澶栫殑缂栨帓
- **澶氬壇鏈竴鑷存€?*锛氶噾涓濋泙 Ingress 鍜屼富 Ingress 鐨?host/path 蹇呴』涓ユ牸鍖归厤
- **涓嶆敮鎸佹洿缁嗙矑搴︾殑鎺у埗**锛氭棤娉曞熀浜庤姹傚唴瀹癸紙濡傚湴鍩熴€佺敤鎴风瓑绾э級鍋氭洿绮剧粏鐨勫垎娴?
## 8.2 Gateway API娴侀噺鍒嗗壊

Gateway API 鍘熺敓鏀寔娴侀噺鍒嗗壊锛屾棤闇€娉ㄨВ锛岀洿鎺ラ€氳繃 `backendRefs` 鐨?`weight` 瀛楁瀹炵幇銆?
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: user-service-route
spec:
  parentRefs:
    - name: production-gateway
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /users
      backendRefs:
        - name: user-service-stable
          port: 80
          weight: 90
        - name: user-service-canary
          port: 80
          weight: 10
```

**Gateway API 娴侀噺鍒嗗壊鐨勪紭鐐?*锛?- **澹版槑寮?*锛氭墍鏈夐厤缃湪鍚屼竴涓?HTTPRoute 涓紝涓嶇敤鍒涘缓涓や釜 Ingress
- **鏍囧噯鍖?*锛氫笉绠′娇鐢ㄥ摢涓?Gateway 瀹炵幇锛宍weight` 鐨勮涔変竴鑷?- **鍙粍鍚?*锛氭祦閲忓垎鍓插彲浠ヤ笌澶村尮閰嶃€佽繃婊ゅ櫒绛夌粍鍚堜娇鐢?
```yaml
# 楂樼骇绀轰緥锛氭寜鐢ㄦ埛浠ｇ悊 + 鏉冮噸缁勫悎
rules:
  - matches:
      - headers:
          - name: User-Agent
            value: "Mobile"
    backendRefs:
      - name: user-service-stable
        port: 80
        weight: 80
      - name: user-service-canary
        port: 80
        weight: 20
  - matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: user-service-stable
        port: 80
        weight: 100
```

## 8.3 涓嶢rgo Rollouts/Istio閰嶅悎

### 8.3.1 Argo Rollouts

Argo Rollouts 鏄竴涓?Kubernetes Controller锛屼负 Deployment 鎻愪緵浜嗚摑缁垮彂甯冨拰閲戜笣闆€鍙戝竷鑳藉姏銆?
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: user-service
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 30
        - pause: {duration: 10m}
        - setWeight: 100
      trafficRouting:
        nginx:
          stableIngress: user-service
          canaryIngress: user-service-canary
```

**Argo Rollouts 宸ヤ綔娴?*锛?1. 鍒涘缓鏂扮殑 ReplicaSet
2. 鎸夊畾涔夌殑 steps 閫愭澧炲姞閲戜笣闆€ Ingress 鐨勬潈閲?3. 鍦ㄦ瘡涓?pause 姝ラ涓紝鐩戞帶 metrics锛堝 Prometheus锛?4. 濡傛灉 metrics 寮傚父锛岃嚜鍔ㄥ洖婊氾紙灏嗘潈閲嶉檷涓?0锛?5. 濡傛灉鎵€鏈夋楠ら€氳繃锛岄噾涓濋泙鐗堟湰鍙樹负鏂扮殑绋冲畾鐗堟湰

### 8.3.2 閲戜笣闆€鍙戝竷鏈€浣冲疄璺?
**娓愯繘寮忔潈閲嶅闀?*锛?```
10% 鈫?瑙傚療 5 鍒嗛挓 鈫?30% 鈫?瑙傚療 10 鍒嗛挓 鈫?50% 鈫?瑙傚療 30 鍒嗛挓 鈫?100%
```

**鍏抽敭鐩戞帶鎸囨爣**锛?- HTTP 5xx 閿欒鐜囷紙閲戜笣闆€鐗堟湰 vs 绋冲畾鐗堟湰锛?- 鍝嶅簲寤惰繜 p50/p95/p99
- 搴旂敤绾у埆鐨勪笟鍔℃寚鏍囷紙璁㈠崟鎴愬姛鐜囥€佹敮浠樻垚鍔熺巼绛夛級
- 璧勬簮浣跨敤鐜囷紙CPU銆佸唴瀛樸€丟C锛?
**鍥炴粴绛栫暐**锛?- 鑷姩鍥炴粴锛氬熀浜?metrics 鐨勯槇鍊艰Е鍙戯紙濡?5xx 閿欒鐜囪秴杩?1% 绔嬪嵆鍥炴粴锛?- 鎵嬪姩鍥炴粴锛氶€氳繃 kubectl 鎴?GitOps 灏嗘潈閲嶉檷涓?0
- 鍥炴粴鍚庣殑浜嬪悗鍒嗘瀽锛氫繚鐣欓噾涓濋泙鐗堟湰鐨勬棩蹇楀拰鎸囨爣锛屽畾浣嶉棶棰樻牴鍥?
# 9.甯歌闂涓庢帓鏌?
鐢熶骇鐜涓紝Ingress 鐩稿叧鐨勬晠闅滃線寰€鐩存帴瀵艰嚧鐢ㄦ埛鏃犳硶璁块棶鏈嶅姟鈥斺€旇繖灏辨槸銆屽叆鍙ｆ晠闅溿€嶇骇鍒殑閲嶅ぇ浜嬫晠銆備互涓嬫槸鏈€甯歌鐨勫嚑绫婚棶棰樺強鍏剁郴缁熷寲鎺掓煡鏂规硶銆?
## 9.1 404/503閿欒鎺掓煡

### 9.1.1 404 Not Found

404 琛ㄧず璇锋眰鍒拌揪浜?Ingress Controller锛屼絾娌℃湁鍖归厤鍒颁换浣曡矾鐢辫鍒欐垨鍚庣 Service銆?
**鎺掓煡娓呭崟**锛?
```shell
# 1. 妫€鏌?Ingress 鏄惁瀛樺湪
kubectl get ingress -n <namespace>

# 2. 鏌ョ湅 Ingress 璇︾粏閰嶇疆
kubectl describe ingress <ingress-name> -n <namespace>

# 3. 妫€鏌?host 鍜?path
kubectl get ingress <name> -n <namespace> -o yaml | grep -A 10 "rules:"

# 4. 妫€鏌ュ悗绔?Service
kubectl get svc <service-name> -n <namespace>

# 5. 鏌ョ湅 Controller 鏃ュ織
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100

# 6. 妫€鏌ョ敓鎴愮殑 nginx.conf
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- cat /etc/nginx/nginx.conf | grep -A 20 "server_name <host>"
```

**甯歌鍘熷洜**锛?
| 鍘熷洜 | 鐜拌薄 | 瑙ｅ喅鏂规 |
|------|------|---------|
| host 涓嶅尮閰?| 鍥炲埌榛樿 server 杩斿洖 404 | 纭 Host 澶翠笌 Ingress rules 涓€鑷?|
| path 涓嶅尮閰?| 璺緞 `/v1/users` 浣嗚鍒欏彧鏈?`/v2/` | 妫€鏌?path 鍜?pathType |
| Service 涓嶅瓨鍦?| Ingress 鎸囧悜鐨?Service 鏈垱寤?| `kubectl get svc` 妫€鏌?|
| Service 绔彛涓嶅尮閰?| 绔彛鍙蜂笌 Service 瀹氫箟涓嶄竴鑷?| 鏍稿绔彛鍙?|
| 鍛藉悕绌洪棿闅旂 | Ingress 鍜?Service 涓嶅湪鍚屼竴鍛藉悕绌洪棿 | 鍚堝苟鍒板悓涓€鍛藉悕绌洪棿 |
| IngressClass 涓嶅尮閰?| `ingressClassName` 涓?Controller 涓嶅尮閰?| 妫€鏌?IngressClass |
| pathType 杩囦弗 | `Exact` 涓嶅尮閰嶅甫灏鹃儴鏂滄潬鐨勮矾寰?| 鍒囨崲涓?`Prefix` |
| Host 澶磋淇敼 | CDN/WAF 淇敼浜?Host 澶?| 閰嶇疆 Host 澶撮€忎紶 |

### 9.1.2 503 Service Unavailable

503 琛ㄧず Ingress 瑙勫垯鍖归厤鎴愬姛锛屼絾鏃犳硶杩炴帴鍒板悗绔?Pod銆傝繖鏄渶甯歌鐨勫叆鍙ｆ晠闅溿€?
**鎺掓煡娓呭崟**锛?
```shell
# 1. 妫€鏌?Service Endpoints
kubectl get endpoints <service-name> -n <namespace>
# 濡傛灉 ENDPOINTS 涓虹┖锛岃鏄庨棶棰樺湪 Service-Pod 閾捐矾

# 2. 妫€鏌?Pod 鐘舵€佸拰灏辩华鐘舵€?kubectl get pods -n <namespace> -l app=<label>
kubectl describe pod <pod-name> -n <namespace>

# 3. 鐩存帴浠?Pod 娴嬭瘯杩為€氭€?kubectl exec -n <namespace> <pod-name> -- curl localhost:<port>/healthz

# 4. 妫€鏌?Ingress Controller 鏄惁鍙揪 Service
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>/

# 5. 鏌ョ湅 Controller 鏃ュ織涓殑 upstream 閿欒
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200 | grep -i "upstream\|error\|503"
```

**甯歌鍘熷洜**锛?
| 鍘熷洜 | 鐜拌薄 | 瑙ｅ喅鏂规 |
|------|------|---------|
| Pod 鏈氨缁?| `kubectl get endpoints` 涓虹┖ | 绛夊緟 Pod 灏辩华鎴栨鏌?Readiness Probe |
| 绔彛鍙烽敊璇?| Service 绔彛涓?Pod containerPort 涓嶄竴鑷?| 鏍稿 targetPort 鍜?containerPort |
| 缃戠粶绛栫暐闃绘 | NetworkPolicy 闃绘浜?Controller 鍒?Pod 鐨勬祦閲?| 璋冩暣 NetworkPolicy |
| Pod 鍏ㄩ儴 CrashLoopBackOff | 鎵€鏈?Pod 閮藉浜庡紓甯哥姸鎬?| 鏌ョ湅 Pod 鏃ュ織鎺掓煡搴旂敤宕╂簝鍘熷洜 |
| kube-proxy 寮傚父 | iptables/ipvs 瑙勫垯鏈洿鏂?| 妫€鏌?kube-proxy 鐘舵€佸拰鏃ュ織 |
| Service selector 涓嶅尮閰?| Service 鏃犳硶閫夋嫨鍒?Pod | 鏍稿 Service selector 鍜?Pod labels |
| 璧勬簮涓嶈冻 OOMKilled | Pod 琚?OOM Killer 缁堟 | 澧炲姞鍐呭瓨闄愬埗鎴栨帓鏌ュ唴瀛樻硠婕?|

## 9.2 TLS璇佷功涓嶇敓鏁?
```shell
# 1. 妫€鏌?Secret 鏄惁瀛樺湪涓旀牸寮忔纭?kubectl get secret <secret-name> -n <namespace> -o yaml

# 2. 楠岃瘉璇佷功鍐呭
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# 3. 妫€鏌ヨ瘉涔︽槸鍚﹁繃鏈?kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -dates -noout

# 4. 楠岃瘉绉侀挜鍜岃瘉涔﹀尮閰嶏紙涓や釜 MD5 鍊肩浉鍚屽垯琛ㄧず鍖归厤锛?kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -modulus | openssl md5
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.tls\.key}' | base64 -d | openssl rsa -noout -modulus | openssl md5

# 5. 浣跨敤 OpenSSL 娴嬭瘯 TLS 鎻℃墜
openssl s_client -connect api.example.com:443 -servername api.example.com -tls1_3

# 6. 妫€鏌ユ槸鍚︽湁澶氫釜 TLS Secret 绔炰簤
kubectl get ingress --all-namespaces -o yaml | grep -B 5 "secretName: <name>"
```

**甯歌鍘熷洜涓庤В鍐?*锛?
| 鍘熷洜 | 瑙ｅ喅鏂规 |
|------|---------|
| Secret 绫诲瀷涓嶆槸 `kubernetes.io/tls` | 浣跨敤 `kubectl create secret tls` 鍒涘缓 |
| `tls.crt` 缂哄皯涓棿璇佷功 | 浣跨敤瀹屾暣璇佷功閾撅紙fullchain.pem锛?|
| 绉侀挜鏈夊瘑鐮佷繚鎶?| 绉婚櫎绉侀挜瀵嗙爜 |
| 璇佷功杩囨湡 | 缁湡璇佷功鎴栭厤缃?cert-manager 鑷姩缁湡 |
| cert-manager Issuer 閰嶇疆閿欒 | 妫€鏌?Issuer/ClusterIssuer 鐘舵€?|
| DNS-01 Challenge 璁板綍鏈敓鏁?| 绛夊緟 DNS 浼犳挱锛堟渶闀?10 鍒嗛挓锛?|
| SNI 涓嶅尮閰?| 纭璇锋眰鐨?Host 澶村湪 `tls.hosts` 鍒楄〃涓?|

## 9.3 Ingress Controller涓峳eload

**甯歌鍘熷洜涓庢帓鏌?*锛?
```shell
# 1. 鏉冮檺涓嶈冻锛氭鏌?Controller 鐨?RBAC
kubectl auth can-i watch ingresses --as=system:serviceaccount:ingress-nginx:ingress-nginx -n <namespace>

# 2. ConfigMap 閰嶇疆閿欒
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=50 | grep -i "error\|fail\|template"

# 3. Nginx 閰嶇疆璇硶閿欒锛坰nippet 娉ㄨВ瀵艰嚧鐨勶級
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- nginx -t

# 4. 妫€鏌?Controller 璧勬簮浣跨敤鏄惁 OOM
kubectl top pod -n ingress-nginx

# 5. 妫€鏌ュ悓姝ュ懆鏈?kubectl get deploy ingress-nginx-controller -n ingress-nginx -o yaml | grep sync-period

# 6. 鏌ョ湅 Controller 鏄惁姝ｅ父 Watch
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep -i "watching\|sync\|reload"
```

## 9.4 鎺掓煡鍛戒护涓庡伐鍏?
### 9.4.1 鍩虹鎺掓煡鍛戒护娓呭崟

```shell
# Ingress 鐘舵€佹鏌?kubectl get ingress --all-namespaces
kubectl describe ingress <name> -n <namespace>

# 鏌ョ湅 Ingress 鍦板潃鏄惁宸插垎閰?kubectl get ingress <name> -n <namespace> -o wide

# Service 鍜?Endpoints 妫€鏌?kubectl get svc,endpoints -n <namespace>

# Pod 鐘舵€佸拰鏃ュ織
kubectl get pods -n <namespace> -o wide
kubectl logs -f <pod-name> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>

# Ingress Controller 鏃ュ織
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200 -f

# 妫€鏌?IngressClass
kubectl get ingressclass
kubectl describe ingressclass <name>

# 妫€鏌?TLS Secret
kubectl get secret -n <namespace> -l type=kubernetes.io/tls

# RBAC 鏉冮檺妫€鏌?kubectl auth can-i --list --as=system:serviceaccount:ingress-nginx:ingress-nginx

# 娴嬭瘯 Ingress 瑙勫垯
curl -v -H "Host: api.example.com" http://<ingress-lb-ip>/path

# DNS 妫€鏌?nslookup api.example.com
dig api.example.com
```

### 9.4.2 楂樼骇璇婃柇宸ュ叿

**鐩存帴浠?Ingress Controller Pod 鍐呮祴璇?*锛?
```shell
# 鑾峰彇 Controller Pod 鍚嶇О
POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')

# 妫€鏌ョ敓鎴愮殑 nginx.conf
kubectl exec -n ingress-nginx $POD -- cat /etc/nginx/nginx.conf | less

# 娴嬭瘯 nginx 閰嶇疆璇硶
kubectl exec -n ingress-nginx $POD -- nginx -t

# 娴嬭瘯鍒板悗绔?Service 鐨勮繛閫氭€?kubectl exec -n ingress-nginx $POD -- curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>/
```

**閫氳繃涓存椂 Pod 娴嬭瘯缃戠粶杩為€氭€?*锛?
```shell
kubectl run test-pod --rm -it --image=nicolaka/netshoot -- /bin/bash

# 鍦?Pod 鍐呮墽琛?curl -v -H "Host: api.example.com" http://<ingress-lb-ip>/users
nslookup <service-name>.<namespace>.svc.cluster.local
telnet <service-name>.<namespace>.svc.cluster.local 8080
```

### 9.4.3 鐩戞帶涓庡憡璀?
**鍏抽敭 Prometheus 鎸囨爣**锛?
```
# Nginx Ingress Controller 鏍稿績鎸囨爣
nginx_ingress_controller_requests              # 璇锋眰鎬绘暟
nginx_ingress_controller_nginx_process_requests_total  # 璇锋眰閫熺巼
nginx_ingress_controller_request_duration_seconds_bucket  # 璇锋眰寤惰繜鍒嗗竷
nginx_ingress_controller_ingress_upstream_latency_seconds  # upstream 寤惰繜

# 閿欒鎸囨爣
nginx_ingress_controller_requests{status=~"5.."}  # 5xx 閿欒鏁?nginx_ingress_controller_requests{status="404"}    # 404 閿欒鏁?nginx_ingress_controller_requests{status="503"}    # 503 閿欒鏁?```

**鎺ㄨ崘鍛婅瑙勫垯**锛?
```yaml
# 503 閿欒鐜囧憡璀?- alert: High503Rate
  expr: rate(nginx_ingress_controller_requests{status="503"}[5m]) / rate(nginx_ingress_controller_requests[5m]) > 0.01
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "HTTP 503 rate is above 1%"

# 璇佷功鍗冲皢杩囨湡
- alert: CertificateExpiring
  expr: certmanager_certificate_expiration_timestamp_seconds - time() < 86400 * 14
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "TLS certificate will expire within 14 days"

# Ingress Controller 鏈氨缁?- alert: IngressControllerDown
  expr: up{job="ingress-nginx-controller"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Ingress controller pod is down"
```

### 9.4.4 甯歌闂閫熸煡琛?
| 鐥囩姸 | 绗竴鏃堕棿妫€鏌?| 甯哥敤鍛戒护 |
|------|------------|---------|
| 鎵€鏈夎姹傝繑鍥?404 | Ingress 鏄惁瀛樺湪銆乺ules 鏄惁姝ｇ‘ | `kubectl get ingress -A` |
| 鎵€鏈夎姹傝繑鍥?503 | Pod 鐘舵€併€丒ndpoints | `kubectl get endpoints` |
| 鐗瑰畾鍩熷悕 404 | host 瀛楁鍖归厤 | `kubectl get ingress -o yaml` |
| 鐗瑰畾璺緞 404 | path/pathType 閰嶇疆 | `kubectl describe ingress` |
| HTTPS 涓嶇敓鏁?| TLS Secret銆佽瘉涔﹂摼 | `openssl s_client -connect` |
| 鏂?Ingress 涓嶇敓鏁?| IngressClass銆丆ontroller 鏃ュ織 | `kubectl logs -n ingress-nginx deploy/...` |
| 闂存瓏鎬?503 | 璧勬簮闄愬埗銆丠PA 閰嶇疆 | `kubectl top pod`, `kubectl describe hpa` |
| Controller 棰戠箒閲嶅惎 | OOM銆侀厤缃敊璇?| `kubectl describe pod`, `kubectl logs` |
| 璇佷功鍒版湡 | cert-manager 鐘舵€?| `kubectl get certificate -A` |
| 璺ㄥ煙闂 | CORS 娉ㄨВ閰嶇疆 | `kubectl get ingress -o yaml` |

## 9.5 鎬ц兘璋冧紭鍙傝€?
### 9.5.1 Nginx Ingress Controller 璋冧紭

```yaml
# ConfigMap 绾у埆鎬ц兘璋冧紭
data:
  worker-processes: "auto"                  # 鑷姩鍖归厤 CPU 鏍稿績鏁?  max-worker-connections: "65536"           # 姣忎釜 worker 鏈€澶ц繛鎺ユ暟
  worker-cpu-affinity: "auto"              # CPU 浜插拰鎬?  upstream-keepalive-connections: "320"     # 淇濇寔杩炴帴鏁?  upstream-keepalive-timeout: "60"          # 淇濇寔杩炴帴瓒呮椂
  upstream-keepalive-requests: "10000"      # 姣忎釜杩炴帴鏈€澶ц姹傛暟
  proxy-buffers-number: "8"                # 缂撳啿鍖烘暟閲?  proxy-buffer-size: "16k"                 # 缂撳啿鍖哄ぇ灏?  disable-ipv6: "true"                     # 濡傛灉涓嶉渶瑕?IPv6
  disable-ipv6-dns: "true"                 # 濡傛灉 DNS 涓嶉渶瑕?AAAA 璁板綍
  enable-brotli: "true"                    # Brotli 鍘嬬缉锛堟瘮 gzip 鏇撮珮鏁堬級
```

### 9.5.2 鍐呮牳鍙傛暟璋冧紭

```shell
# 鑺傜偣绾у埆 sysctl 璋冧紭
sysctl -w net.core.somaxconn=65535          # 渚﹀惉闃熷垪澶у皬
sysctl -w net.ipv4.tcp_max_syn_backlog=8192 # SYN 闃熷垪
sysctl -w net.core.netdev_max_backlog=5000  # 缃戝崱绉帇
sysctl -w net.ipv4.tcp_tw_reuse=1           # TIME_WAIT 閲嶇敤
sysctl -w net.ipv4.ip_local_port_range="1024 65535"  # 鏈湴绔彛鑼冨洿
```

