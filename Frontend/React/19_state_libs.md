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

## 1.3 store 长在 hook 上:树外也能读写

```jsx
// create 的返回值本身就是 hook,同时挂着 getState/setState/subscribe
useStore.getState().show('保存成功')        // ✓ 拦截器、事件总线、路由守卫里也能读写
const unsub = useStore.subscribe(s => console.log(s.count))

// ⚠ selector 返回"新建对象"会引发无限重渲染:
const list = useStore(s => ({ a: s.a }))    // ✗ 每次调用都是新引用,永远"不等"
const list = useStore(useShallow(s => ({ a: s.a })))  // ✓ 浅比较兜底
```

★ **关键认知**：Zustand 的 store 是**树外单例**——模块导入即可用,组件内外读写同一个入口,这是"不需要 Provider"的技术底气(订阅底层走 React 18 的 `useSyncExternalStore`)。⚠ 两个代价:其一,**selector 返回新对象字面量就破坏"相等则跳过"**,要配 `useShallow`;其二,没有 Provider 意味着**默认全局单例**,单测并行、微前端多实例要显式 `createStore` 造独立实例。

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

## 2.2 派生的真正价值:全局算一次

```jsx
const cartAtom = atom([])
const totalAtom = atom(get => get(cartAtom).reduce((s, i) => s + i.price * i.qty, 0))
// 任何组件 useAtom(totalAtom):不必重复写 reduce、不重复算(结果有缓存)、上游变自动唤醒订阅者
```

★ **关键认知**：`atom(get => …)` 用 **getter 声明依赖,自动构建依赖图**——上游变,派生重算一次并只通知订阅它的人。⚠ 对照"各组件自己 `useMemo`":那是**每处各算各的**;派生原子是**全局一份、共享缓存**。依赖关系一复杂、多处要同一派生值,原子模型就明显占优;反过来,只有两三块粗粒度状态时,Jotai 的"碎"反而不如 Zustand 顺手。

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
| 无跨树共享、低频配置 | 不上库:Context/useReducer |

★ **关键认知**：选型核心是**匹配"状态的性质 + 团队偏好"**——轻全局用 Zustand、碎片化用 Jotai、强约束用 Redux、响应式癖好用 MobX;⚠ 而**服务端数据别进全局 store**(交给第 20 篇 React Query),这条共识能让状态模型清晰一大截。

## 4.1 共同内核:最小订阅

```
传统 Context 的广播:Provider value 一变 → 所有消费者全部重渲染,粗细不分
现代状态库的定向唤醒:store 一变 → 逐个跑订阅者的 selector → 结果没变就不唤醒

  store: { count: 1, theme: 'dark' }
  A 订阅 s => s.count   → count 1→2 → 唤醒 ✓
  B 订阅 s => s.theme   → theme 未变 → 不唤醒 ✓  ← 性能差距的全部来源
```

★ **关键认知**：Redux/Zustand/Jotai 的性能差异常被夸大,它们的**内核其实是同一件事——把订阅粒度从"组件树"降到"数据字段"**,且底层统一踩在 `useSyncExternalStore`(外部 store 接入并发渲染的安全桥)上。⚠ 选型时别比 API 行数,要比两点:**订阅粒度**(selector/atom 谁更贴合你的数据形状)与**派生缓存**(Jotai 内建、Zustand 靠 selector 手写、Redux 靠 reselect);Context 广播最粗,这正是第 11 篇要"拆 context + useMemo"的根因。

## 4.2 先问一句:你真的需要状态库吗

```jsx
function SearchPage() {
  const [kw, setKw] = useState('')                    // ✓ 局部:留在组件
  const { data } = useQuery(['search', kw], fetcher)  // ✓ 服务端数据:交给 Query
  return <ResultList data={data} />
}
// ⚠ 反面:把 kw 塞进全局 store、把接口返回灌进 Redux——
// 状态库存的不是"所有状态",是"确实需要跨树共享的客户端状态"
```

★ **关键认知**：状态库只解决三个具体问题——**跨树共享**(隔五层的组件要同一份值)、**细粒度订阅**(Context 广播太重)、**树外读写**(拦截器/工具函数要碰状态)。⚠ 三条都不沾,就用 useState/useReducer/Context;拿不准时先数"有几个组件同时需要它",答案 ≤2 的几乎都该留在局部。

# 5. 本篇小结与下一步

| 库 | 模型 | 订阅粒度 | 一句话定位 |
|----|------|---------|-----------|
| Redux Toolkit | 集中 + reducer | selector 字段级 | 强约束、可回放、大团队 |
| Zustand | 集中 hook-store | selector 字段级 | 无 Provider,最省事 |
| Jotai | 原子 + 依赖图 | atom 级 | 派生多、状态碎片化 |
| MobX | 响应式对象 | observer 属性级 | 类 Vue 体验,范式另类 |
| Context | 广播 | Provider 级(最粗) | 依赖注入,不是状态管理 |

★ **关键认知**：Zustand(极简全局)、Jotai(原子化)、Recoil(维护模式)、MobX(响应式)各擅胜场;选型看状态性质与团队。切记:全局状态管"客户端共享状态",服务端数据归 React Query。

★ 机制层补一句:现代状态库殊途同归的内核是**最小订阅**——selector/atom 把"变了一块"翻译成"只唤醒关心它的人";没有跨树共享就别上库。

下一步第 20 篇:数据请求 React Query——把"请求、缓存、重试、乐观更新"这件苦差事交给专业选手。
