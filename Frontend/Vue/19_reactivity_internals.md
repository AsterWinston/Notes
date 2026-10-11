第 3 篇建立了一个直觉：ref/reactive 把数据变成"被盯梢"的代理。本篇掀开那张盖子，讲清响应式系统最核心的三元组——**effect（要执行的工作）、track（收集依赖）、trigger（触发依赖）**——以及 scheduler 如何让多次变更"合并成一次更新"。读完你能手写一个最小响应式系统，也能秒懂"为什么这里没更新""为什么会无限循环"这类疑难杂症。

# 1. 三个核心角色

## 1.1 effect：会被重新执行的工作

```
    effect(fn)          ── 立即执行 fn，并记录"当前 effect = fn"
         │
   fn 里读了 state.count   ── 触发 Proxy 的 get 拦截
         │
   get 里 track(state, 'count')   ── 把当前 effect 记进 count 的依赖集合
         │
   state.count 被改     ── 触发 Proxy 的 set 拦截
         │
   set 里 trigger(state, 'count')  ── 取出依赖集合，全部重新执行
```

★ **关键认知**：响应式就是**"读的时候记住谁在读，写的时候把读过的都叫醒"**。那个"被叫醒重新执行的工作单元"就是 **effect**——组件渲染、`computed`、`watch` 的底层都是 effect。这就是为什么"改数据 → 视图自动更新"：**视图渲染函数本身就是一个大 effect**。

## 1.2 track 与 trigger 各司其职

```typescript
// 简化的依赖存储：target → (key → 依赖集合)
const targetMap = new WeakMap<object, Map<PropertyKey, Set<Function>>>()

function track(target: object, key: PropertyKey) {
  if (!activeEffect) return                       // 不在 effect 里读，不收集
  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))
  let deps = depsMap.get(key)
  if (!deps) depsMap.set(key, (deps = new Set()))
  deps.add(activeEffect)                           // 记录"当前 effect 依赖这个 key"
}

function trigger(target: object, key: PropertyKey) {
  const deps = targetMap.get(target)?.get(key)
  deps?.forEach(fn => fn())                        // 把依赖了 key 的 effect 全跑一遍
}
```

★ **关键认知**：`track` 在 **get** 里收集"谁读了这个 key"，`trigger` 在 **set** 里执行"读过这个 key 的 effect"。⚠ 存储结构是**三层**：`WeakMap<对象, Map<属性, Set<effect>>>`——精确到"某个对象的某个属性"，这也是为什么 Vue 的更新粒度能细到**组件级**（而不是全局重渲染）。

# 2. effect 的完整实现

## 2.1 一个最小可运行版本

```typescript
let activeEffect: Function | null = null

function effect(fn: Function) {
  const run = () => {
    activeEffect = run            // 标记"当前正在执行的 effect"
    fn()                          // 执行时触发的 get 会收集到 run
    activeEffect = null
  }
  run()                           // 立即执行一次，完成首次收集
  return run
}

function reactive<T extends object>(obj: T): T {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key)          // 读：收集依赖
      return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
      const old = Reflect.get(target, key, receiver)
      const ok = Reflect.set(target, key, value, receiver)
      if (old !== value) trigger(target, key)   // 写：值真变了才触发
      return ok
    }
  })
}
```

```typescript
// 使用效果
const state = reactive({ count: 0 })
effect(() => console.log(state.count))   // 打印 0
state.count = 1                           // 触发 → 打印 1
```

★ **关键认知**：**核心就这 20 行**——effect 用一个全局 `activeEffect` 变量标记"当前执行者"，get 里把它收进依赖，set 里把它跑出来。⚠ 两个细节决定正确性：① `set` 里要**比较新旧值**，值没变就不触发（防止无效更新与死循环）；② 嵌套 effect（一个 effect 里触发另一个）需要**栈**而非单变量，Vue 用 `effectStack` 处理。

## 2.2 依赖收集的精确性

```typescript
const state = reactive({ ok: true, count: 0 })

effect(() => {
  // 这个 effect 只在 ok 为 true 时才读 count
  console.log(state.ok ? state.count : '无')
})
// 打印 0

state.ok = false           // 触发（读了 ok）
// 之后 effect 不再读 count，依赖被"清掉"重新收集
state.count = 100          // 不再触发（count 已不在依赖里）
```

★ **关键认知**：Vue 的依赖收集是**动态且会自动清理**的——每次 effect 执行前先清掉旧依赖，执行时根据**本次实际读到的**重新收集。这保证依赖集合**始终反映真实的读取路径**，避免"已经不再用 count 却还因 count 触发"的冗余更新。⚠ 这也意味着 effect 里有分支时，只有**当前分支读到的**数据才会触发它——极小但极关键的正确性保障。

# 3. scheduler 与批量更新

## 3.1 加入调度器

```typescript
type EffectFn = { fn: Function; scheduler?: (fn: EffectFn) => void }

function trigger(target: object, key: PropertyKey) {
  const deps = targetMap.get(target)?.get(key)
  deps?.forEach(effect => {
    // 有 scheduler 就走调度器，否则直接执行
    effect.scheduler ? effect.scheduler(effect) : effect()
  })
}
```

★ **关键认知**：trigger 时**每个 effect 可以带一个 scheduler**——有 scheduler 就不立刻执行，而是把执行**交给调度器排队**。这为"异步批量"与"computed 惰性求值"提供了插槽：组件渲染 effect 的 scheduler 就是"把更新任务塞进微任务队列，合并后统一执行"。

## 3.2 批量更新的来龙去脉

```
count.value = 1      ┐
count.value = 2      ├─ 同步连改三次，每次都 trigger
count.value = 3      ┘
        │
        ▼  （scheduler 只做"标记组件待更新"，不立即渲染）
   同一个组件被标记 3 次，但只有一次真正进入微任务队列
        │
        ▼  （下一个微任务）
   统一渲染一次，DOM 上 count 直接到 3
```

★ **关键认知**：第 4、10 篇反复出现的"更新是异步批量"，根源就在这——**scheduler 不立即渲染，而是把组件加入一个待更新队列，同一组件多次变更只入队一次**，微任务里统一 flush。⚠ 这解释了两个现象：`count++` 三连写只渲染一次；改完数据**同步**读 DOM 是旧值、要 `await nextTick()`。

# 4. computed / watch 的 effect 复用

## 4.1 computed 是带缓存的 effect

```typescript
function computed<T>(getter: () => T) {
  let dirty = true
  let value: T

  const effect = reactiveEffect(getter, {
    scheduler: () => { dirty = true }   // 依赖变了只标记"脏了"，不立即求值
  })

  return {
    get value() {
      if (dirty) value = effect.run()   // 只有脏了才重新算
      dirty = false
      return value
    }
  }
}
```

★ **关键认知**：`computed` 的缓存（第 4 篇讲过）就来自这个设计——**getter 包成 effect，scheduler 只把 `dirty` 置真**；访问 `value` 时若 `dirty` 才重算，否则返回缓存。这同时实现了"依赖变了不求值、读到了才求值（惰性）"和"依赖没变不重算（缓存）"。⚠ 正是"依赖变了、值没被读"时它才始终脏着，读一次就干净——这就是 computed 缓存语义的完整机制。

## 4.2 watch 是带回调的 effect

```typescript
function watch(source: () => any, cb: (n: any, o: any) => void) {
  let oldValue: any
  const effect = reactiveEffect(source, {
    scheduler: () => {
      const newValue = source()
      cb(newValue, oldValue)        // 依赖变化时执行回调
      oldValue = newValue
    }
  })
  oldValue = effect.run()           // 首次执行拿到初始值
}
```

★ **关键认知**：`watch` 的 getter 包成 effect，**scheduler 里执行用户回调**，所以"数据变了才触发、不立即触发"（默认惰性）。⚠ 这里能清晰看到 `watch` 与 `watchEffect` 的差异：`watch` 显式给 source、scheduler 里调 cb；`watchEffect` 直接把用户函数当 effect（立即执行 + 自动收集）。

# 5. 常见陷阱定位

## 5.1 为什么"没更新"

```typescript
import { reactive } from 'vue'

let state = reactive({ count: 0 })
state = { count: 1 }        // ✗ 整体替换，代理丢了（第 3 篇 2.3）

const { count } = state     // ✗ 解构丢响应性（第 3 篇 2.3）

state.newProp = 1           // ✗ Vue2 时代不行；Vue3 Proxy 可以，但要看是否被 track
```

★ **关键认知**：所有"为什么没更新"最终都归因到**依赖追踪链断了**三类——**替换了代理对象本身**、**解构/传值断了引用**、**读了未 track 的位置**。掌握本篇的 track/trigger 模型，你就能从"哪一步没被 track"反推排查，而不是瞎猜。

## 5.2 为什么会"无限循环"

```typescript
const state = reactive({ count: 0 })

// ⚠ 自己改自己依赖的：读又写，写又触发读，死循环
watchEffect(() => {
  state.count++   // 读了 count，又改 count → 改触发 effect → 又改 → 无限
})
```

★ **关键认知**：effect 里**写了自己依赖的数据**会形成"读 → 写 → 触发 → 再读"的自触发环。Vue 有安全阀（`activeEffect` 置空后才 trigger、`allowRecurse` 标记），但根本的解法是**别在 effect/computed/watchEffect 里无保护地修改它依赖的值**——需要"改但别触发"时会用到本篇讲的"暂时解除依赖"技巧（对应 API 层第 22 篇的 `markRaw` / 手动 `effectScope`）。

# 6. 本篇小结与下一步

★ **关键认知**：响应式 = **effect（工作单元）+ track（收集依赖）+ trigger（触发依赖）+ scheduler（合并调度）**。effect 是渲染/计算/侦听的公共底座，scheduler 是异步批量的来源。看懂这套，Vue 里"何时更新、为何不更新"再无秘密。

下一步第 20 篇：虚拟 DOM 与 diff——数据变了之后，框架如何"最小化地"更新真实 DOM。