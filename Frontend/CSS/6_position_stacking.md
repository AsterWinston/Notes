`z-index` 不生效、"定位元素"到底是啥、为什么明明设了 999 还被压在下面——这些玄学,根源都在 **position 与层叠上下文(stacking context)**。本篇讲清 position 五种值、z-index 的生效前提,以及"层叠上下文"这个决定了谁在谁上面的隐藏坐标系。

# 1. position 五种值

## 1.1 static(默认,文档流)

```css
.box { position: static; }   /* 默认:按文档流排列,top/left/z-index 无效 */
```

★ **关键认知**：`static` 是默认值,元素**正常进文档流**排布,此时 `top/right/z-index` **无效**。⚠ 大部分元素就该是 static,只有需要"脱离/偏移/层叠"时才改 position。

## 1.2 relative(相对定位)

```css
.box {
  position: relative;   /* 相对自己原本位置偏移 */
  top: 10px;            /* 向下移(原来是"从顶部"偏移) */
  left: 20px;
}
```

★ **关键认知**：`relative` **保留原占位**(文档流里位置还在),只是视觉上相对自己偏移。⚠ 它最常见的用途不是移动,而是**给子元素的 absolute 当"定位参照物"**(见 1.3)——父相对、子绝对,是定位的黄金搭配。

## 1.3 absolute(绝对定位)

```css
.parent { position: relative; }   /* 父:相对定位(锚点) */
.child {
  position: absolute;   /* 相对"最近的已定位祖先"定位 */
  top: 0;
  right: 0;             /* 贴父元素右上角 */
}
```

★ **关键认知**：`absolute` 让元素**脱离文档流**(不再占位),相对**最近的已定位祖先**(position 非 static)**进行定位**;找不到就相对 **body/初始包含块**。⚠ "绝对于哪个父"是新手最易错点——不是最近的父,而是**最近的"已定位(非 static)"祖先**,所以常给父元素加 `position: relative` 做锚点。

## 1.4 fixed(固定定位)

```css
.nav {
  position: fixed;   /* 相对视口固定,滚动也不动 */
  top: 0; left: 0; right: 0;
}
```

★ **关键认知**：`fixed` 相对**视口(viewport)**定位,页面滚动时它**纹丝不动**(常用于吸顶导航、悬浮按钮)。⚠ 特例:若祖先有 `transform`/`filter` 等,**fixed 会变成相对那个祖先定位**(不再是视口),这是"吸顶失效"的高频原因。

## 1.5 sticky(粘性定位)

```css
.table-head {
  position: sticky;   /* 滚动到阈值前像 relative,之后像 fixed */
  top: 0;             /* 距视口顶 0 时"粘住" */
}
```

★ **关键认知**：`sticky` 是 relative 与 fixed 的**混合体**——在滚动到阈值(`top: 0`)前正常流动,超过后被"粘"在视口(常见于**吸顶表头、侧栏**)。⚠ sticky 的父容器必须"够高"容它滚,**父容器滚完它也随之滚走**;它**不脱离文档流**,占位仍在。

# 2. z-index 与层叠上下文

## 2.1 z-index 的生效前提

```css
.box {
  position: relative;  /* 必须先"已定位" */
  z-index: 10;         /* 才生效 */
}
/* static 元素的 z-index 会被忽略 */
```

★ **关键认知**：`z-index` **只对已定位元素(position 非 static)生效**——这让 `static` 元素设 z-index 毫无效果。⚠ "为什么我的 z-index 不生效"第一答案:元素没被定位;第二答案:不在同一层叠上下文里(见下)。

## 2.2 层叠上下文是什么

★ **关键认知**：**层叠上下文(stacking context)** 是一个"独立的 z-index 坐标系"——在同一上下文内,才按 z-index 比较上下;不同上下文的元素之间,**比的是上下文本身的层级,而非内部子元素的 z-index**。⚠ 这解释了经典困惑:"我给了子元素 z-index: 999,还是被 z-index: 1 的另一块盖住"——因为两个子元素分属**不同的层叠上下文**,999 只在它自己的上下文里"数大",和外面那个 1 不在一个尺子上比。

## 2.3 谁在创建层叠上下文

```css
/* 能创建层叠上下文的常见触发: */
.a { position: absolute; z-index: 1; }  /* 已定位 + z-index */
.b { opacity: 0.9; }                    /* opacity < 1 */
.c { transform: scale(1); }             /* transform/filter/perspective */
.d { will-change: transform; }
```

★ **关键认知**：创建层叠上下文的常见条件——**定位 + z-index、opacity<1、transform、filter、will-change** 等。⚠ 正因为 `opacity`/`transform` 也会偷偷创建层叠上下文,很多"z-index 莫名失效"的锅是它们背的——排查时看父级是否踩了这些触发条件。

# 3. 同一上下文内的层叠顺序

## 3.1 从上到下的默认顺序

```
1. 正 z-index 的定位元素
2. z-index: 0 / auto 的定位元素
3. 负 z-index 的定位元素
4. 块级(block)元素
5. 浮动(float)元素
6. 内联(inline)元素
7. 背景与边框(最底层)
```

★ **关键认知**：同一层叠上下文内,无 z-index 时也有一套默认顺序——**内联在上、块级在下、背景最底**。⚠ 这就是为什么未定位的文字会"浮"在已定位块的背景之上:它们的默认层叠层级不同。

# 4. 定位实战

## 4.1 子元素居中于父

```css
.parent { position: relative; }
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);  /* 用 transform 抵消自身一半,精确居中 */
}
```

★ **关键认知**：absolute + `top/left: 50%` + `translate(-50%,-50%)` 是**绝对居中**的经典写法(translate 百分比相对**自身尺寸**,正好抵消一半)。⚠ 这是"定位脱离文档流"的典型应用;现代也可用 flex/grid 居中(第 8、9 篇),绝对居中更适用于"脱离文档流"的悬浮层(如 loading、弹窗 mask)。

## 4.2 满屏遮罩

```css
.mask {
  position: fixed;      /* 相对视口 */
  inset: 0;             /* 等于 top/right/bottom/left 全 0 */
  background: rgba(0,0,0,0.5);
  z-index: 100;
}
```

★ **关键认知**：`position: fixed + inset: 0` 铺满视口,是**弹窗遮罩/mask** 的标准写法。⚠ 弹窗的层级管理(遮罩与内容谁在上)靠 z-index,而它们的 z-index 若要跨组件稳定,常需要统一规划(如遮罩固定 100、弹窗 101)。

# 5. 本篇小结与下一步

★ **关键认知**：position 五值——static 文档流、relative 保留占位做锚点、absolute 脱离流相对已定位祖先、fixed 相对视口、sticky 滚动粘滞。z-index 只对已定位元素生效,且受**层叠上下文**约束——不同上下文的 z-index 不互通,这是所有"层级玄学"的真相。

下一步进入**阶段二 · 布局**,第 7 篇:display 与文档流——display 取值、float 与清除浮动。