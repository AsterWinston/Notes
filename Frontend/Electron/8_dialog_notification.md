网页没有"系统对话框"和"系统通知"——这是桌面应用才有的能力。Electron 的 `dialog` 提供打开/保存文件、消息框,`Notification` 弹出系统通知。本篇讲清它们的用法、模态与焦点管理、"原生对话框 vs 自绘窗口"的取舍、通知权限与平台差异,以及主/渲染进程的分工。

# 1. dialog 系统对话框

## 1.1 打开 / 保存文件

```javascript
const { dialog } = require('electron')

// 打开文件:返回选中的文件路径
async function openFile(win) {
  // 第一参数传 win → 模态挂在该窗口上(macOS 变 sheet 下滑)
  const result = await dialog.showOpenDialog(win, {
    title: '选择文件',
    filters: [
      { name: '文本文件', extensions: ['txt', 'md'] },   // 文件类型过滤
      { name: '所有文件', extensions: ['*'] }
    ],
    properties: ['openFile']        // 或 multiSelections 多选
  })
  if (!result.canceled) {
    return result.filePaths[0]      // 用户选中的路径
  }
}

// 保存文件:返回要写入的路径
const save = await dialog.showSaveDialog(win, {
  title: '保存到',
  defaultPath: 'untitled.txt'
})
```

★ **关键认知**：`showOpenDialog` / `showSaveDialog` 分别弹**打开/保存文件**对话框,返回 `{ canceled, filePaths }`。⚠ **`dialog` 只能在主进程用**——渲染进程要弹文件对话框,得走 IPC 请主进程调(这是"职责划分"的又一例)。`filters` 过滤文件类型,是体验的关键。

## 1.2 消息框

```javascript
const choice = await dialog.showMessageBox(win, {
  type: 'warning',               // none/info/error/question/warning
  title: '确认',
  message: '确定要删除吗?',
  detail: '该操作无法撤销,将同时删除磁盘上的文件',   // 次级说明文字
  buttons: ['取消', '删除'],      // 按钮数组
  defaultId: 0,
  cancelId: 0,
  checkboxLabel: '不再提示',      // 附带勾选框(配合 checkboxChecked 读回)
  noLink: true                   // Windows:按 Windows 惯例重排按钮顺序
})
if (choice.response === 1 && !choice.checkboxChecked) { /* 点了"删除" */ }
```

★ **关键认知**：`showMessageBox` 弹**消息/确认框**,`buttons` 定义按钮,`response` 返回点的是第几个。⚠ 用原生确认框 vs 页面内自绘弹窗的取舍:**系统对话框更"原生、一致"**,但**无法定制样式**;需要精美 UI 的用页面弹窗,需要简单可靠确认的用 `showMessageBox`。

## 1.3 show 与 showModal:模态与焦点归属

```javascript
const { dialog, BrowserWindow } = require('electron')

dialog.showMessageBox(win, opts)                    // ✓ 推荐:第一参数传父窗口
dialog.showMessageBoxSync(win, opts)                // 同步版:阻塞主进程,尽量少用

// "无主窗口"时弹框:对话框不属于任何窗口
dialog.showMessageBox({ message: '启动失败' })      // 没有 win 参数 → 独立浮动模态
```

```text
模态与焦点的行为差异:
showMessageBox(win)  → 挂到 win:win 禁用、对话框居中其上、macOS 呈 sheet 下滑
showMessageBox()     → 无主:浮在桌面中间,任务栏可能单列一个窗口
无主 + macOS         → sheet 无处可挂,退化成独立窗口,观感突变
```

★ **关键认知**：**第一个参数传不传父窗口,决定对话框的"模态归属"**——传了就是"该窗口的模态"(父窗口禁用、焦点随关闭归还父窗口);没传就是"独立模态",父窗口的其他窗口仍可乱点,焦点管理全靠系统。⚠ 两个坑:**`showMessageBoxSync` 会阻塞主进程事件循环**,期间该窗口所有 IPC/窗口事件全冻结,几乎没有正当使用理由;**无窗口可挂时**(启动失败、全部窗口已关)只能弹无主框,此时消息要自解释,因为没有窗口上下文。

# 2. Notification 系统通知

## 2.1 弹通知

```javascript
// 主进程
const { Notification } = require('electron')

function notify() {
  if (Notification.isSupported()) {
    new Notification({
      title: '任务完成',
      body: '文件已成功导出',
      silent: true,              // 静音(Windows/mac 都认)
      hasReply: true             // mac 可直接在通知里回复(回复走 reply 事件)
    }).show()
  }
}
```

★ **关键认知**：`Notification` 弹**系统通知**(标题+正文,出现在系统通知中心)。⚠ **在主进程创建**才稳定;渲染进程也能用 `new Notification()`,但 Windows 需要 app 已设置 AppUserModelId(主进程建更省心)。`isSupported()` 先判断支持情况。

## 2.2 通知的点击与平台差异

```javascript
const n = new Notification({ title: '更新', body: '新版本可用' })
n.on('click', () => mainWin.show())    // 点击通知 → 显示窗口
n.show()
```

★ **关键认知**：通知可监听 `click`(用户点了 → 聚焦窗口/跳转)。⚠ 平台差异多:macOS 通知样式/权限、Windows 的通知需应用有正确标识;**通知是"提醒"不是"打断"**,高频弹通知会惹用户反感或触发系统的关闭授权。

## 2.3 权限、AppUserModelId 与静音

```javascript
// Windows:通知按"应用身份"归类,这个 ID 不对 → 通知直接被系统丢弃
app.setAppUserModelId('com.example.myapp')   // 要与打包时 appId/快捷方式一致(第 13 篇)

// macOS:首次弹通知触发系统授权;被拒后 show() 静默失败
n.on('failed', (event, error) => console.log('通知失败:', error))

// 渲染进程 Web Notification API 的权限查询(网页那套同构)
const perm = await win.webContents.executeJavaScript('Notification.permission')
```

★ **关键认知**：Windows 通知按 **AppUserModelId(AUMID)** 归属到"某个应用"——开发态(`electron.exe`)与安装态(快捷方式里的 AUMID)不一致是"dev 能弹、打包后不弹"的头号原因;macOS 走**用户授权**,被拒后不报错只是不显示,要监听 `failed`/`close` 事件自查。⚠ **静音**:`silent: true` 只保证不发声,挡不住"横幅样式"被用户在系统设置里改成横幅/角标——通知的去留权限在用户手里,应用只能"请求 + 节制"。

## 2.4 Badge 角标与"任务完成"提示

```javascript
// macOS/Linux:Dock/任务栏角标数字
app.setBadgeCount(3)          // 3 条未读;0 清除
// Windows:任务栏角标走的是另一条路(overlay icon / 用户"专注助手"),setBadgeCount 在 Win 是 no-op
win.setOverlayIcon(shellImage, '3 个下载完成')   // Windows 任务栏图标上叠小图
```

★ **关键认知**：**角标能力三平台不对等**——`setBadgeCount` 在 macOS(和多数 Linux 桌面)有效,**Windows 上直接空操作**,要用 `setOverlayIcon` 或系统 toast 的"进度条"字段替代。⚠ 下载/长任务的提示层次:**进行中 → 任务栏进度(win.setProgressBar)/toast 进度条,完成 → 一条通知 + 角标**,别把"进行中"也做成通知刷屏。

# 3. 主/渲染分工

## 3.1 渲染进程请主进程弹框

```javascript
// preload 暴露
contextBridge.exposeInMainWorld('api', {
  openFile: () => ipcRenderer.invoke('dialog:open')
})

// 主进程
ipcMain.handle('dialog:open', async (event) => {
  const win = BrowserWindow.fromWebContents(event.sender)   // 找到发起者作为父窗口
  const r = await dialog.showOpenDialog(win, { properties: ['openFile'] })
  return r.canceled ? null : r.filePaths[0]
})

// 渲染进程
const path = await window.api.openFile()
```

★ **关键认知**：对话框/通知这类**系统能力都归主进程**,渲染进程通过 **preload + IPC** 间接调用。⚠ 这是全套"用例"的完整走通:preload 暴露 `openFile` → 渲染 invoke → 主进程弹框 → 返回路径,一环不少。别忘了 `BrowserWindow.fromWebContents(event.sender)`:多窗口应用里"弹在谁头上"应该由**发起者**决定,而不是硬弹主窗口。

# 4. 平台差异速览

| 项 | 说明 |
|----|------|
| 文件对话框 | 三平台都有,macOS 可用 sheet 停在窗口上 |
| 消息框 | 三平台都有,按钮顺序/文案习惯各异 |
| 通知 | macOS 款式统一;Windows 依赖 AppUserModelId;需用户授权 |
| 角标 | mac/Linux `setBadgeCount` 有效,Windows 需 overlay icon |

★ **关键认知**：对话框/通知的 API 是**跨平台统一的**,但**呈现细节有平台差异**(按钮顺序、通知样式、角标能力)。⚠ 别假设"按钮左是确定"这种固定习惯;关键文案写清楚(用"删除"而非"确定"),比依赖平台惯例更稳。

# 5. 本篇小结与下一步

★ **关键认知**：dialog(打开/保存/消息框)与 Notification(通知)都是"网页没有的桌面能力",归主进程,渲染经 IPC 调用。对话框第一参数传父窗口才有模态归属(`showMessageBoxSync` 阻塞主进程少用);Windows 通知认 AppUserModelId、mac 要授权、角标 mac/Linux 才有,Windows 用 overlayIcon;多窗口时用 `fromWebContents(event.sender)` 决定"弹在谁头上"。

下一步第 9 篇:系统集成——剪贴板、shell 打开外部、原生图像与开机自启。
