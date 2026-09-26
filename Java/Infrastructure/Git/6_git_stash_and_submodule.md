
# 1. Git Stash — 临时存储

## 1.1 基本概念

`git stash` 用于**临时保存工作目录和暂存区的修改**，让你可以切换分支或执行其他操作，之后再恢复。

```
stash 前：
Working Tree: 修改了 file.txt
Index: 暂存了 login.js
HEAD: 指向 commit C

git stash 后：
Working Tree: 干净（恢复到 commit C 的状态）
Index: 干净
Stash Stack: [stash@{0}: 保存了 file.txt 和 login.js 的修改]

git stash pop 后：
Working Tree: 恢复修改
Index: 恢复暂存
Stash Stack: []（已弹出）
```

## 1.2 基本操作

```bash
# 保存当前修改（暂存区和工作目录）
git stash

# 保存时包含未跟踪的文件
git stash -u
git stash --include-untracked

# 保存时包含所有文件（包括忽略的）
git stash -a
git stash --all

# 带消息的保存
git stash push -m "WIP: login form validation"

# 查看 stash 列表
git stash list
# stash@{0}: On feature/login: WIP: login form validation
# stash@{1}: On main: fix: temp save

# 查看某个 stash 的内容
git stash show stash@{1}
git stash show stash@{1} -p  # 显示差异

# 恢复最近的 stash 并移除
git stash pop

# 恢复指定 stash 并移除
git stash pop stash@{2}

# 恢复不移除
git stash apply
git stash apply stash@{2}

# 删除某个 stash
git stash drop stash@{1}

# 清空所有 stash
git stash clear
```

## 1.3 高级用法

**暂存区分离保留**：

```bash
# 默认 stash 会同时保存暂存区和工作目录的修改
# 恢复时暂存区的文件不再暂存（变为 staged → unstaged）

# 保存时保留暂存区状态
git stash push --staged

# 恢复时也恢复暂存区
git stash apply --index
```

**从 stash 创建分支**：

```bash
# 如果 stash 应用时有冲突，可以基于 stash 创建分支
git stash branch new-feature-branch stash@{1}
# 这会：
# 1. 基于 stash 保存时的 commit 创建新分支
# 2. 在新分支上应用 stash
# 3. 删除 stash
```

**部分文件 stash**：

```bash
# 只 stash 工作目录中指定的文件
git stash push -m "only login changes" src/login.js src/auth.js

# 交互式选择要 stash 的 hunk
git stash push -p
```

## 1.4 Stash 的底层原理

Stash 本质上是通过 commit 实现的：

```bash
# stash 创建时，Git 会创建两个 commit：
# 一个是暂存区的状态
# 一个是工作目录的状态
# 然后通过一个特殊的引用 refs/stash 指向它们

cat .git/refs/stash
# abc123def456...

# stash 栈就是 refs/stash 的链表
# stash@{0} = refs/stash
# stash@{1} = refs/stash^1 (第一个 parent)
# stash@{2} = refs/stash^2 (取决于合并提交的结构)
```

# 2. Git Worktree — 多工作目录

## 2.1 基本概念

`git worktree` 允许你同时 checkout 多个分支到不同的目录，在同一个仓库中并行工作。

```bash
# 主要工作区
/Users/me/project (main)

# 新添加的工作区
/Users/me/project-hotfix (hotfix branch)

# 两个工作区共享同一个 .git 目录
/Users/me/project/.git
```

## 2.2 基本操作

```bash
# 添加工作树
git worktree add ../project-hotfix hotfix/urgent-fix

# 基于特定 commit 创建只读工作树
git worktree add --detach ../project-archive abc123

# 列出工作树
git worktree list
# /Users/me/project          abc123 [main]
# /Users/me/project-hotfix   def456 [hotfix/urgent-fix]

# 锁定工作树（防止被误删）
git worktree lock ../project-hotfix

# 解锁
git worktree unlock ../project-hotfix

# 修剪（清理已删除的工作树引用）
git worktree prune

# 删除工作树
git worktree remove ../project-hotfix
```

## 2.3 使用场景

```bash
# 场景：正在 feature 分支开发，但需要紧急修 bug
# 不需要 stash/commit 当前工作

cd /Users/me/project
git worktree add ../hotfix hotfix/security-fix
cd ../hotfix
# 在独立的工作目录中修 bug
git add fix.js
git commit -m "fix: security vulnerability"
cd ../project
git worktree remove ../hotfix

# 场景：并行查看不同分支的代码
git worktree add ../docs-view main
cd ../docs-view
# 可以打开 IDE 查看 main 分支的代码
# 同时主工作区在 feature 分支开发
```

# 3. Git Submodule — 子模块

## 3.1 基本概念

Submodule 允许你在一个 Git 仓库中包含另一个 Git 仓库作为子目录。

```
父仓库 project/
├── .gitmodules          ← 子模块配置
├── src/
│   └── ...
└── libs/
    └── common/          ← 子模块（指向另一个仓库的某个 commit）
        ├── .git         ← 子模块自己的 Git 目录
        ├── README.md
        └── src/
```

## 3.2 添加子模块

```bash
# 添加子模块
git submodule add https://github.com/user/common-lib.git libs/common

# 添加强制使用 HTTPS
git submodule add https://github.com/user/common-lib.git libs/common --force

# 添加到指定分支
git submodule add -b main https://github.com/user/common-lib.git libs/common
```

## 3.3 克隆含子模块的仓库

```bash
# 方式 1：克隆主仓库后初始化子模块
git clone https://github.com/user/project.git
cd project
git submodule init
git submodule update

# 方式 2：克隆时递归初始化子模块
git clone --recurse-submodules https://github.com/user/project.git

# 方式 3：更新到子模块的最新提交
git submodule update --remote

# 方式 4：递归克隆包含嵌套子模块的仓库
git clone --recursive https://github.com/user/project.git
```

## 3.4 子模块日常操作

```bash
# 进入子模块目录操作（像操作独立仓库一样）
cd libs/common
git checkout main
git pull origin main
cd ..
git add libs/common
git commit -m "chore: update common-lib to latest"

# 更新所有子模块到最新
git submodule update --remote --merge

# 更新特定子模块
git submodule update --remote libs/common

# 查看子模块状态
git submodule status
#  abc123def456 libs/common (heads/main)
# +def456abc123 libs/utils (v1.0.0)  # + 表示已初始化
# -789abcdef01 libs/old (xxx)       # - 表示未初始化

# 查看子模块的更新信息
git diff --submodule
```

## 3.5 .gitmodules 文件

```ini
[submodule "libs/common"]
    path = libs/common
    url = https://github.com/user/common-lib.git
    branch = main
    update = rebase  # 更新策略：checkout/rebase/merge/none
```

## 3.6 子模块的最佳实践

```bash
# 在子模块中使用相对路径（Fork 友好）
# .gitmodules 中使用相对 URL
git submodule add ../common-lib.git libs/common

# 子模块指向特定的 commit 而非分支
# 父仓库记录的是子模块某个特定 commit 的 hash
# 这保证了可重现性

# 批量 foreach 命令
git submodule foreach 'git status'
git submodule foreach 'git checkout main && git pull'
git submodule foreach --recursive 'git clean -fd'
```

## 3.7 子模块的陷阱

```bash
# 陷阱 1：子模块在 detached HEAD 状态
# 子模块更新后默认在 detached HEAD 状态
cd libs/common
git checkout main  # 回到分支

# 陷阱 2：忘记推送子模块
# 父仓库指向了子模块的新 commit，但子模块没 push
git push --recurse-submodules=on-demand  # 自动推送子模块

# 陷阱 3：子模块更新后没提交
git diff --submodule  # 检查子模块是否有变更
```

# 4. Git Subtree — 子树合并

## 4.1 基本概念

Subtree 是 Submodule 的替代方案，它将外部仓库的代码**直接合并**到当前仓库的某个子目录中。

```
Submodule 方式：
project/
└── libs/common   ← 指向外部仓库（独立 .git）

Subtree 方式：
project/
├── libs/common/  ← 外部仓库的代码直接存在于当前仓库中
│   ├── src/      ← 可以被 father 仓库直接提交修改
│   └── ...
└── .git（仅一个，包含所有历史）
```

## 4.2 添加子树

```bash
# 添加外部仓库到子目录
git subtree add --prefix=libs/common \
  https://github.com/user/common-lib.git main --squash

# --squash 会将外部仓库的历史压缩为一个提交
# 不压缩则保留完整历史
```

## 4.3 更新子树

```bash
# 拉取外部仓库的最新版本
git subtree pull --prefix=libs/common \
  https://github.com/user/common-lib.git main --squash
```

## 4.4 推送子树修改

```bash
# 将对子目录的修改推送到外部仓库
git subtree push --prefix=libs/common \
  https://github.com/user/common-lib.git new-feature
```

## 4.5 Submodule vs Subtree 对比

| 对比维度 | Submodule | Subtree |
|---------|-----------|---------|
| 代码存储 | 指针引用外部仓库 | 代码直接存在父仓库 |
| 克隆体验 | 需要 `--recurse-submodules` | 正常 clone 即可 |
| 修改外部代码 | 需在子模块内操作 | 可在父仓库直接编辑 |
| 学习复杂度 | 较复杂 | 相对简单 |
| 版本锁定 | 锁定特定 commit | 代码已存在于父仓库 |
| 适用场景 | 大型外部依赖 | 小工具库、内部共享库 |
| 历史复杂度 | 不增加父仓库历史 | 可能增加大量历史 |

**选择建议**：

```bash
# 用 Submodule 的场景：
# - 外部依赖较大且不常修改
# - 需要精确控制版本
# - 多个项目共享同一份依赖代码

# 用 Subtree 的场景：
# - 需要随时修改外部库的代码
# - 简化团队成员的 clone 体验
# - 外部库较小，历史不复杂
```

# 5. 实际应用场景

## 5.1 组合使用 Stash 和 Worktree 优化工作流

```bash
# 场景：并行处理多个功能
# 主工作目录在 feature/login，需要在 feature/payment 上工作

cd ~/project
git worktree add ../project-payment feature/payment

# 两个终端分别打开不同工作目录
# 终端 1：~/project (feature/login)
# 终端 2：~/project-payment (feature/payment)

# 需要临时切换时用 stash
cd ~/project
git stash -u -m "WIP: login validation"
git checkout main
# 处理 main 上的事情
git checkout feature/login
git stash pop
```

## 5.2 Submodule 的典型项目结构

```ini
# .gitmodules
[submodule "libs/orm"]
    path = libs/orm
    url = git@github.com:company/orm-lib.git
    branch = main
[submodule "libs/logging"]
    path = libs/logging
    url = git@github.com:company/logging-lib.git
    branch = main
[submodule "protos"]
    path = protos
    url = git@github.com:company/protobuf-definitions.git
    branch = main
```

```bash
# 初始化项目
git clone --recurse-submodules git@github.com:company/big-project.git
cd big-project

# 同时更新所有子模块
git submodule foreach --recursive 'git checkout main && git pull'
```
