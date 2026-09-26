# 1.Pod概述

## 1.1 什么是Pod

Pod 是 Kubernetes 中**最小、最基本**的调度和部署单元。它代表集群中运行的一个或一组容器的集合。理解 Pod 的关键在于——Pod 不是一个具体的"进程"或"容器"，而是一个**逻辑主机**（Logical Host），它封装了一个或多个紧密耦合的应用容器，并为这些容器提供共享的运行环境。

从物理视角看，一个 Pod 中的容器总是：
- 被调度到**同一个 Node** 上
- **同时启动、同时终止**
- 共享相同的 Linux 命名空间（Network、IPC 等）
- 可以共享存储卷（Volume）

从概念视角看，Pod 相当于一个**虚拟机**或**物理机**：其内部运行的多个容器就像是同一台机器上的多个进程——它们共享网络、可以通过 localhost 通信、可以共享文件系统。

Pod 的"原子性"体现在：
- **调度原子性**：调度器将 Pod 作为一个整体分配到某个 Node，不会将 Pod 内的不同容器分散到不同 Node
- **生命周期原子性**：Pod 中的容器共同经历 Pod 的生命周期阶段（Pending → Running → Succeeded/Failed）
- **资源原子性**：Pod 内所有容器的资源请求（requests）总和决定了调度决策

Kubernetes 的设计哲学中有一个著名的"每个 Pod 一个容器"模式——多数情况下一个 Pod 只运行一个主容器。但 Pod 的设计能力原生支持多容器协同工作，这为 Sidecar、Ambassador、Adapter 等模式提供了基础。

### Pod内共享命名空间的设计哲学

Pod 通过 Linux 命名空间（Namespace）实现容器间的隔离与共享。Pod 的设计选择了**共享部分命名空间**的方案，这种"选择性共享"体现了精妙的设计权衡：

**Pod内共享的命名空间**：

| 命名空间 | 是否共享 | 说明 |
|---------|---------|------|
| **Network** | 共享 | 所有容器共享同一个网络命名空间：同一个 IP、同一个端口空间、通过 localhost 通信 |
| **IPC** | 共享 | 共享 System V IPC 和 POSIX 消息队列，允许容器间通过 IPC 通信 |
| **UTS** | 共享（可配置）| 共享主机名和域名，Kubernetes 默认将其设为 Pod 名称 |
| **PID** | 可选共享 | 通过 `shareProcessNamespace: true` 启用，允许容器看到彼此的进程 |
| **Mount** | 独立 | 每个容器有独立的文件系统视图（rootfs），Volume 可以共享挂载 |
| **User** | 独立 | 每个容器可以有独立的 UID/GID 映射 |

**为什么这样设计**：

Network 命名空间共享是最核心的设计。这意味着：
- Pod 中的所有容器共享同一个 IP 地址，避免了复杂的容器间网络配置
- 容器间可以通过 `localhost` 进行高性能通信，无需经过网络协议栈的完整路径
- 端口不能冲突——同一 Pod 内的两个容器不能监听同一个端口，这要求开发者做好端口规划

这种设计直接解决了"紧密耦合容器"的需求。比如一个 Web 容器和一个日志收集 Sidecar 容器——日志收集器通过 `localhost` 就能读取 Web 容器的日志，无需额外的网络配置和服务发现。

PID 命名空间共享是可选的，默认不共享。这是出于安全考虑——不共享 PID 命名空间时，每个容器只能看到自己的进程，提供了更好的隔离性。当需要容器间进程管理（如共享进程信号）时，可以显式启用。

## 1.2 为什么需要Pod

要理解 Pod 存在的必要性，需要回答一个问题：**为什么不能直接调度容器？**

事实上，Kubernetes 完全可以直接调度容器——容器运行时（如 containerd）本身就支持单容器的生命周期管理。但 Kubernetes 引入了 Pod 这一层抽象，是出于以下深层原因：

### 1. 紧密耦合容器的协同调度需求

在实际应用中，经常存在需要"绑在一起运行"的容器组：

```
场景1：Web服务 + 日志收集
┌─────────────────────────┐
│  Pod                    │
│  ┌─────────┐ ┌────────┐ │
│  │  Nginx  │ │ Fluentd│ │  Fluentd 通过 localhost 从 Nginx 读取日志
│  │ :80     │ │        │ │  两者必须在同一台机器上
│  └─────────┘ └────────┘ │
└─────────────────────────┘

场景2：应用 + 配置更新
┌─────────────────────────┐
│  Pod                    │
│  ┌─────────┐ ┌────────┐ │
│  │  App    │ │ Config │ │  Config Sidecar 通过 localhost 向 App 推送配置更新
│  │ :8080   │ │Syncer  │ │
│  └─────────┘ └────────┘ │
└─────────────────────────┘

场景3：服务网格 Sidecar
┌─────────────────────────┐
│  Pod                    │
│  ┌─────────┐ ┌────────┐ │
│  │  App    │ │  Envoy │ │  Envoy 代理拦截并管理 App 的所有出入流量
│  │         │ │ (proxy)│ │  通过 iptables 规则，共享网络栈
│  └─────────┘ └────────┘ │
└─────────────────────────┘
```

如果没有 Pod，这些容器需要**手动**确保被调度到同一台机器，并且手动配置它们之间的网络互通——这既复杂又脆弱。

### 2. 资源共享与生命周期管理的统一

Pod 作为"容器组"，提供：
- **统一的资源计量**：Pod 内所有容器的 `requests` 合计才是调度器考虑的占用量
- **统一的终止信号**：删除 Pod 时，所有容器同步收到 SIGTERM，共享 `terminationGracePeriodSeconds`
- **统一的状态机**：Pod 只有一套 Phase（Pending/Running/Succeeded/Failed），简化了上层控制器的状态管理

### 3. 模式抽象的价值

Pod 为上层工作负载抽象提供了统一基础：

- **Deployment** 管理的是 Pod 的副本集，而不是一组容器的副本集
- **Service** 选择的是 Pod，负载均衡目标也是 Pod
- **NetworkPolicy** 作用于 Pod 边界
- **PersistentVolumeClaim** 绑定到 Pod 使用

有了 Pod 这一层抽象，上层所有控制器都操作统一的对象，无需关心底层是单个容器还是多个容器。

### 4. 历史原因：Borg 的影响

Pod 的概念直接来源于 Google 内部系统 Borg 的 `alloc`（allocation）：
- Borg 中，任务不是单独运行的，而是分配到一个 `alloc` 中
- 一个 `alloc` 可以容纳多个 task，它们共享资源和网络
- Kubernetes 的 Pod 就是 Borg `alloc` 的简化与标准化版本

## 1.3 Pod vs 容器 vs 进程 的类比

理解 Pod、容器、进程三者关系的最佳方式是通过类比：

| 概念层次 | Kubernetes | 类比：物理机/VM | 类比：操作系统 |
|---------|-----------|---------------|-------------|
| **调度单元** | Pod | 物理机 / 虚拟机 | — |
| **执行单元** | Container | 机器上的进程 | 进程 |
| **共享环境** | Pod 命名空间 | 同一台机器的 OS | — |
| **通信方式** | localhost + Pod IP | IP 地址 | 本机 IPC / 信号 |
| **存储** | Volume | 磁盘 | 文件系统 |
| **生命周期** | Pod Phase | 机器开关机 | 进程组 |
| **管理单元** | Deployment/StatefulSet | 集群管理软件 | 进程管理器/init |

**深度类比**：

```
操作系统视角：
  一台服务器
    ├── 进程1（Nginx）
    │     ├── PID 1000
    │     └── 监听 0.0.0.0:80
    ├── 进程2（日志收集）
    │     ├── PID 1001
    │     └── 读取进程1的日志文件
    └── 共享：同一内核、同一网络栈、同一文件系统

Kubernetes 视角：
  一个 Pod
    ├── 容器1（Nginx）
    │     └── 监听 0.0.0.0:80
    ├── 容器2（日志收集）
    │     └── 读取容器1的日志（共享Volume）
    └── 共享：同一 Network NS、同一 IPC NS、同一 Pod IP
```

**关键区别**：

1. **容器 ≠ 进程**：容器内可以运行多个进程（虽然不推荐）。容器更像是"被命名空间隔离的进程组"——它拥有独立的 rootfs、PID namespace、mount namespace。

2. **Pod ≠ 虚拟机**：Pod 比 VM 更轻量。Pod 内的容器共享内核，没有完整的 OS 开销。Pod 也不提供 VM 级别的强隔离——容器间共享的命名空间意味着一个容器可能影响其他容器。

3. **Pod = 逻辑主机边界**：Pod 定义了一个"小环境"的边界——这个环境内的成员共享网络、可以本地通信、共同被调度。这类似于一个"微型虚拟机"，但没有虚拟化开销。

**设计上的哲学思考**：

Kubernetes 的设计者有意让 Pod 成为这个"逻辑主机"的边界，而不是让容器成为独立的调度单元。这样做的好处是：
- 保持了与物理机/VM 模式相近的心智模型（同一台"机器"上可以有多个"进程"）
- 允许容器镜像保持"一个进程一个容器"的最佳实践（每个容器做一件事），同时支持容器间紧密协作
- 为 Sidecar、Init Container、Ephemeral Container 等模式提供了统一的执行环境

## 1.4 Pod内资源共享机制

### Network 命名空间共享

Pod 内所有容器共享同一个网络命名空间，这意味着：

```
Pod IP: 10.244.1.5
┌───────────────────────────────────┐
│  Network Namespace (podns)        │
│  ┌─────────────┐ ┌─────────────┐  │
│  │ Container A │ │ Container B │  │
│  │ 监听 :8080  │ │ 监听 :9090  │  │
│  │             │ │             │  │
│  │ localhost───┼─┤ localhost   │  │  ← 通过 loopback 直接通信
│  │   通信      │ │   通信      │  │
│  └─────────────┘ └─────────────┘  │
│         │               │         │
│         └─── eth0 ──────┘         │  ← 共享同一个网络接口
│                │                   │
└────────────────┼───────────────────┘
                 │
          CNI 网络 (flannel/calico/cilium...)
```

**底层实现**：
1. kubelet 调用 CRI 创建 Pod 时，容器运行时（如 containerd）首先创建一个 **pause 容器**（也叫 infra 容器或 sandbox 容器）
2. pause 容器创建并持有 Network Namespace
3. 后续的 Pod 内业务容器通过 `docker run --net=container:<pause_container_id>` 的方式加入到 pause 容器的网络命名空间
4. 因此，Pod 的 IP 地址实际上是分配给 pause 容器的网络命名空间，其他容器复用这个网络栈

**pause 容器的作用**：
- pause 容器是一个非常轻量的容器（镜像大小约 700KB），仅执行 `pause()` 系统调用，永远处于休眠状态
- 它的存在是为了"锚定"共享的命名空间——即使所有业务容器都崩溃重启，pause 容器仍然存活，网络命名空间不会丢失，Pod IP 不会改变
- pause 容器是 Pod 中第一个被创建、最后一个被销毁的容器

**实际影响**：
- 同一 Pod 内容器端口不能冲突（共享同一个端口命名空间）
- Pod 内通信不需要服务发现，直接 `localhost:<port>` 即可，性能极高
- Sidecar 模式的核心技术基础就在于此

### IPC 命名空间共享

Pod 内所有容器共享 IPC（Inter-Process Communication）命名空间，允许使用：
- System V 信号量（semget/semop）
- POSIX 消息队列（mq_open/mq_send/mq_receive）
- 共享内存段（shmget/shmat）

```yaml
# Pod 共享 IPC 命名空间是默认行为
apiVersion: v1
kind: Pod
metadata:
  name: ipc-demo
spec:
  containers:
  - name: producer
    image: alpine
    command: ["sh", "-c", "ipcmk -M 1024 && ipcs -m && sleep 3600"]
  - name: consumer
    image: alpine
    command: ["sh", "-c", "sleep 5 && ipcs -m && sleep 3600"]
  # IPC 命名空间自动共享，两个容器可以看到同样的共享内存段
```

**注意**：虽然共享 IPC 命名空间，但实际中 Pod 内进程间通信更多使用 localhost 网络或共享 Volume，直接使用 System V IPC 的场景较少。

### PID 命名空间共享（可选）

默认情况下，Pod 内容器不共享 PID 命名空间——每个容器只能看到自己的进程。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pid-share-demo
spec:
  shareProcessNamespace: true  # 启用 PID 命名空间共享
  containers:
  - name: app
    image: nginx
  - name: debugger
    image: alpine
    command: ["sh", "-c", "sleep 3 && ps aux && sleep 3600"]
    # debugger 容器中 ps aux 可以看到 nginx 容器的进程
```

**使用场景**：
- 调试：Sidecar 需要查看主容器进程状态
- 信号传递：需要在容器间发送信号（如 SIGHUP 重载配置）
- 进程监控：Sidecar 监控 Pod 内所有容器的进程

**安全考量**：共享 PID 命名空间会暴露进程信息，降低了容器间的隔离性。在共享 PID 命名空间后，`/proc` 文件系统也会暴露其他容器的进程信息，这可能带来安全隐患。因此默认关闭。

### UTS 命名空间共享

Pod 内所有容器共享 UTS（UNIX Time-Sharing）命名空间，即共享主机名（hostname）和域名。Kubernetes 默认将 Pod 的主机名设置为 Pod 名称。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  hostname: custom-host  # 显式指定主机名
  subdomain: my-svc      # 指定子域名，形成 FQDN
  containers:
  - name: app
    image: nginx
    # 在容器内 hostname 命令将返回 "custom-host"
```

### Volume 共享

Volume 是 Pod 内资源共享的另一个重要维度。Pod 级别定义 Volume，容器通过 `volumeMounts` 挂载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-share-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  - name: app-config
    configMap:
      name: my-config
  containers:
  - name: writer
    image: alpine
    command: ["sh", "-c", "echo 'hello from writer' > /data/msg.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: alpine
    command: ["sh", "-c", "sleep 5 && cat /data/msg.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
  # writer 容器写入 shared-data 的文件，reader 容器可以读取
```

**Volume 类型与共享特性**：

| Volume 类型 | 共享范围 | 生命周期 | 典型用途 |
|------------|---------|---------|---------|
| `emptyDir` | Pod 内所有容器 | 与 Pod 相同 | 临时数据、Sidecar 通信 |
| `configMap` | Pod 内所有容器 | 独立于 Pod | 配置文件 |
| `secret` | Pod 内所有容器 | 独立于 Pod | 敏感信息 |
| `hostPath` | 同一 Node | 独立于 Pod | 访问节点文件系统 |
| `persistentVolumeClaim` | Pod 内所有容器 | 独立于 Pod | 持久化数据 |
| `projected` | Pod 内所有容器 | 与 Pod 相同 | 多源投射到同一目录 |

**emptyDir 的底层实现**：
- 创建 Pod 时，kubelet 在 Node 的文件系统上创建目录（通常在 `/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/`）
- 这个目录被 bind mount 到 Pod 内各个容器的指定路径
- `emptyDir` 可以使用 `medium: Memory` 选项挂载为 tmpfs
- Pod 删除时，`emptyDir` 数据被清理；容器重启不会清理 `emptyDir`

# 2.Pod YAML详解

## 2.1 完整YAML结构

一个完整的 Pod YAML 定义包含以下核心部分：

```yaml
apiVersion: v1            # API 版本，Pod 属于核心 v1 组
kind: Pod                 # 资源类型
metadata:                 # 元数据
  name: my-app
  namespace: default
  labels:
    app: my-app
  annotations:
    description: "演示用Pod"
spec:                     # 期望状态（规约）
  containers:
  - name: main-container
    image: nginx:1.25
    ports:
    - containerPort: 80
  restartPolicy: Always
status:                   # 实际状态（由系统维护，创建时不需要填写）
  phase: Running
  conditions:
  - type: Ready
    status: "True"
```

**apiVersion/kind** 标识了这是一个 Pod 对象。Kubernetes 中每个 API 对象都需要这两个字段来标识其类型。

**metadata vs spec vs status** 的三段式结构是 Kubernetes API 的通用设计模式：
- **metadata**：资源的身份信息和标签，用于标识、分类和组织资源
- **spec**：用户声明的期望状态（Desired State），告诉 Kubernetes "你想要什么"
- **status**：系统观测到的实际状态（Actual State），由 Kubernetes 自动更新，告诉用户"现在是什么状态"

这种"期望状态 vs 实际状态"的分离是声明式 API 的核心。控制器持续比较 spec 和 status，执行操作使 status 向 spec 收敛——这就是 **控制循环（Control Loop）** 的精髓。

## 2.2 metadata字段详解

### name

Pod 的名称，在同一个 namespace 内必须唯一。命名规则：
- 最长 253 字符
- 只能包含小写字母、数字、`-` 和 `.`
- 必须以字母数字开头和结尾

```yaml
metadata:
  name: nginx-deployment-7b5f8c9d4-x7k2l  # ReplicaSet 生成的 Pod 名称
  # 格式：<deployment-name>-<replicaset-hash>-<pod-hash>
```

### namespace

Kubernetes 使用命名空间实现资源的逻辑隔离：

```yaml
metadata:
  name: my-pod
  namespace: production  # 如果不指定，默认为 "default"
```

命名空间的作用：
- 资源隔离：不同命名空间的资源互不可见
- 访问控制：RBAC 可以按命名空间授权
- 资源配额：ResourceQuota 作用于命名空间
- 网络策略：NetworkPolicy 按命名空间控制流量

注意：并非所有资源都属于命名空间。Node、PersistentVolume、Namespace 本身、StorageClass 等是集群级别的资源。

### labels

标签是键值对，用于标识和选择资源。它是 Kubernetes 中最重要的组织机制：

```yaml
metadata:
  labels:
    app: my-app              # 应用名称
    version: v1.2.3           # 版本号
    environment: production   # 环境（dev/staging/production）
    tier: frontend            # 层级（frontend/backend/database）
    track: stable             # 部署轨迹（stable/canary）
    release: r2024-01         # 发布标识
```

**标签 vs 注解的选择**：
- Labels：用于 Kubernetes 内部的选择和分组（Service 选择 Pod 靠的就是标签选择器）
- Annotations：用于存储非标识性元数据（构建信息、版本、联系方式等），不能被选择器查询

**标签选择器（Label Selector）**：

```yaml
# 等值选择器
selector:
  matchLabels:
    app: my-app
    environment: production

# 集合选择器
selector:
  matchExpressions:
  - key: environment
    operator: In
    values: ["production", "staging"]
  - key: version
    operator: NotIn
    values: ["v0.1"]
  - key: tier
    operator: Exists
```

支持的集合操作符：`In`、`NotIn`、`Exists`、`DoesNotExist`。

**标签最佳实践**：
- 使用反向域名前缀组织标签：`example.com/team: frontend`
- 推荐的通用标签（来自 Kubernetes 官方建议）：`app.kubernetes.io/name`、`app.kubernetes.io/instance`、`app.kubernetes.io/version`、`app.kubernetes.io/component`、`app.kubernetes.io/part-of`、`app.kubernetes.io/managed-by`

### annotations

注解用于存储非标识性的元数据：

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
    build: "two-tone-248"
    imageregistry: "https://hub.docker.com/"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod",...}
```

注解的典型用途：
- 构建和发布信息（镜像版本、Git commit SHA）
- 监控配置（Prometheus scrape 配置）
- 客户端工具配置（kubectl、Helm 等）
- 云平台集成（AWS Load Balancer 配置等）
- 字段级文档或联系人信息

### ownerReferences

`ownerReferences` 建立了资源间的父子关系。当父资源被删除时，子资源也会被级联删除（垃圾回收）：

```yaml
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: nginx-deployment-7b5f8c9d4
    uid: 3c5b4e2c-1234-4d5e-8f9a-1b2c3d4e5f6g
    controller: true
    blockOwnerDeletion: true
```

字段说明：
- `apiVersion` / `kind` / `name` / `uid`：标识父资源
- `controller: true`：标明这是控制器管理的关系（一个资源可以有多个 ownerReferences，但只能有一个 controller=true）
- `blockOwnerDeletion: true`：阻止删除父资源直到此资源被删除（防止挂载了 PVC 的 Pod 被一起删除导致数据丢失）

这就是 Deployment → ReplicaSet → Pod 的级联关系在底层如何实现的。当你删除一个 Deployment 时，Kubernetes 通过 ownerReferences 找到并删除对应的 ReplicaSet 和 Pod。

## 2.3 spec字段详解

### containers（必需字段）

Pod 中最核心的部分，定义要运行的容器列表。至少需要一个容器。

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    command: ["/bin/sh"]
    args: ["-c", "nginx -g 'daemon off;'"]
    workingDir: /usr/share/nginx/html
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
    - name: https
      containerPort: 443
      protocol: TCP
    env:
    - name: ENVIRONMENT
      value: "production"
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
    - name: app-data
      mountPath: /data
    - name: config
      mountPath: /etc/config
      readOnly: true
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 20
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
```

### initContainers

初始化容器，在应用容器启动前按顺序执行：

```yaml
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.36
    command: ["sh", "-c", "until nslookup myservice; do echo waiting; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.36
    command: ["sh", "-c", "until nslookup mydb; do echo waiting; sleep 2; done"]
  containers:
  - name: app
    image: myapp:latest
```

Init 容器特性：
- 按定义的顺序**依次**执行，前一个成功后才会启动下一个
- 所有 init 容器成功后，应用容器才启动
- 如果某个 init 容器失败，kubelet 根据 `restartPolicy` 决定是否重启（`Never` 会让整个 Pod 失败）
- Init 容器可以使用与普通容器不同的镜像，因为它们有独立的文件系统

### volumes

Pod 级别的卷定义，供 Pod 内所有容器挂载使用：

```yaml
spec:
  volumes:
  - name: app-data
    emptyDir: {}
  - name: app-log
    emptyDir:
      medium: Memory       # 使用内存作为存储介质
      sizeLimit: 128Mi    # 限制大小
  - name: config-volume
    configMap:
      name: my-config
      items:               # 只映射特定 key
      - key: app.properties
        path: app.properties
  - name: secret-volume
    secret:
      secretName: my-secret
      defaultMode: 0400    # 文件权限
  - name: host-volume
    hostPath:
      path: /var/log
      type: DirectoryOrCreate
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### restartPolicy

定义 Pod 内容器的重启策略：

| 值 | 行为 | 适用场景 |
|----|------|---------|
| `Always`（默认）| 容器退出后总是重启 | 长期运行的服务（Web 服务、API） |
| `OnFailure` | 仅在容器非零退出码时重启 | Batch 任务、Job |
| `Never` | 永不重启 | 一次性任务、初始化任务 |

```yaml
spec:
  restartPolicy: OnFailure
```

### serviceAccountName

指定 Pod 使用的 ServiceAccount，决定了 Pod 与 API Server 交互时的身份和权限：

```yaml
spec:
  serviceAccountName: my-sa
  automountServiceAccountToken: false  # 禁止自动挂载 SA Token
```

当不指定时，使用命名空间的 `default` ServiceAccount。最佳实践是为每个应用创建专用的最小权限 ServiceAccount。

### nodeName

直接指定 Pod 要调度到的节点名称，会绕过调度器的调度过程：

```yaml
spec:
  nodeName: worker-node-1
```

**应用场景**：
- 紧急情况需要手动指定节点
- 测试和调试
- 某些特殊场景下的静态 Pod

注意：使用 `nodeName` 后，即使节点不可调度（cordon），Pod 仍会强制调度到该节点。这绕过了调度器的预选和优选逻辑，一般不推荐在生产环境使用。

### 其他重要 spec 字段

```yaml
spec:
  # 节点选择器
  nodeSelector:
    disktype: ssd

  # DNS 策略
  dnsPolicy: ClusterFirst     # ClusterFirst/Default/ClusterFirstWithHostNet/None
  dnsConfig:                  # 自定义 DNS 配置
    nameservers:
    - 8.8.4.4
    searches:
    - my-namespace.svc.cluster.local
    options:
    - name: ndots
      value: "2"

  # 主机网络和 PID
  hostNetwork: false          # 使用主机网络命名空间
  hostPID: false              # 使用主机 PID 命名空间
  hostIPC: false              # 使用主机 IPC 命名空间

  # 主机名
  hostname: custom-pod-host   # 覆盖 Pod 主机名
  subdomain: myservice        # 配合 headless service 使用

  # 容忍
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300

  # 亲和性
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - zone-a
            - zone-b

  # 优雅终止时间
  terminationGracePeriodSeconds: 30

  # 优先级
  priorityClassName: high-priority

  # 安全上下文 (Pod级别)
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
    runAsNonRoot: true

  # 调度器名称
  schedulerName: default-scheduler
```

# 3.容器详解

## 3.1 Container定义

容器是 Pod 中最核心的执行单元。一个完整的容器定义如下：

```yaml
spec:
  containers:
  - name: my-app                    # 容器名称，Pod内唯一
    image: nginx:1.25-alpine        # 镜像
    imagePullPolicy: IfNotPresent   # 镜像拉取策略
    command: ["/bin/sh"]            # 覆盖镜像的 ENTRYPOINT
    args: ["-c", "nginx -g 'daemon off;'"]  # 覆盖镜像的 CMD
    workingDir: /app                # 工作目录
    ports:
    - name: http                    # 端口名称
      containerPort: 8080           # 容器监听端口
      hostPort: 8080                # 主机端口（不推荐）
      hostIP: 192.168.1.100         # 绑定的主机IP
      protocol: TCP                 # TCP/UDP/SCTP
    env:
    - name: APP_ENV
      value: "production"
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName  # 引用 Pod 字段
    - name: CPU_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: my-app
          resource: limits.cpu      # 引用容器资源限制
    envFrom:
    - configMapRef:
        name: app-config            # 注入整个 ConfigMap
    - secretRef:
        name: app-secrets           # 注入整个 Secret
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
    volumeMounts:
    - name: app-data
      mountPath: /data
      subPath: my-app/data          # 只挂载卷中的子目录
      readOnly: false
    - name: app-config
      mountPath: /etc/config/app.conf
      subPath: app.conf             # 挂载单个文件而非目录
      readOnly: true
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo 'started' > /tmp/started"]
      preStop:
        exec:
          command: ["/bin/sh", "-c", "nginx -s quit && sleep 10"]
    securityContext:                # 容器级别安全上下文
      runAsUser: 1000
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
      readOnlyRootFilesystem: true
    stdin: false                    # 是否分配标准输入
    stdinOnce: false                # 标准输入是否只打开一次
    tty: false                      # 是否分配 TTY
    terminationMessagePath: /dev/termination-log  # 终止消息路径
    terminationMessagePolicy: File  # File/FallbackToLogsOnError
```

**command 与 args 覆盖规则**：

这是容器镜像中 ENTRYPOINT 和 CMD 与 Kubernetes 配置的覆盖关系：

| | 镜像 ENTRYPOINT | 镜像 CMD | 容器 command | 容器 args | 最终执行 |
|---|---|---|---|---|---|
| 场景1 | `/entrypoint.sh` | `--help` | 未设置 | 未设置 | `/entrypoint.sh --help` |
| 场景2 | `/entrypoint.sh` | `--help` | `/bin/sh` | 未设置 | `/bin/sh` |
| 场景3 | `/entrypoint.sh` | `--help` | 未设置 | `--version` | `/entrypoint.sh --version` |
| 场景4 | 未设置（shell形式）| `python app.py` | `/bin/sh` | `-c` | `sh -c` |

简言之：`command` 覆盖 ENTRYPOINT，`args` 覆盖 CMD。

**terminationMessagePath**：容器退出时的最后几行输出会被写入此路径的文件（默认 `/dev/termination-log`），然后 kubelet 会读取此内容并设置到 Pod 的 `status.containerStatuses[].state.terminated.message` 字段。这对于调试容器启动失败的原因非常有用。

## 3.2 镜像拉取策略

`imagePullPolicy` 决定 kubelet 何时拉取镜像：

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `IfNotPresent`（默认）| 本地有镜像则使用本地，没有则拉取 | 生产环境、稳定版本 |
| `Always` | 每次都尝试拉取 | 开发环境、`:latest` 标签 |
| `Never` | 从不拉取，仅使用本地镜像 | 离线环境、预构建的镜像 |

**默认策略的隐含逻辑**：

| 镜像标签 | 默认 pullPolicy |
|---------|----------------|
| `:latest` 或无标签 | `Always` |
| 指定具体版本（如 `:v1.2.3`）| `IfNotPresent` |
| 指定了 digest（如 `@sha256:...`）| `IfNotPresent` |

**为什么 `:latest` 使用 `Always` 策略**？
`:latest` 标签是可变的，同一节点上缓存的 `:latest` 可能已经过时。使用 `Always` 确保获得最新版本。但生产环境应避免使用 `:latest` 标签——它会让部署行为不可预测。

**拉取机制底层**：
1. kubelet 通过 CRI 调用容器运行时的 `PullImage` 接口
2. 容器运行时根据 `image` 字段解析 registry 地址
3. 如果 registry 需要认证，kubelet 使用 Pod 所在命名空间的 `imagePullSecrets` 中的凭证
4. 拉取成功后解析 manifest，获取镜像层信息并下载

```yaml
spec:
  imagePullSecrets:             # 私有镜像仓库认证
  - name: my-registry-secret
  containers:
  - name: app
    image: private-registry.example.com/myapp:v1.0
    imagePullPolicy: IfNotPresent
```

## 3.3 环境变量

Kubernetes 提供多种环境变量注入方式，灵活度极高：

### 直接定义（env）

```yaml
env:
- name: APP_ENV
  value: "production"
- name: LOG_LEVEL
  value: "info"
- name: PYTHONUNBUFFERED
  value: "1"
```

### 从 ConfigMap 引用（valueFrom.configMapKeyRef）

```yaml
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
      name: app-config           # ConfigMap 名称
      key: app.mode              # ConfigMap 中的 key
      optional: true             # 设为可选，ConfigMap不存在时不报错
```

### 从 Secret 引用（valueFrom.secretKeyRef）

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secrets
      key: password
      optional: false
```

### 从 Pod 字段引用（valueFrom.fieldRef）

```yaml
env:
- name: MY_NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName     # Pod 被调度到的节点名
- name: MY_POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name     # Pod 名称
- name: MY_POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: MY_POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
- name: MY_POD_UID
  valueFrom:
    fieldRef:
      fieldPath: metadata.uid
- name: MY_SERVICE_ACCOUNT
  valueFrom:
    fieldRef:
      fieldPath: spec.serviceAccountName
```

### 从容器资源字段引用（valueFrom.resourceFieldRef）

```yaml
env:
- name: CPU_LIMIT
  valueFrom:
    resourceFieldRef:
      containerName: my-app
      resource: limits.cpu
      divisor: 1m                # 单位换算，不指定则为默认单位
- name: MEMORY_REQUEST
  valueFrom:
    resourceFieldRef:
      containerName: my-app
      resource: requests.memory
      divisor: 1Mi
```

### 批量注入（envFrom）

```yaml
envFrom:
- configMapRef:
    name: app-config             # 将 ConfigMap 所有key-value注入为环境变量
    optional: false
- secretRef:
    name: app-secrets            # 将 Secret 所有key-value注入为环境变量
```

**注意事项**：
- 环境变量在容器启动时确定，之后不可更改
- 如果用 `envFrom` 注入，ConfigMap/Secret 的 key 不能包含非法环境变量名（如连字符 `-`），否则该 key 会被跳过
- 大量环境变量会增加 API Server 压力（每次 Pod 创建时 kubelet 需要读取 ConfigMap/Secret）
- 环境变量不适合存储大段配置，此时应使用 Volume 挂载配置文件

## 3.4 资源限制

资源管理是 Kubernetes 稳定性的基石。`resources` 字段定义了容器的资源需求和限制：

```yaml
resources:
  requests:        # 调度时的资源保证（下限）
    memory: "128Mi"
    cpu: "250m"
  limits:          # 运行时的资源上限（上限）
    memory: "256Mi"
    cpu: "500m"
```

### CPU 资源单位

| 单位 | 含义 | 等值 |
|------|------|------|
| `1` | 1 个 vCPU / 1 个超线程 | 1 AWS vCPU = 1 GCP Core = 1 Azure vCore |
| `0.5` 或 `500m` | 0.5 个 CPU 核心 | 500 毫核（millicores）|
| `100m` | 0.1 个 CPU 核心 | 100 毫核 |
| `1m` | 最小单位 | 0.001 个 CPU 核心 |

**CPU 是可压缩资源**：当 CPU 使用超过 limits 时，容器不会被杀死，而是被**限流**（throttle）。CPU 的 `requests` 决定了调度和 CPU 时间片分配。

### 内存资源单位

| 单位 | 含义 |
|------|------|
| `128974848`、`129e6`、`129M`、`123Mi` | 都表示约 123 MiB |
| `1Gi` | 1,073,741,824 字节 |
| `1G` | 1,000,000,000 字节（十进制）|

Kubernetes 使用 Ki/K（2^10=1024）和 Mi/M（2^20）表示二进制单位，K/M/G（10^3）表示十进制单位。

**内存是不可压缩资源**：当内存使用超过 limits 时，容器会被 **OOMKilled**。这是 Linux 内核的 OOM Killer 机制，无法通过限流来缓解。

### requests 与 limits 的关系

```
limit (500m)  ────────────────────  可能被限流
          │                        
          │    超配区域（Overcommit）
          │    调度不保证这些资源
          │    
request(250m) ────────────────────  调度保证
          │                        
          │    保证资源
          │    
0         ────────────────────
```

**调度器只看 requests**：
- 节点的可分配资源 ≥ Pod 中所有容器 requests 之和
- limits 可以远大于 requests（超配），但不影响调度决策
- 如果设置了 limits 但未设置 requests，requests 默认等于 limits

**超配（Overcommit）**：
超配是云计算中常见的资源利用率优化手段。例如，一个 4C8G 的节点上可以调度多个 requests 总和为 4C8G 的 Pod，即使它们的 limits 总和远超 4C8G。这基于一个假设：大部分时间，容器不会同时达到其 limits。

超配的风险：当多个容器同时达到高负载时，可能导致资源争抢和 OOM。生产环境中，通常：
- 对关键服务设置 `requests == limits`，确保稳定性和可预测性
- 对非关键服务可适度超配，提高资源利用率

## 3.5 安全上下文（SecurityContext）

SecurityContext 可以在 Pod 级别和容器级别设置，用于控制运行时的安全参数：

### Pod 级别 SecurityContext

```yaml
spec:
  securityContext:               # Pod 级别，对所有容器生效
    runAsUser: 1000              # 以 UID 1000 运行所有容器
    runAsGroup: 3000             # 以 GID 3000 运行所有容器
    runAsNonRoot: true           # 必须以非 root 用户运行
    fsGroup: 2000                # 卷挂载时的文件系统组
    fsGroupChangePolicy: OnRootMismatch  # 文件系统组变更策略
    seccompProfile:              # Seccomp 配置
      type: RuntimeDefault       # RuntimeDefault/Localhost
    supplementalGroups: [4000]   # 补充组
    sysctls:                     # 内核参数
    - name: net.core.somaxconn
      value: "1024"
```

### 容器级别 SecurityContext

容器级别的设置在 Pod 级别的基础上进一步细化或覆盖：

```yaml
containers:
- name: app
  securityContext:                      # 容器级别
    runAsUser: 1000
    runAsNonRoot: true
    allowPrivilegeEscalation: false     # 禁止权限提升（setuid等）
    privileged: false                   # 非特权模式
    readOnlyRootFilesystem: true        # 根文件系统只读
    capabilities:
      drop:                             # 删除的 Linux Capabilities
      - ALL                             # 先删除所有
      add:                              # 添加需要的
      - NET_BIND_SERVICE                # 允许绑定 1024 以下端口
      - NET_RAW                         # 允许原始套接字
    procMount: Default                  # /proc 挂载类型 (Default/Unmasked)
```

### 关键字段详解

**runAsUser / runAsGroup**：
指定容器进程的 UID/GID。这是一个基本但关键的安全实践——不以 root（UID 0）运行容器。即使容器镜像默认以 root 运行，此设置也会覆盖。

```yaml
securityContext:
  runAsUser: 65534     # nobody 用户
  runAsGroup: 65534
  runAsNonRoot: true   # 如果容器以 root 启动，会因违反此设置而被阻止
```

**capabilities**：
Linux Capabilities 将传统的 root 全权限拆分为细粒度权限。Docker/Kubernetes 默认授予容器一组有限的 capabilities（`CHOWN`、`DAC_OVERRIDE`、`FSETID`、`FOWNER`、`MKNOD`、`NET_RAW`、`SETGID`、`SETUID`、`SETFCAP`、`SETPCAP`、`NET_BIND_SERVICE`、`SYS_CHROOT`、`KILL`、`AUDIT_WRITE`）。

```yaml
securityContext:
  capabilities:
    drop:
    - ALL              # 最佳实践：删除所有，只添加必须的
    add:
    - NET_BIND_SERVICE # 仅添加必需的能力
    - SYS_TIME         # 如果需要修改系统时间
```

**privileged**：
将 `privileged: true` 等同于给容器授予了所有 capabilities，并解除了设备 cgroup 的所有限制。这极其危险，仅用于需要直接访问主机硬件的场景（如 kube-proxy 的 IPVS 模式要求操作网络栈）。

**readOnlyRootFilesystem**：
将容器的根文件系统挂载为只读。这意味着容器不能向其镜像层写入任何内容。所有需要写入的数据必须写入挂载的 Volume。这是容器安全加固的基本措施之一。

```yaml
securityContext:
  readOnlyRootFilesystem: true
  # 如果应用需要写入临时文件，需要挂载 emptyDir 到 /tmp
volumeMounts:
- name: tmp
  mountPath: /tmp
```

**seccompProfile**：
Seccomp（Secure Computing Mode）限制进程可以发起的系统调用。`RuntimeDefault` 使用容器运行时的默认 seccomp profile（大多数运行时默认已经过滤掉了约 300+ 个危险系统调用）。`Localhost` 使用节点上的自定义 seccomp profile。

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault   # 推荐的安全设置
```

# 4.Pod生命周期

## 4.1 Pod Phase状态机

Pod 的 `status.phase` 是对 Pod 在其生命周期中所处位置的简单、高层次的总结。它是一个状态机，定义了 Pod 的可能状态：

```
                    ┌──────────┐
                    │  Pending │
                    └────┬─────┘
                         │ 调度成功 + 所有容器启动
                         ▼
                    ┌──────────┐
          ┌─────────│ Running  │─────────┐
          │         └────┬─────┘         │
          │              │               │
     所有容器成功退出   │          所有容器失败退出
          │              │               │
          ▼              │               ▼
    ┌──────────┐         │         ┌──────────┐
    │Succeeded │         │         │  Failed  │
    └──────────┘         │         └──────────┘
                         │
                    ┌────▼─────┐
                    │  Unknown │ (节点失联，状态无法获取)
                    └──────────┘
```

### Pending（等待中）

Pod 已被 Kubernetes 接受，但有一个或多个容器尚未创建。此阶段包括：
- 调度器正在为 Pod 选择节点
- 镜像正在下载
- Init 容器正在执行

Pod 可能**长时间处于 Pending** 的原因：
- 没有满足资源要求的节点（resources.requests 太大）
- 没有满足亲和性/反亲和性约束的节点
- 没有满足污点/容忍要求的节点
- 存储卷（PVC）未绑定
- 节点资源耗尽

### Running（运行中）

Pod 已被绑定到节点，所有容器至少有一个正在运行（或者正在启动/重启）。但 Running ≠ 可用——容器可能正在启动、可能还没通过 Readiness Probe。

### Succeeded（成功）

Pod 中所有容器已正常终止（退出码 0），且不会重启。这通常发生在 `restartPolicy: Never` 或 `OnFailure` 的 Job Pod 中。

### Failed（失败）

Pod 中所有容器已终止，至少有一个容器以非零退出码终止。这意味着系统不会重启这些容器。

### Unknown（未知）

无法获取 Pod 状态，通常因为 kubelet 与 API Server 通信中断。可能是节点宕机、网络分区等原因。

**状态转换的底层实现**：
- kubelet 定期向 API Server 发送心跳（node status update）和 Pod status update
- API Server 不主动修改 Pod status，所有 status 更新由 kubelet 发起
- 如果 kubelet 长时间未更新 Pod status，Pod 不会自动变为 Unknown——需要 Node Controller 检测到节点 NotReady 后，Pod 才会被标记为 Unknown（需配合 `--pod-eviction-timeout` 等参数）

## 4.2 Container State

每个容器在 Pod 的 `status.containerStatuses` 中有更细粒度的状态：

### Waiting

容器尚未运行，正在执行运行前所需的操作：

```yaml
state:
  waiting:
    reason: ImagePullBackOff    # 或 ContainerCreating / CrashLoopBackOff / ErrImagePull
    message: "Back-off pulling image..."
```

**常见 Waiting 原因**：

| Reason | 含义 |
|--------|------|
| `ContainerCreating` | 容器正在创建（拉取镜像、挂载卷等）|
| `ErrImagePull` | 拉取镜像失败（镜像不存在/认证失败/网络问题）|
| `ImagePullBackOff` | 拉取失败后的退避等待（`kubelet` 会按指数退避重试）|
| `CrashLoopBackOff` | 容器启动后立即崩溃，进入退避重启循环 |
| `CreateContainerConfigError` | ConfigMap/Secret 不存在或引用错误 |
| `CreateContainerError` | 容器创建失败 |

### Running

容器正在执行，没有出现问题：

```yaml
state:
  running:
    startedAt: "2024-01-15T10:00:00Z"
```

### Terminated

容器已结束运行：

```yaml
state:
  terminated:
    exitCode: 137            # 退出码
    signal: 9                # 终止信号（SIGKILL）
    reason: OOMKilled        # 终止原因
    message: "..."
    startedAt: "2024-01-15T10:00:00Z"
    finishedAt: "2024-01-15T11:00:00Z"
    containerID: "containerd://abc123..."
```

**常见 Terminated 原因**：

| Reason | 典型 exitCode | 含义 |
|--------|-------------|------|
| `Completed` | 0 | 正常退出 |
| `Error` | 非0 | 异常退出 |
| `OOMKilled` | 137 | 被 OOM Killer 杀死（exit code 137 = 128 + 9 (SIGKILL)）|
| `ContainerCannotRun` | 126/127 | 容器无法执行 |
| `DeadlineExceeded` | - | 超时 |

**exitCode 137 = OOMKilled** 的意义：
- 128 表示进程因信号而终止
- 9 是 SIGKILL 的信号编号
- 137 = 128 + 9，这个 exit code 明确表明容器因内存超限被内核 OOM Killer 杀死

## 4.3 Pod Conditions

Pod Conditions 提供了比 Phase 更详细、更结构化的 Pod 状态信息：

```yaml
status:
  conditions:
  - type: PodScheduled
    status: "True"
    lastProbeTime: null
    lastTransitionTime: "2024-01-15T10:00:00Z"
    reason: "Scheduled"
    message: "Successfully assigned default/my-pod to worker-node-1"
  - type: Initialized
    status: "True"
    lastTransitionTime: "2024-01-15T10:00:05Z"
  - type: ContainersReady
    status: "True"
    lastTransitionTime: "2024-01-15T10:00:30Z"
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-01-15T10:00:30Z"
```

### 四种核心 Condition

| Condition Type | 含义 | 依赖于 |
|---------------|------|--------|
| `PodScheduled` | Pod 已被调度到节点 | 调度器已将 `spec.nodeName` 赋值 |
| `Initialized` | 所有 Init 容器成功完成 | Init 容器全部以 0 退出 |
| `ContainersReady` | Pod 中所有容器就绪（通过 Readiness Probe）| 所有容器 Readiness Probe 通过 |
| `Ready` | Pod 可以接收流量 | `ContainersReady` + 网络策略 + 其他条件 |

**Conditions 的时间线**：

```
Pod 创建
  │
  ▼
PodScheduled = True        ← 调度器分配节点
  │
  ▼
Initialized = True         ← Init 容器执行完成
  │
  ▼
ContainersReady = True     ← 所有容器 Readiness Probe 通过
  │
  ▼
Ready = True               ← Pod 开始接受 Service 流量
```

**Ready vs ContainersReady 的区别**：
- `ContainersReady` 只关心容器级别的就绪状态
- `Ready` 是 Pod 级别的"可用"判断，可以包含其他自定义条件（如自定义网络策略是否就绪）
- Service 的 Endpoint 控制器使用 `Ready` Condition 来决定是否将 Pod 加入 Service 的后端列表

## 4.4 生命周期钩子

Kubernetes 提供了两个容器生命周期钩子：`PostStart` 和 `PreStop`。

### PostStart

容器创建后**立即**执行。注意：PostStart 和 ENTRYPOINT 的执行顺序是**并行的**，不能保证 PostStart 在 ENTRYPOINT 之前完成。

```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo 'Container started at $(date)' >> /var/log/startup.log"]
```

**重要陷阱**：如果 PostStart 钩子挂起或失败，容器会进入 `Waiting` 状态，`reason` 为 `PostStartHookError`。kubelet 会根据 `restartPolicy` 决定是否重启容器。

### PreStop

容器终止**之前**执行。它是**同步**的——kubelet 会等待 PreStop 执行完成后才发送 SIGTERM 给容器主进程。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "nginx -s quit && while killall -0 nginx; do sleep 1; done"]
```

PreStop 的重要特性：
- kubelet 先执行 PreStop 钩子，然后才发送 SIGTERM
- PreStop 必须在 `terminationGracePeriodSeconds` 时间内完成，否则容器被强制 SIGKILL
- 如果 PreStop 需要很长时间（如排空连接），要确保 `terminationGracePeriodSeconds` 足够大

### 三种钩子处理器

```yaml
# 1. exec - 在容器内执行命令
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo hello"]

# 2. httpGet - 发送 HTTP GET 请求
lifecycle:
  preStop:
    httpGet:
      path: /shutdown
      port: 8080
      host: localhost           # 默认是 Pod IP
      scheme: HTTP              # HTTP/HTTPS
      httpHeaders:
      - name: X-Custom-Header
        value: "pre-stop"

# 3. tcpSocket - 检查 TCP 端口
lifecycle:
  preStop:
    tcpSocket:
      port: 8080
      host: localhost
```

## 4.5 终止流程

Pod 的终止是一个优雅但有时限的过程。完整的终止流程如下：

```
外部请求删除 Pod (kubectl delete / API调用 / controller缩减副本)
  │
  ▼
API Server 设置 Pod deletionTimestamp 和 deletionGracePeriodSeconds
  │ (如果 gracePeriodSeconds 未指定，使用 pod.spec.terminationGracePeriodSeconds, 默认 30s)
  │
  ▼
Endpoint Controller 从 Service 后端列表移除 Pod IP
  │ (Pod 状态变为 Terminating)
  │
  ▼
kubelet 检测到 Pod 的 deletionTimestamp 被设置
  │
  ├──► 1. 执行 PreStop 钩子（如果定义了）
  │       │
  │       ▼
  ├──► 2. 发送 SIGTERM 信号给 Pod 中所有容器的主进程（PID 1）
  │       │
  │       │ (应用开始优雅关闭：关闭监听端口、完成正在处理的请求、清理资源)
  │       │
  │       ▼
  │   等待 terminationGracePeriodSeconds 时间
  │       │
  │       │ 如果进程在此时间内退出 → 正常终止流程
  │       │ 如果超时 → 发送 SIGKILL (-9) 强制终止
  │       ▼
  └──► 3. kubelet 向 API Server 更新 Pod 状态，移除 finalizer
          │
          ▼
       API Server 清理 Pod 对象（从 etcd 中删除）
```

**关键参数 `terminationGracePeriodSeconds`**：
- 默认值：30 秒
- 可以通过 `kubectl delete pod <name> --grace-period=<seconds>` 覆盖
- 设为 0 表示立即强制删除（跳过 PreStop，直接 SIGKILL）
- 应用需要在此时间内优雅关闭

**SIGTERM → SIGKILL 的深层原理**：
在 Linux 中，SIGTERM（信号 15）是一个可以被捕获、阻塞或忽略的"温和"信号，应用收到后可以执行清理工作。而 SIGKILL（信号 9）是终极武器，内核直接终止进程，进程无法捕获或忽略——它保证了进程最终会被终止，即使应用代码未能正确处理 SIGTERM。

## 4.6 RestartPolicy

`restartPolicy` 控制 Pod 内容器退出后的行为。注意：重启是**Pod 本地操作**，由 kubelet 执行，不涉及重新调度。

### 三种策略

```yaml
spec:
  restartPolicy: Always     # 默认值
  # restartPolicy: OnFailure
  # restartPolicy: Never
```

| restartPolicy | 容器退出后行为 | 适用工作负载 |
|--------------|-------------|------------|
| `Always` | 总是重启 | Deployment、StatefulSet、DaemonSet（长期运行服务）|
| `OnFailure` | 仅非零退出码时重启 | Job（批处理任务）|
| `Never` | 从不重启 | 一次性 Job、初始化任务 |

### 退避重启延迟（Back-off）

kubelet 使用指数退避（exponential back-off）策略重启失败容器，避免频繁重启导致系统负载过高：

```
首次崩溃 → 立即重启 (0s)
再次崩溃 → 等待 10s
再次崩溃 → 等待 20s
再次崩溃 → 等待 40s
再次崩溃 → 等待 80s
...
最大 → 等待 300s (5分钟)
```

退避时间上限为 300 秒（5 分钟），重置条件：容器成功运行 10 分钟以上。

当容器进入 CrashLoopBackOff 状态时，这意味着容器被重启了多次但仍然失败。此时：
- `status.containerStatuses[].state.waiting.reason = "CrashLoopBackOff"`
- kubelet 会按照退避时间等待，在等待期间容器不会被重启
- 这是一个保护机制，避免反复启动失败容器浪费资源

# 5.容器探针

## 5.1 Liveness Probe（存活探针）

Liveness Probe 回答的问题是：**"容器是否还活着？还是已经死锁/卡死？"**

如果 Liveness Probe 失败，kubelet 会**杀死容器**并按照 `restartPolicy` 重启。Liveness 探针的失败意味着容器已经进入不可恢复的状态——应用进程可能还在运行，但已经无法正常响应请求（如死锁、无限循环、内存泄漏导致的 OOM）。

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: "liveness-check"
  initialDelaySeconds: 30    # 首次探测延迟
  periodSeconds: 10          # 探测间隔
  timeoutSeconds: 5          # 超时时间
  failureThreshold: 3        # 连续失败次数后才判定失败
  successThreshold: 1        # 连续成功次数后才判定成功
```

**Liveness Probe 的工作原理**：
1. kubelet 在 Pod 所在节点的容器运行时层执行探测
2. 如果探测失败，kubelet 累计失败计数
3. 达到 `failureThreshold` 后，kubelet 认为容器不健康
4. kubelet 停止容器（相当于发送 SIGTERM，等待 grace period，然后 SIGKILL）
5. 根据 `restartPolicy`，kubelet 重新启动容器

**重要警告**：Liveness Probe 的探测逻辑应该**独立于外部依赖**。如果你的 `/healthz` 端点检查了数据库连接，那么数据库故障会导致 Liveness Probe 失败，进而触发容器重启——而重启容器并不能修复数据库故障，反而会加剧系统不稳定。详见 5.6 节。

## 5.2 Readiness Probe（就绪探针）

Readiness Probe 回答的问题是：**"容器是否准备好接收流量？"**

如果 Readiness Probe 失败，kubelet 不会杀死容器，而是将 Pod 从关联的 Service 后端列表中移除——即 Pod 不再接收流量，但它继续运行。探针成功后，Pod 会被重新加入 Service 后端列表。

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 1
```

**Readiness Probe 的使用场景**：
- 应用启动时间长，需要在完全初始化后才接收流量
- 应用需要预热（如加载缓存、建立数据库连接池）
- 应用过载，暂时无法处理更多请求（实现"背压"机制）
- 应用依赖的外部服务暂时不可用（如数据库连接丢失，应用需要暂时摘除流量）

**Readiness Probe 如何影响流量**：
1. Service 的 Endpoints 控制器持续监视 Pod 的 Ready Condition
2. Pod 的 Ready Condition 由 ContainersReady（所有容器 Readiness Probe 通过）决定
3. 当 Readiness Probe 失败时，Endpoint 控制器从 Endpoints 对象中移除 Pod IP
4. kube-proxy 更新 iptables/IPVS 规则，不再将流量转发到该 Pod

## 5.3 Startup Probe（启动探针）

Startup Probe 是 Kubernetes 1.16 引入的特性，回答的问题是：**"容器是否已完成启动？"**

它的出现解决了 Liveness Probe 的一个重要问题：**慢启动容器**。对于启动时间长的容器（如 Java 应用、加载大模型的 ML 服务），如果 Liveness Probe 的 `initialDelaySeconds` 设置不当，容器可能在启动过程中就被误判为失败而被杀死。

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30       # 允许最多 30 次失败（即 30 * periodSeconds 的启动窗口）
  periodSeconds: 10          # 每 10 秒探测一次
  # 总共提供 300 秒（5 分钟）的启动窗口

# 如果定义了 Startup Probe，则在它成功之前，Liveness 和 Readiness 探测都被禁用
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
```

**Startup Probe 的工作流程**：
1. 容器启动后，Startup Probe 立即开始探测（没有 initialDelaySeconds？注意 Startup Probe 不支持 initialDelaySeconds）
2. 在 Startup Probe 成功之前，Liveness Probe 和 Readiness Probe 都被暂停
3. Startup Probe 失败达到 `failureThreshold` 后，容器被杀死并重启
4. Startup Probe 成功一次后，Liveness Probe 接管容器的存活检查

这样设计的好处是：你不需要精确估算容器的启动时间。你只需设置一个足够宽松的 `failureThreshold * periodSeconds`，Startup Probe 会一直等待，直到容器启动完成。对于启动时间不确定的容器（如首次启动需要初始化数据的应用），这比精确设置 `initialDelaySeconds` 更加健壮。

## 5.4 探针配置参数详解

所有探针共享相同的配置参数结构：

```yaml
probe:
  initialDelaySeconds: 0     # 首次探测前的等待时间（秒）。Startup Probe 不支持此字段
  periodSeconds: 10          # 探测间隔（秒），默认 10，最小 1
  timeoutSeconds: 1          # 探测超时时间（秒），默认 1，最小 1
  successThreshold: 1        # 连续成功次数（Liveness/Startup 必须为 1）
  failureThreshold: 3        # 连续失败次数，默认 3
```

**各参数的深层含义**：

| 参数 | Liveness | Readiness | Startup |
|------|----------|-----------|---------|
| `initialDelaySeconds` | ✓ | ✓ | ✗（不支持）|
| `periodSeconds` | ✓ | ✓ | ✓ |
| `timeoutSeconds` | ✓ | ✓ | ✓ |
| `successThreshold` | 固定为 1 | 任意值（通常 1）| 固定为 1 |
| `failureThreshold` | ✓ | ✓ | ✓ |

**successThreshold 的差异**：
- Liveness Probe 和 Startup Probe 的 `successThreshold` 必须为 1——探测成功一次就意味着容器健康
- Readiness Probe 的 `successThreshold` 可以大于 1——意味着容器需要连续多次通过就绪检查才会被视为就绪。这用于处理"抖动"（flapping）问题

**时间计算公式**：

```
Liveness 探测失败到容器被杀死的时间：
= periodSeconds × failureThreshold + timeoutSeconds × failureThreshold
≈ periodSeconds × failureThreshold

例如：periodSeconds=10, failureThreshold=3
容器故障后约 30 秒被杀死重启

Readiness 探测失败到 Pod 被摘除流量的时间：
≈ periodSeconds × failureThreshold

例如：periodSeconds=5, failureThreshold=3
容器变为未就绪约 15 秒后从 Service 移除
```

## 5.5 探针实现方式对比

### exec

在容器内执行命令，退出码为 0 表示成功：

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

**优点**：
- 最灵活，可以执行任意检测逻辑
- 可以检查容器内部状态（文件存在性、进程状态等）
- 不依赖应用暴露 HTTP 端点

**缺点**：
- 每次探测启动一个新进程，有额外开销
- 命令本身可能成为故障源（依赖环境、shell 等）
- 需要将检测脚本打包到镜像中

### httpGet

发送 HTTP GET 请求，2xx 或 3xx 状态码表示成功：

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
    scheme: HTTPS              # HTTP 或 HTTPS
    httpHeaders:
    - name: X-Check-Type
      value: "readiness"
```

**优点**：
- 最常用的方式，语义明确
- 探测由 kubelet 发起，不消耗容器资源（相比 exec）
- 支持自定义 Header（可用于区分 Metric 检查和 Liveness 检查）

**缺点**：
- 要求应用暴露 HTTP 端点
- 探测端点的实现可能引入依赖检查（需注意区分）
- HTTPS 涉及的证书验证可能引入额外复杂性

### tcpSocket

检查 TCP 端口是否可连接：

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 10
```

**优点**：
- 最简单的检查方式，无需应用暴露任何特殊端点
- 适用于 TCP 服务（数据库、缓存等）
- 开销极小

**缺点**：
- 只能验证端口是否可连接，不能验证应用是否真正工作
- 端口可连接 ≠ 应用健康（例如应用线程池满，但端口仍可连接）

### gRPC（Kubernetes 1.24+）

```yaml
readinessProbe:
  grpc:
    port: 50051
    service: your.package.HealthService  # gRPC 服务名（可选）
  initialDelaySeconds: 5
```

**优点**：
- 原生支持 gRPC 健康检查协议（grpc-health-probe）
- 不需要在镜像中安装额外的健康检查工具
- 直接由 kubelet 的 gRPC 客户端发起

**缺点**：
- Kubernetes 1.24+ 才稳定（1.23 为 Alpha）
- 容器运行时需要支持此特性
- 目前不支持 TLS 认证

### 四种方式对比总结

| 探测方式 | 开销 | 准确性 | 侵入性 | 适用场景 |
|---------|------|--------|--------|---------|
| `exec` | 高（启动进程）| 高 | 低 | 灵活自定义检测 |
| `httpGet` | 低（仅 HTTP 请求）| 中-高 | 高（需要暴露端点）| Web 应用 |
| `tcpSocket` | 极低 | 低（仅端口可达）| 无 | TCP 服务 |
| `gRPC` | 低 | 高 | 高 | gRPC 服务 |

## 5.6 探针最佳实践与常见陷阱

### 最佳实践

**1. Liveness Probe 不要检查外部依赖**

```yaml
# 错误示例：Liveness 检查数据库连接
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  # /healthz 端点检查了数据库连接...
  # 数据库故障 → Liveness 失败 → 容器被重启 → 重启不能修复数据库问题
  # 更糟的是，大量容器同时重启会导致"雪崩"

# 正确做法：Liveness 只检查容器自身的健康
livenessProbe:
  httpGet:
    path: /livez
    port: 8080
  # /livez 只检查应用进程是否响应，不检查数据库等外部依赖
```

**2. 合理设置 initialDelaySeconds 或使用 Startup Probe**

```yaml
# 保守做法：为启动时间设置充足的缓冲
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
# 提供 300 秒的启动窗口

livenessProbe:
  httpGet:
    path: /livez
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
# 启动完成后，30 秒内无响应判定为故障
```

**3. 为 Liveness 和 Readiness 设计不同的检查逻辑**

```yaml
# Liveness：只检查进程级健康
livenessProbe:
  httpGet:
    path: /livez       # 只检查：应用是否响应？线程池是否满？
    port: 8080

# Readiness：检查是否准备好接收流量
readinessProbe:
  httpGet:
    path: /readyz      # 检查：缓存是否预热？数据库连接池是否就绪？
    port: 8080
```

**4. 避免过于激进的探测频率**

```yaml
# 过于激进：1秒检测一次，可能给应用和 kubelet 带来压力
periodSeconds: 1       # 不推荐

# 合理：5-10 秒间隔
periodSeconds: 10      # 推荐

# 慢速服务可以适当增大
periodSeconds: 30      # 适用于判断稳定的服务
```

### 常见陷阱

**陷阱1：Liveness Probe 与 Readiness Probe 的混淆**

```
错误认知：Liveness 失败 → Pod 被重启
正确理解：Liveness 失败 → 容器被本地 kubelet 杀死并重启（在同一个节点上）
          这不是 Pod 级别的重新调度！Pod IP 保持不变！

错误认知：Readiness 失败 → Pod 被删除
正确理解：Readiness 失败 → Pod 从 Service 后端列表移除
          Pod 继续运行，不受影响（除了不接收流量）
```

**陷阱2：探针配置不当导致"重启风暴"**

```yaml
# 危险的配置
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5      # 启动时间可能不足
  periodSeconds: 3            # 过于频繁
  failureThreshold: 2         # 过于敏感（6秒判定故障）
  timeoutSeconds: 1
```

这种配置下，即使应用短暂 GC 暂停，也可能触发重启。在生产中可能导致：
- 反复重启 CPU 消耗
- Service 后端频繁变动
- 请求被中断

**陷阱3：exec 探针的副作用**

```yaml
# 危险的 exec 探针
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - "curl http://localhost:8080/health"  # curl 可能不存在于镜像中
```

exec 探针每次执行都会创建新进程。频繁的 exec 探测（如每 3 秒一次）会在容器内产生大量进程创建开销，并且命令本身需要的工具（如 `curl`、`wget`）必须存在于容器镜像中。

**陷阱4：探针 HTTP 端点的实现问题**

探针端点应该轻量且快速。一个常见的错误是在探针端点中执行重操作：

```python
# 错误：/healthz 端点执行了耗时的数据库查询
@app.route('/healthz')
def healthz():
    db.execute("SELECT COUNT(*) FROM large_table")  # 不！
    return "OK", 200

# 正确：只做简单检查
@app.route('/healthz')
def healthz():
    return "OK", 200
```

# 6.Init容器

## 6.1 Init容器概念与用途

Init 容器是 Pod 中的特殊容器，在应用容器启动**之前**运行。它们用于执行初始化、设置、验证等前置操作。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: init-db-check
    image: busybox:1.36
    command: ["sh", "-c", "until nslookup mydb; do echo 'waiting for DB...'; sleep 2; done"]
  - name: init-migration
    image: myapp:latest
    command: ["python", "manage.py", "migrate"]
    env:
    - name: DB_HOST
      value: "mydb-service"
  containers:
  - name: app
    image: myapp:latest
    command: ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

**典型用途**：

1. **等待依赖服务就绪**：等待数据库、缓存、消息队列等服务可用后再启动主容器
2. **数据库迁移/Schema 初始化**：在应用启动前执行 schema 迁移
3. **文件权限设置**：修改共享卷中的文件权限/所有权（`chown`, `chmod`）
4. **从外部获取资源**：从 Git 仓库拉取代码、从对象存储下载配置
5. **注册服务**：向服务注册中心注册 Pod
6. **环境检查**：验证所需的配置文件存在、环境变量正确

## 6.2 Init容器执行顺序与特性

### 执行顺序

Init 容器严格按照定义的顺序**串行**执行，且每个 Init 容器必须成功退出后才能执行下一个：

```
Pod 创建
  │
  ▼
拉取所有 Init 容器镜像（可并行）
  │
  ▼
Init Container 1 启动 ──► 完成（exit 0）
  │
  ▼
Init Container 2 启动 ──► 完成（exit 0）
  │
  ▼
Init Container N 启动 ──► 完成（exit 0）
  │
  ▼
所有应用容器并行启动
```

### 关键特性

| 特性 | Init 容器 | 普通容器 |
|------|----------|---------|
| 执行时机 | 应用容器之前 | 应用容器启动后 |
| 执行顺序 | 严格串行 | 并行 |
| 镜像 | 可以与主容器不同 | - |
| restartPolicy 影响 | 受 `restartPolicy` 影响 | 受 `restartPolicy` 影响 |
| 资源限制 | 独立设置 | 独立设置 |
| Volume 共享 | 可以挂载相同 Volume | 可以挂载相同 Volume |
| 生命周期钩子 | 不支持 | 支持 |
| 探针 | 不支持 Liveness/Readiness/Startup | 支持 |
| 环境变量 | 独立设置 | 独立设置 |
| SecurityContext | 独立设置 | 独立设置 |

**重要规则**：
- 如果任何一个 Init 容器失败，kubelet 会根据 `restartPolicy` 决定是否重启它
- 如果 `restartPolicy: Never`，Init 容器失败会导致整个 Pod 失败
- Init 容器失败后的重试也是按退避策略执行的
- Init 容器不支持 `lifecycle` 钩子和探针，因为它们的执行本身就是一次性的
- Init 容器每次 Pod 重启都会重新执行（不同于应用容器重启——应用容器重启只重启该容器，而 Pod 重启意味着所有容器重新创建）

**Init 容器的特殊优势**：
- 可以使用与主容器不同的镜像（如使用 `busybox` 做网络检查，使用 `alpine/git` 拉取代码）
- 可以访问主容器因安全策略无法访问的资源（如 ConfigMap 创建前的初始化）
- 可以在不修改主容器镜像的情况下完成前置操作（关注点分离）

## 6.3 Init容器 vs 普通容器 vs Sidecar容器

| 维度 | Init 容器 | 普通容器 | Sidecar 容器（1.28+）|
|------|----------|---------|-------------------|
| 运行时机 | 主容器之前 | Pod 运行期间 | 主容器之前开始，持续运行 |
| 生命周期 | 一次性（运行完退出）| 持续运行 | 持续运行 |
| 重启行为 | 失败按 restartPolicy 重启 | 按 restartPolicy 重启 | 失败按独立 restartPolicy 重启 |
| 探针支持 | 不支持 | 支持 | 不支持 |
| 典型用途 | 初始化、验证 | 业务逻辑 | 日志收集、代理、监控 |

### 何时使用 Init 容器而非 Sidecar

使用 Init 容器：
- 任务是一次性的（数据库迁移、文件准备、依赖等待）
- 任务必须在主容器启动前完成
- 不需要持续运行

使用 Sidecar：
- 需要持续运行（如日志收集、监控代理、服务网格代理）
- 需要与主容器共享生命周期

## 6.4 Sidecar容器

Kubernetes 1.28 引入了 Sidecar 容器的原生支持（Alpha，1.29 进入 Beta）。Sidecar 容器通过 Init 容器定义，但设置 `restartPolicy: Always`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-with-sidecar
spec:
  initContainers:
  - name: envoy-proxy       # Sidecar 容器
    image: envoyproxy/envoy:v1.28
    restartPolicy: Always   # 关键：设为 Always
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
  volumes:
  - name: envoy-config
    configMap:
      name: envoy-config
```

**Sidecar 容器的特性**：
- Sidecar 容器在主容器启动之前启动（保持与 Init 容器一致的启动顺序）
- Sidecar 容器持续运行，与 Pod 同生命周期
- Sidecar 容器的 `restartPolicy` 独立于 Pod 的 `restartPolicy`
- Sidecar 容器不支持探针（与 Init 容器相同）
- Pod 终止时，Sidecar 容器在应用容器之后收到 SIGTERM（反向顺序）

**启动和关闭顺序**：

```
启动顺序：
  Sidecar 容器 → 主容器
  （Sidecar 先启动，确保代理/日志等服务已就绪）

关闭顺序：
  主容器 → Sidecar 容器
  （主容器先关闭，Sidecar 在其后关闭，确保能收集完最后的日志/流量）
```

**Sidecar 容器 vs 传统 Sidecar**：
在 1.28 之前，Sidecar 是一个非正式的实践——通过普通容器实现。1.28+ 的原生 Sidecar 容器提供了更好的生命周期顺序保证和独立的 restartPolicy。对于 Istio 等服务网格的场景，这能避免 Sidecar 代理还没启动时，主容器就已经收到流量的问题。

# 7.Pod调度约束

## 7.1 nodeSelector

`nodeSelector` 是最简单的节点选择机制，通过节点标签将 Pod 调度到特定节点：

```yaml
spec:
  nodeSelector:
    disktype: ssd
    kubernetes.io/os: linux
```

节点需要有匹配的标签：

```bash
kubectl label nodes worker-node-1 disktype=ssd
kubectl label nodes worker-node-2 disktype=hdd
```

**工作原理**：
调度器的预选阶段（Filtering/Feasibility check）会检查节点是否拥有所有 `nodeSelector` 指定的标签。如果没有任何节点匹配，Pod 会一直处于 Pending 状态，并产生事件：

```
Warning  FailedScheduling  pod/my-pod  0/3 nodes are available:
  1 node(s) didn't match Pod's node affinity/selector.
  2 node(s) didn't match node selector.
```

**局限性**：
- 只支持简单的等值匹配（key==value）
- 不支持表达式逻辑（不能表达 "key in (a, b, c)"）
- 不支持"软"要求（是硬性条件）

对于更复杂的调度需求，应使用 `nodeAffinity`。

## 7.2 nodeName

`nodeName` 是最直接但最不灵活的调度方式——跳过调度器，直接将 Pod 绑定到指定节点：

```yaml
spec:
  nodeName: worker-node-1
```

**工作机制**：
1. 当 Pod spec 中有 `nodeName` 时，调度器**跳过**该 Pod 的调度
2. kubelet 发现 `spec.nodeName` 等于自己的节点名，直接在该节点上启动 Pod
3. 即使节点不可调度（cordon），Pod 仍会被强制执行

**注意事项**：
- 绕过了调度器的所有检查（资源是否充足、污点检查等）
- 绕过了亲和性/反亲和性规则
- 如果指定节点不存在或不可达，Pod 将无法创建
- 一般仅用于测试、调试或特殊场景

## 7.3 亲和性与反亲和性

亲和性（Affinity）提供了丰富的调度约束表达能力：

### nodeAffinity（节点亲和性）

```yaml
spec:
  affinity:
    nodeAffinity:
      # 硬性要求：必须满足才能调度
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - zone-a
            - zone-b
          - key: node.kubernetes.io/instance-type
            operator: NotIn
            values:
            - t3.nano

      # 软性偏好：优先满足，但不强制
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80                          # 权重 1-100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 20
        preference:
          matchExpressions:
          - key: cpu-arch
            operator: In
            values:
            - arm64
```

**关键语义**：

| 表达式 | 含义 |
|--------|------|
| `requiredDuringScheduling...` | 硬性约束。调度时必须满足，否则 Pod 不调度 |
| `...IgnoredDuringExecution` | 调度后不再检查。节点标签变化不影响已运行的 Pod |
| `preferredDuringScheduling...` | 软性偏好。调度时尽量满足，但不是必须 |
| `...RequiredDuringExecution` | （计划中，未实现）调度后标签变化会导致驱逐 |

**nodeSelectorTerms 的逻辑**：
- 多个 `nodeSelectorTerms` 之间是 **OR** 关系
- 每个 `nodeSelectorTerms` 内的 `matchExpressions` 之间是 **AND** 关系
- 即：只要满足任意一个 `nodeSelectorTerms`（其中的所有表达式都匹配），该节点就满足条件

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:        # Term 1: zone-A 的 SSD 节点
    - key: zone
      operator: In
      values: [zone-a]
    - key: disktype
      operator: In
      values: [ssd]
  - matchExpressions:        # Term 2: zone-B 的任何磁盘类型节点
    - key: zone
      operator: In
      values: [zone-b]
  # 逻辑：(zone=zone-a AND disktype=ssd) OR (zone=zone-b)
```

### podAffinity（Pod亲和性）

Pod 亲和性基于**已运行 Pod 的标签**来调度，而非节点标签。表达"我想靠近某些 Pod"：

```yaml
spec:
  affinity:
    podAffinity:
      # 硬性要求：必须与带有 app=cache 标签的 Pod 在同一拓扑域
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [cache]
        topologyKey: topology.kubernetes.io/zone
        # 含义：Pod 必须被调度到与 app=cache Pod 相同 availability zone 的节点

      # 软性偏好：尽量靠近
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web-server
          topologyKey: kubernetes.io/hostname
          # 含义：尽量与 app=web-server 的 Pod 在同一个节点
```

**topologyKey 的含义**：
`topologyKey` 定义了一个"拓扑域"的边界。它的值是节点标签的 key：
- `topology.kubernetes.io/zone`：按可用区分组
- `topology.kubernetes.io/region`：按区域分组
- `kubernetes.io/hostname`：按主机名分组（即同节点）

### podAntiAffinity（Pod反亲和性）

Pod 反亲和性表达"我不想和某些 Pod 在一起"，用于高可用部署：

```yaml
spec:
  affinity:
    podAntiAffinity:
      # 硬性要求：不与同 app 的其他 Pod 在同一节点
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: [my-app]
        topologyKey: kubernetes.io/hostname

      # 软性偏好：尽量不在同一 zone
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-app
          topologyKey: topology.kubernetes.io/zone
```

**典型应用**：
- 将同一应用的多副本分散到不同节点/机架/可用区，实现高可用
- 避免资源密集型 Pod 集中到同一节点

## 7.4 TopologySpreadConstraints

`TopologySpreadConstraints` 是 Kubernetes 1.19 稳定的特性，提供了比 Pod 反亲和性更灵活的"均匀分布"能力：

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                                    # 最大倾斜度
    topologyKey: topology.kubernetes.io/zone      # 拓扑域
    whenUnsatisfiable: DoNotSchedule              # 不满足时的行为
    labelSelector:                                # 选择要计算分布的 Pod
      matchLabels:
        app: my-app

  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway             # 即使不均匀也调度
    labelSelector:
      matchLabels:
        app: my-app
```

**核心参数**：

| 参数 | 说明 |
|------|------|
| `maxSkew` | 最大不均衡度。任意两个拓扑域中匹配 Pod 数量之差不能超过此值 |
| `topologyKey` | 拓扑域划分的依据（节点标签的 key）|
| `whenUnsatisfiable` | `DoNotSchedule`（不满足时不调度）/ `ScheduleAnyway`（即使不满足也调度）|
| `labelSelector` | 选择哪些 Pod 参与分布计算 |

**maxSkew 计算示例**：

```
3 个 zone (zone-a, zone-b, zone-c)，需要调度 6 个 Pod
maxSkew = 1, whenUnsatisfiable = DoNotSchedule

zone-a: 2 Pods  (2 - 2 = 0 ≤ 1) ✓
zone-b: 2 Pods  (2 - 2 = 0 ≤ 1) ✓
zone-c: 2 Pods  (2 - 2 = 0 ≤ 1) ✓

如果 zone-a 已有 3 个 Pod，zone-b 有 1 个 Pod：
zone-a: 3 Pods  (3 - 1 = 2 > 1) ✗ 不符合 maxSkew
新的 Pod 必须调度到 zone-b 或 zone-c
```

**与 podAntiAffinity 的区别**：
- `podAntiAffinity`：绝对的排斥关系（"不要和X在一起"），粒度是"有"还是"没有"
- `topologySpreadConstraints`：分布关系（"要均匀分布在X上"），粒度是"多少个"
- 如果有 100 个节点，`podAntiAffinity` 只能确保每个节点最多 1 个 Pod；`topologySpreadConstraints` 可以确保每个 zone 的 Pod 数量偏差不超过 maxSkew，单个节点可以有多个 Pod

## 7.5 污点与容忍

### Taints（污点）

污点标注在**节点**上，排斥不符合条件的 Pod：

```bash
# 给节点添加污点
kubectl taint nodes worker-node-1 dedicated=experimental:NoSchedule

# 移除污点
kubectl taint nodes worker-node-1 dedicated=experimental:NoSchedule-

# 查看节点污点
kubectl describe node worker-node-1 | grep Taints
```

**污点格式**：`key=value:Effect`

| Effect | 行为 |
|--------|------|
| `NoSchedule` | 不允许新 Pod 调度到此节点（已运行的 Pod 不受影响）|
| `PreferNoSchedule` | 尽量不调度到该节点（软性约束）|
| `NoExecute` | 不允许新 Pod 调度，且驱逐已在运行的 Pod |

**内置污点**：

| 污点 | 含义 |
|------|------|
| `node.kubernetes.io/not-ready:NoExecute` | 节点 NotReady 时添加 |
| `node.kubernetes.io/unreachable:NoExecute` | 节点不可达时添加 |
| `node.kubernetes.io/disk-pressure:NoSchedule` | 磁盘压力 |
| `node.kubernetes.io/memory-pressure:NoSchedule` | 内存压力 |
| `node.kubernetes.io/pid-pressure:NoSchedule` | PID 压力 |
| `node.kubernetes.io/network-unavailable:NoSchedule` | 网络不可用 |
| `node.kubernetes.io/unschedulable:NoSchedule` | 节点不可调度（cordon）|

### Tolerations（容忍）

容忍标注在**Pod**上，允许 Pod 调度到有污点的节点：

```yaml
spec:
  tolerations:
  # 容忍特定污点
  - key: "dedicated"
    operator: "Equal"       # Equal 或 Exists
    value: "experimental"
    effect: "NoSchedule"

  # 容忍任何包含此 key 的污点
  - key: "dedicated"
    operator: "Exists"      # 不检查 value
    effect: "NoSchedule"

  # 容忍所有类型的 effect
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300  # 容忍 300 秒（仅对 NoExecute 有效）

  # 容忍一切污点（谨慎使用）
  - operator: "Exists"
```

**`tolerationSeconds` 机制**：
对于 `NoExecute` 污点，`tolerationSeconds` 定义了 Pod 在节点被添加该污点后可以继续运行的时间：

```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
# 含义：节点 NotReady 后，Pod 可以继续运行 300 秒（5分钟），之后被驱逐
```

如果没有 `tolerationSeconds`，Pod 会一直容忍该污点（永不被驱逐）。如果不指定 `NoExecute` 容忍，Pod 会立即被驱逐。

**容忍匹配规则**：
1. 如果 `operator` 是 `Exists`，不需要 `value`（匹配所有 value）
2. 如果 `operator` 是 `Equal`，`value` 必须完全匹配
3. 空 `effect` 匹配所有 effect
4. 空 `key` 和 `operator: Exists` 匹配所有污点

## 7.6 优先级与抢占

### PriorityClass

PriorityClass 定义了 Pod 的优先级，决定在资源不足时哪些 Pod 被驱逐，以及哪些 Pod 可以抢占其他 Pod 的位置：

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000                          # 优先级数值（越大越高）
globalDefault: false                    # 是否作为全局默认
preemptionPolicy: PreemptLowerPriority  # PreemptLowerPriority / Never
description: "用于生产环境关键服务"
```

```yaml
# 使用 PriorityClass
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
```

**系统内置 PriorityClass**：

| PriorityClass | 优先级值 | 用途 |
|--------------|---------|------|
| `system-cluster-critical` | 2000000000 | 集群关键组件 |
| `system-node-critical` | 2000001000 | 节点关键组件 |

### 抢占（Preemption）机制

当高优先级 Pod 无法调度时，调度器会尝试**驱逐**低优先级的 Pod：

```
调度流程：
  新 Pod 到达（priority=1000000）
    │
    ▼
  调度器预选 → 无满足条件的节点
    │
    ▼
  调度器启用抢占逻辑：
    │
    ├──► 选择一组候选节点
    │
    ├──► 在每个节点上选择可以被驱逐的低优先级 Pod
    │    （满足以下条件：新 Pod 可以调度到释放资源的节点上）
    │
    ├──► 评估驱逐代价（被驱逐的 Pod 数量、优先级损失）
    │
    ├──► 选择代价最小的节点
    │
    ├──► 优雅驱逐低优先级 Pod（发送 deletion 请求）
    │
    └──► 将新 Pod 调度到释放资源的节点
```

**被抢占 Pod 的终止**：
- 被抢占的 Pod 会**优雅终止**——经历完整的 PreStop → SIGTERM → GracePeriod → SIGKILL 流程
- 被抢占 Pod 的 `status` 会记录 `reason: Preempted`
- 被抢占 Pod 的控制器（如 Deployment）会检测到 Pod 消失，并尝试重新创建（可能会被再次抢占）

**最佳实践**：
- 只为真正重要的服务设置高优先级
- 避免所有 Pod 都设置相同优先级（等于没有优先级）
- `system-cluster-critical` 和 `system-node-critical` 级别的 PriorityClass 应保留给系统组件
- 业务 Pod 的 PriorityClass 通常不超过 1000-10000

# 8.Pod资源与QoS

## 8.1 Resource Requests vs Limits

资源请求（requests）和资源限制（limits）是 Kubernetes 资源管理的两个基本概念：

```yaml
resources:
  requests:        # 向调度器"预订"的最小资源量
    cpu: "500m"    # 0.5 个 vCPU
    memory: "256Mi"
  limits:          # 运行时允许使用的最大资源量
    cpu: "1000m"   # 1 个 vCPU
    memory: "512Mi"
```

**两者的关系**：

```
          limits (1000m)
          ┌─────────────────────┐
          │  可突发使用的区间      │
          │  (Burst Region)     │  这部分资源在节点空闲时可使用
requests  │─────────────────────│  但在资源紧张时可能被回收
(500m)    │  保证使用的区间       │
          │  (Guaranteed)       │  这部分资源被调度器"预订"
          └─────────────────────┘
          0
```

**关键区别**：

| 维度 | Requests | Limits |
|------|----------|--------|
| **作用阶段** | 调度时 | 运行时 |
| **调度影响** | 决定 Pod 能否被调度 | 不影响调度（超配基础）|
| **CPU 超限行为** | - | 被限流（throttle），不会杀死 |
| **内存超限行为** | - | 被 OOMKiller 杀死 |
| **QoS Class 影响** | ✓ | ✓ |

**为什么需要两者**：
- `requests` 保证应用的**最低资源**：在最坏情况下（节点资源紧张），应用至少有这些资源可用
- `limits` 防止应用成为**资源黑洞**：一个内存泄漏的容器不能无限制地消耗节点内存，影响其他容器
- 超配（limits > requests/总和 > 节点容量）允许提高资源利用率，代价是应用可能因资源争抢而性能波动

## 8.2 QoS Class

Kubernetes 根据 Pod 内容器的 `requests` 和 `limits` 配置，自动将 Pod 分为三个 QoS（Quality of Service）等级：

### Guaranteed（保证型）

**条件**：每个容器都同时设置了 CPU 和内存的 requests 和 limits，且 requests == limits。

```yaml
containers:
- name: app
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "500m"       # 等于 requests
      memory: "256Mi"   # 等于 requests
- name: sidecar
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "250m"
      memory: "128Mi"
```

### Burstable（突发型）

**条件**：至少有一个容器设置了 requests（或 limits），但不满足 Guaranteed 的条件。

```yaml
# 情况1：requests != limits
containers:
- name: app
  resources:
    requests:
      memory: "128Mi"
    limits:
      memory: "256Mi"   # limits != requests

# 情况2：只设置了部分资源类型
containers:
- name: app
  resources:
    requests:
      cpu: "500m"       # 没有设置 memory requests，也没有 memory limits
```

### BestEffort（尽力型）

**条件**：没有任何容器设置 requests 或 limits。

```yaml
containers:
- name: app
  image: nginx
  # 没有 resources 字段
```

## 8.3 QoS与驱逐优先级的关系

当节点资源紧张时，kubelet 按 QoS 等级决定驱逐顺序：

```
驱逐优先级（先被驱逐）：
  BestEffort  ──►  Burstable（超出requests部分）──►  Burstable（requests内）──►  Guaranteed
  (最低)                                                                    (最高/最后被驱逐)
```

**kubelet 的驱逐决策过程**：

1. **监控节点资源**：kubelet 持续监控节点内存、磁盘、PID 等资源

2. **资源压力检测**：
   - `memory.available < eviction-hard`（如 `< 100Mi`）时触发内存驱逐
   - `nodefs.available < eviction-hard` 时触发磁盘驱逐

3. **驱逐顺序**：
   - 首先驱逐 BestEffort Pod（因为没有资源保证）
   - 其次驱逐 Burstable Pod 中**超出 requests**部分使用量的 Pod（即内存使用 > requests，且超出的部分最大的先被驱逐）
   - 最后驱逐 Guaranteed 和 Burstable Pod 的 requests 内部分（按优先级）

4. **同 QoS 内的排序**：
   - 内存：按使用量超过 requests 的比例排序，比例越高的先被驱逐
   - CPU：Uncompressible，不会因为 CPU 压力驱逐（只会 throttle）

```bash
# 查看 Pod 的 QoS Class
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
```

## 8.4 OOM Score调整机制

OOM Score 是 Linux 内核在内存不足时决定杀死哪个进程的评分标准。Kubernetes 通过设置容器的 `oom_score_adj` 值来影响内核的 OOM Killer 决策。

### OOM Score 计算公式

```
oom_score = 进程内存使用百分比 × 1000 + oom_score_adj
```

`oom_score` 越高，越容易被 OOM Killer 选中杀死。

### Kubernetes 的 oom_score_adj 设置策略

Kubelet 根据 QoS 和 Pod 资源使用量为容器设置 `oom_score_adj`：

| QoS Class | 条件 | oom_score_adj |
|-----------|------|--------------|
| Guaranteed | 都在 requests 内 | -997 |
| Burstable | 使用量 ≤ requests | -997（或基于使用量计算）|
| Burstable | 使用量 > requests | min(max(2, 1000 - 1000 × memoryRequest / memoryLimit), 999) |
| BestEffort | 无限制 | 1000 |

**数值含义**：
- `-1000`：完全豁免，不会被 OOM Killer 选中
- `-997`：接近豁免，几乎不会被选中（Kubernetes 保留 -1000 给 kubelet 和 docker/containerd）
- `0`：默认值
- `1000`：总是被选中

**为什么 Guaranteed 是 -997 而不是 -1000**：
kubelet 和容器运行时自身需要被优先保护。如果 Guaranteed Pod 的 OOM score 设为 -1000，那么在极端内存不足时，容器运行时（如 containerd）和 Pod 的"豁免权"相同，内核可能杀死容器运行时导致更大范围故障。设置 -997 确保容器运行时（-1000）始终比任何 Pod 更安全。

**实际效果**：
- Guaranteed Pod 在当前 requests 范围内的内存使用几乎不会被 OOM Kill
- BestEffort Pod 几乎总是第一个被 OOM Kill
- Burstable Pod 处于中间，取决于其使用量与 requests 的比例

**OOM Kill 的日志特征**：

```
# 查看 Pod 是否被 OOMKilled
kubectl describe pod my-pod

State:         Terminated
  Reason:      OOMKilled
  Exit Code:   137
```

# 9.临时容器（Ephemeral Container）

## 9.1 用途与限制

临时容器（Ephemeral Container）是 Kubernetes 1.23 稳定的特性（1.16 引入 Alpha），用于向正在运行的 Pod 中注入一个**临时调试容器**。

**设计初衷**：
在日常运维和故障排查中，经常需要进入一个正在运行的 Pod 进行调试。但传统的容器（stdin + tty 交互）不好修改；而直接登录运行的容器可能受限：
- 容器可能只安装了最小化依赖（distroless 镜像没有 shell）
- 容器的 rootfs 可能是只读的
- 容器的安全策略可能不允许 exec 访问

临时容器解决了这些问题——它允许你在现有 Pod 中**临时添加**一个新容器，用于调试和故障排查。

**关键限制**：
- 临时容器**不能通过 Pod spec 添加**——必须通过 API 子资源 `/ephemeralcontainers` 特殊处理
- 临时容器**不保证**资源（无 requests/limits）
- 临时容器**永远不会自动重启**
- 临时容器**不能有探针**
- 临时容器**不能被删除后重新添加**——一旦退出就被视为终止
- 临时容器**没有端口**概念（没有 ports 字段）
- 临时容器主要用于**交互式故障排查**（如 `kubectl debug`）

## 9.2 使用方式

### kubectl debug 方式

```bash
# 创建一个带调试工具的临时容器
kubectl debug my-pod -it --image=nicolaka/netshoot --target=my-container

# 使用自定义镜像进行调试
kubectl debug my-pod -it --image=busybox -- sh

# 创建一个 Pod 副本用于调试（推荐：不影响原 Pod）
kubectl debug my-pod -it --copy-to=my-pod-debug -- sh
```

### 底层 API 实现

```yaml
# 临时容器通过 Pod 的 status 注入（不是 spec！）
# kubectl debug 会向 API Server 发送 PATCH 请求：
PATCH /api/v1/namespaces/default/pods/my-pod/ephemeralcontainers

# 请求体：
{
  "spec": {
    "ephemeralContainers": [{
      "name": "debugger",
      "image": "nicolaka/netshoot",
      "command": ["/bin/bash"],
      "stdin": true,
      "tty": true,
      "targetContainerName": "my-container"  # 共享目标容器的进程命名空间
    }]
  }
}
```

**`targetContainerName` 的含义**：
如果指定了 `targetContainerName`，临时容器会共享目标容器的进程命名空间（前提是 Pod 启用了 `shareProcessNamespace: true` 或目标容器的进程信息可见）。同时还可以挂载目标容器的文件系统。

### 典型调试场景

```bash
# 场景1：网络调试（使用 netshoot 镜像）
kubectl debug my-pod -it --image=nicolaka/netshoot -- bash
# 进入临时容器后：
#   tcpdump -i eth0
#   nslookup service-name
#   curl http://localhost:8080/healthz
#   netstat -tlnp

# 场景2：进程调试（需要 shareProcessNamespace: true）
kubectl debug my-pod -it --image=busybox --target=my-container -- sh
# 进入后可以使用 ps 查看主容器进程

# 场景3：文件系统调试（带调试工具的副本来排查）
kubectl debug my-pod -it --copy-to=my-pod-debug \
  --container=debug --image=ubuntu -- bash
# 创建了原 Pod 的副本，可以在不影响生产 Pod 的情况下调试
```

# 10.Pod安全

## 10.1 Pod Security Standards

Pod Security Standards（PSS）定义了三个安全策略级别，从宽松到严格：

### Privileged（特权级）

最宽松的策略，几乎无限制。适用于需要访问主机资源的系统级 Pod：

```yaml
# 标签表示：允许 Privileged 级别的 Pod
metadata:
  labels:
    pod-security.kubernetes.io/enforce: privileged
```

**允许的操作**：
- 运行特权容器（`privileged: true`）
- 使用主机命名空间（hostNetwork、hostPID、hostIPC）
- 挂载主机路径（hostPath）
- 以 root 用户运行
- 使用所有 Linux capabilities

### Baseline（基线级）

中等限制，阻止已知的权限提升路径。适合大多数应用：

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: baseline
```

**主要限制**：
- 禁止特权容器
- 禁止 hostPath 卷（CSS 类型除外）
- 禁止 hostNetwork、hostPID、hostIPC
- 限制 Linux capabilities
- 禁止 HostPorts
- 限制 AppArmor、Seccomp、SELinux 的使用
- 限制 /proc 挂载类型

### Restricted（限制级）

最严格的策略，遵循 Pod 加固最佳实践：

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

**主要限制**（在 Baseline 基础上增加）：
- 必须放弃所有 capabilities（`drop: ["ALL"]`）
- 禁止卷类型：hostPath、Host、NFS、GlusterFS 等
- 必须以非 root 用户运行（`runAsNonRoot: true`）
- 必须设置 seccompProfile 为 `RuntimeDefault` 或 `Localhost`
- 必须设置容器级别的 SecurityContext（至少有 `allowPrivilegeEscalation: false`）

### 启用 Pod Security Admission

Pod Security Admission（1.25 GA）是内置的准入控制器，按命名空间级别应用安全策略：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline        # 执行级别
    pod-security.kubernetes.io/enforce-version: v1.27   # 应用于的 K8s 版本
    pod-security.kubernetes.io/audit: restricted        # 审计级别（记录但不阻止）
    pod-security.kubernetes.io/warn: restricted         # 警告级别（给用户警告但不阻止）
```

三种模式：
- `enforce`：违反策略的 Pod 会被拒绝创建
- `audit`：违反策略的 Pod 被允许创建，但会记录审计日志
- `warn`：违反策略的 Pod 被允许创建，但用户会收到警告

## 10.2 SecurityContext各字段详解

### Pod 级别字段

```yaml
spec:
  securityContext:
    runAsUser: 1000                      # 运行 Pod 的 UID
    runAsGroup: 3000                     # 运行 Pod 的 GID
    runAsNonRoot: true                   # 强制非 root 运行
    fsGroup: 2000                        # 卷挂载时的文件系统属组
    fsGroupChangePolicy: OnRootMismatch  # 何时更改卷的属组
    seccompProfile:                      # Seccomp 安全计算模式
      type: RuntimeDefault
    supplementalGroups: [4000, 5000]     # 补充组
    sysctls:                             # 内核参数调整
    - name: net.core.somaxconn
      value: "1024"
    - name: net.ipv4.tcp_tw_reuse
      value: "1"
```

### 容器级别字段

```yaml
containers:
- name: app
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    allowPrivilegeEscalation: false      # 禁止通过 setuid/setgid 提权
    privileged: false
    readOnlyRootFilesystem: true
    capabilities:
      drop:
      - ALL                              # 删除所有能力
      add:
      - NET_BIND_SERVICE                 # 只添加必需的
      - NET_RAW
    procMount: Default                   # /proc 的挂载类型
    seccompProfile:
      type: RuntimeDefault
```

### 字段优先级

容器级别的 SecurityContext 会**覆盖** Pod 级别的同名设置。例如：

```yaml
spec:
  securityContext:
    runAsUser: 2000              # Pod 级别默认 UID 2000
  containers:
  - name: app1
    # 未设置 runAsUser，继承 Pod 级别 UID 2000
  - name: app2
    securityContext:
      runAsUser: 3000            # 容器级别覆盖为 UID 3000
```

## 10.3 PodSecurityPolicy → PodSecurity Admission过渡

### PodSecurityPolicy（已弃用）

PodSecurityPolicy（PSP）曾经是 Kubernetes 中控制 Pod 安全策略的机制（v1.21 弃用，v1.25 移除）：

```yaml
# 已弃用的 PSP（不推荐使用）
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  fsGroup:
    rule: MustRunAs
    ranges:
    - min: 1
      max: 65535
```

**PSP 废弃的原因**：
- 配置复杂：需要创建 PSP + ClusterRole + RoleBinding 才能生效
- 权限模型混乱：PSP 是集群级别资源，通过 RBAC 绑定到命名空间
- 缺少审计/警告模式：只能在 enforce 层面工作
- 与第三方准入控制器（如 OPA/Gatekeeper）功能重叠

### 迁移路径

```
PSP (v1.21 deprecation → v1.25 removal)
        ↓
Pod Security Admission (内置，命名空间级别)
        ↓ 或
OPA/Gatekeeper、Kyverno (第三方，更灵活)
```

**Pod Security Admission 的优势**：
- 内置：随 Kubernetes 一起发行，无需额外安装
- 简单：只需给命名空间添加标签
- 三种模式：enforce / audit / warn，支持渐进式启用

# 11.Static Pod

## 11.1 概念与工作原理

Static Pod 是由 kubelet 直接管理的 Pod，**不经过** API Server 和调度器。kubelet 监视本地文件系统中的 Pod 清单文件，自动创建和管理这些 Pod。

```
普通 Pod：
  API Server ◄── kubectl/user ──► etcd ──► Scheduler ──► kubelet ──► Container Runtime

Static Pod：
  /etc/kubernetes/manifests/*.yaml ──► kubelet 直接读取 ──► Container Runtime
```

**关键特性**：
- 不存储在 etcd 中，不受 API Server 管理
- 不受调度器调度，总是运行在 kubelet 所在节点
- kubelet 会在 API Server 中创建一个**镜像 Pod**（Mirror Pod），这是一个只读的表示，让你可以通过 `kubectl get pods` 看到它
- 镜像 Pod 的 `metadata.annotations` 包含 `kubernetes.io/config.source: file` 等标记
- 删除镜像 Pod 不会删除 Static Pod（kubelet 会立即重新创建镜像 Pod）
- Static Pod 的生命周期完全由 kubelet 管理

## 11.2 配置方式

### 通过 kubelet 参数配置

```bash
# kubelet 启动参数
kubelet --pod-manifest-path=/etc/kubernetes/manifests

# 或者通过 kubelet 配置文件
# /var/lib/kubelet/config.yaml
staticPodPath: /etc/kubernetes/manifests
```

kubelet 会定期（默认 20 秒）扫描 `staticPodPath` 目录：
- 发现新的 YAML 文件 → 创建 Static Pod 和对应的 mirror Pod
- 发现文件被修改 → 更新 Pod（滚动更新 Static Pod）
- 发现文件被删除 → 删除 Static Pod 和 mirror Pod

### 创建 Static Pod

直接在配置目录中放置 Pod 清单文件：

```yaml
# /etc/kubernetes/manifests/etcd.yaml（kubeadm 集群中的 etcd）
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
  labels:
    component: etcd
    tier: control-plane
spec:
  hostNetwork: true
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.9-0
    command:
    - etcd
    - --advertise-client-urls=https://192.168.1.10:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    # ... 其他参数
    volumeMounts:
    - name: etcd-data
      mountPath: /var/lib/etcd
    - name: etcd-certs
      mountPath: /etc/kubernetes/pki/etcd
  volumes:
  - name: etcd-data
    hostPath:
      path: /var/lib/etcd
  - name: etcd-certs
    hostPath:
      path: /etc/kubernetes/pki/etcd
```

## 11.3 与控制平面组件的关系

kubeadm 部署的 Kubernetes 集群中，控制平面组件本身以 Static Pod 方式运行：

```
/etc/kubernetes/manifests/
├── etcd.yaml                     # etcd
├── kube-apiserver.yaml           # kube-apiserver
├── kube-controller-manager.yaml  # kube-controller-manager
└── kube-scheduler.yaml           # kube-scheduler
```

```bash
# 在控制平面节点上，kubelet 扫描 /etc/kubernetes/manifests/ 目录
# 自动创建这些 Static Pod
ls /etc/kubernetes/manifests/

# 你可以通过 kubectl 查看它们（作为镜像 Pod）
kubectl get pods -n kube-system | grep -E 'etcd|apiserver|controller|scheduler'

# 修改控制平面组件：直接编辑清单文件，kubelet 会自动更新
# 例如修改 kube-apiserver 的参数
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# kubelet 检测到文件变化 → 滚动更新该 Static Pod
```

**为什么控制平面组件使用 Static Pod**：
1. **自举问题**：在集群启动时，API Server 还不存在，无法通过 API Server 创建 Pod。Static Pod 绕过了这个循环依赖
2. **自愈能力**：kubelet 会监视清单文件并确保对应的 Static Pod 持续运行。如果控制平面组件崩溃，kubelet 会自动重启它，无需依赖其他组件
3. **简单可靠**：控制平面组件直接由节点上的 kubelet 管理，不需要调度器、控制器管理器等额外组件参与

**缺陷**：
- Static Pod 无法使用高级调度特性（亲和性、反亲和性等）
- 不能通过 `kubectl delete pod` 删除（镜像 Pod 被删除后 kubelet 会立即重建）
- 更新需要直接访问节点文件系统

# 12.Pod常见问题与排查

## 12.1 各状态原因分析

### Pending（长时间停留）

Pod 一直处于 Pending 状态，说明一个或多个容器尚未创建：

```
常见原因：
├── 资源不足
│   ├── 节点 CPU/内存 requests 总和超过节点分配资源
│   ├── 节点暂时没有足够的资源满足 Pod 的 requests
│   └── 排查：kubectl describe nodes | grep -A5 "Allocated resources"
│
├── 调度约束无法满足
│   ├── nodeSelector/nodeAffinity 没有匹配的节点
│   ├── Pod 亲和性/反亲和性无法满足
│   ├── Taints 没有对应的 Tolerations
│   └── 排查：kubectl describe pod my-pod | grep Events
│
├── 存储问题
│   ├── PVC 未绑定（PersistentVolume 不存在或不匹配）
│   ├── StorageClass 的 provisioner 不工作
│   └── 排查：kubectl get pvc && kubectl describe pvc my-pvc
│
├── 镜像拉取问题
│   ├── 镜像不存在或名称错误
│   ├── 私有仓库认证失败
│   ├── 网络无法访问镜像仓库
│   └── 排查：kubectl describe pod my-pod | grep -A10 Events
│
└── Init 容器阻塞
    ├── Init 容器逻辑等待依赖服务（如 DNS 检查一直失败）
    └── 排查：kubectl logs my-pod -c <init-container-name>
```

### CrashLoopBackOff

容器启动后立即崩溃，不断重启：

```
常见原因：
├── 应用启动失败
│   ├── 配置文件错误或缺失
│   ├── 环境变量不正确
│   ├── 依赖服务不可达（数据库、缓存等）
│   └── 排查：kubectl logs my-pod --previous
│
├── OOM（内存不足）
│   ├── 内存 limits 设置过低
│   ├── 应用存在内存泄漏
│   └── 排查：kubectl describe pod | grep OOMKilled
│
├── Liveness Probe 配置不当
│   ├── initialDelaySeconds 太短，应用还没启动完就被判定失败
│   ├── 探测逻辑检查了外部依赖
│   └── 排查：检查探针配置和探测端点实现
│
├── 安全限制
│   ├── runAsNonRoot 但镜像默认以 root 运行
│   ├── seccomp/AppArmor/SELinux 阻止了某些系统调用
│   └── 排查：kubectl describe pod 查看详细退出信息
│
└── 镜像问题
    ├── 镜像中缺少依赖库（"no such file" 错误）
    ├── 启动命令错误
    └── 排查：检查镜像 ENTRYPOINT 和 CMD
```

### ImagePullBackOff

镜像拉取失败后的退避状态：

```
常见原因：
├── 镜像名称错误
│   └── 拼写错误、标签不存在、镜像仓库地址错误
│
├── 认证问题
│   ├── imagePullSecrets 不存在或配置错误
│   ├── Docker config JSON 格式错误
│   └── 排查：kubectl get secrets && kubectl describe secret my-registry-secret
│
├── 网络问题
│   ├── 节点无法访问镜像仓库
│   ├── DNS 解析失败
│   └── 排查：在节点上手动 docker pull 测试
│
└── 镜像仓库限制
    ├── Docker Hub 拉取速率限制（匿名用户 100 pulls/6h）
    └── 排查：在镜像仓库侧检查日志
```

### Evicted（被驱逐）

Pod 被驱逐，通常因节点资源压力：

```
常见原因：
├── 节点内存压力
│   └── 节点可用内存低于 eviction-hard 阈值
│
├── 节点磁盘压力
│   ├── 容器镜像占用空间（imagefs 满）
│   ├── 节点磁盘空间不足（nodefs 满）
│   └── 排查：kubectl describe node | grep -A5 "Conditions"
│
├── 节点 PID 压力
│   └── 节点上运行的进程数超过限制
│
└── 节点不可达
    └── 控制平面长时间无法联系节点
```

```bash
# 查看被驱逐的 Pod（虽然被删除了，但会有记录）
kubectl get pods --all-namespaces --field-selector=status.phase=Failed

# 查看节点资源状况
kubectl top nodes
kubectl describe node <node-name>
```

### Error / Unknown / Terminating 卡住

**Error**：
- 容器启动过程中出现错误（如挂载卷失败、创建容器失败）
- 通常是暂时性错误，kubelet 会重试

**Unknown**：
- kubelet 与 API Server 通信中断
- 可能原因：节点宕机、网络分区、kubelet 崩溃
- API Server 不会主动设置 Unknown，是 Node Controller 在节点 NotReady 一段时间后的行为

**Terminating 卡住**：
Pod 的 `deletionTimestamp` 已设置，但 Pod 一直不消失：

```
常见原因：
├── Finalizer 未被移除
│   ├── 自定义 Finalizer 的后端逻辑卡住或出错
│   ├── 存储相关 Finalizer（如 PVC 保护）阻止删除
│   └── 排查：kubectl get pod my-pod -o json | jq '.metadata.finalizers'
│
├── PreStop 钩子挂起
│   ├── PreStop 命令一直不返回
│   └── 排查：检查 PreStop 钩子定义
│
├── 应用不响应 SIGTERM
│   ├── 应用未处理 SIGTERM 信号
│   ├── PID 1 进程不是应用本身（被 shell/sh 代理）
│   └── 排查：检查容器的 ENTRYPOINT 和 CMD
│
├── kubelet 与 API Server 通信问题
│   └── kubelet 无法上报 Pod 终止状态
│
└── 临时解决方法
    ├── kubectl delete pod my-pod --force --grace-period=0
    ├── 手动移除阻塞的 Finalizer（kubectl patch）
    └── 如果节点已宕机：kubectl delete pod my-pod --force --grace-period=0
```

## 12.2 排查命令与技巧

### 基础排查命令

```bash
# 1. 查看 Pod 状态概览
kubectl get pods -n <namespace> -o wide
kubectl get pods -n <namespace> --show-labels
kubectl get pods -n <namespace> --sort-by=.status.startTime

# 2. 查看 Pod 详细信息
kubectl describe pod <pod-name> -n <namespace>

# 3. 查看 Pod 日志
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -c <container-name>      # 多容器 Pod 指定容器
kubectl logs <pod-name> -n <namespace> --previous               # 查看前一个崩溃容器的日志
kubectl logs <pod-name> -n <namespace> --tail=100               # 查看最后 100 行
kubectl logs <pod-name> -n <namespace> --since=5m               # 查看最近 5 分钟日志
kubectl logs <pod-name> -n <namespace> -f                       # 实时跟踪日志

# 4. 进入容器交互式调试
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
kubectl exec -it <pod-name> -n <namespace> -c <container-name> -- /bin/sh

# 5. 查看 Pod 事件
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20

# 6. 获取 Pod 的 YAML/JSON（包括 status）
kubectl get pod <pod-name> -n <namespace> -o yaml
kubectl get pod <pod-name> -n <namespace> -o json | jq '.status'
```

### 高级诊断技巧

```bash
# 1. 查看 Pod 中所有容器的状态
kubectl get pod <pod-name> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state}{"\n"}{end}'

# 2. 查看 Pod 的 QoS Class
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# 3. 查看 Pod 的最近事件（按时间排序）
kubectl get events --sort-by=.lastTimestamp | grep <pod-name>

# 4. 调试 CrashLoopBackOff - 获取容器的终止原因
kubectl get pod <pod-name> -o json | jq '.status.containerStatuses[].lastState.terminated'

# 5. 检查 Pod 资源使用（需要 metrics-server）
kubectl top pod <pod-name> -n <namespace>
kubectl top pod <pod-name> -n <namespace> --containers  # 容器级别

# 6. 检查 Init 容器日志
kubectl logs <pod-name> -c <init-container-name>

# 7. 复制文件到/从 Pod
kubectl cp <pod-name>:/path/to/file ./local-file -n <namespace>
kubectl cp ./local-file <pod-name>:/path/to/file -n <namespace>

# 8. 临时运行一个调试 Pod
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
# 在调试 Pod 中测试网络连通性、DNS 解析等

# 9. 使用 ephemeral container 进行在线调试
kubectl debug <pod-name> -it --image=busybox --target=<container-name> -- sh
```

### 常见排查工作流

**排查 Workflow 1：Pod 无法启动**

```bash
# Step 1: 检查 Pod 事件
kubectl describe pod <pod-name> | grep -A20 Events

# Step 2: 检查当前容器状态和前一容器状态
kubectl get pod <pod-name> -o json | jq '.status.containerStatuses[] | {name, state, lastState, restartCount}'

# Step 3: 如果容器被创建但崩溃，检查前一容器的日志
kubectl logs <pod-name> --previous

# Step 4: 如果有 Init 容器，检查 Init 容器日志
kubectl logs <pod-name> -c <init-container-name>

# Step 5: 检查资源是否不足
kubectl describe pod <pod-name> | grep -A5 "FailedScheduling"
kubectl top nodes

# Step 6: 检查镜像是否存在且可访问
kubectl describe pod <pod-name> | grep -A5 "Failed.*image"
# 在节点上手动验证
# crictl pull <image-name>  # 在节点上测试拉取
```

**排查 Workflow 2：Pod 运行但不接收流量**

```bash
# Step 1: 检查 Readiness Probe
kubectl describe pod <pod-name> | grep -A10 Readiness

# Step 2: 检查 Pod 的 Ready Condition
kubectl get pod <pod-name> -o json | jq '.status.conditions[] | select(.type=="Ready")'

# Step 3: 检查 Service 的 Endpoints
kubectl get endpoints <svc-name> -n <namespace>
# 确认 Pod IP 在 endpoints 列表中

# Step 4: 从集群内其他 Pod 测试连通性
kubectl run test --rm -it --image=busybox -- wget -O- http://<pod-ip>:<port>
```

**排查 Workflow 3：Pod 频繁重启（CrashLoopBackOff）**

```bash
# Step 1: 检查容器退出码和原因
kubectl get pod <pod-name> -o json | jq '.status.containerStatuses[].lastState.terminated'

# 退出码含义：
# 0 - 正常退出
# 1 - 应用错误
# 137 - 被 SIGKILL 杀死（通常是 OOM）
# 139 - SIGSEGV（段错误）
# 143 - SIGTERM（被优雅终止信号）

# Step 2: 如果是 OOM（exit code 137），检查内存使用
kubectl top pod <pod-name> --containers
kubectl describe pod <pod-name> | grep -A5 "State:"

# Step 3: 检查是否是 Liveness Probe 导致的重启
# 如果是 Liveness Probe，exit code 通常是 137 或 143
kubectl describe pod <pod-name> | grep -A15 Liveness

# Step 4: 查看崩溃前的日志（包括前一个容器实例）
kubectl logs <pod-name> --previous --tail=200
```

**排查 Workflow 4：节点级别的 Pod 问题**

```bash
# 检查节点上的所有 Pod
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node-name>

# 检查节点状态
kubectl describe node <node-name>
kubectl get node <node-name> -o json | jq '.status.conditions'

# 检查节点上 kubelet 的日志（需要在节点上执行）
# journalctl -u kubelet -n 100 --no-pager

# 查看节点资源使用
kubectl top node <node-name>
kubectl describe node <node-name> | grep -A10 "Allocated resources"
```

### 实用工具和命令组合

```bash
# 找出所有命名空间中状态异常的 Pod
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded

# 找出所有 CrashLoopBackOff 的 Pod
kubectl get pods --all-namespaces | grep CrashLoopBackOff

# 找出所有 ImagePullBackOff 的 Pod
kubectl get pods --all-namespaces | grep ImagePullBackOff

# 查看 Pod 及其所有者的关系
kubectl get pod <pod-name> -o json | jq '.metadata.ownerReferences'

# 统计各命名空间的 Pod 数量
kubectl get pods --all-namespaces --no-headers | awk '{print $1}' | sort | uniq -c | sort -nr

# 快速检查 Pod 的安全上下文配置
kubectl get pod <pod-name> -o json | jq '.spec.containers[].securityContext'

# 获取 Pod 的节点调度信息
kubectl get pod <pod-name> -o wide
```

以上涵盖了 Pod 从概念、配置、生命周期、调度、安全到故障排查的完整知识体系。掌握这些内容，就能深入理解 Kubernetes 如何通过 Pod 这一核心抽象来编排和管理容器化工作负载。
