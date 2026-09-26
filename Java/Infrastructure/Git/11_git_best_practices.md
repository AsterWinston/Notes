
# 1. Conventional Commits 规范

## 1.1 提交信息格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

**必填**：`type` 和 `subject`
**可选**：`scope`、`body`、`footer`

## 1.2 Type 类型

| Type | 含义 | 是否影响版本 |
|------|------|-------------|
| `feat` | 新功能 | MINOR |
| `fix` | Bug 修复 | PATCH |
| `docs` | 文档变更 | 否 |
| `style` | 代码格式（不影响功能） | 否 |
| `refactor` | 代码重构 | 否 |
| `perf` | 性能优化 | PATCH |
| `test` | 测试相关 | 否 |
| `chore` | 构建/工具变更 | 否 |
| `ci` | CI 配置变更 | 否 |
| `build` | 构建系统或外部依赖变更 | 否 |
| `revert` | 回退提交 | 否 |

## 1.3 示例

```
feat: add user login functionality

Implement OAuth2-based login with JWT token support.
- Add login form component
- Add auth service integration
- Add token refresh mechanism

Closes #123
BREAKING CHANGE: Removed the deprecated password-based auth
```

```
fix(auth): resolve token expiry race condition

The refresh token was not being invalidated after password change,
causing a brief window where old tokens remained valid.

Fixes #456
```

```
docs: update API documentation

- Add missing endpoint descriptions
- Update request/response examples
- Fix typos in authentication section
```

## 1.4 自动版本推断

基于 Conventional Commits 可以自动推断版本号：

```bash
# 检查提交历史自动生成版本
feat + fix → PATCH
feat       → MINOR
BREAKING  → MAJOR

# 配合 standard-version 使用
npm run release
# 自动分析提交，更新版本号，生成 CHANGELOG，创建 tag
```

# 2. 分支命名规范

## 2.1 推荐格式

```
feature/<issue-id>-<short-description>
bugfix/<issue-id>-<short-description>
hotfix/<issue-id>-<short-description>
release/<version>
chore/<description>
docs/<description>
test/<description>
```

**示例**：

```
feature/123-user-login
bugfix/456-fix-npe-on-empty-input
hotfix/789-security-patch-cve-2024
release/v1.2.0
chore/update-dependencies
docs/add-api-usage-guide
test/add-unit-tests-for-auth
```

## 2.2 不推荐的命名

```
feature/my-new-feature      # 太长，无 Issue 关联
fix                        # 太简短
johns-branch               # 人名相关
asdf1234                   # 无意义
FIX-BUG                    # 大小写不一致
feature/add_login          # 使用了下划线
```

## 2.3 分支清理策略

```bash
# 合并后自动删除本地分支
git config --global --add merge.autoBranchDelete true

# 定期清理本地已合并的分支
alias git-cleanup='git branch --merged | grep -v "\*\|main\|develop" | xargs -n 1 git branch -d'

# 定期清理远端已不存在的跟踪分支
git fetch --prune
```

# 3. 提交规范实践

## 3.1 提交粒度

```
太细：太过琐碎，没有意义
  ✗ "fix typo"
  ✗ "add space"
  ✗ "fix formatting"

太粗：难以追踪问题
  ✗ "implement whole project"
  ✗ "fix many bugs"
  ✗ "various changes"

推荐粒度：每个提交完成一个逻辑单元
  ✓ "feat: add login form validation"
  ✓ "fix: resolve NPE on empty user input"
  ✓ "refactor: extract validation logic to separate module"
```

## 3.2 原子提交原则

每个提交应该：
1. 完成一个独立的功能/修复
2. 可以通过所有测试
3. 包含相关的文档更新
4. 可单独部署

```bash
# 好的做法
git add src/login.js
git commit -m "feat: add login form"
git add src/auth.js
git commit -m "feat: add auth service"

# 不好的做法
git add .
git commit -m "lots of changes"
```

## 3.3 提交信息最佳实践

```
1. 第一行不超过 50 个字符
2. 第一行首字母小写（Conventional Commits 规范）
3. 第一行不加句号
4. 第二行为空行
5. body 每行不超过 72 字符
6. body 说明"为什么改"而非"改了哪里"
```

# 4. Code Review 实践

## 4.1 PR/MR 的大小

```yaml
推荐: 200-400 行变更
最大: 不超过 1000 行

理由:
- 200 行以下：审查速度最快，bug 检出率最高
- 400 行以上：审查效率明显下降
- 1000 行以上：bug 检出率急剧下降
```

## 4.2 PR 描述模板

```markdown
## 变更描述
简要描述本次 PR 的目的和范围

## 关联 Issue
Closes #123

## 变更类型
- [ ] 新功能 (feat)
- [ ] Bug 修复 (fix)
- [ ] 重构 (refactor)
- [ ] 文档更新 (docs)
- [ ] 测试 (test)

## 测试验证
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 手动测试完成

## 截图（如果是 UI 变更）
[Before/After 截图]

## 注意事项
需要关注的潜在影响
```

## 4.3 Review 清单

```yaml
功能正确性:
  - 功能是否按需求实现
  - 边界情况是否处理
  - 错误处理是否完善

代码质量:
  - 是否符合项目编码规范
  - 是否存在重复代码
  - 命名是否清晰
  - 复杂度是否可接受

安全性:
  - 是否存在 SQL 注入风险
  - 是否对用户输入做了校验
  - 敏感信息是否泄露

性能:
  - 是否存在 N+1 查询
  - 是否有不必要的循环
  - 资源是否正确释放

测试:
  - 是否有对应的单元测试
  - 测试覆盖率是否合理
  - 边界情况是否测试
```

# 5. Git 配置最佳实践

## 5.1 推荐全局配置

```bash
# 开发者信息
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# 默认行为
git config --global init.defaultBranch main
git config --global push.default current
git config --global pull.rebase true         # 推荐 rebase 方式 pull
git config --global fetch.prune true         # 自动清理远程跟踪

# 编辑器
git config --global core.editor vim
git config --global core.autocrlf input      # macOS/Linux
git config --global core.autocrlf true       # Windows

# 颜色
git config --global color.ui auto
git config --global color.diff.meta "yellow bold"
git config --global color.status.changed "red"

# 安全
git config --global commit.gpgsign false     # GPG 签名
git config --global credential.helper osxkeychain  # macOS
git config --global credential.helper manager-core  # Windows

# 别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.df diff
git config --global alias.dfs "diff --staged"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "restore --staged"
git config --global alias.wip "commit -am 'WIP'"
```

## 5.2 忽略文件管理

```gitignore
# .gitignore 文件应包含的通用模式
.DS_Store
Thumbs.db
*.swp
*.swo
*~
.vscode/
.idea/
*.iml
.env
.env.local
.env.*.local
*.log
node_modules/
dist/
build/
target/
*.class
*.jar
*.war
```

# 6. 安全最佳实践

## 6.1 敏感信息防护

```bash
# 1. 永远不要提交敏感信息
# 2. 使用环境变量而非配置文件硬编码
# 3. 使用 pre-commit hook 检测

# .pre-commit-config.yaml 中的检测
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

# 4. 如果不慎提交，立即更新并通知团队成员
```

## 6.2 签署验证

```bash
# 配置 GPG 签名
git config --global user.signingkey ABCDEF1234567890
git config --global commit.gpgsign true

# 签署标签
git tag -s v1.0.0 -m "Release v1.0.0"

# 验证
git tag -v v1.0.0
git log --show-signature -1
```

# 7. CI/CD 集成最佳实践

## 7.1 分支策略与 CI 配合

```yaml
# GitHub Actions 示例
on:
  push:
    branches:
      - main          # 主分支：自动部署
      - 'feature/**'  # 功能分支：运行测试
      - 'hotfix/**'   # 热修复：完整 CI
  pull_request:
    branches:
      - main          # PR：必须通过 CI

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      - run: npm run build
```

## 7.2 自动合并检查

```yaml
# 合并前必须满足的条件
# - CI 通过
# - 至少 1 人 Code Review 通过
# - 无合并冲突
# - 分支是最新的（已 rebase main）
# - 提交信息符合规范
```
