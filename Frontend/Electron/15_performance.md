Electron 最大的性能"黑点"是**体积大、内存高、启动慢**——它扛着一个 Chromium。因此 Electron 的性能优化,很大程度是"**驯服 Chromium 的开销**"。本篇讲清启动时序优化、进程级内存管理与泄漏定位、长列表与 webContents 复用、主进程事件循环阻塞,以及调试工具的使用。

# 1. 启动速度

## 1.1 冷启动的耗时点

```text
启动链路(双击到可见的时序):
  exe 启动 → 主进程 JS 初始化 → app.whenReady(Chromium 就绪)
    → new BrowserWindow → 渲染进程创建 → loadFile/loadURL
    → 前端资源加载/框架挂载 → 首屏可交互
每一段都能优化:JS 少做、窗口预建、资源本地化、首屏瘦身
```

★ **关键认知**：首次启动慢的三大来源——**Chromium 本身、主进程初始化、页面加载**。⚠ 优化思路:**主进程别在启动时做一堆重活**(延迟到需要时)、**窗口 ready-to-show 再 show**(避免白屏等待感)、**渲染进程首屏轻量化**。优化前先测量:`app.on('ready')` 前后打点 + 首屏时间上报,大头往往在顶层 require 与远程资源。

## 1.2 延迟初始化

```javascript
// ✗ 启动时:同步读大配置、注册一堆东西、加载大模块
// ✓ 启动时只做"显示窗口必需"的,其余按需(delayed require / 懒初始化)

let heavyModule
function useHeavyModule() {
  if (!heavyModule) heavyModule = require('./heavy')   // 用到才加载
  return heavyModule
}

// 非关键路径挪到首帧之后:
win.once('ready-to-show', () => {
  win.show()
  setImmediate(() => { initTray(); checkForUpdates(); registerGlobalShortcuts() })  // 让首帧先落地
})
```

★ **关键认知**：主进程**按需加载**重量模块、**延迟初始化**非关键功能(自启、检查更新放后台)。⚠ "启动只做显示窗口这件事",是桌面应用启动优化的第一原则;把可省的初始化统统往后挪。渲染侧同理:**本地资源优先**(生产 `loadFile`,dev 才连 dev server),网络型首屏用 splash 窗口过渡。

## 1.3 窗口预创建(热窗口)

```javascript
// 高频"新建窗口"场景(点搜索结果开新窗):预先藏一个热窗口
let hotWin
function warmWindow() {
  hotWin = new BrowserWindow({ show: false, webPreferences: { preload: ... } })
  hotWin.loadFile('editor.html')               // 提前备好渲染进程和页面
}
function openEditor(data) {
  if (!hotWin) return createNormal(data)
  hotWin.webContents.send('load-data', data)   // 复用,只发数据
  hotWin.show()
  setTimeout(warmWindow, 1000)                 // 补一个热备
}
```

★ **关键认知**：热窗口把"新建 BrowserWindow + 渲染进程启动 + 页面加载"的数百 ms 开销**挪到用户看不见的空闲期**,点击开新窗从"等半秒"变"秒开"。⚠ 它是**空间换时间**(常驻一个渲染进程的内存),只给高频入口用;send 要等页面 `did-finish-load`,否则消息丢在空页面里。

# 2. 内存占用

## 2.1 Electron 为什么吃内存

```text
Chromium 多进程:每个窗口一个渲染进程,每个都有基准内存开销
主进程 + GPU 进程 + 各渲染进程 + (可能的)工具进程
→ 天然比原生应用吃内存(总 RSS 三五百 MB 是常态)
```

★ **关键认知**：Electron 内存高是**架构代价**——多进程隔离(每个渲染进程独立)带来安全,也带来内存开销。⚠ 所以内存优化集中在"**减少进程数、让每个进程少占**",而不是妄想追平原生。任务管理器里要**按进程归类看**:"总内存 1GB"里属于你代码的,是单个渲染进程的 JS 堆,不是总 RSS。

## 2.2 减少内存的实践

```javascript
// 窗口不用时销毁,而非隐藏(隐藏仍占内存)
win.close()  // 而非 win.hide()

// 关闭后台的"标签页崩溃/无用的子窗口"
// 懒加载大组件,避免一次全挂载
```

★ **关键认知**：**不复用/不显示的东西就"关闭"而非"隐藏"**(隐藏仍占内存);卸载大对象引用;渲染进程内懒加载大组件。⚠ "隐藏省内存"是误区——隐藏的窗口渲染进程还活着;真省内存要 destroy。工程折中:**高频开关用 hide(秒开)、低频用 close(省内存)**,别一刀切。

## 2.3 泄漏定位

```text
定位套路(基线 → 操作 N 次 → 对比):
1. 渲染进程: DevTools → Memory → Heap snapshot 拍两次,
   中间做 N 次可疑操作,过滤"新增且未释放"的对象
2. 主进程: 定时上报 process.memoryUsage() 的 RSS/heapUsed,看长期趋势
3. 反复开合某功能后内存台阶上涨且不回落 → 泄漏实锤
常见元凶: ipcRenderer.on 重复注册没 off、全局监听器、
没清的 setInterval、闭包持大对象、已销毁 webContents 的引用
```

★ **关键认知**：Electron 特有的泄漏大头是**监听器与窗口引用**——`ipcRenderer.on` 在组件每次 mount 注册、窗口销毁后引用链还在。⚠ 所以 preload 暴露 API 时应**返回注销函数**,渲染侧写进 useEffect cleanup:

```javascript
// preload: 把"注销"一起给出去,泄漏责任交给调用方闭环
onUpdate: (cb) => {
  const listener = (_e, data) => cb(data)
  ipcRenderer.on('update', listener)
  return () => ipcRenderer.removeListener('update', listener)
}
```

# 3. 多窗口的代价

## 3.1 每个窗口都是开销

★ **关键认知**：**每个 BrowserWindow 都是一个完整渲染进程**(有独立内存、独立 JS 环境)。⚠ 因此"开一堆窗口"是 Electron 内存失控的头号原因;设计上**能用一个窗口内做多视图(路由/多标签页),就别开多个 BrowserWindow**。

## 3.2 进程共享的边界

★ **关键认知**：想让多个窗口**共享资源/内存**,现实是它们仍是独立进程,只能靠 IPC 传数据,不能共享 JS 对象。⚠ 这是为什么"多窗口"要在设计阶段就慎重;若只是"多个页面",优先**单窗口 + 前端路由**,而非多 BrowserWindow。

## 3.3 长列表与 webContents 复用

```javascript
// 长列表三档:分页/无限滚动 → 虚拟列表(只渲染视口内节点)→ 独立子窗口隔离
win.webContents.send('switch-view', { list: chunk })   // 数据分片推,而非一次全量
```

★ **关键认知**：列表卡的隐形坑在 **IPC 传大块**——invoke/send 走结构化克隆,一次回传几十 MB 数组会让两端都明显卡;正确姿势是**分片流式推送**,或让渲染进程直接查本地数据源,主进程不当"数据搬运工"。⚠ 判断归属先看 DevTools Performance 录一段:Layout/Scripting 长任务 → 前端渲染问题;消息间隙过大 → IPC/主进程问题,药方不同。

# 4. GPU 与渲染

## 4.1 硬件加速

```javascript
// 某些场景(GPU 兼容性问题、虚拟机、无头环境)可关闭硬件加速
app.disableHardwareAcceleration()
```

★ **关键认知**：默认开启**硬件加速(GPU 渲染)**,通常更流畅;⚠ 但在**虚拟机、远程桌面、某些老旧显卡**下 GPU 反而出问题(花屏、崩溃),此时 `disableHardwareAcceleration()` 关掉能救急。这是"白屏/花屏"排查的第一开关;`chrome://gpu` 能看当前加速状态。

## 4.2 渲染进程内的性能

★ **关键认知**：渲染进程内,前端那套优化(懒加载、虚拟列表、减少重渲染)完全适用——它是 Chromium。⚠ 别把桌面应用的渲染进程当"无限性能",大列表、大动画该优化照样优化;前端优化的所有技巧在这里都有效。调试面也是同款:**渲染进程直接享受 DevTools 全家桶**(Performance 录长任务/布局抖动、Memory 查堆快照);**主进程**要 `electron --inspect=5858 .` 再 chrome://inspect 挂 Node 侧火焰图——DevTools 里找不到主进程的账,别录错了地方。

## 4.3 主进程事件循环阻塞

```javascript
// ✗ 主进程同步大 IO/大计算:所有窗口的 IPC、菜单、窗口动画一起僵住
const big = fs.readFileSync(path)
// ✓ 异步 + 把重活挪出去
const big = await fs.promises.readFile(path)
// 更重的 CPU 活:utilityProcess(第 17 篇) / worker_threads
```

★ **关键认知**：主进程是**单事件循环**,一次同步大 IO 是**全局卡顿**——浏览器里主线程只卡一个页,Electron 里卡所有窗口,这是它独有的卡顿形态。⚠ 红线:主进程不出现大文件 `readFileSync`、循环里大 JSON `parse`;CPU 密集走 `utilityProcess`(Electron 22+ 的后台进程形态)或 worker_threads。

# 5. 性能清单

| 手段 | 针对 |
|------|------|
| 延迟初始化/按需 require | 启动慢 |
| ready-to-show 再显示 | 白屏等待感 |
| 热窗口预创建 | 开窗慢 |
| 关闭而非隐藏窗口 | 内存高 |
| 注销 IPC 监听/快照对比 | 内存泄漏 |
| 单窗口 + 路由代替多窗口 | 多进程开销 |
| 虚拟列表 + IPC 分片 | 长列表卡 |
| 主进程禁同步大 IO | 全局卡顿 |
| disableHardwareAcceleration | GPU 兼容问题 |
| 前端懒加载/虚拟列表 | 渲染进程性能 |

★ **关键认知**：Electron 性能优化 = **"少加载(延迟)、少进程(单窗口)、少占(关闭勿隐藏)、少白屏(ready-to-show)、少阻塞(主进程异步)"** + 前端优化照搬。它不像原生那样资源无限,别把 Chromium 当纯原生用。

# 6. 本篇小结与下一步

★ **关键认知**：Electron 性能重点在"驯服 Chromium"——启动靠延迟初始化与热窗口预创建、内存靠关闭而非隐藏+监听器注销防泄漏、多窗口要克制(优先单窗口+路由)、长列表虚拟滚动且 IPC 别传大块、主进程绝不跑同步大活、GPU 问题可关硬件加速。调试面就是 Chromium 那套:渲染进程 DevTools、主进程 --inspect。前端优化技巧在渲染进程照常有效。

下一步进入**阶段四 · 安全与进阶**,第 16 篇:安全最佳实践——contextIsolation、沙箱、CSP、权限最小化。
