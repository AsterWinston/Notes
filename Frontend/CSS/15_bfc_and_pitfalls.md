写 CSS 越写越会遇到一类"玄学"——外边距怎么没生效、父容器高度怎么塌了、子元素怎么溢出来了。它们大多指向同一个隐藏机制:**格式化上下文(BFC/IFC)**。本篇把 BFC 讲透,顺带拆解外边距折叠、包含块等高频坑,让你从"试一把"升级到"懂原理"。

# 1. 格式化上下文是什么

## 1.1 BFC / IFC

★ **关键认知**：**格式化上下文**是 CSS 排版的"作用域"——元素在某个上下文里,按一套规则被布局。最常见两种:**BFC(Block Formatting Context,块级格式化上下文)** 与 **IFC(Inline,行内)**。⚠ 可以把 BFC 理解成一个"**独立的小世界**":它内部如何排、外边距怎么算、浮动怎么包,**与外界隔离、互不干扰**——很多"玄学"的破解钥匙就在这。

## 1.2 BFC 的三个核心作用

```
BFC 的作用:
1. 包裹浮动(解决父容器塌陷)
2. 阻止外边距折叠(父子/兄弟 margin 合并)
3. 与外界的浮动互不侵入(两栏布局不重叠)
```

# 2. 如何触发 BFC

```css
/* 触发 BFC 的常见方式: */
.a { overflow: hidden; }          /* 最常用 */
.b { display: flow-root; }        /* 最"干净"(专为触发 BFC 而生,无副作用) */
.c { display: inline-block; }
.d { position: absolute; }
.e { float: left; }
.f { display: flex; }             /* flex/grid 容器也是 BFC */
```

★ **关键认知**：触发 BFC 的常用方式——`overflow: hidden`、`display: flow-root`、浮/定位、flex/grid。⚠ **首选 `display: flow-root`**:它唯一目的就是触发 BFC,**没有任何 `overflow:hidden` 那种"裁剪内容"的副作用**;`overflow:hidden` 虽常用,但会悄悄裁掉溢出的内容(那个阴影/角标没了)。

# 3. 用 BFC 解决三大问题

## 3.1 解决父容器塌陷

```css
/* 父容器包住浮动的子元素 */
.parent { overflow: hidden; }   /* 或 display: flow-root */
.child { float: left; }
```

★ **关键认知**：父元素成 BFC 后,**会自动把浮动的子元素"包"进去**(计算高度时纳入浮动)。⚠ 对比第 7 篇的 clearfix 伪元素方案,BFC 方案更简洁——父容器 `flow-root` 一行搞定塌陷。

## 3.2 解决外边距折叠

```html
<div class="parent">
  <p class="child">内边距塌缩出父</p>
</div>
```

```css
/* 子元素的 margin-top 会"穿透"父容器,把父一起往下推 */
/* 解法:父容器触发 BFC */
.parent { overflow: hidden; }
```

★ **关键认知**：**父子之间**首个子元素的 margin 会"溢出"到父元素之外,导致"父没预期地被往下推";父容器触发 BFC 即可阻断。⚠ 这是"给子元素加 margin,结果父元素动了"的经典元凶;同理,改为给父加 padding 也是解法之一。

## 3.3 两栏不重叠

```css
/* 浮动 + BFC 做"左定宽右自适应"(老式做法,现多用 flex) */
.left { float: left; width: 200px; }
.right { overflow: hidden; }   /* 成 BFC,不与左侧浮动重叠 */
```

★ **关键认知**：右列触发 BFC 后,**不会侵入左侧浮动的占据区**,实现"左定宽 + 右自适应"。⚠ 这是 float 时代的经典两栏技法,如今被 flex/grid 取代;但理解它,能帮你读懂大量历史代码。

# 4. 外边距折叠(margin collapsing)

## 4.1 折叠规则

```css
/* 兄弟元素:上下相邻块级 margin 重叠,取较大值(不相加) */
.first  { margin-bottom: 30px; }
.second { margin-top: 20px; }
/* 间距是 30px,不是 50px */
```

★ **关键认知**：**垂直方向的相邻块级 margin 会折叠(取较大)**;水平方向**不会**;flex/grid 容器里的子项也**不会**折叠。⚠ 一半"我明明留了间距怎么不够"的疑问,答案都是"旁边的那个 margin 更大,把小的合并掉了"。

## 4.2 折叠的边界

★ **关键认知**：外边距折叠发生在**同一 BFC 内**;两个元素若处于**不同 BFC**(或加了 padding/border 隔开),则**不发生折叠**。⚠ 这也是为什么"触发 BFC""加 border/padding"能阻止外边距折叠——它们破坏了折叠发生的条件。

# 5. 其他高频坑

## 5.1 包含块与百分比

★ **关键认知**：百分比尺寸的**基准是"包含块(containing block)"**——通常就是最近的块级祖先;但**定位元素的包含块是"最近的已定位祖先"**。⚠ 这就是"absolute 元素的 width:100% 为什么相对那个 relative 父"的底层原因;回想第 6 篇的定位锚点,两者是同一件事的两面。

## 5.2 行内块元素的基线空隙

```html
<!-- 两个 inline-block 之间会有 4px 左右的空隙(源码里的空白/换行所致) -->
<div class="a"></div>
<div class="b"></div>
```

```css
/* 解法:父容器 font-size: 0,或用 flex 替代 inline-block */
.parent { font-size: 0; }
```

★ **关键认知**：`inline-block` 元素之间源码里的**空白/换行**会产生**莫名空隙**(行内的空白字符所致)。⚠ 解法——父容器 `font-size: 0` 或直接改用 flex;这是研究"为什么两个 div 中间有条缝"的经典答案。

# 6. 本篇小结与下一步

★ **关键认知**：格式化上下文是 CSS 排版的"隔离作用域";BFC 能**包浮动、阻断外边距折叠、隔离浮动侵入**。外边距垂直折叠取较大、包含块决定百分比基准、inline-block 有空白缝。这些"坑"背后都是确定性的规则,不是玄学。

下一步第 16 篇:视觉效果——box-shadow、filter、clip-path、mask 与混合模式,让界面更有质感。