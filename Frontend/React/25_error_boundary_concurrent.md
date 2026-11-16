组件报错不该"白屏一整页";大数据计算不该"卡死输入框"。这两件事分别对应**错误边界(ErrorBoundary)** 与**并发特性(Concurrent Features)**。本篇讲清 ErrorBoundary 的写法与边界、Suspense 的异步边界、以及 startTransition/useDeferredValue 背后的"可中断渲染"。

# 1. 错误边界 ErrorBoundary

## 1.1 为什么需要

```jsx
// 某个子组件抛了错,React 默认会"卸载整棵树" → 白屏
// ErrorBoundary 把错误"圈"起来,只降级这部分,其余照常
```

★ **关键认知**：React 渲染时一个组件抛错,**默认会卸载整个树**(白屏);**错误边界**把错误限制在局部,展示 fallback。⚠ 这是"健壮性"而非"可选项"——尤其生产环境,一处报错不该毁掉整个页面。

## 1.2 类组件实现(不可用函数组件)

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false }

  static getDerivedStateFromError(error) {
    return { hasError: true }          // 捕获错误,切换降级 UI
  }
  componentDidCatch(error, info) {
    // 上报错误日志
  }

  render() {
    if (this.state.hasError) return this.props.fallback
    return this.props.children
  }
}

// 使用
<ErrorBoundary fallback={<div>出错了,请刷新</div>}>
  <RiskyComponent />
</ErrorBoundary>
```

★ **关键认知**：错误边界**目前只能用类组件**(靠 `getDerivedStateFromError`/`componentDidCatch` 两个生命周期),函数组件没有对应的 Hook。⚠ 这几乎是现代 React 里"**必须用类组件**"的唯一场景;通常用现成库(react-error-boundary)或直接写一个。

## 1.3 边界能/不能捕获什么

| 能捕获 | 不能捕获 |
|--------|---------|
| 渲染时错误(子组件渲染抛错) | 事件处理器的错误 |
| 生命周期错误 | 异步代码(setTimeout/请求) |
| 构造函数错误 | 服务端渲染错误 |

★ **关键认知**：错误边界能捕获**渲染/生命周期期间**的错误,⚠ 但**不能捕获事件处理器、异步回调、SSR**里的错误(那些要 try/catch 或 window.onerror)。这是 React 错误边界最容易被误解的边界。

# 2. Suspense 的异步边界

## 2.1 不只用于懒加载

```jsx
function Profile() {
  const user = useQuery({ ... })   // 若配 suspense 模式,数据没就绪就"挂起"
  return <div>{user.name}</div>
}

<Suspense fallback={<Spinner />}>
  <Profile />
</Suspense>
```

★ **关键认知**：Suspense 是**通用的"异步就绪边界"**——不限于懒加载,任何"数据没就绪而挂起"的组件(fetch、React Query suspense 模式)都由它兜住 fallback。⚠ 这是 React 对"异步加载"的统一建模,与第 23 篇、第 29 篇流式 SSR 一脉相承。

## 2.2 Suspense 与 ErrorBoundary 的组合

```jsx
<ErrorBoundary fallback={<div>加载失败</div>}>
  <Suspense fallback={<div>加载中…</div>}>
    <Profile />
  </Suspense>
</ErrorBoundary>
```

★ **关键认知**：**Suspense(管加载中)+ ErrorBoundary(管加载失败)** 是优美的组合——一个管"进行中",一个管"出错了",把异步组件的三种状态(加载/失败/成功)分而治之。

# 3. 并发特性(Concurrent)

## 3.1 可中断渲染

★ **关键认知**：React 18 的**并发渲染(Concurrent)**——渲染可被**中断、暂停、恢复**;紧急更新能**打断**低优先级更新。⚠ 这是从"同步一口气渲染完(可能卡 UI)"到"分优先级、可让路"的根本变化,底层靠 Fiber(第 30 篇)。

## 3.2 startTransition 与 useDeferredValue

```jsx
const [isPending, startTransition] = useTransition()

startTransition(() => {
  setSearchResults(expensiveSearch(keyword))   // 标记为低优先级,可被输入打断
})
```

★ **关键认知**：`startTransition`/`useDeferredValue`(第 14 篇)是并发的**用户面**——把"大数据重算"标成低优先级,让"输入框更新"保持流畅。⚠ 它的**效果**是"重算不卡输入";**前提**是用了可中断的渲染(Fiber),否则只是普通延迟。

# 4. 并发特性的收益

## 4.1 更流畅的交互

★ **关键认知**：并发特性的价值场景——**搜索/过滤大数据、tab 切换重列表、页面过渡**——让"昂贵更新"不再阻塞"用户正在做的操作"。⚠ 不需要处处用,针对"慢计算拖垮交互"的局部上 startTransition/useDeferredValue 即可。

## 4.2 与旧模型的心智差异

★ **关键认知**：并发渲染改变了"渲染是同步、一次性"的假设——现在**渲染中可能被打断、重来、丢弃**。⚠ 这要求组件渲染是"**纯的、幂等的**"(可安全重来),也解释为什么 React 强调"渲染纯函数"(第 1 篇)、拒绝在渲染里做副作用。

# 5. 本篇小结与下一步

★ **关键认知**：ErrorBoundary 兜住渲染错误(类组件实现、不捕获异步/事件),Suspense 兜住异步加载,两者组合管"加载/失败/成功"三态。并发特性(可中断渲染)让 startTransition/useDeferredValue 实现"重算不卡交互"。

下一步第 26 篇:TypeScript 集成——泛型组件、事件类型、Hooks 类型,让 React 也类型安全。