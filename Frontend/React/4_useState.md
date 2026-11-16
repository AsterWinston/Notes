没有状态的组件是死的,有了 `useState`,组件就有了"记忆"。本篇讲清状态的定义与更新、**不可变更新**的原则、**批处理**的行为,以及最著名也最反直觉的**闭包陷阱**——这些是 React 一切状态逻辑的地基。

# 1. useState 基础

## 1.1 定义与读取

```jsx
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)   // [值, 更新函数] = useState(初始值)
  return <button>{count}</button>         // 直接读 count
}
```

★ **关键认知**：`useState(初始值)` 返回 `[state, setState]`——**当前值**与**更新函数**;初始值只在**首次渲染**时用一次。⚠ 每次重新渲染,`count` 都是**本次渲染的"快照值"**——它不会"自己变",只有 `setCount` 触发重渲染后,新一次渲染里 count 才是新值。

## 1.2 初始值可以是函数

```jsx
// 惰性初始化:初始值计算昂贵时,传函数(只在首次渲染执行一次)
const [todos, setTodos] = useState(() => loadTodos())
```

★ **关键认知**：初始值是函数时,**惰性求值**,只在首次渲染执行(适合昂贵的初始化)。⚠ 若直接写 `useState(loadTodos())`,每次渲染都会调用一次(虽然只取第一次结果)——性能浪费,惰性函数可避免。

# 2. 更新与不可变

## 2.1 setState 触发重渲染

```jsx
function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(count + 1)}>{count}</button>
  // 点击 → setCount(count+1) → 触发重渲染 → count 变为新值
}
```

★ **关键认知**：`setState(新值)` 的完整链路——**触发重新渲染 → 组件函数再次执行 → 读到新值**。⚠ "React 的更新永远靠重新渲染"——没有"改一下这个变量"这回事,状态是快照,变了就得整套重渲染(配合 Diff 只改必要 DOM)。

## 2.2 不可变更新(对象/数组)

```jsx
function TodoList() {
  const [todos, setTodos] = useState([])

  function add(text) {
    // ✗ 错误:直接 push 到原数组再 set,React 可能"看不出"变了
    // todos.push(text); setTodos(todos)

    // ✓ 正确:创建新数组(展开旧数组 + 新项)
    setTodos([...todos, text])
  }
}
```

★ **关键认知**：更新对象/数组必须**产生新引用**(不可变 immutable),用展开 `[...todos, x]` 或 `{...obj, key: val}`——否则 React 比较前后引用**发现一样,就跳过更新**。⚠ 这是"为什么我 push 了却界面不变"的头号答案:原地改坏了,新引用才触发。深层对象要"**逐层**新引用"(见 2.3)。

## 2.3 深层更新的展开

```jsx
const [user, setUser] = useState({ name: 'Alice', info: { age: 18 } })

// 深层更新:每一层都要展开新对象
setUser({ ...user, info: { ...user.info, age: 19 } })
```

★ **关键认知**：更新嵌套对象,**改到哪层、哪层就新建对象**(上面的 `...user` 和 `...user.info` 都不能省)。⚠ 深层嵌套用展开会变啰嗦,可选 `Immer`(第 12 篇)或把状态设计得更扁平——"浅状态"本身就是好的设计。

# 3. 批处理(batching)

## 3.1 一次渲染合并多次 setState

```jsx
function handle() {
  setCount(c => c + 1)
  setCount(c => c + 1)
  setCount(c => c + 1)
  // 三个 setCount 合并在同一次渲染里,count 一次加 3(不是渲染 3 次)
}
```

★ **关键认知**：React 18 对**同一事件/同步块里的多个 setState 做批处理(batching)**——合并成**一次重渲染**,提升性能。⚠ 也因此"连三次 setCount"不会渲染三次;若拿旧值 `count + 1` 连写三次,因闭包里 count 不变,结果还是 +1(见第 4 节)。

## 3.2 函数式更新(基于前值)

```jsx
setCount(count + 1)      // ❌ 拿的是"本次渲染快照"的 count,连写三次只 +1
setCount(c => c + 1)     // ✓ 函数式:基于"前一个值"算,连写三次 +3
```

★ **关键认知**：当新值**依赖旧值**、且可能**连续多次更新**时,用**函数式更新** `setCount(c => c + 1)`。⚠ 这是最重要的一个 React 规则:**值可能"过期"时永远用 `c => ...` 形式**(旧值来自"当前排队应用前的值",而非渲染快照),尤其点击/定时器里连续更新。

# 4. 闭包陷阱(stale closure)

## 4.1 定时器里的旧值

```jsx
const [count, setCount] = useState(0)

useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1)     // ⚠ 闭包捕获的是初始 count=0,永远 +1
  }, 1000)
}, [])                     // 空依赖,effect 只跑一次
```

★ **关键认知**：**闭包陷阱**——回调函数**捕获了"创建那一刻"的 state 快照**,如果 effect 只跑一次(空依赖),定时器里读到的 `count` 永远是初始值。⚠ 解法:`setCount(c => c + 1)`(函数式更新,不读闭包里的旧值),这是闭包陷阱的第一解药。

## 4.2 为什么会出现

★ **关键认知**：每次渲染,组件函数**重新执行**,产生**一套新的 state 与闭包**;旧的闭包"冻结"了旧值。⚠ 理解这一点才能真正理解 React 的"**快照语义**":event handler、effect 里读到的 state,是**它们那次渲染的值**——想要"当前值",要么用函数式更新,要么把依赖写对(第 8 篇)。

# 5. 本篇小结与下一步

★ **关键认知**：useState 返回 `[值, 更新函数]`;更新对象/数组要**不可变**(新引用);多次 setState 会**批处理**;值依赖旧值时用**函数式更新** `c => c+1`;闭包陷阱的根因是"快照语义",解药是函数式更新 + 正确依赖。

下一步第 5 篇:事件处理与列表 key——合成事件、事件委托,以及列表为什么必须有 key。