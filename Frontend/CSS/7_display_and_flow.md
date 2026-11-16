CSS 布局的起点,是理解"元素默认怎么流动"——**文档流**。`display` 决定一个元素是块还是行内,`float` 让元素绕排(也埋下一堆坑)。本篇讲清 display 的关键取值、块级/行内/行内块的差异、float 的用途与"清除浮动"的来龙去脉。

# 1. display 关键取值

## 1.1 block / inline / inline-block

```css
.block       { display: block; }         /* 块级:独占一行,可设宽高 */
.inline      { display: inline; }        /* 行内:随文字流,宽高取决于内容 */
.inline-block { display: inline-block; } /* 行内块:行内排列,但可设宽高 */
```

★ **关键认知**：三者差异——`block` **独占一行、可设宽高**;`inline` **随文字流动、宽高由内容决定、上下 margin/padding 影响有限**;`inline-block` 是**折中**:像行内一样排列,但**可以设宽高**(常用于水平排列的按钮/卡片)。⚠ `inline` 元素设 `width/height` 无效,想"行内但能定尺寸"就用 `inline-block`。

## 1.2 none 与 flex/grid

```css
.hidden { display: none; }   /* 彻底移除:不占位、不渲染(连 DOM 都在但视觉消失) */
.flex   { display: flex; }   /* 弹性布局(第 8 篇) */
.grid   { display: grid; }   /* 网格布局(第 9 篇) */
```

★ **关键认知**：`display: none` **彻底移除**(不渲染、不占位、不可见);与此对照,`visibility: hidden` 只是**隐藏但仍占位**。⚠ "隐藏"两兄弟要分清:`display:none`(不占空间)、`visibility:hidden`(占空间)、`opacity:0`(占空间且可交互误点)。flex/grid 是现代的**布局模式**,将在第 8、9 篇展开。

# 2. 文档流与格式化上下文

## 2.1 正常流(normal flow)

★ **关键认知**：**正常流(normal flow)** 是元素默认的排布规则——块级元素**自上而下**堆叠,行内元素**自左向右**流动(放不下换行)。⚠ 改变布局的手段(浮动、定位、flex、grid)本质上都是**让元素"脱离"或"重组"正常流**;理解正常流,才能理解各种布局方式"动了什么手脚"。

## 2.2 块级格式化上下文(BFC)预告

★ **关键认知**：块级元素在一个**BFC**(Block Formatting Context)里垂直排列;BFC 决定"内部如何排、外边距怎么算、浮动怎么包"。⚠ BFC 的完整机制、触发方式、以及它如何解决"外边距合并""浮动画破"两大难题,见第 15 篇专章——这里先知道"有这么个东西"。

# 3. float 浮动

## 3.1 float 的原始用途

```html
<p>
  <img src="pic.jpg" style="float: left; width: 100px;">
  这段文字会绕着图片排列……(文字环绕是 float 的本意)
</p>
```

★ **关键认知**：`float` 的本意是**文字环绕**——让图片浮到一边,文字绕排。元素设 `float: left/right` 后**脱离正常流**,向左/右浮动,后续行内内容环绕它。⚠ float 曾被滥用来做**整页多栏布局**,如今已基本被 flex/grid 取代——它现在**只该用于"文字环绕"**这个原始场景。

## 3.2 float 的副作用:父容器塌陷

```css
.parent { border: 1px solid; }   /* 父:没有高度 */
.child  { float: left; }         /* 子:全浮起,脱离文档流 */
/* 结果:父容器高度塌陷成 0,边框只剩一条线 */
```

★ **关键认知**：float 元素**脱离文档流**,父容器若没有其它内容撑高,会**高度塌陷**——这是 float 最著名的坑。⚠ 因为父元素"看不见"浮动的子元素,不知道要给它们留多高;解决之道就是下面要讲的"清除浮动"。

# 4. 清除浮动

## 4.1 clear 属性

```css
.clearfix::after {
  content: "";        /* 伪元素必须 */
  display: block;     /* 块级 */
  clear: both;        /* 清除左右浮动 */
}
```

★ **关键认知**：`clear: both` 让元素**避开浮动、不被环绕**,从而把父容器"撑开"到能包住浮动子元素。⚠ `clear` 作用于**浮动元素之后的元素**(或用 `::after` 造一个),经典 **clearfix** 三件套=`content` + `display: block` + `clear: both`。

## 4.2 现代替代

```css
/* 现代解法一:父元素触发 BFC,自动包裹浮动(第 15 篇) */
.parent { overflow: hidden; }   /* 或 display: flow-root */

/* 现代解法二:父元素用 flex/grid,根本不需要 float */
.parent { display: flex; }
```

★ **关键认知**：清除浮动有更现代的替代——**让父元素建 BFC**(`overflow: hidden` 或 `display: flow-root`)自动包裹浮动,或**直接用 flex/grid 布局**(浮动问题不复存在)。⚠ 老项目的 clearfix 是历史记忆,新代码里"用 float 做布局"本身就该避免,浮动的坑自然消失。

# 5. 本篇小结与下一步

★ **关键认知**：display 决定块/行内/行内块(flex/grid 是布局模式);正常流是元素默认排布;float 只该用于文字环绕,它会脱离流、导致父塌陷,靠 clear 或 BFC/现代布局化解。布局的现代正道,是下一篇的 Flexbox。

下一步第 8 篇:Flexbox——主轴/交叉轴、对齐,以及 grow/shrink/basis 的弹性计算。