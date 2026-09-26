
# 1. 内核版本管理

## 1.1 版本号规则

```
主版本.次版本.补丁版本-发行版后缀
  6      8       0      -generic

主版本：重大变更（不向后兼容）
次版本：功能更新（向后兼容）
补丁版本：安全修复、小 bug 修复
```

**旧版（2.6.x 之前）**：偶数为稳定版，奇数为开发版
**现代**：不再用奇偶区分，所有版本都是稳定版

## 1.2 查看内核版本

```bash
uname -r                  # 内核发行版本
# 6.8.0-31-generic

uname -a                  # 全部信息
# Linux hostname 6.8.0-31-generic #33-Ubuntu SMP PREEMPT_DYNAMIC x86_64 x86_64 x86_64 GNU/Linux

cat /proc/version         # 内核编译信息
# Linux version 6.8.0-31-generic (buildd@lcy02-amd64-065) (x86_64-linux-gnu-gcc-13 (Ubuntu 13.2.0-23ubuntu4) 13.2.0, GNU ld (GNU Binutils for Ubuntu) 2.42) #33-Ubuntu SMP PREEMPT_DYNAMIC Fri Apr 26 10:57:28 UTC 2024

lsb_release -a            # 发行版信息
```

# 2. 内核模块管理

## 2.1 模块机制

Linux 内核是单内核但支持模块化加载，模块是可以在运行时加载/卸载的内核代码，用于支持硬件驱动、文件系统、网络协议等。

**优势**：
- 缩小内核体积
- 按需加载减少内存占用
- 无需重启即可添加/更新驱动

## 2.2 模块命令

```bash
# 查看已加载模块
lsmod
# Module                  Size  Used by
# nf_conntrack          172032  5
# xt_tcpudp              16384  6
# ext4                  720896  2
# ...

# 加载模块
sudo modprobe vfio-pci              # 自动解决依赖
sudo insmod /path/to/module.ko      # 直接插入（不解依赖）

# 卸载模块
sudo modprobe -r vfio-pci
sudo rmmod module_name

# 模块信息
modinfo nvidia
# filename:       /lib/modules/6.8.0-31-generic/kernel/drivers/video/nvidia.ko
# license:        MIT
# description:    NVIDIA Video Driver
# depends:        drm,drm_kms_helper
# parm:           modeset:int

# 列出所有可用模块
find /lib/modules/$(uname -r) -name "*.ko"
ls /lib/modules/$(uname -r)/kernel/
```

## 2.3 模块配置

```bash
# 模块别名和参数
ls /lib/modules/$(uname -r)/modules.alias
ls /lib/modules/$(uname -r)/modules.dep

# 配置文件
/etc/modules                  # 开机自动加载的模块列表（每行一个）
/etc/modprobe.d/*.conf        # 模块参数和别名配置

# 示例：传递模块参数
# /etc/modprobe.d/nvidia.conf
options nvidia modeset=1

# 黑名单（禁止自动加载）
# /etc/modprobe.d/blacklist.conf
blacklist nouveau
```

## 2.4 模块与内核的对应关系

```bash
# 查看模块对应的内核版本
modinfo ext4 | grep vermagic
# vermagic: 6.8.0-31-generic SMP preempt mod_unload modversions
```

**内核模块必须与运行中的内核版本匹配**，否则无法加载。

# 3. sysctl — 内核运行时参数

## 3.1 查看和修改

```bash
# 查看所有参数
sysctl -a | less

# 查看特定参数
sysctl net.ipv4.tcp_tw_reuse
# net.ipv4.tcp_tw_reuse = 1

# 临时修改
sudo sysctl -w vm.swappiness=10

# 查看 /proc/sys 中的值
cat /proc/sys/vm/swappiness
# 10

# 永久写入
# /etc/sysctl.conf 或 /etc/sysctl.d/*.conf
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf

# 重新加载
sudo sysctl -p
sudo sysctl --system
```

**注意**：sysctl 下的参数对应 `/proc/sys/` 目录结构：
```
sysctl net.ipv4.tcp_tw_reuse
    = /proc/sys/net/ipv4/tcp_tw_reuse
```

## 3.2 重要参数分类

**网络相关**（最常调整）：
```bash
# 网络安全
net.ipv4.ip_forward = 1                    # 启用 IP 转发（路由器/NAT）
net.ipv4.conf.all.rp_filter = 1            # 反向路径过滤
net.ipv4.icmp_echo_ignore_broadcasts = 1   # 忽略 ICMP 广播

# TCP 优化
net.ipv4.tcp_tw_reuse = 1                  # 复用 TIME_WAIT 连接
net.ipv4.tcp_fin_timeout = 15              # FIN-WAIT-2 超时
net.ipv4.tcp_keepalive_time = 300          # TCP 保活时间
net.ipv4.tcp_congestion_control = bbr      # 拥塞控制算法
net.core.somaxconn = 65535                 # 最大连接积压
net.core.netdev_max_backlog = 5000         # 网卡接收队列

# 内存相关
net.core.rmem_max = 16777216               # 接收缓冲区最大值
net.core.wmem_max = 16777216               # 发送缓冲区最大值
```

**内存与 VM**：
```bash
vm.swappiness = 10                         # Swap 使用倾向
vm.dirty_ratio = 20                        # 脏页上限（同步写）
vm.dirty_background_ratio = 10             # 脏页后台写回阈值
vm.vfs_cache_pressure = 100                # 回收 dentry/inode 倾向
vm.overcommit_memory = 1                   # 内存过载策略
vm.min_free_kbytes = 65536                 # 保留空闲内存
```

**内核**：
```bash
kernel.pid_max = 65536                     # 最大 PID
kernel.hostname = hostname                 # 主机名
kernel.core_uses_pid = 1                   # core dump 文件名含 PID
kernel.msgmax = 65536                      # 消息队列最大值
```

## 3.3 持久化配置

```bash
# /etc/sysctl.d/99-tuning.conf
# 网络优化
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0

# 虚拟内存
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5

# 文件系统
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288

# 应用
sudo sysctl --system
```

# 4. /sys 文件系统

`/sys` 暴露内核对象和设备信息，常用于硬件管理和调优：

```bash
/sys/
├── block/                    # 块设备
│   ├── sda/
│   │   ├── queue/            # I/O 调度参数
│   │   └── device/           # 设备属性
├── class/                    # 按类别分类（net, block, input...）
│   ├── net/                  # 网络接口
│   └── dmi/                  # 硬件信息
├── devices/                  # 设备拓扑
├── fs/                       # 文件系统
│   ├── cgroup/               # cgroups
│   └── ext4/                 # ext4 参数
├── kernel/                   # 内核参数
│   └── mm/                   # 透明大页等
├── power/                    # 电源管理
└── firmware/                 # 固件信息
```

**实用操作**：
```bash
# 查看 CPU 信息
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
ls /sys/devices/system/cpu/cpu0/cache/

# 查看磁盘信息
cat /sys/block/sda/size                 # 磁盘扇区数
cat /sys/block/sda/queue/rotational     # 0=SSD, 1=HDD

# 设置 S.M.A.R.T. 状态
cat /sys/block/sda/device/model
```

# 5. 内核消息与调试

## 5.1 dmesg — 内核环形缓冲区

```bash
dmesg                              # 所有内核消息
dmesg -T                           # 人类可读时间戳
dmesg -l err                       # 仅错误
dmesg | grep -i "error\|fail"
dmesg -w                           # 实时跟踪（类似 tail -f）
```

## 5.2 内核恐慌（Kernel Panic）

当内核遇到无法恢复的错误时，会触发 kernel panic：

```bash
# 配置 panic 后的行为
# 自动重启（设置 panic 超时秒数）
sysctl -w kernel.panic=10          # panic 后 10 秒自动重启

# 保存 panic 信息
# 需要配置 kdump 服务
sudo apt install kdump-tools
# /etc/default/kdump-tools
```

# 6. 编译内核

## 6.1 为什么编译内核

- 启用特定硬件支持
- 实验内核新特性
- 性能优化（定制编译选项）
- 安全加固（减少攻击面）

## 6.2 编译流程

```bash
# 1. 下载内核源码
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.8.tar.xz
tar -xf linux-6.8.tar.xz
cd linux-6.8

# 2. 配置
make defconfig                        # 默认配置
make menuconfig                       # 图形化菜单配置
make localmodconfig                   # 基于当前加载模块的最小配置

# 3. 编译
make -j$(nproc)                       # 并行编译

# 4. 安装模块
sudo make modules_install

# 5. 安装内核
sudo make install

# 6. 更新启动引导
sudo update-grub

# 7. 重启到新内核
sudo reboot
```

## 6.3 清理

```bash
make clean               # 清理编译产物
make mrproper            # 彻底清理（含配置文件）
```
