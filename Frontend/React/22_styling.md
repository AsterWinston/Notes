React 不管样式,于是样式方案百花齐放:**CSS Modules**(作用域隔离)、**CSS-in-JS**(把样式写进 JS)、**原子化 Tailwind**(工具类)。本篇逐一讲清它们的写法、优劣与选型,回应"组件化时代 CSS 怎么管"这个经典问题。

# 1. 全局 CSS 的困境

## 1.1 组件化需要样式隔离

```css
/* 全局 title 会互相覆盖、名字撞车 */
.title { color: red; }
```

★ **关键认知**：常规 CSS 是**全局的**,组件化项目里"每个组件一份样式"会互相污染(撞类名)。⚠ React 的样式方案,核心诉求都是**"给样式加作用域/隔离"**——与 Vue 的 scoped 面对同一问题(Vue 第 14 篇、CSS 第 19 篇)。

## 1.2 选择器的权重战争

```text
全局 CSS 的隐性成本不止撞名,还有权重叠加:
  .card .title (0,2,0) / .title (0,1,0) / #nav .title (1,1,0)
  同一元素命中多条,谁赢看"权重+书写顺序"——改一条,要猜其余几条叠加结果

组件库更怕"外部穿透":页面里一个裸 .title 就能改掉你组件的标题
  → CSS Modules 的哈希名同时挡住两个方向:我污染不了别人,别人选不到我
```

★ **关键认知**：老项目里 `.card > .title` 越写越长、`!important` 越用越多,本质是**全局选择器下的防御性膨胀**——CSS Modules 把"防御"从写法问题变成编译问题(哈希名无人能撞),这才是它"隔离"二字的完整含义。

# 2. CSS Modules

## 2.1 作用域哈希

```css
/* Button.module.css */
.button { background: #42b883; }
```

```jsx
import styles from './Button.module.css'

function Button() {
  return <button className={styles.button}>按钮</button>  // 类名被哈希化,局部作用域
}
```

★ **关键认知**：CSS Modules 把类名**哈希化**成局部作用域,`styles.button` 拿到唯一类名,冲突根除。⚠ 它依然写**真实 CSS**(有完整 CSS 能力),只是"类名隔离"——这是最接近"传统 CSS + 隔离"的方案,学习成本最低。

## 2.2 组合与全局

```css
/* composes:复用别的类 */
.button { composes: base from './base.module.css'; }
:global(.global-class) { /* 故意全局 */ }
```

★ **关键认知**：CSS Modules 支持 `composes` 复用、`:global()` 破例全局。⚠ 日常组件化用它很舒服;唯一"心智"是记得 `import styles` 后用对象取类名。

## 2.3 Vite 集成细节:哈希在构建期,动态拼接会失效

```jsx
// vite 对 *.module.css 开箱即用;但哈希名是构建期生成的——
styles['button']         // ✓ 静态取键
styles[`btn-${size}`]    // ⚠ 拼出来的键未必存在,哈希后缀也不可控

// 条件类名用 clsx 组合,而不是动态取 key:
import clsx from 'clsx'
<button className={clsx(styles.button, size === 'lg' && styles.large)} />
```

```ts
// 想让 TS 校验类名拼写:上 typescript-plugin-css-modules,styles.xxx 打错即报错
{
  "compilerOptions": { "plugins": [{ "name": "typescript-plugin-css-modules" }] }
}
```

★ **关键认知**：哈希发生在**构建期**,`styles` 对象只是打包器生成的映射表——**运行时拼类名、外部全局选择器定位组件内部节点**都会落空(哈希名带随机后缀)。⚠ 需要"外部可覆盖组件样式",正解是开一个 `className` prop 透传(与第 24 篇 props 稳定引用配合),而不是指望外人写出你的哈希类名;跨组件共享的一段样式,则显式 `:global` 或用 `composes` 复用同一个基类。

# 3. CSS-in-JS

## 3.1 styled-components 等

```jsx
import styled from 'styled-components'

const Button = styled.button`
  background: #42b883;
  padding: 8px 16px;
  ${props => props.primary && 'font-weight: bold;'}
`
// <Button primary>按钮</Button>
```

★ **关键认知**：CSS-in-JS(如 styled-components/Emotion)把**样式写进 JS**——组件与样式同处一文件,还能**用 props 动态算样式**。⚠ 它的好处是"**样式随组件走、无类名冲突、天然支持动态**";代价是运行时开销(生成样式)、包体积、SSR 复杂度。

## 3.2 争议与趋势

★ **关键认知**：CSS-in-JS 近年有争议——运行时性能、与 React 并发/流式 SSR 的兼容问题。⚠ 趋势是"**零运行时**"的编译期方案(如 vanilla-extract、PandaCSS)兴起,把"CSS-in-JS 的体验"搬到"编译期生成静态 CSS";纯运行时 CSS-in-JS 不再是默认推荐。

## 3.3 困局的机制根源:样式产物在"运行时"确定

```jsx
// ⚠ 运行时 CSS-in-JS 的前提假设:"组件渲染时,在浏览器里往 <style> 注入样式"
// RSC/流式 SSR 把渲染搬到服务端——HTML 流出后组件不再"渲染",谁来注入样式表?
//   styled-components:SSR 要专门收集样式随 HTML 序列化,流式下尤其别扭
//   Emotion:同理需要 SSR 提取管线

// vanilla-extract:写法像 CSS-in-JS,产物是纯 CSS 文件
import { recipe } from '@vanilla-extract/recipes'
export const button = recipe({
  base: { padding: '8px 16px' },
  variants: { primary: { background: '#42b883' } },
})   // 构建期抽进 .css,运行时只剩类名字符串——零开销,RSC 天然兼容
```

★ **关键认知**：CSS-in-JS 衰落的根源不是"慢"这么简单,而是**它对运行时的依赖与 React 架构演进方向相反**——并发渲染、流式 SSR、RSC 都要求样式在"HTML 流出之前"就是**确定的静态产物**,运行时注入 `<style>` 的组件无法被静态收集。⚠ 结论是"运行时 CSS-in-JS"退位,不是"CSS 写进 TS 的体验"退位:vanilla-extract/Panda 保留体验、把产物搬回构建期;CSR 为主的老项目继续用 styled-components 没有不妥,别为迁移而迁移。

# 4. Tailwind(原子化)

## 4.1 工具类写法

```jsx
function Button() {
  return (
    <button className="bg-emerald-500 hover:bg-emerald-600 text-white px-4 py-2 rounded-lg">
      按钮
    </button>
  )
}
```

★ **关键认知**：Tailwind 把样式**拆成工具类堆在 className 上**(第 CSS 篇 20 篇原子化),不需要单独的 CSS 文件。⚠ 它在 React 生态**极受欢迎**——因 JSX 本就是"内聚单文件",与"样式就地写"的气质契合;也是现代组件库(如 shadcn/ui)的默认基础。

## 4.2 变体、任意值与"复用单元"

```jsx
<div className="hidden md:grid lg:grid-cols-3 gap-4">      {/* 响应式变体 */}
<p className="text-[13px] leading-[1.7]">方括号:预设外按需取值</p>

// 重复的 class 串不要手粘,提常量/组件化 + cn() 合并才是正解:
const card = 'rounded-xl border bg-white p-4 shadow-sm'
export const Card = ({ className = '', ...p }) => <div className={cn(card, className)} {...p} />
```

★ **关键认知**：Tailwind 复用的正确单元是**"组件 + cn()"**(clsx + tailwind-merge),`tailwind-merge` 专门解决"外部传 `p-2`、基类已有 `p-4`,谁赢"——按工具类语义去重,后者胜出。⚠ 反过来大量 `@apply` 把原子类烩回 .css 文件,等于放弃原子化的隔离与就地可改、只留缺点;`@apply` 越少越好。

## 4.3 v4 的变化与"原子化为什么省"

```css
/* Tailwind v4(2025):入口一行,配置进 CSS,不再要 tailwind.config.js */
@import "tailwindcss";
@theme { --color-brand: oklch(0.7 0.15 160); }   /* 设计 token = CSS 变量 */
```

```text
产物对比(为什么原子化体积可控):
  200 个组件 → 实际用过的工具类约 150 条
  最终 CSS ≈ 150 条极短规则,跨组件 100% 复用,样式成本不随组件数增长
  vs 每组件一份 .css/BEM 命名:规则数随组件线性膨胀,还难保证复用
```

★ **关键认知**：原子化省体积的机制是**类名即"复用键"**——`.p-4` 在全站只存在一条规则,谁用谁引用;扫描源码按需生成,保证"没用到就不进产物"。⚠ v4 已换 Oxide 引擎(OXC 解析,构建快一个量级)、oklch 色彩与 `@theme` 原生变量;旧教程的 `tailwind.config.js + PostCSS + content 数组`套路已过时——抄配置前先看版本。

# 5. 选型决策表

| 场景 | 推荐 |
|------|------|
| 传统 CSS 手感 + 隔离 | CSS Modules |
| 样式强依赖 props/主题动态 | CSS-in-JS(或编译期方案) |
| 快速开发 + 一致设计 token | Tailwind |
| 组件库/设计系统 | CSS Modules 或 Tailwind |
| RSC/Next App Router 项目 | Tailwind 或 CSS Modules(避运行时 CSS-in-JS) |
| 团队统一优先 | 选一个坚持用 |

★ **关键认知**：选型核心——**CSS Modules 最贴近传统、Tailwind 最"现代组件化"、CSS-in-JS 最"动态但重"**。⚠ 没有唯一正解,关键是**项目内统一**,别混成"半个 Module + 半个 Tailwind";现代 React 项目的主流趋势明显偏向 **Tailwind(或 CSS Modules)**。

# 6. 本篇小结与下一步

| 方案 | 隔离手段 | 动态样式 | 运行时代价 | RSC 友好 |
|------|---------|---------|-----------|---------|
| CSS Modules | 构建期哈希类名 | JS 里条件选类 | 无 | ✓ |
| styled-components | 运行时生成类名 | props 插值 | 有 | ✗ |
| vanilla-extract | 构建期抽静态 CSS | recipe variants | 无 | ✓ |
| Tailwind | 类名天然原子化 | 变体 + 任意值 | 无 | ✓ |

★ **关键认知**：React 样式三派——CSS Modules(哈希隔离、接近传统)、CSS-in-JS(样式进 JS、动态但重)、Tailwind(工具类、现代主流)。共同诉求是"组件化作用域";选型统一最重要。

★ 机制层再记一句:各派真正的分野是**"样式产物在构建期确定还是运行时确定"**——流式 SSR/RSC 时代,构建期方案(Tailwind、CSS Modules、vanilla-extract)全面占优;Vite 下 CSS Modules 零配置,但哈希在构建期生成,运行时拼类名、外部选择器穿透都会失效。

下一步第 23 篇:代码分割与懒加载——React.lazy/Suspense、路由懒加载,把首屏体积降下来。
