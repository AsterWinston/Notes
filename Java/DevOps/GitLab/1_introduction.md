# 1. 什么是 GitLab

GitLab 是一个基于 Web 的**DevOps 平台**，以 Git 仓库为起点，把源码托管、代码评审、CI/CD、制品管理、安全扫描、项目管理直到监控反馈整合在同一个应用里。它的核心主张是 **Single Application for the entire DevOps lifecycle**：团队不需要把 Jira + GitHub + Jenkins + Nexus + 一堆扫描工具拼起来，一个 GitLab 覆盖大部分环节。

## 1.1 open core：一套代码，两条产品线

GitLab Inc. 创立于 2014 年，项目遵循 **open core** 模式：

- **GitLab Community Edition（CE）**：开源（MIT 协议），任何人可以自己部署，功能已覆盖源码托管 + CI/CD + 基础项目管理。
- **GitLab Enterprise Edition（EE）**：在 CE 之上叠加企业功能（审批、安全中心、合规等）。EE 是一个代码库、一个安装介质，**付费功能靠订阅许可证解锁**——不是"装两个包"。

一个容易踩的命名坑：**CE ≠ Free，EE ≠ 全付费**。CE 实例本来就没有付费层；EE 实例不挂许可证时就是一个功能更全的免费实例（Free 层）。谈"免费版能干什么"时，主语是订阅层级（Free/Premium/Ultimate），载体是发行版（CE/EE），两条轴别混。

```text
实例 A ─ 发行版 CE  ─ 无订阅概念          → 开源核心功能，永远如此
实例 B ─ 发行版 EE  ─ 未挂许可证（Free）  → 免费层，可付费解锁 Premium/Ultimate
实例 C ─ 发行版 EE  ─ 挂了 Ultimate 试用   → 同一介质，功能面立刻变宽
```

## 1.2 三种获取形态

| 形态 | 谁运维 | 说明 |
|------|--------|------|
| GitLab.com（SaaS） | GitLab 官方 | 注册即用；限额见第 4 篇（Free 仓库 10 GiB） |
| GitLab Self-Managed | 你自己 | CE/EE 包、Docker、Helm Chart 三种主流载体（第 2 篇） |
| GitLab Dedicated | GitLab 官方托管你的专属实例 | 介于 SaaS 与自建之间，合规敏感团队的选择 |

核心定位：**以 Git 为中心的一体化协作与交付平台**，而不只是"另一个 GitHub"。

# 2. 为什么需要 GitLab

先看没有一体化平台时的典型痛点：

| 痛点 | 描述 | GitLab 的解法 |
|------|------|--------------|
| 工具链拼装 | 代码托管、CI、制品库、缺陷跟踪各自一套账号与集成 | 一个平台、一个账号体系、数据天然互通 |
| 代码与流水线割裂 | Jenkinsfile 放在仓库里但 CI 配置游离在平台外 | `.gitlab-ci.yml` 与代码同仓、同版本、同评审 |
| 评审流程难约束 | 合并与否靠口头约定 | 保护分支 + MR 审批 + CI 通过才能合并 |
| 权限模型复杂 | 仓库、构建、部署三套权限各自维护 | Group/Project 统一角色体系贯穿始终 |
| 安全滞后 | 上线后被安全团队告知有漏洞 | SAST/依赖扫描/密钥检测内嵌在 MR 与流水线里 |

## 2.1 "一体化"省掉的到底是什么

不是许可证费用，而是**集成与对账成本**：跨系统的账号同步、Webhook 胶水、"CI 说测试过了但安全平台另有口径"的扯皮、审计时四套系统的导出拼表。GitLab 的交换条件是：单实例很重（第 2 篇的硬件基线）、升级要按规矩来（季度停点，第 13 篇）、深度定制不如拼装灵活。

## 2.2 对 Java 团队最关键的一点

**CI Runner 与仓库同平台**：提交即触发流水线、MR 页面直接看测试结果和安全报告、`mvn deploy` 直推内置 Maven 仓库、镜像推进内置 Registry——反馈闭环不跨系统。Jenkins 时代"插件拼出来的闭环"在这里是默认路径。最小说明一体化手感——一个文件、与代码同一个 commit 就能跑：

```yaml
# .gitlab-ci.yml：Java 项目最小流水线
build-test:
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn -B verify
  artifacts:
    paths: [target/*.jar]
```

## 2.3 什么时候 GitLab 不是最优解

诚实的边界：开源项目要最大化社区曝光（GitHub 网络效应无可替代）；团队极小且只用 Git 托管（轻量方案更省）；已有深度绑定某云的托管 DevOps（Azure DevOps/GitHub Enterprise 顺势）；只想要一个极轻的 Git 服务器（Gitea 类足够）。选型看主矛盾：**交付链路是否值得用一个平台换掉四五个系统的集成成本**。

# 3. 与 GitHub、Gitee 的对比

| 维度 | GitLab | GitHub | Gitee |
|------|--------|--------|-------|
| 部署形态 | 自托管为主，也有官方 SaaS | SaaS 为主，企业版可私有化 | SaaS 为主，企业版私有化 |
| CI/CD | GitLab CI（内置，一个 yml 起步） | Actions（市场生态庞大） | Go 语言自研 CI |
| 定位 | 一体化 DevOps 平台 | 开源社交 + 开发者平台 | 国内代码托管 + 协同 |
| 权限与管控 | 细粒度角色、IP 白名单、合规框架（EE） | 组织角色 + Enterprise 功能 | 企业版权限管控 |
| 制品管理 | 内置 Package Registry（Maven/npm/PyPI/Docker 等） | Packages + Registry | 企业版制品库 |
| 典型选型动机 | 数据必须留在自己机房；想要全家桶 | 开源项目曝光；Actions 生态 | 国内网络与合规要求 |

## 3.1 和 GitHub 的心智差异浓缩成三条

1. **CI 是"跑在你选的 Runner 上的 Job"**：GitLab CI 的默认画像是自建 Runner + 容器执行，算力自己管；Actions 默认吃 SaaS 分钟数 + 市场复用。迁移思维时"谁提供算力"要先换算。
2. **MR 是一等公民**：讨论、审批规则、流水线状态、安全报告全部收在同一个合并请求页上，门禁配置成组出现（第 4、5 篇）。
3. **自托管是默认假设**：备份、升级、监控、HA 全部有官方运维路径（第 13、14 篇）——这是很多国内团队选它的根本原因。

> **行业实践**：国内对代码资产敏感的中大型研发团队（金融、政企、自研中间件公司）普遍自建 GitLab；开源项目与追求生态集成的团队更多选 GitHub。两者不互斥，常见组合是"内部 GitLab + 开源镜像 GitHub"——镜像用 Push 镜像维持（第 4 篇第 9 节，Free 档可用）。

```text
Jenkins 时代                      GitLab 里
──────────────────────────────────────────────────────────
Jenkins UI 上建 job、贴脚本  →    仓库根 .gitlab-ci.yml，改动走 MR
公司自建 slave，容量自管     →    同样自建 Runner——这一条没变
凭证存 Jenkins Credentials   →    CI/CD Variables（Masked/Protected）+ 令牌体系
```

# 4. 版本与授权模型

GitLab 每个订阅层级能用的功能不同，选型时容易踩坑，先建立整体印象。**下表口径与本系列第 12 篇逐项核实过的 Tier 标注一致**；单个功能以官方文档每页顶部的 `Tier:` 横幅为准。

| 层级 | 费用 | 代表性能力（已在本系列核实） |
|------|------|------------------------------|
| Free | 0 | 托管、MR、CI/CD、基本 Issue/Board、Container Registry、**全部三种合并方法**、SAST/密钥检测/容器扫描/IaC 扫描 |
| Premium | 按用户/年 | MR 审批规则、Code Owner 强制审批、群组级推送规则下发、Pull 镜像、Merge Train、Value Stream、部署看板 |
| Ultimate | 更贵 | 安全中心/漏洞管理、**依赖扫描与推送时密钥拦截**、DAST、合规框架、审计事件导出、第三方状态检查 |
| Dedicated | 商谈 | 官方代运维的专属实例，功能对齐 Self-Managed 订阅层级 |

## 4.1 三个反直觉的点

- **"半包含合并是付费的"是旧口径**：Merge commit / Semi-linear / Fast-forward 三种合并方法现在全档位可用（第 4 篇已核实）；还在这么写的教程要整体怀疑其年代。
- **扫描不是一刀切免费**：SAST、Secret Detection、Container Scanning、IaC 扫描在 Free 可跑；**Dependency Scanning 和 Secret Push Protection 仍是 Ultimate**（第 12 篇口径）。"免费层可跑全套扫描"的说法不准确。
- **付费点集中在"治理与安全"**：审批、合规、安全中心、审计导出。核心交付流程（托管→评审→CI→制品→部署）Free 层是完整的——这是它和"基础版砍到没法用"的商业软件最大的差别。

## 4.2 计费与席位的坑

席位按"计费用户"算，**Auditor 这类特殊 User type 也占席位**（第 12 篇）；群组 IP 限制、IP 白名单类策略对某些令牌类型会直接返回 404 而不是 403（同样是第 12 篇核实过的行为）。评估成本时把"只读审计账号要不要钱"这种问题提前问清。

## 4.3 怎么确认"我的实例里这功能有没有"

三层验证，优先级从高到低：

1. 打开该功能的官方文档页，看 `Tier: Free, Premium, Ultimate` 横幅与 `Offering`（GitLab.com / Self-Managed / Dedicated 可能不一致）；
2. 在自己的实例里实测——UI 入口在不在只是表象，**API/CI 展开结果才算数**：

```bash
# 读回实例的许可证与自身能力口径（字段以实际返回为准，先管道给 jq 看结构）
glab api license | jq
glab api current | jq '.id, .is_admin'

# 看 include 合并后的真实流水线（第 15/16 篇同款纪律）
glab api --method POST "projects/$P/ci/lint" -f "content=$(cat .gitlab-ci.yml)" | jq -r '.merged_yaml'
```

```yaml
# 更硬的证据：把扫描模板 include 进来跑一次，Free 实例上展开并执行成功才是"有"
include:
  - template: Security/Secret-Detection.gitlab-ci.yml
```

3. 本系列各篇的 Tier 表（批次 1-4 逐项核实过，但版本会漂移，长期以官方为准）。

# 5. 产品全景：一个平台里有什么

以使用者视角，GitLab 左侧导航即是它的功能分区：

```text
┌────────────────────────────────────────────────────────┐
│                     GitLab 功能全景                     │
├──────────────┬─────────────────────────────────────────┤
│  Plan        │ Issue、Board、Milestone、Wiki、路线图    │
│  Code        │ 仓库、Web IDE、分支保护、MR 与代码评审    │
│  CI/CD       │ Pipeline、Runner、Environments、部署看板  │
│  Secure      │ SAST / DAST / Secret / 依赖扫描、漏洞报告│
│  Release     │ 版本、Release 资产、软件包仓库、Container Registry │
│  Configure   │ CI 分析、用量统计、成员与权限、集成       │
└──────────────┴─────────────────────────────────────────┘
```

## 5.1 各分区对应的深度笔记

| 分区 | 本目录后续笔记 |
|------|---------------|
| 安装与架构 | 2、3 |
| 仓库与协作 | 4、5、11 |
| CI/CD | 6、7、8、9 |
| 制品 | 10 |
| 安全与治理 | 12 |
| 管理与观测 | 13、14、16 |
| 实践与总结 | 15、17 |

## 5.2 推荐阅读顺序 ≠ 分区顺序

分区是"功能地图"，学习是"任务链"。第 8 节给出的学习路径按"先跑起来→再协作→再交付→后治理"排，两套顺序不要互相纠正——地图用来查，路线用来走。

# 6. 核心概念速览

后续章节全部围绕这几个对象展开，先一次记住它们的关系：

- **Group（群组）**：项目的容器，可嵌套子群组；权限、CI 模板、安全策略都挂在群组上。相当于"部门/团队"。
- **Project（项目）**：一个仓库及其 Issue、流水线、Wiki 的集合，是权限与协作的最小单位（磁盘上最多三个仓库：主仓 + wiki + design，第 4 篇）。
- **Repository（仓库）**：项目里的 Git 数据本体，存储由 Gitaly 服务负责，磁盘路径是哈希不是名字（第 3 篇）。
- **Merge Request（MR）**：分支合并请求 + 代码评审载体，合并前要过讨论、审批、流水线三道关。
- **Pipeline（流水线）**：`.gitlab-ci.yml` 定义的一次 CI/CD 执行，由 **Job**（最小执行单元）组成。
- **Stage（阶段）**：Job 的逻辑分组（build → test → deploy），阶段间串行、阶段内并行。
- **Runner（执行器）**：真正执行 Job 的独立进程/容器，注册到 GitLab 后按标签接活。
- **Environment（环境）**：部署目标（staging/production…），关联部署历史与动态域名。

## 6.1 对象关系一图流

```text
Group ── 拥有 ──► Project ── 包含 ──► Repository / Issues / Pipeline
                                        │
                          .gitlab-ci.yml│ 定义
                                        ▼
              Pipeline = Stage1[Job, Job] → Stage2[Job] → ...
                                        │
                              由 Runner  │ 执行（按 tag 匹配）
                                        ▼
                 artifacts / packages / image → 推入内置 Registry
```

## 6.2 三组最容易混的概念对

| 概念对 | 一句话区分 | 展开在哪 |
|--------|-----------|----------|
| artifacts vs cache | 前者是流水线数据通道、可靠；后者是缓存、随时可 miss | 第 6/7 篇 |
| PAT vs Project/Group Token vs Job Token | 权限随人 / 随项目 / 随单次构建，自动化选型第一题 | 第 12 篇 |
| 保护分支 vs Push Rules | "谁能推谁能合" vs "推上来的内容长什么样"，服务端两道闸 | 第 4 篇 |

后两组先各记一条能落地的最小事实，细节到对应篇展开：

```bash
# Push Rules 现场报错原文：保护的是"谁能推"，内容底线靠推送规则
$ git push origin main
remote: GitLab: You are not allowed to push code to protected branches on this project.
```

# 7. 发展历史

| 时间 | 事件 |
|------|------|
| 2011 | Dmitriy Zaporozhets 与 Sid Sijbrandij 发起 GitLab 项目（Ruby on Rails），最初作为 GitHub 的自托管替代 |
| 2014 | GitLab Inc. 成立，开创远程全员协作公司文化 |
| 2016 | 8.x 整合 GitLab CI，"一体化"路线成形 |
| 2017 | 9.0 明确"单一应用覆盖整个 DevOps 生命周期"战略 |
| 2018 | 11.x 引入 Auto DevOps |
| 2020-2021 | 13.x/14.x：包仓库全家桶扩张、规划与安全功能整合进主界面 |
| 2023 | 15.10/16.x：**Runner 认证令牌机制**铺开（registration token 于 17.0 移除） |
| 2024 | 17.x：CI 元数据向独立数据库拆分起步；升级节奏规则化（17.5 起固定季度停靠点 x.2/x.5/x.8/x.11） |
| 2025 | 18.x：Duo AI 功能进入默认包并持续加码；多 ref 仓库备份从小时级压到分钟级（18.0）；Valkey 成为 Redis 的官方替代 |
| 2026 | 19.x：项目/群组转移全面异步化（19.4 GA）；组件清单加入 AI Gateway、Duo Workflow Service 等新成员；PostgreSQL 支持矩阵推进到 17.x/18.x |

历史表的正确用法不是背年份，而是看方向：**平台边界一直在向外吃（CI→制品→安全→AI），但"配置与代码同仓、决策在 Rails、字节在 Go"这两个基本盘从未变过。**

# 8. 学习路径建议

1. 先跑起来：**2 安装与配置**——本地 Docker 起一个 GitLab CE，有体感再谈概念。
2. 理解对象模型：**3 架构、4 仓库与分支**——Group/Project/分支保护怎么配。
3. 掌握协作闭环：**5 MR 与代码评审、11 Issue 与看板**——日常每天都在用的东西。
4. 吃透交付主干：**6/7 CI 两篇 + 8 Runner + 9 环境与部署 + 10 镜像仓库**——这是 GitLab 相对 GitHub 最大的差异化。
5. 进阶治理：**12 权限与安全、13 管理、14 监控**——从"会用"到"能带团队用好"。

## 8.1 按角色的最短路径

| 角色 | 必读 | 选读 |
|------|------|------|
| 普通开发者 | 4、5、6 | 11、12（令牌部分） |
| CI 负责人 | 6、7、8、10 | 9、15 |
| 平台管理员 | 2、3、13、14、16 | 12 |
| 安全/合规 | 12 | 10、13（审计与备份部分） |

## 8.2 和"15 最佳实践、16 排错、17 面试"三篇的关系

这三篇不提供新概念，是**前 14 篇的组织方式变换**：15 按工程主题重组、16 按故障现象索引、17 按问答压缩。遇到具体问题先进 16 定位现象，再回对应编号篇看机制；面试前用 17 自检，答不出的条目按篇号回补。

全系列共用的一套纪律，第一天就养成：

```text
不背 UI 路径与字段名 → 先 glab api ... | jq 读回现值
不抄旧教程的档位结论 → 看文档页 Tier 横幅 + 自己实例实测
改配置前先备份       → gitlab-backup create 且单独备份 /etc/gitlab
```

# 9. 小结

| 要点 | 一句话 |
|------|--------|
| 本质 | 以 Git 为中心的一体化 DevOps 平台（open core：CE 免费 + EE 增值） |
| 两条轴 | 发行版（CE/EE）与订阅层（Free/Premium/Ultimate）互相独立，别混 |
| 差异化 | CI/CD、制品、安全扫描全部内置，配置与代码同仓 |
| 选型 | 数据自持、流程治理诉求强 → GitLab；生态与开源曝光 → GitHub |
| 核心对象 | Group → Project → Repository/MR/Pipeline → Runner/Environment |
| Free 层 | 已可完成托管→评审→CI→制品→部署全流程；依赖扫描与推送密钥拦截仍是 Ultimate |

自检清单：

- [ ] 能向同事讲清"EE 实例不挂许可证 = Free 层"这条命名陷阱？
- [ ] 知道扫描能力里哪些 Free、哪些 Ultimate（对照第 12 篇）？
- [ ] 能画出 Group→Project→Pipeline→Runner 的对象关系并说出自建 Runner 的含义？
- [ ] 明确了自己角色对应的最短阅读路径？

下一篇进入安装：用最少的步骤把一个可用的 GitLab 跑起来，并把管理员初始化的几个必做项处理干净。
