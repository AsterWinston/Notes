只要两个线程会同时访问同一个可写变量，就必须有东西把它们隔开。互斥量（mutex）就是最基础的隔离工具：同一时刻只允许一个线程进入临界区。但"加锁"本身也会引入新问题——忘记解锁、锁粒度太粗、两把锁顺序相反导致死锁。本篇把 `std::mutex` 和三种 RAII 锁讲透，并给出锁粒度的工程判断标准。

# 1. 数据竞争与互斥量

## 1.1 竞争是怎么发生的

先看一个看起来毫无问题的计数器：

```cpp
#include <thread>

long long counter = 0;

void add() {
    for (int i = 0; i < 1000000; ++i) {
        ++counter;                 // 读-改-写，三条机器指令
    }
}
// 主线程创建两个 std::thread 调用 add，join 后打印 counter
```

`++counter` 在机器层面大致是"加载到寄存器 → 加一 → 写回内存"。两个线程可能同时加载到旧值 100，各自加一后都写回 101，于是一次自增凭空消失。

★ **关键认知**：**数据竞争**的定义是——两个线程并发访问同一内存位置，至少一个是写，且没有同步。C++ 标准规定数据竞争是**未定义行为**，不是"结果可能不准"这么温和：编译器可以假设它不发生，从而做出任意优化。

## 1.2 mutex 最小示例

```cpp
#include <iostream>
#include <mutex>
#include <thread>

std::mutex mtx;                    // 保护 counter 的互斥量
long long counter = 0;

void add() {
    for (int i = 0; i < 1000000; ++i) {
        std::lock_guard<std::mutex> lock(mtx);   // 构造即加锁
        ++counter;
    }                                            // 离开作用域自动解锁
}

int main() {
    std::thread a(add);
    std::thread b(add);
    a.join();
    b.join();
    std::cout << counter << '\n';  // 稳定输出 2000000
}
```

这里每次循环都加解锁一次，性能很差，但作为正确性示范足够了。真实代码会把整批累加放到局部变量里，最后只加一次锁：

```cpp
void add() {
    long long local = 0;
    for (int i = 0; i < 1000000; ++i) {
        ++local;                   // 局部变量不共享，无需加锁
    }
    std::lock_guard<std::mutex> lock(mtx);
    counter += local;              // 只在这几行临界区里加锁
}
```

★ **关键认知**：互斥量保护的不是"变量"本身，而是**一段访问该变量的代码路径**。所有访问点必须使用**同一个**互斥量，保护才成立；只给写加锁、读不加锁，等于没加。

## 1.3 手工 lock/unlock 的问题

```cpp
void work() {
    mtx.lock();
    if (something_failed()) {
        return;                    // 忘记 unlock，其他线程永久阻塞
    }
    do_something();                // 抛异常也会漏掉 unlock
    mtx.unlock();
}
```

手工配对 `lock`/`unlock` 在提前返回、异常、多分支的情况下极易漏掉。标准库因此提供了 RAII 包装：构造加锁，析构解锁，无论怎么离开作用域都能释放。

⚠ **关键认知**：不要直接调用 `mutex::lock()`/`unlock()`，除非你在实现自己的锁包装。RAII 不只是"少写一行"，而是把"异常安全"和"提前返回安全"这两件事一次性解决。

# 2. 三种 RAII 锁

## 2.1 lock_guard：最轻的守卫

```cpp
{
    std::lock_guard<std::mutex> lock(mtx);   // 加锁
    ++counter;
}                                             // 解锁
```

`lock_guard` 只有构造和析构，没有 `unlock()`、没有移动、没有延迟加锁。它是最轻量的选择，**绝大多数场景优先用它**；C++17 起还可以用 `std::lock_guard lock(mtx);` 省略模板参数。

## 2.2 unique_lock：可延迟、可解锁

```cpp
std::unique_lock<std::mutex> lock(mtx, std::defer_lock);  // 先不加锁
// ... 做一些不需要锁的准备 ...
lock.lock();                                              // 手动加锁
if (should_release) {
    lock.unlock();                                        // 可以中途解锁
}
// 析构时若仍持有锁，会自动解锁
```

`unique_lock` 支持三种构造策略：

| 策略 | 含义 |
|------|------|
| `std::defer_lock` | 构造时不加锁，之后手动 `lock()` |
| `std::try_to_lock` | 尝试加锁，失败不阻塞，用 `owns_lock()` 判断 |
| `std::adopt_lock` | 接管一个已经锁住的 mutex，构造时不再加锁 |

它还能配合 `std::timed_mutex` 使用 `try_lock_for(100ms)`，在超时后返回 `false`，适合"拿不到锁就降级"的路径。

★ **关键认知**：`unique_lock` 比 `lock_guard` 重，因为它要维护"是否持有锁"的状态，还支持移动。只在需要**延迟加锁、中途解锁、超时、配合条件变量**时才用它；普通临界区用 `lock_guard` 即可。

⚠ **关键认知**：条件变量必须搭配 `unique_lock`，因为 `wait` 内部要先解锁再阻塞、醒来后重新加锁，`lock_guard` 没有这个能力。这是第 3 篇的核心机制。

## 2.3 scoped_lock：一次锁多把

```cpp
std::mutex m1, m2;

// C++17：同时锁住两把，内部用死锁避免算法
std::scoped_lock lock(m1, m2);
do_work();
```

`scoped_lock` 是 `lock_guard` 的多锁版本，构造时按标准库的死锁避免算法一次性锁住所有传入的互斥量，析构时逆序解锁。C++11 的等效写法是 `std::lock(m1, m2)` 再配两个 `adopt_lock` 的 `lock_guard`。

| 类型 | 锁数量 | 可延迟 | 可中途解锁 | 典型用途 |
|------|--------|--------|-----------|---------|
| `lock_guard` | 1 | 否 | 否 | 普通临界区 |
| `unique_lock` | 1 | 是 | 是 | 条件变量、超时、灵活控制 |
| `scoped_lock` | 多 | 否 | 否 | 同时锁多把，防死锁 |

★ **关键认知**：需要同时持有两把以上锁时，**首选 `std::scoped_lock`**。它内部会先尝试 `try_lock`，失败就全部释放重来，从根上避免"你等我、我等你"的循环等待。

# 3. 死锁

## 3.1 四个必要条件

死锁同时满足以下四条才会发生：

| 条件 | 含义 | 破坏方式 |
|------|------|---------|
| 互斥 | 资源一次只能被一个线程占用 | 尽量少用锁、用原子/无锁 |
| 持有并等待 | 拿着锁还去申请另一把 | 一次性申请全部锁 |
| 不可抢占 | 别人不能强行夺走锁 | 用带超时的 `try_lock` |
| 循环等待 | 形成 A 等 B、B 等 A 的环 | 全局统一锁顺序 |

## 3.2 代码示例与修复

```cpp
std::mutex m1, m2;

void thread_a() {
    std::lock_guard<std::mutex> a(m1);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));  // 放大交错窗口
    std::lock_guard<std::mutex> b(m2);                          // 等 m2
}

void thread_b() {
    std::lock_guard<std::mutex> b(m2);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard<std::mutex> a(m1);                          // 等 m1
}
```

两个线程各持一把、各等一把，永远解不开。首选修复是 `std::scoped_lock`：

```cpp
void thread_a() {
    std::scoped_lock lock(m1, m2);   // 一次性锁两把，内部避免死锁
    do_work();
}
void thread_b() {
    std::scoped_lock lock(m1, m2);   // 参数顺序无所谓
    do_work();
}
```

另一种是全局固定顺序，所有线程一律先 `m1` 后 `m2`；还可以用带超时的 `try_lock`，拿不到就释放已有锁、退让重试。

★ **关键认知**：死锁不一定发生在两把锁上。**同一把非递归 mutex 被同一线程锁两次**也会自死锁：

```cpp
std::mutex m;
void g() { std::lock_guard<std::mutex> lock(m); }   // 内部锁同一把 m
void f() {
    std::lock_guard<std::mutex> lock(m);
    g();                            // 永久阻塞
}
```

这类问题在"公有函数加锁、内部复用另一个加锁函数"时最常见。解决办法是拆出**不加锁的私有实现**，公有函数负责加锁，内部调用私有版本。

## 3.3 锁顺序规范

当程序里有多把锁时，给它们定一个**全局顺序**并写进文档：

```text
全局锁顺序：配置锁 → 缓存锁 → 会话锁 → 日志锁

规则：任何线程只能按从左到右的顺序申请，
     不允许反向，也不允许跳过后回头申请。
```

⚠ **关键认知**：锁顺序必须靠规范和代码评审维持，编译器帮不了你。一个实用技巧是给锁编号（`config_mutex = 1`、`cache_mutex = 2`），在调试构建里断言"当前线程持有的最大编号小于要申请的编号"。

# 4. 锁粒度

锁粒度指**临界区的范围**。太粗会让并发退化成串行，太细会让加锁开销和复杂度上升。

```cpp
void process(const Request& req) {
    auto result = heavy_compute(req);   // 锁外计算
    save_to_disk(result);               // 锁外 IO
    std::lock_guard<std::mutex> lock(mtx);
    cache_[req.id] = result;            // 只锁容器操作
}
```

★ **关键认知**：临界区的判断标准是"**有没有碰到共享可变数据**"，不是"这段代码重不重要"。计算、格式化、日志、网络请求、文件 IO 都应该尽量放在锁外。锁里只留内存读写。

⚠ **关键认知**：也不要把锁切得过碎。每次 `lock/unlock` 都是一次原子操作加可能的上下文切换；高频小临界区会让锁本身成为瓶颈。经验做法是：先用一把粗锁保证正确，profile 确认锁竞争严重后再细分。

| 粒度 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| 粗（一把大锁） | 简单、不易死锁 | 并发度低 | 原型、低竞争 |
| 细（每对象一把） | 并发度高 | 复杂、易死锁 | 热路径、大对象池 |
| 读写锁 | 读多写少时并发高 | 写饥饿、实现复杂 | 配置、缓存 |

读写锁用 `std::shared_mutex`（C++17）：

```cpp
std::shared_mutex rw_mtx;
Config cfg;

Config read_config() {
    std::shared_lock lock(rw_mtx);   // 多个读者可同时进入
    return cfg;
}
void write_config(const Config& next) {
    std::unique_lock lock(rw_mtx);   // 写者独占
    cfg = next;
}
```

# 5. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 程序卡住不动 | 同一线程重复锁非递归 mutex | 拆出无锁私有函数，或改用 `recursive_mutex` |
| 偶发卡死 | 两把锁顺序相反 | `scoped_lock` 或全局锁顺序 |
| 提前 return 后别人卡死 | 手工 `lock` 忘记 `unlock` | 一律用 RAII |
| 加了锁还是算错 | 有的访问点没加同一把锁 | 检查所有读写路径 |
| 性能随线程数下降 | 锁粒度过粗、竞争激烈 | 缩小临界区、分片锁 |
| 持锁调用回调导致死锁 | 回调里又申请同一把锁 | 锁外调用外部代码 |

⚠ **关键认知**：**绝对不要在持锁时调用未知代码**——回调、虚函数、用户提供的比较器、日志 sink 都可能回头申请同一把锁。稳妥做法是把数据拷贝出来，解锁后再调用。

```cpp
std::vector<Callback> local;
{
    std::lock_guard<std::mutex> lock(mtx);
    local = callbacks_;            // 锁内只拷贝
}
for (auto& cb : local) {
    cb(event);                     // 锁外调用，避免回调回头加锁
}
```

# 6. 动手练习

1. 把开头的计数器改成"每个线程本地累加、最后加一次锁"，用 `time` 比较两种写法在 4 线程下的耗时差异。
2. 写两个线程分别以相反顺序锁 `m1`、`m2`，观察死锁；再用 `std::scoped_lock` 修复，确认程序能稳定退出。
3. 实现一个线程安全的 `Counter` 类，提供 `increment()`、`value()`，要求 `value()` 也不读到半更新状态；再把它改成用 `std::atomic<long long>`，比较两种实现的复杂度。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 数据竞争 | 并发读写同一位置且无同步，属于未定义行为 |
| 保护范围 | 是访问路径，不是变量本身 |
| `lock_guard` | 最轻，构造锁析构解，默认首选 |
| `unique_lock` | 可延迟、可解锁、可超时，条件变量必需 |
| `scoped_lock` | 一次锁多把，内部避免死锁 |
| 死锁四条件 | 互斥、持有并等待、不可抢占、循环等待 |
| 锁顺序 | 全局固定，禁止反向申请 |
| 锁粒度 | 只包共享内存访问，IO 和计算放锁外 |

下一篇进入条件变量，讲清楚"等待某个条件成立"为什么不能靠轮询、`wait` 为什么要和 `unique_lock` 配合、虚假唤醒和丢失唤醒分别怎么防，并用一个有界阻塞队列把生产者消费者完整实现出来。
