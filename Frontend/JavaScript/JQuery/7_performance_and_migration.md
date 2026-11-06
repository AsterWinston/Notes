jQuery 的性能问题，很少是"jQuery 本身慢"，而是**用法把它用慢了**：在循环里反复查 DOM、每次追加都触发重排、给上千个元素逐个绑事件。迁移也一样，难点不在 API 对照，而在"如何在不推倒重来的前提下，把风险控制在可回滚的范围内"。

# 1. 性能问题的来源

## 1.1 重排、重绘与合成

```text
布局 Layout（算位置尺寸） → 绘制 Paint（填像素） → 合成 Composite（图层合并）
```

| 操作 | 触发阶段 | 成本 |
|------|---------|------|
| 改 `width`、`height`、`top`、`left`、`margin` | 布局 + 绘制 + 合成 | 高 |
| 改 `color`、`background` | 绘制 + 合成 | 中 |
| 改 `transform`、`opacity` | 合成 | 低 |

★ **关键认知**：性能优化的核心不是"少写代码"，而是**少触发布局**。读取 `offsetHeight`、`getBoundingClientRect()`、`$(el).width()` 会强制浏览器立刻算一次布局；读写交替进行，就会形成 **layout thrashing（布局抖动）**。

```javascript
// 差：每次循环都"写样式 → 再读尺寸"，反复强制布局
$('.item').each(function () {
  $(this).css('width', $(this).width() + 10)
})

// 好：先集中读，再集中写
const widths = $('.item').map(function () { return $(this).width() }).get()
$('.item').each(function (i) { $(this).css('width', widths[i] + 10) })
```

## 1.2 jQuery 的隐式成本

| 写法 | 隐式行为 | 成本 |
|------|---------|------|
| `$('.item')` | 调 `querySelectorAll` + 包装 | 一次查询 |
| `$(this)` 在循环里 | 每次都新建 jQuery 对象 | 高频时明显 |
| `.append()` 在循环里 | 每次插入都可能重排 | 很高 |
| `:visible` / `:animated` | 逐个元素读布局属性 | 很高 |

```javascript
// 差：1000 次 $(this) 包装
$('.item').each(function () { $(this).addClass('x') })

// 好：直接用原生 classList
$('.item').each(function () { this.classList.add('x') })
```

⚠ 不是说"jQuery 方法不能用"，而是**在热点循环里**，能退到原生就退到原生。普通页面差异可忽略；上千条数据渲染时会放大到肉眼可见。

# 2. 选择器优化

## 2.1 从右向左匹配

```javascript
$('#list .item')
```

浏览器先找**所有** `.item`，再向上检查祖先是否是 `#list`。所以右侧越具体，候选集越小：

```javascript
$('.wrap *')              // 差：右侧通配，候选集是全部元素
$('#list').find('.item')  // 好：先 ID 锁定范围，再 find
```

## 2.2 优先使用原生可加速的选择器

| 选择器 | 能否被原生加速 | 建议 |
|--------|---------------|------|
| `#id` | 能，最快 | 优先 |
| `.class` | 能 | 常用 |
| `[attr="v"]` | 能 | 可用 |
| `:first`、`:eq()` | 不能，走 Sizzle | 热路径避免 |
| `:visible`、`:hidden` | 不能，且读布局 | 热路径避免 |

```javascript
$('.row:visible')      // 慢：逐个判断可见性
$('.row.is-visible')   // 快：用类名标记状态
```

★ **关键认知**：jQuery 扩展伪类（`:eq`、`:first`、`:visible`、`:contains`）无法下推给 `querySelectorAll`，只能由 Sizzle 逐级过滤。**把状态写成类名**，既能让选择器走原生快路径，也让 CSS 与 JS 的职责更清晰。

## 2.3 缓存集合

```javascript
// 差：每次点击都重新查
$('#btn').on('click', function () { $('.item').addClass('active') })

// 好：模块初始化时查一次
const $items = $('.item')
$('#btn').on('click', function () { $items.addClass('active') })
```

⚠ 缓存的前提是**元素不会被整体替换**。如果容器内容会被 `html()` 重写，缓存的集合就指向了已被移除的旧节点，操作不会生效。

# 3. 批量 DOM 操作

## 3.1 减少插入次数

```javascript
// 差：100 次 append，每次都可能触发重排
for (let i = 0; i < 100; i++) {
  $('#list').append('<li>' + i + '</li>')
}

// 好：拼成一次插入
const html = Array.from({ length: 100 }, (_, i) => `<li>${i}</li>`).join('')
$('#list').append(html)
```

数据来自用户输入时，不要拼 HTML 字符串，改用"离线构建 + 一次插入"：

```javascript
const $fragment = $(document.createDocumentFragment())

data.forEach(function (item) {
  $('<li>').text(item.name).appendTo($fragment)   // text 自动转义
})

$('#list').append($fragment)   // 只触发一次插入
```

## 3.2 批量改样式

```javascript
// 差：多次改样式
$('.box').css('width', 200).css('height', 100).css('background', 'red')

// 好：一次传入对象；更好的是交给 CSS 类
$('.box').css({ width: 200, height: 100, background: 'red' })
```

## 3.3 事件委托替代批量绑定

```javascript
// 差：1000 个 li 绑 1000 个监听
$('#list li').on('click', function () { $(this).toggleClass('done') })

// 好：只在容器上绑一个
$('#list').on('click', 'li', function () { $(this).toggleClass('done') })
```

| 方式 | 监听数量 | 动态元素 | 内存 |
|------|---------|---------|------|
| 逐个绑定 | 等于元素数 | 需重新绑定 | 高 |
| 事件委托 | 1 | 自动生效 | 低 |

★ **关键认知**：事件委托同时解决了**性能**与**动态元素**两个问题。⚠ 但委托不是万能的：`focus`、`blur` 不冒泡，要改用 `focusin`/`focusout`；委托选择器也不要过宽。
# 4. 动画、AJAX 与内存

动画优先用 CSS 过渡 + `transform`/`opacity`，避免用 jQuery 动画化 `width`、`left` 这类持续触发重排的属性。搜索类 AJAX 要防抖并取消上一次请求。移除元素时 `.remove()` 会清理 jQuery 数据与事件，`.detach()` 会保留；但绑在 `window`、`document` 上的全局监听不会自动清理，必须手动 `off('.namespace')`。

# 5. 迁移到原生

## 5.1 先测量，再动手

```text
迁移前先拿四个数字：jQuery 占比、首屏延迟、依赖文件数、插件依赖数。
```

没有数据的迁移，很容易变成"花两周删掉 jQuery，性能没变，bug 变多"。

## 5.2 常用 API 对照

| jQuery | 原生替代 |
|--------|---------|
| `$('#id')` | `document.getElementById('id')` |
| `$('.item')` | `document.querySelectorAll('.item')` |
| `$(el)` | 直接使用 `el` |
| `.addClass()` | `el.classList.add()` |
| `.toggleClass()` | `el.classList.toggle()` |
| `.text()` | `el.textContent` |
| `.html()` | `el.innerHTML`（注意 XSS） |
| `.val()` | `el.value` |
| `.on('click', fn)` | `addEventListener('click', fn)` |
| `.off()` | `removeEventListener()` |
| 事件委托 | 父元素监听 + `e.target.closest()` |
| `$.ajax` | `fetch` + `AbortController` |
| `$(fn)` | `DOMContentLoaded` 或 `defer` |
| `$.Deferred` | `Promise` |
| `.animate()` | CSS transition / Web Animations |

```javascript
// 迁移示例：事件委托
// jQuery
$('#list').on('click', 'li', function () { this.classList.toggle('done') })

// 原生
document.getElementById('list').addEventListener('click', function (e) {
  const li = e.target.closest('li')
  if (li && this.contains(li)) li.classList.toggle('done')
})
```

★ **关键认知**：迁移不是"逐行翻译"，而是**换一种组织方式**。jQuery 的隐式迭代、链式调用、自动事件清理，在原生里都要显式处理。先写一个小工具函数承接高频操作，再逐步替换，通常比直接硬改更稳。

## 5.3 渐进式改造

```text
┌──────────────────────────────────────────┐
│ 旧 jQuery 区域  │  新原生/框架区域（新功能）│
└──────────────────────────────────────────┘
```

新功能用新写法，旧功能在"正好要改"时顺手迁移，每次一小块，保证可回滚，用测试或截图对比兜底。

# 6. 迁移到 Vue / React

## 6.1 边界原则

**同一棵 DOM 子树只能有一个"所有者"**。Vue/React 管理的节点，不要让 jQuery 去改；jQuery 插件管理的节点，也不要让框架去 diff。

```javascript
// Vue 3 示意：Vue 只提供容器，内部交给插件
onMounted(function () {
  instance = $(container.value).somePlugin()
})
onBeforeUnmount(function () {
  if (instance) instance.destroy()   // 必须销毁
})
```

React 对应的是 `useEffect`：初始化放进去，返回清理函数调用 `instance.destroy()`。

⚠ 不要在 `render` 期间调用 jQuery，也不要用 jQuery 去改 React 管理的节点。两者对 DOM 的假设不同，混用会产生"状态与界面不一致"的随机 bug。

## 6.2 分阶段路径

| 阶段 | 动作 | 风险 |
|------|------|------|
| 1 | 新建页面用 Vue/React，旧页面保留 jQuery | 低 |
| 2 | 抽出公共组件，替换重复的 jQuery 模块 | 低 |
| 3 | 按路由/页面整体迁移，避免同页混用 | 中 |
| 4 | 移除 jQuery 依赖与插件 | 中 |

★ **关键认知**：迁移到框架的最大风险不是语法，而是**状态所有权**。坚持"一个区域一个所有者"，即使 jQuery 和 Vue 长期共存也不会有大问题；反过来，为了赶进度让两边同时改同一块 DOM，后期的 bug 成本远高于重写。

# 7. 维护老项目的建议

| 建议 | 原因 |
|------|------|
| 锁定 jQuery 与插件版本 | 避免自动升级引入不兼容 |
| 建立依赖清单 | 知道哪些插件还在维护 |
| 给关键交互补测试 | 迁移时才有回归保障 |
| 统一请求封装 | 便于从 `$.ajax` 换到 `fetch` |
| 禁止新增全局 `$` 依赖 | 控制迁移成本不再扩大 |
| 定期审计 XSS | 老代码常用 `html()` 拼用户输入 |

```javascript
// 统一请求入口，未来替换实现时只改一处
const api = {
  get(url, params) { return $.getJSON(url, params) },
  post(url, data) { return $.ajax({ url, method: 'POST', data: JSON.stringify(data) }) }
}
```

# 8. 常见坑

| 坑 | 表现 | 解决 |
|----|------|------|
| 读写交替 | 布局抖动 | 先读后写 |
| 大量逐个绑定 | 内存高、动态元素失效 | 事件委托 |
| 缓存了会被替换的集合 | 操作无效 | 重新查询或委托 |
| 迁移动了 DOM 所有权 | 状态错乱 | 一区域一所有者 |

# 9. 练习

## 9.1 练习一：优化渲染

下面代码渲染 500 条数据，请指出至少三个性能问题并改写：

```javascript
for (let i = 0; i < 500; i++) {
  const $li = $('<li>').text('item ' + i)
  $('#list').append($li)
  $li.on('click', function () { $(this).toggleClass('done') })
}
```

## 9.2 练习二：原生改写

把一段 jQuery 代码（选择、加类、委托点击、发请求）改写成原生 DOM + `fetch`，保持行为一致，并说明哪些地方需要手动清理。

## 9.3 练习三：迁移方案

选一个你熟悉的 jQuery 页面，列出：① 依赖 jQuery 的功能清单；② 哪些能直接换原生；③ 哪些需要包成 Vue/React 组件；④ 第一周可以安全迁移的最小范围。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 性能本质 | 少触发布局，避免读写交替 |
| 选择器 | 右侧具体、优先原生可加速语法、缓存集合 |
| 批量 DOM | 一次插入、批量改样式、事件委托 |
| 动画 | 优先 transform/opacity + CSS |
| 内存 | 全局监听要手动解绑，remove 会清理数据 |
| 原生迁移 | 先测量，工具函数过渡，逐块替换 |
| 框架迁移 | 一区域一所有者，插件包组件并销毁 |
| 老项目 | 锁版本、补测试、收口请求、控制新增依赖 |

下一篇是模块收尾：用高频自检题把前面的知识点串起来，补上与 Vue/React 的对比，并给出维护老项目的行动清单与整个模块的知识地图。
