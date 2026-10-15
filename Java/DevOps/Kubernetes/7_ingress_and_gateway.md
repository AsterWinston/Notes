Service 解决了 Pod 的 IP 不稳定问题，但它只把问题从"四层"推向"七层"——NodePort 端口爆炸、LoadBalancer 一个服务一台 LB，都无法支撑真实业务的统一入口需求。Ingress 用一份声明式规则把 HTTP/HTTPS 路由、TLS 终结、域名分流收敛到集群边界，而 Gateway API 则是它在角色分离与协议扩展上的下一代答案。本篇讲清两者的设计动机、配置细节、主流 Controller 的实现差异与线上排障方法。

（说明：本文件历史版本存在编码损坏，已按原有章节结构重写。）

# 1. 为什么需要Ingress

## 1.1 NodePort和LoadBalancer的局限性

在 Ingress 出现之前，把集群内的服务暴露给集群外只有两条路：NodePort 和 LoadBalancer。这两条路在单服务 Demo 时都很好用，一旦服务数量上到两位数就会立刻崩溃。

### 1.1.1 NodePort 模式：端口爆炸与运维噩梦

NodePort 的原理是在**每个 Node 上**监听一个 30000–32767 范围内的端口，并把该端口的流量转发到对应的 Service，再由 Service 转发到 Pod：

```
外部客户端
    │
    ▼  http://<任一NodeIP>:30080
┌─────────────────────────────────────────────────┐
│ Node A (10.0.1.10)                              │
│   :30080 ──┐                                    │
│ Node B (10.0.1.11)                              │
│   :30080 ──┼──▶ Service (order-svc) ──▶ Pod      │
│ Node C (10.0.1.12)                              │
│   :30080 ──┘                                    │
└─────────────────────────────────────────────────┘
```

它有三个无法回避的问题：

1. **端口数量是硬上限**。可用端口区间只有 2768 个，而集群内需要对外暴露的服务可能远超这个数，端口分配本身就成了需要人工协调的资源。
2. **端口号无法记忆**。用户访问的是 `http://1.2.3.4:31234` 这种地址，既没有域名语义，也无法做多环境统一——测试环境是 31234，生产环境可能是 30080。
3. **每个服务都要占端口**。10 个服务就是 10 个 NodePort，还必须逐个在负载均衡器上配置转发规则；新增服务要先申请端口、再改网关配置、再通知调用方，发布流程被拉得极长。

```yaml
# NodePort Service 示例：端口管理混乱的缩影
apiVersion: v1
kind: Service
metadata:
  name: order-svc
spec:
  type: NodePort
  selector:
    app: order
  ports:
    - port: 8080          # Service 端口
      targetPort: 8080    # Pod 端口
      nodePort: 30080     # 每个 Node 上开放的端口，需人工分配且不可冲突
```

服务的数量一多，"哪个端口对应哪个服务"就只能靠一张外部表格来维护，而这张表格永远不会是最新的。

### 1.1.2 LoadBalancer 模式：成本与粒度的困境

LoadBalancer 让云厂商自动创建一个外部负载均衡器（AWS NLB/ALB、GCP LB 等），并把这个 LB 的地址写回 Service 的 `status.loadBalancer.ingress`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-svc
spec:
  type: LoadBalancer
  selector:
    app: order
  ports:
    - port: 80
      targetPort: 8080
```

它的核心缺陷是**粒度太细**：一个 Service 对应一台 LB。

- **成本线性增长**。云厂商的 LB 按实例计费，一个 NLB 每月约 20 美元起，加上 LCU/流量费用。50 个对外服务就是 50 台 LB，每月固定成本轻松上千美元，而这部分流量的七层路由能力（基于路径、Header 的分流）其实完全用不上。
- **LB 数量成为配额瓶颈**。账号级别的 LB 数量配额通常是几十个，中大型集群很容易触顶，需要走工单申请。
- **七层能力欠缺**。NLB 是四层转发，无法基于 HTTP 路径、Host、Header 做路由，也无法做基于 Cookie 的会话保持和 URL 重写；要用这些能力就得换成 ALB，而 ALB 的 Ingress 注解又是云厂商私有的，可移植性差。
- **配置分散**。50 个 LB 就是 50 个分散的配置面板，没有统一视图。

### 1.1.3 两种模式的共同缺陷总结

| 维度 | NodePort | LoadBalancer | Ingress |
|------|----------|--------------|---------|
| 成本 | 免费，但端口是稀缺资源 | 每个服务一台 LB，成本线性增长 | 一台入口承载全部服务 |
| 入口地址 | 每服务一个 `IP:端口` | 每服务一个独立 IP | 一个域名/一条 LB 承载所有服务 |
| 七层路由 | 不支持 | 视 LB 类型而定，通常不支持 | 原生支持 Host/Path/Header 路由 |
| TLS 终结 | 需应用自己处理 | 需在每个 LB 上单独配置 | 在入口统一配置，证书集中管理 |
| 新增服务 | 申请端口 + 改网关 | 新建 LB + 等分配 IP | 加一条规则，秒级生效 |
| 可移植性 | 好 | 强绑定云厂商 | 标准 API，跨云一致 |

两者共同的病根是：**把"暴露服务"和"路由流量"混为一谈**。NodePort/LoadBalancer 只关心"这个服务能不能被访问到"，而真实业务关心的是"什么样的请求应该落到哪个服务"。前者是四层问题，后者是七层问题。

## 1.2 Ingress解决什么问题

Ingress 的思路是把七层路由能力从每个服务里抽出来，集中放到集群边界的一个统一入口上。一个 Ingress 对象描述的是"**哪些请求应该转发到哪里**"，而不是"哪个服务开哪个端口"。

### 1.2.1 统一入口（Single Entry Point）

所有外部流量先到同一个入口（通常是一个 LoadBalancer 类型的 Service 背后的 Ingress Controller），再由它按规则分发：

```
                        ┌──────────────────────────────┐
   用户请求              │      Ingress Controller      │
  api.example.com  ─────▶│  (Nginx / Traefik / ...)     │
  admin.example.com ────▶│                              │
  www.example.com  ─────▶│  按 Host / Path 规则分发      │
                        └───────┬──────────┬───────────┘
                                │          │
                    ┌───────────▼──┐  ┌────▼─────────┐
                    │  api-svc     │  │  admin-svc   │
                    └──────────────┘  └──────────────┘
```

好处是成本模型从"每服务一台 LB"变成"整个集群一台 LB"：无论后面挂 5 个还是 500 个服务，对外只有一条入口，LB 费用固定，DNS 也只需要维护一条记录。

### 1.2.2 七层路由（L7 Routing）

一个 Ingress 对象可以同时声明多条规则，命中顺序由 Host 与 Path 的匹配规则决定：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-v1
                port:
                  number: 8080
          - path: /v2/orders
            pathType: Prefix
            backend:
              service:
                name: order-v2
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-frontend
                port:
                  number: 80
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-console
                port:
                  number: 8080
```

这个配置承载了四件事：域名分流（api 与 admin 走不同后端）、路径分流（v1 与 v2 灰度共存）、默认兜底（`/` 交给前端）、以及一个新服务上线只需要改动几行 YAML。

路由匹配遵循**最长前缀优先**：`/v2/orders/1` 会命中 `/v2/orders` 而不是 `/`，因为前者前缀更长。这一点在排查"请求走错后端"时非常关键。

### 1.2.3 TLS/SSL 终结（TLS Termination）

证书不放在每个应用里，而是统一挂到 Ingress 上。客户端与 Ingress 之间是 HTTPS，Ingress 与后端 Pod 之间走集群内网，通常是明文 HTTP：

```yaml
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls        # 存放证书与私钥的 Secret
  rules:
    - host: api.example.com
      # ...
```

这样后端应用完全不用关心证书（不需要在 Java 代码里配置 keystore），证书续期、加密套件基线、HTTP 强制跳转 HTTPS 都收敛到一处。代价是后端到入口这一段是明文，安全性由集群网络策略保证；对合规要求高的场景可以开启 mTLS 或后端 HTTPS。

### 1.2.4 其他核心能力

除了路由和 TLS，Ingress 层能做的事情基本等同于"一个可编程的反向代理"：

| 能力 | 典型用法 |
|------|---------|
| URL 重写 | `/api/v1/users` 对外暴露为 `/users` |
| 重定向 | HTTP 301/302 跳转、强制 HTTPS |
| 跨域 | 在入口统一注入 CORS 响应头 |
| 认证 | Basic Auth、OAuth2、Client Certificate |
| 限流 | 单 IP 并发连接数、每秒请求数、突发额度 |
| 会话保持 | 基于 Cookie 的粘性会话 |
| 长连接 | WebSocket、SSE、gRPC |
| 灰度发布 | 按权重 / Header / Cookie 切流 |

这些能力在 Nginx Ingress 里大多通过 **annotation** 实现，而 annotation 正是后来 Gateway API 要解决的问题（见第 7 节）。

# 2. Ingress架构

## 2.1 Ingress Controller + Ingress Resource 分离设计

理解 Ingress 最重要的一点是：**Ingress 只是一份规则声明，它本身没有任何转发能力**。真正干活的是 Ingress Controller。这两者分离是 Ingress 模型的核心设计。

### 2.1.1 设计哲学：标准 API + 可替换实现

Kubernetes 把"描述规则"和"实现规则"拆成了两个东西：

```
┌────────────────────────────────────────────────────────┐
│                    控制平面（API 对象）                  │
│   Ingress Resource：我要把 api.example.com/v1 转发到    │
│                     order-v1:8080，TLS 用 api-tls       │
└────────────────────────┬───────────────────────────────┘
                         │ Watch / List
                         ▼
┌────────────────────────────────────────────────────────┐
│                  数据平面（实际代理）                    │
│   Ingress Controller：监听 Ingress 变化 → 生成 nginx.conf │
│                       → reload → 真正转发流量            │
└────────────────────────────────────────────────────────┘
```

好处与 Service/lb 的关系类似：

- **规则与厂商解耦**。同一份 Ingress YAML 可以在 AWS、GCP、自建机房通用，只要换一个 Controller。
- **实现可替换**。Nginx 性能不够可以换 HAProxy，想要动态配置可以换 Traefik 或 APISIX，业务侧的 YAML 不用改。
- **多实现共存**。通过 `ingressClassName` 可以让不同的 Ingress 对象落到不同的 Controller 上，比如内网服务走 Traefik、外网服务走 Nginx。

### 2.1.2 Ingress Resource：声明式规则定义

Ingress Resource 是一份纯粹的状态描述，字段结构如下：

```yaml
apiVersion: networking.k8s.io/v1          # 1.19+ 的稳定版本
kind: Ingress
metadata:
  name: demo
  namespace: default
  annotations:                            # Controller 私有扩展，非标准字段
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx                 # 指定由哪个 Controller 处理
  defaultBackend:                         # 没命中任何 rules 时的兜底
    service:
      name: fallback
      port:
        number: 80
  tls:                                    # TLS 终结配置
    - hosts: [demo.example.com]
      secretName: demo-tls
  rules:                                  # 路由规则
    - host: demo.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8080
```

关键点是 `spec` 里**没有地址、没有端口、没有 IP**——它只描述意图，不描述实现。"入口地址是多少"由 Controller 决定并写回到 `status.loadBalancer.ingress`。

### 2.1.3 Ingress Controller：配置驱动的代理实例

Ingress Controller 本质上是一个运行在集群里的 Pod（通常以 Deployment 部署），它同时扮演两个角色：

1. **控制器**：Watch Ingress/Service/EndpointSlice/Secret 等资源，把变化翻译成代理配置。
2. **代理**：自身就是一个 Nginx/Traefik/Envoy 进程，承载真实流量。

以 Nginx Ingress Controller 为例，一个 Pod 里实际跑了两个进程：

```
┌──────────────────────────────────────────────┐
│  ingress-nginx-controller Pod                │
│                                              │
│  ┌────────────────┐   写文件   ┌───────────┐  │
│  │ Go 写的控制器   │──────────▶│ nginx.conf│  │
│  │ (监听 API 变化) │           └─────┬─────┘  │
│  └────────────────┘                 │ reload │
│                                     ▼        │
│                            ┌────────────────┐│
│                            │  Nginx Worker  ││ ← 真实流量
│                            └────────────────┘│
└──────────────────────────────────────────────┘
```

它通常以 `hostNetwork: true` 或 `LoadBalancer` Service 的方式对外暴露。用 `LoadBalancer` 时，云 LB 只创建一次，后续所有 Ingress 都复用这个入口。

### 2.1.4 分离设计的工程实践

分离带来灵活性，也带来几个实践要点：

- **Controller 必须与 Ingress 的 `ingressClassName` 对齐**。没有匹配的 Controller，Ingress 会被"晾着"——对象创建成功，但 `status` 里永远没有地址，流量也进不来。这是新手最常见的"部署成功但访问不通"。
- **Controller 是集群级组件，通常只部署一份**（或按入口分组部署多份）。它天然是单点，需要多副本 + PodDisruptionBudget + 反亲和性保证可用性。
- **Controller 权限很大**。它需要读取全集群的 Ingress/Secret，因此对应的 ClusterRole 权限也要收紧，避免被滥用于横向读取证书。

## 2.2 请求处理流程

### 2.2.1 完整请求链路

一次外部请求到达 Pod，中间要穿过好几层。理解这条链路是排障的基础：

```
 1. 客户端 DNS 解析 api.example.com → 得到 Ingress 入口 IP
        │
        ▼
 2. 流量到达云 LB / Node:30080（取决于 Controller 的暴露方式）
        │
        ▼
 3. 进入 Ingress Controller Pod 的 Nginx 进程
        │  ├─ TLS 握手（用 Secret 里的证书）
        │  ├─ 按 Host + Path 匹配 location / server_name
        │  └─ 应用 annotation 对应的 rewrite / CORS / 限流等指令
        ▼
 4. Nginx 以 upstream 形式转发到 Service ClusterIP（或 EndpointSlice 里的 Pod IP）
        │
        ▼
 5. kube-proxy 的 iptables/IPVS 规则把 Service IP 转成某个 Pod IP
        │
        ▼
 6. 到达后端 Pod，返回响应，原路返回
```

其中第 3 步的匹配顺序是固定的：先按 `server_name`（Host）选定 server 块，再在该 server 内按 `location` 最长前缀匹配 Path。如果 Host 没有任何 server 命中，就落到 `defaultBackend`。

### 2.2.2 性能与可靠性考量

- **跳数与延迟**。相比"客户端直连 Pod"，Ingress 多了一次七层代理，增加了约 0.5–2ms 的延迟（视配置复杂度）。这对绝大多数 HTTP 业务可忽略，但对超低延迟场景要评估。
- **源 IP 丢失**。默认情况下后端 Pod 看到的是 Controller 的 Pod IP，而不是客户端真实 IP。需要显式开启：

  ```yaml
  # 对外部流量保留真实客户端 IP
  # Nginx Ingress 的 Service 需要设置 externalTrafficPolicy: Local
  spec:
    externalTrafficPolicy: Local
  ```

  配合 `use-forwarded-headers: "true"`，后端才能从 `X-Forwarded-For` 取到真实 IP。`externalTrafficPolicy: Local` 的代价是流量只会落到运行着 Controller Pod 的节点，需要保证每个节点都有实例，否则会有节点级单点。
- **Controller 是新的单点**。它承载全部南北向流量，必须配置多副本、合理的资源 limits、以及 HPA 或基于连接数的扩容策略。
- **连接超时**。入口的 keepalive、proxy 超时要与后端应用的超时匹配，否则会出现"入口已经断开、后端还在处理"的幽灵请求。

## 2.3 Ingress Controller工作原理

### 2.3.1 Watch 机制：Kubernetes 的实时事件驱动

Controller 不轮询，而是通过 client-go 的 **Informer** 机制与 API Server 建立长连接，监听资源的 Add/Update/Delete 事件：

```
┌──────────────┐   Watch 长连接   ┌──────────────────────┐
│  API Server  │◀────────────────│ Ingress Controller   │
└──────────────┘                  │                      │
                                  │  ┌────────────────┐  │
  监听对象：                       │  │  Informer      │  │
  - Ingress                       │  │  (本地缓存 +    │  │
  - Service                       │  │   事件回调)     │  │
  - EndpointSlice                 │  └───────┬────────┘  │
  - Secret (TLS 证书)             │          │ 入队       │
  - IngressClass                  │          ▼           │
  - ConfigMap (全局配置)          │  ┌────────────────┐  │
                                  │  │  Work Queue    │  │
                                  │  └───────┬────────┘  │
                                  │          │ 去重/合并  │
                                  │          ▼           │
                                  │    生成 nginx.conf   │
                                  └──────────────────────┘
```

三个设计要点：

- **本地缓存**。Informer 在本地维护一份资源快照，生成配置时读本地缓存而不是逐个请求 API Server，避免把 API Server 打爆。
- **Work Queue 去重**。同一个 Ingress 在短时间内被多次修改，队列会合并成一次处理，避免频繁 reload。
- **Resync 兜底**。即使 Watch 连接中断漏了事件，定期全量 resync 也能把状态纠正回来。

### 2.3.2 配置生成与 Reload

Nginx Ingress Controller 的配置更新过程是"全量生成 + reload"：

1. 从本地缓存取出所有 Ingress、Service、EndpointSlice，渲染成一份完整的 `nginx.conf`。
2. 写入临时文件，做语法校验（`nginx -t`），通过后原子替换。
3. 向 Nginx master 进程发送 `SIGHUP`，master 启动新的 worker 并平滑退出旧 worker。

```bash
# 在 Controller Pod 里查看生成的配置（排障必备）
kubectl exec -n ingress-nginx <controller-pod> -- cat /etc/nginx/nginx.conf

# 校验并重载
kubectl exec -n ingress-nginx <controller-pod> -- nginx -t
kubectl exec -n ingress-nginx <controller-pod> -- nginx -s reload
```

全量 reload 的代价是：**配置规模大时 reload 开销显著**。几千个 Ingress 的场景下，每次 reload 都要重新生成几十 MB 的配置，且 reload 瞬间会有内存峰值。缓解手段包括：

- 开启 `--enable-dynamic-certificates`，证书更新不再触发 reload。
- 用 Lua/OpenResty 的动态 upstream（`balancer_by_lua`）把后端变更从 reload 中移除。
- 换用 APISIX/Envoy 这类支持 xDS 增量推送的实现，从根本上避免全量 reload。

### 2.3.3 配置同步的竞态与一致性

Ingress 与它引用的 Service/Endpoints 是分开的对象，更新有先后，因此存在天然的瞬时不一致窗口：

- **Service 刚创建、Ingress 先到**。Controller 渲染配置时找不到后端，会生成一个没有 upstream 的规则，请求返回 503。等 Service 事件到达后自动恢复，因此这类 503 通常是秒级的。
- **后端 Pod 已删除、EndpointSlice 未更新**。会在一个很短的窗口内把请求转发到已经不存在的 Pod，表现为零星 502/504。这属于正常收敛过程，靠 `readinessProbe` + `preStop` 优雅退出把窗口压到最小。
- **证书 Secret 与 Ingress 引用不同步**。Ingress 引用了不存在的 Secret 时，Nginx 会退回使用默认自签证书（`nginx-ingress-controller-tls`），浏览器报证书不受信任。

排障时不要只看最终状态，要关注**事件的时间顺序**：

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp | tail -30
kubectl describe ingress <name> -n <namespace>   # 看 Events 里的告警
```

### 2.3.4 多 Controller 的事件过滤

一个集群里可以有多个 Ingress Controller（比如 Nginx 处理外网、Traefik 处理内网）。它们的过滤机制有两套，新旧并存：

| 方式 | 字段 | 说明 |
|------|------|------|
| `ingressClassName`（推荐） | `spec.ingressClassName: nginx` | 1.18+ 引入，配合 IngressClass 对象使用 |
| annotation（旧） | `kubernetes.io/ingress.class: nginx` | 1.18 之前的方式，仍被兼容 |

匹配规则很简单：**Controller 只处理与自身 class 匹配的 Ingress，其余直接忽略**。两个坑：

1. **只认 `spec.ingressClassName`，不认 `metadata.name`**。`IngressClass` 的 `metadata.name` 是给管理员看的标识，实际匹配靠 `spec.controller`（如 `k8s.io/ingress-nginx`）或 Controller 启动参数里的 `--ingress-class`，别混淆。
2. **两者同时存在时行为不确定**。如果 `spec.ingressClassName` 和旧 annotation 都写了，不同版本行为不一致，迁移期应当二选一。

```bash
# 查看集群里有哪些 IngressClass，以及谁被标记为默认
kubectl get ingressclass
kubectl get ingressclass nginx -o yaml | grep -A2 annotations

# 把某个 IngressClass 标记为集群默认（没写 ingressClassName 的 Ingress 会落到它身上）
kubectl annotate ingressclass nginx ingressclass.kubernetes.io/is-default-class=true
```

# 3. Ingress YAML详解

## 3.1 完整YAML结构

一个字段齐全的 Ingress 对象长这样：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: full-demo
  namespace: production
  labels:
    app: web
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: fallback-svc
      port:
        number: 80
  tls:
    - hosts:
        - demo.example.com
      secretName: demo-tls
  rules:
    - host: demo.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  name: http          # 也可以用 number
```

顶层字段只有 6 个：`metadata`、`spec.ingressClassName`、`spec.defaultBackend`、`spec.tls`、`spec.rules`，以及 `status`（由 Controller 写入，用户不可改）。

### 3.1.1 API 版本演进

Ingress 的 API 版本经历过三次变化，是升级集群时最容易踩的坑：

| API 版本 | 引入 | 状态 | 主要差异 |
|----------|------|------|---------|
| `extensions/v1beta1` | 1.1 | 1.22 已移除 | 无 `pathType`，后端用 `serviceName`/`servicePort` |
| `networking.k8s.io/v1beta1` | 1.14 | 1.22 已移除 | 增加 `ingressClassName`，补齐 `pathType` |
| `networking.k8s.io/v1` | 1.19 | 当前稳定版 | 后端必须是 `service.name` + `service.port` 结构 |

从 `v1beta1` 迁到 `v1` 有两处结构性改动：

```yaml
# 旧：v1beta1 —— 后端字段是扁平的
backend:
  serviceName: api-svc
  servicePort: 8080
```

```yaml
# 新：v1 —— 后端字段是嵌套对象，pathType 必填
backend:
  service:
    name: api-svc
    port:
      number: 8080
pathType: Prefix        # v1 里是必填字段，漏写会被 API Server 拒绝
```

`pathType` 在 v1 里是**必填**的，这是从旧版本迁移时最常见的报错来源：`spec.rules[0].http.paths[0].pathType: Required value`。

### 3.1.2 metadata 字段

`metadata` 里除了标准的 `name`/`namespace`/`labels`，最关键的是 `annotations`——**Ingress 的绝大部分高级能力都靠 annotation 扩展**：

```yaml
metadata:
  annotations:
    # 重写请求路径
    nginx.ingress.kubernetes.io/rewrite-target: /
    # 上传大小上限（默认 1m，文件上传场景必调）
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    # 后端协议升级为 HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # 会话保持
    nginx.ingress.kubernetes.io/affinity: "cookie"
```

annotation 的命名空间是 Controller 私有的：`nginx.ingress.kubernetes.io/*` 只有 Nginx Ingress 认，换到 Traefik 就全部失效。这正是第 7 节要讲的 Gateway API 试图解决的问题。

## 3.2 rules: host/http/paths/pathType

`rules` 是一个数组，每个元素描述"某个 Host 下的若干条路径规则"。

### 3.2.1 host 字段：域名匹配

`host` 是精确的域名匹配（大小写不敏感，不支持端口号）：

```yaml
rules:
  - host: api.example.com      # 只匹配这个域名
  - host: "*.example.com"      # 泛域名：匹配 a.example.com，不匹配 a.b.example.com
  # 不写 host 则表示匹配任意域名（兜底规则）
```

几个容易踩的点：

- **泛域名只覆盖一级**。`*.example.com` 匹配 `api.example.com`，但**不匹配** `v1.api.example.com`，后者需要 `*.api.example.com`。
- **`host` 不能带端口**。写 `api.example.com:8080` 是非法的；端口由 Controller 的监听配置决定。
- **不写 `host` 的规则是兜底**，会匹配所有 Host。它常常被无意中写成"万能规则"，导致本该 404 的请求被转发到某个默认服务。

### 3.2.2 http.paths 数组：路径匹配

每个 `host` 下的 `http.paths` 是一个路径数组，按**最长前缀优先**匹配：

```yaml
paths:
  - path: /api/v2
    pathType: Prefix
    backend: { service: { name: order-v2, port: { number: 8080 } } }
  - path: /api
    pathType: Prefix
    backend: { service: { name: order-v1, port: { number: 8080 } } }
  - path: /
    pathType: Prefix
    backend: { service: { name: web, port: { number: 80 } } }
```

请求 `/api/v2/orders` 会命中第一条（前缀最长），`/api/users` 命中第二条，`/login` 命中第三条。**同一 Host 下最多一个通配路径**，且较长路径必须写在前面不是必需的——匹配由 `pathType` 语义决定而非书写顺序，但把精确的写在前面更利于人工阅读。

### 3.2.3 backend 字段：后端服务引用

`backend` 只指向一个 Service，不直接指向 Pod：

```yaml
backend:
  service:
    name: api-svc          # 必须与 Ingress 在同一命名空间
    port:
      number: 8080         # 或者用 name: http（引用 Service 里命名的端口）
```

用 `name` 比用 `number` 更健壮：Service 端口号变了不用改 Ingress。前提是 Service 的 `ports` 里给端口起了名：

```yaml
# Service 侧
ports:
  - name: http             # Ingress 可以用 port.name: http 引用
    port: 8080
    targetPort: 8080
```

**跨命名空间引用是不允许的**。Ingress 只能指向同命名空间的 Service，这是安全边界设计；要跨命名空间共享入口，只能靠 Gateway API 的 `ReferenceGrant`（见 7.4）。

## 3.3 pathType (Exact/Prefix/ImplementationSpecific)

`pathType` 决定路径怎么匹配。它在 `networking.k8s.io/v1` 里是必填的，有三种取值。

### 3.3.1 Exact：精确匹配

```yaml
path: /users
pathType: Exact
```

**只匹配 `/users` 本身**，`/users/`、`/users/1`、`/users?page=2` 都不匹配。注意查询字符串不参与匹配（`/users?page=2` 的路径部分仍是 `/users`），但末尾斜杠参与匹配。

适用场景：健康检查端点、极少数必须严格区分的路径。

### 3.3.2 Prefix：前缀匹配

```yaml
path: /api
pathType: Prefix
```

按**路径段（segment）**匹配，而不是按字符前缀：

| 请求路径 | `path: /api` + `Prefix` 是否匹配 | 说明 |
|----------|--------------------------------|------|
| `/api` | ✅ | 完全相等 |
| `/api/users` | ✅ | 后续段匹配 |
| `/api/v2/users` | ✅ | 后续段匹配 |
| `/apixyz` | ❌ | 不是按字符前缀，`apixyz` 不是 `api` 的后续段 |
| `/api/v2/../admin` | ✅ | 路径规范化的责任在客户端/代理 |

这一点与很多人的直觉相反：`/apixyz` **不会**命中 `Prefix: /api`，因为匹配是按 `/` 分割后的段进行的。这也意味着 `Prefix` 比想象中更安全，不会意外吞掉相邻路径。

### 3.3.3 ImplementationSpecific：由 Controller 决定

```yaml
path: /api
pathType: ImplementationSpecific
```

语义完全交给 Controller：Nginx Ingress 把带正则的路径（如 `/api/v[0-9]+`）和不带正则的路径分开处理，带正则的按正则匹配，不带正则的退化为 Prefix 行为。

**实践建议**：除非确实需要正则，一律用 `Prefix`。`ImplementationSpecific` 的行为在不同 Controller、甚至同一 Controller 的不同版本之间都可能变化，是"换个 Controller 就路由错乱"的常见根因。

三个取值的对比：

| pathType | 匹配方式 | 是否支持正则 | 可移植性 |
|----------|---------|-------------|---------|
| `Exact` | 全等 | 否 | 高 |
| `Prefix` | 按路径段前缀 | 否 | 高（推荐） |
| `ImplementationSpecific` | 由实现决定 | Nginx 下支持 | 低 |

## 3.4 defaultBackend (默认后端)

`defaultBackend` 是"没有任何规则命中时的兜底后端"。

### 3.4.1 典型用法

```yaml
spec:
  defaultBackend:
    service:
      name: default-http-backend
      port:
        number: 80
```

它通常指向一个返回 404 页面或统一错误页的服务。Nginx Ingress 默认提供一个 `default-http-backend`，返回 404。

### 3.4.2 与 rules 的配合

命中逻辑是一个两段判断：先看 Host 有没有匹配的 `server`，再看 Path 有没有匹配的 `location`。任一段不命中，就落到 `defaultBackend`：

```
请求 admin.example.com/orders
  │
  ├─ 有 host: admin.example.com 的规则吗？
  │     没有 ──▶ 落到 defaultBackend
  │     有 ──┬─▶ 这个 host 下有能匹配 /orders 的 path 吗？
  │          │        没有 ──▶ 落到该 server 的默认 location（通常也是 defaultBackend）
  │          │        有   ──▶ 转发到对应 Service
```

值得注意的是：**如果 Ingress 里完全没有 `host` 规则**（所有规则都不写 host），那么任意 Host 都会命中第一条规则。这在多租户环境里可能造成越权访问——A 租户的域名可能落到 B 租户的服务上。因此生产环境应当要么显式写 host，要么用 `defaultBackend` 兜底并让它返回 404。

## 3.5 TLS配置 (hosts/secretName)

### 3.5.1 基本结构

```yaml
spec:
  tls:
    - hosts:
        - demo.example.com
        - www.example.com
      secretName: demo-tls
  rules:
    - host: demo.example.com
      # ...
```

`tls[].hosts` 声明这张证书覆盖哪些域名，`secretName` 指向存放证书的 Secret。**两者必须对应**：证书的 SAN 里必须包含 `hosts` 里列出的域名，否则浏览器会报域名不匹配。

### 3.5.2 Secret 格式要求

TLS Secret 是标准的 `kubernetes.io/tls` 类型，只认两个 key：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-tls
  namespace: production        # 必须与 Ingress 同命名空间
type: kubernetes.io/tls
data:
  tls.crt: <base64>            # 证书链（服务器证书 + 中间证书）
  tls.key: <base64>            # 私钥，绝不能提交到 Git
```

用命令创建更安全：

```bash
kubectl create secret tls demo-tls \
  --cert=fullchain.pem \
  --key=privkey.pem \
  -n production
```

注意 `--cert` 传的应该是**证书链**（fullchain），即"服务器证书 + 中间 CA"，而不是只有服务器证书。只传服务器证书时，浏览器会因为缺少中间证书而报"证书链不完整"——桌面浏览器往往能靠 AIA 自动补齐，移动端和 curl 则会直接失败，这是"电脑上好的、手机上打不开"的经典原因。

### 3.5.3 TLS 匹配规则

一个 Ingress 可以配多组 TLS，Controller 按 SNI（TLS 握手里的 `server_name`）选择对应证书：

```yaml
tls:
  - hosts: [api.example.com]
    secretName: api-tls
  - hosts: [www.example.com]
    secretName: www-tls
```

如果 SNI 没有匹配任何 `tls[].hosts`，Nginx 会使用默认证书（通常是 Controller 的自签证书），浏览器报"连接不安全"。

### 3.5.4 通配符证书

```yaml
tls:
  - hosts:
      - "*.example.com"
    secretName: wildcard-tls
```

通配符证书只覆盖**一级**子域。`*.example.com` 覆盖 `api.example.com`，不覆盖 `v1.api.example.com`。多级子域要么申请多张证书，要么在证书的 SAN 里显式列出，要么改用逐域名的自动签发（cert-manager，见 6.2）。

### 3.5.5 TLS 与 HTTP 共存

同时监听 80 和 443 时，通常希望 HTTP 强制跳转到 HTTPS：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"          # 全局强制跳转
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"    # 即使没有 TLS 也跳
```

`ssl-redirect` 是默认开启的：只要 Ingress 配了 `tls`，HTTP 请求就会被 301 到 HTTPS。如果想保留某条路径走 HTTP（比如 ACME 的 `/.well-known/acme-challenge`），需要单独放行，否则证书续期会被自己的跳转规则挡住——这是一个很隐蔽的自锁问题。

## 3.6 IngressClass

### 3.6.1 为什么需要 IngressClass

IngressClass 解决的是"**这份 Ingress 该由哪个 Controller 处理**"的问题。在它出现之前，这个信息写在 annotation 里（`kubernetes.io/ingress.class`），既不是类型安全的字段，也无法表达"哪个是默认类"。

沿用旧方式的问题在于：

- annotation 拼写错误不会被 API Server 校验，Ingress 会静默地不被任何 Controller 处理。
- 无法标记默认值，每个 Ingress 都必须显式指定。

### 3.6.2 IngressClass 资源结构

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx                       # 这个名字才是 Ingress 里要引用的
spec:
  controller: k8s.io/ingress-nginx  # 标识哪个 Controller 实现
  parameters:                       # 可选：该 class 的额外参数（各实现自定义）
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
```

对应 Ingress 侧：

```yaml
spec:
  ingressClassName: nginx           # 引用 IngressClass 的 metadata.name
```

`spec.controller` 是 Controller 用来认领 IngressClass 的标识，由实现方定义（Nginx 是 `k8s.io/ingress-nginx`）。**部署 Controller 时它会自动创建自己的 IngressClass**，通常不需要手写。

### 3.6.3 默认 IngressClass

```bash
# 查看所有 IngressClass
kubectl get ingressclass

# 把 nginx 设为集群默认
kubectl annotate ingressclass nginx \
  ingressclass.kubernetes.io/is-default-class=true
```

设为默认后，没写 `ingressClassName` 的 Ingress 会自动落到它身上。两个坑：

1. **集群里只能有一个默认 IngressClass**。设了两个，没写 class 的 Ingress 行为不确定，不同 Controller 的实现可能各自认领，导致流量被两个入口同时处理。
2. **默认值只影响新对象**。已存在的 Ingress 不会因为改了默认值而改变归属，需要逐个补上 `ingressClassName`。

从旧 annotation 迁移到 `ingressClassName` 的推荐顺序是：先给所有 Ingress 补上 `ingressClassName`，确认 Controller 正常接管，再移除 annotation。反过来操作会出现一段"两边都不认或都认"的灰色期。

# 4. 注解 (Annotations) 体系

Ingress 标准字段只覆盖了"路由到哪"这一个维度，其余全部能力——重写、限流、认证、超时、灰度——都靠 annotation 扩展。annotation 是控制器私有的，因此这一节以 Nginx Ingress 为主，最后对比不同实现的差异。

## 4.1 Nginx Ingress常用注解

### 4.1.1 URL 重写与重定向

路径重写是把"对外暴露的 URL"与"后端实际路径"解耦的主要手段。Nginx Ingress 用一个组合实现：`use-regex` 让路径支持正则并捕获，`rewrite-target` 用捕获组重写：

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    # 把 /api/v1/xxx 重写为 /xxx
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1(/|$)(.*)      # $1 = "/" 或空，$2 = 后续路径
            pathType: ImplementationSpecific
            backend:
              service:
                name: user-svc
                port:
                  number: 8080
```

请求 `/api/v1/users/1` 到达后端时变成 `/users/1`。这里有三个必须记住的约束：

- **`use-regex` 会让该 Host 下所有 path 都按正则解释**，包括不打算用正则的那些。正则会按书写顺序依次匹配，容易造成难以排查的错序。
- **`rewrite-target` 一旦配置，所有 path 都会应用**。只想重写部分路径时，可以用 `nginx.ingress.kubernetes.io/rewrite-target` 的路径级覆盖写法，或干脆把需要重写的规则拆到独立 Ingress。
- **正则中的 `.` 要转义**，`/api/v1.0` 会把 `.` 当通配符。

重定向类注解：

```yaml
annotations:
  # 永久重定向
  nginx.ingress.kubernetes.io/permanent-redirect: https://new.example.com
  nginx.ingress.kubernetes.io/permanent-redirect-code: "308"
  # 临时重定向
  nginx.ingress.kubernetes.io/temporal-redirect: https://maintenance.example.com
  # 强制 HTTPS
  nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
  # 去掉前缀后转发（不重写、只剥离）
  nginx.ingress.kubernetes.io/app-root: /app
```

`rewrite-target`（重写，客户端地址栏不变）与 `permanent-redirect`（重定向，客户端会重新发起请求）语义完全不同，混用会导致多一次往返甚至死循环。

### 4.1.2 跨域（CORS）

在入口统一处理 CORS，业务代码就不用重复实现：

```yaml
annotations:
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com,https://admin.example.com"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
  nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization"
  nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
  nginx.ingress.kubernetes.io/cors-max-age: "600"        # 预检请求缓存秒数
```

两个坑：

- **`cors-allow-credentials: "true"` 时 `cors-allow-origin` 不能是 `*`**。浏览器规定携带凭证的请求必须指定具体源，写 `*` 会被浏览器拒绝。这是"本地测试好的，浏览器一开 Cookie 就失败"的原因。
- **预检请求会打到后端**。除非 Controller 显式拦截 `OPTIONS`，否则预检请求也会被转发，后端需要处理或不处理；`cors-max-age` 设小了会造成大量预检流量。

### 4.1.3 认证与安全

```yaml
annotations:
  # Basic Auth（凭据存在 Secret 里，格式为 htpasswd）
  nginx.ingress.kubernetes.io/auth-type: basic
  nginx.ingress.kubernetes.io/auth-secret: basic-auth
  nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

  # 外部 OAuth2 / OIDC 网关
  nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com/oauth2/auth"
  nginx.ingress.kubernetes.io/auth-signin: "https://auth.example.com/oauth2/start?rd=$escaped_request_uri"

  # IP 白名单
  nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"

  # 客户端证书（mTLS）
  nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
  nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
  nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"

  # 隐藏版本号
  nginx.ingress.kubernetes.io/server-snippet: |
    server_tokens off;
```

`whitelist-source-range` 依赖真实客户端 IP。如果前面还有一层 LB，且没开 `use-forwarded-headers` + `externalTrafficPolicy: Local`，白名单拿到的会是 LB 的 IP，导致白名单形同虚设或全量拦截。这是一个典型的"配置正确但 IP 取错"问题。

### 4.1.4 限流与速率控制

```yaml
annotations:
  # 单 IP 并发连接数
  nginx.ingress.kubernetes.io/limit-connections: "20"
  # 单 IP 每秒请求数
  nginx.ingress.kubernetes.io/limit-rps: "10"
  # 突发额度（桶容量）
  nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
  # 限流后返回的状态码，默认 503，通常改成 429 更语义化
  nginx.ingress.kubernetes.io/limit-rpm: "600"
  nginx.ingress.kubernetes.io/limit-whitelist: "10.0.0.0/8"
```

`limit-rps` 与 `limit-rpm` 同时配置时取更严格者。限流是**按 IP 维度**的，NAT 出口后的多个用户会共享同一个额度，容易造成误伤；对已登录用户做限流应基于用户维度，需要在应用层或 API 网关实现。

### 4.1.5 请求/响应大小与超时

```yaml
annotations:
  # 请求体大小上限，默认 1m，文件上传必调
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  # 代理缓冲区大小
  nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"
  # 关闭响应缓冲（SSE / WebSocket 场景必须）
  nginx.ingress.kubernetes.io/proxy-buffering: "off"
  # 建连 / 读 / 发送超时（秒）
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
  # upstream keepalive 空闲连接数
  nginx.ingress.kubernetes.io/upstream-keepalive-connections: "100"
```

`proxy-read-timeout` 默认 60 秒。长耗时接口（导出报表、大文件生成）超过 60 秒会被入口掐断并返回 504，即使后端仍在正常处理——排障时要同时看入口日志和后端日志，只看后端会误判为"后端没问题"。

### 4.1.6 负载均衡与 Session 保持

```yaml
annotations:
  # 负载均衡算法：round_robin（默认）/ least_conn / ip_hash / ewma
  nginx.ingress.kubernetes.io/load-balance: "ewma"
  # 基于 Cookie 的会话保持
  nginx.ingress.kubernetes.io/affinity: "cookie"
  nginx.ingress.kubernetes.io/session-cookie-name: "route"
  nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"
  nginx.ingress.kubernetes.io/session-cookie-secure: "true"
  nginx.ingress.kubernetes.io/session-cookie-samesite: "Lax"
```

`ewma`（指数加权移动平均）按后端延迟动态调整权重，对响应时间差异大的后端比 `round_robin` 更稳。会话保持会让流量分布不均，除非确实依赖本地状态（如本地缓存的会话），否则应优先让应用无状态化。

### 4.1.7 WebSocket 与长连接

```yaml
annotations:
  # WebSocket 默认已启用，需要同时调大读超时
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  # SSE 必须关闭缓冲，否则事件会被攒在缓冲区里不立刻下发
  nginx.ingress.kubernetes.io/proxy-buffering: "off"
  nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
```

WebSocket 握手依赖 `Upgrade`/`Connection` 头，需要 `proxy-http-version: "1.1"` 才能透传。SSE 的典型故障是"服务端日志显示已推送，浏览器却半天没有反应"——几乎总是 `proxy-buffering` 没关。

### 4.1.8 金丝雀发布

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  # 按权重切流
  nginx.ingress.kubernetes.io/canary-weight: "10"
  # 按请求头切流
  nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
  nginx.ingress.kubernetes.io/canary-by-header-value: "always"
  # 按 Cookie 切流
  nginx.ingress.kubernetes.io/canary-by-cookie: "canary_user"
```

金丝雀 Ingress 与稳定版 Ingress **共用同一个 host**，靠 annotation 区分。优先级是 header > cookie > weight。细节见第 8 节。

### 4.1.9 其他常用注解

```yaml
annotations:
  # 后端协议：HTTP / HTTPS / GRPC / GRPCS
  nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
  # 是否校验后端证书（自签证书时通常设为 false）
  nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
  # 自定义错误页
  nginx.ingress.kubernetes.io/custom-http-errors: "404,503"
  nginx.ingress.kubernetes.io/default-backend: "custom-error-svc"
  # 注入自定义 Nginx 配置片段（慎用，语法错误会导致 reload 失败）
  nginx.ingress.kubernetes.io/configuration-snippet: |
    more_set_headers "X-Served-By: ingress-nginx";
```

`configuration-snippet` 和 `server-snippet` 威力大但风险高：片段语法错误会让整份配置校验失败，**影响的是整个 Controller 上的所有 Ingress**，而不是单个服务。生产环境应配合准入策略（如 OPA/Gatekeeper）限制谁能写这类 annotation。

## 4.2 不同Ingress Controller的注解差异

### 4.2.1 功能对比表

同一件事，不同实现的写法完全不同：

| 能力 | Nginx Ingress | Traefik | Kong | APISIX |
|------|---------------|---------|------|--------|
| 路径重写 | `rewrite-target` | Middleware CRD | `konghq.com/rewrite` | `apisix.ingress.kubernetes.io/rewrite` |
| 限流 | `limit-rps` | Middleware CRD | RateLimitPlugin | `limit-count` |
| 认证 | `auth-url` / basic | Middleware CRD | Plugins | `authz-keycloak` |
| 长连接 | `proxy-*` 系列 | 默认支持 | 默认支持 | 默认支持 |
| 配置模型 | annotation 为主 | CRD 为主 | CRD + annotation | CRD + annotation |

可以看出，Nginx 的实现以 annotation 为唯一手段，而 Traefik/Kong/APISIX 都引入了 CRD。**CRD 是有类型的、可校验的、可复用的**，这正是 Gateway API 的方向。

### 4.2.2 Traefik 的 Middleware CRD

Traefik 把通用能力抽成可复用的 `Middleware` 对象：

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: production
spec:
  rateLimit:
    average: 100
    burst: 50
    period: 1s
```

然后在 Ingress 上引用：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: production
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: production-rate-limit@kubernetescrd
spec:
  # ...
```

好处是**一份 Middleware 可以被多个 Ingress 复用**——限流规格改了只需要改一个对象，而 Nginx 的 annotation 必须逐个 Ingress 修改。

### 4.2.3 注解带来的可移植性问题

annotation 模型的根本问题有三点：

- **无类型校验**。拼错一个字母，API Server 照样接受，只有 Controller 静默忽略。排障时要靠对比 Controller 日志和实际生效的配置。
- **无法复用**。同一组 CORS/限流配置在 20 个 Ingress 上要复制 20 遍，改一处漏一处。
- **强绑定实现**。`nginx.ingress.kubernetes.io/*` 换到 Traefik 就是一堆无效注解，迁移等于重写全部的入口配置。

这三点构成了 Gateway API 的设计动因，第 7 节展开。

# 5. 常见Ingress Controller

Ingress 是标准，Controller 是选择。这一节横向对比主流实现的架构特点与适用场景，第 5.6 节给出选型建议。

## 5.1 Nginx Ingress Controller

### 5.1.1 架构：基于 OpenResty

Nginx Ingress Controller 由一个 Go 编写的控制器和一个 OpenResty（Nginx + LuaJIT）数据平面组成，二者在同一个 Pod 里：

```
┌──────────────────────────────────────────────────────┐
│  Pod: ingress-nginx-controller                       │
│                                                      │
│  ┌─────────────────────┐      ┌──────────────────┐   │
│  │ controller (Go)     │      │ nginx (OpenResty)│   │
│  │ - Informer 监听 API │      │ - worker 进程    │   │
│  │ - 渲染 nginx.conf   │─────▶│ - 承载真实流量   │   │
│  │ - 触发 reload       │      │ - Lua 动态逻辑   │   │
│  └─────────────────────┘      └──────────────────┘   │
└──────────────────────────────────────────────────────┘
```

选择 OpenResty 而非原生 Nginx 的原因，是 Lua 让部分配置可以**运行时动态生效**而无需 reload：

- 证书热更新（`--enable-dynamic-certificates`）走 Lua 直接读 Secret，不再触发 reload。
- 动态 upstream（`balancer_by_lua`）让后端 Endpoint 变动不必重写配置文件。

这两项优化把大型集群里最频繁的两类 reload 消掉了，是 Nginx Ingress 能在几千个 Ingress 规模下可用的关键。

### 5.1.2 社区版 vs NGINX INC 版

| 维度 | kubernetes/ingress-nginx（社区版） | NGINX Ingress Controller（官方版） |
|------|-------------------------------------|-------------------------------------|
| 维护方 | Kubernetes SIG Network | F5 / NGINX Inc |
| 镜像 | `registry.k8s.io/ingress-nginx/controller` | `docker.io/nginx/nginx-ingress` |
| 定位 | 与 K8s 同步演进，事实标准 | 企业级支持、与 NGINX Plus 联动 |
| 配置方式 | annotation 为主 | annotation + 原生 NGINX 配置 |
| 支持 | 社区 | 商业订阅提供 SLA |

两者的 annotation 前缀相同（`nginx.ingress.kubernetes.io/`），但**并非所有注解都互相兼容**。迁移时不能只看前缀，要逐项核对。绝大多数场景选社区版即可。

### 5.1.3 部署方式

用 Helm 安装是最常见的做法：

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.replicaCount=3 \
  --set controller.metrics.enabled=true \
  --set controller.podAnnotations."prometheus\.io/scrape"=true \
  --set controller.service.type=LoadBalancer \
  --set controller.config.proxy-body-size="50m"
```

关键参数：

| 参数 | 作用 |
|------|------|
| `controller.replicaCount` | 副本数，生产至少 2–3 |
| `controller.service.type` | `LoadBalancer`（云）或 `NodePort`（自建） |
| `controller.hostNetwork` | `true` 时直接用节点端口，省一层转发 |
| `controller.config.*` | 全局 ConfigMap 配置，对所有权 Ingress 生效 |
| `controller.metrics.enabled` | 暴露 Prometheus 指标 |

自建机房没有云 LB，通常用 `NodePort` + 外部 HAProxy/LVS，或用 MetalLB 提供 `LoadBalancer` 语义：

```bash
# 检查入口地址是否已分配
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

这个命令返回空，说明 LB 还没分配地址——此时 Ingress 一定访问不通，问题在入口层而不是路由规则。

### 5.1.4 配置文件（ConfigMap）

`controller.config` 生成一个全局 ConfigMap，作用于该 Controller 上的**所有** Ingress：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # 全局请求体上限
  proxy-body-size: "50m"
  # 保留真实客户端 IP
  use-forwarded-headers: "true"
  # 计算真实 IP 时信任的代理网段
  compute-full-forwarded-for: "true"
  # 开启 brotli 压缩（比 gzip 更高效）
  enable-brotli: "true"
  # SSL 协议基线
  ssl-protocols: "TLSv1.2 TLSv1.3"
  # 日志格式，方便排查
  log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" '
```

全局配置与单 Ingress 的 annotation 冲突时，**annotation 优先**。因此排障时要同时看两处：

```bash
# 全局配置
kubectl get configmap -n ingress-nginx ingress-nginx-controller -o yaml
# 单服务覆盖的注解
kubectl get ingress <name> -o jsonpath='{.metadata.annotations}' | tr ',' '\n'
```

## 5.2 Traefik

### 5.2.1 核心特性

Traefik 的定位是"云原生边缘路由器"，三个特点：

- **配置全动态**。基于 CRD（`IngressRoute`、`Middleware`、`TLSOption`），不生成静态配置文件，没有 reload 概念。
- **自动服务发现**。除 Ingress 外，还直接支持通过 Service annotation 暴露服务，不强制经过 Ingress 抽象。
- **内置可观测性**。Web Dashboard、Prometheus 指标、OpenTelemetry 链路追踪开箱可用。

### 5.2.2 部署与配置

```bash
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set ingressRoute.dashboard.enabled=true \
  --set providers.kubernetesIngress.enabled=true \
  --set providers.kubernetesCRD.enabled=true
```

Traefik 同时提供 Ingress 与 CRD 两套 API。**Ingress 是兼容层，CRD 才是完整能力**：Ingress 只能表达路由，中间件必须靠 CRD + annotation 引用。

### 5.2.3 中间件示例

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: api-middleware
  namespace: production
spec:
  rateLimit:
    average: 100
    burst: 50
  headers:
    customResponseHeaders:
      X-Served-By: traefik
  circuitBreaker:
    expression: "NetworkErrorRatio() > 0.30"     # 错误率超 30% 熔断
```

`circuitBreaker` 是 Nginx Ingress 没有的能力——它需要在应用层或 Service Mesh 里实现。

### 5.2.4 Traefik 的弱点

- **生态与资料少于 Nginx**。大量排障经验、Snippet 都是 Nginx 语境，Traefik 遇到问题可参考的案例更少。
- **性能略逊于 Nginx**。Go 实现的吞吐上限低于 Nginx 的 C 实现，超高并发场景要实测。
- **只做 Ingress 时会损失部分能力**。仅用 Ingress API 时，Traefik 的优势（CRD 中间件）用不上。

## 5.3 HAProxy Ingress

### 5.3.1 核心优势

HAProxy 以稳定性和性能著称，适合"已有 HAProxy 运维经验"的团队：

- **极高的吞吐与稳定性**，在高并发短连接场景下表现突出。
- **配置更新优雅**，支持无缝 reload，长连接不中断。
- **丰富的负载均衡算法**（leastconn、first、source 等）与健康检查策略。

### 5.3.2 配置方式

HAProxy Ingress 通过 ConfigMap 提供全局配置，通过 annotation 做单服务覆盖：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-ingress
  namespace: haproxy-controller
data:
  # 全局默认后端超时
  timeout-server: "50s"
  # 默认负载均衡算法
  balance-algorithm: "leastconn"
```

配置表达式依赖模板变量（如 `{{ $svcName }}`），语法比 annotation 更晦涩，学习成本较高。

## 5.4 Kong Ingress Controller

### 5.4.1 核心优势

Kong 是 API 网关出身，因此它的 Ingress Controller 天然带"网关"能力：

- **插件生态丰富**。限流、认证、鉴权、日志、转换、Mock 等以插件形式提供，开箱即用。
- **`KongPlugin` CRD 可复用**。一个插件对象可被多个服务引用，改一处全生效。
- **多协议支持**。HTTP/gRPC/TCP/TLS 统一管理。
- **与 Kong Gateway 生态打通**。已有的 Kong 集群可以平滑接入 K8s。

代价是组件更多（Kong + Controller + 可能的 Postgres），运维复杂度高于纯代理方案。

## 5.5 APISIX Ingress Controller

### 5.5.1 核心特性

Apache APISIX 基于 OpenResty，用 etcd 存储配置，特点是**配置全动态、无 reload**：

- **动态配置**。新增 Ingress、改限流规则、换证书都通过 etcd 推送，毫秒级生效，不触发 reload——这是它在超大规模集群下的核心优势。
- **丰富插件**。80+ 内置插件，覆盖限流、认证、可观测性、协议转换。
- **多协议**。HTTP、gRPC、Dubbo、MQTT 等。
- **Dashboard 可视化**。

引入 etcd 意味着多一个要运维的有状态组件，这是选型时必须计入的运维成本。

## 5.6 各Controller对比与选型

### 5.6.1 综合对比表

| 维度 | Nginx Ingress | Traefik | HAProxy | Kong | APISIX |
|------|--------------|---------|---------|------|--------|
| 实现语言 | Go + Nginx(C) + Lua | Go | C | OpenResty(Lua) | Nginx(C) + Lua |
| 配置模型 | annotation | CRD 为主 | ConfigMap + annotation | CRD + annotation | CRD + annotation |
| 配置生效 | 部分需 reload | 动态 | 优雅 reload | 动态 | 动态（etcd） |
| 性能 | 极高 | 高 | 极高 | 极高 | 极高 |
| K8s 集成度 | 最高 | 高 | 中 | 高 | 高 |
| 社区规模 | 最大 | 大 | 中 | 中 | 快速增长 |
| 内置 Web UI | 无 | 有 | 无 | 有 | 有 |
| 插件/中间件 | 靠 annotation | Middleware CRD | ConfigMap snippet | KongPlugin CRD | Plugin CRD |
| gRPC 支持 | 有 | 有 | 有 | 有 | 有 |
| TCP/UDP 代理 | 有 | 有 | 有 | 无 | 有 |
| 学习曲线 | 中等 | 低 | 高 | 中高 | 中等 |
| 商业支持 | 否 | Traefik Enterprise | HAProxy Enterprise | Kong Konnect | API7.ai |

### 5.6.2 选型决策树

```
需要 API 网关能力（API Key、开发者门户）？
├─ 是 ──▶ Kong 或 APISIX
└─ 否 ──▶ 继续
     需要零 reload、自动服务发现、可视化管理？
     ├─ 是 ──▶ Traefik
     └─ 否 ──▶ 继续
          追求极致性能、已有 HAProxy 运维经验？
          ├─ 是 ──▶ HAProxy Ingress
          └─ 否 ──▶ Nginx Ingress Controller（最稳妥的选择）
```

一般推荐：**默认选择 Nginx Ingress Controller（社区版）**——文档最全、社区最大、稳定性最好、几乎所有教程和排障经验都基于它。中小团队、开发友好选 Traefik；已有 API 网关需求选 Kong 或 APISIX；高流量、性能关键选 HAProxy Ingress。

# 6. TLS/HTTPS

TLS 是现代 Web 的基础安全层。在 Kubernetes 中，管理 TLS 证书涉及 Secret 存储、自动签发、续期、配置等多个方面。

## 6.1 TLS Secret管理

### 6.1.1 手动创建 TLS Secret

最直接的方式是用 `kubectl create secret tls`：

```bash
# 使用 OpenSSL 生成自签名证书（仅用于测试）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=api.example.com/O=MyOrg"

# 创建 Kubernetes Secret
kubectl create secret tls example-tls \
  --key tls.key \
  --cert tls.crt \
  -n production
```

生产环境应当使用受信任 CA 签发的证书，并把私钥妥善保管——**私钥一旦进入 Git 历史就等于泄露**，即使用 `kubectl delete` 也删不掉历史记录。

### 6.1.2 证书链的重要性

TLS Secret 中的 `tls.crt` 必须是**完整的证书链**（从服务器证书到中间证书到根证书，根证书通常可省）：

```bash
# 检查证书链是否完整
kubectl get secret example-tls -n production \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl crl2pkcs7 -nocrl -certfile /dev/stdin | \
  openssl pkcs7 -print_certs -text -noout | grep "Subject:"
```

如果只放了服务器证书，缺中间证书会表现为：

| 客户端 | 现象 |
|--------|------|
| 桌面 Chrome/Firefox | 通常正常（靠 AIA 自动补全中间证书） |
| 手机 App / 部分移动浏览器 | 报错，无法建立连接 |
| `curl` / Java HttpClient | 报 `unable to get local issuer certificate` |

这是"电脑上好好的，手机上打不开"的经典根因，排查时不要只在自己浏览器里验证。

### 6.1.3 Secret 的命名空间隔离

TLS Secret 是命名空间级资源，**Ingress 只能引用同命名空间的 Secret**：

```
namespace: production          namespace: staging
  ┌────────────┐                 ┌────────────┐
  │ api-tls    │◀── 引用         │ api-tls    │
  │ Ingress    │                 │ (不会跨用)  │
  └────────────┘                 └────────────┘
```

因此多环境需要各自维护一份证书 Secret。用 cert-manager 可以让每套环境自动签发，避免手工复制。

### 6.1.4 证书格式要求

| 格式 | 用途 | 说明 |
|------|------|------|
| PEM | K8s Secret、Nginx | 文本格式，`-----BEGIN CERTIFICATE-----` 开头 |
| DER | Java keystore 转换 | 二进制，K8s 不直接支持 |
| PKCS#12 (.pfx) | Windows / Java | 需先转 PEM 再入库 |
| ECDSA 密钥 | 现代推荐 | 比 RSA 更短、更快、更安全 |

生成 ECDSA 密钥（推荐，密钥短、握手快）：

```bash
openssl ecparam -genkey -name prime256v1 -out tls.key
openssl req -new -x509 -key tls.key -out tls.crt -days 365 \
  -subj "/CN=api.example.com"
```

## 6.2 自动证书管理 (cert-manager + Let's Encrypt)

### 6.2.1 Let's Encrypt 简介

Let's Encrypt 是免费、自动化的 CA，通过 ACME 协议签发证书，有效期 90 天，因此**自动化续期是必须的**（手工续期必然会在某个周末过期）。cert-manager 是 K8s 上最主流的 ACME 客户端。

### 6.2.2 cert-manager 安装

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true \
  --version v1.14.0
```

必须开启 `installCRDs`，否则 `Certificate`、`Issuer`、`ClusterIssuer` 这些自定义资源不存在，后续 YAML 会被拒绝。

### 6.2.3 配置 Issuer 和 Certificate

第一步，创建 ACME 账户并声明 challenge 方式（`ClusterIssuer` 为集群级）：

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

第二步，声明需要哪张证书：

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls
  namespace: production
spec:
  secretName: api-tls-secret          # 签发结果写进这个 Secret
  duration: 2160h                     # 90 天
  renewBefore: 720h                   # 到期前 30 天开始续期
  dnsNames:
    - api.example.com
    - www.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

### 6.2.4 在 Ingress 中使用 cert-manager

实际使用时通常不手写 `Certificate`，而是让 cert-manager 直接读 Ingress 注解生成：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls-secret      # cert-manager 会创建它
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: api-svc, port: { number: 8080 } }
```

流程是：cert-manager 看到注解 → 创建 `Certificate` → 创建 `Challenge` → 通过 HTTP-01/DNS-01 完成校验 → 把证书写入 `api-tls-secret` → Ingress 立即生效。

排查签发失败：

```bash
kubectl get certificate -n production
kubectl describe certificate api-tls -n production     # 看 Events 与 Message
kubectl get challenge -n production
kubectl describe challenge <name> -n production
kubectl get order -n production
```

四层对象（Certificate → Order → Challenge → 自检）逐层往下看，报错信息就在最下面那层。

## 6.3 cert-manager架构

### 6.3.1 核心资源关系

```
┌──────────────────┐        ┌──────────────────┐
│  ClusterIssuer   │        │     Issuer       │
│  （集群级）       │        │  （命名空间级）    │
└────────┬─────────┘        └────────┬─────────┘
         │                           │
         └──────────┬────────────────┘
                    ▼ 声明使用
          ┌──────────────────┐
          │   Certificate    │  ← 期望：为哪些域名签发证书
          └────────┬─────────┘
                   ▼ 触发
          ┌──────────────────┐
          │      Order       │  ← ACME 一次订单
          └────────┬─────────┘
                   ▼ 校验
          ┌──────────────────┐
          │    Challenge     │  ← HTTP-01 / DNS-01 验证
          └────────┬─────────┘
                   ▼ 签发成功
          ┌──────────────────┐
          │  Secret(tls)     │  ← 证书落盘，Ingress 引用
          └──────────────────┘
```

### 6.3.2 Issuer vs ClusterIssuer

```yaml
# Issuer（命名空间级）：只能签本命名空间的证书
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt
  namespace: production
```

```yaml
# ClusterIssuer（集群级）：可被所有命名空间引用
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
```

选择标准：多命名空间共享同一套 ACME 账号用 `ClusterIssuer`；需要按租户隔离、各自用不同账号或不同 CA 用 `Issuer`。注意 `ClusterIssuer` 的 ACME 私钥存在 cert-manager 自己的命名空间里，属于集群级凭据。

### 6.3.3 Certificate 生命周期

| 字段 | 作用 | 建议值 |
|------|------|--------|
| `duration` | 证书有效期 | 2160h（90 天，与 Let's Encrypt 一致） |
| `renewBefore` | 提前多久续期 | 720h（30 天） |
| `usages` | 用途 | `digital signature, key encipherment` |
| `secretName` | 落盘的 Secret | 与 Ingress `tls.secretName` 一致 |

`renewBefore` 必须小于 `duration`。过小的续期窗口会在 CA 临时不可用时导致证书过期；过大的窗口会让续期频率变高，浪费 ACME 的速率配额（Let's Encrypt 限制同一域名每周 50 张证书）。

## 6.4 HTTP-01和DNS-01 Challenge

ACME 需要证明"你确实控制这个域名"，两种验证方式各有适用场景。

### 6.4.1 HTTP-01 Challenge

CA 请求 `http://<域名>/.well-known/acme-challenge/<token>`，期望返回特定内容：

```
1. cert-manager 创建一个临时 Ingress/Pod 承载 token
        │
        ▼
2. Let's Encrypt 从外部访问 http://api.example.com/.well-known/acme-challenge/xxx
        │
        ▼
3. 校验通过 → 签发证书
```

约束：

- **域名必须能从公网解析到本集群入口**。内网域名、还没配 DNS 的域名无法使用。
- **必须走 80 端口**。如果入口强制 HTTPS 跳转，就要给 `/.well-known/acme-challenge` 单独放行，否则会被 301 挡住（自锁）。
- **不支持泛域名**。`*.example.com` 不能用 HTTP-01。

### 6.4.2 DNS-01 Challenge

CA 查询 DNS 记录 `_acme-challenge.<域名>` 的 TXT 值：

```
1. cert-manager 通过云厂商 API 写入 TXT 记录
        │   _acme-challenge.example.com = "随机token"
        ▼
2. Let's Encrypt 查询该 TXT 记录
        │
        ▼
3. 校验通过 → 签发证书（含泛域名）
```

优势是**支持泛域名**、**不要求域名指向集群**，因此内网域名也能签发。代价是需要在云厂商侧配置 API 凭据：

```yaml
solvers:
  - dns01:
      route53:
        region: us-east-1
        # 凭据通过 IRSA / Secret 注入，不要硬编码 AK/SK
```

### 6.4.3 Challenge 选型指南

| 维度 | HTTP-01 | DNS-01 |
|------|---------|--------|
| 泛域名 | 不支持 | 支持 |
| 域名需公网可达 | 需要 | 不需要 |
| 依赖 | 入口可访问 80 端口 | 云厂商 DNS API 权限 |
| 自动化复杂度 | 低 | 中（需配凭据） |
| 生效速度 | 快 | 慢（等 DNS 传播） |
| 适用场景 | 单一域名、公网服务 | 泛域名、内网域名、多域名 |

## 6.5 TLS版本与密码套件配置

### 6.5.1 安全基线

在 Nginx Ingress 的全局 ConfigMap 里配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"
  ssl-prefer-server-ciphers: "true"
  ssl-session-cache: "true"
  hsts: "true"
  hsts-max-age: "31536000"
  hsts-include-subdomains: "true"
```

**必须禁用 TLS 1.0/1.1**（RFC 8996 已弃用）。检查方法：

```bash
# 确认服务端不再接受 TLS 1.0/1.1
nmap --script ssl-enum-ciphers -p 443 api.example.com
# 或
openssl s_client -connect api.example.com:443 -tls1_1
# 期望：handshake failure
```

HSTS 一旦下发就无法回退——浏览器会强制该域名走 HTTPS 一年。上线前确认所有子域都已支持 HTTPS，否则会把用户锁在门外。

### 6.5.2 TLS 1.3 的优势

- **握手只需 1-RTT**（会话恢复时 0-RTT），比 1.2 少一个往返，移动网络下延迟改善明显。
- **只保留 AEAD 密码套件**，把 CBC、RC4 等有历史漏洞的算法彻底移除。
- **默认前向保密**，密钥泄露也不影响历史流量。

TLS 1.3 的密码套件无法单独配置（套件与密钥交换绑定），只有 `TLS_AES_128_GCM_SHA256` 等少数几个，因此配置 `ssl-ciphers` 只影响 1.2 及以下。

### 6.5.3 密码套件解析

```
ECDHE-ECDSA-AES128-GCM-SHA256
  │      │     │       │      │
  │      │     │       │      └─ 哈希算法：SHA256
  │      │     │       └─ 加密模式：GCM（AEAD，认证加密）
  │      │     └─ 对称密钥长度：128 位
  │      └─ 身份认证算法：ECDSA 证书
  └─ 密钥交换：ECDHE（临时椭圆曲线，提供前向保密）
```

选套件时优先 ECDHE（前向保密）+ GCM（AEAD），保留少量 CBC 套件兼容老旧客户端即可。

### 6.5.4 Mozilla SSL Configuration Generator

不必手写套件列表，Mozilla 的配置生成器按兼容性分三档：

| 档位 | 兼容性 | 说明 |
|------|--------|------|
| Modern | 仅现代浏览器 | 只开 TLS 1.3，最安全 |
| Intermediate | 主流浏览器 + 部分老客户端 | 开 TLS 1.2 + 1.3，**推荐** |
| Old | 兼容 IE11 等 | 需开 TLS 1.0，不建议 |

用 Intermediate 档的套件列表替换 `ssl-ciphers` 即可，这是兼顾安全与兼容的默认选择。

# 7. Gateway API

Gateway API 是 Ingress 的官方继任者，由 SIG Network 维护，目标是解决 Ingress 在角色分离、协议扩展、配置复用上的结构性缺陷。它不是 Ingress 的补丁，而是一次重新设计。

## 7.1 Ingress的局限性

### 7.1.1 注解混乱（Annotation Hell）

Ingress 的标准字段只覆盖路由，其余能力全靠 annotation。不同实现各有一套前缀，同一件事有 N 种写法：

```yaml
# 同一个"限流"需求，不同实现完全不同
nginx.ingress.kubernetes.io/limit-rps: "10"       # Nginx
traefik.ingress.kubernetes.io/rate-limit-average  # Traefik（示意）
konghq.com/plugins: rate-limiting                 # Kong
```

由此产生三个问题：**无类型校验**（拼错不报错）、**无法复用**（20 个 Ingress 复制 20 遍）、**无法移植**（换 Controller 等于重写）。

### 7.1.2 缺乏角色分离（Lack of Role-Oriented Design）

一个 Ingress 对象同时包含了三种角色的关注点：

| 角色 | 关心什么 | 在 Ingress 里的位置 |
|------|---------|-------------------|
| 集群管理员 | 入口用哪个 LB、监听哪些端口、TLS 基线 | `ingressClassName`、annotation |
| 平台/网关运维 | 域名分配、证书、共享策略 | `spec.tls`、`spec.rules[].host` |
| 应用开发者 | 我的服务挂哪个路径 | `spec.rules[].http.paths[]` |

三者挤在同一个对象里，导致**改一个路径要动整个 Ingress**。RBAC 也做不了细粒度控制：想让开发者只能改自己服务的路径，就必须把整个 Ingress 的写权限给他，连带能改 TLS 和域名。

### 7.1.3 仅限 HTTP/HTTPS（L7 Only）

Ingress 只能表达 HTTP 语义（`host`、`path`、`http`）。要暴露 TCP/UDP 服务（数据库、MQTT、游戏服务），只能退回 `LoadBalancer`/`NodePort`，无法复用统一的入口和证书管理。

### 7.1.4 不能跨命名空间共享

Ingress 只能引用同命名空间的 Service 和 Secret。多团队共享一个入口时，每个团队都要自建 Ingress，域名与证书管理被割裂，且没有任何机制能约束"谁可以占用哪个域名"。

### 7.1.5 缺乏高级流量管理

Ingress 标准里没有流量切分、重试、超时、镜像流量等概念，这些能力只能靠 annotation 实现，且各实现语义不一致。渐进式发布（金丝雀、蓝绿）因此需要依赖 Argo Rollouts、Flagger、Istio 等外部工具，增加了架构复杂度。

## 7.2 Gateway API设计理念

### 7.2.1 面向角色（Role-Oriented）

Gateway API 把 Ingress 的一个对象拆成三个，每个对应一个角色：

```yaml
# 集群管理员：定义"入口基础设施"
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: k8s.io/ingress-nginx
---
# 平台/网关运维：创建一个具体的入口（LB + 监听端口 + 证书）
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: infra
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-tls
---
# 应用开发者：声明"我的服务走哪个路径"
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: order-route
  namespace: shop
spec:
  parentRefs:
    - name: prod-gateway
      namespace: infra
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /orders
      backendRefs:
        - name: order-svc
          port: 8080
```

这样 RBAC 可以精确切分：开发者只有 `HTTPRoute` 的写权限，改不了 Gateway（LB 和证书），也改不了别人的命名空间。**域名占用**也可以通过 Gateway 的 `hostname` + `allowedRoutes` 约束。

### 7.2.2 可移植（Portable）

Gateway API 是**标准 CRD**，不是某家实现的私有扩展。`HTTPRoute` 的字段在所有实现（Nginx、Traefik、Istio、Envoy Gateway、云厂商）里语义一致，因此：

- 换实现不需要改应用侧的 `HTTPRoute`。
- 高级能力（权重切流、请求头改写、重试）进入标准字段，不再靠 annotation 私有约定。

### 7.2.3 可扩展（Extensible）

需要实现特有功能时，用 `*Route` 的 `filters` 引用实现自定义的 CRD（如 `ExtensionRef`），而不是污染标准字段。扩展是显式的、可发现的，而不是散落在 annotation 字符串里。

## 7.3 核心资源模型 (GatewayClass/Gateway/HTTPRoute)

### 7.3.1 GatewayClass

对应 Ingress 的 `IngressClass`，是集群级资源，由实现方提供：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: k8s.io/ingress-nginx    # 哪个控制器认领它
  parametersRef:                           # 可选：实现特有参数
    group: ""
    kind: ConfigMap
    name: nginx-gw-params
    namespace: infra
```

`controllerName` 由实现定义，与 IngressClass 的 `controller` 同义。

### 7.3.2 Gateway

表示一个具体的入口实例，承载监听器（listener）定义：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: infra
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:                   # 允许哪些命名空间的 Route 挂上来
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate                # 在网关上终结 TLS
        certificateRefs:
          - kind: Secret
            name: wildcard-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: "true"
```

关键设计是 `allowedRoutes`：**Gateway 决定谁能把路由挂到自己身上**。这是跨命名空间共享的授权机制，取代了 Ingress 里"只能同命名空间"的硬限制。

`tls.mode` 有三种：`Terminate`（网关终结）、`Passthrough`（透传到后端，SNI 路由）、`Terminate + 后端 mTLS`。

### 7.3.3 HTTPRoute

应用侧的核心对象，通过 `parentRefs` 挂到 Gateway 上：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-route
  namespace: shop
spec:
  parentRefs:
    - name: prod-gateway
      namespace: infra
      sectionName: https               # 可选：指定挂到哪个 listener
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /orders
        - headers:
            - name: X-Api-Version
              value: v2
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            set:
              - name: X-Routed-By
                value: gateway
      backendRefs:
        - name: order-v2
          port: 8080
```

`sectionName` 让一个 HTTPRoute 挂到 Gateway 的特定监听器上，同一份路由定义可以复用在不同入口。

## 7.4 HTTPRoute详解

### 7.4.1 路由匹配规则（Matches）

`matches` 支持三类条件的**与**组合：

```yaml
matches:
  - path:
      type: PathPrefix        # Exact / PathPrefix / RegularExpression
      value: /api
    method: GET               # HTTP 方法
    queryParams:              # 查询参数
      - name: version
        value: v2
    headers:                  # 请求头
      - name: X-Canary
        value: "true"
```

与 Ingress 相比，Gateway API **原生支持按请求头、查询参数、HTTP 方法匹配**，而且支持 `RegularExpression` 路径类型作为标准语义，不需要 `use-regex` 这种实现特有的开关。

一个 `rules` 内的多个 `matches` 是**或**关系，命中任意一个即可；`rules` 之间按顺序求值，第一条命中即返回。

### 7.4.2 过滤器（Filters）

过滤器描述"匹配之后对请求/响应做什么"，是标准字段：

```yaml
filters:
  # 请求头改写
  - type: RequestHeaderModifier
    requestHeaderModifier:
      set:
        - name: X-Forwarded-Host
          value: shop.example.com
      remove: [X-Internal-Token]

  # 响应头改写
  - type: ResponseHeaderModifier
    responseHeaderModifier:
      add:
        - name: X-Content-Type-Options
          value: nosniff

  # URL 重定向
  - type: RequestRedirect
    requestRedirect:
      scheme: https
      statusCode: 301

  # URL 重写
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /

  # 请求镜像（把流量复制一份到新后端，用于影子测试）
  - type: RequestMirror
    requestMirror:
      backendRef:
        name: order-v3-shadow
        port: 8080

  # 引用实现特有的扩展 CRD
  - type: ExtensionRef
    extensionRef:
      group: traefik.io
      kind: Middleware
      name: rate-limit
```

`RequestMirror` 是 Ingress 完全不具备的能力——可以把生产流量复制给新版本做对照验证，而不影响真实响应。

### 7.4.3 流量分割（BackendRefs + Weight）

`backendRefs` 的 `weight` 是标准字段，让金丝雀发布不再依赖 annotation：

```yaml
backendRefs:
  - name: order-v1
    port: 8080
    weight: 90          # 90% 流量
  - name: order-v2
    port: 8080
    weight: 10          # 10% 流量
```

这是 Gateway API 相对 Ingress 最实用的改进之一：**渐进式发布成为第一等公民**，不需要 Canary Ingress + annotation 那套约定。

## 7.5 其他Route类型

### 7.5.1 TLSRoute

用于 SNI 路由和 TLS 透传，目标是非 HTTP 的 TLS 服务：

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: db-tls
spec:
  parentRefs:
    - name: prod-gateway
      sectionName: tls-passthrough
  hostnames:
    - db.example.com
  rules:
    - backendRefs:
        - name: postgres-svc
          port: 5432
```

### 7.5.2 TCPRoute

转发原始 TCP，适用于数据库、Redis、MQTT 等：

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: redis-tcp
spec:
  parentRefs:
    - name: prod-gateway
      sectionName: redis
  rules:
    - backendRefs:
        - name: redis-svc
          port: 6379
```

### 7.5.3 UDPRoute

UDP 转发，用于 DNS、游戏服务：

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: UDPRoute
metadata:
  name: dns-udp
spec:
  parentRefs:
    - name: prod-gateway
      sectionName: dns
  rules:
    - backendRefs:
        - name: coredns-svc
          port: 53
```

### 7.5.4 GRPCRoute

gRPC 原生路由，可以按 service/method 匹配，不再需要靠 HTTP path 模拟：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: order-grpc
spec:
  parentRefs:
    - name: prod-gateway
  hostnames:
    - grpc.example.com
  rules:
    - matches:
        - method:
            service: order.OrderService
            method: GetOrder
      backendRefs:
        - name: order-grpc-svc
          port: 9090
```

四种 Route 类型覆盖了 L4 与 L7、HTTP 与非 HTTP，这是 Ingress "仅限 L7 HTTP"限制的正解。

## 7.6 Gateway API vs Ingress 对比

### 7.6.1 全面对比

| 维度 | Ingress | Gateway API |
|------|---------|-------------|
| 角色分离 | 单对象混合三种角色 | GatewayClass / Gateway / HTTPRoute 分离 |
| 扩展方式 | annotation（无类型） | 标准字段 + ExtensionRef（CRD） |
| 协议 | 仅 HTTP/HTTPS | HTTP、HTTPS、TLS、TCP、UDP、gRPC |
| 跨命名空间 | 不支持 | `allowedRoutes` + `ReferenceGrant` 授权 |
| 流量切分 | 靠 annotation | `backendRefs.weight` 标准字段 |
| 请求头/方法匹配 | 靠正则或 annotation | `matches.headers/method/queryParams` |
| 请求镜像 | 不支持 | `RequestMirror` filter |
| 配置复用 | 复制 annotation | 路由/过滤器对象可复用 |
| 可移植性 | 实现私有 annotation | 标准 CRD，语义一致 |
| 成熟度 | 生产验证充分 | v1 已 GA（HTTPRoute/Gateway/GatewayClass） |

### 7.6.2 迁移策略

Gateway API 与 Ingress **可以在同一集群共存**，因此不要求一次性切换。推荐路径：

1. **先建 Gateway**。部署支持 Gateway API 的 Controller（较新版本的 Nginx Ingress、Traefik 3.x、Envoy Gateway、Istio 均支持），创建 GatewayClass 与 Gateway。
2. **新服务直接写 HTTPRoute**。不要让新流量再产生新的 Ingress。
3. **存量 Ingress 按域名/团队分批迁移**。一次迁一个 host，用 DNS 或同一 Gateway 的不同 listener 灰度。
4. **验证等价性**。迁移前后对比同一组请求的响应头、状态码、路由落点，尤其注意路径匹配语义的细微差异（Ingress 的 `Prefix` 与 HTTPRoute 的 `PathPrefix` 在边界情况上不完全等价）。
5. **最后删除 Ingress**。

注意：`ReferenceGrant` 是跨命名空间引用 Service/Secret 时必须额外创建的对象，否则 `backendRefs` 指向别的命名空间会被拒绝。

## 7.7 Gateway API生态与实现

### 7.7.1 主要实现

| 实现 | 维护方 | 特点 |
|------|--------|------|
| ingress-nginx | Kubernetes SIG | 新版本已支持 Gateway API，社区最大 |
| Traefik 3.x | Traefik Labs | 原生支持，配置全动态 |
| Envoy Gateway | CNCF | 基于 Envoy，专为 Gateway API 设计 |
| Istio | Istio/CNCF | 用 Gateway API 替代原有 Gateway 资源 |
| Cilium | Isovalent | 基于 eBPF，网络层集成 |
| 云厂商 | AWS/GCP/Azure | 托管实现，与自家 LB 打通 |

### 7.7.2 当前状态与选型

- **核心资源已 GA**：`GatewayClass`、`Gateway`、`HTTPRoute` 自 v1.0（2023-10）起为稳定版，可以放心用于生产。
- **扩展资源仍在演进**：`TLSRoute`、`TCPRoute`、`UDPRoute` 还在 `v1alpha2`，接口可能变化，用于生产需评估锁定版本。
- **实现能力参差**：不同实现覆盖的字段范围不同，选型时要核对官方兼容性矩阵，尤其注意 `filters` 的支持情况。

**实践建议**：已有 Nginx Ingress 且运行稳定，不必急于迁移；新建集群或正在做入口重构，直接上 Gateway API。核心 HTTP 场景已经足够成熟。

# 8. 金丝雀发布与流量管理

金丝雀发布要解决的是"新版本上线时，只让一小部分流量打到新版本，确认无异常后再逐步放大"。Kubernetes 的 Deployment 本身只支持按副本比例的粗粒度滚动，真正的按流量比例切分要靠入口层。

## 8.1 Ingress层实现金丝雀

### 8.1.1 Nginx Ingress Canary 注解

做法是**为同一个 host 再建一个 Canary Ingress**，用 annotation 声明它承担多少流量：

```yaml
# 稳定版 Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-stable
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: order-v1
                port:
                  number: 8080
---
# 金丝雀 Ingress：同 host，靠 annotation 区分
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-canary
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"      # 10% 流量
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: order-v2
                port:
                  number: 8080
```

两个 Ingress **必须使用完全相同的 host 和 path**，否则会变成两条互相独立的路由而不是切流。

### 8.1.2 Canary 流量分发策略

Nginx Ingress 提供的三种策略优先级不同：

| 注解 | 优先级 | 说明 |
|------|--------|------|
| `canary-by-header` | 最高 | 带指定请求头走金丝雀，适合内部测试 |
| `canary-by-header-value` | 高 | 配合 header 使用，指定具体的匹配值 |
| `canary-by-cookie` | 中 | 带指定 Cookie 走金丝雀，适合定向放量 |
| `canary-weight` | 最低 | 按百分比随机切分 |

```yaml
# 内部同学带 X-Canary: always 走新版本，外部用户不受影响
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
  nginx.ingress.kubernetes.io/canary-by-header-value: "always"
```

常用的渐进节奏是：先用 header 让内部 QA 验证 → 再放开 1% → 5% → 20% → 50% → 100%。每一档停留时间要覆盖业务的一个完整周期（比如订单系统至少跨一个结算周期）。

### 8.1.3 Ingress Canary 的局限性

- **按连接/请求随机切分，不保证会话一致**。同一个用户的多次请求可能一部分走 v1、一部分走 v2，对依赖会话的应用会出问题。需要配合 `canary-by-cookie`。
- **没有自动回滚**。权重调整全靠人工或外部工具，异常时不能自动退回。
- **不支持指标驱动的渐进**。无法"根据错误率自动决定是否继续放量"。
- **多路径场景笨重**。一个服务有 5 条 path，就要写 5 个 Canary Ingress。

这些正是 Gateway API 权重字段和 Argo Rollouts 要解决的问题。

## 8.2 Gateway API流量分割

Gateway API 把切流做成标准字段，单个 HTTPRoute 内即可完成，不需要第二个对象：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: order-route
  namespace: shop
spec:
  parentRefs:
    - name: prod-gateway
      namespace: infra
  hostnames:
    - shop.example.com
  rules:
    # 内部流量：带 header 的全走 v2
    - matches:
        - headers:
            - name: X-Canary
              value: always
      backendRefs:
        - name: order-v2
          port: 8080
    # 其余流量：按权重 90/10 切分
    - backendRefs:
        - name: order-v1
          port: 8080
          weight: 90
        - name: order-v2
          port: 8080
          weight: 10
```

相比 Canary Ingress，优势是：**一个对象管完**、规则顺序明确（header 命中优先）、权重是标准字段不会被实现私有语义影响。

```yaml
# 按用户代理做权重 + 定向的组合（进阶用法）
rules:
  - matches:
      - headers:
          - name: User-Agent
            type: RegularExpression
            value: ".*(canary|internal).*"
    backendRefs:
      - name: order-v2
        port: 8080
  - backendRefs:
      - name: order-v1
        port: 8080
        weight: 95
      - name: order-v2
        port: 8080
        weight: 5
```

## 8.3 与Argo Rollouts/Istio配合

### 8.3.1 Argo Rollouts

Argo Rollouts 用 `Rollout` 替代 `Deployment`，把"放量决策"交给控制器自动执行：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: order-canary
      stableService: order-stable
      trafficRouting:
        nginx:
          stableIngress: order-stable      # 由它自动改 Canary Ingress 的权重
      steps:
        - setWeight: 10                    # 先放 10%
        - pause: { duration: 5m }          # 观察 5 分钟
        - setWeight: 30
        - pause: { duration: 10m }
        - setWeight: 100
      analysis:
        templates:
          - templateName: success-rate     # 关联指标校验模板
        startingStep: 1
```

关键能力是 **analysis + 自动回滚**：控制器按 Prometheus 查询判断成功率/延迟是否达标，不达标立即把权重退回 0。这把"人工盯盘"变成了自动化的发布门禁。

配合 Nginx Ingress 时，Argo Rollouts 通过 `trafficRouting.nginx` 自动创建并调整 Canary Ingress 的权重，业务侧不需要手写 Canary 对象。

### 8.3.2 金丝雀发布最佳实践

1. **必须有可量化的门禁指标**。至少覆盖请求成功率、P95 延迟、业务核心指标（下单成功率、支付成功率）。只看 CPU/内存没有意义。
2. **放量节奏与业务周期匹配**。低峰期发布的观察窗口要覆盖到高峰，否则问题会在放量后才暴露。
3. **做好流量标记与追溯**。给响应注入版本头（如 `X-Version: v2`），出问题能立刻定位是哪个版本，也能验证流量比例是否符合预期。
4. **会话一致性优先**。有状态服务必须用 Cookie 或 header 维度切流，不能纯靠权重。
5. **回滚要能在分钟级完成**。回滚路径提前演练过，并确认数据库变更可回退（不做破坏性 schema 变更，用向前兼容的迁移）。
6. **权重与副本数解耦**。金丝雀 Pod 的副本数不必与流量比例一致，但要保证小流量下也有足够样本量，不能被一两个请求放大成假信号。

# 9. 常见问题与排查

入口层的故障有个共性：**报错信息往往在入口，根因却在下游**。这一节按现象组织，给出逐层排查的路径。

## 9.1 404/503错误排查

### 9.1.1 404 Not Found

404 表示请求到达了入口，但**没有匹配到任何路由规则**（或落到了 defaultBackend）：

```bash
# 1. 检查 Ingress 是否存在、是否属于预期的 IngressClass
kubectl get ingress -n <namespace>
kubectl describe ingress <name> -n <namespace>   # 看 Class 与 Events

# 2. 查看 Ingress 详细配置，确认 host 与 path 写法
kubectl get ingress <name> -n <namespace> -o yaml

# 3. 检查 host 和 path 是否与请求一致
#    常见错误：host 少了泛域名前缀、path 缺前导斜杠、pathType 写错

# 4. 检查后端 Service 是否存在且端口匹配
kubectl get svc <svc-name> -n <namespace>

# 5. 查看 Controller 日志，确认请求落在了哪个 location
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100

# 6. 检查生成的 nginx.conf，确认规则真的被渲染进去了
kubectl exec -n ingress-nginx <controller-pod> -- cat /etc/nginx/nginx.conf | grep -A5 "server_name"
```

高频原因排序：**`ingressClassName` 不匹配**（Ingress 被晾着）> **host 写错**（含泛域名层级）> **path 缺前导 `/`** > **pathType 与实际请求不符**。

### 9.1.2 503 Service Unavailable

503 表示**路由匹配到了，但后端不可用**——通常是 Service 没有健康的 Endpoints：

```bash
# 1. 检查 Service Endpoints 是否为空
kubectl get endpoints <svc-name> -n <namespace>
# 如果 ENDPOINTS 为空，说明问题在 Service-Pod 链路：
#   - selector 与 Pod label 不匹配
#   - Pod 没有 Ready（readinessProbe 失败）

# 2. 检查 Pod 状态和就绪状态
kubectl get pods -n <namespace> -l app=<label>

# 3. 直接从 Pod 内测试连通性，排除网络策略问题
kubectl exec -n <namespace> <pod-name> -- curl -v localhost:<port>/healthz

# 4. 确认 Ingress Controller 是否可达 Service
kubectl exec -n ingress-nginx <controller-pod> -- \
  curl -v http://<svc-name>.<namespace>.svc.cluster.local:<port>/

# 5. 查看 Controller 日志中的 upstream 错误
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep -i "upstream\|connect"
```

如果 Endpoints 正常但仍 503，检查 `targetPort` 是否写错——Service 的 `port` 能通但不代表 `targetPort` 是容器真正监听的端口，这是很常见的一处笔误。

## 9.2 TLS证书不生效

```bash
# 1. 确认 Secret 是否存在且格式正确
kubectl get secret <secret-name> -n <namespace> -o yaml
kubectl describe secret <secret-name> -n <namespace>   # 应包含 tls.crt 与 tls.key

# 2. 验证证书内容
kubectl get secret <secret-name> -n <namespace> \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text | head -20

# 3. 检查证书是否过期
kubectl get secret <secret-name> -n <namespace> \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -dates -noout

# 4. 验证私钥和证书是否匹配（两个 MD5 值相同则表示匹配）
kubectl get secret <secret-name> -n <namespace> \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -modulus | openssl md5
kubectl get secret <secret-name> -n <namespace> \
  -o jsonpath='{.data.tls\.key}' | base64 -d | openssl rsa -noout -modulus | openssl md5

# 5. 使用 OpenSSL 测试 TLS 握手，确认返回的是预期证书
echo | openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates

# 6. 检查是否有多个 TLS Secret 竞争同一个域名
kubectl get ingress -A -o json | grep -i secretName
```

最隐蔽的一类问题是**同一域名被两个 Ingress 声明了不同的 `tls.secretName`**，Nginx 随机选一个 server 块，表现为"证书时好时坏"。排查时要 `-A` 全命名空间搜索。

## 9.3 Ingress Controller不reload

现象是改了 Ingress，但 `nginx.conf` 没变、行为没变。按顺序排查：

```bash
# 1. 权限不足：检查 Controller 的 RBAC
kubectl auth can-i list ingresses --as=system:serviceaccount:ingress-nginx:ingress-nginx
kubectl auth can-i get secrets --as=system:serviceaccount:ingress-nginx:ingress-nginx

# 2. ConfigMap 配置错误（比如 snippet 语法错误）导致 reload 失败
kubectl get configmap -n ingress-nginx ingress-nginx-controller -o yaml
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep -i "error\|invalid"

# 3. Nginx 配置语法错误（configuration-snippet/server-snippet 引起）
kubectl exec -n ingress-nginx <controller-pod> -- nginx -t

# 4. 检查 Controller 资源使用是否 OOM（OOM 后重启，配置丢失）
kubectl top pod -n ingress-nginx
kubectl describe pod -n ingress-nginx <controller-pod> | grep -A5 "Last State"

# 5. 检查同步周期配置（默认会定期 resync）
kubectl get deploy -n ingress-nginx ingress-nginx-controller -o yaml | grep sync-period

# 6. 查看 Controller 是否正常 Watch
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep -i "watch\|informer"
```

**`configuration-snippet` 语法错误是最常见的原因**：reload 失败时 Nginx 会继续用旧配置运行，所以表现为"配置完全没生效"。这也是为什么生产环境应当用准入控制器限制谁能写 snippet。

## 9.4 排查命令与工具

### 9.4.1 基础排查命令清单

```bash
# ---- Ingress 状态检查 ----
kubectl get ingress --all-namespaces                     # 总览
kubectl get ingress <name> -n <ns> -o wide               # 查看地址是否已分配
kubectl describe ingress <name> -n <ns>                  # 看 Events

# ---- Service 和 Endpoints 检查 ----
kubectl get svc,endpoints -n <ns>
kubectl get endpointslice -n <ns> -l kubernetes.io/service-name=<svc>

# ---- Pod 状态和日志 ----
kubectl get pods -n <ns> -o wide
kubectl logs -n <ns> <pod> --tail=200

# ---- Ingress Controller 日志 ----
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep "GET /api"

# ---- 检查 IngressClass ----
kubectl get ingressclass
kubectl describe ingressclass nginx

# ---- 检查 TLS Secret ----
kubectl get secret -n <ns> | grep tls

# ---- RBAC 权限检查 ----
kubectl auth can-i --list --as=system:serviceaccount:ingress-nginx:ingress-nginx

# ---- 测试 Ingress 规则（绕过 DNS，直接用 Host 头）----
curl -v -H "Host: api.example.com" http://<ingress-lb-ip>/users

# ---- DNS 检查 ----
nslookup api.example.com
dig api.example.com +short
```

`curl -H "Host: ..."` 直连入口 IP 是最高效的验证手段：它把 DNS 解析排除在外，能立刻区分"是 DNS 没生效"还是"路由规则有问题"。

### 9.4.2 高级诊断工具

```bash
# 获取 Controller Pod 名称
POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o name | head -1)

# 查看生成的 nginx.conf
kubectl exec -n ingress-nginx $POD -- cat /etc/nginx/nginx.conf

# 测试 nginx 配置语法
kubectl exec -n ingress-nginx $POD -- nginx -t

# 测试到后端 Service 的连通性
kubectl exec -n ingress-nginx $POD -- curl -v http://order-svc.default.svc.cluster.local:8080/

# 在 Pod 内模拟带 Host 头的真实请求
kubectl exec -n ingress-nginx $POD -- curl -v -H "Host: api.example.com" http://127.0.0.1/users

# 查看 nginx 的 upstream 状态（需开启 status 模块）
kubectl exec -n ingress-nginx $POD -- curl -s http://127.0.0.1:10254/healthz
```

其他有用的工具：`kubectl debug` 挂临时容器进 Controller Pod、`kubectl trace`（eBPF）抓包、`istioctl analyze`（如果同时用 Istio）。

### 9.4.3 监控与告警

Nginx Ingress Controller 暴露 Prometheus 指标，按类别关注：

```promql
# ---- Nginx Ingress Controller 核心指标 ----
# 请求总量（按状态码、Host、Path 分组）
sum(rate(nginx_ingress_controller_requests[5m])) by (status, host, path)

# 请求延迟 P95
histogram_quantile(0.95,
  sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by (le, host))

# 活跃连接数
nginx_ingress_controller_nginx_process_connections{state="active"}

# 配置重载次数（异常增长说明配置频繁变动或有问题）
increase(nginx_ingress_controller_config_last_reload_successful_timestamp_seconds[1h])

# ---- 错误指标 ----
# 5xx 比例
sum(rate(nginx_ingress_controller_requests{status=~"5.."}[5m]))
  / sum(rate(nginx_ingress_controller_requests[5m])) > 0.05

# 上游连接失败
rate(nginx_ingress_controller_upstream_peer_connection_failures[5m])
```

告警规则示例：

```yaml
groups:
  - name: ingress
    rules:
      # 503 错误率告警
      - alert: High503Rate
        expr: |
          sum(rate(nginx_ingress_controller_requests{status="503"}[5m]))
            / sum(rate(nginx_ingress_controller_requests[5m])) > 0.05
        for: 5m
        annotations:
          summary: "入口 503 比例超过 5%"

      # 证书即将过期
      - alert: CertificateExpiringSoon
        expr: |
          (nginx_ingress_controller_ssl_expire_time_seconds - time()) / 86400 < 14
        annotations:
          summary: "证书 14 天内过期"

      # Controller 未就绪
      - alert: IngressControllerDown
        expr: up{job="ingress-nginx"} == 0
        for: 2m
        annotations:
          summary: "Ingress Controller 不可用"
```

证书过期告警必须配——Let's Encrypt 证书 90 天，续期链路一旦出问题（DNS 凭据失效、ACME 限流），不会有任何其他信号提醒你。

### 9.4.4 常见问题速查表

| 现象 | 最可能的原因 | 快速验证 |
|------|-------------|---------|
| 404 | `ingressClassName` 不匹配 / host 写错 | `kubectl describe ingress` 看 Class |
| 503 | Service Endpoints 为空 | `kubectl get endpoints <svc>` |
| 502 | 后端进程崩溃 / 协议不符（后端是 HTTPS 但配了 HTTP） | 直连 Pod IP 测试 |
| 504 | `proxy-read-timeout` 太小 / 后端真的慢 | 比对入口与后端日志时间 |
| 证书不受信任 | 证书链不完整 / Secret 不匹配 | `openssl s_client` 看证书链 |
| 配置不生效 | snippet 语法错误 / Controller 无权限 | `nginx -t` + Controller 日志 |
| 上传大文件 413 | `proxy-body-size` 默认 1m | 调大该注解 |
| 真实 IP 拿不到 | `externalTrafficPolicy` 不是 Local | 查 Service 与 `use-forwarded-headers` |
| SSE 不实时 | `proxy-buffering` 未关闭 | 看注解是否设置 |
| 间歇性 502 | 后端优雅退出没配（preStop/readinessProbe） | 看滚动更新期间的错误率 |

## 9.5 性能调优参考

### 9.5.1 Nginx Ingress Controller 调优

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # 每核的连接数上限
  max-worker-connections: "65536"
  # worker 进程数，auto 表示跟随 CPU 核数
  worker-processes: "auto"
  # 开启 keepalive 复用后端连接，显著降低长连接场景延迟
  upstream-keepalive-connections: "320"
  upstream-keepalive-requests: "10000"
  upstream-keepalive-timeout: "60"
  # 压缩
  enable-gzip: "true"
  enable-brotli: "true"                    # Brotli 压缩（比 gzip 更高效）
  # 日志：高流量下建议关掉 access log 或改为采样
  disable-access-log: "false"
```

关键的几项：

- **`upstream-keepalive-connections`**：不开启时每次转发都要新建到后端 Pod 的 TCP 连接，在高 QPS 下会大量消耗端口与握手开销。这是最值得调的一项。
- **`worker-processes: auto`**：默认可能是 1，导致多核机器上只用了一个核。
- **`max-worker-connections`**：默认为 512 或 1024，高并发下不够。

Controller 自身的资源也要放开：

```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 4
    memory: 2Gi
```

`limits` 设太低会导致 OOMKill；Controller 重启期间入口不可用，是影响面很大的故障。

### 9.5.2 内核参数调优

```bash
# 节点级别 sysctl 调优
sysctl -w net.core.somaxconn=65535                    # 监听队列大小
sysctl -w net.ipv4.tcp_max_syn_backlog=8192           # SYN 队列
sysctl -w net.core.netdev_max_backlog=5000            # 网卡积压
sysctl -w net.ipv4.tcp_tw_reuse=1                     # TIME_WAIT 重用
sysctl -w net.ipv4.ip_local_port_range="1024 65535"   # 本地端口范围
sysctl -w net.ipv4.tcp_fin_timeout=15                 # FIN 超时
sysctl -w fs.file-max=2097152                         # 文件描述符上限
```

`net.ipv4.ip_local_port_range` 与 `tcp_tw_reuse` 两项和 upstream keepalive 直接相关：不开启连接复用又碰上端口范围窄，会在高 QPS 下出现"无法建立连接"（`Cannot assign requested address`）。这是在压测环境里很典型的一类问题。

这些参数要么通过节点的 `sysctl` 配置持久化，要么用 `initContainer` + `securityContext.privileged` 在 Controller Pod 里设置（后者有安全代价，需评估）。
