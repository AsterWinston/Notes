
# 1. Git 核心对象类型

Git 是一个**内容寻址文件系统**，其核心是四种对象类型：

| 对象类型 | 用途 | 存储内容 |
|---------|------|----------|
| **blob** | 存储文件内容 | 文件数据（二进制） |
| **tree** | 存储目录结构 | 文件名、权限、指向 blob/tree 的指针 |
| **commit** | 存储一次提交快照 | tree hash、parent hash、作者、提交信息 |
| **tag** | 为特定提交命名 | 对象引用、标签信息（annotated tag） |

## 1.1 对象存储格式

所有对象都按以下格式存储：

```
对象类型 + 空格 + 内容长度 + 空字符 + 原始内容
```

```python
# 以 blob 为例
raw_content = b"hello\n"
store_content = b"blob 6\0" + raw_content
# SHA-1 = ce013625030ba8dba906f756967f9e9ca3944648
```

```bash
# 手动验证
echo "hello" | git hash-object --stdin
# ce013625030ba8dba906f756967f9e9ca3944648
```

## 1.2 .git 目录结构

```
.git/
├── HEAD                   # 当前分支引用
├── config                 # 仓库配置
├── description            # 仓库描述
├── index                  # 暂存区文件
├── objects/               # 对象存储
│   ├── 00/               # SHA-1 前缀目录
│   │   └── abc123...     # 对象文件（zlib 压缩）
│   ├── info/
│   │   └── packs         # 包文件索引
│   └── pack/             # pack 文件
│       ├── pack-xxx.pack
│       └── pack-xxx.idx
├── refs/                  # 引用
│   ├── heads/            # 本地分支
│   │   └── main
│   ├── tags/             # 标签
│   │   └── v1.0.0
│   └── remotes/          # 远程分支
│       └── origin/
│           └── main
├── logs/                  # reflog
│   ├── HEAD
│   └── refs/
└── hooks/                # 钩子脚本
    ├── pre-commit.sample
    ├── commit-msg.sample
    └── ...
```

# 2. Blob 对象

Blob 是 Git 中最基础的对象，用于存储文件内容。Blob 只存储内容，不存储文件名。

## 2.1 创建与查看 Blob

```bash
# 创建 blob 对象（写入 .git/objects）
echo "hello world" | git hash-object -w --stdin
# 3b18e512dba79e4c8300dd08aeb37f8e728b8dad

# 仅计算 hash（不写入）
echo "hello world" | git hash-object --stdin

# 从文件创建 blob
git hash-object -w file.txt

# 查看 blob 内容
git cat-file -p 3b18e51    # 查看内容
git cat-file -t 3b18e51    # 查看类型（blob）
git cat-file -s 3b18e51    # 查看大小
```

## 2.2 Blob 的特性

- **内容去重**：相同内容的文件共享同一个 blob（无论文件名是否相同）
- **不可变**：一旦创建，内容不可更改（更改会产生新的 hash）
- **不存文件名**：文件名存储在 tree 对象中

```
文件 "a.txt" (内容 "hello") ──→ blob(ce0136)
文件 "b.txt" (内容 "hello") ──→ blob(ce0136)  ← 相同 hash，共享对象
文件 "a.txt" (内容 "world") ──→ blob(f75ba0)  ← 内容不同，不同 hash
```

# 3. Tree 对象

Tree 对象对应文件系统的目录，记录了文件名、文件权限和指向 blob 或子 tree 的引用。

## 3.1 Tree 内部格式

```
<mode> <name>\0<SHA-1 二进制>
```

```
100644 hello.c\0<20字节 SHA-1>
100755 run.sh\0<20字节 SHA-1>
40000 lib\0<20字节 SHA-1>
```

## 3.2 权限模式

| 模式 | 含义 |
|------|------|
| `100644` | 普通文件（大部分文件） |
| `100755` | 可执行文件 |
| `040000` | 子目录（tree） |
| `120000` | 符号链接 |
| `160000` | gitlink（submodule） |

## 3.3 查看 Tree

```bash
# 查看最新 commit 的 tree
git cat-file -p HEAD^{tree}
# 或
git ls-tree HEAD

# 查看特定 tree
git cat-file -p <tree-hash>

# 递归查看
git ls-tree -r HEAD
git ls-tree -r -t HEAD  # 包含 tree 自身
```

## 3.4 完整的对象关系

```
commit (abc123)
│
│  tree: def456
│  parent: 789abc
│  author: John
│  message: "initial commit"
│
▼
tree (def456)
│
├── 100644 README.md → blob(111aaa)
├── 100755 start.sh  → blob(222bbb)
├── 040000 src/      → tree(333ccc)        # 子目录
│   ├── 100644 main.js → blob(444ddd)
│   └── 100644 utils.js → blob(555eee)
└── 160000 lib/       → submodule-hash      # 子模块
```

# 4. Commit 对象

Commit 对象是对项目在某一时刻的快照（tree）的封装，包含元数据。

## 4.1 Commit 内部结构

```
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
parent c6c5e4c0b8c0efc1b9b8e4f0e8f0e0a0b0c0d0e0  # 普通提交
author John <john@example.com> 1712345678 +0800
committer John <john@example.com> 1712345678 +0800

feat: add user authentication

- Add login form component
- Add auth API integration
- Add session management
```

```bash
# 查看 commit 原始内容
git cat-file -p HEAD

# 查看 commit 元数据
git log -1 --format=raw HEAD

# 只查看 tree hash
git rev-parse HEAD^{tree}
```

## 4.2 首次提交 vs 普通提交 vs 合并提交

```bash
# 首次提交（无 parent）
git log --format=raw --max-parents=0 HEAD

# 普通提交（1个 parent）
git log --format=raw HEAD

# 合并提交（2个 parent）
git log --format=raw -1 --min-parents=2 HEAD
```

```
# 首次提交
tree a1b2c3...
(parent 不存在)

# 普通提交
tree d4e5f6...
parent a1b2c3...

# 合并提交
tree g7h8i9...
parent d4e5f6...
parent x9y0z1...
```

## 4.3 Commit 的不可变性

Commit 对象也是不可变的。修改历史的本质是创建新的 commit 对象，将分支指针指向新的 commit：

```bash
# 修改最近一次 commit 信息（实际创建新 commit，抛弃旧的）
git commit --amend -m "new message"
# 旧的 commit 变为 dangling object
```

# 5. Tag 对象

## 5.1 轻量标签 vs 附注标签

```bash
# 轻量标签（只是一个引用指针）
git tag v1.0.0
# 存储位置：.git/refs/tags/v1.0.0
# 内容：指向 commit 的 hash

# 附注标签（完整的 Git 对象）
git tag -a v1.0.0 -m "Release v1.0.0"
# 存储位置：.git/objects/xx/xxxx...
# 包含：tagger、日期、消息、可签名
```

```bash
# 查看 tag 对象
git cat-file -p v1.0.0     # 轻量 tag → commit 内容
git cat-file -t v1.0.0     # 轻量 tag → commit

# 查看 annotated tag
git cat-file -p v1.0.0     # 显示 tag 对象内容
# object c6c5e4c...        # 指向的 commit
# type commit
# tag v1.0.0
# tagger John <...>
# Release v1.0.0
```

## 5.2 签名标签

```bash
# GPG 签名标签
git tag -s v1.0.0 -m "Release v1.0.0"

# 验证签名
git tag -v v1.0.0
```

# 6. 对象存储与压缩

## 6.1 Loose 对象存储

新创建的对象以 loose 格式存储：
- 路径：`objects/XX/YYYYYY...`（XX = hash 前两个字符）
- 内容：zlib 压缩后的对象数据

```bash
# 查看 loose 对象
# .git/objects/ce/013625030ba8dba906f756967f9e9ca3944648
ls -la .git/objects/ce/
# 使用 zlib 解压可还原内容
```

## 6.2 Pack 文件

当 loose 对象积累到一定数量，Git 会自动执行 GC（垃圾回收），将多个对象打包成一个 pack 文件：

```bash
git gc                    # 手动触发 GC
git gc --aggressive       # 更激进的压缩（更小体积，更慢速度）
git gc --auto             # 自动模式（仅在需要时执行）
```

**Pack 文件格式**：

```
.pack 文件：存储实际对象数据（使用增量压缩）
.idx  文件：索引，快速定位对象在 .pack 中的位置
```

**Delta 压缩**：
Git 在打包时会识别相似的对象，只存储差异部分：

```
Objects:
  v1: "hello world"         ← base object
  v2: "hello world!!!"     ← delta from v1 (只存 "+!!!" 的差异)
  v3: "hello world!!!"     ← 引用 v2 (指向相同的)

pack 文件：
  base object: "hello world"
  delta 1: "+!!!" (偏移量 11, 3 字节)
```

```bash
# 查看 pack 文件信息
git count-objects -v
git verify-pack .git/objects/pack/*.idx

# 查看对象在 pack 中的分布
git verify-pack .git/objects/pack/*.idx | sort -k3 -n | tail -20
```

# 7. 引用（References）

## 7.1 引用类型

| 引用 | 位置 | 说明 |
|------|------|------|
| `HEAD` | `.git/HEAD` | 当前检出分支/提交 |
| `refs/heads/*` | `.git/refs/heads/main` | 本地分支 |
| `refs/remotes/*` | `.git/refs/remotes/origin/main` | 远程跟踪分支 |
| `refs/tags/*` | `.git/refs/tags/v1.0` | 标签 |
| `refs/stash` | `.git/refs/stash` | 存储栈 |

## 7.2 HEAD 的两种状态

```bash
# 1. 指向分支（正常状态）
cat .git/HEAD
# ref: refs/heads/main

# 2. 直接指向提交（detached HEAD）
cat .git/HEAD
# c6c5e4c0b8c0efc1b9b8e4f0e8f0e0a0b0c0d0e0
```

## 7.3 符号引用

```bash
# 查看符号引用
git symbolic-ref HEAD
# refs/heads/main

# 设置符号引用（改变当前分支）
git symbolic-ref HEAD refs/heads/feature
```

# 8. Reflog（引用日志）

Reflog 记录了 HEAD 和分支引用的所有变化（是 Git 数据恢复的最后保障）。

## 8.1 Reflog 的工作原理

```
.git/logs/HEAD 内容格式：
<old-hash> <new-hash> <committer> <timestamp> <action>
```

```bash
# 查看 HEAD 的 reflog
git reflog
# 等价于
git log --oneline -g

# 查看特定分支的 reflog
git reflog show main

# 查看所有 reflog
git reflog show --all
```

## 8.2 Reflog 使用场景

```bash
# 恢复到误操作之前的状态
git reflog
# abc123 HEAD@{0}: commit: fix bug
# def456 HEAD@{1}: commit: add feature
# 789abc HEAD@{2}: reset: moving to HEAD~1
git reset --hard HEAD@{2}   # 恢复到 HEAD@{2} 的状态

# 恢复误删的分支
git branch recovered-branch HEAD@{5}

# 查看某个时间点的状态
git show HEAD@{3.days.ago}
git show HEAD@{yesterday}
git show HEAD@{2.hours.ago}
```

## 8.3 Reflog 过期策略

```bash
# 默认保留 90 天
git config gc.reflogExpire 90.days
git config gc.reflogExpireUnreachable 30.days

# 手动清理 reflog
git reflog expire --expire=30.days --all
```

# 9. 对象搜索与检查

```bash
# 在仓库中搜索对象
git rev-list --objects --all                  # 列出所有对象
git rev-list --objects --all | grep file.txt  # 查找某个文件的对象

# 查看对象所属的 commit
git describe --always <object-hash>

# 查找指定对象的祖先
git rev-list --parents --ancestry-path <commit>

# 查看哪些文件占用最多空间
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sed -n 's/^blob //p' | sort --numeric-sort --key=2 | tail -10
```

# 10. Git 对象模型的设计哲学

## 10.1 为什么 Git 设计为内容寻址？

- **数据完整性**：任何内容变化都会产生不同的 hash，篡改可被立即发现
- **去重存储**：相同内容只存一次，节省空间
- **不可变性**：对象不可变，简化了并发和备份
- **分布式友好**：内容寻址意味着不需要中心化的命名空间

## 10.2 为什么不直接存储文件差异？

传统的 VCS（如 SVN）存储文件差异（delta）：

```
SVN: v1 → v2(delta) → v3(delta) → ...
    （查看 v3 需要：v1 + delta1 + delta2）
```

Git 存储完整快照 + 增量压缩：

```
Git: v1(full) v2(full) v3(full) → pack(delta compression)
    （本地操作极快，pack 时做 delta 压缩节省空间）
```

这种设计的优势：
- 本地操作 O(1) 而非 O(n)
- 分支切换无需计算差异
- 数据恢复更可靠

## 10.3 SHA-1 冲突的可能性

理论上 SHA-1 有碰撞可能，但实际上概率极低：

- 地球上沙子总数：约 7.5 × 10^18
- SHA-1 空间：2^160 ≈ 1.46 × 10^48
- 需要约 2^51 次操作才可能找到碰撞

Git 已在迁移至 SHA-256（Git 2.29+ 支持），通过 `git config core.repositoryFormatVersion 1`
