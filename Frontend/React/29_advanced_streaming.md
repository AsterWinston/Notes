SSR 解决"首屏快",但"整页一起等最慢的数据"仍是痛点。**流式渲染(Streaming)** 让 HTML 分块流出、先到先显示;**Suspense for SSR** 让慢的部分各自独立加载;缓存则把重复计算省掉。本篇讲清这三样,把 SSR 的性能抠到极致。

# 1. 流式渲染 Streaming

## 1.1 传统 SSR 的"整页等待"

```
传统 SSR:等"最慢的那个数据"就绪,才一次性返回完整 HTML
  → 一个慢接口拖累整页首屏
流式渲染:HTML 分块流式返回,快的部分先渲染给用户
  → 头部、静态部分先到,慢的部分后再补
```

★ **关键认知**：**流式渲染**把 HTML **分块(流)** 发给浏览器——不等的部分先渲染,**最慢的那块不再拖累全局**。⚠ 这是 SSR 性能的关键升级:首屏 TTFB(首个字节)大幅提前,慢数据"各自慢各自的"。

## 1.2 流式的体感

★ **关键认知**：用户感知上,流式渲染让"**骨架/静态内容先出现,数据区随后填充**"——比"整页白屏等最慢一起出来"好太多。⚠ 这也是 Next.js App Router 里 RSC/Suspense 的默认行为:慢组件用 Suspense 包住,其余先渲染。

## 1.3 两个就绪回调:shell 与人、all 与爬虫

```jsx
// Node 侧的流式 API(React 18+)
import { renderToPipeableStream } from 'react-dom/server'

const { pipe, abort } = renderToPipeableStream(<App />, {
  onShellReady() { pipe(res) },         // 外壳(除 Suspense 内容)就绪 → 立刻开始流
  onAllReady()   { /* 全量就绪:爬虫/静态导出走这里 */ },
})
setTimeout(() => abort(), 5000)          // 超时中断,降级为"外壳 + 客户端兜底"

// Edge/Worker 环境用 Web Streams 版:renderToReadableStream(<App />)
```

★ **关键认知**：流式 SSR 的开关是**两个就绪时机**——`onShellReady`(静态壳+能同步渲染的部分)先到,`onAllReady`(含全部 Suspense 内容)留给爬虫与预渲染;人看走 shell,机器看走 all。⚠ 自建服务端最容易栽的坑:**压缩/缓存中间件把流憋成整包**(nginx 默认缓冲、gzip 攒满再发),分块优势瞬间归零——"开了 streaming 还白屏"先查响应是否 `Transfer-Encoding: chunked`;Next.js 内部已处理,自己搭的要盯紧。

## 1.4 慢块是"迟到补丁":HTML 如何自我修复

```html
<!-- 同一条流里先后到达(伪代码) -->
<div id="S:0"><template hidden><!-- 骨架占位 --></template></div>
…慢数据就绪后流里追加:…
<div hidden id="S:0">商品列表真实 HTML</div>       <!-- 真身藏在 hidden 容器 -->
<script>$RC("B:0","S:0")</script>                  <!-- 运行时指令:替换进占位处 -->
```

★ **关键认知**：流式 SSR 的实现是**"占位符 + hidden 真身 + 内联脚本替换"**——不需要二次请求,HTML 自带"自我修补"指令,浏览器边收边解析边换。⚠ 由此有个体验细节:真身替换骨架时**页面高度可能突变(CLS)**,fallback 骨架的高度应尽量贴近真实内容。

# 2. Suspense for SSR

## 2.1 让慢的部分独立加载

```jsx
// 服务端渲染里,把"慢数据"的组件用 Suspense 包住
<Page>
  <Header />                         {/* 静态,立即渲染 */}
  <Suspense fallback={<Skeleton />}>  {/* 慢数据,自己等 */}
    <ProductList />
  </Suspense>
</Page>
// 流式:Header 先到,ProductList 就绪后再"补"进来,且各自处理加载态
```

★ **关键认知**：**Suspense for SSR** = 用 Suspense 把慢组件隔开,**各自"流式"就绪**——快的先渲染,慢的到了再补,fallback 是期间的占位。⚠ 这是第 25 篇 Suspense 在服务端的延伸:同一个异步边界,客户端管加载,服务端管流式。

## 2.2 粒度越细,体验越好

★ **关键认知**：Suspense 可以**嵌套、拆得很细**——把"慢数据"尽量拆小,Suspense 边界越细,"被慢部分拖累"的范围越小。⚠ 最佳实践:按"数据依赖"划 Suspense 边界,让每个慢请求只影响它自己的那一小块。

## 2.3 边界还是"并行开关"

```jsx
function Page() {
  return (
    <>
      <Shell />                                     {/* 零数据依赖,shell 里直接出 */}
      <Suspense fallback={<Skel />}><Profile /></Suspense>   {/* 内部各自 await */}
      <Suspense fallback={<Skel />}><Orders /></Suspense>
    </>
  )
}
// Profile/Orders 内部自己 fetch → 两条 IO 并发发出,谁先回谁先流
```

★ **关键认知**：Suspense 边界的数量决定了 **SSR 阶段请求的并行度**——同层多个边界内的数据请求**并发**进行、先到先补;若把 await 全提到页面顶层"串行取完再渲染",流式只剩一个空壳可发,退化成"传统 SSR 的延迟 + 流式的复杂度"。⚠ "把 await 埋进各 Suspense 区块内部"在 App Router 里不只是代码风格,它就是**调度结构**。

# 3. 缓存 cache 与数据

## 3.1 请求级缓存

```jsx
// React 的 cache:同一渲染内,相同参数的函数只执行一次(去重)
import { cache } from 'react'

const getUser = cache(async (id) => {
  return await fetch(`/api/user/${id}`)
})
// 多个组件要同一用户数据时,不会重复请求
```

★ **关键认知**：React 的 **`cache()`** 让"同一渲染内相同参数的函数"**只执行一次**——多个组件请求同一数据时**去重**,避免重复请求。⚠ 这是 RSC 场景下"组件各自请求、框架负责合并去重"的基础设施,替代手写"查重"。

## 3.2 缓存的层次

| 层 | 手段 | 作用 |
|----|------|------|
| 请求级 | React cache() | 同渲染内去重 |
| 组件级 | React.memo | 跳重渲染 |
| 数据级 | React Query | 跨请求缓存 |
| 页面级 | Next.js revalidate | 定时再生成 |

★ **关键认知**：缓存是多层的——**请求去重(cache)、组件渲染(memo)、数据缓存(Query)、页面再生(revalidate)** 各管一段。⚠ 理解"哪个手段解决哪一层的重复计算",SSR 的性能优化就有了地图,而不是零散堆优化。

## 3.3 cache 的键与生命周期

```jsx
const getUser = cache((id) => fetchUser(id))
getUser(1); getUser(1)                    // ✓ 只请求一次:参数按"值"比较(标量稳定)
getUser({ id: 1 }); getUser({ id: 1 })    // ⚠ 两个不同对象字面量——不保证命中,传标量!
```

★ **关键认知**：`cache()` 的键**按参数逐个比较**,标量(数字/字符串)最可靠,**每次新建的对象字面量可能不命中**;生命周期是"**本次渲染**"——请求结束即失效。⚠ 把它当全局 LRU 用是最常见误解;`cache` 管"同渲染去重",框架层 revalidate 管"跨渲染复用",两层别混。

# 4. 与客户端的关系

## 4.1 流式后的水合

★ **关键认知**：流式渲染后,水合(hydrate)也要**渐进**——流到哪块就交互到哪块(React 的 selective hydration)。⚠ 这是流式渲染的完整闭环:"先看到内容,交互随后跟上";核心是不让"水合"重新成为新的"整页等待"。

## 4.2 渐进增强的心智

★ **关键认知**：SSR + 流式 + RSC 的整体心智是"**渐进增强**"——内容优先(SSR)、快块先行(流式)、交互补上(hydrate)。⚠ 这条链要求组件"**客户端/服务端都渲染得一致**"(呼应第 27 篇水合不匹配的坑),是服务端渲染的一贯约束。

## 4.3 Selective Hydration:水合可以插队

```
朴素水合:整页 HTML 就位 → 从根逐棵树跑 render 接事件 → 大页面 = "看得见但点不动"
选择性水合(React 18,建立在可中断渲染上,第 30 篇):

  <Page>
    <SlowList>                      ◀─ 还在流式补 HTML / 排队水合
      <button onClick={fn}>领取</button>   ◀─ 用户点了这里!
    </SlowList>
  </Page>

1. 未水合的树上,React 先用原生监听(捕获阶段)兜住事件
2. 点击发生 → 该子树优先级提升 → 中断当前水合,优先水合"被点的这棵"
3. 水合完成 → 重放刚才暂存的事件 → 按钮响应,其余子树继续排队
```

★ **关键认知**：selective hydration 的本质是**把"交互"当成最高优先级的水合信号**——用户点哪,哪棵子树先"活",靠的是 Fiber 的可中断/可恢复(第 30 篇)。⚠ 它插队的是**顺序**而非豁免:事件会被**暂存重放**,若水合前后 DOM 不一致(第三方脚本、时间/随机数导致的 mismatch),重放链会放大报错;`suppressHydrationWarning` 只豁免文本差异,救不了结构差异。

## 4.4 RSC 的另一半:数据流而非 HTML 流

```jsx
// 服务端组件把 promise 作为 prop 传下 'use client' 边界,客户端 use() 到点才等
async function Page() {
  const p = fetchUser(id)                  // 立即发起,不 await
  return <Gated data={p} />                // Gated 是客户端组件
}
function Gated({ data }) {
  const user = use(data)                   // 挂起 → 就近 Suspense 兜住
  return <div>{user.name}</div>
}
```

★ **关键认知**：RSC 的输出不只是 HTML,还有一条**可流的序列化 UI 描述(Flight)**——客户端导航时只传变化的组件子树,**客户端组件的状态按引用保留**,不像整页 SSR 那样推倒重来。⚠ `use()` 是唯一能消费"跨边界传来的 promise"的 Hook(可在条件里调,与 useState 纪律不同);但服务端组件树里**乱换 key/结构**,等于在数据流里亲手重置客户端子树状态——第 5 篇的 key 机制在 RSC 时代照样生效。

# 5. 本篇小结与下一步

★ **关键认知**：流式渲染分块返回(慢不再拖累全局)、Suspense for SSR 让慢组件独立加载、cache() 去重同渲染内的重复请求。SSR 的性能三板斧,核心是"**快的不等慢的、重复的不重算**"。

★ 机制层补一句:三板斧背后是同一条设计线——shell/all 两档就绪、$RC 迟到补丁、边界并行、水合插队、Flight 增量,React 把"整页等待"逐级拆成了"每块各自的等待"。

下一步进入**阶段五 · 原理与面试**,第 30 篇:Fiber 架构与协调——可中断渲染、双缓存树、lanes 与时间切片。
