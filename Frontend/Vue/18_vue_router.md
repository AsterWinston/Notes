SPA 的多"页面"体验靠 URL 与组件的映射——前端路由。Vue Router 是官方路由器：URL 变 → 匹配路由表 → 渲染对应组件。本篇讲清路由配置、动态路由、嵌套路由、导航守卫、路由懒加载，以及组合式 API 下的路由访问方式。

# 1. 安装与基本配置

## 1.1 路由表

```typescript
// src/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import Home from '../views/Home.vue'
import About from '../views/About.vue'

const router = createRouter({
  history: createWebHistory(),   // HTML5 history 模式
  routes: [
    { path: '/', component: Home },
    { path: '/about', component: About }
  ]
})

export default router
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App).use(router).mount('#app')
```

★ **关键认知**：路由表是 **`path → 组件`** 的映射数组；`createWebHistory()` 用 HTML5 History API（干净的 `/about` 路径），另一种 `createWebHashHistory()` 用 `#/about`（兼容老环境、免服务器配置）。⚠ History 模式部署时需要服务器把所有路径回退到 `index.html`，否则刷新子路由会 404。

## 1.2 RouterView 与 RouterLink

```vue
<template>
  <!-- 当前路由匹配到的组件渲染在这 -->
  <RouterView />

  <!-- 跳转链接：to 指明目标 -->
  <RouterLink to="/about">去关于页</RouterLink>
  <!-- active 高亮类：默认 .router-link-active -->
</template>
```

★ **关键认知**：`<RouterView>` 是"路由组件的出口"（渲染匹配组件的地方），`<RouterLink>` 是声明式的跳转链接，自动管理 `active` 高亮类。⚠ 编程式跳转用 `router.push('/about')`，二者等价；`<a href>` 硬跳转会刷新整页，脱离 SPA。

# 2. 动态与嵌套路由

## 2.1 动态路由参数

```typescript
const routes = [
  // :id 是动态段，匹配 /user/123 等
  { path: '/user/:id', component: UserDetail }
]
```

```vue
<script setup lang="ts">
import { useRoute } from 'vue-router'
const route = useRoute()
console.log(route.params.id)   // 读取动态参数
</script>
```

★ **关键认知**：`/:id` 定义动态段，`route.params.id` 读取。⚠ 两次跳转**同路由不同参数**（`/user/1` → `/user/2`）时组件**复用**不重建——因此依赖 `params` 变化而更新的逻辑要用 `watch(() => route.params.id, ...)`，不能只靠 `onMounted` 那一次。

## 2.2 嵌套路由

```typescript
const routes = [
  {
    path: '/users/:id',
    component: User,
    children: [
      { path: 'profile', component: UserProfile },  // 子路由 path 不写 /
      { path: 'posts', component: UserPosts }
    ]
  }
]
```

```vue
<!-- User.vue：子路由渲染在嵌套的 <RouterView> 里 -->
<template>
  <div class="user">
    <h2>用户 {{ $route.params.id }}</h2>
    <RouterView />   <!-- 渲染 /users/:id/profile 或 /users/:id/posts -->
  </div>
</template>
```

★ **关键认知**：嵌套路由让页面具备**层级结构**（外层布局 + 内层内容），子路由组件渲染在**父组件内部的 `<RouterView>`**。⚠ 子路由的 `path` **不要以 `/` 开头**（会逃离父路径），写相对路径会拼接到父路径后。

# 3. 导航守卫

## 3.1 全局守卫

```typescript
// beforeeach：每次导航前触发，常用于登录鉴权
router.beforeEach((to, from) => {
  const isLoggedIn = localStorage.getItem('token')
  // 去需要登录的页面但没登录 → 重定向到登录页
  if (to.meta.requiresAuth && !isLoggedIn) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }
  // return true / 不返回 → 放行
})
```

★ **关键认知**：`beforeEach` 是全局前置守卫，return 一个**路由位置**即"跳转拦截"，return `true`/`undefined` 即"放行"。⚠ 登录鉴权几乎是它的唯一高频用途——判断 `to.meta.requiresAuth`（路由元信息标志）后决定放行还是重定向。

## 3.2 守卫全家桶

| 守卫 | 位置 | 用途 |
|------|------|------|
| `beforeEach` | 全局 | 登录鉴权、全局拦截 |
| `afterEach` | 全局 | 导航后动作（埋点、改标题） |
| `beforeEnter` | 单个路由 | 路由级拦截 |
| `beforeRouteEnter/Update` | 组件 | 组件级前置 |
| `beforeRouteLeave` | 组件 | 离开确认（未保存拦截） |

★ **关键认知**：守卫按**优先级链**执行：全局 beforeEach → 路由 beforeEnter → 组件 beforeRouteEnter。⚠ 组合式 API 里组件内守卫用 `onBeforeRouteUpdate` / `onBeforeRouteLeave`（不能在 `<script setup>` 外用），对应选项式的 `beforeRouteUpdate`。

# 4. 懒加载与代码分割

## 4.1 动态 import

```typescript
const routes = [
  // 静态 import：打包进主 chunk
  // import Home from '../views/Home.vue'

  // 动态 import：分包，访问到才加载
  { path: '/about', component: () => import('../views/About.vue') }
]
```

★ **关键认知**：路由组件用**动态 `import`** 即为懒加载——每个路由及其依赖被切成独立 chunk（第 13 篇见过 `About-xxx.js`），**首次访问该路由时才下载**。⚠ 首屏路由（如 `Home`）不必懒加载，其余全部懒加载是 SPA 提速的标配。

## 4.2 好处与示意

```
首屏：只加载 index.js（含 Home）
   │  用户点"关于"
   ▼
动态请求 About-8fKq7n3m.js → 渲染关于页
```

★ **关键认知**：懒加载把"一次下载全部页面"变成"按需下载"，**显著降低首屏 JS 体积**，是路由层面最重要的性能手段之一（与第 22 篇的性能优化呼应）。

# 5. 组合式 API 访问

```vue
<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()       // 当前路由信息（只读）
const router = useRouter()     // 路由器实例（操作跳转）

function go() {
  router.push('/about')                    // 跳转
  router.push({ name: 'user', params: { id: '1' } })  // 命名路由跳转
  router.replace('/about')                 // 替换（不留历史）
  router.go(-1)                            // 前进/后退
}
</script>
```

★ **关键认知**：组合式 API 用 `useRoute()`（读）、`useRouter()`（写）替代选项式的 `this.$route` / `this.$router`。⚠ 传参三式——`query`（`?a=1`，显示在 URL、会随刷新保留）与 `params`（`/user/1`，路径参数）语义不同，命名路由用 `name` 跳转更健壮（不受路径改动影响）。

# 6. 本篇小结与下一步

★ **关键认知**：路由 = `path → 组件` 映射，`<RouterView>` 渲染出口；动态参数 `:id`、嵌套 `children`；鉴权用 `beforeEach`；性能靠**动态 import 懒加载**。读用 `useRoute`、写用 `useRouter`。

下一步进入**阶段四 · 原理与高级**，第 19 篇：响应式系统原理——掀开 Proxy 的盖子，看 effect / track / trigger 到底怎么运转。