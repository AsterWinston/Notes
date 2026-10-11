组件是 React 的积木,props 是组件间的"唯一数据入口"。本篇讲清函数组件的写法、props 的只读与解构、children 这个特殊 prop,以及 React 的**组合优于继承**哲学——它决定了你组织组件的心智方式。

# 1. 组件与 props 基础

## 1.1 props 从哪来

```jsx
function Card(props) {
  return (
    <div className="card">
      <h2>{props.title}</h2>
      <p>{props.content}</p>
    </div>
  )
}

// 调用处:属性(attribute)就是 props
<Card title="标题" content="内容" />
```

★ **关键认知**：**props 就是组件的"参数"**——调用处写在标签上的属性,打包成一个对象传给组件函数。⚠ props 的命名是**单向约定**:调用处有 `title` 属性,组件里就读 `props.title`,没有"全局注册"这回事(对照 Vue 的 defineProps 显式声明,React 不强制声明)。

## 1.2 解构与默认值

```jsx
function Card({ title, content = '默认内容' }) {   // 解构 + 默认值
  return <div><h2>{title}</h2><p>{content}</p></div>
}

function Avatar({ size = 48 }) {   // 常用默认值写法
  return <img width={size} />
}
```

★ **关键认知**：解构 props(含默认值)是主流写法;`size = 48` 是**参数默认值**。⚠ 默认值只在"**该 prop 是 undefined**"时生效(传 `null` 不会触发默认)——这与 JS 参数默认值语义一致。

# 2. props 是只读的

## 2.1 不可修改 props

```jsx
function Badge({ count }) {
  // ✗ 绝对不要改 props!
  // count = count + 1          // 语法就报错(参数是局部变量)
  // props.count = 99          // 运行时不安全,违背单向数据流
  return <span>{count}</span>
}
```

★ **关键认知**：props 必须当**只读**——组件**绝不能修改 props**(想"改变"它,是父组件的责任)。⚠ 这是"单向数据流"的铁律:数据只能从父流向子,子要影响父只能通过**回调**或**状态提升**(第 11 篇)。违反它有双重后果:React 报错(严格模式)且状态来源无法追踪。

## 2.2 通过回调通知父组件

```jsx
function Child({ onIncrement }) {
  return <button onClick={onIncrement}>+1</button>   // 调用父传的回调
}

function Parent() {
  const [count, setCount] = useState(0)
  return <Child onIncrement={() => setCount(count + 1)} />  // 状态在父,改也是父改
}
```

★ **关键认知**：props 向下传数据,回调向上传"意图"——父把 `setCount` 包成回调传给子,子是**间接"请"父改状态**。⚠ 状态永远**归属它真正的主人**(父组件),子组件只是"受控的视图"——这个"受控"思维在表单(第 7 篇)里会再次出现。

# 3. children 特殊 prop

## 3.1 嵌套内容即 children

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>   // 渲染标签之间的内容
}

<Card>
  <h2>标题</h2>                    {/* 标签之间的东西就是 children */}
  <p>正文内容</p>
</Card>
```

★ **关键认知**：`children` 是**隐式的特殊 prop**——写在组件标签**之间的内容**会作为 children 传进来。⚠ 它是 React "组合"思想的载体,等价于 Vue 的插槽(slot);`<Card>...</Card>` 里的 JSX 就是 children。

## 3.2 children 与组合

```jsx
function Layout({ header, children }) {
  return (
    <div>
      <header>{header}</header>       {/* 命名内容 */}
      <main>{children}</main>         {/* 主内容 */}
    </div>
  )
}

<Layout header={<h1>站点标题</h1>}>
  <p>这是主内容区</p>
</Layout>
```

★ **关键认知**：除了 `children`,你还可自定义 prop 传 JSX(如 `header={...}`),实现"**命名插槽**"。⚠ 这是 React 里"父组件向子组件注入内容"的标准姿势——比传一堆配置项更灵活、更声明式。

# 4. 组合优于继承

## 4.1 没有"组件继承"

```jsx
// React 里没有"class BaseComponent extends React.Component 然后 extends BaseComponent"
// 组件复用靠"组合",不靠"继承"
function Dialog({ children }) {
  return <div className="dialog">{children}</div>
}
function ConfirmDialog() {
  return <Dialog><p>确定吗?</p><button>确定</button></Dialog>  // 组合复用 Dialog
}
```

★ **关键认知**：React **不鼓励组件继承**——复用靠**组合**(把一个组件嵌进另一个)。⚠ 官方明确:"组件之间复用代码,用 props + 组合;逻辑复用用 Hook(第 13 篇)"。这跟面向对象那套"继承/重写"的心智彻底不同。

## 4.2 特化(specialization)

```jsx
// 用"特定组件包裹通用组件"实现特化
function SpecialCard(props) {
  return <Card {...props} variant="featured" />   // 透传 props + 固化 variant
}
```

★ **关键认知**：**特化**=用一个"更具体"的组件包住"更通用"的组件,通过 `{...props}` 透传 + 固化某些 prop。⚠ `{...spread}` 透传在"包装组件"场景高频出现,但**别滥用**(透传太模糊会让数据流难追踪),能显式写就显式写。

# 5. 本篇小结与下一步

★ **关键认知**：props 是组件的只读参数(解构+默认值);children 是标签间的隐式 prop;React 的哲学是**组合优于继承**——复用结构靠组合,复用逻辑靠 Hook。单向数据流:传数据用 props,改数据靠回调。

下一步第 4 篇:状态 useState——引入状态,并直面"不可变更新""批处理""闭包陷阱"三大核心。