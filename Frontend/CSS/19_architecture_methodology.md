CSS 的全局性是把双刃剑——方便,但项目一大就"类名漫天飞、样式互相覆盖"。**架构方法论**就是给这种混乱立规矩。本篇讲清 BEM、OOCSS、SMACSS 三大经典方法论、现代的 CSS Modules,以及"在大型项目里怎么选"。

# 1. CSS 的全局性难题

## 1.1 为什么要"架构"

```css
/* 全局性:谁都能往 .title 上写,越写越互相覆盖 */
/* a.css */  .title { color: red; }
/* b.css */  .title { font-size: 30px; }
```

★ **关键认知**：CSS 选择器**全局生效、层叠冲突、无作用域**,项目越大越容易"改 A 崩了 B"。⚠ 架构方法论的本质,是**用命名/组织约定,人为制造"隔离与可预测"**——让"这个类属于谁、长什么样"一目了然。

# 2. 三大经典方法论

## 2.1 BEM(块/元素/修饰符)

```css
/* Block__Element--Modifier */
.card { }                 /* 块 */
.card__title { }          /* 元素(块的组成部分) */
.card--featured { }       /* 修饰符(状态/变体) */
.card__title--large { }
```

```html
<div class="card card--featured">
  <h2 class="card__title card__title--large">标题</h2>
</div>
```

★ **关键认知**：BEM 把类名拆成 **块(Block)__元素(Element)--修饰符(Modifier)** 三段,约定清晰、见名知义。⚠ 它是**最流行**的方法论,短板是类名**略显冗长**;但换来的是"**零级联依赖、平铺可读**",大型项目里非常耐维护。修饰符是 **--**,元素是 **__**,别记反。

## 2.2 OOCSS(面向对象)

```css
/* 结构与皮肤分离 + 容器与内容分离 */
.btn { padding: 8px 16px; }        /* 结构 */
.btn-primary { background: blue; }  /* 皮肤(可叠加) */

/* 复用而非重复 */
```

★ **关键认知**：OOCSS 两条原则——**结构(skin)与皮肤(structure)分离、容器与内容分离**,追求**最大化复用**(一套 `.btn` + 变体而不是每个按钮写一遍)。⚠ 它的思想沉淀到了"组件化"里;BEM 的"块"概念也受它影响,可理解为 BEM 的父辈。

## 2.3 SMACSS(分层)

```css
/* 按"类别"分层组织:CSS 分五类 */
/* Base(元素默认) / Layout(布局) / Module(组件) / State(状态) / Theme(主题) */
```

★ **关键认知**：SMACSS 的核心是**"分层"**——把样式按职责分成 base/layout/module/state/theme,写清楚"这一层能不能覆盖另一层"。⚠ 它更像一套**组织规范**而非命名约定,常与 BEM 的命名结合使用。

# 3. 现代方案:CSS Modules

## 3.1 作用域隔离

```css
/* style.module.css */
.title { color: red; }
```

```jsx
// 组件里按"模块"导入,拿到哈希化后的类名
import styles from './style.module.css'
<p className={styles.title}>...</p>
```

★ **关键认知**：CSS Modules 把每个 CSS 文件当**独立作用域**——类名被**哈希化**(`.title` → `.title_1a2b3`),通过构建工具导出映射对象。⚠ 它**从机制上根除了命名冲突**(类名唯一),是"组件化时代"最干净的隔离方案;与 Vue 的 `scoped`(Vue 第 14 篇)是同一诉求的不同实现。

## 3.2 与 scoped 对照

| 维度 | CSS Modules | Vue scoped |
|------|-------------|-----------|
| 隔离手段 | 类名哈希 | 属性选择器 `[data-v]` |
| 冲突 | 根除 | 隔离(可 :deep 穿透) |
| 适用 | React/通用 | Vue |
| 命名 | 仍可语义化(局部) | 仍可语义化 |

★ **关键认知**：CSS Modules 与 Vue scoped 的目标一致(**组件级样式隔离**),手段不同(哈希 vs 属性)。⚠ 在 JS 里取类名选 Modules,在 `.vue` 里选 scoped;两者都让"命名冲突"这个 CSS 老问题在组件边界内自动消失。

# 4. 怎么选

| 场景 | 推荐 |
|------|------|
| 组件化项目(Vue/React) | scoped / CSS Modules(隔离天然解决) |
| 传统多页/全局 CSS | BEM(命名约定清晰) |
| 设计系统/组件库 | BEM + 变量(CSS 变量统一 token) |
| 极致复用 | OOCSS 思想 + 原子化(见第 20 篇) |

★ **关键认知**：选的本质是"**用工具隔离还是用人肉约定**"——组件化项目直接上 **scoped/CSS Modules**(机制隔离),没有组件边界的传统 CSS 项目用 **BEM**(约定隔离)。⚠ 最忌"**没有约定、也没有隔离**",那才是项目后期样式崩坏的根源。

# 5. 一个落地的 BEM 例子

```css
/* 组件:卡片 card,带标题、按钮,可 featured 高亮 */
.card { border: 1px solid #eee; }
.card__title { font-size: 20px; }
.card__action { padding: 8px; }
.card--featured { border-color: #42b883; }
.card--featured .card__title { color: #42b883; }
```

```html
<article class="card card--featured">
  <h2 class="card__title">标题</h2>
  <button class="card__action">按钮</button>
</article>
```

★ **关键认知**：落地 BEM 的关键是**每个组件一套 Block 前缀**,元素用 `__`、状态用 `--`,**不要裸写 `.title` 这种全局类**。⚠ 名字长了点,但换来的是"一眼看懂这个类属于哪个组件、什么变体",大项目里这笔账很划算。

# 6. 本篇小结与下一步

★ **关键认知**：CSS 全局性→需要架构。经典方法论:BEM(命名,__/--)、OOCSS(复用)、SMACSS(分层);现代机制:CSS Modules/scoped(组件级哈希隔离)。选型:"组件化用隔离机制,传统项目用 BEM 约定"。

下一步第 20 篇:Tailwind 与原子化 CSS——utility-first 的颠覆式写法,以及它的取舍。