"Python 有 GIL，所以多线程是假的"——这句话流传很广，但只说对了一半。GIL 确实让多线程无法并行执行 Python 字节码，可它会在 IO 阻塞时释放，所以线程对 IO 密集任务依然有效。本篇把 GIL、threading、multiprocessing、锁与队列、线程/进程池串成一条完整的并发决策链。

# 1. GIL

## 1.1 GIL 是什么

GIL（全局解释器锁）是 CPython 的一把互斥锁：**同一时刻只允许一个线程执行 Python 字节码**。

```text
CPython 进程
┌─────────────────────────────────────┐
│  线程 A ──┐                          │
│  线程 B ──┼──▶ [ GIL ] ──▶ 解释器     │
│  线程 C ──┘        ▲                 │
│                    └─ 同一时刻只有一把 │
└─────────────────────────────────────┘
```

★ **关键认知**：GIL 保护的不是"你的数据"，而是 **CPython 解释器内部状态**（引用计数、内存分配）。它让解释器实现简单、单线程性能更好。**GIL 不是语言规范**，而是 CPython 的实现机制，Jython、IronPython 就没有。

## 1.2 GIL 对性能的影响

关键在于：**GIL 何时释放？**

| 任务类型 | 例子 | 多线程有效 | 原因 |
|---------|------|-----------|------|
| IO 密集 | 网络、文件、`sleep` | 有效 | 阻塞时释放 GIL |
| CPU 密集 | 纯计算、图像处理 | 无效 | 全程抢 GIL，实际串行 |
| 调 C 扩展 | NumPy、数据库驱动 | 常常有效 | 扩展内部主动释放 |

★ **关键认知**：判断"该用线程还是进程"，只看一个问题——**任务在等 IO，还是在烧 CPU？** 等待多用线程；计算多用多进程。这不是风格偏好，而是 GIL 决定的硬约束。

⚠ **不要用 GIL 当作"线程安全"的借口**。`x += 1` 在字节码层面是"读 → 加 → 写"三步，线程切换可能发生在任意一步之间，仍会产生竞态。

Python 3.13 引入了实验性的 free-threaded 构建（PEP 703），可关闭 GIL，但有性能代价和生态兼容问题，**不是默认构建，也不建议生产使用**。3.11/3.12 仍按有 GIL 设计并发。

# 2. threading

## 2.1 创建线程

```python
import threading
import time

def worker(name: str, seconds: float):
    print(f"{name} 开始")
    time.sleep(seconds)
    print(f"{name} 结束")

# 传函数和参数创建线程
t = threading.Thread(target=worker, args=("线程 A", 1.0))
t.start()
t.join()        # 等待线程结束
```

## 2.2 join 与守护线程

```python
threads = []
for i in range(3):
    t = threading.Thread(target=worker, args=(f"T{i}", 1.0))
    t.start()
    threads.append(t)

for t in threads:
    t.join()        # 主线程等所有子线程结束
print("全部完成")
```

⚠ **不 `join` 也不设 daemon 的线程会阻止程序退出**。主线程结束后 Python 会等待所有非守护线程；某个线程卡死，程序就永远退不出去。后台任务要么设 `daemon=True`，要么显式管理生命周期。

## 2.3 线程安全

```python
import threading

counter = 0
lock = threading.Lock()

def increment():
    global counter
    for _ in range(100_000):
        with lock:          # 保证"读-改-写"原子
            counter += 1

threads = [threading.Thread(target=increment) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)   # 400000，加锁后结果正确
```

★ **关键认知**：`with lock:` 保护的是**临界区**，不是变量本身。锁的范围要刚好覆盖"读-改-写"，太小无效，太大又退化成串行。能用不可变数据、局部变量或队列传递结果时，尽量不用共享状态——**最好的同步是不需要同步**。

# 3. 锁与队列

## 3.1 Lock 与 RLock

```python
lock = threading.Lock()

# 非阻塞获取，拿不到立刻返回 False，避免死等
if lock.acquire(blocking=False):
    try:
        ...
    finally:
        lock.release()
```

`RLock` 允许同一线程多次获取，适合加锁函数互相调用的场景。

⚠ **普通 `Lock` 在同一线程里重复 `acquire` 会死锁**。加锁的函数可能互相调用时用 `RLock`；否则优先 `Lock`，语义更简单。

## 3.2 queue.Queue

```python
import queue
import threading

q = queue.Queue()

def producer():
    for i in range(5):
        q.put(i)            # 满时阻塞
        print(f"生产 {i}")
    q.put(None)             # 哨兵：通知消费者结束

def consumer():
    while True:
        item = q.get()      # 空时阻塞
        if item is None:
            break
        print(f"消费 {item}")
        q.task_done()

t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)
t1.start(); t2.start()
t1.join(); t2.join()
```

```text
┌──────────┐   put    ┌─────────┐   get    ┌──────────┐
│ 生产者 1  │ ───────▶ │  Queue  │ ───────▶ │ 消费者 1  │
├──────────┤          │ (有界)  │          ├──────────┤
│ 生产者 2  │ ───────▶ │         │ ───────▶ │ 消费者 2  │
└──────────┘          └─────────┘          └──────────┘
```

★ **关键认知**：`queue.Queue` 的价值是**把"共享状态"变成"消息传递"**。生产者只管 `put`，消费者只管 `get`，两边不需要知道对方的锁。这比"共享 list 加锁"更安全。用 `Queue(maxsize=N)` 做背压，防止生产速度远超消费导致内存膨胀。

# 4. multiprocessing

## 4.1 为什么能绕开 GIL

多进程**每个进程有独立解释器和独立内存**，各自有一把 GIL，可以真正并行：

```text
进程 1 ── GIL ── CPU 核 1
进程 2 ── GIL ── CPU 核 2
进程 3 ── GIL ── CPU 核 3
```

```python
from multiprocessing import Process

def cpu_task(n):
    return sum(range(n))

if __name__ == "__main__":      # Windows 必须放在这个判断下
    p = Process(target=cpu_task, args=(10_000_000,))
    p.start()
    p.join()
```

⚠ **Windows 用 `spawn` 启动进程**，子进程会重新导入主模块。**所有进程创建代码必须放在 `if __name__ == "__main__":` 下**，否则会无限递归创建进程，直接卡死。

## 4.2 进程池与通信

```python
from multiprocessing import Pool, Process, Queue

def square(x):
    return x * x

def send(q):
    q.put("来自子进程")

if __name__ == "__main__":
    with Pool(processes=4) as pool:
        print(pool.map(square, range(10)))   # [0, 1, 4, 9, ...]
        print(pool.apply_async(square, (10,)).get(timeout=5))   # 100

    q = Queue()
    p = Process(target=send, args=(q,))
    p.start()
    print(q.get())
    p.join()
```

★ **关键认知**：多进程的代价是**启动慢、内存高、数据要序列化**。传给子进程的对象必须能被 `pickle`——lambda、局部函数、打开的文件句柄、数据库连接通常不行。这也是多进程不如多线程"随手可用"的原因。

# 5. concurrent.futures

## 5.1 统一接口

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def task(x):
    return x * x

# 线程池：适合 IO 密集
with ThreadPoolExecutor(max_workers=4) as executor:
    print(list(executor.map(task, range(5))))   # [0, 1, 4, 9, 16]

# 进程池：适合 CPU 密集
with ProcessPoolExecutor(max_workers=4) as executor:
    print(list(executor.map(task, range(5))))
```

★ **关键认知**：日常并发**优先用 `concurrent.futures`，而不是裸 `threading`/`multiprocessing`**。它自带线程/进程复用、异常传播、超时控制、结果收集，代码量少且不容易漏 `join`。切换线程/进程只需改类名。

## 5.2 submit 与 as_completed

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def fetch(url: str) -> str:
    time.sleep(1)               # 模拟网络请求
    return f"{url} 的内容"

urls = [f"https://example.com/{i}" for i in range(3)]

with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(fetch, url) for url in urls]

    for future in as_completed(futures):    # 谁先完成先处理谁
        print(future.result())
```

| 方法 | 行为 |
|------|------|
| `map(func, iterable)` | 按输入顺序返回，阻塞到全部完成 |
| `submit(func, *args)` | 立即返回 `Future`，不阻塞 |
| `as_completed(futures)` | 按完成顺序产出 |
| `future.result(timeout=...)` | 取结果，可设超时 |

⚠ **`executor.map` 按输入顺序返回**，即使后面任务先完成也要等前面。需要"完成即处理"用 `as_completed`；需要"保持顺序"才用 `map`。

⚠ **子线程里的异常不会自动打印**。不调用 `result()` 或 `exception()`，异常会被 `Future` 悄悄存着，看起来"什么都没发生"。用 `as_completed` + `result()` 确保异常被看到。

# 6. 如何选择

```text
任务是什么类型？
├─ IO 密集（网络、磁盘、数据库） → ThreadPoolExecutor
├─ CPU 密集（计算、编解码）      → ProcessPoolExecutor
├─ 海量并发连接                  → asyncio（见下一篇）
└─ 简单后台任务                  → threading.Thread(daemon=True)
```

# 7. 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| 用线程跑 CPU 密集 | 没加速甚至更慢 | 用进程池 |
| 以为 GIL 保证线程安全 | 竞态仍在 | 加锁或用队列 |
| 忘记 `join` | 程序不退出 | `join` 或 daemon |
| Windows 进程代码不在 `__main__` 下 | 无限创建进程 | 加 `if __name__ == "__main__":` |
| 子进程传 lambda | 无法 pickle | 用模块级函数 |
| 不取 `Future.result()` | 异常被吞 | 显式取结果 |

# 8. 练习

1. 写一个 IO 密集任务（每个 `time.sleep(0.5)`，共 8 个），分别用串行、`ThreadPoolExecutor`、`ProcessPoolExecutor` 执行，比较耗时并解释差异。
2. 用 `queue.Queue` 实现生产者-消费者：2 个生产者各生产 10 条消息，3 个消费者消费，用哨兵值通知结束，确保所有消息都被处理。
3. 用 `ThreadPoolExecutor` 并发模拟下载 5 个 URL，用 `as_completed` 按完成顺序打印结果；给任务加超时，并捕获其中一个任务故意抛出的异常。

# 9. 小结

| 概念 | 要点 |
|------|------|
| GIL | 同一时刻只有一个线程执行字节码 |
| 释放时机 | IO 阻塞时释放，CPU 计算时不释放 |
| 线程 | 适合 IO 密集，共享内存需加锁 |
| 队列 | 用消息传递替代共享状态 |
| 多进程 | 独立解释器，真正并行，启动开销大 |
| `__main__` | Windows 进程创建必须放这里 |
| `concurrent.futures` | 统一接口，日常首选 |
| `as_completed` | 按完成顺序处理，异常要取 `result()` |

下一篇进入异步，讲清楚事件循环如何用单线程处理海量并发、`async/await` 的执行模型、`Task` 的调度、超时与取消，以及异步代码最容易踩的"阻塞调用"陷阱。
