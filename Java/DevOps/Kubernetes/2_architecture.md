# 1.Kubernetes 整体架构

## 1.1 架构设计原则

Kubernetes 的架构建立在以下核心设计原则之上：

| 原则 | 说明 |
|------|------|
| **声明式优于命令式** | 用户描述期望状态（Desired State），系统负责驱动实际状态向期望状态收敛 |
| **控制循环（Control Loop）** | 每个组件都是独立的控制循环，持续观察 → 比较 → 调整 |
| **最终一致性** | 不追求强一致性瞬达，而是保证最终收敛到期望状态 |
| **无状态控制面** | 所有持久化状态存储在 etcd，控制面组件本身可重启、可替换 |
| **关注点分离** | 单个组件职责单一，组件间通过 API Server 松耦合 |
| **可插拔架构** | CRI、CNI、CSI 等标准接口允许替换和扩展底层实现 |
| **面向 API 的设计** | 一切皆资源，统一通过 REST API 管理 |

## 1.2 宏观架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Control Plane（控制平面）                       │
│                                                                   │
│  ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌───────────┐  │
│  │   etcd   │  │  API Server  │  │  Scheduler │  │Controller │  │
│  │          │  │              │  │            │  │  Manager  │  │
│  │ 分布式KV │◄─┤  REST API   │◄─┤ Pod 调度   │  │           │  │
│  │  存储    │  │ 认证授权准入  │  │ 过滤+打分   │  │多控制器聚合│  │
│  └──────────┘  └──────────────┘  └────────────┘  └───────────┘  │
│                     │                                             │
└─────────────────────┼─────────────────────────────────────────────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
┌─────────▼───┐ ┌─────▼─────┐ ┌──▼───────────┐
│  Worker Node│ │Worker Node│ │ Worker Node   │
│             │ │           │ │               │
│ ┌─────────┐ │ │┌─────────┐│ │┌─────────────┐│
│ │ kubelet │ │ ││ kubelet ││ ││  kubelet    ││
│ │         │ │ ││         ││ ││ (节点主管)    ││
│ └────┬────┘ │ │└────┬────┘│ │└──────┬──────┘│
│ ┌────▼────┐ │ │┌────▼────┐│ │┌──────▼──────┐│
│ │Container│ │ ││Container││ ││ Container   ││
│ │Runtime  │ │ ││Runtime  ││ ││ Runtime     ││
│ │(contain │ │ ││(contain ││ ││(containerd) ││
│ │ erd)    │ │ ││  erd)   ││ ││             ││
│ └────┬────┘ │ │└────┬────┘│ │└──────┬──────┘│
│ ┌────▼────┐ │ │┌────▼────┐│ │┌──────▼──────┐│
│ │  Pods   │ │ ││  Pods   ││ ││    Pods     ││
│ │┌──┐┌──┐ │ ││┌──┐┌──┐ ││ ││┌────┐┌────┐ ││
│ ││C1││C2│ │ │││C1││C2│ ││ │││ C1 ││ C2 │ ││
│ │└──┘└──┘ │ ││└──┘└──┘ ││ ││└────┘└────┘ ││
│ └─────────┘ │ │└─────────┘│ │└─────────────┘│
│ ┌─────────┐ │ │┌─────────┐│ │┌─────────────┐│
│ │kube-proxy│ ││kube-proxy││ ││ kube-proxy  ││
│ └─────────┘ │ │└─────────┘│ │└─────────────┘│
└─────────────┘ └───────────┘ └───────────────┘
```

## 1.3 控制平面组件深度解析

### 1.3.1 kube-apiserver —— 集群统一入口

kube-apiserver 是 Kubernetes 控制平面的**前端**，它是整个系统的"中枢神经"。

**核心职责**：
1. 提供 Kubernetes API 的 REST 接口（所有 `kubectl` 命令最终都转化为对 API Server 的 HTTP 请求）
2. 认证（Authentication）、授权（Authorization）、准入控制（Admission Control）的统一入口
3. 作为集群状态的前端，所有数据通过它写入/读取 etcd
4. 提供 Watch 机制，供其他组件订阅资源变更事件
5. 处理 API 对象的版本转换和校验

**架构细节**：

```
请求 → 认证（AuthN）→ 授权（AuthZ）→ 准入控制（Admission）→ etcd 写入
  │         │              │                │
  │         │              │                │
  ▼         ▼              ▼                ▼
客户端  验证身份      检查权限        修改/验证对象      持久化数据
证书/TLS    RBAC      Mutating+      写入etcd
Token   Node/ABAC    Validating
OIDC               Webhook
```

**请求处理链（Handler Chain）**：
1. **HTTPS Handler**：TLS 终结，解析请求
2. **Authentication Handler**：验证调用方身份（X509 证书、Bearer Token、OIDC）
3. **Authorization Handler**：检查是否拥有执行该操作（verb）的权限（RBAC、Node、ABAC、Webhook）
4. **Admission Handler**：Mutating Admission（修改对象）→ Object Schema Validation → Validating Admission（验证对象）
5. **Registry**：将对象写入 etcd

API Server 是无状态的，可以水平扩展。多个 API Server 实例之间通过负载均衡器分发请求，它们共享同一个 etcd 集群。

**安全设计细节**：
- API Server 监听两个端口：
  - `6443`：安全端口，强制执行 TLS 和认证
  - `8080`：非安全端口（默认关闭，K8s 新版本已完全移除），仅用于本地测试
- 支持的认证方式：
  - X509 客户端证书（kubelet、scheduler、controller-manager 等组件使用）
  - Bearer Token（ServiceAccount Token）
  - Bootstrap Token（用于节点加入集群）
  - OpenID Connect（OIDC）Token
  - Webhook Token Authentication
  - Authenticating Proxy

### 1.3.2 etcd —— 分布式键值存储

etcd 是 CoreOS 开发的分布式一致性键值存储系统（用 Go 语言编写），基于 Raft 共识算法。它是 K8s 集群的**单一数据源（Single Source of Truth）**，存储了所有 API 对象（Pod、Service、ConfigMap、Secret 等）的信息。

详细内容见后续章节 `15_etcd.md`。

**在 K8s 中的角色**：
- 存储所有集群状态数据
- 支持 Watch 机制（K8s 控制器的 List-Watch 核心依赖）
- 提供乐观并发控制（ResourceVersion 用于冲突检测）
- 事务支持（用于保证多对象操作的原子性）

**数据存储形式**：
- 所有 K8s 对象以 JSON 或 Protobuf 格式存储在 etcd 中
- key 格式：`/registry/<resource-type>/<namespace>/<name>`（如 `/registry/pods/default/nginx-7cfc6d-abcde`）
- 每个对象的 `metadata.resourceVersion` 是 etcd 的全局递增版本号，用于乐观并发控制

### 1.3.3 kube-scheduler —— Pod 调度器

kube-scheduler 负责将新创建的、尚未分配 Node 的 Pod（即 `.spec.nodeName` 为空的 Pod）调度到集群中的一个合适节点上。

**调度流程的两阶段**：

**阶段一：过滤（Filtering / Predicates）**
排除所有不满足 Pod 需求的节点：

| 过滤条件 | 说明 |
|----------|------|
| PodFitsResources | 节点是否有足够的 CPU、内存、GPU 等资源 |
| PodFitsHost | 节点名称是否匹配 Pod 的 nodeName |
| PodFitsHostPorts | 节点上指定的 HostPort 是否被占用 |
| PodMatchNodeSelector | 节点标签是否匹配 Pod 的 nodeSelector |
| NoVolumeZoneConflict | 节点是否满足存储卷的区域限制 |
| NoDiskConflict | 节点是否有存储卷冲突 |
| CheckNodeMemoryPressure | 节点是否处于内存压力状态 |
| CheckNodePIDPressure | 节点 PID 是否耗尽 |
| CheckNodeDiskPressure | 节点磁盘是否不足 |
| MatchInterPodAffinity | 节点是否满足 Pod 亲和性/反亲和性规则 |
| TolerateTaints | 节点污点是否被 Pod 容忍 |

**阶段二：打分（Scoring / Priorities）**
为过滤后的每个节点打分排序：

| 打分策略 | 说明 |
|----------|------|
| LeastRequestedPriority | 空闲资源最多的节点得分高（均衡负载） |
| MostRequestedPriority | 空闲资源最少的节点得分高（紧凑装箱） |
| BalancedResourceAllocation | CPU 和内存使用比例均衡的节点得分高 |
| NodeAffinityPriority | 匹配软亲和性（preferred）的节点得分高 |
| TaintTolerationPriority | 容忍污点的数量影响得分 |
| ServiceSpreadingPriority | Pod 跨节点、Zone 分散得分高 |
| ImageLocalityPriority | 已经缓存了容器镜像的节点得分高（减少镜像拉取时间） |
| InterPodAffinityPriority | 满足 Pod 间亲和性的节点得分高 |

**调度过程详细步骤**：
1. Scheduler 通过 List-Watch 机制监听 API Server 中 `spec.nodeName` 为空的 Pod
2. 将待调度 Pod 放入调度队列（Scheduling Queue）
3. 从队列中取出 Pod，执行过滤和打分
4. 如果找到合适节点，通过 API Server 更新 Pod 的 `spec.nodeName` 字段（Binding 操作）
5. 调度失败时，Pod 停留在 Pending 状态，Scheduler 定期重试

**调度器扩展性**：
- **Scheduling Framework**（v1.19+）：将调度过程拆分为多个扩展点（PreFilter、Filter、PostFilter、PreScore、Score、NormalizeScore、Reserve、Permit、PreBind、Bind、PostBind），允许自定义插件
- **Multiple Scheduler**：可部署多个调度器，Pod 通过 `schedulerName` 指定使用哪个
- **Scheduler Extender**：通过 HTTP Webhook 调用外部调度策略

### 1.3.4 kube-controller-manager —— 控制器管理器

kube-controller-manager 是一个**控制器集合**，它将多个独立的控制循环打包在一个进程中运行以减少运维复杂度。

**核心控制器列表**：

| 控制器 | 职责 | 触发条件 |
|--------|------|----------|
| **Node Controller** | 监控节点健康状态（心跳），处理节点失联 | 节点心跳超时（默认 40s） |
| **Replication Controller** | 确保 RC 指定的 Pod 副本数 | 监听 RC 和 Pod 变更 |
| **ReplicaSet Controller** | 确保 RS 指定的 Pod 副本数 | 监听 RS 和 Pod 变更 |
| **Deployment Controller** | 管理 Deployment（创建 RS、滚动更新） | 监听 Deployment 变更 |
| **StatefulSet Controller** | 管理 StatefulSet（有序启动/终止） | 监听 StatefulSet 变更 |
| **DaemonSet Controller** | 确保每个节点运行一个 Pod 副本 | 监听 DaemonSet 和 Node 变更 |
| **Job Controller** | 管理 Job（运行到完成） | 监听 Job 变更 |
| **CronJob Controller** | 管理 CronJob（定时创建 Job） | 按 Cron 表达式触发 |
| **Service Controller** | 管理 Service 的 Endpoints | 监听 Service 和 Pod 变更 |
| **EndpointSlice Controller** | 管理 EndpointSlice 对象（取代 Endpoints） | 监听 Service 和 Pod 变更 |
| **Namespace Controller** | 管理 Namespace（删除时级联删除资源） | 监听 Namespace 变更 |
| **ServiceAccount Controller** | 为每个 Namespace 创建默认 ServiceAccount | 监听 Namespace 变更 |
| **Token Controller** | 管理 ServiceAccount Token Secret | 监听 ServiceAccount 变更 |
| **PVC Protection Controller** | 保护使用中的 PVC 不被删除 | 监听 PVC 变更 |
| **Garbage Collector** | 垃圾回收（级联删除、孤儿清理） | 监听 OwnerReference 变更 |

**控制器工作模式（以 Deployment Controller 为例）**：

```
1. 通过 Informer 监听 Deployment 对象的变更
        ↓
2. 当 Deployment 被创建/更新，触发 reconcile 函数
        ↓
3. 查看当前 ReplicaSet 状态
        ↓
4. 对比期望状态（Deployment.spec）与实际状态
        ↓
5. 如需创建新 ReplicaSet（滚动更新），创建之
        ↓
6. 调整新旧 ReplicaSet 副本数
        ↓
7. 更新 Deployment 的 status 字段
```

**控制循环的并发安全**：
- 使用队列（WorkQueue）保证同类事件有序处理
- 通过 `metadata.resourceVersion` 实现乐观并发控制，防止同时修改
- 控制器使用 Leader Election（通过 etcd 的租约机制）确保高可用部署时只有一个实例在活跃工作

### 1.3.5 cloud-controller-manager —— 云控制器管理器

cloud-controller-manager 是云厂商相关的控制器，将云平台特定的逻辑从核心控制器中分离出来。

**核心控制器**：
| 控制器 | 职责 |
|--------|------|
| **Node Controller** | 通过云厂商 API 检查云中节点是否已删除，清理失联节点 |
| **Route Controller** | 配置云平台的路由表，使 Pod 网络在不同节点间可达 |
| **Service Controller** | 创建、更新、删除云平台的负载均衡器（对应 type: LoadBalancer 的 Service）|

**设计意义**：
- 将云厂商特定代码与核心 K8s 代码解耦
- 云厂商可以独立开发和发布自己的 cloud-controller-manager（如 AWS、Azure、GCP、阿里云、腾讯云都有各自的实现）
- 本地部署（裸金属）不需要此组件

## 1.4 工作节点组件深度解析

### 1.4.1 kubelet —— 节点主管

kubelet 是运行在每个 Node 上的核心代理，是 K8s 在节点上的"代言人"。它负责管理本节点上的所有 Pod 及其容器的生命周期。

**核心职责**：

1. **Pod 生命周期管理**：
   - 从 API Server 或本地文件（Static Pod）获取 PodSpec
   - 通过 CRI 接口调用容器运行时创建、启动、停止容器
   - 执行 Pod 探针（Liveness、Readiness、Startup Probe）
   - 执行 Pod 生命周期钩子（PostStart、PreStop）

2. **资源管理**：
   - 通过 cgroups 实施容器的资源限制（CPU、Memory）
   - 监控容器资源使用情况
   - 根据资源压力执行 Pod 驱逐（Eviction）

3. **节点管理**：
   - 向 API Server 注册节点
   - 定期上报节点状态（心跳，默认每 10s）
   - 上报节点资源容量

4. **卷管理**：
   - 挂载 Pod 所需的 Volume（通过 CSI 插件或其他卷插件）
   - 管理 Secret、ConfigMap 等特殊卷

5. **健康检查**：
   - 执行 Liveness Probe（失败时重启容器）
   - 执行 Readiness Probe（失败时从 Service 摘除）
   - 执行 Startup Probe（启动期间单独探测，成功后切换为 Liveness）

**kubelet 的内部组件**：

```
kubelet
├── Pod Manager：管理 Pod 生命周期
├── Volume Manager：管理 Volume 的挂载/卸载
├── Probe Manager：管理容器探针
├── Secret/ConfigMap Manager：管理配置数据的更新
├── CRI Shim：对接容器运行时
├── CNI Shim：调用 CNI 插件配置容器网络
├── CSI Shim：调用 CSI 插件管理存储
├── Node Status Manager：上报节点状态
├── Eviction Manager：资源压力下的 Pod 驱逐
├── CPU Manager：CPU 亲和性策略和独占 CPU 分配
├── Memory Manager：内存的亲和性分配
├── Device Manager：管理扩展设备（GPU、FPGA 等）
└── Image Manager：容器镜像管理（拉取、缓存、垃圾回收）
```

### 1.4.2 kube-proxy —— 网络代理

kube-proxy 运行在每个节点上，实现了 K8s Service 的网络抽象——将访问 Service VIP（ClusterIP）的流量负载均衡到后端 Pod 上。

**三种代理模式**：

**1. userspace 模式（已废弃）**
- kube-proxy 在用户态监听一个随机端口，通过 iptables 将 Service 流量转发到该端口，再由 kube-proxy 代理转发到后端 Pod
- 缺点：用户态和内核态切换，性能极差
- 已在 v1.2 后被 iptables 替代

**2. iptables 模式（默认，广泛使用）**
- kube-proxy 监听 Service 和 Endpoints 的变化，动态生成 iptables 规则
- 流量路径：

```
客户端 → ClusterIP → iptables DNAT 规则 → 随机选择一个 Pod IP → 转发
```

- 转发规则链：

```
KUBE-SERVICES → KUBE-SVC-XXXX → KUBE-SEP-XXXX（每个 Endpoint 一条链）
```

- 工作原理：

```
# Service ClusterIP 到 Endpoints 的随机负载均衡
-A KUBE-SERVICES -d 10.96.0.1/32 -p tcp -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m statistic --mode random --probability 0.33333 -j KUBE-SEP-AAA
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m statistic --mode random --probability 0.50000 -j KUBE-SEP-BBB
-A KUBE-SVC-NPX46M4PTMTKRN6Y -j KUBE-SEP-CCC
# 每个 KUBE-SEP-XXX 执行 DNAT 到具体 Pod IP:Port
-A KUBE-SEP-AAA -p tcp -m tcp -j DNAT --to-destination 10.244.1.2:8080
```

- 优点：纯内核态，性能好；成熟稳定
- 缺点：
  - 大规模集群中 iptables 规则数量爆炸（O(n) 复杂度，n = 节点数 × Service 数）
  - iptables 不支持高级负载均衡算法（仅随机）
  - 规则变更时需要全量替换 iptables 规则，可能造成短暂中断
  - 没有连接追踪，无法做基于会话的亲和性（iptables 的连接亲和性通过 conntrack 实现，但有 10800s 超时限制）

**3. IPVS 模式（推荐用于大规模集群）**
- IPVS（IP Virtual Server）是 Linux 内核中的传输层负载均衡器（LVS 的核心模块）
- 使用 IPVS 替代 iptables 的 DNAT 规则
- kube-proxy 创建虚拟服务器（Virtual Server），后端管理真实服务器（Real Server）

- 支持的调度算法：

| 算法 | 说明 |
|------|------|
| rr | 轮询（Round Robin） |
| lc | 最少连接（Least Connection） |
| dh | 目标地址哈希（Destination Hashing） |
| sh | 源地址哈希（Source Hashing） |
| sed | 最短期望延迟（Shortest Expected Delay） |
| nq | 从不排队（Never Queue） |

- 优点：
  - 性能远高于 iptables（O(1) 查找，而非 O(n) 遍历）
  - 支持丰富的调度算法
  - 规则变更时增量更新，不影响已有连接
- 缺点：需要在节点上加载 IPVS 内核模块

**4. nftables 模式（K8s v1.29+ Alpha）**
- 使用 nftables 替代 iptables 作为数据面
- nftables 是 iptables 的继任者，规则更简洁、性能更好

**5. eBPF / kube-proxy-free 模式**
- Cilium 等 CNI 插件通过 eBPF 直接在数据路径中实现 Service 负载均衡，**完全不需要 kube-proxy**
- 这是未来的趋势方向

### 1.4.3 Container Runtime —— 容器运行时

容器运行时负责运行容器。K8s 通过 CRI（Container Runtime Interface）接口与容器运行时通信。

**CRI 接口定义**：

```protobuf
// RuntimeService 定义了容器和 Pod 的生命周期操作
service RuntimeService {
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse);
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse);
    rpc ListContainers(ListContainersRequest) returns (ListContainersResponse);
    rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse);
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse);
    rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse);
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse);
    rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse);
    // ... 以及其他方法
}

// ImageService 定义了镜像管理操作
service ImageService {
    rpc PullImage(PullImageRequest) returns (PullImageResponse);
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse);
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse);
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse);
    // ... 以及其他方法
}
```

**常见容器运行时**：

| 运行时 | 简介 | CRI 支持 |
|--------|------|----------|
| **containerd** | CNCF 毕业项目，Docker 的核心组件独立发展而来，K8s 最推荐的生产级运行时 | 内置 CRI 插件 |
| **CRI-O** | Red Hat 主导，专为 K8s 设计的轻量级 CRI 运行时 | 原生 CRI |
| **Docker Engine** | 最早的容器运行时，但在 K8s v1.24 后 dockershim 被移除 | 需通过 cri-dockerd 适配器 |
| **Kata Containers** | 安全容器运行时，每个容器运行在独立的轻量级虚拟机中 | 通过 CRI-O/containerd 间接支持 |
| **gVisor** | Google 开发的用户态内核，为容器提供额外的安全隔离层（Sentry） | 通过 containerd 的 runsc shim |

**dockershim 的移除**：
- Docker 本身不是一个 CRI 兼容的运行时（因为 CRI 是在 Docker 之后定义的）
- K8s 在早期版本中内置了 `dockershim`（dockershim），将 CRI 调用转换为 Docker API 调用
- 为了减少维护负担并推动标准 CRI，K8s 从 v1.20 宣布弃用 dockershim，在 v1.24 中完全移除
- 迁移路径：使用 containerd 替代 Docker（Docker 底层也使用 containerd，迁移成本低）

## 1.5 标准接口体系

### 1.5.1 CRI（Container Runtime Interface）

CRI 是 kubelet 与容器运行时之间的通信接口。

```
kubelet
   │
   │ gRPC (Unix Socket)
   │
   ▼
┌──────────────────┐
│   CRI Shim       │  ← 容器运行时需实现此接口
│                  │
│  RuntimeService  │  ← Pod Sandbox 和 Container 生命周期管理
│  ImageService    │  ← 镜像管理
└──────────────────┘
   │
   ▼
┌──────────────────┐
│  Container Runtime│  ← containerd / CRI-O
└──────────────────┘
```

### 1.5.2 CNI（Container Network Interface）

CNI 是容器网络插件接口标准，定义了容器网络的配置方式。

```
Pod (网络命名空间)
   │
   │ 调用 CNI 插件
   │
   ▼
┌──────────────────┐
│   CNI Plugin     │  ← 配置网络接口、IP 地址、路由
│                  │
│  ADD (创建Pod)    │  ← 分配 IP，创建 veth pair，配置路由
│  DEL (删除Pod)    │  ← 回收 IP，删除 veth pair，清理路由
│  CHECK (检查)     │  ← 验证网络配置是否仍然正确
└──────────────────┘
   │
   ▼
┌──────────────────┐
│  Network Backend │  ← Calico、Flannel、Cilium 等
└──────────────────┘
```

Pod 的网络配置流程：
1. kubelet 创建 Pod Sandbox（Pause 容器，占用网络命名空间）
2. kubelet 调用 CNI 插件（通过标准输入/输出传递配置）
3. CNI 插件为 Pod 创建网络接口（如 veth pair），分配 IP 地址，配置路由
4. CNI 插件返回配置结果（IP 地址、网关、DNS 等）

### 1.5.3 CSI（Container Storage Interface）

CSI 是容器存储接口标准，定义了存储系统与容器编排平台的交互方式。

```
┌──────────┐      ┌──────────────────────────────────────┐
│ kubelet  │      │        CSI 组件                      │
│          │ gRPC │                                      │
│  ┌───────┼──────┤  ┌──────────────────────────────┐   │
│  │ CSI   │      │  │    CSI Controller Plugin      │   │
│  │ Driver│      │  │  (CreateVolume, DeleteVolume, │   │
│  │ Regis │      │  │   ControllerPublishVolume)    │   │
│  │ trar  │      │  └──────────────────────────────┘   │
│  └───────┼──────┤  ┌──────────────────────────────┐   │
│          │      │  │     CSI Node Plugin           │   │
│          │      │  │  (NodeStageVolume,             │   │
│          │      │  │   NodePublishVolume,           │   │
│          │      │  │   NodeUnpublishVolume)         │   │
│          │      │  └──────────────────────────────┘   │
└──────────┘      └──────────────────────────────────────┘
```

**CSI 服务的三个组件**：

| 组件 | 职责 |
|------|------|
| **CSI Identity** | 返回插件的身份信息（名称、支持的版本、能力） |
| **CSI Controller** | 处理卷的生命周期管理（创建、删除、快照、扩容、挂载到节点） |
| **CSI Node** | 在节点上将卷挂载到 Pod 的目录中 |

## 1.6 核心工作流程深入

### 1.6.1 Pod 创建的完整流程

以下是一个 Pod 从创建到运行的完整时序：

```
Step 1: 用户通过 kubectl 提交 Pod YAML
   │
   ▼
Step 2: kubectl 将请求发送到 API Server（HTTPS POST）
   │
   ▼
Step 3: API Server 处理请求链
   │  → Authentication（验证 kubectl 的身份）
   │  → Authorization（验证用户/ServiceAccount 是否有创建 Pod 的权限）
   │  → Admission（Mutating Webhook 修改 Pod → Schema 验证 → Validating Webhook 校验）
   │
   ▼
Step 4: API Server 将 Pod 对象写入 etcd
   │  → Pod 状态初始为 Pending
   │  → 向所有 Watch 该资源的订阅者发送事件
   │
   ▼
Step 5: Scheduler Watch 到新的 Pending Pod
   │  → 进入调度队列
   │  → 执行过滤（排除不合格节点）
   │  → 执行打分（选出最优节点）
   │  → 执行 Bind（将 nodeName 写入 Pod 对象）
   │  → API Server 更新 etcd 中的 Pod 对象
   │
   ▼
Step 6: 目标节点上的 kubelet Watch 到分配给自己的 Pod
   │  → 创建 Pod Sandbox（通过 CRI，创建 Pause 容器和网络命名空间）
   │  → 调用 CNI 插件配置 Pod 网络（分配 IP、配置路由）
   │  → 通过 CRI 拉取 init 容器镜像 → 启动 init 容器 → 等待 init 容器完成
   │  → 通过 CRI 拉取应用容器镜像
   │  → 通过 CRI 创建并启动应用容器
   │  → 执行 PostStart Hook（如已定义）
   │  → 更新 Pod Status 到 API Server
   │
   ▼
Step 7: kubelet 持续上报 Pod 状态
   │  → 执行 Readiness Probe（成功后标记 Pod 为 Ready）
   │  → 执行 Liveness Probe（检查容器健康）
   │  → Service Controller 将 Ready 的 Pod IP 加入 Service 的 Endpoints
   │  → kube-proxy 更新转发规则（iptables/IPVS）
   │
   ▼
Step 8: Pod 进入 Running 状态，开始接收流量
```

### 1.6.2 Deployment 滚动更新的完整流程

```
初始状态:
  Deployment replicas=3
  ReplicaSet-v1 replicas=3 (nginx:1.24)
  Pod-1, Pod-2, Pod-3  (running)

1. 用户更新 Deployment 镜像为 nginx:1.25
   kubectl apply -f deployment.yaml  (或 kubectl set image)

2. Deployment Controller 监听到变更：
   - 创建新 ReplicaSet-v2 (replicas=0, image=nginx:1.25)
   - 逐步扩容 ReplicaSet-v2，逐步缩容 ReplicaSet-v1

3. 滚动更新过程（maxUnavailable=1, maxSurge=1）：
   第 1 步: ReplicaSet-v2=0→1 (创建 1 个新 Pod)
            等待新 Pod Ready
            第 2 步: ReplicaSet-v1=3→2 (删除 1 个旧 Pod)
   第 3 步: ReplicaSet-v2=1→2 (创建 1 个新 Pod)
            等待新 Pod Ready
            第 4 步: ReplicaSet-v1=2→1 (删除 1 个旧 Pod)
   第 5 步: ReplicaSet-v2=2→3 (创建 1 个新 Pod)
            等待新 Pod Ready
            第 6 步: ReplicaSet-v1=1→0 (删除最后 1 个旧 Pod)
   ReplicaSet-v1 保留（revision history limit 决定保留几个历史 RS）

4. 滚动更新属性：
   - maxUnavailable: 最多允许多少个 Pod 不可用
   - maxSurge: 最多允许多少个 Pod 超出期望副本数
   - minReadySeconds: Pod 变为 Ready 后等待多少秒才认为是可用
   - progressDeadlineSeconds: 更新超时时间（超时标记为 Failed）
```

### 1.6.3 节点心跳与失联处理

```
正常运行：
kubelet → 每 10s → API Server → 更新 Node 的 .status.conditions (Ready 状态)
                                       → 更新 .status.conditions 的 lastHeartbeatTime

心跳超时：
Node Controller → 每 5s → 检查所有 Node 的 lastHeartbeatTime
   │
   ├── 超过 node-monitor-grace-period（默认 40s）：更新 Node 为 Unknown
   │
   └── 超过 pod-eviction-timeout（默认 5min）：
        → 将 Node 标记为 NotReady 并打上 taint
        → 触发该 Node 上所有 Pod 的驱逐流程
        → Scheduler 将 Pod 调度到其他健康节点
        → 被驱逐的 Pod 被原节点的 kubelet 清理（如果该节点恢复）
```

## 1.7 组件间通信安全

### 1.7.1 控制平面内部通信

```
┌─────────────────────────────────────────────┐
│              通信矩阵（Control Plane）         │
│                                              │
│  etcd ←→ API Server          (TLS 双向认证)   │
│  API Server ←→ kubelet       (TLS 认证+授权)   │
│  Scheduler → API Server      (TLS + RBAC)    │
│  Controller Manager → API Server (TLS + RBAC) │
│  API Server ←→ cloud-controller-manager (TLS) │
└─────────────────────────────────────────────┘
```

### 1.7.2 工作节点与控制平面通信

```
工作节点：
  kubelet → API Server：
    认证：kubelet 客户端证书 或 Bootstrap Token
    授权：Node Authorizer（特殊授权机制，允许 kubelet 管理自己节点上的 Pod）
    端口：6443

  kube-proxy → API Server：
    认证：Node 的 kubeconfig（客户端证书）
    授权：RBAC，需要 Watch Service、Endpoints 等资源
    端口：6443
```

### 1.7.3 节点授权机制（Node Authorizer）

Node Authorizer 是 kubelet 专用的特殊授权机制：

```
kubelet 只能操作与自己的节点相关的资源：
  - 读取：Service、Endpoint、Node、Pod、Secret、ConfigMap、PV、PVC
  - 写入：节点状态（Node Status）、Pod 状态（Pod Status）
  - 限制：kubelet 不能修改其他节点的 Pod、不能创建跨越节点的资源
```

## 1.8 集群高可用架构

### 1.8.1 控制平面高可用

```
                    ┌──────────────┐
                    │   Load       │
                    │   Balancer   │  ← HAProxy / Nginx / 云 LB
                    │  (vIP:6443)  │
                    └──┬───┬───┬───┘
                       │   │   │
          ┌────────────▼───▼───▼────────────┐
          │          API Server X 3          │
          │  (active-active, 无状态，水平扩展)  │
          └────────────┬───┬───┬────────────┘
                       │   │   │
          ┌────────────▼───▼───▼────────────┐
          │           etcd Cluster           │
          │   etcd-1     etcd-2     etcd-3   │
          │   (Raft: 1 Leader + 2 Followers)  │
          └──────────────────────────────────┘

Scheduler 和 Controller Manager 高可用：
  - 部署 3 副本（每个控制平面节点 1 个）
  - 通过 Leader Election（etcd 租约）确保同一时间只有 1 个活跃实例
  - 活跃实例故障后，其他实例通过选举接管工作
  - Scheduler 和 Controller Manager 各自独立选举 Leader
```

### 1.8.2 etcd 高可用

- etcd 集群至少需要 3 个节点（Raft 多数派要求：2N+1 个节点容忍 N 个故障）
- etcd 推荐奇数个节点：3、5、7（超过 7 个节点写入延迟会增加）
- 数据一致性保证了 K8s 集群所有控制平面组件看到的集群状态一致

### 1.8.3 高可用中的组件配置参数

```shell
# kube-apiserver
--etcd-servers=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379  # etcd 集群地址
--advertise-address=<本机IP>         # 广播地址
--bind-address=0.0.0.0               # 监听地址

# kube-controller-manager
--leader-elect=true                  # 启用 Leader 选举
--leader-elect-lease-duration=15s    # 租约时长
--leader-elect-renew-deadline=10s    # 续约超时
--leader-elect-retry-period=2s       # 重试间隔
--controllers=*,bootstrapsigner,tokencleaner  # 启用的控制器列表

# kube-scheduler
--leader-elect=true                  # 启用 Leader 选举（参数与 Controller Manager 相似）
```

## 1.9 组件版本兼容性

### 1.9.1 版本偏差策略（Version Skew Policy）

K8s 的组件之间需要版本兼容，官方定义了版本偏差策略：

| 关系 | 允许的版本偏差 |
|------|---------------|
| kube-apiserver | 1.X（基准版本） |
| kube-controller-manager、kube-scheduler、cloud-controller-manager | ≤1 个次版本（1.X 或 1.(X-1)） |
| kubelet、kube-proxy | ≤2 个次版本（1.X、1.(X-1)、1.(X-2)） |
| kubectl | 1.X ± 1 个次版本 |

### 1.9.2 为什么控制平面组件不能高于 API Server

- Controller Manager 和 Scheduler 等组件在与 API Server 通信时，可能依赖 API 的新特性或新字段
- 如果控制平面组件版本高于 API Server，它可能期望使用 API Server 不存在的特性，导致行为异常
- 降级兼容通常被保证（低版本的组件可以与高版本 API Server 通信），但反过来不保证

## 1.10 架构层面常见面试题

| 问题 | 要点 |
|------|------|
| K8s 的架构是什么样的？ | 控制平面（API Server、etcd、Scheduler、Controller Manager）+ 工作节点（kubelet、kube-proxy、Container Runtime） |
| API Server 在整个架构中的作用？ | 集群统一入口、认证授权准入、REST API 网关、唯一写入 etcd 的组件 |
| etcd 在 K8s 中的作用？ | 单一数据源，存储所有集群状态，支持 Watch 机制 |
| Scheduler 是如何工作的？ | 监听 Pending Pod → 过滤不合格节点 → 打分排序 → 绑定到最优节点 |
| kubelet 的职责是什么？ | Pod 生命周期管理、探针执行、卷挂载、节点状态上报、资源管理 |
| kube-proxy 的作用和工作模式？ | Service 负载均衡实现，支持 userspace、iptables、IPVS 三种模式 |
| 什么是 CRI、CNI、CSI？ | 容器运行时接口、容器网络接口、容器存储接口，都是 K8s 的可插拔标准 |
| K8s 如何实现高可用？ | API Server 多实例无状态水平扩展 + etcd Raft 集群 + Scheduler/Controller Manager Leader Election |
