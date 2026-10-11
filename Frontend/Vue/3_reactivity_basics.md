响应式是 Vue 的灵魂——`ref` 与 `reactive` 把普通数据变成"被框架盯梢"的代理，数据一变，依赖它的视图自动更新。本篇讲清两种响应式数据的本质、`.value` 与自动解包的完整规则，以及 `shallowRef` / `shallowReactive` 在性能边界上的取舍；机制先建立直觉，完整源码级原理见第 19 篇。

# 1. ref 的本质

## 1.1 ref 包装单个值

```typescript
import { ref } from 'vue'

const count = ref(0)          // 返回 Ref<number>，值藏在 .value 里
console.log(count.value)      // 0
count.value = 1               // 改 .value 才触发响应式更新
console.log(count.value)      // 1

// count = 2                  // ✗ 千万别这样：直接覆盖整个 ref 会丢失响应式
```

★ **关键认知**：`ref` 解决的是"**基本类型没法被拦截**"的问题——`number` / `string` / `boolean` 是原始值，改它没有"属性被赋值"这个可监听的事件。ref 的做法是套一层**对象壳** `{ value }`，读写都经过 `value` 属性，Proxy 才能拦得住。⚠ 两条铁律：**JS 里读/写都要 `.value`**；**永远别整体覆盖 ref 变量本身**，要改就改 `.value`。

## 1.2 ref 也可以装对象

```typescript
import { ref } from 'vue'

const user = ref({ name: 'Alice' })   // ref 装对象时内部会转成 reactive
user.value.name = 'Bob'               // 深层属性也是响应式的

// 打印时 value 里是个响应式代理
console.log(user.value)               // Proxy(Object) { name: 'Bob' }
```

★ **关键认知**：`ref(对象)` 内部会调用 `reactive` 把对象转成深响应式代理。所以你写 `user.value.name = 'Bob'` 也能触发更新。⚠ 但请注意：`ref` 的监听只精确到"**替换整个 value**"，对象内部的细粒度追踪由内层 `reactive` 完成——这引出一个后续陷阱：数组/对象的深层属性通过 ref 改才是响应式的，而一些替换式写法要注意。

# 2. reactive 的本质

## 2.1 reactive 代理对象

```typescript
import { reactive } from 'vue'

const state = reactive({ count: 0, user: { name: 'Alice' } })
state.count++                  // 直接改，无需 .value
state.user.name = 'Bob'        // 深层嵌套也是代理，改动可追踪
console.log(state.count)       // 1
```

★ **关键认知**：`reactive` 直接返回 `Proxy` 包裹的对象，**读写都不用 `.value`**，因为对象属性本身就是可拦截的。与 `ref` 的对比：

| 维度 | ref | reactive |
|------|-----|----------|
| 适用 | 任意值（基本类型 / 对象） | **仅对象**（数组/对象/Map） |
| 访问 | 需要 `.value` | 直接读写属性 |
| 替换整体 | 可（改 `.value`） | ⚠ 破坏响应式（见 2.3） |
| 解构 | ⚠ 丢响应式 | ⚠ 丢响应式 |
| 一个经验 | 单个值、需替换 → ref | 复杂对象、表单数据 → reactive |

## 2.2 深响应式与 Map/Set

```typescript
import { reactive } from 'vue'

const state = reactive({
  list: [1, 2, 3],
  map: new Map<string, number>(),
  obj: { nested: { deep: 0 } }
})

state.list.push(4)             // ✓ 数组方法追踪（第 5 篇）
state.map.set('a', 1)          // ✓ Map 也被代理，set 可追踪
state.obj.nested.deep++        // ✓ 深层嵌套逐层代理
```

★ **关键认知**：`reactive` 是**深响应式**——对象里嵌套的对象、数组、`Map`/`Set` 都会被递归代理。⚠ 但 `Map`/`Set` 必须用**本身的方法**（`.set()` / `.add()`）来改才可追踪，直接 `state.map = new Map(...)` 替换整体会丢失响应式（同下）。

## 2.3 reactive 的两个致命坑

```typescript
import { reactive } from 'vue'

let state = reactive({ count: 0 })

// ===== 坑一：整体替换 =====
state = { count: 1 }   // ✗ 不通！state 重新指向了普通对象，跟代理没关系了

// ===== 坑二：解构丢响应式 =====
const { count } = state   // count 被解构成普通 number，改它不会更新
count++                   // ✗ 无效

// 正确：用 toRef / toRefs 保留链接
import { toRefs } from 'vue'
const { count: countRef } = toRefs(state)
countRef.value++          // ✓ 通过 Ref 保留与源对象的链接
```

★ **关键认知**：`reactive` 的代理是**创建那一刻的对象**。整体替换等于把变量指向全新对象，代理丢在半路；解构则把值"抄"出来，等于断了追踪链。⚠ 这两个坑是 Vue 新手翻车最多的点，`toRefs` 是官方给的解构方案（把属性变成保留链接的 `Ref`）。想在组件里返回解构的数据，几乎总要用 `toRefs`。

# 3. 自动解包规则

## 3.1 模板里的解包

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(1)
const obj = { count: ref(2) }
</script>

<template>
  <p>{{ count }}</p>        <!-- 1：顶层 ref 自动解包，无需 .value -->
  <p>{{ count + 1 }}</p>    <!-- 2 -->
  <p>{{ obj.count }}</p>    <!-- ✗ 嵌套在普通对象里的 ref 模板不解包，显示的是 Ref 对象本身 -->
  <p>{{ obj.count + 1 }}</p> <!-- ✗ 结果变成 [object Object]1，因为拿到的还是对象 -->
</template>
```

★ **关键认知**：模板里**只有"顶层" ref 自动解包**——直接写在 `<script setup>` 里的 `ref` 变量会被解包，但**普通对象属性里的 ref 不会**（要读须写 `obj.count.value`）。⚠ 一个常见翻车：把 `ref` 塞进普通对象再在模板里读，发现渲染出 `[object Object]`。顶层 ref 少用对象包一层，或干脆用 `reactive`（reactive 内会自动解包 ref，见 3.2）。

## 3.2 reactive 里的 ref 解包

```typescript
import { reactive, ref } from 'vue'

const count = ref(1)
const obj = reactive({ count })   // reactive 对象里的 ref 会被"吸平"
console.log(obj.count)            // 1（自动解包，不是 Ref 对象）

// 给 reactive 对象的 ref 属性赋新值，会同时更新外层 ref
const inner = ref(2)
const obj2 = reactive({ n: inner })
obj2.n = 5                        // 等价于 inner.value = 5
```

★ **关键认知**：`reactive` 对象里的 `ref` 属性会被**自动解包**——读写都像普通值。而**数组 / Map 里的 ref 不会解包**（`[ref(1)]` 的元素仍是 Ref）。这个"解包边界"是响应式体系里最微妙、最需要背下来的一套规则，面试也爱考。

# 4. shallow 家族：性能边界

## 4.1 shallowRef 与 shallowReactive

```typescript
import { shallowRef, shallowReactive, triggerRef } from 'vue'

// shallowRef：只有 .value 整体替换才触发，内部对象不深追踪
const state = shallowRef({ count: 0 })
state.value.count++          // ✗ 内部属性改动不触发更新
state.value = { count: 1 }   // ✓ 整体替换触发

// shallowReactive：只有第一层属性是响应式的
const s = shallowReactive({ nested: { count: 0 }, n: 1 })
s.n = 2                      // ✓ 第一层可追踪
s.nested.count++             // ✗ 深层不追踪

// 改完浅层数据想手动通知更新，用 triggerRef
triggerRef(state)            // 手动触发 state 的依赖
```

★ **关键认知**：`shallowRef` / `shallowReactive` 用"**放弃深层追踪**"换取性能——适合**大对象、只整体替换**的场景（如引入的第三方大配置、图表实例）。⚠ 它们是**性能优化手段，不是默认选择**：内部改了不更新会让人抓狂，只有当你明确"这个数据我只整体替换"时才用；实在混用了，`triggerRef` 手动兜底。

## 4.2 何时该选谁

| 场景 | 推荐 | 理由 |
|------|------|------|
| 计数器、开关等单个值 | `ref` | 基本类型只能 ref |
| 表单、配置对象、需逐字段改 | `reactive` | 免 `.value`，结构清晰 |
| 需要整体替换的对象 | `ref` | `.value = newObj` 才触发 |
| 大型第三方对象/图表实例 | `shallowRef` | 只整体替换，避免深代理开销 |
| 永不变化的数据 | `markRaw` / 普通对象 | 干脆别进响应式系统（第 22 篇） |

★ **关键认知**：选择的本质是**匹配"更新粒度"**——追踪越深开销越大，但语义越安全。日常 90% 场景 `ref` + `reactive` 就够，`shallow` 是到了性能瓶颈才动用的高级开关。

# 5. 深入一步：Proxy 初探

## 5.1 为什么 Proxy 能拦得住

```typescript
const raw = { count: 0 }
const proxy = new Proxy(raw, {
  get(target, key) {
    console.log('读取', key)
    return target[key]      // 读的时候记录"谁在读"
  },
  set(target, key, value) {
    console.log('写入', key, value)
    target[key] = value      // 写的时候通知"读过的人"
    return true
  }
})

proxy.count        // 打印"读取 count"
proxy.count = 1    // 打印"写入 count 1"
```

★ **关键认知**：`Proxy` 让"对象属性读写"变成了**函数调用**，框架就能在 `get` 里收集依赖、在 `set` 里触发更新。Vue 2 用的是 `Object.defineProperty`，**只能拦截已存在的属性**，所以"给对象新增属性"这种操作它感知不到，还得 `Vue.set`；Vue 3 的 Proxy 拦得住**任意属性（含新增）**和整个 `keys`，彻底解决了这个历史问题。完整链路（effect、track、trigger、scheduler）见第 19 篇。

## 5.2 Reflect 的作用

```typescript
const obj = {
  get value() { return this.count },
  count: 1
}
const proxy = new Proxy(obj, {
  get(target, key, receiver) {
    // 用 Reflect 并传 receiver，保证 getter 里的 this 指向代理而非原始对象
    return Reflect.get(target, key, receiver)
  }
})
```

★ **关键认知**：`Proxy` 配 `Reflect` 是标准搭档——`Reflect` 提供与 handler 一一对应的静态方法，并把 `this` 正确地绑定到代理上（否则 getter 里的 `this.count` 会绕过代理读原始对象，追踪失效）。⚠ 自己用 `target[key]` 而非 `Reflect.get`，是手写响应式系统最常见的隐性 bug 来源。

# 6. 本篇小结与下一步

★ **关键认知**：`ref` 靠 `.value` 壳解决基本类型的追踪，`reactive` 靠 Proxy 解决对象的追踪，二者在模板/location 里各有自动解包规则——**背熟"顶层 ref 模板解包、reactive 内 ref 解包、数组内 ref 不解包"这三条**，就能避开一半响应式坑。

下一步第 4 篇：`computed` 与 `watch`——当数据是"派生"出来的、当数据变了要"做点事"，这两个工具如何各司其职。