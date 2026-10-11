组件的存在不是一瞬，而是一条从"创建 → 挂载 → 更新 → 卸载"的生命线。把握生命周期，才能知道"数据到底在哪个时间点生效""DOM 什么时候能碰""副作用该什么时候清理"。本篇讲清组合式 API 的钩子全集、`setup` 本身的执行时机、更新与卸载的清理纪律，以及它和 `nextTick` 的关系。

# 1. 生命周期全景

## 1.1 一条线从生到死

```
   创建      挂载       更新（可多次）        卸载
     │         │            │                  │
  setup()  ┌───────┐   ┌─────────┐        ┌───────┐
  ──▶ onBeforeMount ──▶ onMounted ──▶ onBeforeUpdate ──▶ onUpdated ──▶ onBeforeUnmount ──▶ onUnmounted
     │         ▲                       (重复)               │
     ▲         │                                            ▼
   组件实例  DOM 已插入                                 DOM 已移除
```

## 1.2 组合式钩子全集

```vue
<script setup lang="ts">
import {
  ref,
  onBeforeMount, onMounted,
  onBeforeUpdate, onUpdated,
  onBeforeUnmount, onUnmounted
} from 'vue'

const count = ref(0)

onBeforeMount(() => console.log('1. 挂载前，DOM 未生成'))
onMounted(() => console.log('2. 挂载后，可安全碰 DOM'))
onBeforeUpdate(() => console.log('3. 数据变了、重渲染前'))
onUpdated(() => console.log('4. 重渲染后'))
onBeforeUnmount(() => console.log('5. 卸载前，适合清理'))
onUnmounted(() => console.log('6. 卸载后，组件彻底没了'))
</script>
```

★ **关键认知**：组合式生命周期的钩子都以 `onXxx` 命名，**必须在 `setup` 执行期间（同步）调用**，异步注册（如 `setTimeout(() => onMounted(...))`）会失效或不报警。⚠ 这些钩子内部可以多次调用——同一个 `onMounted` 写两处，两个回调都会跑。

## 1.3 与选项式的对应

| 组合式 | 选项式 |
|--------|--------|
| `setup()` | `setup` |
| `onBeforeMount` | `beforeMount` |
| `onMounted` | `mounted` |
| `onBeforeUpdate` | `beforeUpdate` |
| `onUpdated` | `updated` |
| `onBeforeUnmount` | `beforeUnmount` |
| `onUnmounted` | `unmounted` |

★ **关键认知**：两者一一对应，只是组合式去掉了 `this`。`<script setup>` 本身就是 `setup` 函数体——写在顶层的代码（`const count = ref(0)` 等）都在 `setup` 阶段执行，这就是为什么**模板里可直接用顶层变量**。

# 2. setup 与事件注册的时机

## 2.1 setup 里直接写就是 setup 阶段

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)   // 这就是在 setup 里：组件实例还没挂到 DOM
</script>
```

★ **关键认知**：`<script setup>` 的整个顶层作用域**就是 `setup` 函数**。这里执行的时机是**组件实例创建后、DOM 挂载前**。所以这里能安全地创建响应式数据、注册 watch，但**不能碰 DOM**（DOM 还没生成）。

## 2.2 为什么要挂 onMounted 里

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const el = ref<HTMLDivElement | null>(null)

// ✗ DOM 还没挂载，el.value 是 null
// console.log(el.value?.offsetHeight)

onMounted(() => {
  console.log(el.value?.offsetHeight)  // ✓ 这时 DOM 已在文档里
})
</script>
```

★ **关键认知**：**一切依赖 DOM 存在的逻辑都必须放 `onMounted`**——测量尺寸、初始化第三方图表/地图、手动 `focus()`。⚠ 挂载前访问模板 ref 只会得到 `null`，这是新手最常见的心急坑。

# 3. 清理纪律：onBeforeUnmount

## 3.1 副作用要在卸载前清理

```vue
<script setup lang="ts">
import { onBeforeUnmount, onMounted } from 'vue'

function onResize() { console.log('resize') }

onMounted(() => {
  window.addEventListener('resize', onResize)   // 挂上全局监听
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize) // 卸载前移除，防止内存泄漏
})
</script>
```

★ **关键认知**：`onMounted` 里挂的任何**全局副作用**（`addEventListener`、定时器、订阅、第三方实例），都必须在 `onBeforeUnmount` 里**对称清理**。⚠ 事件监听器若不清除，组件销毁后它仍活着引用组件，导致**内存泄漏**与诡异的"幽灵回调"。

## 3.2 watchEffect 的自动清理

```vue
<script setup lang="ts">
import { ref, watchEffect } from 'vue'
const id = ref(1)

watchEffect(() => {
  // watchEffect 在组件卸载时自动停止，无需手动清理
  console.log('fetch', id.value)
})
</script>
```

★ **关键认知**：`watch` / `watchEffect` / `computed` 在**组件内**注册时，会随组件卸载**自动停止**，不必手动 `stop`。这是"组件级作用域自动管理"的好处；只有**组件外**手动创建的 `watch` 才需要自己保管停止函数（第 4 篇）。

# 4. 更新时机与 nextTick

## 4.1 更新是异步批量的

```vue
<script setup lang="ts">
import { ref, watch, nextTick } from 'vue'
const count = ref(0)

function triple() {
  count.value++
  count.value++
  count.value++
  // 同步代码里连改三次，也只会触发一次重新渲染
  console.log('同步打日志时 DOM 还没更新')
}
</script>
```

★ **关键认知**：Vue 把所有同步改动**攒成一批**，在微任务里**合并成一次渲染**。`count.value++` 连写三次，最终只渲染一次、`count` 直接到 3。⚠ 这意味着**改完数据后立刻读 DOM 读到的是旧值**——想要"最新 DOM"必须等 `nextTick`。

## 4.2 nextTick 拿最新 DOM

```vue
<script setup lang="ts">
import { ref, nextTick } from 'vue'
const count = ref(0)
const el = ref<HTMLDivElement | null>(null)

async function add() {
  count.value++
  await nextTick()                // 等这次渲染完成
  console.log(el.value?.textContent)  // ✓ 已是更新后的文本
}
</script>
```

★ **关键认知**：`nextTick()` 返回一个 Promise，`await` 它之后，本次批量的 DOM 更新**已落到文档**。这正是"改数据 → 等渲染 → 读 DOM"三步的标准节奏。⚠ 它不等于"下一次事件循环"，只保证"这一次 Vue 的渲染任务已完成"。

# 5. 本篇小结与下一步

★ **关键认知**：生命周期纪律三条——**setup 里造数据、onMounted 里碰 DOM、onBeforeUnmount 里清副作用**。把握"更新是异步批量"就理解了 nextTick 存在的意义。

下一步第 11 篇：组合式函数——把逻辑封装成可复用的 `useXxx`，这是组合式 API 最核心的收益。