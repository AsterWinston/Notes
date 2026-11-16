props 传的是"数据"，插槽传的是"内容"——它让父组件把自己的模板片段"塞"进子组件的指定位置，实现真正的内容分发。本篇讲清默认插槽、具名插槽、作用域插槽（子组件把数据回传给父组件的字符），以及动态插槽名，最后点破插槽在底层其实就是函数。

# 1. 插槽基础

## 1.1 默认插槽

```vue
<!-- 子组件 Card.vue：<slot> 是内容"落点" -->
<template>
  <div class="card">
    <slot>这里是默认内容：父组件没传时显示</slot>
  </div>
</template>
```

```vue
<!-- 父组件：写在子组件标签内部的内容，就是插槽内容 -->
<Card>
  <p>这是父组件塞进来的内容</p>
</Card>

<!-- 不传则显示子组件的默认内容 -->
<Card />
```

★ **关键认知**：插槽的本质是**父组件把自己的模板片段注入子组件预留的 `<slot>` 位置**。`<slot>` 标签之间的内容是**后备内容（fallback）**，父组件没传时展示。⚠ 一个组件可以只用一个默认 `<slot>`，这是最常见形态。

## 1.2 插槽内容的编译时机

★ **关键认知**：插槽内容（写在父组件里的那段模板）的**数据作用域是父组件**的——它访问的是父组件的状态，而非子组件的。这点和作用域插槽（见第 3 节）形成对照：默认插槽里你拿不到子组件的数据。

## 1.3 具名插槽

```vue
<!-- 子组件 Layout.vue -->
<template>
  <header><slot name="header">默认头</slot></header>
  <main><slot>默认主内容</slot></main>
  <footer><slot name="footer">默认脚</slot></footer>
</template>
```

```vue
<!-- 父组件：# 是 v-slot: 的简写 -->
<Layout>
  <template #header>
    <h1>标题</h1>
  </template>
  <!-- 不写名字的内容进默认插槽 -->
  <p>主内容</p>
  <template #footer>
    <p>版权</p>
  </template>
</Layout>
```

★ **关键认知**：多个落点时用**具名插槽**——子组件 `<slot name="x">`，父组件 `<template v-slot:x>`（简写 `#x`）。⚠ 没有 `name` 的 `<slot>` 就是默认插槽（隐含 `name="default"`），父组件不带 `v-slot` 的裸内容都进默认插槽。

## 1.4 动态插槽名

```vue
<template>
  <Comp>
    <!-- #[] 动态插槽名：名字来自数据 -->
    <template #[dynamicSlotName]>
      内容
    </template>
  </Comp>
</template>
```

★ **关键认知**：`v-slot:[dynamicName]` 支持**动态插槽名**（类似动态属性/事件）。⚠ 动态插槽名不能是任意表达式，必须能静态解析，且不能与静态插槽名混用在同一 `<template>` 上。

# 2. 作用域插槽：数据回传

## 2.1 子组件把数据传给父组件

```vue
<!-- 子组件 List.vue：slot 上带属性，如"回传"每个 item -->
<script setup lang="ts">
import { ref } from 'vue'
const items = ref([{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }])
</script>

<template>
  <ul>
    <!-- 循环里把 item 挂在 slot 上 -->
    <li v-for="item in items" :key="item.id">
      <slot :item="item" :index="item.id"></slot>
    </li>
  </ul>
</template>
```

```vue
<!-- 父组件：v-slot 接住这些"回传"的数据 -->
<List v-slot="{ item, index }">
  <span>{{ index }}: {{ item.name }}</span>
</List>
```

★ **关键认知**：**作用域插槽**让数据**反向流动**——子组件在 `<slot>` 上挂属性，父组件用 `v-slot="{ ... }"` 解构接住。这里的作用域是：**数据由子组件提供，渲染逻辑由父组件决定**。⚠ 具名作用域插槽写作 `v-slot:header="{ item }"` 或 `#header="{ item }"`。

## 2.2 经典用例：自定义列表渲染

```vue
<!-- 一个"数据自己管、每条长啥样由使用者定"的通用列表 -->
<DataTable :data="users" v-slot="{ row }">
  <td>{{ row.name }}</td>
  <td>{{ row.email }}</td>
</DataTable>
```

★ **关键认知**：作用域插槽是**通用组件**的标配——组件负责"拿数据 + 循环"，使用者负责"每一条怎么渲染"。这是比传 render 函数更模板化、更 Vue 的复用方式。

# 3. 插槽的底层：本质是函数

```javascript
// 编译器把插槽编译成一个函数：传入作用域数据，返回 vnode
// <slot :item="x"> 大致等价于：
slot.default({ item: x })
```

★ **关键认知**：**插槽在底层就是函数**——`v-slot` 声明的是一段"接收作用域参数、返回节点"的模板函数。这解释了三个现象：①作用域插槽能"回传"数据，本质是调用这个函数时传参；②插槽内容作用域是父组件（闭包在父组件那侧创建）；③插槽可以拿到渲染函数 `useSlots()` 来程序化地调用（第 21 篇渲染原理会再提）。⚠ 理解"插槽 = 函数"是吃透高阶组件、命令式调用插槽的钥匙。

# 4. 本篇小结与下一步

★ **关键认知**：默认插槽塞内容、具名插槽塞位置、作用域插槽塞"带数据的模板函数"。三者都是"内容分发"的不同粒度，而底层统一是"父给子的渲染函数"。

下一步第 10 篇：生命周期——组件从创建到销毁经历了什么、何时该 `onMounted` 何时该 `onBeforeUnmount`。