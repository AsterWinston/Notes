# React 学习笔记

React 是当前最流行的 UI 库,核心思想是**组件化 + 声明式 + 单向数据流**——把 UI 写成"状态 → 视图"的纯函数,数据一变,框架用 diff 算出最小更新。本模块共 33 篇,从 JSX 与组件讲到 Hooks 全家桶、工程化生态(路由/状态/数据请求/样式),再到并发特性、SSR/RSC 与 Fiber/源码级原理。主线为**函数组件 + Hooks**。受众定位:已熟悉 JavaScript 与现代前端(最好读过本仓库 Vue 系列,二者互相印证),想系统掌握 React 18/19 的开发者。

# 1. 目录

## 1.1 阶段一：基础（1-7 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [1_introduction.md](1_introduction.md) | 简介与核心思想 | 组件化/声明式/单向数据流、JSX 由来、函数组件 |
| [2_jsx_and_element.md](2_jsx_and_element.md) | JSX 与元素 | JSX 语法、元素 vs 组件、与 HTML/Vue 模板对比 |
| [3_component_and_props.md](3_component_and_props.md) | 组件与 props | 函数组件、props、组合 vs 继承、children |
| [4_useState.md](4_useState.md) | 状态 useState | 不可变更新、批处理、闭包陷阱 |
| [5_event_and_list.md](5_event_and_list.md) | 事件处理与列表 key | 合成事件、事件委托、列表渲染与 key |
| [6_conditional_rendering.md](6_conditional_rendering.md) | 条件渲染与控制流 | &&/三元/渲染守卫、循环控制流 |
| [7_form_and_controlled.md](7_form_and_controlled.md) | 表单与受控组件 | 受控/非受控、值绑定、单选复选 |

## 1.2 阶段二：Hooks 全家桶（8-15 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [8_useEffect.md](8_useEffect.md) | useEffect 与副作用 | 依赖数组、清理、心智模型 |
| [9_usememo_usecallback.md](9_usememo_usecallback.md) | useMemo 与 useCallback | 记忆化、何时用、性能 |
| [10_useref_forwardref.md](10_useref_forwardref.md) | useRef 与命令式 | ref/DOM/保存可变值、forwardRef/useImperativeHandle |
| [11_usecontext.md](11_usecontext.md) | useContext 与状态提升 | 跨层传递、组合 context、与 prop drilling |
| [12_usereducer.md](12_usereducer.md) | useReducer 与复杂状态 | reducer/dispatch/不可变模式 |
| [13_custom_hook.md](13_custom_hook.md) | 自定义 Hook | 逻辑复用、命名、与 Vue composable 对照 |
| [14_other_hooks.md](14_other_hooks.md) | 其余内建 Hooks | useId/useLayoutEffect/useTransition/useDeferredValue 等 |
| [15_hooks_rules.md](15_hooks_rules.md) | Hooks 规则与心智模型 | 为什么不能条件调用、依赖原理 |

## 1.3 阶段三：工程化（16-23 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [16_hoc_render_props.md](16_hoc_render_props.md) | 组合模式 vs HOC | HOC、render props、组合替代 |
| [17_react_router.md](17_react_router.md) | React Router | 路由配置、嵌套、守卫、懒加载、loader/action |
| [18_redux_toolkit.md](18_redux_toolkit.md) | 状态管理 Redux Toolkit | slice/store、dispatch、与 Pinia 对照 |
| [19_state_libs.md](19_state_libs.md) | 其他状态库选型 | Zustand/Jotai/Recoil/MobX |
| [20_react_query.md](20_react_query.md) | 数据请求 React Query | useQuery/缓存/无限滚动/乐观更新 |
| [21_react_hook_form.md](21_react_hook_form.md) | 表单 React Hook Form | useForm、校验 Zod、非受控 |
| [22_styling.md](22_styling.md) | 样式方案 | CSS Modules/CSS-in-JS/Tailwind |
| [23_code_splitting.md](23_code_splitting.md) | 代码分割与懒加载 | React.lazy/Suspense、路由懒加载 |

## 1.4 阶段四：高级与架构（24-29 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [24_performance.md](24_performance.md) | 性能优化 | React.memo、不可变、虚拟化、Profiler |
| [25_error_boundary_concurrent.md](25_error_boundary_concurrent.md) | 错误边界与并发特性 | ErrorBoundary、Suspense、useTransition |
| [26_typescript.md](26_typescript.md) | TypeScript 集成 | 泛型组件、事件类型、Hooks 类型 |
| [27_ssr_rsc_next.md](27_ssr_rsc_next.md) | SSR/RSC/Next.js | 服务端渲染、Server Components、流式渲染 |
| [28_testing.md](28_testing.md) | 测试 | RTL/Jest、组件/Hooks 测试、e2e |
| [29_advanced_streaming.md](29_advanced_streaming.md) | 服务端与流式渲染进阶 | 流式、Suspense for SSR、缓存 |

## 1.5 阶段五：原理与面试（30-33 篇）

| 篇目 | 主题 | 一句话简介 |
|------|------|-----------|
| [30_fiber.md](30_fiber.md) | Fiber 架构与协调 | 双缓存树、lanes、时间切片、可中断渲染 |
| [31_diff.md](31_diff.md) | Diff 算法 | 单节点/列表 diff、key |
| [32_hooks_impl.md](32_hooks_impl.md) | Hooks 实现原理 | 链表、闭包、依赖比对 |
| [33_interview_and_summary.md](33_interview_and_summary.md) | 面试要点与总结 | 高频面试题、知识地图、进阶方向 |

# 2. 推荐阅读路径

```
Vue 熟练者         前端老手          需工程化           冲刺深入
     │                │                │                    │
     ▼                ▼                ▼                    ▼
  4 → 8 → 13      1 → 4 → 8      17 → 18 → 20        24 → 30 → 31
  17 → 24 → 30    11 → 12 → 15   21 → 22 → 23        32 → 33
```

- **速成路线（约 1 周）**：1 → 2 → 3 → 4 → 5 → 6 → 8 → 11 → 17 → 20,覆盖日常开发 90% 场景。
- **完整路线（约 3~5 周）**：按序号 1~33 顺读,每篇代码亲手敲一遍。
- **面试路线**：4 → 8 → 11 → 13 → 15 → 24 → 30 → 31 → 32 → 33。
- **对照建议**：读过 Vue 系列者,关注"依赖追踪 vs 依赖数组""模板编译 vs 运行时""composables vs Hooks"三组对照。

# 3. 阅读约定

| 标记 | 含义 |
|------|------|
| `★ **关键认知**` | 该节最核心的机制/结论，优先记住 |
| `⚠` | 常见坑点或易误解处 |
| 代码块 | 均带逐行中文注释，可直接运行（标注"故意报错"的反例除外） |
| 命令清单 | 可照抄执行；npm 源均使用 npmmirror 国内镜像 |
| 组件示例 | 统一函数组件 + Hooks 写法 |