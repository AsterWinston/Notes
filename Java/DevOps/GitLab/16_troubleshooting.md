# 1. 排错方法论：先分层，再动手

GitLab 故障排查的总纲对应第 3 篇架构图——**先判断请求走的是哪条线，再到对应组件找日志**：

```text
用户报障 ──► 是 Web/API？        → NGINX → Workhorse → Puma(Rails) → PostgreSQL/Redis
          ──► 是 git clone/push？ → NGINX/SSH → Gitaly → 磁盘
          ──► 是流水线不动？      → Runner 注册/tags/队列 → Sidekiq
          ──► 是通知/状态延迟？    → Sidekiq 队列积压
第一步永远是: sudo gitlab-ctl status  +  gitlab-rake gitlab:check
```

## 1.1 三分钟固定信息采集

排错最贵的成本是"信息不全就开始改配置"。下面这段脚本在任意 Omnibus 实例上跑一次，产出足以判断层次的最小证据包：

```bash
#!/usr/bin/env bash
# triage.sh —— 一次性采集排错上下文（只读，不改任何东西）
set -uo pipefail
TS=$(date +%F_%H%M%S); OUT="/tmp/gitlab-triage-$TS"; mkdir -p "$OUT"

sudo gitlab-ctl status                                   | tee "$OUT/status.txt"
sudo gitlab-rake gitlab:env:info                          > "$OUT/env_info.txt" 2>&1
sudo gitlab-rake gitlab:check SANITIZE=true 2>&1 | tail -60 > "$OUT/check.txt"
glab api version                                        | tee "$OUT/version.json"
uptime; free -h; df -h /var/opt/gitlab                 > "$OUT/resources.txt" 2>&1
dmesg -T 2>/dev/null | grep -i -E 'oom|killed process' | tail -20 > "$OUT/oom.txt"
sudo journalctl -u gitlab-runsvdir -p err --since "-30 min" --no-pager > "$OUT/journal_err.txt"
tail -n 200 /var/log/gitlab/nginx/gitlab_error.log        > "$OUT/nginx_err.txt" 2>&1
echo "证据包: $OUT"; tar czf "$OUT.tar.gz" -C "$(dirname "$OUT")" "$(basename "$OUT")"
```

## 1.2 影响面判定：先划边界再查根因

| 现象范围 | 优先怀疑 | 首查命令 |
|---------|---------|---------|
| 全站所有用户 | 入口/共享依赖（PG、Redis、磁盘、证书、DNS） | `gitlab-ctl status`、`df -h`、`gitlab-rake gitlab:check` |
| 单集群某节点 | 该节点进程/Puma/Workhorse、时钟漂移 | 逐节点 `gitlab-ctl tail puma` |
| 只有某项目 | 项目数据/仓库异常、成员与令牌、CI 变量 | `gitlab-rake "gitlab:project_check[<ID>]"` |
| 只有某人 | 账号、2FA、权限、本地凭据与代理 | Admin Area → Users → 对该用户 impersonate 验证 |
| 只有某类操作（如 push） | Gitaly/Runner/网络链路 | `gitlab-ctl tail gitaly` 复现 |

> **纪律**：一次只改一项配置，改完立刻 `gitlab-ctl reconfigure` 并验证；"顺手改三个"会让故障无法归因，也阻断后续同类问题的排查经验沉淀。

# 2. Web 界面类

| 现象 | 高频原因 | 处理 |
|------|---------|------|
| 502 | Puma 未就绪（刚重启/迁移中）或 worker 全忙 | 等 2-3 分钟；`gitlab-ctl tail puma`；看 backlog 指标 |
| 页面极慢 | worker 数 < 并发、PG 慢查询、Sidekiq 抢占 CPU | 调 `puma['worker_processes']`（别超内存预算）、查慢查询日志 |
| 504 在克隆大仓库页面 | Workhorse/nginx 超时 | 增大 `nginx['proxy_read_timeout']`，改用 git CLI |
| 登录循环跳回登录页 | Cookie/HTTPS 配置、Redis 会话丢失、时钟漂移 | 核对 `external_url` 与实际访问域一致；Redis 健康 |
| 500 只在某项目 | 项目数据异常（迁移残留） | `gitlab-rake "gitlab:project_check[<ID>]"` |
| 静态资源 404 | Nginx 配置被手改/资产未预编译 | 回归 gitlab.rb + reconfigure |

## 2.1 502：把"未就绪"和"真挂了"分开

典型报错：浏览器/`curl` 返回 `502 Bad Gateway`，`gitlab_error.log` 里是 `upstream prematurely closed connection while reading response header from upstream`。

```bash
# ① 区分是 Puma 没起来，还是 Workhorse 找不到上游
sudo gitlab-ctl status puma gitlab-workhorse nginx
sudo gitlab-ctl tail puma | tail -50            # 看到 "======== Puma starting ========" 且无 backtrace 才算就绪
grep -c 'upstream prematurely closed' /var/log/gitlab/nginx/gitlab_error.log

# ② reconfigure / 迁移期间的 502 属正常窗口，判断依据是启动进度而不是报错本身
sudo gitlab-ctl tail | grep -iE 'migrat|bootsnap|Worker.*booting'
sudo gitlab-rails runner "puts ActiveRecord::Base.connection.select_value('select max(version) from schema_migrations')"   # 能连库并返回迁移版本 → Rails 层已可用

# ③ 长期 502（不是刚重启）：worker 全忙或内存不足
curl -s localhost:8080/metrics | grep '^puma'          # backlog / workers 等排队指标
curl -s localhost:9168/sidekiq | grep '^sidekiq_queue' | head
dmesg -T | grep -i 'killed process' | tail       # OOM 杀了 puma → 见第 5.4 节
```

处理顺序：**先等（迁移/启动窗口）→ 再单点重启 `gitlab-ctl restart puma` → 然后砍并发换稳定（降 worker 数）→ 最后才考虑扩容**。

## 2.2 页面极慢 / 偶发 504

```bash
# 用结构化日志定位"慢在 DB 还是视图"（application_json.log 字段见第 14 篇）
tail -n 20000 /var/log/gitlab/gitlab-rails/application_json.log \
  | jq -r 'select(.duration_s>2) | [.duration_s,.db_duration_s,.view_duration_s,.controller,.action] | @tsv' \
  | sort -nr | head -10
# 数据库侧确认是否连接耗尽 / 长事务
sudo gitlab-psql -d gitlabhq_production -c "select count(*) , state from pg_stat_activity group by 2;"
sudo gitlab-psql -d gitlabhq_production -c "select pid, extract(epoch from now()-xact_start) tx, left(query,80)
  from pg_stat_activity where xact_start is not null order by tx desc limit 5;"
```

```ruby
# 预算内调 Puma/Sidekiq（每 1 个 puma worker 大约吃 1-2GB，先算内存再调）
puma['worker_processes'] = 4
puma['min_threads'] = 4
puma['per_worker_max_memory_mb'] = 1200   # 超阈值优雅回收，替代早已废弃的 worker killer 手改
sidekiq['concurrency'] = 20
```

504 且只在打开大仓库/长请求时出现：`nginx['proxy_read_timeout']`、`nginx['client_max_body_size']`（与第 3.1 节同源）；Workhorse 侧核对 `gitlab_workhorse['feature_limits']` 是否把请求挡在入口。**根治方向是把长操作交给异步任务，而不是无限拉长超时**。

## 2.3 登录后跳回登录页

三处必查（顺序即概率）：

```bash
# ① external_url 与实际访问域名/协议是否完全一致（含 http/https、端口）
grep -vE '^\s*#' /etc/gitlab/gitlab.rb | grep -E "external_url|_external_url"
sudo gitlab-ctl diff-config        # 对比生成配置与 gitlab.rb 意图，看有没有被手改覆盖
# ② Redis 会话与持久性：重启后掉线、集群脑裂都会表现为"登录循环"
sudo gitlab-ctl tail redis | tail -30
sudo gitlab-rails runner 'Gitlab::Redis::Cache.with { |r| puts r.info("clients") }'
# ③ 时钟漂移导致 Cookie/Token 立刻失效
timedatectl status | sed -n '1,6p'
```

HTTPS 场景额外两点：证书链不完整（浏览器报 ERR_PROTO 但 curl 正常）与 **HSTS**（改回 HTTP 测试时浏览器仍强制跳转——用无痕/其它浏览器验证，避免误判为"配置没生效"）。

## 2.4 413、上传失败与静态资源 404

| 报错原文 | 根因 | 修复 | 验证 |
|---------|------|------|------|
| `413 Request Entity Too Large` | `nginx['client_max_body_size']` 小于上传体积（大 push、大制品、Wiki 图片） | 上调该值 + reconfigure | `curl -I -T 大文件` 或直接重试上传 |
| `HTTP 400 ... too many files`（Web IDE/上传） | Workhorse 单请求文件数/大小限制 | 走 Git LFS 或分批上传；调 `gitlab_workhorse` 相关限制 | 再传一次 |
| 静态资源 404、页面样式错乱 | 手改过生成的 nginx 配置 / `assets:precompile` 未完成 / CDN 指向旧版本 | 回归 `gitlab.rb` 后 `reconfigure`；升级后跑 `gitlab-ctl restart` | 硬刷新 + 看 `/-/version` 与前端资源版本一致 |

> **注意**：所有定制走 `gitlab.rb`，**手改 `/var/opt/gitlab/nginx/gitlab_config` 之类的生成文件必被 reconfigure 覆盖**，这是"越修越坏"的第一来源。需要自定义片段用官方支持的注入点：`nginx['custom_gitlab_server_config']`、`nginx['custom_locations']`。

## 2.5 只在某个项目报 500

```bash
# 拿项目 ID：路径转 ID
ID=$(glab api "projects/payments%2Forder-service" | jq -r .id)
sudo gitlab-rake "gitlab:project_check[$ID]" SANITIZE=true 2>&1 | tail -40
# 常见三分支：仓库目录损坏 / 迁移残留脏数据 / 导入项目分支与标签缺失
sudo gitlab-rails runner "p Project.find($ID).repository.exists?"
# 需要更深的 Git 对象一致性检查时，在 Gitaly 节点用官方 git 工具针对该 storage 路径执行，
# 或直接收集 storage 路径 + 报错时间窗交给 GitLab 支持；不要在应用层手改仓库目录。
# 修完必须验证三件事：项目页、CI 页、一次真实 clone
```

# 3. Git 操作类（clone/push/fetch）

| 现象 | 定位 | 处理 |
|------|------|------|
| push 大包 504/413 | nginx/workhorse 体积限制 | 提高 `nginx['client_max_body_size']`；改用分批 push/对象存储 |
| `The remote end hung up unexpectedly` | Gitaly 超时/内存不足、网络 MTU | 看 gitaly 日志；调并发与超时；大仓库走 SSH |
| SSH 克隆 Permission denied | 密钥未加、Docker 端口未转发、`gitlab_shell['ssh_port']` 不一致 | `ssh -T git@host -p <port>` 验证；核对显示端口 |
| 克隆地址 http/https 不对 | `external_url` 与访问方式不符 | 改 external_url + reconfigure |
| 大仓库操作越来越慢 | `.git` 膨胀（松散对象、LFS 滥用） | 联系管理员触发 **Housekeeping**（Gitaly 自动 gc）；仓库瘦身 |
| Web hook 收不到 push 事件 | Sidekiq 积压 / 目标地址不通（出站限制） | Admin → Hooks 看失败记录，重放 |

## 3.1 大包 push 被掐断

```bash
# ① 先在客户端拿证据（区分是限制还是超时）
GIT_TRACE=1 GIT_CURL_VERBOSE=1 git push 2>&1 | grep -iE '< HTTP|error|hung up' | tail -20
# ② 服务端按请求线看：nginx access 里状态码与体积
tail -n 5000 /var/log/gitlab/nginx/gitlab_access.log | grep -E 'git-upload-pack|git-receive-pack' \
  | awk '{print $9, $10}' | sort | uniq -c | sort -nr | head
```

```ruby
# ③ 限制项集中在这几处（改完 reconfigure，一次只改一类）
nginx['client_max_body_size'] = '512m'
nginx['proxy_read_timeout']   = '600s'
gitlab_workhorse['feature_limits'] = { body_limit: 536_870_912 }   # Workhorse 侧请求体上限
gitaly['configuration'] = {
  # 大仓库首次导入/推全量时最常见的超时项（键名随 Gitaly 版本微调，以 gitlab-ctl 生成的配置为准）
  operations: { git_recv_pack: { timeout: '15m', concurrency: { max_per_storage: 10 } } },
}
```

**结构性解法**（超时只是止痛）：大二进制进 LFS 或制品库、历史瘦身（第 3.5 节）、按模块拆仓库；否则每次故障都会重演。

## 3.2 `The remote end hung up unexpectedly` / Gitaly 侧超时

```bash
# 用相关 ID 把一次 push 在三层日志里串起来
RID=<客户端时间点对应的 request_id 或 correlation_id>
sudo gitlab-ctl tail gitaly | grep -F "$RID"
grep -F "$RID" /var/log/gitlab/gitlab-rails/production.log
# Gitaly 常见结论：命令被并发拒绝、内存/IO 打满、磁盘满、git 进程被 kill
grep -E 'concurrency limit reached|Command aborted|no space left on device' /var/log/gitlab/gitaly/current | tail
```

| 报错关键片段 | 根因 | 修复 |
|------------|------|------|
| `concurrency limit reached` | Gitaly 并发上限（保护性拒绝，非故障） | 提高 `operations.*.concurrency.max_per_storage`，或分流多 Gitaly 节点 |
| `rpc error: ... DeadlineExceeded` | 单次操作超默认超时 | 调 operations timeout；拆小提交 |
| `write error / no space left on device` | 磁盘/inode 满（`.git` 打包需要临时空间） | 清过期制品与 Prometheus 数据、清理实例临时目录、扩容 |
| `Killed` 且 `dmesg` 有 oom | Gitaly 被内核杀 | 限 `gitaly` 内存/按角色拆机（第 5.4 节） |
| 仅 HTTPS 失败、SSH 正常（或相反） | 反向代理/端口转发/MTU 问题 | 分别验证 443 与 22；查中间 LB idle timeout；`ip route` MTU |

## 3.3 SSH 与克隆地址类

```bash
ssh -T -p <port> git@gitlab.example.com          # 期望：Welcome to GitLab, @user!
glab api user/keys | jq -r '.[] | "\(.id)\t\(.title)\t\(.created_at)"'   # 密钥是否真在账号上
# Docker 部署最常漏：端口没映射
docker ps --format '{{.Names}}\t{{.Ports}}' | grep gitlab
```

```ruby
gitlab_shell['ssh_port'] = 22            # 对内监听端口
gitlab_shell['ssh_url']  = 'git@gitlab.example.com'   # 对外展示/克隆串
gitlab_rails['gitlab_ssh_url'] = 'git@gitlab.example.com'
```

**判读**：GitLab 页面显示的克隆地址由 `external_url`（HTTP）与 `gitlab_shell`/`gitlab_rails` 的 ssh 配置（SSH）共同决定。改完必须 `reconfigure`，且**已保存的 remote 不会自动变**——`git remote -v` 与 `git remote set-url` 属于用户侧动作，别在服务端找这个 bug。

## 3.4 仓库越来越慢：Housekeeping 与瘦身

```bash
# ① 官方统计端点（最快拿到分项体积）
glab api "projects/$P/storage" | jq '{repository_size, lfs_objects_size, wiki_size, packages_size}'
# ② 落到磁盘：仓库在 Gitaly 节点的 storage 哈希目录下，先问出路径再看大小
sudo gitlab-rails runner 'p Project.find_by_full_path("payments/order-service").repository.disk_path'
sudo du -sh "/var/opt/gitlab/git-data/repositories/<上一步输出的 disk_path>.git"
```

- 轻度：项目 → Settings → Repository → **Repository housekeeping**（调低触发阈值），由 Gitaly 执行 gc。
- 重度：删除大文件历史（`git filter-repo`/BFG）→ 强推 → **通知管理员清理不可达对象**（`git gc --prune=now` 由实例侧执行，否则磁盘不降）。历史重写会影响所有协作者与 CI 缓存，属变更操作，走 MR/变更单。
- 长期：大文件一律 LFS/制品库，Push Rules 阻断新增超限文件。

## 3.5 Git LFS 专项

| 报错原文 | 根因 | 修复 | 验证 |
|---------|------|------|------|
| `Git LFS: (0 of N files) ... 404` | 项目未开 LFS 存储 / 对象存储路径错 | 项目启用 LFS；检查 `object_store` 的 `lfs` 配置 | `git lfs push --all origin main` |
| `batch response: Authentication required` / `403` | CI 用 Job Token 但无 LFS 读权限、或 Deploy Token scope 不足 | 换用有 `read_repository` 的凭据 | 干净目录 `git lfs clone` |
| `smudge filter lfs failed` | 本地拿不到对象（网络/对象未上传） | `git lfs fetch --all`；找回上传者 | `git lfs fsck` |
| `OpenGitCredentials`/`git-credential` 弹窗（Windows） | 凭据管理器与 PAT 混用 | `git config --global credential.helper manager` 后重新登录 `glab auth login` | 再 push 一次 |

```bash
git lfs env | sed -n '1,12p'      # endpoint 是否指向本项目，是最快的一眼判据
git lfs ls-files -s | head        # 带大小列出，定位"谁把 2GB 塞进了 LFS"
```

# 4. CI/CD 类：Job 不动或动得不对

| 现象 | 排查顺序 |
|------|---------|
| Job 永远 pending | ①Runner 在线吗（Admin → Runners）②tags 匹配吗 ③`run_untagged`/保护分支限制 ④并发满 |
| 流水线压根没创建 | workflow:rules 全不命中；`.gitlab-ci.yml` lint 报错；Auto DevOps 被关且无 yml |
| 该有的 job 消失 | job 级 rules 未命中（rules 默认=不创建）；`changes` 未命中路径 |
| YAML 明明对却 invalid | 缩进 Tab、`include` 路径错、同名 key 被覆盖 |
| job 卡 running 不退出 | Runner 断网但 job 租约未到期；script 真挂起（timeout 兜底） |
| 缓存不命中 | key 设计（跨分支）、Runner 不同机器、cache 路径不在 `$CI_PROJECT_DIR` |
| 制品下载 404 | artifacts 过期策略；对象存储未配 |
| MR 流水线与分支流水线双跑 | 用 workflow:rules 二选一（第 7 篇） |
| 定时任务不跑 | Schedule 分支/变量错；Sidekiq cron 延迟 |

## 4.1 pending 决策树

```bash
# ① job 为什么没被领：看 Runner 侧过滤结果（最常用的一条命令）
glab api "projects/$P/jobs/$JOB_ID" | jq '{name,stage,ref,tag_list,coverage}'
glab api projects/$P/runners | jq -r '.[] | "\(.description)\ttags=\(.tag_list)\tonline=\(.online)\tpaused=\(.paused)"'
# ② Runner 机器上确认它连的是这个实例、且认证有效
sudo gitlab-runner verify 2>&1 | tail
sudo journalctl -u gitlab-runner --since "-10 min" --no-pager | grep -iE 'Checking for jobs...|error|403|409|Reservation'
```

| 判定条件 | 结论 | 动作 |
|---------|------|------|
| 没有任何在线 Runner 带匹配 tag | 调度不到 | 起对应 tag 的 Runner，或去掉 tag |
| Runner 在线但 `paused=true` / `is_locked` | 被暂停或未解锁 | 恢复/解锁（Admin → Runners） |
| job 在受保护分支但需要 Protected variables | 变量缺失导致拒绝 | 变量设 Protected，或分支纳入 protected |
| `run_untagged=false` 且 job 无 tag | 私有 Runner 不接无 tag job | 打 tag 或允许 untagged |
| `limit` / `concurrent` 满 | 容量瓶颈 | 提并发或扩 Runner（第 8 篇容量规划） |
| 环境是 protected environment，执行人无部署权 | 权限门禁 | 加入 Allowed to deploy（第 9 篇） |
| Runner 报 `403 Forbidden ... invalid token` | 令牌类型/过期问题（新的 authentication token 流程） | 重新注册，用 agent/authentication token（第 8 篇） |

## 4.2 流水线没创建 / job 消失

```bash
# 用 lint 端点看"合并后的最终配置"，这是回答"我明明写了却没生成"最快的一招
glab api --method POST "projects/$P/ci/lint" -f "content=$(cat .gitlab-ci.yml)" | jq '{valid, errors, warnings}'
glab api --method POST "projects/$P/ci/lint" -f "content=$(cat .gitlab-ci.yml)" | jq -r '.merged_yaml' | head -60
# 已存在流水线但 job 为空 → 查 workflow 命中情况（Web：流水线页 ▸ "Why did this job not run?"）
glab api "projects/$P/pipelines?per_page=5" | jq -r '.[] | "\(.id)\t\(.ref)\t\(.source)\t\(.status)"'
```

| 现象 | 高频原因 | 修复 |
|------|---------|------|
| push 后完全没流水线 | `workflow:rules` 一条都不命中（最常见：只写了 `merge_request_event`，却直接 push 到分支） | 补 `- if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH` |
| 无 `.gitlab-ci.yml` 也不跑 | Auto DevOps 被关闭且项目未指定 `ci_config_path` | 恢复模板 include（第 7 篇） |
| job 少了一半 | 同名 key 被后面的定义覆盖；`needs` 指向不存在的 job | CI Lint 看 warnings；用唯一 job 名 |
| 只有 `changes` 命中才跑，但一直不跑 | 首次/无历史对比时 `changes` 判空；路径写法不含通配前缀 | 加 `- if: $CI_PIPELINE_SOURCE == "push"` 兜底 |
| MR 与分支流水线双跑 | 未做二选一 | `- if: $CI_OPEN_MERGE_REQUESTS` → `when: never`；MR 事件才建 |

## 4.3 job 卡 running / 不退出

```bash
# ① 判断是"真在跑"还是"Runner 失联但租约未过期"
glab api "projects/$P/jobs/$JOB_ID/trace" | tail -30
glab api "runners/$RUNNER_ID/jobs?per_page=20" | jq -r '.[] | "\(.id)\t\(.status)\t\(.name)"'
# ② 失联表现：日志长时间无新行、Runner 页 online=false。job 会在租约到期（默认 3h）后失败
# ③ 真挂起：给流水线兜底
timeout: 15m
```

处理顺序：先看 trace 最后一行判断阶段 → 网络断（Runner 与实例之间 443 是否通）→ 需要立刻放行时 Cancel 该 job（历史状态会变 `canceled`）→ 事后为长任务开 `interruptible: true` 避免旧流水线继续占 Runner。

## 4.4 缓存不命中与制品 404

| 症状 | 判据 | 原因 | 修复 |
|------|------|------|------|
| 每次都"Created cache"但不"Restored cache" | job 日志开头 | key 与还原 key 不一致 / 分支作用域 | `key:files: [pom.xml]` 或 `key:{prefix}`；避免按分支隔离需求 |
| 分布式 Runner 时好时坏 | 只在某些 Runner 命中 | 本地缓存不共享 | 配 S3/GCS 共享缓存（Runner `config.toml` 的 `[runners.cache]`） |
| 缓存"看起来命中"但依赖没更新 | 构建产物与最新 pom 不符 | 缓存 key 不含依赖文件 | 用 `key:files` 让 pom 变更自动换 key |
| `ERROR: Failed to extract cache / no such file` | 日志 | 缓存目录不在 `$CI_PROJECT_DIR` 相对路径 | cache paths 用项目内相对路径 |
| 制品下载 `404`（几小时后失效） | 打开制品 URL | `artifacts:expire_in` 到期 | 延长有效期或把交付物推到包仓库（第 10 篇） |
| 制品整体不可下载 | 全实例都失败 | 对象存储配置错误/证书/桶权限 | 校验 `artifacts` 对象存储配置；`gitlab-ctl tail registry`/rails 日志 |

```bash
glab api "projects/$P/artifacts" | jq -r '.[] | [.id,.expire_in,.size,.filename] | @tsv' | head
```

## 4.5 Runner 侧命令集

```bash
sudo gitlab-runner verify                              # 认证与连通性
sudo gitlab-runner list 2>&1 | tail                    # 本地注册了哪些
sudo gitlab-runner -v                                  # Runner 与实例版本需匹配（跨大版本容易出问题）
grep -A5 '\[\[runners\]\]' /etc/gitlab-runner/config.toml | head -30
sudo journalctl -u gitlab-runner --since "-15 min" --no-pager | grep -icE 'error'
# 容器类 Executor 的常见二次故障
docker ps -a --format '{{.Names}}\t{{.Status}}' | head           # helper 镜像起不来
docker images | grep -E 'helper|alpine'                          # helper 镜像缺失/被清理
```

# 5. 性能与容量类

| 症状 | 元凶概率排序 | 处方 |
|------|-------------|------|
| Sidekiq 队列持续增长 | 邮件/Webhook 风暴、慢任务占满线程 | 加 Sidekiq 并发或扩实例；Admin → Sidekiq 面板看 slowest |
| 全站偶发超时 | PG 连接耗尽 | `postgresql['shared_buffers']`/max_connections；应用机与库机分离 |
| 磁盘涨疯了 | 制品/LFS/备份未清 | 清理策略（第 10 篇）、配额（第 13 篇）、对象存储卸载 |
| 内存 OOM killer 杀进程 | Puma/Sidekiq/Gitaly 叠加超配 | 按角色拆机；先砍 worker 数量换稳定 |

## 5.1 Sidekiq 积压的定位与止血

```bash
# 哪个队列在积压、延迟多少（队列指标名以实际输出为准）
curl -s localhost:9168/sidekiq | grep -E '^sidekiq_queue_(latency|enqueued|size)' \
  | awk '{print $1, $2}' | sort -k2 -nr | head -15
```

止血三选一（视成因）：调 `sidekiq['concurrency']` 并加 Sidekiq 节点（吞吐型）；把外部依赖的超时收紧（Webhook 目标不通会拖满线程）；对可丢弃任务降级/限流（邮件风暴）。**注意**：Sidekiq 的 `critical/default` 队列混跑会让低优先级任务饿死高优先级——按队列拆进程是标准做法。

## 5.2 PG 连接与索引膨胀

```ruby
# 上 PgBouncer（transact 模式），应用侧不再直连 400 个连接
pgbouncer['enable'] = true
pgbouncer['default_pool_size'] = 20
pgbouncer['max_client_connections'] = 400
gitlab_rails['db_adapter'] = 'postgresql'
gitlab_rails['db_host'] = '/var/run/postgresql/'
```

```bash
# 膨胀与不可达索引（只读查询，升级窗口前必查）
sudo gitlab-psql -d gitlabhq_production -c "
  select relname, n_dead_tup, last_vacuum, last_autovacuum from pg_stat_user_tables
  order by n_dead_tup desc limit 10;"
```

升级前的两个硬检查：所需扩展是否齐（`pg_stat_statements` 等）、索引是否 `INVALID`——两者任一不满足，迁移会中途失败并触发长时间 502。

## 5.3 OOM 与磁盘

```bash
dmesg -T | grep -i -E 'out of memory|killed process' | tail -10
sudo grep -h 'Killed process' /var/log/* 2>/dev/null | tail
# 谁吃满了内存：Puma worker×数、Sidekiq 并发、Gitaly 打包任务三者最可能叠加
free -h; sudo gitlab-ctl status | awk '{print $1}' | sort | head -20
# 磁盘定位到目录级：/var/opt/gitlab 下这几处最大
sudo du -xh --max-depth=1 /var/opt/gitlab 2>/dev/null | sort -h | tail -8
sudo du -xh --max-depth=2 /var/opt/gitlab/git-data 2>/dev/null | sort -h | tail -5
```

内存预算公式（自建实例通用）：`所需 ≈ puma_workers×1.5GB + sidekiq_concurrency×0.5GB + Gitaly/PG/Redis 预留`。**宁可少 worker 稳定，也不要靠 swap 撑住**——swap 会让 P99 抖成故障。

# 6. 邮件与通知

- 发不出：`gitlab-rails console` 里 `Notify.test_email('you@x.com','t','b').deliver_now` 直接测 SMTP。
- 25 端口被云厂商封 → 465/587；`gitlab.rb` smtp 改完必须 reconfigure。
- 收到了但延迟：Sidekiq mailer 队列积压，回到本篇第 5 节处理。

## 6.1 SMTP 端到端验证

```bash
# ① 配置是否被真正加载（不要凭印象读 gitlab.rb）
sudo gitlab-rails runner 'p ActionMailer::Base.smtp_settings'
# ② 实测发信（异常会直接抛，日志里能看到 SMTP 协议对话）
sudo gitlab-rails console
> Notify.test_email('someone@example.com', 'GitLab SMTP test', 'body').deliver_now
```

```ruby
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = 'smtp.qiyou.cn'
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = 'ci@example.com'
gitlab_rails['smtp_password'] = 'AppPasswordHere'
gitlab_rails['smtp_domain'] = 'example.com'
gitlab_rails['smtp_authentication'] = 'login'
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = 'ci@example.com'
user['gitlab_email_from'] = 'ci@example.com'
```

| 报错原文 | 根因 | 处理 |
|---------|------|------|
| `Connection refused / timeout :25` | 云厂商默认封 25 | 改 465（`smtp_tls=true`）或 587（STARTTLS），并申请解封 |
| `Net::SMTPAuthenticationError` | 用了网页密码/未开应用专用密码 | 用 SMTP 授权码；核对 `smtp_authentication` |
| 邮件全进垃圾箱 | SPF/DKIM/DMARC 缺失、发信域与实例域不符 | 配 DNS TXT；`gitlab_email_from` 与实例域一致 |
| `550 5.7.1 Relaying denied` | 未认证或 IP 未白名单 | 加认证/申请中继白名单 |
| 收不到 MR 通知但登录邮件正常 | 用户邮件通知级别为 Disabled / 项目被静音 | 用户 Settings → Notifications；项目页 bell 检查 |

## 6.2 通知延迟与去重

`action_mailer` 队列积压 → 与第 5.1 节同治。团队侧更常见的真问题是**噪声导致静音**：用群组/项目通知级别 + Issue 标签订阅（`Settings → Notifications → Custom`）替代"全部关闭"，比事后调 SMTP 更有价值。

# 7. 日志速查表（配合第 14 篇）

| 场景 | 一条命令 |
|------|---------|
| 全组件状态 | `sudo gitlab-ctl status` |
| 实时看错误 | `sudo gitlab-ctl tail`（全量）/ `tail puma`/`tail gitaly` |
| 结构化错误过滤 | `sudo journalctl -u gitlab-runsvdir -p err` |
| 数据库慢查询 | `sudo gitlab-ctl tail postgresql` + `postgresql['log_min_duration_statement']` |
| API 审计 | `/var/log/gitlab/gitlab-rails/audit_json.log`（含 token 使用者） |
| 排错前自检 | `sudo gitlab-rake gitlab:check SANITIZE=true` |
| 版本与许可 | `glab api version` / `sudo gitlab-rails runner 'puts Gitlab::CurrentSettings.current_application_versions'` |

## 7.1 用相关 ID 串起三层日志

```bash
# 一次 Web/API 请求：request_id（入口日志里给）→ Rails 日志 → 异常栈
grep -F "$RID" /var/log/gitlab/nginx/gitlab_access.log
grep -hF "$RID" /var/log/gitlab/gitlab-rails/api_json.log
# 一次 Git 操作：correlation_id 由 Workhorse 生成并向下游传递，最可靠
grep -F "$CID" /var/log/gitlab/gitlab-workhorse/current /var/log/gitlab/gitaly/current
# 一次流水线 job：job 日志末尾的 trace 与 Rails 的 ci 相关日志时间对齐即可定位归属
glab api "projects/$P/jobs/$JOB_ID/trace" | tail -5
```

`correlation_id` 是**跨组件唯一线索**，比按时间猜要可靠得多。建议在网关/拨测里把 `request_id` 返回给客户端（GitLab 的 API 响应含相关头），用户报障时直接要这个串，可省掉一半排查。

## 7.2 复现与止血纪律

1. **止血优先于根因**：可回滚的先回滚（配置、发布），恢复服务再查。
2. **一次一个变量**：改配置 → `reconfigure` → 立刻验证；记录"改了什么、期望看到什么"。
3. **留下证据**：本篇一开头的 triage 输出 + 相关 ID + 时间窗，是提工单/内部复盘的最小信息集。
4. **同类故障要转化**：每处理一次，就把"判据 + 一条命令"补进第 14 篇指标或本篇表格——排错手册的价值随事故数增长。

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 分层定位 | Web 线 / Git 线 / CI 线 / 异步线，四线日志各管一段 |
| 三板斧 | `gitlab-ctl status` → 对应组件 tail → `gitlab:check` |
| CI 排错 | pending 查 Runner/tags，缺 job 查 rules，用 lint 端点看展开结果 |
| 配置纪律 | 只改 gitlab.rb + reconfigure，不碰生成物 |
| 容量问题 | 指标先行（队列/连接/磁盘），备份兜底 |
| 证据链 | request_id / correlation_id 串三层日志，止血优先、一次一变量 |

**故障发生后的前 5 分钟（贴值班手册）**

```bash
sudo gitlab-ctl status                                   # ① 有没有进程挂了
df -h /var/opt/gitlab; free -h                           # ② 磁盘/内存是否见底
dmesg -T | grep -i -E 'oom|killed process' | tail -5    # ③ 有没有被内核杀
curl -s -o /dev/null -w '%{http_code}\n' https://gitlab.example.com/-/readiness   # ④ 依赖是否就绪
sudo gitlab-ctl tail puma gitlab-workhorse nginx | head -60   # ⑤ 入口这条线现在在报什么
```

**排错手册自检清单**

- [ ] 四条线（Web / Git / CI / 异步）各自的日志入口命令写进了团队值班文档
- [ ] `triage.sh` 可一键执行，输出证据包路径，值班新人也会用
- [ ] 每次事故后新增一条"报错原文 → 根因 → 修复 → 验证"表格行，而不是只写复盘结论
- [ ] 所有配置改动都能追溯到 `gitlab.rb`，`/var/opt/gitlab` 下的生成物无人手改
- [ ] 高危动作（重启、清理、历史重写、回滚）都有二次确认与回退路径

至此使用者视角的模块齐了。最后一篇按仓库惯例做面试问答汇总。
