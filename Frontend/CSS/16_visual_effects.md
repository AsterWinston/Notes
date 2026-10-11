布局懂了之后,界面"有没有质感"就看视觉效果的细节了——投影、模糊、裁剪形状、遮罩、混合模式。本篇讲清 `box-shadow`、`filter`、`clip-path`、`mask` 与混合模式这组"特效"属性,以及它们的性能边界。

# 1. box-shadow 与 text-shadow

## 1.1 box-shadow

```css
.card {
  /* 水平位移 垂直位移 模糊 扩散 颜色 */
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  /* 多层阴影:逗号分隔 */
  box-shadow: 0 1px 2px rgba(0,0,0,0.1), 0 4px 8px rgba(0,0,0,0.1);
}
```

★ **关键认知**：`box-shadow` 五值——**水平偏移、垂直偏移、模糊半径、扩散半径、颜色**;逗号分隔可叠多层。⚠ **卡片立体感**的经典配方是"多层小阴影"(`0 1px` + `0 4px`),比单层大阴影更自然;投射的阴影**不占布局空间**。

## 1.2 内阴影与文字阴影

```css
.box { box-shadow: inset 0 0 5px #000; }   /* inset:内阴影 */
.text { text-shadow: 0 2px 4px rgba(0,0,0,0.5); }  /* 文字阴影 */
```

★ **关键认知**：`inset` 把投影转到**内部**(凹槽感),`text-shadow` 给文字加阴影(常做凸起字或夜景文字)。⚠ 文字的阴影借用背景图的明暗,能显著提升"文字压在图上"的可读性。

# 2. filter 滤镜

## 2.1 常用滤镜函数

```css
img {
  filter: blur(4px);          /* 模糊 */
  filter: grayscale(100%);    /* 变灰 */
  filter: brightness(0.8);    /* 亮度 */
  filter: contrast(120%);     /* 对比度 */
  filter: opacity(0.5);       /* 透明度 */
  filter: drop-shadow(0 4px 8px rgba(0,0,0,0.3));  /* 投影(随形状) */
}
```

★ **关键认知**：`filter` 是一组**视觉效果滤镜**——模糊、灰度、亮度、对比度、投影等,可叠加。⚠ `drop-shadow` 与 `box-shadow` 的区别:`drop-shadow` **沿元素/图片的实际形状投影**(含透明轮廓),而 box-shadow 只投矩形边框——给透明背景的 logo 投影要用 drop-shadow。

## 2.2 filter 与性能

★ **关键认知**：`filter` 昂贵(尤其 blur),还会**落地成新的层叠上下文**(影响 z-index,见第 6 篇)。⚠ 大范围/常动画的 filter 要警惕性能;模糊这种重操作,优先"一次性处理"(进图而非运行时 blur)。

# 3. clip-path 裁剪

## 3.1 基本形状

```css
.avatar {
  clip-path: circle(50%);      /* 裁剪成圆形 */
  clip-path: polygon(0 0, 100% 0, 100% 80%, 0 100%);  /* 多边形 */
  clip-path: inset(10px);      /* 从四边各裁 10px */
}
```

★ **关键认知**：`clip-path` 用**形状裁剪元素**——只显示形状内的部分,是"圆形头像""异形卡片"的实现。⚠ 它**裁剪视觉但不改变布局**(被裁掉的部分仍占位);`clip-path: circle(50%)` 做圆形头像时,元素本身得是正方形。

## 3.2 polygon 与特效

```css
/* 切角、斜边、飘带等形状 */
.ribbon { clip-path: polygon(0 0, 100% 0, 100% 70%, 50% 100%, 0 70%); }
```

★ **关键认知**：`polygon` 用**顶点坐标**定义任意多边形,能做切角、斜切、飘带等形状。⚠ 顶点是相对元素的**百分比坐标**,需要一点几何直觉;调试时配合可视化工具会省力很多。

# 4. mask 遮罩

## 4.1 遮罩与渐变遮罩

```css
.masked {
  /* 用渐变当遮罩:内容从实到虚(渐隐) */
  -webkit-mask: linear-gradient(to right, black, transparent);
  mask: linear-gradient(to right, black, transparent);
}
```

★ **关键认知**：`mask` 用一张图/渐变作**遮罩**——遮罩的**不透明度**决定内容可见度(黑=显示、透明=隐藏)。⚠ "图片/文字**渐隐到透明**"的效果,就是 `mask + linear-gradient`(黑到透明)实现的;浏览器兼容需加 `-webkit-` 前缀。

## 4.2 与 clip-path 的区别

| 维度 | clip-path | mask |
|------|-----------|------|
| 本质 | 硬裁剪(要么显示要么不显示) | 软遮罩(可半透明过渡) |
| 渐变 | 无(形状边界是硬的) | 可(渐隐渐显) |
| 典型 | 圆形头像、切角 | 渐隐、纹理遮罩 |

★ **关键认知**：`clip-path` 是**硬裁剪**(边界利落),`mask` 是**软遮罩**(可渐变过渡)。⚠ 需要"渐隐/渐显"用 mask,需要"切成某形状"用 clip-path。

# 5. 混合模式 blend 与 mix-blend-mode

## 5.1 混合模式

```css
.blend {
  mix-blend-mode: multiply;   /* 与下层内容混合(正片叠底) */
  background-blend-mode: overlay;  /* 背景层之间的混合 */
}
```

★ **关键认知**：`mix-blend-mode` 让元素与**下层内容**按算法混合(multiply 正片叠底、screen 滤色、overlay 叠加…);`background-blend-mode` 是**背景层之间**混合。⚠ 混合模式在"图片调色、文字压图融合"上能出特色效果,但**结果难预测**,需要实际调试,不适合大面积用。

## 5.2 应用与注意

★ **关键认知**：混合模式的风险——**兼容差异 + 结果不可控**,且它会创建层叠上下文。⚠ 属于"锦上添花的特效",不是功能必需品;需要时逐场景调,别引进来却撑不起整体视觉一致性。

# 6. 本篇小结与下一步

★ **关键认知**：box-shadow(drop-shadow 沿形状)、filter(模糊/灰度)、clip-path(硬裁剪)、mask(软遮罩)、混合模式,让界面从"能用"到"有质感"。注意 filter/blend 的性能与层叠副作用。特效是调味,不是主菜。

下一步进入**阶段四 · 工程化**,第 17 篇:预处理器 Sass/SCSS——嵌套、变量、mixin、函数。