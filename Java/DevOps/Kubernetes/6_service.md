# 1.Service概述

## 1.1 为什么需要Service

在 Kubernetes 集群中，Pod 是最小的调度单元，承载着实际运行的容器。然而，如果直接使用 Pod 的 IP 来访问应用，将会遇到一系列致命的问题。理解这些问题的本质，才能真正理解 Service 设计的必要性。

### Pod IP 的不稳定性

每个 Pod 在创建时会被分配一个集群内部的 IP 地址（来自 CNI 网络插件分配的 Pod CIDR）。但这个 IP 是**临时且不可靠**的，原因包括：

1. **Pod 重建导致 IP 变化**：当 Pod 因任何原因被删除并重建时（崩溃、节点故障、滚动更新、缩容再扩容），新 Pod 会获得一个全新的 IP 地址。Deployment 控制器的调谐循环会杀死旧的 Pod，创建新的 Pod——新 Pod 的 IP 与旧 Pod 完全不同。

2. **节点故障导致 Pod 漂移**：当某个 Node 宕机后，其上的 Pod 会被调度到其他健康 Node 上重新创建。这些漂移后的 Pod 不仅 IP 改变，连所在的物理节点也变了。

3. **滚动更新的 IP 变化**：Deployment 执行滚动更新（RollingUpdate）时，旧的 ReplicaSet 的 Pod 被逐步终止，新的 ReplicaSet 的 Pod 被逐步创建。整个过程客户端看到的 IP 地址一直在变化。

4. **调度绑定之前的 IP 不确定**：Pod 只有在被调度到具体 Node 上之后，才会由 CNI 插件分配 IP 地址。在 Pending 阶段 Pod 根本没有 IP。

举个具体例子来感受严重性：

```
初始状态:
  Pod A: IP=10.244.1.5
  Pod B: IP=10.244.2.8
  Pod C: IP=10.244.1.9

Pod A 崩溃，Deployment 重建:
  Pod A'(新): IP=10.244.2.15  ← IP 变了

Node 2 宕机，Pod B 漂移到 Node 3:
  Pod B'(新): IP=10.244.3.22  ← IP 又变了

滚动更新，全量替换:
  Pod A'': IP=10.244.1.28
  Pod B'': IP=10.244.3.31
  Pod C'': IP=10.244.2.35  ← 所有 IP 全变了
```

如果应用之间通过写死的 IP 地址互相通信，运维将变成一场灾难。每次 Pod 重启都需要重新配置连接信息，这在拥有数十个微服务、数百个 Pod 实例的生产环境中是完全不可接受的。

### 负载均衡的需求

即使 Pod IP 固定不变，仍然存在另一个核心问题：**多副本负载均衡**。

当一个应用需要横向扩展（Scale Out）时，通常会运行多个副本 Pod。客户端需要将请求均匀分发到这些副本上，以实现负载均衡。常见的问题包括：

1. **客户端负载均衡的复杂性**：如果让客户端自己维护后端 Pod IP 列表并实现负载均衡算法（轮询、最少连接、一致性哈希等），那么每个客户端都需要嵌入这部分逻辑。这不仅重复开发，而且难以维护——添加了新的后端 Pod 客户端如何感知？

2. **后端 Pod 集合的动态变化**：生产环境中，Pod 集合不是静态的。自动伸缩（HPA）会增加/减少 Pod 副本，滚动更新会替换 Pod，节点故障会导致 Pod 漂移。客户端如何实时跟踪这种变化？

3. **健康检查与故障隔离**：某个 Pod 虽然还在运行但已经无法正常响应（如进程卡死、OOM），客户端应该停止向它发送请求。如果由客户端负责健康检查，复杂度进一步上升。

4. **连接管理的一致性**：当后端 Pod 集发生变化时，已经建立的连接应该如何处理？新连接是否应该排除正在终止的 Pod？

### 服务发现的简化

在没有 Kubernetes 的经典架构中，通常会使用独立的外部服务发现系统（如 Consul、ZooKeeper、etcd）来管理服务注册与发现。工作流程是：

```
应用启动 → 注册到服务发现 → 定期心跳
客户端 → 查询服务发现 → 获取后端列表 → 客户端负载均衡 → 发起请求
```

这种方式存在几个问题：
- 应用代码需要嵌入服务发现 SDK（对应用有侵入性）
- 需要额外运维服务发现集群（增加系统复杂度）
- 需要处理服务发现集群本身的故障（分布式系统又引入新的分布式问题）

Kubernetes 的目标是将这些基础设施能力**下沉到平台层**，让应用代码无需感知服务发现的细节。

### 抽象统一访问入口的需求

假设一个电商系统的订单服务部署了 5 个 Pod 副本：

```
订单服务Pod: 10.244.1.10, 10.244.2.11, 10.244.3.12, 10.244.1.13, 10.244.2.14
```

支付服务需要调用订单服务，它应该连接哪个 IP？如果用 `10.244.1.10`，这个 Pod 的负载会特别高（所有请求都打到这一个），而其他 4 个 Pod 处于闲置状态。而且一旦 `10.244.1.10` 对应的 Pod 挂了，支付服务就彻底不可用了。

**Service 正是为了解决以上所有问题而设计的**——它提供一个**稳定的虚拟 IP 和 DNS 名称**，自动将请求负载均衡到一组健康的 Pod 上。

---

## 1.2 Service是什么

Service 是 Kubernetes 中定义的一种资源对象，它为后端的一组 Pod 提供**稳定的、统一的网络访问端点**。可以将 Service 理解为一个"四层代理"（L4 Load Balancer），它工作在 OSI 模型的传输层（TCP/UDP/SCTP）。

### Service 的核心能力

Service 提供了以下核心能力：

1. **稳定 IP（ClusterIP）**：Service 被分配一个虚拟 IP（Virtual IP），这个 IP 在 Service 的整个生命周期中保持不变。无论后端的 Pod 如何创建、销毁、漂移，这个 VIP 始终不变。客户端只需连接这个 VIP 即可。

2. **负载均衡**：发往 Service VIP 的流量被自动分发到后端健康的 Pod 上。默认使用轮询（Round Robin）或随机算法（取决于 kube-proxy 模式）。

3. **服务发现**：Service 创建后会自动注册到集群 DNS（CoreDNS）中，其他 Pod 可以通过 `<service-name>.<namespace>.svc.cluster.local` 这个 FQDN 来访问 Service，无需知道 IP 地址。

4. **端口映射**：Service 可以将外部可访问的端口映射到 Pod 内部的端口，提供灵活的端口管理。

5. **Pod 选择器**：通过 Label Selector，Service 动态地识别哪些 Pod 属于其后端，无需硬编码 IP 列表。

### Service 的抽象模型

从概念上理解，Service 扮演了以下角色：

```
                     ┌──────────────┐
                     │   Client     │
                     │  (任意Pod)   │
                     └──────┬───────┘
                            │ 连接到 Service IP:Port
                            │ 例如 10.96.100.1:8080
                            ▼
                   ┌─────────────────┐
                   │    Service      │
                   │  ClusterIP:     │
                   │  10.96.100.1    │
                   │  Port: 8080     │
                   │  Selector:      │
                   │    app=order    │
                   └────────┬────────┘
                            │ 负载均衡到后端Pod
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Pod A   │ │  Pod B   │ │  Pod C   │
        │ IP:      │ │ IP:      │ │ IP:      │
        │10.244.1.5│ │10.244.2.8│ │10.244.1.9│
        │ :8080    │ │ :8080    │ │ :8080    │
        │          │ │          │ │          │
        │ labels:  │ │ labels:  │ │ labels:  │
        │ app=order│ │ app=order│ │ app=order│
        └──────────┘ └──────────┘ └──────────┘
```

客户端永远连接 `10.96.100.1:8080`，具体将请求转发到哪个 Pod 完全由 Service 决定。Pod A 被销毁重建后的新 IP 变成 `10.244.3.20`——只要它的 Label 仍然是 `app=order`，它就会被 Service 自动纳入后端列表。

### Service 并非真正存在的网络实体

需要特别强调的是：**Service 的 ClusterIP 只是一个虚拟 IP，它不对应任何具体的网络接口**。你在任何 Node 上执行 `ip addr` 都看不到这个 IP。它只存在于：

- kube-proxy 写入的 **iptables 规则**中（iptables 模式）
- kube-proxy 创建的 **IPVS 虚拟服务器**中（IPVS 模式）
- CoreDNS 的 **DNS 记录**中

这意味着：你不能 `ping` 一个 ClusterIP（它不是一个真实的存在），只能通过 TCP/UDP 连接到它。这是初学者最常见的困惑之一。

### Service 的工作范围

Service 工作在 OSI 模型的第 4 层（传输层），这意味着：
- **能做**：基于 TCP/UDP/SCTP 的端口转发和负载均衡
- **不能做**：基于 HTTP Host 头的七层路由（需要 Ingress Controller）
- **不能做**：基于 URL 路径的流量分发（需要 Ingress Controller）
- **不能做**：TLS 终止（需要 Ingress Controller 或 Service Mesh）

当需要在七层进行流量管理时，需要配合 Ingress 或 Gateway API 来实现。

---

## 1.3 Service与Pod的解耦

Service 与 Pod 之间的关键设计原则是**松耦合**（Loose Coupling）。Service 不通过硬编码的 IP 列表来指定后端 Pod，而是通过 **Label Selector**（标签选择器）动态发现后端。

### Label 与 Selector 的匹配机制

Service 使用 `spec.selector` 字段来定义后端 Pod 的选择条件：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order
    version: v1
  ports:
  - port: 8080
    targetPort: 8080
```

这个 Service 会选择所有同时满足以下条件的 Pod：
- `app=order`
- `version=v1`

注意：Selector 的多个条件是**AND（与）**关系，Pod 必须满足所有条件才会被选中。

### 解耦带来的工程优势

这种基于 Label 的解耦带来了巨大的工程灵活性：

1. **动态成员管理**：Pod 创建时如果带有匹配的 Label，自动成为 Service 的后端；Pod 被删除时自动从后端移除。这个过程对 Service 的使用者完全透明。

2. **金丝雀发布（Canary Deployment）**：可以创建同一个应用的不同版本，通过不同的 Label 区分：

```yaml
# 稳定版本 Pod
apiVersion: v1
kind: Pod
metadata:
  name: order-v1-xxx
  labels:
    app: order
    version: v1          # 稳定版本

---
# 金丝雀版本 Pod
apiVersion: v1
kind: Pod
metadata:
  name: order-v2-xxx
  labels:
    app: order
    version: v2          # 金丝雀版本

---
# 稳定 Service 选择 v1
apiVersion: v1
kind: Service
metadata:
  name: order-stable
spec:
  selector:
    app: order
    version: v1

---
# 金丝雀 Service 选择 v2
apiVersion: v1
kind: Service
metadata:
  name: order-canary
spec:
  selector:
    app: order
    version: v2
```

这样可以在同一套基础设施上运行两个版本的应用，用不同的 Service 分别暴露，进行流量对比和验证。

3. **蓝绿部署（Blue-Green Deployment）**：维护两套完整的环境，通过切换 Service 的 Selector 来实现流量切换。这种方式的切换速度极快（只需修改 Service 对象），回滚也同样迅速。

```yaml
# 蓝环境 Pod Label: app=order, color=blue
# 绿环境 Pod Label: app=order, color=green

# Service 初始选择蓝环境
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order
    color: blue
  ports:
  - port: 8080

# 切换时修改 Selector
# selector:
#   app: order
#   color: green
```

4. **跨命名空间引用**：Service 只能选择同一 Namespace 内的 Pod。这是 Kubernetes 的隔离边界设计。如果需要跨命名空间通信，可以使用 ExternalName Service 或显式指定完整的跨命名空间 DNS 名称。

5. **无 Selector 的 Service**：在某些场景下，Service 不需要 Selector。此时 Service 不会自动管理 Endpoints，需要用户手动创建和维护 Endpoints 对象（或 EndpointSlice），手动指定后端的 IP 和端口列表。这种模式用于：
   - 将集群外部服务（如外部数据库）引入集群内部 DNS 发现
   - 将其他 Kubernetes 集群中的服务引用到本地集群
   - 自定义服务后端的特殊场景

```yaml
# 无 Selector 的 Service
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  ports:
  - port: 3306
    targetPort: 3306
---
# 手动管理 Endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: external-database       # 名字必须与 Service 相同
subsets:
- addresses:
  - ip: 192.168.1.100           # 外部数据库的实际 IP
  - ip: 192.168.1.101
  ports:
  - port: 3306
```

### 健康检查与就绪探测

Service 不会简单地将所有匹配 Label 的 Pod 加入后端列表。它还会检查 Pod 的 **Ready 状态**：

1. **Readiness Probe（就绪探测）**：Pod 必须通过 `readinessProbe` 检查，才会被 Service 标记为 Ready。这确保只有真正能处理请求的 Pod 才会接收流量。

2. **Readiness Gate**：Kubernetes 还支持 Readiness Gate，允许外部控制器（如 AWS Load Balancer Controller）向 Pod 注入额外的就绪条件。

3. **Deletion 与流量摘除**：当 Pod 被标记为 `Terminating` 时，Service 会立即将其从后端列表摘除。这个过程与 Pod 的 `terminationGracePeriodSeconds` 配合，确保 Pod 在终止期间不再接收新请求，但已有请求可以正常完成。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-pod
  labels:
    app: order
spec:
  containers:
  - name: app
    image: order:latest
    ports:
    - containerPort: 8080
    readinessProbe:              # 就绪探测
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5     # 容器启动后等待5秒才开始探测
      periodSeconds: 3           # 每3秒探测一次
      failureThreshold: 3        # 连续失败3次才标记为 Not Ready
      successThreshold: 1        # 成功1次即标记为 Ready
```

### Endpoints Controller 的工作流

Service 与 Pod 解耦的背后，是 Endpoints Controller 的持续工作。完整流程如下：

```
1. 用户创建 Service（指定 selector: app=order）
       │
       ▼
2. API Server 持久化 Service 对象到 etcd
       │
       ▼
3. Endpoints Controller 监听到新 Service
       │  根据 selector 查找匹配的 Pod
       │  过滤条件：Pod 的 Phase 为 Running + Readiness Probe 通过
       ▼
4. Endpoints Controller 创建/更新 Endpoints 对象
       │  Endpoints.subsets[].addresses[] 记录所有 Ready 的 Pod IP:Port
       │  Endpoints.subsets[].notReadyAddresses[] 记录未就绪的 Pod IP:Port
       ▼
5. kube-proxy 监听 Endpoints 对象的变更
       │  将后端 Pod IP:Port 写入 iptables/IPVS 规则
       ▼
6. 客户端流量经过 Service ClusterIP → DNAT → 后端 Pod IP:Port
```

当 Pod 状态变化时（新建、删除、Ready 状态变化），Endpoints Controller 会实时更新 Endpoints 对象，kube-proxy 随之更新转发规则。

---

## 1.4 Service的虚拟IP (ClusterIP) 实现原理

ClusterIP 是 Service 最核心的设计概念之一。它本质上是一个**虚拟 IP**（Virtual IP），不对应任何真实的网络接口，仅存在于内核的 netfilter（iptables）或 IPVS 规则中。

### 虚拟IP的本质：不存在的IP如何工作

理解 ClusterIP 的关键在于：**数据包在网络层的处理是由内核完成的**。当一个 Pod 想要连接 ClusterIP `10.96.100.1:8080` 时，发生了什么？

1. 应用程序调用 `connect("10.96.100.1", 8080)`，发起 TCP 连接。
2. 操作系统构造一个 SYN 数据包：源 IP=Pod IP，目标 IP=10.96.100.1，目标端口=8080。
3. 数据包经过 Pod 所在 Node 的网络栈。
4. 在 **OUTPUT 链** 或 **PREROUTING 链**，iptables 规则链拦截这个数据包。
5. iptables 规则检测到目标 IP 是某个 Service 的 ClusterIP，目标端口匹配 Service Port。
6. iptables 执行 **DNAT**（Destination Network Address Translation），将目标 IP 改为一个后端 Pod 的 IP，目标端口改为 Pod 的 targetPort。
7. 数据包继续正常路由，最终到达后端 Pod。

关键洞察：**ClusterIP 从来没有被任何网络接口应答。没有任何 ARP 回应，没有网卡声称拥有这个 IP。数据包永远不会被"路由到"这个 IP，而是在到达路由决策之前就被 iptables 规则拦截并重写了目标地址。**

### iptables 规则链的完整路径

以 iptables 模式为例，发往 ClusterIP 的数据包经过的规则链如下（Node 上的规则）：

```
PREROUTING → KUBE-SERVICES → KUBE-SVC-XXXX → KUBE-SEP-XXXX → DNAT to Pod IP
```

详细展开：

1. **PREROUTING**：所有进入本机的数据包首先经过 PREROUTING 链。
2. **KUBE-SERVICES**：kube-proxy 创建的自定义链，包含所有 Service 的匹配规则。
3. **KUBE-SVC-XXXX**：每个 Service 对应一条自定义链（后缀是 Service 的哈希值）。
4. **KUBE-SEP-XXXX**：**S**ervice **E**nd**P**oint，每个后端 Pod 对应一条自定义链。
5. **DNAT**：最终执行 DNAT，将目标地址改写为后端 Pod IP。

### DNAT 后的数据包走向

DNAT 之后，数据包的新目标 IP 是后端 Pod 的 IP。此时需要区分两种情况：

**情况1：后端 Pod 与源 Pod 在同一 Node 上**

数据包在 OUTPUT 链被 DNAT 后，源 IP 仍是源 Pod IP，目标 IP 变成了同 Node 上另一个 Pod 的 IP。数据包通过 CNI 网络（通常是 veth pair + bridge）直接到达目标 Pod，不会离开 Node。

**情况2：后端 Pod 在不同 Node 上**

数据包在 OUTPUT 链被 DNAT 后，目标 IP 变成了另一个 Node 上的 Pod IP。数据包经过正常的跨节点路由：CNI 网络通过 VXLAN/BGP/IP-in-IP 等 Overlay 或 Underlay 方式将数据包发送到目标 Node，然后到达目标 Pod。

### ClusterIP 的分配机制

Service 的 ClusterIP 由 `kube-apiserver` 中的 `ServiceClusterIPAllocator` 负责分配：

1. **地址范围配置**：`kube-apiserver` 的启动参数 `--service-cluster-ip-range` 定义了所有 Service ClusterIP 的 CIDR 范围，默认是 `10.0.0.0/24`（通常配置为 `10.96.0.0/12` 或 `10.0.0.0/16` 等）。

2. **位图分配算法**：API Server 使用位图（bitmap）来管理 ClusterIP 的分配。每个 Cluster IP 对应位图中的一个位。分配时遍历位图寻找空闲位，释放时清除对应位。

3. **IP 保留**：
   - CIDR 的第一个 IP（网络地址）和最后一个 IP（广播地址）不可用
   - 用户可以通过 `spec.clusterIP` 显式指定一个 IP（必须在 CIDR 范围内且未分配）
   - 如果不指定，系统自动分配
   - Headless Service 的 `clusterIP: None` 不占用该地址池

4. **分配流程**：

```
创建 Service 请求到达 API Server
  │
  ▼
校验：Service 类型、端口冲突、Selector 有效性
  │
  ▼
如果 spec.clusterIP 未指定:
  │  Allocator 扫描位图，寻找第一个未使用的 IP
  │  将 IP 分配给 Service
  │  更新位图
  │
如果 spec.clusterIP 已指定:
  │  检查 IP 是否在 service-cluster-ip-range 内
  │  检查 IP 是否已被其他 Service 使用
  │  通过则直接使用
  │
  ▼
将 ClusterIP 写入 Service 对象
持久化到 etcd
通知所有 watcher（包括 kube-proxy）
```

### 为什么不能 Ping ClusterIP

这是 Kubernetes 新手最常见的困惑。`ping` 使用 ICMP 协议，它的工作流程是：

1. 源主机发出 ICMP Echo Request，目标 IP = ClusterIP
2. 操作系统需要**先通过 ARP 解析目标的 MAC 地址**
3. 由于 ClusterIP 是虚拟 IP，没有真实的网络接口，**ARP 请求得不到回应**
4. ARP 解析失败，数据链路层无法封装帧，ICMP 数据包无法发出

而 TCP/UDP 连接之所以能成功，是因为 iptables 规则在**网络层**拦截了数据包——数据包根本不需要经过 ARP 解析。iptables 的 DNAT 在 netfilter 框架的 NF_IP_PRE_ROUTING 或 NF_IP_LOCAL_OUT 钩子点执行，发生在路由决策之前。

验证方式：

```bash
# 无法 ping ClusterIP
$ ping 10.96.100.1
PING 10.96.100.1 (10.96.100.1) 56(84) bytes of data.
(无回应，超时)

# 但可以用 curl 建立 TCP 连接
$ curl http://10.96.100.1:8080
200 OK
```

### Service 的并发访问安全

多个客户端同时访问同一个 Service 的 ClusterIP 时，iptables/IPVS 的负载均衡行为如何？

**iptables 模式**：
- iptables 使用 `statistic` 模块实现概率性负载均衡
- 每个数据包独立决策，不维护连接状态（conntrack 负责连接追踪）
- 长连接的多个数据包由 conntrack 确保发送到同一个后端（基于连接追踪表）
- 规则更新期间可能出现短暂的不一致

**IPVS 模式**：
- IPVS 维护真正的连接表，支持多种调度算法
- 调度器为每个新连接选择一个后端，之后的同一个连接的所有数据包都发往该后端
- 连接断开/超时后，下次新连接可能被调度到不同的后端

### ClusterIP 的不可路由性

ClusterIP 只在集群内部有效。这有两个层面的含义：

1. **物理层面**：ClusterIP 地址段（如 `10.96.0.0/12`）不属于任何物理网络的 IP 段。集群外部的路由器不知道如何路由这些 IP 地址。

2. **配置层面**：iptables/IPVS 规则只存在于集群内的每个 Node 上。集群外部的机器上没有这些规则，即使数据包到达也不会被 DNAT。

因此，要从集群外部访问 Service，必须使用 NodePort 或 LoadBalancer 类型的 Service，它们提供了外部访问的入口点。ClusterIP 类型是"仅限集群内部"的。
# 2.Service类型详解

## 2.1 ClusterIP (集群内部访问，默认类型)

ClusterIP 是 Service 的**默认类型**。如果不显式指定 `spec.type`，则默认创建 ClusterIP 类型的 Service。它也是最基础的类型，其他类型（NodePort、LoadBalancer）都是在 ClusterIP 基础上扩展的。

### 典型 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-clusterip-service
  namespace: default
spec:
  type: ClusterIP                    # 默认值，可以省略
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80                        # Service 对外暴露的端口
    targetPort: 8080                # 后端 Pod 的容器端口
  sessionAffinity: None             # 默认值，不使用会话亲和性
```

### 创建后的资源状态

```shell
$ kubectl get svc my-clusterip-service
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
my-clusterip-service    ClusterIP   10.96.100.42    <none>        80/TCP    10s
```

注意 EXTERNAL-IP 列显示 `<none>`，说明该 Service 没有外部可访问的 IP。

### 访问方式

ClusterIP Service 可以从以下位置访问：

1. **集群内的任意 Pod**：通过 ClusterIP 或 Service 的 DNS 名称访问
2. **集群内的任意 Node**：通过 ClusterIP（因为 Node 上也运行着 kube-proxy 和相应的 iptables/IPVS 规则）
3. **在 Node 上直接执行的进程**：同样通过 ClusterIP（Node 上的 iptables 规则在 OUTPUT 链也有配置）

**不能**从以下位置访问：
- 集群外部的客户端（无 NodePort，无外部 LB）
- 其他 Kubernetes 集群中的 Pod（除非有 VPN 或 VPC Peering 打通网络）

### 内部访问模式的流量路径

```
客户端 Pod (10.244.1.5)
    │
    │ 目标: 10.96.100.42:80
    ▼
Pod 所在 Node 的网络栈
    │ iptables PREROUTING/OUTPUT 链
    │ 匹配 ClusterIP:Port
    │ 执行 DNAT
    ▼
目标改为后端 Pod IP:8080 (如 10.244.2.8)
    │
    │ 通过 CNI 网络路由
    ▼
后端 Pod (10.244.2.8:8080)
```

### 使用场景

ClusterIP 是最常用的 Service 类型，适用于绝大部分集群内部服务间通信场景：

- 微服务架构中服务之间的调用：订单服务调用库存服务
- 前端应用调用后端 API
- 数据库连接：应用 Pod 通过 Service 连接数据库 Pod
- 缓存层访问：应用通过 Service 访问 Redis 集群

一般生产环境中，80% 以上的 Service 都是 ClusterIP 类型。只有少数需要从外部访问的服务才使用 NodePort 或 LoadBalancer。

### ClusterIP 的不变性

一旦 Service 被创建并分配了 ClusterIP，这个 IP 在 Service 的整个生命周期中都不会改变（除非删除并重新创建 Service）。即使你修改 Service 的 Selector、Port 等配置，ClusterIP 保持不变。这个特性保证了：

- DNS 记录稳定（DNS 名称始终指向同一个 ClusterIP）
- 客户端缓存的连接信息不会失效
- 不会因为配置变更导致服务发现失效

你可以在创建时通过 `spec.clusterIP` 字段显式指定一个 ClusterIP（必须在 `service-cluster-ip-range` 内且未被使用）：

```yaml
spec:
  type: ClusterIP
  clusterIP: 10.96.100.200         # 显式指定 ClusterIP
```

推荐让 Kubernetes 自动分配 IP，避免手动管理 IP 池带来的冲突风险。

---

## 2.2 NodePort (每个节点开端口，外部可访问)

NodePort 类型的 Service 是在 ClusterIP 基础上，额外暴露一个 Node 上的端口（称为 NodePort），使得集群外部的客户端可以通过 `<任意Node的IP>:<NodePort>` 访问到 Service。

### 典型 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort                     # 类型设置为 NodePort
  selector:
    app: web-app
  ports:
  - name: http
    protocol: TCP
    port: 80                        # Service ClusterIP 的端口
    targetPort: 8080                # 后端 Pod 的容器端口
    nodePort: 30080                 # Node 上暴露的端口 (可选，30000-32767)
```

### NodePort 端口范围

NodePort 的端口范围默认为 **30000-32767**，可以通过 `kube-apiserver` 的 `--service-node-port-range` 参数配置：

```
--service-node-port-range=30000-32767
```

如果指定的 `nodePort` 在这个范围内且未被其他 NodePort Service 占用，系统会接受；如果不指定，系统自动分配一个未被占用的端口。

### 创建后的资源状态

```shell
$ kubectl get svc my-nodeport-service
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
my-nodeport-service    NodePort   10.96.100.43    <none>        80:30080/TCP   5s
```

PORT(S) 列显示为 `80:30080/TCP`，含义是：
- ClusterIP 端口 80 映射到 Node 端口 30080
- 访问 ClusterIP:80 → 后端 Pod:8080（集群内部路径）
- 访问 NodeIP:30080 → ClusterIP:80 → 后端 Pod:8080（外部路径）

### 外部访问的流量路径

```
外部客户端 (192.168.1.100)
    │
    │ 目标: Node1的IP:30080 (如 10.0.0.1:30080)
    ▼
Node1 (10.0.0.1)
    │ iptables PREROUTING 链
    │ 匹配目标端口 30080 (NodePort)
    │ 执行 DNAT: 目标改为 ClusterIP:80
    ▼
Node1 网络栈
    │ iptables 继续匹配 ClusterIP:80
    │ 执行 DNAT: 目标改为后端 Pod IP:8080
    │ 如果后端 Pod 不在 Node1 上，通过 CNI 跨节点路由
    ▼
后端 Pod (可能在 Node1 或 Node2 上)
```

需要特别注意的是：即使客户端连接的 Node 上没有运行该 Service 的后端 Pod，请求也能正常到达。iptables 规则会将流量从 Node1 DNAT 到 ClusterIP，再从 ClusterIP DNAT 到实际后端的 Pod IP——如果 Pod 在 Node2 上，数据包会经过跨节点路由到达 Node2。

### 双重 DNAT 过程

NodePort 请求经历了**两次 DNAT**：

```
第一次 DNAT: 目标 IP:NodeIP, Port:30080 → 目标 IP:ClusterIP, Port:80
第二次 DNAT: 目标 IP:ClusterIP, Port:80 → 目标 IP:PodIP, Port:8080
```

这就是为什么即使在集群内部，你也可以通过 NodePort 访问 Service——数据经过两次 DNAT 最终到达后端 Pod。

### NodePort 的优缺点

**优点**：
- 配置简单，不需要额外的负载均衡器
- 每个 Node 上相同的端口都能访问，提供了一定的高可用性
- 适合开发和测试环境

**缺点**：
- **端口管理困难**：NodePort 端口范围有限（30000-32767），在大型集群中可能不够用
- **端口冲突风险**：需要在集群中全局协调端口分配
- **安全性隐患**：直接暴露 Node 端口给外部，依赖防火墙规则保护
- **客户端耦合**：客户端需要知道 Node IP 和端口，不够灵活
- **没有外部负载均衡**：需要在客户端侧或通过外部负载均衡器分发到多个 Node

### 生产环境中的使用

NodePort 在生产环境中**不建议直接暴露给外部用户**。更常见的做法是：

1. 使用 LoadBalancer 类型 Service（云环境），它自动创建外部 LB 并配置后端为 NodePort
2. 使用 Ingress Controller，它是集群内部的反向代理，通过 NodePort 或 LoadBalancer 暴露
3. 使用手动配置的外部负载均衡器（如 HAProxy/Nginx/F5），它的后端指向各 Node 的 NodePort

---

## 2.3 LoadBalancer (云负载均衡器，自动创建外部LB)

LoadBalancer 是云环境中最常用的 Service 类型。它在 NodePort 的基础上，通过云服务商的 API 自动创建一个外部的云负载均衡器（如 AWS ELB/NLB、GCP Cloud Load Balancer、Azure Load Balancer），并将其配置为将流量转发到各 Node 的 NodePort。

### 典型 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-lb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"    # 云厂商特定注解
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  loadBalancerSourceRanges:           # 可选：限制访问来源 IP CIDR（白名单）
  - 203.0.113.0/24
  - 198.51.100.0/24
```

### 云控制器管理器 (Cloud Controller Manager)

LoadBalancer Service 的自动创建依赖 **Cloud Controller Manager (CCM)**，它是 Kubernetes 控制平面的一个组件，负责与底层云平台交互：

1. **Node Controller**：监控节点状态，在云平台上添加/删除节点的路由
2. **Route Controller**：在云平台 VPC 中配置网络路由
3. **Service Controller**：监听 LoadBalancer 类型 Service 的变更，调用云平台 API 创建/更新/删除负载均衡器

Service Controller 的工作流程：

```
1. 用户创建 LoadBalancer 类型的 Service
       │
       ▼
2. API Server 持久化 Service 对象到 etcd
       │
       ▼
3. kube-proxy 自动为该 Service 分配 NodePort
       │
       ▼
4. Cloud Controller Manager - Service Controller 检测到 LoadBalancer Service
       │  调用云平台 API 创建外部 LB
       │  LB 监听器：外部端口 80 → 后端端口 (NodePort)
       │  LB 目标组：所有 Node 的 IP:NodePort
       │  LB 健康检查：定期检查 NodePort 是否可达
       │
       ▼
5. 云平台返回 LB 的外部 IP（或 DNS 名称）
       │
       ▼
6. Service Controller 将外部 IP 写入 Service.status.loadBalancer
       │
       ▼
7. 用户通过 EXTERNAL-IP 访问服务
```

### 创建后的资源状态

```shell
$ kubectl get svc my-lb-service
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
my-lb-service     LoadBalancer   10.96.100.44    203.0.113.50     80:30081/TCP   30s
```

EXTERNAL-IP 列显示了一个公网可访问的 IP 地址。这个 IP 是云负载均衡器的 VIP。

### 流量路径

```
外部客户端 (Internet)
    │
    │ 目标: LB EXTERNAL-IP:80 (203.0.113.50:80)
    ▼
云负载均衡器 (AWS NLB / GCP LB / Azure LB)
    │
    │ 负载均衡: 分发到各 Node IP:NodePort
    │ 健康检查: 确认 NodePort 可达
    ▼
┌───────────────┬───────────────┬───────────────┐
│    Node 1     │    Node 2     │    Node 3     │
│ :30081        │ :30081        │ :30081        │
│    ↓          │    ↓          │    ↓          │
│  iptables     │  iptables     │  iptables     │
│    ↓          │    ↓          │    ↓          │
│ KUBE-SVC 链   │ KUBE-SVC 链   │ KUBE-SVC 链   │
│    ↓          │    ↓          │    ↓          │
│ DNAT to Pod   │ DNAT to Pod   │ DNAT to Pod   │
└───────────────┴───────────────┴───────────────┘
```

### 云厂商特定的实现差异

不同云厂商的 LoadBalancer Service 实现有显著差异：

**AWS**：
- `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"` 创建 Network Load Balancer (L4)
- `service.beta.kubernetes.io/aws-load-balancer-type: "clb"` 创建 Classic Load Balancer（旧版）
- 支持通过 annotation 配置 SSL 证书、安全组、子网等
- AWS NLB 支持通过 `externalTrafficPolicy: Local` 保留源 IP

**GCP**：
- 自动创建外部 TCP/UDP Network Load Balancer
- 通过 BackendConfig CRD 可以配置 Cloud CDN、IAP、安全策略等
- 支持 Internal Load Balancer（内部 LB，`networking.gke.io/load-balancer-type: Internal`）

**Azure**：
- 创建 Azure Standard Load Balancer
- 支持 Internal Load Balancer 通过 annotation 配置
- 通过 `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` 创建内部 LB

**裸金属 (Bare Metal)**：
- 裸金属环境没有云 LoadBalancer 可用
- 可以使用 **MetalLB**（裸金属 LoadBalancer 实现），通过 ARP/NDP 或 BGP 协议实现外部 LB
- MetalLB 为 LoadBalancer Service 提供可路由的 EXTERNAL-IP

### loadBalancerSourceRanges

通过 `spec.loadBalancerSourceRanges` 可以限制哪些来源 IP 可以访问 LoadBalancer：

```yaml
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
  - 10.0.0.0/8          # 仅允许内网
  - 203.0.113.0/24      # 特定的公网 IP 段
```

这个配置会被翻译为云平台 LB 对应的安全组/防火墙规则。这是在平台层面实现的安全隔离，比在应用层面配置更高效、更安全。

### 内部负载均衡器 (Internal LoadBalancer)

LoadBalancer 不一定是面向公网的。通过云厂商特定的注解，可以创建仅在 VPC 内部可访问的内部负载均衡器：

```yaml
# AWS 内部 NLB
apiVersion: v1
kind: Service
metadata:
  name: internal-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

内部 LB 的 EXTERNAL-IP 是一个 VPC 内部 IP，只能在 VPC 内（包括 VPN 隧道和 VPC Peering 连接的网络）访问。

---

## 2.4 ExternalName (DNS CNAME别名)

ExternalName 是一种非常特殊的 Service 类型。它**不创建 ClusterIP，不创建 Endpoints，不进行任何代理或负载均衡**。它只是在集群 DNS 中创建一个 **CNAME 记录**，将一个 Kubernetes Service DNS 名称映射到一个外部的 DNS 名称。

### 典型 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
  namespace: production
spec:
  type: ExternalName
  externalName: db.example.com       # 外部 DNS 名称
```

### DNS 解析行为

创建上述 Service 后，集群内的 Pod 访问 `external-database.production.svc.cluster.local` 时：

```
Pod 内应用解析 external-database.production.svc.cluster.local
    │
    ▼
CoreDNS 查找 external-database Service
    │  Service 类型为 ExternalName
    │  externalName = db.example.com
    ▼
CoreDNS 返回 CNAME 记录: db.example.com
    │
    ▼
Pod 内的 DNS 解析器继续解析 db.example.com
    │  如果 db.example.com 是集群外的域名，使用上游 DNS
    ▼
最终获得 db.example.com 的实际 IP 地址
```

### 使用场景

1. **访问外部服务**：将数据库、消息队列等运行在集群外的服务通过内部 DNS 名称暴露给 Pod。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-redis
spec:
  type: ExternalName
  externalName: redis-cluster.abcdef.0001.use1.cache.amazonaws.com
```

Pod 内应用配置 `external-redis:6379` 即可连接，无需硬编码 AWS ElastiCache 的端点域名。如果将来迁移到 GCP Memorystore，只需修改 Service 的 `externalName` 字段，应用代码无需改动。

2. **跨命名空间访问的灵活性**：

常规跨命名空间访问：
```
Service名称.命名空间.svc.cluster.local
例如: database.production.svc.cluster.local
```

使用 ExternalName，可以在 Namespace A 中创建一个指向 Namespace B 中 Service 的别名，简化配置。

3. **环境适配**：开发环境中的 Service 映射到本地开发服务，生产环境映射到云厂商托管服务。

### 限制与注意事项

- **没有负载均衡**：ExternalName 只是 DNS CNAME，不做任何流量代理
- **没有端口映射**：ExternalName 仅仅做 DNS 解析，调用方必须知道目标服务的端口
- **没有健康检查**：Kubernetes 不会对 ExternalName 指向的服务做健康检查
- **TLS 问题**：如果使用 HTTPS，TLS 证书中的主机名可能与 Service 名称不匹配，需要使用正确的 SNI
- **DNS 缓存**：CNAME 记录的 TTL 可能导致地址变更后仍有短暂的缓存延迟

---

## 2.5 Headless Service (ClusterIP=None，直接返回Pod IP列表)

Headless Service 是一种特殊的 Service 配置，通过设置 `clusterIP: None` 来创建。它不分配 ClusterIP，也不进行负载均衡。DNS 查询会直接返回后端所有 Ready Pod 的 IP 地址列表（A 记录或 AAAA 记录）。

### 典型 YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-svc
spec:
  clusterIP: None                    # 关键：设置为 None
  selector:
    app: cassandra
  ports:
  - port: 9042
    targetPort: 9042
```

### DNS 解析行为

当客户端 Pod 查询 Headless Service 的 DNS 时：

```shell
# 正常 Service (ClusterIP)
$ nslookup normal-service.default.svc.cluster.local
Name:    normal-service.default.svc.cluster.local
Address: 10.96.100.42                    # 返回一个 ClusterIP

# Headless Service (clusterIP: None)
$ nslookup headless-svc.default.svc.cluster.local
Name:    headless-svc.default.svc.cluster.local
Address: 10.244.1.5                      # 直接返回 Pod IP
Address: 10.244.2.8
Address: 10.244.3.12
```

DNS 返回的是 A 记录列表，而不是一个单独的 ClusterIP。这意味着：
- 客户端知道所有后端 Pod 的真实 IP
- 客户端需要自己实现负载均衡逻辑
- 客户端可以直接与特定 Pod 建立连接（如需要粘性会话的场景）

### 与 StatefulSet 的配合

Headless Service 最常见的用法是与 StatefulSet 配合使用。StatefulSet 会给每个 Pod 分配一个稳定的**主机名**和**网络标识**（如 `cassandra-0.cassandra.default.svc.cluster.local`）。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
  labels:
    app: cassandra
spec:
  clusterIP: None
  selector:
    app: cassandra
  ports:
  - port: 9042
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
spec:
  serviceName: cassandra              # 必须引用 Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: cassandra
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      containers:
      - name: cassandra
        image: cassandra:4.0
        ports:
        - containerPort: 9042
```

在此配置下，三个 Pod 具有稳定的 DNS 名称：
- `cassandra-0.cassandra.default.svc.cluster.local` → IP 固定（StatefulSet 特性）
- `cassandra-1.cassandra.default.svc.cluster.local` → IP 固定
- `cassandra-2.cassandra.default.svc.cluster.local` → IP 固定

对于 Cassandra/Elasticsearch/etcd/ZooKeeper 这类需要节点间互相发现和通信的分布式系统，这种稳定的网络标识至关重要。

### 适用场景

1. **分布式数据库**：Cassandra、MongoDB、Elasticsearch 等需要节点间直接通信
2. **集群中的集群**：etcd、ZooKeeper、Consul 等需要自定义选举协议
3. **自定义负载均衡**：使用客户端库（如 gRPC 的客户端负载均衡）替代 kube-proxy
4. **需要知道所有成员**：Storm、Apache Spark 等大数据框架需要知道集群完整拓扑

---

## 2.6 各类型对比与选择指南

### 对比表格

| 特性 | ClusterIP | NodePort | LoadBalancer | ExternalName | Headless |
|------|-----------|----------|--------------|--------------|----------|
| **ClusterIP** | 有 | 有 | 有 | 无 | 无 |
| **外部访问** | 不可 | 可（通过 NodeIP:Port） | 可（通过 LB IP） | 可（DNS 解析） | 不可直接 |
| **负载均衡** | 四层 LB | 四层 LB | 四层 LB（LB）+ 四层 LB（kube-proxy） | 无 | 无(客户端自实现) |
| **DNS 解析** | A 记录 (ClusterIP) | A 记录 (ClusterIP) | A 记录 (ClusterIP) | CNAME 记录 | A 记录列表 (Pod IP) |
| **云平台依赖** | 无 | 无 | 需要云 LB 或 MetalLB | 无 | 无 |
| **性能** | 高（直接 DNAT） | 中（双重 DNAT） | 中（LB 层 + DNAT） | 无代理 | 高（直连 Pod） |
| **客户端源IP保留** | SNAT 后丢失 | 取决于 externalTrafficPolicy | 取决于配置 | 无影响 | 无影响 |
| **端口管理** | 自动 | 需配置 NodePort范围 | 自动分配 | 无需 | 自动 |
| **使用复杂度** | 低 | 中 | 中 | 低 | 中 |

### 选择决策树

```
需要访问集群外部服务？
  └── 是 → 使用 ExternalName 或 Headless Service (手动管理 Endpoints)

需要从集群外部访问？
  └── 否 → 使用 ClusterIP
  └── 是 → 运行在云平台且有 LB 可用？
        └── 是 → 使用 LoadBalancer
        └── 否 → 使用 NodePort（或部署 MetalLB 后使用 LoadBalancer）

需要客户端直接发现所有 Pod IP？
  └── 是 → 使用 Headless Service (clusterIP: None)

需要稳定的 Pod 网络标识？
  └── 是 → 使用 Headless Service + StatefulSet

只需要简单的 DNS 别名指向外部？
  └── 是 → 使用 ExternalName
```

### 生产环境推荐实践

1. **集群内部服务间通信**：默认使用 ClusterIP。一个典型的微服务集群中，90% 以上的 Service 都是 ClusterIP 类型。

2. **对外暴露服务**：
   - 云环境：优先使用 LoadBalancer + `externalTrafficPolicy: Local`（保留源 IP）
   - 裸金属/自建机房：使用 NodePort 配合外部硬件 LB（F5/HAProxy/Nginx），或者部署 MetalLB
   - 推荐额外部署 Ingress Controller（Nginx Ingress/Traefik/HAProxy Ingress），统一管理七层路由

3. **Stateful 服务**：使用 Headless Service + StatefulSet 组合。这是 Cassandra、Kafka、ZooKeeper、Elasticsearch 等的标准部署模式。

4. **外部依赖**：使用 ExternalName 或手动管理的 Headless Service。特别是云厂商托管服务（如 AWS RDS、ElastiCache）通过 ExternalName 引入集群内部 DNS。

5. **端口规划**：NodePort 的 30000-32767 范围是集群级别共享的。大型集群中，建议通过 `--service-node-port-range` 扩展范围或使用指定端口来避免冲突。

# 3.Service YAML详解

## 3.1 完整YAML结构

一个完整的 Service YAML 包含以下所有可用字段：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service                          # Service 名称（同 Namespace 内唯一）
  namespace: default                        # 命名空间
  labels:                                   # Service 本身的标签
    app: my-app
    team: backend
  annotations:                              # 注解（元数据，云厂商或 Ingress 使用）
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  type: ClusterIP                           # Service 类型: ClusterIP|NodePort|LoadBalancer|ExternalName
  clusterIP: 10.96.100.50                   # 可选：显式指定 ClusterIP，或设为 None (Headless)
  clusterIPs:                               # 双栈场景下的 ClusterIP 列表
  - 10.96.100.50
  - 2001:db8::1
  selector:                                 # Pod 标签选择器
    app: my-app
    version: v1
  ports:
  - name: http                              # 端口名称（多端口时必须命名）
    protocol: TCP                           # 协议：TCP|UDP|SCTP
    port: 80                                # Service 暴露的端口
    targetPort: 8080                        # 目标 Pod 的端口（可以是数字或端口名称）
    nodePort: 30080                         # Node 端口（仅 NodePort/LoadBalancer 类型，30000-32767）
    appProtocol: http                       # 应用层协议标识（v1.19+）
  - name: grpc
    protocol: TCP
    port: 50051
    targetPort: grpc-port                   # 引用容器端口名称
  sessionAffinity: None                     # 会话亲和性：None|ClientIP
  sessionAffinityConfig:                    # 会话亲和性配置（仅当 sessionAffinity=ClientIP 时）
    clientIP:
      timeoutSeconds: 10800                 # 超时时间 0-86400，默认 10800(3小时)
  publishNotReadyAddresses: false           # 是否发布未就绪 Pod 的地址
  externalTrafficPolicy: Cluster            # 外部流量策略：Cluster|Local
  internalTrafficPolicy: Cluster            # 内部流量策略（v1.22+）：Cluster|Local
  ipFamilies:                               # IP 协议族列表（v1.20+）
  - IPv4
  ipFamilyPolicy: SingleStack               # IP 协议族策略：SingleStack|PreferDualStack|RequireDualStack
  loadBalancerIP: 203.0.113.50             # 请求特定的 LB IP
  loadBalancerSourceRanges:                 # 允许访问 LB 的源 IP 段
  - 10.0.0.0/8
  allocateLoadBalancerNodePorts: true       # 是否为 LB 分配 NodePort（v1.20+）
  externalName: external.example.com        # ExternalName 类型的外部域名
  healthCheckNodePort: 32000               # 自定健康检查 NodePort
  externalIPs:                              # 可接收流量的外部 IP 列表（不推荐）
  - 1.2.3.4
status:
  loadBalancer:                             # LB 状态（由云控制器填充）
    ingress:
    - ip: 203.0.113.50
    - hostname: a1234.elb.amazonaws.com
  conditions:                               # Service 的条件状态
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-01-01T00:00:00Z"
```

### 各字段层级关系

```
Service
├── metadata (标准资源元数据)
│   ├── name
│   ├── namespace
│   ├── labels
│   └── annotations
├── spec (Service 规格)
│   ├── type (ClusterIP/NodePort/LoadBalancer/ExternalName)
│   ├── clusterIP / clusterIPs
│   ├── selector
│   ├── ports[]
│   │   ├── name
│   │   ├── protocol
│   │   ├── port
│   │   ├── targetPort
│   │   ├── nodePort
│   │   └── appProtocol
│   ├── sessionAffinity
│   ├── sessionAffinityConfig
│   ├── publishNotReadyAddresses
│   ├── externalTrafficPolicy
│   ├── internalTrafficPolicy
│   ├── ipFamilies / ipFamilyPolicy
│   ├── loadBalancerIP
│   ├── loadBalancerSourceRanges
│   ├── allocateLoadBalancerNodePorts
│   ├── externalName
│   └── externalIPs
└── status (运行时状态)
    └── loadBalancer
        └── ingress[]
```

---

## 3.2 port/targetPort/nodePort/containerPort含义与关系

Service 配置中有多个端口相关的字段，它们之间的关系经常让初学者困惑。以下是每个字段的精确定义：

### port (Service 端口)

**定义**：Service 对外暴露的端口，即客户端连接 Service 时使用的端口。

```
客户端 → Service_IP:port → ...
```

`port` 是 Service 本身的属性。对于 ClusterIP 类型的 Service，客户端通过 `ClusterIP:port` 来访问。对于 NodePort/LoadBalancer 类型，这个端口也会被保留。

### targetPort (目标端口)

**定义**：后端 Pod 上容器监听的实际端口。Service 将流量转发到 Pod 的 `targetPort`。

```
... → Service(port) → DNAT → Pod_IP:targetPort
```

`targetPort` 可以是一个数字（如 8080），也可以引用 Pod 定义中容器的端口名称。

```yaml
# 方式1: 使用数字
ports:
- port: 80
  targetPort: 8080

# 方式2: 使用容器端口名称
# Pod 中定义: ports: [{name: http, containerPort: 8080}]
ports:
- port: 80
  targetPort: http         # 引用端口名称
```

使用端口名称的好处是：即使 Pod 中的实际端口号改变（比如从 8080 改为 8081），只要保持端口名称不变，Service 就不需要修改。这提高了 Pod 模板与 Service 之间的解耦程度。

### nodePort (节点端口)

**定义**：在集群每个 Node 上暴露的端口，用以接收来自集群外部的流量。仅在 `type: NodePort` 或 `type: LoadBalancer` 时有效。

```
外部客户端 → Node_IP:nodePort → Service_IP:port → DNAT → Pod_IP:targetPort
```

- 端口范围：默认 30000-32767
- 可以通过 `--service-node-port-range` 配置
- 如果未指定，系统自动分配

### containerPort (容器端口)

**定义**：在 Pod 的 Container 规格中定义的端口。

```yaml
# Pod YAML 中
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - name: http
      containerPort: 8080        # 这是 containerPort
      protocol: TCP
```

`containerPort` 在容器层面是**信息性的**——它告诉容器运行时这个容器会监听哪个端口。设置 `containerPort` 并不会让端口自动暴露出来（容器的端口默认都是可达的），但它有以下作用：

1. 为 `targetPort` 提供命名引用
2. 生成环境变量 `{SERVICE}_PORT` 等
3. 在 Service 定义中通过端口名称 `targetPort: http` 来引用
4. 供平台工具和监控系统使用

### 四种端口的关系图

```
                     ┌──────────────────────────────────┐
                     │          Service                  │
                     │                                  │
                     │  port: 80                        │
 external traffic ──→│  (Service 对外的端口)              │
                     │         │                        │
                     │         │ 映射关系                │
                     │         ▼                        │
                     │  targetPort: 8080                │
                     │  (转发到 Pod 的端口)              │
                     │         │                        │
                     │         │ 转发到                  │
                     └─────────┼────────────────────────┘
                               │
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
           ┌──────────┐ ┌──────────┐ ┌──────────┐
           │  Pod A   │ │  Pod B   │ │  Pod C   │
           │          │ │          │ │          │
           │containerPort: 8080   │ │          │
           │(Pod 内部容器监听端口) │ │          │
           │          │ │          │ │          │
           │  ┌─────┐ │ │ ┌─────┐  │ │ ┌─────┐ │
           │  │App  │ │ │ │App  │  │ │ │App  │ │
           │  │:8080│ │ │ │:8080│  │ │ │:8080│ │
           │  └─────┘ │ │ └─────┘  │ │ └─────┘ │
           └──────────┘ └──────────┘ └──────────┘

nodePort: 30080 (仅在 NodePort/LoadBalancer 类型)
   外部客户端 → Node_IP:nodePort → Service_IP:port → Pod_IP:targetPort
```

### 常见配置范例

```yaml
# 场景1: Service 端口与 Pod 端口相同（简化配置）
ports:
- port: 8080
  targetPort: 8080       # 可以不写 targetPort（未指定时默认等于 port）

# 场景2: Service 端口与 Pod 端口不同（隐藏内部端口）
ports:
- port: 80
  targetPort: 8080       # 对外暴露 80，内部 Pod 实际监听 8080

# 场景3: NodePort 外部访问
ports:
- name: web
  port: 80
  targetPort: 8080
  nodePort: 30080        # 外部访问 Node:30080 → Pod:8080

# 场景4: 多端口 Service
ports:
- name: http
  port: 80
  targetPort: 8080
- name: metrics
  port: 9090
  targetPort: 9090
- name: grpc
  port: 50051
  targetPort: 50051
```

---

## 3.3 SessionAffinity (ClientIP, 超时配置)

### 会话亲和性的概念

默认情况下，Service 对每个新连接独立地进行负载均衡（随机或轮询）。这意味着同一个客户端发起的多次连接可能被转发到不同的后端 Pod。对于大多数无状态应用来说，这是理想的行为。

但某些应用需要**会话亲和性**（Session Affinity / Sticky Session）：来自同一个客户端的连接始终转发到同一个后端 Pod。这在使用本地缓存、WebSocket 连接、或有状态协议时是必需的。

### Kubernetes 支持的会话亲和性

Kubernetes 只支持一种会话亲和性策略：**ClientIP**。它基于客户端的源 IP 地址来决定将连接转发到哪个 Pod。相同源 IP 的客户端在 `timeoutSeconds` 时间段内，始终连接到同一个后端 Pod。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  sessionAffinity: ClientIP                  # 启用会话亲和性
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800                  # 默认 10800 秒 (3小时)
```

### ClientIP 的底层实现

**iptables 模式**：kube-proxy 使用 iptables 的 `recent` 模块来实现 ClientIP 会话亲和性。规则逻辑如下：

```
1. 新连接到达 KUBE-SVC-XXXX 链
2. 检查 recent 模块中的 "已记录客户端" 列表
3. 如果该源 IP 在列表中且未超时：
   → 跳转到同一个 KUBE-SEP-XXXX 链（与上次相同的后端 Pod）
4. 如果该源 IP 不在列表中或已超时：
   → 在 KUBE-SEP 链中随机选择
   → 记录 "源 IP → 后端 Pod 映射" 到 recent 列表
   → 设置超时计时器
```

**IPVS 模式**：IPVS 原生支持持久连接（Persistence）。kube-proxy 将 `sessionAffinity: ClientIP` 翻译为 IPVS 的持久性参数。

### 超时配置详解

```yaml
sessionAffinityConfig:
  clientIP:
    timeoutSeconds: 10800     # 范围: 0 - 86400 (24小时)
                              # 默认: 10800 (3小时)
                              # 0 表示永不过期（不推荐）
```

超时时间的含义：从**最后一次该客户端 IP 建立新连接**开始计算。如果在这段时间内该客户端 IP 没有建立新的连接，则亲和性过期。下次新连接可能被分配到不同的后端 Pod。

```yaml
# 示例：短会话亲和性
sessionAffinity: ClientIP
sessionAffinityConfig:
  clientIP:
    timeoutSeconds: 300       # 5分钟无活动后重新分配
```

### 注意事项与局限性

1. **只支持 ClientIP**：Kubernetes 原生不支持基于 Cookie 或其他 HTTP 层信息的会话亲和性。需要七层会话亲和性时，使用 Ingress Controller（如 Nginx Ingress 通过 annotation `nginx.ingress.kubernetes.io/affinity: cookie` 支持）。

2. **SNAT 的影响**：当使用 `externalTrafficPolicy: Cluster`（默认）时，来自集群外部的请求的源 IP 会被 SNAT 为接收 Node 的 IP。这意味着所有来自集群外部的请求看起来都来自相同的 IP（接收 Node 的 IP），会话亲和性失效。

   **解决方案**：设置 `externalTrafficPolicy: Local`，这样源 IP 会被保留。但代价是如果接收 Node 上没有后端 Pod，请求会被丢弃。

3. **Client IP 粒度过粗**：来自同一个 NAT 网关后的所有客户端共享同一个源 IP，它们会被分配到同一个后端 Pod，可能导致负载不均衡。

4. **Kubernetes 只支持 ClientIP 级别的亲和性**：这是四层（L4）的亲和性限制。对于需要 HTTP Cookie 级别的亲和性，应该使用 Ingress Controller 来实现。

5. **超时期间的后端变化**：如果在超时时间内，原始后端 Pod 被删除，则亲和性被打破。kube-proxy 会移除不可用后端的规则，后续连接将重新负载均衡到其他 Pod。

---

## 3.4 publishNotReadyAddresses

### 默认行为

默认情况下（`publishNotReadyAddresses: false`），只有通过了 Readiness Probe 的 Pod 才会出现在 Service 的 Endpoints 中，并接收流量。这确保了：
- 启动中的 Pod 不会收到请求（避免冷启动导致的超时）
- 就绪探测失败的 Pod 不会收到请求（自动摘除不健康的 Pod）

### publishNotReadyAddresses: true 的使用场景

当设置为 `true` 时，即使 Pod 还未就绪（Readiness Probe 未通过），它也会出现在 Service 的 Endpoints 中。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stateful-service
spec:
  publishNotReadyAddresses: true           # 关键配置
  clusterIP: None
  selector:
    app: cassandra
  ports:
  - port: 9042
```

### 场景分析

**场景1：分布式数据库集群的发现**
Cassandra、Elasticsearch 等系统在启动阶段需要互相发现和通信来完成集群组建。如果启动中的节点不暴露其地址，其他节点无法联系它，集群可能永远无法形成。通过 `publishNotReadyAddresses: true`，即使 Pod 还在启动中，其 IP 也在 DNS 中可解析，其他节点可以发现并与之通信。

**场景2：StatefulSet 的顺序启动**
StatefulSet 默认按序启动（从 0 到 N-1）。Pod-1 的启动可能依赖 Pod-0 已启动但不一定需要 Pod-0 完全就绪。通过 `publishNotReadyAddresses: true`，Pod-1 可以在 Pod-0 还未完全就绪之前就解析 Pod-0 的地址。

### 风险与注意事项

- **流量丢失**：客户端可能将请求发送给尚未就绪的 Pod，导致请求失败
- **需要应用层重试**：启用此配置的应用必须能处理连接失败或请求超时
- **与真正的就绪探测冲突**：Readiness Probe 的作用被部分削弱

### 与 externalTrafficPolicy 的交互

当 `publishNotReadyAddresses: true` 与 `externalTrafficPolicy: Local` 组合使用时：
- 未就绪的 Pod 地址被发布
- 但 externalTrafficPolicy 只将外部流量转发到本节点的 Pod
- 如果一个 Node 上只有未就绪的 Pod，外部请求可能被拒绝

---

## 3.5 externalTrafficPolicy (Cluster/Local)

`externalTrafficPolicy` 控制来自 Service 外部的流量的路由行为。它对以下流量有效：
- NodePort 类型的外部流量
- LoadBalancer 类型的外部流量

它**不影响**集群内部的流量（Pod 到 Pod 通过 ClusterIP 的流量）。

### externalTrafficPolicy: Cluster（默认值）

**行为**：
- 外部流量到达任何 Node 的 NodePort 后，可以被转发到集群内**任意 Node** 上的后端 Pod
- 无论请求到达哪个 Node，只要集群内有 Ready 的 Pod，请求就能被处理

**流量路径**：

```
外部客户端 (10.0.1.100)
    │
    │ 连接到 Node2:30080
    ▼
Node2 (10.0.0.2)
    │ iptables: 目标端口 30080
    │ DNAT → ClusterIP:80
    │
    │ 源 IP 被 SNAT (改写成 Node2 的 IP: 10.0.0.2)
    │ 这是为了确保响应能回到 Node2
    ▼
Node2 → 跨节点路由 → Node1
    │
    ▼
Node1 (10.0.0.1)
    │ 后端 Pod (10.244.1.5:8080)
    ▼
Pod 收到请求，源 IP = 10.0.0.2 (Node2 的 IP)
Pod 响应 → Node2 → 外部客户端
```

**核心问题：源 IP 丢失**
由于 SNAT，后端 Pod 看到的源 IP 是 Node 的 IP，而不是客户端的真实 IP。许多应用需要知道客户端真实 IP（日志审计、地理位置、限流、白名单等）。

**额外跳数**：
如果外部请求到达 Node2，但后端 Pod 在 Node1 上，数据包需要跨节点传输一次，增加延迟。

### externalTrafficPolicy: Local

**行为**：
- 外部流量到达某个 Node 后，**只能**转发到该 Node 上的后端 Pod
- 如果接收请求的 Node 上没有该 Service 的后端 Pod，连接被拒绝（数据包被丢弃）
- 不会进行 SNAT，后端 Pod 可以直接看到客户端的真实 IP

**流量路径**：

```
外部客户端 (10.0.1.100)
    │
    │ 连接到 Node1:30080
    ▼
Node1 (10.0.0.1)
    │ iptables: 目标端口 30080
    │ 检查本节点是否有 Ready 的 Pod
    │ 有: DNAT 直接到本节点 Pod IP:8080
    │ 没有: 丢弃数据包
    │
    ▼
后端 Pod (10.244.1.5:8080)
Pod 收到请求，源 IP = 10.0.1.100 (客户端真实 IP 保留)
响应直接返回 Node1 网络栈 → 外部客户端
```

**优势**：
- 保留客户端源 IP
- 减少跨节点跳数（更低的延迟）
- 不需要额外 SNAT 规则

**劣势**：
- 负载均衡粒度较粗：外部请求只能到达有 Pod 的 Node
- 需要在外部 LB 层配置健康检查
- 如果某个 Node 上 Pod 全部不可用，该 Node 会拒绝外部请求

### 配置示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local        # 保留客户端源 IP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

### 与 LoadBalancer 的配合

当 `externalTrafficPolicy: Local` 与 LoadBalancer 配合时：
- 云 LB 的健康检查需要能检测到 Node 上是否有 Ready 的 Pod
- 通过 `spec.healthCheckNodePort` 指定健康检查端口

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  healthCheckNodePort: 32000         # kube-proxy 在此端口暴露健康检查
```

kube-proxy 在所有有 Ready Pod 的 Node 上监听 `healthCheckNodePort` 端口并返回 200 OK，在无 Ready Pod 的 Node 上拒绝连接。云 LB 据此决定哪些 Node 可以接收流量。

---

## 3.6 internalTrafficPolicy (Cluster/Local, K8s v1.22+)

`internalTrafficPolicy` 是 Kubernetes v1.22 引入的功能（Beta，v1.26 GA），控制集群**内部**流量的路由行为（即 Pod 到 Service 的流量）。

### internalTrafficPolicy: Cluster（默认值）

**行为**：集群内部流量可以被转发到集群中**任何 Node** 上的后端 Pod，不限制在源 Pod 所在的 Node。

### internalTrafficPolicy: Local

**行为**：集群内部流量**只能**转发到与源 Pod **在同一 Node** 上的后端 Pod。如果源 Pod 所在的 Node 上没有该 Service 的后端 Pod，流量被丢弃。

**配置示例**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cache-service
spec:
  selector:
    app: redis
  ports:
  - port: 6379
  internalTrafficPolicy: Local       # 优先使用本节点 Pod
```

### 使用场景

1. **拓扑感知访问**：将流量限制在本节点，减少跨节点网络延迟
2. **本地缓存优化**：DaemonSet 部署的缓存代理（如 Envoy Sidecar）
3. **节点级故障隔离**：本节点后端不可用，即快速失败，不将流量发送到其他节点（避免级联效应）

### internalTrafficPolicy 与 externalTrafficPolicy 的组合

```yaml
# 场景: 外部流量本地保留源IP，内部流量也本地化
spec:
  externalTrafficPolicy: Local
  internalTrafficPolicy: Local
```

### 与 Service Topology/Topology Aware Hints 的关系

`internalTrafficPolicy: Local` 是最简单的拓扑感知形式——按节点限制。更复杂的拓扑感知（如按可用区 AZ）需要使用 Topology Aware Hints 特性。

---

## 3.7 ipFamily/ipFamilyPolicy (IPv4/IPv6单栈或双栈)

Kubernetes v1.20 引入了双栈（Dual-Stack）支持（Alpha），v1.23 成为 Beta（默认启用），v1.27 达到 Stable。双栈允许 Service 同时拥有 IPv4 和 IPv6 两个 ClusterIP。

### ipFamilyPolicy

控制 Service 使用哪种 IP 协议族：

| 值 | 含义 |
|----|------|
| `SingleStack` | 单栈：只分配一个 ClusterIP（IPv4 或 IPv6） |
| `PreferDualStack` | 优先双栈：如果集群支持双栈则分配两个 ClusterIP，否则回退到单栈 |
| `RequireDualStack` | 必须双栈：必须分配两个 ClusterIP，如果集群不支持则 Service 创建失败 |

### ipFamilies

指定 IP 协议族的顺序。第一个 IP 族用于 `spec.clusterIP`，所有族用于 `spec.clusterIPs`。

```yaml
# IPv4 单栈（默认）
apiVersion: v1
kind: Service
metadata:
  name: single-stack-ipv4
spec:
  ipFamilyPolicy: SingleStack
  ipFamilies:
  - IPv4
  selector:
    app: my-app
  ports:
  - port: 80
```

```yaml
# IPv6 单栈
apiVersion: v1
kind: Service
metadata:
  name: single-stack-ipv6
spec:
  ipFamilyPolicy: SingleStack
  ipFamilies:
  - IPv6
  selector:
    app: my-app
  ports:
  - port: 80
```

```yaml
# 双栈
apiVersion: v1
kind: Service
metadata:
  name: dual-stack-svc
spec:
  ipFamilyPolicy: RequireDualStack
  ipFamilies:
  - IPv4
  - IPv6
  clusterIP: 10.96.100.50
  clusterIPs:
  - 10.96.100.50
  - 2001:db8::1
  selector:
    app: my-app
  ports:
  - port: 80
```

### 集群级别的双栈配置

要启用双栈支持，kube-apiserver、kube-controller-manager 和 kube-proxy 都需要配置双网段：

**kube-apiserver**:
```
--service-cluster-ip-range=10.96.0.0/12,2001:db8::/108
--feature-gates=IPv6DualStack=true
```

**kube-controller-manager**:
```
--cluster-cidr=10.244.0.0/16,2001:db8:1::/48
--service-cluster-ip-range=10.96.0.0/12,2001:db8::/108
--feature-gates=IPv6DualStack=true
```

**kube-proxy**:
```
--cluster-cidr=10.244.0.0/16,2001:db8:1::/48
--feature-gates=IPv6DualStack=true
```

### DNS 中的双栈记录

双栈 Service 在 CoreDNS 中会同时创建 A 记录（IPv4）和 AAAA 记录（IPv6）：

```shell
$ nslookup my-service.default.svc.cluster.local
Name:    my-service.default.svc.cluster.local
Address: 10.96.100.50         # A 记录 (IPv4)
Address: 2001:db8::1          # AAAA 记录 (IPv6)
```

---

## 3.8 allocateLoadBalancerNodePorts

`allocateLoadBalancerNodePorts` 控制是否为 LoadBalancer 类型的 Service 自动分配 NodePort。这在 v1.20 引入，用于优化不需要 NodePort 的场景。

### allocateLoadBalancerNodePorts: true（默认）

对于 LoadBalancer 类型 Service，除了分配 LB 外部 IP，还会自动分配 NodePort。外部 LB 通过 NodePort 将流量转发到 Node。

```yaml
spec:
  type: LoadBalancer
  allocateLoadBalancerNodePorts: true   # 默认行为
```

### allocateLoadBalancerNodePorts: false

在某些场景下，NodePort 是不必要的：

1. **使用容器网络接口 (CNI) 直接路由到 Pod**：某些云平台支持 LB 直接向 Pod IP 发送流量（如 AWS VPC CNI 的 Pod SNAT 禁用模式）
2. **使用 AWS Load Balancer Controller 的 IP 模式**：在 AWS EKS 上使用 AWS Load Balancer Controller 的 `ip` 模式，LB 直接将流量发送到 Pod IP
3. **安全性考虑**：不希望在每个 Node 上开放额外的端口

```yaml
spec:
  type: LoadBalancer
  allocateLoadBalancerNodePorts: false   # 不分配 NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

设置后，Service 仍然创建，但不会占用 NodePort 范围内的端口。

# 4.Service底层实现

## 4.1 kube-proxy角色与工作方式

### kube-proxy 的核心职责

kube-proxy 是运行在 Kubernetes 集群每个 Node 上的网络代理（以 DaemonSet、静态 Pod 或系统进程方式部署）。它是 Service 抽象的具体实现者，负责将发往 Service ClusterIP 的流量转发到后端的 Pod。

kube-proxy 的核心职责包括：
1. 监听 API Server 中 Service 和 Endpoints/EndpointSlice 的变更
2. 在每个 Node 上维护网络转发规则（iptables/IPVS/eBPF）
3. 将发往 Service VIP 的流量负载均衡到后端 Pod
4. 在后端 Pod 集合变化时更新规则

### kube-proxy 的架构

kube-proxy 的内部架构包含以下关键组件：

```
+-----------------------------------------------------+
|                   kube-proxy                         |
|                                                      |
|  +--------------+    +----------------------+       |
|  |  Informer    |    |   Proxier Interface   |       |
|  |  (Watch API) |--->|   +----------------+  |       |
|  +--------------+    |   |  iptables       |  |       |
|                       |   |  Proxier        |  |       |
|  +--------------+    |   +----------------+  |       |
|  |  Endpoints   |    |                        |       |
|  |  Slice       |    |   +----------------+  |       |
|  |  Informer    |--->|   |  IPVS Proxier   |  |       |
|  +--------------+    |   +----------------+  |       |
|                       |                        |       |
|  +--------------+    |   +----------------+  |       |
|  |  Node        |    |   |  nftables       |  |       |
|  |  Informer    |--->|   |  Proxier        |  |       |
|  +--------------+    |   +----------------+  |       |
|                       +----------------------+       |
+-----------------------------------------------------+
```

### Informer 机制

kube-proxy 使用 Informer 模式来高效监听 API 变化，而不是轮询。每次 `syncProxyRules()` 调用都会：

1. 构建完整的期望规则集合
2. 计算与当前规则的差异（增量更新，iptables 模式下是完整重建）
3. 写入新规则到内核

### kube-proxy 的三种工作模式

| 模式 | 引入版本 | 成熟度 | 默认/推荐 |
|------|---------|--------|----------|
| **userspace** | v1.0 | **已弃用** | - |
| **iptables** | v1.2 | GA | 长期默认，仍然广泛使用 |
| **IPVS** | v1.9 (Beta), v1.11 GA | GA | 高性能场景推荐 |
| **nftables** | v1.29 (Alpha) | Alpha | 未来可能取代 iptables |
| **eBPF (Cilium)** | 第三方 | 第三方 | 云原生网络未来方向 |

### kube-proxy 启动参数

```bash
kube-proxy \
  --kubeconfig=/var/lib/kube-proxy/kubeconfig.conf \
  --proxy-mode=iptables \               # 模式: iptables, ipvs, nftables
  --cluster-cidr=10.244.0.0/16 \        # Pod CIDR（用于 IPVS 模式确定出流量 SNAT）
  --masquerade-all=false \              # 是否对所有流量进行 SNAT
  --conntrack-max-per-core=32768 \      # 每个 CPU 核心的 conntrack 表项上限
  --conntrack-tcp-timeout-established=86400 \  # TCP 已建立连接超时
  --conntrack-tcp-timeout-close-wait=3600 \
  --ipvs-scheduler=rr \                 # IPVS 调度算法
  --iptables-sync-period=30s \          # iptables 规则同步间隔
  --nodeport-addresses=                 # NodePort 绑定的 IP 地址
  --healthz-bind-address=0.0.0.0:10256  # 健康检查地址
  --metrics-bind-address=0.0.0.0:10249  # Metrics 暴露地址
```

---

## 4.2 iptables模式详解 (规则链: KUBE-SERVICES->KUBE-SVC->KUBE-SEP, DNAT)

### iptables 基础回顾

iptables 是 Linux 内核的 netfilter 框架的用户态管理工具。它通过在内核中创建规则链（Chain）来管理网络数据包的流向。

netfilter 提供了 5 个钩子（Hook）点：

```
网络层数据包路径:

-> [NF_IP_PRE_ROUTING] -> [路由决策] -> [NF_IP_FORWARD] -> [NF_IP_POST_ROUTING] ->
                                |
                        [NF_IP_LOCAL_IN]
                                ^
                         [本地进程]
                                |
                        [NF_IP_LOCAL_OUT]
```

kube-proxy 主要在 `PREROUTING`、`OUTPUT` 和 `POSTROUTING` 链上添加规则。

### kube-proxy 的 iptables 规则结构

kube-proxy 在 iptables 的 `nat` 表中创建了一系列自定义链：

```
nat 表结构（kube-proxy 部分）:

PREROUTING
  +-- KUBE-SERVICES          # 所有 Service 的入口链
        +-- 匹配 ClusterIP:Port -> KUBE-SVC-XXXX
        |                         +-- (概率匹配) -> KUBE-SEP-AAAA -> DNAT -> Pod A
        |                         +-- (概率匹配) -> KUBE-SEP-BBBB -> DNAT -> Pod B
        |                         +-- (兜底)      -> KUBE-SEP-CCCC -> DNAT -> Pod C
        |
        +-- 匹配 NodePort -> KUBE-NODEPORTS -> KUBE-SVC-XXXX -> ...
        +-- 匹配 ExternalIP -> ...

OUTPUT
  +-- KUBE-SERVICES          # 同 PREROUTING，本地进程出流量也走此链

POSTROUTING
  +-- KUBE-POSTROUTING        # 出站流量 SNAT
        +-- MASQUERADE (Service 流量 SNAT)
        +-- MASQUERADE (Pod 流量伪装为 Node IP)
```

### 自定义链的命名规则

kube-proxy 使用 Service 名称和 Namespace 的哈希值来命名自定义链：

- `KUBE-SERVICES`：主入口链，所有的 Service 匹配规则跳转到此链
- `KUBE-SVC-<hash>`：每个 Service 对应一个链，实现该 Service 的后端分布
- `KUBE-SEP-<hash>`：每个 Service Endpoint（即后端 Pod）对应一个链
- `KUBE-NODEPORTS`：NodePort 类型 Service 的入口链
- `KUBE-POSTROUTING`：出站流量的 SNAT 链
- `KUBE-MARK-MASQ`：标记需要 SNAT 的数据包
- `KUBE-MARK-DROP`：标记需要丢弃的数据包

### 概率性负载均衡实现

iptables 本身不支持加权轮询或真正的负载均衡。kube-proxy 使用 iptables 的 `statistic` 模块来实现概率性随机负载均衡：

```
对于 N 个后端 Pod，kube-proxy 创建 N 条规则，每条规则有不同的匹配概率：

Pod A: 概率 1/3 (33.33%)
Pod B: 概率 1/2 (50% 的剩余流量 = 33.33%)
Pod C: 概率 1/1 (100% 的剩余流量 = 33.33%)

规则:
1. -m statistic --mode random --probability 0.33333 -> KUBE-SEP-AAAA (Pod A)
2. -m statistic --mode random --probability 0.50000 -> KUBE-SEP-BBBB (Pod B)
3. 无条件跳转                                      -> KUBE-SEP-CCCC (Pod C)
```

每条新连接被概率匹配，被分配到不同的后端 Pod。通过 conntrack，同一个连接的所有后续数据包都发送到同一个后端。

### KUBE-SEP 链的内部

单个 KUBE-SEP 链包含以下规则：

```
KUBE-SEP-XXXX:
  1. 如果数据包被标记为 LOCAL（本地生成）且目标地址与源地址相同:
     MARK 0x4000 (标记为需要 MASQUERADE，解决 hairpin NAT 问题)
  2. DNAT: 目标 IP -> Pod IP, 目标端口 -> targetPort
```

### DNAT 与 SNAT 的完整流程

以集群内 Pod A（10.244.1.5）访问 ClusterIP（10.96.100.42:80）-> 后端 Pod B（10.244.2.8:8080）为例：

```
源 Pod: 10.244.1.5:45678
目标: 10.96.100.42:80

1. Pod A 发出 TCP SYN:
   src=10.244.1.5:45678, dst=10.96.100.42:80

2. 进入 Pod A 所在 Node 的 OUTPUT 链
   -> KUBE-SERVICES 链
   -> 匹配 ClusterIP:Port 规则
   -> 跳转到 KUBE-SVC-XXXX

3. KUBE-SVC-XXXX 链
   -> 概率匹配到 KUBE-SEP-BBBB

4. KUBE-SEP-BBBB 链
   -> DNAT: dst=10.96.100.42:80 -> dst=10.244.2.8:8080
   -> 此时: src=10.244.1.5:45678, dst=10.244.2.8:8080

5. 路由决策: 目标是另一个 Node 上的 Pod
   -> 通过 CNI 网络跨节点发送

6. POSTROUTING 链
   -> KUBE-POSTROUTING 链
   -> 可能需要 SNAT/MASQUERADE (取决于 CNI 配置)

7. 数据包到达 Pod B 所在 Node
   -> 通过 bridge/veth pair 到达 Pod B
   -> Pod B 收到 SYN: src=10.244.1.5:45678, dst=10.244.2.8:8080
```

### Hairpin NAT 问题

当一个 Pod 通过 Service ClusterIP 访问同一个 Service，而负载均衡恰好选中了该 Pod 自己时，会产生 Hairpin NAT 问题：

```
Pod A (10.244.1.5:45678) -> ClusterIP (10.96.100.42:80) -> DNAT -> Pod A (10.244.1.5:8080)

问题: 目标 IP 变成了 Pod 自己的 IP，且目标端口与源端口不同。
     网络栈需要将数据包"回环"到同一个网络接口。
     这在一些 CNI 实现中可能失败。
```

kube-proxy 通过 MARK 标记来解决此问题（在 KUBE-SEP 链中标记 hairpin 流量），然后依赖 CNI 和 bridge 处理回环流量。

### iptables 模式的可观察性

```shell
# 查看 nat 表中 kube-proxy 创建的所有链
iptables -t nat -L | grep KUBE

# 查看特定 Service 的规则
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.100.42

# 查看 KUBE-SVC 链的概率分布
iptables -t nat -L KUBE-SVC-XXXX -n -v

# 查看 conntrack 中的 Service 连接
conntrack -L -d 10.96.100.42

# 查看 iptables 规则数量
iptables -t nat -S | wc -l
# 大型集群中可能达到数万甚至十万条规则
```

---

## 4.3 IPVS模式详解 (虚拟服务器+真实服务器, 调度算法)

### IPVS 简介

**IPVS**（IP Virtual Server）是 Linux 内核的一个模块，实现了传输层的负载均衡。它是 Linux Virtual Server（LVS）项目的核心组件，从 Linux 2.4 开始就已存在于内核中。

与 iptables 不同，IPVS 是一个**专门的负载均衡框架**，它维护一个高效的连接调度表，支持多种调度算法，并且性能不受规则数量线性影响。

### IPVS 的核心概念

IPVS 有三个核心概念：

1. **虚拟服务器（Virtual Server）**：对应 Kubernetes 的 Service（ClusterIP + Port 组合）
2. **真实服务器（Real Server）**：对应 Kubernetes 的后端 Pod（Pod IP + Port）
3. **调度算法（Scheduler）**：决定将请求分配给哪个真实服务器的算法

```
虚拟服务器 (VIP:VPort)
  +-----------------+
  | 10.96.100.42:80 |  <- ClusterIP:Port
  +--------+--------+
           | 调度算法 (rr/wlc/lc/sed/nq/dh/sh/...)
     +-----+-----+
     v     v     v
  真实服务器    真实服务器    真实服务器
  10.244.1.5    10.244.2.8    10.244.3.12
  :8080         :8080         :8080
```

### kube-proxy 如何创建 IPVS 规则

kube-proxy 通过 netlink 接口直接与内核 IPVS 模块通信，而不是通过 iptables 命令。创建流程简化为：

1. 获取所有 Service 的期望状态
2. 对每个 Service 查找或创建虚拟服务器（VIP:Port）
3. 对每个 Service 的后端 Pod 添加真实服务器（Pod IP:targetPort）
4. 清理不存在的后端和虚拟服务器

### 支持的调度算法

IPVS 支持多种调度算法，通过 `kube-proxy` 的 `--ipvs-scheduler` 参数配置：

| 算法 | 全称 | 描述 |
|------|------|------|
| **rr** | Round Robin | 轮询分配，默认算法 |
| **wrr** | Weighted Round Robin | 加权轮询 |
| **lc** | Least Connection | 最少连接数 |
| **wlc** | Weighted Least Connection | 加权最少连接 |
| **sh** | Source Hashing | 源地址哈希（基于源 IP） |
| **dh** | Destination Hashing | 目标地址哈希 |
| **sed** | Shortest Expected Delay | 最短预期延迟 |
| **nq** | Never Queue | 最少连接 + 从不排队 |

默认使用 `rr`（轮询）。对于大多数应用场景来说轮询就足够了。

### IPVS 的持久连接

IPVS 原生支持持久连接（Persistence），对应 Kubernetes 的 `sessionAffinity: ClientIP`。kube-proxy 将 `sessionAffinityConfig.clientIP.timeoutSeconds` 翻译为 IPVS 的持久超时参数。

```
ipvsadm -L -n --persistent-conn
IP Virtual Server version 1.2.1
Prot LocalAddress:Port            Flags    -> RemoteAddress:Port
TCP  10.96.100.42:80              rr persistent 10800
  -> 10.244.1.5:8080              Masq    1      0          0
  -> 10.244.2.8:8080              Masq    1      0          0
  -> 10.244.3.12:8080             Masq    1      0          0
```

### IPVS 模式下的 iptables 角色

IPVS 模式仍然需要 iptables，但角色大大简化。主要用于：
- 数据包的初始拦截（将发往 ClusterIP 的数据包引导到 IPVS）
- 必要的 SNAT/MASQUERADE
- 防火墙/mark 规则

iptables 规则数量从 O(N*M) 降低到 O(N)（N=Service 数，比 iptables 模式的 O(N*M) 低得多，M=Pod 数）。

### 观察 IPVS 状态

```shell
# 安装 ipvsadm 工具
apt-get install ipvsadm

# 查看所有虚拟服务器
ipvsadm -L -n

# 输出示例
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.100.42:80 rr
  -> 10.244.1.5:8080              Masq    1      0          0
  -> 10.244.2.8:8080              Masq    1      0          2
  -> 10.244.3.12:8080             Masq    1      0          0

# 查看连接统计
ipvsadm -L -n --stats

# 查看活跃连接
ipvsadm -L -n --rate
```

---

## 4.4 iptables vs IPVS 性能对比

### 规则规模对比

**iptables 模式**的规则数量与服务数量和后端 Pod 数量强相关：

```
# iptables 规则数量 ≈ O(N_Services x M_Pods)
# 1000 个 Service，每个 Service 10 个 Pod
# 规则数量约: 1000 x 10 x 2(NAT链 + 跳转链) = 20000+ 条规则
```

iptables 使用链表结构存储规则。每个数据包需要**线性遍历**匹配链中的规则，时间复杂度 O(N)。当规则数量上万时，每个新连接的数据包都要遍历数万条规则，延迟显著增加。

**IPVS 模式**使用哈希表存储虚拟服务器和真实服务器：

```
# IPVS 数据结构的复杂度
# 虚拟服务器查找: O(1) 哈希查找
# 真实服务器调度: O(1) (基于调度算法的常数时间选择)
```

### 性能基准测试

以下是典型测试场景的对比（来源于 Kubernetes 社区的性能测试数据）：

| 指标 | iptables | IPVS |
|------|----------|------|
| **1000 Service, 10K Pod** 规则数 | ~50,000+ 条 | ~2,000 条 iptables + 1,000 VS 10K RS |
| **新连接建立延迟 (P99)** | 5-10ms | <1ms |
| **规则同步时间** | 5-15秒（全量重建） | 1-3秒（增量更新） |
| **内存占用** | 中等（规则全在内存） | 较低 |
| **CPU 占用** | 高（规则同步 + 匹配） | 低 |
| **吞吐量** | 受规则数量线性影响 | 几乎不受影响 |

### iptables 的性能瓶颈

1. **全量规则同步**：kube-proxy 在 iptables 模式下，每次 `syncProxyRules()` 调用都会完全重建整个规则集。这会短暂锁定 iptables，导致瞬时的规则真空期。

2. **线性遍历匹配**：每个数据包进入 KUBE-SERVICES 链后，需要一条条匹配规则直到命中。对于 10000 条规则，最坏情况需要匹配 10000 次。

3. **conntrack 表压力**：iptables 依赖 conntrack 来追踪连接状态。每个连接占用一个 conntrack 条目，高并发场景下 conntrack 表溢出会导致丢包。

4. **规则更新不可中断**：iptables-restore 是原子操作，大量规则一次性提交。提交期间无法处理增量更新。

### 何时必须切换到 IPVS

如果你的集群出现以下信号，说明应该从 iptables 切换到 IPVS：

- **Service 总数超过 1000**
- **后端 Pod 总数超过 10000**
- **iptables 规则同步时间超过 5 秒**
- **conntrack 表频繁达到上限**（检查 `nf_conntrack: table full, dropping packet` 内核日志）
- **新连接建立 P99 延迟超过 10ms**
- **kube-proxy 进程 CPU 使用率持续 > 50%**

### 切换方法

```bash
# 1. 在所有 Node 上加载 IPVS 内核模块
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack

# 2. 安装 ipvsadm 工具（用于调试）
apt-get install ipvsadm

# 3. 修改 kube-proxy 配置
kubectl edit configmap kube-proxy -n kube-system
# 修改 mode: "ipvs"

# 4. 重启 kube-proxy Pod
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```

---

## 4.5 eBPF/Cilium替代kube-proxy的方式 (未来趋势)

### eBPF 技术简介

**eBPF**（Extended Berkeley Packet Filter）是 Linux 内核中的一项革命性技术。它允许在**内核空间**执行沙箱化的用户态程序，而不需要修改内核源代码或加载内核模块。

eBPF 程序可以挂载到内核的各种钩子点（如网络数据包处理路径上的 TC、XDP、套接字层），对数据包进行高效处理。

使用 eBPF 替代 iptables/IPVS 的优势：
- **不需要规则链表遍历**：eBPF 使用高效的哈希表查找
- **内核态直接处理**：避免用户态/内核态切换
- **可编程性强**：可以实现复杂的网络策略
- **性能极高**：接近线速的数据包处理

### Cilium 的 kube-proxy Replacement

**Cilium** 是一个基于 eBPF 的 CNI 插件和网络平台，它可以在不需要 kube-proxy 的情况下实现 Service 的负载均衡。

Cilium 使用 eBPF 程序直接在内核中处理 Service 的 DNAT：

```
传统路径（kube-proxy）:
  数据包 -> iptables/IPVS (内核) <- 规则由 kube-proxy 用户态进程管理

Cilium eBPF 路径:
  数据包 -> eBPF 程序 (内核) <- 程序由 Cilium Agent 直接加载到内核
```

Cilium 的 Service 负载均衡实现：
- 在 **BPF sockmap/sockhash** 层拦截 socket 操作
- 在 **TC (Traffic Control)** 层拦截网络数据包
- 使用 **eBPF Map** 存储 Service 到 EP 的映射
- 支持 **Maglev一致性哈希**算法保证后端变化时连接影响最小

### 启用 kube-proxy Replacement

在 Cilium 中启用 kube-proxy 替换：

```yaml
# cilium-config ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
  namespace: kube-system
data:
  kube-proxy-replacement: "strict"      # 完全替换 kube-proxy
  enable-host-reachable-services: "true"
  bpf-lb-sock: "true"                   # 使用 socket 级别的负载均衡
  bpf-lb-acceleration: "native"         # 原生加速
```

### 性能对比

Cilium 的基准测试数据（相对传统 kube-proxy）：

| 指标 | kube-proxy (iptables) | kube-proxy (IPVS) | Cilium eBPF |
|------|----------------------|-------------------|-------------|
| **吞吐量** | 基准 (1x) | ~2x | ~4-8x |
| **延迟 (P99)** | 基准 | ~0.5x | ~0.1-0.2x |
| **CPU 开销** | 基准 | ~0.7x | ~0.2x |
| **规则更新速度** | 秒级 | 秒级 | 微秒级 |
| **连接建立速率** | 基准 | ~3x | ~10x |

启用后，kube-proxy 可以完全移除。Service 的创建、更新、删除会通过 Cilium Agent 转换为 eBPF Map 的条目更新。

---

## 4.6 iptables模式下的连接追踪 (conntrack) 问题与优化

### conntrack 简介

**conntrack**（Connection Tracking）是 Linux 内核的连接追踪子系统。它记录系统中每个网络连接的元数据（源IP、源端口、目标IP、目标端口、协议、状态），用于：

- NAT 的正确工作（记录转换关系）
- 有状态防火墙规则
- iptables 的 `-m state` 和 `-m conntrack` 匹配

对于 Kubernetes 集群来说，conntrack 表特别重要——每个通过 Service 的 TCP/UDP 连接都在 conntrack 表中占用一个条目。

### conntrack 表在 Kubernetes 中的压力

在 iptables 模式下，Kubernetes 的高度动态特性给 conntrack 带来了巨大压力：

1. **海量短连接**：微服务架构中 HTTP/gRPC 调用频繁，每次连接都占用 conntrack 条目
2. **连接频率高**：每秒数万新连接的大型集群
3. **条目生命周期长**：TCP TIME_WAIT 默认 120 秒，CLOSE_WAIT 默认 60 秒（远长于实际连接）
4. **SNAT 导致额外条目**：externalTrafficPolicy: Cluster 时，每个外部连接都产生一个 SNAT 条目

### conntrack 表满的症状

```shell
# 内核日志
$ dmesg | grep conntrack
[12345.678] nf_conntrack: nf_conntrack: table full, dropping packet
[12346.789] nf_conntrack: nf_conntrack: table full, dropping packet

# 或查看当前使用量
$ cat /proc/sys/net/netfilter/nf_conntrack_count
524288      # 当前条目数

$ cat /proc/sys/net/netfilter/nf_conntrack_max
524288      # 最大条目数（已满！）
```

当 conntrack 表满时，新的连接无法被追踪，内核会丢弃数据包（SYN 包）。表现为：
- 客户端连接超时
- 间歇性连接失败
- 名为"随机掉线"的诡异行为

### conntrack 表的配置优化

```bash
# conntrack 表的最大条目数
sysctl -w net.netfilter.nf_conntrack_max=1048576

# 持久化配置
cat >> /etc/sysctl.d/99-kubernetes.conf <<EOF
net.netfilter.nf_conntrack_max = 1048576
EOF
sysctl -p /etc/sysctl.d/99-kubernetes.conf
```

### conntrack 超时时间优化

```bash
# 查看当前超时配置
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_time_wait

# 调优超时参数
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400    # 24小时
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30         # 30秒
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=3600      # 1小时
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_fin_wait=30          # 30秒
```

### conntrack 哈希表大小优化

```bash
# 哈希表大小（bucket 数量）
sysctl -w net.netfilter.nf_conntrack_buckets=262144

# 合理设置：buckets ≈ conntrack_max / 4
# 如果 conntrack_max = 1048576, buckets 应为 262144
```

### 监控 conntrack

```shell
# 查看当前使用量和最大容量
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 使用 conntrack 工具查看统计
conntrack -C              # 当前连接总数
conntrack -S              # 详细统计信息

# 查看特定 Service 的连接
conntrack -L -d 10.96.100.42

# Prometheus 监控指标
# node_nf_conntrack_entries        # 当前条目数
# node_nf_conntrack_entries_limit  # 最大条目数
```

### 缓解 conntrack 压力的架构方案

1. **切换到 IPVS 模式**：IPVS 维护自己的连接表，减少了对 conntrack 的依赖
2. **使用 eBPF/Cilium**：Cilium 使用自己的连接追踪实现（基于 eBPF Map），不依赖传统 conntrack
3. **使用 EndpointSlice**：减少 Endpoints 对象的变更频率，降低 iptables/IPVS 规则同步频率
4. **避免不必要的 SNAT**：使用 `externalTrafficPolicy: Local`（保留源 IP，减少 SNAT 条目）
5. **应用层优化**：使用 HTTP Keep-Alive 复用连接，使用 gRPC 的长连接模型，引入连接池

# 5.Endpoints与EndpointSlice

## 5.1 Endpoints对象 (传统: 所有后端Pod IP集合)

### Endpoints 对象的结构

Endpoints 是 Kubernetes 的一个原生资源对象，它存储了 Service 对应的所有后端 Pod 的网络信息（IP + Port）。每个 Service 会自动创建一个同名的 Endpoints 对象。

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service                 # 必须与 Service 同名
  namespace: default
subsets:
- addresses:                       # Ready 的 Pod 地址
  - ip: 10.244.1.5
    nodeName: node-1               # Pod 所在的 Node
    targetRef:                     # 指向对应 Pod 的引用
      kind: Pod
      name: my-app-7d4f8c9b6-abc12
      namespace: default
  - ip: 10.244.2.8
    nodeName: node-2
    targetRef:
      kind: Pod
      name: my-app-7d4f8c9b6-def34
      namespace: default
  ports:
  - name: http
    port: 8080                     # Pod 的 targetPort
    protocol: TCP
  notReadyAddresses:               # Not Ready 的 Pod 地址
  - ip: 10.244.3.12
    nodeName: node-3
    targetRef:
      kind: Pod
      name: my-app-7d4f8c9b6-ghi56
      namespace: default
```

### Endpoints Controller 的工作机制

Endpoints Controller 是 `kube-controller-manager` 中的一个组件，它负责根据 Service 的 Selector 自动管理 Endpoints 对象：

```
Endpoints Controller 的控制循环:
for each Service:
    1. 获取 Service 的 selector (如 app=my-app)
    2. 列出所有匹配 selector 的 Pod (全集群范围的 Pod)
    3. 对每个匹配的 Pod:
       - 检查 Pod 是否 Running
       - 检查 Pod 的 Readiness Gate 是否全部通过
       - 检查 Pod 是否正在被删除 (DeletionTimestamp 不为空)
       - 提取 Pod 的 IP 和 containerPort
    4. 将 Ready 的 Pod 放入 addresses 列表
    5. 将 Not Ready 的 Pod 放入 notReadyAddresses 列表
    6. 更新 Endpoints 对象
```

### Endpoints 的性能问题

在大型集群中，Endpoints 对象存在明显的性能问题：

1. **单一对象过大**：一个 Service 的所有后端 Pod 信息都存储在一个 Endpoints 对象中。如果有 5000 个 Pod，这个 Endpoints 对象就包含 5000 条地址记录。任何 Pod 的变更都需要完整更新这个巨型对象。

2. **频繁的全量更新**：每次后端 Pod 集合变化（Pod 创建/删除/就绪状态变化），Endpoints Controller 都需要完整地重新计算和更新整个 Endpoints 对象。

3. **Watch 事件过大**：API Server 将 Endpoints 对象的变更通过 Watch 接口推送给所有 watcher（如 kube-proxy）。每个 Node 上的 kube-proxy 都会收到完整的 Endpoints 对象，即使其中只有少量变化。

4. **etcd 写入放大**：Endpoints 对象的每次更新都会触发一次完整的 etcd 事务。大对象的序列化和反序列化开销巨大。

典型表现：
- 一个 5000 Pod 的 Service，Endpoints 对象大小约 1MB
- 每次滚动更新，每秒可能有数十次 Endpoints 更新
- kube-proxy 需要解析 1MB 的对象来确定只有 1 个 IP 发生了变更

### 注意事项

对于有 Selector 的 Service，Endpoints 由 Endpoints Controller 自动管理。手动修改 Endpoints 会立即被控制器回写覆盖。只有无 Selector 的 Service 才适合手动管理 Endpoints。

---

## 5.2 EndpointSlice (v1.21 GA: 按Pod分组切片, 减少大Service的API对象大小)

### EndpointSlice 的诞生

为了解决 Endpoints 对象的性能瓶颈，Kubernetes 社区设计了 **EndpointSlice**（端点切片）。它在 v1.17 引入（Alpha），v1.21 达到 GA（默认启用）。

核心设计思想：**将一个 Service 的后端 Pod 按组切分为多个 EndpointSlice 对象，每个 Slice 最多包含指定数量的端点**。

```
传统 Endpoints:
+--------------------------------------+
|              Endpoints               |
|  addresses:                          |
|    - ip: 10.244.1.5                  |
|    - ip: 10.244.1.6                  |
|    ... (5000 条记录)                  |
|    - ip: 10.244.100.8                |
+--------------------------------------+
单个对象包含所有 5000 个 Pod IP

EndpointSlice:
+-----------------+ +-----------------+ +-----------------+
| EndpointSlice 1 | | EndpointSlice 2 | | EndpointSlice 3 |
|                 | |                 | |                 |
| endpoints:      | | endpoints:      | | endpoints:      |
| - 10.244.1.5    | | - 10.244.2.7    | | - 10.244.3.9    |
| - 10.244.1.6    | | - 10.244.2.8    | | - 10.244.3.10   |
| ... (最多100条)  | | ... (最多100条)  | | ... (最多100条)  |
+-----------------+ +-----------------+ +-----------------+
50 个 Slice 对象 (5000/100)，每个最多 100 个端点
```

### EndpointSlice 的结构

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abc12              # 自动生成名称
  namespace: default
  labels:
    kubernetes.io/service-name: my-service   # 关联的 Service
  ownerReferences:                           # 由 Service 拥有
  - apiVersion: v1
    kind: Service
    name: my-service
addressType: IPv4                       # IPv4 | IPv6 | FQDN
endpoints:
- addresses:
  - 10.244.1.5
  conditions:
    ready: true                        # Pod 是否就绪
    serving: true                      # Pod 是否正在服务（v1.20+）
    terminating: false                 # Pod 是否正在终止（v1.22+）
  nodeName: node-1                     # Pod 所在 Node（用于拓扑感知）
  zone: us-east-1a                     # 可用区（用于拓扑感知）
  hostname: my-app-7d4f8c9b6-abc12     # Pod 主机名
  targetRef:                           # 指向对应 Pod 的引用
    kind: Pod
    name: my-app-7d4f8c9b6-abc12
    namespace: default
- addresses:
  - 10.244.2.8
  conditions:
    ready: true
  nodeName: node-2
  zone: us-east-1b
  targetRef:
    kind: Pod
    name: my-app-7d4f8c9b6-def34
    namespace: default
ports:
- name: http
  port: 8080
  protocol: TCP
```

### EndpointSlice 的关键优势

1. **拓扑信息前置**：每个端点包含 `nodeName` 和 `zone` 信息，kube-proxy 可以快速实现拓扑感知路由，无需额外查询 Node/Pod API

2. **增量更新**：只有发生变化的 Slice 需要更新，减少 Watch 事件的大小

3. **并行处理**：多个 Slice 可以被不同的控制器 goroutine 并行处理

4. **易于扩展**：新增端点到已有 Slice 或创建新 Slice 的开销都很小

5. **减少内存占用**：kube-proxy 只缓存相关 Slice，不需要在内存中维护整个巨型 Endpoints 对象

### 每个 Slice 的容量

默认每个 EndpointSlice 最多包含 **100 个端点**。这个数量可通过 `kube-controller-manager` 的 `--max-endpoints-per-slice` 参数配置：

```bash
kube-controller-manager --max-endpoints-per-slice=100
```

100 是默认值，也是推荐值。过大的 Slice 会失去切片的优势，过小则增加 Slice 数量。

---

## 5.3 EndpointSlice控制器工作原理

### 架构概述

EndpointSlice 控制器（EndpointSlice Controller）是 `kube-controller-manager` 中的一个控制器，替代了传统的 Endpoints 控制器。在 Kubernetes v1.21+ 中默认启用。

它的核心职责：
1. 监听 Service 的创建/更新/删除
2. 监听 Pod 的创建/更新/删除（通过 Pod 变化的 Selector 匹配）
3. 计算每个 Service 应该拥有的 EndpointSlice 集合
4. 创建、更新、删除 EndpointSlice 对象以匹配期望状态

### 控制器工作流程

```
EndpointSlice Controller 控制循环:
for each Service (event-driven):
    1. 获取 Service 的 selector
    2. 列出所有匹配 selector 的 Pod
    3. 对 Pod 按某种策略分组（默认按哈希分配到不同 Slice）
    4. 为每组分片创建一个 EndpointSlice 对象
    5. 对比当前已有的 EndpointSlice 与期望状态
    6. 创建/更新/删除 EndpointSlice 以匹配期望状态
```

### 分片策略

EndpointSlice 控制器使用以下策略将 Pod 分配到不同的 Slice：

1. **默认分片（哈希分组）**：将 Pod 的 IP 地址进行哈希，然后取模分配到不同 Slice。这样可以确保：
   - Pod 集合变化时，只有少量 Slice 需要更新
   - 每个 Slice 大小相对均匀
   - 同一 Node/Zone 的 Pod 可能分布在多个 Slice 中

2. **Node 分组（可选）**：将同一 Node 上的 Pod 分组到同一个 Slice。适合拓扑感知负载均衡场景。

3. **Zone 分组（可选）**：将同一可用区（Zone）的 Pod 分组到同一个 Slice。

### 边界条件处理

**Pod 列表过大**：当 Service 有超过 `max-endpoints-per-slice` 个 Pod 时，控制器自动创建多个 Slice。

**Pod 正在终止**：设置了 `deletionTimestamp` 的 Pod 被标记为 `terminating: true`，但仍在 Slice 中。kube-proxy 可以提前准备摘除流量，实现优雅终止。

**就绪状态变化**：Pod 从 Ready 变为 Not Ready 时，只需要更新对应 Slice 中该端点的 `ready` 字段，而不是重建整个 Slice。

**Service Selector 变化**：如果 Service 的 Selector 被修改（如从 `app=v1` 改为 `app=v2`），旧的后端 Pod 不再匹配。控制器会识别这种变化，重建所有 Slice。

### 性能对比

使用 EndpointSlice 替代 Endpoints 后，在大规模 Service 场景下的改善：

| 场景 (5000 Pod) | Endpoints | EndpointSlice (100/slice) |
|-----------------|-----------|--------------------------|
| 对象大小 | ~1 MB | ~200 KB (单个Slice) |
| 更新事件大小 | ~1 MB | ~200 KB |
| etcd 写入量 | 1 MB/次 | 200 KB/次 (只有变化的Slice更新) |
| Watch 传输数据 | 1 MB/次 | 200 KB/次 |
| 内存占用 (kube-proxy) | ~1 MB/Service | ~200 KB x 活跃Slice数 |

---

## 5.4 手动管理Endpoints (自定义Service后端)

### 无 Selector 的 Service

当 Service 的 `spec.selector` 字段为空时，Kubernetes 不会自动创建 Endpoints 或 EndpointSlice。用户需要手动创建同名的 Endpoints（或 EndpointSlice）对象来定义后端。

```yaml
# 无 Selector 的 Service
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  ports:
  - name: https
    port: 443
    targetPort: 443
  - name: http
    port: 80
    targetPort: 80
---
# 手动创建 Endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: external-api                 # 名字必须与 Service 完全相同
subsets:
- addresses:
  - ip: 192.168.1.100               # 任意 IP 地址
  - ip: 192.168.1.101
  ports:
  - name: https
    port: 443
  - name: http
    port: 80
```

### 使用场景

1. **将外部服务引入集群内 DNS**：外部数据库、消息队列、第三方 API 可以通过 Service DNS 名称访问

```yaml
# 外部 Redis 集群
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
spec:
  ports:
  - port: 6379
---
apiVersion: v1
kind: Endpoints
metadata:
  name: redis-cluster
subsets:
- addresses:
  - ip: 10.0.1.100                  # Redis 节点1
  - ip: 10.0.1.101                  # Redis 节点2
  - ip: 10.0.1.102                  # Redis 节点3
  ports:
  - port: 6379
```

2. **跨集群服务引用**：在多集群环境中，集群 A 的 Service 通过 Endpoints 指向集群 B 中的 Pod

3. **自定义负载均衡**：将 Service 后端指向非 Pod 的端点（如虚拟机上的服务）

4. **迁移过渡方案**：在将服务从虚拟机迁移到 Kubernetes 期间，Service 的后端混合包含 Pod 和 VM

### 手动 Endpoints vs ExternalName

| 特性 | 手动 Endpoints | ExternalName |
|------|---------------|--------------|
| **机制** | IP + Port 转发 | DNS CNAME |
| **负载均衡** | 支持（kube-proxy 对多 IP 负载均衡） | 不支持 |
| **端口映射** | 支持 | 不支持 |
| **健康检查** | 不支持（无 Endpoints Controller） | 不支持 |
| **适用场景** | 外部服务有多 IP，需要负载均衡 | 单一外部域名别名 |

### 使用 EndpointSlice 进行手动管理

从 v1.21 开始，推荐使用 EndpointSlice 而不是 Endpoints 进行手动管理：

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-api-1              # 可以是任意名称
  labels:
    kubernetes.io/service-name: external-api   # 关联的 Service
addressType: IPv4
endpoints:
- addresses:
  - 192.168.1.100
  conditions:
    ready: true                     # 手动标记为就绪
- addresses:
  - 192.168.1.101
  conditions:
    ready: true
ports:
- name: https
  port: 443
  protocol: TCP
- name: http
  port: 80
  protocol: TCP
```

### 自动化外部端点管理

对于需要动态管理外部端点的高级场景，可以编写自定义控制器（Operator）来自动维护 Endpoints/EndpointSlice：

```
自定义控制器:
  1. 定期查询外部服务的健康端点或服务发现系统（如 Consul）
  2. 对比当前 Endpoints 中的 IP 列表
  3. 添加新的健康 IP
  4. 移除不健康的 IP
  5. 更新 Endpoints 对象
```

这种方式将 Service 变成了通用的服务发现抽象层，无论 Pod 还是外部服务都用统一的 Service 接口暴露。

---

# 6.服务发现

## 6.1 环境变量注入 (启动时注入Service信息，不推荐)

### 环境变量注入机制

当 Pod 启动时，kubelet 会在 Pod 的容器中注入一组环境变量，这些环境变量包含了当前 Namespace 中**已经存在**的 Service 的连接信息：

```shell
# 假设存在一个名为 "redis" 的 Service，监听端口 6379

# 注入到 Pod 的环境变量
REDIS_SERVICE_HOST=10.96.100.42        # Service ClusterIP
REDIS_SERVICE_PORT=6379                # Service Port
REDIS_SERVICE_PORT_TCP=6379            # 协议+端口
REDIS_PORT=tcp://10.96.100.42:6379
REDIS_PORT_6379_TCP=tcp://10.96.100.42:6379
REDIS_PORT_6379_TCP_PROTO=tcp
REDIS_PORT_6379_TCP_PORT=6379
REDIS_PORT_6379_TCP_ADDR=10.96.100.42
```

### 环境变量注入的严重问题

**问题1：时序依赖**

环境变量**只在容器启动时注入一次**。如果 Pod A 启动时 Service B 还未创建，那么 Pod A 中不会有 Service B 的环境变量。即使后来 Service B 创建了，Pod A 也不会感知到。

```
时间线:
  T1: 创建 Pod A (应用容器启动)
  T2: 创建 Service B
  T3: Pod A 尝试连接 Service B -> 找不到环境变量 -> 连接失败
```

**问题2：Service 变更不更新**

如果 Service 被修改（如端口变更、ClusterIP 重建），已经运行的 Pod 中的环境变量不会更新。

**问题3：命名空间污染**

每个 Service 生成多达 7-8 个环境变量。在拥有数百个 Service 的 Namespace 中，每个 Pod 会注入上千个环境变量。

**问题4：安全风险**

环境变量可以被同一 Pod 内的任何容器读取。

### 强烈不推荐使用

环境变量注入在 Kubernetes 社区已被**强烈不推荐**用于生产环境。现代 Kubernetes 应用应使用 **DNS 服务发现**。

### 禁用环境变量注入

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  enableServiceLinks: false             # 禁用 Service 环境变量注入
  containers:
  - name: app
    image: my-app:latest
```

---

## 6.2 DNS服务发现 (CoreDNS, FQDN格式, SRV记录)

### DNS 服务发现的工作原理

Kubernetes 集群内置了 DNS 服务（默认使用 CoreDNS），为每个 Service 自动创建 DNS 记录。Pod 通过 DNS 解析来发现和连接到 Service。

DNS 服务发现的优势：
- **动态更新**：Service 创建/删除时，DNS 记录实时更新
- **无需重启**：Service 变更后 Pod 无需重启即可解析到新地址
- **跨命名空间**：统一的 FQDN 格式支持跨 Namespace 访问
- **丰富记录类型**：支持 A、AAAA、SRV、PTR 等多种记录类型

### FQDN 格式

Kubernetes 为 Service 自动创建的 FQDN 格式：

```
<service-name>.<namespace>.svc.<cluster-domain>

例如:
  redis.default.svc.cluster.local
  nginx.production.svc.cluster.local
  order-service.staging.svc.cluster.local
```

DNS 解析层级：

```
redis.default.svc.cluster.local
|      |       |   |       |
|      |       |   |       +-- 集群域名 (默认 cluster.local)
|      |       |   +---------- "svc" 固定后缀 (表示 Service)
|      |       +-------------- Namespace 名称
|      +---------------------- Service 名称
+----------------------------- 根域名
```

### DNS 解析的短名称

在 Pod 内部，DNS 解析器配置了搜索域（search domain），使得可以使用短名称来访问 Service：

```
Pod 的 /etc/resolv.conf:
nameserver 10.96.0.10                       # CoreDNS Service ClusterIP
search default.svc.cluster.local \           # 搜索域（多个）
       svc.cluster.local \
       cluster.local
options ndots:5                              # 域名中点的数量阈值
```

基于这些搜索域，以下短名称都可以解析：

| 使用名称 | 实际解析 |
|---------|---------|
| `redis` | `redis.default.svc.cluster.local` |
| `redis.default` | `redis.default.svc.cluster.local` |
| `redis.default.svc` | `redis.default.svc.cluster.local` |
| `redis.default.svc.cluster.local` | (完整 FQDN) |
| `redis.production` | `redis.production.svc.cluster.local` |

### DNS 记录类型

**A 记录（IPv4 地址）**：

```shell
$ nslookup my-service.default.svc.cluster.local
Name:    my-service.default.svc.cluster.local
Address: 10.96.100.42
```

Headless Service 返回多条 A 记录，指向后端 Pod IP。

**SRV 记录（服务端口发现）**：

SRV 记录用于服务发现的命名端口。格式为 `_port-name._protocol.service-name.namespace.svc.cluster.local`：

```shell
$ nslookup -type=SRV _http._tcp.my-service.default.svc.cluster.local
_http._tcp.my-service.default.svc.cluster.local  service = 0 100 80 my-service.default.svc.cluster.local
```

SRV 记录的字段含义：`优先级 权重 端口 目标`

### Headless Service 的 DNS 特殊行为

Headless Service（`clusterIP: None`）的 DNS 行为与普通 Service 不同：
- 返回所有 Ready Pod 的 IP（A/AAAA 记录列表）
- 配合 StatefulSet 时，每个 Pod 有独立的 DNS 名称

### ndots 配置与 DNS 查询优化

`ndots:5` 的含义：如果域名中的点（.）数量少于 5 个，则首先使用搜索域进行查询。

```shell
查询 redis:
  redis 有 0 个点 (< 5) -> 先尝试搜索域
  查询1: redis.default.svc.cluster.local.
  查询2: redis.svc.cluster.local.
  查询3: redis.cluster.local.
  查询4: redis. (绝对查询)

查询 google.com:
  google.com 有 1 个点 (< 5) -> 先尝试搜索域
  查询1: google.com.default.svc.cluster.local.  <- 不必要的查询！
  查询2: google.com.svc.cluster.local.           <- 不必要的查询！
  ...

查询 www.google.com:
  www.google.com 有 2 个点 (< 5) -> 先尝试搜索域
  (3次不必要的查询)
```

可以通过自定义 `dnsConfig` 调整 ndots：

```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"                          # 减少搜索域查询
```

---

## 6.3 CoreDNS架构与配置 (Corefile, 插件链)

### CoreDNS 简介

CoreDNS 是 Kubernetes 集群的默认 DNS 服务器（从 v1.13 开始替代 kube-dns）。它是一个用 Go 编写的、基于插件架构的灵活 DNS 服务器，以 Deployment 方式在集群中运行。

CoreDNS 的核心特点：
- **插件架构**：所有功能通过插件实现，链式处理 DNS 请求
- **高性能**：Go 协程模型，处理大量并发 DNS 查询
- **Kubernetes 集成**：内置 kubernetes 插件，直接通过 API Server 获取 Service/Pod 信息
- **可扩展**：支持自定义插件和转发策略

### CoreDNS 在 Kubernetes 中的部署

```yaml
# CoreDNS Deployment (简化)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  replicas: 2                              # 通常 2 个副本（高可用）
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    spec:
      serviceAccountName: coredns
      containers:
      - name: coredns
        image: registry.k8s.io/coredns/coredns:v1.10.1
        args: [ "-conf", "/etc/coredns/Corefile" ]
        ports:
        - containerPort: 53
          protocol: UDP
        - containerPort: 53
          protocol: TCP
        - containerPort: 9153                 # Metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
```

通过 Service 暴露 DNS：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
spec:
  clusterIP: 10.96.0.10                     # 固定的 DNS Service IP
  ports:
  - name: dns
    port: 53
    targetPort: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    targetPort: 53
    protocol: TCP
  selector:
    k8s-app: kube-dns
```

### Corefile 配置

Corefile 是 CoreDNS 的核心配置文件，定义了插件链。一个典型的 Kubernetes Corefile：

```hcl
.:53 {
    errors                  # 错误日志
    health {                # 健康检查端点
        lameduck 5s
    }
    ready                   # 就绪检查端点
    kubernetes cluster.local in-addr.arpa ip6.arpa {   # Kubernetes 插件
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153        # Prometheus 指标
    forward . /etc/resolv.conf {    # 向上游 DNS 转发
        max_concurrent 1000
    }
    cache 30                # 缓存配置
    loop                    # 环路检测
    reload                  # 自动重载配置
    loadbalance             # 上游服务器负载均衡
}
```

### 插件链详解

CoreDNS 的每个 DNS 请求按配置顺序经过每个插件：

```
DNS 请求 -> errors -> health -> kubernetes 插件 -> prometheus -> forward -> cache -> loop
```

关键插件说明：

1. **kubernetes 插件**：核心插件，处理 `cluster.local` 域名的解析。通过 API Server 查询 Service 和 Pod 信息。

2. **forward 插件**：将不匹配的 DNS 查询转发到上游 DNS 服务器。通常转发到宿主机的 `/etc/resolv.conf` 中配置的 DNS 服务器。

3. **cache 插件**：缓存 DNS 查询结果，默认 TTL 30 秒。减少对 API Server 和上游 DNS 的压力。

4. **loadbalance 插件**：对上游 DNS 返回的多条 A 记录进行轮询负载均衡。

### CoreDNS 的副本数和高可用

```yaml
# CoreDNS Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: coredns
  namespace: kube-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coredns
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

配合 Pod Anti-Affinity 确保 CoreDNS Pod 分散在不同节点：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  replicas: 2
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                k8s-app: kube-dns
            topologyKey: kubernetes.io/hostname
```

### 监控 CoreDNS

```shell
# CoreDNS 暴露的指标
curl http://coredns-service:9153/metrics

# 关键指标
coredns_dns_requests_total           # DNS 请求总数
coredns_dns_responses_total          # DNS 响应总数（按 rcode 分类）
coredns_dns_request_duration_seconds # DNS 请求延迟
coredns_kubernetes_dns_programming_duration_seconds # DNS 编程延迟
coredns_cache_hits_total             # 缓存命中数
coredns_cache_misses_total           # 缓存未命中数
```

---

## 6.4 DNS解析策略 (dnsPolicy: ClusterFirst/Default/None)

Kubernetes 为 Pod 提供了三种 DNS 策略，通过 `spec.dnsPolicy` 字段配置：

### ClusterFirst（默认策略）

所有使用 `cluster.local` 后缀的 DNS 查询都先发送到集群 DNS（CoreDNS）。非集群域名的查询根据 Pod 的 `dnsConfig` 或上游设置决定。

```yaml
spec:
  dnsPolicy: ClusterFirst
```

此策略下 `/etc/resolv.conf` 的内容大致为：

```
nameserver 10.96.0.10           # CoreDNS Service IP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### Default

继承 Pod 所在 Node 的 DNS 解析配置（即宿主机的 `/etc/resolv.conf`）。集群内 DNS 服务发现将不可用。

```yaml
spec:
  dnsPolicy: Default
```

适用场景：
- Pod 不需要访问集群内 Service
- 需要使用宿主机特定的 DNS 配置（如企业内网 DNS）

### None

不使用 Kubernetes 的默认 DNS 配置。必须配合 `spec.dnsConfig` 显式指定 DNS 设置。

```yaml
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
    - 1.2.3.4                     # 自定义 DNS 服务器
    - 5.6.7.8
    searches:
    - my-namespace.svc.cluster.local
    - svc.cluster.local
    options:
    - name: ndots
      value: "2"
    - name: timeout
      value: "1"
```

适用场景：
- 需要使用非标准 DNS 配置
- Pod 需要特定的 DNS 解析行为
- 使用 DNS 加密（如 DNS-over-TLS）

### 三种策略对比

| 策略 | 集群DNS可用 | /etc/resolv.conf 来源 | 适用场景 |
|------|-----------|---------------------|---------|
| **ClusterFirst** | 是 | kubelet 自动生成 | 绝大多数 Pod，默认策略 |
| **Default** | 否 | 继承宿主机 | 不需要集群服务的特殊 Pod |
| **None** | 取决于 dnsConfig | 用户自定义 | 需要完全控制 DNS 配置 |

### 不需要集群 DNS 方案的 Pod

某些系统级 Pod（如网络插件、某些 DaemonSet）在集群 DNS 可用之前就启动，或者不需要访问集群内 Service。它们应使用 `dnsPolicy: Default` 配合 `hostNetwork: true`。

---

## 6.5 自定义DNS配置 (dnsConfig)

`spec.dnsConfig` 允许用户在 Pod 级别覆盖 DNS 配置的各个方面。这比整个集群级别的 CoreDNS 配置更精细。

### dnsConfig 字段

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None                     # 配合使用
  dnsConfig:
    nameservers:                      # DNS 服务器列表（最多 3 个）
    - 8.8.8.8
    - 1.1.1.1
    searches:                         # DNS 搜索域列表（最多 32 个域，每个最多 256 字符）
    - my-app.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    options:                          # DNS 解析器选项
    - name: ndots
      value: "2"                      # ndots 阈值
    - name: timeout
      value: "2"                      # DNS 查询超时（秒）
    - name: attempts
      value: "3"                      # DNS 查询重试次数
    - name: single-request-reopen     # 不使用同一 socket 发送查询（某些 glibc 版本需要）
    - name: edns0                     # 启用 EDNS0 扩展
  containers:
  - name: app
    image: nginx:1.21
```

### 实际生成 /etc/resolv.conf

kubelet 根据以上配置生成的 `/etc/resolv.conf`：

```
nameserver 8.8.8.8
nameserver 1.1.1.1
search my-app.svc.cluster.local svc.cluster.local cluster.local
options ndots:2 timeout:2 attempts:3 single-request-reopen edns0
```

### 使用场景

1. **减少外部域名查询延迟**：将 `ndots` 从 5 降到 2，减少不必要的搜索域查询

```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"
```

2. **Service Mesh 场景**：Istio/Envoy sidecar 需要拦截 DNS 请求做服务路由

3. **使用私有 DNS**：企业内网应用需要使用公司内部 DNS 服务器解析内部域名

4. **DNS over HTTPS/TLS**：配置安全的 DNS 解析通道

### 与 dnsPolicy 的配合

| dnsPolicy | dnsConfig 作用 |
|-----------|---------------|
| `ClusterFirst` | dnsConfig 可以**追加**搜索域和选项，但 nameservers 列表会**替换** |
| `Default` | dnsConfig 完全覆盖，只使用 dnsConfig 中的内容 |
| `None` | 必须使用 dnsConfig，完全自定义 |

### Pod DNS 配置的排查

```shell
# 进入 Pod 查看 DNS 配置
kubectl exec -it my-pod -- cat /etc/resolv.conf

# 测试 DNS 解析
kubectl exec -it my-pod -- nslookup kubernetes.default
kubectl exec -it my-pod -- nslookup google.com

# 详细的 DNS 调试
kubectl exec -it my-pod -- nslookup -debug my-service.default.svc.cluster.local

# 查看 DNS 查询统计
kubectl exec -it coredns-xxx -n kube-system -- cat /proc/net/udp
```



# 7.无头服务 (Headless Service)

## 7.1 ClusterIP=None的含义

Headless Service 的核心标志是 `clusterIP: None`。当这个字段被设置为 `None` 时，Kubernetes 的 Service ClusterIP Allocator 不会为该 Service 分配虚拟 IP 地址。

### 与普通 Service 的本质区别

```
普通 Service (ClusterIP: 10.96.100.42):
  DNS 查询 -> 返回一个 A 记录: 10.96.100.42
  客户端 -> 连接到 ClusterIP -> kube-proxy 负载均衡 -> 后端 Pod
  对客户端透明：不知道后端有多少 Pod，也不知道 Pod 的 IP

Headless Service (ClusterIP: None):
  DNS 查询 -> 返回多个 A 记录: 10.244.1.5, 10.244.2.8, 10.244.3.12
  客户端 -> 直接连接到其中一个 Pod IP
  客户端可见：知道所有后端 Pod 的 IP，需要自己决定连接哪个
```

### 底层原理

从 kube-proxy 的视角看：
- `clusterIP: None` -> kube-proxy **不会**为该 Service 创建 iptables/IPVS 规则
- 没有任何 DNAT 规则截获数据包
- 客户端与 Pod 之间是直接通信，无代理

从 CoreDNS 的视角看：
- `clusterIP: None` -> DNS 查询返回**一组** Pod IP 地址（而非一个 ClusterIP）
- 如果配合 StatefulSet，还会为每个 Pod 创建独立的 DNS 名称

从 kube-apiserver 的视角看：
- `clusterIP: None` -> 不占用 Service ClusterIP CIDR 中的任何 IP
- Endpoints/EndpointSlice 仍然正常维护

### 创建 Headless Service 实例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None              # 核心：设为 None
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
```

---

## 7.2 DNS返回Pod IP列表 (非VIP)

### DNS 查询行为对比

```shell
# 普通 Service
$ nslookup nginx.default.svc.cluster.local
Name:      nginx.default.svc.cluster.local
Address:   10.96.100.100         <- 返回 ClusterIP（1条）

# Headless Service
$ nslookup nginx-headless.default.svc.cluster.local
Name:      nginx-headless.default.svc.cluster.local
Address:   10.244.1.5             <- Pod IP（多条）
Address:   10.244.2.8
Address:   10.244.3.12
```

DNS 记录是动态的：当 Pod 增加、删除或就绪状态变化时，DNS 记录实时更新。

### StatefulSet 的独立 DNS 名称

当 Headless Service 配合 StatefulSet 使用时，每个 Pod 还有独立的 DNS 名称（A 记录）：

```
<pod-name>.<service-name>.<namespace>.svc.<cluster-domain>

例如:
  web-0.nginx.default.svc.cluster.local
  web-1.nginx.default.svc.cluster.local
  web-2.nginx.default.svc.cluster.local
```

同时，`<service-name>.<namespace>.svc.<cluster-domain>` 返回所有 Pod 的 IP 列表（多条 A 记录）：

```shell
$ nslookup nginx.default.svc.cluster.local
Address: 10.244.1.5    # web-0 的 IP
Address: 10.244.2.8    # web-1 的 IP
Address: 10.244.3.12   # web-2 的 IP
```

---

## 7.3 适用场景 (StatefulSet/数据库集群/自定义负载均衡)

### 场景1：StatefulSet + Headless Service

最经典的组合。适用于需要稳定网络标识的系统：Kafka、ZooKeeper、etcd、Cassandra、MongoDB 副本集、Elasticsearch 集群。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-svc
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - port: 9092
    name: kafka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-svc        # 引用 Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        env:
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(HOSTNAME).kafka-svc.default.svc.cluster.local:9092"
        ports:
        - containerPort: 9092
```

每个 Kafka Broker 有独立的 DNS 名称用于节点间通信和客户端连接。

### 场景2：数据库集群的自发现

Cassandra、MongoDB 等数据库需要在启动时发现集群中的其他节点，通过 DNS 查询 Headless Service 获取所有成员的 IP。

### 场景3：自定义客户端负载均衡

当应用使用 gRPC 客户端负载均衡时：

```go
// gRPC 客户端示例
conn, err := grpc.Dial(
    "dns:///my-service.default.svc.cluster.local:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
    grpc.WithInsecure(),
)
```

gRPC 的 DNS resolver 解析 Headless Service，获取所有后端 Pod IP，然后在客户端侧进行轮询负载均衡。避免了 kube-proxy 的额外跳数和延迟。

### 场景4：需要发现所有成员的分布式系统

Apache Spark、Apache Flink 等大数据框架需要知道所有 Worker 节点的信息。通过 Headless Service，Master 节点可以解析到所有 Worker Pod 的 IP。

---

# 8.拓扑感知服务 (Topology Aware Service)

## 8.1 Service Topology (v1.17-1.21, 已被弃用)

### 历史背景

Service Topology 是 Kubernetes v1.17 引入的一个 Alpha 特性，旨在让 Service 能够根据网络拓扑（如节点、可用区、区域）将流量优先路由到就近的端点。由于配置复杂和行为不确定，在 v1.21 后被弃用。

---

## 8.2 Topology Aware Hints / Topology Aware Routing (v1.21+)

### 新设计理念

Topology Aware Hints（v1.21 Alpha -> v1.23 Beta -> v1.27 GA 更名为 Topology Aware Routing）是 Service Topology 的替代方案。它自动根据节点的拓扑标签（特别是 `topology.kubernetes.io/zone`）优化 EndpointSlice 中端点的分布。

### 工作原理

Topology Aware Routing 改进了 EndpointSlice 的分配方式：

```
传统分配:
  EndpointSlice 按 IP 哈希分组，不考虑拓扑位置

Topology Aware Routing:
  EndpointSlice 按 Zone（可用区）分组
  kube-proxy 优先选择本 Zone 的 Slice 中的端点
```

### 启用方式

Topology Aware Routing 在 v1.27+ 默认启用。通过 Service 的 annotation 控制：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.kubernetes.io/topology-mode: Auto   # 启用拓扑感知路由
spec:
  selector:
    app: my-app
  ports:
  - port: 80
```

### 触发条件

拓扑感知路由只在以下条件全部满足时生效：

1. `EndpointSlice` 控制器启用了 `TopologyAwareHints` 功能门
2. 节点被标记了拓扑标签（`topology.kubernetes.io/zone`）
3. 各 Zone 的端点数量**均匀分布**（每个 Zone 的端点比例偏差不超过特定阈值）
4. Service 设置了 annotation `service.kubernetes.io/topology-mode: Auto`

如果端点分布不均匀，系统会自动回退到标准路由模式。

### 均匀性检查

系统使用以下逻辑判断分布是否均匀：

```
# 给定阈值为 sl = 0.2（可配置）

总端点总数: N
将 N * (1 + sl) 向上取整 = M

如果一个 Zone 的端点数量 > M，则分布不均匀
系统回退到非拓扑感知路由
```

---

## 8.3 就近路由的实现原理

### kube-proxy 的 Node Zone 感知

kube-proxy 启动时会通过 Node 对象的标签获取当前 Node 所在的 Zone。当 kube-proxy 消费 EndpointSlice 时，会检查每个端点的 `zone` 字段：

```yaml
endpoints:
- addresses:
  - 10.244.1.5
  zone: us-east-1a          # <- 端点的 Zone
  conditions:
    ready: true
```

### 就近路由逻辑

kube-proxy 在选择后端 Pod 时的优先级逻辑：

```
1. 遍历本 Zone 的 EndpointSlice
2. 如果本 Zone 有 Ready 的端点 -> 仅对本 Zone 的端点负载均衡
3. 如果本 Zone 没有 Ready 的端点 -> 对所有 Zone 的端点负载均衡（自动降级）
```

### iptables 模式下的实现

在 iptables 模式下，kube-proxy 为每个 Service 生成两组规则链：

```
KUBE-SVC-XXXX:
  (优先) KUBE-SVL-YYYY (本 Zone 的 Local 端点) 
         -> KUBE-SEP-AAAA (本 Zone Pod A)
         -> KUBE-SEP-BBBB (本 Zone Pod B)
  (降级) KUBE-SVL-ZZZZ (所有 Zone 的端点)
         -> KUBE-SEP-CCCC (其他 Zone Pod C)
         -> KUBE-SEP-DDDD (其他 Zone Pod D)
```

### 端到端流量路径

```
Pod A (us-east-1a)
    | DNS查询: my-service.default.svc.cluster.local
    v
CoreDNS -> 返回 ClusterIP: 10.96.100.42
    | TCP 连接 10.96.100.42:80
    v
Pod A 所在 Node (us-east-1a)
    | kube-proxy: 本 Zone = us-east-1a
    | 检查 EndpointSlice: 本 Zone 有 Ready 端点
    | DNAT -> 本 Zone Pod IP (10.244.1.5:8080)
    v
Pod B (us-east-1a, 同一 Zone)
    | 响应直接返回（无跨 Zone 延迟）
    v
Pod A 收到响应
```

### 优势与限制

**优势**：减少跨可用区流量（降低延迟和成本）；自动降级；无需手动配置。

**限制**：只在 Zone 分布均匀时生效；跨 Zone 的流量计费问题需要云平台配合；不适合某些需要全局负载均衡的场景。

---

# 9.流量策略

## 9.1 externalTrafficPolicy详解 (Cluster: 集群内节点间转发, 源IP SNAT; Local: 仅本节点Pod, 保留源IP)

### externalTrafficPolicy: Cluster（默认值）

**行为**：外部流量到达任何 Node 的 NodePort 后，可以被转发到集群内**任意 Node** 上的后端 Pod。

**流量路径**：

```
外部客户端 (10.0.1.100)
    | 连接到 Node2:30080
    v
Node2 (10.0.0.2)
    | DNAT -> ClusterIP:80
    | 源 IP 被 SNAT (改写成 Node2 的 IP: 10.0.0.2)
    v
Node2 -> 跨节点路由 -> Node1
    | 后端 Pod (10.244.1.5:8080)
    v
Pod 收到请求，源 IP = 10.0.0.2 (Node2 的 IP)  <- 源 IP 丢失！
```

**核心问题**：由于 SNAT，后端 Pod 看到的源 IP 是 Node 的 IP，而不是客户端的真实 IP。

### externalTrafficPolicy: Local

**行为**：外部流量到达某个 Node 后，**只能**转发到该 Node 上的后端 Pod。如果接收请求的 Node 上没有后端 Pod，连接被拒绝。不会进行 SNAT，后端 Pod 可以直接看到客户端的真实 IP。

```
外部客户端 (10.0.1.100)
    | 连接到 Node1:30080
    v
Node1 (10.0.0.1)
    | 检查本节点是否有 Ready 的 Pod
    | 有: DNAT 直接到本节点 Pod IP:8080
    | 没有: 丢弃数据包
    v
后端 Pod (10.244.1.5:8080)
Pod 收到请求，源 IP = 10.0.1.100 (客户端真实 IP 保留)
```

**优势**：保留客户端源 IP；减少跨节点跳数。

**劣势**：负载均衡粒度较粗；需要外部 LB 层配置健康检查。

### 配置示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local        # 保留客户端源 IP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

---

## 9.2 internalTrafficPolicy (Cluster/Local)

`internalTrafficPolicy` 控制集群**内部**流量的路由行为（Pod 到 Service 的流量），在 v1.22 引入。

### internalTrafficPolicy: Cluster（默认值）

集群内部流量可以被转发到集群中**任何 Node** 上的后端 Pod。

### internalTrafficPolicy: Local

集群内部流量**只能**转发到与源 Pod **在同一 Node** 上的后端 Pod。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cache-service
spec:
  selector:
    app: redis
  ports:
  - port: 6379
  internalTrafficPolicy: Local       # 优先使用本节点 Pod
```

### 使用场景

1. **拓扑感知访问**：将流量限制在本节点，减少跨节点网络延迟
2. **本地缓存优化**：DaemonSet 部署的缓存代理（如 Envoy Sidecar）
3. **节点级故障隔离**：本节点后端不可用，即快速失败

### externalTrafficPolicy + internalTrafficPolicy 的组合

```yaml
spec:
  externalTrafficPolicy: Local
  internalTrafficPolicy: Local
```

---

## 9.3 保留客户端源IP的方案对比

### 方案1：externalTrafficPolicy: Local

工作原理：不自做 SNAT，直接将外部流量的源 IP 传递给本节点 Pod。

**优点**：原生支持，无需额外组件；性能高（无额外跳数）；配置简单。

**缺点**：Pod 必须分布在所有接收流量的 Node 上；外部 LB 需要配健康检查。

**适用**：云环境 LoadBalancer + 均匀分布的 Pod。

### 方案2：PROXY Protocol

在外部 LB 到 Node 的连接中插入 PROXY Protocol 头，携带客户端真实 IP。

```yaml
# AWS NLB + PROXY Protocol v2
apiVersion: v1
kind: Service
metadata:
  name: proxy-protocol-svc
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
spec:
  type: LoadBalancer
  ports:
  - port: 80
```

**优点**：源 IP 完美保留，不受 `externalTrafficPolicy` 限制。

**缺点**：需要应用支持 PROXY Protocol。

### 方案3：X-Forwarded-For (HTTP 层面)

通过 Ingress Controller 在 HTTP 头中添加 `X-Forwarded-For`。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-forwarded-headers: "true"
```

**优点**：HTTP 层解决方案，应用广泛。**缺点**：仅适用于 HTTP/HTTPS。

### 方案对比总结

| 方案 | 协议支持 | 配置复杂度 | 性能 | 源 IP 准确性 |
|------|---------|-----------|------|------------|
| externalTrafficPolicy: Local | TCP/UDP | 低 | 高 | 完全准确 |
| PROXY Protocol | TCP | 中 | 高 | 完全准确 |
| X-Forwarded-For | HTTP only | 低 | 高 | 依赖中间代理可信度 |

---

# 10.Service常见问题与排查

## 10.1 Service不可达的排查流程

### 排查步骤（自底向上）

**第1步：检查 Service 本身**

```shell
kubectl get svc <service-name> -n <namespace>
kubectl describe svc <service-name> -n <namespace>
# 关键检查: Type, Selector, Port/TargetPort, Endpoints, Session Affinity
```

**第2步：检查 Endpoints**

```shell
kubectl get endpoints <service-name> -n <namespace>
# 如果为空：检查 Selector 匹配 + Pod Ready 状态
```

**第3步：检查 Pod 选择器匹配**

```shell
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'
kubectl get pods -n <namespace> -l <selector> -o wide
```

**第4步：测试 Pod 之间直连**

```shell
kubectl exec -it <source-pod> -n <namespace> -- curl http://<pod-ip>:<target-port>
# 如果直连成功但通过 Service 失败，说明 kube-proxy 规则有问题
```

**第5步：检查 kube-proxy**

```shell
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=100

# 在 Node 上检查 iptables 规则
iptables -t nat -L KUBE-SERVICES -n | grep <cluster-ip>

# 在 Node 上检查 IPVS 规则
ipvsadm -L -n | grep <cluster-ip>
```

**第6步：检查 DNS 解析**

```shell
kubectl exec -it <test-pod> -- nslookup <service-name>.<namespace>.svc.cluster.local
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
```

**第7步：网络层面排查**

```shell
kubectl exec -it <test-pod> -- curl http://<service-name>.<namespace>:<port>
curl http://<cluster-ip>:<port>  # 从 Node 上测试
```

### 快速定位命令集合

```shell
SVC=my-service
NS=default
kubectl get svc $SVC -n $NS
kubectl get endpoints $SVC -n $NS
kubectl describe svc $SVC -n $NS | grep -A5 "Endpoints:"
```

---

## 10.2 Endpoints为空的原因

### 原因1：Selector 不匹配

```shell
$ kubectl get svc my-service -o jsonpath='{.spec.selector}'
{"app":"my-app","version":"v1"}

$ kubectl get pods --show-labels
NAME                    LABELS
my-app-xxx              app=my-app,version=v2  <- version 不匹配！
```

**解决方案**：确保 Pod 的 Labels 与 Service 的 Selector 完全匹配。

### 原因2：Pod 未就绪

```shell
$ kubectl get pods -l app=my-app
NAME                    READY   STATUS
my-app-xxx             0/1     Running       <- READY 0/1

$ kubectl describe pod my-app-xxx
Readiness probe failed: HTTP probe failed with statuscode: 500
```

**解决方案**：检查 Readiness Probe 配置（端口、路径、超时）；等待应用启动完成。

### 原因3：Pod 处于 Pending 状态

```shell
$ kubectl describe pod my-app-xxx
Events:
  Warning  FailedScheduling  0/3 nodes are available: insufficient cpu.
```

**解决方案**：解决资源不足、节点选择器、污点/容忍等问题。

### 原因4：Pod 正在终止

Pod 有 `deletionTimestamp` 时，其 IP 从 Endpoints 中移除。等待新 Pod 创建并就绪。

### 原因5：Service 类型为 ExternalName

ExternalName 类型 Service 没有 Selector，不会有 Endpoints。这是预期行为。

### 原因6：命名空间不匹配

Service 和 Pod 必须在同一个 Namespace 中。

### 原因7：publishNotReadyAddresses 未启用

对于 StatefulSet 的集群组建场景，如果 Pod 在启动时需要通过 Service 发现其他成员，需要设置 `publishNotReadyAddresses: true`。

```yaml
spec:
  publishNotReadyAddresses: true
```

---

## 10.3 kube-proxy规则同步延迟

### 延迟的组成

```
总延迟 = API 延迟 + 规则同步延迟 + 规则生效延迟

API 延迟: API Server 接收变更 + etcd 持久化 + Watch 推送 = 通常 < 500ms
规则同步延迟: syncProxyRules() 处理 + 生成规则 + 写入 = iptables 1-10秒 / IPVS < 1秒
规则生效延迟: 内核处理新规则 = 通常 < 100ms
```

### iptables 模式下的规则同步间隔

```bash
kube-proxy --iptables-sync-period=30s          # 全量同步间隔
kube-proxy --iptables-min-sync-period=1s       # 最小同步间隔
```

### 减少同步延迟的方法

1. **切换到 IPVS 模式**：增量更新代替全量重建
2. **调整同步间隔**（权衡 CPU 使用）：
   ```bash
   kube-proxy --iptables-sync-period=10s --iptables-min-sync-period=1s
   ```
3. **使用 EndpointSlice 替代 Endpoints**：减少 Watch 事件大小
4. **减少不必要的变更**：合理设置 Readiness Probe 的 `periodSeconds`；使用 PodDisruptionBudget

### 检测同步延迟

```shell
curl http://localhost:10249/metrics | grep kubeproxy

# 关键指标
kubeproxy_sync_proxy_rules_duration_seconds    # 规则同步耗时
kubeproxy_sync_proxy_rules_last_timestamp_seconds  # 上次同步时间戳
kubeproxy_network_programming_duration_seconds      # 网络编程延迟
```

### 应用层优雅处理

应用开发时应假设 Service 后端的变更存在秒级延迟：
1. **重试机制**：连接失败时自动重试
2. **连接池**：减少频繁建立新连接
3. **断路器**：在后端不稳定时自动熔断
4. **客户端负载均衡**：使用 gRPC 等支持客户端 LB 的方案
5. **Readiness Probe 与 preStop Hook**：配合 `terminationGracePeriodSeconds` 实现优雅摘流
