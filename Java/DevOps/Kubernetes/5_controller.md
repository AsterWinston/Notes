# 1. 控制器概述

## 1.1 什么是控制器

Kubernetes 控制器（Controller）是 Kubernetes 架构中最核心的组件之一，它是驱动集群从"当前状态"向"期望状态"不断收敛的引擎。要理解控制器，首先必须理解 Kubernetes 的声明式设计哲学。

### 1.1.1 声明式 API 与控制器模式

Kubernetes 采用声明式 API（Declarative API），用户通过 YAML 或 JSON 清单文件向 API Server 提交资源的"期望状态"（Desired State），而不是编写一系列"先做什么、再做什么"的命令式脚本。例如：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx:1.21
```

用户只声明了"我想要 3 个 nginx:1.21 的 Pod 副本"，至于如何创建、调度、监控这些 Pod，完全由控制器负责。这种声明式模型带来的优势包括：

1. **自愈能力**：如果某个 Pod 意外崩溃，控制器检测到当前副本数变为 2（与期望的 3 不一致），会自动创建新的 Pod 来补偿。
2. **简化运维**：运维人员不需要编写监控脚本、重启脚本、扩容脚本，只需修改 YAML 中的 `replicas` 字段并 `kubectl apply`。
3. **版本控制友好**：YAML 文件可以纳入 Git 管理，通过 GitOps 流程实现基础设施即代码（IaC）。
4. **可观察性**：控制器的工作状态通过 `status` 子资源暴露，用户可以随时查看期望状态与实际状态的差异。

### 1.1.2 控制循环（Control Loop）

控制器的核心是一个无限循环，称为控制循环（Control Loop）或调谐循环（Reconciliation Loop）。其伪代码逻辑非常简单：

```
for {
    期望状态 = 从 API Server 获取资源的 spec 字段
    当前状态 = 从 API Server 获取资源对应的实际状态（Pod 运行情况等）

    if 期望状态 == 当前状态 {
        sleep(一段时间)
        continue
    }

    执行调谐操作，驱动当前状态向期望状态收敛
}
```

这个循环的哲学基础是"最终一致性"（Eventual Consistency）。控制器不要求状态立即一致，而是持续不断地检查和修正，确保在有限时间内达到一致。这与关系型数据库的强一致性不同，是分布式系统中常见的妥协——牺牲瞬时一致性换取高可用和分区容错性（CAP 定理中的 AP 模型）。

### 1.1.3 为什么控制器能实现自愈

假设用户创建了一个 `replicas: 3` 的 Deployment：

1. 管理员通过 `kubectl apply` 提交 YAML 到 API Server。
2. API Server 将 Deployment 对象持久化到 etcd。
3. Deployment Controller 监听到新 Deployment，创建对应的 ReplicaSet。
4. ReplicaSet Controller 监听到新 ReplicaSet，发现期望 3 个 Pod，当前 0 个，于是创建 3 个 Pod。
5. Scheduler 监听到未被调度的 Pod，将其绑定到合适的 Node。
6. 目标 Node 上的 kubelet 监听到被调度到本节点的 Pod，拉取镜像并启动容器。

此时，如果 Node2 宕机，其上的 Pod 状态变为 Unknown 或 Terminating：
- Node Controller 检测到 Node2 不健康，将该节点标记为 NotReady。
- ReplicaSet Controller 发现属于该 RS 的 Pod 中有一个已经不在运行状态，当前副本数 = 2 < 期望 3。
- ReplicaSet Controller 在另一个健康节点上创建新的 Pod，恢复副本数为 3。

整个过程无需人工干预，这就是 Kubernetes 自愈能力的本质。

### 1.1.4 控制器与 Operator 的关系

控制器是 Kubernetes 的设计模式，而 Operator 是这种模式的工程化实现。Operator = CRD（Custom Resource Definition）+ 自定义控制器。例如 Prometheus Operator 定义了 `Prometheus` 和 `ServiceMonitor` 等 CRD，并提供了对应的控制器来管理 Prometheus 实例的生命周期。理解原生控制器（Deployment、StatefulSet、Job 等）是开发自定义 Operator 的基础。

---

## 1.2 控制器工作原理

### 1.2.1 Informer 机制

控制器不能每一轮循环都直接查询 API Server（轮询模式），因为这会带来三个问题：

1. **API Server 压力过大**：集群中可能有数十个控制器，每个控制器管理成千上万个资源，轮询会产生海量请求。
2. **延迟高**：轮询间隔不好设置——太短则压力大，太长则响应慢。
3. **重复传输**：每次轮询都拉取全量数据，而大部分数据可能没有变化。

Kubernetes 的解决方案是 **Informer 机制**，它基于 API Server 的 Watch 接口，实现了高效的"事件驱动 + 本地缓存"模型。

```
                     ┌──────────────┐
                     │  API Server  │
                     └──────┬───────┘
                            │ Watch (长连接，推送事件)
                            ▼
                ┌───────────────────────┐
                │      Reflector        │
                │  (负责 List & Watch)  │
                └───────────┬───────────┘
                            │ 将事件写入
                            ▼
                ┌───────────────────────┐
                │     Delta FIFO        │
                │ (存储待处理的变更)     │
                └───────────┬───────────┘
                            │ 弹出事件
                            ▼
                ┌───────────────────────┐
                │      Informer         │
                │  ┌─────────────────┐  │
                │  │   Indexer       │  │
                │  │ (本地缓存+索引) │  │
                │  └─────────────────┘  │
                │  ┌─────────────────┐  │
                │  │ Event Handlers  │  │
                │  │ (Add/Update/Del)│  │
                │  └────────┬────────┘  │
                └───────────┼───────────┘
                            │ 将资源Key推入
                            ▼
                ┌───────────────────────┐
                │     WorkQueue         │
                │ (限速队列，去重)      │
                └───────────┬───────────┘
                            │ Worker从队列取出
                            ▼
                ┌───────────────────────┐
                │   Reconcile Loop      │
                │ (核心业务逻辑)        │
                └───────────────────────┘
```

**Reflector（反射器）**

Reflector 是 Informer 的最前端组件。它首先执行一次 `List` 操作获取资源的全量快照，然后打开一个 `Watch` 长连接接收增量事件（ADDED / MODIFIED / DELETED）。Watch 连接可能因网络问题断开，Reflector 内置了重连机制，重连后会重新 List 全量数据以确保没有漏掉事件。

技术细节：API Server 的 Watch 机制基于 etcd 的 Watch 功能，etcd 会为每个 key 维护一个全局递增的 `ResourceVersion`。客户端 Watch 时携带上次看到的 `ResourceVersion`，API Server 只会推送该版本之后发生的事件。如果事件间隔太长导致 etcd 压缩了历史版本，API Server 会返回 `410 Gone` 错误，此时 Reflector 必须重新 List。

**Delta FIFO**

Reflector 接收到的事件被封装成 Delta（变更记录），存入 Delta FIFO 队列。FIFO 保证了事件的顺序处理（先入先出），同时 Delta 机制可以合并同一对象的多个变更（例如对同一 Pod 的多次更新可以合并为一次处理，减少无意义的调谐）。

**Indexer（本地缓存）**

Indexer 是控制器本地的线程安全存储，本质是一个带索引的内存映射表（map-based cache）。它存储了资源的全量最新状态，控制器读取资源时直接查本地缓存而非调用 API Server，极大减少了网络开销和延迟。

Indexer 支持多种索引函数，最常用的是 `MetaNamespaceKeyFunc`（格式为 `<namespace>/<name>`），也可以自定义索引（例如按 Pod 的 `status.phase` 索引）。

**Event Handlers（事件处理器）**

Informer 注册的事件处理器（ResourceEventHandler）包含三个回调：

```go
type ResourceEventHandler interface {
    OnAdd(obj interface{})
    OnUpdate(oldObj, newObj interface{})
    OnDelete(obj interface{})
}
```

这些回调不会直接执行调谐逻辑（那样会阻塞事件分发），而是将受影响的资源 Key 推入 WorkQueue，由 Worker 异步处理。

**WorkQueue（工作队列）**

WorkQueue 不是普通的 FIFO 队列，它具备以下关键特性：

1. **去重**：如果同一个资源 Key 已经在队列中，后续的推入会被忽略（合并）。例如，在 1 秒内 Pod 被连续修改 10 次，队列中只会保留一项。
2. **限速（Rate Limiting）**：对于失败的重试，采用指数退避（Exponential Backoff）策略，避免持续对 API Server 发起无效请求。限速器支持三种模式：
   - `BucketRateLimiter`：令牌桶算法，控制每秒请求数（QPS）。
   - `ItemExponentialFailureRateLimiter`：针对每个 Item 的指数退避。
   - `MaxOfRateLimiter`：组合多种限速器，取最保守的值。
3. **延迟入队**：允许将 Key 延迟指定时间后入队（例如等待其他依赖资源就绪）。

**Reconcile 循环（核心调谐逻辑）**

Worker 从 WorkQueue 取出资源 Key，从 Indexer（本地缓存）读取该资源的最新状态，执行调谐逻辑。调谐的核心原则是 **幂等性（Idempotency）**——无论对同一个资源调谐多少次，最终状态应该是一致的。

调谐函数的典型模式：

```go
func (c *Controller) reconcile(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    // 1. 从本地缓存获取资源
    obj, err := c.lister.Get(name)
    // 2. 检查资源是否被删除（DeletionTimestamp 非空）
    if obj.DeletionTimestamp != nil {
        // 执行清理逻辑
        return c.cleanup(obj)
    }
    // 3. 计算当前状态与期望状态的差异
    // 4. 执行操作，驱动状态收敛
    // 5. 更新资源的 status 子资源
}
```

返回 error 时，Worker 会将 Key 重新放入队列（经过限速），实现自动重试。

---

## 1.3 控制器分类概览

Kubernetes 生态系统中有数十个控制器，按功能可以分为五大类：

### 1.3.1 副本类控制器（Replication Controllers）

这一类控制器负责确保指定数量的 Pod 副本始终处于运行状态。

| 控制器 | 说明 |
|--------|------|
| **ReplicaSet** | 最基础的副本控制器，通过 Selector 管理一组 Pod，确保 Pod 副本数符合期望。通常不直接使用，而是被 Deployment 间接管理。 |
| **Deployment** | 在 ReplicaSet 之上提供声明式更新、滚动更新、回滚、暂停/恢复等高级部署策略。这是生产环境中最常用的工作负载控制器。 |
| **StatefulSet** | 专为有状态应用设计，提供稳定的 Pod 名称、网络标识、持久存储和有序的部署/终止。适用于数据库、消息队列等场景。 |
| **DaemonSet** | 确保每个（或部分）节点上恰好运行一个 Pod 副本。适用于日志采集、监控代理、网络插件等节点级守护进程。 |

### 1.3.2 批处理类控制器（Batch Controllers）

这一类控制器用于管理需要运行到完成的短期任务。

| 控制器 | 说明 |
|--------|------|
| **Job** | 管理一次性任务，确保指定数量的 Pod 成功完成。支持并行执行、失败重试、超时控制。 |
| **CronJob** | 在 Job 之上增加了定时调度能力，按照 Cron 表达式周期性地创建 Job。 |

### 1.3.3 伸缩类控制器（Scaling Controllers）

这一类控制器根据负载动态调整 Pod 副本数。

| 控制器 | 说明 |
|--------|------|
| **HorizontalPodAutoscaler (HPA)** | 根据 CPU/内存使用率或自定义指标，自动调整 Deployment、ReplicaSet 或 StatefulSet 的副本数。 |
| **VerticalPodAutoscaler (VPA)** | 根据历史使用情况，自动调整 Pod 中容器的资源请求和限制（不属核心控制器，由社区维护）。 |

### 1.3.4 节点类控制器（Node Controllers）

这一类控制器负责管理和监控集群节点。

| 控制器 | 说明 |
|--------|------|
| **Node Controller** | 监控节点健康状态，在节点不可达时驱逐 Pod、标记节点为 NotReady 等。 |
| **Node IPAM Controller** | 为节点分配 CIDR 块，管理 Pod 网络 IP 地址池。 |

### 1.3.5 服务与网络类控制器（Service & Network Controllers）

| 控制器 | 说明 |
|--------|------|
| **EndpointSlice Controller** | 根据 Service 的 Selector 和 Pod 状态，自动生成 EndpointSlice 对象，记录后端 Pod 的 IP 和端口。 |
| **Ingress Controller** | 监听 Ingress 资源变更，更新负载均衡器的配置。Ingress Controller 不是核心组 __ 件，由各厂商或社区提供（如 ingress-nginx、Traefik、Istio Gateway）。 |

### 1.3.6 其他基础设施控制器

| 控制器 | 说明 |
|--------|------|
| **Garbage Collector** | 基于 OwnerReference 实现级联删除，自动清理孤儿资源。 |
| **TTL Controller** | 为已完成 Job 或已过期资源设置 TTL，到期后自动删除。 |
| **Namespace Controller** | 在 Namespace 被删除时，清理该命名空间下的所有资源。 |
| **ServiceAccount Token Controller** | 自动为 ServiceAccount 创建 Secret（K8s 1.22-）或通过 TokenRequest API 生成令牌。 |

# 2. ReplicaSet

## 2.1 ReplicaSet 的作用与 YAML 定义

### 2.1.1 设计动机

ReplicaSet（简称 RS）是 Kubernetes 中最基础的 Pod 副本管理单元。它的唯一职责是：**在任何时刻，确保匹配特定标签选择器的 Pod 数量等于用户指定的 replicas 值**。

在 Kubernetes 的早期版本中，这个角色由 ReplicationController（RC）承担。随着 Kubernetes API 的演进，Selector 从等式匹配（`app=nginx`）扩展到集合匹配（`env in (production, staging)`），ReplicaSet 作为更通用的替代方案被引入，RC 现已标记为废弃。

### 2.1.2 为什么需要 ReplicaSet

直接创建裸 Pod（Bare Pod）存在以下问题：

1. **无自愈能力**：裸 Pod 因节点故障或被意外驱逐（Eviction）而终止后，不会自动重建。
2. **无法水平伸缩**：要增加副本，必须手动创建新 Pod 并确保名称不冲突。
3. **无法滚动更新**：更新镜像需要手动删除旧 Pod 并创建新 Pod，过程无法控制。

ReplicaSet 通过控制循环解决了这些问题。

### 2.1.3 YAML 示例

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
    tier: frontend
spec:
  replicas: 3                        # 期望副本数
  minReadySeconds: 30                # Pod 就绪后至少稳定运行多长时间才计入"就绪"
  selector:                          # 选择器，用于匹配要管理的 Pod
    matchLabels:
      app: nginx
    matchExpressions:                # 可选，更灵活的选择器
      - key: tier
        operator: In
        values:
          - frontend
          - backend
  template:                          # Pod 模板
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

### 2.1.4 关键字段解析

- **replicas**：期望的 Pod 副本数量。控制器会持续监控，若实际数量不符，则创建或删除 Pod 以达到期望值。设为 0 相当于停止该 RS（Pod 全部删除），但 RS 对象本身保留。
- **minReadySeconds**：Pod 通过 Readiness Probe 变为 Ready 后，必须持续保持 Ready 状态达到该秒数，控制器才将其计入"可用副本"（availableReplicas）。这个参数对于滚动更新至关重要——防止新 Pod 短暂 Ready 后又崩溃，导致假性可用。
- **template**：Pod 模板，与独立 Pod 的 spec 相同。需要注意的是，template 中的 `metadata.labels` 必须与 selector 匹配，否则 API Server 会拒绝创建（webhook 校验）。
- **selector**：标签选择器，负责定义 RS 的管理范围。RS 不能管理没有匹配标签的 Pod，也不能管理标签匹配但不属于该 RS 的 Pod（因为 RS 会检查 OwnerReference）。

---

## 2.2 Selector 机制

Selector 是 ReplicaSet 识别和管理 Pod 的核心机制。Kubernetes 支持两种类型的选择器：

### 2.2.1 matchLabels（等式选择器）

最简单的匹配方式，要求 Pod 的标签与指定键值对完全一致。多个 matchLabels 之间是 AND 关系。

```yaml
selector:
  matchLabels:
    app: nginx
    env: production
    # Pod 必须同时具有 app=nginx 和 env=production 两个标签
```

### 2.2.2 matchExpressions（集合选择器）

更灵活的匹配方式，支持四种操作符：

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `In` | 标签值在指定集合中 | `tier In [frontend, backend]` |
| `NotIn` | 标签值不在指定集合中 | `tier NotIn [database]` |
| `Exists` | 标签键存在（值任意） | `version`（只要有此标签） |
| `DoesNotExist` | 标签键不存在 | `decomissioned`（必须无此标签） |

```yaml
selector:
  matchLabels:
    app: nginx
  matchExpressions:
    - key: tier
      operator: In
      values:
        - frontend
        - backend            # tier 为 frontend 或 backend
    - key: version
      operator: NotIn
      values:
        - canary             # version 不能是 canary
    - key: environment
      operator: Exists       # 必须有 environment 标签
```

matchLabels 和 matchExpressions 可以同时指定，此时 Pod 必须同时满足所有条件（AND 逻辑）。

### 2.2.3 Selector 的不可变性

ReplicaSet 创建后，`selector` 字段不可修改。这是因为 Kubernetes 需要在创建 RS 时确保 template 的 labels 包含 selector 要求的所有标签，如果后续允许修改 selector，可能导致：
- RS 失去对已有 Pod 的管理（Pod 不再匹配 selector），造成孤儿 Pod。
- RS 接管了不应属于它的 Pod（Pod 刚好匹配了新 selector）。

如果确实需要修改 selector，只能删除并重建 RS（或者通过 Deployment 的滚动更新，由 Deployment 创建新的 RS）。

---

## 2.3 ReplicaSet 控制器工作原理

ReplicaSet Controller 是 kube-controller-manager 中的一个独立 goroutine，其工作流程如下：

### 2.3.1 第一步：事件驱动

RS Controller 通过 Informer 机制监听 ReplicaSet 和 Pod 的变更事件：

```
事件源                            触发的调谐
─────────────────────────────────────────────────
新的 RS 被创建                    → 按 RS 的 template 创建 Pod
RS 的 replicas 字段被修改         → 扩容或缩容 Pod
Pod 的 OwnerReference 指向该 RS   → 检查 Pod 状态，更新 RS status
Pod 被删除（非缩容触发）           → 检测到副本不足，创建新 Pod
Pod 变为 Ready/NotReady          → 更新 RS 的 readyReplicas
节点故障导致 Pod 被驱逐            → 在其他节点创建新 Pod
```

### 2.3.2 第二步：调谐逻辑

RS 控制器的核心逻辑可以概括为以下伪代码：

```
func reconcile(rs) {
    活跃Pod = 筛选所有 OwnerReference 指向该 RS 且 phase != Failed/Succeeded 的 Pod
    多余Pod = 活跃Pod中labels不匹配selector的（被接管或被手动修改了labels）

    // 删除多余 Pod
    for _, pod := range 多余Pod {
        如果 Owner 指向该 RS 但不匹配 selector → 删除
        如果 Owner 不指向该 RS 但匹配 selector → 跳过（RS 接管失败，因为 Pod 已被其他 RS 管理）
    }

    当前副本数 = len(活跃Pod) - 待删除的Pod数
    差异 = rs.spec.replicas - 当前副本数

    if 差异 < 0 {
        // 缩容：删除多余 Pod
        按优先级排序 Pod（优先删除未调度、未就绪的 Pod）
        删除 |差异| 个 Pod
    } else if 差异 > 0 {
        // 扩容：创建新 Pod
        创建差异个 Pod（基于 template）
    }

    更新 rs.status.replicas、rs.status.readyReplicas、rs.status.availableReplicas
}
```

### 2.3.3 第三步：缩容时的 Pod 优先级

当 RS 需要缩容时，控制器不会随机删除 Pod，而是按照以下优先级排序：

1. **Pending Pod**（尚未被调度）优先删除——因为它们还没运行起来，删除代价最小。
2. **未就绪的 Pod**（NotReady）优先于就绪的 Pod。
3. **创建时间晚的 Pod**优先删除——保留更早创建的 Pod（这些 Pod 可能已经"预热"好了缓存）。
4. **分布集中的 Pod**优先删除——如果某个节点上有过多匹配的 Pod，优先删除该节点上的 Pod 以实现跨节点的均匀分布。
5. **Pod Conditions 异常的 Pod**优先于正常的 Pod。
6. 如果以上条件都相同，则随机选择。

### 2.3.4 OwnerReference 保护

ReplicaSet 在创建 Pod 时，会在 Pod 的 metadata 中设置 OwnerReference：

```yaml
metadata:
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: nginx-rs
      uid: 12345-67890
      controller: true          # 标记该 RS 是该 Pod 的管理控制器
      blockOwnerDeletion: true  # 级联删除：RS 删除时阻止 Pod 被意外保留
```

`controller: true` 确保了当 Pod 同时匹配多个 RS 时，只有被标记为 `controller: true` 的 RS 才负责该 Pod。这解决了"两个 RS 争抢同一 Pod"的问题。

---

## 2.4 ReplicaSet vs ReplicationController

### 2.4.1 为什么 RC 被废弃

ReplicationController（RC）是 Kubernetes 最早的 Pod 副本管理器，它在 v1 版本引入。ReplicaSet 在 v1.2 版本引入，作为 RC 的进化版本。两者的核心区别：

| 特性 | ReplicationController | ReplicaSet |
|------|-----------------------|------------|
| API 组 | `v1`（核心 API） | `apps/v1` |
| 选择器 | 仅支持等式匹配（`app=nginx`） | 支持等式匹配 + 集合匹配（matchExpressions） |
| 与 Deployment 集成 | 旧版 Deployment 使用 RC 管理 Pod | 现代 Deployment 使用 RS 管理 Pod |
| 状态 | 已废弃（但仍然可用） | 推荐使用 |

RC 选择器的局限性在复杂场景中会暴露问题。例如，假设一个 RC 的选择器是 `version=1.0`，当需要区分 `version=1.0-beta` 和 `version=1.0-stable` 时，等式选择器无法做到，因为 `version=1.0` 会同时匹配两者。而 RS 的 matchExpressions 可以通过 `version NotIn [1.0-beta]` 来排除。

### 2.4.2 迁移建议

- 新部署直接使用 Deployment（底层自动管理 RS），无需手动创建 RS。
- 已有 RC 的集群不需要紧急迁移，RC 在未来很长一段时间内仍然可用。
- 如果手动管理 Pod 副本（罕见场景），使用 RS 而非 RC。

---

## 2.5 孤立 Pod 与接管

### 2.5.1 孤立 Pod 的产生

以下场景会导致 Pod 成为"孤立 Pod"（Orphan Pod）：

**场景一：修改 Pod 标签**

如果运维人员手动修改了一个由 RS 管理的 Pod 的标签，使 Pod 不再匹配 RS 的 selector，RS 会认为该 Pod 已消失，于是创建一个新 Pod 来维持副本数。而原来的 Pod 依然在运行，但没有控制器管理它，成为孤立 Pod。

```bash
# 假设 RS nginx-rs 的 selector 是 app=nginx
# 修改 Pod 标签
kubectl label pod nginx-rs-abc123 app=not-nginx --overwrite
# 结果：RS 创建一个新 Pod，原来的 Pod 变成孤立 Pod
```

**场景二：删除 RS 而不删除 Pod**

使用 `--cascade=orphan` 参数删除 RS 时，Pod 不会被删除，成为孤立 Pod。

```bash
kubectl delete rs nginx-rs --cascade=orphan
```

### 2.5.2 接管（Adoption）

ReplicaSet 的接管指的是 RS 将符合其 selector 但未被任何控制器管理的 Pod 纳入管理。

接管过程：

1. RS Controller 在调谐时不仅检查 OwnerReference 指向自己的 Pod，还扫描集群中所有匹配其 selector 的 Pod。
2. 对于匹配 selector 但 OwnerReference 为空或 `controller: false` 的 Pod，RS 会尝试接管。
3. 如果该 Pod 的 OwnerReference 中已有另一个控制器标记为 `controller: true`，RS 会跳过该 Pod（不争抢）。
4. 接管成功后，RS 在 Pod 的 OwnerReference 中添加自己。

实际工程中，手动利用接管实现"将一个 RS 的 Pod 迁移到另一个 RS"是不推荐的。正确的方式是通过 Deployment 的滚动更新或更改 Service 的 selector 来切换流量。

### 2.5.3 安全排查

```bash
# 查找集群中没有控制器的孤立 Pod
kubectl get pods --all-namespaces -o json | jq '.items[] | select(.metadata.ownerReferences == null) | .metadata.name'

# 查看 Pod 的 OwnerReference
kubectl get pod <pod-name> -o yaml | grep -A5 ownerReferences
```

---

# 3. Deployment

## 3.1 Deployment 设计目标

Deployment 是 Kubernetes 中最重要的高级工作负载控制器，它在 ReplicaSet 之上提供了一层抽象，核心设计目标包括：

### 3.1.1 声明式更新（Declarative Updates）

用户只需修改 Deployment YAML（例如 `kubectl edit deploy/nginx` 或 `kubectl set image deploy/nginx nginx=nginx:1.22`），Deployment Controller 会自动完成后续的所有操作——包括创建新 ReplicaSet、执行滚动更新、清理旧 ReplicaSet。

相比之下，手动操作 ReplicaSet 实现更新需要：

```bash
# 反例：手动更新（不推荐）
kubectl scale rs nginx-v1 --replicas=0
kubectl create -f nginx-v2-rs.yaml
# 问题：有停机时间、无流量切换、无进度监控
```

### 3.1.2 滚动更新（Rolling Updates）

Deployment 支持零停机时间的滚动更新：逐批将旧版本 Pod 替换为新版本，整个过程对外提供不间断服务。

滚动更新的关键权衡：
- **速度优先**：允许更多 Pod 在新旧版本之间临时存在（更大的 maxSurge/maxUnavailable），更新更快但可能消耗更多资源。
- **稳定优先**：严格控制每次替换的 Pod 数量，更新较慢但风险更低。

### 3.1.3 版本管理与回滚

Deployment 自动保存每次更新的历史记录（通过保留旧 ReplicaSet），支持：

- **查看历史**：`kubectl rollout history deployment/nginx`
- **回滚到上一版本**：`kubectl rollout undo deployment/nginx`
- **回滚到指定版本**：`kubectl rollout undo deployment/nginx --to-revision=3`
- **查看具体版本的差异**：比较不同版本的 Pod Template

### 3.1.4 多副本集协调

Deployment 的核心工作模型是"多代 ReplicaSet 并存"：

```
Deployment (replicas: 3)
├── ReplicaSet v1 (replicas: 0)    ← 旧版本，已缩容至 0，保留以备回滚
├── ReplicaSet v2 (replicas: 1)    ← 中间版本，逐步缩容中
└── ReplicaSet v3 (replicas: 2)    ← 当前版本，逐步扩容中
```

这种设计保证了：
- 任何时候都可以回滚到任意历史版本（因为 RS 和 template 都保留了）。
- 滚动更新过程中如果新版本有问题，可以快速中止并回滚。
- 不影响已经稳定服务的老版本 Pod。

---

## 3.2 Deployment YAML 详解

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
  annotations:
    description: "Nginx web server deployment"
spec:
  replicas: 3                                    # 期望副本数
  revisionHistoryLimit: 10                       # 保留的历史 ReplicaSet 数量
  minReadySeconds: 30                            # Pod 就绪后需持续稳定运行此时间
  progressDeadlineSeconds: 600                   # 滚动更新须在此时限内完成，否则置为 Failed
  paused: false                                  # 暂停滚动更新
  strategy:                                      # 更新策略
    type: RollingUpdate                          # RollingUpdate 或 Recreate
    rollingUpdate:
      maxSurge: 25%                              # 更新期间允许多出多少个 Pod（数字 或 百分比）
      maxUnavailable: 25%                        # 更新期间允许最多多少个 Pod 不可用
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: "1.21"
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
```

### 3.2.1 关键字段详解

**strategy**

- **type: RollingUpdate**：默认策略，滚动更新。通过逐步替换 Pod 实现零停机更新。
- **type: Recreate**：先删除所有旧版本 Pod，再创建新版本 Pod。适用于不支持多版本并存的场景（如数据库 Schema 变更）。

**revisionHistoryLimit**

默认值为 10。Deployment 每次更新会创建一个新的 ReplicaSet，旧的 ReplicaSet 被保留（缩容至 0）以备回滚。`revisionHistoryLimit` 限制了保留的旧 RS 数量。超出该值时，最老的 RS 会被删除。

**调优建议**：
- 历史版本占用 etcd 存储空间，生产环境建议设为 3~5。
- 如果经常需要回滚到多个版本之前，适当增大此值。
- 设得太小可能导致需要回滚到已经被清理的版本，此时需要通过 Git 中的 YAML 重新部署。

**minReadySeconds**

Pod 变为 Ready 后，必须保持 Ready 状态至少 `minReadySeconds` 秒，Deployment 才将其计入可用副本（availableReplicas）。这个参数的主要作用是：

1. **防止"假性可用"**：Pod 的 Readiness Probe 可能短暂通过，但应用实际上还没加载完数据。minReadySeconds 提供了一个缓冲期。
2. **减慢滚动更新速度**：新 Pod Ready 后需要等待 minReadySeconds 秒，Deployment 控制器才会继续处理下一批 Pod，这为问题发现提供了时间窗口。

**progressDeadlineSeconds**

滚动更新必须在 `progressDeadlineSeconds` 秒内完成，否则 Deployment 的状态会被标记为 `ProgressDeadlineExceeded`（置为 Failed）。

触发该超时的条件：
- Pod 一直无法变为 Ready（镜像拉取失败、健康检查失败等）。
- maxUnavailable/maxSurge 设置过小导致更新极慢（不太可能，但有理论可能）。

超过 deadline 后，Deployment 控制器**不会自动回滚**，但会停止操作。用户需要手动处理：

```bash
# 查看失败原因
kubectl describe deployment nginx-deployment
# 手动回滚
kubectl rollout undo deployment/nginx-deployment
```

**paused**

将 `paused` 设为 `true` 后，Deployment 控制器停止对当前 Deployment 做任何更改（包括滚动更新、缩扩容）。但对已存在的 ReplicaSet 的扩缩容仍由 RS 控制器负责。

这是实现金丝雀发布的基础——先创建新 RS，再暂停 Deployment，然后通过修改 Service selector 将部分流量切到新版本验证。

```bash
kubectl rollout pause deployment/nginx-deployment
# 更新镜像等操作会被记录，但不执行滚动更新
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
# 恢复后，Deployment 会执行一次完整的滚动更新
kubectl rollout resume deployment/nginx-deployment
```

---

## 3.3 滚动更新策略（RollingUpdate）

### 3.3.1 maxSurge

`maxSurge` 定义了滚动更新期间，允许超出期望副本数的最大 Pod 数量。它可以是绝对值（如 `3`）或百分比（如 `25%`）。

**工作原理**：在更新过程中，Deployment Controller 先创建新版本 Pod（因此 Pod 总数会超过期望值），等新 Pod Ready 后再删除旧 Pod。

| replicas | maxSurge | 更新期间最多 Pod 数 | 效果 |
|----------|----------|-------------------|------|
| 3 | 1 | 4 | 先创建 1 个新 Pod，再删 1 个旧 Pod |
| 10 | 25% | 13 (ceil) | 先创建 3 个新 Pod，再删 3 个旧 Pod |

**为什么允许超出期望值**：如果每次必须先删除旧 Pod 才能创建新 Pod，服务容量会暂时下降，可能无法承载当前流量。允许临时超额，保证了服务容量不下降。

### 3.3.2 maxUnavailable

`maxUnavailable` 定义了滚动更新期间，允许不可用的 Pod 最大数量。它也可以是绝对值或百分比。

| replicas | maxUnavailable | 含义 |
|----------|---------------|------|
| 10 | 2 | 更新期间至少有 8 个 Pod 保持可用 |
| 10 | 25% | 更新期间至少有 8 个 Pod 保持可用（四舍五入向下取整） |

### 3.3.3 参数组合效果

maxSurge 和 maxUnavailable 的不同组合决定了滚动更新的行为模式：

| maxSurge | maxUnavailable | 更新模式 |
|----------|---------------|---------|
| 1 | 0 | **逐 Pod 替换，零停机**。每次创建 1 个新 Pod，Ready 后删除 1 个旧 Pod。更新速度最慢，最安全。 |
| 0 | 1 | **逐 Pod 替换，允许短暂降级**。每次删除 1 个旧 Pod，再创建 1 个新 Pod。Pod 总数不超过期望值，但服务容量可能短暂下降。 |
| 25% | 25% | **批量替换**。每次同时处理一批 Pod（replicas × 25%），更新速度快但瞬时资源开销大。 |

### 3.3.4 计算示例

假设 replicas = 10，maxSurge = 25%，maxUnavailable = 25%：

```
maxSurge = ceil(10 × 25%) = ceil(2.5) = 3
maxUnavailable = floor(10 × 25%) = floor(2.5) = 2

更新过程：
第1轮：创建 3 个新 Pod（总数 = 10 + 3 = 13）
    → 等待新 Pod Ready
    → 删除 3 个旧 Pod（总数 = 10）
    → 等待更多旧 Pod 终止

第2轮：创建 3 个新 Pod
    → 等待新 Pod Ready
    → 删除 3 个旧 Pod

第3轮：创建 3 个新 Pod
    → 等待新 Pod Ready
    → 删除 3 个旧 Pod（至此新 Pod = 10，旧 Pod = 0）

第4轮：仅剩 1 个旧 Pod，创建 1 个新 Pod，删除 1 个旧 Pod。
完成。
```

### 3.3.5 注意事项

1. **maxSurge 不能为 0 且 maxUnavailable 不能同时为 0**——否则更新无法进行（不能创建新 Pod 也不能删除旧 Pod）。
2. **百分比取整规则**：
   - maxSurge 向上取整（`ceil`），确保至少创建 1 个新 Pod。
   - maxUnavailable 向下取整（`floor`），确保最小可用保证不被突破。
3. 如果 replicas = 1，无论参数如何，实际上都只能逐个替换。

---

## 3.4 Recreate 策略

### 3.4.1 适用场景

Recreate 策略的逻辑非常简单：先删除所有旧版本 Pod，等它们完全终止后，再创建新版本 Pod。

```
步骤：
1. 将旧 RS 的 replicas 缩减至 0（删除所有旧 Pod）
2. 等待所有旧 Pod 完全终止
3. 创建新 RS
4. 将新 RS 的 replicas 扩展至期望值
```

Recreate 适用于以下场景：

- **数据库 Schema 不兼容**：新版本应用需要修改数据库表结构，旧版本无法工作。此时必须全部停机更新。
- **只允许单实例运行的应用**：某些传统应用（如老旧的企业软件）不支持多实例并发，如果使用滚动更新会出现两个实例同时运行。
- **需要完整的资源释放**：旧版本占用的存储卷、网络端口等资源必须先完全释放。
- **更新速度要求极高，且可接受停机**：批量删除和创建通常比逐个滚动要快。

### 3.4.2 YAML 配置

```yaml
spec:
  strategy:
    type: Recreate
```

### 3.4.3 与 RollingUpdate 的对比

| 维度 | RollingUpdate | Recreate |
|------|--------------|----------|
| 停机时间 | 零 | 有（取决于 Pod 启动时间） |
| 资源消耗 | 更新期间可能超额分配 | 更新期间无超额 |
| 版本兼容性要求 | 新旧版本必须兼容 | 无要求 |
| 回滚复杂度 | 低（快速切回旧 RS） | 中（需要重新启动旧版本） |
| 适用场景 | Web 服务、API、微服务 | 数据库、传统单体应用 |

---

## 3.5 更新过程深度解析

### 3.5.1 完整的滚动更新时序

当用户更新 Deployment 的 Pod Template（例如修改镜像版本）时，以下事件按顺序发生：

```
时间线
│
├─ T0: kubectl set image deploy/nginx nginx=nginx:1.22
│       → API Server 更新 Deployment spec.template
│
├─ T1: Deployment Controller 检测到 template 变更
│       → 计算新旧 template 的 Hash（pod-template-hash）
│       → 创建新的 ReplicaSet (RS-v2, replicas=0)
│
├─ T2: Deployment Controller 计算本轮更新目标
│       → 当前状态: RS-v1(replicas=3), RS-v2(replicas=0)
│       → 目标状态: RS-v1(replicas=? ), RS-v2(replicas=? )
│       → 根据 maxSurge/maxUnavailable 计算第一轮操作
│
├─ T3: Deployment Controller 扩容 RS-v2
│       → RS-v2 replicas: 0 → 1 (maxSurge 允许)
│       → RS-v2 Controller 创建 Pod
│       → Scheduler 调度 Pod
│       → kubelet 启动容器
│       → Readiness Probe 通过
│       → 等待 minReadySeconds
│
├─ T4: Deployment Controller 缩容 RS-v1
│       → RS-v1 replicas: 3 → 2
│       → RS-v1 Controller 按优先级删除 1 个 Pod
│       → API Server 发送 SIGTERM 给 Pod
│       → 等待 terminationGracePeriodSeconds (默认30s)
│       → 强制 SIGKILL
│
├─ T5: 重复 T3-T4，直到 RS-v1 replicas=0, RS-v2 replicas=3
│
├─ T6: 更新完成
│       → Deployment status.conditions 中新增 Progressing=True 条件
│       → 旧 RS (RS-v1) 保留（replicas=0），以备回滚
│
└─ T7: 如果 revisionHistoryLimit 超限，删除最老的 RS
```

### 3.5.2 pod-template-hash

每当 Deployment 的 Pod Template 发生变化，Deployment Controller 会计算 template 的 Hash 值，并以标签的形式附加到 ReplicaSet 和 Pod 上：

```yaml
metadata:
  labels:
    app: nginx
    pod-template-hash: 7d5c8f9b6c   # <-- 自动添加
```

这个 Hash 的作用：

1. **唯一标识每个 ReplicaSet**：不同版本的 RS 有不同的 hash，避免 selector 冲突。
2. **关联 Pod 与 RS**：RS 的 selector 包含 `pod-template-hash` 标签，使得每个 RS 只会管理自己创建的 Pod，不会争抢其他 RS 的 Pod。
3. **滚动更新时的精确控制**：Deployment Controller 通过 hash 精确知道哪些 Pod 属于旧版本，哪些属于新版本。

### 3.5.3 Status 字段的变化

在更新过程中，Deployment 的 `status` 字段实时反映更新进度：

```yaml
status:
  observedGeneration: 3              # Deployment Controller 观察到的 spec 版本
  replicas: 5                        # 当前总 Pod 数（可能包含新旧版本）
  updatedReplicas: 3                 # 已更新到最新版本的 Pod 数
  readyReplicas: 4                   # 就绪的 Pod 数
  availableReplicas: 4              # 可用的 Pod 数（Ready + minReadySeconds 达标）
  unavailableReplicas: 1            # 不可用的 Pod 数
  conditions:
    - type: Available
      status: "True"
      reason: MinimumReplicasAvailable
    - type: Progressing
      status: "True"
      reason: ReplicaSetUpdated
```

解读技巧：
- `updatedReplicas < replicas`：滚动更新仍在进行中。
- `unavailableReplicas > maxUnavailable`：可能出现了问题（如镜像拉取失败）。
- `updatedReplicas == replicas && availableReplicas == replicas`：滚动更新已完成。

---

## 3.6 回滚机制

### 3.6.1 回滚的实现原理

Deployment 的回滚并非真正"撤销操作"，而是**切换到旧 ReplicaSet**。

当用户执行 `kubectl rollout undo` 时：

1. Deployment Controller 查找到上一个版本的 ReplicaSet（保存在 `deployment.kubernetes.io/revision` 注解中）。
2. 将该旧 RS 的 Pod Template 设置为 Deployment 的当前 template。
3. 触发一次新的滚动更新——从当前版本"更新"到旧版本。
4. 这会产生一个新的 ReplicaSet（revision 号递增），尽管它的 template 与旧版本相同。

```bash
# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

# 回滚到指定的历史版本
kubectl rollout undo deployment/nginx-deployment --to-revision=3

# 查看可回滚的历史版本
kubectl rollout history deployment/nginx-deployment

# 查看特定版本的详细信息
kubectl rollout history deployment/nginx-deployment --revision=3
```

### 3.6.2 revision 的存储

Deployment 的历史版本信息不存储在 Deployment 对象本身，而是散布在：

1. **ReplicaSet 对象**：每个历史 RS 上有一个注解 `deployment.kubernetes.io/revision: "3"`。
2. **Deployment 的 `revisionHistoryLimit`**：限制保留多少个历史 RS。
3. RS 的 `ownerReferences` 指向 Deployment，Deployment 由此获知 RS 的归属。

### 3.6.3 重要注意事项

**回滚不会恢复其他资源**：回滚只恢复 Pod Template（镜像、环境变量、资源请求等），不会恢复以下内容：
- ConfigMap 和 Secret 的内容变更（需要单独回滚）。
- PVC（PersistentVolumeClaim）的变更（存储不受 Deployment 管理）。
- Service、Ingress 等网络配置的变更。

**变更类型区分（通过 CHANGE-CAUSE）**：

```bash
# 设置 CHANGE-CAUSE 注解以记录变更原因
kubectl annotate deployment/nginx-deployment \
  kubernetes.io/change-cause="Upgrade nginx to 1.22"

kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         Upgrade nginx to 1.22
# 3         Rollback to revision 1
```

`CHANGE-CAUSE` 仅在执行 `kubectl apply/create` 时被 Deployment Controller 自动从注解中复制到 revision，`kubectl edit` 不会触发设置。

---

## 3.7 暂停/恢复更新

### 3.7.1 用例：实现金丝雀发布的前置操作

```bash
# 1. 创建 Deployment
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# 2. 暂停 Deployment
kubectl rollout pause deployment/nginx

# 3. 修改镜像（但不会触发滚动更新）
kubectl set image deployment/nginx nginx=nginx:1.22

# 4. 修改资源限制（同样不会触发滚动更新）
kubectl set resources deployment/nginx --limits=cpu=200m,memory=256Mi

# 5. 恢复 Deployment——此时一次性执行所有累积的变更
kubectl rollout resume deployment/nginx
```

### 3.7.2 暂停后的限制

暂停（`paused: true`）期间：
- Deployment Controller **不会**执行任何滚动更新操作。
- **但是**，对已存在的 ReplicaSet 的直接扩缩容仍由 RS Controller 执行（如果 RS 的 replicas 被手动修改）。
- pause 状态会被记录在 Deployment 的 `.spec.paused` 字段中。
- pause 状态在 Deployment 更新期间持久化，即使 Controller Manager 重启也不会丢失。

### 3.7.3 常见问题

**Q: 暂停后修改 replicas 会怎样？**

A: Deployment Controller 不会执行滚动更新，但会修改当前活跃 RS 的 replicas（直接扩缩容）。这是允许的，因为扩缩容不属于"更新"。

**Q: 如何查看 Deployment 是否处于暂停状态？**

```bash
kubectl get deployment nginx-deployment -o jsonpath='{.spec.paused}'
# 输出: true
```

---

## 3.8 Deployment 更新失败场景与处理

### 3.8.1 场景一：镜像拉取失败（ImagePullBackOff / ErrImagePull）

**症状**：
- 新 Pod 状态为 `ImagePullBackOff` 或 `ErrImagePull`。
- Deployment `status.unavailableReplicas` 持续增高。
- 最终 `ProgressDeadlineExceeded`。

**根因**：
- 镜像名称错误（拼写错误、tag 不存在）。
- 私有镜像仓库认证失败（imagePullSecrets 未配置或过期）。
- 网络问题导致无法访问镜像仓库。

**处理**：

```bash
# 1. 查看具体错误
kubectl describe pod <new-pod-name> | grep -A5 "Events:"

# 2. 常见修复
kubectl set image deployment/nginx nginx=nginx:1.22  # 修正 tag
kubectl rollout undo deployment/nginx                 # 快速回滚恢复服务

# 3. 确认 imagePullSecrets 正确
kubectl get serviceaccount default -o yaml
```

**预防**：
- 使用 CI/CD 流程自动验证镜像存在性。
- 对于生产环境，使用不可变 tag 或 Digest（如 `nginx@sha256:...`）。

### 3.8.2 场景二：健康检查失败

**症状**：
- 新 Pod 状态为 `Running` 但 `READY 0/1`。
- Readiness Probe 返回失败。
- 新 Pod 无法通过 minReadySeconds 等待期。

**根因**：
- 应用的启动时间超过了 `initialDelaySeconds`。
- Readiness Probe 路径错误或响应格式不匹配。
- 应用本身存在启动 bug，启动失败。

**处理**：

```bash
# 查看 Readiness Probe 配置
kubectl describe pod <pod-name>

# 查看 Pod 日志
kubectl logs <pod-name> --previous  # 如果容器已重启

# 调整 Readiness Probe
kubectl edit deployment/nginx
# 增加 initialDelaySeconds 或调整探针路径
```

### 3.8.3 场景三：资源配额不足

**症状**：
- 新 Pod 状态为 `Pending`。
- `kubectl describe pod` 显示 `Insufficient cpu/memory`。
- 或显示 `exceeded quota`（ResourceQuota 限制）。

**处理**：

```bash
# 检查节点资源
kubectl top nodes
kubectl describe node <node-name> | grep -A5 "Allocated resources"

# 检查命名空间资源配额
kubectl describe resourcequota -n <namespace>

# 检查 LimitRange
kubectl describe limitrange -n <namespace>

# 解决方案：
# - 扩容节点
# - 降低 Pod 资源请求
# - 驱逐低优先级 Pod
# - 调整 ResourceQuota 限制
```

### 3.8.4 场景四：更新过程中被中断

如果 Controller Manager 在更新过程中崩溃重启，Deployment Controller 重启后会：
1. 从 API Server 重新 List 所有 Deployment、ReplicaSet 和 Pod。
2. 重建本地缓存（Informer 的 Indexer）。
3. 重新计算每个 Deployment 的当前状态 vs 期望状态。
4. 从之前中断的点继续执行滚动更新。

这得益于 Deployment Controller 的无状态设计——所有状态都存储在 etcd 中，Controller 本身不保存任何内存状态。这也是为什么 Controller Manager 可以水平扩展（通过 Leader Election 选举主实例）。

### 3.8.5 状态转换图

```
Deployment 更新状态机：

  Progressing=True              Progressing=True
  (NewReplicaSetCreated) ───→ (NewReplicaSetAvailable)
        │                            │
        │ 任何 Pod 失败/超时          │ 所有 Pod 就绪
        ▼                            ▼
  Progressing=False            Available=True
  (ProgressDeadlineExceeded)   Complete
```

---

## 3.9 金丝雀发布

### 3.9.1 原理

金丝雀发布（Canary Deployment）是一种渐进式发布策略：将新版本部署到少量实例上，只让一部分用户流量访问新版本，验证无误后再全量切换。

Kubernetes 本身没有内置"金丝雀"控制器，但可以通过组合 Deployment 和 Service 的 Label Selector 来实现。

### 3.9.2 实现步骤

**第一步：创建稳定的主 Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-stable
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
      track: stable
  template:
    metadata:
      labels:
        app: nginx
        track: stable
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**第二步：创建 Service，Selector 不包括 track 标签**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx          # 注意：只有 app=nginx，没有 track
  ports:
  - port: 80
    targetPort: 80
```

此时 Service 将流量分发给所有 `app: nginx` 的 Pod，包括 stable 和 canary。

**第三步：创建金丝雀 Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-canary
spec:
  replicas: 1           # 只部署少量副本
  selector:
    matchLabels:
      app: nginx
      track: canary
  template:
    metadata:
      labels:
        app: nginx
        track: canary
        version: canary
    spec:
      containers:
      - name: nginx
        image: nginx:1.22  # 新版本
```

由于 Service 的 selector 是 `app: nginx`，金丝雀 Pod 也会接收流量。金丝雀的比例 = canary replicas / (stable replicas + canary replicas) = 1/5 = 20%。

**第四步：验证与推广**

验证金丝雀版本无误后，更新 stable Deployment 的镜像为 `nginx:1.22`，然后删除 canary Deployment。

### 3.9.3 缺点与替代方案

手动组合 Deployment + Service 实现金丝雀发布存在以下限制：

1. **流量比例不精确**：基于 Pod 数量的流量分配是"近似"的，当 Pod 数量少时不准确。例如 1 个 canary Pod + 4 个 stable Pod 时，Service 的 iptables/IPVS 负载均衡是基于连接的，不保证精确的 20% 流量分配。
2. **手动操作多**：需要手动创建/删除 canary Deployment，调整流量比例需要修改 replicas。
3. **缺乏监控集成**：需要自行建立监控和回滚决策机制。

**更好的替代方案**：

- **Argo Rollouts**：CNCF 项目，原生支持蓝绿部署、金丝雀部署、流量加权。
- **Flagger**：Weaveworks 开源，与 Istio、Linkerd、NGINX 等集成，支持自动化金丝雀分析和回滚。
- **Istio VirtualService**：通过 Istio 的流量路由实现精确的流量切分（例如 5% 流量到新版本）。
- **OpenKruise Rollout**：阿里开源的渐进式交付框架。

# 4. StatefulSet

## 4.1 有状态应用的需求

### 4.1.1 有状态 vs 无状态

Deployment 管理的 Pod 是"无状态"的（Stateless）：所有 Pod 完全相同，可以随时被替换。对于 Web 服务器、API 网关这类应用，这种同构性是合适的。

但以下类型的有状态应用需要更多保证：

| 需求 | 无状态应用 | 有状态应用 |
|------|-----------|-----------|
| Pod 身份 | 随机名称 (nginx-7d5c8f-abc) | 稳定的序号 (mysql-0, mysql-1, mysql-2) |
| 网络标识 | 无固定 DNS | 固定的 DNS 名称 |
| 存储 | 通常不需要持久存储 | 每个 Pod 需要独立的持久卷 |
| 启动/停止顺序 | 无序、可并行 | 有序（主从架构必须先启动主库） |
| 集群拓扑 | 不关心 | Pod 需要知道其他 Pod 的身份（如 etcd 集群成员） |

### 4.1.2 StatefulSet 提供的保障

1. **稳定的网络标识**：每个 Pod 有固定的序号和 DNS 名称。Pod 重新调度到其他节点后，名称和 DNS 不变。
2. **稳定的持久存储**：每个 Pod 绑定独立的 PVC。Pod 重建后，重新挂载同一个 PVC，数据不丢失。
3. **有序的部署和终止**：Pod 按照序号 0 → N-1 顺序创建，按照 N-1 → 0 顺序删除。这对于主从复制等场景至关重要。
4. **有序的滚动更新**：Pod 按照 N-1 → 0 逆序更新，确保主副本最后更新。

---

## 4.2 StatefulSet YAML 详解

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  serviceName: mysql-headless        # 必须关联一个 Headless Service
  replicas: 3
  podManagementPolicy: OrderedReady  # OrderedReady 或 Parallel
  revisionHistoryLimit: 10
  minReadySeconds: 10
  updateStrategy:
    type: RollingUpdate              # RollingUpdate 或 OnDelete
    rollingUpdate:
      partition: 0                   # 分区更新（仅更新 >= partition 的 Pod）
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 60   # 优雅终止等待时间
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - name: mysql
          containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        readinessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: mysql
    spec:
      accessModes:
        - ReadWriteOnce           # 每个Pod独立访问
      storageClassName: fast-ssd  # 指定存储类
      resources:
        requests:
          storage: 20Gi
```

### 4.2.1 关键字段详解

**serviceName（必填）**

StatefulSet 必须关联一个 Headless Service（ClusterIP 为 None）。这个 Service 负责为每个 Pod 提供 DNS 解析。Headless Service 的定义：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None          # Headless Service
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

注意：除了 Headless Service，通常还会创建一个普通的 ClusterIP Service 供客户端访问（流量被分发到任意一个 Pod）。

**podManagementPolicy**

- **OrderedReady**（默认）：Pod 严格按序号顺序创建/删除/更新。对于每个 Pod，控制器会等待其 Ready 后，才处理下一个。这保证了集群启动时的拓扑正确性。
- **Parallel**：Pod 可以并行创建/删除，不等待其他 Pod 就绪。适用于对启动顺序不敏感的有状态应用（如纯计算型应用，或应用层自己处理集群发现）。

**volumeClaimTemplates**

StatefulSet 不会在 `spec.template.spec.volumes` 中定义卷，而是通过 `volumeClaimTemplates` 为每个 Pod 动态创建 PVC。每个 Pod 的 PVC 命名规则为 `<volumeClaimTemplates.metadata.name>-<StatefulSetName>-<序号>`，例如 `data-mysql-0`。

**重要特性**：Pod 被删除或重新调度，原有的 PVC 不会被自动删除（K8s 1.27+ 可以配置保留策略）。当 Pod 在新节点重建时，会重新挂载原来的 PVC。这保证了数据的持久性。

---

## 4.3 网络标识

### 4.3.1 DNS 名称规则

StatefulSet 中每个 Pod 有一个稳定、可预测的 DNS 名称，格式为：

```
<Pod名称>.<Headless Service名称>.<Namespace>.svc.<clusterDomain>
```

例如，名为 `mysql` 的 StatefulSet 在 `default` 命名空间：

```
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

即使 Pod 被删除重建，甚至迁移到其他节点，这个 DNS 名称保持不变。

### 4.3.2 为什么需要稳定网络标识

以 MySQL 主从复制集群为例：

```ini
# 每个 MySQL Pod 启动时需要知道：
# - 自己的 server-id（可以用序号）
# - 如果自己是 Slave，Master 的地址是什么

# Pod mysql-0（Master）
server-id = 0

# Pod mysql-1（Slave）
server-id = 1
CHANGE MASTER TO MASTER_HOST='mysql-0.mysql-headless.default.svc.cluster.local'
```

如果没有稳定的 DNS 名称，Slave 无法找到 Master 的地址（因为 Pod IP 和名称都会变化）。

### 4.3.3 应用如何获取自己的序号和 DNS 名称

通过 Downward API 可以将 Pod 的名称和序号注入到环境变量或文件中：

```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name        # mysql-2
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace   # default
```

应用可以从 `POD_NAME` 中提取序号（如 `mysql-2` → `2`）。

---

## 4.4 Pod 顺序管理

### 4.4.1 OrderedReady 模式（默认）

**创建过程（0 → N-1）**

```
T0: StatefulSet 创建，replicas=3
T1: 创建 mysql-0 → 等待调度 → 等待容器启动 → 等待 Readiness Probe 通过
T2: mysql-0 Ready 后，创建 mysql-1 → 等待 Ready
T3: mysql-1 Ready 后，创建 mysql-2 → 等待 Ready
T4: 全部 Ready，StatefulSet 状态变为 Complete
```

**为什么顺序很重要**：以 ZooKeeper 集群为例，ZK 节点的配置文件中包含所有节点的地址。如果先启动 `zk-2`，zk-2 会发现 `zk-0` 和 `zk-1` 不存在，可能进入异常状态或启动失败。有序启动确保集群中最关键的节点首先就绪。

**缩容过程（N-1 → 0）**

缩容是逆序的。例如 replicas: 3 → 1：

```
T0: 删除 mysql-2 → 等待终止 → 等待 PVC 清理策略
T1: mysql-2 终止后，删除 mysql-1 → 等待终止
T2: mysql-1 终止后，完成缩容。mysql-0 继续运行。
```

逆序删除确保最后创建的从副本先被移除，最小化对集群的影响。

**更新过程（逆序）**

滚动更新也是逆序的（N-1 → 0）。replicas=3 的更新顺序：mysql-2 → mysql-1 → mysql-0。这与 Deployment 正序或无序的更新方式不同，原因是：
- 至少有一个副本保持旧版本（更安全）。
- 主副本（序号最小）最后更新，避免主从切换带来的混乱。

### 4.4.2 Parallel 模式

在 `podManagementPolicy: Parallel` 模式下：

- 创建：所有 Pod 同时创建，不等待前一个就绪。
- 删除：所有 Pod 同时删除。
- 更新：所有 Pod 同时更新（受 maxUnavailable 限制，但不保证顺序）。

适用于：
- 应用自建了集群发现和管理机制（如 Consul、etcd 的自动集群形成）。
- 应用不关心启动顺序，可以容忍部分节点先启动带来的短暂不一致。

---

## 4.5 持久存储

### 4.5.1 volumeClaimTemplates 的工作原理

StatefulSet 的 `volumeClaimTemplates` 字段定义了 PVC 模板。当 StatefulSet 创建 Pod 时，会自动为每个 Pod 创建一个独立的 PVC。

```
StatefulSet: mysql
├── Pod: mysql-0
│   └── PVC: data-mysql-0  (自动创建)
│       └── PV: pvc-abc123
│           └── 云磁盘: disk-xyz
├── Pod: mysql-1
│   └── PVC: data-mysql-1  (自动创建)
│       └── PV: pvc-def456
│           └── 云磁盘: disk-uvw
└── Pod: mysql-2
    └── PVC: data-mysql-2  (自动创建)
        └── PV: pvc-ghi789
            └── 云磁盘: disk-rst
```

关键特性：

1. **PVC 生命周期独立于 Pod**：删除 Pod 不会自动删除 PVC（K8s 1.27 之前）。这确保了即使 Pod 被驱逐或重新调度，数据仍然保留。
2. **Pod 重建后重新挂载同一 PVC**：StatefulSet Controller 会在创建新 Pod 时，通过标签查找属于该序号的 PVC，并重新挂载。
3. **存储拓扑感知**：如果 PV 绑定到特定可用区的云盘，Pod 重新调度时会被约束到同一可用区。

### 4.5.2 PVC 保留策略（Kubernetes 1.27+）

从 Kubernetes 1.27 开始（Beta），StatefulSet 支持通过 `persistentVolumeClaimRetentionPolicy` 控制 PVC 的保留行为：

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain   # Retain 或 Delete
    whenScaled: Retain    # Retain 或 Delete
```

- **whenDeleted**：StatefulSet 被删除时，PVC 的处理策略。
  - `Retain`（默认）：保留 PVC 和底层数据。
  - `Delete`：自动删除 PVC（注意：底层 PV 的回收策略也必须是 Delete）。
- **whenScaled**：StatefulSet 缩容时，PVC 的处理策略。
  - `Retain`（默认）：保留缩容 Pod 的 PVC。
  - `Delete`：自动删除缩容 Pod 的 PVC。

在 1.27 之前，缩容时 PVC 总是被保留。这可能导致大量"孤儿"PVC 占用存储资源，需要手动清理。

### 4.5.3 存储类选择

为 StatefulSet 选择存储类时需要考虑：

| 存储类 | 访问模式 | 延迟 | 适用场景 |
|--------|---------|------|---------|
| 本地 SSD (hostPath/local PV) | ReadWriteOnce | 极低 | 对延迟敏感，且能接受节点故障时数据不可用（如 Kafka、Elasticsearch） |
| 网络块存储 (EBS/PD) | ReadWriteOnce | 低 | 大多数数据库（MySQL、PostgreSQL） |
| 分布式文件系统 (CephFS/GlusterFS) | ReadWriteMany | 中 | 需要多 Pod 共享数据的场景 |
| 网络文件存储 (NFS/EFS) | ReadWriteMany | 中 | 共享配置、日志、上传文件 |

注意：StatefulSet 的 volumeClaimTemplates 会为每个 Pod 创建独立 PVC。如果使用 ReadWriteMany 存储，多个 Pod 可以访问同一个文件系统（但通常不推荐，因为有数据竞争风险）。

---

## 4.6 StatefulSet 更新策略

### 4.6.1 RollingUpdate（默认）

StatefulSet 的滚动更新与 Deployment 不同：

1. **逆序更新**：从序号最大的 Pod 开始更新（N-1 → 0），而非 Deployment 的随机顺序。
2. **严格就绪等待**：每个 Pod 必须变为 Ready 后，才处理下一个 Pod（OrderedReady 模式）。
3. **maxUnavailable 仅支持整数**：StatefulSet 的 maxUnavailable 不支持百分比（因为 Pod 总数通常较少，百分比没有意义）。
4. **没有 maxSurge**：StatefulSet 不支持超额创建 Pod（因为每个 Pod 绑定独立的 PVC，不能创建"多余"的 Pod）。

更新过程：

```
初始: mysql-0 (v1), mysql-1 (v1), mysql-2 (v1)
Step 1: mysql-2 删除（v1） → 创建 mysql-2 (v2) → 等待 Ready
Step 2: mysql-1 删除（v1） → 创建 mysql-1 (v2) → 等待 Ready
Step 3: mysql-0 删除（v1） → 创建 mysql-0 (v2) → 等待 Ready
完成
```

### 4.6.2 OnDelete 策略

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

此策略下，StatefulSet Controller 不会主动重启任何 Pod。只有当用户手动删除 Pod 后，Controller 才会用新 template 创建替换的 Pod。

**适用场景**：
- 需要精确控制每个 Pod 的更新时间（如数据库维护窗口）。
- 更新需要额外的外部操作（如手动切换主从关系）。
- 金丝雀测试：先手动删除一个从副本，验证新版行为，再逐步删除其他副本。

---

## 4.7 分区更新

### 4.7.1 partition 参数

StatefulSet 的滚动更新策略支持 `partition` 参数，实现"分期更新"：

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2     # 仅更新序号 >= 2 的 Pod
```

只有序号大于或等于 `partition` 的 Pod 会被更新到新 template。序号小于 partition 的 Pod 保持不变（即使被删除重建，也用旧 template）。

### 4.7.2 实现金丝雀发布

```
StatefulSet replicas=5, partition=4:

mysql-4: 新版本(v2) ✓  ← 金丝雀 Pod
mysql-3: 旧版本(v1)      ← partition 之下，不受影响
mysql-2: 旧版本(v1)
mysql-1: 旧版本(v1)
mysql-0: 旧版本(v1)
```

验证 `mysql-4` 没有问题后，逐步减小 partition：

```bash
# 验证通过，再更新一个 Pod
kubectl patch statefulset mysql -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":3}}}}'
# 现在 mysql-3 和 mysql-4 都会更新到 v2
```

### 4.7.3 与 OnDelete 的对比

| 策略 | partition | OnDelete |
|------|-----------|----------|
| 控制粒度 | 按序号范围 | 手动逐个删除 |
| 自动化程度 | 半自动（修改 partition 触发更新） | 手动（手动删除 Pod 触发更新） |
| 分区感知 | 支持多个"时间段" | 无内置分区概念 |
| 适用场景 | 需精确控制更新范围的场景 | 需要每个 Pod 都有特殊处理步骤的场景 |

---

## 4.8 StatefulSet 缩放

### 4.8.1 扩容注意事项

**扩容时（replicas 增加）**，StatefulSet Controller 按顺序（从小到大）创建新 Pod。整个过程是"惰性"的——必须等待上一个 Pod Ready 后才创建下一个。

扩容过程中需要注意：

1. **前置条件检查**：如果新 Pod 需要的 PVC 对应的 PV 存储类不可用、或存储配额已满，扩容会停滞。
2. **集群拓扑检查**：某些有状态应用（如 ZK、etcd）要求奇数个节点以实现多数派投票。直接扩容到偶数可能导致无效扩容。
3. **数据复制时间**：新加入的副本需要从主节点同步数据，启动时间可能远长于 Readiness Probe 的 initialDelaySeconds。应设置足够的 `initialDelaySeconds` 和 `periodSeconds`。

### 4.8.2 缩容注意事项

**缩容时（replicas 减少）**，StatefulSet Controller 按逆序（从大到小）删除 Pod。

缩容时需要特别小心：

1. **数据完整性**：删除 Pod 前，确保该 Pod 上的数据已经完成备份或不需要保留。
2. **PVC 未自动删除**：在 K8s 1.27 之前，缩容不会删除 PVC。如果需要删除，须手动执行：

   ```bash
   kubectl delete pvc data-mysql-2
   ```

3. **集群重新配置**：某些有状态应用（如 Redis Cluster、Cassandra）需要在删除节点前执行 decommission 操作，将数据迁移到其他节点。StatefulSet 的 `PreStop` 生命周期钩子可以自动化这一步骤：

   ```yaml
   lifecycle:
     preStop:
       exec:
         command:
         - /bin/bash
         - -c
         - |
           # 从集群中移除自己
           redis-cli cluster reset
           # 等待数据迁移完成
           sleep 30
   ```

4. **避免数据丢失**：例如，一个 3 节点的 etcd 集群，如果直接缩容到 2 节点，虽然理论上仍保持多数派（2/2），但在一个节点故障时丧失写入能力。最佳实践是保持奇数节点。

### 4.8.3 全量缩容到 0

```bash
kubectl scale statefulset mysql --replicas=0
```

将所有 Pod 删除，但 PVC 保留（K8s 1.27 之前默认保留）。重新扩容时，Pod 按序号重新创建，并重新挂载原有 PVC。这在存储成本敏感的场景中很有用（如开发环境夜间关机）。

---

# 5. DaemonSet

## 5.1 DaemonSet 设计目标

DaemonSet（简称 DS）确保所有（或部分）节点上运行一个 Pod 的副本。它是一种特化的控制器，设计目标与 Deployment/StatefulSet 截然不同：

### 5.1.1 核心语义

- **每个节点一个 Pod**：当新节点加入集群时，DaemonSet Controller 自动在该节点上创建 Pod。当节点离开集群时，对应的 Pod 被 GC 回收。
- **节点级服务**：DS 管理的 Pod 通常提供与节点强相关的服务（日志采集、监控、网络、存储插件）。
- **无视 Scheduler**：默认情况下，DaemonSet 绕过调度器（Scheduler）直接分配 Pod 到节点，除非显式指定了 `nodeName` 才会被调度器接管。

### 5.1.2 与 Deployment 的本质区别

| 维度 | Deployment | DaemonSet |
|------|-----------|-----------|
| Pod 分布 | 任意节点，由调度器决定 | 每个匹配节点一个 |
| 副本数控制 | 用户指定 replicas | 自动等于节点数 |
| 更新方式 | maxSurge / maxUnavailable | maxUnavailable（仅整数） |
| 典型用例 | Web 服务、API | 日志、监控、网络、存储代理 |

---

## 5.2 DaemonSet YAML 详解

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-logging
  namespace: kube-system
  labels:
    app: fluentd
spec:
  revisionHistoryLimit: 10
  minReadySeconds: 10
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2          # 最多同时有几个节点的 Pod 不可用
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:                # 容忍污点，允许调度到有污点的节点
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
        operator: Exists
      hostNetwork: true           # 使用宿主机网络
      hostPID: true               # 可以访问宿主机的进程
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 500m
            memory: 500Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluentd/etc
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluentd-config
```

### 5.2.1 关键特性说明

- **tolerations**：默认情况下，DaemonSet Pod 不会被调度到有 NoSchedule 污点的节点（如 Control Plane Node）。显式添加 tolerations 可以让 DaemonSet 在控制平面节点上也运行。
- **hostNetwork/hostPID**：许多 DaemonSet 应用需要访问宿主机级别的资源（网络、进程、文件系统）。示例中 Fluentd 需要读取 `/var/log` 和容器日志路径。
- **volume source: hostPath**：DaemonSet 最常见的卷类型是 hostPath，直接挂载宿主机目录。在多节点集群中，每个节点的 `hostPath` 内容不同。

---

## 5.3 调度机制

### 5.3.1 默认调度（bypass scheduler）

DaemonSet Controller 为每个匹配条件的节点生成 Pod，并直接设置 `nodeName` 字段（而非让调度器决策）。这有以下影响：

1. **绕过调度器**：调度器不参与 DaemonSet Pod 的调度。
2. **无视资源请求**：默认不检查节点是否有足够的 CPU/内存（可能导致节点资源过载）。
3. **必须使用 `NodeAffinity` 或 `nodeSelector` 来限定目标节点**。

### 5.3.2 通过 nodeSelector 限制目标节点

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
        # 仅在 disktype=ssd 的节点上运行
```

### 5.3.3 通过 nodeAffinity 限制目标节点

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - bare-metal
```

`requiredDuringSchedulingIgnoredDuringExecution`：硬性要求，节点必须满足。`preferredDuringSchedulingIgnoredDuringExecution`：软性偏好，尽量满足。

### 5.3.4 节点亲和性默认添加（Kubernetes 1.12+）

从 K8s 1.12 开始，DaemonSet Controller 会自动为 Pod 添加与节点角色相关的 nodeAffinity。例如，Control Plane 节点的 DaemonSet Pod 会自动添加 `node-role.kubernetes.io/control-plane` nodeAffinity。用户可以通过显式设置 nodeAffinity 来覆盖默认值。

### 5.3.5 Pod 驱逐与优先级

DaemonSet Pod 默认被标记为 `priorityClassName: system-node-critical`（高优先级）。在资源紧张时，kubelet 的驱逐管理器（Eviction Manager）会优先驱逐低优先级的普通 Pod，保护 DaemonSet Pod。但这并不意味着 DaemonSet Pod 永不驱逐——在极端内存不足时，连 `system-critical` Pod 也可能被驱逐。

### 5.3.6 节点删除时的行为

当节点从集群中移除时：
- DaemonSet Controller 检测到节点不存在，不会为该节点创建新 Pod。
- 该节点上已有的 DaemonSet Pod 会被 kubelet 清理（如果节点正常下线）或由 GC 清理（如果节点异常宕机）。

---

## 5.4 DaemonSet 更新策略

### 5.4.1 RollingUpdate

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 3    # 最多同时有 3 个节点的 Pod 不可用
```

与 Deployment 的滚动更新不同之处：

1. **不支持 maxSurge**：因为每个节点只能有一个 Pod，无法临时创建额外的 Pod。
2. **逐节点替换**：每次选择 maxUnavailable 个节点，删除旧 Pod，等待新 Pod 就绪，再处理下一批。
3. **没有固定顺序**：更新顺序没有保证（不像 StatefulSet 的逆序）。

更新过程：

```
初始: Node1(旧), Node2(旧), Node3(旧), Node4(旧), Node5(旧)
Step 1: 删除 Node1, Node2, Node3 上的旧 Pod (maxUnavailable=3)
        → 创建新 Pod (新)
        → 等待新 Pod Ready
Step 2: 删除 Node4, Node5 上的旧 Pod
        → 创建新 Pod (新)
完成
```

### 5.4.2 OnDelete

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

DaemonSet Controller 只在 Pod 被手动删除后才创建新版本 Pod。适用于需要严格控制每个节点的更新时机的场景。

### 5.4.3 查看更新状态

```bash
kubectl rollout status daemonset/fluentd-logging -n kube-system

kubectl get ds fluentd-logging -n kube-system
# NAME               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
# fluentd-logging    5         5         5       5            5           <none>          10d
```

- **DESIRED**：应该运行此 DaemonSet 的节点数。
- **CURRENT**：已经运行 Pod 的节点数（可能包含新旧版本）。
- **READY**：就绪的 Pod 数。
- **UP-TO-DATE**：已更新到最新 template 的 Pod 数。
- **AVAILABLE**：可用的 Pod 数（Ready + minReadySeconds 达标）。

---

## 5.5 典型应用场景

### 5.5.1 场景一：日志采集（Fluentd / Filebeat）

日志采集是 DaemonSet 最经典的应用场景。每个节点上的 Fluentd 负责：

- 读取 `/var/log/containers/*.log`（容器标准输出日志）。
- 读取应用写入 `hostPath` 的自定义日志文件。
- 解析、过滤日志，并发送到集中式日志系统（Elasticsearch、Loki、Kafka）。

```yaml
# Fluentd DaemonSet 简化示例
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16-debian-1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: containerlogs
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: containerlogs
        hostPath:
          path: /var/lib/docker/containers
```

### 5.5.2 场景二：监控代理（Prometheus Node-Exporter）

Node Exporter 收集节点级别的硬件和操作系统指标（CPU、内存、磁盘、网络），通常以 DaemonSet 方式部署。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.0
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          mountPropagation: HostToContainer
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

**为什么必须用 hostPath 和 hostNetwork**：Node Exporter 需要读取宿主机的 `/proc`、`/sys` 文件系统，这些在容器内是隔离的（容器的 `/proc` 只显示容器自己的进程）。通过 hostPath 挂载，容器可以访问宿主机的内核信息。

### 5.5.3 场景三：网络插件（Calico / Flannel / Cilium）

CNI 网络插件通常有一个"节点代理"组件，以 DaemonSet 形式在每个节点上运行，负责：

- 配置节点的 iptables/IPVS 规则。
- 管理网络接口和路由表。
- 处理 Pod 的网络流量（如 Calico 的 Felix、Cilium 的 Agent）。

```yaml
# Calico Node DaemonSet 关键配置
spec:
  hostNetwork: true
  containers:
  - name: calico-node
    image: calico/node:v3.25.0
    env:
    - name: CALICO_IPV4POOL_CIDR
      value: "192.168.0.0/16"
    securityContext:
      privileged: true       # 需要特权模式操作网络
    volumeMounts:
    - mountPath: /var/lib/calico
      name: var-lib-calico
    - mountPath: /var/run/calico
      name: var-run-calico
  volumes:
  - name: var-lib-calico
    hostPath:
      path: /var/lib/calico
  - name: var-run-calico
    hostPath:
      path: /var/run/calico
```

### 5.5.4 场景四：存储插件（CSI Node Plugin）

CSI（Container Storage Interface）驱动通常分为 Controller 插件（Deployment）和 Node 插件（DaemonSet）。Node 插件负责在节点上：

- 挂载/卸载存储卷。
- 格式化文件系统。
- 管理卷的生命周期。

```yaml
# AWS EBS CSI Node DaemonSet
spec:
  containers:
  - name: ebs-plugin
    image: public.ecr.aws/ebs-csi-driver/aws-ebs-csi-driver:v1.22.0
    args:
    - node
    - --endpoint=$(CSI_ENDPOINT)
    securityContext:
      privileged: true
    volumeMounts:
    - name: kubelet-dir
      mountPath: /var/lib/kubelet
      mountPropagation: Bidirectional    # 允许将卷传播给 Pod
    - name: device-dir
      mountPath: /dev
  volumes:
  - name: kubelet-dir
    hostPath:
      path: /var/lib/kubelet
      type: Directory
  - name: device-dir
    hostPath:
      path: /dev
      type: Directory
```

---

# 6. Job

## 6.1 Job 设计目标

Job 是 Kubernetes 中管理**一次性任务**（Run-to-Completion）的控制器。与 Deployment/StatefulSet（持续运行的服务）不同，Job 管理的 Pod 在执行完毕后应该终止，而不是被重启。

### 6.1.1 核心设计目标

1. **任务完成保证**：确保指定数量的 Pod 成功完成（而非永远运行）。
2. **并行执行**：允许多个 Pod 并行处理任务，提高吞吐量。
3. **失败处理**：自动重试失败的 Pod，直到达到成功次数或重试上限。
4. **资源清理**：任务完成后，支持自动或手动清理 Pod 和 Job 对象。

### 6.1.2 典型场景

- **ETL 任务**：从数据库提取数据、转换、加载到数据仓库。
- **批量处理**：图片转码、视频处理、邮件群发。
- **数据库迁移**：运行 Schema 变更脚本。
- **科学计算**：模型训练、蒙特卡洛模拟。
- **备份与恢复**：定期备份数据库或文件系统。

---

## 6.2 Job YAML 详解

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
  labels:
    app: math
spec:
  parallelism: 3                     # 并行运行的 Pod 数
  completions: 100                   # 需要成功完成的 Pod 总数
  backoffLimit: 4                    # 失败重试次数上限（默认 6）
  activeDeadlineSeconds: 3600       # Job 整体超时时间（秒）
  ttlSecondsAfterFinished: 600      # Job 完成后自动清理时间（秒）
  completionMode: Indexed           # NonIndexed（默认）或 Indexed
  suspend: false                     # 挂起 Job
  template:
    spec:
      restartPolicy: OnFailure      # Job Pod 必须设为 OnFailure 或 Never（不能是 Always）
      containers:
      - name: pi
        image: perl:5.34
        command:
        - perl
        - -Mbignum=bpi
        - -wle
        - |
          use strict;
          use warnings;
          my $digits = 5000;
          print bpi($digits);
        resources:
          requests:
            cpu: 500m
            memory: 256Mi
```

### 6.2.1 关键字段详解

**parallelism**

控制同时运行的 Pod 数量。如果不指定或设为 0，默认值为 1（串行执行）。

- `parallelism=1, completions=N`：串行执行 N 次，一次一个 Pod。
- `parallelism=M, completions=N`：并行执行，最多同时 M 个 Pod 运行，总共需要 N 个 Pod 成功。

**completions**

Job 需要成功完成的 Pod 总数。默认值为 1（单个 Pod 成功完成即结束）。

- 如果 `completions` 未指定且 `parallelism` 也未指定，Job 创建一个 Pod 并等待其成功或失败。
- 如果指定了 `completions`，Job 会持续创建 Pod（受 `parallelism` 限制），直到有指定数量的 Pod 成功。

**backoffLimit**

失败重试次数上限。一个失败的 Pod 会计入重试计数。当重试次数达到 `backoffLimit` 后：

- Job 标记为 Failed。
- 所有还在运行的 Pod 被终止。
- 控制器不再创建新 Pod。

默认值为 6。建议根据任务特性设置：
- 快速失败的任务（如编译）：设小一些（2~3）。
- 可能需要多次重试才能成功的任务（如外部 API 调用）：设大一些（10~20）。

注意：`backoffLimit` 是按 Job 全局计算的，不是单个 Pod 的重试次数。

**activeDeadlineSeconds**

Job 的总超时时间（秒）。从 Job 创建开始计时，一旦超过此时间：

- 所有正在运行的 Pod 被终止。
- Job 状态变为 Failed，状态原因为 `DeadlineExceeded`。
- `activeDeadlineSeconds` 优先级高于 `backoffLimit`（即使 `backoffLimit` 还有剩余，超时也直接失败）。

**ttlSecondsAfterFinished**

Job 完成（成功或失败）后自动清理的等待时间（秒）。Kubernetes 的 TTL Controller 会在该时间后将 Job 及其 Pod 删除。如果不设置此值，Job 和 Pod 会一直保留，需要手动清理。生产环境强烈建议设置此值以避免集群中积累大量已完成 Job。

**completionMode**

Kubernetes 1.21+ 的新特性（Beta in 1.24, GA in 1.25）：

- **NonIndexed**（默认）：传统的 Job 模式，Pod 之间无索引区分。
- **Indexed**：每个 Pod 获得一个唯一的索引（0 到 completions-1），通过环境变量或文件注入到容器中。

---

## 6.3 Job 执行模式

### 6.3.1 模式一：单 Pod 单完成（Non-parallel Job）

```yaml
spec:
  completions: 1      # 默认值
  parallelism: 1      # 默认值
```

最简单的模式。创建一个 Pod，等待其成功（exit code 0）或失败。适用于简单的一次性任务。

### 6.3.2 模式二：固定完成次数（Fixed Completion Count）

```yaml
spec:
  completions: 100    # 需要 100 个 Pod 成功
  parallelism: 5      # 同时运行 5 个 Pod
```

Job Controller 维护一个计数器，每当一个 Pod 成功退出，计数器加 1。当计数器达到 `completions` 时，Job 完成。

这种模式适用于：
- 任务可以分割为固定数量的独立工作项。
- 每个 Pod 处理一个或多个工作项。
- 例如：对 1000 张图片进行转码，completions=1000，每个 Pod 处理 1 张图片。

### 6.3.3 模式三：工作队列（Work Queue Pattern）

```yaml
spec:
  # completions 未设置     # 不限制成功次数
  parallelism: 5          # 同时运行 5 个 Pod 消费消息队列
```

不指定 `completions`，Pod 自行决定何时结束。典型场景：
- Pod 从消息队列（RabbitMQ、Kafka）中拉取任务。
- 当队列为空时，Pod 自己以成功状态退出（exit 0）。
- 只要有 Pod 在运行，Job Controller 就保持活跃。
- 当所有 Pod 成功退出时，Job 完成。

这种模式的优点：
- 不需要预先知道工作项数量。
- 自动适应队列长度变化。
- Pod 之间可以协调（例如通过共享计数器或分布式锁避免重复消费）。

### 6.3.4 四种模式的对比

| 模式 | completions | parallelism | 适用场景 |
|------|------------|-------------|---------|
| 单 Pod | 默认 1 | 默认 1 | 简单的一次性任务 |
| 固定完成次数 | 指定 N | 指定 M | 已知工作量的并行处理 |
| 工作队列 | 不设置 | 指定 M | 生产者-消费者模型，动态工作量 |
| Indexed | 指定 N | 指定 M | 每个 Pod 需要知道自己处理的"分片" |

---

## 6.4 Job 失败处理与重试

### 6.4.1 Pod 失败类型

Pod 的失败分为两种：

1. **容器级别的失败**：容器以非零退出码退出。此时 kubelet 根据 `restartPolicy` 决定是否重启容器。
2. **Pod 级别的失败**：Pod 被驱逐（Eviction）、节点宕机、Pod 被删除等。对于 Job，`restartPolicy` 只能是 `OnFailure` 或 `Never`。

### 6.4.2 restartPolicy 的选择

```yaml
spec:
  template:
    spec:
      restartPolicy: OnFailure   # 容器失败时在同一 Pod 中重启容器
```

- **OnFailure**（推荐）：
  - 容器失败后，kubelet 在同一 Pod 中重启容器。
  - 重启仍然算同一个 Pod，不增加 `backoffLimit` 计数（在同一 Pod 内重启不计入失败）。
  - 只有当整个 Pod 失败（例如节点故障），才计入 `backoffLimit`。
  - 适合于：容器偶尔崩溃是正常的（如 OOM），在同一节点重启即可恢复。

- **Never**：
  - 容器失败后，kubelet 不重启容器。Pod 被标记为 Failed。
  - 每次容器失败都计入 `backoffLimit`。
  - Job Controller 会在其他节点上创建新的 Pod。
  - 适合于：任务不可重入（重启可能导致副作用），每次失败需要全新环境。

### 6.4.3 重试机制

```
Job 生命周期中的重试逻辑：

Pod 1 失败 → backoffLimit 计数器 + 1
Pod 2 创建 → 失败 → 计数器 + 1 → 达到 backoffLimit
→ Job 状态变为 Failed
→ 所有运行中的 Pod 被终止
```

**指数退避**：Job Controller 不是立即创建新 Pod，而是使用指数退避（10s, 20s, 40s, ... 最长 6 分钟）。这是为了避免持续对 API Server 和节点造成压力。

### 6.4.4 使用 restartPolicy=Never 时的 podFailurePolicy

Kubernetes 1.26+ 引入了 `podFailurePolicy`（Alpha→Beta），允许更精细地控制 Pod 失败处理：

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob
      onExitCodes:
        containerName: main
        operator: In
        values: [1, 2]              # 退出码 1 或 2 则中止 Job
    - action: Ignore
      onPodConditions:
      - type: DisruptionTarget      # 节点故障导致的 Pod 删除不算失败
```

- **Ignore**：不计入 `backoffLimit`。
- **FailJob**：立即将 Job 标记为失败。
- **Count**：正常计入 `backoffLimit`（默认行为）。

---

## 6.5 Job 清理策略

### 6.5.1 手动清理

```bash
# 删除 Job 及其 Pod
kubectl delete job pi-calculation

# 仅删除 Job，保留 Pod（--cascade=orphan）
kubectl delete job pi-calculation --cascade=orphan

# 删除所有已完成的 Job
kubectl delete jobs --field-selector status.successful=1 -n my-namespace

# 删除所有失败的 Job
kubectl delete jobs --field-selector status.failed=1 -n my-namespace
```

### 6.5.2 自动清理：TTL Controller

通过 `ttlSecondsAfterFinished` 自动清理：

```yaml
spec:
  ttlSecondsAfterFinished: 3600   # 1小时后自动删除
```

TTL Controller 会定期扫描集群中已完成的 Job（status 中有 `completionTime` 或 `failedTime`），如果 `当前时间 - 完成时间 > ttlSecondsAfterFinished`，则删除 Job 和所有关联的 Pod。

**工作原理**：
1. TTL Controller 是 kube-controller-manager 的一部分。
2. 使用 `TTLAfterFinished` Feature Gate，该特性从 K8s 1.21 开始稳定（GA）。
3. Controller 每 30 秒扫描一次符合条件的 Job。
4. 删除是级联的：删除 Job 会级联删除其创建的 Pod（通过 OwnerReference）。

### 6.5.3 清理策略的最佳实践

1. **始终为生产 Job 设置 ttlSecondsAfterFinished**，避免已完成 Job 的海量积累。
2. **根据合规要求设置 TTL**：某些行业要求保留任务记录 7 天、30 天等。
3. **对调试/测试 Job 设置短 TTL**（如 60 秒）。
4. **如果 Job 的输出需要查看日志，设置足够长的 TTL**（如 7 天），确保日志系统已采集完毕。

---

## 6.6 Indexed Job（Kubernetes 1.21+）

### 6.6.1 问题背景

传统的 Job 中，多个并行 Pod 之间没有区分。每个 Pod 不知道自己处理的是"第几个"任务。要实现数据分片处理，需要在应用层引入外部协调机制（如分布式队列或共享计数器）。

Indexed Job 为每个 Pod 分配一个唯一的、从 0 开始的整数索引，使 Pod 能够基于索引执行分片处理。

### 6.6.2 YAML 配置

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job-example
spec:
  completions: 10              # 需要有 10 个 Pod 成功
  parallelism: 3               # 同时运行 3 个 Pod
  completionMode: Indexed      # 启用索引模式
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: my-worker:latest
        env:
        - name: JOB_COMPLETION_INDEX    # Kubernetes 自动注入的索引
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
          # Pod-0/10 → JOB_COMPLETION_INDEX=0
          # Pod-1/10 → JOB_COMPLETION_INDEX=1
          # Pod-9/10 → JOB_COMPLETION_INDEX=9
        command:
        - python
        - process_batch.py
        - --shard-index=$(JOB_COMPLETION_INDEX)
        - --total-shards=10
```

### 6.6.3 索引的注入方式

Kubernetes 通过两种方式向 Pod 注入索引：

1. **环境变量**：`JOB_COMPLETION_INDEX`（自动注入）。
2. **注解**：`batch.kubernetes.io/job-completion-index: "5"`（在 Pod 的 metadata.annotations 中）。

### 6.6.4 应用场景

```python
# process_batch.py
import os
import sys

shard_index = int(os.environ['JOB_COMPLETION_INDEX'])
total_shards = int(sys.argv[2])  # 10

# 例如：处理数据库中的一部分记录
# Pod-0 处理 id % 10 == 0 的记录
# Pod-1 处理 id % 10 == 1 的记录
for record in query("SELECT * FROM table WHERE id %% ? = ?", total_shards, shard_index):
    process(record)
```

每个 Pod 独立处理自己的数据分片，不需要向外部协调服务注册任务。这大幅简化了批量并行处理的实现。

### 6.6.5 与 StatefulSet 序号的对比

| 特性 | Indexed Job Pod | StatefulSet Pod |
|------|----------------|-----------------|
| 索引范围 | 0 ~ completions-1 | 0 ~ replicas-1 |
| 生命周期 | 短生命周期，完成后终止 | 长生命周期，持续运行 |
| 网络标识 | 无固定 DNS | 有固定 DNS |
| 持久存储 | 无（除非手动挂载 PVC） | 有 volumeClaimTemplates |
| 序号复用 | 从不复用（Job 级别唯一） | Pod 被新 Pod 替代时复用 |

---

# 7. CronJob

## 7.1 CronJob 设计目标

CronJob 是 Job 的定时调度器，它在指定的时间点（或周期性间隔）创建 Job。其设计灵感来源于 Unix 的 `cron` 守护进程，但运行在 Kubernetes 的容器化环境中。

核心功能：
1. **定时执行**：按照 Cron 表达式在指定时间创建 Job。
2. **并发控制**：控制同一 CronJob 的多次触发是否可以并发执行。
3. **历史保留**：控制保留多少个成功和失败的 Job 记录。
4. **挂起/恢复**：临时暂停定时调度。

CronJob 控制器是 kube-controller-manager 的一个组件。它大约每 10 秒检查一次是否有应该触发的 CronJob。

---

## 7.2 CronJob YAML 详解

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: production
  labels:
    app: backup
spec:
  schedule: "0 2 * * *"                                  # Cron 表达式：每天凌晨2点
  timeZone: "Asia/Shanghai"                               # K8s 1.24+，时区设置
  startingDeadlineSeconds: 300                           # 任务启动的最大延迟时间
  concurrencyPolicy: Forbid                              # Allow / Forbid / Replace
  suspend: false                                          # 是否暂停调度
  successfulJobsHistoryLimit: 3                          # 保留的成功 Job 数量
  failedJobsHistoryLimit: 1                              # 保留的失败 Job 数量
  jobTemplate:
    spec:
      parallelism: 1
      completions: 1
      backoffLimit: 3
      activeDeadlineSeconds: 1800                       # 单个 Job 超时：30分钟
      ttlSecondsAfterFinished: 86400                     # Job 完成后 24 小时自动清理
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME \
                | gzip > /backup/db-$(date +%Y%m%d-%H%M%S).sql.gz
            envFrom:
            - secretRef:
                name: db-credentials
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            resources:
              requests:
                cpu: 500m
                memory: 512Mi
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

### 7.2.1 关键字段详解

**schedule（必填）**

Cron 表达式，定义了任务触发的时间规则。格式与 Unix Cron 相同：`分 时 日 月 星期`。

**timeZone（Kubernetes 1.24+）**

```yaml
spec:
  timeZone: "Etc/UTC"    # 或 Asia/Shanghai, America/New_York 等
```

在 1.24 之前，CronJob 的时区总是 UTC。1.24 引入的 Beta 特性允许指定任何 IANA 时区。如果不指定，默认使用 kube-controller-manager 的时区（通常是 UTC）。

**注意**：如果控制器管理器运行在 `Etc/UTC`，而 `schedule` 是 `"0 2 * * *"` 且 `timeZone` 是 `"Asia/Shanghai"`，任务会在北京时间凌晨 2 点触发（即 UTC 18:00）。

**startingDeadlineSeconds**

如果 CronJob Controller 因故（如 Controller Manager 宕机、网络分区）错过了触发时间，它会检查从错过的时间到现在是否超过了 `startingDeadlineSeconds`。如果未超过，它仍然触发该 Job；如果超过，则跳过此次触发。

```
错过了 2 次触发：
  第1次：错过 200 秒 → within 300s → 仍然触发
  第2次：错过 400 秒 → exceed 300s → 跳过
```

**默认值**：如果不设置，没有 deadline 限制（永远不跳过）。
**建议值**：设为 `schedule` 间隔的 2~3 倍，确保短期故障不会丢失任务，但长期故障不会堆积大量过期任务。

**concurrencyPolicy**

控制同一个 CronJob 的多个 Job 实例是否可以并发运行：

- **Allow**（默认）：允许多个 Job 实例同时运行。
- **Forbid**：如果前一次 Job 还未执行完毕，跳过本次触发。
- **Replace**：如果前一次 Job 还在运行，取消它，创建新 Job。

```yaml
concurrencyPolicy: Forbid
# 场景：每小时执行一次备份。如果备份耗时 90 分钟，
# 下次触发时前一次还未完成，跳过本次触发。
```

**选择建议**：
- 幂等性强的任务（如报告生成）：Allow。
- 任务不可同时执行（如数据库备份可能导致冲突）：Forbid。
- 需要确保最新数据（如缓存刷新）：Replace。

**successfulJobsHistoryLimit / failedJobsHistoryLimit**

保留的已完成 Job 数量（成功和失败分开计算）。默认值都是 3。

```yaml
successfulJobsHistoryLimit: 0   # 不保留成功的 Job（节省 etcd 空间）
failedJobsHistoryLimit: 10      # 保留更多失败的 Job 用于排查
```

设为 0 时，Job 完成后立即被清理。但注意：如果 `ttlSecondsAfterFinished` 同时设置，TTL Controller 和 CronJob Controller 都可能在尝试清理 Job，不会产生错误但行为由先执行者决定。

**suspend**

```bash
# 暂停 CronJob（不触发新 Job，但已运行的 Job 不受影响）
kubectl patch cronjob db-backup -p '{"spec":{"suspend":true}}'

# 恢复
kubectl patch cronjob db-backup -p '{"spec":{"suspend":false}}'
```

---

## 7.3 Cron 表达式格式

Kubernetes CronJob 的 `schedule` 字段使用标准的 Unix Cron 格式，包含 5 个字段：

```
┌─────────── 分钟 (0 - 59)
│ ┌─────────── 小时 (0 - 23)
│ │ ┌─────────── 日 (1 - 31)
│ │ │ ┌─────────── 月 (1 - 12)
│ │ │ │ ┌─────────── 星期 (0 - 6) (0 = 星期日)
│ │ │ │ │
* * * * *
```

### 7.3.1 特殊字符

| 字符 | 含义 | 示例 | 解释 |
|------|------|------|------|
| `*` | 任意值 | `* * * * *` | 每分钟 |
| `,` | 枚举 | `0 2,14 * * *` | 每天 2:00 和 14:00 |
| `-` | 范围 | `0 9-17 * * 1-5` | 工作日 9-17 点每整点 |
| `/` | 步长 | `*/15 * * * *` | 每 15 分钟 |
| `?` | 不指定 | 用于日或星期字段 | |

### 7.3.2 常见表达式

| 表达式 | 含义 |
|--------|------|
| `0 0 * * *` | 每天午夜 0 点 |
| `0 2 * * 0` | 每周日凌晨 2 点 |
| `0 0 1 * *` | 每月 1 号午夜 |
| `*/5 * * * *` | 每 5 分钟 |
| `0 */6 * * *` | 每 6 小时（0:00, 6:00, 12:00, 18:00） |
| `30 3 * * 1-5` | 工作日凌晨 3:30 |
| `0 0 1 1 *` | 每年 1 月 1 日 |

### 7.3.3 注意事项

1. **Kubernetes 不支持 `L`、`W`、`#` 等非标准 Cron 扩展**（Quartz 或某些 Cron 实现支持）。
2. **日 和 星期是 OR 关系**：如果同时指定了日（`*` 之外的特定值）和星期（`*` 之外的特定值），两者之一满足即触发。
3. **不支持秒级调度**：最小粒度是分钟。
4. **`schedule` 字段中的 `?`**：Kubernetes 的 Cron 实现支持 `?` 作为占位符，但通常不必要。

---

## 7.4 并发策略

### 7.4.1 Allow（默认）

```
Timeline:
  12:00 → Job-1 开始执行
  12:30 → CronJob 再次触发 → Job-2 开始执行（Job-1 仍在运行）
  12:45 → Job-1 完成
  13:00 → Job-2 完成

两个 Job 同时运行，互不干扰。
```

**适用场景**：
- 任务之间完全独立。
- 即使多个实例并发也不会产生副作用（如报告生成、数据清理）。
- 每个 Job 处理的数据是幂等的或互斥的。

**风险**：
- 并发过多可能超出集群容量。
- 如果不注意幂等性，可能导致数据错误。

### 7.4.2 Forbid

```
Timeline:
  12:00 → Job-1 开始执行
  12:30 → CronJob 触发 → 检测 Job-1 仍在运行 → 跳过本次触发
  13:15 → Job-1 完成
  13:30 → 正常触发 → Job-2 执行
```

**适用场景**：
- 任务依赖独占资源（如数据库锁、文件锁）。
- 任务不能重入（如数据库迁移脚本）。
- 资源密集型任务，并发可能导致 OOM。

**注意**：
- 如果任务执行时间波动大，可能导致多次触发被跳过。
- 可以通过 `startingDeadlineSeconds` 控制是否可以"追赶"错过的触发。

### 7.4.3 Replace

```
Timeline:
  12:00 → Job-1 开始执行
  12:30 → CronJob 触发 → 检测 Job-1 仍在运行
        → 删除 Job-1（所有 Pod 被终止）
        → 创建 Job-2 开始执行
```

**适用场景**：
- 每次执行的结果应该是最新的（如缓存刷新、数据同步）。
- 旧的执行结果已经没有意义。
- 例如：每 5 分钟从外部 API 同步一次数据。如果上次同步耗时超过 5 分钟，说明出现了问题，应该中止并重试。

**注意**：
- 被替换的 Job 中的 Pod 会被强制终止，可能丢失正在处理的中间数据。
- 应确保 Pod 能正确处理 SIGTERM 信号（优雅退出）。

---

## 7.5 CronJob 时区

### 7.5.1 Kubernetes 1.24 前的时区处理

在 K8s 1.24 之前，`schedule` 中的时间始终基于 kube-controller-manager 所在节点的系统时区（通常是 UTC）。这意味着：

- 在 UTC 集群中，`schedule: "0 2 * * *"` 意味着 UTC 时间凌晨 2 点触发。
- 如果运维人员在北京时间（UTC+8）工作，北京早上 10:00 对应 `schedule: "0 2 * * *"`。

这种设计容易导致理解偏差，特别是在夏令时（DST）变更的时区。

### 7.5.2 Kubernetes 1.24+ 的 timeZone 字段

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Shanghai"    # 北京时间凌晨 2 点 (UTC 18:00 前一天)
```

`timeZone` 必须是 IANA 时区数据库中的有效名称。常用时区：

| 时区标识 | 地区 | UTC 偏移 |
|----------|------|----------|
| `Etc/UTC` | 协调世界时 | +00:00 |
| `Asia/Shanghai` | 中国标准时间 | +08:00 |
| `America/New_York` | 美国东部时间 | -05:00（冬季） |
| `America/Los_Angeles` | 美国太平洋时间 | -08:00（冬季） |
| `Europe/London` | 英国时间 | +00:00（冬季）/ +01:00（夏季） |
| `Asia/Tokyo` | 日本标准时间 | +09:00 |

### 7.5.3 注意事项

1. **控制器管理器版本要求**：`timeZone` 字段在 K8s 1.24 中为 Beta，需要 API Server 和 Controller Manager 都 >= 1.24。
2. **夏令时**：如果时区观察夏令时（如 `America/New_York`），CronJob 在"回拨"（fall back）时可能会有重复触发（CronJob 不做特殊处理）。
3. **迁移**：从老版本升级集群后，已有的 CronJob 默认按原 UTC 时区运行，不会自动切换。

# 8. HPA (HorizontalPodAutoscaler)

## 8.1 HPA 工作原理

HPA 是 Kubernetes 实现"弹性伸缩"的核心控制器。它根据 Pod 的资源使用情况（CPU、内存）或自定义指标，自动调整 Deployment、ReplicaSet 或 StatefulSet 的副本数。

### 8.1.1 完整工作流程

```
                        ┌──────────────────┐
                        │   Metrics Server  │  (集群内)
                        │   或 Prometheus   │  (自定义指标)
                        └────────┬─────────┘
                                 │ 每15秒采集一次 Pod 指标
                                 ▼
┌──────────────────┐    ┌──────────────────┐
│   HPA Controller │───▶│   获取目标Pod的  │
│   (每15秒循环)    │    │   CPU/Mem 指标   │
└────────┬─────────┘    └──────────────────┘
         │
         │ 计算目标副本数
         │ desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
         │
         ▼
┌──────────────────┐
│  更新目标资源的   │
│  replicas 字段    │  (Deployment / RS / StatefulSet)
└──────────────────┘
```

### 8.1.2 时间窗口

- **指标采集间隔**：Metrics Server 默认每 15 秒采集一次。
- **HPA 检查间隔**：HPA Controller 默认每 15 秒检查一次。
- **缩容冷却期**：默认 5 分钟（300 秒），防止频繁的缩容和扩容。
- **扩容冷却期**：默认 3 分钟（180 秒），但实际扩容无冷却期（为了快速响应负载）。

### 8.1.3 为什么需要冷却期

HPA 通过冷却期（Stabilization Window）避免"震荡"（Thrashing）。如果没有冷却期，可能出现：

```
T+0s:  负载飙升，HPA 扩容 3 → 10
T+30s: 负载回落，HPA 缩容 10 → 3
T+60s: 负载再次飙升，HPA 扩容 3 → 10
...
```

频繁的扩缩容不仅浪费资源（每次扩容需要时间拉取镜像、启动应用），还可能影响服务稳定性。通过冷却期（特别是缩容冷却期较长），HPA 只在负载持续变化时才做出响应。

---

## 8.2 HPA YAML 详解

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2                     # 最小副本数
  maxReplicas: 10                    # 最大副本数
  behavior:                          # 伸缩行为控制（K8s 1.18+）
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容稳定窗口：5分钟
      policies:
      - type: Percent
        value: 10                     # 每分钟最多缩容 10% 的 Pod
        periodSeconds: 60
      - type: Pods
        value: 2                      # 每分钟最多缩容 2 个 Pod
        periodSeconds: 60
      selectPolicy: Min               # Min=最保守，Max=最激进，Disabled=禁止
    scaleUp:
      stabilizationWindowSeconds: 0   # 扩容稳定窗口：0秒（立即响应）
      policies:
      - type: Percent
        value: 100                    # 每分钟最多扩容 100% 的 Pod
        periodSeconds: 15
      - type: Pods
        value: 4                      # 每 15 秒最多扩容 4 个 Pod
        periodSeconds: 15
      selectPolicy: Max               # 取最大值（更激进）
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70       # 目标平均 CPU 使用率 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi          # 目标平均内存使用 512Mi
```

### 8.2.1 关键字段详解

**scaleTargetRef**

HPA 可以作用于任何拥有 Scale 子资源（`/scale`）的资源类型：
- `apps/v1` 的 Deployment、ReplicaSet、StatefulSet。
- 自定义资源（Custom Resource）只要实现了 Scale 子资源接口。

**minReplicas / maxReplicas**

- `minReplicas`：确保最少运行的副本数。即使负载极低，HPA 也不会缩容到低于此值。对于高可用应用，建议至少为 2。
- `maxReplicas`：最大的副本数限制。防止 HPA 无限扩容消耗集群资源，也起到"熔断"作用。

**behavior（Kubernetes 1.18+）**

`behavior` 字段提供了对扩缩容速率的精细控制：

- **scaleDown.stabilizationWindowSeconds**：缩容前查看的历史窗口（秒）。HPA 会收集该时间窗口内的所有缩容建议，取最大值。默认 300 秒（5 分钟）。

  ```
  T+0:   建议 replica=10
  T+60:  建议 replica=8
  T+120: 建议 replica=4
  T+180: 建议 replica=6
  稳定窗口=300s，取最大值 → replica=10（保守选择，暂不缩容）
  ```

- **scaleUp.stabilizationWindowSeconds**：扩容前的稳定窗口。默认 0 秒（不使用），意味着只要负载上升就立即扩容。

- **policies**：定义了扩缩容速率的多种策略。

  ```yaml
  policies:
  - type: Percent       # 按百分比
    value: 10            # 10%
    periodSeconds: 60    # 每分钟
  - type: Pods          # 按绝对数量
    value: 4             # 4 个
    periodSeconds: 15    # 每 15 秒
  selectPolicy: Max      # 取最激进值：max(10%/min, 4/15s) = 4/15s
  ```

  `selectPolicy` 取值：
  - `Max`：取变化量最大的策略（更激进）。
  - `Min`：取变化量最小的策略（更保守）。
  - `Disabled`：禁止该方向的伸缩。

---

## 8.3 支持的指标类型

### 8.3.1 Resource Metrics（资源指标）

```yaml
metrics:
- type: Resource
  resource:
    name: cpu                       # cpu 或 memory
    target:
      type: Utilization
      averageUtilization: 70       # 百分比
```

这是最简单的指标类型，直接使用 Metrics Server 提供的 Pod 资源使用率数据。支持的资源：
- `cpu`：CPU 使用核数 或 使用率百分比。
- `memory`：内存使用字节。

`target.type` 支持：
- `Utilization`：相对 Pod 的 `resources.requests` 的百分比。
- `AverageValue`：绝对值（如 `500m` 表示 0.5 核）。
- `Value`：原始值（较少用，一般与对象指标配合）。

### 8.3.2 Pod Metrics（自定义 Pod 指标）

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: 1000           # 每个 Pod 平均 1000 QPS
```

这需要部署 Prometheus Adapter 或类似的自定义指标适配器。指标来源于 Pod 级别的指标（通常通过 Prometheus 采集后暴露给 API Server）。

### 8.3.3 Object Metrics（对象指标）

```yaml
metrics:
- type: Object
  object:
    metric:
      name: requests-per-second
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: main-ingress
    target:
      type: Value
      value: 10k                   # Ingress 总 QPS 目标
```

与某个集群对象（而非 Pod）关联的指标。例如，监控整个 Ingress 的总 QPS。

### 8.3.4 External Metrics（外部指标）

```yaml
metrics:
- type: External
  external:
    metric:
      name: cloud_pubsub_queue_length
      selector:
        matchLabels:
          queue: worker-queue
    target:
      type: AverageValue
      averageValue: 30             # 队列长度目标
```

从集群外部的系统获取指标（如云服务的 Pub/Sub 队列长度、SQS 消息数、自定义的监控系统等）。

### 8.3.5 多指标支持

HPA 可以配置多个指标，Controller 会**分别计算每个指标对应的目标副本数，然后取其中最大值**。

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      averageUtilization: 70       # CPU 指标：建议 4 个副本
- type: Pods
  pods:
    metric:
      name: requests_per_second
    target:
      averageValue: 2000           # QPS 指标：建议 7 个副本

# HPA 最终使用 max(4, 7) = 7
```

这种"取最大值"的合并策略体现了"安全优先"原则：宁可多扩容一些，不可因为资源不足导致服务降级。

---

## 8.4 副本数计算公式

### 8.4.1 核心公式

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

其中 `ceil` 表示向上取整。

### 8.4.2 CPU 利用率计算示例

假设：
- Deployment 有 3 个 Pod，每个 Pod 的 `resources.requests.cpu: 200m`（0.2 核）
- 当前总 CPU 使用：Pod-A 使用 150m，Pod-B 使用 180m，Pod-C 使用 90m
- HPA 目标：CPU 利用率 50%

```
当前总使用量 = 150 + 180 + 90 = 420m
当前平均使用量 = 420m / 3 = 140m = 0.14 核

每个 Pod 的 request 为 200m = 0.2 核
当前利用率 = 0.14 / 0.2 = 70%

desiredReplicas = ceil[3 * (70 / 50)] = ceil[3 * 1.4] = ceil[4.2] = 5
```

HPA 将 replicas 调整为 5。

### 8.4.3 内存使用量计算示例

假设：
- HPA 目标：平均内存 256Mi
- 3 个 Pod，当前平均内存 512Mi

```
desiredReplicas = ceil[3 * (512 / 256)] = ceil[3 * 2] = 6
```

### 8.4.4 数值为 0 的特殊处理

如果当前指标值为 0 或接近 0：

```
desiredReplicas = ceil[3 * (0 / 50)] = 0
→ 但 0 < minReplicas(2)
→ 实际 replicas = 2 (保底)
```

如果期望指标值为 0（不合理设置）：

```
desiredReplicas = ceil[3 * (70 / 0)] = ∞
→ 截断到 maxReplicas(10)
→ 实际 replicas = 10
```

### 8.4.5 精度和容差

HPA 内置了一个 10% 的容差（Tolerance）来避免过度敏感的伸缩：

```
如果 |1 - (currentMetric / desiredMetric)| < 0.1
→ 不进行调整（认为当前状态已经足够接近目标）
```

这意味着如果当前 CPU 使用率在 45%~55%（目标是 50%），HPA 不会做任何调整。

---

## 8.5 HPA 配置最佳实践

### 8.5.1 为容器设置正确的 resources.requests

HPA 的 `Utilization` 类型指标依赖 `resources.requests` 计算百分比。如果 Pod 没有设置 `requests`，HPA 无法计算利用率（会使用默认值或报错）。

```yaml
# 错误：没有设置 resources
containers:
- name: app
  image: myapp:v1

# 正确：设置合理的 requests
containers:
- name: app
  image: myapp:v1
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

### 8.5.2 minReplicas 至少为 2

对于生产环境，`minReplicas` 建议至少设为 2，以确保：
- 单个 Pod 故障时仍有服务可用。
- 滚动更新时有备份 Pod 处理流量。
- 节点故障时 Pod 可以迁移。

`minReplicas=1` 的问题：
- 流量激增时，从 1 扩容到 N 需要时间，期间单 Pod 可能被压垮。
- 唯一的 Pod 故障 = 服务完全不可用。

### 8.5.3 合理设置 maxReplicas

`maxReplicas` 不是越大越好：
- 过大可能导致集群资源耗尽，影响其他服务。
- 某些后端系统（如数据库）的连接数有限，Pod 过多会导致数据库连接池耗尽。

设定 `maxReplicas` 时应考虑：
- 集群节点的总容量。
- 后端依赖的承受能力。
- 成本预算。

### 8.5.4 使用 behavior 控制缩容速率

默认缩容冷却期是 5 分钟，但某些场景下需要更精细的控制：

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 600  # 10 分钟的观察窗口
    policies:
    - type: Pods
      value: 1                        # 每分钟最多缩容 1 个 Pod
      periodSeconds: 60
```

### 8.5.5 CPU 还是 Memory 作为伸缩指标

| 指标 | 优点 | 缺点 |
|------|------|------|
| CPU | 与请求量直接相关，响应及时 | 不适用于 IO 密集型应用 |
| Memory | 反映内存压力 | 内存不随请求量线性变化，响应滞后 |

通常建议：
- **Web 应用/API**：以 CPU 为主指标，Memory 为辅助（如果语言有内存泄漏风险，可用 Memory 限制做"兜底"）。
- **缓存服务（Redis/Memcached）**：以 Memory 为主指标。
- **数据处理**：使用自定义指标（队列长度、请求延迟）。

### 8.5.6 自定义指标的延迟

Metrics Server 的数据有约 30~60 秒的延迟（采集间隔 + 聚合 + API 响应）。Prometheus Adapter 的延迟取决于 Prometheus 的采集间隔（通常 15~30 秒）加上 `rate()` 等查询的计算时间。

因此 HPA 不应该期望"实时"响应，对于极端突发流量（秒级），可能需要预留足够的缓冲容量。

### 8.5.7 与其他控制器的交互

- 不要对同时被 VPA（VerticalPodAutoscaler）管理的资源使用 HPA——两者会冲突。
- HPA 修改 Deployment 的 replicas 后，Deployment Controller 会执行滚动更新吗？不会——HPA 只修改 replicas 数字，不修改 template，所以不会触发滚动更新。

---

## 8.6 HPA 与 Cluster Autoscaler 的协同

### 8.6.1 协同的场景

HPA 负责增加 Pod 副本数，但当集群节点资源不足时，新增的 Pod 会处于 Pending 状态无法调度。此时需要 Cluster Autoscaler（CA）介入，自动增加节点。

```
负载上升
  → HPA 检测到 CPU 使用率 > 目标值
  → HPA 增加 Deployment replicas: 5 → 8
  → Scheduler 调度新增的 3 个 Pod
  → 发现所有 Node 资源已满
  → 新 Pod 状态：Pending (Unschedulable)
  → Cluster Autoscaler 检测到 Pending Pod
  → CA 向云平台请求新节点
  → 节点加入集群（2~3分钟）
  → Scheduler 将 Pending Pod 调度到新节点
  → 服务容量恢复
```

### 8.6.2 协同的关键配置

**1. PodDisruptionBudget (PDB)**

缩容时，CA 需要驱逐节点上的 Pod。如果 Pod 受 PDB 保护，CA 可能无法驱逐足够的 Pod 来腾空节点，导致缩容失败。

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 2    # 至少保持 2 个 Pod 可用
  selector:
    matchLabels:
      app: nginx
```

**2. Pod Priority**

关键组件（如 CoreDNS、kube-proxy）应该有高优先级，避免被 CA 驱逐。CA 会优先在低优先级 Pod 所在的节点中寻找可驱逐节点。

**3. 节点自动扩缩的延迟**

CA 从检测到 Pending Pod 到新节点就绪，通常需要 2~5 分钟（取决于云平台和镜像大小）。这个延迟需要在系统设计中考虑：
- 设置足够的缓冲容量（Headroom），不要等到资源用尽才扩容。
- 对于变化可预测的流量，使用定时伸缩（CronJob + 调整 replicas）做"预扩容"。

**4. 缩容时的协同**

```
负载下降
  → HPA 检测到 CPU 使用率 < 目标值
  → HPA 减少 Deployment replicas: 8 → 5
  → 释放 3 个 Pod 占用的资源
  → CA 检测到节点资源利用率低
  → CA 将空闲节点上的 Pod 迁移到其他节点
  → CA 从云平台删除空闲节点
```

CA 会在删除节点前等待至少 10 分钟（可配置），确保节点确实不再需要。这个等待时间避免了"HPA 缩容后立即再次扩容"导致的节点抖动。

### 8.6.3 注意事项

1. **DaemonSet Pod 的影响**：CA 在计算节点可缩容性时，会考虑 DaemonSet Pod——这些 Pod 无法迁移，如果节点上有 DaemonSet Pod 且有特定约束，节点可能无法缩容。
2. **本地存储 PV**：使用 Local PV 的 Pod 绑定到特定节点，CA 不能迁移这些 Pod，因此对应的节点也不能被缩容。
3. **自定义 Expander**：CA 支持自定义 Expander，可以根据价格、区域、实例类型等策略选择最优节点类型。

# 9. 其他控制器

## 9.1 TTL Controller

### 9.1.1 设计目标

TTL Controller（Time-To-Live Controller）是 kube-controller-manager 中的一个轻量级控制器，负责清理已完成或已过期的资源。它解决了一个实际问题：已完成 Job 不会自动删除，会不断消耗 etcd 存储空间，也可能影响 `kubectl get jobs` 的响应速度。

### 9.1.2 工作原理

TTL Controller 定期扫描集群中设置了 TTL 的资源：

```
1. 扫描所有 Job 对象
2. 如果 Job 的 .spec.ttlSecondsAfterFinished 已设置
3. 且 Job 已完成（.status.conditions 中包含 Complete 或 Failed）
4. 且 当前时间 - .status.completionTime > ttlSecondsAfterFinished
5. → 删除该 Job（级联删除关联的 Pod）
```

### 9.1.3 支持的资源类型

目前 TTL Controller 主要支持：
- **Job**：通过 `ttlSecondsAfterFinished` 设置。
- 未来可能扩展到其他资源（如 Completed Pod）。

### 9.1.4 配置示例

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cleanup-task
spec:
  ttlSecondsAfterFinished: 3600    # 完成后 1 小时自动删除
  template:
    spec:
      containers:
      - name: cleanup
        image: busybox
        command: ["sh", "-c", "echo Done"]
      restartPolicy: Never
```

### 9.1.5 设计考量

**为什么不在 Job Controller 中直接清理**：
解耦设计——每个控制器只做一件事。Job Controller 专注于管理 Job 和 Pod 的生命周期（创建、重试、完成），而 TTL Controller 专注于"到期清理"。这种分离使得：
1. 代码更简单、更容易维护。
2. 可以复用到其他资源类型。
3. 如果 TTL 逻辑有问题，不会影响 Job 的正常执行。

**扫描间隔**：TTL Controller 默认每 30 秒扫描一次。这意味着实际删除时间可能有最多 30 秒的延迟（对于 TTL 设得很短的场景需要注意）。

**不保证精确删除时间**：TTL Controller 的删除是"尽力而为"的。如果 Controller Manager 在删除时间附近宕机，删除会推迟到 CM 恢复后。

### 9.1.6 与 CronJob 历史保留的交互

如果 Job 由 CronJob 创建，有两个清理机制在同时工作：
1. CronJob 的 `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`。
2. Job 的 `ttlSecondsAfterFinished`。

两者独立工作，哪个先触发就先清理。例如：

```yaml
# CronJob 保留最近 3 个成功的 Job
successfulJobsHistoryLimit: 3

# 每个 Job 完成后 1 小时自动删除
ttlSecondsAfterFinished: 3600
```

如果任务每小时执行一次，`successfulJobsHistoryLimit: 3` 会保留最近 3 小时的任务记录。但如果 `ttlSecondsAfterFinished: 3600`，Job 会在完成后 1 小时被 TTL Controller 删除，实际上保留不到 3 个 Job。

**建议**：同时使用两者时，确保 TTL 大于 CronJob 的 schedule 间隔 × (historyLimit + 1)，否则 CronJob 的历史限制会被 TTL Controller 提前截断。

---

## 9.2 Garbage Collector（垃圾回收器）

### 9.2.1 设计目标

Garbage Collector（GC）是 Kubernetes 中负责级联删除的控制器。它基于 `OwnerReference` 机制，确保当父资源被删除时，其子资源也能被正确清理。

在没有 GC 的情况下：

```
删除 Deployment → Deployment 对象消失
但 ReplicaSet 仍然存在 → RS 管理的 Pod 仍然运行
→ 造成"僵尸"Pod
```

GC 通过级联删除解决此问题：

```
删除 Deployment → GC 自动删除所有子 RS → GC 自动删除所有子 Pod
```

### 9.2.2 OwnerReference 机制

GC 的核心数据结构是 `OwnerReference`，它定义在子资源的 `metadata.ownerReferences` 中：

```yaml
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: nginx-7d5c8f9b6c       # 父资源名称
    uid: 3ae8a679-7a09-44df       # 父资源的 UID（关键！）
    controller: true               # 该父资源是该 Pod 的管理控制器
    blockOwnerDeletion: true       # 阻止父资源被删除（如果此 Pod 不能被删除）
```

关键点：
- **UID 必需**：GC 通过 UID 查找父资源，UID 是全局唯一的，不会因为资源名称复用而产生错误关联。
- **controller: true**：标记该 OwnerReference 是"管理控制器"。一个资源可以有多个 OwnerReference（多个父资源），但只有一个应该标记 `controller: true`。
- **blockOwnerDeletion: true**：如果子资源不能被删除（例如因为 PDB 保护），则阻止父资源被删除。这保护了数据完整性。

### 9.2.3 级联删除模式

**Foreground Deletion（前台删除）**

```
1. 用户删除父资源
2. API Server 设置父资源的 deletionTimestamp 和 deletionGracePeriodSeconds
3. GC 首先删除所有 blockOwnerDeletion=true 的子资源
4. GC 然后删除所有其他子资源
5. 所有子资源被删除后，GC 删除父资源
```

前台模式下，父资源在子资源清理完毕前不会完全消失（通过 `foregroundDeletion` finalizer 阻止）。用户可以通过 `kubectl get` 看到父资源处于 `Terminating` 状态。

**Background Deletion（默认，后台删除）**

```
1. 用户删除父资源
2. API Server 立即删除父资源对象
3. GC 在后台异步删除所有子资源
```

用户可以立即观察不到父资源，但子资源在后台被逐步清理。

**Orphan Deletion（孤立删除）**

```
1. 用户使用 --cascade=orphan 删除父资源
2. API Server 移除子资源的 ownerReferences
3. 父资源被删除，子资源保留
```

适用场景：需要保留 Pod 但删除其上层控制器（如从 Deployment 迁移到直接管理）。

```bash
kubectl delete deployment nginx --cascade=orphan
# Deployment 被删除，但 RS 和 Pod 保留
```

### 9.2.4 垃圾回收的实现

GC 的扫描周期是动态的，基于集群中资源的变更频率。大致流程：

```
1. GC 维护一个"脏资源"集合（有变更的资源）
2. 定期扫描脏资源集合
3. 对每个脏资源：
   a. 查找其 OwnerReference 指向的所有父资源
   b. 检查父资源是否存在
   c. 如果父资源不存在 → 标记该资源为孤儿 Candidate
   d. 如果父资源正在删除（deletionTimestamp 非空）
      → 根据删除策略决定是否删除该子资源
4. 删除确认是孤儿且超过"孤儿容忍期"的资源
```

### 9.2.5 孤儿容忍期

GC 有一个"孤儿容忍期"（Orphan Tolerance Period），防止误删：
- 父资源刚被删除时，GC 不会立即删除子资源。
- 等待一段短暂时间（通常几秒），确认父资源不是暂时不可见（API Server 短暂不可用或网络分区）。
- 超过容忍期后仍然找不到父资源 → 确认是孤儿 → 执行清理。

### 9.2.6 GC 的限制和注意事项

1. **不处理跨命名空间引用**：OwnerReference 要求父子资源在同一 Namespace。跨 Namespace 的 GC 不支持。如果需要跨 Namespace 的级联操作，需要使用 Admission Webhook 或自定义 Controller 实现。
2. **循环引用**：如果 A 的 ownerReference 指向 B，B 的 ownerReference 指向 A，GC 不会进入死循环——它会检测到循环并中断（但设计上应该避免循环引用）。
3. **大批量删除的性能**：当 Namespace 被删除时，GC 需要级联删除该 Namespace 下的所有资源。在大型集群中（数万个资源），这需要数分钟时间。GC 内置了限速机制（Rate Limiting），避免对 API Server 造成过大压力。
4. **Finalizer 的影响**：如果子资源有 Finalizer（如 `kubernetes.io/pvc-protection`），GC 必须等待 Finalizer 被处理完毕才能删除。这可能导致级联删除被长时间阻塞。
5. **不要手动修改 OwnerReference**：手动修改 OwnerReference 可能导致 GC 行为异常。始终通过控制器管理 OwnerReference。

### 9.2.7 调试技巧

```bash
# 查看某个资源的 OwnerReference
kubectl get pod <pod-name> -o jsonpath='{.metadata.ownerReferences}'

# 查看资源的所有子资源（通过标签筛选）
kubectl get all -l app=nginx

# 查看 GC 是否有孤儿资源等待清理
kubectl get events --all-namespaces | grep -i garbage

# 监控级联删除进度
kubectl get deployment,rs,pod -l app=nginx -w
```

---

> 本文档是 Kubernetes 控制器体系的深入学习笔记。理解控制器的设计哲学和工作原理，是掌握 Kubernetes 核心架构的关键一步。在实际工程中，建议先通过 `kubectl` 命令行操作建立感性认识，再阅读 kube-controller-manager 源码中的控制器实现（如 `pkg/controller/deployment/`、`pkg/controller/job/` 等），最后尝试开发自定义 Controller 或 Operator 来巩固理解。
