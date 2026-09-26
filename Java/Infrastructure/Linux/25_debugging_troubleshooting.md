
# 1. 调试方法论

## 1.1 故障排查通用流程

```
1. 明确症状：发生了什么？何时开始？影响范围？
2. 收集信息：日志、监控数据、变更历史
3. 提出假设：可能的原因列表
4. 验证假设：用最小影响的方式测试每个假设
5. 定位根因：确认根本原因
6. 修复验证：实施修复并验证
7. 文档记录：记录故障过程和解决方案
```

## 1.2 分层排查

```bash
# 从上层到下层排查
1. 应用层：检查应用日志、进程状态
2. 系统层：CPU、内存、磁盘、网络
3. 内核层：dmesg、系统调用跟踪
4. 硬件层：dmesg 硬件错误、SMART 状态
```

# 2. strace — 系统调用跟踪

## 2.1 基本用法

```bash
# 跟踪命令的系统调用
strace ls /tmp

# 附加到运行的进程
strace -p 1234

# 常用选项
strace -f command                        # 跟踪子进程（fork）
strace -e trace=open,read,write command  # 只跟踪特定系统调用
strace -c command                        # 统计系统调用次数和时间
strace -t command                        # 显示时间戳
strace -T command                        # 显示每个调用的耗时
```

## 2.2 实用案例

```bash
# 诊断 "Permission denied"
strace -e trace=access,open cat /etc/shadow

# 找出配置文件读取路径
strace -e trace=open nginx -t 2>&1 | grep -E "open.*\.conf"

# 查看程序卡在哪里
sudo strace -p $(pgrep myapp)

# 统计最耗时的系统调用
strace -c -p $(pgrep myapp)

# 跟踪网络相关调用
strace -e trace=network curl http://example.com

# 跟踪信号
strace -e trace=signal command
```

## 2.3 输出解读

```bash
open("/etc/passwd", O_RDONLY) = 3
read(3, "root:x:0:0:root:/root:/bin/bash\n"..., 4096) = 1234
write(1, "root:x:0:0:root:/root:/bin/bash\n", 31) = 31
close(3) = 0
```

**常见错误返回**：
| 返回值 | 含义 |
|--------|------|
| -1 (ENOENT) | 文件/目录不存在 |
| -1 (EACCES) | 权限不足 |
| -1 (EAGAIN) | 资源暂时不可用（如非阻塞读无数据）|
| -1 (ECONNREFUSED) | 连接被拒绝 |

# 3. ltrace — 库函数调用跟踪

```bash
# 跟踪库函数调用
ltrace ls

# 显示时间戳
ltrace -t command

# 统计调用
ltrace -c command

# 特定函数
ltrace -e malloc+free command

# 附加到进程
ltrace -p 1234
```

**strace vs ltrace**：
- strace：跟踪**系统调用**（内核接口）
- ltrace：跟踪**库函数**调用（如 printf、malloc）

# 4. GDB — GNU 调试器

## 4.1 基本用法

```bash
# 编译时需加 -g 选项生成调试符号
gcc -g -o myapp myapp.c

# 启动 GDB
gdb ./myapp
gdb ./myapp core           # 分析 core dump 文件
gdb ./myapp 1234           # 附加到运行中的进程
```

## 4.2 GDB 常用命令

```bash
# 断点
break main                  # 在 main 函数设置断点
break file.c:42             # 在文件的第 42 行
break func_name             # 在函数处
info breakpoints            # 列出断点
delete 1                    # 删除断点 1

# 运行控制
run                         # 开始运行
run arg1 arg2               # 带参数运行
continue                    # 继续执行
next                        # 下一行（不进入函数）
step                        # 下一步（进入函数）
finish                      # 执行到函数返回

# 检查
print var                   # 打印变量值
print *ptr                  # 打印指针指向内容
backtrace                   # 栈回溯（类似 bt）
info locals                 # 所有局部变量
info registers              # 寄存器值
list                        # 显示源代码

# 其他
set var x=42                # 修改变量值
thread apply all bt         # 所有线程的栈回溯
quit                        # 退出
```

## 4.3 Core Dump 分析

```bash
# 启用 core dump
ulimit -c unlimited                     # 当前 Shell
# 或永久配置
echo "* soft core unlimited" >> /etc/security/limits.conf

# core dump 位置（默认当前目录，或由 kernel.core_pattern 指定）
cat /proc/sys/kernel/core_pattern
# |/usr/share/apport/apport %p %s %c %P

# 分析 core dump
gdb ./myapp /var/crash/core.myapp.1234
```

# 5. lsof — 打开文件列表

## 5.1 基本用法

```bash
lsof                              # 列出所有打开的文件
lsof -p 1234                      # 特定进程的文件
lsof -u alice                     # 特定用户的文件
lsof -i :80                       # 监听或连接 80 端口的进程
lsof -i TCP:1-1024                # 端口范围
lsof file.txt                     # 谁在打开这个文件
lsof +D /path                     # 目录下所有打开的文件
```

## 5.2 实用场景

```bash
# 查看端口被谁占用
lsof -i :8080
# COMMAND   PID   USER   FD   TYPE  DEVICE  SIZE/OFF  NODE  NAME
# java     12345 alice  102u  IPv6  123456  0t0       TCP   *:8080 (LISTEN)

# 文件不能卸载（device busy），找到占用进程
lsof /mnt/data

# 查看进程的网络连接
lsof -i -a -p 1234

# 查看 Unix 域套接字
lsof -U

# 恢复被删除但仍被进程持有的文件
# /proc/<PID>/fd/ 下仍有文件句柄
ls -la /proc/1234/fd/
cp /proc/1234/fd/3 recovered.log
```

# 6. tcpdump — 网络抓包

## 6.1 基本用法

```bash
# 抓取所有包
sudo tcpdump

# 选择接口
sudo tcpdump -i eth0

# 不解析主机名（更快更清晰）
sudo tcpdump -n

# 写入文件（供后续分析）
sudo tcpdump -w capture.pcap

# 从文件读取
tcpdump -r capture.pcap

# 限定包数量
sudo tcpdump -c 100

# 显示包内容（ASCII）
sudo tcpdump -A

# 显示包内容（HEX + ASCII）
sudo tcpdump -X
```

## 6.2 过滤表达式

```bash
# 主机过滤
sudo tcpdump host 192.168.1.100
sudo tcpdump src host 192.168.1.100
sudo tcpdump dst host 192.168.1.100

# 端口过滤
sudo tcpdump port 80
sudo tcpdump src port 80
sudo tcpdump dst port 443

# 协议过滤
sudo tcpdump tcp
sudo tcpdump udp
sudo tcpdump icmp

# 组合条件
sudo tcpdump "tcp port 80 and host 192.168.1.100"
sudo tcpdump "tcp port 80 or udp port 53"

# 高级过滤
sudo tcpdump "tcp[tcpflags] & (tcp-syn) != 0"              # SYN 包
sudo tcpdump "tcp[tcpflags] & (tcp-rst) != 0"              # RST 包
sudo tcpdump "icmp[icmptype] = 8"                          # ping 请求

# 抓取 HTTP 请求
sudo tcpdump -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

## 6.3 实用场景

```bash
# 查看 HTTP 请求
sudo tcpdump -A -i any port 80

# 查看 DNS 查询
sudo tcpdump -n -i any port 53

# 抓取两个主机之间的所有通信
sudo tcpdump -i any host 10.0.0.1 and host 10.0.0.2

# 查看 TCP 三次握手
sudo tcpdump -n -i any "tcp port 80 and (tcp[tcpflags] & (tcp-syn|tcp-ack) != 0)"

# Web 服务器排错
sudo tcpdump -i any -n 'tcp port 80 and tcp[tcpflags] & (tcp-syn) != 0'
```

# 7. curl — HTTP 调试

```bash
# 查看 HTTP 响应头
curl -I https://example.com

# 详细输出（查看 SSL 证书、请求/响应头）
curl -v https://example.com

# 查看完整请求和响应
curl --trace-ascii trace.txt https://example.com

# 测试延迟
curl -w "TCP handshake: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" -o /dev/null -s https://example.com

# 指定源 IP
curl --interface 192.168.1.100 https://example.com

# 使用 SOCKS 代理
curl --socks5-hostname localhost:1080 https://example.com
```

# 8. 救援与恢复模式

## 8.1 进入救援模式

```bash
# 方法1：GRUB 菜单按 e，在 linux 行末尾添加 single
# 方法2：添加 systemd.unit=rescue.target
# 方法3：添加 init=/bin/bash
```

## 8.2 救援操作

```bash
# 以 rw 重新挂载根文件系统（在 init=/bin/bash 模式下）
mount -o remount,rw /

# 修复 GRUB
grub-install /dev/sda
update-grub

# 修复损坏的包
dpkg --configure -a
apt-get install -f

# 重置密码
passwd root

# 修复文件系统
fsck /dev/sda1
```

# 9. 系统运行时间

```bash
uptime
# 10:30:00 up 30 days,  2:15,  2 users,  load average: 0.15, 0.10, 0.05

last reboot               # 最近的启动记录
who -b                    # 上次启动时间
```
