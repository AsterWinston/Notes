`sync.WaitGroup` 能等一组 goroutine 结束，但它有两个短板：拿不到错误，也没法"一个失败就取消其他"。`errgroup` 正好补上这两点。而当并发数量需要精确控制时，光有 errgroup 还不够，还要配合信号量。本篇讲清楚 `errgroup`、`semaphore` 的用法与边界，以及一个绕不开的问题：**goroutine 里的 panic 不会被 errgroup 捕获**。

# 1. errgroup 基础

## 1.1 WaitGroup 的不足

```go
var wg sync.WaitGroup
var mu sync.Mutex
var firstErr error

for _, u := range urls {
    u := u
    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := fetch(u); err != nil {
            mu.Lock()
            if firstErr == nil { firstErr = err }
            mu.Unlock()
        }
    }()
}
wg.Wait()
```

这段代码能跑，但每次都要手写锁、判空、取消逻辑。`errgroup` 把这些收敛成一个类型。

| 能力 | WaitGroup | errgroup |
|------|-----------|----------|
| 等待全部完成 | 有 | 有 |
| 获取错误 | 无 | 有（第一个） |
| 首个错误自动取消 | 无 | 有（WithContext） |
| 限制并发数 | 无 | 有（SetLimit） |
| 捕获 panic | 无 | **无** |

## 1.2 最小示例

```go
func fetchAll(urls []string) error {
    var g errgroup.Group
    for _, u := range urls {
        u := u // Go 1.22 前需显式复制
        g.Go(func() error {
            resp, err := http.Get(u)
            if err != nil {
                return fmt.Errorf("请求 %s 失败: %w", u, err)
            }
            defer resp.Body.Close()
            return nil
        })
    }
    return g.Wait()
}
```

★ **关键认知**：`g.Go` 接受一个 `func() error`，内部起 goroutine 执行；`g.Wait` 阻塞到所有任务结束，并返回**第一个非 nil 错误**。不调用 `Wait` 就退出，任务照样会被进程终止，和裸 goroutine 一样。

# 2. WithContext 的取消语义

## 2.1 首个错误自动取消

```go
g, ctx := errgroup.WithContext(context.Background())
for _, item := range items {
    g.Go(func() error { return process(ctx, item) })
}
if err := g.Wait(); err != nil {
    log.Println("失败:", err)
}
```

★ **关键认知**：`errgroup.WithContext` 返回的 `ctx` 会在**任意一个任务返回非 nil 错误时被自动取消**，同时 `Wait` 返回那个错误。其余任务若能感知 `ctx.Done()`，就会提前退出。⚠ 这个取消**只在第一个错误发生时触发一次**；`Wait` 也只保留第一个错误，完整聚合见第 5 节。

## 2.2 取消不等于中断

⚠ **关键认知**：`errgroup` 的取消和 context 一样，只是发信号。如果 `process` 不检查 `ctx`，它不会因为别的任务失败而停下来。所以 `WithContext` 版本里的函数签名必须接收 `ctx`，并在阻塞处 `select` 监听：

```go
func process(ctx context.Context, item Item) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(time.Second):
        return nil
    }
}
```

# 3. 并发限制

## 3.1 SetLimit（Go 1.20+）

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(8) // 最多同时运行 8 个任务

for _, job := range jobs {
    g.Go(func() error {
        return handle(ctx, job)
    })
}
```

★ **关键认知**：`SetLimit(n)` 让 `g.Go` 在达到上限时**阻塞**，直到有空位。它限制的是"同时运行的任务数"，不是"排队的任务数"。`SetLimit(0)` 表示不限制。Go 1.21 起还有 `TryGo`：拿不到名额时立即返回 `false`，适合"满了就跳过/丢弃"的场景。

## 3.2 channel 信号量

在不方便用 `errgroup` 的地方，一个带缓冲 channel 就是最朴素的信号量：

```go
sem := make(chan struct{}, 8) // 容量即并发上限

for _, job := range jobs {
    sem <- struct{}{} // 获取名额，满了阻塞
    go func() {
        defer func() { <-sem }() // 释放名额
        handle(job)
    }()
}
// 收尾：等所有名额归还
for i := 0; i < cap(sem); i++ { sem <- struct{}{} }
```

★ **关键认知**：`sem <- struct{}{}` 是"获取"，`<-sem` 是"释放"，`struct{}` 不占内存。最后的收尾循环是在等所有名额归还，本质上是一种"用信号量当 WaitGroup"的技巧。⚠ 这种写法在任务 panic 时不会释放名额，必须用 `defer`。

## 3.3 官方 semaphore

```go
import "golang.org/x/sync/semaphore"

sem := semaphore.NewWeighted(10) // 支持加权，适合资源配额不均的场景

if err := sem.Acquire(ctx, 1); err != nil {
    return err // ctx 取消时放弃获取
}
defer sem.Release(1)

doWork()
```

| 方案 | 适合 | 注意 |
|------|------|------|
| `errgroup.SetLimit` | 任务用 errgroup 管理 | 最简单，推荐首选 |
| channel 信号量 | 轻量、无依赖 | 需手动 defer 释放 |
| `semaphore.Weighted` | 资源权重不同、需可取消 | 多一个依赖包 |

★ **关键认知**：`semaphore.Weighted` 的 `Acquire` 接收 `ctx`，因此**在等待名额时也能被取消**；普通 channel 信号量的发送是阻塞且不可取消的。需要"排队太久就放弃"时，用 Weighted 更合适。

# 4. panic 处理

## 4.1 errgroup 不会捕获 panic

```go
var g errgroup.Group
g.Go(func() error {
    panic("boom") // 整个进程崩溃，g.Wait 不会返回
})
_ = g.Wait()
```

★ **关键认知**：`errgroup` 只是把 `func() error` 放进 goroutine，**没有内置 recover**。goroutine 里的 panic 会一路向上，导致整个进程退出。这和主 goroutine 的 panic 行为一致——Go 没有"隔离某个 goroutine 的崩溃"的机制，除非你自己 `recover`。

## 4.2 统一的 recover 包装

```go
func safeGo(g *errgroup.Group, fn func() error) {
    g.Go(func() (err error) {
        defer func() {
            if r := recover(); r != nil {
                err = fmt.Errorf("panic: %v\n%s", r, debug.Stack())
            }
        }()
        return fn()
    })
}
```

之后用 `safeGo(g, func() error { return handle(job) })` 提交任务即可。

★ **关键认知**：`recover` 必须写在**发生 panic 的那个 goroutine 的 defer 里**。在外层 `Wait` 附近写 `recover` 是无效的，因为 panic 已经在另一个 goroutine 的栈上展开了。⚠ 恢复 panic 后，这个任务的返回值应当是一个错误，否则调用方会以为它成功了。

# 5. 错误聚合

## 5.1 errgroup 只给第一个

如果你需要"所有任务都跑完，再一次性报告全部错误"，`errgroup` 的默认语义不合适。可以自己收集：

```go
type MultiError struct {
    mu   sync.Mutex
    errs []error
}

func (m *MultiError) Add(err error) {
    if err == nil { return }
    m.mu.Lock()
    defer m.mu.Unlock()
    m.errs = append(m.errs, err)
}

func (m *MultiError) Err() error {
    m.mu.Lock()
    defer m.mu.Unlock()
    return errors.Join(m.errs...) // Go 1.20+
}

// 使用：任务返回 nil，避免 errgroup 取消其他任务
var g errgroup.Group
var me MultiError
for _, job := range jobs {
    job := job
    g.Go(func() error {
        me.Add(handle(job))
        return nil
    })
}
_ = g.Wait()
if err := me.Err(); err != nil {
    log.Println(err)
}
```

★ **关键认知**：这里刻意让闭包**返回 nil**，这样 `errgroup` 不会取消其他任务，实现"全部跑完再汇总"。如果你用了 `WithContext` 又返回错误，就会变成"首个失败即取消"，两种语义要按业务选。

## 5.2 errors.Join 的结构

```go
err := errors.Join(errA, errB)
fmt.Println(err)            // 多行错误文本
errors.Is(err, errSentinel) // 遍历所有被 Join 的错误
```

★ **关键认知**：`errors.Join` 返回的错误支持 `errors.Is` / `errors.As` 逐个匹配，是 Go 1.20 之后聚合多个错误的标准方式。⚠ 它的 `Error()` 输出是多行文本，适合日志，不适合直接展示给终端用户。

# 6. 组合示例：带限制与恢复的批量请求

```go
func fetchAll(ctx context.Context, urls []string, limit int) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(limit)

    for _, u := range urls {
        u := u
        g.Go(func() (err error) {
            defer func() {
                if r := recover(); r != nil {
                    err = fmt.Errorf("panic on %s: %v", u, r)
                }
            }()
            req, err := http.NewRequestWithContext(ctx, http.MethodGet, u, nil)
            if err != nil { return err }
            resp, err := http.DefaultClient.Do(req)
            if err != nil { return fmt.Errorf("请求 %s: %w", u, err) }
            defer resp.Body.Close()
            return nil
        })
    }
    return g.Wait()
}
```

★ **关键认知**：这个函数一次性组合了四件事——并发上限、请求级超时（`WithContext`）、panic 恢复、错误包装（`%w`）。这是生产代码里最常见的形态。注意 `SetLimit` 要写在 `Go` 之前，否则不生效。

# 7. 常见错误

```text
1. 用 errgroup 但函数不接收 ctx       → 取消形同虚设
2. 期望 Wait 返回全部错误             → 只返回第一个
3. 忘记处理 goroutine 内 panic        → 进程崩溃
4. SetLimit 写在 g.Go 之后            → 前几个不受限
5. 闭包捕获循环变量（Go 1.22 前）     → 全部处理最后一个元素
6. 信号量忘 defer 释放                → 名额耗尽，后续永久阻塞
7. 用 WithContext 却返回 nil 吞掉错误 → 取消不触发
```

⚠ **关键认知**：第 4 条容易被忽略。`SetLimit` 的作用是让**后续**的 `Go` 受限制，如果先启动了 100 个再 `SetLimit(8)`，那 100 个已经在跑了。正确顺序永远是"先设限，再提交"。

# 8. 练习

**练习 1**：用 `errgroup` 并发请求 5 个 URL，限制并发为 3，任意一个失败就取消其余请求。打印最终错误，并用 `errors.Is` 判断它是否是 `context.Canceled`。

**练习 2**：改造练习 1，要求"全部跑完再汇总错误"，使用 `errors.Join` 返回所有失败 URL 的错误。对比两次的行为差异。

**练习 3**：写一个 `safeGo` 包装器，让任务 panic 时返回包含堆栈的错误，而不是让进程崩溃。用一个必然 panic 的任务验证 `g.Wait()` 能正常返回。

# 9. 小结

| 主题 | 要点 |
|------|------|
| errgroup | `Go` 提交任务，`Wait` 等待并返回首个错误 |
| WithContext | 首个错误自动取消其余任务，但需函数主动监听 |
| SetLimit | 限制同时运行数，必须在 `Go` 之前调用 |
| 信号量 | channel 轻量但不可取消，Weighted 支持 ctx |
| panic | errgroup 不捕获，必须在任务内 `recover` |
| 错误聚合 | 默认只留第一个，全量用 `errors.Join` 自建收集 |
| 组合 | 限额 + 超时 + recover + `%w` 是生产标配 |

下一篇进入限流与背压：当上游速度持续高于下游处理能力时，如何用令牌桶、漏桶和有界队列保护系统不被压垮。
