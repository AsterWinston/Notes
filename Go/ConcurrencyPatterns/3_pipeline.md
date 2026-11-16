pipeline 是把一个复杂任务拆成若干**顺序阶段**、用 channel 首尾相连的并发模式。它像工厂流水线：数据从一端流入，每经过一个工位就被加工一次，最后从另一端流出。和 worker 池相比，pipeline 强调的是**阶段之间的依赖与数据流**，而不是"把一堆独立任务分给谁做"。本篇从最小三段流水线讲起，重点解决关闭传播、错误传递和取消这三个真正容易出错的地方。

# 1. 什么是 pipeline

## 1.1 结构

```text
generate ──► square ──► filter ──► print
  (生产)      (加工)      (过滤)     (消费)
    │            │           │
  out ch  ──► in/out  ──► in/out ──► 读取
```

每个阶段通常是一个函数：**接收上游的只读 channel，返回下游的只写 channel**，内部起一个 goroutine 边读边加工边写。

```go
func stage(in <-chan int) <-chan int
```

★ **关键认知**：pipeline 的每个阶段都是**独立的 goroutine**，阶段之间只通过 channel 通信，不共享内存。这带来两个好处：每个阶段可以独立测试；并发度由阶段自然决定（慢的阶段可以多开几个副本，即 fan-out）。代价是错误和取消必须沿着 channel 显式传播，不能靠函数返回值层层上抛。

## 1.2 最小三段流水线

```go
package main

import "fmt"

// 阶段 1：生产数据
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out) // 生产完必须关闭，通知下游结束
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// 阶段 2：平方
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in { // 上游关闭后循环结束
            out <- n * n
        }
    }()
    return out
}

func main() {
    for v := range square(generate(1, 2, 3, 4)) {
        fmt.Println(v) // 1 4 9 16
    }
}
```

★ **关键认知**：这段代码能正确结束，靠的是**每个阶段的 `defer close(out)`**。`close` 会像多米诺骨牌一样向下游传播：generate 关闭 → square 的 `range` 结束 → square 关闭自己的 out → main 的 `range` 结束。少了任何一个 `close`，下游的 `range` 就会永远阻塞。

# 2. 构建阶段

## 2.1 阶段函数的约定

一个规范化的阶段应该满足：

| 约定 | 原因 |
|------|------|
| 参数是 `<-chan T` | 阶段只读上游，不能关闭它 |
| 返回 `<-chan U` | 阶段只写下游，关闭责任在阶段内部 |
| 内部起 goroutine | 调用方拿到 channel 后可以立即继续串联 |
| `defer close(out)` | 保证任何退出路径都会通知下游 |
| 监听 `ctx.Done()` | 保证可取消，避免泄漏 |

## 2.2 四阶段完整示例

```go
package main

import (
    "context"
    "fmt"
)

func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return // 下游不要了，提前退出
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func filterEven(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 != 0 {
                continue // 过滤掉奇数，不向下游发送
            }
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // 提前返回时释放资源

    nums := generate(ctx, 1, 2, 3, 4, 5, 6)
    squares := square(ctx, nums)
    evens := filterEven(ctx, squares)

    for v := range evens {
        fmt.Println(v) // 4 16 36
    }
}
```

★ **关键认知**：注意每个阶段内部的发送都包在 `select` 里。如果下游提前退出（比如 `main` 里 `break` 了），下游就不会再读，上游的 `out <- n` 会永久阻塞；加上 `case <-ctx.Done()` 后，`cancel()` 一调用，所有阻塞的发送都能被唤醒并退出。这是可取消 pipeline 的关键细节。

# 3. 关闭传播

## 3.1 关闭由谁负责

pipeline 里有一条铁律：**每个 channel 由它的发送方（也就是拥有 out 的那个阶段）关闭**。

```text
generate 拥有 ch1  → generate 负责 close(ch1)
square   拥有 ch2  → square   负责 close(ch2)
main     不拥有任何 channel → 不关闭，只读取
```

⚠ **关键认知**：如果你的 `main` 里写了 `close(evens)`，那是在关闭一个由 `filterEven` 拥有的 channel，属于越权。虽然这里可能恰好不 panic，但只要 filterEven 还在往里发，就会触发 `send on closed channel`。接收方永远不该关闭上游 channel。

## 3.2 `range` 的本质

```go
for v := range ch {
    // 等价于：
    // for {
    //     v, ok := <-ch
    //     if !ok { break }
    //     ...
    // }
}
```

★ **关键认知**：`range ch` 会在 channel **关闭且缓冲区数据取完**后自动结束，这是 pipeline 各阶段能自然收尾的机制。⚠ 但 `range` 不会因为 `context` 取消而结束——channel 没关闭，它就一直等。所以在可取消的场景，阶段内部要用 `select`，而不是把 `range` 当作"取消感知"手段。

# 4. 错误处理

## 4.1 错误不能塞进数据流

最直接的想法是让 channel 传 `struct{ val int; err error }`，但这样每个阶段都要检查错误，而且下游可能处理不了上游的错误。

| 方案 | 做法 | 适用 |
|------|------|------|
| 结果结构体 | `chan Result{V, Err}` | 单阶段、错误与数据强相关 |
| 独立 error channel | `chan error`，与数据 channel 并列 | 多阶段、需要统一上报 |
| errgroup + context | 任一阶段失败即取消全部 | 无严格顺序依赖的并行阶段 |

## 4.2 独立 error channel

```go
func squareWithErr(ctx context.Context, in <-chan int) (<-chan int, <-chan error) {
    out := make(chan int)
    errCh := make(chan error, 1) // 缓冲 1，保证一定能写入
    go func() {
        defer close(out)
        defer close(errCh)
        for n := range in {
            if n < 0 {
                select {
                case errCh <- fmt.Errorf("负数无法处理: %d", n):
                case <-ctx.Done():
                }
                return // 出错后停止本阶段
            }
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out, errCh
}
```

★ **关键认知**：error channel 至少给 1 个缓冲，否则在"没人同时读错误"时，阶段会卡在发送错误上，反而让整条流水线停住。这个模式里，错误阶段 `return` 后会 `close(out)`，下游自然结束，再由最外层统一读 `errCh`。

⚠ **关键认知**：如果上游阶段因为错误提前 `return`，它不会再读自己的 `in`，而更上游的阶段可能正阻塞在发送上。所以**错误必须配合 `cancel()`** 使用，让上游也收到取消信号，否则整条链会卡死。

# 5. 取消与早停

## 5.1 用 context 贯穿所有阶段

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

out := filterEven(ctx, square(ctx, generate(ctx, 1, 2, 3, 4, 5, 6)))
for v := range out {
    if v > 10 {
        break // 消费者提前停止
    }
    fmt.Println(v)
}
```

★ **关键认知**：所有阶段共享同一个 `ctx`，`cancel()` 或超时一到，每个阶段的 `select` 都会走 `<-ctx.Done()` 分支并退出。这就是取消在 pipeline 里的传播方式——不是靠 channel 关闭，而是靠一个**共享的广播信号**。

## 5.2 早停必须排空或取消

消费者提前 `break` 时，有两种收尾方式：

```text
方式 A：cancel() 取消整条链
  main break → cancel() → 各阶段 select 走 Done 分支 → 全部退出

方式 B：继续把上游读空（drain）
  main break 后仍 range 剩余数据丢弃 → 上游能发完 → 各阶段自然 close
```

```go
// 方式 B 的收尾：把剩余数据读掉，避免上游永久阻塞
for range out {
}
```

⚠ **关键认知**：如果既 `break` 又不 `cancel` 又不 drain，上游阶段就会永久阻塞在 `out <- v`，它所在的 goroutine 永远不退出，连同它持有的内存一起泄漏。这是 pipeline 最典型的泄漏场景，第 8 篇会用 `goleak` 检测它。

# 6. 常见错误

```text
1. 某个阶段忘了 defer close(out)     → 下游 range 永远不结束
2. 下游关闭上游 channel               → send on closed channel
3. 阶段发送不监听 ctx.Done()          → 下游提前退出时上游泄漏
4. error channel 无缓冲且无人读        → 阶段卡在发送错误
5. 消费者 break 后既不取消也不排空     → goroutine 泄漏
6. 在阶段之间共享可变切片              → 竞态，违背 pipeline 的隔离原则
7. 阶段数量过多、每段只做一点点        → goroutine 开销大于收益
```

★ **关键认知**：pipeline 的正确性可以用一句话检查——**每条 channel 都有明确的拥有者、明确的关闭点、明确的取消路径**。画一张图，把每个阶段的 in/out、谁 close、ctx 传到哪，标出来，漏洞会立刻现形。

# 7. 练习

**练习 1**：实现一条 `generate → double → sum` 的流水线，最后输出所有数的两倍之和。要求每个阶段都能被 `context` 取消。

**练习 2**：给练习 1 加上错误处理：`double` 阶段遇到负数时，把错误写入独立的 error channel 并停止该阶段，主程序打印错误后退出。

**练习 3**：在练习 1 的消费者里，读到第 3 个结果就 `break`。先用"不 cancel 也不 drain"的方式运行，观察发生了什么（提示：用 `runtime.NumGoroutine()` 看 goroutine 数量）；再改成 `defer cancel()`，对比结果。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 阶段约定 | 收 `<-chan`、返 `<-chan`、内部起 goroutine |
| 关闭传播 | 每个 channel 由发送方 `defer close`，逐级向下游传递 |
| range | 关闭且取空后自动结束，但不感知取消 |
| 错误处理 | 独立 error channel（带缓冲）或 errgroup |
| 取消 | 共享 ctx，每个发送/接收都 `select` 监听 Done |
| 早停 | 必须 cancel 或 drain，否则上游泄漏 |
| 检查方法 | 每条 channel 都有拥有者、关闭点、取消路径 |

下一篇进入 fan-out/fan-in：当 pipeline 中某一阶段特别慢时，如何把任务分发给多个副本并行处理，再把结果汇聚回一条流，并处理乱序与结果聚合问题。
