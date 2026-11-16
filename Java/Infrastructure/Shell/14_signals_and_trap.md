
# 1. 信号概述

**信号**是 Unix/Linux 中进程间通信的一种**异步通知机制**，用于通知进程发生了特定事件。

```bash
# 查看系统支持的所有信号
kill -l
# 或
trap -l
```

## 1.1 信号分类

**不可靠信号（1~31）：** 可能丢失，无排队机制
**可靠信号（34~64）：** 支持排队，不会丢失

## 1.2 常用信号速查

| 编号 | 名称 | 默认行为 | 说明 |
|------|------|----------|------|
| 1 | SIGHUP | Term | 挂断（终端关闭）。常被守护进程用于重新加载配置 |
| 2 | SIGINT | Term | 终端中断（Ctrl+C） |
| 3 | SIGQUIT | Core | 退出并产生核心转储（Ctrl+\） |
| 9 | SIGKILL | Term | 强制杀死（**无法捕获/忽略**） |
| 15 | SIGTERM | Term | 终止（默认 kill 信号） |
| 17 | SIGCHLD | Ign | 子进程状态改变（默认忽略） |
| 18 | SIGCONT | Cont | 继续执行（恢复 STOP 的进程） |
| 19 | SIGSTOP | Stop | 停止执行（**无法捕获/忽略**） |
| 20 | SIGTSTP | Stop | 终端停止（Ctrl+Z） |
| 10 | SIGUSR1 | Term | 用户自定义信号 1 |
| 12 | SIGUSR2 | Term | 用户自定义信号 2 |
| 28 | SIGWINCH | Ign | 终端窗口大小改变 |
| 0 | EXIT | — | Bash 伪信号：Shell 退出时触发 |
| ERR | ERR | — | Bash 伪信号：命令返回非零时触发 |
| DEBUG | DEBUG | — | Bash 伪信号：每条命令执行前触发 |
| RETURN | RETURN | — | Bash 伪信号：函数/source 返回时触发 |

**默认行为说明：**
- **Term**：终止进程
- **Ignore**：忽略信号
- **Core**：终止并产生 core dump
- **Stop**：暂停（挂起）进程
- **Cont**：继续运行暂停的进程

# 2. trap 基本语法

```bash
trap '命令' 信号列表
trap - 信号列表        # 恢复信号默认处理
trap -p                # 列出当前设置的 trap
trap '' 信号列表       # 忽略信号
```

## 2.1 简单示例

```bash
#!/bin/bash

# 捕获 SIGINT (Ctrl+C)
trap 'echo "捕获到 Ctrl+C"; exit 1' SIGINT

# 或多信号
trap 'echo "收到中断信号"' INT TERM

# 主循环
while true; do
    echo "运行中... (按 Ctrl+C 退出)"
    sleep 2
done
```

# 3. 常用 trap 模式

## 3.1 退出时清理（最常用）

```bash
#!/bin/bash
TMPDIR=$(mktemp -d)
TMPFILE=$(mktemp)

cleanup() {
    local exit_code=$?
    echo "清理中..."
    rm -rf "$TMPDIR"
    rm -f "$TMPFILE"
    echo "脚本退出，退出码: $exit_code"
    exit $exit_code
}

# 注册 EXIT trap（无论正常退出还是异常退出都触发）
trap cleanup EXIT

# 业务逻辑
echo "工作目录: $TMPDIR"
echo "临时文件: $TMPFILE"
# ... 执行操作 ...
```

**最佳实践：** 将清理函数和 `trap cleanup EXIT` 放在脚本开头，确保无论什么原因退出都不会遗留临时文件。

## 3.2 捕获错误并退出

```bash
#!/bin/bash
set -E     # ERR trap 在函数中也会被继承

error_handler() {
    echo "错误发生在行 ${BASH_LINENO[0]}: 命令 '$BASH_COMMAND' 返回 $?" >&2
    exit 1
}

trap error_handler ERR

# 所有错误都会被捕获
echo "开始执行"
false       # 触发 ERR trap
echo "这行不会执行"
```

## 3.3 优雅关闭（处理 SIGTERM）

```bash
#!/bin/bash

SHOULD_EXIT=0

graceful_shutdown() {
    echo "收到终止信号，正在优雅关闭..."
    SHOULD_EXIT=1
}

trap graceful_shutdown SIGTERM SIGINT

# 长时间运行的服务
while [ $SHOULD_EXIT -eq 0 ]; do
    echo "服务运行中 $(date)"
    sleep 5
done

echo "服务已停止"
```

## 3.4 重新加载配置（处理 SIGHUP）

```bash
#!/bin/bash

CONFIG_FILE="/etc/myapp/config.conf"

reload_config() {
    echo "重新加载配置: $CONFIG_FILE"
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
        echo "配置已更新"
    else
        echo "配置文件不存在" >&2
    fi
}

trap reload_config SIGHUP

# 首次加载
reload_config

# 主循环
while true; do
    echo "运行中，PID: $$ (发送 kill -HUP $$ 重载配置)"
    sleep 10
done
```

## 3.5 动态 trap（开启/关闭）

```bash
# 临时忽略中断信号（保护关键操作）
echo "关键操作开始..."
trap '' SIGINT    # 忽略 Ctrl+C
# ... 关键操作 ...
trap - SIGINT     # 恢复默认
echo "关键操作完成"
```

# 4. Bash 伪信号

## 4.1 DEBUG — 每条命令执行前触发

```bash
#!/bin/bash

# 在每条命令前输出信息
trap 'echo "执行: $BASH_COMMAND"' DEBUG

echo "Hello"
ls /tmp
date
```

## 4.2 ERR — 命令出错时触发

```bash
#!/bin/bash

# 错误时自动输出堆栈
trap 'echo "错误: 命令 $BASH_COMMAND 失败于 $LINENO"' ERR

echo "ok"
false        # 触发 ERR
echo "不会执行"  # 这行仍然会执行（ERR 不会阻止继续）
```

**注意：** ERR 不会使脚本退出（除非配合 `set -e`）。需要传入函数时使用 `set -E`。

## 4.3 RETURN — 函数/source 返回时

```bash
#!/bin/bash

trap 'echo "函数返回: $FUNCNAME"' RETURN

my_func() {
    echo "函数体中"
}

echo "主脚本中"
my_func
source /dev/null
```

# 5. 信号与子进程

```bash
# 默认：信号会传递给子进程
trap 'echo "收到 INT"' INT
sleep 100 &
# Ctrl+C → sleep 和 当前脚本都收到 SIGINT

# 在子 Shell / 管道中 trap 的继承
(
    trap 'echo "子 Shell 收到 INT"' INT
    sleep 100
) &
kill -INT %1
```

# 6. 信号处理的高级模式

## 6.1 防重复执行（锁文件 + 信号清理）

```bash
#!/bin/bash
LOCKFILE="/var/run/myscript.lock"

cleanup() {
    rm -f "$LOCKFILE"
    echo "释放锁文件"
}

# 注册退出清理
trap cleanup EXIT INT TERM

# 获取锁
if [ -f "$LOCKFILE" ]; then
    echo "脚本已在运行（锁文件存在）" >&2
    exit 1
fi

echo $$ > "$LOCKFILE"

# 业务逻辑
echo "获取锁成功，PID: $$"
sleep 30
```

## 6.2 超时模式

```bash
#!/bin/bash

# 5 秒后自动 SIGALRM
timeout_handler() {
    echo "操作超时！"
    exit 124
}

trap timeout_handler ALRM

(
    sleep 5
    kill -ALRM $$   # 5 秒后给父进程发信号
) &

echo "请在 5 秒内输入:"
read -r input
echo "输入: $input"
```

## 6.3 多重信号处理

```bash
#!/bin/bash

declare -i signal_count=0

signal_handler() {
    signal_count+=1
    echo "收到信号 $1（第 $signal_count 次）"

    if [ $signal_count -ge 3 ]; then
        echo "收到第 3 次信号，强制退出"
        exit 1
    fi
}

trap 'signal_handler INT' SIGINT
trap 'signal_handler TERM' SIGTERM
trap 'signal_handler HUP' SIGHUP

echo "PID: $$ — 试试 kill -INT/-TERM/-HUP $$"
while true; do sleep 1; done
```

# 7. 常见信号场景

## 7.1 信号在管道中

```bash
# 管道中的所有进程都在同一个进程组中
# Ctrl+C 发送 SIGINT 到整个前台进程组

# 单独控制
trap 'echo "主进程收到 INT"' INT
long_command | another_command
```

## 7.2 忽略不应捕获的信号

```bash
# SIGKILL (9) 和 SIGSTOP (19) 无法被捕获或忽略
# 尝试捕获它们不会有任何效果

# 其他无法恢复继续运行的信号（如 SIGCONT）
trap '' SIGWINCH     # 忽略窗口大小变化
trap '' SIGCHLD      # 忽略子进程退出（不产生僵尸）
```

## 7.3 信号安全函数

`trap` 处理函数中调用的命令应尽量简单。在信号处理函数中：
- 避免使用 `malloc`/`free`（不会在 bash 中直接遇到）
- 使用内置命令而非外部命令
- 避免复杂的 I/O 操作
- 最佳实践：只设置标志位，在主循环中检查

```bash
received_signal=0

trap 'received_signal=1' INT TERM

while [ $received_signal -eq 0 ]; do
    # 处理任务
    sleep 1
done

echo "收到信号，退出"
```

# 8. 错误处理的完整模式

```bash
#!/bin/bash
set -Eeuo pipefail

# 错误堆栈
error_handler() {
    local lineno=$1
    local msg=$2
    echo "错误 (行 $lineno): $msg" >&2

    # 打印调用栈
    local i=0
    while caller $i; do
        ((i++))
    done >&2
}

trap 'error_handler ${LINENO} "$BASH_COMMAND"' ERR

# 清理
cleanup() {
    local exit_code=$?
    echo "清理中... 退出码: $exit_code"
}
trap cleanup EXIT

# 业务逻辑...
```
