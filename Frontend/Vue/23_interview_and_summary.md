本篇是全系列的收官：先把 Vue 3 面试最高频的 10 道题按"要点 + 示例"过一遍，再汇总常考的手写题思路，最后画出 23 篇的知识地图与进阶路线。无论备战面试还是系统复盘，这篇都值得收藏。

# 1. 高频面试题 10 道

## 1.1 ref 与 reactive 的区别

**要点**：`ref` 用 `.value` 包装任意值（适合基本类型、替换整体）；`reactive` 用 Proxy 代理对象（仅对象、免 `.value`、深响应）。`ref` 装对象时内部转 `reactive`。

```typescript
import { ref, reactive } from 'vue'

const n = ref(0)                     // Ref<number>，读 n.value
const state = reactive({ count: 0 }) // 直接 state.count，深响应
state.count++                         // ✓ 免 .value
// reactive(1)                       // ✗ reactive 不接受基本类型
```

★ **关键认知**：选择看"值还是对象、要不要整体替换"；解构都丢响应性，要保留用 `toRefs`（第 3 篇）。

## 1.2 computed 与 watch 的区别

**要点**：`computed` 是**缓存派生值**（依赖不变不重算、有返回值）；`watch` 是**变化驱动副作用**（不产生新值、可拿新旧值）。

```typescript
import { ref, computed, watch } from 'vue'
const a = ref(1)
const double = computed(() => a.value * 2)   // 派生值，有缓存
watch(a, (n, o) => console.log(o, '→', n))   // 副作用，不返回值
```

★ **关键认知**：派生用 computed、副作用用 watch；别把发请求写进 computed（第 4 篇）。

## 1.3 v-if 与 v-show 的区别

**要点**：`v-if` 条件**挂载**（DOM 增删，有创建/销毁开销）；`v-show` 条件**显示**（`display:none`，切换成本低）。切换频繁用 v-show，少见用 v-if。

★ **关键认知**：v-if 配 `v-else`，v-show 无；v-if 会触发组件的生命周期反复创建/销毁（第 5 篇）。

## 1.4 为什么 v-for 要有 key，为什么不用 index 当 key

**要点**：`key` 是 diff 判断"新旧哪个节点是同一个"的**身份标识**，有稳定 key 才能**复用** DOM 与组件状态；`index` 作 key 时插入/删除会"原地复用错位"（如 input 内容串行）。

```vue
<div v-for="u in users" :key="u.id">{{ u.name }}</div>  <!-- 用业务 id -->
```

★ **关键认知**：key 的复用机制见第 20 篇双端 diff；index 只适用于纯静态永不变序的列表（第 5、20 篇）。

## 1.5 为什么说 Vue 3 用 Proxy 比 Vue 2 的 defineProperty 好

**要点**：`Object.defineProperty` 只能拦截**已存在属性**（新增属性、删属性、数组下标/`length` 都拦不住，要 `Vue.set`）；`Proxy` 拦得住**任意属性读写、新增、删除、`keys`**，且直接支持数组下标。

★ **关键认知**：Proxy 解决了 Vue 2 的"新增属性不响应""数组下标失效"两大经典坑（第 19 篇）。

## 1.6 响应式的核心流程

**要点**：effect（工作单元）+ track（读时收集依赖）+ trigger（写时触发依赖）+ scheduler（异步批量合并更新）。

```
读 state.count → get 拦截 → track 收集当前 effect
写 state.count → set 拦截 → trigger 执行依赖的 effect（经 scheduler 排队合并）
```

★ **关键认知**：组件渲染、computed、watch 底层都是 effect；"异步批量更新"来自 scheduler 把更新合并进微任务队列（第 19 篇）。

## 1.7 nextTick 的作用

**要点**：Vue 更新是**异步批量**的，同步改完数据 DOM 未更新；`await nextTick()` 保证"本次渲染已落到 DOM"后再操作。

```typescript
count.value++
await nextTick()               // 此刻 DOM 已是最新
console.log(el.value?.textContent)
```

★ **关键认知**：nextTick ≠ 下一次事件循环，只保证"这一轮 Vue 渲染任务完成"（第 10 篇）。

## 1.8 单向数据流是什么，为什么

**要点**：props 只读向下流，子组件不能改 prop，想改变就 `emit` 通知父组件改。保证**状态来源单一可追溯**，避免"谁改的"无从排查。

★ **关键认知**：props down / emit up 是 Vue/React 组件体系的共同铁律；需要双向同步用 `defineModel`（第 7、8 篇）。

## 1.9 组合式 API 相比选项式 API 的优势

**要点**：按**逻辑关注点**组织（而非按 data/methods 类别拆散）、逻辑可**复用**（组合式函数）、**类型友好**、**数据来源显式**（优于 mixin 的隐式合并）。

★ **关键认知**：核心收益是"逻辑复用 + 显式追踪"；组合式函数还能脱离组件使用（第 11 篇）。

## 1.10 Vue 的编译期优化有哪些

**要点**：**静态提升**（静态节点只创建一次）、**PatchFlags**（只比动态字段）、**Block tree**（只遍历动态节点）——更新成本从"模板规模"降到"动态绑定数量"。

★ **关键认知**：模板的"表达力受限"换来"可静态分析"，这就是 Vue 敢用模板且性能不弱的根本（第 21 篇）。

# 2. 手写题思路清单

| 题目 | 核心考点 |
|------|---------|
| 手写响应式 `reactive` | Proxy + Reflect + track/trigger |
| 手写 `ref` | 对象壳 `{ value }` + 依赖追踪 |
| 手写 `computed` | 包装 effect + dirty 标志缓存 |
| 手写 `watch` | getter effect + scheduler 里调回调 |
| 手写简单 diff | 双端比较 + key 复用 |

```typescript
// 手写 reactive 的骨架（面试常考）
function reactive<T extends object>(obj: T): T {
  const deps = new Map<PropertyKey, Set<Function>>()
  let active: Function | null = null
  function effect(fn: Function) {
    active = fn
    fn()
    active = null
  }
  function track(key: PropertyKey) {
    if (!active) return
    if (!deps.has(key)) deps.set(key, new Set())
    deps.get(key)!.add(active)
  }
  function trigger(key: PropertyKey) {
    deps.get(key)?.forEach(fn => fn())
  }
  return new Proxy(obj, {
    get(target, key, recv) {
      track(key as PropertyKey)
      return Reflect.get(target, key, recv)
    },
    set(target, key, val, recv) {
      Reflect.set(target, key, val, recv)
      trigger(key as PropertyKey)
      return true
    }
  })
}
```

★ **关键认知**：手写题的价值不在背诵，而在**证明你懂机制**——能说出"effect 用全局变量标记当前执行者、get 里收集、set 里触发、Reflect 传 receiver 保 this"这几个点，比复述代码更能拿分。

# 3. 全系列知识地图

```
               Vue 3 学习路线（23 篇 · 四阶段）
 ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
 │阶段一 基础  │ → │阶段二 组件  │ → │阶段三 工程  │ → │阶段四 原理  │
 │ 1 简介响应式│   │ 7 组件基础 │   │ 13 SFC构建 │   │ 19 响应式内核│
 │ 2 应用模板 │   │ 8 组件通信 │   │ 14 样式方案 │   │ 20 vdom/diff │
 │ 3 响应式基础│   │ 9 插槽     │   │ 15 过渡动画 │   │ 21 编译渲染 │
 │ 4 计算侦听 │   │ 10 生命周期│   │ 16 TS集成   │   │ 22 性能优化 │
 │ 5 条件列表 │   │ 11 组合函数│   │ 17 Pinia    │   │ 23 面试总结 │
 │ 6 事件表单 │   │ 12 依赖注入│   │ 18 Router   │   │             │
 └────────────┘   └────────────┘   └────────────┘   └────────────┘
  声明式 + 响应式    组件化与复用      真实项目落地      内核与性能
```

| 阶段 | 篇目 | 核心一句话 |
|------|------|-----------|
| 一 基础 | 1~6 | 声明式渲染、响应式数据、计算与侦听、条件列表、事件表单 |
| 二 组件 | 7~12 | 组件通信、插槽、生命周期、组合式函数、依赖注入 |
| 三 工程化 | 13~18 | SFC 构建、样式、动画、TS、Pinia、Router |
| 四 原理 | 19~23 | 响应式内核、vdom/diff、编译、性能、面试 |

★ **关键认知**：四阶段的暗线是"**从会用 → 会组织 → 会工程 → 懂内核**"。与 React 对照，Vue 的深水区在**响应式系统 + 编译优化**，React 的深水区在**渲染模型 + Hooks 心智**——两门框架"看得见"的 API 都简单，"看不见"的那层机制才是分水岭。

# 4. 进阶方向

| 方向 | 做法 | 收益 |
|------|------|------|
| 官方文档精读 | 通读 vuejs.org 的 API 与最佳实践 | 补全细节盲区 |
| 源码浅读 | 读 `@vue/reactivity` 与 `runtime-core` | 印证响应式与 patch 机制 |
| SSR / Nuxt | 学 Nuxt 3，掌握服务端渲染 | 覆盖 SEO 与大流量场景 |
| 测试 | Vitest + Vue Test Utils 写组件测试 | 保障重构安全 |
| 状态与工程 | Pinia 规模化组织、monorepo | 大型项目协同 |
| 性能实战 | 打包分析 + 长列表虚拟滚动 | 深水区优化 |

★ **关键认知（全系列收束）**：Vue 3 的本质是**一套"声明式描述 + 响应式追踪 + 编译期优化"的渲染框架**——你描述数据与视图的依赖关系，Proxy 负责追踪，编译器负责把模板翻译成高效 render，patch 负责最小化落 DOM。把 23 篇浓缩成一句：**用响应式数据表达状态，用 props/emit 划清组件边界，用 computed 派生、用 watch 副作用，用 key 保复用、用编译优化保性能**——吃透这条主线，你已经越过大多数 Vue 使用者的水平线。