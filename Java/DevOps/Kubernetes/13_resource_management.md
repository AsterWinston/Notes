# 1.资源管理概述

## 1.1 为什么需要资源管理

Kubernetes 集群的资源（CPU、内存、存储、GPU 等）是有限的。在多租户环境中，如果没有有效的资源管理机制，会出现以下严重问题：

**"吵闹的邻居"问题**：一个 Pod 无节制地消耗资源，导致同一节点上的其他 Pod 无法正常工作。例如，一个存在内存泄漏的应用会逐渐吞噬节点的全部内存，最终触发 Linux OOM Killer 杀死其他无辜的容器。

**资源碎片化**：Pod 被随意调度导致节点上留下大量不可用的小块资源。例如，一个 4C8G 的节点剩余 1.5C3G 的资源，无法再调度一个需要 2C4G 的 Pod，这 1.5C3G 就被浪费了。

**成本失控**：没有资源限制，团队可以无限制地部署服务，集群资源很快耗尽，导致需要不断扩容节点，云成本失控。

**调度混乱**：调度器不知道 Pod 需要多少资源，只能随机分配节点，导致某些节点负载极高而另一些节点几乎空闲。

Kubernetes 通过三层资源管理机制解决上述问题：

```
┌──────────────────────────────────────────────────────┐
│  层1: 容器级资源控制                                   │
│  resources.requests / resources.limits                │
│  "这个容器需要多少资源？上限是多少？"                     │
├──────────────────────────────────────────────────────┤
│  层2: 命名空间级资源控制                                 │
│  ResourceQuota / LimitRange                          │
│  "这个 Namespace 总共能用多少资源？"                     │
├──────────────────────────────────────────────────────┤
│  层3: 节点级资源控制                                    │
│  QoS / Eviction / OOM Score                         │
│  "资源不够时先杀谁？"                                   │
└──────────────────────────────────────────────────────┘
```

# 2.容器级资源管理：Requests 和 Limits

## 2.1 核心概念

```yaml
containers:
- name: app
  image: myapp:latest
  resources:
    requests:           # 调度保证（下限）
      cpu: "250m"
      memory: "256Mi"
    limits:             # 运行上限
      cpu: "500m"
      memory: "512Mi"
```

### Requests（请求）

- **意义**：容器启动所需的最小资源保证。调度器根据所有容器的 requests 总和来决定 Pod 应该调度到哪个节点
- **CPU requests**：决定了容器在 CPU 时间片分配中的权重。即使节点有剩余的 CPU 空闲，低 requests 的容器在高竞争时会获得较少的 CPU 时间
- **Memory requests**：决定了调度时的内存预留。节点可分配内存 ≥ 所有 Pod 的 memory requests 总和

### Limits（限制）

- **意义**：容器允许使用的最大资源上限
- **CPU limits**：容器超过 limits 时，CPU 被限流（throttle），容器**不会被杀死**
- **Memory limits**：容器超过 limits 时，容器被 **OOMKilled**（进程收到 SIGKILL，退出码 137）

### CPU 与内存的本质区别

```
CPU（可压缩资源）：
  0% ───────── request(250m) ───────── limit(500m) ──────────
       [保证]              [可超用]              [被限流]
  
  行为：超限后被 throttling，进程变慢但不被杀

Memory（不可压缩资源）：
  0% ───────── request(256Mi) ─────────── limit(512Mi) ──────
       [保证]                  [可超用]        [OOMKilled]
  
  行为：超限后被 OOM Killer 杀死，容器重启
```

## 2.2 资源单位详解

### CPU 单位

| 表示方式 | 含义 |
|---------|------|
| `1` | 1 个 vCPU / 1 个超线程 |
| `500m` | 0.5 个 CPU 核心（500 millicores） |
| `100m` | 0.1 个 CPU 核心（100 millicores） |
| `2500m` | 2.5 个 CPU 核心 |

1 个 vCPU 在不同云平台对应：
- AWS: 1 vCPU = 1 个 Intel Xeon 超线程
- GCP: 1 vCPU = 1 个超线程
- Azure: 1 vCPU = 1 个超线程

### 内存单位

| 单位 | 字节数 | 使用示例 |
|------|--------|---------|
| `Ki` | 1,024 | `256Ki` |
| `Mi` | 1,048,576 | `256Mi` |
| `Gi` | 1,073,741,824 | `4Gi` |
| `Ti` | 1,099,511,627,776 | `1Ti` |
| `k` | 1,000 | （十进制，不推荐） |
| `M` | 1,000,000 | （十进制，不推荐） |
| `G` | 1,000,000,000 | （十进制，不推荐） |

始终使用二进制单位（Ki/Mi/Gi）以避免混淆。

### 巨页（Huge Pages）和扩展资源

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
    hugepages-2Mi: "100Mi"        # 巨页内存
    nvidia.com/gpu: "1"           # GPU 扩展资源
    example.com/fpga: "2"         # FPGA 扩展资源
  limits:
    hugepages-2Mi: "100Mi"
    nvidia.com/gpu: "1"
```

## 2.3 Requests 和 Limits 的关系与最佳实践

### 四种配置策略

| 配置策略 | CPU | Memory | 使用场景 |
|---------|-----|--------|---------|
| **不设任何值** | 不限 | 不限 | ❌ 不推荐，可能被 OOM Kill 或过度占用资源 |
| **设 Limits 不设 Requests** | Requests=Limits | Requests=Limits | 简单场景，资源保证高但不灵活 |
| **设 Requests 不设 Limits** | 可以超用 | 可以超用 | 允许突发流量，但可能被驱逐 |
| **Requests < Limits** | 保证 + 可超用 | 保证 + 超用危险 | 平衡性能和利用率（推荐） |

### 生产环境推荐配置

```yaml
# 关键服务：Requests = Limits，保证稳定性
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "1"
    memory: "2Gi"

# 一般服务：Requests < Limits，提高资源利用率
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"
```

**为什么 CPU 的 Limits 可以宽松但 Memory 的 Limits 不能？**
- CPU 超限只是变慢（throttling），不影响可用性
- Memory 超限会直接 OOM Kill，导致容器重启和服务中断

## 2.4 资源使用的底层机制

### CPU 的 CFS 调度器实现

Linux 使用 CFS（Completely Fair Scheduler）的 cgroup 子系统实现 CPU 限制：

```
容器 CPU request = 250m：
  cpu.shares = 250  → 权重为 250，决定竞争时的 CPU 分配比例

容器 CPU limit = 500m：
  cpu.cfs_period_us = 100000  (100ms)
  cpu.cfs_quota_us = 50000    (50ms)
  → 每 100ms 周期内最多使用 50ms 的 CPU 时间
```

### Memory 的 OOM 机制

```
容器 Memory limit = 512Mi：
  memory.limit_in_bytes = 536870912  (512Mi)

当容器内存使用超过 512Mi 时：
  1. 内核 OOM Killer 选择该 cgroup 中的进程
  2. 发送 SIGKILL 信号
  3. 容器退出码 137 (128 + 9)
  4. kubelet 根据 restartPolicy 重启容器
```

### 为什么内存是"不可压缩"的

CPU 时间可以"排队"——进程等待 CPU 时间片即可，系统不会崩溃。但内存不能"排队"——进程请求的内存必须立即分配。当节点物理内存耗尽时，必须杀死进程来释放内存。

# 3.Quality of Service (QoS) 等级

## 3.1 三种 QoS 等级

Kubernetes 根据 Pod 中容器的 resources 配置自动为 Pod 分配 QoS 等级：

### Guaranteed（有保证）

**条件**：Pod 中所有容器的 `memory requests == memory limits` 且 `cpu requests == cpu limits`

```yaml
containers:
- name: app
  resources:
    limits:
      cpu: "1"
      memory: "1Gi"
    requests:
      cpu: "1"
      memory: "1Gi"
```

这类 Pod 具有最高优先级，在资源紧张时**最不可能被驱逐**。

### Burstable（可突发）

**条件**：Pod 不符合 Guaranteed 的条件，但至少有一个容器设置了 `memory requests` 或 `cpu requests`

```yaml
# 设置了 requests 但 limits 不同（或未设）
containers:
- name: app
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2"
      memory: "2Gi"

# 只设置了 requests
containers:
- name: app
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
```

这类 Pod 具有一定保证，但在资源紧张时可能被驱逐。

### BestEffort（尽力而为）

**条件**：Pod 中没有任何容器设置 `resources.requests` 或 `resources.limits`

```yaml
containers:
- name: app
  image: nginx
  # 没有 resources 字段
```

这类 Pod 没有资源保证，在资源紧张时**最先被驱逐**。

## 3.2 QoS 等级的影响

### 对 OOM Score 的影响

```bash
# OOM Killer 根据 oom_score_adj 决定杀死哪个进程
# QoS Guaranteed:     oom_score_adj = -998  (几乎不会被杀)
# QoS Burstable:      oom_score_adj = min(max(2, 1000 - 1000 * (内存使用量/request)), 999)
# QoS BestEffort:     oom_score_adj = 1000  (最先被杀)
```

OOM Score 越高，越容易被 OOM Killer 选中。BestEffort Pod 的 OOM Score 最高，总是第一个被杀死。

### 对 Eviction 的影响

kubelet 在节点资源紧张时驱逐 Pod，按照以下优先级：

```
1. BestEffort 的 Pod（最先驱逐）
2. 超出 requests 的 Burstable Pod（其次驱逐）
3. 未超出 requests 的 Burstable Pod
4. Guaranteed 的 Pod（最后驱逐）
```

### 对调度的影响

- Guaranteed Pod 的资源被节点 100% 预留
- Burstable Pod 只有 requests 部分被预留
- BestEffort Pod 不被预留任何资源

## 3.3 查看 Pod 的 QoS 等级

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
# 输出: Guaranteed / Burstable / BestEffort

kubectl describe pod <pod-name> | grep "QoS Class"
# QoS Class: Burstable
```

# 4.ResourceQuota（资源配额）

## 4.1 概念与作用

ResourceQuota 是 Namespace 级别的资源总量限制。它防止单个 Namespace 过度消耗集群资源，实现多租户之间的公平分配。

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-a
spec:
  hard:
    # 计算资源总量限制
    requests.cpu: "10"              # 所有 Pod 的 CPU requests 总和 ≤ 10
    requests.memory: "20Gi"         # 所有 Pod 的 Memory requests 总和 ≤ 20Gi
    limits.cpu: "20"                # 所有 Pod 的 CPU limits 总和 ≤ 20
    limits.memory: "40Gi"           # 所有 Pod 的 Memory limits 总和 ≤ 40Gi

    # 存储资源限制
    requests.storage: "500Gi"       # 所有 PVC 请求的存储总量
    persistentvolumeclaims: "20"    # PVC 数量上限

    # 数量限制
    pods: "50"                      # Pod 数量上限
    services: "20"                  # Service 数量上限
    configmaps: "50"                # ConfigMap 数量上限
    secrets: "50"                   # Secret 数量上限
    services.nodeports: "5"         # NodePort Service 数量上限
    count/deployments.apps: "20"    # Deployment 数量上限
    count/jobs.batch: "10"          # Job 数量上限
```

## 4.2 配额的作用机制

ResourceQuota 通过**准入控制器**（Admission Controller）实现：

```
用户创建 Pod
     │
     ▼
API Server 准入控制器检查 ResourceQuota
     │
     ├── 当前 Pod 创建后是否超出配额？
     │    ├── 未超出 → 允许创建
     │    └── 超出 → 拒绝请求，返回 403 Forbidden
     │         错误信息：exceeded quota: compute-quota,
     │         requested: requests.memory=2Gi,
     │         used: requests.memory=18Gi,
     │         limited: requests.memory=20Gi
```

**重要提示**：ResourceQuota 强制要求 Pod 必须设置 resources（requests 和/或 limits）。如果创建 Pod 时没有设置，而 ResourceQuota 中定义了相关限制，API Server 会直接拒绝创建。

## 4.3 多 ResourceQuota 和 PriorityClass 配额

### 多配额叠加

一个 Namespace 可以有多个 ResourceQuota，它们的效果叠加：

```yaml
# 配额一：计算资源
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "10"
    limits.memory: "40Gi"
---
# 配额二：对象数量
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    pods: "50"
    services: "20"
```

### 基于 PriorityClass 的配额

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-priority-quota
spec:
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values:
      - high-priority
      - medium-priority
  hard:
    pods: "10"
```

四种可用范围：
- `Terminating`：`spec.activeDeadlineSeconds >= 0` 的 Pod
- `NotTerminating`：`spec.activeDeadlineSeconds` 为 nil 的 Pod
- `BestEffort`：QoS 为 BestEffort 的 Pod
- `NotBestEffort`：QoS 非 BestEffort 的 Pod

# 5.LimitRange（范围限制）

## 5.1 概念与作用

ResourceQuota 限制的是 Namespace 的**总量**，LimitRange 限制的是单个 Pod/容器的**上下限和默认值**。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: team-a
spec:
  limits:
  # 容器级别的限制
  - type: Container
    max:
      cpu: "2"                    # 单个容器最大 CPU limit
      memory: "4Gi"              # 单个容器最大 Memory limit
    min:
      cpu: "100m"                 # 单个容器最小 CPU request
      memory: "128Mi"            # 单个容器最小 Memory request
    default:                      # 默认 Limits（未指定时应用）
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:               # 默认 Requests（未指定时应用）
      cpu: "250m"
      memory: "256Mi"
    maxLimitRequestRatio:         # limits/requests 的最大比例
      cpu: "4"                    # limits.cpu ≤ requests.cpu × 4
      memory: "2"                 # limits.memory ≤ requests.memory × 2

  # Pod 级别的限制
  - type: Pod
    max:
      cpu: "4"                    # 单个 Pod 所有容器 CPU limit 之和的上限
      memory: "8Gi"

  # PVC 级别的限制
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"            # 单个 PVC 最大容量
    min:
      storage: "1Gi"              # 单个 PVC 最小容量
```

## 5.2 LimitRange 的作用机制

当一个不符合 LimitRange 要求的 Pod 被创建时，API Server 的准入控制器会：

1. **验证**：检查 resources 是否在 min/max 范围内，不在则拒绝
2. **填充默认值**：如果 Pod 没有指定 requests/limits，使用 `default` 和 `defaultRequest` 填充
3. **检查比例**：验证 limits/requests 的比例是否超过 `maxLimitRequestRatio`

```bash
# 创建 Pod 时未指定 resources，LimitRange 自动填充默认值
# 然后 ResourceQuota 检查是否超出配额
# 因此 LimitRange 和 ResourceQuota 通常配合使用
```

# 6.Pod 驱逐 (Eviction)

## 6.1 kubelet 驱逐机制

kubelet 在节点资源紧张时主动驱逐 Pod，以保护节点不被完全耗尽。

### 触发条件

kubelet 监控节点的以下指标，超过阈值时触发驱逐：

| 驱逐信号 | 条件 | 默认硬驱逐阈值 |
|---------|------|-------------|
| `memory.available` | 节点可用内存过低 | `< 100Mi` |
| `nodefs.available` | 节点文件系统可用空间过低 | `< 10%` |
| `nodefs.inodesFree` | 节点 inode 剩余过低 | `< 5%` |
| `imagefs.available` | 镜像文件系统可用空间过低 | `< 15%` |
| `imagefs.inodesFree` | 镜像文件系统 inode 剩余过低 | — |

### 软驱逐 vs 硬驱逐

```yaml
# kubelet 配置
evictionHard:                        # 硬驱逐：超过阈值立即驱逐
  memory.available: "100Mi"
  nodefs.available: "10%"

evictionSoft:                        # 软驱逐：超过阈值后等待 grace period 再驱逐
  memory.available: "200Mi"
  nodefs.available: "20%"
evictionSoftGracePeriod:             # 软驱逐宽限期
  memory.available: "1m30s"
  nodefs.available: "2m"
```

### 驱逐顺序

kubelet 按照以下优先级选择要驱逐的 Pod：

```
1. 按 QoS 等级
   BestEffort → Burstable(超出request) → Burstable(未超出) → Guaranteed

2. 同 QoS 等级内按资源使用量
   内存使用相对于 request 的超出比例最大的 Pod 先被驱逐

3. 综合考虑
   优先级 + Pod Age + 是否有 PDB 保护
```

## 6.2 节点压力驱逐流程

```
节点内存紧张 (available < 100Mi)
     │
     ▼
Node Condition: MemoryPressure=True
     │ 添加 taint: node.kubernetes.io/memory-pressure:NoSchedule
     │
     ▼
kubelet Eviction Manager 开始驱逐 Pod
     │
     ├── 按优先级排序驱逐候选 Pod
     │   1. BestEffort Pod
     │   2. 超用最多的 Burstable Pod
     │   3. Guaranteed Pod（极少）
     │
     ├── 遵守 PDB (PodDisruptionBudget)
     │   如果驱逐会违反 PDB，跳过该 Pod
     │
     └── 调用 Eviction API 删除 Pod
         Pod 收到 SIGTERM → 优雅终止 → 在其他节点重建
```

## 6.3 优先级抢占驱逐

当高优先级 Pod 无法调度时，会触发抢占驱逐：

```
高优先级 Pod Pending
     │
     ▼
Scheduler 尝试抢占
     │ 模拟驱逐低优先级 Pod
     │ 选择影响最小的节点
     │
     ▼
Eviction API 驱逐低优先级 Pod
     │
     ▼
高优先级 Pod 被调度
```

# 7.生产实践

## 7.1 资源规划的黄金法则

```
法则 1: 所有生产 Pod 必须设置 resources
  - 至少设置 requests
  - 推荐同时设置 requests 和 limits

法则 2: Memory requests = Memory limits（或接近）
  - 内存不可压缩，超配风险高
  - 如果追求利用率，最多 limits = requests × 2

法则 3: CPU requests < CPU limits
  - CPU 可压缩，允许利用空闲资源
  - 典型比例：limits = requests × 2~4

法则 4: 每个 Namespace 配置 ResourceQuota 和 LimitRange
  - 防止意外无限扩展
  - 确保所有 Pod 都有资源声明

法则 5: 监控实际使用量并定期调整
  - 使用 VPA 推荐器获取建议值
  - 不要设置过低导致 OOM 或过高导致浪费
```

## 7.2 资源使用监控

```bash
# 查看节点资源使用
kubectl top nodes

# 查看 Pod 资源使用
kubectl top pods -n production

# 查看 Namespace 中 ResourceQuota 使用情况
kubectl describe resourcequota compute-quota -n team-a

# 查看 LimitRange 配置
kubectl get limitrange -n team-a
kubectl describe limitrange resource-limits -n team-a
```

## 7.3 常见问题诊断

### "exceeded quota" 错误

```bash
# Pod 创建失败，错误信息包含 "exceeded quota"
# 检查当前使用量
kubectl describe resourcequota -n <namespace>

# 解决方案：
# 1. 清理不需要的 Pod/PVC/Service
# 2. 增大 ResourceQuota 的 hard 限制
# 3. 优化现有 Pod 的 requests（减少不必要的预留）
```

### Pod 频繁 OOMKilled

```bash
kubectl describe pod <pod-name> | grep OOMKilled
# State: Terminated, Reason: OOMKilled, Exit Code: 137

# 解决方案：
# 1. 增大 memory limits
# 2. 检查应用是否有内存泄漏
# 3. 增加 JVM/应用的内存限制参数（-Xmx）
```

### 节点资源不足导致 Pending

```bash
kubectl describe pod <pod-name> | grep Events
# Events: 0/5 nodes are available: 5 Insufficient cpu.

# 解决方案：
# 1. 减少 Pod 的 requests
# 2. 缩容其他低优服务
# 3. 添加新节点
# 4. 启用 Cluster Autoscaler
```
