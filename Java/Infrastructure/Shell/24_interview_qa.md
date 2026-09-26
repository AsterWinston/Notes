
# 1. 基础概念题

## 1.1 什么是 Shell？Shell 和终端有什么区别？

**Shell** 是一个**命令解释器**，作为用户与操作系统内核之间的接口层。它接收用户输入的命令，将其翻译为系统调用交由内核执行，最后将结果返回给用户。常见的 Shell 有 bash、zsh、fish、dash、ksh 等。

**终端（Terminal）** 是一个**输入输出设备**（物理或虚拟），负责接收键盘输入和显示文本输出。终端本身不懂命令，它只是一个字符界面的呈现载体。

核心区别：

| 概念 | 本质 | 类比 |
|------|------|------|
| 终端 | 硬件/模拟硬件设备，提供输入输出通道 | 显示器 + 键盘 |
| Shell | 运行在终端内的程序，解释命令 | 操作系统内的命令行程序 |
| 终端模拟器 | GUI 程序，模拟传统物理终端 | iTerm2、Windows Terminal、GNOME Terminal |

```bash
# 查看当前运行的终端
echo $TERM           # xterm-256color
tty                  # /dev/pts/0

# 查看当前使用的 Shell
echo $SHELL          # /bin/bash
echo $0              # -bash（登录 shell）或 bash
ps -p $$             # 查看当前进程
```

**常见误解澄清：**

```bash
# "打开终端"实际做了三件事：
# 1. 终端模拟器启动（分配一个 PTY 伪终端设备，如 /dev/pts/1）
# 2. 启动默认 Shell（bash/zsh/...），将其 stdin/stdout/stderr 绑定到该 PTY
# 3. Shell 显示提示符，等待用户输入

# bash 和 Terminal 的区别：
# bash 是程序，Terminal 是容器——同一个终端可以运行不同的 Shell
exec zsh             # 在同一个终端中切换到 zsh
exec fish            # 切换到 fish
```

## 1.2 `$@` 和 `$*` 有什么区别？

两者都表示**所有位置参数**，但在被双引号包裹时行为完全不同：

| 写法 | 展开行为 | 实际效果 |
|------|----------|----------|
| `$@` | 每个参数独立单词（word splitting） | `"$1" "$2" "$3" ...` |
| `$*` | 所有参数合并为一个字符串（空格分隔） | `"$1 $2 $3 ..."` |
| `"$@"` | **每个参数独立引用**（推荐） | `"$1" "$2" "$3" ...` |
| `"$*"` | **所有参数作为一个字符串**，用 IFS 首字符连接 | `"$1c$2c$3..."`（c 为 IFS 首字符） |

```bash
# 演示脚本
#!/bin/bash
# 假设调用：./test.sh "a b" "c d" "e f"

echo "=== \$@ (无引号) ==="
for arg in $@; do echo "[$arg]"; done
# [a] [b] [c] [d] [e] [f] —— 因为 word splitting 把引号内的空格也拆了

echo "=== \"\$@\" (有引号) ==="
for arg in "$@"; do echo "[$arg]"; done
# [a b] [c d] [e f] —— 正确保持了每个参数的原形

echo "=== \"\$*\" (有引号) ==="
for arg in "$*"; do echo "[$arg]"; done
# [a b c d e f] —— 所有参数合并为一个（IFS 默认空格分隔）

# 修改 IFS 看 "$*" 的行为
IFS=","
echo "$*"
# a b,c d,e f —— 用逗号连接

# 实际使用场景
# 1. 传递参数给其他命令（推荐 "$@"）
some_command "$@"

# 2. 以 "$*" 做日志输出
echo "参数列表：$*" >> script.log
```

**`$@` 的最大坑：不加引号时等同于 `$*`（不加引号时行为）——二者都会做 word splitting 和 globbing。**

## 1.3 单引号、双引号、反引号有什么区别？

| 引号类型 | 变量展开 | 命令替换 | 转义字符 | 用途 |
|----------|----------|----------|----------|------|
| `'...'` 单引号 | 不展开 | 不执行 | 全部原样（`'` 自身除外） | 字面量字符串 |
| `"..."` 双引号 | 展开 `$var` | 执行 `$(cmd)` | 部分转义（`$` `` ` `` `"` `\`） | 带变量插值的字符串 |
| `` `...` `` 反引号 | — | 执行命令 | — | 命令替换（已过时，用 `$()` 代替） |

```bash
name="world"
echo 'Hello $name'     # Hello $name   （变量未展开）
echo "Hello $name"     # Hello world   （变量已展开）

# 转义差异
echo 'a\tb'            # a\tb          （\t 原样输出）
echo "a\tb"            # a\tb          （\t 在双引号中不特殊处理，但 $echo 默认不解析）
echo -e "a\tb"         # a    b        （-e 开启转义解析）
echo $'a\tb'           # a    b        （$'...' ANSI-C 引用）

# 命令替换
echo "Today is $(date)"   # Today is Sun May 17 ... （现代写法，推荐）
echo "Today is `date`"    # Today is Sun May 17 ... （过时写法，不推荐）

# 嵌套问题
echo "Result: $(echo $(whoami))"     # 可以嵌套，清晰
echo "Result: `echo \`whoami\``"    # 需要转义内部反引号，混乱且容易出错
```

```bash
# 最佳实践
# 1. 默认用双引号包裹变量，防止空变量和通配符问题
file="my file.txt"
rm "$file"            # 正确
# rm $file            # 展开为 rm my file.txt —— 错误

# 2. 纯字面量用单引号
awk '{print $1}' data.txt    # $1 不会被 bash 展开

# 3. 命令替换用 $() 而非反引号
result=$(ls /tmp)
```

## 1.4 `[ ]` 和 `[[ ]]` 有什么区别？

**本质区别：`[` 是一个命令（builtin），`[[` 是一个关键字（keyword）。**

| 特性 | `[ ]` | `[[ ]]` |
|------|-------|---------|
| 本质 | 命令（`/usr/bin/[` 或 builtin） | Bash 关键字 |
| POSIX 兼容 | 是 | 否（bash/zsh 专有） |
| 变量引号 | 必须加引号防裂 | 无需引号（自动保护） |
| 通配符匹配 | 不支持（需要 `expr`） | 支持 `==` 右侧的 `* ?` 匹配 |
| 正则匹配 | 不支持 | 支持 `=~` 运算符 |
| 逻辑运算 | `-a` `-o` | `&&` `||`（直觉式，且短路求值） |
| `<` `>` 比较 | 被解释为重定向 | 被解释为字符串字典序比较 |
| 空变量安全 | 不安全 | 安全 |

```bash
type [       # [ is a shell builtin
type [[      # [[ is a shell keyword

# === [ ] 的坑 ===
# 1. 空变量导致语法错误
var=""
[ $var = "abc" ]          # 展开后：[ = abc ] —— 语法错误！
[ "$var" = "abc" ]        # 正确：引号保护
[[ $var = "abc" ]]        # 正确：关键字自动保护

# 2. -a 和 -o 的不确定行为
[ "$a" = "x" -a "$b" = "y" ]    # POSIX 标准说行为未定义
# 应改为：
[ "$a" = "x" ] && [ "$b" = "y" ]    # 两个独立 test

# 3. < 和 > 在 [ ] 中是 I/O 重定向
[ "b" > "a" ]              # 危险！会创建文件名为 "a" 的文件！
[[ "b" > "a" ]]            # 按字符串字典序比较

# === [[ ]] 的增强特性 ===
# 1. 模式匹配（通配符）
file="report.xml"
[[ "$file" == *.xml ]] && echo "XML 文件"     # 右侧支持通配符（无需引号）
# 等价于 case 语句
case "$file" in *.xml) echo "XML 文件" ;; esac

# 2. 正则匹配
phone="13812345678"
re='^1[3-9][0-9]{9}$'
[[ "$phone" =~ $re ]] && echo "手机号格式正确"

# 3. 短路逻辑
[[ -f "$file" && -s "$file" ]]   # 直觉式写法，且短路求值

# 4. 模式匹配的高级用法
path="/home/user/project"
[[ "$path" == /home/*/project ]] && echo "匹配"
[[ "$path" != /root/* ]] && echo "不在 root 目录"
```

**选型建议：**

- **交互终端脚本**：优先 `[[ ]]`（安全、强大）
- **需 POSIX 兼容**：`[ ]` + 始终加双引号
- **正则匹配 / 模式匹配**：必须 `[[ ]]`

## 1.5 `$()` 和反引号有什么区别？

两者都用于**命令替换**，但 `$()` 是现代 POSIX 标准写法，全面优于反引号。

| 特性 | `$(cmd)` | `` `cmd` `` |
|------|----------|-------------|
| 嵌套 | 直接嵌套：`$(cmd1 $(cmd2))` | 需要转义：`` `cmd1 \`cmd2\` ` `` |
| 可读性 | 清晰（括号匹配明显） | 差（反引号容易与单引号混淆） |
| 转义规则 | 内部自成作用域，转义规则直观 | 内部转义规则混乱，反斜杠行为不一致 |
| POSIX 标准 | POSIX 推荐 | POSIX 仍支持但标记为过时 |
| shell 兼容 | bash/zsh/ksh 原生支持 | 所有 Bourne 系列 shell 支持 |

```bash
# 嵌套对比（最具说服力的例子）
# $() 嵌套——清晰直观
result=$(echo "当前用户: $(whoami)，位于 $(pwd)")
echo "$result"

# 反引号嵌套——晦涩难懂
result=`echo "当前用户: \`whoami\`，位于 \`pwd\`"`
echo "$result"

# 深嵌套场景
# 三层的反引号几乎不可维护
packages=$(dpkg -l | grep $(uname -r | sed 's/-generic//'))
# vs
# packages=`dpkg -l | grep \`uname -r | sed 's/-generic//'\``

# 反斜杠在反引号中的混乱行为
echo `echo \\$var`       # 需要两个反斜杠
echo $(echo \$var)       # 一个反斜杠足矣

# 嵌套引号的问题
echo "Result: $(echo "$(whoami)")"   # 外层双引号，嵌套也能用双引号
# 反引号无法做到这一点
```

**总结：永远使用 `$()`，反引号仅在维护史前脚本时才会遇到。**

---

# 2. 脚本编写题

## 2.1 如何获取脚本所在目录？（多种方式对比）

这是一个看似简单但极易出错的问题。核心难点在于脚本可能通过符号链接、相对路径、`source` 等方式调用。

```bash
# 方式一：BASH_SOURCE + pwd（最推荐，解决 source 和符号链接问题）
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# 方式二：使用 $0（最简单，但有局限）
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# 方式三：readlink -f 处理符号链接（Linux）
SCRIPT_DIR="$(dirname "$(readlink -f "$0")")"

# 方式四：BASH_SOURCE + 符号链接解析（最通用的健壮版本）
get_script_dir() {
    local source="${BASH_SOURCE[0]}"
    while [ -h "$source" ]; do   # 如果 source 是符号链接
        local dir="$(cd -P "$(dirname "$source")" && pwd)"
        source="$(readlink "$source")"
        # 如果 readlink 返回相对路径，拼接成全路径
        [[ $source != /* ]] && source="$dir/$source"
    done
    cd -P "$(dirname "$source")" && pwd
}
SCRIPT_DIR="$(get_script_dir)"
```

```bash
# 完整对比与演示
cat > /tmp/test_dir.sh << 'SCRIPT'
#!/bin/bash

echo "========== 各种获取方式对比 =========="
echo "BASH_SOURCE[0]: ${BASH_SOURCE[0]}"
echo "\$0: $0"
echo "\$PWD: $PWD"
echo "pwd: $(pwd)"
echo "readlink -f \$0: $(readlink -f "$0" 2>/dev/null || echo 'N/A')"

echo ""
echo "========== 计算脚本目录 =========="
echo "方式1 (dirname \$0):              $(dirname "$0")"
echo "方式2 (dirname BASH_SOURCE):      $(dirname "${BASH_SOURCE[0]}")"
echo "方式3 (cd dirname; pwd):          $(cd "$(dirname "$0")" && pwd)"
echo "方式4 (cd dir; pwd -P 物理路径):  $(cd "$(dirname "$0")" && pwd -P)"
SCRIPT

# 场景测试：各种调用方式的行为差异
# bash /tmp/test_dir.sh        —— $0 是绝对路径
# cd /tmp && bash test_dir.sh  —— $0 是相对路径
# source /tmp/test_dir.sh      —— $0 是 -bash，BASH_SOURCE[0] 仍正确
# ./test_dir.sh（若为符号链接） —— $0 是链接路径，readlink -f 可解析真实路径
```

**`$0` vs `${BASH_SOURCE[0]}` 的关键区别：**

| 调用方式 | `$0` | `${BASH_SOURCE[0]}` |
|----------|------|---------------------|
| `bash script.sh` | `script.sh`（相对/绝对路径） | `script.sh` |
| `./script.sh` | `./script.sh` | `./script.sh` |
| `source script.sh` | `-bash`（当前 shell 名） | `script.sh` |
| 被其他脚本 source | `-bash`（不变） | 当前被执行脚本的路径 |

**所以 `${BASH_SOURCE[0]}` 始终指向当前脚本文件，`$0` 只在脚本作为进程被执行时才指向它。**

## 2.2 如何遍历数组？

```bash
#!/bin/bash

# 定义数组
fruits=("apple" "banana" "cherry" "date" "elderberry")
declare -A capitals=( ["中国"]="北京" ["日本"]="东京" ["法国"]="巴黎" )

echo "=== 方式一：for item in 遍历（最常用） ==="
for fruit in "${fruits[@]}"; do
    echo "水果：$fruit"
done

echo ""
echo "=== 方式二：按索引遍历 ==="
for i in "${!fruits[@]}"; do
    echo "索引 $i: ${fruits[$i]}"
done

echo ""
echo "=== 方式三：C 风格 for 循环（需要已知长度） ==="
for ((i=0; i<${#fruits[@]}; i++)); do
    echo "[$i] = ${fruits[$i]}"
done

echo ""
echo "=== 方式四：while + 索引计数器 ==="
i=0
while [ $i -lt ${#fruits[@]} ]; do
    echo "${fruits[$i]}"
    ((i++))
done

echo ""
echo "=== 关联数组遍历 ==="
for key in "${!capitals[@]}"; do
    echo "$key 的首都是 ${capitals[$key]}"
done

# 关键点
# 1. 遍历值：for item in "${arr[@]}"     —— 用 @，加引号，保护元素中的空格
# 2. 遍历索引：for i in "${!arr[@]}"     —— 用 ! 取键名/索引
# 3. 数组长度：${#arr[@]}
```

**`"${arr[@]}"` vs `"${arr[*]}"` 在 for 循环中的区别：**

```bash
arr=("a b" "c d" "e f")

# 正确：每个元素独立迭代
for item in "${arr[@]}"; do
    echo "[$item]"
done
# [a b]
# [c d]
# [e f]

# 错误：所有元素合并为一个
for item in "${arr[*]}"; do
    echo "[$item]"
done
# [a b c d e f]   —— 只循环一次

# 不加引号的后果：包含空格的元素被拆散
for item in ${arr[@]}; do
    echo "[$item]"
done
# [a] [b] [c] [d] [e] [f]
```

## 2.3 如何判断一个变量是否为空？

```bash
#!/bin/bash

var=""
unset_variable=""   # 故意不设置，让其为空

# === 方式一：-z（判断长度为零）—— 最常用 ===
if [ -z "$var" ]; then
    echo "var 为空"
fi

# === 方式二：-n（判断长度非零）—— -z 的反面 ===
if [ -n "$var" ]; then
    echo "var 不为空"
else
    echo "var 为空"
fi

# === 方式三：直接判断（利用 [ ] 对空字符串返回假） ===
if [ "$var" ]; then
    echo "var 不为空"
else
    echo "var 为空"
fi

# === 方式四：使用 [[ ]]（更安全，无需引号） ===
if [[ -z $var ]]; then    # [[ ]] 内可以不加引号
    echo "var 为空"
fi

# === 方式五：使用默认值展开 ===
echo "${var:-默认值}"              # 空时使用默认值
echo "${var:=默认值}"              # 空时赋值并使用
echo "${var:?变量未设置或为空}"     # 空时报错退出
echo "${var:+有值才显示}"           # 非空时使用替代值

# === 方式六：区分"未设置"与"空字符串" ===
# -z 不区分"未设置"和"空字符串"
# 若要区分：
if [ -z "${var+x}" ]; then   # ${var+x}：若 var 未设置则展开为空，否则展开为 x
    echo "var 未设置"
elif [ -z "$var" ]; then
    echo "var 已设置但值为空"
else
    echo "var = $var"
fi

# === 实际应用 ===
if [ -z "$CONFIG_FILE" ]; then
    CONFIG_FILE="/etc/app/config.yml"
    echo "未指定配置文件，使用默认：$CONFIG_FILE"
fi
```

**`-z` vs `-n` vs 直接判断对照表：**

| 条件 | var 为空 | var 为非空 |
|------|----------|-------------|
| `[ -z "$var" ]` | 真（退出码 0） | 假（退出码 1） |
| `[ -n "$var" ]` | 假 | 真 |
| `[ "$var" ]` | 假 | 真 |

**常见陷阱：**

```bash
var=""
# 陷阱一：不加引号
[ -n $var ]       # 展开为 [ -n ] —— 这是真！（-n 单独出现=测试非空字符串 "-n"=真）
[ -n "$var" ]     # 正确：结果为假

# 陷阱二：将空字符串误判为非空
var=" "
[ -z "$var" ]     # 假（空格是一个字符）
var="   "
[ -z "${var// }" ] # 通过去除空格后再判断
```

## 2.4 如何找出目录下最大的 10 个文件？

```bash
#!/bin/bash

TARGET_DIR="${1:-.}"    # 默认为当前目录
N="${2:-10}"            # 默认前 10 个

echo "在 $TARGET_DIR 中查找最大的 $N 个文件..."

# === 方式一：du + sort（推荐，含目录） ===
find "$TARGET_DIR" -type f -exec du -h {} + 2>/dev/null \
    | sort -rh \
    | head -n "$N"

# === 方式二：ls + sort（仅当前目录，不含子目录） ===
ls -lhS "$TARGET_DIR" 2>/dev/null | head -n "$((N+1))"
# -S 按大小排序，第一行是 total，所以 +1

# === 方式三：更精确地同时显示 KB/MB/GB ===
find "$TARGET_DIR" -type f -printf '%s\t%p\n' 2>/dev/null \
    | sort -rn \
    | head -n "$N" \
    | awk -F'\t' '{
        size=$1
        if (size>=1073741824)      printf "%.2f GB\t%s\n", size/1073741824, $2
        else if (size>=1048576)     printf "%.2f MB\t%s\n", size/1048576, $2
        else if (size>=1024)        printf "%.2f KB\t%s\n", size/1024, $2
        else                        printf "%d B\t%s\n", size, $2
      }'

# === 方式四：排除特定目录（如 .git、node_modules） ===
find "$TARGET_DIR" -type f \
    -not -path "*/.git/*" \
    -not -path "*/node_modules/*" \
    -exec du -h {} + 2>/dev/null \
    | sort -rh | head -n "$N"

# === 方式五：只统计特定扩展名文件 ===
find "$TARGET_DIR" -type f -name "*.log" -exec du -h {} + 2>/dev/null \
    | sort -rh | head -n "$N"
```

**`du` vs `ls` 的关键区别：**

- `du` 显示的是磁盘占用（disk usage），以 block 为单位，总是 4K 的整数倍
- `ls -l` 显示的是文件的逻辑大小（字节数），更精确
- `du -h` 的 `-h` 是人类可读格式（1K, 234M, 2G）
- `sort -h` 能理解这种人类可读格式进行排序

## 2.5 如何统计日志文件中每个 IP 的访问次数？

```bash
#!/bin/bash

LOG_FILE="${1:-/var/log/nginx/access.log}"

# === 场景一：Nginx/Apache 标准日志格式 ===
# 日志格式：192.168.1.1 - - [17/May/2026:10:00:00 +0800] "GET /index.html HTTP/1.1" 200 2326
echo "===== IP 访问次数 Top 10 ====="
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10

# 或者一步到位（只保留 awk）
awk '{count[$1]++} END {for(ip in count) print count[ip], ip}' "$LOG_FILE" \
    | sort -rn | head -10

# === 场景二：日志格式不标准（IP 不在第一列） ===
# 用 grep -o + 正则提取 IP
grep -oE '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' "$LOG_FILE" \
    | sort | uniq -c | sort -rn | head -10

# === 场景三：统计每个 IP 的流量（Nginx body_bytes_sent 通常在第 10 列） ===
awk '{
    ip=$1; bytes=$10
    count[ip]++; total_bytes[ip]+=bytes
} END {
    for(ip in count)
        printf "%8d 次  %10d 字节  %s\n", count[ip], total_bytes[ip], ip
}' "$LOG_FILE" | sort -rn | head -10

# === 场景四：统计特定时间段的 IP ===
# awk 支持时间范围过滤（以日志时间戳在 $4 为例）
START="10:00:00"
END="12:00:00"
awk -v s="$START" -v e="$END" '
    $4 >= s && $4 <= e {count[$1]++}
    END {for(ip in count) print count[ip], ip}
' "$LOG_FILE" | sort -rn | head -10

# === 场景五：实时监控模式（tail -f 配合） ===
tail -f "$LOG_FILE" | awk '{print $1}' | \
    while read ip; do
        # 实时处理
        echo "[$(date '+%H:%M:%S')] 来自 $ip 的请求"
    done
```

**`sort | uniq -c` vs `awk` 数组对比：**

| 方面 | `sort \| uniq -c` | `awk` 数组 |
|------|-------------------|-------------|
| 内存使用 | 管道流式，内存友好 | 所有数据加载到内存 |
| 大文件 | 适合超大文件 | 受内存限制 |
| 灵活性 | 简单统计 | 可做复杂聚合（求和、平均等） |
| 速度 | sort 开销大 | 遍历一次即可 |

```bash
# uniq -c 输出格式说明
echo -e "192.168.1.1\n192.168.1.2\n192.168.1.1" | sort | uniq -c
#       2 192.168.1.1
#       1 192.168.1.2
#        ^^^ 前面是次数，默认有前导空格，sort -rn 按数值逆序排列
```

## 2.6 编写一个带超时的函数

```bash
#!/bin/bash

# === 方式一：使用 timeout 命令（最简单，推荐） ===
run_with_timeout() {
    local timeout_sec="$1"
    shift
    timeout "$timeout_sec" "$@"
    local exit_code=$?
    if [ $exit_code -eq 124 ]; then
        echo "错误：命令超时（${timeout_sec} 秒）" >&2
        return 124
    fi
    return $exit_code
}

# 使用示例
# run_with_timeout 5 sleep 10    # 5 秒超时，sleep 10 会被杀掉
# run_with_timeout 10 curl https://example.com

# === 方式二：使用后台进程 + kill（兼容没有 timeout 命令的环境） ===
run_with_timeout_bg() {
    local timeout_sec="$1"
    shift
    local cmd="$@"

    # 后台执行命令
    eval "$cmd" &
    local cmd_pid=$!

    # 启动超时监控
    (
        sleep "$timeout_sec"
        kill -0 "$cmd_pid" 2>/dev/null && kill -TERM "$cmd_pid" 2>/dev/null
        sleep 1   # 给 TERM 信号一点时间
        kill -0 "$cmd_pid" 2>/dev/null && kill -KILL "$cmd_pid" 2>/dev/null
    ) &
    local watcher_pid=$!

    # 等待命令完成
    wait "$cmd_pid" 2>/dev/null
    local exit_code=$?

    # 清理监控进程
    kill "$watcher_pid" 2>/dev/null
    wait "$watcher_pid" 2>/dev/null

    if [ $exit_code -gt 128 ]; then
        echo "错误：命令超时并被终止（${timeout_sec} 秒）" >&2
        return 124
    fi
    return $exit_code
}

# === 方式三：使用 read -t（适用于读取输入超时） ===
prompt_with_timeout() {
    local timeout_sec="${1:-10}"
    local prompt="${2:-请输入：}"
    local var_name="${3:-user_input}"

    read -t "$timeout_sec" -p "$prompt" "$var_name"
    local exit_code=$?

    if [ $exit_code -eq 142 ]; then   # read -t 超时返回码 > 128
        echo ""
        echo "输入超时（${timeout_sec} 秒未响应）" >&2
        return 1
    fi
    return 0
}

# === 方式四：使用 trap + ALRM 信号（进程内超时） ===
run_with_alarm() {
    local timeout_sec="$1"
    shift

    # 设置 ALRM 信号处理器
    trap 'echo "错误：超时（${timeout_sec}s）" >&2; return 124' ALRM

    # 因 bash 不支持毫秒级 alarm，用子 shell + sleep 模拟
    (
        sleep "$timeout_sec" && kill -ALRM $$ &
    ) &
    local alarm_pid=$!

    "$@"
    local exit_code=$?

    # 清理
    kill "$alarm_pid" 2>/dev/null
    trap - ALRM

    return $exit_code
}

# === 完整测试 ===
echo "=== 测试方式一（timeout 命令） ==="
run_with_timeout 3 sleep 5
echo "退出码：$?"

echo ""
echo "=== 测试 prompt_with_timeout ==="
prompt_with_timeout 3 "3 秒内输入内容：" my_var
echo "输入的内容：${my_var:-无}"
```

**各方式对比：**

| 方式 | 环境要求 | 跨平台 | 适用场景 |
|------|----------|--------|----------|
| `timeout` 命令 | coreutils | GNU/Linux | 外部命令超时控制 |
| 后台进程 + kill | 纯 bash | 全平台 | POSIX 兼容环境 |
| `read -t` | bash builtin | bash | 交互式输入超时 |
| ALRM 信号 | bash | 全平台 | 函数内部超时 |

---

# 3. 文本处理题

## 3.1 如何用 sed 替换文件中的字符串？

```bash
# === 全局替换 ===
sed -i 's/old/new/g' file.txt
#          |   |  |
#          |   |  └── g = 全局（每行所有匹配），不加 g 只替换每行第一个
#          |   └──── 替换文本
#          └──────── 搜索模式

# === 仅替换第 N 个匹配（每行）= 整行中只替换第 N 处 ===
sed -i 's/old/new/2' file.txt     # 替换每行第 2 个匹配
sed -i 's/old/new/3g' file.txt    # 替换每行第 3 个及之后所有匹配

# === 仅替换特定行 ===
sed -i '3s/old/new/' file.txt     # 仅替换第 3 行的第一个匹配
sed -i '3s/old/new/g' file.txt    # 仅替换第 3 行的所有匹配
sed -i '2,5s/old/new/g' file.txt  # 替换第 2-5 行的所有匹配
sed -i '/pattern/s/old/new/g' file.txt   # 仅在匹配 pattern 的行中替换
sed -i '/start/,/end/s/old/new/g' file.txt  # 在 start 到 end 区间内替换

# === 第 N 个匹配（全局范围，非按行，仅替换整个文件的第 N 处匹配） ===
sed 's/old/new/1' file.txt   # 只替换文件中第一个 old（每行内）
# 真正替换全局第 N 个需要 awk：
awk '{for(i=1;i<=NF;i++) if($i=="old"&&++c==3) $i="new"} 1' file.txt

# === 按模式匹配替换 ===
sed -i 's/^#.*//' file.txt           # 删除注释行（# 开头）（留下空行）
sed -i '/^#/d' file.txt              # 真正删除注释行
sed -i 's/[[:space:]]*$//' file.txt  # 删除行尾空白

# === 高级用例 ===
# 分组引用（\1, \2 等）
sed -i 's/\([0-9]\{4\}\)-\([0-9]\{2\}\)-\([0-9]\{2\}\)/\3\/\2\/\1/' file.txt
# 将 2024-12-31 变成 31/12/2024

# -E 扩展正则（不用转义括号和花括号）
sed -i -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/' file.txt

# 在匹配行前后插入内容
sed -i '/pattern/i\插入在此行之前' file.txt
sed -i '/pattern/a\追加在此行之后' file.txt

# 修改整行
sed -i '/pattern/c\替换整行内容' file.txt

# 多个 sed 命令
sed -i -e 's/old/new/g' -e '/^$/d' -e 's/\r$//' file.txt
```

**`-i` 参数的安全性：**

```bash
# 危险：直接修改原文件（无法恢复）
sed -i 's/foo/bar/g' important.conf

# 安全：创建备份文件
sed -i.bak 's/foo/bar/g' important.conf    # 生成 important.conf.bak
# macOS 注意：sed -i '' 's/foo/bar/g' file.txt  # macOS 需要空字符串参数
```

## 3.2 如何用 awk 统计某列的平均值/总和？

```bash
# 假设有 data.txt：
# 姓名  数学  语文  英语
# 张三  85   90   88
# 李四  92   78   85
# 王五  78   95   90

# === 计算第 2 列（数学）总和 ===
awk '{sum+=$2} END {print "总和:", sum}' data.txt

# === 计算第 2 列（数学）平均值 ===
awk '{sum+=$2; count++} END {printf "平均值: %.2f\n", sum/count}' data.txt

# === 跳过标题行 ===
awk 'NR>1 {sum+=$2; count++} END {printf "平均数学成绩: %.2f\n", sum/count}' data.txt

# === 统计空行和非空行的行数 ===
awk 'BEGIN {empty=0; nonempty=0}
     /^$/ {empty++; next}
     {nonempty++}
     END {print "空行:", empty, "非空行:", nonempty}' file.txt

# === 按条件聚合（按第一列分组，求第二列平均值） ===
# 假设文件：张三 85, 张三 90, 李四 78 ...
awk '{
    name=$1; score=$2
    sum[name]+=score
    count[name]++
} END {
    for(name in sum)
        printf "%s: 平均 %.2f (共 %d 次)\n", name, sum[name]/count[name], count[name]
}' data.txt

# === 求最大值/最小值/中位数 ===
awk '{
    if(NR==1) {max=$2; min=$2}
    if($2>max) max=$2
    if($2<min) min=$2
} END {print "最大值:", max, "最小值:", min}' data.txt

# === 格式化输出（生成报告） ===
awk 'BEGIN {
    printf "%-10s %-8s %-8s %-8s %-8s\n", "姓名", "数学", "语文", "英语", "总分"
    printf "---------------------------------------------\n"
}
NR>1 {
    total=$2+$3+$4
    printf "%-10s %-8d %-8d %-8d %-8d\n", $1, $2, $3, $4, total
}' data.txt

# === 统计某列值的分布 ===
awk '{count[$2]++} END {for(val in count) print val, count[val]}' data.txt \
    | sort -k2 -rn
```

**`sum+=value` 的坑：awk 将未初始化的变量视为 0 或空字符串，所以 `sum+=$2` 在第一次执行时 `sum` 默认为 0，不会报错。**

```bash
# 常见边界处理
awk '
NF < 2 {next}            # 跳过不够列数的行
$2 ~ /^[0-9]+$/ {        # 仅处理数值
    sum+=$2; count++
}
END {
    if(count>0) printf "平均: %.2f\n", sum/count
    else print "无有效数据"
}' data.txt
```

## 3.3 如何用 grep 找出不包含某模式的所有文件？

```bash
# === 方式一：grep -L（列文件名，最直接） ===
grep -rL "pattern" /path/to/dir/
# -r: 递归
# -L: --files-without-match，列出不包含模式的文件的文件名

# === 方式二：grep -Lv（-v 反转，-l 列出文件名） ===
grep -rlv "pattern" /path/to/dir/

# === 方式三：find + grep（更多控制） ===
find /path/to/dir -type f -name "*.log" ! -exec grep -q "ERROR" {} \; -print

# === 方式四：输出不包含模式的行（-v 反转匹配） ===
grep -v "pattern" file.txt                      # 显示不匹配的行
grep -v -e "pattern1" -e "pattern2" file.txt    # 排除多个模式

# === 方式五：找出同一个文件中同时满足多个条件的行 ===
grep -v "DEBUG" app.log | grep "ERROR"
# 先排除 DEBUG 行，再从剩余行中找 ERROR

# === 常见场景 ===
# 1. 找出不包含 TODO 注释的 .java 文件
grep -rL "TODO" --include="*.java" src/

# 2. 找出没有 license 头的文件
grep -rL "Copyright" --include="*.{c,h,py}" .

# 3. 排除二进制文件
grep -rLI "pattern" /path/    # -I 忽略二进制文件

# 4. 找出不含某关键词的配置文件
find /etc/nginx -type f -name "*.conf" -exec grep -L "ssl_certificate" {} \;
```

**`grep -L` vs `grep -l` vs `grep -v` 对比：**

| 选项 | 含义 | 输出内容 |
|------|------|----------|
| `-l` | --files-with-matches | 包含匹配的**文件名** |
| `-L` | --files-without-match | 不包含匹配的**文件名** |
| `-v` | --invert-match | 不匹配的**行内容** |

## 3.4 如何删除文件中的空行和注释行？

```bash
# 假设文件内容：
# # 这是注释
# server_name example.com;  # 行内注释
# 
# listen 80;
#     # 缩进注释

# === 方式一：grep（最简洁） ===
# 删除空白行和以 # 开头的注释行
grep -v '^[[:space:]]*#' config.conf | grep -v '^[[:space:]]*$'

# 或者一条 grep 搞定
grep -v -e '^[[:space:]]*#' -e '^[[:space:]]*$' config.conf

# 或者用 -E 扩展正则
grep -vE '^[[:space:]]*(#|$)' config.conf

# === 方式二：sed（原地修改） ===
sed -i '/^[[:space:]]*#/d; /^[[:space:]]*$/d' config.conf

# 更简洁：一条 sed
sed -i '/^[[:space:]]*\(#.*\)\?$/d' config.conf

# === 方式三：awk ===
awk '!/^[[:space:]]*#/ && !/^[[:space:]]*$/' config.conf

# 或者更紧凑
awk 'NF && !/^[[:space:]]*#/' config.conf

# === 方式四：sed 同时处理行尾注释 ===
# 先删除行内注释，再删除空行
sed -E 's/[[:space:]]*#.*//' config.conf | sed '/^[[:space:]]*$/d'

# === 各种场景的变体 ===
# 1. 仅删除真正空行，保留缩进的"空行"（即只包含空格的行保留）
sed -i '/^$/d' file.txt

# 2. 删除以 # 开头的注释行，但保留 #!/bin/bash 开头的 shebang
sed -i '/^[[:space:]]*#/{
    /^#!/!d
}' script.sh

# 3. 保留空行但删除注释
sed -i 's/[[:space:]]*#.*//' file.txt   # 删除注释内容但保留空行

# 4. 压缩连续空行为单个空行
sed -i '/^$/N; /^\n$/D' file.txt

# 5. 仅删除某模式区间内的空行和注释
sed -i '/^<config>/,/^<\/config>/{
    /^[[:space:]]*#/d
    /^[[:space:]]*$/d
}' config.xml    # （假设 XML 配置）
```

---

# 4. 调试与错误处理题

## 4.1 `set -e` 的作用是什么？有哪些例外情况？

`set -e`（`set -o errexit`）让脚本在**任何命令返回非零退出码时立即终止**。其目的是"fail fast"。

**核心规则：** 也即 `set -e` 生效的上下文：

```bash
#!/bin/bash
set -e

true            # 退出码 0，继续
false           # 退出码 1，脚本立即退出
echo "不会执行到这里"
```

**`set -e` 不生效的例外情况（这才是重点）：**

```bash
#!/bin/bash
set -e

# 例外 1：if / while / until 的条件部分
if false; then
    echo "不执行"
fi
echo "仍然执行（if 条件是 false 但不会退出）"

# 例外 2：&& 和 || 的左侧命令
false && echo "不执行"      # false 不会触发退出
true || echo "不执行"       # || 后的 false 也不会触发
# 但下面这样会退出：
# false || false

# 例外 3：管道中的命令（除非同时设置 pipefail）
false | true                # 不会退出
set -o pipefail
false | true                # 会退出！

# 例外 4：while / until 循环的最后一个命令
while false; do
    echo "不会进入循环体"
done
echo "循环结束，继续执行"    # false 在这里不触发退出

# 例外 5：! 取反后的命令
! false                     # 不会退出（因为 ! 反转了退出码）

# 例外 6：命令替换中失败的命令
local var=$(false)          # 不会退出
var=$(false)                 # 也不会退出
# 但直接赋值 var=$(false) 不触发
# 如果 local var 和 $(false) 分开：
# local var
# var=$(false)              # 这个不会退出，因为是命令替换

# 例外 7：子 shell 中的命令（括号内）
(false)
echo "子 shell 失败不影响父进程"

# 例外 8：函数内部有些复杂情况
# 如果函数被用在条件表达式或管道中：
my_func() { false; }
if my_func; then :; fi      # 不退出
my_func || echo "fallback"  # 不退出
```

```bash
# 完整示例：展示 set -e 的各个例外
#!/bin/bash
set -e

echo "1. 测试 if 条件中的失败"
if grep -q "notfound" /etc/hosts; then
    echo "找到"
else
    echo "没找到（脚本未退出——if 条件例外）"
fi

echo "2. 测试 while 条件中的失败"
while ! ping -c 1 -W 1 10.0.0.1 &>/dev/null; do
    sleep 1
done
echo "ping 成功或循环结束（! 取反例外）"

echo "3. 测试 || 中的失败"
grep -q "notfound" /etc/hosts || echo "使用默认值（|| 例外）"

echo "4. 测试命令替换中的失败"
result=$(false)    # 不会退出
echo "命令替换失败，但脚本继续（例外 6）"

echo "5. 真正会退出的情况"
false    # 这里脚本会退出！
echo "这行不会执行"
```

**推荐搭配：`set -euo pipefail`**

```bash
#!/bin/bash
set -euo pipefail
#  -e: 命令失败即退出
#  -u: 引用未定义变量即退出
#  -o pipefail: 管道中任一失败即整体失败
```

## 4.2 `set -u` 和 `set -o pipefail` 的作用？

**`set -u`（`set -o nounset`）：引用未定义变量时立即退出。**

```bash
#!/bin/bash
set -u

echo "用户名：$USER"          # USER 通常已定义，正常
echo "未定义变量：$UNDEFINED"  # 立即退出：UNDEFINED: unbound variable

# 安全的未定义检测方式
echo "${UNDEFINED:-默认值}"    # 提供默认值，不会触发 -u
echo "${UNDEFINED:=默认值}"    # 未定义时赋值并使用

# 检查变量是否已定义
if [ -z "${VAR+x}" ]; then
    echo "VAR 未定义"
fi

# -u 的局限：不能区分"未定义"和"定义为空字符串"
# 以下场景 set -u 不会报错：
MY_VAR=""
echo "$MY_VAR"                 # 已定义但为空，不报错
```

**`set -o pipefail`：管道中任何一个命令失败，整个管道的退出码为该失败命令的退出码。**

```bash
#!/bin/bash

# === 默认行为（pipefail 关闭） ===
# 管道的退出码 = 最后一个命令的退出码
false | true
echo "退出码：$?"       # 0（因为 true 成功）

# === pipefail 开启后 ===
set -o pipefail
false | true
echo "退出码：$?"       # 1（因为 false 失败了）

# === 实际陷阱案例 ===
# 没有 pipefail：
grep "pattern" huge_file.log | wc -l
# 即使 grep 失败（文件不存在），wc -l 也会返回行数 0，退出码为 0
# 脚本认为一切正常，实际上数据都没读进来！

# 有 pipefail：
set -o pipefail
grep "pattern" huge_file.log | wc -l
# grep 失败导致整个管道失败，脚本（若配合 set -e）会退出

# === pipefail + set -e 的组合 ===
set -eo pipefail
cat missing_file | while read line; do
    echo "处理：$line"
done
# cat 失败 → pipefail 标记整个管道失败 → set -e 使脚本退出

# === 获取管道中每个命令的退出码（PIPESTATUS 数组） ===
set -o pipefail
cmd1 | cmd2 | cmd3
echo "整体退出码：$?"
echo "cmd1: ${PIPESTATUS[0]}"
echo "cmd2: ${PIPESTATUS[1]}"
echo "cmd3: ${PIPESTATUS[2]}"

# 即使 set -o pipefail，依然可以逐个检查
grep "pattern" file.txt | sort | uniq -c
if [ ${PIPESTATUS[0]} -ne 0 ]; then
    echo "grep 阶段失败" >&2
fi
```

## 4.3 如何在脚本中处理错误？

```bash
#!/bin/bash

# === 策略一：trap ERR（全局错误捕获） ===
set -e
trap 'echo "错误发生在第 $LINENO 行，命令: $BASH_COMMAND，退出码: $?" >&2' ERR

# ERR trap 在命令返回非零且 set -e 会触发退出时执行
# 注意：ERR trap 在那些 set -e 的例外情况中也不会触发

# === 策略二：trap EXIT（脚本退出时清理） ===
cleanup() {
    local exit_code=$?
    echo "清理临时文件..."
    rm -rf "$TEMP_DIR"
    echo "脚本退出，退出码: $exit_code"
    exit $exit_code
}
trap cleanup EXIT

# === 策略三：函数级错误处理 ===
safe_run() {
    local cmd="$1"
    local error_msg="${2:-命令执行失败}"

    eval "$cmd"
    local exit_code=$?

    if [ $exit_code -ne 0 ]; then
        echo "$error_msg (退出码: $exit_code)" >&2
        return $exit_code
    fi
    return 0
}

safe_run "grep -q pattern file.txt" "未找到 pattern"
safe_run "mkdir -p /important/dir" "创建目录失败"

# === 策略四：错误码检查模式 ===
download_file() {
    local url="$1"
    local output="$2"

    if ! curl -fsSL "$url" -o "$output"; then
        echo "下载失败: $url" >&2
        return 1
    fi

    if [ ! -s "$output" ]; then
        echo "下载的文件为空: $output" >&2
        return 1
    fi

    echo "下载成功: $output"
    return 0
}

# === 策略五：完整错误处理框架 ===
#!/bin/bash
set -euo pipefail

# 配置
LOG_FILE="/var/log/myapp/script.log"
TEMP_DIR=$(mktemp -d)
readonly LOG_FILE TEMP_DIR

# 错误处理
on_error() {
    local line=$1
    local command=$2
    local code=$3

    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR:"
    echo "  脚本: ${BASH_SOURCE[1]:-$0}"
    echo "  函数: ${FUNCNAME[1]:-main}"
    echo "  行号: $line"
    echo "  命令: $command"
    echo "  退出码: $code"
} >> "$LOG_FILE"

# 退出清理
on_exit() {
    local code=$?
    echo "[$(date)] 脚本退出，退出码: $code" >> "$LOG_FILE"
    rm -rf "$TEMP_DIR"
    exit $code
}

trap 'on_error $LINENO "$BASH_COMMAND" $?' ERR
trap on_exit EXIT
```

## 4.4 如何调试一个 Shell 脚本？

```bash
#!/bin/bash

# === 方法一：set -x（命令追踪） ===
set -x
# 执行前打印每条命令到 stderr，前缀为 PS4（默认 "+"）
name="debug demo"
echo "Hello, $name"
set +x   # 关闭追踪

# === 方法二：bash -x 命令行参数 ===
# bash -x script.sh
# bash -xv script.sh    # -v 额外打印原始行（未展开）

# === 方法三：局部追踪（仅关键部分） ===
function critical_section() {
    set -x
    # 关键逻辑
    local result=$(compute_something)
    # ...
    set +x
}

# === 方法四：PS4 增强 ===
# 默认 PS4="+ " 太简单，增强后可显示行号、函数名、时间戳
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
# 输出形如：+(script.sh:42): main(): name=debug demo

# 更进一步：带时间戳
export PS4='+[$(date "+%H:%M:%S")] ${BASH_SOURCE}:${LINENO}: '

# === 方法五：BASH_XTRACEFD（将追踪输出重定向到文件） ===
exec 5>/tmp/script_debug.log
BASH_XTRACEFD=5
set -x
# 之后所有追踪输出写入 /tmp/script_debug.log 而非 stderr
# 适合生产环境排查

# === 方法六：bash -n（语法检查不执行） ===
bash -n script.sh      # 仅检查语法错误

# === 方法七：trap DEBUG（逐条命令调试） ===
trap 'echo "即将执行 (line $LINENO): $BASH_COMMAND"' DEBUG
# 每条命令执行前触发

# === 方法八：日志函数 ===
LOG_LEVEL="${LOG_LEVEL:-INFO}"
log() {
    local level="$1"; shift
    echo "[$(date '+%H:%M:%S')] [$level] $*" >&2
}
log_debug() { [ "$LOG_LEVEL" = "DEBUG" ] && log "DEBUG" "$@"; }
log_info()  { log "INFO" "$@"; }
log_error() { log "ERROR" "$@"; }

# === 调试工具选择矩阵 ===
# 问题类型              推荐方法
# 变量值不确定           set -x + 增强 PS4
# 脚本崩溃位置不明       trap ERR + 行号
# 语法错误               bash -n
# 生产环境排查           BASH_XTRACEFD 输出到文件
# 函数调用链模糊         trap DEBUG + FUNCNAME
# 管道中间环节失败       PIPESTATUS + pipefail
# 性能瓶颈               time 命令包裹关键部分

# === 实战：完整的调试脚本头部 ===
#!/bin/bash
# debug 模式：DEBUG=1 ./script.sh
if [ "${DEBUG:-0}" = "1" ]; then
    set -x
    export PS4='+(${BASH_SOURCE##*/}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
fi

set -euo pipefail
trap 'echo "[ERROR] line $LINENO: $BASH_COMMAND (exit: $?)" >&2' ERR
```

---

# 5. 进阶与陷阱题

## 5.1 管道中的 while 循环为什么不能修改外部变量？如何解决？

**原因：管道会创建子 shell（subshell），while 循环在子 shell 中运行，对变量的修改只在子 shell 内生效，父 shell 看不到。**

```bash
#!/bin/bash

# === 问题演示 ===
count=0
echo -e "a\nb\nc" | while read line; do
    ((count++))
    echo "子 shell 内：count=$count"
done
echo "父 shell 中：count=$count"     # 输出 0，而非 3！

# 为什么？
# 管道 | 的两端各在一个子 shell 中执行
# 可以这样验证：
echo "父 shell PID: $$"
echo -e "a\nb\nc" | while read line; do
    echo "子 shell PID: $$"           # $$ 在子 shell 中是父 shell 的 PID（不会反映子 shell）
    echo "实际子 shell: $BASHPID"     # $BASHPID 才是真实的子 shell PID
done
```

**解决方案一：使用进程替换（`< <(...)`）**

```bash
count=0
while read line; do
    ((count++))
done < <(echo -e "a\nb\nc")
echo "count=$count"    # 输出 3 —— 正确！

# 原理：将命令的输出当作文件输入给 while，while 运行在当前 shell
```

**解决方案二：使用 here-string**

```bash
count=0
while read line; do
    ((count++))
done <<< "$(echo -e "a\nb\nc")"
echo "count=$count"    # 输出 3

# 原理：$() 命令替换先执行，结果存储在内存中，然后通过 <<< 传给当前 shell 的 while
```

**解决方案三：使用命名管道（named pipe / FIFO）**

```bash
count=0
mkfifo /tmp/myfifo
echo -e "a\nb\nc" > /tmp/myfifo &
while read line; do
    ((count++))
done < /tmp/myfifo
rm -f /tmp/myfifo
echo "count=$count"    # 输出 3
```

**解决方案四：使用最后一个命令的技巧（有限场景）**

```bash
count=0
echo -e "a\nb\nc" | {
    while read line; do
        ((count++))
    done
    echo "count=$count"   # 在同一个子 shell 内，可以访问到
}
# 但出了 {} 后 count 还是 0
```

**解决方案对比：**

| 方案 | 内存占用 | 复杂度 | 适用场景 |
|------|----------|--------|----------|
| 进程替换 `< <(cmd)` | 流式，低 | 低 | 大文件处理（推荐） |
| here-string `<<< "$(cmd)"` | 需要等命令执行完全部写入内存 | 低 | 小数据量 |
| 命名管道 | 流式，低 | 高 | POSIX 兼容需求 |
| 重定向到文件 `while ... done < file` | 需磁盘 I/O | 低 | 已有文件 |

## 5.2 `cmd > file 2>&1` 和 `cmd 2>&1 > file` 有什么区别？

**这是 Shell 面试最高频的重定向陷阱题。区别在于重定向的顺序决定了最终目标。**

```bash
# === 正确答案 ===
# cmd > file 2>&1：stdout 和 stderr 都写入 file
# cmd 2>&1 > file：stderr 写入终端（原 stdout 目标），stdout 写入 file

# === 原理解释（按顺序执行） ===
# 1. cmd > file 2>&1
#    步骤1：> file        → fd1(stdout) 指向 file
#    步骤2：2>&1           → fd2(stderr) 指向 fd1 当前目标(file)
#    结果：stdout → file，stderr → file

# 2. cmd 2>&1 > file
#    步骤1：2>&1           → fd2(stderr) 指向 fd1 当前目标(终端)
#    步骤2：> file         → fd1(stdout) 指向 file
#    结果：stdout → file，stderr → 终端（错误！）
```

```bash
# === 可视化演示 ===
# 创建一个同时输出 stdout 和 stderr 的命令
test_cmd() {
    echo "这是 stdout"
    echo "这是 stderr" >&2
}

echo "===== 正确方式：cmd > file 2>&1 ====="
test_cmd > /tmp/output1.txt 2>&1
echo "文件内容："
cat /tmp/output1.txt
# 这是 stdout
# 这是 stderr   ← 两条都在文件里

echo ""
echo "===== 错误方式：cmd 2>&1 > file ====="
test_cmd 2>&1 > /tmp/output2.txt
echo "文件内容："
cat /tmp/output2.txt
# 这是 stdout   ← 只有 stdout 在文件里
# stderr 输出到了终端上
```

**快捷写法对比：**

```bash
# 所有 stdout 和 stderr 都到 file
cmd > file 2>&1       # 传统写法
cmd &> file            # bash 4.0+ 简写
cmd >& file            # 另一种简写（不推荐，csh 含义不同）

# 只追加不覆盖
cmd >> file 2>&1       # 追加方式
cmd &>> file           # bash 4.0+ 追加简写

# 只重定向 stderr（保留 stdout 到终端）
cmd 2> error.log       # stderr → error.log，stdout → 终端

# stdout 和 stderr 到不同文件
cmd > stdout.log 2> stderr.log

# 丢弃 stderr
cmd 2>/dev/null

# 丢弃所有输出
cmd &>/dev/null
```

## 5.3 `[ -n "$var" ]` 和 `[ -n $var ]` 有什么区别？

**不加引号时，若 `$var` 为空，`[ -n $var ]` 展开为 `[ -n ]`，这是一个永真条件（因为 `-n` 是非空字符串，且此时 `-n` 不再是测试操作符）。**

```bash
#!/bin/bash

var=""

# === 加引号（正确） ===
[ -n "$var" ]
echo "加引号，退出码：$?"     # 1（假，var 为空字符串）
# 展开后：[ -n "" ] → -n 操作符测试空字符串 → 假

# === 不加引号（错误！） ===
[ -n $var ]
echo "不加引号，退出码：$?"   # 0（真！但 var 明明为空！）
# 展开后：[ -n ] → test 收到单个参数 "-n"，规则：
#   单参数且非空 → 测试该字符串是否非空 → "-n" 是非空字符串 → 真

# === 更直观的等价验证 ===
[ -n ]       # 等价于 [ -n $var ]（var 为空时）
echo $?      # 0（真）

[ -n "" ]    # 等价于 [ -n "$var" ]（var 为空时）
echo $?      # 1（假）

[ "" ]       # 单参数空字符串
echo $?      # 1（假——空字符串测试为假）

[ "-n" ]     # 单参数非空字符串
echo $?      # 0（真——非空字符串测试为真，这里 -n 被当成了普通字符串！）
```

**`test` 命令（`[`）的参数数量规则：**

| 参数数量 | 行为 | 示例 |
|----------|------|------|
| 0 个参数 | 假（退出码 1） | `[ ]` → 1 |
| 1 个参数 | 若参数非空则为真 | `[ "hello" ]` → 0，`[ "" ]` → 1 |
| 2 个参数 | 第一个是操作符，第二个是操作数 | `[ -n "" ]` → 1 |
| 3 个参数 | 二元运算 | `[ "a" = "b" ]` → 1 |

**引号是 Shell 变量安全的核心守则——不确定时，加上双引号。**

## 5.4 `rm "$file"` 和 `rm $file` 有什么区别？

**不加引号时，若文件名包含空格、制表符或通配符，会被 word splitting 和 globbing 拆成多个参数，导致误删文件。**

```bash
#!/bin/bash

# === 创建测试文件 ===
touch "report 2024.txt"
touch "report hello.txt"
touch "report"

# === 场景一：文件名含空格 ===
file="report 2024.txt"

rm $file
# 展开为：rm report 2024.txt
# 结果：删除了 "report" 和 "2024.txt" 两个文件（不是期望的）

rm "$file"
# 展开为：rm "report 2024.txt"
# 结果：正确删除 "report 2024.txt"

# === 场景二：文件名含通配符 ===
# 创建文件：a*.txt（literal 星号）
touch "a*.txt"
file="a*.txt"

rm $file
# 展开为：rm a*.txt
# Shell 先做 globbing，a*.txt 匹配到啊 a*.txt 自身（恰好只有一个文件）
# 但如果目录中还有 a1.txt a2.txt 等，会全部被删除！

rm "$file"
# 展开为：rm "a*.txt"
# 正确删除名为 "a*.txt" 的文件

# === 场景三：变量为空（灾难级） ===
# 假设 file 变量意外为空
file=""
rm $file
# 展开为：rm （无参数，rm 等待 stdin 或报错但不会删除文件）
# 某些命令更危险：
# rm -rf ${dir}/
# 若 dir 为空：rm -rf /   —— 系统毁灭

rm "$file"
# 展开为：rm "" 
# rm 报错 "cannot remove '': No such file or directory"——安全

# === 场景四：一个更危险的例子 ===
config_dir="/etc/myapp"
rm -rf $config_dir/backup
# 如果 config_dir 意外为空：
# rm -rf /backup  → 想删除 /etc/myapp/backup，却删了 /backup

rm -rf "${config_dir}/backup"
# 安全
```

**黄金法则：所有变量引用加双引号，除非明确需要 word splitting 或 globbing。**

```bash
# 安全的脚本模板
set -u   # 未定义变量报错，防止空变量导致灾难

# set -u 配合变量引号的双重保护
target_dir="$1"
rm -rf "${target_dir:?请指定目录}"    # :? 防止空变量展开
# 若 target_dir 为空或未定义，脚本直接报错退出
```

## 5.5 `exit` vs `return` vs `exec` 的区别？

| 命令 | 作用范围 | 行为 | 典型用法 |
|------|----------|------|----------|
| `exit N` | 整个脚本进程 | 终止当前 Shell 进程，返回退出码 N | `exit 0`, `exit 1` |
| `return N` | 函数 / source 脚本 | 从函数或 source 文件中返回，返回码 N | `return 1` |
| `exec CMD` | 当前进程 | 用指定命令**替换**当前进程（不创建新进程） | `exec bash`, `exec > file` |

```bash
#!/bin/bash

# === exit ===
echo "脚本开始"
exit 42                # 脚本立即终止，返回码 42
echo "不会执行"

# === return ===
# 1. 在函数中使用
my_func() {
    if [ ! -f "$1" ]; then
        echo "文件不存在: $1" >&2
        return 1         # 从函数返回，退出码 1
    fi
    echo "文件存在"
    return 0
}
my_func "/etc/hosts"
echo "函数调用完成，脚本继续"

# 2. 在 source 中使用
# 文件 lib.sh：
#!/bin/bash
if [ -z "$REQUIRED_VAR" ]; then
    echo "REQUIRED_VAR 未设置" >&2
    return 1             # 从 source 返回，不终止调用者
fi
# 主脚本：
# source lib.sh || exit 1

# === exec ===
# 1. exec CMD：替换当前进程
echo "当前进程 PID: $$"
exec sleep 100
echo "这行不会执行"      # 进程已被 sleep 替换
# 在另一个终端 kill 该 sleep 后，终端关闭（因为没有 Shell 进程了）

# 2. exec 重定向：在脚本中切换文件描述符
exec > /tmp/log.txt 2>&1   # 之后所有 echo 输出到文件而不是终端
echo "这条输出到了文件"

# 3. exec 重定向 stdin
exec < /tmp/input.txt
while read line; do
    echo "读取：$line"
done

# === 综合示例 ===
#!/bin/bash
# exit vs return 的典型应用

setup() {
    # 返回码表达"是否成功"
    mkdir -p "$1" || return 1
    cd "$1" || return 1
    return 0
}

main() {
    setup "/tmp/mydir" || {
        echo "初始化失败" >&2
        exit 1          # 退出整个脚本
    }
    echo "初始化成功，继续执行"
    exit 0              # 正常退出
}

main "$@"
```

## 5.6 Bash 中如何实现浮点数运算？

**Bash 原生不支持浮点数，所有算术都是整数。需要借助外部工具。**

```bash
#!/bin/bash

# === 方式一：bc（最常用，POSIX） ===
echo "3.14 * 2.5" | bc              # 7.85
echo "scale=4; 10/3" | bc           # 3.3333（scale 控制小数位数）
echo "scale=2; 3.14159 * 2" | bc    # 6.28

# bc 高级用法
result=$(echo "scale=4; sqrt(2)" | bc -l)   # -l 加载数学库
echo "根号 2 = $result"

# 比较运算
if [ "$(echo "$a > $b" | bc)" -eq 1 ]; then
    echo "$a 大于 $b"
fi

# === 方式二：awk（推荐，语法丰富） ===
echo | awk '{print 3.14 * 2.5}'              # 7.85
awk 'BEGIN {printf "%.2f\n", 10/3}'          # 3.33
awk "BEGIN {printf \"%.2f\n\", $a + $b}"     # 使用 shell 变量
awk 'BEGIN {print (123.456 > 100)}'          # 1（true）

# === 方式三：python/perl（复杂计算） ===
python3 -c "print(3.14 * 2.5)"
python3 -c "import math; print(f'{math.pi:.4f}')"
perl -e "print 3.14 * 2.5"

# === 方式四：zsh 原生支持（若使用 zsh） ===
# zsh 中直接计算：
# echo $((3.14 * 2.5))     # zsh 原生支持

# === 方式五：整数模拟浮点（常用于低资源环境） ===
# 例如模拟两位小数，将所有值乘以 100 后用整数运算
price=1999    # 代表 $19.99（单位：分）
tax=8         # 8% 的税率（单位：0.01%）
total_cents=$((price * (100 + tax) / 100))
echo "总价：$((total_cents / 100)).$(printf "%02d" $((total_cents % 100)))"   # 21.58

# 或者用 printf 格式化
total_dollars=$((price * (100 + tax) / 10000))          # 整数部分
total_cents=$(( (price * (100 + tax) / 100) % 100))     # 小数部分
printf "总价: %d.%02d\n" $total_dollars $total_cents

# === 综合浮点工具函数 ===
calc() {
    # 用法：calc "3.14 * 2.5"
    # 输出：7.85
    echo "scale=6; $*" | bc -l | sed 's/^\./0./'
}

sin()  { echo "scale=6; s($1)" | bc -l; }
cos()  { echo "scale=6; c($1)" | bc -l; }
sqrt() { echo "scale=6; sqrt($1)" | bc -l; }
```

## 5.7 什么是 here document？有哪些使用场景？

**Here Document（此处文档）是一种将多行文本直接嵌入脚本的 I/O 重定向技术。语法：`<<DELIMITER ... DELIMITER`**

```bash
# === 基本语法 ===
command << 'EOF'
多行文本内容
可以包含 $变量（取决于 EOF 是否被引号包围）
EOF

# 带引号的 EOF（单引号/双引号/转义） = 不展开变量
# 不带引号的 EOF = 展开变量和命令替换

# === 场景一：多行文本输出 ===
cat << 'EOF'
========================================
  系统信息报告（生成时间：$(date)）
========================================
主机名: $(hostname)
操作系统: $(uname -s)
# 注意：因为 EOF 被引号包围，$(date) 和 $(hostname) 不会被展开
EOF

cat << EOF
========================================
  系统信息报告（生成时间：$(date)）
========================================
主机名: $(hostname)
操作系统: $(uname -s)
# 这次 EOF 没加引号，变量和命令会展开
EOF

# === 场景二：生成配置文件 ===
generate_nginx_conf() {
    local domain="$1"
    local port="$2"

    cat > "/etc/nginx/sites-available/${domain}.conf" << EOF
server {
    listen ${port};
    server_name ${domain};

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
    # 注意：\$host 和 \$remote_addr 加了反斜杠，不会被展开
    # 它们应该作为 Nginx 变量原样写入配置
}

# === 场景三：向命令提供多行输入 ===
# 使用 mysql 执行多条 SQL
mysql -u root -p << 'SQL'
CREATE DATABASE IF NOT EXISTS myapp;
USE myapp;
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE
);
SQL

# === 场景四：heredoc 写入文件（避免 echo -e 的混乱） ===
cat > /tmp/script.sh << 'SCRIPT'
#!/bin/bash
# 这是一个生成的自包含脚本
echo "Hello from generated script!"
SCRIPT
chmod +x /tmp/script.sh

# === 场景五：here document 在管道中使用 ===
cat << 'EOF' | while read line; do
item1
item2
item3
EOF
    echo "处理：$line"
done

# === 场景六：带缩进的 here document（<<- 前缀制表符缩进） ===
if [ "$DEBUG" = "1" ]; then
    cat <<- 'EOF'
        调试信息：
        当前用户: $(whoami)
        当前目录: $(pwd)
	EOF
    # <<- 开头的 here doc，行首的 TAB 制表符会被移除（空格不会）
    # 注意：必须是真正的 TAB 字符，空格无效
fi

# === 场景七：here string（<<< 单行变体） ===
grep "pattern" <<< "$long_string_variable"
# 等价于 echo "$long_string_variable" | grep "pattern"
# 但避免了管道的子 shell 问题
```

**Here Document 分隔符规则：**

| 写法 | 变量展开 | 命令替换 | 转义 |
|------|----------|----------|------|
| `<< EOF` | 展开 | 执行 | `\$` 可阻止 |
| `<< 'EOF'` | 不展开 | 不执行 | 全部原样 |
| `<< "EOF"` | 不展开 | 不执行 | 全部原样 |
| `<< \EOF` | 不展开 | 不执行 | 全部原样 |
| `<<- EOF` | 展开 | 执行 | 且允许 TAB 缩进 |

---

# 6. 场景设计题

## 6.1 设计一个防重复执行的脚本（锁文件机制）

**核心原理：使用文件锁（lockfile）确保同一时间只有一个实例运行。锁文件存储当前进程 PID，启动时检查已有实例是否存活。**

```bash
#!/bin/bash
# 防重复执行脚本框架

set -euo pipefail

readonly LOCK_FILE="/var/run/myapp.lock"
readonly LOCK_TIMEOUT=3600    # 锁超时时间（秒），防止死锁

# === 获取锁 ===
acquire_lock() {
    # 检查锁文件是否存在
    if [ -f "$LOCK_FILE" ]; then
        local existing_pid
        existing_pid=$(cat "$LOCK_FILE" 2>/dev/null)

        # 检查进程是否还活着
        if [ -n "$existing_pid" ] && kill -0 "$existing_pid" 2>/dev/null; then
            echo "错误：已有实例运行中（PID: $existing_pid）" >&2
            return 1
        fi

        # 检测死锁：锁文件存在但进程已不存在
        local lock_age
        lock_age=$(($(date +%s) - $(stat -c %Y "$LOCK_FILE" 2>/dev/null || echo 0)))
        if [ "$lock_age" -gt "$LOCK_TIMEOUT" ]; then
            echo "警告：检测到过期锁文件（${lock_age} 秒），强制清理" >&2
            rm -f "$LOCK_FILE"
        else
            echo "错误：锁文件存在但进程已死，等待超时..." >&2
            return 1
        fi
    fi

    # 创建锁文件（原子操作）
    echo "$$" > "$LOCK_FILE"   # $$ 是当前进程 PID

    # 验证写入成功且锁属于自己（防止竞态条件）
    if [ "$(cat "$LOCK_FILE" 2>/dev/null)" != "$$" ]; then
        echo "错误：获取锁失败（可能被其他进程抢占）" >&2
        return 1
    fi

    echo "锁获取成功（PID: $$）"
    return 0
}

# === 释放锁 ===
release_lock() {
    if [ -f "$LOCK_FILE" ] && [ "$(cat "$LOCK_FILE" 2>/dev/null)" = "$$" ]; then
        rm -f "$LOCK_FILE"
        echo "锁已释放"
    fi
}

# === 主流程 ===
main() {
    # 注册退出时的清理
    trap release_lock EXIT INT TERM

    acquire_lock || exit 1

    # ===== 以下是实际业务逻辑 =====
    echo "开始执行关键任务..."
    sleep 30    # 模拟长时间任务
    echo "任务完成"
    # ===== 业务逻辑结束 =====

    # 正常退出，trap 会自动释放锁
    exit 0
}

main "$@"
```

```bash
# === 进阶版：使用 flock（Linux 内核级文件锁，更可靠） ===
#!/bin/bash
# flock 提供了内核级别的文件锁，更可靠且不存在竞态条件

readonly LOCK_FILE="/var/run/myapp.lock"

exec 200>"$LOCK_FILE"                    # 打开文件描述符 200
if ! flock -n 200; then                  # -n = 非阻塞（立即返回）
    echo "错误：已有实例运行中" >&2
    exit 1
fi

# 脚本退出时自动释放锁（fd 200 关闭时内核自动释放）
trap 'flock -u 200; rm -f "$LOCK_FILE"' EXIT

# ===== 业务逻辑 =====
echo "PID $$ 开始工作..."
sleep 30
echo "工作完成"
# ===== 结束 =====

# flock 的优势：
# 1. 内核级别锁，进程崩溃时内核自动释放
# 2. 无竞态条件（原子操作）
# 3. 支持共享锁（-s）和排他锁（-x）
# 4. 支持超时等待（-w seconds）
```

**锁文件机制的常见陷阱：**

```bash
# 陷阱一：竞态条件（检查与创建非原子）
# 错误写法：
if [ ! -f lockfile ]; then
    echo $$ > lockfile    # ⚠️ 在「检查」和「创建」之间的时间窗口内，另一个进程可能也创建了锁
fi

# 正确写法：用 mkdir 的原子性
if mkdir /var/run/myapp.lock 2>/dev/null; then
    trap 'rmdir /var/run/myapp.lock' EXIT
    # 业务逻辑
else
    echo "已有实例运行" >&2
    exit 1
fi
# mkdir 是原子操作，两个进程无法同时创建同名目录

# 或使用 flock（最佳方案）
```

## 6.2 设计一个日志轮转脚本

```bash
#!/bin/bash
# 通用日志轮转脚本

set -euo pipefail

# ===== 配置参数 =====
LOG_DIR="${LOG_DIR:-/var/log/myapp}"
LOG_FILE="${LOG_FILE:-app.log}"
MAX_SIZE_MB="${MAX_SIZE_MB:-100}"              # 单文件最大大小（MB）
MAX_FILES="${MAX_FILES:-7}"                    # 保留的旧日志份数
COMPRESS="${COMPRESS:-true}"                   # 是否压缩旧日志
TIMESTAMP_FORMAT="%Y%m%d_%H%M%S"

readonly FULL_PATH="${LOG_DIR}/${LOG_FILE}"

# ===== 工具函数 =====
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >&2; }
file_size_mb() { stat -c %s "$1" 2>/dev/null | awk '{printf "%d", $1/1048576}'; }

# ===== 检测是否需要轮转 =====
need_rotate() {
    [ ! -f "$FULL_PATH" ] && return 1

    local size_mb
    size_mb=$(file_size_mb "$FULL_PATH")

    if [ "$size_mb" -ge "$MAX_SIZE_MB" ]; then
        log "日志大小 ${size_mb}MB >= ${MAX_SIZE_MB}MB，触发轮转"
        return 0
    fi

    return 1
}

# ===== 执行轮转 =====
rotate_logs() {
    log "开始日志轮转..."

    local timestamp
    timestamp=$(date +"$TIMESTAMP_FORMAT")
    local rotated_file="${FULL_PATH}.${timestamp}"

    # 1. 重命名当前日志文件
    mv "$FULL_PATH" "$rotated_file"
    log "当前日志重命名为：$rotated_file"

    # 2. 创建新的空日志文件（保持原有权限）
    touch "$FULL_PATH"
    if [ -n "${LOG_OWNER:-}" ]; then
        chown "${LOG_OWNER}" "$FULL_PATH"
    fi
    log "新日志文件已创建"

    # 3. 通知应用程序重新打开日志（可选）
    if [ "${RELOAD_CMD:-}" ]; then
        log "发送重载信号..."
        eval "$RELOAD_CMD" || log "警告：重载命令失败"
    fi

    # 4. 压缩旧日志
    if [ "$COMPRESS" = "true" ] && command -v gzip &>/dev/null; then
        gzip "$rotated_file" &
        log "后台压缩：${rotated_file}.gz"
    fi

    # 5. 清理超出保留数量的旧日志
    clean_old_logs
}

# ===== 清理旧日志 =====
clean_old_logs() {
    log "清理旧日志（保留最近 ${MAX_FILES} 份）..."

    # 查找所有轮转日志文件（含 .gz）
    local old_logs
    old_logs=$(find "$LOG_DIR" -maxdepth 1 \
        -name "${LOG_FILE}.*" \
        -type f \
        -printf '%T@ %p\n' 2>/dev/null \
        | sort -rn \
        | tail -n +$((MAX_FILES + 1)) \
        | cut -d' ' -f2-)

    if [ -n "$old_logs" ]; then
        echo "$old_logs" | while read -r file; do
            log "删除旧日志：$file"
            rm -f "$file"
        done
    else
        log "无需清理"
    fi
}

# ===== 按日期轮转（每日轮转模式） =====
daily_rotate() {
    local today
    today=$(date +%Y%m%d)
    local last_rotate_file="${LOG_DIR}/.last_rotate_date"

    # 读取上次轮转日期
    local last_date=""
    [ -f "$last_rotate_file" ] && last_date=$(cat "$last_rotate_file")

    if [ "$today" = "$last_date" ]; then
        log "今日已轮转，跳过"
        return 0
    fi

    # 重命名日志
    local rotated="${FULL_PATH}.${today}"
    if [ -f "$FULL_PATH" ]; then
        mv "$FULL_PATH" "$rotated"
        touch "$FULL_PATH"

        [ "$COMPRESS" = "true" ] && gzip -f "$rotated" &
    fi

    # 记录轮转日期
    echo "$today" > "$last_rotate_file"

    # 清理 N 天前的日志
    find "$LOG_DIR" -maxdepth 1 -name "${LOG_FILE}.*.gz" -mtime +"$MAX_FILES" -delete 2>/dev/null
}

# ===== 主流程 =====
main() {
    mkdir -p "$LOG_DIR"

    # 读取命令行参数
    while getopts "fd" opt; do
        case $opt in
            f) force_rotate=true ;;
            d) daily_mode=true ;;
            *) echo "用法: $0 [-f 强制轮转] [-d 每日模式]" >&2; exit 1 ;;
        esac
    done

    if [ "${daily_mode:-false}" = "true" ]; then
        daily_rotate
    elif [ "${force_rotate:-false}" = "true" ] || need_rotate; then
        rotate_logs
    else
        log "无需轮转"
    fi
}

main "$@"
```

## 6.3 设计一个并行任务执行框架

```bash
#!/bin/bash
# 基于后台进程和 wait 的并行任务框架

set -euo pipefail

# ===== 配置 =====
MAX_JOBS="${MAX_JOBS:-4}"          # 最大并发数
JOB_TIMEOUT="${JOB_TIMEOUT:-300}"  # 单个任务超时（秒）
LOG_DIR="${LOG_DIR:-/tmp/parallel_logs}"

# ===== 全局状态 =====
declare -a PIDS=()                 # 运行中的后台进程 PID 数组
declare -a JOB_STATUS=()           # 每个任务的退出状态码
FAILURE_COUNT=0
SUCCESS_COUNT=0
TOTAL_JOBS=0

# ===== 工具函数 =====
log()  { echo "[$(date '+%H:%M:%S')] [$1] $2"; }
job_log() {
    local job_id="$1"
    local message="$2"
    log "JOB-$job_id" "$message"
}

# ===== 等待一个任务完成 =====
wait_for_slot() {
    # 如果当前运行数已达上限，等待至少一个任务完成
    while [ ${#PIDS[@]} -ge "$MAX_JOBS" ]; do
        for i in "${!PIDS[@]}"; do
            local pid="${PIDS[$i]}"
            if ! kill -0 "$pid" 2>/dev/null; then
                # 进程已结束，回收
                wait "$pid" 2>/dev/null
                local exit_code=$?

                # 记录任务状态
                local job_id=$((i + 1))
                if [ $exit_code -eq 0 ]; then
                    job_log "$job_id" "完成（PID: $pid）"
                    ((SUCCESS_COUNT++))
                else
                    job_log "$job_id" "失败（PID: $pid，退出码: $exit_code）"
                    ((FAILURE_COUNT++))
                fi

                # 从数组中移除
                unset "PIDS[$i]"
                PIDS=("${PIDS[@]}")   # 重新索引
                return
            fi
        done
        sleep 0.1   # 短暂休眠，避免忙等
    done
}

# ===== 提交单个任务 =====
submit_job() {
    local job_id="$1"
    local command="$2"
    local log_file="${LOG_DIR}/job_${job_id}.log"

    # 等待有空闲槽位
    wait_for_slot

    # 后台执行任务，输出重定向到日志文件
    mkdir -p "$LOG_DIR"
    (
        {
            echo "===== 任务 $job_id 开始于 $(date) ====="
            echo "命令: $command"
            echo "===== 输出 ====="
            eval "timeout $JOB_TIMEOUT $command"
            local exit_code=$?
            echo "===== 任务 $job_id 结束，退出码: $exit_code ====="
            exit $exit_code
        } >> "$log_file" 2>&1
    ) &
    local pid=$!

    PIDS+=("$pid")
    job_log "$job_id" "已启动（PID: $pid）"
    return 0
}

# ===== 等待所有任务完成 =====
wait_all_jobs() {
    log "INFO" "等待所有任务完成（剩余: ${#PIDS[@]}）..."

    for pid in "${PIDS[@]}"; do
        wait "$pid" 2>/dev/null
        local exit_code=$?

        if [ $exit_code -eq 0 ]; then
            ((SUCCESS_COUNT++))
        else
            ((FAILURE_COUNT++))
        fi
    done

    PIDS=()
    log "INFO" "全部任务已完成"
}

# ===== 主框架 =====
main() {
    log "INFO" "并行任务框架启动（最大并发: $MAX_JOBS）"

    # 读取任务列表（从参数或标准输入）
    local tasks=()
    if [ $# -gt 0 ]; then
        tasks=("$@")
    else
        # 从 stdin 读取任务
        while IFS= read -r line; do
            [ -n "$line" ] && tasks+=("$line")
        done
    fi

    TOTAL_JOBS=${#tasks[@]}
    log "INFO" "共 $TOTAL_JOBS 个任务"

    # 提交所有任务
    local job_index=0
    for task in "${tasks[@]}"; do
        ((job_index++))
        submit_job "$job_index" "$task"
    done

    # 等待全部完成
    wait_all_jobs

    # 输出汇总报告
    echo ""
    echo "======================================"
    echo "  任务执行汇总"
    echo "======================================"
    echo "  总任务数: $TOTAL_JOBS"
    echo "  成功: $SUCCESS_COUNT"
    echo "  失败: $FAILURE_COUNT"
    echo "  日志目录: $LOG_DIR"
    echo "======================================"

    return $FAILURE_COUNT
}

# === 使用示例 ===
# 方式一：传递命令列表
# bash parallel.sh "sleep 3" "sleep 2" "sleep 1" "sleep 4"

# 方式二：从文件读取
# cat tasks.txt | bash parallel.sh
# tasks.txt 内容：
# curl -s https://api1.example.com
# curl -s https://api2.example.com
# curl -s https://api3.example.com

# 方式三：在脚本中编程式调用
# for url in "${urls[@]}"; do
#     submit_job "$((++counter))" "curl -fsSL -o /dev/null $url"
# done

main "$@"
```

```bash
# === 进阶：使用 GNU parallel（专业的并行工具） ===
# 如果环境允许安装 GNU parallel，它可以替代上述脚本
# 安装：apt install parallel / brew install parallel

# 基本用法
# parallel -j 4 curl -s {} ::: https://api1.example.com https://api2.example.com

# 从文件读取
# parallel -j 10 --joblog jobs.log < tasks.txt

# 带超时和重试
# parallel -j 5 --timeout 60 --retries 3 < tasks.txt

# 带进度条
# parallel -j 8 --bar --joblog jobs.log ::: $(seq 1 10000)

# 如需控制并发顺序和依赖，考虑 xargs：
# xargs -P 4 -I {} sh -c 'curl -s "{}"' < urls.txt
```

**并行框架的关键设计点：**

1. **槽位控制**：维护 PID 数组，当数组长度达到上限时，通过轮询等待释放
2. **任务隔离**：每个任务在独立子 shell 执行，输出重定向到独立日志文件
3. **超时保护**：每个任务用 `timeout` 或 `sleep + kill` 做超时控制
4. **清理陷阱**：`trap` 确保退出时清理所有后台进程
5. **失败汇总**：记录每个任务的退出码，最终生成汇总报告

---

# 7. 附录：面试速查表

## 7.1 最易出错的知识点 Top 10

| 排名 | 知识点 | 关键要点 |
|------|--------|----------|
| 1 | 变量引用必须加双引号 | `"$var"` 而非 `$var` |
| 2 | 管道创建子 shell | while 用 `< <(cmd)` 代替 `cmd \| while` |
| 3 | `[ ]` 中的空格规则 | `[ "$a" = "$b" ]` 必须有空格 |
| 4 | `set -e` 的例外情况 | if/while/\|\|/&&/管道 中不生效 |
| 5 | 重定向顺序 | `> file 2>&1` 正确，`2>&1 > file` 错误 |
| 6 | `[ -n "$var" ]` vs `[ -n $var ]` | 不加引号空变量导致永真 |
| 7 | `$@` vs `$*` | `"$@"` 保持参数独立性 |
| 8 | [[ ]] vs [ ] | `[[` 是关键字，更安全，支持 `=~` 正则 |
| 9 | shebang 不能有空格 | `#!/bin/bash` 必须是文件第一个字节 |
| 10 | exit vs return | exit 退出脚本，return 退出函数 |

## 7.2 常用一行命令（oneliner）速查

```bash
# 查找并替换
find . -name "*.txt" -exec sed -i 's/old/new/g' {} \;

# 统计代码行数
find . -name "*.sh" | xargs wc -l | tail -1

# 批量重命名
for f in *.jpg; do mv "$f" "${f/.jpg/_backup.jpg}"; done

# 查找大文件
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# 杀死所有匹配进程
ps aux | grep 'process_name' | awk '{print $2}' | xargs kill -9

# 监控日志实时变化
tail -f /var/log/syslog | grep --line-buffered "ERROR"

# JSON 格式化
curl -s https://api.example.com/data | python3 -m json.tool

# 下载并解压
curl -sL https://example.com/file.tar.gz | tar xz

# 生成随机密码
openssl rand -base64 32 | tr -d '/+=' | cut -c1-16

# 查找重复文件
find . -type f -exec md5sum {} \; | sort | uniq -w32 -d
```
