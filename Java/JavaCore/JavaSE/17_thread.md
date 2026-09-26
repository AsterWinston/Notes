# 1.多线程的实现方式

## 1.1继承Thread类的方式进行实现:这种方式扩展性差

```java
1.1.1MyThread extends Thread
1.1.2rewrite public void run(){}
1.1.3MyThread thread = new MyThread()//MyThread extends Thread
1.1.4thread.start()//启动线程，执行 run() 方法
```

## 1.2实现Runnable接口的方式进行实现:改善了扩展性，但是无法获取返回值

```java
1.2.1MyRun implements Runnable
1.2.2rewrite public void run(){}//在实现 run 函数时若要访问当前线程信息，可使用 Thread 中的静态方法 currendThread() 获取当前线程对象
1.2.3MyRun mr = new MyRun()
1.2.4Thread thread = new Thread(mr)
1.2.5thread.start()
```

## 1.3利用Callable接口和Future接口方式实现

```java
1.3.1implements Callable<Type>// 接口带有泛型，表示返回值类型
1.3.2rewrite public Type call() throws Exception{}// 核心方法，有返回值且可以抛出异常
1.3.3MyCallable mc = new MyCallable()
1.3.4FutureTask<Type> ft = new FutureTask<>(mc) // FutureTask 既实现了 Runnable 又实现了 Future
1.3.5Thread thread = new Thread(ft)
1.3.6thread.start()
1.3.7ft.get()// to get the running result (获取运行结果，该方法是阻塞的)
```

# 2.Thread常见成员方法

```
2.1String getName()// 获取线程名称
2.2void setName(String name)// 设置线程名称
2.3static Thread currentThread()// 获取当前正在执行的线程对象
2.4static void sleep(long time)// 让线程休眠指定时间（单位：毫秒）
2.5setPriority(int newPriority)// 设置线程优先级 1-10，默认为 5，Java 多线程为抢占式，优先级越高，线程抢到 CPU 的概率越高
2.6final void setDaemon(boolean on)// 设置为守护线程，当非守护线程（用户线程）全部结束时，守护线程会陆续结束（不是直接结束），即使守护线程没有执行完毕
2.7public static void yield()// 出让线程/礼让线程（即让当前线程出让 CPU 执行权，回到就绪状态，但下次仍有可能立即抢到）
2.8public static void join()// 插入线程/插队线程（把某个线程插入当前线程，当前线程必须等待被插入线程执行完毕后才能继续执行）
```

# 3.线程的生命周期

```
线程的生命周期包括：新建、就绪、运行、阻塞、等待、结束等状态。
```

# 4.线程安全

## 4.1同步代码块

```java
synchronized(<lock_obj>){
	codes
}
// 锁对象需要是唯一的，不然就无效了，比如可以使用某个类的 .class（字节码对象）又或者是自定义一个 static 对象作为锁对象
// 4.1.1 锁默认打开，有一个线程进去了，锁自动关闭
// 4.1.2 里面的代码全部执行完毕，线程出来，锁自动打开
```

## 4.2同步方法

```java
<authority> synchronized <ret_val_type> <func_name>(params){...}
// 4.2.1 同步方法是锁住方法里面所有的代码
// 4.2.2 锁对象不能自己指定
// 4.2.2.1 非静态方法: 默认锁对象是 this (当前实例对象)
// 4.2.2.2 静态方法: 默认锁对象是当前类的字节码文件对象（ClassName.class）
```

## 4.3ReentrantLock（实现了Lock接口的类。多个线程访问同一个资源，锁对象必须是唯一的）

```java
4.3.1lock()// 获取锁，阻塞方法，如果获取不到（被占用）就会一直等待
4.3.2unlock()// 释放锁（必须在 finally 块中释放，以保证无论是否发生异常都能释放锁）
```

## 4.4死锁

```
死锁：多个线程相互等待对方释放锁，导致所有线程都无法继续执行的状态。
```

# 5.等待唤醒机制（方法使用多个线程共享的锁来调用，这样，唤醒的就是绑定这个锁的线程）

```java
// 注意：以下方法必须在同步代码块或同步方法中调用，且通过锁对象调用
5.1void wait()// 当前线程等待，直到被其他线程唤醒（释放锁并进入等待池）
5.2void notify()// 随机唤醒单个等待（wait）中的线程
5.3void notifyAll()// 唤醒所有等待中的线程
```

# 6.阻塞队列（内部自带锁）（继承结构为Iteralbe-\>Collection-\>Queue-\>BlockingQueue）

```
6.1ArrayBlockingQueue// 基于数组的有界阻塞队列
6.2LinkedBlockingQueue// 基于链表的有界/无界阻塞队列
```

# 7.线程的状态

```
7.1新建状态(NEW)// 线程对象创建但未调用 start()
7.2就绪状态(RUNNABLE)// 调用 start() 后，等待 CPU 调度
7.3阻塞状态(BLOCKING)// 等待获取锁（进入 synchronized 块/方法）
7.4等待状态(WAITING)// 无限期等待其他线程通知（如调用 wait(), join()）
7.5计时状态(TIMED_WAITING)// 在指定时间后自动唤醒（如调用 sleep(), wait(time), join(time)）
7.6结束状态(TERMINATED)// run() 方法执行完毕或异常退出
```

# 8.线程池

## 8.1构造方法（Executors类中的方法）

```java
8.1.1public static ExecutorService newCachedThreadPool()// 创建一个没有上限的线程池（核心线程数为 0，空闲线程 60s 后回收）
8.1.2public static ExecutorService newFixedThreadPool(int nTreads)// 创建有上限的线程池（核心线程数和最大线程数相等，空闲线程永不回收）
```

## 8.2成员方法

```java
8.2.1Future<?> submit(Runnable task)// 提交 Runnable 任务，返回 Future 对象用于检查状态，无返回值
8.2.2<T> Future<T> submit(Runnable task, T result)// 提交 Runnable 任务，返回 Future 对象用于获取指定的 result 结果
8.2.3<T> Future<T> submit(Callable<T> task)// 提交 Callable 任务，返回 Future 对象用于获取 T 类型返回值
```

# 9.自定义线程池

## 9.1ThreadPoolExecutor构造方法

```java
9.1.1public ThreadPoolExecutor(int corePoolSize, // 核心线程数
                              int maximumPoolSize, // 最大线程数
                              long keepAliveTime, // 非核心线程的空闲存活时间
                              TimeUnit unit, // 存活时间单位
                              BlockingQueue<Runnable> workQueue, // 任务队列
                              ThreadFactory threadFactory, // 创建新线程的工厂
                              RejectedExecutionHandler handler) // 拒绝策略
```

## 9.2ThreadPoolExecutor成员方法

```
常用方法包括 execute(Runnable) 和 submit(Callable/Runnable)。
```

## 9.3线程池大小的合理设置

```
9.3.1cpu密集型运算: 核心线程数一般设置为 **CPU 最大并行数 + 1**
9.3.2i/o密集型运算: 核心线程数设置公式：**最大并行数 * (1 + 期望 CPU 利用率 * (CPU 计算时间 + 等待时间) / CPU 计算时间)** (通常为 CPU 核心数的 2 倍或更高，取决于 I/O 耗时比例)
```