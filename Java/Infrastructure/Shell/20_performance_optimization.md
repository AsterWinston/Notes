
# 1. 理解开销来源

Shell 脚本的性能瓶颈通常不在"代码逻辑"，而在**进程创建**和**I/O 模式**。理解底层发生了什么，才能写出高效的脚本。

## 1.1 fork/exec 成本

**fork 成本：** 当 Shell 执行一个外部命令（如 `grep`、`sed`、`awk`、`cat`）时，内核必须：

1. **fork()**：克隆当前进程的页表、文件描述符表、信号处理表。在 Linux 上由于 COW（Copy-On-Write），页表复制开销约为 0.5~2ms（取决于进程内存大小）。
2. **execve()**：加载新的 ELF 可执行文件，解析动态链接器（`ld.so`），加载共享库（`libc.so`、`libpcre.so` 等），初始化 BSS 段。
3. **调度延迟**：新进程进入就绪队列，等待 CPU 调度。

```bash
# 使用 strace 观察一个简单的 echo 外部命令调用的系统调用
strace -c -f bash -c 'echo hello'
# fork 调用次数: 1
# execve 调用次数: 1

# 对比：外部命令 vs 内置命令
strace -c -f bash -c 'command echo hello'
# 比上面的多了 fork + exec

# 对比：管道中每个命令都是独立进程
strace -c -f bash -c 'cat /etc/passwd | grep root | wc -l'
# fork 调用次数: 4（cat、grep、wc 各一个，加上 bash 自身一个子 shell）
```

**典型 fork/exec 耗时参考（Intel Xeon E5-2670 @ 2.60GHz）：**

| 操作 | 耗时 |
|------|------|
| bash 内置 `echo` | ~0.01 ms |
| 外部 `/bin/echo` | ~0.5 ms |
| `sed` 简单替换（含 fork/exec） | ~1.5 ms |
| `grep` 简单匹配（含 fork/exec） | ~1.5 ms |
| `awk` 简单计算（含 fork/exec） | ~2.0 ms |
| `perl` / `python` 单行（含 fork/exec） | ~8~15 ms |

**结论：** 在循环内调用外部命令是 Shell 脚本最大的性能杀手。10 万次循环 × 1.5ms = **2.5 分钟**，而同样的逻辑用 bash 内置功能可能只需 0.5 秒。

## 1.2 管道（Pipe）缓冲机制

管道是 Shell 组合命令的核心机制，理解其缓冲行为对性能分析至关重要：

```bash
# 管道底层：pipe() + fork() × 2
# 管道容量（默认）：
#   Linux >= 2.6.11: 65536 字节（64KB，16 个页）
#   较旧 Linux: 4096 字节（4KB，PIPE_BUF）
#   每个管道的 page 数量可通过 /proc/sys/fs/pipe-max-size 查看
```

**管道缓冲的关键行为：**

```bash
# 示例：管道阻塞实验
# producer 写入 64KB 后阻塞，直到 consumer 开始读取
dd if=/dev/zero bs=65536 count=1 2>/dev/null | (sleep 1; wc -c)
# 第一个 dd 写入 64KB 后阻塞 1 秒

# 超过 64KB 写入时，write() 阻塞
dd if=/dev/zero bs=65537 count=1 2>/dev/null | (sleep 1; wc -c)
# 第二个 dd 写入第一个 64KB 后立即阻塞，直到 sleep 结束
```

**管道缓冲区满的影响：**

- 生产者（管道左侧）`write()` 速度远大于消费者 `read()` 速度时，生产者频繁阻塞
- 造成"锯齿状"吞吐量模式——快写一段，阻塞，快写一段，阻塞
- 这种上下文切换本身也消耗 CPU

```bash
# 优化：使用缓冲区工具平滑管道流
# pv（Pipe Viewer）或 buffer 命令可提供大缓冲区
tar czf - /large/dir | pv -B 128M | ssh user@host "tar xzf -"
# -B 128M 提供 128MB 缓冲区，减少网络往返导致的阻塞
```

## 1.3 子 Shell 创建场景

子 Shell（subshell）虽不是外部命令，但仍需 fork（不需要 exec），成本约为外部命令的 60%~70%。

**触发子 Shell 的所有场景：**

```bash
# 1. 命令替换 $(...) 和 `...`
result=$(grep pattern file)          # fork + 管道读取

# 2. 管道中的每个命令段
cmd1 | cmd2 | cmd3                   # 每个都是子 Shell（或独立进程）

# 3. 显式子 Shell
( cd /tmp && ls )                    # fork 一个子 Shell

# 4. 后台任务
cmd &                                # fork + 后台执行

# 5. 进程替换 >(...)  <(...)
diff <(sort a.txt) <(sort b.txt)     # 每个进程替换都是一个子 Shell

# 6. coproc 协程
coproc myproc { while read; do ...; done; }
```

**命令替换中 $() 与 `` 的开销完全相同——都是 fork + 管道读取。**

# 2. 避免不必要的 fork

减少 fork/exec 是 Shell 优化的第一法则。

## 2.1 内置命令优先

Bash 内置命令运行在 Shell 进程内部，不需要 fork——这是零成本执行。

**Bash 完整内置命令列表（bash 5.x，共约 60 个）：**

| 类别 | 命令 |
|------|------|
| Shell 控制 | `.` `source` `:` `true` `false` `exit` `return` `break` `continue` |
| 变量/参数 | `declare` `typeset` `local` `readonly` `export` `unset` `shift` `set` `unset` `alias` `unalias` |
| I/O | `echo` `printf` `read` `readarray` `mapfile` `exec` |
| 目录/文件 | `cd` `pwd` `dirs` `popd` `pushd` |
| 作业控制 | `jobs` `fg` `bg` `wait` `kill` `disown` `suspend` |
| 条件/循环 | `[` `[[` `test` |
| 流程控制 | `command` `builtin` `enable` `eval` `times` `trap` `type` `hash` |
| 历史 | `history` `fc` |
| 其他 | `caller` `compgen` `complete` `getopts` `help` `logout` `shopt` `ulimit` `umask` `bind` |

```bash
# ❌ 低效：每个 echo 都是外部命令
for i in {1..1000}; do
    /bin/echo "$i"           # 1000 次 fork/exec
done

# ✅ 高效：使用内置命令
for i in {1..1000}; do
    echo "$i"                # 0 次 fork
done
# 耗时差异：~500ms vs ~5ms（约 100 倍）
```

```bash
# ❌ 低效：使用外部 true/false
while /bin/true; do
    # ...
done

# ✅ 高效：使用内置 true/false
while true; do
    # ...
done
```

```bash
# ❌ 低效：外部 kill
/bin/kill -0 $pid 2>/dev/null

# ✅ 高效：内置 kill
kill -0 $pid 2>/dev/null
```

## 2.2 减少命令替换次数

每次 `$()` 都创建一个子 Shell（fork），并通过管道将输出传回父进程。

```bash
# ❌ 低效：循环内反复命令替换（N 次 fork）
for file in *.txt; do
    lines=$(wc -l < "$file")           # 每次循环 fork wc
    echo "$file: $lines"
done

# ✅ 高效：批量操作，单次命令替换
wc -l *.txt | while read -r lines file; do
    echo "$file: $lines"
done
# 对比：1000 个文件，从 ~2000ms 降至 ~10ms

# ✅ 更高效：使用 awk 一次性处理
wc -l *.txt | awk '{print $2 ": " $1}'
```

```bash
# ❌ 每一个命令替换都是一次 fork
name=$(basename "$path")
dir=$(dirname "$path")
ext="${name##*.}"

# ✅ 全部用 bash 参数展开替代（0 次 fork）
name="${path##*/}"
dir="${path%/*}"
ext="${name##*.}"
```

```bash
# ❌ 多次命令替换读取同一文件
count=$(grep -c "ERROR" app.log)
latest=$(tail -1 app.log)
first=$(head -1 app.log)

# ✅ 一次读取，多次使用
{
    read -r first
    while read -r line; do
        [[ "$line" == *ERROR* ]] && ((count++))
        latest="$line"
    done
} < app.log
```

## 2.3 合并 sed/grep/awk 调用

```bash
# ❌ 低效：管道串行多个外部命令，多次 fork
cat access.log | grep "2024" | grep "ERROR" | sed 's/\[//g' | awk '{print $1}' | sort | uniq -c

# ✅ 只用 awk 一口气完成（1 次 fork）
awk '/2024/ && /ERROR/ { gsub(/\[/, ""); print $1 }' access.log | sort | uniq -c
# 从 6 个进程减至 3 个

# ✅ 进一步优化：awk 内排序去重
awk '
/2024/ && /ERROR/ { gsub(/\[/, ""); count[$1]++ }
END { for (ip in count) print count[ip], ip }
' access.log

# ✅ 终极版：无需排序，awk 内用关联数组自动去重计数
# 上面这个方案已经不依赖外部 sort/uniq，只有 1 个外部进程
```

```bash
# sed 合并：一个 sed 调用完成多个替换
# ❌ 三次 sed 调用，三次读取文件
sed 's/foo/bar/g' file.txt | sed 's/hello/world/g' | sed 's/^\s*//'

# ✅ 一次 sed 调用完成所有替换
sed -e 's/foo/bar/g' -e 's/hello/world/g' -e 's/^\s*//' file.txt

# ✅ 使用分号合并（某些 sed 版本需要）
sed 's/foo/bar/g; s/hello/world/g; s/^\s*//' file.txt
```

```bash
# grep 合并：多个 grep 串联
# ❌ 三次 grep，三次遍历文件
grep "pattern1" file | grep "pattern2" | grep -v "pattern3"

# ✅ 一次 awk/grep 完成（awk 方式更灵活）
# 用 grep -E（ERE 正则）
grep -E 'pattern1' file | grep -E 'pattern2' | grep -v 'pattern3'

# ✅ 最佳：用 awk 一次完成
awk '/pattern1/ && /pattern2/ && !/pattern3/' file
```

# 3. 循环优化

循环是 Shell 脚本中最容易产生性能问题的地方——尤其是在循环体内进行 fork。

## 3.1 避免循环内 fork

```bash
# ── 案例 1：循环内执行 date ──

# ❌ 1000 次 fork date
for i in {1..1000}; do
    timestamp=$(date +%s)        # 每次 fork 一次 date
    echo "$timestamp"
done
# 耗时：~2000ms

# ✅ 使用内置 printf 和 $SECONDS / $EPOCHSECONDS
for i in {1..1000}; do
    printf '%s\n' "$EPOCHSECONDS"    # 纯 bash，0 次 fork
done
# 耗时：~10ms
# 注意：$EPOCHSECONDS 需要 bash >= 5.0，否则用 printf '%(%s)T' -1

# ✅ bash 4.2+ 用 printf 时间格式化（仍然 0 fork）
printf -v timestamp '%(%Y-%m-%d %H:%M:%S)T' -1
```

```bash
# ── 案例 2：循环内执行 basename/dirname ──

# ❌ 每次循环 fork 两次 basename
for path in /a/b/c /d/e/f /g/h/i; do
    name=$(basename "$path")
    dir=$(dirname "$path")
done
# 耗时：10000 次循环 ~9000ms

# ✅ 参数展开替代（0 fork）
for path in /a/b/c /d/e/f /g/h/i; do
    name="${path##*/}"
    dir="${path%/*}"
done
# 耗时：10000 次循环 ~60ms（约 150 倍提升）
```

```bash
# ── 案例 3：累加计算使用 awk，避免循环内 expr ──

# ❌ 循环内 fork expr（极度低效）
sum=0
for i in {1..10000}; do
    sum=$(expr "$sum" + "$i")    # 每次 fork expr
done

# ❌ 稍好但仍低效：算术命令替换
sum=0
for i in {1..10000}; do
    sum=$((sum + i))             # 0 fork，但循环本身开销仍在
done

# ✅ 最佳：用 awk 直接计算
awk 'BEGIN { for(i=1;i<=10000;i++) s+=i; print s }'
# 或使用数学公式 sum = n*(n+1)/2
```

## 3.2 while read 替代 for 遍历

`for` 循环在遍历命令输出时，必须先把整个输出读入内存（或通过分词展开），而 `while read` 是流式处理。

```bash
# ── 案例 1：for 遍历命令输出 ──

# ❌ 低效：for 需要等待命令完成，将所有结果分词展开到内存
for line in $(cat huge_file.txt); do
    # ...
done
# 问题：
# 1. 先读取整个文件到内存
# 2. 分词将破坏每行的空白结构
# 3. 路径展开可能意外触发 glob

# ✅ 高效：while read 流式处理
while IFS= read -r line; do
    # ...
done < huge_file.txt
```

```bash
# ── 案例 2：for 遍历文件列表 ──

# ❌ 问题：遇到包含空格的文件名会出错
for file in $(ls *.txt); do
    echo "$file"
done

# ❌ 稍好但仍低效（大目录时 glob 展开占用大量内存）
for file in *.txt; do
    echo "$file"
done

# ✅ 稳健方式：流式处理
find . -name "*.txt" -print0 | while IFS= read -r -d '' file; do
    echo "$file"
done
```

```bash
# ── 案例 3：while read 的 fork 陷阱 ──

# ❌ 管道中的 while read 在子 Shell 中执行，变量修改丢失
count=0
cat file.txt | while read -r line; do
    ((count++))
done
echo "$count"    # 输出 0！因为在子 Shell 中的修改不影响父 Shell

# ✅ 方案 A：进程替换 + 重定向（bash 专用，避免子 Shell）
count=0
while read -r line; do
    ((count++))
done < <(cat file.txt)
echo "$count"    # 正确输出

# ✅ 方案 B：直接重定向文件（最简单）
count=0
while read -r line; do
    ((count++))
done < file.txt
echo "$count"    # 正确输出

# ✅ 方案 C：shopt -s lastpipe（bash 4.2+，管道最后一个命令在
#           当前 Shell 执行，需关闭作业控制）
shopt -s lastpipe
count=0
cat file.txt | while read -r line; do
    ((count++))
done
echo "$count"    # 正确输出（需在非交互式脚本中使用）
```

## 3.3 大括号展开替代 seq

`seq` 是外部命令，每次调用产生 fork；而大括号展开 `{1..N}` 是 bash 内置的，零 fork。

```bash
# ❌ 外部命令 seq（每次 fork）
for i in $(seq 1 10000); do
    echo "$i"
done

# ✅ 大括号展开（0 fork）
for i in {1..10000}; do
    echo "$i"
done
# 10 万次循环：seq 版本约 2500ms，大括号版本约 200ms（~12 倍）

# ── 注意：大括号展开无法使用变量 ──
start=1
end=100
# ❌ 无法工作：{1..100} 是字面量，不展开变量
for i in {$start..$end}; do echo "$i"; done
# 输出：{1..100}（字面字符串）

# ✅ 变量范围用 C 风格 for 循环
for ((i=start; i<=end; i++)); do
    echo "$i"
done

# ✅ 或使用 eval（谨慎）
eval "for i in {$start..$end}; do echo \"\$i\"; done"
```

```bash
# ── 大数据量下的内存考量 ──

# ⚠️ 大括号展开会一次性生成所有数字到内存
for i in {1..10000000}; do
    :   # 1000 万个数一次性展开在内存中（约 80MB）
done

# ✅ C 风格 for 循环不预生成序列，内存 O(1)
for ((i=1; i<=10000000; i++)); do
    :
done
```

## 3.4 until 循环替代方案

```bash
# ❌ 外部命令做条件检测
while [[ $(cat /proc/loadavg | awk '{print $1}') > "$threshold" ]]; do
    sleep 1
done

# ✅ 内置 read 直接读取 /proc
while IFS=' .' read -r load _; do
    [[ "$load" -le "$threshold" ]] && break
    sleep 1
done < /proc/loadavg

# 或最简方式，用 bash 内置
while read -r load _; do
    (( ${load/./} <= threshold * 100 )) && break
    sleep 1
done < <(cut -d' ' -f1 /proc/loadavg)
```

# 4. 文件 I/O 优化

## 4.1 批量读写策略

```bash
# ❌ 逐行读写（每次 read/write syscall 仅处理一行）
while IFS= read -r line; do
    echo "$line" >&3
done < input.txt 3> output.txt
# 100 万行：约 200 万次系统调用（read + write 各 100 万）

# ✅ 批量读取然后批量写入
# 方案 A：使用 mapfile/readarray 批量读入
mapfile -t lines < input.txt           # 一次 read 分批载入（底层缓冲 8192 字节）
printf '%s\n' "${lines[@]}" > output.txt   # 一次 write

# 方案 B：dd 批量复制
dd if=input.txt of=output.txt bs=1M status=progress
# bs=1M 以 1MB 块为单位读写，大幅减少系统调用

# 方案 C：批量 cat + 重定向（最简单）
cat input.txt > output.txt
```

**dd 块大小对性能的影响（复制 1GB 文件）：**

| bs 值 | 系统调用次数（约） | 用户态耗时 | 系统态耗时 | 总耗时 |
|-------|-------------------|-----------|-----------|--------|
| 1 byte | 10 亿次 read/write | 极高 | 极高 | > 10 分钟 |
| 512 B | 200 万次 | 高 | 高 | ~30 秒 |
| 4 KB | 25 万次 | 中 | 中 | ~3 秒 |
| 64 KB | 1.6 万次 | 低 | 低 | ~1 秒 |
| 1 MB | 1000 次 | 极低 | 极低 | ~0.5 秒 |
| 8 MB | 100 次 | 极低 | 极低 | ~0.3 秒 |

```bash
# 最佳实践：根据场景选择 bs
# 普通文件复制
dd if=src of=dst bs=64K status=progress

# 大文件高速复制（内存充足时）
dd if=src of=dst bs=1M status=progress

# SSD / NVMe 文件复制
dd if=src of=dst bs=4M status=progress

# 网络文件系统（NFS/CIFS）
dd if=src of=dst bs=128K status=progress
# 避免太大导致网络分片后重组开销
```

## 4.2 mmap 简介与 Shell 限制

mmap 将文件映射到进程地址空间，允许以内存访问的方式读写文件，省去 read/write 系统调用。**但 Shell 脚本本身无法直接使用 mmap**——这是 C/Python 等语言的能力。

```bash
# Shell 中的"近似 mmap"技术：
# 1. vmtouch 工具（将文件锁定到 page cache）
vmtouch -l huge_file.txt    # 将文件锁入内存
vmtouch -t huge_file.txt    # 预加载到 page cache（touch）

# 2. 利用 OS 的 page cache 预热
cat huge_file.txt > /dev/null   # 将文件读入 page cache
# 此后所有对该文件的读取都从内存中服务，避免磁盘 I/O

# 3. 使用 tmpfs（内存文件系统）
cp huge_file.txt /dev/shm/             # 复制到 tmpfs
process_file /dev/shm/huge_file.txt    # 从内存中处理
rm /dev/shm/huge_file.txt
```

```bash
# page cache 的利用策略
# Linux 的 page cache 自动缓存文件 I/O 结果
# 第一次读取：磁盘 I/O
time cat large_file.log > /dev/null
# real    0m2.500s   ← 磁盘 I/O

# 第二次读取：page cache 命中，纯内存操作
time cat large_file.log > /dev/null
# real    0m0.050s   ← 快了 50 倍

# 清空 page cache（需要 root）
echo 3 > /proc/sys/vm/drop_caches
```

## 4.3 避免逐字节处理

```bash
# ❌ 低效：逐字节读取（极度不推荐）
while IFS= read -r -n1 byte; do
    # 处理每个字节...
done < file.bin
# 原因：每次 read -n1 都是一次 read() 系统调用（虽然 stdio 缓冲
#       可能帮你做缓冲，但 bash 的 read 每次只请求 1 个字节）

# ❌ 低效：逐字段处理大文件
while IFS=, read -r f1 f2 f3 f4 f5 f6; do
    echo "$f1,$f3,$f5"
done < huge.csv

# ✅ 高效：用 cut/awk 批量处理
cut -d, -f1,3,5 huge.csv
# 或
awk -F, '{print $1 "," $3 "," $5}' huge.csv
# 原因：cut/awk 使用 8KB~64KB 缓冲区，批量 read
```

```bash
# ❌ 逐字符替换（低效）
while IFS= read -r line; do
    result=""
    for ((i=0; i<${#line}; i++)); do
        char="${line:$i:1}"
        [[ "$char" == "a" ]] && char="A"
        result+="$char"
    done
    echo "$result"
done < input.txt

# ✅ 批量替换
sed 's/a/A/g' input.txt
# 或 bash 内置
while IFS= read -r line; do
    echo "${line//a/A}"
done < input.txt
```

## 4.4 文件描述符复用

减少重复打开/关闭文件的系统调用开销：

```bash
# ❌ 在循环中反复打开同一个文件
for i in {1..10000}; do
    echo "$i" >> output.log       # 每次重定向都 open() + lseek() + write() + close()
done
# 10000 次 open/close 对

# ✅ 预先打开文件描述符，循环内复用
exec 3>>output.log                # 一次 open
for i in {1..10000}; do
    echo "$i" >&3                 # 仅 write，不 open/close
done
exec 3>&-                         # 一次 close
# 仅 1 次 open/close，~40 倍提升
```

```bash
# ❌ 管道中多次读写同一个临时文件
grep "ERROR" app.log > /tmp/errors
grep "2024" /tmp/errors > /tmp/errors_2024
wc -l /tmp/errors_2024

# ✅ 使用进程替换一次性处理
grep "ERROR" app.log | grep "2024" | wc -l
# 避免了两次文件 I/O（虽然增加了两个管道进程，但都在内存中）
```

## 4.5 零拷贝思想在 Shell 中的应用

Linux 的 `sendfile()`、`splice()` 系统调用可以在内核态直接传输数据，无需用户态拷贝。Shell 中可间接利用：

```bash
# cp 命令在支持 reflink 的 FS 上实现零拷贝
# Btrfs / XFS / OCFS2 支持 reflink
cp --reflink=always source_file dest_file
# 不复制数据块，仅创建新的 inode 指向相同的数据区（COW）

# rsync 使用增量传输减少 I/O
rsync -av --inplace source/ dest/

# tee 的零拷贝应用
# 一份输入，多份输出（通过内核 VFS 层复制）
grep "ERROR" app.log | tee >(gzip > errors.gz) >(wc -l) >/dev/null
```

# 5. 字符串操作优化

## 5.1 参数展开替代外部工具

Bash 的参数展开（Parameter Expansion）是内置功能，零 fork，是字符串处理的第一选择。

**完整对照表：**

| 需求 | ❌ 外部命令 | ✅ bash 参数展开 |
|------|-----------|-----------------|
| 取文件名 | `basename "$path"` | `"${path##*/}"` |
| 取目录名 | `dirname "$path"` | `"${path%/*}"` |
| 取扩展名 | `echo "$f" \| sed 's/.*\.//'` | `"${f##*.}"` |
| 去掉扩展名 | `echo "$f" \| sed 's/\.[^.]*$//'` | `"${f%.*}"` |
| 转大写 | `echo "$s" \| tr '[:lower:]' '[:upper:]'` | `"${s^^}"` (bash 4+) |
| 转小写 | `echo "$s" \| tr '[:upper:]' '[:lower:]'` | `"${s,,}"` (bash 4+) |
| 替换（首次） | `echo "$s" \| sed 's/old/new/'` | `"${s/old/new}"` |
| 替换（全部） | `echo "$s" \| sed 's/old/new/g'` | `"${s//old/new}"` |
| 替换前缀 | `echo "$s" \| sed 's/^prefix//'` | `"${s#prefix}"` |
| 替换后缀 | `echo "$s" \| sed 's/suffix$//'` | `"${s%suffix}"` |
| 取子串 | `echo "$s" \| cut -c1-5` | `"${s:0:5}"` |
| 字符串长度 | `echo "$s" \| wc -c` | `"${#s}"` |
| 默认值 | `[ -z "$x" ] && x="default"` | `"${x:-default}"` |
| 去除首尾空白 | `echo "$s" \| sed 's/^\s*\|\s*$//g'` | `"${s##*( )}"` 配合 extglob |

```bash
# 实战示例：处理一批文件路径
paths=(
    "/home/user/docs/report.pdf"
    "/var/log/app/access.log"
    "/tmp/data/backup.tar.gz"
)

# ❌ 方式：每个操作都 fork 外部命令
for p in "${paths[@]}"; do
    name=$(basename "$p")                         # fork
    dir=$(dirname "$p")                           # fork
    ext=$(echo "$name" | sed 's/.*\.//')          # fork
    base=$(echo "$name" | sed 's/\.[^.]*$//')     # fork
    echo "文件: $base, 扩展名: $ext, 目录: $dir"
done
# 4 个文件 → 16 次 fork

# ✅ 全部参数展开，0 次 fork
for p in "${paths[@]}"; do
    name="${p##*/}"
    dir="${p%/*}"
    ext="${name##*.}"
    base="${name%.*}"
    echo "文件: $base, 扩展名: $ext, 目录: $dir"
done
```

## 5.2 合适的工具选择

> "不要手里拿着锤子，看什么都是钉子。"

```bash
# ── 纯文本替换：tr（C 实现，速度最快）vs sed ──
# 测试：100MB 文件的字符替换
time tr 'a' 'A' < large.txt > /dev/null          # ~0.3s
time sed 'y/a/A/' large.txt > /dev/null          # ~0.8s
time sed 's/a/A/g' large.txt > /dev/null         # ~2.5s
time awk '{gsub(/a/,"A");print}' large.txt > /dev/null  # ~8s
# tr 最快：单一职责，单字节映射，使用 SIMD 优化

# ── 固定字符截取：cut vs awk ──
time cut -d, -f1,3,5 large.csv > /dev/null        # ~0.4s
time awk -F, '{print $1","$3","$5}' large.csv     # ~1.5s
# cut 更快：因为不做正则匹配，只是分割
```

**工具选择速查（按速度排序，快→慢）：**

```
功能                   首选                次选              避免
─────────────────────────────────────────────────────────────────
单字节替换            tr                  sed 'y/.../.../'   awk gsub
按分隔符取列          cut                 awk -F             read
模式匹配+打印         grep                awk '/re/'         sed -n '/re/p'
模式匹配+计数         grep -c             awk '/re/{c++}'    wc -l + grep
复杂文本处理          awk                 perl               bash 循环
数学计算              awk                 bc                 bash $((  ))
多条件过滤            awk                 grep | grep       多次 grep 管道
行排序                sort                awk 关联数组       bash 循环排序
去重                  sort -u             awk '!a[$0]++'    uniq（需预排序）
文件查找              find                ls | grep          遍历+test
```

## 5.3 Here-String vs Here-Doc vs echo 管道

```bash
# ── Here-String（<<<）：bash 专用，最快 ──
# 原理：创建临时文件 /tmp/sh-thd.XXXXXX，写入字符串，立即重定向
grep "pattern" <<< "$variable"

# ── Here-Doc（<<EOF）：最快的大段文本传递 ──
# 原理：bash 直接通过管道传递，不创建临时文件
grep "pattern" <<'EOF'
line1
line2
line3
EOF

# ── echo + 管道：额外多一个进程 ──
echo "$variable" | grep "pattern"

# 性能对比（10 万次调用）：
# here-string: ~200ms
# here-doc:    ~180ms（大段文本优势明显）
# echo 管道:   ~600ms（多 fork 一个 echo）
```

# 6. 并行处理模式

## 6.1 & + wait 模式

最基础的 Shell 并行，但需要手动控制并发数。

```bash
# ── 基础并行模式：固定并发数 ──
# 假设要处理 1000 个文件，每个处理 2 秒，串行需要 2000 秒
# 并行 8 个，只需 250 秒

MAX_PROCS=8                           # 最大并发数
running=0

process_file() {
    local file="$1"
    # 模拟耗时操作
    gzip -9 "$file"
}

for file in *.log; do
    # 等待直到有空闲槽位
    while (( running >= MAX_PROCS )); do
        wait -n                       # 等待任意一个子进程结束
        ((running--))
    done

    process_file "$file" &
    ((running++))
done

# 等待所有子进程结束
wait
echo "All files processed"
```

```bash
# ── 进阶模式：基于文件名的分片并行 ──
# 将工作分片给固定数量的 worker

WORKERS=4
TASK_DIR="./tasks"
RESULT_DIR="./results"

mkdir -p "$RESULT_DIR"

worker() {
    local id="$1"
    local task_file="$2"
    while IFS= read -r task; do
        [[ -z "$task" ]] && continue
        # 处理任务...
        process_line "$task" > "$RESULT_DIR/result_${id}_$$"
    done < "$task_file"
}

# 用 split 将任务文件分成 N 份
split -n "l/$WORKERS" "$TASK_DIR/all_tasks.txt" "$TASK_DIR/chunk_"

# 启动 worker
for ((i=0; i<WORKERS; i++)); do
    worker "$i" "$TASK_DIR/chunk_$(printf '%02d' $((i+1)))" &
done
wait

# 合并结果
cat "$RESULT_DIR"/result_* > "$RESULT_DIR/final.txt"
```

## 6.2 xargs -P 并行模式

`xargs -P` 是最简单的并行工具，无需安装，适合"对每个输入项执行相同命令"的场景。

```bash
# ── xargs -P 基础用法 ──
# -P N：最多同时运行 N 个进程
# -n M：每次传递 M 个参数给一个命令
# -I {}：占位符替换模式（隐式 -n 1）

# 例：并行 gzip 压缩所有 .log 文件
find . -name "*.log" -print0 | xargs -0 -P 8 -n 1 gzip -9
# -0: 以 null 分隔（配合 find -print0）
# -P 8: 最多 8 个并发
# -n 1: 每次传 1 个文件给 gzip

# 例：并行下载（注意控制并发，避免被限流）
cat urls.txt | xargs -P 4 -n 1 curl -s -O

# 例：并行处理 CSV，每个 chunk 100 行
xargs -P 4 -n 100 -a huge.csv process_chunk.sh
```

```bash
# ── xargs -P 的批量模式优化 ──

# ❌ 每行一个进程（-n 1），100 万行 = 100 万个进程
cat huge.txt | xargs -P 8 -n 1 my_script.sh

# ✅ 每 1000 行一个进程，减少进程创建开销
cat huge.txt | xargs -P 8 -n 1000 my_script.sh
# 100 万个进程 vs 1000 个进程，启动开销降低 1000 倍

# ✅ 使用 -L 按行分组（而非参数个数）
xargs -P 8 -L 1000 -a huge.txt my_script.sh
```

```bash
# ── xargs -P 的输出顺序问题 ──
# xargs -P 并行执行时，输出是交错的（乱序的）
# 如果需要保持顺序，用 --output-marker 或手动加序号

# ❌ 输出乱序
seq 1 100 | xargs -P 4 -n 1 echo "Processing item"
# 可能输出：Processing item 3, 1, 4, 2, ...

# ✅ 方案 1：结果写入文件，按序号重新排序
seq 1 100 | xargs -P 4 -I {} bash -c 'echo "Result for {}" > result_{}.txt'
cat result_*.txt | sort -t_ -k2 -n

# ✅ 方案 2：使用 GNU parallel 的 --keep-order 选项
```

## 6.3 GNU parallel 高级并行

GNU parallel 是 xargs -P 的超集，功能强大但需要额外安装。

```bash
# ── 安装 ──
# Debian/Ubuntu: apt install parallel
# RHEL/CentOS:   yum install parallel
# macOS:         brew install parallel

# ── 基础用法 ──
# 并行 gzip（自动检测 CPU 核心数）
parallel gzip -9 ::: *.log

# 指定并发数
parallel -j 8 gzip -9 ::: *.log

# 从文件读取输入
parallel -j 4 curl -s -O :::: urls.txt

# ── 保持输出顺序 ──
parallel --keep-order echo "Processing {}" ::: {1..100}
# 输出始终是 1, 2, 3, ... 100（虽然并行执行）

# ── 进度条 ──
parallel --bar -j 8 gzip -9 ::: *.log
# 显示进度条

# ── 失败重试 ──
parallel --retries 3 --joblog joblog.txt ./flaky_script.sh ::: input_*.txt
# 失败后最多重试 3 次，记录到 joblog.txt
```

```bash
# ── GNU parallel 高级模式 ──

# 跨服务器并行（SSH）
parallel -S server1,server2,server3 --workdir . process.sh ::: *.data

# 跨服务器同时限制每台并发数
parallel -S 4/server1,4/server2 process.sh ::: *.data
# server1 和 server2 各最多 4 个并发

# 管道模式（每个输入项通过管道处理）
cat urls.txt | parallel -j 8 -N 1 'curl -s {} | analyze.sh'

# 三路输入组合（笛卡尔积）
parallel echo "param1={1} param2={2} param3={3}" \
    ::: a b c ::: 1 2 3 ::: x y z
# 输出 3×3×3 = 27 行组合

# 链接输入（一一对应，非笛卡尔积）
parallel --link echo "{1} -> {2}" ::: a b c ::: 1 2 3
# 输出："a -> 1" "b -> 2" "c -> 3"
```

```bash
# ── parallel 环境变量传递 ──
# 导出当前 Shell 变量到 parallel 子进程
export DB_HOST DB_PORT DB_NAME

# --env 传递特定变量
parallel --env DB_HOST --env DB_PORT query_db.sh ::: queries/*.sql

# 传递函数定义（bash 函数也可以并行执行）
my_func() {
    echo "Processing $1: result=$(( $1 * 2 ))"
}
export -f my_func
parallel my_func ::: {1..100}
```

```bash
# ── GNU parallel vs xargs -P 对照 ──

# xargs -P
find . -name "*.log" -print0 | xargs -0 -P 8 -n 1 gzip

# GNU parallel（等价的）
find . -name "*.log" -print0 | parallel -0 -j 8 gzip
# 或更简洁
parallel -j 8 gzip ::: *.log
```

**GNU parallel vs xargs -P 特性对比：**

| 特性 | xargs -P | GNU parallel |
|------|----------|--------------|
| 安装 | 系统自带 | 需单独安装 |
| 保持输出顺序 | ❌ | `--keep-order` |
| 进度显示 | ❌ | `--bar`, `--progress` |
| 失败重试 | ❌ | `--retries N` |
| SSH 远程并行 | ❌ | ✅ `-S host` |
| 函数传递 | ❌ | `export -f` + `parallel` |
| 引用安全（空格） | `-0` / `-d '\n'` | 自动引用安全 |
| 交错输出 | xargs 默认 | `--line-buffer` / `--group` |
| 作业日志 | ❌ | `--joblog file` |
| 超时控制 | ❌ | `--timeout N` |

## 6.4 并行化的陷阱

```bash
# ── 陷阱 1：竞争条件（Race Condition） ──

# ❌ 多个进程同时写同一个文件
parallel process.sh ::: *.txt > output.txt
# 问题：输出被截断、交错、损坏

# ✅ 每个进程写临时文件，最后合并
parallel process.sh {} ">" result_{#}.tmp ::: *.txt
cat result_*.tmp > output.txt
rm result_*.tmp

# ── 陷阱 2：资源耗尽 ──

# ❌ 1000 个文件同时 gzip 会耗尽 I/O 带宽和内存
parallel -j 1000 gzip ::: *.log

# ✅ 限制并发数
# 规则：I/O 密集型 ≤ 磁盘数 × 2
#       CPU 密集型 ≤ 核心数
#       网络密集型 ≤ 带宽 / 单任务带宽 + 留余量
parallel -j "$(nproc)" gzip ::: *.log

# ── 陷阱 3：stdin 竞争 ──

# ❌ 多个子进程竞争 stdin
parallel my_script ::: a b c
# 如果 my_script 中 read 从 stdin 读取，行为不确定

# ✅ 显式重定向 stdin
parallel my_script {} '</dev/null' ::: a b c
```

# 7. 大文件处理策略

## 7.1 分块处理

```bash
# ── split 命令分块 ──
# 按行数分块（每块 100 万行）
split -l 1000000 huge_file.csv chunk_ --additional-suffix=.csv

# 按大小分块（每块 500MB）
split -b 500M huge_file.csv chunk_ --additional-suffix=.csv

# 处理后合并
for chunk in chunk_*.csv; do
    process_chunk "$chunk" > "processed_$chunk"
done
cat processed_chunk_* > final_result.csv
```

```bash
# ── head + tail 分段读取 ──
# 避免一次读入整个文件，按行号区间读取
process_range() {
    local start="$1"
    local end="$2"
    local file="$3"
    # 读取第 start~end 行
    tail -n "+$start" "$file" | head -n "$((end - start + 1))" | process_lines
}

# 对于超大文件，一个进程负责一个区间
# 但 tail + head 的方式每次都要从头 skip，大文件效率低
# 更好的方式是一次性分块：
split -l 100000 huge.csv chunk_
```

```bash
# ── dd 随机访问分段 ──
# 效率高于 tail（不需要从头读取）
# 按字节偏移分段读取
total_size=$(stat -c%s "huge_file.bin")
chunk_size=104857600     # 100MB
chunks=$(( (total_size + chunk_size - 1) / chunk_size ))

for ((i=0; i<chunks; i++)); do
    offset=$((i * chunk_size))
    dd if="huge_file.bin" bs=1M skip="$((offset / 1048576))" count=100 \
       2>/dev/null | process_binary_chunk &
done
wait
```

## 7.2 流式处理

流式处理（streaming）是处理大文件的核心思想——**永远不要一次读入全部数据**。

```bash
# ── 管道流式处理链 ──
# 每一步都是流式的，内存占用 O(1)（或很小）
cat huge_access.log \
    | grep "ERROR" \
    | awk '{print $1, $4}' \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -20
# 唯一非流式的是中间的 sort（需要内存排序或外部归并）

# ── 解决 sort 瓶颈 ──
# sort 默认使用 /tmp 做外部归并排序（--buffer-size 控制内存用量）
sort -S 2G --parallel=4 --temporary-directory=/fast_ssd huge_file.txt
# -S 2G: 内存 buffer 2GB
# --parallel=4: 4 个线程
# --temporary-directory: 临时文件放在快速磁盘
```

```bash
# ── awk 流式聚合（无需 sort） ──
# awk 关联数组在内存中聚合，适合基数不大的 key
# 8GB 内存可以容纳约 5000 万到 1 亿个唯一 key
awk '
{
    status[$9]++
    sum[$9] += $10
    count[$9]++
}
END {
    for (key in status) {
        printf "%s\t%d\t%.2f\n", key, status[key], sum[key] / count[key]
    }
}' huge_access.log

# 如果 key 基数超过内存，需要两遍处理：
# 第一遍：对大 key 采样；第二遍：分批聚合
```

```bash
# ── sed 流式替换（不修改原文件） ──
# sed 默认就是流式的，内存 O(line_size)
sed 's/\r$//' huge_dos_file.txt > unix_file.txt     # DOS 转 Unix 换行
sed '/^\s*$/d' huge_file.txt > no_empty_lines.txt    # 删除空行

# ── 注意：sed -i 不是流式的！ ──
# -i 会创建临时文件，然后再 rename，等于全量复制
sed -i 's/old/new/g' huge_file.txt   # 内部：创建 tmp → 写全量 → rename
```

## 7.3 避免全量读入内存

```bash
# ❌ 低效：将整个文件读入 Shell 变量（内存爆炸）
data=$(cat 10GB_file.txt)     # 将 10GB 读入 Shell 变量内存
echo "${#data}"               # 崩溃或极慢

# ❌ 低效：mapfile 读入整个文件
mapfile -t lines < 10GB_file.txt    # 每一行存在数组中
# 1000 万行 × 100 字节 ≈ 1GB（不含 bash 数组开销）

# ✅ 流式处理：一次读一行
while IFS= read -r line; do
    # 处理单行
done < 10GB_file.txt

# ✅ 使用批量 mapfile 分批读取
# 每批 10000 行
while mapfile -t -n 10000 lines && ((${#lines[@]})); do
    for line in "${lines[@]}"; do
        # 处理行
    done
    lines=()
done < 10GB_file.txt
# 比逐行 read 快 10~100 倍（减少了 read 系统调用次数）
```

```bash
# ── 内存占用分析：注意隐式内存消耗 ──

# 命令替换的内存占用
result=$(command)   # 整个命令输出全部读入内存（作为字符串）

# 进程替换不占用 Shell 内存（数据在管道缓冲区中）
while read -r line; do
    :
done < <(command)   # 数据流经管道，不在 Shell 内存中

# 管道内存占用
command | while read -r line; do
    :
done
# 管道缓冲区 64KB，满了 command 会阻塞——天然的背压机制
```

# 8. LC_ALL=C 的性能提升原理

## 8.1 原理

`LC_ALL=C` 将区域设置（Locale）切换为最简 POSIX 模式。这带来的性能提升源于以下机制：

1. **字符分类使用字节比较，而非 Unicode 查表**
   - `[a-z]` 在 UTF-8 locale 下需查 Unicode 字符属性表（GB18030、CJK 等几千个字符）
   - `LC_ALL=C` 下 `[a-z]` 仅比较 ASCII 97~122，O(1) 字节比较

2. **排序使用 `strcmp()`，而非 `strcoll()`**
   - `strcoll()` 需要加载 locale 排序规则表（`LC_COLLATE`）
   - `strcmp()` 直接逐字节比较，走 CPU cache 友好路径

3. **正则引擎简化路径**
   - UTF-8 regex 需要处理多字节字符边界，使用 NFA/DFA 状态机
   - 单字节模式下引擎路径更短

4. **避免了 iconv 风格的字符转换和宽字符 `mbrtowc()` 调用**

## 8.2 性能数据

```bash
# ── 性能测试 ──
# 测试文件：100MB 纯 ASCII 文本

# grep 测试
time grep 'pattern' large_file.txt > /dev/null               # ~0.8s
time LC_ALL=C grep 'pattern' large_file.txt > /dev/null       # ~0.3s  (2~3x 快)

# sort 测试
time sort large_file.txt > /dev/null                          # ~15s
time LC_ALL=C sort large_file.txt > /dev/null                 # ~4s   (3~4x 快)

# sed 测试（正则替换）
time sed 's/[a-z]/X/g' large_file.txt > /dev/null             # ~3.5s
time LC_ALL=C sed 's/[a-z]/X/g' large_file.txt > /dev/null    # ~0.8s  (4x 快)

# awk 测试
time awk '{print tolower($0)}' large_file.txt > /dev/null     # ~5s
time LC_ALL=C awk '{print tolower($0)}' large_file.txt > /dev/null # ~3s (1.5x 快)
```

**各命令受 LC_ALL=C 影响程度：**

| 命令 | 提速倍数 | 原因 |
|------|---------|------|
| `sort` | 3~5x | 排序规则的 `strcoll()` → `strcmp()` |
| `sed`（regex） | 2~4x | 正则引擎的多字节 → 单字节路径 |
| `grep` | 2~3x | 字符匹配的 locale 查表 → 字节比较 |
| `awk` | 1.5~2x | 字符处理函数（tolower/toupper）locale 依赖 |
| `tr` | 1.5~2x | 字符集映射简化 |
| `wc` | 1.1~1.3x | 影响较小 |
| `cat` | 1x | 几乎无影响 |

## 8.3 使用姿势

```bash
# ── 方式 1：脚本整体设置 ──
#!/bin/bash
export LC_ALL=C
# 整个脚本及子进程都使用 C locale

# ── 方式 2：局部使用（推荐） ──
# 仅在关键命令前设置，不影响其他逻辑
LC_ALL=C sort huge_file.txt
LC_ALL=C grep -E '[a-z]+' huge_file.txt

# ── 方式 3：函数封装 ──
fast_grep() {
    LC_ALL=C grep "$@"
}
fast_sort() {
    LC_ALL=C sort "$@"
}
fast_sed() {
    LC_ALL=C sed "$@"
}
```

```bash
# ── 注意事项 ──

# ⚠️ 1. 非 ASCII 字符行为可能改变
echo "café" | LC_ALL=C tr 'a-z' 'A-Z'   # 输出 "CAFé"（é 不变）
echo "café" | LC_ALL=en_US.UTF-8 tr '[:lower:]' '[:upper:]'  # 输出 "CAFÉ"

# ⚠️ 2. 排序顺序不同
printf "a\nA\nb\nB\n" | LC_ALL=en_US.UTF-8 sort    # a A b B
printf "a\nA\nb\nB\n" | LC_ALL=C sort               # A B a b（ASCII 顺序）

# ⚠️ 3. 如果你的数据全是纯 ASCII（如日志、CSV、JSON），
#     使用 LC_ALL=C 是安全的且应成为默认优化手段
```

# 9. 原生 bash 特性替代外部命令对照表

下表汇总了可以用 bash 内置功能完全替代外部命令的场景。表中的"外部命令"列每行代表一次 fork/exec。

| 外部命令 | bash 内置替代 | 性能提升 | bash 版本要求 |
|---------|-------------|---------|-------------|
| `basename "$path"` | `"${path##*/}"` | ~100x | 任意 |
| `dirname "$path"` | `"${path%/*}"` | ~100x | 任意 |
| `expr $a + $b` | `$((a + b))` | ~500x | 任意 |
| `expr "$s" : '.*'` | `${#s}` | ~500x | 任意 |
| `echo "$s" \| tr 'a-z' 'A-Z'` | `${s^^}` | ~200x | bash 4.0+ |
| `echo "$s" \| tr 'A-Z' 'a-z'` | `${s,,}` | ~200x | bash 4.0+ |
| `echo "$s" \| sed 's/a/b/'` | `${s/a/b}` | ~300x | 任意 |
| `echo "$s" \| sed 's/a/b/g'` | `${s//a/b}` | ~300x | 任意 |
| `echo "$s" \| sed 's/^pre//'` | `${s#pre}` | ~300x | 任意 |
| `echo "$s" \| sed 's/suf$//'` | `${s%suf}` | ~300x | 任意 |
| `echo "$s" \| cut -c1-5` | `${s:0:5}` | ~200x | 任意 |
| `echo "$s" \| cut -c6-` | `${s:5}` | ~200x | 任意 |
| `test -z "$s"` | `[[ -z $s ]]` | ~1.5x | bash 2.0+ |
| `[ "$a" = "$b" ]` | `[[ $a == $b ]]` | ~1.5x | bash 2.0+ |
| `[ "$a" -gt "$b" ]` | `(( a > b ))` | ~2x | 任意 |
| `date +%s` | `$EPOCHSECONDS` / `printf '%(%s)T' -1` | ~500x | bash 4.2+/5.0+ |
| `seq 1 1000` | `{1..1000}` | ~10x | 任意 |
| `sleep 0.5` | `read -t 0.5` | ~1x | bash 2.0+ |
| `echo -n "$s"` | `printf '%s' "$s"` | ~1x（但更可移植） | 任意 |
| `wc -l < file` | `readarray -t lines < file; echo ${#lines[@]}` | 取决于文件大小 | bash 4.0+ |
| `wc -c < file` | 无直接等效，用 `stat` 或 `read` 方案 | — | — |
| `head -n1` | `read -r first < file` | ~50x | 任意 |
| `mkdir -p a/b/c` | `mkdir -p a/b/c`（本身是外部命令，无替代） | — | — |

```bash
# ── 实战案例：将依赖 10+ 个外部命令的脚本改为全内置 ──

# ❌ 改造前：每次调用都 fork
#!/bin/bash
NOW=$(date +%Y%m%d_%H%M%S)                      # fork date
DAY=$(date +%d)                                  # fork date
YEAR=$(date +%Y)                                 # fork date
BASENAME=$(basename "$0")                        # fork basename
DIRNAME=$(dirname "$0")                          # fork dirname
COUNT=$(wc -l < input.txt)                       # fork wc
EXT=$(echo "$FILENAME" | sed 's/.*\.//')          # fork sed
UPPER=$(echo "$TEXT" | tr 'a-z' 'A-Z')            # fork tr
SUM=$(expr $A + $B)                              # fork expr
LEN=$(expr "$STR" : '.*')                        # fork expr
seq 1 "$COUNT" > /tmp/indices                    # fork seq

# ✅ 改造后：0 次 fork
#!/bin/bash
printf -v NOW '%(%Y%m%d_%H%M%S)T' -1            # bash 4.2+ builtin
printf -v DAY '%(%d)T' -1
printf -v YEAR '%(%Y)T' -1
BASENAME="${0##*/}"                              # 参数展开
DIRNAME="${0%/*}"                                 # 参数展开
readarray -t lines < input.txt; COUNT=${#lines[@]}
EXT="${FILENAME##*.}"
UPPER="${TEXT^^}"                                # bash 4.0+
SUM=$((A + B))
LEN=${#STR}
eval "echo {1..$COUNT}" > /tmp/indices           # 括号展开 + eval
```

# 10. 性能测量与分析

## 10.1 time 命令

```bash
# ── bash 内置 time ──
# 格式：real（墙钟时间） user（用户态CPU） sys（内核态CPU）
time ls -la /
# real    0m0.015s     ← 实际经过的墙钟时间
# user    0m0.008s     ← CPU 在用户态花费的时间
# sys     0m0.004s     ← CPU 在内核态花费的时间

# 关键指标：
# - real ≈ user + sys → CPU 密集型（进程一直在运行）
# - real >> user + sys → I/O 密集型 / 等待型（大量时间在等磁盘/网络）
# - real < user + sys  → 多核并行执行（多线程/多进程）

# 自定义 time 格式
TIMEFORMAT=$'real: %R\nuser: %U\nsys: %S\nCPU%%: %P%%'
time sleep 1
# real: 1.000
# user: 0.000
# sys: 0.000
# CPU%: 0.00%           ← 1 秒中 99.9999% 在 sleep
```

```bash
# ── /usr/bin/time -v（GNU time，详细版） ──
# 提供更多系统级指标

# 安装：apt install time / yum install time
# 注意：/usr/bin/time 和 bash 内置 time 是不同版本

/usr/bin/time -v gzip large_file.txt
#         Command being timed: "gzip large_file.txt"
#         User time (seconds): 2.34
#         System time (seconds): 0.12
#         Percent of CPU this job got: 99%
#         Elapsed (wall clock) time (h:mm:ss or m:ss): 0:02.47
#         Average shared text size (kbytes): 0
#         Average unshared data size (kbytes): 0
#         Average stack size (kbytes): 0
#         Average total size (kbytes): 0
#         Maximum resident set size (kbytes): 3072       ← 最大内存占用
#         Average resident set size (kbytes): 0
#         Major (requiring I/O) page faults: 0            ← 磁盘页故障
#         Minor (reclaiming a frame) page faults: 180
#         Voluntary context switches: 5                   ← 主动切出（I/O等待等）
#         Involuntary context switches: 25                ← 被动切出（时间片到）
#         Swaps: 0
#         File system inputs: 20480                       ← 文件系统读 (KB)
#         File system outputs: 10240                      ← 文件系统写 (KB)
#         Socket messages sent: 0
#         Socket messages received: 0
#         Signals delivered: 0
#         Page size (bytes): 4096
#         Exit status: 0

# 关键解读：
# - Voluntary context switches 高 → 频繁 I/O 等待
# - Involuntary context switches 高 → CPU 竞争激烈
# - Major page faults > 0 → 发生了磁盘 swap / 文件未缓存
# - Maximum resident set size → 实际物理内存峰值
```

```bash
# ── 自定义 time 格式速查 ──
TIMEFORMAT=$'\nreal: %R  user: %U  sys: %S  cpu: %P%%\n'
# %R: real (elapsed) 时间
# %U: user mode CPU 秒数
# %S: kernel mode CPU 秒数
# %P: CPU 使用百分比 = (%U + %S) / %R * 100
# %E: 已用时间（格式 [hours:]minutes:seconds）
# %I: 文件系统输入次数
# %O: 文件系统输出次数
# %r: socket 消息接收次数
# %s: socket 消息发送次数
# %K: 平均总内存使用（KB）
# %M: 最大常驻内存（KB）
# %W: 被换出到 swap 的次数
# %c: 上下文切换次数（非自愿）
# %w: 上下文切换次数（自愿）
# %F: 缺页次数（major）
# %R: 缺页次数（minor）
```

## 10.2 strace 统计系统调用

`strace` 能精确告诉你程序发起了多少次、哪些类型的系统调用，是定位性能瓶颈的利器。

```bash
# ── 基础用法：统计系统调用 ──
strace -c bash -c 'for i in {1..1000}; do echo "$i"; done'
# 输出（bash echo 内置，无 fork）：
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- --------
#  85.23    0.003248           3      1000           write
#  10.45    0.000398           3       120           read
#   4.32    0.000164           0      1000           rt_sigprocmask
# ...（无 clone/execve）

# 对比：使用外部命令
strace -c bash -c 'for i in {1..1000}; do /bin/echo "$i"; done 2>&1'
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- --------
#  60.23    0.152300         152      1000           clone    ← fork
#  15.32    0.038750          38      1000           execve   ← exec
#  12.45    0.031500          31      1000           wait4
# ...（共 ~5000 次系统调用，内置版只有 ~2200 次）
```

```bash
# ── strace 高级用法 ──

# 1. 只跟踪特定系统调用类型
strace -e trace=clone,execve -c bash script.sh
# 只看进程创建相关的系统调用

# 2. 查看 fork/exec 的耗时分布
strace -T -e trace=clone,execve bash script.sh
# -T: 显示每个系统调用的耗时

# 3. 跟踪子进程
strace -f -c bash script.sh
# -f: 同时跟踪所有子进程

# 4. 按时间戳显示
strace -tt -T -o trace.log -f bash script.sh
# -tt: 微秒级时间戳
# -o trace.log: 输出到文件

# 5. 用 strace 分析的快速对照表
# | 指标                              | 问题诊断               |
# |-----------------------------------|----------------------|
# | clone/execve 调用数 > 预期         | 多余的 fork，检查循环  |
# | read/write 次数远大于数据量/缓冲区 | 缓冲区太小，增大缓冲区  |
# | open/close 重复调用同一文件        | 复用文件描述符         |
# | mmap/munmap 频繁                   | 临时文件或动态分配过多  |
# | futex 调用频繁                     | 锁竞争（多线程场景）   |
```

```bash
# ── 分析案例：定位循环内 fork ──
strace -e trace=clone,execve -c bash -c '
for f in *.txt; do
    basename "$f"          # 这里每次循环都 fork
done
'
# clone    1000  ← 露馅了！循环 1000 次就有 1000 次 fork

# 修复后：
strace -e trace=clone,execve -c bash -c '
for f in *.txt; do
    echo "${f##*/}"        # 参数展开，无 fork
done
'
# clone    0     ← 完美！
```

## 10.3 其他性能分析工具

```bash
# ── perf stat：CPU 级别的性能统计 ──
perf stat -d bash script.sh
# 输出：
#          5,234.12 msec task-clock                # CPU 利用率
#             1,234      context-switches          # 上下文切换
#                45      cpu-migrations            # CPU 迁移
#             2,345      page-faults               # 缺页
#    12,345,678,901      cycles                    # CPU 周期
#     8,765,432,109      instructions              # 指令数
#       345,678,901      branches                  # 分支指令
#         2,345,678      branch-misses             # 分支预测失败
#       123,456,789      L1-dcache-loads           # L1 缓存加载
#        12,345,678      L1-dcache-load-misses     # L1 缓存未命中

# ── /usr/bin/time -v（如前述） ──
# ── ps 快照分析 ──
# 实时查看脚本的内存和 CPU
while true; do
    ps -o pid,rss,vsz,pcpu,pmem,comm -p $$
    sleep 1
done

# ── vmstat / iostat：系统级瓶颈分析 ──
vmstat 1         # 每秒打印系统 CPU、内存、swap、I/O 状态
iostat -x 1      # 每秒打印磁盘 I/O 使用率、等待时间、吞吐量
```

## 10.4 性能分析决策树

```
你的脚本慢在哪里？
│
├─ CPU 使用率接近 100%（user+sys ≈ real）
│  │
│  ├─ strace -c 显示大量 clone/execve
│  │  └→ 循环内 fork → 替换为内置命令、减少子 Shell
│  │
│  ├─ strace -c 显示大量 read/write（与数据量不成比例）
│  │  └→ 缓冲区过小 → 增大 dd bs、使用 mapfile 批量读入
│  │
│  └─ perf stat 显示高 branch-misses 或 cache-misses
│     └→ 算法本身低效（虽然 Shell 层面较难优化）
│
├─ CPU 使用率很低（user+sys << real）
│  │
│  ├─ 大量 voluntary context switches
│  │  └→ I/O 等待 → 升级磁盘(SSD)、使用 page cache、异步 I/O
│  │
│  ├─ 大量网络等待（socket 相关 syscall 耗时高）
│  │  └→ 增加并发（xargs -P）、批量请求、连接复用
│  │
│  └─ sleep / 锁等待
│     └→ 检查是否有不必要的 sleep 或等待逻辑
│
└─ 进程被 OOM / swap
   └→ 内存占用过高 → 流式处理替代全量加载、限制并发数
```

# 11. 基准测试案例

## 11.1 测试环境与说明

所有测试在同台机器上运行，取 5 次测试的中位数。

- CPU: Intel Xeon E5-2670 v3 @ 2.30GHz (12C/24T)
- 内存: 64GB DDR4
- 磁盘: Samsung 860 EVO SSD (SATA)
- OS: Ubuntu 22.04 LTS, Kernel 5.15
- Bash: 5.1.16

## 11.2 案例一：for 循环 10 万次 echo

```bash
# ── 测试脚本 ──

# 方案 A：外部 /bin/echo
time (for i in {1..100000}; do /bin/echo "$i" > /dev/null; done)
# real: 21.3s  user: 8.4s  sys: 14.2s  fork: 100000 次

# 方案 B：bash 内置 echo
time (for i in {1..100000}; do echo "$i" > /dev/null; done)
# real: 0.28s  user: 0.25s  sys: 0.03s  fork: 0 次

# 方案 C：bash printf
time (for i in {1..100000}; do printf '%d\n' "$i" > /dev/null; done)
# real: 0.30s  user: 0.27s  sys: 0.03s  fork: 0 次
# printf 与 echo 开销几乎相同
```

**结果：** 内置命令比外部命令快约 **76 倍**（21.3s vs 0.28s）

## 11.3 案例二：basename vs 参数展开

```bash
# ── 10 万次 basename/dirname vs 参数展开 ──

# 方案 A：外部命令
time (
    for i in {1..100000}; do
        path="/a/b/c/d/e/f/g/file_${i}.txt"
        n=$(basename "$path")
        d=$(dirname "$path")
    done
)
# real: 48.2s  fork: 200000 次（basename + dirname 各 10 万）

# 方案 B：参数展开
time (
    for i in {1..100000}; do
        path="/a/b/c/d/e/f/g/file_${i}.txt"
        n="${path##*/}"
        d="${path%/*}"
    done
)
# real: 0.31s  fork: 0 次
```

**结果：** 参数展开比外部命令快约 **155 倍**（48.2s vs 0.31s）

## 11.4 案例三：循环内命令替换

```bash
# ── 场景：读取 10000 个文件的行数 ──
# 先用脚本生成 10000 个测试文件（每个 100 行）
for i in {1..10000}; do seq 1 100 > "file_$i.txt"; done

# 方案 A：循环内 wc -l（最差）
time (
    total=0
    for f in file_*.txt; do
        lines=$(wc -l < "$f")
        ((total += lines))
    done
    echo "$total"
)
# real: 23.4s  fork: 10000 次

# 方案 B：一次 wc -l（流式处理）
time (
    total=0
    while read -r lines _; do
        ((total += lines))
    done < <(wc -l file_*.txt)
    echo "$total"
)
# real: 0.09s  fork: 1 次

# 方案 C：用 awk 直接计算
time (
    wc -l file_*.txt | awk '{sum += $1} END {print sum}'
)
# real: 0.08s  fork: 2 次（wc + awk）
```

**结果：** 批量处理比逐个 fork 快约 **290 倍**（23.4s vs 0.08s）

## 11.5 案例四：字符串大小写转换

```bash
# ── 100MB 文本大小写转换 ──
# 生成 100MB 测试文件
LC_ALL=C tr -dc 'a-zA-Z0-9 \n' < /dev/urandom | head -c 104857600 > 100mb.txt

# 方案 A：tr
time (LC_ALL=C tr 'a-z' 'A-Z' < 100mb.txt > /dev/null)
# real: 0.28s  user: 0.22s  sys: 0.06s

# 方案 B：sed
time (LC_ALL=C sed 's/.*/\U&/' 100mb.txt > /dev/null)
# real: 1.52s  user: 1.45s  sys: 0.07s

# 方案 C：awk
time (LC_ALL=C awk '{print toupper($0)}' 100mb.txt > /dev/null)
# real: 2.80s  user: 2.72s  sys: 0.08s

# 方案 D：bash 内置（逐行 ${line^^}）
time (
    while IFS= read -r line; do
        echo "${line^^}"
    done < 100mb.txt > /dev/null
)
# real: 12.5s  user: 11.2s  sys: 1.3s
```

**结果：** `tr` 最快。对于大文本的字符级操作，bash 内置反而更慢——因为 `while read` 逐行循环本身就是瓶颈。**正确选择工具比盲目用内置更重要。**

## 11.6 案例五：sort 的 LC_ALL=C 效果

```bash
# 生成 1000 万行乱序数据（约 500MB）
seq 1 10000000 | shuf > 10m_lines.txt

# 方案 A：默认 locale
time sort 10m_lines.txt > /dev/null
# real: 38.5s  user: 35.2s  sys: 3.3s

# 方案 B：LC_ALL=C
time LC_ALL=C sort 10m_lines.txt > /dev/null
# real: 8.2s  user: 7.5s  sys: 0.7s
```

**结果：** `LC_ALL=C` 使排序提速约 **4.7 倍**（38.5s vs 8.2s）

## 11.7 案例六：grep 多次管道 vs 单次 awk

```bash
# 生成 1000 万行日志
for i in {1..10000000}; do
    echo "$(date +%s) INFO some message here with random data $RANDOM"
done > large.log

# 其实用 awk 生成更快：
awk 'BEGIN {srand(); for(i=1;i<=10000000;i++)
    printf "%d INFO some message here with random data %d\n",
    systime(), int(rand()*65535)}' > large.log

# 方案 A：多次 grep 管道
time (grep "INFO" large.log | grep -c "data" > /dev/null)
# real: 4.2s  processes: 2

# 方案 B：单次 awk
time (awk '/INFO/ && /data/ {count++} END {print count}' large.log > /dev/null)
# real: 3.8s  processes: 1

# 此处差异不大（瓶颈在 I/O 而非进程数），但在 CPU 密集型场景
#（如复杂正则、大量短匹配）中差异会显著放大
```

## 11.8 案例七：并行处理 100 个任务

```bash
# ── 模拟：100 个任务，每个耗时 1 秒 ──

# 方案 A：串行
time (
    for i in {1..100}; do
        sleep 1
    done
)
# real: 100.5s

# 方案 B：xargs -P 并行
time (printf '%s\n' {1..100} | xargs -P 10 -n 1 sleep 1)
# real: 10.2s   ← 10 并发，10 波

# 方案 C：GNU parallel
time (parallel -j 10 sleep 1 ::: {1..100})
# real: 10.1s

# 方案 D：& + wait 手动并行
time (
    for i in {1..10}; do
        for j in {1..10}; do
            sleep 1 &
        done
        wait
    done
)
# real: 10.1s
```

**结果：** 并行比串行快约 **10 倍**（与并发数 `-j 10` 成正比）

## 11.9 案例八：批量 mapfile vs 逐行 read

```bash
# ── 250 万行文件（约 50MB） ──
# 生成测试文件
seq 1 2500000 > 2.5m_lines.txt

# 方案 A：逐行 read
time (
    count=0
    while IFS= read -r line; do
        ((count++))
    done < 2.5m_lines.txt
    echo "count=$count"
)
# real: 0.72s  user: 0.58s  sys: 0.14s

# 方案 B：mapfile 批量读入（10000 行一批）
time (
    count=0
    while mapfile -t -n 10000 batch && ((${#batch[@]} > 0)); do
        ((count += ${#batch[@]}))
    done < 2.5m_lines.txt
    echo "count=$count"
)
# real: 0.18s  user: 0.15s  sys: 0.03s

# 方案 C：wc -l（终极方案，不是 bash 循环）
time wc -l 2.5m_lines.txt
# real: 0.03s
```

**结果：** `mapfile` 批量读取比逐行 `read` 快约 **4 倍**。但最简单计行数直接用 `wc -l`，比任何 bash 循环都快 1+ 倍——**正确使用工具比优化循环更重要。**

## 11.10 综合优化前后对比

```bash
# ── 场景：读取 5GB 日志文件，统计每个 HTTP 状态码的出现次数 ──
# 日志格式：... [timestamp] "METHOD /path HTTP/1.1" STATUS SIZE ...

# ── 优化前：典型的"新人写法" ──
cat access.log \
    | awk '{print $9}' \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -10
# 进程: 6 个（cat, awk, sort, uniq, sort, head）
# 临时文件: sort 产生的外部归并临时文件（可能 2~3 倍于输入大小）

# ── 优化后 ──
LC_ALL=C awk '
{ count[$9]++ }
END {
    PROCINFO["sorted_in"] = "@val_num_desc"
    for (s in count) {
        if (++i > 10) break
        print s, count[s]
    }
}' access.log
# 进程: 1 个
# 临时文件: 0
# 性能提升: 减少 5 个无关进程, 1 次外部排序
```

**综合基准对比（5GB 日志，约 5000 万行）：**

| 指标 | 优化前 | 优化后 | 提升 |
|------|--------|--------|------|
| 进程数 | 6 | 1 | 6x |
| 磁盘临时文件 | sort 产生 ~10GB | 0 | 消除 |
| wall clock | 4min 23s | 1min 08s | **3.8x** |
| CPU 时间 (user+sys) | 185s | 62s | 3.0x |
| 峰值内存 | 2.1GB (sort) | 380MB (awk) | 5.5x |
| 上下文切换 | 128,000 | 12,000 | 10.6x |

## 11.11 优化检查清单

- [ ] 循环内是否有命令替换 `$(...)`？ → 移到循环外或批量处理
- [ ] 循环内是否有外部命令（`sed`/`awk`/`grep`/`wc`/`basename`...）？ → 用 bash 内置或参数展开替代
- [ ] 是否用 `for i in $(cat file)` 遍历文件？ → 改用 `while IFS= read -r`
- [ ] 是否链式管道多个 `grep`/`sed`？ → 合并为一个 `awk` 调用
- [ ] 是否有重复打开同一文件？ → 使用文件描述符 `exec N>file`
- [ ] `sort` 处理的数据是纯 ASCII？ → 加 `LC_ALL=C`
- [ ] 大文件是否全量读入内存？ → 改用流式处理
- [ ] 独立任务是否可并行？ → 使用 `xargs -P` 或 `parallel`
- [ ] 是否使用 `seq`？ → 改用大括号展开 `{1..N}` 或 C 风格 `for ((...))`
- [ ] 是否逐字符/逐字节处理字符串？ → 转为批量替换
- [ ] 是否有多次 `wc -l < file`？ → 一次 `wc -l` 批量处理所有文件
- [ ] 管道中的 `while read` 是否修改外层变量？ → 改用进程替换 `< <(cmd)` 或 `lastpipe`
- [ ] 是否重复使用 `expr`？ → 改用算术展开 `$((...))`
