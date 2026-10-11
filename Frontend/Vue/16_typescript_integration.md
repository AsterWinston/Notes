前 15 篇的示例已经是 TypeScript，但"能写 TS"和"写好 TS"是两回事。本篇讲清 Vue 里 TS 的真正用法：`defineProps` / `defineEmits` 的类型声明、`ref` 与 `reactive` 的类型推导、泛型组件、以及 `vue-tsc` 如何全项目做类型检查——看完这篇，你就能让类型系统在 Vue 里"真跑起来"。

# 1. 类型声明基础

## 1.1 defineProps 的三种写法

```vue
<script setup lang="ts">
// 税一：纯类型（推荐，编译期校验）
const props = defineProps<{ title: string; count?: number }>()

// 写法二：类型 + 默认值（需要 withDefaults）
const props2 = withDefaults(defineProps<{ title: string; count?: number }>(), {
  count: 0
})

// 写法三：运行时声明（对象式）
const props3 = defineProps({ title: String, count: Number })
</script>
```

★ **关键认知**：`defineProps<T>()` 用**类型参数**声明 props 类型，最简洁、类型最准；需要**默认值**时用 `withDefaults` 包裹；对象式是运行时校验，类型通过构造函数推断但较粗。⚠ 三者选择：**纯类型 + withDefaults 覆盖 99% 场景**，对象式只在需要运行时 `validator` 时用（第 7 篇）。

## 1.2 defineEmits 类型

```vue
<script setup lang="ts">
// 调用签名形式：参数类型 + 细粒度事件名
const emit = defineEmits<{
  (e: 'update', value: number): void
  (e: 'reset'): void
}>()

// 简短形式（事件不带 payload 时）
const emit2 = defineEmits(['reset'])
</script>
```

★ **关键认知**：`defineEmits<{...}>()` 里每一条**调用签名就是一个事件**——事件名写在第一参数 `e` 的类型位置，载荷写在后续参数。这比数组形式多一层**载荷类型安全**。⚠ 模板里的 `@update="..."` 会据此获得事件参数的补全与校验。

# 2. ref 与 reactive 的类型

## 2.1 ref 的类型推导

```typescript
import { ref } from 'vue'

const count = ref(0)                    // Ref<number>
const name = ref('')                    // Ref<string>
const user = ref<{ id: number } | null>(null)   // 显式标联合类型
```

★ **关键认知**：`ref` 的类型是**自动推导**的（`ref(0)` → `Ref<number>`）。⚠ 空值初始化必须**显式标注类型**：`ref<number | null>(null)`，否则推成 `Ref<null>`，后续赋值 `count.value = 1` 会报类型错——这是 Vue + TS 的最高频报错之一。

## 2.2 reactive 与 readonly

```typescript
import { reactive, readonly } from 'vue'

const state = reactive({ count: 0, user: { name: '' } })
state.user.name = 'Alice'   // 深层属性类型也推导出

const ro = readonly(state)  // DeepReadonly<...>，所有属性只读
```

★ **关键认知**：`reactive` 深度代理后，**深层属性的类型一并推导**，且 `reactive` 保留了完整的对象类型（不像 ref 多了 `.value` 壳）。`readonly` 返回**深度只读**类型（`DeepReadonly`），编译期就禁止写入。⚠ `reactive` 不能接受基本类型（`reactive(1)` 报错），基本类型必须 `ref`。

## 2.3 模板 ref 的类型

```vue
<script setup lang="ts">
import { ref } from 'vue'
import Comp from './Comp.vue'

// 原生元素：标具体元素类型
const inputEl = ref<HTMLInputElement | null>(null)

// 组件：用 InstanceType 拿组件实例类型
const compRef = ref<InstanceType<typeof Comp> | null>(null)
</script>

<template>
  <input ref="inputEl" />
  <Comp ref="compRef" />
</template>
```

★ **关键认知**：模板 ref 变量**必须标注类型**且允许 `| null`（挂载前是 null）——原生元素用 `HTMLInputElement` 等，组件用 `InstanceType<typeof Comp>`。⚠ 否则 TS 推导不出正确类型，回退成 `any`，白白丢了类型安全。

# 3. 泛型组件

## 3.1 泛型 <script setup>

```vue
<!-- 泛型组件：T 在 props/emit/内部都可引用 -->
<script setup lang="ts" generic="T">
const props = defineProps<{ items: T[]; selected: T }>()
const emit = defineEmits<{ (e: 'change', value: T): void }>()
</script>

<template>
  <div>{{ props.items }}</div>
</template>
```

★ **关键认知**：`<script setup generic="T">` 把组件声明为**泛型组件**，泛型参数 `T` 在整个 setup 作用域可用，`defineProps`/`defineEmits` 都能引用。⚠ `defineProps` 的泛型默认值是 `unknown`，约束如何写见下例。

## 3.2 泛型约束与默认值

```vue
<script setup lang="ts" generic="T extends { id: number }, R = string">
// 约束：T 必须是含 id 的对象；默认：R 默认 string
const props = defineProps<{ items: T[]; gen: () => R }>()
</script>
```

★ **关键认知**：`generic` 支持 **extends 约束**（`T extends {...}`）和**默认类型参数**（`R = string`），用法与 TS 泛型完全一致。泛型组件的典型例子是通用**表格、下拉选择器、列表**——它们"承载任意类型的数据"，用泛型让类型随调用处流动。⚠ 记忆限制：泛型组件在多处 `v-for` 里会被编译器自动**去重泛型签名**，一般无需手动处理。

# 4. vue-tsc 类型检查

## 4.1 为什么需要 vue-tsc

```powershell
# Vite 只做转译、不做类型检查
npm run dev       # 传的错误是"语法/运行时"错误，不是类型错

# 跑一次全项目的 Vue + TS 类型检查
npm run type-check    # 等价于 vue-tsc --noEmit
```

★ **关键认知**：**Vite 不检查类型**（它用 esbuild 擦掉类型直接转译，与第 1 篇 TS 笔记的"tsc vs esbuild"同理）。`.vue` 文件里的类型错误要靠 **`vue-tsc`** ——它是能解析 `.vue` 模板的 tsc。⚠ 要保证类型检查不被遗漏，`vue-tsc --noEmit` 应进 CI 或提交前脚本，否则类型只存在于编辑器里、不真正把关。

## 4.2 类型检查的边界

```powershell
npm install -D vue-tsc typescript
npx vue-tsc --noEmit        # 只检查不产出
```

★ **关键认知**：`vue-tsc` 的价值点与 tsc 一致，但**多了一样能力——检查模板**：模板里 `{{ foo.bar }}`、`@click="fn(x)"`、props 传值都能得到类型校验。⚠ `vue-tsc` 检查有边界：`v-if` 收窄、模板里的动态组件/插槽这类**运行时复杂结构**的推断有限，复杂的类型逻辑要尽量放进 `<script>` 而非模板。

# 5. 本篇小结与下一步

★ **关键认知**：Vue 里 TS 的四个要点——**`defineProps/defineEmits` 走类型参数**、**泛型组件走 `generic`**、**模板 ref 要显式标注**、**`vue-tsc` 负责把类型检查真正跑起来**。记住：Vite 转译 ≠ 类型检查。

下一步第 17 篇：Pinia——当状态跨组件、跨页面共享时，请出状态管理。