# 1. 为什么要了解架构

GitLab 不是"一个 Rails 应用"那么简单：Omnibus 装的单机实例里同时跑着 **十来个进程**。了解架构的目的不是面试八股，而是为了解释使用者日常会遇到的现象——

- 为什么 push 大仓库这么慢？（Gitaly）
- 为什么页面偶尔转圈、重启后要先 warm 一下？（Puma worker）
- 为什么流水线状态变了但 Issue 上的合并按钮没刷新？（Sidekiq 异步队列积压）
- 为什么 GitLab 吃内存？（Rails 单体的固有代价）

一句话架构观：**GitLab 是"元数据 + Git 数据 + 异步任务"三条线的组合**，元数据在 PostgreSQL，Git 数据在 Gitaly，异步任务由 Sidekiq 消化。

## 1.1 官方的分层视角

官方架构文档给每个组件标了"层"：Gitaly、PostgreSQL、Redis 属于 **Core Service (Data)**——数据的唯一持有者；Workhorse、GitLab Shell、Puma、Sidekiq 属于 **Core Service (Processor)**——流量的处理者。这个分层直接解释了扩容策略：**Processor 层可以多实例横向加，Data 层才是高可用真正要攻克的**（复制、共识、故障转移都发生在这一层）。

## 1.2 两种用户接口

官方开篇就把系统切成两个"门面"，后面所有请求路径分析都建立在这条分界上：

- **Web HTTP**：浏览器与 REST/GraphQL API —— 走 Rails；
- **Git HTTP/SSH**：push/pull/fetch —— Rails 只做鉴权决策，字节搬运归 Workhorse/Gitaly。

有些进程两头都用（Gitaly、PG、Redis），有些只属于一头（Puma 属于 Web，GitLab Shell 属于 Git/SSH）。排障先问"这是哪条接口的请求"，能砍掉一半怀疑面。

# 2. 组件全景

```text
                          浏览器 / git CLI / API 客户端
                                    │
                          ┌─────────▼─────────┐
                          │     NGINX         │  TLS 终结、静态文件、反向代理
                          └─────────┬─────────┘
                                    │
                          ┌─────────▼─────────┐
                          │     Workhorse     │  大文件守门员：LFS/附件/大 push
                          └─────────┬─────────┘
                 ┌──────────────────┼──────────────────┐
                 │ (Git RPC)        │ (HTTP 代理)      │
           ┌─────▼─────┐      ┌─────▼─────┐           │
           │  Gitaly   │      │   Rails   │◄──────────┘
           │ Git 服务   │      │  (Puma)   │  业务逻辑/API/UI
           └─────┬─────┘      └─────┬─────┘
                 │                  │ 读写
           ┌─────▼─────┐      ┌─────▼────────┐   发布/订阅
           │ Praefect  │      │ PostgreSQL   │◄──────┐
           │ (集群可选) │      │  元数据库     │       │
           └───────────┘      └──────────────┘   ┌───┴────┐
                                                  │ Redis  │ 缓存/队列/会话
               Rails ──入队──► Sidekiq ◄──消费────┘
               （异步任务执行者：邮件、Webhook、CI 调度、清理）
```

| 组件 | 进程类型 | 职责 | 崩了会怎样 |
|------|---------|------|-----------|
| NGINX | C/Nginx | 入口、TLS、限速 | 全站不可访问 |
| Workhorse | Go | 智能反向代理：接 LFS 上传、校验后交给 Rails；流式转发 git 请求 | 上传/克隆失败 |
| Rails (Puma) | Ruby | 页面、REST API、业务规则 | 页面不可用，git SSH 仍可能活 |
| Gitaly | Go | 所有 Git 操作的唯一入口（收发 push、算 diff、读 blob） | 仓库操作全挂 |
| PostgreSQL | C | 用户/项目/Issue/MR/流水线等一切元数据 | 全站瘫痪（最核心） |
| Redis | C | 缓存、会话、Sidekiq 队列、实时事件 | 登录态丢失、任务停摆 |
| Sidekiq | Ruby | 异步任务：发邮件、跑 Webhook、CI 状态传播、GC | 功能延迟而非立即故障 |
| GitLab Shell | Go | SSH 版 git 会话处理 + authorized_keys 管理 | SSH 克隆失败，Web 无恙 |
| Prometheus/Grafana | 可选 | 包安装默认捆绑并启用监控栈 | 无监控，不影响业务 |

## 2.1 官方组件清单的安装状态

包安装（Omnibus）默认装好的：NGINX、Workhorse、Shell、Puma/Rails、Sidekiq、Gitaly、PG、Redis、GitLab Exporter、Prometheus、Grafana。**需要额外配置才有的**：Pages、GitLab agent (KAS)、Alertmanager、Geo、Consul（EE，用于数据库节点发现/故障转移）、对象存储。**需要额外部署的**：Elasticsearch/OpenSearch（EE，高级搜索）、Zoekt（精确代码搜索，独立服务）、Sentry/Jaeger（错误与追踪）、外部 Runner、AI Gateway、Secrets Manager（OpenBao，独立 K8s 服务 + 专用 PG）。这张"默认/配置/外加"三档表就是回答"我这台实例少了什么"的对照表。

## 2.2 用进程读回真相

任何时刻，进程列表是比架构图更诚实的答案：

```bash
sudo gitlab-ctl status                       # Omnibus：托管的全部服务与运行时长
ps aux | grep '^git'                         # 源码安装的官方检查法：git 用户下的 puma/sidekiq
sudo gitlab-ctl restart gitaly               # 组件粒度操作，排障时别 restart 全家
```

组件集合随版本与启用的功能而变（比如 Pages、KAS、Registry 是独立进程），所以给同事讲"GitLab 有哪些进程"时，现场跑一条 `gitlab-ctl status` 比背图可靠。

# 3. 一次请求的旅程

**Web/API 请求（如合并 MR）**：

```text
浏览器 → NGINX → Workhorse(判断该去 Rails 还是别处，给 Puma 减负) → Puma(Rails)
       → 写 PostgreSQL(MR 状态) → 发事件给 Redis
       → Sidekiq 消费: 发邮件通知、触发 Webhook、更新流水线
```

官方对这条路径的拆解就四步：NGINX 是第一道反代，**Workhorse 决定"这个请求要不要惊动 Rails"**，Puma 处理业务，数据落在 PG/Gitaly/Redis。"转圈很久"的页面请求，八成卡在 Puma 队列或它下游的某次慢查询。

**git push（HTTP）**：官方时序把职责切得很干净（以 fetch 为例，push 换成 `git-receive-pack` 与 `PostReceivePack`）：

```text
git CLI → NGINX → Workhorse: GET /info/refs?service=git-receive-pack
       → Workhorse 回头问 Rails 内部 API（鉴权 preflight）→ Rails 答 git_http_ok
       → Workhorse → Gitaly: SmartHTTPService.PostReceivePack
       → Gitaly 派生 git receive-pack（服务端）收发 packfile、跑钩子
       → Rails(钩子回调: 更新 MR、触发 Pipeline) → Sidekiq(后续异步)
```

三个要点：鉴权永远由 Rails 裁决（Workhorse 只执行它的判断）；**字节流从头到尾不经过 Rails 进程**；Gitaly 才是真正 fork `git` 二进制的地方。

**git clone/pull（HTTP）**：同构——`info-refs?service=git-upload-pack` 与 `POST /git-upload-pack`，Workhorse 转发给 Gitaly 的 `SmartHTTPService.PostUploadPack`。**大流量不过 Rails**，这是 Workhorse 存在的意义，官方定义它就是个 "smart reverse proxy"，设计目标明确写着"缓解 Puma 压力"。

**git 请求（SSH）**——和直觉差别最大的一条路径：

```text
git CLI → sshd（唯一说 SSH 的组件）→ 全部以 git 系统用户身份落地，
       用客户端公钥区分 GitLab 用户：
  [快速查key开启] AuthorizedKeysCommand=gitlab-shell-authorized-keys-check
       → 问 Rails 内部 API /authorized_keys → 得到强制命令 "gitlab-shell upload-pack key_id=N"
       → gitlab-shell → 问 Rails /internal/api/allowed（动作级鉴权）→ 拿到 Gitaly 地址
       → Gitaly SSHService.SSHUploadPack → 派生 git upload-pack ↔ 双向流
```

细节三条：没有任何 GitLab 组件"说 SSH"，sshd 终止连接后才轮到 Shell/Rails；没开 fast SSH key lookup 时，`~git/.ssh/authorized_keys` 文件由 Rails 的 `AuthorizedKeysWorker` 在每次密钥变更时维护——**"刚加的部署密钥不生效"先查这个异步任务有没有积压**；用 SSH 证书认证时对应换成 `AuthorizedPrincipalsCommand`。

> **设计争议**：为什么 Git 不直接给 Rails 处理？因为 Ruby 进程处理流式大字节性价比极低，把"字节搬运"剥离给 Go 写的 Workhorse/Gitaly，Rails 只做决策，是 GitLab 性能架构的核心取舍。

# 4. Gitaly：被低估的主角

自 11.3 起，GitLab 里**所有 Git 层访问都收敛到 Gitaly**——它的设计初衷正是让分布式部署不再需要 NFS 共享 Git 目录。

## 4.1 Gitaly 的六个客户

Gitaly 是 gRPC 服务器，官方点名的客户端（消费者）有六个：Rails、GitLab Shell、Workhorse、Elasticsearch Indexer、Zoekt Indexer、GitLab Agent for Kubernetes (KAS)。这意味着：搜索索引重建、代码搜索、Agent K8s 集成全都排队走 Gitaly——**"我没 push，Gitaly 为什么忙"**多半是索引器在扫仓库。另外官方架构图里 Gitaly 还会反向 TCP 调用 Workhorse；仓库级的大体积搬运（迁移/复制）走 **Bundle URIs** 机制，由 Gitaly 直接从签名 URL 拉包——两条都说明 Gitaly 不是"末端哑服务"，扩容评估时要把它周围的调用环一起看。

## 4.2 磁盘布局：哈希存储

仓库磁盘上不是裸 `.git` 结构，Omnibus 下的路径是：

```text
/var/opt/gitlab/git-data/repositories/@hashed/<前2位>/<次2位>/<sha256(项目ID)>.git
```

- 哈希用**项目 ID** 算 SHA256，与名称/namespace 无关——所以**改名/改组只是一个数据库事务，磁盘零移动**，这是哈希存储存在的理由；
- wiki 是旁边那个 `.wiki.git`（第 4 篇说过"每项目最多 3 仓库"，磁盘上就在这同一个 `@hashed` 树里）；
- 官方口径：**直接访问包含仓库的目录已被标记为弃用**，一切走 Gitaly/API/管理命令，别写脚本去扫 `git-data`；
- 项目名 ↔ 磁盘路径的互查：Admin Area 项目详情里的 **Relative path** 字段，或 Rails 控制台：

```ruby
Project.find(16).disk_path
Project.find_by_full_path('group/project').disk_path
```

## 4.3 钩子与拦截的正确位置

Gitaly 模型下，服务器上没有"ssh 上去往 `.git/hooks` 写脚本"的执行环境。自定义 `pre-receive` 要走 **Gitaly Hooks**（官方文档：custom hooks 目录 `/var/opt/gitlab/gitaly/custom_hooks`），平台级的拦截用 Push Rules（第 4 篇）。在裸 Git 服务器思维里写的"仓库内钩子"脚本，迁移到 GitLab 时十有八九是无效的。

## 4.4 I/O 画像与并发限制

Gitaly 是重 I/O 进程，官方强烈建议所有 Gitaly 节点上 **SSD**，并给了 GitLab.com 生产集群的参考画像：读稳态 **500–1000 IOPS、峰值 3500**；写稳态约 **500 IOPS、峰值 3000+**（原始数据写峰值可到 8000）——盘吞吐要能扛住工作日的尖峰而不是平均值。

单仓库的并发可以在 Gitaly 侧限流（内存受限环境的官方示例直接就是它）：

```ruby
gitaly['configuration'] = {
  concurrency: [
    { 'rpc' => "/gitaly.SmartHTTPService/PostReceivePack", 'max_per_repo' => 3 },
  ],
}
```

"一个大仓库把整台 Gitaly 拖死"的场景（monorepo 集中 push）就靠 per-RPC + max_per_repo 这类限制兜底，完整并发治理见 Gitaly 文档的 concurrency limiting。

## 4.5 Gitaly Cluster（Praefect）

要 Git 层高可用就上是 **Gitaly Cluster = Praefect + 多副本存储**：客户端依旧连 gRPC，由 Praefect 做副本一致性协调。**注意它的隐藏成本：Praefect 需要一个独立的 PostgreSQL 实例**（官方要求与主库分开；全 HA 时这个库自己也要高可用）——很多人规划集群时漏掉这条，上线才发现"给 Gitaly 配的 HA 还附赠一套数据库运维"。

# 5. Workhorse 与 Shell：两道 Go 大门

Rails 只该做决策，不该搬字节——GitLab 的答案是把"搬字节"的活拆给两个 Go 进程：HTTP 侧是 Workhorse，SSH 侧是 GitLab Shell。认住它们的故障指纹，能省掉大量无头苍蝇式重启。

## 5.1 Workhorse

定位是"智能反向代理"：从 NGINX 接请求，判断该去 Rails（业务）、对象存储（LFS/附件）、还是 Gitaly（git 流量）。上传大文件时它把字节流直接缓冲/转走，Rails 只收到"校验完成后的元数据请求"；克隆时它把 `git-upload-pack` 的输出流式转发给客户端。官方文档还专门写过它的由来（"A brief history of GitLab Workhorse"），核心动机就一条：Rails 处理大流量太贵。故障表现非常特征化：**页面正常，但上传和 HTTP 克隆失败**——第 9 节的路由表按这个对号入座。

## 5.2 GitLab Shell

处理 SSH 版 git 会话，并维护 authorized_keys 机制；官方特意强调 **它不是 Unix shell，也不是 Bash/Zsh 的替代**。它还承载 SSH 证书认证、`authorized_keys` 快速查询入口（3.4 的时序里它是"sshd 之后、Gitaly 之前"的那一跳）。故障表现：SSH 克隆在会话建立阶段就被拒绝、报与 gitlab-shell 相关的错误，而 Web 与 HTTP 克隆一切正常。Shell 与 Workhorse 的分工一句话：**同一套内部 API 鉴权问题，一个在 SSH 入口问，一个在 HTTP 入口问**——所以"SSH 能推、HTTPS 不能推"这种不对称故障，多半是两道门之一（或它的前置）出了问题，而不是权限数据本身。

# 6. Rails、Puma 与 Sidekiq

## 6.1 Puma：Web 线的发动机

Rails 由 Puma 集群模式承载（一个 master + N 个 worker，源码安装的默认画像就是 master + 2 worker + 1 sidekiq）。worker 数量与内存直接挂钩——**"GitLab 吃内存"的本体就是每个 Puma worker 一个完整 Ruby 应用**。官方 sizing 表、内存看门狗（RSS 超限重启并留 WARN 日志）与调优入口都在第 2 篇 5.2，这里只记架构含义：Puma 是**无状态**的，多实例是 Web 线扩容的正道。

## 6.2 Sidekiq：异步线的心脏

所有"稍后才发生"的事都在这里：邮件、Webhook、CI 状态传播、后台迁移、仓库 GC 排程。资源基线（官方数字）：**每进程至少 200 MB，万人级按每进程 ≥1 GB 规划**。它同样是**无状态**的，可横向加实例（官方文档给了 extra Sidekiq processes 与"按 job class 分机消费"的方案）。队列积压的用户感知就是第 1 节那条"状态变了但按钮没刷新"——观测入口在 Admin → Operations → Sidekiq（队列长度与延迟）。

## 6.3 Redis

四件事：缓存、会话、Sidekiq 队列、WebSocket/Action Cable 实时通道。架构约束三条（全是官方口径）：**不支持 Redis Cluster**（只能用单机/哨兵模式的外部实例）、serverless 形态不支持、**Redis 不备份**（第 13 篇）——丢了等于全员重新登录、队列清空（未消费的异步任务会丢，这是"备份里没有 Redis"的代价）。

## 6.4 PostgreSQL

唯一"不能丢也不能换姿势"的组件：所有元数据、且**全站只有它承载权限模型**。架构要点：大版本支持矩阵与必需扩展清单见第 2 篇 2.3；连接需求由 Puma worker × thread 与 Sidekiq 并发共同决定（官方 tune 文档给的公式就是这个乘积关系，改 worker 数前先算连接）；从 17.x 起元数据进一步拆分 **main + ci 双库**（第 13 篇 PgBouncer 节里 `GITLAB_BACKUP_MAIN_PG*`/`GITLAB_BACKUP_CI_PG*` 两套参数的由来）；Ultimate 的漏洞库约 1 GB，所以 PG 磁盘按 5–12 GB 起规划。

# 7. 元数据与 Git 数据的分离

| 数据 | 存储 | 备份归属 |
|------|------|---------|
| 用户、权限、Issue、MR、流水线、CI 变量 | PostgreSQL | `gitlab-backup create` 包含 |
| Git 对象、LFS 对象、CI 制品、附件、Pages | 磁盘（或对象存储） | 同上，分开打包 |
| 会话、缓存、队列 | Redis | 不备份，丢了重新登录即可 |

## 7.1 对象存储的位置

官方现行建议：**对象存储对分布式部署是必需、对所有安装都推荐**——LFS 对象、CI 制品、上传附件、容器镜像、备份都可以落到任意 S3 兼容存储。它的架构含义：数据离开应用磁盘后，**备份包不再包含对象存储里的数据**（第 13 篇备份"进包清单"的反面就是这条），容灾方案要把桶的复制/版本控制单独算进去。

## 7.2 设计考量

- PostgreSQL 是**唯一不能丢**的组件；Git 数据丢了是资产灾难；Redis 和（部分）制品可以从其他系统重建。
- 大规模自建时把这三样拆到独立机器，应用机器只做"计算层"——这正是 8 节演进路径的中间形态。

# 8. 单机到扩展部署的演进路径

```text
阶段1 单机 Omnibus          阶段2 拆分存储            阶段3 高可用集群
┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│ 全部组件     │ ──► │ 应用机            │ ──► │ 负载均衡 + 多应用节点 │
│ (≤1000人够用)│     │ PG/Redis/Gitaly   │     │ PG 主从 / Redis 哨兵 │
│             │     │ 独立机器          │     │ Gitaly Cluster      │
└─────────────┘     └──────────────────┘     └─────────────────────┘
```

对使用者最有用的推论：**单机基线达标（第 2 篇 8C/16G）撑千人团队没问题**，不要过早做集群；但一旦要做高可用，Puma/Sidekiq 分机多实例只是起点，难的是 Data 层。

## 8.1 官方参考架构与 sizing 方法论

官方参考架构分档：**1k、2k、3k、5k、10k、25k、50k 用户** + cloud-native 专篇。现行 sizing 文档的方法论值得记住——它拒绝"按人数定档"，改成**按实测负载画像**：先用 Prometheus 指标抽峰值 RPS，把工作负载特征化成 Small/Medium/Large/Extra Large（重 CI、重 API 自动化、活跃 monorepo 会把不同组件压出不同瓶颈），再映射到最近档参考架构；大 monorepo 与网络密集型负载单列"特殊基础设施需求"。官方还直说：Large/XL 规模的运维复杂度往往值得上专业服务，或直接考虑 Dedicated/SaaS。

## 8.2 无状态加机器，有状态换姿势

- **Puma 与 Sidekiq 无状态**，官方明确支持 autoscaling group 横扩；
- PostgreSQL：主从 + 故障转移（EE 生态用 Consul 做节点发现/切换）；
- Redis：单机或哨兵——**Cluster 模式不支持**；
- Gitaly：独立机器 + SSD 起步，再上 Praefect（记得那个专用 PG）。

## 8.3 拓扑红线

跨节点延迟 **< 5 ms**（同步复制前提）、奇数个 AZ、**单实例绝不跨地理区域**——跨 region 的诉求答案是 Geo，而不是把一套实例拉长部署。

# 9. 排错视角：调用方向定位法

排错时记住调用方向就能定位八成问题：

- 页面慢但 git 操作快 → Puma/Sidekiq/PG（Web 线）。
- git 操作慢但页面正常 → Gitaly（字节线）；只有 SSH 挂 → Shell。
- 上传/HTTP 克隆失败但页面正常 → Workhorse。
- MR 状态/通知延迟 → Sidekiq 队列积压：Admin → Operations → Sidekiq 看队列长度。
- 登录态反复丢失 → Redis。

## 9.1 日志与关联 ID

Omnibus 的日志全部在 `/var/log/gitlab/<组件>/`（源码安装在 `/home/git/gitlab/log/` 等处，SSH 认证看 `/var/log/auth.log`）。Rails 的 `application_json.log` 每条请求日志都带 **`correlation_id`** 字段，Workhorse/Gitaly 的 JSON 日志同样携带它——用同一个 ID 串起多组件日志，是重建"一次请求之旅"的最快办法：

```bash
sudo gitlab-ctl tail puma | grep <correlation_id>
sudo gitlab-ctl tail gitaly | grep <correlation_id>
```

## 9.2 两条检查线

`gitlab-ctl status` 是**进程线**健康检查（活着≠配好），`gitlab-rake gitlab:check SANITIZE=true` 是**数据线**检查（权限、钩子、仓库可达）；两条都绿再谈更深的问题。后台任务视角的积压/失败，看 Admin 的 Sidekiq 与后台迁移界面（第 13 篇工具箱）。

# 10. 小结

| 要点 | 一句话 |
|------|--------|
| 三条线 | 元数据（PG）、Git 数据（Gitaly）、异步（Redis+Sidekiq）；接口分 Web 与 Git 两个门面 |
| 分层 | Processor 层（Workhorse/Shell/Puma/Sidekiq）可横扩，Data 层（PG/Redis/Gitaly）决定 HA 难度 |
| Workhorse | 官方定义 smart reverse proxy：大流量不过 Rails，鉴权决策仍归 Rails |
| SSH 路径 | 只有 sshd 说 SSH；fast key lookup/authorized_keys、内部 API、Gitaly SSHService 三段接力 |
| Gitaly | 一切 Git 操作唯一入口，六个 gRPC 客户；@hashed 目录不可手工触碰、直接访问已弃用 |
| 备份观 | PG 不可丢、Git 是资产、Redis 不备份、对象存储数据在包外 |
| 演进 | 按实测 RPS 画像选档而不是人数；无状态加机器，有状态（PG/Redis/Praefect）换姿势 |

自检清单：

- [ ] 能对着 `gitlab-ctl status` 说出每个进程属于哪条线、挂了影响什么？
- [ ] 知道仓库在磁盘上是 `@hashed/xx/yy/<sha256(id)>.git`，且改名不动盘？
- [ ] 遇到"上传失败页面正常"第一反应是 Workhorse 而不是 NGINX？
- [ ] 计划过 Praefect 的那个独立 PG、Redis 无 Cluster、对象存储不进备份包这三件事？
- [ ] 会用 correlation_id 串 Rails/Gitaly 日志复现一次请求？

架构清楚了，下一篇回到日常：项目与仓库怎么建、分支模型怎么选、保护分支和 Push Rules 怎么配。
