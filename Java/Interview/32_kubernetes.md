本篇覆盖 Kubernetes 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：架构与核心对象、网络与暴露、存储配置与安全、调度资源与弹性、运维与生态。容器运行时与镜像见 23_docker.md，Spring Boot 的优雅停机与探针配合见 10_springboot.md（4.1.2），网关/入口层的七层路由见 19_spring_cloud_gateway.md、12_nginx.md，发布流水线见 33_cicd.md。

# 1. 架构与核心对象

## 1.1 控制面与节点

### 1.1.1 K8s 架构与控制面组件职责？什么是"声明式 + 调谐循环"？

**控制面（Control Plane）**：

| 组件 | 职责 | 关键点 |
|------|------|--------|
| **kube-apiserver** | 集群唯一入口：REST API、认证/授权/准入、watch 通知 | **无状态、可多副本**；所有组件只跟它通信，不直接读 etcd |
| **etcd** | 分布式 KV（Raft），存**全部集群状态** | 3/5 节点奇数部署；**备份=备份 etcd**；性能瓶颈常在这 |
| **kube-scheduler** | 给新 Pod 选 Node（过滤 + 打分 + 绑定） | 只负责"选"，不负责"起" |
| **kube-controller-manager** | 一堆控制器的集合：Deployment/ReplicaSet/Node/Job/Endpoint… | 每个控制器跑**调谐循环**（reconcile） |
| **cloud-controller-manager** | 与云厂商对接（LB、节点、路由） | 云上才有 |

**工作节点（Node）**：**kubelet**（接收 PodSpec、调 CRI 起容器、上报状态、执行探针）、**kube-proxy**（维护 Service 的 iptables/IPVS 规则，或用 eBPF 替代）、**容器运行时**（containerd/CRI-O，通过 **CRI** 接口）。

**"声明式 + 调谐"是 K8s 的灵魂**：你提交的是**期望状态**（`replicas: 3`），控制器**不断对比"当前状态"与"期望状态"并采取动作**——不是"执行一次部署脚本"，而是**永不停歇的纠偏**（Pod 挂了重建、副本少了补、镜像变了滚动）。这解释了 K8s 的自愈能力，也解释了"为什么 `kubectl apply` 之后还要看 `rollout status`"（提交成功 ≠ 已达到期望状态）。

**易错**：说"kubelet 调度 Pod"（调度是 scheduler，kubelet 只执行）；"apiserver 直接写 etcd 之外还缓存状态"（apiserver 有无状态缓存但真相在 etcd）；把 controller-manager 当一个控制器（是一组）；忘记 **CRI** 的存在（"K8s 1.24 移除 dockershim"指的是不再用 dockerd 当运行时，镜像格式仍是 OCI，23_docker.md 1.2.1）。

### 1.1.2 一个 Pod 从提交到运行经历了什么？

**完整链路（按组件顺序，排障时按段定位）**：

```text
① kubectl apply → apiserver：认证 → 授权(RBAC) → 准入(Mutating→Validating) → 写入 etcd
② Deployment Controller 看到新 ReplicaSet → 创建 Pod 对象（phase=Pending，未绑定节点）
③ Scheduler 监听到未绑定 Pod：过滤(资源/污点/亲和) → 打分 → 绑定 Node（写回 apiserver）
④ 目标 Node 的 kubelet watch 到"分配给我的 Pod"：
   → 拉镜像（ImagePull，失败即 ImagePullBackOff）
   → 调 CRI 创建 sandbox（pause 容器，建网络命名空间，CNI 分配 IP）
   → 启动 init 容器（按序）→ 启动业务容器
⑤ 探针开始工作：startup → readiness（通过才进 Endpoints）→ liveness 持续检查
⑥ Endpoints/EndpointSlice 控制器把就绪 Pod 加入 Service 后端；kube-proxy/eBPF 更新转发规则
```

**四个"卡住点"对应的排查**：②③ 之间卡住=**Pending**（资源不足/污点/亲和/PVC）；④ 拉镜像失败=**ImagePullBackOff**（镜像名/凭据/网络）；容器反复退出=**CrashLoopBackOff**（应用/探针/配置/OOM）；⑤ 就绪不过=**Service 无 Endpoints**（readiness 失败/selector 不匹配）。

**易错**：以为"调度成功=启动成功"（还有拉镜像/CNI/挂载/启动）；忽略 **admission webhook** 的拦截（"apply 报错但 YAML 没问题"——看 apiserver 返回的准入错误）；Pod 状态是**最终一致**的（watch 有延迟，别用"立刻 get"判断失败）。

## 1.2 工作负载

### 1.2.1 Pod 的模型与生命周期是什么？pause、init、sidecar 各是什么？

**Pod = 最小调度单元**：一个或多个容器**共享网络命名空间（同一 IP、localhost 互通）、IPC、挂载的 Volume**，但**文件系统与进程空间默认隔离**（同一 Pod 内 PID namespace 默认不共享，可配 `shareProcessNamespace`）。

**pause（infra）容器**：每个 Pod 一个"什么都不做"的基础容器，**持有网络/IPC 命名空间**——业务容器加入它的命名空间，业务容器崩溃重启时**网络命名空间不丢**（IP 不变）。这是"Pod 内容器共享网络"的实现基础。

**init 容器**：在主容器**之前按序执行、必须成功退出**——用于初始化（等依赖就绪、生成配置、下载资源）；**失败会按 restartPolicy 重试**（Pod 停在 Init:CrashLoopBackOff）；**不支持探针**（除 exec 外）。

**sidecar（边车）**：与主容器**并行**、共享网络的辅助容器（日志采集、代理、配置同步）；**传统 sidecar 的问题**：Job 里 sidecar 不退出会导致 Job 永不完成；**原生 sidecar（1.29 引入、后续版本稳定）**：把 sidecar 写成 `initContainers` 里的容器并设 `restartPolicy: Always`——**启动顺序在主容器前、退出时随 Pod 生命周期管理**，解决"sidecar 生命周期"经典难题（版本与行为以官方为准）。

**生命周期**：**Pod phase**（Pending/Running/Succeeded/Failed/Unknown）是"阶段"，**容器状态**（Waiting/Running/Terminated）是"细节"；`restartPolicy`（Always/OnFailure/Never）决定容器退出后是否重启；**Pod 一旦绑定 Node 就不会"迁移"**（Node 挂了由控制器创建新 Pod，不是把旧 Pod 搬走）。

**易错**：把"Pod 重启"当"容器重启"（容器重启 IP 不变；Pod 重建 IP 会变）；init 容器里写"长期运行的服务"（应该用 sidecar）；以为 sidecar 一定在主容器之后启动（原生 sidecar 反了）；`shareProcessNamespace` 与 `localhost` 混淆。

### 1.2.2 Deployment、ReplicaSet、StatefulSet、DaemonSet、Job 怎么选？

| 工作负载 | 保证什么 | 适用 | 关键特性 |
|---------|---------|------|---------|
| **Deployment** | 副本数 + **滚动更新/回滚** | 无状态服务（Web/API） | 管理 ReplicaSet；`maxSurge/maxUnavailable`；`rollout undo` |
| **ReplicaSet** | 指定副本数始终运行 | 一般由 Deployment 管理（**别直接建**） | 只保证数量，不管更新策略 |
| **StatefulSet** | **稳定标识 + 稳定存储 + 有序** | 数据库、MQ、ZK/etcd 类 | Pod 名 `app-0..N`、**Headless Service DNS**、`volumeClaimTemplates` 每 Pod 独立 PVC、按序启停 |
| **DaemonSet** | **每个（或部分）Node 一个 Pod** | 日志采集、监控 agent、CNI、kube-proxy | 新节点自动加；`nodeSelector/affinity` 圈定范围 |
| **Job / CronJob** | 运行到**成功完成** / 定时 | 批处理、迁移、备份 | `completions/parallelism`、`backoffLimit`、`activeDeadlineSeconds`；CronJob 的并发策略 |

**StatefulSet 的三个"稳定"（有状态应用的核心）**：**稳定网络标识**（`mysql-0.mysql.ns.svc` 的 DNS 固定，重建也不变）、**稳定存储**（每个序号绑定自己的 PVC，Pod 漂移后仍挂回）、**有序**（默认按 0→N 创建、逆序删除/更新——配合主从/选主）。**代价**：扩缩容慢、故障恢复要等序号、不如 Deployment 灵活。

**易错**：用 Deployment 跑数据库（副本各自随机名/共享 PVC，主从语义无法保证）；StatefulSet 没配 Headless Service（DNS 不稳定）；DaemonSet 忘了容忍控制面污点（master 上不跑，但日志 agent 可能需要）；Job 的 `restartPolicy` 只能是 OnFailure/Never（不能 Always）。

# 2. 网络与暴露

## 2.1 Service 与 Ingress

### 2.1.1 Service 有哪几种类型？ClusterIP 的流量到底怎么转发的？

**四种类型**：

| 类型 | 暴露范围 | 实现 |
|------|---------|------|
| **ClusterIP**（默认） | 集群内 | 虚拟 IP + kube-proxy/eBPF 转发到 Pod |
| **NodePort** | 每个 Node 的固定端口（默认 30000-32767） | 在 ClusterIP 基础上，每 Node 开端口 → 转发 |
| **LoadBalancer** | 云厂商 LB 分配外部 IP | 云控制器创建 LB → 指向 NodePort/ Pod |
| **ExternalName** | 集群内访问外部域名 | DNS CNAME，无代理/转发 |

**ClusterIP 的"虚拟 IP"不是真实网卡**：`10.96.x.x` 只存在于转发规则里——**没有任何主机拥有它**；访问它时，内核在 **PREROUTING/OUTPUT** 阶段匹配规则，**DNAT 成某个 Pod IP**（kube-proxy 维护 iptables 链），回包再 SNAT/conntrack 还原。**IPVS 模式**用哈希表替代 iptables 链（大规模 Service 下性能更好，支持 rr/lc 等调度算法）；**eBPF（Cilium 等）** 直接在内核程序里做转发，可完全替代 kube-proxy。

**Headless Service（`clusterIP: None`）**：不做 VIP/负载均衡，DNS 直接返回**所有 Pod IP**——StatefulSet 的稳定 DNS、客户端自己做负载均衡/选主时用（2.1.3）。

**Endpoints/EndpointSlice**：Service 的 selector 选出 Pod → 控制器写入 Endpoints（新版本用 EndpointSlice 分片）——**"Service 没后端"先看 Endpoints 是否为空**（selector 不匹配/readiness 不过/Pod 没起）。

**易错**：以为 ClusterIP 能 ping 通（它是虚拟 IP，ICMP 未必响应，TCP 才行）；NodePort 当生产入口（端口固定、无七层能力、Node IP 暴露）；LoadBalancer 在裸机不可用（需要 MetalLB 类方案）；Service 的 `targetPort` 与容器 `containerPort` 混淆（前者是 Pod 端口，后者只是声明）。

### 2.1.2 Ingress 与 Gateway API 有什么区别？Ingress Controller 怎么工作？

**Ingress**：**七层（HTTP/HTTPS）路由规则**——按 **host + path** 把请求转发到不同 Service，支持 TLS 终止；**只是规则，不是实现**——必须有 **Ingress Controller**（nginx-ingress、Traefik、云 ALB Ingress）持续 watch Ingress 资源并**生成自己的配置**（如 nginx.conf + reload），数据面才是真正干活的代理。

**与 Service 的分工**：Service 是**四层**（TCP/UDP，按 IP:Port 转发、不做 HTTP 语义）；Ingress 是**七层**（看 Host/Path/Header，做 TLS、重写、灰度）——典型链路：`外部 → Ingress Controller（LB/NodePort）→ Ingress 规则 → Service → Pod`。

**Gateway API（Ingress 的继任者，必答的现代点）**：Ingress API 多年冻结、靠 annotations 塞厂商特性；Gateway API 用**角色分离**的模型——**GatewayClass**（基础设施提供商）→ **Gateway**（集群运维：监听端口/TLS）→ **HTTPRoute/GRPCRoute**（应用团队：路由规则），表达力更强（Header 匹配、流量拆分、跨命名空间引用），是 K8s 网络入口的未来方向（19_spring_cloud_gateway.md 4.2.1 的"入口归平台"同源）。

**易错**：装了 Ingress 资源但没装 Controller（规则不生效）；把 Ingress 当"负载均衡器"（它本身不实现转发）；多个 Controller 争抢同一 IngressClass（`ingressClassName` 要指定）；TLS 证书放 Secret 但没配 `tls` 段。

### 2.1.3 K8s 的服务发现与 DNS 怎么工作？Headless 与普通 Service 的 DNS 有什么不同？

**CoreDNS**：集群 DNS（以 Deployment 跑在 `kube-system`），为 Service/Pod 提供记录：
- 普通 Service：`<svc>.<ns>.svc.cluster.local` → **ClusterIP**（A 记录）；
- **Headless Service**：同一名字 → **每个就绪 Pod 的 IP 列表**（A 记录多条），StatefulSet 还能拿到 `<pod>.<svc>.<ns>.svc.cluster.local` 的**每 Pod 稳定域名**；
- Pod：`<pod-ip-dashed>.<ns>.pod.cluster.local`（少用）。

**客户端解析细节（高频坑）**：Pod 的 `/etc/resolv.conf` 有 `search` 域与 **`ndots:5`**——短域名（如 `order-service`）会**先尝试多个 search 后缀**（可能多次 DNS 查询）才走绝对解析；**跨命名空间**要写全（`order-service.payment.svc.cluster.local`）或用 `ns.svc`；**DNS 超时/重试**（`timeout:2 attempts:5`）会放大延迟——"服务间调用偶发慢"先查 DNS（`ndots` 调小、用 FQDN、NodeLocal DNSCache）。

**服务发现的两条路**：**DNS**（应用按名字连）与 **环境变量**（Pod 启动时注入 `XXX_SERVICE_HOST`，只对先后顺序敏感、不推荐）；**Service 的 ClusterIP 负载均衡在四层**，应用层（gRPC 长连接）容易"连上就不换"——需要客户端 LB/服务网格/Headless + 客户端解析（19_spring_cloud_gateway.md、18_feign.md 的 LB 语义在 K8s 的对应物）。

**易错**：用 Headless 却期望 VIP 负载均衡（它给的是 IP 列表）；Service 名与 Namespace 写错（404/解析失败）；CoreDNS 副本太少（DNS 成为瓶颈）；忘记 `cluster.local` 可配（自定义域名）。

## 2.2 网络模型

### 2.2.1 K8s 网络模型是什么？CNI 与跨节点通信怎么走？

**三条模型约束**：① **每个 Pod 有唯一 IP**；② **Pod 之间直接通信，不做 NAT**（源 IP 保持）；③ **Node 与 Pod、Pod 与外部**按各自网络策略可达。这要求一个**扁平的三层网络**——由 **CNI 插件**实现（K8s 只定义接口，不实现网络）。

**CNI 的两种实现路线**：
- **Overlay（隧道）**：Pod 包封装在宿主网络里传（VXLAN/IPIP），跨节点像"同一张虚拟二层网"——Flannel（VXLAN）、Calico（IPIP 模式）；**优点**：不依赖底层网络，**缺点**：封装开销、MTU 要减（常见 1450）、排障多一层；
- **Underlay/路由**：把 Pod 网段通过 **BGP/路由**通告给物理网络（Calico BGP 模式、Cilium native routing）——**无封装、性能好**，但要求网络设备配合。

**数据路径**：同节点 Pod → veth pair → 网桥/路由 → 对端 veth；跨节点 → 节点路由/隧道 → 对端节点 → Pod；**Service 流量**再由 kube-proxy/eBPF 做 DNAT（2.1.1）。**IPAM**：CNI 给 Pod 分配 IP（按节点分配网段，避免冲突）。

**易错**：MTU 不匹配（大包被丢，表现为"小请求正常、大响应卡住"——`ping -M do -s` 可测）；Pod 网段与公司内网冲突（跨网访问黑洞）；以为"Pod IP 可以直接对外"（默认不可路由，要 Ingress/LB）；CNI 与 NetworkPolicy 支持不是一回事（Flannel 默认不支持策略，2.2.2）。

### 2.2.2 NetworkPolicy 与 Service Mesh 各解决什么？sidecar 与 ambient 怎么选？

**NetworkPolicy（L3/L4 网络策略）**：按 **Pod 标签/命名空间/IP 段** 控制"谁能访问谁、哪个端口"——**默认全通，声明后变白名单**（**默认拒绝**要显式写 `podSelector: {}` + 空 ingress/egress）；**前提**：CNI 必须支持（Calico/Cilium 支持，Flannel 不支持）——"写了策略没生效"先查 CNI。

**Service Mesh（L7 治理）**：把**服务间通信的横切能力**（mTLS、重试/超时、熔断、流量拆分、灰度、遥测）从应用代码下沉到**数据面代理**：
- **Sidecar 模式**（Istio/Linkerd 经典）：每个 Pod 注入一个 Envoy，**劫持所有进出流量**（iptables 重定向）——能力全，但**每 Pod 一个代理**（资源/延迟/升级成本）；
- **Ambient 模式**（Istio 新架构）：**节点级 ztunnel** 处理 L4（mTLS/遥测），需要 L7 时再用 **waypoint 代理**（按命名空间/服务）——**不用每 Pod 注入**，降低资源与运维成本（版本与成熟度以官方为准）。

**两者分工**："**NetworkPolicy 管'通不通'（L3/L4 白名单），Mesh 管'怎么通'（L7 路由/安全/观测）**"；**没有 Mesh 也能做 L7**（Ingress/Gateway API + 应用内治理，19_spring_cloud_gateway.md 的取舍），Mesh 的代价是"多一层代理 + 一套控制面"——**团队规模与多语言程度**决定值不值。

**易错**：以为 NetworkPolicy 是"防火墙全功能"（无 L7、无域名、默认拒绝要自己配）；sidecar 注入后应用探针/出网异常（端口/排除列表）；ambient 模式下 L7 策略要 waypoint（不是自动全有）；Mesh 与 Ingress 的职责重叠（入口用 Gateway，东西向用 Mesh）。

# 3. 存储、配置与安全

## 3.1 存储

### 3.1.1 Volume、emptyDir、hostPath、PV、PVC、StorageClass 是什么关系？

**从"Pod 内"到"集群级"的四层**：

| 对象 | 生命周期 | 用途 | 注意 |
|------|---------|------|------|
| **emptyDir** | **随 Pod**（Pod 删除即清空） | 同 Pod 内容器共享临时数据、缓存 | 可指定 `medium: Memory`（tmpfs，计入内存）；节点磁盘满会影响 |
| **hostPath** | 随节点 | 访问宿主机目录/设备（日志、docker.sock、CNI） | **安全风险**（逃逸面）、Pod 漂移后数据不可达——生产慎用 |
| **PV（PersistentVolume）** | **集群级**资源 | 一块真实存储（云盘/NFS/本地盘） | 由管理员静态创建或 StorageClass 动态供给 |
| **PVC（PersistentVolumeClaim）** | **命名空间级**请求 | "我要 10Gi、读写一次" | 绑定到某个 PV；Pod 通过 `persistentVolumeClaim` 引用 |
| **StorageClass** | 集群级 | **动态供给模板**（CSI 驱动 + 参数） | PVC 指定 SC → 自动创建 PV（云盘/NFS/本地盘） |

**三条关键机制**：① **绑定**：PVC 按容量/访问模式/StorageClass 匹配 PV（或触发动态创建），**绑定后独占**（RWO 的云盘不能跨节点同时挂载）；② **访问模式**：`RWO`（单节点读写）、`ROX`（多节点只读）、`RWX`（多节点读写，通常要 NFS/CephFS 类共享存储）、`RWOP`（单 Pod）；③ **回收策略**：`Delete`（删 PVC 连存储一起删）、`Retain`（保留数据，手工清理）——**生产数据库类必须 `Retain`**（误删 PVC 不丢盘）。

**CSI（Container Storage Interface）**：存储厂商实现 CSI 驱动（挂载/扩容/快照），K8s 通过标准接口调用——"换云厂商只换 StorageClass/驱动"。

**易错**：以为 PVC 删了数据就没了（看回收策略）；RWO 用在多副本（第二个 Pod 挂不上，卡在 ContainerCreating）；本地盘用 `hostPath` 冒充持久化（节点故障数据丢，应该用 local PV + 亲和）；忘记 `volumeBindingMode: WaitForFirstConsumer`（云盘在错误可用区创建，Pod 调度不上）。

### 3.1.2 StatefulSet 的稳定标识与存储怎么配合？扩缩容与更新有什么坑？

**三件套配合**：
1. **Headless Service**（`clusterIP: None`）给每个 Pod 稳定 DNS：`<pod>.<svc>.<ns>.svc.cluster.local`；
2. **Pod 名带序号**（`mysql-0/1/2`）——重建后名字不变，DNS 不变；
3. **`volumeClaimTemplates`**：每个 Pod 自动创建**独立 PVC**（`data-mysql-0` 等）——Pod 漂移到别的节点，**仍挂回自己的盘**（配合 local PV 的节点亲和/云盘的区域约束）。

**扩缩容**：默认**按序号顺序**（0→N 创建、N→0 删除），保证"先有主再有从"；`podManagementPolicy: Parallel` 可并行（适合无启动顺序要求的集群，如纯分片）；**缩容不会自动删 PVC**（数据保留，要手工清理——这是"特性"也是"账单刺客"）。

**更新的坑**：`updateStrategy: RollingUpdate` 默认**逆序**逐个更新（先更新最大序号）——对有主从的集群**可能先重启从节点**（通常安全），但**主节点最后更新**意味着"主切换"发生在最后；`partition` 字段可做**金丝雀**（只更新序号 ≥ partition 的 Pod）。**StatefulSet 不保证"应用层就绪"**——只保证"Pod Ready"（探针要自己配好），主从切换/数据同步要应用自己处理（或用 Operator，5.2.1）。

**易错**：用 Deployment 跑有状态应用（无稳定 DNS/PVC，主从乱套）；StatefulSet 的 Service 不是 Headless（DNS 不解析到 Pod）；缩容后 PVC 残留导致"重新扩容挂到旧数据"（有意为之但要清楚）；跨可用区的云盘与 Pod 调度冲突（用 `topologySpread` + `WaitForFirstConsumer`）。

## 3.2 配置与权限

### 3.2.1 ConfigMap 与 Secret 怎么注入？"热更新"为什么经常不生效？

**注入方式**：**环境变量**（`envFrom`/`valueFrom`）与**卷挂载**（每个 key 一个文件）；**环境变量在容器启动时固化**——改了 ConfigMap **必须重启 Pod** 才生效；**卷挂载会由 kubelet 周期性同步更新**（分钟级），但**应用不一定会重新读文件**（要应用支持 reload，如 Spring Cloud 的 `@RefreshScope`/配置中心，17_nacos.md）；**`subPath` 挂载不会更新**（kubelet 的已知限制）——"挂了 ConfigMap 但改了不生效"的第一嫌疑。

**Secret 的真相**：**base64 只是编码不是加密**（`kubectl get secret -o yaml` 就能解）；生产要：**etcd 静态加密**（EncryptionConfiguration）、**RBAC 限制读取**（能读 Secret 就能拿凭据）、**外部密钥管理**（Vault/External Secrets Operator/云 KMS）、**镜像拉取凭据**（imagePullSecrets）、**轮换**（过期/泄露时能换）。**Secret 类型**：Opaque（通用）、`kubernetes.io/dockerconfigjson`（拉镜像）、`kubernetes.io/tls`（证书）。

**最佳实践**：**配置与镜像分离**（12-factor）；**不可变 ConfigMap**（`immutable: true`，防止误改+提升性能）；**版本化配置**（名字带 hash，滚动发布时"新配置新 Pod"）；敏感配置**不要进环境变量**（进程环境可被 dump/子进程继承，优先文件挂载 + 严格权限）。

**易错**：改 ConfigMap 以为应用自动生效（环境变量不会、文件要应用重读）；`subPath` 热更新失效；Secret 当"加密存储"（base64 不是）；把 Secret 挂成 777 文件（权限过宽）；Git 里提交明文 Secret（SealedSecrets/SOPS 也要配合密钥管理）。

### 3.2.2 ServiceAccount、RBAC 与准入控制怎么落地？

**ServiceAccount（SA）**：Pod 的**身份**——挂载短期 token（Projected Volume，1.24+ 默认不再用长期 Secret token），用于访问 apiserver；**不同应用用不同 SA**（别全用 default），并**关闭不需要的 automount**（`automountServiceAccountToken: false`）。

**RBAC 四件套**：**Role/ClusterRole**（权限集合：apiGroups/resources/verbs）+ **RoleBinding/ClusterRoleBinding**（把角色绑到 subject：User/Group/ServiceAccount）；**最小权限**：只给需要的资源与动词（`get/list/watch` 与 `create/delete` 分开），**避免 `cluster-admin` 滥用**；`kubectl auth can-i` 验证。

**准入控制（Admission）**：apiserver 在"认证→授权"之后、写 etcd 之前执行——**Mutating**（改对象：注入 sidecar、加默认 resources）→ **Validating**（拒绝：策略校验）；**PodSecurity Admission**（内置三档：privileged/baseline/restricted，按命名空间打标签）替代已废弃的 PSP；**策略引擎**（OPA Gatekeeper/Kyverno）做"禁止 latest 镜像/必须设 limits/禁止特权"等组织策略；**审计日志**记录"谁改了什么"。

**安全上下文（securityContext）**：`runAsNonRoot`、`readOnlyRootFilesystem`、`allowPrivilegeEscalation: false`、`capabilities.drop: [ALL]`、`seccompProfile: RuntimeDefault`——**容器安全的四件套**（23_docker.md 5.2.2 在 K8s 的落地）。

**易错**：给 SA 绑 `cluster-admin`（等于集群 root）；只配 Role 不配 Binding（权限不生效）；以为 PSP 还在（1.25 移除，用 PSA）；准入 webhook 挂了导致"所有 Pod 建不出来"（要配 failurePolicy 与超时，避免自锁）。

# 4. 调度、资源与弹性

## 4.1 调度与资源

### 4.1.1 requests 与 limits 有什么区别？QoS 与驱逐怎么联动？

**requests**：调度依据 + **CPU 份额保障**（节点资源紧张时按 requests 比例分 CPU）；**limits**：运行上限——**CPU 超限=被节流（throttle，变慢不杀）**，**内存超限=被 OOMKill（杀容器）**。

**QoS 三档（由 requests/limits 的组合决定）**：

| QoS | 条件 | 驱逐优先级 |
|-----|------|-----------|
| **Guaranteed** | 所有容器 CPU/内存的 requests = limits | 最后被驱逐 |
| **Burstable** | 设置了 requests，但不等于 limits | 中间 |
| **BestEffort** | 完全没设 requests/limits | **最先被驱逐** |

**节点压力驱逐（Eviction）**：节点内存/磁盘/PID 达到阈值（`memory.available<100Mi` 等）时，kubelet 按 **QoS + Priority** 驱逐 Pod；**软驱逐**有宽限期、**硬驱逐**立即——"Pod 莫名 Evicted"先看节点压力与 QoS（`kubectl describe node` 的 Conditions/Allocated resources）。

**工程口径**：**内存 requests=limits**（Guaranteed，避免被 OOM/驱逐误伤）；**CPU requests 按稳态、limits 适当放宽**（CPU 可压缩，限太死会 throttle 导致延迟毛刺——30_linux.md 的 CFS 节流同源）；**所有 Pod 都设 requests**（不设=调度瞎猜，节点超卖）；**LimitRange/ResourceQuota** 在命名空间层兜底（防止"一个 Pod 吃满"）。

**易错**：limits 设得比 requests 小（非法/被拒）；只设 limits 不设 requests（调度按 limits 还是 0？K8s 默认 requests=limits，但语义容易误解）；CPU limits 设很低导致 GC/JIT 线程被节流（2_jvm.md/30_linux.md 的容器感知）；节点超卖导致"调度成功但跑不动"。

### 4.1.2 调度策略有哪些？亲和、污点、拓扑分布怎么组合？

**四类约束**：
1. **nodeSelector/nodeAffinity**：按**节点标签**选（`requiredDuringScheduling` 硬约束 / `preferred` 软约束）——"必须 GPU/必须在某可用区"；
2. **podAffinity/podAntiAffinity**：按**其他 Pod 的标签**选（`topologyKey` 定义"同一域"）——"和缓存部署在一起"（亲和）/"副本不要在同一节点"（反亲和）；
3. **Taints/Tolerations**：**节点主动排斥**（`NoSchedule`/`PreferNoSchedule`/`NoExecute`），Pod 声明容忍才能上去——专用节点（GPU/数据库）、控制面隔离、故障节点驱逐（`NoExecute` 会把不容忍的 Pod 赶走）；
4. **topologySpreadConstraints**：按拓扑域（zone/host）**均匀打散**（`maxSkew`/`whenUnsatisfiable`）——多可用区高可用的现代首选（比 podAntiAffinity 更精细）。

**调度流程**：**过滤（Filter）**（资源/污点/亲和/端口）→ **打分（Score）**（资源均衡、拓扑、镜像本地性）→ **绑定（Bind）**；`PriorityClass` 影响抢占（高优先级可驱逐低优先级 Pod）。

**组合范式**：**多 AZ 高可用** = `topologySpreadConstraints`（跨 zone 均匀）+ `podAntiAffinity`（同 zone 内不堆同副本）+ 节点池按 zone 打标签；**专用节点** = 污点 + 容忍 + nodeAffinity；**大促扩容** = 节点池 + Cluster Autoscaler + 优先级（5.2 的弹性链路）。

**易错**：反亲和用了 `required` 但副本数 > 节点数（永远 Pending）；`topologyKey` 写错（如用 hostname 但节点没打标签）；污点忘了容忍（Pod 全 Pending，`describe pod` 的 Events 会直说）；`NoExecute` 与 tolerationSeconds 的语义混淆。

## 4.2 弹性与可用性

### 4.2.1 HPA、VPA、KEDA、Cluster Autoscaler 怎么分工？

| 组件 | 管什么 | 依据 | 关键点 |
|------|--------|------|--------|
| **HPA** | **Pod 副本数**（水平） | CPU/内存/自定义/外部指标 | 需要 metrics-server（或 Prometheus Adapter）；`behavior` 控制扩缩速度与冷却（防抖动）；**扩容快、缩容慢** |
| **VPA** | **单 Pod 的 requests/limits**（垂直） | 历史用量推荐 | `recommend` 模式只给建议、`auto` 会**重建 Pod**；与 HPA 同指标会打架（别同时用 CPU） |
| **KEDA** | **事件驱动的副本数**（含 **scale to zero**） | 队列长度/Kafka lag/HTTP QPS 等 | 补 HPA 的"无流量缩到 0"；适合消费者/定时任务 |
| **Cluster Autoscaler / Karpenter** | **节点数**（集群容量） | Pending Pod 的资源需求 | Pod 扩了但节点没容量 → 加节点；**缩节点前要能安全驱逐**（PDB/4.2.3） |

**联动链路**：流量涨 → HPA 扩 Pod → 节点资源不足 → Pod Pending → **CA/Karpenter 加节点** → 调度成功；流量降 → HPA 缩 Pod → 节点利用率低 → CA 缩节点。**HPA 的指标口径**：CPU 利用率是"**实际/requests**"（requests 设太小会导致"看起来利用率高"而频繁扩容）；自定义指标（QPS/队列）更贴近业务，但要防止"指标抖动→扩缩震荡"（`behavior` 的 stabilizationWindow）。

**易错**：HPA 目标利用率按 limits 算（是按 requests）；没有 metrics-server（HPA 一直 Unknown）；VPA 与 HPA 同时改 CPU（冲突）；CA 缩节点把"有本地盘/PVC"的 Pod 赶走（不能缩）；KEDA 缩到 0 后冷启动延迟（业务可接受性）。

### 4.2.2 三种探针怎么配？优雅下线为什么经常"仍有 502"？

**三种探针**：**startup**（启动成功前**暂停 liveness**，慢启动应用必配）、**readiness**（失败则**从 Endpoints 摘除**，不再接流量）、**liveness**（失败则**重启容器**——配太激进会把"启动慢/下游慢"变成"无限重启"）。探测方式：httpGet/tcpSocket/exec/grpc；参数：`initialDelaySeconds/periodSeconds/timeoutSeconds/failureThreshold/successThreshold`。

**优雅下线的完整时序（502 的根源）**：
```text
删除 Pod（滚动更新/缩容）
 ① Pod 进入 Terminating，同时从 Endpoints 摘除（异步！）
 ② kubelet 执行 preStop hook（如 sleep 5~10s 或发信号）
 ③ 发 SIGTERM 给容器 → 应用开始优雅停机（停止接新请求、处理完在途请求）
 ④ terminationGracePeriodSeconds（默认 30s）到点 → SIGKILL
```
**502 的根因**：**"从 Endpoints 摘除"与"容器收到 SIGTERM"是并行的**——kube-proxy/Ingress 的规则更新有延迟，如果应用收到 SIGTERM 立刻停止监听，**仍有请求被转发过来** → 502。**标准解法**：`preStop: sleep 5~10s`（等规则传播完再停），`terminationGracePeriodSeconds` > preStop + 应用停机时间（Spring 的 `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase`，10_springboot.md 4.1.2）；Ingress/网关的优雅摘除（12_nginx.md 的 upstream 摘除时序）。

**易错**：liveness 探针打业务接口（下游慢导致重启风暴）；readiness 与 liveness 用同一个端点（分不清"该摘流量"还是"该重启"）；preStop 里做长时间操作（超过 grace period 被 SIGKILL）；探针 `timeoutSeconds` 大于 `periodSeconds`（行为混乱）。

### 4.2.3 怎么用 PDB、多副本、反亲和保证可用性？

**四个层次**：
1. **副本数**：无状态服务至少 2~3 副本（跨节点），数据库按副本/选主模型（StatefulSet）；
2. **拓扑打散**：`topologySpreadConstraints`（跨 zone/节点）+ `podAntiAffinity`（同应用副本不共节点）——**单节点/单 AZ 故障不团灭**；
3. **PDB（PodDisruptionBudget）**：`minAvailable`/`maxUnavailable` 限制**自愿中断**（`kubectl drain`、节点升级、CA 缩节点、descheduler）时"一次能赶走多少 Pod"——**没有 PDB，滚动维护可能把副本全赶走**；PDB 只防"自愿中断"，**不防节点故障/驱逐**（那是 4.1.1 的 QoS/多副本）；
4. **就绪门（readinessGates）与优雅摘流**（4.2.2）：确保"新 Pod 真正可用才接流量"。

**SLO 视角**：可用性 = 副本数 × 拓扑分布 × 故障检测/恢复速度；**PDB + drain 演练**才算"高可用被验证过"（33_cicd.md 的发布演练、30_linux.md 的故障演练同款）。

**易错**：PDB 设 `minAvailable: 100%`（任何 drain 都卡住，升级做不了）；只设反亲和不设 PDB（维护时仍可能全下）；多副本但都在同一节点（反亲和/拓扑没配）；把 PDB 当"防故障"（它只管自愿中断）。

# 5. 运维与生态

## 5.1 发布与运维

### 5.1.1 滚动更新、蓝绿、金丝雀怎么选？回滚怎么做？

**三种发布策略**：

| 策略 | 机制 | 资源开销 | 流量风险 | 适用 |
|------|------|---------|---------|------|
| **滚动更新（RollingUpdate）** | 逐批替换 Pod（`maxSurge` 多起几个、`maxUnavailable` 允许多少不可用） | 低（+ surge） | 新旧版本**短暂共存** | 无状态服务默认 |
| **蓝绿（Blue-Green）** | 两套完整环境（蓝=旧、绿=新），**切换 Service selector/入口** | **双倍** | 切换瞬间全量；回滚=切回 | 要"秒级回滚"、版本差异大 |
| **金丝雀（Canary）** | 新版本先接**小比例/特定用户**流量，观察指标后逐步放量 | 低~中 | 可控（按比例/Header） | 高风险变更、需要真实流量验证 |

**滚动更新的关键参数**：`maxSurge`（额外最多几个，可百分比）、`maxUnavailable`（最多几个不可用）、`minReadySeconds`（新 Pod 稳定多久算就绪）、`progressDeadlineSeconds`（超时判定失败）；**回滚**：`kubectl rollout undo deployment/x [--to-revision=N]`、`rollout history/status`、`revisionHistoryLimit` 保留历史版本。**前提**：**新版本必须能兼容旧版本的流量与数据**（readiness 探针、向后兼容的 DB 迁移——"先加字段后用、先双写后切读"的发布纪律，19_spring_cloud_gateway.md 3.2.1 同源）。

**金丝雀的落地**：简单版=两个 Deployment + Service 按比例分（或用 Ingress 权重/Header）；进阶=**Argo Rollouts / Flagger**（自动按指标推进/回滚）+ **Service Mesh/Gateway** 做流量切分；**没有流量切分能力时，"金丝雀"退化成"蓝绿+人工观察"**。

**回滚的三个层次**：**应用镜像**（rollout undo）；**配置**（ConfigMap 版本化/配置中心回滚，17_nacos.md）；**数据**（DB 变更**不可回滚**——只能向前兼容/双写/补偿，发布前评审迁移脚本）。**"能回滚"是发布设计的一部分，不是事后操作**。

**易错**：滚动更新与数据库迁移同时上（新旧代码不兼容字段）；`maxUnavailable: 0` + 副本少（更新卡住）；蓝绿切 Service 但**忘了切 Ingress/外部 LB**（流量还在旧入口）；金丝雀只看错误率不看延迟/业务指标（"没报错但很慢"照样事故）。

### 5.1.2 Pod 起不来怎么排查？给一张"状态 → 根因"速查表。

| 状态/事件 | 常见根因 | 排查动作 |
|-----------|---------|---------|
| **Pending** | 资源不足（requests 太大）、污点不容忍、亲和/反亲和不满足、PVC 未绑定、配额超限 | `kubectl describe pod` 看 Events（"0/3 nodes available: insufficient cpu"等）；`describe node` 看可分配资源；查 PVC/StorageClass |
| **ImagePullBackOff / ErrImagePull** | 镜像名/tag 错、私有仓库无凭据（imagePullSecrets）、网络/DNS、仓库限流 | `describe pod` 看具体错误；`docker pull` 同镜像验证；检查 Secret 与 `imagePullSecrets` |
| **CrashLoopBackOff** | 应用启动失败（配置/依赖/端口）、探针太激进、OOMKilled、命令/入口错 | `kubectl logs <pod> --previous`（**上一次崩溃的日志**）；`describe` 看 Exit Code（137=OOM/被杀，1=应用错误）；核对 ConfigMap/Secret/探针 |
| **CreateContainerConfigError** | 引用的 ConfigMap/Secret/key 不存在 | 核对名字/命名空间/key |
| **ContainerCreating（长时间）** | 镜像拉取慢、**卷挂载失败**（RWO 被占用/跨区）、CNI 分配 IP 失败 | `describe` Events；查 PVC 绑定节点与 Pod 调度节点；查 CNI 日志 |
| **Evicted** | 节点内存/磁盘/PID 压力，按 QoS 驱逐 | `describe node` 的 Conditions；`kubectl get pod -o yaml` 看 `status.reason`；调 requests/清理节点 |
| **Terminating 卡住** | finalizer 未清、kubelet 失联、grace period 长 | `describe` 看 finalizers；`kubectl delete --force --grace-period=0`（**最后手段**）；查节点 |
| **Running 但 Service 无流量** | selector 不匹配、readiness 失败、端口名/targetPort 错 | `kubectl get endpoints <svc>`（空=没就绪）；核对 labels/port；`exec` 里 curl 本地端口 |
| **DNS 解析失败** | CoreDNS 异常、ndots/search 配错、NetworkPolicy 拦 DNS | 起测试 Pod `nslookup`；查 CoreDNS 日志/副本；查 NetworkPolicy（2.2.2） |
| **Node NotReady** | kubelet/运行时/网络异常、磁盘满、心跳超时 | `describe node`；节点上 `systemctl status kubelet`、`journalctl -u kubelet`；30_linux.md 的磁盘/网络排查 |

**通用排查顺序**：`get`（状态）→ `describe`（Events 是金矿）→ `logs --previous`（崩溃前日志）→ `exec`/`debug`（进容器看配置/网络）→ 节点层（kubelet/CRI/CNI/磁盘）→ 控制面（apiserver/scheduler/controller 日志）。**保留现场**：崩溃前日志、`describe` 输出、节点状态（重启前取证，30_linux.md 5.2.1 同款纪律）。

**易错**：只看 `logs` 不看 `--previous`（CrashLoop 时当前容器可能还没日志）；把 Pending 当"调度器坏了"（先看 Events 的资源/污点）；`delete pod` 期待"修好"（控制器会重建同样的 Pod，根因没变）；不看 Events 直接猜。

## 5.2 生态

### 5.2.1 Helm、Kustomize、Operator、CRD 各解决什么？怎么选？

**四者的定位**：

| 工具/机制 | 解决什么 | 关键概念 |
|-----------|---------|---------|
| **Helm** | **应用打包与分发**（像 apt/yum） | Chart（模板+values+metadata）、Release（安装实例）、Repository、`helm upgrade --install`、`rollback`、hooks |
| **Kustomize** | **环境差异化配置**（无模板） | base + overlays、patches（strategic merge/json6902）、`kubectl apply -k`；**不引入模板语言**，适合"同一套 YAML 多环境" |
| **CRD** | **扩展 K8s API**（自定义资源类型） | 定义 `kind: MyDatabase`，像内置资源一样 `kubectl get/apply` |
| **Operator** | **把运维知识编码成控制器**（Day-2 自动化） | CRD + 自定义 Controller 的**调谐循环**（备份/扩缩/故障转移/升级） |

**选型口径**：**打包发布第三方/自研应用 → Helm**（生态最广，Chart 仓库成熟）；**同一应用多环境（dev/staging/prod）差异 → Kustomize**（或 Helm + 多 values，二者可组合：Helm 渲染 + Kustomize 打补丁）；**有状态/复杂运维（数据库、MQ、Prometheus）→ Operator**（否则"部署容易、运维靠人"）；**只是加几个字段/简单资源 → CRD 就够，不必写 Operator**。

**"别过度"**：小团队几个服务用 `kubectl apply -f` + Kustomize 就够；Helm 的模板调试成本与"values 迷宫"是真实负担；Operator 的开发/维护成本高（除非核心中间件）。**判断标准**：**变更频率 × 环境数量 × 运维复杂度**。

**易错**：Helm 把 Secret 明文放 values（配合 SealedSecrets/External Secrets）；`helm upgrade` 不带 `--install`（首次失败）；Kustomize 的 patch 目标写错（静默不生效，要 `kustomize build` 验证）；Operator 的 CRD 升级不兼容（老 CR 解析失败）。

### 5.2.2 GitOps 与可观测性体系怎么搭？

**GitOps（ArgoCD/Flux）**：**Git 是唯一真相**——集群状态由 Git 里的清单声明，**CD 控制器在集群内拉取并同步**（pull 模式，不用 CI 持有集群凭据）；核心能力：**漂移检测**（集群实际状态 vs Git）、**自动/手动同步**、**回滚=Git revert**、**多集群/多环境**（目录或 ApplicationSet）、**权限与审计**（谁能改哪个环境=Git 权限）。**与 CI 的分工**：CI 负责"构建镜像+跑测试+更新清单里的镜像 tag"；CD（GitOps）负责"把清单同步到集群"——**构建与部署解耦**（33_cicd.md 的流水线设计同源）。

**可观测三支柱（K8s 版）**：
- **Metrics**：Prometheus（+ Thanos/VictoriaMetrics 长期存储）+ Grafana + Alertmanager；**关键指标**：Pod/节点资源、HPA 副本、重启次数、apiserver 延迟、etcd、Ingress 错误率/延迟（15_spring_cloud.md 4.2.2 的三支柱在 K8s 的落点）；
- **Logs**：Loki（轻量）/EFK（Elasticsearch+Fluent Bit+Kibana，24_elastic_search.md）；**统一采集**（DaemonSet/Agent）与结构化字段（traceId）；
- **Traces**：OpenTelemetry Collector + Jaeger/Tempo；服务网格（2.2.2）可自动产生 L7 遥测。

**告警与 SLO**：按**用户体验**设 SLO（错误率/延迟/可用性），用**燃烧率**告警（比阈值告警更少误报）；**容量与成本**（Kubecost/OpenCost）也要进观测（"资源利用率低"与"账单高"是两件事）。

**收口**："**K8s 的运维成熟度 = 声明式（GitOps）+ 可观测（三支柱）+ 自动化（Operator/HPA）+ 演练（PDB/drain/故障注入）**——四件套齐了才叫'平台'，否则只是'把应用跑在容器里'。"