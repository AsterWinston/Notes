map 是 Go 内置的哈希表，但它有几个反直觉的行为：遍历顺序随机、nil map 不能写但能读、并发读写会直接崩溃。理解它的底层实现（桶 + 溢出链 + 渐进式扩容），才能写出正确且高性能的代码。

# 1. map 的基本用法

## 1.1 创建与操作

```go
// 三种创建方式
m1 := map[string]int{"a": 1, "b": 2}     // 字面量
m2 := make(map[string]int)                // 空 map
m3 := make(map[string]int, 100)           // 预分配容量（性能优化）

m1["c"] = 3           // 写入
v := m1["a"]          // 读取
delete(m1, "b")       // 删除

// 判断键是否存在（重要惯用法）
v, ok := m1["x"]      // ok 为 false 表示键不存在
if v, ok := m1["x"]; ok {
    // 键存在
}
```

★ **关键认知**：读取 map 用**双返回值**区分"键不存在"和"值为零值"。`v, ok := m["key"]` 中，`ok` 为 false 表示键不存在。这是 Go 里非常核心的惯用法，因为 map 的零值是 nil，仅凭返回值无法区分。

## 1.2 键类型的限制

★ **关键认知**：map 的键类型必须是**可比较类型**（`==` 和 `!=` 有定义）：基本类型、指针、字符串、数组、结构体（只要字段都可比较）都可以。但**切片、map、函数不能作键**，因为它们不可比较。

```go
m := map[[2]int]string{}       // 数组可比较，可作键
type Point struct{ X, Y int }
m2 := map[Point]string{}       // 结构体可比较，可作键
```

# 2. map 的底层实现

## 2.1 哈希表结构

★ **关键认知**：Go 的 map 是**哈希表 + 链地址法（桶）** 的实现，核心结构在 `runtime/map.go` 的 `hmap` 中：

```go
// runtime/map.go（简化）
type hmap struct {
    count     int        // 元素个数
    flags     uint8      // 状态标志（如正在写入）
    B         uint8      // 桶数量的对数：桶数 = 2^B
    noverflow uint16     // 溢出桶数量
    hash0     uint32     // 哈希种子（随机）
    buckets   unsafe.Pointer   // 指向桶数组
    oldbuckets unsafe.Pointer  // 扩容时的旧桶数组
    nevacuate uintptr     // 扩容进度
}

type bmap struct {        // 每个桶（bucket）能存 8 个键值对
    tophash [8]uint8      // 8 个键的哈希高 8 位（快速比较）
    keys    [8]keytype
    values  [8]valuetype
    overflow *bmap        // 溢出桶指针
}
```

```
hmap
 └── buckets → [bucket0][bucket1][bucket2]...（2^B 个桶）
                    │
                    ├── tophash[8] + keys[8] + values[8]
                    └── overflow → 溢出桶（桶满 8 个后链出去）
```

## 2.2 查找过程

★ **关键认知**：查找一个键时，Go 的流程是：

```
1. 计算 key 的哈希值 hash(key)（混合了随机种子 hash0，防哈希碰撞攻击）
2. 取哈希低 B 位，定位到目标桶
3. 取哈希高 8 位作为 tophash，与桶内 8 个 tophash 快速比对
4. 命中 tophash 后再逐一比较真实 key（防止哈希冲突误判）
5. 桶满则沿 overflow 链继续找
```

★ **关键认知**：哈希函数混入**随机种子 `hash0`**（每次 map 创建时随机生成），是为了防止"哈希碰撞攻击"——攻击者构造大量同哈希的键，把 map 退化成链表。随机种子让攻击者无法预测哈希值。

# 3. 遍历顺序随机性

```go
m := map[int]string{1: "a", 2: "b", 3: "c", 4: "d"}
for k, v := range m {
    fmt.Println(k, v)   // 每次运行顺序都可能不同！
}
```

★ **关键认知**：Go 的 map 遍历顺序是**故意随机化的**——每次 `range` 的起始桶和起始位置都随机。这不是 bug，而是官方有意为之，防止开发者**错误地依赖遍历顺序**。若需要有序遍历，必须自己收集 keys 并排序：

```go
keys := make([]int, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Ints(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

# 4. nil map 与空 map

```go
var m1 map[string]int        // nil map
m2 := map[string]int{}       // 空 map（已初始化）

m1["a"] = 1   // panic: assignment to entry in nil map
v := m1["a"]  // 合法！读取 nil map 返回零值
delete(m1, "a") // 合法！删除 nil map 不报错
len(m1)       // 合法，返回 0
```

★ **关键认知**：**nil map 可以读、可以 len、可以 delete，但绝不能写**（写入会 panic）。因为 nil map 没有分配底层哈希表。正确的初始化方式是 `make` 或字面量。这也是"用 make 初始化 map"这条惯用法的原因。

# 5. 扩容机制

## 5.1 两种扩容

★ **关键认知**：map 扩容发生在桶装载因子过高或溢出桶过多时，分两种：

```
1. 等量扩容：桶数量不变，但把溢出桶的元素重新整理进主桶
   （触发条件：溢出桶太多，说明大量删除导致空洞）
2. 翻倍扩容：桶数量 × 2，元素重新哈希分布
   （触发条件：装载因子 count / (2^B) > 6.5，平均每桶超过 6.5 个元素）
```

## 5.2 渐进式（增量）扩容

★ **关键认知**：Go map 的扩容是**渐进式**的——扩容触发后，不是一次性搬完所有元素，而是**每次读写时顺便搬迁一部分**（`evacuate` 迁移一两个桶）。这样避免了扩容导致的一次性长时间停顿，把搬迁开销均摊到多次操作中。`hmap.oldbuckets` 保存旧桶，`nevacuate` 记录搬迁进度。

# 6. 并发安全

★ **关键认知**：Go 的 map **不是并发安全的**。多个 goroutine 同时读写同一个 map，会触发运行时检测并直接 **fatal error**（不是普通 panic，无法 recover）：

```
fatal error: concurrent map read and map write
```

解决方案：

```go
// 方案1：sync.Mutex 加锁
var mu sync.Mutex
mu.Lock()
m["k"] = 1
mu.Unlock()

// 方案2：sync.RWMutex（读多写少场景）
// 方案3：sync.Map（特定场景，见并发章节）
```

# 7. 小结

| 概念 | 要点 |
|------|------|
| 双返回值 | `v, ok := m[k]` 判断键是否存在 |
| 键类型 | 必须可比较，slice/map/func 不能作键 |
| 底层 | hmap + 桶(8 键值) + 溢出链 |
| 哈希种子 | 随机 hash0，防碰撞攻击 |
| 遍历 | 故意随机，需排序用 sort |
| nil map | 可读不可写 |
| 扩容 | 渐进式，翻倍/等量两种 |
| 并发 | 非安全，读写竞争直接 fatal |

下一篇进入指针，讲清楚 Go 指针与 C 指针的区别（无指针运算）、new 与 make 的分工，以及 unsafe.Pointer 的预告。
