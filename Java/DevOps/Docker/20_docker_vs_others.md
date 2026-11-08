
# 1. 先明确比较维度

很多"Docker vs XX"的争论之所以没有结论，是因为比较维度混在一起了。正确做法是按层次比较：

1. **打包方式**：把应用与依赖如何封装起来？
2. **运行隔离**：进程、文件系统、网络如何隔离？
3. **交付方式**：如何分发与版本化？
4. **编排治理**：多实例、多主机、HA、发布如何管理？
5. **资源与安全**：如何限制与加固？

Docker 主要解决的是"容器化打包 + 运行"这一层；编排则是另一个层次的问题（见 `17_orchestration.md`）。"Docker vs K8s""Podman vs Kubernetes"这类问法本身就是层次错配。要理解生态竞争，先看清 OCI 标准把整个体系切成了三块：

```text
┌───────────────────────── 开放容器倡议 OCI ─────────────────────────┐
│  image-spec          runtime-spec          distribution-spec       │
│  (镜像格式:层/config/manifest)  (runtime.json+bundle: 如何起容器)  (registry HTTP API)
│        │                    │                       │              │
│  任何工具可产镜像 ────▶ 任何 OCI 运行时可消费 ──▶ 任何 Registry 可分发 │
└─────────────────────────────────────────────────────────────────────┘

按层看实现:

  用户界面层   Docker CLI │ Podman CLI │ nerdctl │ crictl │ kubectl
  ────────────┴──────────┴────────────┴─────────┴────────┴─────────
  高层运行时   dockerd(containerd) │ podman(无daemon) │ containerd(直连)
  ──────────────────────────────────────────────────────────
  低级运行时   runc(默认) │ crun(更快启动) │ runsc(gVisor沙箱) │ kata(VM级隔离)
  ──────────────────────────────────────────────────────────
  编排/平台    Swarm │ Kubernetes(CRI: containerd/CRI-O) │ Nomad
```

**这个分层是全部选型讨论的前提**：只要各方都实现 OCI 标准，镜像与 registry 就是通用货币，竞争发生在"用户界面层与高层运行时的体验/架构差异"上，而不是"格式互斥"。

# 2. Docker vs 虚拟机（VM）

## 2.1 Docker 的特点

1. 共享宿主机内核，启动快、资源开销小。
2. 镜像分层，交付效率高。
3. 适合微服务与快速迭代。

## 2.2 VM 的特点

1. 每个虚拟机有自己的操作系统内核，隔离更强。
2. 启动相对慢、资源开销大。
3. 适合强隔离、异构内核、传统虚拟化场景。

## 2.3 选择建议

1. 需要快速交付、密集部署、CI/CD：Docker 更合适。
2. 需要更强隔离、完整 OS 语义、特殊内核特性：VM 更合适。

| 维度 | 容器 | 虚拟机 |
|------|------|--------|
| 启动 | 毫秒~秒 | 十几秒~分钟 |
| 密度 | 单机数百容器 | 受内存/CPU 虚拟化开销限制 |
| 隔离 | namespace/cgroup/seccomp, 共享内核 | hypervisor 硬边界 |
| 逃逸面 | 内核 + 配置 | hypervisor(更窄) |
| 异构 OS | 不可(只能 Linux 内核) | 可(Linux/Windows 任意) |
| 内核定制 | 受限(全节点共享) | 自由(每 VM 独立内核) |

两者不是终态替代关系，而是**融合**：生产上"VM 里跑容器"是云厂商与客户私有化的主流形态（GKE 自动节点、各类托管 K8s 都跑在 VM 里），多租户强隔离场景用 Kata Containers/gVisor 把容器包进轻量 VM，兼得 OCI 生态与虚拟化边界。

# 3. Podman：daemonless 的直接竞争者

Podman 的常见卖点：

1. Rootless 体验更自然。
2. 无中心 daemon 模式（架构上不同）。
3. 与 OCI 生态兼容较好。

Docker 的优势：

1. 生态成熟、文档与工具链广泛。
2. 团队学习成本低，实践案例多。

**架构差异**是理解两者所有体验差别之源：

```text
Docker:  docker CLI ──▶ dockerd(常驻, root) ──▶ containerd ──▶ runc
                         │ 所有状态与子进程都挂在 daemon 下
Podman:  podman CLI ──(fork/exec)──────────────────────────▶ conmon ──▶ runc
         无常驻守护进程, 每次命令直接操作 cgroup/创建容器,
         容器进程父级是 systemd(conmon 过继), rootless 天然可用
```

由此推出的行为差异：

| 维度 | Docker | Podman |
|------|--------|--------|
| daemon | dockerd 常驻（单点、升级影响面） | 无，CLI 直连 |
| rootless | rootless 模式是"额外部署形态" | 默认设计即 rootless |
| API 兼容 | - | `podman system service` 提供 docker.sock 兼容 API，多数 docker SDK/Compose v2 可直接换 |
| 迁移成本 | - | `docker`→`podman` 别名几乎零改动；compose 经 podman-compose 或 docker-compose 指向兼容 socket |
| Pod 概念 | 无 | 支持 K8s 式 pod（`podman play kube` 可跑/生成 K8s YAML） |
| 生态集成 | Compose/Docker Desktop/Registry UI 成熟 | RHEL/Fedora 默认；Win/Mac 需 VM（podman machine），体验略逊 |

选择建议：

1. 你要的是"Docker 生态与使用习惯"，Docker 仍是主流选择。
2. 如果你特别关注 rootless/daemon-less 体验，可评估 Podman。

> **注意**："无 daemon"不是纯优势——没有常驻进程意味着**没有开机默认拉起容器的机制**（要靠 systemd unit/podman generate systemd），镜像预热、网络策略集中管理也需自建。Red Hat 官方口径是：开发工作站推 Podman，生产集群交给 OpenShift(K8s)，"单节点容器平台"这一格两边都能填但都不完美。

# 4. containerd 与 CRI-O：K8s 时代"去掉 Docker"的真相

Kubernetes 1.24 移除 dockershim 后，kubelet 通过 **CRI（容器运行时接口）** 直接对接高层运行时：

```text
kubelet ──CRI(gRPC)──▶ containerd ──▶ runc        （主流, 云厂商托管集群）
kubelet ──CRI(gRPC)──▶ CRI-O     ──▶ runc/crun    （OpenShift/安全加固场景）
(曾经) kubelet ──dockershim──▶ dockerd ──▶ containerd ──▶ runc  （多一跳, 1.24 移除）
```

两个常被误读的点必须澄清：

1. **"K8s 抛弃 Docker"是标题党**。被移除的是 kubelet→docker 的 shim 胶水层；Docker 构建的镜像（OCI 兼容）在 containerd 上原样运行，registry 也无感知。节点上甚至仍可并存：人在 SSH 里用 docker CLI 管理开发容器，kubelet 用 containerd 管理 Pod，只是两套 daemon 各占资源，不建议。
2. **你本地用的 Docker Engine 内部就是 containerd**（见 `2_architecture.md`），差异只在"是否暴露 CRI 接口 + 是否带 dockerd 的 Compose/网络体验层"。containerd 官方提供 `nerdctl` 给出近似 docker CLI 的体验；CRI-O 则是 Red Hat 主导的"只为 K8s 精简"的运行时，不带通用 CLI。

**设计考量**：Java 团队选型真正的分水岭不是 Docker/Podman 之争，而是**构建环境放哪**——开发者本机（Docker Desktop/Podman machine）、CI runner（BuildKit/Kaniko）、还是平台内置构建（OpenShift Streams/Tekton）；三者的镜像内容必须逐字节一致，否则"本地能跑线上不行"的锅最终都算到容器头上。

# 5. Docker vs Kubernetes

这不是同一层次的替代关系：

1. Docker 主要解决容器构建与运行。
2. Kubernetes 解决大规模编排治理。

关系可以理解为：

```text
应用构建 -> 镜像制作（Docker） -> 单机/多机运行 -> 编排治理（Kubernetes）
```

选择建议：

1. 本地开发、测试、小规模部署：Docker/Compose 足够。
2. 多副本、跨节点、HA、发布策略：Kubernetes 更适合。

# 6. Docker vs 直接在宿主机运行

宿主机直接跑应用的缺点：

1. 依赖污染，版本冲突难控制。
2. 环境不可复制。
3. 部署/回滚/迁移困难。

Docker 的优势：

1. 环境一致性更好。
2. 更容易在 CI/CD 中标准化。
3. 回滚简单（切换镜像版本）。

补充一层成本清单才公允：引入容器同时付出内核共享的安全面（`13_security.md`）、学习曲线（构建/网络/存储的抽象都要懂才能排障）、以及 Docker Desktop 在 macOS/Windows 上的 VM 性能税。**"值不值"的分界线**是：部署目标 ≥2 台机器、或依赖冲突发生过一次、或需要环境在 CI 与生产间严格一致——过线就值得，纯个人玩具与单机固定环境则裸机+systemd 更省心。

# 7. Java 开发者视角的选型判断

| 场景 | 推荐 | 理由 |
|------|------|------|
| 本地开发/联调 | Docker Desktop + Compose | IDE 集成、Volume 性能优化、生态最全 |
| 公司统一 CI 构建镜像 | Docker/BuildKit 或 Kaniko | BuildKit 缓存成熟; K8s 内构建选 Kaniko 免 dind |
| RHEL/Fedora 开发机、强 rootless 需求 | Podman | 系统默认、无 daemon、SELinux 集成好 |
| 生产运行平台 | Kubernetes（containerd 运行时） | 事实标准; 镜像用 docker/BuildKit 构建照常 |
| 单机交付的私有化产品 | Compose 或 Podman Quadlet(systemd) | 无集群运维负担, 重启策略交给 systemd 更稳 |
| 多租户/对外不可信负载 | Kata/gVisor 或 VM | 容器边界不够, 上 VM 级隔离 |

**判断表（Docker 还是 Podman）**：

| 问题 | 答"是"→ |
|------|---------|
| 依赖 Docker Desktop 图形体验/Compose 全家桶？ | Docker |
| 团队脚本与 CI 已大量使用 docker CLI/SDK？ | Docker（Podman 有兼容层但迁移要回归测试） |
| 共享开发机/堡垒机，需要 rootless 降权？ | Podman |
| 内核禁止 dockerd 的 iptables/cgroup 玩法，安全基线极严？ | Podman + 受限配置 |
| 跑 K8s 且节点层面运行时由平台组统一？ | 无需选——containerd/CRI-O 在底下，你只管 docker build |

> **行业实践**：Java 团队的现实路径几乎总是"**开发期 Docker/Podman 任选，生产交给 K8s**"。开发者真正要投资的是 OCI 通用能力——Dockerfile 质量、构建缓存、SBOM/签名流程——这些在 Docker、Podman、Buildah、Tekton、K8s 生态间完全可迁移；纠结 daemon 存在与否，远不如把镜像瘦身和依赖治理做扎实（`15_cicd.md`、`18_best_practices.md`）。

# 8. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| Docker 和 VM 的区别 | 共享内核 vs 独立内核，启动速度、隔离、资源开销 |
| OCI 标准包含什么 | image-spec/runtime-spec/distribution-spec, 分层解耦格式与实现 |
| Docker 和 K8s 的关系 | 不同层次：构建/运行 vs 编排治理 |
| 1.24 dockershim 真相 | 移除的是胶水层, 镜像与 registry 无感知, 节点运行时=containerd/CRI-O |
| Podman vs Docker | daemonless+原生 rootless vs 生态成熟; socket 兼容可互替 |
| 强隔离怎么做 | Kata 容器/gVisor, 或 VM 里跑容器 |

# 9. 小结

这一篇的方法论是"**先分层，再比较**"：OCI 三份规范把生态切成镜像格式、运行时、registry 协议三层，格式通用后，工具之争只剩架构与体验——Docker（daemon 换全家桶生态）、Podman（daemonless 换 rootless 与 systemd 亲和）、containerd/CRI-O（K8s 原生的运行时形态）、Kata/gVisor（用 VM 边界补容器隔离强度）、K8s（编排层，不与 Docker 同维）。Java 开发者的务实姿势：本地 Docker/Podman 皆可，生产交给 K8s，个人能力押注在 OCI 通用实践（Dockerfile、BuildKit、SBOM、签名）上，这部分不随平台选择贬值。性能视角（存储驱动、IO、启动加速）是最后一篇 `21_performance.md` 的主题。
