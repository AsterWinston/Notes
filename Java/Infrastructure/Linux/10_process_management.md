
# 1. 进程基础概念

## 1.1 什么是进程

进程（Process）是正在执行的程序实例，是操作系统进行资源分配的基本单位。每个进程拥有独立的地址空间、文件描述符、堆栈和环境。

**程序 vs 进程**：
- 程序：静态的二进制文件（存储在磁盘上）
- 进程：动态的执行实例（在内存中运行）

## 1.2 进程标识

```bash
PID     # 进程 ID（Process ID），每个进程唯一的数字标识
PPID    # 父进程 ID（Parent Process ID）
PGID    # 进程组 ID
SID     # 会话 ID
```

**init 进程（PID=1）**：所有进程的祖先进程，由内核启动的第一个用户空间进程（在现代 Linux 中通常是 systemd）。

## 1.3 进程状态

```
R (Running/ Runnable)   # 正在运行或在运行队列中等待
S (Sleeping)            # 可中断睡眠（等待某事件完成）
D (Uninterruptible Sleep) # 不可中断睡眠（通常是等待 I/O）
T (Stopped)             # 停止状态（收到 SIGSTOP 或 SIGTSTP）
Z (Zombie)              # 僵尸状态（已终止但未被父进程回收）
I (Idle)                # 空闲内核线程（内核 2.6+）
```

**进程状态转换**：
```
创建 → 就绪 (R) ↔ 运行 (R) ↔ 睡眠 (S/D)
           ↕               ↕
          停止 (T)    ← 终止 → 僵尸 (Z) → 回收
```

## 1.4 僵尸进程

僵尸进程是已终止但未被父进程 wait() 回收的进程。僵尸进程几乎不消耗资源（仅保留 PCB），但过多的僵尸进程会耗尽 PID 资源。

**处理方式**：
```bash
# 杀死僵尸进程的父进程（使僵尸被 init 收养并回收）
kill -9 <PPID>

# 如果父进程是 init（PID=1），通常需要重启系统
```

# 2. ps — 进程快照

## 2.1 常用用法

```bash
ps                        # 当前终端下的进程
ps -ef                    # 所有进程完整信息（System V 风格）
ps aux                    # 所有进程（BSD 风格）
ps -e                     # 所有进程
ps -f                     # 完整格式
ps -l                     # 长格式

# 组合
ps aux --sort=-%mem       # 按内存使用率降序
ps aux --sort=-%cpu       # 按 CPU 使用率降序
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu  # 自定义输出字段
```

## 2.2 常用字段含义

| 字段 | 含义 |
|------|------|
| PID | 进程 ID |
| PPID | 父进程 ID |
| %CPU | CPU 使用率 |
| %MEM | 内存使用率 |
| RSS | 常驻内存大小（KB）|
| VSZ | 虚拟内存大小（KB）|
| STAT | 进程状态码 |
| START | 启动时间 |
| TIME | 累计 CPU 时间 |
| CMD | 命令名（含参数）|

## 2.3 进程状态码（STAT）

| 状态码 | 含义 |
|--------|------|
| R | 运行/可运行 |
| S | 睡眠 |
| D | 不可中断睡眠 |
| T | 停止 |
| Z | 僵尸 |
| < | 高优先级 |
| N | 低优先级 |
| s | 会话领导者 |
| + | 前台进程组 |
| l | 多线程 |

示例：`Ssl` = 睡眠 + 会话领导者 + 多线程

## 2.4 查找特定进程

```bash
ps aux | grep nginx
pgrep nginx                   # 只返回 PID
pgrep -u alice                # 查找用户 alice 的进程
pgrep -f "python script.py"   # 匹配完整命令行
pgrep -l nginx                # 显示 PID 和进程名
```

# 3. top / htop — 动态监控

## 3.1 top

```bash
top                           # 启动 top
top -u alice                  # 只显示特定用户的进程
top -p 1234,5678              # 只监控指定 PID
top -d 2                      # 刷新间隔为 2 秒
```

top 界面中各列含义：

```
top - 10:30:00 up 30 days,  2:15,  2 users,  load average: 0.15, 0.10, 0.05
Tasks: 180 total,   1 running, 179 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.5 us,  0.8 sy,  0.0 ni, 96.5 id,  0.2 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7861.0 total,   2048.3 free,   3200.5 used,   2612.2 buff/cache
MiB Swap:   2048.0 total,   1850.0 free,    198.0 used.   4320.7 avail Mem
```

**交互命令**（top 运行时按键）：

| 按键 | 作用 |
|------|------|
| `1` | 展开/折叠每个 CPU 核心 |
| `P` | 按 CPU 使用率排序 |
| `M` | 按内存使用率排序 |
| `T` | 按运行时间排序 |
| `k` | 杀死进程（输入 PID 和信号）|
| `r` | 修改优先级（renice）|
| `u` | 筛选用户 |
| `H` | 切换线程模式 |
| `c` | 显示完整命令行 |
| `q` | 退出 |

## 3.2 htop（增强版 top）

```bash
# 安装
sudo apt install htop
htop
```

htop 相比 top 的优势：
- 彩色显示
- 鼠标操作
- 垂直和水平滚动
- 树状视图（F5）
- 更直观的进程操作

# 4. 进程控制

## 4.1 前台与后台

```bash
command &                     # 后台运行
command > output.log 2>&1 &  # 后台运行并重定向输出
nohup command &               # 后台运行，退出终端后继续执行

Ctrl+Z                        # 暂停前台进程并放入后台
jobs                          # 列出后台任务
fg %1                         # 将后台任务1调回前台
bg %1                         # 在后台继续运行暂停的任务
```

## 4.2 进程优先级（Nice 值）

nice 值范围：**-20（最高优先级）到 19（最低优先级）**，默认 0。

```bash
# 启动时指定优先级
nice -n 10 ./script.sh        # 以较低优先级运行
nice --10 ./script.sh          # 以较高优先级运行（需要 root）

# 运行时修改优先级
renice -n 5 -p 1234           # 修改 PID 1234 的优先级
renice -n -5 -u alice         # 修改用户所有进程的优先级

# 查看优先级
top -o NI
ps -eo pid,ni,cmd
```

**注意**：
- 普通用户只能降低优先级（增加 nice 值）
- root 可以提升优先级（降低 nice 值）
- 实时调度优先级（SCHED_FIFO/SCHED_RR）使用 chrt 命令

## 4.3 信号（Signal）

```bash
kill -l                       # 列出所有信号
```

**常用信号**：

| 信号 | 编号 | 作用 |
|------|------|------|
| SIGHUP | 1 | 挂起（重新加载配置）|
| SIGINT | 2 | 中断（Ctrl+C）|
| SIGQUIT | 3 | 退出并生成 core dump |
| SIGKILL | 9 | 强制终止（不能捕获/忽略）|
| SIGTERM | 15 | 正常终止（默认，可被捕获）|
| SIGSTOP | 19 | 暂停执行（不能捕获/忽略）|
| SIGTSTP | 20 | 暂停（Ctrl+Z）|

**发送信号**：

```bash
kill -9 1234                  # 强制终止 PID 1234
kill -15 1234                 # 优雅终止（默认）
kill -1 1234                  # 重新加载配置（如 nginx -s reload）

# 其他工具
killall nginx                 # 杀死所有 nginx 进程
pkill -9 -u alice            # 强制终止用户所有进程
pkill -f "python server.py"  # 匹配命令行终止
```

## 4.4 进程终止的正确顺序

```bash
# 正确的终止顺序
kill PID                      # 1. 尝试 SIGTERM（优雅终止）
sleep 5                       # 2. 等待进程自行清理
kill -9 PID                   # 3. 如果还在运行，强制终止

# 或使用 timeout
timeout 10 command            # 10 秒后发送 SIGTERM
```

# 5. /proc 文件系统

`/proc` 由内核动态生成，每个运行中的进程在 `/proc/[pid]/` 下有对应目录：

```bash
/proc/1234/
├── cmdline          # 完整的命令行参数
├── cwd              # 符号链接到当前工作目录
├── environ          # 环境变量
├── exe              # 符号链接到可执行文件
├── fd/              # 打开的文件描述符
│   ├── 0            # stdin
│   ├── 1            # stdout
│   └── 2            # stderr
├── limits           # 资源限制
├── maps             # 内存映射
├── mountinfo        # 挂载信息
├── net/             # 网络信息
├── root             # 符号链接到根目录
├── sched            # 调度信息
├── smaps            # 详细内存使用
├── stack            # 内核栈回溯
├── status           # 进程状态摘要（最常用）
└── task/            # 线程信息
```

**实用操作**：

```bash
# 查看进程打开的文件（即使文件已被删除，仍可读取）
ls -la /proc/1234/fd/

# 查看进程内存映射
cat /proc/1234/maps

# 恢复被删除但仍被进程打开的文件
cp /proc/1234/fd/3 recovered.log
```

# 6. 守护进程

## 6.1 什么是守护进程

守护进程（Daemon）是在后台运行、不受终端控制的服务进程。名称通常以 `d` 结尾（sshd、httpd、crond）。

**传统创建守护进程的步骤**：
1. fork() 后父进程退出
2. 子进程调用 setsid() 创建新会话
3. 关闭文件描述符
4. 改变工作目录到 /
5. 重定向 stdin/stdout/stderr 到 /dev/null

## 6.2 现代守护进程管理

现代 Linux 使用 systemd 管理守护进程，不再需要手动编写复杂的 daemon 化代码：

```bash
# systemd service 示例
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
Restart=on-failure
User=myapp
Group=myapp

[Install]
WantedBy=multi-user.target
```

# 7. Linux 进程调度

## 7.1 CFS（完全公平调度器）

Linux 默认调度器 CFS（Completely Fair Scheduler，内核 2.6.23+）的核心思想：

- 将 CPU 时间建模为虚拟运行时（vruntime）
- 使用**红黑树**管理就绪队列，最左边的节点（vruntime 最小）优先执行
- 调度延迟目标（如 6ms）内确保每个可运行任务至少运行一次
- 权重（nice 值）影响时间片分配而非时间片大小

## 7.2 调度策略

```bash
# 查看和修改调度策略
chrt -p 1234                  # 查看 PID 1234 的调度策略
chrt -f 50 ./realtime_prog    # 以 SCHED_FIFO 优先级50运行
chrt -r 50 ./realtime_prog    # 以 SCHED_RR 优先级50运行
```

| 策略 | 说明 | 优先级范围 |
|------|------|-----------|
| SCHED_OTHER | 普通分时调度（默认）| nice -20..19 |
| SCHED_BATCH | 批处理（降低唤醒频率）| nice -20..19 |
| SCHED_IDLE | 极低优先级 | 不可调 |
| SCHED_FIFO | 实时 FIFO（直到阻塞或更高优先级抢占）| 1..99 |
| SCHED_RR | 实时轮转（时间片轮转）| 1..99 |
