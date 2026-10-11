表单是 React 状态的"回流口"——用户输入要同步进 state。围绕它诞生了**受控组件**与**非受控组件**两个核心概念。本篇讲清两者的区别与取舍、各种表单控件(文本/复选/单选/下拉)的状态绑定,以及为什么 React 推荐受控。

# 1. 受控与不受控

## 1.1 受控组件

```jsx
function Input() {
  const [value, setValue] = useState('')
  return (
    <input
      value={value}                                  // 值来自 state(单一数据源)
      onChange={e => setValue(e.target.value)}       // 变化同步回 state
    />
  )
}
```

★ **关键认知**：**受控组件(controlled)**——表单值**由 React state 完全掌控**("唯一数据源"是 state),用户输入经 `onChange` 写回 state。⚠ "受控"的意思是:**你输入 → 触发 setState → 重渲染 → 输入框显示新值**,每一步都经过 React;也因此你能随时改值、校验、格式化。

## 1.2 非受控组件

```jsx
import { useRef } from 'react'

function Input() {
  const ref = useRef(null)                       // 用 ref 直接拿 DOM
  function submit() {
    console.log(ref.current.value)               // 提交时才读值
  }
  return <input ref={ref} defaultValue="初始值" />
}
```

★ **关键认知**：**非受控(uncontrolled)**——表单值**由 DOM 自己维护**,React 不追踪,需要时用 `ref` 读。⚠ 非受控适合"**只在提交时读一次**"的简单场景;复杂交互(实时校验、格式化、联动)非受控就力不从心。

## 1.3 受控 vs 非受控

| 维度 | 受控 | 非受控 |
|------|------|--------|
| 数据源 | React state | DOM |
| 实时读值 | 随时可读 | 需 ref |
| 实时校验/格式化 | 易 | 难 |
| 代码量 | 略多 | 少 |
| 推荐 | 多数场景 | 简单提交场景 |

★ **关键认知**：**React 官方推荐受控**——它是"声明式"的延伸(UI 由 state 决定),灵活可控。⚠ 但"每个输入都受控"在大表单里很啰嗦,这正是后面 React Hook Form(第 21 篇)用"非受控 + 按需受控"来解决问题的原因。

# 2. 各类控件

## 2.1 文本与多行

```jsx
const [text, setText] = useState('')
<input value={text} onChange={e => setText(e.target.value)} />
<textarea value={text} onChange={e => setText(e.target.value)} />  {/* 无子内容,值也是 value */}
```

★ **关键认知**：`input`/`textarea` 受控都用 `value + onChange`。⚠ JSX 的 `<textarea>` **不用写子内容**,值放 `value`(与 HTML 不同,HTML 的 textarea 是子文本)——这是 JSX 与 HTML 的一个差异点。

## 2.2 复选框

```jsx
const [checked, setChecked] = useState(false)
<input type="checkbox" checked={checked} onChange={e => setChecked(e.target.checked)} />
```

★ **关键认知**：复选框用 `checked`(不是 value)绑定布尔,读 `e.target.checked`。⚠ 多选框绑定一个**数组**,勾选/取消时往数组增删,`checked` 是"是否在数组里"的判断。

## 2.3 单选钮与下拉

```jsx
const [gender, setGender] = useState('male')
<input type="radio" value="male" checked={gender === 'male'}
       onChange={() => setGender('male')} /> 男

const [city, setCity] = useState('')
<select value={city} onChange={e => setCity(e.target.value)}>
  <option value="beijing">北京</option>
  <option value="shanghai">上海</option>
</select>
```

★ **关键认知**：单选用 `value + checked`(比较是否选中),下拉(`select`)受控直接绑在 `select` 的 `value` 上(而非 option)。⚠ `select` 受控时初始值要落在某个 `<option value>` 上,否则会显示空;这是 React 表单里一个易错点。

# 3. 表单的进阶价值

## 3.1 受控带来的能力

```jsx
const [name, setName] = useState('')
const upper = name.toUpperCase()   // 随时从 state 派生(校验、格式化)

return (
  <>
    <input value={name} onChange={e => setName(e.target.value)} />
    <p>预览:{upper}</p>
    <button disabled={name.length < 1}>提交</button>   // 实时禁用
  </>
)
```

★ **关键认知**：受控的**真正价值**是"值在 state 里,想怎么处理都行"——**实时校验、格式化、按钮禁用、联动**都唾手可得。⚠ 这是非受控给不了的;也是为什么"需要实时反馈的表单"必然受控。

## 3.2 受控的代价

★ **关键认知**：受控的代价——每个字段一个 state + onChange,**大表单会非常啰嗦**,且**每次击键都重渲染整个表单**。⚠ 这两点正是 React Hook Form 第 21 篇的切入点:用"非受控(register)+ 最小重渲染"解决。

# 4. 本篇小结与下一步

★ **关键认知**：受控(state 是单一数据源,灵活,官方推荐)vs 非受控(DOM 维护,ref 读,简单场景);文本/下拉用 value,复选/单选用 checked。受控的代价(啰嗦 + 每击键重渲染)引出第 21 篇的 React Hook Form。

下一步进入**阶段二 · Hooks 全家桶**,第 8 篇:useEffect 与副作用——依赖数组、清理、心智模型,React 最重要的 Hook。