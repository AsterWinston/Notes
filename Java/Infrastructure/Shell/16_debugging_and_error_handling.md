
# 1. set 命令详解

`set` 是 Bash 最核心的运行时配置命令，通过选项开关控制 Shell 的执行行为。合理组合这些选项是写出健壮脚本的第一步。

## 1.1 选项速查

| 选项 | 长选项 | 作用 | 风险等级 |
|------|--------|------|----------|
| `-x` | `-o xtrace` | 执行前打印每条命令（调试输出） | 安全 |
| `-e` | `-o errexit` | 任何命令返回非零时立即退出 | **高**（有陷阱） |
| `-u` | `-o nounset` | 引用未定义变量时退出 | 中 |
| `-o pipefail` | 无短选项 | 管道中任一命令失败则整体失败 | 低 |
| `-v` | `-o verbose` | 读取时打印每行（展开前） | 安全 |
| `-n` | `-o noexec` | 仅检查语法，不执行 | 安全 |
| `-C` | `-o noclobber` | 禁止 `>` 覆盖已存在文件 | 低 |
| `-f` | `-o noglob` | 禁用通配符展开 | 中 |

## 1.2 set -x：命令追踪

`-x` 在执行每条命令前，将展开后的命令打印到 stderr，前缀由 `PS4`（默认 `+`）控制。

**原理：** Bash 在解析完命令、完成所有展开后，在 `execve()` 之前调用 DEBUG trap 输出。它不是简单的 echo，而是走 Bash 内部 `xtrace_fd`（默认 stderr，可通过 `BASH_XTRACEFD` 修改）。

```bash
#!/bin/bash
set -x

name="world"
echo "Hello, $name"
# 输出：
# + name=world
# + echo 'Hello, world'
# Hello, world

# 局部开关
set -x
critical_section
set +x
```

**重定向 xtrace 到文件：**

```bash
#!/bin/bash
exec 5>debug.log
BASH_XTRACEFD=5

set -x
echo "this goes to debug.log"
set +x
```

## 1.3 set -e：错误即退出

`-e` 使得任何命令返回非零退出码时 Shell 立即退出（类似"快速失败"）。

**原理：** Bash 在每个简单命令执行后检查 `$?`，若非零且命令不在"豁免列表"中，则调用 `exit`。

**-e 的豁免场景（不触发退出）：**

- `if cmd; then ...` 中的条件部分
- `while cmd; do ...` 中的条件部分
- `cmd || true` 或 `cmd || echo "fallback"`
- `cmd && other_cmd` 中左侧命令
- 管道中最右侧命令失败（除非启用 `pipefail`）
- 在 `! cmd` 取反后

```bash
#!/bin/bash
set -e

# 这些不会触发 -e 退出：
if ! grep -q "pattern" file.txt; then
    echo "pattern not found"
fi

while ! ping -c1 8.8.8.8 >/dev/null 2>&1; do
    sleep 1
done

# 这些会触发 -e 退出：
grep "nothing" /nonexistent          # 文件不存在 → 退出
cd /tmp && rm -f important           # cd 失败 → 退出
```

**-e 的经典陷阱（命令替换中的失败）：**

```bash
#!/bin/bash
set -e

# 陷阱 1：命令替换中的失败不触发 -e（Bash 4.4 之前）
result=$(grep "pattern" nonexistent)  # 不退出！$result 为空，但不报错
echo "result: $result"

# 陷阱 2：子 Shell 中 -e 丢失
( set -e; false; echo "still running" )  # 可能因 Bash 版本不同而异

# 陷阱 3：函数调用不触发
foo() { false; echo "函数内继续"; }
foo  # -e 在函数末尾 false 可能不生效（Bash 4.0 之前）

# 安全做法：显式检查
result=$(grep "pattern" nonexistent) || true
result=$(grep "pattern" nonexistent 2>&1) || exit 1
```

## 1.4 set -u：未定义变量报错

```bash
#!/bin/bash
set -u

echo "$UNDEFINED_VAR"       # 报错：UNDEFINED_VAR: unbound variable

# 允许未定义变量的安全写法
echo "${UNDEFINED_VAR:-default}"   # 使用默认值
echo "${UNDEFINED_VAR:=default}"   # 设置默认值（同时赋值）
echo "${UNDEFINED_VAR:?error}"     # 显式报错

# 检查变量是否定义
if [ -z "${VAR+x}" ]; then
    echo "VAR is unset"
fi
```

**-u 与位置参数：**

```bash
#!/bin/bash
set -u

# $1 可能未定义，安全做法：
first="${1:-default}"
echo "$first"

# 检查参数数量
if [ $# -lt 2 ]; then
    echo "用法: $0 <arg1> <arg2>" >&2
    exit 1
fi
```

## 1.5 set -o pipefail：管道全面检查

默认情况下，管道的退出码是最后一个命令的退出码，前面的失败会被掩盖。

```bash
#!/bin/bash
set -e

# 没有 pipefail：不退出（因为 cat 成功）
grep "pattern" hugefile.txt | cat > /dev/null
# grep 失败被掩盖了

set -o pipefail

# 有 pipefail：grep 失败 → 退出
grep "pattern" hugefile.txt | cat > /dev/null
```

**获取管道中所有退出码：**

```bash
set -o pipefail

cmd1 | cmd2 | cmd3
echo "管道整体退出码: $?"
echo "各命令退出码: ${PIPESTATUS[@]}"
# 输出：管道整体退出码: 1
# 各命令退出码: 1 0 0
```

## 1.6 set -v：源代码回显

与 `-x` 不同，`-v` 打印的是**展开前**的原始代码行，不做展开。

```bash
#!/bin/bash
set -v
name="world"
echo "Hello, $name"
# 输出：
# name="world"
# echo "Hello, $name"   ← 未展开
# Hello, world
```

**-x 与 -v 的区别：**

```bash
#!/bin/bash
set -x
set -v  # 先 -v 再 -x

name=$USER
echo "Hello, $name"

# -v 输出（原样）：
# name=$USER
# echo "Hello, $name"

# -x 输出（已展开）：
# + name=root
# + echo 'Hello, root'
```

## 1.7 set -n：语法检查

不执行脚本，仅检查语法错误。适合在部署前验证脚本语法。

```bash
# 语法检查
bash -n script.sh
# 或
set -n
source script.sh  # 只解析不执行
```

## 1.8 推荐组合与原理

**最强健的组合——写在脚本开头：**

```bash
#!/bin/bash
set -euo pipefail
# 或更长的显式写法：
set -o errexit
set -o nounset
set -o pipefail
# 打印追踪
set -x
```

**各选项的交互关系：**

```
set -e  + set -o pipefail  → 管道中任一命令失败就退出
set -e  + set -u           → 未定义变量导致退出
set -x  + set -e           → 在报错前看到最后执行的命令
```

**何时不用 `-e`：**

- 脚本中有大量需要检查返回值的操作（grep 常返回非零但并非错误）
- 函数库文件（lib 不应控制执行流程，应返回错误码）
- 交互式 Shell 配置文件（`.bashrc` 中启用 `-e` 会导致终端异常退出）

```bash
# grep 的经典尴尬：
set -e
grep "rare_pattern" file.txt  # 没找到 → 脚本退出！
# 正确做法：
grep "rare_pattern" file.txt || true
# 或局部关闭
set +e
grep "rare_pattern" file.txt
status=$?
set -e
```

## 1.9 运行时感知当前 set 状态

```bash
# 查看所有 set 选项
echo "$-"
# 输出如：himBHs（包含了 e u x 等当前启用的选项）

# 检查特定选项是否开启
shopt -qo pipefail && echo "pipefail 已开启"

# 保存/恢复 set 状态
old_opts=$(set +o)  # 保存所有选项
set -euo pipefail   # 设置新选项
# ... 执行操作 ...
eval "$old_opts"    # 恢复
```

# 2. PS4 自定义调试输出

`PS4` 是 `set -x` 输出行的前缀，默认值为 `+`。通过自定义 `PS4` 可以嵌入时间戳、行号、函数名等上下文信息，极大提升调试效率。

## 2.1 PS4 变量展开机制

`PS4` 在**每次**输出 xtrace 行前展开一次，因此可以嵌入动态变量。展开规则与 `PS1` 相同（左提示符），支持反斜杠转义序列。

```bash
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'

# 原始 xtrace 输出：
# + cd /tmp

# 增强后输出：
# +(script.sh:42): main(): cd /tmp
```

## 2.2 嵌入时间戳

```bash
# 使用 \t 和 \D{} 格式化时间
export PS4='+\t [\D{%Y-%m-%d %H:%M:%S}] '

# 或使用 $(date)（性能开销大，不推荐频繁场景）
export PS4='+[$(date "+%H:%M:%S")] '

# 使用内置变量 SECONDS（脚本已运行秒数）
export PS4='+[${SECONDS}s] '

# 高精度：Bash 5.0+ 支持 EPOCHREALTIME
export PS4='+[${EPOCHREALTIME}] '
```

## 2.3 完整的生产级 PS4 配置

```bash
#!/bin/bash

# 生产级调试前缀：时间戳 + 脚本名 + 行号 + 函数名 + 层级缩进
export PS4='+[$(date "+%T")][${BASH_SOURCE##*/}:${LINENO}]${FUNCNAME[0]:+[${FUNCNAME[0]}]}: '

# 或者基于 SECONDS 的低开销版本（不需要额外的 date fork）
export PS4='+[${SECONDS}s][${BASH_SOURCE##*/}:${LINENO}]${FUNCNAME[0]:+[${FUNCNAME[0]}]}: '

# 可选：产生缩进效果（手动维护深度变量）
export PS4='+${BASH_XTRACE_DEPTH:+${BASH_XTRACE_DEPTH# }} '

# 使用示例：在函数入口增加深度
xtrace_depth=0

increment_depth() {
    local spaces=""
    for ((i=0; i<xtrace_depth; i++)); do spaces+="  "; done
    BASH_XTRACE_DEPTH="$spaces"
}

my_func() {
    ((xtrace_depth++))
    increment_depth
    # ... 函数体 ...
    ((xtrace_depth--))
}
```

## 2.4 BASH_XTRACEFD 定向输出

```bash
#!/bin/bash
exec 9> /var/log/myscript.xtrace.log
BASH_XTRACEFD=9

export PS4='+${SECONDS}s: '
set -x

# xtrace 输出写入 log 文件而非终端
echo "Hello"
```

# 3. DEBUG trap 高级调试

`DEBUG` trap 是 Bash 提供的最强大的调试钩子——在**每条简单命令执行前**触发，允许插入自定义追踪逻辑。与 `set -x` 不同，它给予完全的控制权。

## 3.1 基础用法：逐条打印命令

```bash
#!/bin/bash

trap 'echo "[DEBUG] 即将执行: $BASH_COMMAND"' DEBUG

name="Alice"
echo "Hello, $name"
date
ls /nonexistent

trap - DEBUG  # 取消 trap
```

**输出：**
```
[DEBUG] 即将执行: name="Alice"
[DEBUG] 即将执行: echo "Hello, $name"
Hello, Alice
[DEBUG] 即将执行: date
Sun May 17 10:30:00 CST 2026
[DEBUG] 即将执行: ls /nonexistent
ls: cannot access '/nonexistent': No such file or directory
```

## 3.2 BASH_COMMAND 变量

`BASH_COMMAND` 包含当前即将执行的命令原文（展开前）。在 DEBUG trap 中使用，而在 ERR trap 中则包含失败的命令。

```bash
#!/bin/bash

trap 'echo "[TRACE] 行${LINENO}: $BASH_COMMAND"' DEBUG

for i in 1 2 3; do
    echo "i=$i"
done
# 注意：for/while/if 的条件部分不会触发 DEBUG trap
```

## 3.3 条件断点

```bash
#!/bin/bash

# 逻辑：启用调试追踪
debug_enabled=true   # 设 false 关闭

trap '
    if [[ "$debug_enabled" == "true" ]]; then
        echo "|-- [#${LINENO}] $BASH_COMMAND" >&2
    fi
' DEBUG

x=1
y=2
result=$((x + y))
echo "result = $result"

debug_enabled=false
# 此处之后不再打印 DEBUG 信息
echo "this is not traced"
```

## 3.4 调用栈追踪

```bash
#!/bin/bash

trap 'print_callstack' DEBUG

print_callstack() {
    local depth=${#FUNCNAME[@]}
    # 跳过 trap 自身
    local frame
    echo -n "[TRACE] "
    for ((frame=1; frame<depth; frame++)); do
        echo -n "${FUNCNAME[$frame]}(${BASH_LINENO[$frame-1]}) > "
    done
    echo "$BASH_COMMAND"
}

foo() {
    bar() {
        baz() {
            echo "deep in callstack"
        }
        baz
    }
    bar
}

foo
```

**输出：**
```
[TRACE] main > foo(30) > bar(24) > baz(21) > echo "deep in callstack"
```

## 3.5 性能分析与耗时统计

```bash
#!/bin/bash

profiling_start=0
profiling_results=()

trap '
    if [[ $profiling_start -gt 0 ]]; then
        local now elapsed
        now=$(date +%s%N)
        elapsed=$(( (now - profiling_start) / 1000000 ))  # 转毫秒
        if [[ $elapsed -gt 100 ]]; then  # 只记录 >100ms 的命令
            profiling_results+=("[${elapsed}ms] $BASH_LAST_COMMAND")
        fi
    fi
    profiling_start=$(date +%s%N)
' DEBUG

# 重置第一次触发
profiling_start=0

sleep 0.5
echo "Hello"
sleep 0.3

trap - DEBUG

echo "=== 慢命令报告 ==="
printf '%s\n' "${profiling_results[@]}"
```

## 3.6 DEBUG trap 的限制与注意事项

- **不会对 `for`/`while`/`until`/`case`/`if` 的条件部分触发**
- **不会对函数定义体触发**（只在函数调用内的命令触发）
- **会递归触发**：trap 处理函数自身中的命令也会触发 DEBUG trap，需要做防护
- **对 `declare`/`typeset`/`local` 不触发**（Bash 3.x）
- **trap 函数中的命令输出会混入正常输出**——务必重定向到 stderr

```bash
# 递归防护：
trap '
    if [[ $BASH_COMMAND != *"trap"* && $BASH_COMMAND != *"print_trace"* ]]; then
        print_trace
    fi
' DEBUG
```

# 4. ERR trap 错误捕获

`ERR` trap 在命令返回非零退出码时触发（与 `set -e` 的判断逻辑一致），是构建健壮错误处理体系的核心。

## 4.1 基础错误捕获

```bash
#!/bin/bash

on_error() {
    echo "!!! 错误发生在第 $1 行，命令: $BASH_COMMAND" >&2
    echo "!!! 退出码: $?" >&2
}
trap 'on_error $LINENO' ERR

false
ls /nonexistent
```

## 4.2 完整调用栈打印

```bash
#!/bin/bash

print_error_stack() {
    local exit_code=$?
    local frame=0

    echo "========================================" >&2
    echo "  错误报告" >&2
    echo "========================================" >&2
    echo "  退出码: ${exit_code}" >&2
    echo "  失败命令: ${BASH_COMMAND}" >&2
    echo "  脚本: ${BASH_SOURCE[1]}" >&2
    echo "  行号: ${BASH_LINENO[0]}" >&2
    echo "----------------------------------------" >&2
    echo "  调用栈:" >&2

    # FUNCNAME[0] 是 trap 函数自身，[1] 是从脚本入口开始
    for ((frame=1; frame<${#FUNCNAME[@]}; frame++)); do
        echo "    ${FUNCNAME[$frame]}()" >&2
        echo "      at ${BASH_SOURCE[$frame]}:${BASH_LINENO[$frame-1]}" >&2
    done
    echo "========================================" >&2
}

trap print_error_stack ERR

# 测试
level1() {
    level2() {
        level3() {
            false  # 故意失败
        }
        level3
    }
    level2
}

level1
```

**输出：**
```
========================================
  错误报告
========================================
  退出码: 1
  失败命令: false
  脚本: test.sh
  行号: 46
----------------------------------------
  调用栈:
    level3()
      at test.sh:46
    level2()
      at test.sh:48
    level1()
      at test.sh:50
    main()
      at test.sh:53
========================================
```

## 4.3 记录错误到日志文件

```bash
#!/bin/bash

ERROR_LOG="/var/log/myscript_error.log"
touch "$ERROR_LOG" 2>/dev/null || ERROR_LOG="./error.log"

log_error() {
    local exit_code=$?
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    {
        echo "[${timestamp}] ====== 错误发生 ======"
        echo "  退出码: ${exit_code}"
        echo "  命令: ${BASH_COMMAND}"
        echo "  脚本: ${BASH_SOURCE[1]##*/}:${BASH_LINENO[0]}"
        echo "  PID: $$"
        echo "  调用栈:"
        local frame
        for ((frame=1; frame<${#FUNCNAME[@]}; frame++)); do
            echo "    ${FUNCNAME[$frame]}() in ${BASH_SOURCE[$frame]}:${BASH_LINENO[$frame-1]}"
        done
        echo "  环境信息:"
        echo "    PWD=$PWD"
        echo "    USER=$USER"
        echo "    HOSTNAME=${HOSTNAME:-$(hostname)}"
        echo ""
    } >> "$ERROR_LOG"
}

trap log_error ERR
```

## 4.4 错误处理策略：分级响应

```bash
#!/bin/bash

# 错误严重程度
readonly E_WARNING=1    # 警告：继续运行
readonly E_ERROR=2      # 普通错误：退出当前函数
readonly E_FATAL=3      # 致命错误：退出脚本

# 当前上下文（函数级）
current_function="main"
error_mode="fatal"  # fatal | graceful | warn

handle_error() {
    local exit_code=$?
    local cmd="$BASH_COMMAND"
    local line="${BASH_LINENO[0]}"
    local func="${FUNCNAME[1]:-main}"

    case "$error_mode" in
        fatal)
            echo "[FATAL] 函数 '$func' 第 $line 行失败 (退出码: $exit_code)" >&2
            echo "[FATAL] 失败命令: $cmd" >&2
            exit $exit_code
            ;;
        graceful)
            echo "[ERROR] 函数 '$func' 第 $line 行失败，继续执行" >&2
            ;;
        warn)
            echo "[WARN] 函数 '$func' 第 $line 行返回非零，忽略" >&2
            return 0  # 吞掉错误
            ;;
    esac
}

trap handle_error ERR

# 使用：
error_mode=warn
grep "maybe_not_exist" file.txt || true  # 设为 warn 时不会退出

error_mode=fatal
rm /important/file  # 失败时立即退出
```

## 4.5 ERR trap 的继承与覆盖

```bash
#!/bin/bash

# 全局 ERR trap
trap 'echo "全局 ERR: $BASH_COMMAND 失败"' ERR

func_a() {
    # 子函数可以覆盖 ERR trap
    trap 'echo "func_a 的 ERR trap"' ERR
    false
}

func_b() {
    # 继承全局 trap
    false
}

func_a  # 使用自己的 trap
func_b  # 使用全局 trap
```

# 5. bash -x 与命令行调试

## 5.1 基本用法

```bash
# 整个脚本开启 xtrace
bash -x script.sh
bash -o xtrace script.sh

# 输出到文件
bash -x script.sh 2>debug.log

# 配合 -v（先打印原代码，再打印展开后的命令）
bash -xv script.sh

# 交互式开启
bash
set -x
# ... 逐条执行并观察 ...
```

## 5.2 局部调试：在脚本中嵌入

```bash
#!/bin/bash

# 全局不开 -x，只在关键区域开启
echo "=== 开始关键区域调试 ==="
set -x

result=$(curl -s https://api.example.com/data)
parsed=$(echo "$result" | jq '.items[]')

set +x
echo "=== 结束关键区域调试 ==="
```

## 5.3 条件触发 xtrace

```bash
#!/bin/bash

# 用函数封装条件 xtrace
xtrace_on() {
    if [[ "${DEBUG:-0}" == "1" ]]; then
        set -x
    fi
}

xtrace_off() {
    if [[ "${DEBUG:-0}" == "1" ]]; then
        set +x
    fi
}

# 使用：
DEBUG=1 ./script.sh   # 开启调试
./script.sh           # 正常执行
```

## 5.4 嵌套调用的缩进效果

`set -x` 在函数嵌套调用时不会自动缩进，但可以通过 `PS4` 结合深度计数器实现：

```bash
#!/bin/bash

# 利用 FUNCNAME 数组长度计算调用深度
export PS4='+$(printf "%$(( ${#FUNCNAME[*]} * 2 ))s")${BASH_SOURCE##*/}:${LINENO}: '

set -x

outer_func() {
    middle_func() {
        inner_func() {
            echo "最深层"
        }
        inner_func
    }
    middle_func
}

outer_func
```

**输出：**
```
+script.sh:14: outer_func
  +script.sh:9: middle_func
    +script.sh:6: inner_func
      +script.sh:7: echo '最深层'
```

## 5.5 Bash 4.0+ 的 BASH_XTRACEFD 实战

```bash
#!/bin/bash

# 将 xtrace 重定向到独立的文件描述符
# 适用于：标准错误已被业务逻辑占用时

exec 11>/tmp/xtrace.log
BASH_XTRACEFD=11

# 标准错误保持正常输出
exec 3>&2  # 保存原始 stderr
exec 2>/var/log/myscript.log  # 业务日志走 stderr

set -x
# 业务逻辑...
echo "this goes to error log" >&2
# xtrace 输出到 /tmp/xtrace.log
set +x

# 还原
exec 2>&3
```

# 6. 错误处理模式

Shell 脚本没有异常机制（try/catch），错误处理依赖于返回值检查、短路运算符、和 trap 的组合。

## 6.1 返回值检查（基础模式）

```bash
# 模式 A：if 检查
if ! cmd; then
    echo "cmd 失败" >&2
    exit 1
fi

# 模式 B：短路运算符
cmd || { echo "cmd 失败" >&2; exit 1; }

# 模式 C：保留退出码
cmd
status=$?
if [ $status -ne 0 ]; then
    echo "cmd 失败，退出码: $status" >&2
    exit $status
fi
```

## 6.2 || { ... } 模式

```bash
#!/bin/bash

# 单行（注意 { } 内部的空格要求）
cd /some/dir || { echo "无法进入目录" >&2; exit 1; }

# 多行——必须在 { 后换行
grep "pattern" file.txt || {
    echo "模式未找到或文件不存在" >&2
    echo "请检查 file.txt" >&2
    exit 2
}

# 函数形式——更清晰
die() {
    echo "[ERROR] $*" >&2
    exit 1
}

cd /some/dir || die "无法进入目录"
grep "pattern" file.txt || die "模式未找到"
```

## 6.3 if ! cmd 模式

```bash
#!/bin/bash

# 适合处理"失败是正常分支"的场景
if ! grep -q "WARN" app.log; then
    echo "没有找到警告信息（这是好事）"
else
    echo "发现警告信息"
fi

# 或需要做复杂恢复
if ! mount /mnt/backup; then
    echo "挂载失败，尝试修复..."
    fsck /dev/sdb1
    if ! mount /mnt/backup; then
        echo "修复后仍无法挂载，退出" >&2
        exit 1
    fi
fi
```

## 6.4 trap ERR 模式（集中式错误处理）

```bash
#!/bin/bash

error_handler() {
    local exit_code=$?
    local line_no=$1
    local bash_lineno=$2
    local last_command=$3
    local func_trace=$4

    echo "====================" >&2
    echo "错误: 命令失败" >&2
    echo "  行号: ${line_no}" >&2
    echo "  命令: ${last_command}" >&2
    echo "  退出码: ${exit_code}" >&2
    echo "  调用栈: ${func_trace[*]}" >&2
    echo "====================" >&2
}

trap 'error_handler ${LINENO} ${BASH_LINENO[*]} "${BASH_COMMAND}" "${FUNCNAME[*]}"' ERR
```

## 6.5 三种模式对比

| 维度 | `|| { ... }` | `if ! cmd` | `trap ERR` |
|------|-------------|------------|------------|
| 粒度 | 每条命令 | 每条命令 | 全局 |
| 代码量 | 每行都有检查代码 | 每行都有检查代码 | 一处定义 |
| 灵活性 | 最灵活 | 灵活 | 统一策略 |
| 失败上下文 | 丰富 | 丰富 | 有限（BASH_COMMAND 等） |
| 可组合性 | 与 set -e 冲突 | 与 set -e 兼容 | 与 set -e 兼容 |
| 易漏 | 容易遗漏 | 容易遗漏 | 不会遗漏 |

**推荐组合策略：**

```bash
#!/bin/bash
set -euo pipefail

# 全局兜底
trap 'echo "未预期的错误，行: ${LINENO}" >&2' ERR

# 关键操作显式检查（覆盖全局策略）
critical_step() {
    # 这个函数的错误需要特殊处理
    trap 'handle_critical_error' ERR

    curl -s https://api.example.com || {
        echo "API 不可达，使用缓存" >&2
        return 1
    }

    trap - ERR  # 恢复全局 trap
}
critical_step
```

## 6.6 错误码约定

```bash
#!/bin/bash

# 定义业务错误码
readonly E_SUCCESS=0
readonly E_GENERAL=1
readonly E_USAGE=2        # 使用错误
readonly E_PERMISSION=77  # 权限不足
readonly E_NETWORK=100    # 网络错误
readonly E_FILE=101       # 文件错误
readonly E_CONFIG=102     # 配置错误
readonly E_TIMEOUT=103    # 超时

validate_config() {
    if [ ! -f "config.yaml" ]; then
        echo "配置文件不存在" >&2
        return $E_CONFIG
    fi
}

validate_config || exit $?
```

# 7. 防御性编程

防御性编程的核心信条：**永远不信任输入、环境、和外部依赖**。

## 7.1 检查命令返回值

```bash
#!/bin/bash

# 原则：每个可能失败的命令都显式检查

# 好的写法：
if ! result=$(curl -s --max-time 5 https://api.example.com); then
    echo "API 调用失败" >&2
    return 1
fi

# 更好的写法：同时检查 HTTP 状态码
response=$(curl -s -w "\n%{http_code}" https://api.example.com)
http_code=$(echo "$response" | tail -n1)
body=$(echo "$response" | sed '$d')
if [ "$http_code" != "200" ]; then
    echo "HTTP 错误: $http_code" >&2
    return 1
fi

# 危险的写法（使用 set -e 也不安全）：
result=$(curl https://api.example.com)  # 网络故障时 $result 为空，但脚本继续
```

## 7.2 检查文件存在与权限

```bash
#!/bin/bash

# 完整的文件前置检查
require_file() {
    local file="$1"
    local desc="${2:-文件}"

    if [ ! -e "$file" ]; then
        echo "错误: ${desc} '$file' 不存在" >&2
        exit 1
    fi
    if [ ! -r "$file" ]; then
        echo "错误: ${desc} '$file' 不可读" >&2
        exit 1
    fi
    if [ ! -s "$file" ]; then
        echo "警告: ${desc} '$file' 为空" >&2
    fi
}

# 目录检查
require_dir() {
    local dir="$1"
    local desc="${2:-目录}"

    if [ ! -d "$dir" ]; then
        echo "错误: ${desc} '$dir' 不是目录" >&2
        exit 1
    fi
    if [ ! -w "$dir" ]; then
        echo "错误: ${desc} '$dir' 不可写" >&2
        exit 1
    fi
}

# 使用：
require_file "/etc/myapp/config.yaml" "配置文件"
require_dir "/var/log/myapp" "日志目录"
```

## 7.3 检查变量非空

```bash
#!/bin/bash

# 方式 1：:? 语法（设置 -u 时必须用）
required_var="${REQUIRED_VAR:?请设置 REQUIRED_VAR 环境变量}"
required_var="${1:?缺少第一个参数}"

# 方式 2：主动检查
validate_not_empty() {
    local var_name="$1"
    local var_value="${!var_name}"  # 间接引用

    if [ -z "$var_value" ]; then
        printf "错误: 变量 '%s' 未设置或为空\n" "$var_name" >&2
        exit 1
    fi
}

validate_not_empty "DB_HOST"
validate_not_empty "DB_PASSWORD"

# 方式 3：组合检查
required_vars=("DB_HOST" "DB_PORT" "DB_USER" "DB_PASSWORD")
for var in "${required_vars[@]}"; do
    if [ -z "${!var:-}" ]; then
        echo "必需的环境变量未设置: $var" >&2
        exit 1
    fi
done
```

## 7.4 安全的临时文件

```bash
#!/bin/bash

# 永远不要手动拼接临时文件路径
# 错误：
TMP=/tmp/myscript_$$.tmp  # 可预测，可能被攻击

# 正确：
TMPFILE=$(mktemp)                    # 在 /tmp 下创建
TMPFILE=$(mktemp -p /var/tmp)        # 指定目录
TMPDIR=$(mktemp -d)                  # 临时目录

# 确保清理
cleanup() {
    local exit_code=$?
    rm -f "$TMPFILE"
    rm -rf "$TMPDIR"
    exit $exit_code
}
trap cleanup EXIT INT TERM
```

## 7.5 安全的路径处理

```bash
#!/bin/bash

# 原则：脚本化路径，避免相对路径陷阱

# 获取脚本所在目录（不受调用目录影响）
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# 只读根目录
readonly SCRIPT_DIR

# 切换到脚本目录
cd "$SCRIPT_DIR" || exit 1

# 安全的文件引用
CONFIG="${SCRIPT_DIR}/config/app.conf"
LIB="${SCRIPT_DIR}/lib/utils.sh"
```

## 7.6 输入验证白名单模式

```bash
#!/bin/bash

# 原则：白名单 > 黑名单

# 危险：黑名单模式
if [ "$action" != "delete" ] && [ "$action" != "drop" ]; then
    echo "安全的操作"
fi
# → 新的危险操作"truncate"漏过了

# 安全：白名单模式
valid_actions=("start" "stop" "restart" "status")

validate_action() {
    local action="$1"
    for valid in "${valid_actions[@]}"; do
        if [ "$action" = "$valid" ]; then
            return 0
        fi
    done
    echo "非法操作: $action，允许: ${valid_actions[*]}" >&2
    return 1
}

validate_action "$1" || exit 1
```

## 7.7 引号原则

```bash
#!/bin/bash

# 规则：变量始终用双引号包裹
# 例外：需要单词分割（罕见）或通配符展开（罕见）

# 危险：
if [ $var = "yes" ]; then ...  # var 为空时语法错误

# 正确：
if [ "$var" = "yes" ]; then ...

# 危险：
for file in $(ls *.txt); do    # 文件名含空格时失败
    process "$file"
done

# 正确：
for file in *.txt; do
    [ -e "$file" ] || continue
    process "$file"
done

# 危险：
cp $src $dst                    # 参数含空格时错误

# 正确：
cp "$src" "$dst"
```

# 8. shellcheck 静态分析

shellcheck 是 Shell 脚本的事实标准静态分析工具，能发现数百种常见错误和反模式，远胜于 `bash -n`。

## 8.1 安装

```bash
# Linux (apt)
sudo apt install shellcheck

# Linux (dnf)
sudo dnf install ShellCheck

# macOS
brew install shellcheck

# Docker（无需本地安装）
docker run --rm -v "$PWD:/mnt" koalaman/shellcheck:latest *.sh

# 在线使用
# https://www.shellcheck.net/
```

## 8.2 基本使用

```bash
# 检查单个文件
shellcheck script.sh

# 检查目录下所有脚本
shellcheck **/*.sh

# 指定 Shell 方言
shellcheck -s bash script.sh    # bash 模式
shellcheck -s sh script.sh      # POSIX 模式（检测 bashisms）
shellcheck -s dash script.sh    # dash 模式

# 输出格式
shellcheck -f json script.sh    # JSON 格式（适合 CI 解析）
shellcheck -f gcc script.sh     # GCC 兼容格式（编辑器集成）
shellcheck -f diff script.sh    # diff 格式

# 忽略特定规则
shellcheck -e SC2086 script.sh    # 忽略 SC2086（未引号变量）

# 严格模式
shellcheck -o all script.sh       # 启用所有可选检查
```

## 8.3 核心规则详解

| 规则 | 级别 | 说明 | 常见场景 |
|------|------|------|----------|
| SC2086 | warning | 变量未用双引号 | `[ $var = "yes" ]` → `[ "$var" = "yes" ]` |
| SC2046 | warning | 命令替换未引号 | `for i in $(ls)` → `for i in *.txt` |
| SC2164 | warning | cd 未检查返回值 | `cd /dir` → `cd /dir \|\| exit` |
| SC2068 | error | 数组未正确引用 | `$arr` → `"${arr[@]}"` |
| SC2155 | warning | declare/export 掩盖返回值 | `export FOO=$(cmd)` → 分开写 |
| SC2162 | warning | read 未用 -r | `read line` → `read -r line` |
| SC2181 | style | 用 $? 而非直接测试 | 用 `if cmd` 代替 `cmd; if [ $? -eq 0 ]` |
| SC2129 | style | 重定向顺序 | `>>file cmd` 行为怪异 |
| SC2001 | style | `${var//search/replace}` 更优 | 代替 `echo \| sed` |
| SC1113 | error | 分行不完整（非标准引号） | 混用 ' 和 `'` |
| SC2154 | warning | 变量未定义就使用 | 直接引用 `$FOO` 而非 `${FOO:?}` |

**详解几个高频规则：**

**SC2086——引号缺失：**

```bash
# 错误：
if [ $value = "yes" ]; then ...   # $value 为空时 → [ = "yes" ] 语法错误

# 正确：
if [ "$value" = "yes" ]; then ...

# 例外（确实需要单词分割时）：
# shellcheck disable=SC2086
gcc $CFLAGS -o program source.c   # CFLAGS 需要单词分割
```

**SC2164——cd 未检查：**

```bash
# 错误：
cd /some/dir
rm -rf *                           # cd 失败时删当前目录！

# 正确：
cd /some/dir || exit 1
rm -rf *

# 更佳（自动 CDPATH 安全）：
cd /some/dir || exit 1
rm -rf ./*
```

**SC2155——声明与赋值分离不当：**

```bash
# 错误（命令替换失败时，declare 仍返回 0）：
export FOO=$(false)
# → 尽管 false 失败，export 成功，$? = 0，set -e 不触发

# 正确：
FOO=$(false)
export FOO

# 更多例子：
local x=$(false)    # 错误：掩盖失败
local x; x=$(false) # 正确
```

## 8.4 内联禁用检查

```bash
#!/bin/bash

# 禁用整个文件的某条规则
# shellcheck disable=SC1090
source "$HOME/.config/myscript"

# 禁用特定行的检查
# 下一行禁用 SC2086
# shellcheck disable=SC2086
echo $unsafe_var

# 禁用多条规则
# shellcheck disable=SC2086,SC1091
source "${MY_LIB}"

# 精确范围控制
# shellcheck disable=SC2164
cd /tmp || exit
# shellcheck enable=SC2164
```

## 8.5 .shellcheckrc 配置文件

```bash
# ~/.shellcheckrc 或项目根 .shellcheckrc

# 全局禁用的规则
disable=SC1090,SC1091

# 启用可选规则（默认关闭的）
enable=all

# 指定默认 Shell 方言
shell=bash

# 外部来源文件（suppress SC1091）
source-path=SCRIPTDIR/lib/
source-path=/usr/local/share/mylib/
```

## 8.6 CI 集成

**GitHub Actions：**

```yaml
# .github/workflows/shellcheck.yml
name: ShellCheck
on: [push, pull_request]
jobs:
  shellcheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run ShellCheck
        uses: ludeeus/action-shellcheck@master
        with:
          scandir: './scripts'
          format: tty
          severity: warning
```

**GitLab CI：**

```yaml
# .gitlab-ci.yml
shellcheck:
  image: koalaman/shellcheck-alpine:latest
  stage: test
  script:
    - find . -name "*.sh" -type f -exec shellcheck -s bash {} +
```

**Makefile 集成：**

```makefile
.PHONY: lint
lint:
	@echo "Running shellcheck..."
	@find . -name '*.sh' -not -path './vendor/*' \
		-exec shellcheck -s bash -f tty {} +
	@echo "All scripts passed."
```

**pre-commit hook：**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/koalaman/shellcheck-precommit
    rev: v0.10.0
    hooks:
      - id: shellcheck
        args: ["--severity=warning"]
```

## 8.7 shellcheck 的局限性

- 无法理解复杂的间接引用（`${!var}`）的逻辑
- 不能验证外部命令是否存在（那是运行时检查）
- 对通过 `eval` 动态执行的代码无能为力
- 不追踪跨文件的变量状态
- 误报率约 5%~10%（尤其对于故意省略引号的高级用法）

# 9. bashdb 交互式调试器

bashdb 是受 gdb 启发的 Bash 脚本交互式调试器，支持断点、单步执行、变量查看、调用栈等完整调试功能。

## 9.1 安装

```bash
# Linux (apt)
sudo apt install bashdb

# macOS
brew install bashdb

# 源码安装
git clone https://github.com/rocky/bashdb.git
cd bashdb
./configure && make && sudo make install
```

## 9.2 基本使用

```bash
# 调试模式启动
bashdb script.sh
bashdb -- script.sh arg1 arg2

# 在脚本中嵌入断点
# 在需要中断的地方添加：
# _Dbg_debugger  # 旧版写法
```

## 9.3 调试命令速查

| 命令 | 缩写 | 说明 |
|------|------|------|
| `help` | `h` | 帮助 |
| `run` | `R` | 从头开始运行脚本 |
| `restart` | | 重启调试 |
| `step` | `s` | 单步执行（进入函数） |
| `next` | `n` | 单步执行（跳过函数） |
| `finish` | `f` | 执行到当前函数返回 |
| `continue` | `c` | 继续运行到下一个断点 |
| `break <line>` | `b 42` | 在第 42 行设置断点 |
| `break <func>` | `b my_func` | 在函数入口设置断点 |
| `tbreak <line>` | | 设置一次性断点 |
| `delete <num>` | `d 1` | 删除断点 |
| `info break` | `i b` | 列出所有断点 |
| `print <var>` | `p $x` | 查看变量值 |
| `examine <var>` | `x $x` | 详细查看变量 |
| `list` | `l` | 显示当前行附近的源代码 |
| `backtrace` | `bt` | 打印调用栈 |
| `frame <n>` | | 切换到第 n 层调用帧 |
| `watch <var>` | | 监视变量变化 |
| `set variable` | | 修改变量值 |
| `quit` | `q` | 退出调试器 |
| `condition <n> <expr>` | | 给断点添加条件 |

## 9.4 调试示例

```bash
#!/bin/bash
# demo.sh——用于演示 bashdb

calculate() {
    local a=$1
    local b=$2
    local result=$((a * b + a))
    echo "$result"
}

main() {
    local x=5
    local y=10
    local sum=$((x + y))
    echo "Sum: $sum"

    local product
    product=$(calculate "$x" "$y")
    echo "Product variant: $product"
}

main
```

**bashdb 交互式调试过程：**

```
$ bashdb demo.sh
bashdb<0> break 5            # 在 calculate 函数内设置断点
Breakpoint 1 set in file demo.sh, line 5.
bashdb<1> run
Sum: 15                            # ← 执行到断点前
Breakpoint 1 hit.
3:      local b=$2
bashdb<2> print $a                 # 查看参数
5
bashdb<3> p $b
10
bashdb<4> next                     # 执行下一行
4:      local result=$((a * b + a))
bashdb<5> step                     # 进入表达式
bashdb<6> p $result
55
bashdb<7> continue
Product variant: 55
```

## 9.5 条件断点

```bash
# 只在变量满足条件时中断
bashdb<0> break 10 if x > 10
bashdb<0> break 15 if $count -ge 100

# 监视点：变量值改变时中断
bashdb<0> watch counter
```

## 9.6 在脚本中触发调试器

```bash
#!/bin/bash

debug_break() {
    if [ -n "${BASHDB_BREAK:-}" ]; then
        # 判断是否在 bashdb 中运行
        if declare -F _Dbg_debugger >/dev/null 2>&1; then
            _Dbg_debugger
        fi
    fi
}

# 在关键位置插入
data=$(fetch_data)
debug_break  # ← 在此暂停检查 data
```

## 9.7 bashdb 与 set -x 的比较

| 特性 | bashdb | set -x |
|------|--------|--------|
| 交互式断点 | ✓ | ✗ |
| 变量查看/修改 | ✓ | ✗ |
| 单步执行 | ✓ | ✗ |
| 调用栈查看 | ✓ | 需配合 FUNCNAME 数组 |
| 条件断点 | ✓ | 需通过 DEBUG trap 模拟 |
| 配置复杂度 | 高 | 低（一行代码） |
| 适用场景 | 复杂逻辑排查 | 日常快速调试 |
| 性能开销 | 中 | 低 |

# 10. 日志记录

## 10.1 分级日志函数

```bash
#!/bin/bash

# 日志级别
readonly LOG_TRACE=0
readonly LOG_DEBUG=1
readonly LOG_INFO=2
readonly LOG_WARN=3
readonly LOG_ERROR=4
readonly LOG_FATAL=5

# 日志级别名称映射
readonly LOG_NAMES=("TRACE" "DEBUG" "INFO" "WARN" "ERROR" "FATAL")
readonly LOG_COLORS=("\e[90m" "\e[36m" "\e[32m" "\e[33m" "\e[31m" "\e[35m")
readonly LOG_RESET="\e[0m"

# 全局日志级别
LOG_LEVEL="${LOG_LEVEL:-$LOG_INFO}"
LOG_FILE="${LOG_FILE:-/dev/stderr}"

# 设置日志级别
set_log_level() {
    case "${1^^}" in
        TRACE) LOG_LEVEL=$LOG_TRACE ;;
        DEBUG) LOG_LEVEL=$LOG_DEBUG ;;
        INFO)  LOG_LEVEL=$LOG_INFO ;;
        WARN)  LOG_LEVEL=$LOG_WARN ;;
        ERROR) LOG_LEVEL=$LOG_ERROR ;;
        FATAL) LOG_LEVEL=$LOG_FATAL ;;
        *) echo "无效的日志级别: $1" >&2; return 1 ;;
    esac
}

# 核心日志函数
_log() {
    local level=$1
    local message=$2
    local timestamp
    local func_name="${FUNCNAME[2]:-main}"
    local line_no="${BASH_LINENO[1]}"

    # 级别过滤
    if [ "$level" -lt "$LOG_LEVEL" ]; then
        return
    fi

    timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # 是否输出到终端（支持颜色）
    if [ "$LOG_FILE" = "/dev/stderr" ] || [ "$LOG_FILE" = "/dev/stdout" ]; then
        printf "%b[%s] [%s] [%s:%s] %s%b\n" \
            "${LOG_COLORS[$level]}" \
            "$timestamp" \
            "${LOG_NAMES[$level]}" \
            "$func_name" \
            "$line_no" \
            "$message" \
            "$LOG_RESET" >&2
    else
        printf "[%s] [%s] [%s:%s] %s\n" \
            "$timestamp" \
            "${LOG_NAMES[$level]}" \
            "$func_name" \
            "$line_no" \
            "$message" >> "$LOG_FILE"
    fi
}

# 便捷宏
log_trace() { _log $LOG_TRACE "$*"; }
log_debug() { _log $LOG_DEBUG "$*"; }
log_info()  { _log $LOG_INFO  "$*"; }
log_warn()  { _log $LOG_WARN  "$*"; }
log_error() { _log $LOG_ERROR "$*"; }
log_fatal() { _log $LOG_FATAL "$*"; exit 1; }

# 使用示例
log_info "脚本启动"
log_debug "配置路径: $CONFIG_PATH"
log_warn "磁盘使用率超过 80%"
log_error "数据库连接失败"
# log_fatal "关键配置缺失"  # ← 会 exit 1
```

## 10.2 日志文件轮转

```bash
#!/bin/bash

# 日志轮转函数
rotate_log() {
    local logfile="$1"
    local max_size="${2:-10485760}"    # 默认 10MB
    local max_files="${3:-5}"           # 默认保留 5 个归档

    if [ ! -f "$logfile" ]; then
        return
    fi

    # 检查文件大小（字节）
    local size
    size=$(stat -c%s "$logfile" 2>/dev/null || stat -f%z "$logfile" 2>/dev/null)

    if [ "$size" -lt "$max_size" ]; then
        return
    fi

    # 删除最旧的归档文件
    if [ -f "${logfile}.${max_files}" ]; then
        rm -f "${logfile}.${max_files}"
    fi

    # 向后滚动
    local i
    for ((i = max_files - 1; i >= 1; i--)); do
        if [ -f "${logfile}.${i}" ]; then
            mv "${logfile}.${i}" "${logfile}.$((i + 1))"
        fi
    done

    # 当前文件变为 .1
    mv "$logfile" "${logfile}.1"
    touch "$logfile"

    log_info "日志已轮转: ${logfile}"
}

# 使用：
LOG_FILE="/var/log/myscript/app.log"
log_info "写入日志"
rotate_log "$LOG_FILE" 10485760 10  # 10MB × 10 个归档
```

## 10.3 日志同时输出到终端和文件

```bash
#!/bin/bash

LOG_FILE="/var/log/myscript/app.log"
LOG_TO_STDOUT="${LOG_TO_STDOUT:-true}"

log() {
    local level="$1"; shift
    local ts
    ts=$(date '+%Y-%m-%d %H:%M:%S')
    local msg="[$ts] [$level] $*"

    # 写入文件
    echo "$msg" >> "$LOG_FILE"

    # 同时输出到终端
    if [ "$LOG_TO_STDOUT" = "true" ]; then
        echo "$msg"
    fi
}
```

## 10.4 进程替换实现 tee 风格的日志

```bash
#!/bin/bash

LOG_FILE="/var/log/myscript/app.log"

# 利用 tee 将 stderr 同时送到文件和终端
exec 2> >(tee -a "$LOG_FILE" >&2)

# 之后所有 stderr 输出都会同时出现在文件和终端
echo "这条信息会出现在终端和日志文件" >&2

# 风险：tee 进程在脚本退出后可能残留
# 解决：
trap 'exec 2>&4' EXIT  # 还原 stderr
exec 4>&2               # 保存原始 stderr
```

## 10.5 结构化日志（JSON Lines）

```bash
#!/bin/bash

json_log() {
    local level="$1"; shift
    local ts
    ts=$(date -u '+%Y-%m-%dT%H:%M:%SZ')

    # 构建 JSON
    local json="{\"timestamp\":\"$ts\",\"level\":\"$level\",\"pid\":$$"

    # 追加额外字段
    for pair in "$@"; do
        key="${pair%%:*}"
        value="${pair#*:}"
        json="$json,\"$key\":\"$value\""
    done

    json="$json}"
    echo "$json"
}

# 使用：
json_log "INFO" "message:脚本启动" "config:/etc/app.conf" "user:$USER"

# 传递给日志收集系统（如 ELK / Loki）
json_log "ERROR" "message:连接失败" "host:db.example.com" "port:5432" | tee -a /var/log/app.jsonl
```

# 11. 常见错误类型与排查方法

## 11.1 变量未加引号

**症状：** `[: too many arguments`、`binary operator expected`、参数截断。

```bash
#!/bin/bash

# 错误演示：
filename="my document.txt"
[ -f $filename ]        # → [: too many arguments
[ $status = "ok" ]      # status 为空时 → [: =: unexpected operator

# 排查方法：
# 1. shellcheck 直接报 SC2086
# 2. set -x 观察展开
# 3. 始终加引号

# 正确：
[ -f "$filename" ]
[ "$status" = "ok" ]

# 特殊：整数比较
[ "$count" -gt 10 ]           # 加引号但确保是整数
# 更稳妥（空值保护）：
[ "${count:-0}" -gt 10 ]      # count 为空时视为 0
```

## 11.2 路径中的空格和特殊字符

```bash
#!/bin/bash

# 问题根源：IFS 默认包含空格，导致路径被分割成多个单词
# 排查：set -x 查看展开后的命令

# 错误：
path="/home/user/My Documents/report.txt"
cp $path /tmp    # cp 接收到 3 个参数而非 2 个

# 正确：
cp "$path" /tmp

# 批量处理文件的最安全方式：
find . -name "*.txt" -print0 | while IFS= read -r -d '' file; do
    echo "处理: $file"
    wc -l "$file"
done
```

## 11.3 set -e 的七宗罪

**罪状 1：grep 无匹配时退出**

```bash
set -e
count=$(grep -c "ERROR" app.log)   # 没有 ERROR → 退出
# 修正：
count=$(grep -c "ERROR" app.log) || true
# 或
count=$(grep -c "ERROR" app.log || true)
```

**罪状 2：let 结果为 0 时退出**

```bash
set -e
let "a = 0"       # let 的退出码是表达式结果，0 → 退出状态码 1！
# 修正：
(( a = 0 )) || true
# 或
a=0
```

**罪状 3：子 Shell 中 -e 行为不一致**

```bash
set -e
( false; echo "Bash 4.4+ 会退出子 Shell，旧版本可能不退出" )
echo "此行在某些 Bash 版本中仍执行"
```

**罪状 4：函数中的错误被吞没**

```bash
set -e
myfunc() {
    false          # -e 触发 → 退出函数
    echo "done"    # 不执行
}
# 但函数调用处——如果作为条件则不退出
if myfunc; then    # myfunc 内的 false 导致 -e，但 if 阻止了全局退出
    echo "never"
fi
```

**罪状 5：命令替换中的失败被忽略**

```bash
set -e
result=$(cat /nonexistent 2>/dev/null)  # 不退出（Bash 4.4 之前）！
# 修正：
result=$(cat /nonexistent 2>/dev/null) || exit 1
```

**罪状 6：后台进程中的失败不触发 -e**

```bash
set -e
{ false; echo "bg"; } &   # 后台中 -e 不生效
wait                       # wait 获取的是后台进程退出码，但不触发 -e
```

**罪状 7：管道与 -e 的复杂交互**

```bash
set -e
# 没有 pipefail 时：
false | true     # 不退出（管道整体 $? = 0）
# 有 pipefail 时：
set -o pipefail
false | true     # 退出（false 导致 $? = 1）

# 但：
set -eo pipefail
true && false | true   # 不退出：&& 优先级高于 |，整体是 true && (false | true)
```

## 11.4 引号层级混淆

```bash
#!/bin/bash

# 错误：引号层级混乱
cmd="find . -name '*.log' -exec grep 'ERROR' {} \;"
eval "$cmd"

# 正确：使用数组避免引号地狱
cmd=(find . -name "*.log" -exec grep "ERROR" {} \;)
"${cmd[@]}"

# 或：直接写，不要用变量存命令
find . -name "*.log" -exec grep "ERROR" {} \;
```

## 11.5 IFS 修改的副作用

```bash
#!/bin/bash

# 隐患：修改 IFS 后忘记恢复
OLD_IFS="$IFS"
IFS=':'
read -r user pass uid gid name home shell < /etc/passwd
IFS="$OLD_IFS"        # 务必恢复

# 更安全：使用子 Shell 局部修改
parse_passwd() {
    local IFS=':'
    read -r user pass uid gid name home shell < /etc/passwd
}
parse_passwd           # IFS 只在函数内修改
```

## 11.6 正确的数字比较

```bash
#!/bin/bash

a="5"
b="10"

# 错误：字符串比较
if [ "$a" \> "$b" ]; then  # 字符串按字典序，"5" > "10" 为真！
    echo "a > b (错误)"
fi

# 正确：整数比较
if [ "$a" -gt "$b" ]; then    # -gt / -lt / -eq / -ne / -ge / -le
    echo "a > b"
fi

# 更安全（算术上下文，天然处理空值）：
if (( a > b )); then
    echo "a > b"
fi
```

## 11.7 排查工具链总结

```bash
# 排查流程：
# 第 1 步：语法检查
bash -n script.sh

# 第 2 步：静态分析
shellcheck script.sh

# 第 3 步：开启所有调试
bash -euxo pipefail script.sh 2>&1 | tee debug.log

# 第 4 步：检查退出码
echo "Exit: $?"

# 第 5 步：交互式排查（如果仍无法定位）
bashdb script.sh

# 第 6 步：针对性加 DEBUG/ERR trap
# 在第 6 步之前，在脚本中添加：
trap 'echo "第${LINENO}行: $BASH_COMMAND 失败 ($?)"' ERR
```

# 12. 调试脚本模板

以下是一个集成了本文所有最佳实践的生产级脚本模板。

```bash
#!/bin/bash
#===============================================================================
# 脚本名称: template.sh
# 说明:     生产级 Shell 脚本模板（集调试、错误处理、日志于一体）
# 用法:     ./template.sh [options] <args>
# 选项:
#   -d, --debug     启用调试模式（set -x + DEBUG trap）
#   -v, --verbose   详细输出（日志级别 DEBUG）
#   -h, --help      显示帮助
#===============================================================================

#-------------------------------------------------------------------------------
# 1. 全局配置
#-------------------------------------------------------------------------------
set -euo pipefail
IFS=$'\n\t'

SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME SCRIPT_DIR

#-------------------------------------------------------------------------------
# 2. 日志系统
#-------------------------------------------------------------------------------
readonly LOG_TRACE=0 LOG_DEBUG=1 LOG_INFO=2 LOG_WARN=3 LOG_ERROR=4 LOG_FATAL=5
readonly LOG_NAMES=("TRACE" "DEBUG" " INFO" " WARN" "ERROR" "FATAL")

LOG_LEVEL="${LOG_LEVEL:-$LOG_INFO}"
LOG_FILE="${LOG_FILE:-/dev/stderr}"
LOG_COLOR="${LOG_COLOR:-true}"

_log() {
    local level=$1; shift
    local message="$*"
    local ts func_name="${FUNCNAME[2]:-main}" line_no="${BASH_LINENO[1]}"

    [ "$level" -lt "$LOG_LEVEL" ] && return

    ts="$(date '+%Y-%m-%d %H:%M:%S')"
    printf "[%s] [%s] [%s:%s] %s\n" \
        "$ts" "${LOG_NAMES[$level]}" "$func_name" "$line_no" "$message" >&2
}

log_trace() { _log $LOG_TRACE "$*"; }
log_debug() { _log $LOG_DEBUG "$*"; }
log_info()  { _log $LOG_INFO  "$*"; }
log_warn()  { _log $LOG_WARN  "$*"; }
log_error() { _log $LOG_ERROR "$*"; }
log_fatal() { _log $LOG_FATAL "$*"; exit 1; }

#-------------------------------------------------------------------------------
# 3. 错误处理
#-------------------------------------------------------------------------------
on_error() {
    local exit_code=$?
    local frame=0

    {
        echo ""
        echo "======================================================================"
        echo "  脚本错误报告"
        echo "======================================================================"
        echo "  脚本:     ${BASH_SOURCE[1]:-$0}"
        echo "  行号:     ${BASH_LINENO[0]}"
        echo "  退出码:   ${exit_code}"
        echo "  失败命令: ${BASH_COMMAND}"
        echo "  时间:     $(date '+%Y-%m-%d %H:%M:%S')"
        echo "  PID:      $$"
        echo "----------------------------------------------------------------------"
        echo "  调用栈:"
        for ((frame=1; frame<${#FUNCNAME[@]}; frame++)); do
            printf '    [%d] %s() at %s:%s\n' \
                "$frame" \
                "${FUNCNAME[$frame]}" \
                "${BASH_SOURCE[$frame]:-}" \
                "${BASH_LINENO[$frame-1]}"
        done
        echo "======================================================================"
    } >&2
}

trap on_error ERR

# 退出清理
cleanup() {
    local exit_code=$?
    log_info "脚本退出，退出码: ${exit_code}"

    # 清理临时文件
    if [ -n "${TMPDIR:-}" ] && [ -d "$TMPDIR" ]; then
        rm -rf "$TMPDIR"
        log_debug "已清理临时目录: $TMPDIR"
    fi

    exit $exit_code
}

trap cleanup EXIT INT TERM

#-------------------------------------------------------------------------------
# 4. 调试系统
#-------------------------------------------------------------------------------
DEBUG_MODE="${DEBUG_MODE:-false}"

setup_debug() {
    local saved_opts
    saved_opts="$(set +o)"

    # PS4：时间戳 + 脚本名 + 行号 + 函数名 + 缩进
    export PS4='+[${SECONDS}s][${BASH_SOURCE##*/}:${LINENO}]${FUNCNAME[0]:+[${FUNCNAME[0]}]}: '

    set -x
    log_debug "调试模式已启用"

    # 返回关闭调试的函数
    _disable_debug() { eval "$saved_opts"; }
}

# DEBUG trap：条件触发追踪
if [[ "${ENABLE_DEBUG_TRAP:-false}" == "true" ]]; then
    trap 'log_trace "→ ${BASH_COMMAND}"' DEBUG
fi

#-------------------------------------------------------------------------------
# 5. 工具函数
#-------------------------------------------------------------------------------
die() {
    log_fatal "$@"
}

usage() {
    cat <<EOF
用法: $SCRIPT_NAME [选项] <参数>

选项:
  -d, --debug     启用 xtrace 调试
  -v, --verbose   详细日志输出
  -h, --help      显示此帮助信息

示例:
  $SCRIPT_NAME input.txt
  DEBUG_MODE=true $SCRIPT_NAME input.txt
  LOG_LEVEL=0 $SCRIPT_NAME input.txt
EOF
    exit 0
}

require_command() {
    local cmd="$1"
    if ! command -v "$cmd" &>/dev/null; then
        log_fatal "依赖命令 '$cmd' 未安装"
    fi
}

require_file() {
    local file="$1" desc="${2:-文件}"
    if [ ! -f "$file" ]; then
        log_fatal "${desc} 不存在: $file"
    fi
    if [ ! -r "$file" ]; then
        log_fatal "${desc} 不可读: $file"
    fi
}

#-------------------------------------------------------------------------------
# 6. 参数解析
#-------------------------------------------------------------------------------
parse_args() {
    local args=()
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -d|--debug)
                DEBUG_MODE=true
                shift
                ;;
            -v|--verbose)
                LOG_LEVEL=$LOG_DEBUG
                shift
                ;;
            -vv)
                LOG_LEVEL=$LOG_TRACE
                shift
                ;;
            -h|--help)
                usage
                ;;
            --)
                shift
                args+=("$@")
                break
                ;;
            -*)
                log_error "未知选项: $1"
                usage
                ;;
            *)
                args+=("$1")
                shift
                ;;
        esac
    done

    # 验证必需参数
    if [ "${#args[@]}" -eq 0 ]; then
        log_error "缺少必需参数"
        usage
    fi

    # 导出位置参数
    set -- "${args[@]}"
}

#-------------------------------------------------------------------------------
# 7. 主业务逻辑
#-------------------------------------------------------------------------------
main() {
    parse_args "$@"

    # 启用调试（如果请求）
    if [[ "$DEBUG_MODE" == "true" ]]; then
        setup_debug
    fi

    log_info "脚本启动: $SCRIPT_NAME"
    log_debug "参数: $*"
    log_debug "脚本目录: $SCRIPT_DIR"
    log_debug "工作目录: $PWD"

    # 检查依赖
    require_command "jq"
    require_command "curl"

    # 业务逻辑开始
    local input_file="$1"

    require_file "$input_file" "输入文件"

    log_info "开始处理: $input_file"

    # ... 你的业务代码 ...
    local line_count
    line_count=$(wc -l < "$input_file")
    log_info "文件行数: $line_count"

    # 业务逻辑结束
    log_info "脚本执行完毕"
}

#-------------------------------------------------------------------------------
# 入口点
#-------------------------------------------------------------------------------
main "$@"
```

## 12.1 模板使用指南

```bash
# 基本执行
chmod +x template.sh
./template.sh data.txt

# 调试模式
./template.sh --debug data.txt

# 详细日志
./template.sh -v data.txt

# 全调试（TRACE 级别 + xtrace + DEBUG trap）
ENABLE_DEBUG_TRAP=true LOG_LEVEL=0 ./template.sh --debug data.txt

# 日志输出到文件
LOG_FILE=./app.log ./template.sh data.txt

# 管道前确认语法
bash -n template.sh && ./template.sh data.txt
```

## 12.2 模板自定义接入点

| 接入点 | 位置 | 用途 |
|--------|------|------|
| `log_*` 函数 | 第 2 节 | 替换为 syslog 协议或企业日志 SDK |
| `on_error` | 第 3 节 | 接入告警系统（钉钉、Slack、PagerDuty） |
| `cleanup` | 第 3 节 | 释放数据库连接、解锁文件、清理资源 |
| `require_command` | 第 5 节 | 添加版本检查（如 `jq >= 1.6`） |
| `parse_args` | 第 6 节 | 扩展为 getopt 完整参数解析 |
