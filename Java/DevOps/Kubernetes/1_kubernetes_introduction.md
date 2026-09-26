# 1.认识 Kubernetes

## 1.1 什么是 Kubernetes

Kubernetes（简称 K8s，K 与 s 之间有 8 个字母）是一个开源的**容器编排与调度平台**，由 Google 于 2014 年 6 月首次宣布并将其捐赠给 CNCF（Cloud Native Computing Foundation，云原生计算基金会），由 CNCF 进行托管。它是目前**事实上的容器编排标准**，也是云原生生态的核心基石。

Kubernetes 这个名称来源于希腊语"κυβερνήτης"，意为"舵手"或"领航员"，与"Cybernetics（控制论）"同词根，寓意着对容器集群的自动驾驶与精准控制。

Kubernetes 的核心定位并非仅仅是一个"容器管理工具"，而是一个**分布式操作系统的内核**：
- 它提供了对底层基础设施（计算、存储、网络）的**统一抽象**
- 它为应用提供了声明式的**期望状态管理（Desired State）**
- 它通过**控制循环（Control Loop）**持续驱动实际状态向期望状态收敛
- 它屏蔽了底层基础设施的异构性（裸金属、虚拟机、公有云、私有云）

从工程角度来看，Kubernetes 解决的核心问题是：**如何在多台机器上可靠地运行和管理容器化应用**。这看似简单的问题，在分布式系统中涉及容器调度、服务发现、负载均衡、滚动更新、自动扩缩容、自愈、配置管理、密钥管理、存储编排、网络策略等一整套复杂机制。

## 1.2 为什么需要 Kubernetes —— 容器时代的困境

### 1.2.1 单体容器时代的局限

Docker 的出现解决了应用打包和隔离的问题，但在生产环境中，仅有 Docker 是不够的：

| 问题 | 具体表现 |
|------|----------|
| **单机限制** | 单个 Docker 宿主机的计算、存储、网络资源有限，无法承载大规模应用 |
| **手动运维** | 容器启停、扩缩容、健康检查需要手动操作或编写大量脚本 |
| **零散的管理** | 应用在哪些机器上运行、运行了多少副本、预期状态是什么，缺乏统一的"真理源" |
| **服务发现问题** | 容器 IP 动态变化，微服务之间如何互相发现和通信 |
| **滚动更新** | 如何平滑地升级应用而不导致服务中断，出现问题如何快速回滚 |
| **负载均衡** | 多个容器副本之间如何分发流量 |
| **存储管理** | 有状态应用（数据库、消息队列）的持久化数据如何与容器生命周期解耦 |
| **配置管理** | 配置变更如何不重启容器就生效，敏感信息（密钥、证书）如何安全地注入容器 |
| **自愈能力** | 容器崩溃、节点故障时如何自动恢复 |
| **资源调度** | 如何决定容器应该运行在集群中的哪台机器上，以最大化资源利用率 |

### 1.2.2 业界竞品的历史背景

在 Kubernetes 诞生之前及同期，业界有多种容器编排方案：

| 方案 | 简介 | 最终结局 |
|------|------|----------|
| **Docker Swarm** | Docker 官方推出的容器编排工具，与 Docker Engine 深度集成，使用简单 | 2019 年 Docker Enterprise 被 Mirantis 收购，Swarm 不再作为独立产品发展，但 Swarm Mode 至今仍内置于 Docker Engine 中 |
| **Apache Mesos + Marathon** | Mesos 是一个通用的资源调度框架，Marathon 作为其上的容器编排框架 | Mesos 项目已于 2021 年正式退役（进入 Apache Attic），被 Kubernetes 完全替代 |
| **HashiCorp Nomad** | HashiCorp 推出的单一二进制调度器，支持容器和非容器化应用（JAR、二进制、虚拟机） | 仍在活跃发展，在特定场景（简单、轻量、非容器化工作负载）中有其生态位 |
| **Rancher 1.x（Cattle）** | Rancher Labs 早期自研的编排引擎 | Rancher 2.0 全面转向 Kubernetes，Cattle 被废弃 |

Kubernetes 最终胜出的核心原因：
1. **Google 的工程基因**：源自 Borg 和 Omega 系统 15 年的大规模生产经验，设计上天然具有大规模、高可用的工程能力
2. **声明式 API 与控制器模式**：这是 K8s 的杀手级设计，通过声明期望状态 + 控制循环自动收敛，远比命令式配置更适应大规模、动态变化的场景
3. **可扩展性**：通过 CRD（自定义资源）、Operator、Admission Webhook、Scheduler Extender、CNI/CSI/CRI 接口，K8s 成为了一个"平台平台"，任何人都可以在此基础上构建自己的平台能力
4. **CNCF 的中立治理**：由 CNCF 托管而非单一商业公司控制，吸引了全行业的贡献者，形成了最大的云原生开源生态
5. **生态网络效应**：几乎所有云原生项目都以 K8s 为标准集成目标，形成了强大的网络效应

## 1.3 Kubernetes 的发展历史

### 1.3.1 前身：Borg 和 Omega

Kubernetes 并非凭空设计的产物，它的设计哲学和架构直接继承了 Google 内部两个大规模生产系统：

**Borg（2003-2015）**
- Google 的内部容器管理系统，管理着 Google 所有在线和离线服务
- 直接支撑 Google Search、Gmail、GFS、Bigtable 等核心服务
- 核心设计：Cell（集群单元）、Alloc（资源预留）、Job/Task（工作单元）、Priority（优先级与抢占）、Quota（配额管理）
- Borg 的架构经验直接影响了 K8s 的设计：
  - K8s Pod 对应 Borg Alloc（一组紧密耦合的容器共享资源和命名空间）
  - K8s Namespace 对应 Borg Cell 内的逻辑分区
  - K8s Controller Manager 的思想来源于 Borgmaster

**Omega（2010-2014）**
- Google 内部对 Borg 的演进方案，采用了**乐观并发控制（Optimistic Concurrency Control）** 和**共享状态架构**
- 所有组件直接读写共享状态（Paxos 副本），而非通过单一的 Borgmaster 串行处理
- 虽然 Omega 在内部分阶段淘汰了 Borg，但其声明式共享状态的设计理念深刻影响了 K8s 的 API Server + etcd 架构

### 1.3.2 正式诞生与关键里程碑

| 时间 | 事件 |
|------|------|
| **2014.6** | Google 宣布 Kubernetes 项目开源，由 Joe Beda、Brendan Burns、Craig McLuckie 发起 |
| **2015.3** | K8s 发布 v0.14，引入 ReplicationController（后来演化为 ReplicaSet） |
| **2015.7** | K8s v1.0 正式发布，Google 与 Linux 基金会共同成立 CNCF，K8s 作为种子项目捐赠给 CNCF |
| **2015.11** | K8s v1.1 发布，引入 HorizontalPodAutoscaler（HPA）和 Ingress |
| **2016.3** | K8s v1.2 发布，引入 Deployment、ConfigMap、DaemonSet |
| **2016.9** | K8s v1.4 发布，kubeadm 工具诞生，简化集群部署 |
| **2016.12** | K8s v1.5 发布，引入 StatefulSet（替代 PetSet）、PodDisruptionBudget、CRI |
| **2017.3** | K8s v1.6 发布，引入 RBAC（正式稳定）、etcd v3、增强的调度功能 |
| **2017.6** | K8s v1.7 发布，引入 NetworkPolicy API、Local Persistent Volumes、CRD |
| **2017.12** | K8s v1.9 发布，引入 Apps/v1 API（Deployment 等控制器毕业为正式版） |
| **2018.3** | K8s v1.10 发布，引入 CSI（容器存储接口） | 
| **2018.9** | K8s v1.12 发布，kubelet TLS 自举、RuntimeClass 引入 |
| **2019.3** | K8s v1.14 发布，Windows 节点支持毕业为正式版 |
| **2019.9** | K8s v1.16 发布，CRD 毕业为 GA（正式可用），Admission Webhook GA |
| **2019.12** | K8s v1.17 发布，Service Topology、CSI Migration 开始 |
| **2020.3** | K8s v1.18 发布，Server-side Apply、IngressClass 引入 |
| **2020.8** | K8s v1.19 发布，Ingress/V1 GA，Seccomp 默认启用，kube-scheduler 支持插件框架 |
| **2020.12** | K8s v1.20 发布，dockershim 弃用公告（移除进度开始） |
| **2021.4** | K8s v1.21 发布，CronJob GA、不可变 Secret/ConfigMap、PodSecurityPolicy 弃用 |
| **2021.8** | K8s v1.22 发布，移除大量已弃用的 beta API，Server-side Apply GA |
| **2021.12** | K8s v1.23 发布，IPv4/IPv6 双栈 GA、HPA v2 GA、PodSecurity 准入控制器引入（替代 PSP） |
| **2022.4** | K8s v1.24 发布，**dockershim 正式移除**，需要迁移到 containerd/CRI-O |
| **2022.8** | K8s v1.25 发布，PodSecurityPolicy 正式移除，PodSecurity Admission GA |
| **2023.4** | K8s v1.27 发布，SeccompDefault GA，原地升级资源（In-place Resource Resize）Alpha |
| **2023.8** | K8s v1.28 发布，SidecarContainer 毕业为 Alpha（后来名为 InitContainers restartPolicy），混合版本代理（Mixed Version Proxy）引入 |
| **2023.12** | K8s v1.29 发布，cgroups v2 GA，ReadWriteOncePod PVC 访问模式 GA |
| **2024.4** | K8s v1.30 发布，结构化认证配置（Structured Authentication Config）引入 |
| **2024.8** | K8s v1.31 发布，cgroups v1 弃用，最大允许的 Service NodePort 范围扩展 |
| **2024.12** | K8s v1.32 发布，DRA（动态资源分配）增强，原地资源调整 Beta |

Kubernetes 的版本策略：
- 每年发布 **3 个版本**，大约每 15-17 周发布一个版本
- 每个版本有约 **14 个月**的补丁支持周期（v1.25 之后）
- 版本号格式为 `v1.X.Y`（主版本 v1 长期保持，次版本 X 递增，补丁版本 Y 递增）
- 当前（2026 年）主流使用版本集中在 v1.30 ~ v1.33 之间

## 1.4 Kubernetes 的核心设计哲学

### 1.4.1 声明式 API（Declarative API）

这是 K8s 最根本的设计理念，区别于命令式（Imperative）操作。

**命令式**（告诉系统"怎么做"）：
```shell
# 手动启动 3 个 nginx 容器
docker run -d --name nginx1 nginx:1.25
docker run -d --name nginx2 nginx:1.25
docker run -d --name nginx3 nginx:1.25
# 手动扩容到 5 个
docker run -d --name nginx4 nginx:1.25
docker run -d --name nginx5 nginx:1.25
```

**声明式**（告诉系统"我想要什么"）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

将 `replicas: 3` 改为 `replicas: 5` 即可完成扩容，K8s 负责将实际状态调整为目标状态。

声明式 API 的优势：
- **幂等性**：多次应用同样的声明，结果一致，不会产生副作用
- **自愈性**：当某个副本因故障消失，K8s 会自动调谐，重新创建缺失的副本
- **可版本化**：YAML/JSON 文件可以放入 Git 进行版本控制（GitOps 的基石）
- **可审计**：声明本身就是意图的记录，便于审计和审批
- **关注点分离**：用户只需关注"最终状态是什么"，而非"如何达到最终状态"

### 1.4.2 控制循环（Control Loop）

控制循环是声明式 API 的实现机制。每个控制器都是一个控制循环：

```
for {
    desired := getDesiredState()   // 从 API Server 读取期望状态
    current := getCurrentState()   // 从集群获取实际状态
    if current != desired {
        makeChanges(current, desired)  // 驱动实际状态向期望状态收敛
    }
    sleep(reconcileInterval)
}
```

这个控制循环被称为 **Reconcile Loop（调谐循环）**，是 K8s 最核心的运行机制。它的精妙之处在于：
- **最终一致性**：不保证即刻达成目标，但保证最终达成
- **容错性**：任何组件的短暂故障都不会影响系统的最终状态——当故障恢复后，控制循环继续工作
- **无状态控制平面**：控制器不需要维护内存中的状态，所有状态存储在 etcd 中

### 1.4.3 控制器模式（Controller Pattern）

K8s 中的每一种资源（Deployment、Service、Pod 等）都有对应的控制器（Controller）来管理其生命周期。控制器的职责单一：

| 控制器 | 职责 |
|--------|------|
| Deployment Controller | 管理 ReplicaSet（滚动更新、回滚） |
| ReplicaSet Controller | 管理 Pod 副本数量 |
| StatefulSet Controller | 管理有状态 Pod（有序启动、持久标识） |
| DaemonSet Controller | 确保每个节点运行一个 Pod 副本 |
| Job Controller | 管理一次性任务 |
| CronJob Controller | 管理定时任务 |
| Node Controller | 管理节点生命周期（心跳检测、状态更新） |
| Service Controller | 管理 Endpoints（Service 与 Pod 的关联） |
| EndpointSlice Controller | 管理 EndpointSlice 对象 |
| HPA Controller | 根据指标自动调整 Pod 副本数 |

### 1.4.4 可插拔架构（Pluggable Architecture）

K8s 不是一个封闭系统，它的核心架构中预留了多种标准接口，允许替换和扩展：

| 接口 | 全称 | 作用 | 常见实现 |
|------|------|------|----------|
| **CNI** | Container Network Interface | 容器网络 | Calico、Flannel、Cilium、Weave |
| **CSI** | Container Storage Interface | 容器存储 | Ceph CSI、NFS CSI、EBS CSI、Longhorn |
| **CRI** | Container Runtime Interface | 容器运行时 | containerd、CRI-O |
| **Scheduler Extender** | — | 扩展调度策略 | 自定义调度器插件 |
| **Admission Webhook** | — | 准入控制扩展 | Kyverno、OPA/Gatekeeper |
| **Device Plugin** | — | 设备资源管理 | NVIDIA GPU Plugin、Intel SGX Plugin |

### 1.4.5 关注点分离（Separation of Concerns）

K8s 的 API 对象设计遵循了 Unix 哲学的一个变体——每个对象做一件事，做好这件事：

```
Deployment → 管理 Pod 模板和更新策略
  └── ReplicaSet → 管理 Pod 副本数量
       └── Pod → 容器运行单元
            └── Container → 应用进程
```

这种分层设计带来了：
- 每个抽象层的职责清晰，易于理解和调试
- 可以独立扩展每一层而不影响其他层
- 组合优于继承，通过组合不同对象来构建复杂的应用管理方案

## 1.5 Kubernetes 的核心特性

### 1.5.1 自动化容器调度

K8s Scheduler 根据 Pod 的资源需求、节点可用资源、亲和性/反亲和性规则、污点与容忍等约束，自动决定 Pod 应该运行在哪个节点上。调度器是可插拔的，支持自定义调度策略。

### 1.5.2 自愈（Self-Healing）

K8s 持续监控容器和节点健康状态：
- **容器自愈**：容器崩溃 → kubelet 自动重启（RestartPolicy 控制）
- **Pod 自愈**：整个 Pod 失败 → ReplicaSet 控制器创建新的 Pod 替换
- **节点自愈**：节点失联（默认 5 分钟无心跳）→ Node Controller 将该节点上的 Pod 调度到其他健康节点
- **健康检查**：支持 Liveness Probe（存活探针）、Readiness Probe（就绪探针）、Startup Probe（启动探针）

### 1.5.3 自动扩缩容（Auto-Scaling）

K8s 提供多层级的自动扩缩容能力：
- **HPA（Horizontal Pod Autoscaler）**：根据 CPU、内存或自定义指标（如 QPS、延迟）自动调整 Pod 副本数
- **VPA（Vertical Pod Autoscaler）**：自动调整 Pod 中容器的 CPU 和内存 Request/Limit
- **Cluster Autoscaler**：当 Pod 无法调度时自动扩容集群节点，空闲节点过多时缩容
- **KEDA（Kubernetes Event-Driven Autoscaling）**：基于事件源（Kafka 积压、RabbitMQ 队列长度、Prometheus 指标等）驱动扩缩容，支持缩容到零

### 1.5.4 服务发现与负载均衡

- 每个 Pod 有独立的 IP 地址（Pod IP）
- Service 提供稳定的虚拟 IP（ClusterIP）和 DNS 名称
- CoreDNS 提供集群内部的 DNS 服务发现
- kube-proxy 实现 Service 的负载均衡（支持 iptables 和 IPVS 两种模式）
- Ingress/Ingress Controller 提供七层 HTTP/HTTPS 负载均衡

### 1.5.5 存储编排

K8s 通过 PV（PersistentVolume）、PVC（PersistentVolumeClaim）、StorageClass 体系实现了存储与计算的生命周期解耦：
- **PV**：集群管理员提供的存储资源（NFS、Ceph、云磁盘等的抽象）
- **PVC**：用户对存储的请求（大小、访问模式、存储类别）
- **StorageClass**：动态存储供给（Dynamic Provisioning），按需自动创建 PV
- **CSI**：容器存储接口标准，允许第三方存储厂商开发与 K8s 集成的存储驱动（Ceph、NFS、AWS EBS、GCP PD、Azure Disk 等）

### 1.5.6 配置与密钥管理

- **ConfigMap**：存储非敏感配置（环境变量、配置文件、命令行参数）
- **Secret**：存储敏感信息（密码、Token、密钥、TLS 证书），支持 Opaque、tls、dockerconfigjson 等类型
- 支持热更新（通过文件挂载或环境变量的方式，具体行为取决于应用如何读取配置）

### 1.5.7 滚动更新与回滚

- Deployment 支持 **RollingUpdate**（滚动更新，零停机）和 **Recreate**（先删后建）两种策略
- 更新过程中可以控制最大不可用 Pod 数和最大超出期望的 Pod 数
- 保留历史 ReplicaSet，支持一键回滚（`kubectl rollout undo`）
- 支持暂停/恢复更新过程
- 蓝绿部署、金丝雀发布可通过 Service 标签选择器配合多个 Deployment 实现
- 金丝雀发布的高级模式可通过 Argo Rollouts 或 Istio 实现流量精细控制

### 1.5.8 批处理与定时任务

- **Job**：管理一次性任务，确保指定数量的 Pod 成功完成
- **CronJob**：基于 Cron 表达式定期创建 Job
- 支持并行执行（Parallelism）、完成次数（Completions）、失败重试策略、TTL 自动清理

### 1.5.9 网络策略

**NetworkPolicy** 是 K8s 内置的"防火墙"：
- 基于 Pod Selector 和 Namespace Selector 定义 Pod 间的网络通信规则
- 支持 Ingress（入站）和 Egress（出站）规则
- 默认策略：未匹配任何策略时默认放行或默认拒绝（取决于 NetworkPolicy 配置）
- 需要 CNI 插件支持（Calico、Cilium 提供丰富的网络策略实现）

### 1.5.10 多租户与资源隔离

- **Namespace**：资源资源的逻辑隔离
- **ResourceQuota**：限制 Namespace 的资源消耗总量
- **LimitRange**：限制单个 Pod/Container 的资源上下限
- **RBAC**：基于角色的访问控制
- **NetworkPolicy**：网络层隔离
- **Pod Security Standards (PSS)**：Pod 安全标准（Privileged/Baseline/Restricted）

## 1.6 Kubernetes 的架构概览

### 1.6.1 宏观架构

Kubernetes 集群由两类节点组成：

**控制平面节点（Control Plane / Master）**：
- **kube-apiserver**：集群的统一入口，所有操作的 REST API 网关
- **etcd**：分布式键值存储，保存集群的所有状态数据（ConfigMap、Secret、Pod 定义、Service 定义等）
- **kube-scheduler**：Pod 调度器，决定 Pod 运行在哪个节点
- **kube-controller-manager**：控制器管理器，运行各类控制器（Deployment Controller、ReplicaSet Controller、Node Controller 等）
- **cloud-controller-manager**（可选）：与云平台 API 交互的控制器（Node Controller、Route Controller、Service Controller）

**工作节点（Worker Node）**：
- **kubelet**：节点上的"管家"，负责管理本节点上的 Pod 生命周期（根据 PodSpec 创建/删除容器）
- **kube-proxy**：节点上的网络代理，实现 Service 的负载均衡
- **Container Runtime**：容器运行时（containerd、CRI-O），负责运行容器

### 1.6.2 核心工作流程

以创建一个 Deployment 为例：

1. 用户通过 `kubectl apply -f deployment.yaml` 向 API Server 发送创建请求
2. API Server 对请求进行认证（Authentication）、授权（Authorization）、准入控制（Admission Control）
3. 请求通过后，数据写入 etcd
4. Deployment Controller 监听到新的 Deployment 创建事件，创建对应的 ReplicaSet 对象并写入 etcd
5. ReplicaSet Controller 监听到新的 ReplicaSet，根据期望副本数创建 Pod 对象并写入 etcd（此时 Pod 的 `nodeName` 字段为空，状态为 `Pending`）
6. Scheduler 监听到 `nodeName` 为空的 Pod，进行调度：
   - 过滤阶段：排除不满足条件的节点（资源不足、存在污点、有亲和性约束）
   - 打分阶段：对剩余节点按算法打分排序
   - 选择得分最高的节点，更新 Pod 的 `nodeName` 字段
7. 对应 Node 上的 kubelet 监听到有 Pod 被分配到自己节点上：
   - 通过 CRI 拉取镜像
   - 通过 CRI 创建并启动容器
   - 通过 CNI 配置容器的网络
   - 上报 Pod 状态到 API Server

### 1.6.3 关键设计：控制器模型与 Watch 机制

Kubernetes 中的所有组件通过 **Watch + Informer** 机制感知资源变化，而非轮询：

- **Watch**：API Server 支持资源的流式变更通知（长连接），当资源发生变更时，推送事件给订阅者
- **Informer**：客户端的本地缓存 + 事件处理器，通过 List-Watch 机制与 API Server 同步数据，减少 API Server 压力
- **Level-Triggered**：事件是水平触发的（通知"当前状态"），而非边缘触发的（通知"发生了什么变化"），即使组件错过了一些事件，也能通过全量 List 恢复到最新状态

## 1.7 Kubernetes 生态系统

Kubernetes 是现代云原生技术栈的核心，围绕它形成了一个庞大的生态系统（CNCF Landscape）：

### 1.7.1 CNCF 全景图中的关键项目

| 分类 | 常见项目 |
|------|----------|
| **运行时** | containerd、CRI-O、gVisor、Kata Containers |
| **网络** | Flannel、Calico、Cilium（eBPF）、Weave Net |
| **存储** | Rook（Ceph）、Longhorn、OpenEBS、Portworx |
| **服务网格** | Istio、Linkerd、Consul Connect、Kuma |
| **可观测性** | Prometheus、Grafana、Jaeger、OpenTelemetry、Fluentd、Loki、Thanos、VictoriaMetrics |
| **CI/CD** | ArgoCD、FluxCD、Tekton、Drone、Jenkins X |
| **包管理** | Helm |
| **镜像仓库** | Harbor、Dragonfly（P2P 镜像分发） |
| **密钥管理** | HashiCorp Vault、Sealed Secrets、External Secrets Operator |
| **策略管理** | OPA/Gatekeeper、Kyverno |
| **安全扫描** | Trivy、Falco |
| **服务代理** | Envoy、Traefik、Nginx Ingress、APISIX Ingress |
| **备份恢复** | Velero |
| **开发工具** | Skaffold、Tilt、Telepresence、Mirrord |
| **多集群** | Karmada、KubeFed、Cluster API (CAPI) |
| **Serverless** | Knative、OpenFaaS、Fission |

### 1.7.2 常见的 K8s 发行版与托管服务

| 类型 | 名称 | 特点 |
|------|------|------|
| **本地开发** | minikube、kind、k3d、MicroK8s | 单机部署，用于本地开发和测试 |
| **轻量级** | k3s、k0s、MicroK8s | 适用于边缘计算、IoT、资源受限环境 |
| **企业级** | Rancher(RKE2)、OpenShift(OKD)、KubeSphere | 增强的多集群管理、安全、UI、DevOps 功能 |
| **云托管** | GKE（Google）、EKS（AWS）、AKS（Azure）、TKE（腾讯）、ACK（阿里） | 云厂商提供的托管 K8s 服务，免运维控制平面 |

### 1.7.3 主要云厂商的 K8s 特性对比

| 特性 | GKE | EKS | AKS |
|------|-----|-----|-----|
| **控制平面费用** | Autopilot 按 Pod 计费，Standard 免费 | $0.10/时/集群（控制平面） | 免费 |
| **自动升级** | 支持且推荐 | 需手动触发 | 自动补丁升级 |
| **Auto Scaling** | Node Auto Provisioning | Karpenter / Cluster Autoscaler | Cluster Autoscaler |
| **网络** | 原生支持 Cilium | VPC CNI + Calico 可选 | Azure CNI |
| **多集群** | Fleet（舰队管理） | — | Azure Arc |
| **GPU 支持** | 原生支持多种 GPU | 需额外配置 | 支持 GPU Node Pool |
| **Serverless** | 原生（GKE Autopilot） | EKS Fargate | Virtual Nodes (ACI) |

## 1.8 Kubernetes 的适用场景与局限性

### 1.8.1 适用场景

- **微服务架构**：K8s 天生适合微服务，提供全套服务发现、负载均衡、滚动更新、熔断降级（通过 Service Mesh 扩展）能力
- **CI/CD 流水线**：动态分配构建环境、自动化部署
- **大数据与 AI/ML**：Spark on K8s、Flink on K8s、Kubeflow、Ray on K8s
- **边缘计算**：k3s、KubeEdge 将 K8s 集群延伸到边缘节点
- **混合云与多云**：统一的 API 屏蔽底层基础设施差异，实现跨云部署
- **有状态应用**：通过 StatefulSet + Operator 模式管理数据库、消息队列、缓存等有状态服务

### 1.8.2 局限性与挑战

| 挑战 | 说明 |
|------|------|
| **学习曲线陡峭** | K8s 涉及大量概念、组件和工具，新手需要较长学习周期 |
| **运维复杂度** | 大规模集群的运维需要专业团队，涉及网络、存储、安全、升级、备份等多个领域 |
| **资源开销** | 控制平面组件本身消耗资源（etcd 的内存需求显著），对小规模应用来说"杀鸡用牛刀" |
| **状态管理复杂** | 虽然有 StatefulSet 和 Operator，但管理数据库等有状态服务仍然比无状态服务复杂得多 |
| **调试困难** | 分布式系统的调试本身就是难题，需要完善的日志、监控、追踪体系 |
| **版本升级风险** | 大版本升级时有 API 弃用、迁移等成本 |
| **网络模型理解门槛** | 扁平网络、Service、Ingress、NetworkPolicy 等模型有一定理解成本 |
| **存储的复杂性** | PV/PVC/StorageClass/CSI 的体系需要一定时间来熟悉 |

### 1.8.3 何时不应选择 Kubernetes

- 非常简单的应用（单个虚拟机或少量容器即可满足需求）
- 团队规模小且没有 K8s 运维经验（建议先用云厂商的 Serverless 或 PaaS 方案）
- 对延迟极度敏感且不能接受 K8s 网络 Overhead 的场景
- 传统的单体应用且没有微服务化计划

## 1.9 学习 Kubernetes 的路径建议

### 1.9.1 前置知识

在学习 Kubernetes 之前，建议具备以下基础：

| 知识领域 | 要求程度 | 说明 |
|----------|----------|------|
| **Linux 基础** | 必须 | 命令行操作、systemd、网络基础、文件系统 |
| **容器技术（Docker）** | 必须 | 理解镜像、容器、Dockerfile、docker-compose |
| **网络基础** | 重要 | TCP/IP、DNS、HTTP、负载均衡、iptables |
| **分布式系统基础** | 重要 | CAP 理论、共识算法、服务发现、最终一致性 |
| **YAML/JSON** | 必须 | K8s 所有资源通过 YAML 或 JSON 定义 |
| **Git** | 推荐 | GitOps 的基础 |
| **Go 语言** | 进阶 | K8s 及其生态工具多用 Go 编写，深度理解源码需要 |
| **etcd** | 进阶 | Raft 共识算法、分布式 KV 存储原理 |

### 1.9.2 学习阶段建议

**第一阶段：基础入门**
- 理解容器与 Pod 的关系
- 掌握 kubectl 常用命令
- 会创建 Deployment、Service、ConfigMap、Secret
- 理解 Pod 生命周期和探针

**第二阶段：核心能力**
- 深入理解控制器模式（Deployment、StatefulSet、DaemonSet、Job/CronJob）
- 掌握 Ingress 和 Gateway API
- 熟悉 PV/PVC/StorageClass 体系
- 理解 RBAC 和 ServiceAccount
- 能使用 Helm 管理应用

**第三阶段：进阶能力**
- 理解 Scheduler 调度机制（亲和性、污点容忍）
- 掌握 NetworkPolicy 和网络安全
- 能搭建和管理高可用集群
- 理解 CRI、CNI、CSI 接口
- 熟悉 Prometheus + Grafana 监控体系
- 掌握 EFK/ELK 日志体系

**第四阶段：专业能力**
- 理解 K8s 源码架构（API Server、Scheduler、Controller Manager、kubelet）
- 掌握 Operator 开发（Kubebuilder / Operator SDK）
- 熟悉 Service Mesh（Istio）
- 能进行大规模集群优化
- 掌握 GitOps 工作流（ArgoCD / FluxCD）
- 熟悉多集群管理

## 1.10 本系列笔记使用的环境

- Kubernetes 版本：**v1.30.x**
- 容器运行时：**containerd**
- CNI 网络插件：**Calico**
- 操作系统：**Ubuntu 22.04 LTS / Rocky Linux 9**
- 包管理工具：**Helm v3.x**
- kubectl 版本：**v1.30.x**

## 1.11 关键命令快速预览

### 1.11.1 kubectl 常用命令分类

```shell
# 集群信息
kubectl cluster-info              # 查看集群信息
kubectl get nodes                 # 查看节点
kubectl describe node <node-name> # 查看节点详情
kubectl top node                  # 查看节点资源使用

# 资源查看
kubectl get pods                  # 查看 Pod
kubectl get deployments           # 查看 Deployment
kubectl get services              # 查看 Service
kubectl get all                   # 查看所有资源
kubectl describe <resource> <name> # 查看资源详情
kubectl get <resource> -o wide    # 显示更多列
kubectl get <resource> -o yaml    # 以 YAML 格式输出
kubectl get <resource> -o json    # 以 JSON 格式输出

# 资源创建与删除
kubectl apply -f <file.yaml>      # 声明式创建/更新资源
kubectl create -f <file.yaml>     # 命令式创建资源
kubectl delete -f <file.yaml>     # 删除资源
kubectl delete <resource> <name>  # 按名称删除资源

# Pod 管理
kubectl logs <pod-name>           # 查看日志
kubectl logs -f <pod-name>        # 流式查看日志
kubectl exec -it <pod-name> -- /bin/sh  # 进入容器
kubectl port-forward <pod-name> 8080:80  # 端口转发

# 调试
kubectl get events                # 查看集群事件
kubectl describe <resource> <name> # 查看资源事件和状态
kubectl debug                     # 调试容器（v1.18+）

# 扩缩容
kubectl scale deployment/<name> --replicas=5  # 调整副本数

# 滚动更新
kubectl rollout status deployment/<name>      # 查看更新状态
kubectl rollout history deployment/<name>     # 查看更新历史
kubectl rollout undo deployment/<name>        # 回滚
```

## 1.12 补充：Kubernetes 与云原生全景

### 1.12.1 云原生定义

CNCF 对云原生的定义：

> 云原生技术有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。云原生的代表技术包括容器、服务网格、微服务、不可变基础设施和声明式 API。
>
> 这些技术能够构建容错性好、易于管理和便于观察的松耦合系统。结合可靠的自动化手段，云原生技术使工程师能够轻松地对系统作出频繁和可预测的重大变更。

### 1.12.2 Kubernetes 在云原生中的位置

Kubernetes 是云原生生态的**基石**，它扮演着"**分布式应用的操作系统**"角色：
- 向下：管理异构基础设施（裸金属、虚拟机、公有云、私有云）
- 向上：为应用提供统一的管理接口和运行环境
- 周边：通过可插拔接口与网络、存储、安全、监控、CI/CD 等生态项目集成

```
┌──────────────────────────────────────────────────┐
│                   应用层                           │
│        (微服务、Serverless、AI/ML 工作负载)         │
├──────────────────────────────────────────────────┤
│            平台层（Operator、CRD、Service Mesh）    │
├──────────────────────────────────────────────────┤
│               Kubernetes 核心                      │
│   (API Server, Scheduler, Controller Manager,     │
│    kubelet, kube-proxy, etcd, CRI/CNI/CSI)       │
├──────────────────────────────────────────────────┤
│           基础设施层（计算、网络、存储）              │
└──────────────────────────────────────────────────┘
```

### 1.12.3 云原生社区资源

| 资源 | 地址 | 说明 |
|------|------|------|
| Kubernetes 官方文档 | https://kubernetes.io/zh-cn/docs/ | 官方文档，中文翻译质量较好 |
| CNCF Landscape | https://landscape.cncf.io/ | 云原生全景图，浏览所有相关项目 |
| K8s GitHub | https://github.com/kubernetes/kubernetes | 源码仓库 |
| Kubernetes Slack | https://slack.k8s.io/ | 社区 Slack 频道 |
| CNCF 项目导航 | https://www.cncf.io/projects/ | 毕业、孵化、沙箱项目列表 |
| Artifact Hub | https://artifacthub.io/ | Helm Chart、Operator 等资源中心 |
| killercoda | https://killercoda.com/kubernetes | 在线 K8s 实验环境 |
