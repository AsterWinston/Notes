JSX 是 React 的"母语"——你天天写,但它的边界规则、元素与组件的微妙区别,未必真正吃透。本篇把 JSX 的语法规则一次讲清:表达式、属性、渲染数组、元素与组件的分野,以及与 Vue 模板的对照。

# 1. JSX 的表达式

## 1.1 花括号里写 JS

```jsx
const name = 'Alice'
const user = { firstName: 'Alice', lastName: 'Wang' }

function Welcome() {
  return (
    <div>
      {/* {} 里是"单条 JS 表达式",可写变量、运算、函数调用 */}
      <p>{name}</p>
      <p>{1 + 2}</p>
      <p>{user.firstName.toUpperCase()}</p>
      <p>{new Date().getFullYear()}</p>
    </div>
  )
}
```

★ **关键认知**：JSX 里的 `{}` 是"**切回 JavaScript**"的通道,里面是**单条表达式**(能求值的),不是语句。⚠ `{}` 里**不能写语句**(`if`、`for`、`let` 声明都不行)——需要循环/分支,得在 JSX 之外算好,或在表达式里用三元/`.map`。

## 1.2 属性与对象

```jsx
const url = 'https://example.com/logo.png'

<img src={url} alt="logo" />          {/* 属性值用 {} 绑定变量 */}
<div style={{ color: 'red', fontSize: 16 }} />  {/* style 是对象 */}
<div className={isActive ? 'on' : 'off'} />     {/* 三元表达式 */}
```

★ **关键认知**：属性用 `{}` 绑定动态值,`style` 是**对象**(内层 `{}` 是对象字面量)。⚠ 与 Vue 的 `:src`、`:style` 对照:React 没有"指令",**一切动态都是 JS 表达式**——这也让 JSX 比模板更"自由",但也更靠 JS 功底。

## 1.3 JSX 编译成什么:classic vs automatic

```jsx
// 你写的 JSX:
const el = <h1 className="title">Hello {name}</h1>

// classic runtime(早期 babel 默认):编译成 React.createElement,children 是第 3 个起的参数
const classic = React.createElement('h1', { className: 'title' }, 'Hello ', name)

// automatic runtime(React 17+ / 现代工具链默认):编译成 react/jsx-runtime 的 jsx/jsxs
import { jsx as _jsx } from 'react/jsx-runtime'
const automatic = _jsx('h1', { className: 'title', children: ['Hello ', name] })
// automatic 的开启方式:babel preset 默认即可;TS 用 "jsx": "react-jsx"
```

★ **关键认知**：JSX 只是**语法糖**,编译器把标签变成函数调用——老的 **classic runtime** 调 `React.createElement(type, config, ...children)`,新的 **automatic runtime** 调 `jsx()/jsxs()`(单个 children 走 `jsx`,多个走 `jsxs`)。⚠ 差异不止名字:automatic 把 `key` **从 props 里剥离,作为 `jsx()` 的第三个独立参数**(见 2.3),children 收进 props;也因此不用再 `import React`。Babel/SWC/esbuild/ts 全部默认走 automatic,但读懂两种产物仍是排错基本功。

## 1.4 元素的真实长相

```jsx
const el = <h1 className="title">Hi</h1>
// createElement/jsx 返回的是一个普通对象——"元素"就是这份描述:
{
  $$typeof: Symbol.for('react.element'),   // 元素标记(React 19 起为 react.transitional.element)
  type: 'h1',                              // 字符串=宿主标签;函数/class=组件
  props: { className: 'title', children: 'Hi' },
  key: null,                               // key 被单独提取,不在 props 里
  ref: null
}
```

★ **关键认知**：`jsx()` 产出的是**冻结的普通对象(描述)**,不是 DOM 实例。`$$typeof` 是 `Symbol`,**JSON 序列化不了 Symbol**——服务端返回的恶意 JSON 无法伪装成 React 元素注入渲染,这是它的设计动机之一。⚠ 这也解释了 2.2 的"元素不可变":元素只是一次渲染的**快照描述**,React 只拿它去 diff,从不修改;页面上看到的"变化",都是"换了一份新描述"。

# 2. 元素与组件

## 2.1 element vs component

```jsx
// 元素(element):一段"UI 的描述",是普通对象
const element = <h1>Hello</h1>

// 组件(component):返回元素的函数(或类)
function Hello() {
  return <h1>Hello</h1>      // 组件渲染出一个元素
}
```

★ **关键认知**：**组件是"函数",元素是"组件/标签渲染出的描述对象"**——组件返回元素,元素再被 React 转化成 DOM。⚠ 常说"渲染组件",严格说是"**调用组件函数 → 得到元素 → React 渲染元素**";分不清这层,后面 diff/复用都难懂。

## 2.2 元素是不可变的

```jsx
const element = <h1>{new Date().toLocaleTimeString()}</h1>
// element 一旦创建,其 props 不可变(想更新时间,就得重新渲染一个元素)
```

★ **关键认知**：**元素是不可变(immutable)的**——一创建,内容和属性就定死,不能像 DOM 那样 `element.textContent = x` 去改。⚠ React 更新 UI 的方式永远不是"改旧元素",而是"**换新元素**":状态变了 → 生成新元素 → Diff 出差异 → 更新 DOM(第 30、31 篇)。

## 2.3 key 为什么不是 props

```jsx
// classic runtime:key 混在 config 对象里,但 createElement 会把它"摘出来",不下传
React.createElement(Item, { key: 'a', id: 1 })   // Item 的 props 里只有 id,没有 key

// automatic runtime 更彻底:key 是 jsx() 的独立第三参数,与 props 物理分离
const el = _jsx(Item, { id: 1 }, 'a')
```

★ **关键认知**：**`key` 是 React 预留的"协调身份信息",在元素创建时就被提取**——组件永远读不到自己的 key(`props.key` 拿不到),因为它不是给你渲染用的数据,而是给 reconciler 配对用的。⚠ 由此有两个高频现象:(a) `{...item}` 展开的字段里若混着 `key`,classic runtime 会警告 `Item: key prop declared` 并可能让业务字段丢失;(b) 想读"这一行的标识"请另起名(`id={item.key}`)。同理 **`ref` 也不进 props**(React 19 起 ref 可以当普通 prop,但 key 仍不行——口径别记混)。

# 3. JSX 的渲染规则

## 3.1 数组与列表

```jsx
const items = ['a', 'b', 'c']

function List() {
  return (
    <ul>
      {/* .map 生成元素数组,React 会展开渲染 */}
      {items.map(item => <li key={item}>{item}</li>)}
    </ul>
  )
}
```

★ **关键认知**：JSX 里 `{}` 中放**数组**,React 会把数组**展开渲染**(数组里每个元素都渲染出来)。⚠ 列表必须给**每个子元素 `key`**(第 5 篇详解),否则 React 会警告、diff 也可能错位。

## 3.2 null / undefined 不渲染

```jsx
{null}          {/* 什么都不渲染 */}
{undefined}
{false}
{condition && <div>只有真才显示</div>}   {/* 短路求值的条件渲染(第 6 篇) */}
```

★ **关键认知**：JSX 里 `null`/`undefined`/`false` 都是**"不渲染"**——不会产生任何 DOM。⚠ 正因如此,`{condition && <El/>}` 才成立:condition 为 false 时短路返回 false,等于"不渲染";但注意 `0` 会渲染(见第 6 篇的坑)。

## 3.3 Fragment 与"单根"

```jsx
function Row() {
  return (
    <>                     {/* Fragment:不产生真实 DOM 节点 */}
      <td>1</td>
      <td>2</td>
    </>
  )
  // 长形式 <Fragment key={id}>…</Fragment> 才能带 key(用于列表片段)
}
```

★ **关键认知**：组件 return 只能是**一个根**(一个元素/一个数组/一个 Portal),因为"一个 JSX 值"本质是**一个表达式**;`<>…</>`(Fragment)解决"为了包一层而多出不必要的 div"的问题,不产生节点。⚠ 短语法 `<>` **不能带任何属性**,要给 key 必须用长形式 `<Fragment key={…}>`;表格/定义列表/多个兄弟组件这三类"不能插 wrapper div"的场景最常见。

## 3.4 JSX 与 HTML 的属性差异

```jsx
<label htmlFor="name">名字</label>       {/* for → htmlFor(for 是 JS 保留字) */}
<div tabIndex={0} />                    {/* 属性名按 DOM property(驼峰)写 */}
<div dangerouslySetInnerHTML={{ __html: html }} />  {/* 注入 HTML 的唯一逃生门 */}
```

★ **关键认知**：JSX 属性名按 **DOM property**(驼峰)而非 HTML attribute(短横线)写——你绑的是 DOM 节点的属性,不是标签文本。⚠ `dangerouslySetInnerHTML` 故意起一个又长又刺眼的名字,让"注入 HTML"在 code review 里无法被忽略;不可信内容绝不能走它——"React 默认转义,你显式解禁"。

# 4. JSX vs Vue 模板

| 维度 | React JSX | Vue 模板 |
|------|-----------|---------|
| 本质 | JS 语法扩展,编译成函数调用 | 受限的 HTML,编译成 render |
| 编译产物 | `createElement`/`jsx()` 对象描述 | render 函数 + patchFlag 补丁标记 |
| 动态 | `{}` 任意 JS 表达式 | `{{ }}` + 指令 |
| 循环 | `.map` + key | `v-for` |
| 条件 | 三元/&&/提前 return | `v-if/v-show` |
| 可静态优化 | 难(太自由) | 能(static hoist/patchFlag) |

★ **关键认知**：JSX 与模板的根本差异——**JSX 是"自由的 JS"**(编译器难优化但表达力无上限),模板是"受限的 HTML"(编译器可优化但表达力有边界)。⚠ 这正是第 30~31 篇要讲的:Vue 把优化放**编译期**,React 把优化放**运行时**(Fiber/调度),两条路各有代价。

# 5. 本篇小结与下一步

★ **关键认知**：JSX 里 `{}` 是 JS 表达式通道(不能写语句);元素不可变、是"描述";组件是"返回元素的函数";数组展开渲染、null/false 不渲染。JSX = 自由的 JS,与 Vue 模板的"受限可优化"形成对照。

★ 编译视角补一句:JSX 是语法糖,classic 编译成 `React.createElement`、automatic 编译成 `jsx()/jsxs()`,而 `key` 在元素创建时被单独提取、永远不是 props——记住"元素=冻结的描述对象",diff、复用、不可变性全都顺理成章。

下一步第 3 篇:组件与 props——函数组件的边界、props 只读、组合优于继承。
