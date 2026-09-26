
# 1. 误操作恢复

## 1.1 误 Reset --hard

```bash
# 症状：执行 git reset --hard 后丢失了未提交的修改
git reset --hard HEAD~3

# 解决方案：使用 reflog 恢复
git reflog
# abc123 HEAD@{0}: reset: moving to HEAD~3
# def456 HEAD@{1}: commit: important work
git reset --hard HEAD@{1}
# 或
git reset --hard def456

# 如果 reset 前没有提交过，修改无法恢复！
# 养成频繁提交的习惯，或者使用 git stash
```

## 1.2 误删除分支

```bash
# 症状：删除了未合并的分支
git branch -D feature/important

# 解决方案 1：通过 reflog
git reflog --all
# 找到分支最后一个 commit
git branch feature/important abc123

# 解决方案 2：通过 dangling commit
git fsck --lost-found
# Checking object directories: 100%
# dangling commit abc123def456...
git branch feature/important abc123def456

# 预防：使用 git branch -d（安全模式，会检查是否已合并）
```

## 1.3 误修改了最近一次提交

```bash
# 症状：commit --amend 后想恢复原始内容
git commit --amend -m "wrong message"
git commit --amend --reset-author

# 解决方案
git reflog
# abc123 HEAD@{0}: commit (amend): wrong message
# def456 HEAD@{1}: commit: original message
git reset --hard HEAD@{1}
```

## 1.4 误 Rebase 后想恢复

```bash
# 症状：rebase 后发现搞砸了
git rebase main
# 冲突很多，或结果不对

# 方案 1：使用 ORIG_HEAD
git reset --hard ORIG_HEAD

# 方案 2：使用 reflog
git reflog
git reset --hard HEAD@{5}

# 方案 3：abort（仅在 rebase 进行中）
git rebase --abort
```

## 1.5 恢复已删除的 Stash

```bash
# 症状：stash drop 或 stash clear
git stash drop stash@{2}

# 恢复刚 drop 的 stash
git fsck --unreachable | grep commit | cut -d' ' -f3 | xargs git log --oneline --no-walk
# 找到对应的 commit
git stash apply abc123
```

# 2. 合并冲突处理

## 2.1 冲突标记残留

```bash
# 场景代码中还有 <<<<<<< HEAD
# 快速查找所有冲突标记
grep -r "<<<<<<<" .
grep -r "=======" .
grep -r ">>>>>>>" .

# 使用 Git 检查
git diff --check

# 使用 grep 在暂存文件中查找
git grep -l "<<<<<<<"
git grep -l "======="
```

## 2.2 合并后反悔

```bash
# 场景：刚执行完 merge，但发现有问题
# 方法 1：简单回退
git merge --abort  # 仅在合并过程中可用

# 方法 2：使用 ORIG_HEAD
git reset --hard ORIG_HEAD  # 合并前 Git 会自动保存 ORIG_HEAD

# 方法 3：revert 合并
git revert -m 1 HEAD  # -m 1 保留第一父分支（当前分支）
```

## 2.3 二进制文件冲突

```bash
# 二进制文件不能手动解决冲突
# 方案 1：选择其中一方
git checkout --ours image.png
git checkout --theirs image.png

# 方案 2：重新生成
# 手动合并两个版本的二进制文件

# 预防：对二进制文件使用 LFS
git lfs track "*.png"
```

# 3. Push/Pull 问题

## 3.1 非快进推送被拒绝

```bash
# 场景：其他人已经推送了新提交
git push origin main
# ! [rejected] main -> main (non-fast-forward)
# error: failed to push some refs

# 解决方案 1：rebase 方式
git fetch origin
git rebase origin/main
git push origin main

# 解决方案 2：merge 方式
git pull origin main
git push origin main

# 预防：推送前先拉取
git pull --rebase
```

## 3.2 Force Push 后恢复

```bash
# 场景：同事 force push 导致你丢失了提交
# 方案 1：从其他人的本地 reflog 恢复
# 如果有另一个同事还没有 fetch，可以从他的本地恢复
git push origin abc123:main  # 将丢失的 commit 推回

# 方案 2：使用 reflog（当前仓库还有）
git reflog
git push origin HEAD@{1}:main --force-with-lease

# 方案 3：从 CI/备份恢复
# 检查 CI 的构建记录
```

## 3.3 Push 大文件失败

```bash
# 场景：push 时出现 http.postBuffer 错误
# error: RPC failed; HTTP 413 curl 22

# 增加缓冲区
git config http.postBuffer 524288000  # 500MB

# 如果还是失败，尝试 SSH 方式
git remote set-url origin git@github.com:user/repo.git

# 大文件应该使用 LFS
git lfs track "*.zip"
```

## 3.4 认证问题

```bash
# 场景：HTTPS 认证失败

# 方案 1：使用 SSH
git remote set-url origin git@github.com:user/repo.git

# 方案 2：更新认证信息
git credential reject
# 输入 protocol=https
# host=github.com
# 按回车（空行提交）

# 方案 3：使用 GitHub CLI
gh auth login

# 方案 4：使用 Git credential manager
git config --global credential.helper manager-core
```

# 4. 分支与 HEAD 问题

## 4.1 Detached HEAD（分离头指针）

```bash
# 症状：不在任何分支上
git status
# HEAD detached at abc123

# 如果你有修改需要保存：
git checkout -b new-branch-name

# 如果你只是查看代码，离开即可：
git checkout main

# 如果你做了提交但不在分支上：
git log --oneline
git checkout -b recovered-changes
```

## 4.2 分支名称冲突

```bash
# 场景：标签和分支有相同名字
# Git 会优先使用标签

# 查看具体引用
git show-ref --heads   # 分支
git show-ref --tags    # 标签

# 删除冲突的标签
git tag -d conflicted-name
git push origin --delete refs/tags/conflicted-name

# 或者指定引用类型
git show refs/heads/conflicted-name
```

# 5. 文件与内容问题

## 5.1 文件大小写问题

```bash
# 场景：Windows/macOS 不区分大小写导致的问题
# 文件名为 README.md 和 readme.md 被认为是同一个文件

# 让 Git 区分大小写
git config core.ignorecase false

# 修复大小写问题
git mv --force readme.md README.md

# 从文件系统中移除错误的文件
git rm --cached readme.md
```

## 5.2 提交了敏感信息

```bash
# 场景：不小心提交了密码、Token、密钥等

# 立即处理步骤：
# 1. 撤销提交中的敏感文件
git reset --soft HEAD~1

# 2. 从暂存区移除敏感文件
git rm --cached .env

# 3. 重新提交
git commit -m "fix: remove sensitive file"

# 4. 通知团队更新凭证
# 5. 彻底从历史中清除
# 立即废弃泄露的密钥/密码
```

## 5.3 Line Ending 问题

```bash
# 症状：git diff 显示整个文件被修改（实际只改了 whitespace）
# 原因：CRLF/LF 行尾不一致

# 配置统一的行尾处理
# Windows
git config --global core.autocrlf true

# macOS/Linux
git config --global core.autocrlf input

# 修复现有文件
git add --renormalize .
```

# 6. 子模块问题

## 6.1 子模块未初始化

```bash
# 症状：克隆后子模块目录为空
git submodule status
# -abc123 libs/common  # 前边的 - 表示未初始化

# 解决方案
git submodule init
git submodule update

# 或者一步完成
git submodule update --init --recursive
```

## 6.2 子模块 URL 变更

```bash
# 症状：子模块 URL 变了，需要更新
git submodule sync
git submodule update --init --recursive

# 手动修改 .gitmodules
git config -f .gitmodules submodule.libs/common.url https://new-url.git
git submodule sync
```

# 7. 性能相关问题

## 7.1 Git 操作缓慢

```bash
# 诊断工具
time git status
time git log --oneline
git count-objects -v

# 优化步骤
git gc --aggressive
git repack -a -d --depth=250 --window=250

# 如果仍然慢，检查是否有大文件
git rev-list --objects --all | \
  git cat-file --batch-check='%(objectsize:disk)' | \
  sort -rn | head -20
```

## 7.2 磁盘空间不足

```bash
# 查看仓库占用
du -sh .git
git count-objects -v

# 释放空间
git gc --aggressive
git prune
git reflog expire --expire=now --all
```

# 8. 问题排查的通用方法论

```bash
# 1. 查看当前状态
git status
git log --oneline -5
git reflog | head -20

# 2. 备份当前状态（防止误操作造成更大损失）
git stash -u -m "backup before recovery"
# 或创建临时分支
git checkout -b recovery-backup

# 3. 查找关键信息
git fsck --lost-found
git reflog --all

# 4. 如果所有恢复手段失败
# - 检查 IDE 的本地历史
# - 检查 CI 的构建工件
# - 检查其他同事的仓库
# - 检查备份系统
```
