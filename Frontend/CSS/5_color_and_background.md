颜色与背景是界面"第一印象"的载体——从纯色到渐变再到多背景叠加。本篇讲清颜色的五种表示方式、`background` 的子属性、线性/径向渐变,以及"一个元素叠多层背景"的写法。

# 1. 颜色的表示

## 1.1 五种写法

```css
.box {
  color: red;                  /* 颜色名 */
  color: #ff0000;              /* 十六进制(HEX) */
  color: rgb(255, 0, 0);       /* RGB */
  color: rgba(255, 0, 0, 0.5); /* RGBA 带透明度 */
  color: hsl(0, 100%, 50%);    /* HSL(色相/饱和度/亮度) */
  color: hsl(0 100% 50% / 0.5);/* 现代语法,带透明度 */
}
```

★ **关键认知**：五种颜色表示——**HEX**(最常用)、**RGB/RGBA**、**HSL/HSLA**(按"色相/饱和度/亮度"思考,调色更直觉)、颜色名。⚠ **HEX 简写** `#f00` 等价 `#ff0000`;HSL 比 RGB 更符合"我想调亮/调灰一点"的心智,做主题色系时好用。

## 1.2 透明度与函数

```css
.box {
  background: rgb(0 0 0 / 0.5);  /* 现代函数语法,/ 后是透明度 */
  background: #00000080;         /* 8 位 HEX,末两位是透明度 */
}
```

★ **关键认知**：透明度的三种途径——`rgba()`、现代空格语法 `/ 0.5`、8 位 HEX 的**末尾两位**(`80` 即 50%)。⚠ **只让背景透明**用 `rgba`,**整个元素透明**(含文字)才用 `opacity`——二者语义不同,别混。

# 2. background 背景

## 2.1 子属性

```css
.box {
  background-color: #fff;          /* 背景色 */
  background-image: url(bg.png);   /* 背景图 */
  background-repeat: no-repeat;    /* 平铺方式 */
  background-position: center;     /* 位置 */
  background-size: cover;          /* 尺寸 */
  background-attachment: scroll;   /* 滚动/固定 */
}
```

★ **关键认知**：`background-size` 是最常用的一个——`cover`(铺满容器、可能裁剪)与 `contain`(完整放入、可能留白)。⚠ **背景图头部居中**的"全屏图"标准组合是 `background: url(...) center/cover no-repeat`;背景图**不占布局空间**,只作装饰。

## 2.2 background 简写

```css
.box {
  /* 简写:顺序大体为 颜色/图/位置/尺寸/平铺 */
  background: #f5f5f5 url(bg.png) center/cover no-repeat;
}
```

★ **关键认知**：`background` 简写能一次写全,`position/size` 用 `/` 分隔。⚠ 简写会**重置所有背景子属性**(没写的回到默认),所以"先简写、后单独覆盖某个子属性"时要注意顺序与覆盖。

# 3. 渐变 gradient

## 3.1 线性渐变

```css
.box {
  background: linear-gradient(to right, red, blue);        /* 方向:从左到右 */
  background: linear-gradient(45deg, red, white, blue);    /* 45 度 + 多色 */
  background: linear-gradient(red 0%, blue 100%);          /* 颜色带位置 */
}
```

★ **关键认知**：`linear-gradient` 线性渐变——**方向 + 颜色列表**(可带停靠位置百分比)。⚠ 方向关键词 `to right`(向右)、`to bottom`(默认,向下);多个颜色会平滑过渡,颜色后跟百分比可控制"从哪变色"。

## 3.2 径向渐变与多背景

```css
.circle {
  /* 径向渐变:从中心向外 */
  background: radial-gradient(circle, white, black);
}

.layer {
  /* 多背景:逗号分隔,先写的在上层 */
  background:
    linear-gradient(rgba(0,0,0,0.5), rgba(0,0,0,0.5)),
    url(photo.jpg);
}
```

★ **关键认知**：`radial-gradient` 从中心向外渐变(适合光斑、聚光);**多背景用逗号分隔,先写的叠在上层**。⚠ "图片 + 半透明渐变遮罩"是**文字加暗层**的经典写法(背景图上层罩一个半透明渐变色),用于提升背景图上文字的可读性。

# 4. 实用场景

## 4.1 背景图封面 + 文字遮罩

```css
.hero {
  height: 400px;
  background:
    linear-gradient(rgba(0,0,0,0.4), rgba(0,0,0,0.4)),
    url(hero.jpg) center/cover no-repeat;
  color: #fff;   /* 文字白色,压在半透明黑罩上,可读 */
}
```

★ **关键认知**：这个组合是**首屏 Banner 的万能模板**——背景图铺满 + 半透明遮罩 + 白字,兼顾美观与可读。⚠ 遮罩颜色/透明度可按需调;纯渐变(无图)也常用来做按钮、渐变色块。

# 5. 本篇小结与下一步

★ **关键认知**：颜色有 HEX/RGB/HSL 等写法(透明度用 rgba/8位HEX);背景靠 background 子属性(cover 铺满);渐变分 linear/radial;多背景逗号分隔、先写在先。核心套路"背景图 + 半透明遮罩 + 白字"值得记牢。

下一步第 6 篇:定位与层叠上下文——position 五种值、z-index 的玄学,与层叠上下文的真相。