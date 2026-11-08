# 1. 什么是 Linux

Linux 是一种自由开源的操作系统内核，由 Linus Torvalds 于 1991 年创建。广义上的 Linux 指基于 Linux 内核的完整操作系统（GNU/Linux），包含了内核、系统工具库（glibc）、Shell、桌面环境、应用软件等。**内核只负责最核心的资源抽象——进程调度、内存管理、文件系统、网络栈、设备驱动；外围的命令行工具与库主要来自 GNU 项目**，两者结合才是你安装到服务器上的那个系统，这也是 "GNU/Linux" 写法的由来。

核心特征：

- **多用户多任务**：支持多个用户同时登录、多个进程并发执行，权限边界清晰
- **一切皆文件**：将硬件设备、进程、网络等抽象为文件接口，统一用 open/read/write 操作
- **模块化设计**：内核可动态加载/卸载模块（.ko），不必重编内核就能加驱动和文件系统
- **POSIX 兼容**：遵循可移植操作系统接口标准，为同一份代码跨 Unix 家族移植提供契约

对后端工程师来说，**"一切皆文件"是理解 Linux 的第一把钥匙**：/proc 下的虚拟文件暴露进程状态（详见 10_process_management.md），/sys 暴露设备与内核参数，socket 也是一类文件描述符。排查线上 Java 应用的问题——CPU 高、句柄泄漏、被 OOM Killer 杀掉——最终都要落到读这些文件上。

# 2. 发展历史

| 时间 | 事件 |
|------|------|
| 1969 | Ken Thompson 和 Dennis Ritchie 在贝尔实验室开发 Unix |
| 1983 | Richard Stallman 发起 GNU 项目，目标构建自由 Unix 系统 |
| 1987 | Andrew Tanenbaum 发布 MINIX 用于教学 |
| 1991 | Linus Torvalds 发布 Linux 0.01 |
| 1992 | Linux 采用 GPLv2 许可证 |
| 1993 | Slackware 和 Debian 项目启动 |
| 1994 | Linux 1.0 发布 |
| 2003 | Red Hat Enterprise Linux (RHEL) 发布 |
| 2004 | Ubuntu 首个版本发布 |
| 2005 | Git 由 Linus 开发用于管理 Linux 内核源码 |
| 2007 | Android 基于 Linux 内核发布 |
| 2015 | systemd 成为主流发行版默认 init 系统 |
| 2024 | Linux 内核 6.x 系列，覆盖从嵌入式到超算的全场景 |

历史里有三条主线值得记住：**GPLv2 决定了 Linux 的商业形态**——衍生内核必须开源，厂商的竞争转向发行、支持与云集成；**宏内核与微内核之争以宏内核胜出告终**，Linux 采用单体内核 + 模块机制，性能优先（详见 2_architecture.md）；**2014 年 CentOS 成为 RHEL 兼容版第一选择、2020 年 CentOS 转向 Stream 滚动预发布版**，直接改变了国内服务器发行版格局（第 3 节展开）。

# 3. 主要发行版分类

发行版 = Linux 内核 + 软件包管理 + 默认工具链 + 支持策略。选发行版本质是选**更新节奏与支持模式**。

## 3.1 Debian 系

- **Debian**：稳定性优先，包管理使用 apt/dpkg，软件包格式 .deb，自由软件原则最严格
- **Ubuntu**：基于 Debian，以易用性著称，每两年发布 LTS 版本（支持 5 年），服务器装机量大
- **Linux Mint**：基于 Ubuntu，适合桌面用户
- **Kali Linux**：基于 Debian，面向安全测试

## 3.2 RHEL 系

- **Red Hat Enterprise Linux**：商业发行版，稳定且提供企业级支持，订阅制收费
- **CentOS**：RHEL 的免费重建版，**2020 年底红帽宣布 CentOS 8 提前停服并转向 CentOS Stream**——Stream 是 RHEL 的上游预发布（滚动更新），不再是 1:1 下游重建，生产环境依赖它等于把未冻结的版本当稳定版用
- **Fedora**：RHEL 的上游试验场，采用较新软件包，约 6 个月一版
- **Rocky Linux / AlmaLinux**：CentOS 停更后的替代方案，由原 CentOS 创始人与 CloudLinux 分别发起，承诺与 RHEL 逐位兼容（bug-for-bug）

> **行业实践**：CentOS 7 于 2024 年 6 月正式 EOL 后，国内存量迁移的主流去向是 **Rocky/Alma（追求 RHEL 等价）、Ubuntu LTS（社区资料多）、以及龙蜥 OpenAnolis / openEuler 等国产发行版（信创场景）**。云厂商镜像默认项也在同步切换，新建服务不再建议选 CentOS。

## 3.3 其他重要发行版

- **Arch Linux**：滚动更新，高度可定制，面向高级用户
- **openSUSE**：SUSE 的社区版本，支持 YaST 管理工具
- **Alpine Linux**：极小体积（5MB 级），基于 musl libc + BusyBox，广泛用于容器镜像

> **注意**：发行版之间命令大体一致，但**包管理与服务单元路径有差异**——Debian 系用 apt、服务配置常在 /lib/systemd 与 /etc/default；RHEL 系用 dnf/yum、网络配置历史上依赖 /etc/sysconfig（NetworkManager 时代改用 nmcli）。跨发行版抄教程时，先确认对方是哪个系，否则容易把 Ubuntu 的指令敲进 Rocky 里。

## 3.4 选择建议

| 场景 | 推荐发行版 | 理由 |
|------|-----------|------|
| 生产服务器 | Ubuntu LTS / Rocky / AlmaLinux | 长期支持 + 生态兼容 |
| 桌面开发 | Ubuntu / Fedora / Linux Mint | 驱动与新版软件友好 |
| 嵌入式 | Buildroot / Yocto / Alpine | 可裁剪到最小系统 |
| 容器镜像 | Alpine / Distroless / Ubuntu | 体积小、攻击面小 |
| 学习内核原理 | Arch / Gentoo / LFS | 自己动手拼装系统 |

> **设计争议**：Alpine 在容器里流行（小、快），但 musl libc 与 glibc 的行为差异会坑到 Java/Go 应用——DNS 解析（musl 不读 /etc/resolv.conf 的部分选项）、线程栈默认值、locale 缺失导致中文乱码。对以 JVM 为主的团队，**"镜像最小化"与"运行时兼容性"之间优先选后者**，官方 eclipse-temurin 镜像的 glibc 版本比 Alpine 版更省心。

# 4. Linux 设计哲学

1. **每个程序只做一件事，并做好它**：单一职责原则，grep 只过滤、wc 只统计
2. **程序之间通过文本接口协作**：管道和重定向实现组合，stdout 是通用总线
3. **一切皆文件**：统一的操作接口简化了资源管理
4. **存储配置信息于文本文件**：易于管理和版本控制，/etc 就是系统的"源码"
5. **避免不必要的用户交互**：自动化脚本和批处理，一切操作可用命令复现
6. **模块化原则**：内核模块、工具链、库都独立可替换

这套哲学落地成工作习惯就是：**能用命令行的不要碰图形界面，能组合小工具的不要写大程序**。典型例子是排查一行日志里的错误分布：`grep ERROR app.log | awk '{print $4}' | sort | uniq -c | sort -rn | head`，五个各司其职的小工具经管道串起来，五秒钟得到按出现次数排序的 Top 错误。与之相对，Windows 生态偏重 GUI 与单体应用，这也是服务器领域 Linux 赢下市场的深层原因——**可脚本化 = 可自动化 = 可规模化**。

# 5. Linux 的优势与应用场景

**核心优势：**

- 开源免费，无许可证限制
- 高度稳定可靠，服务器连续运行数年不重启
- 安全性好，权限模型严谨（DAC + MAC + 网络栈隔离，详见 19/28 篇）
- 资源占用低，可在老旧硬件上运行
- 生态丰富，几乎所有编程语言和框架都优先支持 Linux

**应用场景：**

| 场景 | 占比/说明 |
|------|----------|
| 服务器 | 全球约 96% 的 Web 服务器运行 Linux |
| 云计算 | AWS/Azure/GCP 等云平台基础设施几乎全部基于 Linux |
| 嵌入式/IoT | Android、路由器、智能设备广泛使用 Linux 内核 |
| 超算 | TOP500 超算中约 100% 运行 Linux |
| 容器 | Docker/Kubernetes 底层依赖 Linux 内核特性 |

**后端工程师为什么必须会 Linux**：你的 Java/Go 服务 100% 跑在 Linux 上（或它的容器里），而绝大多数线上故障的现场证据只存在于操作系统层——`top` 里那个 100% CPU 的线程、`dmesg` 里的 OOM Killer 记录、`ss -s` 里的 TIME_WAIT 堆积、`df -i` 爆满的 inode。**只会写代码不会看系统，故障排查就只能重启碰运气**；反过来，JVM 调优（线程栈大小、堆外内存）、网络调优（somaxconn、tcp_tw_reuse）、K8s 的 namespace/cgroup 资源限制，全部是 Linux 概念的直接应用。

# 6. 本模块导航

本模块按"基础操作 → 系统管理 → 内核进阶"三层组织，各篇定位如下：

| 篇 | 主题 | 内容速览 |
|----|------|----------|
| 2 | 内核与系统架构 | 宏内核、模块机制、用户态/内核态切换 |
| 3 | 启动流程 | BIOS/UEFI → bootloader → systemd 各阶段 |
| 4 | 目录结构（FHS） | /etc、/var、/proc 等目录的职责边界 |
| 5-7 | 文件与编辑 | 文件操作、内容检索（grep/sed/awk）、Vim |
| 8-9 | 用户与权限 | UID/GID、sudo、rwx 与 ACL、特殊权限 |
| 10-11 | 进程与内存 | ps/top/信号、/proc、虚拟内存与 OOM |
| 12-14 | 磁盘与文件系统 | 分区挂载、LVM/RAID、ext4/xfs 特性 |
| 15 | 包管理 | apt/dpkg、yum/dnf、rpm |
| 16-18 | 网络与远程 | 网络配置、DNS 解析、SSH 免密与运维 |
| 19/28 | 防火墙与安全 | iptables/nftables、加固基线 |
| 20 | 服务管理 | systemd unit、target、journal |
| 21-22 | 定时与日志 | cron、logrotate 与日志体系 |
| 23/25 | 性能与排障 | CPU/内存/IO 监控工具链、故障案例 |
| 24/26-27 | 内核与周边 | 内核基础、编译工具链、时间同步 |

**学习建议**：5-9 是命令与权限的基本功，所有后端同学必学；10-11 与 23（进程、内存、性能监控）直接对应线上 Java 故障排查，优先级最高；20（systemd）是部署与运维的地基；24-26 偏内核与编译，除非做基础架构，按需选读。

# 7. 学习路线图

学习 Linux 建议由浅入深，分为以下几个阶段：

```text
第一阶段（基础入门）
├── 安装与基本操作（VM/双系统/WSL）
├── 文件系统结构与常用命令
├── Vi/Vim 基本使用
└── 用户与权限管理

第二阶段（系统管理）
├── 进程与服务管理（systemd）
├── 磁盘与文件系统管理
├── 网络配置与排错
├── 包管理（apt/dpkg）
└── Shell 脚本基础

第三阶段（深入进阶）
├── Linux 内核原理（进程调度、内存管理、VFS）
├── 性能监控与调优
├── 安全加固与审计
├── 网络协议栈与防火墙
└── 容器技术（cgroups/namespace）

第四阶段（专家方向）
├── 内核模块开发
├── 性能分析与火焰图
├── eBPF 与可观测性
├── 内核调试（kgdb/kdump）
└── 系统架构设计
```

环境选择上，**学运维与排障优先云服务器或虚拟机，学命令用 WSL/Docker 也足够**；真正的分水岭在第三阶段——能解释 `cat /proc/loadavg`、`free` 的 available 与 buff/cache 关系、CFS 调度的 vruntime 时，Linux 才从"会敲命令"变成"理解系统"。本模块后续各篇即按此路线展开。

# 8. 小结

- **Linux = 内核 + GNU 工具链的完整系统**，"一切皆文件、多用户多任务、GPLv2、POSIX"四大特征决定了它的运维形态。
- **发行版选择即支持策略选择**：CentOS 停服后，生产首选 Ubuntu LTS / Rocky / AlmaLinux；容器里的 Alpine 要警惕 musl 与 JVM 的兼容坑。
- **设计哲学（小工具 + 管道 + 文本配置 + 可脚本化）是 Linux 赢下服务器市场的根因**，也是后端工程师日常工作流的底色。
- **后端必须会 Linux**：线上故障证据在 top/dmesg/ss//proc 里，JVM 与 K8s 调优的概念全部源自内核机制。
- 本模块按"基本功（5-9）→ 排查与部署（10-11、20、23）→ 内核进阶（24-26）"三层导航，遵循四阶段路线图推进即可。
