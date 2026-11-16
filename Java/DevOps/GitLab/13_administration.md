# 1. 管理者的工作面：治理而非装机

本篇面向群组 Owner / 实例管理员的日常职责，按使用者视角裁剪：**备份恢复、升级、群组治理、配额与容量、Webhook 集成**。深度高可用部署（Gitaly Cluster/Geo）点到为止，细节属于专职运维范畴。

# 2. 备份与恢复：先搞清楚备什么

```bash
# 全量备份（在 GitLab 机器上，Omnibus）
sudo gitlab-backup create SKIP=registry        # 可跳过部分数据

# 备份包结构
# /var/opt/gitlab/backups/<timestamp>_gitlab_backup.tar
#   ├── db.dump            数据库（元数据，最关键）
#   ├── uploads.tar        附件、LFS、制品（若用对象存储则不在包内）
#   ├── repositories.tar   Git 仓库数据
#   └── features.tar       CI 制品等
```

| 数据 | 是否进备份 | 备注 |
|------|-----------|------|
| PostgreSQL 元数据 | ✔ | 不可丢 |
| Git 仓库 | ✔（`gitlab-rake gitlab:backup:create` 默认包含） | 体积大头 |
| LFS/制品/上传 | ✔ 或对象存储直连 | 上对象存储后备份只存引用 |
| `gitlab.rb`/密钥文件 | ✘ **不包含** | 必须单独 `tar /etc/gitlab`（含 secrets，权限 600） |
| Redis | ✘ 不需要 | 会话可重建 |

**恢复**：`gitlab-backup restore BACKUP=<时间戳>`（要求目标实例版本与备份一致）。

> **行业实践**：备份的价值取决于**恢复演练**。每季度至少一次：新起一台同版本机器 → 恢复 → `gitlab-rake gitlab:check` → 抽样克隆仓库、访问 MR。没演练过的备份等于没有备份。异地备份用对象存储（S3 兼容）直接落 `connection` 配置，别靠 cron scp。

# 3. 升级：按路径走，别跳级

- 升级路径规则：**小版本顺走；跨大版本必须经过每个大版本的"最后一个补丁版"**（如 16.11 → 17.0 → 17.x）。官方提供 upgrade-path 工具查询。
- 标准流程：全量备份 → 阅读该版本 breaking changes → `apt install gitlab-ce=<版本>`（Docker 改镜像 tag）→ reconfigure → `gitlab-rake db:migrate` 自动执行 → 冒烟测试（登录、克隆、跑一条流水线）。
- 升级窗口内 Web 只读/短暂不可用；大版本迁移（schema 变更）耗时与数据量成正比，提前在预发环境演练。

# 4. 群组治理：结构、权限与模板下发

群组是治理载体，管理员常用动作：

| 能力 | 位置 | 用途 |
|------|------|------|
| 群组可见性/分享 | Group → Information → Settings | Internal/Private；限制"项目分享给群组外" |
| 成员批量授权 | Group → Members（按来源分组显示继承） | 一次授权子群组全继承；离职清理从这里入口 |
| **CI/CD 模板下发** | 群组级 CI 变量、群组模板仓库 | 全部门统一流水线骨架（第 7 篇 include 的企业形态） |
| 安全策略下发 | 群组 Settings → Security & Compliance（Premium 起） | 默认扫描器、push rules、IP 白名单继承 |
| 配额 | Admin Area → Applications → Settings（仓库大小/CI 分钟数） | 限制失控膨胀：`Maximum artifacts size`、存储配额 |
| 自动批准加入 | Group → Access requests | Owner 可设自动审批策略 |

**设计考量**：群组层级一旦建好，权限继承与策略继承都跟着层级走；**事后重构群组结构 = 全量 URL/CI 引用迁移**，所以第 4 篇的"组织结构设计"建议在试点阶段就定调。

# 5. Webhook 与事件集成：GitLab 作为事件源

三类"事件出口"能力，选对机制能省大量轮询：

| 机制 | 触发方 | 可靠性 | 典型场景 |
|------|--------|--------|---------|
| **Project/Group Webhook** | GitLab 主动推 HTTP POST | 有重试，可查投递记录 | 通知内部系统：MR 合并触发发布、push 触发文档重建 |
| **System Hooks**（Admin） | 全实例事件 | 同上 | 接安全审计：谁删了项目 |
| **Integrations**（内置） | 预制模板 | — | Jira/Slack/Microsoft Teams/PagerDuty 联动 |

```yaml
# 例：MR 合并后通知外部部署系统 —— 用 Webhook 而不是让 CI 依赖外部系统
# Group → Settings → Webhooks：触发事件选 Merge Request Events
```

**设计考量**：Webhook 是"通知"语义（fire-and-forget，签名校验 + 重试），不是"编排"语义；需要强事务的联动（合并即发布且必须成功），把发布做成 CI job 而非 webhook 消费端。

**入方向**：外部系统可通过 REST API + OAuth/项目令牌在 GitLab 里建 Issue、触发流水线（`POST /projects/:id/trigger/pipeline`）、评论 MR——把 GitLab 变成其他系统的协作后端。

# 6. 容量与性能的日常治理

管理员每月看这几个数（Admin Area → Metrics）：

- **Sidekiq 队列深度**：持续积压 → 加 Sidekiq 进程/机器，或查 `slow_query` 日志。
- **Puma 饱和度**：`puma_backlog` 高 → worker 不够（内存允许时加 `puma['worker_processes']`）。
- **Gitaly 请求延迟**：clone/push 变慢先查它，通常是磁盘 IO。
- **存储增长曲线**：LFS/制品/附件各占多少？对照保留策略是否生效（第 10 篇）。
- **CI 分钟数**：Runner 利用率 + job 时长分布，指导扩 Runner 还是加 `rules: changes` 裁剪。

# 7. 高可用：使用者视角的"够用层级"

```text
层级1 单实例 + 好备份            ← 小团队：恢复时间以小时计，可接受
层级2 应用层冗余（多 Puma/PG 主从）← 数百人：PG 主从 + 自动 failover 是关键
层级3 全组件冗余 + Gitaly Cluster  ← 数千人：Praefect 复制 Git 数据（16.x 后取代 NFS 方案）
层级4 Geo 多站点                 ← 多地访问延迟/容灾合规
```

> **行业实践**：GitLab 官方立场是"能用 GitLab.com/托管版就别自建高可用"；真要自建，**把 PostgreSQL 的高可用做好收益最大**（它挂全站挂），Gitaly/PG 的数据一致性远复杂于无状态层。多数团队到层级 2 就够。

# 8. Admin Area 速览（管理员日常入口）

| 页面 | 干什么 |
|------|--------|
| Users | 封禁/激活、审计登录方式、2FA 合规视图 |
| Projects/Groups | 全局搜索、归属转移、批量归档 |
| Settings → General | 注册、可见性默认、签名要求、文件限额 |
| Settings → CI/CD | 全局变量、Runner 注册限制、Auto DevOps 开关 |
| Runners | 实例 Runner 池、成本因子（配额计费） |
| Abuse reports / Spam | 公网实例的举报处理 |
| Health Check (`/-/readiness`) | 负载均衡探活端点 |

# 9. 小结

| 要点 | 一句话 |
|------|--------|
| 备份 | 备 PG+仓库+对象数据，`/etc/gitlab` 单独备；定期恢复演练 |
| 升级 | 沿 path 逐级走，大版本先在预发演练 |
| 治理 | 群组承权限/模板/策略下发，结构宜早定 |
| 集成 | Webhook 通知、API 入站；强一致联动放进 CI |
| 容量 | 盯 Sidekiq/Puma/Gitaly/存储四张曲线 |
| HA | PG 高可用优先，按用户规模逐级演进 |

治理之外还要"看得见"，下一篇讲 GitLab 自带的监控观测能力与日志排错入口。
