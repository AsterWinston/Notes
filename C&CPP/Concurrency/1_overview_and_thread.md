并发是 C++ 里"看起来简单、写对很难"的典型代表：创建线程只需要一行 `std::thread`，但线程什么时候启动、参数怎么传、对象析构时该 `join` 还是 `detach`，每一步都埋着会让程序直接崩溃的细节。本篇从并发与并行的区别讲起，把 `std::thread` 的创建、传参、生命周期和硬件并发数一次说清。

# 1. 并发与并行

## 1.1 两者到底差在哪

很多人把"并发"和"并行"当同义词，其实它们描述的是两件事：

| 概念 | 关注点 | 典型场景 |
|------|--------|---------|
| 并发（concurrency） | 多个任务在**同一时间段内**推进 | 单核上时间片轮转、IO 等待时切换任务 |
| 并行（parallelism） | 多个任务在**同一时刻**真正同时执行 | 多核 CPU 同时跑多个计算线程 |

一句话：**并发是"结构"，并行是"执行"**。你可以写一个并发程序，但它跑在单核机器上，任何时刻只有一个线程在运行；也可以写一个并行程序，用四个核把一次矩阵乘法切开。

★ **关键认知**：并发解决的是"任务如何组织、如何互不阻塞"，并行解决的是"如何用更多硬件同时算"。单核也能并发，靠的是快速切换；多核才谈得上并行，靠的是真同时。

## 1.2 为什么需要并发

- **响应性**：界面线程不能被文件读写卡住，耗时工作交给后台线程。
- **吞吐量**：服务器同时处理多个连接，等待网络时去服务别人。
- **利用多核**：计算密集型任务拆开并行，缩短总时间。
- **结构清晰**：把定时任务、监听任务、工作任分别建模，比一个巨大的事件循环更好维护。

⚠ **关键认知**：并发**不等于更快**。线程创建、上下文切换、锁竞争、缓存一致性流量都是成本。一个本身只需 1 毫秒的计算，拆成 8 个线程可能反而要 3 毫秒。只有当任务足够重、或者存在大量等待时，并发才划算。

## 1.3 并发的代价

| 代价 | 说明 |
|------|------|
| 数据竞争 | 多线程同时读写同一变量，行为未定义 |
| 死锁 | 互相等待对方持有的锁，全部卡死 |
| 调试困难 | 问题依赖时序，加日志就复现不了 |
| 上下文切换 | 线程切换要保存/恢复寄存器，打乱缓存 |
| 内存开销 | 每个线程默认栈通常 1-8 MB |

★ **关键认知**：并发 bug 的根源是**共享可变状态**。只要两个线程能同时看到一个可写变量，就必须有明确的同步手段；没有共享，就没有竞争。

# 2. std::thread 入门

## 2.1 最小可运行示例

```cpp
#include <iostream>
#include <thread>

void hello(int id) {
    std::cout << "线程 " << id << " 开始运行\n";
}

int main() {
    std::thread t1(hello, 1);        // 构造即启动，参数按值拷贝进去
    std::thread t2(hello, 2);        // 第二个线程

    t1.join();                       // 主线程阻塞，等 t1 结束
    t2.join();                       // 等 t2 结束
    std::cout << "主线程结束\n";
    return 0;
}
```

编译时要链接线程库：

```bash
g++ -std=c++20 -pthread main.cpp -o main
clang++ -std=c++20 -pthread main.cpp -o main
```

Windows 上 MSVC 不需要额外链接参数，`cl /std:c++20 main.cpp` 即可。

逐段说明：

- `std::thread t1(hello, 1);`：把函数 `hello` 和参数 `1` 打包成一个新线程。**构造函数返回时线程已经开始跑**，不是"准备好等你启动"。
- `t1.join()`：当前线程（这里是 `main`）阻塞，直到 `t1` 执行完。一个线程对象只能 `join` 一次。
- 不写 `join` 也不写 `detach` 就离开作用域，`std::thread` 的析构函数会调用 `std::terminate`。

★ **关键认知**：`std::thread` 的构造**立即启动**线程。你无法先创建一个"暂停的线程"再手动启动；需要延迟执行就把逻辑放进任务队列（第 6 篇）。

## 2.2 参数传递

线程函数的参数默认**按值拷贝**到线程的存储里：

```cpp
void work(std::string msg, int n) {
    std::cout << msg << " " << n << '\n';
}

std::string text = "任务";
std::thread t(work, text, 3);     // text 被拷贝一份，主线程再改也不影响
text = "改过了";                   // 安全：线程里拿的是副本
t.join();
```

想传引用必须显式用 `std::ref`：

```cpp
void accumulate(std::vector<int>& out, int value) {
    out.push_back(value);          // 直接改调用方的 vector
}

std::vector<int> data;
std::thread t(accumulate, std::ref(data), 42);   // 不加 ref 会编译失败
t.join();
// 此处 data 里有 42；前提是 join 保证了线程已经结束
```

★ **关键认知**：按值拷贝是有意设计的——它避免了"主线程的局部变量先销毁、线程还在用引用"这类悬垂问题。需要共享写入时才用 `std::ref`，并且必须自己保证对象活到线程结束。

⚠ **关键认知**：`std::ref` 只负责"传引用"，**不负责延长生命周期**。下面这段是未定义行为：

```cpp
void work(const std::string& s) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << s << '\n';        // 一秒后读，但对象已经销毁
}

void start() {
    std::string text = "局部字符串";
    std::thread t(work, std::ref(text));   // 传引用，不拷贝
    t.detach();                            // 不等待
}   // text 在这里销毁，工作线程仍持有它的引用
```

`start` 返回后局部对象 `text` 立即销毁，而分离出去的工作线程可能一秒后才读它。正确做法是按值传递，或者让对象活到线程结束（`join` 或共享所有权）。

## 2.3 线程标识与硬件并发数

```cpp
#include <iostream>
#include <thread>

int main() {
    std::cout << "硬件并发数: "
              << std::thread::hardware_concurrency() << '\n';   // 常见为逻辑核数

    std::thread t([] {
        std::cout << "工作线程 id: " << std::this_thread::get_id() << '\n';
    });
    std::cout << "主线程 id: " << std::this_thread::get_id() << '\n';
    t.join();
    return 0;
}
```

`hardware_concurrency()` 返回的是**建议值**，可能是 0（无法获取时），也可能包含超线程的逻辑核。它适合作为线程池默认大小的起点，不适合当作精确的物理核数。

★ **关键认知**：`std::thread::id` 可以比较、可以哈希，常用来做"每个线程一份"的 `thread_local` 缓存键或日志标记。它**不保证是整数**，不要强转成 `int` 做业务逻辑。

# 3. 线程生命周期：join 与 detach

## 3.1 join：等待并回收

```cpp
std::thread t([]{ /* 干活 */ });
t.join();                     // 阻塞直到 t 结束
// 此后 t 不再关联任何线程，joinable() == false
```

`join` 做两件事：等待线程函数返回，然后回收线程资源。它必须由**另一个线程**调用——线程不能 `join` 自己。

```cpp
std::thread t([]{ std::this_thread::get_id(); });
// t.join();  主线程调用，合法
// 在线程内部调用 t.join() 会抛 std::system_error
```

## 3.2 detach：放养

```cpp
std::thread t([] {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "后台任务完成\n";
});
t.detach();                    // 与线程对象脱钩，线程独立运行
// t 现在 joinable() == false，析构不会 terminate
```

`detach` 之后你**再也无法等待它、无法知道它何时结束、无法回收它的结果**。它适合真正的守护型任务，比如心跳、日志刷盘；不适合"我待会儿要结果"的计算。

⚠ **关键认知**：`detach` 最大的风险是**访问已销毁的对象**。线程捕获了 `this`、局部变量引用、或者指向栈上的指针，主线程一返回，这些东西全没了，线程还在跑。工程上更推荐"不 detach，用明确的关闭协议 join"（第 6 篇）。

## 3.3 析构前必须做决定

`std::thread` 的析构函数检查 `joinable()`：如果仍关联着一个线程，直接 `std::terminate`。这是标准委员会故意的设计——**不允许悄悄地丢弃一个还在跑的线程**。

```cpp
void bad() {
    std::thread t([]{ /* 长任务 */ });
}                                  // t 析构，joinable 为真，程序终止
```

三种正确写法：

```cpp
// 写法一：join
std::thread t1(work);
t1.join();

// 写法二：detach（确认生命周期安全）
std::thread t2(work);
t2.detach();

// 写法三：C++20 的 jthread，析构时自动 request_stop + join
std::jthread t3(work);             // 离开作用域自动 join
```

用 RAII 包装 `std::thread` 也很常见：

```cpp
class ThreadGuard {
public:
    explicit ThreadGuard(std::thread t) : t_(std::move(t)) {}
    ~ThreadGuard() {
        if (t_.joinable()) {
            t_.join();             // 保证不会 terminate
        }
    }
    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;
private:
    std::thread t_;
};
```

线程对象的状态迁移：

```text
构造成功
   │
   ▼
joinable == true
   │
   ├── join()   ──► 等待结束 ──► joinable == false
   ├── detach() ──► 脱钩     ──► joinable == false
   └── 析构     ──► std::terminate（不允许）
```

★ **关键认知**：`joinable()` 为真只表示"对象还关联着一个线程"，**不表示线程还活着**。线程可能早已执行完，只是你还没 `join` 回收。判断"是否还在运行"没有标准接口，只能靠自己的标志位或 `future`。

# 4. 常见坑

| 现象 | 原因 | 修复 |
|------|------|------|
| 程序无输出直接 abort | 线程对象析构时仍 joinable | `join`、`detach` 或改用 `jthread` |
| `std::system_error: Invalid argument` | 对同一线程 `join` 两次 | 每次 join 前检查 `joinable()` |
| 线程里读到乱码/崩溃 | 传了局部变量的引用或指针 | 按值捕获，或保证生命周期 |
| 输出交错混乱 | 多线程同时写 `std::cout` | 每次输出加锁，或汇总后统一打印 |
| 以为创建线程就有加速 | 任务太小，开销大于收益 | 用线程池，批量任务 |
| 在成员函数里起线程捕获 `this` | 对象先析构 | 用 `shared_ptr` + `weak_ptr`，或确保 join |

⚠ **关键认知**：`std::cout` 本身不会因为多线程产生数据竞争（标准库保证字符级不撕裂），但**多次 `<<` 之间可能被别的线程插入**，所以输出会交错。要整行原子输出，得自己加锁或先拼成 `std::string` 再一次写出。

```cpp
std::mutex io_mutex;
void log(const std::string& line) {
    std::lock_guard<std::mutex> lock(io_mutex);   // 保证整行连续
    std::cout << line << '\n';
}
```

# 5. 动手练习

1. 启动 4 个线程，每个线程打印自己的 `std::thread::id` 和 `hardware_concurrency()`，主线程全部 `join` 后退出。观察多次运行输出顺序是否变化。
2. 写一个函数 `sum_range(const std::vector<int>&, size_t begin, size_t end, long long& out)`，用两个线程分别算前半段和后半段，用 `std::ref` 传结果，`join` 后比较总和是否正确。把 `std::ref` 去掉，看编译报什么错。
3. 故意写一个"线程对象离开作用域但不 join"的程序，用 `gdb` 或查看退出码确认它触发了 `std::terminate`；再改成 `std::jthread` 验证正常退出。

# 6. 小结

| 概念 | 要点 |
|------|------|
| 并发 vs 并行 | 并发是任务组织，并行是同时执行 |
| 创建线程 | `std::thread` 构造即启动 |
| 传参 | 默认按值拷贝，引用用 `std::ref` |
| `join` | 阻塞等待并回收，只能一次 |
| `detach` | 脱钩后无法等待，生命周期自负 |
| 析构 | 仍 joinable 就 `std::terminate` |
| `jthread` | C++20，析构自动 join，可协作取消 |
| 并发数 | `hardware_concurrency()` 只是建议值 |

下一篇进入互斥量与锁，讲清楚数据竞争是怎么产生的、`lock_guard`/`unique_lock`/`scoped_lock` 各自适合什么场景、死锁的四个条件和避免办法，以及锁粒度该如何权衡。
