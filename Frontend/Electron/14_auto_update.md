桌面应用发版后,怎么让用户"下次打开就更新"?这就是**自动更新**。Electron 生态的主流方案是 `electron-updater`——应用启动/定时检查新版本,下载并安装。本篇讲清更新流程、更新服务器配置、以及灰度与回滚策略。

# 1. 更新的完整流程

```
服务器(放发布产物 + metadata)
  ↑ 检查(electron-updater 请求最新版本信息)
应用启动/定时 → 发现新版本 → 下载 → 校验 → 安装(下次启动生效) → 提示用户
```

★ **关键认知**：自动更新的链路是——**发布到服务器 → 应用 `checkForUpdates` → 比对版本 → 下载 → 安装**。⚠ 更新的"可行前提"是**有签名 + 版本号规律 + 一个可访问的更新服务器(或 GitHub Releases)**;缺签名/缺服务器,更新无从谈起。

# 2. electron-updater

## 2.1 基本用法

```javascript
import { autoUpdater } from 'electron-updater'

autoUpdater.autoDownload = true          // 发现新版本自动下载
autoUpdater.autoInstallOnAppQuit = true  // 退出应用时自动安装

app.whenReady().then(() => {
  autoUpdater.checkForUpdates()          // 启动时检查(不要 await 阻塞)
})

// 监听事件,推给渲染进程显示进度
autoUpdater.on('update-available', info => {} )    // 有更新
autoUpdater.on('update-not-available', () => {})   // 已最新
autoUpdater.on('download-progress', p => {})       // 下载进度
autoUpdater.on('update-downloaded', () => {})      // 下载完,可提示重启安装
```

★ **关键认知**：`electron-updater` 提供 `checkForUpdates` + 一系列事件(`update-available`/`download-progress`/`update-downloaded`)。⚠ 典型做法:**启动时静默检测,下载进度推给 UI,下载完提示"重启以安装"**,再由 `quitAndInstall()` 安装重启。

## 2.2 触发安装重启

```javascript
autoUpdater.on('update-downloaded', () => {
  // 推给渲染:弹窗"新版本已就绪,是否重启安装?"
  mainWin.webContents.send('update:ready')
})
// 用户点"重启" → 主进程:
ipcMain.on('update:install', () => autoUpdater.quitAndInstall())
```

★ **关键认知**：`quitAndInstall()` 退出应用并安装更新。⚠ 安装前最好**先提示用户保存工作**(重启会关闭应用),别冷不丁自动重启造成数据丢失;自动更新 + 好提示,才是完整体验。

# 3. 更新服务器与发布

## 3.1 用 GitHub Releases(最省事)

```javascript
// electron-builder 配置里:publish 指向 GitHub Releases
"publish": [{ "provider": "github", "owner": "...", "repo": "..." }]
// 每次发版把安装包传到 Release,electron-updater 自动去那里查
```

★ **关键认知**：**GitHub Releases 是免费而省事的更新源**——把产物发到 Release,electron-updater 自动拉取比对。⚠ 适合开源/个人项目,零服务器成本;注意国内网络访问 GitHub 可能不稳,商业产品可能要考虑国内可达的更新源/CDN。

## 3.2 自建更新源

★ **关键认知**：商业/内网产品常**自建更新服务器**(静态文件服务器 + metadata + 正确的版本目录结构)。⚠ 它需要额外运维,但换来**可控的更新源、国内可达、可鉴权**;electron-builder 支持 generic 等 provider。

# 4. 灰度与回滚

## 4.1 灰度发布

```
不一次推给所有用户,而是分批:
  先推 5% → 观察崩溃率/反馈 → 没问题 → 50% → 100%
```

★ **关键认知**：**灰度(分阶段发布)**——新版本先给小部分用户,观察无问题再逐步放量。⚠ 这是"大用户量桌面应用"降低风险的关键;简单产品可跳过,但用户一多,一次性全员更新的"翻车"代价太大。

## 4.2 版本回退策略

★ **关键认知**：更新出严重问题时,**回退**——让用户装回旧版(重新发布旧版本号更高的"修复版",或提供旧安装包)。⚠ 因此**版本号要精心管理**(semver),且"发布前保留可回退的旧产物";更新是"有风险的操作",留后路是成熟做法。

# 5. 实务清单

| 点 | 说明 |
|----|------|
| 检测时机 | 启动静默检测 + 定时检查 |
| 下载 | autoDownload 自动,进度推 UI |
| 安装 | 下载完提示,用户确认后 quitAndInstall |
| 源 | GitHub Releases(省事)/自建(可控) |
| 灰度 | 分批放量,观察再全量 |
| 回退 | 版本号管理 + 保留旧包 |

# 6. 本篇小结与下一步

★ **关键认知**：自动更新 = 发布到源 + electron-updater 检测/下载/安装;事件推 UI;GitHub Releases 最省事,商业产品常自建;灰度降风险、版本管理留回退。核心是"更新要可观测、可回退、有提示"。

下一步第 15 篇:性能优化——启动速度、内存、多窗口开销与 GPU。