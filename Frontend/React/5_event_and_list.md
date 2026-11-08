交互的核心是事件,列表的核心是 key。本篇讲清 React 的**合成事件**(为什么用一个不存在的 `e.target`跨浏览器)、事件绑定的写法、列表渲染与 key 的作用,以及"某些事件不冒泡"这类细节坑。

# 1. 事件绑定

## 1.1 基本写法

```jsx
function Button() {
  function handleClick(e) {        // e 是合成事件对象
    console.log('点击了', e.target)
  }
  return <button onClick={handleClick}>点我</button>   // 驼峰命名,传函数
}
```

★ **关键认知**：React 事件是**驼峰命名**的 JSX 属性(`onClick` 而非 `onclick`),值是**函数**(而非字符串)。⚠ 与原生 `addEventListener` 的两点差异:名字驼峰化、传的是函数不是字符串——`onClick={handleClick}` 传引用,`onClick={() => ...}` 传内联函数。

## 1.2 传参

```jsx
function Item({ id, onDelete }) {
  return (
    <button onClick={() => onDelete(id)}>删除</button>   // 内联箭头函数传参
  )
  // 或 onClick={onDelete.bind(null, id)}
}
```

★ **关键认知**：要带参数的事件处理,用**内联箭头函数**包一层(`() => onDelete(id)`)。⚠ 常见错误:`onClick={onDelete(id)}`——这会在**渲染时立即调用**(不是点击时),因为括号让函数执行了。区分"传函数"和"调函数"。

# 2. 合成事件

## 2.1 为什么是"合成"的

```jsx
function handle(e) {
  e.preventDefault()   // 与原生一致的 API
  e.stopPropagation()
  console.log(e.target)   // 目标元素
  console.log(e.currentTarget) // 绑定处理器的元素
}
```

★ **关键认知**：React 用**合成事件(SyntheticEvent)** 包装原生事件——**跨浏览器统一 API**,并实现**事件委托**(事件其实挂在了 root,而非每个元素)。⚠ 日常用起来和原生几乎一样(preventDefault/stopPropagation 都在),但**它不是原生事件**,拿原生属性需小心。

## 2.2 事件委托与性能

★ **关键认知**：React 的合成事件基于**事件委托**——不必给每个元素单独加监听,而是在顶层统一捕获/处理,高效、且**新挂载的元素自动有事件**(因为靠冒泡到顶层)。⚠ 这是 React 区别于"手写 addEventListener"的性能设计,也是理解"为什么卸载组件后事件不会泄漏"的线索。

```
委托的具体结构与一次点击的流程:
document
└─ #root  ◀── 每类事件(如 click)只有这一个统一原生监听
   └─ <App>
      └─ <li><button onClick={fn}>…</button></li>   ← fn 只是存在 fiber 的 props 里

1. 原生事件在 DOM 上冒泡,到达 root container 的统一监听器
2. React 从 e.target 沿 Fiber 树向上,收集路径上所有 onClick
3. 按"捕获 Capture → 冒泡"顺序在 React 自己的事件系统里依次调用
4. React 17+ 把委托目标从 document 改到 root container —— 多 root(微前端)互不串门
```

★ **关键认知**：React 17 起委托目标是 **root container**(`createRoot` 的容器)而非 `document`,目的是**多实例隔离**与事件行为可逐步停用。⚠ 由此产生高频坑:用 `document.addEventListener('click', …)` 做"点击外部关闭下拉"时,它的触发时机与 React 内部处理**交错**,且 `stopPropagation` 拦不住已冒到 root 之外的事——更稳的做法是用 `onPointerDown` 或在 root 上注册捕获阶段监听,别依赖"原生一定先于/后于 React"的旧直觉。

## 2.3 passive event:滚动性能与警告

```jsx
// 控制台:Unable to preventDefault inside passive event listener invocation.
<div onWheel={e => e.preventDefault()} />        // ⚠ 无效并告警
<div onWheel={e => { /* 只读不拦,交给页面滚动 */ }} />  // ✓ passive 的本意
```

★ **关键认知**：浏览器把 `touchstart/touchmove/wheel` **默认按 passive 注册**(承诺不 `preventDefault`,合成层滚动不必等 JS),React 17 的委托同样如此——于是这些回调里调 `preventDefault()` **失效并告警**。⚠ 真要阻止默认滚动(地图/画布手势),只能对具体元素上原生 `addEventListener('wheel', fn, { passive: false })`,合成事件体系不提供这个开关。

# 3. 列表与 key

## 3.1 列表渲染

```jsx
const users = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }]

function UserList() {
  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  )
}
```

★ **关键认知**：列表用 `.map` 生成元素数组,每个元素要带 **`key`**(在其兄弟间**唯一且稳定**)。⚠ 缺 key 或 key 不稳定,React 会报警,更重要的是 **reorder/diff 复用会错乱**(第 31 篇)。

## 3.2 为什么不能用 index

```jsx
// ⚠ 反模式:用 index 当 key,插入/删除/重排时复用错位
{list.map((item, i) => <li key={i}>{item.text}</li>)}
```

★ **关键认知**：`key` 的作用是让 React 判断"**新旧的哪个元素是同一个**",从而**复用**对应 DOM/状态。⚠ 用 `index` 当 key,列表插入/重排时"下标不变但内容变了"的元素会被**原地复用**,导致组件状态错乱(输入框文字串行)——**用业务唯一 id 当 key**,与 Vue 第 5 篇是同一原则。

```jsx
// 事故演示:头部插一条,index 全体平移
//   旧: key=0(Alice)   key=1(Bob)
//   新: key=0(新消息)  key=1(Alice)  key=2(Bob)
// React 认定 key=0 "还是原来那个" → 原地复用 → 新消息继承了 Alice 的内部状态
```

## 3.3 key 只在兄弟间唯一

```jsx
const items = [...]
<>
  {items.map(i => <Item key={i.id} {...i} />)}   {/* 同一列表内 key 唯一即可 */}
  {otherItems.map(o => <Other key={o.id} {...o} />)}  {/* 不同列表可重复 */}
</>
```

★ **关键认知**：key **只需在"同一层兄弟"间唯一**,全局可重复。⚠ 数组 `.map` 里 key 写在**最外层返回的元素**上(不是元素内部的某个子标签)——写错位置等于没写。

## 3.4 key 与 reconciliation:按身份"认领"

```
协调器对比两组子节点的核心动作(简化):
旧: [A(k=1), B(k=2), C(k=3)]
新: [C(k=3), A(k=1), B(k=2)]

1. 新数组的 key 汇成 Map: {3→C, 1→A, 2→B}
2. 遍历旧 children,按 key 进 Map "认领" → 认领到的复用 Fiber+DOM,只挪位置
3. 没被认领的旧节点删除,Map 剩余的是新增节点

⚠ key 不是"防警告的仪式",是这套"按身份认领"算法的入场券;
   没有 key 就退化为"按下标同位对比"——删一行,后面每个组件都被判"变了"。
```

★ **关键认知**：**列表性能的本质是"给协调器提供稳定身份"**——key 稳定,React 才能"移动 DOM 而不是销毁重建";index 作 key 时任何头部插入都让后续项身份错位,整段列表退化为重建。⚠ 反过来,**每次渲染重新生成 key**(`key={crypto.randomUUID()}`)等于亲手宣布"全部是新的",memo、状态、焦点统统失效——这是 key 事故里比 index 更隐蔽的一种。

## 3.5 换 key 的"正用":强制重置状态

```jsx
// key 变 → React 视为不同元素 → 销毁旧 Fiber,状态/effect 全部重来
function Editor({ docId }) {
  return <TextAreaWrapper key={docId} />   // 切换文档,编辑器内部 state 自动清零
}
// 同理:表单重置不必逐字段 setValue('')——给 <Form key={formKey}> 换个 key(见第 7 篇)
```

★ **关键认知**：key 除了"复用"还有反向用途——**改 key 即重置组件**。⚠ 要克制:重建会丢焦点、滚动位置、动画状态,只在"语义上这就是一个新实例"时用;想"隐藏但保留状态"的,用 CSS 的 `display: none` 或条件渲染上移(第 6 篇),而不是乱换 key。

# 4. 事件的细节坑

## 4.1 受控与 this

★ **关键认知**：函数组件里**没有 `this`**——不用像类组件那样纠结 `this.handleClick` 的绑定。⚠ 这是函数组件比类组件舒服的一个点:事件处理器就是普通函数,闭包直接可用。

## 4.2 一些不冒泡的事件

```jsx
// onScroll、onMouseEnter 等不遵循常规冒泡;onFocus/onBlur React 用 onFocus/onBlur 冒泡替代
<input onFocus={() => ...} onBlur={() => ...} />
```

★ **关键认知**：大部分事件冒泡,但**个别不冒泡**(scroll 等);焦点类 `onFocus`/`onBlur` 在 React 里**已经做成"冒泡的"**(内部用了 focusin/focusout)。⚠ 需要"捕获阶段"处理时,用 `onClickCapture` 这类带 `Capture` 后缀的属性。

# 5. 本篇小结与下一步

| 主题 | 要点 |
|------|------|
| 事件绑定 | 驼峰属性 + 传函数引用;带参用内联箭头,别渲染时误调用 |
| 委托 | 监听挂在 root container;原生 document 监听的时机别想当然 |
| passive | wheel/touch 默认 passive,preventDefault 失效需原生监听 + passive:false |
| key 本质 | 协调器"按身份认领"的凭证;index/随机 key 都会破坏复用 |
| key 反用 | 换 key = 重置组件(表单重置、切文档) |

★ **关键认知**：事件驼峰命名、传函数;合成事件跨浏览器 + 事件委托;列表 `.map` 必须给稳定唯一 `key`(别用 index)。函数组件无 this,事件即闭包。

下一步第 6 篇:条件渲染与控制流——&&、三元、提前 return,以及 0 那个著名的坑。
