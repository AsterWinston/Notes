宣称"跨平台"容易,真正三端都跑好很难——路径、通知、快捷键、窗口行为,各平台都有细微差异。本篇把 Electron 跨平台的常见坑点归类讲清,并给出"至少三端各测一次"的排查心法。

# 1. 路径与文件系统

## 1.1 路径分隔符与大小写

```javascript
const path = require('path')
// ✗ 硬编码路径分隔符:"C:\Users\..." 与 "/home/..." 不通用
// ✓ 用 path 模块拼
const p = path.join(app.getPath('userData'), 'data', 'config.json')
```

★ **关键认知**：Windows 用 `\`、macOS/Linux 用 `/`,而 **`path.join`/`path.sep` 交给 Node 处理**。⚠ 别手写路径分隔符、别拼字符串;还要注意 **Windows 路径不区分大小写、macOS/Linux 区分**(文件名大小写敏感),以及 `app.getPath`(第 10 篇)帮你定位系统目录。

## 1.2 环境变量与主目录

★ **关键认知**：跨平台取"用户主目录、临时目录、桌面"等,一律用 `app.getPath` 系而非 `process.env.HOME`/`USERPROFILE`(各平台变量名不同)。⚠ "**系统标准目录用 getPath,路径拼接用 path.join**"——这两条能避开 80% 的跨平台路径坑。

# 2. 窗口与菜单行为

## 2.1 关窗 ≠ 退出的平台差异

```javascript
// macOS:关完窗口应用仍在(点 Dock 激活);Windows/Linux:关完窗口默认退出
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit()   // 第 3 篇讲过
})
```

★ **关键认知**：**窗口生命周期有平台差异**——macOS 关窗不退出(需 activate 重建窗口),Windows/Linux 关窗退出。⚠ 这是最经典的平台差异(第 3 篇);所有"关窗/托盘/退出"逻辑都要按 `process.platform` 分支测。

## 2.2 菜单的位置差异

★ **关键认知**：**macOS 的应用菜单在系统顶部菜单栏**,Windows/Linux 在每个窗口顶部——`Menu.setApplicationMenu` 的表现不同。⚠ 所以菜单项(尤其 mac 的"关于/退出")在 macOS 有约定俗成的位置,Windows 下则随窗口;要三端各看一遍菜单效果。

# 3. 通知与快捷键

## 3.1 Notification 的平台差异

★ **关键认知**：通知在三端表现不同——**macOS 样式统一且需授权、Windows 依赖 AppUserModelId(应用标识)、Linux 靠桌面环境(可能不支持)**。⚠ 所以"通知能不能弹"是三端差异大而常被忽略的点;发布前务必三端各验证通知。

## 3.2 全局快捷键冲突

★ **关键认知**：`globalShortcut`(第 7 篇)**不同平台系统已占用的快捷键不同**(macOS 的 Cmd+Space 是 Spotlight、Windows 一堆系统组合)。⚠ 选全局快捷键要**避开系统常用组合**,且 `register` 返回 false 时要有降级(提示用户换个键)。

# 4. 安装与权限

## 4.1 安装位置与权限

★ **关键认知**：安装/数据目录在三端不同——Windows 装到 `Program Files`(需管理员)、macOS 装 `Applications`、Linux 各发行版;数据目录都归 `userData`。⚠ 又一次指向:数据写 `userData`(有权限)、别写安装目录(可能只读)——这与第 10 篇呼应,也是三端最常见的数据写入坑。

## 4.2 系统权限申请

★ **关键认知**：macOS 有**隐私权限**(摄像头、麦克风、通知、完全磁盘访问)需在 Info.plist 声明+用户授权;Windows/Linux 各自有权限模型。⚠ 用了相机/麦克风/键盘监听等能力,要**按平台声明权限**并处理"用户拒绝"的情况,否则功能静默失效。

# 5. 跨平台排查心法

## 5.1 平台差异速查

| 域 | 差异 |
|----|------|
| 路径 | 分隔符 \ vs /、大小写敏感 |
| 窗口 | macOS 关窗不退出 |
| 菜单 | macOS 顶部菜单栏 |
| 通知 | 授权/标识/桌面环境差异 |
| 快捷键 | 系统占用各自不同 |
| 权限 | macOS 隐私权限、Windows 管理员 |

★ **关键认知**：跨平台的本质是"**API 统一,行为各异**"——Electron 的 API 是跨平台的,但**呈现/行为/权限有平台差异**。⚠ 排查心法:**遇到"某平台不对"的问题,先查是不是平台差异**,再查代码逻辑;同时"三端各测一遍"是发布前的底线。

## 5.2 条件平台的写法

```javascript
if (process.platform === 'darwin') { /* macOS 专用逻辑 */ }
if (process.platform === 'win32')  { /* Windows 专用 */ }
// 把平台差异集中、显式地写,而不是隐式地"碰巧能跑"
```

★ **关键认知**：平台差异要用 `process.platform === 'xxxx'`**显式分支**(darwin/win32/linux),集中、可读。⚠ 别靠"在 Windows 上碰巧没报错就以为跨平台";把差异显式化,既是文档,也方便三端定位。

# 6. 本篇小结与下一步

★ **关键认知**：跨平台坑集中在"路径(用 path/getPath)、窗口(mac 关窗不退出)、菜单(mac 顶部)、通知/快捷键(平台差异大)、权限(按平台声明)"。心法:API 统一但行为各异,差异要显式分支,三端各测一遍。

下一步第 21 篇:面试要点与总结——Electron 高频面试题 + 全系列知识地图,收官。