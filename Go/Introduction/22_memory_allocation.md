Go 的内存分配器是性能调优的隐藏功臣。它借鉴了 TCMalloc（Google 的 C++ 内存分配器），用**多级缓存**实现极快的对象分配。理解 mspan/mcache/mcentral/mheap 的结构，能解释 Go 为什么分配快、以及如何避免内存碎片。

# 1. 内存分配器概览

## 1.1 借鉴 TCMalloc

★ **关键认知**：Go 的内存分配器设计源自 Google 的 **TCMalloc**，核心思想是**分级缓存 + 大小分类**：

```
TCMalloc 的核心思想：
1. 按对象大小分类（size class），每个类别有独立的内存池
2. 每个线程（Go 里是 P）有本地缓存，避免加锁
3. 本地缓存不足时，从全局缓存批量获取
```

## 1.2 四级结构

★ **关键认知**：Go 内存分配器是四级层次结构：

```
mheap（全局堆）
  └── mcentral（全局中央缓存，按大小分类）
        └── mcache（每个 P 的本地缓存，无锁）
              └── mspan（一段连续内存，按大小分类切块）
```

```
分配路径（从小到大）：
mcache → mcentral → mheap → 操作系统
（本地缓存 → 中央缓存 → 全局堆 → 系统内存）
```

# 2. 内存块：mspan

## 2.1 mspan 与大小分类

★ **关键认知**：`mspan` 是 Go 内存分配的基本单位——一段连续的页（page），被切分成**相同大小**的对象块：

```go
// runtime/mheap.go（简化）
type mspan struct {
    startAddr uintptr   // 起始地址
    npages    uintptr   // 占用页数
    spanclass spanClass // 大小分类
    freeindex uintptr   // 空闲对象索引
    allocBits *gcBits   // 分配位图
    ...
}
```

## 2.2 大小分类（size class）

★ **关键认知**：Go 把对象按大小分成约 70 个**大小分类（size class）**，每个分类对应一个固定的对象大小：

```
size class 示例（简化）：
class 1: 8 字节   （对象 0~8 字节都归入此类）
class 2: 16 字节  （对象 9~16 字节归入此类）
class 3: 24 字节
class 4: 32 字节
...
class 67: 32KB（大对象）

规则：
- 小对象（≤32KB）按 size class 分配
- 大对象（>32KB）直接从 mheap 分配（不经过缓存）
```

★ **关键认知**：size class 的粒度保证**内存利用率与分配速度的平衡**。申请 15 字节的对象，实际会分配到 16 字节的块（向上取整）。这种"固定大小池"避免了碎片，也让分配/释放变成简单的位图操作。

# 3. 本地缓存：mcache

★ **关键认知**：`mcache` 是**每个 P 独有的**本地缓存，分配小对象时**无需加锁**（因为同一时刻一个 P 只被一个 M 使用）：

```go
// runtime/mcache.go（简化）
type mcache struct {
    tiny       uintptr    // 微小对象分配器
    tinyoffset uintptr
    alloc      [numSpanClasses]*mspan   // 每个 size class 一个 mspan
    ...
}
```

★ **关键认知**：mcache 是"无锁分配"的关键——goroutine 分配内存时，优先从**自己所属 P 的 mcache** 取，完全无锁，速度极快。只有 mcache 的 mspan 用尽时，才需要向 mcentral 申请（这一步才需要加锁）。

# 4. 微小对象分配：tiny allocator

★ **关键认知**：对于**极小对象**（小于 16 字节），Go 用 **tiny allocator** 进一步优化——把多个微小对象"塞"进同一个 16 字节块：

```
微小对象分配：
1. mcache 有一个 tiny 指针，指向一个 16 字节的空闲块
2. 微小对象直接往这个块里"追加"
3. 块满了再申请新的
```

★ **关键认知**：tiny allocator 让"字符串、指针"这类高频小对象（如 `&string`、`&int`）的分配开销降到几乎为零。这是 Go 大量使用小指针却仍有高分配性能的原因。

# 5. 中央缓存：mcentral

★ **关键认知**：`mcentral` 是**全局的中央缓存**，按 size class 分组。当 mcache 不足时，从这里批量获取 mspan：

```go
// runtime/mcentral.go（简化）
type mcentral struct {
    spanclass spanClass
    partial   [2]spanSet   // 部分空闲的 mspan
    full      [2]spanSet   // 已满的 mspan
}
```

★ **关键认知**：mcentral 维护每个 size class 的"部分空闲"和"已满"mspan 列表。mcache 需要时从这里拿，用完了再还回来。这里需要加锁（多个 P 可能同时访问），但访问频率远低于 mcache。

# 6. 全局堆：mheap

★ **关键认知**：`mheap` 是**整个进程的堆管理器**，负责向操作系统申请/释放内存，并切分成 mspan：

```go
// runtime/mheap.go（简化）
type mheap struct {
    lock      mutex
    pages     pageAlloc    // 页分配器（管理所有页）
    central   [numSpanClasses]mcentral  // 中央缓存
    arenas    [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena  // 堆区域
    ...
}
```

★ **关键认知**：mheap 通过 `pages`（页分配器）管理内存页，把页切分成 mspan 供 mcentral/mcache 使用。当整个堆都不够时，mheap 才向操作系统申请新的内存区域（arena），这是最慢的路径。

# 7. 栈 vs 堆的分配

★ **关键认知**：Go 的内存分为**栈**和**堆**两部分，分配机制完全不同：

```
栈分配：
  - 每个 goroutine 有独立栈（2KB 起）
  - 分配/释放只是移动栈指针（SP），几乎零成本
  - 函数返回时自动释放

堆分配：
  - 走 mcache→mcentral→mheap 的分配器
  - 需要 GC 回收，有额外开销
  - 由逃逸分析决定对象放栈还是堆
```

★ **关键认知**：栈分配**极快**（一个指针加减），堆分配有分配器和 GC 双重开销。因此**逃逸分析**（决定对象能否放栈上）对性能至关重要——能留栈上的对象，就不要让它逃逸到堆（详见逃逸分析章节）。

# 8. 分配的性能优化启示

★ **关键认知**：基于以上机制，优化内存分配的方向：

```
1. 减少分配次数：复用对象（sync.Pool）、预分配容量
2. 让对象留在栈上：避免返回指针、避免闭包捕获（配合逃逸分析）
3. 用微小对象：小对象走 tiny allocator 更快
4. 避免大对象频繁分配：>32KB 的大对象直接走 mheap，开销大
```

# 9. 小结

| 概念 | 要点 |
|------|------|
| 分配器 | 借鉴 TCMalloc |
| mspan | 连续页切块，按 size class |
| mcache | 每 P 本地缓存，无锁 |
| mcentral | 全局中央缓存 |
| mheap | 全局堆，向 OS 申请 |
| tiny allocator | 微小对象塞同块 |
| 栈 vs 堆 | 栈近乎零成本，堆走分配器 |

下一篇进入逃逸分析与内联，讲清楚编译器如何决定对象放栈还是堆（逃逸分析），以及内联优化如何消除函数调用开销——这是 Go 编译器优化的核心。
