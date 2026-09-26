
# 1. 日常运维规范

## 1.1 命令使用习惯

```
- 危险操作前先确认路径：ls 确认 → 再执行 rm
- 使用绝对路径或明确相对路径运行重要脚本
- 善用 Tab 补全减少拼写错误
- 定期查看 history，避免明文密码出现在历史记录
- 生产环境避免使用 -f 强制操作
- 命令输出重定向覆盖文件前确认
```

## 1.2 日志管理

```bash
# 设置合理的日志轮替策略
# 确保 /var/log 分区空间充足（单独分区最佳）
# 日志远程备份到集中日志服务器
# 关键日志：auth.log, syslog, kern.log
```

## 1.3 备份策略

```
3-2-1 备份原则：
- 3 份数据副本（1份生产 + 2份备份）
- 2 种不同介质（本地磁盘 + 远程存储）
- 1 份异地备份

备份验证比备份本身更重要
```

# 2. 性能调优最佳实践

## 2.1 系统层面

```bash
# SSD 优化挂载选项
defaults,noatime,nodiratime,discard

# 调整内核参数（/etc/sysctl.d/99-tuning.conf）
net.core.somaxconn = 65535
net.ipv4.tcp_fastopen = 3
vm.swappiness = 10

# I/O 调度器选择
# NVMe: none
# SSD: mq-deadline
# HDD: bfq（桌面）或 mq-deadline（服务器）
```

## 2.2 应用层面

- 数据库：使用 XFS 文件系统，禁用 atime
- Web 服务器：调整 worker 数量 = CPU 核心数
- 消息队列：使用独立磁盘分区，启用异步刷盘

# 3. 安全最佳实践

```bash
# 最小安装原则：不安装不必要的软件包
# 定期更新安全补丁
sudo apt update && sudo apt upgrade

# 关闭不需要的服务
sudo systemctl disable --now <service>

# SSH 安全：禁用密码认证，使用 ed25519 密钥
# 配置 fail2ban 防暴力破解
# 定期检查不需要的 SUID 文件
# 配置 iptables 默认 DROP 策略
```

# 4. 工具最佳实践

```bash
# 使用 ip 替代 ifconfig、route、arp
# 使用 ss 替代 netstat
# 使用 dstat 或 glances 替代多次执行独立命令
# 使用 htop 替代 top（交互更友好）
# 使用 rsync 替代 scp（增量传输）
# 使用 mosh 处理高延迟 SSH 连接
```

# 5. 高频面试题精选

## 5.1 系统管理

**Q: Linux 启动过程是怎样的？**
A: BIOS/UEFI → 引导加载程序（GRUB）→ 内核解压初始化 → initramfs → 挂载根文件系统 → init（systemd）→ 启动服务 → 登录界面

**Q: 什么是平均负载（load average）？如何判断是否正常？**
A: 平均负载是单位时间内系统处于可运行状态和不可中断睡眠状态的进程平均数量。通常 load < CPU 核心数×0.7 为正常，> CPU 核心数 表示需要关注。

**Q: 进程状态中 D 和 Z 分别表示什么？**
A: D（Uninterruptible Sleep）不可中断睡眠，通常等待 I/O；Z（Zombie）僵尸进程，已终止但未被父进程回收。

**Q: 如何找到最消耗 CPU 的进程？**
A: `ps aux --sort=-%cpu | head` 或 `top 按 P 排序` 或 `pidstat 1`。

## 5.2 文件系统

**Q: 硬链接和软链接的区别？**
A: 硬链接共享 inode，不能跨文件系统，不能链接目录；软链接有独立 inode，可以跨文件系统，可以链接目录，原文件删除后断裂。

**Q: inode 耗尽怎么办？**
A: 删除不需要的小文件。`find /dir -xdev -type f | wc -l` 统计文件数，`df -i` 查看 inode 使用率。

**Q: 误删文件如何恢复？**
A: 如果进程仍持有文件句柄，可从 `/proc/<PID>/fd/` 恢复。否则使用 extundelete（ext4）等工具（不一定成功，预防为主）。

## 5.3 内存管理

**Q: Linux 中 buffer 和 cache 的区别？**
A: Buffer 是块设备元数据缓存；Cache 是文件内容缓存（Page Cache）。两者都用于加速 I/O，内存紧张时可回收。

**Q: Swap 分区的作用？什么时候应该增大？**
A: Swap 用作内存扩展，将不常用页面换出。当物理内存不足时使用。区分内存压力场景：内存密集且允许换出（如批处理）可配置较多 Swap；延迟敏感服务则应减少 Swap 使用并增加物理内存。

## 5.4 网络

**Q: 如何查看某个端口被哪个进程占用？**
A: `ss -tuln | grep :8080` 或 `lsof -i :8080`。

**Q: 如何排查网络连接问题？**
A: 按层排查：ping 测试连通性 → traceroute 看路由 → telnet/nc 测试端口 → tcpdump 抓包分析 → 检查防火墙规则。

## 5.5 安全

**Q: 如何加固一台 Linux 服务器？**
A: 修改 SSH 默认端口、禁用密码登录、配置防火墙、关闭不必要的服务、最小权限原则、定期更新补丁、安装 fail2ban、配置日志审计、启用 SELinux/AppArmor。

## 5.6 综合

**Q: 查找 /var/log 下 7 天内被修改且大于 100M 的日志文件**
```bash
find /var/log -type f -name "*.log" -mtime -7 -size +100M
```

**Q: 统计 access.log 中每个 IP 的访问次数并取前 10**
```bash
awk '{ips[$1]++} END {for (ip in ips) print ips[ip], ip}' access.log | sort -rn | head -10
```

**Q: 检测 /etc/cron.hourly 下的脚本是否有语法错误**
```bash
for f in /etc/cron.hourly/*; do bash -n "$f" 2>&1 && echo "$f: OK" || echo "$f: SYNTAX ERROR"; done
```

# 6. 持续学习资源

| 资源 | 说明 |
|------|------|
| Linux Kernel Documentation | 内核官方文档 |
| man pages / info | Linux 命令手册 |
| The Linux Programming Interface | 系统编程经典书籍 |
| Brendan Gregg 的博客 | 性能分析领域权威 |
| LWN.net | Linux 内核和开源技术新闻 |
| Kernel Newbies | 内核新手社区 |

# 7. 常用参考速查

```bash
# 系统信息速查
uname -a                  # 所有系统信息
cat /etc/os-release       # 发行版信息
lsb_release -a            # 发行版详细信息
hostnamectl               # 主机名和系统信息

# 硬件信息
lscpu                     # CPU 信息
lsmem                     # 内存布局
lsblk                     # 块设备
lspci                     # PCI 设备
lsusb                     # USB 设备
dmidecode                 # DMI/SMBIOS 信息

# 系统限制
ulimit -a                 # 当前用户资源限制
cat /proc/cpuinfo         # CPU 详细信息
cat /proc/meminfo         # 内存详细信息
```
