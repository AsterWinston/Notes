# Electron 学习笔记

Electron 是用 Web 技术(HTML/CSS/JS)构建**桌面应用**的框架——它把 Chromium(渲染界面)与 Node.js(访问系统)缝合成一个可打包分发的桌面程序，VS Code、Slack、Figma 都建立在它之上。本模块共 21 篇，从进程模型与 IPC 讲到窗口/原生能力、工程化打包更新，再到安全与多进程进阶。主线是**主进程 / 渲染进程/ 预加载脚本**三件套。受众定位：已熟悉前端(建议先读本仓库 Vue 或 React 系列)与 Node 基础，想开发桌面应用的开发者。

# 1. 目录

## 1.1 阶段一：基础（1-5 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_introduction.md](1_introduction.md) | 简介与架构 | Electron 定位、Chromium+Node、主渲染进程模型、典型应用 |
| [2_setup.md](2_setup.md) | 环境搭建与快速上手 | electron-vite 脚手架、项目结构、启动流程、热更新 |
| [3_process_model.md](3_process_model.md) | 主进程与渲染进程 | BrowserWindow、进程职责、生命周期、main 入口 |
| [4_preload.md](4_preload.md) | 预加载脚本 | preload、contextBridge、contextIsolation、暴露安全 API |
| [5_ipc_basics.md](5_ipc_basics.md) | IPC 通信基础 | ipcMain/ipcRenderer、同步/异步、invoke/handle、事件 |

## 1.2 阶段二：窗口与原生能力（6-10 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [6_window_management.md](6_window_management.md) | 窗口管理 | BrowserWindow 选项、多窗口、无边框、托盘 Tray |
| [7_menu_shortcut.md](7_menu_shortcut.md) | 菜单与快捷键 | Menu、accelerator、contextMenu、globalShortcut |
| [8_dialog_notification.md](8_dialog_notification.md) | 对话框与通知 | dialog、Notification、快捷键 |
| [9_system_integration.md](9_system_integration.md) | 系统集成 | clipboard、shell、nativeImage、powerMonitor、autoLaunch |
| [10_file_and_storage.md](10_file_and_storage.md) | 文件与存储 | fs 文件操作、electron-store、SQLite、userData 目录 |

## 1.3 阶段三：工程化（11-15 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [11_framework_integration.md](11_framework_integration.md) | 前端框架集成 | Vue/React + Electron、通信封装、预加载最佳实践 |
| [12_state_flow.md](12_state_flow.md) | 状态与数据流 | 跨进程状态协同、主进程持有数据、状态同步 |
| [13_build_package.md](13_build_package.md) | 打包与构建 | electron-builder/forge、asar、代码签名、win/mac/linux |
| [14_auto_update.md](14_auto_update.md) | 自动更新 | electron-updater、更新服务器、灰度发布、增量更新 |
| [15_performance.md](15_performance.md) | 性能优化 | 渲染内存、启动速度、多窗口开销、GPU |

## 1.4 阶段四：安全与进阶（16-21 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [16_security.md](16_security.md) | 安全最佳实践 | contextIsolation、沙箱、CSP、权限最小化、危险 API |
| [17_multiprocess_worker.md](17_multiprocess_worker.md) | 多进程与 worker | utilityProcess、worker_threads、child_process、任务分发 |
| [18_native_modules.md](18_native_modules.md) | 原生模块与 Node 集成 | native addons、ffi、Node 能力边界 |
| [19_test_debug.md](19_test_debug.md) | 测试与调试 | 单元测试、e2e(Playwright)、DevTools、日志 |
| [20_cross_platform.md](20_cross_platform.md) | 跨平台疑难 | 平台差异、路径/通知/快捷键坑点、排查 |
| [21_interview_and_summary.md](21_interview_and_summary.md) | 面试要点与总结 | 高频面试题、知识地图、进阶方向 |

# 2. 推荐阅读路径

```
前端开发者          Node 有基础        要上生产           冲刺深入
     │                  │                  │                  │
     ▼                  ▼                  ▼                  ▼
  1 → 2 → 3          3 → 4 → 5        6 → 10 → 13        16 → 17 → 18
  4 → 5 → 6          11 → 12          14 → 15            19 → 21
```

- **速成路线（约 3 天）**：1 → 2 → 3 → 4 → 5 → 6 → 13，能做出一个能打包的桌面小程序。
- **完整路线**：按序号 1~21 顺读，每篇代码亲手敲一遍。
- **生产上线路**：3 → 5 → 13 → 14 → 16。
- **前置建议**：先读 `Frontend/Vue` 或 `Frontend/React` 系列，再读本系列第 11 篇会顺畅得多。

# 3. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 该节最核心的机制/结论，优先记住 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均带逐行中文注释，可放入 electron-vite 项目直接运行 |
| 命令清单 | 可照抄执行；npm 源均使用 npmmirror 国内镜像 |