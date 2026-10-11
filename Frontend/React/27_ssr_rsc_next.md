React 的战场早已从"浏览器"延伸到"服务端"——**SSR(服务端渲染)** 提速首屏与 SEO,**RSC(Server Components)** 让服务端直接渲染组件,而 **Next.js** 是把这些落地的全栈框架。本篇讲清 SSR 与 CSR 的差别、RSC 的模型、Next.js 的形态,以及它改变了什么。

# 1. CSR 与 SSR 的差别

## 1.1 从 CSR 说起

```
CSR(客户端渲染):
  浏览器 → 下载空壳 HTML → 下载 JS → JS 在浏览器里渲染出内容
  问题:首屏慢(要等 JS 下载执行)、SEO 差(爬虫看到的是空壳)
```

★ **关键认知**：普通 React SPA 是 **CSR(客户端渲染)**——HTML 是空壳,内容靠 JS 在浏览器里渲染。⚠ 两大代价:**首屏慢**(要下完 JS 才见内容)与 **SEO 弱**(爬虫抓不到内容)。这正是 SSR 要解决的。

## 1.2 SSR 的思路

```
SSR(服务端渲染):
  浏览器 → 服务端先渲染出"带内容的 HTML"返回(首屏快、有内容可抓)
       → 浏览器再"水合"(hydrate)成可交互的 React 应用
```

★ **关键认知**：SSR 让**服务端先渲染出完整 HTML**——首屏马上有内容(无需等 JS),搜索引擎也能抓到。⚠ 术语 **hydrate(水合)**:服务端渲染的静态 HTML,在浏览器里"接上" JS 事件与状态,变成可交互的 React 应用。SSR 不等于"没有 JS",而是"**先给内容、再补交互**"。

# 2. Server Components(RSC)

## 2.1 服务端直接渲染组件

```jsx
// RSC 模型:默认组件在"服务端"渲染
async function ProductList() {
  const products = await fetchProducts()     // 服务端直接读 DB / 请求
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>
}
// 客户端不参与,也不把这些数据的代码打进客户端 bundle
```

★ **关键认知**：**RSC(React Server Components)** 让组件**在服务端渲染、结果作为 HTML/流发给客户端**——服务端能直接读数据库、请求,且这些代码**不进客户端 bundle**。⚠ 与单纯 SSR 的区别:SSR 是"整个页面在服务端渲染一遍",RSC 是"**组件粒度**决定哪些在服务端、哪些在客户端",更精细。

## 2.2 "use client" 标记

```jsx
'use client'          // 声明这是"客户端组件"(有交互、有状态)

import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)   // 这类交互组件必须标 client
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

★ **关键认知**：RSC 里**默认服务端组件**,需要交互/状态的组件用 **`'use client'`** 显式标记。⚠ 这形成"**服务端组件(数据+静态)+ 客户端组件(交互)**"的组合——数据靠服务端、交互靠客户端,各取所长,这是 React 全新的服务端模型。

# 3. Next.js

## 3.1 App Router 形态

```jsx
// Next.js App Router:基于文件系统路由,文件即路由
// app/page.tsx          → 路径 /
// app/about/page.tsx    → 路径 /about
// app/users/[id]/page.tsx → 动态路由 /users/123

export default async function Page() {
  const data = await getData()   // 服务端组件可 async,直接 await 数据
  return <div>{data}</div>
}
```

★ **关键认知**：**Next.js 14+ 的 App Router**——**文件系统即路由**,服务端组件可 **`async` 直接 `await` 数据**。⚠ 这是 RSC + SSR + 路由 + 数据获取的**一站式封装**,是 React 官方的"全栈答案";文件结构(而不是路由配置)决定 URL。

## 3.2 Next.js 涵盖什么

★ **关键认知**：Next.js 把 React 生态的散件**集成**起来——SSR/RSC、流式渲染、路由、数据获取、图片/字体优化、部署。⚠ 它不是"另一个框架",而是"React 的一套最佳实践封装";用 Next 就是接受了它的一整套约定(文件路由、服务端组件优先)。

# 4. SSR 的代价与边界

## 4.1 复杂度上升

★ **关键认知**：SSR/RSC 的代价——**服务端环境、串行/并行的数据、水合一致性、缓存策略**,复杂度远高于纯 CSR。⚠ "首屏/SEO 值不值得这份复杂度"是关键判断:**内容型站点(博客、电商、文档)值得,后台系统/纯工具类 SPA 未必**。

## 4.2 水合的坑

★ **关键认知**：SSR 的一个经典坑是**水合不匹配(hydration mismatch)**——服务端渲染的 HTML 与客户端首次渲染不一致,React 报错。⚠ 根因常是"渲染依赖了浏览器才有的东西"(如 `window`、`Math.random`、当前时间),解法是"服务端/客户端渲染结果保持一致"或"副作用放 useEffect"。

# 5. 本篇小结与下一步

★ **关键认知**：CSR 首屏慢/SEO 弱;SSR 服务端预渲染 HTML + 客户端 hydrate;RSC 让组件粒度上分服务端/客户端(`use client`);Next.js 是这套的集成封装。核心变化:React 从"纯客户端库"走向"**服务端 + 客户端**"的全栈模型。

下一步第 28 篇:测试——React Testing Library/Jest/vitest,组件与 Hook 的测试之道。