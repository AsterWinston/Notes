# 1. MR 是什么：评审 + 合并 + 交付的交汇点

Merge Request 表面上是"请求把一个分支合进另一个分支"，在 GitLab 里它实际是**一个工作流单元**：

```text
开发者: feature 分支 push ──► 创建 MR ──► 自动触发 Pipeline
                                        │
评审者: 逐行评论 / 建议（可一键应用） ◄──┤ 看 diff、看 CI、看安全扫描
                                        │
门禁:   审批数 ✔  CI 绿 ✔  无冲突 ✔ ──►  合并（可删源分支）──► 触发下游
```

- MR 页面聚合了讨论、代码变更、流水线、审批、安全报告、关联 Issue——**所有上下文不离开这个页面**。
- 关联 Issue 写 `Closes #123`，合并时自动关闭 Issue，这是把项目管理挂到交付流上的钩子。

## 1.1 三个容易混的角色

| 角色 | 字段 | 含义 | 门禁影响 |
|------|------|------|---------|
| Author | 创建者 | 提交变更的人 | 可配置为不能自批 |
| **Assignee** | Assignees | **干活的人**（谁来改代码/推进合并） | 无 |
| **Reviewer** | Reviewers | **该评审的人**（专门提醒，页面有 review 状态） | 可配置为审批候选人来源 |
| Approver | Approvals 列表 | 已行使审批权的人 | 数量受 approval rules 约束 |

一个常见误区：把"指派给谁评审"填进 Assignee，结果提醒发给了写代码的人。配置自动化时也应区分：`/assign` 与 `/review` 是两个不同的快捷操作。

## 1.2 MR 生命周期与状态语义

```text
open ──► (Draft ⇄ Ready) ──► 评审/CI/审批并行推进
  │                              │
  │                    门禁全部满足 → merged（可选删源分支）
  ├──► closed（放弃，或被另一个 MR 取代）
  └──► 陈旧（长期无活动，MR 列表按"stale"标记）
合并瞬间的连带效果：Closes 关键字关 Issue、触发下游流水线/父子流水线、写入 Release Notes、
                 删除源分支后 MR 仍可查（diff 来自版本快照）
```

`Draft:` 前缀（旧称 WIP）由 GitLab 自动识别，也可用快捷操作切换：`/draft`、`/ready`。Draft 状态下**不显示合并按钮**，但仍可评审、可跑流水线——这正是"提前求方向"的价值。

# 2. MR 的创建与关键选项

push feature 分支后 GitLab 会提示快速建 MR（或 Web/CLI 创建）。创建时的几个选择决定后续体验：

| 选项 | 含义 | 建议 |
|------|------|------|
| Target branch | 合入目标，通常是默认分支 | 别合进别人的 feature 分支 |
| **Draft 标记**（原 WIP） | 标题加 `Draft:` 前，门禁允许提前评审但禁止合并 | 大改动先 Draft 求方向反馈 |
| Squash commits | 建议开启，合并时压成一个 commit | 保持主干历史干净 |
| Delete source branch | 合并后自动删分支 | 默认勾选 |
| 指派 Assignee/Reviewer | 干活的 / 评审的（不同角色） | 明确到人，Reviewer 会收专门提醒 |
| Labels/Milestone | 分类与进度追踪 | 配合 Board 使用 |

> **行业实践**：团队普遍约定 **MR 规模上限**（如 diff ≤ 400 行）。小 MR 是评审质量的前提，GitLab 的 Stack MR（基于另一 MR 分支的链式 MR）适合大特性拆解，但会增加复杂度，慎用。

## 2.1 从命令行一次建好（push options）

日常最省事的路径是把 MR 元数据随 push 一起提交，避免"分支推上去却忘了建 MR"：

```bash
git push -u origin feat/partial-refund \
  -o merge_request.create \
  -o merge_request.target=main \
  -o merge_request.title="feat(order): 支持部分退款" \
  -o merge_request.reviewers=@tech-lead \
  -o merge_request.label="area::payments" \
  -o merge_request.assign \
  -o merge_request.remove_source_branch
```

| push option | 用途 | 备注 |
|-------------|------|------|
| `merge_request.create` | 建 MR | 已存在则不重复创建 |
| `merge_request.target=<branch>` | 目标分支 | 跨项目 fork 场景常用 |
| `merge_request.title` / `.description` | 填标题/正文 | 会覆盖模板占位内容 |
| `merge_request.reviewers` / `.assign` / `.unassign` | 指派与评审人 | 多值可重复该选项 |
| `merge_request.label` / `.milestone` | 打标签、挂里程碑 | 与 Board/Milestone 联动 |
| `merge_request.set_to_draft` | 直接建成 Draft | 大改动推荐 |
| `merge_request.remove_source_branch` | 合并后删分支 | 也可在页面勾选 |

也可全程用 CLI：`glab mr create --fill --reviewer @tech-lead --label "area::payments"`、`glab mr list`、`glab mr note -m "..."`。

## 2.2 MR 模板

放多个模板时，创建页面会出现**模板下拉框**，按场景选择而不是"一个 Default.md 包打天下"：

```text
.gitlab/merge_request_templates/
├── Default.md          # 常规特性
├── Bugfix.md           # 必填：复现步骤、回归测试范围
├── Hotfix.md           # 必填：影响面、回滚方式、上线后观察指标
└── Dependency.md       # Renovate 类：变更内容、漏洞编号、验证方式
```

```markdown
<!-- .gitlab/merge_request_templates/Hotfix.md -->
## 现象与根因
- 报错原文：
- 关联 Incident：#

## 修复
- 为什么这样改（一行讲清因果，不要复述 diff）：

## 回归与观察
- [ ] 已补一条能重现该故障的测试
- 观察指标与阈值：error_rate、p99、队列延迟

/label ~"type::hotfix"
/review @oncall
```

> MR 模板只支持 Markdown；Issue 才支持结构化 `form` YAML 模板（第 11 篇），不要把两者语法混用。

## 2.3 标题与分支命名

```text
feat(order): 支持部分退款          ← 约定式标题，Release Notes 直接可用
fix(payment): 修复超时未回滚
refactor(db): 拆出 OrderRepository
chore(deps): 升级 spring-boot 3.4.2

分支：<type>/<简短描述>  例  feat/partial-refund
禁止：个人拼音缩写、日期串（rebase 与筛选成本极高）
```

标题格式约定收益在第 5 节显现；如果团队不守，用 CI 校验比培训有效（`commitlint` 类检查放在 MR 流水线里跑 `git log --format=%s origin/main..HEAD`）。

# 3. 评审：讨论、建议与审批

评审发生在三个层次：

**行级评论**：diff 任意行加评论；评审者用 ````suggestion` 代码块直接给出替换代码，作者一键应用——把"嘴上提意见"变成"按钮改代码"。

````markdown
```suggestion
    return List.copyOf(items);   // 用不可变副本替代直接返回内部列表
```
````

## 3.1 建议（suggestion）的正确用法

| 场景 | 写法 | 说明 |
|------|------|------|
| 替换被评论的那一行 | ` ```suggestion ... ``` ` | 偏移量默认 0，最简单 |
| 替换连续多行 | ` ```suggestion:-0+3 ... ``` ` | `-0` 从锚定行开始，`+3` 向下覆盖 3 行 |
| 在行间插入新行 | 评论锚定在**上一行**，suggestion 里包含原内容 + 新增行 | 直接贴新代码会覆盖原行 |
| 一次提交多条建议 | MR 页 "Commit N suggestions" | 建议可批量落到同一 commit，减少来回 |
| 建议不生效 | 检查评论是否发在"Changes"行的上下文中（非文件正文） | 只有 diff 行能挂 suggestion |

```markdown
评审留言 + 建议可以同一条评论完成：

这个分支没有处理 refundAmount 为 null 的情况，建议改成：

```suggestion:-0+1
    if (refundAmount == null) { throw new IllegalArgumentException("refundAmount required"); }
```

顺手把日志级别也降下来（另一条评论）。
```

权限要点：只有对源分支有写权限的人能 Apply suggestion，外部贡献者（fork MR）的建议只能由 Maintainer 应用；因此评审者若要直接改对方分支，需在创建 MR 时保留 **"Allow commits from members who can merge to the target branch"**（跨项目 MR 时由作者勾选）。

**建议之外**：讨论可 resolve/unresolve（resolve 后仍可追加评论），可 `Re-request a review` 提醒漏看的人，评审者可 **Request changes**（要求修改后重新评审）——该状态会阻止合并，比"口头说不要合"可靠。

## 3.2 审批（Approvals）与规则设计

MR 右侧 Approve 按钮；Premium/Ultimate 可配规则，Free 层的审批只是"记录"（Developer 以上都能批，且**不阻断合并**）：

| 审批维度 | 配置位置 | 示例 |
|----------|---------|------|
| 最少审批数 | Settings → Merge requests → Merge request approvals | 至少 2 人 |
| 按分组/按用户 | Approval rules | 后端组至少 1 人 |
| Code Owner | CODEOWNERS + 规则 | 涉及支付路径必须 Owner 批 |
| 防止"批准后又改" | Reset approvals on push（作者推新提交清空审批） | 开启 |
| 报告人不能自批 | Prevent approval by merge request creator | 开启 |

```bash
P='payments%2Forder-service'
# 规则用 API 下发，才能进 IaC/审计（页面手点无法批量保证一致）
glab api --method POST "projects/$P/approval_rules" \
  -f name='security-team' -f approvals_required=1 -f 'group_ids[]=42'
glab api --method POST "projects/$P/approval_rules" \
  -f name='db-changes' -f approvals_required=1 -f 'user_ids[]=108'
# 项目级门禁开关（字段名以当前版本为准，改前先读回现值）
glab api --method PUT "projects/$P/approval_settings" \
  -f prevent_author_approval=true \
  -f prevent_committer_approval=true \
  -f reset_approvals_on_push=true \
  -f disable_overriding_approvers_per_merge_request=true
glab api "projects/$P/approval_rules" | jq -r '.[] | "\(.name)\tneed=\(.approvals_required)\ttype=\(.rule_type)"'
```

## 3.3 审批静默失效的四个场景

这是最容易踩的一类问题：**门禁看起来配了，实际没生效**。

| 场景 | 表现 | 判据命令 | 纠正 |
|------|------|---------|------|
| 规则候选人为空/只有作者 | 规则显示 **Auto approved**，MR 可直接合并 | `glab api projects/$P/merge_requests/!42/approval_state` | 修 `CODEOWNERS` 路径或规则成员；实例级规则兜底 |
| 要求审批数 > 候选人 | 同上（Auto approved） | 看 `eligible_approver_count` | 降数量或加人 |
| MR 页面被临时改规则 | 提交人自己加为审批人 | `disable_overriding_approvers_per_merge_request` | 群组/实例级规则覆盖 |
| 只在受保护分支生效 | 非保护分支合并不卡 | 规则的 `protected_branch_id` | 目标分支纳入保护 |

> 治理做法：把"是否存在 Auto approved 规则"做成每日巡检脚本（第 15 篇有下发校验样例），比任何培训都有效。

## 3.4 讨论线程的语义

- **Resolve all discussions** 打开后：任何未 resolve 的线程都会挡住合并按钮。
- resolve 权限：默认评论作者与参与者都能 resolve（可收紧为仅 Maintainer/Owner），因此"由提意见的人自己关掉讨论"是可以配置的。
- 线程状态不等于审批：一条 resolve 的评论表示"有结论"，不表示"已改代码"。真正的代码变更约束靠 suggestion + CI + 审批三件套。

# 4. CI 门禁：流水线如何卡住合并

MR 的质量门禁由 CI 驱动，链路是：`MR 创建/更新 → 触发 MR 流水线 → 结果回显到 MR 页面 → 合并按钮受 Status checks 约束`。

```yaml
# .gitlab-ci.yml 关键机制：workflow 控制何时跑
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"   # 只为 MR 建流水线（推荐）
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH        # 主干也跑

job:
  stage: test
  script: mvn -B verify
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

- **MR 流水线 vs 分支流水线**：推荐用 `merge_request_event`，在合并**前**验证"合进去的结果"（GitLab 会自动在 MR 源分支 + 目标分支的合并结果上跑）。
- 必须绿的 job 不要标 `allow_failure: true`，否则形同虚设。
- **Merge train（Premium）**：多个待合并 MR 排队，每个都基于前序合并后的结果重新跑流水线再入队，根治"各自都绿、合完就挂"的并发合并问题。

## 4.1 去重：别让同一份改动跑两遍

只写 `merge_request_event` 会漏掉直接 push 分支的场景；两者都写又常见"双流水线"。标准写法是**有 MR 时只跑 MR 流水线**：

```yaml
workflow:
  rules:
    # 有开着的 MR：只建 MR 流水线，分支流水线让路
    - if: $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

判据变量：`$CI_PIPELINE_SOURCE`（`merge_request_event` / `push` / `schedule` / `web` / `trigger` / `pipeline`）、`$CI_MERGE_REQUEST_IID`、`$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME`、`$CI_MERGE_REQUEST_TARGET_BRANCH_NAME`。

```bash
# 排错：这条流水线到底为什么建/为什么 job 为空
glab api "projects/$P/pipelines" | jq -r '.[0] | .source, .ref'
# MR 页每个 job 都有"此 job 为何未运行"的判定入口（rules 求值结果），先读它再改 yml
```

MR 流水线跑在 GitLab 生成的 `refs/merge-requests/<iid>/…` 引用上，因此能发现"合并后才出现"的问题（例如目标分支新增了方法签名）。代价：**主干有新提交时 MR 需要更新（rebase/merge）才能变绿**，这正是"Merge when pipeline succeeds"与 Merge Train 存在的原因。

## 4.2 Status checks 与"绿"的定义

| 卡点 | 配置 | 常见漏洞 |
|------|------|---------|
| 流水线必须成功 | 保护分支 → Pipelines must succeed | 关键 job 被 `allow_failure: true` 降级，绿灯假象 |
| 讨论全部 resolve | Settings → Merge requests | 用"先 resolve 再改代码"绕过 |
| 审批数达标 | Approval rules | 3.3 节的 Auto approved |
| Code Owner 审批 | CODEOWNERS + 规则 | 路径写错 → 规则无人命中 |
| 无冲突 | 合并前必须解决（Web IDE 可解冲突） | 长期陈旧 MR 积累冲突 |
| Draft 状态 | Draft 禁合并 | 忘记 `/ready` 导致过期评审失效 |
| 安全策略门禁 | Ultimate：漏洞等级触发审批/阻断 | 未配 policy 时扫描只出报告 |

```yaml
# 让"必须绿"真正成立：关键 job 显式不允许失败
test-core:
  stage: test
  script: mvn -B verify
  allow_failure: false          # 默认即 false，写出来是为了评审时一眼确认
sast-advisory:
  stage: test
  allow_failure: true           # 存量治理期的降级必须写注释说明意图与终态
```

## 4.3 自动合并：MWPS 与 Merge Train 的分工

| 机制 | 语义 | 适用 |
|------|------|------|
| Merge when pipeline succeeds (MWPS) | 现在审批够了但流水线还没跑完，预约"绿了就合" | 单 MR、无并发冲突风险 |
| Merge Train | 排队，每个 MR 在"前序已合并"的结果上重跑流水线 | 高并发主干、微服务仓库、必须绿的共享库 |
| 手动合并 | 人点按钮 | 生产发布、变更窗口受控场景 |

Merge Train 前置条件（缺一则不入队）：使用 MR 流水线、勾选 **Pipelines must succeed**、开启 Merge train 设置、审批与讨论已满足。入队方式：MR 页 **Merge train** 按钮，或快捷操作 `/auto-merge`（MWPS）。

```bash
glab api "projects/$P/merge_requests/42/merge_train" | jq -r '.[] | "\(.merge_request.iid)\t\(.status)\t\(.pipeline.status)"'
```

## 4.4 从 CLI 完成一次完整评审闭环

```bash
glab mr list --reviewer=@me                      # 我的待评审队列
glab mr view 42 --output json | jq '{title, approvals_left:.approvals_left, pipeline:.head_pipeline.status}'
glab mr diff 42 | head -40                       # 终端先扫一遍
glab ci status --branch feat/partial-refund      # 看这条 MR 的流水线
glab mr approve 42
glab mr note 42 -m 'suggestion 见第 87 行；另外补一条 null 分支的测试'
glab mr merge 42 --when-pipeline-succeeds --squash --delete-branch
```

# 5. 变更日志与合并信息

GitLab 自动把 MR 标题/描述汇总为 Release 的 changelog（Release notes from MR）。约定式提交/规范标题收益在这里显现：

- 团队约定 MR 标题格式：`feat(order): 支持部分退款`，Release 页一键生成可读变更说明。
- 合并 commit 消息模板可在群组/项目设置统一（`$TITLE` 占位符）。

## 5.1 四种合并方式的取舍

| `merge_method` | 历史形态 | 优点 | 代价 |
|----------------|---------|------|------|
| Merge commit（默认） | 保留分支与合并节点 | 追溯完整 | 主干图形噪声大；回滚需 revert 合并节点 |
| Fast-forward (`ff`) | 线性，无合并节点 | 主干可读性最好 | 要求 MR 与目标分支同步，冲突/陈旧需先 rebase（**rebase 会重置审批**） |
| Merge with squash | 一个 MR 一个 commit | 主干历史最干净、回滚最容易 | 丢失中间过程；需配置 squash commit 格式 |
| `ff-squash` | 线性 + 压缩 | 上述两者优点 | 同样需要 MR 保持最新 |

推荐基线：**ff/ff-squash + squash 标题即 MR 标题**，回滚策略因此退化为 `git revert` 单 commit。

合并提交的文案模板在 **Admin/群组 → Settings → Repository → Merge commit message template**（以及 squash 模板）统一，可用 `$TITLE`、`$DESCRIPTION`、`$IID`、`$LINK` 占位符；把模板与约定式标题绑定，第 5.3 节的 Release Notes 才有稳定输入。

```bash
glab api --method PUT "projects/$P" -f merge_method=ff -f squash_option=enable
glab api --method PUT "projects/$P/merge_requests/42" -f squash=true      # 单条 MR 决定是否压缩
```

## 5.2 用 `!fixup` 修补已合并的提交

squash 前发现某个 commit 有问题，不用交互式 rebase：

```bash
# 把修补提交自动 fixup 到指定 commit（GitLab 原生支持的 commit 前缀）
git commit -m '!fixup 1a2b3c4d 修正空指针'
git push
```

## 5.3 Release Notes 自动化

```yaml
# .gitlab-ci.yml —— 用 Semantic Release 由 MR 标题驱动版本与 Release Notes
release:
  stage: release
  image: node:22-alpine
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - npx semantic-release
```

```json
// release.config.json —— 走约定式提交，MR 标题即变更日志条目来源
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/gitlab", { "assets": [{ "path": "dist/*.jar", "label": "JAR" }] }]
  ]
}
```

不用第三方工具时，Release 页勾选 **Selecting commits manually / Release notes from MR**，把 MR 标题聚合成草稿同样有效（第 15 篇的发布基线即此）。

# 6. 与 Issue 的联动

- `Closes #123` / `Fixes` / `Implements` 等关键字，合并即关闭对应 Issue；一个 MR 可关多个。
- Issue 页面显示"关联 MR"及其实时状态，产品/测试不用追代码。
- Board 列可按 MR 状态流转（需 Premium 的 Issue boards 高级布局；MR 本身也可看板，见后续篇）。

## 6.1 关键字与引用写法全表

| 写法 | 效果 |
|------|------|
| `Closes #123` / `Fixes` / `Resolves` / `Implements` / `Refs`（正文或评论） | 合并时自动关闭 Issue；`Refs` 只关联不关闭 |
| `Closes group/proj#45` | 跨项目关闭 |
| `Closes #123,#124,#125` | 一次关多个 |
| `/link #123`、`/related #123` | 仅建立"相关"链接，不改变状态 |
| `/blocks #123`、`/is-blocked-by #123` | 依赖关系，Issue 页显示阻塞链 |
| `/duplicates #123` | 标记重复并自动关联 |
| `/copy #123` | 把某 Issue 的标签/里程碑/指派复制过来 |
| `/close`、`/reopen` | 直接改状态 |
| `/target_branch release/2.14` | 改合并目标分支 |

**关键字只认 MR 描述里的写法才会"合并即关"**；写在评论里同样生效，但"合并后才关闭"的时序与 squash 后的 commit 关联要留意：squash 时若模板里丢掉了 `Closes` 行，联动就断了——这是把模板固定下来的又一个理由。

## 6.2 快捷操作速查（评审日常）

| 目的 | 快捷操作 |
|------|---------|
| 切换 Draft | `/draft`、`/ready` |
| 指派/评审 | `/assign @me`、`/review @tech-lead`、`/unassign` |
| 审批 | `/approve`、`/revoke_approval @user` |
| 合并 | `/merge`、`/auto-merge on`（= MWPS）、`/auto-merge off` |
| 历史与同步 | `/rebase`、`/squash on` |
| CI | `/pipeline`、`/retry_failed`、`/cancel_previous_pipelines` |
| 元数据 | `/label ~"area::x"`、`/milestone %2.14`、`/due 2026-12-01`、`/title 新标题` |
| 工时 | `/start_spent_time`、`/stop_spent_time`、`/set_estimate 4h` |
| 关联 | `/link #123`、`/blocks #123`、`/copy #123` |

# 7. 高效使用 MR 的清单

| 实践 | 理由 |
|------|------|
| 小 MR + Draft 早评审 | 评审质量与速度 |
| MR 描述写模板（测试步骤、影响面、回滚方式） | Issue template 同理，降低沟通成本 |
| 开启 squash + 删源分支 | 历史干净 |
| CI 必须绿 + 讨论必须 resolve 才能合 | 流程不靠自觉 |
| 避免大"合并 MR"混入功能 MR | 先单独合 main 的 MR，再评审功能 MR |
| 用 Cherry pick 按钮回补 hotfix | GitLab 原生支持把 MR 挑到 release 分支 |

## 7.1 "无法合并"的排查表（报错原文 → 动作）

| MR 页提示 | 根因 | 动作 |
|----------|------|------|
| `The pipeline failed` / 合并按钮灰 | 流水线未通过（或从未跑） | `glab ci status`；检查 `workflow:rules` 是否命中 |
| `You do not have permission to push to the protected branch` | 没有该项目的 push 权限 | 由 Maintainer 执行合并，或授予 Developer 并配审批 |
| `The source branch is gone` 后 diff 异常 | 合并前删了分支（历史仍在） | 正常，若需继续改必须重开分支新建 MR |
| `Merge request must be approved by code owner` | CODEOWNERS 命中路径 | `glab mr view <iid> --output json` 查 rules 与候选审批人 |
| `All discussions must be resolved` | 有未 resolve 线程 | 回评论给结论后 resolve，不要盲点 |
| `Merge conflicts must be resolved` | 主干有新提交导致冲突 | "Resolve conflicts" tab 或本地 rebase + push（注意审批重置） |
| `Approvals are out of date` | 批准后作者又推了新提交 | 重新审批（这是 `reset_approvals_on_push` 的预期行为） |
| 只能入队不能立即合 | Merge Train 排队中 | 查 `merge_train` 状态；前序 MR 变绿后自动推进 |
| `Draft` / `WIP` | 仍是草稿 | `/ready` |
| 提示需要签名提交 | 项目要求 GPG/SSH 签名提交 | 配置签名密钥后重推 |
| 长时间 MR 变 `stale`/提示"需要更新" | MR 与目标分支偏离过远 | 定期 rebase，或拆分 MR |

## 7.2 什么算健康的 MR 流程

| 指标 | 健康区间 | 不健康信号 | 处置 |
|------|---------|-----------|------|
| MR 创建 → 首次评审响应 | < 4 小时（工作时间） | 长期 > 1 天 | 评审轮值 / 自动指派（`/review`、CODEOWNERS、OAuth 提醒） |
| diff 规模 | ≤ 400 行 | 大量 > 1000 行 MR | 约定拆分规则，Stack MR |
| MR 平均存活时间 | < 2 天 | 周级陈旧 MR | 先合主干（trunk-based），减少长分支 |
| 合并失败率（入队后被挤出） | 低 | Merge Train 频繁重跑 | 冲突源集中：拆模块、减少同时改共享文件 |
| rebase 导致审批重置次数 | 少 | 大量"审批后又推 commit" | 要求本地先同步再提 MR |

> **反模式**：把 MR 当作"提交前的最后检查点"（写三天代码开一个 5000 行 MR）。这类 MR 的评审质量必然为 0，审批沦为形式，且让 4.1/4.2 的所有门禁失去意义。

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 定位 | MR = 评审 + 门禁 + 发布上下文的统一工作流单元 |
| 评审三层 | 行级建议（suggestion）、审批规则、讨论线程 |
| 门禁 | 保护分支 + Status checks + Draft + 审批规则 |
| CI 衔接 | MR 流水线验证"合并结果"，`$CI_OPEN_MERGE_REQUESTS` 去重，Merge train 解决并发合并 |
| 联动 | Closes 关键字让 Issue 随合并自动关闭 |
| 静默失效 | 审批候选人不足会 **Auto approved**，必须巡检 |

**落地检查清单**

- [ ] MR 模板多份可下拉选择，且含回滚/验证必填项
- [ ] push options / `glab mr create` 已成团队习惯，分支推上去必有 MR
- [ ] `workflow:rules` 无重复流水线，关键 job 未误标 `allow_failure`
- [ ] 审批规则下发走 API，且验证过候选人数量与 Auto approved 情形
- [ ] 合并方式统一（ff/ff-squash + squash），回滚路径明确为 revert 单 commit
- [ ] Release Notes 由 MR 标题聚合，无需人工整理

MR 讲完，协作侧还剩 Issue/看板/路线图（第 11 篇）；接下来进入 GitLab 最有分量的部分——CI/CD，从流水线的基本构成与 `.gitlab-ci.yml` 开始。
