# 1. 基础概念

## 1.1 定位与差异

**Q: GitLab 是什么？和 GitHub 有什么区别？**

A: GitLab 是开源的一体化 DevOps 平台，以 Git 托管为核心，内置 MR 评审、CI/CD、制品仓库、安全扫描和项目管理。区别：GitLab 以自托管为主、open core 授权（CE 免费/EE 增值），CI 配置 `.gitlab-ci.yml` 与代码同仓；GitHub 以 SaaS 与开源生态为主，CI 靠 Actions 市场。选型上，数据主权和流程治理诉求强的企业多自建 GitLab。

追问方向——"一体化省掉了什么"：不是许可证钱，是跨系统账号同步、Webhook 胶水、多平台口径对账；代价是实例重（硬件基线 8C/16G 起）、升级运维要按规矩来。

## 1.2 核心对象模型

**Q: 介绍一下 GitLab 的核心对象模型。**

A: Group 是权限与策略容器（可嵌套），Project 是协作最小单位（一个仓库 + Issue + 流水线 + Wiki + 包；磁盘上最多 3 个仓库：主仓 + `-wiki` + `-design`）。仓库内以分支承载代码，Merge Request 承载评审与合并，Pipeline 由 `.gitlab-ci.yml` 定义，拆成 Stage（串行）和 Job（并行），Job 由注册到实例的 Runner 执行，部署结果用 Environment 记录。权限是"角色 × 作用域 + 继承"：Guest/Reporter/Developer/Maintainer/Owner。

## 1.3 CI 四件套的关系

**Q: GitLab CI 里 Pipeline、Stage、Job、Runner 的关系？**

A: 一次 Git 事件触发一条 Pipeline；Pipeline 按 stages 数组划分阶段，阶段间串行；阶段内 Job 默认并行；Job 是执行单元，声明 image/script/artifacts 等；Job 进入队列后由 tags 匹配的 Runner 领取，按 executor（docker/kubernetes/shell）创建实际运行环境执行。

## 1.4 版本与授权层级

**Q: CE 和 EE 是什么关系？Free 层能干什么？**

A: CE 是开源核心（MIT），EE 在其上叠加企业功能——但注意**两条轴独立**：EE 实例不挂许可证就是 Free 层，"CE≠Free、EE≠全付费"。Free 已有托管、MR、CI/CD、Container/Maven 等包仓库、SAST/密钥检测/容器扫描/IaC 扫描，以及全部三种合并方法；Premium 加审批规则、Code Owner、群组推送规则下发、Pull 镜像、Merge Train、Value Stream；Ultimate 加安全中心、DAST、**依赖扫描与推送时密钥拦截**、合规框架、审计导出。核心交付流程 Free 完整，付费点集中在治理与安全。

**Q: 关于订阅层级，最容易被过时资料误导的三点？**

A: ①"半包含合并是 Premium"——过时，三种合并方法现在全档位；②"免费层能跑全套扫描"——不准确，Dependency Scanning 和 Secret Push Protection 是 Ultimate；③把网页功能当成档位依据——以文档每页 `Tier:` 横幅和自己的 CI 展开结果为准，Self-Managed 与 SaaS 的 Offering 还可能有差异。

## 1.5 发布与升级节奏

**Q: GitLab 的发版节奏？升级有什么规矩？**

A: minor 每月一发，补丁小版本随时发布（含例行安全补丁）。17.5 起升级路径固定**季度停靠点：x.2 / x.5 / x.8 / x.11**——不是"每个小版本都得跟"，但停靠点必须停、**每个停靠点要等后台迁移跑完**再走下一步；同一 minor 内永远升到最新补丁（16.8.7 而非 16.8.0）。降级不受官方支持，唯一后悔药是升级前的 `gitlab-backup create` + 快照。跨大版本先查 PostgreSQL 支持矩阵（如 GitLab 19.x 要求 PG ≥17），数据库常常要先行。

# 2. CI/CD 机制

## 2.1 触发条件

**Q: .gitlab-ci.yml 里 only/except 和 rules 的区别？**

A: only/except 是旧的条件语法，只能表达分支/tag 黑白名单，且与其他默认规则组合语义复杂。rules 是新标准：列表自上而下首条命中生效，支持 if 表达式（正则）、changes（按路径变更裁剪）、exists、when（manual/delayed/on_failure）并可在规则里附加 variables，不命中时 job 默认不创建。workflow:rules 是流水线级总闸，常用来避免 MR 流水线与分支流水线双跑。新项目一律用 rules。

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

## 2.2 DAG 与提速

**Q: 怎么把 stage 串行优化成并行？needs 是干什么的？**

A: 用 needs 声明精确依赖，把流水线从"阶段全串行"变成 DAG：job 只等它 needs 的上游，不等整个阶段，同时只下载 needs 上游的 artifacts。配合 rules:changes 按目录裁剪、parallel:matrix 跑版本矩阵、interruptible 取消过期流水线，可以大幅压缩 MR 反馈时间。

```yaml
test-core: { stage: test, script: [mvn -pl core test] }
test-web:  { stage: test, needs: ["test-core"], script: [mvn -pl web test] }
```

## 2.3 规模化复用

**Q: 几十个仓库如何统一 CI 规范？**

A: 建一个 ci-templates 模板仓库，把公共 job/变量/扫描配置做成模板文件，业务仓库用 include（project + ref 锁版本）引入，只覆盖差异变量。模板仓库自身走 MR 评审 + semver tag 发布。Monorepo 场景则用父子流水线：根 yml 按目录 changes 用 trigger 拉起各子项目独立流水线。

```yaml
include:
  - project: 'ci-templates'
    ref: 'v2.4.0'                    # 必须锁版本，模板 main 漂移会击穿全公司
    file: '/templates/java.yml'
```

## 2.4 artifacts 与 cache

**Q: artifacts 和 cache 的区别？**

A: artifacts 是 Job 输出的正式交付物（jar、报告），跟随 Pipeline 保存（受 expire_in 控制），可被下游 Job 通过 dependencies 精确下载，是 Job 间数据传递的唯一正确通道。cache 是加速用的依赖缓存（.m2、node_modules），存在 Runner 本地或对象存储，跨 Pipeline 命中但不保证存在（Runner 异构、被清理都会 miss），绝不能当数据通道。

## 2.5 Runner 调度

**Q: Runner 怎么把 Job 调度到指定机器？docker executor 和 kubernetes executor 怎么选？**

A: 调度靠 tags：Job 写 tags，Runner 注册时声明同样的标签集合（AND 匹配），并可关闭 run_untagged 防止接野活。docker executor 每 Job 起容器、镜像即环境，是事实标准，Runner 机器需装容器运行时；kubernetes executor 把 Job 提交为集群 Pod，天然利用节点弹性伸缩与资源配额，适合已有 K8s 平台的企业，注意构建负载与业务负载做节点池隔离。注册方式记最新口径：**authentication token**（REST 创建 Runner 后取得），老 registration token 已在 17.0 移除。

## 2.6 持续交付

**Q: GitLab 里怎么实现"持续交付"（生产人工确认）？**

A: CI 阶段全自动，部署生产用 when: manual 按钮，或 start_in 延迟窗口；再叠加 environment: production（Protected 变量只对保护分支/Tag 可见密钥）+ 只有 Tag 或 main 触发生产 job。需要多人审批的发布流常用 Premium 审批或把编排交给 Argo CD 等声明式 CD。

## 2.7 资源治理

**Q: 流水线把 Runner 池跑挂了，有哪些控制手段？**

A: 分层治理：①workflow:rules 收紧触发面（别双跑、非主干不跑全量）；②`interruptible: true` + `workflow: auto_cancel` 让过期 MR 流水线互斥取消；③`resource_group` 串行化对同一资源的部署（同名 resource_group 同时只跑一个 job）；④利用项目/群组/实例层的流水线配额与速率限制设置（字段以设置页读回为准，别背）；⑤Runner 侧并发上限（`limit`）与标签隔离。观测入口：Admin → Operations → Runner jobs 与队列。

```yaml
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  auto_cancel:
    on_new_commit: interruptible     # 新提交到来时中断 interruptible 的旧 job

deploy-prod:
  resource_group: production         # 同一时刻只允许一个生产部署
```

## 2.8 事件与流水线类型

**Q: 哪些事件会创建流水线？各自对应什么 source？**

A: 分支 push（`push`）、MR 创建/更新（`merge_request_event`）、打 tag（`tag` 或 `push`+`CI_COMMIT_TAG`）、定时（`schedule`）、手动/重跑（`web`/`api`/`trigger`/`pipeline`）、上游流水线（父子 `pipeline`）。面试考点是"MR 流水线 vs 分支流水线双跑"：用 workflow:rules 限定 `merge_request_event`，或项目设置关掉分支流水线；`$CI_PIPELINE_SOURCE` 是裁剪的第一变量。

# 3. 分支与协作

## 3.1 保护分支与 Push Rules

**Q: 保护分支能控制什么？和 Push Rules 什么区别？**

A: 保护分支控制"谁能推、谁能合、能不能 force push/删除"，可按通配批量保护（release/*），多条规则命中同一分支时**取最宽松**；配合 Merge 设置实现"CI 必须绿 + 审批数 + 讨论必须 resolve"才能合并。Push Rules 是服务端在接收 push 时的硬校验（提交信息正则、拒绝未签名 commit、拦截密钥文件名），先于 CI、客户端 `--no-verify` 绕不过；一个管"合并流程"，一个管"提交内容底线"。

```bash
$ git push origin main
remote: GitLab: You are not allowed to push code to protected branches on this project.
```

## 3.2 MR 最佳实践

**Q: MR 的最佳实践？**

A: 小 MR（≤400 行）+ Draft 早评审；描述用模板写影响面/测试/回滚；squash 设 Encourage/Require + 合并删源分支保持主干干净；Closes #issue 让需求随合并关闭；关键路径用 CODEOWNERS 强制 Owner 审批；并发合并易踩的仓库上 Merge Train。防"批准后又改"开 Reset approvals on push。

## 3.3 推送规则的继承陷阱

**Q: 群组配了推送规则，为什么存量项目没生效？**

A: 这是设计不是 bug。全局/群组推送规则**只作为模板被"配置之后新建"的项目继承一份副本**，项目级是替换不是叠加，改动也不回灌存量。批量刷存量走 API（端点是单数 `push_rule`）：

```bash
glab api projects/:id/push_rule | jq            # 先读回现值
glab api --method POST projects/:id/push_rule \
  -f commit_message_regex='^(feat|fix|chore): .+' -f prevent_secrets=true
```

同类陷阱还有分支命名校验：默认分支**永远**被命名规则放行；40 位十六进制样式的分支名被服务端直接禁止。

## 3.4 默认分支保护

**Q: 新项目的默认分支保护能统一下发吗？有几档？**

A: 能，实例级与群组级可给"每个新仓库的默认分支"下发初始保护，五档：Fully protected（默认）/ Fully protected after initial push / Protected against pushes / Partially protected / Not protected。安全追问：官方明确警告**只要不是 Fully protected，恶意开发者可把外带 CI 变量的 `.gitlab-ci.yml` 提交上受保护分支**，流水线一跑群组变量就泄漏——所以"给小团队开 Partially"要先过安全这关。

## 3.5 CODEOWNERS 排错

**Q: 配了 CODEOWNERS 为什么没拦住合并？**

A: 按生效链条查三处：①**文件是否被找到**——每仓库只认一个 CODEOWNERS，按 根 → `docs/` → `.gitlab/` 顺序**第一命中制**，三处都有时只有根那份算数；②**目标分支是否受保护且开了 Code Owner 审批**——两个条件缺一不可；③**比对的是目标分支上的 CODEOWNERS 版本**——owner 规则没合进主干前不生效。另外没命中任何规则的文件走普通审批，敏感路径要有兜底通配。

## 3.6 三种合并方法

**Q: Merge commit、Semi-linear、Fast-forward 怎么选？**

A: 全档位免费可用。Merge commit 最宽松（每次合并留 merge commit，等价 `git merge --no-ff`），但不保证"MR 绿则主干绿"；Semi-linear 要求合并前可 FF（必要时自动 rebase）仍留 merge commit；FF 出纯线性历史（`--ff-only`，配 squash 即"每 MR 一条原子提交"）。主干开发 + 高频合并选 FF + Require squash；要保留分支拓扑或跨 fork 协作选 Merge commit。squash 现在是四档策略（Do not allow/Allow/Encourage/Require），不是勾选框。

# 4. 架构与原理

## 4.1 push 链路

**Q: 讲一下一次 git push 在 GitLab 内部的处理链路。**

A: 请求经 NGINX 到 Workhorse（Go，流式接收 pack，控制体积），Workhorse 回头问 Rails 内部 API 做 preflight 鉴权（能否推该分支、Push Rules 校验），通过后把数据交给 Gitaly 真正落库（执行 `git receive-pack`、跑钩子），Gitaly 回调 Rails 更新 MR/触发 Pipeline，Sidekiq 异步消化 Webhook、邮件与状态传播。要点是"决策在 Rails、字节在 Go"：大流量不过 Ruby 进程。

## 4.2 Gitaly 与哈希存储

**Q: 为什么 Gitaly 重要？仓库磁盘结构和裸 Git 服务器有何不同？**

A: Gitaly 是 11.3 起所有 Git 读写的唯一 RPC 入口，官方点名的客户端有六个（Rails、Shell、Workhorse、ES Indexer、Zoekt Indexer、KAS）——"没 push 但 Gitaly 很忙"多半是索引器在扫库。磁盘上是 `git-data/repositories/@hashed/<前2位>/<次2位>/<sha256(项目ID)>.git`：以项目 ID 哈希命名，改名/改组只是数据库事务、磁盘零移动。**不要手工挪动仓库目录**（官方已把直接访问仓库目录标记为弃用），路径互查用 Admin Area 的 Relative path 或 `Project.find(id).disk_path`。钩子没有 `.git/hooks` 手工环境，自定义走 Gitaly custom hooks 目录或 Push Rules。

## 4.3 数据存储与备份

**Q: GitLab 的元数据、Git 数据、缓存队列分别存在哪？备份策略怎么定？**

A: 元数据在 PostgreSQL（用户/项目/Issue/MR/流水线），Git 对象/LFS/制品/附件在磁盘或对象存储，Redis 承担会话、缓存与 Sidekiq 队列。`gitlab-backup create` 覆盖库 + 仓库 + 对象数据，**但不含 Redis、对象存储里的数据、`/etc/gitlab`（含 gitlab-secrets.json）**——配置与密钥必须单独备份；恢复后要做 `gitlab-rake gitlab:check SANITIZE=true` + `gitlab:doctor:secrets` 校验，演练恢复才是备份有效性的证明。

```bash
sudo gitlab-backup create
sudo tar czf etc-gitlab-$(date +%F).tar.gz /etc/gitlab   # 配置+密钥单独进保险柜
```

## 4.4 SSH 克隆链路

**Q: SSH 克隆在 GitLab 内部怎么鉴权？**

A: 没有任何 GitLab 组件"说 SSH"——sshd 终止连接，所有会话都以 `git` 系统用户落地，用客户端公钥区分 GitLab 用户。开 fast SSH key lookup 时走 `gitlab-shell-authorized-keys-check`（AuthorizedKeysCommand）→ Rails 内部 API 查 key → 强制命令交给 gitlab-shell → 再问 `/internal/api/allowed` 做动作级鉴权 → 最后由 Gitaly `SSHService` 派生 `git upload-pack`。没开时 `~git/.ssh/authorized_keys` 由 Rails 的 AuthorizedKeysWorker 异步维护——"新加的部署密钥不生效"先查这个任务是否积压。

## 4.5 故障指纹

**Q: 页面正常但上传/HTTP 克隆失败，怀疑谁？**

A: Workhorse（大文件与 git 流量的搬运工挂了特征就是这个）。配套指纹：SSH 克隆全挂、Web 正常 → GitLab Shell；页面转圈、git 正常 → Puma/PG（Web 线）；仓库操作慢、页面正常 → Gitaly；状态延迟不报错 → Sidekiq 积压；登录态反复丢 → Redis。进程线与数据线两条检查命令；Rails JSON 日志带 correlation_id，能串起多组件重建一次请求。

```bash
sudo gitlab-ctl status                        # 进程线：谁没在跑
sudo gitlab-rake gitlab:check SANITIZE=true   # 数据线：权限/钩子/仓库可达
```

## 4.6 组件约束

**Q: Redis 和 PostgreSQL 在架构上有什么硬性约束？**

A: Redis：**不支持 Redis Cluster**、不支持 serverless 形态，只能单机/哨兵；且不在备份内，丢了=全员重登 + 队列清空。PG：唯一不可丢组件，大版本支持矩阵跟 GitLab 版本走（18.x 要 ≥16.5，19.x 要 ≥17.x）；必需扩展 `pg_trgm`、`btree_gist`、`amcheck`；17.x 起元数据拆 main/ci 双库，连接需求 = Puma worker×thread + Sidekiq 并发，调 worker 前先算 `max_connections`。

# 5. 安全与工程实践

## 5.1 扫描落地

**Q: GitLab 的安全扫描有哪些？怎么落地？**

A: 编码期 SAST + Secret Detection（Free 可跑），构建期 Container Scanning + SBOM（Free）、**依赖扫描与推送时密钥拦截是 Ultimate**，运行期 DAST（Ultimate）。落地方式：include 官方模板进 CI，结果自动回显到 MR 页面与安全中心；关键一步是给扫描 job 配 rules 并用策略（如高危漏洞阻断合并、SLA 倒计时）让扫描有牙齿，而不是只出报告。离线实例还要配 `SECURE_ANALYZERS_PREFIX` 指向内网镜像的分析器镜像仓库（第 12 篇）。

## 5.2 令牌体系

**Q: GitLab 有哪几种 Token？自动化应该用哪种？**

A: PAT（个人、权限随人漂移，不宜自动化）、Project/Group Access Token（系统集成，绑服务账号）、CI Job Token（job 内自动注入、短时效、可跨项目授权拉包）、Deploy Token/Key（只读拉取/Registry）、Runner 认证令牌、Agent Token、Impersonation Token（代客，慎用）。注意三个行为细节：多数令牌前缀都是 `glpat-`（前缀不能用来区分类型）；**过期按 UTC 零点执行**不是自然日；群组 IP 限制命中时对 project/group token 返回 404 而非 403（第 12 篇核实）。自动化禁用个人 PAT；CI 内推拉制品用 Job Token，服务器长期拉镜像用 Deploy Token。

## 5.3 防密钥入库

**Q: 怎么防止密钥被提交进仓库？**

A: 三层：①Push Rules 的 `prevent_secrets`（内置文件名黑名单：`id_rsa`/`*.pem`/`*.key`/`.aws/credentials` 等，服务端拦截；注意只拦新推送、不看内容）+ Ultimate 的 Secret Push Protection（内容级扫描才更强）；②CI include Secret Detection 模板对 MR 扫内容并阻断高危；③组织侧推行 CI Variables（Masked/Protected）与外部凭据注入，杜绝"写死在 yml"的动机。事后泄露 = 立即轮换 + 历史重写（先想清楚 fork/重 clone 的坑，第 4 篇 11.5）。

## 5.4 CI 变量泄漏面

**Q: GitLab 的 CI/CD 变量可能被谁拿走？**

A: 四条典型路径：①非 Fully protected 的默认分支上，任何能提交者写一段 `echo $SECRET` 的 `.gitlab-ci.yml` 就能触发（官方点名的场景，见 3.4）；②fork 的 MR 默认拿不到 Protected 变量，但配置不当的开放 Runner 会接活；③Masked 不等于安全——能被 job 输出间接推断时仍可还原，生产密钥用 Vault/短时凭据；④令牌外泄（个人 PAT 权限随人）。防线一句话：**谁能提交 yml = 谁能执行带变量的环境**，保护分支、审批、Runner 注册管控是同一件事的三面墙。

# 6. 容量与运维

## 6.1 容量规划

**Q: 生产环境用 GitLab，容量规划从哪几处入手？**

A: 应用层按内存调 Puma worker 与 Sidekiq 并发；Git 层独立 Gitaly + SSD；数据库独立实例并可主从；Runner 容量按"触发频率 × 平均 Job 时长"估并发，弹性交给 kubernetes/docker+machine executor。治理面盯 Sidekiq 队列延迟、puma backlog、Gitaly 延迟、PG 连接数、存储增长五类指标。

## 6.2 硬件口径

**Q: 现在给自建 GitLab 申请资源，底线是多少？**

A: 单机官方基线 **8 vCPU / 16 GB**（内存受限最低 8 GB 走专门调优档），别按老教程的 4C/4G+swap 报预算——现行口径是**尽量关 swap**、给足内存。磁盘 SSD，应用节点 40 GB 起，NFS/EFS/Azure Files 官方点名避开；突发性能实例（burstable）不推荐。PG 5-12 GB（Ultimate 12 起，漏洞库占 1 GB）。

## 6.3 升级编排

**Q: 从 17.x 升到 19.x，完整动作序列？**

A: 查路径停靠点（x.2/x.5/x.8/x.11 逐个停）→ 每个停靠点前备份 + 跑升级前检查 → 升到该 minor 最新补丁 → **等后台迁移清零**再下一跳 → 每跳后 `gitlab-rake gitlab:check`。中间穿插：PG 版本先满足目标矩阵、Runner 版本跟紧（官方建议同版本）、低峰窗口（升级期 Web 只读）。降级不支持，回退方案只有"恢复演练过的备份"。

## 6.4 备份规模问题

**Q: 实例 PG 已经 150 GB，`gitlab-backup create` 跑不动怎么办？**

A: 官方阈值：PG **>100 GB 时 pg_dump 路径基本不可用**（Git 总量 ~400 GB 时备份命令整体接近不可用）。解法分轨：仓库走服务端备份 + 增量：

```bash
# 仓库走服务端（tar 在 Gitaly 侧生成），数据库交给第三方工具
sudo REPOSITORIES_SERVER_SIDE=true SKIP=db gitlab-backup create
```

数据库改 pgBackRest/barman 等专用工具，应用层按 13 篇的大实例三轨架构拆；对象存储里的数据本来就不进包，桶侧单独做复制/版本。核心口径：**备份策略是架构问题，不是脚本问题**——等跑不动才想就晚了。

# 7. 场景题

## 7.1 迁移一体化

**Q: 团队从 Jenkins + Gerrit/GitLab 双轨迁到 GitLab 一体化，方案怎么给？**

A: 先定组织结构（群组层级、项目粒度）——这是改起来最贵的。仓库用导入工具整体迁移（保留分支/tag，钩子改 Push Rules/CI；Gerrit 的评审模型和 MR 不同，要专门做流程映射）。Jenkinsfile 按语言翻成 `.gitlab-ci.yml`，公共逻辑沉到 ci-templates 仓库统一 include；老 Jenkins 长期并存承接部署编排，新仓库默认 GitLab CI。门禁从"保护分支 + 流水线必须绿 + 审批"三件起步，扫描与 Merge Train 二期再上。

## 7.2 Job pending

**Q: Job 一直 pending，怎么排查？**

A: 依次确认：Runner 是否在线且状态绿；Job 的 tags 与 Runner 标签是否全匹配（AND）；Runner 是否关闭了 run_untagged 而 Job 没写 tags；是否受保护分支/环境限制导致特定 Runner 才能接；并发上限是否被占满。页面上"Why is this job pending"通常直接给出原因。

## 7.3 大仓库提速

**Q: 大仓库克隆慢、CI 拉代码慢，怎么优化？**

A: 平台侧：Gitaly 独立部署上 SSD、触发 Housekeeping 做 gc、检查网络出口带宽。使用侧：浅克隆、Git LFS 治理大文件按需拉、CI Runner 与仓库同机房、按 needs/changes 减少全量构建所需 checkout。Monorepo 则评估父子流水线按目录拆分触发范围。

```yaml
variables:
  GIT_DEPTH: "10"          # CI 浅克隆；需要全历史时再 fetch --unshallow
```

## 7.4 DORA 指标

**Q: 老板要 DORA 指标，用 GitLab 怎么搭？**

A: 前提是把交付流数据化：MR 走保护分支 + CI 门禁，部署 job 声明 environment（staging/production）。之后 Analyze 里直接出部署频率、变更前置时间（MR 创建→部署生产）、变更失败率（结合回滚/hotfix 标记）；Value Stream 看四阶段停留。原则：指标看团队趋势，不进个人考核。

# 8. 高频追问速答

| 追问 | 一句话答案（展开篇号） |
|------|------------------------|
| include 能引用别的 Git 服务吗？ | 能，file/remote/template 三种源，跨实例用 remote URL（第 7 篇） |
| 变量优先级谁高？ | 触发时变量 > 项目 > 群组 > 实例，同名覆盖有方向，别赌顺序（第 7 篇） |
| 为什么 webhook 没收到？ | 先查 `hooks/:hook_id/events` 投递历史（只保留 7 天），重放走 resend（第 13 篇） |
| Auditor 角色是什么？ | 不是新角色，是 User type：全只读 + 强制审计视角，**照样占席位**（第 12 篇） |
| Geo 是备份吗？ | 不是，Geo 是灾备/读分担，数据一致性靠复制，备份仍是 `gitlab-backup`（第 13 篇） |
| 恢复后仓库显示为空？ | 查 fapolicyd 类安全模块拦截 Gitaly 读对象（第 13 篇恢复隐性失败清单） |
| 保护分支上 Developer 能点合并吗？ | 能——Allowed to merge 单独放开即可，推与合是两个旋钮（第 4 篇） |
| merge train 解决什么？ | "批准时都绿、合并后主干红"：按队列串行 rebase 验证再入库（Premium，第 5 篇） |
| Runner 的 shell executor 能用吗？ | 能但隔离性差，仅裸机场景；生产主流 docker/kubernetes（第 8 篇） |
| 实例默认分支名能统一吗？ | 能，实例/群组设置统一下发新仓库默认分支名（第 4 篇） |

# 9. 面试速记总结

| 主题 | 一句话 |
|------|--------|
| 定位 | open core 一体化 DevOps：托管+MR+CI+制品+扫描同平台；CE/EE 与订阅层是两条轴 |
| 对象 | Group→Project→Pipeline(Stage/Job)→Runner/Environment |
| CI 语法 | workflow/rules 管创建，needs 管 DAG，include 管复用，artifacts≠cache，resource_group 管串行 |
| Runner | tags 调度、docker 主流、K8s 弹性、认证令牌注册（registration token 已删） |
| 架构 | 决策在 Rails、字节在 Go（Workhorse/Gitaly）、异步在 Sidekiq；@hashed 盘不可碰 |
| 门禁 | 保护分支+Pipelines must succeed+审批+CODEOWNERS；推送规则"新建才继承、项目即替换" |
| 安全 | 扫描分档（DS/推送拦截=Ultimate）、令牌最小化、变量防护=保护分支问题 |
| 运维 | 基线 8C/16G 关 swap、季度停靠点升级、备 PG+仓库+/etc/gitlab、Redis 不进包 |
