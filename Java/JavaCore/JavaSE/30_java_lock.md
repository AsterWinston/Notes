"锁"是 Java 并发里最深的一块，也是面试里区分"用过并发"和"懂并发"的分水岭。第 17 篇讲了锁**怎么用**（`synchronized`、`ReentrantLock`、`wait/notify`），本篇往下钻一层，回答三个问题：**锁为什么能保证线程安全（JMM 层面）**、**JDK 到底是怎么实现一把锁的（对象头、锁升级、AQS、CAS）**、**面对具体场景该选哪种锁（读写锁、StampedLock、信号量、各类并发工具）**。读完这一篇，再回头看第 17 篇的 `synchronized` 代码块，你会知道 JVM 在背后替你做了多少事。

# 1. 并发问题的根源：JMM 与三大特性

## 1.1 Java 内存模型（JMM）

要理解锁，先要理解**为什么不用锁就会出错**。答案在 Java 内存模型（Java Memory Model，JMM）里。

JMM 规定：所有变量都存在**主内存**（Main Memory）中，而每条线程还有自己的**工作内存**（Working Memory）。线程对变量的所有操作都必须在工作内存中进行，不能直接读写主内存的变量；线程间变量的传递也必须经过主内存中转。

```text
        ┌───────── 线程 A ─────────┐        ┌───────── 线程 B ─────────┐
        │  ┌────────────────────┐  │        │  ┌────────────────────┐  │
        │  │  工作内存（副本）   │  │        │  │  工作内存（副本）   │  │
        │  │  flag = false      │  │        │  │  flag = false      │  │
        │  └─────────┬──────────┘  │        │  └─────────┬──────────┘  │
        └────────────┼─────────────┘        └────────────┼─────────────┘
                     │  read / write                      │
                     ▼                                    ▼
        ┌──────────────────────────────────────────────────────────────┐
        │                        主内存（Main Memory）                  │
        │                          flag = false                        │
        └──────────────────────────────────────────────────────────────┘
```

关键点：**工作内存是"副本"**。线程 A 把 `flag` 改成 `true` 只改了自己工作内存里的副本，什么时候同步回主内存、线程 B 什么时候从主内存重新读，都是不确定的。这就是可见性问题的来源。

> **注意**：JMM 里的"工作内存"是一个**抽象概念**，并不真的对应 CPU 的一级/二级缓存或寄存器。它是对 CPU 缓存、写缓冲区、编译器指令重排等硬件与运行期优化的统一抽象。理解了这一点，才能理解后面所有的内存屏障。

## 1.2 三大特性：可见性、原子性、有序性

并发安全问题最终都能归到这三个特性上，而**锁（以及 `volatile`）本质上是用来补齐丢失的那几个特性**。

| 特性 | 含义 | 破坏它的原因 | 靠什么保证 |
|------|------|-------------|-----------|
| **原子性** | 一个或多个操作要么全部执行且不被中断，要么全部不执行 | 线程切换（时间片用完、IO 阻塞、`yield`） | `synchronized`、`Lock`、CAS |
| **可见性** | 一个线程修改了共享变量，其他线程能立即看到 | 工作内存副本、CPU 缓存、编译器优化 | `volatile`、`synchronized`、`final` |
| **有序性** | 程序执行的顺序符合代码的先后顺序 | 编译器重排、指令级并行重排、内存系统重排 | `volatile`、`synchronized`、happens-before |

逐个看一个例子：

```java
// ① 可见性问题：主线程改了 flag，子线程可能永远看不到
static boolean flag = false;

public static void main(String[] args) {
    new Thread(() -> {
        while (!flag) { }        // 子线程读的是自己工作内存里的副本，可能一直是 false
        System.out.println("子线程结束");
    }).start();

    try { Thread.sleep(1000); } catch (InterruptedException e) { }

    flag = true;                 // 主线程改了，但没同步到主内存；子线程看不到 → 死循环
}
// 把 flag 改成 volatile 就能正常退出
```

```java
// ② 原子性问题：count++ 不是一步完成的
static int count = 0;
// 线程 A、B 各执行 10000 次 count++，结果大概率小于 20000
// 因为 count++ 在字节码层面是四条指令：
//   getstatic  取值
//   iconst_1   常量 1
//   iadd       相加
//   putstatic  写回
// 线程 A 执行到 iadd 时被切走，线程 B 也读到同一个旧值，两次自增只生效一次
```

```java
// ③ 有序性问题：指令重排导致"半初始化对象"被别的线程看到
// 详见 4.4 双重检查锁
```

## 1.3 happens-before 规则

JMM 用 **happens-before（先行发生）** 这套规则来定义"哪些操作对其他线程可见"，它取代了让人头秃的"指令重排规则表"。核心思想是：

**如果 A happens-before B，那么 A 的执行结果对 B 可见，且 A 的执行顺序排在 B 之前。**

八条主要规则：

| 规则 | 内容 |
|------|------|
| **程序次序规则** | 在一个线程内，按代码顺序，前面的操作 happens-before 后面的操作 |
| **监视器锁规则** | 对一个锁的 `unlock` happens-before 于随后对这个锁的 `lock` |
| **volatile 变量规则** | 对一个 `volatile` 变量的**写** happens-before 于后续对这个变量的**读** |
| **传递性** | A hb B，B hb C ⟹ A hb C |
| **线程启动规则** | `Thread.start()` happens-before 于该线程中的每一个动作 |
| **线程终止规则** | 线程中的所有操作 happens-before 于其他线程检测到该线程已终止（`join()` 返回、`isAlive()` 为 false） |
| **线程中断规则** | `interrupt()` 调用 happens-before 于被中断线程检测到中断事件 |
| **对象终结规则** | 对象的构造函数执行结束 happens-before 于 `finalize()` 方法开始 |

> **注意**：happens-before **不等于**"时间上先发生"。它描述的是**可见性**和**合法重排的边界**——只要不违反 happens-before，JVM 和 CPU 就可以随意重排和优化。所以在单线程里观察不到的问题，在多线程里会冒出来。

# 2. 锁的分类总览

Java 里的"锁"不是一种东西，而是从不同**维度**切出来的概念。先把全貌摆出来，后面逐节展开。

| 分类维度 | 类型 | 说明 | 典型代表 |
|----------|------|------|----------|
| **加锁时机** | 悲观锁 / 乐观锁 | 先加锁再操作 / 先操作再验证冲突 | `synchronized` / CAS |
| **获取顺序** | 公平锁 / 非公平锁 | 按排队顺序 / 允许插队 | `ReentrantLock(true/false)` |
| **是否可重复获取** | 可重入锁 / 不可重入锁 | 同一线程可再次获取 / 会死锁 | `ReentrantLock` / 手写锁 |
| **同时可持有者** | 独享锁 / 共享锁 | 只能一个线程 / 可多个线程同时持有 | 写锁 / 读锁 |
| **加锁策略** | 自旋锁 / 自适应自旋锁 | 忙等 / 根据上次结果动态调整自旋次数 | CAS 循环 / 轻量级锁 |
| **JVM 实现层次** | 偏向锁 / 轻量级锁 / 重量级锁 | `synchronized` 的三级状态 | 对象头 Mark Word |
| **粒度** | 分段锁 / 非分段锁 | 分区加锁提高并发 | `ConcurrentHashMap`(JDK7) |
| **加锁对象** | 对象锁 / 类锁 | 锁实例 / 锁 Class 对象 | `synchronized(this)` / `synchronized(X.class)` |

> **注意**：同一个锁往往会同时贴好几个标签。比如 `ReentrantLock` 默认是"**非公平、可重入、独享、悲观**锁"。

# 3. synchronized 详解

`synchronized` 是 JVM 层面内建的锁，它最重要的好处是**不用手动释放**——无论代码正常结束还是抛异常，JVM 都会保证锁被释放。

## 3.1 三种用法与对应的锁对象

| 用法 | 写法 | 锁对象 | 作用范围 |
|------|------|--------|----------|
| 修饰实例方法 | `public synchronized void m() {}` | `this`（当前实例对象） | 整个方法体 |
| 修饰静态方法 | `public static synchronized void m() {}` | `类名.class`（Class 对象） | 整个方法体 |
| 修饰代码块 | `synchronized (obj) { }` | 括号里的对象 | 代码块内 |

**一张必须记住的表**：不同类型的同步方法之间是否互斥？

| 持有 | 实例方法（this 锁） | 静态方法（Class 锁） | 代码块（this 锁） |
|------|-------------------|---------------------|------------------|
| 同一实例调实例方法 | 互斥 | 不互斥 | 互斥 |
| 不同实例调实例方法 | **不互斥** | 不互斥 | 不互斥 |
| 任意实例调静态方法 | 不互斥 | 互斥 | 不互斥 |

> **陷阱**：最常见的误解是"加了 `synchronized` 就安全"。如果一个 Spring 单例 Bean 的同步方法用的是 `this` 锁，没问题；但如果这个对象被 new 了多份（比如在循环里 `new Service()`），每份都有自己的 `this` 锁，**根本不互斥**。需要全局互斥时，务必用 `synchronized (类名.class)` 或静态同步方法。

## 3.2 字节码层面的实现

`synchronized` 到底编译成了什么？`javap -c` 一看便知。

**同步代码块**编译为成对的 `monitorenter` / `monitorexit` 指令。注意编译器会生成**两个** `monitorexit`：

```text
public void method();
  ...
  monitorenter        // 获取锁：把对象头的 Mark Word 关联到 Monitor 并置位
  ...                 // 临界区代码
  monitorexit         // 正常路径：释放锁
  goto 20
  astore_2
  aload_1
  monitorexit         // 异常路径：也必须释放锁，这就是"自动解锁"的来源
  aload_2
  athrow
```

**同步方法**则没有这两条指令，而是在方法的访问标志里加一个 `ACC_SYNCHRONIZED`：

```text
public synchronized void method();
  flags: ACC_PUBLIC, ACC_SYNCHRONIZED
```

方法调用时，JVM 检查这个标志，自动在进入前获取锁、退出时释放。

## 3.3 对象头与 Mark Word

每个 Java 对象在内存里由三部分组成：

```text
┌───────────────────────────────────────────────────────────┐
│                       Java 对象内存布局                    │
├────────────────────┬──────────────────────────────────────┤
│   对象头 Header     │  实例数据 Instance Data（各字段）     │ ... 对齐填充
│  ┌──────┬────────┐ │                                      │
│  │ Mark │ Klass  │ │                                      │
│  │ Word │ Pointer│ │                                      │
│  └──────┴────────┘ │                                      │
└────────────────────┴──────────────────────────────────────┘
   32/64 bit  32/64 bit
```

- **Mark Word**：默认存对象的 `hashCode`、GC 分代年龄、锁状态标志等，**会随锁状态变化而复用同一块空间**（这就是"锁升级"的物理基础）。
- **Klass Pointer**：指向方法区中该对象的类型元数据。
- 数组对象还会多 4 字节记录数组长度。

64 位 JVM 下 Mark Word 在不同锁状态下的内容（**这张表是理解锁升级的关键**）：

| 锁状态 | 25/54 bit | 31/62 bit | 4 bit | 1 bit | 2 bit |
|--------|-----------|-----------|-------|-------|-------|
| 无锁 | unused | `hashCode`(31) | 分代年龄 | 0 | **01** |
| 偏向锁 | 线程 ID（54） + Epoch（2） | unused | 分代年龄 | **1** | **01** |
| 轻量级锁 | 指向栈中锁记录的指针（62） | | | | **00** |
| 重量级锁 | 指向 Monitor 的指针（62） | | | | **10** |
| GC 标记 | 空 | | | | **11** |

> **注意**：为什么无锁状态的最后两位是 `01`？因为要保证"至少有一位是 1"，所以锁标志位只有 `00`（轻量级）、`10`（重量级）、`11`（GC 标记）三种非 `01` 状态。

## 3.4 监视器 Monitor 与重量级锁

**Monitor（管程）**是重量级锁的核心。在 HotSpot 里由 C++ 的 `ObjectMonitor` 实现，主要字段有：

| 字段 | 含义 |
|------|------|
| `_owner` | 指向持有该 Monitor 的线程 |
| `_recursions` | **重入次数**（这就是"可重入"的实现） |
| `_EntryList` | 存放等待获取锁而被阻塞的线程（`BLOCKED`） |
| `_WaitSet` | 存放调用 `wait()` 后等待被唤醒的线程（`WAITING`） |
| `_count` | 重入计数 |

获取锁的过程是：线程进入 `_EntryList` → 拿到锁后 `_owner` 指向自己、`_count + 1` → 执行完毕 `_count - 1`，减到 0 时释放锁并唤醒 `_EntryList` 里的线程。

> **注意**：Monitor 底层的同步最终要依赖操作系统的 **mutex（互斥量）**，涉及**用户态到内核态的切换**，开销可达纳秒到微秒级。这正是"重量级"的由来，也是 JVM 要在它之上做偏向锁和轻量级锁优化的原因。

## 3.5 锁升级全过程

JDK 6 之前 `synchronized` 直接就是重量级锁，性能很差；JDK 6 引入了**锁升级**，让它在无竞争或低竞争时几乎零成本。

```text
        ┌──────────┐  第一个线程访问同步块
        │  无锁     │──────────────────────────▶
        └──────────┘
                                    ┌──────────────────────┐
                                    │ 偏向锁                │
                                    │ Mark Word 记录线程 ID │
                                    │ 同一线程再进入：零开销 │
                                    └──────────┬───────────┘
                                               │ 有另一个线程来竞争
                                               ▼
                                    ┌──────────────────────┐
                                    │ 轻量级锁              │
                                    │ 栈中锁记录 + CAS 自旋  │
                                    └──────────┬───────────┘
                                               │ 自旋超过阈值 / 竞争加剧
                                               ▼
                                    ┌──────────────────────┐
                                    │ 重量级锁              │
                                    │ Monitor + 操作系统 mutex│
                                    │ 竞争线程直接阻塞       │
                                    └──────────────────────┘
```

**① 偏向锁（Biased Locking）**：大多数情况下，锁不仅没有多线程竞争，而且总是由**同一个线程**多次获得。于是干脆在 Mark Word 里记下这个线程 ID，下次该线程再进来时，检查一下 ID 相同就直接通过——连 CAS 都不用做。

- 撤销偏向锁需要**等到全局安全点（Safe Point）**，成本较高；
- 如果对象的 `hashCode()` 被调用过，偏向锁会失效（因为 Mark Word 的位置被 `hashCode` 占了）—— 这也是"调用了 `hashCode` 之后偏向锁就升级为轻量级锁"的原因。

> **行业实践**：偏向锁由于撤销成本高、在现代高并发场景下收益有限，**JDK 15 起默认被禁用（JEP 374），JDK 18 已彻底废弃**。所以现在谈偏向锁主要是为了理解历史设计和面试原理，新版本 JVM 上直接看轻量级锁即可。

**② 轻量级锁**：当有第二个线程来竞争时，偏向锁升级为轻量级锁。线程会在**自己的栈帧中**开辟一块"锁记录（Lock Record）"空间，把对象头的 Mark Word 复制进去，然后尝试用 **CAS** 把对象头的 Mark Word 替换成指向这个锁记录的指针：

- CAS 成功 → 拿到锁；
- CAS 失败 → 说明有竞争，先**自旋**重试若干次，仍然失败就升级为重量级锁。

**③ 重量级锁**：竞争激烈时，自旋反而白耗 CPU，干脆让竞争线程进入 `_EntryList` 阻塞，靠操作系统调度唤醒。

**锁升级不可逆**：只会从低到高，不会降级（因为降级收益低、实现复杂）。

## 3.6 锁优化：锁消除、锁粗化、自适应自旋

除了锁升级，JIT 编译器还会做三种优化：

**① 锁消除（Lock Elimination）**：基于**逃逸分析**，如果发现某段代码里的锁对象不可能被其他线程访问到，就直接把这个锁去掉。

```java
public String concat(String s1, String s2) {
    // StringBuffer 是线程安全的，append 都带 synchronized
    // 但这里的 sb 是局部变量，不可能"逃逸"出去被别的线程访问
    // JIT 会消除掉这些锁 —— 所以"用 StringBuffer 一定比 StringBuilder 慢"是错的
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb.toString();
}
```

**② 锁粗化（Lock Coarsening）**：如果在一段代码里反复对同一个对象加锁解锁，JIT 会把锁的范围**扩大**到整个操作序列之外，避免频繁的加解锁开销。

```java
// 优化前：循环里 1000 次加锁解锁
for (int i = 0; i < 1000; i++) {
    synchronized (lock) {
        doSomething(i);
    }
}
// 优化后（JIT 粗化）：只在循环外加一次锁
```

**③ 自适应自旋（Adaptive Spinning）**：自旋的次数不是固定的，而是根据**同一个锁上上一次自旋的成功率**动态调整——上次自旋很快就拿到了锁，这次就多转几圈；上次转了半天也没拿到，这次就少转甚至直接阻塞。

## 3.7 synchronized 的特性小结

| 特性 | 是否具备 |
|------|---------|
| 保证原子性 | ✓ |
| 保证可见性 | ✓（解锁前把工作内存刷回主内存，加锁时清空工作内存从主内存重读） |
| 保证有序性 | ✓（通过 happens-before 的监视器锁规则） |
| 可重入 | ✓（靠 Monitor 的 `_recursions`） |
| 可中断 | ✗ |
| 可超时 | ✗ |
| 公平性 | 非公平（但 JVM 不保证不会饥饿） |

# 4. volatile 关键字

`volatile` 比 `synchronized` 轻——它**只保证可见性和有序性，不保证原子性**。

## 4.1 两大语义

**① 可见性**：对一个 `volatile` 变量的写，会立即刷新到主内存；对这个变量的读，会从主内存重新读取，而不是用工作内存里的副本。

**② 有序性**：禁止指令重排序。具体说，JVM 会在 `volatile` 读写前后**插入内存屏障**，保证：

- `volatile` 写之前的操作，不会被重排到写之后；
- `volatile` 读之后的操作，不会被重排到读之前。

> **注意**：这条"禁止重排"的边界是**单向**的，看下面的 JMM 规范更容易记：
>
> | 第一个操作 | 第二个操作 | 能否重排 |
> |-----------|-----------|---------|
> | 普通读/写 | `volatile` 读 | 能 |
> | 普通读/写 | `volatile` 写 | **不能** |
> | `volatile` 读 | 普通读/写 | **不能** |
> | `volatile` 写 | 普通读/写 | 能 |
> | `volatile` 写 | `volatile` 读 | **不能** |

## 4.2 底层实现：内存屏障

JVM 在 `volatile` 变量的读写处插入四类内存屏障：

| 屏障类型 | 作用 |
|----------|------|
| `LoadLoad` | 保证 `Load1` 先于 `Load2` 完成（读读有序） |
| `StoreStore` | 保证 `Store1` 先于 `Store2` 刷回主内存（写写有序） |
| `LoadStore` | 保证 `Load1` 先于 `Store2` 完成（读写有序） |
| `StoreLoad` | 保证 `Store1` 先于 `Load2` 完成（写读有序，**开销最大**） |

字节码层面，`volatile` 体现为字段的 `ACC_VOLATILE` 访问标志。在 x86 平台上，`volatile` 写最终被编译成带 **`lock` 前缀**的指令——`lock` 前缀会锁住总线（现代 CPU 用缓存一致性协议 MESI 优化为锁缓存行），并把写缓冲区的内容刷回主内存，同时让其他 CPU 里对应的缓存行失效，从而被其他线程"看见"。

## 4.3 volatile 不保证原子性

这是最容易被误解的一点。`volatile` 能让 `i++` 的每一步都可见，但**不能阻止线程切换发生在中间**。

```java
public class VolatileTest {
    private static volatile int count = 0;   // 加了 volatile

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000; j++) count++;   // 仍然不是原子的
            }).start();
        }
        Thread.sleep(2000);
        System.out.println(count);   // 期望 10000，实际大概率小于 10000
    }
}
```

要让 `i++` 原子化，只有两条路：加锁，或者用原子类（见第 5 节）。

> **最佳实践**：`volatile` 真正适用的场景只有两个——**① 作为状态标志位**（`volatile boolean shutdown`），**② 双重检查锁单例中修饰 `instance`**。凡是涉及"读—改—写"的复合操作，`volatile` 都无能为力。

## 4.4 典型应用：双重检查锁单例

这是 `volatile` 最经典也是最能说明"有序性"问题的场景。

```java
public class Singleton {
    // 注意：少了 volatile，这个单例就是错的
    private static volatile Singleton instance;

    private Singleton() { }

    public static Singleton getInstance() {
        if (instance == null) {                     // 第一重检查：避免每次都加锁
            synchronized (Singleton.class) {        // 加锁
                if (instance == null) {             // 第二重检查：防止多个线程重复创建
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

问题出在 `instance = new Singleton()` 这一行。它在 CPU 层面由三步组成：

```text
① 分配内存空间                     memory = allocate()
② 在这块内存上初始化对象             ctorInstance(memory)
③ 把引用赋给 instance               instance = memory
```

② 和 ③ **没有数据依赖，可以重排**。如果重排成 ①③②：

```text
线程 A                              线程 B
────────────────────────────        ────────────────────────────
① 分配内存
③ instance = memory  ← 这时 instance 已经 != null
                                    if (instance == null) → false
                                    直接 return instance（但对象还没初始化！）
                                    后续使用 → NullPointerException / 拿到错误数据
② 初始化对象
```

`volatile` 的作用就是**禁止 ② 和 ③ 重排**，保证看到非 null 引用时对象一定已经初始化完毕。

> **最佳实践**：单例还有更简单的写法——**静态内部类**（利用类加载的线程安全）和**枚举单例**（《Effective Java》推荐，天然防反射和反序列化攻击）。DCL 主要用来讲原理，工程上优先选后两种。

## 4.5 volatile 与 synchronized 对比

| 维度 | `volatile` | `synchronized` |
|------|-----------|----------------|
| 原子性 | ✗ | ✓ |
| 可见性 | ✓ | ✓ |
| 有序性 | ✓ | ✓ |
| 是否阻塞 | 不阻塞 | 会阻塞 |
| 性能 | 轻量（内存屏障） | 竞争时较重 |
| 适用范围 | 变量 | 代码块 / 方法 |
| 可重入 | 无关 | ✓ |

# 5. CAS 与原子类

## 5.1 CAS 原理

**CAS（Compare And Swap）**是无锁并发的基石。它包含三个操作数：

- **V**：要更新的变量当前值（内存地址）
- **E**：期望值（Expected）
- **N**：新值（New）

只有 **V == E** 时，才把 V 更新为 N；否则什么都不做，但会返回当前的真实值。整个"比较并交换"由 **CPU 的一条指令原子完成**（x86 上是 `cmpxchg`，配合 `lock` 前缀保证多核下的原子性），中间不会被中断。

```java
// AtomicInteger 的加一，本质是一个 CAS 自旋循环
public final int incrementAndGet() {
    for (;;) {                                     // 自旋
        int current = get();                       // 读取当前值
        int next = current + 1;
        if (compareAndSet(current, next))          // CAS：只有当前还是 current 才更新
            return next;                           // 成功则返回
        // 失败说明被别人改过，重新读一遍再试
    }
}
```

JDK 里 CAS 的入口是 `sun.misc.Unsafe` 的 `compareAndSwapInt` / `compareAndSwapObject` 等本地方法。

## 5.2 CAS 的三大问题

**① ABA 问题**

线程 1 读到值 A，线程 2 把 A 改成 B 又改回 A，线程 1 的 CAS 仍然成功——**它不知道值中途变过**。

大多数场景下 ABA 无害（比如单纯计数），但如果关心"中间是否被改过"，就会出错。经典反例是**用 CAS 实现的无锁栈**：栈顶 A → B → C，线程 1 读到栈顶 A 和下一个节点 B，准备 `CAS(top, B)`；此时线程 2 把 A 和 B 都弹出，又压入一个新的 A；线程 1 的 CAS 成功，但栈结构已经错了。

解决办法是加**版本号**：

```java
// 每次修改版本号 +1，CAS 时同时比较"值 + 版本号"
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 0);

int stamp = ref.getStamp();                 // 取当前版本
ref.compareAndSet(100, 200, stamp, stamp + 1);
// 即使值被改回 100，版本号也变了，CAS 会失败

// 只关心"是否被改过"而不关心改了几次，可以用
AtomicMarkableReference<Integer> markRef;   // 版本退化为一个 boolean 标记
```

**② 循环时间长开销大**

CAS 失败就重试，高竞争下大量线程反复空转（自旋），白白消耗 CPU。这也是 `LongAdder` 出现的原因（见 5.4）。

**③ 只能保证一个共享变量的原子操作**

多个变量无法用一次 CAS 搞定。解决办法是把多个变量**封装成一个对象**，用 `AtomicReference` 对整体做 CAS。

## 5.3 原子类家族

`java.util.concurrent.atomic` 包按用途分成五类：

| 类别 | 类 | 说明 |
|------|-----|------|
| **基本类型** | `AtomicInteger`、`AtomicLong`、`AtomicBoolean` | 对 int/long/boolean 做原子操作 |
| **数组** | `AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray` | 原子更新数组里的某个元素 |
| **引用类型** | `AtomicReference`、`AtomicStampedReference`、`AtomicMarkableReference` | 原子更新对象引用（后两个解决 ABA） |
| **字段更新器** | `AtomicIntegerFieldUpdater`、`AtomicLongFieldUpdater`、`AtomicReferenceFieldUpdater` | 原子更新**某个对象的某个 volatile 字段**，无需把整个类改造成原子类 |
| **累加器（JDK8+）** | `LongAdder`、`DoubleAdder`、`LongAccumulator`、`DoubleAccumulator` | 高并发计数场景，性能远超 `AtomicLong` |

`AtomicIntegerFieldUpdater` 的典型用法——注意**字段必须是 `volatile` 且不能用 `private`**：

```java
public class Account {
    public volatile int balance;      // 必须是 volatile，且访问权限要够
}

AtomicIntegerFieldUpdater<Account> updater =
        AtomicIntegerFieldUpdater.newUpdater(Account.class, "balance");
updater.incrementAndGet(account);
```

## 5.4 LongAdder 与分段累加

`AtomicLong` 在**高并发计数**下性能很差，因为所有线程都在 CAS 同一个 `value`，冲突率高、自旋重试多。

`LongAdder` 的思路是**分散热点**：

```text
                    ┌──────────────────────────────────┐
                    │            base                  │  无竞争时直接用 base
                    └──────────────────────────────────┘
                                    │ 有竞争时
                                    ▼
        ┌───────────┬───────────┬───────────┬───────────┐
        │  Cell[0]  │  Cell[1]  │  Cell[2]  │  Cell[3]  │  每个线程 CAS 自己的 Cell
        └───────────┴───────────┴───────────┴───────────┘
                                    │
                        sum() = base + Σ Cell[i]
```

- 维护一个 `base` 变量和一个 **Cell 数组**；
- 无竞争时直接 CAS `base`；
- 出现竞争时，把不同线程的累加**分散到不同的 Cell** 上，各自 CAS 各自的，冲突大幅降低；
- 求和时把所有 Cell 和 base 加起来。

> **注意**：`LongAdder` 的 `sum()` **不是强一致的**——求和的瞬间可能有其他线程正在写 Cell，所以它只保证"最终一致"。因此 `LongAdder` 适合**统计计数**（QPS、埋点），不适合要求精确的读（那还是得用 `AtomicLong`）。

# 6. AQS 框架

**AQS（AbstractQueuedSynchronizer）**是 `java.util.concurrent` 包的骨架。`ReentrantLock`、`ReentrantReadWriteLock`、`Semaphore`、`CountDownLatch`、`ThreadPoolExecutor` 的 Worker 全都基于它。理解了 AQS，上层的这些类就都是"换汤不换药"。

## 6.1 设计思想：模板方法 + state + 队列

AQS 用了一个非常漂亮的设计——**模板方法模式**：

```text
┌──────────────────────────────────────────────────────────────┐
│                     AQS（父类，实现通用逻辑）                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 通用：阻塞、排队、唤醒、中断处理                          │  │
│  │       acquire()  release()  acquireShared() ...         │  │
│  └──────────────────────┬─────────────────────────────────┘  │
│                         │ 调用（回调）                        │
│  ┌──────────────────────▼─────────────────────────────────┐  │
│  │ 抽象：tryAcquire()  tryRelease()  tryAcquireShared()    │  │
│  │       tryReleaseShared()  isHeldExclusively()           │  │
│  │       ← 由子类实现，决定"什么叫获取成功"                  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ▲ 继承
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ReentrantLock        Semaphore           CountDownLatch
   （state=重入次数）    （state=许可数）      （state=剩余计数）
```

AQS 管理的核心资源就是一个 `volatile int state`：

| 同步器 | `state` 的语义 |
|--------|---------------|
| `ReentrantLock` | 锁的重入次数（0 = 未持有） |
| `ReentrantReadWriteLock` | **高 16 位 = 读锁持有数，低 16 位 = 写锁重入数** |
| `Semaphore` | 剩余许可数 |
| `CountDownLatch` | 剩余待完成的计数 |

## 6.2 核心数据结构

```java
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;            // 同步状态
    private transient volatile Node head;  // 队头（哨兵节点）
    private transient volatile Node tail;  // 队尾

    static final class Node {
        static final int CANCELLED =  1;   // 线程已取消
        static final int SIGNAL    = -1;   // 后继节点需要被唤醒
        static final int CONDITION = -2;   // 在 Condition 队列中等待
        static final int PROPAGATE = -3;   // 共享模式下需要向后传播
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
    }
}
```

等待队列是一个 **CLH 队列的变体**——一个双向链表，`head` 是一个不存线程的**哨兵节点**：

```text
   head                                              tail
    │                                                 │
    ▼                                                 ▼
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│ 哨兵   │◀──▶│ Node1  │◀──▶│ Node2  │◀──▶│ Node3  │
│(无线程)│    │ 线程B  │    │ 线程C  │    │ 线程D  │
└────────┘    └────────┘    └────────┘    └────────┘
   waitStatus    SIGNAL        SIGNAL       (自旋中)
```

`waitStatus` 最常用的两个值：`SIGNAL(-1)` 表示"当前节点释放锁后要唤醒下一个节点"，`CANCELLED(1)` 表示该节点已放弃等待。

## 6.3 独占模式：acquire 与 release

**获取锁 `acquire(int arg)`** 的流程：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg)                       // ① 先尝试直接获取（子类实现）
        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // ② 失败则入队并等待
        selfInterrupt();                       // ③ 等待中被打断过，补上中断标记
}
```

拆开看：

```text
acquire()
   │
   ├─▶ tryAcquire(arg)
   │      ├─ 成功 → 直接返回，不排队（这就是"非公平"的来源）
   │      └─ 失败 ↓
   │
   ├─▶ addWaiter(Node.EXCLUSIVE)
   │      └─ 用 CAS 把当前线程包装成 Node，挂到队尾
   │
   └─▶ acquireQueued(node)
          └─ 自旋：
               ① 如果前驱是 head → 再试一次 tryAcquire()
               ② 失败 → 把前驱的 waitStatus 置为 SIGNAL → LockSupport.park() 挂起
               ③ 被唤醒后重复 ①
         拿到锁后把当前节点设为 head，并把原 head 断开（方便 GC）
```

**释放锁 `release(int arg)`**：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {              // 子类决定是否真正释放完毕
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);         // 唤醒队列中的下一个节点
        return true;
    }
    return false;
}
```

> **注意**：可重入锁的 `tryRelease` 里，`state` 减到 **0** 才算真正释放，此时才唤醒后继节点——这就是"可重入"在 AQS 层面的实现。

## 6.4 共享模式

共享模式（`acquireShared` / `releaseShared`）允许多个线程同时持有，用于 `Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock` 的读锁。

与独占模式的区别：

- `tryAcquireShared` 返回值是 `int`：负数表示失败，`0` 表示成功但后续线程无法再获取，正数表示成功且后续还能获取；
- 多了一个 `PROPAGATE` 状态和 `doReleaseShared` 的**传播唤醒**逻辑，保证被唤醒的线程能继续唤醒后面的线程（否则共享模式下可能有线程永远醒不来）。

## 6.5 条件队列 ConditionObject

`Condition` 是 AQS 的**内部类**，用来实现 `synchronized` 里 `wait/notify` 的等价功能，但更灵活——一把锁可以创建**多个条件队列**：

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull  = lock.newCondition();   // 生产者等这个
Condition notEmpty = lock.newCondition();   // 消费者等这个
// 精确唤醒，避免 synchronized 只能 notifyAll 引发的"惊群"
```

每个 `ConditionObject` 维护一个**单向的条件队列**。线程 `await()` 时会：

```text
await() 流程：
  ① 把当前节点从同步队列（AQS 队列）移到条件队列
  ② 释放全部锁（state 清零），并唤醒同步队列中的后继
  ③ 挂起（park）等待被 signal
  ④ 被 signal 后，把节点从条件队列移回同步队列
  ⑤ 重新竞争锁，成功后 await() 返回

signal() 流程：
  ① 检查当前线程是否持有锁（否则抛 IllegalMonitorStateException）
  ② 把条件队列的首节点移到同步队列
  ③ 不立刻唤醒，等调用方释放锁后再唤醒
```

## 6.6 基于 AQS 的组件一览

| 组件 | 模式 | `state` 含义 | 用途 |
|------|------|-------------|------|
| `ReentrantLock` | 独占 | 重入次数 | 互斥 |
| `ReentrantReadWriteLock.ReadLock` | 共享 | 高 16 位 | 读读并发 |
| `ReentrantReadWriteLock.WriteLock` | 独占 | 低 16 位 | 读写/写写互斥 |
| `Semaphore` | 共享 | 剩余许可数 | 限流 |
| `CountDownLatch` | 共享 | 剩余计数 | 等待 N 个任务完成 |
| `ThreadPoolExecutor.Worker` | 独占 | 是否被占用 | 线程池 worker 中断控制 |

# 7. ReentrantLock 详解

## 7.1 可重入的实现

`ReentrantLock` 内部有个 `Sync` 抽象类继承 AQS，`NonfairSync` 和 `FairSync` 是它的两个实现。核心逻辑在 `tryAcquire`：

```java
// 非公平锁的 tryAcquire（简化）
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {    // 无锁状态，CAS 抢占
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {   // ← 重入的关键
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);                                // 只是把 state 累加
        return true;
    }
    return false;
}
```

**可重入的本质**：判断"当前线程是不是锁的持有者"，是的话直接把 `state` 加一，而不是重新排队。释放时 `state` 减一，减到 0 才真正释放。

```java
// 可重入演示：同一个线程可以反复拿同一把锁
reentrantLock.lock();     // state: 0 → 1
try {
    reentrantLock.lock(); // state: 1 → 2（不会死锁）
    try {
        System.out.println(reentrantLock.getHoldCount());  // 输出 2
    } finally {
        reentrantLock.unlock();  // state: 2 → 1
    }
} finally {
    reentrantLock.unlock();      // state: 1 → 0，此时才真正释放
}
```

> **陷阱**：`lock()` 和 `unlock()` **必须成对**。加了几次就要解几次，少解锁不释放（其他线程永久阻塞），多解抛 `IllegalMonitorStateException`。

## 7.2 公平锁与非公平锁

| 维度 | 公平锁 `new ReentrantLock(true)` | 非公平锁 `new ReentrantLock()`（默认） |
|------|--------------------------------|--------------------------------------|
| 获取策略 | 严格按入队顺序，新来的线程要先看队列里有没有人在等 | 新来的线程直接 CAS 抢占，抢不到才排队 |
| 实现差异 | `tryAcquire` 里多一步 `hasQueuedPredecessors()` 检查 | 无此检查，直接 CAS |
| 吞吐量 | 较低（线程切换频繁） | **较高**（减少了唤醒开销） |
| 饥饿 | 不会 | 理论上可能（但 JVM 不保证公平锁一定公平） |

```java
// 公平锁的 tryAcquire 关键差异
if (c == 0) {
    if (!hasQueuedPredecessors() &&          // 关键差异：队列里还有人等着就不抢
        compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
```

> **设计考量**：为什么默认非公平？因为"唤醒一个排队线程"需要从 park 状态恢复，涉及**用户态/内核态切换**，成本远高于直接 CAS 抢一次。如果被唤醒的线程还在恢复的路上，锁就被新来的线程抢走了，那这次唤醒就白费了。非公平锁允许这种"插队"，换来更高的吞吐。

## 7.3 可中断与超时获取

这是 `ReentrantLock` 相比 `synchronized` 最有价值的能力——`synchronized` 一旦阻塞就**无法中断、无法超时**，可能永久卡死。

```java
Lock lock = new ReentrantLock();

// ① 可中断：等待锁的过程中可以被 interrupt() 打断
try {
    lock.lockInterruptibly();     // 阻塞在此处时，其他线程调用该线程的 interrupt() 会抛 InterruptedException
} catch (InterruptedException e) {
    // 处理中断：放弃获取锁，做清理
} finally {
    if (lock.isHeldByCurrentThread()) lock.unlock();
}

// ② 可超时：等不到就放弃，避免无限期阻塞
try {
    if (lock.tryLock(3, TimeUnit.SECONDS)) {    // 最多等 3 秒
        try {
            // 业务逻辑
        } finally {
            lock.unlock();
        }
    } else {
        // 等超时了，走降级逻辑
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();         // 恢复中断标记
}

// ③ 立即返回：拿不到就立刻返回 false，不等待
if (lock.tryLock()) { ... }
```

> **行业实践**：在分布式或调用链较深的系统里，**"超时"是防止故障扩散的关键**。用 `tryLock(timeout)` 替代无限等待，能让线程快速失败、快速释放，避免一个慢操作拖垮整个线程池。这也是为什么高并发场景下 `ReentrantLock` 往往比 `synchronized` 更受青睐。

## 7.4 Condition 精确唤醒

`synchronized` 只有一个等待队列，`notifyAll()` 会把所有等待者都唤醒（**惊群效应**），被唤醒的线程还得挨个判断条件、大多数又回去等。

`ReentrantLock` 可以 `newCondition()` 出多个队列，实现**精确唤醒**：

```java
// 有界阻塞队列的简化实现
class BoundedQueue<T> {
    private final Object[] items;
    private int count, putIndex, takeIndex;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();   // 队列满时，生产者在此等待
    private final Condition notEmpty = lock.newCondition();   // 队列空时，消费者在此等待

    public void put(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();              // 满了：生产者等 notFull
            }
            items[putIndex] = t;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            notEmpty.signal();                // 只唤醒消费者，不惊动其他生产者
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();             // 空了：消费者等 notEmpty
            }
            T t = (T) items[takeIndex];
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            notFull.signal();                 // 只唤醒生产者
            return t;
        } finally {
            lock.unlock();
        }
    }
}
```

## 7.5 ReentrantLock vs synchronized 的选择

| 场景 | 推荐 |
|------|------|
| 一般互斥，代码简单 | `synchronized`（写法简单，不会忘记解锁） |
| 需要超时/可中断/公平锁 | `ReentrantLock` |
| 需要多个条件队列精确唤醒 | `ReentrantLock` + `Condition` |
| 读多写少 | `ReentrantReadWriteLock` 或 `StampedLock` |
| 高并发计数 | `LongAdder`（不是锁） |

> **最佳实践**：优先 `synchronized`，它由 JVM 直接优化（锁消除、锁粗化、自适应自旋），写错的可能性更低。只有在确实需要 `ReentrantLock` 的**高级特性**时才替换——不要因为"听说 ReentrantLock 性能好"就无脑换。

# 8. 读写锁与 StampedLock

## 8.1 ReentrantReadWriteLock 的设计

普通的互斥锁太"严格"了——**读读之间其实没有冲突**，却也被串行化了。读写锁把锁拆成两把：

| 锁 | 共享性 | 互斥关系 |
|----|--------|---------|
| 读锁（共享锁） | 多个线程可**同时**持有 | 与写锁互斥 |
| 写锁（独占锁） | 只允许一个线程持有 | 与读锁、写锁都互斥 |

```text
           读锁    写锁
  读锁      ✓      ✗
  写锁      ✗      ✗
```

适用场景：**读多写少**。比如缓存、配置中心——读很频繁，写很少。

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock  = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// 读操作：允许多个线程并发
readLock.lock();
try {
    return cache.get(key);
} finally {
    readLock.unlock();
}

// 写操作：独占
writeLock.lock();
try {
    cache.put(key, value);
} finally {
    writeLock.unlock();
}
```

## 8.2 state 的高低位拆分

`ReentrantReadWriteLock` 只有一个 `state`，却要同时记录读锁和写锁的状态，靠的是**按位拆分**：

```text
                  int state（32 位）
  ┌────────────────────────┬────────────────────────┐
  │  高 16 位 = 读锁持有数   │  低 16 位 = 写锁重入数  │
  └────────────────────────┴────────────────────────┘
              SHARED_SHIFT = 16
```

```java
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);   // 1 << 16
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;  // 65535

// 获取读锁数量：state 右移 16 位
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
// 获取写锁重入次数：state 与 0xFFFF 按位与
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

> **注意**：因为只有 16 位，读写锁的**最大重入次数都是 65535**，超过会抛 `Error`。另外读锁是**共享的但可重入**——多个线程可以同时持有，而同一个线程也可以多次获取读锁。

## 8.3 锁降级

读写锁支持**写锁降级为读锁**，但**不支持读锁升级为写锁**。

**锁降级**指：持有写锁 → 获取读锁 → 释放写锁。这是合法的，而且有实际意义：

```java
// 锁降级的典型场景：保证"修改后立刻读到自己改的值"，且不让中间被人插队
writeLock.lock();
try {
    // ① 修改数据
    data = newValue;

    // ② 在释放写锁之前先拿到读锁（降级）
    readLock.lock();
} finally {
    writeLock.unlock();      // ③ 释放写锁，此时仍持有读锁
}

try {
    // ④ 这里读到的数据一定是自己刚写的，且其他写线程进不来
    use(data);
} finally {
    readLock.unlock();
}
```

如果不做降级，`writeLock.unlock()` 和后面的读之间会有一个窗口，别的线程可能插进来修改数据，导致读到的不是自己写的那份（这在"缓存更新 + 保证一致性"的场景里是致命的）。

> **注意**：**读锁不能升级为写锁**。如果线程持有读锁后去申请写锁，会因为写锁需要等待所有读锁释放、而自己又不释放读锁，导致**永久死锁**。`ReentrantReadWriteLock` 直接不允许这种情况（写锁的 `tryAcquire` 检测到已有读锁会失败）。

## 8.4 写饥饿与 StampedLock 的乐观读

`ReentrantReadWriteLock` 有一个缺点：**写锁可能被饿死**。因为读锁默认是非公平的，读请求源源不断时，写请求可能迟迟排不上队。

而且"读多写少"的场景里，即使读锁能并发，**每次读仍要写一次 `state`（CAS）**，还是有开销。

JDK 8 引入的 **`StampedLock`** 用**乐观读**解决了这个问题：

```java
StampedLock lock = new StampedLock();

// ① 乐观读：不加锁！只拿一个"版本戳"
long stamp = lock.tryOptimisticRead();
// ② 读数据到局部变量
int x = this.x;
int y = this.y;
// ③ 校验：读的过程中有没有写操作发生？
if (!lock.validate(stamp)) {
    // ④ 有写操作 → 升级为悲观读锁，重读一遍
    stamp = lock.readLock();
    try {
        x = this.x;
        y = this.y;
    } finally {
        lock.unlockRead(stamp);
    }
}
```

写操作则用 `writeLock()` / `unlockWrite(stamp)`。

`StampedLock` 的三种模式对比：

| 模式 | 方法 | 特点 |
|------|------|------|
| 悲观读 | `readLock()` | 类似 `ReentrantReadWriteLock` 的读锁，会阻塞写 |
| 乐观读 | `tryOptimisticRead()` | **不加锁**，只返回版本戳，读完 `validate` 校验 |
| 写 | `writeLock()` | 独占 |

> **陷阱**：`StampedLock` **不支持重入**，**不支持 `Condition`**，而且 `stamp` 用完必须正确释放对应的那一个（别把读的 stamp 拿去 `unlockWrite`）。它比读写锁快，但**用起来更危险**，只在"读极多、写极少、且确实需要极致性能"时才考虑。

# 9. 并发工具类

这些类不是锁，但都基于 AQS，常与锁一起出现，这里一并说清。

## 9.1 Semaphore（信号量）

**控制同时访问某资源的线程数量**——限流。

```java
// 停车场只有 3 个车位
Semaphore semaphore = new Semaphore(3);   // 公平模式下传 true

// 每辆车进入
semaphore.acquire();       // 获取许可，没有许可就阻塞（相当于 P 操作）
try {
    // 停车（访问共享资源）
} finally {
    semaphore.release();   // 释放许可（相当于 V 操作）
}
```

- `acquire(n)`：获取 n 个许可，不够就阻塞；
- `release(n)`：归还 n 个许可；
- `tryAcquire(timeout)`：超时获取；
- `availablePermits()`：当前剩余许可数。

> **行业实践**：`Semaphore` 最典型的用途是**接口限流**和**数据库连接池**。注意它**不保证线程安全地访问资源本身**——它只是"发牌子"，资源本身的互斥还得自己保证。

## 9.2 CountDownLatch（倒计时门闩）

**一个线程等待其他 N 个线程完成**。`state` 就是剩余计数。

```java
// 场景：主线程等 5 个子任务都跑完，再做汇总
CountDownLatch latch = new CountDownLatch(5);

for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        try {
            // 干活
        } finally {
            latch.countDown();     // 计数减一
        }
    }).start();
}

latch.await();                     // 主线程阻塞，直到计数归零
System.out.println("全部完成，开始汇总");
```

> **注意**：`CountDownLatch` 是**一次性的**——计数归零后就不能重置，要重用只能用 `CyclicBarrier` 或新建一个。

## 9.3 CyclicBarrier（循环栅栏）

**一组线程互相等待，直到都到达栅栏点，再一起继续**。与 `CountDownLatch` 的关键区别是"**可循环**"。

```java
// 场景：5 个运动员都准备好后，同时起跑
CyclicBarrier barrier = new CyclicBarrier(5, () ->
    System.out.println("所有人都到了，发令枪响！"));   // 第二个参数是"到齐后执行的栅栏动作"

for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        System.out.println(Thread.currentThread().getName() + " 已就位");
        try {
            barrier.await();           // 等待其他线程
            System.out.println(Thread.currentThread().getName() + " 起跑");
        } catch (Exception e) { }
    }).start();
}
```

`CountDownLatch` 与 `CyclicBarrier` 的对比：

| 维度 | `CountDownLatch` | `CyclicBarrier` |
|------|-----------------|----------------|
| 等待方向 | 一等多（主线程等子线程） | 相互等（线程之间） |
| 是否可重用 | ✗ 一次性 | ✓ 计数到 0 后自动重置 |
| 计数方式 | `countDown()` 主动减 | `await()` 到齐自动减 |
| 底层实现 | AQS 共享模式 | `ReentrantLock` + `Condition` |

## 9.4 Phaser（阶段器）

`CyclicBarrier` 的升级版，支持**动态增减参与者**、**多阶段**同步。`state` 里同时记录阶段号和参与者数（用位运算拆分）。

```java
Phaser phaser = new Phaser(3);       // 初始 3 个参与者

// 参与者可以随时 register() 加入、arriveAndDeregister() 退出
phaser.arriveAndAwaitAdvance();      // 到达并等待其他参与者
```

适合"分批处理、每批参与者数量不固定"的场景。

## 9.5 Exchanger（交换器）

**两个线程之间交换数据**的同步点。

```java
Exchanger<String> exchanger = new Exchanger<>();

// 线程 A
String dataFromB = exchanger.exchange("A 的数据");   // 阻塞，直到 B 也来交换
// 线程 B
String dataFromA = exchanger.exchange("B 的数据");
```

适用场景：遗传算法、流水线式的双缓冲、校对工作。

## 9.6 工具类对比与选择

| 工具类 | 一句话用途 | 底层 |
|--------|-----------|------|
| `Semaphore` | 限制并发数量（限流） | AQS 共享模式 |
| `CountDownLatch` | 主线程等多个子任务完成 | AQS 共享模式 |
| `CyclicBarrier` | 一组线程互相等待，可循环 | `ReentrantLock` + `Condition` |
| `Phaser` | 多阶段、参与者可动态变化 | 自研（`state` 位运算） |
| `Exchanger` | 两线程交换数据 | `LockSupport` + `CAS` |

# 10. ThreadLocal

`ThreadLocal` 严格说**不是锁**，但它解决的是"多线程共享数据"这个同类问题——**用空间换隔离**，让每个线程有自己独立的变量副本，从根上避开竞争，连锁都不需要。

## 10.1 原理

```text
        ┌──────────────┐          ┌──────────────────────────────┐
        │  Thread A    │─────────▶│ ThreadLocalMap               │
        │              │          │  ┌────────────┬──────────┐   │
        │ threadLocals │          │  │ Entry(key) │  value   │   │
        └──────────────┘          │  ├────────────┼──────────┤   │
                                  │  │ TL 对象(弱) │ "A的值"   │   │
        ┌──────────────┐          │  └────────────┴──────────┘   │
        │  Thread B    │─────────▶└──────────────────────────────┘
        │ threadLocals │          ┌──────────────────────────────┐
        └──────────────┘          │ ThreadLocalMap               │
                                  │  ┌────────────┬──────────┐   │
                                  │  │ TL 对象(弱) │ "B的值"   │   │
                                  │  └────────────┴──────────┘   │
                                  └──────────────────────────────┘
```

关键点：**`ThreadLocalMap` 是挂在 `Thread` 对象上的**（字段 `threadLocals`），不是挂在 `ThreadLocal` 上。`ThreadLocal` 只是一个"钥匙"——用自己作为 key 去当前线程的 Map 里取/存值。

```java
ThreadLocal<String> tl = new ThreadLocal<>();

// 线程 A
tl.set("A 的值");
System.out.println(tl.get());   // A 的值

// 线程 B
tl.set("B 的值");
System.out.println(tl.get());   // B 的值，两个线程互不干扰
tl.remove();                    // 用完务必清理
```

## 10.2 内存泄漏

`ThreadLocalMap` 的 `Entry` 继承自 `WeakReference<ThreadLocal<?>>`——**key 是弱引用，value 是强引用**。

```text
   Entry
   ┌──────────────────────────────┐
   │  key   ──▶ ThreadLocal 对象   │  弱引用：没有外部强引用时会被 GC 回收
   │  value ──▶ 用户数据            │  强引用：一直存在，直到线程结束
   └──────────────────────────────┘
```

内存泄漏的发生过程：

1. `ThreadLocal` 的强引用被置为 null（比如方法结束）；
2. 由于 key 是弱引用，GC 时这个 key 被回收，`Entry` 变成 `(null, value)`；
3. 但 **value 还被 `Entry` 强引用着**，而 `Entry` 又被 `Thread` 的 `ThreadLocalMap` 引用着；
4. 如果这个线程是**线程池里的常驻线程**（生命周期极长），这个 value 就永远回收不掉 → **内存泄漏**。

`ThreadLocalMap` 在 `set`/`get`/`remove` 时会顺带清理 key 为 null 的 Entry（探测式清理），但这**不保证一定能清到**。

> **最佳实践**：**用完必须 `remove()`**，在 `try...finally` 里调用：
> ```java
> try {
>     threadLocal.set(value);
>     // 业务逻辑
> } finally {
>     threadLocal.remove();   // 在线程池环境下这是必须的
> }
> ```
> 不 `remove` 除了内存泄漏，还会导致**下一个复用该线程的任务读到上一个任务的脏数据**——这个 bug 比内存泄漏更难查。

## 10.3 与锁的关系

| 思路 | 手段 | 代价 |
|------|------|------|
| 共享数据 + 加锁 | `synchronized` / `Lock` | 时间换空间：串行化，有等待开销 |
| 数据隔离 | `ThreadLocal` | 空间换时间：每线程一份，无竞争 |

典型的 `ThreadLocal` 应用：Spring 的 `TransactionSynchronizationManager`（保存当前线程的事务上下文）、`RequestContextHolder`（保存当前请求）、`SimpleDateFormat` 的线程安全改造（每个线程一个实例）。

# 11. 锁的概念辨析（高频）

这一节把所有容易混淆的"锁"名词成对摆开，面试里问到的几乎都在这里。

## 11.1 乐观锁 vs 悲观锁

| 维度 | 悲观锁 | 乐观锁 |
|------|--------|--------|
| 核心思想 | 认为一定会冲突，先加锁再操作 | 认为冲突很少，先操作，提交时再验证 |
| 实现 | `synchronized`、`ReentrantLock`、数据库 `for update` | CAS、版本号、`AtomicStampedReference` |
| 适用场景 | 写多读少、冲突激烈 | 读多写少、冲突少 |
| 缺点 | 加锁开销，可能阻塞 | 冲突时失败重试，重试成本高 |

> **注意**：数据库层面的"乐观锁"通常指在表里加一个 `version` 字段，更新时 `UPDATE ... SET version = version + 1 WHERE id = ? AND version = ?`，靠影响行数判断是否成功——这跟 Java 的 CAS 是一个思路。

## 11.2 公平锁 vs 非公平锁

- **公平锁**：按申请顺序排队，先到先得，不会饥饿，但吞吐低（频繁唤醒、上下文切换）。
- **非公平锁**：允许插队，新来的线程直接抢，抢不到才排队，吞吐高但理论上可能饥饿。

Java 里只有 `ReentrantLock`/`ReentrantReadWriteLock` 可以通过构造参数选择公平性，`synchronized` **永远是非公平的**。

## 11.3 可重入锁 vs 不可重入锁

- **可重入锁**：同一线程可以多次获取同一把锁，`state` 累加，减到 0 才释放。`ReentrantLock`、`synchronized` 都是。
- **不可重入锁**：同一线程第二次获取会死锁。`StampedLock` 就是不可重入的。

可重入的意义：避免"递归调用同步方法"或"同步方法相互调用"时把自己锁死。

## 11.4 独享锁 vs 共享锁

- **独享锁（排他锁）**：一次只能一个线程持有，如写锁、`ReentrantLock`。
- **共享锁**：可被多个线程同时持有，如读锁、`Semaphore`。

两者是**互斥关系**：一个线程持有共享锁时，其他线程可以拿共享锁，但拿不了独享锁，反之亦然。

## 11.5 自旋锁 vs 自适应自旋锁

- **自旋锁**：拿不到锁时不阻塞，而是循环重试（忙等），避免线程切换开销。适合锁持有时间极短的场景。
- **自适应自旋锁**：自旋次数不再固定，根据**同一个锁上一次自旋的成功率**动态调整。

> **注意**：自旋的前提是"临界区很短"。如果锁持有时间长，自旋就是纯粹烧 CPU，此时不如直接阻塞。

## 11.6 分段锁

把一把大锁拆成多把小锁，各管一段数据，不同段之间互不影响，从而提升并发度。

`ConcurrentHashMap`（JDK 7）就是典型：默认 16 个 `Segment`，每个 `Segment` 是一把 `ReentrantLock`，只锁自己那段桶。JDK 8 之后改成了 **CAS + `synchronized` 锁单个桶**，粒度更细（连 Segment 都没有了）。

## 11.7 死锁、活锁与饥饿

这三个都是"线程拿不到资源"，但形态不同：

| 现象 | 特征 | 例子 |
|------|------|------|
| **死锁** | 线程互相持有对方需要的锁，全部永久阻塞，状态不再变化 | A 持锁 1 等锁 2，B 持锁 2 等锁 1 |
| **活锁** | 线程没阻塞，一直在重试，但始终无法推进 | 两个人在窄路上互相让路，同时左移同时右移 |
| **饥饿** | 某个线程长期拿不到资源 | 非公平锁下低优先级线程一直被插队 |

死锁的四个必要条件（第 17 篇已展开）——**互斥、请求并保持、不可剥夺、循环等待**，破坏任意一条即可避免。最实用的手段是**按固定顺序加锁**（破坏循环等待）。

```bash
# 排查死锁：jstack 会自动检测并打印
jstack <pid>
# 输出中会看到：
# Found one Java-level deadlock:
# "Thread-1":  waiting to lock monitor 0x... (object 0x...)
# "Thread-0":  waiting to lock monitor 0x... (object 0x...)
```

活锁的典型场景是"重试时没有随机退避"——解决办法是**加入随机等待时间**。饥饿则可以通过使用公平锁缓解。

# 12. 锁的最佳实践

**选型：**

| 场景 | 首选 |
|------|------|
| 一般互斥 | `synchronized` |
| 需要超时/可中断/公平 | `ReentrantLock` |
| 读多写少 | `ReentrantReadWriteLock`；读远多于写且追求极致性能用 `StampedLock` |
| 高并发计数 | `LongAdder` |
| 限流 | `Semaphore` |
| 简单状态标志 / DCL 单例 | `volatile` |
| 线程内数据隔离 | `ThreadLocal` |

**编码规范：**

1. **锁的范围尽量小**——只在真正操作共享数据的语句上加锁，不要包住耗时操作（IO、远程调用）。
2. **`ReentrantLock` 的 `unlock()` 必须放在 `finally`**，且 `lock()` 要写在 `try` 之外，避免"加锁失败还去解锁"。
   ```java
   lock.lock();            // 写在 try 外面
   try {
       // 业务
   } finally {
       lock.unlock();      // 无条件释放
   }
   ```
3. **不要用 `String` 常量、`Integer` 缓存值、`Boolean` 这类可能被共享的对象做锁对象**。
4. **`ThreadLocal` 用完必须 `remove()`**，尤其在线程池里。
5. **加锁顺序全局统一**，从设计上消灭循环等待。
6. **能用无锁（CAS/原子类/并发容器）就不用锁**，能减小锁粒度就减小。
7. **不要在锁内调用外部方法**（可能被重写、可能阻塞、可能回调回来造成死锁）。
8. **`volatile` 只用来做标志位**，凡是"读—改—写"都要用原子类或锁。

> **行业实践**：真实的线上问题里，**死锁和高并发锁竞争**是最常见的两类。前者靠"统一加锁顺序"预防、靠 `jstack` 定位；后者靠"减小锁粒度 + 无锁替代 + 压测调优"。任何时候都不要在不确定场景下"顺手加个 `synchronized`"——锁的代价往往比你想象的大。

# 13. 小结

| 主题 | 一句话要点 |
|------|-----------|
| 并发根源 | JMM 里每线程有工作内存副本，导致可见性、原子性、有序性三类问题；happens-before 定义了合法重排的边界 |
| `synchronized` | 编译为 `monitorenter/exit`；锁信息存在对象头 Mark Word；经历无锁 → 偏向锁 → 轻量级锁 → 重量级锁的升级，只升不降 |
| `volatile` | 只保证可见性和有序性，**不保证原子性**；底层是内存屏障；两个典型场景：状态标志位、DCL 单例 |
| CAS | CPU 一条指令原子完成；三大问题：ABA（用 `AtomicStampedReference` 解决）、自旋开销、只能保证一个变量 |
| `LongAdder` | 分段累加分散热点，`sum()` 非强一致，适合统计不适合精确读 |
| AQS | `volatile state` + CLH 变体队列 + 模板方法；上层组件只需实现 `tryAcquire`/`tryRelease` 决定"什么叫获取成功" |
| `ReentrantLock` | 靠 `state` 累加 + owner 判断实现可重入；公平/非公平的差异在于是否检查 `hasQueuedPredecessors()` |
| 读写锁 | 读读共享、读写写写互斥；`state` 高低 16 位拆分；支持**锁降级**，不支持升级 |
| `StampedLock` | 乐观读不加锁，读完 `validate` 校验；性能更好但不可重入、无 `Condition`，用起来更危险 |
| 并发工具 | `Semaphore` 限流、`CountDownLatch` 一等多、`CyclicBarrier` 相互等且可循环、`Phaser` 多阶段、`Exchanger` 交换数据 |
| `ThreadLocal` | 空间换隔离；`ThreadLocalMap` 挂在 `Thread` 上，key 弱引用 value 强引用 → **必须 `remove()`** |
| 概念辨析 | 悲观/乐观、公平/非公平、可重入/不可重入、独享/共享、自旋/自适应自旋、分段锁；死锁/活锁/饥饿形态不同 |
| 最佳实践 | 缩小锁范围、`unlock` 放 `finally`、统一加锁顺序、能用无锁就不用锁 |

---

至此 JavaSE 部分接近尾声。本篇讲的是**锁的机制与选型**，往下还有两条延伸路线：

- **JUC 并发工具包的源码级剖析**（`ConcurrentHashMap` 的分段锁到 CAS + `synchronized` 的演进、`CopyOnWriteArrayList` 写时复制、`BlockingQueue` 家族、`CompletableFuture`、`ForkJoin`）—— 见 `Java/Interview/31_java_concurrency.md` 与 `Java/Interview/30_java_collections.md`；
- **JVM 层面的锁优化**（偏向锁的撤销时机、锁消除与逃逸分析在编译期的实现、`hs_err_pid` 里的锁信息）—— 见 `JavaCore/JVM/19_lock_optimization.md`。

线程池的使用主线在第 17 篇，本节的 AQS 是理解 `ReentrantLock`、`Semaphore`、`CountDownLatch` 三者共同底层的总钥匙，建议对照着读。
