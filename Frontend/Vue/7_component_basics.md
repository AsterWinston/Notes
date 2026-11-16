组件是 Vue 应用的积木——界面被切成一块块独立、可复用的组件，数据通过 props 向下流、事件通过 emits 向上流。本篇讲清组件在 `<script setup>` 里的定义方式、全局与局部注册、props 的类型校验、emits 的声明，并立下"单向数据流"这条铁律。

# 1. 组件的定义与注册

## 1.1 组件就是一个 SFC

```vue
<!-- src/components/MyButton.vue -->
<script setup lang="ts">
const props = defineProps<{ label: string }>()
const emit = defineEmits<{ (e: 'click'): void }>()
</script>

<template>
  <button @click="emit('click')">{{ props.label }}</button>
</template>
```

★ **关键认知**：组件的最小单位是**单文件组件（SFC）**——`.vue` 文件里 `template` + `script` + `style` 三块（第 13 篇详解结构）。组件名惯例是**大驼峰（PascalCase）**（`MyButton`），文件名建议与组件名一致。

## 1.2 局部注册 vs 全局注册

```vue
<!-- 局部注册：用 <script setup> 里 import 即自动注册 -->
<script setup lang="ts">
import MyButton from './components/MyButton.vue'
import MyCard from './components/MyCard.vue'
</script>

<template>
  <MyButton label="确定" />
  <MyCard />
</template>
```

```typescript
// 全局注册：在 main.ts 里 app.component 注册，处处可用、无需 import
import { createApp } from 'vue'
import App from './App.vue'
import MyButton from './components/MyButton.vue'

const app = createApp(App)
app.component('MyButton', MyButton)   // 全局注册
app.mount('#app')
```

★ **关键认知**：`<script setup>` 里 `import` 进来的组件**自动就是局部注册**，模板里直接能用，无需额外写 components 选项。局部注册**按需加载、可被 tree-shaking**；全局注册省 import，但会**全局打包**、不利于按需。⚠ 通用组件，尤其是会被很多页面复用的（按钮、弹窗），用全局注册；业务组件一律局部注册。

# 2. props：父 → 子

## 2.1 defineProps 定义与使用

```vue
<!-- 子组件 Child.vue -->
<script setup lang="ts">
// 对象形式：带默认值
const props = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 }
})

// 纯类型形式（推荐 + TS 友好），校验交给 TypeScript
// const props = defineProps<{ title: string; count?: number }>()
</script>

<template>
  <h2>{{ props.title }}</h2>
  <p>{{ props.count }}</p>
</template>
```

```vue
<!-- 父组件 -->
<script setup lang="ts">
import Child from './Child.vue'
</script>

<template>
  <Child title="标题" :count="5" />
</template>
```

★ **关键认知**：`defineProps` 是**编译期宏**，无需 import、只在 `<script setup>` 里用。它声明子组件接收哪些属性。TS 项目里推荐**纯类型形式**（`defineProps<{ ... }>()`），把校验交给类型系统；需要运行时默认值才用对象形式。⚠ props 是**只读的**——子组件绝不能直接改 `props.xxx`，这触及下面的单向数据流。

## 2.2 props 的本质：单向数据流

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ===== ⚠ 反例：直接在子组件里改 prop =====
// props.count++                 // ✗ 编译警告/运行时错误，且父组件感知不到

// ===== 正确：需要派生本地状态时，用计算属性或内部变量 =====
import { computed } from 'vue'
const doubled = computed(() => props.count * 2)   // 只读派生
</script>
```

★ **关键认知**：props 是父组件"下发"的**只读数据**，数据流**只能自上而下**——父改 prop 子跟随更新，但子不能反向篡改 prop。⚠ 想"改变"父传来的值，正确姿势是**子组件 emit 事件通知父组件改**，而不是自己动手。这条"单向数据流"铁律是 Vue/React 组件体系的共同基石，违背它会导致状态来源混乱、难排查。

# 3. emits：子 → 父

## 3.1 defineEmits 声明与触发

```vue
<!-- 子组件 Counter.vue -->
<script setup lang="ts">
// 声明能发出哪些事件（配合类型 payload）
const emit = defineEmits<{
  (e: 'update', value: number): void
  (e: 'reset'): void
}>()

function add() {
  emit('update', 1)     // 发出事件 + 载荷
}
</script>

<template>
  <button @click="add">+1</button>
</template>
```

```vue
<!-- 父组件监听 -->
<script setup lang="ts">
import { ref } from 'vue'
import Counter from './Counter.vue'
const total = ref(0)
</script>

<template>
  <Counter @update="(v) => total += v" @reset="total = 0" />
</template>
```

★ **关键认知**：`emit` 是子组件"向上喊话"的唯一通道——**props 下、emit 上**，二者合起来构成组件间数据流动的完整闭环。⚠ emits 列表不仅要"发"，更要在定义里**声明**：声明能获得类型补全，也方便阅读者一眼看清组件的对外契约。

## 3.2 事件名规范

```vue
<!-- ⚠ 反例：驼峰事件名在 DOM 模板里会大小写不敏感，推荐 kebab 或约定一致 -->
<script setup lang="ts">
const emit = defineEmits<{ (e: 'onClick'): void }>()
</script>
```

★ **关键认知**：DOM 模板中事件名**大小写不敏感**，`@onClick` 与 `@onclick` 等价，容易出歧义。社区约定事件名用 **kebab-case**（如 `@update:model-value`）或统一 camelCase，关键是**项目内一致**。SFC 的 `template` 因为走编译期，对大小写更宽松，但不建议养成依赖它的习惯。

# 4. props 校验的运行时兜底

```vue
<script setup lang="ts">
const props = defineProps({
  status: {
    type: String,
    required: true,
    validator: (v: string) => ['active', 'inactive'].includes(v)  // 自定义校验
  },
  size: {
    type: String,
    default: 'medium'
  }
})
</script>
```

★ **关键认知**：对象形式的 props 能做**运行时校验**（type / required / default / validator），不合法值会在开发模式控制台报警。⚠ 运行时校验**不是**类型系统——要编译期强类型还得用 TS 纯类型形式；两者可结合：复杂默认值 + 运行时约束用对象式，纯类型约束用类型式（第 16 篇深入）。

# 5. 本篇小结与下一步

★ **关键认知**：组件的三条骨架——**import 即注册**、**props 只读向下**、**emit 向上通知**。数据流被"单向数据流"牢牢约束，才让状态来源可追溯。

下一步第 8 篇：组件通信进阶——`defineModel` 让组件双向绑定、`$attrs` 透传属性、`defineExpose` 暴露子组件方法。