菜单是桌面应用"原生感"的重要来源——顶部应用菜单、右键上下文菜单、全局快捷键。本篇讲清 `Menu` 的构造与 role、macOS 菜单栏约定、`accelerator` 快捷键与跨平台修饰符映射、`setApplicationMenu` 与 `popup` 的分工,以及 `contextMenu` 右键菜单与 `globalShortcut` 全局快捷键。

# 1. 应用菜单 Menu

## 1.1 构造与挂载

```javascript
const { Menu } = require('electron')

const menu = Menu.buildFromTemplate([
  {
    label: '文件',
    submenu: [
      { label: '新建', accelerator: 'CmdOrCtrl+N', click: () => { /* 新建 */ } },
      { type: 'separator' },                                   // 分隔线
      { role: 'quit', label: '退出' }                          // 内置 role
    ]
  },
  {
    label: '编辑',
    submenu: [
      { role: 'undo', label: '撤销' },
      { role: 'redo', label: '重做' }
    ]
  }
])

Menu.setApplicationMenu(menu)   // 设为应用菜单(Windows/Linux 显示在窗口顶部)
```

★ **关键认知**：`Menu.buildFromTemplate` 用**模板数组**(label/submenu)构造菜单,`Menu.setApplicationMenu` 挂载。⚠ `type: 'separator'` 是分隔线;子菜单里可用**内置 `role`**(`undo`/`copy`/`quit` 等),role 会**自动实现标准行为与本地化**,比自己 click 实现省心。

## 1.2 role 的价值

★ **关键认知**：`role` 是**内置的菜单项行为**——`role: 'undo'` 自动做撤销、`role: 'copy'` 自动复制,还带**平台正确的快捷键与文案**。⚠ 能用 role 就别写 click:复制/粘贴/剪切/全选/退出这些,role 都现成且符合平台习惯;只有"真正自定义的动作"才写 click。

role 还有一批**整组模板**:`appMenu`/`fileMenu`/`editMenu`/`viewMenu`/`windowMenu` 一键生成对应子系统。macOS 有两条硬约定——**第一个菜单会被系统渲染成"应用名"菜单**(about/services/hide/quit 放这里,没有它连"退出"都不规范),**windowMenu 有系统特殊行为**;所以跨平台工程常见写法是分叉拼模板:

```javascript
const template = [
  ...(process.platform === 'darwin' ? [{ role: 'appMenu' }] : []),
  { role: 'fileMenu' },
  { role: 'editMenu' },
  { role: 'viewMenu' },
  { role: 'windowMenu' },
  { label: '帮助', submenu: [{ label: '关于', click: () => showAbout() }] }
]
```

## 1.3 三种来源与动态菜单项

```text
1. 默认菜单:  什么都不设时,Electron 自动生成一份(含 reload/devtools/quit)
2. 模板菜单:  buildFromTemplate + setApplicationMenu 完全接管(全局唯一,后设覆盖先设)
3. 无菜单:    Menu.setApplicationMenu(null) → 顶部栏消失(无边框/自绘 UI 常用)
```

```javascript
{
  label: '查看',
  submenu: [
    { type: 'checkbox', label: '状态栏', checked: showStatus, click: (item) => toggleStatus(item.checked) },
    { type: 'radio', label: '紧凑', checked: density === 'compact' },   // 相邻 radio 自动成组
    { label: '刷新', enabled: online, accelerator: 'CmdOrCtrl+R' }      // enabled: false 置灰
  ]
}
// 模板是一次性快照:状态变了要重新 buildFromTemplate + setApplicationMenu
```

★ **关键认知**：发布版**不建议把菜单 null 掉**——Windows 用户会找不到复制粘贴入口,macOS 没有应用菜单连"关于/退出"都不规范,正确做法是"精简角色"而非"清空";`checkbox/radio/enabled/visible` 让菜单项带状态,但**菜单不是响应式的**,数据变了必须重建。⚠ 高频状态别塞进菜单,重建有闪烁与性能成本。

# 2. 快捷键 accelerator

## 2.1 绑定快捷键

```javascript
{ label: '保存', accelerator: 'CmdOrCtrl+S', click: () => save() }
// CmdOrCtrl = macOS 的 Cmd、Windows/Linux 的 Ctrl(跨平台友好)
// 其他:Shift+Alt+Cmd+F、Cmd+1、Up、F5 等

// 修饰符跨平台映射:
//   CmdOrCtrl/CommandOrControl → mac:⌘  其他:Ctrl
//   Alt  → mac:⌥(Option), 其他:Alt;而 Option 在 Win/Linux 是 AltGr,慎用
//   Super → mac 上等同 Cmd;Windows 上是 Win 键,语义不同,别当跨平台关键字用
```

★ **关键认知**：`accelerator` 给菜单项绑快捷键;**`CmdOrCtrl`** 是关键跨平台写法(macOS 用 Cmd,Windows/Linux 用 Ctrl)。⚠ 用 `CmdOrCtrl` 而非硬编码 `Ctrl`,一处代码三端都对;组合键用 `Shift+Alt+X` 这种 `+` 连接。

★ **关键认知**：三个易错映射——**macOS 的 `Alt` 是 ⌥ 不是 AltGr**;`Cmd+W`/`Cmd+Q` 系统级强占,自定义菜单抢不到;`+` 键要写 `Plus`(裸 `+` 会被当分隔符)。⚠ 菜单里不想显示但想生效的快捷键,`{ label, accelerator, visible: false }` 即可——**hidden 项的 accelerator 依然注册**。

## 2.2 快捷键冲突与仅注册

★ **关键认知**：快捷键在**菜单里**是"随菜单注册";若某动作不需要菜单项、只要快捷键,可用 `globalShortcut`(见下)或"菜单里藏一个不可选项"。⚠ 快捷键要在不同平台的手感上斟酌(如 `Cmd+Q` 退出是 macOS 标配),避开系统已有组合。

```javascript
// 窗口内快捷键的兜底代理:菜单 accelerator 没覆盖的按键在此截获
win.webContents.on('before-input-event', (event, input) => {
  if (input.type === 'keyDown' && input.control && input.key.toLowerCase() === 'k') {
    openCommandPalette()
    event.preventDefault()      // 拦下来,不再传给页面
  }
})
```

★ **关键认知**：按键处理有层级链——**globalShortcut(系统级)最先截 → 菜单 accelerator → `before-input-event` → 页面 keydown**。⚠ 同一组合多处注册会"无声失效"(不报错,先注册的赢),排查冲突沿这条链从上往下查;`globalShortcut.isRegistered()` 可先探测,`preventDefault()` 只拦"传给页面",拦不住已生效的菜单 accelerator。

# 3. 右键菜单 contextMenu

## 3.1 页面右键菜单

```javascript
// 主进程:监听某窗口的 context-menu 事件,弹出自定义右键菜单
win.webContents.on('context-menu', (event, params) => {
  const items = []
  if (params.selectionText) items.push({ role: 'copy' })
  if (params.mediaType === 'image') items.push({ label: '图片另存为…', click: () => saveImage(params.srcURL) })
  items.push({ label: '重新加载', click: () => win.webContents.reload() })
  Menu.buildFromTemplate(items).popup({ window: win })
})
```

★ **关键认知**：`webContents` 的 **`context-menu`** 事件在页面右键时触发,`menu.popup()` 弹出菜单。⚠ 这是"页面右键自定义菜单"的标准做法;`params` 能拿到右键处信息(是否选中文字、图片 URL 等),据此动态生成菜单项。

## 3.2 setApplicationMenu 与 popup 的分工

```text
setApplicationMenu(menu)  应用菜单:常驻、全局唯一、macOS 在屏幕顶部
menu.popup({window,x,y})  浮出菜单:弹一次就销毁,右键/托盘/按钮下拉都用它
menu.closePopup()         主动收起(托盘左键切换菜单时常用)
```

★ **关键认知**：两类菜单**生命周期不同**——应用菜单是"挂着的",popup 是"用完即弃"的,每次右键重建 popup 反而保证 `params` 数据新鲜。⚠ popup 不给 x/y 就在**鼠标位置**弹出(macOS 托盘需手动定位到图标下方);同一个 Menu 实例**别重复 popup**(macOS 行为异常),每次新建最稳。

# 4. 全局快捷键 globalShortcut

## 4.1 注册全局快捷键

```javascript
const { globalShortcut } = require('electron')

app.whenReady().then(() => {
  const ok = globalShortcut.register('CmdOrCtrl+Shift+K', () => {
    mainWin.show()          // 应用没焦点时也能触发
  })
  if (!ok) console.log('注册失败(可能被占用)')
})

app.on('will-quit', () => globalShortcut.unregisterAll())  // 退出前注销
```

★ **关键认知**：`globalShortcut` 注册**系统级全局快捷键**——**应用没焦点时也生效**(如"全局唤起窗口")。⚠ 它是"全局"的,所以**可能与其他应用冲突**(register 返回 false 表示没注册上);退出前要 `unregisterAll` 清理,否则快捷键一直占着。

## 4.2 边界与坑

★ **关键认知**：globalShortcut 抢不过**操作系统保留**组合(macOS 的 `Cmd+Space` Spotlight、Windows 的 `Win+L`),成败**以返回值为准**,别假设"写了就生效"。⚠ 同一组合**第二次注册会覆盖第一次**;多窗口共享全局快捷键时,回调里要用 `BrowserWindow.getFocusedWindow()` 或约定好的主窗口决定"唤起哪个",别随手 `win.show()` 第一个窗口。注册要放在 `app.whenReady` 之后。

# 5. 菜单实践小结

| 需求 | API |
|------|-----|
| 顶部应用菜单 | `Menu.buildFromTemplate` + `setApplicationMenu` |
| 标准动作(复制/退出) | `role` |
| 菜单快捷键 | `accelerator` + `CmdOrCtrl` |
| 页面右键菜单 | `webContents context-menu` + `menu.popup` |
| 全局快捷键 | `globalShortcut.register` |

# 6. 本篇小结与下一步

★ **关键认知**：菜单有默认/模板/无三种来源,应用菜单全局唯一、状态变化要重建;role 复用标准动作(还有 appMenu/editMenu 整组模板 + macOS 第一菜单约定),accelerator 用 `CmdOrCtrl` 跨平台映射(Alt 在 mac 是 ⌥、`Plus` 转义、hidden 项照样注册);冲突按"globalShortcut → 菜单 → before-input-event → 页面"链路排查;右键菜单靠 context-menu + popup(用完即弃);全局快捷键以返回值为准、退出必清理。这些是"原生桌面感"的核心构件。

下一步第 8 篇:对话框与通知——dialog 系统对话框、Notification 系统通知。
