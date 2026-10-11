同一套页面,要在手机、平板、桌面都好看,靠的是**响应式(responsive)**。本篇讲清响应式的三大支柱——viewport 与流式布局、`@media` 媒体查询、以及 `min/max/clamp` 这类"自适应函数",并给出"移动优先"的实战方法。

# 1. 响应式的三大支柱

## 1.1 从固定到流动

```css
/* 支柱一:流式(fluid)——用百分比/fr 让尺寸随容器流动 */
.container { width: 100%; max-width: 1200px; }
.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); }
```

★ **关键认知**：响应式的底座是**流式布局**——用 `%`、`fr`、`minmax` 让元素**随容器尺寸流动**,而非写死像素。⚠ 一个关键原则:**先让布局"流动起来",再用媒体查询处理"流动不了的断点"**;别把所有适配都押在媒体查询上。

## 1.2 支柱二:媒体查询 @media

```css
/* 屏幕宽度 ≤ 768px 时应用(手机/平板) */
@media (max-width: 768px) {
  .grid { grid-template-columns: 1fr; }  /* 从多列变单列 */
}
```

★ **关键认知**：`@media` 是**条件样式**——满足条件(如宽度、横向/纵向、打印)才应用。⚠ `max-width` 与 `min-width` 的选择:`max-width` 是"**这条以下**"(适合移动优先),`min-width` 是"**这条以上**";方向别写反。

## 1.3 支柱三:自适应函数

```css
.title { font-size: clamp(1rem, 4vw, 2rem); }
/* clamp(最小, 理想, 最大):字号随视口缩放到 4vw,但不小于 1rem 不大于 2rem */
.box { width: min(100%, 600px); }   /* 取两者较小 */
.box { width: max(300px, 50%); }    /* 取两者较大 */
```

★ **关键认知**：`min()` 取最小、`max()` 取最大、`clamp(min, 理想, max)` 夹在区间内——它们让**单个属性本身就有响应能力**,少写媒体查询。⚠ `clamp(1rem, 4vw, 2rem)` 是"**流式字号**"的标准写法:字号随屏幕缩放但有限制,是减少断点的利器。

# 2. viewport 与断点

## 2.1 断点怎么定

```css
/* 常见断点(非强制,按内容需求调整) */
@media (max-width: 576px) { }   /* 手机 */
@media (max-width: 768px) { }   /* 平板竖 */
@media (max-width: 992px) { }   /* 平板横/小屏本 */
@media (max-width: 1200px) { }  /* 桌面 */
```

★ **关键认知**：断点应**按"内容开始变丑"来定**,而非死记设备尺寸——内容挤了、换行了、截断了,才需要设置断点。⚠ 移动端 viewport 是前提(HTML 第 3 篇的 `<meta name="viewport">`),没有它,媒体查询的宽度判断是错的。

# 3. 移动优先(mobile-first)

## 3.1 从小到大写

```css
/* 移动优先:基础样式是"最小屏",用 min-width 逐步"往上加" */
.card { font-size: 14px; }              /* 默认(手机) */
@media (min-width: 768px) {
  .card { font-size: 16px; }            /* 平板及以上放大 */
}
@media (min-width: 1200px) {
  .card { font-size: 18px; }            /* 桌面再放大 */
}
```

★ **关键认知**：**移动优先** = 基础样式面向**手机**,用 `min-width` **逐级增强**到平板/桌面。⚠ 它与"桌面优先(默认桌面 + max-width 降级)"相反;移动优先更符合"先保证最小屏可用、再锦上添花",是主流推荐。

## 3.2 为什么移动优先

★ **关键认知**：移动优先的收益——**CSS 更精简**(基础样式简单,大屏才叠加)、**移动端天生最卡所以值得优先优化**、**与移动流量占比趋势一致**。⚠ 成本是要改变"先在桌面调好再往小屏压"的惯性,一开始就按手机宽度写。

# 4. 响应式实战套路

## 4.1 响应式导航(汉堡菜单思路)

```css
.nav-links { display: none; }          /* 手机默认隐藏 */
@media (min-width: 768px) {
  .nav-links { display: flex; }        /* 大屏展开 */
}
/* 手机端用 JS 切出抽屉菜单 */
```

★ **关键认知**：响应式导航是"移动优先"的经典案例——手机端菜单**收起**(汉堡按钮 + 抽屉)、桌面端**展开**(水平导航)。⚠ 这涉及 JS 交互与断点配合,但 CSS 层面就是"默认隐藏,大屏 flex 展开"。

## 4.2 图片与字体的响应式

```css
img { max-width: 100%; height: auto; }  /* 图片不超容器,自动等比缩放 */
html { font-size: 16px; }
.title { font-size: clamp(1.25rem, 2vw, 2rem); }
```

★ **关键认知**：响应式图片的 CSS 底线是 **`img { max-width: 100%; height: auto }`**(让图不溢出不拉伸);更精细的换图用 HTML 第 5 篇的 `picture/srcset`。字号用 clamp 流式缩放。

# 5. 本篇小结与下一步

★ **关键认知**：响应式三支柱——**流式布局 + 媒体查询 + min/max/clamp**。方法:先让布局流动、按内容定断点、移动优先(min-width 增强)。别把一切都压给媒体查询,能用 clamp/auto-fill 解决的就别加断点。

下一步第 11 篇:布局实战与取舍——把 flex/grid/定位串起来,解决常见布局,并给出三大布局手段的选型。