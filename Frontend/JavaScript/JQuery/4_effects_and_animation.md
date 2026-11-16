jQuery 的动画 API 是它当年最直观的卖点：`.slideDown()`、`.fadeIn()` 一行就能做出"像样"的交互。但动画一旦和快速点击、队列、`stop()` 混在一起，问题就来了——元素越点越慢、动画排队几十次、回调在错误的时机执行。本篇把效果 API、`animate` 的规则、队列机制和 `stop` 的参数讲透。

# 1. 效果 API 概览

## 1.1 三组基础效果

| 组别 | 方法 | 改的是什么 |
|------|------|-----------|
| 显示隐藏 | `.show()` / `.hide()` / `.toggle()` | `display` + 宽高透明度（带动画时） |
| 淡入淡出 | `.fadeIn()` / `.fadeOut()` / `.fadeToggle()` / `.fadeTo()` | `opacity` + `display` |
| 滑动 | `.slideDown()` / `.slideUp()` / `.slideToggle()` | `height` + `padding` + `margin` + `display` |

```javascript
$('#box').hide()          // 立即隐藏
$('#box').show(400)       // 400ms 内显示
$('#box').fadeOut('slow')
```

## 1.2 速度参数与异步性

```javascript
// duration：毫秒数，或 "fast"(200) / "slow"(600)，默认 400
$('#box').fadeIn(300, function () {
  // 动画结束后执行，this 是原生 DOM
  console.log('淡入完成', this)
})
```

★ **关键认知**：动画是**异步**的。`.fadeOut(300)` 后面的代码会立刻执行，而不是等 300ms 后。需要"结束后再做某事"，放进回调，或用 `.promise().done(...)`：

```javascript
// 错误预期：以为下一行在动画结束后才执行
$('#box').fadeOut(300)
console.log('已经隐藏了吗？')

// 正确
$('#box').fadeOut(300).promise().done(function () {
  console.log('现在才真正结束')
})
```

# 2. show / hide / toggle

## 2.1 无参是瞬时切换

```javascript
$('#box').hide()      // 相当于 css('display', 'none')
$('#box').show()      // 恢复到之前的 display 值
$('#box').toggle()    // 可见则隐藏，隐藏则显示
```

jQuery 会在第一次隐藏时把元素原本的 `display` 记在内部数据里，`show()` 时恢复。原本是 `inline-block` 或 `flex` 的元素不会被改错。

## 2.2 带动画时改的是什么

```javascript
$('#box').show(400)
```

带时长时，jQuery 会先把元素设为可见，再对 `width`、`height`、`opacity`、`padding`、`margin` 做插值，所以要求这些属性是**可计算的数值**。给元素明确宽高并加 `overflow: hidden`，能避免收起时内容溢出。

⚠ 对 `display: none` 且没有显式宽高的元素做 `slideDown`，jQuery 需要先测量尺寸，可能闪烁或高度不准。稳妥做法是给明确尺寸，或改用 CSS 过渡。

# 3. 淡入淡出

```javascript
$('#tip').fadeIn(300)       // opacity 0 → 1，同时恢复 display
$('#tip').fadeOut(300)      // opacity 1 → 0，结束后 display: none

$('#tip').fadeTo(300, 0.5)  // 变到半透明，元素仍然可见
```

★ **关键认知**：`fadeOut` 结束后元素是 `display: none`，不再参与布局；`fadeTo(300, 0)` 结束后元素只是"透明"，仍然占位、仍能接收鼠标事件。做半透明遮罩用 `fadeTo`，做彻底消失用 `fadeOut`。

# 4. 滑动

```javascript
$('#panel').slideDown(300)     // 高度 0 → 原高度
$('#panel').slideUp(300)       // 高度 → 0，结束后隐藏
$('#panel').slideToggle(300)   // 开合切换
```

滑动适合折叠面板、手风琴、下拉菜单。它本质是动画化 `height` 与上下 `padding`、`margin`，因此折叠内容时要加 `overflow: hidden`。

⚠ `slideUp` 会把 `height` 动画到 0，但**不会**处理 `border`。元素若有上下边框，收起后仍会留一条线，需要额外处理。

# 5. animate 自定义动画

## 5.1 数值属性与相对值

```javascript
$('#box').animate({
  left: 200,          // 单位默认 px
  opacity: 0.5        // 无单位属性
}, 400)

$('#box').animate({
  left: '+=50',       // 当前值加 50
  top: '-=20',        // 当前值减 20
  height: 'toggle'    // 在 0 与原值之间切换
})
```

能动画的属性必须是**单个数值**。`left` 要生效，元素必须是定位元素（`position: absolute/relative/fixed`）。

## 5.2 缓动与逐属性配置

```javascript
// 核心只有 "swing"（默认）与 "linear"
$('#box').animate({ left: 200 }, 400, 'linear')

// 每个属性单独指定目标值与 easing
$('#box').animate({
  width: ['toggle', 'swing'],
  opacity: [0.3, 'linear']
}, 400)

// options 对象写法
$('#box').animate({ left: 200 }, {
  duration: 400,
  easing: 'swing',
  complete: function () { console.log('done') }
})
```

★ **关键认知**：jQuery 核心**只支持 `swing` 和 `linear`**。`easeInOutCubic` 之类的名字来自 jQuery UI 或额外插件，直接写在核心 `animate` 里不会生效。需要丰富缓动，优先考虑 CSS `transition-timing-function`。

## 5.3 不能动画的属性

```javascript
// 无效：颜色、transform 都无法插值
$('#box').animate({ backgroundColor: 'red' }, 400)
$('#box').animate({ transform: 'rotate(45deg)' }, 400)

// 有效替代：CSS 类 + transition
$('#box').addClass('rotated')
```

⚠ 颜色动画需要 jQuery UI 的 `color` 模块或专门插件；`transform`、`filter` 这类复合属性核心 `animate` 无法处理。**凡是 `animate` 不支持的效果，都应该交给 CSS**。

# 6. 队列

## 6.1 默认 fx 队列

同一个元素上的动画默认进入名为 `fx` 的队列，**串行执行**：

```javascript
$('#box')
  .slideUp(300)
  .slideDown(300)
  .fadeOut(300)
```

```text
fx 队列：slideUp → slideDown → fadeOut，依次串行，各 300ms
```

非动画方法（`.addClass()`、`.css()`）**不进入队列**，会立即执行：

```javascript
$('#box').slideUp(300).addClass('hidden')
// addClass 立刻生效，不等 slideUp 结束

// 需要延后，用回调
$('#box').slideUp(300, function () {
  $(this).addClass('hidden')
})
```

## 6.2 queue / dequeue / delay

```javascript
// 在 fx 队列中插入一个自定义步骤
$('#box')
  .slideUp(300)
  .queue(function (next) {
    $(this).addClass('hidden')
    next()                 // 必须调用，否则队列卡住
  })
  .slideDown(300)

$('#box').fadeIn(300).delay(1000).fadeOut(300)  // 延迟队列
$('#box').clearQueue()                          // 清空未执行队列
```

★ **关键认知**：`.delay()` 只作用于**队列中的动画**，对 CSS 过渡、普通回调、`setTimeout` 没有替代作用。⚠ 在 `.queue(fn)` 里忘记调用 `next()`，队列会永远停住，后续动画全部不执行。

# 7. stop 与 finish

## 7.1 动画为什么会越点越慢

```javascript
// 用户快速点击 10 次，就排了 10 个动画
$('#btn').on('click', function () {
  $('#panel').slideToggle(400)
})

// 正确：先停掉当前，再开始新的
$('#btn').on('click', function () {
  $('#panel').stop(true, true).slideToggle(400)
})
```

## 7.2 stop 的参数

```javascript
.stop()                  // 停止当前，队列中的继续
.stop(true)              // 停止当前，并清空队列
.stop(true, true)        // 停止、清空，并跳到当前动画终点
```

| 参数 | 含义 | 常见用途 |
|------|------|---------|
| `clearQueue` | 是否清空未执行的队列 | 快速连点必须为 `true` |
| `jumpToEnd` | 是否立即完成当前动画 | 避免停在半透明/半高状态 |

```text
stop(true, true)：
1. 当前动画立即跳到最终状态
2. 清空 fx 队列中尚未执行的动画
3. 不触发当前动画的 complete 回调
```

## 7.3 finish 与 stop 的区别

```javascript
$('#box').finish()   // 停止当前，清空队列，并完成所有动画
```

`.finish()` 会让队列中的每一个动画都跳到终点并触发各自的 `complete`；`.stop(true, true)` 只完成当前一个，其余丢弃。做"立即结束全部动画"用 `finish()`，做"打断并重新开始"用 `stop(true, true)`。

⚠ `.stop()` 默认不调用当前动画的 `complete`。业务逻辑若依赖回调（如动画结束才解除 loading），打断后可能状态卡住。

# 8. 与 CSS 动画取舍

| 维度 | jQuery animate | CSS transition/animation |
|------|----------------|--------------------------|
| 触发方式 | JS 调用，灵活 | 类名/状态切换 |
| 可动画属性 | 单数值 CSS 属性 | 几乎所有可插值属性 |
| transform | 不支持 | 支持 |
| 性能 | 逐帧改行内样式，易触发布局 | 可走合成层，GPU 加速 |
| 与 JS 逻辑耦合 | 强，回调方便 | 需监听 `transitionend` |
| 适合 | 依赖 JS 计算、老项目 | 纯视觉过渡、现代项目 |

```css
/* 现代写法：用类名驱动过渡 */
.panel {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease;
}
.panel.open { max-height: 500px; }
```

```javascript
// 只负责切状态，动画交给 CSS
$('#panel').toggleClass('open')
```

★ **关键认知**：**能用 CSS 表达的视觉过渡，优先用 CSS**；只有"目标值需要 JS 实时计算""必须与队列/回调强耦合"时，才用 jQuery `animate`。⚠ 不要用 jQuery 动画化 `width`、`top`、`left` 这类高频触发重排的属性，低端设备上会明显掉帧。

# 9. 常见坑

| 坑 | 表现 | 解决 |
|----|------|------|
| 快速点击不 stop | 动画排队、越点越慢 | `.stop(true, true)` 后再启动 |
| queue 里忘调 `next()` | 后续动画卡住 | 每个自定义步骤都要 `next()` |
| 以为 `animate` 支持颜色 | 颜色瞬间跳变 | 用 CSS 或颜色插件 |
| 动画 `left` 但元素 static | 位置不变 | 设置 `position` |
| 依赖 `complete` 却被 stop | 状态未复位 | 用 `finish()` 或手动复位 |

# 10. 练习

## 10.1 练习一：手风琴

三个面板，点击标题展开当前面板并收起其他面板。要求快速连点时不出现动画堆积。

## 10.2 练习二：自定义队列

让一个方块依次执行：右移 100px → 停顿 200ms → 变半透明 → 回到原位。要求全部通过队列串起来，而不是嵌套回调。

## 10.3 练习三：CSS 改造

把 `$('#box').animate({ width: 300, opacity: 1 }, 300)` 改写成 CSS 过渡，JS 只负责切换类名，并说明改造后哪些行为需要额外处理。

# 11. 小结

| 主题 | 要点 |
|------|------|
| 基础效果 | show/hide 改 display，fade 改 opacity，slide 改高度 |
| 异步性 | 动画立即返回，后续逻辑放回调或 promise |
| animate | 只支持单数值属性；核心只有 swing/linear |
| 队列 | 同一元素动画串行；非动画方法不排队 |
| stop | `(clearQueue, jumpToEnd)` 两个开关决定行为 |
| finish | 完成全部队列，与 stop 的"丢弃"不同 |
| 取舍 | 视觉过渡优先 CSS，JS 计算/队列耦合才用 animate |

下一篇进入 AJAX 与 Deferred，讲 `$.ajax` 的完整参数、`then/done/fail` 的差异、并发请求的组织方式，以及如何正确取消一个正在进行的请求。
