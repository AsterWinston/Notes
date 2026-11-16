React 默认"父组件重渲染,子组件也跟着重渲染",优化就是**让不该重渲染的跳过**。本篇讲清 React 性能的四件套——`React.memo`(跳过重渲染)、不可变数据、**列表虚拟化**(长列表)、**Profiler**(测量),以及"先测量再优化"的纪律。

# 1. React.memo

## 1.1 跳过重渲染

```jsx
import { memo } from 'react'

const Child = memo(function Child({ name }) {
  console.log('渲染 Child')
  return <div>{name}</div>
})
// memo 后:props 没变(浅比较),Child 就不重渲染
```

★ **关键认知**：`memo(Component)` 包一层后,React 会**浅比较 props**,props 没变就**跳过该组件的重渲染**。⚠ 默认 React 是"父渲染子就渲染",一层不做比较;memo 加上"props 没变就复用上次结果"的判断——这是最基础的组件级优化。

## 1.2 与 useCallback 配合

```jsx
// memo 生效的前提:props 引用稳定(否则每次渲染 props 都是新的,memo 形同虚设)
const onClick = useCallback(() => {...}, [])   // 稳定函数引用
<List onClick={onClick} />                      // 配合 memo 的 List 才真能跳过
```

★ **关键认知**：memo 只对"**props 没变**"有效,而函数/对象 props 每次渲染是新引用,会让 memo 失效——所以要配 `useCallback`/`useMemo`(第 9 篇)稳住引用。⚠ "**memo 卡住组件 + useCallback 卡住引用**"是性能优化的黄金搭档,二者缺一,另一孤立无援。

# 2. 不可变数据的价值

## 2.1 引用比较的基础

```jsx
// memo、PureComponent、useEffect 依赖,都靠"引用相等"判断"变没变"
setTodos([...todos, newTodo])   // 新数组引用 → 能判断出"变了"
// todos.push(x)                // 原地改 → 引用没变 → 判断不出,优化全失效
```

★ **关键认知**：React 的所有"变化检测"都基于**不可变**(引用比较)——`[...arr, x]` 产生新引用,才能被 memo/依赖识别出"变了"。⚠ 第 4 篇的"不可变更新"在这里与性能打通:**原地改对象/数组,memo 和依赖数组都会失效**。

# 3. 长列表虚拟化

## 3.1 只渲染可见项

```jsx
// 10000 条数据全渲染 = 10000 个 DOM,卡死
// 虚拟化:只渲染"视口内"的几十条,滚动时动态换
// 用 react-window / react-virtual 等库
import { FixedSizeList } from 'react-window'

<FixedSizeList
  height={400}
  itemCount={items.length}
  itemSize={40}
>
  {({ index, style }) => (
    <div style={style}>{items[index]}</div>   // 只为可见项渲染
  )}
</FixedSizeList>
```

★ **关键认知**：**虚拟化(virtualization)** 只渲染**视口可见的那一小部分**,滚动时复用/替换——万级列表也能丝滑。⚠ 这是"数据量巨大"时的王牌优化,DOM 节点数从"全部"降到"可见数",性能量级之差。

## 3.2 何时需要

★ **关键认知**：普通几十上百条列表**不需要**虚拟化;到**上千上万条**或"每条渲染很重"时才值得。⚠ 虚拟化引入库和复杂度,别过早优化;先确认瓶颈在"列表长度"再上。

# 4. Profiler 性能测量

## 4.1 React DevTools Profiler

```jsx
import { Profiler } from 'react'

<Profiler id="App" onRender={(id, phase, actualDuration) => {
  console.log(id, phase, actualDuration)   // 每次渲染的耗时
}}>
  <App />
</Profiler>
```

★ **关键认知**：**React DevTools 的 Profiler 面板**能记录每个组件**渲染耗时、重渲染次数、为什么渲染**,是定位"哪个组件拖慢了"的显微镜。⚠ 优化前**先用 Profiler 找到真正的慢点**——凭直觉优化常扑空;它还会告诉你"这个组件为什么重渲染了"(props 变/Hook 变/父渲染)。

## 4.2 先测量再优化

★ **关键认知**：性能优化的第一纪律是"**先测量,找到瓶颈,再动手**"——别凭感觉上一堆 memo。⚠ 优化要"对症":渲染次数多→memo+useCallback;列表长→虚拟化;计算贵→useMemo;体积大→代码分割。Profiler 帮你判断"病在哪"。

# 5. 性能优化清单

| 手段 | 针对 | 收益 |
|------|------|------|
| React.memo + useCallback | 无谓重渲染 | 跳过不变组件 |
| 不可变数据 | 引用比较失效 | 让优化判定正确 |
| 列表虚拟化 | 长列表 DOM 过多 | 节点数巨降 |
| useMemo | 昂贵计算 | 跳过重算 |
| 代码分割(第 23 篇) | 首屏体积 | 首屏加快 |
| Profiler | 找不到瓶颈 | 精准定位 |

★ **关键认知**：五件套的分工——**memo 跳渲染、虚拟化降 DOM、useMemo 跳计算、代码分割降体积、Profiler 找瓶颈**。⚠ 优化永远"测量在前、手段在后",且**优先大道**(分割、虚拟化),再**抠细节**(memo/useMemo)。

# 6. 本篇小结与下一步

★ **关键认知**：React.memo 靠浅比较跳过重渲染(需 useCallback 稳住引用);不可变是引用比较的基础;长列表用虚拟化;Profiler 先测量。核心:让"不该重渲染的不渲染、不该算的不算、不该下载的不下载"。

下一步第 25 篇:错误边界与并发特性——ErrorBoundary 兜住错误,Suspense/useTransition 的并发能力。