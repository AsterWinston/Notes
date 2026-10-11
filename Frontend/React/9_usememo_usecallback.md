`useMemo` 缓存计算结果,`useCallback` 缓存函数本身——它们都是"记忆化",但也是最容易被**滥用**的 Hook。本篇讲清两者的区别、真正该用的场景(昂贵计算、引用稳定),以及"为什么别到处加 useMemo"。

# 1. useMemo 缓存值

## 1.1 基本用法

```jsx
import { useMemo } from 'react'

function List({ items, filter }) {
  // 只依赖 items/filter,依赖不变就返回缓存的数组(不重算)
  const filtered = useMemo(
    () => items.filter(i => i.name.includes(filter)),
    [items, filter]
  )
  return <ul>{filtered.map(i => <li key={i.id}>{i.name}</li>)}</ul>
}
```

★ **关键认知**：`useMemo(fn, deps)` 把"**计算结果**"缓存起来——**依赖不变,就不重新计算**,直接返回上次的值。⚠ 用途是**跳过昂贵的重算**(大数组过滤/排序、复杂派生),依赖数组的意义与 useEffect 一致。

## 1.2 与直接计算的对比

```jsx
// 不用 useMemo:每次渲染都重算
const filtered = items.filter(heavy)

// 用 useMemo:仅 items/filter 变才重算
const filtered = useMemo(() => items.filter(heavy), [items, filter])
```

★ **关键认知**：useMemo 的价值在于**"计算本身昂贵"**——`filter` 一个万亿数组、复杂排序、递归求值才值得。⚠ **轻量计算用 useMemo 是负优化**(多了依赖比较和缓存开销),别为 `1+1` 也包一层。

# 2. useCallback 缓存函数

## 2.1 基本用法

```jsx
import { useCallback } from 'react'

function Parent() {
  // 缓存 handleClick:依赖不变,返回同一个函数引用
  const handleClick = useCallback(() => {
    doSomething()
  }, [])
  return <Child onClick={handleClick} />
}
```

★ **关键认知**：`useCallback(fn, deps)` 缓存**函数本身**——依赖不变,返回**同一个函数引用**(而非每次渲染新建)。⚠ 它等价于 `useMemo(() => fn, deps)` 的语法糖;目的是**保持函数引用稳定**。

## 2.2 为什么要稳住函数引用

```jsx
const Child = memo(function Child({ onClick }) { ... })   // React.memo 只在 props 变时重渲染

// 若 handleClick 每次渲染都是新引用 → Child 的 props 变了 → memo 失效 → 每次重渲染
const handleClick = useCallback(() => {...}, [])  // 稳住引用,memo 才生效
```

★ **关键认知**：`useCallback` 的真正价值是**配合 `React.memo`**(第 24 篇)——函数引用稳定,子组件才能"跳过无意义重渲染"。⚠ **单独用 useCallback(不配 memo)几乎无用**——它本身不减少渲染,只是稳引用;这是它被误解滥用的根源。

# 3. useMemo vs useCallback

| 维度 | useMemo | useCallback |
|------|---------|-------------|
| 缓存什么 | 计算结果(值) | 函数(引用) |
| 等价 | — | useMemo 的语法糖 |
| 场景 | 昂贵计算 | 稳函数引用给 memo/依赖 |

★ **关键认知**：**useMemo 缓存值、useCallback 缓存函数**;`useCallback(fn, deps)` === `useMemo(() => fn, deps)`。⚠ 记这个等价,能帮你判断"该用哪个":要"算出的那个值"就用 useMemo,要"那个函数引用"就用 useCallback。

# 4. 何时真正该用

## 4.1 该用的场景

```jsx
// 场景一:昂贵计算(大列表排序/过滤)
const sorted = useMemo(() => [...list].sort(heavyCompare), [list])

// 场景二:配合 memo,给子组件传稳定引用
const onSelect = useCallback(id => {...}, [])
<Row onSelect={onSelect} />
```

★ **关键认知**：两个**真正**该用的场景——**① 昂贵的派生计算;② 配合 React.memo 给子组件稳定引用**(或函数出现在其他 Hook 的依赖里需稳)。⚠ 除此外,多数 useMemo/useCallback 都是**过早优化**。

## 4.2 不该用的场景

```jsx
// ✗ 过度优化:计算很轻,还要额外开销
const value = useMemo(() => count + 1, [count])

// ✗ 配不上 memo,白写
const fn = useCallback(() => {...}, [])   // 没任何 memo 子组件用它,稳引用无意义
```

★ **关键认知**：**先写直白的代码,测出性能问题了再优化**——React 官方明确说"useMemo 不是必须的,是优化手段"。⚠ 大量无谓的 useMemo/useCallback 会让代码更难读、还可能更慢(依赖比较的开销)。

## 4.3 决策流程

```
这个计算昂贵吗? ──否──▶ 直接算,不用 useMemo
     │是
     ▼
依赖稳定吗? ──否──▶ 用了也白用(每次依赖都变)
     │是
     ▼
  用 useMemo 缓存
```

★ **关键认知**：判断是否用 useMemo 的三问——**"昂贵吗?依赖稳定吗?"** 都满足才值得;useCallback 则先问"**有 memo 子组件需要稳定引用吗?**"。⚠ 拿不准就不加,函数式编程的简单代码好过过早的缓存。

# 5. 本篇小结与下一步

★ **关键认知**：useMemo 缓存值(昂贵计算)、useCallback 缓存函数(稳引用给 memo);两者都是"优化手段"不是"必需品"。先平凡后优化,别到处包 useMemo/useCallback。

下一步第 10 篇:useRef 与命令式——ref 拿 DOM、保存可变值、forwardRef 与 useImperativeHandle。