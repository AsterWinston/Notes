Sass 是"换一种写法",**PostCSS** 则是"处理 CSS 本身"——它把 CSS 解析成 AST,再让插件逐条改写,最出名的用途就是 `autoprefixer`(自动加浏览器前缀)。本篇讲清 PostCSS 是什么、autoprefixer 在做什么、以及它广大的插件生态。

# 1. PostCSS 是什么

## 1.1 定位:CSS 的处理管线

```
  CSS(或 SCSS 编译后的 CSS)
        │
        ▼
  PostCSS 解析成 AST(抽象语法树)
        │
        ▼
  插件们依次改写 AST(A 插件加前缀、B 插件嵌套展开…)
        │
        ▼
  输出处理后的 CSS
```

★ **关键认知**：PostCSS **不是预处理器**,而是**CSS 处理器**——把 CSS 转成 AST,插件在 AST 上做任意改写。⚠ 它与 Sass 的区别:**Sass 有"自己的语法"(变量/mixin),PostCSS 处理的是"标准 CSS"**;一个是"换写法",一个是"动内容"。

## 1.2 它挂在哪

```powershell
# 作为 Vite/webpack 的一个插件运转(第 13 篇构建工具)
npm install -D postcss autoprefixer
```

★ **关键认知**：PostCSS 通常**寄生在构建工具里**(Vite/webpack 的 postcss 配置),构建时自动处理 CSS。⚠ 你几乎不"直接运行" PostCSS,而是通过构建工具或它配套的配置文件(`postcss.config.js`)使用。

# 2. autoprefixer 自动加前缀

## 2.1 省掉手写前缀

```css
/* 你写(标准语法) */
.button { user-select: none; }

/* autoprefixer 自动产出(按 targets 决定加哪些前缀) */
.button {
  -webkit-user-select: none;
  user-select: none;
}
```

★ **关键认知**：`autoprefixer` 依据 `browserslist`(目标浏览器列表)**自动判断哪些前缀需要加**,你只写标准语法即可。⚠ 它解决的是"**手写 `-webkit-`/`-moz-` 前缀**"的苦差——过去要背哪些属性要前缀,现在交给工具。

## 2.2 browserslist

```json
// package.json 里的 browserslist,browserslist 决定的兼容目标
{
  "browserslist": [
    "last 2 versions",
    "> 1%",
    "not dead"
  ]
}
```

★ **关键认知**：`browserslist` 声明"**我要兼容到哪些浏览器**",autoprefixer、Babel 等工具都据此工作。⚠ 这是"兼容策略"的**单一事实来源**——改这一处,前缀、JS 转译、polyfill 都跟着变;别在多个工具里散落不同的兼容配置。

# 3. 常用 PostCSS 插件

## 3.1 扩展语法

```css
/* postcss-nesting:让原生 CSS 也能嵌套 */
.card {
  &:hover { color: red; }
}

/* postcss-preset-env:按 browserslist 转译"未来的 CSS"为现有语法 */
:root { --x: 1; }
```

★ **关键认知**：常见插件——**`postcss-preset-env`**(综合:把未来的/尚不兼容的 CSS 转成兼容写法,可理解为 CSS 的 "babel")、**`postcss-nesting`**(嵌套)。⚠ `preset-env` 一个插件顶一堆,是现代工程的常用首选;它能"用了新 CSS,还能跑在老浏览器"。

## 3.2 工程化插件

```powershell
npm install -D cssnano         # cssnano:压缩 CSS(build 时)
npm install -D postcss-import  # 合并 @import 的文件
```

★ **关键认知**：还有 **`cssnano`**(生产压缩 CSS)、**`postcss-import`**(打包时把 `@import` 的文件内联合并)。⚠ 这些"压缩、合并"的活都归 PostCSS 生态,构建工具串联它们完成产物的优化。

# 4. 与 Sass 的关系

## 4.1 要不要一起用

★ **关键认知**：Sass 与 PostCSS **不冲突,常"串联"**——先 **Sass 编译**(嵌套/变量/mixin 展开)→ 再 **PostCSS 处理**(autoprefixer 加前缀、cssnano 压缩)。⚠ 但随原生 CSS 越来越强(变量、嵌套、min/clamp),**"只用 PostCSS + 原生 CSS"也在兴起**;预处理器不再是必需品,是"要不要额外语法"的选择。

## 4.2 一张对照

| 维度 | Sass | PostCSS |
|------|------|---------|
| 性质 | 预处理器(自有语法) | CSS 处理器(处理标准 CSS) |
| 核心 | 嵌套/变量/mixin/函数 | 插件式的 AST 改写 |
| 前缀 | 不能(可手写 mixin) | autoprefixer 自动 |
| 兼容转译 | 不负责 | preset-env 负责 |
| 关系 | 可选前置 | 构建中几乎必用 |

★ **关键认知**：记住这条分工——**Sass 管"写法的丰富",PostCSS 管"兼容与产出"**。⚠ 现代工程最稳的组合是"SCSS(或原生)+ PostCSS(autoprefixer + preset-env + cssnano)",前两者按偏好选,后者基本必配。

# 5. 实战:一个最小 PostCSS 配置

```javascript
// postcss.config.js
module.exports = {
  plugins: {
    autoprefixer: {},               // 自动加前缀
    'postcss-preset-env': {},       // 转译未来 CSS
    ...(process.env.NODE_ENV === 'production' ? { cssnano: {} } : {})
  }
}
```

★ **关键认知**：一套典型 PostCSS 配置 = **autoprefixer + preset-env + (生产时) cssnano**。⚠ 插件**顺序**看它们各自 README(多为声明式,无需纠结);关键是"前缀→转译→压缩"这条处理链,交给工具就是我们的心态。

# 6. 本篇小结与下一步

★ **关键认知**：PostCSS 是"处理标准 CSS"的 AST 管线,插件各司其职——autoprefixer 加前缀(按 browserslist)、preset-env 转译未来语法、cssnano 压缩。它藏在构建工具里,是"兼容与产出"的自动化保证。

下一步第 19 篇:CSS 架构方法论——BEM、OOCSS、SMACSS、CSS Modules,如何让样式在大型项目里不乱。