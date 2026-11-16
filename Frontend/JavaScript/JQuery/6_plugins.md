jQuery 的生态之所以庞大，靠的不是核心有多强，而是 `$.fn` 这个"谁都能往上挂方法"的扩展点。写一个合格插件并不难，难的是写得**不污染全局、不破坏链式、能反复初始化、能干净销毁**。本篇从一个 20 行的骨架讲起，把默认参数、状态存放、命令模式与命名冲突逐层补齐。

# 1. 插件机制

## 1.1 两个挂载点

| 挂载点 | 调用形式 | 适合放什么 |
|--------|---------|-----------|
| `$.fn`（即 `jQuery.prototype`） | `$('#el').myPlugin()` | 针对元素集合的操作 |
| `$`（jQuery 本身） | `$.myUtil()` | 与元素无关的工具函数 |

```javascript
// 实例方法：作用于选中的元素
$.fn.red = function () {
  return this.css('color', 'red')
}
$('.item').red()

// 静态工具：与 DOM 无关
$.sum = function (arr) {
  return arr.reduce(function (a, b) { return a + b }, 0)
}
$.sum([1, 2, 3])   // 6
```

★ **关键认知**：`$.fn` 就是 `jQuery.prototype`。给 `$.fn` 加方法，等于给所有 jQuery 对象加方法；给 `$` 加属性，等于加一个全局工具。

## 1.2 命名约定

```javascript
// 推荐：所有能力收在一个名字下
$.fn.myPlugin = function () {}
$.fn.myPlugin.defaults = {}
$.myPlugin = { version: '1.0.0', format: function () {} }
```

插件内部事件也应带命名空间：`click.myPlugin`，销毁时用 `off('.myPlugin')` 精确清理。

# 2. 第一个插件

## 2.1 最小骨架

```javascript
// IIFE 包起来：内部 $ 一定指向 jQuery，避免被其他库覆盖
(function ($) {
  'use strict'

  $.fn.highlight = function (color) {
    // this 是 jQuery 对象，可能包含多个元素
    return this.each(function () {
      // this 是当前原生 DOM 元素
      this.style.backgroundColor = color || 'yellow'
    })
  }
})(jQuery)
```

```javascript
$('.item').highlight('lightblue')
```

## 2.2 逐行解释

- IIFE 把 `jQuery` 作为参数传入，函数内部的 `$` 是局部变量，即使外部 `$` 被占用也不受影响。
- `'use strict'` 开启严格模式，避免隐式全局变量等低级错误。
- 方法挂在 `$.fn` 上，所有 jQuery 对象都能调用。
- 用 `this.each()` 遍历集合，保证多元素场景正确；回调里的 `this` 是原生 DOM。
- `each` 返回原集合，所以插件也返回 `this`，链式不断。

## 2.3 链式返回

```javascript
// 因为返回了 this，可以继续调用其他方法
$('.item').highlight().addClass('marked').fadeIn()
```

⚠ 忘记 `return this` 是插件最常见的破坏性 bug：调用方写 `.highlight().addClass(...)` 会直接报 `Cannot read properties of undefined`。**任何"写操作"插件都应该返回 `this`**。

# 3. 参数与默认值

## 3.1 $.extend 合并

```javascript
(function ($) {
  'use strict'

  $.fn.tooltip = function (options) {
    const settings = $.extend({}, $.fn.tooltip.defaults, options)

    return this.each(function () {
      const $el = $(this)
      $el.attr('title', settings.text || $el.attr('title'))
      $el.css({ color: settings.color, fontSize: settings.fontSize })
    })
  }

  $.fn.tooltip.defaults = {
    color: '#333',
    fontSize: '14px',
    text: ''
  }
})(jQuery)
```

```javascript
$('.tip').tooltip()                    // 使用默认值
$('.tip').tooltip({ color: 'red' })    // 覆盖部分默认值
```

★ **关键认知**：`$.extend({}, defaults, options)` 的第一个参数必须是**空对象**，表示把默认值和选项合并进新对象，从而不修改 defaults 本身。⚠ 写成 `$.extend(defaults, options)` 会**原地污染默认值**：第一次调用传了 `color: 'red'`，之后所有不传 color 的调用都会变红。

## 3.2 深合并

```javascript
const defaults = { style: { color: 'red', size: 14 } }

// 深合并：递归合并嵌套对象，保留 color
const deep = $.extend(true, {}, defaults, { style: { size: 20 } })
```

| 写法 | 行为 | 适用 |
|------|------|------|
| `$.extend({}, a, b)` | 浅拷贝，嵌套对象整体替换 | 配置项都是扁平值 |
| `$.extend(true, {}, a, b)` | 深拷贝，嵌套对象递归合并 | 有嵌套配置 |

# 4. 状态管理

## 4.1 用 data 存实例

```javascript
(function ($) {
  'use strict'

  function Plugin(element, options) {
    this.$el = $(element)
    this.count = 0
    this.init()
  }

  Plugin.prototype.init = function () {
    const self = this
    this.$el.text(this.count)
    this.$el.on('click.counter', function () {
      self.count += 1                 // 用 self 保住实例引用
      self.$el.text(self.count)
    })
  }

  Plugin.prototype.destroy = function () {
    this.$el.off('.counter')          // 只解绑自己的事件
    this.$el.removeData('counter')    // 清掉实例
  }

  $.fn.counter = function (options) {
    return this.each(function () {
      let instance = $(this).data('counter')
      if (!instance) {
        instance = new Plugin(this, options)
        $(this).data('counter', instance)
      }
    })
  }
})(jQuery)
```

```javascript
$('#btn').counter()                    // 初始化
$('#btn').data('counter').destroy()    // 销毁
```

★ **关键认知**：`.data('counter', instance)` 把实例"挂"在元素上，是 jQuery 插件保存状态的标准做法。它随元素一起被清理（`remove()` 时），不会像全局变量那样泄漏；同时支持"同一元素只初始化一次"的判断。⚠ 不要用 `$('#btn')[0]._counter = instance` 这类直接挂属性的方式，既不规范，也容易和别的库冲突。

## 4.2 data 的读写细节

```javascript
$('#el').data('count', 1)
$('#el').data('count')          // 1
$('#el').removeData('count')    // 删除
```

⚠ `.data()` 首次读取 `data-*` 后会缓存。之后用 `attr('data-count', 2)` 改 HTML 特性，`.data('count')` 仍可能返回旧值，两者不要混用。

# 5. 命令模式

插件常需要 `$('#el').modal('show')` 这样的方法调用。做法是把第一个参数当命令：对象或无参走初始化，字符串走方法调用。

```javascript
$.fn.modal = function (option) {
  const args = Array.prototype.slice.call(arguments, 1)

  return this.each(function () {
    const $el = $(this)
    // 假设 Modal 已在别处定义；首次调用传对象，字符串调用只取已有实例
    const instance = $el.data('modal') || new Modal(this, typeof option === 'object' ? option : {})
    $el.data('modal', instance)

    if (typeof option === 'string' && typeof instance[option] === 'function') {
      instance[option].apply(instance, args)
    }
  })
}
```

★ **关键认知**：命令模式的关键是**区分第一次调用和后续调用**。⚠ 调用方法前必须判断 `instance[option]` 是否存在，否则用户写错方法名会静默失败，极难排查。

# 6. 命名冲突与 noConflict

## 6.1 命名空间

把插件所有能力收在 `myPlugin` 这个名字下，事件也带命名空间：`this.$el.on('click.myPlugin', handler)`，销毁时用 `off('.myPlugin')` 精确清理。

## 6.2 noConflict

```javascript
// 让出 $，交给之前占用它的库
const jq = $.noConflict()
jq('#box').addClass('active')     // 用 jq 代替 $
jQuery('#box').addClass('active') // jQuery 这个名字仍可用

// 连 jQuery 变量也让出（多版本共存）
const myJQ = jQuery.noConflict(true)
```

插件内部完全不受影响：

```javascript
(function ($) {
  // 这里的 $ 是参数，不是全局变量
})(jQuery)
```

★ **关键认知**：`noConflict()` 解决的是**全局变量占用**，不是"两个 jQuery 版本互相干扰"。真正的多版本共存要靠 `noConflict(true)` 把当前版本完全交出，再各自用不同变量持有。⚠ 现代项目更推荐用模块打包隔离作用域，而不是依赖 `noConflict`。

# 7. 插件类型与边界

## 7.1 常见插件类型

| 类型 | 代表 | 说明 |
|------|------|------|
| UI 组件 | jQuery UI、Bootstrap 4 | 日期、弹窗、下拉等 |
| 表格 | DataTables | 排序、分页、筛选 |
| 选择器增强 | Select2、Chosen | 搜索、多选、远程数据 |
| 校验 | jQuery Validation | 表单规则校验 |

## 7.2 什么时候不该用插件

```text
引入一个插件前，先问四个问题：
1. 这个效果用 CSS / 原生 API 能不能在 20 行内解决？
2. 插件是否还在维护？最近一次提交是什么时候？
3. 它依赖哪些库？会不会和现有 jQuery 版本冲突？
4. 它是否直接拼接 HTML？有没有 XSS 风险？
```

⚠ 老插件常见的三个隐患：**停止维护导致的安全漏洞**、**依赖旧版 jQuery**、**直接操作 `innerHTML` 带来的 XSS**。引入前至少看一眼源码里有没有 `html(`、`eval`、`innerHTML` 这类危险调用。

如果项目已经迁移到 Vue/React，不要再引入新的 jQuery 插件。正确做法是找对应框架的原生组件，或把插件包一层指令/组件，明确生命周期与销毁逻辑。

# 8. 常见坑

| 坑 | 表现 | 解决 |
|----|------|------|
| 忘记 `return this` | 链式调用报错 | 写操作一律返回 `this` |
| `$.extend(defaults, options)` | 默认值被污染 | 第一个参数传 `{}` |
| 默认对象共享引用 | 多个实例互相影响 | 深拷贝或每次新建 |
| 事件不带命名空间 | 销毁时误删别人的监听 | `click.myPlugin` + `off('.myPlugin')` |
| 重复初始化 | 事件叠加，点一次触发多次 | `data` 判断是否已初始化 |
| 用全局变量存实例 | 内存泄漏、互相覆盖 | `.data()` 挂在元素上 |

# 9. 练习

## 9.1 练习一：写一个高亮插件

实现 `$.fn.highlight(options)`：支持 `color`、`duration` 两个配置；作用于多个元素时全部生效；返回 `this` 保证链式调用。

## 9.2 练习二：加上销毁能力

给练习一的插件加 `destroy()`：移除行内背景色、解绑事件、删除 `data`。要求通过 `$('#el').data('highlight').destroy()` 调用。

## 9.3 练习三：命令模式改造

把插件改成 `$('.item').highlight('setColor', 'red')` 与 `$('.item').highlight('reset')` 的命令式接口，并处理"方法名不存在"的情况（抛出明确错误或 `console.warn`）。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 挂载点 | `$.fn` 实例方法，`$` 静态工具 |
| 骨架 | IIFE 包住，内部 `$` 安全 |
| 链式 | 写操作返回 `this` |
| 默认值 | `$.extend({}, defaults, options)`，别污染默认对象 |
| 状态 | 实例存 `.data()`，事件带命名空间 |
| 命令模式 | 对象初始化，字符串调方法 |
| 命名冲突 | 单一命名空间 + `noConflict` |
| 引入边界 | 评估维护状态、依赖、XSS 与替代方案 |

下一篇进入性能与迁移，讲选择器与 DOM 操作的成本从哪来、如何批量更新、事件委托如何替代海量绑定，以及从 jQuery 迁到原生或 Vue/React 的可执行路径。
