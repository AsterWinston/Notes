网页并非孤岛——`iframe` 镶嵌别的页面,`canvas` 画位图,`svg` 画矢量,`template` 囤积可复用的片段。本篇讲清这四种"嵌入与组合"技术各自的定位、用法与坑,并厘清它们与现代组件化(Web Components)的边界。

# 1. iframe 内嵌页面

## 1.1 基本用法与属性

```html
<!-- 嵌入另一个页面 -->
<iframe src="https://example.com" width="600" height="400"
        title="嵌入的内容描述"></iframe>
```

★ **关键认知**：`iframe` 把另一个完整文档**嵌入**当前页,有独立的浏览上下文(独立 window/document)。⚠ `title` 对 iframe **几乎是必填**——它是读屏用户"要不要进去"的唯一依据,没有 title 读屏只会播报"框架"。

## 1.2 安全与限制

```html
<!-- sandbox:沙箱,限制 iframe 的能力(脚本、表单、跳转等) -->
<iframe src="..." sandbox="allow-scripts allow-same-origin"></iframe>

<!-- loading="lazy":懒加载 iframe,首屏性能 -->
<iframe src="..." loading="lazy"></iframe>
```

★ **关键认知**：`sandbox` 属性把 iframe 内容**关进沙箱**,默认禁用脚本/表单/弹窗等能力,可按需 `allow-*` 放开;`loading="lazy"` 让 iframe 进入视口才加载。⚠ 嵌入**不信任的第三方内容**时,`sandbox` 是防 XSS 的关键防线;`allow-same-origin` 与 `allow-scripts` 同时给会大幅削弱沙箱防护,慎用组合。

# 2. canvas 位图画布

## 2.1 画布与上下文

```html
<canvas id="c" width="300" height="150"></canvas>
```

```javascript
const canvas = document.getElementById('c')
const ctx = canvas.getContext('2d')   // 拿到 2D 绘图上下文
ctx.fillStyle = '#42b883'
ctx.fillRect(10, 10, 100, 50)         // 画一个矩形
```

★ **关键认知**：`canvas` 提供一个**可编程的位图区域**,绘图全靠 JS——获取上下文(`2d` 或 `webgl`)后调用绘图 API。⚠ 它是**位图**(缩放会模糊),且**画上去的东西不产生 DOM 节点**——供 JS 动态绘图(图表、游戏、图像处理),不适合用来做"可选中可交互"的 UI(那是 DOM 的活)。

## 2.2 canvas 的可访问性

```html
<!-- canvas 里的图形屏幕阅读器读不到,需提供 fallback 文本 -->
<canvas id="chart">
  图表内容:2025 年销售额 120 万元
  你的浏览器不支持 canvas。
</canvas>
```

★ **关键认知**：canvas 内的内容对读屏**完全不可见**,标签里的文字是 fallback。⚠ 用 canvas 做图表时,**同步提供文字描述/数据表格**作替代,否则视障用户拿不到任何信息——这是 canvas 最大的可访问性短板。

# 3. svg 矢量图形

## 3.1 内联 svg

```html
<!-- svg 是矢量:无限缩放不模糊,且每个形状都是真实元素 -->
<svg width="100" height="100">
  <circle cx="50" cy="50" r="40" fill="#42b883" />
</svg>
```

★ **关键认知**：`svg` 画**矢量图**——缩放永不模糊,且**每个图形都是 DOM 元素**(可用 CSS 样式、JS 操作、可加 title 做可访问)。⚠ 与 canvas 的对比:**图标、插画用 svg**(清晰 + 可交互 + 可访问),**大量动态像素绘图用 canvas**。内联 svg 还避免了额外的图片请求。

## 3.2 svg 的可访问与使用方式

```html
<!-- 直接作 img 引用 -->
<img src="icon.svg" alt="图标">

<!-- 内联:可加 <title>/aria,可被 CSS 着色 -->
<svg role="img" aria-label="放大镜">
  <title>放大镜</title>
  <path d="..."/>
</svg>
```

★ **关键认知**：svg 有三种用法——`<img>` 引用(简单)、**内联**(可 CSS/JS 控制、可加 title/aria)、CSS 背景。⚠ 需要**图标变色**(跟随主题)时内联;纯静态装饰用 img + alt。内联 svg 记得加 `role="img"` + `aria-label` 保证可访问。

# 4. template 片段

## 4.1 不渲染的片段

```html
<!-- template 的内容不会渲染,可被 JS 克隆后插入 -->
<template id="card-tpl">
  <div class="card">
    <h3>标题</h3>
    <p>内容</p>
  </div>
</template>
```

```javascript
const tpl = document.getElementById('card-tpl')
const clone = tpl.content.cloneNode(true)   // 克隆内容
document.body.appendChild(clone)             // 插入文档
```

★ **关键认知**：`<template>` 是**惰性片段**——内容不渲染、不加载资源,专门囤积"可复用的 HTML 结构",由 JS 按需 `cloneNode` 复制后再插入。⚠ 它是"原生 HTML 模板"的机制,配合 `slot` 构成 Web Components 的基础设施(Vue 的 `<template>` 与之概念同源)。

# 5. 与现代组件化的边界

## 5.1 Web Components 的一瞥

```html
<template id="my-card-tpl">
  <style>…</style>
  <div><slot name="title"></slot></div>
</template>

<script>
  class MyCard extends HTMLElement {
    constructor() {
      super()
      const tpl = document.getElementById('my-card-tpl')
      this.attachShadow({ mode: 'open' })
        .appendChild(tpl.content.cloneNode(true))
    }
  }
  customElements.define('my-card', MyCard)
</script>

<!-- 使用自定义元素 -->
<my-card><span slot="title">卡片标题</span></my-card>
```

★ **关键认知**：**Web Components** 由三件套构成——`customElements`(自定义标签)、`<template>`+`<slot>`(结构模板)、`Shadow DOM`(样式隔离)——它是**浏览器原生**的组件化。⚠ 与 Vue/React 的关系:**前者是语言级组件**(跨框架通用),后者是框架级组件(生态更完善);本系列前端框架部分(第 13 篇及 Vue/React 系列)会在此边界上展开。

# 6. 本篇小结与下一步

★ **关键认知**：iframe(嵌页面,注意 sandbox+title)、canvas(位图绘图,注意可访问)、svg(矢量图形,可交互可着色)、template(惰性片段,组件化基石)。选择上:**图标 svg、动态绘图 canvas、嵌第三方 iframe、复用片段 template**。

下一步进入**阶段四 · 工程与性能**,第 12 篇:资源加载与性能——preload/prefetch、defer/async、懒加载与关键渲染路径。