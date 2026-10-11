Electron 应用加载的是"网页",而网页可能是**远程内容或被注入的恶意脚本**——所以安全是 Electron 的重头戏。历史上大量 Electron 漏洞都源于"图省事开权限"。本篇讲清 Electron 的安全默认、沙箱、CSP、权限最小化,以及渲染进程危险等于什么。

# 1. 安全主线:渲染进程不可信

## 1.1 一切都源于"渲染进程可能被攻破"

```
渲染进程加载的可能不只有你的代码:
  远程内容、iframe、用户打开的文件、被注入的脚本
一旦渲染进程被攻破,它能做什么,取决于你给了它什么(Node 权限?IPC?)
```

★ **关键认知**：Electron 安全的第一原则是——**把渲染进程当"不可信"的**。⚠ 因为渲染进程可能加载**远程/第三方/恶意内容**,一旦被攻破,它拥有的权限就是攻击者的权限。所以**默认不给 Node、最小化 IPC 暴露、沙箱隔离**,是贯穿始终的红线。

## 1.2 两个致命默认(已改)

```javascript
// 历史上的"便利"默认,现在都是安全雷区,务必显式关闭:
webPreferences: {
  nodeIntegration: false,     // 渲染进程不掺 Node(默认已是 false)
  contextIsolation: true      // 上下文隔离(默认已是 true)
}
```

★ **关键认知**：`nodeIntegration: false`(渲染进程无 Node 权限)与 `contextIsolation: true`(preload 与页面隔离)是**安全基石**,现代 Electron 默认就是安全值。⚠ **切勿为了"方便在渲染进程里 require fs"而把它们改回危险值**——那是给自己埋雷;要系统能力,走 preload + IPC(第 4、5 篇)。

# 2. 沙箱 sandbox

## 2.1 渲染进程沙箱

```javascript
webPreferences: {
  sandbox: true    // 渲染进程跑的更严格的沙箱(进一步受限)
}
```

★ **关键认知**：`sandbox: true` 让渲染进程跑在**更严格的沙箱**里(更接近浏览器标签页的隔离)。⚠ 它进一步提升安全,但**会限制某些 Node 能力**(沙箱里 `require` 受限),与 preload 的配合方式也要注意;多数应用可开,但要验证 preload 是否兼容。

# 3. CSP 内容安全策略

## 3.1 限制能加载什么

```html
<!-- 渲染进程的 index.html 里加 CSP meta,限制脚本/资源来源 -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self'; script-src 'self'">
<!-- 含义:资源默认只能来自自身,脚本只能来自自身(禁内联/inline) -->
```

★ **关键认知**：**CSP(Content Security Policy)** 声明"页面能从哪加载脚本/资源"——`default-src 'self'` 即"只允许自身来源",**挡掉注入的内联脚本与外部恶意加载**。⚠ 这是防御 XSS 的核心:即便有脚本注入,没被 CSP 允许的来源也执行不了;Electron 官方强烈建议配 CSP。

## 3.2 为什么内联脚本要禁

★ **关键认知**：`script-src 'self'` 禁止**内联 script**(`<script>alert(1)</script>`)——因为 XSS 注入的往往是内联脚本。⚠ 代价是你自己也要把脚本抽到 `.js` 文件、避免内联事件处理器;这是"安全换一点点不便"的典型。

# 4. 权限最小化

## 4.1 只暴露必需

```javascript
// preload:只暴露"必需的最小 API"(第 4 篇)
contextBridge.exposeInMainWorld('api', {
  saveFile: content => ipcRenderer.invoke('file:save', content)  // 就这么一个
  // 而不是把 ipcRenderer 整个丢出去
})
```

★ **关键认知**：**权限最小化**贯彻到每层——preload 只暴露最小 API、主进程 IPC 校验参数与来源、不用不申请(如不自启就不申请自启权限)。⚠ 安全是"**每多给一分权限,就多一分风险**",所以默认最小、按需放开。

## 4.2 主进程的校验清单

```javascript
ipcMain.handle('dangerous', (event, arg) => {
  // 1. 验证来源:是不是我自己的文件页面发的
  if (!event.senderFrame.url.startsWith('file://')) throw new Error('bad source')
  // 2. 验证参数:类型、范围、白名单
  if (typeof arg !== 'string' || !allowedList.includes(arg)) throw new Error('bad arg')
  // 3. 再执行敏感操作
})
```

★ **关键认知**：主进程处理敏感 IPC 要**三道闸**——**验来源(senderFrame.url)、验参数(类型/白名单)、再执行**。⚠ 别假设"渲染进程发来的一定是可信的":渲染进程可能已被攻破,所以主进程永远要做最后的校验。

# 5. 远程内容与导航

## 5.1 禁止任意导航

```javascript
mainWin.webContents.on('will-navigate', (e, url) => {
  // 只允许自己应用内的导航,外部跳转让系统浏览器开(第 9 篇 shell)
  if (!url.startsWith(appUrlPrefix)) {
    e.preventDefault()
    shell.openExternal(url)   // 外部链接交给系统浏览器
  }
})
```

★ **关键认知**：拦截 `will-navigate`,**禁止应用窗口被导航到外部站点**(否则你的窗口就成了别人的网页容器)。⚠ 用户点的外链,正确做法是 `shell.openExternal`(第 9 篇)在系统浏览器开,而非在自己的窗口里 `loadURL`。

## 5.2 不加载不信任的远程内容

★ **关键认知**：尽量**只加载本地内容**(`loadFile`);必须加载远程时,要**充分信任该来源 + 收紧所有权限**。⚠ "渲染进程加载远程页面"是风险最高的形态(你无法控制那份代码),能避免就避免,不能避免就把安全配置拉到最严。

# 6. 安全清单与警告

| 项 | 安全做法 |
|----|---------|
| nodeIntegration | 保持 false |
| contextIsolation | 保持 true |
| sandbox | 尽量 true |
| CSP | 配 default-src 'self' |
| preload | 只暴露最小 API |
| 主进程 IPC | 验来源 + 验参数 |
| 外部链接 | shell.openExternal,禁窗口内导航 |

⚠ **一句话总结**:不要让渲染进程拥有任何它不绝对需要的权限,不要信任渲染进程发来的任何东西。

# 7. 本篇小结与下一步

★ **关键认知**：渲染进程不可信是安全主线;守住 nodeIntegration/contextIsolation/sandbox 三开关、配 CSP、preload 最小暴露、主进程三道校验、禁任意导航。安全是分层防御,每层最小权限。

下一步第 17 篇:多进程与 worker——utilityProcess、worker_threads、child_process,把重活挪出主进程。