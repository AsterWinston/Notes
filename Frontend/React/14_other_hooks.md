常用的 useState/useEffect 之外,React 还内置了一批"进阶 Hook",各解决一类具体问题:稳定 ID、同步布局副作用、并发渲染的优先级控制。本篇讲清 useId、useLayoutEffect、useTransition、useDeferredValue、useSyncExternalStore 这几个,以及它们各自的适用边界。

# 1. useId:生成稳定唯一 ID

## 1.1 用法

```jsx
import { useId } from 'react'

function Input() {
  const id = useId()      // 生成形如 :r0: 的唯一 ID
  return (
    <>
      <label htmlFor={id}>姓名</label>
      <input id={id} />   {/* label 与 input 精确关联 */}
    </>
  )
}
```

★ **关键认知**：`useId` 生成**每个组件实例唯一的稳定 ID**,用于 `htmlFor`/`id`、`aria-labelledby` 等关联。⚠ 用它替代"手写随机 id 或自增计数器"——它**稳定**(SSR 下 hydrate 不会错位、同组件多实例不冲突),这是手写 id 给不了的。

## 1.2 为什么不用 Math.random

★ **关键认知**：`Math.random()` 或自增 id 的坑——**SSR 下服务端和客户端生成的不一致**(hydrate 报错)、组件**多实例会撞**。⚠ `useId` 由 React 维护,稳定且唯一,是"关联 id"的正解;但注意它**不是用来做列表 key** 的(列表 key 靠业务 id)。

# 2. useLayoutEffect:同步副作用

## 2.1 与 useEffect 的差异

```jsx
import { useLayoutEffect } from 'react'

// useLayoutEffect:在 DOM 提交后、浏览器绘制"之前"同步执行
useLayoutEffect(() => {
  const el = ref.current
  const h = el.offsetHeight      // 测量 → 立即改样式,避免闪烁
  el.style.height = `${h}px`
}, [])
```

★ **关键认知**：`useLayoutEffect` 与 useEffect 的执行时机几乎一样,**区别在"绘制前还是绘制后"**——layoutEffect 在**浏览器重绘前同步执行**,effect 在**重绘后异步执行**。⚠ 用途是"**改了 DOM 要避免闪烁**":先测量再改样式,让用户看不到中间态;但它**阻塞绘制**,别放重逻辑。

## 2.2 什么时候用它

| 场景 | 用 |
|------|-----|
| 发请求、订阅、日志 | useEffect |
| 测量并同步改 DOM(防闪) | useLayoutEffect |

★ **关键认知**：**默认用 useEffect**,只有"不立即改 DOM 用户会看到一帧错误画面"(如闪烁、跳出)时才用 useLayoutEffect。⚠ 一个判断口诀:"改了会不会闪?会 → layoutEffect;不会 → effect"。

# 3. useTransition:标记低优先级更新

## 3.1 基本用法

```jsx
import { useState, useTransition } from 'react'

function Search() {
  const [keyword, setKeyword] = useState('')
  const [results, setResults] = useState([])
  const [isPending, startTransition] = useTransition()  // isPending 是否在过渡中

  function onChange(e) {
    const kw = e.target.value
    setKeyword(kw)                      // 紧急:输入框立即更新
    startTransition(() => {             // 非紧急:搜索结果更新可被中断/延后
      setResults(search(kw))
    })
  }

  return (
    <>
      <input value={keyword} onChange={onChange} />
      {isPending ? '搜索中…' : <ResultList results={results} />}
    </>
  )
}
```

★ **关键认知**：`useTransition` 用 `startTransition` 把**非紧急更新标记为低优先级**——输入框这种"必须立刻响应"的是紧急,大数据计算结果"可以慢一点"的就是过渡。⚠ 这是**并发特性(第 25 篇)**的入口:紧急更新可以**打断**低优先级的过渡更新,保证交互流畅;`isPending` 告诉你过渡是否进行中(可用来显示 loading)。

## 3.2 解决的痛

★ **关键认知**：过去"输入搜索 → 重算大列表"会把输入框**卡住**(重算阻塞渲染);useTransition 让"输入框更新"**不被"结果重算"拖累**。⚠ 这是 React 从"同步不可中断"走向"**可中断、分优先级**"(第 30 篇 Fiber)的用户面体现。

# 4. useDeferredValue:延迟一个值

## 4.1 用法

```jsx
const deferredQuery = useDeferredValue(query)  // 延迟 query,低优先级
const results = useMemo(() => search(deferredQuery), [deferredQuery])
```

★ **关键认知**：`useDeferredValue(value)` 返回一个**"延迟"的 value 副本**——紧急更新时它暂不跟新,等空闲了再更新到最新。⚠ 与 useTransition 的区别:useTransition 包装"**更新动作**",useDeferredValue 包装"**一个值**";效果都类似(延迟非紧急部分),选哪个看你要延迟的是"set 调用"还是"某值"。

# 5. useSyncExternalStore

## 5.1 订阅外部 store

```jsx
import { useSyncExternalStore } from 'react'

// 订阅"非 React"的外部状态(如 Zustand/Redux/自定义 store)
const state = useSyncExternalStore(
  store.subscribe,     // 订阅函数(store 变了通知 React)
  store.getSnapshot    // 取当前快照
)
```

★ **关键认知**：`useSyncExternalStore` 让组件**订阅 React 之外的状态源**——第三方 store、浏览器 API、全局变量。⚠ 它是"外部 store 与 React 同步"的官方通道,配合并发特性能避免"撕裂"(tearing,并发渲染读到不一致的状态);日常很少直接写它(通常由 Zustand/Redux 内部实现),但理解它有助读懂那些库。

# 6. 其余 Hook 速查

| Hook | 用途 |
|------|------|
| `useId` | 稳定唯一 ID |
| `useLayoutEffect` | 绘制前同步副作用 |
| `useTransition` | 标记低优先级更新 |
| `useDeferredValue` | 延迟一个值 |
| `useSyncExternalStore` | 订阅外部 store |
| `useInsertionEffect` | CSS-in-JS 库专用(样式注入时机) |
| `useDebugValue` | 自定义 Hook 调试标签 |

# 7. 本篇小结与下一步

★ **关键认知**：useId 稳定 ID、useLayoutEffect 防闪同步副作用、useTransition/useDeferredValue 是并发特性的"低优先级"入口、useSyncExternalStore 订阅外部 store。这些"进阶 Hook"大多是**为特定痛点而生**,懂了场景才懂用法。

下一步第 15 篇:Hooks 规则与心智模型——为什么不能条件调用 Hooks,依赖数组背后的原理。