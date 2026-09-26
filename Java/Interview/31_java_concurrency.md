# 1. 知识点
- **线程6种状态**：NEW（新建未start）、RUNNABLE（运行中/就绪）、BLOCKED（等待同步锁）、WAITING（无限期等待notify/notifyAll/LockSupport.unpark）、TIMED_WAITING（限时等待sleep/join/wait(timeout)）、TERMINATED（终止）
- **状态转换**：NEW → start() → RUNNABLE；获取锁成功RUNNABLE→RUNNABLE，获取锁失败RUNNABLE→BLOCKED，释放锁BLOCKED→RUNNABLE；wait()/join()/park() → WAITING，notify()/notifyAll()/unpark() → RUNNABLE；sleep(time)/wait(time)/join(time)/parkNanos → TIMED_WAITING，超时或唤醒 → RUNNABLE；run完成 → TERMINATED
- **线程创建方式**：继承Thread重写run、实现Runnable传入Thread、实现Callable+FutureTask（有返回值可抛异常）、线程池Executors提交；start()启动新线程，run()只是普通方法调用
- **start vs run**：start()由JVM调用系统调用创建新线程，在新线程中执行run()；直接调run()只是当前线程的普通方法调用，不开启新线程
- **synchronized**：JVM内置锁，可修饰方法（对象锁/类锁）和代码块；基于Monitor对象（ObjectMonitor），依赖操作系统Mutex Lock
- **对象头MarkWord**：HotSpot对象头包含MarkWord（哈希码/GC分代年龄/锁状态标志/偏向线程ID/偏向时间戳等），用2bit标识4种锁状态：01无锁/01偏向锁/00轻量级锁/10重量级锁/11GC标记
- **锁升级**（JDK6+）：无锁 → 偏向锁（CAS将MarkWord设为当前线程ID，仅首次用CAS）→ 轻量级锁（有竞争时偏向锁撤销，线程栈创建LockRecord，CAS将MarkWord设为LockRecord指针，自旋）→ 重量级锁（自旋一定次数后膨胀，MarkWord设为ObjectMonitor指针，阻塞等待）
- **锁消除**：JIT编译器分析，如果锁对象不可能逃逸出当前线程，则消除锁操作
- **锁粗化**：JIT编译器将多个连续的加锁/解锁操作合并为一次范围更大的锁操作
- **自适应自旋**：自旋等待时间由前一次在同一个锁上的自旋时间和锁的拥有者状态决定，若之前成功则增加自旋次数
- **volatile可见性**：写volatile变量后立即刷新到主内存，读volatile变量前从主内存加载最新值；底层通过lock前缀指令实现（触发MESI缓存一致性协议，通知其他CPU缓存行失效）
- **volatile有序性**：禁止指令重排序，插入内存屏障：写前StoreStore、写后StoreLoad、读前LoadLoad、读后LoadStore；JMM规定volatile写happens-before后续的volatile读
- **volatile不保证原子性**：i++等复合操作（读-改-写）不是原子的，volatile只能保证每次读都是最新值，不能保证中间不被其他线程修改
- **DCL单例为何需要volatile**：双重检查锁定单例中，`instance = new Singleton()`这行代码可能被重排序为：①分配内存 ②赋值给instance ③初始化对象。若②在③之前，另一个线程读到非null的instance返回未初始化完成的对象，造成问题。volatile禁止②在③之前的重排序。
- **CAS(Compare And Swap)**：硬件原语（cmpxchg指令），比较内存值V和预期值A，相等则替换为新值B，返回true；否则不操作，返回false。整个过程原子不可中断。
- **Unsafe类**：sun.misc.Unsafe，提供CAS操作（compareAndSwapInt/Object/Long）、内存操作、线程调度（park/unpark）。JDK9+被VarHandle替代
- **ABA问题**：CAS过程中，值A被改为B又改回A，CAS检测不出变化，可能产生隐患。解决：AtomicStampedReference（加版本号/时间戳）或AtomicMarkableReference（布尔标记）
- **AQS(AbstractQueuedSynchronizer)**：JUC核心框架，基于FIFO的CLH变体队列（双向链表），volatile int state表示同步状态，Node节点存储线程；提供独占（acquire/release，如ReentrantLock）和共享（acquireShared/releaseShared，如Semaphore/CountDownLatch）两种模式
- **AQS acquire流程**：tryAcquire尝试获取 → 失败则addWaiter(Node.EXCLUSIVE)将当前线程封装为Node入队 → acquireQueued自旋，前驱是head尝试tryAcquire，失败则park；共享模式类似，成功获取后会setHeadAndPropagate传播唤醒后续节点
- **ReentrantLock**：可重入独占锁。state=0无锁，state>0被持有。非公平锁lock先CAS抢一次再走AQS队列；公平锁先判断队列中是否有等待节点hasQueuedPredecessors()再CAS。可重入：同一线程重复加锁state++，解锁state--直到0释放锁。可中断：lockInterruptibly()。条件变量Condition：lock.newCondition()，await/signal
- **ReentrantLock vs synchronized**：ReentrantLock可中断、可超时tryLock(time, unit)、可公平锁、多Condition、需手动释放；synchronized JVM层面自动释放、代码简洁、JVM优化好（锁升级）、语义清晰。性能上JDK6后差不多，推荐优先synchronized，有高级需求再用ReentrantLock
- **ReentrantReadWriteLock**：读写锁。读锁共享（多个线程可同时持有读锁），写锁独占。写锁和读锁互斥。写锁可降级为读锁（先获取读锁再释放写锁），读锁不可升级为写锁（防止死锁）。有写锁饥饿问题（读线程过多导致写线程一直获取不到锁），JDK8 StampedLock解决
- **CountDownLatch**：减法计数器。构造指定count，countDown()减1，await()阻塞直到count=0。不可重置，一次性的。典型场景：主线程等待多个子线程完成
- **CyclicBarrier**：加法屏障。构造指定parties和可选Runnable回调。await()阻塞直到指定数量的线程到达，然后执行回调（由最后一个到达的线程执行），所有线程继续。可重复使用（计数器可重置reset()）
- **Semaphore**：信号量，基于AQS共享模式。构造指定permits许可数。acquire()获取许可（许可-1，不足阻塞），release()释放许可（许可+1，唤醒等待线程）。用于限流：限制同时访问某资源的线程数
- **ThreadLocal**：线程本地变量，每个线程有独立的变量副本。Thread内部维护ThreadLocalMap（key是ThreadLocal弱引用，value是变量值），ThreadLocal的get/set/remove操作都是操作当前线程的ThreadLocalMap
- **ThreadLocalMap**：类似HashMap，但解决哈希冲突使用开放地址法（线性探测，找下一个空位），默认初始容量16，扩容阈值2/3
- **ThreadLocal内存泄漏**：ThreadLocalMap的Entry key是弱引用（WeakReference<ThreadLocal>），value是强引用。当ThreadLocal对象没有外部强引用被GC时，key为null但value依然被Thread → ThreadLocalMap → Entry → value强引用链引用，无法回收。如果线程不结束（线程池），value一直存在导致内存泄漏。解决：使用完必须调用remove()
- **InheritableThreadLocal**：子线程可从父线程继承变量值。Thread构造时调用init()会把父线程的inheritableThreadLocals浅拷贝给子线程
- **线程池7参数**：corePoolSize核心线程数、maximumPoolSize最大线程数、keepAliveTime非核心线程空闲存活时间、unit时间单位、workQueue阻塞队列、threadFactory线程工厂、handler拒绝策略
- **线程池执行流程**：提交任务 → 当前线程数 < corePoolSize → 创建核心线程执行；核心线程满了 → 放入工作队列；队列满了 → 当前线程数 < maximumPoolSize → 创建非核心线程执行；最大线程满了 → 执行拒绝策略
- **拒绝策略4种**：AbortPolicy抛异常（默认）、CallerRunsPolicy调用者线程执行（提交速度降低）、DiscardPolicy直接丢弃不通知、DiscardOldestPolicy丢弃队列最老的再重试提交；可实现RejectedExecutionHandler自定义
- **常用线程池**：newFixedThreadPool(nThreads)，core=max，LinkedBlockingQueue无界（风险：OOM）；newCachedThreadPool，core=0，max=Integer.MAX_VALUE，SynchronousQueue，60s超时（风险：无限创建线程OOM）；newSingleThreadExecutor，core=max=1；newScheduledThreadPool，支持定时/周期任务；ForkJoinPool，工作窃取
- **为什么不推荐Executors创建线程池**：newFixedThreadPool和newSingleThreadExecutor使用无界队列LinkedBlockingQueue，可能堆积OOM；newCachedThreadPool最大线程数量无限，可能创建过多线程OOM。推荐使用ThreadPoolExecutor手动指定参数
- **CompletableFuture**：JDK8异步编排工具，实现Future和CompletionStage。runAsync无返回值，supplyAsync有返回值。链式调用：thenApply（转换，有参数有返回）、thenAccept（消费，有参数无返回）、thenRun（无参数无返回）、thenCompose（扁平化组合）、thenCombine（合并两个）。多任务：allOf等待所有完成、anyOf等待任一完成。异常处理：exceptionally、handle、whenComplete
- **ForkJoin**：JDK7引入，分治思想的线程池。RecursiveTask有返回值，RecursiveAction无返回值。工作窃取算法：每个工作线程维护一个双端队列，任务先进后出（LIFO），空闲线程从其他忙碌线程的队列尾部窃取任务（FIFO），减少竞争提高CPU利用率
- **死锁**：两个或多个线程互相持有对方需要的锁导致永久阻塞。四条件：互斥（资源不能共享）、占有且等待（持有资源并等待新资源）、不可剥夺（已获得的资源不能强制剥夺）、循环等待（形成资源请求环）。排查：jps查进程ID -> jstack PID查看线程信息，搜索"deadlock"
- **线程间通信**：wait()/notify()/notifyAll()（synchronized块内使用，属于Object，释放锁）；lock.newCondition().await()/signal()/signalAll()；LockSupport.park()/unpark(Thread)；CountDownLatch/CyclicBarrier/Semaphore；BlockingQueue.put/take
- **wait vs sleep**：wait是Object方法，释放锁，在同步块中使用，通过notify唤醒；sleep是Thread静态方法，不释放锁，可在任意地方使用，超时自动唤醒
- **虚拟线程(Java21 Loom)**：JVM管理的轻量级线程，由JVM调度到OS线程上运行（M:N调度），创建成本极低（与创建Java对象相当），不池化。Thread.ofVirtual().start()或Executors.newVirtualThreadPerTaskExecutor()。阻塞操作（IO/sleep）时自动卸载让出OS线程。与传统线程相比，可支持百万级并发。

# 2. 面试题

## 2.1 线程的生命周期和状态转换？
Java线程6种状态（Thread.State枚举）：
- **NEW**：new Thread()但未调用start()
- **RUNNABLE**：调用start()后，JVM创建底层线程开始执行。包含'就绪'和'运行中'（OS的running/ready统一为RUNNABLE）
- **BLOCKED**：等待进入synchronized方法/代码块（等待获取Monitor锁）
- **WAITING**：wait(), join(), park()，无限期等待其他线程唤醒
- **TIMED_WAITING**：sleep(time), wait(time), join(time), parkNanos(time)，超时或唤醒后回到RUNNABLE
- **TERMINATED**：run()方法执行完毕或异常终止

转换流程：NEW → (start()) RUNNABLE → (等待synchronized锁失败) BLOCKED → (获取到锁) RUNNABLE → (wait/join/park) WAITING → (notify/notifyAll/unpark) RUNNABLE → (sleep/wait(time)) TIMED_WAITING → (超时/唤醒) RUNNABLE → (执行完毕) TERMINATED

## 2.2 synchronized的锁升级过程？
JDK6后synchronized通过对象头MarkWord的低2位标识锁状态，实现4级锁升级（只能升级不能降级）：
1. **无锁**：01，无竞争时状态
2. **偏向锁**：01，MarkWord记录线程ID（54位）。当一个线程反复获取同一个锁，只需比较线程ID，无需CAS。有竞争时撤销偏向（Revoke Bias，需全局安全点），升级为轻量级锁
3. **轻量级锁**：00，线程在栈帧中创建LockRecord，CAS将MarkWord替换为LockRecord指针。自旋等待（自适应自旋）。自旋超过一定次数（默认与CPU核数相关）或等待线程超过1个，升级为重量级锁
4. **重量级锁**：10，MarkWord指向ObjectMonitor指针。ObjectMonitor包含EntryList（竞争队列）和WaitSet（wait队列）。未持有锁的线程被阻塞（内核态挂起），由JVM线程调度器控制唤醒

触发升级条件：多个线程竞争、自旋超时、调用wait/notify（需要WaitSet）

## 2.3 volatile的作用？为什么DCL单例需要volatile？
volatile三大作用：
1. **保证可见性**：写操作立即刷新到主内存，读操作始终从主内存读取（基于MESI缓存一致性协议）
2. **禁止指令重排序**：插入内存屏障（LoadStore/StoreStore/LoadLoad/StoreLoad）
3. **不保证原子性**：复合操作（如i++）不是原子的

DCL单例需要volatile：
```java
public class Singleton {
    private static volatile Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {                 // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {         // 第二次检查
                    instance = new Singleton(); // ①分配内存 ②初始化 ③赋值
                }
            }
        }
        return instance;
    }
}
```
`instance = new Singleton()`编译后分3步：分配内存、初始化对象（调用构造）、将引用赋值给instance。JIT可能重排序为：分配内存、赋值引用、初始化对象。若线程A执行到赋值引用但未初始化时，线程B通过第一次检查读到非null的instance，使用后会得到未初始化的对象。volatile禁止此重排序。

## 2.4 CAS原理和ABA问题及解决方案？
CAS（Compare And Swap）是CPU级别的原子指令（如x86的CMPXCHG）。过程：读取内存当前值V → 比较V和预期值A → 相等则将V更新为B，否则不更新。

Java中通过Unsafe类（JDK9+VarHandle）提供CAS方法：compareAndSwapInt/compareAndSwapObject/compareAndSwapLong。

ABA问题：值从A变为B再变回A，CAS检测不到，误判为未变化。如：链表操作中，节点A被删除又重新插入，CAS认为头节点没变，但实际上链表结构已变。

解决方案：
- AtomicStampedReference：内部维护Pair(引用+stamp版本号)，每次更新stamp+1，CAS时同时检查引用和版本号
- AtomicMarkableReference：维护布尔mark，适用于"只关心是否被动过"的场景（如是否被删除）

## 2.5 AQS的原理是什么？
AQS（AbstractQueuedSynchronizer）是实现锁和同步器的核心框架，核心要素：
- **state**：volatile int，表示同步状态。ReentrantLock中=0无锁>0有锁；Semaphore中=permits数量
- **CLH变体队列**：FIFO双向链表，Node节点封装线程、等待状态waitStatus（CANCELLED/SIGNAL/CONDITION/PROPAGATE）、模式（独占/共享）
- **需要子类实现的方法**：tryAcquire、tryRelease（独占）；tryAcquireShared、tryReleaseShared（共享）；isHeldExclusively

独占模式acquire流程：
```java
if (!tryAcquire(arg))
    acquireQueued(addWaiter(Node.EXCLUSIVE), arg);
// addWaiter: 创建Node入队尾（CAS循环）
// acquireQueued: 自旋，前驱是head时tryAcquire，失败park等待唤醒
```

共享模式acquireShared：成功获取后setHeadAndPropagate，可能继续唤醒后续共享节点（传播）。

## 2.6 ReentrantLock的实现原理？公平锁和非公平锁的区别？
ReentrantLock基于AQS独占模式实现，内部类Sync继承AQS，FairSync和NonfairSync继承Sync。

**非公平锁lock流程**：
1. CAS尝试将state从0设为1（直接抢一次，不管队列）
2. 成功则设置exclusiveOwnerThread=当前线程，返回
3. 失败则调用acquire(1)，acquire中tryAcquire检查state==0则CAS（仍抢一次），state>0且当前线程是持有者则state+1（重入），否则加入AQS队列等待

**公平锁lock流程**：
1. 直接调用acquire(1)
2. tryAcquire中先调用hasQueuedPredecessors()检查AQS队列中是否有等待节点（判断head.next != null且不是当前线程），有等待者则不CAS，乖乖入队
3. 无等待者才CAS抢锁

**可重入原理**：state>0表示锁已被持有。加锁时如果当前线程==exclusiveOwnerThread，state+1。解锁时state-1，state==0时释放锁，设置exclusiveOwnerThread=null。

**Condition**：AQS内部类ConditionObject实现。await()释放锁并加入Condition等待队列park；signal()将Condition队列中第一个节点移到AQS同步队列中等待获取锁。

## 2.7 CountDownLatch和CyclicBarrier的区别？
| 特性 | CountDownLatch | CyclicBarrier |
|------|---------------|---------------|
| 原理 | 减法计数器 | 加法计数器 |
| 使用次数 | 一次性，不可重置 | 可重用(reset()) |
| 阻塞方法 | await() | await() |
| 触发方法 | countDown()（计数-1） | await()（计数+1） |
| 回调线程 | 无 | 可设置Runnable，最后一个到达线程执行 |
| 典型场景 | 主线程等所有子线程完成 | 多个线程互相等待，一起开始 |
| 异常处理 | 某线程异常不通知 | 某线程异常可调用reset()或broken |

CountDownLatch如运动会：所有运动员（线程）就位，发令员释放latch后所有运动员开始跑。CyclicBarrier如团建：所有人到齐barrier.await()后一起出发。

## 2.8 ThreadLocal的原理和内存泄漏问题？
**原理**：每个Thread对象内部有一个ThreadLocalMap类型的threadLocals字段。ThreadLocal的get/set操作都会先获取当前线程的ThreadLocalMap，然后以ThreadLocal自身为key操作Map。

get()流程：获取当前线程 → 获取ThreadLocalMap → 若map==null则setInitialValue() → 否则map.get(this)获取Entry → 返回value。

**内存泄漏原因**：ThreadLocalMap的Entry继承了WeakReference<ThreadLocal>，即key是弱引用。当ThreadLocal对象没有外部强引用时，GC会回收ThreadLocal，但Entry中的value是强引用，线程 → ThreadLocalMap → Entry → value这条引用链使得value无法被回收。若线程是线程池中的线程（长期存活），则value永远无法回收，造成内存泄漏。

**解决方案**：
1. 在使用完ThreadLocal后显式调用remove()（最推荐）
2. ThreadLocal设计时内部做了清理：get/set/remove时会清理key为null的Entry（过期条目，expungeStaleEntry），但不能完全依赖
3. 将ThreadLocal声明为static final，避免被GC回收但也不能乱创建

## 2.9 线程池的核心参数和执行流程？
**7大核心参数**：
```java
public ThreadPoolExecutor(
    int corePoolSize,        // 核心线程数（常驻线程）
    int maximumPoolSize,     // 最大线程数
    long keepAliveTime,      // 非核心线程空闲存活时间
    TimeUnit unit,           // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,        // 线程工厂（设置线程名/daemon等）
    RejectedExecutionHandler handler    // 拒绝策略
)
```

**执行流程**：
1. 当前线程数 < corePoolSize → 创建新核心线程执行任务（addWorker）
2. 当前线程数 >= corePoolSize → 任务加入workQueue
3. workQueue已满 → 当前线程数 < maximumPoolSize → 创建新非核心线程执行
4. workQueue已满 且 当前线程数 == maximumPoolSize → 执行拒绝策略
5. 非核心线程空闲超过keepAliveTime → 回收。JDK6后allowCoreThreadTimeOut=true允许回收核心线程

**为什么不推荐Executors**：FixedThreadPool/SingleThreadPool使用无界LinkedBlockingQueue，请求堆积OOM；CachedThreadPool允许无限创建线程，线程数暴增OOM。

## 2.10 4种线程池拒绝策略分别是什么？
1. **AbortPolicy（默认）**：丢弃任务并抛出RejectedExecutionException异常，调用方需要自行处理
2. **CallerRunsPolicy**：由调用者所在线程（提交任务的线程）执行该任务。此策略会降低新任务提交速度，起到削峰作用。主线程执行任务期间无法提交新任务
3. **DiscardPolicy**：直接丢弃新提交的任务，不抛异常，不通知。风险：任务静默丢失
4. **DiscardOldestPolicy**：丢弃workQueue队首（最老的）任务，然后重新submit当前任务（相当于挤出最老任务给新任务让位）

可自定义实现RejectedExecutionHandler接口。常用方案：结合降级策略，如将任务持久化到DB或消息队列。

## 2.11 CompletableFuture常用方法和使用场景？
**创建**：runAsync（无返回）、supplyAsync（有返回），可指定线程池
**中间操作**：thenApply（转换返回值）、thenAccept（消费返回值）、thenRun（无参数无返回）、thenCompose（扁平化，类似flatMap）、thenCombine（两个Future结果合并）
**多任务编排**：allOf(多个Future).thenRun()等待所有完成；anyOf(多个Future).thenAccept()任一完成即继续
**异常处理**：exceptionally（异常时提供默认值）、handle（无论是否异常都执行，可恢复）、whenComplete（类似finally，可获取异常和结果）

典型场景：多个RPC调用并行执行后聚合结果、异步处理链路（查询→处理→落库→发送通知）、超时控制orTimeout/completeOnTimeout。

## 2.12 什么是死锁？如何排查和避免？
死锁：两个或多个线程互相等待对方释放锁，导致无限期阻塞。

**四个必要条件**：
1. 互斥：资源同一时刻只能被一个线程使用
2. 占有且等待：线程持有资源的同时等待其他资源
3. 不可剥夺：线程持有的资源不能被强制夺走
4. 循环等待：线程A等线程B的资源，线程B等线程A的资源，形成环

**排查**：
1. `jps -l` 找到Java进程PID
2. `jstack PID` 查看线程堆栈，末尾会列出死锁信息（Found one Java-level deadlock:...）
3. `jconsole` 可视化监控线程阻塞状态
4. Arthas `thread -b` 找出持有死锁的线程

**避免**：
1. 固定加锁顺序（所有线程按相同顺序获取锁）
2. 使用tryLock(time, unit)带超时的锁（超过时间放弃避免死锁）
3. 减少锁粒度，减小同步范围
4. 使用JUC的高级并发工具（如使用ConcurrentHashMap替代手动加锁的HashMap）

## 2.13 Java21虚拟线程是什么？与传统线程的区别？
虚拟线程（Virtual Threads）是JEP 444在Java21正式发布的特性，由Project Loom引入。

**核心概念**：虚拟线程是JVM管理的轻量级线程（协程），由JVM调度器将大量虚拟线程映射到少量OS平台线程上执行（M:N调度）。

**创建方式**：
```java
Thread.ofVirtual().start(() -> { ... });
Thread.startVirtualThread(() -> { ... });
Executors.newVirtualThreadPerTaskExecutor();
```

**与传统平台线程的区别**：
- 平台线程是OS线程的1:1包装，创建和上下文切换成本高（约1MB栈空间）
- 虚拟线程是JVM管理的轻量级对象，创建成本极低（只需一个Java对象），栈空间在堆上动态使用
- 当虚拟线程执行阻塞操作（IO、sleep）时，JVM自动将其从OS线程卸载，让OS线程去执行其他就绪的虚拟线程
- 虚拟线程不应池化（创建很便宜），应每个任务创建一个
- 可支持百万级虚拟线程并发

**注意事项**：synchronized块中阻塞会固定（pinned）平台线程，影响吞吐量（未来版本可能解决）；CPU密集型不适合虚拟线程。
