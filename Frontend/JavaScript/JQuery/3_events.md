事件是页面"活起来"的入口。jQuery 的事件系统在原生 `addEventListener` 之上补了三样东西：**统一的绑定接口**、**事件委托**、**命名空间**。用好了，动态列表、批量解绑、组件式交互都很干净；用不好，就会出现"点一次触发三次""解绑把别人的监听也删了"这类事故。

# 1. 事件模型

## 1.1 原生事件的三个阶段

```text
window / document
      │ ① 捕获（capture）
      ▼
   祖先元素
      ▼
   目标元素   ← ② 目标阶段
      ▼
   祖先元素   ← ③ 冒泡（bubble）
```

绝大多数交互代码用的是**冒泡**：点击子元素，事件会一层层向上传，父元素上的监听也能收到。事件委托正是利用了这个特性。

★ **关键认知**：jQuery 的 `.on()` 默认在**冒泡阶段**触发。原生 `addEventListener(type, fn, true)` 的第三个参数 `true` 表示捕获，jQuery 没有直接暴露这个开关。理解"冒泡"，是理解委托的前提。

## 1.2 事件 API 一览

| API | 作用 | 推荐度 |
|-----|------|--------|
| `.on(events, [selector], [data], handler)` | 绑定，支持委托 | 首选 |
| `.one(...)` | 绑定一次，触发后自动解绑 | 常用 |
| `.off(events, [selector], [handler])` | 解绑 | 首选 |
| `.trigger('type', [extra])` | 触发并冒泡 | 常用 |
| `.triggerHandler('type')` | 触发但不冒泡、不执行默认行为 | 特定场景 |
| `.click(fn)` 等简写 | 绑定单个事件 | 老代码常见，新代码建议 `on` |

# 2. 绑定与解绑

## 2.1 on 的基本用法

```javascript
// 1. 直接绑定：this 是原生 DOM
$('#btn').on('click', function () {
  console.log('点击了', this)
})

// 2. 一次绑定多个事件，共用处理器
$('#input').on('focus blur', function (e) {
  console.log(e.type)
})

// 3. 用对象分别指定处理器
$('#box').on({
  mouseenter: function () { $(this).addClass('hover') },
  mouseleave: function () { $(this).removeClass('hover') }
})

// 4. 传额外数据
$('#btn').on('click', { id: 42 }, function (e) {
  console.log(e.data.id)      // 42
})
```

★ **关键认知**：`.on()` 第一个参数是事件名，可带命名空间；第二个参数若是字符串就是**委托选择器**，若是对象就是**额外数据**；最后才是处理器。参数位置固定，别把委托选择器与数据搞混。

## 2.2 事件对象

```javascript
$('#btn').on('click', function (e) {
  e.type            // "click"
  e.target          // 实际触发的最深元素
  e.currentTarget   // 当前执行处理器的元素（等于 this）
  e.delegateTarget  // 委托绑定的容器
  e.which           // 按键或鼠标键编号

  e.preventDefault()            // 阻止默认行为，如链接跳转
  e.stopPropagation()           // 阻止冒泡
})
```

```javascript
// 典型用法：阻止表单默认提交
$('#form').on('submit', function (e) {
  e.preventDefault()
  console.log('提交', $('#name').val())
})
```

⚠ `e.target` 与 `e.currentTarget` 是排查事件问题的第一把钥匙：点击 `<li><span>文字</span></li>` 时，`target` 可能是 `span`，`currentTarget` 才是绑定处理器的 `li`。

## 2.3 off 与 one

```javascript
function handleClick() {
  console.log('clicked')
}

$('#btn').on('click', handleClick)
$('#btn').off('click', handleClick)   // 精确解绑
$('#btn').off('click')                // 解绑该元素所有 click

$('#btn').one('click', function () {
  console.log('只会执行一次')
})
```

★ **关键认知**：`.off('click', fn)` 必须传入**同一个函数引用**。两次写匿名函数，引用不同，解绑无效：

```javascript
// 错误：第二个匿名函数不是同一个引用
$('#btn').on('click', function () { /* ... */ })
$('#btn').off('click', function () { /* ... */ })

// 正确
const handler = function () { /* ... */ }
$('#btn').on('click', handler)
$('#btn').off('click', handler)
```

# 3. 事件委托

## 3.1 为什么需要委托

```javascript
// 直接绑定：只对绑定那一刻已存在的 li 生效
$('#list li').on('click', function () {
  $(this).toggleClass('done')
})

// 新增的 li 没有监听，点了没反应
$('#list').append('<li>新任务</li>')
```

每次新增后重新绑定既麻烦又容易绑重复。更好的做法是把监听挂在**不会变的父容器**上，靠冒泡统一处理。

## 3.2 委托写法与原理

```javascript
// 第三个参数 "li" 是委托选择器
$('#list').on('click', 'li', function () {
  // this 指向匹配 "li" 的元素，而不是 #list
  $(this).toggleClass('done')
})
```

```text
点击流程：
1. 用户点击某个 <li> 内部的 <span>
2. 事件从 span 冒泡到 li，再到 #list
3. #list 上的处理器被调用
4. jQuery 用 e.target 向上匹配选择器 "li"
5. 命中后，以该 li 作为 this 执行回调
```

★ **关键认知**：委托的处理器只在容器上注册一次，新增子元素天然生效，内存里也只有一份监听。⚠ 委托选择器要具体，别写 `$('body').on('click', '*', fn)`，那等于每次点击都做一次全量匹配。

## 3.3 委托中的判断与限制

```javascript
$('#list').on('click', 'li', function (e) {
  // 点删除按钮时，不触发整行的切换
  if ($(e.target).closest('.delete').length) {
    $(this).remove()
    return
  }
  $(this).toggleClass('done')
})
```

| 事件 | 能否委托 | 说明 |
|------|---------|------|
| `click`、`mousedown` | 可以 | 会冒泡 |
| `focus`、`blur` | **不行** | 不冒泡，改用 `focusin` / `focusout` |
| `load`、`error` | 不行 | 不冒泡 |

⚠ 委托 `focus` / `blur` 是高频坑：写了 `$('#form').on('focus', 'input', fn)` 却永远不触发，就是因为这两个事件不冒泡。

# 4. 命名空间

## 4.1 给事件分类

```javascript
// 同一元素上多方绑定的 click，用命名空间区分来源
$('#btn').on('click.moduleA', function () { /* ... */ })
$('#btn').on('click.moduleB', function () { /* ... */ })

$('#btn').off('click.moduleA')   // 只解绑 moduleA
$('#btn').off('.moduleA')        // 解绑该命名空间下所有类型

// 命名空间可以叠加
$('#btn').on('click.ui.analytics', fn)
$('#btn').off('.analytics')      // 命中
```

## 4.2 组件化场景

```javascript
const Modal = {
  open() {
    $(document).on('keydown.modal', function (e) {
      if (e.key === 'Escape') Modal.close()
    })
  },
  close() {
    $(document).off('keydown.modal')   // 只清自己的监听
  }
}
```

★ **关键认知**：命名空间的本质是**给监听打标签**，让"清理"可以按来源批量执行，而不是靠保存一堆函数引用。凡是绑在 `document`、`window` 上的全局监听，都应该带命名空间，否则组件销毁时极难清理干净。

# 5. trigger 与自定义事件

## 5.1 触发事件

```javascript
$('#btn').trigger('click')      // 触发原生事件，会冒泡并执行默认行为

// 传额外参数
$('#btn').on('custom', function (e, a, b) {
  console.log(a, b)             // "hello" 42
})
$('#btn').trigger('custom', ['hello', 42])
```

| 特性 | `.trigger()` | `.triggerHandler()` |
|------|-------------|---------------------|
| 执行默认行为 | 是 | 否 |
| 事件冒泡 | 是 | 否 |
| 触发所有匹配元素 | 是 | 只触发第一个 |
| 返回值 | jQuery 对象 | 最后一个处理器的返回值 |

⚠ `.trigger('click')` 会真的触发链接跳转、表单提交等默认行为。只想执行绑定的回调，用 `.triggerHandler()`。

# 6. ready

## 6.1 ready 在等什么

```javascript
// 三种等价写法
$(document).ready(function () { /* ... */ })
$(function () { /* ... */ })
jQuery(function ($) { /* ... */ })   // 参数形式，避免 $ 冲突
```

`ready` 等待的是 **DOM 构建完成**，不等待图片、样式、iframe。它对应原生 `DOMContentLoaded`，通常远早于 `window.onload`。

```javascript
// 所有资源（图片等）加载完才触发
$(window).on('load', function () { /* ... */ })
```

## 6.2 ready 与 load 的时序

```text
HTML 解析完成
   │
   ├── DOMContentLoaded / $(ready)   ← 可以安全操作 DOM
   │
   ├── 图片、样式、子框架继续加载
   │
   └── window.load / $(window).on('load')
```

★ **关键认知**：只要脚本放在 `<body>` 末尾，DOM 已经就绪，`ready` 并非必需；但放在 `<head>` 或由构建工具注入时，`ready` 是安全网。现代做法是把 `<script>` 放末尾或加 `defer`。

# 7. 常见坑

| 坑 | 表现 | 解决 |
|----|------|------|
| 重复绑定 | 点一次触发多次 | 先 `off` 再 `on`，或委托只绑一次 |
| `focus` 委托失效 | 回调不执行 | 改用 `focusin` |
| 匿名函数无法解绑 | `off` 无效 | 用具名函数/保存引用 |
| 事件里用箭头函数 | `this` 不是元素 | 用 `function` 或 `e.currentTarget` |
| 返回 `false` | 同时阻止默认与冒泡 | 显式调用 `preventDefault` / `stopPropagation` |
| 全局事件未清理 | 页面切换后泄漏 | 命名空间 + 销毁时 `off` |

# 8. 练习

## 8.1 练习一：动态待办列表

输入框回车添加条目，点击条目切换完成状态，点击条目内的"删除"按钮移除该条。要求只在一个父容器上绑定事件。

## 8.2 练习二：命名空间清理

写一个 `Tooltip` 模块：`init()` 时在 `document` 上绑定 `click.tooltip` 与 `keydown.tooltip`，`destroy()` 时一次性清理自己绑定的所有事件，不影响页面其他 `click` 监听。

## 8.3 练习三：ready 与 load

在页面中分别记录 `$(function(){})` 与 `$(window).on('load', ...)` 的执行时间，放入一个含大图的页面，观察两者相差多少毫秒，并解释差异来源。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 事件流 | 捕获 → 目标 → 冒泡；jQuery 默认在冒泡阶段 |
| 绑定 | `on(events, selector, data, handler)`，推荐统一用 `on` |
| 解绑 | 必须传同一函数引用；命名空间可批量清理 |
| 委托 | 监听挂父容器，动态元素天然生效 |
| 命名空间 | `click.ns`，给监听打标签，按来源解绑 |
| trigger | `trigger` 会冒泡并执行默认行为，`triggerHandler` 不会 |
| ready | 等 DOM 就绪，早于 `window.load` |

下一篇进入效果与动画，讲 `show/hide/fade/slide` 的底层、`animate` 的数值规则、队列的执行顺序，以及 `stop()` 为什么常常要传两个参数。
