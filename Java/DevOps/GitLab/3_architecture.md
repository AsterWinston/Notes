# 1. 为什么要了解架构

GitLab 不是"一个 Rails 应用"那么简单：Omnibus 装的单机实例里同时跑着 **十来个进程**。了解架构的目的不是面试八股，而是为了解释使用者日常会遇到的现象——

- 为什么 push 大仓库这么慢？（Gitaly）
- 为什么页面偶尔转圈、重启后要先 warm 一下？（Puma worker）
- 为什么流水线状态变了但 Issue 上的合并按钮没刷新？（Sidekiq 异步队列积压）
- 为什么 GitLab 吃内存？（Rails 单体的固有代价）

一句话架构观：**GitLab 是"元数据 + Git 数据 + 异步任务"三条线的组合**，元数据在 PostgreSQL，Git 数据在 Gitaly，异步任务由 Sidekiq 消化。

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
| Workhorse | Go | 代理大流量：接 LFS 上传、校验后交给 Rails；流式转发 git 请求 | 上传/克隆失败 |
| Rails (Puma) | Ruby | 页面、REST API、业务规则 | 页面不可用，git SSH 仍可能活 |
| Gitaly | Go | 所有 Git 操作的唯一入口（收发 push、算 diff、读 blob） | 仓库操作全挂 |
| PostgreSQL | C | 用户/项目/Issue/流水线等一切元数据 | 全站瘫痪（最核心） |
| Redis | C | 缓存、会话、Sidekiq 队列、实时事件 | 登录态丢失、任务停摆 |
| Sidekiq | Ruby | 异步任务：发邮件、跑 Webhook、CI 状态传播、GC | 功能延迟而非立即故障 |
| Prometheus/Grafana | 可选 | 内置监控栈 | 无监控，不影响业务 |

# 3. 一次请求的旅程

**Web/API 请求（如合并 MR）**：

```text
浏览器 → NGINX → Workhorse(鉴权前挡大请求体) → Puma(Rails)
       → 写 PostgreSQL(MR 状态) → 发事件给 Redis
       → Sidekiq 消费: 发邮件通知、触发 Webhook、更新流水线
```

**git push（HTTP）**：

```text
git CLI → NGINX → Workhorse(流式接收 pack, 限大小)
        → Rails(Preflight 鉴权: 你能不能 push 这个分支)
        → Gitaly(接收 pack、执行 receive-pack、跑钩子)
        → Rails(钩子回调: 更新 MR、触发 Pipeline) → Sidekiq(后续异步)
```

**git clone/pull**：Rails 只做鉴权，之后 `git-upload-pack` 流量由 Workhorse 直接转发给 Gitaly——**大流量不过 Rails**，这是 Workhorse 存在的意义。

> **设计争议**：为什么 Git 不直接给 Rails 处理？因为 Ruby 进程处理流式大字节性价比极低，把"字节搬运"剥离给 Go 写的 Workhorse/Gitaly，Rails 只做决策，是 GitLab 性能架构的核心取舍。

# 4. Gitaly：被低估的主角

所有 Git 读写都收敛到 Gitaly，带来三个直接影响：

- **仓库磁盘上不再是裸 `.git` 结构**：Omnibus 下仓库在 `/var/opt/gitlab/git-data/repositories/<namespace>/<hash>/.git`，目录名是哈希不是项目名，**不要手工在文件系统里挪仓库**，一切走 API/管理命令。
- **服务器上没有仓库内钩子的执行环境**：`pre-receive` 等自定义钩子要走 Gitaly hooks 或平台级 Push Rules，不能像裸 Git 服务器那样 ssh 上去写 `.git/hooks`。
- **大仓库/大 diff 的瓶颈在这里**：Monorepo 团队扩容的第一站是 Gitaly 独立部署 + SSD + 集群（Gitaly Cluster/Praefect）。

# 5. 元数据与 Git 数据的分离

| 数据 | 存储 | 备份归属 |
|------|------|---------|
| 用户、权限、Issue、MR、流水线、CI 变量 | PostgreSQL | `gitlab-backup create` 包含 |
| Git 对象、LFS 对象、CI 制品、附件、Pages | 磁盘（或对象存储） | 同上，分开打包 |
| 会话、缓存、队列 | Redis | 不备份，丢了重新登录即可 |

**设计考量**：

- PostgreSQL 是**唯一不能丢**的组件；Git 数据丢了是资产灾难；Redis 和（部分）制品可以从其他系统重建。
- 大规模自建时把这三样拆到独立机器，甚至对象存储（LFS/制品/备份上 S3 兼容存储），应用机器只做"计算层"。

# 6. 单机到扩展部署的演进路径

```text
阶段1 单机 Omnibus          阶段2 拆分存储            阶段3 高可用集群
┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│ 全部组件     │ ──► │ 应用机            │ ──► │ 负载均衡 + 多应用节点 │
│ (≤1000人够用)│     │ PG/Redis/Gitaly   │     │ PG 主从 / Redis 哨兵 │
└─────────────┘     │ 独立机器          │     │ Gitaly Cluster      │
                    └──────────────────┘     └─────────────────────┘
```

对使用者最有用的推论：**4C8G 单机撑到千人团队没问题**，不要过早做集群；但一旦要做高可用，Rails（Puma）与 Sidekiq 必须分机部署，且 PostgreSQL 不能再"裸奔"。

# 7. 服务发现与依赖方向（排错视角）

排错时记住调用方向就能定位八成问题：

- 页面慢但 git 操作快 → Puma/Sidekiq/PG（Web 线）。
- git 操作慢但页面正常 → Gitaly（字节线）。
- MR 状态/通知延迟 → Sidekiq 队列积压：Admin → Operations → Sidekiq 看队列长度。
- 上传大文件失败 → Workhorse 与对象存储/LFS。
- 登录态反复丢失 → Redis。

`gitlab-rake gitlab:check` 与 `gitlab-ctl status` 分别是数据线与进程线的健康检查入口。

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 三条线 | 元数据（PG）、Git 数据（Gitaly）、异步（Redis+Sidekiq） |
| Workhorse | 大流量挡在 Rails 外面的 Go 守门员 |
| Gitaly | 一切 Git 操作唯一入口，磁盘结构不可手工触碰 |
| 备份观 | PG 不可丢、Git 是资产、Redis 可重建 |
| 演进 | 单机 → 拆存储 → 高可用，按用户量自然升级 |

架构清楚了，下一篇回到日常：项目与仓库怎么建、分支模型怎么选、保护分支和 Push Rules 怎么配。
