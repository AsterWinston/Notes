
# 1. 整体分层结构

Linux 系统从顶层到底层可以分为四个层次：

```
User Applications（用户应用程序）
        ↕
System Libraries / Shell（系统库与 Shell，如 glibc、bash）
        ↕      ← 系统调用接口（System Call Interface）
Linux Kernel（内核）
        ↕      ← 硬件接口（设备驱动）
Hardware（硬件：CPU、内存、磁盘、网卡等）
```

# 2. 内核态与用户态

## 2.1 为什么需要区分

CPU 指令集分为**特权指令**和**非特权指令**。如果所有程序都能直接执行特权指令（如修改内存映射、关闭中断），系统将极不稳定且不安全。因此引入**特权级（Ring）** 隔离：

| 环级 | 名称 | 运行内容 | 权限 |
|------|------|----------|------|
| Ring 0 | 内核态（Kernel Mode） | 内核代码、设备驱动 | 全部权限：可执行任何 CPU 指令、访问任何内存地址 |
| Ring 3 | 用户态（User Mode） | 应用程序、库代码 | 受限：不能直接访问硬件、不能修改内存映射、不能执行特权指令 |

（Ring 1 和 Ring 2 在 x86 中基本未使用，部分驱动/虚拟化场景使用）

## 2.2 切换过程

当用户态程序需要执行特权操作时，通过**系统调用**陷入内核：

```
用户态调用函数（如 read()）
    → glibc 封装
    → 触发软中断或 syscall 指令
    → CPU 切换到 Ring 0
    → 执行内核 sys_read()
    → 返回结果
    → CPU 切回 Ring 3
    → 返回用户态
```

每次上下文切换都有性能开销，这是系统调用的主要成本来源。

# 3. 系统调用

## 3.1 什么是系统调用

系统调用（syscall）是用户态程序请求内核服务的唯一合法途径。Linux 提供约 300-400 个系统调用（取决于内核版本和架构）。

## 3.2 常见系统调用分类

| 分类 | 典型系统调用 |
|------|-------------|
| 进程控制 | fork()、execve()、exit()、wait4()、clone() |
| 文件操作 | open()、read()、write()、close()、lseek()、mmap() |
| 网络通信 | socket()、connect()、bind()、listen()、accept()、send() |
| 内存管理 | brk()、mmap()、munmap()、mprotect() |
| 权限管理 | chmod()、chown()、setuid()、setgid() |
| 时间相关 | clock_gettime()、nanosleep()、gettimeofday() |
| 信号处理 | signal()、kill()、signalfd() |

## 3.3 系统调用 vs 库函数

```
应用程序
    ↓
库函数（如 printf()） ← glibc
    ↓
系统调用（如 write()） ← 内核
```

- **库函数**运行在用户态，可以完全在用户态完成（如 strcpy()），也可以封装系统调用（如 malloc() 内部调用 brk()/mmap()）
- **系统调用**必然涉及内核态切换，开销远大于普通函数调用

# 4. Linux 内核子系统

## 4.1 进程调度子系统

负责管理 CPU 时间分配。Linux 使用**完全公平调度器（CFS, Completely Fair Scheduler）**：

- 将 CPU 时间视为一种资源，按权重公平分配给各进程
- 使用红黑树（Red-Black Tree）管理就绪队列
- 支持多种调度策略：
  - `SCHED_OTHER`：普通分时调度（默认）
  - `SCHED_FIFO`：实时先进先出
  - `SCHED_RR`：实时轮转
  - `SCHED_BATCH`：批处理调度（降低唤醒频率）
  - `SCHED_IDLE`：空闲调度（极低优先级）

## 4.2 内存管理子系统

- 虚拟内存管理：为每个进程提供独立的 4GB（32位）/ 128TB（64位）虚拟地址空间
- 页分配与回收：伙伴系统（Buddy System）+ slab 分配器
- 页面置换：LRU 链表管理
- 内存映射：mmap 实现文件与内存的直接映射
- KSM（Kernel Samepage Merging）：合并相同内容的物理页面

## 4.3 虚拟文件系统（VFS）

VFS 是 Linux 文件系统的抽象层，使得用户空间可以用同一套 API 操作不同文件系统：

```
用户：open() / read() / write()
    ↓
VFS 层：vfs_open() / vfs_read() / vfs_write()
    ↓
具体文件系统：ext4_read() / xfs_write() / ntfs_read()
    ↓
块设备层（Block Layer）→ I/O 调度器 → 设备驱动
```

VFS 支持以下对象模型：
- **super_block**：已挂载文件系统的实例信息
- **inode**：文件的元数据（大小、权限、时间戳、数据块指针）
- **dentry**：目录项，负责路径名与 inode 的映射（含缓存加速）
- **file**：已打开文件的上下文（当前偏移量、打开模式等）

## 4.4 网络协议栈

- 完整的 TCP/IP 协议栈实现
- Socket 层作为用户空间接口
- Netfilter 框架支持防火墙、NAT、连接跟踪
- 多队列网卡支持、RPS/RFS/XPS 等分发机制
- 零拷贝（sendfile()、splice()）和 TCP Offload

## 4.5 设备驱动模型

- 使用 **struct device_driver** 和 **struct device** 分离驱动与设备
- 支持热插拔：udev 在用户态管理设备节点
- 设备树（Device Tree）在嵌入式平台描述硬件
- 字符设备、块设备、网络设备的统一管理

# 5. Linux 内核源代码结构

Linux 内核源码树（Documentation/process/coding-style.rst 有详细说明）：

```
arch/           # 架构相关代码（x86, arm, riscv 等）
block/          # 块层 I/O 调度
crypto/         # 加密 API
drivers/        # 设备驱动（最大目录）
fs/             # 文件系统实现（ext4, xfs, btrfs 等）
include/        # 头文件
init/           # 初始化代码
ipc/            # 进程间通信（System V IPC）
kernel/         # 核心子系统（调度、定时器、信号等）
lib/            # 内核工具函数
mm/             # 内存管理
net/            # 网络协议栈
scripts/        # 编译和配置脚本
security/       # 安全模块（LSM、SELinux 等）
sound/          # 音频子系统
virt/           # 虚拟化（KVM）
```

# 6. GNU 工具链与系统库

Linux 系统不仅是内核，还依赖完整的用户空间工具集：

| 组件 | 说明 |
|------|------|
| glibc | GNU C 库，提供系统调用封装、标准 C 库函数 |
| coreutils | 核心工具集（ls, cp, mv, rm, cat 等） |
| util-linux | 系统工具（mount, fdisk, kill, login 等） |
| bash | Bourne Again SHell，默认 Shell |
| systemd | init 系统与服务管理器（正被越来越多发行版采纳） |

# 7. /sys 与 /proc 虚拟文件系统

这两个文件系统不占用磁盘空间，由内核动态生成，用于暴露内核数据结构和状态：

## 7.1 /proc
- `/proc/cpuinfo`：CPU 信息
- `/proc/meminfo`：内存使用情况
- `/proc/[pid]/`：每个进程的详细信息（文件描述符、内存映射、环境变量等）
- `/proc/sys/`：可通过 sysctl 调整的内核参数

## 7.2 /sys
- `/sys/class/`：按设备类别分类（net, block, input 等）
- `/sys/devices/`：系统设备的真实拓扑结构
- `/sys/kernel/`：内核参数与信息
- `/sys/fs/`：文件系统相关参数（如 cgroup 挂载点）
