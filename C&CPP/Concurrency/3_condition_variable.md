"等队列里有任务再取"是并发程序最常见的需求。用轮询（隔一会儿查一次）能实现，但会白白烧 CPU，还拿不准延迟。条件变量（condition variable）就是为此而生：它让线程在条件不满足时真正睡眠，条件可能成立时由另一个线程唤醒。本篇讲清 `wait`/`notify` 的配合方式、谓词的作用、虚假唤醒与丢失唤醒，并用一个有界阻塞队列实现生产者消费者。

# 1. 为什么需要条件变量

## 1.1 轮询的代价

```cpp
while (queue.empty()) {
    // 反面教材：忙等，CPU 空转且无同步
}
```

这段代码有两个问题：一是空转占满一个核；二是 `queue.empty()` 本身就是对共享数据的无同步读取，属于数据竞争。加个 `sleep_for(10ms)` 能省 CPU，但又带来最多 10 毫秒的延迟，而且睡眠期间即使任务来了也看不见。

## 1.2 条件变量的模型

条件变量提供三个动作：

| 动作 | 作用 |
|------|------|
| `wait(lock, pred)` | 条件不满足时原子地解锁并睡眠，被唤醒后重新加锁并检查条件 |
| `notify_one()` | 唤醒一个正在等待的线程 |
| `notify_all()` | 唤醒全部等待线程 |

它的核心是"**睡眠和检查条件在同一个互斥量保护下完成**"，这样就不会出现"我刚检查完条件、还没睡下，别人就改了条件并通知"的窗口。

★ **关键认知**：条件变量本身**不存储状态**。它只是一个通知机制，"条件是否成立"必须由你自己用共享变量（布尔标志、队列长度等）表达，并始终用同一把互斥量保护。忘掉状态、只等通知，必然遇到丢失唤醒。

# 2. wait 与 notify

## 2.1 最小可运行示例

```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;                 // 被保护的条件状态

void worker() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });   // 条件不成立就睡，成立才继续
    std::cout << "工作线程开始执行\n";
}

int main() {
    std::thread t(worker);

    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;               // 修改条件必须持锁
    }                               // 先解锁
    cv.notify_one();                // 再通知，避免唤醒后立刻又阻塞在锁上

    t.join();
    return 0;
}
```

逐段说明：

- `std::unique_lock<std::mutex> lock(mtx);`：条件变量要求 `unique_lock`，因为 `wait` 需要在内部解锁和重新加锁。
- `cv.wait(lock, predicate)`：等价于 `while (!predicate()) cv.wait(lock);`。谓词为假时睡眠，为真时返回。
- 修改 `ready` 时持有 `mtx`，保证工作线程不会在"读 ready"和"进入等待"之间看到中间状态。
- `notify_one()` 放在解锁之后是优化：如果持锁通知，被唤醒的线程会立刻尝试加锁却拿不到，又得睡回去。

## 2.2 为什么 wait 必须搭配 mutex

假设没有互斥量，`wait` 分成两步："检查条件 → 睡眠"。两个线程可能这样交错：

```text
消费者：检查 queue 为空          ← 准备睡
生产者：放入任务，notify()        ← 此时消费者还没睡
消费者：进入睡眠                  ← 通知已经错过
结果：任务躺在队列里，消费者永远睡着
```

这就是**丢失唤醒（lost wakeup）**。互斥量把"检查条件"和"进入等待"绑成一个原子操作：消费者持有锁检查条件时，生产者无法修改条件；消费者真正睡下时，锁已经由 `wait` 原子释放，生产者才能进来修改并通知。

★ **关键认知**：`wait` 的"解锁 + 睡眠"是**原子**的（相对于同一把 mutex 上的通知），这是条件变量能避免丢失唤醒的根本原因。这也是它必须接收 `unique_lock`、而不能用 `lock_guard` 的原因。

## 2.3 谓词版本与裸 wait

```cpp
// 推荐：带谓词，内部自动处理虚假唤醒
cv.wait(lock, [] { return !queue.empty() || closed; });

// 等价的手写版本
while (!(!queue.empty() || closed)) {   // 用 while，不是 if
    cv.wait(lock);
}
```

`wait(lock)` 不带谓词时返回只代表"被唤醒过"，**不代表条件成立**。所以外面必须套 `while` 重新检查。带谓词的版本把这段循环封装好了，更不容易写错。

⚠ **关键认知**：永远用 `while` 或带谓词的 `wait`，不要用 `if`。被唤醒的原因可能是另一个线程改了别的状态、可能是虚假唤醒、也可能是 `notify_all` 唤醒了多个但只有一个能继续。`if` 只检查一次，条件不成立就会带着错误假设往下跑。

# 3. 虚假唤醒与丢失唤醒

## 3.1 虚假唤醒

**虚假唤醒（spurious wakeup）**指线程没有收到任何 `notify` 却从 `wait` 返回。标准允许这种情况发生，底层实现和操作系统调度都可能造成。

```cpp
// 错误：if 只检查一次
if (queue.empty()) {
    cv.wait(lock);
}
auto task = queue.front();    // 虚假唤醒时队列仍为空，越界！
```

修复就是循环检查：

```cpp
cv.wait(lock, [this] { return !queue.empty(); });
auto task = queue.front();
```

★ **关键认知**：虚假唤醒不是"某个平台的 bug"，而是标准明确允许的行为。任何 `wait` 都必须放在循环里重新验证条件，这是写条件变量的第一纪律。

## 3.2 丢失唤醒

丢失唤醒的根源是"修改条件时没持锁"或"通知发生在等待之前且没有状态记录"。两种典型错误：

```cpp
// 错误一：修改条件不持锁
void producer_bad() {
    queue.push(task);      // 数据竞争，且与消费者的检查无法排序
    cv.notify_one();
}

// 错误二：以为 notify 会"存下来"
void consumer_bad() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock);         // 如果 notify 在 wait 之前发生，这里永远等不到
}
```

第二种错误的本质是：条件变量没有记忆。`notify_one()` 在没有等待者时**什么也不做**，通知直接丢弃。正确做法是让状态先落地（`ready = true` 或 `queue.push`），等待方先检查状态，再决定是否睡。

```cpp
// 正确：状态 + 锁 + 谓词
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;          // 状态落地
}
cv.notify_one();           // 通知可丢，状态不会丢
```

★ **关键认知**：**通知可以丢，状态不能丢**。只要等待方在睡之前检查状态、在醒来后重新检查状态，即使通知丢失，它也能在下一轮检查中发现条件已成立。

## 3.3 notify_one 还是 notify_all

| 方法 | 唤醒数量 | 适用 |
|------|---------|------|
| `notify_one` | 一个 | 每次生产一个任务，只需一个消费者 |
| `notify_all` | 全部 | 状态切换（关闭、重置）、多个线程等不同条件 |

```cpp
// 关闭队列：所有消费者都必须醒来检查 closed
{
    std::lock_guard<std::mutex> lock(mtx);
    closed = true;
}
cv.notify_all();
```

⚠ **关键认知**：多个消费者等同一个队列时，用 `notify_one` 每次只唤醒一个，避免"惊群"（所有线程醒来抢锁，最后只有一个拿到任务）。但关闭、广播类事件必须 `notify_all`，否则睡着的线程永远不知道要退出。

# 4. 生产者消费者

## 4.1 有界阻塞队列实现

```cpp
// 头文件：condition_variable、cstddef、mutex、optional、queue、utility
template <typename T>
class BlockingQueue {
public:
    explicit BlockingQueue(std::size_t capacity) : capacity_(capacity) {}
    // 入队：队列满时阻塞，直到有空位或队列被关闭
    bool push(T value) {
        std::unique_lock<std::mutex> lock(mtx_);
        not_full_.wait(lock, [this] {
            return queue_.size() < capacity_ || closed_;
        });
        if (closed_) {
            return false;                    // 已关闭，拒绝新任务
        }
        queue_.push(std::move(value));
        not_empty_.notify_one();             // 唤醒一个消费者
        return true;
    }
    // 出队：队列空时阻塞；关闭且排空后返回 nullopt
    std::optional<T> pop() {
        std::unique_lock<std::mutex> lock(mtx_);
        not_empty_.wait(lock, [this] {
            return !queue_.empty() || closed_;
        });
        if (queue_.empty()) {
            return std::nullopt;             // 已关闭且没有剩余任务
        }
        T value = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();              // 唤醒一个生产者
        return value;
    }
    void close() {
        std::lock_guard<std::mutex> lock(mtx_);
        closed_ = true;
        not_empty_.notify_all();             // 让所有消费者醒来退出
        not_full_.notify_all();              // 让所有生产者醒来放弃入队
    }
private:
    std::mutex mtx_;
    std::condition_variable not_empty_;
    std::condition_variable not_full_;
    std::queue<T> queue_;
    std::size_t capacity_;
    bool closed_ = false;
};
```

## 4.2 逐段解释

- **两把条件变量**：`not_empty_` 给消费者等"有货"，`not_full_` 给生产者等"有空位"。如果只用一个，生产者可能被"队列非空"的通知唤醒，醒来发现还是满的，又睡回去。
- **谓词带 `closed_`**：关闭时条件直接成立，所有等待线程都能返回，不会永久卡在 `wait`；`push` 里的 `if (closed_) return false;` 则保证关闭后即使有空位也拒绝新任务。
- **`if (queue_.empty()) return nullopt;`**：关闭后如果队列已排空，消费者拿不到东西，返回空表示"没有更多任务了"。

生产使用：

```cpp
// 头文件：iostream、thread、vector
int main() {
    BlockingQueue<int> queue(4);         // 容量 4，超过就阻塞生产者
    std::thread producer([&queue] {
        for (int i = 0; i < 10; ++i) {
            if (!queue.push(i)) break;   // 关闭后停止生产
        }
        queue.close();                    // 生产完毕，通知消费者收尾
    });
    std::vector<std::thread> consumers;
    for (int i = 0; i < 2; ++i) {
        consumers.emplace_back([&queue, i] {
            while (auto item = queue.pop()) {
                std::cout << "消费者 " << i << " 拿到 " << *item << '\n';
            }
        });
    }
    producer.join();
    for (auto& t : consumers) t.join();
}
```

## 4.3 有界与无界的取舍

| 队列类型 | 优点 | 缺点 | 适用 |
|---------|------|------|------|
| 无界 | 生产者永不阻塞 | 内存可能爆掉 | 生产速度可控 |
| 有界 | 内存可控，天然背压 | 生产者可能被阻塞 | 生产快、消费慢 |

★ **关键认知**：有界队列的阻塞不是缺点，而是**背压（backpressure）**：当下游处理不过来时，让上游慢下来，比无限堆积任务最后 OOM 更健康。线程池的任务队列也应设上限，第 6 篇会展开。

⚠ **关键认知**：不要在持锁时执行耗时操作。上面的 `push` 只做内存操作；如果入队后还要写日志、发网络包，应该解锁后再做，否则所有生产者和消费者都会被拖慢。

# 5. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 线程永远不醒 | 通知发生在等待之前，且没有状态 | 先改状态再通知，等待用谓词 |
| 偶尔越界/崩溃 | 用 `if` 而非 `while`，虚假唤醒 | 带谓词的 `wait` |
| 关闭后线程卡死 | 只唤醒了一个等待者 | 关闭用 `notify_all` |
| 数据竞争 | 修改条件时没持同一把锁 | 条件读写都在锁内 |
| 性能差 | 持锁 `notify` 或持锁做 IO | 解锁后再通知，锁内只碰内存 |
| `wait` 编译不过 | 传了 `lock_guard` | 改用 `unique_lock` |

⚠ **关键认知**：条件变量析构时如果还有线程在 `wait`，行为未定义。关闭流程必须保证"先 `notify_all`、再 `join` 所有等待线程、最后销毁条件变量"。这也是为什么线程池的 `shutdown` 必须先通知再 `join`。

# 6. 动手练习

1. 用 `std::condition_variable` 实现一个"一次性事件"：主线程等待 `ready` 变真，工作线程准备 2 秒后置真并通知。故意把 `notify_one()` 放在 `ready = true` 之前，观察是否出现挂起，并解释原因。
2. 扩展本篇的 `BlockingQueue`，增加 `try_push`（队列满时立即返回 `false`，不阻塞）和 `size()`，思考 `size()` 返回的值为什么只能当参考。
3. 用两个消费者和一个生产者跑上面的示例，把 `notify_one()` 改成 `notify_all()`，观察输出和 CPU 占用变化；再在 `close()` 里改回 `notify_one()`，验证是否有消费者卡住。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 条件变量 | 只负责通知，不存储状态 |
| `wait` 三件事 | 解锁、睡眠、醒来后重新加锁 |
| 谓词 | 内部循环检查，天然防虚假唤醒 |
| 虚假唤醒 | 标准允许，必须用 `while` 重查 |
| 丢失唤醒 | 通知可丢，状态不能丢 |
| `notify_one` | 单任务唤醒，避免惊群 |
| `notify_all` | 关闭、广播状态变化 |
| 有界队列 | 阻塞生产者形成背压 |

下一篇进入原子操作与内存序，讲清楚 `std::atomic` 为什么能替代锁、CAS 循环怎么写、六种 `memory_order` 分别保证什么，以及无锁编程的真实边界在哪里。
