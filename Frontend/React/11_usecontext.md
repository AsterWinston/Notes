组件层级一深,"爷爷的数据传给重孙子"就要层层转手,这叫 **prop drilling**。`useContext` 就是解法——提供一种"**跨层广播**"机制,让任意后代直接读写上下文。本篇讲清 context 的三件套、何时该用(它是"依赖注入"不是"状态管理")、以及拆 context 的实践。

# 1. prop drilling 之痛

## 1.1 逐层传递的烦恼

```jsx
function App() {
  const [theme, setTheme] = useState('light')
  return <Page theme={theme} />        // 只是为了往下传
}
function Page({ theme }) {
  return <Content theme={theme} />     // 又转一次手
}
function Content({ theme }) {
  return <Button theme={theme} />      // 真正用到的还在下面
}
```

★ **关键认知**：**prop drilling(逐层下钻)**——某个数据要传给深层子组件,中间每一层组件**被迫转手自己根本用不到的 props**。⚠ 它的问题不只是啰嗦:中间组件与数据**产生了无谓耦合**,数据路径一变,一堆"搬运工"组件都要改。

# 2. Context 三件套

## 2.1 create / provide / use

```jsx
import { createContext, useContext, useState } from 'react'

const ThemeContext = createContext('light')     // 1. 创建(带默认值)

function App() {
  const [theme, setTheme] = useState('dark')
  return (
    <ThemeContext.Provider value={theme}>        {/* 2. 提供(Provider 包裹) */}
      <Page />
    </ThemeContext.Provider>
  )
}

function Button() {
  const theme = useContext(ThemeContext)          // 3. 消费(useContext)
  return <button className={theme}>按钮</button>
}
```

★ **关键认知**：Context 三件套——**`createContext(默认)` 创建、`<X.Provider value>` 提供、`useContext(X)` 消费**。⚠ 任意深度后代 `useContext` 能读到**最近的 Provider 的 value**;没有 Provider 时读默认值。Provider 的 value **一变,所有消费它的组件都重渲染**。

## 2.2 value 的稳定性

```jsx
// ⚠ 反模式:value 写成内联对象,每次渲染都是新引用 → 消费组件全重渲染
<ThemeContext.Provider value={{ theme, setTheme }}>

// ✓ 用 useMemo(或把 value 提成稳定引用)避免无谓重渲染
const value = useMemo(() => ({ theme, setTheme }), [theme])
<ThemeContext.Provider value={value}>
```

★ **关键认知**：Provider 的 `value` **每次渲染都是新对象**的话,所有消费组件都会**跟着重渲染**(即使值没实质变)。⚠ 因此 value 是对象时,常配合 `useMemo` 保持引用稳定——这是 Context 用得不好会"全树重渲染"的第一原因。

# 3. Context 的定位

## 3.1 它是依赖注入,不是状态管理

★ **关键认知**：Context 解决的是**"传"(传递机制)**,不是**"管"(状态管理)**——它让数据能跨层直达,但**状态本身仍由某个组件持有**(如上例 theme 在 App)。⚠ 因此"用 Context 做全局状态管理"需要**自己再搭一层**(把 state + 方法一起放 context,或用第 18 篇的 Redux/Zustand);纯 context 没有"时间旅行、模块化"这些状态管理能力。

## 3.2 适合与不适合

| 适合 | 不适合 |
|------|--------|
| 主题/语言/登录态 | 高频变化的数据(每次变全树重渲染) |
| 全局配置 | 需要精确"谁订阅谁"的数据 |
| 深层共享的少量数据 | 大而复杂的状态 |

★ **关键认知**：Context 适合**低频、全局、少量**的数据(主题、语言、用户信息);⚠ **高频变化**的数据放 Context,会让消费它的所有组件频繁重渲染(需配合拆分/memo 优化)。这与 Vue 的 provide/inject 是同一思想、同一边界。

# 4. 拆 Context 的实践

## 4.1 一个 context 管一件事

```jsx
const ThemeContext = createContext()
const UserContext = createContext()
const LocaleContext = createContext()

// 各自独立 Provider,谁变只重渲染"关心它的人"
```

★ **关键认知**：**一个 Context 只承载一类数据**——主题、用户、语言各自独立,避免"一个巨 context 装所有,任何值一变全树重渲染"。⚠ 大部分 context 性能问题,都源于"**一个 context 塞太多**",拆分就是解药。

## 4.2 封装 Provider + 自定义 Hook

```jsx
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light')
  const value = useMemo(() => ({ theme, setTheme }), [theme])
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
}

function useTheme() {
  return useContext(ThemeContext)     // 封装:消费时不用直接碰 createContext
}
```

★ **关键认知**：工程里常把 **Provider 封装成组件**、**消费封装成 `useTheme` Hook**——对外只暴露干净接口,内部 stable value 也一并处理。⚠ 这是 context 的最佳实践形态,也是做"轻量状态(如主题)"的标准套路。

# 5. 本篇小结与下一步

★ **关键认知**：Context 解决 prop drilling——`createContext`/`Provider`/`useContext` 三件套,跨层直达。定位是"依赖注入"非"状态管理";适合低频全局数据;value 要稳定(配 useMemo);一 context 管一事。

下一步第 12 篇:useReducer 与复杂状态——reducer/dispatch 模式,把状态更新逻辑集中化。