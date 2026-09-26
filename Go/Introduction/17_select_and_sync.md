上一章讲了 goroutine 和 channel 的基础，这一章深入并发的"工具箱"：`select` 多路复用、以及 `sync` 包里的并发原语（Mutex、RWMutex、WaitGroup、Once、atomic）。理解它们的底层实现，才能在并发场景做出正确的选择。

# 1. select 多路复用

## 1.1 select 的机制

★ **关键认知**：`select` 让一个 goroutine 同时等待**多个 channel 操作**，哪个先就绪就执行哪个：

```go
select {
case v := <-ch1:
    fmt.Println("recv ch1", v)
case v := <-ch2:
    fmt.Println("recv ch2", v)
case ch3 <- 42:
    fmt.Println("sent ch3")
default:
    fmt.Println("no case ready")
}
```

## 1.2 select 的随机选择

★ **关键认知**：当**多个 case 同时就绪**时，`select` 会**伪随机**地选择一个执行，而不是按书写顺序。这是刻意的设计——防止某个 case 被"饿死"，也防止程序依赖特定的选择顺序。

## 1.3 select 的常见模式

```go
// 1. 超时模式
select {
case v := <-ch:
    fmt.Println(v)
case <-time.After(3 * time.Second):
    fmt.Println("timeout")
}

// 2. 非阻塞发送/接收（配合 default）
select {
case ch <- 1:
    fmt.Println("sent")
default:
    fmt.Println("buffer full, skip")
}

// 3. 退出通知
select {
case <-ctx.Done():    // context 取消
    return
case v := <-ch:
    // 处理
}
```

★ **关键认知**：`time.After` 返回一个 channel，在指定时间后发送值——配合 select 实现超时。`default` 让 select 变成**非阻塞**操作。这两个模式是并发编程中最常用的组合。

# 2. sync.Mutex 互斥锁

## 2.1 基本用法

```go
var mu sync.Mutex
var count int

func increment() {
    mu.Lock()         // 加锁
    count++
    mu.Unlock()       // 解锁
}
```

★ **关键认知**：Mutex 保证同一时刻只有一个 goroutine 能进入临界区。Go 社区惯例：**把 Mutex 和被它保护的变量放在同一个结构体里**，通过命名体现保护关系。

## 2.2 底层实现（CAS 与信号量）

★ **关键认知**：Go 的 Mutex 基于 **CAS（Compare-And-Swap）原子操作** 实现，核心字段在 `sync/mutex.go`：

```go
type Mutex struct {
    state int32    // 锁状态（是否加锁 + 等待者数量等）
    sema  uint32   // 信号量，用于阻塞/唤醒等待者
}
```

```
加锁流程（简化）：
1. 用 CAS 尝试把 state 从"未锁"改为"已锁"
2. 成功 → 立即获得锁（快路径，无系统调用）
3. 失败 → 进入自旋（短暂忙等，等待持锁者很快释放）
4. 自旋后仍失败 → 挂入信号量等待队列，阻塞（慢路径，系统调用）
```

★ **关键认知**：Mutex 的"快路径"用 CAS 无锁竞争，只有争抢激烈时才退化到信号量阻塞。这是"乐观并发"思想——多数情况下锁很快释放，自旋等待比立即阻塞更高效。

# 3. sync.RWMutex 读写锁

## 3.1 读写锁的语义

```go
var rw sync.RWMutex

rw.RLock()      // 读锁：允许多个 goroutine 同时持有
rw.RUnlock()

rw.Lock()       // 写锁：独占，排斥所有读和写
rw.Unlock()
```

★ **关键认知**：RWMutex 遵循"**读读共享、读写互斥、写写互斥**"。适用于**读多写少**的场景（如配置缓存），能显著提升并发读性能。写锁是排他的，会阻塞所有读锁。

## 3.2 写锁优先

★ **关键认知**：Go 的 RWMutex 实现有"**写者优先**"倾向——当有 goroutine 等待写锁时，新的读锁请求会被阻塞，避免写者被源源不断的读者饿死。这是并发设计中重要的公平性考量。

# 4. sync.WaitGroup 等待组

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
    wg.Add(1)          // 计数器 +1
    go func(i int) {
        defer wg.Done()   // 完成时计数器 -1
        fmt.Println(i)
    }(i)
}

wg.Wait()              // 阻塞，直到计数器归零
```

★ **关键认知**：WaitGroup 用于等待一组 goroutine 全部完成。三条规则：

```
1. Add 必须在 goroutine 启动前调用（或至少在 Wait 前）
2. Done 等价于 Add(-1)，通常用 defer 保证执行
3. Wait 阻塞直到计数器归零
```

⚠ **关键认知**：`Add` 的时机很关键——若在 `go` 之后才 `Add`，可能 Wait 已经通过（计数为 0）而 goroutine 还没启动。正确顺序是"先 Add，再 go"。

# 5. sync.Once 单次执行

```go
var once sync.Once

for i := 0; i < 100; i++ {
    go func() {
        once.Do(initResource)   // initResource 只会执行一次
    }()
}
```

★ **关键认知**：`Once.Do` 保证传入的函数**只执行一次**，即使被多个 goroutine 并发调用。它内部用原子操作 + Mutex 实现，是线程安全的"懒加载"惯用法（如初始化单例、连接池）。注意：即使函数执行 panic，Once 也会标记为"已执行"。

# 6. sync/atomic 原子操作

## 6.1 原子操作 vs 锁

★ **关键认知**：原子操作是**更底层**的并发原语——直接操作 CPU 的原子指令（如 CMPXCHG），无需锁的上下文切换，性能极高：

```go
import "sync/atomic"

var counter int64

atomic.AddInt64(&counter, 1)      // 原子加 1
atomic.LoadInt64(&counter)        // 原子读
atomic.StoreInt64(&counter, 0)    // 原子写
atomic.CompareAndSwapInt64(&counter, old, new)   // CAS
```

★ **关键认知**：对于**简单计数**这类场景，`atomic` 比 Mutex 快得多（无锁竞争、无阻塞）。但它只能保证**单个操作**的原子性，无法保护多步骤的复合操作（如"读-改-写"的复杂逻辑），那些仍需用锁。

## 6.2 atomic.Value

```go
var v atomic.Value    // 原子地存/取任意类型的值
v.Store(map[string]int{"a": 1})
m := v.Load().(map[string]int)
```

★ **关键认知**：`atomic.Value` 提供任意值的原子存取，常用于**无锁地共享配置**（读多写少的场景），配合"整体替换"模式（copy-on-write）实现无锁并发读。

# 7. sync.Cond 条件变量

★ **关键认知**：`sync.Cond` 用于"等待某个条件成立"的场景，比忙等（busy-wait）高效：

```go
var mu sync.Mutex
cond := sync.NewCond(&mu)

// 等待方
mu.Lock()
for !condition() {          // 必须用 for 而非 if（防止虚假唤醒）
    cond.Wait()             // 释放锁并阻塞，被唤醒后重新加锁
}
mu.Unlock()

// 通知方
cond.Signal()    // 唤醒一个等待者
cond.Broadcast() // 唤醒所有等待者
```

★ **关键认知**：Cond 使用较少（多数场景 channel 更简洁），但理解它的"等待条件"模式很有价值。⚠ 经典陷阱：`Wait()` 后必须重新检查条件（用 `for` 而非 `if`），因为可能存在**虚假唤醒**。

# 8. 小结

| 原语 | 用途 | 适用场景 |
|------|------|----------|
| select | 多路复用 channel | 超时/非阻塞/退出 |
| Mutex | 互斥锁 | 保护临界区 |
| RWMutex | 读写锁 | 读多写少 |
| WaitGroup | 等待组 | 等待 goroutine 完成 |
| Once | 单次执行 | 懒加载/单例 |
| atomic | 原子操作 | 简单计数器/无锁配置 |
| Cond | 条件变量 | 等待条件成立 |

下一篇进入反射，讲清楚 reflect.Type/Value 的本质、反射的底层原理，以及为什么反射有性能代价、什么时候该避免它。
