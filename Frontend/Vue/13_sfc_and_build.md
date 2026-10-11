前面每篇都在写 `.vue` 文件，但从没回答过：这个三合一文件到底是怎样变成浏览器可跑的代码的？本篇讲清单文件组件（SFC）的三块结构与 `<script setup>` 的编译本质，再拆解 Vite 在开发与生产两套截然不同的构建方式，最后看懂 `dist` 里到底产出了什么。

# 1. 单文件组件 SFC

## 1.1 三块结构

```vue
<script setup lang="ts">
// ① script：逻辑
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <!-- ② template：结构 -->
  <button @click="count++">{{ count }}</button>
</template>

<style scoped>
/* ③ style：样式（scoped 表示只作用于本组件，第 14 篇） */
button { color: #42b883; }
</style>
```

★ **关键认知**：SFC 把**结构（template）+ 逻辑（script）+ 样式（style）**收进一个文件，对应一个组件。这不同于 React 的"全 JS + CSS-in-JS"，也不同于老式的"HTML/CSS/JS 三分离"——它按**组件边界**而非技术类型组织代码。

## 1.2 三块如何协同

```
                    MyButton.vue
    ┌─────────────────┼─────────────────┐
    ▼                 ▼                 ▼
 <script>          <template>         <style>
 逻辑、数据    →    模板（编译成      scoped 样式
 (setup 函数)       render 函数）     （属性选择器隔离）
    │                  │                 │
    └──────────┬───────┘                 │
               ▼                         ▼
         组件的 render 函数 ──▶ 生成的 vnode ──▶ 挂载时注入样式
```

★ **关键认知**：`<script>` 的内容构成组件的 **`setup` + 状态**，`<template>` **在编译期被转成 `render` 函数**（第 21 篇详解），`<style>` 在挂载时注入。三者最终都是"JS + CSS"，SFC 只是**编译期的组织方式**，运行时没有".vue"这个东西。

## 1.3 其他块：custom blocks

```vue
<template>...</template>
<script setup>...</script>
<style>...</style>

<!-- 自定义块：常被测试/文档工具利用 -->
<docs>
  这是一段给文档生成器读的元数据
</docs>
```

★ **关键认知**：SFC 支持**自定义块**（`<docs>`、`<i18n>` 等），由相应插件在编译期消费，普通逻辑可不理会。⚠ 顶层块只有一种合法顺序惯例（script → template → style），但顺序并不强制，编译器按块名解析。

# 2. <script setup> 的编译本质

## 2.1 语法糖背后是标准写法

```vue
<script setup>
import { ref } from 'vue'
const count = ref(0)
function inc() { count.value++ }
</script>
```

★ **关键认知**：`<script setup>` 是**编译期语法糖**，编译器会把它转成等价的 `setup()` 选项——顶层变量/函数/import 自动成为 `setup` 的返回内容，供模板使用。所以"模板直接用顶层变量"不是魔法，是编译器替你做了 `return { count, inc }`。这也解释了为什么 `<script>` 与 `<script setup>` 可在同一文件共存（普通 script 放模块级/全局逻辑）。

## 2.2 编译期宏

```vue
<script setup lang="ts">
// defineProps / defineEmits / defineModel / defineExpose / defineOptions
// 这些"宏"无需 import，是编译期被替换的标记，运行时不存在
const props = defineProps<{ name: string }>()
const emit = defineEmits<{ (e: 'go'): void }>()
</script>
```

★ **关键认知**：`defineProps` 等宏是**编译期常量**，只在 `<script setup>` 内有效，运行时没有对应同名函数——所以它们**不能从外部 import 进来再用**，也**不能传给 `setup` 之外的函数**（否则编译报错）。⚠ 有个微妙点：宏不能在包裹它的函数/闭包里调用，必须顶层直接写。

# 3. Vite 的构建方式

## 3.1 开发模式：按需即时编译

```
浏览器请求 /src/App.vue
        │
        ▼
   Vite dev server（基于原生 ESM）
        │  拦截 .vue 请求
        ▼
  @vitejs/plugin-vue 即时转译 → 返回可执行 JS 模块
        │
        ▼
   浏览器原生 import 执行（无打包、极快启动）
```

★ **关键认知**：开发时 **Vite 不打包**，它利用浏览器原生 ESM，把每个 `.vue`/`.ts` 文件**单独编译**后按需返回。所以**首次启动几乎瞬时**（只转译当前页面用到的文件），文件多了也不慢——这就是 Vite 相对 webpack 的核心优势。

## 3.2 生产模式：Rollup 打包

```
源码文件们（.vue/.ts/.css）
        │
        ▼
  Rollup 打包 + 压缩 + tree-shaking
        │
        ▼
  dist/  ┌─ assets/index-[hash].js   （业务 + 框架代码，已压缩）
         ├─ assets/index-[hash].css  （提取出的样式）
         └─ index.html                （引用上述资源的壳）
```

★ **关键认知**：生产构建用 **Rollup** 把一切**打包成少量带 hash 的静态文件**，做 tree-shaking（剔除没 import 到的死代码）、压缩、代码分割。⚠ 开发与生产是**两套工具链**（dev：即时转译；build：Rollup 打包），某些行为（如动态 import 的切分、目录结构）只有 build 后才真实可见。

# 4. 认识 dist 产物

## 4.1 一个典型构建产物

```
dist/
├── index.html
├── assets/
│   ├── index-Df3xH2kL.js      # 主 chunk（入口 + 共享代码）
│   ├── About-8fKq7n3m.js      # 路由懒加载切出的独立 chunk
│   └── index-C8nYp3vq.css     # 提取合并的样式
└── favicon.ico
```

★ **关键认知**：文件名里的 `-Df3xH2kL` 是**内容 hash**——内容一变 hash 就变，浏览器据此判断要不要重新下载，配合长缓存策略（第 22 篇）。路由懒加载的页面会切成**独立 chunk**（`About-xxx.js`），访问到才加载。

## 4.2 为什么代码能变小

★ **关键认知**：生产产物比源码小几个量级，来自四件事——**tree-shaking**（剔除未引用的导出）、**压缩混淆**、**复用 chunk**（共享依赖合并）、**Vue 运行时 vs 模板编译产物**（模板已在编译期优化成高效 render 函数，见第 21 篇）。理解"build 做了编译期优化"是读懂 Vue 性能的关键。

# 5. 本篇小结与下一步

★ **关键认知**：SFC 是**编译期组织方式**（运行时没 .vue），`<script setup>` 是**语法糖**（编译成 setup 返回），Vite 开发时**即时转译**、生产时 **Rollup 打包**。三句合起来，就懂了 Vue 工程的"编译"到底发生了什么。

下一步第 14 篇：样式方案——scoped 隔离的原理、CSS Modules、`:deep()` 及其它深度选择器。