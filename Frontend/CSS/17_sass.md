原生 CSS 用久了会感觉啰嗦——选择器层层加盖、值四处重复、没有函数。**Sass/SCSS** 补上这些:嵌套、变量、mixin、函数、模块化。本篇讲清 SCSS 的核心特性与编译方式,以及它与原生 CSS 的关系(哪些已被原生取代)。

# 1. 什么是 Sass/SCSS

## 1.1 两种语法

```scss
// SCSS:在 CSS 基础上扩展(推荐,与 CSS 兼容)
$color: red;
.box { color: $color; }

// 缩进语法(Sass):去大括号,靠缩进
$color: red
.box
  color: $color
```

★ **关键认知**：Sass 是一门**CSS 预处理语言**,有两种语法——**SCSS**(与 CSS 兼容,主流)与**缩进版 Sass**。⚠ 现代项目**都用 SCSS**;它会被编译成普通 CSS,浏览器看到的是编译产物。

## 1.2 编译方式

```powershell
# 通过 Vite 等构建工具直接支持(见第 13 篇 Vite)
# 或命令行:Sass 编译
npx sass input.scss output.css          # 单次编译
npx sass input.scss output.css --watch  # 监听编译
```

★ **关键认知**：Sass 由**构建工具(Vite/webpack)或 sass CLI** 编译成 CSS。⚠ 现代前端工程里,Vite 开箱支持 `lang="scss"`(见 Vue 第 14 篇),几乎不用手动跑 sass 命令。

# 2. 嵌套

## 2.1 选择器嵌套

```scss
.nav {
  ul { list-style: none; }          /* 编译成 .nav ul */
  li {
    display: inline;
    &:hover { color: red; }        /* & 指代父选择器,编译成 .nav li:hover */
  }
}
```

★ **关键认知**：嵌套让选择器**层级清晰**,`&` 指代**父选择器**(用于伪类、状态)。⚠ 嵌套过深(3 层以上)会产出**超长、过度具体**的选择器(`.nav ul li a span`),既难覆盖又低性能——**别超过 3 层**是 SCSS 的公认纪律。

## 2.2 属性嵌套

```scss
.box {
  font: {
    family: serif;
    size: 16px;      /* 编译成 font-size:16px */
    weight: 700;
  }
  border: { top: 1px solid; bottom: 1px solid; }
}
```

★ **关键认知**：属性嵌套把 `font-size`、`border-top` 这类**带连字符的属性**分组撰写(属小众特性)。⚠ 用处有限,团队里若没人用可略过,重点是选择器嵌套。

# 3. 变量与运算

## 3.1 变量

```scss
$primary: #42b883;
$spacing: 16px;

.card {
  color: $primary;
  padding: $spacing * 2;      /* 可运算 */
}
```

★ **关键认知**：SCSS 变量 `$name` 在**编译期**替换,能做**运算**(`$spacing * 2`)。⚠ 与原生 CSS 变量(`--x`)的区别:SCSS 变量是**编译时**的(编译后消失,不可运行时改),CSS 变量是**运行时**的(可 JS 动态改)——**做主题/动态值用 CSS 变量,做编译期复用/运算用 SCSS 变量**。

## 3.2 插值

```scss
$name: "primary";
.#{$name}-btn { ... }        /* #{} 插值:编译成 .primary-btn */
```

★ **关键认知**：`#{}` 插值把**变量塞进选择器/属性名**里,用于批量生成类名。⚠ 用得少,但生成"同一套前缀的系列类"时会很省。

# 4. mixin 与函数

## 4.1 mixin(混入)

```scss
@mixin center {
  display: flex;
  justify-content: center;
  align-items: center;
}
@mixin box($w, $h: $w) {     /* 带参数、默认值 */
  width: $w; height: $h;
}

.card {
  @include center;           /* 引入 mixin */
  @include box(100px);
}
```

★ **关键认知**：`@mixin` 定义**可复用的样式块**(可带参数/默认值),`@include` 引入。⚠ 与"类"的区别是 mixin **展开到调用处**(复制一份代码),适合**带参数**的复用(如 `@mixin size($w,$h)`);不带参数的纯复用应优先**用类/占位符(%placeholder)**,mixin 会放大产物体积。

## 4.2 函数与条件

```scss
@function half($n) {         /* 自定义函数 */
  @return $n / 2;
}
.card { width: half(100px); }

@mixin theme($dark: false) { /* 条件逻辑 */
  @if $dark {
    background: #000; color: #fff;
  } @else {
    background: #fff; color: #000;
  }
}
```

★ **关键认知**：`@function` + `@return` 定义**返回值的函数**,`@if/@else` 支持逻辑条件。⚠ **函数用于"算值"(返回一个值),mixin 用于"输出一段样式"**——想 `width: half(100px)` 就写函数,想 `@include center` 就写 mixin。

# 5. 模块化与工程化

## 5.1 @use 与 partial

```scss
// _variables.scss(下划线开头 = partial,不会被单独编译)
$primary: #42b883;

// main.scss
@use 'variables' as v;        // 引入 partial,命名空间
.card { color: v.$primary; }
```

★ **关键认知**：SCSS 用 **partial(下划线前缀文件)** + **`@use`** 做模块化——变量/mixin 分文件管理,`@use 'x' as ns` 引入并加命名空间。⚠ **`@use` 取代了老的 `@import`**(后者会全局污染命名空间、重复编译);现代 sass 里 `@import` 已弃用。

## 5.2 工程结构建议

```
src/styles/
├── _variables.scss    # 变量(颜色、间距、断点)
├── _mixins.scss       # 公共 mixin
├── _base.scss         # 基础 reset
└── main.scss          # 入口:@use 汇总
```

★ **关键认知**：SCSS 工程化惯例——**变量/mixin/base 分成 partial,由入口 `@use` 汇总**。⚠ 这套"设计 token 集中管理"的结构,是组件库/大项目样式可维护的基础,也是进入第 19 篇"CSS 架构"的衔接。

# 6. 本篇小结与下一步

★ **关键认知**：SCSS 补的能力——**嵌套(≤3 层)、`$变量`(编译期运算)、mixin(复用样式块)、函数(算值)、@use(模块化)**。注意区分:编译期用 SCSS 变量,运行时动态用 CSS 变量;复用带参数用 mixin,纯复用用类。

下一步第 18 篇:PostCSS 与 autoprefixer——让现代 CSS 兼容老浏览器、用插件扩展 CSS。