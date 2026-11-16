锁能保证正确，但每次加解锁都有开销，而且有些场景（计数器、标志位、无锁队列）用一把锁显得太重。`std::atomic` 提供了另一条路：把单个变量的读写变成不可分割的操作，不加锁也能避免数据竞争。但原子类型只保证"这一步不撕裂"，不自动保证"多步之间的顺序"，于是就有了 `memory_order`。本篇从 CAS 讲到六种内存序，最后划清无锁编程的能力边界。

# 1. 数据竞争与原子类型

## 1.1 为什么 ++ 不安全

```cpp
long long counter = 0;
// 两个线程各执行 100 万次 ++counter
// 结果通常小于 200 万
```

`++counter` 是"读 → 加 → 写"三步，不是一条不可分割的指令。两个线程可能读到同一个旧值，各自加一后写回，丢失一次更新。

上一篇文章用互斥量解决了它。原子类型给的是另一种解法：

```cpp
#include <atomic>

std::atomic<long long> counter{0};

void add() {
    for (int i = 0; i < 1000000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);   // 原子自增
    }
}
```

## 1.2 atomic 的基本操作

```cpp
std::atomic<int> x{0};

x.store(42);                    // 原子写
int v = x.load();               // 原子读
x.fetch_add(1);                 // 原子加，返回旧值
x.fetch_sub(1);                 // 原子减
x.exchange(10);                 // 原子交换，返回旧值
int old = x.load();
x.compare_exchange_strong(old, 20);   // CAS
```

自增自减还可以直接写 `++x`、`x++`、`--x`、`x--`，以及 `x += n`、`x -= n`、`x &= n` 等复合赋值，它们都是原子的。

```cpp
std::atomic<bool> stop{false};
if (stop.load()) { /* 读标志 */ }
stop.store(true);                // 置标志
```

★ **关键认知**：`std::atomic<T>` 的原子性只覆盖**单个对象的一次操作**。`x.fetch_add(1)` 不会被撕裂，但"先 `load` 判断、再 `store` 修改"这种**复合逻辑**仍然可能被别的线程插队，需要用 CAS 循环或锁保护。

## 1.3 哪些类型可以原子

- 所有整型、指针、`bool`：标准保证无锁（通常硬件直接支持）。
- 其他可平凡拷贝的类型：可以实例化，但可能内部用锁实现。
- `std::atomic<double>`：C++20 前没有 `fetch_add`，只能 `load`/`store`/CAS。

用 `is_lock_free()` 判断是否真的无锁：

```cpp
struct Big { char data[64]; };
std::atomic<long long> a{0};
a.is_lock_free();                 // 通常为 true
std::atomic<Big> b;               // 可以实例化，但可能内部加锁
b.is_lock_free();                 // 很可能是 false
```

⚠ **关键认知**：`std::atomic<T>` 不是"任何类型都能无锁"的魔法。大于机器字长的类型往往退化成内部锁，性能不一定比直接用 `std::mutex` 好。需要原子操作大对象时，通常该重新设计数据结构。

# 2. CAS：比较并交换

## 2.1 compare_exchange 的语义

CAS 做一件事：**如果当前值等于期望值，就写入新值并返回 `true`；否则把当前值写回 `expected` 并返回 `false`**。

```cpp
std::atomic<int> x{10};
int expected = 10;
bool ok = x.compare_exchange_strong(expected, 20);
// 如果 x 此时是 10：ok == true，x 变成 20
// 如果 x 已被别人改成 30：ok == false，expected 被更新为 30
```

第二个行为非常关键：**失败时 `expected` 会被改成当前值**，所以循环重试时不用手动重新 `load`。

```cpp
std::atomic<int> counter{0};

void increment() {
    int old = counter.load(std::memory_order_relaxed);
    while (!counter.compare_exchange_weak(old, old + 1,
                                          std::memory_order_relaxed)) {
        // 失败时 old 已是别人写入的新值，直接用新值再试
    }
}
```

`weak` 与 `strong` 的区别：

| 版本 | 失败原因 | 适用 |
|------|---------|------|
| `compare_exchange_weak` | 值不等，或伪失败（spurious failure） | 循环里，通常更便宜 |
| `compare_exchange_strong` | 仅值不等 | 只试一次，或不想处理伪失败 |

★ **关键认知**：在 `while` 循环里优先用 `weak`。它允许在值其实相等时也返回失败（某些架构上 CAS 指令本身如此），但循环会自然重试；`strong` 会在内部帮你重试，单次调用更贵。循环外只调用一次时用 `strong`。

## 2.2 用 CAS 更新最大值

```cpp
void update_max(std::atomic<int>& target, int candidate) {
    int old = target.load(std::memory_order_relaxed);
    while (old < candidate &&
           !target.compare_exchange_weak(old, candidate,
                                         std::memory_order_relaxed)) {
        // old 已被更新为当前值；若它已 >= candidate，循环条件自然退出
    }
}
```

这段代码是标准的 CAS 循环模板：**读取 → 本地计算 → CAS 尝试 → 失败则用最新值重算**。所有"读-改-写"的原子操作都可以套这个结构。

## 2.3 无锁栈的入栈

```cpp
struct Node { int value; Node* next; };
std::atomic<Node*> head{nullptr};

void push(int value) {
    Node* node = new Node{value, nullptr};
    node->next = head.load(std::memory_order_relaxed);
    while (!head.compare_exchange_weak(node->next, node,
                                       std::memory_order_release,
                                       std::memory_order_relaxed)) {
        // node->next 被更新为当前栈顶，继续尝试
    }
}
```

入栈是安全的：只有成功的那次 CAS 会让新节点可见，失败方不会被任何线程看到。但**出栈要难得多**——"读栈顶 → 取 next → CAS 改 head"期间，别的线程可能已经弹出并释放了节点，造成悬垂指针和 ABA 问题。第 7 篇会专门讲 ABA。

⚠ **关键认知**：单变量 CAS 能实现无锁入栈，但**无锁出栈需要解决内存回收问题**（hazard pointer、引用计数、epoch/RCU 等），不是加个 CAS 就完事。不要在生产代码里随手写无锁容器。

# 3. memory_order

## 3.1 为什么需要内存序

CPU 和编译器都会重排指令。对单线程来说，只要结果一致，重排是透明的；但多线程下，一个线程的写可能以出人意料的顺序被另一个线程看到。

```cpp
int data = 0;
std::atomic<bool> ready{false};

void producer() {
    data = 42;                                  // ① 普通写
    ready.store(true, std::memory_order_relaxed); // ② 标志写
}

void consumer() {
    while (!ready.load(std::memory_order_relaxed)) {} // ③ 等标志
    assert(data == 42);                          // ④ 可能失败！
}
```

即使 ② 是原子写、③ 是原子读，`data = 42` 与 `ready = true` 之间也可能被重排。消费者看到 `ready == true` 时，`data` 可能还没写进去。要建立顺序，必须用 `release`/`acquire`。

## 3.2 六种内存序

| 内存序 | 保证 | 典型用途 |
|--------|------|---------|
| `relaxed` | 只保证原子性，不保证任何顺序 | 独立计数器、统计 |
| `consume` | 数据依赖顺序（实践中很少用，多数编译器按 acquire 处理） | 几乎不用 |
| `acquire` | 本线程之后的读写不能重排到该读之前 | 读标志/取指针 |
| `release` | 本线程之前的读写不能重排到该写之后 | 发布数据/放指针 |
| `acq_rel` | 同时具备 acquire 和 release | RMW 操作用于双向同步 |
| `seq_cst` | 全局单一总顺序，最严格 | 默认值，拿不准就用它 |

★ **关键认知**：`memory_order` 描述的是**可见性和重排约束**，不是"指令执行快慢"。`relaxed` 不是"更快执行"，而是"允许 CPU 和编译器自由重排，只保留原子性"；`seq_cst` 也不是"更慢地执行同一条指令"，而是要求一条全局一致的顺序。

## 3.3 release/acquire 配对

```cpp
int data = 0;
std::atomic<bool> ready{false};

void producer() {
    data = 42;                                     // ① 先写数据
    ready.store(true, std::memory_order_release);  // ② 发布：① 不能跑到 ② 后面
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {} // ③ 获取：④ 不能跑到 ③ 前面
    assert(data == 42);                            // ④ 必然成立
}
```

配对规则：

- 生产者用 `release` 写 `ready`，消费者用 `acquire` 读 `ready`。
- 当消费者读到生产者写入的值时，**生产者 release 之前的所有写，对消费者 acquire 之后的所有读都可见**。

这条关系叫 **happens-before**：① happens-before ②，② synchronizes-with ③，③ happens-before ④，所以 ④ 一定能看到 ①。

⚠ **关键认知**：`release` 和 `acquire` 必须作用在**同一个原子变量**上，并且是"写-读"配对才有 synchronizes-with。在 `x` 上 release、在 `y` 上 acquire 不建立任何跨变量顺序。

## 3.4 relaxed 什么时候够用

如果多个原子操作之间没有"用一个值去保护另一个值"的关系，`relaxed` 就够了：

```cpp
std::atomic<long long> requests{0};
std::atomic<long long> errors{0};

void handle() {
    requests.fetch_add(1, std::memory_order_relaxed);   // 只统计次数
    if (failed) {
        errors.fetch_add(1, std::memory_order_relaxed);
    }
}
```
这两个计数器互不依赖，只需要各自不丢更新，不需要规定谁先谁后；但下面这种就不行：

```cpp
std::atomic<int> value{0};
std::atomic<bool> published{false};

// 错误：两个 relaxed 之间没有顺序
value.store(42, std::memory_order_relaxed);
published.store(true, std::memory_order_relaxed);

// 消费者可能先看到 published == true，再读到 value == 0
```

★ **关键认知**：**默认用 `seq_cst`**。它最严格、最不容易写出微妙 bug，在 x86 上多数读操作的成本与 acquire 相同。只有 profile 证明内存序是瓶颈，并且你能写清楚正确性论证时，才降级到 `acquire`/`release`/`relaxed`。

## 3.5 x86 的实际表现

x86 是较强的内存模型（TSO），只有 Store-Load 允许重排，所以 `seq_cst` 的 store 常编译成带 `xchg` 或 `mfence` 的指令，比 `relaxed` 多一层开销。这段平台知识只用于理解性能，**写代码仍以标准语义为准**。

⚠ **关键认知**：在 x86 上能跑对的代码，搬到 ARM 上可能立刻出错。ARM 是弱内存模型，store-store、load-load 都可能重排。所有跨线程发布数据的路径都必须写清 `release`/`acquire`。

# 4. 无锁的边界

"无锁"听起来很美好，但它有明确的适用条件：

| 条件 | 说明 |
|------|------|
| 状态能用单个原子变量表达 | 多变量不变式无法靠单次 CAS 维护 |
| 不需要阻塞等待 | 否则应叫"非阻塞"而非"无锁" |
| 内存回收可控 | 出栈、删除节点涉及悬垂指针 |
| 有 ABA 防护 | 地址复用会让 CAS 误判 |

真正的无锁算法要求**系统整体持续前进**：某个线程被挂起，其他线程仍能完成操作。很多所谓"无锁队列"只是"无阻塞"（lock-free 或 obstruction-free），实现难度和验证成本都极高。

★ **关键认知**：工程上 **99% 的场景不需要无锁数据结构**。用互斥量 + 条件变量写清楚，性能往往已经够用；真正需要无锁时，优先用成熟库（如 `boost::lockfree`、folly），而不是自己发明。

⚠ **关键认知**：用 `std::atomic` 保护一个计数器，不等于整个程序"无锁"。只要还有一处用 `mutex`，程序就不是无锁的。判断标准是"是否存在阻塞路径"，不是"用了多少个 atomic"。

# 5. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 原子变量还是算错 | 用 `load` + `store` 做复合更新 | CAS 循环或锁 |
| 消费者读到半成品 | relaxed 未建立发布顺序 | release/acquire 配对 |
| CAS 死循环 | 忘记失败时 `expected` 已更新 | 直接复用 `expected` 重试 |
| 把 `volatile` 当原子 | `volatile` 不保证原子性/顺序 | 用 `std::atomic` |
| ABA 导致误删 | 地址被复用，CAS 误判 | 带标签指针、hazard pointer |
| 无锁队列性能更差 | 伪共享、缓存行争用 | 填充缓存行、分片计数 |

```cpp
if (count.load() == 0) {     // 错误：检查和修改之间会被插队
    count.store(1);
}
int expected = 0;            // 正确：用 CAS 合成一步
count.compare_exchange_strong(expected, 1);
```

⚠ **关键认知**：`volatile` 在 C++ 里只用于内存映射 IO 和信号处理，**不提供线程间同步**。它不会被优化掉，但 `volatile int` 的 `++` 依然不是原子的，也不能建立 happens-before。并发场景一律用 `std::atomic`。

# 6. 动手练习

1. 用 `std::atomic<long long>` 和 `memory_order_relaxed` 重写开头的计数器，跑 4 线程各 100 万次，确认结果为 400 万；再用普通 `long long` 跑一次，观察结果偏差。
2. 实现 `update_max` 的 CAS 版本，用 8 个线程各自随机写入 1 到 1000，最后验证结果等于所有写入值的最大值。
3. 把 release/acquire 示例中的 `memory_order_release` 和 `memory_order_acquire` 都改成 `relaxed`，在 ARM 机器（或开启优化的 x86 多跑几次）上观察断言是否可能失败，并解释为什么这类 bug 难以复现。

# 7. 小结

| 概念 | 要点 |
|------|------|
| `std::atomic` | 单变量操作不可分割，不自动保护复合逻辑 |
| CAS | 值相等才写入，失败时更新 `expected` |
| `weak` vs `strong` | 循环用 weak，单次用 strong |
| `relaxed` | 只保原子性，不保顺序 |
| `release` | 之前的写不能后移，用于发布 |
| `acquire` | 之后的读写不能前移，用于获取 |
| `seq_cst` | 全局总顺序，默认首选 |
| 无锁边界 | 单变量、可回收、防 ABA，否则用锁 |

下一篇进入 `future`/`async`，讲清楚如何把一次任务的返回值、异常和完成状态打包给调用方，`std::async` 的启动策略有什么陷阱，以及 `future` 析构为什么会阻塞。
