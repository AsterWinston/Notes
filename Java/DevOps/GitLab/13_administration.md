# 1. 管理员工作面：本篇怎么用

本篇面向**群组 Owner 与实例管理员的日常职责**，按"使用者要做的事"裁剪，而不是按产品菜单罗列。深度部署调优（Gitaly Cluster 参数、Geo 站点拓扑、参考架构选型）只给结论与边界，细节属于专职 SRE 范畴，落在第 2、3 篇与官方文档。

## 1.1 六条工作线

| 工作线 | 你要回答的问题 | 章节 |
|--------|----------------|------|
| **备份与恢复** | 出事后能回到哪一刻、要多久、谁批准演练 | 第 2、3 节 |
| **升级** | 能不能跳、要停几站、停机多久、怎么回滚 | 第 4 节 |
| **实例设置与门禁下发** | 全公司的默认值由谁定、改了怎么保证老项目也生效 | 第 5、6 节 |
| **群组治理** | 结构、成员继承、配额、模板与策略下发 | 第 7 节 |
| **事件出口与席位** | 谁在动 GitLab（Webhook/System hooks）、多少钱（席位） | 第 8、9 节 |
| **容量与可用性** | 现在离崩多远、崩了怎么办、要不要上 HA | 第 10、11 节 |

> 权限模型本身（角色、令牌、保护分支、推送规则、扫描、审计事件、账号处置）在第 12 篇，本篇**不重复展开**，只讲"管理员怎么把这些设定下发到全实例"。观测与日志（Prometheus 指标名、日志文件地图、慢查询）在第 14 篇，故障现场处置在第 16 篇。

## 1.2 三条纪律（本篇全部结论的地基）

1. **任何"我以为实例是这样配的"都要先读回现值。** `GET /api/v4/application/settings` 能读回绝大多数实例设置键；改之前先 `jq` 出一遍贴进变更单，改之后再读一遍对比。UI 文案随版本变，键名不会骗人。
2. **备份的价值等于最近一次成功恢复演练的价值。** 官方在升级文档里把"回滚计划"定义为"备份流程 + 恢复流程"两件事，并要求**在需要之前测试过**。没演练过的备份，在事故现场只是一份未知格式的大文件。
3. **升级的停靠站（upgrade stop）不能省，且每站之间要等后台迁移跑完。** 这不是保守，是官方写死的规则：`In GitLab 17.5 and later, to provide a predictable upgrade schedule for instance administrators, required upgrade stops occur at versions x.2.z, x.5.z, x.8.z, and x.11.z`，并且要求"Allow the background migrations for the upgrade to finish"再走下一站。

---

# 2. 备份：先搞清楚包里有什么

官方备份工具是 `gitlab-backup`（Omnibus/Docker/源码安装），默认产出**一个 tar 压缩包**。先记住一句话：**它备的是"应用数据"，不备"配置与密钥"**，而恢复时两者缺一不可。

## 2.1 决策全景：先按数据量选路线

```text
你的 PostgreSQL 多大？        Git 仓库多大？        对象数据在哪？
   < 100 GB                    单仓库 < ~40 GB       本地磁盘
        │                            │                    │
        └── 2.4 默认 tar 全备 ────────┴────────────────────┘
             （需要时 STRATEGY=copy / COMPRESS_CMD 并行压缩）

   > 100 GB                    仓库很大 / 有巨型单体   已在对象存储
        │                            │                    │
        ├── PG 交给原生备份（RDS/Cloud SQL/PITR，2.6）      │
        ├── 仓库走 server-side + 增量（2.6）───────────────┤
        └── 对象存储由云侧自身复制/版本控制负责（2.2、2.6）─┘
```

| 路线 | 适用 | 代价 | 见 |
|------|------|------|----|
| 默认 `gitlab-backup create` | PG < 100 GB、数据在本地磁盘 | 备份窗口与 DB 体积线性相关 | 2.3–2.5 |
| + `STRATEGY=copy` | 数据活跃、报 `file changed as we read it` | 最多多占 1 倍磁盘 | 2.4 |
| + 远端上传（S3/GCS/Azure） | 备份不能只留在本机所在故障域 | 需 IAM/加密/生命周期三件事配齐 | 2.7 |
| 仓库 server-side + 增量 | 仓库体积主导、全备跑不完 | 增量链断裂即不可恢复；Helm 不支持该组合 | 2.6 |
| PG 原生备份 + 对象存储侧复制 | 大实例唯一可持续路线 | 恢复流程由多套工具拼，**必须演练** | 2.6 |
| 磁盘快照 | 灾备兜底 | **官方明说超出支持范围**，多节点要逐个拍 | 4.8 |

> **别把"备份命令跑成功"当"备份策略成立"**。策略要回答四件事：RPO（能回到几分钟前）、RTO（多久能恢复可用）、范围（2.2 那张"不包含"表你补了哪几块）、演练（3.8）。这四件事写不进变更单，就等于没有策略。

## 2.2 进包与不进包（官方清单）

**默认包含**：

```text
<timestamp>_gitlab_backup.tar
  ├── db.dump / *_db.dump      PostgreSQL 元数据（主库与 CI 库分别导出）
  ├── repositories/            Git 仓库数据（含 wiki）
  ├── lfs/                     LFS 对象
  ├── artifacts/               CI job 制品
  ├── builds/                  CI job 输出日志
  ├── uploads/                 附件与项目上传
  ├── registry/                容器仓库镜像
  ├── packages/                包仓库数据
  ├── pages/                   GitLab Pages 内容
  ├── terraform_state/         Terraform state
  ├── ci_secure_files/         项目级 Secure Files
  ├── external_diffs/          外部 MR diff（17.1 起进包）
  ├── snippets / wikis         代码片段与 wiki
  └── agent_plan_content/ ci_catalog_bundles/
```

**不包含**（每一条都是真实故障源）：

| 不包含的东西 | 为什么 / 后果 | 正确做法 |
|--------------|---------------|----------|
| **Redis（含 Sidekiq 队列）** | 会话与排队任务可重建 | 无需备，但要接受"排队中的 job 丢失" |
| **对象存储里的数据**（LFS/制品/上传/registry/packages 等） | Omnibus/Docker/源码安装下，`gitlab-backup` **不备份已在对象存储里的内容** | 用对象存储自身的版本控制/复制/生命周期策略，或先跑回本地再备 |
| **`/etc/gitlab`（含 `gitlab-secrets.json`）** | 密钥与密文不能同处存放 | 见 2.8，单独备 |
| **TLS / SSH 证书与密钥**、SSH host keys | 系统文件 | 单独备；不备会在整机恢复后触发 MITM 告警 |
| **全局 server hooks / file hooks** | 属于文件系统级 Git hook | 单独备（第 8 节 system hooks 是数据库里的，不同东西） |

> **Java 团队最常漏的两项**：容器仓库镜像与 Maven 包。它们**在对象存储里时不进备份包**，很多团队"备份一切正常"却在 registry 迁走那天才发现包里没有镜像。判定方法：`cat /etc/gitlab/gitlab.rb | grep -E 'registry_object_store|packages_object_store|artifacts_object_store|lfs_object_store'` 一旦配了 `object_store`，对应数据就归对象存储负责。

**官方对"哪些数据必须被备份覆盖"的分类（决定你 RPO 的三块）**：

| 数据类别 | 事实 | 备份含义 |
|----------|------|----------|
| **PostgreSQL** | 是 Web 侧内容（Issue/MR/评论/权限/凭据）的**唯一真相源**；还缓存渲染后的 Markdown、默认也存 MR diff（可 offload）；**Praefect 也用 PG 当真相源** | 只有 `pg_dump` 路线：库越大越慢，>100 GB 就不可用（2.6） |
| **Git 仓库** | 一个实例可以有**多个 repository shard**（每 shard 一个 Gitaly 或 Gitaly Cluster）；每个项目**最多 3 个仓库**：主仓库、`-wiki`、`-design`（design 只存索引，资源在 LFS）；**个人/项目 snippet 与群组 wiki 也是 Git 仓库**；fork 用 pool repository 去重 | 备份对每个仓库产出一个 **bundle** 再打包 —— **pool 数据会被复制进每个 fork**，所以 fork 多的实例备份体积会远超直觉；官方实测 **100 GB 仓库 ≈ 2 小时**（含上传 S3），**约 400 GB 仓库时备份命令基本不可用** |
| **Blobs（附件/LFS）** | 存文件系统**或**对象存储 | 在对象存储里就**完全不进包**，必须靠云侧备份 |
| **Container registry** | 存文件系统**或**对象存储 | 同上；另外**开了 registry metadata database 时，备份必须额外配该库的凭据**，否则镜像元数据不在备份里（Omnibus/Helm 各有配置项） |

**两条一致性要求（很多人直到出事才知道）**：

- **Redis 不在备份里**，而它是 Sidekiq 的持久队列。官方原话：`in order to take a consistent backup with the backup command, there must be no pending or running background jobs`。也就是说**严格一致的全备要挑队列清空的窗口**；日常在线备份只能接受"排队中的任务丢失"这一 RPO 缺口，并在恢复后核对 job 历史（第 14 篇队列观测）。
- **Elasticsearch 也不备份**，但它属于**可从 PostgreSQL 重建**的数据（高级搜索索引），恢复后重新索引即可，不需要备份。

**性能相关的一条好消息**：官方在 **18.0 起显著改善了"引用（分支/标签）数量很多"的仓库备份性能**，受影响仓库的备份时间**从数小时降到数分钟**，且**无需任何配置**。所以"我们仓库多分支导致备份 6 小时"这类历史结论，在升到 18.0 之后要重新测量。

## 2.3 备份命令：四种部署形态与"在哪台机器上跑"

```bash
# Linux package (Omnibus)
sudo gitlab-backup create

# Docker：在宿主机上执行
docker exec -t <container name> gitlab-backup create

# 源码安装
sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production

# Helm chart (Kubernetes)：进 toolbox pod
kubectl exec -it <gitlab-toolbox-pod> -- backup-utility
```

**跑在哪台机器上不是随便挑的**，官方三条要求：

- **持久节点**，不能是会被自动伸缩回收的实例；
- 该节点**已装 GitLab Rails 应用**（判据：Puma 或 Sidekiq 在跑就说明装了）；
- 有**足够磁盘与内存**产出备份文件（备份是"读全量 + 压缩 + 落盘"，中途满盘等于白跑）。

另外两条前置条件：

- **Rsync 必须可用**（备份与恢复都要）：Linux package 自带，**源码安装要自己检查并安装 `rsync`**。
- 仓库备份由 **`gitaly-backup` 二进制**完成（它取代了"Rails 直接调 Gitaly RPC"的老方式），默认路径 `/opt/gitlab/embedded/bin/gitaly-backup`，一般不用管；特殊环境（自定义安装路径、容器里缺二进制）用 `gitlab_rails['backup_gitaly_backup_path'] = '/path/to/gitaly-backup'` + reconfigure 指过去。备份报"找不到 gitaly-backup"就是这一项。

**Helm 侧不是 `gitlab-backup` 而是 toolbox 里的 `backup-utility`**，参数名不同（`--skip a --skip b` 形式、`--repositories-server-side`、恢复用 `--restore -t <backup-id>`），并且**组件清单与 Omnibus 略有差异**（Helm 的 `artifacts` 同时覆盖制品与输出日志，即没有独立的 `builds`）。

**一次正常备份的输出长这样**（对照它判断卡在哪一步）：

```plaintext
Dumping database tables:
- Dumping table events... [DONE]
- Dumping table issues... [DONE]
...
Dumping repositories:
- Dumping repository abcd... [DONE]
Creating backup archive: <backup-id>_gitlab_backup.tar [DONE]
Deleting tmp directories...[DONE]
Deleting old backups... [SKIPPING]
```

最后一行 `Deleting old backups... [SKIPPING]` 不是 bug：只有配了保留期（2.5）才会真删旧备份。

## 2.4 参数矩阵

**组件名清单**（源码在 `lib/backup/manager.rb`，官方文档逐项列出）：`db`、`repositories`（**含 wiki**）、`uploads`、`builds`（CI job 输出日志）、`artifacts`、`pages`、`lfs`、`terraform_state`、`registry`、`packages`、`ci_secure_files`、`agent_plan_content`、`ci_catalog_bundles`、`external_diffs`；`SKIP=` 另接受两个特殊值 `tar`、`remote`。**Helm 的清单没有独立 `builds`**（其 `artifacts` 覆盖制品与日志）。

| 参数 | 作用 | 默认 | 必须知道的副作用 |
|------|------|------|------------------|
| `STRATEGY=copy` | 先把数据复制到临时目录再 `tar`/`gzip` | 直接 stream | 解 `file changed as we read it`；**副作用是最多多占 1X 磁盘**（每阶段会尽量清理，但大实例仍可能很痛） |
| `BACKUP=<id>` | 覆盖文件名里的 backup-id 段（产出 `<id>_gitlab_backup.tar`） | `<时间戳>_<版本>` | **自定义文件名与本地保留期清理互斥**（官方两处都写了警告）；对 rsync 增量搬运有益，配合 `GZIP_RSYNCABLE=yes` |
| `COMPRESS_CMD` / `DECOMPRESS_CMD` | 覆盖压缩/解压命令 | `gzip -c -1` / `gzip -cd` | 默认只对 **DB dump 与 blobs**（上传、制品、外部 MR diff）做 Gzip 快速压缩；在管道里用 → **必须输出 stdout**；产物名仍以 `.gz` 结尾；**非默认压缩库官方不做常规测试**，出问题官方要求先退回默认；`pigz`/`zstd` **不在 Linux package 里，要自装** |
| `GZIP_RSYNCABLE=yes` | 给 gzip 加 `--rsyncable` | 关 | **只有和自定义 `BACKUP=` 一起用才有意义**；某些发行版的 gzip 没有该选项，先 `gzip --help` 验 |
| `SKIP=<组件,...>` | 跳过组件 | 全备 | 组件名见上一段；不同安装形态可跳过的项略有差异 |
| `SKIP=tar` | 不生成 tar，把中间目录留着给别的备份软件取 | — | **配了对象存储备份时无法跳过 tar**（官方两处重复强调）；文件留在中间目录，**下次备份会覆盖它 → 同一时刻系统上只能有一份**，要及时搬走 |
| `SKIP=remote` | 生成 tar 但不上传远端 | — | 只用于"本地留一份"的排障或离线搬运 |
| `DIRECTORY=daily` | 在远端存储里分子目录 | — | **只对远端存储生效** |
| `GITLAB_BACKUP_MAX_CONCURRENCY` | 同时打包的项目仓库数 | **逻辑 CPU 数** | 多 storage 时调它来吃满 CPU |
| `GITLAB_BACKUP_MAX_STORAGE_CONCURRENCY` | 每个 storage 上同时打包的项目数 | `2` | 例：4 个 storage 想打满就设 `MAX_CONCURRENCY=4` + `STORAGE_CONCURRENCY=1` |
| `REPOSITORIES_STORAGES=` | 只备指定 storage | 全部 | 逗号分隔 storage 名（多 storage 场景的横向拆分） |
| `REPOSITORIES_PATHS=` / `SKIP_REPOSITORIES_PATHS=` | 只备/跳过指定项目或群组路径 | 全部 | 给**群组路径**＝含其**所有下级**项目；恢复侧同名选项语义见 3.5 |
| `INCREMENTAL=yes` + `PREVIOUS_BACKUP=<id>` | 仓库增量备份 | 全量 | **只有 repositories 支持增量**，其余组件仍是全量覆写；要在**新实例**上恢复这条链，基线那次全备**不能 SKIP 任何组件**；增量链上任一文件被生命周期策略删掉 → **整链不可恢复**（详见 2.6） |
| `CRON=1` | 无错时静默 | — | cron 场景用它防邮件刷屏；排障时换成 `--trace` |
| `REPOSITORIES_SERVER_SIDE=true`（Omnibus）/ `--repositories-server-side`（toolbox） | 让**托管该仓库的 Gitaly 节点**自己打包并流式写入对象存储，备份包里不再含仓库数据 | 由工具箱代拉代传 | 17.0 起支持；需先在**每台 Gitaly 节点**配 server-side 备份目标（`gitaly['configuration'].backup`）；**大仓库必须一起调 `buffer_size`**（2.10 #4）；`gitlab-backup` 仍会上传含还原所需元数据的 tar |
| `gitlab_rails['backup_archive_permissions']` | 改备份包权限 | `git:git` + `0600` | 默认就是"防同机其他用户读 GitLab 数据"；设 `0644` 变全局可读，交给第三方备份软件读时常用，**要在安全评审里记一笔** |
| `GITLAB_BACKUP_PG*` | **绕过 PgBouncer** 直连主库跑备份/恢复 | 用 `database.yml` | 见 2.9，用了 PgBouncer 不配这组就会出事 |

> **一个反直觉写法**：压缩命令要放在**命令前面**，不是后面。官方原话是 `gitlab-backup create COMPRESS_CMD="pigz -c --best"` 这种写法**不生效**（`doesn't work as intended`）。正确形式：

```bash
# 用 pigz 4 进程压缩（注意变量在 gitlab-backup 之前）
sudo COMPRESS_CMD="pigz --stdout --fast --processes 4" gitlab-backup create
# 恢复时 pigz 产出的是 gzip 格式，可以用默认 gzip 解；也可以指定
sudo DECOMPRESS_CMD="pigz --decompress --stdout" gitlab-backup restore BACKUP=<id>

# 目标端已有自动压缩（某些 NFS/快照盘）时干脆不压
sudo COMPRESS_CMD=tee gitlab-backup create
sudo DECOMPRESS_CMD=tee gitlab-backup restore BACKUP=<id>
```

**跳过组件 / 并发 / 指定范围**：

```bash
sudo gitlab-backup create SKIP=db,uploads           # 只备仓库等，DB 另有手段
sudo gitlab-backup create SKIP=registry,packages    # 这两个已在对象存储、由公司统一备

# 多 storage：吃满 CPU，但每个 storage 只跑 1 个仓库任务
sudo gitlab-backup create GITLAB_BACKUP_MAX_CONCURRENCY=4 GITLAB_BACKUP_MAX_STORAGE_CONCURRENCY=1

# 只备指定 storage / 指定路径（备份与恢复两侧都有这组参数）
sudo gitlab-backup create REPOSITORIES_STORAGES=storage1,storage2
sudo gitlab-backup create REPOSITORIES_PATHS=group-a,group-b/project-c SKIP_REPOSITORIES_PATHS=group-a/project-d

# Helm：给群组 A 备仓库、跳过项目 a2，其余组件不备
# REPOSITORIES_PATHS=group-a SKIP_REPOSITORIES_PATHS=group-a/project_a2 \
#   backup-utility --skip db,registry,uploads,artifacts,lfs,packages,external_diffs,terraform_state,ci_secure_files,agent_plan_content,ci_catalog_bundles,pages

# rsync 搬运友好（增量传输）：固定文件名 + rsyncable
sudo gitlab-backup create BACKUP=dump GZIP_RSYNCABLE=yes

# 不打 tar 的增量（供外部备份软件直接取中间目录）：注意此时**不能**再指定 PREVIOUS_BACKUP / BACKUP
sudo gitlab-backup create INCREMENTAL=yes SKIP=tar
```

## 2.5 定时任务与本地保留期

```bash
# root crontab：每天 02:00
0 2 * * * /opt/gitlab/bin/gitlab-backup create CRON=1
```

```ruby
# /etc/gitlab/gitlab.rb —— 本地旧备份保留 7 天（秒）
gitlab_rails['backup_keep_time'] = 604800
# 备份文件目录，默认 /var/opt/gitlab/backups
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"
```

```bash
sudo gitlab-ctl reconfigure
```

```plaintext
# 源码安装：git 用户的 crontab（sudo -u git crontab -e）
0 2 * * * cd /home/git/gitlab && PATH=/usr/local/bin:/usr/bin:/bin bundle exec rake gitlab:backup:create RAILS_ENV=production CRON=1
```

四条要点：

- `backup_keep_time`（源码装是 `backup.keep_time`）**只管本地文件**，并且是**下一次备份任务运行时**才删超期的旧包 —— 备份停了，清理也停了。
- 官方明说"GitLab 不清理第三方对象存储里的旧文件，因为你未必有列目录和删除权限"，远端保留要自己配生命周期策略（S3 lifecycle / GCS 保留策略）。
- cron **不备配置文件与 SSH host key**（官方原话警告），所以生产 cron 通常配两行：备份 + `tar /etc/gitlab`。
- 配了 `BACKUP=` 自定义文件名 → 清理逻辑失效；这是最常见的"磁盘被备份塞满"原因。

## 2.6 大实例：PG 超过 100 GB 就要换路线（官方三轨方案）

官方口径很直接：**`gitlab-backup` 用 `pg_dump`，不适合 100 GB 以上的数据库**（原文：`not appropriate for databases over 100 GB`，你**必须**选一个有原生可靠备份能力的 PostgreSQL 方案）。原因是备份里唯一"必须串行、又最耗时"的就是 DB dump，它跟数据量成正比；仓库与对象数据都能并行/旁路搬运。

官方为此专门写了一页 **参考架构级备份方案**（`backup_large_reference_architectures.md`），把它拆成**四轨**，各归各的工具：

| 轨道 | 官方做法 | 要点 |
|------|----------|------|
| **PostgreSQL** | 交给数据库自身：AWS Backup 备 RDS（**同时开持续备份 + 快照**以拿到 PITR）；GCP 用 Cloud SQL 每日备份（可留一年，事务日志默认 7 天支撑 PITR） | **跨 region 复制备份**：AWS 明说"备份只能还原到它所在 region" |
| **对象存储数据** | **推荐对象存储而不是 NFS** 存 blobs/registry 等；备份用云侧能力：AWS Backup 备 S3，或 GCS **Storage Transfer Service** 每日作业把各 GitLab 桶拷到备份桶 | 传输作业选 **`When to overwrite` = Never**（GitLab 对象按不可变设计，防被篡改后同步覆盖）、**`When to delete` = Never**（否则源站被误删/恶意删除时你没有任何 recover 路径） |
| **Git 仓库** | **Gitaly server-side backups**：在每个 Gitaly 节点配 `gitaly['configuration'].backup` 目标桶，然后在 **Rails 节点（跑 Puma 或 Sidekiq 的机器）**执行 `REPOSITORIES_SERVER_SIDE=true SKIP=db` | 见下方命令 |
| **配置与 secrets** | 若配置由外部系统（Secrets Manager + 跨 region 复制等）下发，备份跟着那套体系走；若只存在于部署内部，则按 2.8 抽取后**上传到权限更收紧的独立桶** | |

```bash
# 轨道 1：先做一次"仓库全备"（不含 DB），并记下输出的 backup ID
sudo gitlab-backup create REPOSITORIES_SERVER_SIDE=true SKIP=db
#   2024-02-22 02:17:47 UTC -- Backup 1708568263_2024_02_22_16.9.0-ce is done.   ← backup ID
# 检查：Gitaly 的备份桶与常规备份桶**都要有数据**（前者是仓库，后者是还原所需的 tar 元数据）

# 轨道 2：之后每天走增量（PREVIOUS_BACKUP 必填，但官方明说它的值目前并不会被真正使用，issue 429141）
sudo gitlab-backup create REPOSITORIES_SERVER_SIDE=true SKIP=db \
  INCREMENTAL=yes PREVIOUS_BACKUP=1708568263_2024_02_22_16.9.0-ce
```

```text
# 官方给的 cron 形态：每月 1 号全备，其余日期增量，限制恢复时要回放的增量段数
0 2 1    * * /opt/gitlab/bin/gitlab-backup create REPOSITORIES_SERVER_SIDE=true SKIP=db CRON=1
0 2 2-31 * * /opt/gitlab/bin/gitlab-backup create REPOSITORIES_SERVER_SIDE=true SKIP=db INCREMENTAL=yes PREVIOUS_BACKUP=<基线ID> CRON=1
```

**Helm 侧的同款方案**（工具箱里跑，注意**两处差异**：不支持 server-side + 增量组合（charts issue 3421），且要把所有非仓库组件显式跳过）：

```bash
kubectl exec <toolbox-pod> -it -- backup-utility --repositories-server-side \
  --skip db,builds,pages,registry,uploads,artifacts,lfs,packages,external_diffs,terraform_state,ci_secure_files
# cron 形态：把上面这些参数放进 gitlab.toolbox.backups.cron.extraArgs
```

**三条容易被忽略的对齐规则**（都来自官方那一页）：

- **对象存储备份的时间要 ≥ 数据库备份时间**，否则会出现"DB 里有记录、对象已不存在"的不一致窗口（原文：`Ensure that backups are performed at the same time or later than Cloud SQL backups`）。
- **桶同步（transfer job）会把已删除的文件也留在备份里**：还原后它们是"孤儿对象"，GitLab 用户看不到（DB 里没有对应记录），**只是白占存储**。可以用 `gitlab:cleanup:remote_upload_files` 清理**一部分**（该任务只覆盖部分文件类型），所以官方也给出"按日子目录"的替代法，代价是备份成本大幅上升。
- 迁移/恢复这类桶的清理是**不可逆**操作，先 `DRY_RUN`（`gitlab:cleanup:remote_upload_files`，见 2.10 #3）。

> **磁盘快照**同样是官方列出的替代策略之一，但两条限制要写进方案：多节点必须**每个节点都拍**（跨节点时间戳不一致 = 数据不一致），且官方明确该流程 **out of scope for GitLab Support**，出事时 Support 不会替你把快照拼回来。

## 2.7 备份上传到对象存储

注意分工：这一节解决的是**把 `gitlab-backup` 产出的那个 tar 送到异地**，**不等于**备了对象存储里的数据（那些数据本来就不进包，见 2.2、2.6）。

```ruby
# /etc/gitlab/gitlab.rb —— S3 / S3 兼容
gitlab_rails['backup_upload_connection'] = {
  'provider' => 'AWS',
  'region' => 'eu-west-1',
  'use_iam_profile' => true            # 或改用 aws_access_key_id / aws_secret_access_key
}
gitlab_rails['backup_upload_remote_directory'] = 'my.s3.bucket'
# 大文件分片上传阈值（字节），100 MB 起
gitlab_rails['backup_multipart_chunk_size'] = 104857600
```

```json
// 用 IAM Profile 时，跑备份的那台机器需要的最小策略
{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow",
  "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
  "Resource": "arn:aws:s3:::<backups-bucket>/*" }] }
```

服务端加密的三种模式，配置项不同，别抄错：

| 模式 | 配置 |
|------|------|
| **SSE-S3** | `gitlab_rails['backup_upload_storage_options'] = { 'server_side_encryption' => 'AES256' }` |
| **SSE-KMS** | 同上，`'server_side_encryption' => 'aws:kms'` + `'server_side_encryption_kms_key_id' => '<key ARN>'` |
| **SSE-C** | `gitlab_rails['backup_encryption'] = 'AES256'` + `gitlab_rails['backup_encryption_key'] = '<**必须正好 32 字节**>'`；含二进制字符时用 `gitlab_rails['env'] = { 'GITLAB_BACKUP_ENCRYPTION_KEY' => ... }` |

其他后端的实测坑（都来自官方故障说明）：

```ruby
# Google Cloud Storage：控制台建 Interoperability 密钥；CNAME 桶会撞证书校验问题
gitlab_rails['backup_upload_connection'] = {
  'provider' => 'Google',
  'google_storage_access_key_id' => 'Access Key',
  'google_storage_secret_access_key' => 'Secret'
  # 'path_style' => true   # CNAME 桶（foo.example.com）上传报 SSL 主机名不匹配时打开
}
gitlab_rails['backup_upload_remote_directory'] = 'my.google.bucket'

# Azure Blob（可省略 access key 改用 managed identity）
gitlab_rails['backup_upload_connection'] = {
  'provider' => 'AzureRM',
  'azure_storage_account_name' => '<ACCOUNT>',
  'azure_storage_access_key' => '<KEY>',
  'azure_storage_domain' => 'blob.core.windows.net'
}
gitlab_rails['backup_upload_remote_directory'] = '<CONTAINER>'
```

- **DigitalOcean Spaces 报 `400 Bad Request`** → 多半是启用了 `backup_encryption`，Spaces 不支持，注释掉该行。
- **非 AWS 的 S3 兼容端点（含自建 MinIO 一类）报 `411 Length Required`** → 官方建议把 `aws_signature_version` 从默认值降到 `2`（Fog 的已知问题）。
- 官方还提醒"并非所有 S3 服务商都与 Fog 完全兼容"，**上线前用一次真实备份验证上传与下载**，别只测凭据。

**上传到本地挂载盘（NFS/CIFS/SMB）用 Fog 的 `Local` provider**：

```ruby
gitlab_rails['backup_upload_connection'] = {
  :provider => 'Local',
  :local_root => '/mnt/backups'
}
# 挂载目录下的子目录；想直接放根目录就写 '.'
gitlab_rails['backup_upload_remote_directory'] = 'gitlab_backups'
```

三条限制：挂载后 `local_root` 必须**由 `git` 用户拥有**（CIFS/SMB 用 `uid=` 挂成 git），否则上传失败；**不要把 `backup_path` 和 `local_root` 设成同一路径** —— 官方解释得很直白：那样上传会因为"目标已存在同名文件"而失败，而**上传逻辑把这个已存在的文件当成"上传失败留下的残留文件"删掉**，于是你的备份被自己删了；另外官方一贯立场：**不建议用云文件系统（NFS 一类）做存储**，文件系统性能会直接拖垮 GitLab。

## 2.8 配置与 secrets：为什么不能进包

官方给的直接原因是安全语义：数据库里存着**2FA 与 CI/CD 加密变量的密文**，`gitlab-secrets.json` 里有**数据库加密密钥**，"把加密信息和它的密钥放在同一处，就失去了加密的意义"，所以工具故意不打包它。同时它提醒：**secrets 文件可能在升级后变化**，所以每次升级前除了数据备份，也要把 `/etc/gitlab` 一起归档。

最低限度要单独备份的文件：

| 安装形态 | 必须单独备份 |
|----------|--------------|
| Linux package | `/etc/gitlab/gitlab-secrets.json`、`/etc/gitlab/gitlab.rb`（外加 `/etc/gitlab/ssl`、`/etc/gitlab/trusted-certs`、SSH host keys） |
| 源码安装 | `/home/git/gitlab/config/secrets.yml`、`/home/git/gitlab/config/gitlab.yml` |
| Docker | 配置卷（按文档默认在 `/srv/gitlab/config`） |
| Helm chart | 按 charts 文档"Back up the secrets"流程 |

```bash
# 一条能直接抄的配置文件备份（权限与 secrets 同等重要）
sudo tar czf /var/opt/gitlab/backups/etc-gitlab-$(date +%F).tar.gz -C / etc/gitlab \
  && sudo chmod 600 /var/opt/gitlab/backups/etc-gitlab-$(date +%F).tar.gz
sudo gitlab-rake gitlab:doctor:secrets   # 校验当前密钥能否解开库里的密文
```

**丢了 secrets 会具体坏掉哪些功能**（官方 troubleshooting 逐项列出，比"2FA 进不来"更完整）：数据库里所有加密列都解不开，受影响的是 **CI/CD 变量**、**Runner 认证**、**Kubernetes / GCP 集成**、**自定义 Pages 域名及其证书**、**项目 Error Tracking**、**项目镜像（push/pull mirroring）**、**Integrations**、**Webhooks**、**Deploy tokens**。表现包括**流水线卡住（stuck jobs）**、**500 错误**、访问带集成/webhook 配置的仓库时报错。

```bash
sudo gitlab-rake gitlab:doctor:secrets   # 先确认"是不是 secrets 的锅"：列出无法解密的记录
```

处置动作（官方流程，顺序不能换）：**先做全量数据库备份**，再按下面顺序重置受影响的密文；官方明确警告**这些步骤会造成上述数据丢失**，Premium/Ultimate 客户应当**先开工单**。

```sql
-- sudo gitlab-rails dbconsole --database main
-- 1) 清掉无法解密的 CI 变量（可按 group/project 缩小范围）
DELETE FROM ci_group_variables;               -- DELETE FROM ci_group_variables WHERE group_id = <GROUPID>;
DELETE FROM ci_variables;                     -- DELETE FROM ci_variables WHERE project_id = <PROJECTID>;

-- 2) 清掉各类注册/认证令牌（最后一条会让 Runner 无法领 job，必须重新注册 Runner）
UPDATE projects            SET runners_token = null, runners_token_encrypted = null;
UPDATE namespaces          SET runners_token = null, runners_token_encrypted = null;
UPDATE application_settings SET runners_registration_token_encrypted = null;
UPDATE application_settings SET encrypted_ci_jwt_signing_key = null;   -- 可能连带打断 $CI_JWT_TOKEN
UPDATE ci_runners          SET token_encrypted = null;
UPDATE p_ci_builds         SET token_encrypted = null;                 -- 清掉排队中 job 的令牌

-- 3) 集成与 webhook 表：整表清空（这两句是"删光所有集成配置"，务必先确认根因）
TRUNCATE integrations, chat_names, issue_tracker_data, jira_tracker_data,
         slack_integrations, web_hooks, zentao_tracker_data, web_hook_logs_daily CASCADE;
```

另外 2FA：`disable user two-factor authentication` 一节要求**为全体用户关闭 2FA**（之后用户必须重新登记），所以"丢 secrets"的真实代价是**全公司重新绑定一次 2FA + 重注册所有 Runner + 重填所有 CI 变量**。这也是 2.8 那张"必须单独备份"表存在的理由。

> **换主机名的隐藏代价**：把备份恢复到**不同 FQDN** 的实例时，**所有 WebAuthn 设备（如 YubiKey）会被禁用** —— WebAuthn 注册与 origin 密码学绑定，官方要求用户重新注册。做异地容灾演练时这是必测项，不是理论问题。

## 2.9 PgBouncer：不绕过就出事（备份与恢复都是）

官方措辞很强硬：**不要通过 PgBouncer 连接来备份或恢复 GitLab**，这两个任务**必须绕过 PgBouncer 直连 PostgreSQL primary**，否则会造成 GitLab  outage。

```plaintext
# 走 PgBouncer 跑备份/恢复时你会看到：
ActiveRecord::StatementInvalid: PG::UndefinedTable
# 并且每次备份之后开始批量 500，PostgreSQL 侧日志同时出现"表不存在"：
ERROR: relation "tablename" does not exist at character 123
```

**根因不是权限、不是网络**：`pg_dump` 为了应对 **CVE-2018-1058**，会**把 search path 置空**并在每条 SQL 里显式写 schema；而 PgBouncer 的 **transaction pooling** 复用连接，这个"清空 search path"就导致默认 `public` schema 查找失败，于是表和列看起来都不存在。

两条解法（官方给的）：

```bash
# ① 用 GITLAB_BACKUP_ 前缀的环境变量临时覆盖 DB 连接（不改 database.yml）
sudo GITLAB_BACKUP_PGHOST=192.168.1.10 GITLAB_BACKUP_PGPORT=5432 /opt/gitlab/bin/gitlab-backup create

# 多库部署：变量名里带库名，PG* 部分不变
sudo GITLAB_BACKUP_MAIN_PGHOST=192.168.1.10 GITLAB_BACKUP_CI_PGHOST=192.168.1.12 /opt/gitlab/bin/gitlab-backup create
```

可用前缀项：`GITLAB_BACKUP_PGHOST`、`PGPORT`、`PGUSER`、`PGPASSWORD`、`PGSSLMODE`、`PGSSLKEY`、`PGSSLCERT`、`PGSSLROOTCERT`、`PGSSLCRL`、`PGSSLCOMPRESSION`（多库时写成 `GITLAB_BACKUP_<db>_PG*`）。

② 长期方案是把跑备份的那台节点**重构成直连 PostgreSQL primary**（PgBouncer 文档里的 bypass 流程）。**恢复同理**：恢复前也要确认连接不走 PgBouncer。

> 这条与 3.1 前提 2（目标实例必须已经能跑）合起来看：**PgBouncer 环境的恢复失败，八成是连接走错**，而不是备份包坏了。排障顺序里应该先验证 `GITLAB_BACKUP_PGHOST` 生效（用 `gitlab-psql` 直连主库对比 `pg_stat_activity` 里的来源 IP）。

## 2.10 备份失败：五个真实报错原文

排障时先 `--trace`（比 `CRON=1` 罗嗦但能看到是哪一步炸），再对照下面五条。

| # | 报错原文（节选） | 根因 | 处置 |
|---|------------------|------|------|
| 1 | `tar: ... file changed as we read it` | 边读边写（stream 策略读到正在变更的数据） | 改用 `STRATEGY=copy`，代价是最多多占 1 倍磁盘 |
| 2 | `gzip: stdout: Input/output error` + `Backup failed` | **磁盘不够**：官方说默认策略常需要"**实例体积一半**"的空闲空间；另一诱因是 **NFS 挂载 `timeout` 选项**（默认 `600`，调小就会触发该错） | 清/扩备份盘或改走 2.6；NFS 别把 timeout 调小 |
| 3 | `Problem: <class 'OSError: [Errno 36] File name too long: ...` | `uploads` 表/文件系统里有**超过 246 字符（含扩展名）**的文件名；历史 bug 导致父资源删除后对象存储里残留文件 | 见下方三步流程 |
| 4 | `backup repository: manager: write bundle: ... error: pack-objects died`（**server-side 备份大仓库**） | 真凶不是 Git：Gitaly 用 **multipart** 直接流式上传，默认 part **5 MB**、S3 上限 **10,000 part**，即 bundle **约 50 GB** 封顶，超限被对象存储中止、断管报成 `pack-objects died` | 增大 Gitaly `backup.buffer_size`（上限 = `buffer_size × 10,000`；`10485760` → 约 100 GB） |
| 5 | 恢复阶段 `ERROR: permission denied to create extension "pg_stat_statements"` / `must be owner of view pg_stat_statements` | dump 里带了 `public` schema 下的扩展创建语句，而恢复用**非 superuser** | 见下方"把扩展挪出 public" |

**#3 的处理（官方流程，可能丢数据，务必先备份）**：

```bash
# 1) 清掉"对象存储里有、DB 的 uploads 表里没有"的孤儿上传（先看清单，再去掉 DRY）
sudo gitlab-rake gitlab:cleanup:remote_upload_files RAILS_ENV=production
sudo gitlab-rake gitlab:cleanup:remote_upload_files RAILS_ENV=production DRY_RUN=false
```

```sql
-- 2) 找出文件名超 246 字符的 uploads 记录（官方建议按 0-10000 分批，避免大实例一次全表扫）
CREATE TEMP TABLE uploads_with_long_filenames AS
SELECT ROW_NUMBER() OVER(ORDER BY id) row_id, id, path
FROM uploads AS u
WHERE LENGTH((regexp_match(u.path, '[^\\/:*?"<>|\r\n]+$'))[1]) > 246;
CREATE INDEX ON uploads_with_long_filenames(row_id);

SELECT id, path,
       (regexp_match(path, '[^\\/:*?"<>|\r\n]+$'))[1] AS current_filename,
       CONCAT(LEFT(SPLIT_PART((regexp_match(path, '[^\\/:*?"<>|\r\n]+$'))[1], '.', 1), 242),
              COALESCE(SUBSTRING((regexp_match(path, '[^\\/:*?"<>|\r\n]+$'))[1] FROM '\.(?:.(?!\.))+$'))) AS new_filename
FROM uploads_with_long_filenames WHERE row_id > 0 AND row_id <= 10000;

-- 3) 更新时**先包在 BEGIN/ROLLBACK 里核对**，确认无误再把 COMMIT 换上（官方给的即"安全试跑"写法）
--    改完 DB 还要：a) 用 pg_dump + grep 找出引用这些路径的其它表并同步改名；
--                 b) **手工把文件系统/对象存储里的文件也改成新名字**；然后重跑备份。
BEGIN;  UPDATE uploads SET path = ... FROM uploads_with_long_filenames ...;  ROLLBACK;
```

**#4 的配置**（每台 Gitaly 节点都要改，然后 reconfigure + restart gitaly）：

```ruby
gitaly['configuration'] = {
  backup: {
    go_cloud_url: 's3://<bucket>?region=<region>',
    buffer_size: 10485760,     # 10 MB × 10,000 = 约 100 GB bundle 上限
  },
}
```

```bash
# 先找出真正的大 pack，决定 buffer_size 要开多大
sudo find /var/opt/gitlab/git-data -name "*.pack" -size +10G -ls
sudo gitlab-ctl reconfigure && sudo gitlab-ctl restart gitaly
```

**#5 的预防**（一次性，之后备份/恢复都干净）：

```sql
CREATE SCHEMA adm;
CREATE EXTENSION pg_stat_statements SCHEMA adm;      -- 新装：直接放非 public
-- ALTER EXTENSION pg_stat_statements SET SCHEMA adm; -- 已装在 public：挪走
SELECT * FROM adm.pg_stat_statements LIMIT 0;         -- 查询要带 schema 前缀，或 SET search_path TO public,adm;
```

已经产出的备份包也能救：抽出包内 `db/database.sql.gz`，解压删掉 `CREATE EXTENSION IF NOT EXISTS pg_stat_statements ...` 与对应 `COMMENT ON EXTENSION ...` 两行，重新压缩回包（官方给的即这套手工改包流程）。

---

# 3. 恢复：前提比命令重要

恢复流程的失败绝大多数不是命令敲错，而是前提没满足。先把 3.1 当检查表用。

## 3.1 七项前提（官方逐条给出）

| # | 前提 | 细节与踩坑表现 |
|---|------|----------------|
| 1 | **备份目录有足够空闲空间** | 解包前会检查，需要**≥ 归档大小 3 倍**的空间；不够或读不到就直接中止。用未打包备份（`SKIP=tar`）时**不跑这个检查**，可用 `BACKUP_SKIP_STORAGE_CHECK` 跳过 |
| 2 | **目标实例本身已经能跑** | 执行恢复的 `git` 用户通常无权建/删 `gitlabhq_production`，所以必须"先装好一个可用的 GitLab 再恢复"，不是"裸机直接灌" |
| 3 | **目标实例不能已有数据** | 官方按数据类型分别说明行为：**PG 数据自动清除**（但**恢复 SQL 数据会跳过属于 PostgreSQL 扩展的 view**，这就是 2.10 #5 那类 `pg_stat_statements` 报错的来源）；**Git 仓库同名已存在 → `repository already exists` 直接失败**（issue 118459）；**文件系统数据会先尽量移动到单独目录**再还原；**对象存储数据不会自动清，必须手工清桶**，否则留孤儿数据。结论：生产→预发这类自动化恢复，官方建议**用与备份同版本的全新安装** |
| 4 | **版本与类型必须完全一致** | 只能恢复到**创建该备份时完全相同版本、相同 CE/EE 类型**的实例（例：`15.1.4-ce`）。不一致就要先降级或升级 |
| 5 | **secrets 必须先恢复** | 库里存的是**数据库加密密钥 + CI/CD 变量 + 2FA 相关变量**；缺了就是"恢复显示成功、但 2FA 用户进不来、Runner 登录不上"。各形态路径：**Omnibus** `/etc/gitlab/gitlab-secrets.json`、**Docker**（若把 `/etc/gitlab` 挂在 `/srv/gitlab/config`）`/srv/gitlab/config/gitlab-secrets.json`、**Helm** 走 charts 的 restore secrets 流程（且 **Helm 的 secrets 可以转换成 Linux package 格式**）、**源码** `/home/git/gitlab/.secret`。**注意官方两页口径不一致**：备份页要求备 `config/secrets.yml`+`config/gitlab.yml`，恢复页要求还原 `.secret`。实操按"整个 config 目录一起还原"处理，不要只挑一个文件 |
| 6 | **部分配置与数据耦合** | 原环境有三个仓库存储（`default`、`my-storage-1`、`my-storage-2`）→ 目标环境的配置里**必须至少存在这些存储名**；用本地存储做的备份**恢复到本地存储**，即使目标已配对象存储 —— 迁对象存储要在恢复前后单独做 |
| 7 | **恢复目录是挂载点时必须为空** | 否则 GitLab 会尝试"移动目录"再写，直接报错（NFS 场景常见） |

## 3.2 Omnibus 恢复全流程

官方这一节开头写明三个**前置假设**：已装**与备份完全同版本同类型（CE/EE）**的 GitLab、**至少执行过一次 `gitlab-ctl reconfigure`**、GitLab 当前**正在运行**。也就是说恢复不是"裸机灌数据"。编排顺序如下（注意 **ANALYZE 夹在 restore 与 reconfigure 之间**，这是官方原文要求）：

```bash
# 1) 备份文件放进 gitlab_rails[backup_path]（默认 /var/opt/gitlab/backups），且属主是 git
sudo cp 11493107454_2018_04_25_10.6.4-ce_gitlab_backup.tar /var/opt/gitlab/backups/
sudo chown git:git /var/opt/gitlab/backups/11493107454_2018_04_25_10.6.4-ce_gitlab_backup.tar

# 2) 只停"连着数据库的进程"，其余保持运行
sudo gitlab-ctl stop puma
sudo gitlab-ctl stop sidekiq
sudo gitlab-ctl status            # 必须核对：这两项真的停了

# 3) 确认 3.1 前提全满足；把原实例的 secrets 拷过来之后，**再跑一次 reconfigure**
sudo gitlab-ctl reconfigure

# 4) 恢复（BACKUP= 后面**不带** _gitlab_backup.tar）—— 这一步会覆盖当前数据库内容
sudo gitlab-backup restore BACKUP=11493107454_2018_04_25_10.6.4-ce

# 5) 在 reconfigure 与放流量之前，补齐优化器统计（见 3.4，逐库执行）
#    SET statement_timeout = 0; ANALYZE VERBOSE;

# 6) 在 PostgreSQL 节点再跑一次 reconfigure，然后启动
sudo gitlab-ctl reconfigure
sudo gitlab-ctl start

# 7) 体检（顺序也是官方给的：check → secrets 解密 → 三类上传一致性）
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rake gitlab:doctor:secrets      # secrets 被还原过、或恢复到另一台机器时必查
sudo gitlab-rake gitlab:artifacts:check
sudo gitlab-rake gitlab:lfs:check
sudo gitlab-rake gitlab:uploads:check
```

版本不匹配时中止，报错原文：

```plaintext
GitLab version mismatch:
  Your current GitLab version (16.5.0-ee) differs from the GitLab version in the backup!
  Please switch to the following version and try again:
  version: 16.4.3-ee
```

> **PgBouncer 场景**：备份与恢复都**必须绕过 PgBouncer 直连 primary**（`GITLAB_BACKUP_PGHOST` 一类前缀变量，或把该节点重构成直连），否则会先看到 `ActiveRecord::StatementInvalid: PG::UndefinedTable`、随后持续 500 与 `relation "tablename" does not exist`。详见 2.9。

## 3.3 Docker / Helm / 源码安装的恢复差异

| 形态 | 必须知道的差异 |
|------|----------------|
| **Docker** | 命令都从宿主机 `docker exec -it <容器> ...` 进容器跑：`gitlab-ctl stop puma`/`stop sidekiq`/`status` → `gitlab-backup restore BACKUP=<id>` → **先补统计信息再** `docker restart <容器>` → `gitlab-rake gitlab:check SANITIZE=true` |
| **Docker Swarm** | **坑**：恢复会停 Puma，容器的健康检查随即失败，** Swarm 会在恢复中途把容器重启**。官方解法是恢复期间**临时关掉 healthcheck**：`docker-compose.yml` 里 `healthcheck: disable: true` 后 `docker stack deploy --compose-file docker-compose.yml mystack`（omnibus  issue 6846） |
| **Helm chart** | 不走本文流程，按 **charts 文档** "Restoring a GitLab installation" 执行（toolbox 里 `backup-utility --restore -t <backup-id>`）；secrets 也按 charts 的 restore secrets 流程还原 |
| **源码安装** | 备份目录在 `gitlab.yml` 的 `backup.path`（默认 `/home/git/gitlab/tmp/backups`，属主 `git`）；`sudo service gitlab stop` → `sudo -u git -H bundle exec rake gitlab:backup:restore RAILS_ENV=production` → 需要时还原 `.secret` → 补统计信息 → `sudo service gitlab restart` |

源码安装恢复的典型输出（**`[SKIPPING]` 是常态**，不是"数据丢了"）：

```plaintext
Unpacking backup... [DONE]
Restoring database tables:
-- create_table("events", {:force=>true})
  -> 0.2231s
[...]
- Loading fixture events...[DONE]
- Loading fixture protected_branches...[SKIPPING]
- Loading fixture users...[DONE]
- Loading fixture web_hooks...[SKIPPING]
Restoring repositories:
- Restoring repository abcd... [DONE]
- Object pool 1 ...
Deleting tmp directories...[DONE]
```

最后一行 `Object pool 1 ...` 值得注意：**恢复侧会体现 hashed object pool**，与 2.2 里"fork 靠 pool 仓库去重、备份时 pool 数据会被复制进每个 fork"是同一条链路的两侧。

## 3.4 恢复后必做：统计信息与一致性校验

数据库灌进去之后**不要立刻放流量**。`pg_dump` 恢复出来的库不带优化器统计，查询计划会选错，UI 计数也会看起来不对。官方要求在重启应用进程/恢复流量之前：

```sql
-- 对每个恢复出来的库单独执行（有独立 CI 库时也要跑一遍），同一会话里
SET statement_timeout = 0;
ANALYZE VERBOSE;

-- 可选：还要准备 visibility map（index-only scan 性能相关）时
-- 注意：VACUUM 比 ANALYZE 慢得多，且产生大量 IO/WAL，按你的恢复窗口决定
VACUUM (VERBOSE, ANALYZE);
```

```bash
sudo gitlab-rake gitlab:doctor:secrets      # 密文能否被当前 secrets 解开（换机恢复必查）
sudo gitlab-rake gitlab:artifacts:check     # 制品一致性
sudo gitlab-rake gitlab:lfs:check           # LFS 对象一致性
sudo gitlab-rake gitlab:uploads:check       # 上传附件一致性
sudo gitlab-rake gitlab:check SANITIZE=true # 总体配置与仓库健康
```

官方补充三条细节：

- **要逐个数据库重复做**（有独立 `ci` 库就两个库各跑一遍）。
- 连接时**用有权 ANALYZE 还原出来的表的角色**；Omnibus 下就是走 `gitlab-rails dbconsole`（或 `sudo -u gitlab-psql` 直连 socket），不要用只有 SELECT 权限的账号。
- **跑完要看输出里的错误与被跳过的表**再放流量。`ANALYZE` 与 `VACUUM` 要**分开、在事务块之外**执行；库级 `ANALYZE` 会带上分区父表，而 autovacuum 只分析分区、不会分析父表 —— 这正是"恢复后某些页面极慢"的隐藏原因。**推迟 vacuum 的代价**是 index-only scan 需要大量回表读 heap，直到 vacuum 完成才恢复，所以要按你的恢复窗口决定现在做还是放流量后做。
- 官方也承认"恢复后自动 ANALYZE"还只是一个 feature request（issue 276184）。

**抽样验证**（演练脚本里必须有一段是"业务可读"的，而不是只看到 `[DONE]`）：

```bash
# 登录、仓库可读可写、MR 可访问、CI 可跑
git clone git@$GITLAB_HOST:group/demo.git && cd demo && git push
glab api "projects/group%2Fdemo" | jq '{id, last_activity_at, statistics: .statistics.repository_size}'
glab api "projects/group%2Fdemo/merge_requests?state=opened&per_page=3" | jq -r '.[].iid'
glab api -X POST "projects/group%2Fdemo/ref/main/pipeline" | jq '.status'
```

## 3.5 只恢复一部分：仓库级选项与"单项目"变通

**先记住结论：实例恢复任务不支持只恢复一个项目/群组。** 官方的变通做法固定四步：

1. 装一台**与备份同版本**的临时 GitLab；
2. 把完整备份恢复到这台临时实例；
3. 在临时实例上用**项目导出 / 群组导出**把要的那份导出；
4. 回到生产实例导入，然后销毁临时实例。

（该能力的需求在 issue #17517 里挂着，别指望短期内有原生 `restore --project`。）

仓库层面能"挑"，但**只挑 Git 仓库，不挑数据库条目**：

```bash
# 只恢复指定 storages
sudo gitlab-backup restore BACKUP=<id> REPOSITORIES_STORAGES=storage1,storage2

# 恢复 group-a 全部仓库 + group-b/project-c，并排除 group-a/project-d
sudo gitlab-backup restore BACKUP=<id> \
  REPOSITORIES_PATHS=group-a,group-b/project-c \
  SKIP_REPOSITORIES_PATHS=group-a/project-d

# 全量恢复但跳过某项目的仓库
sudo gitlab-backup restore BACKUP=<id> SKIP_REPOSITORIES_PATHS=group-a/project-d
```

| 语义 | 官方口径 |
|------|----------|
| 传了 `REPOSITORIES_PATHS` / `SKIP_REPOSITORIES_PATHS` | **只恢复指定仓库**，且**不会删除实例上已有仓库**，因此可以连续跑多次互不清场 |
| 两者都不传（全量恢复） | **会删除不在该备份里的仓库**（例如备份之后新建项目的残留仓库） |
| 群组路径 | 指定群组路径 = 该群组及其**所有下级**项目的仓库 |
| 用 `SKIP=db` 做的仓库备份 | **单独拿它无法在新实例上恢复指定仓库** —— 数据库条目不在里面（这正是 2.6 大实例方案里"仓库走 server-side、DB 走原生备份"要靠**两条链路一起还原**的原因） |
| 版本要求 | **17.1 及更早存在竞态会导致数据丢失**（影响用 hashed object pools 的 fork 仓库），官方要求**只用 17.2+ 恢复备份** |
| 19.4 变化 | `SKIP_REPOSITORIES_PATHS` **不再删除**被排除的仓库 |

**server-side 备份怎么恢复**：如果备份是 server-side 方式产的，恢复**默认就走 server-side 机制**（Gitaly 读 backup manifest，**按顺序应用每一个增量**）；前提仍是每台 Gitaly 节点已配好 server-side 备份目标。

```bash
sudo gitlab-backup restore BACKUP=11493107454_2018_04_25_10.6.4-ce            # Omnibus / 源码形态
kubectl exec <toolbox-pod> -it -- backup-utility --restore -t <backup-id> --repositories-server-side
# 用 cron-based backup 的话，把 --repositories-server-side 加进 extraArgs
```

**增量备份怎么恢复**：`gitlab-backup` 产的增量包是**自包含**的（它把从最初全备起的所有步骤都塞进包里），所以**恢复命令与普通包完全一样**；**server-side 增量**则是各段对象分开存放，靠 Gitaly 顺序回放 —— 于是"链上少一段"直接不可恢复（2.6 的红线）。

## 3.6 恢复选项速查

| 场景 | 做法 |
|------|------|
| 备份目录里有多份 | 必须 `BACKUP=<backup-id>` 指定；不指定时，若存在 `SKIP=tar` 产出的未打包备份，会**直接用那份** |
| 脚本化 / 无人值守 | `sudo GITLAB_ASSUME_YES=1 gitlab-backup restore`（`force=yes` 同样关闭确认提示）。**它关掉的正好是三个最危险的确认点**：① 开了 *Write to authorized_keys* 时、删除并重建 `authorized_keys` 之前；② 恢复数据库、**删除现有所有表**之前；③ 数据库 schema 恢复报错后、是否继续。所以 `GITLAB_ASSUME_YES` 只能用在"目标机器本来就该被覆盖"的演练/预发机上 |
| 空间检查误停 | `sudo BACKUP_SKIP_STORAGE_CHECK=true gitlab-backup restore BACKUP=<id>`（**19.5 引入**；官方限定"确认存储足够后再跳"） |
| 只跳过某些组件 | `SKIP=db,uploads,builds,artifacts,lfs,terraform_state,registry,packages,pages,repositories` |
| 恢复 server-side 备份 | 先在 Gitaly 配 server-side 备份目标，再照常 `gitlab-backup restore BACKUP=<id>`；Helm 走 `backup-utility --restore -t <id> --repositories-server-side` |
| 增量备份恢复 | 见官方"Restoring an incremental repository backup"；前提是**基线是从一次未跳过组件的全备开始**，且**增量链文件一个都不能少** |
| PgBouncer | **不要走 PgBouncer 连接做备份或恢复**，用 `GITLAB_BACKUP_PG*` 变量直连主库，见 2.9 |

## 3.7 恢复阶段的成功报错速查

恢复里有几类**看着像失败、其实成功**，以及几类"提示成功、其实数据不可用"的，全部来自官方 troubleshooting，按现象查：

| 现象/报错原文 | 真实结论 | 动作 |
|----------------|----------|------|
| `ERROR: must be owner of extension pg_trgm` / `btree_gist` / `plpgsql`；`WARNING: no privileges could be granted for "public"` | **备份已成功恢复**（官方明说 in spite of these messages）。原因：Rake 任务以 `gitlab` 用户跑，它**不是数据库 superuser**，会去改自己没有权限的对象 | 记录、继续；不必重跑 |
| `ERROR: permission denied to create extension "pg_stat_statements"` / `must be owner of view pg_stat_statements`（清理阶段炸） | dump 里带了 `public` schema 的扩展语句 | 见 2.10 #5：扩展挪到别的 schema，或手工改 `db/database.sql.gz` |
| `restore: pipeline: 1 failures encountered ... setting custom hooks: ... read .../custom_hooks/xxx.d: is a directory` | **Git server hook 用 15.10 及更早的方式做**（在 GitLab 管理路径之外建 symlink）+ 版本 ≥15.11 | 按 15.11+ 的方式重做 Git server hooks（官方 `administration/server_hooks.md`），再恢复。与本文 2.2 那条"server/file hooks 不进备份包"是同一条链路的两侧 |
| 恢复报告成功，**但仓库全是空的** | 装了 **`fapolicyd`**（文件访问策略守护）时会出现；官方指向 Gitaly troubleshooting 的 "repositories are shown as empty after a GitLab restore" | 按 Gitaly 那节处理 fapolicyd 规则；这是"演练通过判定"必须包含仓库可读的原因之一（3.8） |
| 容器镜像全不见了 | 源环境**启用了 registry、目标没启用**时，**registry 不会被恢复** | 先在目标启用 container registry，**再**恢复 |
| 恢复后 push 镜像失败，registry 日志 `filesystem: mkdir /var/opt/gitlab/gitlab-rails/shared/registry/...: permission denied` | 恢复以**非特权 `git` 用户**跑，无法给 registry 文件设正确属主（issue 62759） | `sudo chown -R registry:registry /var/opt/gitlab/gitlab-rails/shared/registry/docker`（自定义路径就 chown 那个路径） |

> 这张表的价值在于**把"恢复成功"定义成一句可判定的话**：`[DONE]` 只说明任务跑完，不代表仓库可读、镜像可推、密文可解。所以 3.8 的演练脚本里，**校验段必须占一半以上**。

## 3.8 每季度演练脚本（可直接进 cron + 工单）

```bash
#!/usr/bin/env bash
# 季度恢复演练：在同版本隔离机上恢复最近一次备份并做业务抽样
set -euo pipefail
GL_VER="$(cat /opt/gitlab/embedded/service/gitlab-rails/VERSION)"   # 与备份文件名里的版本段核对（含 -ee/-ce 后缀）
BACKUP_DIR=/var/opt/gitlab/backups
ARCHIVE="$(ls -1t ${BACKUP_DIR}/*_gitlab_backup.tar | head -1)"
ID="$(basename "$ARCHIVE" _gitlab_backup.tar)"
echo "Restoring: $ID"

# 前提自检：空间 ≥ 3 倍归档
need=$(( $(stat -c%s "$ARCHIVE") * 3 ))
free=$(( $(df -B1 --output=avail "$BACKUP_DIR" | tail -1) ))
[ "$free" -ge "$need" ] || { echo "free space $free < required $need"; exit 1; }

sudo gitlab-ctl stop puma; sudo gitlab-ctl stop sidekiq; sudo gitlab-ctl status
sudo GITLAB_ASSUME_YES=1 gitlab-backup restore BACKUP="$ID"

# 官方顺序：先补统计信息，**再** reconfigure + start（放流量之前）
sudo -u gitlab-psql /opt/gitlab/embedded/bin/psql -h /var/opt/gitlab/postgresql \
  -d gitlabhq_production -c "SET statement_timeout = 0; ANALYZE VERBOSE;" | tail -20
# 有独立 ci 库时同理：-d gitlabhq_ci_production
sudo gitlab-ctl reconfigure
sudo gitlab-ctl start

# 校验（含 3.7 的三类隐性失败：仓库可读、镜像可推、密文可解）
sudo gitlab-rake gitlab:doctor:secrets
sudo gitlab-rake gitlab:artifacts:check
sudo gitlab-rake gitlab:lfs:check
sudo gitlab-rake gitlab:check SANITIZE=true | tail -30
echo "DRILL DONE: $ID"
```

> **演练要记三个数**：备份完成时刻（RPO）、恢复可用时刻（RTO）、抽样通过率。这三个数是"备份策略是否达标"的唯一证据，也是审批升级窗口时长时最有说服力的历史依据。

---

# 4. 升级：路径、检查点、暂停、回滚

## 4.1 停靠站规则（17.5 之后是可预测的）

官方规则两条：**① 必须按 upgrade path 逐站停靠；② 每站之间要等后台迁移跑完再走下一站。**

```text
自 GitLab 17.5 起，必需停靠站固定在每个大版本的 .2 / .5 / .8 / .11（即 x.2.z、x.5.z、x.8.z、x.11.z）。
```

| 大版本 | 必需停靠站 |
|--------|------------|
| **19** | 19.2、19.5、19.8、19.11 |
| **18** | 18.2、18.5、18.8、18.11 |
| **17** | 17.3.7 → 17.5.5 → 17.8.7 → **17.11.7**（都要该 minor 的**最新补丁版**；另有条件站：`ci_pipeline_messages` 表很大的实例需先经过 17.1.8） |
| **16** | 16.11.10、16.7.10、16.3.9（条件站：16.2.11 大流水线变量历史、16.1.8 包仓库里有 npm 包、16.0.10 用户量大） |
| **15** | 15.11.13、15.4.6、15.0.5（条件站：15.1.6 多 Web 节点） |

查路径与补丁号：

- **Upgrade Path tool**（支持团队维护）：`https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/`，输入当前版本与目标版本，列出必停点。
- 补丁版本清单在**包仓库**里查：`https://packages.gitlab.com/ui/browse/gitlab/gitlab-ee/...`，例：`gitlab-ee/ubuntu/noble/pool/main/g/gitlab-ee` 搜 `18.2`。
- Helm chart 另有**版本映射表**（chart 版本 ↔ GitLab 版本）。

> **升级前必查的一件事**：官方要求"如果 upgrade path 包含多个版本，**每次都必须显式指定包版本**"。`apt install gitlab-ee` / `dnf install gitlab-ee` 会直接装最新，**跳过停靠站 = 数据迁移路径未验证**，这是最常见的"升级后迁移失败无法回退"成因。

## 4.2 升级前要收集的信息

官方给的规划清单，照抄成变更评审的目录：

1. 读发布与维护政策；2. 读目标区间的 **upgrade notes**（逐版本）；3. 检查**操作系统与目标 GitLab 版本兼容**；4. 用 Geo 的话要额外读 Geo 升级文档 + 为**每个站点**（主与每个备）分别写升级与回滚计划 + 查 PG 升级对 Geo 的特别步骤；5. 用 4.1 确定停靠站；6. 跑**升级健康检查**（含数据库检查，"有些数据库问题要花时间修"）；7. 写升级计划（能否零停机 + 回滚步骤）；8. **在生产环境的克隆上演练**，顺便量出长耗时迁移要多久。

## 4.3 升级健康检查：UI 两个页面 + 四条命令

```bash
sudo gitlab-rake gitlab:check                 # 总体配置
sudo gitlab-rake gitlab:doctor:secrets        # 密文可解密（secrets 变了会在这里暴露）
sudo gitlab-rake gitlab:db:diagnostics:search_path   # 出错时退出码 1
sudo gitlab-rake gitlab:geo:check             # 用 Geo 时，主与每个 secondary 都要跑
```

**Admin → Monitoring → Database pre-upgrade checks**：

- 在 **Schema health check** 区点 **Run schema check**。它是**后台 job**（所以 Sidekiq 必须在跑），结果**保留一周**，官方要求**临升级前再跑一次**。
- 它会报告缺失的表/索引/外键/序列或 sequence owner 错误 —— **没有自动修复工具，官方要求先联系 GitLab Support**。
- 同一页的 **Search path** 检查：如果 search path 里**多个 schema 都含 GitLab 对象**，迁移可能改错 schema，也要先处理。

**Admin → Monitoring → Database diagnostics → Vacuum information**：

- 找带 **Anti-wraparound** 徽章的 vacuum。它**不会自己取消**，迁移会等它的锁从而失败。**发现就不要开始升级，也不要停它，等它跑完**。
- 这一区显示"当前正在跑"的 vacuum，**开始升级前要再看一次**（不是只看一次记录）。

UI 冒烟清单（官方逐条列出，检查升级前后都过一遍）：用户能登录、项目列表可见、Issue/MR 可访问、能 clone、能 push；CI 侧：Runner 能领到 job、镜像能推能拉；用 ES 的验证搜索成功；可选：预先下载包以确认仓库可达并缩短安装时间。

## 4.4 依赖组件的升级顺序

| 组件 | 顺序与原因（官方口径） |
|------|------------------------|
| **外部 Gitaly** | **先升 Gitaly，再升 GitLab**。否则应用服务器上的 gRPC client 会发旧 Gitaly 不支持的 RPC |
| **GitLab Runner** | 升到与目标 GitLab **相同版本**（升级前的准备步骤里就写了） |
| **GitLab Agent for Kubernetes** | 升到匹配的新 GitLab 版本 |
| **Elasticsearch（高级搜索）** | 升级前确认高级搜索迁移已完成（`gitlab:check` 会报告 pending 项，另有 rake，见下）；**升级后可能要再升 ES 以恢复兼容性**，而官方声明 ES 升级不在支持范围 |
| **PostgreSQL** | 按目标版本要求；Geo 实例有单独的数据库升级文档 |

```bash
# 高级搜索 pending 迁移（官方任务名）
sudo gitlab-rake gitlab:elastic:list_pending_migrations
```

## 4.5 暂停 CI/CD：为什么以及怎么做

官方理由很具体：升级期间 Runner 还在处理 job 时，**trace 更新会失败**（回来后多数能自愈，不能自愈的按错误决定重试或终止），并且 **Runner 只尝试上传制品 3 次，之后 job 直接失败**。

```bash
# 1) 先 pause runners（Admin → Runners 或 Runner 侧配置）
# 2) 用 nginx 拦掉 job 领取，阻止新 job 起跑
sudo vi /etc/gitlab/gitlab.rb
```

```ruby
nginx['custom_gitlab_server_config'] = "location = /api/v4/jobs/request {\n deny all;\n return 503;\n}\n"
```

```bash
sudo gitlab-ctl reconfigure
# 3) 等所有 job 结束（Admin → CI/CD jobs 或 glab api jobs?status=running）
glab api "jobs?scope=running&per_page=100" | jq -r '.[] | [(.id|tostring),.pipeline.id,.name] | @tsv'

# 4) 升级完成后：unpause runners，并**还原**上面的 gitlab.rb 改动 + reconfigure
```

> 顺序细节：**先 pause runner，再 503 拦 jobs/request，再等存量 job 收尾**。反过来做会出现"Runner 已经能领 job 但你又被 nginx 挡住"，日志里全是 503 重试，白白拉长窗口。

## 4.6 执行升级

```bash
# Ubuntu/Debian：显式指定版本（-ee.0 后缀以包仓库里的实际版本串为准）
sudo apt update && sudo apt install gitlab-ee=18.5.3-ee.0
# 可用版本枚举，别凭记忆猜补丁号
apt madison gitlab-ee | head -20

# RHEL/CentOS/Rocky
sudo dnf install gitlab-ee-18.5.3-ee.0.el9
dnf --showduplicates list gitlab-ee | tail -20
```

```bash
# Docker：改镜像 tag，配置/数据卷不动；先备份再动
docker pull gitlab/gitlab-ee:18.5.3-ee.0
docker compose up -d gitlab        # 或 docker stop/rm + 用同一套 -v 重新 run
docker exec -it gitlab gitlab-rake gitlab:check

# Helm chart：values.yaml 里指定镜像 tag，并对照 chart↔GitLab 版本映射表
helm upgrade -i gitlab gitlab/gitlab --set gitlab.version=18.5.3 ...
```

单节点 Omnibus 的官方编排（把 4.1–4.6 串起来）：

```text
开启维护模式（可选但推荐） → 暂停 CI/CD → 升 Runner 到目标版本 → 升 GitLab 包到"下一个停靠站"
→ 等后台迁移跑完 → （如需）再走下一站 → 恢复 CI/CD → 关维护模式 → 跑升级健康检查
```

## 4.7 后台迁移：停靠站之间必须等的那件事

Batched background migrations 由 **Sidekiq 的 `background_migration` 队列**执行，实例可在其运行期间继续服务，但**大实例上性能会下降**。官方态度：**每次升级之前所有迁移必须跑完**，并且"要把各 major/minor 升级之间的时间拉开，留出迁移完成的时间"。要缩短耗时，就**给 `background_migration` 队列加 Sidekiq worker**。

```bash
# 18.5 起有成套命令行任务（ID 统一格式：{database}_{id}，如 main_85 / ci_10）
sudo gitlab-rake gitlab:background_migrations:list
sudo gitlab-rake gitlab:background_migrations:show[ci_1]
sudo gitlab-rake gitlab:background_migrations:pause[main_85]   # 只能暂停 active 状态的
sudo gitlab-rake gitlab:background_migrations:resume[main_85]
sudo gitlab-rake gitlab:background_migrations:execute[ci_10]
sudo gitlab-rake gitlab:background_migrations:execute_all
sudo gitlab-rake gitlab:background_migrations:status        # 汇总视图（与 list 并列的老任务名）
# finalize 的参数**不是** migration_id，而是"任务类名/表/列/参数"，用于手工收尾卡住的迁移
sudo gitlab-rake "gitlab:background_migrations:finalize[<job_class_name>,<table_name>,<column_name>,'<job_arguments>']"

# 19.5 起：给自动化用的闸门，只轮询不代跑；全部完成退出 0，有 failed 就非 0
sudo gitlab-rake gitlab:background_migrations:wait
sudo BATCHED_MIGRATIONS_WAIT_INTERVAL_SECONDS=60 gitlab-rake gitlab:background_migrations:wait
```

`list` 的输出形态（`progress` 里带 ETA，是判断"能不能今晚走下一站"的依据）：

```plaintext
id      | table_name                       | job_class_name                                | status    | progress
--------|----------------------------------|-----------------------------------------------|-----------|---------------------------------------------------------
main_1  | namespace_settings               | UpdateRequireDpopForManageApiEndpointsToFalse | finished  | 100.00%
main_5  | security_policies                | BackfillPipelineExecutionPoliciesMetadata     | active    | 42.50% (estimated time remaining: 5 minutes)
ci_1    | ci_runners                       | MarkAdminBotRunnersAsHosted                 | finalized | 100.00%
```

> `wait` 任务**没有内部超时**，官方要求调用方自己加 `timeout` 或 job deadline；瞬时数据库错误它不重试，整任务重跑即可。**failed 状态必须先用官方"Resolve failed batched background migrations"流程处理掉**，否则 `wait` 永远返回非 0，你的流水线会卡在"升级闸门"上 —— 这是把升级做成自动化后最常见的假故障。

## 4.8 回滚

官方把"回滚计划"定义为两件事的组合：**备份流程 + 恢复流程**，并要求在需要之前就测过。

- 数据层回滚 = 用 3.2 的流程恢复到**升级前版本**（版本必须与备份一致，所以升级前那次备份的版本号要写进变更单）。
- 包层回滚有专门文档：`update/package/downgrade.md`（**降级是受限操作**，不是所有版本都能降）。
- **快照路线**：多节点要**每个节点都拍**，且官方明确"该流程超出 GitLab 支持范围"，也就是说出问题时 Support 不会替你把快照拼回来。

> **写回滚决策的时点**：升级前，不是升级失败后。要事先定好"什么现象就回滚"（例如：迁移报错且 30 分钟内查不到解法、后台迁移 `failed` 且无法 resolve、登录/clone 冒烟不过）。缺这条约定，事故现场会花两小时尝试"再修一下"。

---

# 5. 日常运维工具箱

## 5.1 服务与配置入口

```bash
sudo gitlab-ctl status                 # 各 runit 服务状态
sudo gitlab-ctl reconfigure            # 改完 gitlab.rb 必执行
sudo gitlab-ctl restart puma sidekiq
sudo gitlab-ctl tail                   # 聚合看日志；也可 tail 单个服务：gitlab-ctl tail gitaly
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rails runner -e production 'puts Gitlab::VERSION'
```

## 5.2 四种探活端点，别用错

**前提**：这些监控端点受 **IP allowlist** 管控，默认从 localhost 可访问。

| 端点 | 检查什么 | 正确用途 | 别这么用 |
|------|----------|----------|----------|
| `GET /-/health` | 只看应用进程活着（绕开 Rails controller，最早期的中间件） | 存活探针的极简选项 | 不代表 DB/Gitaly 好 |
| `GET /-/liveness` | 应用服务器是否运行、controller 有没有线程死锁 | K8s liveness probe | 不代表能接流量 |
| `GET /-/readiness`（可 `?all=1`） | 默认只查实例自身；`all=1` 时逐个校验 **DB / Redis / Gitaly** 等依赖并给出各项状态 | LB 摘除 / 滚动发布 readiness | 失败返回 **503**；`readiness` 豁免 Rack Attack 限流 |
| `GET /health_check`（`/health_check/database`、`/cache`、`/migrations`、`/geo`） | 完整栈体检（health_check gem，会验 DB、Redis 等后端） | 监控大盘、排障、告警 | **官方明确警告：不要用于负载均衡或自动伸缩** —— 后端慢就会失败，会把健康节点误摘 |

```bash
curl -s http://localhost/-/health                  # GitLab OK
curl -s http://localhost/-/liveness                # {"status":"ok"}
curl -s 'http://localhost/-/readiness?all=1' | jq  # 各依赖项状态
curl -s http://localhost/health_check/migrations   # 迁移是否 pending
```

## 5.3 维护模式

```bash
# API（管理员令牌）
glab api -X PUT "application/settings?maintenance_mode=true"
glab api "application/settings" | jq '{maintenance_mode, maintenance_mode_message}'
glab api -X PUT "application/settings?maintenance_mode=false"
```

UI：**Admin → Settings → General → Maintenance Mode**，可开关并**自定义横幅文案**（`maintenance_mode_message` 属 **Premium/Ultimate**）。

模式下的行为边界（官方逐条列出）：

- **Git 只读操作照常**：`git clone`、`git pull` 可用；
- **所有写操作失败**（CLI 与 Web IDE 都一样），报错原文：`Git push is not allowed because this GitLab instance is currently in (read-only) maintenance mode.`
- 启用了 Geo 时，**primary 与 secondary 的推送都会失败**；
- 其余面（Issue/MR、REST/GraphQL、CI、部署、容器与包仓库、后台 job、事件管理、Feature flags、安全特性）在官方文档里有逐项说明，**改流程前照那一页核对**，不要按"维护模式=全都禁用"直觉处理。

## 5.4 四档"只读"：粒度、影响面与恢复

| 档位 | 手法 | 影响 |
|------|------|------|
| **仓库只读**（UI 仍可用） | Rails 控制台批量置 `repository_read_only` | 只禁 Git 写；Web 侧仍可看 Issue/MR |
| **切断内部 API** | `sudo gitlab-ctl stop puma` | 内部 API 不可达，效果近似"写全停"，但 sidekiq 仍在排队 |
| **整个 UI 停摆** | `gitlab-ctl stop sidekiq` + `stop puma` | 官方给出的"最省事确保无人能改"的做法 |
| **数据库只读** | 建 `gitlab_read_only` 角色，只 `GRANT SELECT` | 面向"允许人读界面但要 DB 层不可写"的极端场景 |

```ruby
# sudo gitlab-rails console
Project.all.find_each { |project| project.update!(repository_read_only: true) }

# 只对部分项目
[1,2,3].each do |p|
  begin
    Project.find(p).update!(repository_read_only: true)
  rescue ActiveRecord::RecordNotFound
    puts "Project ID #{p} not found"
  end
end

# 解除
Project.all.find_each { |project| project.update!(repository_read_only: false) }
```

```sql
-- 数据库只读用户（官方示例骨架；先做备份再动）
CREATE USER gitlab_read_only WITH password '...';
GRANT CONNECT ON DATABASE gitlabhq_production TO gitlab_read_only;
GRANT USAGE ON SCHEMA public TO gitlab_read_only;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO gitlab_read_only;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO gitlab_read_only;
ALTER DEFAULT PRIVILEGES FOR USER gitlab IN SCHEMA public GRANT SELECT ON TABLES TO gitlab_read_only;
```

> **四档都要写"解除命令"在同一个变更单里**。事故现场最容易发生的是"只读打开了、忘了它是谁打开的、怎么关"，于是业务侧看到"提交不上去"两小时。

## 5.5 限额与实例配置自检

两个官方自检入口：

```bash
# 1) 网页：对所有人可见（未登录只看到与自己相关的部分）
#    Help(?) → Check the current instance configuration，或直接访问：
curl -s https://gitlab.example.com/help/instance_configuration | grep -o 'SSH host keys\|Rate limits\|Size limits' | head

# 2) 控制台：把全部 Plan 限额原样打出来
sudo gitlab-rails console
Plan.default.actual_limits
```

自建实例默认值里，运维上最常撞到的几项（**数值以你实例回读为准**，管理员可用 Plan Limits API 改）：

| 限额键 | 默认 | 含义 |
|--------|------|------|
| `project_hooks` / `group_hooks` | `100` / `50` | 每项目 / 每群组 webhook 条数 |
| `web_hook_calls`（`Plan.default.actual_limits.update!(web_hook_calls: 10)`） | `0`＝**不限速** | 顶层命名空间每分钟 webhook 调用数；project 与 group webhook **共享**同一份额度；超限调用记在 `auth.log` |
| `offset_pagination_limit` | `50000` | REST 分页 offset 上限，超过就要求改 keyset 分页 |
| `repository_size` / `storage_size_limit` | `0`＝不限 | 仓库/存储限额 |
| `maven_max_file_size` | `3221225472` | **字节**，约 3 GiB（`*_max_file_size` 全是字节：npm/nuget ≈500 MiB、golang 100 MiB、helm 5 MiB、generic/cargo 5 GiB） |
| `pages_file_entries` | `200000` | 单个 Pages 站点文件数 |
| `pull_mirror_interval_seconds` | `300` | 拉取镜像最小间隔（第 4 篇） |
| `ci_registered_project_runners` / `ci_registered_group_runners` | `1000` | 可注册 Runner 数 |
| `external_audit_event_destinations` | `5` | 外部审计事件投递目标数（第 12 篇 9.1） |

## 5.6 Rails 控制台的使用纪律

控制台能做很多 API 做不到的事（改 secrets、恢复误撤令牌、批量设置只读），但它**不是审计友好的入口**：

- **控制台里的改动默认不产生审计事件**。第 12 篇 4 节讲过"只有走撤销服务才生成审计事件"这一类的例子；同理，直接 `update!` 改对象不会在 audit events 里留下"谁在什么时候改了什么"。所以：**能走 API 的走 API**，非要走控制台时，把会话命令与输出贴进工单，并在事件单里注明"本次变更无审计事件"。
- **只读优先**：`rails console` 默认是**生产库可写**会话，进去第一件事是确认自己是不是在查而不是在改。
- **批量操作分段跑**（`find_each`、限量、先 `count` 再动），否则一次全表 `update!` 会把 Sidekiq 与 DB 拖住。
- 用完把会话历史留存（`sudo -u gitlab-psql` 之类跨用户操作同理）。

---

# 6. 实例级设置：把门禁下发下去

## 6.1 先读回现值再改

```bash
# 全量快照，贴进变更单
glab api /settings | jq 'to_entries | map({k:.key, v:.value}) | sort_by(.k)' > settings-before.json

# 只看你关心的几组
glab api /settings | jq 'with_entries(select(.key|test("two_factor|admin_mode|maintenance_mode|visibility|git_access|local_requests|adjourned|deletion_protection";"i")))'

# 改（管理员令牌；改完再读一次对比）
glab api -X PUT /settings -f require_two_factor_authentication=true -f two_factor_grace_period=0
glab api /settings | jq '.require_two_factor_authentication, .two_factor_grace_period'
```

## 6.2 关键设置键与官方语义

| 键 | 类型/单位 | 官方语义（照抄要点） |
|----|-----------|----------------------|
| `signup_enabled` | boolean | 是否允许自注册。**内网实例应关** |
| `require_two_factor_authentication` | boolean | 强制 2FA；配套 `require_admin_two_factor_authentication` 单独覆盖管理员角色 |
| `two_factor_grace_period` | **小时**（`required by: require_two_factor_authentication`） | 允许用户跳过配置 2FA 的时长，**填 `0` = 下次登录就必须登记** |
| `admin_mode` | boolean | 管理员执行管理动作前需重新认证（第 12 篇 8.3） |
| `maintenance_mode` / `maintenance_mode_message` | boolean / string | 维护模式；横幅文案属 **Premium/Ultimate** |
| `default_project_visibility` | string | 新项目默认可见性，**默认 `private`**；**不能设成 `restricted_visibility_levels` 里的级别** |
| `restricted_visibility_levels` | array | 非管理员不能使用的级别（`private`/`internal`/`public`）；默认 `null`＝无限制；**不能把默认级别自己也禁掉** |
| `default_branch_protection` | integer | **17.0 起 Deprecated**，改用 `default_branch_protection_defaults` |
| `default_branch_protection_defaults` | hash（17.0 引入） | 键：`allowed_to_push`（支持 Developer 30 / Maintainer 40）、`allow_force_push`、`allowed_to_merge`（同样只支持 30/40）、`developer_can_initial_push`、`code_owner_approval_required` |
| `enabled_git_access_protocol` | string | `ssh` / `http` / `all`。选 `ssh` 后**Job Token 仍可 HTTPS clone**（第 12 篇 3.4） |
| `allow_local_requests_from_web_hooks_and_services` | boolean | **Webhook 与集成能否访问内网**（第 8 节） |
| `allow_local_requests_from_system_hooks` | boolean | System hooks 单独一个开关，且**只作用于默认组织里的 system hooks**；其它组织跟着上一个开关 |
| `asset_proxy_enabled` / `asset_proxy_url` | boolean / string | 开启**需要重启 GitLab**（官方原话 "GitLab restart is required to apply changes"），`asset_proxy_url` 为必填伴随项 |
| `deletion_adjourned_period` | integer，**1–90，默认 30** | 标记删除的项目/群组**等待删除的天数**（不是"设为 0 立即删"） |
| `default_project_deletion_protection` | boolean | 新项目默认开启删除保护 |
| `repository_size_limit` | integer，**MB**，**Premium/Ultimate only** | 每仓库大小上限；超限报 `This repository has exceeded its size limit`；单个项目可在 Admin → Projects 覆盖 |
| `max_attachment_size` | integer，**MB** | 附件上限；上传超限常见 `413 Request Entity Too Large` |
| `push_event_hooks_limit` | integer，**默认 3** | **单次 push 中分支/标签变更数超过该值就不再触发 webhook 与集成**；**设 `0` 并不能关闭限流** |
| `require_personal_access_token_expiry` | boolean | 建 PAT 与群组/项目令牌时**必须填过期日**（服务账号拥有的 PAT 除外） |
| `personal_access_token_prefix` | string（例 `glpat-`） | **PAT 前缀是唯一可自定义的令牌前缀**，其余令牌前缀不可配（第 12 篇第 4 节） |
| `housekeeping_enabled` | boolean | Git housekeeping 开关，**需配套字段** |
| `repository_storages_weighted` | hash | 新仓库落到哪些 storage（按权重随机），对应 `repository_storage_paths` |

> 与"只管新建"同类的坑（第 12 篇推送规则那条规律在这里也成立）：`default_project_visibility`、`default_branch_protection_defaults`、`default_project_deletion_protection`、`repository_storages_weighted` 都是**新项目/新仓库的默认值**，改它们**不会回刷已存在的项目**。统一收紧要写脚本遍历下发（模板见第 15 篇 4.1）。

## 6.3 全局推送规则与"只管新建项目"

第 12 篇 6.2 已核实三层规则的继承语义（全局规则只被**其后新建项目**继承一份副本、改全局不回刷、子群组不继承群组规则、项目规则替换不合并）。管理员侧只补两条操作要点：

```bash
# 1) 写：注意是 POST /projects/:id/push_rule（项目层）；全局层走 Admin 或 application settings 对应端点
glab api -X POST "projects/$P/push_rule" \
  -f "commit_message_regex=^[A-Z]+-[0-9]+" \
  -f prevent_secrets=true \
  -f max_file_size=100          # 单位 MB，0＝不限；LFS 豁免

# 2) 读回验证：从未配过规则的项目，这个端点返回 200 + 字面量 null，不是 404
glab api "projects/$P/push_rule" | jq 'if . == null then "NOT CONFIGURED" else . end'
```

## 6.4 出网与内网访问控制

| 需求 | 做法 |
|------|------|
| 允许 Webhook 打到内网服务 | 打开 `allow_local_requests_from_web_hooks_and_services`（**默认允许会扩大 SSRF 面**，第 12 篇 8.3 建议显式决定而不是留默认） |
| 只放行 System hooks | `allow_local_requests_from_system_hooks`（只管默认组织） |
| 群组 IP 白名单之外再加实例级允许段 | Admin → Settings → Visibility and access controls → **Globally-allowed IP address ranges**（与群组限制组合使用，第 12 篇 3.1） |
| 外部图片代理（CSP/Asset Proxy） | `asset_proxy_enabled` + `asset_proxy_url` + allowlist；**改完要重启** |
| 出站 HTTP 响应限制 | 出站请求有 gzip/大小/JSON 对象数与嵌套深度等限制（`instance_limits.md` 的 HTTP response limits 一节），对接外部系统返回超大 JSON 时要按这些上限设计分页 |

---

# 7. 群组治理：Owner 的日常

## 7.1 结构先行，重构很贵

群组层级决定权限继承、策略继承、IP/域名限制、fork 限制、模板可见范围。事后重构 = **全量 URL/CI 引用迁移**：

```bash
# 迁移前盘点：新旧路径、重定向是否覆盖、CI include 引用
glab api "groups/$G/projects?include_subgroups=true&per_page=100" | jq -r '.[].path_with_id' > before.txt
glab api "groups/$G" | jq '{id, full_path, parent_id, shared_runners_enabled, project_creation_level, request_access_enabled, share_with_group_lock}'
# 用 API 找引用了旧路径的流水线（include: project: 会按路径解析）
for p in $(glab api "groups/$G/projects?include_subgroups=true&per_page=100" | jq -r '.[].id'); do
  glab api "projects/$p" | jq -r '[.id,.path_with_namespace,.http_url_to_repo] | @tsv'
done | column -t
```

群组层可控的字段（Groups API 可回读/下发）：`project_creation_level`（`administrator` / `noone` / `maintainer` / `developer`）、`request_access_enabled`、`share_with_group_lock`、`lfs_enabled`、`enabled_git_access_protocol`、`require_two_factor_authentication`（群组级强制 2FA）、`default_branch_protection_defaults`。

## 7.2 成员、继承与共享

| 能力 | 入口 | 管理员要知道的边界 |
|------|------|--------------------|
| 成员批量授权 | Group → Manage → Members（按来源显示 direct / inherited） | **Group Owner 只能更新 direct membership**；改 inherited 会 403（第 12 篇 3.3） |
| 访问申请 | Group → Manage → Access requests | `request_access_enabled=false` 可关掉入口；`Prevent users from requesting access to a group` 收紧更彻底 |
| 项目共享 | Project/Group → Settings → Permissions | `share_with_group_lock` 阻止本群组内项目被共享给组外；与"禁止邀请组外群组"配合控席位 |
| 群组删除 | **Admin / Groups → Schedule for deletion** | **18.0 起该端点从 Premium 移到 Free**；GitLab.com 保留 30 天，自建保留期跟实例设置（`deletion_adjourned_period`）走 |
| 禁止组外 fork / 锁项目加人 / 域名与 IP 限制 | Settings → General → Permissions and group features | **只能设在顶层群组**（第 12 篇 3.1/3.2/3.4） |

## 7.3 模板与策略下发

群组是"部门级标准"的载体，四种下发通道：

```yaml
# ① 群组级 CI 变量：统一分析器前缀、内网镜像仓库、扫描开关
#    Group → Settings → CI/CD → Variables（选 "Save variable" 后在项目流水线可直接引用）
variables:
  SECURE_ANALYZERS_PREFIX: "harbor.internal.acme.com/analyzers"
  MAVEN_CLI_OPTS: "--no-transfer-progress --settings $CI_PROJECT_DIR/.m2/settings.xml"
```

```yaml
# ② 模板仓库 + include：全部门统一流水线骨架（第 7 篇 include 机制的企业形态）
include:
  - project: 'platform/gitlab-ci-templates'
    ref: 'v2.4.1'          # 固定 tag，别用 main：主干变更会瞬间波及所有项目
    file: '/templates/java-service.yml'
```

```bash
# ③ 群组级保护/审批与推送规则：脚本遍历下发（模板见第 15 篇 4.1）
glab api "groups/$G/projects?include_subgroups=true&per_page=100" | jq -r '.[].id' \
  | xargs -I{} sh -c 'glab api -X POST "projects/{}/protected_branches" -f name=main -f push_access_level=40 -f merge_access_level=40'

# ④ 安全策略/合规框架（Ultimate）：Group → Secure → Security policies / Compliance frameworks
glab api "groups/$G/audit_events?per_page=5&created_after=2026-11-01T00:00:00Z" | jq -r '.[].type'
```

> **版本与订阅差异**要显式写进部门规范：哪些能力需要 Premium/Ultimate（审批规则、群组 IP 限制、域名限制、合规框架、漏洞管理、审计事件层级…）在第 12 篇各节都标了；`1_introduction.md` 的分层表是系列口径。

## 7.4 配额、容量与清理

```bash
# 1) 群组总占用：Groups 详情 + statistics=true（**仅管理员**；顶层群组还会多返回一份 root_storage_statistics，17.4 起）
glab api "groups/$G?statistics=true" | jq '.statistics, .root_storage_statistics'

# 2) 按项目拆解（statistics=true 才有 statistics 段；字段名是 job_artifacts_size，不是旧的 build_artifacts_size）
glab api "groups/$G/projects?include_subgroups=true&statistics=true&per_page=100" \
  | jq -r '.[] | [.path_with_namespace,
        (.statistics.storage_size/1048576|floor|tostring),
        (.statistics.repository_size/1048576|floor|tostring),
        (.statistics.lfs_objects_size/1048576|floor|tostring),
        (.statistics.job_artifacts_size/1048576|floor|tostring),
        (.statistics.packages_size/1048576|floor|tostring),
        (.statistics.container_registry_size/1048576|floor|tostring)] | @tsv' \
  | column -t        # 单位 MiB；container_registry_size 需要容器仓库元数据库开启

# 限额来源：实例级 repository_size_limit（MB，Premium/Ultimate）+ Admin → Projects 单项目覆盖
glab api /settings | jq '{repository_size_limit, max_attachment_size}'
```

清理侧的既有事实（第 10 篇）：制品/镜像/包的清理策略按**保留规则**跑，**包清理策略每 12 小时执行一次、首次配置后要等约 12 小时才见效**；容器仓库受保护 tag、immutable tags 属 Ultimate。**别把"没清掉"当成 bug**，先看策略周期与规则匹配。

## 7.5 休眠项目删除（治理"僵尸仓库把磁盘塞满"）

入口：**Admin → Settings → Repository → Repository maintenance → Delete dormant projects**（管理员权限）。三个字段构成语义：

| 字段 | 含义 |
|------|------|
| **Delete dormant projects that exceed**（MB） | 存储**超过**该大小的项目才纳入判定；填 `0` 表示**全部项目**纳入 |
| **Delete project after**（月） | 休眠达到该时长即**计划删除** |
| **Send warning email**（月） | 提前发警告邮件给该项目的 **Owner 与 Maintainer**；**必须小于** "Delete project after" |

两个官方例子直接解释了"为什么不能把 warn 设得比 delete 大"：

- 设 `exceed=50`、`after=12`、`warn=6`：<50 MB 的项目不算休眠；>50 MB 且休眠>6 个月发邮件，>12 个月计划删除。
- 设 `exceed=0`、`after=12`、`warn=11`：全实例纳入；若某项目**在开启前就已休眠超过 12 个月**，会**立即**发警告，并在 **12−11=1 个月**后计划删除。

> **两条红线**：① 休眠项目**即使已归档（archived）也会被删**；② 这个开关的作用域是"实例"，不是"某个群组"，一旦开启就是全实例扫描。开启前请先用群组导出 + 活动盘点（第 11 篇的 activity 页 + `last_activity_at`）确认名单，并给业务方留足警告期。

---

# 8. 事件出口：Webhook / System hooks / Integrations

## 8.1 三者选型

| 机制 | 触发方 | 粒度 | 可靠性 | 典型场景 |
|------|--------|------|------|----------|
| **Project / Group Webhook** | GitLab 主动 HTTP POST | 项目或群组（群组 webhook 覆盖组内项目） | 有重试、可查看投递记录、会**自动禁用** | MR 合并触发外部发布、push 触发文档重建 |
| **System Hooks**（实例/组织管理员） | 全实例的**元数据事件** | 实例级 | 与 webhook 同一套 push 限制；**实例管理员管的 hook 不会被自动禁用** | 接安全审计：谁删了项目、谁加了成员、谁建了 key |
| **Integrations**（内置模板） | 预制逻辑 | 项目 | — | Jira / Slack / Teams / PagerDuty 联动 |

> **设计结论不变**：Webhook 是"通知"语义（fire-and-forget + 验签 + 重试），不是"编排"语义。**需要强事务的联动（合并即发布且必须成功）要做成 CI job，而不是 webhook 消费端**。入方向则相反：外部系统用 REST API（+ OAuth/项目令牌）在 GitLab 里建 Issue、评论 MR、触发流水线（`POST /projects/:id/trigger/pipeline`），把 GitLab 当协作后端。

## 8.2 请求头与验签（19 起有 HMAC）

官方投递头（现版本清单）：

| Header | 说明 |
|--------|------|
| `X-Gitlab-Event` | 事件类型名，形如 `Push Hook` |
| `X-Gitlab-Event-UUID` | 非递归 webhook 的唯一 ID；由 webhook 触发的**递归调用共享同一值** |
| `X-Gitlab-Webhook-UUID` | 每条 webhook 配置的唯一 ID |
| `X-Gitlab-Instance` | 发送方实例主机名 |
| `X-Gitlab-Token` | 配了 **secret token** 时**明文**带过去 |
| `User-Agent` | `GitLab/<VERSION>`（例 `GitLab/15.5.0-pre`） |
| `Idempotency-Key` / `webhook-id` | 跨重试保持一致的消息 ID（两者相等）；新接入**优先用 `webhook-id`** |
| `webhook-timestamp` | 请求生成的 Unix 秒级时间戳 |
| `webhook-signature` | **仅在配了 signing token 时出现**，空格分隔的多个 `v1,{base64}` 签名 |

**两层验证要分清**：

- **Secret token**：静态串，明文放在 `X-Gitlab-Token`，接收端比对。只能证明"对方知道这个串"。
- **Signing token（19.0 引入并默认开 flag，19.1 GA 并移除 flag）**：按 **Standard Webhooks** 规范对 `{message_id}.{timestamp}.{body}` 计算 **HMAC-SHA256**，密钥是 `whsec_` 前缀去掉后 base64 解码的字节串。能同时证明**来源与完整性**。签名 token **永远不会被 API 读回**，只配置一次。

```python
# 接收端校验（官方 Python 示例的思路）
import base64, hashlib, hmac

def valid(signing_token, message_id, timestamp, body, received):
    key = base64.b64decode(signing_token.removeprefix('whsec_'))
    msg = f"{message_id}.{timestamp}.{body}".encode()
    expected = "v1," + base64.b64encode(hmac.new(key, msg, hashlib.sha256).digest()).decode()
    return any(hmac.compare_digest(expected, s) for s in received.split(' '))
```

> **迁移与防重放**：两种 token **可以并存**，官方给的零停机迁移就是"先都配、接收端见到 `webhook-signature` 就验签、否则回落到 secret token，最后删 secret token"。**务必校验 `webhook-timestamp` 足够新**以抵御重放。

## 8.3 失败、自动禁用与恢复

| 状态 | 触发条件 | 恢复方式 |
|------|----------|----------|
| **Temporarily disabled** | **连续 4 次**失败 | 先禁 **1 分钟**，之后每次失败延长，**最长 24 小时**；到期自动重新启用 |
| **Disabled**（永久） | **连续 40 次**失败（17.11 起；之前文案叫 "Fails to connect"，17.11 改名并改为 40 次阈值） | **不会自动恢复**；**发一次测试请求**，返回 2xx 即重新启用 |

失败的定义：**接收端返回 4xx/5xx**、连接超时、其它 HTTP 错误。自建实例上这套自动禁用由 feature flag `auto_disabling_web_hooks` 控制（官方在 system hooks 一节明写"disabled by default"，项目 webhook 历史里也标注该 flag 于 15.10 在 Self-Managed 关闭），**别照抄 GitLab.com 的行为写运维手册**。

System hooks 的差异（重要）：**组织管理员管的 system hooks 会被自动禁用；实例管理员管的 system hooks 不会被自动禁用、也不受 webhook 速率限制**；重新启用方式是该 hook 的 **Test** 下拉里选一个事件类型发一次。

## 8.4 限额与 push 事件

- 条数：`project_hooks = 100`、`group_hooks = 50`（自建可改，走 Plan Limits API）。
- 速率：顶层命名空间每分钟 webhook 调用数 `web_hook_calls`（project 与 group webhook **共享**），实例管理员管的 system hooks **不限速**；超限调用**记入 `auth.log`**。

  ```ruby
  # sudo gitlab-rails console
  Plan.default.actual_limits.update!(web_hook_calls: 10)   # 0 = 关闭限速
  ```

- **push 事件的硬规则**：单次 push 含**超过 3 个分支或标签变更时，整个 push 事件不触发任何 webhook 与集成**（项目 webhook 与 system hooks 都受此限）。由 `push_event_hooks_limit` 控制，默认 3，且 **设 `0` 不代表不限流**（官方原话 "Setting to `0` does not disable throttling"）。给"批量打 tag 触发下游"的流水线做设计时，要么改由 CI/schedule 触发，要么管理员显式提高该值并记录到变更单。
- GitLab.com 另有实例级 webhook 限制（条数/每分钟/超时），自建实例由管理员改。

## 8.5 System hooks：实例级事件流（审计兜底）

官方事件清单里最常被接进 SIEM 的项：

| 事件 | 触发 |
|------|------|
| `project_create` / `project_destroy` / `project_rename` / `project_transfer` / `project_update` | 项目的建、删、改名/路径、转移、属性变更（路径除外） |
| `group_create` / `group_destroy` / `group_rename` | 群组建、删、路径或名称变化 |
| `user_create` / `user_destroy` / `user_rename` | 账号建、删、改名 |
| `user_add_to_team` / `user_remove_from_team` / `user_update_for_team` | 项目成员增、删、**角色变更** |
| `user_add_to_group` / `user_remove_from_group` / `user_update_for_group` | 群组成员增、删、角色变更 |
| `key_create` / `key_destroy` | SSH key 增删 |
| `user_failed_login` | **被封禁用户尝试登录** |
| `user_access_request_to_group` / `_to_project`、`..._revoked_for_...` | 访问申请与撤回 |
| `repository_update` | **push 里含 tag 或多个分支**（与 8.4 的 push 限制同源） |

**System hooks 与 audit events 的分工**（第 12 篇 9.1 的延伸）：

- `audit events` 回答"**谁**在**什么时候**对**什么对象**做了什么"，是权限与门禁变更的唯一可信来源，但要付费层 + 导出，且查询窗口 ≤30 天。
- `system hooks` 是**推送式**事件流，适合做"实时告警/自动化反应"（谁删了项目立刻通知），不含完整人因上下文。
- 两者**都要**，不要用一个替代另一个。

## 8.6 排障入口

```bash
# 1) 先看有哪些 hook、告警状态与是否被禁用（字段名以 GET /projects/:id/hooks/:hook_id 的回读为准）
glab api "projects/$P/hooks" \
  | jq -r '.[] | [(.id|tostring), .url, (.alert_status // "-"), (.disabled_until // "active"), (.token_present|tostring), (.signing_token_present|tostring)] | @tsv'

# 2) 投递历史：**只保留最近 7 天**，可按状态类别过滤（successful / client_failure / server_failure）
glab api "projects/$P/hooks/$HOOK_ID/events?status=client_failure&per_page=20" \
  | jq -r '.[] | [(.id|tostring), .trigger, .response_status, (.execution_duration|tostring), .url] | @tsv'

# 3) 单条重发 / 用测试请求重新启用被禁用的 hook（test 返回 2xx 即恢复）
glab api -X POST "projects/$P/hooks/$HOOK_ID/events/$EVENT_ID/resend"
glab api -X POST "projects/$P/hooks/$HOOK_ID/test/push"
# 群组侧同构（/groups/:id/hooks/...）；敏感段掩码与自定义头也各有端点
glab api -X PUT  "projects/$P/hooks/$HOOK_ID/url_variables/$KEY" -f variable='hidden'
glab api -X PUT  "projects/$P/hooks/$HOOK_ID/custom_headers/$HEADER" -f value='...'

# 4) 被限速或被内网策略挡：限速丢弃记在 auth.log，请求侧看 rails/nginx
sudo grep -hE 'web_hook|WebHook|rate limit' /var/log/gitlab/gitlab-rails/auth.log* | tail
sudo grep -h 'hooks' /var/log/gitlab/gitlab-rails/production_json.log | tail | jq -c '{time, method, path, status}'

# 5) 目标端拒收的常见三因：TLS 校验失败、连接超时、body 过大
curl -sS -o /dev/null -w '%{http_code} verify=%{ssl_verify_result} t=%{time_total}\n' https://receiver.internal/acme/hook
```

要点：**Enable SSL verification 默认开**，自签证书要么把 CA 放进实例信任链，要么针对单条 hook 关闭校验（并在事件单里记这个风险）；**URL 支持敏感段掩码**（`url_variables` 端点 / UI 的 Add URL masking）、**自定义头**（`custom_headers`）与 **mTLS** 都已支持；**"允许本地网络请求"的两个开关（6.4）决定 webhook 能不能打到内网接收端**，这是"webhook 一直失败但外部接收端正常"的第一嫌疑。投递历史只留 7 天，所以**要长期留痕就得自己落库**（消费端入库，或用 system hooks 转 SIEM）。

---

# 9. 席位与用户：把钱算清楚

## 9.1 谁算 billable（官方口径）

**计费用户**（占席位）：

- 在订阅覆盖的命名空间/顶层群组里有访问权，且角色为 **Guest、Planner、Reporter、Security Manager、Developer、Maintainer、Owner 或自定义角色**（只有 `read_code` 权限的自定义 Guest **除外**）；
  - 其中 **Guest 在 Premium 上计费，在 Free 与 Ultimate 上不计费**；
- **Auditor 用户**；
- 自建 Premium/Ultimate 上的**管理员**；
- 自建 Premium 上**没有命名空间访问权**的用户。
- 同属一个顶层群组（订阅所在层级）多个群组/项目的人**只算一次**。

**不计费**：

- 待批准（pending approval）用户；
- **deactivated / banned / blocked** 用户；
- 不属于任何项目或群组的用户（**Ultimate**）；
- 仅有 Guest 角色（**Ultimate**）；
- 仅有 **Minimal Access** 角色；
- GitLab 自建的服务账号：Ghost User、Support Bot、**项目/群组令牌对应的 bot 用户**及其它 internal users。

> 这条清单解释了第 12 篇两处建议的成本含义：**"给审计岗 Auditor"是对的（不给写权限），但它照样占席位**；**"把人从 Developer 降到 Minimal Access"能腾席位**，但 Minimal Access 连项目内容都看不到（不是"只读"，只读要用 Guest/Reporter）。所以"席位优化"方案要先跟权限方案对齐，否则财务省了、业务瘫了。

## 9.2 超额怎么算

公式：**超额人数 = 计费周期内的最大计费用户数 − 已购席位数**。注意是**峰值**，不是当前值：

| 事件 | 计费用户 | 期间最大值 |
|------|----------|------------|
| 10 人占满 10 席 | 10 | 10 |
| 新增 2 人 | 12 | 12 |
| 3 人离职并被 **block** | 9 | **12** |
| 新增 4 人 | 13 | 13 |

→ 结算按 **13 − 10 = 3** 人超额。席位使用每**季度或每年**复核一次（quarterly reconciliation）。试用（trial）自建许可证的超额值恒为 0。

官方给的防超额三招：禁止邀请组外群组、开启 restricted access、要求新账号管理员审批（并在接近上限时提前买席）。

## 9.3 两种限流：user cap 与 restricted access（互斥）

| 维度 | **User cap** | **Restricted access** |
|------|--------------|------------------------|
| 语义 | 达到上限后，加人/注册需 Owner 或管理员**批准** | 无席位时**直接阻止**新增计费用户 |
| 可用层级 | 顶层群组、实例 | 顶层群组、实例 |
| 历史 | GitLab.com 16.3 启用，17.1 GA | 17.5 引入，18.0 GA；18.7 群组共享行为调整；自建自动开启 19.1 带 flag、**19.6 GA** |
| 生效延迟 | 计费用户数**每天更新一次**，因此可能"已经超过才生效"；把 cap 设得低于当前人数会**立即生效** | 已超限时开启**不改变现有成员**（不降角色、不 block、不移除），只挡新增 |
| 与共享 | 组内存在共享到命名空间外时**无法启用 user cap**；开启期间自动禁止"邀请组外群组"且关不掉 | 与外部群组共享**不兼容**；.com 开启时自动打开"禁止邀请组外群组" |
| 互斥 | **两者不能同时使用**：开启 restricted access 会禁用 user cap | |
| 自建特例 | LDAP/OmniAuth 实例在开关管理员审批时**可能因 Rails 配置变更而短暂停机** | 订阅不允许超额时**自动开启且关不掉** |
| 无席位时的兜底 | 需人工批准 | 经 **SAML / SCIM / LDAP 开通**的用户落到 **Minimal Access**（18.6 带 flag → 18.10 默认开 → **19.5 GA**），有席后手动升上去；已有计费角色不受影响 |

**几个必须写进流程的细节**：

- **待接受的邀请**：无席位时，`.com` 上授予计费角色的邀请**保持 pending**直到腾出席位；自建 **Ultimate** 行为相同；自建 **Premium** 则在**建账号时**就按席位拦截，用户会被告知"账号未创建，请联系管理员"。
- **休眠用户重新登录**：restricted access 生效且无席位时，休眠用户（含 enterprise users）**变成待批准**而不是直接恢复，既有成员关系保留；**非** enterprise 的休眠成员被重新加入时是**移除群组关系**而非置为 deactivated；只持有 Minimal Access 的用户**可直接恢复**（不占席）。
- **官方已知的超额漏口**：SAML/SCIM/LDAP 批量开通（会落到 Minimal Access 而不是被挡）、多人同时加成员；以及**用销售合同续更少的席位**而未先减人 → 直接产生超额费。
- GitLab.com Ultimate 的怪癖：cap=5（3 Developer + 2 Guest）时加满 5 个 Developer 后，**连不占席的 Guest 也加不进来**（issue 441504）。

## 9.4 离职与休眠回收 SOP（第 12 篇 10 节的管理侧收尾）

```bash
# 1) 冻结访问：block（不是 deactivate —— 后者本人重新登录即可自助恢复）
#    这组端点全是 POST：approve / reject / deactivate / activate / block / unblock / ban / unban
glab api -X POST "users/$UID/block" ; glab api "users/$UID" | jq '{username, state}'
# block 的两种 403：该用户由 **LDAP 置为 blocked**（要在目录侧处理，见 12 篇 10 节）、或目标是 **internal user**（bot）

# 2) 撤销其身份下的令牌：个人 PAT + 该用户拥有的项目/群组 bot 令牌（bot 不随人停用）
glab api "personal_access_tokens?user_id=$UID&per_page=100" \
  | jq -r '.[] | [(.id|tostring), .name, .state, (.expires_at // "never"), (.last_used_at // "-")] | @tsv'
glab api "personal_access_tokens?state=active&per_page=100" | jq -r '.[] | select(.user_id=='"$UID"') | .id' \
  | xargs -r -I{} sh -c 'glab api -X DELETE "personal_access_tokens/{}"'

# 3) 腾席位：block/deactivate/banned 都不再计费；但要确认他没有别的群组仍给他计费角色
glab api "users/$UID" | jq '.id' ; glab api "groups?min_access_level=10&per_page=100" | jq -r '.[].full_path'

# 4) 留证：审计事件（30 天窗口内）+ 会话截图 + 工单号
glab api "audit_events?created_after=2026-11-15T00:00:00Z&per_page=100" \
  | jq -r '.[] | select(.details.target_id=="'"$USERNAME"'") | [.created_at, .type, .author.name] | @tsv'
```

> **顺序不能反**：先 block 再清令牌会漏（他的令牌在 block 之前已建立的会话/仍在有效期的 bot 令牌继续可用）；只清令牌不 block，人的账号仍可登录并新建令牌。删除账号会让历史审批记录指向失效 user ID（第 12 篇 8.2），所以默认 **block**。

---

# 10. 容量与性能的日常治理

## 10.1 每天/每月要看的数

| 指标 | 判据 | 第一动作 |
|------|------|----------|
| **Puma 饱和度**（`puma_backlog`、活跃 worker、队列等待） | 登录/浏览变慢但 Git 操作正常 | 内存允许时增 `puma['worker_processes']`；先确认不是 DB 慢拖住 worker |
| **Sidekiq 队列深度**（含 `background_migration` 队列） | 持续积压、webhook/邮件延迟 | 加 Sidekiq 进程/节点；查慢查询；后台迁移积压就给该队列加 worker |
| **Gitaly 请求延迟** | clone/push 变慢 | 先看磁盘 IO 与仓库大小；`gitlab-rake gitlab:check` + Gitaly 日志（第 14/16 篇） |
| **数据库** | 连接数、锁等待、表膨胀、复制延迟 | 反 vacuum / `db:diagnostics`（4.3）；升级前尤其敏感 |
| **存储增长曲线** | LFS/制品/包/镜像各占多少 | 对照保留策略（第 10 篇），别只看总量 |
| **CI 分钟数与 Runner 利用率** | 排队时间 > 执行时间 | 加 Runner 还是用 `rules:changes`/workflow 裁剪流水线；成本因子（第 8 篇） |

> **把"指标名"留给第 14 篇**：管理员日常只需记住"哪四条曲线 + 一条阈值动作表"。真要配告警，Prometheus 抓取与 Grafana 面板在第 14 篇里逐项给。

## 10.2 存储增长拆解（谁把磁盘塞满）

```bash
# 实例侧粗看
sudo du -sh --exclude=backups /var/opt/gitlab/* | sort -h | tail -20
# 分类看（路径随安装与对象存储配置而变，以 gitlab.rb 为准）
for d in gitlab-data/lfs objects builds artifacts uploads registry packages; do
  printf '%-14s ' "$d"; sudo du -sh "/var/opt/gitlab/$d" 2>/dev/null | cut -f1 || echo '(不在本地)'
done
```

```bash
# 业务侧：按项目找大头的正确姿势（统计字段来自 API，别用 du 猜）
# order_by 里能按容量排的是 storage_size / repository_size / packages_size / wiki_size，且**仅管理员可用**
glab api "projects?owned=false&statistics=true&order_by=storage_size&sort=desc&per_page=20&min_access_level=40" \
  | jq -r '.[] | [.id, .path_with_namespace,
       (.statistics.storage_size/1048576|floor|tostring+"MiB"),
       (.statistics.job_artifacts_size/1048576|floor|tostring+"MiB"),
       (.statistics.lfs_objects_size/1048576|floor|tostring+"MiB"),
       (.statistics.packages_size/1048576|floor|tostring+"MiB")] | @tsv' \
  | column -t
```

治理手段按优先级：① 制品保留策略（`artifacts:expire_in` + 项目级清理）；② 镜像清理（受保护 tag 规则，第 10 篇）；③ 包清理策略（**每 12 小时跑一次，首配后约等 12 小时**）；④ LFS 去重/迁移对象存储；⑤ 休眠项目删除（7.5）。

## 10.3 会先撞上的硬限额（自建默认值）

```bash
glab api /settings | jq '{repository_size_limit, max_attachment_size, push_event_hooks_limit}'
```

| 现象 | 相关限制 |
|------|----------|
| 上传附件失败 `413 Request Entity Too Large` | `max_attachment_size`（MB）；nginx 侧 body 上限 |
| 推送报 `This repository has exceeded its size limit` | `repository_size_limit`（MB，**Premium/Ultimate**），Admin → Projects 可单项目覆盖 |
| Maven/npm/PyPI 包 413 | `*_max_file_size` 系列（**字节**）：Maven ≈3 GiB、npm/nuget ≈500 MiB、golang 100 MiB、helm 5 MiB |
| 批量打 tag 后下游没反应 | `push_event_hooks_limit = 3`（8.4） |
| 深分页 API 报错 | `offset_pagination_limit = 50000`，改 keyset 分页 |
| Webhook 部分丢失 | `web_hook_calls` 限速 + `auth.log`；超限是**限速丢弃**，不是失败重试 |
| 外部镜像同步慢 | `pull_mirror_interval_seconds = 300`（第 4 篇） |

## 10.4 参考架构怎么读

官方给的读法很务实：

- **先定规模再选架构**：按用户数与 RPS 选 class；不确定时 **"start large, monitor, then scale down"** —— 尤其是 RPS 估不出、负载可能突峰、有大型 monorepo 或额外工作负载时。
- **2000 用户及以下**：官方明确建议**走 standalone 非 HA**（单节点或多节点），用**自动化备份**换 RTO/RPO，"避免 HA 带来的复杂度"。这就是"多数团队到层级 2 就够"的官方版本。
- 架构族谱：**Linux package (Omnibus)** / **Cloud Native Hybrid** / **Cloud Native** 三类，各有 standalone（非 HA）、**HA**、**Geo（跨区域分发/容灾）**、**大型 monorepo / 额外工作负载** 变体；偏离参考架构有风险，官方另列"不支持的设计"。

---

# 11. 高可用与容灾：够用层级

## 11.1 四个层级

```text
层级1  单实例 + 好备份 + 演练            ← 官方：≤2000 用户推荐这条路线（standalone 非 HA）
层级2  应用层冗余 + PG 主从 + 备份自动化   ← 数百到上千人：PG 是全站单点，收益最大
层级3  全组件冗余 + Gitaly Cluster(Praefect) ← 数千人：Git 数据多副本，取代"用 NFS 撑 HA"的思路
层级4  Geo 多站点                        ← 多地访问延迟 / 容灾合规，而不是本地 HA
```

| 层级 | 你要额外维护的东西 | 常见错判 |
|------|--------------------|----------|
| 1 | 备份 + **恢复演练** + 维护模式流程 | "我们有备份"就当"我们能恢复" |
| 2 | Patroni/复制、连接管理（PgBouncer）、`gitlab-ctl` 分角色、备份选节点 | 忽略 PgBouncer 下备份/恢复必须绕过它（2.9） |
| 3 | Praefect 元数据库、Gitaly 节点奇数与 Consul、外部 Gitaly 先升级（4.4） | 把 Gitaly Cluster 当成 Geo（官方专门写"别混淆"） |
| 4 | 每个站点独立升级与回滚计划、tracking DB、log cursor、防火墙 | 把 Geo 当备份（11.3） |

## 11.2 官方明确不支持的做法

- **在 Kubernetes 里跑有状态的 Postgres / Redis** —— 不支持；要么用外部数据库/Redis，要么用云厂商托管服务（官方还提醒：个别 Gitaly 节点可以跑在 K8s 上，但那是**每仓库单节点的方案**）。
- **同一套环境跨多个 region 部署** —— 不支持：延迟与脑裂风险，且 Consul、Redis Sentinel、Praefect 这类组件要么需要同步复制要么需要奇数节点。要多地，**用 Geo**。
- 偏离参考架构、快照式多节点备份、PG/ES 版本自行升级等，官方都标注"超出 GitLab 支持范围"。

## 11.3 Geo 不是备份（官方逐条列出的"Geo 不适合"）

| 你指望它做 | 官方结论 |
|------------|----------|
| 抵御数据损坏/误删/恶意破坏 | **Geo 会把 primary 上的损坏复制到所有 secondary**，要用它就必须**再配备份**（原文：`complement Geo with backups`） |
| 访问控制 / 数据出境合规 | secondary 的只读代理不是**一级特性**，未来可能不再支持；selective sync 是为省流量与存储，不是合规手段；权限要交给 auth/authorization |
| 替代零停机升级 | 明确列为"不是为此设计" |
| 主主（active-active）HA | 不支持，secondary 是读 + 代理，写入永远在主站 |

Geo 真正的收益：多地读写延迟、CI 流量卸载、迁移到新实例时的**后台预拷贝 + 短切换窗口**（比"备份+恢复"迁移法停机短得多；但不能用于跨 OS 迁 PG）。

---

# 12. Admin Area 地图（管理员日常入口）

| 入口 | 干什么 | 本篇对应 |
|------|--------|----------|
| **Overview → Users** | block / ban / deactivate / delete 四类处置、**New user 里选 User type = Auditor**、待审批与待角色提级队列、2FA 状态 | 第 12 篇 1.2/8.2/9.1；本篇 9.3–9.4 |
| **Overview → Projects / Groups** | 全局搜索、归属转移、归档、单项目限额覆盖、scheduled deletion | 6.2 / 7.2 |
| **Settings → General** | 注册、可见性、Git 协议、**维护模式**、删除保护与冷静期、出网开关、全局 IP 允许段、推送规则 | 6.1–6.4 |
| **Settings → Account and limits** | 2FA 强制与宽限期、**PAT 必须过期日**、PAT 前缀、令牌/SSH key 生命周期、附件与仓库大小、push 限额、休眠用户自动停用、席位控制（user cap / restricted access） | 6.2 / 9.3 / 10.3 |
| **Settings → Repository** | 仓库维护：housekeeping、**休眠项目删除** | 7.4 / 7.5 |
| **Settings → CI/CD** | 全局变量、Runner 注册与实例 Runner 额度、Auto DevOps | 第 8 篇 |
| **Monitoring → Audit events** | 实例级审计（**Premium/Ultimate，仅 Self-Managed**；≤30 天窗口；CSV 导出上限 10 万条） | 第 12 篇 9.1 |
| **Monitoring → Background jobs / Background migrations** | Sidekiq 与后台迁移状态（命令行等价物见 4.7） | 4.7 / 10.1 |
| **Monitoring → Database pre-upgrade checks** | Schema health check、search path | 4.3 |
| **Monitoring → Database diagnostics** | Vacuum 信息（**Anti-wraparound** 必看）、复制与膨胀 | 4.3 |
| **Runners** | 实例 Runner 池、锁定/健康、成本因子 | 第 8 篇 |
| **System hooks** | 实例级事件出口 | 8.5 |
| **Abuse reports / Spam / Pending users** | 公网实例的举报、垃圾内容处置、新账号审批队列 | 第 12 篇 8.2；本篇 9.3 |
| **Licenses / Seat usage** | 订阅与席位用量、导出 | 9.1–9.3 |
| **Health Check** | `/-/health`、`/-/liveness`、`/-/readiness?all=1`、`/health_check*`（受 IP allowlist 管控） | 5.2 |
| **Help → instance_configuration** | 当前实例配置与限额自检页 | 5.5 |

> **一条工作纪律**：Admin Area 的每一项，能对应到 `GET /api/v4/application/settings` 的键就用 API 管（可版本化、可评审、可回滚）；不能对应的（后台迁移、数据库诊断、举报队列）才在 UI 里看。把"UI 里点过什么"变成"MR 里改了什么"，是第 15 篇最佳实践在管理面的落点。

---

# 13. 小结

| 主题 | 一句话结论 |
|------|------------|
| 备份范围 | 包里有 DB/仓库（含 wiki、最多 3 个仓库/项目）/LFS/制品/registry/包/Pages/snippets/wikis；**对象存储里的数据、Redis、Elasticsearch、`/etc/gitlab`、证书与 server/file hooks 都不在包里** |
| 一致性窗口 | Redis 不备份 ⇒ 官方要求**要一致就必须没有 pending/running 后台 job**；日常在线备份只能接受"队列丢失"，并在恢复后核对 job 历史 |
| 备份体积 | 备份对每个仓库产 bundle，**pool 数据会被复制进每个 fork**；官方实测 100 GB 仓库≈2 小时（含上传 S3），**约 400 GB 仓库时备份命令基本不可用**；**18.0 起多分支/多标签仓库备份大幅提速（小时→分钟，零配置）** |
| 备份参数 | `STRATEGY=copy` 换磁盘空间解 `file changed as we read it`；压缩变量要放命令**前面**；自定义 `BACKUP=` 与本地保留期互斥；增量只对仓库有效且**链断即不可恢复** |
| 大实例 | PG **>100 GB** 时 `pg_dump` 路线基本不可用，改走 server-side / 增量 / 原生备份 |
| secrets | 故意不进包（密钥与密文不能同处），丢了＝2FA 用户进不来、Runner 登录不了；**跨 FQDN 恢复会禁用全部 WebAuthn** |
| 恢复前提 | 同版本同 CE/EE、目标干净（PG 自动清、同名仓库报 `repository already exists`、**桶要手工清**）、空间 **≥3×**、secrets 到位、仓库存储名对齐、挂载点为空 |
| PgBouncer | **禁止**经 PgBouncer 备份/恢复：会先 `PG::UndefinedTable`、随后持续 500 + `relation "tablename" does not exist`；用 `GITLAB_BACKUP_PGHOST/PGPORT`（多库加 `_MAIN_`/`_CI_`）直连主库 |
| 恢复顺序 | 官方编排是 stop puma/sidekiq → **restore → ANALYZE → reconfigure → start → checks**；ANALYZE 在放流量之前，不是之后 |
| 恢复收尾 | 放流量前逐库 `ANALYZE VERBOSE`（必要时 `VACUUM (VERBOSE, ANALYZE)`），再跑 `doctor:secrets` + 三个 `:check`；**Docker Swarm 要临时关 healthcheck**否则恢复中途被重启 |
| 恢复的"隐性失败" | 扩展属主类 warning **不影响恢复成功**；但**仓库全空（fapolicyd）**、**registry 未启用则不恢复镜像**、**恢复后 push 报 permission denied 要 `chown -R registry:registry`**、`is a directory` 的 custom hooks 失败（15.10 方式 + ≥15.11）都是"报告成功其实坏了" |
| 部分恢复 | **没有** `--project`：只能"临时同版本实例恢复 → 导出 → 生产导入"；仓库级可用 `REPOSITORIES_PATHS` / `SKIP_REPOSITORIES_PATHS`（19.4 起后者不再删仓库；17.2 起才安全） |
| 升级路径 | 停靠站 **x.2 / x.5 / x.8 / x.11**（17.5 起固定），每站**必须显式指定包版本**并**等后台迁移跑完**；用 Upgrade Path tool 排程 |
| 升级前后 | `gitlab:check` + `doctor:secrets` + **Database pre-upgrade checks**（结果只留一周）+ **Anti-wraparound vacuum 期间不要动手**；外部 Gitaly 与 Runner 先升 |
| 停机窗口 | 先 pause Runner，再用 nginx 对 `/api/v4/jobs/request` 返回 503；制品只重试 3 次，别把 CI 留在线上挨打 |
| 运维入口 | 探活端点四类各有用途，**`/health_check` 不能当 LB 探针**；维护模式下 Git 读可用、写报固定错误串；只读分四档且每一档都要写解法 |
| 设置下发 | 读回现值 → 改 → 再读；`default_branch_protection` 已弃用改 `default_branch_protection_defaults`；大量默认值**只管新建**，老项目要脚本刷 |
| 事件出口 | 通知用 webhook（有 4/40 次自动禁用）、审计兜底用 system hooks、强一致放 CI；**19 起支持 HMAC signing token**，可双配平滑迁移 |
| 席位 | 计费按**周期内峰值**，block/deactivate 不计费，**Auditor 也计费**；user cap 与 restricted access 互斥，无席位时目录同步落 Minimal Access |
| 容量 | 看 Puma/Sidekiq/Gitaly/PG 四条曲线 + 存储拆解；自建硬默认限额里最容易撞的是 push-hooks=3、附件/包大小、深分页 50000 |
| HA | **≤2000 用户走 standalone**；不确定就"先大后小"；不支持 K8s 里有状态 PG/Redis、不支持单环境跨 region；**Geo 会复制损坏，不能当备份** |

**自检清单**：

- [ ] 我能说出备份包里**没有**哪些数据，以及为什么 `gitlab-secrets.json` 故意不进包、丢了会发生什么？
- [ ] 我们的 PG 与仓库各多大？按官方阈值（100 GB / 400 GB）现在该走哪条路线，**"没有 pending/running job"这个一致性条件我们怎么满足**？
- [ ] 用了 PgBouncer 吗？备份/恢复命令里有没有 `GITLAB_BACKUP_PG*`？
- [ ] 给我一台同版本的干净机器，我能按顺序讲出恢复的 7 项前提与版本不匹配时的报错文本，以及恢复后放流量之前必须做的两件事？
- [ ] 我能解释"为什么只恢复一个项目要走临时实例 + 导出导入"，并说出 `REPOSITORIES_PATHS` 与全量恢复在"删除已有仓库"上的行为差异？
- [ ] 我们的当前版本到目标版本要停几站？每站之间要等什么、谁有权判定"可以继续"？
- [ ] 升级窗口内我会怎么暂停 CI？为什么顺序是"先 pause Runner 再拦 jobs/request"？
- [ ] 我能区分四个探活端点，并说清"为什么不能拿 `/health_check` 做 LB 健康检查"？
- [ ] 给我一个实例设置，我能先说出**键名与单位**（小时/天/MB/字节），再说出"它只管新建还是回刷老对象"？
- [ ] 我能说出 webhook 连续失败被禁用的两个阈值、恢复办法，以及"批量打 tag 不触发下游"的根因？
- [ ] 有人离职，我能按顺序讲 4 步回收，并解释 block 与 deactivate 在席位和安全上的双重差别？
- [ ] 我能用三个数（RPO/RTO/抽样通过率）向管理层证明我们的备份策略是否达标？

治理之外还要"看得见"，下一篇讲 GitLab 自带的监控观测能力与日志排错入口。
