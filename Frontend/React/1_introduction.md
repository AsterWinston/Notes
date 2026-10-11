React 的核心就一句话:**UI 是状态的函数**——你声明"状态 X 该渲染成什么视图",数据一变,React 算出最小更新。本篇讲清 React 的定位、组件化/声明式/单向数据流三大思想、JSX 的由来,以及"为什么 React 是库不是框架"。

# 1. React 是什么

## 1.1 一句话定位

React 是**构建用户界面的 JavaScript 库**——它只负责"**把状态映射成视图,并在状态变化时最小化更新 DOM**"。路由、状态管理、数据请求这些,都由生态里的其他库补齐。

```jsx
// 一个组件的本质:状态 → 视图的纯函数
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>     // 输入 props,输出 UI
}
```

★ **关键认知**：React 的定位是**"UI 库"而非"框架"**——它管好"渲染与更新"这一件事,其余靠生态。⚠ 这带来两个结果:**选型自由**(路由/状态想换就换)与**生态碎片化**(一套 React 项目要自己拼齐周边);对照 Vue"渐进式框架"自带 Router/Pinia 的差异。

## 1.2 声明式 vs 命令式

```jsx
// 命令式(手写 DOM):每一步都要自己操作
// element.textContent = count

// 声明式(React):只声明"视图长这样依赖 count"
function Counter() {
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(count + 1)}>{count}</button>
}
```

★ **关键认知**：React 是**声明式**的——你描述"**视图应该长什么样**",React 负责"**状态变了怎么把视图变到位**"。⚠ 与手写 DOM(命令式)相比,你不用关心"先改哪个节点、再插哪个"——这正是 React(和 Vue)共同的核心价值。

# 2. 三大核心思想

## 2.1 组件化

```jsx
function App() {
  return (
    <div>
      <Header />        {/* 组件像标签一样组合 */}
      <Content />
      <Footer />
    </div>
  )
}
```

★ **关键认知**：React 把 UI 拆成**可复用的组件**,组件像积木、像标签一样组合。⚠ 组件名**必须大写开头**(`<Header/>`),小写会被当成 HTML 标签——这是 React 区分"组件 vs 原生标签"的约定。

## 2.2 单向数据流

```
  state(状态在上层)
      │  props 向下传(只读)
      ▼
  子组件(props 不可改,只能通过回调通知上层改)
```

★ **关键认知**：数据**自上而下单向流动**——父组件通过 **props** 传数据给子,子组件**不能改 props**(只能通过回调通知父改)。⚠ 这与 Vue 的 props/emit 是同一套铁律;"谁拥有状态、谁负责改",是排查 React 状态问题的第一把钥匙。

## 2.3 声明式渲染的本质

```jsx
const [name, setName] = useState('World')
const element = <h1>Hello, {name}</h1>   // element 是"这段 UI 的描述"
```

★ **关键认知**：JSX 产生的 **element 是"UI 的描述"(普通对象)**,不是真实 DOM——React 拿它们做 diff,算出最小 DOM 更新(第 30、31 篇)。⚠ 理解"**元素是描述、不是 DOM**",是理解 React 渲染机制的第一步。

# 3. JSX 的由来

## 3.1 JSX 是什么、为什么

```jsx
// 没有 JSX,你得这样写(createElement 调用,可读性差):
// React.createElement('h1', null, 'Hello')

// 有 JSX:像写 HTML 一样写结构,编译成上面的 createElement
const el = <h1 className="title">Hello</h1>
```

★ **关键认知**：JSX 是**JavaScript 的语法扩展**,让你**用"类 HTML"的写法描述 UI**,最终由编译器(Babel/工具链)转成 `createElement` 调用。⚠ JSX **不是模板也不是 HTML**——它是"长得像 HTML 的 JS",所以能用任意 JS 表达式(`{}` 里写什么都行)。

## 3.2 JSX 与 HTML 的差异

```jsx
<div className="box" tabIndex={1}>      {/* class→className */}
  <label htmlFor="name">姓名</label>    {/* for→htmlFor */}
  <p style={{ color: 'red' }}>文本</p>  {/* style 是对象,不是字符串 */}
  {/* 注释要用 {} 包起来 */}
</div>
```

★ **关键认知**：JSX 与 HTML 的常见差异——**`class`→`className`、`for`→`htmlFor`、`style` 是对象**。⚠ 根因是 **JSX 是 JS,要避开 JS 的保留字**(`class` 是关键字,`for` 也是);`style{{}}` 的外层是 JSX 表达式、内层是对象字面量。

# 4. 函数组件与类组件

## 4.1 函数组件(现代主流)

```jsx
function Hello({ name }) {
  return <h1>Hello, {name}</h1>     // 纯函数:props 进,JSX 出
}
```

★ **关键认知**：**函数组件**是当下唯一推荐的写法——一个"**props → UI**"的纯函数,配合 Hooks(第 8 篇起)获得状态与副作用能力。⚠ 类组件已是**遗留写法**(官方不推荐新项目用),但读懂历史代码仍需了解(生命周期对应关系见第 10 篇)。

## 4.2 纯函数的心智

★ **关键认知**：函数组件应**尽量保持"纯"**——同样的 props 应渲染同样的结果,**不要在渲染期间产生副作用**(发请求、直接改外部变量)。⚠ 副作用要放进 `useEffect`(第 8 篇);"渲染是纯的、副作用是显式的",是 React 心智模型的地基。

# 5. 快速上手与工具链

```powershell
# 用 Vite 创建 React + TS 项目(常用、快)
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

★ **关键认知**：React 项目的**开发环境**——现代推荐 **Vite**(快)或 Next.js(全栈框架);ESLint + Prettier 是标配。⚠ 与 Vue 一样,Vite 只转译不检查类型,类型检查靠 `tsc`;JSX 的编译由 Vite 内置 esbuild 完成。

# 6. 本篇小结与下一步

★ **关键认知**：React = UI 库 + 三思想(组件化/声明式/单向数据流)。JSX 是"类 HTML 的 JS",编译成 createElement;元素是"UI 描述"不是 DOM;函数组件 + Hooks 是唯一主流写法。

下一步第 2 篇:JSX 与元素——彻底讲清 JSX 的语法规则、元素与组件的区别。