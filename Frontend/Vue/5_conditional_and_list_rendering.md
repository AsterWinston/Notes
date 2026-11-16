条件渲染决定"一个元素该不该出现"，列表渲染决定"一个模板该重复多少次"。这两类是模板里最高频的指令。本篇讲清 `v-if` 与 `v-show` 的本质差别、`v-for` 的写法与数组响应式方法、`key` 为何直接决定 diff 的正确性与性能，以及 `v-for` 与 `v-if` 同用时那个绕不开的陷阱。

# 1. 条件渲染 v-if 与 v-show

## 1.1 v-if：真正的条件挂载

```vue
<script setup lang="ts">
import { ref } from 'vue'
const visible = ref(true)
</script>

<template>
  <!-- v-if 为假时，元素从 DOM 中彻底移除（不是隐藏） -->
  <p v-if="visible">这段文字存在与否由 visible 决定</p>
  <p v-else-if="!visible">else-if 分支</p>
  <p v-else>else 分支</p>
</template>
```

★ **关键认知**：`v-if` 是**真正的条件渲染**——为假时元素**根本不进 DOM**（连同它内部的组件都会被销毁、重新创建）。它支持 `v-else-if` / `v-else` 链，但要求这三个指令**紧挨着**，中间不能夹别的元素。⚠ 频繁切换的显隐用 `v-if` 会反复"创建/销毁"，代价比想象中大（组件要重建、状态要重来）。

## 1.2 v-show：只是 display 切换

```vue
<script setup lang="ts">
import { ref } from 'vue'
const visible = ref(true)
</script>

<template>
  <!-- v-show 为假时，元素仍在 DOM 里，只是 display: none -->
  <p v-show="visible">只是被 CSS 藏起来了</p>
</template>
```

★ **关键认知**：`v-show` 无论真假，元素**始终在 DOM 里**，只是切换内联的 `display: none` / 移除该样式。因此它**没有"创建/销毁"开销**，切换成本极低，但初次渲染无论如何都要渲染一遍。

## 1.3 v-if vs v-show 决策

| 维度 | v-if | v-show |
|------|------|--------|
| 本质 | 条件挂载（DOM 增删） | 条件显示（display 切换） |
| 初始渲染开销 | 假时不渲染，省 | 无论如何都渲染 |
| 切换开销 | 高（重建/销毁） | 低（只切样式） |
| 适用 | 初次就少出现、切换不频繁、依赖生命周期 | 频繁切换的 tab/弹窗 |
| 无 `v-else` 配套 | 有 v-else-if/else | 无 |

★ **关键认知**：一句话——**切换频繁用 v-show，条件少见且切换成本敏感用 v-if**。tab 切换、抽屉显隐用 `v-show`；登录前后的整块布局、首屏懒加载用 `v-if`。

# 2. 列表渲染 v-for

## 2.1 基本写法

```vue
<script setup lang="ts">
import { ref } from 'vue'
const todos = ref([
  { id: 1, text: '写笔记' },
  { id: 2, text: '学 Vue' }
])
</script>

<template>
  <!-- v-for="(item, index) in list"，可带索引与解构 -->
  <ul>
    <li v-for="(todo, index) in todos" :key="todo.id">
      {{ index + 1 }}. {{ todo.text }}
    </li>
  </ul>

  <!-- 对象也能遍历：value, key, index -->
  <li v-for="(value, key, index) in user" :key="key">{{ key }}: {{ value }}</li>
</template>
```

★ **关键认知**：`v-for` 支持**数组**和**对象**。数组遍历签名是 `(item, index)`，对象是 `(value, key, index)`。⚠ 遍历对象时属性顺序按 `Object.keys()` 的结果，且**新增属性不一定触发更新**（这是响应式追踪的边界，参见第 3 篇）。

## 2.2 key 的作用

```vue
<script setup lang="ts">
import { ref } from 'vue'
const users = ref([
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
])
</script>

<template>
  <!-- key 必须是唯一且稳定的标识，别用 index 当 key -->
  <div v-for="u in users" :key="u.id">{{ u.name }}</div>
</template>
```

★ **关键认知**：`key` 是 diff 算法判断"新旧的哪个节点是同一个"的**身份标识**（第 20 篇详解）。有稳定 key，列表重排/删除时 Vue 能**复用**对应 DOM 与组件状态；没有 key 或用 `index` 作 key，重排/插入会导致**复用错位**——比如列表带了输入框组件，插一个元素后输入框里的字会"串行"。⚠ **永远不要用 index 作 key**（列表稳定且纯粹的静态展示才是唯一例外），用业务唯一 id。

# 3. 数组响应式方法

## 3.1 可追踪的变更方法

```typescript
import { ref } from 'vue'
const list = ref([1, 2, 3])

// ===== 以下"可变"方法会原地改动并被追踪，触发更新 =====
list.value.push(4)        // 末尾追加
list.value.pop()          // 删除末尾
list.value.shift()        // 删除头部
list.value.unshift(0)     // 头部插入
list.value.splice(1, 1)   // 删除/插入/替换（万能）
list.value.sort()         // 排序
list.value.reverse()      // 反转

// 给已存在的下标赋值（ref 里 value 是 reactive 数组，OK）
list.value[0] = 99        // ✓ 触发（因为 reactive 拦得住下标赋值）
```

★ **关键认知**：Vue 3 用 Proxy，数组的**下标赋值**（`arr[0] = x`）和**`length` 直接改**（`arr.length = 0`）都能被拦截追踪，这是 Vue 3 相比 Vue 2 的重大改善（Vue 2 里下标赋值必须用 `Vue.set`）。

## 3.2 不可变方法要重新赋值

```typescript
import { ref } from 'vue'
const list = ref([1, 2, 3])

// filter/map/concat/slice 返回新数组，必须重新赋给 .value
list.value = list.value.filter(n => n > 1)   // ✓ 整体替换触发

// 或配合 computed 直接生成派生列表
import { computed } from 'vue'
const big = computed(() => list.value.filter(n => n > 1))
```

★ **关键认知**：`filter` / `map` / `concat` / `slice` 这些**不可变方法返回新数组**，原地改动没意义，必须**整体替换** `list.value = ...`。也可以交给 `computed` 生成派生列表——这就是"列表需要筛选展示"时的标准姿势。

# 4. v-for 与 v-if 同用的陷阱

## 4.1 同元素上不要混用

```vue
<script setup lang="ts">
import { ref } from 'vue'
const items = ref([1, 2, 3])
</script>

<template>
  <!-- ⚠ 反例：v-if 和 v-for 在同一元素，v-if 无法访问 item，且先判断 v-if -->
  <!-- <li v-for="i in items" v-if="i > 1" :key="i">{{ i }}</li> -->
</template>
```

★ **关键认知**：`v-if` 与 `v-for` 写在**同一元素**上是经典错误——Vue 3 里 `v-if` 优先于 `v-for` 求值，于是 `v-if="i > 1"` 拿不到循环变量 `i` 直接报错；而且每次循环都做一次 `v-if` 判断，语义混乱。⚠ 官方明确不推荐同元素混用。

## 4.2 正确写法：包一层 template 或先过滤

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
const items = ref([1, 2, 3])
const visible = ref(true)

// 派生列表：具体数字的过滤用 computed 先算出数据
const bigItems = computed(() => items.value.filter(i => i > 1))
</script>

<template>
  <!-- 方案一：v-if 移到外层 template，循环变量在内层 v-for -->
  <template v-if="visible">
    <li v-for="i in items" :key="i">{{ i }}</li>
  </template>

  <!-- 方案二：数据的过滤交给 computed，模板只做纯循环 -->
  <li v-for="i in bigItems" :key="i">{{ i }}</li>
</template>
```

★ **关键认知**：两种解法——**"整块显隐"用 `<template v-if>` 包住 `v-for`**；**"按数据过滤"用 `computed` 提前算好**，模板里只循环最终结果。后者更符合"模板只描述、逻辑回 JS"的原则，也是首选。

# 5. 本篇小结与下一步

★ **关键认知**：条件渲染分"挂载级"（v-if）与"显示级"（v-show），列表渲染靠 `key` 保证 diff 复用正确，数组过滤交给 computed 而非混写 `v-if`。这三条共同守护"列表正确 + 更新高效"。

下一步第 6 篇：事件与表单——`@click` 的修饰符、`v-model` 双向绑定到底是不是"魔法"。