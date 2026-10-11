React 的声明式模型覆盖了大部分场景,但总有些事需要"命令式逃逸"——聚焦输入框、拿 DOM 尺寸、保存"不改也无关渲染"的可变值。`useRef` 就是那把逃生门。本篇讲清 ref 的三种用途、`ref` 拿 DOM、`forwardRef` 转发、`useImperativeHandle` 暴露方法。

# 1. useRef 的三种用途

## 1.1 拿 DOM 节点

```jsx
import { useRef, useEffect } from 'react'

function AutoFocus() {
  const inputRef = useRef(null)          // 创建 ref 容器
  useEffect(() => {
    inputRef.current.focus()             // 挂载后聚焦
  }, [])
  return <input ref={inputRef} />        // ref 挂到元素上,current 即 DOM
}
```

★ **关键认知**：`useRef(初始值)` 返回一个 **`{ current }` 容器**;把 ref 挂到 JSX 元素上,渲染后 `current` 指向**那个真实 DOM 节点**。⚠ 访问时机要在**提交后**(useEffect 或事件里),渲染期间 `current` 还没就绪。

## 1.2 保存可变值(不改也不渲染)

```jsx
const timerRef = useRef(null)            // 保存定时器 id
function start() {
  timerRef.current = setInterval(...)
}
// 改 ref.current 不会触发重渲染!
```

★ **关键认知**：`useRef` 也能当**"可变值的保险箱"**——存定时器 id、上一次的值等,改了 **不触发重渲染**。⚠ 与 state 的区别:**state 变了会重渲染,ref 变了不重渲染**;所以"需要触发渲染的值"用 state,"只存不改视图的"用 ref(如计时器、节流标记)。

## 1.3 持有"跨渲染"的引用

★ **关键认知**：ref 的 `current` 在**每次渲染间保持不变**(同一个容器对象),这使它适合存"**不希望被快照、要看最新值**"的东西。⚠ 对比 state:state 每次渲染是"快照",ref 则是"指向最新"——这也是后面(第 14 篇)某些高级 Hook 的原理。

# 2. ref 与 state 的抉择

| 维度 | state | ref |
|------|-------|-----|
| 改后触发渲染 | 是 | 否 |
| 渲染间是否快照 | 是(快照) | 否(最新) |
| 典型用途 | 视图依赖的数据 | DOM/计时器/缓存值 |

★ **关键认知**：一句话抉择——**"视图需要它变而变"用 state,"视图不依赖它"用 ref**。⚠ 别拿 ref 存"本该重渲染"的数据(改了没反应),也别拿 state 存"高频变化又不用渲染"的值(浪费重渲染)。

# 3. forwardRef 转发 ref

## 3.1 把 ref 传给子组件

```jsx
import { forwardRef, useRef } from 'react'

// forwardRef:让父组件能把 ref 传进来,指到子组件内部的元素
const MyInput = forwardRef(function MyInput(props, ref) {
  return <input ref={ref} {...props} />
})

function Parent() {
  const ref = useRef(null)
  return (
    <>
      <MyInput ref={ref} />
      <button onClick={() => ref.current.focus()}>聚焦</button>  {/* 聚焦子组件里的 input */}
    </>
  )
}
```

★ **关键认知**：普通函数组件**不能直接接收 ref**(ref 不是普通 prop);用 `forwardRef` 包裹,才能把父的 ref **转给内部元素**。⚠ 这是"父组件直接操作子组件 DOM"的标准通道(如聚焦、滚动),常用于封装 Input/Button 这类"透传原生元素"的组件。

## 3.2 useImperativeHandle 暴露方法

```jsx
import { forwardRef, useImperativeHandle, useRef } from 'react'

const Modal = forwardRef(function Modal(props, ref) {
  const [visible, setVisible] = useState(false)
  // 暴露"方法"给父组件,而非直接暴露 DOM
  useImperativeHandle(ref, () => ({
    open() { setVisible(true) },
    close() { setVisible(false) }
  }))
  return visible ? <div>弹窗</div> : null
})

// 父组件:ref.current.open() 打开弹窗
```

★ **关键认知**：`useImperativeHandle` 让子组件**自定义"对外暴露什么"**——父拿到的 ref 是你定义的 `{ open, close }`,而非内部 DOM。⚠ 它给"父命令式子组件"(打开弹窗、聚焦、滚动)提供了**克制、受控**的接口,比裸露 DOM 更安全、更解耦。

## 3.3 命令式的边界

★ **关键认知**：ref/forwardRef/useImperativeHandle 都是**"命令式逃逸"**,应该**少用**——优先声明式(用 props/state 描述)。⚠ 只在"确实无法用声明式表达"(聚焦、滚动、测量、命令式弹窗)时才动用;命令式越多,React 的声明式优势越被削弱。

# 4. 实战:滑回顶部的完整例子

```jsx
function Chat() {
  const listRef = useRef(null)

  function scrollToBottom() {
    listRef.current.scrollTop = listRef.current.scrollHeight  // 直接操作 DOM 滚动
  }
  return (
    <div>
      <ul ref={listRef} style={{ overflow: 'auto', height: 200 }}>
        {/* 消息列表 */}
      </ul>
      <button onClick={scrollToBottom}>回到底部</button>
    </div>
  )
}
```

★ **关键认知**："**滚动到底部**"是 ref 的经典用途——滚动位置不是"声明式状态",直接用 ref 操作 DOM 更自然、更高效。⚠ 这类"操作 DOM 但不改数据"的动作,正是命令式逃逸的正当场景。

# 5. 本篇小结与下一步

★ **关键认知**：useRef 三用——**拿 DOM、存不改渲染的可变值、跨渲染持引用**;ref 不改不渲染(对比 state);forwardRef 转发 ref 给子元素,useImperativeHandle 暴露受控方法。命令式是逃生门,能声明式就声明式。

下一步第 11 篇:useContext 与状态提升——跨层传数据、告别 props 逐层穿透(prop drilling)。