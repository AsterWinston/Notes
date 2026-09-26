
# 1. 分支的本质

Git 的分支本质上是一个指向 commit 对象的**可移动指针**。每次提交，当前分支指针自动前移。

## 1.1 分支的物理存储

```
.git/refs/heads/
├── main        # 包含一个 commit hash
├── develop
├── feature/login
└── feature/register
```

```bash
# 分支就是文件里的一个 hash
cat .git/refs/heads/main
# abc123def456...

# 查看所有分支
git branch
git branch -v          # 显示分支与最新提交
git branch -vv         # 显示跟踪关系
```

## 1.2 为什么 Git 分支如此轻量？

对比其他 VCS：

| VCS | 分支实现 | 创建成本 |
|-----|----------|----------|
| Git | 41 字节的指针文件 + 引用 | 毫秒级 |
| SVN | 服务器端目录复制 | 秒级（网络传输） |
| Mercurial | 类似 Git，但分支是永久的 | 毫秒级 |

Git 分支就是一个 41 字节的文件（40 位 hash + 换行），创建和切换几乎无成本。这使得 Git 鼓励**频繁创建和合并分支**的开发模式。

# 2. 分支基本操作

## 2.1 创建与切换

```bash
# 创建分支（不切换）
git branch feature/login

# 切换分支
git checkout feature/login
git switch feature/login  # Git 2.23+ 推荐

# 创建并切换
git checkout -b feature/login
git switch -c feature/login  # Git 2.23+

# 基于特定提交创建分支
git branch feature/hotfix abc123

# 基于远程分支创建本地分支
git checkout -b feature/login origin/feature/login
git switch -c feature/login origin/feature/login
```

## 2.2 查看与管理

```bash
# 列出分支
git branch                    # 本地分支
git branch -r                 # 远程分支
git branch -a                 # 全部
git branch -v                 # 显示最后提交
git branch -vv                # 显示跟踪关系

# 过滤分支
git branch --merged           # 已合并到当前分支的分支
git branch --no-merged        # 未合并的分支
git branch --merged main      # 已合并到 main 的分支

# 重命名分支
git branch -m old-name new-name           # 重命名
git branch -M old-name new-name           # 强制重命名（已存在同名时）

# 删除分支
git branch -d feature/old     # 安全删除（已合并）
git branch -D feature/old     # 强制删除（未合并也删）
```

## 2.3 分支的 commit 归属

```bash
# 查看哪些分支包含某个 commit
git branch --contains abc123
git branch -r --contains abc123  # 远程分支

# 查看哪些分支不包含某个 commit
git branch --no-contains abc123
```

# 3. Git 合并策略详解

## 3.1 Fast-Forward 合并

当目标分支自分支点以来没有新提交时，Git 执行快进合并：

```
合并前：
main:    A──B──C
feature:      C──D──E

合并后：
main:    A──B──C──D──E
feature:      C──D──E
```

特点：
- 不产生 merge commit
- 历史呈线性
- 只能在没有分叉时执行

```bash
git checkout main
git merge feature                # 默认尝试 fast-forward
git merge --ff feature            # 明确使用 fast-forward
git merge --no-ff feature         # 禁止 fast-forward（始终生成 merge commit）
git merge --ff-only feature       # 仅当可以 fast-forward 时合并，否则报错
```

**--no-ff 的合并结果**：

```
main:    A──B──C──────────F
               \          /
feature:        C──D──E──/
```

即使 feature 分支落后于 main（有分叉也适用 --no-ff，但会生成三路合并而非 fast-forward）。

## 3.2 Three-Way Merge（三路合并）

当两个分支都有新提交时，Git 使用三路合并：

```
main:    A──B──C──F
feature: A──B──D──E
共同祖先: B

合并结果（Git 自动计算）：
main:    A──B──C──F──G
               \──D──E──/
```

**三路合并的决策逻辑**：

| 共同祖先 | 我的分支 | 对方分支 | 合并结果 |
|---------|---------|---------|---------|
| A | A | A | A |
| A | B | A | B |
| A | A | B | B |
| A | B | B | B |
| A | B | C | 冲突！需要手动解决 |

```bash
# 触发三路合并
git checkout main
git merge feature
# 产生 merge commit

# 合并完成后查看
git log --oneline --graph --all
```

## 3.3 Octopus Merge（八爪鱼合并）

一次合并多个分支（通常用于同时合入多个特性分支）：

```bash
# 合并多个分支
git merge feature-a feature-b feature-c

# 限制：不能有冲突，不能 fast-forward
# 主要用于一次性集成多个独立功能
```

## 3.4 Squash Merge（压缩合并）

将分支上的**所有提交压缩成一个提交**合并到目标分支：

```bash
git checkout main
git merge --squash feature/login
git commit -m "feat: add login feature"

# 等价于：
git diff feature/login...main  # 查看差异
git apply <差异>                # 应用更改
```

**Squash 合并前后**：

```
合并前 feature 分支：
A──B──C──D──E
    │
    ├── A: "init work"
    ├── B: "fix typo"
    ├── C: "add login"
    ├── D: "fix css"
    └── E: "refactor"

合并到 main 后：
main: ...──F  (F = A+B+C+D+E 的合并结果)
```

适用场景：
- 开发分支的零碎提交不需要保留在主线历史中
- 为每个功能保持一个干净的提交历史
- 减少主线上的提交数量

## 3.5 合并策略与算法

```bash
# 指定合并策略
git merge -s recursive feature
git merge -s resolve feature
git merge -s octopus feature-a feature-b
git merge -s ours feature     # 完全忽略对方分支的内容
git merge -s subtree feature  # 子树合并

# 子策略选项（recursive 策略）
git merge -X patience feature     # 使用 patience diff 算法
git merge -X ignore-all-space     # 忽略空白差异
git merge -X theirs feature       # 遇到冲突时自动使用对方版本
git merge -X ours feature         # 遇到冲突时自动使用我方版本
```

| 策略 | 适用场景 | 特点 |
|------|---------|------|
| `recursive` | 默认，合并两个分支 | 三路合并，支持重命名检测 |
| `resolve` | 合并两个分支 | 更简单的三路合并（无重命名检测） |
| `octopus` | 合并多个分支 | 合并三个以上分支，不能有冲突 |
| `ours` | 丢弃对方更改 | 保留我方历史，忽略对方内容 |
| `subtree` | 合并有子树关联的项目 | 检测子树移动 |

# 4. 冲突处理

## 4.1 冲突的产生条件

两个分支修改了**同一个文件的同一区域**且内容不同：

```bash
# 产生冲突的场景
# main:  "function foo() { return 1; }"
# feature: "function foo() { return 2; }"
# 合并 → 冲突！
```

## 4.2 冲突标记

```bash
git merge feature
# Auto-merging file.txt
# CONFLICT (content): Merge conflict in file.txt
# Automatic merge failed; fix conflicts and then commit the result.
```

冲突文件内容：

```java
<<<<<<< HEAD
function foo() { return 1; }
=======
function foo() { return 2; }
>>>>>>> feature
```

## 4.3 解决冲突

```bash
# 方法 1：手动编辑
# 编辑冲突文件，删除冲突标记，保留想要的代码

# 方法 2：使用合并工具
git mergetool  # 配置 meld、kdiff3 等工具

# 方法 3：使用 ours/theirs
git checkout --ours file.txt    # 保留当前分支版本
git checkout --theirs file.txt  # 保留对方分支版本

# 标记为已解决
git add file.txt
git commit                      # 完成合并提交
```

## 4.4 高级冲突处理

```bash
# 查看冲突文件列表
git diff --name-only --diff-filter=U

# 查看冲突概览
git log --merge --left-right -p

# 查看冲突双方的修改
git diff HEAD...MERGE_HEAD     # 我的改动
git diff MERGE_HEAD...HEAD     # 对方的改动

# 重新合并（当合并策略不对时）
git merge --abort              # 取消合并
git reset --hard HEAD          # 回到合并前状态
git merge -X theirs feature    # 自动使用对方版本

# 手动合并后验证
git diff --check               # 检查是否有残留冲突标记
```

## 4.5 长期分支的冲突预防

```bash
# 频繁同步主分支
git checkout feature/login
git fetch origin
git rebase origin/main

# 使用 git rerere（复用已解决的冲突记录）
git config rerere.enabled true
git rerere status
```

# 5. 分支策略与模型

## 5.1 主分支保护

```bash
# 强制推送保护（服务端配置，GitHub/GitLab 设置）
# 禁止直接 push 到 main
git branch -d feature/old  # 只能删除已合并的分支

# 本地保护
git config --local --add checkout.defaultRemote origin
git config --local push.denyCurrentBranch warn
```

## 5.2 分支命名规范

```
main                     # 生产分支
develop                  # 开发主线分支
feature/<描述>          # 功能开发分支
bugfix/<描述>           # 缺陷修复分支
hotfix/<描述>           # 紧急修复分支
release/<版本号>        # 发布准备分支
support/<描述>          # 支持/补丁分支
dependabot/<描述>       # 依赖更新分支（自动）
```

## 5.3 分支的生命周期管理

```bash
# 创建 feature 分支
git checkout -b feature/user-login develop

# 开发完成，合并到 develop
git checkout develop
git merge --no-ff feature/user-login

# 删除已合并的 feature 分支
git branch -d feature/user-login

# 清理本地已不存在远程跟踪的分支
git fetch --prune
git remote prune origin
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -d
```

# 6. 远程分支与跟踪

## 6.1 远程跟踪分支

远程跟踪分支是本地对远程仓库分支状态的记录：

```bash
# 查看远程跟踪分支
git branch -r
# origin/main
# origin/develop
# origin/feature/login

# 查看远程跟踪分支的 commit
git log origin/main --oneline -5

# 查看跟踪关系
git branch -vv
# main     abc123 [origin/main] commit message
# feature  def456 commit message
```

## 6.2 设置与修改跟踪分支

```bash
# 创建时设置跟踪
git checkout -b feature/login origin/feature/login
git branch -u origin/develop feature/login

# 后续设置跟踪
git branch --set-upstream-to=origin/develop feature/login
git push -u origin feature/login  # 推送并设置上游

# 取消跟踪
git branch --unset-upstream feature/login
```

## 6.3 上游丢失的处理

```bash
# 当远程分支被删除后
git fetch --prune    # 同步远程分支状态
# x [deleted] (none) -> origin/feature/old

# 本地分支的上游不存在了
git branch -vv
# feature/old def456 [origin/feature/old: gone] old commit
git branch -d feature/old  # 删除本地分支
```

# 7. 特殊引用操作

## 7.1 ORIG_HEAD

在执行可能改变历史的重要操作时，Git 会将原始 HEAD 保存在 ORIG_HEAD 中：

```bash
# 合并或 rebase 后，可以从 ORIG_HEAD 恢复
git merge feature
# 合并后发现有问题
git reset --hard ORIG_HEAD  # 恢复到合并前
```

## 7.2 FETCH_HEAD

`git fetch` 拉取的内容会记录在 FETCH_HEAD 中：

```bash
# 拉取特定远程分支
git fetch origin feature
git merge FETCH_HEAD
```

## 7.3 MERGE_HEAD

合并时，被合并分支的最新提交记录在 MERGE_HEAD 中：

```bash
# 在冲突解决过程中
git log MERGE_HEAD --oneline  # 查看被合并分支的提交
git diff HEAD                  # 我的改动
git diff MERGE_HEAD            # 对方的改动
```

# 8. 分支的底层原理

## 8.1 分支切换时 Git 做了什么？

```bash
git checkout feature
```

1. 更新 HEAD 指向 `refs/heads/feature`
2. 读取 `feature` 指向的 commit 的 tree
3. 比较当前工作目录与新的 tree 的差异
4. 只更新有差异的文件（而非全量覆盖）

## 8.2 未跟踪文件影响切换

```bash
# 未跟踪的文件不会影响分支切换（因为它们不在 Git 索引中）
# 但已修改的已跟踪文件会影响（需要 stash 或 commit）

git checkout feature
# error: Your local changes to the following files would be overwritten by checkout:
#   src/main.js
# Please commit your changes or stash them before you switch branches.
git stash
git checkout feature
```

## 8.3 Dirty 状态的工作目录切换

```bash
# Git 允许在不同分支间切换，只要文件在目标分支中没有差异
# 场景：文件 A 在分支1和分支2内容相同，文件 B 有差异
# 我们在分支1修改了文件 A（未提交），可以切换到分支2
# 因为文件 A 在两个分支中相同，Git 会保留本地修改
git checkout branch2
# 切换成功，本地对文件 A 的修改被带到分支2
```
