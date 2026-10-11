组件间的状态若只靠 props/emit 和 provide/inject，一旦要**跨页面、跨模块、可持久化**，就会捉襟见肘。Pinia 是 Vue 3 官方状态管理库——一个集中式、响应式、可拆分的 store 容器。本篇讲清两种 store 写法、state/getters/actions、组件外使用，以及它与 Vuex 的定位差异。

# 1. 安装与创建

## 1.1 初始化

```typescript
// src/stores/index.ts
import { createPinia } from 'pinia'

const pinia = createPinia()
export default pinia
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import pinia from './stores'

createApp(App).use(pinia).mount('#app')   // 注册 pinia 插件
```

★ **关键认知**：Pinia 通过 `app.use(pinia)` 注入应用，为所有组件提供**全局 store 访问**能力。⚠ 与第 1 篇一致，`use` 必须在 `mount` 之前。插件本身极轻，多数项目讲 `use(pinia)` 放在入口即可。

# 2. 两种 store 写法

## 2.1 Option Store：像选项式 API

```typescript
// src/stores/useCounter.ts
import { defineStore } from 'pinia'

export const useCounter = defineStore('counter', {
  state: () => ({ count: 0 }),          // 状态
  getters: {
    double: (state) => state.count * 2   // 派生
  },
  actions: {
    increment() { this.count++ }         // 修改的方法
  }
})
```

★ **关键认知**：Option Store 用 **state / getters / actions** 三个区块组织，结构像选项式 API，直观易懂。⚠ `state` 必须是**函数**（返回新对象）而非直接对象——这是为了每个 store 实例独立、避免共享污染，和组件 `data()` 同理。

## 2.2 Setup Store：像组合式 API

```typescript
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounter = defineStore('counter', () => {
  const count = ref(0)                        // state = ref
  const double = computed(() => count.value * 2)  // getter = computed
  function increment() { count.value++ }      // action = 普通函数

  return { count, double, increment }          // 返回所有要暴露的
})
```

★ **关键认知**：Setup Store 把 store 写成一个**组合式函数**——`ref` 是 state、`computed` 是 getter、普通函数是 action，return 出去即可。它的好处是**完全复用你已会的组合式知识**，还能自由引入别的组合式函数。⚠ 内部不要用 `this`，也别在 store 里注册生命周期相关的钩子（会随 store 而非组件走）。

# 3. 使用 store

## 3.1 组件内使用

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCounter } from '../stores/useCounter'

const store = useCounter()

// 直接解构会丢响应性！用 storeToRefs 保留
const { count, double } = storeToRefs(store)
// actions 直接解构没问题（它们是方法）
const { increment } = store

increment()          // ✓ 触发状态变更
// count.value++     // 也可直接改 store 的 state（Pinia 允许）
</script>

<template>
  <p>{{ count }}</p>     <!-- 模板里 storeToRefs 解出的仍是 ref，但模板自动解包 -->
</template>
```

★ **关键认知**：`useCounter()` 返回 store 实例（每个应用里同一 store 是**单例**）。⚠ 解构 state 时会碰到第 3 篇那个老问题——直接 `const { count } = store` 丢响应性，必须 `storeToRefs` 包装；而 actions 是方法，直接解构无害。**Pinia 允许直接改 state**（`store.count++`），这是它比 Vuex 灵活的一个点。

## 3.2 组件外使用（关键差异）

```typescript
import { createPinia } from 'pinia'
import { useCounter } from './useCounter'

// ⚠ 组件外使用：必须先创建 pinia 实例并注入，否则报"no active pinia"
const pinia = createPinia()
const store = useCounter(pinia)   // 显式传入 pinia

// 或将其设为全局 active
// setActivePinia(pinia)
```

★ **关键认知**：组合式函数般的设计让 store **可以脱离组件使用**（在路由守卫、定时器、axios 拦截器里也能拿到状态）——这是相对 Vuex 的一大便利。⚠ 前提是**显式传入 pinia 实例**或先 `setActivePinia`，否则首次在组件外调用会因"没有活动 pinia"报错。

# 4. 进阶用法

## 4.1 访问其他 store 与组合

```typescript
import { defineStore } from 'pinia'
import { useUser } from './useUser'

export const useCart = defineStore('cart', () => {
  const user = useUser()        // 在 store 里引用另一个 store

  function checkout() {
    console.log(user.name)      // 跨 store 协作
  }
  return { checkout }
})
```

★ **关键认知**：store 之间可以**互相引用**（在 action 里调别的 store），实现模块化拆分——用户、购物车、订单分属不同 store，需要时互相调用，而不是塞进一个巨型 store。⚠ 注意别形成**循环依赖**（A 引 B、B 引 A），拆分时要理清依赖方向。

## 4.2 订阅与持久化

```typescript
import { watch } from 'vue'
import { useCounter } from './useCounter'

const store = useCounter()

// 订阅：任何状态变化都会触发（可用于埋点、持久化）
const unsubscribe = store.$subscribe((mutation, state) => {
  localStorage.setItem('counter', JSON.stringify(state))
})

// 插件：全局持久化（pinia-plugin-persistedstate）
// 或组件里 watch 手动同步
```

★ **关键认知**：`$subscribe` 监听**整个 store** 的变化，是"状态一变就持久化/上报"的正规落点。⚠ 需要跨刷新保留状态时，用 `pinia-plugin-persistedstate` 这类持久化插件，或手写 `$subscribe` 同步 `localStorage`，配合初始化时读回即可。

# 5. Pinia vs Vuex

| 维度 | Pinia | Vuex 4 |
|------|-------|--------|
| 状态变更 | 直接改 state | 必须 commit mutation |
| store 定义 | option / setup 两种 | 单一结构 |
| 组件外使用 | 支持 | 较繁琐 |
| TypeScript | 类型友好 | 需额外努力 |
| 模块 | 扁平多个 store | 嵌套 modules |
| 官方定位 | Vue 3 默认 | 遗留 |

★ **关键认知**：Pinia 相对 Vuex 的三大简化——**去掉 mutation**（直接改 state）、**去掉嵌套 modules**（扁平多 store）、**类型友好 + 组件外可用**。⚠ 旧的 mutations 概念在 Pinia 里就是 actions，迁移时把 `commit('x')` 改成直接调 action 即可。

# 6. 本篇小结与下一步

★ **关键认知**：store = 集中式响应式状态容器，Setup Store 写法与组合式函数同构。三要点——**解构用 storeToRefs**、**去 mutation 直接改**、**组件外用要注入 pinia**。它服务于"跨页面共享状态"，局部状态仍应留在组件内。

下一步第 18 篇：Vue Router——路由如何把 URL 映射到组件、导航守卫怎么拦截。