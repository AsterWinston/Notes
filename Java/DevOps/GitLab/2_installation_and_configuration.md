# 1. 安装方式怎么选

GitLab 官方支持多种安装途径，选型只看一个问题：**你要不要自己运维它**，以及运维团队的技术栈长什么样。

## 1.1 方式对比

| 方式 | 适合 | 优点 | 代价 |
|------|------|------|------|
| Omnibus 包（RPM/DEB，官方叫 Linux package） | 生产自建主力方案 | 一键装齐 NGINX/PostgreSQL/Redis/Gitaly/Prometheus，升级路径清晰 | 目录结构固定，自定义必须走 `gitlab.rb` |
| Docker（`gitlab/gitlab-ce`、`gitlab/gitlab-ee` 官方镜像） | 本地体验、小规模、CI 里起临时实例 | 环境干净、随起随停 | 数据卷/备份自己管好；**镜像内部就是一个 Omnibus**，该吃的资源一点不少 |
| Helm Chart（K8s，CNG 镜像拆分组件） | 已有 K8s 平台和运维能力 | 组件级伸缩、声明式 | 运维心智负担大；存储、PG、Redis 高可用全要自备 |
| GitLab.com / GitLab Dedicated | 不想运维 | 零运维 | 数据在厂商侧、合规管控弱，国内团队慎用 |
| Geo / Gitaly Cluster | 多地容灾、Git 层高可用 | 读写分离复制 / 副本冗余 | 复杂度陡增，小团队别碰 |
| 源码 self-compiled | 特殊平台或极客 | 完全自由 | 文档仍在、但支持口径与升级成本和包安装完全不是一个量级，**生产不要选** |

## 1.2 决策三问

1. **要不要自己运维？** 不要 → SaaS/Dedicated；要 → 下一问。
2. **运维栈是什么？** Linux 传统运维 → Omnibus；K8s 平台成熟 → Helm；都没有 → Omnibus，别为了时髦上 Chart。
3. **用来干什么？** 学习/演示 → Docker 单容器；生产 → Omnibus 包（或 Chart），**试用和生产的安装介质不同不影响数据迁移**（都是 Omnibus 世界观，备份恢复跨介质可行，见第 13 篇）。

## 1.3 关于 Docker 安装的常见误会

- "Docker 版更轻量"：不对。容器里跑的还是那套 Omnibus 服务集，Rails 该吃 1GB+ 还是吃 1GB+，所以官方明确内存要求不打折。
- "镜像 tag 里 `-ce.0`/`-ee.0` 后缀能省"：不能，`gitlab/gitlab-ee:18.5.1-ee.0` 这种完整 tag 才能对上包仓库里的发布版本；用 `latest` 会在某天升级时把你打个措手不及。
- "CI 里起临时实例测集成"：可行，但初始化要几分钟、健康检查要等 `gitlab Reconfigured!`，记得把 `/etc/gitlab` `/var/opt/gitlab` 挂到缓存卷，否则每次冷启动重建数据库。

# 2. 硬件与系统要求

GitLab 是 Rails 单体 + 一堆辅助进程，历史上"4C4G 能跑"是老文档的说法。**现行官方要求已经上调**，别再按老口径给实例申请资源。

## 2.1 官方基线（现行文档口径）

| 项 | 现行官方要求 | 备注 |
|----|-------------|------|
| CPU | 单机 **8 vCPU 起步** | 支持 ARM；**不推荐突发性能实例**（burstable，性能抖动大） |
| 内存 | 单机 **16 GB 起步** | 内存受限场景最低 8 GB，走 memory-constrained 调优（见 5.2） |
| Swap | **能关就关** | 官方口径反转：swap 在高负载下性能劣化明显；关不掉就把内存配足到永远用不到它 |
| 应用节点磁盘 | 40 GB | 包安装本体约 2.5 GB，其余给系统、日志、临时文件 |
| Gitaly 磁盘 | ≥ 全部仓库总和 | SSD，重 I/O；见第 3 篇的 IOPS 参考 |
| PostgreSQL 磁盘 | 5–10 GB（Ultimate ≥12 GB） | 漏洞数据就要 1 GB |
| 网络文件系统 | **避免 NFS/EFS/Azure Files** | 官方点名会造成严重性能问题 |

老笔记里"内存低于 4 GB 必须加 swap"是过时建议，两个方向都反了：既不是 4 GB，也不该指望 swap。

## 2.2 支持的操作系统

官方为以下平台构建包（以 `install/package` 下分平台文档为准）：**Ubuntu、Debian、SUSE、AlmaLinux、Amazon Linux 2/2023**。注意 CentOS/RHEL 系的官方包路径已经不在列表里——RHEL 衍生迁移方向就是 AlmaLinux。官方承诺：停建某 OS 版本至少提前 6 个月通知，且跟随厂商标准/维护支持期（不含扩展支持期），所以选 LTS 发行版、赶在 EOL 前换底座是管理员的固定功课。

## 2.3 组件版本要求（外部服务对接前先对这张表）

| 组件 | 版本要求（现行文档） |
|------|---------------------|
| PostgreSQL | GitLab 19.x：最低 17.x、最高 18.x；18.x：最低 **16.5**、最高 17.x；17.x：最低 14.14、最高 16.x |
| PG 扩展 | 必需：`amcheck`(18.4+)、`btree_gist`、`pg_trgm`、`plpgsql`；推荐 `pg_stat_statements`。装扩展需要超级用户 |
| Redis / Valkey | Redis 推荐 7.2（最低 7.0）；**Valkey 已官方支持**（7.2）。**不支持 Redis Cluster**，不支持 serverless 形态（ElastiCache Serverless 之类不行） |
| Sidekiq 资源 | 每进程至少 200 MB 内存；万人级按每进程 ≥1 GB 规划 |

两个容易翻车的点：外部 PG 必须按文档的"external instances 必需参数"配置（编码、扩展、超时参数缺一不可）；Redis 的淘汰策略要按文档设置——缓存全被驱逐和队列丢失是两种不同的事故。

## 2.4 高可用的网络与拓扑红线

- 节点间延迟 **< 5 ms**（同步复制的硬条件）；
- 跨可用区部署建议**奇数个 AZ**（quorum）；跨机房要奇数个同地域机房 + 冗余链路；
- **单个 GitLab 实例不允许横跨多个地理区域**——多区域请上 Geo（第 13 篇的 Geo 边界节）。

# 3. Omnibus 安装（生产标准）

以 Ubuntu 为例：

```bash
# 1) 安装依赖与官方源
sudo apt install -y curl openssh-server ca-certificates tzdata perl
# 2) 添加 GitLab CE 仓库
curl -fsS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
# 3) 安装并指定外部访问地址（配了 https 会顺带尝试 Let's Encrypt 证书）
sudo EXTERNAL_URL="https://gitlab.example.com" apt install gitlab-ce
```

RHEL 系（Alma）把 `script.deb.sh` 换成 `script.rpm.sh`、`apt install` 换成 `dnf install`，其余同理。

## 3.1 Omnibus 的世界观：一个文件 + 一条命令

**一个配置文件 `/etc/gitlab/gitlab.rb` 管所有组件**，`gitlab-ctl reconfigure` 用 Chef 把它翻译成 NGINX 配置、database.yml、systemd 单元等"生成物"（例如生成后的 NGINX 站点配置在 `/var/opt/gitlab/nginx/conf/service_conf/gitlab-rails.conf`）。推论三条：

1. 改完任何配置**必须 reconfigure** 才生效；
2. reconfigure 幂等、可放心反复执行，但它会**覆盖手改过的生成文件**——所有定制都走 gitlab.rb；
3. `gitlab.rb` 里有明文密码（数据库连接串），权限必须是 **0600**，这本身就是官方文档特意标注的一条。

## 3.2 首次登录与善后

用户名 `root`；安装时没设密码的话，随机密码在：

```bash
sudo cat /etc/gitlab/initial_root_password
```

这条文件的存在本身就是风险（能 sudo 的人都能看到），官方安装后置步骤文档把它列为"初始登录"的第一件事。改完密码，**立刻手动删掉这个文件**，再走 5 节的安全基线三件事。

## 3.3 装完就能跑，不代表配好了

`gitlab-ctl status` 全绿只是进程活着。装完当天要做的验证：

```bash
sudo gitlab-rake gitlab:env:info          # 组件版本与路径快照
sudo gitlab-rake gitlab:check SANITIZE=true   # 数据与权限自检（恢复演练同款，第 13 篇）
sudo gitlab-rails runner 'puts Gitlab::CurrentSettings.current_application_settings.id' # Rails 能摸到数据库
```

外加浏览器访问 `https://你的域名/-/graphql-explorer` 或登录后 Admin Area 转一圈——Web、Git、API 三条线各测一次才算装完（三条线的含义见第 3 篇）。

# 4. Docker 方式快速起一个（体验用）

官方 Docker 安装命令长这样（CE 把 `-ee` 换成 `-ce` 并把 tag 后缀对应改掉）：

```bash
sudo docker run --detach \
  --hostname gitlab.example.com \
  --env GITLAB_OMNIBUS_CONFIG="external_url 'https://gitlab.example.com'" \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  --shm-size 256m \
  gitlab/gitlab-ee:<version>-ee.0
```

## 4.1 三个卷与就绪判断

- `$GITLAB_HOME/config:/etc/gitlab`（gitlab.rb + **密钥文件**）、`logs:/var/log/gitlab`、`data:/var/opt/gitlab`，缺一不可；
- 首次启动 3-5 分钟建库迁移 schema，`docker logs -f gitlab` 看到 `gitlab Reconfigured!` 才算就绪；
- `--shm-size 256m` 不是装饰：缺了它，NGINX/PG 共享内存不足会以玄学报错。SELinux 主机上卷挂载要加 `:Z` 后缀。

## 4.2 改 SSH 端口：先想清楚改哪一头

要用非 22 端口跑 SSH 克隆，有两条路（官方 Docker 文档原方案）：

1. **改宿主 sshd 端口**（推荐）：克隆 URL 不带端口号，`ssh://git@gitlab.example.com/user/project.git` 保持干净；
2. **只改发布端口 + 告知 GitLab 显示值**：

```bash
sudo docker run --detach \
  --hostname gitlab.example.com \
  --env GITLAB_OMNIBUS_CONFIG="external_url 'http://gitlab.example.com:8929'; gitlab_rails['gitlab_shell_ssh_port'] = 2424" \
  --publish 8929:8929 --publish 2424:22 \
  ... \
  gitlab/gitlab-ee:<version>-ee.0
```

注意键名是 **`gitlab_rails['gitlab_shell_ssh_port']`**（管克隆 URL 显示），它只影响"页面告诉你怎么连"，真正的端口翻译由 `-p 2424:22` 完成——两边不匹配时页面给了 URL 也连不上，这是 Docker 安装的头号玄学坑。

## 4.3 备份与"配置不在 gitlab.rb 里"的陷阱

容器内备份就是包安装的命令穿一层 shell：

```bash
docker exec -t gitlab gitlab-backup create
```

官方文档特意警告：如果你的配置全部来自 `GITLAB_OMNIBUS_CONFIG` 环境变量，那 **`gitlab.rb` 里是没有这些内容的**——恢复时"配置 + 密钥"（`/etc/gitlab` 里的 `gitlab-secrets.json` 等）仍需单独保全（第 13 篇的 secrets 清单在 Docker 场景同样成立）。

# 5. gitlab.rb 高频配置项

不必背全表，记住这几组最常用的，以及每组"改了要 reconfigure"。

## 5.1 入口与显示

```ruby
external_url 'https://gitlab.example.com'            # 决定 NGINX/证书/克隆 URL 的总入口
gitlab_rails['gitlab_shell_ssh_port'] = 2424         # 克隆 URL 显示的 SSH 端口（非默认时才配）
gitlab_rails['time_zone'] = 'Asia/Shanghai'          # Rails 时区；页面时间与 cron 排班相关
```

## 5.2 性能：Puma 与 Sidekiq

官方给出的 worker/thread 规划表（**每节点 worker 不得少于 2**，除内存受限场景）：

| 机器规格 | Puma workers | Puma threads |
|----------|--------------|--------------|
| 2C / 8 GB | 2 | 4 |
| 4C / 8 GB | 4 | 4 |
| 8C / 8 GB | 6 | 4 |
| 8C / 16 GB | 8 | 4 |
| 内存 ≤ 2 GB | —（已不满足官方底线） | 1 |

老经验公式"(内存GB − 2) 个 worker"和自动探测结果接近，但规划请直接按上表走。**内存受限环境（8 GB 档）**的官方减配组合：

```ruby
puma['worker_processes'] = 0        # Puma 单进程模式，省 100-400 MB（副作用：并发能力大幅下降）
sidekiq['concurrency'] = 10         # 默认 20；内存紧张降到 10 甚至 5
prometheus['enable'] = false        # 关掉内置监控栈（有外部监控时的常规操作）
```

Sidekiq 的键注意：现行主键是 `sidekiq['concurrency']`（部分教程写 `max_concurrency` 是旧名/扩展写法）。另外 Puma worker 有内存看门狗：单 worker RSS 超限会被重启并在 `application_json.log` 留下 `"message": "rss memory limit exceeded"` 的 WARN（`per_worker_max_memory_mb` 可调）——页面间歇性变慢时先 grep 这条日志。

## 5.3 限额与入口防护

```ruby
gitlab_rails['backup_keep_time'] = 604800          # 本地备份保留 7 天（单位秒），详见第 13 篇
nginx['client_max_body_size'] = '250m'             # 单次 push/上传体积上限（改后看生成的 gitlab-rails.conf 验证）

gitlab_rails['rack_attack_git_basic_auth'] = {     # Git 基本认证暴力破解封禁
  'enabled' => true,                               # 默认 false！公网实例建议开
  'ip_whitelist' => ["127.0.0.1"],
  'maxretry' => 10, 'findtime' => 60, 'bantime' => 3600,
}
```

被封时返回 **403**，排查"某些 IP 突然连不上 git"记得查这条封禁而不是先怀疑防火墙。NGINX 的生效值是 reconfigure 后生成的配置说了算，改完直接读 `/var/opt/gitlab/nginx/conf/service_conf/gitlab-rails.conf` 确认，别信"我记得默认是 250m"。

## 5.4 用外部组件时怎么关内置

**键名以包内默认模板 `gitlab.rb` 注释为准**，最常见的几个（注意 PostgreSQL 的组件键是 `postgresql` 不是 `postgres`）：

```ruby
# 外部 PostgreSQL
postgresql['enable'] = false
gitlab_rails['db_adapter'] = 'postgresql'
gitlab_rails['db_encoding'] = 'utf8'
gitlab_rails['db_host'] = 'pg.example.com'          # 多个逗号分隔地址时，只用第一个连接
gitlab_rails['db_port'] = 5432
gitlab_rails['db_username'] = 'gitlab'
gitlab_rails['db_password'] = 'REDACTED'

# 外部 Redis
redis['enable'] = false
gitlab_rails['redis_host'] = 'redis.example.com'
gitlab_rails['redis_port'] = 6379

nginx['enable'] = false                              # 前置自己的负载均衡
```

关内置 PG 后首次要在新库上 **seed**（建 schema），不是把数据"自动接管"过去——全新安装才好办，存量迁移走第 13 篇的迁移路径。外部 PG 还必须满足 2.3 的版本与扩展清单，缺 `pg_trgm` 之类的后果是 reconfigure 成功、用着用着查询报错。

## 5.5 SMTP 与邮件

```ruby
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.example.com"
gitlab_rails['smtp_port'] = 587
gitlab_rails['smtp_user_name'] = "gitlab@example.com"
gitlab_rails['smtp_password'] = "REDACTED"
gitlab_rails['smtp_domain'] = "example.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_openssl_verify_mode'] = 'peer'
gitlab_rails['gitlab_email_from'] = 'gitlab@example.com'
```

不配 SMTP：邀请、密码重置、MR 通知全哑。云厂商封 25 端口是高频原因，改 465/587 + TLS。验证发送不用写代码，让 Rails 自己发一封：

```bash
sudo gitlab-rails runner "Notify.test_email('you@example.com', 'Hello World', 'This is a test message').deliver_now"
```

收信侧（邮件回复 Issue/MR）是另一套 `incoming_email`，默认关，要用再看文档。

# 6. gitlab-ctl / gitlab-rake / gitlab-rails 三件套

| 命令 | 用途 |
|------|------|
| `gitlab-ctl status` | 各组件（nginx/puma/sidekiq/gitaly/postgresql/redis/...）运行状态 |
| `gitlab-ctl restart puma` / `stop`/`start` | 按组件粒度操作，别一上来 restart 全家 |
| `gitlab-ctl tail puma` | 实时看某组件日志（日志全在 `/var/log/gitlab/<组件>/`） |
| `gitlab-ctl reconfigure` | 应用 gitlab.rb 变更 |
| `gitlab-rake gitlab:check SANITIZE=true` | 数据与权限自检（排错第一招） |
| `gitlab-rake gitlab:env:info` | 版本/路径快照，提 issue 必备 |
| `gitlab-backup create` | 全量备份（策略详见第 13 篇） |
| `gitlab-rails console` | Ruby 控制台直接操作数据（重置 2FA、改用户等） |
| `gitlab-rails runner '代码'` | 非交互执行一行 Ruby，脚本化首选 |

三条使用纪律：console/runner 默认就是 production 上下文不用再 `-e production`；只读诊断用 `runner`，写操作才进 console 并随手退出；所有写库操作前先确认自己连的是哪台实例（别名/跳板机场景最危险）。

# 7. 升级与回滚

## 7.1 现行升级规则（和第 13 篇一致）

老口径"每个小版本都要跟"已经作废。现行规则（17.5 起官方固定节奏）：

1. **季度停点**：升级路径上必须停靠 `x.2.z`、`x.5.z`、`x.8.z`、`x.11.z`（18.x、19.x 全部适用）；
2. 每个停靠点**等后台迁移完成**再继续（`glab api` 或第 13 篇后台迁移节的方法判断）；
3. 同 minor 内永远升到**最新补丁版**再走（`16.8.7` 而不是 `16.8.0`）；
4. 目标版本对 PG 的版本要求先查 2.3 的表——跨大版本时数据库往往要先行。

```bash
# Omnibus 指定版本升级（版本号到 packages.gitlab.com 仓库页查最新补丁）
sudo EXTERNAL_URL="https://gitlab.example.com" apt install gitlab-ce=18.5.1-ce.0
```

升级窗口内 Web 只读（schema 迁移），选低峰；大版本升级的完整编排（备份→停机位→迁移→检查）在第 13 篇升级节。

## 7.2 回滚的现实

官方文档对 rollback 的定性是：**降级不是受支持的操作**——数据库迁移大多不可逆，schema 版本号不认旧代码。回滚的真正兜底是升级前的 `gitlab-backup create` + VM 快照；拍脑袋"dnf downgrade"能回二进制、回不了数据。所以每次升级的 checklist 里，"备份已产出且校验过"必须排在"点了 apt install"之前。

# 8. 常见安装坑

| 现象 | 原因与处理 |
|------|-----------|
| reconfigure 报 `Memory exhausted` | 内存不足：按 5.2 减配 worker/concurrency，或按 2.1 加内存（swap 不是解法） |
| 页面持续 502 | Puma/Sidekiq 还在启动，等 2 分钟；持续 502 `gitlab-ctl tail puma`，再看是否撞上 worker 内存看门狗重启循环（5.2） |
| reconfigure 出现 `Runtime directory '/run' is not a tmpfs` 警告 | `/run` 不是 tmpfs，Rails 指标被自动关闭；要指标就 `runtime_dir '/path/to/tmpfs'`（Omnibus 文档原方案） |
| git clone SSH 失败 / 端口不对 | 发布端口与 `gitlab_rails['gitlab_shell_ssh_port']` 不匹配（4.2），或防火墙没放行 |
| 邮件发不出 | SMTP 未配或云厂商封 25 端口，改 465/587 + TLS，用 `runner` 发测试信验证 |
| 时间显示差 8 小时 | `gitlab_rails['time_zone']` + reconfigure；同时核对宿主 NTP |
| 某些 IP 突然 403 连不上 git | `rack_attack_git_basic_auth` 封禁生效（5.3），查白名单而不是查防火墙 |
| 外部 PG 装好后部分搜索/统计功能报错 | 必需扩展没装全（2.3：`pg_trgm`、`btree_gist`、`amcheck`），或版本不在支持矩阵内 |

# 9. 小结

| 要点 | 一句话 |
|------|--------|
| 选型 | 体验用 Docker（里面还是 Omnibus），生产用包或 Helm，不要源码安装 |
| 资源 | 现行基线 8C/16G、能关 swap 就关；NFS 系文件系统官方点名避开 |
| 版本矩阵 | PG/Redis 支持窗口跟着 GitLab 大版本走，Redis Cluster 不支持、Valkey 可用 |
| 配置入口 | 一切改 `gitlab.rb` + `reconfigure`；生成文件会被覆盖，明文密码文件 0600 |
| 性能旋钮 | Puma worker 按官方表（≥2）、Sidekiq 默认 20 可降、内存受限走单进程 + 关监控 |
| 升级 | 季度停点 x.2/x.5/x.8/x.11 + 等后台迁移 + 最新补丁；降级不受支持，备份是唯一后悔药 |

自检清单：

- [ ] `initial_root_password` 已改密并删除？
- [ ] 注册策略、`rack_attack_git_basic_auth`、SMTP 三件套配了？
- [ ] 外部 PG/Redis 的版本与扩展都在支持矩阵内？
- [ ] `gitlab-rake gitlab:check SANITIZE=true` 全绿？
- [ ] 升级路径、季度停点、备份编排对齐了第 13 篇？

下一篇拆解 GitLab 的内部架构：一次 `git push` 请求进来之后，NGINX、Workhorse、Rails、Gitaly 各自干了什么。
