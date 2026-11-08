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

## 1.3 包装发生在哪:模块顶层,不是组件体内

```jsx
// ⚠ 反面教材:在组件体内调用 HOC —— 每次渲染都造出"新组件类型"
function Bad({ data }) {
  const Enhanced = withLoading(Row)     // 每次 Bad 渲染,Enhanced 的引用都变
  return <Enhanced data={data} />       // type 变 → Row 整个卸载重建,状态丢失
}

// ✓ 正确:在模块顶层包装一次,类型引用恒定
const Enhanced = withLoading(Row)
function Good({ data }) {
  return <Enhanced data={data} />
}
```

```text
HOC 与 class 继承对照(它是"组合"不是"继承"):
  class Enhanced extends Base    ✗ React 不推荐
  Enhanced = props => <Base …/>  ✓ 包一层"兄弟组件",黑盒增强
  —— 复用靠"引用谁"而不是"继承谁",呼应第 3 篇"组合优于继承"
```

★ **关键认知**：HOC 返回的是**新的组件类型**,`withLoading(A) !== withLoading(A)`——包错位置(组件体内)等于每次渲染"换了个组件",子树整个**卸载重建**(对照第 6 篇"换 key 即重置")。⚠ 所以 HOC/lazy 调用永远放**模块顶层**;这是"我的表单为什么一改值就失焦"的经典答案之一。

## 1.4 三个具体老毛病:命名冲突、静态方法、ref

```jsx
// (a) props 名字冲突:内层注入的 user 被外层同名 prop 无声覆盖
withAuth(withTheme(UserProfile))   // 两边都注入 user?后者赢,调试半天

// (b) 静态方法丢失:新组件不继承原组件的静态成员,要靠 hoist 库提升
import hoistNonReactStatics from 'hoist-non-react-statics'
const Enhanced = hoistNonReactStatics(withAuth(Wrapped), Wrapped)

// (c) ref 断链:包装层的 ref 指向 HOC 自己,拿不到真实组件
const r = useRef(); <Enhanced ref={r} />   // r.current ≠ User 实例
```

★ **关键认知**：HOC 的三大顽疾各有标准解法——**冲突靠命名空间约定/显式挑选 prop**、**静态成员靠 hoist-non-react-statics 提升**、**ref 靠内层 forwardRef 转发**;再配 `displayName = 'withAuth(Wrapped)'` 让 DevTools 认得出。⚠ 这些不是历史笑话:今天接内网 `withPermission`、老版 `connect`/`withRouter` 依然会撞;TS 下还要 `Omit` 掉 HOC 注入的 prop 再导出类型,否则调用方"该传的不用传"。

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

## 2.3 换个角度看:它就是"延迟执行的插槽"

```jsx
// render props 没有魔法:props 传 JSX 是第 3 篇的 header={...},
// 传"返回 JSX 的函数"只是多一步延迟——内部数据就绪时才调用
function DataPanel({ fallback, children }) {   // children 当 render prop 用
  const data = useFetch('/api/stats')
  if (!data) return fallback
  return children(data)          // 把"内部数据"倒给"外部模板"
}
```

★ **关键认知**：render props 的本质是**控制反转**——容器管"何时/有何数据",调用方管"怎么渲染";与 Vue 作用域插槽(`#default="{ x }"`)是同一机制的两种拼写。⚠ 它并没有"消亡":`<Route render={…}>`(旧版)、各种 `<X>{ctx => …}</X>` 组件库 API 全是它——"children 是函数就直接调用",看懂这行约定比背概念有用。

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

## 3.3 hooks 时代为何仍需 HOC

```jsx
// 场景一:给"拿不到 hook 的位置"套一层 —— hook 必须在函数组件顶层调
// 路由表要求"组件类型"、批量给页面套 ErrorBoundary/AuthGate 时,HOC 是最小胶水
const withAuthGate = Page => props => <AuthGate><Page {...props} /></AuthGate>
// 场景二:第三方 API 长这样(connect/withRouter),你得会读、会"反包装"迁移
```

★ **关键认知**：HOC 的保留地是**"对未知组件做包装"**——`withX(任意组件)` 的输入是别人写的组件,你无法把 hook"塞进"别人的函数体,只能在外面再套一个能调 hook 的组件。⚠ hooks 取代的是"**同一段有状态逻辑要在 N 个组件里各写一遍**"的场景,取代不了"给一棵不知道形状的树统一加行为"的场景;二者是分工,不是前后浪。

## 3.4 forwardRef 时代的定位

```jsx
// React 19 起 ref 可当普通 prop,forwardRef 正式退役(旧写法仍可用):
function FancyInput({ ref }) {              // 19:props 里直接解构 ref
  return <input ref={ref} className="fancy" />
}
// 18 及以前必须:
const FancyInput = forwardRef((props, ref) => <input ref={ref} {...props} />)

// 对 HOC 的意义:包装组件"把 ref 转给内层"从特殊能力退化为普通 props 透传
function withFocusRing(Wrapped) {
  return function WithFocusRing({ ref, ...props }) {
    return <Wrapped ref={ref} {...props} />
  }
}
```

★ **关键认知**：HOC 三顽疾里 ref 断链最顽固,因为 18 之前 **ref 根本不在 props 里**(第 2 篇),`{...props}` 顺手带不动,必须 forwardRef 特判。React 19 把 ref 收编为普通 prop,这层间接魔法随之消失。⚠ 但 **key 至今仍不是 props**——forwardRef 退役 ≠ key 也能转发;读 18 时代组件库源码,`React.memo(forwardRef(…))` 的套娃仍铺天盖地。

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

★ 机制层再补三句:HOC 是"组合不是继承",必须包在组件外;三大老毛病(命名冲突/静态提升/ref 断链)各有标准解法,其中 ref 一条已被 React 19 语言级抹平;"给未知组件套一层"是 HOC 在 hooks 时代不可替代的位置。

下一步第 17 篇:React Router——路由配置、嵌套、守卫、懒加载与 loader/action。
