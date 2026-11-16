props 逐层传递数据，当层级变深（祖孙之间、跨多级组件）时，中间组件被迫"搬运"自己根本不用的 props，这叫 **prop drilling**。provide / inject 正是解法——祖先"供给"数据，任意后代"注入"使用，中间层零参与。本篇讲清 provide/inject 的用法、响应式传输的关键、Symbol 键与全局注入。

# 1. provide / inject 基础

## 1.1 跨层传递

```vue
<!-- 祖先组件 -->
<script setup lang="ts">
import { provide } from 'vue'
provide('theme', 'dark')       // 供给一个值，key 是字符串 'theme'
</script>
```

```vue
<!-- 任意深度的后代组件 -->
<script setup lang="ts">
import { inject } from 'vue'
const theme = inject('theme')  // 注入，拿到的 'dark'
</script>
```

★ **关键认知**：`provide(key, value)` 在祖先"供给"，`inject(key)` 在**任意直接/间接后代**"取用"，中间组件完全不用转发。⚠ 二者的匹配靠 **key 字符串完全相等**，拼错 key 拿到的就是 `undefined`，且**没有任何报错**——这是 provide/inject 首位的坑。

## 1.2 默认值与兜底

```typescript
import { inject } from 'vue'

// 第二参数：找不到 key 时的默认值
const theme = inject('theme', 'light')

// 第三参数（true）：把该 key 声明为必填，找不到就在开发模式报警
const required = inject('requiredKey', undefined, true)
```

★ **关键认知**：`inject` 支持**默认值**与**必填标记**——默认值挡住"拼错 key 拿到 undefined"，必填标记在开发模式显式报警。⚠ 生产环境不会校验，所以不要把"必填 inject"当成可靠防线，关键路径仍需运行时兜底。

# 2. 响应式传输：关键中的关键

## 2.1 只传值 vs 传响应式对象

```vue
<!-- 祖先 -->
<script setup lang="ts">
import { provide, ref, reactive, readonly } from 'vue'

const count = ref(0)
const state = reactive({ count: 0 })

// 方案 A：provide 一个 ref —— 后代拿到的就是同一个 ref
provide('count', count)

// 方案 B：provide 一个 reactive 对象 —— 后代拿到同一代理
provide('state', state)

// 方案 C：provide 只读包装，禁止后代篡改（推荐）
provide('count', readonly(count))
</script>
```

```vue
<!-- 后代 -->
<script setup lang="ts">
import { inject } from 'vue'
const count = inject<Ref<number>>('count')
// count!.value++   // 若祖先传 readonly，这里会报警
</script>
```

★ **关键认知**：provide/inject 的响应式**取决于你传的是不是响应式对象**——传 `ref`/`reactive`，后代对它的读写都走同一个代理，自然是响应式的；传普通值则只会拿一个"当时"的快照。⚠ 想要"祖先后代共享可变化的状态"，就必须 **provide 一个 ref/reactive**。传 `ref`，后代需要 `.value`；传 `reactive`，后代直接读属性。

## 2.2 修改逻辑放一起，别让后代乱改

```vue
<!-- 祖先：把"值 + 改值的方法"一起 provide，形成封闭 API -->
<script setup lang="ts">
import { provide, ref, readonly } from 'vue'

const count = ref(0)
function increment() { count.value++ }

provide('count', readonly(count))     // 只读值
provide('increment', increment)       // 修改方法
</script>
```

★ **关键认知**：最佳实践是"**值只读、修改靠方法**"——祖先 `provide` 时用 `readonly()` 包住值，同时把修改方法一并 `provide`。这样数据流仍是清晰的单向（谁能改、怎么改都在祖先这一处定义），后代拿到只读视图 + 受控方法，不易失控。⚠ 默认别让后代直接 `inject` 一个可写的 ref 然后随意改，那等于又开了一条混乱的全局通道。

# 3. Symbol 键：避免命名冲突

## 3.1 key 用 Symbol

```typescript
// injectionKeys.ts
import type { InjectionKey } from 'vue'
import type { Ref } from 'vue'

// InjectionKey 携带类型，inject 时类型自动推断
export const countKey: InjectionKey<Ref<number>> = Symbol('count')
```

```vue
<!-- 祖先 -->
<script setup lang="ts">
import { provide } from 'vue'
import { countKey } from './injectionKeys'
provide(countKey, ref(0))
</script>

<!-- 后代 -->
<script setup lang="ts">
import { inject } from 'vue'
import { countKey } from './injectionKeys'
const count = inject(countKey)   // 类型自动是 Ref<number> | undefined
</script>
```

★ **关键认知**：字符串 key 在大型项目里**易撞名且无类型**，`Symbol` + `InjectionKey<T>` 解决两者——唯一性 + 类型推导。⚠ 中型及以上项目，跨文件的 provide/inject 几乎都应走 `InjectionKey` + Symbol，把 key 集中到一个文件导出。

## 3.2 字符串 key 的应用边界

★ **关键认知**：字符串 key 只适合**同一文件内的局部注入**（组件自己的子组件用）。一旦跨文件、跨模块，就升级成 Symbol key。这是从"能用"到"可维护"的分水岭。

# 4. 全局注入 app.provide

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.provide('apiClient', apiClient)   // 整个应用都能 inject 到
app.mount('#app')
```

★ **关键认知**：`app.provide` 在**应用实例**层面注入，全应用任意组件都可 `inject`。这是"全局配置"（API 客户端、主题上下文、语言包）的推荐落点——比把东西挂 `window` 上更干净、更有边界。⚠ 它同样遵循"传响应式对象才响应式"的规则。

# 5. provide/inject 的定位与边界

| 手段 | 层级 | 何时用 |
|------|------|--------|
| props/emit | 父子 | 直接通信首选 |
| provide/inject | 跨多级 | 深度组件共享、主题/配置上下文 |
| Pinia | 全局任意 | 跨页面、跨模块共享业务状态 |

★ **关键认知**：provide/inject 适合"**一条组件树上的向下广播**"，但**不是状态管理的替代品**——它缺乏时间旅行、持久化、模块级访问这些 Pinia 的能力。需要跨路由、跨模块共享可变业务状态时，请用第 17 篇的 Pinia，别硬塞进 provide。

# 6. 本篇小结与下一步

★ **关键认知**：provide/inject 三要点——**key 精确匹配**（拼错无报警）、**传响应式对象才有响应性**、**值 readonly + 方法 provide** 守住单向数据流。它解决 prop drilling，但定位是"树内广播"，非全局状态管理。

下一步进入**阶段三 · 工程化**，第 13 篇：单文件组件与构建——`.vue` 里三块到底怎么协同、Vite 又是怎么把它跑起来的。