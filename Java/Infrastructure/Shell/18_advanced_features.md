
# 1. 子 Shell 深入

## 1.1 什么是子 Shell

子 Shell 是当前 Shell 进程 fork 出的一个**子进程**，它继承父 Shell 的环境（变量、工作目录、文件描述符等），但拥有独立的**进程空间**。子 Shell 中的修改不会影响父 Shell。

```bash
# 验证子 Shell 的进程 ID
echo "父 Shell PID: $$"
( echo "子 Shell PID: $$"; echo "BASH_SUBSHELL: $BASH_SUBSHELL" )
# 输出：
# 父 Shell PID: 12345
# 子 Shell PID: 12345      ← $$ 显示的是父进程 PID（不变）
# BASH_SUBSHELL: 1         ← 0=当前Shell，1=一层子Shell
```

**核心原则：子 Shell 的 `$$` 始终继承父 Shell 的 PID，这是 POSIX 规范。`$BASHPID`（bash 4.0+）可以获取真实子进程 PID：**

```bash
echo "父: BASHPID=$BASHPID"
( echo "子: BASHPID=$BASHPID" )
```

## 1.2 显式子 Shell

使用 `()` 显式创建子 Shell：

```bash
# 1. 变量隔离
name="outer"
( name="inner"; echo "子 Shell: $name" )   # 输出 inner
echo "父 Shell: $name"                      # 输出 outer

# 2. cd 目录隔离
pwd                    # /home/user
( cd /tmp; pwd )       # /tmp
pwd                    # /home/user（不受影响）

# 3. 后台执行组
( sleep 2; echo "done" ) &

# 4. 组合多条命令的输出
( echo "Header"; cat data.txt; echo "Footer" ) > output.txt
```

## 1.3 隐式子 Shell

以下场景会**自动创建子 Shell**：

| 场景 | 示例 | 说明 |
|------|------|------|
| 管道两侧（部分 Shell） | `cmd1 \| cmd2` | bash 中管道左右均在子 Shell，shopt -s lastpipe 可让最后一个在父 Shell |
| 命令替换 | `$(cmd)` / `` `cmd` `` | 始终在子 Shell 中执行 |
| 进程替换 | `<(cmd)` / `>(cmd)` | 始终在子 Shell 中执行 |
| 后台命令 | `cmd &` | 如果不是作业控制的组成部分则为子 Shell |
| 显式子 Shell | `(cmd1; cmd2)` | 明确使用括号 |
| while/read 管道 | `echo data \| while read ...` | 管道右侧是子 Shell |

```bash
# 经典陷阱：管道中的变量丢失
n=0
seq 3 | while read line; do
    ((n++))
done
echo "n=$n"    # n=0（因为 while 在子 Shell 中，修改未传回）

# 解决方案 1：进程替换
n=0
while read line; do
    ((n++))
done < <(seq 3)
echo "n=$n"    # n=3

# 解决方案 2：here string
while read line; do
    ((n++))
done <<< "$(seq 3)"

# 解决方案 3：shopt -s lastpipe（bash 4.2+）
shopt -s lastpipe
n=0
seq 3 | while read line; do
    ((n++))
done
echo "n=$n"    # n=3（仅在作业控制关闭时生效）
```

## 1.4 子 Shell 的环境继承

```bash
# 变量继承（包括 export 和非 export 变量）
name="张三"
export age=25
( echo "name=$name, age=$age" )   # 都能访问

# 函数继承
hello() { echo "Hello"; }
( hello )                          # 可以调用

# 陷阱：函数定义也可传入，但通过 export -f 更可靠
export -f hello
bash -c 'hello'                    # 跨脚本调用

# 文件描述符继承
exec 3> /tmp/custom_fd.txt
( echo "via fd3" >&3 )
exec 3>&-
```

## 1.5 子 Shell 的性能开销

```bash
# 子 Shell 的创建涉及 fork 系统调用，有性能成本

# 低效：每次循环都 fork 一个子 Shell
time for i in {1..10000}; do
    result=$(echo "$i * 2" | bc)
done

# 高效：在父 Shell 中完成
time for i in {1..10000}; do
    result=$((i * 2))
done

# 测试子 Shell 创建开销
time for i in {1..1000}; do
    ( : )
done
# real 约 0.3~0.5 秒（取决于系统）
```

## 1.6 {} 组命令 —— 子 Shell 的零开销替代

`{}` **组命令**在**当前 Shell** 中执行，不创建子进程，无 fork 开销：

```bash
# 语法要点：{ 后面必须有空格，} 前面必须有分号或换行，最后必须有分号
{ echo "line1"; echo "line2"; } > output.txt

# 与 () 的对比
pwd          # /home/user
{ cd /tmp; pwd; }   # /tmp
pwd          # /tmp（目录变了！因为当前 Shell 中执行）

# {} 组命令 + 重定向
{
    echo "=== 开始 ==="
    date
    echo "=== 内容 ==="
    cat data.txt
    echo "=== 结束 ==="
} > report.txt 2>&1

# 条件分支中批量重定向
if [[ -f "$file" ]]; then
    {
        echo "Processing $file"
        wc -l "$file"
        head -3 "$file"
    } >> process.log
fi

# () 与 {} 对照表
# +--------+----------+----------------+------------+
# | 语法   | 进程     | 副作用是否隔离 | 性能       |
# +--------+----------+----------------+------------+
# | (cmd)  | 子 Shell | 是             | 有 fork 开销 |
# | { cmd;}| 当前 Shell| 否             | 无额外开销   |
# +--------+----------+----------------+------------+
```

# 2. 进程替换

## 2.1 原理

进程替换将命令的输出（或输入）表现为一个**伪文件**，路径形如 `/dev/fd/63` 或 `/proc/self/fd/63`（Linux）或 `/dev/fd/63`（macOS）。

```bash
# 查看进程替换产生的文件路径
echo <(ls)
# 输出: /dev/fd/63（Linux）或 /proc/self/fd/11（旧版 Linux）
```

**两种形式：**

| 形式 | 方向 | 含义 | 使用场景 |
|------|------|------|----------|
| `<(cmd)` | 输出端 | cmd 的 stdout 作为伪文件被**读取** | 作为需要文件参数的命令的输入 |
| `>(cmd)` | 输入端 | 伪文件被**写入**时，数据传给 cmd 的 stdin | 作为需要文件参数的命令的输出 |

## 2.2 `<(cmd)` 输出端进程替换

当命令需要文件名参数而不支持 stdin 时使用：

```bash
# diff 需要文件参数，不支持 stdin
diff <(ls /dir1) <(ls /dir2)

# 等价于手动写法（需要临时文件）：
ls /dir1 > /tmp/dir1.txt
ls /dir2 > /tmp/dir2.txt
diff /tmp/dir1.txt /tmp/dir2.txt
rm /tmp/dir1.txt /tmp/dir2.txt

# 多文件比较
diff3 <(sort file1) <(sort file2) <(sort file3)

# vim diff 模式
vimdiff <(curl -s https://example.com/old.conf) <(cat /etc/local.conf)

# comm 找共同行
comm -12 <(sort users1.txt) <(sort users2.txt)
```

**使用 diff 对比两个命令输出的注意事项：**

```bash
# 如果 diff 支持 - 作为 stdin，也可以这么做
sort file1 | diff - <(sort file2)

# 但进程替换更直观、更通用（不要求命令支持 - 参数）
diff <(grep ERROR log1.txt) <(grep ERROR log2.txt)
```

## 2.3 `>(cmd)` 输入端进程替换

将数据写入进程替换产生的伪文件时，数据被送入 cmd 的 stdin：

```bash
# 将输出同时发送到多个目标（类似 tee，但更灵活）
echo "data" | tee >(md5sum > md5.txt) >(sha256sum > sha256.txt) > /dev/null

# 实时压缩日志
tar -czf >(gpg --symmetric --output backup.tar.gz.gpg) /data

# 边下载边解压（避免临时文件）
curl -s https://example.com/data.tar.gz | tee >(tar -xz -C /extracted) | md5sum

# 实时监控 + 记录
./app 2> >(tee stderr.log >&2)
# 将 stderr 同时写入日志文件和终端

# 多副本输出
echo "payload" | tee >(proc1) >(proc2) >(proc3) > /dev/null
```

## 2.4 进程替换与管道的区别

| 特性 | 管道 `\|` | 进程替换 `<()` |
|------|-----------|----------------|
| 数据流向 | stdout → stdin | 伪文件读取/写入 |
| 适用命令 | 支持 stdin 的命令 | 需要文件名参数的命令 |
| 并发关系 | 串行连接 | 并行独立执行 |
| 上下文 | 可能存在子 Shell 陷阱 | 命令在子 Shell 执行后返回文件描述符 |
| 多输入 | 需要多次管道或临时文件 | 自然支持多路输入 |

```bash
# 管道：一个命令的输出是下一个命令的输入（串行）
cmd1 | cmd2 | cmd3

# 进程替换：多个源同时提供给一个命令（并行读取）
diff <(cmd1) <(cmd2)

# 组合使用：管道 + 进程替换
curl -s https://api.example.com/data.json \
  | tee >(jq '.users' > users.json) \
  | jq '.metadata' > metadata.json
```

## 2.5 进程替换的内部机制

```bash
# 以 <(cmd) 为例，bash 内部执行流程：
# 1. 创建匿名管道（pipe 系统调用）
# 2. fork 子进程
# 3. 子进程将 stdout 重定向到管道写入端
# 4. 子进程 exec cmd
# 5. 用 /dev/fd/N 替换命令行中的 <(cmd)
# 6. 管道读取端作为参数传给主命令

# >(cmd) 的流程相反：
# 1. 创建匿名管道
# 2. fork 子进程
# 3. 子进程将 stdin 重定向到管道读取端
# 4. 子进程 exec cmd
# 5. 用 /dev/fd/N 替换 >(cmd)
# 6. 主命令向该文件描述符写入时，数据流入 cmd
```

# 3. 协进程 coproc

## 3.1 协进程概念

协进程（coproc）允许在后台以**异步方式**运行一个命令，并通过**双向管道**与其通信。bash 4.0 版本引入。

```bash
# 基本语法（bash 4.0+）
coproc NAME { cmd; }
coproc cmd

# 默认协进程名称：COPROC
# 通信方式：通过数组 COPROC[0]（读）和 COPROC[1]（写）
```

## 3.2 基本用法

```bash
# 启动协进程：以 bc 计算器为例
coproc BC { bc -q; }
# BC 数组：BC[0]=读取端（从bc的stdout读）、BC[1]=写入端（向bc的stdin写）

# 向 bc 发送计算
echo "123 * 456" >&${BC[1]}

# 从 bc 读取结果
read -u ${BC[0]} result
echo "结果: $result"

# 继续发送更多计算
echo "scale=10; 355 / 113" >&${BC[1]}
read -u ${BC[0]} result
echo "圆周率近似值: $result"

# 关闭写入端（发送 EOF）
eval "exec ${BC[1]}>&-"

# 等待协进程结束
wait $BC_PID
```

## 3.3 简单协进程语法

```bash
# 不使用名称，使用默认 COPROC 数组
coproc tr a-z A-Z

# 写入数据
echo "hello world" >&${COPROC[1]}
# 关闭写入端
exec {COPROC[1]}>&-

# 读取结果
cat <&${COPROC[0]}
# 输出: HELLO WORLD

wait $COPROC_PID
```

## 3.4 协进程实战

```bash
# 1. 与 Python 交互式通信
coproc PY { python3 -u; }

send_cmd() { echo "$1" >&${PY[1]}; }
read_result() { read -t 0.1 -u ${PY[0]} line; echo "$line"; }

send_cmd 'import json'
send_cmd 'print(json.dumps({"name": "张三", "age": 30}))'
read_result

# 2. 与 jq 保持长连接
coproc JQ { jq -c '.'; }
echo '{"name": "test", "value": 42}' >&${JQ[1]}
read -u ${JQ[0]} line
echo "$line"

# 3. 流式日志处理
coproc LOGPROC {
    while read -r line; do
        echo "$(date '+%H:%M:%S') | $line" >&2
    done
}
# 通过协进程实时加时间戳
```

## 3.5 协进程与命名管道的对比

| 特性 | coproc | 命名管道 (mkfifo) | 匿名管道 `\|` |
|------|--------|--------------------|--------------|
| 通信方向 | 双向 | 单向（需两个） | 单向 |
| 文件系统 | 不创建文件 | 创建 FIFO 文件 | 不创建文件 |
| 清理 | 自动（进程结束即清理） | 需要手动删除 FIFO 文件 | 自动 |
| 语法复杂度 | 中等 | 高 | 低 |
| bash 版本 | 4.0+ | 任何版本 | 任何版本 |
| 适用场景 | 持久交互 | 独立进程间通信 | 一次性数据流 |

```bash
# 命名管道实现与协进程相同的效果
mkfifo /tmp/in_pipe /tmp/out_pipe
bc -q < /tmp/in_pipe > /tmp/out_pipe &
echo "123 * 456" > /tmp/in_pipe
read result < /tmp/out_pipe
echo "$result"
rm /tmp/in_pipe /tmp/out_pipe
# 显然 coproc 更简洁
```

# 4. 文件描述符高级操作

## 4.1 自定义文件描述符

```bash
# 打开文件并分配自定义 FD
exec 3> /tmp/custom_out.txt       # 以写入方式打开，分配 FD 3
echo "写入 FD 3" >&3
exec 3>&-                          # 关闭 FD 3

# 以读取方式打开
exec 3< /etc/hosts
read -u 3 first_line              # bash 内置 read -u 读取指定 FD
echo "$first_line"
exec 3<&-

# 以读写方式打开
exec 3<> /tmp/readwrite.dat
echo "write to fd3" >&3
# 注意：需要 seek 回文件开头才能读取
exec 3<&-
```

## 4.2 FD 的复制与恢复

```bash
# 保存 stdout 到 FD 5，临时重定向 stdout 到文件
exec 5>&1                         # 保存 stdout 到 FD 5
exec > /tmp/output.txt            # 重定向 stdout 到文件
echo "这写入文件"
exec 1>&5                         # 恢复 stdout
exec 5>&-                         # 关闭 FD 5
echo "这回到终端"

# 保存 stderr 并临时关闭
exec 6>&2                         # 保存 stderr
exec 2> /tmp/errors.txt
ls /nonexistent                   # 错误写入文件
exec 2>&6                         # 恢复 stderr
exec 6>&-                         # 关闭

# 保存/恢复的通用模式
save_stdout() { exec 8>&1; }
restore_stdout() { exec 1>&8; exec 8>&-; }
redirect_stdout() { save_stdout; exec > "$1"; }
```

## 4.3 `{fd}` 动态分配文件描述符（bash 4.1+）

传统方式需要手动选择未被占用的 FD（通常用 3~9），bash 4.1+ 支持**自动分配**：

```bash
# 语法：exec {变量名}> 文件
# bash 自动分配一个 ≥10 的 FD，并将 FD 编号存入变量

# 自动分配 FD 打开文件
exec {logfd}> /tmp/app.log
echo "FD 编号: $logfd"            # 例如: 10
echo "记录日志" >&${logfd}
exec {logfd}>&-                   # 关闭

# 自动分配多个 FD
exec {in_fd}< /etc/hosts
exec {out_fd}> /tmp/out.txt
echo "in=$in_fd, out=$out_fd"     # 例如: in=10, out=11

# 函数中使用，避免 FD 冲突
write_log() {
    local fd
    exec {fd}>> /var/log/myapp.log
    echo "[$(date)] $*" >&${fd}
    exec {fd}>&-
}
write_log "服务启动"

# 在脚本中批量打开/关闭 FD
declare -A fds
files=(/var/log/syslog /var/log/auth.log)
for f in "${files[@]}"; do
    exec {fd}< "$f"
    fds[$f]=$fd
done
# 使用 ...
for f in "${!fds[@]}"; do
    exec {fds[$f]}<&-
done
```

## 4.4 重定向顺序的精妙陷阱

```bash
# 陷阱 1：2>&1 必须先于 > file
command > file 2>&1      # 正确：stdout→file，stderr→&1(=file)
command 2>&1 > file      # 错误：stderr→终端(stdout当时还是终端)，stdout→file

# 逐步分析：
# 2>&1 > file：
#   步骤1: 2>&1 → stderr 指向 stdout 当前指向的位置（终端）
#   步骤2: > file → stdout 指向 file，但 stderr 仍指向终端

# > file 2>&1：
#   步骤1: > file → stdout 指向 file
#   步骤2: 2>&1 → stderr 指向 stdout 当前指向的位置（file）

# 互换 stdout 和 stderr 的正确写法
command 3>&1 1>&2 2>&3 3>&-
# 步骤：
# 3>&1: FD3 → stdout原目标
# 1>&2: stdout → stderr原目标
# 2>&3: stderr → FD3（stdout原目标）
# 3>&-: 关闭FD3
```

# 5. exec 命令全解

## 5.1 exec 的两种用途

`exec` 在 Shell 中有两种完全不同的用途：

| 用途 | 语法 | 效果 |
|------|------|------|
| 替换当前进程 | `exec COMMAND [ARGS]` | 用 COMMAND 替换当前 Shell，不返回 |
| 重定向操作 | `exec [FD]> file` | 在当前 Shell 中永久修改文件描述符 |

## 5.2 exec 替换当前进程

```bash
# exec 执行命令时，用该命令的进程**替换**当前 Shell 进程
# 不会 fork 新进程，不返回到原 Shell

# 脚本示例：exec_wrapper.sh
#!/bin/bash
echo "启动服务：PID $$"
export APP_ENV=production
exec /usr/local/bin/myapp --port 8080
echo "这行永远不会执行"

# 为什么使用 exec 替换进程：
# 1. 节省进程资源（一个进程而非两个）
# 2. 信号直接传递给服务进程（不会被 Shell 拦截）
# 3. 进程监控清晰（ps 看到的直接是目标进程）

# 在 docker 中至关重要
#!/bin/bash
# 正确的 Docker entrypoint
exec "$@"
# 不正确的写法：
# "$@"         ← 会多一个 bash 进程作为 PID 1
```

## 5.3 exec 重定向永久生效

```bash
# exec 重定向在当前 Shell 中永久生效，直到再次修改或 Shell 退出

# 将所有输出重定向到文件（整个脚本生效）
exec > /tmp/script_output.log 2>&1
echo "这写入日志文件"
echo "这也写入日志文件"
date
# 此后的所有输出都进入日志文件

# exec 读取文件替换 stdin
exec < /tmp/input_data.txt
while read line; do
    echo "处理: $line"
done

# 关闭文件描述符
exec 3>&-          # 关闭 FD 3

# 重定向到多个文件（不能被 tee 完全替代的场景）
exec 3>&1 4>&2    # 备份
exec > /var/log/app.log 2>/var/log/app_error.log
# ... 脚本逻辑 ...
exec 1>&3 2>&4    # 恢复
exec 3>&- 4>&-    # 关闭备份

# exec 重定向的作用域是当前 Shell，子 Shell 不会继承永久重定向
exec > /tmp/parent.log
echo "父 Shell - 写入 parent.log"
( echo "子 Shell - 写入终端"; )       # 子 Shell 恢复默认
bash -c 'echo "子进程 - 写入终端"'
```

## 5.4 exec 重定向的常见使用模式

```bash
# 模式1：脚本级别日志重定向
#!/bin/bash
LOGFILE="/var/log/$(basename "$0").log"
exec > >(tee -a "$LOGFILE") 2>&1
echo "脚本开始"
# 所有输出同时到终端和日志

# 模式2：临时修改，然后恢复
exec 7>&1                           # 保存 stdout
exec 8>&2                           # 保存 stderr
exec > /tmp/capture.log 2>&1        # 重定向
perform_diagnostics                 # 运行诊断
exec 1>&7 2>&8                      # 恢复
exec 7>&- 8>&-                      # 清理

# 模式3：锁定文件描述符使其不被子进程继承
# 默认情况下，子进程会继承父进程的文件描述符
exec 3> lockfile
flock -x 3
# ... 临界区 ...
flock -u 3
exec 3>&-
```

# 6. eval 的使用与风险

## 6.1 eval 的原理 —— 二次解析

`eval` 将其参数拼接成一个字符串，然后交给 Shell **重新进行完整的命令行解析**（包括变量展开、通配符展开、引号处理等）。

```bash
# eval 的执行流程
# 第1次解析（常规）：Shell 读取命令行，执行变量展开/命令替换
# eval 被识别为内置命令
# 第2次解析（eval 触发）：eval 将其所有参数拼接，作为新命令重新解析执行

# 简单示例
cmd="ls -la"
$cmd              # 可能出错（含空格）
eval "$cmd"       # eval 将 "ls -la" 重新解析为命令+参数

# 逐步骤分析
var='$USER'
echo "$var"       # 输出: $USER（只展开一次）
eval echo "$var"  # 第1次: eval echo '$USER' → 第2次: echo $USER → 输出实际用户名
```

## 6.2 eval 的实用场景

```bash
# 1. 间接变量引用（bash 4.3- 的替代方案）
name="Zhang San"
varname="name"
eval "echo \$$varname"    # 输出: Zhang San
# 解析过程：eval "echo $name" → echo Zhang San

# 2. 动态构建命令及参数
criteria=("-name" "*.log" "-mtime" "+7")
eval "find /var/log ${criteria[*]}"  # 需谨慎

# 3. 动态变量赋值
for i in name age city; do
    eval "${i}_upper=\${${i}^^}"   # 创建 NAME_UPPER, AGE_UPPER 等
done

# 4. 生成并执行代码
generate_cmd() {
    local op="$1" a="$2" b="$3"
    case "$op" in
        add) echo "$((a + b))" ;;
        mul) echo "$((a * b))" ;;
    esac
}
eval_result=$(eval "$(generate_cmd add 3 5)")
# 不推荐：可以用函数直接计算

# 5. 读取配置文件中的变量定义
# config.txt: NAME="张三"
# config.txt: AGE=30
while IFS='=' read -r key value; do
    eval "$key=$value"
done < config.txt
```

## 6.3 eval 的风险与安全

```bash
# 风险 1：命令注入
user_input='$(rm -rf /)'
eval "echo $user_input"       # 极其危险！

# 风险 2：未预期的通配符展开
pattern='*'
eval "ls $pattern"            # 展开为当前目录所有文件

# 风险 3：变量内容被当作 Shell 代码执行
cmd='echo "hello"; date; ls /etc/passwd'
eval "$cmd"                   # 会执行所有三条命令

# 安全准则：
# 1. 永远不要对未经过滤的用户输入使用 eval
# 2. 能不用就不用，优先考虑其他方案
# 3. 如果必须使用，变量放在单引号内经过 eval 展开

# 替代 eval 的安全方案：
# 间接变量 → declare -n（bash 4.3+）或 ${!var}
name="张三"
varname="name"
echo "${!varname}"            # 安全的间接引用
```

## 6.4 eval 底层机制

```bash
# eval 本质上是请求 bash parser 对字符串进行"二次扫描"
# 通过源码可以看到 bash/builtins/eval.def 的实现：

# 简化逻辑：
# eval_string = concat_all_args(argv)
# parse_and_execute(eval_string, ...)

# 这意味着 eval 会重新经历：
# 1. 单词分割（Word Splitting）
# 2. 路径名展开（Pathname Expansion / Globbing）
# 3. 变量展开（如果未在第一次解析中被保护）
# 4. 命令替换（如果未在第一次解析中被保护）
# 5. 等等

# 这就是为什么 eval 既能解决问题又充满危险
```

# 7. shopt 选项详解

## 7.1 shopt 概述

`shopt`（Shell Options）是 bash 特有的选项控制命令，用于修改 Shell 的各种行为。与 `set` 不同，`shopt` 控制的是 **bash 扩展行为**而非 POSIX 标准行为。

```bash
# 基本语法
shopt                  # 列出所有选项及当前状态
shopt -s optname       # 开启（set）选项
shopt -u optname       # 关闭（unset）选项
shopt -p optname       # 打印该选项的设置命令（可复现）
shopt -q optname       # 静默查询（用于 if 判断）
```

## 7.2 重要选项详解

```bash
# 1. globstar —— 递归通配符 **
shopt -s globstar
ls **/*.md              # 递归匹配所有层级的 .md 文件
echo **/*.log           # 等价于 find . -name "*.log"

# 2. nullglob —— 无匹配时展开为空
shopt -u nullglob       # 默认：无匹配保留原样
echo *.xyz              # 输出: *.xyz（字面量）
shopt -s nullglob
echo *.xyz              # 输出: （空行，因为无匹配）

# 3. failglob —— 无匹配时报错
shopt -s failglob
echo *.xyz              # bash: no match: *.xyz

# 4. extglob —— 扩展通配符
shopt -s extglob
ls !(backup|temp)       # 排除 backup 和 temp
ls @(*.sh|*.py)         # 只匹配 .sh 或 .py 文件
ls +([0-9])             # 纯数字文件名
ls *([a-z])             # 纯小写字母文件名
ls ?(config|settings)   # config 或 settings（0次或1次）

# 5. nocaseglob —— 大小写不敏感通配
shopt -s nocaseglob
ls *.JPG                 # 也匹配 .jpg .Jpg .jPg 等

# 6. dotglob —— 通配符包含隐藏文件
shopt -s dotglob
echo *                   # 包含 .gitignore .bashrc 等

# 7. histappend —— 历史记录追加而非覆盖
shopt -s histappend     # 多个 Shell 共享历史（而非最后一个覆盖）

# 8. cdspell —— cd 自动纠正
shopt -s cdspell
cd /etc/paswrd          # 自动纠正为 /etc/passwd

# 9. checkwinsize —— 窗口大小变化时更新 LINES/COLUMNS
shopt -s checkwinsize

# 10. autocd —— 目录名即 cd
shopt -s autocd
/etc                     # 等价于 cd /etc
```

## 7.3 脚本中常用的 shopt 组合

```bash
# 严格模式组合
shopt -s failglob          # 通配失败报错
shopt -s nullglob          # 空匹配不保留字面量
# 注意：failglob 和 nullglob 不能同时开启

# 推荐脚本头部
#!/bin/bash
shopt -s extglob           # 扩展通配符
shopt -s globstar          # 递归通配符
shopt -s shift_verbose     # shift 超过范围时报错

# 条件设置
if shopt -q globstar; then
    echo "** 通配符可用"
fi
```

## 7.4 shopt 完整列表速查

```bash
# 获取当前 bash 版本所有选项
shopt | wc -l              # 约 50+ 个选项

# 按类别分类：
# ---- 通配符相关 ----
# globstar, nullglob, failglob, extglob, nocaseglob,
# dotglob, nocaseglob, force_fignore, gnu_errfmt

# ---- 历史相关 ----
# histappend, histreedit, histverify, lithist,
# cmdhist, histexpand

# ---- cd相关 ----
# autocd, cdable_vars, cdspell, direxpand

# ---- 兼容性 ----
# compat31, compat32, compat40, compat41, compat42,
# compat43, compat44, xpg_echo

# ---- 作业控制 ----
# checkjobs, checkwinsize, huponexit

# ---- 提示符 ----
# promptvars, extquote

# ---- 安全 ----
# restricted_shell, progcomp, progcomp_alias

# ---- 其他 ----
# lastpipe, mailwarn, no_empty_cmd_completion,
# expand_aliases, extdebug, login_shell, sourcepath
```

# 8. 大括号展开高级

## 8.1 大括号展开的原理

大括号展开（Brace Expansion）是 bash 的**纯文本生成机制**，在任何其他展开之前执行。它不是通配符——它不检查文件系统，只做**字符串生成**。

```bash
# 展开优先级：大括号展开 > 波浪号展开 > 变量展开 > 命令替换 > 通配符展开
echo file.{txt,md,log}
# 第一步：大括号展开 → file.txt file.md file.log
# （不检查这些文件是否存在）
```

## 8.2 序列与步长

```bash
# 数字序列（可含步长）
echo {1..10}               # 1 2 3 4 5 6 7 8 9 10
echo {01..10}              # 01 02 03 04 05 06 07 08 09 10（前导零）
echo {1..10..2}            # 1 3 5 7 9（步长为2）—— bash 4.0+
echo {10..1..2}            # 10 8 6 4 2（递减，步长为2）
echo {001..010..2}         # 001 003 005 007 009

# 字母序列（步长 bash 4.0+）
echo {a..z}                # a b c d ... z
echo {A..Z..2}             # A C E G I K M O Q S U W Y
echo {z..a..3}             # z w t q n k h e b

# 组合前缀/后缀
echo img_{1..5}.png       # img_1.png img_2.png img_3.png img_4.png img_5.png
echo /var/log/app_log.{1..7}.{gz,bz2}  # 多个组合
```

## 8.3 组合展开（笛卡尔积）

```bash
# 多个大括号同时出现时产生笛卡尔积
echo {a,b}{1,2,3}
# 输出: a1 a2 a3 b1 b2 b3

echo {app,db,web}.{dev,staging,prod}.example.com
# 输出:
# app.dev.example.com app.staging.example.com app.prod.example.com
# db.dev.example.com db.staging.example.com db.prod.example.com
# web.dev.example.com web.staging.example.com web.prod.example.com

# 三维组合
echo {A,B}_{0..2}_{x,y}
# A_0_x A_0_y A_1_x A_1_y A_2_x A_2_y
# B_0_x B_0_y B_1_x B_1_y B_2_x B_2_y

# 生成测试目录结构
mkdir -p project/{src/{main,test}/{java,resources},docs,target}
# 结果：
# project/src/main/java  project/src/main/resources
# project/src/test/java  project/src/test/resources
# project/docs           project/target
```

## 8.4 大括号展开的工程应用

```bash
# 1. 批量备份/重命名文件
cp config.yaml{,.bak}               # cp config.yaml config.yaml.bak
mv /etc/nginx/sites-available/default{,.$(date +%Y%m%d)}

# 2. 批量创建文件
touch report_{jan,feb,mar,apr}_2024.csv

# 3. 生成测试数据
echo "host: server{01..10}.example.com"

# 4. diff 多个版本
diff file_v{1,2}.txt

# 5. for 循环生成
for month in {01..12}; do
    echo "处理 ${month} 月份数据"
done

# 6. 变量中的大括号展开（间接实现）
prefix="server"
suffixes="{01..05}"          # 括号被当作字面量，不会展开
# eval "echo $prefix$suffixes"  # 可以用 eval 但危险
# 更好的方式：用 seq 或数组
for i in $(seq -w 1 5); do
    echo "${prefix}${i}"
done

# 注意：大括号展开不能与变量混合使用
start=1; end=5
echo {$start..$end}          # 输出: {1..5}（不展开！）
# 变量中的括号被当作字面量，需要用 seq 或 eval
seq $start $end
```

## 8.5 大括号展开陷阱与局限性

```bash
# 1. 不能包含变量（在变量展开之前执行）
a=1; b=5
echo {$a..$b}      # 字面量输出 {1..5}
# 解决方法：
eval echo {$a..$b} # 输出 1 2 3 4 5
# 或
seq $a $b

# 2. 不能与通配符混合
echo file.{txt,md}  # 正确：大括号展开
echo file.*.txt    # 通配符展开

# 3. 不能在双引号中使用
echo "file.{a,b}"   # 输出: file.{a,b}（不展开）
echo file.{a,b}     # 输出: file.a file.b

# 4. 空格敏感
echo {a, b, c}      # 输出: a b c（空格被当作数据）
echo {a,b,c}        # 输出: a b c
```

# 9. 内部变量

## 9.1 RANDOM —— 随机数生成

```bash
# $RANDOM 返回 0~32767 之间的随机整数（15位）
echo $RANDOM        # 例如: 25127
echo $RANDOM        # 例如: 9842

# 生成指定范围内的随机数
echo $(( RANDOM % 100 ))              # 0~99
echo $(( RANDOM % 100 + 1 ))          # 1~100
echo $(( RANDOM % 900 + 100 ))        # 100~999（三位数）
echo $(( RANDOM * (max - min + 1) / 32768 + min ))  # [min, max]

# 设置随机种子（使序列可重现）
RANDOM=42
echo $RANDOM                          # 始终返回同一序列的开始值

# 随机选取数组元素
colors=(红 橙 黄 绿 蓝 靛 紫)
echo "${colors[RANDOM % ${#colors[@]}]}"

# 生成随机字符串
tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 16
# 或纯 bash：
printf "%x%x%x%x\n" $RANDOM $RANDOM $RANDOM $RANDOM
```

## 9.2 SECONDS —— 脚本运行时间

```bash
# $SECONDS 返回 Shell 自启动以来经过的秒数
echo "脚本已运行 $SECONDS 秒"

# 重置为 0 开始计时
SECONDS=0
compute_heavy_task
echo "计算耗时: $SECONDS 秒"

# 计时函数
timer_start() { SECONDS=0; }
timer_elapsed() { echo "耗时: $SECONDS 秒"; }

timer_start
sleep 0.5
timer_elapsed    # 注意：SECONDS 是整数，短于1秒显示0

# 多个计时点
SECONDS=0
step1; echo "步骤1: $SECONDS 秒"
step2; echo "步骤1+2: $SECONDS 秒"

# 注意：SECONDS 在不同的子 Shell 中各自独立
( SECONDS=0; sleep 2; echo "子Shell: $SECONDS" )
echo "父Shell: $SECONDS"    # 父Shell的SECONDS不受子Shell影响
```

## 9.3 EPOCHSECONDS 与 EPOCHREALTIME（bash 5.0+）

```bash
# EPOCHSECONDS: Unix 时间戳（秒），整数
echo "当前时间戳: $EPOCHSECONDS"

# EPOCHREALTIME: Unix 时间戳（秒.微秒），浮点数
echo "高精度时间戳: $EPOCHREALTIME"

# 高精度计时（bash 5.0+）
start=$EPOCHREALTIME
compute_task
end=$EPOCHREALTIME
elapsed=$(echo "$end - $start" | bc)
echo "精确耗时: ${elapsed}s"
```

## 9.4 BASH_SOURCE、FUNCNAME、LINENO

```bash
# $BASH_SOURCE: 当前源文件的路径（数组，用于追踪调用栈）
# $FUNCNAME: 当前函数名（数组，索引 0 是当前函数）
# $LINENO: 当前行号

# 增强版日志函数
log() {
    local level="$1"; shift
    local func="${FUNCNAME[1]:-main}"
    local file="${BASH_SOURCE[1]:-$0}"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] [$file:$LINENO:$func] $*"
}
log INFO "开始处理数据"

# 打印调用栈
print_stack_trace() {
    local i=0
    echo "=== 调用栈 ==="
    while caller $i 2>/dev/null; do
        ((i++))
    done
}

# 使用 FUNCNAME 判断调用链
is_called_from() {
    local target_func="$1"
    for f in "${FUNCNAME[@]}"; do
        [[ "$f" == "$target_func" ]] && return 0
    done
    return 1
}

# 获取脚本自身所在的目录（最可靠方法）
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# BASH_SOURCE[0] 即使在 source 的场景下也指向原始脚本
```

## 9.5 PIPESTATUS —— 管道状态数组

```bash
# $PIPESTATUS 是数组，记录管道中每个命令的退出码
# 索引 0 是第一个命令，以此类推

true | false | true
echo "${PIPESTATUS[@]}"     # 输出: 0 1 0

# 检查管道中是否有任何失败
check_pipe() {
    local status
    for status in "${PIPESTATUS[@]}"; do
        (( status )) && return 1
    done
    return 0
}

grep "ERROR" app.log | wc -l
if ! check_pipe; then
    echo "管道中有命令失败"
fi

# 常见模式：检查管道中是否 grep 找到了内容
# 但不能直接用 $?，因为管道只返回最后一个命令的退出码
some_command | grep "pattern"
if [[ ${PIPESTATUS[0]} -ne 0 || ${PIPESTATUS[1]} -ne 0 ]]; then
    echo "命令失败或未找到匹配"
fi

# 注意：任何新命令执行都会覆盖 PIPESTATUS
# 需要在管道执行后立即读取
some_command | filter | formatter
pipe_status=("${PIPESTATUS[@]}")    # 立即保存

# 更安全的方式
output=$(some_command)
if [[ $? -eq 0 ]]; then
    echo "$output" | grep "pattern"
fi
```

## 9.6 其他实用内部变量速查

```bash
# BASH_SUBSHELL: 当前子 Shell 嵌套深度（0=当前Shell）
( echo "$BASH_SUBSHELL"; ( echo "$BASH_SUBSHELL" ) )
# 输出: 1  2

# BASHPID: 当前 bash 进程的真实 PID（$$ 是父 Shell 的 PID）
echo "$$ vs $BASHPID"

# BASH_COMMAND: 当前正在执行的命令（DEBUG trap 中使用）
trap 'echo "执行: $BASH_COMMAND"' DEBUG

# BASH_VERSION: 完整的版本字符串
echo "$BASH_VERSION"
# MAJOR=${BASH_VERSINFO[0]}, MINOR=${BASH_VERSINFO[1]}

# EUID / UID: 有效用户ID和实际用户ID
[[ $EUID -eq 0 ]] && echo "以 root 运行"

# BASH_REMATCH: 正则匹配结果数组
[[ "abc123" =~ ([a-z]+)([0-9]+) ]]
echo "${BASH_REMATCH[1]}"  # abc
echo "${BASH_REMATCH[2]}"  # 123

# GROUPS: 当前用户所属的组ID数组
echo "${GROUPS[@]}"

# HOSTNAME: 主机名
echo "$HOSTNAME"

# OSTYPE: 操作系统类型
echo "$OSTYPE"            # linux-gnu, darwin (macOS), msys (Git Bash)

# MACHTYPE: 机器类型
echo "$MACHTYPE"          # x86_64-pc-linux-gnu

# REPLY: read 无变量名时的默认接收变量
read -p "输入: "
echo "$REPLY"

# TMOUT: 超时秒数，0 表示不超时
TMOUT=5; read -p "5秒内输入: "
```

# 10. 关联数组实战

## 10.1 关联数组基础

```bash
# 声明关联数组（bash 4.0+）
declare -A config

# 赋值
config["host"]="localhost"
config["port"]=3306
config["user"]="root"
config["password"]="secret"

# 访问
echo "主机: ${config[host]}"
echo "端口: ${config[port]}"

# 遍历所有键
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# 遍历所有值
for value in "${config[@]}"; do
    echo "$value"
done

# 检查键是否存在
[[ -v config[host] ]] && echo "host 键存在"
[[ -n "${config[missing]+exists}" ]] && echo "键存在"   # 兼容写法

# 删除键
unset "config[password]"

# 获取大小
echo "配置项数量: ${#config[@]}"
```

## 10.2 多维数据模拟

```bash
# bash 不原生支持多维数组，通过键名约定模拟

# 方案1：复合键名
declare -A matrix

# 设置值
matrix["0,0"]=1; matrix["0,1"]=2; matrix["0,2"]=3
matrix["1,0"]=4; matrix["1,1"]=5; matrix["1,2"]=6
matrix["2,0"]=7; matrix["2,1"]=8; matrix["2,2"]=9

# 读取
get_value() {
    local row="$1" col="$2"
    echo "${matrix["${row},${col}"]:-0}"
}

# 打印矩阵
print_matrix() {
    local rows=3 cols=3
    for ((i=0; i<rows; i++)); do
        for ((j=0; j<cols; j++)); do
            printf "%3s" "$(get_value $i $j)"
        done
        echo
    done
}

# 方案2：用数组嵌套（一维数组存二维数组的引用）
declare -A db
db["users"]="users_arr"
db["orders"]="orders_arr"

declare -A users_arr
users_arr["admin"]="hashed_password_1"
users_arr["guest"]="hashed_password_2"

declare -A orders_arr
orders_arr["1001"]="pending"
orders_arr["1002"]="shipped"

# 间接访问：declare -n 获取嵌套引用
access_db() {
    local table="$1" key="$2"
    local -n t="${db[$table]}"      # nameref
    echo "${t[$key]}"
}

access_db "users" "admin"          # 输出: hashed_password_1
access_db "orders" "1001"          # 输出: pending
```

## 10.3 关联数组构建 JSON

```bash
declare -A json_data
json_data["name"]="张三"
json_data["age"]="30"
json_data["city"]="北京"
json_data["active"]="true"

# 将关联数组序列化为 JSON
assoc_to_json() {
    local -n data="$1"         # nameref 引用关联数组
    local json="{"
    local first=true

    for key in "${!data[@]}"; do
        $first || json+=","
        first=false

        local value="${data[$key]}"
        # 判断值类型
        if [[ "$value" =~ ^[0-9]+(\.[0-9]+)?$ ]] || [[ "$value" == "true" || "$value" == "false" ]]; then
            json+="\"$key\":$value"
        else
            # 字符串值需要转义
            value="${value//\\/\\\\}"       # 转义反斜杠
            value="${value//\"/\\\"}"       # 转义双引号
            json+="\"$key\":\"$value\""
        fi
    done
    json+="}"
    echo "$json"
}

assoc_to_json json_data
# 输出: {"name":"张三","age":30,"city":"北京","active":true}

# 嵌套 JSON（关联数组内嵌关联数组）
declare -A user user_address user_profile
user["id"]="1001"
user["name"]="张三"

user_address["street"]="长安街100号"
user_address["city"]="北京"

user_profile["age"]="30"
user_profile["email"]="zhangsan@example.com"

# 将嵌套结构扁平化
to_nested_json() {
    local -n addr="$1" prof="$2"
    local addr_json=$(assoc_to_json addr)
    local prof_json=$(assoc_to_json prof)
    echo "{\"address\":$addr_json,\"profile\":$prof_json}"
}
```

## 10.4 关联数组实用场景

```bash
# 场景1：配置文件解析
declare -A app_config
parse_ini() {
    local file="$1"
    while IFS='=' read -r key value; do
        [[ -z "$key" || "$key" =~ ^[[:space:]]*# ]] && continue
        key="${key## }"; key="${key%% }"        # trim
        value="${value## }"; value="${value%% }"
        app_config["$key"]="$value"
    done < "$file"
}

# 场景2：计数器/统计
declare -A word_count
while read -r line; do
    for word in $line; do
        ((word_count["$word"]++))
    done
done < document.txt

# 排序输出高频词
for word in "${!word_count[@]}"; do
    echo "${word_count[$word]} $word"
done | sort -rn | head -10

# 场景3：去重
declare -A seen
unique_items=()
for item in "${items[@]}"; do
    [[ -n "${seen[$item]}" ]] && continue
    seen[$item]=1
    unique_items+=("$item")
done

# 场景4：缓存/记忆化
declare -A fib_cache
fib_cache[0]=0; fib_cache[1]=1
fib() {
    local n="$1"
    [[ -n "${fib_cache[$n]}" ]] && { echo "${fib_cache[$n]}"; return; }
    fib_cache[$n]=$(($(fib $((n-1))) + $(fib $((n-2)))))
    echo "${fib_cache[$n]}"
}
```

# 11. 间接引用与 nameref

## 11.1 间接引用 ${!var}

bash 中的 `!` 间接引用操作符用于获取变量名存储在另一个变量中的值：

```bash
# 基础间接引用
name="张三"
varname="name"
echo "${!varname}"           # 输出: 张三

# 等价于两层展开：
# ${!varname} → 先取 varname 的值 "name" → 再取 name 的值 "张三"

# 多层嵌套不直接支持
first="second"
second="third"
third="最终值"
# ${!${!first}} 不行，需要用 nameref

# 数组元素的间接引用
colors=(红 绿 蓝)
idx="colors[1]"
echo "${!idx}"               # 输出: 绿

# 关联数组的间接引用
declare -A user
user["name"]="张三"
key="user[name]"
echo "${!key}"               # 输出: 张三
```

## 11.2 declare -n / nameref（bash 4.3+）

`declare -n` 或 `local -n` 创建**别名引用**——对该变量的任何操作都等同于对目标变量的操作：

```bash
# 基本用法
name="张三"
declare -n ref=name
echo "$ref"                  # 输出: 张三
ref="李四"                   # 通过引用修改
echo "$name"                 # 输出: 李四

# 对数组的引用
declare -a items=("苹果" "香蕉" "橘子")
declare -n arr_ref=items
echo "${arr_ref[1]}"         # 输出: 香蕉
arr_ref+=("西瓜")            # 通过引用添加元素
echo "${items[@]}"           # 苹果 香蕉 橘子 西瓜

# 对关联数组的引用
declare -A settings
settings["theme"]="dark"
settings["lang"]="zh-CN"

declare -n cfg=settings
echo "${cfg[theme]}"         # 输出: dark
cfg["font"]="monospace"      # 通过引用赋值
echo "${settings[font]}"     # 输出: monospace
```

## 11.3 nameref 在函数中的应用

```bash
# 函数接收变量名，通过 nameref 直接操作调用者的变量
increment() {
    local -n ref="$1"
    ref=$((ref + 1))
}

count=0
increment count
echo "$count"                # 输出: 1
increment count
echo "$count"                # 输出: 2

# 返回多个值（通过 nasref 填充多个变量）
split_host_port() {
    local addr="$1"
    local -n out_host="$2"
    local -n out_port="$3"
    out_host="${addr%:*}"
    out_port="${addr##*:}"
}

split_host_port "localhost:8080" host port
echo "主机: $host, 端口: $port"   # 主机: localhost, 端口: 8080

# 接收并修改调用者数组
shuffle_array() {
    local -n arr="$1"
    local len=${#arr[@]} i j tmp
    for ((i=len-1; i>0; i--)); do
        j=$((RANDOM % (i+1)))
        tmp="${arr[i]}"
        arr[i]="${arr[j]}"
        arr[j]="$tmp"
    done
}

my_array=(1 2 3 4 5 6 7 8 9 10)
shuffle_array my_array
echo "${my_array[@]}"
```

## 11.4 nameref 的作用域与循环引用

```bash
# nameref 在局部作用域中的行为
outer_var="外层"
declare -n ref=outer_var          # 全局引用

func() {
    local -n ref=outer_var         # 局部引用，但指向全局变量
    ref="内层修改"
    echo "内部: $ref"
}

func                               # 内部: 内层修改
echo "外层: $outer_var"            # 外层: 内层修改（全局变量被改了）

# 循环引用陷阱（避免）
a="b"
b="a"
declare -n ra=a
declare -n rb=b
# echo "$ra"       # 错误：循环引用，bash 返回错误
# echo "$rb"       # 同上

# 解除 nameref
declare -n ref=original
unset -n ref                       # 只解除引用绑定，不影响 original

# nameref 覆盖变量后恢复
original="原始值"
declare -n ref=original
ref="新值"                         # 修改了 original
unset -n ref
echo "$original"                   # 新值
```

## 11.5 ${!var} 与 nameref 对比

| 特性 | `${!var}` | `declare -n` |
|------|-----------|--------------|
| 间接层级 | 一层 | 一层 |
| 修改原变量 | 不能（只能读取） | 可以（读写均可） |
| 引用数组 | 需要 `idx="arr[1]"` 格式 | 自然支持 |
| 引用关联数组 | 需要 `key="map[key]"` 格式 | 自然支持 |
| 函数参数传递 | 需要手动处理 | local -n 直接使用 |
| 性能 | 略快（无额外绑定开销） | 略慢 |
| bash 版本 | 2.0+ | 4.3+ |

```bash
# 等效代码对比

# ${!var} 方式（只读）
varname="username"
echo "${!varname}"                # 读取

# nameref 方式（读写）
declare -n vref=username
echo "$vref"                      # 读取
vref="新值"                       # 写入

# 在函数中传递数组：两种方式
# 方式1：nameref（推荐，bash 4.3+）
process_array() {
    local -n arr="$1"
    echo "长度: ${#arr[@]}"
    for item in "${arr[@]}"; do
        echo "处理: $item"
    done
}

# 方式2：间接引用（兼容旧版）
process_array_old() {
    eval "local arr=(\"\${${1}[@]}\")"
    echo "长度: ${#arr[@]}"
}
```

# 12. 动态作用域

## 12.1 bash 的变量作用域模型

bash 采用**动态作用域**（Dynamic Scoping）——变量的可见性由**调用链**决定，而非代码的**词法结构**：

```bash
# 动态作用域演示
x="全局"

outer() {
    local x="外层"
    inner                  # 调用 inner 时，它能访问调用链上的变量
}

inner() {
    echo "x = $x"          # 输出什么？
}

# 调用
inner                      # 输出: x = 全局
outer                      # 输出: x = 外层
# inner 中的 $x 取决于谁调用了它，而非它在代码中定义的位置
```

**对比词法作用域（如 Python/JavaScript）：**

```python
# Python 等价代码（词法作用域）
x = "全局"
def outer():
    x = "外层"
    def inner():
        print(x)          # 始终引用 outer 中的 x（定义时的环境）
    inner()

outer()                   # 输出: 外层
# inner() 无法从全局作用域访问
```

## 12.2 调用栈查找机制

```bash
# bash 在函数中查找变量的顺序：
# 1. 当前函数的 local 变量
# 2. 调用者的 local 变量（如果调用者中也是 local 且未退出作用域）
# 3. 再上一层的调用者 ...
# 4. 全局变量

a="全局A"

level3() {
    echo "level3: a=$a"                   # 查找调用链，找到 level1 的 local a
}

level2() {
    level3
}

level1() {
    local a="level1-A"                    # 该变量的作用域一直延伸到调用的所有函数
    level2
}

level1                                    # 输出: level3: a=level1-A
echo "全局: a=$a"                         # 输出: 全局: a=全局A
```

## 12.3 利用动态作用域实现依赖注入

```bash
# 动态作用域可以模拟 依赖注入 / 上下文传递
# 好处：无需通过参数逐层传递

# 场景：在所有子函数中共享 logger 配置
init_logger() {
    local log_level="$1"
    local log_file="$2"

    # 所有函数运行时都能通过调用链访问到这些变量
    log() {
        local level="$1"; shift
        local caller="${FUNCNAME[1]:-main}"
        echo "[$(date)][$level][$caller] $*" >> "$log_file"
    }

    debug() { [[ "$log_level" == "DEBUG" ]] && log "DEBUG" "$@"; }
    info()  { log "INFO" "$@"; }
    error() { log "ERROR" "$@"; }

    do_work() {
        info "开始任务"
        helper_func
        info "任务完成"
    }

    helper_func() {
        debug "内部处理"                # log_level 来自调用链
        error "${err_msg_set_by_caller}" # 甚至可以访问调用者设置的变量
    }

    do_work
}

init_logger "DEBUG" "/tmp/app.log"
```

## 12.4 模拟闭包

```bash
# 利用动态作用域 + 函数定义模拟闭包效果

# 闭包模拟：计数器工厂
make_counter() {
    local count=0
    local counter_name="$1"

    # 将"闭包函数"定义为字符串，内嵌动态作用域变量
    eval "
    ${counter_name}_inc() {
        count=\$((count + 1))
        echo \"\$count\"
    }
    ${counter_name}_dec() {
        count=\$((count - 1))
        echo \"\$count\"
    }
    ${counter_name}_get() {
        echo \"\$count\"
    }
    ${counter_name}_reset() {
        count=0
        echo \"计数器已重置\"
    }
    "
}

make_counter "user"
make_counter "event"

user_inc; user_inc; user_inc    # 输出: 1 2 3
event_inc; event_inc             # 输出: 1 2
user_get                         # 输出: 3
event_get                        # 输出: 2
# user 和 event 的 count 完全独立！

# 利用 nameref + 闭包模式的现代化实现（bash 4.3+）
make_counter_v2() {
    local -n counter=$1
    local -n inc_func=$2
    local -n get_func=$3

    counter=0
    eval "
    ${!inc_func}() { counter=\$((counter + 1)); echo \"\$counter\"; }
    ${!get_func}() { echo \"\$counter\"; }
    "
}
```

## 12.5 动态作用域的陷阱与最佳实践

```bash
# 陷阱 1：意外变量遮蔽
LOGGER="syslog"

func_a() {
    local LOGGER="file"     # 遮蔽了全局变量
    func_b
}

func_b() {
    echo "$LOGGER"          # 输出: file（预期可能是 syslog）
}

# 最佳实践 1：使用独特的变量名前缀
_app_log_level="INFO"       # 前缀 _app_ 避免冲突

# 陷阱 2：未声明 local 污染调用者变量
outer() {
    result=""               # 非 local！
    inner "张三"
    echo "$result"
}

inner() {
    result="Hello, $1"      # 修改了 outer 的 result
}

# 最佳实践 2：函数内始终使用 local
outer_fixed() {
    local result
    inner_fixed "张三"
    echo "$result"
}

inner_fixed() {
    local result="Hello, $1"   # 局部变量，不影响调用者
    echo "$result"
}

# 陷阱 3：循环中函数调用可能误用动态作用域变量
for file in *.txt; do
    process_file
done

process_file() {
    # $file 来自调用环境的 for 循环（有意或无意）
    echo "处理: $file"
}

# 最佳实践 3：明确传入参数，减少对动态作用域的依赖
for file in *.txt; do
    process_file "$file"
done

process_file() {
    local f="$1"       # 明确接收参数
    echo "处理: $f"
}
```

## 12.6 作用域链模式总结

```bash
# bash 变量查找的四级作用域链
# +------------------+
# | 当前函数 local   | ← 最高优先级
# +------------------+
# | 调用者 local     | ← 动态作用域向上查找
# +------------------+
# | ...              | ← 继续沿调用链向上
# +------------------+
# | 全局变量         | ← 最低优先级
# +------------------+

# 显式指定作用域
declare -g GLOBAL_VAR="全局"          # 显式声明全局（在函数内部）
local LOCAL_VAR="局部"               # 显式声明局部

# 验证作用域
scope_demo() {
    local x="局部"
    inner() {
        local y="内部局部"
        echo "inner: x=$x, y=$y, z=$z"
    }
    z="这是调用者的非局部变量"        # 没有 local！
    inner
}
z="全局z"
scope_demo
# 输出: inner: x=局部, y=内部局部, z=这是调用者的非局部变量

# 子 Shell 中的动态作用域
# 子 Shell 继承变量值，但不共享作用域链
outer_var="outer"
(
    outer_var="子Shell修改"           # 仅影响子 Shell
    echo "子Shell: $outer_var"
)
echo "父Shell: $outer_var"            # 输出: outer（未受影响）
```
