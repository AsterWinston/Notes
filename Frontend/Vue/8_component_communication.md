props + emit 解决父子通信的"主干道"，但真实场景充满更多花样：子组件要和父组件保持"双向同步"、要透传一堆属性给更深的元素、父组件要直接调用子组件的方法。本篇讲清 `defineModel`（组件版 v-model）、`$attrs` 属性透传、`defineExpose` 暴露子组件内部，以及 provide/inject 的预告。

# 1. defineModel：组件版 v-model

## 1.1 从 props+emit 到 defineModel

```vue
<!-- ===== 传统写法：v-model 拆成 props + emit，繁琐 ===== -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ (e: 'update:modelValue', v: string): void }>()
</script>

<template>
  <input :value="props.modelValue" @input="emit('update:modelValue', $event.target.value)" />
</template>
```

```vue
<!-- ===== defineModel 写法：一行搞定双向绑定 ===== -->
<script setup lang="ts">
// defineModel 返回一个 ref，读写它即自动同步父组件
const model = defineModel<string>({ required: true })
</script>

<template>
  <input v-model="model" />   <!-- 内部再 v-model 到这个 ref -->
</template>
```

```vue
<!-- 父组件直接 v-model -->
<MyInput v-model="text" />
```

★ **关键认知**：`defineModel` 是 Vue 3.4+ 提供的宏，把"组件对外暴露的 v-model"打包成一个 **ref**——子组件读写 `model.value`，父组件 `v-model` 就自动同步。它内部仍是 `modelValue` prop + `update:modelValue` emit，只是语法糖包装。⚠ 一个组件可有**多个** `defineModel`，对应多个 `v-model:x`。

## 1.2 多个 v-model 与命名

```vue
<!-- 子组件 -->
<script setup lang="ts">
const title = defineModel<string>('title', { required: true })   // 命名 v-model
const content = defineModel<string>('content', { required: true })
</script>

<!-- 父组件：v-model:title 和 v-model:content -->
<Editor v-model:title="t" v-model:content="c" />
```

★ **关键认知**：默认 `defineModel()` 对应 `v-model`（内部属性名 `modelValue`）；`defineModel('title')` 对应 `v-model:title`。表单类复杂组件（标题 + 正文、起始日期 + 结束日期）常用命名 v-model 一次暴露多个双向字段。

# 2. $attrs：透传与继承

## 2.1 默认属性继承

```vue
<!-- MyButton.vue -->
<script setup lang="ts">
// 没写的属性会自动"继承"到根元素上
</script>

<template>
  <button>按钮</button>
</template>

<!-- 使用处：class / id / 监听器自动落到根 <button> 上 -->
<MyButton class="primary" id="save" @click="onClick" />
```

★ **关键认知**：父组件传给子组件、但子组件**未声明为 props 或 emit** 的那些东西（class、style、id、其他属性、事件监听器），会打包进 `$attrs`，并**默认继承到组件的根元素**上。这就是为什么 `<MyButton class="x">` 的 class 能直接作用到 button 上。

## 2.2 显式透传给指定元素

```vue
<!-- 多根组件或用 v-bind="$attrs" 把属性转给想要的元素 -->
<script setup lang="ts">
defineProps<{ label: string }>()
</script>

<template>
  <div class="wrapper">
    <!-- 继承到内部深层元素而非根 -->
    <button v-bind="$attrs">{{ label }}</button>
  </div>
</template>
```

★ **关键认知**：`v-bind="$attrs"` 可以把透传属性**手动指派给任意元素**；多根节点组件（片段）**不会自动继承**，必须手动 `v-bind="$attrs"`，否则属性无处安放。⚠ 用了 `defineProps` 声明过的属性会从 `$attrs` 里**剔除**，不会重复继承。

## 2.3 inheritAttrs: false

```vue
<script setup lang="ts">
// 关闭默认继承，防止样式/监听器污染根元素
defineOptions({ inheritAttrs: false })
</script>
```

★ **关键认知**：`inheritAttrs: false` 关闭默认继承，配合显式 `v-bind="$attrs"` 实现精确控制。⚠ 透传的 `class`/`style` 会与目标元素已有的合并而非覆盖，这是继承机制的一个贴心细节。

# 3. ref 引用组件：defineExpose

## 3.1 模板 ref 拿到 DOM

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)
onMounted(() => {
  inputRef.value?.focus()   // 挂载后聚焦输入框
})
</script>

<template>
  <input ref="inputRef" />
</template>
```

★ **关键认知**：`ref="xxx"` 写在**原生元素**上，拿到的就是**那个 DOM 节点**。⚠ `ref` 变量名必须与模板 `ref="xxx"` 同名；配合 `ref<T>` 标注元素类型（`HTMLInputElement`）。访问时机在 `onMounted` 之后（DOM 已存在）。

## 3.2 ref + defineExpose 调用子组件方法

```vue
<!-- 子组件 Inner.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const visible = ref(false)
function open() { visible.value = true }

// ⚠ <script setup> 默认对父组件"封闭"，必须显式暴露
defineExpose({ open })
</script>

<template>
  <div v-if="visible">弹窗内容</div>
</template>
```

```vue
<!-- 父组件调用子组件方法 -->
<script setup lang="ts">
import { ref } from 'vue'
import Inner from './Inner.vue'
const innerRef = ref<InstanceType<typeof Inner> | null>(null)

function show() {
  innerRef.value?.open()   // 调用子组件暴露的方法
}
</script>

<template>
  <Inner ref="innerRef" />
  <button @click="show">打开</button>
</template>
```

★ **关键认知**：`ref="xxx"` 写在**组件**上，拿到的是组件实例；但 `<script setup>` 组件**默认对父组件封闭**（拿不到内部任何东西），只有通过 `defineExpose` 显式暴露的方法/数据才可见。⚠ 这是 `<script setup>` 与选项式组件的重要差异——选项式组件默认全暴露，`script setup` 默认全隐藏。类型上用 `InstanceType<typeof Inner>` 标注实例。

## 3.3 何时该用 ref 调用

★ **关键认知**：用 ref 直接调用子组件方法属于"**命令式逃逸**"，应谨慎——它绕过了 props/emit 的声明式数据流，多了隐式耦合。⚠ 优先用 props + emit 表达；只有聚焦、打开第三方封装的弹窗、触发动画这类"不属数据而属动作"的场景，才用 `defineExpose`。

# 4. 通信方式全景图

| 关系 | 手段 | 所在篇 |
|------|------|--------|
| 父 → 子（数据） | props | 第 7 篇 |
| 子 → 父（事件） | emit | 第 7 篇 |
| 双向同步 | defineModel / v-model:x | 本篇 |
| 属性透传 | $attrs | 本篇 |
| 调用子方法 | ref + defineExpose | 本篇 |
| 跨层/深层传递 | provide / inject | 第 12 篇 |
| 全局共享状态 | Pinia | 第 17 篇 |

# 5. 本篇小结与下一步

★ **关键认知**：通信的黄金规律是"**越直接越好**"——父子用 props/emit，双向用 defineModel，透传用 $attrs，只有跨层级才考虑 provide/inject（第 12 篇），全局才上 Pinia（第 17 篇）。能用直接手段就绝不上重的。

下一步第 9 篇：插槽——让父组件把自己的内容"塞"进子组件的指定位置。