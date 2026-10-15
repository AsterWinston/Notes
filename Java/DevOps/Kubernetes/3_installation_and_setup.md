# 1. 集群部署方式概览

Kubernetes 集群的部署是运维工作中的第一道门槛。由于 Kubernetes 控制面组件众多、依赖关系复杂，社区发展出了多种部署工具和方式以适配不同场景。理解每种方式的底层原理和设计取舍，是做好生产级集群管理的前提。

## 1.1 部署方式分类

根据自动化程度和控制粒度，Kubernetes 部署方式可分为以下几类：

### 1.1.1 本地开发/测试环境

**minikube**：由 Kubernetes 官方维护的单节点集群工具。其核心理念是"在本地笔记本上跑一个完整的 K8s"。minikube 通过创建一台虚拟机或容器，在其中运行所有控制面组件和一个 node。它支持多种 driver（VirtualBox、Hyper-V、Docker、Podman 等），会将 `~/.kube/config` 自动配置好。minikube 内置了大量 addon（如 dashboard、ingress、metrics-server），方便开发者快速体验 K8s 功能。其缺点在于无法模拟多节点网络、调度策略等生产场景。

**kind (Kubernetes in Docker)**：由 Kubernetes 社区维护，用 Docker 容器来模拟 K8s 节点。每个"节点"本质上是一个运行着 systemd 和 K8s 组件的 Docker 容器。kind 通过 CNI 插件在容器之间建立网络，可实现多控制面 + 多 worker 的拓扑。kind 的一个关键设计是将 K8s 组件的二进制和镜像预置到节点镜像中，避免了安装时的网络下载问题，特别适合 CI/CD 流水线中的集成测试。

**k3d**：在 Docker 中运行 k3s 的工具，类似于 kind 但底层使用 k3s，启动速度更快，适合需要快速部署的场景。

### 1.1.2 轻量级/边缘部署

**k3s**：Rancher 开发的轻量级 Kubernetes，通过三个关键设计实现极致的轻量化：(1) 将 kube-apiserver、kubelet、kube-proxy 等组件打包为单一二进制文件，约 100MB；(2) 用 SQLite 替代 etcd 作为默认存储后端（也支持 etcd），大幅减少内存和磁盘 I/O；(3) 内置 containerd 和 flannel，无需额外安装容器运行时和网络插件。k3s 还移除了云厂商相关的 alpha 功能和遗留插件，使二进制体积进一步缩小。适用于边缘计算、IoT、ARM 设备等资源受限场景。

**k0s**：Mirantis 出品的零依赖 K8s 发行版，同样支持单二进制部署，但采用 etcd 作为默认存储。k0s 的特点是完全不依赖宿主机上的任何 K8s 组件，所有东西都打包在一个二进制中，通过 `k0s worker` 命令即可加入集群。

**MicroK8s**：Canonical 推出的 snap 包形式的 K8s，默认包含 istio、registry、knative 等插件。它通过 snap 的沙箱机制实现自动更新和隔离，深度集成 Ubuntu 生态。缺点是强依赖 snapd，无法在非 Ubuntu 系统上直接使用。

### 1.1.3 标准生产部署

**kubeadm**：Kubernetes 官方推荐的集群引导工具，是 SIG Cluster Lifecycle 的核心成果。kubeadm 的设计哲学是"最小可行引导"（Minimal Viable Bootstrap）：它只负责集群初始化和节点加入，不关心基础设施（机器配置、网络、负载均衡等）。这种职责分离使得 kubeadm 可以适配各种基础设施环境。kubeadm 通过多阶段（phase）的设计，允许用户选择性地执行某个步骤，为高度定制化提供了可能。

**二进制部署**：完全手动地从官方发布页下载各组件二进制文件（kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy），自行创建 systemd 服务单元、生成证书、配置 RBAC。这种方式的学习成本最高，但能让运维人员对每个组件的位置、配置、参数做到完全掌控。在生产环境中，二进制部署的集群排查起来也最为容易，因为没有任何工具的"魔法"来屏蔽细节。

### 1.1.4 云托管

**托管控平面（Managed Control Plane）**：AWS EKS、Azure AKS、Google GKE 等云厂商提供的服务。用户只需管理 Worker Node，控制面组件由云厂商负责运维。云厂商通常按小时对控制面收费，并提供自动升级、自动扩缩、与 IAM/VPC 深度集成等能力。GKE 是公认最成熟的托管 K8s 服务，因为它由 K8s 的核心贡献者 Google 运营。

**Serverless 容器平台**：AWS Fargate、GKE Autopilot、阿里云 ECI 等更进一步，连 Worker Node 的运维都被抽象掉，用户只需定义 Pod 规格，平台自动提供底层算力。

## 1.2 各方式适用场景与对比

| 部署方式 | 适用场景 | 控制面成本 | 运维复杂度 | 生产就绪 |
|---------|---------|-----------|-----------|---------|
| minikube | 本地开发、学习 | 无 | 极低 | 否 |
| kind | CI/CD 测试、本地开发 | 无 | 极低 | 否 |
| k3s | 边缘计算、ARM、开发环境 | 无 | 低 | 是（轻量场景） |
| k0s | 零依赖环境、CI/CD | 无 | 低 | 是（轻量场景） |
| kubeadm | 自建生产集群 | 无 | 中 | 是 |
| 二进制 | 深度定制生产集群 | 无 | 高 | 是 |
| 云托管 | 企业标准化生产 | 按量付费 | 低 | 是 |

选择决策树：如果是本地开发学习，选 minikube 或 kind；如果是边缘/IoT 部署，选 k3s；如果要自建生产集群且需要社区标准，选 kubeadm；如果要深度定制或排查方便，选二进制；如果团队运维能力有限但预算充足，选云托管。

## 1.3 环境准备要求

### 1.3.1 操作系统

Kubernetes 官方支持 Linux 操作系统。推荐的发行版包括：
- **Ubuntu 22.04/24.04 LTS**：社区支持最广泛，大多数教程和文档以此为基础
- **CentOS/RHEL 8/9**：企业级环境常用，但注意 CentOS 8 已 EOL，建议使用 Rocky Linux 或 AlmaLinux
- **Debian 11/12**：稳定版，适合保守的运维策略
- **Flatcar Container Linux**：专为容器设计的不可变操作系统

Windows Server 自 2019 年起支持作为 Worker Node，但控制面必须运行在 Linux 上。

### 1.3.2 内核版本

Kubernetes 官方建议 Linux 内核版本 ≥ 3.10，但推荐使用 4.18+ 以获得更好的 cgroup v2 支持和 eBPF 能力。

内核配置要点：
- `overlay` 文件系统模块：containerd/docker 使用 overlayfs 作为容器镜像存储
- `br_netfilter` 模块：允许 iptables 看到桥接流量，是 kube-proxy 和 CNI 正常工作的前提
- `nf_conntrack` 哈希表大小：高并发场景下 conntrack 表容易满，建议调整为 1048576 以上
- `ip_forward`：必须启用 IP 转发，否则 Pod 无法跨节点通信
- `cgroup v2`：Kubernetes 1.25+ 默认推荐 cgroup v2，1.27+ 的 kubelet 默认假设使用 cgroup v2

### 1.3.3 硬件资源

| 角色 | CPU（最小） | 内存（最小） | 磁盘（最小） | 推荐 |
|------|------------|------------|------------|------|
| 控制面 | 2 核 | 2 GB | 20 GB | 4核 8GB SSD |
| Worker | 2 核 | 2 GB | 20 GB | 8核 16GB SSD |

etcd 对磁盘延迟特别敏感，官方强烈建议使用 SSD。etcd 写入延迟超过 10ms 时就可能出现 leader election 超时。对于生产环境，建议 etcd 使用独立磁盘或至少与其他组件隔离 I/O。

### 1.3.4 网络要求

集群内网络连通性要求：
- 所有节点之间可以互相通信（无 NAT）
- 每个节点上的 kubelet 可以访问控制面的 API Server（通常是 6443 端口）
- 控制面节点之间需要通信（2379-2380 端口用于 etcd，10250-10255 用于 kubelet）
- Pod CIDR 和 Service CIDR 不能与宿主机网络、现有数据中心网络重叠

网络的"完全可达"是 Kubernetes 的最基本假设。如果节点之间由于防火墙规则存在单向连通或部分端口不通，会导致 Pod 间通信异常、kubelet 状态不同步、Service 负载均衡失败等各种难以诊断的问题。

### 1.3.5 容器运行时

从 Kubernetes 1.24 起，dockershim 已被移除，不再原生支持 Docker 作为容器运行时。推荐的容器运行时包括：

**containerd**：CNCF 毕业项目，是从 Docker 中独立出来的容器运行时，也是目前 Kubernetes 社区使用最广泛的运行时。containerd 通过 CRI（Container Runtime Interface）与 kubelet 通信，支持 OCI 标准镜像。

**CRI-O**：由 Red Hat 主导的 CRI 实现，专为 Kubernetes 设计。它不提供 Docker CLI 兼容层，更加精简。OpenShift 的默认运行时就是 CRI-O。

容器运行时配置要点：
- 驱动类型（cgroup driver）：必须与 kubelet 的 cgroup driver 保持一致（均为 systemd 或均为 cgroupfs）。不一致会导致节点资源统计错误和 Pod 驱逐策略异常。
- pause 镜像配置：kubelet 在每个 Pod 启动时会先启动一个 `pause` 容器作为 NetNS 的持有者。containerd 需要配置正确的 pause 镜像地址。
- 镜像仓库认证：私有镜像仓库（如 Harbor）需要在 containerd 配置中添加认证信息。

# 2. kubeadm部署详解

kubeadm 是目前 Kubernetes 官方唯一推荐的"通用生产环境引导工具"。理解 kubeadm 的每一步背后做了什么，是掌握生产级 K8s 部署的关键。

## 2.1 节点规划与前置准备

### 2.1.1 关闭 swap

Kubernetes 设计上不支持 swap，原因是 QoS 模型和 Pod 驱逐策略依赖于内存资源的精确统计。如果开启了 swap，当内存压力大时系统会将部分内存页换出到磁盘，kubelet 无法感知到这部分"被交换出去"的内存，导致 OOM 打分不准、eviction 逻辑失效。Kubernetes 1.22 后已经正式将 swap 支持加入 alpha（NodeSwap 特性门控），但目前还不推荐在生产中启用。

```bash
# 临时关闭
swapoff -a

# 永久关闭：注释掉 /etc/fstab 中的 swap 行
sed -i '/swap/s/^/#/' /etc/fstab
```

### 2.1.2 关闭防火墙

如果基础设施层面已有硬件防火墙或安全组，可以直接关闭节点上的 firewalld/ufw：

```bash
# CentOS/RHEL
systemctl stop firewalld
systemctl disable firewalld

# Ubuntu
ufw disable
```

如果必须保留主机防火墙，需要开放以下端口：

| 端口 | 协议 | 用途 |
|------|------|------|
| 6443/tcp | Kubernetes API Server |
| 2379-2380/tcp | etcd 客户端与对等通信 |
| 10250/tcp | kubelet API |
| 10251/tcp | kube-scheduler |
| 10252/tcp | kube-controller-manager |
| 30000-32767/tcp | NodePort 服务 |
| 8285/udp | flannel VXLAN (若使用) |
| 179/tcp | Calico BGP (若使用) |
| 8472/udp | Canal/Flannel VXLAN |

### 2.1.3 关闭 SELinux

SELinux 的强制访问控制策略可能阻止容器运行时对宿主文件系统的访问。虽然 Kubernetes 从 1.26 开始支持 SELinux 以增强容器安全（SecurityContext 中的 seLinuxOptions），但在大多数部署中仍选择关闭：

```bash
# 临时关闭
setenforce 0

# 永久关闭，编辑 /etc/selinux/config
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 2.1.4 配置内核参数

以下内核参数对 Kubernetes 网络和容器功能至关重要：

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
# 启用 IP 转发，允许 Pod 跨节点通信
net.ipv4.ip_forward = 1

# 允许 iptables 处理桥接流量，CNI 插件依赖此项
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# 不将桥接网络流量发送给 iptables 的 FORWARD 链，提高性能 (K8s 1.25+ 可选)
net.bridge.bridge-nf-call-arptables = 1

# 增大 conntrack 表容量，防止高并发时连接跟踪表溢出
net.netfilter.nf_conntrack_max = 1048576

# 增大 listen backlog，适合高并发 API Server
net.core.somaxconn = 32768

# 扩大本地端口范围，防止 TimeWait 状态下端口耗尽
net.ipv4.ip_local_port_range = 1024 65535

# 启用 TCP 连接复用
net.ipv4.tcp_tw_reuse = 1

# 减少 TCP KeepAlive 探测频率
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10

# 允许 overcommit memory (RAM)，容器运行时需要
vm.overcommit_memory = 1

# 允许足够的 PID 数量，防止 fork 炸弹
kernel.pid_max = 4194304

# 增大 inotify 监控数量 (开发环境尤其需要)
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192
EOF

sysctl --system
```

`br_netfilter` 模块加载：

```bash
modprobe br_netfilter
echo "br_netfilter" > /etc/modules-load.d/br_netfilter.conf
```

**为什么需要 `bridge-nf-call-iptables`？** Kubernetes 的 kube-proxy 通过 iptables/IPVS 规则实现 Service 的负载均衡。当两个 Pod 位于同一宿主机时，它们之间通过 linux bridge 通信。默认情况下，linux 内核不会将桥接流量传递给 iptables，这意味着同节点 Pod 间通信时 iptables 规则不生效，导致 Service ClusterIP 无法正确 DNAT。启用此参数后，内核将桥接流量也交给 iptables 处理。

### 2.1.5 安装 containerd

containerd 是当前推荐的容器运行时。以下以 Ubuntu 22.04 为例：

```bash
# 安装依赖
apt-get update
apt-get install -y ca-certificates curl

# 添加 Docker 官方 GPG 密钥
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# 添加源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 containerd
apt-get update
apt-get install -y containerd.io
```

containerd 配置：

```bash
# 生成默认配置
containerd config default > /etc/containerd/config.toml

# 关键修改：将 SystemdCgroup 设置为 true
# 原因：kubelet 默认使用 systemd cgroup driver，containerd 必须保持一致
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 配置 pause 镜像（如果使用私有仓库或国内镜像源）
# sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9"

# 重启服务
systemctl restart containerd
systemctl enable containerd
```

安装并配置 crictl（用于调试容器运行时）：

```bash
VERSION="v1.30.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz \
  --output crictl.tar.gz
tar zxvf crictl.tar.gz -C /usr/local/bin
rm -f crictl.tar.gz

# 配置 crictl 连接到 containerd
cat <<EOF > /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

## 2.2 安装kubeadm/kubelet/kubectl

### 2.2.1 Ubuntu/Debian

```bash
# 添加 Kubernetes APT 仓库
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg

# 添加社区 GPG 密钥 (K8s 官方)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 添加源
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list

# 安装指定版本（生产环境必须锁定版本）
apt-get update
VERSION=1.30.0-1.1
apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION

# 锁定版本，防止意外升级
apt-mark hold kubelet kubeadm kubectl
```

### 2.2.2 CentOS/RHEL

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF

VERSION=1.30.0
yum install -y kubelet-$VERSION kubeadm-$VERSION kubectl-$VERSION --disableexcludes=kubernetes
systemctl enable kubelet
```

**注意**：安装完成后 kubelet 会进入 crashloop 状态（`systemctl status kubelet` 显示 exited），这是正常的——kubelet 在未加入任何集群前会不断尝试启动并失败，直到 `kubeadm init/join` 完成后才能正常运行。

## 2.3 kubeadm init流程深度解析

`kubeadm init` 是整个集群部署的核心命令。它通过一系列 "phase" 按顺序完成集群初始化：

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.1.10 \
  --control-plane-endpoint=cluster-api.example.com:6443 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.30.0 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --upload-certs
```

### 2.3.1 preflight 阶段

`kubeadm init phase preflight` 是第一步，执行环境检查：

1. **操作系统检查**：验证 Linux 内核版本、操作系统兼容性
2. **容器运行时检查**：通过 CRI socket 检测容器运行时是否正常运行
3. **网络检查**：验证网络接口、IP 转发、bridge-nf-call-iptables 内核参数
4. **端口检查**：确认 API Server 端口（6443）、etcd 端口（2379-2380）未被占用
5. **组件版本检查**：验证 kubeadm、kubelet 版本兼容性
6. **cgroup 驱动检查**：确认容器运行时与 kubelet 使用相同的 cgroup 驱动
7. **swap 检查**：确认 swap 已关闭

preflight 检查的设计理念是"早失败"（fail fast）：在花费大量时间部署组件之前，确保环境满足最低要求。如果某些检查不通过，可以使用 `--ignore-preflight-errors` 跳过，但生产环境不推荐。

### 2.3.2 kubelet 启动

`kubeadm init phase kubelet-start` 在节点上启动 kubelet。具体操作：
1. 创建 `/var/lib/kubelet/config.yaml` 配置文件，指定集群 DNS 域、clusterDNS IP 等参数
2. 生成 `/var/lib/kubelet/kubeadm-flags.env` 环境变量文件
3. 将 kubelet 注册为 systemd 服务并启动

此时 kubelet 以 standalone 模式运行，尚未注册到 API Server。它会尝试连接 API Server（`--kubeconfig` 指向的文件尚不存在），因此会不断重试。

### 2.3.3 证书生成

`kubeadm init phase certs` 生成所有必要的 PKI 证书和私钥。证书结构如下：

| 证书文件 | 用途 | 签发 CA |
|---------|------|---------|
| ca.crt / ca.key | 集群根 CA，自签名 | 自身 |
| apiserver.crt / apiserver.key | API Server 服务端证书 | ca |
| apiserver-kubelet-client.crt | API Server 访问 kubelet 的客户端证书 | ca |
| front-proxy-ca.crt | 前端代理 CA | 自身 |
| front-proxy-client.crt | 前端代理客户端证书 | front-proxy-ca |
| etcd/ca.crt | etcd CA | 自身 |
| etcd/server.crt | etcd 服务端证书 | etcd-ca |
| etcd/peer.crt | etcd 节点间通信证书 | etcd-ca |
| etcd/healthcheck-client.crt | etcd 健康检查客户端证书 | etcd-ca |
| sa.pub / sa.key | ServiceAccount 令牌签名密钥对 | - |
| admin.conf | 集群管理员 kubeconfig（内嵌证书） | - |

这些证书默认存放在 `/etc/kubernetes/pki/`。证书的有效期默认为 1 年（CA 为 10 年）。生产环境中，证书过期会导致 API Server 无法启动、节点无法注册等问题。可以使用 `kubeadm certs check-expiration` 查看证书有效期，使用 `kubeadm certs renew` 续签。

**为什么需要 front-proxy-ca？** 在 Kubernetes 中，API Aggregation 层允许多个 API Server 聚合到主 API Server（如 metrics-server 通过 `metrics.k8s.io` API 注册）。前端代理证书用于验证这些聚合 API Server 的身份，防止伪造的 API 端点。

### 2.3.4 生成静态Pod配置

`kubeadm init phase control-plane` 阶段负责生成控制面组件的 Static Pod 配置。

**为什么是 Static Pod 而不是 systemd 服务？** 这是 kubeadm 的一个关键设计决策。Static Pod 是由 kubelet 直接管理的 Pod，其定义文件放在 `/etc/kubernetes/manifests/` 目录下，kubelet 会监控这个目录的变化。使用 Static Pod 的好处：
1. kubelet 自动重启故障容器，无需额外的 systemd 管理
2. 控制面组件作为一个整体 Pod 管理，便于日志收集和监控
3. 升级时只需更新 manifest 文件，kubelet 自动滚动更新

生成的 Static Pod 包括：
- **kube-apiserver.yaml**：API Server Pod，三个副本（本地 etcd 模式下）
- **kube-controller-manager.yaml**：Controller Manager Pod
- **kube-scheduler.yaml**：Scheduler Pod
- **etcd.yaml**：etcd Pod（本地 etcd 模式下）

每个 manifest 文件中包含了镜像、启动参数、挂载卷等完整定义。以 apiserver 为例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.1.10:6443
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.1.10
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    image: registry.k8s.io/kube-apiserver:v1.30.0
    ...
```

### 2.3.5 etcd 启动

在默认的 stacked etcd 模式下，etcd 作为 Static Pod 在控制面节点上运行。

初始化流程：
1. 生成 etcd 专用 CA 和证书
2. 创建 etcd Static Pod manifest
3. kubelet 检测到 manifest 变化，启动 etcd 容器
4. etcd 启动后监听 2379（客户端）和 2380（peer）端口

etcd 对资源非常敏感。生产环境中应确保：
- 使用 SSD 存储，etcd 数据目录单独挂载
- 为 etcd 设置适当的资源预留（`--quota-backend-bytes` 默认 2GB）
- 定期备份 etcd（`etcdctl snapshot save`）
- 监控 etcd 的磁盘同步延迟（`etcd_disk_wal_fsync_duration_seconds` 和 `etcd_disk_backend_commit_duration_seconds`）

### 2.3.6 上传配置

`kubeadm init phase upload-config` 将 kubeadm 和 kubelet 配置以 ConfigMap 的形式存储到集群中：

- **kubeadm-config ConfigMap**（`kube-system` 命名空间）：存储集群初始化时的全部配置，供 `kubeadm upgrade` 和 `kubeadm join` 使用
- **kubelet-config ConfigMap**（`kube-system` 命名空间）：存储 kubelet 的集中式配置，后续加入的节点会自动获取此配置

### 2.3.7 标记节点与引导 Token

`kubeadm init phase mark-control-plane` 为控制面节点添加标签和污点：
- Label `node-role.kubernetes.io/control-plane: ""`
- Taint `node-role.kubernetes.io/control-plane:NoSchedule`（阻止普通 Pod 调度到控制面节点）

`kubeadm init phase bootstrap-token` 创建一个引导令牌（Bootstrap Token，格式为 `token-id.token-secret`），存入 `kube-system` 命名空间下的 Secret 中。新节点加入集群时使用这个令牌进行身份验证。令牌默认 24 小时过期，可通过 `--token-ttl` 调整。

### 2.3.8 安装 DNS 和代理插件

`kubeadm init phase addon` 安装以下核心插件：
- **CoreDNS**（`kube-dns`）：集群 DNS 服务器，负责 Service 名称到 ClusterIP 的解析。Pod 的 `dnsPolicy: ClusterFirst` 配置使得 DNS 查询首先发给 CoreDNS。CoreDNS 通过 Deployment 运行，Service ClusterIP 写入每个节点的 `/etc/resolv.conf`。
- **kube-proxy**：作为 DaemonSet 在每个节点上运行，负责维护 iptables/IPVS 规则实现 Service 的负载均衡。

**注意**：CNI 插件（网络插件）不在上述 addon 中，需要用户单独安装。

### 2.3.9 完成标记

当所有 phase 成功完成后，kubeadm 输出集群加入命令和 kubeconfig 提示。此时 API Server 已就绪，kubectl 可以开始使用。

## 2.4 kubeadm join流程

当新节点加入集群时，执行：

```bash
kubeadm join <control-plane-endpoint>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///run/containerd/containerd.sock
```

**join 流程详解**：

1. **Discovery（发现）**：新节点使用 bootstrap token 和 CA 证书哈希向控制面发起 TLS 连接。CA 证书哈希用于验证 API Server 证书的合法性（防止中间人攻击），bootstrap token 用于进行身份验证。

2. **TLS Bootstrap（证书引导）**：节点上没有自己的客户端证书和私钥。kubelet 使用 bootstrap token 作为临时凭证调用 API Server 的 Certificate Signing Request (CSR) API，提交 CSR 请求。如果 token 有效且控制器 `csrapproving` 自动批准，kubelet 获得由集群 CA 签发的客户端证书和私钥。

3. **配置获取**：新节点从 `kube-system/kubelet-config` ConfigMap 获取 kubelet 配置，包括 clusterDNS、clusterDomain、cgroupDriver 等参数。

4. **节点注册**：kubelet 生成节点对象并向 API Server 注册。注册信息包括节点名称、操作系统、容器运行时、网络地址等。

5. **标签和污点**：根据 join 参数添加相应的标签（如 `--node-labels`）和污点。Worker 节点默认没有污点。

6. **启动 kube-proxy 和 CNI**：kubelet 从 API Server 获取 DaemonSet 配置，在节点上启动 kube-proxy 和 CNI 插件 Pod。

## 2.5 CNI网络插件安装

Kubernetes 的容器网络接口（CNI，Container Network Interface）是插件化设计——Kubernetes 本身不提供网络实现，而是通过 CNI 规范与网络插件交互。每次 Pod 创建/删除时，kubelet 调用配置的 CNI 插件来完成网络分配。

### 2.5.1 三大主流 CNI 对比

| 特性 | Calico | Flannel | Cilium |
|------|--------|---------|--------|
| 数据面 | iptables/IPIP/VXLAN/eBPF | VXLAN/host-gw | eBPF |
| 网络策略 | 原生支持（丰富） | 需要额外插件 | 原生支持（丰富） |
| 加密 | WireGuard | 无 | WireGuard/IPSec |
| 性能 | 高 | 中 | 最高 |
| 资源消耗 | 中 | 低 | 高（需要内核5.x+） |
| 社区活跃度 | 最活跃 | 活跃 | 快速增长 |
| 适用场景 | 通用生产 | 简单环境 | 高性能/可观测性 |

**Calico** 使用纯三层（L3）路由范式，不进行二层封装。每个 Pod 获得一个可路由的 IP，节点作为路由器将流量转发到目标 Pod。当底层网络不支持直接路由时，Calico 自动退化为 IPIP 或 VXLAN 封装模式。Calico 的 Felix 组件负责编程 iptables 规则实现网络策略。Calico 的控制面使用 etcd 或 Kubernetes API 存储状态。

**Flannel** 是最简单的 CNI 插件，专注于提供一个可工作的 overlay 网络。其核心原理：每个节点分配一个子网，节点间通过 VXLAN（默认）或 host-gw 封装流量。Flannel 不支持网络策略，需要使用 Canal（Calico+Flannel 的组合）来获得策略能力。

**Cilium** 基于 eBPF 技术，在内核层面直接处理网络流量，绕过 iptables。这使得 Cilium 在处理大规模集群（数千 Service）时性能远超基于 iptables 的插件。Cilium 还提供了 Hubble 作为可观测性平台，可以实时可视化服务间的网络流量。Cilium 也支持 L7 网络策略（HTTP/gRPC/Kafka 协议级别）。

### 2.5.2 Calico 安装示例

```bash
# 下载 Calico manifest
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

# 根据 pod-network-cidr 修改 custom-resources.yaml 中的 cidr
# 默认是 192.168.0.0/16，若使用 --pod-network-cidr=10.244.0.0/16 则需修改

# 安装
kubectl create -f tigera-operator.yaml
kubectl create -f custom-resources.yaml

# 等待所有 Calico Pod 就绪
kubectl get pods -n calico-system -w
```

### 2.5.3 Flannel 安装示例

```bash
# Flannel 安装简单，使用预生成的 manifest
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Flannel 默认使用 VXLAN 模式，Pod CIDR 从 kube-controller-manager 的 --cluster-cidr 参数自动检测
```

## 2.6 验证集群与smoke test

```bash
# 1. 检查节点状态（所有节点应为 Ready）
kubectl get nodes -o wide

# 2. 检查控制面组件（以 Static Pod 形式运行）
kubectl get pods -n kube-system

# 3. 检查集群信息
kubectl cluster-info

# 4. 检查组件状态
kubectl get componentstatuses   # 注意：此命令在 1.19+ 已部分废弃

# 5. 部署测试 Pod 验证调度和网络
kubectl create deployment nginx-test --image=nginx:alpine --replicas=3
kubectl expose deployment nginx-test --port=80 --type=NodePort
kubectl get svc nginx-test

# 6. 验证 DNS 解析
kubectl run -it --rm debug --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default

# 7. 验证跨节点通信
# 让每个 nginx Pod 的 nginx 主页内容显示 Pod 名
# 然后在不同节点上通过 ClusterIP 和 Pod IP 分别访问

# 8. 验证持久卷（可选）
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# 清理测试资源
kubectl delete deployment nginx-test
kubectl delete svc nginx-test
kubectl delete pvc test-pvc
```

一个完整的 smoke test 应覆盖：节点注册、Pod 调度、跨节点网络通信、Service 负载均衡、DNS 解析。只有这五项全部通过，集群才被认为是健康可用的。

# 3. HA高可用集群部署

生产环境的控制面必须高可用——单个 API Server 故障会导致整个集群无法接受请求。

## 3.1 HA拓扑设计

Kubernetes HA 有两种拓扑模式：

### 3.1.1 Stacked etcd（堆叠式 etcd）

```
     Load Balancer
      /    |    \
   CP-1   CP-2   CP-3
  [api]  [api]  [api]
  [cm]   [cm]   [cm]
  [sch]  [sch]  [sch]
  [etcd] [etcd] [etcd]
```

每个控制面节点同时运行 etcd 实例，etcd 集群与控制面共享节点。这是 kubeadm 默认模式。

**优点**：部署简单，减少机器数量，管理等证书在同一节点上
**缺点**：故障域耦合——一个节点故障会同时失去 etcd 成员和控制面组件。在某些灾难场景下，可能需要同时恢复控制面和 etcd。

### 3.1.2 External etcd（外部 etcd）

```
     Load Balancer
      /    |    \
   CP-1   CP-2   CP-3
  [api]  [api]  [api]
  [cm]   [cm]   [cm]
  [sch]  [sch]  [sch]
      \    |    /
     etcd-1 etcd-2 etcd-3
```

etcd 集群运行在独立的节点上，控制面通过负载均衡器或 etcd 端点列表连接。

**优点**：故障隔离——etcd 问题和控制面问题互不影响。etcd 可以独立扩缩容、独立备份恢复。
**缺点**：需要额外 3 台节点，总机器数翻倍，管理复杂。

**选择建议**：对于大多数中小型集群（<50 节点），stacked etcd 完全够用。当集群规模达到 100+ 节点、对控制面可用性要求极高、或 etcd 有特殊性能需求时，选择 external etcd。

## 3.2 负载均衡器部署

负载均衡器是 HA 集群的入口——所有 kubectl 请求、kubelet 通信都经过它。

### 3.2.1 HAProxy + Keepalived 方案

Keepalived 通过 VRRP（Virtual Router Redundancy Protocol）协议在多个负载均衡节点之间提供一个浮动虚拟 IP（VIP）。HAProxy 负责实际将请求代理到后端 API Server。

```bash
# 安装 HAProxy 和 Keepalived
apt-get install -y haproxy keepalived
```

**HAProxy 配置**（`/etc/haproxy/haproxy.cfg`）：

```
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 5s
    timeout client 60s
    timeout server 60s

frontend kubernetes-apiserver
    bind *:6443
    default_backend kubernetes-apiserver-backend

backend kubernetes-apiserver-backend
    option httpchk GET /healthz
    http-check expect status 200
    balance roundrobin
    server cp-1 192.168.1.11:6443 check fall 3 rise 2
    server cp-2 192.168.1.12:6443 check fall 3 rise 2
    server cp-3 192.168.1.13:6443 check fall 3 rise 2
```

**Keepalived 配置**（`/etc/keepalived/keepalived.conf`）：

```
vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER              # 另一节点为 BACKUP
    interface eth0
    virtual_router_id 51
    priority 100              # 另一节点为 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.1.100/24
    }
    track_script {
        check_haproxy
    }
}
```

**为什么使用 TCP 模式而非 HTTP 模式？** kubelet 与 API Server 之间的通信基于 HTTP/2 的 gRPC（List/Watch 机制使用双向流），而 HAProxy HTTP 模式会缓冲和解析 HTTP 请求，可能导致长连接流中断。TCP 模式只是四层转发，对应用层协议透明。

## 3.3 多Master部署步骤

### 3.3.1 初始化第一个控制面节点

```bash
kubeadm init \
  --control-plane-endpoint "192.168.1.100:6443" \
  --apiserver-advertise-address 192.168.1.11 \
  --pod-network-cidr 10.244.0.0/16 \
  --upload-certs
```

关键参数说明：
- `--control-plane-endpoint` 必须指向负载均衡器的 VIP 或 DNS 名称，这是 HA 集群的对外入口
- `--upload-certs` 将证书加密后上传到集群（以 `kubeadm-certs` Secret 形式存储），后续控制面节点加入时自动下载

`init` 成功后会输出两条 join 命令：

```bash
# 控制面节点加入命令（含 --control-plane --certificate-key）
kubeadm join 192.168.1.100:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>

# Worker 节点加入命令
kubeadm join 192.168.1.100:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### 3.3.2 加入其他控制面节点

在第二个和第三个控制面节点上执行控制面 join 命令。join 过程会自动：
1. 从第一个控制面节点下载证书（通过 `--certificate-key` 解密）
2. 生成本地 Static Pod manifest
3. 启动 etcd 并加入现有 etcd 集群
4. 将本节点注册为控制面节点

```bash
# 在其他控制面节点执行
kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:abc123... \
  --control-plane \
  --certificate-key def456...
```

### 3.3.3 加入 Worker 节点

```bash
# 在 Worker 节点执行普通 join
kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:abc123...
```

### 3.3.4 验证 HA 集群

```bash
# 从不同的 kubectl 客户端访问 LB VIP，确认 API Server 可访问
kubectl --server=https://192.168.1.100:6443 get nodes

# 查看 etcd 成员状态（需要进入 etcd 容器）
kubectl -n kube-system exec -it etcd-cp-1 -- etcdctl \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  endpoint status --cluster -w table

# 模拟故障：停止一个控制面节点的 etcd/kubelet
# 观察 API Server 是否仍然可用，etcd 是否重新选出 leader
```

## 3.4 etcd集群独立部署

当选择 external etcd 拓扑时，需要先部署独立的 etcd 集群：

```bash
# 在三台 etcd 节点上安装 etcd
ETCD_VER=v3.5.14
curl -L https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /usr/local/bin --strip-components=1

# 配置 systemd 服务 /etc/systemd/system/etcd.service
# 详见 etcd 官方文档

# 完成后使用 kubeadm init 时指定外部 etcd 端点
kubeadm init \
  --control-plane-endpoint "192.168.1.100:6443" \
  --external-etcd-endpoints "https://192.168.1.21:2379,https://192.168.1.22:2379,https://192.168.1.23:2379" \
  --external-etcd-cafile /etc/kubernetes/pki/etcd/ca.crt \
  --external-etcd-certfile /etc/kubernetes/pki/apiserver-etcd-client.crt \
  --external-etcd-keyfile /etc/kubernetes/pki/apiserver-etcd-client.key
```

# 4. minikube本地开发环境

## 4.1 安装与启动

minikube 支持多种虚拟化驱动（driver），根据宿主机环境选择：

| Driver | 适用平台 | 特点 |
|--------|---------|------|
| docker | Linux/macOS/Windows | 依赖 Docker，最常用 |
| hyperv | Windows | Hyper-V 虚拟机，性能好 |
| virtualbox | 跨平台 | 开源虚拟化，兼容性好 |
| podman | Linux | Podman 替代 Docker |
| none | Linux | 直接运行在宿主机，风险高 |

### 4.1.1 安装 minikube

```bash
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# macOS
brew install minikube

# Windows (PowerShell)
winget install minikube
```

### 4.1.2 启动集群

```bash
# 使用 Docker driver（默认）
minikube start --driver=docker

# 指定 Kubernetes 版本
minikube start --kubernetes-version=v1.30.0

# 指定资源
minikube start --cpus=4 --memory=8192 --disk-size=40g

# 启用 CR 容器运行时
minikube start --container-runtime=containerd

# 使用国内镜像源（中国大陆用户）
minikube start --image-mirror-country=cn
```

minikube start 的内部流程：
1. 检查 driver 是否可用
2. 创建虚拟机/容器
3. 在 VM 中安装容器运行时
4. 下载 K8s 组件镜像
5. 生成证书
6. 启动 kubelet、kube-apiserver 等
7. 配置 `~/.kube/config`

## 4.2 常用参数与插件

```bash
# 查看状态
minikube status

# 查看插件列表
minikube addons list

# 启用插件
minikube addons enable ingress      # Nginx Ingress Controller
minikube addons enable metrics-server # 资源指标收集
minikube addons enable dashboard    # Kubernetes Dashboard
minikube addons enable registry     # 内置 Docker Registry

# 访问 Dashboard
minikube dashboard

# SSH 进入节点
minikube ssh

# 查看日志
minikube logs

# 停止/暂停集群
minikube stop
minikube pause    # 暂停（保留状态）
minikube unpause

# 修改资源（需重启）
minikube config set memory 4096
minikube config set cpus 4

# 删除集群
minikube delete
```

## 4.3 多集群管理

minikube 支持通过 profile 管理多个独立的集群，每个 profile 是一个隔离的虚拟机/容器：

```bash
# 创建多个集群
minikube start -p cluster-dev
minikube start -p cluster-test --kubernetes-version=v1.29.0

# 列出所有集群
minikube profile list

# 切换集群
minikube profile cluster-dev

# 针对特定 profile 操作
minikube status -p cluster-test
minikube delete -p cluster-test
```

多 profile 场景在开发中非常有用：可以同时运行不同 K8s 版本、测试升级路径、模拟多集群服务网格等。

# 5. kind (Kubernetes in Docker)

kind 通过 Docker 容器模拟 K8s 节点，是 CI/CD 集成测试的最优选择。

## 5.1 安装与使用

```bash
# 安装
# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# macOS
brew install kind

# 创建默认单节点集群
kind create cluster

# 指定名称和版本
kind create cluster --name my-cluster --image kindest/node:v1.30.0

# 列出集群
kind get clusters

# 切换 kubectl context
kubectl cluster-info --context kind-my-cluster

# 加载 Docker 镜像到 kind 节点（避免推送到 registry）
kind load docker-image my-app:v1 --name my-cluster

# 删除集群
kind delete cluster --name my-cluster
```

## 5.2 多节点集群配置

kind 通过 YAML 配置文件定义集群拓扑：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: multi-node-cluster
nodes:
  # 控制面节点
  - role: control-plane
  - role: control-plane
  - role: control-plane
  # Worker 节点
  - role: worker
    extraMounts:
      - hostPath: /path/on/host
        containerPath: /path/in/node
  - role: worker
  - role: worker
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  # 禁用默认 CNI，安装自己的 CNI
  disableDefaultCNI: false
  # API Server 端口映射到宿主机
  apiServerAddress: "0.0.0.0"
  apiServerPort: 6444
# 自定义 kubeadm 配置
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        "service-node-port-range": "30000-40000"
```

创建集群：

```bash
kind create cluster --config kind-config.yaml
```

**kind 多控制面实现原理**：每个"控制面节点"容器内运行着完整的 K8s 控制面组件（apiserver/etcd/controller-manager/scheduler）。第一控制面容器通过 kubeadm 初始化，后续控制面通过 kubeadm join 加入。kind 使用一个内置的 HAProxy 容器作为控制面的负载均衡器，确保 kubectl 访问点的高可用。

## 5.3 适用场景

kind 的核心优势：
- **CI/CD 集成测试**：在 GitHub Actions、Jenkins 等 CI 流水线中快速启动集群运行 e2e 测试
- **控制面组件开发**：K8s 核心开发者使用 kind 快速测试 kube-apiserver/kube-scheduler 的修改
- **Operator 开发测试**：operator 开发者可以在本地用 kind 验证 CRD 和 controller 逻辑
- **零网络依赖**：镜像预置在 kind 的 node 镜像中，无需访问外部 registry

kind 不适合的场景：
- 生产环境部署（单点故障风险）
- 长期运行的服务
- 需要 GPU 或特殊硬件的场景

# 6. k3s轻量级K8s

## 6.1 架构特点

k3s 的设计原则是"最小化外部依赖"，通过以下几项关键架构简化实现轻量化：

**单二进制**：k3s 将 kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy 的所有代码编译为一个约 100MB 的二进制文件（对比：完整 K8s 部署需要 5 个二进制文件，总计约 500MB+ 镜像）。这意味着"安装 k3s"本质上就是复制一个二进制文件。

**SQLite 替代 etcd**：k3s 默认使用 SQLite 作为数据存储。为什么可以这样做？

- k3s 的目标场景是单节点或小规模集群（3 节点），SQLite 的并发写瓶颈（单写者）在这种情况下不是问题
- SQLite 零配置、零运维，不需要像 etcd 那样管理证书、快照、碎片整理
- 减少了 etcd 的 2-4GB 内存消耗

但在 HA 模式下，k3s 仍支持使用 etcd 作为后端存储。

**内置 containerd 和 flannel**：k3s 二进制内嵌了 containerd 作为容器运行时、flannel 作为 CNI 插件。用户安装 k3s 后即可直接使用，无需额外安装任何组件。

**移除非核心功能**：k3s 移除了以下功能以减少二进制体积：
- 云厂商控制器（cloud-controller-manager）
- Alpha 特性门控
- 不再维护的内置 provider
- 旧版 API 和废弃特性
- Windows 支持（k3s 只支持 Linux）

## 6.2 安装与部署

```bash
# 一行命令安装（需要 root）
curl -sfL https://get.k3s.io | sh -

# 或指定版本
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.30.0+k3s1 sh -

# 安装时指定参数
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --cluster-cidr 10.244.0.0/16 \
  --service-cidr 10.96.0.0/12

# 安装后查看状态
systemctl status k3s
kubectl get nodes   # k3s 自动配置 kubeconfig 到 /etc/rancher/k3s/k3s.yaml
```

在 Worker 节点上加入：

```bash
# 首先从 Server 节点获取 token
cat /var/lib/rancher/k3s/server/node-token

# 在 Worker 节点执行
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 \
  K3S_TOKEN=<node-token> sh -
```

**使用国内镜像源**：

```bash
# 使用国内镜像加速
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn sh -
```

## 6.3 高可用配置

k3s 支持两种 HA 模式：

**嵌入式 etcd 模式**（推荐，k3s v1.19.1+）：

```bash
# 第一台 Server 节点
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --token my-secret-token

# 第二台和第三台 Server
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://<first-server-ip>:6443 \
  --token my-secret-token
```

**外部数据库模式**（使用 MySQL/PostgreSQL 替代 SQLite/etcd）：

```bash
# 所有 Server 节点指定外部数据库
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint "mysql://k3s:k3s-password@tcp(192.168.1.100:3306)/k3s"
```

HA 部署时，建议在 Server 节点前放置 TCP 负载均衡器（如 HAProxy/Nginx），将 6443 端口流量分发到所有 Server 节点。

# 7. 集群升级

## 7.1 升级策略与版本偏差策略

### 7.1.1 Kubernetes 版本偏差策略

Kubernetes 集群的版本偏差（Version Skew）定义了各组件之间允许的最大版本差异：

| 组件关系 | 允许偏差 |
|---------|---------|
| kube-apiserver vs 其他控制面组件 | 无偏差（必须同版本） |
| kube-apiserver vs kubelet | kubelet 版本 ≤ API Server，且 ≤ 2 个 minor 版本 |
| kube-apiserver vs kubectl | kubectl 版本 ≤ API Server，且 ≤ 1 个 minor 版本 |
| kube-apiserver vs kube-proxy | kube-proxy 版本差 ≤ 1 个 minor 版本 |

**设计意图**：这个策略的哲学是"API Server 是最新的"。API Server 是集群的单一权威数据源，其他组件都是它的客户端。限制 kubelet 不能比 API Server 更新，是因为新 kubelet 可能引入旧 API Server 不理解的字段；允许 kubectl 比 API Server 高 1 个 minor 版本，是为了让运维人员可以在升级集群前先升级本地工具。

### 7.1.2 升级策略

**原地升级（In-place Upgrade）**：在同一台机器上替换组件二进制或更新容器镜像。kubeadm 默认采用此方式。

**蓝绿升级（Blue-Green Upgrade）**：创建全新集群，在验证后将流量切换过去。对控制面组件的升级最安全，但成本最高。

**滚动升级（Rolling Upgrade）**：逐个替换节点，适用于 Worker Node 的大规模升级。

### 7.1.3 升级前的必要准备

```bash
# 1. 备份 etcd
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup-$(date +%Y%m%d).db

# 2. 确认版本兼容性
kubeadm version
kubectl version
kubelet --version

# 3. 检查证书有效期
kubeadm certs check-expiration

# 4. 确认当前版本和升级路径（不支持跨多版本升级，如 1.28→1.30 不允许）
kubectl get nodes

# 5. 排出当前节点的 Pod（如果是控制面节点升级）
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

## 7.2 kubeadm升级流程

升级顺序必须是：**控制面 → kubelet → kubectl**。先升级控制面确保集群 API 版本变更已完成，再升级 kubelet 使节点跟上。

### 7.2.1 升级包管理工具中的版本

```bash
# Ubuntu/Debian
apt-get update
apt-get install -y kubeadm=1.30.0-1.1

# CentOS/RHEL
yum install -y kubeadm-1.30.0
```

### 7.2.2 验证升级计划

```bash
# 检查集群是否可以升级
kubeadm upgrade plan

# 输出示例：
# COMPONENT   CURRENT   TARGET
# kubeadm     v1.29.0   v1.30.0
# kubelet     v1.29.0   v1.30.0
# kubectl     v1.29.0   v1.30.0
# ...
# You can now apply the upgrade by executing the following command:
#   kubeadm upgrade apply v1.30.0
```

### 7.2.3 升级第一个控制面节点

```bash
# 执行升级（kubeadm 会按 phase 执行：preflight → control-plane → kubelet-config）
kubeadm upgrade apply v1.30.0

# 如果希望分别执行各阶段便于排查：
# kubeadm upgrade apply v1.30.0 --dry-run
# kubeadm upgrade apply phase preflight v1.30.0
# kubeadm upgrade apply phase control-plane v1.30.0

# 升级 kubelet 和 kubectl
apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
systemctl daemon-reload
systemctl restart kubelet

# 恢复节点调度
kubectl uncordon <node-name>
```

### 7.2.4 升级其他控制面节点

```bash
# 对于非首个控制面节点，使用 upgrade node 而非 upgrade apply
kubeadm upgrade node

# 升级 kubelet
apt-get install -y kubelet=1.30.0-1.1
systemctl restart kubelet
```

### 7.2.5 升级 Worker 节点

```bash
# 1. 从集群排出 Pod
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

# 2. 升级 kubeadm
apt-get install -y kubeadm=1.30.0-1.1

# 3. 升级节点配置
kubeadm upgrade node

# 4. 升级 kubelet
apt-get install -y kubelet=1.30.0-1.1
systemctl restart kubelet

# 5. 恢复调度
kubectl uncordon <worker-node>
```

### 7.2.6 最后升级 kubectl

```bash
# 在所有管理机器上升级 kubectl
apt-get install -y kubectl=1.30.0-1.1
```

**为什么 Worker 节点升级要 drain？** drain 命令会自动驱逐节点上所有非 DaemonSet 管理的 Pod。这个过程确保：
1. Pod 被优雅地终止并重建到其他节点，避免服务中断
2. 运行中的容器不会因为 kubelet 重启而丢失状态
3. PDB（PodDisruptionBudget）得到遵守，不会同时终止超过允许数量的副本

## 7.3 升级注意事项与回滚

### 7.3.1 关键注意事项

- **不支持跨版本升级**：从 1.28 升级到 1.30 不允许，必须经过 1.29。这是 K8s 版本策略的硬性要求。
- **控制面升级期间集群不可管理**：API Server 重启期间，kubectl 无法使用，但其上运行的现有 Pod 不受影响。
- **升级期间不要执行集群变更**：避免创建/删除资源、修改 RBAC 等操作。
- **CNI 版本兼容性**：升级 K8s 时应同步检查 CNI 插件是否有版本要求。部分 CNI 插件需要先升级到特定版本才能兼容新的 K8s API。
- **API 弃用检查**：使用 `kubectl convert` 或 `pluto` 工具检查现有资源是否使用了即将被移除的 API 版本（如 `extensions/v1beta1` 已在 1.16 移除）。

### 7.3.2 回滚方法

kubeadm 本身不提供 `rollback` 命令。回滚方式取决于故障范围：

**控制面回滚**（API Server 启动失败等）：
```bash
# 1. 从 etcd 快照恢复（如果数据需要回滚）
# 2. 用旧版 kubeadm 重新初始化和加入（灾难性场景）
# 3. 或直接手动回滚 Static Pod 的镜像 tag
```

**节点回滚**：
```bash
# 降级 kubelet 包
apt-get install -y kubelet=1.29.0-1.1
systemctl restart kubelet
```

由于 K8s 不允许 kubelet 版本高于 API Server，回滚时必须先将 kubelet 降级到与 API Server 兼容的版本。

# 8. 集群卸载与清理

## 8.1 kubeadm reset流程

`kubeadm reset` 是将节点恢复到加入集群前状态的标准方式：

```bash
# 基本 reset
kubeadm reset

# 完整清除（包括镜像和配置）
kubeadm reset --cri-socket unix:///run/containerd/containerd.sock

# 强制重置（即使 API Server 不可达）
kubeadm reset --force
```

**reset 执行的步骤**：

1. **Preflight**：检查 API Server 是否可达，尝试获取节点信息
2. **删除节点对象**：如果 API Server 可达，从集群中移除本节点（`kubectl delete node`）
3. **清理 etcd**：删除本节点上的 etcd 数据（`/var/lib/etcd`），并尝试从 etcd 集群中移除本成员
4. **停止 kubelet**：`systemctl stop kubelet`
5. **清理 Static Pod 目录**：删除 `/etc/kubernetes/manifests/` 下的所有文件，kubelet 自动停止所有控制面 Static Pod
6. **清理证书和配置**：删除 `/etc/kubernetes/` 目录（可通过 `--keep-pki` 保留证书）
7. **清理网络配置**：删除 CNI 配置（`/etc/cni/net.d/`）和接口
8. **卸载 kubelet 数据目录**：清理 `/var/lib/kubelet/` 内容
9. **清理 iptables/IPVS 规则**：移除所有 K8s 相关的网络规则
10. **清理 IPVS 虚拟接口**：删除 kube-ipvs0 接口
11. **重启 containerd**：清理容器运行时状态

## 8.2 清理残留数据

`kubeadm reset` 不会自动清理以下内容：

```bash
# 1. 清理包管理安装的组件（如果需要完全卸载）
# Ubuntu/Debian
apt-get purge -y kubeadm kubelet kubectl
apt-get autoremove -y

# CentOS/RHEL
yum remove -y kubeadm kubelet kubectl

# 2. 清理 CNI 残留文件
rm -rf /etc/cni/net.d
rm -rf /opt/cni/bin

# 3. 清理 kubeconfig（避免与新集群混淆）
rm -rf ~/.kube/config
rm -rf /root/.kube/config

# 4. 清理内核参数（如果不再需要）
rm -f /etc/sysctl.d/k8s.conf
sysctl --system

# 5. 清理容器运行时中的 K8s 容器和镜像
crictl rm -a              # 删除所有容器
crictl rmi --prune        # 删除未使用的镜像
crictl stopp $(crictl pods -q)  # 停止所有 Pod

# 6. 彻底清理 containerd（如需完全移除）
systemctl stop containerd
apt-get purge -y containerd.io  # 或 yum remove containerd.io
rm -rf /var/lib/containerd
rm -rf /etc/containerd

# 7. 清理 overlay 挂载（防止 umount 失败）
mount | grep overlay | awk '{print $3}' | xargs -r umount

# 8. 重启节点（最终手段，确保一切清理干净）
reboot
```

**多节点清理顺序**：先 `kubeadm reset` 所有 Worker 节点，再 reset 控制面节点。这样能确保 Worker 节点上的 Pod 被优雅驱逐后再清理控制面，避免集群状态不一致。

```bash
# 快速批量清理脚本
for node in worker-{1..3}; do
  ssh $node "sudo kubeadm reset --force"
done

for node in cp-{1..3}; do
  ssh $node "sudo kubeadm reset --force"
done
```
