Vue 的立身之本是一句话：**声明式地描述视图与数据的依赖关系**，剩下"数据变了、界面跟着变"的脏活累活交给框架。本篇讲清 Vue 3 的定位、响应式的核心思想、组合式 API 与选项式 API 的分野，并给出一套可照抄执行的环境搭建命令，为整个系列打好地基。

# 1. Vue 是什么

## 1.1 一句话定位

Vue 是**渐进式（progressive）** JavaScript 框架。所谓渐进式，是指它的能力可以按需裁剪：只想在现有页面上加一点交互，用单个文件引入即可；想构建完整 SPA，可以再引入 Router、Pinia、构建工具链。

```html
<!-- 最轻量的用法：CDN 引入，页面里直接写，无需任何构建 -->
<script src="https://unpkg.com/vue@3"></script>
<div id="app">{{ message }}</div>
<script>
  const { createApp, ref } = Vue
  createApp({
    setup() {
      return { message: 'Hello Vue 3' }
    }
  }).mount('#app')
</script>
```

★ **关键认知**：Vue 的核心只做**一件事——响应式渲染**：把"状态(state) → 视图(view)"的映射声明出来，状态一变，框架自动算出需要更新的 DOM。你的主要工作从"命令式地拼 DOM"变成"描述数据长什么样"。这与 jQuery/原生 JS 手写 DOM 操作有本质区别。

## 1.2 声明式 vs 命令式

```javascript
// ===== 命令式（原生 JS / jQuery）：自己告诉浏览器每一步怎么做 =====
let count = 0
const btn = document.querySelector('#btn')
const text = document.querySelector('#text')
btn.addEventListener('click', () => {
  count++                       // 1. 改数据
  text.textContent = count      // 2. 手动把新值写回 DOM
})

// ===== 声明式（Vue）：只描述"视图依赖什么数据" =====
// <button @click="count++">{{ count }}</button>
// 数据变了，{{ count }} 自动更新，无需碰 DOM
```

★ **关键认知**：命令式要维护"数据 → DOM"这条同步链路，每多一处要更新就多一份漏改风险；声明式把"**更新**"这件易错的事封装进框架，你只需关心"**数据**"。用一张表对比：

| 维度 | 命令式（jQuery） | 声明式（Vue） |
|------|-----------------|--------------|
| 关注点 | 每一步 DOM 操作 | 数据与依赖关系 |
| 更新同步 | 手动，易漏 | 框架自动 |
| 心智负担 | 随 DOM 复杂度线性增长 | 相对恒定 |
| 定位 | 库（工具集合） | 框架（约定 + 运行时） |

## 1.3 响应式的核心：依赖追踪

```javascript
// Vue 3 响应式的本质：对象被 Proxy 代理，属性被读取时记录"谁在读"，
// 属性被修改时通知"读过它的人"重新执行
import { reactive, effect } from 'vue'

const state = reactive({ count: 0 })
effect(() => {
  // 这个函数执行时会读取 state.count，于是被记录为 count 的"依赖"
  console.log(state.count)
}) // 打印 0

state.count++ // 修改触发依赖，effect 重新执行 → 打印 1
```

★ **关键认知**：响应式系统 = **依赖收集（track）+ 依赖触发（trigger）**。读属性时把当前正在执行的 effect 记下来；写属性时把记下的 effect 全部重跑一遍。Vue 3 用 `Proxy` 拦截属性读写，`ref` 则把"单个值"包装成带 `.value` 的响应式引用——这套机制的完整原理在第 3、19 篇展开。

# 2. 组合式 API vs 选项式 API

## 2.1 同一件事，两种写法

```vue
<script>
// ===== 选项式（Options API）：按 data/methods/computed 分门别类 =====
export default {
  data() {
    return { count: 0 }
  },
  methods: {
    increment() { this.count++ }
  },
  computed: {
    double() { return this.count * 2 }
  }
}
</script>
```

```vue
<script setup lang="ts">
// ===== 组合式（Composition API）：按"逻辑"组织，可自由组合复用 =====
import { ref, computed } from 'vue'

const count = ref(0)
function increment() { count.value++ }
const double = computed(() => count.value * 2)
</script>
```

★ **关键认知**：选项式按**选项类别**（data、methods、computed…）组织代码，同一个功能的代码被拆散到不同区块；组合式按**逻辑关注点**组织，一项功能的数据、方法、计算属性写在一起——组件变大、逻辑变多时，组合式更容易把某一功能**整体抽走复用**（第 11 篇组合式函数）。⚠ 组合式和选项式不是互斥的，Vue 3 两者都支持，只是官方推荐新代码用组合式 + `<script setup>`。

## 2.2 组合式 API 的设计动机

```vue
<script setup lang="ts">
// 把"用户信息"这块逻辑整体封装成函数，多个组件共用
import { useUser } from './composables/useUser'
const { user, loading, fetchUser } = useUser()
// 逻辑一行引入，不用关心内部是 ref 还是方法
</script>
```

★ **关键认知**：选项式最大的痛点是**逻辑复用**——mixin 有命名冲突、来源不明；组合式把逻辑做成**普普通通的函数**，命名、来源、类型全都一清二楚。这是 Vue 3 最大的心智转变：**从"面向选项"转向"面向函数"**，也是本系列全程主线的由来。

# 3. 开发环境搭建

## 3.1 可照抄执行的命令清单

```powershell
# ===== 1. 切换 npm 到国内镜像（npmmirror），下载提速 =====
npm config set registry https://registry.npmmirror.com
npm config get registry        # 确认输出 https://registry.npmmirror.com/

# ===== 2. 用官方脚手架创建 Vue 3 + TS 项目（交互式选择） =====
npm create vue@latest my-app
# 交互提示中：TypeScript? Yes / Vue Router? No / Pinia? No / ESLint? 按需

# ===== 3. 安装依赖并启动开发服务器 =====
cd my-app
npm install
npm run dev                     # 输出本地地址，浏览器打开即见欢迎页

# ===== 4. 生产构建与本地预览 =====
npm run build                   # 产出 dist/
npm run preview                 # 本地预览构建产物
```

★ **关键认知**：`npm create vue@latest` 底层用的是 **Vite** 作为构建工具——它开发时用浏览器原生 ESM 做**即时按需编译**，启动快、热更新快。⚠ 别把 `npm create vue` 和老的 `vue-cli`（`@vue/cli`）搞混：后者是基于 webpack 的旧工具链，新项目一律用 Vite。

## 3.2 编辑器与插件

| 工具 | 用途 |
|------|------|
| VS Code + **Volar**（Vue - Official 插件） | 官方推荐，提供 `<script setup>` 的 TS 类型检查、模板补全、跳转 |
| 禁用 **Vetur** | 老插件，与 Volar 冲突，装 Volar 前务必卸载/禁用 |
| `vue-tsc`（项目内置） | 命令行做全项目的 Vue + TS 类型检查（`npm run type-check`） |

⚠ Volar 与 Vetur 二选一：同时启用会导致重复诊断。TypeScript 在 Vue 中的类型能力（`defineProps` 泛型等）由 Volar + `vue-tsc` 提供，不是 Vite 本身，细节见第 16 篇。

# 4. 最小可运行示例

## 4.1 一个计数器

```vue
<script setup lang="ts">
import { ref } from 'vue'

// ref 把 number 包装成响应式引用，读写都要 .value
const count = ref(0)

// 普通函数即可做事件处理器，模板里 @click="increment"
function increment() {
  count.value++          // ref 在 JS 里用 .value 访问
}
</script>

<template>
  <!-- 模板里自动解包，不需要 .value -->
  <button @click="increment">count is {{ count }}</button>
</template>
```

★ **关键认知**：这是 Vue 3 的最小闭环——`ref` 造数据、模板读数据、事件改数据、框架自动更新视图。三个规则先记住：① `<script setup>` 里声明的变量/函数，模板可直接用；② `ref` 在 **JS 里要 `.value`**，在 **模板里自动解包不用写**；③ 改 `ref.value` 才触发更新，改普通变量不会。

## 4.2 与 React 对照锚点

| 概念 | Vue 3 | React | 差异点 |
|------|-------|-------|--------|
| 状态 | `ref` / `reactive` | `useState` | Vue 改 `.value` 触发，React 用 setter 触发 |
| 依赖追踪 | 自动（读谁追谁） | 依赖数组人工声明 | Vue 无需 `useMemo` 那种显式依赖 |
| 计算属性 | `computed` | `useMemo` | Vue 自动缓存且追踪依赖 |
| 副作用 | `watch` / `watchEffect` | `useEffect` | Vue 自动追踪，React 要写依赖数组 |
| 组件渲染 | 模板（编译优化） | JSX（运行时） | Vue 模板可编译期优化，见第 21 篇 |

★ **关键认知**：如果你从 React 转来，最大的思维切换是——**React 靠 props/deps 数组显式宣告依赖，Vue 靠运行时 Proxy 隐式追踪**。Vue 不需要"依赖数组没写对导致闭包旧值"这类心智负担；代价是响应式数据必须是 `ref/reactive` 造的，普通对象不会被追踪（这是新手最常见的坑，第 3 篇详解）。

# 5. 学什么、怎么学

## 5.1 Vue 的收益与代价

| 收益 | 代价 |
|------|------|
| 声明式渲染，告别手动 DOM 同步 | 响应式规则要靠理解（何时 `.value`、何时深拷贝） |
| 依赖自动追踪，副作用不易漏 | 框架内有自己的更新时机（nextTick）要熟悉 |
| 逻辑可组合复用（composables） | 深水区的响应式/渲染/编译原理有门槛 |
| 模板编译期优化，性能下限高 | 模板 DSL 有其限制，复杂逻辑要落回 JS |

## 5.2 本系列阅读路径

| 阶段 | 篇目 | 目标 |
|------|------|------|
| 基础 | 2 ~ 6 | 应用创建、模板语法、响应式、计算/侦听、渲染 |
| 组件 | 7 ~ 12 | 组件通信、插槽、生命周期、组合式函数、注入 |
| 工程化 | 13 ~ 18 | SFC 构建、样式、动画、TS、Pinia、Router |
| 原理与高级 | 19 ~ 23 | 响应式内核、vdom/diff、编译、性能、面试 |

★ **关键认知**：Vue 的学习曲线"**入门平缓、原理陡峭**"——`ref` + 模板一天就能写出能跑的页面，但真正用好需要过三关：**响应式规则**（什么东西会被追踪）、**组件边界**（props/emit 的单向数据流）、**更新时机**（什么时候 DOM 真的变了，第 6、10 篇）。建议严格按上表顺序推进，**每篇示例亲手敲一遍**：响应式的直觉只能从"为什么这里没更新"的排查里练出来。