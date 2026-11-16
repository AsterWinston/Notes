颜色与背景是界面"第一印象"的载体——从纯色到渐变再到多背景叠加。本篇讲清颜色的表示方式、`currentColor` 与新一代色彩函数、`background` 子属性与简写规则、渐变(含停靠位置与插值空间),以及"一个元素叠多层背景"的写法。

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

## 1.3 currentColor 与引用色

```css
.btn {
  color: #2563eb;
  border: 2px solid currentColor;  /* 边框直接引用当前文字色 */
}
.btn:hover { color: #dc2626; }     /* 只改 color,边框自动跟着变 */
.icon { fill: currentColor; }      /* SVG 图标继承文字颜色 */
```

★ **关键认知**：`currentColor` 不是固定颜色,而是**"当前元素 `color` 解析后的值"**——一个活的引用,改 `color` 一处,所有用了它的属性(border、outline、SVG fill/stroke…)联动。⚠ 做**跟随主题色的图标**,SVG 里写 `fill="currentColor"` 是关键一步;换肤系统"一处 color 全局联动"就靠它。

## 1.4 现代色彩函数:oklch、color() 与混色

```css
.box {
  color: oklch(70% 0.15 250);                 /* 感知均匀的 亮度/彩度/色相 */
  color: color(display-p3 1 0.5 0);           /* 广色域(高端屏更艳) */
  color: color-mix(in oklab, red 60%, blue);  /* 两色按比例混合 */
}
```

★ **关键认知**：`rgb/hsl` 建立在 **sRGB** 上,"亮度"与**人眼感知不成正比**——L 减半看起来远不止暗一半,HSL 调出的同 L 色阶有的显亮有的显暗。`oklch()` 改成**感知均匀**(同 L 视觉上真差不多亮),`color-mix()` 直接说"红色掺 60% 蓝"。⚠ 兼容:`oklch/color-mix` 需 2023 年后的浏览器,老环境准备 sRGB 兜底。

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

/* 硬规则演示:color 不在最后 → 整条声明失效 */
.c { background: center/cover #f5f5f5 url(bg.png); }
```

★ **关键认知**：`background` 简写能一次写全,`position/size` 用 `/` 分隔。⚠ 简写会**重置所有背景子属性**(没写的回到默认),"先简写、后单独覆盖某个子属性"要注意顺序与覆盖。顺序有两条硬规则:**background-color 必须放最后**、**size 必须紧贴 position 用 `/` 分隔**——违反其一**整条声明失效**(不是部分生效),这是"简写死活不生效"的第一排查点。

## 2.3 background-position 百分比的原理

```text
偏移量 = (容器尺寸 - 图像尺寸) × 百分比
┌──────── 500px 容器 ────────┐
│ ┌── 300px 图 ──┐           │  差值 200px → 50% 时偏移 100px(居中)
│ └──────────────┘ → 100% 贴右│
└─────────────────────────────┘
```

★ **关键认知**：`background-position: 50%` **不是"图像左上角放到容器的 50% 处"**,而是"**图像自身的 50% 点对齐容器自身的 50% 点**",等价公式 `(容器-图像)×百分比`。⚠ 这解释了:小图在大容器里 `100% 100%` 恰好贴右下角而不是溢出;大图被 `cover` 裁切时,`0% 0%` 露左上、`50% 50%` 露中间——"背景图永远聚焦某个人物",调的就是这个百分比。

## 2.4 多重背景与 background-blend-mode

```css
.hero {
  background:
    radial-gradient(circle at 30% 40%, rgb(255 255 255 / .25), transparent 60%),
    url(texture.png);
  background-blend-mode: overlay, normal;  /* 逐层混合,列表与层对应 */
}
```

★ **关键认知**：多背景**逗号分隔,先写的在上层**,每层可以有自己的位置/尺寸;`background-blend-mode` 也写成逗号列表,**让相邻背景层按算法混合**(multiply 压暗、screen 提亮、overlay 加质感)。⚠ 它和第 16 篇的 `mix-blend-mode` 分工不同:blend-mode 只在**背景层彼此之间**混,不和页面下层内容混——更可控也更便宜;"深色底+纹理+光斑"的氛围背景就这么叠。

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

## 3.3 color-stop 的位置与硬断点

```css
/* 同一色写两个位置 = 硬断点(段内纯色,交界处无过渡带) */
.stripes {
  background: linear-gradient(to right, #ef4444 0 25%, #3b82f6 25% 50%, #ef4444 50% 75%, #3b82f6 75% 100%);
}
.barber {
  /* repeating-* 把整段渐变无限重复:斜纹进度条/警戒条纹 */
  background: repeating-linear-gradient(45deg, #000 0 10px, #fff 10px 20px);
}
```

★ **关键认知**：每个 color-stop 可以带**两个位置**(`red 0 25%`)——第一个位置颜色铺满、第二个位置才让给下一色,相邻色边界**硬切**;`repeating-linear/radial-gradient` 把渐变**周期重复**。⚠ 这套"位置语法"是条纹、饼图(conic)、色带刻度等图形的根基,**不需要任何图片**;老写法 `red 25%, red 25%` 连写两遍是同一件事的旧式表达。

## 3.4 渐变的插值空间

```css
.srgb  { background: linear-gradient(to right, red, blue); }           /* 默认:sRGB 插值 */
.ok    { background: linear-gradient(to right in oklch, red, blue); }  /* 中段不再发灰 */
.hue   { background: linear-gradient(in hsl longer hue to right, red, blue); } /* 色相绕远路 */
.fade  { background: linear-gradient(rgb(0 0 0 / 0), black); }          /* 透明段别用 transparent */
```

★ **关键认知**：渐变是在**色彩空间里逐点插值**的——默认 sRGB 下红到蓝的中段会经过**浑浊的灰紫**;`in oklch` 让中段保持鲜艳,`longer hue`(还有 `shorter/decreasing/increasing`)控制色相绕哪条路走。⚠ 历史坑:`transparent` 参与插值时等价 `rgba(0,0,0,0)`,透明白到不透明白会**途经灰色**——"淡出到白"请写 `rgb(255 255 255 / 0)`。`in <color-space>` 语法要新浏览器(2023+)。

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

★ **关键认知**：颜色有 HEX/RGB/HSL 等写法(透明度用 rgba/8位HEX),进阶有 `currentColor` 联动与 `oklch/color-mix` 感知均匀调色;background 简写只有"color 最后、size 紧跟 position"两条硬顺序规则,百分比定位按 `(容器-图像)×百分比` 计算;渐变分 linear/radial/conic,color-stop 双位置做硬断点、`in oklch` 修插值发灰;多背景逗号分隔、先写在先,可叠 `background-blend-mode`。核心套路"背景图 + 半透明遮罩 + 白字"值得记牢。

下一步第 6 篇:定位与层叠上下文——position 五种值、z-index 的玄学,与层叠上下文的真相。
