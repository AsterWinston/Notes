状态简单用 useState,状态复杂(多个字段、多种更新方式、更新逻辑交织)就该请出 `useReducer`——它把"状态怎么变"集中成一个纯函数。本篇讲清 reducer 模式、dispatch、与 useState 的抉择,以及它配合 Context 做全局状态的形态。

# 1. reducer 模式

## 1.1 reducer 是什么

```jsx
import { useReducer } from 'react'

// reducer:纯函数 (旧的 state, 动作 action) → 新的 state
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 }
    case 'decrement': return { count: state.count - 1 }
    case 'reset':     return { count: 0 }
    default:          return state
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 })
  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+1</button>
      <button onClick={() => dispatch({ type: 'reset' })}>重置</button>
    </div>
  )
}
```

★ **关键认知**：`useReducer(reducer, 初始值)` 返回 `[state, dispatch]`;`dispatch(action)` 把 action 交给纯函数 reducer,算出新 state。⚠ 与 useState 的区别:**useState 的更新逻辑散落在各 setState 调用处,useReducer 把"所有更新方式"集中进 reducer**——这就是它的价值。

## 1.2 action 与 reducer 的约束

★ **关键认知**：reducer 是**纯函数**(同样的 (state, action) 必得同样结果),`action` 通常是一个带 `type` 的对象。⚠ reducer 里**不能有副作用**(发请求、改外部、随机数)——它只是"状态转移表",这让状态变化**可预测、可测试**。

# 2. 何时用 useReducer

## 2.1 复杂状态的标准:更新逻辑交织

```jsx
// 场景:多个字段的状态,且更新方式彼此相关(比如表单、购物车)
function cartReducer(state, action) {
  switch (action.type) {
    case 'add':         // 加商品:同时改列表 + 总数
      return { items: [...state.items, action.item], total: state.total + action.item.price }
    case 'remove':      // 删商品:同时改列表 + 总数
      return { items: state.items.filter(i => i.id !== action.id), ... }
    case 'clear': return { items: [], total: 0 }
  }
}
```

★ **关键认知**：当**多个状态字段需要一起更新**(加购物车要同时改 items 和 total),或**更新逻辑有分支**,useReducer 比一组 useState 更清晰——所有转换集中一处。⚠ 判断标准:**"下一个状态依赖前一个状态 + 有多种转变方式"**,就用 reducer。

## 2.2 useState vs useReducer

| 维度 | useState | useReducer |
|------|----------|------------|
| 状态复杂度 | 简单值 | 对象/多字段/逻辑交织 |
| 更新逻辑位置 | 散布调用处 | 集中 reducer |
| 可测试性 | 一般 | 强(reducer 纯函数) |
| 心智 | 轻 | 略重 |
| 适合 | 计数器、开关 | 表单、购物车、流程 |

★ **关键认知**：**简单就用 useState,复杂也用 useState 直到"更新逻辑散得难受"再换 reducer**——别一上来就 reducer 炫技。⚠ 一个实用信号:**同一个状态有 3 种以上更新方式、或一次更新要动多个字段**,就是 useReducer 的用武之地。

# 3. reducer + Context 做全局状态

## 3.1 组合起来的形态

```jsx
const StateContext = createContext()
const DispatchContext = createContext()

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState)
  return (
    <StateContext.Provider value={state}>       {/* 读状态 */}
      <DispatchContext.Provider value={dispatch}> {/* 发更新 */}
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  )
}
// 消费:const state = useContext(StateContext); const dispatch = useContext(DispatchContext)
```

★ **关键认知**：`useReducer + Context` 能搭出**轻量的全局状态**——state 和 dispatch 分两个 context,读的和写的分开(避免不必要的重渲染)。⚠ 这正是 Redux 的朴素雏形,"读/写分离"的思想也延续到第 18 篇的 Redux Toolkit。

## 3.2 Immer 简化不可变

```jsx
import { useImmerReducer } from 'use-immer'

function reducer(draft, action) {   // draft 是"可写草稿",可直接 draft.items.push(...)
  switch (action.type) {
    case 'add': draft.items.push(action.item); break
  }
}
```

★ **关键认知**：深层不可变更新用手写展开很啰嗦,**Immer** 的 `useImmerReducer` 让你**直接"改"草稿(draft)**、它负责产出新引用。⚠ 这解决的是"reducer 里深层对象的不可变更新写起来痛苦"的问题(呼应第 4 篇 2.3),复杂状态强烈建议配合。

# 4. 本篇小结与下一步

★ **关键认知**：useReducer = 状态更新集中化——`(state, action) → 新 state` 的纯函数;状态复杂、更新方式多、多字段联动时用它;配 Immer 简化不可变;配 Context 可做轻量全局状态。简单状态仍用 useState。

下一步第 13 篇:自定义 Hook——把逻辑封装成 useXxx,React 的复用之道。