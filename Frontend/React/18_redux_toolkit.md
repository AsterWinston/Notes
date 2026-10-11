当状态需要跨组件、跨页面共享,Context 就力不从心了——这时请出**集中式状态管理**。Redux 是 React 生态最经典的选择,而 **Redux Toolkit(RTK)** 则是官方推荐的现代写法。本篇讲清 store/slice/dispatch/selector,以及与 Vue 的 Pinia 对照。

# 1. Redux 三原则

## 1.1 单一数据源 + 只读 + 纯函数

```jsx
// 1. 单一数据源:整个应用一个 store
// 2. state 只读:只能通过 dispatch(action) 改
// 3. 用纯函数 reducer 计算新 state
```

★ **关键认知**：Redux 的三原则——**单一 store、state 只读(仅 action 可改)、纯函数 reducer**。⚠ 这套原则的价值是**可预测、可调试**(时间旅行、回放、DevTools),代价是**样板代码多**——这正是 RTK 要解决的问题。

# 2. Redux Toolkit 的写法

## 2.1 createSlice 定义

```jsx
import { createSlice, configureStore } from '@reduxjs/toolkit'

// createSlice:一个"切片"= 一段状态 + reducer + 自动生成 action
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    incremented(state) { state.value += 1 },   // 可直接"改"(内部用 Immer)
    incrementedBy(state, action) { state.value += action.payload }
  }
})

export const { incremented, incrementedBy } = counterSlice.actions   // 自动生成 action creator
export const counterReducer = counterSlice.reducer

// configureStore:组装整个 store
export const store = configureStore({
  reducer: { counter: counterReducer }
})
```

★ **关键认知**：RTK 用 `createSlice` 一段段定义状态,**`reducers` 里能直接"改" state**(内部用 Immer,自动处理不可变),**自动生成 action creator**。⚠ 对照老 Redux(手写 action type + reducer + 手动不可变),RTK 省掉大量样板——`state.value += 1` 直观,背后 Immer 保证不可变。

## 2.2 组件里 dispatch / selector

```jsx
import { useDispatch, useSelector } from 'react-redux'
import { incremented, incrementedBy } from './counterSlice'

function Counter() {
  const value = useSelector(state => state.counter.value)   // 读状态(selector)
  const dispatch = useDispatch()                             // 拿 dispatch

  return (
    <button onClick={() => dispatch(incremented())}>
      {value}
    </button>
  )
}
```

★ **关键认知**：组件里 **`useSelector` 读、`useDispatch` 发**;`dispatch(actionCreator(...))` 触发更新。⚠ 这是第 16 篇提到的"`connect` 老 API → `useSelector/useDispatch` 新 Hook"的落地——selector 每次拿到的是"你要的那部分状态",RTK 会自动浅比较避免无谓重渲染。

## 2.3 Provider

```jsx
import { Provider } from 'react-redux'
import { store } from './store'

<Provider store={store}>   {/* 应用根部包裹 */}
  <App />
</Provider>
```

★ **关键认知**：用 `<Provider store={store}>` 把 store 注入应用(底层就是 Context)。⚠ 与 React Router 的 RouterProvider 一样,都是"根部包裹一次"的注入模式。

# 3. 与 Pinia 对照

| 维度 | Redux Toolkit | Pinia |
|------|---------------|-------|
| store 结构 | 单一 root + slice | 多个独立 store |
| 改状态 | reducer(经 Immer) | 直接改 state(actions) |
| action | dispatch(action) | 直接调 store 方法 |
| 不可变 | Immer 内部处理 | 响应式自动 |
| 样板 | RTK 已大幅减少 | 更少 |
| 生态 | React 事实标准 | Vue 官方默认 |

★ **关键认知**：两者都收敛到"**集中式、可预测、类型友好**"——RTK 靠 Immer 简化不可变,Pinia 靠 Vue 响应式直接改。⚠ 一个深层差异:Pinia 直接改 state 是**响应式系统**给的便利,Redux 直接改是 **Immer** 在"造假不可变";本质上殊途同归,都是"让你随手写、底层保证正确"。

# 4. 异步与副作用

## 4.1 createAsyncThunk

```jsx
import { createAsyncThunk } from '@reduxjs/toolkit'

const fetchUser = createAsyncThunk('user/fetch', async (id) => {
  return await (await fetch(`/api/user/${id}`)).json()
})

// 在 slice 里用 extraReducers 处理请求的三种状态
// fetchUser.pending / fetchUser.fulfilled / fetchUser.rejected
```

★ **关键认知**：异步用 `createAsyncThunk`,自动管理 **pending/fulfilled/rejected** 三态,在 `extraReducers` 里处理。⚠ 这是 Redux 处理"请求 + 状态"的正规姿势;但"请求缓存、重试、乐观更新"这类复杂需求,社区更倾向交给第 20 篇的 React Query(它专精此道)。

## 4.2 Redux 该装多大

★ **关键认知**：Redux 适合"**大量共享、跨模块、需可预测**"的全局状态(用户、购物车、配置);⚠ **局部状态仍留组件内**(useState),**服务端数据优先 React Query**——"Redux 存一切"是老误区,现代实践是**各司其职**。

# 5. 本篇小结与下一步

★ **关键认知**：RTK = createSlice(状态+reducer+action 自动生成)+ configureStore + useDispatch/useSelector;Immer 让你"直接改"却保不可变。与 Pinia 殊途同归,都做"集中、可预测、类型友好"。

下一步第 19 篇:其他状态库选型——Zustand/Jotai/Recoil/MobX,轻量方案如何弯道超车。