
# 1. Hello World

```bash
#!/bin/bash
# 这是第一个 Shell 脚本
echo "Hello, World!"
```

**运行方式：**

```bash
# 方式1：直接解释执行（无需执行权限）
bash hello.sh

# 方式2：赋予执行权限后直接运行
chmod +x hello.sh
./hello.sh

# 方式3：source 执行（在当前 Shell 环境中运行）
source hello.sh
. hello.sh
```

三种方式的区别：

| 方式 | 子进程 | 环境变量 | 退出后影响 |
|------|--------|----------|-----------|
| `bash script.sh` | 是（新 bash 子进程） | 继承，修改不传回 | 无 |
| `./script.sh` | 是（fork+exec，shebang决定解释器） | 同上 | 无 |
| `source script.sh` | 否（当前 Shell） | 修改直接生效 | 有（变量、cd 等保留） |

# 2. Shebang 详解

**Shebang（#!）** 是脚本文件的第一行，告诉内核用哪个解释器来执行该脚本。

```bash
#!/bin/bash                  # 最常见的写法
#!/bin/sh                    # POSIX 兼容
#!/usr/bin/env bash          # 从 PATH 中查找 bash（更可移植）
#!/bin/bash -e               # 带选项（遇错退出）
#!/usr/bin/env -S bash -e    # env 传参（-S 拆分参数）
```

**`#!/bin/bash` vs `#!/usr/bin/env bash`：**

| 特性 | `/bin/bash` | `/usr/bin/env bash` |
|------|------------|---------------------|
| 查找方式 | 硬编码路径 | 从 PATH 环境变量查找 |
| 优点 | 确定性强，不受 PATH 影响 | 可移植性好（虚拟环境友好） |
| 缺点 | 不同系统 bash 路径可能不同 | 依赖 PATH，可能找到错误版本 |
| 传参 | 直接 `#!/bin/bash -e` | 需 `#!/usr/bin/env -S bash -e` |

**内核如何处理 shebang：** 当执行 `./script.sh` 时，内核的 `execve()` 系统调用检测到 `#!`，读取 shebang 后的解释器路径，然后将脚本作为参数传递给该解释器，重新执行 `execve("/bin/bash", ["bash", "./script.sh"], envp)`。

# 3. 基本语法规则

## 3.1 命令分隔

```bash
# 单行单命令（换行分隔）
echo "first"
echo "second"

# 分号分隔（同一行多条命令，顺序执行，互不影响）
echo "a"; echo "b"; echo "c"

# && 逻辑与（前命令成功才继续）
mkdir /tmp/test && cd /tmp/test && echo "成功"

# || 逻辑或（前命令失败才执行）
grep "pattern" file.txt || echo "未找到"

# 组合使用
command1 && echo "成功" || echo "失败"
# 注意：echo "成功" 也返回 0，所以上面的逻辑不是 if-else！
```

## 3.2 注释

```bash
# 单行注释

<<'COMMENT'
多行注释（Here Document 方式）
这里的所有内容都是注释
不会被执行
COMMENT

: <<'COMMENT'
另一种多行注释（利用 : 空命令）
COMMENT

# 批量注释技巧：用冒号加 here-doc
: '
这些都是注释
可以跨越多行
'
```

## 3.3 退出状态码

每个命令执行完毕后都返回一个**退出状态码**（0~255）：

- **0**：成功
- **1~255**：失败（1 是通用错误，2 是误用，126 是无法执行，127 是未找到命令，128+N 是被信号 N 终止）

```bash
ls /nonexistent
echo $?    # 输出 2

true
echo $?    # 输出 0

false
echo $?    # 输出 1

exit 42     # 脚本退出并返回 42
```

**128+N 规则：** 如果进程被信号 N 杀死，退出码为 128+N。

```bash
sleep 100 &
kill -9 $!      # 发送 SIGKILL (信号 9)
wait $!
echo $?          # 输出 137 (128 + 9)
```

## 3.4 长命令折行

```bash
# 反斜杠折行（必须紧跟换行符，不能有空格）
echo "这是一条" \
     "很长的命令"

# 管道自然折行（不需要反斜杠）
cat file.txt \
    | grep "pattern" \
    | sort \
    | uniq -c
```

# 4. 引号系统

Shell 有三种引号，行为各不相同，这是 Shell 编程中最容易出错的点之一。

| 引号 | 变量展开 | 命令替换 | 通配符 | 转义 | 用途 |
|------|----------|----------|--------|------|------|
| 双引号 `" "` | 是 | 是 | 否 | 是 | 保留空格/换行，允许变量嵌入 |
| 单引号 `' '` | 否 | 否 | 否 | 否 | 完全字面量，一切皆为普通字符 |
| 反引号 `` ` ` `` | 是 | 是（命令替换） | 否 | 是 | 命令替换（不推荐，用 `$()` 代替） |

```bash
name="Alice"
echo "Hello $name"       # Hello Alice  （变量展开）
echo 'Hello $name'       # Hello $name  （字面量）
echo "Today is $(date)"  # Today is Mon ... （命令替换）
echo 'Today is $(date)'  # Today is $(date) （字面量）
```

**双引号的必要性（避免单词分割）：**

```bash
# 危险：不加双引号导致单词分割
files="file1.txt file2.txt"
for f in $files; do echo "$f"; done   # 输出两行（正确，碰巧）
rm $files                              # 删除 file1.txt 和 file2.txt

files="my important file.txt"
rm $files         # 试图删除 my、important、file.txt 三个文件（错误！）
rm "$files"       # 正确：删除 "my important file.txt"
```

**核心原则：除非明确需要单词分割或通配符展开，否则始终给变量加双引号。**

# 5. 转义与特殊字符

```bash
# 反斜杠转义单个字符
echo "Price: \$100"       # Price: $100
echo "Tab: \t, Newline: \n"  # Tab: 	, Newline: 

# echo 的 -e 选项启用转义序列
echo -e "Line1\nLine2\tTabbed"

# 使用 $'...' 语法（ANSI-C 引用，bash 特有能力）
echo $'Line1\nLine2\tTabbed\u2764'

# printf 跨平台转义（推荐）
printf "Line1\nLine2\tTabbed\n"

# 常见转义序列
# \n  换行   \t  制表符   \\  反斜杠
# \"  双引号  \'  单引号   \a  响铃
# \b  退格   \r  回车    \e  ESC
# \xHH 十六进制字符
```

# 6. 命令替换

```bash
# 旧式反引号（不推荐，嵌套困难）
result=`ls -l`

# 新式 $()（推荐，可嵌套）
result=$(ls -l)
nested=$(echo $(date) and $(whoami))

# 命令替换会去除末尾换行符
output=$(echo -e "line1\nline2\n")
echo "[$output]"    # [line1\nline2]  —— 最后的 \n 被去掉了
```

# 7. 标准输出工具对比

Shell 中输出内容有三个核心命令，功能各有侧重：

```bash
echo "hello"           # 简单输出，自动加换行
printf "Name: %s\n" "Alice"  # 格式化输出，不自动加换行
cat <<EOF              # Here Document，多行输出
line 1
line 2
EOF
```

**echo 的跨平台问题：**

```bash
echo -e "hello\nworld"     # bash/zsh 解释 \n
echo -e "hello\nworld"     # dash 输出 -e hello\nworld（当普通字符串）

# 推荐：用 printf 代替 echo，行为完全一致
printf "hello\nworld\n"
```

**echo vs printf 对照：**

| 场景 | echo 写法 | printf 写法 |
|------|-----------|-------------|
| 简单输出 | `echo "hello"` | `printf "hello\n"` |
| 带转义 | `echo -e "a\tb"` | `printf "a\tb\n"` |
| 不换行 | `echo -n "hello"` | `printf "hello"` |
| 格式化 | 不支持 | `printf "%05d\n" 42` |
| 可移植性 | 差 | 好 |

# 8. 输入读取

```bash
# 基本读取（按空格分割到多个变量）
read first last
# 输入：Alice Smith  → first=Alice, last=Smith

# 读取整行到单个变量（不分割）
read -r line        # -r 禁止反斜杠转义

# 带提示读取
read -p "请输入名字: " name

# 静默读取（密码输入）
read -s -p "密码: " password
echo  # 换行

# 超时读取
read -t 5 -p "5秒内输入: " input || echo "超时"

# 限制字符数（每读 N 个字符就触发）
read -n 1 -p "按任意键继续..." key
```

**始终使用 `read -r` 的充分理由：**

```bash
# 不用 -r：\ 会与下一个字符合并
echo 'hello\nworld' | read line
echo "$line"      # hellonworld （错误！

# 用 -r：保持原样
echo 'hello\nworld' | read -r line
echo "$line"      # hello\nworld （正确）
```

# 9. 大括号扩展（Brace Expansion）

```bash
# 最基础的用法：笛卡尔积
echo file{1,2,3}.txt       # file1.txt file2.txt file3.txt
echo {A,B}{1,2}             # A1 A2 B1 B2

# 范围扩展
echo {1..10}               # 1 2 3 4 5 6 7 8 9 10
echo {01..10}              # 01 02 03 ... 10
echo {a..z}                # a b c ... z
echo {Z..A}                # Z Y X ... A（倒序）

# 步长（bash 4.0+）
echo {1..10..2}            # 1 3 5 7 9
echo {10..1..2}            # 10 8 6 4 2

# 实际应用
mkdir project/{src,test,doc,bin}
cp file.{txt,bak}
mv /path/{oldname,newname}
```

**注意：** 大括号扩展在变量展开**之前**执行，所以不能用于变量：

```bash
start=1
end=3
echo {$start..$end}    # 错误！输出 {1..3}（字面量）
echo $(seq $start $end)  # 正确：用 seq 命令
```

# 10. Here Document 与 Here String

## 10.1 Here Document

```bash
# 基本用法
cat <<EOF
这是第一行
这是第二行
EOF

# 带制表符缩进（<<- 去除前导 Tab）
cat <<-EOF
	缩进的行（前导 Tab 被去除）
	另一行
EOF

# 禁用变量展开（分隔符加引号）
cat <<'EOF'
当前用户是 $USER （不会被展开）
EOF

# 写入文件
cat <<EOF > config.conf
server {
    port 8080
    host localhost
}
EOF
```

## 10.2 Here String

```bash
# 将字符串作为命令的标准输入
grep "pattern" <<< "search in this string"
read var1 var2 <<< "hello world"

# 与管道对比：管道创建子进程，here string 不会
echo "hello world" | (read a; echo $a)  # 不输出（子shell中读取）
read a <<< "hello world"; echo $a        # 输出 hello
```
