# 1. 协作对象的层级：Issue → Board → Milestone → Epic

GitLab 的需求管理对象按"颗粒度从小到大"排：

| 对象 | 归属 | 用途 | 类比 |
|------|------|------|------|
| Issue | 项目 | 一个任务/缺陷/讨论 | Jira Story/Bug |
| 标签 Label | 群组/项目 | 分类、优先级、状态 | 优先级 P0、类型 bug |
| 里程碑 Milestone | 群组/项目 | 版本/迭代时间盒 + 燃尽图 | Sprint / Release |
| Board 看板 | 项目/群组 | Issue 的可视化流转 | Scrum/Kanban 板 |
| Epic | 群组（Premium） | 跨项目的长期主题 | 史诗/OKR 载体 |
| Roadmap 路线图 | 群组（Premium） | Epic 按季度排布 | 产品路标 |

**设计考量**：GitLab 原生 Issue 与 Jira 的差距不在"能不能记任务"，而在**流程可配置性**（工作流引擎、字段自定义）。选型经验：以交付为中心、Issue 即工作项 → 用 GitLab；重流程审批、跨部门工单 → Jira 集成 GitLab（MR 合并自动同步 Jira 状态）。

# 2. Issue 的正确打开方式

Issue 不只是"提个 bug"，它的结构化能力来自三个配置：

**Issue 模板**（仓库 `.gitlab/issue_templates/Bug.md`）：

```markdown
<!-- .gitlab/issue_templates/Bug.md -->
## 现象
<!-- 必现/偶现，复现步骤 -->
1.

## 环境
- 版本:
- 环境: dev/staging/prod

## 期望行为


/label ~bug ~"needs-triage"
/assign @oncall-backend
```

模板底部的 **quick actions**（`/label`、`/assign`、`/milestone`、`/duplicate`）会自动执行，统一入口比"口头规范"有效得多。

**标签体系**建议三条正交维度，不要混在一条线上：

| 维度 | 示例 | 说明 |
|------|------|------|
| 类型 | `bug` `feature` `chore` | 决定 Issue 性质 |
| 状态 | `needs-triage` `in-review` `blocked` | 看板列按它映射 |
| 优先级/领域 | `P0` `backend` `security` | 过滤与分配 |

**快速操作（Quick Actions）**在任意评论里生效：`/assign me`、`/label ~bug`、`/close`、`/cp P1`（复制到项目）……用 `/**` 可现场补全，重复劳动全部转成指令。

# 3. Board：看板的组织方式

项目/群组的 Issue Board 支持两种布局：

- **Label-based board**：每列一个标签，Issue 拖列 = 改状态标签。适合"研发流"（Open → Doing → Review → Done）。
- **Status-based board**（16.x 起）：按 GitLab 内置状态（Open/Triage/Closed/...）布局，列内再按标签分组，适合不想维护一堆状态标签的团队。

群组 Board 聚合子项目 Issue（跨项目看全团队工作量）；**Issue 与 MR 在看板上同屏**（MR 视图切换），评审积压直接可见。

> **注意**：Board 列的"拖动改标签"有冲突规则——一个列组内互斥标签集（如状态类），避免一个 Issue 同时出现在两列。

# 4. Milestone 与 Epic：时间维度的组织

- **Milestone**：设定起止日期 + 目标描述，页面自带**燃尽图**；挂在群组上的 milestone 聚合全部子项目。发版时"本版关闭的 Issue 数/未完成顺延数"用它统计。
- **Epic（群组 Premium）**：比 Issue 高一层的容器，Epic 下挂各项目的 Issue；有父子 Epic、时间线视图。Roadmap 页按季度展示 Epic 进度——产品负责人视角。

**层级关系**：

```text
群组 Epic: 支付体系重构
 ├── 项目A Issue: 新增部分退款接口   ← Milestone: 26Q4
 │     └── MR !128（合并后自动 progress → closed）
 ├── 项目B Issue: 前端退款页
 └── 项目C Issue: 对账任务改造
```

# 5. Issue ↔ MR/分支 的双向绑定

这是 GitLab 相对独立工单系统最大的优势——**需求与代码互相可见**：

- 分支名 `issue/123-add-refund` 创建 MR 时自动关联 Issue。
- MR 描述里 `Closes #123` 合并即关闭；多个 `Closes #123, #124`。
- Issue 页右侧 "Merge Requests" 区块实时显示关联 MR 的状态（open/merged/closed），产品看进度不用问研发。
- 从 Issue 页直接"创建 MR"（带 issue 引用），保证每个 MR 有需求锚点。

# 6. Wiki 与文档

- 每个项目自带 Git-backed Wiki（仓库为 `<project>.wiki.git`），支持 Markdown 页面、侧边栏、模板、Mermaid 图。
- 页面可在 Web 编辑，也可 **克隆 wiki 仓库本地批量编辑**（长文档推荐）。

**设计考量**：Wiki vs 仓库内 `docs/`：

| 位置 | 适合 | 理由 |
|------|------|------|
| `docs/` + MR 评审 | 架构决策（ADR）、接口契约 | 变更走评审，与代码同分支 |
| Wiki | 操作手册、新人入职、FAQ | 非开发角色可编辑，不怕 commit 权限 |

> **行业实践**：关键文档（设计文档、Runbook）放仓库 `docs/`，享受 MR 评审与版本对齐；杂项知识放 Wiki。**别把 Wiki 当 Confluence 用**——它没有细粒度页面权限与模板审批。

# 7. 其他高频协作能力

- **Snippet（代码片段）**：内部 Gist，评审/分享小段代码。
- **邮件收件**：项目可配置"邮件转发到专属地址自动建 Issue"（接客服/工单入口）。
- **导入导出**：JIRA CSV 导入脚本、GitHub Issue 迁移工具（官方 gitlab-org 有）。
- **健康状态**（自定义 Issue status 16.x）：Open/Preview/Triage/Investigation/Doing/Review/Blocked/Closed，看板可直接映射。

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 对象层级 | Issue（任务）→ Milestone（时间盒）→ Epic（主题，群组 Premium） |
| 模板化 | issue_templates + quick actions 把规范固化成表单 |
| 标签 | 类型/状态/优先级三维正交，避免标签爆炸 |
| 杀手锏 | Issue ↔ MR 双向自动关联，需求进度在代码侧实时可见 |
| 文档 | 契约类进 docs/ 走 MR，手册类进 Wiki |
| 选型 | 轻量到中度流程用原生；重审批工作流仍交给 Jira/ServiceNow 集成 |

协作侧齐了。接下来进入交付主线的另一半：安全扫描与权限治理（第 12 篇）。
