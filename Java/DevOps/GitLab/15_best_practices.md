# 1. 本篇定位：把前 14 篇拼成一份落地手册

单点功能都讲过了，真实团队引入 GitLab 时的难点是**顺序与取舍**：先定什么、什么值得模板化、哪些设置一旦定下就难改。本篇给一套可直接照抄的落地清单。

# 2. 第一步：组织结构（改起来最贵，最先定）

| 决策 | 推荐 | 理由 |
|------|------|------|
| Group 划分 | 按产品线/业务域，嵌套 ≤3 层 | 权限、模板、安全策略全部沿群组继承（第 1 篇/第 13 篇） |
| 平台团队 | 独立 `platform` 群组：ci-templates 仓库、基础镜像仓库、工具链 | CI 模板与基镜像需要"版本化的内部供应商" |
| 项目粒度 | 一个服务/库一个项目；Monorepo 单独评估 | 项目级流水线、权限、制品边界最自然 |
| Monorepo vs Polyrepo | 跨团队协作强、原子变更多 → Monorepo（配父子流水线/changes 规则）；否则 Polyrepo | 第 7 篇的 rules+父子流水线是 Monorepo 可行性的前提 |

# 3. 第二步：统一仓库规范（用模板仓库固化）

建一个 `platform/repo-defaults` 仓库，每个新项目落地这套文件：

```text
project/
├── .gitlab-ci.yml              # include 平台模板，只留本项目差异
├── .gitlab/
│   ├── issue_templates/        # Bug/Feature/Incident 表单（第 11 篇）
│   └── merge_request_templates/Default.md   # MR 描述模板
├── CODEOWNERS                  # 路径→负责人（第 4 篇）
├── .gitlab/CODEOWNERS          # （或此路径）
├── renovate.json               # 依赖自动更新
└── docs/                       # 设计文档随代码评审
```

CI 模板的标准形态（第 6/7 篇的组合拳）：

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

# 5. 第四步：CI 工程的团队习惯

| 实践 | 收益 |
|------|------|
| `.gitlab-ci.yml` 变更走 MR 评审，像代码一样对待 | 流水线即产品 |
| 镜像化构建环境（固定 `image:` 版本，禁 `latest`） | 可复现、不被上游炸 |
| Maven 缓存 + junit 报告 + `interruptible` | 快、可诊断、省资源（第 6/8 篇） |
| 流水线阶段目标 ≤10 分钟（超时拆并行/按需 changes） | MR 反馈速度决定合并频率 |
| 生产部署只由 Tag/manual 触发，密钥全走 Protected 变量 | 可审计、防误发（第 9/12 篇） |
| 定期跑 pipeline efficiency 报表，清理僵尸 job | CI 成本可控（第 14 篇） |

# 6. 第五步：安全与合规基线

- 实例层：关注册、强制 2FA、SSO、Webhook 出站白名单（第 12 篇清单）。
- 项目层：默认 Private；Secret Detection + 依赖扫描全量开，高危阻断合并。
- 供应链：镜像只由 CI 产出推 Registry；发布 Tag 镜像打不可变版本并签名（cosign），Registry 清理策略豁免发布 tag（第 10 篇）。
- 备份：每日全备 + 异地对象存储 + 季度恢复演练（第 13 篇）。

# 7. 分角色速查

| 角色 | 必读篇 |
|------|--------|
| 普通开发 | 4、5、6 |
| TL / 架构 | 4、5、7、11 |
| DevOps / SRE | 2、3、8、9、13、14、16 |
| 安全/合规 | 12、10 |
| 管理者 | 1、14（DORA）、15 全篇 |

# 8. 常见反模式

| 反模式 | 后果 | 纠正 |
|--------|------|------|
| 把 GitLab 当纯代码托管，CI 留在 Jenkins | 两套门禁、双份维护 | 新仓库默认 GitLab CI，存量逐步迁 |
| 每个仓库手抄一份 yml | 规范漂移，无人负责 | 模板仓库 + include + 版本锁定 |
| main 不设保护、人人可推 | 事故源头 | 第 4 步基线 |
| CI 用个人 PAT/长期 token 推制品 | 离职即断、权限漂移 | Job/Deploy Token |
| 镜像只打 `latest` | 无法回滚 | 短 SHA + 版本 tag + 豁免清理 |
| DORA 指标做 KPI | 指标造假开始 | 趋势自用，不入考核 |

# 9. 小结

| 阶段 | 关键动作 |
|------|---------|
| 定结构 | 群组层级与平台群组，一次定好 |
| 立规范 | 模板仓库固化 CI/Issue/MR/CODEOWNERS |
| 设门禁 | 分支保护 + MR 规则 + CI 必须绿 |
| 养习惯 | 流水线如代码、镜像化环境、快速反馈 |
| 兜底线 | 扫描左移、令牌最小化、备份演练 |

规范篇收束"怎么用好"；下一篇处理"用坏了怎么办"——常见故障排查手册。
