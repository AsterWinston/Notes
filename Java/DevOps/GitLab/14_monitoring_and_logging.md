# 1. GitLab 的可观测性分两层

谈"GitLab 监控"要先分清对象，否则容易在错误的仪表盘里找答案：

| 层 | 监控什么 | 工具入口 |
|----|---------|---------|
| **平台自身健康** | GitLab 实例：Puma/Sidekiq/Gitaly/PG/Redis 的指标与日志 | Omnibus 内置 Prometheus/Grafana、`/metrics`、GitLab Shell logs |
| **CI/CD 效能与交付质量** | 流水线成功率、时长、MR 周期、部署频率 | CI/CD Analytics、Value Stream、DORA 报告（Premium/Ultimate） |

对使用者（非专职运维），第二层才是每天要看的；第一层主要服务于排错（第 16 篇展开日志）。

# 2. 平台自身：内置监控栈

Omnibus 自带一套 Prometheus + Grafana（默认只监听本机）：

```bash
# 开启内置 Grafana 对外访问（gitlab.rb）
grafana['enable'] = true
# 访问 https://gitlab.example.com/-/metrics（需管理员授权）
# 各组件导出器（puma/sidekiq/gitaly/redis/pg 等）监听本机端口，默认不对外
sudo gitlab-ctl reconfigure
sudo gitlab-ctl tail prometheus
```

| 导出器/指标 | 关注点 |
|------------|--------|
| `puma_backlog` / `puma_workers` | Web 请求排队 → 加 worker/加机器 |
| `sidekiq_queue_latency` / `enqueued` | 异步任务积压（MR 合并后通知延迟的元凶） |
| `gitaly_persistent_storage_seconds` | Git 操作耗时 → 磁盘瓶颈 |
| `pg_stat_*`（内置 PG exporter） | 慢查询、连接数、复制延迟 |
| `redis_*` | 内存、命中率、队列长度 |
| Workhorse/nginx 指标 | 入口 5xx、上传大小 |

非 Omnibus 部署（K8s Helm）则用 ServiceMonitor 接团队统一 Prometheus；GitLab 官方提供 **dashboards 仓库**（gitlab-com/grafana-dashboards），关键面板导入即用。

# 3. 日志体系：四类日志各司其职

| 日志 | Omnibus 路径 | 什么时候看 |
|------|-------------|-----------|
| 应用日志 production.log | `/var/log/gitlab/gitlab-rails/` | API/页面行为异常、权限问题 |
| 异步任务 | `.../sidekiq.log` + Admin 的 Sidekiq 队列页 | 通知/Webhook/状态传播延迟 |
| Git 操作 | `gitaly.log`、`gitlab-shell.log` | clone/push 失败与慢 |
| 入口 | `nginx.log`、`workhorse.log` | 502/413/上传失败 |

排错口诀：**先分"哪条线"——Web 线看 rails+sidekiq，Git 线看 gitaly/shell，入口线看 nginx/workhorse**；再拿 `gitlab-rake gitlab:check` 做全局体检（第 2 篇命令表）。

# 4. CI/CD 分析：交付流水线本身的观测

项目/群组 → **Analyze → CI/CD Analytics**：

- **Pipeline/Job 时长趋势**：哪个 job 在拖慢全团队（优化 `rules`/cache/needs 的依据，第 7 篇）。
- **失败率**：按分支/job 统计；持续失败的 job 要么修要么降级为 `allow_failure` 后另行跟踪。
- **Runner 利用率与队列等待**：pending 时间占比高 → 扩 Runner（第 8 篇）。

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

> **行业实践**：DORA/Value Stream 用于**团队自改进**，别用于个人绩效——指标一旦与考核绑定，团队会立刻学会刷低 MR 粒度、合并前造假绿。度量看趋势不看单点。

# 6. 告警与 Incident（Ultimate 偏多）

- **Alert Management**：接收外部监控告警（Prometheus Alertmanager/Webhook 格式），在 GitLab 内分诊、关联 Issue。
- **Incident 时间线**：告警 → 建 Incident Issue → 记录处置评论 → 复盘（Postmortem）；与 MR/部署时间线互相链接——"这次故障对应哪次发布"自动可查。
- 运维侧兜底：把 `/-/readiness`、`/-/liveness` 健康端点接进你们的拨测/负载均衡。

# 7. 用量与成本观测

Admin Area → **Usage Quotas**：存储（仓库/LFS/制品/附件/CI artifacts 分项）、CI 分钟数、活跃用户。群组/项目页也有各自 usage。治理动作（清制品、调保留策略、禁 LFS 大文件）全部由这些数字驱动。

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 两层监控 | 平台健康（Prom 指标+日志）与交付效能（Analytics/Value Stream） |
| 排错地图 | Web/Sidekiq/Gitaly/入口四条日志线各管一段 |
| CI 观测 | 时长、失败率、Runner 队列是 CI 优化的三个入口 |
| DORA | 数据来自 environment 与流水线元数据，配好才有报告 |
| 告警 | Alert Management + Incident 把故障与发布历史串起来 |

观测收尾，下一篇把前面各篇串成"团队落地手册"：结构规划、模板化、CI 规范与安全基线的完整清单。
