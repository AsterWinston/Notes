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

## 1.4 第四种:抽组件

```jsx
// 条件复杂到 JSX 里写不下时,把"这一块"整体提成组件,条件逻辑进组件体
function StatusBanner({ user, config }) {
  if (user.banned) return <BannedBanner reason={user.reason} />
  if (!config.notice) return null
  return <NoticeBanner text={config.notice} />
}

function Profile(props) {
  return (
    <div>
      <StatusBanner {...props} />   {/* 调用处永远干净 */}
      <MainContent {...props} />
    </div>
  )
}
```

★ **关键认知**：JSX 内嵌条件超过两层,正解不是"更精巧的表达式",而是**把带条件的片段抽成组件**——条件回到 JS 语句的世界,`if/return` 随便用。⚠ 这是官方 FAQ 的一贯口径:**渲染的单位是组件,条件的单位也应该是组件**,别把逻辑硬压进一行模板。

# 2. 如何选择

| 场景 | 写法 |
|------|------|
| 二选一 | 三元 `?:` |
| 有则显示 | `&&` |
| 多分支/复杂组件 | 提前 return |
| 条件 + 否定 | 提取布尔变量或三元 |
| JSX 里条件写不下 | 抽成子组件,条件进组件体 |

★ **关键认知**：选择标准——**简单二选一用三元,单条件用 &&,复杂组件用提前 return**。⚠ 核心是**可读性优先**;把复杂条件抽成命名变量的布尔值(`const showEdit = isOwner && canEdit`)再渲染,常比层层嵌套的条件更清晰。

# 3. && 的 0 陷阱

## 3.1 0 会渲染出来

```jsx
const count = 0
// ⚠ 陷阱:0 是 falsy,但 && 返回的是 0,而 JSX 里 0 会被渲染成 "0"!
{count && <div>有商品</div>}   // 结果页面上显示一个 0
```

★ **关键认知**：`&&` 的坑——`count && <X/>` 当 `count` 是 `0` 时,表达式返回 `0`,而 **JSX 渲染 `0`**(只有 null/undefined/false 不渲染)。⚠ 这是 React 最经典的坑之一:数字变量用 `&&` 做条件,为 0 时会**显示个突兀的 0**。

为什么偏偏是 `0`——把"可渲染值"的名单摊开看:

```jsx
{null}       // 不渲染——React 视为"空"
{undefined}  // 不渲染
{false}      // 不渲染(true 同样不渲染)
{0}          // ⚠ 渲染出文本 "0"
{NaN}        // ⚠ 渲染出 "NaN"
{'0'}        // 渲染 "0"——字符串照渲染
{123}        // 渲染 "123"——数字本身就是合法"文本节点"描述
```

★ **关键认知**：数字与字符串在 React 眼里都是**合法的可渲染文本**(否则 `{user.age}`、`{42}` 就没法渲染),被过滤掉的只有 `null/undefined/boolean` 四种;而 `&&` 短路返回的是**左操作数原值**而非布尔,falsy 里恰好只剩 `0`(和 `NaN`)漏进渲染通道。⚠ 同族变体:`{list.length && <X/>}` 空数组显示 `0`、`{count ?? <X/>}` 对 `0` 反而"渲染 0 不渲染兜底"——数字参与逻辑短路,一律先转布尔。

## 3.2 解法

```jsx
{count > 0 && <div>有商品</div>}      // 明确布尔
{!!count && <div>有商品</div>}        // 转布尔
{Boolean(count) && <div>...</div>}    // 更显式
```

★ **关键认知**：解法是把左边**变成真正的布尔**——`count > 0`、`!!count` 或 `Boolean(count)`。⚠ 记住:只要是**数字变量**参与 `&&`,就必须警惕;保证左边是布尔,才安全地依赖"falsy 不渲染"。

补一句工程化防线:typescript-eslint 的 `no-unnecessary-condition` 系规则(以及 React Hooks 专项的 `react/no-leaked-conditional-rendering`)能在编译期抓住"左边可能是数字/空字符串的 &&",比靠 code review 肉眼盯可靠得多。

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

## 4.3 条件切换与"状态去留"

```jsx
// 经典"怪现象":登录/注册间切换,输入框内容居然还在!
function Auth() {
  const [isLogin, setIsLogin] = useState(true)
  return isLogin
    ? <form><input placeholder="账号" /></form>     {/* 两个分支结构相同:form>input */}
    : <form><input placeholder="密码" /></form>
  // diff 看:type 都是 form/input,key 都没有 → 判定"同一个" → 原地复用,值保留
}

// 想"切换即清空"?让 React 认出"这是不同的东西":
return isLogin
  ? <form key="login">…</form>
  : <form key="register">…</form>    // key 不同 → 旧卸载新挂载,状态归零
```

★ **关键认知**：条件渲染的隐规则——React 按**(位置, type, key)**判断是否同一节点:同类型同 key 就**保留 DOM 与 state** 原地更新,类型/key 不同才**卸载旧的、挂新的**。⚠ 两大悬案都由此解开:"两个 `<input>` 串内容"是复用(加 key 区分);切走的组件 effect 跑了 cleanup 是卸载——**条件渲染 = 挂载/卸载,不是显示/隐藏**;对照 Vue 的 `v-show` 那档"隐藏但保状态",React 里要手动用 `style.display` 或把组件留在树里。

## 4.4 early return 与 hooks 顺序

```jsx
function Panel({ open }) {
  const [tab, setTab] = useState(0)     // ✓ hook 全在 return 之前
  if (!open) return null                // 之后才提前 return
  return <div>{tab}</div>
}

function Bad({ open }) {
  if (!open) return null                // ⚠ 错误:return 在 hook 前
  const [tab, setTab] = useState(0)     // 有渲染少调了这个 hook → 报错
  return <div>{tab}</div>
}
```

★ **关键认知**：提前 return **本身与 hooks 不冲突**,但必须排在**所有 hook 调用之后**——React 靠"调用顺序"对齐 hook 槽位,分支跳过一个 hook 就破坏链表(第 15、32 篇)。⚠ 别在 if 里调 hook;想让组件"关闭时零开销",正确做法是在**父组件**条件渲染它(`{open && <Panel/>}`),让它根本不被调用,而不是在它内部早退。

# 5. 本篇小结与下一步

| 写法 | 适用 | 注意 |
|------|------|------|
| 三元 `?:` | 二选一 | 别嵌套成金字塔 |
| `&&` | 有则显示 | 左边必须真布尔,防 0/NaN 泄漏 |
| 提前 return | 边界态/多分支 | 放在所有 hook 之后 |
| 抽子组件 | JSX 写不下 | 条件进组件体,调用处干净 |
| key 区分 | 切换即重置状态 | 同类型会被原地复用 |

★ **关键认知**：三元(二选一)、&&(有则显示,注意 0 陷阱)、提前 return(边界态 + 复杂分支)三种姿势;列表先 filter 再 map,空态显式处理。条件渲染以"可读性"为先。

★ 机制层再记两条:0 陷阱源于"数字本来就是合法可渲染值";条件切换默认"同类型保状态、换类型即卸载",要重置就换 key。

下一步第 7 篇:表单与受控组件——受控/非受控的取舍、各种表单控件的状态绑定。
