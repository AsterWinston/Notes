文字是网页内容的主体,排版水平直接决定可读性。本篇讲清 `font` 与 `text` 两大属性族、`line-height` 与垂直居中的关系、`@font-face` 引入自定义字体,以及 font 简写的坑。

# 1. font 属性族

## 1.1 字体基础

```css
p {
  font-family: "Helvetica Neue", Arial, sans-serif; /* 字体族,可列多个依次回退 */
  font-size: 16px;          /* 字号 */
  font-weight: 700;         /* 字重(100~900,normal=400,bold=700) */
  font-style: italic;       /* 斜体 */
}
```

★ **关键认知**：`font-family` 列**多个字体依次回退**(第一个没有就用下一个);结尾通常放**通用族**(`sans-serif`/`serif`/`monospace`)。⚠ `font-weight` 的数值只有**加载了对应字重的字体**时才真正区分,否则浏览器只能粗化模拟;中文字体名要加引号(含空格更必须)。

## 1.2 字号单位

```css
p  { font-size: 16px; }    /* 像素,绝对稳定 */
h1 { font-size: 2em; }     /* em:相对父元素字号(2 倍) */
h2 { font-size: 1.5rem; }  /* rem:相对根元素 html 字号 */
```

★ **关键认知**：字号单位里 **`rem` 相对根元素(html)字号、`em` 相对父元素字号**。⚠ `em` 会"复合"(父套子越套越大),现代排版**优先 `rem`**(基准稳定,改 html 一处整页缩放);`px` 则用于需要精确控制不随缩放的地方。

# 2. text 排版属性

## 2.1 对齐与装饰

```css
p {
  text-align: center;            /* 水平对齐:left/center/right/justify */
  text-decoration: underline;    /* 下划线(删除线 line-through) */
  text-transform: uppercase;     /* 大小写转换 */
  text-indent: 2em;              /* 首行缩进 */
  letter-spacing: 0.5px;         /* 字间距 */
  word-spacing: 2px;             /* 词间距 */
}
```

★ **关键认知**：`text-align` 对齐、`text-decoration` 装饰线、`letter-spacing` 字距是排版常用。⚠ `text-align: justify`(两端对齐)在中英文混合时可能产生难看的空白,常需配合 `text-justify` 与断词规则。

## 2.2 空白与换行处理

```css
.box {
  white-space: nowrap;      /* 不换行 */
  overflow-wrap: break-word; /* 长单词强制断行 */
  word-break: break-all;     /* 更激进的断词 */
  text-overflow: ellipsis;   /* 超出显示省略号(配 overflow:hidden+nowrap) */
}
```

★ **关键认知**：`white-space` 控制空白与换行保留;`overflow-wrap`/`word-break` 处理长单词溢出;`text-overflow: ellipsis` 做**单行省略号**(需要 nowrap + overflow:hidden 配合)。⚠ 单行省略号是高频需求,三件套缺一不可。

# 3. line-height 行高

## 3.1 行高的作用与取值

```css
p { line-height: 1.6; }     /* 无单位数值:相对自身字号的倍数(推荐) */
p { line-height: 24px; }    /* 固定值 */
p { line-height: 150%; }    /* 百分比(注意:和 em 一样会继承计算后的固定值) */
```

★ **关键认知**：`line-height` 是**文字总行高**(内容区 + 上下半行距),决定行与行的距离。⚠ 取值推荐**无单位数字**(如 `1.6`)——它让后代按**自身字号**计算;而 `%`/`em` 会**继承"计算后的固定值"**,后代字号变大时行距反而挤。

## 3.2 单行文本垂直居中

```css
/* 经典技巧:line-height 等于容器高度,单行文字垂直居中 */
.button {
  height: 40px;
  line-height: 40px;   /* 行高=高度,文字在行框中垂直居中 */
}
```

★ **关键认知**：单行文字垂直居中,`line-height = height` 是最省事的写法(文字在行框内居中)。⚠ 它只对**单行**有效,多行居中交给 flex 的 `align-items: center`(第 8 篇)。

# 4. web 字体 @font-face

## 4.1 引入自定义字体

```css
/* 定义字体:起名 + 指定字体文件源 */
@font-face {
  font-family: "MyFont";
  src: url("./fonts/myfont.woff2") format("woff2"),
       url("./fonts/myfont.woff") format("woff");
  font-weight: 400;                 /* 该文件对应的字重 */
}

/* 使用 */
body { font-family: "MyFont", sans-serif; }
```

★ **关键认知**：`@font-face` 把**外部字体文件**注册成可用的 `font-family`。⚠ 字体格式优先 **woff2**(体积最小,现代通用),可列多个格式回退;一个 @font-face 对应**一个字重**,粗体/斜体需分别声明。

## 4.2 字体加载的性能

```css
/* font-display 控制字体加载期间的显示策略 */
@font-face {
  font-family: "MyFont";
  src: url("myfont.woff2") format("woff2");
  font-display: swap;   /* 先显示系统字体,加载完换成自定义 */
}
```

★ **关键认知**：`font-display: swap` 是网页字体**性能与体验的平衡点**——先用系统字体渲染(避免白屏),字体到了再替换。⚠ 不写 `font-display` 默认 `block` 会导致字体加载期间**文字不可见**(FOIT),拖慢感知速度。

# 5. 本篇小结与下一步

★ **关键认知**：font 管"什么字体、多大、多粗",text 管"怎么排、怎么对齐、怎么断行",line-height 管行距,字体优先 rem + 无单位行高,web 字体用 @font-face + font-display:swap。排版的好坏,一半在这些细节里。

下一步第 5 篇:颜色与背景——颜色的多种表示、background 属性、渐变与多背景。