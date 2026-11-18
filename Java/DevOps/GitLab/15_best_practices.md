# 1. 本篇定位：把前 14 篇拼成一份落地手册

单点功能都讲过了，真实团队引入 GitLab 时的难点是**顺序与取舍**：先定什么、什么值得模板化、哪些设置一旦定下就难改。本篇给一套可直接照抄的落地清单。

## 1.1 五个步骤的排序依据

排序只有一条准则：**按"改起来的代价"从大到小**。

| 顺序 | 决策项 | 后期改动成本 | 为什么贵 |
|------|--------|-------------|---------|
| ① | 群组结构 | 极高 | 权限、CI 模板、安全策略、制品仓库全部沿群组继承，挪一次群组等于重做一遍治理 |
| ② | 仓库规范 | 高 | 模板文件随仓库数量扩散，晚立规范要回改上百个仓库 |
| ③ | 分支与 MR 门禁 | 高 | 保护分支/审批规则与团队习惯绑定，改配置易、改习惯难 |
| ④ | CI 工程细节 | 中 | `.gitlab-ci.yml` 是版本化文件，可以 MR 方式渐进重构 |
| ⑤ | 安全与合规基线 | 中 | 扫描器/令牌策略可后补，但"已经泄露的密钥"无法后补，所以仍要在早期做一次密钥清理 |

## 1.2 本篇怎么用

- **新建实例**：按 ①→⑤ 顺序做，每步都有可执行片段。
- **已运行多年的实例**：从第 8 节"反模式"倒查，命中哪条就从对应步骤切入。
- **验证工具**：`glab`（官方 CLI）+ REST API，本篇脚本都以 `glab` 为入口，避免手写 curl。

```bash
# 装 CLI 后登录自建实例（注意 --hostname）
glab auth login --hostname gitlab.example.com
# 验证身份与实例版本：后面所有脚本都依赖这两项正确
glab api user | jq -r '.username'
glab api version | jq -r '.version'
```

# 2. 第一步：组织结构（改起来最贵，最先定）

| 决策 | 推荐 | 理由 |
|------|------|------|
| Group 划分 | 按产品线/业务域，嵌套 ≤3 层 | 权限、模板、安全策略全部沿群组继承（第 1 篇/第 13 篇） |
| 平台团队 | 独立 `platform` 群组：ci-templates 仓库、基础镜像仓库、工具链 | CI 模板与基镜像需要"版本化的内部供应商" |
| 项目粒度 | 一个服务/库一个项目；Monorepo 单独评估 | 项目级流水线、权限、制品边界最自然 |
| Monorepo vs Polyrepo | 跨团队协作强、原子变更多 → Monorepo（配父子流水线/changes 规则）；否则 Polyrepo | 第 7 篇的 rules+父子流水线是 Monorepo 可行性的前提 |

## 2.1 一棵典型群组树

```text
root
├── platform                      # 平台组：内部供应商
│   ├── ci-templates              # 顶层 templates/ 目录：include 模板 + CI/CD Components（版本化）
│   ├── images                    # 基础镜像仓库（Dockerfile 构建流水线）
│   └── tooling                   # 脚手架、glab 扩展、pre-commit 配置
├── payments                      # 业务域
│   ├── order-service             # 一个服务一个项目
│   ├── payment-gateway
│   └── libs                      # 该域内的共享库
├── infra                         # 基础设施：IaC、集群配置
└── archive                       # 只读归档（Visibility=Private + Archived）
```

要点：

- **业务域群组是权限边界**，成员加在群组上而不是逐个项目加；离职回收时只动一处。
- **`archive` 用 Archive 而非删除**：归档后仓库只读、流水线不再触发，历史与制品仍可查（第 4 篇）。
- **不要把"人"组织进树**（如 `team-a/...`），人员会重组，产品线更稳定。

## 2.2 沿层级下发的三样东西

```text
Group ──► 继承成员与角色        （Settings → Members，成员来源即"Inherited membership"）
Group ──► 模板项目与骨架        （platform/ci-templates 提供 include 目标；模板项目开 Settings → General → Project template）
Group ──► 安全策略 / MR 审批策略 （Premium/Ultimate：.gitlab/security-policies/policy.yml 与群组级审批规则，见第 12 篇）
```

> **别找"群组一键注入 CI 配置"的开关** —— GitLab **没有**"群组统一强制下发 `.gitlab-ci.yml`"的原生设置（`ci_config_path` 只指本项目内的文件路径）。落地手段是三件套：**脚手架保证新项目第一天就 `include` 平台模板** + **平台模板本身版本化（tag/Component）** + **周期性合规检查**（读回每个项目的 merged config，检查必需 job 是否存在）。Ultimate 还可以用安全策略/流水线执行策略强制"必须有某类扫描 job"。

**验证继承是否生效**（新项目建好后立刻验证，别等出事故）：

```bash
# 1) 成员继承：应能看到来自父群组的用户及其角色
glab api projects/payments%2Forder-service/members/all | jq -r '.[] | "\(.username)\t\(.access_level)"'
# 2) CI 入口是否被改到过深路径（应为空或本项目内路径）
glab api projects/payments%2Forder-service | jq '.ci_config_path'
# 3) 平台模板是否真的被这个项目的流水线包含
glab api "projects/payments%2Forder-service/repository/files/.gitlab-ci.yml/raw?ref=main" | grep -n 'platform/ci-templates'
# 4) 群组级变量/令牌可见性
glab api groups/platform/variables | jq -r '.[].key'
```

## 2.3 组织结构上的三个常见错误

| 错误 | 表现 | 纠正 |
|------|------|------|
| 按人/团队建群组 | 组织调整后要批量迁移项目 | 按业务域建，人员用群组继承 |
| 层级过深（5+ 层） | 权限排查需要一路点上去，审批规则失效难定位 | 压到 3 层，超出部分用标签/项目命名解决 |
| 项目内再放"子项目" | 一个大仓库里塞十几个服务，CI/权限无法拆分 | 用父子流水线（第 7 篇）替代拆分需求 |

# 3. 第二步：统一仓库规范（用模板仓库固化）

建一个 `platform/repo-defaults` 仓库，每个新项目落地这套文件：

```text
project/
├── .gitlab-ci.yml              # include 平台模板，只留本项目差异
├── .gitlab/
│   ├── issue_templates/        # Bug/Feature 表单（第 11 篇）
│   └── merge_request_templates/Default.md   # MR 描述模板
├── CODEOWNERS                  # 路径→负责人（第 4 篇）
├── .gitlab/CODEOWNERS          # （或此路径）
├── renovate.json               # 依赖自动更新
└── docs/                       # 设计文档随代码评审
```

## 3.1 Issue 模板：Markdown 是唯一稳定的入口

`.gitlab/issue_templates/bug.md` —— 规范要"进门就生效"，靠的是**模板 + 快捷操作**，不是靠培训：

```markdown
<!-- .gitlab/issue_templates/Bug.md -->
## 现象（含报错原文）
<!-- 把日志原文粘贴在这里，不要截图 -->

## 步骤
1.

## 影响面
- [ ] 生产可用但降级
- [ ] 生产不可用
- [ ] 仅测试环境

## 版本/分支
- 版本：
- 环境：dev / staging / prod

/label ~bug ~"needs-triage"
/assign @oncall-backend
/due in:7d
```

**四条落地要点**（细节见第 11 篇）：

| 要点 | 做法 | 为什么 |
|------|------|--------|
| 默认模板 | `.gitlab/issue_templates/Default.md`（文件名大小写不敏感） | 新建页自动预填，不用人选 |
| **优先级坑** | 项目 **Settings → Issues → Default description**（Premium/Ultimate）**会盖过 `Default.md`** | 两处都配会互相遮蔽，只留一个入口 |
| 跨仓库复用 | 群组 **Settings → General → Templates → Template repository**（需 Premium/Ultimate + Owner，**模板项目必须是该群组直接子项目**） | 一处维护、组内所有项目继承 |
| 强制结构化 | 用 **URL 预填参数**给外部入口（告警机器人、值班表）：`.../-/work_items/new?issue[title]=...&issuable_template=Bug` | 表单化"必填"这件事 GitLab 原生 Markdown 模板做不到，**用入口约束代替字段校验** |

> **版本提醒（勿照抄旧博客）**：早前实验性的 **YAML 表单式 issue 模板**（`type: form` + `body:` 字段校验）在现行官方文档中**已不再作为主推路径**，各版本实例表现不一致。自建实例若仍需"字段级必填校验"，稳妥做法是：Markdown 模板 + CI 里的**描述内容检查**（缺 `## 现象` 就 fail），或走 Service Desk / 外部工单系统。

## 3.2 MR 模板

`.gitlab/merge_request_templates/Default.md`（MR 模板用 Markdown；**默认模板**额外支持 `%{issue}`、`%{issue_title}`、`%{issue_link}`、`%{all_commits}` 等变量，由 GitLab 在创建时替换，见第 11 篇 4.2）：

```markdown
## 变更内容
- [ ] 一句话说明做了什么、为什么

## 影响面
- 涉及模块：
- 是否含 DB 变更（有 → 附迁移脚本与回滚方案）：
- 是否含配置变更（有 → 同步更新 `docs/` 与运维手册）：

## 如何验证
1. `mvn -B verify` 通过，本地/CI 覆盖率变化：
2. 手工验证步骤（给可复制的命令，不要写"自己试一下"）：

## 风险与回滚
- 回滚方式：直接 revert 本 MR / 需要执行 `...`

/assign @tech-lead
/label ~"review::code"
```

模板末尾写 `/assign`、`/label` 这类**快捷操作**，创建时即生效，省去每次手点。

## 3.3 CODEOWNERS

```text
# CODEOWNERS —— 路径 → 必须审批的负责人（Premium/Ultimate 才会强制门禁）
/docs/            @tech-lead @docs
/src/main/java/com/acme/pay/**   @payments-team
/pom.xml          @platform-build
*.sql             @dba
```

配合审批规则里的 **Code Owner approval required**，未列到的人即使有 Maintainer 也无法独自合并——这是"平台组退出业务代码审批"的标准做法。

## 3.4 CI 模板的标准形态

简单的 `include` 模板（跨项目引用 + 版本锁定）：

```yaml
# platform/ci-templates/templates/java-service.yml —— 全公司 Java 服务统一骨架
stages: [build, test, scan, package, deploy]

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

# 约定优于配置：业务仓库只允许覆盖这些变量
```

业务仓库里这样引（**必须锁 ref，模板升级走 MR，而不是让所有人被动漂移**）：

```yaml
# 业务项目 .gitlab-ci.yml
include:
  - project: platform/ci-templates
    ref: v1.4.2                       # 锁 tag，不用 main
    file:
      - templates/java-service.yml
  - local: .gitlab/ci/overrides.yml   # 本项目差异，只允许覆盖变量和追加 job
```

更进一步用 **CI/CD Components**（把模板做成带输入的"函数"，17.x 起官方主推的复用方式；完整规则见第 7 篇第 6 节）：

```yaml
# platform/ci-templates/templates/maven-build.yml
#                     ^^^^^^^^^ 必须在顶层 templates/ 目录，不是 components/
spec:
  description: 统一 Maven 构建骨架
  inputs:
    image:
      default: "maven:3.9-eclipse-temurin-21"
    goals:
      default: "verify"
---
maven-build:
  stage: test
  image: $[[ inputs.image ]]
  script: mvn -B $[[ inputs.goals ]]
  cache:
    key:
      files: [pom.xml]
    paths: [.m2/repository]
```

```yaml
# 业务项目：声明式使用，升级 = 改版本号，可被 Renovate 自动提 MR
include:
  - component: $CI_SERVER_FQDN/platform/ci-templates/maven-build@1.2.0
    inputs:
      image: "maven:3.9-eclipse-temurin-17"
```

**三条容易写错的语法**（写成下面这样会直接报 invalid configuration）：

| 错误写法 | 正确写法 |
|----------|----------|
| `.../maven-build/1.2.0` | `.../maven-build@1.2.0`（**版本用 `@` 分隔**） |
| 硬编码 `gitlab.example.com/...` | 用 `$CI_SERVER_FQDN` 前缀（跨实例可移植） |
| 文件放 `components/*.gitlab-ci.yml` | 放顶层 `templates/<name>.yml` 或 `templates/<name>/template.yml` |

## 3.5 新项目一键初始化

模板仓库要能被"机器使用"，否则规范只会留在文档里：

模板仓库要能被"机器使用"，否则规范只会留在文档里：

```bash
#!/usr/bin/env bash
# create-service.sh <group-path> <project-path> —— 生成合规新项目并下发基线
set -euo pipefail
GROUP=$1; PROJ=$2
HOST=gitlab.example.com
NS_ID=$(glab api "groups/$GROUP" | jq -r .id)

# 1) 建空项目（不自动初始化 README，避免骨架与 README 冲突）
PID=$(glab api --method POST projects \
        -f name="$PROJ" -f path="$PROJ" -f namespace_id="$NS_ID" \
        -f visibility=private -f initialize_with_readme=false | jq -r .id)

# 2) 从骨架仓库灌入初始代码（骨架本身也是一个受评审的 GitLab 项目）
git clone --depth 1 "git@$HOST:platform/repo-defaults.git" /tmp/$PROJ
rm -rf /tmp/$PROJ/.git
git -C /tmp/$PROJ init -q -b main
git -C /tmp/$PROJ remote add origin "git@$HOST:$GROUP/$PROJ.git"
git -C /tmp/$PROJ add -A && git -C /tmp/$PROJ commit -qm "chore: scaffold from repo-defaults"
git -C /tmp/$PROJ push -qu origin main

# 3) 下发 MR/分支基线（见 4.1），并记录到审计流水
bash apply-baseline.sh "$GROUP/$PROJ" | tee -a /var/log/gitlab-scaffold.log
```

> **两条可选捷径**：① 在 UI 建项目时选 **Create from template → 本群组的模板项目**（把骨架项目设为 **Settings → General → Project template**）；② API 侧 `POST /projects` 支持 `use_custom_template` + `group_with_project_templates_id`（**Premium/Ultimate**，群组级自定义模板）。用哪种取决于订阅：**Free 版只能靠上面的 git 灌入法**。
>
> **`ci_config_path` 的边界**：它只是"本项目的 CI 配置文件路径"（Settings → CI/CD → CI/CD configuration file）。**不要**指望用它把配置指到别的仓库 —— 跨仓库复用请在项目自己的 `.gitlab-ci.yml` 里写 `include: project:`（3.4 的写法），这才是可移植、可锁 ref 的正道。

# 4. 第三步：分支与 MR 流程基线（一次配好，全组继承）

推荐基线（可在此上下微调，见第 4/5 篇）：

| 项 | 基线配置 |
|----|---------|
| 默认分支 | main，保护：No force push，Maintainers 可合并（Developer 可提 MR） |
| 合并方式 | Fast-forward + squash；MR 标题即 commit |
| 门禁 | 流水线必须绿 + 全部讨论 resolve + 审批 ≥2（关键库 +Code Owner） |
| 大改动 | Draft MR 先行 |
| 发布 | Tag `vX.Y.Z` 触发发布流水线 + Release Notes 自动聚合 |
| 并发合并风险高的仓库 | Merge Train |

## 4.1 用 API 把基线批量下发

新项目的 MR 设置靠人点必然漂移，把它写进脚手架。**注意三个端点的动词不一样**（这是最容易写错的地方）：

```bash
#!/usr/bin/env bash
# apply-baseline.sh <group/project>
set -euo pipefail
P=$(printf '%s' "$1" | sed 's#/#%2F#')     # URL-encode 命名空间路径

# ① 保护分支：创建用 POST、改字段用 PATCH（不是 PUT）
glab api --method POST "projects/$P/protected_branches" \
  -f name=main -f allow_force_push=false \
  -f push_access_level=40 -f merge_access_level=40 -f unprotect_access_level=40 \
  || glab api --method PATCH "projects/$P/protected_branches/main" -f allow_force_push=false

# ② 项目级 MR 行为：PUT /projects/:id（merge_method 只有 merge|rebase_merge|ff）
glab api --method PUT "projects/$P" \
  -f only_allow_merge_if_pipeline_succeeds=true \
  -f only_allow_merge_if_all_discussions_are_resolved=true \
  -f remove_source_branch_after_merge=true \
  -f merge_method=ff -f squash_option=default_on \
  -f autoclose_referenced_issues=true \
  -f merge_request_title_regex='^(feat|fix|chore|refactor|perf|docs|test|build)(\([a-z0-9.-]+\))?!?: .+' \
  -f merge_request_title_regex_description='标题需用 Conventional Commits 前缀，便于 Release Notes 自动聚合'

# ③ 审批：项目级审批"配置"是 POST /projects/:id/approvals（对，是 POST）
glab api --method POST "projects/$P/approvals" \
  -f reset_approvals_on_push=true \
  -f merge_requests_author_approval=false \
  -f disable_overriding_approvers_per_merge_request=true

# ④ 审批"规则"：approvals_before_merge 早在 12.3 就废弃，改用规则
#    默认规则名通常与项目同名，先列出来拿 id，再改 approvals_required
glab api "projects/$P/approval_rules" | jq -r '.[] | "\(.id)\t\(.name)\t\(.approvals_required)"'
glab api --method POST "projects/$P/approval_rules" \
  -f name='Payments review' -f approvals_required=2 -f 'user_ids[]=123' -f 'user_ids[]=456'

# ⑤ Code Owner 强制：既要在 CODEOWNERS 里写路径，也要开保护分支上的开关
glab api --method PATCH "projects/$P/protected_branches/main" -f code_owner_approval_required=true
```

**下发纪律**：

- 字段名与可选值随版本略有差异（例如 `squash_option` 的取值、`code_owner_approval_required` 需要 Premium/Ultimate）。**下发前先 `glab api "projects/$P" | jq` 读回现值，改完再读一次确认**；把这段"读回比对"做成 CI 定时任务，等于给规范加了自动化守护。
- `code_owner_approval_required` **只在受保护分支上生效**，且路径没有匹配到任何 CODEOWNERS 条目时会静默失效 —— 上完必须造一次"改了 Owner 路径但没请 Owner 审批"的用例验证。
- 群组级基线（所有子项目继承）走 **Settings → General → Merge request approvals** 的群组设置或 `groups/:id/approvals`，比逐项目下发更抗漂移。
- **MR 标题正则 ≠ 需求锚点校验**：真要"每个 MR 必须关联 Issue"，用 CI 检查 MR 描述里是否含 `#`/`Closes`（第 11 篇第 9 节），标题正则只管提交规范。

## 4.2 审批规则基线与两个坑

| 维度 | 配置位置 | 建议值 |
|----------|---------|------|
| 最少审批数 | Settings → Merge requests → Merge request approvals | 2（关键库 3） |
| 分组审批 | Approval rules | 后端组 ≥1、DB 变更 ≥1 |
| Code Owner | CODEOWNERS + 规则 | 支付/权限路径必须 Owner |
| 批准后又有新提交 | Reset approvals on push（"作者推新提交清空审批"） | 开启 |
| 作者自批 | `merge_requests_author_approval=false`（UI 勾选框文案随版本变化） | 关闭自批（审批能力本身要求 Premium/Ultimate） |

**坑 1：规则无法满足时 GitLab 会标 `Auto approved`。** 例如规则要求 3 人审批但候选审批人只有 2 人、唯一候选人是 MR 作者、或规则里没有匹配到任何人（`CODEOWNERS` 写错路径）。此时 MR **不会被卡住** —— 门禁静默失效。上线规范后必须专门造一次"规则不满足"的用例来验证，并在审计脚本里检查每条规则的候选审批人（`approval_rules` 返回的 `users`/`groups` 数组，字段名以现值为准）：

```bash
# iid 是数字，不带 "!"；glab api 直接写 REST 路径
glab api "projects/$P/merge_requests/42/approval_state" | jq
glab api "projects/$P/approval_rules" | jq -r '.[] | "\(.name)\treq=\(.approvals_required)\tusers=\(.users|length)\tgroups=\(.groups|length)\trule=\(.rule_type)"'
```

**坑 2：审批规则能在 MR 里被临时覆盖。** 只要没把 `disable_overriding_approvers_per_merge_request` 设为 `true`，提交人就能在 MR 页面改审批人（把自己加进去）。群组/实例级规则是更抗绕过的兜底 —— 但**群组级规则仍可在项目里被"额外规则"补充**，所以关键库要同时开 ① 群组规则 ② 禁止 MR 内覆盖 ③ 禁止作者自批。

## 4.3 Draft 与 Merge Train 的分工

- **方向未定 → Draft MR**（标题自动前缀 `Draft:`，门禁禁止合并但可评审、可跑流水线）。评审通过去掉 Draft。
- **多个 MR 都绿但合并后互相打破 → Merge Train**（Premium/Ultimate）：排队，每个 MR 都在"前序已合并"的结果上重跑流水线。开启前提：使用 MR 流水线、勾选 **Pipelines must succeed**、合并方式符合约束（建议 fast-forward + squash）。

# 5. 第四步：CI 工程的团队习惯

| 实践 | 收益 |
|------|------|
| `.gitlab-ci.yml` 变更走 MR 评审，像代码一样对待 | 流水线即产品 |
| 镜像化构建环境（固定 `image:` 版本，禁 `latest`） | 可复现、不被上游炸 |
| Maven 缓存 + junit 报告 + `interruptible` | 快、可诊断、省资源（第 6/8 篇） |
| 流水线阶段目标 ≤10 分钟（超时拆并行/按需 changes） | MR 反馈速度决定合并频率 |
| 生产部署只由 Tag/manual 触发，密钥全走 Protected 变量 | 可审计、防误发（第 9/12 篇） |
| 定期跑 pipeline efficiency 报表，清理僵尸 job | CI 成本可控（第 14 篇） |

## 5.1 一条"参考级"流水线

把散落在第 6/7 篇的机制合成一条团队可直接抄的骨架：

```yaml
# .gitlab/ci/java-service.yml（业务项目里 include 它）
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
    - if: $CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "nightly"

stages: [build, test, package, deploy]

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script: mvn -B -DskipTests package
  cache:
    key:
      files: [pom.xml]
    paths: [.m2/repository]
  artifacts:
    paths: [target/*.jar]
    expire_in: 1 week

unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  needs: [build]
  parallel:
    matrix:                              # Maven 不会自动分片：按模块拆，否则 4 份跑同一套全量测试
      - MODULE: [order-api, order-web, order-job]
  script:
    - mvn -B verify -pl $MODULE -am -Dsurefire.rerunFailingTestsCount=2
  artifacts:
    when: always
    expire_in: 1 month
    reports:
      junit: "**/target/surefire-reports/TEST-*.xml"

# 三个内置安全模板的 job 都落在 stage: test（不是 scan）。
# 想让它们单独成段，就要覆盖各自模板的 stage，例如 sast: { stage: scan }。

package-image:
  stage: package
  image:
    name: gcr.io/kaniko-project/executor:v1.23.2-debug
    entrypoint: [""]                     # kaniko 镜像自带 entrypoint，不覆盖会直接 command not found
  needs: [unit-test]
  before_script:
    - mkdir -p /kaniko/.docker
    - printf '{"auths":{"%s":{"auth":"%s"}}}' "$CI_REGISTRY"
        "$(printf '%s:%s' "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')"
        > /kaniko/.docker/config.json    # kaniko 不读 docker login 的默认位置，必须写这个文件
  script:
    - /kaniko/executor
        --context $CI_PROJECT_DIR
        --dockerfile $CI_PROJECT_DIR/Dockerfile
        --destination $IMAGE_TAG
        --cache=true --cache-repo=$CI_REGISTRY_IMAGE/cache
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  interruptible: true

deploy-staging:
  stage: deploy
  needs: [package-image]
  environment:
    name: staging
    url: https://staging.example.com
  script: ./deploy.sh staging $IMAGE_TAG
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
    auto_stop_in: never                  # 合法值含 never（自然语言时长亦可：7 days）
  needs: [package-image]
  rules:
    - if: $CI_COMMIT_TAG
      when: manual                       # 生产：Tag + 人工确认
  resource_group: production             # 串行化，避免并发部署互踩
```

对照要点：`workflow:rules` 决定"什么建流水线"、job 级 `rules` 决定"什么建 job"、`needs` 决定"何时能跑"、`resource_group` 决定"不许并发"、`environment` 决定"发去哪 + DORA 埋点"。

## 5.2 缓存与报告的纪律

| 项 | 规则 | 违反后果 |
|----|------|---------|
| 缓存 key | `key:files: [pom.xml]` 或 `key: "$CI_COMMIT_REF_SLUG"` | 恒定 key 会让依赖升级后仍用旧包，构建诡异失败 |
| 缓存作用域 | 只缓存只读依赖目录，勿缓存 `target/` | job 间脏产物互相污染 |
| 测试报告 | `artifacts:reports:junit` 必配 | MR 页看不到失败用例，评审靠猜 |
| 覆盖率 | `coverage: "Total coverage (\\d+\\.\\d+)%"` | 无法回答"这个 MR 降了多少覆盖率" |
| 制品有效期 | 交付物 1 周、报告 1 月、日志 7 天 | 磁盘被制品吃满（第 13/14 篇配额） |
| `interruptible: true` | 给 MR 类 job 打开 | 每次推 commit 都排队跑完旧流水线，Runner 白白耗尽 |

## 5.3 用数据管 CI：本地 lint、周期复盘

```bash
# 提交前 lint：语法错会直接指出行列，等价于 Web 上的 CI Lint 页面
glab ci lint
# 看模板/include 展开后的"最终配置"：用 lint 端点自查 merged_yaml
glab api --method POST "projects/$P/ci/lint" -f "content=$(cat .gitlab-ci.yml)" | jq -r '.merged_yaml' | head -40
# 流水线页 → Action ▸ View deployment/Config 也能直接看到同一份展开结果

# 找出最近一个月最慢的 job 与最久的 pending 时间
glab api "projects/$P/pipelines?per_page=100&updated_after=2026-10-01T00:00:00Z" \
  | jq -r '.[] | "\(.id)\t\(.coverage)\t\(.created_at)\t\(.finished_at)"'
glab api "projects/$P/jobs?per_page=100&scope[]=success&scope[]=failed" \
  | jq -r '.[] | select(.stage=="test") | "\(.name)\t\((.finished_at|fromdateiso8601) - (.started_at|fromdateiso8601))s"' \
  | sort -k2 -nr | head -20
```

配套指标看项目 **Analyze → CI/CD analytics**（时长趋势、失败率、Runner 队列等待），把"流水线 P90 > 10 分钟"设成每周待办，而不是凭感觉优化。

# 6. 第五步：安全与合规基线

- 实例层：关注册、强制 2FA、SSO、Webhook 出站白名单（第 12 篇清单）。
- 项目层：默认 Private；Secret Detection + 依赖扫描全量开，高危阻断合并。
- 供应链：镜像只由 CI 产出推 Registry；发布 Tag 镜像打不可变版本并签名（cosign），Registry 清理策略豁免发布 tag（第 10 篇）。
- 备份：每日全备 + 异地对象存储 + 季度恢复演练（第 13 篇）。

## 6.1 扫描门禁的分级落地

一刀切"有漏洞就阻断"必然被绕过。按层给策略，并且**存量先只报不卡、新增才卡**：

| 扫描器 | include | 初态 | 终态 |
|--------|---------|------|------|
| Secret Detection | `template: Security/Secret-Detection.gitlab-ci.yml` | MR 阻断（密钥必须轮换，不接受"以后改"） | 保持阻断 |
| SAST | `template: Security/SAST.gitlab-ci.yml` | `allow_failure: true`（报告可见） | 新增高危阻断（Ultimate：MR 审批策略/security policy） |
| Dependency Scanning | `template: Security/Dependency-Scanning.gitlab-ci.yml` | 只报 | 按 SLA 阻断（严重 7 天、高危 30 天） |
| Container Scanning | `template: Security/Container-Scanning.gitlab-ci.yml` | 只报 | 生产镜像阻断严重漏洞 |

```yaml
# 把"只报不卡"写成显式配置，避免团队以为是平台坏了
secret-detection-gate:
  extends: secret_detection
  allow_failure: false          # 唯一不放行的扫描
sast:
  allow_failure: true           # 存量治理期放行，终态改 false
```

> **Secret Detection 全层级可用**；SAST / 依赖扫描 / 容器扫描的**扫描 job 本身**在免费层也能跑起来（各扫描器与版本的层级划分调整过多次，落地前以你实例的 CI 目录页或官方定价页为准）。真正需要 Premium/Ultimate 的是**治理侧能力**：MR 漏洞报告与漏洞看板、按严重级别阻断合并的审批策略、安全策略下发。**不要把"我们没有 Ultimate"当作不开扫描的理由。**

## 6.2 令牌与凭据基线（照抄表）

| 场景 | 该用什么 | 禁用 |
|------|---------|------|
| 流水线内访问 GitLab API / 拉同实例依赖 | **CI Job Token**（`id_tokens` 可换云侧凭据） | 个人 PAT 塞进变量 |
| 外部脚本读写仓库（CI 之外） | Project Access Token（限定 scope + 过期） | 长期不过期 PAT |
| CI 推镜像 / 拉制品 | Deploy Token 或 Job Token | 管理员账号密码 |
| Runner 认证 | Runner **authentication token**（前缀 `glrt-`）或 **Runner agent**（agent token，前缀 `glagent-`） | 老的 registration token 流程（17.0 弃用、**20.0 移除**，第 8 篇） |
| 云厂商访问 | OIDC 角色扮演（`id_tokens`） | 把 AK/SK 存成变量 |
| 生产密钥 | 变量设 **Protected + Masked**，绑定 protected environment | 明文写在 `script` 或仓库文件 |

```bash
# ① 建"只读仓库"权限的 Deploy Token（明文只在创建时返回一次；用户级端点是 POST /user/deploy_tokens）
glab api --method POST user/deploy_tokens -f name=ci-registry -f 'scopes[]=read_registry' | jq '{username, token}'
# ② 存入项目变量并同时设 Protected + Masked：只有受保护分支/tag 的流水线能取到，日志里不原样输出
glab variable set -R payments/order-service REG_USER  '<username>' --protected --masked
glab variable set -R payments/order-service REG_TOKEN '<token>'    --protected --masked
# ③ 盘点是否还有人往项目里塞未保护/未掩码变量（治理脚本每周跑一次）
glab api projects/payments%2Forder-service/variables \
  | jq -r '.[] | select(.protected==false or .masked==false) | "\(.key)\tprotected=\(.protected)\tmasked=\(.masked)"'
```

> **两个细节**：① `--masked` 只保证变量值不在日志里原样打印，**不保证不进缓存/制品**，构建产物仍可能泄露；② 变量在 merged YAML（流水线页 "View configuration"）中是**按引用展开**的，所以**永远不要把密钥写进 `variables:` 字面量** —— 能看流水线配置的人就能看到你写了什么。

## 6.3 备份与恢复演练 runbook（季度一次）

备份只有被恢复验证过才算存在。三步走完，任何一步失败都要开 Issue：

```bash
# 1) 备份（第 13 篇细节：包含 /etc/gitlab 与 gitlab-secrets.json）
sudo gitlab-backup create SKIP=artifacts,registry
sudo tar czf /backup/etc-$(date +%F).tar.gz /etc/gitlab

# 2) 恢复演练：在隔离机/容器里恢复，校验三件事（先停写入服务再 restore）
sudo gitlab-ctl stop puma sidekiq
sudo gitlab-backup restore BACKUP=<timestamp>
sudo gitlab-rake gitlab:check SANITIZE=true
sudo gitlab-rails runner 'puts Project.count, User.count, Ci::Pipeline.maximum(:id)'
# 3) 演练结论写进 Issue：恢复耗时 RTO、数据丢失量 RPO、未覆盖项
```

| 检查项 | 通过标准 |
|--------|---------|
| 登录与权限 | 演练实例能用同样的账号体系打开一个私有项目 |
| 流水线 | 历史流水线可打开、制品可下载 |
| 仓库 | `git clone` 一个中等仓库 + 一次 push 成功 |
| 计时 | 记录恢复总耗时，作为 RTO 依据；备份起始时间差 = RPO |

# 7. 分角色速查

| 角色 | 必读篇 |
|------|--------|
| 普通开发 | 4、5、6 |
| TL / 架构 | 4、5、7、11 |
| DevOps / SRE | 2、3、8、9、13、14、16 |
| 安全/合规 | 12、10 |
| 管理者 | 1、14（DORA）、15 全篇 |

## 7.1 30/60/90 天推进计划

| 阶段 | 目标 | 关键产出 | 判定"完成"的证据 |
|------|------|---------|----------------|
| 第 1-30 天 | 跑得起来 | 实例安装 + 群组结构定稿 + 一个试点项目接入 | 试点项目完成"提交→MR→CI→部署 staging"闭环，全程 ≤15 分钟 |
| 第 31-60 天 | 规范固化 | `repo-defaults` + CI 模板 + 保护分支/审批批量下发脚本 | 新项目从脚手架生成，MR 门禁自动生效，无需人工配置 |
| 第 61-90 天 | 治理与安全 | 扫描门禁分级、令牌收敛、备份恢复演练、DORA 基线 | 一次演练报告 + 泄露密钥清零 + 流水线 P90 达标 |
| 第 90 天后 | 迁移与扩展 | 存量仓库分批迁移、Runner 容量规划、观测接企业 Prometheus | 存量迁移完成率、Runner 队列等待时长曲线 |

# 8. 常见反模式

| 反模式 | 后果 | 纠正 |
|--------|------|------|
| 把 GitLab 当纯代码托管，CI 留在 Jenkins | 两套门禁、双份维护 | 新仓库默认 GitLab CI，存量逐步迁 |
| 每个仓库手抄一份 yml | 规范漂移，无人负责 | 模板仓库 + include + 版本锁定 |
| main 不设保护、人人可推 | 事故源头 | 第 4 步基线 |
| CI 用个人 PAT/长期 token 推制品 | 离职即断、权限漂移 | Job/Deploy Token |
| 镜像只打 `latest` | 无法回滚 | 短 SHA + 版本 tag + 豁免清理 |
| DORA 指标做 KPI | 指标造假开始 | 趋势自用，不入考核 |
| 审批规则写了但没人能批 | 门禁静默失效（`Auto approved`） | 建规则后校验候选审批人数量 |
| `when: manual` 满天飞当"安全" | 没人记得点，发布依赖个人记忆 | 生产用 protected environment + Tag 触发，manual 只在真正需要人时 |

## 8.1 反模式对应的可执行纠正

```bash
# ① 找出"没 include 平台模板"的仓库（ci_config_path 只是本项目内文件路径，不能用来判断复用情况）
for p in $(glab api "groups/payments/projects?per_page=100" | jq -r '.[].path_with_namespace'); do
  enc=$(printf '%s' "$p" | sed 's#/#%2F#')
  body=$(glab api "projects/$enc/repository/files/.gitlab-ci.yml/raw?ref=main" 2>/dev/null || true)
  printf '%s' "$body" | grep -q 'platform/ci-templates' || echo "MISSING-TEMPLATE  $p"
done
# 注：默认分支不是 main 的项目要换成 .ci_config_path 或先读 default_branch；
#     更稳的口径是拉 merged config（见 5.3 的 ci/lint 端点），因为它能展开多层 include。

# ② main 未保护 / 未禁强推的项目（protected_branches 返回字段以现值为准）
for p in $(glab api "groups/payments/projects?per_page=100" | jq -r '.[].path_with_namespace'); do
  n=$(glab api "projects/$p/protected_branches" | jq '[.[] | select(.name=="main")] | length')
  [ "$n" = "0" ] && echo "UNPROTECTED  $p"
done
glab api "projects/$P/protected_branches" \
  | jq -r '.[] | "\(.name)\tforce=\(.allow_force_push)\tpush=\(.push_access_level)\tmerge=\(.merge_access_level)"'
# 审批要求不在 protected branch 上，而在审批规则上：
glab api "projects/$P/approval_rules" | jq -r '.[] | "\(.name)\t\(.approvals_required)\tusers=\(.users|length)"'

# ③ 令牌盘点：GET /personal_access_tokens 管理员可见全实例（非管理员只见自己）
glab api "personal_access_tokens?state=inactive&per_page=100" | jq -r '.[] | "\(.user_id)\t\(.name)\t\(.expires_at)\t\(.scopes)"'
glab api "personal_access_tokens?revoked=false&per_page=100" | jq -r '.[] | select(.expires_at==null) | "NO-EXPIRY \(.user_id)\t\(.name)"'
glab api "project_access_tokens" | jq -r '.[] | select(.expires_at==null) | "\(.name)\t\(.active)"'
glab api "group_access_tokens"   | jq -r '.[] | select(.expires_at==null) | "\(.name)\t\(.active)"'

# ④ latest 依赖盘点：用群组级 blob 搜索（全局 search 需要更高的搜索权限，容易 403）
glab api "groups/payments/search?scope=blobs&search=image%3A%20latest" | jq -r '.[].path'
```

> **收敛长期令牌的正确动作**：`expires_at` 建好之后**不能直接改**（PAT 只能撤销重建；project/group access token 支持 `POST .../rotate` 换发并带新过期时间）。所以"发现一个不过期的 token"= 通知属主撤销 + 用 rotate 或重建补一个 90 天有效期的。

> 上面这些脚本的价值在"每周跑一次并把输出开成 Issue"，不在单次执行。**治理 = 周期性检查 + 明确责任人**，否则第三周就没人再跑了。

# 9. 小结

| 阶段 | 关键动作 |
|------|---------|
| 定结构 | 群组层级与平台群组，一次定好 |
| 立规范 | 模板仓库固化 CI/Issue/MR/CODEOWNERS |
| 设门禁 | 分支保护 + MR 规则 + CI 必须绿 |
| 养习惯 | 流水线如代码、镜像化环境、快速反馈 |
| 兜底线 | 扫描左移、令牌最小化、备份演练 |

**上线自检清单（全部为"是"才算落地）**

- [ ] 新项目能由脚手架生成，含 issue/MR 模板、CODEOWNERS、`include` 平台模板且锁定 `ref`
- [ ] 默认分支保护、审批数、resolve 讨论、流水线必须绿四项已用 API 下发并回读确认
- [ ] 审批规则校验过候选审批人数量，无 `Auto approved` 情况
- [ ] Secret Detection 阻断、其余扫描分级放行，且 `allow_failure` 写明意图
- [ ] CI 内无个人 PAT；生产变量均 Protected + Masked，绑定 protected environment
- [ ] 备份任务有 cron，且最近一次恢复演练报告已归档
- [ ] 流水线 P90 时长、失败率、Runner 队列等待三项指标有可视化位置

规范篇收束"怎么用好"；下一篇处理"用坏了怎么办"——常见故障排查手册。
