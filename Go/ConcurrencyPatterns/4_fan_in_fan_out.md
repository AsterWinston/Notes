pipeline 跑起来后，常常会发现某一阶段特别慢，成为整条链的瓶颈。fan-out/fan-in 就是针对这个问题的模式：把瓶颈阶段**复制多份并行处理**（fan-out），再把多路结果**汇聚回一条流**（fan-in）。它看起来只是"多开几个 goroutine"，但真正难的是分发是否均匀、汇聚后顺序是否还正确、结果怎么不失不重地聚合。本篇把这三点讲透。

# 1. 概念与动机

## 1.1 从瓶颈说起

```text
单副本 pipeline：
generate ──► [ 慢阶段 ] ──► sink
              ↑ 所有数据排队等它

fan-out 后：
             ┌──► 慢阶段副本 1 ──┐
generate ────┼──► 慢阶段副本 2 ──┼──► sink
             └──► 慢阶段副本 3 ──┘
                   （并行处理，再汇聚）
```

★ **关键认知**：fan-out 的本质是**用更多消费者分担同一个输入 channel**；fan-in 的本质是**把多个输出 channel 合并成一个**。两者通常成对出现：不 fan-in 的话，下游就得同时读 N 条 channel，复杂度会扩散到调用方。

## 1.2 适用条件

| 条件 | 说明 |
|------|------|
| 任务彼此独立 | 有先后依赖的任务不能乱序并行 |
| 阶段是瓶颈 | 不是瓶颈的阶段 fan-out 只会增加开销 |
| 结果可合并 | 需要一个明确的聚合规则 |
| 能容忍乱序，或能重排 | 并行必然打乱完成顺序 |

⚠ **关键认知**：fan-out 不会让"总量固定且互相依赖"的任务变快。比如"先查用户、再查订单、再算总价"是一条有依赖的链，拆成三个副本并行毫无意义。只有当**同一个阶段要处理很多条独立数据**时，多副本才有收益。

# 2. fan-out：分发

## 2.1 共享一个输入 channel

最常见也最简单的 fan-out：多个 worker 从**同一个 channel** 读取。

```go
func worker(ctx context.Context, id int, jobs <-chan Job, out chan<- Result) {
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-jobs:
            if !ok {
                return // 输入关闭，本副本退出
            }
            out <- process(job)
        }
    }
}

func fanOut(ctx context.Context, jobs <-chan Job, n int) []chan Result {
    outs := make([]chan Result, n)
    for i := 0; i < n; i++ {
        outs[i] = make(chan Result)
        go worker(ctx, i, jobs, outs[i])
    }
    return outs
}
```

★ **关键认知**：channel 本身就是并发安全的队列，多个 worker 同时 `<-jobs` 时，运行时会保证**每个元素只被一个 worker 取走**。这就是"共享 channel 即天然分发"，不需要额外的调度器。分发顺序是不确定的，但每个任务恰好被处理一次。

## 2.2 两种分发策略

| 策略 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| 共享 channel | 所有 worker 读同一个 ch | 实现简单，自动负载均衡 | 无法指定任务给特定 worker |
| 显式分片 | 每个 worker 一个 ch，按 key 哈希 | 可保证同一 key 落到同一 worker | 需要自己分发，可能不均 |
| 轮询分发 | 主 goroutine 依次发给各 ch | 顺序可控 | 某个 worker 慢会拖住整体 |

显式分片常用于"同一个订单的多次操作必须串行"的场景：

```go
func shard(key string, n int) int {
    h := fnv.New32a()
    h.Write([]byte(key))
    return int(h.Sum32()) % n
}
```

★ **关键认知**：`hash(key) % n` 能把同一个 key 稳定地映射到同一个 worker，从而在 worker 内部用普通变量保存状态，不必加锁。这正是很多分库分表、按会话保序的实现思路。⚠ 但分片数 `n` 变化时映射会整体改变，需要 rehash 或一致性哈希。

# 3. fan-in：汇聚

## 3.1 正确的 merge 写法

多个 worker 往**同一个** out channel 写，再由一个 goroutine 在全部结束后关闭它。

```go
func merge(ctx context.Context, ins ...<-chan Result) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    wg.Add(len(ins))

    for _, in := range ins {
        in := in // Go 1.22 前需显式复制
        go func() {
            defer wg.Done()
            for v := range in {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait() // 等所有输入都读完
        close(out) // 只有一个 goroutine 负责关闭，安全
    }()
    return out
}
```

★ **关键认知**：**不能**在任何一个 worker 里 `close(out)`。有多个发送者时，某个 worker 先关闭，其他 worker 再写就会 panic。正确做法是引入一个"协调者"：等所有发送者都通过 `WaitGroup` 报告完成后，由它单独 `close(out)`。这是 Go 并发里非常通用的收尾套路。

## 3.2 为什么不能直接读多个 channel

如果只 `range ins[0]`，其余 channel 永远不会被读，上游会永久阻塞。要同时读多条 channel，要么用 `select`（但 case 数量必须编译期固定），要么用上面的 merge 把多条合成一条。⚠ 如果只用 `select` 读固定几条 channel，当某条关闭后要记得把它从 select 中移除，否则会一直收到零值，形成忙循环。

# 4. 顺序问题

## 4.1 并行必然乱序

例如输入 1、2、3 的处理耗时分别是 30ms、10ms、20ms，输出顺序可能是 2、3、1。

| 场景 | 是否需要保序 |
|------|--------------|
| 日志聚合、计数求和 | 不需要 |
| 批量写库（无外键依赖） | 通常不需要 |
| 按序输出、增量渲染 | 需要 |
| 有状态的状态机 | 需要（或按 key 分片保序） |

★ **关键认知**：fan-out 之后，**完成顺序由处理耗时决定，而不是输入顺序**。如果业务要求输出与输入同序，就必须额外付出代价：给每个任务编号，在汇聚端缓存乱序结果，等缺口补齐后再按序发出。

## 4.2 编号 + 重排缓冲

```go
type seqResult struct {
    seq   int
    value string
}

// reorder 把乱序到达的结果按 seq 还原成有序输出
func reorder(ctx context.Context, in <-chan seqResult) <-chan seqResult {
    out := make(chan seqResult)
    go func() {
        defer close(out)
        pending := make(map[int]seqResult) // 已到达但还不能发出的结果
        next := 0                          // 下一个应该发出的序号
        for r := range in {
            pending[r.seq] = r
            for { // 连续地吐出所有已就绪的序号
                v, ok := pending[next]
                if !ok {
                    break
                }
                delete(pending, next)
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
                next++
            }
        }
    }()
    return out
}
```

```text
到达顺序：3, 1, 2
pending 变化：
收到 3 → {3}            next=0，不能发
收到 1 → {3,1}          next=0，仍缺 0
收到 2 → {3,1,2}        next=0，仍缺 0
（假设还有 0）…
收到 0 → {3,1,2,0}      依次发出 0,1,2,3
```

★ **关键认知**：重排缓冲会**内存随乱序程度增长**。如果某个序号的任务卡住或丢失，后续所有结果都会堆在 `pending` 里发不出去，形成"队头阻塞"。所以保序方案必须配合超时或取消，确保不会因为一个慢任务拖死整条流。

# 5. 结果聚合

## 5.1 三种聚合方式

| 方式 | 写法 | 适用 |
|------|------|------|
| 互斥锁 + 切片/map | 各 worker 直接写共享结构 | 结果量中等，逻辑简单 |
| 结果 channel + 单收集者 | 所有结果进一个 ch，一个 goroutine 收集 | 与 fan-in 天然契合 |
| 原子量 | 只做计数、求和 | 聚合规则是简单运算 |

## 5.2 结果 channel 收集

```go
func collect(ctx context.Context, results <-chan Result) ([]Result, error) {
    var all []Result
    for {
        select {
        case <-ctx.Done():
            return all, ctx.Err()
        case r, ok := <-results:
            if !ok {
                return all, nil
            }
            all = append(all, r)
        }
    }
}
```

## 5.3 map 聚合（需要去重或分组时）

```go
type Aggregator struct {
    mu   sync.Mutex
    data map[string][]Result
}

func (a *Aggregator) Add(key string, r Result) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.data[key] = append(a.data[key], r)
}
```

★ **关键认知**：如果聚合只是"求和、计数"，用 `atomic.Int64` 最省事；如果需要保留明细或分组，用带锁的 map；如果结果要流式交给下游，就保持 channel 形态不要落地成切片。⚠ 不要在多个 worker 里同时 `append` 同一个切片而不加锁——切片的底层数组扩容会引发数据竞争和丢数据。

# 6. 错误与取消

fan-out/fan-in 里错误处理和 pipeline 一样，推荐用 `errgroup` 统一管理：

```go
func run(ctx context.Context, jobs <-chan Job, n int) error {
    g, ctx := errgroup.WithContext(ctx)
    results := make(chan Result, n) // 缓冲足够容纳各 worker 的产出

    for i := 0; i < n; i++ {
        g.Go(func() error {
            return worker(ctx, jobs, results) // 内部处理，失败时返回 error
        })
    }

    go func() {
        _ = g.Wait()
        close(results) // 所有 worker 结束后由协调者关闭
    }()

    for range results { // 消费结果
    }
    return g.Wait()
}
```

★ **关键认知**：`errgroup.WithContext` 返回的 `ctx` 会在**任意一个任务返回错误时自动取消**，其余 worker 的 `select` 会走 `Done` 分支退出。这就是"一处失败、全局收敛"的标准写法。⚠ 取消后未处理的任务会被丢弃，若业务要求不丢，需要在退出前 drain 输入队列。

# 7. 常见错误

```text
1. 在 worker 里 close(out)          → 其他 worker 写入 panic
2. 多个 goroutine 同时 append 切片  → 丢数据、竞态
3. 忘记关闭输入 channel             → merge 永远等不到 wg.Wait
4. fan-out 后不 fan-in              → 调用方要管理 N 条 channel
5. 用 select 读固定 channel 且不处理关闭 → 忙循环收零值
6. 保序缓冲无上限                   → 一个慢任务拖垮内存
7. worker 数远超下游承受能力         → 打爆数据库
```

⚠ **关键认知**：第 1 条是最常见的 panic 来源。记住一个口诀：**单发送者由发送者关闭，多发送者由协调者关闭，接收者永不关闭**。

# 8. 练习

**练习 1**：实现 `generate → 3 个 worker 并行平方 → merge`，把 1 到 10 的平方输出。多运行几次，观察输出顺序是否固定。

**练习 2**：在练习 1 的基础上给每个任务加序号，用重排缓冲让最终输出严格按输入顺序。人为让偶数任务 `Sleep` 更久，验证重排逻辑生效。

**练习 3**：把练习 1 的结果改为按"奇偶"分组聚合，返回 `map[string][]int`。分别用"互斥锁 map"和"结果 channel + 单收集者"两种方式实现，比较代码复杂度。

# 9. 小结

| 主题 | 要点 |
|------|------|
| fan-out | 多个 worker 读同一 channel，天然负载均衡 |
| 显式分片 | 按 key 哈希，保证同 key 串行 |
| fan-in | 多路写入同一 out，由协调者统一 close |
| 关闭规则 | 多发送者时，等全部完成后再由第三方关闭 |
| 顺序 | 并行必乱序，需编号 + 重排缓冲，代价是内存与队头阻塞 |
| 聚合 | 锁 + 切片/map、结果 channel、原子量按需选择 |
| 取消 | errgroup 一处失败全局取消，注意丢任务语义 |

下一篇进入 context 与取消：把"超时、取消传播、避免 goroutine 泄漏"讲清楚，为后面所有模式提供统一的退出机制。
