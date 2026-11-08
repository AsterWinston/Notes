# 1. 权限模型：角色 × 范围 × 继承

GitLab 权限的核心公式：**用户在某个作用域（Instance/Group/Project）上被授予一个角色，角色决定它能做什么；高作用域的授权自动向下继承**。

| 角色 | Project 能力摘要 | Group 能力摘要 |
|------|-----------------|---------------|
| Guest | 看公开 Issue、不能看私有代码 | 同左 |
| Reporter | 看代码、建 Issue、下载制品、创建 Issue board | 继承到子群组/项目 |
| **Developer** | push 非保护分支、建 MR、跑 CI、推镜像 | 开发主力角色 |
| **Maintainer** | 合并 MR、管理项目设置/成员/保护分支、删制品 | 管理子群组与项目 |
| Owner | 删除/转移/归档、群组所有治理 | 群组最高权限 |
| Minimal Access (17.x) | 只见被明确分享的内容 | 大型组织"只给必要可见性" |

**设计考量**：

- **角色是"包"，不是"点"**：GitLab 故意不做细到按钮的 ACL（与 Jira 的思路相反），换来的是可推理、可审计。个别点状需求（如"不让 Developer 合并"）靠**保护分支 + 审批规则**实现，不要期待拆角色。
- **继承是双刃剑**：群组 Owner 默认对全部子群组/项目 Owner。给"部门经理"加 Owner 前想清楚他要的是治理还是可见性，可用 **Group Owner ≠ Project Maintainer** 的分层避免权限爆炸。
- 项目成员**直接授权优先于继承**（可单独提升/限制），但限制不能超过 Owner 管理规则。

# 2. 令牌体系：别把 PAT 当万能钥匙

GitLab 有多种令牌，用错类型是最常见的安全隐患：

| 令牌 | 作用域 | 生命周期 | 典型用途 |
|------|--------|---------|---------|
| **Personal Access Token（PAT）** | 该用户全部权限 | 长期（强制设过期时间） | CLI `glab`、脚本；**尽量不用** |
| **Project Access Token** | 单项目 | 长期可过期 | 跨系统集成（以项目身份） |
| **Group Access Token** | 群组 | 长期可过期 | 群组级自动化 |
| **Deploy Token** | 仓库/Registry 只读（可配写） | 长期 | 服务器拉镜像、只读部署 |
| **CI Job Token** | 当前 job 的项目内，有限跨项目 | 一次 job | CI 内推拉制品、触发下游 |
| **OAuth Token** | 应用授权 | 可刷新 | 第三方集成应用 |
| **Runner/Agent Token** | Runner/Agent 注册 | 按对象 | CI 执行器接入 |

> **行业实践**：**禁止用个人 PAT 做自动化**——人离职/改密集成全断，且权限随人漂移。系统集成一律 Project/Group Access Token（17.x 起它们可绑定服务账号身份），运维拉取用 Deploy Token，CI 内用 Job Token。

# 3. 安全扫描：把检查嵌进 MR 而不是事后补

GitLab 的安全能力按 SDLC 位置分层，Free 层就包含大部分扫描器（跑在 CI 里，结果回到 MR/安全中心）：

| 扫描器 | 检测对象 | 层级 | 免费层 |
|--------|---------|------|--------|
| SAST | 源代码漏洞（Semgrep/CodeQL/SpotBugs 等） | 编码期 | ✔ |
| Secret Detection | 硬编码密钥、token | 编码期 | ✔ |
| Dependency Scanning | 三方依赖 CVE（osv-maven/npm audit） | 依赖期 | ✔ |
| License Scanning | 依赖许可证风险 | 依赖期 | Premium |
| DAST | 运行中应用的 Web 漏洞（ZAP） | 运行期 | Ultimate |
| DAST API / API Fuzzing | 接口层 | 运行期 | Ultimate |
| Container Scanning | 基础镜像 CVE（Trivy） | 构建期 | ✔ |
| IaC Scanning | Terraform/K8s 清单错误配置 | 构建期 | Premium/Ultimate |
| Coverage 门禁 | 单测覆盖率审计 | 测试期 | Ultimate |

**最小落地方式**（include 内置模板即可）：

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml

# 关键一步：让高危问题阻断合并
sast:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes: ["**/*.java"]
```

**安全中心（Security Dashboard，Ultimate）**聚合所有项目的漏洞列表，支持指派、SLA（"高危 7 天必修"）、批量标记已处理；MR 页面出现"Security 报告"标签页，评审者当场看到本次改动引入的漏洞。

# 4. 分支与合并的安全约束汇总

权限篇第 4 节讲过配置，这里从"防什么"角度再串一遍：

| 风险 | 控制手段 |
|------|---------|
| 绕过评审直推主干 | 保护分支 + "不允许 force push" |
| 带病合并 | Status checks（流水线必须绿）+ 审批规则 + 讨论必须 resolve |
| 审批失效 | Reset approvals on push（新 commit 清空批准） |
| 自批自合 | 审批规则"禁止作者批准" |
| 提交身份不可信 | Push Rules（reject unsigned）+ GPG/SSH 证书 |
| 并发合并踩坏主干 | 半包含/Merge Train |
| 密钥进仓库 | Secret Detection + Push Rules rejectsecrets |

# 5. 账号与实例级安全基线

自建实例上线前的安全清单（Admin → Settings）：

- **强制 2FA**（Premium）：全局要求，宽限期后未开启者踢出。
- **登录防护**：IP 允许列表、异常登录检测（Ultimate）、会话有效期。
- **关闭注册** + 预创建账号/LDAP/OIDC/SSO 接入（SAML、OAuth2、OIDC）。
- **出站限制**：Webhook 与 CI 的出网目标白名单（防 SSRF 把内网 GitLab 当跳板）。
- **审计事件**（Premium）：导出审计流到 SIEM；`Audit events` 页查"谁改了保护分支"。
- **仓库防爆**：限制 Fork、限制下载附件/归档、禁公开项目（Internal 兜底）。

> **行业实践**：**LDAP/AD + SSO 统一身份**是自建 GitLab 标配；本地账号只留 break-glass root。离职流程里"GitLab 账号停用"必须纳入 checklist——GitLab 的权限跟着人走，人不清账号就全留。

# 6. 合规能力（Ultimate）

面向强合规组织：

- **Compliance Frameworks**：把"某群组必须：强制签名 + 二级审批 + 保留策略 + 扫描全开"打包成框架，项目创建即继承。
- **外部审计（External Auditors）**：给审计员只读、限时、按项目授权。
- **受保护环境变量/密钥**：CI 变量 `Protected` 只对保护分支暴露；密钥管理（Secrets Manager 17.x）对接外部 Vault 替代裸变量。

# 7. 供应链安全：从 MR 到镜像的一根线

```text
commit(签名) ──► MR(SAST/密钥/依赖扫描) ──► CI 构建(基础镜像扫描+SBOM)
     │                                          │
  审计日志 ◄── 安全中心(漏洞管理/SLA) ◄── 制品仓库(Signature + attestation)
                                               │
                                        部署(准入验签, 见 K8s 篇)
```

- **SBOM**（ CycloneDX，Premium）：构建时自动产出软件物料清单，挂到 Release/漏洞页。
- **镜像签名与验签**：GitLab 集成 Sigstore/cosign，CI 用 keyless 签名（Fulcio+Rekor），部署侧准入控制器验签——"不是从 GitLab 流水线出来的镜像不许跑"。
- **漏洞管理 SLA**：安全中心按严重级自动倒计时，超期未修阻断/升级通知。

# 8. 小结

| 要点 | 一句话 |
|------|--------|
| 权限公式 | 角色 × 作用域 + 继承；粒度靠"角色包 + 保护分支/审批"组合 |
| 令牌 | 自动化禁用 PAT，按场景选 Project/Group/Deploy/Job Token |
| 扫描左移 | SAST/Secret/依赖/容器扫描 Free 就有，MR 阻断要显式配 rules |
| 治理 | 2FA/SSO/出站限制是自建实例的及格线 |
| 合规 | 框架下发、审计导出、外部审计员（Ultimate） |
| 供应链 | 签名 → 扫描 → SBOM → 验签部署，一根线闭环 |

权限与安全讲完，下一篇从管理员视角看日常运维：备份恢复、升级、高可用与群组治理。
