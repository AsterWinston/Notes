并发 bug 最让人头疼的地方是"不可复现"：单线程跑一万次都对，压测时偶尔崩一次，加个日志又好了。这不是玄学，而是时序、缓存和编译器优化共同作用的结果。本篇把数据竞争、死锁、ABA、迭代器失效四类问题逐个拆开，再给出 ThreadSanitizer 和 gdb 的实操排查路径。

# 1. 数据竞争

## 1.1 典型形态

```cpp
long long total = 0;

void worker() {
    for (int i = 0; i < 1000000; ++i) {
        ++total;                 // 读-改-写，两个线程同时执行
    }
}
```

`++total` 在机器层面是"加载 → 加一 → 写回"三步。两个线程可能同时读到旧值 100，各自加一后都写回 101，于是一次自增凭空消失。

更隐蔽的是"只在某些路径写"：一个线程无锁读 `cfg.ready`，另一个线程持锁写 `cfg.ready` 和 `cfg.value`。两个变量独立，即使 `ready` 最后写，别的线程也可能因重排先看到 `ready == true`、后看到旧的 `value`。

★ **关键认知**：数据竞争的定义是——两个线程并发访问同一内存位置，至少一个是写，且没有同步。C++ 标准规定数据竞争是**未定义行为**，不是"结果可能不准"这么温和：编译器可以假设它不发生，从而做出任意优化。

## 1.2 双重检查锁定的经典错误

```cpp
Singleton* instance = nullptr;
std::mutex mtx;

Singleton* get_bad() {
    if (instance == nullptr) {              // 第一次检查：无锁
        std::lock_guard<std::mutex> lock(mtx);
        if (instance == nullptr) {          // 第二次检查
            instance = new Singleton();     // 构造与指针赋值可能重排
        }
    }
    return instance;                        // 可能拿到未构造完成的对象
}
```

`new Singleton()` 包含"分配内存 → 构造对象 → 返回指针"三步。指针赋值可能先于构造完成可见，另一个线程在第一次检查时看到非空指针，直接使用一个还没构造好的对象。

C++11 起最简单的修复是局部静态变量：

```cpp
Singleton& get() {
    static Singleton instance;   // 标准保证线程安全的惰性初始化
    return instance;
}
```

必须用指针时，写线程在完整构造后用 `memory_order_release` 发布，读线程用 `acquire` 读取，看到非空即代表构造完成。

⚠ **关键认知**：`volatile` 修不了双重检查锁定。它不建立 happens-before，也不阻止构造与指针赋值重排。这个模式在 Java 里需要 `volatile`，在 C++ 里必须用 `atomic` 或局部静态。

## 1.3 检测手段

代码审查靠经验；ThreadSanitizer 只报执行到的竞争，有漏报；Helgrind 慢且误报较多；压力测试只能提高复现概率，不能证明没有。

# 2. 死锁

## 2.1 复现

```cpp
std::mutex m1, m2;

void a() {
    std::lock_guard<std::mutex> g1(m1);
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    std::lock_guard<std::mutex> g2(m2);   // 等 m2
}

void b() {
    std::lock_guard<std::mutex> g2(m2);
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    std::lock_guard<std::mutex> g1(m1);   // 等 m1
}
```

两个线程各持一把、各等一把。`sleep` 是"放大交错窗口"，真实代码里窗口可能只有几条指令，所以更难复现。

## 2.2 用 gdb 定位

启动 gdb 并运行，卡住时按 `Ctrl+C` 中断，然后查看所有线程：

```bash
gdb ./app
(gdb) info threads
(gdb) thread apply all bt
```

典型输出（节选）：

```text
Thread 3 in __lll_lock_wait ()
#2 b() at main.cpp:18
Thread 2 in __lll_lock_wait ()
#2 a() at main.cpp:10
```

两个线程都停在 `pthread_mutex_lock`，一个在 `a()` 等 `m2`，一个在 `b()` 等 `m1`，循环等待成立。glibc 的 `pthread_mutex_t` 里有 `__owner`（持有线程 id），切到对应栈帧 `p m1` 即可确认。也可以直接 attach：`gdb -p <pid>`。Windows 上可用 Visual Studio 的"并行堆栈"窗口，或在 WinDbg 里 `~*k` 查看所有线程栈。

★ **关键认知**：死锁现场的价值在于**同时看到所有线程的栈和锁地址**。只看一个线程永远只能看到"它在等锁"，不知道锁被谁持有。`thread apply all bt` 是排查死锁的第一条命令。

## 2.3 修复与预防

```cpp
// 方案一：scoped_lock 一次性锁多把
void a() { std::scoped_lock lock(m1, m2); do_a(); }
void b() { std::scoped_lock lock(m1, m2); do_b(); }

// 方案二：全局锁顺序，所有线程先 m1 后 m2
void b_fixed() {
    std::lock_guard<std::mutex> g1(m1);
    std::lock_guard<std::mutex> g2(m2);
}
```

预防的核心是**规范**：给锁编号、固定申请顺序、禁止持锁调用外部代码、禁止同一线程重复锁非递归 mutex。

# 3. ABA

## 3.1 场景

无锁栈的出栈操作：

```cpp
struct Node { int value; Node* next; };
std::atomic<Node*> head{nullptr};

Node* pop() {
    Node* old_head = head.load(std::memory_order_acquire);
    while (old_head) {
        Node* new_head = old_head->next;      // ① 读 next
        if (head.compare_exchange_weak(old_head, new_head)) {
            return old_head;                  // ② 成功弹出
        }
    }
    return nullptr;
}
```

问题出在 ① 和 ② 之间：

```text
线程 A：读到 head = A，next = B             ← 暂停
线程 B：pop A，pop B，delete A、delete B
线程 B：new 一个节点，恰好分配到 A 的地址，push 回去
线程 A：恢复，CAS(head, A, B) 成功
        ── 但 B 已被释放，head 变成悬垂指针
```

CAS 只比较指针值，地址被复用后，"还是 A"并不代表"还是同一个 A"。这就是 **ABA 问题**。

## 3.2 防护手段

| 方案 | 思路 | 代价 |
|------|------|------|
| 带标签指针 | 指针 + 版本号打包进一个机器字，CAS 同时比较版本 | 需要双字 CAS 或地址对齐技巧 |
| Hazard pointer | 每个线程登记正在访问的指针，删除前检查 | 实现复杂，有额外读开销 |
| 引用计数 | 节点带原子计数，弹出前增减 | 计数竞争成为瓶颈 |
| Epoch/RCU | 延迟回收，等所有读者离开临界区 | 内存占用高，适合读多写少 |
| 直接加锁 | 放弃无锁 | 最简单，通常够用 |

★ **关键认知**：ABA 的根源不是 CAS 写错了，而是**内存回收与无锁算法的交互**。只要节点会被释放、地址会被复用，单纯的指针 CAS 就不安全。解决 ABA 往往要连同"安全回收"一起解决。

⚠ **关键认知**：ABA 在"节点不释放"的池化结构里不会造成悬垂，但仍可能造成逻辑错误（例如栈长度统计错误）。判断是否需要防护，要看"地址复用后旧值是否还有语义"。

# 4. 迭代器失效

## 4.1 容器与并发

```cpp
std::vector<int> data;
std::mutex mtx;

void writer() {
    for (int i = 0; i < 1000; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        data.push_back(i);          // 可能重新分配，旧迭代器全失效
    }
}
void reader_bad() {
    for (int v : data) process(v);  // 没加锁，且迭代器可能失效
}
```

这里有两个独立问题：读写 `data` 没有同步，属于 UB；`push_back` 触发扩容时旧内存被释放，正在遍历的迭代器指向已释放内存。即使用锁把遍历包起来，长时间持锁也会阻塞写线程。

## 4.2 修复方式

```cpp
// 锁内拷贝快照，锁外处理
std::vector<int> snapshot;
{
    std::lock_guard<std::mutex> lock(mtx);
    snapshot = data;                // 拷贝一份，锁外随便遍历
}
for (int v : snapshot) {
    process(v);
}
// 另一种做法是锁内完成遍历，但临界区里只能做内存操作
```

| 容器 | 什么操作会让迭代器失效 |
|------|----------------------|
| `vector` | 扩容、`insert`/`erase` 位置之后 |
| `deque` | 中间插入/删除、首尾操作可能影响 |
| `list`/`forward_list` | 只有被删除元素失效 |
| `map`/`set` | 只有被删除元素失效 |
| `unordered_*` | 扩容（rehash）让全部失效 |

⚠ **关键认知**：**标准容器都不是线程安全的**。多个线程同时读同一个容器是安全的（只要没有写），但只要有一个线程写，所有访问都必须同步。`const` 方法也不保证安全——`operator[]` 的 `const` 版本可能触发内部缓存或惰性计算。

# 5. ThreadSanitizer

## 5.1 编译与运行

```bash
g++ -std=c++20 -g -O1 -fsanitize=thread -pthread main.cpp -o app_tsan
clang++ -std=c++20 -g -O1 -fsanitize=thread -pthread main.cpp -o app_tsan
./app_tsan
```

关键参数：

- `-g` 保留调试信息，报告才能定位到行号；`-pthread` 链接线程库。
- `-O1` 需要一定优化来暴露重排，但不要用 `-O2` 以上，避免报告失真。
- MSVC 没有 TSan，Windows 上用 WSL/Clang，或改用 Visual Studio 的并发分析工具。

环境变量 `TSAN_OPTIONS="halt_on_error=1 second_deadlock_stack=1" ./app_tsan` 能让第一次报错就退出，方便在 CI 里当门禁。

## 5.2 读报告

```text
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 8 at 0x... by thread T2:
    #0 worker() main.cpp:8
  Previous read of size 8 at 0x... by thread T1:
    #0 worker() main.cpp:8
  Location is global 'total' of size 8
```

阅读顺序：先看冲突类型（`data race` 还是 `lock-order-inversion`），再看两个访问的读写方和源码行，然后确认内存位置是不是同一对象，最后用线程创建栈还原调用链。

★ **关键认知**：TSan 基于 happens-before 分析，**只报告实际执行到的冲突**。测试没覆盖到的路径它发现不了；反过来，它报出来的竞争都是真问题，不要当成误报忽略。

⚠ **关键认知**：TSan 不能和 AddressSanitizer 同时开。检查内存越界用 ASan，检查数据竞争用 TSan，分两次跑。TSan 会让程序慢 5-15 倍、内存涨 5-10 倍，不适合压测，适合功能测试和 CI。

# 6. gdb 排查

| 命令 | 用途 |
|------|------|
| `info threads` | 列出所有线程及其当前栈帧 |
| `thread <n>` | 切到第 n 个线程 |
| `thread apply all bt` | 打印所有线程调用栈（死锁首选） |
| `p <var>` / `p *ptr` | 打印变量 / 解引用指针 |
| `watch <var>` | 监视变量变化（多线程下命中不稳定） |

排查"偶发崩溃"时，先开 core dump，再离线分析：

```bash
ulimit -c unlimited
./app
gdb ./app core
(gdb) thread apply all bt
```

★ **关键认知**：并发 bug 加日志可能改变时序、让问题消失（海森堡 bug）。优先用 TSan 和 core dump 保留现场；必须加日志时，用无锁的环形缓冲区先记录，崩溃后再导出。

# 7. 排查清单

| 症状 | 优先怀疑 | 工具 |
|------|---------|------|
| 计数偏小、结果随机 | 数据竞争 | TSan |
| 程序完全卡死 | 死锁、忘记 notify | `thread apply all bt` |
| 偶发段错误 | 悬垂指针、迭代器失效 | ASan、core dump |
| 无锁结构偶发错误 | ABA、内存回收 | 代码审查、带标签指针 |
| 关闭时卡住 | 条件变量未唤醒 | 检查 shutdown 顺序 |
| 只有 release 崩 | UB 被优化放大 | TSan + `-O1` 复现 |

⚠ **关键认知**：**"加锁了就安全"是错觉**。锁必须保护所有访问路径，且保护的是同一份数据。一个常见错误是"读用原子、写用锁"，两者不在同一个同步域里，依然构成竞争。

# 8. 动手练习

1. 运行开头的计数器竞争示例 10 次，记录结果分布；再用 TSan 编译运行，把报告里的"两个访问"对应到源码行。
2. 复现本篇的死锁，用 `gdb -p` attach 后执行 `thread apply all bt`，找出两个线程各自持有的锁；再用 `std::scoped_lock` 修复。
3. 写一个"一个线程 `push_back`、另一个线程范围 for 遍历"的程序，用 ASan 观察堆释放后使用；再用"锁内拷贝快照"修复并验证。

# 9. 小结

| 问题 | 本质 | 典型修复 |
|------|------|---------|
| 数据竞争 | 无同步的并发读写，UB | 锁、原子、消除共享 |
| 死锁 | 循环等待 | 固定锁顺序、`scoped_lock` |
| ABA | 地址复用让 CAS 误判 | 标签指针、hazard pointer |
| 迭代器失效 | 容器修改导致指针悬垂 | 快照、锁内完成、换容器 |
| TSan / gdb | 竞争检测 / 保留现场看全栈 | `-fsanitize=thread -g -O1`、`thread apply all bt` |

下一篇是模块收尾，把前面七篇的原则收拢成一套可执行的最佳实践：如何从设计上避免共享、为什么优先消息传递、锁顺序和锁粒度怎么定、性能与可维护性如何权衡，并给出上线前的检查清单。
