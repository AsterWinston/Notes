Flex 管一行/一列,Grid 管**二维**——行和列同时排布,是复杂页面(后台管理、画廊、仪表盘)的主力。本篇讲清 Grid 的网格线/轨道模型、定义行列的三个函数、子项定位,以及 `grid-template-areas` 这个"画布局草图"的神器。

# 1. 网格模型

## 1.1 容器与轨道

```css
.container {
  display: grid;                /* 变成网格容器 */
  grid-template-columns: 200px 1fr 1fr;  /* 三列:定宽 + 两等分 */
  grid-template-rows: 80px auto;         /* 两行:固定 + 自动 */
  gap: 16px;                    /* 行列间距 */
}
```

★ **关键认知**：`grid-template-columns/rows` 定义**轨道(track)**——几个值就是几列/几行;`1fr`(fraction)是**按比例瓜分剩余空间**的单位。⚠ `1fr` 是 Grid 最核心单位:**"占一份剩余空间"**,`200px 1fr 1fr` 就是"左边定宽 200,剩下两列等分"。

## 1.2 网格线

```
      1         2         3         4
      |         |         |         |
    ──┼─────────┼─────────┼─────────┼── 1
      |         |         |         |
    ──┼─────────┼─────────┼─────────┼── 2
      |         |         |         |
      ↑列线编号  ↑         ↑         ↑
```

★ **关键认知**：Grid 里,行列之间由**编号的网格线(grid line)**划分——从左/上开始 1、2、3…;**子项靠"从第几条线到第几条线"来定位与跨行跨列**。⚠ 理解"网格线"是理解 Grid 定位的关键,后面 `grid-column`/`grid-row` 用的都是**线的编号**。

# 2. 定义轨道

## 2.1 repeat 与函数

```css
.grid {
  grid-template-columns: repeat(3, 1fr);        /* 3 等分 */
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  /* 自动填充:每列至少 200px,尽可能多列,响应式卡片列表 */
  grid-template-columns: 100px repeat(2, 1fr) 80px;
}
```

★ **关键认知**：`repeat(n, 尺寸)` 重复 n 次;`minmax(最小, 最大)` 设轨道尺寸范围;`auto-fill` 按容器宽度**自动决定列数**。⚠ **`repeat(auto-fill, minmax(200px, 1fr))` 是"响应式卡片网格"的标准配方**——一行能放几列放几列,每列至少 200px,无需媒体查询。

## 2.2 fr 与 auto

```css
.grid {
  grid-template-columns: 1fr 2fr auto;
  /* 1fr:一份,2fr:两份,auto:按内容自适应 */
}
```

★ **关键认知**：轨道单位——`fr`(按比例分剩余)、`px`(固定)、`auto`(按内容自适应)、`minmax()`(范围)。⚠ `fr` 分的是**去掉固定尺寸后的"剩余空间"**,所以 `100px 1fr 1fr` 里两个 1fr 是等分的"剩余",而不是三分之一。

# 3. 子项定位

## 3.1 grid-column / grid-row

```css
.item {
  grid-column: 1 / 3;     /* 横跨第 1 到第 3 条列线(占两列) */
  grid-row: 1 / 2;        /* 占第 1 到第 2 条行线(第一行) */
  grid-column: span 2;    /* 或 span:跨越 2 列 */
}
```

★ **关键认知**：`grid-column` / `grid-row` 用**网格线编号**给子项定位——`1 / 3` 表示"从线 1 到线 3"(跨两列);`span 2` 表示"跨 2 列(从当前位置算)"。⚠ 头部 banner 跨整行就是 `grid-column: 1 / -1`(`-1` 指最后一条线),这是高频写法。

## 3.2 grid-area

```css
/* grid-area 简写:行开始/列开始/行结束/列结束 */
.item { grid-area: 1 / 1 / 2 / 3; }  /* 第一行,跨两列 */
```

★ **关键认知**：`grid-area` 一次写四值(**行始/列始/行末/列末**),是 `grid-row`+`grid-column` 的简写。⚠ 它还有个更爽的用法——配合 `grid-template-areas` 用**名字而非数字**定位(见下)。

# 4. grid-template-areas

## 4.1 用"名字"画布局

```css
.layout {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-areas:
    "header header"     /* 第一行:header 跨两列 */
    "sidebar main"      /* 第二行:侧栏 + 主区 */
    "footer footer";    /* 第三行:footer 跨两列 */
}
.header { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main { grid-area: main; }
.footer { grid-area: footer; }
```

★ **关键认知**：`grid-template-areas` 用**字符串画布局草图**——每个字符串是**一行**,里面的名字对应**一列**,同名占多列即"跨列";子项用 `grid-area: 名字` 对号入座。⚠ 这是 Grid 最"声明式"的用法:**一眼看出页面结构**,复杂布局可读性极佳,等于把 ASCII 图直接写成了 CSS。

# 5. 对齐与实战

## 5.1 对齐

```css
.grid {
  justify-content: center;   /* 网格整体在容器里的水平分布 */
  align-items: center;       /* 子项在各自网格单元里的垂直对齐 */
  /* 还有 place-items: center 一次设双向 */
}
```

★ **关键认知**：Grid 的对齐沿用 flex 思路——`justify-*` 水平、`align-*` 垂直;`place-items: center` 一次居中双向。⚠ 区别在作用对象:`justify-items/align-items` 管**子项在单元格内**对齐,`justify-content/align-content` 管**整个网格在容器内**对齐。

## 5.2 flex vs grid 决策

| 场景 | 用 | 理由 |
|------|-----|------|
| 一行导航/按钮组 | flex | 一维 |
| 居中(水平+垂直) | flex | 简单 |
| 卡片网格(行列) | grid | 二维 |
| 整页两栏/三栏 | grid + areas | 结构清晰 |
| 响应式卡片(自动列数) | grid + auto-fill | 免媒体查询 |

★ **关键认知**：选择标准是"**要布的是几维**"——一维(单行/单列)flex 更简,二维(行列)grid 更强。⚠ 二者**不互斥**,可以组合:页面的"宏观分区"用 grid,分区内部"一维排列"再用 flex,这是现代布局的主流分工。

# 6. 本篇小结与下一步

★ **关键认知**：Grid 用 `fr` 分比例、用网格线/areas 定位子项;`repeat(auto-fill, minmax())` 做响应式卡片;`grid-template-areas` 用字符串画布局。二维布局交给它,它与 flex 分层协作。

下一步第 10 篇:响应式与媒体查询——@media、clamp/min/max,让同一套代码适配所有屏幕。