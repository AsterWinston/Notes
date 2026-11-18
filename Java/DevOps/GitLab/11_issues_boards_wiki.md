# 1. 协作对象全景：Work Item 才是现在的中心

> **前置**：[第 10 篇 制品仓库](10_container_registry.md)
> **本篇目标**：把"需求侧"讲透：工作项体系（Issue/Task/Epic/OKR/状态/看板/里程碑）、模板与 Quick Action 的规范化用法、Wiki 与 `docs/` 的分界、GLQL 动态视图
> **对应实际工作**：团队日常任务流转、迭代规划、跨项目进度对齐、内部文档治理

## 1.1 对象层级：颗粒度从小到大

| 对象 | 归属 | 用途 | 类比 |
|------|------|------|------|
| **Work Item（工作项）** | 项目/群组 | 所有可跟踪对象的统一抽象（Issue/Task/Epic/OKR/Incident/Service Desk/Test case） | Jira Issue Type |
| Issue | 项目 | 一个任务/缺陷/讨论 | Story / Bug |
| Task | 项目/群组 | 更小的工作单元（可挂到 Issue 下） | Sub-task |
| 标签 Label / **Scoped Label** | 群组/项目 | 分类、优先级、互斥状态（`key::value`） | `P0`、`workflow::in-review` |
| 里程碑 Milestone | 群组/项目 | 版本/迭代时间盒 + 燃尽燃上图 | Sprint / Release |
| 迭代 Iteration | 群组 | 固定周长的执行周期（与 Milestone 并行使用） | Sprint |
| Board 看板 | 项目/群组 | 工作项的可视化流转（列表 + 泳道） | Kanban 板 |
| Epic | **群组**（Premium/Ultimate） | 跨项目、跨迭代的长期主题 | 史诗 / OKR 载体 |
| Roadmap 路线图 | 群组 | Epic 按季度排布 | 产品路标 |

**设计考量**：GitLab 原生 Issue 与 Jira 的差距不在"能不能记任务"，而在**流程可配置性**（工作流引擎、字段自定义）。选型经验：以交付为中心、Issue 即工作项 → 用 GitLab；重流程审批、跨部门工单 → Jira/ServiceNow 集成 GitLab（MR 合并自动回写外部状态）。

## 1.2 Work Items 统一视图：入口和 URL 都变了

- **Work items 列表取代分离的 Issues / Epics 列表**：入口 **左栏 Plan → Work items**；曾经固定（pinned）的 Issues/Epics 会被替换为 Work items。
- **URL 归一**：`/-/issues/:iid`、`/-/epics/:iid` 会 **重定向到 `/-/work_items/:iid`**。脚本与文档里硬写老路径仍能用，但新代码建议直接用 work_items 路径与 API。
- 该统一规划视图在 **GitLab 18.10 GA**（18.7 引入时为功能开关），**Premium/Ultimate 还能把 Work items 直接以"按状态分列的 Board 视图"打开**（Plan → Work items → 右侧 Board/List 切换，按项目/群组记忆选择）。
- Epic 也已作为工作项存在（"Epics as work items"，GitLab 18.1 引入），因此 `/set_parent`、`/add_child` 这类层级操作是**跨类型通用**的。

> **对笔记/工具链的实际影响**：搜索栏、看板、API、快捷操作四处的名词正在从"issue"漂移到"work item"。写自动化时**优先用 API 而非爬页面**，并对字段名保留兼容分支。

## 1.3 与代码侧的咬合点（本篇只列，细节在第 5 篇）

```text
Issue/Task ──(创建分支 issue/123-xxx)──► 分支 ──► MR ──Closes #123──► 自动关闭 + 进度回写
    │                                                            │
    ├─ 状态（Status）/ 标签 / 里程碑 / 权重 / 健康状态  ◄──────────┘  流水线与部署信息在同一页面可见
    └─ 关联：relates_to / blocks / duplicate / parent-child
```

---

# 2. 工作项类型与自定义类型

## 2.1 内置类型清单

GitLab 支持的工作项类型：**Issues、Epics、Tasks、Objectives & Key Results（OKR）、Incidents、Service Desk tickets、Test cases**。Incident 打开的是自己的详情页（带告警、时间线等专属区块）。

## 2.2 自定义类型：上限、命名与状态

可在**顶层群组或 organization** 上配置工作项类型（Configurable work item types）：

| 规则 | 值 |
|------|----|
| 类型总数上限 | **每个顶层群组/组织 40 个**（含 GitLab 提供的类型） |
| 名称 | 命名空间内唯一，**≤48 字符**；不能与任何类型重名（含已归档/已禁用的） |
| 保留名（不可用） | `Vulnerability`、`merge request`、`commit`、`pipeline`、`job`、`alert`、`review`、`diff`、`report`、`SHA` |
| 类型状态 | **Enabled**（默认可建可筛）/ **Disabled**（不能新建、不出现在筛选，已有项不受影响，可改名换图标）/ **Archived**（从所有列表、筛选、创建流程移除，只能在顶层群组查看与 unarchive，不能改名）/ **Locked**（绑定特定用途） |
| 改名 | 改名后原名可被新类型使用；若原名未被占用可改回 |

## 2.3 类型转换：`/type` 与 `/promote_to`

```text
# 在 Issue/Task 的评论里直接写
/type task              # Issue → Task
/type issue             # Task → Issue（也可 /promote_to issue）
/type objective         # Key Result → Objective（等价 /promote_to objective）
/promote_to Epic        # Issue → Epic（/type Epic 等价）
/promote_to Incident    # Issue → Incident（/promote_to_incident 是简写）
/move group/sub/project [type:Key Result]   # 跨项目/群组移动，可同时换类型
```

**坑**：`/type` 与 `/promote_to` 不是同义词 —— **能转成什么取决于目标命名空间是否启用了该类型**，自定义类型名含空格时**不用加引号**（`]` 会终止取值）。转换失败最常见的表现是**快捷操作被静默忽略**（见 6.3）。

---

# 3. 状态（Status）：取代"状态标签"的正式机制

## 3.1 默认状态与状态类别

**默认状态**（不可修改）：`To do`、`In progress`、`Done`、`Won't do`、`Duplicate`。

每个状态归属五种类别之一：**Triage / To do / In progress / Done / Canceled**。关键语义：

> **落在 `Done` 或 `Canceled` 类别的状态会自动把工作项置为 closed；其它类别保持 open。**

也就是说："把卡片拖到 Done 列"不再只是改标签，它会**真的关闭工作项**；反过来把已关闭项拖回 `In progress` 列会**自动重开**。这是老团队最容易炸的地方（"我没点关闭它怎么自己关了"）。

## 3.2 Status vs Scoped Label：别用错工具

| 行为 | Status | Scoped labels |
|------|--------|---------------|
| 是否改变 open/closed | **会**（Done/Canceled 类别自动关闭） | **不会** |
| 配置位置 | **只在顶层群组配一次**，子群组与项目共用 | 任意项目/群组各自定义 |
| 单个工作项可有几个 | **一个状态** | 每个 scope 一个，可有多个 scope |
| 支持对象 | Issue、Task、Epic、Service Desk ticket | **MR** + 除 requirement 外所有工作项类型 |
| 订阅 | Premium/Ultimate | Premium/Ultimate |

**选择准则**：

- 状态代表"**这件事到了流程的哪一步**"，且希望它驱动 open/closed 与看板列 → 用 **Status**。
- 只是想给 MR/工作项打互斥属性（`platform::iOS`、`risk::high`），不希望影响状态 → 用 **Scoped Label**。
- 两者**都在企业版**，Free 版仍只能用普通标签当状态（那就必须接受"一个 Issue 同时在两列"的现实，见 7.4）。

## 3.3 Lifecycle 与自定义状态

- **Lifecycle（生命周期）**是"一组状态的集合"，可创建、改名、删除，并可**按工作项类型切换使用哪个生命周期**（例如 `Bug` 走 Triage→Investigation→Fix→Verify，`Chore` 走 To do→Done）。
- **自定义状态在顶层群组配置**；改动**状态类别**会有副作用（把一个已有状态从 `In progress` 改到 `Done`，会让所有挂该状态的工作项**变成 closed**）—— 改类别前先数影响面：

```bash
glab api "groups/$TOP_GROUP_ID" -o json >/dev/null   # 先确认操作对象是顶层群组
# 统计某状态下有多少工作项（字段名以现值为准，先读回再决定改不改）
glab api "projects/:id/issues?state=all&per_page=100" | jq 'length'
```

---

# 4. Issue 的正确打开方式

Issue 的结构化能力来自三个配置：**模板**、**标签**、**快捷操作**。

## 4.1 描述模板：`.gitlab/issue_templates/`

```markdown
<!-- .gitlab/issue_templates/Bug.md -->
## 现象
<!-- 必现/偶现，复现步骤 -->
1.

## 环境
- 版本:
- 环境: dev/staging/prod

## 期望行为

## 影响范围
- [ ] 影响线上 / [ ] 仅测试环境

/label ~bug ~"needs-triage"
/assign @oncall-backend
/due in:7d
```

- 模板底部的 **quick actions**（`/label`、`/assign`、`/milestone`、`/due`）会在提交时自动执行 —— **统一入口比口头规范有效得多**。
- 模板下拉在创建页右上角出现；文件名即模板名（`.md`，可放子目录，子目录会以 `目录/文件名` 形式出现）。
- **默认模板**：`.gitlab/issue_templates/Default.md`（**大小写不敏感**）自动预填。
- **优先级要注意**：`Default.md` **不覆盖**项目设置里配置的默认描述文本（后者仅 Premium/Ultimate：**Settings → Issues → Default description**，MR 侧是 **Settings → Merge requests → Default description template for merge requests**）。也就是说：**设置页里填了文本，仓库里的 Default.md 就被忽略**。

> **版本差异提醒**：早前的 **YAML 表单式 issue 模板**（`type: form` + `about`/`body`/`labels`）在现行官方文档里已不再是主推路径。新建"表单化"入口时，优先用 **Markdown 模板 + URL 预填参数（4.3）+ Quick Actions**；若你的实例仍支持 YAML 模板，以实例上 `/new` 页面的实际下拉为准。

## 4.2 实例级与群组级模板：一处维护、处处可用

| 层级 | 配置入口 | 前提 |
|------|----------|------|
| **实例级** | Admin 配置 **instance template repository**（`issue_template_project` 等 gitlab.rb 项） | Premium/Ultimate，Self-Managed/Dedicated |
| **群组级** | 群组 **Settings → General → Templates → Template repository** 选择模板项目 | Premium/Ultimate；**模板项目必须是该群组的直接子项目**；需 Owner 角色 |
| 项目级 | 仓库内 `.gitlab/issue_templates/`、`.gitlab/merge_request_templates/` | 全订阅 |

MR 默认模板还支持**变量替换**（仅对"默认模板"生效）：`%{all_commits}`、`%{issue}`、`%{issue_title}`、`%{issue_link}`、`%{recent_alerts}` 等 —— 保存 MR 时由 GitLab 替换。这是"从 Issue 一键起 MR 并自动带出需求描述"的正解（旧做法是 `%{...}` 拼在快捷操作里）。

## 4.3 用 URL 预填字段：给外部系统/客服入口用

```text
https://gitlab.example.com/<group>/<project>/-/work_items/new
    ?issue[title]=P1-%20支付超时
    &issue[description]=见%20Grafana%20看板
    &issuable_template=Bug
```

| 字段 | 参数 | 说明 |
|------|------|------|
| 标题 | `issue[title]` | 必须 URL 编码 |
| 类型 | `issue[issue_type]` | `incident` 或 `issue` |
| 模板（Issue/Incident/MR） | `issuable_template` | 传模板名 |
| 模板（Task/OKR/Issue/Epic） | `description_template` | 新类型走这个参数 |

**用法**：把这种链接放进企业微信/钉钉机器人、值班表、监控告警模板里，**保证每个入口产生的 Issue 结构一致**，比培训"请大家按模板填"有效。

## 4.4 邮件与 Service Desk 入口

- **邮件建 Issue**：需要实例配好 **incoming email**（子地址或 catch-all），项目里至少要有一个 Issue，操作者需 Guest/Planner/Reporter/Developer/Maintainer/Owner 角色。项目 Issues 页有 **Email issue to this project**（18.6 起由 "Email a new issue to this project" 改名）。
- **Service Desk（Premium/Ultimate）**：面向**外部报障人**，客户发邮件到项目专属地址即建工单，回复走邮件而不进项目 —— 与"邮件建 Issue"的关键区别是**权限边界**（外部人不需要成为项目成员）。

---

# 5. 标签体系：三条正交维度

## 5.1 维度设计

| 维度 | 示例 | 说明 |
|------|------|------|
| 类型 | `bug` `feature` `chore` | 决定 Issue 性质（现在更推荐用**工作项类型**承担这件事） |
| 状态 | `needs-triage` `in-review` `blocked` | 看板列映射（有 Status 能力后应迁到 Status） |
| 优先级/领域 | `P0` `backend` `security` | 过滤与分配 |

## 5.2 Scoped Label：互斥的真正实现方式

```bash
# 语法：key::value；同一 key 只能有一个 value
/label ~"workflow::in-review"        # 再加 workflow::done 时，旧的那个被自动移除
/label ~"platform::iOS" ~"risk::high"  # 不同 scope 可以并存
```

- **一个工作项不能同时有两个同 key 的 scoped label**：加新值会**替换**旧值 —— 这正是看板"一列一个状态"所需要的互斥保证。
- 过滤：搜索框输入 `platform::*` 命中该 scope 下任意值。**注意 dashboard（我的 Issue/MR 汇总页）不支持按 scoped label 过滤。**
- **保留 scope**：`status::` 等由系统保留（旧版还有 `pipeline::headful` 用于 Auto DevOps）；自定义 scope 前先确认不与内置冲突。
- 用 API 声明式设置标签（适合机器人/迁移脚本）：`POST /projects/:id/issues/:iid/set_labels`（**替换式**），区别于 `PUT .../labels`（**追加式**）。

## 5.3 标签爆炸的治理

- **禁止给每个模块/每个版本各造一个标签**：版本用 Milestone，模块用 `area::xxx`（scoped），人员用 Assignee/Reviewer。
- 定期合并同义标签：**Issues → Labels → 编辑**（改名称/颜色会同步到所有引用；删标签会从所有 Issue 移除）—— 合并前先跑一次"标签使用数"统计（第 15 篇有脚本）。
- 标签前缀约定：`type/`、`area/`、`risk/`，或用 scoped `key::value`；**同一套体系里不要两种风格混用**。

---

# 6. Quick Actions：把重复动作变成指令

## 6.1 高频清单（Issue/Task 侧）

```text
# 归属与调度
/assign @me @zhangsan        /unassign            /reassign @lisi
/due in:7d                   /remove_due_date     /milestone %"26Q4"
/iteration %26Q4W3           /set_parent #456     /add_child #789 #790
/promote_to Epic             /type task           /move group/proj [type:Key Result]

# 分类与度量
/label ~bug ~"high priority" /unlabel ~bug        /relabel ~old ~new
/status "In Progress"        /health_status 🟢    /weight 3   /clear_weight

# 关联与去重
/relate #123 group/proj#456  /blocks #123         /blocked_by #200
/duplicate #123              /copy_metadata #123  /confidential

# 时间与流程
/estimate 2h30m              /spend 45m           /todo
/close                       /reopen              /done
/internal_note               /subscribe           /title 新标题
/run_pipeline production     /create_merge_request
```

**要点**：

- **`/label` 的 `~` 可以省略，但两种写法不能混用**；含空格的标签名**必须加双引号**（`~"high priority"`）。
- 引用外部对象用**完整引用格式**：`#123`（本项）、`group/project#456`（跨项目）、`!789`（MR）、或直接贴 URL。
- **`/cc` 已在 GitLab 18.3 移除** —— 订阅请用 `/subscribe`（旧笔记里的 `/cc @user` 全部要替换）。
- **`/status` 只对 Issue 和 Task 生效**；参数取该命名空间已配置的状态值，例如 `/status "In Progress"`。
- Quick Actions 在**描述、评论、讨论回复**里都生效；`/internal_note` 让整条评论对非成员不可见（客服/事故场景必备）。
- 输入 `/` 或 `*` 触发自动补全（详见 Autocomplete characters 文档），**不要背参数格式，用补全生成**（补全会自动加引号并处理大小写）。

## 6.2 参数规则

- 手工输入的参数**必须用双引号包裹**，除非只含：ASCII 字母、数字、`_ - ? . & @`。
- **参数区分大小写**（用户名、标签名尤其）。
- 多个 Quick Action 可以连写，按出现顺序执行；**动作块之后的正文仍会保留**。

## 6.3 "为什么没生效"的官方答案

> 输入时看自动补全下拉里**是否出现该快捷操作**。若没出现，两种原因：
> ① **该能力受订阅层级或你在该群组/项目的角色限制**；② **前置条件不满足**（例如对没有任何标签的 Issue 执行 `/unlabel`）。

这条排查顺序省掉大量"我明明写了 /assign 但没人被指派"的来回：先看**拼写与参数格式**，再看**订阅/角色**，最后看**前置条件**（目标标签不存在、目标类型未在命名空间启用、状态名不匹配等）。

---

# 7. Board：看板的组织方式

## 7.1 列表（列）类型与卡片信息

| 列表类型 | 内容 | 备注 |
|----------|------|------|
| **Open**（默认） | 所有不属于其它列的**打开** Issue | 永远在最左 |
| **Closed**（默认） | 所有已关闭 Issue | 永远在最右 |
| **Label list** | 带某标签的打开 Issue | 最常用（Free 可用） |
| **Assignee list** | 指派给某人的打开 Issue | |
| **Milestone list** | 属于某里程碑的打开 Issue | |
| **Iteration list** | 属于某迭代的打开 Issue | Premium/Ultimate |
| **Status list** | 某**状态**的工作项 | **18.2 引入、18.4 GA**；与其它列不同：**可包含 open 与 closed**（取决于状态类别映射） |

卡片上可见：标题、标签、编号、指派人、权重、里程碑、迭代（Premium/Ultimate）、到期日、估算时间、**健康状态**。横向分组叫 **swimlane（泳道）**，例如按父 Epic 分组。

## 7.2 多看板、作用域与权限

```text
项目：所有订阅都可建多个看板
群组：Premium/Ultimate 才能多看板；Free 只有 1 个群组看板
创建/编辑看板、加列、移动卡片的前置角色：Planner / Reporter / Developer / Maintainer / Owner
看板数 ≥10 时，看板下拉里出现 Recent（最近访问 4 个）；再次进入自动打开上次看的看板
```

**Configurable boards（Premium/Ultimate）**：建板时可用 **milestone、labels、assignee、weight、iteration** 限定看板范围，从而"一块板 = 一个团队/一条流"。

> **必须知道的副作用**：**一旦看板被某个维度限定（例如绑定了 milestone），搜索栏里就不能再按该维度过滤** —— 想按里程碑筛，必须先把里程碑从看板作用域里移除。没编辑权限的人仍可通过 **Board configuration** 查看作用域设置。

## 7.3 WIP 限制与阻塞可视化（Premium/Ultimate）

- 每个列表可设 WIP 限制，两种口径：**Items**（条数）或 **Weight**（权重和，17.11 起）。
- 列表头显示 `当前/上限`（如 **4/3**、**8/5**），超限数字变红，列表内画一条分隔线把"线内/线外"分开。
- **默认的 Open / Closed 两个列表不能设 WIP 限制**。
- **Blocked issues**：被其它 Issue `blocked_by` 的卡片标题旁出现阻塞图标，悬停显示详情 —— 这是"看板上暴露依赖"的官方途径，比手写 `blocked` 标签可靠。
- 还有 **Sum of issue weights**（列头权重合计）用于容量评估。

## 7.4 拖动语义与性能（三条反直觉规则）

1. **在标签列之间拖动 = 移除来源列标签 + 添加目标列标签**（不是"追加"）。所以状态列必须做成互斥：普通标签做不到互斥 → 用 **scoped label** 或 **status list**。
2. **一个 Issue 有多个标签时会同时出现在多列**，这是"我的卡片怎么三列都有"的原因，不是 bug。
3. **性能与可见性**：每个列表默认只加载**前 20 条**，向下滚动再加载 20 条；列表**自动按标签填充**，不需要"手动加入"。
4. 从 **status list** 拖到普通 open 列表会应用**默认 open 状态**，拖到 closed 列表应用**默认 closed 状态**（可能顺带关闭/重开工作项）。
5. **Work items 的 Board 视图默认带 `State = Open` 过滤**，因此 `Done` 这类 closed 状态列看起来是空的 —— 要看到已关闭项，**先把 State 过滤器去掉**。

## 7.5 看板故障排查

| 现象 | 原因 | 处理 |
|------|------|------|
| 群组看板按 Author/Assignee 过滤时报 `There was a problem fetching users` | **你不是该群组成员**（非成员无权限列成员） | 把用户加入**顶层群组**（Guest/Planner/Reporter/…任一角色） |
| 看板打不开、一直转圈超时 | 卡片排序元数据（relative position）失衡 | 官方做法：Rails console 调 `Issues::RelativePositionRebalancingService.new(p.root_namespace.all_projects).execute`（先 `Project.find_by_full_path('<group>/<project>')`） |
| 卡片在列之间"跳回去" | 并发编辑/权限不足（低于 Planner 角色） | 核对角色与 `set_labels` 是否被 scoped 规则改写 |
| 新建 Issue 不出现在列里 | 不满足看板作用域（milestone/iteration/assignee/weight） | 看 **Board configuration**，补齐作用域属性 |
| WIP 数字不动 | Weight 未设置（用 Items 口径）或卡片不属该列 | 给 Issue 设 `weight` |

---

# 8. Milestone、Iteration、Epic：时间维度的组织

## 8.1 Milestone（Free 即可）

- **项目里程碑**只作用于该项目；**群组里程碑**作用于该群组下所有项目的 Issue/Epic/MR。
- 里程碑页含**燃尽图与燃上图**；侧栏显示：**完成度（已关闭工作项 ÷ 总数）**、起止与到期日、累计 time spent、**Issue 权重合计**、MR 总数/打开/关闭/已合并、关联 Release 链接、可复制的里程碑引用。
- **创建里程碑最低角色是 Planner**（17.7 起从 Reporter 下调）。
- 与 Release 关联后，可用里程碑生成发布证据（Release 页可看到里程碑收口情况）。

**发版统计实践**：`Milestone → 未关闭项数` 就是"顺延清单"，比手工汇总可信；把"版本 + 顺延"两件事都在里程碑上表达，别靠标签。

## 8.2 Epic（Premium/Ultimate，群组级）

- Epic 用于**跨项目、跨迭代**的大主题：拆解为可增量交付的小块、跟踪带起止日期的关联工作项、沉淀范围与决策讨论、构建层级结构、生成可视化路线图。
- **嵌套 Epic（子 Epic）是 Ultimate 能力**：复杂项目可拆多层，子 Epic 下再挂 Issue/Task。
- Epic 进度由子项**自动 progress**（关闭一定比例子项时页面显示进度条），所以**"Epic 手工拖到 100%"是没有约束力的做法** —— 保持子项真实关闭状态即可。

## 8.3 Iteration 与 Roadmap

- **Iteration**（群组级，Premium/Ultimate）：固定周长（如两周）的执行窗口，与 Milestone **并行使用** —— Milestone 管"交付什么版本"，Iteration 管"这两周做哪些"。
- **Roadmap**（群组）：按季度展示 Epic 的 `start_date`/`end_date`；显示空白通常是**Epic 没设起止日期**，不是渲染问题。

## 8.4 层级关系（推荐的组织方式）

```text
群组 Epic: 支付体系重构                         ← Ultimate 下可有子 Epic
 ├── 项目A Issue: 新增部分退款接口   ← Milestone: 26Q4 / Iteration: 26Q4W3
 │     └── MR !128（合并后 Closes #123 自动关闭，进度回写 Epic）
 ├── 项目A Task:  退款幂等单测       ← 挂在上面 Issue 之下（/add_child）
 ├── 项目B Issue: 前端退款页
 └── 项目C Issue: 对账任务改造
```

---

# 9. Issue ↔ MR/分支 的双向绑定

这是 GitLab 相对独立工单系统最大的优势 —— **需求与代码互相可见**。

## 9.1 关联语法

```text
分支名：      issue/123-add-refund        创建 MR 时自动关联 Issue
MR 描述：     Closes #123                 合并即关闭
              Closes #123, #124           多项
              Relates #125                只关联不关闭
Issue 评论：  /create_merge_request       直接从 Issue 起 MR
提交信息：    也支持 Quick Actions（如对 Issue 打标签/关闭）
```

## 9.2 关联类型语义（不要只用一种）

| 关系 | 作用 | 典型用法 |
|------|------|----------|
| `Closes #N` / `/close` | MR 合并后关闭 Issue | 一个 Issue 对应一个交付 MR |
| **relates to**（`/relate #123`） | 双向可见，不影响状态 | 跨项目协同、需求与测试用例 |
| **blocks / blocked by**（`/blocks`、`/blocked_by`） | 依赖方向 + 看板阻塞图标 | 上下游接口依赖 |
| **duplicate**（`/duplicate #123`） | 关闭并标记重复 | 报障去重（保留原始 Issue） |
| **parent/child**（`/set_parent`、`/add_child`） | 层级结构，进度汇总 | Epic ↔ Issue ↔ Task |

## 9.3 为什么"每个 MR 必须有需求锚点"

- 产品/测试在 Issue 页右侧 **Merge Requests 区块**实时看到 open/merged/closed，**不用问研发**。
- 反向可审计：任何一个 MR 都能追到需求（第 5 篇的 MR 规范 + 模板里的 `Closes %{issue}`）。
- 度量可算：Issue → MR → Deployment 的时间差是 DORA 的前提（第 14 篇 Value Stream）。

> **纪律**：分支保护 + `Closes #N` 写进 MR 模板，并在 **群组分叉设置里禁止无关联 Issue 的 MR**（用 CI 检查 MR 描述里是否含 `#` 引用实现 —— 简单、无订阅要求）。

---

# 10. GLQL：把"进度表"写成可执行查询

**GitLab Query Language（GLQL）**：一套跨对象的查询语言，**17.4 引入、17.10 beta、18.3 GA**，Free/Premium/Ultimate 均可用。数据源覆盖 **work items、merge requests、pipelines、jobs、projects** 等。

## 10.1 嵌入视图（Embedded views）

在 Markdown 代码块里写查询，渲染成表格/列表：

````text
```glql
display: table
title: 本周需要我处理的事项
description: 指派给我且未关闭
fields: title, state, health, epic, milestone, weight, updated
limit: 20
query: type = Issue AND group = "pay" AND assignee = currentUser() AND state = opened
```
````

- **可显示位置**：群组/项目 **Wiki**、Epic/Issue/MR/工作项的**描述与评论**、**仓库 Markdown 文件**（18.3 起）。
- 语法是 **YAML 超集**：`query` + 展示参数（`display`、`fields`、`limit`、`sort`、`pagination`、`collapsed`、`title`、`description`）。
- **旧的 YAML front matter 写法在 17.7 起废弃** —— 迁移成上面这种单块语法。
- 表达式形如 `<field> <operator> <value> and ...`，支持 `currentUser()` 这类字段函数、自定义别名；`not in` 等运算符逐个版本补齐。

**落地建议**：把 `README.md`/Wiki 里的"人工维护进度表"换成 GLQL 嵌入视图 —— **过期表是协作噪音的最大来源**。注意：查询以**查看者权限**执行，无权限的对象不会出现在他的视图里（同一页面不同人看到不同行数，属正常）。

## 10.2 Analytics 模式

GLQL 还有 analytics 模式（聚合/分组/粒度/自定义别名/排序），可在文档里放趋势图而不是明细表：适合"每周迭代吞吐"这类需要按天/周聚合的展示。字段与粒度参数以 GLQL 文档为准。

---

# 11. Wiki 与文档

## 11.1 Wiki 就是一个 Git 仓库

- 每个项目/群组都可开 Wiki，底层是独立仓库：**`https://<host>/<group>/<project>.wiki.git`**（SSH 同形）。
- 入口：**左栏 Plan → Wiki**；批量编辑请**克隆 wiki 仓库本地改再 push**（长文档推荐）。
- **只支持特定扩展名**，用错后缀的文件 push 后**不会显示**：Markdown `.mdown .mkd .mkdn .md .markdown`；AsciiDoc `.adoc .ad .asciidoc`。
- **默认分支**继承实例/群组设置（未自定义则为 `main`），改 wiki 默认分支就是改该仓库的默认分支。
- Wiki 的 commit **不计入仓库分析（repository analytics）**，创建/删除/更新事件会出现在用户活动流与项目/群组 Activity 页。

## 11.2 标题与路径的转换规则（改名前必读）

- 默认**文件名即标题**；**空格存成连字符、显示时连字符还原为空格**；**斜杠 `/` 是路径分隔符，不能出现在标题里**（含 `/` 的文件名会被拆成层级）。
- 16.7 引入、**17.2 默认启用、17.3 移除开关**：**front matter 让"标题与路径分离"**，因此老页面重命名标题不再被迫改 URL。
- **长度限制**：文件与目录名有长度上限（见 Wiki 文档 "Length restrictions"），超长中文标题生成的 slug 会被截断 —— 深层目录 + 长标题是"页面 404"的常见来源。

## 11.3 模板、侧边栏、目录、订阅、历史

| 能力 | 做法 |
|------|------|
| **页面模板** | 存放在 **wiki 仓库的 `templates/` 目录**；UI：**Wiki actions → Templates → New template**（需 Developer+）。模板**路径由标题生成且不可编辑**，应用模板不会注入标题 |
| **自定义侧边栏** | 创建名为 **`_sidebar`** 的页面，**完全替换默认侧栏**（Wiki actions → Add custom sidebar） |
| **首页** | 创建 **`home`** 页面作为 Wiki 首页 |
| **目录** | 有标题的页面**自动在侧栏生成 TOC**（17.2 起）；页内 TOC 用 `[[_TOC_]]` |
| **图表** | 支持 **Mermaid**、以及 **Draw.io** 图（页面内嵌编辑） |
| **富文本编辑** | GitLab Flavored Markdown 的沉浸式编辑器（表格结构、代码块高亮等） |
| **订阅** | 页面右上角铃铛订阅；**有人新增评论或回复评论**时通知（不是"页面被改"通知） |
| **历史/回滚** | 每页 History 可对比任意两版并恢复到旧版；移动/重命名后的历史可追溯 |

## 11.4 外部 Wiki 与禁用

- **External wiki**：**Settings → Integrations → External wiki** 填 URL，左栏出现外链入口。**注意：启用后不会自动隐藏内部 Wiki**，要隐藏需另行关闭 Wiki 项目特性。
- **关闭内部 Wiki**：**Settings → General → Visibility, project features, permissions → Wiki** 关掉的后果是"侧栏入口消失、不能增删改、**也看不到任何页面**"；**已有页面内容保留**，重新打开即恢复。群组 Wiki 也可在群组设置里关闭。
- **自建排错**：Apache 反向代理下页面 slug 渲染异常 → 给 `ProxyPass` 加 `nocanon`；wiki 仓库损坏可用 Rails console 重建（Free/Premium/Ultimate，Self-Managed/Dedicated）。

## 11.5 Wiki vs 仓库内 `docs/`

| 位置 | 适合 | 理由 |
|------|------|------|
| `docs/` + MR 评审 | 架构决策（ADR）、接口契约、Runbook | **变更走评审、与代码同分支同版本**，能回答"上线那天文档写的是什么" |
| Wiki | 操作手册、新人入职、FAQ、会议结论 | 非开发角色也能编辑，不需要 commit 权限与分支概念 |

> **行业实践**：关键文档（设计文档、Runbook）放仓库 `docs/`，享受 MR 评审与版本对齐；杂项知识放 Wiki。**别把 Wiki 当 Confluence 用** —— 它没有细粒度页面权限与模板审批流（Wiki 权限就是仓库的 Developer/Maintainer 级别）。

---

# 12. 其他高频协作能力

- **Snippet（代码片段）**：内部 Gist，分享小段代码/复现脚本；可设可见性与过期，适合"贴错误堆栈但别刷屏"。
- **时间跟踪**：`/estimate 2h` + `/spend 45m`，Issue 页与里程碑侧栏汇总 time spent —— **做容量评估前先要求团队填估算**。
- **健康状态**：`🟢🟡🔴` 卡片可见字段，配合周会用（比"进度百分比"诚实）。
- **Issue 权重（weight）**：数值化相对规模，看板列支持按权重合计与权重 WIP 限制。
- **导入导出**：GitHub/Jira 迁移有官方工具（`gitlab-org` 下的导入器）；Jira CSV 走导入脚本。**迁移前先决定"旧工单是否带历史评论"**，这是决定工作量的关键项。
- **待办 Todo / RSS / 保存视图**：个人入口用 Todo list，团队入口用 **Saved views**（工作项列表可存视图），避免每个人都手敲同一串过滤条件。
- **审批与提醒**：`/checkin_reminder`（给工作项设检查提醒）、`/subscribe`（关注通知）。

---

# 13. 落地节奏与自动化（把上面串成制度）

```yaml
# 用 CI 定时做"需求卫生检查"，比人肉盯盘可靠（schedule 触发见第 6 篇）
weekly-hygiene:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  script:
    # 1) 没有 assignee / milestone 的开放 Issue（超过 7 天）
    - |
      glab api "projects/:id/issues?state=opened&per_page=100" \
      | jq -r '.[] | select(.assignees==[] or .milestone==null)
              | select(.created_at < (now - 7*86400 | strftime("%Y-%m-%dT%H:%M:%SZ"))) | "\(.iid)\t\(.title)"'
    # 2) 挂了 Done 状态但仍是 opened 的异常项（改类别造成的历史脏数据）
    - glab api "projects/:id/issues?state=opened&label=status%3A%3Adone" | jq 'length'
```

**推荐节奏**：

| 频率 | 动作 | 用到的能力 |
|------|------|-----------|
| 每天 | 值班分诊：新 Issue 打类型/优先级/负责人，重复项 `/duplicate` | 模板 + scoped label + Quick Action |
| 每个迭代开始 | Iteration + Milestone 双绑；WIP 限制确认 | 群组迭代 + 看板 WIP |
| 每周 | 看板走查：阻塞图标逐个讨论；健康状态刷新 | blocks/blocked_by + health status |
| 每版本 | 里程碑燃尽图收口，未关闭项显式顺延 | Milestone burndown |
| 每月 | 标签/状态清理、`templates/` 与 `_sidebar` 校订 | Labels + Wiki 治理 |

---

# 14. 常见坑与排错

| 现象 | 原因 | 处理 |
|------|------|------|
| 卡片拖到 Done 列，Issue 自己关了 | 该状态属 **Done/Canceled 类别** | 正常语义；不想关闭就把它做成非 closed 类别的状态（但要重新审视流程） |
| Work items 的 Board 视图里 closed 状态列是空的 | 默认 `State = Open` 过滤 | 移除 State 过滤器 |
| 同一 Issue 出现在多个列 | 它有多个标签（且列是标签列） | 改 scoped label 或 status list |
| `/cc @user` 不生效 | **18.3 已移除** | 用 `/subscribe` 或直接 `@user` 提及 |
| 快捷操作写了没反应 | 补全下拉里就没有它 → 订阅/角色不足，或前置条件不满足 | 按 6.3 三步排查 |
| 看板搜索栏里找不到 milestone/assignee 过滤项 | **看板已被该维度限定作用域** | 编辑看板，移除该作用域 |
| 群组看板按人过滤报 `There was a problem fetching users` | 不是该群组成员 | 把用户加到**顶层群组** |
| 看板加载超时 | 排序元数据失衡 | Rails console 跑 `Issues::RelativePositionRebalancingService`（7.5） |
| 改了 Default.md 但新建 Issue 没变化 | 项目设置里配了默认描述，**优先级更高** | 清空设置项或统一只用一个入口 |
| 群组模板不生效 | 模板项目不是该群组的**直接子项目**，或你不是 Owner | 移动项目/换 Owner 操作 |
| 老团队找不到 Issue 菜单 | 已合并进 **Plan → Work items**，URL 自动重定向 | 更新书签与脚本 |
| Wiki 文件 push 了但页面不存在 | 扩展名不在支持列表 | 改成 `.md`/`.adoc` |
| Wiki 页面标题与 URL 不一致 | 空格/连字符转换、或用了 front matter 标题 | 用 11.2 规则理解；重命名走 Wiki 的 move 而不是删重建 |
| 项目关掉 Wiki 后文档"不见了" | 特性开关会隐藏所有页面视图 | 重新打开 Wiki 特性即恢复（内容未删） |

---

# 15. 小结

| 要点 | 一句话 |
|------|--------|
| 中心对象 | **Work Item**：Issue/Task/Epic/OKR/Incident/Service Desk/Test case 统一入口（Plan → Work items，URL 归一到 `/work_items/:iid`） |
| 类型 | 可在顶层群组自定义类型（**≤40 个/顶层群组**，保留名不可用，Enabled/Disabled/Archived/Locked 四态）；转换用 `/type` 与 `/promote_to` |
| 状态 | **Status 会驱动 open/closed**（Done/Canceled 类别），只在顶层群组配一次；`/status` 仅 Issue 与 Task；改状态**类别**有连带关闭风险 |
| 与 scoped label 分工 | Status = 流程阶段且影响状态；Scoped Label = 互斥属性且不改状态，还能用于 MR |
| 规范化 | Markdown 模板 + Quick Actions 把规范固化；默认模板 `Default.md` **优先级低于项目设置**；跨仓库靠 **instance/group template repository**（群组模板项目必须是直接子项目） |
| 入口 | 表单化入口用 **URL 预填参数**；对外报障用 **Service Desk**；对内邮件建单需 incoming email |
| 看板 | 列表有 label/assignee/milestone/iteration/**status**（18.4 GA）五种可选类型；**拖动 = 换标签**；多标签会多列同现；每列先加载 20 条；WIP/泳道/权重合计为企业版；Free 群组只有 1 块板 |
| 时间维度 | Milestone（含燃尽/燃上，Planner 起可建）管版本，Iteration 管周期，Epic（Premium+，嵌套需 Ultimate）管主题 |
| 杀手锏 | **Issue ↔ MR 双向自动关联**：`Closes #N` / `blocks` / `duplicate` / parent-child 各有语义，别只用一个 |
| 动态视图 | **GLQL（18.3 GA）** 把进度表写成 ` ```glql ``` ` 嵌入 README/Wiki/描述，按查看者权限渲染 |
| 文档 | 契约类进 `docs/` 走 MR，手册类进 Wiki（Wiki 是 `.wiki.git` 仓库，`_sidebar`/`home`/`templates/`/`[[_TOC_]]` 是它的四块基础设施） |
| 选型 | 轻量到中度流程用原生；重审批工作流仍交给 Jira/ServiceNow 集成 |

**自检清单**：

- [ ] 我能解释"为什么把工作项拖到 Done 列会关闭它"，以及这与加一个 `status::done` 标签的区别？
- [ ] 我能说出 Free 与 Premium/Ultimate 在看板上的三条具体差异（多群组板、可配置作用域、WIP/状态列/泳道）？
- [ ] 我知道 `Default.md` 与项目设置里默认描述的优先级关系，以及群组模板项目必须满足的两个条件？
- [ ] 一个 Issue 同时出现在三列时，我能在 1 分钟内定位原因并给出两种修法？
- [ ] 我能用 GLQL 写一个"本组未关闭且无负责人"的嵌入表格，并说出它能出现在哪些位置？
- [ ] 我能说清 Wiki 与 `docs/` 的分工，以及关闭 Wiki 特性后内容去哪了？

> 协作侧齐了。接下来进入交付主线的另一半：[第 12 篇 安全扫描与权限治理](12_security_and_permissions.md)。
