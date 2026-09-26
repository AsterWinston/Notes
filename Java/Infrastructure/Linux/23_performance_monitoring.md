
# 1. 性能分析框架

## 1.1 USE 方法

由 Brendan Gregg 提出的系统性能分析方法论：

**USE（Utilization、Saturation、Errors）**：
- **Utilization（利用率）**：资源被使用的时间百分比
- **Saturation（饱和度）**：资源无法服务的排队程度
- **Errors（错误）**：错误计数

对于每个资源（CPU、内存、磁盘、网络），逐一检查这三项指标。

## 1.2 性能分析工具矩阵

| 领域 | 工具 |
|------|------|
| CPU | top, htop, mpstat, pidstat, perf, strace |
| 内存 | free, vmstat, top, valgrind, /proc/meminfo |
| 磁盘 | iostat, iotop, ioping, fio, /proc/diskstats |
| 网络 | ss, netstat, iperf3, nethogs, iftop, tcpdump |
| 系统整体 | dstat, sar, glances, nmon |

# 2. CPU 性能

## 2.1 关键指标

```bash
# 总览
top -n 1
# us: 用户空间占用 CPU 百分比
# sy: 内核空间占用 CPU 百分比
# ni: 改变过优先级的进程占用 CPU
# id: 空闲 CPU
# wa: I/O 等待
# hi: 硬中断
# si: 软中断
# st: 被虚拟机偷走的 CPU（steal）
```

**异常指标**：
| 指标 | 问题 |
|------|------|
| us > 70% | 用户态程序消耗 CPU |
| sy > 30% | 上下文切换或系统调用频繁 |
| wa > 20% | I/O 瓶颈（磁盘慢）|
| st > 10% | 虚拟机 CPU 过度分配 |

## 2.2 详细分析

```bash
# 每个 CPU 核心的使用率（查看负载是否均衡）
mpstat -P ALL 1

# 进程级 CPU 分析
pidstat 1                               # 所有进程
pidstat -p 1234 1                       # 特定进程

# 上下文切换
vmstat 1
# cs（context switch）过高表示线程/进程频繁切换

# 进程上下文切换
pidstat -w 1

# 中断分布
cat /proc/interrupts | head -20

# 运行队列
cat /proc/loadavg
# 1.50 1.20 1.00 2/345 67890
# ^     ^     ^    ^   ^
# |     |     |    |   最后创建的 PID
# |     |     |    当前运行/总线程数
# |     |     15 分钟平均负载
# |     5 分钟平均负载
# 1 分钟平均负载
```

**平均负载解读**：
- load average ≈ CPU 核心数 → 负载正常
- load average > CPU 核心数 × 1.5 → 需要关注
- load average 远高于 CPU 核心数 → 严重过载

## 2.3 perf — 性能分析器

```bash
# 安装
sudo apt install linux-tools-common linux-tools-$(uname -r)

# 采样分析
sudo perf record -a -g -- sleep 60       # 全系统采样 60 秒
sudo perf report                         # 生成报告

# 实时查看 CPU 事件
sudo perf top

# 统计 CPU 时钟周期和指令
sudo perf stat -e cycles,instructions ./myapp

# 特定进程
sudo perf record -p 1234 -g sleep 30
```

## 2.4 CPU 调优

```bash
# CPU 调频策略（笔记本/服务器）
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# powersave / ondemand / performance / conservative

# 设为性能模式
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 进程绑定 CPU（避免缓存抖动）
taskset -c 0-3 ./myapp                  # 限制到 0-3 核心
taskset -cp 0,2 1234                    # 将 PID 1234 绑定到核心 0 和 2

# 开启/关闭超线程
cat /sys/devices/system/cpu/smt/active
```

# 3. 内存性能

## 3.1 关键指标

```bash
# 内存总览
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           7.7G        3.1G        2.0G        156M        2.6G        4.2G
# Swap:          2.0G        200M        1.8G

# 可用内存 = free + buff/cache 中可回收部分
# 当 available 接近 0 时系统将开始使用 swap

# 详细内存
cat /proc/meminfo
```

## 3.2 内存分析

```bash
# 进程内存使用
ps aux --sort=-%mem | head -10
top -o %MEM

# 查看进程详细内存
cat /proc/1234/status | grep -E "VmRSS|VmSize|VmPeak"

# 内存映射
pmap -x 1234

# 大页使用
cat /proc/meminfo | grep Huge
```

## 3.3 内存调优

```bash
# 减少缓存/缓冲区过度使用
sysctl -w vm.vfs_cache_pressure=200

# 调整 swappiness
sysctl -w vm.swappiness=10

# 启用/禁用透明大页
echo always | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

# 4. 磁盘 I/O 性能

## 4.1 关键指标

```bash
# I/O 统计
iostat -x 1
# Device  r/s   w/s   rkB/s   wkB/s  await  svctm  %util
# sda    120.5  30.2  20480.5  5120.2  5.2   0.5     8.0
```

| 指标 | 含义 | 预警值 |
|------|------|--------|
| r/s, w/s | 每秒读写请求数 | 视设备而定 |
| await | 平均 I/O 等待时间（ms）| HDD > 20ms, SSD > 2ms |
| svctm | 平均 I/O 服务时间（ms）| 接近 await 说明无排队 |
| %util | 设备繁忙百分比 | > 80% 表示瓶颈 |

## 4.2 I/O 分析

```bash
# 进程级 I/O
sudo iotop -o                         # 只显示有 I/O 的进程
pidstat -d 1                          # 每个进程的读写速率

# I/O 延迟
ioping -c 10 /dev/sda                 # 测量 I/O 延迟

# I/O 调度器
cat /sys/block/sda/queue/scheduler

# 查看磁盘队列
cat /sys/block/sda/queue/nr_requests

# 文件系统缓存统计
cat /proc/meminfo | grep -E "Dirty|Writeback"
```

## 4.3 I/O 调优

```bash
# 调整 I/O 调度器（NVMe 推荐 none，HDD 推荐 mq-deadline 或 bfq）
echo bfq | sudo tee /sys/block/sda/queue/scheduler

# 提升请求队列深度
echo 512 | sudo tee /sys/block/sda/queue/nr_requests

# 调整脏页写回
sysctl -w vm.dirty_ratio=20
sysctl -w vm.dirty_background_ratio=10

# 文件系统挂载优化（减少 atime 更新）
mount -o noatime,nodiratime,data=ordered /dev/sda1 /mnt
```

# 5. 网络性能

## 5.1 关键指标

```bash
# 网络接口统计
ip -s link show eth0
# RX: bytes  packets  errors  dropped  overrun  mcast
# TX: bytes  packets  errors  dropped  carrier  collsns

# socket 统计
ss -s
# Total: 1234 (kernel)
# TCP:   456 (estab 234, closed 0, orphaned 12, synrecv 0, timewait 34/0)

# 网络带宽
iperf3 -c server -t 30              # TCP 带宽测试
iperf3 -c server -u -b 100M -t 30   # UDP 测试
```

## 5.2 网络调优

```bash
# 增大 TCP 缓冲区
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216

# 启用 TCP BBR（需要内核 4.9+）
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr

# 增大连接积压队列
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# 启用 TIME_WAIT 复用
sysctl -w net.ipv4.tcp_tw_reuse=1

# 调整网络中断亲和性
# 将网卡中断绑定到特定 CPU 核心
echo 1 > /proc/irq/<irq_number>/smp_affinity

# 开启 RPS/RFS（软件分发）—— 如果网卡不支持多队列
```

# 6. 全系统监控工具

## 6.1 dstat

```bash
# 安装
sudo apt install dstat

# 全系统统览
dstat -cdngy 1

# 自定义输出
dstat -c -d -n -l -p -s --top-cpu --top-io --top-mem 1
```

## 6.2 sar（系统活动报告）

```bash
# 安装
sudo apt install sysstat

# 实时查看
sar -u 1 5                        # CPU 使用（每秒采样，5次）
sar -r 1                          # 内存使用
sar -b 1                          # 磁盘 I/O
sar -n DEV 1                      # 网络接口
sar -S 1                          # Swap

# 查看历史数据
sar -u -f /var/log/sysstat/sa12   # 查看当月12号的数据
sar -r -s 10:00 -e 12:00          # 10点到12点的内存数据
```

## 6.3 glances

```bash
# 安装
sudo apt install glances

# 运行
glances                           # 全屏监控
glances -w                        # Web 服务器模式（http://localhost:61208）
glances -c server                 # 客户端模式连接到远程
```

## 6.4 nmon

```bash
# 安装
sudo apt install nmon

# 交互模式
nmon

# 数据收集模式
nmon -f -s 5 -c 120               # 每 5 秒采样，共 120 次（10 分钟）
```

# 7. 性能调优方法论

**系统化调优流程**：
```
1. 明确问题：延迟高？吞吐量低？资源耗尽？
2. 测量基准：收集当前数据作为基线
3. 定位瓶颈：使用 USE 方法逐一排查资源
4. 实施变更：一次只改一个参数
5. 验证效果：对比变更前后的指标
6. 文档记录：记录变更和影响
```

**常见瓶颈速查**：

| 症状 | 可能的瓶颈 |
|------|-----------|
| CPU 使用率 100%，load 高 | CPU 瓶颈 |
| CPU 使用率低，但 load 高 | I/O 瓶颈（多数进程在等 I/O）|
| wa 高 | 磁盘 I/O 瓶颈 |
| si/so 持续 > 0 | 内存不足 |
| 大量 TIME_WAIT | 短连接过多或连接池不回收 |
| 网络丢包 | 网卡/交换机/带宽瓶颈 |
