网页没有"系统对话框"和"系统通知"——这是桌面应用才有的能力。Electron 的 `dialog` 提供打开/保存文件、消息框,`Notification` 弹出系统通知。本篇讲清它们的用法、与主/渲染进程的分工,以及平台差异。

# 1. dialog 系统对话框

## 1.1 打开 / 保存文件

```javascript
const { dialog } = require('electron')

// 打开文件:返回选中的文件路径
async function openFile() {
  const result = await dialog.showOpenDialog({
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
const save = await dialog.showSaveDialog({
  title: '保存到',
  defaultPath: 'untitled.txt'
})
```

★ **关键认知**：`showOpenDialog` / `showSaveDialog` 分别弹**打开/保存文件**对话框,返回 `{ canceled, filePaths }`。⚠ **`dialog` 只能在主进程用**——渲染进程要弹文件对话框,得走 IPC 请主进程调(这是"职责划分"的又一例)。`filters` 过滤文件类型,是体验的关键。

## 1.2 消息框

```javascript
const choice = await dialog.showMessageBox({
  type: 'warning',               // none/info/error/question/warning
  title: '确认',
  message: '确定要删除吗?',
  buttons: ['取消', '删除'],      // 按钮数组
  defaultId: 0,
  cancelId: 0
})
if (choice.response === 1) { /* 点了"删除" */ }
```

★ **关键认知**：`showMessageBox` 弹**消息/确认框**,`buttons` 定义按钮,`response` 返回点的是第几个。⚠ 用原生确认框 vs 页面内自绘弹窗的取舍:**系统对话框更"原生、一致"**,但**无法定制样式**;需要精美 UI 的用页面弹窗,需要简单可靠确认的用 `showMessageBox`。

# 2. Notification 系统通知

## 2.1 弹通知

```javascript
// 主进程
const { Notification } = require('electron')

function notify() {
  if (Notification.isSupported()) {
    new Notification({
      title: '任务完成',
      body: '文件已成功导出'
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

# 3. 主/渲染分工

## 3.1 渲染进程请主进程弹框

```javascript
// preload 暴露
contextBridge.exposeInMainWorld('api', {
  openFile: () => ipcRenderer.invoke('dialog:open')
})

// 主进程
ipcMain.handle('dialog:open', async () => {
  const r = await dialog.showOpenDialog({...})
  return r.canceled ? null : r.filePaths[0]
})

// 渲染进程
const path = await window.api.openFile()
```

★ **关键认知**：对话框/通知这类**系统能力都归主进程**,渲染进程通过 **preload + IPC** 间接调用。⚠ 这是全套"用例"的完整走通:preload 暴露 `openFile` → 渲染 invoke → 主进程弹框 → 返回路径,一环不少。

# 4. 平台差异速览

| 项 | 说明 |
|----|------|
| 文件对话框 | 三平台都有,macOS 可用 sheet 停在窗口上 |
| 消息框 | 三平台都有,按钮顺序/文案习惯各异 |
| 通知 | macOS 款式统一;Windows 依赖 AppUserModelId;需用户授权 |

★ **关键认知**：对话框/通知的 API 是**跨平台统一的**,但**呈现细节有平台差异**(按钮顺序、通知样式)。⚠ 别假设"按钮左是确定"这种固定习惯;关键文案写清楚(用"删除"而非"确定"),比依赖平台惯例更稳。

# 5. 本篇小结与下一步

★ **关键认知**：dialog(打开/保存/消息框)与 Notification(通知)都是"网页没有的桌面能力",归主进程,渲染经 IPC 调用。文件过滤、通知点击、平台差异是细节重点。

下一步第 9 篇:系统集成——剪贴板、shell 打开外部、原生图像与开机自启。