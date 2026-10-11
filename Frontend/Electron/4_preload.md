渲染进程是"网页",不该直接拿到 Node 权限;但它又需要调用系统能力。**预加载脚本(preload)** 就是这个"安全桥"——跑在主进程与渲染进程之间,用 `contextBridge` 把精选的 API 暴露给页面。本篇讲清这个安全设计的核心。

# 1. preload 的定位

## 1.1 它是隔离的中间层

```
主进程(系统能力) ←→ preload(预加载,有访问权) ←→ 渲染进程(网页,受限)
                          │
                用 contextBridge 暴露"白名单 API"
```

★ **关键认知**：preload 脚本**在主进程与渲染进程之间**——它跑在隔离环境里,**能碰主进程的能力,又能把"挑选过的 API"暴露给渲染进程**。⚠ 核心思想是"**白名单**":渲染进程不能"什么都拿",只能拿 preload 里**显式暴露**的那几个方法,从根上杜绝"网页拿到整个 Node"。

## 1.2 它被注入的时机

```javascript
new BrowserWindow({
  webPreferences: {
    preload: join(__dirname, '../preload/index.js')  // 页面加载前,preload 先执行
  }
})
```

★ **关键认知**：preload 在**页面加载前**执行,所以它暴露的 API 页面一起步就能用。⚠ 它运行在**独立的 JS 上下文**(与页面隔开),必须借助 `contextBridge` 才能把 API"桥"给页面——直接挂 `window` 是不行的(见下)。

# 2. contextBridge 与 contextIsolation

## 2.1 contextIsolation 打开了什么

```
contextIsolation: true(默认)
  → preload 和页面是"两个隔离的世界"
  → preload 里的变量页面看不到,反之亦然
  → 必须用 contextBridge.exposeInMainWorld 显式桥接
```

★ **关键认知**：`contextIsolation: true` 把 preload 与页面的 JS 上下文**隔离**——互不可见。⚠ 好处是**大幅提升安全**(页面脚本无法篡改 preload 里的东西);代价是**不能直接往 window 上挂变量**,必须走 `contextBridge`。

## 2.2 contextBridge 暴露 API

```javascript
// preload/index.js
const { contextBridge, ipcRenderer } = require('electron')

// 暴露一个"精选的、命名空间下的 API"给渲染进程 window.api
contextBridge.exposeInMainWorld('api', {
  // 只暴露具体方法,而不是暴露整个 ipcRenderer
  saveFile: (content) => ipcRenderer.invoke('save-file', content),
  openDialog: () => ipcRenderer.invoke('open-dialog'),
  onUpdate: (callback) => ipcRenderer.on('update', (_, data) => callback(data))
})
```

★ **关键认知**：`contextBridge.exposeInMainWorld('api', {...})` 把对象挂到渲染进程的 `window.api` 上——**只暴露你挑好的方法**,不暴露整个 `ipcRenderer`。⚠ 这是安全的最佳实践:**暴露"具体功能"而非"底层通道"**;把 `ipcRenderer` 整个丢给页面,等于把安全门焊死了。

# 3. 渲染进程怎么用

## 3.1 页面里调用

```javascript
// 渲染进程(页面)里:
const result = await window.api.saveFile('内容')   // 调 preload 暴露的方法
window.api.onUpdate((data) => { /* 收到主进程推送 */ })
```

★ **关键认知**：渲染进程通过 `window.api.xxx` 调用 preload 暴露的方法,像调一个普通库一样。⚠ 这里**看不到 ipcRenderer、看不到 Node**——页面眼里只有那个干净的 `api` 对象,这就是"安全桥"落地的样子。

## 3.2 类型支持(TS)

```typescript
// 可写一份类型声明,让渲染进程拿到 api 的完整类型
declare global {
  interface Window {
    api: {
      saveFile(content: string): Promise<string>
      openDialog(): Promise<string | undefined>
    }
  }
}
```

★ **关键认知**：TS 项目里为 `window.api` 写**类型声明**,渲染进程就能获得完整补全与类型检查。⚠ 这是 Electron + TS 的配套动作:preload 是主/渲染的边界,把类型在这里也"接上",类型安全才不断链。

# 4. 安全最佳实践

## 4.1 暴露"最小必要集"

```javascript
// ✓ 好:只暴露"保存"这个具体操作
contextBridge.exposeInMainWorld('api', {
  saveFile: content => ipcRenderer.invoke('save-file', content)
})

// ✗ 坏:把 ipcRenderer 整个暴露,页面可发任意 IPC
contextBridge.exposeInMainWorld('ipc', { ipcRenderer })
```

★ **关键认知**：暴露 API 的黄金原则是"**最小必要集**"——页面要什么功能,就暴露什么方法;绝不暴露 `ipcRenderer` 本体或 Node 能力。⚠ 一旦暴露 `ipcRenderer`,你的白名单形同虚设,页面(或注入的恶意脚本)能发任何消息、触发任何主进程操作。

## 4.2 主进程也要校验

```javascript
// 主进程处理 IPC 时,别无条件信任渲染进程传来的参数
ipcMain.handle('save-file', (event, content) => {
  if (typeof content !== 'string') throw new Error('参数非法')   // 校验
  // 再执行写文件
})
```

★ **关键认知**：即便 preload 做了白名单,**主进程收到 IPC 仍要校验参数**(渲染进程可能被攻破、内容是恶意的)。⚠ 安全是分层的:preload 白名单是"第一道",主进程校验是"最后一道",两层都做才稳(呼应第 16 篇)。

# 5. 本篇小结与下一步

★ **关键认知**：preload 是隔离的"安全桥";contextIsolation 让 preload 与页面隔离,contextBridge 暴露"最小必要"的 API 到 window.api;绝不暴露 ipcRenderer 本体;主进程仍需校验参数。

下一步第 5 篇:IPC 通信基础——ipcMain/ipcRenderer,主进程与渲染进程怎么对话。