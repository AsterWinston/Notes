菜单是桌面应用"原生感"的重要来源——顶部应用菜单、右键上下文菜单、全局快捷键。本篇讲清 `Menu` 的构造与 role、`accelerator` 快捷键、`contextMenu` 右键菜单,以及 `globalShortcut` 全局快捷键。

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

# 2. 快捷键 accelerator

## 2.1 绑定快捷键

```javascript
{ label: '保存', accelerator: 'CmdOrCtrl+S', click: () => save() }
// CmdOrCtrl = macOS 的 Cmd、Windows/Linux 的 Ctrl(跨平台友好)
// 其他:Shift+Alt+Cmd+F、Cmd+1、Up、F5 等
```

★ **关键认知**：`accelerator` 给菜单项绑快捷键;**`CmdOrCtrl`** 是关键跨平台写法(macOS 用 Cmd,Windows/Linux 用 Ctrl)。⚠ 用 `CmdOrCtrl` 而非硬编码 `Ctrl`,一处代码三端都对;组合键用 `Shift+Alt+X` 这种 `+` 连接。

## 2.2 快捷键冲突与仅注册

★ **关键认知**：快捷键在**菜单里**是"随菜单注册";若某动作不需要菜单项、只要快捷键,可用 `globalShortcut`(见下)或"菜单里藏一个不可选项"。⚠ 快捷键要在不同平台的手感上斟酌(如 `Cmd+Q` 退出是 macOS 标配),避开系统已有组合。

# 3. 右键菜单 contextMenu

## 3.1 页面右键菜单

```javascript
// 主进程:监听某窗口的 context-menu 事件,弹出自定义右键菜单
win.webContents.on('context-menu', (event, params) => {
  const menu = Menu.buildFromTemplate([
    { label: '复制', role: 'copy' },
    { label: '粘贴', role: 'paste' },
    { label: '自定义动作', click: () => { /* ... */ } }
  ])
  menu.popup({ window: win })
})
```

★ **关键认知**：`webContents` 的 **`context-menu`** 事件在页面右键时触发,`menu.popup()` 弹出菜单。⚠ 这是"页面右键自定义菜单"的标准做法;`params` 能拿到右键处信息(是否选中文字、图片 URL 等),据此动态生成菜单项。

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

# 5. 菜单实践小结

| 需求 | API |
|------|-----|
| 顶部应用菜单 | `Menu.buildFromTemplate` + `setApplicationMenu` |
| 标准动作(复制/退出) | `role` |
| 菜单快捷键 | `accelerator` + `CmdOrCtrl` |
| 页面右键菜单 | `webContents context-menu` + `menu.popup` |
| 全局快捷键 | `globalShortcut.register` |

# 6. 本篇小结与下一步

★ **关键认知**：菜单用模板构造 + role 复用标准动作 + accelerator 绑快捷键(CmdOrCtrl 跨平台);右键菜单靠 webContents context-menu;全局快捷键用 globalShortcut(注意冲突与清理)。这些是"原生桌面感"的核心构件。

下一步第 8 篇:对话框与通知——dialog 系统对话框、Notification 系统通知。