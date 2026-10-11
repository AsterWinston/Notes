Electron 原生开发要自己拼主进程、渲染进程、打包,略繁琐。**electron-vite** 脚手架把 Vite 的"快 + 热更新"带进 Electron,让开发体验接近前端。本篇讲清项目结构、启动流程、目录职责,以及热更新的原理。

# 1. 脚手架与项目结构

## 1.1 创建项目

```powershell
# 用 electron-vite 创建项目(推荐,基于 Vite)
npm create @quick-start/electron@latest my-app
# 交互选择:Vue 或 React、TypeScript、是否加 ESLint 等

cd my-app
npm install
npm run dev          # 启动开发(打开 Electron 窗口 + 热更新)
```

★ **关键认知**：`electron-vite` 是 Electron 生态的主流脚手架——底层 **Vite**(快、热更新),把"主进程 / 预加载 / 渲染进程"三个 Vite 构建产物**统一管理**。⚠ 它让你像开发前端一样开发 Electron:保存即热更新渲染进程、甚至主进程改动也可自动重启,体验远好于原生手搭。

## 1.2 三个目录

```
my-app/
├── src/
│   ├── main/          # 主进程代码(main.ts:创建窗口)
│   ├── preload/       # 预加载脚本(暴露安全 API)
│   └── renderer/      # 渲染进程(Vue/React 前端项目)
├── electron.vite.config.ts   # 三个 target 的 Vite 配置
└── package.json       # main 字段指向主进程产物
```

★ **关键认知**：electron-vite 的项目分**三个源目录**——`main`(主进程)、`preload`(预加载)、`renderer`(界面)。⚠ 它们**分别被 Vite 打包**成三个产物,再组装成一个 Electron 应用;理解"三个 target、三个产物",是理解 Electron 工程的关键。

# 2. 启动流程

## 2.1 dev 时发生了什么

```
npm run dev
  → Vite 起服务(renderer 走 HMR 热更新)
  → 构建 main/preload
  → Electron 启动,主进程加载 dev 服务器的 renderer
  → 你改 renderer 代码 → HMR 即时生效;改 main → 自动重启 Electron
```

★ **关键认知**：开发模式里,**渲染进程连的是 Vite dev server**(有 HMR),主进程/预加载则以构建产物运行。⚠ 这也解释了"为什么 dev 和 build 看到的不完全一样"——dev 是 dev server 即时编译,build 才是最终打包产物(第 13 篇)。

## 2.2 package.json 关键字段

```json
{
  "main": "./out/main/index.js",   // 主进程产物入口(Electron 从这里启动)
  "scripts": {
    "dev": "electron-vite dev",
    "build": "electron-vite build",
    "build:win": "electron-vite build && electron-builder --win"
  }
}
```

★ **关键认知**：`main` 字段指向**打包后主进程产物**的入口(Electron 启动读它);`electron-vite dev/build` 分别做开发与构建。⚠ `main` 指向的是 `out/` 下的构建产物**不是源码**,这是 Electron 与纯前端的差异点。

# 3. 主进程 / 预加载 / 渲染

## 3.1 三者的最小示例

```javascript
// src/main/index.js —— 主进程:创建窗口
const { app, BrowserWindow } = require('electron')
const { join } = require('path')

function createWindow() {
  const win = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: join(__dirname, '../preload/index.js'),  // 注入预加载
      contextIsolation: true,
      nodeIntegration: false
    }
  })
  // dev 加载 dev server,build 加载打包后的 index.html
  if (process.env.ELECTRON_RENDERER_URL) {
    win.loadURL(process.env.ELECTRON_RENDERER_URL)
  } else {
    win.loadFile(join(__dirname, '../renderer/index.html'))
  }
}
```

★ **关键认知**：主进程建窗口时,通过 `webPreferences.preload` 注入预加载脚本;渲染进程页面通过 `loadURL`(dev)/`loadFile`(prod)加载。⚠ 这套"**主进程 → 预加载 → 页面**"的连线,是每个 Electron 应用的固定起手式;`contextIsolation + nodeIntegration:false` 是安全默认。

## 3.2 为什么预加载在这个位置

★ **关键认知**：预加载脚本**介入主进程与渲染进程之间**——它跑在隔离的沙箱里,既能碰主进程能力,又能把"精选的、安全的 API"暴露给渲染进程。⚠ 这正是 Electron 的安全设计:渲染进程**不该**直接拿 Node 权限,而是通过预加载拿到**白名单 API**(第 4 篇详解)。

# 4. 开发体验

## 4.1 热更新

★ **关键认知**：electron-vite 的热更新分两档——**渲染进程 HMR**(改界面即时刷新,不重启窗口)与**主进程 auto-restart**(改 main 代码自动重启 Electron)。⚠ 这套"改界面秒见、改后端自动重启"的体验,是 Electron 现代开发与"改一次完整重新编译打包"的老方式的分水岭。

## 4.2 DevTools 调试

★ **关键认知**：开发时可在渲染进程打开 DevTools(和浏览器一样调 UI);主进程可直接断点调试。⚠ 渲染进程的调试就是"熟悉的浏览器 DevTools",这让前端开发者几乎零成本上手桌面调试。

# 5. 本篇小结与下一步

★ **关键认知**：electron-vite 统一三个 target(main/preload/renderer),dev 走 Vite dev server + HMR;主进程建窗口、注入 preload、加载页面;`main` 字段指向构建产物。

下一步第 3 篇:主进程与渲染进程——BrowserWindow、进程职责与生命周期,深入理解多进程模型。