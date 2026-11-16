# 1. 基础概念

**Q: GitLab 是什么？和 GitHub 有什么区别？**

A: GitLab 是开源的一体化 DevOps 平台，以 Git 托管为核心，内置 MR 评审、CI/CD、制品仓库、安全扫描和项目管理。区别：GitLab 以自托管为主、open core 授权（CE 免费/EE 增值），CI 配置 `.gitlab-ci.yml` 与代码同仓；GitHub 以 SaaS 与开源生态为主，CI 靠 Actions 市场。选型上，数据主权和流程治理诉求强的企业多自建 GitLab。

**Q: 介绍一下 GitLab 的核心对象模型。**

A: Group 是权限与策略容器（可嵌套），Project 是协作最小单位（一个仓库 + Issue + 流水线 + Wiki + 包）。仓库内以分支承载代码，Merge Request 承载评审与合并，Pipeline 由 `.gitlab-ci.yml` 定义，拆成 Stage（串行）和 Job（并行），Job 由注册到实例的 Runner 执行，部署结果用 Environment 记录。权限是"角色 × 作用域 + 继承"：Guest/Reporter/Developer/Maintainer/Owner。

**Q: GitLab CI 里 Pipeline、Stage、Job、Runner 的关系？**

A: 一次 Git 事件触发一条 Pipeline；Pipeline 按 stages 数组划分阶段，阶段间串行；阶段内 Job 默认并行；Job 是执行单元，声明 image/script/artifacts 等；Job 进入队列后由 tags 匹配的 Runner 领取，按 executor（docker/kubernetes/shell）创建实际运行环境执行。

**Q: CE 和 EE 是什么关系？Free 层能干什么？**

A: CE 是开源核心（MIT），EE 在其上叠加企业功能。EE 按订阅分层：Free 已有托管、MR、CI/CD、Container/Maven 等包仓库、SAST/密钥/依赖扫描；Premium 加审批规则、Code Owner、Merge Train、Value Stream；Ultimate 加安全中心、DAST、合规框架。评估时注意付费点集中在治理与安全，核心交付流程免费层就完整。

# 2. CI/CD 机制

**Q: .gitlab-ci.yml 里 only/except 和 rules 的区别？**

A: only/except 是旧的条件语法，只能表达分支/tag 黑白名单，且与其他默认规则组合语义复杂。rules 是新标准：列表自上而下首条命中生效，支持 if 表达式（正则）、changes（按路径变更裁剪）、exists、when（manual/delayed/on_failure）并可在规则里附加 variables，不命中时 job 默认不创建。workflow:rules 是流水线级总闸，常用来避免 MR 流水线与分支流水线双跑。新项目一律用 rules。

**Q: 怎么把 stage 串行优化成并行？needs 是干什么的？**

A: 用 needs 声明精确依赖，把流水线从"阶段全串行"变成 DAG：job 只等它 needs 的上游，不等整个阶段，同时只下载 needs 上游的 artifacts。配合 rules:changes 按目录裁剪、parallel:matrix 跑版本矩阵、interruptible 取消过期流水线，可以大幅压缩 MR 反馈时间。

**Q: 几十个仓库如何统一 CI 规范？**

A: 建一个 ci-templates 模板仓库，把公共 job/变量/扫描配置做成模板文件，业务仓库用 include（project + ref 锁版本）引入，只覆盖差异变量。模板仓库自身走 MR 评审 + semver tag 发布。Monorepo 场景则用父子流水线：根 yml 按目录 changes 用 trigger 拉起各子项目独立流水线。

**Q: artifacts 和 cache 的区别？**

A: artifacts 是 Job 输出的正式交付物（jar、报告），跟随 Pipeline 保存（受 expire_in 控制），可被下游 Job 通过 dependencies 精确下载，是 Job 间数据传递的唯一正确通道。cache 是加速用的依赖缓存（.m2、node_modules），存在 Runner 本地或对象存储，跨 Pipeline 命中但不保证存在（Runner 异构、被清理都会 miss），绝不能当数据通道。

**Q: Runner 怎么把 Job 调度到指定机器？docker executor 和 kubernetes executor 怎么选？**

A: 调度靠 tags：Job 写 tags，Runner 注册时声明同样的标签集合（AND 匹配），并可关闭 run_untagged 防止接野活。docker executor 每 Job 起容器、镜像即环境，是事实标准，Runner 机器需装 Docker；kubernetes executor 把 Job 提交为集群 Pod，天然利用节点弹性伸缩与资源配额，适合已有 K8s 平台的企业，注意构建负载与业务负载做节点池隔离。

**Q: GitLab 里怎么实现"持续交付"（生产人工确认）？**

A: CI 阶段全自动，部署生产用 when: manual 按钮，或 start_in 延迟窗口；再叠加 environment: production（Protected 变量只对保护分支/Tag 可见密钥）+ 只有 Tag 或 main 触发生产 job。需要多人审批的发布流常用 Premium 审批或把编排交给 Argo CD 等声明式 CD。

# 3. 分支与协作

**Q: 保护分支能控制什么？和 Push Rules 什么区别？**

A: 保护分支控制"谁能推、谁能合、能不能 force push/删除"，可按通配批量保护（release/*），配合 Merge 设置实现"CI 必须绿 + 审批数 + 讨论必须 resolve"才能合并。Push Rules 是服务端在接收 push 时的硬校验（提交信息正则、拒绝未签名 commit、拒绝密钥），先于 CI、客户端绕不过；一个管"合并流程"，一个管"提交内容底线"。

**Q: MR 的最佳实践？**

A: 小 MR（≤400 行）+ Draft 早评审；描述用模板写影响面/测试/回滚；开 squash + 合并删源分支保持主干干净；Closes #issue 让需求随合并关闭；关键路径用 CODEOWNERS 强制 Owner 审批；并发合并易踩的仓库上 Merge Train。防"批准后又改"开 Reset approvals on push。

# 4. 架构与原理

**Q: 讲一下一次 git push 在 GitLab 内部的处理链路。**

A: 请求经 NGINX 到 Workhorse（Go，流式接收 pack，控制体积），再到 Rails 做 Preflight 鉴权（能否推该分支、Push Rules 校验），通过后把数据交给 Gitaly 真正落库（接 pack、执行 receive-pack、跑钩子），Gitaly 回调 Rails 更新 MR/触发 Pipeline，Sidekiq 异步消化 Webhook、邮件与状态传播。要点是"决策在 Rails、字节在 Go"：大流量不过 Ruby 进程。

**Q: 为什么 Gitaly 重要？仓库磁盘结构和裸 Git 服务器有何不同？**

A: Gitaly 是所有 Git 读写的唯一 RPC 入口，性能瓶颈（大仓库、Monorepo）与集群方案（Gitaly Cluster/Praefect 多副本）都围绕它。磁盘上仓库目录是哈希命名而非项目名，服务器上没有可信的 `.git/hooks` 手工编辑环境，自定义钩子要走 Gitaly hooks 或 Push Rules，任何情况下不要手工挪动/复制仓库目录。

**Q: GitLab 的元数据、Git 数据、缓存队列分别存在哪？备份策略怎么定？**

A: 元数据在 PostgreSQL（用户/项目/Issue/MR/流水线），Git 对象/LFS/制品/附件在磁盘或对象存储，Redis 承担会话、缓存与 Sidekiq 队列。备份以 gitlab-backup create 覆盖库+仓库+对象数据，/etc/gitlab 配置与密钥必须单独备份；Redis 不备（可重建）。演练恢复才是备份有效性的证明。

# 5. 安全与工程实践

**Q: GitLab 的安全扫描有哪些？怎么落地？**

A: 编码期 SAST + Secret Detection，依赖期 Dependency Scanning + License，构建期 Container Scanning + SBOM，运行期 DAST（Ultimate）。落地方式：include 官方模板进 CI，结果自动回显到 MR 页面与安全中心；关键一步是给扫描 job 配 rules 并用策略（如高危漏洞阻断合并、SLA 倒计时）让扫描有牙齿，而不是只出报告。

**Q: GitLab 有哪几种 Token？自动化应该用哪种？**

A: PAT（个人、权限随人漂移，不宜自动化）、Project/Group Access Token（系统集成，绑服务账号）、Deploy Token（只读拉取/Registry）、CI Job Token（job 内自动注入、短时效、可跨项目授权拉包）、Runner/Agent Token（执行器接入）。自动化禁用个人 PAT，CI 内推拉制品用 Job Token，服务器拉镜像用 Deploy Token。

**Q: 生产环境用 GitLab，容量规划从哪几处入手？**

A: 应用层按内存调 Puma worker 与 Sidekiq 并发；Git 层独立 Gitaly + SSD；数据库独立实例并可主从；Runner 容量按"触发频率 × 平均 Job 时长"估并发，弹性交给 kubernetes/docker+machine executor。治理面盯 Sidekiq 队列延迟、puma backlog、Gitaly 延迟、PG 连接数、存储增长五类指标。

# 6. 场景题

**Q: 团队从 Jenkins + Gerrit/GitLab 双轨迁到 GitLab 一体化，方案怎么给？**

A: 先定组织结构（群组层级、项目粒度）——这是改起来最贵的。仓库用导入工具整体迁移（保留分支/tag，钩子改 Push Rules/CI）。Jenkinsfile 按语言翻成 .gitlab-ci.yml，公共逻辑沉到 ci-templates 仓库统一 include；老 Jenkins 长期并存承接部署编排，新仓库默认 GitLab CI。门禁从"保护分支 + 流水线必须绿 + 审批"三件起步，扫描与 Merge Train 二期再上。

**Q: Job 一直 pending，怎么排查？**

A: 依次确认：Runner 是否在线且状态绿；Job 的 tags 与 Runner 标签是否全匹配（AND）；Runner 是否关闭了 run_untagged 而 Job 没写 tags；是否受保护分支/环境限制导致特定 Runner 才能接；并发上限是否被占满。页面上"Why is this job pending"通常直接给出原因。

**Q: 大仓库克隆慢、CI 拉代码慢，怎么优化？**

A: 平台侧：Gitaly 独立部署上 SSD、触发 Housekeeping 做 gc、检查网络出口带宽。使用侧：浅克隆（GIT_DEPTH 调小）、Git LFS 治理大文件按需拉、CI Runner 与仓库同机房、按 needs/changes 减少全量构建所需 checkout。Monorepo 则评估父子流水线按目录拆分触发范围。

**Q: 怎么防止密钥被提交进仓库？**

A: 三层：Push Rules rejectsecrets/rejectsecretlinks 在接收时拦截常见模式（服务端不可绕）；CI include Secret Detection 模板对 MR 扫描并阻断高危；组织侧推行 CI Variables（Masked/Protected）与外部 Vault 注入，杜绝"写死在 yml"的动机。事后泄露用历史重写 + 令牌立即轮换处置。

**Q: 老板要 DORA 指标，用 GitLab 怎么搭？**

A: 前提是把交付流数据化：MR 走保护分支 + CI 门禁，部署 job 声明 environment（staging/production）。之后 Analyze 里直接出部署频率、变更前置时间（MR 创建→部署生产）、变更失败率（结合回滚/hotfix 标记）；Value Stream 看四阶段停留。原则：指标看团队趋势，不进个人考核。

# 7. 面试速记总结

| 主题 | 一句话 |
|------|--------|
| 定位 | open core 一体化 DevOps：托管+MR+CI+制品+扫描同平台 |
| 对象 | Group→Project→Pipeline(Stage/Job)→Runner/Environment |
| CI 语法 | workflow/rules 管创建，needs 管 DAG，include 管复用，artifacts≠cache |
| Runner | tags 调度、docker 主流、K8s 弹性、Agent Token 注册 |
| 架构 | 决策在 Rails、字节在 Go（Workhorse/Gitaly）、异步在 Sidekiq |
| 门禁 | 保护分支+Pipelines must succeed+审批+CODEOWNERS |
| 安全 | 扫描左移有牙齿、令牌最小化、Push Rules 服务端拦截 |
| 运维 | 备 PG+仓库+配置，升级按 path，指标看队列/backlog/Gitaly |
