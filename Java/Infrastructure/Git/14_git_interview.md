
# 1. 基础概念

## 1.1 Git 与 SVN 的区别

**Q**: Git 和 SVN 的核心区别是什么？

**A**:

| 维度 | Git（分布式） | SVN（集中式） |
|------|-------------|--------------|
| 架构 | 每个节点有完整仓库 | 中央服务器存所有数据 |
| 离线操作 | 完全支持 | 需要网络 |
| 分支 | 轻量指针，毫秒级创建 | 目录复制，慢 |
| 存储 | 内容寻址，快照存储 | 差异存储 |
| 完整性 | SHA-1 校验 | 无强制校验 |
| 网络依赖 | 仅 push/pull 需要 | 大部分操作需要 |

**Q**: Git 如何保证数据完整性？

**A**: Git 使用 SHA-1 散列算法对所有内容进行校验和。对象存储格式为 `类型 + 长度 + 内容` 的 SHA-1 hash。任何内容变化都会导致 hash 变化。Git 还会定期通过 `git fsck` 验证所有对象的完整性。

## 1.2 Git 三棵树

**Q**: 解释 Git 的三棵树（Three Trees）概念。

**A**:

| 树 | 含义 | 用途 |
|----|------|------|
| HEAD | 最后一次提交 | 当前分支指向的 commit |
| Index | 暂存区 | 下一次要提交的内容 |
| Working Tree | 工作目录 | 实际文件系统的内容 |

操作关系：`git add` 将 Working Tree → Index，`git commit` 将 Index → HEAD。

## 1.3 Git 对象类型

**Q**: Git 有哪四种对象？各自的作用是什么？

**A**:

- **blob**: 存储文件内容，不保存文件名，相同内容的文件共享同一个 blob
- **tree**: 存储目录结构，包含文件名、权限和指向 blob/subtree 的指针
- **commit**: 存储快照引用，包含 tree hash、parent、作者、提交信息
- **tag**: 为特定 commit 命名，支持 GPG 签名

# 2. 分支与合并

## 2.1 Merge 与 Rebase 的区别

**Q**: Git merge 和 git rebase 有什么区别？各自的应用场景是什么？

**A**:

```bash
# Merge：保留完整历史，产生 merge commit
git merge feature
# A──B──C──D──M
#          \──E──/

# Rebase：历史线性，不产生 merge commit
git rebase main
# A──B──C──E'──D'
```

| 对比维度 | Merge | Rebase |
|---------|-------|--------|
| 历史 | 保留真实合并点 | 线性历史 |
| 安全性 | 安全，不修改历史 | 危险，修改提交 hash |
| 冲突解决 | 一次性解决 | 每个提交依次解决 |
| 适用场景 | 公共分支 | 个人功能分支 |
| 还原 | 容易（revert merge） | 困难 |

**黄金法则**：已推送到公共分支的提交永不 rebase。

## 2.2 Fast-Forward 合并

**Q**: 什么是 Fast-Forward 合并？什么时候会产生？

**A**: 当目标分支自分支点以来没有新的提交时，Git 直接将 HEAD 指针前移，不产生 Merge Commit。

```bash
# 可以 Fast-Forward
main: A──B──C
feature: A──B──C──D

# 不能 Fast-Forward（有分叉）
main: A──B──C──E
feature: A──B──C──D
```

## 2.3 冲突解决

**Q**: Git 合并冲突是怎么产生的？如何解决？

**A**: 两个分支修改了同一个文件的同一区域且内容不同时产生冲突。

```bash
# 冲突标记
<<<<<<< HEAD
当前分支的内容
=======
合并分支的内容
>>>>>>> feature

# 解决方式
1. 手动编辑文件，保留正确内容
2. git add 标记已解决
3. git commit 完成合并
```

## 2.4 Cherry-Pick 的应用场景

**Q**: 什么场景下需要使用 cherry-pick？

**A**:

- 从 feature 分支提取特定提交到主分支
- 将 hotfix 应用到多个维护分支
- 从废弃的分支中挑出有用的提交

# 3. 撤销操作

## 3.1 Reset vs Revert

**Q**: 生产环境出现 bug，如何回滚代码？为什么推荐 revert 而非 reset？

**A**:

```bash
# 本地分支未推送：可以使用 reset
git reset --hard HEAD~1

# 公共分支已推送：必须使用 revert
git revert HEAD
```

原因：`reset` 修改历史，其他人 fetch 后 pull 会失败。`revert` 创建一个新提交来抵消之前的修改，历史不变，所有人都可以正常 pull。

## 3.2 Reset 的三个模式

**Q**: `--soft`、`--mixed`、`--hard` 的区别？

**A**:

| 模式 | HEAD | 暂存区 | 工作目录 | 场景 |
|------|------|--------|---------|------|
| --soft | 移动 | 保留 | 保留 | 合并多次提交 |
| --mixed | 移动 | 重置 | 保留 | 撤销暂存 |
| --hard | 移动 | 重置 | 重置 | 完全丢弃修改 |

# 4. 底层原理

## 4.1 .git 目录结构

**Q**: `.git` 目录下有哪些重要文件/目录？各自的作用？

**A**:

```
HEAD        → 指向当前分支
config      → 仓库配置
index       → 暂存区（二进制文件）
objects/    → Git 对象存储（blob/tree/commit/tag）
refs/       → 引用（分支、标签、远程跟踪）
logs/       → reflog
hooks/      → Git hooks 脚本
```

## 4.2 SHA-1 冲突概率

**Q**: Git 使用 SHA-1 存储对象，会不会发生哈希冲突？冲突了怎么办？

**A**: 理论上有冲突可能，但概率极低。SHA-1 有 2^160 个可能值。需要约 2^51 次计算才能找到冲突。Git 在 2.29+ 版本开始支持 SHA-256。如果发生冲突，Git 会添加额外的前缀来区分。

## 4.3 Pack 文件与 Delta 压缩

**Q**: Git 的 pack 文件是如何工作的？什么是 delta 压缩？

**A**: 当 loose 对象积累到一定数量，`git gc` 会将它们打包为 pack 文件。在打包时，Git 会识别相似的对象，只存储差异（delta），然后使用 zlib 进一步压缩。

```bash
# 对象 A: "hello world"
# 对象 B: "hello world!!!"
# Pack 文件存储：
# base: "hello world"
# delta: "+!!!"（偏移 11，长度 3）
```

# 5. 分布式协作

## 5.1 Fetch vs Pull

**Q**: `git fetch` 和 `git pull` 的区别？

**A**:

```bash
# fetch：只下载数据，不合并
git fetch origin
git log origin/main  # 查看远程更新
git merge origin/main  # 手动合并

# pull：fetch + merge（自动合并）
git pull origin main
```

推荐先 fetch 再手动合并/变基，可以查看远程更新后再决定如何合并。

## 5.2 Rebase 的黄金法则

**Q**: 为什么不能对已推送的公共分支执行 rebase？

**A**: Rebase 会修改提交的 hash。如果公共分支被 rebase 后 force push，其他人在下次 pull 时会出现大量重复提交和冲突，造成混乱。

```bash
# 场景：A force push 了 rebase 后的 main
# B 现在 pull：
git pull
# 出现大量冲突和重复提交

# 恢复方法
git pull --rebase
# 或重新 clone
```

# 6. 场景题

## 6.1 紧急修复

**Q**: 正在 feature 分支开发新功能，需要紧急修复生产 bug，手头代码还没提交，怎么办？

**A**:

```bash
# 方法 1：使用 stash（推荐）
git stash -u -m "WIP: login feature"
git checkout -b hotfix/critical-bug main
# 修复、提交、合并
git checkout feature/login
git stash pop

# 方法 2：使用 worktree
git worktree add ../hotfix hotfix/critical-bug
cd ../hotfix
# 修复
cd ../project
git worktree remove ../hotfix
```

## 6.2 合并错误的修复策略

**Q**: 把一个错误的分支合并到了 main，已经 push 到远程。如何在保留历史的情况下修复？

**A**:

```bash
# 使用 revert（推荐，不修改历史）
git revert -m 1 HEAD
git push origin main

# 修复后再合并正确的分支
git checkout main
git merge --no-ff correct-feature
git push origin main
```

## 6.3 大文件误提交

**Q**: 不小心把 2GB 的日志文件提交到了仓库，已经 push 了。如何彻底清理？

**A**:

```bash
# 1. 立即废弃暴露的敏感信息
# 2. 使用 BFG 清除大文件
java -jar bfg.jar --strip-blobs-bigger-than 100M repo.git

# 3. 清理引用
cd repo
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# 4. 强制推送
git push --force --all origin
git push --force --tags origin

# 5. 通知所有协作者重新 clone
```

## 6.4 代码丢失恢复

**Q**: 误删了未合并的功能分支，怎么恢复？

**A**:

```bash
# 方法 1：reflog
git reflog --all
git branch recovered-branch abc123

# 方法 2：dangling commit
git fsck --lost-found
git branch recovered-branch abc123
```

# 7. 实践题

## 7.1 实现一个 Git Flow 工作流

**Q**: 描述 Git Flow 的完整流程，包括 feature、release、hotfix 分支的生命周期。

**A**: Git Flow 定义了 5 种分支：

| 分支 | 来源 | 合并到 | 说明 |
|------|------|--------|------|
| main | — | — | 生产代码 |
| develop | main | — | 开发主线 |
| feature/* | develop | develop | 功能开发 |
| release/* | develop | main + develop | 发布准备 |
| hotfix/* | main | main + develop | 紧急修复 |

## 7.2 持续部署的分支策略

**Q**: 一个 SaaS 产品，每天需要多次部署到生产环境，推荐哪种工作流？为什么？

**A**: 推荐 **GitHub Flow** 或 **Trunk-Based Development**。

原因：
- 简单：只有 main + feature 分支
- 快速：PR 合并后自动部署
- 灵活：支持每天多次部署
- 低摩擦：没有 release 分支管理的开销

## 7.3 多版本维护

**Q**: 产品同时维护 v1.0、v1.1、v2.0 三个版本，如何管理分支？

**A**:

```bash
# 每个版本有独立的维护分支
main          # 当前开发版本 v2.x
v1.0.x        # v1.0 维护分支
v1.1.x        # v1.1 维护分支

# 修复提交流程
git checkout -b hotfix/1.0.1 v1.0.x
git commit -m "fix: critical bug in v1.0"
git checkout v1.0.x
git merge --no-ff hotfix/1.0.1
git tag -a v1.0.1 -m "v1.0.1 hotfix"

# 同步修复到其他版本
git cherry-pick -x abc123  # 将修复应用到 v1.1.x
git cherry-pick -x abc123  # 将修复应用到 main
```

# 8. 快速问答

**Q**: `HEAD~3` 和 `HEAD^^^` 有什么区别？
**A**: 没有区别，都表示向回 3 个提交。`HEAD~3` 等价于 `HEAD^^^`，但 `~` 是沿着第一父分支追溯，`^` 可以指定第几个父分支（如 `HEAD^2` 表示合并提交的第二个父分支）。

**Q**: `git revert` 和 `git reset` 都可以用来撤销，为什么推荐 revert？
**A**: `revert` 不改变历史，产生新的撤销提交，适合公共分支。`reset` 修改历史，只适合本地分支。

**Q**: 什么时候需要 force push？
**A**: 修改了已推送的提交历史后需要 force push。推荐使用 `--force-with-lease`（安全模式）替代 `--force`，它会检查远程是否有新的提交。

**Q**: `git stash` 和 `git worktree` 的使用场景有什么区别？
**A**: `stash` 用于临时保存当前分支的修改，切换到其他分支处理紧急事务。`worktree` 用于同时维护多个分支的工作目录，可以并行工作。

**Q**: Git 中 detached HEAD 是什么？
**A**: HEAD 不指向任何分支，直接指向某个提交。常见于 `git checkout abc123`。在 detached HEAD 状态下提交会创建"孤儿"提交，切换分支后容易丢失（除非创建新分支或记下 commit hash）。

**Q**: `git fetch` 和 `git pull` 的区别？
**A**: `fetch` 只下载远程数据不合并，`pull` = `fetch` + `merge`。推荐先用 fetch 查看差异，再决定合并方式。
