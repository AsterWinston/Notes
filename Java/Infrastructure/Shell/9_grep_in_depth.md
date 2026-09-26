
# 1. grep 家族

| 命令 | 全称 | 默认正则 | 说明 |
|------|------|----------|------|
| `grep` | Global Regular Expression Print | BRE | 基本正则表达式 |
| `egrep` | Extended grep | ERE | 等价 `grep -E` |
| `fgrep` | Fixed grep | 无（字面量） | 等价 `grep -F` |
| `rgrep` | Recursive grep | BRE | 等价 `grep -r` |
| `pgrep` | Process grep | — | 按进程名查找 PID（属于 procps 包） |

```bash
# 现代写法（推荐统一用 grep + 选项）
grep -E "pattern"    # 代替 egrep
grep -F "literal"    # 代替 fgrep
grep -r "text" dir/  # 代替 rgrep
```

# 2. 核心选项

## 2.1 匹配控制

```bash
# -i    忽略大小写
grep -i "error" log.txt

# -v    反向匹配（显示不匹配的行）
grep -v "#" config.conf        # 去掉注释行
grep -v -e "^$" file.txt       # 去掉空行

# -w    匹配整个单词
echo "the cat and catalog" | grep -w "cat"    # 只匹配 cat

# -x    匹配整行
grep -x "exact line content" file.txt

# -E    扩展正则表达式
grep -E "error|warn" log.txt

# -F    固定字符串（不解析正则）
grep -F "*.txt" file.txt       # 匹配字面量 *.txt

# -P    Perl 兼容正则（GNU grep）
grep -P "\d{4}-\d{2}-\d{2}" log.txt
```

## 2.2 输出控制

```bash
# -c    显示匹配行数
grep -c "ERROR" app.log

# -o    只输出匹配到的部分（不是整行）
echo "IP: 192.168.1.1" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'
# 192.168.1.1

# -n    显示行号
grep -n "TODO" *.java

# -l    只列出文件名（不显示内容）
grep -l "FIXME" *.c

# -L    列出不包含匹配的文件名
grep -L "Copyright" *.c

# -m N  最多匹配 N 行后停止
grep -m 5 "error" huge.log    # 找到前 5 个就停

# --color=auto  高亮匹配内容（通常已在 alias 中）
alias grep='grep --color=auto'
```

## 2.3 上下文控制

```bash
# -A N  After：显示匹配行及其后 N 行
grep -A 3 "error" log.txt

# -B N  Before：显示匹配行及其前 N 行
grep -B 2 "error" log.txt

# -C N  Context：显示匹配行及其前后各 N 行
grep -C 5 "error" log.txt
```

## 2.4 文件与递归

```bash
# -r / -R  递归搜索
grep -r "pattern" ./src/
grep -R "pattern" ./src/       # -R 追踪符号链接

# --include / --exclude  过滤文件
grep -r --include "*.java" "TODO" ./src/
grep -r --exclude "*.log" "error" /var/log/
grep -r --exclude-dir={.git,node_modules} "main" .

# 使用 find 配合 grep（更灵活）
find . -name "*.java" -exec grep -l "pattern" {} +
find . -name "*.log" -mtime -7 -exec grep "ERROR" {} +
```

## 2.5 其他常用选项

```bash
# -H    始终显示文件名（默认多文件时自动显示）
grep -H "text" file.txt

# -h    不显示文件名
grep -h "text" *.log

# -q    静默模式（只返回退出状态，不输出）
if grep -q "pattern" file.txt; then
    echo "找到了"
fi

# -s    不显示错误信息（文件不存在等）
grep -s "text" *.txt

# -Z    文件名用 \0 分隔（配合 xargs -0）
grep -lZ "pattern" *.txt | xargs -0 rm

# -b    显示匹配在文件中的字节偏移
grep -b "pattern" file.txt
```

# 3. grep 的正则表达式

## 3.1 BRE vs ERE

```bash
# BRE 中 ( ) { } + ? | 需要转义才有特殊含义
grep '\(group\)\+'       # (group) 后的 + 表示一次或多次

# ERE 中 ( ) { } + ? | 直接有特殊含义
grep -E '(group)+'       # 更简洁直观

# ERE 更接近 Python/JavaScript 的正则体验
```

| 元字符 | BRE | ERE |
|--------|-----|-----|
| `.` 任意字符 | 是 | 是 |
| `*` 零或多次 | 是 | 是 |
| `+` 一或多次 | `\+` | `+` |
| `?` 零或一次 | `\?` | `?` |
| `|` 或 | `\|` | `|` |
| `()` 分组 | `\(\)` | `()` |
| `{}` 量词 | `\{n,m\}` | `{n,m}` |

## 3.2 常用正则模式

```bash
# 匹配 IP 地址
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' file.txt

# 匹配邮箱（简化版）
grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt

# 匹配 URL
grep -oE 'https?://[^ ]+' file.txt

# 匹配日期 YYYY-MM-DD
grep -oE '[12][0-9]{3}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])' file.txt

# 匹配十六进制颜色
grep -oE '#[0-9a-fA-F]{6}' file.txt

# 匹配非空非注释行
grep -vE '^\s*(#|$)' config.conf

# 匹配中文字符
grep -P '[\x{4e00}-\x{9fff}]' file.txt
```

# 4. grep 性能优化

## 4.1 匹配方式对性能的影响

```bash
# 慢 > 快 的顺序：
# 纯正则 > 固定字符串 > 按行匹配（LC_ALL=C 时更快）

# 固定字符串模式（最快）
grep -F "exact string" huge.log

# 设置 LC_ALL=C （跳过 Unicode 处理，提速 10-100 倍）
LC_ALL=C grep "pattern" huge.log

# 对于非常大的文件：先 index 再查
# 或使用专门工具：ag (silver searcher)、rg (ripgrep)、ack
```

## 4.2 巧用 -m 和 -l 提前退出

```bash
# -m N：找到 N 个匹配后停止读取
grep -m 5 "error" 100GB.log    # 立即停止

# -l：找到第一个匹配后就跳到下一个文件
grep -rl "FIXME" src/          # 每个文件只找第一个
```

# 5. 实战案例

## 5.1 日志分析

```bash
# 统计错误出现次数
grep -c "ERROR" app.log

# 统计每类错误的次数
grep -oE "ERROR\s+\[[A-Z]+" app.log | sort | uniq -c | sort -rn

# 查看指定时间段的错误
grep "2025-05-17 1[4-5]:" app.log | grep ERROR

# 排除已知的无用错误
grep ERROR app.log | grep -v "expected\|normal"

# 找出包含堆栈的完整错误块
grep -n "ERROR" app.log | head -1
# 记下行号后：sed -n '1000,1050p' app.log
```

## 5.2 代码搜索

```bash
# 搜索函数定义
grep -rn "^def " --include "*.py" src/

# 搜索实现了某接口的类
grep -rn "implements.*Serializable" --include "*.java" src/

# 搜索 TODO 并按文件统计
grep -rc "TODO" --include "*.java" src/ | sort -t: -k2 -rn

# 搜索不在注释中的 import
grep -r "^import" --include "*.java" src/ | grep -v "//\s*import"
```

## 5.3 系统管理

```bash
# 查看监听的端口
netstat -tlnp 2>/dev/null | grep LISTEN

# 查看特定进程
ps aux | grep -E "java|python|node"

# 查看磁盘使用 > 80%
df -h | awk '$5+0>80 {print}'

# 检查 /var/log 下最近 7 天修改的文件中的错误
find /var/log -mtime -7 -type f | xargs grep -l "ERROR"
```

## 5.4 多模式匹配

```bash
# -e 指定多个模式（或关系）
grep -e "error" -e "warn" -e "fatal" app.log

# 从文件读取模式列表（每行一个模式）
grep -f patterns.txt data.txt

# 管道串联多个 grep（且关系）
grep "ERROR" app.log | grep -v "ignored" | grep "Database"

# 匹配多个模式并提取
grep -oP '(error|warn|fatal) \[.*?\]' app.log
```

# 6. 退出状态码的应用

```bash
# grep 的退出码：
# 0 = 找到匹配
# 1 = 未找到匹配
# 2 = 错误（文件不存在等）

# 经典用法
if grep -q "pattern" file.txt; then
    echo "找到"
fi

# 检测命令是否执行成功
if ! mount | grep -q "/mnt/data"; then
    echo "挂载点未挂载"
fi

# 配合管道
if dmesg | grep -qi "error"; then
    echo "内核日志中有错误"
fi
```

# 7. 现代替代工具

| 工具 | 语言 | 特点 | 速度(相对grep) |
|------|------|------|----------------|
| `ripgrep (rg)` | Rust | 极其快，感知 .gitignore | 10-30x |
| `ag` | C | 快，代码搜索优先 | 5-10x |
| `ack` | Perl | 比 grep 快，代码搜索 | 2-5x |
| `git grep` | C | 只搜索 Git 管理的文件 | 快 |

```bash
# ripgrep 示例（推荐作为日常代码搜索工具）
rg TODO src/
rg -l "pattern" --type java
rg -C 3 "regex" src/
rg --files-without-match "passed" test-results/
```
