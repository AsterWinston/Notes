React 与 TypeScript 是黄金搭档——类型让 props、事件、Hooks 都获得编译期保障。本篇讲清函数组件的 props 类型、`children` 类型、事件类型、Hooks 类型、泛型组件,以及那些"类型写不对"的常见坑。

# 1. 组件的 props 类型

## 1.1 基础

```tsx
interface Props {
  name: string
  age?: number        // 可选
  onSelect: (id: number) => void   // 回调
}

function User({ name, age, onSelect }: Props) {
  return <div onClick={() => onSelect(1)}>{name} {age}</div>
}

// 或用 type + FC(新版推荐直接给参数注解)
```

★ **关键认知**：函数组件的 props 就一行——**给参数标 interface/type**。⚠ 现代 TS + React 推荐**直接注解参数**(而非老式的 `React.FC<Props>`,后者有隐含 children 等小坑);props 类型清晰,调用处补全、传错即报。

## 1.2 children 类型

```tsx
import { ReactNode } from 'react'

interface Props {
  children: ReactNode      // children 用 ReactNode(最宽,涵盖 JSX/字符串/null)
}

function Card({ children }: Props) {
  return <div>{children}</div>
}
```

★ **关键认知**：`children` 的类型用 **`ReactNode`**(最通用,涵盖元素、字符串、null、数组)。⚠ 别用 `JSX.Element` 或 `ReactElement`(太窄,不接受字符串/null);需要"只收元素"时才收窄。

# 2. 事件类型

## 2.1 事件处理

```tsx
function Button() {
  function onClick(e: React.MouseEvent<HTMLButtonElement>) {   // 鼠标事件 + 元素类型
    console.log(e.target)
  }
  function onChange(e: React.ChangeEvent<HTMLInputElement>) {  // 输入变化
    console.log(e.target.value)
  }
  return (
    <>
      <button onClick={onClick}>点</button>
      <input onChange={onChange} />
    </>
  )
}
```

★ **关键认知**：事件参数用 **`React.XxxEvent<元素类型>`**——`MouseEvent<HTMLButtonElement>`、`ChangeEvent<HTMLInputElement>`、`KeyboardEvent` 等。⚠ 泛型里的**元素类型要配对**(button 配 HTMLButtonElement,input 配 HTMLInputElement);内联箭头函数通常可省略类型(自动推导),抽成具名函数才需标注。

## 2.2 常用事件类型速查

| 事件 | 类型 |
|------|------|
| 点击 | `React.MouseEvent<HTMLButtonElement>` |
| 输入 | `React.ChangeEvent<HTMLInputElement>` |
| 键盘 | `React.KeyboardEvent<HTMLInputElement>` |
| 表单提交 | `React.FormEvent<HTMLFormElement>` |
| 焦点 | `React.FocusEvent<HTMLInputElement>` |

# 3. Hooks 的类型

## 3.1 useState / useRef

```tsx
const [count, setCount] = useState<number>(0)          // 自动推导,复杂时显式
const [user, setUser] = useState<User | null>(null)    // 空值必须显式标注

const inputRef = useRef<HTMLInputElement>(null)        // ref 元素类型 + null
// inputRef.current 类型是 HTMLInputElement | null
```

★ **关键认知**：`useState` 通常**自动推导**,但**初始为 null 必须显式**(`useState<User | null>(null)`),否则推成 `null` 类型无法再赋值。⚠ `useRef` 同理——要"拿 DOM"就写 `useRef<HTMLInputElement>(null)`,`current` 会是 `元素 | null`。

## 3.2 自定义 Hook 的返回值类型

```tsx
function useFetch<T>(url: string) {        // 泛型 Hook,返回类型由 T 决定
  const [data, setData] = useState<T | null>(null)
  // ...
  return { data }    // { data: T | null } 自动推导
}

const { data } = useFetch<User>('/api/user')   // data 是 User | null
```

★ **关键认知**：自定义 Hook 用**泛型**让返回类型随调用处流动(`useFetch<T>`)。⚠ 值与组件一样泛型化,让复用逻辑也享有类型安全,这是"类型化 Hook"的完整形态。

# 4. 泛型组件

## 4.1 泛型组件写法

```tsx
// 泛型组件:接收不限类型的列表
function GenericList<T>({ items, renderItem }: {
  items: T[]
  renderItem: (item: T) => ReactNode
}) {
  return <ul>{items.map((item, i) => <li key={i}>{renderItem(item)}</li>)}</ul>
}

<GenericList items={[1, 2, 3]} renderItem={n => <b>{n}</b>} />     {/* T 推断为 number */}
<GenericList items={[{ id: 1 }]} renderItem={u => <b>{u.id}</b>} /> {/* T 推断为对象 */}
```

★ **关键认知**：泛型组件用 `<T>` 让 props 类型随传入数据流动——通用列表、下拉、表格这类"承载任意类型"的组件。⚠ JSX 里泛型组件有**尖括号歧义**问题(像标签),可加 `extends unknown` 或 `.tsx` 里用函数声明式;上面这种**函数声明**(`function GenericList<T>`)能正常写,箭头函数 `const X = <T,>(` 需加逗号。

## 4.2 冒号逗号技巧

```tsx
// 箭头函数泛型在 .tsx 里的写法:<T,> 加个逗号,告诉 TS 这是泛型不是 JSX
const List = <T,>({ items }: { items: T[] }) => ...
```

★ **关键认知**：`.tsx` 里**箭头函数 + 泛型**要写成 `<T,>`(逗号防歧义),否则被认成 JSX 标签。⚠ 用 `function` 声明则无此问题;这是个"知道就好"的小技巧,遇到报错"JSX 元素没有构造"时先查这个。

# 5. 完整用好 TS 的实践

## 5.1 联合/可辨识的类型

```tsx
type Status =
  | { state: 'loading' }
  | { state: 'success'; data: User[] }
  | { state: 'error'; message: string }

function List({ status }: { status: Status }) {
  if (status.state === 'loading') return <Spinner />
  if (status.state === 'error') return <div>{status.message}</div>
  return <ul>{status.data.map(...)}</ul>   // 到这里 data 已收窄
}
```

★ **关键认知**：React 里 TS 的杀手级用法是**可辨识联合**(Vue 篇第 3 篇同款)——请求/状态机的"loading/success/error"用联合建模,`if` 收窄后各分支类型精确。⚠ 这比一堆可选字段起手更严谨,也让 UI 与数据状态严格对应。

## 5.2 配合 React Hook Form/Zod

★ **关键认知**：TS 与 React 生态的协同典范是 **RHF + Zod**(第 21 篇)——表单 schema 一份,zod 管校验、TS 管类型,提交数据自动带类型。⚠ "类型即 schema"的现代实践,让表单这个最容易"手写 any"的地方也类型安全。

# 6. 本篇小结与下一步

★ **关键认知**：props 直接注解参数、children 用 ReactNode、事件用 React.XxxEvent<元素>、空值 useState/useRef 要显式类型、泛型组件让类型流动(<T> 或 <T,>)。React+TS 的合力,让整条数据链路类型安全。

下一步第 27 篇:SSR/RSC/Next.js——服务端渲染与 Server Components,React 的"服务端"大戏。