CSS 不止有"写死的值",还有**变量**(自定义属性)与**函数**(calc/min/max/clamp)——前者让样式"定义一次、处处复用、可动态改",后者让值"会计算、会自适应"。本篇讲清自定义属性的作用域与继承、`calc` 的运算规则,以及 `clamp()` 等现代函数。

# 1. CSS 变量(自定义属性)

## 1.1 定义与使用

```css
:root {
  --primary: #42b883;      /* 全局定义(通常放 :root,html 上) */
  --gap: 16px;
}

.button {
  background: var(--primary);   /* 用 var() 读取 */
  padding: var(--gap);
}
```

★ **关键认知**：CSS 变量(自定义属性)以 `--` 开头,通过 `var()` 使用;定义在 `:root`(html)上即成**全局变量**。⚠ 它值里可存**任何东西**(颜色、尺寸、整段值),甚至可存别的变量引用,是"设计系统/主题"的地基——改 `--primary` 一处,全站主色替换。

## 1.2 作用域与默认值

```css
.card {
  --padding: 20px;         /* 局部定义:只在 card 内有效 */
  padding: var(--padding);
}
.card .big {
  --padding: 40px;         /* 越靠近的元素覆盖 */
  padding: var(--padding);
}

/* var() 第二参数:变量未定义时的回退值 */
color: var(--undefined-color, #333);
```

★ **关键认知**：自定义属性**遵循级联**(继承 + 就近覆盖)——父定义子继承,更近的元素可覆盖;`var(--x, 默认)` 提供**回退值**。⚠ 变量的作用域与覆盖正是"局部主题、暗色模式"的实现基础:在某个容器上重定义 `--primary`,容器内自动换肤。

## 1.3 与 JS 联动

```javascript
// JS 动态改 CSS 变量,实现"运行时主题/尺寸"
document.documentElement.style.setProperty('--primary', '#ff0000')
```

★ **关键认知**：`setProperty` 能在**运行时改 CSS 变量**——这是"JS 控制样式"最优雅的通道(改一个变量全站生效,不用逐个元素改 class)。⚠ 主题切换、用户自定义色这类需求的现代实现,几乎都走"JS 改 CSS 变量"这条路。

# 2. calc 计算

## 2.1 基本运算

```css
.sidebar {
  width: calc(100% - 200px);      /* 计算 */
  height: calc(100vh - 60px);     /* 混合单位 */
  font-size: calc(1rem + 1vw);    /* 流式尺寸 */
}
```

★ **关键认知**：`calc()` 做**数学运算**,支持 `+ - * /`,还能**混合单位**(`100% - 200px`)。⚠ 铁律:**`+` 和 `-` 两边必须带空格**(`calc(100% - 200px)`,写成 `calc(100%-200px)` 无效);`*` `/` 则不需要,但乘除的一方只能是数字。

## 2.2 多背景里的嵌套

```css
.box { width: calc(50% - 20px); }
.box { width: calc(var(--gap) * 2 + 10px); } /* calc 里用变量 */
```

★ **关键认知**：`calc` 可**嵌套使用变量**(`var(--gap) * 2`),让"间距的倍数""尺寸的差值"这类需求表达得干净。⚠ calc 和变量的组合是响应式布局的常用工具:定宽侧栏 + `calc(100% - 侧栏宽)` 主区。

# 3. min / max / clamp

## 3.1 min 与 max

```css
.box { width: min(100%, 600px); }   /* 取较小者:不超容器、最多 600 */
.box { width: max(300px, 50%); }    /* 取较大者:至少 300 */
```

★ **关键认知**：`min()`/`max()` 从多个值里**取最小/最大**。⚠ `min(100%, 600px)` 的典型用途是"**响应式最大宽**"——小屏时 100%、大屏封顶 600px,一条属性替代"width + max-width"两条。

## 3.2 clamp 夹取

```css
.title { font-size: clamp(1rem, 4vw, 2rem); }
/* clamp(小, 理想, 大):理想值 4vw,但不小于 1rem、不大于 2rem */
.box { padding: clamp(10px, 3vw, 30px); }
```

★ **关键认知**：`clamp(min, ideal, max)` 是"**有上下限的流式值**"——理想值随视口变化,但被夹在 min/max 之间。⚠ `clamp(1rem, 4vw, 2rem)` 的**流式字号**几乎取代了"每档断点重设字号"的老写法,是响应式排版的现代标配。

# 4. 实战:可主题化的设计系统

## 4.1 变量 + 函数组合

```css
:root {
  --spacing: 16px;
  --primary: #42b883;
  --content-max: 1200px;
}

.container {
  width: min(100%, var(--content-max));       /* 响应式最大宽 */
  padding: var(--spacing);
}
section + section {
  margin-top: calc(var(--spacing) * 2);       /* 间距的倍数 */
}
```

★ **关键认知**：**变量 + 函数** 是"设计系统"的骨架——间距、色值、尺寸抽成变量,配合 `min/calc` 做响应式计算,改一处全局生效。⚠ 这比散落各处的魔法数字可维护得多,也是进入工程化(第 17~19 篇)前的环境过渡。

## 4.2 暗色模式

```css
:root { --bg: #fff; --text: #333; }
[data-theme="dark"] { --bg: #1e1e1e; --text: #eee; }

body { background: var(--bg); color: var(--text); }
```

★ **关键认知**：暗色模式 = **在根上重定义变量**——`[data-theme="dark"]` 覆盖 `--bg`/`--text`,依赖这些变量的元素自动换肤。⚠ 这是 CSS 变量最出圈的用法:JS 只需给 html 切换一个 data 属性,全站换肤完成。

# 5. 本篇小结与下一步

★ **关键认知**：CSS 变量(`--x` + `var`)让值可复用、可作用域、可动态改(主题/换肤的基石);`calc` 混单位计算(注意运算符空格);`min/max/clamp` 让单个属性自身响应式。三者的组合,是"可配置、会计算"的现代 CSS。

下一步第 15 篇:BFC 与常见坑——格式化上下文、外边距折叠、以及那些"莫名其妙"的布局问题。