JSX 里不能写 `if`,于是 React 的条件渲染有了几套惯用法:`&&` 短路、三元、提前 return。本篇讲清这几种写法的适用场景、可读性取舍,以及 `{count && ...}` 那个会把 `0` 渲染出来的著名坑。

# 1. 条件渲染的三种姿势

## 1.1 三元表达式

```jsx
function Status({ loading }) {
  return (
    <div>
      {loading ? <Spinner /> : <Content />}   {/* 二选一,最常用 */}
    </div>
  )
}
```

★ **关键认知**：三元(`?:`)是"**二选一**"的标准写法,清晰、居中。⚠ 三元**嵌套**会越读越累(条件一多就换成提前 return 或独立函数),别把三元套成金字塔。

## 1.2 && 短路

```jsx
function Notify({ message }) {
  return (
    <div>
      {message && <div className="alert">{message}</div>}  {/* 有值才显示 */}
    </div>
  )
}
```

★ **关键认知**：`&&` 短路用于"**有就渲染、没有就不渲染**"的单分支。⚠ 它依赖于"`&&` 左边 falsy 时整个表达式返回左边、JSX 不渲染 falsy"这条规则(第 2 篇);陷阱见第 3 节。

## 1.3 提前 return

```jsx
function User({ user }) {
  if (!user) return null            // 无数据直接返回空
  if (user.isAdmin) return <AdminPanel />  // 分支多、组件复杂时提前 return 更清晰
  return <UserPanel user={user} />
}
```

★ **关键认知**：**提前 return** 是"**多分支、组件较复杂**"时最清晰的选择——在函数体前面处理边界(空态、加载态、特殊态),主线放在最后。⚠ "先返回边界态,日光大道留主线"是提升组件可读性的成熟模式。

# 2. 如何选择

| 场景 | 写法 |
|------|------|
| 二选一 | 三元 `?:` |
| 有则显示 | `&&` |
| 多分支/复杂组件 | 提前 return |
| 条件 + 否定 | 提取布尔变量或三元 |

★ **关键认知**：选择标准——**简单二选一用三元,单条件用 &&,复杂组件用提前 return**。⚠ 核心是**可读性优先**;把复杂条件抽成命名变量的布尔值(`const showEdit = isOwner && canEdit`)再渲染,常比层层嵌套的条件更清晰。

# 3. && 的 0 陷阱

## 3.1 0 会渲染出来

```jsx
const count = 0
// ⚠ 陷阱:0 是 falsy,但 && 返回的是 0,而 JSX 里 0 会被渲染成 "0"!
{count && <div>有商品</div>}   // 结果页面上显示一个 0
```

★ **关键认知**：`&&` 的坑——`count && <X/>` 当 `count` 是 `0` 时,表达式返回 `0`,而 **JSX 渲染 `0`**(只有 null/undefined/false 不渲染)。⚠ 这是 React 最经典的坑之一:数字变量用 `&&` 做条件,为 0 时会**显示个突兀的 0**。

## 3.2 解法

```jsx
{count > 0 && <div>有商品</div>}      // 明确布尔
{!!count && <div>有商品</div>}        // 转布尔
{Boolean(count) && <div>...</div>}    // 更显式
```

★ **关键认知**：解法是把左边**变成真正的布尔**——`count > 0`、`!!count` 或 `Boolean(count)`。⚠ 记住:只要是**数字变量**参与 `&&`,就必须警惕;保证左边是布尔,才安全地依赖"falsy 不渲染"。

# 4. 循环与条件结合

## 4.1 列表里的条件

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos
        .filter(t => !t.done)          // 先过滤再渲染
        .map(t => <li key={t.id}>{t.text}</li>)}
    </ul>
  )
}
```

★ **关键认知**：列表 + 条件,最佳姿势是**先 `.filter` 再 `.map`**——数据处理归数据处理,渲染归渲染。⚠ 别在 JSX 里叠 `map` + 内层三元(可读性差);把"算哪些要显示"放到渲染前。

## 4.2 空态处理

```jsx
function List({ items }) {
  if (items.length === 0) return <Empty />   // 空态提前 return
  return <ul>{items.map(i => <li key={i.id}>{i}</li>)}</ul>
}
```

★ **关键认知**：**空态(empty state)** 要显式处理——列表为空时返回占位("暂无数据"),而不是渲染个空 `<ul>`。⚠ 这也是提前 return 模式的典型应用:数据没到位/为空,先给出明确反馈。

# 5. 本篇小结与下一步

★ **关键认知**：三元(二选一)、&&(有则显示,注意 0 陷阱)、提前 return(边界态 + 复杂分支)三种姿势;列表先 filter 再 map,空态显式处理。条件渲染以"可读性"为先。

下一步第 7 篇:表单与受控组件——受控/非受控的取舍、各种表单控件的状态绑定。