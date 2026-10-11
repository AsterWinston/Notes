React 受控表单有两大痛点:每个字段一个 state(啰嗦)、每次击键都触发整表单重渲染(性能)。**React Hook Form** 用"非受控 + 最小重渲染"化解。本篇讲清 RHF 的核心 API、与 Zod 的校验集成,以及"为什么要告别一堆 useState 绑定"。

# 1. 受控表单的痛点回顾

```jsx
// 受控:一个字段一个 state,每次输入一下,整个表单组件重渲染一遍
const [name, setName] = useState('')
const [email, setEmail] = useState('')
// onChange 里都要 setXxx...
```

★ **关键认知**：第 7 篇讲受控表单,它的代价就是**字段多时噼里啪啦 setState + 每个击键重渲染**。⚠ 小表单无所谓,但**几十个字段的大表单**,受控写法既啰嗦又可能卡——这是 RHF 的切入点。

# 2. React Hook Form 核心

## 2.1 register 非受控绑定

```jsx
import { useForm } from 'react-hook-form'

function Form() {
  const { register, handleSubmit } = useForm()

  function onSubmit(data) {          // data 是校验通过后的表单值
    console.log(data)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* register 把字段注册为非受控,值由 DOM 维护,不触发重渲染 */}
      <input {...register('name', { required: '请输入姓名' })} />
      <input {...register('email')} />
      <button type="submit">提交</button>
    </form>
  )
}
```

★ **关键认知**：RHF 用 **`register('字段名')` 非受控绑定**——表单单值由 DOM 维护,**输入不触发重渲染**(只有提交/校验时才读值)。⚠ 这直接解决了"受控表单每击键重渲染"的性能问题;`handleSubmit(onSubmit)` 包装提交,自动做校验、通过才调 `onSubmit`。

## 2.2 校验:required + rules

```jsx
<input {...register('age', {
  required: '必填',
  min: { value: 0, message: '不能为负' },
  pattern: { value: /^\d+$/, message: '必须是数字' }
})} />
```

★ **关键认知**：`register` 第二参数传**校验规则**(required/min/pattern),校验失败 `handleSubmit` 会拦住、错误通过 `formState.errors` 读取。⚠ 简单校验这样写够了;复杂校验(字段间关联、类型推导)则交给 Zod(见下)。

# 3. 表单状态与错误

## 3.1 errors 与 formState

```jsx
const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm()

<input {...register('name', { required: '必填' })} />
{errors.name && <span>{errors.name.message}</span>}   {/* 显示错误 */}
```

★ **关键认知**：`formState.errors` 装校验错误,`errors.name.message` 取具体信息。⚠ RHF 默认**提交时才校验**;要**实时校验**,用 `mode: 'onChange'`(代价是回到"高频触发",需权衡)。

## 3.2 为什么值不被 React 控制

★ **关键认知**：RHF 的哲学是"**表单多数情况下不需要受控**"——值存在 DOM 里,**提交/需要时才进 React**,从而最小化重渲染。⚠ 这也是它性能好的根本;需要"实时读值"的特殊字段,仍可用 `watch()`(选择性订阅)单独观察。

# 4. 与现代校验集成 Zod

## 4.1 zodResolver

```jsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  name: z.string().min(1, '必填'),
  email: z.string().email('邮箱格式不对'),
  age: z.number().min(0)
})

function Form() {
  const { register, handleSubmit } = useForm({
    resolver: zodResolver(schema)      // 用 zod schema 做校验
  })
  // onSubmit 里的 data 会带上类型(由 zod 推断)
}
```

★ **关键认知**：`zodResolver` 让 **Zod schema 接管校验**——用 `z.object({...})` 声明字段规则,类型由 schema **自动推导**(提交的 data 有类型)。⚠ Zod 擅长"字段交叉校验、复杂规则、导出类型",是 RHF 校验的首选搭配;简单表单仍可直接用 register 的 rules。

# 5. 完整示例

```jsx
function SignupForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema)
  })

  function onSubmit(data) {
    // data 已通过校验且类型正确:发送请求
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <label>姓名</label>
      <input {...register('name')} />
      {errors.name && <p>{errors.name.message}</p>}

      <label>邮箱</label>
      <input {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}

      <button type="submit">提交</button>
    </form>
  )
}
```

★ **关键认知**：一个 RHF + Zod 的完整表单就长这样——**register 绑定 + resolver 校验 + errors 显示 + handleSubmit 提交**,没有一堆 useState/onChange。⚠ 这是现代 React 表单的主流形态,既省代码又省渲染。

# 6. 本篇小结与下一步

★ **关键认知**：RHF 用"非受控 + register"避免每击键重渲染;校验用 register rules 或 Zod(zodResolver,类型推导);提交走 handleSubmit。核心哲学:表单值多数时候**不必受控**,提交才进 React。

下一步第 22 篇:样式方案——CSS Modules、CSS-in-JS、Tailwind,React 里怎么管样式。