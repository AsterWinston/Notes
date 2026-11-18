# 1. GitLab 的可观测性分两层

谈"GitLab 监控"要先分清对象，否则容易在错误的仪表盘里找答案：

| 层 | 监控什么 | 工具入口 |
|----|---------|---------|
| **平台自身健康** | GitLab 实例：Puma/Sidekiq/Gitaly/PG/Redis 的指标与日志 | Omnibus 内置 Prometheus、各 exporter 端口、`/-/metrics`、`gitlab-ctl tail` |
| **CI/CD 效能与交付质量** | 流水线成功率、时长、MR 周期、部署频率 | CI/CD Analytics、Value Stream、DORA 报告（Premium/Ultimate） |

对使用者（非专职运维），第二层才是每天要看的；第一层主要服务于排错（第 16 篇展开日志）。

## 1.1 谁该看哪一块

| 角色 | 每天必看 | 每周复盘 | 触发式排查 |
|------|---------|---------|-----------|
| 开发 | 自己项目的 MR 流水线、失败用例报告 | CI 时长趋势 | pending 变久、缓存不命中 |
| TL | 团队群组 MR 周期、评审排队 | 失败率 Top job、DORA 趋势 | 主干合并冲突频发 |
| SRE / 平台 | 队列延迟、Puma backlog、磁盘 | 容量趋势、备份成功率 | 502/504、push 超时 |
| 安全/合规 | 审计日志、Secret Detection 结果 | 令牌盘点 | 密钥泄露事件 |

**判层口诀**：用户报"GitLab 慢/报错"→ 第一层；用户报"我的流水线慢/没跑"→ 先看第二层的队列与 Runner，再回第一层看 Sidekiq。

# 2. 平台自身：内置监控栈

Omnibus 打包了 Prometheus 及各组件 exporter，**默认开启且只监听本机**：

```bash
# Prometheus 以 gitlab-prometheus 用户运行，默认 http://localhost:9090
sudo gitlab-ctl status prometheus
curl -s localhost:9090/api/v1/targets | jq -r '.data.activeTargets[] | "\(.labels.job)\t\(.health)\t\(.scrapeUrl)"'
# 单个 exporter 是否真有数据（排错第一步：区分"指标没产生"还是"没被抓取"）
curl -s localhost:9168/sidekiq | head
sudo gitlab-ctl tail prometheus
```

> **安全边界**：Prometheus 与各 exporter **不做用户认证**，谁能连上谁就能读。默认只监听 localhost 正是为此；把它们暴露到网络前，必须先确认网络隔离与防火墙（官方明确不建议直接对外）。

## 2.1 内置栈的开关与收敛

```ruby
# /etc/gitlab/gitlab.rb —— 不需要内置 Prometheus 时，一次关干净
prometheus_monitoring['enable'] = false   # 关掉 Prometheus 及未来新增的所有 exporter
sidekiq['metrics_enabled'] = false        # Sidekiq 指标
puma['exporter_enabled'] = false          # Puma 指标（默认已是 false）
# 保留内置 Prometheus 但收紧监听
prometheus['listen_address'] = 'localhost:9090'
```

```bash
sudo gitlab-ctl reconfigure
sudo gitlab-ctl status | grep -E 'prometheus|exporter'   # 验证已按预期启停
```

内置 Prometheus 的数据目录会自己长大，配保留期而不是等磁盘满：

```ruby
prometheus['flags'] = {
  'storage.tsdb.path'         => '/var/opt/gitlab/prometheus/data',
  'storage.tsdb.retention.time' => '7d',   # 默认 15d，单机监控 7d 足够
  'storage.tsdb.retention.size' => '2GB',  # 实验参数，按磁盘预算设
  'config.file'               => '/var/opt/gitlab/prometheus/prometheus.yml',
}
```

```bash
# 磁盘已经涨疯了的正确处理顺序：先停再删（运行中删 TSDB 数据可能损坏）
sudo gitlab-ctl stop prometheus
sudo rm -rf /var/opt/gitlab/prometheus/data/*
sudo gitlab-ctl start prometheus && sudo gitlab-ctl status prometheus
```

## 2.2 交给企业统一 Prometheus（多节点/已有监控平台的常规做法）

关闭内置 Prometheus，把各 exporter 监听地址放开到内网，并给抓取端做白名单：

```ruby
# /etc/gitlab/gitlab.rb
prometheus['enable'] = false                       # 用外部 Prometheus 抓取
node_exporter['listen_address']      = '0.0.0.0:9100'
gitlab_workhorse['prometheus_listen_addr'] = '0.0.0.0:9229'
gitlab_exporter['listen_address']    = '0.0.0.0'   # 9168：/sidekiq 与 /database 两条路径
gitlab_exporter['listen_port']       = '9168'
sidekiq['listen_address']            = '0.0.0.0'   # 8082
redis_exporter['listen_address']     = '0.0.0.0:9121'
postgres_exporter['listen_address']  = '0.0.0.0:9187'
pgbouncer_exporter['listen_address'] = '0.0.0.0:9188'
registry['debug_addr']               = '0.0.0.0:5001'
gitaly['configuration'] = { prometheus_listen_addr: '0.0.0.0:9236' }

# Rails 的 /-/metrics 与业务流量同端口，必须靠 IP 白名单放行
gitlab_rails['monitoring_whitelist'] = ['127.0.0.0/8', '192.168.0.1']
# NGINX stub_status（8060）同样要 allow/deny
nginx['status']['options'] = { 'server_tokens' => 'off', 'access_log' => 'off',
                               'allow' => '192.168.0.1', 'deny' => 'all' }
gitlab_rails['prometheus_address'] = '192.168.0.1:9090'
```

抓取端配置（官方样例，端口与作业名一一对应上面）：

```yaml
scrape_configs:
  - { job_name: nginx,            static_configs: [ { targets: ['1.1.1.1:8060'] } ] }
  - { job_name: redis,            static_configs: [ { targets: ['1.1.1.1:9121'] } ] }
  - { job_name: postgres,         static_configs: [ { targets: ['1.1.1.1:9187'] } ] }
  - { job_name: node,             static_configs: [ { targets: ['1.1.1.1:9100'] } ] }
  - { job_name: gitlab-workhorse, static_configs: [ { targets: ['1.1.1.1:9229'] } ] }
  - job_name: gitlab-rails
    metrics_path: "/-/metrics"
    scheme: https
    static_configs: [ { targets: ['1.1.1.1'] } ]
  - { job_name: gitlab-sidekiq,            static_configs: [ { targets: ['1.1.1.1:8082'] } ] }
  - { job_name: gitlab_exporter_sidekiq, metrics_path: /sidekiq,  static_configs: [ { targets: ['1.1.1.1:9168'] } ] }
  - { job_name: gitlab_exporter_database, metrics_path: /database, static_configs: [ { targets: ['1.1.1.1:9168'] } ] }
  - { job_name: gitaly,           static_configs: [ { targets: ['1.1.1.1:9236'] } ] }
  - { job_name: registry,         static_configs: [ { targets: ['1.1.1.1:5001'] } ] }
```

**多节点架构**另有标准形态：单独起一台 monitoring 节点（`roles ['monitoring_role']` + Consul 服务发现）。注意坑：`consul['monitoring_service_discovery'] = true` 与 `prometheus['scrape_configs']` **同时设置会报错**，二选一。

K8s（Helm）部署则不用 Omnibus 这套：各组件 Chart 自带 `serviceMonitor.enabled`，接团队 Prometheus Operator。

## 2.3 指标速查与告警阈值起点

| 导出器/指标族 | 关注点 | 起点阈值（自建实例经验值） | 恶化时的动作 |
|------------|--------|--------------------------|-------------|
| `puma_backlog` / `puma_workers` | Web 请求排队 | backlog 持续 > 0 且 5 分钟不降 | 加 worker/加机器；查慢接口（第 16 篇） |
| `sidekiq_queue_*`（延迟、积压） | 异步任务积压 | 关键队列延迟 > 60s | 提 `sidekiq['concurrency']`、拆实例 |
| `sidekiq_jobs_*`（执行/失败） | 任务异常与重试风暴 | 失败率突增 | 看 `sidekiq` 队列页找 slowest |
| `gitaly_*`（命令耗时、并发拒绝） | Git 操作与磁盘瓶颈 | `git-upload-pack` P99 > 5s | 分离 Gitaly、升磁盘、调 concurrency |
| `gitlab_transaction_duration_seconds_*` | Rails 接口耗时/QPS | 首页/项目页 P99 > 3s | 查 N+1、慢查询 |
| `pg_stat_*`（内置 postgres exporter） | 慢查询、连接数、复制延迟 | 连接数 > max_connections×0.8 | 上 PgBouncer / 扩库 |
| `redis_*` | 内存、命中率、队列长度 | 内存 > maxmemory×0.9 | 清理、扩容、检查会话缓存 |
| Workhorse / NGINX 指标 | 入口 5xx、上传大小、stub 活跃连接 | 5xx 占比 > 0.5% | 按第 16 篇 502 链路排查 |

官方给的几条起步 PromQL（可直接做面板）：

```promql
# CPU 使用率
1 - avg without (mode,cpu) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
# 内存可用百分比
((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) or
 ((node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) / node_memory_MemTotal_bytes)) * 100
# 磁盘读写 IOPS
sum by (instance) (rate(node_disk_reads_completed_total[1m]))
sum by (instance) (rate(node_disk_writes_completed_total[1m]))
# Rails RPS（剔除健康检查噪声）
sum(irate(gitlab_transaction_duration_seconds_count{controller!~'HealthController|MetricsController'}[1m])) by (controller, action)
```

**指标名不要靠记忆**：不同版本会新增/改名，用一条命令拿权威清单——

```bash
curl -s localhost:9168/sidekiq | grep '^# HELP' | awk '{print $3}' | sort | head -30
curl -s localhost:9236/metrics | grep '^# HELP gitaly' | head -20
```

## 2.4 面板层：Grafana

内置 Grafana（历史上的 `grafana['enable']`）已不是新版官方推荐的看板路径，现在文档的做法是**外部 Grafana 指向（内置或外部）Prometheus 数据源**：新建数据源选 `Prometheus`，URL 填实例的 Prometheus 监听地址，HTTP method 用 `GET`。好处是面板版本、告警链路与 GitLab 发版节奏解耦。

值得建的最小面板集（四张图即可覆盖 80% 的日常告警）：Puma backlog 与 worker 利用率、Sidekiq 各队列延迟、Gitaly 命令 P99 与并发拒绝、PG 连接数与复制延迟 + 主机磁盘水位。

## 2.5 健康检查端点（拨测/负载均衡用）

```bash
curl -s https://gitlab.example.com/-/liveness    # 进程是否能响应请求
curl -s https://gitlab.example.com/-/readiness   # 依赖（PG/Redis/Gitaly）是否就绪
# 这些端点与用户流量同端口，生产实例要么限内网，要么配 IP 白名单
```

`ip_allowlist` 与 `gitlab_rails['monitoring_whitelist']` 是同一套治理思路：**监控端点默认只放行监控网段**。

# 3. 日志体系：四类日志各司其职

| 日志线 | Omnibus 位置 | 什么时候看 |
|------|-------------|-----------|
| 应用（Rails） | `/var/log/gitlab/gitlab-rails/`：`production.log`、`api_json.log`、`application_json.log`、`json.error.log`、`audit_json.log` | API/页面行为异常、权限问题 |
| 异步任务 | `gitlab-ctl tail sidekiq`（或 `/var/log/gitlab/sidekiq/`）+ Admin 的 Sidekiq 队列页 | 通知/Webhook/状态传播延迟 |
| Git 操作 | `gitlab-ctl tail gitaly`、`/var/log/gitlab/gitaly/current`；SSH 侧 `gitlab-shell/current` | clone/push 失败与慢 |
| 入口 | `/var/log/gitlab/nginx/gitlab_access.log`、`gitlab_error.log`；`gitlab-ctl tail gitlab-workhorse`、`tail puma` | 502/413/上传失败 |

## 3.1 按排障路线取日志

```bash
# 一条请求线：入口 → 应用
sudo gitlab-ctl tail gitlab-workhorse
sudo gitlab-ctl tail puma
tail -F /var/log/gitlab/nginx/gitlab_access.log | grep -E ' (502|504|413) '

# 一次慢接口：从 access log 找 request_id，再顺着 request_id 串所有组件
grep -m5 ' 500 ' /var/log/gitlab/nginx/gitlab_access.log | tail -1
RID=<把上面日志里的 request_id 贴进来>
grep -h "$RID" /var/log/gitlab/gitlab-rails/*.log | tail -50

# 一条 Git 线：按项目路径与命令名过滤（Gitaly 默认 JSON，字段名可直接 jq）
sudo gitlab-ctl tail gitaly | grep -F 'payments/order-service'
tail -F /var/log/gitlab/gitaly/current | grep --line-buffered '"cmd":"git-receive-pack"'
# 一条异步线：某个 MR 的状态没传播，通常是队列延迟
sudo gitlab-ctl tail sidekiq | grep -E 'MergeService|WebHook'
```

## 3.2 结构化日志与字段

Rails 侧的 `*_json.log` 一直是结构化输出，关键字段足以做全链路串联：`time`、`severity`、`duration_s`、`path`、`method`、`format`、`params`、`controller`、`action`、`db_count`/`db_duration_s`、`view_duration_s`、`current_user_id`、`project_id`、`remote_ip`、`correlation_id`、`request_id`。

```bash
# 找最近最慢的 20 个请求，并看它是 DB 慢还是视图慢
tail -n 20000 /var/log/gitlab/gitlab-rails/application_json.log \
  | jq -r 'select(.duration_s != null) | [.duration_s,.db_duration_s,.view_duration_s,.method,.path,.controller] | @tsv' \
  | sort -nr | head -20

# 某用户在实例上干了什么（审计/取证）
jq -r 'select(.username=="alice") | [.time,.path,.method] | @tsv' /var/log/gitlab/gitlab-rails/api_json.log | tail -20
```

其他组件也可切 JSON，便于统一投递：`puma['log_format'] = 'json'`、`sidekiq['log_format'] = 'json'`、`gitlab_shell['log_format'] = 'json'`、`registry['log_formatter'] = 'json'`，Gitaly 在 `gitaly['configuration'].logging.format = 'json'`。

## 3.3 轮转与集中采集

```ruby
# 全局 runsv 日志轮转（Omnibus 统一控制）
logging['logrotate_frequency'] = 'daily'
logging['logrotate_size']      = '50M'    # 超过即切
logging['logrotate_rotate']    = 14       # 保留份数
logging['logrotate_compress']  = 'compress'
# NGINX 与 Registry 有各自的 logrotate 属性（nginx['logrotate_*']），量大时要单独收紧
```

投递到集中日志平台：单机用 Filebeat/Fluent Bit 直接吃 `/var/log/gitlab/**/*_json.log`；日志量大或要做长期留存时按 Elasticsearch 侧的索引与保留策略配 ILM。**先决定留存与检索成本，再决定采集哪些文件**——`production.log` 全量入湖通常是最大的浪费，只入 `*_json.log` + error 级即可。

## 3.4 慢查询与数据库日志

```ruby
# 打开 PG 慢日志阈值（毫秒）；Omnibus 内置 PG 的日志在 /var/log/gitlab/postgresql/ 下
postgresql['log_min_duration_statement'] = '500'
postgresql['log_slow_statement_duration_seconds'] = 5   # GitLab 自带的慢语句采样
```

```bash
sudo gitlab-ctl reconfigure
sudo gitlab-ctl tail postgresql | grep -i 'duration'
# 数据库直连（只读排查用）：psql 与 GitLab 的 rails db 视图
sudo gitlab-rails dbconsole
sudo gitlab-psql -d gitlabhq_production -c "
  select pid, state, wait_event_type, extract(epoch from now()-query_start) as dur, left(query,60)
  from pg_stat_activity where state <> 'idle' order by dur desc limit 10;"
```

排错口诀：**先分"哪条线"——Web 线看 rails+sidekiq，Git 线看 gitaly/shell，入口线看 nginx/workhorse**；再拿 `gitlab-rake gitlab:check` 做全局体检（第 2 篇命令表）。

# 4. CI/CD 分析：交付流水线本身的观测

项目/群组 → **Analyze → CI/CD Analytics**：

- **Pipeline/Job 时长趋势**：哪个 job 在拖慢全团队（优化 `rules`/cache/needs 的依据，第 7 篇）。
- **失败率**：按分支/job 统计；持续失败的 job 要么修要么降级为 `allow_failure` 后另行跟踪。
- **Runner 利用率与队列等待**：pending 时间占比高 → 扩 Runner（第 8 篇）。

## 4.1 用 API 把趋势变成周报

```bash
P='payments%2Forder-service'
glab api "projects/$P/pipelines?per_page=50&updated_after=2026-11-01T00:00:00Z" | jq -r '
  .[] | [.id, .ref, .status, ((.finished_at // "9e9"|fromdateiso8601) - (.created_at|fromdateiso8601))] | @tsv' \
  | awk '{s+=$4; if($4>m)m=$4; n++} END {printf "avg=%.0fs max=%.0fs pipelines=%d\n", s/n, m, n}'

# 各 job 的排队耗时（created → started）：判断"该扩 Runner 还是该拆并行"
glab api "projects/$P/jobs?per_page=100&scope[]=pending&scope[]=running&scope[]=success" | jq -r '
  .[] | select(.started_at) | [.name, ((.started_at|fromdateiso8601) - (.created_at|fromdateiso8601))] | @tsv' \
  | sort -k2 -nr | head -10
```

## 4.2 三个易误读的信号

| 报表现象 | 常见误判 | 正确读法 |
|---------|---------|---------|
| 流水线成功率 99% | 质量很好 | 检查有多少 job 被 `allow_failure: true`，假绿会掩盖问题 |
| 平均时长下降 | 变快了 | 可能只是失败 job 提前退出；要同时看 P90 |
| pending 时间增加 | Runner 不够 | 也可能是 `interruptible` 未开导致旧流水线占位，或 tag 不匹配卡在"找不到 Runner" |

# 5. Value Stream 与 DORA：效能度量（Premium/Ultimate）

```text
Value Stream（编码→部署 四阶段看板）
├── Plan → Code → Test → Deploy → Release → Respond（Ultimate 全链路）
│
DORA 四指标（GitLab 从 MR 与部署事件自动计算）
├── 部署频率 Deployment Frequency
├── 变更前置时间 Lead Time for Changes   ← 从 MR 首次提交到部署生产
├── 变更失败率 Change Failure Rate        ← 部署后多快需要 hotfix/rollback
└── 服务恢复时间 MTTR（结合 Incident 管理）
```

- 前提：`environment` 配好（第 9 篇）、部署 job 标注环境——**DORA 的数据源就是你的流水线元数据**，没声明 environment 等于没埋点。
- **MR 周期分析**：拆解"排队等评审 → 评审 → 修复 → 合并"四段，瓶颈通常第一段（没人批）。

## 5.1 埋点检查清单

| 检查项 | 做法 | 不做的后果 |
|--------|------|-----------|
| 生产部署有 `environment: production` | 部署 job 声明 `name` + `url` | 部署频率为 0，DORA 报告空白 |
| 生产只由受控 ref 触发 | Tag + protected branch + manual | 变更失败率把非生产部署算进来，失真 |
| 回滚/hotfix 可识别 | 用独立 MR + `environment` 记录 | 无法计算 MTTR 与失败率 |
| MR 首次提交时间可靠 | 别用长命分支 squash 掉历史 | Lead Time 偏小，自欺欺人 |
| Incident 关联部署 | `/link` + incident label | Respond 阶段无数据 |

## 5.2 拉数与呈现

```bash
# DORA 指标（群组/项目级，时间窗参数）
glab api "groups/payments/dora/metrics?start_date=2026-10-01&end_date=2026-10-31&interval=weekly" | jq
# Value Stream：合并 MR 的四段时间分布
glab api "groups/payments/merge_requests?state=merged&per_page=100&updated_after=2026-10-01T00:00:00Z" \
  | jq -r '.[] | [.iid, (.created_at|fromdateiso8601), (.merged_at|fromdateiso8601)] | @tsv' \
  | awk '{printf "!%s\t等待评审+修复=%ds\n", $1, $3-$2}' | head -20
# 部署历史（判断部署频率与最近一次生产发布）
glab api "projects/$P/deployments?per_page=50&updated_after=2026-10-01T00:00:00Z" | jq -r '.[].environment'
```

> **行业实践**：DORA/Value Stream 用于**团队自改进**，别用于个人绩效——指标一旦与考核绑定，团队会立刻学会刷低 MR 粒度、合并前造假绿。度量看趋势不看单点。

# 6. 告警与 Incident（Ultimate 偏多）

- **Alert Management**：接收外部监控告警（Prometheus Alertmanager/Webhook 格式），在 GitLab 内分诊、关联 Issue。
- **Incident 时间线**：告警 → 建 Incident Issue → 记录处置评论 → 复盘（Postmortem）；与 MR/部署时间线互相链接——"这次故障对应哪次发布"自动可查。
- 运维侧兜底：把 `/-/readiness`、`/-/liveness` 健康端点接进你们的拨测/负载均衡。

## 6.1 接 Alertmanager

项目 → Deploy → Alert Management → Alert integrations → 新增 **Prometheus** 集成，把它的 webhook URL 填到 Alertmanager 的 `webhook_configs`：

```yaml
# alertmanager.yml（片段）
receivers:
  - name: gitlab
    webhook_configs:
      - url: 'https://gl-alerthook:TOKEN@gitlab.example.com/api/v4/projects/123/alert_management/prometheus_http?env_name=production'
        send_resolved: true
route:
  receiver: gitlab
  matchers: [ 'severity=~"critical|major"' ]
```

GitLab 侧读取的字段：`title`、`description`、`severity`、`monitoring_tool`、`service`、`scoped_at`，以及关键的 **`env_name`（URL 参数）/ `gitlab_environment_name`（annotation）**——对齐环境名，告警才会挂在正确的 Environment 页上。集成支持 **secret token 签名校验**（Alertmanager 侧加 `http_config.authorization`，GitLab 侧配 secret token，校验 `X-Gitlab-Signature`）。

```bash
# 自定义 webhook 集成：手工模拟一条告警，验证连通与字段映射
curl -sS --request POST 'https://gitlab.example.com/api/v4/projects/123/alert_management/webhook' \
  -H 'Content-Type: application/json' \
  -d '{"title":"磁盘水位","description":"/data 使用率 92%","severity":"critical",
       "monitoring_tool":"prometheus","service":"gitaly","env_name":"production"}' | jq
```

## 6.2 Incident 流程与自动回滚

```text
告警进入 → 在告警页点"创建 Incident"（或在 Issue 里用快捷操作 /incident <标题>）
   ├── 自动打 incident 标签 + 建立时间线
   ├── 处置过程全部写在 Issue 评论（便于事后核对时间）
   ├── 与 MR/部署通过 /link 双向关联 → "这次故障对应哪次发布"可查
   └── 恢复后 /status resolved，最后写 Postmortem 并关单
```

Ultimate 的 **Auto Rollback** 能在收到 critical 告警时自动重新部署该环境最近一次成功的部署（Settings → CI/CD → Automatic deployment rollbacks 开启）。要知道它的边界：告警必须带 `gitlab_environment_name`；若当时正在部署则跳过回滚；**三分钟内只回滚一次**（多条告警只处理一条）。

## 6.3 别把 GitLab 当监控主平台

GitLab 的 Alert/Incident 解决的是"**告警与发布历史的关联**"，指标存储、告警规则管理、值班排班仍应留在团队统一监控/值班系统里。反过来说：**值班人员打开 GitLab 时应能在 30 秒内定位"最近一次生产部署是哪个 MR、哪条流水线"**，这才是这块能力的正确用法。

# 7. 用量与成本观测

Admin Area → **Usage Quotas**：存储（仓库/LFS/制品/附件/CI artifacts 分项）、CI 分钟数、活跃用户。群组/项目页也有各自 usage。治理动作（清制品、调保留策略、禁 LFS 大文件）全部由这些数字驱动。

## 7.1 分项取数

```bash
# 项目级统计（仓库/LFS/制品/CI 制品/附件/总计），按总计排序找大户
glab api "groups/payments/projects?per_page=100&statistics=true&order_by=last_activity_at" | jq -r '
  .[] | select(.statistics != null) | [(.statistics.repository_size/1048576|floor),
    (.statistics.lfs_objects_size/1048576|floor), (.statistics.builds_size/1048576|floor),
    (.statistics.packages_size/1048576|floor), .path_with_namespace] | @tsv' | sort -nr | head -15

# 单个大项目的制品明细（决定清哪些 expire_in）
glab api "projects/$P/artifacts" | jq -r '.[] | [.expire_in, .size, .filename] | @tsv' | head   # 该端点自 16.5 起提供

# CI 分钟消耗（自建 Runner 时看 job 时长与频次，SaaS 才看 quota）
glab api projects/$P/runner_jobs?status=success&per_page=100 | jq 'length'
```

## 7.2 配额驱动的治理动作

| 数字信号 | 治理动作 | 落地位置 |
|---------|---------|---------|
| 制品（builds_size）增长快 | 缩短 `artifacts:expire_in`、报告类单独 1 月 | `.gitlab-ci.yml`（第 6 篇） |
| Registry 占用高 | 保留策略 + 发布 tag 豁免 | Settings → General → Registry cleanup（第 10 篇） |
| LFS 大户 | 大文件外置到对象存储/制品库，禁新增 LFS | Push Rules + 群组策略（第 4 篇） |
| 仓库体积超阈值 | 触发仓库瘦身、限制单次提交大小 | 仓库设置 + Housekeeping |
| CI 分钟暴涨 | 查重复流水线（workflow 规则）、开 `interruptible` | 第 7 篇 |

项目/群组配额可用 API 直接下发：

```bash
glab api --method PATCH "projects/$P" -f "max_artifacts_size=500" -f "repository_size_limit=1048576000"
glab api --method PATCH "groups/payments" -f "storage_quota=107374182400"
```

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 两层监控 | 平台健康（Prom 指标+日志）与交付效能（Analytics/Value Stream） |
| 内置栈边界 | Prometheus 默认开且只监听本机、无认证；对外暴露前先加白名单 |
| 排错地图 | Web/Sidekiq/Gitaly/入口四条日志线各管一段，`request_id` 串联 |
| CI 观测 | 时长、失败率、Runner 队列是 CI 优化的三个入口，注意假绿与 P90 |
| DORA | 数据来自 environment 与流水线元数据，配好才有报告 |
| 告警 | Alert Management + Incident 把故障与发布历史串起来，环境名对齐才可用 |
| 成本 | Usage Quotas 的分项数字驱动清理与配额，而不是凭感觉扩容 |

**监控落地自检清单**

- [ ] 明确内置 Prometheus 是"用"还是"关"，二者都别半开（既暴露又没白名单是最差状态）
- [ ] 四条日志线各有一个可复制的取日志命令，团队文档里能查到
- [ ] `storage.tsdb.retention.*` 与 logrotate 保留份数已按磁盘预算设置
- [ ] 关键队列延迟、Puma backlog、Gitaly P99、磁盘水位四项有告警且演练过触发
- [ ] `/-/liveness`、`/-/readiness` 已接入拨测/LB 健康检查
- [ ] 生产部署都有 `environment`，DORA 报告非空
- [ ] 至少一条 Alertmanager → GitLab 告警链路，且 `env_name` 能对齐环境页

观测收尾，下一篇把前面各篇串成"团队落地手册"：结构规划、模板化、CI 规范与安全基线的完整清单。
