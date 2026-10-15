# 1. 存储体系概述

## 1.1 容器存储的挑战

容器本身的设计哲学是"无状态、可丢弃"——容器文件系统是临时的，容器删除后，其写入的数据也随之消失。但现实世界中绝大多数应用都需要持久化存储：数据库需要持久化数据文件、消息队列需要持久化消息日志、内容管理系统需要持久化上传文件、AI/ML 工作负载需要持久化模型和数据集。

在 Kubernetes 出现之前，Docker 提供了两种持久化方案：
- **Volume**：由 Docker 管理，存储在宿主机 `/var/lib/docker/volumes/` 下
- **Bind Mount**：将宿主机任意目录挂载到容器中

这两种方式在单机场景下可用，但在分布式集群中存在致命缺陷：容器的调度不再与特定宿主机绑定。Pod 可能被调度到集群中任意节点，当 Pod 漂移到新节点时，它必须能够访问到同样的数据——而宿主机本地目录显然做不到这一点。

这就是 Kubernetes 引入 PV/PVC/StorageClass 体系的根本原因：**将存储的生命周期与 Pod 的生命周期解耦**，使得存储成为可独立管理、独立申请、跨节点访问的集群资源。

### 1.1.1 存储的核心矛盾

```
容器存储的三角困境：

        数据持久性
           /\
          /  \
         /    \
        /______\
   性能        可移植性

- 本地磁盘：高性能 + 持久，但 Pod 漂移后数据丢失（不可移植）
- NFS：高可移植 + 持久，但性能有限
- 云磁盘（EBS等）：高性能 + 持久，但受限于可用区（部分可移植）
- 分布式存储（Ceph等）：三者均衡，但运维复杂
```

Kubernetes 的存储体系通过多层次抽象来解决这个矛盾——让用户根据业务需求选择不同存储类型，而平台提供统一的使用接口。

## 1.2 PV/PVC/StorageClass 体系的设计哲学

Kubernetes 的存储体系借鉴了"计算资源"的管理模式：就像节点（Node）提供 CPU 和内存资源池、Pod 通过 `resources.requests` 申请资源一样，存储体系也分为"资源供给"和"资源消费"两层。

### 1.2.1 三种资源抽象

```
┌────────────────────────────────────────────────────┐
│                    用户 / 开发者                      │
│                                                      │
│    PersistentVolumeClaim (PVC)    ← 消费端            │
│    "我需要 10Gi 的 SSD 存储"                          │
│         │                                            │
│         ▼                                            │
│    PersistentVolume (PV)          ← 供给端            │
│    "这里有一个 20Gi 的 NFS 卷"                        │
│         │                                            │
│         ▼                                            │
│    StorageClass                   ← 动态供给          │
│    "需要存储时，自动调用 AWS EBS 创建"                 │
└────────────────────────────────────────────────────┘
```

**PV（PersistentVolume，持久卷）**是集群管理员提供的存储资源，类似于"节点"是计算资源。它独立于任何 Pod 存在，有自己的生命周期。PV 可以由管理员手动创建，也可以通过 StorageClass 动态创建。

**PVC（PersistentVolumeClaim，持久卷声明）**是用户对存储的请求，类似于 Pod 请求 CPU 和内存。用户不需要关心底层存储是什么（NFS？Ceph？云磁盘？），只需声明"我需要 10Gi 的读写存储"。Kubernetes 会自动寻找匹配的 PV 并绑定。

**StorageClass（存储类）**定义了存储的"类型"和"动态供给参数"。当用户创建 PVC 时，StorageClass 按照预定义的参数自动创建 PV，实现存储的按需供给。没有了 StorageClass，管理员需要手动创建每一个 PV——这在大型集群中不可行。

### 1.2.2 为什么要分离 PV 和 PVC

这种分离体现了 Kubernetes 的核心设计理念——**关注点分离**：

1. **管理员视角**：管理员关注的是"集群有多少存储可用"，负责配置存储后端（NFS 服务器、Ceph 集群、云磁盘类型）并创建 PV 或定义 StorageClass
2. **用户视角**：用户关注的是"我的应用需要多少存储"，只需创建 PVC 引用到 Pod 中
3. **两者解耦**：用户不需要知道存储的具体配置（NFS 服务器地址、云磁盘 ID），管理员不需要知道哪个应用使用了哪块存储

实际绑定过程：

```
PVC "需要10Gi RWO"     →     Kubernetes 控制循环     →     PV "20Gi NFS RWO"
                                                         PV "10Gi SSD RWO" ✓ 匹配！
                                                         PV "5Gi SSD RWO"
                                                         
匹配条件：
- PV 的 capacity.storage >= PVC 的 resources.requests.storage
- PV 的 accessModes 包含 PVC 请求的 accessModes
- PV 的 storageClassName == PVC 的 storageClassName
- PV 的 volumeMode == PVC 的 volumeMode
```

### 1.2.3 静态供给 vs 动态供给

**静态供给（Static Provisioning）**：管理员手动创建 PV 对象，Kubernetes 从 PV 池中匹配 PVC。适用于存储资源固定、变化不频繁的场景。

```yaml
# 管理员创建 PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

**动态供给（Dynamic Provisioning）**：管理员创建 StorageClass，用户创建 PVC 时，StorageClass 的 Provisioner 自动调用存储后端 API 创建实际的存储卷和对应的 PV 对象。

```yaml
# 管理员创建 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"

---
# 用户只需创建 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
# Kubernetes 自动创建 EBS 磁盘 + PV 对象，然后绑定到此 PVC
```

## 1.3 Volume 类型全景图

Kubernetes 支持数十种卷类型，可以按功能分为以下几类：

### 1.3.1 临时存储（与 Pod 同生命周期）

| 卷类型 | 数据生命周期 | 典型场景 |
|--------|-------------|---------|
| `emptyDir` | 与 Pod 相同，Pod 删除则数据清除 | Pod 内容器间共享临时数据、缓存 |
| `emptyDir.medium:Memory` | 与 Pod 相同（基于 tmpfs） | 高性能临时数据、敏感数据不落盘 |
| `downwardAPI` | 与 Pod 相同 | 将 Pod 元数据（名称、IP、标签）注入容器 |
| `configMap` | 独立于 Pod | 配置文件注入 |
| `secret` | 独立于 Pod | 敏感信息注入 |
| `projected` | 与 Pod 相同 | 将多个卷源投射到同一目录 |

### 1.3.2 持久存储（独立于 Pod 生命周期）

| 卷类型 | 典型实现 | 适用场景 |
|--------|---------|---------|
| `persistentVolumeClaim` | 通过 PVC 引用 PV | 通用的持久化存储（最常用） |
| `hostPath` | 节点本地目录 | DaemonSet 场景、日志采集、开发调试 |
| `local` | 节点本地持久卷（PV 类型） | 高性能本地存储（需配合拓扑调度） |
| `nfs` | NFS 服务器 | 多节点共享文件 |
| `csi` | 任意 CSI 驱动 | 通用接口，几乎所有现代存储 |

### 1.3.3 云厂商特定卷（通过 CSI 或 in-tree 驱动）

| 卷类型 | 云平台 | 特点 |
|--------|--------|------|
| `awsElasticBlockStore` | AWS | EBS 块存储，同可用区绑定 |
| `azureDisk` | Azure | Azure 托管磁盘 |
| `azureFile` | Azure | Azure Files SMB/CIFS |
| `gcePersistentDisk` | GCP | GCE 持久磁盘 |
| `cinder` | OpenStack | OpenStack Cinder 块存储 |

### 1.3.4 特殊用途卷

| 卷类型 | 说明 |
|--------|------|
| `iscsi` | iSCSI 目标挂载 |
| `fc` | 光纤通道卷 |
| `rbd` | Ceph RBD 块设备 |
| `cephfs` | CephFS 文件系统 |
| `glusterfs` | GlusterFS 卷 |
| `csi` | 通用 CSI 接口，可对接任意兼容存储 |

> **注意**：从 Kubernetes 1.23+ 开始，in-tree（内置）的云厂商卷驱动（如 `awsElasticBlockStore`、`azureDisk`）正在逐步迁移到 CSI 驱动。新集群推荐使用 CSI 驱动。

## 1.4 存储的核心概念关系图

```
┌──────────────────────────────────────────────────────────┐
│                      Pod                                 │
│  ┌──────────────────────────────────────────────────┐    │
│  │  containers:                                     │    │
│  │  - name: app                                     │    │
│  │    volumeMounts:                                 │    │
│  │    - name: data          ← 引用 Pod 级别的 Volume │    │
│  │      mountPath: /data                            │    │
│  └──────────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────┐    │
│  │  volumes:                                        │    │
│  │  - name: data                                    │    │
│  │    persistentVolumeClaim:  ← 引用 PVC            │    │
│  │      claimName: my-pvc                           │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│           PersistentVolumeClaim (my-pvc)                  │
│  spec:                                                    │
│    storageClassName: fast-ssd                             │
│    accessModes: [ReadWriteOnce]                           │
│    resources:                                             │
│      requests:                                            │
│        storage: 10Gi                                      │
└──────────────────────┬───────────────────────────────────┘
                       │ 绑定
                       ▼
┌──────────────────────────────────────────────────────────┐
│           PersistentVolume (pvc-abc123)                   │
│  spec:                                                    │
│    capacity:                                              │
│      storage: 10Gi                                        │
│    accessModes: [ReadWriteOnce]                           │
│    csi:                          ← 底层存储实现            │
│      driver: ebs.csi.aws.com                              │
│      volumeHandle: vol-xxxx                               │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│               Storage Backend                             │
│    AWS EBS / NFS / Ceph RBD / GlusterFS / ...            │
└──────────────────────────────────────────────────────────┘
```

# 2. Volume 类型详解

## 2.1 emptyDir

`emptyDir` 是最简单的卷类型。当 Pod 被创建时，kubelet 在节点上创建一个空目录，Pod 内所有容器都可以读写这个目录。Pod 删除时，目录及其中数据被永久删除。容器重启不影响 `emptyDir` 中的数据。

### 2.1.1 创建与生命周期

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: alpine
    command: ["sh", "-c", "echo 'data' > /cache/file.txt && sleep 3600"]
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  - name: reader
    image: alpine
    command: ["sh", "-c", "sleep 5 && cat /cache/file.txt && sleep 3600"]
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir:
      medium: ""           # 默认使用节点磁盘
      sizeLimit: 256Mi     # 限制卷的最大大小
```

### 2.1.2 存储介质选择

- `medium: ""`（默认）：使用节点磁盘，数据存储在 `/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/<volume-name>/`。适合一般场景，性能取决于节点磁盘类型。
- `medium: "Memory"`：使用 tmpfs（内存文件系统），数据存储在内存中。性能极高，但受限于节点内存。Pod 被删除或节点重启时数据丢失。适合缓存、临时密钥等高性能临时存储。

```yaml
volumes:
- name: fast-cache
  emptyDir:
    medium: "Memory"
    sizeLimit: 512Mi     # 限制使用的内存大小
```

设置 `sizeLimit` 后，如果实际使用超过限制，kubelet 会驱逐 Pod（标记为 `Evicted`）。这是通过节点级别的监控实现的，而非在文件系统层面强制实施。

### 2.1.3 底层实现

kubelet 创建 emptyDir 时：
1. 在节点文件系统上创建目录
2. 如果 `medium: Memory`，则挂载 tmpfs
3. 通过 bind mount 将目录挂载到容器中
4. 容器间共享同一目录，对文件的修改对对方立即可见

## 2.2 hostPath

`hostPath` 将节点上的文件或目录挂载到 Pod 中。这是最直接的方式，但也最危险——它将 Pod 与特定节点绑定，破坏了 Pod 的可移植性。

### 2.2.1 使用场景

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: log-collector
    image: fluentd:latest
    volumeMounts:
    - name: varlog
      mountPath: /host/var/log
      readOnly: true
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: varlog
    hostPath:
      path: /var/log
      type: Directory
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
```

### 2.2.2 type 参数（安全检查）

`hostPath` 的 `type` 字段用于在挂载前检查路径的类型，提供了一层安全保护：

| type | 含义 |
|------|------|
| `""`（空）| 不做任何检查（向后兼容） |
| `DirectoryOrCreate` | 必须是目录，不存在则创建（权限 0755，属主 kubelet） |
| `Directory` | 必须是已存在的目录 |
| `FileOrCreate` | 必须是文件，不存在则创建（权限 0644，属主 kubelet） |
| `File` | 必须是已存在的文件 |
| `Socket` | 必须是已存在的 Unix Socket |
| `CharDevice` | 必须是已存在的字符设备 |
| `BlockDevice` | 必须是已存在的块设备 |

### 2.2.3 风险与注意事项

1. **Pod 与节点绑定**：使用 hostPath 的 Pod 一旦被调度到某个节点，如果之后重建时该节点资源不足或不可用，Pod 可能无法在其他节点运行
2. **安全性**：hostPath 可以访问节点任意目录，具有特权级别的访问能力。必须严格限制 RBAC 权限
3. **资源泄露**：`DirectoryOrCreate` 和 `FileOrCreate` 创建的目录/文件在 Pod 删除后不会自动清理
4. **多副本问题**：多个副本 Pod 运行在不同节点时，每个节点有自己的 hostPath，数据不同步

## 2.3 local 持久卷

`local` PV 是 hostPath 的"正规军"——它将节点本地存储作为持久卷管理。与 hostPath 不同，local PV 有完整的 PV/PVC 生命周期管理，支持动态供给（通过 external-provisioner），并且与调度器紧密集成（通过拓扑绑定确保使用该 PV 的 Pod 被调度到正确的节点）。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:                      # 关键：指定 PV 所在的节点
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-1
```

`nodeAffinity` 是 local PV 的必须配置——它告诉调度器这个 PV 只能从特定节点访问。当 Pod 使用此 PV 时，调度器自动将 Pod 调度到对应节点。

local PV 的 `provisioner`（如 `kubernetes.io/no-provisioner`）通常用于静态供给。对于动态供给，可以使用 `rancher.io/local-path` 或 `openebs.io/local` 等外部 Provisioner。

## 2.4 downwardAPI

`downwardAPI` 将 Pod 自身的元数据和资源信息注入到容器中，使容器内的应用能够感知 Kubernetes 的环境信息。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-demo
  labels:
    app: demo
    version: v1
  annotations:
    build: "123"
spec:
  containers:
  - name: app
    image: alpine
    command: ["sh", "-c", "cat /etc/podinfo/labels && cat /etc/podinfo/annotations && sleep 3600"]
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "name"
        fieldRef:
          fieldPath: metadata.name
      - path: "namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "uid"
        fieldRef:
          fieldPath: metadata.uid
      - path: "cpu-limit"
        resourceFieldRef:
          containerName: app
          resource: limits.cpu
          divisor: 1m
      - path: "memory-request"
        resourceFieldRef:
          containerName: app
          resource: requests.memory
          divisor: 1Mi
```

支持的 `fieldRef.fieldPath`：
- `metadata.name`：Pod 名称
- `metadata.namespace`：Pod 所在命名空间
- `metadata.uid`：Pod 的 UID
- `metadata.labels`：所有标签（每行一个 `key=value`）
- `metadata.annotations`：所有注解（每行一个 `key=value`）
- `spec.nodeName`：Pod 被调度到的节点名
- `spec.serviceAccountName`：Pod 使用的 ServiceAccount
- `status.podIP`：Pod IP 地址
- `status.hostIP`：节点 IP 地址

## 2.5 projected（投射卷）

`projected` 卷可以将多个卷源（ConfigMap、Secret、downwardAPI、ServiceAccountToken）投射到同一个目录中，避免创建多个 Volume 和多个挂载点。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-demo
spec:
  containers:
  - name: app
    image: alpine
    command: ["sleep", "3600"]
    volumeMounts:
    - name: all-in-one
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - configMap:
          name: app-config
          items:
          - key: app.properties
            path: app/app.properties
      - secret:
          name: app-secret
          items:
          - key: password
            path: secrets/password
      - downwardAPI:
          items:
          - path: "pod-info/name"
            fieldRef:
              fieldPath: metadata.name
      - serviceAccountToken:       # 自动生成有效期可控的 Token
          path: token
          expirationSeconds: 3600
          audience: api
```

`serviceAccountToken` 是 `projected` 独有的功能——它允许生成一个具有特定有效期和受众的 ServiceAccount Token，而非使用默认的、无过期的 Token。这是 Kubernetes 1.21+ 推荐的 Token 注入方式。

# 3. 持久卷 (PV) 详解

## 3.1 PV 的完整 YAML 结构

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
  labels:
    type: nfs
    environment: production
  annotations:
    description: "NFS persistent volume"
spec:
  capacity:
    storage: 100Gi                    # 存储容量
  volumeMode: Filesystem              # Filesystem 或 Block
  accessModes:
    - ReadWriteMany                   # RWO / ROX / RWX
  persistentVolumeReclaimPolicy: Retain  # Retain / Delete / Recycle(已废弃)
  storageClassName: nfs-storage       # 关联的 StorageClass（空字符串表示禁用动态供给）
  mountOptions:                       # 挂载选项
    - hard
    - nfsvers=4.1
    - timeo=600
  nodeAffinity:                       # 节点亲和性（对 local PV 等必需）
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
  nfs:                                # 具体的后端存储配置
    server: 192.168.1.100
    path: "/exports/data"
    readOnly: false
```

## 3.2 PV 的状态机

PV 在其生命周期中经历以下状态：

```
创建 PV
   │
   ▼
 Available         ← PV 已创建，未被绑定
   │
   │ PVC 绑定成功
   ▼
 Bound             ← PV 已绑定到 PVC，进入使用状态
   │
   │ PVC 被删除
   ▼
 Released          ← PVC 被删除，但根据回收策略处理中
   │
   ├── Retain:  保留数据，PV 变为 Released，等待手动处理
   ├── Delete:  删除底层存储资源和 PV 对象
   └── Recycle: (已废弃) 清空数据后 PV 变回 Available
```

### 3.2.1 状态说明

- **Available**：PV 已创建且未被任何 PVC 绑定。可以被新的 PVC 使用
- **Bound**：PV 已被 PVC 绑定。绑定的 PV 不能被其他 PVC 使用（一对一绑定）
- **Released**：PVC 已被删除，但 PV 尚未被回收。根据回收策略进行下一步处理
- **Failed**：PV 的自动回收失败（如 Delete 策略下删除后端存储失败）

### 3.2.2 PV 与 PVC 的一对一绑定

PV 和 PVC 之间的绑定是一对一关系。一旦 PV 被绑定给某个 PVC，即使 PVC 实际只使用了 PV 容量的 10%，剩余 90% 的容量也无法被其他 PVC 使用。这要求管理员在规划 PV 大小时需要谨慎。

## 3.3 访问模式 (accessModes)

访问模式定义了 PV 可以被多少个节点以何种方式挂载：

| 访问模式 | 缩写 | 含义 | 典型存储后端 |
|----------|------|------|-------------|
| `ReadWriteOnce` | RWO | 卷可以被单个节点以读写模式挂载 | AWS EBS、GCE PD、本地存储 |
| `ReadOnlyMany` | ROX | 卷可以被多个节点以只读模式挂载 | NFS、CephFS |
| `ReadWriteMany` | RWX | 卷可以被多个节点以读写模式挂载 | NFS、CephFS、GlusterFS、Azure Files |
| `ReadWriteOncePod` | RWOP | 卷可以被单个 Pod 以读写模式挂载（v1.29+ GA） | 任何支持 RWO 的存储 |

**关键理解**：`ReadWriteOnce` 约束的是"节点"而非"Pod"。同一个节点上的多个 Pod 可以同时挂载一个 RWO 卷（只要存储后端支持）。`ReadWriteOncePod` 在 v1.29 中 GA，它确保了即使在同一个节点上，只有一个 Pod 能挂载该卷——这对于需要排他性访问的数据存储特别重要。

### 3.3.1 访问模式的选择

```
场景1：单实例数据库（MySQL on K8s）
  → RWO（只有主库写入，虽然可以部署多副本但只有一个活跃）

场景2：共享文件存储（WordPress 的 wp-content）
  → RWX（多个 WordPress Pod 需要读写同一份文件）

场景3：静态资源分发（只读配置文件）
  → ROX（多个 Pod 只读共享）

场景4：严格单实例（某些不支持并发访问的遗留系统）
  → RWOP（v1.29+，强制单 Pod 访问）
```

## 3.4 回收策略 (persistentVolumeReclaimPolicy)

回收策略定义了 PVC 被删除后，对应的 PV 和底层存储如何处理：

### 3.4.1 Retain（保留）

PV 变为 `Released` 状态，不删除底层存储，也不删除 PV 对象。管理员需要手动处理：清理数据、删除 PV、必要时手动删除底层存储。

```yaml
spec:
  persistentVolumeReclaimPolicy: Retain
```

处理流程：
1. 删除 PVC
2. PV 变为 `Released`
3. 管理员手动清理数据（如通过临时 Pod 挂载该 PV）
4. 管理员手动删除 PV 对象（此时 PV 变为 Available 可被重新绑定）
5. 如果是云端存储，管理员还需手动删除云磁盘

### 3.4.2 Delete（删除）

PV 和底层存储资源一起被删除。对于动态供给的卷（通过 StorageClass 创建的），这是默认策略。

```yaml
spec:
  persistentVolumeReclaimPolicy: Delete
```

处理流程：
1. 删除 PVC
2. Kubernetes 调用 Provisioner（如 AWS EBS CSI Driver）的 `DeleteVolume` 接口
3. Provisioner 删除云厂商的存储资源（AWS 删除 EBS 卷、GCP 删除 PD 等）
4. PV 对象被删除

> ⚠️ **生产警告**：使用 `Delete` 策略时，删除 PVC 即删除数据。务必确保有备份机制。推荐使用 Velero 等工具定期备份。

### 3.4.3 Recycle（已废弃）

该策略在 Kubernetes 1.12 中标记为废弃。它会执行 `rm -rf /` 来清空卷数据，然后 PV 变回 Available。这种方式在处理大容量卷时极慢且不可靠，已被 StorageClass 动态供给替代。

## 3.5 挂载选项 (mountOptions)

`mountOptions` 允许指定挂载时的文件系统选项。这些选项并非所有卷类型都支持——它们会被传递给底层的 mount 系统调用。

```yaml
spec:
  mountOptions:
    - hard           # NFS: 硬挂载，请求超时后不断重试
    - nfsvers=4.1    # NFS: 指定协议版本
    - rsize=1048576  # NFS: 读取缓冲区大小
    - wsize=1048576  # NFS: 写入缓冲区大小
    - timeo=600      # NFS: 超时时间（分秒）
    - noatime        # ext4/xfs: 不更新文件访问时间（性能优化）
    - nodiratime     # ext4/xfs: 不更新目录访问时间
```

# 4. 持久卷声明 (PVC) 详解

## 4.1 PVC 的完整 YAML 结构

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
  namespace: default
  labels:
    app: my-app
spec:
  storageClassName: fast-ssd         # 指定 StorageClass（"" 表示无 StorageClass）
  volumeMode: Filesystem             # Filesystem 或 Block
  accessModes:
    - ReadWriteOnce                  # 请求的访问模式
  resources:
    requests:
      storage: 10Gi                  # 请求的存储大小
  selector:                          # 可选：按标签筛选 PV
    matchLabels:
      type: ssd
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
  dataSource:                        # 可选：从现有卷克隆（v1.16+）
    kind: PersistentVolumeClaim
    name: source-claim
  dataSourceRef:                     # 可选：支持自定义资源的跨命名空间数据源
    kind: VolumeSnapshot
    name: my-snapshot
    apiGroup: snapshot.storage.k8s.io
```

## 4.2 绑定机制深入

PVC 的绑定过程由 PersistentVolumeController 执行：

```
1. 用户创建 PVC
       │
       ▼
2. PersistentVolumeController 检测到新的 PVC
       │
       ├── PVC 指定了 storageClassName？
       │     ├── 是 → 查找同名 StorageClass
       │     │      ├── 有 provisioner → 调用动态供给创建 PV
       │     │      └── 无 provisioner → 从现有 PV 池中匹配
       │     └── 否 → storageClassName 未设置（注意：与设为 "" 不同）
       │
       ├── 匹配条件：
       │     1. PV 的 capacity >= PVC 的 request
       │     2. PV 的 accessModes 包含 PVC 的 accessModes
       │     3. PV 的 storageClassName == PVC 的 storageClassName
       │     4. PV 的 volumeMode == PVC 的 volumeMode
       │     5. 如果 PVC 指定了 selector，PV 的 labels 必须匹配
       │
       ▼
3. 找到最小且足够的 PV → 绑定
       │
       ▼
4. PVC.status.phase = Bound
   PV.status.phase = Bound
```

### 4.2.1 storageClassName 的三种情况

| PVC 设置 | 行为 |
|----------|------|
| `storageClassName: "fast"` | 绑定到 StorageClass 为 "fast" 的 PV（或动态供给）|
| `storageClassName: ""` | 只绑定到没有 StorageClass 的 PV（明确拒绝动态供给）|
| 不设置 `storageClassName` | 如果集群有默认 StorageClass，则使用默认；否则只匹配无 StorageClass 的 PV |

### 4.2.2 绑定是排他的

PV 和 PVC 是一对一绑定。绑定后：
- PVC 不能被其他 PV 绑定
- PV 不能被其他 PVC 绑定（即使 PV 容量远超 PVC 请求）
- 绑定是不可逆的（删除 PVC 后 PV 进入 Released 而非直接回到 Available）

## 4.3 在 Pod 中使用 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-pvc
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
      subPath: nginx-data     # 可选：使用 PV 中的子目录
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-claim     # 引用 PVC 的名称
      readOnly: false
```

当 Pod 引用 PVC 时：
1. 调度器确保 Pod 被调度到能够访问该 PV 的节点（对有 `nodeAffinity` 的 local PV 尤其重要）
2. kubelet 调用 CSI 驱动（或 in-tree 驱动）将卷挂载到节点
3. kubelet 将卷 bind mount 到容器的 `mountPath`

## 4.4 PVC 保护机制

当 Pod 正在使用 PVC 时，Kubernetes 不允许删除该 PVC。这是通过 finalizer 实现的：

```yaml
metadata:
  finalizers:
  - kubernetes.io/pvc-protection
```

如果用户在 Pod 仍在运行时删除 PVC，PVC 会进入 `Terminating` 状态，但直到所有使用它的 Pod 被删除后，PVC 才会被真正删除。这个机制防止了"存储被强制卸载导致应用数据损坏"的情况。

# 5. StorageClass 与动态供给

## 5.1 StorageClass 的核心概念

StorageClass 是 Kubernetes 存储体系的"自动供给引擎"。管理员定义一个 StorageClass（描述存储的类型、参数、回收策略），之后用户只需创建 PVC 引用该 StorageClass，Kubernetes 会自动创建底层存储和 PV 对象。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com          # Provisioner 标识
parameters:                            # 传递给 Provisioner 的参数
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:..."
volumeBindingMode: WaitForFirstConsumer  # Immediate / WaitForFirstConsumer
allowVolumeExpansion: true            # 是否允许扩容
reclaimPolicy: Delete                 # 动态供给卷的默认回收策略
mountOptions:                         # 挂载选项
  - noatime
  - nodiratime
allowedTopologies:                    # 卷拓扑限制
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - us-east-1a
    - us-east-1b
```

## 5.2 Provisioner 详解

Provisioner 是 StorageClass 的核心——它是实际创建存储的后端驱动。Kubernetes 社区维护了多种 Provisioner：

| Provisioner | 存储后端 | 说明 |
|-------------|---------|------|
| `ebs.csi.aws.com` | AWS EBS | AWS EBS CSI Driver |
| `efs.csi.aws.com` | AWS EFS | AWS EFS CSI Driver |
| `pd.csi.storage.gke.io` | GCP PD | GCP Persistent Disk |
| `file.csi.azure.com` | Azure Files | Azure Files CSI |
| `disk.csi.azure.com` | Azure Disk | Azure Disk CSI |
| `nfs.csi.k8s.io` | NFS | NFS CSI Driver |
| `csi.trident.netapp.io` | NetApp | NetApp Trident |
| `rook-ceph.rbd.csi.ceph.com` | Ceph RBD | Rook Ceph |
| `local-path-provisioner` | 本地路径 | Rancher 的轻量本地存储 |
| `openebs.io/local` | 本地磁盘 | OpenEBS 本地卷 |
| `kubernetes.io/no-provisioner` | — | 不提供动态供给（仅静态 PV） |

### 5.2.1 动态供给的内部流程

```
1. 用户创建 PVC (storageClassName: fast-ssd, size: 10Gi)
       │
       ▼
2. PersistentVolumeController 检测到 PVC 需要动态供给
       │
       ▼
3. 查找 storageClassName 匹配的 StorageClass
       │
       ▼
4. 调用 StorageClass.provisioner 实现的 Provision 接口
       │
       ▼
5. Provisioner 调用后端 API 创建实际存储
   如：AWS EBS CSI 调用 AWS API 创建 10Gi gp3 卷
       │
       ▼
6. Provisioner 创建 PV 对象，填充：
    - spec.csi.driver
    - spec.csi.volumeHandle (后端存储的 ID)
    - spec.capacity.storage
    - spec.accessModes
    - metadata.annotations (包含 StorageClass 名称)
       │
       ▼
7. PersistentVolumeController 绑定 PV 到 PVC
       │
       ▼
8. Pod 可以使用该 PVC
```

## 5.3 卷绑定模式 (volumeBindingMode)

卷绑定模式决定了 PV 的供给和绑定的时机。这是 StorageClass 最重要的配置之一：

### 5.3.1 Immediate（默认）

PVC 创建后立刻创建 PV 并绑定。不考虑使用该 PVC 的 Pod 被调度到哪里。

```yaml
volumeBindingMode: Immediate
```

**问题**：如果 PV 有拓扑限制（如 AWS EBS 卷只能在特定可用区），但 Pod 还没被调度，可能出现 PV 在 us-east-1a 创建，但 Pod 被调度到 us-east-1b 无法挂载该卷。

### 5.3.2 WaitForFirstConsumer

PVC 创建后不立即创建 PV，而是等待第一个使用该 PVC 的 Pod 被调度后再创建 PV。这样 Provisioner 可以在 Pod 所在的可用区创建卷。

```yaml
volumeBindingMode: WaitForFirstConsumer
```

工作流程：
```
1. 用户创建 PVC (状态: Pending, 等待首个消费者)
2. 用户创建使用该 PVC 的 Pod
3. 调度器为 Pod 选择节点（考虑资源、亲和性等）
4. 调度器触发卷绑定（Volume Binder 组件）
5. Provisioner 在 Pod 所在节点的可用区创建卷
6. PV 创建并绑定到 PVC
7. Pod 调度完成，卷挂载
```

**优点**：
- 卷始终在 Pod 所在区域创建，避免跨可用区挂载
- 更好的拓扑感知（Topology-aware provisioning）

**缺点**：
- PVC 创建后不会立即显示为 Bound
- 增加了 Pod 启动延迟

云环境中（特别是使用 EBS、PD 等可用区绑定的块存储时），**强烈推荐使用 `WaitForFirstConsumer`**。

## 5.4 卷扩容 (Volume Expansion)

Kubernetes 1.11+ 支持在线扩容 PVC（需要底层存储驱动支持）。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true    # 必须为 true
```

扩容步骤：
```bash
# 1. 直接修改 PVC 的存储请求大小
kubectl patch pvc my-claim -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# 2. 查看扩容状态
kubectl describe pvc my-claim
# FilesystemResizePending: 表示正在等待文件系统扩容
# 部分存储可以做到在线扩容，无需重启 Pod
```

扩容的限制：
- 不支持缩容（只能增大不能减小）
- 部分存储类型需要重启 Pod 才能完成文件系统扩容（块设备扩容后需要 resize2fs/xfs_growfs）
- 并非所有 CSI 驱动支持在线扩容

## 5.5 默认 StorageClass

集群可以设置一个默认的 StorageClass，当 PVC 不指定 `storageClassName` 时自动使用：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
```

```bash
# 查看默认 StorageClass
kubectl get storageclass
# NAME            PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
# default (default)  ebs.csi.aws.com   Delete          Immediate           true                   30d
```

> **注意**：即使有默认 StorageClass，如果 PVC 显式设置 `storageClassName: ""`，会绑定到无 StorageClass 的静态 PV，不会触发动态供给。

# 6. CSI (Container Storage Interface)

## 6.1 为什么需要 CSI

在 CSI 出现之前，Kubernetes 的存储驱动是"in-tree"（内置）的——每个存储后端的代码都放在 Kubernetes 核心代码库中。这带来了一系列问题：

1. **Kubernetes 代码库膨胀**：每个新存储后端的添加都需要修改 Kubernetes 核心代码
2. **发布周期绑定**：存储驱动的 bug 修复必须跟随 Kubernetes 的发布周期（每季度），无法独立发布
3. **稳定性风险**：第三方存储驱动的代码质量参差不齐，但都被编译进核心组件
4. **无法扩展**：只有社区接受的存储驱动才能被合并，小众存储无法原生支持

CSI 通过标准化的 gRPC 接口将存储驱动的实现完全移出 Kubernetes 核心代码库，实现了：
- 存储厂商可以独立开发、发布、维护自己的驱动
- 驱动可以有自己的发布周期（快速修复 bug）
- 驱动的稳定性问题不会影响 Kubernetes 核心

## 6.2 CSI 架构

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes 核心组件                      │
│                                                             │
│  ┌────────────┐       ┌──────────────┐                     │
│  │kubelet     │       │kube-apiserver│                     │
│  │            │       │              │                     │
│  │ ┌────────┐ │       │ ┌──────────┐ │                     │
│  │ │CSI     │ │       │ │CSI       │ │                     │
│  │ │Registrar││       │ │Provisioner││                     │
│  │ └────────┘ │       │ └──────────┘ │                     │
│  └─────┬──────┘       └──────┬───────┘                     │
│        │                     │                              │
│        │ gRPC (Unix Socket)  │ gRPC (TCP/Unix Socket)     │
│        ▼                     ▼                              │
│  ┌──────────────────────────────────────────┐              │
│  │          CSI Driver (Sidecar)              │              │
│  │                                            │              │
│  │  ┌───────────────┐  ┌──────────────────┐  │              │
│  │  │ CSI Identity   │  │ CSI Controller   │  │              │
│  │  │  (GetPluginInfo│  │  (CreateVolume   │  │              │
│  │  │   GetPluginCap)│  │   DeleteVolume   │  │              │
│  │  │               │  │   ControllerPub  │  │              │
│  │  │               │  │   ControllerUnpub│  │              │
│  │  │               │  │   CreateSnapshot │  │              │
│  │  │               │  │   ExpandVolume)  │  │              │
│  │  └───────────────┘  └──────────────────┘  │              │
│  │                                            │              │
│  │  ┌───────────────┐                        │              │
│  │  │ CSI Node       │                        │              │
│  │  │  (NodeStage   │                        │              │
│  │  │   NodePublish │                        │              │
│  │  │   NodeUnpub   │                        │              │
│  │  │   NodeUnstage │                        │              │
│  │  │   NodeGetVol  │                        │              │
│  │  │   Stats)      │                        │              │
│  │  └───────────────┘                        │              │
│  └────────────────────┬─────────────────────┘              │
│                       │                                      │
└───────────────────────┼──────────────────────────────────────┘
                        │ 厂商特定协议 (REST/SDK)
                        ▼
              ┌────────────────────┐
              │  Storage Backend    │
              │ (AWS EBS/ NFS /    │
              │  Ceph / NAS / ...) │
              └────────────────────┘
```

## 6.3 CSI 的三个服务接口

### 6.3.1 Identity Service（身份服务）

```protobuf
service Identity {
    rpc GetPluginInfo(GetPluginInfoRequest) returns (GetPluginInfoResponse);
    rpc GetPluginCapabilities(GetPluginCapabilitiesRequest) returns (GetPluginCapabilitiesResponse);
    rpc Probe(ProbeRequest) returns (ProbeResponse);
}
```

返回驱动的名称、版本、支持的能力（如是否支持 Controller 服务、是否支持扩容）。

### 6.3.2 Controller Service（控制器服务）

```protobuf
service Controller {
    rpc CreateVolume(CreateVolumeRequest) returns (CreateVolumeResponse);
    rpc DeleteVolume(DeleteVolumeRequest) returns (DeleteVolumeResponse);
    rpc ControllerPublishVolume(ControllerPublishVolumeRequest) returns (ControllerPublishVolumeResponse);
    rpc ControllerUnpublishVolume(ControllerUnpublishVolumeRequest) returns (ControllerUnpublishVolumeResponse);
    rpc ValidateVolumeCapabilities(ValidateVolumeCapabilitiesRequest) returns (ValidateVolumeCapabilitiesResponse);
    rpc ListVolumes(ListVolumesRequest) returns (ListVolumesResponse);
    rpc GetCapacity(GetCapacityRequest) returns (GetCapacityResponse);
    rpc ControllerGetCapabilities(ControllerGetCapabilitiesRequest) returns (ControllerGetCapabilitiesResponse);
    rpc CreateSnapshot(CreateSnapshotRequest) returns (CreateSnapshotResponse);
    rpc DeleteSnapshot(DeleteSnapshotRequest) returns (DeleteSnapshotResponse);
    rpc ListSnapshots(ListSnapshotsRequest) returns (ListSnapshotsResponse);
    rpc ControllerExpandVolume(ControllerExpandVolumeRequest) returns (ControllerExpandVolumeResponse);
}
```

Controller 服务处理卷的生命周期管理：创建、删除、快照、扩容、挂载到节点。这些接口由 external-provisioner、external-attacher、external-snapshotter 等 Sidecar 容器调用。

### 6.3.3 Node Service（节点服务）

```protobuf
service Node {
    rpc NodeStageVolume(NodeStageVolumeRequest) returns (NodeStageVolumeResponse);
    rpc NodeUnstageVolume(NodeUnstageVolumeRequest) returns (NodeUnstageVolumeResponse);
    rpc NodePublishVolume(NodePublishVolumeRequest) returns (NodePublishVolumeResponse);
    rpc NodeUnpublishVolume(NodeUnpublishVolumeRequest) returns (NodeUnpublishVolumeResponse);
    rpc NodeGetVolumeStats(NodeGetVolumeStatsRequest) returns (NodeGetVolumeStatsResponse);
    rpc NodeExpandVolume(NodeExpandVolumeRequest) returns (NodeExpandVolumeResponse);
    rpc NodeGetCapabilities(NodeGetCapabilitiesRequest) returns (NodeGetCapabilitiesResponse);
    rpc NodeGetInfo(NodeGetInfoRequest) returns (NodeGetInfoResponse);
}
```

Node 服务处理卷在节点上的挂载/卸载。由 kubelet 直接调用。

### 6.3.4 Stage vs Publish 的区别

对于块设备（如 EBS），卷挂载分两个阶段：

1. **Stage**（`NodeStageVolume`）：将块设备挂载到一个全局目录（`/var/lib/kubelet/plugins/kubernetes.io/csi/.../globalmount`）。这一步每个节点只做一次，多个使用同一卷的 Pod 共享这个全局挂载。

2. **Publish**（`NodePublishVolume`）：将全局挂载点 bind mount 到 Pod 的特定目录。每个 Pod 有自己的挂载点。

对于文件系统类存储（如 NFS），Stage 和 Publish 可以在同一操作中完成。

## 6.4 CSI Sidecar 容器

CSI 驱动通常不是单个容器，而是由多个 Sidecar 容器组成一个 Pod：

| Sidecar | 功能 | 调用 CSI 接口 |
|---------|------|-------------|
| `external-provisioner` | 监听 PVC 创建事件，调用 CreateVolume | `Controller.CreateVolume` |
| `external-attacher` | 监听 VolumeAttachment 对象，执行挂载/卸载 | `Controller.ControllerPublishVolume` |
| `external-snapshotter` | 监听 VolumeSnapshot 对象，管理快照 | `Controller.CreateSnapshot` |
| `external-resizer` | 监听 PVC 扩容请求，执行扩容 | `Controller.ControllerExpandVolume` |
| `node-driver-registrar` | 向 kubelet 注册 CSI 驱动 | 通过 CSI Socket 注册 |
| `livenessprobe` | 监控 CSI 驱动的健康状态 | `Identity.Probe` |

```yaml
# 示例：AWS EBS CSI Driver 的 Controller Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ebs-csi-controller
spec:
  template:
    spec:
      containers:
      - name: ebs-plugin
        image: public.ecr.aws/ebs-csi-driver/aws-ebs-csi-driver:latest
        args:
        - --endpoint=$(CSI_ENDPOINT)
      - name: csi-provisioner
        image: registry.k8s.io/sig-storage/csi-provisioner:v4.0.0
      - name: csi-attacher
        image: registry.k8s.io/sig-storage/csi-attacher:v4.3.0
      - name: csi-snapshotter
        image: registry.k8s.io/sig-storage/csi-snapshotter:v7.0.0
      - name: csi-resizer
        image: registry.k8s.io/sig-storage/csi-resizer:v1.10.0
      - name: liveness-probe
        image: registry.k8s.io/sig-storage/livenessprobe:v2.12.0
```

# 7. 存储快照与克隆

## 7.1 VolumeSnapshot

VolumeSnapshot 是 Kubernetes 1.17+ 引入的存储快照功能，允许对 PVC 进行快照备份。

### 7.1.1 核心 CRD

```yaml
# VolumeSnapshotClass - 快照类（类似 StorageClass）
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete         # Delete: 删除快照时删除后端快照 / Retain: 保留
parameters:
  type: snapshot              # 取决于 CSI 驱动
---
# VolumeSnapshot - 快照请求
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: my-pvc   # 要对哪个 PVC 做快照
```

### 7.1.2 从快照恢复

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## 7.2 卷克隆

直接从现有 PVC 克隆新卷：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: fast-ssd
  dataSource:
    name: source-pvc
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

克隆的限制：
- 目标 PVC 的存储大小不能小于源 PVC
- 必须在同一命名空间内
- 源和目标必须使用相同的 StorageClass（或至少存储后端相同）
- 底层 CSI 驱动必须支持克隆（`CLONE_VOLUME` capability）

# 8. 生产实践

## 8.1 各场景存储选型指南

| 应用类型 | 推荐存储 | 原因 |
|---------|---------|------|
| **数据库 (MySQL/PostgreSQL)** | RWO 块存储（EBS/PD/Azure Disk），local PV | 低延迟、高 IOPS，单实例写入 |
| **消息队列 (Kafka)** | local PV 或 RWO 高性能块存储 | 高吞吐、低延迟 |
| **缓存 (Redis)** | local PV 或 Memory emptyDir | 低延迟，数据可丢失 |
| **共享文件 (WordPress)** | RWX（NFS/CephFS/EFS/Azure Files） | 多 Pod 读写同一份文件 |
| **AI/ML 训练数据** | RWX 高性能存储（CephFS/Lustre）或对象存储（S3） | 大数据量、多节点读取 |
| **日志归档** | RWX 廉价存储（NFS/Azure Files） | 大容量、低频访问 |
| **CI/CD 工作空间** | emptyDir | 临时数据，用完即丢 |
| **配置文件** | ConfigMap 挂载（非 PVC） | 小文件、热更新需求 |

## 8.2 常见问题诊断

### 8.2.1 PVC 一直 Pending

```bash
# 查看 PVC 事件
kubectl describe pvc my-pvc

# 常见原因：
# 1. No provisioner for StorageClass - StorageClass 没有 provisioner
# 2. No available PV - 静态 PV 池中没有匹配的 PV
# 3. Provisioner 创建卷失败 - 检查 provisioner 日志
# 4. WaitForFirstConsumer - PVC 等待 Pod 使用（正常现象）
```

### 8.2.2 Pod 因卷无法启动

```bash
# Pod 状态 Pending，描述中有卷相关事件
kubectl describe pod my-pod

# 常见原因：
# 1. PVC not found - PVC 名称错误或不存在
# 2. PV node affinity conflict - Pod 被调度到无法访问该 PV 的节点
# 3. volume node affinity conflict - 多可用区场景下的拓扑冲突
# 4. Mount failure - kubelet 挂载卷失败（启动 kubelet 日志排查）
```

### 8.2.3 卷扩容失败

```bash
# 检查 StorageClass 是否允许扩容
kubectl get sc <class-name> -o yaml | grep allowVolumeExpansion

# 检查 CSI 驱动是否支持扩容
kubectl get csidriver -o yaml | grep -A5 volume-expansion
```

## 8.3 备份策略

```bash
# etcd 备份（Kubernetes 元数据）
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# PV 数据备份（通过 Velero）
velero backup create full-backup --include-namespaces production

# PV 数据备份（通过 VolumeSnapshot）
# 对重要 PVC 创建定期快照，配合 CronJob 使用
```

## 8.4 安全建议

1. **避免 hostPath**：除非必要（如 DaemonSet 的日志采集），不使用 hostPath
2. **PVC 保护**：启用 PVC 保护机制（默认开启），防止误删
3. **回收策略**：生产环境 PVC 建议使用 `Retain` 策略，手动验证后再删除底层存储
4. **加密**：云存储配置加密参数（如 `encrypted: "true"`），CSI 驱动都应支持
5. **RBAC**：限制 PVC 的创建和删除权限，防止恶意或意外操作
