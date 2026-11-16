本篇覆盖 Git 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：模型与底层、日常操作与撤销、分支模型与协作、工程化、故障与追问。分支发布与流水线的配合见 33_cicd.md，Maven 多模块与仓库结构见 3_maven.md，代码评审的方法论见 29_design_patterns.md 之外的团队规范（评审本身属流程题，本篇只讲工具语义）。

# 1. 模型与底层

## 1.1 三区与状态

### 1.1.1 工作区、暂存区、版本库怎么流转？为什么要有暂存区？

**三区模型（Git 的一切操作都能归到"在三个区之间搬内容"）**：

```text
工作区（Working Tree）        你编辑的文件
    │ git add                    ↑ git restore（丢弃工作区改动）
    ▼                            │
暂存区（Index / .git/index）   下一次提交的"内容清单"
    │ git commit                 ↑ git restore --staged / git reset
    ▼                            │
版本库（.git 对象库 + refs）    提交历史（commit/tree/blob）
```

**暂存区的存在理由（这题的分水岭，答"中间层"不算懂）**：
1. **精确构造提交**：一次改动十个文件，只提交其中三个相关的——`git add` 逐个挑，**提交粒度=逻辑粒度**，代码评审与回滚都受益；`git add -p` 还能**按 hunk 部分暂存**（同一文件里只提交一半改动）；
2. **提交前预演**：`git diff --cached` 看"我到底要提交什么"，与工作区未暂存的改动分离——**review 自己的提交**是专业习惯；
3. **冲突与合并的工作台**：merge/rebase 的冲突解决结果先放暂存区，`git add` 是"标记已解决"的语义（2.2.2）；
4. **性能与实现**：index 是路径→blob+元数据的**扁平缓存**（不是目录树），`git status` 靠它快速判断变更——也是为什么"文件被跟踪后 .gitignore 不生效"（4.1.1）。

**一句话收口**："暂存区是 Git 与 SVN 最大的体验差异——**SVN 是'改什么提交什么'，Git 是'选什么提交什么'**；它把'工作状态'与'提交内容'解耦，才有了精心构造历史的能力。"

**易错**：`git commit -a` 跳过暂存（自动 add 已跟踪文件）很方便，但**新文件不会被带上**；`git commit <file>` 直接提交指定文件（绕过暂存区语义，容易与"已暂存的其他内容"混在一起）；以为 `git add` 是"保存"（它只是登记内容，文件内容已在对象库但未形成提交）。

### 1.1.2 文件有哪几种状态？git diff 的三个视角分别比较什么？

**四种状态（`git status` 的输出语义）**：

| 状态 | 含义 | 典型提示 |
|------|------|---------|
| **Untracked** | 新文件，从未被 Git 记录 | `Untracked files`（`git add` 后进入暂存） |
| **Modified** | 已跟踪且工作区有改动，未暂存 | `Changes not staged for commit` |
| **Staged** | 改动已进暂存区，待提交 | `Changes to be committed` |
| **Committed** | 已提交进版本库（工作区干净） | `nothing to commit, working tree clean` |

**diff 的三个视角（用"三棵树"记：HEAD / Index / Worktree）**：

| 命令 | 比较对象 | 回答的问题 |
|------|---------|-----------|
| `git diff` | 工作区 ↔ 暂存区 | "我改了但还没 add 的是什么" |
| `git diff --staged`（= `--cached`） | 暂存区 ↔ HEAD | "我这次 commit 将要提交什么" |
| `git diff HEAD` | 工作区 ↔ HEAD | "相对上次提交，我总共改了什么"（含已暂存+未暂存） |
| `git diff A..B` / `A...B` | 两个提交（两点=直接比较；三点=从共同祖先到 B） | 版本间差异 / PR 的实际变更 |

**两个进阶点**：`git diff --stat`（只统计行数变化）、`git diff --word-diff`（按词而非按行，写文档时好用）；**空白差异**用 `-w` 忽略（CRLF 改造后"全文件都变了"的假象，4.1.2）。**"为什么 git status 说改了但 diff 是空的"**——多半是**换行符/文件权限**（mode 变化）或**行尾空白**：`git diff --ignore-all-space` 与 `core.fileMode` 排查（`git config core.fileMode false` 在跨平台场景常用）。

**易错**：`git diff --cached` 与 `git diff HEAD` 混用（前者只看"将提交"，后者看"全部未提交"）；合并冲突时 `git diff` 显示的是冲突标记，不是最终差异；`git status` 里同一文件同时出现"staged"与"not staged"（部分暂存，正常且强大）。

## 1.2 对象与存储

### 1.2.1 Git 的四种对象是什么？"内容寻址"与"快照"怎么理解？

**Git 的本质：一个内容寻址的键值数据库 + 一组引用（refs）**。四种对象（`git cat-file -p <sha>` 可逐个看）：

| 对象 | 内容 | 类比 |
|------|------|------|
| **blob** | 文件内容（**不含文件名/路径/权限**） | 文件的数据 |
| **tree** | 目录：条目列表（mode + 类型 + SHA + 文件名），指向 blob 或子 tree | 目录 |
| **commit** | 顶层 tree 的 SHA + 父 commit（0/1/多个）+ 作者/提交者 + 时间 + message | 一次快照的"信封" |
| **tag（附注标签）** | 指向某个对象的 SHA + tagger + message（可签名） | 带说明的指针 |

**内容寻址**：对象的键 = `SHA-1(类型 + 长度 + 内容)`（Git 2.29+ 可选 SHA-256，5.2.1）——**同样的内容永远得到同一个 SHA**，所以：相同文件天然去重（两个文件同内容=同一个 blob）；对象不可变（改一个字节=新对象新 SHA）；完整性可校验（`git fsck` 重算哈希）。**引用（refs）** 是"人类可读的名字 → SHA"：分支（`.git/refs/heads/*`）、远程跟踪分支（`refs/remotes/origin/*`）、标签、`HEAD`（当前分支/提交）。

**"快照而非差异"怎么理解（必须讲准，否则被追问 packfile 打脸）**：
- **逻辑层面**：每个 commit 指向一棵**完整目录树**——checkout 任意 commit 都是"整棵树"的视角，不需要像 SVN 那样沿 diff 链条回放；这是"切分支快、历史查询快"的根源；
- **物理层面**：Git **并不真的存 N 份完整文件**——未被修改的文件，tree 直接**复用同一个 blob SHA**（零成本共享）；打包（packfile）时还会做**delta 压缩**（2.1.2）。
- **标准答法**："**Git 在对象模型上存快照，在存储实现上用 delta 压缩**——两者不矛盾：快照是语义（提交即完整状态），delta 是物理（省空间）。"

**一个常用手算题**：`git rev-parse HEAD^{tree}` 拿根 tree；`git ls-tree` 看目录条目；`git cat-file -p HEAD` 看 commit 的父指针与 tree——**"改一个文件为什么整个仓库 SHA 都变"**：文件 blob 变 → 该文件所在的 tree 变 → 一路向上到根 tree → commit 变（父 commit SHA 进新 commit）→ 分支指针移动；**没变的子树 SHA 保持不变**（内容寻址的级联特性）。

**易错**：说"blob 存文件名"（文件名在 tree 里，blob 只有内容——所以重命名文件不产生新 blob）；说"commit 存 diff"（逻辑上是快照，详见上）；以为 tag 一定是对象（**轻量标签只是 refs 里的一个 SHA**，不是对象，3.2.1）。

### 1.2.2 packfile、delta 压缩、gc 是怎么回事？仓库为什么会膨胀？

**两种对象存储形态**：
- **松散对象（loose）**：每次 `git add`/`commit` 产生一个 zlib 压缩的独立文件（`.git/objects/ab/cdef...`）——写快、查慢、占空间；
- **打包对象（packfile）**：`git gc`/`git repack` 把大量对象打包成 `.pack` + `.idx`——**相似对象做 delta 压缩**（存"基准 + 差异指令"），并支持**位图索引（bitmap）**加速历史遍历（clone/fetch 快）。

**delta 压缩与"快照"的关系（1.2.1 的延伸）**：packfile 会选一个"基准对象"，把相似对象存成"相对基准的差异"（可链式多层，深度受控）——**这是存储优化，不是语义变化**；读对象时 Git 自动"解 delta"还原完整内容，上层永远看到完整快照。**所以"Git 存快照"与"Git 内部有 delta"同时为真**——面试把这句话讲清就是满分。

**仓库膨胀的五个来源（运维题的落点）**：
1. **大文件进了历史**（二进制/构建产物/数据集）——即使后来删除，历史里仍在（4.1.3 的清理场景）；
2. **频繁提交大文件的微小改动**（每次一个完整 blob，pack 后仍有成本）；
3. **未清理的悬空对象**（reset/rebase 后的旧提交，reflog 过期前仍占空间）；
4. **packfile 碎片**（多次 fetch 产生多个 pack，`git repack -ad` 合并）；
5. **浅克隆/部分克隆没配**（大仓库全量拉取，4.2.2）。

**运维命令（报得出即实战）**：`git count-objects -vH`（松散对象与 pack 的体积）、`git gc --prune=now`（清理不可达对象，**别在共享仓库随意跑**）、`git repack -adf --depth=...`、`git maintenance start`（2.30+ 的后台维护，推荐替代手工 gc）、`git fsck --lost-found`（找悬空对象，2.1.2 的救火）。

**易错**：`git gc` 不能"删掉已提交的历史"（可达对象永远保留，要改写历史用 4.1.3）；在 CI 里每次 clone 都 `--depth=1` 却又要 `git describe`/查历史（浅克隆拿不到 tag 与祖先，4.2.2）；`git prune` 清掉未过期 reflog 里的"救命提交"（2.1.2 的恢复窗口依赖 reflog 未过期）。

# 2. 日常操作与撤销

## 2.1 提交与撤销

### 2.1.1 reset 的三种模式有什么区别？reset、revert、restore、checkout 怎么分工？

**reset 的本质：移动当前分支指针（HEAD），并按模式决定是否重置暂存区与工作区**：

| 模式 | HEAD | 暂存区（index） | 工作区 | 效果 |
|------|------|----------------|--------|------|
| `--soft` | 移到目标 | **不动** | 不动 | 撤销 commit，改动仍**已暂存**（可重新提交/合并提交） |
| `--mixed`（默认） | 移到目标 | **重置为目标** | 不动 | 撤销 commit 与 add，改动回到**未暂存** |
| `--hard` | 移到目标 | 重置 | **重置** | 彻底丢弃（**未提交改动直接消失**，靠 reflog 才救得回，2.1.2） |

**四个命令的分工（现代 Git 的"正确工具"）**：

| 目的 | 旧命令（仍可用） | 新命令（2.23+） |
|------|-----------------|----------------|
| 切换分支 | `git checkout <branch>` | `git switch <branch>` |
| 创建并切换 | `git checkout -b <b>` | `git switch -c <b>` |
| 丢弃工作区某文件改动 | `git checkout -- <file>` | `git restore <file>` |
| 取消暂存（unstage） | `git reset HEAD <file>` | `git restore --staged <file>` |
| 撤销某次提交（保留历史） | — | **`git revert <commit>`**（生成反向提交） |
| 本地改写历史 | `git reset` | 同左（**只对未推送的提交**） |

**revert vs reset 的判决句**：**"已推送的历史只 revert，未推送的历史才 reset"**——reset 会让提交从分支历史消失（协作者的世界里"你改了历史"，下次 push 被拒或强推覆盖别人），revert 是"新增一个抵消提交"，对所有人安全；**merge commit 的 revert 要 `-m 1` 指定保留哪条主线**（把合并当作"相对第一父的变更"反转），且"revert 一个 revert"才能恢复被撤销的合并。

**易错**：`reset --hard` 前没看 `git status`（未提交改动灰飞烟灭）；`git reset HEAD~1` 后以为"提交没了"（reflog 里还在，2.1.2）；对**已推送**的分支 reset 后 `push --force`（团队协作事故，2.2.3 的 force-with-lease 是补救而非许可）；`restore --staged` 与 `restore` 混用（前者只动暂存区，后者动工作区，2.23 前用 reset 的语义要迁移过来）。

### 2.1.2 误删的提交/分支怎么找回？reflog 的救火流程？

**reflog 是什么**：Git 在 `.git/logs/` 里记录**本地 HEAD 与各分支引用的每次移动**（commit/checkout/reset/rebase/merge 全在内），带时间与原因——**"本地后悔药"**（只在本地，不随 push 同步，2.1.2 的恢复依赖它未过期：默认可达记录保留 **90 天**，不可达对象 30 天）。

**四个救火剧本**：
1. **reset --hard 丢提交**：`git reflog` 找到丢失提交前的那一行（如 `HEAD@{1}`）→ `git reset --hard HEAD@{1}`（或 `git reset --hard <sha>`）；**merge/rebase 刚做完想反悔**：`git reset --hard ORIG_HEAD`（Git 自动记录的操作前位置）；
2. **删掉的分支找回来**：`git reflog`（或 `git reflog show <branch>`）找到该分支最后指向的 SHA → `git branch <name> <sha>` 重建；**已推到远程的分支**直接从远程再拉（`git fetch` 后 `git branch <name> origin/<name>`）；
3. **amend 覆盖了旧提交**：`git reflog` 找 amend 前的 SHA → `git reset --hard <old>`（或 `git cherry-pick` 旧提交到新分支，保留两者）；
4. **stash 误删/丢失**：`git fsck --unreachable --no-reflogs | grep commit` 列出悬空提交 → `git show <sha>` 确认内容 → `git stash apply <sha>` 或 `git branch rescue <sha>`；`git fsck --lost-found` 把悬空对象导出到 `.git/lost-found` 再逐个看。

**操作纪律（比命令更值钱）**：救火前**先 `git branch backup-<date>` 把当前状态钉住**（避免二次破坏）；**别在慌乱中 `git gc --prune=now`**（会把"还能救"的对象清掉）；恢复后核对**文件内容与提交信息**，而不是只看"SHA 存在"；**reflog 过期后**（90 天/30 天）只能靠远程仓库、备份或其他协作者的本地仓库。

**易错**：在**裸仓库/服务端**找 reflog（服务端没有工作区 reflog，只有引用日志，恢复手段不同）；`git reflog` 看到一堆 rebase 记录不会读（每条是"旧 SHA → 新 SHA + 动作"，按时间倒序）；恢复 `--hard` 前的**未提交**改动（reflog 救不了未提交内容——那要靠 stash/编辑器历史/IDE Local History）。

### 2.1.3 stash 和 worktree 各解决什么问题？怎么选？

**stash：把"未完成的改动"暂存起来，让工作区回到干净状态**——场景：正在改 A 功能，线上要紧急修 B（切分支前必须干净）；临时拉一下 main 验证问题；**本质是"临时的、本地的、易忘的"**。

```bash
git stash push -u -m "wip: order-refactor"   # -u 连未跟踪文件；-m 加说明（强烈建议）
git stash list                                # stash@{0} 最新
git stash show -p stash@{0}                   # 看内容
git stash pop                                 # 恢复并删除（冲突时保留 stash）
git stash apply stash@{0}                     # 恢复但保留
git stash branch fix-order stash@{0}          # 从 stash 建新分支再恢复（冲突场景的正解）
git stash drop / clear                        # 删除单条 / 全部（clear 很危险）
```

**worktree：同一仓库检出多个工作目录**——场景：正在开发分支上跑着测试/编译，要同时在另一个分支改 hotfix、对比两个版本、跑长任务；**不复制 .git，不重新 clone**（共享对象库，省磁盘省时间）：

```bash
git worktree add ../proj-hotfix hotfix/urgent   # 新目录检出指定分支
git worktree list / remove ../proj-hotfix / prune
```

**选型句**："**改动几分钟内要回来 → stash；要并行干活（编译/测试/长任务/双版本对比）→ worktree**"——stash 的坑是"忘了自己有 stash"（`git stash list` 成为考古现场），worktree 的坑是"同一分支不能同时检出"（Git 直接拒绝）与 IDE/构建工具对多目录的支持要配。

**易错**：`stash pop` 冲突后 stash **不会自动删**（解决完要手动 drop，否则重复 apply 出乱子）；`stash` 默认**不含未跟踪文件**（新文件丢了以为 stash 没生效）；把 stash 当"长期存档"（它不是备份、不跨机器、不在 push 里）；worktree 目录删了不 `prune`（元数据残留，5.1.1 的清理项）。

## 2.2 分支与合并

### 2.2.1 merge 与 rebase 的区别？fast-forward、--no-ff、squash 怎么选？

**机制差异**：

| 维度 | merge | rebase |
|------|-------|--------|
| 动作 | 把两条历史**连接**起来：生成一个**双父提交**（或快进） | 把当前分支的提交**逐个重放**到新基底上 |
| 历史形状 | 保留真实分叉与合并点（图状） | **线性**（像一直在最新代码上开发） |
| SHA | 原提交不变（新 merge commit 是新增） | **全部重写**（新 SHA，旧提交变悬空） |
| 冲突 | 一次性解决（合并点） | **每个被重放的提交都可能冲突**（可 `--skip`/`--continue`） |
| 适用 | 公共分支集成、保留协作痕迹 | 个人分支同步主干、提交整理 |

**黄金规则**：**"不要 rebase 已推送到公共分支的提交"**——别人基于旧 SHA 的工作会全部错位；个人 feature 分支（只有你在推）随意 rebase。

**三种"合并口味"的选择（团队规范的核心）**：
- **fast-forward**（默认当可分叉不存在时）：HEAD 直接前移，**不产生 merge commit**——历史最干净，但**丢失"这是一次功能合并"的信息**；
- **`--no-ff`**：即使能快进也强制生成 merge commit——**保留"功能分支边界"**，回滚一个功能=revert 一个 merge commit（Git Flow 派系偏好）；
- **`--squash`**：把分支所有提交压成**一个未提交的变更集**（再 commit）——主干历史一条线、每个 PR 一个提交；**代价是丢失分支内提交粒度**，且 Git 不再认为两条历史"已合并"（后续重复合并可能再冲突）；
- **rebase merge**（平台侧选项）：把 PR 提交逐个重放到主干（线性 + 保留粒度），等价于"PR 前 rebase 后快进"。

**merge 策略的底层（追问位）**：默认 **`ort`**（2.34+ 取代 `recursive`，冲突处理更快更准）；`octopus`（多分支一次合并）；`-s ours`（**完全忽略对方内容**，只留合并关系——发布分支"宣告已合并"的冷门用法）；`-X ours/theirs`（**冲突时的偏好提示**，只在冲突块生效，不是"整分支取一边"）——`-s` 与 `-X` 的区别是高频陷阱。

**收口**："**merge 保守、rebase 洁癖、squash 强迫症**——三者没有对错，只有团队契约：主干要求线性 → rebase/squash；要求可追溯功能边界 → no-ff；**契约写进分支保护与 CI，而不是靠自觉**（3.1.2）。"

### 2.2.2 冲突怎么解？cherry-pick 与 backport 的纪律是什么？

**冲突的标准流程（merge 版）**：
1. `git status` 看 `both modified` 的文件；
2. 打开文件看冲突标记（`<<<<<<< HEAD` / `=======` / `>>>>>>> branch`），**结合业务语义**取舍或手工融合——不是"无脑留一边"；
3. `git add <file>` 标记已解决（这是"解决"的协议动作）；
4. `git commit` 完成合并（提交信息里写清冲突处理）；
5. 放弃：`git merge --abort`（回到合并前）。**rebase 版**的差异：每步 `git rebase --continue`、冲突可 `--skip` 跳过该提交、放弃 `git rebase --abort`。

**降痛三件套**：`merge.conflictStyle=zdiff3`（冲突块里带上**共同祖先**的版本，一眼看出两边各改了什么）；`git rerere`（**记录并自动复用**冲突解决方案——长期分支反复 rebase 时是神器，开启 `rerere.enabled=true`）；`git mergetool`（图形化三窗格，复杂冲突更稳）。

**cherry-pick：把某个提交"摘"到当前分支**（生成新 SHA，不是引用原提交）：

```bash
git cherry-pick <sha>            # 应用单个提交
git cherry-pick <a>^..<b>        # 应用一段（不含 a 的父？注意区间语义）
git cherry-pick -x <sha>         # 提交信息里记录来源 SHA（backport 必备）
git cherry-pick -n <sha>         # 只改工作区/暂存区，不自动提交
```

**backport 纪律（发布分支维护的标准动作）**：修复先在 main 合入 → `cherry-pick -x` 到 release 分支 → 打补丁版本 tag（3.2.1）→ **记录 backport 清单**（哪些修复进了哪个版本）。**cherry-pick 的代价**：同一改动在两条历史里是**两个不同 SHA**——未来两条分支合并时 Git 不知道它们"同源"，可能重复冲突；`-x` 的来源记录是给人和工具看的线索，**减少"忘了 backport"与"backport 了两次"**。

**易错**：冲突时删掉标记但没 `git add` 就 commit（Git 拒绝或把标记提交上去）；rebase 冲突解决到一半切分支（状态卡住，先 abort）；cherry-pick 一个 merge commit（要 `-m` 指定父，语义容易错）；把 cherry-pick 当"日常同步分支"（应该 merge/rebase，cherry-pick 只做"跨历史的定点搬运"）。

### 2.2.3 fetch、pull、push 的语义？force 推送怎么做才安全？

**三者的关系**：

```text
git fetch  ：下载远程对象 + 更新"远程跟踪分支"（origin/main），**不动你的本地分支与工作区**
git pull   ：fetch + 集成（默认 merge；pull.rebase=true 则 rebase；pull.ff=only 则只允许快进）
git push   ：把本地提交上传并把**远程分支指针**前移（快进）——非快进会被拒（默认安全阀）
```

**fetch 与 pull 的分工（工程口径）**：`fetch` 让你**先看清再决定**（`git log HEAD..origin/main` 看进来了什么、`git diff` 看差异），`pull` 一步到位但"合并策略由配置决定"——团队里**建议 `pull.ff=only`**（能快进就快进、有分叉就明确报错让你手动 merge/rebase），避免"莫名其妙一个 merge commit"。

**refspec 与跟踪关系（面试深水区）**：`git push origin local:remote` 是显式 refspec；`-u/--set-upstream` 建立"本地分支 ↔ 远程分支"跟踪，之后 `git push/pull` 不带参数即可；`push.default=simple`（当前分支推到同名上游）；`fetch` 的 `+refs/heads/*:refs/remotes/origin/*` 是默认 refspec（`+` 表示允许强制更新远程跟踪引用，这只影响本地缓存，不影响服务端）。

**force 推送的安全姿势**：
- **永远优先 `--force-with-lease`**：它要求"远程分支仍指向我上次看到的 SHA"才允许覆盖——**别人刚推了新提交就失败**，避免"把同事的提交冲掉"；
- `--force-if-includes`（较新版本）再加一道"我确实基于最新远程提交"的校验；
- **`--force` 只用于"确定独占了该分支"**（个人分支 rebase 后）；**受保护分支**（main/release）在平台侧禁止强推（3.1.2），这是制度层防线；
- 强推后**通知协作者**（他们的本地历史已错位，需 `fetch` + `reset --hard origin/<branch>`）。

**易错**：`git push` 被拒就无脑 `--force`（先看 `git log origin/x..x` 与 `..origin/x` 两个方向，判断是谁领先）；push 了**未跟踪的分支**却以为推了 tag（tag 默认不随 push 走，3.2.1）；`pull` 产生意外 merge commit 后又 rebase 已推送的合并（历史二次改写）；删除远程分支用 `push origin --delete <b>` 与"本地删了但远程没删"的混淆（`git branch -d` 只删本地）。

# 3. 分支模型与协作

## 3.1 工作流

### 3.1.1 Git Flow、GitHub Flow、trunk-based 怎么选？

**三种主流模型（按"分支寿命与发布节奏"区分）**：

| 模型 | 分支结构 | 发布方式 | 适合 |
|------|---------|---------|------|
| **Git Flow** | main（生产）+ develop（集成）+ feature/release/hotfix 五类分支 | release 分支准备、打 tag 发布；hotfix 从 main 修 | **多版本并行维护**（客户端/SDK/需要给旧版本打补丁）、发布周期长 |
| **GitHub Flow** | main + 短命 feature 分支 + PR | 合并即部署（或合并后手动触发） | SaaS/互联网 Web 应用、持续交付成熟 |
| **trunk-based** | 主干 + **极短命分支**（小时~1 天），或直接小步提交主干 | 主干随时可发布；**feature flag** 控制未完成功能 | 高成熟度 CI/CD、大团队/单体大仓、追求最快反馈 |

**Git Flow 的现状（诚实版）**：作者 Vincent Driessen 后来在原文加了"**如果你做的是持续交付的 Web 应用，请考虑更简单的工作流**"的注记——**五分支模型的分支合并矩阵（release/hotfix 双向合回）在快节奏下是重税**；它真正的价值场景是"**同时维护 2~3 个已发布版本**"（release/1.x、2.x 各一条）与强发布评审的行业（金融/客户端）。

**选型四问**：① 要同时维护几个版本？（多 → Git Flow/发布分支）；② 部署频率？（每天多次 → trunk-based/GitHub Flow）；③ 功能开关与自动化测试成熟吗？（不成熟 → 短命分支会更稳）；④ 团队规模与仓库形态（monorepo 大团队 → trunk-based + 选择性 CI）。

**共同底线（无论哪种）**：**main 永远可发布**（合入前 CI 绿）、**分支短命**（越短冲突越少）、**保护分支 + 必需评审**（3.1.2）、**发布可追溯**（tag，3.2.1）——模型是壳，这四条是核。

### 3.1.2 PR/Code Review 的工程实践？合并策略怎么定？

**PR 的质量上限在"提 PR 的人"**：
1. **小 PR**：一个 PR 一件事（几百行内为宜），大 PR 拆成"可独立评审的提交序列"——评审质量与 PR 大小成反比是行业共识；
2. **自审先行**：提 PR 前自己按 diff 过一遍（`git diff main...`），删调试代码、补测试、写清**动机/方案/影响面/回滚方式**；
3. **标题与描述可检索**：关联 issue（`Closes #123`）、截图/录屏（前端）、风险标注（DB 变更/配置变更单独标）；
4. **Draft PR** 用于早期对齐方向，Reviewer 不必等"完成"才看。

**评审侧的制度化（平台能力）**：**CODEOWNERS**（按路径指定必审人）；**required reviews**（至少 1~2 人）；**required status checks**（CI/扫描/覆盖率门槛）；**linear history**（禁止 merge commit，强制 squash/rebase）；**merge queue**（高频合并时避免"各自绿、合起来红"）；**protected tags**（发布 tag 只能由流水线打）。

**合并策略与历史形状的取舍（与 2.2.1 呼应）**：

| 平台选项 | 结果 | 适合 |
|---------|------|------|
| **Merge commit** | 保留每个提交 + 合并点 | 要保留分支内演进细节 |
| **Squash and merge** | 一个 PR = 一个提交 | 提交粒度不重要、要干净主干（最常见） |
| **Rebase and merge** | 提交逐个重放、无合并点 | 要线性 + 保留粒度（需提交本身整洁） |

**评审文化三条**（面试的成熟度信号）：评**代码不评人**（用问题句而非命令句）；区分 **blocking（必须改）与 nit（建议）**；**24 小时内响应**的 SLA——"评审排队"是交付周期的隐形大头（33_cicd.md 的 DORA 指标里，评审时长常比构建时长更影响 lead time）。

### 3.2.1 tag 的两种类型？发布纪律怎么定？

**轻量标签 vs 附注标签**：

| | 轻量（lightweight） | 附注（annotated） |
|--|--------------------|-------------------|
| 本质 | **只是一个 ref**（refs/tags/v1.0 → commit SHA） | **一个 tag 对象**（含 tagger、日期、message，可 GPG/SSH 签名） |
| 命令 | `git tag v1.0.0` | `git tag -a v1.0.0 -m "..."`（签名 `-s`） |
| 适用 | 临时标记/本地书签 | **正式发布**（可审计、可验证、`git describe` 更友好） |

**发布纪律（把"打 tag"升级成"发版流程"）**：
1. **版本语义**：SemVer（`MAJOR.MINOR.PATCH`）+ 预发布后缀（`-rc.1`）；tag 名与构建产物/镜像 tag 一一对应（23_docker.md 的不可变 tag 同理）；
2. **从受保护分支打、由流水线打**：人工在本地打 tag 容易"打在错误提交/未合入的代码"上——**tag 触发流水线构建发布**是标准形态；
3. **tag 默认不随 push 走**：`git push origin v1.0.0` 或 `--follow-tags`（只推**可达的附注标签**）；`--tags` 会连轻量一起推（容易误推一堆本地 bookkeeping tag）；
4. **tag 不可变**：**不要移动/删除已发布的 tag**（下游缓存、镜像 digest、审计都按它对齐）；打错了发新 patch 版本，平台侧用 **protected tag** 禁止改写；
5. **发布说明**：GitHub/GitLab Release 关联 tag + changelog（Conventional Commits 自动生成，4.3.1）。

**追问**：`git describe --tags` 用于"从 tag 推导版本"（`v1.2.3-4-gabc123` = 距 tag 4 个提交）——**浅克隆会拿不到 tag/祖先**，CI 里要 `fetch --tags` 或加深克隆（4.2.2）；签名 tag 的验证 `git tag -v v1.0`（公钥分发与信任链是另一层工程）。

# 4. 工程化

## 4.1 忽略、换行与敏感信息

### 4.1.1 .gitignore 的规则怎么写？为什么"加了也不生效"？

**规则语义（按优先级记）**：
- 每行一个 pattern；`#` 注释；`!` 取反（重新纳入）；**末尾 `/` 只匹配目录**；**开头 `/` 锚定仓库根**；`**` 匹配任意层级；`*`/`?`/`[]` 通配；
- **就近原则**：子目录的 `.gitignore` 覆盖父级；**最后匹配的规则生效**（所以"先忽略整个目录、再 `!` 放行某文件"要按顺序写，且被忽略目录内的文件**无法用 `!` 直接放行**——要先放行目录本身）；
- **其他忽略源**：`~/.config/git/ignore`（全局，`core.excludesFile`）、`.git/info/exclude`（本仓库私有，不进版本库）。

**"加了 .gitignore 还显示为修改/被跟踪"的根因**：**.gitignore 只对"未跟踪文件"生效**——文件一旦被 commit 过，忽略规则不再管它。解决：`git rm --cached <file>`（保留工作区文件、从索引移除）→ 提交这次移除 → 之后规则生效；批量：`git rm -r --cached . && git add .`（谨慎，先 `git status` 核对）。**验证规则**：`git check-ignore -v <path>` 直接告诉你"是哪条规则、在哪个文件里命中的"——比猜快一小时。

**工程口径**：忽略**构建产物**（`target/`、`dist/`、`*.class`）、**IDE/系统文件**（`.idea/`、`.DS_Store`）、**依赖目录**（`node_modules/`）、**本地配置与密钥**（`.env`、`*.pem`——但同时要有 `.env.example` 进库）；**忽略"生成物"而不是"源"**，锁文件（`pom.xml` 的依赖版本由 BOM/父 POM 管；`package-lock.json`/`poetry.lock` 应当进库）要按生态判断。

**易错**：`target/` 写成 `/target/`（只忽略根目录，子模块的 target 漏网）；`*.log` 把"需要进库的示例日志"也忽略；`.gitignore` 本身被忽略（罕见但致命，`git check-ignore -v .gitignore` 可查）；空目录 Git 不跟踪（要 `.gitkeep`）。

### 4.1.2 CRLF/LF 换行冲突怎么根治？

**问题根源**：Windows 习惯 CRLF（`\r\n`），Linux/macOS 与**所有脚本解释器/Docker 容器**按 LF（`\n`）——**带 CRLF 的 shell 脚本在 Linux 报 `bad interpreter: /usr/bin/env: 'bash\r'`**（23_docker.md 的 entrypoint 经典事故），同一文件在不同人机器上"整文件都变了"（diff 噪声、评审失效）。

**三层治理（按可靠性排序）**：
1. **仓库级（正解）：`.gitattributes`**——把规则写进版本库，所有人一致：
   ```text
   * text=auto                 # 自动识别文本并规范化（提交进库统一 LF）
   *.sh  text eol=lf           # 脚本强制 LF
   *.bat text eol=crlf         # Windows 批处理保留 CRLF
   *.png binary                # 二进制不做任何转换
   ```
2. **个人级：`core.autocrlf`**——`true`（Windows：提交转 LF、检出转 CRLF）、`input`（Linux/mac：提交转 LF、检出不动）、`false`（不转）；**团队混用时是"各人配置打架"的来源**，所以优先级低于 `.gitattributes`；
3. **校验**：`core.safecrlf=warn|true`（拦截不可逆转换）、CI 里跑 `git diff --check`（尾随空白/冲突标记）。

**存量文件的修复**：`git add --renormalize .`（按新规则重新规范化并提交）——**会产生一个"全文件变更"的大提交**，放到独立 PR、通知团队 `blame` 影响（`git blame --ignore-rev <renormalize-sha>` 可跳过该提交）。

**易错**：只配了 `core.autocrlf` 没配 `.gitattributes`（换一台机器/换一个人就复发）；把 PNG/JAR 也按文本处理（损坏二进制，必须 `binary` 或 `-text`）；改完 `.gitattributes` 不 renormalize（存量文件仍是旧行尾）；**用编辑器"转换行尾"提交**（治标，团队下一个人又改回来）。

### 4.1.3 敏感信息误提交了怎么彻底清除？为什么"删掉文件"没用？

**先立铁律**：**密钥一旦进入过远程仓库（尤其公开仓库），就当作已泄露**——历史可被 fork/缓存/爬虫/平台事件留存；**第一步永远是"轮换/吊销密钥"**，清历史是第二步（降低继续暴露面，不是"当没发生"）。

**清除三步（从"止血"到"根治"）**：
1. **阻止继续传播**：轮换密钥、撤销 token、下线泄露的凭据；
2. **改写历史**：`git filter-repo`（官方推荐，替代已弃用的 `filter-branch`）：
   ```bash
   git filter-repo --path secrets/prod.pem --invert-paths   # 从所有历史删除该路径
   git filter-repo --replace-text replacements.txt          # 按文本替换（如把 AK 换成 ***）
   ```
   或 **BFG Repo-Cleaner**（更快、语法更简单：`bfg --delete-files prod.pem`、`--replace-text`）；
3. **强推所有引用并通知**：`git push --force --all && git push --force --tags`；**所有协作者必须重新 clone**（旧历史 SHA 全变，本地合并会把泄露内容带回来）；平台侧还要处理 **fork、PR 缓存、Release 附件**（GitHub 需工单清理缓存/fork 网络）。

**为什么会"删了文件还在"**：对象一旦提交就进了对象库（1.2.1），后续 `git rm` 只是在新提交里删掉——**旧提交仍可达**，clone 全量历史就能拿到；只有**改写历史 + 强推 + 让所有人丢弃旧克隆**，旧对象才会在远端逐步不可达（本地的 reflog/悬空对象也要 `gc --prune=now` 才清）。

**预防（比清除便宜一百倍）**：`.gitignore` + `.env.example`；**pre-commit 钩子扫密钥**（gitleaks/trufflehog，4.3.1）；CI 里跑密钥扫描；平台的 **secret scanning** 告警；配置/密钥走密钥管理（Vault/KMS/云凭据服务），**代码里只放引用**。

**易错**：只删当前分支的文件（其他分支/tag 里还有）；强推后不通知团队（同事一 push 又把历史推回来）；以为"私有仓库就不算泄露"（权限扩大、离职、供应链都是面）；忘了 CI 缓存/构建产物里也有密钥。

## 4.2 大仓库与依赖管理

### 4.2.1 submodule、subtree、monorepo 怎么选？

**三种"多仓库组织"的机制与代价**：

| 方案 | 机制 | 优点 | 坑 |
|------|------|------|----|
| **submodule** | 主仓库记录子仓库的**一个 commit 指针**（gitlink）+ `.gitmodules` | 子仓库独立版本/权限/CI；主仓库不背子仓库历史 | **克隆要 `--recursive`**、更新要 `git submodule update --init`、默认 detached HEAD（改完忘 push）、指针更新是一次"提交一个 SHA"（评审看不出内容） |
| **subtree** | 把外部仓库**整个并入**主仓库某目录（历史合并进来） | 使用者零心智（clone 即全量）；可改可提交 | 回推上游要 `subtree push --prefix`（易错）；主仓库变大；上游更新要 merge/split |
| **monorepo** | 所有项目一个仓库 | **原子跨项目变更**（改接口+改调用方一个 PR）、统一工具链/版本 | 克隆与 CI 规模问题（4.2.2 的 sparse/partial）、权限粒度粗、工具链要求高（Bazel/Nx/Turborepo/Maven 多模块） |

**选型口径**：**第三方/内部基础库、更新不频繁、消费方不多** → subtree（省心）或**直接走包管理**（Maven/NPM 私有仓库，最常规）；**需要独立发布/权限、联合开发活跃** → submodule（或干脆独立仓库 + 包依赖）；**组织级频繁跨项目重构、统一发布** → monorepo + 选择性构建。**"默认选择"其实是包管理**——把库当依赖引，比任何仓库嵌套都清晰（3_maven.md 的多模块与私服）。

**submodule 的正确姿势（如果非用不可）**：`git clone --recurse-submodules`；日常 `git submodule update --remote` 拉子仓库最新；在子目录里改完**先在子仓库 commit+push，再回主仓库提交指针**（顺序反了就是"指针指向不存在的提交"）；CI 里显式 init/update。

### 4.2.2 大仓库怎么提速？shallow/partial clone、sparse-checkout、LFS 分别解决什么？

**四把刀对应四种"大"**：

| 手段 | 解决什么 | 命令要点 | 代价 |
|------|---------|---------|------|
| **浅克隆 shallow** | **历史太长**（提交多） | `git clone --depth=1`（CI 标配）；`--shallow-since` | 无历史/tag/`describe`，回溯与 blame 受限；需要时 `git fetch --unshallow` |
| **部分克隆 partial** | **历史里的 blob 太大** | `--filter=blob:none`（用时按需拉 blob）或 `--filter=tree:0` | 需要服务端支持；离线/弱网下按需拉取会卡 |
| **稀疏检出 sparse-checkout** | **工作区文件太多**（只要子目录） | `git sparse-checkout set <dir>`（cone 模式） | 只减工作区，不减历史；构建脚本要适配 |
| **Git LFS** | **二进制大文件**（设计稿/模型/视频） | `git lfs track "*.psd"`（写 `.gitattributes`）；clone 自动拉 LFS | 需要 LFS 服务与配额；**LFS 指针不是内容**，迁移/备份要带上 LFS 存储 |

**配套的"仓库体检"**：`git count-objects -vH`、`git commit-graph write`（加速历史遍历）、`git multi-pack-index write`、`core.fsmonitor`/`untracked cache`（`git status` 提速）、`git maintenance start`（后台维护，1.2.2）。

**工程判断**：**CI 用浅克隆**（除非要 tag/历史）；**monorepo 用 sparse-checkout + 选择性构建**；**二进制资产不要进 Git**（放对象存储/制品库：25_minio.md、Nexus）；**大文件已经进了历史** → `filter-repo` 清理 + LFS 迁移（4.1.3 同款流程）。

## 4.3 质量与钩子

### 4.3.1 hooks、提交签名、分支保护怎么组合成"质量门禁"？

**hooks 的两类（关键差异：客户端不可信）**：
- **客户端钩子**（`.git/hooks/`，不随 clone 分发）：`pre-commit`（lint/格式化/密钥扫描）、`commit-msg`（Conventional Commits 校验）、`pre-push`（单测/类型检查）——**靠 husky/lefthook/pre-commit 框架统一安装**；
- **服务端钩子**（`pre-receive`/`update`）：**唯一不可绕过的一层**——分支保护、签名校验、禁止大文件等硬规则放这里（或平台的 branch protection 等价能力）。

**签名与验证**：GPG 或 **SSH 签名**（`gpg.format=ssh`，配置更轻）——`git commit -S`/`commit.gpgsign=true`、`git tag -s`；`git log --show-signature` 验证；平台显示 **Verified** 徽章；配合**分支保护要求签名提交**，防"冒充作者"的供应链攻击。

**完整门禁链（从本地到合并）**：本地 pre-commit（快、可跳过）→ push 触发 CI（编译/测试/静态扫描/依赖漏洞/密钥扫描）→ PR 必需评审 + CODEOWNERS → 必需状态检查通过 → merge queue 串行验证 → 合并后主干流水线构建制品/镜像并打 tag（3.2.1）→ 部署环境再验签/准入（23_docker.md 的镜像签名、32_kubernetes.md 的准入控制）。

**提交规范**：Conventional Commits（`feat:`/`fix:`/`refactor:`…）+ commitlint 校验 + **语义化发布**（根据提交类型自动 bump 版本、生成 changelog）——**提交信息是自动化发布的输入**，"随手写"会让 changelog 与版本号失真。

**易错**：把质量全押在客户端钩子（`--no-verify` 一绕就过）；pre-commit 跑全量测试（太慢，开发者会关掉——本地只跑"快检查"，重活放 CI）；签名配了但没在平台强制（等于没配）；分支保护只挡 push 不挡 force（要显式禁止 force push/删除）。

# 5. 故障排查与原理追问

## 5.1 故障排查

### 5.1.1 协作中的常见故障怎么处置？给一张"症状 → 处置"表。

| 症状 | 根因 | 处置 |
|------|------|------|
| `push` 被拒：`non-fast-forward` | 远程有你本地没有的提交（别人推过/你在旧基础上 rebase 过） | **先 `git fetch`，看两个方向**：`git log HEAD..origin/main`（远程多了什么）与 `..HEAD`（你多了什么）——再 merge/rebase；**独占分支**才 `--force-with-lease`（2.2.3） |
| `You are in 'detached HEAD' state` | checkout 到了某个提交/tag 而非分支 | 想保留改动：`git switch -c <new-branch>`；只是看看：`git switch -` 回原分支（改动会丢，先 stash） |
| 误合并已推送，想撤销 | merge 已进公共历史 | **`git revert -m 1 <merge-sha>`**（保留主线）；不要 reset 公共分支（2.1.1） |
| `index.lock` 存在，操作被拒 | 上一个 Git 进程被强杀/崩溃留下锁 | 确认没有 Git 进程在跑后 `rm .git/index.lock`（**别在操作进行中删**） |
| `refusing to merge unrelated histories` | 两个仓库各自 `init`，没有共同祖先 | 确认确实要合并后 `--allow-unrelated-histories`；更常见的是"拉错仓库"，先核对 remote |
| `detected dubious ownership` | 仓库属主与当前用户不一致（容器/root 挂载/共享盘） | `git config --global --add safe.directory <path>`（**只加可信目录**，别 `*`） |
| 大小写重命名不生效 | 文件系统大小写不敏感（Windows/macOS） | 两步走：`git mv a A.tmp && git mv A.tmp A`；或 `git mv --force` |
| 所有文件都显示 modified，diff 却是空的 | 换行符/文件权限位变化 | `git config core.fileMode false`（Windows）；行尾治理看 4.1.2 |
| `remote: Permission denied (publickey)` | SSH key 未加载/多账号用错 key | `ssh -T git@host` 验证；`~/.ssh/config` 按 Host 指定 IdentityFile；`ssh-add` |
| 推送被拒：文件过大 | 平台单文件上限（如 100MB）/仓库配额 | 大文件转 **LFS**（4.2.2）或从历史清除（4.1.3） |
| `Your branch and 'origin/x' have diverged` | 本地与远程各走各的（常见于 pull 后又本地提交） | `git pull --rebase` 或显式 merge；团队统一 `pull.ff=only` 能提前暴露（2.2.3） |
| 子模块目录是空的/指针不对 | 没 init/update；指针指向未推送的提交 | `git submodule update --init --recursive`；子仓库先 push 再提交指针（4.2.1） |
| `cannot lock ref` / `cannot lock ref 'refs/...'` | 并发 fetch/push 或引用文件残留 | 重试；仍失败检查 `.git/refs/**/*.lock` 与后台进程 |
| 仓库疑似损坏 | 磁盘/异常中断 | `git fsck --full` 定位；**先备份整个目录**，再考虑从远程重新 clone（本地未推送的提交先救，2.1.2） |

**通用处置纪律（比单条命令重要）**：**先备份再动手**（`git branch backup-<timestamp>` / 复制目录）；**先看再改**（`git status`、`git log --oneline --graph --all`、`git reflog` 三连）；**不认识的参数不硬上**（尤其 `--force`/`--hard`/`gc --prune=now` 三件套）；**事故后写复盘**（把处置流程沉淀成团队 runbook，33_cicd.md 的事故管理同款）。

### 5.1.2 仓库越用越慢/越大，怎么系统性治理？

**诊断（先量化再动手）**：
```bash
git count-objects -vH                                   # 松散对象/pack 体积
git rev-list --objects --all | \
  git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' | \
  sort -k2 -n -r | head -20                             # 历史里最大的 20 个对象
```
**四种"大"对应四种处方**：① **历史长**（提交多）→ 浅克隆/commit-graph（4.2.2）；② **历史里有大 blob** → `git filter-repo --strip-blobs-bigger-than 10M`（4.1.3 同族，**会改写历史**）；③ **工作区文件多**（monorepo）→ sparse-checkout + 选择性 CI；④ **二进制资产日常增长** → LFS/制品库（25_minio.md、Nexus）。

**日常维护**：`git maintenance start`（自动 gc/commit-graph/多 pack 索引，**替代手工 gc**）；`git repack -ad` 合并碎片；`git gc --prune=now` 清不可达对象（**只在确认 reflog 里没有要救的东西后**，2.1.2）；`git fsmonitor`/`untracked cache` 让 `git status` 在超大工作区也秒回。

**"治本"的架构动作**：**拆仓**（`filter-repo --path <subdir> --to-subdirectory-filter` 把子目录连同历史抽成新仓库）；**大文件迁 LFS/对象存储**；**CI 缓存依赖而非缓存整个仓库**；**托管平台侧**开 packfile/协议优化（现代平台默认）。

**易错**：`filter-repo` 后忘了强推与通知团队（4.1.3 的完整流程）；在生产主干上直接跑 `gc --prune=now`（把别人正在推的对象清掉）；"仓库慢"其实是**杀毒软件/网络盘/IDE 索引**（先排除环境）；用 `git clone` 反复拉同一仓库做多分支工作（worktree，2.1.3）。

## 5.2 原理追问

### 5.2.1 SHA-1 会碰撞吗？Git 怎么防？SHA-256 迁移到哪一步了？

**结论先行**：**SHA-1 在对抗性场景已被攻破**（2017 年 SHAttered 给出两个不同 PDF 的碰撞；2020 年 chosen-prefix 碰撞进一步降低成本），**但 Git 的日常使用并未因此崩塌**——因为 Git 做了三层防御：
1. **对象格式加前缀**：hash 的是 `"blob 1234\0" + 内容`（类型+长度），**长度与类型参与哈希**，构造"同 SHA 的不同类型对象"更难；
2. **内置碰撞检测**：Git 2.13+ 默认启用 **sha1collisiondetection**（检测到已知碰撞模式直接报错拒绝）；
3. **完整性与签名**：`git fsck` 校验对象；提交/标签签名（GPG/SSH）提供**独立于哈希的身份认证**（4.3.1）——攻击者要伪造历史，还得伪造签名。

**SHA-256 迁移**：Git 2.29+ 支持 **SHA-256 仓库**（`git init --object-format=sha256`），采用"双哈希过渡"设计（新仓库用 SHA-256、与 SHA-1 仓库互操作有兼容层）；但**托管平台与周边工具的支持仍不完整**（GitHub 等主流平台长期以 SHA-1 仓库为主），**2026 年的现实是：SHA-1 + 碰撞检测仍是主流，SHA-256 是"可用但未普及"的前瞻选项**——面试报"知道有迁移路径、生态尚未铺开"即准确，别断言"已经全面切换"。

**答题收口**："Git 的安全模型不是'哈希不可碰撞'，而是**内容寻址 + 碰撞检测 + 签名 + 分布式副本互相印证**——单一 SHA-1 碰撞不足以悄悄替换你 clone 到的历史。"

### 5.2.2 为什么 Git 比 SVN 快？为什么说"分支是 O(1)"？

**五个结构性原因**：
1. **全量本地**：clone 时把**完整历史**拉到本地——`log/diff/blame/切分支`全是本地磁盘操作，没有网络往返；SVN 的 log/blame 要问服务器（集中式的天然延迟）；
2. **不可变对象 + 内容寻址**：对象只增不改、无锁读写（1.2.1）——并发操作不需要"文件锁服务器"，也不会因为别人提交而阻塞；
3. **快照模型**：commit 指向完整 tree，**checkout = 把树"物化"到工作区**，不需要像 SVN 那样沿 delta 链逐版本回放；未被修改的文件直接复用 blob；
4. **packfile + delta + bitmap + commit-graph**：存储紧凑、历史遍历有索引加速（1.2.2）——"Git 存快照"与"物理 delta"并存，兼顾语义与体积；
5. **分支/标签只是 refs 文件**：一个分支就是 `.git/refs/heads/x` 里的一行 SHA——**创建/切换分支 O(1)**；SVN 的分支是"服务器上的目录拷贝"（虽然廉价但是服务端操作），合并追踪靠 `svn:mergeinfo` 属性，复杂且易错。

**"快"的边界（成熟度信号）**：Git 在大二进制仓库上会慢（每次改二进制都是完整 blob、pack 效率低）——这正是 **LFS/partial clone** 存在的原因（4.2.2）；`git status` 在超大工作区慢 → untracked cache/fsmonitor；**历史极深**的 `blame/log -S` 慢 → commit-graph/`blame --ignore-rev`。

**SVN 对照的一句话**："SVN 是**集中式文件版本管理**（服务器权威、目录即分支、合并难），Git 是**分布式内容寻址数据库**（本地全量、指针即分支、合并是一等公民）——两者的性能与协作差异都源于这个模型差异，不是'实现优化'的差别。"