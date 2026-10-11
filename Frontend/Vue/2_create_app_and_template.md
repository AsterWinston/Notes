模板是把数据"放映"到界面上的第一道门。本篇讲清 `createApp` 如何把组件挂载成真实应用、模板里的四种插值与文本渲染，以及三个最核心的指令 `v-bind` / `v-text` / `v-html` 与 `{{ }}` 的取舍——这是后续所有响应式渲染的语法地基。

# 1. 创建应用：createApp 与挂载

## 1.1 从根组件到真实 DOM

```typescript
// main.ts —— 每个 Vue 应用的入口
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)   // 1. 用根组件创建一个"应用实例"
app.mount('#app')            // 2. 挂载到 index.html 里 id 为 app 的元素
```

★ **关键认知**：`createApp(App)` 不会立刻渲染，它返回一个**应用实例（app）**——这是一个"尚未挂载"的配置容器。`.mount('#app')` 才真正把根组件渲染进 `#app` 元素。⚠ 每个页面的 Vue 应用通常**只调用一次 `createApp`**；多个互不相干的应用才各 `createApp` 一次，它们之间不共享状态。

## 1.2 应用实例能做什么

```typescript
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// 使用插件（如 Pinia、Router），必须在 mount 之前
app.use(router)

// 全局注册组件（第 7 篇）与全局注入（第 12 篇）
app.component('MyCard', MyCard)
app.provide('theme', 'dark')

// 全局配置与全局方法
app.config.errorHandler = (err) => console.error(err)

app.mount('#app')
```

★ **关键认知**：`app` 是**全局**层面的入口，`use` 插件、`provide` 注入、`component` 注册都在这里做，且绝大多数**必须在 `mount` 之前**完成——mount 之后应用已定型，再 `use` 插件多数不生效。

## 1.3 根组件里写了什么

```vue
<script setup lang="ts">
// 根组件也是普通组件：下面两行放 App.vue 即可
</script>

<template>
  <h1>Hello</h1>
</template>
```

★ **关键认知**：根组件和普通组件**没有任何特殊语法**——`App.vue` 就是一个普普通通的单文件组件（SFC，第 13 篇详解）。`mount('#app')` 的 `#app` 是 `index.html` 里预先写好的一个空元素，Vue 会把模板内容塞进去。

# 2. 模板插值

## 2.1 文本插值 {{ }}

```vue
<script setup lang="ts">
import { ref } from 'vue'
const msg = ref('Hello')
const num = 1
</script>

<template>
  <!-- 插值表达式：可以是变量、表达式，甚至是三元/方法调用 -->
  <p>{{ msg }}</p>              <!-- Hello（ref 自动解包） -->
  <p>{{ num + 1 }}</p>          <!-- 2 -->
  <p>{{ msg.toUpperCase() }}</p> <!-- HELLO -->

  <!-- ⚠ 赋值/语句插不进去：下面两种都会报错 -->
  <!-- <p>{{ num = 2 }}</p> -->
  <!-- <p>{{ const x = 1 }}</p> -->
</template>
```

★ **关键认知**：`{{ }}` 里是**单条 JS 表达式**——能求值的都行（运算、三元、函数调用、`??`），**语句不行**（`=`、`if`、`let` 都不算表达式）。⚠ 插值里写复杂函数调用会破坏模板可读性，且函数每次渲染都会执行，复杂表达式应提成 `computed`（第 4 篇）。

## 2.2 v-text 与 v-html

```vue
<script setup lang="ts">
import { ref } from 'vue'
const raw = ref('<b>加粗</b>')
</script>

<template>
  <!-- v-text：等价于 {{ raw }}，覆盖元素整个文本 -->
  <p v-text="raw"></p>      <!-- 显示字面量 <b>加粗</b> -->

  <!-- v-html：把字符串当 HTML 解析，危险 -->
  <p v-html="raw"></p>      <!-- 渲染出真·加粗字 -->
</template>
```

★ **关键认知**：`{{ }}` 和 `v-text` 都会把内容**当纯文本**转义输出，`<b>` 会原样显示；`v-html` 才会**解析 HTML 标签**。⚠ `v-html` 是 XSS 的经典入口——把用户输入未经消毒直接 `v-html`，等于允许别人往页面里塞 `<script>` 或钓鱼链接。**除非内容是你完全可信的，否则禁用 `v-html`**。

# 3. 属性绑定 v-bind

## 3.1 绑定属性与简写

```vue
<script setup lang="ts">
import { ref } from 'vue'
const src = ref('https://example.com/logo.png')
const disabled = ref(false)
</script>

<template>
  <!-- 完整写法 v-bind:，简写是冒号 -->
  <img v-bind:src="src" alt="logo">
  <img :src="src" alt="logo">        <!-- 简写，最常用 -->

  <!-- 布尔属性：值 falsy 时属性直接被移除 -->
  <button :disabled="disabled">点击</button>

  <!-- 动态属性名：方括号里写表达式 -->
  <div :[attrName]="value">动态属性</div>
</template>
```

★ **关键认知**：`v-bind` 把属性的值从"写死"变成"从数据来"。⚠ 响应式数据放 HTML 属性上时，`src`、`href` 这类要用 `:` 绑定才跟着变；不绑就永远是首次渲染的字符串。布尔属性（`disabled`、`checked`）绑到 falsy 值时，Vue 会**直接移除该属性**而非写成 `disabled="false"`——这是与 React 不同的一点。

## 3.2 一次性绑定与对象批量绑定

```vue
<script setup lang="ts">
import { ref } from 'vue'
const theme = ref('dark')
const attrs = { id: 'box', class: 'card', tabindex: 1 }
</script>

<template>
  <!-- 全体批量绑定：一个对象搞定多个属性 -->
  <div v-bind="attrs">批量</div>

  <!-- 惯用技巧：对象字面量按条件拼接 class -->
  <div :class="{ active: isActive, 'text-red': isError }">条件类名</div>
  <div :style="{ color: textColor, fontSize: size + 'px' }">行内样式</div>
</template>
```

★ **关键认知**：`:class` / `:style` 是 `v-bind` 的强化版本，支持对象/数组语法——布尔值为 true 的键才生效。这类**条件样式**写法在第 14 篇会深入（scoped、CSS Modules、`:deep()` 等）。这里先记住：**class 绑定的首选是对象字面量**，可读性最好。

# 4. 指令初识

## 4.1 指令是什么

```vue
<template>
  <!-- v- 开头就是指令：帮元素"做点"绑定、条件、循环之类的事 -->
  <p v-if="show">条件渲染</p>
  <p v-for="n in 3" :key="n">第 {{ n }} 行</p>
  <input v-model="input">
</template>
```

★ **关键认知**：指令（directive）是 Vue 模板的**第二套 DSL**（第一套是 `{{ }}` 插值）。它形如 `v-指令名[:参数][.修饰符]="表达式"`，本质是"给元素附加一段带状态的渲染逻辑"。常见指令家族：

| 指令 | 作用 | 所在篇 |
|------|------|--------|
| `v-bind` / `:` | 动态绑定属性 | 本篇 |
| `v-on` / `@` | 绑定事件 | 第 6 篇 |
| `v-model` | 双向绑定表单 | 第 6 篇 |
| `v-if` / `v-show` | 条件渲染 | 第 5 篇 |
| `v-for` | 列表渲染 | 第 5 篇 |
| `v-text` / `v-html` | 文本 / HTML 渲染 | 本篇 |

## 4.2 模板背后的思考：模板 vs JSX

★ **关键认知**：Vue 选模板而非 JSX，最大收益是**可编译性**——模板是受限的、静态结构清晰的 DSL，编译器能据此做**静态提升、PatchFlags 打标、Block tree** 这些优化（第 21 篇）。JSX（React）是普通 JS，灵活但编译器无从下手。代价是模板表达能力有边界（不能写任意 JS 语句），复杂渲染逻辑要退回渲染函数或 computed。这就是 Vue 与 React 一个深层的设计分歧。

# 5. 本篇小结与下一步

★ **关键认知**：本篇四个地基——`createApp + mount` 组成应用入口、`{{ }}` 负责文本、`v-bind` 负责属性、指令负责行为。它们共同服务一个原则：**模板只描述"数据长什么样"，绝不手写 DOM 同步**。

下一步进入第 3 篇，正式揭开响应式的面纱——`ref` / `reactive` 到底把数据变成了什么、`.value` 抓得住的规则。