前面八篇讲了各种"怎么并发"的模式，这一篇反过来问一个更重要的问题：**什么时候不该并发**。并发是有成本的——它增加调度开销、放大不确定性、让调试和测试变难。真正成熟的判断不是"能用并发就用"，而是"只有当并发的收益明确大于它的复杂度时，才用"。本篇给出取舍原则、反模式清单和一张按场景选模式的速查表，作为整个模块的收束。

# 1. 什么时候不该用并发

## 1.1 并发的成本

| 成本 | 具体表现 |
|------|----------|
| 调度开销 | goroutine 创建、切换、channel 收发都有开销 |
| 复杂度 | 关闭、取消、错误传播都要显式处理 |
| 可调试性 | 栈变长、执行顺序不确定、问题难复现 |
| 资源放大 | 并发越高，连接数、内存、下游压力越大 |
| 正确性风险 | 竞态、死锁、泄漏，且往往在压力下才暴露 |

★ **关键认知**：并发只能解决两类问题——**等待重叠**（I/O 密集）和**利用多核**（CPU 密集）。如果一个任务既没有等待、也没有可并行的计算量，那么并发只会让它更慢。先问"瓶颈在哪"，再决定要不要并发。

## 1.2 判断清单

```text
出现以下任一情况，先别急着并发：
1. 任务本身只需要几微秒，创建 goroutine 的开销占比很高
2. 数据有强顺序依赖，拆开还得加锁排队
3. 下游容量有限，并发只会把排队从本地挪到下游
4. 逻辑还没跑通，先串行写对，再考虑优化
5. 没有明确的取消和关闭路径
6. 团队里没人能读懂这段并发代码
```

⚠ **关键认知**：第 4 条尤其重要。**先写对，再写快**。串行版本是并发版本最好的对照——有了正确的串行基线，你才能验证并发版本的结果一致，也才能测量并发到底带来了多少收益。上来就写并发，出了问题连"正确答案"都没有。

## 1.3 一个反例

```go
// 串行版本：清晰、可读、无并发问题
func sum(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// "并发"版本：为了并发而并发
func sumConcurrent(nums []int) int {
    var wg sync.WaitGroup
    var mu sync.Mutex
    total := 0
    for _, n := range nums {
        n := n
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock() // 每次加法都要抢锁
            total += n
            mu.Unlock()
        }()
    }
    wg.Wait()
    return total
}
```

★ **关键认知**：`sumConcurrent` 比串行版本慢得多——每个元素一个 goroutine，再加一把全局锁，开销远超加法本身。只有当"每个元素的计算足够重"时，才值得按块拆分（比如把切片切成 `GOMAXPROCS` 份，每份内部串行累加，最后合并）。**并发的粒度要足够粗，才可能摊平它的固定成本**。

# 2. channel vs mutex

## 2.1 决策原则

Go 社区的经典说法是：

> 不要通过共享内存来通信，而应该通过通信来共享内存。

但这句话常被误读成"永远用 channel"。更准确的原则是：

★ **关键认知**：**channel 用于传递所有权和编排流程，mutex 用于保护共享状态**。当数据在任一时刻只属于一个 goroutine、需要在它们之间"交接"时，用 channel；当多个 goroutine 确实需要同时读写同一块内存（比如缓存、计数器、配置）时，用 mutex 或 atomic。用 channel 模拟锁，通常又慢又难写。

## 2.2 对比表

| 维度 | channel | mutex / atomic |
|------|---------|----------------|
| 主要用途 | 传递数据、编排、通知 | 保护共享状态 |
| 所有权 | 明确，数据"搬家" | 共享，多方访问 |
| 阻塞语义 | 天然支持等待 | 需要条件变量配合 |
| 性能 | 有收发开销 | 无竞争时极快 |
| 可读性 | 流程清晰，但链长了难追 | 临界区短时直观 |
| 死锁风险 | channel 互相等待 | 锁顺序不一致 |
| 适合 | pipeline、worker 队列、取消信号 | 缓存、计数器、配置结构 |

## 2.3 同一问题的两种写法

需求：多个 goroutine 更新一个计数器，最后读取总值。

```go
// 写法一：mutex 保护共享状态（推荐）
type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.n
}
```

```go
// 写法二：channel 传递增量，由一个 goroutine 独占状态
func counter() (inc func(), value func() int) {
    ch := make(chan int)
    done := make(chan int)

    go func() {
        total := 0
        for delta := range ch {
            total += delta
        }
        done <- total
    }()

    inc = func() { ch <- 1 }
    value = func() int {
        close(ch) // 关闭后由上面的 goroutine 返回最终值
        return <-done
    }
    return inc, value
}
```

★ **关键认知**：写法二也能工作，但 `value` 只能调用一次、关闭逻辑分散、调用方要理解背后有个 goroutine。对于"简单计数器"这种需求，写法一明显更直接。**channel 的优势在流程编排，不在状态保护**；反过来，用 mutex 去实现 pipeline 的阶段衔接，也会写得非常别扭。

# 3. 常见反模式

## 3.1 清单

```text
1. 用 time.Sleep 做同步
2. goroutine 无上限、无关闭
3. 用 channel 当锁，或用锁模拟 channel
4. 在 goroutine 里 panic 不 recover
5. 忘记 close / cancel，造成泄漏
6. 用 context 传业务参数
7. 共享变量不加同步就跨 goroutine 读写
8. errgroup 里吞掉错误
9. 为了"看起来并发"而拆得稀碎
10. 关闭 channel 的责任不清
```

## 3.2 逐条说明

| # | 反模式 | 问题 | 正确做法 |
|---|--------|------|----------|
| 1 | 用 `time.Sleep` 做同步 | 负载一高就不准，测试随机失败 | 用 `WaitGroup` 或 channel 同步 |
| 2 | goroutine 无上限 | 瞬间打爆下游 | worker 池或 `SetLimit` |
| 3 | 用 channel 当锁 | 语义不清，容易死锁 | 用 `sync.Mutex` |
| 4 | goroutine 里 panic 不 recover | 整个进程崩溃 | 任务入口统一 `recover` |
| 5 | 忘记 `close` / `cancel` | 定时器、子节点、goroutine 泄漏 | `defer cancel()`，谁拥有谁关闭 |
| 6 | 用 context 传业务参数 | 类型不安全、难追踪 | 显式写进函数签名 |
| 7 | 共享变量裸读写 | 数据竞争 | 加锁或 `atomic` |
| 8 | errgroup 里吞错误 | 失败被静默 | 返回错误或显式记录 |
| 9 | 为了并发而拆得过碎 | 固定开销吃掉并行收益 | 按块拆分，保证单块工作量 |
| 10 | 关闭 channel 责任不清 | 重复关闭、向已关闭发送都会 panic | 单发送者关闭，多发送者由协调者关闭 |

★ **关键认知**：这十条里有八条最终都指向同一件事——**没有想清楚"谁拥有、谁关闭、失败了怎么办"**。写并发代码前先把这三个问题的答案写下来，比事后调试便宜得多。

⚠ **关键认知**：第 4 条和第 7 条最危险：前者让整个进程崩溃，后者让 bug 在压力下才偶发。前者靠统一 `recover` 兜底，后者靠 `go test -race` 暴露。

# 4. 模式选择表

| 场景 | 首选模式 | 关键原语 | 对应篇章 |
|------|----------|----------|----------|
| 一批独立任务、要限并发 | worker 池 | 有界 channel + WaitGroup | 第 2 篇 |
| 任务分成多个加工阶段 | pipeline | 多级 channel + close 传播 | 第 3 篇 |
| 某一阶段是瓶颈 | fan-out/fan-in | 多 worker + merge | 第 4 篇 |
| 请求要能取消/超时 | context | WithCancel/WithTimeout | 第 5 篇 |
| 一组 goroutine 要收集错误 | errgroup | `errgroup.WithContext` | 第 6 篇 |
| 精确控制同时运行数 | 信号量 | `SetLimit` / `semaphore` | 第 6 篇 |
| 控制调用速率 | 令牌桶 | `x/time/rate` | 第 7 篇 |
| 上游太快、要传压力 | 有界队列背压 | 阻塞写入或返回 429 | 第 7 篇 |
| 保护共享缓存/计数器 | mutex/atomic | `sync.Mutex` / `atomic.Int64` | 本篇 |
| 等待一次性初始化 | Once | `sync.OnceValue` | 第 1 篇 |
| 只是等待全部完成 | WaitGroup | `Add/Done/Wait` | 第 1 篇 |

★ **关键认知**：选模式的顺序是"**先看数据流，再看控制流，最后看错误与取消**"。数据是独立任务还是分阶段？控制上要不要限速、限额？失败时是继续还是全体取消？把这三个问题答清楚，模式基本就定了。

# 5. 工程实践清单

## 5.1 设计阶段

```text
1. 先写串行版本，确认正确性和基线性能
2. 明确瓶颈是 I/O 等待还是 CPU 计算
3. 确定并发上限：来自核数、下游配额、内存预算
4. 画出数据流：谁生产、谁消费、谁关闭、谁取消
5. 明确失败语义：丢任务还是重试，全体取消还是继续
```

## 5.2 编码阶段

```text
1. 每个 goroutine 都有明确的退出路径
2. ctx 作为第一个参数，逐层传递，不存结构体
3. 所有 cancel 都配 defer
4. channel 的拥有者负责关闭
5. 共享状态要么加锁，要么用 atomic，要么改为传递
6. 任务入口统一 recover，避免单点 panic 拖垮进程
7. 队列有界，满了有明确策略
8. 不用 Sleep 做同步
```

## 5.3 测试与上线

```text
1. go test -race -count=100 反复跑
2. 用 goleak 把泄漏变成测试失败
3. 覆盖边界：0 任务、1 worker、队列满、下游超时
4. 压测观察 P99 延迟、goroutine 数、队列长度
5. 生产开 pprof（仅内网），保留 goroutine dump 能力
6. 给关键并发路径加指标：队列深度、拒绝数、取消数
```

# 6. 一个综合示例

需求：处理一批 URL，要求全局 QPS 不超过 50、同时不超过 10 个、总超时 30 秒、任一失败即停止。

```go
func fetchAll(parent context.Context, urls []string) error {
    ctx, cancel := context.WithTimeout(parent, 30*time.Second)
    defer cancel()

    limiter := rate.NewLimiter(rate.Limit(50), 10) // 速率 + 突发
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // 并发上限

    for _, u := range urls {
        u := u
        g.Go(func() (err error) {
            defer func() { // panic 不拖垮进程
                if r := recover(); r != nil {
                    err = fmt.Errorf("panic on %s: %v", u, r)
                }
            }()

            if err := limiter.Wait(ctx); err != nil {
                return err // 等令牌时超时/取消
            }

            req, err := http.NewRequestWithContext(ctx, http.MethodGet, u, nil)
            if err != nil {
                return err
            }
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                return fmt.Errorf("请求 %s: %w", u, err)
            }
            defer resp.Body.Close()
            return nil
        })
    }
    return g.Wait()
}
```

★ **关键认知**：这二十几行几乎用到了本模块的全部要素——`context` 管超时与取消、`errgroup` 管错误与整体收敛、`SetLimit` 管并发、`rate.Limiter` 管速率、`recover` 管 panic、`%w` 保留错误链。模式不是孤立的知识点，而是这样组合起来解决一个真实问题的。

# 7. 练习

**练习 1**：找一个你写过的、用了 `go` 关键字的函数，用本篇 1.2 节的清单逐条检查：它的退出路径是什么？并发上限是多少？失败时会发生什么？把发现的问题记下来。

**练习 2**：把 2.3 节的两种计数器实现各写一遍，用 `go test -race` 验证都无竞态。然后思考：如果需求变成"每 100ms 输出一次当前总值"，哪种写法更自然？为什么？

**练习 3**：设计一个"图片批量压缩"服务：输入 1000 张图，CPU 4 核，要求不占满内存、可随时取消、失败要报告具体文件。写出你选择的模式组合（worker 池？pipeline？信号量？），并说明每个参数的理由。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 不该并发的信号 | 任务极短、强顺序依赖、下游有限、逻辑未验证 |
| 并发收益来源 | 等待重叠（I/O）与多核利用（CPU），二者都没有就别并发 |
| channel | 传递所有权、编排流程、广播取消 |
| mutex/atomic | 保护共享状态，临界区尽量短 |
| 反模式 | Sleep 同步、无界 goroutine、panic 不 recover、忘记 cancel |
| 选模式 | 先看数据流，再看控制流，最后看错误与取消 |
| 工程底线 | 有界、可取消、可关闭、可测试、可观测 |

九篇走下来，核心其实只有一句话：**并发是为了解决问题，不是为了展示技巧**。一个能说清楚"谁生产、谁消费、何时停、失败怎么办"的串行程序，远胜一个炫技但没人敢改的并发程序。

本模块到此结束。接下来可以回到 `Go/Introduction` 复习运行时调度与内存模型，或者进入 `Go/Etcd` 等实战模块，观察 context、worker 池和错误聚合在真实项目里是如何被使用的。
