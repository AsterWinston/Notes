
# 1. Git Flow

Git Flow 是 2010 年由 Vincent Driessen 提出的经典工作流模型，定义了严格的分支结构。

## 1.1 分支结构

```
main ──── M1 ────────── M2 ──────────── M3 ────────── M4
            \            \              /            /
develop ── D1 ── D2 ──── D3 ── D4 ──── D5 ── D6 ── D7
           \              \      \         \      \
feature ── F1 ── F2 ──── F3      \         \      \
                                  \         \
release ────────── R1 ── R2 ───── R3 ── R4   \
                                                \
hotfix ─────────────────────────────────── H1 ── H2
```

| 分支类型 | 命名规范 | 来源 | 合并到 | 生命周期 |
|---------|---------|------|--------|---------|
| main | main | — | — | 永久 |
| develop | develop | main | — | 永久 |
| feature | feature/* | develop | develop | 功能开发期间 |
| release | release/* | develop | main + develop | 发布准备期间 |
| hotfix | hotfix/* | main | main + develop | 紧急修复期间 |

## 1.2 完整流程

```bash
# 1. 从 develop 创建 feature 分支
git checkout -b feature/user-login develop

# 2. 在 feature 分支上开发
git add .
git commit -m "feat: add login form"
git commit -m "feat: add auth API"

# 3. 开发完成，合并回 develop
git checkout develop
git merge --no-ff feature/user-login

# 4. 准备发布，创建 release 分支
git checkout -b release/v1.2.0 develop

# 5. 在 release 分支上修复小 bug、更新版本号
git commit -m "fix: update version to 1.2.0"
git commit -m "chore: update changelog"

# 6. release 完成，合并到 main 和 develop
git checkout main
git merge --no-ff release/v1.2.0
git tag -a v1.2.0 -m "Release v1.2.0"

git checkout develop
git merge --no-ff release/v1.2.0

# 7. 紧急修复
git checkout -b hotfix/1.2.1 main
git commit -m "fix: critical security fix"
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix v1.2.1"

git checkout develop
git merge --no-ff hotfix/1.2.1
```

## 1.3 优缺点

| 优点 | 缺点 |
|------|------|
| 分支职责清晰，适合大型项目 | 分支过多，流程复杂 |
| 支持并行开发和多个版本 | 学习曲线陡峭 |
| 发布管理严格，适合有版本周期的项目 | 合并操作频繁，历史复杂 |
| hotfix 流程完善 | 不适合持续交付/部署 |

# 2. GitHub Flow

GitHub Flow 是 GitHub 推崇的轻量级工作流，适合持续部署。

## 2.1 核心原则

```
main ──── M1 ───── M2 ───── M3 ───── M4 ───── M5
           \        /          \        /
feature ── F1 ── F2   feature ── F1 ── F2
           ↑              ↑
        PR review       PR review
```

1. main 分支始终可部署
2. 从 main 创建 feature 分支
3. 频繁推送 feature 分支
4. 通过 Pull Request 发起合并
5. 代码审查通过后合并
6. 合并后立即部署

## 2.2 完整流程

```bash
# 1. 从 main 创建分支
git checkout -b feature/user-login main

# 2. 定期推送
git push -u origin feature/user-login

# 3. 在 GitHub 创建 Pull Request

# 4. 团队审查 PR，提出修改意见

# 5. 根据审查修改
git commit -m "fix: address review comments"
git push origin feature/user-login

# 6. PR 通过，合并到 main（在 GitHub 网页上完成）
# 或命令行合并
git checkout main
git merge --no-ff feature/user-login
git push origin main

# 7. 部署（自动触发）

# 8. 删除 feature 分支
git branch -d feature/user-login
git push origin --delete feature/user-login
```

## 2.3 优缺点

| 优点 | 缺点 |
|------|------|
| 简单，学习成本低 | 没有 develop 分支，main 即开发线 |
| PR 审查集成好 | 不适合多版本并行维护 |
| CI/CD 集成紧密 | 不适合大型复杂项目 |
| 适合持续部署 | release 管理不够灵活 |

# 3. GitLab Flow

GitLab Flow 结合了 Git Flow 的严格性和 GitHub Flow 的简洁性，增加了环境分支。

## 3.1 环境分支模型

```
main ── M1 ── M2 ── M3 ── M4 ── M5
         \           \      \
pre-prod ── P1 ── P2 ─ P3 ── P4
              \           \
production ── R1 ──────── R2
```

## 3.2 完整流程

```bash
# 1. 从 main 创建 feature 分支
git checkout -b feature/user-login main

# 2. 开发并提交
git commit -m "feat: add user login"

# 3. 推送并创建 Merge Request
git push -u origin feature/user-login

# 4. 通过 MR 合并到 main
# 5. 部署到预发布环境
git checkout pre-prod
git merge main

# 6. 验证通过后部署到生产
git checkout production
git merge pre-prod
git tag -a v1.2.0 -m "Release v1.2.0"
```

## 3.3 优缺点

| 优点 | 缺点 |
|------|------|
| 环境分支持清晰 | 环境分支多了也增加复杂度 |
| 支持持续交付 | 不如 GitHub Flow 简单 |
| 适合有多个环境的大项目 | 需要 CI/CD 工具配合 |
| 与 GitLab CI/CD 无缝集成 | — |

# 4. Trunk-Based Development

主干开发模式，所有开发都在 main（trunk）或短期分支上进行。

## 4.1 核心原则

```
main ── M1 ── M2 ── M3 ── M4 ── M5 ── M6 ── M7
          \    /      \    /      \
          F1  F2      F3  F4      F5  F6
         (1天)       (1天)       (1天)
```

- 分支生命周期不超过 1 天
- 频繁合并到 main（每天至少一次）
- 使用功能开关（Feature Toggle）控制未完成的功能
- 强调小步提交、持续集成

## 4.2 完整流程

```bash
# 1. 从 main 创建短期分支
git checkout -b feature/add-button main

# 2. 半天内完成开发
git commit -m "feat: add button component"

# 3. 立即合并回 main（通过 PR 或直接提交）
git checkout main
git pull --rebase origin main
git merge feature/add-button
git push origin main

# 4. 使用功能开关控制可见性
if featureToggle('new-button'):
  showNewButton()
else:
  showOldButton()
```

## 4.3 优缺点

| 优点 | 缺点 |
|------|------|
| 最简单的工作流 | 需要高质量自动化测试 |
| 减少合并冲突 | 必须使用功能开关 |
| CI 反馈快速 | 不适合大团队并行开发 |
| 适合 DevOps 文化 | 需要严格纪律 |

# 5. 工作流对比表格

| 对比维度 | Git Flow | GitHub Flow | GitLab Flow | Trunk-Based |
|---------|----------|-------------|-------------|-------------|
| 复杂度 | 高 | 低 | 中 | 最低 |
| 学习成本 | 高 | 低 | 中 | 低 |
| 长期分支 | 多个 | 无 | 环境分支 | 无 |
| PR/MR | 可选 | 必须 | 推荐 | 可选 |
| 持续部署 | 困难 | 适合 | 适合 | 最适合 |
| 多版本维护 | 适合 | 不适合 | 可支持 | 不适合 |
| 大团队 | 适合 | 适合 | 适合 | 适合（加功能开关） |
| 小团队 | 过于复杂 | 最适合 | 适合 | 适合 |

# 6. 选择建议

## 6.1 按项目类型推荐

```yaml
# 有固定版本周期的应用
recommended: Git Flow
理由: 明确的发布流程，支持多版本维护

# SaaS / Web 应用（持续部署）
recommended: GitHub Flow
理由: 简单，快速部署，PR 审查

# 企业级应用（多环境）
recommended: GitLab Flow
理由: 环境分支清晰，合规性好

# 微服务 / DevOps 团队
recommended: Trunk-Based
理由: 快速迭代，功能开关控制
```

## 6.2 混合实践

实际项目中经常混合使用多种工作流：

```yaml
# 示例：每月发布 + 持续部署混合
基础: Git Flow（版本管理）
日常开发: GitHub Flow（PR 审查）
热修复: Trunk-Based（直接提交）
环境管理: GitLab Flow（pre-prod/production）
```

# 7. 功能开关（Feature Toggle）

配合 Trunk-Based 使用，控制未完成功能的可见性：

```java
// 配置中心控制
if (featureManager.isEnabled("new-login")) {
    return new LoginPage();
} else {
    return oldLoginPage();
}
```

```yaml
# feature-toggle.yml
features:
  new-login:
    enabled: true      # 开发环境
    # enabled: false   # 生产环境
  new-payment:
    enabled: false     # 尚未完成
```

```bash
# 功能开发完成后移除开关
git commit -m "feat: remove old login and feature toggle"
```
