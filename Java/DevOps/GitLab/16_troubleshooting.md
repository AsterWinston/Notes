# 1. 排错方法论：先分层，再动手

GitLab 故障排查的总纲对应第 3 篇架构图——**先判断请求走的是哪条线，再到对应组件找日志**：

```text
用户报障 ──► 是 Web/API？        → NGINX → Workhorse → Puma(Rails) → PostgreSQL/Redis
          ──► 是 git clone/push？ → NGINX/SSH → Gitaly → 磁盘
          ──► 是流水线不动？      → Runner 注册/tags/队列 → Sidekiq
          ──► 是通知/状态延迟？    → Sidekiq 队列积压
第一步永远是: sudo gitlab-ctl status  +  gitlab-rake gitlab:check
```

# 2. Web 界面类

| 现象 | 高频原因 | 处理 |
|------|---------|------|
| 502 | Puma 未就绪（刚重启/迁移中）或 worker 全忙 | 等 2-3 分钟；`gitlab-ctl tail puma`；看 backlog 指标 |
| 页面极慢 | worker 数 < 并发、PG 慢查询、Sidekiq 抢占 CPU | 调 `puma['worker_processes']`（别超内存预算）、查 `pgslow.log` |
| 504 在克隆大仓库页面 | Workhorse/nginx 超时 | 增大 `nginx['proxy_read_timeout']`，改用 git CLI |
| 登录循环跳回登录页 | Cookie/HTTPS 配置、Redis 会话丢失、时钟漂移 | 核对 `external_url` 与实际访问域一致；Redis 健康 |
| 500 只在某项目 | 项目数据异常（迁移残留） | `gitlab-rake gitlab:project_check:<ID>` |
| 静态资源 404 | Nginx 配置被手改/资产未预编译 | 回归 gitlab.rb + reconfigure |

> **注意**：所有定制走 `gitlab.rb`，**手改 `/var/opt/gitlab/nginx/gitlab_config` 之类的生成文件必被 reconfigure 覆盖**，这是"越修越坏"的第一来源。

# 3. Git 操作类（clone/push/fetch）

| 现象 | 定位 | 处理 |
|------|------|------|
| push 大包 504/413 | nginx/workhorse 体积限制 | 提高 `nginx['client_max_body_size']`；改用分批 push/对象存储 |
| `The remote end hung up unexpectedly` | Gitaly 超时/内存不足、网络 MTU | 看 gitaly 日志；调 `gitaly['concurrency']`；大仓库走 SSH |
| SSH 克隆 Permission denied | 密钥未加、Docker 端口未转发、`gitlab_shell['ssh_port']` 不一致 | `ssh -T git@host -p <port>` 验证；核对显示端口 |
| 克隆地址 http/https 不对 | `external_url` 与访问方式不符 | 改 external_url + reconfigure |
| 大仓库操作越来越慢 | `.git` 膨胀（松散对象、LFS 滥用） | 联系管理员触发 **Housekeeping**（Gitaly 自动 gc）；仓库瘦身 |
| Web hook 收不到 push 事件 | Sidekiq 积压 / 目标地址不通（出站限制） | Admin → Hooks 看失败记录，重放 |

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

Runner 侧命令：`gitlab-runner verify`（认证有效吗）、`gitlab-runner list`、`journalctl -u gitlab-runner -f`（或容器 logs）。

# 5. 性能与容量类

| 症状 | 元凶概率排序 | 处方 |
|------|-------------|------|
| Sidekiq 队列持续增长 | 邮件/Webhook 风暴、慢任务占满线程 | 加 Sidekiq 并发或扩实例；Admin → Sidekiq 面板看 slowest |
| 全站偶发超时 | PG 连接耗尽 | `postgresql['shared_buffers']`/max_connections；应用机与库机分离 |
| 磁盘涨疯了 | 制品/LFS/备份未清 | 清理策略（第 10 篇）、配额（第 13 篇）、对象存储卸载 |
| 内存 OOM killer 杀进程 | Puma/Sidekiq/Gitaly 叠加超配 | 按角色拆机；先砍 worker 数量换稳定 |

# 6. 邮件与通知

- 发不出：`gitlab-rails console` 里 `Notify.test_email('you@x.com','t','b').deliver_now` 直接测 SMTP。
- 25 端口被云厂商封 → 465/587；`gitlab.rb` smtp 改完必须 reconfigure。
- 收到了但延迟：Sidekiq mailer 队列积压，回到本篇第 5 节处理。

# 7. 日志速查表（配合第 14 篇）

| 场景 | 一条命令 |
|------|---------|
| 全组件状态 | `sudo gitlab-ctl status` |
| 实时看错误 | `sudo gitlab-ctl tail`（全量）/ `tail puma`/`tail gitaly` |
| 结构化错误过滤 | `journalctl -u gitlab-runsvdir -p err` |
| 数据库慢查询 | `/var/log/gitlab/postgresql/pgslow.log` |
| API 审计 | `gitlab-rails/audit_json.log`（含 token 使用者） |
| 排错前自检 | `sudo gitlab-rake gitlab:check` |

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 分层定位 | Web 线 / Git 线 / CI 线 / 异步线，四线日志各管一段 |
| 三板斧 | `gitlab-ctl status` → 对应组件 tail → `gitlab:check` |
| CI 排错 | pending 查 Runner/tags，缺 job 查 rules |
| 配置纪律 | 只改 gitlab.rb + reconfigure，不碰生成物 |
| 容量问题 | 指标先行（队列/连接/磁盘），备份兜底 |

至此使用者视角的模块齐了。最后一篇按仓库惯例做面试问答汇总。
