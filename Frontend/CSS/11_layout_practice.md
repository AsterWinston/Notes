前几篇把工具都备齐了,这篇把它们**串成实战**——一个页面该先用哪套布局、常见布局模式怎么搭、居中问题一次讲全,最后给一张"flex / grid / 定位 / 文档流"的选型表。瞄准的是"拿到设计稿不再无从下手"。

# 1. 布局心法:从上到下分块

## 1.1 先分大块,再排小块

```
      ┌───────────────────────────┐
      │          header(顶栏)      │   ← 整页 Grid:三行
      ├───────────┬───────────────┤
      │  sidebar  │     main      │   ← 中间:两列(侧栏+主区)
      ├───────────┴───────────────┤
      │          footer(底栏)      │
      └───────────────────────────┘
  主区内部的一排卡片 ← flex / grid 子布局
```

★ **关键认知**：**宏观网格、微观弹性**——整页的**大结构用 grid**(行列分区),每个区块**内部的一维排列用 flex**。⚠ 新手常见病是"一个 flex 硬排整页";记住:**先 grid 画骨架,再 flex 排细节**,两层各司其职。

# 2. 常见布局模式

## 2.1 圣杯/双飞翼(头部 + 侧栏 + 主体 + 底部)

```css
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 240px 1fr;
  grid-template-rows: auto 1fr auto;
}
```

★ **关键认知**：经典"**上中下 + 左中右**"布局,用 grid + `grid-template-areas` 三行代码画清楚。⚠ 这一套已经取代了当年 float 时代的圣杯布局(负 margin 的噩梦),是后台管理页的标准骨架。

## 2.2 卡片流(自适应的网格)

```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 24px;
}
```

★ **关键认知**：卡片列表用 `auto-fill + minmax`,**一行能放几列放几列**,自动响应断点,无需媒体查询。⚠ 每个卡片是高不定时,可用 `align-items: start` 避免被拉等高等。

## 2.3 两栏(侧栏定宽 + 主体自适应)

```css
.sidebar { flex: 0 0 220px; }
.main { flex: 1; min-width: 0; }   /* min-width:0 防内容撑破 */
```

★ **关键认知**：两栏的 flex 套路大家熟,⚠ 但有个隐蔽坑:**主区要加 `min-width: 0`**,否则主区内长文本/代码块会把 flex 子项撑破(flex 子项默认 `min-width: auto` 不收缩)。这是"flex 布局被长内容撑爆"的经典解法。

# 3. 居中问题全集

## 3.1 四种居中

```css
/* 1. 文字水平居中 */
.text-center { text-align: center; }

/* 2. 块级水平居中(定宽) */
.block-center { margin: 0 auto; }

/* 3. 水平+垂直居中(未知尺寸,首选) */
.flex-center { display: flex; justify-content: center; align-items: center; }

/* 4. 脱离文档流的居中(悬浮层) */
.abs-center {
  position: absolute;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
}
```

★ **关键认知**：中路四式——文字用 `text-align`,块用 `margin auto`,常规元素用 **flex 居中**,脱离流的悬浮层用 **absolute + translate**。⚠ 拿不准就用 flex 居中,它覆盖绝大多数"把某个东西放中间"的场景。

# 4. 选型决策表

| 需求 | 首选 | 次选 |
|------|------|------|
| 一维排列(导航/按钮组/单行) | flex | — |
| 二维网格(卡片/表格/画廊) | grid | flex wrap |
| 整页多区块骨架 | grid + areas | flex column |
| 文字环绕图片 | float | — |
| 绝对定位(悬浮/角标/遮罩) | position | — |
| 吸顶/吸底 | sticky / fixed | — |
| 简单水平居中 | margin auto / text-align | flex |

★ **关键认知**：布局没有唯一答案,但**有主次**——**一维 flex、二维 grid、脱离流 position、环绕 float**。⚠ 反模式聚焦:**别用 float 做整页布局**(用 grid/flex)、**别用 table 做布局**(它语义是数据)、**别用 div 硬堆靠 margin/position 摆位**。

# 5. 实战:搭一个标准后台骨架

```css
.admin {
  display: grid;
  height: 100vh;                       /* 满屏 */
  grid-template-columns: 220px 1fr;    /* 侧栏 + 主区 */
  grid-template-rows: auto 1fr;        /* 顶栏 + 正文 */
  grid-template-areas:
    "sidebar header"
    "sidebar main";
}
.sidebar { grid-area: sidebar; }
.header  { grid-area: header; display: flex; align-items: center; }
.main    { grid-area: main; overflow: auto; }
```

★ **关键认知**：`100vh + grid-template-areas` 能搭出**满屏三区**的后台骨架——侧栏固定、顶栏 + 内容区,内容区独立滚动(`overflow:auto`)。⚠ 这是真实项目复现率最高的布局之一,理解它,80% 的"后台布局"情境都能套。

# 6. 本篇小结与下一步

★ **关键认知**：布局心法"**宏观 grid、微观 flex**";常见模式有圣杯(grid areas)、卡片流(auto-fill)、两栏(flex);居中首选 flex;选型"一维 flex、二维 grid、脱离流 position"。拿到设计稿,先分块后细化。

下一步进入**阶段三 · 进阶**,第 12 篇:过渡与动画——transition/animation/keyframes,让界面"动"得优雅。