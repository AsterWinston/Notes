
# 1. 什么是 Git Hooks

Git Hooks 是 Git 在特定事件发生时自动执行的脚本。它们存储在 `.git/hooks/` 目录下，用于在提交、推送、合并等操作前后执行自定义逻辑。

## 1.1 Hook 的位置与生命周期

```
.git/hooks/
├── applypatch-msg.sample       # git am 应用补丁时
├── pre-applypatch.sample       # git am 应用补丁后（提交前）
├── post-applypatch.sample      # git am 提交后
├── pre-commit.sample           # git commit 前
├── pre-merge-commit.sample     # git merge 提交前
├── prepare-commit-msg.sample   # 准备提交信息时
├── commit-msg.sample           # 提交信息编辑后
├── post-commit.sample          # git commit 后
├── pre-rebase.sample           # git rebase 前
├── post-checkout.sample        # git checkout/switch 后
├── post-merge.sample           # git merge 后
├── pre-push.sample             # git push 前
├── pre-receive.sample          # 服务端：收到推送时
├── update.sample               # 服务端：更新分支时
├── post-receive.sample         # 服务端：接收推送后
├── post-update.sample          # 服务端：更新引用后
├── push-to-checkout.sample     # 服务端：push-to-checkout 时
├── pre-auto-gc.sample          # git gc --auto 前
├── post-rewrite.sample         # git rebase/commit --amend 后
├── sendemail-validate.sample   # git send-email 前
└── fsmonitor-watchman.sample   # 文件系统监控
```

## 1.2 Hook 的工作原理

```bash
# 启用一个 hook：去掉 .sample 后缀并设为可执行
mv .git/hooks/pre-commit.sample .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# Hook 脚本必须返回 0（退出码）
# 返回非 0 则终止 Git 操作

#!/bin/sh
# pre-commit hook 示例
echo "Running pre-commit checks..."
# 如果检查失败
exit 1  # 阻止提交
exit 0  # 允许提交
```

# 2. 客户端 Hooks 详解

## 2.1 pre-commit — 提交前检查

在输入提交信息之前运行，用于检查代码质量：

```bash
#!/bin/sh
# .git/hooks/pre-commit

# 检查是否有残留的调试代码
if git diff --cached | grep -E '(console\.log|debugger|TODO)' > /dev/null; then
  echo "ERROR: Found debug code (console.log/debugger/TODO) in staged files"
  exit 1
fi

# 检查是否有冲突标记
if git diff --cached | grep -E '<<<<<<<|=======|>>>>>>>' > /dev/null; then
  echo "ERROR: Found merge conflict markers in staged files"
  exit 1
fi

# 检查文件大小
MAX_SIZE=1048576  # 1MB
for file in $(git diff --cached --name-only); do
  if [ -f "$file" ] && [ $(stat -f%z "$file") -gt $MAX_SIZE ]; then
    echo "ERROR: $file exceeds 1MB"
    exit 1
  fi
done

exit 0
```

## 2.2 prepare-commit-msg — 提交信息预处理

在提交信息编辑器打开前运行，可用于自动生成提交信息模板：

```bash
#!/bin/sh
# .git/hooks/prepare-commit-msg
# 参数：$1=提交信息文件 $2=提交来源 $3=SHA（如果是 amend）

# 自动添加分支名
BRANCH_NAME=$(git symbolic-ref --short HEAD 2>/dev/null)
echo "[$BRANCH_NAME] $(cat $1)" > $1

# 如果是 merge commit，自动生成信息
if [ "$2" == "merge" ]; then
  echo "Merge branch '$3'" > $1
fi
```

## 2.3 commit-msg — 验证提交信息

在用户输入提交信息后运行，用于校验格式：

```bash
#!/bin/sh
# .git/hooks/commit-msg
# 检查 Conventional Commits 格式

COMMIT_MSG=$(cat $1)

# 允许的格式
# feat: ...
# fix: ...
# docs: ...
# style: ...
# refactor: ...
# test: ...
# chore: ...
# perf: ...

PATTERN="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,}$"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo "ERROR: Commit message must follow Conventional Commits format"
  echo "Examples:"
  echo "  feat: add new feature"
  echo "  fix: resolve login bug"
  echo "  docs: update API documentation"
  echo "  refactor: extract validation logic"
  exit 1
fi

# 检查行长度（第一行不超过 72 字符）
HEADER=$(head -n1 "$1")
if [ ${#HEADER} -gt 72 ]; then
  echo "ERROR: First line of commit message exceeds 72 characters"
  exit 1
fi
```

## 2.4 post-commit — 提交后操作

```bash
#!/bin/sh
# .git/hooks/post-commit

# 通知 CI 系统
COMMIT_HASH=$(git rev-parse HEAD)
curl -X POST https://ci.example.com/webhook \
  -H "Content-Type: application/json" \
  -d "{\"commit\": \"$COMMIT_HASH\", \"repo\": \"$(basename $(pwd))\"}"

# 更新本地 changelog
git log --oneline --no-decorate -1 >> .changelog
```

## 2.5 pre-push — 推送前检查

```bash
#!/bin/sh
# .git/hooks/pre-push
# 参数：$1=远程名称 $2=远程 URL

# 检查是否有未推送的子模块
if git submodule status | grep -q '^+'; then
  echo "ERROR: Submodule changes not pushed"
  exit 1
fi

# 检查是否有巨大的文件
MAX_SIZE=$((100 * 1024 * 1024))  # 100MB
while read local_ref local_sha remote_ref remote_sha; do
  for obj in $(git rev-list --objects $local_sha ^$remote_sha 2>/dev/null); do
    size=$(git cat-file -s $obj 2>/dev/null)
    if [ "$size" -gt "$MAX_SIZE" ]; then
      echo "ERROR: File $obj exceeds 100MB"
      exit 1
    fi
  done
done

exit 0
```

# 3. 服务端 Hooks

## 3.1 pre-receive — 推送前验证

服务端在收到推送后、更新引用前执行：

```bash
#!/bin/bash
# hooks/pre-receive

# 读取标准输入（每行格式：<旧值> <新值> <引用名>）
while read oldrev newrev refname; do
  # 禁止删除分支
  if [ "$newrev" = "0000000000000000000000000000000000000000" ]; then
    echo "ERROR: Branch deletion is not allowed"
    exit 1
  fi

  # 检查提交信息格式
  for commit in $(git rev-list $oldrev..$newrev); do
    MSG=$(git log --format=%s -1 $commit)
    if ! echo "$MSG" | grep -qE "^(feat|fix|docs|style|refactor|test|chore|perf):"; then
      echo "ERROR: Commit $commit has invalid message format"
      exit 1
    fi
  done
done
exit 0
```

## 3.2 update — 逐分支检查

与 pre-receive 类似，但为每个更新的分支调用一次：

```bash
#!/bin/bash
# hooks/update
# 参数：$1=引用名 $2=旧值 $3=新值

REFNAME=$1
OLDREV=$2
NEWREV=$3

# 保护 main 分支
if [ "$REFNAME" = "refs/heads/main" ]; then
  # 禁止强制推送
  if [ "$(git merge-base $OLDREV $NEWREV)" != "$OLDREV" ]; then
    echo "ERROR: Force push to main is not allowed"
    exit 1
  fi

  # 要求签名提交
  if ! git verify-commit $NEWREV &>/dev/null; then
    echo "ERROR: Commits to main must be signed"
    exit 1
  fi
fi
exit 0
```

## 3.3 post-receive — 推送后触发

推送完成后执行，常用于 CI/CD 触发、部署等：

```bash
#!/bin/bash
# hooks/post-receive

while read oldrev newrev refname; do
  # 自动部署
  if [ "$refname" = "refs/heads/main" ]; then
    git --work-tree=/var/www/project checkout -f main
    systemctl reload nginx
    echo "Deployed main to production"
  fi

  # 触发 CI
  curl -X POST https://jenkins.example.com/job/deploy/build \
    --user $JENKINS_TOKEN

  # 发送通知
  curl -X POST https://hooks.slack.com/services/xxx/yyy/zzz \
    -H "Content-Type: application/json" \
    -d '{"text":"New deployment to production"}'
done
```

# 4. 使用工具管理 Hooks

## 4.1 Husky（Node.js 项目）

```bash
# 安装 husky
npm install husky --save-dev
# 或
yarn add husky --dev

# 初始化
npx husky init

# 创建 hook
npx husky add .husky/pre-commit "npm run lint"
npx husky add .husky/commit-msg "npx --no -- commitlint --edit \$1"
npx husky add .husky/pre-push "npm run test"
```

`.husky/pre-commit` 示例：

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint-staged
```

## 4.2 pre-commit 框架（多语言通用）

```bash
# 安装 pre-commit 框架
pip install pre-commit

# 创建配置文件 .pre-commit-config.yaml
cat > .pre-commit-config.yaml << EOF
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key

  - repo: https://github.com/psf/black
    rev: 24.2.0
    hooks:
      - id: black

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
EOF

# 安装 hooks
pre-commit install

# 手动运行
pre-commit run --all-files
```

## 4.3 commitlint（提交信息校验）

```bash
# 安装
npm install -D @commitlint/cli @commitlint/config-conventional

# 配置文件 commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'test', 'chore', 'perf', 'ci', 'build', 'revert'
    ]],
    'subject-case': [0],
    'subject-max-length': [2, 'always', 72]
  }
};

# 与 husky 集成
npx husky add .husky/commit-msg "npx --no -- commitlint --edit \$1"
```

## 4.4 lint-staged（只检查暂存的文件）

```bash
# 安装
npm install -D lint-staged

# package.json 配置
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yaml}": ["prettier --write"],
    "*.java": ["java -jar checkstyle.jar"]
  }
}

# .husky/pre-commit
npx lint-staged
```

# 5. Hook 的团队共享

## 5.1 Hook 存储与分发

Hooks 位于 `.git/hooks/`（不受版本控制），团队共享需要额外处理：

```bash
# 方案 1：将 hooks 目录版本化
mkdir -p .githooks
# 将 hook 脚本放入 .githooks/
# 配置 Git 使用自定义 hooks 目录
git config core.hooksPath .githooks

# 方案 2：使用脚本初始化 hooks
```

**方案 1 的完整示例**：

```bash
# 项目仓库中
my-project/
├── .githooks/
│   ├── pre-commit
│   ├── commit-msg
│   └── pre-push
├── .gitignore           # 不忽略 .githooks/
└── setup.sh             # 初始化脚本
```

```bash
# setup.sh — 项目初始化时运行
#!/bin/bash
git config core.hooksPath .githooks
echo "Git hooks configured successfully"
```

## 5.2 跳过 Hook

```bash
# 临时跳过 hook
git commit --no-verify -m "WIP: skip hooks"
git push --no-verify

# 在 CI 环境中自动跳过
export GIT_HOOKS_SKIP=true  # 自定义环境变量

# 设置 Git 变量跳过某些检查
git -c core.hooksPath=/dev/null commit
```

# 6. Hook 的开发调试

```bash
# 查看 hook 的执行日志
GIT_TRACE=1 git commit -m "test"

# 获取传递给 hook 的参数（在 hook 脚本中）
echo "Arguments: $@"
echo "Stdin: $(cat)"

# Hook 的环境变量
echo "GIT_DIR: $GIT_DIR"
echo "GIT_OBJECT_DIRECTORY: $GIT_OBJECT_DIRECTORY"
echo "GIT_INDEX_FILE: $GIT_INDEX_FILE"
echo "GIT_WORK_TREE: $GIT_WORK_TREE"
```

# 7. 自动化工作流实践

## 7.1 完整的 pre-commit 工作流

```bash
# .githooks/pre-commit
#!/bin/bash
set -e

echo "=== Pre-commit checks ==="

# 1. 检查暂存区是否有调试代码
STAGED=$(git diff --cached --name-only --diff-filter=ACM)
if [ -n "$STAGED" ]; then
  # 2. 运行 linter
  npm run lint -- --quiet || exit 1

  # 3. 运行格式化
  npm run format -- --check || exit 1

  # 4. 运行单元测试（仅对修改的文件）
  npm run test -- --changed || exit 1

  # 5. 检查安全漏洞
  npm audit --production --audit-level=high || exit 1
fi

echo "All checks passed ✓"
```

## 7.2 自动生成 CHANGELOG

```bash
# .githooks/post-checkout
#!/bin/bash
# 在新分支 checkout 后自动安装依赖

# 获取新旧分支名
PREV_HEAD=$1
NEW_HEAD=$2
IS_BRANCH_CHECKOUT=$3

if [ "$IS_BRANCH_CHECKOUT" = "1" ]; then
  BRANCH=$(git symbolic-ref --short HEAD 2>/dev/null)

  # 检查是否有 package.json 变化
  if git diff $PREV_HEAD..$NEW_HEAD --name-only | grep -q 'package.json'; then
    echo "Detected package.json changes, running npm install..."
    npm install
  fi
fi
```
