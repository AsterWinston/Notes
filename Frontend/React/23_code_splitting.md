单页应用越写越大,首屏要加载的 JS 越多越慢。**代码分割(Code Splitting)+ 懒加载** 把"一次性加载全部"变成"按需加载"。本篇讲清千里之外的 `React.lazy`、`Suspense`、动态 `import()`,以及路由级懒加载的最佳实践。

# 1. 为什么要分割

## 1.1 bundle 太大的问题

```
不分割:整个应用打成一个巨大的 main.js,用户首屏全要下载
分割:按路由/组件切成多个 chunk,首屏只加载"当前页面需要的那部分"
```

★ **关键认知**：SPA 打包后若只有一个大 bundle,**首屏要下载全部代码**才渲染,体感极慢。⚠ 代码分割把 bundle 拆成小块,**访问到哪个功能才下载哪个**,首屏体积大幅下降——这是 SPA 性能优化的第一优先事项。

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

# 4. 分割的粒度与收益

## 4.1 组件级 vs 路由级

| 粒度 | 例子 | 收益 |
|------|------|------|
| 路由级 | 每个页面懒加载 | 首屏只含当前页(收益最大) |
| 组件级 | 图表/地图/富文本编辑器 | 首屏不计入首屏用不到的重组件 |
| 库级 | 大型第三方库随之分包 | 减少主包 |

★ **关键认知**：**路由级分割收益最大**(首屏直接砍掉其他页面);**组件级**针对"某页面里的大块头"(图表、编辑器)进一步拆。⚠ 切割粒度要权衡:过细会增请求数,通常"**路由级必做,组件级按需**"。

## 4.2 首屏指纹

```
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
```

★ **关键认知**：懒加载会带来"点击后要等一下才下载",可**预加载**(prefetch)下一步——悬停/进入视口时提前下好 chunk。⚠ 这是"懒加载 + 预加载"的互补:懒加载省首屏,预加载补交互流畅。

## 5.2 Suspense 与并发

★ **关键认知**：Suspense 不只用于懒加载,在**并发特性**(第 25 篇、第 29 篇流式 SSR)里也是核心边界——它把"异步就绪"这件事统一建模。⚠ 理解"Suspense = 异步边界",是理解 React 现代数据加载(含 React Query 的 suspense 模式)的钥匙。

# 6. 本篇小结与下一步

★ **关键认知**：代码分割 + 懒加载 = `lazy(() => import())` + `<Suspense fallback>`;路由级分割收益最大;懒的入口是"首屏用不到的东西"。把首屏体积降下来,是 SPA 性能的头等大事。

下一步进入**阶段四 · 高级与架构**,第 24 篇:性能优化——React.memo、不可变、虚拟化与 Profiler。