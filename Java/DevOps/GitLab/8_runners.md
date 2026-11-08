# 1. Runner 在体系中的位置

Job 创建后进入队列，等待**Runner**领取执行。Runner 是一个独立程序（`gitlab-runner`），与 GitLab 实例解耦——它只通过 HTTPS API 与 GitLab 通信（轮询或 gRPC 请求分配），不依赖内网直连 GitLab 的 SSH 权限。

```text
┌──────────┐   ①注册(Agent Token)    ┌──────────────┐
│ GitLab   │ ◄────────────────────── │ Runner 进程   │
│  (CE/EE) │  ②轮询/领取 Job          │ (shell/docker/ │
└──────────┘  ③回传日志与结果          │  kubernetes)  │
                                     └──────┬───────┘
                                            │ 在本地创建构建环境
                                            ▼
                                   容器/Pod/宿主进程执行 script
```

三种归属层级（注册时决定）：

| 类型 | 可用范围 | 典型场景 |
|------|---------|---------|
| **Instance Runner** | 本实例所有项目（可限共享） | 自建统一构建池 |
| **Group Runner** | 群组内项目 | 部门专属资源、GPU 池 |
| **Project Runner** | 单个项目 | 特殊硬件/独占构建机 |

# 2. 注册流程与新令牌机制（重要）

16.x 之前用 registration token + shell 交互注册；**现在统一为"先建 Agent，再认证"**：

```bash
# 1) 在 GitLab 建一个 CI/CD Agent（Admin 界面或 API），得到 agent token: cggn_...
# 2) 安装 Runner（Docker 方式，官方镜像）
docker run -d --name gitlab-runner --restart unless-stopped \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest

# 3) 用 agent 认证方式配置并启动
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com" \
  --agent-token "cggn_xxxx" \
  --executor "docker" \
  --docker-image "alpine:3.20" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock"
```

> **注意**：旧文档/教程里大量出现 `--registration-token`，那是已废弃机制（17.0 起新装 Runner 无法再用）。看到两种写法时以 **agent token** 为准。Runner 的并发与资源配额按 Runner 对象而非令牌管理。

# 3. Executor 选型：Runner 怎么"起环境"

Executor 决定 Job 的落地形态，注册时选定：

| Executor | 环境 | 速度 | 隔离性 | 适合 |
|----------|------|------|--------|------|
| **shell** | Runner 本机进程 | 极快 | 差（共享环境） | 内部低风险、脚本简单 |
| **docker** | 每 Job 一个容器 | 快 | 中 | **事实标准**，绝大多数团队 |
| **docker+machine** | 云端 VM 里跑 docker | 自动伸缩 | 好 | AWS/GCP 弹性构建池 |
| **kubernetes** | 每 Job 一个 Pod | 较快 | 好 | 已有 K8s 平台 |
| **ssh** | 远端机器 | 慢 | 差 | 特殊硬件部署 |

**docker executor 是主流**：`.gitlab-ci.yml` 里每个 job 的 `image:` 直接决定容器镜像，构建环境与生产镜像天然统一。代价是 Runner 机器必须装 Docker，且要注意 Docker-in-Docker 的安全与性能。

# 4. tags：把 job 调度到对的 Runner

GitLab 没有"指定哪台机器"的语法，调度靠**标签匹配**：

```yaml
deploy-prod:
  tags: [k8s, production]     # 只有注册时声明了这两个 tag 的 Runner 才会接活
  script: kubectl apply ...
```

- Runner 注册时配 `run_untagged = true/false`：false 时不接无 tag 的 job，**生产建议 false**，防止随便一个公共 Runner 跑你的发布。
- 一个 job 多个 tag = AND（Runner 必须全有）；job 不写 tags = 匹配所有"允许无标签"的 Runner。
- "Why is this job pending?" 八成答案是 tags 不匹配或 Runner 离线。

# 5. 并发与容量规划

`config.toml` 关键参数：

```toml
concurrent = 10          # 全局并发 Job 上限（按机器 CPU/内存调）

[[runners]]
  name = "docker-runner-01"
  limit = 4              # 单 Runner 并发上限
  [runners.docker]
    privileged = false            # DinD 需要 true，能用 sock 挂载就别特权
    pull_policy = "if-not-present" # 本地缓存基础镜像，提速明显
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    helper_image = ""              # 留空可省 helper 容器拉取
```

**容量估算**：Java 构建典型峰值并发 = 提交频率 × 平均 Job 时长 ÷ 60。例如平均 5 分钟一个 Job、每分钟 2 个触发 → 峰值 ~10 并发，一台 16C32G 起步比较从容；不够就加 Runner 机器或用 docker+machine 弹性扩。

# 6. K8s Executor：云原生团队的主流做法

Runner 不直接建容器，而是向 K8s 集群提交 Job Pod：

```bash
gitlab-runner register \
  --non-interactive --url "https://gitlab.example.com" \
  --agent-token "cggn_xxxx" \
  --executor "kubernetes" \
  --kubernetes-namespace "ci" \
  --kubernetes-image-pull-secrets "harbor-cred"
```

- 天然与集群资源配额、节点弹性伸缩（Cluster Autoscaler）结合，**构建高峰自动加节点**，解决自建 Runner 容量焦虑。
- gitlab-ci 里还能用 `pod` 片段声明多容器（sidecar：`service` 之外挂 ssh 隧道、浏览器测试环境）。
- 需要 ServiceAccount 具备创建 Pod 的 RBAC；Runner 所在命名域与构建命名域分离更安全。

> **行业实践**：自建 GitLab 配 K8s Runner 时，**CI Pod 与业务 Pod 分节点池**（污点/标签隔离），否则一次集成测试风暴会挤爆业务服务。

# 7. 安全实践清单

| 风险 | 对策 |
|------|------|
| Runner 挂载 docker.sock 等于把宿主 root 交给任意 job | 仅内部受信项目使用；对开源 fork 场景改用 K8s/VM 隔离 |
| fork MR 默认不跑流水线（无 token） | 需要外部贡献触发时，走"批准流水线"安全流程 |
| privileged 容器 | 禁用；用 kaniko/buildah/buildkit-rootless 替代 DinD |
| 构建环境被投毒（被劫持的 Runner） | Agent token 短生命周期、Runner 专用最小化 VM |
| 密钥泄露到日志 | CI 变量 Masked；`--hide-column` 防 Sidekiq 日志泄 token |

# 8. 排错速查

| 现象 | 排查点 |
|------|--------|
| job 永远 pending | Runner 离线？tags 不匹配？`run_untagged`？并发满？ |
| `executing the build` 卡住 | Runner 与实例网络/时钟问题；看 Runner 侧 `gitlab-runner list`、日志 |
| job 反复被 `stuck` 取消 | 同 Runner 连续超时，`gitlab-ctl tail` + 检查 gitaly/runner 资源 |
| docker executor 拉不到私有镜像 | Runner 机器 `docker login` 或配 `image_pull_secrets` |
| 缓存丢失/不命中 | cache 落在非项目目录、Runner 异构、policy 为 pull-only |

# 9. 小结

| 要点 | 一句话 |
|------|--------|
| 本质 | Runner = 独立进程 + 领取 Job 队列 + 按 executor 起环境 |
| 注册 | 新机制是 Agent Token，旧 registration token 已废弃 |
| executor | docker 为主流，K8s 为云原生方案，shell 慎用 |
| 调度 | tags 匹配 + run_untagged 收紧 |
| 容量 | 并发数按提交频率×Job 时长估算，弹性交给 docker+machine/K8s |
| 安全 | 特权与 docker.sock 是最大攻击面，外部代码绝不进受信 Runner |

Runner 解决"在哪跑"，下一篇解决"跑完去哪"：Environments、Review Apps 与部署编排。
