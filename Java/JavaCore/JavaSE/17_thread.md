多线程是 Java 语言内置的核心能力，也是从"会写语法"跨到"能写服务端程序"的分水岭。本篇讲清**怎么创建线程、线程有哪些状态、为什么会线程不安全、怎么用锁和等待唤醒机制协调线程**，最后落到**线程池**这一生产环境唯一推荐的用法。关于锁的底层实现（对象头、锁升级、AQS、CAS、各种锁的分类），第 30 篇有专门一章展开，本篇只讲"怎么用"和"为什么这么用"。

# 1. 进程、线程与并发的基本概念

**进程**是操作系统分配资源的基本单位，一个进程拥有独立的内存空间；**线程**是操作系统调度的基本单位，是进程内的一条执行路径。一个进程至少有一个线程（Java 里就是 main 线程），同一个进程内的多个线程**共享堆内存和方法区**，但**各自拥有独立的栈和程序计数器**。

这一点直接决定了后续所有并发问题的形态：

| 内存区域 | 是否共享 | 并发影响 |
|----------|----------|----------|
| 堆（对象实例） | 共享 | 多线程读写同一对象 → 线程安全问题 |
| 方法区（类信息、静态变量） | 共享 | 静态变量同样是共享资源 |
| 虚拟机栈（局部变量、方法调用） | 不共享 | 局部变量天然线程安全 |
| 程序计数器 | 不共享 | 记录当前线程执行到哪条字节码 |

**并发（Concurrency）** 是多个任务交替执行（宏观同时、微观串行），**并行（Parallelism）** 是多个任务真正同时执行。单核 CPU 上只能并发，多核才能并行。Java 的多线程是**抢占式调度**——线程何时被 CPU 切走由操作系统决定，代码无法干预（只能通过优先级给出"建议"）。

> **注意**：正因为调度是抢占式的，"按顺序写就按顺序跑"这个直觉在多线程下不再成立，这是后面所有线程安全问题的总根源。

# 2. 创建线程的三种方式

## 2.1 方式一：继承 Thread 类

`Thread` 类本身代表一个线程，继承它并重写 `run()` 方法，`run()` 里就是线程要执行的任务。

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            // Thread.currentThread() 拿到当前正在执行的线程对象
            System.out.println(Thread.currentThread().getName() + " 输出 " + i);
        }
    }
}

// 使用
public static void main(String[] args) {
    MyThread t1 = new MyThread();
    t1.setName("线程一");
    t1.start();     // 启动线程：由 JVM 通知操作系统创建新线程，再调用 run()
    // t1.run();    // ✗ 错误：这只是普通方法调用，不会开新线程，而是在 main 线程里跑
}
```

**这种方式扩展性差**：Java 是单继承，一旦 `extends Thread`，这个类就再也无法继承其他父类。因此实际开发中很少用。

## 2.2 方式二：实现 Runnable 接口

把"任务"和"线程"拆开——`Runnable` 只描述任务，`Thread` 负责执行。

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        // 实现 Runnable 时没有 getName() 可用，要用静态方法 currentThread() 间接获取
        System.out.println(Thread.currentThread().getName() + " 正在执行");
    }
}

// 使用
MyRunnable mr = new MyRunnable();
Thread t1 = new Thread(mr, "线程一");   // 构造方法第二个参数直接指定线程名
t1.start();
```

**优势**：避免了单继承的局限，一个任务可以被多个线程复用（同一 `Runnable` 对象可以交给多个 `Thread`），任务与线程解耦，更符合面向对象设计。**局限**：`run()` 没有返回值，也无法抛出受检异常。

## 2.3 方式三：实现 Callable 接口 + FutureTask

需要线程有返回值时用这套。`Callable` 是带泛型的函数式接口，`call()` 有返回值且可以抛异常；`FutureTask` 同时实现了 `Runnable` 和 `Future`，起到**适配器**作用——它本身能被 `Thread` 接收，又能通过 `get()` 拿到结果。

```java
public class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        int sum = 0;
        for (int i = 1; i <= 100; i++) {
            sum += i;
        }
        return sum;     // 有返回值，且可以抛异常
    }
}

// 使用
MyCallable mc = new MyCallable();
FutureTask<Integer> ft = new FutureTask<>(mc);   // FutureTask 既实现 Runnable 又实现 Future
new Thread(ft, "计算线程").start();

Integer result = ft.get();   // 阻塞等待线程执行完毕并获取结果
System.out.println(result);  // 5050
```

`FutureTask` 的关键点：

| 方法 | 作用 |
|------|------|
| `get()` | **阻塞**直到任务完成并返回结果；任务抛异常时，这里会抛出包装后的 `ExecutionException` |
| `get(long, TimeUnit)` | 带超时的获取，超时抛 `TimeoutException` |
| `isDone()` | 任务是否已完成（正常结束、异常结束、取消都算完成） |
| `cancel(boolean)` | 尝试取消任务，`true` 表示中断正在执行的线程 |

> **注意**：`Thread` 本身没有直接接收 `Callable` 的构造方法，必须靠 `FutureTask` 中转。真正生产环境不会这么写——都是把 `Callable` 提交给线程池，拿到的 `Future` 语义完全一致（见 8.2）。

## 2.4 三种方式对比

| 维度 | 继承 Thread | 实现 Runnable | Callable + FutureTask |
|------|------------|--------------|----------------------|
| 是否受单继承限制 | 是 | 否 | 否 |
| 能否获取返回值 | 否 | 否 | **能** |
| 能否抛出受检异常 | 否 | 否 | **能** |
| 任务能否被多线程复用 | 否 | 是 | 是 |
| 推荐程度 | ✗ 不推荐 | ✓ 常用 | ✓ 需要返回值时用 |

**实际开发中的第四种方式**（也是最推荐的）：**不自己 `new Thread`，而是把任务提交给线程池**。见第 8 节。

# 3. Thread 常用 API

## 3.1 基础信息类

| 方法 | 说明 |
|------|------|
| `String getName()` | 获取线程名称。默认名字形如 `Thread-0`、`Thread-1` |
| `void setName(String name)` | 设置线程名，建议在 `start()` 之前设置 |
| `static Thread currentThread()` | 获取当前正在执行的线程对象。**静态方法**，用 `Thread.currentThread()` 调用 |
| `long getId()` | 线程的唯一标识（JDK19 后废弃，推荐用 `threadId()`） |
| `static boolean holdsLock(Object obj)` | 当前线程是否持有指定对象的锁 |

## 3.2 控制与调度类

| 方法 | 说明 |
|------|------|
| `static void sleep(long millis)` | 让当前线程睡眠指定毫秒，**不释放锁**，醒来后进入就绪状态 |
| `static void yield()` | 礼让：当前线程回到就绪状态，让出一次 CPU，但下一次仍可能立刻抢到，**不保证让成功** |
| `void join()` | 插队：在 A 线程中调用 `b.join()`，A 必须等 b 执行完才继续。常用于"等子线程跑完再统计结果" |
| `void join(long millis)` | 最多等指定毫秒 |
| `void interrupt()` | 打断线程（不是杀死）。若线程正阻塞在 `sleep/wait/join`，会抛 `InterruptedException` 并**清除中断标记**；否则只是把中断标记置为 `true` |
| `static boolean interrupted()` | 判断当前线程是否被中断，**并清除标记** |
| `boolean isInterrupted()` | 判断是否被中断，**不清除标记** |
| `final void setDaemon(boolean on)` | 设为守护线程。所有非守护线程结束后，守护线程会被 JVM 陆续终止，即使没跑完 |
| `void setPriority(int p)` | 设置优先级 1~10，默认 5。**只是提高抢占概率，不保证顺序**，且强依赖操作系统实现 |
| `static void onSpinWait()` | 提示 CPU 当前处于自旋等待（JDK9+），可用于忙等场景降低功耗 |

几个容易混淆的点：

**sleep 与 wait 的区别**（高频面试题，详见第 6 节）：

| 维度 | `Thread.sleep()` | `Object.wait()` |
|------|-----------------|----------------|
| 所属类 | `Thread` 的静态方法 | `Object` 的成员方法 |
| 是否释放锁 | **不释放** | **释放** |
| 调用前提 | 任何位置都能调用 | 必须在 `synchronized` 同步块/方法中，且用锁对象调用 |
| 唤醒方式 | 时间到了自动醒 | 需要 `notify()/notifyAll()` 或超时 |
| 线程状态 | `TIMED_WAITING` | `WAITING` 或 `TIMED_WAITING` |

**join 的底层**：`join()` 内部就是 `while (isAlive()) { wait(0); }` —— 所以 `join` 本质是等对方线程对象的 `wait`，对方线程结束时 JVM 会调用 `notifyAll()` 唤醒。

**守护线程的典型用法**：GC 线程、日志后台刷盘线程、心跳检测线程都是守护线程。

> **陷阱**：不要把业务任务放进守护线程。main 线程一结束，守护线程会被 JVM 直接终止，没跑完的任务就丢了。

```java
// join 实战：主线程等待两个子线程都算完，再汇总
Thread t1 = new Thread(() -> sum.addAndGet(1));
Thread t2 = new Thread(() -> sum.addAndGet(2));
t1.start();
t2.start();
t1.join();
t2.join();
System.out.println(sum.get());   // 保证是 3
```

# 4. 线程的生命周期与状态

## 4.1 操作系统视角的六种状态

传统的操作系统教材把线程分为：新建、就绪、运行、阻塞、等待、结束。其中**就绪和运行之间的切换由 CPU 调度决定**，代码无法控制。

## 4.2 Java 视角的六种状态

Java 把状态收敛为 `Thread.State` 枚举，**通过 `thread.getState()` 获取**（注意 `BLOCKING` 和 `WAITING` 在操作系统层面都属于"阻塞"，Java 划分得更细）：

| 状态 | 枚举值 | 触发条件 | 出口 |
|------|--------|----------|------|
| 新建 | `NEW` | `new Thread()` 之后，还没 `start()` | 调用 `start()` |
| 可运行 | `RUNNABLE` | 已 `start()`，等待 CPU 或正在运行 | 被调度 / 阻塞 / 结束 |
| 阻塞 | `BLOCKED` | 等待**进入** `synchronized` 临界区（锁被别人占着） | 拿到锁 |
| 无限等待 | `WAITING` | 调用 `Object.wait()`、`join()`、`LockSupport.park()` 且无超时 | 被 `notify`/`unpark` 唤醒 |
| 计时等待 | `TIMED_WAITING` | 调用 `sleep(ms)`、`wait(ms)`、`join(ms)`、`parkNanos()` | 超时或被唤醒 |
| 终止 | `TERMINATED` | `run()` 正常结束或抛异常结束 | —— |

## 4.3 状态转换图

```text
                    ┌──────────────────────────────────────────┐
                    │                                          │
   new Thread()     │  start()        抢到CPU                   │ run()结束
  ─────────────▶ NEW ──────▶ RUNNABLE ────▶ RUNNABLE ──────────▶ TERMINATED
                              ▲    │       （运行中）                 ▲
                              │    │                                │
                              │    │ 拿不到 synchronized 锁          │
                              │    ▼                                │
                              │  BLOCKED ────────────────┐          │
                              │    ▲                     │          │
                              │    │ 拿到锁              │          │
                              │    │                     │          │
                              │  ┌─┴──────────────┐      │          │
                              └──┤  WAITING       │      │          │
                                 │  TIMED_WAITING │◀─────┘          │
                                 └─┬──────────────┘                 │
                                   │  wait/notify 被唤醒 / 超时      │
                                   └────────────────────────────────┘

  WAITING       ← Object.wait() / join() / LockSupport.park()
  TIMED_WAITING ← Thread.sleep(ms) / wait(ms) / join(ms) / parkNanos()
```

> **注意**：`RUNNABLE` 在 Java 里**包含了操作系统层面的"就绪"和"运行"两种状态**（还包含阻塞在 IO 上的情况），所以 `getState()` 返回 `RUNNABLE` 不代表线程真的在占着 CPU。

# 5. 线程安全问题

## 5.1 问题从哪来

当**多个线程同时操作同一个共享资源**时，由于线程调度是抢占式的，一个线程没执行完就被切走，另一个线程读到的是"中间状态"，最终结果就不对了。

```java
// 经典问题：两个线程各卖 100 张票，结果可能卖出 200 张甚至出现负数
public class Ticket implements Runnable {
    private int count = 100;      // 共享资源

    @Override
    public void run() {
        while (true) {
            if (count <= 0) break;
            // ↓ 这三步不是原子的：读 count → 判断 → 写回 count-1
            //   线程 A 刚读到 100 就被切走，线程 B 也读到 100，最终两个线程卖的是同一张票
            System.out.println(Thread.currentThread().getName() + " 卖出第 " + count-- + " 张");
        }
    }
}
```

问题的根源是**竞态条件**——`count--` 这类操作在字节码层面由"读取→计算→写回"多条指令组成，不是**原子操作**。要解决它，就要保证这段临界区代码**同一时刻只能有一个线程执行**。

线程安全问题的三个必要条件（破坏任一即可）：

1. **多线程**环境；
2. 存在**共享资源**；
3. 对共享资源有**非原子**的读写操作。

## 5.2 加锁方式一：同步代码块

```java
synchronized (锁对象) {
    // 临界区：同一时刻只有一个持有该锁对象的线程能进来
}
```

执行规则：

1. 锁默认是打开的，一个线程进去后锁自动**关闭**；
2. 临界区代码全部执行完毕（或异常退出），线程出来，锁自动**打开**；
3. 其他线程进来前必须先拿到锁，拿不到就进入 `BLOCKED` 状态等待。

**锁对象必须唯一**，否则等于没锁。推荐两种写法：

```java
// 写法一：用字节码对象，全 JVM 唯一
synchronized (Ticket.class) {
    // ...
}

// 写法二：自定义一个静态对象（对象必须是 static 的，否则每个实例各有一把锁 = 没锁）
private static final Object LOCK = new Object();
synchronized (LOCK) {
    // ...
}
```

> **注意**：`synchronized (new Object())` 是最典型的错误——每次执行都新建一个锁对象，锁根本不生效。同理，用可变对象（如 `Integer`）做锁对象也有隐患，因为 `Integer` 装箱有缓存机制，可能造成意料之外的锁共享。

## 5.3 加锁方式二：同步方法

把 `synchronized` 加在方法声明上，等于把**方法体全部代码**作为临界区。锁对象不能自己指定，由 JVM 约定：

| 方法类型 | 锁对象 |
|----------|--------|
| 非静态方法 | `this`（当前实例对象） |
| 静态方法 | 当前类的字节码对象 `类名.class` |

```java
public synchronized void sell() {       // 锁对象是 this
    // ...
}

public static synchronized void sell2() { // 锁对象是 Ticket.class
    // ...
}
```

> **陷阱**：如果一个类的同步方法是 `this` 锁，而调用方 `new` 了两个实例，那么两个实例各有一把锁，同步方法**不互斥**。要互斥就必须用静态同步方法或显式的类锁。这也是"单例 + 同步"经常一起出现的原因。

## 5.4 加锁方式三：Lock 接口 / ReentrantLock

`synchronized` 是 JVM 层面的关键字，用起来简单但不够灵活。JDK5 引入的 `java.util.concurrent.locks.Lock` 接口把锁变成了**对象**，可以手动控制加锁解锁。

```java
private static final Lock lock = new ReentrantLock();   // 必须是共享的唯一对象

@Override
public void run() {
    while (true) {
        lock.lock();            // 加锁，拿不到就阻塞等待
        try {
            if (count <= 0) break;
            System.out.println(Thread.currentThread().getName() + " 卖出第 " + count-- + " 张");
        } finally {
            lock.unlock();      // 必须在 finally 中释放，保证异常时也能解锁
        }
    }
}
```

`synchronized` 与 `ReentrantLock` 的对比：

| 维度 | `synchronized` | `ReentrantLock` |
|------|---------------|-----------------|
| 层级 | JVM 关键字，字节码指令 `monitorenter/exit` | JDK 类，`java.util.concurrent` |
| 释放方式 | 自动（代码块结束/异常） | **手动**，必须在 `finally` 中 `unlock()` |
| 可中断 | 不支持 | `lockInterruptibly()` 支持等待时被中断 |
| 可超时 | 不支持 | `tryLock(timeout)` 支持超时放弃 |
| 公平锁 | 只有非公平 | 构造参数可选公平/非公平 |
| 条件变量 | 一个（wait/notify） | 可 `newCondition()` 出**多个**等待队列 |
| 性能 | JDK6 优化后已接近 | 高竞争下略优 |

> **行业实践**：**能用 `synchronized` 就用 `synchronized`**，因为它是 JVM 内建、写法简单、不会忘记解锁；只有在需要**超时、可中断、公平锁、多条件队列**这些高级能力时才换 `ReentrantLock`。锁的原理、升级过程与 AQS 实现，见第 30 篇。

## 5.5 死锁

**死锁**指两个或多个线程互相持有对方需要的锁，又都在等对方释放，导致全部永久阻塞。

```java
// 线程 A 先锁 objA 再锁 objB；线程 B 先锁 objB 再锁 objA —— 典型的环路等待
Object objA = new Object();
Object objB = new Object();

new Thread(() -> {
    synchronized (objA) {
        System.out.println("A 拿到 objA");
        synchronized (objB) { System.out.println("A 拿到 objB"); }
    }
}).start();

new Thread(() -> {
    synchronized (objB) {
        System.out.println("B 拿到 objB");
        synchronized (objA) { System.out.println("B 拿到 objA"); }
    }
}).start();
// 结果：两边都卡住，程序不会结束
```

死锁的四个必要条件（**同时满足**才会死锁，破坏任一条即可避免）：

| 条件 | 含义 | 破坏方式 |
|------|------|----------|
| 互斥 | 资源同一时刻只能被一个线程占用 | 锁的本质，通常无法破坏 |
| 请求并保持 | 持有一个锁的同时去请求另一个锁 | 一次性申请全部资源 |
| 不可剥夺 | 已获得的锁不能被强制抢占 | `tryLock` 超时后主动释放 |
| 循环等待 | 存在一个线程—资源的环形等待链 | **按固定顺序加锁**（最实用） |

**排查手段**：用 `jstack <pid>` 或 `jconsole`，JVM 会自动检测并打印 `Found one Java-level deadlock:` 以及涉及的线程和锁。预防上最有效的一条是**约定全局加锁顺序**，让所有线程都按 `objA → objB` 的顺序申请。

# 6. 等待唤醒机制

## 6.1 wait / notify / notifyAll

`synchronized` 解决了"互斥"，但线程之间还需要"协作"——比如生产者发现队列满了就该等，消费者取走后再通知它。这套机制由 `Object` 的三个方法提供：

| 方法 | 作用 |
|------|------|
| `void wait()` | 当前线程释放锁并进入等待，直到被唤醒 |
| `void wait(long timeout)` | 等待指定毫秒，超时自动醒 |
| `void notify()` | **随机**唤醒一个在该锁上等待的线程 |
| `void notifyAll()` | 唤醒在该锁上等待的**所有**线程（它们会重新竞争锁） |

三条必须记住的规则：

1. **必须写在 `synchronized` 同步代码块或同步方法中**，否则抛 `IllegalMonitorStateException`；
2. **必须通过锁对象调用**——唤醒的就是绑定这把锁的线程；
3. 被 `wait()` 的线程进入 `WAITING` 状态，**会释放锁**（这正是它和 `sleep()` 的关键区别）。

> **注意**：`wait/notify` 的逻辑其实是"释放锁 → 等待"和"唤醒 → 重新竞争锁"，所以被唤醒的线程不一定立刻执行，它还得再抢一次锁。这就是为什么 `wait()` 必须放在 `while` 循环里而不是 `if` 里判断条件——**防止虚假唤醒**（spurious wakeup）。

```java
// ✓ 正确：用 while 防止虚假唤醒
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    // 条件满足，继续处理
}
```

## 6.2 生产者—消费者模型

等待唤醒机制最经典的应用。核心是"**数组/队列 + 两个等待条件 + 一把锁**"：

| 角色 | 行为 |
|------|------|
| 生产者 | 队列满 → `wait()`；生产后 → `notifyAll()` |
| 消费者 | 队列空 → `wait()`；消费后 → `notifyAll()` |
| 信号位 | 用一个 `boolean flag` 或队列长度表示"是否可生产/可消费"，条件不满足就等 |

```java
// 桌面（共享缓冲区）：最多放 1 个汉堡
public class Desk {
    public static final Object LOCK = new Object();
    public static boolean flag = false;   // false=桌上空，true=桌上有
}

// 生产者
public class Cook extends Thread {
    @Override
    public void run() {
        while (true) {
            synchronized (Desk.LOCK) {
                if (Desk.flag) {
                    try { Desk.LOCK.wait(); } catch (InterruptedException e) { throw new RuntimeException(e); }
                } else {
                    System.out.println("厨师做了一个汉堡");
                    Desk.flag = true;
                    Desk.LOCK.notifyAll();   // 唤醒等待这把锁的所有线程（消费者）
                }
            }
        }
    }
}

// 消费者
public class Foodie extends Thread {
    @Override
    public void run() {
        while (true) {
            synchronized (Desk.LOCK) {
                if (!Desk.flag) {
                    try { Desk.LOCK.wait(); } catch (InterruptedException e) { throw new RuntimeException(e); }
                } else {
                    System.out.println("吃货吃掉一个汉堡");
                    Desk.flag = false;
                    Desk.LOCK.notifyAll();   // 唤醒生产者
                }
            }
        }
    }
}
```

> **行业实践**：生产环境**不要自己手写 `wait/notify` 实现生产者消费者**——一来容易写错（漏了 `while`、用了 `notify` 而非 `notifyAll`），二来性能不如并发容器。直接用第 7 节的 **阻塞队列**，它把这套逻辑封装好了。

`ReentrantLock` 也支持等待唤醒，而且能创建**多个条件队列**，让生产者只唤醒消费者、消费者只唤醒生产者（`synchronized` 只有一个等待队列，只能 `notifyAll` 全唤醒，效率更低）：

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull  = lock.newCondition();   // 生产者等待的队列
Condition notEmpty = lock.newCondition();   // 消费者等待的队列

lock.lock();
try {
    while (队列满) notFull.await();
    // 生产...
    notEmpty.signal();      // 精确唤醒消费者
} finally {
    lock.unlock();
}
```

# 7. 阻塞队列

阻塞队列（`BlockingQueue`）是**内部自带锁**的队列，继承结构为 `Iterable → Collection → Queue → BlockingQueue`。它的价值在于"取不到就等、放不下就等"这两个语义天然支持生产者消费者，不用自己写 `wait/notify`。

四种插入/移除方法的区别（**这是使用阻塞队列最容易出错的地方**）：

| 操作 | 抛异常 | 返回特殊值 | 阻塞 | 超时 |
|------|--------|-----------|------|------|
| 插入 | `add(e)` | `offer(e)` 返回 false | `put(e)` | `offer(e, time, unit)` |
| 移除 | `remove()` | `poll()` 返回 null | `take()` | `poll(time, unit)` |
| 检查 | `element()` | `peek()` 返回 null | —— | —— |

常用实现类：

| 实现类 | 底层结构 | 边界 | 特点 |
|--------|----------|------|------|
| `ArrayBlockingQueue` | 数组 | **有界**，必须指定容量 | 一把锁（JDK8 后可分离），内存紧凑 |
| `LinkedBlockingQueue` | 链表 | 可选有界，默认 `Integer.MAX_VALUE` | 双锁分离，吞吐通常更高；默认容量接近无界，容易 OOM |
| `SynchronousQueue` | 无缓冲 | 容量 0 | 每个 `put` 必须等一个 `take`，用于线程池"直接移交" |
| `PriorityBlockingQueue` | 二叉堆 | 无界 | 按优先级出队 |
| `DelayQueue` | 优先队列 | 无界 | 延迟到期才可取出，用于定时任务 |
| `LinkedTransferQueue` | 链表 | 无界 | 支持 `transfer()` 直接移交给消费者 |

> **注意**：`LinkedBlockingQueue` 的默认容量是 `Integer.MAX_VALUE`，`Executors.newFixedThreadPool` 就是用它且不传容量——这是它容易 OOM 的根源，也是**阿里规约禁止用 `Executors` 创建线程池**的原因之一（见 8.2）。

# 8. 线程池

## 8.1 为什么用线程池

线程的创建和销毁要消耗系统资源（内核态切换、栈内存分配）。如果每个任务都 `new Thread()`：

1. **开销大**——高并发下系统花在创建/销毁线程上的时间比处理业务还多；
2. **数量不可控**——瞬间涌入 10 万请求就创建 10 万个线程，直接 OOM 或把 CPU 打满；
3. **无法管理**——不能统一控制并发数、不能监控、不能优雅关闭。

线程池的核心思想是**复用**：预先创建若干线程放在池子里，任务来了就交给空闲线程，任务结束线程不销毁而是回到池中等待下一个任务。

## 8.2 Executors 快捷工厂（及其陷阱）

`Executors` 是线程池的工具类，提供了一批快捷创建方法：

| 方法 | 核心/最大线程数 | 任务队列 | 特点 |
|------|----------------|----------|------|
| `newCachedThreadPool()` | 0 / `Integer.MAX_VALUE` | `SynchronousQueue` | 来多少开多少，空闲 60s 回收；**线程数无上限** |
| `newFixedThreadPool(n)` | n / n | `LinkedBlockingQueue` | 线程数固定，空闲线程不回收；**队列无上限** |
| `newSingleThreadExecutor()` | 1 / 1 | `LinkedBlockingQueue` | 单线程串行执行，保证顺序 |
| `newScheduledThreadPool(n)` | n / `Integer.MAX_VALUE` | `DelayedWorkQueue` | 支持定时/周期任务 |

**成员方法**：

| 方法 | 说明 |
|------|------|
| `Future<?> submit(Runnable task)` | 提交无返回值的任务，返回 `Future` 用于判断是否完成 |
| `<T> Future<T> submit(Runnable task, T result)` | 提交任务并指定"成功时的返回值" |
| `<T> Future<T> submit(Callable<T> task)` | 提交有返回值的任务，`Future.get()` 取结果 |
| `void execute(Runnable command)` | 直接执行，无返回值（属于 `Executor` 接口） |
| `void shutdown()` | 优雅关闭：不再接收新任务，**等待已提交任务执行完** |
| `List<Runnable> shutdownNow()` | 立即关闭：尝试中断正在执行的任务，返回未执行的任务列表 |

```java
ExecutorService pool = Executors.newFixedThreadPool(3);
Future<Integer> future = pool.submit(() -> {
    int sum = 0;
    for (int i = 1; i <= 100; i++) sum += i;
    return sum;
});
System.out.println(future.get());   // 5050
pool.shutdown();                    // 用完必须关闭，否则线程池里的线程会阻止 JVM 退出
```

> **行业实践**：**不要用 `Executors` 创建线程池**。阿里《Java 开发手册》明确禁止，原因是：
> `newFixedThreadPool` / `newSingleThreadExecutor` 用**无界队列**，任务积压会 OOM；`newCachedThreadPool` / `newScheduledThreadPool` 的**最大线程数是 `Integer.MAX_VALUE`**，请求洪峰会创建海量线程而 OOM。
> 正确做法是直接 `new ThreadPoolExecutor(...)`，把七个参数显式写出来（见 8.3）。

## 8.3 ThreadPoolExecutor 七大参数

```java
public ThreadPoolExecutor(
    int corePoolSize,                      // 核心线程数：池中常驻的线程数量
    int maximumPoolSize,                   // 最大线程数：核心 + 临时线程的上限
    long keepAliveTime,                    // 临时线程空闲多久后被回收
    TimeUnit unit,                         // keepAliveTime 的时间单位
    BlockingQueue<Runnable> workQueue,     // 任务队列：核心线程忙时，任务先入队
    ThreadFactory threadFactory,           // 线程工厂：定制线程名、优先级、是否守护线程
    RejectedExecutionHandler handler       // 拒绝策略：队列满且线程满时的兜底
) { }
```

七个参数的含义与常见取值：

| 参数 | 作用 | 建议 |
|------|------|------|
| `corePoolSize` | 常驻线程数，即使空闲也不回收（除非开 `allowCoreThreadTimeOut`） | IO 密集型可大，CPU 密集型取核数 |
| `maximumPoolSize` | 上限，`maximumPoolSize ≥ corePoolSize` | 有界队列时才有意义 |
| `keepAliveTime` + `unit` | 超出核心数的线程空闲存活时长 | 默认对核心线程无效，除非 `allowCoreThreadTimeOut(true)` |
| `workQueue` | 排队区，**强烈建议用有界队列** | `new ArrayBlockingQueue<>(容量)` |
| `threadFactory` | 自定义线程名便于排查问题 | 用 `ThreadFactoryBuilder` 或自定义，命名如 `order-pool-1` |
| `handler` | 兜底策略，见 8.5 | 按业务容忍度选，默认 `AbortPolicy` |

## 8.4 任务提交后的执行流程

这是面试高频考点，务必记牢顺序：

```text
                提交任务 submit() / execute()
                          │
                          ▼
             ┌─── 核心线程数是否已满？ ───┐
             │否                          │是
             ▼                            ▼
      创建核心线程执行          ┌── 任务队列是否已满？ ──┐
             │                  │否                      │是
             │                  ▼                       ▼
             │          任务入队等待        ┌── 线程数是否达到 maximumPoolSize？ ──┐
             │                  │           │否                                    │是
             │                  │           ▼                                      ▼
             │                  │    创建临时线程执行                     执行拒绝策略 handler
             └──────────────────┴───────────┴──────────────────────────────────────┘
```

关键结论（**很多人的直觉是反的**）：

1. 任务优先给**核心线程**；
2. 核心线程忙了，任务进**队列**排队，**而不是先开临时线程**；
3. 只有**队列满了**，才会创建临时线程到 `maximumPoolSize`；
4. 队列满 + 线程数到上限，才触发**拒绝策略**。

> **注意**：正因为"队列满了才开临时线程"，如果用了无界队列（`LinkedBlockingQueue` 默认容量），`maximumPoolSize` 这个参数**永远不会生效**——任务无限排队直到 OOM。这是必须用有界队列的另一个原因。

## 8.5 四种拒绝策略

线程池内置的 `RejectedExecutionHandler` 实现：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `AbortPolicy`（**默认**） | 抛 `RejectedExecutionException` | 重要业务，宁可报错也不能丢 |
| `CallerRunsPolicy` | 让**提交任务的线程**自己执行 | 希望降速而不是丢任务，天然形成背压 |
| `DiscardPolicy` | 静默丢弃，不抛异常 | 允许丢的可选任务（埋点上报） |
| `DiscardOldestPolicy` | 丢弃队列中**最老**的任务，再提交当前任务 | 只关心最新数据的场景 |

自定义拒绝策略只需实现 `RejectedExecutionHandler` 接口：

```java
new RejectedExecutionHandler() {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // 例如：落库、写日志、投递到 MQ 稍后重试
        log.warn("任务被拒绝，当前队列长度 {}", executor.getQueue().size());
    }
}
```

> **行业实践**：核心业务建议用 `CallerRunsPolicy` 或自定义策略（记录 + 落盘 + 告警），`DiscardPolicy` 只适合"丢了也不影响主流程"的场景。

## 8.6 线程数怎么设

| 任务类型 | 推荐公式 | 说明 |
|----------|----------|------|
| **CPU 密集型**（加解密、复杂计算） | `CPU 核数 + 1` | 多 1 个线程用来填补偶发的页缺失等停顿 |
| **IO 密集型**（RPC、数据库、文件） | `CPU 核数 × (1 + 平均等待时间 / 平均计算时间)` | 也叫 Brian Goetz 公式，倍数常取 2~4，视 IO 占比而定 |
| **混合型** | 拆成两个线程池分别处理 | 不要混在一起，会因为 IO 拖累 CPU |

```java
int cores = Runtime.getRuntime().availableProcessors();  // 获取 CPU 逻辑核数
```

> **注意**：公式只是**起点**，真实值必须靠**压测**确定。上线后还要监控队列积压、活跃线程数、任务拒绝数（`ThreadPoolExecutor` 提供了 `getActiveCount()`、`getQueue().size()`、`getCompletedTaskCount()`），据此动态调整。Spring 环境可用 `ThreadPoolTaskExecutor` 配合 `application.yml` 做参数化配置。

# 9. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 线程创建 | 继承 `Thread` 受单继承限制；`Runnable` 解耦但无返回值；`Callable + FutureTask` 有返回值能抛异常；**生产环境都交给线程池** |
| 状态 | Java 六态：`NEW` / `RUNNABLE` / `BLOCKED` / `WAITING` / `TIMED_WAITING` / `TERMINATED`；`sleep` 不释放锁，`wait` 释放 |
| 线程安全 | 根源是"共享资源 + 非原子操作"；三招：同步代码块、同步方法、`ReentrantLock` |
| 死锁 | 四条件破坏其一即可；最实用的是**按固定顺序加锁**；用 `jstack` 排查 |
| 等待唤醒 | `wait/notify` 必须在同步块内、用锁对象调用；`wait` 要放在 `while` 里防虚假唤醒；生产环境优先用阻塞队列 |
| 阻塞队列 | 记住 `put/take`（阻塞）、`offer/poll`（返回特殊值）、`add/remove`（抛异常）三组语义；`LinkedBlockingQueue` 默认近无界 |
| 线程池 | 七大参数；执行顺序 = 核心线程 → 队列 → 临时线程 → 拒绝策略；**禁用 `Executors`，用有界的 `ThreadPoolExecutor`** |

下一篇（第 18 篇）进入网络编程，看 Java 如何用 `Socket` 建立 TCP/UDP 通信；锁的底层原理（对象头、偏向锁、轻量级锁、AQS、CAS）集中在第 30 篇。
