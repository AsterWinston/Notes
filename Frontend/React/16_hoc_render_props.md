自定义 Hook 是复用"逻辑",复用"带渲染的结构"则有两套历史方案——**高阶组件(HOC)** 与 **render props**。本篇讲清两者怎么用、优劣,以及为什么现代 React 更推崇"**组件 + 组合**"与 Hook 的组合拳。

# 1. 高阶组件 HOC

## 1.1 HOC 是什么

```jsx
// HOC = 一个"接收组件、返回新组件"的函数
function withLoading(WrappedComponent) {
  return function WithLoading({ isLoading, ...props }) {
    if (isLoading) return <Spinner />
    return <WrappedComponent {...props} />   // 透传 props
  }
}

const UserListWithLoading = withLoading(UserList)
// <UserListWithLoading isLoading={true} />
```

★ **关键认知**：**HOC** 是"**组件工厂**"——传入组件,包装一层(加 loading、权限、主题),返回增强后的新组件。⚠ 它是 React 早期复用"跨组件逻辑 + 渲染"的主力,常见于 `connect(mapState)(Component)` 这类 Redux 老 API、路由的 `withRouter`。

## 1.2 HOC 的问题

```jsx
// 多重 HOC 会"层层叠叠",形成"包装地狱"
// withAuth(withTheme(withLoading(Component)))
// 且 HOC 透传 props 易冲突、名字易重
```

★ **关键认知**：HOC 的缺陷——**多层嵌套难读("地狱")、props 透传易冲突、类型(TS)推断麻烦**。⚠ 这三点正是它被更简洁的方案逐步替代的原因;理解 HOC 主要是为了读懂历史代码。

# 2. render props

## 2.1 用函数 prop 共享渲染

```jsx
function Mouse({ render }) {
  const [pos, setPos] = useState({ x: 0, y: 0 })
  useEffect(() => { /* 监听 mousemove 更新 pos */ }, [])
  return render(pos)               // 调用 render prop,把数据交给调用方渲染
}

<Mouse render={({ x, y }) => <div>鼠标在 ({x}, {y})</div>} />
```

★ **关键认知**：**render prop** 是"**传一个函数作为 prop,用它把内部数据交给外部渲染**"。⚠ 它解决 HOC "不知道内部有哪些巧计"的问题——渲染逻辑由调用方完全掌控,HOC 则把增强"包"在外部。两个都曾在"复用带渲染逻辑"上各领风骚。

## 2.2 render props 的问题

```jsx
// 经典问题:render prop 作为内联函数,每次渲染新引用 → 子组件无法用 memo 优化
<Mouse render={({x,y}) => <Cat x={x} y={y} />} />
```

★ **关键认知**：render props 的坑——**内联函数每次渲染都是新引用**,破坏 `React.memo` 的优化;嵌套多也会"金字塔"。⚠ 它的这些弱点,让它同 HOC 一样,现代 React 里逐渐被下面更简洁的方案取代。

# 3. 现代方案:Hook + 组合

## 3.1 逻辑用 Hook,结构用组合

```jsx
// 逻辑:抽成 Hook(复用的是"数据与逻辑")
function useMouse() {
  const [pos, setPos] = useState({ x: 0, y: 0 })
  useEffect(() => { /* 监听 */ }, [])
  return pos
}

// 结构:抽成组件 + 插槽式的 children/props(复用的是"渲染结构")
function MouseTracker({ children }) {
  const pos = useMouse()
  return children(pos)          // 或 <div>{children}</div>
}

<MouseTracker>{({ x, y }) => <div>({x}, {y})</div>}</MouseTracker>
```

★ **关键认知**：现代 React 的推荐是"**逻辑用 Hook 复用,结构用组件 + children 组合**"——两者各管一摊,比 HOC/render props 都更直观。⚠ 这是官方明确的方向:自定义 Hook(第 13 篇)替代"复逻辑",组合(第 3 篇)替代"复结构",HOC/render props 退居"遗留"。

## 3.2 为什么 Hook 更好

| 方案 | 复用什么 | 问题 |
|------|---------|------|
| HOC | 逻辑+结构(包裹) | 嵌套地狱、props 冲突 |
| render props | 逻辑+结构(回调) | 内联函数、金字塔 |
| Hook | 纯逻辑 | 无双(只是函数) |
| 组合 | 纯结构 | 无双 |

★ **关键认知**：Hook 与组合的共同优势——**它们都是"平平无奇"的普通函数/组件**,没有 HOC/render props 那套"包来包去、传来传去"的间接层。⚠ 这也回应了 React 的一个长期演进主线:**从"模式"回归"语言的普通特性"**。

# 4. 遗留 API 的迁移

```jsx
// 老:withRouter(Component) 注入路由
// 新:useParams() / useNavigate() 直接 Hook
// 老:connect(mapState, mapDispatch)(Comp)
// 新:useSelector / useDispatch
```

★ **关键认知**：读懂遗留 API 到 Hook 的迁移脉络——`withRouter → useNavigate/useParams`、`connect → useSelector`。⚠ 但存量项目里 HOC 依然大量存在,学会读懂它是必要条件;新代码则优先 Hook。

# 5. 本篇小结与下一步

★ **关键认知**：HOC(组件工厂)与 render props(函数 prop)是复用"结构+逻辑"的两套历史方案,各有嵌套/引用之弊;现代 React 用 **Hook(逻辑)+ 组合(结构)** 取而代之。新代码 Hook+组合,旧代码需能读懂 HOC。

下一步第 17 篇:React Router——路由配置、嵌套、守卫、懒加载与 loader/action。