线程最大的不便在于"结果怎么拿回来"：`std::thread` 不返回任何值，异常也无法传回主线程，只能靠共享变量加锁。`std::future` 把"一次异步任务的结果"抽象成一个可等待的句柄，`std::async`、`std::promise`、`std::packaged_task` 则提供三种产生 `future` 的方式。它们统一解决了返回值、异常传递和完成通知三件事。

# 1. future 基础

## 1.1 最小可运行示例

```cpp
#include <future>
#include <iostream>

int compute(int x) {
    return x * x;
}

int main() {
    std::future<int> result = std::async(std::launch::async, compute, 7);
    std::cout << "主线程继续做别的事\n";
    int value = result.get();          // 阻塞，直到任务完成并取回结果
    std::cout << "结果: " << value << '\n';   // 49
    return 0;
}
```

`std::future<int>` 表示"未来会有一个 `int`"。调用 `get()` 时：

- 如果任务已经完成，立即返回结果；
- 如果还没完成，阻塞当前线程直到完成；
- 如果任务抛了异常，`get()` 会把异常**重新抛出**给调用方。

★ **关键认知**：`future` 是一次性的。`get()` 只能调用一次，调用后 `valid()` 变为 `false`，再调用是未定义行为。需要多处读取同一个结果时，用 `std::shared_future`。

## 1.2 future 的三个来源

| 来源 | 谁产生结果 | 典型场景 |
|------|-----------|---------|
| `std::async` | 标准库帮你起线程/延迟执行 | 简单的并行计算 |
| `std::promise<T>` | 你手动 `set_value`/`set_exception` | 跨线程回调、事件通知 |
| `std::packaged_task<R()>` | 调用这个任务时自动写入结果 | 任务队列、线程池 |

三者最终都指向同一个东西：**共享状态（shared state）**。`future` 是读取端，`promise`/`packaged_task`/`async` 是写入端。

```text
写入端                         共享状态                 读取端
promise::set_value      ──►  ┌──────────────┐  ◄── future::get
packaged_task::operator() ──►│ 值 或 异常    │
async 的线程函数        ──►  └──────────────┘
```

# 2. std::async

## 2.1 启动策略

```cpp
std::future<int> a = std::async(std::launch::async, task);     // 必须新线程
std::future<int> b = std::async(std::launch::deferred, task);  // 延迟到 get/wait
std::future<int> c = std::async(task);                         // 默认策略
```

| 策略 | 行为 |
|------|------|
| `std::launch::async` | 立即在新线程执行（或由实现保证异步执行） |
| `std::launch::deferred` | 不启动线程，直到 `get()`/`wait()` 才在调用线程同步执行 |
| 默认（两者按位或） | 由实现选择，可能异步，也可能延迟 |

★ **关键认知**：默认策略**不保证立即执行**。`std::async(task)` 可能把任务推迟到你第一次 `get()` 时才在当前线程跑，于是"异步"变成了"延迟同步"。需要确定并发时，显式写 `std::launch::async`。

```cpp
// 陷阱：以为后台在跑，其实还没开始
auto f = std::async([]{ std::this_thread::sleep_for(std::chrono::seconds(5)); });
// 这五行代码瞬间执行完，因为任务可能根本没启动
f.get();    // 到这里才真正睡 5 秒
```

## 2.2 异常传递

```cpp
// 头文件：future、iostream、stdexcept
int may_throw(int x) {
    if (x < 0) {
        throw std::invalid_argument("x 不能为负");
    }
    return x * 2;
}

int main() {
    auto f = std::async(std::launch::async, may_throw, -1);
    try {
        std::cout << f.get() << '\n';
    } catch (const std::invalid_argument& e) {
        std::cout << "捕获到: " << e.what() << '\n';   // 异常跨线程传回
    }
}
```

异常被存进共享状态，`get()` 时重新抛出。但要注意：

- 异常类型必须可拷贝（标准要求），所以别抛不可拷贝的怪类型。
- 如果 `future` 被销毁而从未 `get()`，异常会被静默丢弃。
- 如果 `future` 来自 `std::async(std::launch::async, ...)`，析构还会阻塞等待任务完成。

★ **关键认知**：`std::async` 是唯一能**自动把异常带回调用方**的启动方式。用裸 `std::thread` 时线程函数抛异常会直接 `std::terminate`，必须在函数内部 `try/catch` 兜底。

## 2.3 析构会阻塞

```cpp
{
    auto f = std::async(std::launch::async, [] {
        std::this_thread::sleep_for(std::chrono::seconds(10));
        return 1;
    });
    // 没有 get，也没有 wait
}   // f 析构：阻塞等待任务完成，这里会卡 10 秒
```

标准规定：由 `std::async(std::launch::async, ...)` 产生的 `future`，如果它是最后一个引用共享状态的 `future`，析构时**必须等待任务结束**。这相当于隐式的 `join`。

⚠ **关键认知**：这个阻塞行为常常被忽略。把 `async` 的 `future` 存进容器、在循环里反复创建又销毁，可能让程序串行化，完全没有并发效果。要么显式 `get()`/`wait()`，要么把 `future` 的生命周期控制好。

# 3. promise 与 packaged_task

## 3.1 promise：手动写入结果

```cpp
// 头文件：future、iostream、thread
int main() {
    std::promise<int> promise;
    std::future<int> result = promise.get_future();   // 先拿读取端
    std::thread producer([&promise] {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        promise.set_value(42);                         // 写入结果
    });
    std::cout << "等待结果...\n";
    std::cout << result.get() << '\n';                 // 阻塞直到 set_value
    producer.join();
}
```

`promise` 适合"结果不是由某个函数返回，而是在某个事件发生时产生"的场景，比如网络回调、定时器、信号处理转发。

写入异常：

```cpp
std::promise<int> promise;
std::future<int> result = promise.get_future();

std::thread worker([&promise] {
    try {
        throw std::runtime_error("下游失败");
    } catch (...) {
        promise.set_exception(std::current_exception());   // 捕获当前异常并存入
    }
});

try {
    result.get();
} catch (const std::runtime_error& e) {
    std::cout << e.what() << '\n';
}
worker.join();
```

| 操作 | 作用 |
|------|------|
| `set_value(v)` | 写入结果，最多一次 |
| `set_exception(e)` | 写入异常，最多一次 |
| `set_value_at_thread_exit(v)` | 线程退出时才写入 |
| `get_future()` | 获取对应的 `future`，只能调用一次 |

⚠ **关键认知**：`promise` 必须在销毁前完成 `set_value` 或 `set_exception`。如果 `promise` 析构时还没设置结果，共享状态里会被放入 `std::broken_promise` 异常，`get()` 时抛出。这常出现在"工作线程提前退出、忘了写结果"的路径上。

## 3.2 packaged_task：把函数包装成任务

```cpp
// 头文件：future、iostream、thread
int square(int x) {
    return x * x;
}
int main() {
    std::packaged_task<int(int)> task(square);   // 包装一个可调用对象
    std::future<int> result = task.get_future(); // 先取 future

    std::thread t(std::move(task), 9);           // task 不可拷贝，只能移动
    std::cout << result.get() << '\n';           // 81
    t.join();
}
```

`packaged_task` 的 `operator()` 会执行被包装的函数，并把返回值或异常写入共享状态。它对线程池特别有用：任务队列里存 `std::function<void()>`，调用时结果自动进 `future`。

```cpp
std::packaged_task<int()> task([]{ return 123; });
std::future<int> f = task.get_future();

std::queue<std::function<void()>> jobs;
jobs.push([task = std::move(task)]() mutable { task(); });   // 类型擦除

auto job = std::move(jobs.front());
jobs.pop();
job();                                                       // 执行任务
std::cout << f.get() << '\n';                                // 123
```

★ **关键认知**：`packaged_task::operator()` **不会把异常抛给调用者**，而是存进共享状态，由 `future::get()` 重新抛出。所以把它放进线程池时，工作线程不会因为任务异常而崩溃；异常会在调用方 `get()` 时出现。

⚠ **关键认知**：`packaged_task` 不可拷贝，只能移动。放进 `std::function` 时必须用 lambda 捕获它的 `std::move` 版本，并标 `mutable`（因为 `operator()` 是非 const 的）。这是写线程池时最常见的编译错误来源。

# 4. 超时与边界

## 4.1 wait_for 与 wait_until

```cpp
auto f = std::async(std::launch::async, [] {
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return 42;
});

if (f.wait_for(std::chrono::seconds(1)) == std::future_status::timeout) {
    std::cout << "一秒内没完成，先做别的事\n";   // 任务仍在后台跑
} else {
    std::cout << f.get() << '\n';
}
```

| 返回值 | 含义 |
|--------|------|
| `future_status::ready` | 已完成，可以 `get()` |
| `future_status::timeout` | 超时，任务仍在进行 |
| `future_status::deferred` | 任务是延迟执行的，还没开始 |

另一种写法是 `wait_until(deadline)`，传入一个时间点。

★ **关键认知**：`wait_for` 超时**不会取消任务**。C++ 标准没有提供取消 `async` 任务的接口；你只能选择"不再等"，任务仍会继续跑并占用资源。真正的取消需要任务自己检查停止标志（C++20 的 `std::stop_token` 是为此设计的协作式机制）。

```cpp
std::stop_source source;
std::stop_token token = source.get_token();

std::thread t([token] {
    while (!token.stop_requested()) {   // 协作式检查
        do_one_unit();
    }
});

source.request_stop();                  // 请求停止，但是"建议"不是"强杀"
t.join();
```

## 4.2 shared_future：多处等待

```cpp
std::shared_future<int> sf =
    std::async(std::launch::async, [] { return 7; }).share();
std::thread a([sf] { std::cout << sf.get() << '\n'; });   // 各自拷贝一份
std::thread b([sf] { std::cout << sf.get() << '\n'; });
a.join();
b.join();
```

`shared_future::get()` 可以调用多次，返回 `const T&`（对 `T` 为引用类型时有细节差异），因此适合"一个结果多个消费者"。

| 类型 | 可否多次 get | 是否可拷贝 |
|------|-------------|-----------|
| `future<T>` | 否 | 否（可移动） |
| `shared_future<T>` | 是 | 是 |

# 5. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 程序卡住十秒才退出 | `async` 的 future 析构阻塞 | 显式 `get()` 或缩短任务 |
| 任务没有并发执行 | 默认策略选了 deferred | 显式 `std::launch::async` |
| 第二次 `get()` 崩溃 | future 已被取空 | 用 `shared_future` |
| 异常无声消失 | future 未 `get()` 就销毁 | 保证每个 future 都被读取 |
| 线程池任务编译失败 | `packaged_task` 被拷贝 | `std::move` + lambda `mutable` |
| 超时后任务还在跑 | `wait_for` 不取消任务 | 用 `stop_token` 协作取消 |
| `broken_promise` | promise 未设置结果就销毁 | 所有路径都 `set_value`/`set_exception` |

⚠ **关键认知**：`future` 不是"可取消的线程句柄"，它只是"结果的提货单"。需要取消、需要进度、需要多次交互时，应该用条件变量、`stop_token` 或专门的任务框架，而不是硬套 `future`。

# 6. 动手练习

1. 用 `std::async(std::launch::async, ...)` 启动三个任务，分别计算 1 到 N 的和，在主线程 `get()` 后相加。再把启动策略改成默认值，打印每个任务开始执行时的线程 id，观察是否变成主线程执行。
2. 用 `std::promise<std::string>` 实现一个"两秒后返回状态"的模拟请求，主线程用 `wait_for(1s)` 判断超时，超时后仍继续等待并打印最终结果。
3. 用 `std::packaged_task<int(int)>` 包装一个可能抛异常的函数，把它放进 `std::queue<std::function<void()>>`，由另一个线程消费，主线程通过 `future` 捕获异常并打印 `what()`。

# 7. 小结

| 概念 | 要点 |
|------|------|
| `future` | 一次性结果句柄，`get()` 只能一次 |
| `async` | 默认策略可能延迟，需要并发就写 `launch::async` |
| 异常 | 由 `get()` 重新抛出，未取则丢弃 |
| `promise` | 手动写值/异常，适合回调式结果 |
| `packaged_task` | 把函数变成可入队的任务，异常进 future |
| `shared_future` | 可拷贝、可多次 `get()` |
| `wait_for` | 只停止等待，不取消任务 |
| 析构 | async 的 future 析构可能阻塞 |

下一篇进入线程池与任务队列，讲清楚为什么不能"每个任务起一个线程"、线程池的组成结构、任务队列如何做背压、优雅关闭的协议，以及工作线程里异常该怎么兜底。
