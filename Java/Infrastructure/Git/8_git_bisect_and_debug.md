
# 1. Git Bisect — 二分查找

## 1.1 基本原理

`git bisect` 使用二分查找算法在提交历史中定位**首次引入 bug 的提交**。

```
正常提交: ✓
问题提交: ✗

初始状态：
A ✓ → B ✓ → C ✓ → D ✗ → E ✗ → F ✗ → G ✗

二分查找过程：
Step 1: 检查中间提交 D → ✗，bug 可能在 A~D 之间
Step 2: 检查 A~D 中间 B → ✓，bug 在 B~D 之间
Step 3: 检查 B~D 中间 C → ✓，bug 在 C~D 之间
Step 4: D 就是第一个有问题的提交！

6 个提交 → 只需要检查 2 次（二分查找的威力）
```

## 1.2 基本用法

```bash
# 开始 bisect
git bisect start

# 标记当前版本为 bad（有 bug）
git bisect bad

# 标记已知的最后一个正常版本为 good
git bisect good v1.0.0

# Git 会 checkout 一个中间的 commit，让你测试
# 测试后标记：
git bisect good   # 当前 commit 没问题
git bisect bad    # 当前 commit 有 bug

# 重复直到找到第一个 bad commit
# ... 几轮之后 ...
# abc123 is the first bad commit

# 结束 bisect
git bisect reset
```

## 1.3 实际示例

```bash
# 场景：发现当前版本有一个回归 bug
git log --oneline
# d4e5f6 (HEAD) refactor: optimize query
# c3d4e5 feat: add search
# b2c3d4 fix: memory leak
# a1b2c3 feat: init

# 开始二分查找
git bisect start
git bisect bad                 # HEAD 有 bug
git bisect good a1b2c3         # a1b2c3 没有 bug

# Bisecting: 2 revisions left to test after this (roughly 2 steps)
# Git checkout 了 b2c3d4

# 测试 b2c3d4（检查是否有 bug）
npm test
git bisect good                # b2c3d4 没问题

# Bisecting: 1 revision left to test after this
# Git checkout 了 c3d4e5

# 测试 c3d4e5
npm test
git bisect bad                 # c3d4e5 有问题！

# abc123def456 is the first bad commit
# commit abc123def456
# Author: John <john@example.com>
# Date: 2024-01-15
# feat: add search

# 发现 bug！是 "feat: add search" 引入的
# 结束 bisect
git bisect reset
```

## 1.4 自动 Bisect

```bash
# 使用脚本自动运行 bisect
git bisect start HEAD v1.0.0
git bisect run npm test        # 自动运行测试，根据退出码判断 good/bad
git bisect reset

# 使用自定义脚本
git bisect start HEAD v1.0.0
git bisect run ./bisect-test.sh

# bisect-test.sh 示例：
#!/bin/bash
npm run build
npm run test:e2e
# 退出码 0 = good，非 0 = bad
```

## 1.5 Bisect 的高级选项

```bash
# 指定更多 good/bad 边界（加速查找）
git bisect bad HEAD HEAD~2     # 最近 3 个 commit 都有 bug
git bisect good v1.0 v1.1 v1.2 # 多个 good 版本

# 跳过无法测试的 commit
git bisect skip                # 跳过当前 commit
git bisect skip abc123 def456  # 跳过指定 commit

# 查看 bisect 进度
git bisect visualize           # 用 git log 显示剩余范围
git bisect log                 # 显示 bisect 日志

# 使用参数指定 bisect 的范围
git bisect start --first-parent # 只在第一父分支上查找（忽略合并提交）
```

# 2. Git Blame — 代码追溯

## 2.1 基本用法

```bash
# 查看每行代码的提交信息
git blame file.txt

# 输出格式：
# abc1234 (Author Name 2024-01-15 10:30:00 +0800 10)   return value;
# ^^^^^^^ ^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^   ^^^^^^^^^^^^^^
#  commit   作者             时间戳              行号       代码

# 查看一定行数范围
git blame -L 10,30 file.txt

# 忽略空白变更
git blame -w file.txt

# 显示更多上下文
git blame -L 5,+15 file.txt   # 从第 5 行开始，显示 15 行
git blame -L '/function foo/,/^}/' file.txt  # 通过正则匹配范围
```

## 2.2 高级 Blame 用法

```bash
# 显示完整的 commit hash
git blame -l file.txt

# 显示作者邮箱
git blame -e file.txt

# 只显示修改时间（而非提交时间）
git blame --date=relative file.txt

# 追踪重命名
git blame -C file.txt           # 检测同一文件内的代码移动
git blame -C -C file.txt        # 检测同一 commit 内跨文件移动
git blame -C -C -C file.txt     # 检测跨 commit 的代码移动

# 只显示指定作者的行
git blame file.txt | grep "John"

# 忽略某些提交的 blame（如大规模格式化）
git blame --ignore-rev abc123 file.txt
git blame --ignore-revs-file .git-blame-ignore-revs file.txt

# .git-blame-ignore-revs 文件内容：
# abc123def456  # 格式化整个项目
# def456abc123  # 重命名变量
```

## 2.3 Blame 实战场景

```bash
# 场景：找到引入特定代码的人
git blame src/app.js
# 定位到可疑行
git show abc123  # 查看完整提交

# 场景：找到某行代码的历史变迁
git log --follow -p -S 'specific_code' -- src/app.js

# 场景：生成贡献统计
git blame --line-porcelain src/app.js | grep "^author " | sort | uniq -c | sort -rn
```

# 3. Git Grep — 代码搜索

## 3.1 基本用法

```bash
# 在工作树中搜索
git grep "function_name"
git grep -n "function_name"     # 显示行号
git grep -i "function_name"     # 忽略大小写

# 在特定提交中搜索
git grep "function_name" HEAD
git grep "bug" v1.0.0

# 在历史中搜索
git grep "old_api" $(git rev-list --all)  # 在所有提交中搜索
```

## 3.2 高级搜索

```bash
# 使用正则表达式
git grep -E "(TODO|FIXME|HACK)"
git grep -P "(?<=public )\w+\(\)"  # Perl 兼容正则

# 显示行号、文件名、匹配数
git grep -c "error"       # 每个文件匹配行数
git grep -l "error"       # 只显示文件名
git grep --count "error"  # 统计总匹配数

# 搜索并显示上下文
git grep -C 3 "error" src/   # 前后 3 行
git grep -B 5 "error" src/   # 前 5 行
git grep -A 5 "error" src/   # 后 5 行

# 限制文件类型
git grep "error" -- '*.java'
git grep "TODO" -- ':!*.min.js'  # 排除 min 文件
```

# 4. Git Log 高级查询

## 4.1 Pickaxe 搜索

```bash
# 搜索代码的添加或删除（不仅包含）
git log -S "function_name"          # 精确匹���
git log -S "function_name" --all    # 在所有分支中搜索
git log -S "function_name" -p       # 显示对应的 diff

# 正则表达式搜索
git log -G "function\s+\w+\(\)"     # 正则匹配

# 两者的区别：
# -S：某行被添加或删除（即使只有一次）
# -G：匹配正则的行发生变化（更精确）
```

## 4.2 按变更模式搜索

```bash
# 查找添加了特定代码的提交
git log --all --oneline -S "security_vulnerability"

# 查找修改了特定文件的提交
git log --all --oneline -- src/security.js

# 复杂查询组合
git log --all --oneline \
  --author="john" \
  --since="2024-01-01" \
  --until="2024-06-30" \
  -S "vulnerability" \
  -- '*.java' \
  -p
```

# 5. Debug 组合拳

## 5.1 排查回归 bug 的完整流程

```bash
# 1. 确认 bug 在当前版本存在
npm run test:e2e
# FAIL

# 2. 找到最后一个正常版本
git log --oneline -10

# 3. 使用 bisect 定位
git bisect start HEAD v1.2.0
git bisect run npm run test:e2e

# 4. 找到引入 bug 的 commit
# abc123 is the first bad commit

# 5. 查看 commit 的详细信息
git show abc123

# 6. 查看该 commit 的 blame 信息
git blame abc123^ -- src/buggy-file.js  # 查看修改前的状态

# 7. 查看所有相关的更改
git log abc123^..abc123 -p --stat

# 8. 结束 bisect
git bisect reset
```

## 5.2 代码追责流程

```bash
# 1. 找到有问题的代码
git grep "known_bug_pattern" src/

# 2. 查看是谁最后修改了这行
git blame -L 42,42 src/app.js

# 3. 查看完整的提交内容
git show abc123

# 4. 查看该文件的历史变更
git log --oneline --follow src/app.js

# 5. 查看提交的关联信息（PR、Issue 等）
git notes show abc123

# 6. 检查是否有后续修复
git log --all --oneline --grep="abc123"  # 搜索引用该 commit 的提交
```

## 5.3 大范围历史分析

```bash
# 统计代码量变化
git log --format='%aN' | sort -u | while read name; do
  echo -en "$name\t"
  git log --author="$name" --pretty=tformat: --numstat \
    | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added: %s, removed: %s, net: %s", add, subs, loc }'
  echo
done

# 查找最常修改的文件
git log --pretty=format: --name-only | sort | uniq -c | sort -rn | head -20
```
