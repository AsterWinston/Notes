前面三篇讲了原理，本篇把它们落地为**可操作的提速手段**。性能优化的三个层面——**响应式层**（少制造不必要的响应式追踪）、**渲染层**（跳过不必要的渲染与组件加载）、**打包层**（减小首屏体积）——本篇逐条给出"何时用、怎么用、能省什么"。

# 1. 响应式层优化

## 1.1 不该响应的数据别进响应式系统

```typescript
import { reactive, ref, markRaw, shallowRef } from 'vue'

// 大对象但只需整体替换 → shallowRef，省去深度代理
const bigConfig = shallowRef(loadHugeConfig())

// 完全不变的数据（第三方实例、大静态结构）→ markRaw 标记为"永不代理"
const chart = markRaw(createChart())     // 塞进 reactive 也不会被代理

// 纯静态、大量渲染的列表 → shallowRef 数组
const staticList = shallowRef(largeStaticArray)
```

★ **关键认知**：响应式追踪是有开销的（深入代理、依赖收集）。**不会被改动的数据不该被追踪**——用 `shallowRef`（只追整体替换）或 `markRaw`（标记永不代理）把它们挡在响应式系统之外。⚠ 银行级优化的第一原则是"**别为不需要响应性的数据付响应性的税**"，第 3 篇的 shallow 家族在这里终于落地。

## 1.2 控制副作用的触发粒度

```typescript
import { ref, reactive, computed } from 'vue'

// ✗ 反例：watch 监听整个大对象，任何字段变都触发
watch(bigState, cb)

// ✓ 精准监听：getter 只取关心的字段
watch(() => bigState.nested.specific, cb)

// ✓ 派生值用 computed 缓存，别在模板里反复算
const filtered = computed(() => list.value.filter(heavyCondition))
```

★ **关键认知**：watch/effect 的触发粒度决定了"一次改动要跑多少代码"。**精准的 getter、缓存的 computed** 能把副作用范围缩到最小。⚠ 全局大对象配 `watch(state, cb)` 是性能隐患——改任何一处都触发一个可能很重的回调，用 getter 收敛到具体 key。

# 2. 渲染层优化

## 2.1 跳过不必要的渲染

```vue
<script setup lang="ts">
import { ref } from 'vue'
const list = ref([1, 2, 3])
</script>

<template>
  <!-- v-memo：依赖没变时跳过整段子树的重新渲染 -->
  <div v-memo="[list]">
    <!-- 只有 list 引用变了才重渲染这块 -->
    <p v-for="i in list" :key="i">{{ expensive(i) }}</p>
  </div>

  <!-- v-once：只渲染一次，之后永不更新（纯一次性的静态块） -->
  <div v-once>
    这个区块只随首次渲染，之后数据再变也不动
  </div>
</template>
```

★ **关键认知**：两个"渲染跳过"指令——**`v-memo`** 是**有条件跳过**（依赖数组没变就复用上次的子树），**`v-once`** 是**永久跳过**（只渲染一次）。⚠ 它们只适合"确定变化少/不变化"的区块，滥用在会变的地方会导致界面不更新——`v-once` 尤为危险，务必用于真正静态的内容。

## 2.2 组件级优化

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

// 异步组件：用到才加载（配合路由懒加载，第 18 篇）
const HeavyChart = defineAsyncComponent(() => import('./HeavyChart.vue'))
</script>

<template>
  <HeavyChart v-if="showChart" />
</template>
```

```typescript
// 组合式函数提取：避免大 setup 里重复创建
function useDebounced(value: Ref<string>) {
  // 防抖逻辑，减少高频触发
}
```

★ **关键认知**：组件级两条——**异步组件/懒加载**推迟加载（`defineAsyncComponent` + 动态 import），**组合式函数**减少重复开销。⚠ 异步组件首次渲染会有一个"加载中"的空档，配合 `Suspense` 或 loading 状态使用体验更好。

## 2.3 列表优化小结

★ **关键认知**：列表是渲染大头——**稳定的 `key`**（第 5、20 篇）确保复用、**`computed` 过滤**避免每次渲染重算、**`v-memo` 锁住不动的区块**、**`v-once` 静态项**。这几条组合起来，大列表能省下大量 diff 与重渲染。

# 3. 打包层优化

## 3.1 减小首屏体积

```typescript
// 路由懒加载（第 18 篇）：每个页面切独立 chunk
const routes = [
  { path: '/dashboard', component: () => import('../views/Dashboard.vue') }
]

// 按需引入：别整库打包
// ✗ import lodash from 'lodash'        → 打包整个库
// ✓ import { debounce } from 'lodash-es' → 只打包用到的
```

★ **关键认知**：首屏性能的胜负手在**包体积**——**路由/组件懒加载**（按需下载）、**按需引入**（tree-shaking 友好）、**第三方库选 ESM 版本**（可 tree-shaking）。⚠ `lodash`（CJS 全量）与 `lodash-es`（ESM 可摇树）的产物体积能差一个量级，选库时优先选**支持 tree-shaking 的 ESM 包**。

## 3.2 打包分析

```powershell
# 用 rollup-plugin-visualizer 看产物体积分区
npm install -D rollup-plugin-visualizer
# 在 vite.config.ts 引入后，npm run build 产出 stats.html
```

★ **关键认知**：**先测量再优化**——用打包分析工具看清"谁占据了大头"（某个图库？某条误引入的整库？），再针对性下手。⚠ 盲目的"直觉优化"常事倍功半，体积分布图会直接告诉你该砍哪一刀。

## 3.3 分层优化清单

| 层 | 手段 | 收益 |
|----|------|------|
| 响应式 | shallowRef / markRaw / 精准 watch | 减少追踪与触发开销 |
| 渲染 | v-memo / v-once / 稳定 key / computed | 跳过重复渲染与 diff |
| 加载 | 异步组件 / 路由懒加载 | 降低首屏体积 |
| 打包 | 按需引入 / ESM 库 / 分析工具 | 减小产物体积 |

★ **关键认知**：性能优化有主次——**先打包层（收益最大、改动最小）→ 再渲染层（列表、懒加载）→ 最后响应式层（微观、难改）**。别一上来就纠结 "shallowRef 省几毫秒"，先把大块体积和重复渲染解决掉。

# 4. 本篇小结与下一步

★ **关键认知**：三层优化口诀——**不该响应的别进响应式（响应式层）、不变的节点跳过渲染（渲染层）、用不到的代码不下载（打包层）**。它们的共同底色是前几篇的原理：响应式追踪洁、diff 复用、编译优化。

下一步第 23 篇：面试要点与总结——高频面试题 + 全系列知识地图，收官。