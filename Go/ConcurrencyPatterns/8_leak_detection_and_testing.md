并发 bug 有两个讨厌的特点：**不一定复现**、**不报错**。数据竞争可能十次运行才中一次，goroutine 泄漏则完全静默——进程不崩、日志不报，只是内存和 goroutine 数稳步上涨，直到某天 OOM。这一篇是模块的"体检工具箱"：先认脸（常见泄漏场景），再学怎么查（pprof、goroutine dump、race detector），最后把它固化进测试。

# 1. 泄漏的脸谱

## 1.1 什么算泄漏

★ **关键认知**：goroutine 泄漏的判定标准只有一条——**这个 goroutine 是否还存在一条能退出它的路径**。如果它永久阻塞在一个再也不会有结果的 channel、锁或 `Sleep` 上，且没有 context 取消，那它就是泄漏。它不会被 GC 回收，因为运行时仍持有它的栈和闭包引用。

```go
func leak() {
    ch := make(chan int)
    go func() {
        ch <- 1 // 没有接收者，永远阻塞
    }()
    // 函数返回，ch 变成垃圾，但 goroutine 还挂在它上面
}
```

## 1.2 五类典型场景

| 场景 | 表现 | 根因 |
|------|------|------|
| 向无人接收的 channel 发送 | 栈停在 `chan send` | 下游提前退出 |
| 从无人发送的 channel 接收 | 栈停在 `chan receive` | 上游忘了 close |
| 上游阻塞、下游已退出 | 发送方永久阻塞 | 缺 cancel，也没 drain |
| Ticker 未 Stop | 栈停在 `runtime.timer` | 忘了 `ticker.Stop()` |
| 锁永不释放 | 栈停在 `sync.(*Mutex).Lock` | 分支提前 return 未 defer |

```go
// 场景 4：Ticker 未 Stop
func poll() {
    ticker := time.NewTicker(time.Second)
    // 缺少 defer ticker.Stop()
    for range ticker.C {
    }
}

// 场景 5：锁未释放
func (s *Store) Get(k string) int {
    s.mu.Lock()
    return s.data[k] // 缺少 Unlock，后续调用全部阻塞
}
```

★ **关键认知**：五类场景可以归结成一句话——**有阻塞，无退出条件**。排查时不要盯着代码找"哪里起了 goroutine"，而要盯着每个阻塞点问"它靠什么被唤醒"。

# 2. 发现泄漏

## 2.1 最土的指标

```go
import "runtime"

fmt.Println("goroutines:", runtime.NumGoroutine())
```

在操作前后各打印一次，如果数字只增不减，基本可以确认有泄漏。适合写进测试或临时日志，不适合作为长期监控的唯一手段（正常运行时 goroutine 数也会波动）。

## 2.2 pprof：看谁在跑

```go
import (
    "net/http"
    _ "net/http/pprof" // 匿名导入，注册 /debug/pprof 路由
)

go func() {
    http.ListenAndServe("localhost:6060", nil) // 仅限内网/本地
}()
```

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine   # 抓取当前栈
(pprof) top      # 按数量排序
(pprof) list main.worker
(pprof) traces   # 看完整调用栈
```

★ **关键认知**：`/debug/pprof/goroutine` 有两种视图：默认是聚合后的数量，加 `?debug=1` 或 `?debug=2` 可以看到**每一条 goroutine 的完整栈**。排查泄漏时用 `debug=2`，它和 SIGQUIT 的输出格式一致，能直接看到阻塞在哪个文件哪一行。

## 2.3 SIGQUIT：线上应急

向进程发送 `SIGQUIT`（Linux/macOS 下 `kill -QUIT <pid>`，Windows 下可用 `Ctrl+Break` 或调试器），运行时会打印**所有 goroutine 的栈**然后退出。

```bash
kill -QUIT 12345                 # 发送 SIGQUIT，打印全部栈后退出
GOTRACEBACK=all ./server         # panic 时输出全部 goroutine
```

⚠ **关键认知**：`SIGQUIT` 会让进程退出，只能用于"已经出问题、准备重启"的现场取证。日常排查优先用 pprof，它不会中断服务。抓到的 dump 要尽快保存，重启后就没了。

# 3. 读 goroutine 栈

## 3.1 栈帧长什么样

```text
goroutine 42 [chan send, 12 minutes]:
main.producer(0xc000014180)
    /app/pipeline.go:23 +0x4a
created by main.main in goroutine 1
    /app/pipeline.go:18 +0x9c
```

| 字段 | 含义 |
|------|------|
| `goroutine 42` | goroutine 编号 |
| `[chan send, 12 minutes]` | 当前状态与**已阻塞时长** |
| `main.producer` | 阻塞所在的函数 |
| `/app/pipeline.go:23` | 精确到行号 |
| `created by ...` | 谁启动了它 |

★ **关键认知**：**阻塞时长是最有价值的线索**。正常的 goroutine 阻塞几毫秒就恢复，如果一条栈显示 `10 minutes` 还停在同一行，它几乎肯定泄漏了。另外 `created by` 能帮你找到"谁该负责关掉它"。

## 3.2 常见状态速查

```text
chan send           → 发送方等接收方，检查下游是否退出
chan receive        → 接收方等发送方，检查上游是否 close
select              → 等待多个 channel
IO wait             → 网络/文件阻塞，检查是否传了 ctx
sync.Mutex.Lock     → 等锁，检查持锁者是否死锁
semacquire          → 等信号量/WaitGroup
sleep               → 定时器，检查是否可取消
```

## 3.3 一个真实的排查过程

```text
现象：服务运行 6 小时后 goroutine 从 200 涨到 30000

步骤：
1. pprof goroutine?debug=2 抓到 dump
2. 按 created by 分组，发现 29000 条来自 handleRequest
3. 栈统一停在 pipeline.go:23 的 ch <- v
4. 回看代码：下游超时后直接 return，没有 cancel 上游
5. 修复：共享 ctx，下游退出时 cancel()
```

★ **关键认知**：排查顺序永远是"**先按 created by 归类，再看共同阻塞点**"。几万条栈里通常只有一两种模式，找到模式就等于找到了根因。

# 4. race detector

## 4.1 用法

```bash
go test -race ./...
go run -race main.go
go build -race -o app.exe .
```

```go
var counter int

var wg sync.WaitGroup
for i := 0; i < 2; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        counter++ // 两个 goroutine 同时读改写
    }()
}
wg.Wait()
```

## 4.2 报告怎么读

```text
WARNING: DATA RACE
Write at 0x00c0000b4010 by goroutine 7:
  main.main.func1()  /app/main.go:10 +0x44

Previous write at 0x00c0000b4010 by goroutine 6:
  main.main.func1()  /app/main.go:10 +0x44

Goroutine 7 (running) created at:
  main.main()        /app/main.go:8 +0x84
```

★ **关键认知**：报告会给出**两次冲突访问的完整栈**，并且区分读（Read）和写（Write）。只要有一方是写，另一方是读或写，就构成竞态。修复方式通常是加锁、改用 atomic、或者把共享改成 channel 传递。

## 4.3 race detector 的局限

⚠ **关键认知**：race detector 是**动态检测**，只报告"本次运行中实际发生"的竞争。没被调度到的路径不会报，所以它在测试里的覆盖率取决于你跑得够不够多。常用做法是 `go test -race -count=100` 反复跑，或者配合压力测试。它还会让程序慢 5-20 倍、内存占用上升，不适合长期开着跑生产。

| 能力 | race detector | 静态检查 |
|------|---------------|----------|
| 发现真实发生的竞争 | 是 | 有限 |
| 发现潜在但未触发的竞争 | 否 | 部分 |
| 运行时开销 | 高 | 无 |
| 适合 | 测试、CI | 日常提交 |

# 5. 并发测试

## 5.1 表驱动 + t.Parallel

```go
func TestWorkerPool(t *testing.T) {
    cases := []struct {
        name           string
        workers, tasks int
    }{
        {"single", 1, 100},
        {"multi", 8, 1000},
        {"more_workers_than_tasks", 16, 4},
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel() // 子测试之间并行
            p := New(tc.workers, tc.workers)
            for i := 0; i < tc.tasks; i++ {
                if err := p.Submit(func() error { return nil }); err != nil {
                    t.Fatal(err)
                }
            }
            if errs := p.Close(); len(errs) > 0 {
                t.Fatalf("unexpected errors: %v", errs)
            }
        })
    }
}
```

★ **关键认知**：并发代码的测试要覆盖**边界**：worker 数为 1、任务数为 0、任务数小于 worker 数、队列容量为 0。这些边界最容易暴露关闭逻辑和 `WaitGroup` 计数的错误。`t.Parallel()` 会让多个测试同时跑，进一步提高竞态被触发的概率。

## 5.2 goleak：把泄漏变成测试失败

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m) // 整个包的测试结束后检查
}
```

或者在单个测试里：

```go
func TestPipeline(t *testing.T) {
    defer goleak.VerifyNone(t) // 测试结束时若还有可疑 goroutine 就失败

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    out := generate(ctx, 1, 2, 3)
    for range out {
    }
}
```

★ **关键认知**：`goleak` 会在测试结束时对比 goroutine 快照，忽略运行时自身的后台 goroutine，只报告"你启动却没退出"的那些。它把"泄漏"从线上事故变成了**CI 里的红色失败**，是并发代码最值得加的一个依赖。⚠ 使用时要确保被测代码真的能退出——如果测试本身就忘了 `cancel()`，`goleak` 会如实报错，这其实是好事。

## 5.3 压力与模糊测试

```bash
go test -race -run TestWorkerPool -count=100   # 反复运行，提高触发概率
go test -bench=. -benchmem -cpu=1,2,4,8         # 不同 GOMAXPROCS 下基准
go test -fuzz=FuzzParse -fuzztime=30s           # 模糊测试（Go 1.18+）
```

★ **关键认知**：`-cpu=1,2,4,8` 会让基准测试在不同 `GOMAXPROCS` 下各跑一遍，能暴露"单核没事、多核就崩"的竞态。`b.RunParallel` 则模拟多 goroutine 同时提交，比串行基准更接近真实压力。

# 6. 常见错误

```text
1. 只在本地跑过就说"没有竞态"        → 动态检测需要足够运行次数
2. 用 NumGoroutine 单点判断泄漏       → 正常波动会误报，要看趋势
3. pprof 端口暴露在公网               → 信息泄露，只绑 localhost
4. 测试里不 cancel，goleak 报错       → 先修测试，再修代码
5. 靠 Sleep 等待并发结果              → 不稳定，用 channel/WaitGroup 同步
6. race 构建带上生产                    → 性能不可接受
7. 忽略 goroutine 栈里的阻塞时长       → 错过最关键的线索
```

⚠ **关键认知**：第 5 条是并发测试的通病。`time.Sleep(100 * time.Millisecond)` 然后断言，在 CI 机器负载高时会随机失败。正确做法是用 channel 或 `WaitGroup` 做确定性同步，只在测试超时保护上用 `time.After`。

# 7. 练习

**练习 1**：复现 1.2 节的"向无人接收的 channel 发送"泄漏，用 `runtime.NumGoroutine()` 在前后各打印一次，再用 `go tool pprof` 的 `goroutine?debug=2` 找到对应栈。

**练习 2**：给 4.1 节的竞态例子加上 `-race` 运行，读报告并指出两次冲突访问的位置。然后用 `atomic.Int64` 修复，确认报告消失。

**练习 3**：给第 3 篇的 pipeline 示例加一个 `goleak.VerifyNone(t)` 的测试，先故意去掉 `defer cancel()`，确认测试失败；再补上，确认通过。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 泄漏本质 | 有阻塞、无退出条件 |
| 五类场景 | 发送/接收阻塞、上游未取消、Ticker 未停、锁未释放 |
| 快速判断 | `runtime.NumGoroutine()` 看趋势 |
| pprof | `/debug/pprof/goroutine?debug=2` 看完整栈 |
| 栈解读 | 状态 + 阻塞时长 + created by 是三条关键线索 |
| race | `go test -race -count=100`，动态检测，需多跑 |
| goleak | 把泄漏变成 CI 失败 |
| 测试 | 覆盖边界，用同步原语而非 Sleep |

下一篇是模块收尾：什么时候不该用并发、channel 与 mutex 如何取舍、常见反模式清单，以及一张按场景选模式的速查表。
