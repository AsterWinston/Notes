组合式 API 最大的收益不是语法，而是**逻辑复用**——把一组数据、方法、副作用封装成普通函数 `useXxx`，任何组件一行引入即用。本篇讲清组合式函数的写法与命名规范、它和组件生命周期的关系、异步状态处理的惯用模式，以及它相对 React Hooks / mixin 的优劣。

# 1. 什么是组合式函数

## 1.1 一个最小组合式函数

```typescript
// src/composables/useMouse.ts
import { ref, onMounted, onBeforeUnmount } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(e: MouseEvent) {
    x.value = e.clientX
    y.value = e.clientY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onBeforeUnmount(() => window.removeEventListener('mousemove', update))

  return { x, y }        // 返回的是 ref，保留响应性
}

// 使用处
// const { x, y } = useMouse()
```

★ **关键认知**：组合式函数就是**返回响应式状态 + 方法的普通函数**，命名必须 `use` 开头。它内部可以用 `ref`、`onMounted` 等一切响应式/生命周期 API——因为这些 API **必须在组件的"setup 作用域"里调用**，而组合式函数恰好是在 setup 里被调用的，所以天然处于正确的作用域内。

## 1.2 命名与组织规范

```typescript
// 规范约定：
// 1. 文件名 useXxx.ts，函数名 useXxx
// 2. 返回值是 ref 组成的对象，方便解构仍保留响应性
// 3. 输入是显式参数（不要依赖隐式全局）

export function useFetch<T>(url: string) {
  // 参数显式传入，而不是藏在闭包外
}
```

★ **关键认知**：组合式函数的**输入输出约定**是它与 Hooks 一致的核心设计——**输入参数、输出 ref 对象**。⚠ 返回值若直接 return 单个 `ref`（而非包进对象），调用处解构（`const x = useX()`）不会丢响应性（ref 本身是对象）；但 return 多个值必须用对象/数组包裹，否则解构会退化成普通值（见第 3 篇 `toRefs` 那一节）。

# 2. 从组件中抽取逻辑

## 2.1 抽取前：逻辑堆在组件里

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

// 获取用户数据的逻辑和 UI 混在一起
const user = ref(null)
const loading = ref(false)
const error = ref('')
async function load() {
  loading.value = true
  try {
    user.value = await fetch('/api/user').then(r => r.json())
  } catch (e) {
    error.value = String(e)
  } finally {
    loading.value = false
  }
}
onMounted(load)
</script>
```

## 2.2 抽取后：逻辑独立可复用

```typescript
// src/composables/useFetch.ts
import { ref } from 'vue'

export function useFetch<T>(url: string) {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref('')

  async function execute() {
    loading.value = true
    error.value = ''
    try {
      data.value = await (await fetch(url)).json()
    } catch (e) {
      error.value = String(e)
    } finally {
      loading.value = false
    }
  }

  return { data, loading, error, execute }
}
```

```vue
<!-- 组件变得一目了然 -->
<script setup lang="ts">
import { onMounted } from 'vue'
import { useFetch } from './composables/useFetch'

const { data: user, loading, error, execute } = useFetch<User>('/api/user')
onMounted(execute)
</script>
```

★ **关键认知**：抽取的本质是**把"业务逻辑"从"渲染"里剥离**——组件只留"用数据干什么"，数据怎么来、状态怎么流转交给 `useFetch`。脚本改动后逻辑可被任意组件复用，也让单测变得简单（直接测函数，无需挂载组件）。

# 3. 异步状态与并发

## 3.1 完整异步模式

```typescript
export function useTodo() {
  const list = ref<Todo[]>([])
  const loading = ref(false)
  const error = ref('')

  const load = async () => { /* 如上 */ }
  const add = async (text: string) => { /* POST + 刷新列表 */ }
  const remove = async (id: number) => { /* DELETE + 刷新列表 */ }

  return { list, loading, error, load, add, remove }
}
```

★ **关键认知**：组合式函数封装异步时，关键在于**锁住内部状态**——`loading`/`error` 不泄漏到组件，调用方只关心 `list` 和操作方法。⚠ 多个组件同时调用同一组合式函数，**状态是各自独立的**（每次调用都新建 ref），不像以前的 mixin 共享全局。

## 3.2 处理竞态：响应过早返回

```typescript
import { ref, watch } from 'vue'

export function useSearch(keyword: Ref<string>) {
  const results = ref([])

  watch(keyword, async (kw, _old, onCleanup) => {
    let cancelled = false
    onCleanup(() => { cancelled = true })   // 旧的请求被新的覆盖时置位

    const data = await fetch(`/api/search?q=${kw}`).then(r => r.json())
    if (!cancelled) results.value = data    // 只有最新请求才落地
  })

  return { results }
}
```

★ **关键认知**：异步竞态是搜索自动补全的经典坑——用户快速输入，"旧的慢请求"可能比"新的快请求"更晚返回，把过期结果覆盖成当前。解法是 `watch` 的 **`onCleanup`**（新触发前先标记旧请求作废），返回结果前检查 `cancelled` 标志。⚠ 这类"取消过期请求"在 React 里要用 `useEffect` + 清理函数，Vue 的 `watch` 内置了同样的机制，写法更直接。

# 4. 与 mixin / React Hooks 对比

## 4.1 组合式函数 vs mixin

| 维度 | mixin（Vue 2） | 组合式函数（Vue 3） |
|------|---------------|--------------------|
| 命名冲突 | 易冲突（合并后同名互相覆盖） | 无（返回对象可重命名） |
| 数据来源 | 隐式合并，来源不明 | 显式 return，来源清晰 |
| 依赖关系 | 隐藏、难追溯 | 显式传参，一目了然 |
| 类型支持 | 弱 | 强（普通 TS 函数） |

★ **关键认知**：mixin 的消亡源于"**隐式合并**"——两个 mixin 同名属性会悄悄覆盖，数据来源无法追踪。组合式函数把一切**显式化**（显式入参、显式返回、显式命名），换来可维护性与类型安全。这是 Vue 3 弃 mixin、转组合式的根本原因。

## 4.2 组合式函数 vs React Hooks

| 维度 | Vue 组合式函数 | React Hooks |
|------|---------------|-------------|
| 依赖追踪 | 自动（Proxy） | 手动（依赖数组） |
| 执行次数 | setup 里一次 | 每次渲染都执行 |
| 依赖遗漏风险 | 无 | 有（闭包旧值） |
| 记忆化 | computed 自动缓存 | useMemo/useCallback 手动 |

★ **关键认知**：两者思想同源（封装逻辑、`use` 开头、复用时新建状态），但**执行模型不同**——React Hooks 每次渲染都重跑，靠依赖数组决定要不要更新；Vue 组合式函数只跑一次，靠 Proxy 精确追踪。因此 Vue **没有** React 里"依赖数组漏写导致拿到旧值"这类心智负担，也**不需要** `useCallback`/`useMemo` 这些记忆化手段。

# 5. 本篇小结与下一步

★ **关键认知**：组合式函数 = **输入参数 + 输出 ref 对象的普通函数**，是组合式 API 复用逻辑的载体。它靠"显式"战胜 mixin，靠"自动追踪"轻于 React Hooks。

下一步第 12 篇：provide / inject——不靠逐层 props 传递，实现跨层级的依赖注入。