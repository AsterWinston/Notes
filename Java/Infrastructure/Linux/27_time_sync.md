
# 1. Linux 时间体系

## 1.1 硬件时钟 vs 系统时钟

| 类型 | 说明 | 存储位置 |
|------|------|----------|
| 硬件时钟（RTC） | 主板电池供电，关机后继续运行 | 硬件 CMOS |
| 系统时钟（System Clock） | 内核维护，开机时从 RTC 读取 | 内核内存 |

```bash
# 查看硬件时钟
sudo hwclock --show
sudo hwclock --show --verbose

# 查看系统时钟
date
timedatectl
```

## 1.2 UTC vs 本地时间

- **UTC（协调世界时）**：全球统一的时间标准
- **本地时间（Local Time）** = UTC + 时区偏移

**建议**：硬件时钟使用 UTC，系统在启动时转换到本地时间。Linux 默认使用此方式。

```bash
# 查看配置
timedatectl
#                Local time: Mon 2025-05-12 10:30:00 CST
#            Universal time: Mon 2025-05-12 02:30:00 UTC
#                  RTC time: Mon 2025-05-12 02:30:00
#                 Time zone: Asia/Shanghai (CST, +0800)
# System clock synchronized: yes
#               NTP service: active
#           RTC in local TZ: no
```

# 2. timedatectl — 时间管理

## 2.1 基本用法

```bash
timedatectl                           # 查看时间状态
timedatectl status                    # 同上

# 设置时区
timedatectl list-timezones | grep Shanghai  # 查找时区
sudo timedatectl set-timezone Asia/Shanghai

# 设置时间（不推荐手动，建议用 NTP 同步）
sudo timedatectl set-time "2025-05-12 10:30:00"

# 启用/禁用 NTP 同步
sudo timedatectl set-ntp true
sudo timedatectl set-ntp false
```

## 2.2 时区管理

```bash
# 时区文件
ls /usr/share/zoneinfo/               # 所有时区信息

# 传统方式设置时区
sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 查看当前时区
readlink -f /etc/localtime
# /usr/share/zoneinfo/Asia/Shanghai

date +%Z                              # 显示时区缩写
date +%z                              # 显示 UTC 偏移（如 +0800）
```

# 3. NTP 与 chrony

## 3.1 NTP（Network Time Protocol）

NTP 通过层级结构进行时间同步：

```
层0（Stratum 0）：原子钟、GPS（参考时钟）
    ↓
层1（Stratum 1）：直接连接层0的服务器
    ↓
层2（Stratum 2）：从层1同步
    ↓
...
```

## 3.2 chrony（推荐）

chrony 是 Ubuntu 18.04+ 和 RHEL 8+ 的默认 NTP 实现，比旧的 ntpdate/ntpd 更快更准确。

**特点**：
- 更快的时间同步（尤其对间歇性网络）
- 更好的网络延迟抖动处理
- 更低的 CPU 开销

```bash
# 安装
sudo apt install chrony

# 查看状态
chronyc tracking
# Reference ID    : A9FEA9FE (ntp.aliyun.com)
# Stratum         : 2
# Ref time (UTC)  : Mon May 12 02:30:00 2025
# System time     : 0.000123456 seconds slow of NTP time
# Last offset     : +0.000045678 seconds
# RMS offset      : 0.000567890 seconds
# Frequency       : 10.234 ppm slow
# Residual freq   : 0.001 ppm
# Skew            : 0.010 ppm
# Root delay      : 0.023456 seconds
# Root dispersion : 0.001234 seconds
# Update interval : 64.0 seconds
# Leap status     : Normal

# 查看 NTP 源
chronyc sources -v
# 210 Number of sources = 4
# .-- Source mode  '^' = server, '=' = peer, '#' = local clock
# / .- Source state  '*' = current synced, '+' = combined , '-' = not combined
# |   /   .- Accept state  '?' = unreachable, 'x' = time may be in error
# v                                     .- DNS resolve delay (msec)
#                                        .- NTP source reachability
# MS Name/IP address         Stratum Poll Reach LastRx Last sample
# ^- ntp.aliyun.com               2   6   377    45   -123us[ -123us] +/-   21ms
# ^+ ntp.tencent.com              2   6   377    46   +456us[+456us] +/-   15ms
# ^* time.cloudflare.com          3   6   377    44   +234us[+234us] +/-   12ms

# 列出所有 NTP 源
chronyc sources
```

## 3.3 chrony 配置

```bash
# /etc/chrony/chrony.conf

# NTP 服务器（国内常用）
pool ntp.aliyun.com iburst
pool ntp.tencent.com iburst
pool time.cloudflare.com iburst
pool cn.pool.ntp.org iburst

# 允许客户端从本机同步（NTP 服务器模式）
# allow 192.168.0.0/16

# 时间调整设置
makestep 1 -1                    # 启动时允许一次性大幅调整

# 日志
log tracking measurements statistics
logdir /var/log/chrony
```

## 3.4 手动时间同步

```bash
# 强制同步
sudo chronyc -a makestep

# 立即检查时间偏差
sudo chronync tracking

# 传统方式（已弃用）
sudo ntpdate -u ntp.aliyun.com   # 需要安装 ntpdate
```

# 4. systemd-timesyncd

用于简单场景的轻量级 NTP 客户端：

```bash
# 配置文件
# /etc/systemd/timesyncd.conf
[Time]
NTP=ntp.aliyun.com ntp.tencent.com
FallbackNTP=ntp.ubuntu.com
RootDistanceMaxSec=5

# 状态查看
timedatectl status
# NTP service: active

# 手动触发同步
sudo systemctl restart systemd-timesyncd
```

**chrony vs systemd-timesyncd**：
- chrony：完整功能，适合服务器
- systemd-timesyncd：轻量，适合桌面/容器

# 5. 常见问题排查

```bash
# 时间不同步
timedatectl status                    # 检查 NTP 服务是否启用
chronyc tracking                      # 查看同步状态
chronyc sources -v                    # 检查 NTP 源可达性

# 防火墙影响 NTP（UDP 123 端口）
sudo iptables -A OUTPUT -p udp --dport 123 -j ACCEPT

# 时区错误
date
sudo timedatectl set-timezone Asia/Shanghai

# 硬件时钟与系统时钟差异大
sudo hwclock --systohc               # 系统时间写入硬件时钟
sudo hwclock --hctosys               # 硬件时钟写入系统时间
```

# 6. NTP 安全

```bash
# chrony 访问控制
# /etc/chrony/chrony.conf
# 允许特定网段访问
allow 192.168.0.0/24
# 拒绝所有
deny all

# NTP 认证（防止时间欺骗）
# 生成密钥
chronyc keygen 1 MD5 mypassword
# 在配置中添加
keyfile /etc/chrony/chrony.keys
```
