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

# 4. JSX vs Vue 模板

| 维度 | React JSX | Vue 模板 |
|------|-----------|---------|
| 本质 | JS 语法扩展,编译成函数调用 | 受限的 HTML,编译成 render |
| 动态 | `{}` 任意 JS 表达式 | `{{ }}` + 指令 |
| 循环 | `.map` + key | `v-for` |
| 条件 | 三元/&&/提前 return | `v-if/v-show` |
| 可静态优化 | 难(太自由) | 能(static hoist/patchFlag) |

★ **关键认知**：JSX 与模板的根本差异——**JSX 是"自由的 JS"**(编译器难优化但表达力无上限),模板是"受限的 HTML"(编译器可优化但表达力有边界)。⚠ 这正是第 30~31 篇要讲的:Vue 把优化放**编译期**,React 把优化放**运行时**(Fiber/调度),两条路各有代价。

# 5. 本篇小结与下一步

★ **关键认知**：JSX 里 `{}` 是 JS 表达式通道(不能写语句);元素不可变、是"描述";组件是"返回元素的函数";数组展开渲染、null/false 不渲染。JSX = 自由的 JS,与 Vue 模板的"受限可优化"形成对照。

下一步第 3 篇:组件与 props——函数组件的边界、props 只读、组合优于继承。