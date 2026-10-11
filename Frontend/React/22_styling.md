React 不管样式,于是样式方案百花齐放:**CSS Modules**(作用域隔离)、**CSS-in-JS**(把样式写进 JS)、**原子化 Tailwind**(工具类)。本篇逐一讲清它们的写法、优劣与选型,回应"组件化时代 CSS 怎么管"这个经典问题。

# 1. 全局 CSS 的困境

## 1.1 组件化需要样式隔离

```css
/* 全局 title 会互相覆盖、名字撞车 */
.title { color: red; }
```

★ **关键认知**：常规 CSS 是**全局的**,组件化项目里"每个组件一份样式"会互相污染(撞类名)。⚠ React 的样式方案,核心诉求都是**"给样式加作用域/隔离"**——与 Vue 的 scoped 面对同一问题(Vue 第 14 篇、CSS 第 19 篇)。

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

# 5. 选型决策表

| 场景 | 推荐 |
|------|------|
| 传统 CSS 手感 + 隔离 | CSS Modules |
| 样式强依赖 props/主题动态 | CSS-in-JS(或编译期方案) |
| 快速开发 + 一致设计 token | Tailwind |
| 组件库/设计系统 | CSS Modules 或 Tailwind |
| 团队统一优先 | 选一个坚持用 |

★ **关键认知**：选型核心——**CSS Modules 最贴近传统、Tailwind 最"现代组件化"、CSS-in-JS 最"动态但重"**。⚠ 没有唯一正解,关键是**项目内统一**,别混成"半个 Module + 半个 Tailwind";现代 React 项目的主流趋势明显偏向 **Tailwind(或 CSS Modules)**。

# 6. 本篇小结与下一步

★ **关键认知**：React 样式三派——CSS Modules(哈希隔离、接近传统)、CSS-in-JS(样式进 JS、动态但重)、Tailwind(工具类、现代主流)。共同诉求是"组件化作用域";选型统一最重要。

下一步第 23 篇:代码分割与懒加载——React.lazy/Suspense、路由懒加载,把首屏体积降下来。