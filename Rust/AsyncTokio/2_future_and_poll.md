上一篇把 `async/await` 当成黑盒用了：写起来像同步，跑起来不阻塞。这一篇把盒子拆开，看 `Future` 到底长什么样、`poll` 返回什么、`Waker` 为什么是异步的心脏、`Pin` 又在保护什么。内容偏底层，但只有理解了这四件套，后面调试"任务不唤醒""Future 卡死"时你才知道该看哪里。

# 1. Future trait 的真面目

## 1.1 标准库里的定义

`Future` 在标准库中只有一个必需方法：

```rust
pub trait Future {
    type Output;   // 完成时产出的值

    // 推进这个 Future：就绪返回 Ready，未就绪返回 Pending
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),    // 已完成，携带结果
    Pending,     // 还没好，稍后再 poll
}
```

就这么点东西。没有 `await`，没有线程，没有运行时——`Future` 只描述"如何被推进一次"。

★ **关键认知**：`Future` 是一个**状态机接口**，不是"一个在后台跑的任务"。每次 `poll` 相当于问它一句"你好了吗"；它内部保存着自己的状态，可以返回 `Pending` 无数次，直到某次返回 `Ready`。⚠ 一旦返回 `Ready`，再次 `poll` 属于违反契约，可能 panic 或产生未定义行为，所以执行器不会再 poll 已完成的 Future。

## 1.2 手写一个 Future

理解了定义，就能手写一个最简单的 Future：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct ReadyNow(u32);

impl Future for ReadyNow {
    type Output = u32;

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<u32> {
        Poll::Ready(self.0)    // 第一次 poll 就绪
    }
}
```

这个 Future 第一次被 `poll` 就返回 `Ready`；驱动它的完整执行器将在 3.3 节给出。先看一个会返回 `Pending` 的例子：计数器到 3 次才就绪。

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct Countdown {
    remaining: u8,
}

impl Future for Countdown {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<&'static str> {
        if self.remaining == 0 {
            Poll::Ready("发射")
        } else {
            self.remaining -= 1;
            cx.waker().wake_by_ref();   // 告诉执行器：请再 poll 我一次
            Poll::Pending
        }
    }
}
```

★ **关键认知**：返回 `Pending` 前**必须安排一次唤醒**，否则任务会永远沉睡。这里的 `wake_by_ref()` 是"自我唤醒"，等价于说"我现在没好，但请立刻再问我一次"。真实 I/O 的唤醒来自 epoll 事件，由运行时在数据就绪时调用 Waker。

# 2. Poll 与 Waker：一对搭档

## 2.1 Poll 的两种返回值

| 返回值 | 含义 | 执行器接下来做什么 |
|--------|------|-------------------|
| `Poll::Ready(T)` | 完成了，结果是 T | 结束该任务，不再 poll |
| `Poll::Pending` | 还没好 | 挂起任务，等 Waker 唤醒 |

`Pending` 不是"失败"，也不是"超时"，只是"暂停"。一个 Future 可以 Pending 百万次。

## 2.2 Waker 是什么

`Waker` 是一个**类型擦除的回调句柄**，内部包裹着"如何把任务重新入队"的逻辑：

```rust
pub struct Waker { /* 内部是 Arc<dyn Wake> */ }

impl Waker {
    pub fn wake(self);              // 消耗式唤醒
    pub fn wake_by_ref(&self);      // 借用式唤醒，不消耗
}
```

执行器在 poll 前构造一个 `Context`，把 Waker 塞进去。Future 在返回 `Pending` 前把这份 Waker **存起来**（或克隆后交给 I/O 注册），等条件满足时调用 `wake()`，执行器就把任务放回就绪队列。

```text
执行器                            Future
   │                                │
   │── poll(cx 含 waker) ──────────▶│
   │                                │ 未就绪，保存 waker
   │◀──────── Pending ──────────────│
   │ 挂起任务                        │
   │                                │ （I/O 就绪，某个线程）
   │                                │ waker.wake()
   │◀─────── 重新入队 ───────────────│
   │── poll(cx) ───────────────────▶│
   │◀──────── Ready(v) ─────────────│
```

★ **关键认知**：Waker 是**连接 Future 与执行器的唯一通道**。Future 不知道怎么调度自己，它只会"举手"（调用 Waker）；执行器不知道 I/O 何时就绪，它只会"等人举手"。这个解耦让任意 Future 能跑在任意执行器上。⚠ 忘了保存 Waker 或忘了调用 `wake()`，就是"任务永久挂起"的头号原因——程序不报错，只是卡住。

## 2.3 唤醒是可以合并的

执行器通常不保证"调用一次 wake 就 poll 一次"。多次唤醒可能被合并成一次 poll，这叫**唤醒合并（coalescing）**。因此 Future 必须能处理"被多 poll 一次"的情况，不能假设每次 poll 前都恰好有一次事件。

# 3. Pin：为什么需要它

## 3.1 自引用结构体的难题

`async fn` 展开后的状态机里，经常出现**自引用**：某个局部变量被后续状态借用。

```rust
async fn demo() {
    let s = String::from("hello");
    let r = &s;              // r 指向 s
    something().await;       // 挂起点：s 和 r 都要存进状态机
    println!("{}", r);       // 恢复后 r 仍要指向 s
}
```

如果这个状态机被移动（memcpy 到别处），`s` 的地址变了，但 `r` 还指着旧地址——悬垂指针。Rust 的借用检查器默认允许移动值，所以必须有一种机制**禁止移动**某些值：这就是 `Pin`。

```rust
pub struct Pin<P> { pointer: P }
```

`Pin<&mut T>` 是一个保证：**只要 T 没有实现 `Unpin`，它在内存中的地址就不会变**。你无法通过安全的 API 拿到 `&mut T` 去 `mem::swap` 它。

★ **关键认知**：`Pin` 解决的不是"固定到堆上"，而是"**承诺不再移动**"。`Box::pin(fut)` 把 Future 放在堆上并 Pin 住，此后即使 `Box` 本身移动，堆上的 Future 地址不变，自引用依然有效。⚠ 绝大多数你自己写的类型都自动实现 `Unpin`（不含自引用），此时 `Pin` 形同虚设；只有编译器生成的 `async` 状态机才真正需要 Pin 的保护。

## 3.2 Unpin 与 Pin 的关系

| 类型 | 是否 Unpin | 说明 |
|------|-----------|------|
| `u32`、`String`、`Vec<T>` | 是 | 移动无害 |
| 普通 `struct`（字段都 Unpin） | 是 | 自动推导 |
| `async fn` 生成的状态机 | 通常否 | 可能自引用 |
| `PhantomPinned` 标记的结构 | 否 | 显式禁止 Unpin |

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;

struct SelfRef {
    data: String,
    ptr: *const String,       // 指向自己的 data
    _pin: PhantomPinned,      // 取消自动 Unpin
}
```

★ **关键认知**：写给初学者的实用结论只有两条：**（1）用 `Box::pin` 把需要长期保存的 Future 钉在堆上；（2）调用 `poll` 时用 `Pin::new(&mut fut)` 的前提是 `fut: Unpin`，否则用 `unsafe { Pin::new_unchecked(...) }` 或干脆 `Box::pin`。** 日常写业务代码几乎不需要手写 `unsafe`，`async/await` 和 Tokio 已经替你处理了 Pin。

## 3.3 一个能跑的手写执行器

把前面几块拼起来，就能写一个玩具执行器：

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};

struct NoopWaker;   // 只打印唤醒，不真正调度

impl Wake for NoopWaker {
    fn wake(self: Arc<Self>) {
        println!("[waker] 任务被唤醒");
    }
}

fn block_on<F: Future>(fut: F) -> F::Output {
    let mut fut = Box::pin(fut);                       // Pin 住，允许自引用
    let waker = Waker::from(Arc::new(NoopWaker));
    let mut cx = Context::from_waker(&waker);

    loop {
        match fut.as_mut().poll(&mut cx) {
            Poll::Ready(v) => return v,                // 完成，退出
            Poll::Pending => std::thread::yield_now(), // 就绪队列为空时让出 CPU
        }
    }
}
```

这段代码用"忙轮询"代替真正的就绪队列，只适合演示。真实执行器用任务队列 + 事件循环，不会空转。

# 4. 执行器到底在做什么

## 4.1 一次完整的生命周期

```text
1. spawn(task)
     ├─ 把 Future 装箱（Pin<Box<dyn Future>>）
     └─ 包装成 Task，放入就绪队列

2. 事件循环（每轮）
     ├─ 从就绪队列取一个 Task
     ├─ 构造 Context（携带该 Task 的 Waker）
     ├─ 调用 poll
     ├─ Ready → 丢弃 Task，结果交给等待方
     └─ Pending → 任务挂起，等 Waker 唤醒

3. I/O 事件到达
     ├─ epoll 返回就绪的 fd
     ├─ 运行时的 reactor 找到对应 Waker
     └─ 调用 Waker，把 Task 重新推入就绪队列
```

★ **关键认知**：执行器（executor）与反应器（reactor）是两套角色。**executor 负责 poll 任务，reactor 负责监听 I/O 并唤醒任务**。Tokio 把两者集成在同一个运行时里，但概念上要分开——否则你会困惑"为什么 poll 返回 Pending 后，线程并没有去监听 socket"。是 reactor 在后台（或另一线程）替你监听。

## 4.2 为什么 Future 是零成本的

普通函数调用会分配栈帧；而 `async fn` 返回的 Future 是一个**大小在编译期已知的结构体**（状态 + 局部变量），可以内联、可以放在栈上（`async` 块外）。只有跨任务、跨 await 边界长期存活时才需要 `Box::pin` 上堆。

| 方案 | 内存分配 | 能否跨任务 |
|------|---------|-----------|
| 直接 `.await` | 通常零堆分配 | 不需要 |
| `tokio::spawn` | 堆分配一次 | 需要 `'static` |
| `Box::pin` | 堆分配一次 | 可手动传递 |

⚠ 这也是为什么"把每个小函数都 spawn 出去"是不好的：每个任务一次堆分配 + 一次调度，反而比直接 `.await` 更慢。

# 5. 常见错误

```text
错误 1：poll 返回 Pending 但没保存/触发 Waker
  现象：任务永远卡住，程序不报错
  原因：忘了 cx.waker().wake_by_ref()，或自定义 Future 漏了注册
  解决：每个 Pending 分支都要确保有唤醒路径

错误 2：对已完成的 Future 再次 poll
  现象：panic 或逻辑错乱
  原因：违反 Future 契约
  解决：用 FusedFuture（futures 库）记录完成状态

错误 3：手写 Future 时直接用 &mut self
  现象：编译报错，要求 self: Pin<&mut Self>
  解决：理解 Pin 契约，用 Pin::new（Unpin）或 Box::pin

错误 4：以为 Pending 是错误
  现象：把 Pending 当失败处理
  原因：语义误解
  解决：Pending 只是"还没好"，错误要靠 Output 里的 Result 表达
```

# 6. 练习

1. 实现一个 `YieldNow` Future：第一次 poll 返回 `Pending` 并自我唤醒，第二次返回 `Ready`。用第 3.3 节的 `block_on` 驱动，打印每次 poll。
2. 修改 `Countdown`，故意删掉 `wake_by_ref()`，观察玩具执行器为什么会一直空转（真实执行器则会永久挂起）。
3. 写一个包含 `String` 和指向它的引用的 `async` 块，用 `Box::pin` 包住，思考：如果去掉 Pin 直接 `mem::swap`，会破坏什么？

# 7. 小结

| 概念 | 要点 |
|------|------|
| Future | 状态机接口，`poll` 推进一次 |
| Poll | `Ready(值)` 或 `Pending`，Pending 不是错误 |
| Waker | Future 通知执行器"再来 poll 我"的回调 |
| Pin | 承诺值不再移动，保护自引用状态机 |
| Unpin | 可安全移动的类型，Pin 对其无约束 |
| executor | 负责 poll 任务 |
| reactor | 负责监听 I/O 并唤醒任务 |

下一篇进入 Tokio：`#[tokio::main]` 背后到底创建了什么、多线程运行时与 current-thread 运行时如何选择、`block_on` 在哪一层、以及"运行时边界"为什么会导致 panic。理解了本篇的 poll 循环，再看 Tokio 的调度器就顺理成章了。
