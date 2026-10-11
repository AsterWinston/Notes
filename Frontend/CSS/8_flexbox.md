Flexbox 是现代**一维布局**(一行或一列)的事实标准——它让"水平居中、等分、两端对齐、自动伸缩"这些过去要 hack 的需求变得一行搞定。本篇讲清 Flex 的双轴模型、六个容器属性、子项的 flex 缩写,以及最常用的几个布局套路。

# 1. 双轴模型

## 1.1 主轴与交叉轴

```
flex-direction: row(默认)        主轴 →
  ┌──────────────────────────────┐
  │ [item][item][item]           │
  └──────────────────────────────┘
                      交叉轴 ↓
  主轴决定子项排布方向,交叉轴垂直主轴
```

★ **关键认知**：Flex 的核心是**双轴**——**主轴(main axis,子项排列方向,默认水平)**与**交叉轴(cross axis,垂直主轴)**。⚠ 理解 Flex 的关键:**先问"主轴朝哪"**,主轴的改变(`flex-direction: column`)会让"左右对齐"与"上下对齐"互换。默认 row 时,主轴水平、交叉轴垂直。

## 1.2 容器与子项

```css
.container { display: flex; }   /* 变成 flex 容器 */
/* 直接子元素自动成为 flex 子项,按主轴排列 */
```

★ **关键认知**：设 `display: flex` 的元素成为**容器(container)**,其**直接子元素**成为**子项(item)**——子项默认沿主轴方向排列、不换行。⚠ Flex **只作用一维**(主轴方向),二维(行列同时)是 Grid 的活(第 9 篇)。子项默认会收缩(shrink)以适应一行。

# 2. 容器的六个属性

## 2.1 方向与换行

```css
.container {
  flex-direction: row;        /* row(默认)/row-reverse/column/column-reverse */
  flex-wrap: wrap;            /* nowrap(默认)/wrap 换行/wrap-reverse */
  flex-flow: row wrap;        /* direction + wrap 的简写 */
}
```

★ **关键认知**：`flex-direction` 定主轴方向,`flex-wrap` 定超出一行时**是否换行**(默认 nowrap,子项会被压扁也不换行)。⚠ 卡片列表的经典设置是 `flex-wrap: wrap`——不换行则子项被强行压缩。

## 2.2 主/交叉轴对齐

```css
.container {
  justify-content: space-between; /* 主轴:flex-start/center/space-between/space-around */
  align-items: center;            /* 交叉轴:stretch/center/flex-start/flex-end */
  align-content: center;          /* 多行时,行与行在交叉轴的分布 */
}
```

★ **关键认知**：`justify-content` 管**主轴**对齐(常用 `space-between` 两端分散、`center` 居中),`align-items` 管**交叉轴**对齐(单行内的垂直对齐)。⚠ 一行记忆:**justify 沿主轴、align 沿交叉轴**;"水平居中"是 `justify-content: center`(row 时),"垂直居中"是 `align-items: center`。

## 2.3 gap 间距

```css
.container {
  display: flex;
  gap: 16px;        /* 子项之间的间距(行/列都可),替代 margin 硬撑 */
}
```

★ **关键认知**：`gap` 直接设子项**间距**,是"子项之间留 16px"的正解——比给每个子项加 margin 再处理首尾去边优雅得多。⚠ gap 在 flex/grid 都支持,是较新但已广泛兼容的属性。

# 3. 子项的 flex 属性

## 3.1 flex 缩写

```css
.item {
  /* flex: <grow 放大> <shrink 缩小> <basis 基准> */
  flex: 1;          /* flex: 1 1 0% —— 均分剩余空间 */
  flex: 1 1 auto;   /* 有内容基准地放大 */
  flex: 0 0 200px;  /* 固定 200px,不放大不缩小 */
}
```

★ **关键认知**：`flex` 三件套——**grow(有多余空间放大)、shrink(空间不足缩小)、basis(基准尺寸)**。⚠ `flex: 1` 是最常用的"**均分**"写法(等价 `1 1 0%`);`flex: 0 0 200px` 是"**定宽不伸缩**"。理解 grow/shrink 就理解了"flex 弹性"的本质:grow 按比例**瓜分多余空间**,shrink 按比例**分担不足空间**。

## 3.2 经典均分与侧栏布局

```css
/* 经典"侧边栏 + 主内容":侧边定宽,主内容占满剩余 */
.sidebar { flex: 0 0 200px; }   /* 固定 200 */
.main    { flex: 1; }           /* 吃掉剩余全部 */
```

★ **关键认知**："**固定侧栏 + 自适应主体**"是 flex 的招牌套路——固定项 `flex: 0 0 宽`,自适应项 `flex: 1`。⚠ 这比 float + 负 margin 的老式圣杯布局简单太多,正是 flex 取代 float 做布局的原因。

# 4. 常用套路

## 4.1 完美居中

```css
.center {
  display: flex;
  justify-content: center;  /* 主轴(水平)居中 */
  align-items: center;      /* 交叉轴(垂直)居中 */
}
```

★ **关键认知**：**flex 居中**是当下最通用的"水平+垂直居中"——容器上 `justify-content:center` + `align-items:center` 两行搞定,单行多行都行。⚠ 对比第 6 篇的 absolute 居中:flex 居中**不脱离文档流、更简单**,绝大多数场景用它。

## 4.2 导航条与底部对齐

```css
.nav { display: flex; align-items: center; gap: 24px; }

.card {
  display: flex;
  flex-direction: column;      /* 竖排 */
  justify-content: space-between; /* 头尾分居两端 */
}
```

★ **关键认知**：flex 的常见形态——**导航条**(横向 + 居中 + gap)、**卡片内竖排**(`flex-direction: column` + 首尾分布)。⚠ 一个易忽略点:`flex-direction: column` 后,`justify-content` 就管**垂直**方向了(主轴变竖),别再用老的"水平"印象套。

# 5. 本篇小结与下一步

★ **关键认知**：Flex = 双轴 + 六容器属性 + 子项 flex 缩写。口诀:flex-direction 定主轴、justify 沿主轴、align 沿交叉轴、gap 设间距、`flex: 1` 均分。一维布局(导航、卡片、居中)交给它。

下一步第 9 篇:Grid——二维网格布局,网格线与区域,真正现代复杂布局的主力。