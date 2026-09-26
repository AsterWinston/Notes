
# 1. Git 概述

Git 是一个**分布式版本控制系统**，由 Linus Torvalds 于 2005 年为管理 Linux 内核开发而创建。与集中式版本控制系统（如 SVN、CVS）不同，Git 的每个工作副本都是一个完整的仓库，包含完整的历史记录。

## 1.1 Git 与其他 VCS 的对比

| 特性 | Git（分布式） | SVN（集中式） | Perforce（集中式） |
|------|--------------|---------------|-------------------|
| 架构 | 每个节点全量镜像 | 中央服务器存储全部历史 | 中央服务器，客户端按需拉取 |
| 离线操作 | 完全支持 | 不支持 | 有限支持 |
| 分支成本 | 极低（指针） | 高（目录复制） | 中 |
| 性能 | 本地操作极快 | 网络依赖严重 | 中 |
| 数据完整性 | SHA-1 校验和 | 无强制校验 | MD5 校验 |

## 1.2 Git 的核心设计哲学

- **分布式**：每个仓库都是完整的备份
- **内容寻址**：所有对象通过 SHA-1 散列值寻址
- **快照存储**：每次提交都是整个项目的一个快照（而非差异存储）
- **分支是轻量指针**：分支只是指向提交的指针，创建成本极低
- **完整性保证**：所有数据在存储前计算校验和，任何时候都能校验

# 2. Git 安装与配置

## 2.1 安装方式

```bash
# Windows - 下载安装包
https://git-scm.com/download/win

# macOS - Homebrew
brew install git

# Ubuntu/Debian
apt-get install git

# CentOS/RHEL
yum install git

# 验证安装
git --version
```

## 2.2 Git 配置层级

Git 的配置分为三个层级，优先级从高到低：

| 层级 | 配置文件位置 | 作用范围 | 配置命令 |
|------|-------------|----------|----------|
| `--local` | `.git/config` | 当前仓库 | `git config --local` |
| `--global` | `~/.gitconfig` 或 `~/.config/git/config` | 当前用户 | `git config --global` |
| `--system` | `/etc/gitconfig` | 整台机器 | `git config --system` |

```bash
# 基本用户信息配置（首次使用必须设置）
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 常用配置
git config --global core.editor vim
git config --global init.defaultBranch main
git config --global pull.rebase false   # merge 方式 pull
git config --global push.default current

# 查看配置
git config --list
git config --list --global
git config user.name  # 查看单个配置项

# 别名配置（提升效率）
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate --all"
```

## 2.3 SSH 密钥配置

```bash
# 生成 SSH 密钥
ssh-keygen -t ed25519 -C "your.email@example.com"  # 推荐 ed25519
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"  # 兼容旧系统

# 启动 SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 查看公钥并添加到 GitHub/GitLab
cat ~/.ssh/id_ed25519.pub

# 测试连接
ssh -T git@github.com
ssh -T git@gitlab.com
```

## 2.4 HTTP 代理配置

```bash
# 设置 HTTP 代理
git config --global http.proxy http://proxy.example.com:8080
git config --global https.proxy https://proxy.example.com:8080

# 取消代理
git config --global --unset http.proxy
git config --global --unset https.proxy
```

# 3. Git 核心概念：三棵树

Git 维护着三个核心数据结构，称为"三棵树"：

```
   Working Directory         Staging Area (Index)            Repository (.git)
  ┌─────────────────┐      ┌──────────────────┐      ┌──────────────────────┐
  │  实际文件系统    │ ──→  │   暂存快照       │ ──→  │  永久历史记录        │
  │  untracked/modified│   │   staged changes  │      │  commits / objects   │
  └─────────────────┘      └──────────────────┘      └──────────────────────┘
```

| 树 | 名称 | 说明 | 位置 |
|----|------|------|------|
| HEAD | 最后一次提交 | 指向当前分支的最新提交 | `.git/HEAD` |
| Index | 暂存区 | 下一次提交的待定内容 | `.git/index` |
| Working Directory | 工作目录 | 实际文件系统上的文件 | 项目根目录 |

## 3.1 文件状态生命周期

```
Untracked（未跟踪）
    │
    │  git add
    ▼
Staged（已暂存）←────────────── Modified（已修改）
    │                                  ▲
    │  git commit                       │
    ▼                                  │
Committed（已提交）────────────────────┘
                                        git add / 编辑文件
```

- **Untracked**: 新文件，Git 尚未跟踪
- **Unmodified**: 已提交且未修改
- **Modified**: 已修改但未暂存
- **Staged**: 已暂存等待提交

## 3.2 文件状态查看

```bash
# 查看工作目录状态
git status
git status -s  # 简洁模式（short）

# 状态标志解释
??  # untracked 文件
 M  # 已修改未暂存（空格 + M）
M   # 已暂存（M + 空格）
MM  # 已修改、已暂存后又修改
A   # 新添加到暂存区
D   # 已删除
R   # 已重命名
C   # 已复制
U   # 合并冲突未解决
```

# 4. 基本操作详解

## 4.1 初始化与克隆

```bash
# 创建新仓库
mkdir my-project && cd my-project
git init

# 指定初始分支名
git init --initial-branch=main
git init -b main

# 克隆远程仓库
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git           # SSH 方式
git clone --depth 1 git@github.com:user/repo.git # 浅克隆（只拉取最新提交）

# 克隆到指定目录
git clone https://github.com/user/repo.git my-folder

# 裸克隆（用于搭建镜像服务器）
git clone --bare https://github.com/user/repo.git repo.git
```

## 4.2 添加与提交

```bash
# 添加文件到暂存区
git add file.txt              # 添加单个文件
git add src/                  # 添加整个目录
git add .                     # 添加所有文件（包括子目录）
git add -A                    # 添加所有变更（等价于 git add --all）
git add -p file.txt           # 交互式分块暂存（hunk-by-hunk）

# 提交
git commit -m "feat: add login feature"            # 直接提交
git commit                                             # 打开编辑器写提交信息
git commit -a -m "fix: typo"                          # 跳过暂存区，直接提交已跟踪的修改
git commit --amend -m "feat: corrected message"        # 修改最近一次提交信息
git commit --amend --no-edit                           # 不修改信息，仅补充文件

# 添加空目录（Git 不跟踪空目录，需创建 .gitkeep）
touch src/empty-dir/.gitkeep
git add src/empty-dir/.gitkeep
```

## 4.3 查看历史

```bash
# 基本日志
git log
git log --oneline                  # 一行显示
git log --oneline --graph          # 图形化显示分支结构
git log --oneline --decorate       # 显示分支/标签引用
git log --all                      # 显示所有分支历史

# 高级日志
git log --author="name"            # 按作者过滤
git log --since="2024-01-01"       # 按时间过滤
git log --until="2024-12-31"
git log -p                         # 显示每次提交的 diff
git log --stat                     # 显示统计信息（变更文件数）
git log --name-only                # 仅显示变更文件名
git log --grep="fix"               # 按提交信息搜索
git log -S "functionName"          # 按代码内容搜索（pickaxe）
git log --follow file.txt          # 跟踪文件重命名历史

# 格式化日志
git log --pretty=format:"%h - %an, %ar : %s"
# %h = 短 hash, %an = 作者名, %ar = 相对时间, %s = 标题
```

## 4.4 比较差异

```bash
# 工作目录 vs 暂存区
git diff

# 暂存区 vs 最近提交
git diff --staged
git diff --cached

# 工作目录 vs 最近提交
git diff HEAD

# 两次提交之间的差异
git diff commit1..commit2
git diff commit1 commit2

# 分支之间的差异
git diff main..feature
git diff main...feature  # 与两个分支的共同祖先比较

# 仅显示文件名
git diff --name-only
git diff --name-status

# 忽略空白差异
git diff --ignore-space-change
git diff --ignore-all-space
git diff --ignore-blank-lines
```

## 4.5 删除与移动

```bash
# 删除文件（同时从工作目录和 Git 跟踪中移除）
git rm file.txt

# 仅从 Git 跟踪移除（保留工作目录的文件）
git rm --cached file.txt

# 删除目录
git rm -r dir/

# 移动/重命名文件
git mv old.txt new.txt

# 等价于：
mv old.txt new.txt
git rm old.txt
git add new.txt

# 批量重命名
git mv src/foo.js src/bar.js
```

# 5. .gitignore 深入

## 5.1 规则语法

```gitignore
# 注释（# 开头）

# 忽略特定文件
secret.txt

# 忽略目录
build/
node_modules/

# 通配符
*.log
*.tmp
*.swp

# 忽略所有 .a 文件，但保留 lib.a
*.a
!lib.a

# 忽略当前目录下的 TODO 文件（不含子目录）
/TODO

# 忽略任何目录下的 build 文件夹
build/

# 只忽略 doc 目录下的 .txt 文件
doc/**/*.txt

# 忽略所有名字末尾有 ~ 的文件
*~
```

## 5.2 最佳实践示例

```gitignore
# === Java 项目 ===
target/
*.class
*.jar
*.war
*.ear
.settings/
.project
.classpath
*.iml
.idea/
*.log

# === Node.js 项目 ===
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.env
.env.local

# === Python 项目 ===
__pycache__/
*.py[cod]
venv/
.venv/
*.egg-info/

# === 通用 ===
.DS_Store
Thumbs.db
*.swp
*.swo
*~
.vscode/
```

## 5.3 忽略已跟踪的文件

如果文件已被 Git 跟踪，.gitignore 不会生效。需要先移除跟踪：

```bash
# 从 Git 跟踪移除（不删除文件）
git rm --cached file.txt

# 批量从跟踪中移除已忽略的文件
git rm --cached `git ls-files --cached --ignored --exclude-standard`
```

## 5.4 全局忽略

```bash
# 创建全局 .gitignore
git config --global core.excludesfile ~/.gitignore_global

# ~/.gitignore_global 示例
.DS_Store
Thumbs.db
*.swp
*~
```

# 6. Git 的三种合并策略

```bash
# 默认 merge（递归策略）
git merge feature

# 快进合并（fast-forward）
git merge --ff feature

# 禁止快进（生成 merge commit）
git merge --no-ff feature

# 压缩合并（squash）
git merge --squash feature
```

## 6.1 Fast-Forward 合并

```
main: A──B──C
feature:     C──D──E
结果: A──B──C──D──E（线性，无 merge commit）
```

条件：目标分支未产生新的提交，直接将 HEAD 前移。

## 6.2 Three-Way Merge

```
main: A──B──C──F
feature: A──B──D──E
结果: A──B──C──F──G──H（G 为 merge commit）
         \──D──E──/
```

当两个分支都有新提交时，Git 会找到共同祖先（B），结合两个分支的最新状态，进行三路合并。

## 6.3 Squash Merge

```bash
git checkout main
git merge --squash feature
git commit -m "feat: add feature as single commit"
```

将 feature 分支的所有提交压缩成一个提交，合入 main。适用于将开发分支的零碎提交整理为功能完整的单一提交。

# 7. Git 数据完整性保障

## 7.1 SHA-1 校验和

Git 中的所有数据在存储前都会计算 SHA-1 散列值（40位十六进制数）：

```
SHA-1 计算内容 = 对象类型 + 空格 + 内容长度 + 空字节 + 内容
```

```python
# 等效计算方式
import hashlib
content = b"what is up, doc?"
header = f"blob {len(content)}\0".encode()
hash = hashlib.sha1(header + content).hexdigest()
print(hash)  # bd9dbf5aae1a3862dd1526723246b20206e5fc37
```

## 7.2 验证数据完整性

```bash
# 验证对象完整性
git fsck
git fsck --full

# 检查仓库状态
git fsck --unreachable
git count-objects -v
```

# 8. 常见场景示例

## 8.1 日常工作流程

```bash
# 1. 拉取最新代码
git pull origin main

# 2. 创建功能分支
git checkout -b feature/user-auth

# 3. 开发过程中的迭代
git add -p src/login.js          # 交互式分块暂存
git commit -m "feat: add login form"
git add src/api/auth.js
git commit -m "feat: add auth API"

# 4. 同步主分支（避免长时间偏离）
git fetch origin
git rebase origin/main

# 5. 推送分支
git push origin feature/user-auth
```

## 8.2 紧急修复流程

```bash
# 1. 保存当前工作
git stash -u   # -u 包含未跟踪的文件

# 2. 创建修复分支
git checkout -b hotfix/critical-bug main

# 3. 修复并提交
git add fix.js
git commit -m "fix: critical production bug"

# 4. 部署修复
git push origin hotfix/critical-bug

# 5. 恢复原工作
git checkout feature/current-work
git stash pop
```

## 8.3 撤销操作

```bash
# 撤销工作目录的修改（危险！不可恢复）
git restore file.txt
git checkout -- file.txt  # 传统方式

# 从暂存区撤销
git restore --staged file.txt
git reset HEAD file.txt   # 传统方式

# 撤销所有未提交的修改
git restore .
git checkout -- .         # 传统方式

# 撤销最近一次提交（保留修改在暂存区）
git reset --soft HEAD~1

# 撤销最近一次提交（保留修改在工作目录）
git reset --mixed HEAD~1

# 撤销最近一次提交（彻底丢弃修改）
git reset --hard HEAD~1

# 恢复误删除的分支
git reflog
git checkout -b recovered-branch HEAD@{2}
```

# 9. Git 帮助系统

```bash
# 查看帮助
git help <command>
git <command> --help

# 打开 git 官方手册
git help git

# 列出所有可用命令
git help -a

# 列出所有配置选项
git help -c
```
