
# 1. 概述

Shell 条件判断通过**退出状态码**确定 true/false（0 = true，非 0 = false），这与大多数编程语言相反。

```bash
# shell 的真假逻辑
true;  echo $?  # 0（真）
false; echo $?  # 1（假）

if true; then echo "执行了"; fi     # 执行了
if false; then echo "不执行"; fi    # 不执行
```

# 2. test 命令与方括号

## 2.1 三种等价写法

```bash
test -f file.txt      # test 命令
[ -f file.txt ]       # [ 是 test 的别名（需要一个 ] 作为最后一个参数）
[[ -f file.txt ]]     # bash 扩展（更强，更安全）
```

**关键区别：`[` 是命令，`[[` 是关键字。**

```bash
type [     # [ is a shell builtin
type [[    # [[ is a shell keyword
```

## 2.2 `[ ]` 的注意事项

```bash
# [ 的语法要求
[ "$a" = "$b" ]       # 正确（参数之间有空格）
[ "$a"="$b" ]         # 错误！[ 只收到一个参数 "[a=b]"
[ "$a" = "$b"]        # 错误！缺少 ] 前的空格

# 必须加双引号（否则空变量导致语法错误）
a=""
[ $a = "test" ]       # 展开后：[ = test ] —— 语法错误！
[ "$a" = "test" ]     # 正确：[ "" = "test" ]

# 一个常见但危险的简写
[ -n "$var" ]         # 检查 var 是否非空（正确）
[ -n $var ]           # 如果 var 为空：变成 [ -n ] —— 这是真值！（错误）
```

# 3. 字符串比较

```bash
a="hello"
b="world"

# 相等比较
[ "$a" = "$b" ]       # test 标准写法
[ "$a" == "$b" ]      # bash 支持（单 = 更可移植）
[[ "$a" == "$b" ]]    # bash 扩展
[[ "$a" = "$b" ]]     # 也可以

# 不等比较
[ "$a" != "$b" ]

# 字典序比较（[[ ]] 专有）
[[ "$a" < "$b" ]]     # a 在 b 之前（字典序）
[[ "$a" > "$b" ]]     # a 在 b 之后

# 注意：在 [ ] 中 < 和 > 是重定向！
[ "a" < "b" ]         # 危险！会重定向到文件名为 "b" 的文件！
```

**`=` vs `==`：**

| | `[ ]` | `[[ ]]` |
|--|-------|---------|
| 字符串相等 | `=`（POSIX） | `=` 或 `==` 均可 |

**建议：字符串比较总是使用 `[[ ]]`，因为：**
1. 不需要加引号防止空变量错误
2. 支持 `=~` 正则匹配
3. 支持 `&&` `||` 逻辑运算符
4. `<` `>` 不会被解释为重定向

# 4. 数值比较

```bash
# [ ] / test 的数值运算符
[ 3 -eq 3 ]   # equal（相等）
[ 3 -ne 4 ]   # not equal（不等）
[ 3 -gt 2 ]   # greater than（大于）
[ 3 -ge 2 ]   # greater or equal（大于等于）
[ 3 -lt 4 ]   # less than（小于）
[ 3 -le 4 ]   # less or equal（小于等于）

# [[ ]] 也可以用上述运算符，但推荐用 (( )) 做数值比较
(( 3 == 3 ))     # 相等
(( 3 != 4 ))     # 不等
(( 3 > 2 ))      # 大于
(( a >= b ))     # 大于等于
(( a < b ))      # 小于
(( a <= b ))     # 小于等于

# (( )) 的优势：直接用 C 风格语法
if (( a > b && b < c )); then
    echo "a 最大"
fi
```

**三种数值比较方式的对照：**

```bash
a=10; b=20
[ "$a" -gt "$b" ]      # test 风格（兼容性最好）
[[ $a -gt $b ]]        # [[ 风格（无需引号）
(( a > b ))             # 算术风格（最简洁的写法）
```

# 5. 文件测试

```bash
# 存在性
[ -e file ]      # 存在（任何类型）
[ -f file ]      # 存在且是普通文件
[ -d dir  ]      # 存在且是目录
[ -L file ]      # 存在且是符号链接
[ -h file ]      # 同上（-h 是 -L 的别名）

# 权限
[ -r file ]      # 可读
[ -w file ]      # 可写
[ -x file ]      # 可执行
[ -u file ]      # SUID 位设置
[ -g file ]      # SGID 位设置
[ -k file ]      # Sticky bit 设置
[ -O file ]      # 当前用户是文件所有者
[ -G file ]      # 当前用户组是文件所在组

# 大小与内容
[ -s file ]      # 存在且大小 > 0（非空）
[ file1 -nt file2 ]  # file1 比 file2 新（newer than）
[ file1 -ot file2 ]  # file1 比 file2 旧（older than）
[ file1 -ef file2 ]  # file1 和 file2 指向同一 inode

# 文件描述符
[ -t fd ]        # 文件描述符是否关联到终端
[ -t 0 ]         # 标准输入是否是终端（判断是否管道）

# 管道检测
if [ -t 0 ]; then
    echo "交互模式"
else
    echo "管道输入模式"
fi
```

## 5.1 文件测试实战模式

```bash
# 常见的防御性文件检测
config_file="/etc/myapp/config.conf"

if [ ! -f "$config_file" ]; then
    echo "配置文件 $config_file 不存在"
    exit 1
fi

if [ ! -r "$config_file" ]; then
    echo "配置文件 $config_file 不可读"
    exit 1
fi

# 创建目录的安全方式
data_dir="/var/lib/myapp/data"
if [ ! -d "$data_dir" ]; then
    mkdir -p "$data_dir" || { echo "无法创建目录"; exit 1; }
fi

# 检测命令是否存在
if [ -x "$(command -v docker)" ]; then
    docker ps
else
    echo "Docker 未安装"
fi
```

# 6. 逻辑运算

## 6.1 test / [ ] 的逻辑运算符

```bash
# 使用 -a (AND) 和 -o (OR)
[ "$a" = "x" -a "$b" = "y" ]   # 两条件都成立
[ "$a" = "x" -o "$b" = "y" ]   # 任一条件成立

# 使用 [ ] [ ] 组合（更可读，推荐）
[ "$a" = "x" ] && [ "$b" = "y" ]
[ "$a" = "x" ] || [ "$b" = "y" ]

# 使用否定 !（注意空格）
[ ! -f file.txt ]
[ ! "$a" = "$b" ]
```

## 6.2 [[ ]] 的逻辑运算符

```bash
# 直接使用 && 和 ||
[[ $a == "x" && $b == "y" ]]
[[ $a == "x" || $b == "y" ]]

# 分组
[[ ($a == "x" || $a == "y") && $b == "z" ]]

# 否定
[[ ! $a == "x" ]]
```

## 6.3 (( )) 的逻辑运算符

```bash
(( a > 1 && b < 10 ))
(( a > 1 || b < 10 ))
(( ! a > 1 ))          # 非
```

# 7. if 语句

## 7.1 基本语法

```bash
if 条件; then
    命令
fi

if 条件; then
    命令
else
    命令
fi

if 条件1; then
    命令1
elif 条件2; then
    命令2
elif 条件3; then
    命令3
else
    默认命令
fi
```

**注意：`then` 前必须有 `;` 或换行。`fi` 是 `if` 的反写。**

## 7.2 实际模式

```bash
# 模式 1：命令直接作为条件
if grep -q "pattern" file.txt; then
    echo "找到了"
fi

# 模式 2：检查变量
if [ -z "$var" ]; then
    echo "var 为空"
fi

# 模式 3：多条链式判断
if [ -f config.conf ]; then
    source config.conf
elif [ -f /etc/myapp/config.conf ]; then
    source /etc/myapp/config.conf
else
    echo "找不到配置文件"
    exit 1
fi

# 模式 4：用!取反
if ! ping -c 1 8.8.8.8 > /dev/null; then
    echo "无法互联网连接"
fi
```

# 8. case 语句

```bash
case 表达式 in
    模式1)
        命令
        ;;
    模式2 | 模式3)
        命令
        ;;
    *)
        默认命令
        ;;
esac
```

## 8.1 实际案例

```bash
# case 的网络服务管理
service=$1

case "$service" in
    start)
        echo "启动服务..."
        ;;
    stop)
        echo "停止服务..."
        ;;
    restart|reload)
        echo "重启服务..."
        ;;
    status)
        echo "查看状态..."
        ;;
    *)
        echo "用法: $0 {start|stop|restart|reload|status}"
        exit 1
        ;;
esac
```

## 8.2 高级模式匹配

```bash
# case 支持通配符匹配
read -p "输入 y/n: " answer
case "$answer" in
    [Yy] | [Yy][Ee][Ss]) echo "你选了是" ;;
    [Nn] | [Nn][Oo])     echo "你选了否" ;;
    *)                   echo "无效输入" ;;
esac

# 按扩展名判断文件类型
file="document.tar.gz"
case "$file" in
    *.tar.gz | *.tgz)     echo "tar 压缩包" ;;
    *.zip)                echo "zip 压缩包" ;;
    *.jpg | *.jpeg | *.png) echo "图片文件" ;;
    *.sh)                 echo "Shell 脚本" ;;
    *)                    echo "其他文件" ;;
esac

# 匹配变量是否包含子串
case "$PATH" in
    */usr/local/bin*) echo "PATH 包含 /usr/local/bin" ;;
    *)                echo "PATH 不包含 /usr/local/bin" ;;
esac
```

# 9. `[[ ]]` 高级匹配

## 9.1 模式匹配

```bash
# 通配符匹配
[[ "hello.txt" == *.txt ]]     # true
[[ "hello.txt" == *.log ]]     # false

# 多模式
[[ $var == @(pattern1|pattern2) ]]  # 匹配任一（extglob 需要开启）
```

## 9.2 正则表达式匹配 `=~`

```bash
# 基本正则匹配
[[ "abc123" =~ ^[a-z]+[0-9]+$ ]]  # true

# 捕获组
if [[ "2025-05-17" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    year=${BASH_REMATCH[1]}    # 2025
    month=${BASH_REMATCH[2]}   # 05
    day=${BASH_REMATCH[3]}     # 17
fi

# 正则变量
regex="^[a-z]+$"
[[ "hello" =~ $regex ]]       # 使用变量

# 注意：不要给正则表达式加引号，否则会被当作字面量
[[ "hello" =~ "^[a-z]" ]]     # false（"^[a-z]" 被当作字面量）
[[ "hello" =~ ^[a-z] ]]       # true（无引号才是正则）
```

**`BASH_REMATCH`** 数组保存了最近一次 `=~` 匹配的捕获结果：
- `BASH_REMATCH[0]`：完整匹配
- `BASH_REMATCH[1]`：第一个捕获组
- `BASH_REMATCH[2]`：第二个捕获组
- ...

# 10. 条件表达式速查

## 10.1 字符串测试

| 表达式 | 含义 |
|--------|------|
| `-z "$str"` | 字符串为空（长度 0） |
| `-n "$str"` | 字符串非空（长度 > 0） |
| `"$s1" = "$s2"` | 字符串相等 |
| `"$s1" != "$s2"` | 字符串不等 |

## 10.2 数值测试

| 表达式 | 含义 |
|--------|------|
| `n1 -eq n2` | 等于 |
| `n1 -ne n2` | 不等于 |
| `n1 -lt n2` | 小于 |
| `n1 -le n2` | 小于等于 |
| `n1 -gt n2` | 大于 |
| `n1 -ge n2` | 大于等于 |

## 10.3 文件测试

| 表达式 | 含义 |
|--------|------|
| `-e file` | 存在 |
| `-f file` | 普通文件 |
| `-d file` | 目录 |
| `-s file` | 非空文件 |
| `-r file` | 可读 |
| `-w file` | 可写 |
| `-x file` | 可执行 |

# 11. 命令组合的条件判断

```bash
# 利用 && || 实现简单分支
[ -d /backup ] && echo "目录存在" || echo "目录不存在"
# 注意：上例如果 echo "目录存在" 失败了，会错误地执行 || 部分

# 更安全的写法
if [ -d /backup ]; then
    echo "目录存在"
else
    echo "目录不存在"
fi

# 一行 if（适合简单逻辑）
[ -f config ] && source config
[ -d "$dir" ] || mkdir -p "$dir"

# 使用 { } 分组执行多个命令
[ -f config ] && { source config; echo "已加载配置"; }
```
