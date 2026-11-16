线程能解决 IO 等待，但每个线程都要占内存和调度开销，上万个并发连接时成本很高。`asyncio` 用**单线程 + 事件循环**做到"等待时不占线程"，把并发成本压到极低。代价是：一旦在协程里写了阻塞调用，整个循环都会被卡住。本篇把事件循环、协程、Task、超时取消和常见陷阱讲清楚。

# 1. 为什么需要异步

## 1.1 同步 IO 的瓶颈

同步代码里，一次网络请求会让整个线程停在那里等，5 个请求就要 5 秒。

线程方案能并发，但 5 万个线程会耗尽内存和调度能力。而绝大多数时间，这些线程只是**在等**。

★ **关键认知**：异步的核心不是"更快地计算"，而是**在等待时把控制权交出去**。一个请求等网络时，事件循环立刻去处理另一个请求；数据回来再切回来。所以 `asyncio` 的收益与"等待时间占比"成正比——纯 CPU 计算用异步没有意义。

## 1.2 三种并发模型

```text
同步：    请求1 ──等待──▶ 请求2 ──等待──▶ 请求3
线程：    请求1 ──▶
          请求2 ──▶  同时进行
协程：    请求1 ─┐
          请求2 ─┼─ 一个线程内交替推进
          请求3 ─┘
```

# 2. 事件循环

## 2.1 事件循环是什么

事件循环（event loop）是一个**调度器**：维护任务队列，反复取出就绪的协程执行，遇到 `await` 挂起就切换到下一个，等 IO 就绪再恢复。

```text
┌─────────────────────────────────────────┐
│              事件循环                    │
│   ┌────────┐   await 挂起   ┌────────┐  │
│   │ 协程 A │ ────────────▶ │ 就绪队列│  │
│   └────────┘               └────┬───┘  │
│        ▲                        │      │
│        └──── IO 完成，恢复 ◀─────┘      │
│   同一时刻只有一个协程在跑（单线程）      │
└─────────────────────────────────────────┘
```

★ **关键认知**：`asyncio` 是**单线程**的。它不靠多核并行，而靠"一个线程内高效切换"。所以协程里**任何阻塞调用都会卡死整个循环**——不是只卡住当前协程，是所有协程一起停摆。这是异步编程最重要的约束。

## 2.2 运行一个协程

```python
import asyncio

async def main():
    print("开始")
    await asyncio.sleep(1)      # 非阻塞等待，交还控制权
    print("结束")

asyncio.run(main())
```

`asyncio.run()` 负责创建事件循环、运行协程、清理。**一个程序通常只调用一次**，它是异步程序的入口。

⚠ **不能在已有事件循环里再调用 `asyncio.run()`**。Jupyter、某些 Web 框架里循环已经在跑，此时应直接 `await` 或 `create_task`，否则报 "asyncio.run() cannot be called from a running event loop"。

# 3. async/await

## 3.1 定义协程

```python
async def fetch(url: str) -> str:
    print(f"开始请求 {url}")
    await asyncio.sleep(1)      # 模拟 IO
    print(f"完成请求 {url}")
    return f"{url} 的内容"
```

用 `async def` 定义的函数叫**协程函数**，调用它**不会执行函数体**，而是返回一个协程对象：

⚠ **只调用不 await 会得到 `RuntimeWarning: coroutine was never awaited`**，函数体一行都不会跑。

## 3.2 await 的语义

`await` 后面跟一个**可等待对象**（协程、Task、Future）。它做两件事：

1. **执行**这个对象，直到完成或挂起；
2. 如果挂起，**把控制权交还事件循环**，让别的任务运行。

```python
async def main():
    result = await fetch("A")     # 等 A 完成
    print(result)
```

★ **关键认知**：`await` 不等于"并发"。连续 `await` 两个协程是**串行**的。要并发，必须先把协程包装成 `Task` 或交给 `gather`/`TaskGroup`，让它们同时被调度。

# 4. Task 与调度

## 4.1 create_task

`create_task` 把协程包装成 `Task`，**立即排入事件循环**，不阻塞当前协程：

```python
import asyncio

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"{name} 完成"

async def main():
    task1 = asyncio.create_task(fetch("A", 2))
    task2 = asyncio.create_task(fetch("B", 1))

    print("任务已创建，继续做别的事")
    print(await task1)        # 等 A
    print(await task2)        # 等 B（此时 B 早已完成）

asyncio.run(main())
```

★ **关键认知**：`create_task` 是**启动并发**的关键动作。调用后协程就开始被调度，而不是等到 `await` 才启动。但**创建了 Task 就必须持有引用并 await（或确保它完成）**，否则任务可能被垃圾回收，或在后台悄悄抛异常而无人知晓。

## 4.2 gather

```python
import asyncio
import time

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return name

async def main():
    start = time.perf_counter()
    results = await asyncio.gather(
        fetch("A", 2),
        fetch("B", 1),
        fetch("C", 1.5),
    )
    print(results)                                  # ['A', 'B', 'C']，保持顺序
    print(f"耗时 {time.perf_counter() - start:.1f}s")  # 约 2 秒

asyncio.run(main())
```

⚠ **`gather` 默认会把异常抛给调用方**。要"一个失败就取消其余"，用 3.11 的 `TaskGroup` 更合适。

## 4.3 TaskGroup（3.11+）

`TaskGroup` 是更现代的结构化并发：**块结束时保证所有子任务完成；任一任务失败则取消其余，并把异常汇总抛出**。

```python
import asyncio

async def fetch(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    if name == "B":
        raise RuntimeError("B 失败了")
    return name

async def main():
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(fetch("A", 1))
            tg.create_task(fetch("B", 0.5))
            tg.create_task(fetch("C", 2))
    except* RuntimeError as err:
        print(f"捕获异常组：{err.exceptions}")

asyncio.run(main())
```

★ **关键认知**：`TaskGroup` 相比 `gather` 的优势是**结构化**——子任务生命周期被限制在 `async with` 块内，块不结束就不会漏掉任务；异常以 `ExceptionGroup` 汇总，用 `except*` 分别处理。**新代码优先用 `TaskGroup`**。

# 5. 超时与取消

## 5.1 超时

```python
import asyncio

async def slow():
    await asyncio.sleep(10)
    return "完成"

async def main():
    try:
        result = await asyncio.wait_for(slow(), timeout=2)
    except TimeoutError:
        print("超时了")

asyncio.run(main())
```

`wait_for` 超时会**取消**被等待的协程并抛 `TimeoutError`（3.11 起是内置 `TimeoutError`）。

★ **关键认知**：`asyncio.timeout` 比 `wait_for` 更灵活，因为它能**包住一整段代码**，对其中所有 `await` 统一计时，而不是只针对单个协程。

## 5.2 取消

取消是协作式中断：`task.cancel()` 会在协程下一个 `await` 处抛 `CancelledError`。

```python
import asyncio

async def worker():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("收到取消，做清理")
        raise               # ★ 清理完必须重新抛出

async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(1)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("任务已取消")

asyncio.run(main())
```

⚠ **捕获 `CancelledError` 后一定要重新 `raise`**。它是控制信号，不是普通异常。吞掉会让任务"取消不掉"，事件循环无法正确清理，甚至导致 `TaskGroup` 卡住。

# 6. 常见陷阱

## 6.1 在协程里调用阻塞函数

```python
import asyncio
import time

async def bad():
    time.sleep(3)       # 错误！阻塞整个事件循环

async def good():
    await asyncio.sleep(3)   # 正确，非阻塞

async def main():
    # 三个协程本应并发，但 bad 会逐个卡住，总耗时 9 秒
    await asyncio.gather(bad(), bad(), bad())
```

★ **关键认知**：**异步代码里不能出现同步阻塞调用**。`time.sleep`、`requests.get`、大量 CPU 计算、同步文件 IO 都会卡住整个循环。替代方案：

| 阻塞操作 | 异步替代 |
|---------|---------|
| `time.sleep` | `await asyncio.sleep` |
| `requests` | `httpx.AsyncClient` / `aiohttp` |
| 同步文件读写 | `asyncio.to_thread` 或 `aiofiles` |
| CPU 密集 | `loop.run_in_executor` / 进程池 |

```python
import asyncio

async def main():
    # 把阻塞函数丢到线程池，避免卡住循环
    result = await asyncio.to_thread(sum, range(10_000_000))
    print(result)

asyncio.run(main())
```

## 6.2 忘记 await

这类错误往往只表现为一条 `RuntimeWarning`，然后"什么都没发生"。**看到协程没跑，第一反应就是检查是不是漏了 `await`。**

另外，`await` 只能出现在 `async def` 内部。同步函数要调用协程，必须通过 `asyncio.run()` 或 `asyncio.run_coroutine_threadsafe()`。

# 7. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| 协程里 `time.sleep` | 阻塞整个循环 | `await asyncio.sleep` |
| 忘记 `await` | 协程不执行 | 检查每个协程调用 |
| 连续 `await` 当并发 | 实际串行 | `gather` / `TaskGroup` |
| 吞掉 `CancelledError` | 取消失效 | 清理后 `raise` |
| 不持有 Task 引用 | 任务被回收 | 保存引用并 await |
| 在运行中的循环里 `asyncio.run` | 报错 | 直接 `await` |

# 8. 练习

1. 写三个协程分别 `await asyncio.sleep(1/2/3)`，先用连续 `await` 执行并计时，再用 `asyncio.gather` 执行并计时，解释耗时差异。
2. 用 `asyncio.TaskGroup` 并发执行 3 个任务，让其中一个抛异常，用 `except*` 捕获 `ExceptionGroup`，确认其余任务被取消。
3. 用 `asyncio.timeout` 给一段包含两次 `await` 的逻辑设置 2 秒上限；再用 `asyncio.to_thread` 包住一个 `time.sleep(3)` 的阻塞函数，验证事件循环不被卡住。

# 9. 小结

| 概念 | 要点 |
|------|------|
| 事件循环 | 单线程调度器，等待时切换 |
| `async def` | 定义协程函数，调用返回协程对象 |
| `await` | 执行并可能挂起，交还控制权 |
| `create_task` | 立即调度，实现并发 |
| `gather` | 批量等待，保持顺序 |
| `TaskGroup` | 结构化并发，异常汇总 |
| `timeout` | 3.11+，包住整段逻辑 |
| 取消 | 协作式，`CancelledError` 要重新抛 |
| 禁忌 | 协程里不能有阻塞调用 |

下一篇进入测试，讲清楚 `unittest` 与 `pytest` 的差异、fixture 的作用域、参数化、mock 的边界、覆盖率怎么看，以及测试目录如何组织。
