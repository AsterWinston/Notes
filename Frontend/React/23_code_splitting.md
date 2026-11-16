单页应用越写越大,首屏要加载的 JS 越多越慢。**代码分割(Code Splitting)+ 懒加载** 把"一次性加载全部"变成"按需加载"。本篇讲清千里之外的 `React.lazy`、`Suspense`、动态 `import()`,以及路由级懒加载的最佳实践。

# 1. 为什么要分割

## 1.1 bundle 太大的问题

```text
不分割:整个应用打成一个巨大的 main.js,用户首屏全要下载
分割:按路由/组件切成多个 chunk,首屏只加载"当前页面需要的那部分"

  main.js 4.2MB(全下)                    main.js 280KB ──▶ 首屏
  ├─ 框架+全部 20 个页面                 ├─ chart-900KB ──▶ 进报表页才下
  ├─ 图表/编辑器/日期库…                 ├─ admin-600KB ─▶ 进后台才下
  └─ 首屏真正用得到的不到 1/5             └─ …各自成块
```

★ **关键认知**：SPA 打包后若只有一个大 bundle,**首屏要下载全部代码**才渲染,体感极慢。⚠ 代码分割把 bundle 拆成小块,**访问到哪个功能才下载哪个**,首屏体积大幅下降——这是 SPA 性能优化的第一优先事项。

## 1.2 import() 的分包原理:打包器眼中的"切割信号"

```js
// 静态 import:打包器可完整摇树,合并进同一张依赖图 → 同一个 chunk
import { Chart } from './chart'

// 动态 import():返回 Promise,依赖"运行时才确定"——打包器只能把它切成独立 chunk
const mod = await import('./chart')       // → 产物多出 chart-c3d4.js

// ⚠ 完全动态的路径会切断整棵目录(打包器不知道要留谁):
import(`./pages/${x}`)                    // ✗ 不可用/整目录进一个懒 chunk
import('./pages/' + x + '.js')            // ✓ 带目录前缀的"模板",只切该目录
```

```text
构建产物与运行时:
  dist/assets/main-a1b2.js    ← 静态图:入口 + 所有静态 import 链
  dist/assets/chart-c3d4.js   ← 每个动态 import() 各自一块(文件名带内容哈希)
  运行时:加载执行 main.js → 走到 import('./chart') 才请求 chart-c3d4.js
          → Promise resolve → 组件才可渲染 → Suspense 收起 fallback
```

```ts
// vite.config.ts:框架单独成包,业务更新不击穿缓存(webpack 用 splitChunks)
export default defineConfig({
  build: { rollupOptions: { output: { manualChunks: { vendor: ['react', 'react-dom'] } } } }
})
```

★ **关键认知**：分包不是运行时技巧,而是**构建期的依赖图切割**——`import()` 是 webpack/Rollup/Vite 共同认账的切割信号:静态依赖合并、动态依赖切开,产物各自带内容哈希。⚠ 两个高频推论:(a) 只要被某个**静态 import 链**摸到,组件照样进主包——"明明 lazy 了却没切出去",多半是别处还有一行静态 import;(b) `import()` 参数含变量时必须给出目录前缀模式,否则构建警告甚至失败。

# 2. React.lazy 与 Suspense

## 2.1 基本用法

```jsx
import { lazy, Suspense } from 'react'

// lazy:动态 import 一个组件,返回"懒加载组件"
const HeavyChart = lazy(() => import('./HeavyChart'))

function Dashboard() {
  return (
    <Suspense fallback={<div>图表加载中…</div>}>
      <HeavyChart />     {/* 渲染到这里才下载并挂载 */}
    </Suspense>
  )
}
```

★ **关键认知**：`lazy(() => import(...))` 定义懒加载组件,`<Suspense fallback={...}>` 兜住"加载中"状态。⚠ `import()` 是**动态导入**,打包工具据此把该组件**切进独立 chunk**;`fallback` 是加载时显示的占位。

```jsx
// lazy 的三条硬规矩:
const X = lazy(() => import('./X'))                          // ① 只认 default 导出
const Y = lazy(() => import('./mod').then(m => ({ default: m.Named })))  // 具名要手工桥
// ② 必须被 Suspense 包裹;③ 必须定义在组件外——组件体内 lazy 每次渲染造新 type,
//    子树反复卸载重建(与第 16 篇 HOC 包错位置是同一个坑)

// ④ chunk 加载失败(发版后旧页面还在跑,旧哈希文件已删):lazy 会"记住失败"不重试,
//    要靠外层 ErrorBoundary 提示刷新,或包一层失败重导的工具函数
```

★ **关键认知**：lazy 内部就是"**把 import 的 Promise 挂到渲染里**"——chunk 未到就 throw promise 让 Suspense 接住,到达后取 default 并缓存,同一 lazy 组件只下载一次。⚠ 第 3 条"定义位置"最阴:它会引发"打字掉焦点""图表反复重建"一类怪病,code review 盯紧 `lazy(` 出现的行号。

## 2.2 Suspense 的作用

★ **关键认知**：`Suspense` 是"**异步组件的边界**"——在懒加载组件**未就绪时渲染 fallback**。⚠ 没有 Suspense 包裹,lazy 组件加载时会直接抛错(React 不知道显示什么);Suspense 也可**嵌套多层**,做粒度不同的加载态。

# 3. 路由级懒加载

## 3.1 最常见的分割点

```jsx
import { lazy, Suspense } from 'react'
import { createBrowserRouter } from 'react-router-dom'

// 每个路由组件都懒加载:不同页面切不同 chunk
const Home = lazy(() => import('./pages/Home'))
const About = lazy(() => import('./pages/About'))
const Dashboard = lazy(() => import('./pages/Dashboard'))

const router = createBrowserRouter([
  { path: '/', element: <Suspense fallback={...}><Home /></Suspense> },
  { path: '/about', element: <Suspense fallback={...}><About /></Suspense> }
])
```

★ **关键认知**：**路由是代码分割最自然的切割点**——"访问哪个页面才加载哪个页面的代码",每个页面一个 chunk。⚠ 这是与第 17 篇路由懒加载的合流:首屏只带 Home,About/Dashboard 都在各自 chunk 里,访问到时才下载。

## 3.2 route.lazy:连路由模块本身也拆出去

```tsx
// React Router 6.4+ 的 data router:路由"定义"(loader+组件+配置)整模块按需加载
const router = createBrowserRouter([
  {
    path: '/dashboard',
    lazy: () => import('./routes/dashboard'),   // 模块导出 RouteObject 的一部分
  },
])

// ./routes/dashboard.tsx
export default {
  loader: dashboardLoader,        // 数据请求函数也随之延迟进 chunk
  Component: DashboardPage,
} satisfies Partial<RouteObject>
```

★ **关键认知**：`element={<Lazy/>}` 与 `route.lazy` 是**两层不同的分割**:前者只拆"组件代码",loader 等仍在主包;后者拆"整个路由模块",大型应用首屏连路由配置都不带。⚠ `lazy` 与顶层 `Component/loader/action` 是**互斥**写法,混了 Router 会直接抛错;迁移一个路由时二选一。

# 4. 分割的粒度与收益

## 4.1 组件级 vs 路由级

| 粒度 | 例子 | 收益 |
|------|------|------|
| 路由级 | 每个页面懒加载 | 首屏只含当前页(收益最大) |
| 组件级 | 图表/地图/富文本编辑器 | 首屏不计入首屏用不到的重组件 |
| 库级 | 大型第三方库随之分包 | 减少主包 |

★ **关键认知**：**路由级分割收益最大**(首屏直接砍掉其他页面);**组件级**针对"某页面里的大块头"(图表、编辑器)进一步拆。⚠ 切割粒度要权衡:过细会增请求数,通常"**路由级必做,组件级按需**"。

## 4.2 首屏指纹

```text
首屏 chunk:
  主 bundle(框架 + 路由 + 首页代码)
  核心第三方库(React 本身、首页依赖)

非首屏(懒):About.js、Dashboard.js、图表库…
```

★ **关键认知**：分割后首屏只含"**框架 + 当前页 + 首屏依赖**",其余都是懒加载 chunk。⚠ 进一步可用打包分析(如 vite 的 rollup-plugin-visualizer)看清 chunk 分布,针对"误打入首屏的大库"做优化。

# 5. 与懒加载协同的优化

## 5.1 preload 下一步

```jsx
// 用户悬停导航时,提前 preload 下一个页面的 chunk(进站更丝滑)
// 部分路由库支持 <Link prefetch> 或手动 import() 预热

// 策略矩阵:什么时候"提前"拿 chunk
<Link to="/report" onMouseEnter={() => import('./pages/Report')}>   {/* ① 悬停,命中率最高 */}

useEffect(() => {                                                   /* ② 空闲预取高频路由 */
  const run = () => Promise.allSettled([import('./pages/Report'), import('./pages/Settings')])
  'requestIdleCallback' in window ? requestIdleCallback(run) : setTimeout(run, 2000)
}, [])
// ③ 视口预取:列表项进入视口即预热其详情页;④ 点击预测:pointerdown 比 click 早几十 ms
```

★ **关键认知**：懒加载会带来"点击后要等一下才下载",可**预加载**(prefetch)下一步——悬停/进入视口时提前下好 chunk。⚠ 这是"懒加载 + 预加载"的互补:懒加载省首屏,预加载补交互流畅。

★ 追加一句机制:`import()` 是**幂等**的——预取过的 chunk 真正渲染时直接命中浏览器/模块缓存,毫秒级就绪;但**全量预取等于没分割**,常规配置是"流量前 5 的路由空闲预取 + 其余悬停兜底",并尊重 `navigator.connection.saveData` 时放弃预取。

## 5.2 Suspense 与并发

★ **关键认知**：Suspense 不只用于懒加载,在**并发特性**(第 25 篇、第 29 篇流式 SSR)里也是核心边界——它把"异步就绪"这件事统一建模。⚠ 理解"Suspense = 异步边界",是理解 React 现代数据加载(含 React Query 的 suspense 模式)的钥匙。

# 6. 本篇小结与下一步

| 概念 | 要点 |
|------|------|
| import() | 构建期切割信号;产物=带哈希独立 chunk,运行时 Promise 加载 |
| React.lazy | 只认 default;须 Suspense 包裹;定义在组件外;失败不自动重试 |
| route.lazy | 连路由模块(loader/配置)一起拆,首屏更瘦 |
| prefetch | 悬停/空闲/视口三级预热,懒加载的体验补丁 |

★ **关键认知**：代码分割 + 懒加载 = `lazy(() => import())` + `<Suspense fallback>`;路由级分割收益最大;懒的入口是"首屏用不到的东西"。把首屏体积降下来,是 SPA 性能的头等大事。

下一步进入**阶段四 · 高级与架构**,第 24 篇:性能优化——React.memo、不可变、虚拟化与 Profiler。
