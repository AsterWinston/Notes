响应式数据写好之后，两类需求随之而来：一是"从现有数据**派生**出新数据"（派生值），二是"数据变了**执行副作用**"（如发请求、存 localStorage）。前者用 `computed`，后者用 `watch` / `watchEffect`。本篇讲清三者的机制差异、`computed` 的缓存本质、`watch` 的监听对象与 flush 时机，并给出一张"何时用谁"的决策表。

# 1. computed：带缓存的派生值

## 1.1 从方法到计算属性

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const list = ref([
  { name: '苹果', price: 5 },
  { name: '香蕉', price: 3 }
])

// ===== 朴素写法：模板里调方法，每次渲染都重新算 =====
function totalByFn() {
  console.log('方法被调用')
  return list.value.reduce((s, i) => s + i.price, 0)
}

// ===== 计算属性写法：依赖不变就不重算 =====
const total = computed(() => {
  console.log('computed 被调用')
  return list.value.reduce((s, i) => s + i.price, 0)
})
</script>

<template>
  <p>{{ totalByFn() }}</p>  <!-- 每次渲染都计算 -->
  <p>{{ total }}</p>        <!-- 只有 list 变了才重算 -->
</template>
```

★ **关键认知**：`computed` 返回一个**只读的派生 Ref**，它内部做了**缓存**——只有它依赖的响应式数据（这里是 `list`）变了，getter 才会重新执行；依赖没变，多少次访问都直接返回上次的缓存值。⚠ 它和"在模板里调方法"的区别正在于此：方法是**每次渲染无脑执行**，computed 是**依赖追踪 + 惰性缓存**。当计算量里有 `filter`/`sort`/大数据遍历时，这个区分的性能差异极大。

## 1.2 computed 是只读的

```typescript
import { computed, ref } from 'vue'

const firstName = ref('张')
const lastName = ref('三')
const fullName = computed(() => firstName.value + lastName.value)

console.log(fullName.value)   // 张三（注意：computed 也是 ref，读要 .value）
// fullName.value = '李四'    // ✗ 只读！computed 默认没有 setter
```

★ **关键认知**：`computed(() => ...)` 返回的是**只读 ref**，直接赋值会报错。需要可写时传带 `get/set` 的对象：

```typescript
const fullName = computed({
  get: () => firstName.value + lastName.value,
  set: (v: string) => {
    [firstName.value, lastName.value] = [v[0] ?? '', v.slice(1)]  // 拆分配名
  }
})
```

⚠ 可写 computed 本质上把"多个源合成一个可写目标"，常用于封装"一个 UI 状态背后对应多个数据"的场景。

# 2. watch：监听变化执行副作用

## 2.1 监听单个 ref

```typescript
import { ref, watch } from 'vue'

const keyword = ref('')

// 监听 keyword，变化时（新值, 旧值）传入回调
watch(keyword, (newVal, oldVal) => {
  console.log(`搜索词：${oldVal} → ${newVal}`)
})
```

★ **关键认知**：`watch(source, cb)` 的 `source` 是**被监听的数据**。它默认是**惰性**的——只有 `keyword` 真正变化时才触发，初始化时**不会**执行一次。⚠ 回调的 `oldVal` 对**基本类型**是真实旧值，但**监听对象/数组时新旧值指向同一引用**（都是同一个代理），`oldVal === newVal`，拿不到"深拷贝的快照"。

## 2.2 监听 reactive 对象

```typescript
import { reactive, watch } from 'vue'

const state = reactive({ count: 0, nested: { n: 1 } })

// 直接传 reactive 对象：默认深监听，任何深层属性变化都触发
watch(state, (newVal, oldVal) => {
  console.log('变了', newVal.count)
})
state.nested.n++   // 触发
```

★ **关键认知**：`watch` 直接监听 `reactive` 对象时，默认就是**深监听**，嵌套属性变化也会触发。⚠ 但监听 `ref` 且 `ref` 里是对象时，想深监听必须**显式传 `{ deep: true }`**——这是官方 API 里一个著名的不对称陷阱：

```typescript
const obj = ref({ a: { b: 1 } })
watch(obj, cb)                 // ✗ 只有 obj.value 被整体替换才触发
watch(obj, cb, { deep: true }) // ✓ 深层属性变化也触发
```

## 2.3 监听多个源与 getter 函数

```typescript
import { ref, reactive, watch } from 'vue'

const a = ref(1)
const b = ref(2)
const state = reactive({ x: 1 })

// 数组形式：任一源变化都触发
watch([a, b], ([na, nb], [oa, ob]) => {
  console.log(na, nb)
})

// getter 函数：只监听"计算出的那个值"（更精确）
watch(() => state.x, (n) => console.log('x 变了', n))
```

★ **关键认知**：`source` 除了 ref/ 数组，还能传 **getter 函数**（`() => 某值`）。getter 是**精准监听某一具体派生值**的标准姿势，比监听整个对象更省。⚠ 监听 `reactive` 的可选属性时，getter 返回 `undefined` 会收不到变化，此时建议 getter 返回整个对象或配合 `deep`。

## 2.4 立即执行与停止

```typescript
import { ref, watch } from 'vue'

const count = ref(0)

// immediate：注册后立刻执行一次回调
watch(count, (n) => console.log(n), { immediate: true })

// watch 返回一个"停止函数"，调用即解除监听
const stop = watch(count, (n) => console.log(n))
stop()   // 之后 count 再变也不触发了
```

★ **关键认知**：`{ immediate: true }` 把默认的"变化才触发"改成"**先跑一次再监听**"；`watch` 的返回值是**停机函数**，主动调用可解除监听。⚠ 在组件里，`watch` 会随组件卸载**自动停止**，但你若在**组件外**（如全局单例、异步回调里）手动 `watch`，就要自己保管 `stop` 避免泄漏。

# 3. watchEffect 与 flush 时机

## 3.1 watchEffect：不用写依赖，自动追踪

```typescript
import { ref, watchEffect } from 'vue'

const count = ref(0)
const double = ref(0)

// 不需要声明依赖：函数里读了谁，就自动监听谁
watchEffect(() => {
  // count 和 double 都被读取，于是任意一方变化都会重跑
  console.log(count.value + double.value)
})
```

★ **关键认知**：`watchEffect` 是 `watch` 的**自动档**——回调里读到哪些响应式数据，就自动追踪哪些，无需手写依赖列表。⚠ 代价是"全自动"意味着**你没法精确控制监听范围**，回调里碰了多少响应式数据就绑多少，稍不留意会过度触发。审慎用法：副作用逻辑简单、想省心时用。

## 3.2 watch vs watchEffect 对照

| 维度 | `watch` | `watchEffect` |
|------|---------|---------------|
| 依赖声明 | 显式传 source | 自动追踪 |
| 旧值 | 提供 `oldVal` | 不提供 |
| 立即执行 | 默认惰性，可 `immediate` | 默认立即执行一次 |
| 精确控制 | 强（getter 精确取值） | 弱 |
| 典型场景 | 值变化时做特定动作、比较新旧 | 副作用读取多源、无需旧值 |

## 3.3 flush 时机与 nextTick

```typescript
import { ref, watch, nextTick } from 'vue'

const count = ref(0)
watch(count, () => {
  // 此时 DOM 可能还没更新！默认 flush 是 pre，回调先于 DOM 更新
  console.log('watch 触发')
})

count.value++
// 同步执行到这里 DOM 还没渲染，若想操作最新 DOM：
await nextTick()
console.log(document.querySelector('#app')?.textContent)  // 已是最新
```

★ **关键认知**：Vue 的更新是**异步批量**的——一次同步代码里改 N 次数据，只在微任务里**合并成一次**重新渲染。`watch` 默认在渲染前触发（`flush: 'pre'`）。想在**渲染后**做 DOM 相关副作用，用 `{ flush: 'post' }` 或 `watchPostEffect`，或手动 `await nextTick()`。这个"异步更新时机"是 Vue 最反直觉的机制之一，面试必考，第 10 篇会结合生命周期展开。

# 4. 何时用谁：决策表

| 需求 | 选择 | 理由 |
|------|------|------|
| 从数据派生出新值，随读随新、可缓存 | `computed` | 惰性缓存，性能好 |
| 数据变化后做一次动作（发请求、写日志） | `watch` | 明确 source + 旧值 |
| 副作用逻辑简单、读多源、无需旧值 | `watchEffect` | 自动追踪省事 |
| 需要操作"渲染后的 DOM" | `watch` + `{ flush: 'post' }` | 等 DOM 更新后再动作 |
| 模板里纯展示计算 | `computed` | 别用方法函数 |

★ **关键认知**：一句话记账——**派生值用 computed，副作用用 watch**。最常见的错误是把"发请求"写进 `computed`（每次依赖变化甚至访问都可能触发异步副作用，且 computed 应为纯函数），或把"纯计算"写进 `watchEffect`（丢了缓存、可能过度触发）。

# 5. 本篇小结与下一步

★ **关键认知**：`computed` 缓存派生值，`watch` 响应旧值驱动的副作用，`watchEffect` 自动追踪副作用——三者的共同底座是响应式系统的**依赖收集**，区别只是"读"来的依赖用来干嘛（算值还是做动作）。

下一步第 5 篇：`v-if` 与 `v-for`——条件与列表渲染如何用、`key` 为什么直接决定 diff 的正确性与性能。