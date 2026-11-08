桌面应用发版后,怎么让用户"下次打开就更新"?这就是**自动更新**。Electron 生态的主流方案是 `electron-updater`——应用启动/定时检查新版本,下载并安装。本篇讲清更新链路的完整流程、feedURL 与更新源、electron-updater 与构建工具的解耦、delta 更新,以及灰度、回滚与断网降级策略。

# 1. 更新的完整流程

```text
┌────────────────────────────────────────────────┐
│ 发布侧                                          │
│  electron-builder 产出 → exe/dmg + latest.yml   │
│         └→ 上传到更新源(GitHub Release / CDN)  │
└──────────────────────┬─────────────────────────┘
                       │ GET latest.yml(比对 version)
┌──────────────────────▼─────────────────────────┐
│ 客户端(electron-updater)                        │
│  checkForUpdates → 有新版本 → 下载+sha512校验    │
│    → 暂存到 userData 缓存目录                    │
│    → 提示用户 → quitAndInstall → 安装器接管      │
│    → 新版本启动,旧进程退出                       │
└────────────────────────────────────────────────┘
```

★ **关键认知**：自动更新的链路是——**发布到服务器 → 应用 `checkForUpdates` → 比对版本 → 下载 → 安装**。⚠ 更新的"可行前提"是**有签名 + 版本号规律 + 一个可访问的更新服务器(或 GitHub Releases)**;缺签名/缺服务器,更新无从谈起。注意**下载与安装分离**:下载在运行期完成,真正的"替换文件"发生在退出重启时(由 NSIS 安装器执行),所以更新永远"下次启动生效"。

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
autoUpdater.on('download-progress', p => {
  mainWin.webContents.send('update:progress', p.percent)   // 推 UI
})
autoUpdater.on('update-downloaded', () => {})      // 下载完,可提示重启安装
autoUpdater.on('error', err => console.error(err)) // 断网/404/校验失败都走这里
```

★ **关键认知**：`electron-updater` 提供 `checkForUpdates` + 一系列事件(`update-available`/`download-progress`/`update-downloaded`)。⚠ 典型做法:**启动时静默检测,下载进度推给 UI,下载完提示"重启以安装"**,再由 `quitAndInstall()` 安装重启。`checkForUpdates()` **不要 await 阻塞启动**;`error` 事件必接,否则断网时"检查了但永远没动静"无从排查。

## 2.2 触发安装重启

```javascript
autoUpdater.on('update-downloaded', () => {
  // 推给渲染:弹窗"新版本已就绪,是否重启安装?"
  mainWin.webContents.send('update:ready')
})
// 用户点"重启" → 主进程:
ipcMain.on('update:install', async () => {
  if (await hasUnsavedWork()) { askUserToSave(); return }   // 先确认没有未保存工作
  autoUpdater.quitAndInstall(true, true)   // isSilent 静默装 + 装完自动拉起
})
```

★ **关键认知**：`quitAndInstall()` 退出应用并安装更新。⚠ 安装前最好**先提示用户保存工作**(重启会关闭应用),别冷不丁自动重启造成数据丢失;自动更新 + 好提示,才是完整体验。参数组合 `(isSilent, isForceRunAfter)` 是桌面应用常见形态;`autoInstallOnAppQuit` 是兜底——用户一直不点"重启",下次自然退出时也会装上。

# 3. 更新服务器与发布

## 3.1 用 GitHub Releases(最省事)

```javascript
// electron-builder 配置里:publish 指向 GitHub Releases
"publish": [{ "provider": "github", "owner": "...", "repo": "..." }]
// 每次发版把安装包传到 Release,electron-updater 自动去那里查
// 客户端也可运行时覆盖:
autoUpdater.setFeedURL({ provider: 'github', owner: 'me', repo: 'myapp' })
```

★ **关键认知**：**GitHub Releases 是免费而省事的更新源**——把产物发到 Release,electron-updater 自动拉取比对。⚠ 适合开源/个人项目,零服务器成本;注意国内网络访问 GitHub 可能不稳,商业产品可能要考虑国内可达的更新源/CDN。

## 3.2 自建更新源

```javascript
// generic:静态目录即更新源(nginx/CDN 挂几个文件就能跑)
autoUpdater.setFeedURL({
  provider: 'generic',
  url: 'https://dl.example.com/electron/win/',    // 分平台目录:win/ mac/ linux/
  channel: 'latest'                                // 对应 latest.yml
})
// 目录里放: latest.yml + MyApp Setup 1.2.0.exe(+ .blockmap)
autoUpdater.requestHeaders = { Authorization: `Bearer ${token}` }  // 私有源鉴权
```

★ **关键认知**：商业/内网产品常**自建更新服务器**——`provider: 'generic'` 只要求"一个静态目录 + 正确的 yml 文件名",CDN 同步 `latest.yml`/安装包/`.blockmap` 三件套即可,换来**可控、国内可达、可鉴权**;`channel` 对应 yml 文件名(`latest.yml`/`beta.yml`),这就是**多渠道**的实现。⚠ generic 源的坑全是运维坑:**yml 里的 sha512 与包不匹配 → error "校验失败"**;CDN 缓存旧 yml → 用户反复横跳"已最新/有更新"(发版后主动刷缓存);url 不带平台子目录时,mac 的 zip 与 win 的 exe 混排会解析错文件。

## 3.3 与构建工具解耦、delta 更新

```text
electron-builder(打包) 与 electron-updater(更新) 是解耦的两半:
  builder 负责"生产 latest.yml + 安装包 + blockmap"
  updater 负责"消费这三样",它不关心你用 builder 还是自研脚本产出
  → 换打包脚手架,更新端代码零改动;手写 yml 也能喂 updater
```

★ **关键认知**：更新协议本质是**文件布局约定**(元数据 yml + 安装包 + 校验和),所以"用什么构建工具"与"怎么自动更新"互不牵连。⚠ **delta 增量更新仅 Windows(NSIS)支持**:靠 `.blockmap` 只下差异块,小改动可省 60-90% 流量,默认自动尝试、失败回退全量;mac/Linux 无 delta——体积敏感的 Windows 产品**别把 .blockmap 从产物里裁掉**,发版 CDN 也要同步它。

# 4. 灰度与回滚

## 4.1 灰度发布

```text
不一次推给所有用户,而是分批:
  先推 5% → 观察崩溃率/反馈 → 没问题 → 50% → 100%

实现手段(按省事排序):
  1. channel 分池: 先发 beta.yml,白名单用户 setFeedURL 指 beta
  2. 服务端分流: 检查更新前先请求自家 API,按设备 ID 哈希
     决定"给你哪个 yml 的 URL"(经典做法,比例即哈希阈值)
  3. 分目录: /gray/ 与 /stable/ 两份 yml,按人群指 feedURL
```

★ **关键认知**：**灰度(分阶段发布)**——新版本先给小部分用户,观察无问题再逐步放量。⚠ electron-updater **没有内置灰度开关**,通用做法是把"要不要给你新 yml"的决策上移到自家 API;这是"大用户量桌面应用"降低风险的关键,用户一多,一次性全员更新的"翻车"代价太大。

## 4.2 版本回退策略

★ **关键认知**：更新出严重问题时,**回退**——让用户装回旧版(重新发布旧版本号更高的"修复版",或提供旧安装包)。⚠ 因此**版本号要精心管理**(semver),且"发布前保留可回退的旧产物";更新是"有风险的操作",留后路是成熟做法。注意 semver 的硬规则:updater 只认"版本号变大",**同版本号重传不会触发更新**,内部灰度回滚常用"旧版本+1 的修复版"而不是降级。

## 4.3 断网与降级

★ **关键认知**：更新链路必须**默认降级**——断网/超时/校验失败时静默放弃本轮,旧版照常可用;只有硬故障(如协议不兼容)才弹"请手动更新"。⚠ 两个边界:更新包暂存在 `userData`,**磁盘满**会下到一半失败(归入 error);公司代理/防火墙拦掉 CDN 域名是"部分用户永远收不到更新"的头号原因——给"检查请求发出去但没结果"埋点,能区分这两类问题。

# 5. 实务清单

| 点 | 说明 |
|----|------|
| 检测时机 | 启动静默检测 + 定时检查 |
| 下载 | autoDownload 自动,进度推 UI |
| 安装 | 下载完提示,用户确认后 quitAndInstall |
| 源 | GitHub Releases(省事)/generic+CDN(可控) |
| 增量 | Win 的 blockmap/delta,CDN 要同步 |
| 灰度 | channel/服务端分流,分批放量 |
| 回退 | 版本号管理 + 保留旧包 |
| 降级 | error 静默记日志,不打扰用户 |

# 6. 本篇小结与下一步

★ **关键认知**：自动更新 = 发布到源 + electron-updater 检测/下载/安装;下载与安装分离、永远"下次启动生效";事件推 UI 且必接 error。GitHub Releases 最省事,generic+channel 实现自建源与多渠道;builder 与 updater 解耦在"文件布局约定",Win delta 靠 blockmap 省流量;灰度靠分流 yml、回退靠版本管理、断网默认静默降级。核心是"更新要可观测、可回退、有提示"。

下一步第 15 篇:性能优化——启动速度、内存、多窗口开销与 GPU。
