Electron 是"用 Web 技术写桌面应用"的框架——把 **Chromium(渲染界面)** 与 **Node.js(访问系统)** 缝在一起,让你用 HTML/CSS/JS 做出能在 Windows/macOS/Linux 上跑的桌面程序。本篇讲清 Electron 的定位、主/渲染进程模型、以及它为什么"又爱又重",为整个系列定下心智坐标。

# 1. Electron 是什么

## 1.1 一句话定位

Electron 是一个**用 Web 技术构建桌面应用**的框架——它打包了一个精简的 **Chromium** 负责界面,内嵌 **Node.js** 负责访问文件、系统,两相结合,桌面程序 = 网页界面 + 系统能力。

```
  你的 Electron 应用
    ├─ Chromium ── 渲染界面(HTML/CSS/JS,和写网页一样)
    └─ Node.js ─── 访问文件系统、网络、系统 API(web 做不到的事)
```

★ **关键认知**：Electron 的核心价值一句话——**用你已经会的 Web 技术(HTML/CSS/JS)做桌面软件**,同时借 Node.js **突破浏览器沙箱**(读写文件、原生菜单、系统托盘)。⚠ 代价是**体积大**(每个应用都带一个 Chromium,几十到上百 MB),这是它最大的特点,也是最常被对比的点。

## 1.2 谁在用

★ **关键认知**：VS Code、Slack、Figma、Discord、Notion 桌面版…… 大量知名桌面应用都是 Electron。⚠ 它们的共同点——**跨平台 + Web 技术栈 + 快速迭代**;如果这三条是你的诉求,Electron 就是候选;如果极致轻量/原生性能是首要,才该考虑 Tauri(native webview)或原生开发。

# 2. 主/渲染进程模型

## 2.1 两个进程的世界

```
Electron 应用 = 两种进程:
  主进程(Main Process)   —— 一个,掌管窗口、系统 API、Node 全能力
  渲染进程(Renderer)     —— 每个窗口一个,负责界面(像浏览器标签页,受限)
```

★ **关键认知**：Electron 的架构核心是**多进程**——**一个主进程**管理应用生命周期与窗口,**每个窗口一个渲染进程**负责界面。⚠ 理解"哪个进程能做什么"是 Electron 的第一课:**主进程是"管家"(系统权限全),渲染进程是"网页"(能力受限如浏览器)**。两者靠 IPC(第 5 篇)通信。

## 2.2 为什么分主渲染

★ **关键认知**：分进程的理由与浏览器分标签页相同——**隔离与安全**:一个窗口崩了、被攻破了,不影响其他窗口与主进程;渲染进程被"YouTube 那样的页面"塞满,也不会拖垮系统 API 所在的进程。⚠ 由此诞生两条职责线:**UI 在渲染进程,系统能力在主进程**,它们之间用 IPC 桥接——这是贯穿本系列的骨架。

# 3. 与其他方案的对比

## 3.1 Electron vs Tauri vs 原生

| 维度 | Electron | Tauri | 原生(Win/mac) |
|------|----------|-------|---------------|
| 界面技术 | Chromium | 系统 WebView | 原生控件 |
| 体积 | 大(带 Chromium) | 小(用系统 webview) | 最小 |
| 内存 | 较高 | 较低 | 最低 |
| 后端语言 | Node.js(JS) | Rust | 各自 |
| 成熟度/生态 | 最成熟 | 较新 | 最强但成本高 |

★ **关键认知**：**Tauri**(用系统自带的 webview + Rust 后端)是 Electron 最热门的轻量替代;Electron 胜在**生态成熟、Chromium 行为一致**(不依赖系统 webview 的差异)。⚠ 选型关键——**要"跨平台 UI 完全一致 + JS 全栈"选 Electron;要"体积小、愿碰点 Rust"选 Tauri**。

# 4. 一个最小应用

## 4.1 主进程入口

```javascript
// main.js —— 主进程入口
const { app, BrowserWindow } = require('electron')

function createWindow() {
  const win = new BrowserWindow({
    width: 800,
    height: 600
  })
  win.loadFile('index.html')        // 加载渲染进程的页面
}

app.whenReady().then(() => {
  createWindow()                    // 应用就绪后创建窗口
})
```

★ **关键认知**：Electron 应用从**主进程入口(main.js)** 启动——`app.whenReady()` 后创建 `BrowserWindow`(窗口),窗口加载一个 HTML 页面成渲染进程。⚠ 这个"main.js 建窗口 + index.html 是界面"的最小骨架,就是所有 Electron 应用的地基(第 3 篇展开)。

## 4.2 安全红线预览

```javascript
// 创建窗口时的两个"安全必备"选项(第 16 篇详解)
new BrowserWindow({
  webPreferences: {
    contextIsolation: true,   // 渲染进程上下文隔离(安全)
    nodeIntegration: false    // 渲染进程不直接给 Node(安全)
  }
})
```

★ **关键认知**：Electron 里有条贯穿始终的安全红线——**渲染进程默认不该有 Node 权限**(它加载的可能是远程内容)。⚠ `nodeIntegration: false` + `contextIsolation: true` 是官方默认与强烈建议;很多 Electron 安全漏洞都源于"图省事把 nodeIntegration 打开"(第 16 篇专章)。

# 5. 本篇小结与下一步

★ **关键认知**：Electron = Chromium(界面)+ Node(系统)的多进程框架——**主进程管系统、渲染进程管界面、IPC 桥接**。最大特点是"Web 技术写桌面、跨平台一致",最大代价是"体积大"。安全红线:渲染进程别给 Node。

下一步第 2 篇:环境搭建与快速上手——用 electron-vite 脚手架跑起第一个应用。