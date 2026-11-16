
# 1. 为什么需要编排

当你不再是"单机跑几个容器"，你会遇到这些问题：

1. 多主机部署：服务要跑在哪台机器上？
2. 高可用：某台机器挂了，服务如何自动迁移？
3. 扩缩容：流量上来自动加实例，流量下去自动缩。
4. 滚动升级：不中断服务地升级版本，失败可回滚。
5. 服务发现与负载均衡：实例数量变化时，调用方如何找到它们？
6. 配置与密钥：如何安全地注入到运行时？
7. 资源与隔离：配额、优先级、抢占与调度策略。
8. 可观测性与治理：健康检查、自动重启、指标、日志、审计。

这些问题合在一起就是"编排"要解决的范畴。单机 Docker/Compose 只覆盖第 5、6 项的一部分与第 8 项的最低形态，剩下的全部需要集群级控制面——这就是编排系统与 Compose 的分界线：**Compose 描述"一台机器上的拓扑"，编排系统描述"一个集群的期望状态"**。

# 2. 编排系统的核心能力拆解

可以把编排系统看成一组能力集合：

1. **期望状态（Desired State）**：你声明"我要 3 个副本"，系统负责逼近这个状态。
2. **调度（Scheduling）**：把工作负载放到合适节点（考虑资源、亲和性、约束）。
3. **自愈（Self-healing）**：进程挂了重启、节点挂了迁移。
4. **发布策略（Rollout）**：滚动更新、金丝雀、蓝绿、回滚。
5. **服务发现/入口（Service/Ingress）**：稳定的访问入口与负载均衡。
6. **配置与密钥（Config/Secret）**：运行时注入与权限控制。
7. **网络模型（Networking）**：跨主机通信、隔离、策略。
8. **存储编排（Storage）**：持久化卷、动态供给、故障恢复。

其中 1 是哲学、3 是底线、2 是引擎。期望状态 + 控制循环（reconciliation loop）是所有编排系统的共同内核：你提交声明，控制器对比"实际状态 vs 期望状态"并持续采取动作消除差异。理解这一点后，"编排 = 写 YAML"的误解自然消散——YAML 只是输入，**平台价值在于那个永远在线的差异收敛循环**。

```text
        期望状态(YAML/CLI/API)
              │ apply
              ▼
  ┌───────────────────────────┐        ┌──────────┐
  │ 控制面: 调度器/控制器/状态存储 │──动作──▶│ 节点 agent │──▶ 容器
  └───────────────────────────┘        └──────────┘
              ▲  实际上报(心跳/健康/资源)      │
              └──────────────────────────────┘
        控制循环: 不断对比 期望 vs 实际, 消除差异
```

# 3. 三大编排平台对比：Swarm、Kubernetes、Nomad

| 维度 | Docker Swarm | Kubernetes | Nomad |
|------|--------------|------------|-------|
| 抽象单位 | Service/Task（即容器） | Pod（容器组）+ 丰富工作负载 | Job/TaskGroup（容器或非容器进程均可） |
| 状态存储 | Raft（内置） | etcd（Raft） | 内置 Raft 或外部存储 |
| 调度模型 | 单机调度器, 规则简单 | 多过滤器+打分, 亲和/污点/拓扑 | 过滤+打分, 强调 bin-packing |
| 网络 | 内置 overlay + VIP/IPVS | CNI 插件生态 | 依赖外部（可接 Consul） |
| 存储 | volume 插件 | CSI 生态 + StorageClass 动态供给 | 较薄, 走插件 |
| 发布治理 | 滚动更新/回滚(基础) | Deployment/金丝雀(生态)/蓝绿 | 原生 update strategy |
| 服务网格/扩展 | 弱 | CRD/Operator/Service Mesh 全家桶 | 弱, 靠 Consul 生态 |
| 学习/运维成本 | 低 | 高 | 中 |
| 事实地位 | 已边缘化（Mirantis 社区维护） | 行业标准 | 特定场景(Hybrid 负载) |

**设计考量**：

- **Swarm 的卖点曾是"零额外组件"**：docker 自带、`docker swarm init` 一句话起集群，compose 文件稍改就是 stack。代价是抽象停留在"容器"层，没有 Pod 语义、扩展机制弱。
- **Nomad 的差异化**是"不只是容器编排"：传统二进制、Java 进程（非容器）、批处理任务都能调，适合容器与非容器混部的存量系统；但对纯容器场景，K8s 生态虹吸让 Nomad 的周边（网络、存储、服务发现）需要自己拼装。
- **K8s 的复杂度是交换物**：换来声明式 API、控制器模式的可扩展性（CRD+Operator 把"运维知识"编码进平台）、以及无法替代的生态位。

# 4. Swarm 速览：stack 示例与调度约束

虽然 Swarm 已边缘化，理解它仍有价值：它是最小可用的编排系统，概念量只有 K8s 的几分之一。

```yaml
# order-stack.yml —— docker stack deploy -c order-stack.yml order
services:
  order-service:
    image: harbor.corp.com/pay/order-service:1.2.3-8f3c2a1
    ports:
      - "8080:8080"
    deploy:
      mode: replicated
      replicas: 3
      update_config:
        parallelism: 1          # 一次换 1 个副本
        delay: 10s              # 每个之间观察窗口
        failure_action: rollback # 失败自动回滚, 这是 Swarm 的亮点之一
        order: start-first      # 3.x 起支持: 先起新再停旧(需应用容忍瞬时双实例)
      restart_policy:
        condition: any
        delay: 5s
      placement:
        constraints:
          - node.labels.zone == dc1-a   # 亲和约束
          - node.role != manager        # 不占控制面资源
      resources:
        limits: { cpus: '2.0', memory: 512M }
        reservations: { memory: 256M }

  mysql:
    image: mysql:8.0.33
    deploy:
      mode: global              # 每节点一份(对比 replicated: 按副本数散布)
      placement:
        constraints: [ node.labels.disk == ssd ]
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

```bash
docker swarm init --advertise-addr 192.168.1.10
docker node update --label-add zone=dc1-a worker-1
docker stack deploy -c order-stack.yml order
docker service ls / docker service ps order_order-service   # 看任务落点与历史
docker service update --image harbor.corp.com/pay/order-service:1.2.4 order_order-service
docker service rollback order_order-service                  # 回滚上一次变更
```

调度的三层控制：**constraints**（硬性过滤，不满足就不调度，如上标签/角色约束）、**preference**（软性加分，`--placement-pref spread=node.labels.zone` 让副本跨区打散）、**资源预留**（reservation 参与节点筛选）。服务更新策略与回滚的"自动失败回滚"是 Swarm 开箱体验最好的部分，K8s 原生 Deployment 反而要靠 Argo Rollouts/Flagger 补齐等价能力。

> **注意**：`docker stack deploy` 用的 compose 文件只是"Compose 的子集 + deploy 扩展"，不支持 build、不支持 profiles；stack 是"期望状态快照"，直接 `docker run` 加进节点的容器不会被 stack 管理，混用两套操作必然漂移。

> **行业实践**：仍在 Swarm 上的团队迁移 K8s 的普遍路径是"先补可观测（Prometheus）与 CI/CD（GitOps），再平移工作负载"——无状态服务改 Deployment+Service 一天能迁完，真正的周期都花在存储（volume 插件→CSI）与网络（overlay→CNI）的语义对齐上。

# 5. Kubernetes 一页纸定位

与 Swarm 概念对齐后，K8s 的增量一目了然：

| 问题 | Swarm | Kubernetes |
|------|-------|------------|
| 部署单位 | Service(replicated/global) | Deployment/DaemonSet/StatefulSet/Job |
| 最小调度单位 | Task(容器) | Pod(共享 netns 的容器组) |
| 服务发现 | 内置 DNS+VIP/IPVS | CoreDNS+Service(kube-proxy iptables/IPVS/nft) |
| 入口 | ingress 路由网格 | Ingress/Gateway API |
| 配置/密钥 | Config/Secret(全量更新触发重启) | ConfigMap/Secret + 热挂载 + 外部 KMS 生态 |
| 自动伸缩 | 基本无(实验性) | HPA/VPA/Cluster Autoscaler/KEDA |
| 有状态 | 弱 | StatefulSet + Headless Service + CSI |
| 扩展 | 无 | CRD + Operator + Admission Webhook |

这里不展开 K8s 教程（本仓库 `Java/DevOps/Kubernetes` 板块有完整体系），只强调一个事实：**Java 业务开发者接触 K8s 的界面是 Deployment YAML + Service + Probe + 资源参数**，其余控制器机制属于平台工程。

# 6. 为什么生产基本选 K8s

1. **生态惯性**：CI/CD（Argo）、GitOps、可观测（Prometheus/OTel）、服务网格（Istio）、安全（OPA/Kyverno）、多租户、混合云（KubeVela/Cluster API）全部以 K8s 为第一公民，选其它方案意味着自建这些。
2. **厂商与云中立**：EKS/GKE/AKS/ACK 托管控制面同质可用，Swarm 无云厂商托管路线，Nomad 仅 HCP Nomad。
3. **人才市场**：招聘与资料供给决定长期维护成本。
4. **抽象可演进**：Pod 语义支持 sidecar/初始化容器/共享卷，容器运行时可换（containerd/CRI-O，甚至 Kata 安全容器），平台能力通过 CRD 扩展而不改上游。
5. **Swarm 现状客观描述**：Docker 公司 2023 年后将 Swarm 维护移交社区（Mirantis 赞助的 "Swarm Community Edition"），不再有商业路线图——新项目选型实际已无悬念。

本篇的定位据此明确：**它是"Docker 视角看向编排"的桥梁文档**——你已会单机 Docker，遇到多机需求时先分清"用现成平台（首选 K8s）、用 Swarm（存量/极简需求）、还是 Nomad（混部存量进程）"，而不是自研调度。Compose 到 K8s 的过渡路径见 `20_docker_vs_others.md` 的 CRI 生态部分。

# 7. 选型建议（经验法则）

1. 本地开发/集成测试：Compose 是首选。
2. 单机生产/小规模：Compose 也可用，但要明确 HA/发布/监控的补齐方式。
3. 多主机、需要 HA 与发布治理：优先考虑 Kubernetes（或企业既有平台）。
4. 存量非容器进程与容器混跑、团队无 K8s 经验：评估 Nomad。
5. 边缘/单节点集群、资源极受限：K3s/K0s 类轻量发行版（仍是 K8s API），比 Swarm 更保值。

# 8. 常见误区

1. 把编排等同于"写 YAML"：编排的本质是期望状态与控制循环。
2. 以为 healthcheck 就等于高可用：健康检查只是信号，高可用需要调度、自愈、流量切换与数据层策略。
3. 忽视有状态服务：数据库的 HA/备份恢复/一致性通常比无状态服务复杂得多，"把 MySQL 塞进 StatefulSet"不等于"MySQL 上 K8s 就绪"，多数团队的最优解仍是用云数据库/专用中间件平台。
4. 为"可能需要的规模"提前上 K8s：三五个内部工具服务、单机部署的产品交付，Compose 的维护成本远低于集群；规模化的收益只有在多副本、多节点、频繁发布出现时才成立。

# 9. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 为什么需要编排 | 多主机、HA、扩缩容、滚动升级、服务发现 |
| 编排系统核心机制 | 期望状态 + 控制器/调度 + 自愈（控制循环） |
| Compose 的边界 | 单机，多容器；不擅长集群治理 |
| Swarm vs K8s | 容器 Task vs Pod 抽象；内置简单 vs 生态可扩展；Swarm 已社区维护 |
| Nomad 差异点 | 非容器进程也能编, bin-packing, 混部存量场景 |
| K8s 为什么流行 | 生态、扩展性、声明式控制器模式、云厂商中立 |

# 10. 小结

编排的本质一句话：**用持续运行的控制循环把"期望状态"收敛为"实际状态"**，自愈、发布、伸缩、服务发现都是这个循环的衍生物。三大平台记住各自坐标——Swarm（最小可用、docker 原生、已交社区维护，存量参考）、Kubernetes（事实标准，复杂度换生态与可扩展抽象，生产默认项）、Nomad（容器与非容器混部的差异化选项）。Docker 用户理解 Swarm 的 stack/constraints/update_config+rollback 之后，迁移到 K8s 只是把 Service 拆成 Deployment+Service、把 VIP/IPVS 换成 kube-proxy、把"全量更新触发重启"换成热挂载 ConfigMap 与探针体系。本篇是 Docker 板块通向 K8s 板块的桥，再往前的工程沉淀收敛在最佳实践（`18_best_practices.md`）与性能（`21_performance.md`）。
