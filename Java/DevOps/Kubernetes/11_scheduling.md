# 1. 调度器架构深度解析

## 1.1 调度器在 Kubernetes 中的定位

kube-scheduler 是 Kubernetes 控制平面的核心组件之一，负责将新创建的、尚未分配节点（`spec.nodeName` 为空）的 Pod 调度到集群中的一个合适节点上。它的工作看似简单——"为 Pod 找个节点"——但其背后涉及复杂的资源评估、约束匹配和优先级计算。

调度器不负责运行 Pod（这是 kubelet 的职责），它只做一件事：**选择一个最优节点，并通过 API Server 将 Pod 的 `spec.nodeName` 设置为该节点名称**。

### 1.1.1 为什么调度如此关键

在大规模集群中，调度决策直接影响：
- **资源利用率**：合理的调度可以显著提高集群资源利用率，降低成本
- **应用性能**：Pod 分布影响网络延迟、数据本地性
- **高可用性**：跨故障域（可用区、机架、节点）分散 Pod，避免单点故障
- **特殊硬件利用**：GPU、FPGA、PMEM 等昂贵设备需要精确调度

## 1.2 调度流程：两个阶段

```
待调度 Pod (spec.nodeName 为空)
        │
        ▼
┌───────────────────────────────┐
│  第一阶段: Filtering (过滤/预选)  │
│                                 │
│  遍历所有节点，排除不满足条件的:   │
│  - 资源不足的节点                │
│  - 有不可容忍污点的节点            │
│  - 不满足亲和性的节点             │
│  - 端口冲突的节点                │
│  - ...                         │
│                                 │
│  输出: 可行节点列表 (Feasible Nodes) │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│  第二阶段: Scoring (打分/优选)    │
│                                 │
│  对可行节点按策略打分:            │
│  - 资源均衡度                   │
│  - 镜像已缓存                   │
│  - Pod 分散度                  │
│  - 亲和性偏好                   │
│  - ...                         │
│                                 │
│  输出: 得分最高的节点             │
└───────────────┬───────────────┘
                │
                ▼
            绑定 Pod 到节点
     (写入 spec.nodeName)
```

## 1.3 调度框架 (Scheduling Framework)

Kubernetes 1.19 引入调度框架（Scheduling Framework）重构了调度器架构，将调度过程拆分为多个扩展点（Extension Points），允许通过插件自定义调度逻辑。

```
调度周期 (Scheduling Cycle)
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│  Sort → PreFilter → Filter → PostFilter → PreScore → Score  │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  Reserve → Permit  ───────────────────→ PreBind → Bind │   │
│  │     (等待)    (批准/拒绝/等待)                PostBind   │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────┘

每个扩展点可以注册多个插件，插件按权重和顺序执行。
```

### 1.3.1 扩展点说明

| 扩展点 | 作用 | 典型插件 |
|--------|------|---------|
| **Sort** | 对调度队列中的 Pod 排序 | PrioritySort |
| **PreFilter** | 预处理 Pod 信息，计算调度周期共享数据 | NodeResourcesFit, InterPodAffinity |
| **Filter** | 过滤不满足条件的节点 | NodeUnschedulable, TaintToleration, NodeAffinity |
| **PostFilter** | 过滤后无可选节点时的补救（抢占） | DefaultPreemption |
| **PreScore** | 打分前的预处理 | InterPodAffinity |
| **Score** | 对节点打分 | NodeResourcesBalancedAllocation, ImageLocality |
| **Reserve** | 预留资源（乐观并发） | VolumeBinding |
| **Permit** | 批准/拒绝/等待调度 | — (用于批量调度等场景) |
| **PreBind** | 绑定前的准备工作 | VolumeBinding |
| **Bind** | 执行绑定 | DefaultBinder |
| **PostBind** | 绑定成功后的通知 | — (记录事件等) |

## 1.4 调度器的配置方式

调度器可以通过 KubeSchedulerConfiguration 配置文件或启动参数进行配置：

```yaml
# scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: true
clientConnection:
  kubeconfig: /etc/kubernetes/scheduler.conf
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      disabled:
      - name: NodeResourcesBalancedAllocation
      enabled:
      - name: MyCustomScore
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1

# 启动参数指定配置文件
# kube-scheduler --config=/etc/kubernetes/scheduler-config.yaml
```

# 2. 节点亲和性 (Node Affinity)

## 2.1 为什么需要节点亲和性

最简单的 Pod 调度控制方式是指定 `nodeSelector`：

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

`nodeSelector` 的问题在于它只支持"必须满足"的等式匹配（AND 逻辑）。在实际生产环境中，我们经常需要更灵活的调度策略：
- "优先调度到带 SSD 的节点，但如果 SSD 节点都不可用，也可以调度到 HDD 节点"
- "不能调度到 GPU 节点（这些节点留给 AI 训练任务）"
- "必须调度到可用区 zone-a 或 zone-b 中的节点"

Node Affinity 正是为了解决这些需求而设计，它支持 `required`（硬约束）和 `preferred`（软偏好）两种模式。

## 2.2 Node Affinity 的 YAML 配置

### 2.2.1 requiredDuringSchedulingIgnoredDuringExecution（硬约束）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-demo
spec:
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
          - key: node-type
            operator: NotIn
            values:
            - gpu
```

字段名解析：`requiredDuringSchedulingIgnoredDuringExecution`
- `requiredDuringScheduling`：调度时必须满足（否则 Pod 一直 Pending）
- `IgnoredDuringExecution`：运行期间如果节点标签变化，已运行的 Pod 不受影响（不会被驱逐）

### 2.2.2 preferredDuringSchedulingIgnoredDuringExecution（软偏好）

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80                     # 权重 1-100，越高越优先
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 20
        preference:
          matchExpressions:
          - key: node-role.kubernetes.io/compute
            operator: Exists
```

### 2.2.3 操作符说明

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `In` | 标签值在指定列表中 | `zone In [zone-a, zone-b]` |
| `NotIn` | 标签值不在指定列表中 | `gpu-type NotIn [p100]` |
| `Exists` | 标签键存在（值任意） | `ssd` (只要节点有此标签) |
| `DoesNotExist` | 标签键不存在 | `legacy-system` (节点不能有此标签) |
| `Gt` | 大于（数字比较） | `node-index Gt [5]` |
| `Lt` | 小于（数字比较） | `node-index Lt [10]` |

### 2.2.4 `nodeSelectorTerms` 的逻辑关系

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:                # 多个 term 之间是 OR 关系
  - matchExpressions:               # term 内的多个 expression 是 AND 关系
    - key: zone
      operator: In
      values: [zone-a]
    - key: disktype
      operator: In
      values: [ssd]
  - matchExpressions:
    - key: zone
      operator: In
      values: [zone-b]
    - key: disktype
      operator: In
      values: [nvme]
  # 逻辑: (zone=zone-a AND disktype=ssd) OR (zone=zone-b AND disktype=nvme)
```

### 2.2.5 内置节点标签

Kubernetes 自动为每个节点设置以下标签，可用于调度策略：

| 标签 | 含义 | 示例值 |
|------|------|--------|
| `kubernetes.io/hostname` | 节点主机名 | `worker-1` |
| `topology.kubernetes.io/zone` | 可用区 | `us-east-1a` |
| `topology.kubernetes.io/region` | 地域 | `us-east-1` |
| `node.kubernetes.io/instance-type` | 实例类型 | `m5.large` |
| `kubernetes.io/arch` | CPU 架构 | `amd64`, `arm64` |
| `kubernetes.io/os` | 操作系统 | `linux` |
| `beta.kubernetes.io/arch` | (废弃) | — |
| `topology.kubernetes.io/region` | (替代 failure-domain.beta) | — |

# 3. Pod 亲和性与反亲和性 (Pod Affinity/Anti-Affinity)

## 3.1 调度的关系维度

Node Affinity 解决的是"Pod → Node"的关系，而 Pod Affinity/Anti-Affinity 解决的是"Pod → Pod（及 Pod 所在的 Node）"的关系。

### 3.1.1 典型场景

**Pod Affinity（聚合）**：
- "将我的 Web Pod 和 Redis Pod 调度到同一个节点或同一个可用区"
- 原因：减少网络延迟，利用 localhost 通信
- 适用：紧密耦合的服务

**Pod Anti-Affinity（分散）**：
- "将同一个 Deployment 的副本分散到不同节点"
- 原因：避免单节点故障导致服务完全不可用
- 适用：所有需要高可用的服务

## 3.2 Pod Affinity 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: kubernetes.io/hostname   # 同节点
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web
              topologyKey: kubernetes.io/hostname # 同节点分散
```

### 3.2.1 `topologyKey` 是关键概念

`topologyKey` 定义了"拓扑域"的边界——在这个边界内，亲和性规则生效。

| topologyKey | 含义 | 效果 |
|-------------|------|------|
| `kubernetes.io/hostname` | 节点级别 | 同节点 |
| `topology.kubernetes.io/zone` | 可用区级别 | 同可用区 |
| `topology.kubernetes.io/region` | 地域级别 | 同地域 |
| 自定义标签 | 机架/楼层/任意 | 自定义拓扑域 |

### 3.2.2 requiredDuringScheduling vs preferredDuringScheduling

```yaml
# 硬约束：必须和 cache Pod 在同一节点
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: cache
    topologyKey: kubernetes.io/hostname

# 软偏好：尽量不和自己的其他副本在同一节点
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: web
      topologyKey: kubernetes.io/hostname
```

## 3.3 生产实战：高可用分散部署

```yaml
# 标准的高可用部署配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: api-server
            topologyKey: topology.kubernetes.io/zone    # 必须分散到不同可用区
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: api-server
              topologyKey: kubernetes.io/hostname        # 尽量分散到不同节点
      containers:
      - name: api
        image: api-server:v1.0
```

这个配置确保：
1. 3 个副本必须分布在不同的可用区（如果集群有 3 个可用区）
2. 在可用区内部，尽量分布到不同节点

如果只有 2 个可用区但有 3 个副本，第三个副本将处于 Pending 状态（因为没有满足 hard anti-affinity 的第三个可用区）。

## 3.4 与 Node Affinity 的组合

```yaml
spec:
  affinity:
    nodeAffinity:              # Pod → Node
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values: [compute]
    podAntiAffinity:           # Pod → Pod
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname
```

三种亲和性可以同时使用，调度器取所有约束的交集。

# 4. 污点与容忍 (Taints and Tolerations)

## 4.1 污点和容忍的设计哲学

Node Affinity 是"Pod 选择 Node"——Pod 制定规则，选择满足条件的节点。Taints & Tolerations 是"Node 排斥 Pod"——节点设置排斥规则，只有明确声明可以容忍这些排斥条件的 Pod 才能调度上来。

```
Node Affinity:  Pod → "我只去有 SSD 标签的节点"（吸引）
Taints:        Node → "没有 GPU 容忍的 Pod 禁止来我这里"（排斥）
```

两者配合使用才能实现精细的调度控制：
- Node Affinity 吸引 Pod 到合适节点
- Taints 阻止不合适的 Pod 调度到节点
- Tolerations 允许特定 Pod 上受污染节点

## 4.2 Taint 的构成

Taint 由三个要素组成：`key=value:effect`

```bash
# 查看节点污点
kubectl describe node worker-1 | grep Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
# 格式: key=value:effect

# 添加污点
kubectl taint node worker-1 dedicated=gpu:NoSchedule

# 删除污点（在 key 后加 -）
kubectl taint node worker-1 dedicated=gpu:NoSchedule-
```

### 4.2.1 三种 Effect（效果）

| Effect | 行为 | 使用场景 |
|--------|------|---------|
| **NoSchedule** | 新 Pod 不会被调度到该节点，已有 Pod 不受影响 | 节点预留、专用节点 |
| **PreferNoSchedule** | 尽量不调度新 Pod 到该节点（软约束） | 节点有轻微问题但仍可用 |
| **NoExecute** | 新 Pod 不会被调度，已在运行的 Pod 如不容忍则被驱逐 | 节点故障、节点下线 |

## 4.3 Toleration 配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300    # 只容忍 300 秒，超时后仍被驱逐
  containers:
  - name: gpu-app
    image: nvidia/cuda:11.0
```

Toleration 的匹配规则：
- `operator: Equal`：`key`、`value` 和 `effect` 必须完全相同
- `operator: Exists`：只要 `key` 和 `effect` 相同，`value` 任意

### 4.3.1 `tolerationSeconds` 详解

`tolerationSeconds` 定义了 Pod 在被驱逐前可以容忍多长时间：

```yaml
# 不设置 tolerationSeconds：永久容忍
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoExecute"

# 设置 tolerationSeconds：只在指定时间内容忍
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300       # 节点 NotReady 后最多容忍 5 分钟
```

## 4.4 内置污点与节点状态

Kubernetes 的 Node Controller 在检测到节点异常时自动添加以下污点，配合 `tolerationSeconds` 实现优雅的 Pod 驱逐：

### 4.4.1 节点失联处理流程

```
节点正常运行
     │
     │ kubelet 停止上报心跳
     ▼
~40s: Node Controller 标记节点为 Unknown
     添加 taint: node.kubernetes.io/unreachable:NoExecute
     添加 taint: node.kubernetes.io/unreachable:NoSchedule
     │
     │ 现有 Pod 自带容忍 (默认 tolerationSeconds=300)
     ▼
~300s: Pod 容忍超时
     │ Node Controller 开始驱逐该节点上的 Pod
     │ 被驱逐的 Pod 被调度到其他健康节点
     ▼
Pod 在新节点重建运行
```

默认容忍（kubelet 自动为所有 Pod 添加，仅当 Pod 不显式覆盖时）：

```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300

- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

### 4.4.2 节点状态与对应污点

| 节点 Condition | 对应 Taint |
|---------------|-----------|
| `Ready: Unknown` | `node.kubernetes.io/unreachable:NoExecute` |
| `Ready: False` | `node.kubernetes.io/not-ready:NoExecute` |
| `MemoryPressure: True` | `node.kubernetes.io/memory-pressure:NoSchedule` |
| `DiskPressure: True` | `node.kubernetes.io/disk-pressure:NoSchedule` |
| `PIDPressure: True` | `node.kubernetes.io/pid-pressure:NoSchedule` |
| `NetworkUnavailable: True` | `node.kubernetes.io/network-unavailable:NoSchedule` |

## 4.5 生产场景实战

### 4.5.1 场景一：专用 GPU 节点

```bash
# 给 GPU 节点打标签和污点
kubectl label node gpu-node-1 accelerator=nvidia-t4
kubectl label node gpu-node-2 accelerator=nvidia-t4
kubectl taint node gpu-node-1 gpu=true:NoSchedule
kubectl taint node gpu-node-2 gpu=true:NoSchedule
```

```yaml
# GPU Pod 配置
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values: [nvidia-t4]
  containers:
  - name: gpu-app
    resources:
      limits:
        nvidia.com/gpu: 1
```

### 4.5.2 场景二：控制面节点隔离

```bash
# 控制面节点默认已有此污点
# Taints: node-role.kubernetes.io/control-plane:NoSchedule

# 如果希望某些管理工具也能运行在控制面节点
```

```yaml
# 允许某些关键服务调度到控制面节点
spec:
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
```

### 4.5.3 场景三：临时节点下线维护

```bash
# 阻止新 Pod 调度
kubectl cordon node worker-1

# 驱逐现有 Pod（优雅迁移）
kubectl drain node worker-1 --ignore-daemonsets --delete-emptydir-data

# drain 命令等价于：
# 1. cordon 节点（添加 node.kubernetes.io/unschedulable:NoSchedule）
# 2. 驱逐所有 Pod（通过 eviction API）

# 维护完成后恢复
kubectl uncordon node worker-1
```

# 5. 拓扑分布约束 (Topology Spread Constraints)

## 5.1 为什么需要拓扑分布约束

Pod Anti-Affinity 可以实现副本在节点间分散，但它存在局限性：
1. 如果一个节点挂了，之前的 Anti-Affinity 被打破后不会自动重新平衡
2. 难以表达"尽量均匀分布"的语义（只有 hard 和 preferred 两级）
3. 无法定义"最大偏差"——比如"任一可用区的 Pod 数量不能超过总 Pod 数量的 30%"

Topology Spread Constraints（v1.19 stable）解决了这些问题，它提供了一种声明式的方式来表达"Pod 应该在拓扑域之间均匀分布"。

## 5.2 配置示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      topologySpreadConstraints:
      # 约束一：跨可用区均匀分布
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule       # DoNotSchedule / ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web
            version: v1
      # 约束二：跨节点均匀分布
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway      # 软约束
        labelSelector:
          matchLabels:
            app: web
```

### 5.2.1 关键参数详解

**`maxSkew`（最大偏差）**：任何两个拓扑域之间的 Pod 数量差的最大允许值。

```
可用区  Pod 数
zone-a    2
zone-b    2
zone-c    2     maxSkew = 0（完全均匀）

zone-a    3
zone-b    2
zone-c    2     maxSkew = 1（zone-a 比最少的多 1 个）

zone-a    4
zone-b    2
zone-c    2     maxSkew = 2（超出 maxSkew=1，不满足约束）
```

**`whenUnsatisfiable`（不满足时的行为）**：
- `DoNotSchedule`：硬约束，不满足则不调度 Pod
- `ScheduleAnyway`：软约束，尽量满足但允许违反

**`topologyKey`**：定义拓扑域（与 Pod Affinity 中的概念相同）

**`labelSelector`**：计算分布时考虑哪些 Pod（通常匹配与当前 Pod 同一应用的 Pod）

### 5.2.2 与 Pod Anti-Affinity 的对比

| 维度 | Pod Anti-Affinity | Topology Spread Constraints |
|------|-------------------|---------------------------|
| 控制粒度 | 每个 Pod 对 | 全局分布视角 |
| 均匀性 | 尽力而为 | 精确的最大偏差控制 |
| 新增副本时的行为 | 尽量选不同节点 | 选 Pod 最少的拓扑域 |
| 对已存在 Pod 的影响 | 不影响 | 不影响（不重新平衡） |
| 复杂度 | 低 | 中 |

## 5.3 进阶：多约束场景

```yaml
topologySpreadConstraints:
# 约束一：跨可用区分布（硬约束）
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: web
# 约束二：在可用区内跨节点分布（软约束）
- maxSkew: 2
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: web
```

多个约束同时生效时，调度器取所有约束的交集。注意不要在同一个 `topologyKey` 上同时设置 `PodAntiAffinity` 和 `TopologySpreadConstraints`，可能导致冲突。

## 5.4 不均问题的自动修复

通过 `minDomains` 参数（v1.25 alpha, v1.30 beta）可以处理"全局最小值"问题：

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  minDomains: 3               # 至少考虑 3 个可用区
  labelSelector:
    matchLabels:
      app: web
```

`minDomains` 保证至少有指定数量的拓扑域被纳入计算，防止因为某些区域没有 Pod 而导致不均衡。

# 6. 优先级与抢占 (Priority and Preemption)

## 6.1 Priority 机制

Priority（优先级）允许你为 Pod 分配相对重要性，在资源紧张时：
1. 高优先级 Pod 优先被调度
2. 如果集群资源不足，高优先级 Pod 可以驱逐（preempt）低优先级 Pod

### 6.1.1 PriorityClass 定义

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000                          # 优先级数值越大优先级越高
globalDefault: false                    # 是否作为全局默认
preemptionPolicy: PreemptLowerPriority  # PreemptLowerPriority / Never
description: "重要业务使用的优先级别"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: true
preemptionPolicy: PreemptLowerPriority
description: "默认优先级，用于非关键服务"
```

### 6.1.2 Pod 使用 Priority

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: important-job
spec:
  priorityClassName: high-priority
  containers:
  - name: job
    image: busybox
```

## 6.2 抢占 (Preemption) 机制

当高优先级 Pod 无法调度时，调度器会尝试抢占低优先级 Pod：

```
1. 高优先级 Pod 进入调度队列
       │
       ▼
2. 调度器执行 Filter → 没有节点满足条件
       │
       ▼
3. PostFilter 阶段：DefaultPreemption 插件触发
       │
       ├── 模拟在哪些节点上驱逐哪些低优先级 Pod 可以让高优 Pod 调度
       │      - 优先选择驱逐后受影响 Pod 最少的节点
       │      - 优先驱逐优先级最低的 Pod
       │      - 遵守 PDB (PodDisruptionBudget)
       │
       ▼
4. 选出"牺牲品" Pod 列表
       │
       ▼
5. 调用 Eviction API 驱逐低优先级 Pod
       │  低优先级 Pod 正常终止（收到 SIGTERM，遵守 terminationGracePeriodSeconds）
       │
       ▼
6. 等待被抢占节点释放资源
       │
       ▼
7. 高优先级 Pod 被调度到释放了资源的节点
```

### 6.2.1 抢占策略

| preemptionPolicy | 行为 |
|-----------------|------|
| `PreemptLowerPriority` | 可以抢占低优先级 Pod（默认） |
| `Never` | 从不抢占其他 Pod，调度失败就等待 |

### 6.2.2 防止核心组件被抢占

```yaml
# kube-system 中的关键组件使用最高优先级
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-cluster-critical
value: 2000000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority      # 可以抢占任何更低优先级的

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-node-critical
value: 2000001000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
```

## 6.3 生产实践：优先级分级

推荐的优先级分级方案：

| PriorityClass | value | 用途 |
|--------------|-------|------|
| `system-cluster-critical` | 2000000000 | API Server、etcd、CoreDNS 等集群核心组件 |
| `system-node-critical` | 2000001000 | kube-proxy、CNI 等节点关键组件 |
| `production-critical` | 1000000 | 核心业务服务 |
| `production-high` | 100000 | 重要业务服务 |
| `production-medium` | 10000 | 一般业务服务 |
| `production-low` | 1000 | 低优先级服务 |
| `batch` | 100 | 批处理任务 |
| `default` | 0 | 默认 |

# 7. 调度策略综合实战

## 7.1 多策略组合的场景

以下是一个综合了多种调度策略的生产级 Pod 配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 5
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
        tier: critical
    spec:
      priorityClassName: production-critical
      serviceAccountName: payment-sa
      # 节点选择
      nodeSelector:
        kubernetes.io/os: linux
      # 组合三种亲和性
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values: [compute]
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values: [ssd]
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: payment
            topologyKey: topology.kubernetes.io/zone
      # 拓扑分布约束
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: payment
      # 容忍控制面节点（如果必要时调度其上）
      tolerations:
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 300
      containers:
      - name: app
        image: payment:v2.3.0
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

## 7.2 常见调度问题诊断

### 7.2.1 Pod 一直 Pending

```bash
# 查看调度失败原因
kubectl describe pod <pod-name>

# 常见输出及原因:
# "0/3 nodes are available: 1 Insufficient cpu, 2 node(s) had untolerated taint"
#                                      ↑ 资源不足    ↑ 无法容忍节点的污点

# "0/3 nodes are available: 3 pod affinity/anti-affinity rules not met"
#                                      ↑ 亲和性规则无法满足

# "0/3 nodes are available: 3 node(s) didn't match pod's node affinity"
#                                      ↑ 节点标签不匹配
```

### 7.2.2 调试命令集合

```bash
# 查看调度器日志
kubectl logs -n kube-system kube-scheduler-<node> | tail -50

# 查看节点的污点和标签
kubectl describe node <node-name> | grep -E "Taints:|Labels:"

# 查看 Pod 的优先级
kubectl get pod <pod-name> -o jsonpath='{.spec.priorityClassName}'
kubectl get pod <pod-name> -o jsonpath='{.spec.priority}'

# 模拟调度（不实际调度，查看调度结果）
# 需要使用 kube-scheduler 的 debug 模式或调度器模拟器

# 查看集群中的 PriorityClass
kubectl get priorityclass
```

## 7.3 调度器性能考量

在大规模集群中（5000+ 节点），调度性能可能成为瓶颈：

1. **减少 Filter 候选节点**：使用 `percentageOfNodesToScore` 配置限制打分的节点百分比
2. **使用 Node Affinity 预过滤**：明确的 Node Affinity 可以显著减少需评估的节点数
3. **启用 Bound Service Account Token**：减少 Token 相关的 API 调用
4. **合理配置调度器缓存**：确保调度器的 Informer 缓存同步周期合理

```yaml
# scheduler 配置中的性能参数
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
percentageOfNodesToScore: 30   # 只对 30% 的节点打分（大规模集群优化）
```
