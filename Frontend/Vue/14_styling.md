组件化要求"样式也不互相污染"，但 CSS 天生是全局的——类名一旦重复就打架。Vue 用 `scoped` 在编译期给选择器打"隔离标签"，用 CSS Modules 做真正的类名哈希，用 `:deep()` 精细破例。本篇讲清 scoped 的隔离原理、几个深度选择器的精确语义，以及动态 class/style 与 CSS 变量的工程化搭配。

# 1. scoped 的隔离原理

## 1.1 加了 scoped 发生了什么

```vue
<style scoped>
.title { color: red; }
</style>
```

★ **关键认知**：`scoped` 让样式**只作用于本组件**，原理是编译期**双向改造**——给本组件的每个元素加一个唯一的**属性标记**（如 `data-v-xxxxxx`），同时给每条 CSS 选择器**追加属性选择器** `[data-v-xxxxxx]`。源码里 `.title` 编译后变成 `.title[data-v-xxxxxx]`，从而只命中带这个标记的本组件元素。

## 1.2 直观对比

```
源码： .title { color: red; }
编译： .title[data-v-f3f3eg9] { color: red; }

模板 <h1 class="title"> 编译成 <h1 class="title" data-v-f3f3eg9>
```

★ **关键认知**：scoped 隔离的**载体是"组件实例标签"**，不是真正的 CSS 作用域。因此它有边界：**子组件的根元素**会同时带上父组件的 scoped 标记（便于父组件给子组件根元素加样式），但**子组件内部更深层**的元素不受影响。⚠ 理解这一点，才能明白 `:deep()` 为什么存在。

# 2. 深度选择器 :deep() 等

## 2.1 :deep() 穿透到子组件内部

```vue
<!-- 父组件想给子组件内部元素加样式 -->
<style scoped>
/* ✗ 普通写法：子组件深层元素不带父的 data-v，命中不了 */
.child .inner { color: red; }

/* ✓ :deep()：编译后父标记只挂在外层，inner 部分不被加属性选择器 */
:deep(.inner) { color: red; }
</style>
```

★ **关键认知**：`<style scoped>` 下，子组件**内部**元素没有父组件的 `data-v`，普通选择器够不着。`:deep(xxx)` 告诉编译器"这段选择器的**后半部分不要加 scoped 标记**"，从而穿透到子组件内部。⚠ 这与 Vue 2 的 `>>>` 或 `/deep/` 等价，Vue 3 统一用 `::v-deep` 的新写法 `:deep()`。

## 2.2 :slotted() 与 :global()

```vue
<style scoped>
/* :slotted：命中"插槽传进来的内容"（它们在父组件作用域，本身不带本组件 data-v） */
:slotted(.slot-item) { color: blue; }

/* :global：这条规则不做 scoped 限制，回归全局 */
:global(.global-class) { color: green; }
</style>
```

★ **关键认知**：三个深度选择器各管一段——**`:deep()` 穿透子组件内部**、**`:slotted()` 针对插槽内容**、**`:global()` 直接声明全局规则**。⚠ `:slotted()` 之所以特殊，是因为插槽内容的 DOM 编译在父组件，天然不带子组件标记；`:global()` 则是少数"我就想全局"需求的正规出口，比去掉 `scoped` 更精细。

# 3. CSS Modules

## 3.1 真正的类名哈希

```vue
<template>
  <!-- $style 是 CSS Modules 导出的类名映射对象 -->
  <h1 :class="$style.title">标题</h1>
</template>

<style module>
.title { color: red; }
</style>
```

★ **关键认知**：`<style module>` 启用 **CSS Modules**——类名被编译成**带哈希的唯一名**（如 `title` → `_title_1a2b3`），并通过 `$style` 对象暴露给模板。与 `scoped` 的"属性选择器"不同，CSS Modules 是**真正的类名级隔离**，甚至能脱离组件在 `useCssModule()` 里拿到。⚠ 它适合"类名必须稳定可编程、或要在 JS 里操作"的场景，日常小项目 scoped 更省心。

## 3.2 scoped vs CSS Modules

| 维度 | scoped | CSS Modules |
|------|--------|-------------|
| 隔离方式 | 属性选择器 `[data-v]` | 类名哈希 |
| 穿透 | 需 :deep() | 类名唯一，天然隔离 |
| JS 访问 | 不提供 | `$style` / `useCssModule()` |
| 上手 | 简单 | 需习惯绑定写法 |
| 适用 | 通用 | 组件库、需在 JS 用类名 |

# 4. 动态 class 与 style

## 4.1 三种 class 绑定

```vue
<script setup lang="ts">
import { ref } from 'vue'
const isActive = ref(true)
const error = ref(false)
</script>

<template>
  <!-- 对象字面量：true 的键才加入 -->
  <div :class="{ active: isActive, 'text-error': error }">对象</div>

  <!-- 数组：多个条件/静态类合并 -->
  <div :class="['base', isActive && 'active']">数组</div>

  <!-- 混合：数组里放对象 -->
  <div :class="['base', { active: isActive }]">混合</div>
</template>
```

★ **关键认知**：`:class` 支持**对象 / 数组 / 它们的混合**，最终被扁平化为一个字符串。对象形式最可读，数组形式适合拼静态类 + 条件类。⚠ 静态类（`class="base"`）和动态类（`:class="..."`）写在同一元素会**合并**，不是覆盖——这是 Vue 做的贴心处理。

## 4.2 :style 与 CSS 变量

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
const width = ref(100)
const color = ref('#42b883')

// CSS 变量的响应式用法：让变量值跟着数据走
const themeVars = computed(() => ({ '--primary': color.value }))
</script>

<template>
  <!-- 对象形式绑定行内样式，camelCase 或 kebab 均可 -->
  <div :style="{ width: width + 'px', color }">行内</div>

  <!-- 通过 CSS 变量做主题化，配合 computed 批量生效 -->
  <div :style="themeVars">CSS 变量驱动</div>
</template>
```

★ **关键认知**：`:style` 同样接受对象/数组；`:style` 绑定 **CSS 自定义属性（`--xxx`）** 是主题化/动态样式的现代解法——把可变的量抽成 CSS 变量，数据一变整棵子树跟着变。⚠ 需要**自动加浏览器前缀**的样式，用 `:style` 绑定时可以加对应前缀属性，或直接交给构建阶段的 `autoprefixer`。

# 5. 本篇小结与下一步

★ **关键认知**：scoped 靠 `data-v` 属性选择器隔离（`:deep()` 精确破例），CSS Modules 靠类名哈希隔离，动态样式用 `:class`/`:style` 对象与 CSS 变量。选型：**日常 scoped，组件库/JS 操作类名用 Modules，主题化用 CSS 变量**。

下一步第 15 篇：过渡与动画——`Transition` / `TransitionGroup` 如何在元素进出时优雅地上演动画。