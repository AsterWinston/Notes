# CSS 学习笔记

CSS 是网页的皮肤——用选择器命中元素、用属性描述样式、用层叠规则解决冲突，最终把 HTML 的结构"画"成美观、响应式的界面。本模块共 22 篇，从选择器与盒模型讲到布局系统（Flex/Grid/响应式）、动画与进阶特效，再到预处理器、Tailwind 与工程化，覆盖现代前端 CSS 全链路。受众定位：已熟悉 HTML、想系统掌握 CSS 布局与现代工程化手段的开发者。

# 1. 目录

## 1.1 阶段一：基础（1-6 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_introduction_and_cascade.md](1_introduction_and_cascade.md) | 引入方式与层叠 | 样式三种引入、层叠规则、继承、来源与优先级 |
| [2_selectors_specificity.md](2_selectors_specificity.md) | 选择器与特异性 | 基础/组合/属性/伪类/伪元素、specificity 计算 |
| [3_box_model.md](3_box_model.md) | 盒模型与尺寸 | content/padding/border/margin、box-sizing、宽高计算 |
| [4_text_and_font.md](4_text_and_font.md) | 文本与字体 | font 体系、text 排版、line-height、@font-face |
| [5_color_and_background.md](5_color_and_background.md) | 颜色与背景 | color 表示、background、渐变、多背景 |
| [6_position_stacking.md](6_position_stacking.md) | 定位与层叠上下文 | position/z-index/层叠上下文/containing block |

## 1.2 阶段二：布局（7-11 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [7_display_and_flow.md](7_display_and_flow.md) | display 与文档流 | display 取值、文档流、float、清除浮动 |
| [8_flexbox.md](8_flexbox.md) | Flexbox | 主轴/交叉轴、对齐 justify/align、弹性 grow/shrink |
| [9_grid.md](9_grid.md) | Grid | 网格线/区域、命名轨道、align/justify、auto 布局 |
| [10_responsive.md](10_responsive.md) | 响应式与媒体查询 | viewport、@media、clamp/min/max、移动优先 |
| [11_layout_practice.md](11_layout_practice.md) | 布局实战与取舍 | 常见布局模式、flex vs grid 选择、居中合集 |

## 1.3 阶段三：进阶（12-16 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [12_transition_animation.md](12_transition_animation.md) | 过渡与动画 | transition/animation/keyframes、动画性能 |
| [13_transform.md](13_transform.md) | 变形 | translate/rotate/scale、transform-origin、3D |
| [14_vars_and_functions.md](14_vars_and_functions.md) | CSS 变量与函数 | 自定义属性、calc/min/max/clamp/var |
| [15_bfc_and_pitfalls.md](15_bfc_and_pitfalls.md) | BFC 与常见坑 | 格式化上下文、外边距折叠、包含块、高频坑点 |
| [16_visual_effects.md](16_visual_effects.md) | 视觉效果 | shadow、filter、clip-path、mask、混合模式 |

## 1.4 阶段四：工程化（17-22 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [17_sass.md](17_sass.md) | 预处理器 Sass/SCSS | 嵌套、变量、mixin、函数、模块化 |
| [18_postcss.md](18_postcss.md) | PostCSS | autoprefixer、插件生态、CSS polyfill |
| [19_architecture_methodology.md](19_architecture_methodology.md) | CSS 架构方法论 | BEM、OOCSS、SMACSS、CSS Modules 选型 |
| [20_tailwind.md](20_tailwind.md) | Tailwind 与原子化 CSS | utility-first、配置、JIT、与语义化取舍 |
| [21_performance.md](21_performance.md) | 性能优化 | 选择器性能、重绘回流、will-change、contain |
| [22_interview_and_summary.md](22_interview_and_summary.md) | 面试要点与总结 | 高频面试题、知识地图、进阶方向 |

# 2. 推荐阅读路径

```
零基础入门          有基础补漏         冲刺面试/深入
     │                  │                  │
     ▼                  ▼                  ▼
  1 → 2 → 3         3 → 6 → 8         6 → 8 → 9
  4 → 5 → 6         9 → 10 → 15       12 → 15 → 21
  7 → 8 → 9                            22
```

- **速成路线（约 1 周）**：1 → 2 → 3 → 4 → 6 → 8 → 9 → 10，覆盖日常布局 90% 场景。
- **完整路线**：按序号 1~22 顺读，每篇示例亲手敲一遍。
- **面试路线**：2 → 3 → 6 → 8 → 9 → 12 → 15 → 21 → 22。

# 3. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 该节最核心的机制/结论，优先记住 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | CSS 均带中文注释，可直接放到样式表或 `<style>` 里运行 |
| 命令清单 | 可照抄执行；npm 源均使用 npmmirror 国内镜像 |