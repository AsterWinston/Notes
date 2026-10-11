动画让状态切换从"生硬地蹦"变成"平滑地过渡"。Vue 的 `Transition` 组件在元素进入/离开时，自动在合适的时机挂上/摘下约定好的 CSS 类，你只需写好起止样式。本篇讲清 `Transition` / `TransitionGroup` 的用法、六个切换类名、JS 钩子，以及列表动画的技巧。

# 1. Transition 基础

## 1.1 一个最小淡入淡出

```vue
<script setup lang="ts">
import { ref } from 'vue'
const show = ref(false)
</script>

<template>
  <button @click="show = !show">切换</button>

  <!-- Transition 包住"可能要进出"的元素 -->
  <Transition name="fade">
    <p v-if="show">你好，世界</p>
  </Transition>
</template>

<style>
/* 六个约定类名，name="fade" 作为前缀 */
.fade-enter-from { opacity: 0; }        /* 进入起点 */
.fade-enter-active { transition: opacity 1s; }  /* 进入过程 */
.fade-enter-to { opacity: 1; }          /* 进入终点 */
.fade-leave-from { opacity: 1; }        /* 离开起点 */
.fade-leave-active { transition: opacity 1s; }  /* 离开过程 */
.fade-leave-to { opacity: 0; }          /* 离开终点 */
</style>
```

★ **关键认知**：`Transition` 的工作方式是**在元素进入/离开时，依次叠加这六个类名**，你负责在 `-active` 类里写 `transition`、在 `-from`/`-to` 类里写起止样式，动画由 CSS 自己完成。⚠ `Transition` 只对**单个元素/组件**生效，且该子元素必须是**条件渲染（v-if/v-show）或动态切换**才触发进出动画；常驻元素不会动。

## 1.2 六类名时间线

```
进入进入                       离开
  v-enter-from ──▶ v-enter-to     v-leave-from ──▶ v-leave-to
     （加 v-enter-active 的 transition）
```

★ **关键认知**：六个类名本质是**三对**——进入（from/active/to）与离开（from/active/to）。`-active` 负责"过渡在此期间生效"，`-from`/`-to` 负责"起点与终点的样式"。不写 `name` 时前缀是默认的 `v-`。

# 2. 高级过渡

## 2.1 自定义过渡类名与动画库

```vue
<script setup lang="ts">
import { ref } from 'vue'
const show = ref(false)
</script>

<template>
  <Transition
    enter-from-class="opacity-0"
    enter-active-class="transition-opacity duration-500"
    leave-to-class="opacity-0"
  >
    <p v-if="show">配合 Tailwind/Velocity 等库</p>
  </Transition>
</template>
```

★ **关键认知**：通过 `enter-from-class` 等 props 可以**自定义类名**，从而配 **Tailwind、Animate.css** 等现成动画库——把 `-active` 换成库提供的类即可，无需手写 keyframes。⚠ 提供自定义类名会**覆盖**默认的 `name` 前缀类，二者取其一。

## 2.2 JS 钩子

```vue
<template>
  <Transition
    @before-enter="onBeforeEnter"
    @enter="onEnter"
    @after-enter="onAfterEnter"
    @before-leave="onBeforeLeave"
    @leave="onLeave"
    @after-leave="onAfterLeave"
  >
    <p v-if="show">用 JS 精确控制动画</p>
  </Transition>
</template>
```

```vue
<script setup lang="ts">
// 以 GSAP 为例：在 JS 里操作属性，enter/leave 必须调用 done
function onEnter(el: Element, done: () => void) {
  // gsap.from(el, { opacity: 0, onComplete: done })
  done()
}
</script>
```

★ **关键认知**：JS 钩子让动画脱离 CSS 的束缚，交给 GSAP、Veloicity 这类库做弹簧/序列动画。⚠ `enter` / `leave` 钩子里，**动画结束时必须手动调用 `done`**，否则 Vue 不知道动画已结束，会卡住后续状态；只有纯 CSS 动画才不用 `done`。

## 2.3 首次渲染与过渡模式

```vue
<template>
  <!-- appear：首次挂载也播放进入动画 -->
  <Transition appear>
    <p v-if="show">首次也过渡</p>
  </Transition>

  <!-- mode="out-in"：旧元素先离开，新元素再进入（避免两个同时出现） -->
  <Transition name="fade" mode="out-in">
    <p :key="page">{{ page }}</p>
  </Transition>
</template>
```

★ **关键认知**：`appear` 让**首次挂载**也走进入动画；`mode="out-in"` / `mode="in-out"` 控制**交替元素**的进出顺序——`out-in` 先出后进（推荐，避免重叠闪烁），`in-out` 先进后出。⚠ 元素间用 `key` 区分（`<p :key="page">`），Vue 才知道这是"切换"而非"原地改内容"。

# 3. TransitionGroup 列表动画

## 3.1 列表增删的过渡

```vue
<script setup lang="ts">
import { ref } from 'vue'
const list = ref([1, 2, 3])
function add() { list.value.push(list.value.length + 1) }
</script>

<template>
  <button @click="add">添加</button>

  <TransitionGroup name="list" tag="ul">
    <li v-for="i in list" :key="i">{{ i }}</li>
  </TransitionGroup>
</template>

<style>
.list-enter-from { opacity: 0; transform: translateY(20px); }
.list-enter-active { transition: all 0.5s; }
.list-leave-active { transition: all 0.5s; position: absolute; }
.list-leave-to { opacity: 0; transform: translateY(-20px); }
/* move 类：处理"位置变化"的平滑移动 */
.list-move { transition: transform 0.5s; }
</style>
```

★ **关键认知**：`TransitionGroup` 是 `Transition` 的**列表版**，为每个带唯一 `key` 的子项独立应用进出动画。⚠ 独特在 `.xxx-move` 这个类——它让列表**排序/重排时**元素"平滑滑到新位置"而非瞬移，这是做"列表重排动画"的灵魂。`tag` 指定包裹的真实标签（否则默认渲染成无 `ul` 包裹）。

## 3.2 Transition 与 TransitionGroup 对照

| 维度 | Transition | TransitionGroup |
|------|-----------|-----------------|
| 包裹内容 | 单个元素/组件 | 多个元素（v-for 列表） |
| 需要 key | 切换场景建议 | 每个子项必须 |
| move 动画 | 无 | 有（.xxx-move） |
| 渲染标签 | 无自身标签 | 用 tag 指定 |

# 4. 本篇小结与下一步

★ **关键认知**：`Transition` 靠六类名 + CSS transition 做单体进出动画，`TransitionGroup` 靠 `.xxx-move` 做列表重排动画。核心心法：**动画的本质是"起止样式 + 期间让 transition 生效"**，Vue 只负责在正确时机挂类名。

下一步第 16 篇：TypeScript 集成——`defineProps` 泛型、泛型组件、`vue-tsc`，让 Vue 也能享受完整的类型安全。