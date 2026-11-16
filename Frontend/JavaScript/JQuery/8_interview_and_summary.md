前面七篇分别讲了定位、选择器与 DOM、事件、动画、AJAX、插件、性能与迁移。这一篇做三件事：用一组高频自检题帮你查漏，用对比表把 jQuery 与 Vue/React 的差异钉死，最后给出一份维护老项目的行动清单和整个模块的知识地图。

# 1. 怎么用这一篇

建议先**遮住答案**，逐题口述；能说清"是什么 + 为什么 + 什么时候会踩坑"，才算真正掌握。遇到卡壳的题，回到对应篇目复习。

# 2. 高频自检题

## 2.1 jQuery 对象与原生 DOM 有什么区别

jQuery 对象是**类数组包装器**，可能包含 0 个、1 个或多个原生 DOM；原生 DOM 是浏览器提供的真实节点。jQuery 对象有 `.addClass()`、`.on()`，原生 DOM 有 `.style`、`.classList`。两者通过下标或 `$()` 显式互转，`$el.style` 与 `el.addClass` 都不存在。

## 2.2 $() 可以接收哪些参数

```javascript
$(function () {})          // 1. 函数：DOM 就绪后执行
$('#id')                   // 2. 选择器字符串
$('<li>新条目</li>')        // 3. HTML 字符串：创建元素
$(domElement)              // 4. 原生 DOM：包装
$(jQueryObject)            // 5. jQuery 对象：复制集合
```

★ **关键认知**：`$` 是函数也是对象。`$(...)` 负责"造集合"，`$.ajax(...)`、`$.extend(...)` 负责"调工具"，两个身份不要混。

## 2.3 ready 与 window.onload 的区别

`ready` 等的是 **DOM 构建完成**（对应 `DOMContentLoaded`），此时可以安全查询和操作 DOM；`window.onload` 等的是**所有资源加载完成**，包括图片、样式、iframe，通常更晚。

```javascript
$(function () { console.log('DOM 就绪') })
```

⚠ `ready` 可以绑定多次，都会执行；原生 `window.onload = fn` 后赋值会覆盖前一个。

## 2.4 on 的参数与事件委托原理

签名是 `.on(events, [selector], [data], handler)`。传了 `selector` 就是委托：监听挂在**父容器**上，事件冒泡到容器后，jQuery 用 `e.target` 向上匹配选择器，命中才执行回调。

```javascript
$('#list').on('click', 'li', function (e) {
  // this：匹配到的 li（原生）
  // e.target：实际点击的最深元素，可能是 li 里的 span
  // e.delegateTarget：#list
})
```

★ **关键认知**：委托的价值有两个——**动态元素自动生效**、**大量元素只绑一个监听**。⚠ `focus`、`blur` 不冒泡，委托要改用 `focusin`、`focusout`。

## 2.5 为什么 off 有时解绑不了

`.off(type, handler)` 必须传入**同一个函数引用**。匿名函数每次都是新对象，无法匹配：

```javascript
// 解绑失败：匿名函数不是同一个引用
$('#btn').on('click', function () {})
$('#btn').off('click', function () {})

// 正确：保存引用，或按命名空间批量解绑
const handler = function () {}
$('#btn').on('click.moduleA', handler)
$('#btn').off('.moduleA')
```

## 2.6 链式调用的原理是什么

写操作的方法返回 `this`（当前 jQuery 对象），所以能继续调用下一个方法。`.find()`、`.filter()` 这类方法会改变集合，jQuery 内部维护一个**集合栈**，`.end()` 可以回到上一层。

```javascript
$('#box')
  .find('span')       // 集合变成 span
  .addClass('red')
  .end()              // 回到 #box
  .addClass('active')
```

⚠ 并不是所有方法都返回 jQuery 对象：`.get()`、读模式的 `.text()`/`.val()` 返回具体值，链在这里就断了。

## 2.7 attr 与 prop 怎么选

`attr` 读写 HTML 特性（attribute），`prop` 读写 DOM 属性（property）。**布尔状态**（`checked`、`selected`、`disabled`）要用 `prop`，因为用户交互改的是 property，不是 attribute。

```javascript
$('#agree').prop('checked')          // 实时状态，布尔值
$('#agree').attr('checked')          // HTML 初始值，可能是 undefined
$('#link').attr('href')              // 普通特性用 attr 即可
```

## 2.8 remove、detach、empty 的区别

| 方法 | 移除什么 | 保留事件与数据 |
|------|---------|---------------|
| `.remove()` | 元素本身及子节点 | 否 |
| `.detach()` | 元素本身及子节点 | 是 |
| `.empty()` | 所有子节点，保留自身 | 否（子节点数据被清理） |

需要"暂时摘下来，稍后插回去且事件还在"，用 `detach()`；彻底不要了用 `remove()`。

## 2.9 动画为什么会排队，stop 怎么用

同一元素上的动画进入 `fx` 队列，串行执行。快速点击会不断往队列里塞动画，表现为"越点越慢"。

```javascript
$('#panel').stop(true, true).slideToggle(300)
```

- 第一个参数 `clearQueue`：清空未执行的队列。
- 第二个参数 `jumpToEnd`：当前动画直接跳到终点。

`.finish()` 与 `.stop(true, true)` 的区别是：`finish` 会完成队列里的**所有**动画并触发各自回调；`stop(true, true)` 只完成当前一个，其余丢弃。

## 2.10 jqXHR 与原生 Promise 有什么区别

`$.ajax()` 返回 jqXHR，它既是 XHR 包装，又有 Promise 风格接口。相比原生 Promise，它多了 `.abort()`、`.done()`、`.fail()`、`.always()`、`progress`，但部分方法返回原对象而不是新 Promise。

```javascript
const xhr = $.ajax('/api/user')
xhr.done(render).fail(showError)
xhr.abort()                 // 原生 Promise 没有

// 需要串联时用 then，返回值会传给下一个 then
$.ajax('/api/user/1')
  .then(function (user) { return $.ajax('/api/orders/' + user.id) })
  .catch(showError)
```

★ **关键认知**：`.then()` 返回新 promise，可以转换值、串联请求、统一捕获异常；`.done()`/`.fail()` 只是监听，不能转换。

## 2.11 选择器性能从哪来

浏览器解析选择器是**从右向左**的，右侧越具体，候选集越小。此外，jQuery 扩展伪类（`:eq`、`:first`、`:visible`、`:contains`）无法交给原生 `querySelectorAll`，只能走 Sizzle 逐个过滤。

```javascript
$('.wrap *')              // 差
$('#list').find('.item')  // 好
$('.row.is-visible')      // 用类名表达状态，比 :visible 快
```

## 2.12 写一个 jQuery 插件的要点

四件事——挂 `$.fn`、遍历 `this.each`、返回 `this`、用 `data` 存状态。

```javascript
(function ($) {
  $.fn.myPlugin = function (options) {
    const settings = $.extend({}, $.fn.myPlugin.defaults, options)
    return this.each(function () {
      $(this).data('myPlugin', { settings: settings })
    })
  }
  $.fn.myPlugin.defaults = { color: 'red' }
})(jQuery)
```

⚠ `$.extend(defaults, options)` 会污染默认值，第一个参数必须传 `{}`；事件要带命名空间，销毁时才能只清理自己。

## 2.13 jQuery 与 Vue/React 的本质区别

jQuery 是**命令式 DOM 工具库**，你负责"数据变了以后改哪些 DOM"；Vue/React 是**声明式 UI 框架**，你只描述"视图依赖什么数据"，框架负责把变化映射到 DOM。前者 `count++; $('#output').text(count)`，后者只写 `count.value++`，模板里的 `{{ count }}` 自动更新。

## 2.14 什么时候不该用 jQuery

新项目、状态复杂的 SPA、大列表高频更新、已有成熟框架的团队，都不建议再引入 jQuery。反过来，老项目维护、服务端渲染的多页应用、CMS 主题、轻交互活动页，jQuery 仍然高效。判断标准不是"新旧"，而是**状态复杂度**与**迁移成本**。

# 3. 与现代框架对比

## 3.1 心智模型

| 维度 | jQuery | Vue 3 | React |
|------|--------|-------|-------|
| 编程范式 | 命令式 | 声明式 | 声明式 |
| 数据与视图 | 手动同步 | 响应式自动追踪 | 状态 + 重新渲染 |
| DOM 所有权 | 开发者 | 框架 | 框架 |
| 组件化 | 无 | SFC / 组合式函数 | 函数组件 + Hooks |
| 事件 | `.on()` 委托 | 模板 `@click` | JSX `onClick` |
| 动画 | `animate` + 队列 | `<Transition>` + CSS | CSS / 动画库 |
| 包体积 | ~30KB gzip | ~35KB+ | ~45KB+ |
| 适合 | 局部交互、老项目 | 中大型应用 | 中大型应用 |

## 3.2 同一功能的两种写法

需求：列表渲染 + 点击切换完成状态。

```javascript
const tasks = [{ text: '学习', done: false }]

function render() {
  const $list = $('#list').empty()
  tasks.forEach(function (task, index) {
    $('<li>').text(task.text).toggleClass('done', task.done)
      .attr('data-index', index).appendTo($list)
  })
}

$('#list').on('click', 'li', function () {
  const index = $(this).data('index')
  tasks[index].done = !tasks[index].done
  render()                    // 手动重渲染
})
```

```html
<ul>
  <li v-for="(task, index) in tasks" :key="index"
      :class="{ done: task.done }" @click="task.done = !task.done">
    {{ task.text }}
  </li>
</ul>
```

★ **关键认知**：jQuery 版本里，"改数据"和"改界面"是两件事，必须记得同时做；Vue/React 版本里只改数据，界面由框架推导。**代码量的差距会随状态复杂度迅速拉大**。

## 3.3 选择建议

页面状态简单、交互零散时，原生或 jQuery 都行；状态由多份数据共同决定时，选 Vue/React；介于两者之间，原生 + 少量工具函数通常足够。

# 4. 维护老项目的建议

## 4.1 安全

| 风险 | 场景 | 处理 |
|------|------|------|
| XSS | `.html(userInput)`、字符串拼 HTML | 优先 `.text()`，必要时转义 |
| 危险 URL | `attr('href', userInput)` | 校验协议，禁止 `javascript:` |
| 旧版漏洞 | jQuery 1.x / 老插件 | 升级到 3.7.1+，移除废弃插件 |
| CDN 劫持 | 外链脚本 | 使用 SRI（子资源完整性）校验 |

```javascript
$('#box').html(userInput)                    // 危险
$('#box').text(userInput)                    // 安全
$('#box').html(DOMPurify.sanitize(userInput)) // 需要富文本时先清洗
```

## 4.2 稳定性

```text
老项目稳定运行的四条底线：
1. 锁版本：jQuery 与插件写死版本号
2. 收口请求：所有 AJAX 走统一封装
3. 事件命名空间：全局监听必须可清理
4. 补测试：迁移前给关键路径加回归用例
```

## 4.3 渐进迁移

```text
┌──────────────────────────────────────────┐
│ 旧 jQuery 区域  │  新框架区域（新功能）    │
└──────────────────────────────────────────┘
```

- **新功能用新写法**，不再扩大 jQuery 的依赖面。
- **旧功能在改动时顺手迁移**，不做纯为迁移而迁移的重构。
- **一区域一所有者**：Vue/React 管理的 DOM，不要让 jQuery 去改。

⚠ 最常见的翻车方式是"为了赶进度，让框架和 jQuery 同时操作同一棵子树"。短期能跑，长期一定会出现"界面和数据对不上"的随机 bug。

# 5. 模块知识地图

```text
jQuery 模块
├── 1 定位：DOM 工具库，不是框架
├── 2 选择与 DOM：$()、读写二相性、链式/end
├── 3 事件：on/off、委托、命名空间、ready
├── 4 动画：show/fade/slide、animate、队列、stop
├── 5 AJAX：$.ajax、jqXHR、then/when、abort
├── 6 插件：$.fn、默认参数、data、命名冲突
├── 7 性能与迁移：布局、批量 DOM、原生/框架
└── 8 自检：对比、维护、总结
```

# 6. 练习

## 6.1 练习一：闭卷自测

不看答案，口述以下五个问题，并用代码验证：① `get(0)` 与 `eq(0)` 的区别；② `attr('checked')` 与 `prop('checked')` 的区别；③ 事件委托中 `this`、`e.target`、`e.delegateTarget` 各指什么；④ `.stop(true, true)` 的两个参数；⑤ `$.when` 中一个请求失败，其他请求会不会被取消。

## 6.2 练习二：老代码体检

找一段项目里的 jQuery 代码，逐项检查：循环内是否查 DOM / append、是否大量逐个绑定、是否用了 `:visible` / `:eq`、是否有 `html()` 拼接用户输入、全局监听是否带命名空间、请求是否有超时与取消。

## 6.3 练习三：迁移评估

选一个页面，写一份一页纸的迁移评估：依赖清单、风险点、可先迁移的最小范围、回滚方案、验证方式。要求至少包含一个"不迁移"的理由。

# 7. 小结

| 主题 | 最该记住的一句话 |
|------|-----------------|
| 定位 | jQuery 是命令式 DOM 工具库，不是框架 |
| 选择器 | 右侧具体、缓存集合、少用扩展伪类 |
| DOM | 读写二相性；`get` 原生、`eq` jQuery |
| 事件 | 委托解决动态元素与性能；命名空间解决清理 |
| 动画 | 队列串行；快速交互先 `stop(true, true)` |
| AJAX | jqXHR 可 abort；`then` 能串联，`done` 只监听 |
| 插件 | 返回 `this`、默认值不污染、状态放 data |
| 性能 | 少触发布局，批量读写，委托代替海量绑定 |
| 迁移 | 先测量、一区域一所有者、渐进可回滚 |

jQuery 的历史使命已经完成，但它留下的设计思想——选择器抽象、链式 API、事件委托、插件扩展——仍然值得学习。读到这里，如果你还想深入，建议接着看 `Frontend/JavaScript/Browser` 里的原生 DOM 与事件篇，把 jQuery 帮你屏蔽掉的底层细节补回来；再进入 `Frontend/Vue` 或 React 系列，理解声明式 UI 如何接管"数据到视图"的同步工作。
