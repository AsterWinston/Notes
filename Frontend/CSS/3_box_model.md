所有元素,本质上都是一个盒子。盒模型(content/padding/border/margin 四层)决定元素占多大、内容之间留多少空;`box-sizing` 则决定"我写 width 到底量到哪"。本篇讲清盒模型的四层结构、两种 box-sizing、宽高的计算与百分比陷阱——这是理解一切布局的地基。

# 1. 盒模型四层

## 1.1 从内到外

```
┌────────────────────────────┐
│          margin(外边距)     │  与相邻元素的间距
│  ┌──────────────────────┐  │
│  │       border(边框)    │  │
│  │  ┌────────────────┐  │  │
│  │  │   padding(内边距) │  │  │
│  │  │  ┌──────────┐  │  │  │
│  │  │  │ content  │  │  │  │  内容
│  │  │  │ (内容区)  │  │  │  │
│  │  │  └──────────┘  │  │  │
│  │  └────────────────┘  │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

★ **关键认知**：盒子从内到外四层——**content(内容)→ padding(内边距,内容到边框)→ border(边框)→ margin(外边距,到相邻元素)**。⚠ margin 是"元素外"、padding 是"元素内",一个最直观的分辨:"**背景色只铺到 border 内,含 padding 但不含 margin**"。

# 2. box-sizing:width 量到哪

## 2.1 content-box vs border-box

```css
.box {
  width: 200px;
  padding: 20px;
  border: 10px solid black;

  /* content-box(默认):width 只算内容,实际占 200+40+20=260 宽 */
  box-sizing: content-box;

  /* border-box:width 含 padding 和 border,总宽就是 200 */
  box-sizing: border-box;
}
```

★ **关键认知**：`box-sizing` 决定 `width` 的**测量范围**——`content-box`(默认)只量内容区(实际总宽 = width + padding + border),`border-box` 量到边框(写 200 就是 200)。⚠ **现代工程几乎全项目改用 `box-sizing: border-box`**——它让"我写多宽就是多宽",不必再心算 padding/border 的额外占用,是新手到实战的第一个关键切换。

## 2.2 全局切换

```css
/* 经典全局 reset:所有元素改用 border-box */
*, *::before, *::after {
  box-sizing: border-box;
}
```

★ **关键认知**：用通配符 + 伪元素全局切到 border-box,是所有 CSS 工程的标准开场。⚠ 切换是"安全"的——它只改变 width/height 的语义,不改变布局结果,几乎不会有副作用,放心加。

# 3. margin

## 3.1 简写规则

```css
.box { margin: 10px; }            /* 四边都是 10 */
.box { margin: 10px 20px; }       /* 上下 10,左右 20 */
.box { margin: 10px 20px 30px; }  /* 上 10,左右 20,下 30 */
.box { margin: 1px 2px 3px 4px; } /* 上 右 下 左(顺时针) */
.box { margin: 0 auto; }          /* 上下 0,左右 auto → 水平居中 */
```

★ **关键认知**：margin 简写的四值顺序是**上、右、下、左(顺时针)**,一到四个值按规则扩展。⚠ `margin: 0 auto` 是块级元素**水平居中**的经典写法(需元素有确定宽度)。margin 可为**负值**(把元素往外/重叠方向拉)。

## 3.2 margin 合并(collapse)

```css
/* 上下相邻的块级元素,margin 会"合并"取较大值,而非相加 */
.top    { margin-bottom: 20px; }
.bottom { margin-top: 30px; }
/* 两者的垂直间距是 30px,不是 50px */
```

★ **关键认知**：**垂直相邻的块级元素 margin 会合并(塌陷)**——取较大者而非相加;水平方向不合并。⚠ 这是最反直觉的点:"我上下各留 20px,怎么还是 20 间距?" 解决方向见第 15 篇 BFC(如父元素建 BFC、改用 padding 等)。父子之间的第一个/最后一个子元素的 margin 也会"溢出合并"到父元素。

# 4. padding 与 border

## 4.1 padding 的特性

```css
.box { padding: 20px 30px; }   /* 内边距:上下 20,左右 30 */
```

★ **关键认知**：padding 是**内容到边框的内间距**,不可为负值,会**撑大元素**(在 content-box 下)。⚠ padding 的**百分比是相对父元素宽度**(哪怕写的是上下 padding),这是 CSS 一个著名的"反直觉"——百分比 padding/margin 大多相对容器的**宽度**。

## 4.2 border 简写

```css
.border {
  border: 1px solid #ccc;       /* 粗细 样式 颜色 的简写 */
  border-radius: 8px;           /* 圆角 */
  border-bottom: 2px dashed red;/* 单边覆盖 */
}
```

★ **关键认知**：border 简写顺序是**宽、样式、颜色**;`border-radius` 控制圆角。⚠ border 的 `style` 是**必填**的——默认 `border-style: none`,光写颜色粗细不显示边框;`border-radius: 50%` 可做成圆。

# 5. 尺寸与百分比

## 5.1 width / height 的百分比基准

```css
.child { width: 50%; }   /* 相对父元素 content 宽度的 50% */
```

★ **关键认知**：`width` 百分比**相对父元素内容区宽度**;`height` 百分比则更"挑剔"——**需要父元素有明确的高度**才有效(否则常失效)。⚠ "height: 100% 不生效"是高频坑:因为父容器高度可能由内容撑开、未定,此时百分比高度无从计算,可改用 flex/grid 或显式给父级高度。

## 5.2 内容撑开与溢出

```css
.box {
  width: 100px;
  overflow: hidden;   /* 内容超出怎么办:隐藏/滚动/自动 */
}
```

★ **关键认知**：内容超出固定尺寸时,用 `overflow` 处理——`hidden`(裁掉)、`scroll`(永远滚动条)、`auto`(需要时滚动)。⚠ 不处理的溢出会视觉上"漏到外面",但**布局上仍占空间**,常常是版式错乱的元凶。

# 6. 本篇小结与下一步

★ **关键认知**：盒模型 content→padding→border→margin 四层;`box-sizing: border-box` 让 width"落地即实";margin 垂直会合并、百分比相对父宽;height 百分比要父级有确定高度。掌握这些,布局的"尺寸感"就立住了。

下一步第 4 篇:文本与字体——font 体系、text 排版、line-height,以及 web 字体 @font-face。