context 是 Go 里传递"取消、超时、请求级元数据"的统一载体。前几篇的模式解决的是"怎么把任务组织起来"，这一篇解决的是"怎么让它们体面地停下来"。停不下来比跑不起来更危险：一个卡在 channel 上的 goroutine 会带着它引用的所有内存一起泄漏，而且不会报错，只会在监控曲线上慢慢显现。本篇从 context 的树结构讲到取消传播，再到最常见的泄漏场景，给出一套可复制的退出模板。

# 1. context 的模型

## 1.1 Context 接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) // 截止时间，ok 表示是否设置
    Done() <-chan struct{}                   // 取消时关闭的 channel
    Err() error                              // 取消原因
    Value(key any) any                       // 请求级数据
}
```

★ **关键认知**：`Done()` 返回的 channel 是 context 的**核心**。它只有两种状态：未关闭（还在运行）和已关闭（已取消）。所有"是否该退出"的判断，本质都是 `select { case <-ctx.Done(): }`。`Err()` 在 `Done` 关闭后才返回非 nil，取值只有三个：`context.Canceled`、`context.DeadlineExceeded`、或者用 `WithCancelCause` 设置的自定义原因。

## 1.2 树结构与传播

```text
              Background
                  │
          WithTimeout(3s)
             /          \
   WithCancel(c1)    WithValue(traceID)
        │                  │
   WithCancel(c2)     WithCancel(c3)
```

```text
取消 c1：
1. c1 的 Done channel 关闭
2. c1 递归取消它的所有子节点 c2
3. 与 c1 平级的 c3 不受影响

取消根节点：
所有后代全部取消
```

★ **关键认知**：context 是**单向的树**——父取消会传播到所有子，子取消**不会**影响父或其他分支。因此同一个请求下不同子任务可以各自取消，而顶层仍然掌控全局。⚠ 这种树结构也意味着**每创建一个派生 context 都会挂到父节点上**，如果忘记 `cancel()`，父节点会一直持有这个子节点的引用，直到父节点自己被取消。

# 2. 创建与派生

## 2.1 根 context

```go
ctx := context.Background() // 程序根节点，永不取消、无值
```

★ **关键认知**：`Background` 是程序的根，用于 `main`、初始化、测试入口这些真正的起点。⚠ 不要在业务函数里用 `Background()` 凭空造一个根，那会切断取消传播——上层取消了，下层因为拿着另一个根而继续跑。

## 2.2 三种派生方式

```go
// 1. 手动取消
ctx, cancel := context.WithCancel(parent)
defer cancel() // 用完必须调用，释放子节点与定时器

// 2. 超时
ctx, cancel := context.WithTimeout(parent, 3*time.Second)
defer cancel()

// 3. 截止时间
ctx, cancel := context.WithDeadline(parent, time.Now().Add(time.Minute))
defer cancel()

// 4. 携带取消原因（Go 1.20+）
ctx, cancel := context.WithCancelCause(parent)
defer cancel(nil)
cancel(fmt.Errorf("用户主动退出"))
fmt.Println(context.Cause(ctx)) // 用户主动退出
```

★ **关键认知**：`cancel()` 不是"可选的清理"，而是**必须执行的资源释放**。`WithTimeout` 内部会注册一个定时器，`WithCancel` 会把子节点挂到父节点；不调用 `cancel`，这些资源会存活到父节点结束。这就是为什么几乎所有 context 代码都紧跟一行 `defer cancel()`。

## 2.3 WithValue 的边界

```go
type ctxKey string

const traceKey ctxKey = "trace-id"

ctx = context.WithValue(ctx, traceKey, "abc123")
id, ok := ctx.Value(traceKey).(string)
```

★ **关键认知**：`WithValue` 只用于**请求级元数据**（trace ID、用户身份、租户），不用于传业务参数。key 必须用自定义类型而不是裸 `string`，避免不同包之间冲突。⚠ 值会沿整棵树一路查找到根，深度越大越慢；而且类型断言失败会得到零值，容易静默出错。能用显式参数表达的就不要塞进 context。

# 3. 取消传播

## 3.1 关闭 channel 就是广播

```go
ctx, cancel := context.WithCancel(context.Background())

for i := 0; i < 3; i++ {
    go func(id int) {
        <-ctx.Done() // 所有等待者同时被唤醒
        fmt.Println("worker", id, "退出:", ctx.Err())
    }(i)
}

time.Sleep(100 * time.Millisecond)
cancel() // 一次调用，广播给所有监听者
time.Sleep(100 * time.Millisecond)
```

★ **关键认知**：Go 里"关闭 channel"天然是**一对多广播**：所有阻塞在该 channel 上的接收者都会立即被唤醒，而且关闭后的接收永远返回零值。context 正是利用这一点实现取消广播，因此**不需要为每个 goroutine 单独发一个取消消息**。

## 3.2 传播到每一层

```go
func handler(ctx context.Context) error {
    return service(ctx) // 原样往下传
}

func service(ctx context.Context) error {
    return repo(ctx) // 继续往下传
}

func repo(ctx context.Context) error {
    rows, err := db.QueryContext(ctx, "SELECT ...")
    if err != nil {
        return fmt.Errorf("查询失败: %w", err) // 包装但保留原因
    }
    defer rows.Close()
    return nil
}
```

★ **关键认知**：context 的取消能否"贯穿调用链"，取决于**每一层是否把它继续往下传**。标准库和主流框架的 I/O 方法几乎都有 `Context` 变体（`QueryContext`、`NewRequestWithContext`、`DialContext`），传下去之后，底层驱动会监听同一个 `Done`，实现真正的及时中断。⚠ 用 `fmt.Errorf("...: %w", err)` 包装错误，才能让上层用 `errors.Is(err, context.DeadlineExceeded)` 判断取消原因。

# 4. 超时控制

## 4.1 基本用法

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

result, err := slowOperation(ctx)
if err != nil {
    switch {
    case errors.Is(err, context.DeadlineExceeded):
        log.Println("超时了")
    case errors.Is(err, context.Canceled):
        log.Println("被主动取消")
    default:
        log.Println("其他错误:", err)
    }
}
```

## 4.2 超时预算会取最小值

```text
父 context 剩余 2s
    └── 子 WithTimeout(5s)  → 实际 2s 生效（父先到）
    └── 子 WithTimeout(1s)  → 实际 1s 生效（子先到）
```

★ **关键认知**：子 context 的截止时间**不会晚于父 context**。派生时会取两者中较早的那个，这叫"超时预算的继承"。这个设计保证了整体的 SLA 不被某个下层私自延长。给下游调用设置超时时，通常写成比上层剩余时间更短的值，留出错误处理和返回的余量。

## 4.3 超时不是万能的

⚠ **关键认知**：`context` 只能发出"该停了"的信号，**不能强制杀死 goroutine**。如果被调用的函数不检查 `ctx.Done()`、也不把 ctx 传给底层 I/O，那么超时时间到了它也照跑不误，`<-ctx.Done()` 返回了但工作仍在后台继续。这不是 context 的缺陷，而是约定：**每个可能长时间阻塞的操作都必须主动监听取消**。

```go
// 错误示范：ctx 传进来了却没用
func work(ctx context.Context) {
    time.Sleep(10 * time.Second) // 不理会 ctx，超时也停不下来
}

// 正确做法：把等待变成可取消的 select
func work(ctx context.Context) error {
    select {
    case <-time.After(10 * time.Second):
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

# 5. 避免 goroutine 泄漏

## 5.1 三种典型泄漏

```go
// 泄漏 1：发送方永远阻塞，因为没人接收
func leak1() {
    ch := make(chan int)
    go func() { ch <- 1 }() // 卡在这里，goroutine 永不退出
}

// 泄漏 2：接收方永远阻塞，因为没人发送也不关闭
func leak2() {
    ch := make(chan int)
    go func() {
        for v := range ch { // 卡在等数据
            _ = v
        }
    }()
}

// 泄漏 3：下游提前退出，上游还在发送
func leak3() {
    ch := make(chan int)
    go func() {
        for i := 0; ; i++ {
            ch <- i // 消费者只读一个就走了，这里永远阻塞
        }
    }()
    <-ch
}
```

★ **关键认知**：goroutine 泄漏的判定标准是"**这个 goroutine 是否还有可能退出**"。只要它阻塞在一个永远不会有结果的 channel 操作、锁、或者 `time.Sleep` 上，且没有任何取消路径，它就是泄漏。它不会被 GC 回收，因为它还在运行栈上，连它引用的闭包变量也一起留着。

## 5.2 可取消的模板

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return // 取消路径
        case job, ok := <-jobs:
            if !ok {
                return // 输入关闭路径
            }
            process(ctx, job)
        }
    }
}

func producer(ctx context.Context) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out) // 无论正常结束还是取消，都关闭
        for i := 0; ; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

★ **关键认知**：一个不会泄漏的 goroutine，**每条阻塞路径都必须有一个对应的退出条件**。检查方法很机械：列出这个 goroutine 里所有的 `<-ch`、`ch <- v`、`Lock()`、`Sleep()`，逐个问"如果永远不满足，它能被什么唤醒"。答案要么是 channel 被关闭，要么是 `ctx.Done()`。

## 5.3 用监控兜底

在测试里对比操作前后的 `runtime.NumGoroutine()`，是发现泄漏最直接的手段。生产环境可以定期上报这个指标，观察它是否随请求量单调上升。第 8 篇会讲 `goleak` 和 `pprof` 的完整用法。

# 6. 常见错误

```text
1. 忘记 defer cancel()                 → 定时器和子节点泄漏
2. 用 Background() 另起炉灶            → 切断取消传播
3. 把 context 存进结构体字段           → 生命周期混乱，难以追踪
4. 传 nil context                      → 直接 panic
5. 用 string 作 WithValue 的 key       → 跨包冲突
6. 只判断 Done 不读 Err()              → 分不清超时与取消
7. 在被调函数里不监听 ctx.Done()       → 超时形同虚设
8. 循环里反复 WithTimeout 但不 cancel  → 定时器堆积
```

⚠ **关键认知**：第 3 条值得展开。context 应该作为**第一个参数显式传递**，而不是塞进结构体。存进结构体后，这个对象的生命周期就和某次请求绑定了，后续复用时会带着过期的取消状态，而且调用方无法从签名看出它依赖 context。

# 7. 练习

**练习 1**：写一个 `slowTask(ctx)`，内部用 `select` 等待 5 秒或 `ctx.Done()`。用 `WithTimeout(1s)` 调用它，确认 1 秒后返回 `context.DeadlineExceeded`。

**练习 2**：构造一个三层的 context 树（根 → 两个子 → 其中一个子再派生一个孙），取消根，验证所有后代都被取消，并用 `context.Cause` 打印原因。

**练习 3**：复现 5.1 节的 `leak1`，在函数返回后打印 `runtime.NumGoroutine()`，再改成带 `ctx` 的版本，对比两次的 goroutine 数量。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 核心 | `Done()` 关闭即取消，一对多广播 |
| 树结构 | 父取消传播到全部后代，子取消不影响父 |
| 派生 | WithCancel / WithTimeout / WithDeadline / WithValue |
| cancel | 必须 `defer cancel()`，释放定时器与子节点 |
| 超时预算 | 子截止时间取父与子的较早者 |
| 传播 | 每一层都要把 ctx 往下传，用 `%w` 保留原因 |
| 泄漏 | 每条阻塞路径都要有退出条件 |
| 判断原因 | `errors.Is` + `context.Canceled/DeadlineExceeded` |

下一篇进入 errgroup 与 semaphore：在 context 的基础上，如何用 `errgroup` 统一管理一组 goroutine 的错误与取消，并用信号量精确控制并发上限。
