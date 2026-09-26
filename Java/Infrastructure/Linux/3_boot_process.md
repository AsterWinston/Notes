
# 1. 整体启动阶段

Linux 系统的完整启动分为以下主要阶段：

```
硬件上电
    ↓
BIOS / UEFI（固件阶段）
    ↓
引导加载程序（GRUB 2）
    ↓
Linux 内核初始化
    ↓
init 进程启动（systemd）
    ↓
系统服务启动 → 登录界面/命令行
```

# 2. 固件阶段：BIOS vs UEFI

## 2.1 BIOS（Legacy）

- 读取 MBR（主引导记录，磁盘第一个 512 字节）
- MBR 中包含引导代码（446 字节）+ 分区表（64 字节）+ 签名（0x55AA）
- 寻找活动分区，加载该分区的 VBR（卷引导记录）
- 限制：MBR 最多 4 个主分区，磁盘最大 2TB

## 2.2 UEFI

- 读取 ESP（EFI System Partition，FAT32 格式）
- 执行 `/EFI/BOOT/BOOTX64.EFI` 或 NVRAM 中记录的引导程序
- 支持 GPT 分区表（无 2TB 限制，支持 128 个分区）
- 支持安全启动（Secure Boot）：验证引导程序签名

## 2.3 二者的启动流程对比

```
BIOS:
POST → BIOS → 读取 MBR → 引导加载程序阶段1 → 阶段2 → 内核

UEFI:
POST → UEFI → 读取 ESP → 直接加载引导程序（efi文件）→ 内核
```

# 3. 引导加载程序：GRUB 2

## 3.1 作用

GRUB（GRand Unified Bootloader）是目前 Linux 发行版最常用的引导程序，负责加载内核和 initramfs 到内存并传递启动参数。

## 3.2 GRUB 启动过程

```
BIOS/UEFI 加载 boot.img（512 字节，位于 MBR 或 GPT 的专用分区）
    ↓
boot.img 加载 core.img（位于 MBR 后的扇区或 BIOS Boot Partition）
    ↓
core.img 包含文件系统驱动，能够读取 /boot/grub/
    ↓
显示菜单，加载用户选择的内核和 initramfs
    ↓
执行 boot 命令，跳转到内核入口
```

## 3.3 GRUB 配置文件

主配置文件：`/boot/grub/grub.cfg`（**不要手动编辑**，由 `update-grub` 或 `grub-mkconfig` 生成）

自定义配置写入：`/etc/default/grub`

```bash
# /etc/default/grub
GRUB_DEFAULT=0                # 默认启动项（0表示第一个）
GRUB_TIMEOUT=5                # 倒计时秒数
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"  # 内核启动参数
GRUB_CMDLINE_LINUX=""         # 无论如何都附加的内核参数
```

修改后执行：
```bash
# Debian/Ubuntu
update-grub

# RHEL/CentOS
grub2-mkconfig -o /boot/grub2/grub.cfg
```

# 4. 内核初始化阶段

## 4.1 内核解压

- GRUB 将压缩的内核映像（`vmlinuz`）加载到内存
- 内核入口 `startup_32()` / `startup_64()` 进行架构相关初始化
- 解压内核到高端内存（`decompress_kernel()`）
- 跳转到解压后的内核入口 `start_kernel()`

## 4.2 start_kernel() 初始化流程

`start_kernel()` 是内核初始化 C 语言入口（在 `init/main.c` 中），执行顺序：

1. **关中断**：`local_irq_disable()`
2. **早期架构初始化**：`setup_arch()` — 解析启动参数、初始化页表、内存布局
3. **内存管理初始化**：`mm_init()` — 建立内存分配器
4. **中断与定时器**：`trap_init()`（异常处理）、`init_IRQ()`（中断控制器）、`sched_init()`（调度器）、`time_init()`（系统时钟）
5. **控制台初始化**：`console_init()` — 输出启动日志
6. **initramfs 挂载**：`populate_rootfs()` — 解压 initramfs 到根目录
7. **RCU 与锁机制**：`rcu_init()`、`lockdep_init()`
8. **创建初始进程**：`arch_call_rest_init()` → `kernel_init()` → 创建 PID=1 的 init 进程

## 4.3 initramfs/initrd

**作用**：在真正的根文件系统挂载前，提供一个最小的临时文件系统，包含加载根文件系统所需的驱动模块（如磁盘控制器驱动、文件系统驱动、LVM 等）。

**机制**：
```
内核 → 解压 initramfs-<version>.img 为 tmpfs
     → 执行 /init 脚本
     → 加载必要内核模块（磁盘、文件系统、加密等）
     → 挂载真正的根文件系统
     → pivot_root() 切换到根文件系统
     → 执行 /sbin/init
```

查看 initramfs 内容：
```bash
lsinitramfs /boot/initrd.img-$(uname -r)
# 或
zcat /boot/initrd.img-$(uname -r) | cpio -idmv
```

# 5. init 进程启动

内核初始化完成后，启动第一个用户空间进程（PID=1）。

## 5.1 init 程序的演变

| 类型 | 特点 | 使用场景 |
|------|------|----------|
| SysV init | 串行启动脚本，依赖 /etc/inittab 和 /etc/rc.d/ | 旧发行版 |
| Upstart | 事件驱动，并行启动（Ubuntu 9.10-14.10） | 已被取代 |
| systemd | 并行启动，socket/dbus 激活，兼容性最好 | 现代主流发行版 |

## 5.2 systemd 启动过程

```bash
/sbin/init → systemd（符号链接）
```

systemd 启动目标的默认目标（target）：

```
systemd
    ↓
default.target（通常链接到 graphical.target 或 multi-user.target）
    ↓
sysinit.target + basic.target（基础系统初始化）
    ├── local-fs.target（挂载本地文件系统）
    ├── swap.target（激活交换分区）
    ├── systemd-modules-load.service（加载内核模块）
    ├── systemd-sysctl.service（设置 sysctl 参数）
    └── systemd-tmpfiles-setup.service（创建临时文件/目录）
    ↓
multi-user.target（多用户命令行模式）
    ├── network.target 等网络服务
    ├── sshd.service（SSH 服务）
    └── crond.service 等服务
    ↓
graphical.target（图形界面，可选）
    ├── display-manager.service（如 gdm3、lightdm）
    └── 桌面环境启动
```

查看当前目标：
```bash
systemctl get-default
# multi-user.target 或 graphical.target
```

# 6. 启动日志分析

## 6.1 查看启动日志

```bash
# 内核日志（dmesg）
dmesg | less
journalctl -k           # 仅内核日志
journalctl -b           # 本次启动的所有日志

# 分析启动耗时
systemd-analyze         # 总启动时间
systemd-analyze blame   # 各服务的启动耗时
systemd-analyze critical-chain  # 关键启动链
systemd-analyze plot > boot.svg # 生成启动时间图
```

## 6.2 常见启动问题排查

| 症状 | 排查方法 |
|------|----------|
| 内核 panic | 查看 dmesg 最后输出，通常是驱动或根文件系统问题 |
| GRUB 无法启动 | Live CD 启动，chroot 修复 GRUB |
| 根文件系统挂载失败 | 检查 /etc/fstab，使用 initramfs shell 调试 |
| systemd 超时 | `journalctl -p err -b` 查看错误，禁用失败的服务 |

# 7. 内核启动参数

可以在 GRUB 菜单按 `e` 编辑启动参数，常见参数：

| 参数 | 作用 |
|------|------|
| `quiet` | 减少内核启动信息输出 |
| `splash` | 显示开机动画 |
| `single` | 单用户模式（救援模式） |
| `init=/bin/bash` | 直接启动 bash 替代 init（密码恢复） |
| `ro` | 挂载根文件系统为只读 |
| `rw` | 挂载根文件系统为读写 |
| `nomodeset` | 禁用内核模式设置（解决显卡黑屏） |
| `acpi=off` | 禁用 ACPI（解决 ACPI 兼容性问题） |
| `mem=4G` | 限制可用内存大小 |
| `console=ttyS0` | 将控制台输出重定向到串口 |
| `systemd.unit=rescue.target` | 启动到救援模式 |
