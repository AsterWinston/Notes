并发不是"多开几个 goroutine 就一定更快"，而是用一套结构把任务拆开、协调、收束。本篇是并发模式模块的起点，先把 goroutine、channel、select、sync 这四件基础工具重新过一遍，并厘清并发与并行的区别，为后面的 worker 池、pipeline、fan-out/fan-in 打好地基。

# 1. 并发与并行

## 1.1 两个概念的区别

初学者最容易把"并发"和"并行"当成同义词。它们描述的是两件不同的事：一个是**程序结构**，一个是**物理执行**。

| 维度 | 并发（concurrency） | 并行（parallelism） |
|------|---------------------|---------------------|
| 关注点 | 任务的**组织方式** | 任务的**同时执行** |
| 硬件要求 | 单核即可 | 必须多核 |
| 典型表现 | 交替推进、互不阻塞 | 真正同时运行 |
| 解决的问题 | 等待与协作 | 计算吞吐 |

```text
单核上的并发（交替执行）：
时间 →  ┌────┬────┬────┬────┬────┐
G1      │执行│等待│执行│等待│执行│
G2      │等待│执行│等待│执行│等待│   ← 同一时刻只有一个在跑

多核上的并行（同时执行）：
时间 →  ┌─────────────┐
G1      │   持续执行   │
G2      │   持续执行   │   ← 同一时刻真的有两个在跑
```

★ **关键认知**：并发是一种**让多个任务交替推进、互不阻塞**的设计能力，它不要求多核；并行是**多个任务在同一时刻真的一起跑**，它依赖多核。Go 提供的是编写并发程序的语言设施，运行时负责把这些 goroutine 映射到可用的 CPU 核心上，能不能变成并行取决于 `GOMAXPROCS` 和机器核数。

## 1.2 一个直观对比

```go
// 串行：两个等待相加，约 200ms
start := time.Now()
time.Sleep(100 * time.Millisecond) // 模拟 I/O
time.Sleep(100 * time.Millisecond)
fmt.Println("串行耗时:", time.Since(start))

// 并发：两个等待重叠，约 100ms
start = time.Now()
var wg sync.WaitGroup
wg.Add(2)
for i := 0; i < 2; i++ {
    go func() {
        defer wg.Done()
        time.Sleep(100 * time.Millisecond)
    }()
}
wg.Wait()
fmt.Println("并发耗时:", time.Since(start))
```

★ **关键认知**：这个例子里两个 goroutine 大部分时间在 `Sleep`（等待），并没有消耗两个 CPU 核心，属于典型的**并发而非并行**。真正的收益来自"等待时间被重叠"。如果你的任务全是纯计算、没有等待，并发不会变快，只有并行才有意义——而并行受核数限制。

# 2. goroutine 回顾

## 2.1 启动与生命周期

`go` 关键字把一个函数调用交给一个**新的 goroutine** 去执行，调用处立即返回。

```go
package main

import "fmt"

func main() {
    go fmt.Println("子 goroutine 在跑") // 立即返回，不保证何时执行
    fmt.Println("主 goroutine 结束")
    // 输出可能是：
    //   主 goroutine 结束
    // 子 goroutine 根本没来得及打印，程序就退出了
}
```

★ **关键认知**：主 goroutine（`main` 函数所在的那个）退出时，**整个进程立即结束**，所有其他 goroutine 被直接杀掉，不会等待、也不会执行 `defer`。所以并发程序必须有显式的等待机制：`sync.WaitGroup`、channel、或者 `errgroup`。

```go
var wg sync.WaitGroup
wg.Add(1) // 必须在 go 之前 Add
go func() {
    defer wg.Done() // 完成时减 1
    fmt.Println("子 goroutine 完成")
}()
wg.Wait() // 阻塞直到计数归零
```

## 2.2 轻量在哪里

| 特性 | OS 线程 | goroutine |
|------|---------|-----------|
| 初始栈 | 通常 1-8MB 固定 | 约 2KB，按需增长 |
| 创建成本 | 系统调用，较贵 | 用户态分配，极低 |
| 调度者 | 操作系统内核 | Go 运行时（G-M-P） |
| 切换成本 | 内核态切换 | 用户态切换 |
| 常见数量 | 数百到数千 | 数十万到上百万 |

```text
G（goroutine） → 由 P（逻辑处理器）绑定 → 运行在 M（OS 线程）上
     │
     ├── G 阻塞在系统调用 → P 与 M 解绑，另找空闲 M 继续跑其他 G
     └── G 阻塞在 channel  → 调度器直接切到另一个可运行的 G
```

★ **关键认知**：goroutine 的"轻"来自两点：**初始栈极小且可动态伸缩**，以及**由运行时在用户态调度**，切换不陷入内核。这让你可以为每个请求、每个连接创建一个 goroutine，而不必维护线程池。⚠ 但"轻"不等于"免费"：一个永远阻塞在 channel 上、又没有退出路径的 goroutine，会连同它引用的内存一起泄漏，这类问题在第 8 篇专门讲。

# 3. channel 回顾

## 3.1 两种缓冲形态

```go
ch := make(chan int)     // 无缓冲：容量 0
buf := make(chan int, 3) // 有缓冲：容量 3
```

| 类型 | 发送阻塞条件 | 接收阻塞条件 | 语义 |
|------|--------------|--------------|------|
| 无缓冲 | 没有接收者就绪 | 没有发送者就绪 | 同步握手 |
| 有缓冲 | 缓冲区已满 | 缓冲区为空 | 异步队列 |

```go
ch := make(chan int, 2)
ch <- 1 // 不阻塞
ch <- 2 // 缓冲区刚好满
// ch <- 3  // 阻塞：缓冲区满
fmt.Println(<-ch, <-ch) // 1 2
```

★ **关键认知**：无缓冲 channel 的发送必须等到接收方就绪，是一次**同步握手**，天然适合做"交接"和"完成通知"；有缓冲 channel 是一个**定长队列**，用于吸收生产者和消费者之间的速度差。缓冲区大小不是越大越好——它只是把背压延后，并不会消除背压（见第 7 篇）。

## 3.2 关闭规则与方向

```go
func producer(out chan<- int) { // 只写
    for i := 0; i < 3; i++ {
        out <- i
    }
    close(out) // 由发送方负责关闭
}

func consumer(in <-chan int) { // 只读
    for v := range in { // channel 关闭且读完后自动退出循环
        fmt.Println(v)
    }
}
```

★ **关键认知**：channel 的方向可以在函数签名里限定为 `chan<- T`（只写）或 `<-chan T`（只读），让编译器帮你挡住误用。关闭规则可以浓缩成四条：

```text
1. 向已关闭的 channel 发送  → panic
2. 重复关闭同一个 channel    → panic
3. 关闭 nil channel          → panic
4. 从已关闭的 channel 接收   → 立即返回零值，ok == false
```

⚠ **关键认知**：关闭 channel 的职责属于**发送方**，因为只有发送方知道"不会再有数据了"。接收方去关闭一个可能还有别的发送者的 channel，会让那些发送者 panic。多个发送者时，通常用一个额外的 `done` channel 或者 `sync.Once` 来协调关闭。

# 4. select 回顾

`select` 让一个 goroutine 同时等待多个 channel 操作。

```go
select {
case v := <-ch1:
    fmt.Println("来自 ch1:", v)
case ch2 <- 42:
    fmt.Println("已发送到 ch2")
case <-ctx.Done():
    fmt.Println("被取消:", ctx.Err())
default:
    fmt.Println("没有任何 case 就绪")
}
```

★ **关键认知**：多个 case 同时就绪时，`select` 会**伪随机**挑一个执行，而不是按书写顺序。这是刻意设计——防止某个 case 长期被饿死，也防止程序依赖不确定的顺序。`default` 让 select 变成非阻塞；把 `ctx.Done()` 放进 select，是让 goroutine 可被取消的标准写法。

⚠ **关键认知**：`for { select { ... default: } }` 这种写法如果没有阻塞点，会变成**CPU 空转**的忙循环。要么去掉 `default` 让 select 阻塞，要么在 default 分支里加 `time.Sleep` 或 `runtime.Gosched()`。这是新手最常踩的性能坑之一。

# 5. sync 原语回顾

## 5.1 Mutex 与 RWMutex

```go
type Counter struct {
    mu sync.Mutex // 和它保护的字段放在一起
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.n++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.n
}
```

★ **关键认知**：`Mutex` 保护的是**临界区**，不是变量本身。Go 的惯例是把锁和被保护的字段放进同一个结构体，并且**锁不导出、不复制**。`RWMutex` 提供"读读共享、读写互斥、写写互斥"，适合读多写少；但读操作极短时，`RWMutex` 的额外记账开销可能不如普通 `Mutex`。

⚠ **关键认知**：含 `Mutex` 的结构体不能按值复制。`go vet` 会检查这件事。把锁传给函数时用指针，别写成 `func f(c Counter)`。

## 5.2 WaitGroup 与 Once

```go
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(id int) { // Go 1.22 起循环变量每轮独立，这里传参仍更清晰
        defer wg.Done()
        fmt.Println("worker", id)
    }(i)
}
wg.Wait()
```

```go
var once sync.Once
var conn *Conn

func getConn() *Conn {
    once.Do(func() { conn = dial() }) // 只执行一次
    return conn
}
```

★ **关键认知**：`Add` 必须在 `Wait` 可能返回之前调用，惯例是"先 Add，再 go"。`Once.Do` 保证函数只执行一次，即使函数 panic 也会被标记为已执行。Go 1.21 起还提供了 `sync.OnceFunc`、`sync.OnceValue`，省去手写结构体字段。

## 5.3 atomic

```go
import "sync/atomic"

var hits atomic.Int64 // Go 1.19 起的类型化原子量

hits.Add(1)
fmt.Println(hits.Load())
hits.Store(0)
```

★ **关键认知**：原子操作适合**单个变量的读改写**，例如计数器、开关、状态位，性能远高于加锁。但它无法保证**多个变量之间的一致性**——如果你需要"同时更新 A 和 B，别人要么都看到新值、要么都看到旧值"，那必须用锁或 channel。

# 6. 原语怎么选

| 需求 | 首选 | 说明 |
|------|------|------|
| 等待一组任务结束 | `sync.WaitGroup` | 简单直接，不传错误 |
| 等待并收集错误 | `errgroup.Group` | 第 6 篇详述 |
| 传递数据 / 交接所有权 | channel | 数据只被一个 goroutine 拥有 |
| 保护共享结构体 | `sync.Mutex` | 临界区尽量短 |
| 读多写少缓存 | `sync.RWMutex` 或 `atomic.Value` | 整体替换用后者 |
| 单纯计数 | `atomic.Int64` | 比锁快 |
| 一次性初始化 | `sync.Once` | 懒加载单例 |
| 取消 / 超时 | `context.Context` | 第 5 篇详述 |

★ **关键认知**：选择原语的核心问题是"**谁拥有这份数据**"。如果数据在任一时刻只属于一个 goroutine，用 channel 传递所有权，逻辑最清晰；如果确实需要多方同时读写同一块内存，用锁把临界区圈起来。不要为了"显得并发"而把所有东西都塞进 channel。

# 7. 常见错误清单

```text
1. main 直接返回，没等子 goroutine      → 任务丢失
2. WaitGroup.Add 写在 go 之后           → Wait 可能提前返回
3. 接收方关闭 channel                   → 发送方 panic
4. for + select + default 无阻塞点      → CPU 100%
5. 复制含 Mutex 的结构体                → 锁失效，竞态
6. goroutine 里忘了 defer cancel()      → context 泄漏
7. 无缓冲 channel 同 goroutine 内自收自发 → 死锁
```

⚠ **关键认知**：第 7 条值得单独说：同一个 goroutine 既发送又接收一个无缓冲 channel，没有任何第三方来配对，就会直接死锁。

# 8. 练习

**练习 1**：写一个程序，启动 5 个 goroutine，每个打印自己的编号，用 `WaitGroup` 等它们全部结束后再打印"全部完成"。观察多次运行输出顺序是否固定。

**练习 2**：用无缓冲 channel 实现两个 goroutine 的"乒乓"：A 发送 1 给 B，B 收到后加 1 发回 A，往返 10 次后双方退出。体会无缓冲 channel 的同步握手语义。

**练习 3**：把第 1.2 节的 `serial` 改成 10 次 100ms 的 `Sleep`，再写一个并发版本，比较两者耗时。然后把这 10 个任务换成纯计算（例如累加一千万次），再比较一次，解释为什么两次的加速比不同。

# 9. 小结

| 概念 | 一句话要点 |
|------|-----------|
| 并发 vs 并行 | 并发是结构，并行是同时执行，后者依赖多核 |
| goroutine | 用户态轻量协程，主 goroutine 退出即全灭 |
| channel | 无缓冲同步握手，有缓冲异步队列 |
| close | 发送方关闭，向已关闭 channel 发送会 panic |
| select | 多路等待，多 case 就绪时随机选择 |
| Mutex | 保护临界区，结构体不可复制 |
| WaitGroup | 先 Add 再 go，Done 配 defer |
| atomic | 单变量原子操作，不保证多变量一致性 |

下一篇进入 worker 池：当任务数量远大于并发上限时，如何用固定数量的 goroutine 消费任务队列，做到可控并发、优雅关闭和错误收集。
