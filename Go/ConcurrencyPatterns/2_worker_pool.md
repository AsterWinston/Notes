当任务数量远大于机器能同时处理的上限时，最朴素的写法是"来一个任务起一个 goroutine"。在低负载下这没问题，但流量一上来，goroutine 数量、内存占用、下游连接数都会失控。worker 池用**固定数量的消费者**去消费一个**任务队列**，把并发从"随任务增长"变成"由你设定上限"。本篇从零实现一个可用的 worker 池，并补齐优雅关闭、错误收集和容量估算。

# 1. 为什么需要 worker 池

## 1.1 朴素写法的问题

```go
// 错误示范：每个任务一个 goroutine，无上限
func handle(tasks []Task) {
    for _, t := range tasks {
        go t() // 任务有多少，goroutine 就有多少
    }
}
```

这段代码有三个隐患：任务量突增时 goroutine 数量无上限；下游数据库、RPC 会被瞬间打满；没有等待机制，函数返回时任务可能还在跑。

```text
无上限并发：
任务 ──┬──► goroutine 1 ──► 下游
       ├──► goroutine 2 ──► 下游
       ├──► goroutine 3 ──► 下游
       └──► ...（数量不可控）

worker 池：
任务 ──► [有界队列] ──► worker 1 ──┐
                        worker 2 ──┼──► 下游（并发上限固定）
                        worker N ──┘
```

★ **关键认知**：worker 池的本质是**用队列做缓冲、用固定消费者做限流**。它同时解决两个问题：并发数量可控（不会打垮下游），以及 goroutine 复用（避免频繁创建销毁）。注意队列只是"缓冲"，不是"无限吸收"——队列满了以后生产者必须被阻塞或拒绝，这就是背压（第 7 篇）。

## 1.2 适用与不适用

| 场景 | 适合 worker 池 | 说明 |
|------|----------------|------|
| 大量短小的 I/O 任务 | 是 | 如批量调用 API、处理消息 |
| CPU 密集计算 | 是 | worker 数约等于核数 |
| 任务之间有严格先后依赖 | 否 | 应改用 pipeline（第 3 篇） |
| 任务需要动态扩缩容 | 谨慎 | 固定池更简单，弹性池复杂度高 |
| 单个长驻后台任务 | 否 | 直接起一个 goroutine 即可 |

# 2. 最小可用的 worker 池

## 2.1 结构设计

一个 worker 池只需要三样东西：任务队列、固定数量的 worker、一个等待全部 worker 退出的机制。落到结构体上就是 `jobs`（有界队列）、`wg`（等待组）、`closeOnce`（防重复关闭）、`mu` 与 `errs`（错误收集）、`closed`（关闭标志）。

★ **关键认知**：`jobs` 用**有界** channel，容量就是队列长度。无界队列在 Go 里不存在（硬要模拟就得自己维护切片加锁），而有界队列带来的阻塞正是背压的来源。`closeOnce` 保证重复调用 `Close` 不会 panic。

## 2.2 完整实现

```go
package pool

import (
    "fmt"
    "sync"
)

type Task func() error

type Pool struct {
    jobs      chan Task
    wg        sync.WaitGroup
    closeOnce sync.Once
    mu        sync.Mutex
    errs      []error
    closed    bool
}

// New 创建固定 workers 个消费者、队列容量为 queue 的池
func New(workers, queue int) *Pool {
    if workers <= 0 { workers = 1 }
    if queue < 0 { queue = 0 }
    p := &Pool{jobs: make(chan Task, queue)}
    p.wg.Add(workers)
    for i := 0; i < workers; i++ {
        go p.worker(i)
    }
    return p
}

func (p *Pool) worker(id int) {
    defer p.wg.Done()
    for task := range p.jobs { // 队列关闭且取空后循环自动结束
        if err := task(); err != nil {
            p.mu.Lock()
            p.errs = append(p.errs, fmt.Errorf("worker %d: %w", id, err))
            p.mu.Unlock()
        }
    }
}

// Submit 提交任务；池已关闭时返回错误而不是 panic
func (p *Pool) Submit(t Task) error {
    p.mu.Lock()
    if p.closed {
        p.mu.Unlock()
        return fmt.Errorf("pool is closed")
    }
    p.mu.Unlock()
    p.jobs <- t // 队列满时在这里阻塞，形成背压
    return nil
}

// Close 关闭队列并等待所有 worker 退出
func (p *Pool) Close() []error {
    p.closeOnce.Do(func() {
        p.mu.Lock()
        p.closed = true
        p.mu.Unlock()
        close(p.jobs) // 只有关闭方关闭队列
    })
    p.wg.Wait()
    p.mu.Lock()
    defer p.mu.Unlock()
    return append([]error(nil), p.errs...)
}
```

★ **关键认知**：`Submit` 用互斥锁保护 `closed` 标志，避免"检查通过后、发送前池被关闭"的竞态。⚠ 但这个实现有一个细微窗口：`Submit` 释放锁之后、执行 `p.jobs <- t` 之前，如果 `Close` 恰好关闭了 channel，发送仍会 panic。生产代码里通常用 `context` 或额外的 `done` channel 来彻底消除这个窗口，下面 4.2 会给出更稳妥的版本。

# 3. 任务队列的设计

## 3.1 队列容量怎么定

| 容量 | 行为 | 适用 |
|------|------|------|
| 0（无缓冲） | 提交必须等到有空闲 worker | 强背压，延迟最低 |
| 小（如 workers 的 1-2 倍） | 吸收短暂抖动 | 大多数场景 |
| 大 | 吸收突发流量，但内存占用高 | 上游突发、下游稳定 |
| 无限（自行实现） | 不阻塞生产者 | 危险，等于隐藏了过载 |

★ **关键认知**：队列容量决定的是"能容忍多久的生产者快于消费者"。容量越大，突发流量被吸收得越多，但**任务在队列里等待的时间也越长**，而且内存会随积压增长。多数场景下，队列长度取 worker 数的 1-2 倍就够，把过载显式暴露出来比悄悄堆在内存里更安全。

## 3.2 关闭时的边界

关闭队列必须满足两个条件：**没有新的发送者**、**发送动作已经全部完成**。这就是为什么通常由"拥有队列的池"来关闭，而不是由 worker 关闭。正确顺序是：标记 `closed` → `close(jobs)` → `wg.Wait()` → 返回错误。

⚠ **关键认知**：`close(jobs)` 之后 worker 仍会把**队列中剩余的任务**处理完，这称为"排空（drain）"。如果你希望取消时立即丢弃未处理任务，就不能只靠 `close`，而要让 worker 在 `ctx.Done()` 时提前返回，见下一节。

# 4. 优雅关闭

## 4.1 基础版：drain 后退出

上面 `Close` 的实现就是 drain 语义：先把队列里的任务做完，再退出。适合**关停时不能丢任务**的场景，比如订单落库、日志刷盘。

```go
p := New(4, 8)
for _, item := range items {
    it := item
    if err := p.Submit(func() error { return process(it) }); err != nil {
        break
    }
}
errs := p.Close() // 等所有已提交任务完成
```

## 4.2 带取消的版本

当上游要求"5 秒内必须停"，drain 可能超时，这时需要 `context`。

```go
func (p *Pool) worker(ctx context.Context, id int) {
    defer p.wg.Done()
    for {
        select {
        case <-ctx.Done():
            return // 立即放弃，不再取新任务
        case task, ok := <-p.jobs:
            if !ok { return } // 队列已关闭且取空
            if err := task(); err != nil {
                p.record(id, err)
            }
        }
    }
}

func (p *Pool) Submit(ctx context.Context, t Task) error {
    select {
    case p.jobs <- t:
        return nil
    case <-ctx.Done():
        return ctx.Err() // 队列满且上下文取消，放弃提交
    }
}
```

★ **关键认知**：`select` 同时等待"队列可写"和"上下文取消"，让提交方在队列长期满载时也能被打断，而不是永久阻塞。⚠ 取消时 worker 直接 `return`，**队列里未处理的任务会被丢弃**，所以取消版和 drain 版的语义不同，选哪个取决于业务能否容忍丢任务。

# 5. 错误收集

## 5.1 加锁切片 vs 错误 channel

| 方式 | 优点 | 缺点 |
|------|------|------|
| 互斥锁 + 切片 | 实现简单，结束时一次性返回 | 高频写入有锁竞争 |
| 带缓冲 error channel | 无锁写入 | 需要额外的收集 goroutine，缓冲满会阻塞 |
| `errgroup` | 自动聚合，支持取消 | 语义偏"任一失败即取消" |

错误 channel 的写法：

```go
errCh := make(chan error, workers) // 缓冲至少等于 worker 数
var collectWG sync.WaitGroup
collectWG.Add(1)
go func() {
    defer collectWG.Done()
    for err := range errCh {
        log.Printf("任务失败: %v", err)
    }
}()
close(errCh) // 所有 worker 结束后
collectWG.Wait()
```

★ **关键认知**：错误 channel 的缓冲至少要能容纳"所有 worker 同时各写一条错误"，否则 worker 可能因为写错误而阻塞，反过来拖慢任务处理。更好的做法是只记录**第一条错误**或**前 N 条**，避免错误风暴本身把系统压垮。

## 5.2 用 errgroup 简化

如果需求是"任意任务失败就取消其余任务"，`errgroup.WithContext` 比自己维护错误切片更合适：

```go
g, ctx := errgroup.WithContext(ctx)
sem := make(chan struct{}, 8) // 限制并发

for _, item := range items {
    g.Go(func() error {
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
        case <-ctx.Done():
            return ctx.Err()
        }
        return process(ctx, item)
    })
}
if err := g.Wait(); err != nil {
    log.Println("有任务失败:", err)
}
```

⚠ **关键认知**：`errgroup` 的 `Wait` 只返回**第一个**非 nil 错误，其余错误被丢弃。需要完整错误清单时，还是得自己聚合，第 6 篇会展开讲 `errgroup` 和 `semaphore`。

# 6. 容量估算

worker 数量不能拍脑袋。一个实用的估算起点是 **Little 定律**：

```text
并发数 ≈ 吞吐量（个/秒） × 单个任务平均耗时（秒）
```

例如目标每秒处理 500 个请求、每个请求平均耗时 20ms，则并发数约为 `500 × 0.02 = 10`。再根据任务类型调整：

| 任务类型 | worker 数建议 | 理由 |
|----------|---------------|------|
| CPU 密集 | 约等于 `GOMAXPROCS` | 超过核数只会增加切换开销 |
| I/O 密集 | 可远大于核数 | 大部分时间在等待 |
| 混合型 | 先按 Little 定律估，再压测微调 | 以实际 P99 延迟为准 |
| 下游有限流 | 不超过下游配额 | 否则只会制造超时 |

★ **关键认知**：估算只是起点，**最终以压测为准**。观察指标应该是队列长度、任务等待时间、下游错误率和 P99 延迟，而不是"worker 开得多不多"。如果队列长期接近满、等待时间持续上升，说明消费者不够或下游太慢；如果 worker 大部分空闲，说明开多了。

# 7. 常见错误

```text
1. 任务闭包捕获循环变量出错          → Go 1.22 前必须显式复制
2. Close 后仍调用 Submit             → panic 或返回错误，要有明确策略
3. 多个 goroutine 同时 close(jobs)   → panic，需 sync.Once
4. 错误 channel 无缓冲               → worker 写错误时阻塞
5. worker 里 panic 未捕获            → 整个进程崩溃（第 6 篇讲恢复）
6. 只 close 不 Wait                  → 主程序退出，任务丢失
7. 队列开得过大                      → 内存暴涨、延迟变长
```

⚠ **关键认知**：第 1 条在 Go 1.22 之后已经改变——循环变量改为每轮独立。但如果你维护的模块要求兼容更早版本，仍然要写 `item := item`，这是最稳妥的写法。

# 8. 练习

**练习 1**：实现一个 worker 池，提交 20 个任务，worker 数为 4，每个任务打印自己的编号和执行的 worker，观察输出中并发的任务数量是否始终不超过 4。

**练习 2**：给池加上"任务超时"能力：每个任务在独立的 goroutine 里执行，超过 2 秒就记录超时错误并继续。提示：用 `context.WithTimeout` 和 `select`。

**练习 3**：用 Little 定律估算：目标 QPS 为 1000、单任务平均耗时 50ms，需要多少并发？如果下游限流为每秒 300 次，worker 数应该取多少？写出你的推理。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 核心结构 | 有界任务队列 + 固定 worker + WaitGroup |
| 队列容量 | 小队列强背压，大队列吸突发但延迟高 |
| 优雅关闭 | 标记关闭 → close(jobs) → Wait，排空剩余任务 |
| 取消语义 | worker 监听 ctx.Done()，未处理任务会被丢弃 |
| 错误收集 | 加锁切片或带缓冲 channel；只留前 N 条防错误风暴 |
| 容量估算 | Little 定律起步，压测微调，看 P99 而非平均值 |
| 关键陷阱 | Close/Submit 竞态、重复 close、panic 未恢复 |

下一篇进入 pipeline：把一个大任务拆成多个阶段，用 channel 把阶段串起来，让数据像流水线一样流动，并处理关闭传播与取消。
