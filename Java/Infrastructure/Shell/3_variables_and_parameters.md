
# 1. 变量基础

## 1.1 定义与赋值

```bash
# 基本赋值（等号前后不能有空格！）
name="Alice"
age=25
pi=3.14

# 命令结果赋值
files=$(ls)
today=$(date +%Y-%m-%d)
md5=$(md5sum file.txt | cut -d' ' -f1)

# 多变量赋值
readonly HOST="localhost"   # 只读变量
declare -i count=0          # 整数变量
export PATH="/usr/local/bin:$PATH"  # 导出为环境变量
```

**赋值的核心规则：等号两边不能有空格，否则 Shell 会将变量名解释为命令。**

```bash
name = "Alice"   # 错误！Shell 试图执行命令 name
name="Alice"     # 正确
```

## 1.2 变量引用

```bash
echo $name        # 基础引用
echo ${name}      # 花括号引用（推荐，避免歧义）

# 何时必须使用花括号
echo "${name}file"   # Alicefile（正确）
echo "$namefile"     # 空（shell 找的是 $namefile 变量）

food="apple"
echo "I like ${food}s"  # I like apples
echo "I like $foods"    # I like（$foods 未定义）
```

## 1.3 变量作用域

```bash
name="global"        # 全局变量

func() {
    local name="local"   # 局部变量（只能在函数内使用 local）
    echo "$name"         # local
}

func
echo "$name"         # global（局部变量的修改不影响全局）
```

**关键点：**
- Bash 中变量默认是**全局的**（即使在函数内定义）
- 只有使用 `local`/`declare` 声明的变量才是局部的
- `export` 让变量对子进程可见
- `readonly` 让变量不可修改

# 2. 环境变量 vs Shell 变量

| 特性 | Shell 变量 | 环境变量 |
|------|-----------|----------|
| 作用域 | 当前 Shell | 当前 Shell + 所有子进程 |
| 创建方式 | `name=value` | `export name=value` |
| 查看 | `echo $name` | `env` / `printenv` |
| 传递性 | 不传递到子进程 | 传递给子进程 |

```bash
# 演示区别
shell_var="only here"
export env_var="everywhere"

bash -c 'echo "shell_var: $shell_var"'   # 空
bash -c 'echo "env_var: $env_var"'       # everywhere
```

**常见重要环境变量：**

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `HOME` | 当前用户主目录 | `/home/alice` |
| `PATH` | 命令搜索路径 | `/usr/local/bin:/usr/bin:/bin` |
| `USER`/`LOGNAME` | 当前用户名 | `alice` |
| `SHELL` | 当前 Shell | `/bin/bash` |
| `PWD` | 当前工作目录 | `/home/alice/project` |
| `OLDPWD` | 上一个工作目录 | `/home/alice` |
| `IFS` | 内部字段分隔符 | `$' \t\n'` (空格 Tab 换行) |
| `LANG` | 语言/区域设置 | `en_US.UTF-8` |
| `PS1` | 主提示符 | `\u@\h:\w\$` |
| `RANDOM` | 随机数 (bash) | 每次引用返回不同值 |
| `SECONDS` | 脚本运行秒数 (bash) | 自增计数 |

# 3. 位置参数

```bash
#!/bin/bash
# 脚本名：test.sh
# 执行：./test.sh aaa bbb ccc ddd

echo "\$0 = $0"     # ./test.sh（脚本名）
echo "\$1 = $1"     # aaa（第 1 个参数）
echo "\$2 = $2"     # bbb（第 2 个参数）
echo "\$3 = $3"     # ccc
echo "\$4 = $4"     # ddd
echo "\$# = $#"     # 4（参数个数）
echo "\$@ = $@"     # aaa bbb ccc ddd
echo "\$* = $*"     # aaa bbb ccc ddd
echo "\$$ = $$"     # 2555（当前 Shell 的 PID）
echo "\$! = $!"     # 最近一个后台进程的 PID
echo "\$- = $-"     # himBHs（当前 Shell 选项）
echo "\$? = $?"     # 上一个命令的退出码
```

## 3.1 `$@` vs `$*` 的区别

这是 Shell 中最精微的区别之一，只在**双引号**中体现：

```bash
#!/bin/bash
echo "=== \$@ 与 \$* 的区别 ==="

# 不加引号：两者完全相同
for arg in $@; do echo "[$arg]"; done   # 逐词展开
for arg in $*; do echo "[$arg]"; done   # 逐词展开（一样）

# 加双引号：行为不同
echo "--- \"\$@\" ---"
for arg in "$@"; do echo "[$arg]"; done
# [aaa]     ← 每个参数独立保留

echo "--- \"\$*\" ---"
for arg in "$*"; do echo "[$arg]"; done
# [aaa bbb ccc ddd]  ← 所有参数合并为一个字符串，用 IFS 第一个字符连接
```

**黄金法则：传递给函数或脚本的参数，永远使用 `"$@"`。** 它是唯一能保留每个参数独立性的写法。

## 3.2 shift 移位参数

```bash
#!/bin/bash
# shift 将参数左移，丢弃最左边的参数

while [ $# -gt 0 ]; do
    echo "当前参数: $1，剩余: $#"
    shift
done

# shift N 一次移动 N 个参数
echo "原始: $1 $2 $3 $4"
shift 2
echo "移位2后: $1 $2"
```

## 3.3 set 设置位置参数

```bash
set -- "arg1" "arg2" "arg3"
echo $1  # arg1
echo $2  # arg2

# 实际用法：用 set 操作命令输出
set -- $(date)
echo $1  # Mon
echo $2  # May
echo $6  # 2025

# set -- 的安全写法（-- 防止参数被当作选项）
set -- "-n"    # 把 -n 赋值给 $1，而不是 set -n 选项
```

# 4. 变量展开（Parameter Expansion）

这是 Bash 最强大的特性之一，`${var...}` 提供了丰富的字符串操作。

## 4.1 赋默认值

```bash
# ${var:-default}  —— 如果 var 未设置或为空，返回 default（不改变 var）
echo ${name:-"Anonymous"}   # Anonymous（name 仍是空）

# ${var-default}  —— 仅当 var 未设置时返回 default（var 为空字符串时不触发）
unset name
echo ${name-"set"}    # set
name=""
echo ${name-"set"}    # （空，因为 name 已设置，只不过是空字符串）

# ${var:=default}  —— 如果 var 未设置或为空，赋值并返回 default
echo ${count:=0}       # 0（count 被赋值为 0）

# ${var:?message}  —— 如果 var 未设置或为空，打印错误并退出
echo ${DB_HOST:?必须设置数据库地址}
```

## 4.2 字符串长度

```bash
str="hello world"
echo ${#str}          # 11

# 数组长度
arr=(a b c d)
echo ${#arr[@]}       # 4
echo ${#arr[0]}       # 1（第一个元素的长度）
```

## 4.3 子串提取与删除

```bash
path="/home/user/document.pdf"

# 提取子串：${var:offset:length}
echo ${path:6}        # user/document.pdf（从位置6到末尾）
echo ${path:6:4}      # user（从位置6取4个字符）
echo ${path: -4}      # .pdf（负偏移取末尾，注意空格）
echo ${path: -4:3}    # .pd

# 从开头删除最短匹配：${var#pattern}
echo ${path#*/}       # home/user/document.pdf

# 从开头删除最长匹配：${var##pattern}
echo ${path##*/}      # document.pdf（文件名）

# 从末尾删除最短匹配：${var%pattern}
echo ${path%.*}       # /home/user/document

# 从末尾删除最长匹配：${var%%pattern}
echo ${path%%.*}      # /home/user/document（同上，但针对最后一个.）

# 实际技巧：快速获取文件名、扩展名、目录
file="backup.tar.gz"

dirname=${file%/*}           # /home/user
basename=${file##*/}         # backup.tar.gz
name=${basename%%.*}         # backup（去所有扩展名）
extension=${basename#*.}     # tar.gz（保留完整扩展名）
simple_ext=${file##*.}       # gz（最后一个扩展名）
```

## 4.4 查找与替换

```bash
text="hello world, hello universe"

# ${var/pattern/replacement}  —— 替换第一个匹配
echo ${text/hello/hi}        # hi world, hello universe

# ${var//pattern/replacement} —— 替换所有匹配
echo ${text//hello/hi}       # hi world, hi universe

# ${var/#pattern/replacement} —— 只在开头替换
echo ${text/#hello/hi}       # hi world, hello universe
echo ${text/#world/hi}       # hello world, hello universe（world不在开头）

# ${var/%pattern/replacement} —— 只在结尾替换
echo ${text/%universe/mars}  # hello world, hello mars
```

## 4.5 大小写转换（Bash 4.0+）

```bash
name="john doe"

echo ${name^}       # John doe（首字母大写）
echo ${name^^}      # JOHN DOE（全大写）
echo ${name,}       # john doe（首字母小写）
echo ${name,,}      # john doe（全小写）

# 大写风格切换
camel="camelCaseVar"
# 简单的写法：${camel~~} 反转大小写
echo ${camel~~}     # CAMELcASEvAR
```

# 5. 算术展开

## 5.1 整数运算

```bash
# $(( )) 算术展开（支持整数运算）
echo $((2 + 3))      # 5

a=10
b=3
echo $((a + b))      # 13
echo $((a - b))      # 7
echo $((a * b))      # 30
echo $((a / b))      # 3（整数除法，向下取整）
echo $((a % b))      # 1（取模）
echo $((a ** b))     # 1000（乘方，bash 4.2+）
echo $((a++))        # 10（后置自增）
echo $((++a))        # 12（前置自增）

# 位运算
echo $((1 << 3))     # 8（左移）
echo $((8 >> 2))     # 2（右移）
echo $((5 & 3))      # 1（按位与）
echo $((5 | 3))      # 7（按位或）
echo $((5 ^ 3))      # 6（按位异或）
echo $((~0))         # -1（按位取反）
```

## 5.2 let 命令

```bash
let "a = 1 + 2"    # 等价于 ((a = 1 + 2))
let a++             # 自增
let "a += 5"        # 加等于
```

## 5.3 浮点数运算

Bash **不支持**浮点数运算，需要使用外部工具：

```bash
# 使用 bc（高精度计算器）
echo "scale=2; 10/3" | bc          # 3.33
echo "scale=4; sqrt(2)" | bc -l    # 1.4142（-l 加载数学库）

# 使用 awk
awk 'BEGIN {print 10/3}'           # 3.33333
awk 'BEGIN {printf "%.2f", 10/3}'  # 3.33

# 使用 python
python3 -c "print(10/3)"

# 使用 printf（有限精度）
printf "%.2f\n" $((100 * 10 / 3))e-2  # 不精确，不推荐
```

# 6. declare/typeset 类型声明

```bash
declare -i num=10       # 整数
declare -r PI=3.14      # 只读
declare -l name="HELLO"  # 小写（赋值时自动转换）
declare -u name="hello"  # 大写（赋值时自动转换）
declare -a arr           # 索引数组
declare -A map           # 关联数组
declare -x VAR           # 导出为环境变量（等价 export）
declare -g VAR=value     # 在函数内声明全局变量
declare -p VAR           # 打印变量的属性和值

# 查看变量属性
declare -p PATH          # declare -x PATH="..."
```

# 7. 变量展开的执行顺序

这是理解 Shell 行为的关键。以下展开依序发生：

```
1. 大括号展开    echo {1..3}      → 1 2 3
2. 波浪号展开    echo ~/dir       → /home/user/dir
3. 参数/变量展开  echo $var       → value
4. 命令替换      echo $(cmd)     → cmd输出
5. 算术展开      echo $((1+2))   → 3
6. 单词分割      基于 IFS 分割
7. 文件名展开    echo *.txt      → a.txt b.txt
8. 引号移除      去除用于引用的引号字符
```

**重要影响：**
- 变量展开（步骤3）在文件名展开（步骤7）之前，所以 `$pattern` 中的 `*` 会被展开
- 单词分割在参数展开之后，所以不加双引号的变量会被分割
- 算术展开在变量展开之后，可以在 `$((var+1))` 中使用变量

# 8. IFS 内部字段分隔符

`IFS` 决定了 Shell 如何**分割**单词和处理 `read` 命令。

```bash
# 默认 IFS: 空格 + Tab + 换行
echo "$IFS" | cat -A     # ^I$（肉眼看不见，cat -A 展示）

# IFS 影响 $* 的连接
IFS=":"; echo "$*"         # arg1:arg2:arg3

# IFS 影响 read
IFS=":" read -r user pass uid gid desc home shell <<< "root:x:0:0:root:/root:/bin/bash"
echo $user   # root
echo $shell  # /bin/bash

# 按行读取（临时修改 IFS，读完后自动恢复）
while IFS= read -r line; do
    echo "[$line]"
done < file.txt
```

**`while IFS= read -r line` 是读取文件的最安全写法：**
- `IFS=` 禁止单词分割，保留行首/行尾空白
- `-r` 禁止反斜杠转义

# 9. 间接引用

```bash
var_name="PATH"
echo "${!var_name}"       # 输出 PATH 的值（间接引用）

# 创建动态变量名（使用 declare）
prefix="user_"
name="alice"
declare "${prefix}${name}=123"
echo "$user_alice"        # 123

# 使用 nameref（bash 4.3+）
declare -n ref=varname    # ref 是 varname 的引用
ref="new value"
echo "$varname"           # new value
```
