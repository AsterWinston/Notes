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

# 3. 评审：讨论、建议与审批

评审发生在三个层次：

**行级评论**：diff 任意行加评论；评审者用 ````suggestion` 代码块直接给出替换代码，作者一键应用——把"嘴上提意见"变成"按钮改代码"。

````markdown
```suggestion
    return List.copyOf(items);   // 用不可变副本替代直接返回内部列表
```
````

**审批（Approvals）**：MR 右侧 Approve 按钮；Premium 可配规则：

| 审批维度 | 配置位置 | 示例 |
|----------|---------|------|
| 最少审批数 | Settings → Merge requests | 至少 2 人 |
| 按分组/按用户 | Approval rules | 后端组至少 1 人 |
| Code Owner | CODEOWNERS + 规则 | 涉及支付路径必须 Owner 批 |
| 防止"批准后又改" | Reset approvals on push | 新 commit 清空已有审批 |
| 报告人不能自批 | 审批规则选项 | 开启 |

**状态讨论（threads）**：未解决的讨论线程不关掉就不能合并（设置项 resolve all discussions），保证每个意见都有结论。

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

# 5. 变更日志与合并信息

GitLab 自动把 MR 标题/描述汇总为 Release 的 changelog（Release notes from MR）。约定式提交/规范标题收益在这里显现：

- 团队约定 MR 标题格式：`feat(order): 支持部分退款`，Release 页一键生成可读变更说明。
- 合并 commit 消息模板可在群组/项目设置统一（`$TITLE` 占位符）。

# 6. 与 Issue 的联动

- `Closes #123` / `Fixes` / `Implements` 等关键字，合并即关闭对应 Issue；一个 MR 可关多个。
- Issue 页面显示"关联 MR"及其实时状态，产品/测试不用追代码。
- Board 列可按 MR 状态流转（需 Premium 的 Issue boards 高级布局；MR 本身也可看板，见后续篇）。

# 7. 高效使用 MR 的清单

| 实践 | 理由 |
|------|------|
| 小 MR + Draft 早评审 | 评审质量与速度 |
| MR 描述写模板（测试步骤、影响面、回滚方式） | Issue template 同理，降低沟通成本 |
| 开启 squash + 删源分支 | 历史干净 |
| CI 必须绿 + 讨论必须 resolve 才能合 | 流程不靠自觉 |
| 避免大"合并 MR"混入功能 MR | 先单独合 main 的 MR，再评审功能 MR |
| 用 Cherry pick 按钮回补 hotfix | GitLab 原生支持把 MR 挑到 release 分支 |

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 定位 | MR = 评审 + 门禁 + 发布上下文的统一工作流单元 |
| 评审三层 | 行级建议（suggestion）、审批规则、讨论线程 |
| 门禁 | 保护分支 + Status checks + Draft + 审批规则 |
| CI 衔接 | MR 流水线验证"合并结果"，Merge train 解决并发合并 |
| 联动 | Closes 关键字让 Issue 随合并自动关闭 |

MR 讲完，协作侧还剩 Issue/看板/路线图（第 11 篇）；接下来进入 GitLab 最有分量的部分——CI/CD，从流水线的基本构成与 `.gitlab-ci.yml` 开始。
