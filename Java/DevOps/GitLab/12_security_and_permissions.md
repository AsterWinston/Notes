# 1. 权限体系的心智模型

> **前置**：[第 4 篇 仓库与分支](4_repositories_and_branching.md)、[第 5 篇 MR 与评审](5_merge_requests_and_review.md)
> **本篇目标**：把"谁能做什么、用什么身份做、做错了怎么发现"讲清楚：角色与继承、令牌全家桶、访问边界（IP/域名/SSO/LDAP）、分支与推送门禁、审计与泄露响应
> **对应实际工作**：权限申请/回收、403 排查、安全审计、等保与合规检查

## 1.1 核心公式

**用户在某个作用域（Instance / Group / Project）上被授予一个角色 → 角色决定能力 → 高作用域的授权自动向下继承。**

GitLab **故意不做细到按钮的 ACL**（与 Jira 的思路相反），换来的是"可推理、可审计"。于是出现一条铁律：

> **个别点状需求不要指望改角色，要用"角色 + 保护分支 + 审批规则 + 推送规则"组合出来。**
> 例：想"Developer 能提 MR 但不能合主干" → 不是降低角色，而是把 `main` 的 **Allowed to merge** 设成 Maintainers。

## 1.2 默认角色表（现行）

| 角色 | 能力摘要（官方口径） | 备注 |
|------|---------------------|------|
| **Minimal Access** | 只看到有限的群组信息，**不含任何项目**；Premium/Ultimate | **不占付费席位**（18.9 起，前提是他在全实例没有其它角色）。**它不给权限表里的任何权限** —— 连 wiki/Issue/仓库都看不到；要让他真看到某个项目，Owner 必须**再以 Guest 及以上角色加进那个项目/子群组**。18.11 起对它也强制 2FA 要求 |
| **Guest** | 查看/评论 Issue 与 Epic；**不能推代码、不能访问仓库** | 仅对 **private/internal** 项目有意义（public 项目本来人人可见） |
| **Planner** | 创建与管理 Issue/Epic/Milestone/Iteration，可查看并协作代码变更 | 规划岗专用，**不累积** |
| **Reporter** | 看代码、建 Issue、出报表；**不能推代码、不能管理保护分支** | 拿到 clone 权限的最低常用角色 |
| **Security Manager** | 查看与管理漏洞、合规配置、审计事件；**无推码权限** | 安全岗专用，**不累积** |
| **Developer** | 推**非保护**分支、建 MR、跑流水线；**不能管理项目设置** | 开发主力 |
| **Maintainer** | 管分支/MR/CI 设置/项目成员；**不能删项目** | 项目负责人 |
| **Owner** | 完全控制，含删除与可见性 | 群组 Owner = 所有子群组与项目的 Owner |

> 另一条默认值常被忽略：**所有用户默认都能创建顶层群组**（管理员可在实例设置里改掉）。等保/合规检查里"普通员工能否自行建群组"通常是必答题。

**三个容易踩的认知差**：

1. **只有 Guest → Reporter → Developer → Maintainer → Owner 是累积的**；**Planner 与 Security Manager 是"专用角色"，不保证包含前序角色的权限**（官方原话：要确认某个权限，去查对应的权限表）。给"项目经理"加 Planner 而不是 Reporter，是正确的收紧做法。
2. **`Guest` 看不到代码，但 `Internal` 项目对所有登录用户可见** —— "我们把项目设成 Internal 所以没人能看代码"是完全相反的理解。
3. **Auditor（审计员）不是"角色"，是用户类型**：在 **Admin → Overview → Users → New user** 里把 **User type** 选成 **Auditor**（也可用 SAML 的 auditor groups 或 Users API 设置）。它给你**跨全实例的只读可见性 + 查看全部群组/项目审计事件的权限**，不给你改代码的能力。所以：合规/审计部门给 Auditor，**不要为了"只读"把人加成 Reporter**（Reporter 能建 MR、下载制品、开 Issue board），更不要给 Maintainer。

## 1.3 自定义角色（Custom roles）：能力与限制

自定义角色 = **一个"基座默认角色" + 若干附加权限点（abilities）**，用来解决"这个角色包太大 / 太小"。要点全部以官方文档为准：

| 事项 | 事实 |
|------|------|
| 基座 | 必须选一个内置角色作为 base（**不能用 Auditor 当基座**）；自定义权限可以放开"通常只有 Maintainer/Owner 才能做"的动作 —— 例如给了"管理 CI/CD 变量"，就**连别人建的变量也能管** |
| 数量上限 | **每个命名空间少于 10 个自定义角色** |
| 生效范围 | **GitLab.com：只在创建它的顶层群组之下**（群组 Settings → **Roles and permissions**）；**Self-Managed/Dedicated：全实例**（**Admin → Settings → Roles and permissions**，且还可建 **custom admin role**，仅自建） |
| 程序化 | 走 GraphQL（`memberRoleCreate` 等 mutation），不是 REST |
| 命名 | 权限点是**具名的**，例如管理保护分支需要 `admin_protected_branch`；没这个权限点的角色改不了保护分支 |
| 边界 | 自定义角色改的是"**角色能力**"，**不改变对象级门禁**：保护分支、审批规则、推送规则照样叠加生效；**也不解决"这个人只能碰这几个目录"**（那是 CODEOWNERS） |

> **实践建议**：先用足内置角色 + 门禁组合，再考虑自定义角色。自定义角色一旦多起来，"权限推理"就不再是可口头交流的了，必须配一份"角色 → 权限点 → 岗位"表进 Wiki，并在成员变更走 MR 审批（第 11 篇的 Code Owner 思路同样适用于 `.gitlab/` 下的权限定义文件）。

## 1.4 权限继承与"看起来不对"的四种情形

```text
Instance ──(默认分支保护级别 / 全局推送规则 / 允许的功能)──► 所有项目
Top-level Group ──(成员、IP/域名限制、禁止外部 fork、群组推送规则、群组变量)──► 子群组与项目（部分只作用于"新建"）
Project ──(直接成员、保护分支、审批规则)──► 该项目
```

| 现象 | 真实原因 | 定位方法 |
|------|----------|----------|
| "他是 Developer 却合不了 main" | 保护分支把 merge 权限收到 Maintainer | `glab api "projects/$P/protected_branches"` |
| "加到群组了，项目里还是看不到" | 加的是**别的**群组/项目被 share；或项目可见性 + 继承被显式覆盖 | `members/all` 看来源 |
| "改了群组推送规则，老项目没生效" | **推送规则只对新项目生效**（3.1 节） | 逐项目读回 `GET /projects/:id/push_rule` |
| "离职的人还有权限" | 人可能只是被 deactivate（休眠机制，重新登录即恢复），或其创建的 **project/group bot 令牌仍在有效期内**（bot 不随人停用） | Block 用户 + 4.4 令牌盘点 |

```bash
# 看"实际生效"的成员与来源（inherited 标记来自上层群组）
glab api "projects/payments%2Forder-service/members/all?per_page=100" \
  | jq -r '.[] | "\(.username)\t\(.access_level)\t\(.inheritance_type // "direct")"'
# 判断某人对某资源的访问级别（比人肉推理可靠）
glab api "projects/payments%2Forder-service/members/$USER_ID" | jq '{access_level, state}'
```

---

# 2. 令牌全景：从字符串前缀认出身

## 2.1 前缀表（官方口径）

| 令牌 | 前缀 | 用途 |
|------|------|------|
| Personal access token | `glpat-` | 人/脚本以"某用户身份"访问 |
| **Project / Group access token** | **`glpat-`（与 PAT 相同）** | 机器人身份访问单个项目/群组 |
| Impersonation token（管理员模拟） | `glpat-` | 排障，等价于以该用户身份 |
| OAuth Application Secret | `gloas-` | 应用密钥（不是访问令牌本身） |
| Deploy token | `gldt-` | 仓库/Registry 拉推，**无 API 权限** |
| CI/CD **Job** token | `glcbt-` | 单个 Job 内，短期 |
| Trigger token（`trigger:` / pipelines API） | `glptt-` | 触发流水线 |
| Runner authentication token | `glrt-`（经 registration 流程创建的为 `glrtr-`） | Runner 领 Job（第 8 篇） |
| GitLab agent for Kubernetes token | `glagent-` | GitOps / Agent 访问 |
| SCIM token | `glsoat-` | 用户目录同步 |
| Feed token | `glft-` | RSS/Atom 订阅 |
| Incoming email token | `glimt-` | 邮件建 Issue |
| Workspace token | `glwt-`（18.2+） | 工作区 |
| Feature Flags client token | `glffct-` | 客户端读 flag |
| 会话 Cookie | `_gitlab_session=` | 浏览器登录态 |

> **关键陷阱（官方明确写了）**：**PAT、Project token、Group token、Impersonation token 全都是 `glpat-` 前缀**，"两个长得一模一样的令牌行为可能完全不同"。**不能靠肉眼区分令牌类型** —— 治理手段只能是"创建时就写清名字与描述"（2.3 节）+ "用 API 反查"（第 4 节）。

## 2.2 能力矩阵（默认 scope，决定"能干什么"）

| 令牌 | API | Registry | Repository |
|------|-----|----------|-----------|
| PAT | ✅ | ✅ | ✅ |
| OAuth 2.0 token | ✅ | ❌ | ✅ |
| Impersonation token | ✅ | ✅ | ✅ |
| Project access token | ✅（仅该项目） | ✅（仅该项目） | ✅（仅该项目） |
| Group access token | ✅（仅该群组） | ✅（仅该群组） | ✅（仅该群组） |
| **Deploy token** | **❌** | ✅ | ✅ |
| Deploy key | ❌ | ❌ | ✅（单仓库） |
| Runner registration / authentication token | ❌ | ❌ | 有限（只用于注册/认证 Runner） |
| **Job token** | **有限**（仅白名单端点） | **❌（作为 API 令牌）** | ✅ |

**两条必须解释清楚的地方**：

- **"Job token 对 Registry 是 ❌"与"CI 里用 `CI_JOB_TOKEN` 登录 Registry"不矛盾**：后者是容器仓库认证流程把 Job Token 兑换成一次性的 registry 凭据（`CI_REGISTRY_USER=gitlab-ci-token` + `CI_REGISTRY_PASSWORD=$CI_JOB_TOKEN`），不是拿它去调 `/api/v4` 做包管理。
- **Job token 不能当"万能内部令牌"**：跨项目拉仓库/包要在**被访问项目**的 **Settings → CI/CD → Job token permissions** 里授权（第 10 篇 10.3），否则报 `401/403`。

## 2.3 令牌选择决策表（Java 团队常见场景）

| 场景 | 选什么 | 为什么 |
|------|--------|--------|
| 流水线内访问 GitLab API（建 MR、查包、传报告） | **Job token**（scope 最小） | Job 结束即失效；官方建议顺序就是 job → project → group |
| 流水线内需要 Job token 覆盖不到的权限（例：跨群组建 Issue） | **Project access token**（存为 Protected+Masked 变量） | 权限绑定项目、可设过期、可独立撤销 |
| 外部系统只读拉镜像/依赖（生产服务器、堡垒机） | **Deploy token**（`read_registry`） | 没有 API 权限 = 泄露时爆炸半径小 |
| 需要"只读某个仓库"的机器（CI 之外的部署脚本） | **Deploy key**（只读） | 单仓库、无用户身份 |
| 云厂商资源访问（OSS/S3/镜像仓库跨云） | **OIDC id_token 联邦**（`id_tokens`，第 7 篇） | 不在 GitLab 存任何长期 AK/SK |
| 本地 `glab` CLI、个人调试 | PAT，**必须设过期 + 最小 scope** | 人离职即断，别把它塞进 CI |
| 集成应用（第三方插件） | OAuth 应用 | 有授权流、可刷新 |

> **纪律**：**CI/CD 变量里不要放 PAT**（官方原文建议：因 scope 过宽应尽量避免；确需时按 job token → project token → group token 的顺序升级）。

## 2.4 命名规范：审计能不能做下去，取决于这一条

官方给出的命名要回答三件事：**动作 + 资源 + 环境/归属者**。

```text
ci-deploy-gitlab-production        # CI 部署到生产
api-read-reporting-dashboard       # 报表系统只读 API
automation-sync-vulnmapper-staging # staging 的漏洞同步脚本
```

**禁用**：`test`、`mytoken`、`token1`、`GITLAB_API_TOKEN`、`API_TOKEN`、`default` —— 审计时无法判断用途，出事后不知道该不该撤销。

创建时同时写 **description**（包含调用方 URL/系统名），并**为一次性任务设置"任务完成即到期"的过期时间**（例：导入任务设几小时）。

---

# 3. 访问边界：群组级"谁能进来"

## 3.1 按 IP 限制群组（Premium/Ultimate，三种 offering）

配置入口：**顶层群组** Settings → General → **Permissions and group features** → **Restrict access by IP address**，逐条填 CIDR：

```text
192.168.1.0/24
10.0.0.0/8
2001:db8::/32
203.0.113.5/32
```

**填法上的三条硬规定**（官方原文）：

- **必须逐条手工添加**，**不支持用逗号或空格一次性粘贴多个**（批量能力还在 issue 里讨论）。
- 条目数量**没有上限**；但**无法按"SSH / HTTP"分别设** —— 一份名单同时管两种授权。
- Self-Managed 还可以叠加**实例级"全局允许 IP 段"**（Admin → Settings → Visibility and access controls → Globally-allowed IP address ranges），两者组合使用。

**作用范围**：GitLab UI（含子群组、项目、Issue）+ **API**；**不影响 GitLab Pages**。

> **`X-Forwarded-For` 必须"覆盖"而不是"追加"**，否则客户端可以自己塞一个内网 IP 完成 IP 欺骗。官方给的两种部署写法：
>
> ```nginx
> # 前面没有代理/LB：以真实来源直接覆盖
> proxy_set_header X-Forwarded-For $remote_addr;
> # 前面有代理/LB：由代理保留完整链路（GitLab 收到从原始客户端开始的整条 IP 链）
> proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
> ```

**"这不是防火墙"——官方列出的绕过面（每一条都真实存在，写安全评估时照抄）**：

| 即使来源 IP 不在白名单，仍然可用 | 说明 |
|----------------------------------|------|
| 管理员访问群组设置与群组内项目（**含 clone 代码**） | 所以"IP 白名单能防管理员"是不成立的 |
| **群组 Owner 访问子群组**（但不含群组/子群组里的项目） | Owner 身份优先于 IP 限制 |
| 被**共享**的资源 | 共享关系不受本限制 |
| 群组与项目的**名称、层级结构** | 组织结构对所有用户可见 |
| **邮件回复**建/改 Issue 与 MR 评论；用 **incoming email token** 邮件事务 | **邮件通道不受 IP 限制** —— 这是最小的一条攻击面 |
| 个人 dashboard 上的 push/merge/issue/评论事件 | 间接信息泄露 |
| **公开项目**本身受限制，但**缓存过的项目文件有时仍可访问** | 别把 public 项目当"数据不外泄" |
| fork 出去的项目、以及"间接引用"的操作（例如 MR 描述里用 issue 引用关闭受限群组内的 Issue） | 官方明确"might not apply" |

**Git 与 CI 侧的边界（最容易出生产故障）**：

- `.com` 与 Dedicated 上 **SSH Git 操作一定受限**；Self-Managed 要用 **`gitlab-sshd` 并启用 PROXY protocol**，否则拿不到真实客户端 IP。
- **CI Job 的 clone 受限**，但 **GitLab.com hosted runner 例外**（其网段在全局允许段内）。
- **Git LFS 下载与容器仓库请求一定受限（包括 hosted runner）** → 典型症状：job 能 clone，但 `docker pull/push` 或 LFS 拉取失败。官方给的两条处置：**把 hosted runner 的 IP 段加进白名单**（hosted runner 可能跨多个 GCP region，只加一个 region 会**间歇性失败**，region 清单见 GitLab.com 文档的 IP range 页），或**改用有固定 IP 的 group/project Runner**；要彻底禁掉 hosted runner，就**关闭该群组的 instance runners 且不允许子群组/项目覆盖**。
- **Runner 注册不受 IP 限制**，但注册成功后领到的 job 会在执行时失败，报错原文：

  ```text
  This job could not be executed because group IP address restrictions are enabled, and the runner's IP address is not in the allowed range.
  ```

- 受限主体覆盖：**Project/Group access token、service account、deploy key、deploy token**。

**定位手法（404 别乱猜）**：

```bash
# ① 先粗筛：确认请求是不是被"群组 IP 限制"拒了
sudo grep -h 'Attempting to access IP restricted group' /var/log/gitlab/gitlab-rails/auth.log* | tail

# ② 再结构化：看 GitLab 当时认为的客户端 IP 与判定结果
#    （日志里的键名带点，必须用 ."remote.ip" 这种写法取值）
sudo grep -h 'IP restricted' /var/log/gitlab/gitlab-rails/auth.log* \
  | jq -c 'select(."json.allowed" == false) | {time, client: ."remote.ip"}' | tail

# ③ 双栈机器自查：这个客户端有没有 IPv6 出口（有的话必须一起加白）
ip -6 addr show scope global | grep inet6
```

> **IPv6 是最隐蔽的坑**：双栈系统常**优先走 IPv6**，而终端里看不出区别。加了 IP 白名单后"能开网页但 clone 不了"，先确认是否走了 v6，再把对应 IPv6 段补进名单。

## 3.2 按邮箱域名限制成员（Premium/Ultimate）

同页面 **Restrict membership by email domain**（设在**顶层命名空间**，子群组继承）：加成员时，该用户的 **primary email 域名**必须命中名单。

| 事实 | 影响 |
|------|------|
| **不能把公共邮箱域设成"受限"**：`gmail.com`、`outlook.com`、`hotmail.*`、`icloud.com`、`me.com`、`live.com`、`mail.com`、`msn.com`、`yahoo.com`、`yandex.com`、`proton.me`、`protonmail.com`、`tutanota.com`、`zohomail.com`、`aol.com` | 想靠它"禁止 gmail 用户"是无效的（这些域名列在允许清单里才起作用） |
| **共享群组时，源与目标命名空间都必须允许成员邮箱域名** | 跨部门共享前先核对两边名单 |
| **把某个域从名单里删掉，不会移除已加入的该域用户** | 收紧后必须**人工盘点并移除**不合规成员 |
| 群组被共享给别的群组时，**目标群组可以往自己的名单里加源群组没有的域** | 官方原话：**该特性不保证现有成员始终符合名单** —— 别把它当合规控制，只当准入控制 |

> 与 SSO 的域名校验是两件事：这里管"**能不能成为成员**"，SAML/OIDC 的域名允许列表管"**能不能登录到这个命名空间**"。

## 3.3 SSO / SCIM / LDAP：把身份交给企业目录

| 能力 | 文档位置 | 已核实的关键点 |
|------|----------|----------------|
| **SAML SSO**（群组级，含 **SSO gate**：非 SAML 身份禁止访问群组） | `user/group/saml_sso/_index.md` | 群组级 SSO 属 Premium/Ultimate；启用前应完成域名校验，否则 gate 形同虚设 |
| **Group Sync（SAML）** | `saml_sso/group_sync.md`（**Premium/Ultimate**，三种 offering） | IdP 组 ↔ GitLab 群组**多对多**映射；**不会创建群组**（先建好群组再映射）；**GitLab.com 默认已配好，Self-Managed 必须自己配**（Azure 侧要 `GroupMember.Read.All` + `User.Read.All` 并 **Grant admin consent**，client secret **过期后集成直接失效**）；角色冲突时按 **Role prioritization** 取更高者；可配 **Automatic member removal**；**Global SAML group memberships lock** 会锁住手工加人 |
| **SCIM**（用户/组开通） | `saml_sso/scim_setup.md` | 令牌前缀 **`glsoat-`**；与 IdP provisioning 对齐后，**IdP 停用即 GitLab 停用** —— 这是离职自动化的正解 |
| **OAuth2 / OpenID Connect 登录** | `administration/auth/oidc.md`、`jwt.md`、`smartcard.md`、`crowd.md`、`cognito.md` | 自建实例统一身份的几种官方方式；参数（`auto_link`、`bypass_two_factor` 等）按版本核对 |
| **LDAP + Group Links** | `user/group/access_and_permissions.md`（**Premium/Ultimate，仅 Self-Managed**）、`administration/auth/ldap/` | 见下 |

**LDAP Group Links 的五条实操结论**（都来自官方文档，不是经验谈）：

1. 用 **CN** 或 **user filter** 建映射，入口是群组的 **Settings → LDAP Synchronization**（删除链接在 **Settings → Active synchronization**）。
2. **配好之后"用户可能要 1 小时以上才同步进来"** —— 改完立刻验证失败就回滚是最常见的错误操作。
3. 同一 GitLab 群组挂多个 LDAP 组时，**取角色更高的那个**；两个自定义角色 **base 相同**时无法比较大小，**取最早创建的那条 group link**（17.2 起支持给同步用户分配自定义角色）。
4. **加上 LDAP 同步的瞬间，组里"不属于该 LDAP 组"的既有成员会被移除**；反过来**删掉 group link 后，已有的成员关系与角色会保留**（此时才可以直接编辑这些成员）。
5. 即使群组开了 LDAP 同步，**群组 Owner 仍可用 API 增删"服务账号"**（`member_roles` / `group_members`），但**不能增删普通用户** —— 所以"给机器人账号留口子"是合规可行的做法。
6. **管理员可手工覆盖 LDAP 授予的权限**：Manage → Members → 铅笔 → **Edit permissions**。注意判定规则 —— LDAP 给的角色**高于**上层群组继承时显示为 **direct member**，**等于或低于**时显示为 **inherited**（要筛 direct 才能改）。

> **顺带一个高频 403**：群组 Owner 改不了某个成员的权限，通常不是 bug —— 他改的是 **inherited membership**。**Group Owner 只能更新 direct membership**，先在成员页筛选"直接成员"再操作。

**排障纪律**：任何"权限没生效"的问题，先分清是**授权模型**（角色/保护分支/门禁）还是**同步时序**（LDAP/SCIM/Group Sync 周期、IP 限制的 IPv6、受保护 tag 的 JWT 5/15 分钟延迟，见第 10 篇）。

## 3.4 群组权限收紧开关（Settings → General → Permissions and group features）

| 开关 | 效果 | 层级与坑 |
|------|------|----------|
| **Enabled Git access protocols** | 群组仓库只允许 SSH 或只允许 HTTPS | 群组；**实例级被管理员锁定时这里直接禁用** |
| **Prevent forking outside top-level group** | 组内项目不能被 fork 到顶层群组之外 | **只能设在顶层群组**，子群组继承且**不可关闭** |
| **Prevent members from being added to projects** | 锁定"逐项目加成员"，强制走群组授权 | 群组 Owner 仍可加**群组级**成员；适合审计期冻结 |
| **Prevent users from requesting access to a group** | 关掉"自行申请访问"入口 | 收紧流程时用，配合审批工作流 |
| **Restrict access by IP address / by email domain** | 见 3.1 / 3.2 | **顶层群组** |
| **Default role to use pipeline variables**（17.10 起） | 设定该群组**新建项目**默认"最低可运行带流水线变量流水线的角色"（含 **No one allowed**） | **只能设在顶层群组**，需群组 Maintainer/Owner；**只作为新项目的默认值**，项目建好后 Maintainer/Owner 可改 —— 老项目要逐个核对项目侧 `Variables` 设置 |

> **实例级有个反直觉细节**：Admin → Settings → General → **Enabled Git access protocols** 选 **Only SSH** 之后，**CI Job Token 仍然可以走 HTTPS 做 clone/fetch** —— 因为 Runner 依赖它。"禁了 HTTPS"不等于"外部只能用 SSH"。

---

# 4. 令牌治理与故障排查（本篇最实用的一节）

## 4.1 生命周期规则

| 规则 | 事实 |
|------|------|
| 过期时刻 | **在过期日期当天的 UTC 零点**失效（不是"当天 23:59"） |
| 过期后表现 | `401 Unauthorized`；**UI 里不再显示**该令牌 |
| GitLab.com 连带效应 | 同一 IP 短时间大量认证失败 → 返回 `403 Forbidden`（限流/封禁），"令牌没问题但突然全 403"要想到这条 |
| 轮换 | **rotate 会立刻让原令牌失效**，且**沿用原有 scope，无法补上缺失的 scope**（缺 scope 只能新建） |
| 撤销后可恢复 | **仅 Self-Managed/Dedicated**：管理员在 Rails 控制台把 `revoked` 改回 `false`（项目/群组令牌底层就是 PAT 记录，用 `bot_namespace` 判断归属），且**必须在"未使用的 project/group 令牌保留期"结束前恢复**，超期记录已被删除；**GitLab.com 不支持恢复** |
| 升级后"令牌莫名过期" | 老版本曾给"永不过期令牌"自动补过期时间；**17.3 起不再自动设置**。可用 access token Rake tasks 分析/延长/清除 |
| 管理员可全局关令牌 | 有实例开关可关闭 access token 的使用；被关掉时所有令牌请求都会失败 |
| 使用记录 | 令牌详情含 `last_used_at` / IP；**使用时间是每 10 分钟、IP 每 1 分钟批量更新** —— 刚调用完看不到更新是正常的；若始终不更新，说明请求根本没到 GitLab |

## 4.2 "令牌是 active，请求却失败"的官方归因表

这是最常被误判为"GitLab 有 bug"的场景。`active` 只表示"存在且未过期未撤销"，**不表示"能发起这个请求"**：

| 原因 | 处理 |
|------|------|
| 缺少该请求需要的 **scope** | 新建带正确 scope 的令牌（rotate 补不了） |
| Project/Group token 的**角色**不够 | 令牌权限 = min(角色, scope)；给令牌更高的角色（需把它作为成员加入） |
| 令牌类型够不到目标资源 | PAT 只能访问"这个人能访问的"；project token 只能碰本项目；group token 只能碰本群组及其子项目 |
| **群组 IP 限制**拦截 | 表现为 **404**；从允许的出口发起，或让顶层群组 Owner 加白 |
| **External authorization** 开启 | **PAT 与 Project token 无法访问容器/包仓库**；需要走支持该机制的身份或关闭外部授权 |
| 管理员关闭了 access token | 找管理员恢复 |
| 令牌被撤销 / 已 rotate 但仍用旧值 | 换新值；Self-Managed 可恢复（4.1） |

## 4.3 用日志定位：`api_json.log` 的认证失败字段（17.2+）

```bash
# 在 GitLab 节点上（Omnibus 路径）
sudo grep '"status": 401' /var/log/gitlab/gitlab-rails/api_json.log* \
  | jq -r 'select(."meta.auth_fail_reason") | [."meta.auth_fail_reason", ."meta.auth_fail_token_id", ."meta.auth_fail_requested_scopes", ."meta.auth_fail_token_type", ."meta.auth_fail_auth_header_type"] | @tsv' \
  | sort | uniq -c | sort -nr | head
```

| 字段 | 取值 |
|------|------|
| `meta.auth_fail_reason` | `token_expired` / `token_revoked` / `insufficient_scope` / `impersonation_disabled` |
| `meta.auth_fail_token_id` | 例 `PersonalAccessToken/12`（直接给出要处理的令牌 ID） |
| `meta.auth_fail_requested_scopes` | 请求所需的 OAuth scope（空格分隔） |
| `meta.auth_fail_token_type` | `PersonalAccessToken` / `CiJobToken` / `unknown` |
| `meta.auth_fail_auth_header_type` | `private_token_header` / `private_token_param` / `bearer` / `other` |

> **注意**：这些字段偶尔会出现在非 401 的响应上（公开项目的 Git HTTP 请求、Unleash 端点、Workhorse `/authorize` 预授权），**别把它当"必然失败"的证据**。18.9+ 还会带上 `meta.user`。

## 4.4 令牌盘点（把"谁在用哪个令牌"变成可重复跑的脚本）

```bash
# 1) 全实例 PAT（管理员返回全部，非管理员只返回自己的）
glab api "personal_access_tokens?revoked=false&per_page=100" \
  | jq -r '.[] | select(.expires_at==null) | "NO-EXPIRY\t\(.user_id)\t\(.name)\t\(.scopes|join(","))"'

# 2) 最近 30 天没用过、但还没过期的令牌（优先下线）
glab api "personal_access_tokens?state=active&last_used_before=$(date -u -d '-30 days' +%Y-%m-%dT%H:%M:%SZ)" \
  | jq -r '.[] | "\(.id)\t\(.name)\t\(.last_used_at)"'

# 3) 项目/群组访问令牌（含绑定的 bot 用户与角色）
glab api "project_access_tokens" | jq -r '.[] | "\(.name)\tactive=\(.active)\texp=\(.expires_at)\tsc=\(.scopes|join(","))"'
glab api "group_access_tokens"   | jq -r '.[] | "\(.name)\tactive=\(.active)\texp=\(.expires_at)"'

# 4) Deploy Token 只能逐项目/群组列（没有全局端点），配合群组项目列表批量跑
```

> 盘点输出请**落到 Issue/表格里带负责人**，"看过但没动作"等于没盘。

---

# 5. 分支与合并门禁

## 5.1 保护分支：`Allowed to merge` 与 `Allowed to push and merge` 是两个不同的开关

| 设置 | 控制什么 | **全部清空时的行为** |
|------|----------|---------------------|
| **Allowed to merge** | 谁能通过 MR 合并，以及能否通过 UI/API 创建新的保护分支 | **没人能合并**（除非他有 push and merge） |
| **Allowed to push and merge** | 谁能**直接 push** 到该保护分支（同时也具备通过 MR 合并的能力） | 没人能直接 push |

> **坑**：`Allowed to push and merge` **自带 merge 能力** —— 你以为"只给了推送权限"，实际也把合并权限给了。UI 里选 **No one** 会清掉其它角色选项，但 **API 可以同时设置多个 access level**，两边行为不完全一致。

**官方给出的生产分支推荐配置**：

```text
Allowed to merge            = Maintainers
Allowed to push and merge   = No one          ← 强制所有人走 MR
Require code owner approval = on
Approvals required          = ≥2（关键库）
Allow force push            = off
```

## 5.2 通配符与优先级（反直觉，必查）

- 保护分支支持 `release-*`、`production/*`、`*gitlab*` 这类通配，但 **分支名与通配都是大小写敏感**（`dev` ≠ `Dev`）。
- **多条规则命中同一分支时，"最宽松的那条"决定行为**（官方原话：the most permissive rule controls how the branch behaves）。所以加通配前先想清楚：你以为 `release-*` = No one 收紧了，实际可能被某条 `*` = Maintainers 直接盖过去。
- 官方对通配场景的配套建议：**`Allowed to push and merge` 要设成比 `Allowed to merge` 更宽（broader）的用户集合**，合并控制才符合预期 —— 与 5.1 的"push and merge 自带 merge 能力"一起读，结论是：**用 `Allowed to merge` 收紧合并资格没有意义，真正要管的是 `Allowed to push and merge` 给了谁**。
- 谁能创建/管理保护分支：Maintainer/Owner，或带 `admin_protected_branch` 权限点的自定义角色；**群组级保护分支**在群组设置里配，用于"组内新项目自动保护"。

## 5.3 实例与群组的默认保护

```bash
# 实例默认保护级别（Admin → Settings → Repository and network 里的 default branch protection）
glab api /settings | jq 'with_entries(select(.key|test("default_branch_protection";"i")))'
# 某项目当前保护了哪些分支、允许谁 push/merge/unprotect
glab api "projects/$P/protected_branches" | jq -r '.[] | "\(.name)\tmerge=\(.merge_access_level)\tpush=\(.push_access_level)\tforce=\(.allow_force_push)\tunprotect=\(.unprotect_access_level)"'
```

> **`unprotect_access_level` 是最容易被忽略的口子**：能 unprotect 就等于能绕过所有保护。生产库建议设成 **No one / 仅 Owner**（UI 与 API 都要设，API 可多值并存）。

## 5.4 CODEOWNERS：路径级审批权

```text
# CODEOWNERS 或 .gitlab/CODEOWNERS（项目根或 .gitlab/ 下均可）
/docs/                              @tech-lead @docs
/src/main/java/com/acme/pay/**      @payments-team
/pom.xml                            @platform-build
*.sql                               @dba
```

- 只有**配合保护分支上的 "Require code owner approval"** 才会阻断合并；CODEOWNERS 单独存在只是一份文本。
- 规则**必须能命中实际路径**：写成 `/src/main/java/com/acme/payment/**` 但代码在 `pay/`，等于没有规则。
- **候选审批人不足时 GitLab 会显示 `Auto approved` 并直接放行**（第 15 篇 4.2 详解）—— 上线当天必须专门造一次"该批的人不在"的用例。
- 想让 Code Owner 之外的人（包括 Maintainer）不能独自合并关键路径：群组级设置 **Prevent project maintainers from approving**（Premium/Ultimate）。

## 5.5 策略可以盖过项目设置

若实例启用了 **MR 审批策略 / 安全策略**（`.gitlab/security-policies/policy.yml`，Ultimate），项目里看到的审批与保护设置**可能被策略强制改写**，包括"能否 unprotect/删分支、能否 force push"。

> 排查顺序永远是：**项目设置 → 群组设置 → 策略（policy）→ 实例默认**。看到"我明明改了又变回去"，基本就是策略在兜底重刷。

---

# 6. Push Rules：Git 层的最后一道闸

Push rules 是 **pre-receive 校验**：不合法的 push 在进仓库前就被拒，MR 阶段看不到。

## 6.1 三层规则的语义（这是最容易误解的地方）

| 层级 | 配置入口 | 生效范围（官方口径） |
|------|----------|---------------------|
| **Global（实例）** | Admin Area → **Push rules** | **只是"新项目模板"**：配置后新建的项目继承一份副本；**已有项目不受影响**；**之后修改全局规则也不会回刷已配置过的项目** |
| **Group**（Premium/Ultimate） | 群组 **Settings → Repository → Pre-defined push rules** | 只作用于该群组内**新建的项目**；**子群组不会继承**（要到子群组里单独配）；项目从"最近的、定义了推送规则的父群组"继承 |
| **Project** | 项目 **Settings → Repository → Push rules** | 项目自身的规则**独立存在并替换**之前的规则，不与全局合并 |

**结论**：**GitLab 没有"实例级统一收紧推送规则"的原生能力**，只能靠"改全局模板 + 脚本逐项目下发 + 周期性校验"三件套（第 9 节给脚本）。想验证效果，官方建议就是"在子群组里建一个测试项目，看它的 push rules 是什么"。

## 6.2 可用校验项与坑位

| 校验 | 字段（API 名，逐项目 `GET /projects/:id/push_rule` 可见） | 坑 |
|------|------|----|
| 阻止推送密钥文件 | `prevent_secrets`（内置名单见 `ee/lib/gitlab/checks/files_denylist.yml`：`.aws/credentials`、`id_rsa`/`id_ed25519` 等、`*.pem`、`*.key`…） | **按"文件名模式"匹配，不分析内容**；对**已提交进仓库的文件无效**；**已有项目必须逐个更新配置才生效**（`prevent_secrets` 只在新建项目时继承） |
| 禁止文件名正则 | `file_name_regex` | 只对"仓库中原本不存在"的新增/更新文件生效；空值 = 不限制；大小写与路径写法要实测 |
| 最大文件体积 | `max_file_size` | **单位 MB**，`0` = 不限制，**Git LFS 文件豁免** —— 大 jar/模型文件被拒先查这里，而不是网络 |
| 要求提交签名 | `reject_unsigned_commits` | **只校验外部 `git push`**；GitLab UI/API 产生的提交（Web IDE、MR 操作、API 建文件）**免签** → 历史里仍会出现未签名提交。且开启后 **Web IDE 默认不能提交**，要管理员 `Feature.disable(:reject_unsigned_commits_by_gitlab)` 才放开。支持 GPG / SSH / X.509，签名无效或损坏直接拒 |
| 作者必须是 GitLab 用户 | `member_check`（UI 勾"Check whether the commit author is a GitLab user"） | **author 与 committer 两个邮箱都必须命中某个 GitLab 用户的已验证邮箱**；**bot 用户（项目/群组令牌生成的机器人账号）必须把其生成的邮箱后缀加入白名单**，否则 Renovate/自动发版全被拦 |
| 拒绝无法验证的提交者 | `commit_committer_check` | committer 邮箱必须是"该用户自己的已验证邮箱/私有提交邮箱"之一 |
| 拒绝与账号名不一致的作者名 | `commit_committer_name_check` | 仅在 author 与 committer 邮箱相同时检查（cherry-pick/rebase 场景会跳过）；**它防的是配置错误，不防冒充** —— 防冒充用签名 |
| 作者邮箱正则 | `author_email_regex` | author 与 committer 邮箱都要匹配；留空 = 允许任意邮箱 |
| 提交信息正则 | `commit_message_regex` / `commit_message_negative_regex` | 与 MR 标题正则（`merge_request_title_regex`）是两套，别混 |
| 分支名正则 | `branch_name_regex` | 用于强约束 `feature/xxx-123` 这类命名 |
| 禁止删除标签 | `deny_delete_tag` | 发布标签防误删，与"受保护标签"配合（第 5 篇） |
| DCO | `reject_non_dco_commits` | 开源合规场景，要求提交带 `Signed-off-by` |

> **两条边界**：① 除 `deny_delete_tag` 外，**推送规则作用于 commit，不作用于 tag**；② GitLab 自己签发的提交用的是实例邮箱（形如 `noreply@example.com`，它不属于任何 GitLab 用户），所以"拒绝无法验证的用户 / 作者必须是 GitLab 用户 / 作者邮箱正则"这几条**对这类提交不检查 committer 邮箱，只检查 author 邮箱**（19.3 起该豁免同样适用于镜像同步来的提交）。

```bash
# 逐项目读写 push rules（批量收紧的唯一可靠做法）
glab api "projects/$P/push_rule" | jq '{prevent_secrets, member_check, commit_committer_check, author_email_regex, deny_delete_tag, reject_unsigned_commits, max_file_size, file_name_regex}'
# 项目从未配置过推送规则时，这个端点返回 200 且 body 是字面量 "null" —— 脚本要先判空
glab api --method POST "projects/$P/push_rule" -f deny_delete_tag=true -f member_check=true \
  -f author_email_regex='@acme\.com$'
# 已存在规则时用 PUT；prevent_secrets 是布尔
glab api --method PUT "projects/$P/push_rule" -f prevent_secrets=true -f max_file_size=0
```

> 上面的字段名与取值以 `glab api projects/$P/push_rule` 返回为准（不同版本有增减）。**批量下发前先在 1-2 个项目验证，并记下回滚值**（推送规则一旦配错，全组提不了代码就是即时故障）。

## 6.3 与受保护标签/制品保护的关系

```text
git push ──► Push rules（提交/文件级：文件名、体积、签名、邮箱） ──► 保护分支（分支级） ──► MR 门禁（流程级）
发布 tag   ─► 受保护 tag / immutable tag（谁能推/删） ──► 包保护规则（谁能发/删） ──► 清理策略豁免（会不会被自动删）
```

四层各管一段，**缺任何一层都会出现"绕过"**：只配保护分支、不配推送规则，就挡不住"密钥文件以命中的文件名被合进非保护分支"；只配推送规则、不上 Secret Detection，就拦不住换个文件名提交的令牌。

---

# 7. 安全扫描与门禁（能力边界，配置在第 15 篇）

## 7.1 扫描器定位

| 扫描器 | 查什么 | 文档首页标注的 Tier | 备注 |
|--------|--------|-------------------|------|
| SAST | 源码缺陷/注入 | **Free, Premium, Ultimate** | 分析器多引擎（含 Semgrep 系、Brakeman 等），按语言自动选择 |
| Secret Detection | 硬编码密钥 | **Free, Premium, Ultimate** | MR 里逐条 finding + "创建排除项"的体验较新版本才完善 |
| Container Scanning | 基础镜像 CVE | **Free, Premium, Ultimate** | 基于 Trivy 系；扫描 tar 等能力 18.0 起补强 |
| IaC Scanning | Terraform/K8s 等配置错误 | **Free, Premium, Ultimate** | 与 MR 报告联动的能力层级更高 |
| **Dependency Scanning** | 依赖 CVE（含传递依赖） | **文档标注 Ultimate** | 别照抄旧教程的"免费可跑"；SBOM 比对、恶意包匹配（19.4/19.5）层级更高 |
| **Secret Push Protection** | **推送期**就拦密钥 | **文档标注 Ultimate** | 免费层的替代是 Push Rules 的 `prevent_secrets`，但那只按**文件名**拦 |
| DAST / API Fuzzing / Coverage-Guided Fuzzing | 运行期行为 | Ultimate（按需） | 需要一个能被扫描器访问到的部署地址（与第 9 篇的 Environment 联动） |
| License Scanning | 许可证合规 | 以实例为准 | 层级在版本间反复调整 |

> **不要背这张表的 Tier 列**：GitLab 多次把扫描能力下放/回收（又把"治理与阻断"留在付费层），而 **"job 能不能创建"与"finding 能不能进面板 / 能不能卡合并"是两个不同的层级**。落地前用 **CI Lint API** 看展开结果（这一步不会真的跑流水线）：
>
> ```bash
> # 把待验证的 yml 作为 content 提交给 Lint API，要求返回合并后的 YAML
> jq --null-input --arg yaml "$(cat probe.yml)" '.content=$yaml' \
>   | curl -s --request POST \
>       --url "$GL_URL/api/v4/projects/$P/ci/lint?include_merged_yaml=true" \
>       --header "PRIVATE-TOKEN: $TOKEN" --header 'Content-Type: application/json' \
>       --data @- \
>   | jq -r 'if .valid then (.merged_yaml | fromjson) else (.errors // .warning_messages) end' \
>   | grep -E '^[A-Za-z_.-]+:'
> ```
>
> `probe.yml` 只放 `include:` 与 `stages:`。展开后**能看到 `dependency_scanning` / `secret_detection` 等 job 名**，说明模板在这台实例上可用；若返回里带 `errors`，按报错原文处理而不是猜订阅层级。

## 7.2 门禁的正确形状

```yaml
include:
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml

# 意图必须写进配置：哪些卡、哪些不卡
secret_detection:
  allow_failure: false        # 密钥：不接受"以后改"，泄露即失效
sast:
  allow_failure: true         # 存量治理期先只报不卡
  artifacts:
    paths: [gl-sast-report.json]   # 报告产物，供后续统计/导入
```

- **"阻断合并"有三种实现**：① job `allow_failure: false`（最简单，全有全无）；② MR 设置 **Pipelines must succeed**；③ **Approval rule 基于扫描报告**（`report_approver`，Ultimate）—— 只有 ③ 能做到"新增高危才卡"。
- **漏洞管理面板、SLA 倒计时、按严重度阻断** 属于治理能力，需要更高订阅；MR 页里的 **Security 报告**标签同理。
- **Secret Push Protection**（推送期就拦，**文档标注 Ultimate**）与 Push Rules 的 `prevent_secrets` 是两条互补的路：前者看**内容**（真实 token 形态），后者只看**文件名**。免费层只能靠后者 + 流水线期的 Secret Detection，所以"密钥不进仓库"在免费层的正确承诺是"**流水线拦住合并**"，而不是"推送即拦"。

> **模板路径写法**：本系列统一用 `- template: Security/<Scanner>.gitlab-ci.yml`；官方文档现在多用 `- template: Jobs/<Scanner>.gitlab-ci.yml`。**两者都在仓库的模板目录里存在**（`lib/gitlab/ci/templates/{Jobs,Security}/`），但**不是每个扫描器都两个都有**（例如 DAST 只有 `Security/DAST.gitlab-ci.yml`）。以 CI Lint 展开结果为准，别靠记忆。

## 7.3 供应链这一根线

```text
commit(签名/SSH 证书) ─► MR(SAST/密钥/依赖) ─► 构建(基础镜像扫描 + SBOM) ─► 制品库(tag 保护 + 签名)
        │                                                                   │
     审计事件 ◄────── 安全中心(漏洞/SLA) ◄────────────────────────── 部署(准入验签)
```

- **签名**：cosign v2+ 且 `--registry-referrers-mode oci-1-1`（第 10 篇 3.4）；展示签名依赖 registry metadata database。
- **SBOM**：随构建产出并挂到 Release；核对"这个版本用了哪些依赖"是事故时的第一问。
- **依赖混淆（dependency confusion）**：删除包 + 开启上游转发 = 风险。实例级转发开关是 `maven_package_requests_forwarding` / `npm_package_requests_forwarding` / `pypi_package_requests_forwarding`（`glab api /settings | jq` 读回）。

## 7.4 内网/离线实例接扫描（自建 Java 团队最常撞的墙）

内网 GitLab 跑扫描，**卡点从来不是订阅，而是"扫描器要出网"**。官方把离线要求拆成四块，缺一块就出现"job 红了"或"job 绿了但报告是空的"：

| 要搬进来的东西 | 官方口径 | 踩坑表现 |
|----------------|----------|----------|
| **① Analyzer 镜像** | 从 `registry.gitlab.com/security-products/<analyzer>:<major>` 复制到**内网 registry**，然后用一个 CI/CD 变量把整套模板的分析器前缀改掉：`SECURE_ANALYZERS_PREFIX: "harbor.internal.acme.com/analyzers"`（SAST、Secret Detection 等都用它） | Runner 的 docker executor **默认 `pull_policy: always`**，即使本地已有镜像也会去外网拉。改成 `if-not-present` 能用本地镜像，但**会错过分析器更新** —— 非离线场景不要这么设 |
| **② 镜像的持续更新** | 在一个**新建空项目**里 `include: template: Security/Secure-Binaries.gitlab-ci.yml`：流水线把分析器镜像存成 **job artifact**（或推到该项目容器仓库），再离线 `docker load`。执行它的 Runner 必须**同时能访问 gitlab.com / registry.gitlab.com 与内网实例**，且要 **privileged**（job 里要用 `docker`），一般放 DMZ/跳板机、只服务这个项目 | 该模板**默认只在提交 `.gitlab-ci.yml` 时跑一次** —— 不配 **pipeline schedule** 就再没有新签名；而且**它不包含 Container Scanning 分析器**，容器扫描要单独按自己的离线章节处理 |
| **③ 规则/漏洞数据库** | 各扫描器默认联网拉最新签名与规则；离线要么关掉自动更新、用镜像内置库并手工更新，要么指向内网副本。Container Scanning（Trivy 系）另需考虑 **Java 漏洞库镜像**；Secret Detection 的 analyzer 镜像是 `security-products/secrets:<major>` | "扫描结果永远不带新 CVE"就是这一层没更新，**跟订阅无关**，别去升级套餐 |
| **④ Package Metadata Database** | **Dependency Scanning 与 Continuous Vulnerability Scanning 读的是"包元数据数据库"这个数据制品，不是 analyzer 镜像** —— 只搬镜像不够，离线实例在拷入导出文件之前**根本没有这份数据** | 典型症状：**job 全绿、`gl-dependency-scanning-report.json` 是空的**；这时先怀疑 ④，而不是 ① |

```yaml
# 内网实例：所有安全分析器统一走内网镜像前缀
variables:
  SECURE_ANALYZERS_PREFIX: "harbor.internal.acme.com/analyzers"
  ADDITIONAL_CA_CERT_BUNDLE: "-----BEGIN CERTIFICATE-----\nMIID..."   # 自签 CA 时必须给，否则 analyzer 里 curl 直接失败

include:
  - template: Jobs/Secret-Detection.gitlab-ci.yml
  - template: Security/SAST.gitlab-ci.yml
```

**排障顺序**（比查任何二手资料都快）：

```bash
# ① 先看 job 日志里镜像到底从哪拉、失败在哪一步
glab api "projects/$P/jobs/$JOB/trace" | grep -E 'Pulling|Pull policy|ErrImagePull|ImagePullBackOff|dial tcp|certificate'

# ② 确认展开后的分析器镜像名是否已被 SECURE_ANALYZERS_PREFIX 替换
jq --null-input --arg yaml "$(cat .gitlab-ci.yml)" '.content=$yaml' \
  | curl -s --request POST --url "$GL_URL/api/v4/projects/$P/ci/lint?include_merged_yaml=true" \
      --header "PRIVATE-TOKEN: $TOKEN" --header 'Content-Type: application/json' --data @- \
  | jq -r '.merged_yaml | fromjson' | grep -E 'image:'

# ③ 报告为空时，验证是不是缺 Package Metadata Database
glab api "projects/$P/jobs/$JOB/artifacts/gl-dependency-scanning-report.json" | jq '{scanner, dependencies: (.dependencies|length), vulns: (.vulnerabilities|length)}'
```

> 各扫描器都有**自己那一页的 offline 章节**（Container Scanning / SAST / Secret Detection / DAST / API Fuzzing / License Scanning 各不同）。**变量与镜像 tag 会随版本变化，务必以你实例上对应文档页为准**，不要长期抄这一份表。

---

# 8. 实例侧安全基线（自建）

## 8.1 身份与登录

| 项 | 位置 | 要点 |
|----|------|------|
| 注册开关 | Admin → Settings → General → **Sign-up restrictions** | 生产实例应关闭自助注册（`signup_enabled=false`），改为 LDAP/OIDC/SAML 或管理员预建 |
| 强制 2FA（所有人） | Admin → Settings → General → **Sign-in restrictions** → **Enforce two-factor authentication** | API 键 `require_two_factor_authentication`；**含管理员** |
| 2FA 宽限期 | 同页 **Two-factor grace period**（`two_factor_grace_period`，单位小时） | **填 `0` = 下次登录就必须登记 2FA**；宽限期后未配置的人被挡在门外 |
| 强制 2FA（仅管理员） | 同页 **Enforce two-factor authentication for administrators** | API 键 `require_admin_two_factor_authentication`；**同样覆盖"持有自定义管理员角色的普通用户"** —— 这条是"不能只靠自觉"的兜底 |
| Email OTP | 同页需**同时**勾选 **Enable email-based one-time passwords** 与 **Require email verification when account is locked** | 少勾一个不生效 |
| 密码/通行密钥 | 同页 **Password and passkey authentication** | 可关闭 Web 密码登录、关闭 Git over HTTP(S) 的密码认证、对"已有 SSO 身份的用户"禁用密码登录 |
| 未知地点登录提醒 | 同页 **Email notification for unknown sign-ins** | 低成本、高价值：异常登录第一时间到人邮箱 |
| **Admin Mode** | 同页 **Enable Admin Mode**（API `admin_mode=true`；Rails `::Gitlab::CurrentSettings.update!(admin_mode: true)`） | 开启后**对全实例管理员生效**：管理员默认**进不了 Admin 区**，每次执行管理操作都要重新认证。等于给"最高权限"加了二次确认，是审计最爱问的一项 |
| 休眠账号自动处置 | Admin → Settings → General → **Account and limit** → Dormant users | 勾选后按"无活动天数"（**默认与下限都是 90 天**）每天跑一次任务，**每天最多停用 10 万人** |
| 未确认邮箱自动删除（Premium/Ultimate，Self-Managed） | Rails：`Gitlab::CurrentSettings.update(delete_unconfirmed_users: true)`、`unconfirmed_users_delete_after_days: 365` | 每小时跑一次，只在开启邮箱校验时生效 |
| Access token 总开关 | Admin 设置（"disable access tokens"） | 泄露事件应急时可临时关掉所有令牌访问 |

## 8.2 账号处置：block / deactivate / ban / delete 选错就是事故

| 动作 | 语义（官方口径） | 适用 |
|------|------------------|------|
| **Block** | **不能登录、不能访问任何仓库**；数据保留；**不占席位**；LDAP 封禁的用户需先删 LDAP identity 或（19.5+）**Resync with LDAP** | **离职、攻击、可疑账号 —— 首选** |
| **Deactivate** | 面向**休眠用户**腾席位：判定依据是 `last_active_at`；**用户重新登录即可自助恢复**；对"非 dormant"用户调 API 直接返回 `403` | **绝不是离职手段**，只是省 license 的机制 |
| **Ban** | Block + **隐藏其贡献** | 骚扰/滥用内容需要"消失"时 |
| **Delete** | 可选"只删用户"或"连同贡献一起删（贡献移入 ghost user）" | 谨慎：**审批记录是按 user ID 关联的，其他贡献没有 user ID 关联**，删号后 approval 记录会指向失效 ID —— 官方因此建议优先 block/ban/deactivate |

> **离职回收的正确顺序**（很多团队第 4 步才做，事故就出在这里）：
>
> 1. **Block 用户**（立刻断登录与仓库访问）。
> 2. **撤销该用户名下所有 PAT**：`glab api "personal_access_tokens?user_id=$UID&revoked=false" | jq -r '.[].id'` 后逐个 DELETE。
> 3. **盘点 TA 创建的 bot 令牌**：Project/Group Access Token 属于**项目/群组的 bot 用户**，**不随人停用而失效** —— 人走了，`ci-deploy-gitlab-production` 照样能推镜像。必须按 4.4 的脚本逐项目/群组过一遍，改owner 或直接撤销。
> 4. 移交 Owner 角色与项目归属（群组 Owner 不足会卡住移交，删库/转移需要 Owner）。
> 5. 检查其名下 **Webhook / Deploy Token / Runner 注册 / CI 变量**（变量里塞过 PAT 的话，第 2 步撤销后流水线会当场红）。
> 6. **不要顺手 Delete**：等审计与审批链核对完成再说（上表已说明原因）。

## 8.3 出站与内容风险（SSRF）

```bash
# 读回当前实例的"本地请求"策略：Webhook 默认是允许访问内网的！
glab api /settings | jq '{
  allow_local_requests_from_web_hooks_and_services,
  allow_local_requests_from_system_hooks,
  enabled_git_access_protocol,
  asset_proxy_enabled, asset_proxy_url, asset_proxy_allowlist
}'
```

| 风险 | 处置 |
|------|------|
| Webhook/集成可被用来探测内网（ SSRF 的经典入口） | 关掉 `allow_local_requests_from_web_hooks_and_services`；`allow_local_requests_from_system_hooks` 默认更严，别顺手放开 |
| Markdown 里的外部图片泄露访问者 IP / 混合内容告警 | 启用 **Asset Proxy（Camo）**：`asset_proxy_enabled` + `asset_proxy_url` + `asset_proxy_allowlist`（旧字段名 `asset_proxy_whitelist` 仍会出现在响应里，以读回值为准） |
| 只想要 SSH | `enabled_git_access_protocol=ssh`（或群组级单独设），**但 CI Job Token 仍可走 HTTPS 拉代码**（3.4 已说明） |
| Git/包/issue 创建被刷 | `administration/settings/` 下的各类 `*_rate_limits` 页（git http、LFS、files API、package registry、import/export、incident management…），按需收紧 |

## 8.4 可见性与误操作防护

Admin → Settings → General → **Visibility and access controls**：

| 开关 | 语义与坑 |
|------|----------|
| **Default minimum role required to create projects**（No one / Administrators / Owners / Maintainers / Developers） | **只是"新群组的默认值"**，已存在的群组保留各自权限 —— 收紧后要逐群组核对群组自身的"谁能加项目"。另外 `allow_project_creation_for_guest_and_below`（默认 `true`）管的是"Guest 及以下能否建群组与个人项目"，两者不是一回事。若选 **Administrators** 且开了 **Admin Mode**，管理员建项目必须先进 Admin Mode |
| **Allowed to delete projects**（Administrators / Owners and administrators） | **Premium/Ultimate，Self-Managed/Dedicated**；选 Administrators 后连项目 Owner 都删不掉，配合项目级 **Deletion protection** 才是完整的防误删 |
| **Default project / group / snippet visibility** | 默认 Private 是底线 |
| **Restricted visibility levels** | 限制 Public ⇒ 只有管理员能建 public 群组/项目/片段，且**用户 profile 仅登录可见、GraphQL 不再暴露用户属性**；限制 Internal/Private 同理。**两条硬限制**：① **不能 restrict 当前被设为默认值的层级**（二者互斥）；② **对个人命名空间下新建的项目不生效** |
| **Enabled Git access protocols** | 见 3.4（含"Job Token 例外"） |
| **Customize Git clone URL for HTTP(S)** | 反向代理/多域名场景改 clone 链接，防"页面显示内网域名 clone 不到" |
| SSH key 类型与数量 | 可限制 RSA/DSA/ECDSA/ED25519 及 `*_SK` 变体的启用与数量 |

```bash
glab api /settings | jq 'with_entries(select(.key|test("creation_level|visibility|deletion|adjourned|enabled_git_access";"i")))'
```

> `deletion_adjourned_period`（删除冷静期，天）也在这一组里：**它给的是"删错了还能恢复"的时间窗**，别为了"省磁盘"把它设成 0。

---

# 9. 审计、合规与"可追溯"

## 9.1 四层审计事件与"看得到"的门槛

| 层级 | 入口 | Tier / Offering | 谁能看 |
|------|------|-----------------|--------|
| **Sign-in** | 个人头像 → **Edit profile** → **Access → Authentication log** | **唯一在所有订阅层（含 Free）都可用的审计事件**（成功登录） | 本人 |
| **Group** | 群组 → **Secure → Audit events** | Premium, Ultimate；GitLab.com / Self-Managed / Dedicated | 看**全组所有人**：需该群组 **Owner**；只看**自己的操作**：**Developer** 起 |
| **Project** | 项目 → **Secure → Audit events** | Premium, Ultimate；三种 offering | 看**项目内所有人**：需项目 **Maintainer**；只看自己：**Developer** 起 |
| **Instance** | **Admin → Monitoring → Audit events** | Premium, Ultimate；**仅 Self-Managed**（过滤器：被操作的用户/群组/项目 + 日期区间） | 管理员 |

> **Auditor 账号的真正的价值就在这张表**：拥有 **Auditor access level** 的用户可以**看到所有群组与项目的审计事件**，不必被逐个加成成员 —— 所以"给审计部门 Reporter"是既越权又不够用的错误做法。

**查询与导出的硬约束（都踩过才知道）**：

- **API 与 UI 的日期范围最多 30 天**（`created_after` / `created_before` 差值上限 30 天）—— 按季度拉数据必须自己切片。
- 实例审计事件的查询受**可配置的按用户速率限制**约束（Admin 设置里的 audit events API rate limit），SIEM 集成一次拉太猛会直接 403/429。
- **Export as CSV 上限 100,000 条**，超出部分**静默截断** —— 交付审计材料前一定要核对条数，而不是"导出成功 = 全量"。CSV 用 `,` 分隔、必要时 `"` 包裹、`\n` 分行，首行是表头。
- **时区三套**：UI 用你的本地时区；API 默认 **UTC**（Self-Managed 可按实例配置时区）；**CSV 一律 UTC**。对账脚本请统一转 UTC 再比。
- **模拟登录会留双份痕迹**（Premium/Ultimate，Self-Managed）：被模拟用户的操作事件里会**带上执行模拟的管理员信息**，并且**会话开始与结束各额外记录一条** —— 这是回答"管理员凭什么能看私有仓库"的标准证据链。

```bash
# API 侧（三个端点同构），接 SIEM 前先用它验证事件结构与字段
glab api "groups/$TOP_GROUP_ID/audit_events?created_after=$START&created_before=$END&per_page=100" \
  | jq -r '.[] | "\(.created_at)\t\(.author.name)\t\(.type)\t\(.details.target_type // "")\t\(.details.entity_path // "")"'
glab api "projects/$P/audit_events?per_page=20" | jq -r '.[] | [.id,.created_at,.type] | @tsv'
# 实例级（管理员令牌，30 天窗口内）
glab api "audit_events?per_page=50" | jq -r '.[] | [.created_at,.type,.details.action // ""] | @tsv'
```

> **给导出自动化选身份**：用**实例服务账号**（instance service account）+ `PUT /users/:id` 设 `auditor=true`（**Premium/Ultimate only**，需管理员），再给它 `read_api` + `read_repository` scope 的令牌 —— 官方明确写了：这样的服务账号**能列出、读取并 clone 它不是成员的私有项目**，所以不需要把它加成任何项目的成员。**不要用某个管理员的 PAT 跑导出**：人一走，采集断掉，而且审计日志里全是"管理员在动"。

**用法要点**：

- 审计事件是**回答"谁在什么时候改了门禁"的唯一可信来源**（UI 的 Activity 页只反映部分事件，且不同角色可见范围不同）。
- 想长期留存就要**导出/外发到你自己的日志系统**；较新版本还支持把审计事件**投递到外部目标**，目标类型（对象存储/Syslog/自定义等）随版本与订阅变化 —— **落地前先在实例 UI 里确认有哪些目标可选**，别照抄别人的对接文档。
- 常见必查事件：成员增删、角色变更、保护分支/审批规则改动、令牌创建与撤销、CI 变量新增、Runner 注册、Webhook 变更、群组 IP 限制变更。

## 9.2 合规框架与安全策略（Ultimate 为主）

| 能力 | 用途 |
|------|------|
| **Compliance Frameworks** | 把"这个群组必须：强制签名 + 二级审批 + 保留策略 + 扫描全开"打包，项目创建即继承；还能限制可见性/禁用 fork 等 |
| **MR approval policies / pipeline execution policies** | 以策略形式**覆盖**项目设置（5.5）；可用于"必须有某类扫描 job 才允许合并" |
| **Audit event reports** | 定期生成合规报表（给审计方的交付物） |
| **Auditor 用户** | 只读审计身份，不给 Reporter |
| **Security Center / Dashboard** | 跨项目漏洞聚合与 SLA |
| **Secrets Manager / 凭据清单** | 把 CI 密钥接外部 Vault，并提供"实例里有哪些凭据"的盘点入口 |

## 9.3 治理脚本（把规范变成可重复检查）

```bash
# ① 门禁体检：列出组内所有项目的 main 保护情况与"能否被 unprotect"
for p in $(glab api "groups/payments/projects?per_page=100" | jq -r '.[].path_with_namespace'); do
  enc=${p//\//%2F}
  glab api "projects/$enc/protected_branches" \
    | jq -r --arg p "$p" '.[] | select(.name=="main") | "\($p)\tunprotect=\(.unprotect_access_level)\tforce=\(.allow_force_push)\tpush=\(.push_access_level)"'
done

# ② 推送规则体检：哪些项目没开"禁删 tag"和"作者必须是用户"
for p in $(glab api "groups/payments/projects?per_page=100" | jq -r '.[].path_with_namespace'); do
  enc=${p//\//%2F}
  glab api "projects/$enc/push_rule" 2>/dev/null \
    | jq -r --arg p "$p" 'select(.deny_delete_tag!=true or .member_check!=true) | "PUSH-RULE-GAP \($p)"'
done

# ③ 群组 IP/域名限制是否配在正确层级（只有顶层群组能设）
glab api "groups/payments" | jq '{id, full_path, parent_id, ip_allowlist: (.ip_restrictions // "n/a")}'

# ④ 令牌健康度：无过期时间 / 长期未使用（见 4.4）
# ⑤ 扫描覆盖率：哪些项目 30 天没跑过 secret_detection
glab api "groups/payments/projects?per_page=100" | jq -r '.[].path_with_namespace' | while read -r p; do
  enc=${p//\//%2F}
  n=$(glab api "projects/$enc/jobs?scope[]=success&per_page=100" 2>/dev/null | jq '[.[]|select(.name=="secret_detection")]|length')
  [ "$n" = "0" ] && echo "NO-SCAN  $p"
done
```

> 这些脚本进 CI 定时任务（第 6 篇 schedule 触发），输出**自动开 Issue 并指派安全负责人** —— 治理的可持续性来自自动化，不是来自会议。

---

# 10. 泄露事件响应 Runbook

**前提认知**：清理 Git 历史**不能**撤销"别人已经 clone/CI 已缓存"这一事实。**先吊销，再谈清理。**

```bash
# ① 立即定位影响面：这个令牌/密钥被用过吗、从哪用的
glab api "personal_access_tokens?search=<token_name>" | jq '.[] | {id, last_used_at, scopes, expires_at}'
sudo grep '<token_fingerprint>' /var/log/gitlab/gitlab-rails/api_json.log* | jq -r '[.time,.path,.meta.auth_fail_token_id]|@tsv' | head

# ② 吊销（UI 或 API），并让管理员临时收紧 access token 相关能力
glab api --method DELETE "personal_access_tokens/$TOKEN_ID"

# ③ 轮换下游：CI 变量 / 外部系统 / Deploy Token
glab variable update -R "$P" DEPLOY_TOKEN --value "$NEW"   # 具体子命令以 glab 版本为准

# ④ 如果密钥已进 Git 历史：改写历史 + 强制推送 + 通知所有 fork/CI 缓存失效
#    并同步做：删除镜像里的旧层（第 10 篇清理策略 + GC）、清 Runner 缓存

# ⑤ 事后：把"为什么没在推送阶段被拦住"补上（Push Rules + Secret Detection + 阻断合并）
```

**Self-Managed 特有的两个后门**：

- 误撤销的 PAT/Project/Group token 可用 Rails 控制台恢复：`PersonalAccessToken.find_by_token('…')` 或 `User.find(<bot_user_id>).personal_access_tokens.find(<id>)`，用 `token.user.bot_namespace&.full_path` 确认归属，再 `token.update!(revoked: false)`。**注意：走撤销服务会生成审计事件，直接改库不会** —— 事故处理必须留痕，优先用受支持的撤销/恢复路径。
- Agent token 等无 UI 入口的令牌也支持控制台撤销（同样区分"带审计"与"不带审计"两种做法）。

**恢复动作的红线**：直接改库属于"最后手段"，官方要求先在测试环境演练、并有可恢复的备份（第 13 篇）。

---

# 11. 常见报错原文与处理

| 报错原文（Git 侧为 pre-receive 输出） | 含义 | 处理 |
|--------------------------------------|------|------|
| `You are not allowed to push code to protected branches on this project.` | 该分支受保护且你不在 push 名单 | 改走 MR，或调整 **Allowed to push and merge**（注意它会附带 merge 权限） |
| `You are not allowed to force push code to a protected branch on this project.` | 保护分支禁止强推 | 别绕；确需重写历史走"新分支 + MR" |
| `You are not allowed to merge code into protected branches on this project.` | 你不在 merge 名单 | 请 Maintainer，或调整 **Allowed to merge** |
| `You are not allowed to delete protected branches from this project. Only a project maintainer or owner can delete a protected branch.` | 删保护分支权限不足 | 由 Maintainer/Owner 操作 |
| `You can only delete protected branches using the web interface.` | 该分支只允许在 UI 删除 | 去 UI 删 |
| `The default branch of a project cannot be deleted.` | 默认分支不可删 | 先换默认分支 |
| `You are not allowed to create protected branches on this project.` | 无权创建保护分支 | 需要 Maintainer/Owner 或 `admin_protected_branch` |
| `401 Unauthorized` | 令牌过期/撤销/无效（**过期时刻是过期日 UTC 零点**） | 看 `meta.auth_fail_reason`；重建或轮换 |
| `403 Forbidden`（令牌正常） | scope 不足、角色不足、被实例关闭 access token；**GitLab.com 上还可能同 IP 认证失败过多被限流** | 对照 4.2 表 |
| `404 Not Found`（访问明明存在的资源） | 无权限、项目被隐藏，**或群组 IP 限制拦截（令牌请求）** | 先确认出口 IP 是否加白 |
| `insufficient_scope`（日志字段） | 请求需要令牌没有的 scope | rotate **不会**补 scope，只能新建 |
| `push declined ... secret files / forbidden filename` | Push Rules 命中密钥文件或文件名黑名单 | 移除该文件；确认是否需要在该项目上覆盖全局规则（6.1） |
| `push declined ... unsigned commit` / Web IDE 无法提交 | `reject_unsigned_commits` 生效；UI/API 提交被免签，外部 push 被拒 | 配置签名（GPG/SSH/X.509）；Web IDE 场景需管理员动 feature flag |
| Renovate/机器人 push 被拒（要求作者是 GitLab 用户） | `member_check` + bot 邮箱后缀未加白 | 把 bot 生成的邮箱域名加入 `author_email_regex`/白名单 |
| `410 Gone - runner registration disallowed` | Runner 注册被关（第 8 篇） | 用 `glrt-` 令牌或让管理员建 Runner |
| 镜像/包 push `403`（CI 突然失败） | **受保护包/受保护 tag 规则命中**，或 Deploy Token 推受保护包（第 10 篇） | 核对保护规则与角色，注意规则生效有 JWT 延迟 |
| `unauthorized to access repository`（跨项目拉包） | Job Token 未入站授权 | 被访问项目 **Job token permissions** 加白名单 |

---

# 12. 小结

| 要点 | 一句话 |
|------|--------|
| 权限模型 | 8 个默认角色中**只有 5 个是累积的**（Planner / Security Manager 是专用角色）；细粒度靠"角色 + 保护分支 + 审批 + 推送规则"组合，不靠拆角色 |
| 只读身份 | 审计岗给 **Auditor**（可看全部群组/项目审计事件），**别给 Reporter**（Reporter 能建 MR、下载制品）；导出自动化用**服务账号 + `auditor=true`** |
| 令牌识别 | **PAT / Project / Group / Impersonation 全是 `glpat-` 前缀**，肉眼分不出来；能力由"角色 ∩ scope"决定，行为差异靠命名 + 描述 + API 反查 |
| 令牌选择 | 官方顺序 **job → project → group**；外部只读用 Deploy Token/Deploy key；云侧用 OIDC；**CI 里别放 PAT** |
| 令牌生命周期 | **过期日 UTC 零点失效**；rotate 立即作废原值且**不能补 scope**；Self-Managed 可恢复误撤令牌（要在"未使用令牌保留期"内）；17.3 起不再自动给旧令牌补过期时间 |
| 401/403/404 | active ≠ 可用；**群组 IP 限制拦 Project/Group token 时返回 404**；external authorization 开启后 PAT/项目令牌访问不到容器与包仓库；用 `api_json.log` 的 `meta.auth_fail_*` 直接定位 |
| 访问边界 | IP/域名限制、禁外部 fork、锁定项目加人**只能配在顶层群组**；LDAP Group Sync 生效可能要 1 小时以上；`Only SSH` 之后 Job Token 仍可 HTTPS |
| 分支门禁 | **`Allowed to push and merge` 自带 merge 权限**；**通配多条规则命中时取最宽松**，官方建议 push-and-merge 集合**不窄于** merge；`unprotect_access_level` 是最容易漏的口子；CODEOWNERS 必须配保护分支开关才阻断，候选人不足会 `Auto approved` |
| 推送规则 | 全局/群组规则**只作用于新建项目**、**子群组不继承**、项目规则**替换不合并** → 统一收紧必须脚本下发；拦密钥的字段是 `prevent_secrets`（**按文件名**，不是内容）；`max_file_size` 单位 MB 且 **LFS 豁免** |
| 签名规则 | `reject_unsigned_commits` **不校验 GitLab 自己产生的提交**，还会让 Web IDE 默认不可用 |
| 扫描 | SAST / 密钥 / 容器 / IaC 文档标 Free 可用；**Dependency Scanning 与推送期拦截（Secret Push Protection）文档标 Ultimate**；层级随版本变，用 CI Lint API 展开验证 |
| 内网实例 | 扫描器的真实门槛是**出网**：镜像（Secure-Binaries 模板 + privileged Runner）、签名规则库自动更新、**Package Metadata Database 是独立数据制品**（只搬镜像不够） |
| 实例基线 | 关注册、2FA（全员/仅管理员两个开关，宽限期可 0）、**Admin Mode**、**默认允许 Webhook 访问内网要关掉**、可见性默认 Private、删除保护与冷静期（1-90 天，默认 30）、Asset Proxy |
| 账号处置 | **Block 才是离职/攻击的正确动作**；Deactivate 面向休眠用户且**本人登录即可自助恢复**；Delete 会让审批记录指向失效 ID |
| 审计 | 四层事件、**查询窗口 30 天**、**CSV 导出上限 10 万条且静默截断**、UI 本地时区 / API 与 CSV UTC；模拟登录留双份痕迹 |
| 泄露 | **先吊销再清理历史**；清理历史不等于撤销泄露 |

**自检清单**：

- [ ] 我能说出哪些角色是累积的、为什么给 PM 加 Planner 而不是 Reporter、给审计部门加 Auditor 而不是 Maintainer？
- [ ] 给我一个 `glpat-` 开头的令牌，我知道**为什么不能靠前缀判断类型**，以及用哪个 API 反查它的 scope/角色/最后使用时间（以及"为什么刚调用完看不到使用记录更新"）？
- [ ] 我能解释"令牌 active 但请求 404"最可能是哪种配置，以及怎么证实？
- [ ] 我能说出 `Allowed to push and merge` 与 `Allowed to merge` 的差异、通配规则冲突时哪条生效、官方为什么建议 push-and-merge 集合更宽？
- [ ] 我知道改全局推送规则后为什么"老项目一个都没被管住"，并给出正确的下发与验证方法？
- [ ] 我能说出开启"要求签名提交"后仍然出现未签名提交的原因，以及 Web IDE 为什么提交不了？
- [ ] 同事离职，我能按顺序说出 6 步回收动作，并解释为什么第 3 步（bot 令牌）最容易被漏、为什么不能直接 Delete？
- [ ] 遇到密钥泄露，我第一步做什么、为什么不能先去做历史清理？

> 权限与安全讲完，下一篇转到管理员日常：[第 13 篇 实例管理与升级](13_administration.md) —— 备份恢复、升级路径、容量与高可用。
