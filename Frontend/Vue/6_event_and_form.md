交互的本质是"用户做了什么 → 数据怎么变"。事件处理（`@`）负责捕获用户动作，`v-model` 负责把表单输入反向同步回数据。本篇讲清事件绑定与修饰符全集、`v-model` 的双向绑定本质——它其实就是一个语法糖——以及 `.lazy/.number/.trim` 与组件 v-model 的预告。

# 1. 事件处理 v-on

## 1.1 绑定事件与简写

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)

function increment(step: number, e: Event) {
  count.value += step
  console.log(e.type)        // 'click'
}
</script>

<template>
  <!-- v-on:click 的简写是 @click -->
  <button v-on:click="increment">+</button>
  <button @click="increment">+</button>

  <!-- 内联传参：想访问原生事件，显式传 $event -->
  <button @click="increment(2, $event)">+2</button>

  <!-- 直接内联语句也行 -->
  <button @click="count++">+1</button>
</template>
```

★ **关键认知**：`@click="handler"` 时 Vue 会自动把**原生事件对象**作为第一个参数传给 handler；`@click="handler(2, $event)"` 时用 `$event` 显式占位。⚠ `@click="count++"` 这种内联语句在编译时会被包成一个函数执行，可以写，但复杂逻辑还是提成方法。

## 1.2 事件修饰符全集

```vue
<template>
  <!-- .stop：阻止冒泡，等价 e.stopPropagation() -->
  <button @click.stop="onClick">停止冒泡</button>

  <!-- .prevent：阻止默认，等价 e.preventDefault() -->
  <form @submit.prevent="onSubmit">提交不刷新页面</form>

  <!-- .stop.prevent 可链式串联 -->
  <a @click.stop.prevent="doThis"></a>

  <!-- .once：只触发一次 -->
  <button @click.once="once">点到即止</button>

  <!-- .self：只有事件源是自身才触发（子元素冒泡来的不触发） -->
  <div @click.self="onSelf">只响应自身点击</div>

  <!-- .capture：捕获阶段触发（默认是冒泡阶段） -->
  <div @click.capture="onCapture">捕获阶段</div>

  <!-- .passive：不阻止默认，滚轮性能优化 -->
  <div @scroll.passive="onScroll">滚动</div>
</template>
```

★ **关键认知**：修饰符把"处理 DOM 事件细节"这件样板活从代码搬到模板，让 handler 只需关心业务。几个高频记忆点：**`.prevent` 配表单提交 / 链接跳转**，**`.stop` 配阻止冒泡**，**`.passive` 配滚动**（告诉浏览器"我绝不阻止默认"，滚动才能流畅）。⚠ `.passive` 与 `.prevent` 语义冲突，不可同用。

## 1.3 事件修饰符背后的思想

★ **关键认知**：Vue 把 DOM 事件里**最常见的 90% 需求**（阻止默认、阻止冒泡、只触发一次）抽成链式修饰符，而不是让你在 handler 里手写 `e.preventDefault()` 这类样板——这是 Vue"模板 DSL"优于裸 DOM 的地方。handler 因此是**纯函数**（只拿业务数据，不碰事件细节），更容易测试。

# 2. 表单绑定 v-model

## 2.1 v-model 是语法糖

```vue
<script setup lang="ts">
import { ref } from 'vue'
const text = ref('')
const checked = ref(false)
const picked = ref('')
</script>

<template>
  <!-- 输入框：v-model 把"值"和"input 事件"绑一起 -->
  <input v-model="text">
  <!-- 等价于 -->
  <input :value="text" @input="text = $event.target.value">

  <!-- 复选框 -->
  <input type="checkbox" v-model="checked">

  <!-- 单选钮 -->
  <input type="radio" value="a" v-model="picked">
</template>
```

★ **关键认知**：`v-model` **不是魔法**，它就是"`:value` + `@input`"的语法糖——把输入框的值和某变量**双向绑定**：值从数据流到视图，输入事件从视图流回数据。理解了这一点，后面组件上的 `v-model`（第 8 篇）就水到渠成。⚠ 绑定对象是**字符串**，即使你输入数字，`v-model` 默认也给你 `string`（要数字用 `.number`）。

## 2.2 v-model 修饰符

```vue
<script setup lang="ts">
import { ref } from 'vue'
const num = ref(0)
const name = ref('')
const msg = ref('')
</script>

<template>
  <!-- .lazy：失焦/回车才同步，而非每次输入 -->
  <input v-model.lazy="msg">

  <!-- .number：自动转成 number（能转才转） -->
  <input type="number" v-model.number="num">

  <!-- .trim：去掉首尾空格 -->
  <input v-model.trim="name">
</template>
```

★ **关键认知**：`.lazy` 改同步时机（从每次 input 到 change），适合搜索框节流；`.number` 把输入转数字，避免"数字框里拿到字符串"的经典坑；`.trim` 清首尾空格。⚠ `.number` 用 `parseFloat` 转换，**转换失败会回退成原字符串**，所以别指望它做严格校验。

## 2.3 多行、选择框与组合绑定

```vue
<script setup lang="ts">
import { ref } from 'vue'
const text = ref('')
const selected = ref('')
const multi = ref<string[]>([])
const agree = ref(false)
</script>

<template>
  <!-- 多行文本 -->
  <textarea v-model="text"></textarea>

  <!-- 下拉单选 -->
  <select v-model="selected">
    <option value="a">A</option>
    <option value="b">B</option>
  </select>

  <!-- 下拉多选（multiple）绑定数组 -->
  <select v-model="multi" multiple>
    <option value="a">A</option>
    <option value="b">B</option>
  </select>

  <!-- 真值/假值语法：勾选与否直接映射布尔 -->
  <input type="checkbox" true-value="yes" false-value="no" v-model="agree">
</template>
```

★ **关键认知**：`v-model` 会根据表单控件类型自动适配——复选组绑数组、单选框绑选中值、多选下拉绑数组。⚠ `select` 的初始值若与任何 `option` 的 `value` 都不匹配，iOS 上会显示空白，务必给 `option` 显式 `value` 且与初值一致。

# 3. 双向绑定小结与组件预告

## 3.1 v-model 的心智模型

```
        :value="text"         ←─── 数据 → 视图
  data:text ─────────────────────▶  <input>
                                   │
        @input="text=$event.target.value"
                                   │
        ◀──────────── 视图 → 数据 ─┘（用户输入）
```

★ **关键认知**：把 v-model 拆成"`:value` 单向向下 + `@input` 单向向上"，它就完全透明了。这条心智模型在**组件通信**里会再次出现——父传数据给子（props down）、子通知父改数据（emit up），合起来就是组件版"双向绑定"。

## 3.2 预告：组件上的 v-model

```vue
<!-- 组件也能 v-model：本质还是 props + emit 的组合，第 8 篇详解 -->
<MyInput v-model="text" />
```

★ **关键认知**：组件上的 `v-model` 拆开后是 `:modelValue="text"` + `@update:modelValue="text = $event"`。理解这篇的单向/双向、props/emit 后，第 8 篇的 `defineModel` 会变得轻而易举。

# 4. 本篇小结与下一步

★ **关键认知**：`@` 绑定事件，修饰符把 DOM 样板移出 handler；`v-model` 是 `:value + @input` 的语法糖，双向绑定只是两条单向链路的包装。到此基础篇收官——你已能写出带交互、带表单、带列表的完整页面。

下一步进入**阶段二 · 组件**，第 7 篇：把界面拆成组件、用 props/emits 划定数据结构化的边界。