Electron 的魅力与复杂度,都在它的**多进程模型**里——主进程管系统、渲染进程管界面,各守其职、靠 IPC 协作。本篇深入讲清主进程的能力与职责、BrowserWindow 窗口、应用生命周期的关键事件,以及"什么该放哪个进程"的判据。

# 1. 主进程的职责

## 1.1 主进程能做什么

```javascript
// 主进程独占的能力(渲染进程没有):
const { app, BrowserWindow, dialog, Menu, Tray, ipcMain } = require('electron')
// 创建窗口、显示系统对话框、原生菜单、系统托盘、监听 IPC……
```

★ **关键认知**：**主进程(Main)是应用的"管家"**——它独占**窗口管理、系统对话框、原生菜单、托盘、文件系统**等系统级能力。⚠ 记住一个判断:"**这个能力是不是浏览器网页也有的?**" 网页有 → 渲染进程能做;网页没有(菜单、托盘、窗口)→ 必须是主进程。整条 Electron 开发主线,很大程度上就是"把主进程能力通过 IPC 暴露给渲染进程"。

## 1.2 主进程为什么"一个就够"

★ **关键认知**：整个应用**只有一个主进程**(`app` 是单例),它管理所有窗口与应用生命周期。⚠ 但主进程能**开多个渲染进程**(每个 BrowserWindow 一个),实现"一管家 + 多网页"的结构;主进程代码要写得健壮——它崩了,整个应用就崩。

# 2. BrowserWindow 窗口

## 2.1 创建与选项

```javascript
const win = new BrowserWindow({
  width: 800,
  height: 600,
  title: '我的应用',
  webPreferences: {
    preload: join(__dirname, '../preload/index.js'),
    contextIsolation: true,
    nodeIntegration: false
  }
})
win.loadFile('index.html')
```

★ **关键认知**：`BrowserWindow` 就是**一个窗口**——创建时配尺寸、标题、webPreferences(预加载等),然后 `loadFile`/`loadURL` 加载内容。⚠ `webPreferences` 里那两个安全选项(第 1、16 篇)几乎每次都要写;窗口的更多形态(无边框、多窗口、托盘)见第 6 篇。

## 2.2 窗口生命周期事件

```javascript
win.on('closed', () => { win.close() })     // 窗口关闭
win.on('focus', () => {})                  // 获得焦点
win.on('blur', () => {})                   // 失去焦点
win.once('ready-to-show', () => win.show()) // 内容就绪再显示(避免白屏)
```

★ **关键认知**：窗口有一系列**生命周期事件**(closed/focus/blur/ready-to-show…)。⚠ 常用技巧:**`ready-to-show` 后再 `show()`**,避免"窗口先出现白屏再加载内容"的闪烁;配合 `show: false` 创建时隐藏。

# 3. 应用生命周期

## 3.1 app 的关键事件

```javascript
const { app } = require('electron')

app.whenReady().then(() => { /* 应用就绪,可创建窗口 */ })   // 就绪(核心)
app.on('window-all-closed', () => {                          // 所有窗口关闭
  if (process.platform !== 'darwin') app.quit()              // macOS 之外直接退出
})
app.on('activate', () => {                                   // macOS 点 Dock 图标
  if (BrowserWindow.getAllWindows().length === 0) createWindow()
})
app.on('before-quit', () => { /* 退出前清理 */ })
```

★ **关键认知**：`app` 的生命周期事件是主进程的骨架——**`whenReady`(就绪后建窗口)、`window-all-closed`(窗口全关)、`activate`(macOS 重激活)**。⚠ 有个**平台差异**要记:macoS 上"关掉所有窗口"应用**不退出**(点 Dock 重新激活),Windows/Linux 则默认退出——所以 `window-all-closed` 里要判 `platform === 'darwin'`。

## 3.2 单实例与生命周期

```javascript
const gotLock = app.requestSingleInstanceLock()   // 单实例锁
if (!gotLock) {
  app.quit()          // 已有实例,直接退出
} else {
  app.on('second-instance', () => { /* 用户又启动了一次,聚焦已有窗口 */ })
}
```

★ **关键认知**：`requestSingleInstanceLock` 确保应用**只开一个实例**(双击图标不重复启动)。⚠ 桌面应用通常都该单实例(尤其有托盘/后台的);没这个锁,用户狂点图标会开出一堆进程。

# 4. 什么放哪个进程

## 4.1 职责划分

| 能力 | 主进程 | 渲染进程 |
|------|--------|---------|
| UI/界面/交互 | — | ✓ |
| DOM/CSS/前端框架 | — | ✓ |
| 窗口/菜单/托盘 | ✓ | — |
| 文件系统/系统 API | ✓ | — |
| 网络请求 | ✓(可) | ✓(受 CORS 限制) |
| IPC 监听 | ✓(ipcMain) | ✓(ipcRenderer) |

★ **关键认知**：划分的黄金标准是"**浏览器网页能做的 → 渲染进程;需要系统权限的 → 主进程**"。⚠ 常见误区:**把一切逻辑堆进主进程**(它成了大杂烩)或**渲染进程硬要碰 Node**(安全风险);正确做法是"**主进程做系统能力,渲染进程做界面,预加载/IPC 做桥**"。

## 4.2 一个典型交互的链路

```
用户点击"保存文件"按钮(渲染进程)
  → ipcRenderer 发消息(第 5 篇)
  → 主进程 ipcMain 收到 → 调 dialog.showSaveDialog(系统对话框)
  → 用户选路径 → 主进程 fs.writeFileSync 写文件
  → 主进程把结果回传渲染进程 → 界面提示"已保存"
```

★ **关键认知**：这条"**渲染进程发起 → 主进程执行系统操作 → 结果回传**"的链路,是 Electron 里 90% 功能交互的通用范式。⚠ 早理解它,后面 IPC、文件、对话框各篇都是它的具体化。

# 5. 本篇小结与下一步

★ **关键认知**：主进程管系统(窗口/菜单/托盘),渲染进程管界面,判据是"网页能不能做"。app 生命周期(whenReady/window-all-closed/activate)与 macOS 平台差异是必考;典型响应流程"渲染发起 → 主进程执行 → 回传"。

下一步第 4 篇:预加载脚本——preload、contextBridge、contextIsolation,安全暴露 API 的正确姿势。