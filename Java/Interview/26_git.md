# 1. 知识点
- **工作区(Working Directory)**：本地文件系统中 Git 管理的目录，开发者直接编辑文件的地方
- **暂存区(Stage/Index)**：`git add` 后文件到达的中间区，记录即将提交的变更清单
- **仓库区(.git目录)**：`git commit` 后数据永久保存到 Git 本地仓库数据库中
- **Untracked**：新创建的文件未被 Git 跟踪
- **Modified**：被跟踪的文件内容已修改但未 add
- **Staged**：修改已 add 到暂存区，等待 commit
- **Committed**：修改已 commit 到本地仓库
- **git add**：将文件从工作区移动到暂存区
- **git commit**：将暂存区的更改提交到本地仓库，生成一个新的 commit 对象
- **git reset HEAD^**：撤销最近一次 commit，变更回退到工作区（--mixed 默认模式），不影响工作区文件。HEAD^ 表示上一个commit，HEAD~n 表示上 n 个
- **git log --oneline**：简洁显示提交历史，每行一个 commit
- **git reflog**：记录所有 HEAD 操作历史（包括已删除的 commit、rebase 记录），可找回误删提交
- **git revert**：创建一个新的反向 commit 来撤销指定 commit 的变更，不会修改历史，适合已推送到远程的 commit
- **git reset**：重置当前分支的 HEAD 到指定提交，修改本地历史。三种模式：--soft 只回退 commit 不改暂存区和工作区、--mixed 回退 commit 和暂存区不改工作区、--hard 全部丢弃回退到干净状态
- **git diff**：比较文件差异。不加参数比较工作区和暂存区差异；--staged 比较暂存区和最后一个 commit；HEAD 比较工作区和最新 commit
- **git stash**：临时保存工作区和暂存区未提交的修改，使工作目录变干净，适合紧急切换分支
- **git stash pop**：恢复最近一次 stash 的内容并删除该 stash 记录
- **git stash list**：查看所有 stash 记录
- **git stash apply**：恢复 stash 内容但不删除 stash 记录
- **git rm**：从版本控制中删除文件，可配合 --cached 只删除跟踪不删本地文件
- **git remote**：管理远程仓库，git remote -v 查看远程仓库地址，git remote add 添加远程仓库
- **git branch**：操作分支，不带参数列出所有本地分支，-a 列出包括远程分支，-d 删除分支
- **git checkout -b**：新建分支并立即切换到新分支
- **git switch**：Git 2.23+ 引入的新命令，专门用于切换分支，git switch -c 创建并切换
- **git merge**：将指定分支合并到当前分支
- **fast-forward合并**：默认行为，当被 merge 的分支直接领先当前分支时（无分叉），Git 只是将 HEAD 指针快速移动到被合并分支的最新提交，不产生 merge commit
- **--no-ff**：禁止快进合并，即使可以快进也强制创建一个 merge commit，保留分支历史痕迹
- **merge vs rebase**：merge 保留完整合并历史但分支历史可能繁杂；rebase 将提交变基到目标分支上生成线性历史更干净，但改写了提交历史不可推送到共享分支
- **merge冲突解决**：同时修改同一文件可能产生冲突，手动编辑冲突文件 → `git add` 标记已解决 → `git commit` 完成合并
- **rebase冲突**：在 rebase 过程中遇到冲突，手动解决冲突 → `git add` → `git rebase --continue`；放弃用 `git rebase --abort`
- **git fetch**：从远程仓库拉取最新代码到本地，但不自动合并到工作区
- **git pull**：相当于 git fetch + git merge，拉取远程代码并合并到本地当前分支。--rebase 拉取后使用 rebase 方式合并
- **git push**：将本地提交推送到远程仓库
- **upstream跟踪**：本地分支与远程分支建立跟踪关系，之后可以简单使用 git push/pull 不加参数
- **PR(Pull Request)**：GitHub/GitLab 的协作工具，开发者在自己的分支开发完毕后发起 PR 请求合并到目标分支，团队成员 Code Review 后合并
- **Code Review**：通过 PR 进行代码审查，对变更进行评审、讨论、提出修改意见，提高代码质量
- **Git Flow**：经典的 Git 分支管理模型：主分支 main/master(正式发布)、develop(集成开发)、feature(特性开发)、release(发布准备)、hotfix(紧急修复)
- **main/master分支**：Git Flow 中的主分支，记录正式发布历史，每次发布打 tag
- **develop分支**：开发主集成分支，所有 feature 分支合并到 develop
- **feature分支**：从 develop 分出，开发新特性，完成后合并回 develop 并删除
- **release分支**：从 develop 分出，准备发布，修复小 bug、更新版本号，完成后合并回 main 和 develop
- **hotfix分支**：从 main 分出，紧急修复线上 bug，完成后合并回 main 和 develop，并打新版本 tag
- **git reset --soft**：回退 commit，保留暂存区和工作区，变更还在暂存区
- **git reset --mixed**（默认）：回退 commit 和暂存区，但保留工作区修改，变为 Unstaged
- **git reset --hard**：回退 commit、暂存区和工作区，所有变更完全丢弃，可能丢失数据（可通过 reflog 找回）
- **git checkout -- file**：丢弃工作区某个文件的修改
- **git restore**：Git 2.23+ 新命令，取代 checkout 的文件恢复功能，git restore --staged 取消暂存
- **git tag**：给 commit 打标签，轻量标签仅记录名称，附注标签(annotated tag)记录 tagger/date/message
- **git push --tags**：将本地所有标签推送到远程
- **附注标签(Annotated Tag)**：完整对象存储在 Git 数据库中，包含打标签的人/日期/说明信息，可 GPG 签名
- **Git内部对象**：Git 核心是内容寻址的文件系统，存储四种对象：blob(文件内容)、tree(目录结构)、commit(提交指针)、tag(标签)
- **内容寻址存储**：Git 通过 SHA-1 哈希值标识和寻址所有对象，相同内容有相同哈希
- **快照而非差异**：Git 对每个 commit 存储完整快照(snapshot)，而非文件差异(diff)。相同内容复用已有 blob
- **.gitignore**：指定 Git 忽略的文件和目录模式，如 `*.class` `target/` `.idea/` `node_modules/`
- **CRLF/LF 换行冲突**：Windows 使用 CRLF(\r\n)，Linux/Mac 使用 LF(\n)，Git 可通过 core.autocrlf 配置自动转换
- **core.autocrlf**：true(提交时转LF/检出时转CRLF)、input(提交时转LF/检出时不转)、false(不做任何转换)
- **core.safecrlf**：warn(警告但允许)、true(禁止包含混合换行的文件) 防止不可逆转的转换
- **敏感信息清除**：如果误提交密码/密钥，用 `git filter-branch` 或 BFG Repo-Cleaner 彻底从历史中删除
- **BFG Repo-Cleaner**：比 filter-branch 更简单快速的大文件/敏感信息清除工具
- **submodule**：将一个 Git 仓库作为另一个仓库的子目录管理，但子仓库独立拉取
- **subtree**：将外部仓库合并为当前仓库的一个子目录（完全融合）
- **monorepo**：单一代码库管理多项目，不需要 submodule

# 2. 面试题

## 2.1 Git 的工作区、暂存区和版本库分别是什么？
- **工作区(Working Directory)**：你本地能看到的开发目录，直接编辑文件的地方
- **暂存区(Stage/Index)**：Git 的中间层，在 `.git/index` 文件中。通过 `git add` 将文件改动放入暂存区，暂存区可以让你精确控制要提交的内容（可以有选择地 add 文件，而不是一次提交所有修改）
- **版本库(Repository)**：`.git` 目录，保存所有 commit 记录和 Git 对象数据库。通过 `git commit` 将暂存区内容保存到版本库

三者的流转：工作区 →(git add)→ 暂存区 →(git commit)→ 版本库

## 2.2 git merge 和 git rebase 有什么区别？使用场景是什么？
| 维度 | merge | rebase |
|------|-------|--------|
| 原理 | 创建一个新的 merge commit 整合两条分支 | 将当前分支的 commit 逐个"重放"到目标分支顶端 |
| 历史 | 保留真实的分支合并历史，但可能有较多 merge commit | 产生线性的更干净的提交历史 |
| 冲突 | 只需解决一次冲突 | 每个 commit 重放时可能都需要解决冲突 |
| commit | 不改写已有 commit | 重置 commit SHA，改写历史 |
| 远程推送 | 安全可推 | 已推送的 commit rebase 后不能再次推送（历史改变） |

**使用场景**：
- **merge**：给公共分支（main/develop）合并 feature 分支时使用，保留完整的协作历史
- **rebase**：个人 feature 分支同步 main 的最新变更时使用，保持提交历史线性干净。Golden Rule：**不要对已推送到公共仓库的 commit 做 rebase**

## 2.3 git reset 和 git revert 有什么区别？
- **git revert**：生成一个新的 commit 来"反向操作"要撤销的 commit，原有历史不会被删除。是安全的撤销方式，适合已经推送到远程的 commit
  - 例：git revert HEAD（生成一个新 commit 撤销上一个 commit 的变更）
- **git reset**：将 HEAD 指针移动到指定 commit，后续 commit 从历史中"移除"。会修改历史，适合撤销本地未推送的 commit
  - --soft：只移动 HEAD，暂存区和工作区保持不变
  - --mixed（默认）：移动 HEAD 并清空暂存区，工作区保持不变
  - --hard：移动 HEAD 并清空暂存区和工作区，所有未提交变更丢失

## 2.4 git reset 的三种模式有什么区别？
- **--soft**：只移动 HEAD 指针到目标 commit。暂存区和工作区不变，commit 变更回到暂存区（绿色）。适用场景：撤销 commit 但保留所有修改暂存状态
- **--mixed**（默认）：移动 HEAD 并重置暂存区（与目标 commit 的时刻一致）。工作区不变，commit 变更从暂存区回到工作区（红色）。适用场景：撤销 commit 和 add，但保留文件修改
- **--hard**：移动 HEAD 并重置暂存区和工作区。所有未提交的变更全部丢失。适用场景：彻底废弃 commit 和所有变更、丢弃当前所有修改恢复到干净状态

## 2.5 git pull 和 git fetch 有什么区别？
- **git fetch**：从远程仓库下载最新的提交和引用（分支、tag），但**不会合并**到当前本地工作分支，需要手动 git merge/git rebase。适合先看有什么更新，再决定如何处理
- **git pull**：git fetch + git merge 或 git fetch + git rebase（加 --rebase 参数）。直接从远程拉取代码并合并到当前分支，一步到位。特点是可能有合并冲突且无法先观察再决定

建议：团队协作中优先用 git fetch + git merge（或 rebase）两步走，更具控制力。

## 2.6 如何解决 Git 合并冲突？
1. 当 merge/rebase 产生冲突时 Git 会指出冲突文件
2. 打开冲突文件，找到冲突标记：
   ```
   <<<<<<< HEAD (当前分支)
   你的修改
   =======
   被合并分支的修改
   >>>>>>> feature-branch
   ```
3. 手动编辑决定保留哪些内容，删除冲突标记
4. `git add <conflict-file>` 标记冲突已解决
5. `git commit` 完成合并（rebase 冲突用 `git rebase --continue`）
6. 如果要放弃合并：`git merge --abort`，rebase 冲突放弃：`git rebase --abort`

## 2.7 Git Flow 工作流包含哪几种分支？各自职责是什么？
| 分支 | 来源 | 合并到 | 说明 |
|------|------|-------|------|
| main/master | - | - | 生产可用代码，不可直接提交，每次发布打 tag |
| develop | - | - | 开发集成分支，所有 feature 合并到这里 |
| feature | develop | develop | 开发新功能，完成后合并并删除 |
| release | develop | main + develop | 发布前准备（修 bug、改版本号、文档） |
| hotfix | main | main + develop | 紧急修复线上 bug，从 main 分出，合回双方并 tag |

## 2.8 .gitignore 有什么作用？如何编写规则？
`.gitignore` 文件告诉 Git 忽略特定的文件或文件夹，不将其纳入版本控制。常见配置：
```
# 编译输出
target/
build/
*.class
*.jar

# IDE 配置
.idea/
*.iml
.vscode/
.settings/

# 系统文件
.DS_Store
Thumbs.db

# 依赖
node_modules/
vendor/

# 日志与临时文件
*.log
*.tmp
*.swp

# 环境变量文件
.env
.env.local
```

**规则语法**：
- `*` 匹配任意字符（`*.class` 匹配所有 .class 文件）
- `!` 取反排除（`!important.class` 不忽略该文件）
- `/` 开头只在根目录匹配（`/target`，不匹配子目录中的 target）
- `#` 开头的行是注释
- 目录以 `/` 结尾（`node_modules/`）

如果文件已被跟踪（tracked），需要先 `git rm --cached <file>` 使其不再被跟踪，再配合 .gitignore 生效。

## 2.9 如何处理 Windows 和 Linux 的换行符冲突（CRLF/LF）？
Git 提供了 `core.autocrlf` 配置来处理：
```bash
# Windows 开发者推荐：提交时转 LF，检出到工作区时转 CRLF
git config --global core.autocrlf true

# Linux/Mac 开发者推荐：提交时转 LF，检出时不转换
git config --global core.autocrlf input

# 不做任何转换（不推荐）
git config --global core.autocrlf false
```

最佳实践：在项目根目录添加 `.gitattributes` 文件：
```
* text=auto
*.bat text eol=crlf
*.sh text eol=lf
```
`* text=auto` 让 Git 自动判断文件是否为文本，自动规范化换行。

## 2.10 如何从 Git 历史中删除误提交的敏感信息？
**BFG Repo-Cleaner（推荐）**：
```bash
# 删除包含密码的文件
bfg --delete-files passwords.txt

# 替换文件中所有匹配的文本为 ***REMOVED***
bfg --replace-text passwords.txt

# 清理历史并回收空间
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push --force --all
```
然后：所有团队成员的本地仓库必须删除并重新 clone！历史 SHA 已全部改变。

**传统 `git filter-branch` 方式**（灰常复杂已不推荐，推荐使用 BFG 或 git filter-repo）：
```bash
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secret.txt' \
  --prune-empty --tag-name-filter cat -- --all
```

## 2.11 Git 的内部存储原理是什么？四种对象是什么？
Git 是一种内容寻址文件系统，用 SHA-1 哈希值标识所有内容。四种核心对象：
1. **blob（二进制大对象）**：存储文件内容（只存内容二进制，不含文件名和路径）。相同内容只存一份
2. **tree（树对象）**：相当于目录，记录文件名、文件 blob 引用、子目录 tree 引用，由指针组成
3. **commit（提交对象）**：指向顶层 tree（项目根目录）、包含父 commit 指针、作者/提交者/时间戳/提交信息
4. **tag（标签对象）**：指向前三种对象的指针，附注标签(release)额外包含 tagger/日期/留言/可签名

Git 每个 commit 存储完整快照而非差分(diff)：
- commit → 顶层 tree 快照
- tree → blob(文件) + tree(子目录)
- 相同内容的文件复用同一个 blob（因为 SHA-1 值相同，自动去重）

## 2.12 git stash 的使用场景和常用命令？
当正在开发功能但工作区未完成，需要紧急切换分支处理 bug 或做其他操作时，使用 stash 暂存当前工作进度。

```bash
git stash                    # 暂存当前所有修改（包括工作区和暂存区）
git stash save "描述信息"     # 暂存并添加描述
git stash list               # 查看所有 stash 列表
git stash show -p stash@{0}  # 查看 stash 的详细内容
git stash pop                # 恢复最近一个 stash 并删除该 stash
git stash apply stash@{0}    # 恢复指定 stash 但不删除
git stash drop stash@{0}     # 删除指定 stash
git stash branch <分支名>    # 从 stash 创建一个新分支并恢复到该分支
```

## 2.13 git tag 有哪些类型？如何创建和推送？
两种类型：
1. **轻量标签（Lightweight Tag）**：只是一个 commit 的引用指针，不存储额外信息
2. **附注标签（Annotated Tag）**：完整 Git 对象，包含 tagger/日期/message，可以 GPG 签名

```bash
# 轻量标签（不推荐生产发布使用）
git tag v1.0.0

# 附注标签（推荐用于版本发布）
git tag -a v1.0.0 -m "Release version 1.0.0"

# 给历史 commit 打标签
git tag -a v0.9.0 <commit-sha> -m "Release version 0.9.0"

# 推送标签
git push origin v1.0.0          # 推送单个标签
git push origin --tags          # 推送所有本地标签
git push --follow-tags          # 推送时同时推送关联的附注标签

# 删除标签
git tag -d v1.0.0               # 删除本地标签
git push origin --delete v1.0.0 # 删除远程标签
```
