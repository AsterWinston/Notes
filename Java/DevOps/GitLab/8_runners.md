# 1. Runner 在体系中的位置

> **前置**：[第 7 篇 CI 进阶](7_gitlab_ci_advanced.md)
> **本篇目标**：搞清楚 Runner 的**归属层级、注册机制（含令牌现状）、执行器选型、config.toml 关键参数、并发与容量、生命周期运维、故障排查**
> **对应实际工作**：自建 CI 构建池、把流水线跑在自家内网/K8s 上

## 1.1 通信方向：Runner 只出不进

Job 创建后进入队列，等待 **Runner** 领取执行。Runner 是一个独立程序（`gitlab-runner`），与 GitLab 实例解耦 —— **它主动通过 HTTPS(gRPC) 访问 GitLab API**，因此：

```text
┌──────────┐   ①注册（令牌认证）        ┌──────────────┐
│ GitLab   │ ◄───────────────────────── │ Runner 进程   │
│  (CE/EE) │ ────────────────────────►  │              │
│          │   ②下发 Job（Runner 轮询/长 │  executor:    │
│          │     连接拉取，不是 GitLab    │  shell/docker/ │
│          │     主动连接 Runner）        │  kubernetes    │
│          │   ③回传日志与结果            └──────┬───────┘
└──────────┘                                    │ 本地创建构建环境
                                                ▼
                                     容器 / Pod / 宿主进程执行 script
```

**三条推论**：

1. **Runner 不需要暴露任何入站端口**（除非启用 debug terminal 的 session server），内网 Runner 可以放在只有出网权限的网段。
2. **Runner 与实例之间只需 443 可达**；`gitlab-runner` 所在机器的 DNS、代理、CA 证书问题会表现为"Job 永远 pending"，而不是报错。
3. **Job 的整个执行过程由 Runner 本地完成**：GitLab 挂了不影响正在跑的 Job 执行，但会中断状态回报（表现为 Job 卡在某个输出，最终 `worker_timeout`）。

## 1.2 四种归属层级：注册位置决定可用范围

| 类型 | 创建位置 | 可用范围 | 队列策略 | 典型场景 |
|------|----------|----------|----------|----------|
| **Instance Runner** | Admin Area → CI/CD → Runners | 本实例所有项目（可限制是否共享） | **Fair usage queue（公平队列）** | 自建统一构建池 |
| **Group Runner** | 群组 → Build → Runners | 组内（含子组）项目 | **FIFO** | 部门专属资源、GPU 池 |
| **Project Runner** | 项目 → Settings → CI/CD → Runners | 仅该项目 | **FIFO** | 特殊硬件、独占构建机 |
| **Instance Runner（shared=false）** | 同上 | 指定项目/群组（较新版本可配 `maximum timeout` / 项目共享） | FIFO | 折中方案 |

**为什么队列策略重要**：Instance Runner 用公平队列，**单个项目刷 500 条流水线不会饿死别人**；Group/Project Runner 用 FIFO，**一个项目刷屏就会独占全部槽位**。这正是"部门共享 Runner 被一个批量任务打满"的根因 —— 解法是给重活单独建 Project Runner，或用 `limit` 限并发。

**Project Runner 的锁定语义（易踩）**：Runner 用项目令牌注册的**第一个项目即成为 owner，不能解绑，只能删除 Runner**。要改归属必须新建 Runner。Group Runner 有 **Lock to current projects**（锁定后不能再被别组用），实例级 Runner 无此锁定。

## 1.3 Runner 对象 vs 令牌：配额挂在谁身上

Runner 的并发、超时、锁定、标签都是 **Runner 对象**的属性（存在实例 DB 里），令牌只是"这个进程证明自己是谁"的凭据。所以：

- **换令牌不会改变 Runner 的标签和配额**；
- **删 Runner 不会自动停止机器上的进程**（必须 `gitlab-runner unregister`，见第 9 节）；
- 同一台机器可以注册多个 Runner 对象（各自 `[[runners]]` 块），用于**不同标签/不同配额**。

---

# 2. 注册流程与令牌机制（重要，此处纠正多数旧教程）

## 2.1 令牌现状（必须按现行行为理解）

| 令牌 | 前缀 | 注册参数 | 现状 |
|------|------|----------|------|
| **Runner authentication token**（推荐） | `glrt-` | `--token` | 现行主路径，向导与非交互式都用它 |
| **Registration token**（旧） | 新版实例可见 `glrtr-` 前缀（旧版为长随机串） | `--registration-token` | **GitLab 17.0 起标记弃用（"no longer recommended"），计划 GitLab 20.0 移除**；管理员可关闭 |
| Agent token | `glagent-`（以 UI 显示为准） | 与 `gitlab-runner` 注册**无关** | Git 访问 / CI 访问 Agent 用的令牌，**不能当 `--token` 传给 gitlab-runner** |

> **常见错误来源**：很多 2023-2024 年的教程把 Runner 注册写成 `--agent-token "cggn_..."`，并声称"registration token 已不可用"。**两处都不对**：正确的令牌是 `glrt-` 开头的 runner authentication token、参数名是 `--token`；registration token 只是被弃用，**到 GitLab 20.0 才移除**，在大量现存实例上仍然可用（但管理员可以随时关掉）。Agent token 的当前前缀是 `glagent-`（源码 `Clusters::AgentToken::TOKEN_PREFIX`），早期教程里的 `cggn_`/`cggnat_` 写法都已过时 —— **别拿任何"看起来像令牌"的串去试注册**，从对应页面重新创建。

**判断你的实例走哪条路**（不要凭记忆）：

```bash
# 看实例是否还允许 registration token 注册（管理员）
glab api "instance/runner_settings" | jq '{
  enabled,
  max_timeout,
  minutes_timeout,
  allowed_gitlab_instance,
  enable_shared_runners,
  group_runners_enabled
}'
# 字段名随版本略有差异；先在页面上看 Settings → CI/CD → Runners 的开关，再对 API 字段
```

## 2.2 方式一：向导 UI 注册（最省事，推荐首次使用）

```text
目标层级页面（Admin Area → CI/CD → Runners，或 群组/项目 → Build → Runners）
  → Create instance/group/project runner
  → 填 Runner 名称、描述、tags、是否 Run untagged jobs、Access level（Not protected / Ref protected）
  → 提交后得到 **Runner authentication token（glrt-...）** 与一条完整的 gitlab-runner 命令
  → 在目标机器上执行该命令
```

向导的好处：**Runner 对象先建好，属性（标签、锁定、access level）全部由服务端确定**，注册命令只是把进程接上。避免"命令行传了 `--tag-list` 但被忽略"的问题（见 2.4）。

## 2.3 方式二：非交互式认证令牌注册（推荐用于自动化/镜像/Ansible）

```bash
# 1) 安装 Runner（Docker 方式，官方镜像）
docker run -d --name gitlab-runner --restart unless-stopped \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest

# 2) 用 runner authentication token 注册（glrt- 开头；参数是 --token）
docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com" \
  --token "glrt-XXXXXXXXXXXXXXXXXXXX" \
  --executor "docker" \
  --docker-image "alpine:3.20" \
  --description "docker-runner-01" \
  --docker-volumes "/cache" \
  --docker-volumes "/var/run/docker.sock:/var/run/docker.sock" \
  --template-config 'runners.docker.services_alias = "gitlab-helper"'
```

要点：

- **`--token` 传 runner authentication token**；这个流程会**保留向导里设置的属性**（标签、access level 等）。
- **`--template-config`** 可以在注册时注入 config.toml 片段（K8s 卷、复杂 Docker 参数），免去"注册后再手改配置文件"，适合 IaC：

```bash
  --template-config '
runners.docker.volumes = ["/cache", "/opt/ccache:/opt/ccache"]
runners.docker.shm_size = 1073741824
runners.docker.pull_policy = ["if-not-present"]
'
```

- 注册完立刻验证：

```bash
docker exec -it gitlab-runner gitlab-runner verify      # 逐条打印 VERIFIED / ERROR
docker exec -it gitlab-runner gitlab-runner list        # 看本机注册了哪些 Runner
```

## 2.4 方式三：legacy-compatible registration（registration token，仅在无法用前两者时）

```bash
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com" \
  --registration-token "<project/group registration token>" \
  --executor docker --docker-image alpine:3.20 \
  --tag-list "java,docker" \
  --run-untagged="false" \
  --locked="false" \
  --access-level="ref_protected" \
  --maximum-timeout=3600 \
  --maintenance-note="migrated from runner-03"
```

**这条路径最大的坑：`legacy-compatible registration process` 会忽略 `--locked`、`--access-level`、`--run-untagged`、`--maximum-timeout`、`--paused`、`--tag-list`、`--maintenance-note`。** 也就是说你以为设了标签和受保护级别，实际服务端属性是空的 —— Job 永远 pending 而 Runner 显示 online，就是这种"注册参数没生效"。

**排查方法**：注册后从服务端读回属性，逐项对照，而不是信命令行：

```bash
glab api "runners/$RUNNER_ID" | jq '{description,tag_list,run_untagged_jobs,access_level,locked,paused,online,status}'
```

> **纪律**：新注册的 Runner **一律走 authentication token**；把 `glrt-` 令牌写进密钥管理（Vault/SSM），不要写进 Git。已有 legacy Runner 记入"迁移清单"，在 GitLab 20.0 之前完成替换。

## 2.5 注册失败的报错原文

| 报错 | 含义 | 处理 |
|------|------|------|
| `410 Gone - runner registration disallowed` / `ERROR: Verifying runner... Registration is not allowed` | 管理员关闭了 Runner 注册 | 打开对应层级的注册开关，或让管理员预建 Runner 用 `glrt-` 令牌 |
| `ERROR: Registering runner... couldn't register runner: ... 403 Forbidden` / `Check registration token` | 令牌错、过期、或用在错误层级（项目令牌注册到组） | 确认令牌来源层级与 `--url` 指向同一实例 |
| `ERROR: Resolving gitlab-runner authentication token... 404 Not Found`（用 `--token` 时） | 传的不是 `glrt-` 令牌（例如误传 `glagent-` agent token、或令牌已被轮换/删除） | 从 Runner 向导页复制 authentication token |
| `ERROR: Job failed: ... could not connect to ... x509: certificate signed by unknown authority` | 内网 CA 未装 | 注册时加 `--tls-ca-file /etc/ssl/certs/internal-ca.pem`，或把 CA 装进系统信任库 |
| `ERROR: Job failed (system failure): ... 401 Unauthorized`（运行期） | 令牌被轮换/Runner 被删 | 见第 9.3 节轮换流程 |

---

# 3. config.toml：结构与关键参数

## 3.1 文件位置与"哪个文件生效"

| 安装方式 | 配置文件 |
|----------|----------|
| Linux 包（systemd） | `/etc/gitlab-runner/config.toml` |
| Docker 容器 | 容器内 `/etc/gitlab-runner/config.toml`（宿主机映射卷，如 `/srv/gitlab-runner/config`） |
| Windows 服务 | `C:\Program Files\GitLab-Runner\config.toml` |
| 任意方式 | 用 `--config <path>` 显式指定（多套 Runner 池隔离时用） |

```bash
gitlab-runner --config /etc/gitlab-runner/config.toml verify   # 验证所有 Runner 令牌
gitlab-runner list                                             # 本机已注册的 Runner
```

> **注意**：较新版本 Runner 把 **runner token 存放在独立的 credentials 文件**（默认与 config.toml 同目录的 `credentials`），`config.toml` 里可能看不到 `token =` 字段。**迁移机器时两个文件都要带走**，只拷 config.toml 会导致 Runner 全部掉线。版本差异请以 `gitlab-runner verify` 的输出与该机实际文件为准。

## 3.2 一份带注释的最小可用配置

```toml
concurrent = 10              # 全局：这台机器同时最多跑 10 个 Job（所有 Runner 共享）
check_interval = 0           # 轮询间隔（秒）；0 = 用默认值。多 Runner 时调大可减轻 API 压力
sentry_dsn = ""
log_level = "info"           # 排错时临时改 debug
output_limit = 8192          # 单位 KB：Job 日志上限（默认 4096 = 4MB），日志被截断改这里

[[runners]]
  name = "docker-runner-01"
  url = "https://gitlab.example.com"
  token = "glrt-XXXX"                     # 新版本可能改存 credentials 文件
  executor = "docker"
  limit = 4                               # 本 Runner 并发上限（0 = 用全局 concurrent）
  request_concurrency = 1                 # K8s 执行器调大它能提高建 Pod 吞吐
  output_limit = 8192
  builds_dir = "/builds"                  # 留空则用容器/宿主默认工作目录
  cache_dir = "/cache"
  environment = ["GIT_SSL_NO_VERIFY=false", "LANG=C.UTF-8"]
  git_strategy = "fetch"                  # clone | fetch | copy；大仓库用 fetch/copy 显著提速
  clone_url = ""                          # 覆盖 git 克隆地址（走内网镜像时用）
  pre_clone_script = ""                   # 常在 shell 执行器里做 ssh-agent / 代理准备
  pre_build_script = ""
  post_build_script = ""

  [runners.docker]
    image = "alpine:3.20"
    privileged = false                     # DIND 才需要 true；优先用其它方案（第 7.2 节）
    pull_policy = ["if-not-present"]       # 首次或更新镜像时临时改 ["always"]
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 1073741824                  # 1GB：Chromium/前端构建常因 /dev/shm 太小崩
    memory = "4g"
    cpus = "2"
    network_mtu = ""                        # 覆盖 MTU 用；跨主机 overlay 网络抖动时排查项
    wait_for_services_timeout = 60          # 服务容器等待上限；拉大镜像慢时报 "Service response timed out"
    helper_image = ""                       # 留空可省 helper 容器（会失去缓存/上传优化能力，谨慎）
    allowed_images = ["registry.example.com/*", "alpine:*"]   # 限制 Job 能用的镜像（安全边界）
    security_opt = ["seccomp=unconfined"]   # 仅在确有需要时开
    user = "git:git"

  [runners.cache]                            # 多 Runner 必须用远端缓存后端
    Type = "s3"
    Shared = true                            # Shared=true 才能跨 Runner 命中缓存
    [runners.cache.s3]
      server_address = "s3.example.com"
      access_key = ""                        # 建议用 IAM/环境变量，别写进文件
      secret_key = ""
      bucket_name = "ci-cache"
      location = "eu-central-1"
      insecure = false

[session_server]                             # 启用 debug terminal 才需要
  listen_address = "0.0.0.0:8093"
  advertise_address = "runner-01.example.com:8093"   # 必须是 GitLab 能连回的地址
  session_timeout = 1800
```

## 3.3 配置文件的权限纪律

```bash
sudo chown root:root /etc/gitlab-runner/config.toml
sudo chmod 600 /etc/gitlab-runner/config.toml
# 原因：config.toml / credentials 里含 runner token 与可能的 S3 key；
# 600 之外权限会让"能读文件的账号 = 能冒充你的 Runner 领 Job 并拿到 CI 变量"
sudo systemctl restart gitlab-runner
journalctl -u gitlab-runner -n 100 --no-pager
```

---

# 4. Executor 选型：Runner 怎么"起环境"

## 4.1 对照表

| Executor | 环境 | 速度 | 隔离性 | 运维成本 | 适合 |
|----------|------|------|--------|----------|------|
| **shell** | Runner 本机进程 | 极快 | **差**（共享环境、可污染宿主） | 低 | 内部低风险、需要直接操作宿主的部署脚本 |
| **docker** | 每 Job 一个容器 | 快 | 中 | 中 | **事实标准**，绝大多数团队 |
| **docker+machine** | 云端 VM 里跑 docker | 自动伸缩 | 好 | 高（依赖云 API） | AWS/GCP 弹性构建池 |
| **kubernetes** | 每 Job 一个 Pod（+helper Pod） | 较快 | 好 | 高（需 K8s 平台） | 已有 K8s 平台 |
| **ssh** | 远端机器 | 慢 | 差 | 低 | 特殊硬件、真实环境部署 |
| **virtualbox / parallels** | 本地 VM | 慢 | 好 | 高 | iOS/桌面、特定 OS 构建 |

## 4.2 shell 执行器的适用边界

```toml
[[runners]]
  executor = "shell"
  shell = "bash"            # Windows 用 "powershell"
  environment = ["PATH=/usr/local/bin:/usr/bin:/bin"]
```

- **适合**：部署类 Job（需要在跳板机上跑 `ssh`/`rsync`）、需要访问本地硬件/许可证的构建。
- **不适合**：编译依赖复杂的构建。shell 执行器**共享宿主环境**，"我本地能跑、CI 也能跑、另一个 Job 之后就跑不了了"的三连问题基本都是它 —— 因为**上一个 Job 改了宿主的 `~/.m2/settings.xml` 或装了包**。
- **硬要求**：`gitlab-runner` 用户必须能免密访问所需资源，且**不要用 root 跑 shell 执行器**。

## 4.3 docker executor 是主流

`.gitlab-ci.yml` 里每个 Job 的 `image:` 直接决定容器镜像，**构建环境与生产镜像天然统一**。代价：Runner 机器必须装 Docker，且要注意 Docker-in-Docker 的安全与性能（第 7 节）。

---

# 5. tags：把 Job 调度到对的 Runner

## 5.1 匹配规则

```yaml
deploy-prod:
  tags: [k8s, production]     # 只有注册时声明了这两个 tag 的 Runner 才可能接活
  script: [kubectl apply -f k8s/]
```

- 一个 Job 多个 tag = **AND**（Runner 必须全部具备）。
- Runner 侧 `run_untagged = true/false`：`false` 时不接无标签 Job。**生产建议 false**，防止随便一个公共 Runner 跑你的发布。
- **`access_level = ref_protected`（protected runner）**：只跑**受保护 ref** 上的 Job。发布类 Runner 应设成 protected，并确保只有运维有权限改它。
- 一个 Runner 可以同时挂到多个项目/群组（instance/group 级），标签是所有归属的并集。

## 5.2 "Why is this job pending?" 的完整判定顺序

```bash
# 1) 列出候选 Runner 及其标签/状态（比在页面上翻快）
glab api "runners?per_page=100&status=online" | jq -r '.[] | [.id,.description,(.tag_list|join(",")),.run_untagged_jobs,.access_level,.online] | @tsv'
# 2) 看这个 Job 需要哪些标签
glab api "projects/$CI_PROJECT_ID/jobs/$JOB_ID" | jq '{name,tags,status,failure_reason}'
# 3) 交叉核对：tags 是否全覆盖、run_untagged、ref 是否受保护、项目是否有权限用这个 Runner
```

| 现象 | 判定 |
|------|------|
| Job tags = `[k8s]`，在线 Runner 只有 `[docker]` | 标签不匹配（最常见） |
| Job 没写 tags，Runner `run_untagged = false` | 没人认领 |
| Runner 是 protected，Job 跑在非保护 ref | 拒绝调度 |
| Instance Runner 未共享给该项目 | 无权限使用 |
| Runner online 但 `limit` 已满 | 排队，不是失败；看 `queued duration` |
| Runner 状态 stale/offline | 进程挂了 / 出网被阻断 |

---

# 6. 并发与容量规划

## 6.1 三层并发参数（顺序不可颠倒）

```toml
concurrent = 10                 # ① 全局上限（一台机器一个 config.toml，所有 Runner 共享）

[[runners]]
  limit = 4                     # ② 单个 Runner 对象上限（0 = 不额外限制）
  request_concurrency = 1       # ③ 向 GitLab 并发请求 Job 的数量（K8s 执行器建议 >1）
```

- **`concurrent` 是硬顶**：调了 `limit` 但忘调 `concurrent`，多 Runner 之和会被 `concurrent` 卡住（表现为"两个 Runner 都空闲，但只有 Job 被一个接走"）。
- **`request_concurrency`** 对 K8s executor 影响明显：默认 1，Pod 创建慢时整条队列被串行化，**调 5-10 常能把排队时间砍一半**。
- K8s 侧还要看 **集群资源配额**（ResourceQuota/LimitRange）与 **调度器**，Runner 只是提交者。

## 6.2 容量估算与排队观测

**估算公式**：峰值并发 ≈ 触发频率（个/分钟） × 平均 Job 时长（分钟）。
例：平均 5 分钟一个 Job、每分钟 2 次触发 → 峰值 ~10 并发。一台 16C32G 起步比较从容；不够就加机器或改用 `docker+machine`/K8s 弹性。

```bash
# 是"没 Runner 可用"还是"Runner 忙"？看队列指标
curl -s localhost:9168/sidekiq | grep -E 'ci_(pending|build)'      # GitLab 侧（自建）
glab api "projects/:id/jobs?scope=pending&per_page=100" | jq -r '.[].name'
# 从 trace 判断：有 "Running with gitlab-runner" 说明已被认领，问题在执行；
# 只有 "Waiting for a runner to pick up this job..." 说明调度问题
```

**危险信号**：`docker` 执行器单机并发调过高会 **OOM 连锁**（一个 Job 触发 `Killed` → 表现为 `exit code 137` / `runner_system_failure`）。安全做法是给 `[runners.docker]` 设 `memory`/`cpus`，让 `concurrent × memory ≤ 物理内存的 70%`。

---

# 7. Docker executor 深入

## 7.1 挂载、缓存与工作目录

```toml
  [runners.docker]
    image = "maven:3.9-eclipse-temurin-17"
    volumes = [
      "/cache",                                       # 缓存目录（named volume）
      "/srv/ccache:/srv/ccache",                      # 共享 ccache
      "/var/run/docker.sock:/var/run/docker.sock"     # 见 7.2 的取舍
    ]
    shm_size = 1073741824
```

- **容器重建会丢掉容器内一切**（`/root/.m2`、`/usr/local/bin` 装的包都不保留）。要持久化必须 `volumes` 挂出来，或做进自定义镜像。
- **缓存路径要与 CI 里的 `cache:paths` 对齐**：`cache: [.m2/repository]` 是相对 `$CI_PROJECT_DIR`，走 GitLab 缓存后端；而 `[runners.cache]` 是**缓存上传到哪里**。两者搞混会导致"配了缓存但从不命中"。
- **`builds_dir`** 留空时容器内用 `/builds/<namespace>/<project>`；磁盘紧张可挂 tmpfs（注意 Job 数据落盘与缓存失败）。

## 7.2 Docker-in-Docker：三种方案的安全与性能取舍

| 方案 | 配置 | 隔离 | 性能 | 适用 |
|------|------|------|------|------|
| **挂 docker.sock**（`volumes` 挂 socket） | 无需 `privileged` | **无隔离**：Job 等价于宿主 root | 好（共享层缓存，快） | 完全受信的内部项目 |
| **特权 DIND**（`privileged = true` + `services: docker`） | 需 privileged | 弱 | 差（TLS 握手 + 独立层缓存） | 需要构建且**无法**用其它方案时 |
| **rootless / kaniko / buildah** | 不挂 socket、非特权 | 较好 | 中（无层缓存时慢） | 多租户、跑外部代码的集群 |

```yaml
# 方案 A：共享宿主 socket（最快，但等于交出宿主）
build-image:
  image: docker:27
  variables: { DOCKER_HOST: "unix:///var/run/docker.sock" }
  script:
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .

# 方案 B：TLS DIND（需 privileged=true）
build-image-tls:
  image: docker:27
  services: [{ name: docker:27-dind, alias: docker }]
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_HOST: "tcp://docker:2376"
    DOCKER_CERT_PATH: "/certs/client"
    DOCKER_TLS_VERIFY: "1"
  script: [ docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" . ]
```

**必须知道的 DIND 症状**：`Cannot connect to the Docker daemon at tcp://docker:2375. Is the docker daemon running?` —— 99% 是 `DOCKER_HOST`/`DOCKER_TLS_CERTDIR` 与实际监听方式不一致，或 `[runners.docker] privileged` 没开导致 dind 起不来（提示 `dockerd-rootless` / `iptables` 权限错误）。

## 7.3 私有镜像与拉取策略

```toml
  [runners.docker]
    pull_policy = ["if-not-present"]   # 排错时临时改 ["always"]，"我改了镜像但 CI 还是旧的"就是它
    allowed_images = ["registry.example.com/*", "docker.io/library/*"]
```

```bash
# 让 Runner 能拉私有镜像：三种做法
# ① 宿主 docker login（简单，但凭据在宿主持久存在）
docker login registry.example.com -u "$USER" -p "$PASS"
# ② CI 变量 + 脚本内登录（推荐，凭据随 Job 生命周期）
echo "$HARBOR_PASSWORD" | docker login "$CI_REGISTRY" -u "$CI_DEPLOY_USER" --password-stdin
# ③ 用 CI_JOB_TOKEN 拉 GitLab 内置 registry（需在目标项目授予 Job Token 入站权限）
docker login -u gitlab-ci-token -p "$CI_JOB_TOKEN" "$CI_REGISTRY"
```

> **`pull_policy = if-not-present` 的副作用**：镜像 tag 不变但内容变了（例如一直用 `:latest`），Job 会用旧镜像，症状是"CI 里的工具版本比本地老"。团队规范：**镜像必须用不可变 tag 或 digest**（第 15 篇）。

---

# 8. Kubernetes executor：云原生团队的主流做法

## 8.1 注册与关键参数

```bash
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com" \
  --token "glrt-XXXXXXXXXXXXXXXXXXXX" \
  --executor "kubernetes" \
  --kubernetes-namespace "ci" \
  --kubernetes-image "gitlab-runner-helper:x86_64-latest" \
  --kubernetes-image-pull-secrets "harbor-cred" \
  --kubernetes-service-account "gitlab-runner" \
  --kubernetes-max-requests-per-transaction 5 \
  --kubernetes-cpu-request-limit "500m" \
  --kubernetes-memory-request-limit "1Gi"
```

```toml
[[runners]]
  executor = "kubernetes"
  request_concurrency = 10            # 建 Pod 吞吐，几乎总是要调大
  [runners.kubernetes]
    namespace = "ci"
    namespace_overwrite_allowed = "ci-.*"        # 允许 CI 里覆盖命名空间的模式
    service_account = "gitlab-runner"
    image = "alpine:3.20"
    helper_image = "registry.example.com/gitlab-runner-helper:x86_64-17"   # 固定版本
    image_pull_secrets = "harbor-cred"
    cpu_request = "500m"
    memory_request = "1Gi"
    cpu_limit = "2"
    memory_limit = "4Gi"
    ephemeral_storage_limit = "20Gi"
    termination_grace_period_seconds = 30
    poll_interval = 5
    node_selector = "workload=ci"
    pod_labels = { team = "platform" }
    pod_annotations = { "k8s.io/something" = "x" }
    allow_privilege_escalation = false
    privileged = false
    [runners.kubernetes.volumes.pvc]
      name = "m2-cache"
      mount_path = "/home/gradle/.gradle"
    [runners.kubernetes.volumes.empty_dir]
      name = "work"
      mount_path = "/builds"
      medium = "Memory"
```

## 8.2 权限模型：Runner 需要什么

```bash
# ServiceAccount 最小权限：在指定 namespace 内管理 Pod/Secret/Service/...（RBAC 由平台预置）
kubectl -n ci auth can-i create pods --as=system:serviceaccount:ci:gitlab-runner
kubectl -n ci auth can-i create secrets --as=system:serviceaccount:ci:gitlab-runner
kubectl -n ci auth can-i list pods --as=system:serviceaccount:ci:gitlab-runner
```

- Runner 只应拿到**指定 namespace** 的权限，**不要用 cluster-admin**（否则一个 CI Job 可以读遍全集群 Secret）。
- 需要 `gitlab-runner` 有 **`pods/exec`** 才能用 debug terminal；需要 **`services`** 才能支持 `services:` 的 K8s 实现（较新版本用 Pod 内多容器/或 headless service，取决于实例与 Runner 版本）。
- **证书/token 来源**：优先用 `~/.kube/config` 里的 context（`--kubernetes-namespace` + kubeconfig），或在 config.toml 里显式 `ca_file`/`client_key`/`client_cert`/`token`。**不要用过期的 SA token** —— 新版 K8s 的 SA token 不再自动长期有效，需显式创建 `Secret` 类型 token。

> **版本兼容提醒**：旧键 `namespace_per_job` 与 `rbac:`（由 Runner 自动建 SA/RoleBinding）在新版本 Runner 中已被替换/移除，改为 `create_namespace_per_build` 系列能力 + **由平台预置 RBAC**。**跨大版本升级 Runner 前必读对应 release notes**，并用 `gitlab-runner verify` + 单条灰度流水线验证。

## 8.3 pod 片段与 sidecar（K8s 独有能力）

```yaml
e2e:
  image: maven:3.9-eclipse-temurin-17
  services:
    - name: mysql:8.0
      alias: db
  script: [ ./ci/e2e.sh ]

# 直接声明多容器（K8s executor 专属；Docker executor 会被忽略）
browser-test:
  script: [ ./ci/e2e-chrome.sh ]
  parallel:
    matrix:
      - BROWSER: [chrome, firefox]
```

```yaml
# 需要自定义 Pod 时：用 services + 挂载，或用 Runner 级模板注入
# （--template-config 'runners.kubernetes.pod_labels = {...}'，复杂 Pod 片段建议放 Runner 模板而不是 CI YAML）
```

## 8.4 K8s executor 的高频坑

| 现象 | 根因 | 处理 |
|------|------|------|
| Job 一直 `Waiting for pod to be running` | 集群资源不足 / Pending Pod | `kubectl -n ci get pods` + `kubectl describe pod` 看 FailedScheduling |
| `ImagePullBackOff` | 缺 `image_pull_secrets` 或镜像名不对 | 在 Runner 配置里加 pull secret，验证 `kubectl -n ci get secret` |
| `ERROR: Job failed: ... 403 Forbidden ... cannot create` | SA 权限不足 / namespace 限制 | `kubectl auth can-i` 逐项核对 |
| CI Pod 把业务 Pod 挤下线 | CI 与业务共节点池 | **分节点池 + 污点/标签隔离**（`node_selector`） |
| helper Pod 拉不到（版本对不上） | `helper_image` 用了 `:latest` 且被缓存 | 固定 helper_image 到 Runner 版本对应的 tag |
| API Server 被 Runner 打爆（429） | `request_concurrency` 高 + `poll_interval` 小 | 调大 `api_cache_size`、降 `poll_interval` 的激进程度，或分批 Runner |
| `Pod is stuck in Terminating` | `termination_grace_period_seconds` 与业务优雅停机冲突 | 对齐 K8s `terminationGracePeriodSeconds` 与应用 |

---

# 9. 状态、生命周期与运维

## 9.1 四种状态与其真实含义

| 状态 | 判定 | 处理 |
|------|------|------|
| `online` | 距最后一次联系 **< 2 小时** | 正常 |
| `offline` | 距最后一次联系 **> 2 小时** | 进程/网络/时钟问题 |
| `stale` | 距最后一次联系 **> 7 天**（且曾上线） | 僵尸 Runner，可删除；UI 提示 "This runner is stale" |
| `never contacted` | 注册后从未连接 | 令牌错或根本没启动进程 |

```bash
# Runner 侧自查
journalctl -u gitlab-runner --since -10m | grep -Ei 'error|verify|poll'
gitlab-runner verify
gitlab-runner list
# GitLab 侧读属性（含 online/status/last_contact/upgrade 提示）
glab api "runners/$RUNNER_ID" | jq '{online,status,last_contact,version,revision,platform,health,run_untagged_jobs,tag_list}'
```

> UI 出现 **Upgrade available / Upgrade recommended** 说明 Runner 版本明显落后于实例，个别关键字会被拒绝（"Job 定义里出现 `unknown keys` 但 GitLab 侧 lint 通过" → 很可能是 **Runner 版本太旧**不支持该关键字）。

## 9.2 维护动作：暂停、维护备注、升级

```bash
# 维护前：先暂停接新 Job（在跑的不受影响），并留一条备注让同事看到原因
glab api --method PUT "runners/$RUNNER_ID" -f paused=true -f maintenance_note="2026-11-20 02:00-03:00 内核升级"
# 等 Job 排空
glab api "runners/$RUNNER_ID/jobs?status=running&per_page=100" | jq 'length'
# 升级二进制/镜像
sudo apt-get install --only-upgrade gitlab-runner   # 或 docker pull gitlab/gitlab-runner:<新 tag>
sudo systemctl restart gitlab-runner
gitlab-runner verify
glab api --method PUT "runners/$RUNNER_ID" -f paused=false
```

- **`maintenance_note` 上限 255 字符**，超长会被拒绝；它显示在 CI 页面 Runner 详情里，是给"Job 排队很久"最省事的解释。
- **`paused` 与 `run_untagged` 是两个不同开关**：暂停只影响调度，不影响标签。
- **升级顺序：Runner 不低于 GitLab 实例的小版本太多**，且 K8s 集群 Runner 应灰度（先升一台带专属标签的，跑几条流水线再全量）。

## 9.3 删除、卸载与令牌轮换（顺序不能反）

```bash
# ⚠️ 只在 GitLab 里删 Runner 对象，机器上的进程仍会带着旧令牌轮询，日志刷 401/403
# 正确顺序：先停调度 → 本地 unregister → 再删服务端对象
glab api --method PUT "runners/$RUNNER_ID" -f paused=true
sudo gitlab-runner unregister --url "https://gitlab.example.com" --token "glrt-XXXX"
# 或按名字：sudo gitlab-runner unregister --name "docker-runner-01"
sudo gitlab-runner stop && sudo gitlab-runner uninstall && sudo rm -f /etc/gitlab-runner/config.toml

# 令牌轮换（Runner 对象保留、属性不变）：
# 1) 服务端 Recreate authentication token（Runner 详情页；按钮文案随版本略有差异）
# 2) 更新本地凭据并重启
sudo gitlab-runner unregister --token "<旧 glrt-令牌>"
sudo gitlab-runner register --non-interactive --url "https://gitlab.example.com" --token "<新 glrt-令牌>" --executor docker --docker-image alpine:3.20
sudo gitlab-runner verify && sudo systemctl restart gitlab-runner
```

> **Stale group runner cleanup（自动清理长期未联系的组 Runner）是 Ultimate 能力**；其他订阅需自己写脚本调 API 清理，见第 16 篇。

---

# 10. 安全实践清单

| 风险 | 对策 |
|------|------|
| 挂 `docker.sock` = 把宿主 root 交给任意 Job | 仅内部受信项目使用；跑外部代码改用 K8s / VM 隔离，或用 rootless/kaniko |
| `privileged = true` | 默认禁用；需要构建镜像优先非特权方案 |
| 公共 Runner 跑你的生产发布 | 生产 Job 显式 `tags:`，发布 Runner 设 **`access_level = ref_protected`** 且 `run_untagged = false` |
| Fork MR 拿到内网凭据 | 外部贡献者的流水线**不要**调度到内网 Runner；用独立的外部 Runner 池 + 限制 `allowed_images` |
| 令牌泄露 | `glrt-` 令牌入密钥管理、按 Runner 独立、定期轮换；`config.toml`/`credentials` 权限 600 |
| 密钥进日志 | CI 变量 Masked（注意字符集约束，第 6 篇）；`after_script` 里不要 `env` 全量打印 |
| 构建环境被投毒（Runner 被劫持） | Runner 用最小化专用 VM/节点；出网白名单只允许 GitLab 与依赖仓库 |
| Job 能改 Runner 配置 | Runner 容器/机器禁止 SSH 给 CI 身份；不挂载 `/`、`/etc`、docker socket 之外的宿主敏感目录 |
| debug terminal 成为后门 | 不默认开 `[session_server]`；开启时 `advertise_address` 只暴露内网并加访问控制 |

---

# 11. 排错速查

| 现象 | 排查点 |
|------|--------|
| Job 永远 pending | 第 5.2 节判定顺序：tags → `run_untagged` → protected → 归属权限 → `limit/concurrent` 满 |
| 注册时 `410 Gone` / `Registration is not allowed` | 管理员关闭了注册；改用向导 + `glrt-` 令牌（第 2.5 节） |
| 注册"成功"但标签/access level 是空的 | 走了 legacy 流程，命令行参数被忽略（第 2.4 节）→ 从服务端读回属性核对 |
| trace 停在 `Running with gitlab-runner ...` 之前 | Runner 未认领；`gitlab-runner verify` + `journalctl` |
| `executing the build` 卡住 | 出网被防火墙/代理阻断、DNS 解析慢、时钟偏差导致 TLS 校验失败 |
| Job 日志尾部被截断 | `output_limit` 太小（第 3.2 节） |
| `Cannot connect to the Docker daemon` | `DOCKER_HOST`/`TLS_CERTDIR` 与实际不符，或 `privileged` 未开（第 7.2 节） |
| CI 用了旧镜像/旧工具版本 | `pull_policy = if-not-present` + `:latest` tag（第 7.3 节） |
| `exit code 137` / `Killed` | OOM：`concurrent × 内存` 超物理内存（第 6.2 节） |
| 私有镜像拉不到（Docker） | 宿主 `docker login` / CI 内 `--password-stdin` |
| 私有镜像拉不到（K8s） | `image_pull_secrets` 缺失或命名空间不对 |
| `Service response timed out`（services 连不上） | `wait_for_services_timeout` 太短，或服务镜像拉取慢 |
| 缓存 403 / 命中率 0 | `[runners.cache]` 后端凭据与 `Shared = true`；多 Runner 用本地缓存必然低命中（第 3.2 节） |
| 删了 Runner 但日志刷 401 | 忘记本地 `gitlab-runner unregister`（第 9.3 节） |
| 新关键字报 `unknown keys` 但 lint 通过 | **Runner 版本过旧**（第 9.1 节） |

**一条通用采集命令**（把结果贴进工单，比描述"CI 挂了"有用得多）：

```bash
{ gitlab-runner --version; gitlab-runner verify; \
  grep -E 'concurrent|check_interval|output_limit|executor|tags|limit' /etc/gitlab-runner/config.toml; \
  journalctl -u gitlab-runner --since "-30m" --no-pager | tail -80; \
  df -h /var/lib/docker /builds; free -g; timedatectl status | head -8; } > /tmp/runner-triage.txt 2>&1
```

---

# 12. 小结

| 要点 | 一句话 |
|------|--------|
| 本质 | Runner = 独立进程，**只出不进**地通过 HTTPS/gRPC 领 Job，并按 executor 起环境 |
| 归属 | Instance（公平队列）/ Group / Project（FIFO）三种作用域；**Project Runner 首个项目即 owner，不能解绑** |
| 注册 | **现行主路径是 runner authentication token（`glrt-`）+ `--token`**；registration token 仅弃用未删除（**20.0 才移除**），legacy 流程会**忽略**标签/锁定/access level 等参数 |
| 令牌混淆 | `glrt-` 才是 Runner authentication token；`glagent-` 是 Agent token，**不能**用于 `gitlab-runner register` |
| 配置 | `concurrent`（全局）> `limit`（Runner）> `request_concurrency`（K8s 吞吐）；`output_limit` 决定日志是否截断；新版本 token 可能存独立 credentials 文件 |
| executor | docker 是主流（注意 socket vs 特权 vs rootless 的取舍），K8s 是云原生方案（RBAC 最小化 + CI 独立节点池），shell 慎用 |
| 调度 | tags 全匹配 + `run_untagged=false` + protected runner 三重收紧，生产发布才安全 |
| 生命周期 | 状态四档（online<2h / offline>2h / stale>7d / never contacted）；维护用 `paused` + `maintenance_note`；**删除必须先本地 unregister** |
| 安全 | 特权与 docker.sock 是最大攻击面；**外部代码绝不进受信 Runner** |

**自检清单**：

- [ ] 我能写出用 `glrt-` 令牌的注册命令，并说明 `--registration-token` 与 `--agent-token` 各自的正确含义？
- [ ] 我知道 legacy 注册流程会忽略哪些参数，并知道注册后要从服务端读回属性核对？
- [ ] 我能按"tags → run_untagged → protected → 归属 → 并发满"的顺序解释一个 pending Job？
- [ ] 我能说清 `concurrent`、`limit`、`request_concurrency` 分别卡在哪个环节？
- [ ] 我知道删除 Runner 的正确顺序，以及为什么只删服务端对象会留下 401 日志？

> Runner 解决"在哪跑"，下一篇解决"跑完去哪"：[第 9 篇 Environments 与部署编排](9_environments_and_deployment.md)。
