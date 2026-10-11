# Vue 3 学习笔记

Vue 3 是当前主流的渐进式前端框架，核心思想是把"数据变了、界面跟着变"这件事从手动 DOM 操作里彻底解放出来——开发者只描述"视图依赖哪些数据"，框架负责依赖追踪与更新。本模块共 23 篇，从响应式思想讲到组件体系、工程化实战，再到渲染与编译原理，主线为**组合式 API + `<script setup>`**，示例统一用 TypeScript 编写、可直接运行。受众定位：已熟悉 HTML/CSS/JavaScript、想系统掌握 Vue 3 的开发者；SSR/Nuxt、单元测试等专题不在本系列，仅在阅读路径中指明进阶方向。

# 1. 目录

## 1.1 阶段一：基础（1-6 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_introduction.md](1_introduction.md) | 简介与响应式思想 | 声明式 + 响应式的核心思想、组合式 vs 选项式、可照抄的 Vite + Volar 环境搭建命令 |
| [2_create_app_and_template.md](2_create_app_and_template.md) | 创建应用与模板语法 | createApp / 挂载流程、模板插值、指令初识 v-bind / v-text / v-html、模板 vs JSX |
| [3_reactivity_basics.md](3_reactivity_basics.md) | 响应式基础 | ref / reactive / shallow 家族、`.value` 与自动解包规则、Proxy 原理初探 |
| [4_computed_and_watch.md](4_computed_and_watch.md) | 计算属性与侦听器 | computed 缓存机制、watch / watchEffect、flush 时机、三者取舍矩阵 |
| [5_conditional_and_list_rendering.md](5_conditional_and_list_rendering.md) | 条件与列表渲染 | v-if vs v-show 抉择、v-for、key 与 diff 复用、数组响应式方法、v-for + v-if 陷阱 |
| [6_event_and_form.md](6_event_and_form.md) | 事件处理与表单绑定 | 事件修饰符全集、v-model 语法糖本质、修饰符 .lazy/.number/.trim、组件 v-model 预告 |

## 1.2 阶段二：组件（7-12 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [7_component_basics.md](7_component_basics.md) | 组件基础与注册 | 组件定义、全局 vs 局部注册、props 校验、emits 声明、单向数据流铁律 |
| [8_component_communication.md](8_component_communication.md) | 组件通信 | props down / emit up、defineModel（组件 v-model）、$attrs、defineExpose 子组件引用 |
| [9_slots.md](9_slots.md) | 插槽 | 默认/具名/作用域插槽、动态插槽名、插槽与渲染函数的关系 |
| [10_lifecycle.md](10_lifecycle.md) | 生命周期 | 组合式生命周期钩子、setup 执行时机、onMounted/onBeforeUnmount、与选项式对照表 |
| [11_composables.md](11_composables.md) | 组合式函数 | useXxx 命名规范、逻辑复用封装、异步状态处理、与 mixin / React Hooks 对比 |
| [12_provide_inject.md](12_provide_inject.md) | 依赖注入 | provide/inject 跨层级传值、响应式注入、Symbol 键、app.provide 全局注入 |

## 1.3 阶段三：工程化（13-18 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [13_sfc_and_build.md](13_sfc_and_build.md) | 单文件组件与构建 | SFC 结构、`<script setup>`、Vite 工作原理、dev vs build 产物差异 |
| [14_styling.md](14_styling.md) | 样式方案 | scoped 属性选择器原理、CSS Modules、:deep()/:slotted()/:global()、动态样式 |
| [15_transition_and_animation.md](15_transition_and_animation.md) | 过渡与动画 | Transition / TransitionGroup、CSS 过渡、JS 钩子、列表动画 |
| [16_typescript_integration.md](16_typescript_integration.md) | TypeScript 集成 | defineProps/defineEmits 类型、泛型组件、ref 泛型、vue-tsc、类型推断陷阱 |
| [17_state_management_pinia.md](17_state_management_pinia.md) | 状态管理 Pinia | option/setup store 两种写法、state/getters/actions、组件外使用、与 Vuex 对比 |
| [18_vue_router.md](18_vue_router.md) | 路由 | 路由配置、动态/嵌套路由、导航守卫、路由懒加载、组合式路由 API |

## 1.4 阶段四：原理与高级（19-23 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [19_reactivity_internals.md](19_reactivity_internals.md) | 响应式系统原理 | Proxy/Reflect、依赖收集 track、依赖触发 trigger、effect、scheduler、收集陷阱 |
| [20_vdom_and_diff.md](20_vdom_and_diff.md) | 虚拟 DOM 与 diff | vdom 结构、patch 算法、双端 diff、key 与复用、为什么需要 vdom |
| [21_compiler_render.md](21_compiler_render.md) | 模板编译与渲染原理 | 模板 → render 函数、编译器三阶段、静态提升、PatchFlags、Block tree |
| [22_performance.md](22_performance.md) | 性能优化 | 响应式副作用控制、v-memo/v-once、组件懒加载、打包优化与性能分析 |
| [23_interview_and_summary.md](23_interview_and_summary.md) | 面试要点与总结 | 高频面试题、响应式/渲染/编译知识地图、进阶方向 |

# 2. 推荐阅读路径

```
零基础前端         有 JS 基础          需要工程化           冲刺深入/面试
     │                │                    │                    │
     ▼                ▼                    ▼                    ▼
  1 → 2 → 3      1 → 3 → 5 ─┐        13 → 14 → 16        19 → 20 → 21
  4 → 5 → 6      6 → 7 → 8 ─┤        17 → 18              22 → 23
                            ├→ 9 → 10 → 11
                            │   12
```

- **速成路线（约 1 周）**：1 → 2 → 3 → 4 → 5 → 6 → 7 → 17 → 18，覆盖日常开发 90% 场景。
- **完整路线（约 3~4 周）**：按序号 1~23 顺读，每篇代码亲手敲一遍。
- **面试路线**：3 → 4 → 8 → 11 → 17 → 19 → 20 → 21 → 23。
- **前置建议**：先读 `Frontend/TypeScript` 第 1~6 篇（类型基础），再读本系列第 16 篇会顺畅得多。

# 3. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 该节最核心的机制/结论，优先记住 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均带逐行中文注释，可直接运行（标注"故意报错"的反例除外） |
| 命令清单 | 可照抄执行；npm 源均使用 npmmirror 国内镜像 |
| 组件示例 | 统一 `<script setup lang="ts">` 写法，个别反例单独标注 |