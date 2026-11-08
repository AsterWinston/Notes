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
| 每击键重渲染 | 是 | 否 |
| 代码量 | 略多 | 少 |
| 推荐 | 多数场景 | 简单提交场景 |

★ **关键认知**：**React 官方推荐受控**——它是"声明式"的延伸(UI 由 state 决定),灵活可控。⚠ 但"每个输入都受控"在大表单里很啰嗦,这正是后面 React Hook Form(第 21 篇)用"非受控 + 按需受控"来解决问题的原因。

## 1.4 value + onChange 为什么必须成对

```text
受控闭环拆解:
用户敲一个键 → DOM 值先变了(浏览器原生行为,拦不住这一步)
  → onChange 触发 → setValue → 重渲染
  → React 把 value prop 再写回 DOM(与当前值一致就跳过)

只写 value 不写 onChange:state 永远按不住 → 打字"没反应"(只读)+ 警告
```

```jsx
<input value={v} />                                        // ✗ 只读 + 警告
<input value={v} onChange={e => setV(e.target.value)} />   // ✓ 完整闭环
<input defaultValue="abc" />                               // ✓ 非受控:只首次挂载写一次
```

★ **关键认知**：受控的本质是"**渲染时把 value 强行按到 state 值上**"——`value` 与 `onChange` 是一个闭环的两半,缺一半就断流:只写 `value` 时每次渲染都把 DOM 按回初值,表现为"打字没反应"。⚠ `value` 和 `defaultValue` 二选一:同一控件先受控后非受控(或混写),React 会警告"switching between controlled and uncontrolled"——控件的"数据源所有权"不允许中途换主人。

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

// 多选绑数组:checked 是"在不在数组里",onChange 做增删
const [tags, setTags] = useState([])
<input type="checkbox"
       checked={tags.includes('fe')}
       onChange={e => setTags(t => e.target.checked ? [...t, 'fe'] : t.filter(x => x !== 'fe'))} />
```

★ **关键认知**：复选框用 `checked`(不是 value)绑定布尔,读 `e.target.checked`。⚠ 多选框绑定一个**数组**,勾选/取消时往数组增删,`checked` 是"是否在数组里"的判断。

## 2.3 单选钮与下拉

```jsx
const [gender, setGender] = useState('male')
<input type="radio" value="male" checked={gender === 'male'}
       onChange={() => setGender('male')} /> 男

const [city, setCity] = useState('')
<select value={city} onChange={e => setCity(e.target.value)}>
  <option value="">请选择</option>          {/* 空值占位,否则初始显示空白 */}
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

## 3.3 重置表单:key 技巧与 form.reset()

```jsx
// 技巧一(受控世界):给表单套 key,换 key = 子树卸载重建,所有字段归零
const [formKey, setFormKey] = useState(0)
<Form key={formKey} />
<button onClick={() => setFormKey(k => k + 1)}>再填一单</button>
// 机制:key 变 → React 视为不同元素 → 内部 useState/effect 全部重来(第 5 篇 3.5)

// 技巧二(非受控世界):原生 form 元素一键回到 defaultValue
const formRef = useRef(null)
<form ref={formRef}>…</form>
<button onClick={() => formRef.current.reset()}>清空</button>
```

★ **关键认知**：受控表单"重置"逐字段 `set('')` 麻烦且易漏,**换 key 整树重建**是最省心的兜底;非受控交给原生 `form.reset()`。⚠ 换 key 会连带丢焦点、滚动与内部所有状态——只适合"语义上就是新表单"的场景(弹窗重开、再填一单)。

## 3.4 经典坑:IME 组合输入与"半受控"

```jsx
// ⚠ 中文输入法经历 composition(组合)阶段:拼音还没变成汉字,onChange 已在逐字符触发
// 受控组件若在组合期把值"改写回 DOM"(截断/过滤/大写化),浏览器会打断组词:
// 表现为"拼音打一半消失/变成了英文字母"

function Search() {
  const [kw, setKw] = useState('')
  const composing = useRef(false)
  return (
    <input
      value={kw}
      onCompositionStart={() => (composing.current = true)}
      onCompositionEnd={e => { composing.current = false; setKw(e.target.value) }}
      onChange={e => { if (!composing.current) setKw(e.target.value) }}  // 组词期跳过写回
    />
  )
}

// ⚠ 坑二:后端字段可能为 null → value={null}(≈不传=非受控),数据到达后变受控,
//    React 警告 uncontrolled → controlled → 永远兜底成字符串:
<input value={form.name ?? ''} onChange={e => setName(e.target.value)} />
```

★ **关键认知**：IME 坑的根源是**受控闭环在组合期被"改写"打断**——`value` 写回 DOM 会重置 composition,所以组词期间必须"放行不干预"(或搜索防抖也延后到 `compositionend`)。⚠ 第二个坑同理:数字/文本字段用 `?? ''` 兜底,别让"还没加载的数据"把控件从受控踢回非受控——这两类都是"每次渲染都稳定"原则在表单上的投影。

## 3.5 React 19:表单有了原生搭档

```jsx
import { useActionState } from 'react'   // React 19 内置
const [, formAction, isPending] = useActionState(
  async (prev, formData) => search(formData.get('q')),  // action 直接收 FormData
  null
)
return <form action={formAction}><input name="q" /><button disabled={isPending}>搜索</button></form>
```

★ **关键认知**：React 19 的 `useActionState`/`useFormStatus`/`useOptimistic` 把"提交—pending—错误回填"标准化——`<form action={fn}>` 让提交成为一次过渡,**非受控件"只在提交时读"重新变得顺手**(配合第 29 篇 Server Actions)。⚠ 它缓解的是提交流程的啰嗦,**不改变受控/非受控的本质抉择**;字段级实时校验大表单仍归 React Hook Form(第 21 篇)。

# 4. 本篇小结与下一步

| 概念 | 要点 |
|------|------|
| 受控闭环 | value 强按 state;value 与 onChange 必须成对 |
| 非受控 | DOM 维护,ref/defaultValue,提交时结算 |
| 控件绑定 | 文本/下拉 value;复选/单选 checked;数组用 includes |
| 重置 | 换 key 重建子树;非受控 form.reset() |
| IME 坑 | 组合期不写回;value 永远兜底成字符串 |

★ **关键认知**：受控(state 是单一数据源,灵活,官方推荐)vs 非受控(DOM 维护,ref 读,简单场景);文本/下拉用 value,复选/单选用 checked。受控的代价(啰嗦 + 每击键重渲染)引出第 21 篇的 React Hook Form。

下一步进入**阶段二 · Hooks 全家桶**,第 8 篇:useEffect 与副作用——依赖数组、清理、心智模型,React 最重要的 Hook。
