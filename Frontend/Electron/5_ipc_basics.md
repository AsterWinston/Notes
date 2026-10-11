主进程与渲染进程隔离运行,协作全靠 **IPC(进程间通信)**。Electron 的 IPC 有两条 API 线:`ipcMain`(主进程侧)与 `ipcRenderer`(渲染进程侧)。本篇讲清四种通信方式——单向、双向(invoke/handle)、发送、以及它们的取舍与安全。

# 1. IPC 的两种角色

## 1.1 ipcMain 与 ipcRenderer

```
ipcRenderer(渲染进程)  ────消息────▶  ipcMain(主进程)
ipcMain(主进程)        ────推送────▶  ipcRenderer(渲染进程)
```

★ **关键认知**：IPC 的两端——**`ipcRenderer`(渲染进程用,发起请求/收推送)与 `ipcMain`(主进程用,监听请求/主动推)**。⚠ 对称是理解 IPC 的钥匙:渲染发、主收;主推、渲染收;一个通道名两端各挂处理。

# 2. 单向发送 send/on

## 2.1 渲染 → 主(发送通知)

```javascript
// 渲染进程
ipcRenderer.send('app-ready', '数据')

// 主进程
ipcMain.on('app-ready', (event, data) => {
  console.log('收到:', data)   // 单向:不回复
})
```

★ **关键认知**：`send`/`on` 是**单向**——渲染进程发个通知,主进程处理,**不把结果回传**。⚠ 适合"告知一下"(如"窗口已加载完"),不适合"要结果"的场景(那用 invoke,见下)。

## 2.2 主 → 渲染(主动推送)

```javascript
// 主进程:主动推消息给某个窗口
win.webContents.send('update', { version: '1.1' })

// 渲染进程:监听
ipcRenderer.on('update', (event, data) => {
  console.log('收到主进程推送', data)
})
```

★ **关键认知**：主进程用 `webContents.send` 向**指定窗口**推送,渲染进程 `ipcRenderer.on` 收。⚠ 这是"主推渲染"的范式(如下载进度、全局事件);注意主进程推的是"**某个窗口**"(拿到 win 的 webContents),这是句柄的关键用途。

# 3. 双向 invoke/handle

## 3.1 请求 + 响应

```javascript
// 渲染进程:发起请求,拿 Promise 结果
const result = await ipcRenderer.invoke('read-file', '/path/to/file')

// 主进程:handle 处理并返回值(可 async)
ipcMain.handle('read-file', async (event, filePath) => {
  const content = await fs.promises.readFile(filePath, 'utf-8')
  return content       // 返回值会自动传给渲染进程的 await
})
```

★ **关键认知**：`invoke`/`handle` 是**双向的请求-响应**——渲染进程 `invoke` 发请求拿 `Promise`,主进程 `handle` 处理并 `return` 结果。⚠ 这是**最常用**的 IPC 方式(要"结果"的都用它);它的本质等价于"`send` + 一次回调",但 API 更干净(直接 await)。

## 3.2 handle 的要点

```javascript
// handle 只注册一次;每个通道一个处理函数;可 async
ipcMain.handle('get-user', async (event, id) => {
  const user = await db.getUser(id)
  return user                 // 可以返回任意可序列化的值(对象/数组/字符串)
})
```

★ **关键认知**：`handle` 一个通道**只注册一次**,可 `async`(直接 await 异步操作),返回值**需可序列化**(对象/数组/基本类型)。⚠ 返回值会跨进程序列化(结构化克隆),**不能返回函数/非可序列化对象**(如 DOM、含方法的对象)——这是 IPC 的硬边界。

# 4. 事件对象与安全

## 4.1 event / event.sender

```javascript
ipcMain.on('some-message', (event, data) => {
  const win = event.sender          // 发来消息的那个渲染进程/窗口
  win.send('reply', '...')          // 通过它回复
})
```

★ **关键认知**：IPC 处理函数的第一个参数 **`event`** 携带来来处——`event.sender` 是**发起请求的窗口**。⚠ 通过 `event.sender` 能精确回复"发起者",避免"回复到了哪个窗口都不确定"的混乱;也可借它取 `item` 类的发送者信息。

## 4.2 senderFrame 与校验

```javascript
ipcMain.handle('dangerous-op', (event, param) => {
  if (!event.senderFrame.url.startsWith('file://')) {
    throw new Error('不受信任的来源')   // 校验发送者(第 4、16 篇的安全)
  }
  // 再执行危险操作
})
```

★ **关键认知**：`event.senderFrame.url` 是**发送者的来源地址**,可用于校验"发消息的是不是可信页面"。⚠ 这是 IPC 的安全防线之一:主进程处理敏感操作前,**先验来源 + 验参数**(呼应第 4、16 篇)。

# 5. 四种方式速查

| 方式 | API | 方向 | 有返回值 |
|------|-----|------|---------|
| 单向通知 | send / on | 渲染 → 主 | 否 |
| 请求响应 | invoke / handle | 渲染 → 主 | 是 |
| 主动推送 | webContents.send / on | 主 → 渲染 | 否 |
| 双向旧式 | send + 回调(已少用) | — | 是 |

★ **关键认知**：主流的两种——**`invoke/handle`(要结果)与 `send/on`(告知/推送)**。⚠ 老代码里可能见到 `ipcRenderer.sendSync`(同步 IPC)——**尽量避免**:它会阻塞渲染进程,拖慢 UI,官方也不推荐(要同步就优先 invoke 后 await 的逻辑组织)。

# 6. 本篇小结与下一步

★ **关键认知**：IPC 两端 ipcRenderer/ipcMain 对称;invoke/handle 拿结果(最常用),send/on 做通知,webContents.send 主推渲染;event.sender 是精确回复/校验的来源。返回值须可序列化。

下一步进入**阶段二 · 窗口与原生能力**,第 6 篇:窗口管理——多窗口、无边框、系统托盘。