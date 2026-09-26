
# 1. 文件描述符

Unix 下每个进程默认打开三个**文件描述符（FD）**：

| 描述符 | 名称 | 符号 | 默认指向 |
|--------|------|------|----------|
| 0 | stdin | 标准输入 | 键盘 |
| 1 | stdout | 标准输出 | 终端 |
| 2 | stderr | 标准错误 | 终端 |

```bash
# 文件描述符的本质
ls -l /proc/$$/fd/
# lrwx------  0 -> /dev/pts/0
# lrwx------  1 -> /dev/pts/0
# lrwx------  2 -> /dev/pts/0
```

**理解：文件描述符就是内核为每个进程维护的一张表，每个 FD 指向一个打开的文件。**

# 2. 输出重定向

```bash
# 覆盖写入文件
echo "hello" > file.txt          # stdout → file.txt
echo "error" > file.txt 2>&1     # stdout + stderr → file.txt
echo "error" &> file.txt         # stdout + stderr → file.txt（bash 4+ 简写）

# 追加写入文件
echo "line2" >> file.txt
echo "error2" &>> file.txt

# 只重定向 stderr
command 2> error.log
command 2>> error.log

# stdout 和 stderr 分别重定向到不同文件
command > stdout.log 2> stderr.log
```

## 2.1 重定向的本质

```bash
# > file 的原理
# 1. 打开/创建文件 file
# 2. 将文件描述符 1（stdout）指向该文件
# 3. 执行命令

# 这意味着 > 重定向在执行命令之前就已经完成了文件截断
seq 5 > nums.txt          # nums.txt 内容为 1\n2\n3\n4\n5\n
cat nums.txt > nums.txt   # nums.txt 被清空！（先截断再读取）
```

**重要陷阱：`cmd file > file` 几乎总是会把文件清空。** 因为 Shell 在启动命令前先打开并截断文件。

# 3. 输入重定向

```bash
# 文件作为输入
command < input.txt

# Here Document
command << 'EOF'
line1
line2
EOF

# Here String
command <<< "single line input"

# 输入与输出结合
sort < unsorted.txt > sorted.txt
```

## 3.1 Here Document 高级用法

```bash
# 禁止变量展开（分隔符加引号）
cat << 'EOF'
$USER 不会被展开
$(date) 也不会
EOF

# 用 Tab 缩进（<<- 去除前导 Tab）
if true; then
    cat <<- EOF
		函数内部的多行文本
	EOF
fi

# 动态生成脚本并执行
shell="bash"
$shell << 'SCRIPT'
echo "运行在子 Shell 中"
echo "PID: $$"
SCRIPT
```

# 4. 文件描述符操作

## 4.1 复制与移动

```bash
# n>&m  —— 将文件描述符 n 指向 m 所指向的位置

# 把 stderr 合并到 stdout（最常见）
command 2>&1

# 把 stdout 复制一份到 stderr
command 1>&2

# 将 stderr 重定向到文件，同时保留 stdout
command 2> error.log 1>&2     # 错误！stdout 和 stderr 都去了 error.log
# 正确做法：先保存 stdout，再操作
command 3>&1 1>stdout.log 2>&1  # 把所有输出保存到 stdout.log

# 交换 stdout 和 stderr
command 3>&1 1>&2 2>&3 3>&-
```

## 4.2 自定义文件描述符

```bash
# 打开文件描述符 3 进行写入
exec 3> output.log
echo "写入 FD3" >&3
exec 3>&-             # 关闭

# 打开文件描述符进行读取
exec 3< input.txt
read -r line <&3
exec 3<&-             # 关闭

# 同时读写（不常用）
exec 3<> /tmp/pipe

# 常见模式：保存/恢复 stdout
exec 3>&1             # 备份 stdout 到 FD3
exec > /tmp/capture   # 重定向 stdout 到文件
echo "这会被写入文件"
exec 1>&3             # 恢复 stdout
exec 3>&-             # 关闭 FD3
echo "这回到终端"
```

# 5. 重定向顺序陷阱

```bash
# 错误：想把 stderr 一起重定向到文件
command 2>&1 > file.log
# 顺序：先把 stderr(2) 指向当前 stdout(1) → 终端
#       再把 stdout(1) 指向 file.log
# 结果：stdout → 文件，stderr → 终端（旧 stdout）

# 正确：先改 stdout，再复制到 stderr
command > file.log 2>&1
# 顺序：先把 stdout(1) 指向 file.log
#       把 stderr(2) 指向当前 stdout(1) → file.log
# 结果：stdout 和 stderr 都 → 文件
```

**记住：重定向从左到右执行。先改目标，再复制。**

# 6. 管道

## 6.1 管道基础

```bash
# 连接命令的 stdout 和下一个命令的 stdin
cmd1 | cmd2 | cmd3

# 只传递 stdout（不传递 stderr）
cmd1 2>/dev/null | cmd2

# 把 stderr 也通过管道传递（bash 4+）
cmd1 |& cmd2          # 等价 cmd1 2>&1 | cmd2
```

## 6.2 管道的退出状态

```bash
# 默认：管道的退出状态是最后一个命令的状态
false | true
echo $?     # 0

# set -o pipefail：任一命令失败则管道失败
set -o pipefail
false | true
echo $?     # 1

# 获取管道中所有命令的状态
false | true | true
echo "${PIPESTATUS[@]}"     # 1 0 0
```

## 6.3 管道与子 Shell

**管道中的每个命令都在独立的子 Shell 中运行（bash 5.1+ lastpipe 除外）。**

```bash
result=0
echo "1 2 3" | while read -r num; do
    result=$((result + num))
done
echo "$result"     # 0！修改发生在子 Shell 中，对父 Shell 不可见
```

**解决方案：**

```bash
# 方案1：进程替换（推荐）
while read -r num; do
    result=$((result + num))
done < <(echo "1 2 3")

# 方案2：Here String
while read -r num; do
    result=$((result + num))
done <<< "$(echo "1 2 3")"

# 方案3：启用 lastpipe（bash 4.2+）
shopt -s lastpipe  # 最后一个管道命令在**当前 Shell** 执行
```

## 6.4 tee 命令

```bash
# 输出到终端的同时保存到文件
command | tee output.log

# 追加模式
command | tee -a output.log

# 同时写入多个文件
command | tee file1.log file2.log

# 与 sudo 配合（> 重定向不受 sudo 影响）
echo "config" | sudo tee /etc/app/config.conf > /dev/null

# 同时捕获到变量（tee 到 stderr 再重定向）
result=$(command | tee /dev/stderr)
```

# 7. 特殊文件

```bash
# /dev/null    —— 数据黑洞，写入的内容被丢弃
command > /dev/null 2>&1    # 完全静默

# /dev/zero    —— 无限提供零字节
dd if=/dev/zero of=file bs=1M count=100

# /dev/random  —— 高质量随机数（可能阻塞）
# /dev/urandom —— 伪随机数（不阻塞）
head -c 16 /dev/urandom | base64

# /dev/fd/N    —— 指向文件描述符 N 的符号链接
diff <(cmd1) <(cmd2)    # 进程替换本质上是在 /dev/fd/ 下创建管道
```

# 8. 进程替换

```bash
# 进程替换：将命令的输出作为文件传给另一个命令

# <(cmd)  —— 输出端：cmd 的 stdout 被放在一个 FIFO 中，可以像文件一样读取
diff <(ls dir1) <(ls dir2)
cat <(date) <(uptime)

# >(cmd)  —— 输入端：写入的文件内容成为 cmd 的 stdin
tar -czf >(ssh remote "tar -xzf -") /path/to/dir
echo "hello" | tee >(grep "h") >(grep "l")

# 底层原理：Shell 创建一个命名管道（FIFO），用 /dev/fd/N 作为路径
echo <(date)
# /dev/fd/63
```

# 9. 命名管道（Named Pipe / FIFO）

```bash
# 创建命名管道
mkfifo /tmp/myfifo

# 终端1：写入（默认阻塞直到有读者）
echo "hello" > /tmp/myfifo    # 阻塞...

# 终端2：读取
cat /tmp/myfifo               # hello

# 实战：并行处理
mkfifo /tmp/queue

producer() {
    for i in {1..100}; do
        echo "item $i" > /tmp/queue
    done
    # 发送结束信号
    echo "EOF" > /tmp/queue
}

consumer() {
    while read -r line && [ "$line" != "EOF" ]; do
        echo "处理: $line"
    done < /tmp/queue
}

producer &
consumer

rm /tmp/queue
```

# 10. 重定向与管道的底层原理

## 10.1 fork+exec 过程中 FD 的继承

```
1. Shell 解析命令行：ls | grep txt > result
   ↓
2. 创建管道：pipe() 返回 fd[0](读端), fd[1](写端)
   ↓
3. fork() 创建子进程1（grep txt）
   ├── close(fd[1])          关闭写端
   ├── dup2(fd[0], STDIN)    将读端复制到 stdin
   ├── open("result", O_WRONLY)  打开输出文件
   ├── dup2(file_fd, STDOUT) 输出定向到 result
   └── exec("grep", "txt")   执行 grep
   ↓
4. fork() 创建子进程2（ls）
   ├── close(fd[0])          关闭读端
   ├── dup2(fd[1], STDOUT)   将写端复制到 stdout
   └── exec("ls")            执行 ls
   ↓
5. Shell 关闭 pipe 两端，wait 两个子进程
```

## 10.2 缓冲模式

| 缓冲类型 | 描述 | 表现形式 |
|----------|------|----------|
| 无缓冲 | 立即输出 | `stderr` 默认 |
| 行缓冲 | 遇到换行输出 | 终端输出的 `stdout` |
| 全缓冲 | 填满缓冲区才输出 | 管道/文件的 `stdout` |

```bash
# 缓冲导致的问题：管道中输出延迟
for i in {1..100}; do
    echo "$i"
    sleep 1
done | while read line; do echo "收到: $line"; done

# 解决：使用 stdbuf 禁用缓冲
stdbuf -oL command     # stdout 行缓冲
stdbuf -o0 command     # stdout 无缓冲
```

# 11. 实战模式

## 11.1 日志同时输出到终端和文件

```bash
exec > >(tee -a app.log) 2>&1
echo "所有后续输出都会同时出现在终端和 app.log"
```

## 11.2 捕获子进程 stdio

```bash
# 捕获命令的 stdout 和 stderr 到不同变量
{
    stdout=$(command 2>&1 1>&3-)
    exitcode=$?
    stderr=$(cat)
    exec 3>&-
} 3>&1 <<< ""

# 更简单的做法
output=$(command 2>&1)    # 全部合并
if [ $? -ne 0 ]; then
    echo "错误: $output"
fi
```

## 11.3 条件重定向

```bash
# 根据变量选择输出目标
if [ "$DEBUG" = "1" ]; then
    exec 3>&1            # 输出到终端
else
    exec 3> /dev/null    # 丢弃
fi
echo "调试信息" >&3
```
