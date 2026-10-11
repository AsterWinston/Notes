现实里的 Electron 应用,界面几乎都用 Vue/React 写——于是真正的问题是"**前端框架怎么与主进程优雅协作**"。本篇讲清框架与 Electron 的分层、通信的封装模式、以及让 `window.api` 在 TS 下类型安全的最佳实践。

# 1. 分层视图

```
┌─────────────────────────────┐
│      渲染进程(前端框架)       │  Vue/React:界面、组件、UI 状态
│  window.api.xxx 调用 IPC     │
└──────────────┬──────────────┘
               │ IPC
┌──────────────▼──────────────┐
│        主进程(业务 + 系统)    │  文件、DB、系统 API、窗口管理
└─────────────────────────────┘
```

★ **关键认知**：框架与 Electron 的**分层**——**界面与 UI 状态归框架(渲染进程),系统能力与持久化归主进程**,中间只靠 `window.api` 这条窄接口通信。⚠ 关键纪律:**别把文件/DB 操作写进 Vue/React 组件里**(渲染进程没那权限且不安全),组件只调 `window.api`,具体实现全在主进程。

# 2. 通信封装

## 2.1 把 IPC 封装成语义方法

```javascript
// preload 暴露一批"语义清晰"的方法,而非裸 IPC
contextBridge.exposeInMainWorld('api', {
  // 文件
  openFile: () => ipcRenderer.invoke('file:open'),
  saveFile: (path, content) => ipcRenderer.invoke('file:save', path, content),
  // 窗口
  minimize: () => ipcRenderer.send('win:minimize'),
  // 订阅主进程推送
  onProgress: (cb) => ipcRenderer.on('progress', (_e, v) => cb(v))
})
```

★ **关键认知**：把 IPC **封装成语义方法**(`openFile`/`saveFile`),组件里用起来像一个"桌面能力 SDK"。⚠ 别在组件里到处写裸 `ipcRenderer.invoke('xxx')`——散落的通道名难维护;统一在 preload 收口,组件只面对 `window.api`。

## 2.2 组件里的使用

```vue
<script setup lang="ts">
import { ref } from 'vue'

const content = ref('')
async function save() {
  await window.api.saveFile('note.txt', content.value)   // 像调普通异步函数
}

window.api.onProgress(p => console.log(p))   // 订阅推送
</script>
```

★ **关键认知**：组件里调 `window.api.xxx` 就像调**普通异步函数/订阅**,完全不关心底层是 IPC。⚠ 这是好分层的回报:**前端开发者只用关心"有什么能力",不用关心"它是主进程还是渲染进程"**。

# 3. TypeScript 类型安全

## 3.1 为 api 写类型

```typescript
// 一份类型声明(如 src/preload/index.d.ts 或共享类型文件)
export interface DesktopAPI {
  openFile(): Promise<string | null>
  saveFile(path: string, content: string): Promise<void>
  minimize(): void
  onProgress(cb: (value: number) => void): void
}

declare global {
  interface Window {
    api: DesktopAPI
  }
}
```

★ **关键认知**：为 `window.api` 写**接口类型**,组件里就获得完整补全与类型检查——这是 Electron + TS 的关键一步。⚠ 主进程/preload/渲染进程**共享这份类型**,三端接口一致,改了 preload 的 API,渲染进程立刻编译器报错提醒。

## 3.2 共享类型的工程组织

```
src/
├── main/         # 主进程(实现 API 的主进程侧)
├── preload/      # preload(桥接,类型与实现对齐)
├── renderer/     # 渲染进程(消费 window.api)
└── shared/       # 共享:IPC 通道名常量、接口类型
```

★ **关键认知**：把**IPC 通道名、接口类型**放 `shared/` 目录三端共享——避免"主进程通道名手写 'file:open'、渲染进程又手写一次 'file:save'"的漂移。⚠ 这是 Electron 大项目的工程化重点:单一事实来源(通道名常量 + 类型),三端复用。

# 4. 最佳实践

## 4.1 通道命名规范

```javascript
// 用"领域:动作"命名通道,清晰分组
'file:open' / 'file:save'
'dialog:openFile'
'win:minimize' / 'win:maximize'
'settings:get' / 'settings:set'
```

★ **关键认知**：通道名用 **`领域:动作`** 规范(`file:open`),像 REST 的 URL 一样清晰。⚠ 乱命名('foo'/'bar'/'a1')会让工程不可读;统一的命名约定,配合 shared 常量,是"可维护 IPC"的两根支柱。

## 4.2 主进程侧的实现组织

```javascript
// main/ipc.js —— 把 IPC 处理函数集中到一个文件,而非散落各处
ipcMain.handle('file:open', handleOpenFile)
ipcMain.handle('file:save', handleSaveFile)
// 每个 handler 独立函数,便于测试
```

★ **关键认知**：主进程的 IPC 处理**集中在一个模块**,每个 handler 是独立可测的函数。⚠ "preload 收口渲染侧、ipc.js 收口主进程侧",两边夹着一条干净的 IPC 边界,这是大型 Electron 应用可维护的基石。

# 5. 本篇小结与下一步

★ **关键认知**：框架管 UI、主进程管系统,只靠 window.api 通信;preload 把 IPC 封装成语义方法;TS 下为 api 写共享类型(shared 目录 + 命名规范);主进程 IPC 处理集中在一个模块。核心是"分层清晰 + 类型打通 + 命名统一"。

下一步第 12 篇:状态与数据流——跨进程状态协同,主进程持有数据如何同步到渲染。