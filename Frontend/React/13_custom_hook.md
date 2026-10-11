自定义 Hook 是 React 的**复用之道**——把"状态 + 副作用 + 逻辑"打包成以 `use` 开头的函数,任何组件一行引入。本篇讲清自定义 Hook 的写法与规则、经典用例(useFetch、useLocalStorage)、以及它与 Vue composable 的对照。

# 1. 自定义 Hook 是什么

## 1.1 抽取共享逻辑

```jsx
import { useState, useEffect } from 'react'

// useFetch:把"数据请求"这块逻辑整体封装成 Hook
function useFetch(url) {
  const [data, setData] = useState(null)
  const [loading, setLoading] = useState(false)

  useEffect(() => {
    let cancelled = false
    setLoading(true)
    fetch(url).then(r => r.json()).then(d => {
      if (!cancelled) setData(d)
      setLoading(false)             // 竞态处理:被取消的请求不落地
    })
    return () => { cancelled = true }  // 清理:新请求到来时取消旧的
  }, [url])

  return { data, loading }
}

// 组件里直接用
function Page() {
  const { data, loading } = useFetch('/api/user')
  if (loading) return <div>加载中</div>
  return <div>{data?.name}</div>
}
```

★ **关键认知**：自定义 Hook = **以 `use` 开头、内部调用其他 Hook、返回状态/方法的普通函数**。⚠ 命名必须 `use` 开头——这是 React 识别"这是 Hook、要遵守 Hooks 规则"的约定(也是 ESLint 检查 Hooks 规则的依据)。

## 1.2 复用而非重写

★ **关键认知**：自定义 Hook 的价值是**复用逻辑 + 隔离关注点**——`useFetch`/`useLocalStorage`/`useDebounce` 这类通用逻辑,写一次处处用。⚠ 与"复制粘贴相同逻辑"相比,自定义 Hook 让逻辑**单点维护、类型可用、来源清晰**,这正是对比 Vue composable(第 11 篇)与 React 的 mixin 时代的演进方向。

# 2. 经典用例

## 2.1 useLocalStorage

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {             // 惰性初始化:读 localStorage
    const stored = localStorage.getItem(key)
    return stored ? JSON.parse(stored) : initialValue
  })

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value))     // 值变了持久化
  }, [key, value])

  return [value, setValue]
}
// 用法:[name, setName] = useLocalStorage('name', 'Alice')
```

★ **关键认知**：`useLocalStorage` 把"**state + 持久化**"粘成一个 Hook——组件里像用 useState 一样用,但值自动存进 localStorage。⚠ 复用价值一目了然:任何"要背下来的输入"都可直接换这个 Hook,不用每处写一遍 localStorage 逻辑。

## 2.2 useDebounce

```jsx
function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay)  // 延迟更新
    return () => clearTimeout(timer)                            // 期间重设定时器
  }, [value, delay])

  return debounced
}
// 搜索输入防抖:const debouncedKeyword = useDebounce(keyword)
```

★ **关键认知**：`useDebounce` 是"**值变化太快,等它稳定再算**"的通用解(搜索输入、窗口缩放)。⚠ 它的清理函数(clearTimeout)正是第 8 篇"effect 清理"的实战——值又变了就先取消上一次的定时器。

# 3. 与 Vue composable 对照

| 维度 | React 自定义 Hook | Vue 组合式函数 |
|------|-------------------|----------------|
| 命名 | `useXxx` | `useXxx` |
| 复用单位 | 函数 | 函数 |
| 依赖追踪 | 手动依赖数组 | 自动 |
| 执行 | 每次渲染都跑 | setup 里跑一次 |
| 记忆化 | useMemo/useCallback | computed 自动缓存 |

★ **关键认知**：两者**思想同源**(封装逻辑、相同命名、复用即新建状态),**执行模型不同**——React 的 Hook 每次渲染都执行、靠依赖数组,所以需要 useMemo/useCallback 记忆化;Vue 的组合式函数 setup 里跑一次、靠 Proxy 自动追踪。⚠ 这个差异(第 11、15 篇反复出现)是理解两大框架"同样需求,不同心智"的关键。

# 4. 写自定义 Hook 的纪律

## 4.1 保持 Hook 可复用

```jsx
// 好:输入参数,输出状态/方法,自包含
function useToggle(initial = false) {
  const [on, setOn] = useState(initial)
  return [on, () => setOn(v => !v)]   // [值, 切换函数]
}
// 坏:假设了外部某个变量/某个组件结构,不可独立复用
```

★ **关键认知**：好的自定义 Hook 要**参数进、返回值出、自包含**——不依赖"外部某个具体变量",才能在任意组件复用。⚠ 通用 Hook 追求"单一职责"(一个 Hook 只做一件事),需要组合时在组件里组合多个 Hook。

## 4.2 返回值建议用对象/元组

```jsx
return { data, loading, error }        // 对象:字段多、可解构重命名
return [value, setValue]               // 元组:像 useState 那样,顺序即约定
```

★ **关键认知**：返回值两种风格——**对象**(字段多、解构自由)与**元组**(像 useState,顺序约定)。⚠ 元组的好处是调用处可重命名(`const [a, setA] = ...`),对象的好处是字段明确;选与团队惯例一致即可。

# 5. 本篇小结与下一步

★ **关键认知**：自定义 Hook = `use` 开头、内部用其他 Hook、复用逻辑的函数;典型 useFetch/useLocalStorage/useDebounce。它与 Vue composable 同源,差异在执行模型(每次渲染 vs 跑一次)。

下一步第 14 篇:其余内建 Hooks——useId/useLayoutEffect/useTransition/useDeferredValue 等进阶 Hook。