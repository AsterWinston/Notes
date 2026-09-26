
# 1. 基本内容查看

## 1.1 cat — 连接并显示

```bash
cat file.txt                      # 显示文件全部内容
cat -n file.txt                   # 显示行号（包括空行）
cat -b file.txt                   # 显示行号（不包括空行）
cat -s file.txt                   # 压缩连续空行为一行
cat file1.txt file2.txt           # 拼接多个文件
cat file1.txt file2.txt > merged.txt  # 合并到新文件
```

## 1.2 less — 分页查看（推荐）

```bash
less file.txt                     # 打开文件分页查看
```

less 操作快捷键：

| 按键 | 作用 |
|------|------|
| `Space` / `f` | 下一页 |
| `b` | 上一页 |
| `j` / `k` | 向下/向上滚动一行 |
| `g` | 跳到文件开头 |
| `G` | 跳到文件末尾 |
| `/pattern` | 向下搜索 |
| `?pattern` | 向上搜索 |
| `n` / `N` | 下一个/上一个匹配 |
| `q` | 退出 |
| `h` | 帮助 |
| `F` | 跟踪模式（类似 tail -f） |

## 1.3 more — 简单分页

```bash
more file.txt                     # 基本分页查看
```

less 是 more 的增强版（**less is more**），日常推荐使用 less。

## 1.4 head 和 tail — 查看首尾

```bash
head file.txt                     # 显示前10行
head -n 20 file.txt               # 显示前20行
head -c 100 file.txt              # 显示前100字节

tail file.txt                     # 显示后10行
tail -n 20 file.txt               # 显示后20行
tail -f file.txt                  # 实时跟踪文件追加内容（follow）
tail -F file.txt                  # 跟踪文件，处理日志轮替
tail -f access.log | grep "404"   # 实时过滤日志
```

**tail -f 原理**：默认每秒检查一次文件变更，读取新增内容。`-F` 选项在文件被轮替后重新打开同名文件。

# 2. 内容统计与比较

## 2.1 wc — 计数

```bash
wc file.txt                       # 输出：行数 单词数 字节数
wc -l file.txt                    # 只统计行数
wc -w file.txt                    # 只统计单词数
wc -c file.txt                    # 只统计字节数
wc -m file.txt                    # 只统计字符数（考虑多字节字符）
```

## 2.2 sort — 排序

```bash
sort file.txt                     # 按字典序排序
sort -n file.txt                  # 按数值排序
sort -r file.txt                  # 逆序排序
sort -u file.txt                  # 排序并去重（unique）
sort -k2 file.txt                 # 按第二列排序
sort -t: -k3 -n /etc/passwd      # 以冒号分隔，按第三列数值排序
sort -h file.txt                  # 按人类可读数字排序（2K < 1M < 2G）
```

## 2.3 uniq — 去重与计数

```bash
uniq file.txt                     # 去除相邻重复行
uniq -c file.txt                  # 显示重复次数（前缀）
uniq -d file.txt                  # 仅显示重复行
uniq -u file.txt                  # 仅显示不重复的行

# 重要：uniq 只去除相邻重复行，一般先 sort
sort file.txt | uniq -c | sort -nr  # 统计各行出现次数，从高到低
```

## 2.4 diff — 比较文件差异

```bash
diff file1.txt file2.txt          # 显示差异
diff -u file1.txt file2.txt       # 统一格式（unified，补丁常用）
diff -r dir1/ dir2/               # 递归比较目录
diff -q file1.txt file2.txt       # 仅报告是否相同
```

## 2.5 cut — 截取字段

```bash
cut -d: -f1 /etc/passwd           # 以冒号分割，取第一个字段（用户名）
cut -d: -f1,3 /etc/passwd        # 取第1和第3字段
cut -d: -f1-3 /etc/passwd        # 取第1到第3字段
cut -c1-10 file.txt               # 取每行前10个字符
cut -c3- file.txt                 # 从第3个字符到行尾
```

**常见模式**：
```bash
# 提取所有用户名
cut -d: -f1 /etc/passwd

# 提取 IP 地址
ip addr show | grep "inet " | awk '{print $2}' | cut -d/ -f1
```

# 3. grep — 文本搜索

## 3.1 基本用法

```bash
grep "pattern" file.txt           # 在文件中搜索模式
grep "error" /var/log/syslog      # 日志搜索
```

## 3.2 常用选项

```bash
grep -i "pattern" file.txt        # 忽略大小写
grep -v "pattern" file.txt        # 反向匹配（不包含的行）
grep -n "pattern" file.txt        # 显示行号
grep -c "pattern" file.txt        # 只统计匹配行数
grep -r "pattern" /path/          # 递归搜索目录
grep -l "pattern" *.txt           # 仅显示匹配的文件名
grep -w "pattern" file.txt        # 全词匹配（不匹配子串）
grep -x "pattern" file.txt        # 整行精确匹配
grep -A3 "pattern" file.txt       # 显示匹配行及其后3行（After）
grep -B2 "pattern" file.txt       # 显示匹配行及其前2行（Before）
grep -C2 "pattern" file.txt       # 显示匹配行及其前后各2行（Context）
```

## 3.3 正则表达式

```bash
# 基础正则（BRE）
grep "^root" /etc/passwd          # 以 root 开头的行
grep "bash$" /etc/passwd          # 以 bash 结尾的行
grep "r.t" file.txt               # r 任意字符 t
grep "[0-9]\{3\}" file.txt        # 连续3个数字（BRE 需要转义 {}）

# 扩展正则（ERE）
grep -E "[0-9]{3}" file.txt       # 连续3个数字（ERE 不需要转义）
grep -E "error|fail" file.txt     # 匹配 error 或 fail
grep -E "colou?r" file.txt        # 匹配 color 或 colour
grep -P "\d{3}-\d{4}" file.txt    # Perl 兼容正则（PCRE）
```

## 3.4 实用案例

```bash
# 查找所有包含 TODO 的源文件
grep -rn "TODO" --include="*.py" --include="*.java" .

# 排除目录
grep -r "password" --exclude-dir=node_modules .

# 在进程中查找
ps aux | grep nginx

# 只看 IP 地址
grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" access.log

# 统计错误数量
grep -c "ERROR" app.log
```

# 4. sed — 流编辑器

## 4.1 基本语法

```bash
sed '操作' file.txt               # 处理并输出到 stdout（不修改原文件）
sed -i '操作' file.txt            # 原地修改文件
sed -i.bak '操作' file.txt        # 修改前备份原文件
```

## 4.2 替换（最常用）

```bash
sed 's/old/new/' file.txt                   # 替换每行第一个匹配
sed 's/old/new/g' file.txt                  # 替换所有匹配（全局）
sed 's/old/new/2' file.txt                  # 替换每行第二个匹配
sed 's/old/new/gI' file.txt                 # 忽略大小写
sed 's/old/new/gw output.txt' file.txt      # 替换并写入输出文件
sed '3s/old/new/g' file.txt                 # 只对第3行操作
sed '2,5s/old/new/g' file.txt               # 只对第2-5行操作
sed '/pattern/s/old/new/g' file.txt         # 只对匹配 pattern 的行操作
sed 's/\/usr\/local/\/opt/g' file.txt       # 用 \ 转义分隔符
sed 's|/usr/local|/opt|g' file.txt          # 改用 | 作为分隔符（更清晰）
```

## 4.3 行操作

```bash
sed '3d' file.txt                 # 删除第3行
sed '2,5d' file.txt               # 删除第2-5行
sed '/^#/d' file.txt              # 删除注释行（以 # 开头）
sed '/^$/d' file.txt              # 删除空行
sed '3i\插入的行' file.txt       # 在第3行前插入
sed '3a\追加的行' file.txt       # 在第3行后追加
sed '3c\替换的行' file.txt       # 替换第3行
```

## 4.4 实用案例

```bash
# 去除行首空格
sed 's/^[[:space:]]*//' file.txt

# 去除行尾空格
sed 's/[[:space:]]*$//' file.txt

# 压缩连续空行为一个空行
sed '/^$/{N;/^\n$/d}' file.txt

# 打印文件的第5-10行
sed -n '5,10p' file.txt

# 在每行前加行号
sed = file.txt | sed 'N;s/\n/\t/'

# 多命令执行
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt
sed 's/foo/bar/g; s/baz/qux/g' file.txt
```

# 5. awk — 文本分析工具

## 5.1 基本模式

```bash
awk 'pattern {action}' file.txt
```

awk 的核心思想：将输入逐行分割为字段，对匹配模式的行执行动作。

## 5.2 内置变量

| 变量 | 含义 |
|------|------|
| `$0` | 整行内容 |
| `$1`, `$2`, ... | 第1、2...个字段 |
| `NF` | 字段数量 |
| `NR` | 当前行号 |
| `FS` | 字段分隔符（默认空白字符） |
| `OFS` | 输出字段分隔符（默认空格） |
| `RS` | 记录分隔符（默认换行符） |
| `ORS` | 输出记录分隔符（默认换行符） |

## 5.3 基本用法

```bash
awk '{print $1}' file.txt                 # 打印每行第一个字段
awk '{print $1, $3}' file.txt             # 打印第1和第3字段
awk '{print NR, $0}' file.txt             # 打印行号和整行

# 指定分隔符
awk -F: '{print $1}' /etc/passwd          # 以冒号分割，取用户名
awk -F: '{print $1, $6}' /etc/passwd      # 用户名和家目录

# 条件过滤
awk '/error/ {print}' /var/log/syslog     # 仅打印包含 error 的行
awk '$3 > 50 {print $1, $3}' data.txt     # 第3列大于50的行
awk 'NR > 1 && NR < 10 {print}' file.txt  # 第2-9行
```

## 5.4 内置函数

```bash
awk '{print length($0)}' file.txt         # 每行长度
awk '{print toupper($1)}' file.txt        # 转大写
awk '{print substr($1, 1, 3)}' file.txt   # 取前3字符
awk '{printf "%-10s %5d\n", $1, $2}'      # 格式化输出
```

## 5.5 BEGIN 和 END

```bash
# 在开始处理前和执行结束后执行
awk 'BEGIN {print "=== Report ==="} 
     {print $1, $3} 
     END {print "=== End ==="}' file.txt

# 汇总统计
awk '{sum += $1} END {print "Total:", sum}' numbers.txt
awk '{count++} END {print "Lines:", count}' file.txt
awk '{sum+=$1; count++} END {print "Avg:", sum/count}' numbers.txt
```

## 5.6 实用案例

```bash
# 列出所有用户名和 Shell
awk -F: '{printf "%-12s %s\n", $1, $7}' /etc/passwd

# 统计 HTTP 状态码
awk '{status[$9]++} END {for (s in status) print s, status[s]}' access.log

# 找出访问量最大的 IP 前10
awk '{ips[$1]++} END {for (ip in ips) print ips[ip], ip}' access.log | sort -rn | head -10

# 计算平均响应时间
awk '{sum+=$NF} END {print "Avg:", sum/NR}' response_times.txt
```

# 6. tr — 字符转换

```bash
tr 'a-z' 'A-Z' < file.txt                 # 小写转大写
tr -d ' ' < file.txt                      # 删除空格
tr -s '\n' < file.txt                     # 压缩连续换行为一个
tr ',' '\t' < file.txt > output.txt       # 逗号转制表符
tr -dc 'a-zA-Z0-9' < file.txt            # 只保留字母数字（删除其他）

# 删除 Windows 行尾的 ^M 字符
tr -d '\r' < win.txt > unix.txt
```
