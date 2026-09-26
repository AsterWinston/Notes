
# 1. 什么是 FHS

FHS（Filesystem Hierarchy Standard）是 Linux 文件系统目录结构的标准规范，定义了各个目录的用途和存放内容类型。遵守 FHS 使得不同发行版之间保持一致性和可预测性。

当前版本为 FHS 3.0（2015 年发布）。

# 2. 根目录 `/` 结构总览

```
/                   # 根文件系统
├── bin/            # 基本用户命令二进制文件
├── boot/           # 引导加载程序相关文件
├── dev/            # 设备文件
├── etc/            # 系统配置文件
├── home/           # 用户家目录
├── lib/            # 基础共享库和内核模块
├── lib64/          # 64位共享库
├── media/          # 可移动介质挂载点
├── mnt/            # 临时挂载点
├── opt/            # 可选应用软件包
├── proc/           # 进程与内核虚拟文件系统
├── root/           # root 用户家目录
├── run/            # 运行时变量数据（tmpfs）
├── sbin/           # 系统管理命令（二进制）
├── srv/            # 服务数据
├── sys/            # 内核与设备虚拟文件系统
├── tmp/            # 临时文件
├── usr/            # 二级层次（用户工具与应用）
└── var/            # 可变数据
```

# 3. 核心目录详解

## 3.1 `/bin` — 基本用户命令

包含所有用户（包括单用户模式）都需要的命令：

```bash
ls cp mv rm cat echo chmod chown mount
grep sed awk ps kill login bash sh
```

**注意**：在较新的发行版中，`/bin` 通常是 `/usr/bin` 的符号链接。

## 3.2 `/boot` — 引导文件

存放引导加载程序和内核映像：

```bash
/boot/vmlinuz-<version>          # 压缩的内核映像
/boot/initrd.img-<version>       # initramfs 映像
/boot/System.map-<version>       # 内核符号表
/boot/config-<version>           # 内核编译配置
/boot/grub/                      # GRUB 配置文件
```

## 3.3 `/dev` — 设备文件

Linux 中一切皆文件，硬件设备通过设备文件访问：

```bash
/dev/sda        # 第一块 SCSI/SATA 磁盘
/dev/sda1       # 第一块磁盘的第一个分区
/dev/tty1       # 第一个虚拟终端
/dev/ttyS0      # 第一个串口
/dev/null       # 空设备（丢弃写入的数据）
/dev/zero       # 无限输出 \0
/dev/random     # 随机数发生器（阻塞）
/dev/urandom    # 随机数发生器（非阻塞）
/dev/loop0      # loop 设备
```

**设备文件类型**：
- 字符设备（c）：如终端、串口，数据传输以字符流方式
- 块设备（b）：如磁盘，以块为单位读写，支持随机访问

## 3.4 `/etc` — 配置文件

系统级别的配置文件，基本上是纯文本：

```bash
/etc/passwd          # 用户账户信息
/etc/shadow          # 用户密码哈希
/etc/group           # 用户组信息
/etc/sudoers         # sudo 权限配置
/etc/fstab           # 文件系统表（开机自动挂载）
/etc/hostname        # 主机名
/etc/hosts           # 主机名与 IP 映射
/etc/resolv.conf     # DNS 配置
/etc/apt/            # apt 配置（Debian/Ubuntu）
/etc/ssh/            # SSH 配置
/etc/nginx/          # Nginx 配置
/etc/cron.d/         # cron 任务
```

**重要原则**：`/etc` 下的配置文件不应该包含可执行文件，应该纯文本可编辑。

## 3.5 `/home` — 用户家目录

每个普通用户的家目录，通常包含用户的个人文件、配置、数据：

```bash
/home/
├── alice/
│   ├── .bashrc        # bash 配置
│   ├── .ssh/          # SSH 密钥和配置
│   ├── Documents/
│   └── Downloads/
└── bob/
    └── ...
```

**注意**：root 用户的家目录在 `/root` 而非 `/home/root`。

## 3.6 `/proc` — 进程虚拟文件系统

由内核动态生成的虚拟文件系统，不占用磁盘空间：

```bash
/proc/cpuinfo          # CPU 详细信息
/proc/meminfo          # 内存使用详情
/proc/version          # Linux 内核版本
/proc/uptime           # 系统运行时间
/proc/loadavg          # 系统负载平均值
/proc/[pid]/           # 每个进程的详细目录
    /proc/[pid]/fd/        # 打开的文件描述符
    /proc/[pid]/maps       # 内存映射
    /proc/[pid]/environ    # 环境变量
    /proc/[pid]/cmdline    # 命令行参数
    /proc/[pid]/status     # 进程状态
/proc/sys/             # 可通过 sysctl 调整的内核参数
```

## 3.7 `/usr` — 用户系统资源

**历史**：最初用于存放用户 home 目录，现在是最主要的软件安装目录。

`/usr` 的层次结构与根目录类似，是系统的二级层次：

```bash
/usr/
├── bin/            # 用户可用的绝大多数命令
├── lib/            # 库文件
├── local/          # 本地安装的软件（编译安装默认前缀）
│   ├── bin/
│   ├── lib/
│   └── share/
├── sbin/           # 系统管理命令
├── share/          # 架构无关数据（文档、图标、man pages）
├── include/        # C/C++ 头文件
└── src/            # 源码（可选）
```

**现代趋势**：许多发行版将 `/bin`、`/sbin`、`/lib` 符号链接到 `/usr` 下对应的目录（称为 UsrMerge）。

## 3.8 `/var` — 可变数据

存放运行时会变化的数据：

```bash
/var/
├── log/            # 日志文件
│   ├── syslog
│   ├── auth.log
│   └── kern.log
├── cache/          # 应用程序缓存数据
│   ├── apt/
│   └── pip/
├── lib/            # 数据库文件、状态信息
│   ├── mysql/
│   └── docker/
├── spool/          # 排队数据（邮件、打印任务）
│   ├── mail/
│   └── cron/
├── tmp/            # 重启后保留的临时文件
└── www/            # Web 服务器文档根目录（常见实践）
```

## 3.9 `/tmp` — 临时文件

- 所有用户均可写入
- 重启后 `/tmp` 中的文件通常被清除
- 现代系统使用 tmpfs 挂载（内存中的文件系统）

## 3.10 `/run` — 运行时变量数据

- tmpfs 挂载，系统启动时创建
- 存放 PID 文件、套接字文件等运行时信息
- 替代了传统的 `/var/run`（后者现在是符号链接）

## 3.11 `/opt` — 可选应用

用于安装第三方商业软件或自定义软件包：

```bash
/opt/google/chrome/
/opt/VBoxGuestAdditions/
```

# 4. 目录功能速查表

| 目录 | 静态/动态 | 关键特性 |
|------|-----------|----------|
| `/bin` | 静态 | 所有用户必需的命令 |
| `/boot` | 静态 | 引导加载程序文件 |
| `/dev` | 动态（重启重建） | 设备文件 |
| `/etc` | 静态（但可配置） | 系统配置文件 |
| `/home` | 动态 | 用户个人数据 |
| `/proc` | 动态（运行时） | 内核与进程信息 |
| `/root` | 动态 | root 用户家目录 |
| `/run` | 动态（tmpfs） | 运行时状态数据 |
| `/sbin` | 静态 | 系统管理命令 |
| `/sys` | 动态（运行时） | 内核对象和设备信息 |
| `/tmp` | 动态（可清理） | 临时文件 |
| `/usr` | 静态 | 系统主要软件和数据 |
| `/var` | 动态 | 运行时可变数据 |

# 5. 文件路径表示

Linux 使用正斜杠 `/` 作为路径分隔符（Windows 使用反斜杠 `\`）。

```bash
# 绝对路径：从根目录开始
/etc/apt/sources.list

# 相对路径：相对于当前工作目录
Documents/notes.txt
../config/settings.conf

# 特殊目录
.            # 当前目录
..           # 父目录
~            # 当前用户的家目录
~alice       # 用户 alice 的家目录
-            # 上一个工作目录（cd -）
```
