# 1. 什么是 GitLab

GitLab 是一个基于 Web 的**DevOps 平台**，以 Git 仓库为起点，把源码托管、代码评审、CI/CD、制品管理、安全扫描、项目管理直到监控反馈整合在同一个应用里。它的核心主张是 **Single Application for the entire DevOps lifecycle**：团队不需要把 Jira + GitHub + Jenkins + Nexus + 一堆扫描工具拼起来，一个 GitLab 覆盖大部分环节。

GitLab Inc. 创立于 2014 年，项目本身遵循 **open core** 模式：

- **GitLab Community Edition（CE）**：开源（MIT 协议），任何人可以自己部署，功能已经覆盖源码托管 + CI/CD + 基础项目管理。
- **GitLab Enterprise Edition（EE）**：在 CE 之上叠加高级功能（多应用、审批规则、安全中心、合规等），部分 EE 功能需要付费订阅（Premium/Ultimate）。

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

对 Java 团队尤其关键的一点：**CI Runner 与仓库同平台**。提交即触发流水线、MR 页面直接看测试结果和安全报告，反馈闭环不跨系统。

# 3. 与 GitHub、Gitee 的对比

| 维度 | GitLab | GitHub | Gitee |
|------|--------|--------|-------|
| 部署形态 | 自托管为主，也有官方 SaaS | SaaS 为主，企业版可私有化 | SaaS 为主，企业版私有化 |
| CI/CD | GitLab CI（内置，一个 yml 起步） | Actions（市场生态庞大） | Go 语言自研 CI |
| 定位 | 一体化 DevOps 平台 | 开源社交 + 开发者平台 | 国内代码托管 + 协同 |
| 权限与管控 | 细粒度角色、IP 白名单、合规框架（EE） | 组织角色 + Enterprise 功能 | 企业版权限管控 |
| 制品管理 | 内置 Package Registry（Maven/npm/PyPI/Docker 等） | Packages + Registry | 企业版制品库 |
| 典型选型动机 | 数据必须留在自己机房；想要全家桶 | 开源项目曝光；Actions 生态 | 国内网络与合规要求 |

> **行业实践**：国内对代码资产敏感的中大型研发团队（金融、政企、自研中间件公司）普遍自建 GitLab；开源项目与追求生态集成的团队更多选 GitHub。两者不互斥，常见组合是"内部 GitLab + 开源镜像 GitHub"。

# 4. 版本与授权模型

GitLab 每个版本层级能用的功能不同，选型时容易踩坑，先建立整体印象：

| 层级 | 费用 | 代表性能力 |
|------|------|-----------|
| Free | 0 | 托管、MR、CI/CD、基本 Issue/Board、Container Registry、SAST/密钥检测（17 起下放） |
| Premium | 按用户/年 | MR 审批规则、Code Owner 强制、部署看板、需求管理（Epic）、价值流分析、半包含合并 |
| Ultimate | 更贵 | 安全中心/漏洞管理、合规框架、DAST、单元覆盖率审计、审计事件流 |
| Self-Managed / SaaS | 同上两级 | 同一套功能按"托管方式"再分：自建（Self-Managed）、GitLab.com、 Dedicated |

**设计考量**：

- **Free 层已经够用**：中小团队用 CE Free + 自建 Runner，能完成"托管→评审→CI→制品→部署"全流程，这是 GitLab 和很多"基础版砍到你没法用"的商业软件最大的差别。
- **付费点集中在治理**：审批、合规、安全中心这些"管理诉求"才需要 Premium/Ultimate，评估成本时按治理需求付费，不是按功能尝鲜付费。

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

各分区对应的深度笔记：

| 分区 | 本目录后续笔记 |
|------|---------------|
| 安装与架构 | 2、3 |
| 仓库与协作 | 4、5、11 |
| CI/CD | 6、7、8、9 |
| 制品 | 10 |
| 安全与治理 | 12 |
| 管理与观测 | 13、14、16 |

# 6. 核心概念速览

后续章节全部围绕这几个对象展开，先一次记住它们的关系：

- **Group（群组）**：项目的容器，可嵌套子群组；权限、CI 模板、安全策略都挂在群组上。相当于"部门/团队"。
- **Project（项目）**：一个仓库及其 Issue、流水线、Wiki 的集合，是权限与协作的最小单位。
- **Repository（仓库）**：项目里的 Git 数据本体，存储由 Gitaly 服务负责。
- **Merge Request（MR）**：分支合并请求 + 代码评审载体，合并前要过讨论、审批、流水线三道关。
- **Pipeline（流水线）**：`.gitlab-ci.yml` 定义的一次 CI/CD 执行，由 **Job**（最小执行单元）组成。
- **Stage（阶段）**：Job 的逻辑分组（build → test → deploy），阶段间串行、阶段内并行。
- **Runner（执行器）**：真正执行 Job 的独立进程/容器，注册到 GitLab 后按标签接活。
- **Environment（环境）**：部署目标（staging/production…），关联部署历史与动态域名。

它们的关系可以一图记住：

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

# 7. 发展历史

| 时间 | 事件 |
|------|------|
| 2011 | Dmitriy Zaporozhets 与 Sid Sijbrandij 发起 GitLab 项目（Ruby on Rails），最初作为 GitHub 的自托管替代 |
| 2014 | GitLab Inc. 成立，开创远程全员协作公司文化 |
| 2016 | GitLab 9.0 提出"单一应用覆盖整个 DevOps 生命周期"战略 |
| 2018 | GitLab 11.x 强化 CI、引入 Auto Devops |
| 2020 | GitLab 13.x：包仓库全家桶、安全扫描向 Free/Premium 下放 |
| 2021 | GitLab 14.x：Value Stream、安全候选与依赖扫描全面整合 |
| 2023 | GitLab 16.x：**Runner 认证令牌机制**（替代 registration token）、Gitaly 集群化 |
| 2024 | GitLab 17.x：Plan 免费层、安全扫描进一步免费化（DAST API 等） |
| 2025 | GitLab 18.x：Duo AI 助手进入付费默认包，性能与扩展性持续优化 |

# 8. 学习路径建议

1. 先跑起来：**2 安装与配置**——本地 Docker 起一个 GitLab CE，有体感再谈概念。
2. 理解对象模型：**3 架构、4 仓库与分支**——Group/Project/分支保护怎么配。
3. 掌握协作闭环：**5 MR 与代码评审、11 Issue 与看板**——日常每天都在用的东西。
4. 吃透交付主干：**6/7 CI 两篇 + 8 Runner + 9 环境与部署 + 10 镜像仓库**——这是 GitLab 相对 GitHub 最大的差异化。
5. 进阶治理：**12 权限与安全、13 管理、14 监控**——从"会用"到"能带团队用好"。

# 9. 小结

| 要点 | 一句话 |
|------|--------|
| 本质 | 以 Git 为中心的一体化 DevOps 平台（open core：CE 免费 + EE 增值） |
| 差异化 | CI/CD、制品、安全扫描全部内置，配置与代码同仓 |
| 选型 | 数据自持、流程治理诉求强 → GitLab；生态与开源曝光 → GitHub |
| 核心对象 | Group → Project → Repository/MR/Pipeline → Runner/Environment |
| Free 层 | 已可完成托管→评审→CI→制品→部署全流程 |

下一篇进入安装：用最少的步骤把一个可用的 GitLab 跑起来，并把管理员初始化的几个必做项处理干净。
