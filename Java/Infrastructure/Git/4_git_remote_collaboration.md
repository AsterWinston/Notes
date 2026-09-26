
# 1. 远程仓库概述

Git 是分布式版本控制系统，远程仓库是团队协作的核心枢纽。每个开发者都可以有多个远程仓库，常见场景：

- **GitHub/GitLab/Gitee**：代码托管平台
- **公司自建 Git 服务器**：内部私有仓库
- **同事的仓库**：通过 SSH 或直接文件共享

## 1.1 远程仓库的物理结构

远程仓库本质是一个**裸仓库**（bare repository），没有工作目录：

```bash
# 本地仓库与裸仓库的区别
# 本地仓库（有工作目录）
.git/          # Git 数据
src/           # 工作目录
README.md

# 裸仓库（无工作目录，仅用于共享）
repo.git/
├── HEAD
├── config
├── objects/
├── refs/
└── hooks/
```

```bash
# 创建裸仓库
git init --bare project.git
# 或克隆为裸仓库
git clone --bare https://github.com/user/repo.git repo.git
```

# 2. 远程仓库管理

## 2.1 添加与管理远程仓库

```bash
# 添加远程仓库
git remote add origin https://github.com/user/repo.git
git remote add origin git@github.com:user/repo.git    # SSH
git remote add upstream https://github.com/other/repo.git

# 查看远程仓库
git remote
git remote -v                      # 显示 fetch/push URL
git remote show origin             # 显示详细信息
git remote get-url origin          # 获取 URL

# 修改远程仓库
git remote set-url origin https://new-url.com/repo.git
git remote rename origin upstream

# 删除远程仓库
git remote remove upstream
```

## 2.2 多远程仓库协作

```bash
# 典型的 Fork 协作模式
# origin: 自己的 fork
# upstream: 原始项目仓库

git remote add origin git@github.com:myuser/repo.git
git remote add upstream https://github.com/original/repo.git

# 同步上游更新
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# 从不同的远程仓库拉取
git fetch origin        # 从 origin 拉取
git fetch upstream      # 从 upstream 拉取

# 推送到特定远程
git push origin feature/login
git push upstream main
```

## 2.3 远程仓库协议对比

| 协议 | URL 格式 | 认证 | 传输速度 | 适用场景 |
|------|---------|------|---------|---------|
| HTTPS | `https://github.com/user/repo.git` | 密码/Token | 中（加密） | 通用，防火墙友好 |
| SSH | `git@github.com:user/repo.git` | 密钥对 | 快（加密） | 日常开发推荐 |
| Git | `git://github.com/user/repo.git` | 无认证 | 最快（无加密） | 只读，极少使用 |
| Local | `/path/to/repo.git` | 文件权限 | 最快 | 本地文件系统 |

**HTTPS vs SSH 选择**：

```bash
# HTTPS（推荐新用户/公司防火墙环境）
git clone https://github.com/user/repo.git
# 使用个人访问令牌（Token）代替密码
# GitHub: Settings → Developer settings → Personal access tokens

# SSH（推荐日常开发，免密操作）
ssh-keygen -t ed25519 -C "your@email.com"
# 公钥添加到 GitHub/GitLab/Gitee
git clone git@github.com:user/repo.git
```

# 3. 核心协作操作

## 3.1 Fetch — 安全拉取

`fetch` 从远程仓库下载提交和引用，但**不修改本地分支**：

```
fetch 前：
本地        main: A──B──C
远程 origin/main: A──B──D──E

fetch 后：
本地        main: A──B──C（不变）
远程跟踪    origin/main: A──B──D──E
```

```bash
# 基本 fetch
git fetch origin
git fetch origin main           # 只拉取 main 分支
git fetch --all                 # 拉取所有远程仓库
git fetch --prune               # 拉取并清理已删除的远程分支

# fetch 后的操作
git log origin/main --oneline    # 查看远程新提交
git diff main origin/main       # 比较本地与远程的差异
git merge origin/main           # 合并远程到本地
git rebase origin/main          # 在远程基础上变基
```

## 3.2 Pull — 拉取并合并

`pull` = `fetch` + `merge`（默认）：

```bash
# 基本 pull
git pull origin main

# 等价于：
git fetch origin main
git merge origin/main

# rebase 方式 pull
git pull --rebase origin main
# 等价于：
git fetch origin main
git rebase origin/main

# 设置默认 pull 方式
git config pull.rebase false   # merge（默认）
git config pull.rebase true    # rebase
git config pull.ff only        # 仅允许 fast-forward
```

**pull 的三种策略对比**：

```
场景：本地 main 有提交 C，远程 main 有提交 D

pull (merge)：
A──B──C──D──M  (M = merge commit，合并 C 和 D)

pull --rebase：
A──B──D──C'     (C' 在 D 基础上重新应用)

pull --ff-only：
无法执行（因为不是 fast-forward，有分叉）
```

## 3.3 Push — 推送

```bash
# 基本推送
git push origin feature/login

# 设置上游并推送
git push -u origin feature/login
# -u = --set-upstream，设置跟踪关系，后续可直接用 git push

# 推送到远程的另一个分支名
git push origin feature/login:feature/new-name

# 删除远程分支
git push origin --delete feature/old
git push origin :feature/old     # 等价于上面（推送空到远程）

# 推送 tags
git push origin v1.0.0            # 推送单个 tag
git push origin --tags            # 推送所有 tags

# 强制推送（危险操作）
git push --force origin feature/login         # 简单强制
git push --force-with-lease origin feature    # 安全强制（检查远程是否变化）
```

**--force vs --force-with-lease**：

```bash
# --force：无条件覆盖远程
git push --force origin feature

# --force-with-lease：检查远程是否被其他人推送过
# 如果远程有新提交，push 会被拒绝
git push --force-with-lease origin feature
```

**Push 被拒绝的常见原因**：

```bash
# 1. 本地落后于远程
git push origin main
# ! [rejected] main -> main (non-fast-forward)
# error: failed to push some refs
# 解决：
git pull --rebase origin main   # 先变基拉取
git push origin main            # 再推送

# 2. 分支保护（GitHub/GitLab 设置）
# 解决方法：创建 PR/MR 而非直接 push
```

# 4. 远程与本地分支的同步

## 4.1 创建本地分支跟踪远程分支

```bash
# 方式 1：基于远程分支创建
git checkout -b feature/login origin/feature/login
git switch -c feature/login origin/feature/login

# 方式 2：现有分支设置跟踪
git branch -u origin/develop feature/login

# 方式 3：推送时设置
git push -u origin feature/login
```

## 4.2 同步远程已删除的分支

```bash
# 远程分支被删除后，本地跟踪引用不会自动删除
git branch -r
# origin/feature/old  # 远程已删除，但本地还有

# 清理远程跟踪引用
git fetch --prune
git remote prune origin

# 查看上游丢失的分支
git branch -vv
# feature/old abc123 [origin/feature/old: gone] old work

# 批量删除上游已不存在的本地分支
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -d
```

## 4.3 推送本地分支到远程

```bash
# 推送新分支
git push origin feature/login

# 推送所有分支
git push --all origin

# 推送当前分支到同名远程分支
git push origin HEAD

# 推送并删除其他不存在的远程分支
git push --prune origin
```

# 5. 深入理解 Upstream（上游）

Upstream 是本地分支与远程分支的跟踪关系，定义了 `git pull` 和 `git push` 的默认行为。

## 5.1 Upstream 配置

```bash
# 查看上游
git branch -vv
# * main    abc123 [origin/main] last commit
#   feature def456 [origin/feature: ahead 2, behind 1] work

# ahead 2：本地比远程多 2 个提交
# behind 1：远程比本地多 1 个提交

# 手动设置上游
git branch -u origin/develop
git branch --set-upstream-to=origin/develop

# 取消上游
git branch --unset-upstream
```

## 5.2 没有上游时的行为

```bash
# 本地分支没有上游时
git pull
# There is no tracking information for the current branch.
# Please specify which branch you want to merge with.

git push
# fatal: The current branch feature has no upstream branch.

# 解决方法：指定完整的远程分支
git pull origin feature
git push origin feature
```

## 5.3 Push 策略

```bash
# git push 的默认行为由 push.default 控制
git config push.default nothing      # 必须显式指定
git config push.default current      # 推送当前分支到同名远程分支（推荐）
git config push.default simple       # 推送当前分支到上游同名分支（默认）
git config push.default matching     # 推送所有匹配的本地分支到远程
git config push.default upstream     # 推送到上游分支（可能不同名）
```

# 6. 标签的远程同步

## 6.1 推送与拉取标签

```bash
# 推送单个标签
git push origin v1.0.0

# 推送所有标签
git push origin --tags

# 删除远程标签
git push --delete origin v1.0.0
git push origin :refs/tags/v1.0.0  # 等价

# 拉取标签（fetch 会自动拉取）
git fetch origin --tags
git fetch origin 'refs/tags/*:refs/tags/*'  # 显式拉取
```

## 6.2 标签与分支的冲突处理

```bash
# 标签名和分支名不能相同
git tag main  # 错误！会与 main 分支冲突

# 查看特定引用类型
git show-ref --heads      # 只显示分支
git show-ref --tags       # 只显示标签
```

# 7. 跨平台协作

## 7.1 GitHub Fork 协作流程

```bash
# 1. Fork 项目到自己的 GitHub 账号
# 2. 克隆自己的 Fork
git clone git@github.com:myuser/repo.git
cd repo

# 3. 添加上游仓库
git remote add upstream https://github.com/original/repo.git

# 4. 创建特性分支
git checkout -b feature/new-feature

# 5. 开发并提交
git add .
git commit -m "feat: add new feature"

# 6. 推送到自己的 Fork
git push origin feature/new-feature

# 7. 在 GitHub 创建 Pull Request

# 8. 保持分支同步
git fetch upstream
git rebase upstream/main
git push --force-with-lease origin feature/new-feature
```

## 7.2 GitLab Merge Request 协作

```bash
# GitLab 流程类似，但有以下差异：
# - Fork 在 GitLab 中称为 "Fork project"
# - PR 在 GitLab 中称为 "Merge Request"

# GitLab CI/CD 集成
# .gitlab-ci.yml 会在 MR 创建时自动触发流水线
```

## 7.3 Gitee 协作

```bash
# Gitee 的 Pull Request 流程与 GitHub 类似
git remote add gitee git@gitee.com:myuser/repo.git

# 同时推送到多个平台
git push origin main
git push gitee main
```

## 7.4 多平台镜像同步

```bash
# 配置多个远程仓库实现镜像同步
git remote add github git@github.com:user/repo.git
git remote add gitee git@gitee.com:user/repo.git
git remote add gitlab git@gitlab.com:user/repo.git

# 推送所有远程
git push github main
git push gitee main
git push gitlab main

# 或使用 git remote set-url --add 添加多个 URL
git remote set-url --add origin git@gitee.com:user/repo.git
git push origin main  # 会同时推送到两个 URL
```

# 8. 远程仓库的安全操作

## 8.1 敏感信息处理

```bash
# 误提交敏感信息的紧急修复
# 1. 修改本地文件
# 2. 使用 filter-branch 或 BFG 清除历史
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch .env" \
  --prune-empty --tag-name-filter cat -- --all

# 3. 强制推送到远程（需要协调团队）
git push --force --all origin

# 4. 告知所有协作者重新克隆（重要！）
```

## 8.2 分支保护规则

```bash
# 服务端配置（GitHub/GitLab Web UI）
# 常见保护规则：
# - 禁止直接 push main 分支
# - 要求 PR/MR 审核
# - 要求 CI 通过
# - 要求线性历史
# - 要求签名的提交

# 配合 pre-receive hook 实现自定义规则
# .git/hooks/pre-receive
```

# 9. 深入理解 Git 数据传输协议

## 9.1 智能协议 vs 哑协议

Git 支持两种传输协议：

| 协议 | 描述 | 性能 |
|------|------|------|
| 智能协议 | 服务器端协商最小传输量（SSH/HTTPS） | 快 |
| 哑协议 | 直接传输文件（HTTP 无智能支持） | 慢（极少用） |

智能协议工作流程：

```
客户端：服务器，我有 commit A，缺少哪些？
服务器：你需要 B、C、D，这里有个 pack
客户端：收到，解包并更新引用
```

## 9.2 Git 传输的最小单位

```bash
# 传输单位不是文件，而是对象
# Git 传输的是对象图的增量

git fetch
# remote: Counting objects: 10, done.
# remote: Compressing objects: 100% (6/6), done.
# remote: Total 10 (delta 4), reused 8 (delta 3)
# Receiving objects: 100% (10/10), done.
# Resolving deltas: 100% (4/4), done.
```

## 9.3 Reference Specification（Refspec）

Refspec 定义了 fetch/push 时引用的映射规则：

```bash
# fetch 默认 refspec
git config remote.origin.fetch
# +refs/heads/*:refs/remotes/origin/*
# + 号表示允许非 fast-forward 更新

# push 默认 refspec
git config remote.origin.push
# refs/heads/*:refs/heads/*

# 自定义 refspec
git fetch origin refs/heads/*:refs/remotes/origin/*
git push origin refs/heads/feature:refs/heads/feature

# 仅拉取特定分支
git fetch origin refs/heads/main:refs/remotes/origin/main

# 拉取 tags
git fetch origin refs/tags/*:refs/tags/*

# 交换分支（危险操作）
git push origin :refs/heads/feature  # 删除远程分支
git push origin refs/heads/feature:refs/heads/develop  # 推送 feature 到 develop
```
