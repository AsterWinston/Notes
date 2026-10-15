
# 1. Git 性能瓶颈分析

## 1.1 影响 Git 性能的因素

| 因素 | 影响 | 百分比 |
|------|------|--------|
| 仓库历史大小 | clone/fetch/push 速度 | 60% |
| 文件数量 | status/log/diff 速度 | 20% |
| 文件大小 | 每次操作 IO 开销 | 10% |
| 网络延迟 | 远程操作 | 10% |

## 1.2 性能诊断

```bash
# 查看仓库大小
git count-objects -v
# size-pack: 524288    # pack 文件大小（KB）
# size-gc: 12345       # GC 节省的空间

# 查看大文件
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sort -t' ' -k3 -n -r | head -20

# 查看操作耗时
GIT_TRACE_PERFORMANCE=1 git status
GIT_TRACE_PERFORMANCE=1 git log --oneline -100
```

# 2. 浅克隆（Shallow Clone）

## 2.1 基本浅克隆

只克隆最近的 N 次提交，大幅减少数据传输量：

```bash
# 只克隆最近 1 次提交
git clone --depth 1 https://github.com/user/repo.git

# 克隆最近 5 次提交
git clone --depth 5 https://github.com/user/repo.git

# 浅克隆 + 指定分支
git clone --depth 1 --branch main https://github.com/user/repo.git
```

**性能对比**：

```yaml
# 例如 Linux 内核仓库
完整克隆: ~3GB, 需要 30 分钟
depth=1:  ~200MB, 需要 2 分钟
```

## 2.2 浅克隆的操作限制

```bash
# 浅克隆不能执行的操作
git log --all               # 只能看到浅层历史
git checkout old-tag        # 切换到较老的 tag
git merge remote/branch     # 可能失败（找不到共同祖先）

# 将浅克隆转换为完整克隆
git fetch --unshallow
git fetch --depth=1000000   # 加深深度
```

## 2.3 CI 环境中的浅克隆

```yaml
# GitHub Actions 默认使用浅克隆
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # 设置 0 表示完整克隆（如需 lint-staged 等需要完整历史）
    fetch-depth: 1  # 默认值，浅克隆
    fetch-depth: 50 # 根据需要设置
```

```yaml
# GitLab CI 配置
variables:
  GIT_DEPTH: "10"    # 限制 Git 深度
  GIT_STRATEGY: fetch  # clone/fetch
```

# 3. 稀疏检出（Sparse Checkout）

## 3.1 基本用法

只检出仓库中的部分目录/文件，减少工作目录大小：

```bash
# 方式 1：初始化时配置
git clone --filter=blob:none --sparse https://github.com/user/repo.git
cd repo
git sparse-checkout set src/api src/common

# 方式 2：已有仓库配置
git sparse-checkout init
git sparse-checkout set src/api

# 添加更多目录
git sparse-checkout add docs

# 列出已配置的目录
git sparse-checkout list

# 使用模式匹配
git sparse-checkout set 'src/*' '!src/test/*'

# 取消稀疏检出
git sparse-checkout disable
```

## 3.2 Partial Clone（部分克隆）

Git 2.19+ 支持的部分克隆，可以延迟下载 blob 对象：

```bash
# 不下载 blob（文件内容），按需获取
git clone --filter=blob:none https://github.com/user/repo.git

# 不下载 tree 和 blob，按需获取
git clone --filter=tree:0 https://github.com/user/repo.git

# 指定协议（推荐）
git clone --filter=blob:none --also-filter-submodules \
  https://github.com/user/repo.git
```

**`--filter` 类型对比**：

| Filter | 初始下载 | 可用操作 | 适用场景 |
|--------|---------|---------|---------|
| `blob:none` | 所有 tree + commits | commit log 正常 | CI/日常开发 |
| `tree:0` | 仅 commits | 基本 log | 仅需提交历史 |
| `blob:limit=<size>` | 小文件下载 | 小文件操作 | 大文件仓库 |

## 3.3 组合使用

```bash
# 最优化克隆：浅克隆 + 部分克隆 + 稀疏检出
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/user/repo.git
cd repo
git sparse-checkout set src/app
```

# 4. Git LFS（大文件存储）

## 4.1 工作原理

Git LFS 将大文件替换为指针文件，实际内容存储在远程 LFS 服务器：

```
仓库中存储（指针文件）：
version https://git-lfs.github.com/spec/v1
oid sha256:abc123def456...
size 52428800

LFS 服务器存储（实际文件）：
/objects/ab/c123def456... (实际大文件)
```

## 4.2 安装与配置

```bash
# 安装 LFS
git lfs install  # 全局安装一次

# 在仓库中跟踪大文件
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.tar.gz"
git lfs track "*.mp4"

# 查看跟踪规则
git lfs track
# *.psd filter=lfs diff=lfs merge=lfs -text

# .gitattributes 文件会被自动创建
# 需要提交 .gitattributes
git add .gitattributes
git commit -m "chore: configure LFS for large files"
```

## 4.3 LFS 日常操作

```bash
# 正常使用（与普通 Git 一样）
git add design.psd
git commit -m "feat: add design mockup"
git push origin main  # LFS 文件自动上传到 LFS 服务器

# 查看 LFS 状态
git lfs status
git lfs ls-files                       # 列出 LFS 管理的文件
git lfs ls-files --all                 # 包括历史版本

# 拉取 LFS 文件
git lfs pull
git lfs fetch --all                    # 拉取所有 LFS 文件（包括历史）

# 查看 LFS 文件使用情况
git lfs env                            # LFS 环境信息
git lfs track                          # 跟踪规则
```

## 4.4 迁移现有文件到 LFS

```bash
# 将仓库中已有的 .psd 文件迁移到 LFS
git lfs migrate import --include="*.psd" --everything

# 迁移并只保留最近 N 次提交的文件（减小仓库）
git lfs migrate import --include="*.psd" --everything --above=1MB

# 查看迁移结果
git lfs migrate info --everything

# 注意：migrate 会重写历史！需要团队协调
# 迁移后需要 force push
git push --force origin main
```

## 4.5 LFS 配额与成本

```yaml
GitHub LFS:
  免费: 1GB 存储, 1GB/月带宽
  付费: $5/月 50GB, 更多档位

GitLab LFS:
  免费: 5GB 存储
  付费: 取决于套餐

自建 LFS 服务器: 无限制
```

# 5. GC 与仓库维护

## 5.1 Git GC 原理

```bash
# 自动 GC（Git 在满足条件时自动触发）
git gc --auto

# 手动触发完整 GC
git gc

# 激进 GC（更慢但压缩更多）
git gc --aggressive

# Git 自动触发 GC 的条件：
# - loose 对象数量超过 6700
# - 有多个 pack 文件且 loose 对象超过 50
```

## 5.2 手动仓库优化

```bash
# 1. 压缩仓库（等同于 git gc --aggressive）
git repack -a -d --depth=250 --window=250

# 2. 修剪对象
git prune

# 3. 清理 reflog
git reflog expire --expire=now --all

# 4. 清理多余的远程跟踪引用
git remote prune origin
git fetch --prune

# 5. 优化仓库大小
git count-objects -v
```

## 5.3 自动 GC 配置

```bash
# GC 相关配置
git config gc.auto 6700                   # 触发自动 GC 的 loose 对象数
git config gc.autoPackLimit 50            # 触发自动 GC 的 pack 文件数
git config gc.bigPackThreshold 2g         # 大 pack 文件阈值
git config gc.reflogExpire 90.days        # reflog 保留时间
git config gc.reflogExpireUnreachable 30.days  # 不可达 reflog 保留
```

# 6. 大仓库优化策略

## 6.1 分层存储策略

```yaml
# 将不同频率修改的文件分开管理
高频（~50MB）: src/        # 频繁修改，需要完整历史
低频（~50MB）: docs/       # 偶尔修改
静态（~500MB）: assets/     # 大文件，应使用 LFS
```

## 6.2 使用 Git 子系统

```bash
# 将大目录提取为独立仓库
git filter-branch --subdirectory-filter assets/ -- --all

# 或者使用 subtree 分拆
git subtree split --prefix=assets/ -b assets-only
```

## 6.3 移除大文件（重写历史）

```bash
# 方法 1：使用 filter-branch
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch large-file.zip" \
  --prune-empty --tag-name-filter cat -- --all

# 方法 2：使用 BFG Repo-Cleaner（更快）
java -jar bfg.jar --strip-blobs-bigger-than 100M repo.git
cd repo.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# 之后需要 force push
git push --force --all origin
git push --force --tags origin
```

# 7. 配置文件优化

```bash
# 推荐的大仓库 Git 配置
git config core.preloadindex true          # 并行加载 index
git config core.fscache true               # 文件系统缓存（Windows）
git config gc.writeCommitGraph true        # 编写 commit graph
git config fetch.writeCommitGraph true     # fetch 时编写 commit graph

# 关闭 Git 的安全检查（仅在安全和性能之间权衡时使用）
git config core.protectNTFS false          # 关闭 NTFS 保护（Windows）
git config core.longpaths true             # 支持长路径（Windows）
```

# 8. 性能基准测试

```bash
# 创建性能基准
hyperfine 'git status' 'git diff' 'git log --oneline -10'
hyperfine 'git log --oneline --graph --all' 'git log --oneline -1000'

# 测试 clone 速度
time git clone --depth 1 https://github.com/user/repo.git
time git clone https://github.com/user/repo.git
```
