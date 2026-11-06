每来一个任务就 `std::thread` 起一个线程，在低负载时看不出问题，请求一多就会暴露：线程创建要几十微秒、每个线程占几 MB 栈、线程数超过核数后大量时间花在上下文切换上。线程池的思路是**预先建好固定数量的工作线程，让它们反复从队列里取任务执行**。本篇从结构设计讲到完整实现，重点解决任务队列、背压、优雅关闭和异常兜底四个工程问题。

# 1. 为什么需要线程池

## 1.1 每任务一线程的代价

每个请求创建一个 `std::thread` 再 `detach`，看似简单，代价却集中在下面几项：

| 问题 | 后果 |
|------|------|
| 创建销毁开销 | 每次几十微秒，高频短任务几乎全花在管理上 |
| 栈内存 | 每线程默认 1-8 MB，一千个线程就是几 GB |
| 数量失控 | 线程数远超核数，调度和缓存抖动严重 |
| 无法限流 | 突发流量直接把系统压垮 |
| 无法优雅退出 | `detach` 后无法等待，关闭时任务被硬砍 |

★ **关键认知**：线程是**昂贵的执行资源**，任务是**廉价的逻辑单元**。线程池把两者解耦：线程数量固定，任务数量可以很多，用队列吸收突发流量。这也是几乎所有服务端框架的默认结构。

## 1.2 线程池的组成

```text
             submit(task)
                  │
                  ▼
        ┌──────────────────┐
        │   有界任务队列    │  ← 满了就阻塞/拒绝（背压）
        └────────┬─────────┘
                 │ 取任务
     ┌───────────┼───────────┐
     ▼           ▼           ▼
 ┌───────┐   ┌───────┐   ┌───────┐
 │工作线程│   │工作线程│   │工作线程│
 └───┬───┘   └───┬───┘   └───┬───┘
     │           │           │
     └───────────┴───────────┘
                 │
                 ▼
        执行结果写入 future
```

核心组件只有四个：存放 `std::function<void()>` 的任务队列、固定数量的工作线程、负责睡眠与唤醒的互斥量加条件变量，以及一个停止接收新任务的关闭标志。

## 1.3 线程数量怎么定

| 任务类型 | 建议线程数 | 理由 |
|---------|-----------|------|
| CPU 密集 | `hardware_concurrency()` | 超过核数只会增加切换 |
| IO 密集 | 核数的 2-4 倍或更多 | 线程大部分时间在等待 |
| 混合 | 分两个池，分别调参 | 避免慢 IO 堵住快计算 |
| 不确定 | 从核数开始，压测调整 | 没有万能公式 |

⚠ **关键认知**：线程池不是越大越好。CPU 密集任务线程数超过物理核数后，吞吐不会上升，延迟反而抖动；IO 密集任务可以超配，但要有上限，否则和"每任务一线程"没有区别。

# 2. 任务队列

## 2.1 有界与无界

无界队列的 `push` 永不阻塞，生产快于消费时任务无限堆积，最后耗尽内存。有界队列在满时让提交方等待或失败，把压力反馈给上游，这就是**背压（backpressure）**。

| 队列 | 生产者行为 | 风险 |
|------|-----------|------|
| 无界 | 永不阻塞 | 内存无限增长，OOM |
| 有界阻塞 | 满时阻塞 | 提交线程被拖慢，需防死锁 |
| 有界拒绝 | 满时立即失败 | 调用方必须处理失败 |

★ **关键认知**：队列长度是**系统能容忍的积压量**，不是随手填的数字。它约等于"峰值 QPS × 可接受延迟"。队列太长，请求排队到超时；太短，突发流量直接触发拒绝。这个值要跟着 SLA 调。

## 2.2 背压策略

| 策略 | 做法 | 适用 |
|------|------|------|
| 阻塞提交 | 队列满时 `wait` | 内部流水线，能接受反压 |
| 立即拒绝 | 返回错误/抛异常 | 对外服务，快速失败 |
| 丢弃最旧 | 弹出队首再入队 | 实时性优先、旧数据无价值 |
| 调用方执行 | 提交线程自己跑这个任务 | 避免死锁，降低突发 |
| 扩容队列 | 动态增大上限 | 不推荐，只是推迟问题 |

⚠ **关键认知**：**在持锁状态下阻塞等待队列空位，可能死锁**。如果工作线程执行的任务又向同一个池提交子任务并等待结果，而队列已满，工作线程会卡在提交上，永远无法腾出空位。线程池要么禁止嵌套提交，要么提供"调用方执行"的兜底策略。

# 3. 最小线程池实现

## 3.1 完整代码

```cpp
#include <condition_variable>
#include <functional>
#include <future>
#include <iostream>
#include <mutex>
#include <queue>
#include <stdexcept>
#include <thread>
#include <type_traits>
#include <utility>
#include <vector>

class ThreadPool {
public:
    explicit ThreadPool(std::size_t workers,
                        std::size_t max_queue = 1024)
        : max_queue_(max_queue) {
        for (std::size_t i = 0; i < workers; ++i) {
            workers_.emplace_back([this] { worker_loop(); });
        }
    }
    ~ThreadPool() {
        shutdown();
    }
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;
    template <typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>> {
        using Result = std::invoke_result_t<F, Args...>;
        auto task = std::make_shared<std::packaged_task<Result()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...));
        std::future<Result> future = task->get_future();
        {
            std::unique_lock<std::mutex> lock(mtx_);
            not_full_.wait(lock, [this] {
                return tasks_.size() < max_queue_ || closed_;
            });
            if (closed_) {
                throw std::runtime_error("线程池已关闭");
            }
            tasks_.emplace([task] { (*task)(); });   // 类型擦除为 void()
            not_empty_.notify_one();
        }
        return future;
    }
    void shutdown() {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            if (closed_) {
                return;
            }
            closed_ = true;          // 停止接收新任务
        }
        not_empty_.notify_all();      // 唤醒空闲工作线程
        not_full_.notify_all();       // 唤醒卡在提交上的生产者
        for (auto& worker : workers_) {
            if (worker.joinable()) {
                worker.join();        // 等待队列排空、线程退出
            }
        }
        workers_.clear();
    }
private:
    void worker_loop() {
        for (;;) {
            std::function<void()> job;
            {
                std::unique_lock<std::mutex> lock(mtx_);
                not_empty_.wait(lock, [this] {
                    return closed_ || !tasks_.empty();
                });
                if (closed_ && tasks_.empty()) {
                    return;           // 关闭且已排空，线程退出
                }
                job = std::move(tasks_.front());
                tasks_.pop();
                not_full_.notify_one();   // 腾出空位，唤醒生产者
            }
            try {
                job();                // 任务异常已被 packaged_task 存入 future
            } catch (const std::exception& e) {
                std::cerr << "工作线程捕获异常: " << e.what() << '\n';
            } catch (...) {
                std::cerr << "工作线程捕获未知异常\n";
            }
        }
    }
    std::mutex mtx_;
    std::condition_variable not_empty_;
    std::condition_variable not_full_;
    std::queue<std::function<void()>> tasks_;
    std::vector<std::thread> workers_;
    std::size_t max_queue_;
    bool closed_ = false;
};
```

## 3.2 逐段解释

- **构造**：先启动 `workers` 个工作线程，它们立刻进入 `worker_loop`，发现队列为空就睡在 `not_empty_` 上。
- **`submit`**：把可调用对象和参数用 `std::bind` 绑定，塞进 `packaged_task`，这样结果和异常都有地方存，返回的 `future` 给调用方。
- **`tasks_.emplace(...)`**：队列元素类型是 `std::function<void()>`，而 `packaged_task` 是具体类型，用 lambda 包一层完成类型擦除。
- **`not_full_.wait`**：队列满时提交方在这里等待，形成背压；`closed_` 进谓词，保证关闭时能醒来。
- **`worker_loop` 的退出条件**：`closed_ && tasks_.empty()`。注意是"关闭**且**排空"，关闭后队列里已有的任务仍会执行完。
- **`job()` 外面包 `try/catch`**：`packaged_task` 已经把任务自身的异常存进 `future`，这里的 catch 是最后一道保险，防止 `std::function` 包装层或内存分配抛出异常导致线程崩溃。

## 3.3 使用示例

```cpp
// 头文件：future、iostream、vector
int main() {
    ThreadPool pool(4, 64);      // 4 个工作线程，队列上限 64
    std::vector<std::future<int>> results;
    for (int i = 0; i < 10; ++i) {
        results.push_back(pool.submit([i] { return i * i; }));
    }
    int sum = 0;
    for (auto& f : results) {
        sum += f.get();          // 逐个取结果；任务可能已并行完成
    }
    std::cout << "平方和: " << sum << '\n';   // 285
    // pool 析构时自动 shutdown，排空队列并 join
}
```

★ **关键认知**：`submit` 返回 `future` 后，调用方可以选择"立刻 `get`"（退化成同步）、"稍后 `get`"（真正的流水线）或"不 `get`"（结果被丢弃）。**但要小心不 `get`**：如果任务抛异常，异常留在共享状态里，没人观察就被静默丢弃。

# 4. 优雅关闭

## 4.1 关闭顺序

```text
shutdown() 调用
   │
   ├─ 1. 持锁置 closed_ = true        ← 新 submit 立刻失败
   ├─ 2. notify_all 唤醒所有线程       ← 空闲线程醒来检查退出
   ├─ 3. 工作线程继续排空队列           ← 已接收的任务不丢
   ├─ 4. 队列空且 closed_ → 线程返回
   └─ 5. join 所有线程                 ← 确认资源回收
```

顺序不能乱：

- 先置标志再通知，否则线程醒来看到 `closed_ == false` 又睡回去。
- 先通知再 `join`，否则工作线程还在睡，`join` 永远不返回。
- 关闭后 `submit` 必须失败，不能再往队列里塞任务。

## 4.2 两种关闭语义

| 语义 | 行为 | 适用 |
|------|------|------|
| 排空（drain） | 执行完队列中已有任务再退出 | 数据不能丢，如写盘、发消息 |
| 立即取消 | 清空队列，尽快退出 | 快速重启、任务可丢弃 |

立即取消只需在 `shutdown` 里把 `closed_` 置真、用一个空队列 `swap` 掉 `tasks_`，再执行同样的 `notify_all` 和 `join`；工作线程看到 `closed_` 就直接返回，不再排空。

⚠ **关键认知**：**正在执行的任务无法被强制取消**。C++ 没有安全的线程强杀机制；`pthread_cancel` 会导致锁不释放、对象半销毁。所谓"取消"只能靠任务自己检查停止标志（`std::stop_token` 或 `std::atomic<bool>`），在安全点主动返回。

# 5. 异常处理

线程池里的异常有三层：

| 层级 | 异常去向 | 处理方式 |
|------|---------|---------|
| 任务函数抛出 | 被 `packaged_task` 存入 `future` | 调用方 `get()` 时重新抛出 |
| `future` 无人读取 | 共享状态销毁时丢弃 | 关键任务必须有人 `get()` |
| 工作线程框架抛出 | 未捕获则 `std::terminate` | `worker_loop` 外层 `try/catch` 兜底 |

```cpp
auto f = pool.submit([] {
    throw std::runtime_error("任务失败");
});
try {
    f.get();
} catch (const std::runtime_error& e) {
    std::cerr << "调用方处理: " << e.what() << '\n';
}
```

★ **关键认知**：`packaged_task::operator()` **不会把任务异常抛给工作线程**，而是存进共享状态。所以工作线程不会因为业务异常退出，异常会在调用方 `get()` 时出现。这既是优点（隔离故障）也是坑（没人 `get` 就静默丢失）。

# 6. 调参与常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 程序退出时卡住 | 工作线程还在等 `not_empty_` | 关闭时先 `notify_all` 再 `join` |
| 线程池析构后任务还在跑 | 用了 `detach` 或未 `join` | 析构里 `shutdown`，禁止 `detach` |
| 提交任务后一直阻塞 | 队列满且消费者被阻塞 | 检查是否嵌套提交、增大队列或调用方执行 |
| 任务异常没人知道 | `future` 未 `get` | 关键路径必须读取结果 |
| 吞吐上不去 | 线程数过多或临界区过大 | 压测调线程数，缩小锁范围 |
| 关闭后仍能提交 | `closed_` 检查缺失 | 提交前后各检查一次 |
| 数据竞争 | 任务捕获了失效引用 | 按值捕获，或保证对象生命周期 |

⚠ **关键认知**：**不要在工作线程里同步等待自己提交的子任务**。池只有 N 个线程，如果 N 个线程都在等子任务，而子任务还在队列里排队，就形成"线程池饥饿"死锁。需要嵌套并行时，用独立的池，或者让等待方参与执行任务（caller-runs）。

# 7. 动手练习

1. 编译并运行本篇的线程池，提交 100 个睡眠 10 毫秒的任务，分别用 1、4、8 个工作线程，记录总耗时，解释为什么 8 个线程不一定比 4 个快。
2. 把 `max_queue` 改成 2，提交 20 个任务，观察主线程是否被阻塞；再实现一个"队列满时抛异常"的 `try_submit` 版本，比较两种背压策略。
3. 给线程池增加 `shutdown_now()`，提交一批长任务后立即调用，用计数器统计实际执行了多少个，验证"取消只对未开始的任务有效"。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 线程池价值 | 复用线程、限制数量、队列吸收突发 |
| 队列 | 有界才有背压，长度按延迟目标定 |
| 任务存储 | `packaged_task` + lambda 类型擦除成 `void()` |
| 返回值 | `submit` 返回 `future`，异常随之传递 |
| 关闭 | 置标志 → 通知 → 排空 → join |
| 取消 | 协作式，无法强杀运行中的任务 |
| 异常 | 任务异常进 future，框架异常要兜底 |
| 饥饿 | 禁止工作线程同步等待自己池里的子任务 |

下一篇进入并发 bug 与调试，讲清楚数据竞争、死锁、ABA、迭代器失效这四类问题分别长什么样，如何用 ThreadSanitizer 和 gdb 定位，以及一份可执行的排查清单。
