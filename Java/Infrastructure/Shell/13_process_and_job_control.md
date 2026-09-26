
# 1. 进程基础

## 1.1 进程概念

**进程**是正在执行的程序的实例。每个进程有独立的 PID（进程 ID）、内存空间、文件描述符表、环境变量。

```bash
# 查看当前 Shell 的 PID
echo $$

# 查看父进程 PID
echo $PPID

# 查看进程树
pstree -p $$
```

## 1.2 进程状态

| 状态码 | 名称 | 含义 |
|--------|------|------|
| R | Running | 正在运行或就绪 |
| S | Sleeping | 可中断休眠（等待 I/O） |
| D | Disk Sleep | 不可中断休眠（等待硬件） |
| Z | Zombie | 僵尸进程（已终止但未回收） |
| T | Stopped | 被信号停止（如 Ctrl+Z） |
| X | Dead | 已死亡（瞬时状态） |

```bash
# 查看进程状态
ps aux
ps -eo pid,stat,cmd

# STAT 列的附加标志
# <  高优先级
# N  低优先级
# +  前台进程组
# s  会话领导者
# l  多线程
```

# 2. ps — 查看进程

```bash
# BSD 风格（最常用）
ps aux                  # 所有用户所有进程
ps aux | grep nginx

# Unix 风格
ps -ef                  # 全格式
ps -eLf                 # 显示线程
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%cpu | head

# 进程树
ps -ejH
ps axjf

# 按终端
ps -a                   # 当前终端所有用户进程
ps -t pts/0             # 指定终端

# 实时监控
top                     # 交互式
htop                    # 增强版（需安装）
```

# 3. pgrep / pkill — 按名称操作

```bash
# 按名称查找 PID
pgrep -f nginx
pgrep -u alice
pgrep -x sshd               # 精确匹配

# 按名称杀死
pkill -f "python script.py"
pkill -HUP nginx             # 发送重载信号

# 查找并列出详细信息
pgrep -fl java
pgrep -u root -l
```

# 4. 前台与后台

## 4.1 基本操作

```bash
# 后台运行（命令后加 &）
sleep 300 &
long_running_task &

# 查看后台作业
jobs

# 将当前前台进程放到后台（Ctrl+Z 暂停，然后 bg）
# 终端操作：Ctrl+Z → bg

# 将后台进程调回前台
fg            # 最近的
fg %1         # 指定 job ID
fg %sleep     # 按命令名前缀

# 后台运行暂停的进程
bg %1
```

## 4.2 nohup — 忽略挂断信号

```bash
# 启动一个不受终端关闭影响的进程
nohup long_task.sh &

# 默认输出到 nohup.out
nohup python app.py &

# 自定义输出位置
nohup java -jar app.jar > app.log 2>&1 &

# 查看 nohup 进程
ps aux | grep nohup
```

## 4.3 disown — 从作业表中移除

```bash
# 场景：任务已后台运行，但忘了加 nohup
# 1. Ctrl+Z 暂停
# 2. bg 放到后台
# 3. disown 从 job table 移除（终端关闭不影响）

disown         # 移除最近的
disown %1      # 按 job ID
disown -a      # 移除所有
disown -r      # 只移除正在运行的
disown -h %1   # 标记为不接收 SIGHUP（但保留在 job table）
```

# 5. 进程管理命令

## 5.1 kill — 发送信号

```bash
# kill 本质是发信号，不只是"杀死"
kill PID                # 默认 SIGTERM (15)
kill -l                 # 列出所有信号
kill -9 PID             # SIGKILL（强制杀死）
kill -15 PID            # SIGTERM（优雅终止）
kill -HUP PID           # SIGHUP（重新加载配置）
kill -USR1 PID          # SIGUSR1（用户自定义信号 1）

# 给多个进程发信号
kill -TERM 1234 5678

# killall 按名称
killall -HUP nginx
killall -9 java
```

## 5.2 wait — 等待子进程

```bash
# 等待所有后台进程
sleep 3 &
sleep 5 &
wait
echo "所有任务完成"

# 等待特定 PID
long_task &
TASK_PID=$!
# ...做其他事情...
wait $TASK_PID
echo "任务完成，退出码: $?"

# 等待任意一个子进程完成（bash 4.3+）
wait -n
```

## 5.3 timeout — 限时执行

```bash
# 命令执行超过 5 秒则终止
timeout 5 slow_command

# 指定信号
timeout -s SIGKILL 10s command
timeout -s 9 30m command

# 时间格式
timeout 10s command     # 10 秒
timeout 5m command      # 5 分钟
timeout 2h command      # 2 小时
timeout 1d command      # 1 天
```

# 6. 进程优先级

## 6.1 nice / renice

```bash
# nice 启动进程时设置优先级（-20 最高，19 最低，默认 0）
nice -n 10 cpu_intensive_task       # 降低优先级
nice -n -5 important_task           # 提高优先级（需 root）

# renice 修改运行中进程的优先级
renice -n 5 -p 1234                 # 降低 PID 1234 的优先级
renice -n -10 -u alice              # 提高用户 alice 所有进程的优先级
```

## 6.2 CPU 亲和性（taskset）

```bash
# 查看进程的 CPU 亲和性
taskset -p PID

# 绑定进程到特定 CPU 核心
taskset -cp 0,1 PID              # 绑定到 CPU 0 和 1
taskset -c 2-3 command            # 启动时绑定到 CPU 2-3
```

# 7. 进程组与会话

## 7.1 概念层级

```
会话 (Session)
 ├── 前台进程组 (Foreground Process Group)
 │   ├── 进程 A
 │   ├── 进程 B
 │   └── 进程 C (管道中的每个命令)
 └── 后台进程组 (Background Process Group)
     └── 进程 D
```

```bash
# 查看进程组 ID
ps -eo pid,pgid,sid,cmd

# 创建新的会话（守护进程化）
setsid command

# 在脚本中创建守护进程
(
    # 子 Shell 中执行
    nohup long_running_task &
)
```

# 8. 作业控制实战

## 8.1 长时间任务管理

```bash
#!/bin/bash
# 任务并发控制

MAX_JOBS=4
TASK_COUNT=0

process_item() {
    echo "处理: $1"
    sleep $((RANDOM % 10 + 1))
    echo "完成: $1"
}

for item in {1..20}; do
    # 控制并发数
    while [ "$(jobs -r | wc -l)" -ge "$MAX_JOBS" ]; do
        sleep 0.5
    done

    process_item "$item" &
done

# 等待所有完成
wait
echo "全部完成"
```

## 8.2 超时与重试

```bash
#!/bin/bash
# 带超时的命令执行

run_with_timeout() {
    local timeout=$1
    shift

    # 启动后台命令
    "$@" &
    local pid=$!

    # 等待超时
    (
        sleep "$timeout"
        kill -TERM $pid 2>/dev/null
        sleep 1
        kill -KILL $pid 2>/dev/null
    ) &
    local killer_pid=$!

    # 等待命令完成
    wait $pid 2>/dev/null
    local exit_code=$?

    # 清理超时进程
    kill $killer_pid 2>/dev/null
    wait $killer_pid 2>/dev/null

    return $exit_code
}
```

## 8.3 进程监控

```bash
#!/bin/bash
# 监控并自动重启进程

restart_process() {
    echo "进程 $1 已退出，正在重启..."
    sleep 2
    "$@" &
}

monitor_process() {
    "$@" &
    local pid=$!
    echo "正在监控 PID: $pid"

    while kill -0 $pid 2>/dev/null; do
        sleep 1
    done

    echo "PID $pid 已终止"
    restart_process "$@"
    monitor_process "$@"     # 递归监控
}

# monitor_process ./my_daemon.sh
```

# 9. 守护进程编写

```bash
#!/bin/bash
# 将脚本守护进程化

daemonize() {
    # 1. fork 并退出父进程
    if [ "$DAEMONIZED" != "1" ]; then
        export DAEMONIZED=1
        nohup "$0" "$@" > /dev/null 2>&1 &
        echo "守护进程已启动，PID: $!"
        exit 0
    fi

    # 2. 子进程继续
    # 3. 创建新会话
    # 4. 切换工作目录
    cd / || exit
    # 5. 重定向标准文件描述符
    exec 0</dev/null
    exec 1>/dev/null
    exec 2>&1
    # 6. 关闭不必要的文件描述符
    # 7. 信号处理
    trap '' SIGHUP

    # 8. 业务逻辑
    while true; do
        echo "运行中... $(date)"
        sleep 60
    done
}

daemonize "$@"
```

# 10. 子系统与 cgroups（容器视角）

```bash
# 查看进程的 cgroup
cat /proc/$$/cgroup

# 限制内存
# 创建 cgroup v2 控制器
echo "100M" | sudo tee /sys/fs/cgroup/myapp/memory.max
echo $$ | sudo tee /sys/fs/cgroup/myapp/cgroup.procs

# 限制 CPU
echo "50000 100000" | sudo tee /sys/fs/cgroup/myapp/cpu.max  # 50% CPU
```
