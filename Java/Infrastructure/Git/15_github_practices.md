
# 1. GitHub 核心功能

## 1.1 平台架构

GitHub 是全球最大的代码托管平台，基于 Git，提供了代码托管、协作、CI/CD、项目管理等完整功能。

| 功能模块 | 说明 | 适用场景 |
|---------|------|---------|
| Repository | 代码仓库 | 存储与管理代码 |
| Pull Request | 代码审查 | 团队协作、代码质量控制 |
| Issues | 问题跟踪 | Bug 报告、功能请求 |
| Actions | CI/CD | 自动化构建、测试、部署 |
| Projects | 项目管理 | Sprint 规划、看板管理 |
| Wiki | 文档 | 项目文档、知识库 |
| Discussions | 社区讨论 | 开源项目交流 |
| Pages | 静态网站 | 文档站点、个人博客 |
| Security | 安全扫描 | 依赖漏洞检测、密钥扫描 |

## 1.2 仓库配置最佳实践

```yaml
# 仓库设置推荐
General:
  Default branch: main
  Branch protection: enabled

Branch protection rules:
  - Require pull request reviews: 1+
  - Dismiss stale reviews: true
  - Require status checks: true
  - Require branches up-to-date: true
  - Require conversation resolution: true
  - Include administrators: true

Merge button:
  - Allow merge commits: true
  - Allow squash merging: true
  - Allow rebase merging: true
  - Auto-merge: true
```

# 2. Pull Request 工作流

## 2.1 PR 的完整生命周期

```bash
# 1. Fork 或创建分支
git checkout -b feature/new-feature main

# 2. 开发并推送
git add .
git commit -m "feat: add new feature"
git push -u origin feature/new-feature

# 3. 在 GitHub 创建 PR
# 选择 base（目标分支）和 compare（源分支）

# 4. PR 触发 CI
# GitHub Actions 自动运行测试

# 5. 代码审查
# Reviewers 添加评论、建议修改

# 6. 根据反馈修改
git commit -m "fix: address review comments"
git push origin feature/new-feature

# 7. PR 通过，合并
# 方式：Merge commit / Squash / Rebase
```

## 2.2 PR 模板

创建 `.github/PULL_REQUEST_TEMPLATE.md`：

```markdown
## 变更描述

## 关联 Issue
Closes #

## 变更类型
- [ ] feat: 新功能
- [ ] fix: Bug 修复
- [ ] refactor: 重构
- [ ] docs: 文档
- [ ] test: 测试
- [ ] chore: 其他

## 测试验证
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] E2E 测试通过

## 检查清单
- [ ] 代码遵循项目规范
- [ ] 添加了必要的测试
- [ ] 更新了相关文档
- [ ] 没有引入新的安全风险

## 截图（如适用）

## 备注
```

## 2.3 基于 Issue 的 PR 工作流

```yaml
工作流：
1. 用户/产品创建 Issue
2. 开发者认领 Issue
3. 创建分支: feature/<issue-number>-description
4. 提交: "feat: #123 add login form"
5. 创建 PR: "Closes #123"
6. 审查、合并
7. Issue 自动关闭

优点：
- 所有变更可追溯
- PR 与 Issue 关联
- 里程碑管理方便
```

# 3. GitHub Actions

## 3.1 基本概念

GitHub Actions 是 GitHub 内置的 CI/CD 平台，使用 YAML 定义工作流。

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm run test

      - name: Build
        run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
```

## 3.2 工作流高级模式

```yaml
# 矩阵构建
name: Matrix Build
on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        include:
          - os: ubuntu-latest
            node: 20
            coverage: true

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm run test
      - if: matrix.coverage
        run: npm run coverage

# 依赖工作流
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test:
    needs: lint  # 等待 lint 通过
    runs-on: ubuntu-latest
    steps:
      - run: npm run test

  deploy:
    needs: [lint, test]  # 等待 lint 和 test 都通过
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - run: npm run deploy
```

## 3.3 常用 Actions

| Action | 用途 | 示例 |
|--------|------|------|
| `actions/checkout` | 检出代码 | `uses: actions/checkout@v4` |
| `actions/setup-node` | 配置 Node.js | `uses: actions/setup-node@v4` |
| `actions/setup-java` | 配置 Java | `uses: actions/setup-java@v4` |
| `actions/setup-python` | 配置 Python | `uses: actions/setup-python@v5` |
| `actions/cache` | 缓存依赖 | `uses: actions/cache@v4` |
| `actions/upload-artifact` | 上传构建产物 | `uses: actions/upload-artifact@v4` |
| `actions/github-script` | GitHub API 调用 | `uses: actions/github-script@v7` |
| `docker/login-action` | Docker 登录 | `uses: docker/login-action@v3` |
| `docker/build-push-action` | Docker 构建推送 | `uses: docker/build-push-action@v5` |

## 3.4 安全最佳实践

```yaml
# 使用机密变量而非硬编码
steps:
  - name: Deploy
    env:
      DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
    run: |
      echo "$DEPLOY_KEY" | ssh-add -

# 限制权限
permissions:
  contents: read
  issues: write
  pull-requests: read

# 使用 OIDC 代替长期凭证
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE }}
    aws-region: us-east-1
```

# 4. Issues 与项目管理

## 4.1 Issue 模板

创建 `.github/ISSUE_TEMPLATE/` 目录：

```markdown
<!-- bug_report.md -->
---
name: Bug 报告
about: 帮助我们改进产品
title: '[BUG] '
labels: bug
assignees: ''
---

## 描述
清晰描述 bug

## 重现步骤
1. Go to '...'
2. Click on '...'
3. See error

## 预期行为
清晰描述期望的结果

## 环境
- OS: [e.g. Windows 11]
- Browser: [e.g. Chrome 120]
- Version: [e.g. v1.2.0]

## 截图

## 日志
```

## 4.2 Labels 分类

```yaml
类型:
  - bug: 缺陷
  - enhancement: 增强
  - feature: 新功能
  - documentation: 文档

优先级:
  - priority-high: 紧急
  - priority-medium: 中等
  - priority-low: 低优先级

状态:
  - needs-triage: 待分类
  - in-progress: 进行中
  - blocked: 阻塞
  - stale: 已过时

其他:
  - good-first-issue: 适合新手
  - help-wanted: 寻求帮助
  - security: 安全问题
```

## 4.3 Projects（看板）

```yaml
GitHub Projects 使用示例:

列:
  📋 Backlog       # 待办事项
  📝 Ready         # 已规划
  🔨 In Progress   # 开发中
  👀 In Review     # 审查中
  ✅ Done          # 已完成

自动化规则:
  - PR opened → In Review
  - PR merged → Done
  - Issue assigned → In Progress
  - Issue closed → Done
```

# 5. GitHub 安全特性

## 5.1 Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
      - "auto-merge"
    reviewers:
      - "team-maintainers"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "monthly"
```

## 5.2 Secret Scanning

GitHub 自动扫描仓库中的密钥：

```yaml
检测的内容:
  - AWS/GCP/Azure 密钥
  - GitHub Token
  - 私钥 (BEGIN RSA PRIVATE KEY)
  - 数据库连接串
  - API Key

最佳实践:
  - 使用 secrets.GITHUB_TOKEN
  - 环境变量代替配置文件
  - pre-commit hook 检测
  - 定期轮换密钥
```

## 5.3 CodeQL 代码分析

```yaml
# .github/workflows/codeql.yml
name: "CodeQL"

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # 每周日

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript
      - uses: github/codeql-action/analyze@v3
```

# 6. GitHub Pages

## 6.1 部署静态站点

```yaml
# .github/workflows/pages.yml
name: Deploy Pages

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: |
          npm ci
          npm run build

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: './dist'

      - name: Deploy
        uses: actions/deploy-pages@v4
```

## 6.2 项目文档站（使用 MkDocs）

```yaml
name: Deploy MkDocs
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - run: pip install mkdocs-material
      - run: mkdocs gh-deploy --force
```

# 7. 开源协作

## 7.1 贡献指南

创建 `CONTRIBUTING.md`：

```markdown
# 贡献指南

## 工作流
1. Fork 项目
2. 从 main 创建分支: feat/description
3. 开发并提交（遵循 Conventional Commits）
4. 推送到你的 Fork
5. 创建 Pull Request

## 开发规范
- 代码风格：遵循 ESLint 配置
- 提交信息：Conventional Commits
- 测试：新功能必须包含测试
- 文档：公共 API 必须有 JSDoc

## PR 要求
- 关联 Issue
- CI 通过
- 至少 1 人 Review 通过
- 保持提交历史整洁
```

## 7.2 社区文件

```yaml
开源项目必备文件:
  README.md:             项目说明
  LICENSE:               许可证
  CONTRIBUTING.md:       贡献指南
  CODE_OF_CONDUCT.md:    行为准则
  SECURITY.md:           安全策略
  SUPPORT.md:            支持信息
  
  .github/:
    PULL_REQUEST_TEMPLATE.md:  PR 模板
    ISSUE_TEMPLATE/:          Issue 模板
    CODEOWNERS:               代码所有者
    workflows/:              Actions 工作流
```
