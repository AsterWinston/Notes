jQuery 是前端历史上绕不开的名字：它曾经是"写网页"的同义词，如今却常被贴上"过时"的标签。本篇不站队，而是把它的定位、崛起原因、3.x 现状，以及它与原生 DOM、Vue、React 的真实边界讲清楚——只有先弄明白它解决的是什么问题，才能判断今天该不该用它。

# 1. jQuery 是什么

## 1.1 一句话定位

jQuery 是一个**操作 DOM、处理事件、发送 AJAX、做动画**的 JavaScript 工具库。它不提供组件化，不提供响应式，也不规定项目结构，本质上是把浏览器原生 API 中"啰嗦、碎片化、兼容性差"的部分，包装成一套**统一而简短**的接口。

```javascript
// 原生写法：找元素、遍历、改样式，三步都要自己写
const items = document.querySelectorAll('.item')
items.forEach((el) => el.classList.add('active'))

// jQuery 写法：选择器 + 方法，一行完成，且自动遍历所有匹配元素
$('.item').addClass('active')
```

★ **关键认知**：jQuery 的核心价值不是"某个炫酷功能"，而是**用一套一致的抽象抹平浏览器差异**。它返回的"jQuery 对象"是一个类数组包装器，绝大多数方法都会**自动对集合里的每个元素执行一遍**，这被称为隐式迭代。理解这一点，就理解了一半的 jQuery。

## 1.2 它解决了什么痛点

2006 年前后的前端环境，和今天完全不同：

| 痛点 | 当年的处境 | jQuery 的回应 |
|------|-----------|--------------|
| 选择元素 | `getElementById` 能用，复杂选择全靠手写遍历 | 引入 CSS 选择器语法 `$('#id .item')` |
| 浏览器兼容 | IE6/7/8 各有各的怪癖 | 内部做特性检测与兼容分支，对外一套 API |
| 事件绑定 | `attachEvent` 与 `addEventListener` 并存 | `.on()` 统一绑定与委托 |
| AJAX | `XMLHttpRequest` 样板代码冗长 | `$.ajax()` 与 `$.get()` 封装 |
| 动画 | 只能靠定时器手写 | `.animate()` 与内置效果 |
| DOM 增删改 | 大量属性判断与字符串拼接 | `.append()`、`.addClass()` 等语义化方法 |

★ **关键认知**：jQuery 的成功是**时代红利 + 工程抽象**的叠加。它诞生于浏览器碎片化最严重的年代，"一次编写、各处运行"的诱惑是刚需；而它的 API 设计又足够克制——选择器复用 CSS 语法，方法名基本是英文动词，学习成本极低。

## 1.3 版本线与 3.x 现状

| 版本 | 特点 | 现状 |
|------|------|------|
| 1.x | 兼容 IE6/7/8，体积最大 | 仅老项目维护 |
| 2.x | 移除 IE6-8 兼容，体积变小 | 已停止维护 |
| 3.x | 现代浏览器，重写 Deferred、动画用 `requestAnimationFrame` | **当前主线，3.7.1 为常用稳定版** |
| 4.x | 进一步精简旧浏览器兼容与废弃 API | 新版本推进中，迁移需评估 |

截至本文写作时，生产项目使用 jQuery 3.x 应优先选择 **3.7.1** 或更高的 3.x 补丁版本。⚠ 不要因为"反正只是操作 DOM"就随手引入一个来源不明的旧版 1.x：旧版本存在已公开的安全问题，且与现代构建工具的配合更差。

# 2. 历史价值与设计取舍

## 2.1 兼容层是它最厚重的部分

很多人以为 jQuery 只是"简写"，其实它大量代码花在**兼容与兜底**上：事件系统的跨浏览器差异、`className` 与 `classList` 的取舍、`getComputedStyle` 的返回值差异、旧 IE 的盒模型问题，都被封在内部。

```javascript
// 开发者看到的
$('#box').css('width', 100)

// 内部要处理：单位补全、数值与字符串、批量设置、返回值读取等
// 读模式下返回计算后的样式，写模式下返回 jQuery 对象以支持链式调用
```

★ **关键认知**：jQuery 的 API 有一个鲜明的"**读写二相性**"——同一个方法，传参数是写，不传参数是读。`.css('width')` 读，`.css('width', 100)` 写；`.text()` 读，`.text('hi')` 写；`.attr()`、`.val()`、`.prop()` 全是这个套路。记住这条规律，API 就不用死记。

## 2.2 "写得少、做得多"的代价

抽象带来便利，也带来不透明：

| 收益 | 代价 |
|------|------|
| 语法短、开发快 | 隐式迭代让人忘记真实元素数量 |
| 兼容性好 | 包体积约 30KB(gzip)，对轻页面偏重 |
| 插件生态庞大 | 插件质量参差，容易引入连锁依赖 |
| 链式调用流畅 | 出错时调用栈深，定位不如原生直观 |

## 2.3 与原生 DOM 的关系

jQuery 不是替代浏览器，而是**建在原生 API 之上的一层壳**。它最终仍然调用 `querySelectorAll`、`addEventListener`、`appendChild`、`XMLHttpRequest` 或 `fetch` 时代的等价物（3.x 的 AJAX 仍以 XHR 为主）。

```javascript
// jQuery 对象可以随时"退回"原生 DOM
const $box = $('#box')       // jQuery 对象
const el = $box[0]           // 原生 DOM 元素（下标访问）
const same = $box.get(0)     // 同上，语义更明确

// 原生元素可以重新"包装"成 jQuery 对象
const $again = $(el)
```

★ **关键认知**：`$box` 和 `el` 是两种完全不同的东西。`$box.addClass` 存在，`el.addClass` 不存在；`el.style` 存在，`$box.style` 不存在。**混用是新手最常见的错误来源**，写代码时要时刻清楚手里拿的是哪一类。

# 3. 与原生、Vue、React 的对比

## 3.1 四者定位

| 维度 | 原生 DOM | jQuery | Vue | React |
|------|---------|--------|-----|-------|
| 本质 | 浏览器 API | DOM 工具库 | 渐进式框架 | UI 库 |
| 更新方式 | 命令式，手动改 | 命令式，手动改 | 声明式，数据驱动 | 声明式，数据驱动 |
| 数据与视图 | 无关联 | 无关联 | 自动同步 | 自动同步 |
| 组件化 | 无 | 无 | 内置 | 内置 |
| 路由/状态 | 无 | 无 | 官方生态 | 社区生态 |
| 典型体积 | 0 | ~30KB gzip | ~35KB+ | ~45KB+ |
| 上手成本 | 低但琐碎 | 很低 | 中 | 中高 |
| 适合场景 | 少量交互、教学 | 老项目、服务端渲染页面增强 | 中大型应用 | 中大型应用 |

## 3.2 同一需求的三种写法

需求：点击按钮，让计数器加一，并把结果显示在页面上。

```javascript
// ===== 原生 DOM：自己找元素、自己改文本 =====
let count = 0
const btn = document.querySelector('#btn')
const output = document.querySelector('#output')
btn.addEventListener('click', () => {
  count += 1                    // 1. 改数据
  output.textContent = count    // 2. 手动同步视图
})
```

```javascript
// ===== jQuery：选择器更短，但同步逻辑一模一样 =====
let count = 0
$('#btn').on('click', function () {
  count += 1                    // 1. 改数据
  $('#output').text(count)      // 2. 手动同步视图
})
```

Vue 只描述"视图依赖哪个数据"：

```html
<button @click="count++">{{ count }}</button>
```

★ **关键认知**：原生与 jQuery 属于**命令式**——你得记住"数据变了以后，哪些 DOM 要跟着改"，漏一处就出 bug；Vue/React 属于**声明式**——你只改数据，框架负责把变化映射到 DOM。jQuery 并不"低级"，它只是把命令式写得更顺手，没有改变命令式的本质。

## 3.3 jQuery 仍然合理的场景

| 场景 | 原因 |
|------|------|
| 维护中的老项目 | 大规模重写成本高，渐进改造更现实 |
| 服务端渲染的多页应用 | 页面由后端输出，只需少量交互增强 |
| CMS / WordPress 主题 | 生态默认带 jQuery，随大流成本最低 |
| 一次性活动页、Demo | 引入成本低，不值得上构建链 |
| 需要支持老旧内嵌浏览器 | 现代框架对旧环境支持有限 |

⚠ 判断标准不是"jQuery 老不老"，而是"**这个页面的状态复杂度有多高**"。如果页面只有开合菜单、Tab 切换、表单校验这类局部交互，jQuery 依然高效；如果要做购物车、聊天、仪表盘这类状态交织的界面，硬用 jQuery 会把自己拖进"手动同步地狱"。

# 4. 什么时候不该用 jQuery

## 4.1 新项目默认不该引入

今天的主流浏览器已经统一支持 `querySelector`、`classList`、`fetch`、`addEventListener`、`Promise`。jQuery 当年抹平的差异基本消失，新项目再引入它，多数时候只是多了一层依赖。

```javascript
// 现代原生写法已经足够简洁
document.querySelectorAll('.item').forEach((el) => {
  el.classList.toggle('active')
})
```

## 4.2 状态复杂的单页应用

当界面由多份数据共同决定时，jQuery 的手动同步会指数级变难：

```text
数据 A 变 → 要改视图 1、2、3
数据 B 变 → 要改视图 2、4
数据 C 变 → 要改视图 1、5、6
```

一旦漏掉某条连线，界面就和数据不一致。Vue/React 的响应式或虚拟 DOM 正是为消灭这类连线而生的。

## 4.3 性能敏感的大列表

jQuery 的选择器与隐式迭代在几千个节点上仍可用，但频繁的 `.append()`、`.css()` 会触发大量重排。现代框架的批量更新、虚拟列表，或原生 `DocumentFragment` 会更合适。

★ **关键认知**：**"能用 jQuery"不等于"该用 jQuery"**。技术选型看的是团队、项目生命周期与状态复杂度，而不是"我会不会"。⚠ 反过来也一样：在老项目里为了"技术先进"强行塞入 Vue，往往会得到两套状态体系并存的烂摊子。

# 5. 最小可运行示例

## 5.1 通过 CDN 引入

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>jQuery 最小示例</title>
</head>
<body>
  <button id="toggle">切换高亮</button>
  <p class="tip">点一下按钮试试。</p>

  <script src="https://code.jquery.com/jquery-3.7.1.min.js"></script>
  <script>
    // $ 是 jQuery 的别名，这里传入函数等价于 $(document).ready(...)
    $(function () {
      $('#toggle').on('click', function () {
        // toggleClass 对集合中每个元素切换类名，无需自己遍历
        $('.tip').toggleClass('highlight')
      })
    })
  </script>
</body>
</html>
```

## 5.2 示例拆解

```javascript
$(function () {
  // 1. $ 是函数，也是对象：既能 $(selector) 造集合，也能 $.ajax 调工具方法
  // 2. 传入函数时，它在 DOM 就绪后执行，等价于原生 DOMContentLoaded
  // 3. $('#toggle') 返回 jQuery 对象，不是原生按钮
  // 4. .on('click', fn) 绑定事件，this 在回调里指向原生 DOM 元素
  // 5. .toggleClass() 返回同一个集合，所以还能继续链式调用
})
```

★ **关键认知**：jQuery 的入口只有两个身份——**`$(...)` 造集合**与 **`$.xxx(...)` 调工具**。前者面向页面元素，后者面向全局能力（AJAX、类型判断、工具函数）。把这两个身份分清，读任何 jQuery 代码都不会迷路。

# 6. 常见误区

| 误区 | 纠正 |
|------|------|
| "jQuery 是一门语言" | 它是 JavaScript 库，语法仍是 JS |
| "用了 jQuery 就不用懂 DOM" | 出问题时仍要回到原生 API 排查 |
| "jQuery 已经死了" | 新项目占比下降，但存量系统与 CMS 仍在大量使用 |
| "jQuery 一定比原生慢" | 简单操作差距很小，瓶颈通常在频繁重排与不当选择器 |
| "$ 在任何页面都能用" | 可能被其他库占用，需用 `jQuery.noConflict()` |

⚠ `$` 只是一个普通变量名。如果页面同时引入了 Prototype、Zepto 或某些老插件，`$` 可能被覆盖，表现为"明明引入了 jQuery 却报 `$ is not a function`"。排查时先看 `window.jQuery` 是否存在。

# 7. 练习

## 7.1 练习一：原生与 jQuery 对照

用原生 DOM 和 jQuery 各写一遍：给所有 `.card` 元素添加 `is-active` 类，并统计数量打印到控制台。

```javascript
// 原生版本提示：querySelectorAll 返回 NodeList，需要遍历
// jQuery 版本提示：addClass 会自动遍历，length 可直接读数量
```

## 7.2 练习二：判断选型

以下三个需求，分别判断更适合 jQuery 还是 Vue/React，并说明理由：

1. 一个由后端模板渲染的商品列表页，需要点击"加入购物车"后弹出轻提示。
2. 一个带筛选、排序、分页、实时搜索的后台数据表格。
3. 一个只有"展开/收起"的 FAQ 静态页面。

## 7.3 练习三：识别对象类型

```javascript
const a = document.querySelector('#box')
const b = $('#box')
console.log(a.addClass)   // 输出什么？为什么？
console.log(b.addClass)   // 输出什么？为什么？
console.log(b[0] === a)   // 输出什么？
```

先写出答案，再在浏览器控制台验证。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 定位 | DOM 操作工具库，不是框架 |
| 核心抽象 | jQuery 对象 + 隐式迭代 + 链式调用 |
| 历史价值 | 抹平浏览器差异，极大降低 DOM 操作成本 |
| 3.x 现状 | 维护主线，3.7.1 常用；4.x 逐步推进 |
| 适用场景 | 老项目、服务端渲染页面增强、轻交互 |
| 不适用场景 | 状态复杂的 SPA、大列表高频更新 |
| 选型标准 | 看状态复杂度与项目生命周期，而非"新旧" |

下一篇进入 jQuery 的选择器与 DOM 操作，讲清楚 `$()` 到底返回什么、jQuery 对象与原生 DOM 如何互转，以及增删改查、遍历与链式调用的完整用法。
