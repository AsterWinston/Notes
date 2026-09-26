
# 1. 什么是 Linux

Linux 是一种自由开源的操作系统内核，由 Linus Torvalds 于 1991 年创建。广义上的 Linux 指基于 Linux 内核的完整操作系统（GNU/Linux），包含了内核、系统工具库、Shell、桌面环境、应用软件等。

核心特征：
- **多用户多任务**：支持多个用户同时登录、多个进程并发执行
- **一切皆文件**：将硬件设备、进程、网络等抽象为文件接口
- **模块化设计**：内核可动态加载/卸载模块
- **POSIX 兼容**：遵循可移植操作系统接口标准

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

# 3. 主要发行版分类

## 3.1 Debian 系
- **Debian**：稳定性优先，包管理使用 apt/dpkg，软件包格式 .deb
- **Ubuntu**：基于 Debian，以易用性著称，每两年发布 LTS 版本
- **Linux Mint**：基于 Ubuntu，适合桌面用户
- **Kali Linux**：基于 Debian，面向安全测试

## 3.2 RHEL 系
- **Red Hat Enterprise Linux**：商业发行版，稳定且提供企业级支持
- **CentOS**：RHEL 的免费重建版（已转向 CentOS Stream）
- **Fedora**：RHEL 的上游试验场，采用较新软件包
- **Rocky Linux / AlmaLinux**：CentOS 停更后的替代方案

## 3.3 其他重要发行版
- **Arch Linux**：滚动更新，高度可定制，面向高级用户
- **openSUSE**：SUSE 的社区版本，支持 YaST 管理工具
- **Alpine Linux**：极小体积，基于 musl libc + BusyBox，广泛用于容器镜像

## 3.4 选择建议

| 场景 | 推荐发行版 |
|------|-----------|
| 服务器 | Ubuntu LTS / Debian / Rocky Linux |
| 桌面开发 | Ubuntu / Fedora / Linux Mint |
| 嵌入式 | Buildroot / Yocto / Alpine |
| 容器镜像 | Alpine / Ubuntu / Distroless |
| 学习内核 | Arch Linux / Gentoo |

# 4. Linux 设计哲学

1. **每个程序只做一件事，并做好它**：单一职责原则
2. **程序之间通过文本接口协作**：管道和重定向实现组合
3. **一切皆文件**：统一的操作接口简化了资源管理
4. **存储配置信息于文本文件**：易于管理和版本控制
5. **避免不必要的用户交互**：自动化脚本和批处理
6. **模块化原则**：内核模块、工具链、库都独立可替换

# 5. Linux 的优势与应用场景

**核心优势：**
- 开源免费，无许可证限制
- 高度稳定可靠，服务器连续运行数年不重启
- 安全性好，权限模型严谨
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

# 6. 学习路线图

学习 Linux 建议由浅入深，分为以下几个阶段：

```
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
