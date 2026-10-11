桌面应用相比网页,最大的"桌面感"就来自窗口与托盘——多窗口、无边框自绘、最小化到系统托盘。本篇讲清 BrowserWindow 的常用选项、多窗口管理、无边框窗口的定制,以及 Tray 系统托盘的实现。

# 1. BrowserWindow 常用选项

## 1.1 窗口外观

```javascript
const win = new BrowserWindow({
  width: 900,
  height: 600,
  frame: false,          // 无边框(自绘标题栏时)
  resizable: true,
  alwaysOnTop: false,    // 是否置顶
  show: false,           // 创建时不显示(等 ready-to-show)
  backgroundColor: '#fff' // 背景色(防白屏)
})
win.once('ready-to-show', () => win.show())
```

★ **关键认知**：窗口外观靠构造选项——`frame`(边框)、`resizable`(可缩放)、`alwaysOnTop`(置顶)、`show: false`+`ready-to-show`。⚠ `frame: false` 是"**无边框窗口**"的开关(做自绘标题栏/登录窗/悬浮面板),代价是**失去系统标题栏**(要自己实现拖拽、关闭),见 2.3。

## 1.2 窗口内容加载

```javascript
win.loadFile('index.html')              // 加载本地文件
win.loadURL('https://example.com')      // 加载远程 URL
// loadURL 返回 Promise,失败可 catch 重试/错误页
```

★ **关键认知**：`loadFile` 加载本地页面(常态)、`loadURL` 加载远程(如第三方登录页)。⚠ 加载远程内容是**信任问题**——它等于在你的窗口里跑别人的代码,安全配置(第 16 篇)尤其要收紧。

# 2. 多窗口管理

## 2.1 维护窗口集合

```javascript
const { BrowserWindow } = require('electron')
const wins = new Set()

function createWindow(opts) {
  const win = new BrowserWindow(opts)
  wins.add(win)
  win.on('closed', () => wins.delete(win))   // 关闭后移出集合
  return win
}

// 需要时遍历:如主进程广播消息给所有窗口
function broadcast(channel, data) {
  for (const w of wins) w.webContents.send(channel, data)
}
```

★ **关键认知**：多窗口时**自己维护窗口集合**(Set/Map),关闭时移除;遍历集合可**广播**消息给所有窗口。⚠ `BrowserWindow.getAllWindows()` 也提供全部窗口,但自己维护集合能附带更多元数据(如"这是设置窗还是主窗")。

## 2.2 主窗与子窗

```javascript
// 父窗口打开子窗口(如设置页、预览窗)
const child = new BrowserWindow({ parent: mainWin, modal: false })
child.loadFile('settings.html')
```

★ **关键认知**：`parent` 选项让子窗**附属于父窗**(一起最小化、保持在父窗之上),`modal: true` 则**模态**(阻断父窗操作)。⚠ 多窗口是桌面应用的常态(主窗 + 设置/预览/关于),但要管好"谁是谁、怎么关",否则窗口满天飞。

# 3. 无边框窗口的自绘

## 3.1 自绘标题栏

```javascript
// 无边框窗口 + 页面内自绘标题栏(HTML/CSS 画"最小化/最大化/关闭")
new BrowserWindow({ frame: false })
```

```html
<!-- 渲染进程里,标题栏区域实现拖拽与按钮 -->
<div class="titlebar" style="-webkit-app-region: drag">  <!-- 可拖拽 -->
  <button style="-webkit-app-region: no-drag">关闭</button> <!-- 按钮不可拖 -->
</div>
```

★ **关键认知**：无边框窗口靠 **`-webkit-app-region: drag`** 让区域**可拖拽**(替代系统标题栏的拖动),按钮用 `no-drag` 排除。⚠ 这是 Electron 独有的 CSS 能力;忘了 `drag` 无边框窗口就"拖不动",忘了按钮的 `no-drag` 则按钮点了没反应。

## 3.2 通过 IPC 控制窗口

```javascript
// 自绘标题栏的"关闭/最小化"按钮,要通知主进程操作窗口
// 渲染:ipcRenderer.send('win:close')
// 主:ipcMain.on('win:close', (e) => BrowserWindow.fromWebContents(e.sender).close())
```

★ **关键认知**：自绘窗口的关闭/最小化按钮,要**渲染进程发 IPC → 主进程调 win.close()/minimize()**。⚠ 因为"控制窗口"是主进程权限,渲染进程只能请求;这是"职责划分"(第 3 篇)在窗口控制上的具体体现。

# 4. 系统托盘 Tray

## 4.1 创建托盘

```javascript
const { Tray, Menu, nativeImage } = require('electron')

let tray
app.whenReady().then(() => {
  const icon = nativeImage.createFromPath('./icon.png')
  tray = new Tray(icon)                    // 创建托盘图标
  tray.setToolTip('我的应用')              // 悬停提示
  const menu = Menu.buildFromTemplate([    // 托盘右键菜单
    { label: '显示主窗口', click: () => mainWin.show() },
    { label: '退出', click: () => app.quit() }
  ])
  tray.setContextMenu(menu)
})
```

★ **关键认知**：`Tray` 在**系统托盘**放一个图标,配右键菜单;点击事件能显示/隐藏窗口。⚠ 托盘的典型用途是"**关窗不退出、常驻后台**"(用户关窗口 → 隐藏 → 托盘图标仍在,点图标重开);注意托盘图标在 mac/win 对尺寸格式要求不同。

## 4.2 关窗进托盘

```javascript
let isQuitting = false
app.on('before-quit', () => { isQuitting = true })

mainWin.on('close', (e) => {
  if (!isQuitting) {
    e.preventDefault()       // 阻止真的关闭
    mainWin.hide()           // 改为隐藏到托盘
  }
})
```

★ **关键认知**："**点关闭其实是隐藏到托盘**"的实现——拦截 `close` 事件 `preventDefault`,改 `hide()`;真正退出时(菜单"退出")先置 `isQuitting` 标志再 `app.quit()`。⚠ **必须留一条真正的退出通道**(否则应用永远退不掉);这个"标志位区分'关窗'与'退出'"是托盘应用的标准写法。

# 5. 本篇小结与下一步

★ **关键认知**：窗口的桌面感来自 frame/置顶/多窗口/托盘;无边框靠 `-webkit-app-region: drag` 自绘;托盘用 Tray,关窗进托盘要"close 拦截 + 退出标志位"。多窗口要自己维护集合。

下一步第 7 篇:菜单与快捷键——应用菜单、上下文菜单、全局快捷键。