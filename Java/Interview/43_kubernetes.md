# 1. 知识点
- **Master管理节点**：Kubernetes 集群的控制平面，负责整体调度、管理和决策
- **api-server**：K8s 的统一入口，所有 REST 操作通过 API Server 与 etcd 交互，提供认证、授权、准入控制
- **etcd**：分布式键值存储，保存集群所有配置和状态数据（Pod、Service、Deployment 等），保证集群状态一致性
- **controller-manager**：控制器管理器，运行各种控制器（Deployment Controller、ReplicaSet Controller、Node Controller 等），将集群从当前状态调谐到期望状态
- **scheduler**：调度器，负责为新创建的 Pod 选择最优 Node 节点，基于资源/亲和性/污点等策略
- **Worker Node工作节点**：运行实际业务负载的节点，接收 Master 调度指令
- **kubelet**：运行在每个 Node 上的代理，接收 API Server 的 Pod 调度指令，管理本节点上的 Pod 和容器生命周期
- **kube-proxy**：网络代理，维护 Node 上的网络规则（iptables/IPVS），实现 Service 的负载均衡
- **容器运行时(Container Runtime)**：实际运行容器的软件，如 containerd、CRI-O，K8s 通过 CRI 接口与运行时交互
- **CRI（Container Runtime Interface）**：K8s 的容器运行时标准接口，containerd 实现 CRI 取代了 Docker（dockershim 已废弃）
- **Pod**：K8s 最小调度和部署单元，一个 Pod 可包含一个或多个紧密关联的容器
- **Pod内容器共享网络namespace和存储**：同一 Pod 内的容器共享同一个网络命名空间（同一 IP、可通过 localhost 通信）和挂载的存储卷
- **pause基础容器**：每个 Pod 中 infral 容器（pause 容器），hold 住网络命名空间，其他业务容器加入该命名空间
- **Service**：一个抽象层，为一组功能相同的 Pod 提供统一的访问地址（ClusterIP）和负载均衡
- **ClusterIP**：Service 的默认类型，分配集群内部虚拟 IP，仅集群内部可访问
- **kube-proxy iptables/IPVS规则**：kube-proxy 监听 Service 变化，将 VIP 转发到后端 Pod IP（iptables 或 IPVS 模式）
- **NodePort**：在每个 Node 上暴露固定端口（30000-32767），将请求转发到 Service，适合无外部负载均衡器的场景
- **LoadBalancer**：使用云厂商的负载均衡器（如 AWS ELB、阿里云 SLB）对外暴露 Service
- **Ingress**：HTTP/HTTPS 七层路由规则，将外部请求根据域名和路径路由到集群内部不同的 Service
- **Ingress Controller**：实现 Ingress 规则的控制器，如 nginx-ingress-controller、traefik，需要独立部署
- **Deployment**：管理无状态应用（如 Web 服务），通过 ReplicaSet 管理 Pod 副本，支持滚动更新和回滚
- **ReplicaSet**：保证指定数量的 Pod 副本始终运行，由 Deployment 自动管理，通常不需要手动创建
- **滚动更新(RollingUpdate)**：Deployment 默认更新策略，逐步替换旧 Pod 为新 Pod，保证服务不间断
- **StatefulSet**：用于有状态应用（如数据库），为每个 Pod 提供稳定的网络标识和持久化存储
- **稳定的网络标识**：StatefulSet 的 Pod 名称固定（如 mysql-0、mysql-1），即使重新调度也保留同样名称
- **持久化存储**：StatefulSet 通过 volumeClaimTemplates 为每个 Pod 自动创建独立的 PVC
- **DaemonSet**：保证所有（或部分）Node 上运行一个 Pod 副本，如日志收集 Fluentd、监控 node-exporter
- **ConfigMap**：将非敏感配置数据以明文方式存储，可以环境变量、命令行参数或卷挂载方式注入 Pod
- **Secret**：存储敏感信息（密码、Token、密钥），以 base64 编码存储，可以作为卷挂载或环境变量，也支持 etcd 加密
- **资源管理**：容器可以通过 resources 字段声明资源需求（requests）和上限（limits）
- **requests**：声明容器启动所需的最小资源保障，调度器基于此决策将 Pod 放置到合适的 Node
- **limits**：容器能使用的最大资源上限，超出 limits CPU 会被限流 throttle，内存会被 OOMKill
- **CPU单位millicore**：1 CPU = 1000m（millicores），如 250m 意为 0.25 个 CPU 核心
- **内存单位Mi**：内存单位为 Ki、Mi、Gi，如 128Mi = 128 Mebibyte（Ki=1024, Mi=1024Ki, Gi=1024Mi）
- **Service Account**：提供给 Pod 访问 API Server 的身份认证账户，配合 RBAC 权限控制
- **RBAC权限管理**：基于角色的访问控制，通过 Role（命名空间级）和 ClusterRole（集群级），关联到 ServiceAccount
- **Volume**：Pod 级存储，生命周期与 Pod 相同（emptyDir）或独立于 Pod（PersistentVolume）
- **emptyDir**：临时存储，Pod 创建时分配，Pod 删除时清空，适合 Pod 内容器间共享临时数据
- **hostPath**：将宿主机目录挂载到 Pod，Pod 重启/漂移后数据丢失或不可达，不推荐生产使用
- **PVC**：持久卷声明，用户声明对存储的请求（大小、访问模式、存储类等）
- **PV**：持久卷，由管理员预先配置或通过 StorageClass 动态创建
- **动态供给**：通过 StorageClass 自动创建 PV，用户只需创建 PVC 即可获得对应持久化存储
- **调度策略**：nodeSelector（按标签选择节点）、nodeAffinity（亲和性调度到特定节点更灵活）、Taint Toleration（污点容忍反向选择）
- **Taint Toleration**：节点添加污点 Taint 排除不匹配的 Pod，Pod 通过 Toleration 声明可容忍特定污点
- **HPA(HorizontalPodAutoscaler)**：水平自动扩容/缩容，根据 CPU/内存或其自定义指标自动增减 Pod 副本数
- **Readiness探针**：检查 Pod 是否已准备好接收流量，就绪探针成功才会将 Pod 加入 Service 的 endpoints，失败则移出
- **Liveness探针**：检查容器是否存活，探针失败则重启容器，防止容器假死
- **Startup探针**：启动探针，适用于慢启动容器，启动期间 Liveness 探针被推迟执行
- **kubectl get**：列出资源（pods、services、deployments 等）
- **kubectl describe**：查看资源详细信息和事件
- **kubectl logs**：查看 Pod 日志
- **kubectl exec**：进入容器执行命令
- **kubectl apply**：声明式地应用 YAML/JSON 配置到集群
- **kubectl delete**：删除指定资源
- **kubectl create**：创建资源
- **kubectl top**：查看节点或 Pod 资源使用情况
- **kubectl scale**：调整 Deployment 副本数
- **kubectl rollout**：管理 Deployment 滚动更新流程（status/history/undo/restart）
- **kubectl edit**：直接编辑集群中的资源
- **kubectl annotate**：对资源添加或修改注解
- **Helm**：K8s 应用包管理器，类似 Linux 的 apt/yum，通过 Chart 打包应用
- **Chart**：Helm 包，包含应用所需的资源定义 YAML 模板 + values 变量 + 元数据
- **values.yaml**：定义模板中的默认变量，安装时可通过 --set 或自定义 values 文件覆盖
- **Release**：Chart 安装到 K8s 集群后生成的一个实例，同一 Chart 可安装多个 Release
- **Service Mesh**：服务网格，将服务间通信的基础设施层（流量管理、安全、可观测性）下沉到 Sidecar 代理
- **Istio**：最流行的 Service Mesh，通过 Envoy sidecar 代理接管微服务间流量
- **Envoy**：高性能代理，作为 Sidecar 与业务容器共享 Pod，拦截和处理所有进出流量
- **控制面(Istiod)**：Istio 合并了 Pilot、Citadel、Mixer 等组件，管理 Sidecar 配置
- **K8s vs 传统虚拟机部署**：K8s 以容器为单位自动化编排，声明式配置、自愈、自动扩缩；虚拟机部署需人工管理、资源配置静态

- **K8s YAML文件核心内容**: apiVersion(K8s API版本如apps/v1); kind(类型: Deployment/Service/ConfigMap等); metadata(name/labels/namespace); spec.replicas(Pod副本数); spec.selector(标签选择器); spec.template(Pod模板: metadata+containers/volumes等); containers: image/ports/env/resources(requests/limits)/volumeMounts/livenessProbe/readinessProbe

# 2. 面试题

## 2.1 Kubernetes 架构组件有哪些？各自的作用是什么？
**Master 节点**（控制平面）：
- **API Server**：K8s 统一入口，所有组件通过 API Server 通信，提供 RESTful 接口、认证/授权/准入控制
- **etcd**：分布式 KV 存储，存储集群所有数据（配置、状态），是集群的唯一真相来源
- **Controller Manager**：运行各种控制器，通过调谐循环将集群当前状态调谐到期望状态（如 Deployment Controller 保证副本数）
- **Scheduler**：为新创建的 Pod 选择最优 Node，考虑资源、标签、亲和性/反亲和性、污点容忍等策略

**Worker 节点**：
- **kubelet**：每个 Node 的 agent，接收 API Server 指令，管理 Pod 和容器生命周期的具体执行
- **kube-proxy**：维护网络规则（iptables/IPVS），实现 Service 的负载均衡
- **Container Runtime**：通过 CRI 接口运行容器（containerd/CRI-O）

## 2.2 Pod 是什么？一个 Pod 内的容器有什么特点？
Pod 是 K8s 中最小的调度和部署单元，通常包含一个应用主容器和一个或多个 Sidecar 容器。同一 Pod 内的容器共享：
1. **网络命名空间**：共享同一个 IP 地址和端口空间，可以通过 localhost 互相通信
2. **IPC 命名空间**：可以通过进程间通信（SystemV、POSIX 消息队列）
3. **存储卷（Volume）**：挂载到 Pod 上的 Volumes 可以被所有容器共享

每个 Pod 中还有一个不会终止的 **pause 基础容器**，它创建并 hold 住网络和 IPC 命名空间，业务容器加入该命名空间。

## 2.3 Service 有哪些类型？各自的使用场景？
- **ClusterIP（默认）**：只在集群内部可访问的虚拟 IP，用于内部微服务间通信
- **NodePort**：在每个 Node 上开放固定端口（30000-32767），可从集群外部通过 NodeIP:NodePort 访问。适用于开发测试或没有 LoadBalancer 的裸机环境
- **LoadBalancer**：使用云厂商的负载均衡器，自动分配外部 IP，将流量转发到 NodePort。适用于生产环境有云厂商支持的场景
- **ExternalName**：将 Service 映射到外部 DNS 名称，用于访问集群外部服务

## 2.4 Deployment 和 StatefulSet 有什么区别？
| 维度 | Deployment | StatefulSet |
|------|-----------|-------------|
| 适用场景 | 无状态应用（Web、API） | 有状态应用（数据库、消息队列） |
| Pod 标识 | 随机名称（如app-5f6b8c-abc） | 固定序号名称（如mysql-0, mysql-1） |
| 网络标识 | 无稳定网络标识 | 稳定的网络标识（DNS名固定） |
| 持久存储 | 所有 Pod 共享 PVC 或无需存储 | 每个 Pod 有独立 PVC（volumeClaimTemplates） |
| 启动顺序 | 并行启动 | 按序号顺序启动（0→1→2），逆序关闭 |
| 扩缩容 | 随意扩缩 | 按顺序扩缩容，保证数据一致性 |

## 2.5 Ingress 是什么？和 Service 有什么区别？
Ingress 是七层（HTTP/HTTPS）路由规则，根据请求的域名和 URL 路径将外部请求路由到集群内部不同的 Service。Service 是四层（TCP/UDP）负载均衡。

区别：
- Service 四层，Ingress 七层，Ingress 可有基于域名和路径的灵活路由规则
- Ingress 需要部署 Ingress Controller（如 nginx-ingress）才能工作
- Ingress 支持 SSL 终结、TLS 配置、重写等高级功能
- 一般架构：外部流量 → Ingress → Service → Pod

## 2.6 ConfigMap 和 Secret 有什么区别？如何注入到 Pod？
- **ConfigMap**：存储非敏感的明文配置数据，直接查看
- **Secret**：存储敏感数据（密码、Token、密钥），base64 编码存储（不是加密），可启用 etcd 静态加密保护

注入方式：
1. **环境变量**：通过 `env.valueFrom.configMapKeyRef` 或 `secretKeyRef` 将指定 key 注入为环境变量
2. **卷挂载**：将整个 ConfigMap/Secret 以文件形式挂载到容器中，每个 key 成为一个文件
3. **命令行参数**：配合环境变量使用

Secret 的 base64：`echo -n 'password' | base64`，解码：`echo 'cGFzc3dvcmQ=' | base64 -d`

## 2.7 如何配置 Pod 的健康检查？三种探针的区别？
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

| 探针 | 作用 | 失败处理 |
|-----|------|---------|
| Liveness | 检测容器是否存活 | 杀死容器，根据 restartPolicy 重启 |
| Readiness | 检测容器是否准备好接收流量 | 从 Service Endpoints 中移除，不接收流量 |
| Startup | 检测容器是否已启动 | 启动成功前 Liveness 暂时不执行，失败重试直至 success |

## 2.8 HPA 是什么？如何工作？
HorizontalPodAutoscaler 水平自动扩缩容，根据观察到的 CPU/内存利用率或自定义指标（如 QPS、队列长度）自动调整 Deployment/StatefulSet 的 Pod 副本数。

工作流程：
1. HPA 控制器定期（默认15s）从 Metrics Server 获取 Pod 的资源使用率
2. 计算期望副本数 = ceil(当前副本数 × 当前利用率 / 目标利用率)
3. 如果期望副本数与当前不同，通过 Scale 接口修改 Deployment 的 replicas
4. 缩容有冷却期（默认5分钟），防止抖动

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
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

## 2.9 如何理解 Kubernetes 的资源 requests 和 limits？
- **requests**：声明容器启动时保证获得的最低资源（CPU/内存），调度器基于 requests 判断 Node 是否有足够资源。是 Pod 能稳定运行的最低保障
- **limits**：容器能使用的最高资源上限。CPU 超过 limits 被限流（throttle），但不会被杀；内存超过 limits 会被 OOM Kill

最佳实践：
- requests 和 limits 都应设置，避免资源竞争
- CPU 请求量和限制量一般设为 1:2 到 1:5 之间
- 内存 requests = limits（保持 Guaranteed QoS 类别，稳定性最高）
- 不要在 limits 中设置过高的内存值，否则 OOM 前可能影响 Node 稳定性

## 2.10 Kubernetes 的调度策略有哪些？
1. **nodeSelector**：最简单的调度约束，Pod 指定标签，只能调度到有匹配标签的 Node
2. **nodeAffinity（节点亲和性）**：比 nodeSelector 更灵活，支持硬约束(required)和软约束(preferred)
3. **podAffinity/podAntiAffinity**：Pod 间亲和性/反亲和性，决定 Pod 应该或不应该和哪些 Pod 部署在一起
4. **Taints & Tolerations（污点与容忍）**：Node 打上污点(Taint)排斥不匹配 Pod，Pod 声明容忍(Toleration)才可调度到该 Node
5. **Topology Spread Constraints**：控制 Pod 在拓扑域（如 zone、host）间的分布均匀度
6. **资源调度**：根据 Node 剩余资源(CPU/Memory)与 Pod 资源声明匹配，默认 LeastRequestedPriority 策略

## 2.11 Helm 是什么？核心概念有哪些？
Helm 是 Kubernetes 的包管理工具，类似于 yum/apt。核心概念：
- **Chart**：K8s 应用包，包含模板文件、默认 values、Chart.yaml 元数据等
- **Repository**：Chart 的仓库（如同 Docker Hub），可公用或私有
- **Release**：Chart 安装在集群上的实例，同一 Chart 可安装多个 Release
- **values.yaml**：Chart 模板的默认配置变量，安装时可通过命令行 `--set` 或自定义 values 文件覆盖
- **Revision**：Release 的版本记录，支持 `helm rollback` 回退到历史版本

## 2.12 什么是 Service Mesh？Istio 的核心组件有哪些？
Service Mesh 是服务网格，将微服务间通信的通用功能（服务发现、负载均衡、流量管理、安全、可观测性）从应用代码中剥离，下沉到基础设施层的 Sidecar 代理中，应用无需感知。

**Istio 核心组件**：
- **Envoy Sidecar**：每个业务 Pod 中注入一个 Envoy 代理，拦截和处理所有进出流量
- **Istiod（控制面）**：合并了 Pilot（服务发现/流量管理）、Citadel（证书/安全）、Galley（配置管理），统一管理和分发配置到 Envoy
- **Ingress/Egress Gateway**：作为入口/出口边界代理

优势：业务代码无需修改流量治理代码，透明实现金丝雀发布、熔断、限流、链路追踪等功能。

## 2.13 Kubernetes vs 传统虚拟机部署的优势？
| 维度 | Kubernetes | 传统虚拟机 |
|------|-----------|-----------|
| 部署方式 | 声明式配置(`kubectl apply -f`) | 手动或脚本操作，过程式 |
| 自愈能力 | 自动重启失败容器、重新调度 | 需人工监控恢复 |
| 弹性伸缩 | HPA 自动水平伸缩 | 手动扩容或复杂脚本 |
| 滚动更新 | Deployment 原生支持 | 需自行实现 |
| 资源密度 | 容器级共享，密度高 | 每 VM 独立 OS，密度低 |
| 服务发现 | 内置 DNS + Service 自动发现 | 需额外组件（Consul/Eureka） |
| 配置管理 | ConfigMap/Secret 热更新 | 修改配置需重部署 |

## 2.14 k8s的yml/deployment文件包含哪些核心内容？
Kubernetes Deployment YAML 文件包含三层核心结构：

1. **apiVersion**：声明使用的K8s API版本，如`apps/v1`（Deployment）、`v1`（Service/ConfigMap）
2. **kind**：资源类型，如Deployment、Service、ConfigMap、Ingress等
3. **metadata**：元数据，包含name（资源名称）、labels（标签）、namespace（命名空间）、annotations（注解）

4. **spec（规格）**：
   - **replicas**：Pod期望副本数，`spec.replicas: 3` 表示运行3个Pod
   - **selector**：标签选择器，用于关联和管理Pod，`matchLabels`与Pod模板的labels必须一致
   - **template**：Pod模板，定义Pod的详细配置：
     - **template.metadata**：Pod自身标签（必须匹配selector）
     - **template.spec.containers**：容器列表：
       - **name**：容器名称
       - **image**：镜像地址（支持版本标签如`:v1.0.0`）
       - **ports**：暴露端口（containerPort）
       - **env**：环境变量（可引用ConfigMap/Secret）
       - **resources**：资源请求（requests）和上限（limits）
       - **volumeMounts**：卷挂载到容器内的路径
       - **livenessProbe**：存活探针配置
       - **readinessProbe**：就绪探针配置
     - **template.spec.volumes**：声明数据卷（ConfigMap/Secret/PVC等）
