Redux 不是唯一答案。随着"局部状态、派生状态、原子状态"等需求分化,一批更轻、更专注的状态库崛起——**Zustand、Jotai、Recoil、MobX**。本篇逐一讲清它们的定位与适用场景,并给出一张选型决策表。

# 1. Zustand:极简全局状态

## 1.1 极简 API

```jsx
import { create } from 'zustand'

const useStore = create((set) => ({
  count: 0,
  increment: () => set(state => ({ count: state.count + 1 })),  // 用 set 更新
  reset: () => set({ count: 0 })
}))

function Counter() {
  const count = useStore(state => state.count)     // selector 订阅
  const increment = useStore(state => state.increment)
  return <button onClick={increment}>{count}</button>
}
```

★ **关键认知**：Zustand 用 **`create` 造 store + `useStore(selector)` 订阅**,不要求 Provider、不要求 reducer,极简。⚠ 它最大的卖点是"**不套 Provider**"与"**选择器细粒度订阅**"(只订阅用到的字段,避免无谓重渲染)——很多人就从 Redux 迁到 Zustand 图这个轻。

## 1.2 与 Redux 的对比

| 维度 | Zustand | Redux Toolkit |
|------|---------|---------------|
| Provider | 不需要 | 需要 |
| 样板 | 极少 | 中等 |
| 概念 | store+set | slice+dispatch |
| 学习 | 快 | 略重 |
| 适用 | 中小全局状态 | 大型/强约束团队 |

★ **关键认知**：Zustand 是"**够用就好**"的全局状态——APIs 少、上手快、性能好;Redux 是"**大团队的强约束与可预测**"(DevTools、规范、时间旅行)。⚠ 选择看**团队规模与是否需要强约束**,而非技术高下。

# 2. Jotai:原子化状态

## 2.1 atom 与派生

```jsx
import { atom, useAtom } from 'jotai'

const countAtom = atom(0)                            // 一个"原子"即一块状态
const doubleAtom = atom(get => get(countAtom) * 2)   // 派生原子:依赖别的原子

function Counter() {
  const [count, setCount] = useAtom(countAtom)
  const [double] = useAtom(doubleAtom)
  return <button onClick={() => setCount(c => c + 1)}>{count} / {double}</button>
}
```

★ **关键认知**：Jotai 的**原子(atom)**模型——每个原子是一小块可独立订阅的状态,原子还能**派生**(依赖别的原子)。⚠ 它比 Zustand 更"细粒度"(状态拆成一堆原子),适合**状态分散、各自独立**、又有派生关系的场景。

# 3. Recoil 与 MobX

## 3.1 Recoil(已进入维护模式)

```jsx
import { atom, useRecoilState, selector } from 'recoil'

const countState = atom({ key: 'count', default: 0 })      // 原子
const doubleState = selector({ key: 'double', get: ({ get }) => get(countState) * 2 })  // 派生
```

★ **关键认知**：Recoil 与 Jotai 同是"原子模型",由 Meta 出品。⚠ **Recoil 已进入维护模式**(Meta 停止大投入),新项目**优先 Jotai**(同类思想、更活跃);了解 Recoil 主要是读旧代码/对比。

## 3.2 MobX:响应式,非"React 思维"

```jsx
import { makeAutoObservable } from 'mobx'
import { observer } from 'mobx-react-lite'

class CounterStore {
  count = 0
  constructor() { makeAutoObservable(this) }   // 属性自动变响应式
  increment() { this.count++ }
}

const Counter = observer(function Counter({ store }) {
  return <button onClick={() => store.increment()}>{store.count}</button>  // 观察者组件
})
```

★ **关键认知**：MobX 走**响应式**路线(类 Vue)——状态是"可观察的",组件包 `observer` 后**自动追踪依赖**、直接改 state。⚠ 它与 React 官方"单向 + 不可变 + 依赖数组"的心智**完全不同**(更接近 Vue 的 Proxy 思维);会用因人喜欢的开发体验,但**与 React 生态的主流范式有隔阂**。

# 4. 选型决策表

| 场景 | 推荐 |
|------|------|
| 少量全局状态、要轻 | Zustand |
| 状态细碎、多派生关系 | Jotai |
| 大型团队、强约束/DevTools | Redux Toolkit |
| 偏爱响应式、类 Vue 体验 | MobX |
| 服务端数据(请求/缓存) | React Query(第 20 篇) |

★ **关键认知**：选型核心是**匹配"状态的性质 + 团队偏好"**——轻全局用 Zustand、碎片化用 Jotai、强约束用 Redux、响应式癖好用 MobX;⚠ 而**服务端数据别进全局 store**(交给第 20 篇 React Query),这条共识能让状态模型清晰一大截。

# 5. 本篇小结与下一步

★ **关键认知**：Zustand(极简全局)、Jotai(原子化)、Recoil(维护模式)、MobX(响应式)各擅胜场;选型看状态性质与团队。切记:全局状态管"客户端共享状态",服务端数据归 React Query。

下一步第 20 篇:数据请求 React Query——把"请求、缓存、重试、乐观更新"这件苦差事交给专业选手。