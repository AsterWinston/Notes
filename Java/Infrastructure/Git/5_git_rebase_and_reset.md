
# 1. Rebase 原理与操作

## 1.1 什么是 Rebase

Rebase 的核心思想是**将一系列提交从一个基础移动到另一个基础**。与 Merge 不同，Rebase 不会产生 merge commit，而是改写提交历史，使其保持线性。

```
Merge 方式：
main:    A──B──C──────────M
               \          /
feature:        C──D──E──/

Rebase 方式：
main:    A──B──C
feature: A──B──C──D──E  ← 原来基于 B

feature rebase main 后：
main:    A──B──C
feature: A──B──C──D'──E'  ← D' 和 E' 是基于 C 的新提交
```

## 1.2 Rebase 的底层机制

Rebase 不是一个"移动"操作，而是**重放**（replay）：

```
1. 找到共同祖先：B
2. 收集 feature 分支在 B 之上的所有提交：D、E
3. 将 HEAD 指向 main 的最新提交 C
4. 逐个应用 D、E（可能产生冲突，需要解决）
5. 生成新的提交 D'、E'（不同的 hash，不同的 parent）
```

```bash
# 查看 rebase 过程中 Git 的步骤
git rebase main
# Applying: commit D message
# Applying: commit E message
```

## 1.3 基本 Rebase

```bash
# 将当前分支变基到目标分支
git checkout feature
git rebase main

# 一步完成
git rebase main feature

# 交互式变基（最强大的功能）
git rebase -i HEAD~3  # 对最近 3 个提交进行操作

# 交互式变基到指定提交
git rebase -i abc123  # 对 abc123 之后的提交进行操作
```

## 1.4 交互式 Rebase

交互式变基提供了多种操作类型：

```
pick abc123 feat: add login form
pick def456 fix: typo in login
pick 789abc feat: add validation
pick 012def fix: validation bug

# 可用命令：
# p, pick     = 使用该提交
# r, reword   = 使用提交，但修改提交信息
# e, edit     = 使用提交，但停下来修改
# s, squash   = 合并到上一个提交（保留信息）
# f, fixup    = 合并到上一个提交（丢弃信息）
# x, exec     = 运行 shell 命令
# b, break    = 在此处停止（用于编辑）
# d, drop     = 删除该提交
```

**实战场景：整理零碎提交**

```bash
# 假设你有这些零碎的提交：
git log --oneline
# 789abc fix: typo in validation
# 012def fix: validation bug
# 345abc feat: add validation
# 678def fix: login button css
# 901abc feat: add login form

git rebase -i 901abc~1  # 从第一个提交之前开始

# 编辑为：
pick 901abc feat: add login form
fixup 678def fix: login button css    # 合并到上一个
pick 345abc feat: add validation
fixup 012def fix: validation bug      # 合并到上一个
fixup 789abc fix: typo in validation  # 合并到上一个

# 结果：从 5 个提交压缩为 2 个有意义的提交
```

**实战场景：修改历史提交信息**

```bash
git rebase -i HEAD~3
# 将 pick 改为 reword 即可修改对应提交的 message
```

**实战场景：拆分提交**

```bash
git rebase -i HEAD~3
# 将某个提交的 pick 改为 edit
# 保存退出后 Git 会停在该提交处
git reset HEAD~1           # 撤销该提交，保留修改在工作目录
git add src/login.js
git commit -m "feat: add login form"
git add src/validation.js
git commit -m "feat: add validation"
git rebase --continue
```

## 1.5 Rebase 冲突处理

```bash
# Rebase 过程中遇到冲突
git rebase main
# First, rewinding head to replay your work on top of it...
# Applying: feat: add login form
# Applying: feat: add validation
# Applying: feat: add validation rules
# Applying: feat: add tests
# error: Failed to merge in the changes.
# Patch failed at 0003 feat: add validation rules

# 解决冲突后
git add resolved-file.js
git rebase --continue

# 跳过当前提交
git rebase --skip

# 终止 rebase，回到原始状态
git rebase --abort
```

**Rebase 冲突与 Merge 冲突的区别**：

- Merge 冲突：一次性解决所有冲突，产生一个 merge commit
- Rebase 冲突：每个提交都可能产生冲突，需要逐个解决

```bash
# Rebase 的冲突解决策略
git rebase -X theirs main   # 冲突时自动使用 theirs（远程版本）
git rebase -X ours main     # 冲突时自动使用 ours（本地版本）
```

## 1.6 Rebase 的风险与规范

**绝对不能 rebase 的提交**：
- 已经推送到远程且被他人使用的提交
- 作为其他分支基础的提交
- 包含协作审核的提交

**安全规范**：

```bash
# ✓ 安全的 rebase（仅本地分支）
git rebase main         # 本地 feature 分支变基
git rebase -i HEAD~3    # 本地整理提交

# ✗ 危险的 rebase
git push --force origin feature  # 如果 feature 有协作者

# ✓ 安全的方式：force-with-lease
git push --force-with-lease origin feature
```

## 1.7 Rebase vs Merge 决策指南

| 场景 | 推荐 | 原因 |
|------|------|------|
| 公共分支（main/develop） | Merge | 保留真实历史，所有协作者可见 |
| 个人功能分支 | Rebase | 保持历史整洁，避免无意义的 merge commit |
| 多人协作功能分支 | Merge | 避免 rebase 导致他人冲突 |
| 拉取上游更新 | Rebase | 避免产生 merge commit |
| 发布分支 | Merge | 保留准确的发布历史 |

# 2. Cherry-Pick

## 2.1 基本用法

Cherry-pick 将其他分支的**特定提交**应用到当前分支：

```bash
# 应用单个提交
git cherry-pick abc123

# 应用连续多个提交
git cherry-pick abc123..def456  # 从 abc123 之后到 def456（不含 abc123）
git cherry-pick abc123^..def456 # 包含 abc123 到 def456

# 应用不连续的多个提交
git cherry-pick abc123 def456 789ghi
```

## 2.2 Cherry-Pick 场景

**场景 1：Hotfix 需要同时应用到多个分支**

```bash
# 修复 main 分支上的 bug
git checkout -b hotfix/security-fix main
# 修复并提交
git commit -m "fix: security vulnerability"
git push origin hotfix/security-fix

# 合并到 main
git checkout main
git merge hotfix/security-fix

# Cherry-pick 修复到 release 分支
git checkout release/v1.0
git cherry-pick main  # 或指定具体的 commit hash
```

**场景 2：从 feature 分支只提取部分提交**

```bash
git checkout main
git cherry-pick abc123  # 只提取某个 commit，不要整个分支
```

## 2.3 Cherry-Pick 选项

```bash
# -n：不自动提交（暂存修改，可以继续 pick 后统一提交）
git cherry-pick -n abc123
git cherry-pick -n def456
git commit -m "feat: combined changes"

# -x：在提交信息中添加来源（标记 cherry-pick 来源）
git cherry-pick -x abc123
# 提交信息会包含：(cherry picked from commit abc123)

# --signoff：添加 Signed-off-by 签名
git cherry-pick --signoff abc123

# -e：编辑提交信息
git cherry-pick -e abc123
```

# 3. Reset — 重置当前分支

## 3.1 三棵树模型回顾

Git 维护三个树状结构：

```
HEAD          → 当前分支指向的提交（最近一次提交）
Index         → 暂存区（下一次提交的内容）
Working Tree  → 实际文件系统的内容
```

不同的 reset 模式影响不同的树：

| 模式 | HEAD | Index | Working Tree | 应用场景 |
|------|------|-------|-------------|----------|
| `--soft` | ✓ | ✗ | ✗ | 撤销 commit，保留 staged 状态 |
| `--mixed` | ✓ | ✓ | ✗ | 撤销 commit 和 stage，保留工作目录修改 |
| `--hard` | ✓ | ✓ | ✓ | 完全丢弃未提交的修改 |

## 3.2 Soft Reset

```bash
git reset --soft HEAD~1
```

```
执行前：
HEAD → commit C (最新)
Index → C 的内容
Working Tree → C 的内容

执行 --soft HEAD~1 后：
HEAD → commit B
Index → C 的内容（暂存区保留了 C 的更改）
Working Tree → C 的内容（文件不变）

结果：撤销了提交，但修改内容还在暂存区，可以直接重新 commit
```

**应用场景**：合并多个提交为一个

```bash
# 假设有 3 个零碎提交
git log --oneline
# abc123 fix: typo
# def456 fix: format
# 789aaa feat: add login

# 合并为 1 个提交
git reset --soft HEAD~3
git commit -m "feat: add login module"
```

## 3.3 Mixed Reset（默认）

```bash
git reset HEAD~1
# 等价于
git reset --mixed HEAD~1
```

```
执行前：
HEAD → commit C
Index → C 的内容
Working Tree → C 的内容

执行 --mixed HEAD~1 后：
HEAD → commit B
Index → B 的内容（暂存区恢复到 B 的状态）
Working Tree → C 的内容（文件不变，但变为 modified）

结果：撤销了提交和暂存，修改保留在工作目录
```

**应用场景**：

```bash
# 撤销暂存
git reset HEAD file.txt
git reset -- file.txt  # 最新语法

# 从暂存区移除所有文件
git reset

# 回退提交，重新整理
git reset HEAD~1
git add src/
git commit -m "better message"
```

## 3.4 Hard Reset

```bash
git reset --hard HEAD~1
```

```
执行前：
HEAD → commit C
Index → C 的内容
Working Tree → C 的内容

执行 --hard HEAD~1 后：
HEAD → commit B
Index → B 的内容
Working Tree → B 的内容

结果：完全回到 commit B 的状态，C 的所有修改全部丢失！
```

```bash
# 丢弃所有本地修改
git reset --hard

# 恢复到远程的最新版本
git fetch origin
git reset --hard origin/main

# 完全丢弃本地提交（回退 N 次）
git reset --hard HEAD~5

# 完全丢弃合并
git reset --hard ORIG_HEAD
```

**警告**：`--hard` 会丢失工作目录中未提交的修改。

## 3.5 Reset 与路径参数

```bash
# 只重置特定文件（仅影响 Index，不影响 HEAD 和工作树）
git reset file.txt
git reset HEAD file.txt    # 从暂存区移除文件
git reset --hard file.txt  # 不支持路径模式！
```

使用路径时，reset 只会影响 Index，不会移动 HEAD。

# 4. Revert — 撤销提交

## 4.1 基本用法

Revert 创建一个**新的提交**来撤销指定提交的更改，不会修改历史：

```bash
# 撤销特定提交
git revert abc123

# 撤销多个提交（按时间从新到旧）
git revert abc123 def456

# 撤销一段范围的提交
git revert --no-commit HEAD~3..HEAD  # 撤销最近 3 个提交但不提交

# 撤销合并提交（需要指定主分支）
git revert -m 1 merge-commit-hash  # -m 1 表示保留第一父分支（主分支）
```

## 4.2 Revert 的底层机制

```bash
# 假设历史：
# d4e5f6 fix: bug
# c3d4e5 feat: add validation
# b2c3d4 fix: login bug
# a1b2c3 feat: add login

# 执行 revert c3d4e5 后：
git revert c3d4e5
# 创建一个新的提交 e5f6a7，其内容与 c3d4e5 的修改相反
# 历史变为：
# e5f6a7 Revert "feat: add validation"
# d4e5f6 fix: bug
# c3d4e5 feat: add validation
# b2c3d4 fix: login bug
# a1b2c3 feat: add login
```

## 4.3 Revert vs Reset 对比

| 对比维度 | Revert | Reset |
|---------|--------|-------|
| 是否修改历史 | 否（安全） | 是（危险，尤其是 --hard） |
| 是否产生新提交 | 是 | 否 |
| 适用场景 | 公共分支（已推送） | 本地分支（未推送） |
| 冲突可能 | 可能有 | 只在 --hard 时无冲突 |
| 对协作的影响 | 安全，其他人可直接 pull | 需要 force push，影响协作者 |

**选择的黄金法则**：

```bash
# 已推送到公共分支 → 使用 revert
git revert HEAD

# 仅在本地 → 可以使用 reset
git reset --hard HEAD~1
```

# 5. Reflog 与数据恢复

## 5.1 误操作恢复示例

```bash
# 场景 1：误 reset --hard
git reset --hard HEAD~5
# 突然意识到搞错了！
git reflog
# abc123 HEAD@{0}: reset: moving to HEAD~5
# def456 HEAD@{1}: commit: important work
git reset --hard HEAD@{1}
# 或
git reset --hard def456

# 场景 2：误删除分支
git branch -D feature/login
git branch -D feature/register
# 分支丢了！
git reflog --all  # 查看所有 reflog（包括分支）
# 或者
git fsck --lost-found  # 查找所有 dangling 的对象
# 找到丢失的 commit
git branch recovered-branch abc123

# 场景 3：误 rebase
git rebase main
# 结果发现 rebase 搞乱了
git reflog
git reset --hard HEAD@{1}  # 回到 rebase 之前
```

## 5.2 Reflog 的持久性

```bash
# Reflog 数据保留时间（默认）
git config gc.reflogExpire 90.days          # 可达引用的 reflog 保留 90 天
git config gc.reflogExpireUnreachable 30.days  # 不可达引用的 reflog 保留 30 天

# 在这些时间之前，都可以通过 reflog 恢复
# 超过时间后，git gc 会清理这些记录

# 手动保留重要的 reflog 条目
git config gc.reflogExpire never       # 永久保留（不推荐）

# 强制清理 reflog（无法再恢复！）
git reflog expire --expire=now --all
```

# 6. 撤销操作的汇总对照

| 目标 | 本地（未推送） | 远程（已推送） |
|------|--------------|--------------|
| 撤销工作目录修改 | `git restore file` | `git restore file` |
| 撤销暂存区 | `git restore --staged file` | `git restore --staged file` |
| 修改最近提交信息 | `git commit --amend` | `git commit --amend; git push --force-with-lease` |
| 修改最近提交内容 | `git commit --amend` | `git commit --amend; git push --force-with-lease` |
| 撤销最近一次提交 | `git reset --soft HEAD~1` | `git revert HEAD` |
| 撤销多次提交 | `git reset --hard HEAD~3` | `git revert HEAD~3..HEAD` |
| 删除分支 | `git branch -D feature` | `git push origin --delete feature` |
| 恢复误删内容 | `git reflog; git reset --hard <hash>` | 需要先恢复远程备份 |
