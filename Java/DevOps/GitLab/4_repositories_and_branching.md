# 1. Group、Project 与 Namespace：先把组织结构想清楚

GitLab 的一切权限、CI 模板、安全策略、IP 白名单都挂在 Group 上向下继承；项目一旦建出来，仓库路径、包命名（尤其 npm 的 root namespace 约定）、Fork 网络、令牌作用域就全部锚定在 namespace 上。**事前设计结构的成本远低于事后迁移**——本篇就是"仓库协作"这条线（第 4、5、11 篇）的地基：先把容器和规则想清楚，再谈 MR 流程与合并策略。

## 1.1 三个容器：Group / Project / Namespace

- **Group**：顶层容器，可无限嵌套（subgroup）。对应公司里的"部门/业务线/产品线"。群组承载的东西比直觉多：CI 模板与群组级变量、安全策略与扫描配置的继承、IP 白名单、推送规则下发（Premium）、默认分支保护档位、成员与邀请。
- **Project**：权限与协作的最小单位。一个项目 = 主仓库 +（可选）`-wiki` 仓库 +（可选）`-design` 仓库——**每个项目最多 3 个 Git 仓库**（第 13 篇备份节核实过：备份会按仓库打包，wiki/design 各占一份）；再加上 Issues/MR、流水线、包仓库、环境、容器镜像。
- **Namespace**：项目 URL 的归属前缀，个人 namespace 或 group namespace。包仓库坐标、Pages 域名、镜像配置里的路径都跟着它走，改名/转移的连锁影响都从这里出发。

## 1.2 结构设计的两条常见路线

| 路线 | 结构示例 | 适合 |
|------|---------|------|
| 按组织 | `backend/order-service`、`frontend/portal` | 矩阵组织、权限跟部门走 |
| 按产品 | `commerce/checkout-api`、`commerce/web`（跨部门共用） | 产品制团队、想按产品聚合统计与安全策略 |

两条路线混用最痛苦：安全策略、CI 模板、群组级审批都按"树"下发，团队按矩阵切分时，同一个项目会期望继承两套互相冲突的父级策略，最后只能靠项目级替换硬压，规则变得不可推理。

## 1.3 官方推荐边界：没有硬限制，但有明确症状

GitLab 对 group 的成员数、层级深度、子群数量**都没有强制上限**（官方原话：no enforced limits），但文档给出了推荐值和越界后的典型症状，值得抄进内部规范：

| 结构 | 推荐值 | 硬上限 | 越界后的典型症状（官方口径） |
|------|--------|--------|------------------------------|
| 每组成员数 | < 1,000 | 无 | 性能劣化；≥ 1,500 时部分操作超时或返回 `500` |
| 每父群子群数 | < 100 | 无 | 导航与权限计算变慢 |
| 层级深度 | ≤ 5 层 | 20 层 | 数据库查询超时 |

三个要点：其一，推荐值≠最大限制，超过仍"受支持"，但性能问题**会叠加**——"深嵌套 × 多子群"比任一单项更早触发超时；其二，群组仪表盘加载慢、权限计算超时这类现象多半是结构问题而不是实例性能问题；其三，实践上 2-3 层最舒服，层级每加一层，"这个项目到底继承了谁的策略"的推理成本翻倍。

## 1.4 推理继承的纪律：读回现值，别靠记忆

群组级策略是"**新建才继承、项目侧替换**"模型：CI 模板、安全策略、推送规则、默认分支保护都是父级配置在新项目创建时落一份副本或建立继承关系，项目级设置是**替换**而不是叠加（推送规则的这条在 7.1 展开）。所以永远不要凭页面记忆推理"现在生效的是什么"，用 API 读回现值：

```bash
# 项目当前实际生效的推送规则（群组下发 ≠ 已应用到本项目，见 7.3）
glab api projects/42/push_rule | jq
# 项目的归属层级与可见性
glab api projects/42 | jq '{path_with_namespace, visibility, namespace: .namespace.full_path}'
# 群组容量构成（statistics 字段口径见第 13 篇容量节）
glab api "groups/7?statistics=true" | jq '.statistics'
```

# 2. 创建项目与可见性、功能开关

## 2.1 四种建仓方式

| 方式 | 场景 | 要点 |
|------|------|------|
| 空白新建 | 全新模块 | 顺手初始化 README——空仓库没有默认分支，很多功能（MR、Pages）无从谈起 |
| **从模板新建** | 团队统一脚手架 | GitLab 自带 CI/安全模板；企业常用群组模板库做内部脚手架，配合第 8 篇 `include` 消费 |
| 导入 | 迁移 | GitHub/GitLab 导出包/裸文件等来源以创建页实际列表为准；GitLab.com 导入包上限 5 GiB（见 11.2），超限走批量导入或先拆仓库 |
| 反向持续同步 | canonical 在别处 | Push/Pull 镜像，见第 9 节，不是一次性迁移手段 |

## 2.2 可见性级别

可见性级别决定"不登录/无权限的人能不能看到"：

| 级别 | 谁能看 | 谁能克隆 | 风险 |
|------|--------|---------|------|
| Private（默认推荐） | 项目成员 | 成员 | 无 |
| Internal | 全体登录用户 | 全体登录用户 | 适合内部开源，但仍需账号 |
| Public | 任何人 | 任何人 | 自建实例慎用，代码完全裸奔 |

项目级入口统一在 **Settings → General → Visibility, project features, permissions**；实例级默认值在 Admin Area 的 Settings → General 收紧（建议把实例默认改为 Private），避免新人误建 Internal 项目。

## 2.3 Internal ≠ 安全

Internal 只是"免去逐个授权"，任何拿到账号的人——包括外包、离职前的员工、被钓鱼的自动化账号——都能 `git clone` 全量源码和历史。历史里有什么？硬编码的旧密钥、内部接口清单、供应商信息。**敏感业务必须 Private**，并且配合第 12 篇的密钥检测把历史翻一遍，而不是只盯增量。

## 2.4 功能开关的依赖链（关一个瘫一片）

项目设置里的功能开关有明确的连带关系，官方文档直接列了依赖表：

| 关掉 | 连带不可用 |
|------|-----------|
| Work items（Issues） | Issue Boards、Service Desk（Milestone 仍可从 MR 访问） |
| Work items + Merge Requests | Labels、Milestones |
| Repository | Merge requests、CI/CD、Git LFS、Packages |

两个实践细节："只关 Issues、留 Repository"常用来把项目降级为纯代码宿主；关 Analytics 只是隐藏左侧入口，底层数据仍在计算、API 仍可读——不要用"关入口"当"停采集"来向合规团队交代。

# 3. 分支模型选型

GitLab 对分支模型没有强制立场，但它的 MR/流水线机制天然适配几种主流模型：

| 模型 | 分支 | 适合 | GitLab 配套特性 |
|------|------|------|----------------|
| Git Flow | develop + feature/* + release/* + hotfix/* | 版本化交付（客户端、私有化软件） | 环境分支部署、Release 里程碑 |
| GitLab Flow | main + 环境分支（pre/prod） | 半持续交付，需逐环境验证 | Environments 按分支映射 |
| Trunk-Based | 单一 main，短命分支或特性开关 | 高频部署的互联网/SaaS | MR 快速合并、Auto DevOps、半包含合并 |
| 分支即发布（Release Branch） | release/17.x | 长期维护多版本（SDK、中间件） | 每分支独立流水线 + 制品仓库 |

> **注意**：不要"模型原教旨"。Java 团队最常见的落地是**主干 + feature 分支 + MR 合并**，版本发布时再拉 `release/x.y`；Git Flow 的 `develop` 分支在 MR 工作流里往往只增加摩擦。

## 3.1 Java 团队落地模板：主干 + 短命分支 + release 线

一条被反复验证的组合拳：

1. `main` 常驻绿：直接推 main 关掉（保护分支，见第 5 节），一切改动走 MR。
2. `feature/JIRA-123-xxx` 短命分支，存活 ≤ 3 天，超过就说明 MR 切得太大。
3. 发布时从 main 拉 `release/x.y`（用通配保护 `release/*` 一次锁一批），只在 release 线上 cherry-pick 修复，定期把修复合回 main。
4. hotfix 视团队习惯从 main 拉、修完走 MR 双向合入。
5. 分支类型直接编码进流水线规则，避免"所有分支都跑全量"：

```yaml
# .gitlab-ci.yml：workflow 按来源裁剪触发（第 6 篇讲 rules 的完整语法）
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH =~ "^release/"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

## 3.2 分支命名先立规矩，再靠 Push Rules 兜底

口头约定活不过第一个新人。把命名写成一条正则，塞进第 7 节的 Push Rules 里让服务端强制执行：

```text
^(feature|hotfix|release|chore)/[A-Za-z0-9._/-]+$
```

两条服务端已经替你管掉的边界情况（均为官方文档明示）：**默认分支永远被命名规则放行**（规则再严也不拦 `main`）；**40 位十六进制样式的分支名被禁止创建**——这是安全设计，防止用"看起来像 commit hash"的分支名伪装引用。

# 4. 默认分支

## 4.1 默认分支承担的四件事

默认分支（Default branch）不只是一个 checkout 目标：它是 **MR 的默认目标**、**CI 预定义变量 `$CI_DEFAULT_BRANCH` 的所指**、**Release 与 Tag 的默认挂载点**、以及所有"仓库级入口设置"（默认分支保护、Pages 源分支等）的作用对象。团队统一引用主干时写 `$CI_DEFAULT_BRANCH` 而不是硬编码 `main`，改名那天就不用满仓库 grep。

## 4.2 新建仓库的默认分支保护档位

管理员和群组 Owner 可以给"每个新仓库的默认分支"统一下发初始保护级别，五档（官方列表，注意这是**新建时**的初始化策略，存量仓库仍需逐个配置或用 API 批处理）：

| 档位 | Developer 可推 | Maintainer 可推 | 可 force push | 适用 |
|------|---------------|----------------|---------------|------|
| Fully protected（默认值） | ✗ | ✓ | ✗ | 绝大多数团队 |
| Fully protected after initial push | 仅首个提交 | ✓ | ✗ | 空仓批量导入场景 |
| Protected against pushes | ✗（但可点合并按钮） | ✓ | ✗ | 严格"只走 MR" |
| Partially protected | ✓ | ✓ | ✗ | 小团队信任制 |
| Not protected | ✓ | ✓ | ✓ | 危险，仅限个人实验 |

官方给了一段分量很重的警告，值得原样贴进内部规范：**只要不是 Fully protected，恶意开发者就可能把带外带 CI/CD 变量的 `.gitlab-ci.yml` 提交到受保护分支**，等流水线一跑，群组级变量就被外带了。分支保护与 CI 变量的安全耦合关系，详见第 12 篇与第 7 篇。

## 4.3 master → main 改名清单

改名动作本身在项目的默认分支设置里完成，不同大版本入口位置略有差异，别背路径，直接在 Settings → Repository / General 页面里搜 "Default branch"。真正的工作量在改名之后：

- 流水线里硬编码 `master`/`main` 的规则、触发分支、环境映射；
- 镜像配置（第 9 节）里的分支白名单；
- Jira/CI 外部集成里写死分支名的 webhook 过滤；
- 文档、README 徽章 URL 里的分支段。

GitLab 对默认分支改名保留了引用跳转缓冲（官方有 "Default branch rename redirect" 专节），但缓冲是给你救急的，不是给你赖着不改的。排查存量项目的默认分支，GraphQL 一条批量出（REST 的 `default_branch` 字段也行）：

```bash
glab api projects/42 | jq .default_branch
```

# 5. 保护分支：谁能推、谁能合

保护分支（Protected Branches）回答两个问题：**谁能推、谁能合**。在 Settings → Repository → Protected branches 配置：

| 权限 | 可选项 | 典型配置 |
|------|--------|---------|
| 允许 force push / 删除 | Maintainers / No one | main 设 No one |
| 允许合并 | Maintainers / +Developer+ | 配合审批规则时给 Developer |
| 允许指定模式 | 通配 `release/*` | 一次保护一批分支 |

## 5.1 通配保护与匹配口径

一个仓库可以同时存在 `main`、`release/*` 这样的多条保护规则，**分支命中多条规则时取最宽松**（这条在第 5 篇协作批已经核实过口径，此处沿用）。所以设计通配时要小心"好心加一条宽规则"实际放宽了所有被它命中的分支。

## 5.2 "能合但不能推"与典型报错

"Allowed to merge" 单独给 Developer 是常态配置：Developer 不能直接 `git push`，但可以在自己的 MR 上点合并（前提是流水线/审批过了），配合 squash 时 GitLab 服务端代为完成合并动作。被挡时的标准报错长这样：

```bash
$ git push origin main
remote: GitLab: You are not allowed to push code to protected branches on this project.
```

看到这条不要先去问"是不是权限没加"，先读回现值确认规则本体：

```bash
glab api projects/42/protected_branches | jq '.[] | {name, allowed_to_push, allow_force_push}'
```

保护分支与 Merge 检查、CODEOWNERS 联动后的完整门禁效果（"谁能写"+"谁要批"+"什么条件才放行"），在第 5 篇 MR 篇统一推演。

# 6. Merge 方法与质量门禁

Settings → Merge requests 里几个关键开关，构成"质量门禁"。先纠正两个老口径：三种合并方法**现在全档位可用**（含 Free，历史上 Fast-forward 曾限付费，如果教程还写 Premium 请以自己实例的设置页为准）；squash 也不是勾选框，而是**四档策略**。

| 设置 | 作用 | 建议 |
|------|------|------|
| **Merge method** | Merge commit / Semi-linear / Fast-forward | 见 6.1 选型表 |
| **Squash commits when merging** | Do not allow / Allow / Encourage / Require | 主干开发选 Encourage 或 Require |
| **Status checks → Pipelines must succeed** | CI 不过不能合 | 开，这是 MR 流程的核心价值 |
| **All threads must be resolved** | 有未解决讨论不能合 | 开，成本低收益高 |
| **Delete source branch** | MR 页默认勾选删源分支 | 开；19.6 起实例级也能给默认值 |
| **审批（Premium）** | 需要 N 人 approve | 关键仓库设 2 人（细节第 5 篇） |

## 6.1 三种合并方法的真实差异

| 方法 | 历史形态 | 等价 git 命令 | 代价 |
|------|---------|--------------|------|
| Merge commit（默认） | 每次合并生成 merge commit | `git merge --no-ff` | 历史最真实，图最乱 |
| Semi-linear | 只在可 FF 时合并，仍留 merge commit | 先 rebase 再 `git merge --no-ff` | 需要"合并前同步"的纪律 |
| Fast-forward | 线性、无 merge commit | `--ff-only`；配 squash 时 `git merge --squash` + commit | 分支上下文丢失（squash 后只剩一条） |

注意一个常见误解：Semi-linear **不是**"不强求先 rebase 的折中"——它的语义恰恰是"合并前必须能 fast-forward"，源分支落后目标分支时页面会给你 rebase 按钮（或开 Automatic rebase before merge 让 GitLab 自动补）。它与 FF 的区别只在于留不留 merge commit。两者都能保证"MR 流水线绿 ⇒ 合并后的主干流水线大概率也绿"，Merge commit 方法则做不到这个保证，这正是"防止合并后挂"的机制学名。

## 6.2 squash 四档语义

- **Do not allow**：永不压缩，合并人无权更改；
- **Allow**：默认不压缩，可手选；
- **Encourage**：默认勾选压缩，可取消（多数团队的最优解）；
- **Require**：永远压缩成一条 commit，合并人无权更改（配 FF 即"每 MR 一个原子提交"的纯净线性史）。

要求 MR 标题最终入库形态统一的，再配 commit message 模板（`docs/merge_request_templates` 体系，见第 5 篇）。

## 6.3 FF/Semi-linear 下的日常自救

分支落后导致"不能合并"时，MR 页有按钮，命令行等价物：

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease origin feat/JIRA-123
```

`--force-with-lease` 而不是 `--force`：保护分支上你本来也 force 不动，但源分支被同事共享时，lease 能防止你覆盖别人刚推的提交。

# 7. Push Rules：服务端硬拦截

Push Rules 直接写在服务器端仓库配置里，在 `git push` 事务落地**之前**校验，先于 CI 生效。与客户端 pre-commit hook、CI 里跑扫描的关系：客户端 `--no-verify` 能绕 hook、CI 晚了一步才绿，**Push Rules 是服务端拦截，绕不过**。注意旧笔记里 `git config --add pushrules.*` 那组命令是杜撰的——Push Rules 不在 git 客户端配置里，只存在于服务器端设置（UI/API），别再抄。

## 7.1 三级来源与继承语义

| 层级 | 入口 | 档位 | 语义 |
|------|------|------|------|
| 全局 | Admin → Push rules | 管理员 | **模板**：仅配置之后新建的项目继承一份副本；已有项目不受影响，改动也不回灌 |
| 群组 | Settings → Repository → Pre-defined push rules | Premium/Ultimate | 同"新建才继承"；也可按项目逐个覆盖 |
| 项目 | Settings → Repository → Push rules | 全档 | 项目侧不是叠加，是**替换**副本 |

推论：想让存量项目吃到新规则，只有两条路——逐个进项目改，或 API 批量刷（7.3）。全局/群组配置改了什么都不发生是**正常行为**，不是 bug。

## 7.2 规则矩阵（字段名 = REST API 实名）

| 规则 | API 字段 | 说明 |
|------|----------|------|
| 提交信息必须匹配 | `commit_message_regex` | 多行模式，`(?-m)` 可关；例 `JIRA\-\d+` |
| 提交信息禁止匹配 | `commit_message_negative_regex` | 命中即拒 |
| 分支名必须匹配 | `branch_name_regex` | 默认分支永远放行；40-hex 名被禁 |
| 作者邮箱必须匹配 | `author_email_regex` | 卡"@公司域名" |
| 作者须为存量用户 | `member_check` | 提交作者邮箱必须对应已有 GitLab 账号 |
| 校验提交者身份 | `commit_committer_check` / `commit_committer_name_check` | committer 邮箱须是本人已验证邮箱 / 姓名须与账号一致 |
| 禁止文件名 | `file_name_regex` | **RE2 语法，无正/负向前瞻**；比对"目录+文件名"全路径 |
| 最大文件尺寸 | `max_file_size` | 单位 MB，0 = 不限；**Git LFS 跟踪的文件豁免** |
| 禁删 tag | `deny_delete_tag` | `git push --delete` tag 被拒 |
| 拒绝未签名提交 | `reject_unsigned_commits` | GPG/SSH 签名 |
| 拒绝无 DCO 提交 | `reject_non_dco_commits` | 要求 `Signed-off-by`（`git commit --signoff`） |
| 拦截密钥文件 | `prevent_secrets` | 内置文件名黑名单（7.4） |

## 7.3 读回与批量下发

```bash
# 读回现值（做任何批量操作之前先看一眼，别盲改）
glab api projects/42/push_rule | jq

# 创建/替换某项目的推送规则（字段名见 7.2）
glab api --method POST projects/:id/push_rule \
  -f commit_message_regex='^(feat|fix|chore|revert)(\([a-z-]+\))?: .+' \
  -f branch_name_regex='^(feature|hotfix|release|chore)/[A-Za-z0-9._/-]+$' \
  -f reject_unsigned_commits=true -f prevent_secrets=true -f max_file_size=8
```

群组级端点 `/groups/:id/push_rule`、全局级端点同样存在（项目级是 `push_rule` 单数路径，四个动词同路径）。批量给全群组项目刷规则的正规姿势：枚举群组项目 → 对每个 `POST /projects/:id/push_rule`（存在则 `PUT`），脚本化前先在 1-2 个试点项目验证读回结果。

## 7.4 密钥黑名单拦截什么

`prevent_secrets` 的命中集来自服务端 `files_denylist.yml`，典型条目：`.aws/credentials` 各变体、`id_rsa`/`id_dsa`/`id_ed25519`/`id_ecdsa`/`*_sk` 及其 `.ssh/` 路径变体、`*.pem`、`*.key`、`*.history`/`*_history`。边界感要拎清：**它只拦"新推上来的文件"，不追溯仓库里已有的历史内容**；它是文件名/路径模式匹配，不看内容——把密钥改名塞进 `app.js` 它管不了，那是第 12 篇 Secret Detection 与 Push Protection 的活（两者是不同机制、不同档位）。

## 7.5 四个真实踩坑

1. **Web IDE 被误伤**：`reject_unsigned_commits` 会拒掉部分由 Web IDE 生成的提交（官方在 "Prevent unintended consequences" 里明确点名）。开启前先和用 Web IDE 改文档的团队打招呼。
2. **UI 提交的换行是 `\r\n`**：在 GitLab 网页里创建的 commit message 用 `\r\n` 换行，按 `\n` 写正则匹配不到，要写 `JIRA:(\r\n?|\n)\w+` 这种兼容式。
3. **RE2 没有前瞻**：想表达"除了 .yml 都禁"是做不到的，只能用组合枚举 `(\.exe|^config\.yml|(^|\/)install\.exe)$` 这种"白名单反面=列全黑名单"的思路。
4. **提交末尾标点这种细活**：`[[:^punct:]]\b$` 能拒"结尾是标点"的提交——`\b` 是必须的，因为 Git 会给消息加尾部换行。这类 POSIX 字符类写法值得存进团队正则速查表。

# 8. Fork 与对象池

## 8.1 Fork 工作流

开源协作与"给公司内部其他团队提 MR"都用 Fork：

```bash
git remote add upstream https://gitlab.example.com/org/lib.git
git fetch upstream
git merge upstream/main        # 或 rebase，保持与上游同步
```

UI 侧 fork 项目页有现成的同步入口（比命令行少一步记 remote），fork 后可随时 **unlink**（切断 fork 关系变成独立仓库，常见于"fork 只是临时用，不想被上游网络绑架"）。

## 8.2 可见性约束

官方硬规则：**fork 的可见性必须不宽于"上游项目"与"fork 所在 namespace"两者中更严格的**。也就是说不可能把 Private 上游 fork 到 Public 群组里公开。上游从 Private 改 Public **不会**自动放宽你 fork 的可见性——需要自己调。合规仓库直接在设置里禁止 fork，比事后清理便宜。

## 8.3 Hashed Storage 与 Object Pool：fork 为什么不太占空间

GitLab 用 hashed storage（按 ID 哈希的磁盘路径）存放主仓 + `-wiki` + `-design` 最多三个仓库。创建**第一个 fork** 时，服务端会：

1. 新建一个 object pool 仓库，装入原仓库全部对象；
2. 用 Git 的 alternates 机制把两个仓库链接到 pool；
3. repack 原仓库，丢弃自己的对象副本、改从 pool 读取。

之后所有 fork 都链向同一个 pool，各自只存差异对象。逻辑上很优雅，代价在第 13 篇备份节已经见过：**备份打包时 pool 会被复制进每个 fork**，fork 网络的备份体积和时长会被显著放大——fork 数量大的公共库要提前把备份时长算进去，而不是等 `gitlab-backup create` 跑到半夜才发现。

## 8.4 Fork 网络的治理动作

- 定期看 **Settings → Usage quotas**：fork 的存储明细可查；
- 大 fork 网络的仓库**不能直接重写历史**（11.5），unlink 或"归档重建"是仅有的两条路；
- fork 数量与仓库创建数受实例配额约束（群组容量统计走 `statistics`，口径见第 13 篇容量节）。

# 9. 仓库镜像：持续同步不是迁移

## 9.1 方向与档位

| 方式 | 档位 | 语义 |
|------|------|------|
| Push 镜像（GitLab → 外部） | Free | canonical 在 GitLab，把旧地址（GitHub/内网 Git）当只读投影继续供血 |
| Pull 镜像（外部 → GitLab） | Premium/Ultimate | canonical 在别处，GitLab 做只读副本 |
| Bidirectional | 可用 | 官方直接警告会产生冲突，除非你有明确的双活需求，别用 |

典型场景：私有实例想开源部分项目（push 出去 + 手工回流）；迁移过渡期旧地址不能断（push 镜像）；把外部上游库吸进来统一挂 issue/CI（pull）。

## 9.2 调度节奏与"为什么迟迟不更新"

Pull 镜像由 Sidekiq cron **每分钟评估一次**排班；一次拉取成功后，**至少等 30 分钟**才会安排下一次（这是排队语义，不是"精确 30 分钟到点就拉"）。手动 force update 同样受最小间隔约束；自建实例的间隔参数由管理员配置。排障先看镜像详情页的最近错误和时间戳，而不是猜。

## 9.3 认证与同步范围

认证支持用户名/密码（不推荐）、token、SSH key；用 SSH 时**必须先校验并信任远端主机密钥**（GitLab 提供获取本机公钥和验证 host key 的入口），否则连接静默失败。同步范围可以收紧成"**只镜像保护分支**"或按分支名单过滤——给外网投影时尤其必要：内部分支名、实验分支本身就泄露信息。

# 10. CODEOWNERS：把"谁对哪块代码负责"变成机器门禁（Premium）

## 10.1 生效链条：两个条件缺一不可

CODEOWNERS 文件本身只声明归属。要让它拦住 MR，必须同时满足：**目标分支受保护** + **该保护分支开启 Code Owner 审批（Require code owner approval）**。"设置了 owner 却不拦截"的经典问题九成是这两处开关没配对。开启后可用的能力：必须 owner 批准、可要求多个 owner 批准（按规则里的人数）、可选段（`[Optional]`）不拦截只提示。

## 10.2 文件位置：第一命中制

每个仓库只认**一个** CODEOWNERS，按以下顺序查找，**第一个找到即生效，其余全部忽略**：

1. 仓库根：`./CODEOWNERS`
2. `./docs/CODEOWNERS`
3. `./.gitlab/CODEOWNERS`

（旧笔记写"根或 `.gitlab/` 二选一"不准确：三处都有时生效的是根那份。）这决定了排障习惯——"我明明写了 owner"先查是不是另一处有个更靠前的文件把整个规则集挡住了。

## 10.3 语法示例

```gitignore
# CODEOWNERS 示例：路径规则 + 负责人
/docs/          @tech-writer
*.java          @backend-lead
/src/main/resources/application-*.yml  @ops-team @security
```

## 10.4 无主文件与版本语义

GitLab 拿 **MR 目标分支上的 CODEOWNERS 版本**做比对：你刚改的 owner 规则要先进了目标分支才算数。没有命中任何规则的文件走普通审批，不强制 owner。所以安全敏感路径（密钥配置、支付模块）记得给兜底通配规则，宁可 `[Optional]` 也别漏成无主。

# 11. 仓库体积、Housekeeping 与瘦身

## 11.1 尺寸是怎么算出来的

- **Repository size**：语义约等于对 hashed storage 路径跑 `du --summarize --bytes`；仓库变化时重算，但**最多每 15 分钟一次**——刚删完大文件看数字不动是正常的。
- **Project size** ⊇ repository + LFS + job artifacts + packages + wiki。查看入口：**Settings → Usage quotas → Storage** 标签，逐行看构成。
- 自建实例的管理员限制项就叫 Repository size limit（账号与限制设置页）；GitLab.com 是预置值。

## 11.2 GitLab.com 限额速查（现行文档口径）

| 项 | 数值 |
|----|------|
| 仓库体积（含 LFS），Free | 10 GiB |
| 仓库体积（含 LFS），Premium/Ultimate | 500 GiB |
| 导入包上限 | 5 GiB |
| 导出包上限 | 40 GiB |

到限后 push、创建 MR、上传 LFS 对象会被拒。自建实例的对应值由管理员在限制设置里配，读法同第 1 篇纪律：`glab api` 读回现值，别背默认值。

## 11.3 Housekeeping：两种策略、三种触发

Gitaly 的仓库维护分两种打法：**Eager**（不管仓库状态，指定什么任务就干什么，push 触发与手动触发用它）和 **Heuristical**（检查 loose object 数量、packfile 数量、loose ref、commit-graph，缺什么补什么；对象越多重打包越勤、ref 越多反而重打包越懒——专门为大 monorepo 调的）。

触发途径三条：
1. push 数达到阈值自动排（Admin → Settings → Repository → Repository maintenance，"Optimize repository period" 就是**多少个 push 之后**让 Gitaly 优化一次）；
2. 项目管理员手动：Settings → General → Advanced → **Run housekeeping**（异步后台任务）；
3. 自建实例的定时兜底任务（下面 11.4）。

顺手一条冷知识：**未被引用的 LFS 对象每 200 次 push 清一次**——LFS 占用下不去时先别急着提工单。

## 11.4 Gitaly daily maintenance：给休眠仓库上保险

只靠 push 触发的维护会漏掉"从不接收 push、只有读流量"的休眠仓库。Gitaly 有全量轮扫的兜底，**默认每天 12:00 跑 10 分钟**，可改：

```ruby
# /etc/gitlab/gitlab.rb → gitaly['configuration']
daily_maintenance: {
  start_hour: 23,
  start_minute: 00,
  duration: '1h',
  storages: ['default'],
}
```

到 `duration` 时正在跑的任务会被优雅取消，下一轮随机洗牌仓库顺序继续。验证有没有在跑，看 Gitaly 日志这两行：

```json
{"level":"info","msg":"maintenance: daily scheduled","scheduled":"2023-09-27T13:10:00+13:00"}
{"msg":"maintenance: daily completed","actual_duration":321181874818,"max_duration":"1h0m0s","error":null}
```

`actual_duration` 是纳秒。Praefect 集群可以让各节点的起始时间错开，避免同时扫描。

## 11.5 清理与历史重写的正确顺序

官方在 repository size 文档里把"删大文件/删敏感文件"的前置坑列得很全，浓缩成操作顺序：

1. **停写**：确认没有运行中的流水线——它们会访问仓库、导致清理不彻底。
2. 触发 housekeeping；注意它的 prune 可达对象的 **grace period 是 2 周**。
3. 想马上回收空间：单独手动 **Prune unreachable objects**，grace 缩短为 **30 分钟**——Git GC 本来就只清"不可达且至少 30 分钟前"的对象，所以任何 prune 之前先等满 30 分钟，否则白跑。
4. **通知全员重 clone 或彻底同步**：留着旧 clone 的人下次 push 就会把"已删除的文件"原样复活。
5. Fork 关系的仓库**不能直接重写历史**（对象池共享），先 unlink 或走"归档旧项目 + 新建"路线。
6. 最后记住：prune 不保证清掉泄漏的密钥（对象可能从别的 ref 可达），删密钥走第 12 篇的专门修复流程。

# 12. 归档、转移与删除：生命周期三连

## 12.1 归档（Archive）

只读冻结：不能 push、不能建 MR、流水线停跑，但可浏览、可克隆、可查 Issue。项目下线但保留证据链的正确姿势；误归档随时 unarchive。归档中的项目**不能被转移**——先解档再转。

## 12.2 转移（Transfer）：改 namespace

前提：对该项目是 **Owner**、对目标群组是 **Maintainer/Owner**、目标群组开放建项目。转移现在是**异步**执行的（18.11 引入、19.4 GA；大项目要等后台任务跑完才能在目标 namespace 下看到项目），成功时给订阅了 "Moved project" 通知的人发邮件，失败时给发起人一条 to-do 写明原因；**旧 URL 自动重定向**到新位置。

随转移走的东西（官方清单）：Issues/MR/讨论串、流水线与配置、仪表盘与 wiki、**代码与 Git 历史、项目访问令牌**、直接成员与待处理邀请；标签在目标群组缺同名时新建；Epic 会被复制进目标群组（多项目共用的 epic 会变成多份拷贝，转前要意识到）。

已知硬约束：
- **继承型成员**（通过旧父群组拿到的权限）若不属于目标群组，转移瞬间失去访问——转完必须过一遍成员表；
- 启用了容器镜像仓库的项目：GitLab.com 上只能在同一顶级群组内转，自建实例则要求**不含镜像**才能转；镜像仓库超过 1,000 个直接禁止转移；
- npm 包命名绑 root namespace：跨顶级 namespace 转移后要么改 scope 重发，要么接受包不再符合命名约定。

转移后的固定动作清单：

```bash
# 每个本地 clone
git remote set-url origin https://gitlab.example.com/new-group/project.git
git fetch origin
```

外加：CI/CD 变量与 runner 标签复核、webhook 指向复核、安全策略继承重算、包坐标与制品下载地址更新（第 8/10 篇）。

## 12.3 删除与恢复窗口

项目和群组都支持两种删除：**计划删除**（排入到期窗口，期内可从"待删除"列表一键恢复）与**立即删除**。窗口时长由管理员配置，但请把心智模型摆正：恢复窗口是防手滑的，不是备份——真正兜底永远是第 13 篇那套 `gitlab-backup` + secrets 检查流程。

# 13. 日常浏览与 Web IDE

- **Web IDE**：任意仓库页按 `.` 进入，改完直接建分支提 MR，适合小修小补；大改动仍建议本地 IDE。关键认知：**Web IDE 的提交在服务端就是一次普通 push**，Push Rules（含签名要求，见 7.5）、历史保护检查全部照过——"网页上点的"不等于"管理端豁免的"。
- 文件 **Blame/History**：每个文件页都能看行级作者与历史；`/blob/main/pom.xml` 这类 URL 可直接分享，`#L12-20` 锚点定位到行。
- 快捷键：仓库内按 `t` 唤文件跳转，`/` 唤全局搜索（代码、MR、Issue、Wiki 分域）。搜索语法支持按 `project:`、`author:` 等限定符收窄，跨群组排查"这个配置到底谁在写"时比翻页快一个数量级。
- 行级分享 + Blame 是评审外的沟通利器：跨团队答疑时贴永久链接前先想一层——**分支会变，tag 不会**，引用要落 tag 或用 commit 固定。
- 并发改大文件（如锁文件、生成代码）易互相踩踏：文件页提供 **Lock file**，锁定期间别人不能经 UI 编辑同一文件——它是 UI 层的礼貌机制，**不拦 `git push`**，真正的串行化还得靠分支纪律和 MR 流程（第 5 篇）。

# 14. 小结

| 要点 | 一句话 |
|------|--------|
| 组织结构 | group 无硬上限但推荐"成员<1000 / 子群<100 / 深度≤5"；策略是"新建才继承、项目即替换" |
| 可见性 | 实例默认 Private；Internal 不是安全；关功能要看依赖链 |
| 分支模型 | 主干 + 短命分支 + release 线是 Java 团队最优解；命名规则落到服务端正则而不是口头 |
| 默认分支 | 五档初始保护统一下发，非 Fully protected 存在 CI 变量外泄风险；引用统一写 `$CI_DEFAULT_BRANCH` |
| 门禁组合 | 保护分支（谁能写）+ Merge 方法/检查（怎么合）+ Push Rules（服务端拦截）+ CODEOWNERS（谁批） |
| 体积治理 | size=du 语义、15 分钟节流；housekeeping 双策略 + Gitaly 轮扫；重写历史前先想 fork 和重 clone |
| 生命周期 | 归档只读；转移带成员与 token、旧 URL 重定向、容器/npm 有硬约束；恢复窗口不是备份 |

自检清单：

- [ ] 每个新项目的推送规则用 `glab api projects/:id/push_rule | jq` 读回过？
- [ ] 群组/实例下发的默认分支保护档位是 Fully protected？
- [ ] CODEOWNERS 命中的是"第一命中制"里预期的那份文件，且保护分支开了 owner 审批？
- [ ] 大仓库看过 Usage quotas 构成、Gitaly 日志里有 `maintenance: daily completed`？
- [ ] 转移后的 remote、成员权限、CI 变量、包坐标全部核对过？

仓库与分支的地基到此打完。MR 评审、审批规则、合并按钮背后的完整行为，在第 5 篇展开；权限矩阵与扫描档位在第 12 篇。
