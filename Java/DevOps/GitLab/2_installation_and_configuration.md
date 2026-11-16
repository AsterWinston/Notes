# 1. 安装方式怎么选

GitLab 官方支持多种安装途径，选型只看一个问题：**你要不要自己运维它**。

| 方式 | 适合 | 优点 | 代价 |
|------|------|------|------|
| Omnibus 包（RPM/DEB） | 生产自建主力方案 | 一键装齐 NGINX/PostgreSQL/Redis/Gitaly，升级简单 | 目录结构固定，自定义受限 |
| Docker（gitlab/gitlab-ce 镜像） | 本地体验、小规模、CI 里起临时实例 | 环境干净、随起随停 | 数据卷/备份需自己管好，官方镜像本质也是 Omnibus |
| GitLab Geo / 集群 | 多地容灾 | 读写分离复制 | 复杂度高，小团队别碰 |
| GitLab.com SaaS | 不想运维 | 零运维 | 数据在境外、管控弱，国内团队少选 |
| Source from scratch | 几乎不再推荐 | 完全自由 | 官方明确表示源码安装不受支持，升级是噩梦 |

**使用建议**：学习/试用用 Docker；生产用 Omnibus；已经在 K8s 里就用官方 Helm Chart（同样是 Omnibus 镜像）。

# 2. 硬件与系统要求

GitLab 是 Rails 单体 + 一堆辅助进程，**吃内存**是它最出名的特点：

| 规模 | CPU | 内存 | 说明 |
|------|-----|------|------|
| 500 人以下（官方最小值） | 4 核 | 4 GB | Puma 工作进程会按内存自动调，4G 只能跑 1 个 worker |
| 1000 人左右 | 8 核 | 8 GB | 推荐起步 |
| 万人级 | 16 核+ | 16-32 GB | 需要把 PostgreSQL/Redis/Gitaly 拆到独立机器 |

- 操作系统：Ubuntu/Debian、CentOS/RHEL 及衍生（Rocky/Alma）官方支持；内存低于 4 GB 时必须加 swap，否则 `reconfigure` 阶段直接 OOM。
- 存储：Git 仓库 + LFS + CI 制品 + 备份都落本地盘，**SSD 起步**，并预留 2 倍仓库大小做备份空间。

# 3. Docker 方式快速起一个（体验用）

```bash
docker run -d --name gitlab \
  --restart unless-stopped \
  -p 443:443 -p 80:80 -p 22:22 \
  -v /srv/gitlab/config:/etc/gitlab \
  -v /srv/gitlab/logs:/var/log/gitlab \
  -v /srv/gitlab/data:/var/opt/gitlab \
  registry.gitlab.com/gitlab-org/build/cng/gitlab-ce-mirror:latest   # 或 Docker Hub 的 gitlab/gitlab-ce:latest
```

要点：

- 三个卷分别对应配置、日志、数据，**缺一不可**；config 卷里有初始 root 密码文件。
- 第一次启动要 3-5 分钟初始化（建库、迁移 schema），`docker logs -f gitlab` 看到 `gitlab Reconfigured!` 才算就绪。
- 想 SSH 用非 22 端口时，需在 `gitlab.rb` 里同步改 `gitlab_shell['ssh_port']`，否则克隆地址显示错误。

# 4. Omnibus 方式（生产标准）

以 Ubuntu 为例：

```bash
# 1) 安装依赖与官方源
sudo apt install -y curl openssh-server ca-certificates tzdata perl
# 2) 添加 GitLab CE 仓库
curl -fsS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
# 3) 安装并指定外部访问地址（会顺带申请 Let's Encrypt 证书）
sudo EXTERNAL_URL="https://gitlab.example.com" apt install gitlab-ce
```

Omnibus 的世界观很简单：**一个配置文件 `gitlab.rb` 管所有组件**，`gitlab-ctl reconfigure` 会把它翻译成 NGINX 配置、数据库连接串、systemd 服务等。改完任何配置都要 reconfigure 才生效。

# 5. 初始化必做的几件事

首次登录后（用户名 root，初始密码见下），立刻处理：

```bash
# root 初始密码（安装后 24 小时内有效，之后文件自动删除）
sudo cat /etc/gitlab/initial_root_password
```

| 事项 | 位置/操作 | 为什么 |
|------|----------|--------|
| 改 root 密码 | User settings | 初始密码文件谁都能 sudo 看到 |
| 关闭公开注册 | Admin → Settings → Sign-up restrictions | 默认开放注册，公网实例会被机器人注册灌水 |
| 配置 SMTP | `gitlab.rb` → `gitlab_rails['smtp_*']` + reconfigure | 不配就收不到邀请、通知、MR 提醒邮件 |
| 设置时区/语言 | 个人 Profile → Preferences | 影响 Issue 与流水线的显示时间 |

# 6. gitlab.rb 高频配置项

不必背全表，记住这几组最常用的：

```ruby
# gitlab.rb —— Omnibus 核心配置文件

external_url 'https://gitlab.example.com'          # 对外访问地址，决定 NGINX/证书/克隆 URL

# 性能：Puma 工作进程（Rails 应用并发）
puma['worker_processes'] = 4                        # 按 (内存GB - 2) 估算，别超
sidekiq['max_concurrency'] = 10                     # 异步任务并发

# 备份保留
gitlab_rails['backup_keep_time'] = 604800           # 7 天，单位秒

# 防止仓库无限膨胀的入口限制（前置 NGINX 同理）
nginx['client_max_body_size'] = '250m'              # 单次 push/上传体积上限

# 关闭不需要的内置组件（已有独立服务时省资源）
postgres['enable'] = false                          # 配外部 PostgreSQL
nginx['enable'] = false                             # 前置自己的 NGINX/负载均衡
```

> **注意**：`gitlab.rb` 默认全文被注释，取消注释才能改；改完必须 `sudo gitlab-ctl reconfigure`。reconfigure 幂等，可以随时放心执行，但它会重写各组件配置，**手工改过的 nginx.conf 会被覆盖**——所有定制都应走 gitlab.rb。

# 7. gitlab-ctl 与 gitlab-rake 日常命令

| 命令 | 用途 |
|------|------|
| `gitlab-ctl status` | 查看各组件（nginx/puma/sidekiq/gitaly/...）运行状态 |
| `gitlab-ctl restart` | 重启全部服务 |
| `gitlab-ctl tail puma` | 实时看某组件日志（日志都在 `/var/log/gitlab/`） |
| `gitlab-ctl reconfigure` | 应用 gitlab.rb 变更 |
| `gitlab-backup create` | 全量备份 |
| `gitlab-rake gitlab:check` | 自检安装是否健康（排错第一招） |
| `gitlab-rails console` | Ruby 控制台直接操作数据（重置 2FA、改用户等） |

# 8. 升级与回滚

- **每个小版本都要跟**（17.0 → 17.1 → 17.2），跨大版本必须经过指定的"中间版本"（path），用官方 upgrade path 工具查，否则会报错卡死。
- 升级前先 `gitlab-backup create`，再 `apt install gitlab-ce=17.x.y-ce.0` 指定版本，Omnibus 会自动跑数据库迁移。
- 升级窗口内 GitLab 只读（后台在做 schema migration），选低峰期。

# 9. 常见安装坑

| 现象 | 原因与处理 |
|------|-----------|
| reconfigure 报 `Memory exhausted` | 内存不足，加 swap 或调低 puma workers |
| 页面 502 | Puma/Sidekiq 还在启动，等 2 分钟；持续 502 看 `gitlab-ctl tail puma` |
| git clone 卡在认证 | Docker 安装未转发 22 端口或未配 `gitlab_shell['ssh_port']` |
| 邮件发不出 | SMTP 未配或被云厂商封 25 端口，改用 465/587 TLS |
| 时间显示差 8 小时 | 宿主机时区不对：`gitlab_rails['time_zone'] = 'Asia/Shanghai'` + reconfigure |

# 10. 小结

| 要点 | 一句话 |
|------|--------|
| 选型 | 体验用 Docker，生产用 Omnibus（或 Helm），不要源码安装 |
| 资源 | 4C4G 是底线，内存大头给了 Rails 与 Sidekiq |
| 配置入口 | 一切改 `gitlab.rb` + `reconfigure`，不要手改生成的配置 |
| 安全基线 | 改 root 密码、关公开注册、配 SMTP，三件事装完就做 |
| 升级 | 按官方 path 逐级升，升级前必备份 |

下一篇拆解 GitLab 的内部架构：一次 `git push` 请求进来之后，NGINX、Workhorse、Rails、Gitaly 各自干了什么。
