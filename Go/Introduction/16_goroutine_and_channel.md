并发是 Go 的灵魂，也是它区别于大多数语言的杀手锏。goroutine 的轻量、channel 的优雅、以及"通过通信共享内存"的哲学，共同构成了 Go 的并发模型。这一章深入 goroutine 与 channel 的底层，为后续调度器和 sync 章节打基础。

# 1. 并发与并行的区别

★ **关键认知**：先厘清两个常被混淆的概念：

```
并发（concurrency）：多个任务"交替"执行（单核也能并发），关注结构
并行（parallelism）：多个任务"同时"执行（需多核），关注物理执行

Go 的并发模型让你"编写并发程序"，运行时负责把它映射到多核上并行执行。
```

# 2. goroutine

## 2.1 创建 goroutine

```go
go func() {
    fmt.Println("in goroutine")
}()

go someFunction()    // 用 go 关键字启动一个 goroutine

// 主 goroutine 结束时，所有 goroutine 立即结束
// 因此需要等待机制（sync.WaitGroup 或 channel）
```

★ **关键认知**：`go` 关键字把函数调用放入一个**新的 goroutine** 并发执行。注意主 goroutine（main 函数）结束时，程序直接退出，**不会等待其他 goroutine 完成**——所以并发程序需要显式的同步等待。

## 2.2 goroutine 的轻量本质

★ **关键认知**：goroutine 不是操作系统线程，而是**用户态的轻量级协程**，由 Go 运行时调度：

| 特性 | OS 线程 | goroutine |
|------|---------|-----------|
| 栈初始大小 | ~8MB（固定） | ~2KB（可动态增长） |
| 创建成本 | 高（系统调用） | 极低（几 KB） |
| 调度者 | 内核 | Go 运行时（用户态） |
| 切换成本 | 高（内核态切换） | 低（用户态切换） |
| 数量级 | 通常数百 | 可轻松上百万 |

★ **关键认知**：goroutine 之所以"轻"，核心在两点：**栈小且可动态增长**（2KB 起步，按需扩大到 GB 级），以及**用户态调度**（切换不陷入内核）。这让你能"任性"地为每个请求、每个连接创建一个 goroutine，而不用担心线程开销。调度细节见运行时章节。

# 3. channel 基础

## 3.1 channel 的创建与方向

```go
ch := make(chan int)        // 无缓冲 channel
ch := make(chan int, 10)    // 有缓冲 channel（容量 10）

// 方向
func send(ch chan<- int) { ch <- 1 }        // 只发送
func recv(ch <-chan int)  { v := <-ch }     // 只接收
```

## 3.2 发送与接收

```go
ch := make(chan int, 1)
ch <- 42          // 发送
v := <-ch         // 接收

// 判断 channel 是否已关闭
v, ok := <-ch     // ok 为 false 表示 channel 已关闭且无数据
```

★ **关键认知**：channel 是 Go 内建的类型，用于 goroutine 之间安全地传递数据。它的本质是**并发安全的队列**——多 goroutine 读写无需额外加锁。

# 4. channel 的底层结构

## 4.1 hchan 结构

★ **关键认知**：channel 的底层实现是 `runtime/chan.go` 中的 `hchan` 结构：

```go
// runtime/chan.go（简化）
type hchan struct {
    qcount   uint           // 当前队列中的元素个数
    dataqsiz uint           // 环形队列容量（缓冲大小）
    buf      unsafe.Pointer // 环形缓冲区指针
    elemsize uint16         // 元素大小
    closed   uint32         // 是否已关闭
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex          // 互斥锁（保护 channel 结构）
}
```

```
hchan 结构：
┌─────────────────────────────┐
│ 环形缓冲区 buf（容量 dataqsiz）│
│  [elem][elem][elem][elem]    │
│  recvx↑          sendx↑      │
├─────────────────────────────┤
│ recvq：等待接收的 goroutine  │
│ sendq：等待发送的 goroutine  │
│ lock：互斥锁                 │
└─────────────────────────────┘
```

## 4.2 发送与接收的流程

★ **关键认知**：channel 的发送/接收是"环形队列 + 等待队列"的经典实现：

```
发送 ch <- v 的流程：
1. 加锁
2. 若 recvq 有等待接收者 → 直接把数据交给它（不经过缓冲区，高效）
3. 否则若缓冲区未满 → 写入环形缓冲区
4. 否则 → 发送者阻塞，挂入 sendq 等待队列
5. 解锁

接收 <-ch 的流程对称
```

★ **关键认知**：channel 的同步语义（阻塞/非阻塞）完全由这个结构决定——**无缓冲** channel 相当于"容量为 0"的队列，发送必须等接收者就绪；**有缓冲** channel 在缓冲未满时发送不阻塞。

# 5. 无缓冲 vs 有缓冲 channel

## 5.1 无缓冲 channel（同步）

★ **关键认知**：无缓冲 channel 提供**同步的**通信——发送和接收必须**同时就绪**，否则阻塞：

```go
ch := make(chan int)     // 无缓冲
go func() {
    v := <-ch            // 阻塞等待，直到有发送
    fmt.Println(v)
}()
ch <- 42                 // 阻塞，直到接收者就绪（此时被上面的 goroutine 接收）
```

无缓冲 channel 保证"发送的数据一定被接收了"，常用作**信号/完成通知**。

## 5.2 有缓冲 channel（异步）

```go
ch := make(chan int, 2)   // 缓冲 2 个
ch <- 1                   // 不阻塞
ch <- 2                   // 不阻塞（缓冲满了）
ch <- 3                   // 阻塞！缓冲满，等待接收
```

★ **关键认知**：有缓冲 channel 像一个**容量固定的队列**，常用于**解耦**生产者和消费者的速度、或实现**限流**（缓冲满了就阻塞生产者）。

# 6. channel 的关闭

## 6.1 close 的语义

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)          // 关闭 channel

v, ok := <-ch      // ok=true, v=1
v, ok = <-ch       // ok=true, v=2
v, ok = <-ch       // ok=false, v=0（零值），channel 已空且关闭
```

## 6.2 close 的规则与陷阱

★ **关键认知**：channel 关闭有严格规则，违反会 panic：

```
1. 向已关闭的 channel 发送 → panic
2. 重复关闭 channel → panic
3. 从已关闭的 channel 接收 → 返回零值 + ok=false（不 panic）
4. 关闭 nil channel → panic
```

⚠ **关键认知**：关闭 channel 的**责任归属**很重要——通常由**发送方**关闭（因为发送方才知道何时不再发送）。接收方无法安全关闭一个可能还有发送者的 channel。经典错误是接收方 close 导致发送方 panic。

# 7. 通过通信共享内存

★ **关键认知**：Go 的并发哲学浓缩为一句话：

> "不要通过共享内存来通信，而应该通过通信来共享内存。"

```
传统方式（共享内存 + 锁）：
  多个 goroutine 共享变量，用互斥锁保护，易出死锁/竞态

Go 的方式（channel 通信）：
  数据在 goroutine 间"传递"而非"共享"，每个时刻数据只被一个 goroutine 拥有
```

```go
// 通信共享内存：把数据发给一个专门的 goroutine 处理
counter := make(chan int)
go func() {
    total := 0
    for v := range counter {   // 独占变量，无需锁
        total += v
    }
}()
counter <- 1
counter <- 2
```

★ **关键认知**：这个哲学让并发程序的正确性更容易推理——数据所有权清晰，避免共享可变状态的竞态。当然，它**不是银弹**：某些场景（如高性能计数器）共享内存 + 原子操作/锁反而更合适。两者结合使用才是成熟的 Go 并发编程。

# 8. 小结

| 概念 | 要点 |
|------|------|
| goroutine | 用户态轻量协程，2KB 栈起步 |
| hchan | 环形队列 + 等待队列 + 锁 |
| 无缓冲 | 同步通信，发送接收须同时就绪 |
| 有缓冲 | 异步队列，可解耦/限流 |
| close | 发送方关闭，重复关闭 panic |
| 哲学 | 通过通信共享内存 |

下一篇进入 select 与 sync 包，讲清楚 select 的多路复用机制，以及 Mutex、RWMutex、WaitGroup、Once、atomic 等并发原语的底层实现。
