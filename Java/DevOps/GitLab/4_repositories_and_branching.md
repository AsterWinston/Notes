# 1. Group 与 Project：先想清楚组织结构

GitLab 的一切权限、模板、继承策略都挂在 Group 上，建项目之前先把组织结构想好，事后迁移很痛苦。

- **Group**：顶层容器，可无限嵌套（subgroup）。对应公司里的"部门/业务线/产品线"。
- **Project**：权限与协作的最小单位，一个项目 = 一个仓库 + Issues + 流水线 + Wiki + 包仓库。
- **Namespace**：项目 URL 的归属前缀，个人 namespace 或 group namespace。

**结构设计的两条常见路线**：

| 路线 | 结构示例 | 适合 |
|------|---------|------|
| 按组织 | `backend/order-service`、`frontend/portal` | 矩阵组织、权限跟部门走 |
| 按产品 | `commerce/checkout-api`、`commerce/web`（跨部门共用） | 产品制团队、想按产品聚合统计与安全策略 |

> **行业实践**：group 嵌套层级官方建议**不超过 5 层**（实际体验 2-3 层最舒服）；群组级 CI 模板、安全策略、IP 白名单都向下继承，层级越深继承规则越难推理。

# 2. 创建项目与可见性

三种建仓方式：Web 新建、从模板新建（**GitLab 自带 CI/安全模板，团队统一脚手架首选**）、导入已有仓库（GitHub/Bitbucket/GitLab 自身）。

可见性级别决定"不登录/无权限的人能不能看到"：

| 级别 | 谁能看 | 谁能克隆 | 风险 |
|------|--------|---------|------|
| Private（默认推荐） | 项目成员 | 成员 | 无 |
| Internal | 全体登录用户 | 全体登录用户 | 适合内部开源，但仍需账号 |
| Public | 任何人 | 任何人 | 自建实例慎用，代码完全裸奔 |

**设计考量**：

- 默认值在 Admin → General → Visibility 可以收紧（建议把实例默认改为 Private），避免新人误建 Internal 项目。
- Internal 不等于安全：任何登录用户都能下载全部源码，敏感业务必须 Private。

# 3. 分支模型选型

GitLab 对分支模型没有强制立场，但它的 MR/流水线机制天然适配几种主流模型：

| 模型 | 分支 | 适合 | GitLab 配套特性 |
|------|------|------|----------------|
| Git Flow | develop + feature/* + release/* + hotfix/* | 版本化交付（客户端、私有化软件） | 环境分支部署、Release 里程碑 |
| GitLab Flow | main + 环境分支（pre/prod） | 半持续交付，需逐环境验证 | Environments 按分支映射 |
| Trunk-Based | 单一 main，短命分支或特性开关 | 高频部署的互联网/SaaS | MR 快速合并、Auto DevOps、半包含合并 |
| 分支即发布（Release Branch） | release/17.x | 长期维护多版本（SDK、中间件） | 每分支独立流水线 + 制品仓库 |

> **注意**：不要"模型原教旨"。Java 团队最常见的落地是**主干 + feature 分支 + MR 合并**，版本发布时再拉 `release/x.y`；Git Flow 的 `develop` 分支在 MR 工作流里往往只增加摩擦。

# 4. 保护分支

保护分支（Protected Branches）回答两个问题：**谁能推、谁能合**。在 Settings → Repository → Protected branches 配置：

| 权限 | 可选项 | 典型配置 |
|------|--------|---------|
| 允许 force push / 删除 | Maintainers / No one | main 设 No one |
| 允许合并 | Maintainers / +Developer+ | 配合审批规则时给 Developer |
| 允许指定模式 | 通配 `release/*` | 一次保护一批分支 |

保护分支带来的实际约束：

- 非授权用户无法直接 `git push` 到 main（push 被拒绝），改动只能走 MR。
- 流水线失败时无法合并（配合下面两条）。
- 合并 MR 时要求分支最新（防止"合并后挂"）。

# 5. Merge 设置：把流程焊死

Settings → Merge requests 里几个关键开关，构成"质量门禁"：

| 设置 | 作用 | 建议 |
|------|------|------|
| **Status checks → Pipelines must succeed** | CI 不过不能合 | 开，这是 MR 流程的核心价值 |
| **Merge method: Fast-forward merge** | 强制线性历史，feature 分支必须基于最新 main | 搭配主干开发时开 |
| **半包含合并（Semi-linear，Premium）** | 不强求先 rebase，合并时自动 FF | 团队大、rebase 冲突频繁时的折中 |
| **压缩合并（Squash）** | 一个 MR 一个 commit | 默认推荐，保持 main 历史干净 |
| **Delete source branch** | 合并后删源分支 | 默认勾选，控制分支数量 |
| **审批（Premium）** | 需要 N 人 approve | 关键仓库设 2 人 |

# 6. Push Rules：仓库级硬性守门员

Push Rules 直接写在仓库配置里（Premium 可在群组级下发），在服务器端拦截 push，先于 CI 生效：

```bash
# 示例：拒绝未签名 commit、拒绝含密钥的推送
git config --add pushrules.commitmesregex '^\[JIRA-123\] '   # 提交信息格式
git config --add pushrules.rejectunsignedcommits true        # 拒绝未 GPG 签名
git config --add pushrules.rejectsecrets true                 # 内置密钥模式拦截
git config --add pushrules.rejectsecretlinks true             # 拒绝硬编码的 GitLab 链接 token
```

与 GitLab CI 的 pre-commit/扫描区别：**Push Rules 是服务端拦截，客户端绕不过**（`--no-verify` 无效）；CI 扫描晚了一步。二者互补。

# 7. 默认分支、归档与转移

- **默认分支**（Default branch）：MR 的目标、CI 的 `main` 触发源、Release 的挂载点。改名（master→main）在项目设置里一键完成，注意同步流水线里写死的分支名。
- **归档（Archive）**：只读冻结，不能 push/建 MR，但可克隆、可查 Issue——项目下线但不删时的正确姿势。
- **转移（Transfer）**：把项目挪到另一个 namespace，**路径会变**，旧路径 410 前 GitLab 有重定向缓冲期；转移会带走 Issue/MR/流水线，但不带走 Fork 关系（Fork 会脱离），谨慎操作。

# 8. Fork 与上游同步

开源协作与"给公司内部其他团队提 MR"都用 Fork：

```bash
git remote add upstream https://gitlab.example.com/org/lib.git
git fetch upstream
git merge upstream/main        # 或 rebase，保持与上游同步
```

设置里可控制 **Fork 可见性是否跟随源项目**、是否允许 Fork（合规仓库常禁）。

# 9. 代码所有者 Code Owner（Premium）

在仓库根放 `CODEOWNERS`（`.gitlab/CODEOWNERS` 或根目录），按路径规则指定审批人：

```gitignore
# CODEOWNERS 示例：路径规则 + 负责人
/docs/          @tech-writer
*.java          @backend-lead
/src/main/resources/application-*.yml  @ops-team @security
```

开启"Code Owner approval required"后，涉及这些路径的 MR **必须有对应 Owner approve** 才能合并——把"谁对哪块代码负责"从口头约定变成机器门禁。

# 10. Web IDE 与日常浏览

- **Web IDE**：按 `.` 键随时进入，改完直接建分支提 MR，适合小修小补；大改动仍建议本地 IDE。
- 文件 **Blame/History**：每个文件页都能看行级作者与历史，`/blob/main/pom.xml` URL 可直接分享定位到行（`#L12-20` 锚点）。
- 搜索：全局搜索支持代码、MR、Issue、Wiki；仓库内按 `t` 唤出文件跳转，`/` 唤出搜索。

# 11. 小结

| 要点 | 一句话 |
|------|--------|
| 组织结构 | 权限与策略靠 Group 承载，层级宜浅 |
| 可见性 | 实例默认 Private，Internal 不是安全 |
| 分支模型 | 主干 + feature + MR 是大多数团队的最优解，模型是手段不是信仰 |
| 门禁组合 | 保护分支（谁能写）+ Merge 设置（怎么合）+ Push Rules（服务端拦截）+ CODEOWNERS（谁批） |
| 生命周期 | 归档只读、转移改路径，动手前想清楚 |

协作流程的另一半——MR 评审的细节，在下一篇展开。
