jQuery 的日常代码，八成都在做两件事：**找到元素**、**改动元素**。`$()` 看似简单，但它返回的东西、增删改 API 的语义、链式调用的边界，都有不少容易踩的细节。本篇把选择器、jQuery 对象、DOM 操作、遍历与性能串成一条线。

# 1. 选择器

## 1.1 基础与层级

jQuery 直接借用 CSS 选择器语法，会写 CSS 就会选元素。

```javascript
$('#app')            // ID，至多一个
$('.item')           // 类，返回所有匹配
$('p')               // 标签
$('h1, h2')          // 并集
$('ul li')           // 后代，任意层级
$('ul > li')         // 子代，仅直接子元素
$('h2 + p')          // 相邻兄弟，紧跟的第一个
$('h2 ~ p')          // 一般兄弟，之后所有同级
```

★ **关键认知**：`$()` 永远返回 **jQuery 对象**，不是原生 DOM。即使只匹配一个元素，拿到的也是长度为 1 的集合。所以 `$('#app').style` 是 `undefined`，必须写 `$('#app')[0].style` 或 `.css()`。

## 1.2 属性、伪类与表单

```javascript
$('[data-id]')             // 拥有属性
$('[type="checkbox"]')     // 属性等于
$('[href^="https"]')       // 开头
$('[href$=".pdf"]')        // 结尾
$('[class*="btn"]')        // 包含子串
$('[name!="email"]')       // 不等于，jQuery 私有扩展

$('li:first')              // 集合第一个
$('li:last')               // 最后一个
$('li:eq(2)')              // 下标 2（从 0 开始）
$('li:contains("文本")')    // 文本包含
$(':hidden')

$(':input')                // 所有表单控件
$(':checked')              // 选中的 checkbox/radio
$(':selected')             // 选中的 option
```

★ **关键认知**：`:eq()`、`:first`、`:contains()`、`:visible` 是 **jQuery 扩展伪类**，由 Sizzle 解析，无法交给原生 `querySelectorAll` 加速；`:first-child`、`:nth-child()`、`:not()` 是 CSS 标准，能被原生处理。名字像，性能不在一个量级。

# 2. jQuery 对象

## 2.1 类数组结构

```javascript
const $list = $('li')
$list.length          // 元素数量
$list[0]              // 第一个原生 li
$list.toArray()       // 转真数组
```

## 2.2 与原生 DOM 互转

```javascript
// jQuery → 原生
const $box = $('#box')
const el1 = $box[0]
const el2 = $box.get(0)        // get 支持负数，get(-1) 取最后一个

// 原生 → jQuery
const $box2 = $(document.getElementById('box'))
const $items = $(document.querySelectorAll('.item'))  // NodeList 可整体包装
```

```javascript
// 错误：原生 API 不认 jQuery 对象
$('#btn').addEventListener('click', fn)   // TypeError

// 正确
$('#btn')[0].addEventListener('click', fn)
$('#btn').on('click', fn)
```

⚠ `.get(0)` 返回原生 DOM，`.eq(0)` 返回 jQuery 对象，`.first()` 等价于 `.eq(0)`。写 `.get(0).addClass()` 一定报错；想继续链式调用必须用 `.eq()`。这是高频面试题。

# 3. DOM 操作

## 3.1 读与写

jQuery 方法普遍遵循**读写二相性**：不传参是读，传参是写。

```javascript
$('#msg').text()              // 读纯文本
$('#msg').text('你好')         // 写，内部标签被清掉
$('#box').html()              // 读 HTML
$('#box').html('<b>加粗</b>')  // 写，注意 XSS
$('#name').val('张三')         // 写表单值
$('a').attr('href')           // 读 HTML 特性
$('a').attr('href', '/home')  // 写
$('#agree').prop('checked')   // 读实时状态
```

★ **关键认知**：`attr` 读写 HTML 特性（初始值），`prop` 读写 DOM 属性（当前状态）。`checked`、`selected`、`disabled` 这类布尔状态，用户交互改的是 property，必须用 `prop`。

```javascript
// 错误：用户勾选后仍可能读到初始值
if ($('#agree').attr('checked')) { /* ... */ }

// 正确
if ($('#agree').prop('checked')) { /* ... */ }
```

## 3.2 增加节点

```javascript
const $li = $('<li>').text('新条目')   // HTML 字符串造元素

$('ul').append($li)      // 内部末尾
$('ul').before($li)      // 外部前面，成为兄弟
$('ul').after($li)
```

⚠ 同一元素 `append` 到第二个父节点时，jQuery 会**移动**它，不是复制。要复制先 `.clone()`；`.clone(true)` 连事件一起复制。

## 3.3 删除节点

| 方法 | 行为 | 保留事件与数据 |
|------|------|---------------|
| `.remove()` | 移除元素及子节点 | 否 |
| `.detach()` | 移除但保留数据与事件 | 是 |
| `.empty()` | 清空子节点，保留自身 | 否（子节点数据被清理） |

```javascript
const $item = $('#item')
$item.detach()                 // 稍后还要插回，用 detach
$('#container').append($item)  // 事件仍有效
$item.remove()                 // 彻底不要了
$('#container').empty()        // 只清空内容
```

## 3.4 类名与样式

```javascript
$('.item').addClass('active')
$('.item').removeClass('active')
$('.item').toggleClass('active')
$('.item').hasClass('active')      // 布尔值

$('.box').css({ width: 200, height: '100px', backgroundColor: '#f5f5f5' })
```

★ **关键认知**：能用类名解决的样式，不要用 `.css()` 逐条写。`.css()` 改的是行内样式，优先级高、难覆盖，还会让样式散落在 JS 里；`.addClass()` 把样式留在 CSS 中，可维护性更好。

# 4. 遍历

## 4.1 上下左右找亲戚

```javascript
const $item = $('.item').eq(1)

$item.parent()              // 直接父元素
$item.parents('div')        // 所有祖先，可过滤
$item.closest('.card')      // 从自身向上找最近的匹配，最常用
$item.find('.label')        // 所有后代，最常用
$item.siblings()            // 所有兄弟（不含自身）
$item.next() / $item.prev()
```

★ **关键认知**：`.closest()` 从**自身**开始向上找，`.parents()` 只找祖先。事件委托里判断"点击是否在某个业务元素内"，标准写法是 `$(e.target).closest('.card')`。

## 4.2 过滤与 each

```javascript
$('li').filter('.active')    // 保留匹配
$('li').not('.active')       // 排除匹配

$('li').each(function (index, el) {
  // this 与 el 都是原生 DOM
  $(this).text(index + ': ' + $(el).text())
})

// return false 相当于 break
$('li').each(function () {
  if ($(this).hasClass('stop')) return false
})
```

⚠ `$().each()` 与 `$.each()` 不同：前者遍历 jQuery 对象，回调是 `(index, element)`；后者遍历数组/对象，回调是 `(key, value)`。两者 `return false` 都会中断。

# 5. 链式调用

## 5.1 原理

写操作的方法返回 `this`，所以能一直点下去：

```javascript
$('#box')
  .addClass('active')
  .css('color', 'red')
  .find('span')
  .text('完成')
```

原理很简单：这些方法在完成操作后返回 `this`，也就是当前 jQuery 对象，所以能继续调用下一个方法。

## 5.2 end 与集合栈

`.find()`、`.filter()`、`.children()` 会改变当前集合，jQuery 用栈记录历史，`.end()` 回退上一层：

```javascript
$('#box')
  .find('span')       // 集合变成 span
  .addClass('red')
  .end()              // 回到 #box
  .addClass('active')
```

```text
集合栈：
$('#box')          → [div#box]
  .find('span')    → [span, span]   入栈
  .end()           → [div#box]      出栈
```

⚠ 并非所有方法都返回 jQuery 对象：`.get()`、读模式的 `.text()`/`.val()` 返回具体值，链在这里就断了。链超过 4 到 5 步也建议拆成中间变量，方便调试。

# 6. 性能注意

## 6.1 选择器与缓存

浏览器解析选择器是**从右往左**的，右侧越具体，候选集越小。

```javascript
// 慢：右侧是通配，候选集是全部元素
$('.wrap *')

// 快：先 ID 缩范围，再 find
$('#list').find('.item')

// 差：每次点击都重新查
$('#btn').on('click', function () { $('.item').addClass('x') })

// 好：查一次，复用集合
const $items = $('.item')
$('#btn').on('click', function () { $items.addClass('x') })
```

## 6.2 批量插入

```javascript
// 差：每次 append 都可能触发重排
for (let i = 0; i < 100; i++) {
  $('#list').append('<li>' + i + '</li>')
}

// 好：先拼好，一次性插入
const html = Array.from({ length: 100 }, (_, i) => `<li>${i}</li>`).join('')
$('#list').append(html)
```

⚠ 缓存集合的前提是元素不会被整体替换。如果容器会被 `.html()` 重写，缓存的集合指向旧节点，操作不会生效。

# 7. 常见错误

| 错误写法 | 问题 | 正确写法 |
|---------|------|---------|
| `$('#box').style.color = 'red'` | jQuery 对象没有 `style` | `.css('color', 'red')` |
| `$('.item').get(0).addClass('x')` | `get` 返回原生元素 | `.eq(0).addClass('x')` |
| `$('#agree').attr('checked')` | 读到初始值 | `.prop('checked')` |
| `$('#list').append(li).append(li)` | 同一元素被移动 | 先 `.clone()` |
| `.each` 里用箭头函数 | `this` 不指向元素 | 用 `function` 或第二个参数 |

# 8. 练习

## 8.1 练习一：列表筛选

点击按钮后只保留文本包含"前端"的 `li`，其余隐藏。要求用 `.filter()` 与 `.hide()` 完成，且不手动写 `for` 循环。

## 8.2 练习二：安全插入

把 `data` 数组渲染成表格行，要求只插入一次 DOM，并对文本做转义。提示：用 `$('<td>').text(...)` 造单元格，不要拼字符串后 `html()`。

## 8.3 练习三：链式改写

把下面三段合并成一条链，并用 `.end()` 验证集合切换：

```javascript
const $card = $('.card')
$card.addClass('active')
const $title = $card.find('.title')
$title.text('已激活')
```

# 9. 小结

| 主题 | 要点 |
|------|------|
| 选择器 | 复用 CSS 语法；扩展伪类无法被原生加速 |
| jQuery 对象 | 类数组；`get` 原生、`eq` jQuery |
| 读写二相性 | 传参是写，不传是读；布尔状态用 `prop` |
| 增删节点 | `append` 是移动，复制用 `clone`；暂存用 `detach` |
| 遍历 | `find` 找后代、`closest` 向上找、`each` 逐项 |
| 链式调用 | 方法返回 `this`；`end()` 回退集合栈 |
| 性能 | 缓存集合、右侧具体、批量插入 |

下一篇进入事件系统，讲清楚 `on` 的绑定与委托、命名空间、`trigger`、`ready` 的时机，以及事件对象里必须知道的字段。
