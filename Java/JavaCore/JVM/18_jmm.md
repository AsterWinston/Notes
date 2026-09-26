
# 1. 什么是 JMM

- JMM（Java Memory Model）定义了一套多线程读写共享变量的规则。
- 核心目标：定义程序中各变量的**访问规则**，即变量读取和写入的底层细节。
- JMM 屏蔽了不同硬件/操作系统的内存访问差异。

## 1.1 为什么需要 JMM

```java
public class VisibilityProblem {
    private static boolean flag = false;
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            while (!flag) {
                // 线程可能永远看不到 flag 被改为 true
            }
            System.out.println("done");
        }).start();
        
        Thread.sleep(1000);
        flag = true;  // 主线程修改了 flag
    }
}
```

- 没有 JMM 的保证，线程间变量修改的可见性不能保证。
- 可能原因：CPU 缓存、编译器指令重排序、JIT 优化。

# 2. 主内存与工作内存

```
┌────────────────────────────────────────────────┐
│                      JVM                        │
│  ┌─────────────────┐    ┌─────────────────┐    │
│  │    线程 1        │    │    线程 2        │    │
│  │  ┌───────────┐  │    │  ┌───────────┐  │    │
│  │  │ 工作内存   │  │    │  │ 工作内存   │  │    │
│  │  │ (CPU缓存)  │  │    │  │ (CPU缓存)  │  │    │
│  │  └─────┬─────┘  │    │  └─────┬─────┘  │    │
│  └────────┼────────┘    └────────┼────────┘    │
│           │       交互             │            │
│           ▼                     ▼            │
│  ┌──────────────────────────────────────┐      │
│  │            主内存 (Main Memory)       │      │
│  │               (共享变量)             │      │
│  └──────────────────────────────────────┘      │
└────────────────────────────────────────────────┘
```

- **主内存**：所有线程共享，存放实例字段、静态字段、数组元素。
- **工作内存**：线程私有，是主内存中变量的副本。
- 8 种原子操作：`lock`、`unlock`、`read`、`load`、`use`、`assign`、`store`、`write`。

# 3. 重排序

## 3.1 三种重排序类型

| 类型 | 说明 |
|------|------|
| 编译器重排序 | JIT 编译器在不改变单线程语义下调整指令顺序 |
| 指令级并行重排序 | CPU 指令流水线优化（如果数据不依赖） |
| 内存系统重排序 | CPU 缓存写入主内存的顺序可能与执行顺序不同 |

```
源代码 → 编译器优化重排序 → 指令级并行重排序 → 内存系统重排序 → 最终执行的指令序列
```

## 3.2 重排序导致的问题

```java
// 经典的"双重检查锁定"问题（JDK 5 之前）
public class Singleton {
    private static Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                    // new Singleton() 分解为：
                    // 1. 分配内存
                    // 2. 调用构造函数
                    // 3. 将引用赋值给 instance
                    // 重排序后可能 1→3→2，其他线程拿到未构造完的对象
                }
            }
        }
        return instance;
    }
}
```

# 4. volatile

## 4.1 volatile 的两个语义

1. **可见性**：一个线程对 volatile 变量的修改，其他线程**立即可见**。
2. **禁止指令重排序**：volatile 变量的读写前后不能重排序。

```java
// volatile 保证可见性
volatile boolean flag = false;

// 线程1
flag = true;  // 写入后立即刷回主内存

// 线程2
if (flag) {   // 每次读取都从主内存获取最新值
    // ...
}
```

## 4.2 volatile 的内存屏障

JMM 通过**内存屏障**（Memory Barrier）实现 volatile 语义。

```
StoreStore 屏障：禁止普通写和 volatile 写重排序
  ↓
volatile 写
  ↓
StoreLoad 屏障：禁止 volatile 写和后续 volatile 读/写重排序

---

volatile 读
  ↓
LoadLoad 屏障：禁止 volatile 读和后续普通读重排序
  ↓
LoadStore 屏障：禁止 volatile 读和后续普通写重排序
```

## 4.3 volatile 的典型用法

**1. 状态标记**
```java
volatile boolean shutdown = false;

public void shutdown() {
    shutdown = true;
}

public void work() {
    while (!shutdown) {
        doWork();
    }
}
```

**2. DCL 单例（JDK 5+）**
```java
public class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**3. 开销较低的读写锁**
```java
volatile int count = 0;

// 读：不需要锁
public int getCount() { return count; }

// 写：需要锁保证原子性
public synchronized void increment() { count++; }
```

## 4.4 volatile 不能保证原子性

```java
volatile int count = 0;

// 多线程执行 count++ — 依然有并发问题！
// count++ 是三个操作：读-改-写
Thread t1 = new Thread(() -> { for (int i = 0; i < 10000; i++) count++; });
Thread t2 = new Thread(() -> { for (int i = 0; i < 10000; i++) count++; });
// 结果不一定是 20000
```

# 5. happens-before 原则

happens-before 是 JMM 中判断数据是否存在竞争、是否线程安全的主要依据。

若操作 A happens-before 操作 B，则 A 的结果对 B 可见，且 A 的执行顺序在 B 之前。

**8 条规则：**

| 规则 | 说明 | 示例 |
|------|------|------|
| 程序次序规则 | 同一线程中前面的操作 happens-before 后面的 | A=1; B=A; (A happens-before B) |
| 监视器锁规则 | unlock happens-before 后续的 lock | synchronized 块 |
| volatile 规则 | volatile 写 happens-before 后续的读 |
| 线程启动规则 | Thread.start() happens-before 线程内所有操作 |
| 线程终止规则 | 线程内所有操作 happens-before Thread.join() 返回 |
| 线程中断规则 | interrupt() happens-before 被中断线程检测到中断 |
| 对象终结规则 | 构造方法结束 happens-before finalize() 开始 |
| 传递性 | A hb B, B hb C → A hb C |

```java
// happens-before 实例
int a = 0;
volatile boolean flag = false;

// 线程 1
a = 1;          // 1
flag = true;    // 2

// 线程 2
if (flag) {     // 3  (volatile 读)
    int b = a;  // 4  (由于2 hb 3，且1 hb 2，所以1 hb 4，b一定=1)
}
```

# 6. final 域的内存语义

## 6.1 final 域的重排序规则

1. **写**：在构造函数内对 final 域的写入，与随后构造函数的引用赋值给一个引用变量，这两个操作**不能重排序**。
2. **读**：初次读一个包含 final 域的对象引用，与随后读这个 final 域，这两个操作**不能重排序**。

```java
// final 保证线程安全的发布
public class SafePublication {
    private final int x;
    private int y;
    
    public SafePublication(int x) { 
        this.x = x;   // final 写
        // StoreStore 屏障：保证 x 的写不被重排序到构造函数之外
        this.y = 1;
    }
    
    // 多线程安全：读取安全发布的对象时，final 域一定被正确初始化
}
```

## 6.2 final 域是引用类型

```java
public class RefFinal {
    private final int[] array;
    
    public RefFinal() {
        array = new int[10];   // final 写
        array[0] = 1;          // 数组元素的写在构造函数 return 前完成
    }
}
```

- 构造函数内对 final 引用指向的对象的成员域的修改，在构造函数 return 后才能被其他线程看见。

# 7. 内存屏障（Memory Barrier）

| 屏障类型 | 指令示例 | 说明 |
|----------|----------|------|
| LoadLoad | Load1; LoadLoad; Load2 | Load1 的读取一定先于 Load2 |
| StoreStore | Store1; StoreStore; Store2 | Store1 的写入对其他处理器先于 Store2 可见 |
| LoadStore | Load1; LoadStore; Store2 | Load1 的读取一定先于 Store2 的写入 |
| StoreLoad | Store1; StoreLoad; Load2 | Store1 的写入对所有处理器可见后，才能执行 Load2 |

**StoreLoad 是最重的屏障**（同时具有其他三个屏障的效果），但开销最大。

# 8. JMM 与上层 Java 内存工具

| 工具 | JMM 基础 | 典型场景 |
|------|----------|----------|
| volatile | 可见性 + 禁止重排序 | 状态标记、DCL 单例 |
| synchronized | 原子性 + 可见性 + 禁止重排序 | 互斥访问 |
| Lock (J.U.C) | 基于 volatile + CAS | 更灵活的锁 |
| AtomicXXX | CAS (底层是 LOCK CMPXCHG 指令) | 无锁计数、状态更新 |
| final | 构造函数安全发布 | 不可变对象 |

# 9. 实战：验证可见性问题

```java
public class VisibilityDemo {
    // 不加 volatile
    private static boolean flag = false;
    // private static volatile boolean flag = false;  // 加上可见性
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            int i = 0;
            while (!flag) {
                i++;
                // 如果加入 System.out.println，可能因为 sout 内部的
                // synchronized 导致工作内存被刷新，flag 突然可见
            }
            System.out.println("Thread stopped, i=" + i);
        }).start();
        
        Thread.sleep(1000);
        flag = true;
        System.out.println("Flag set to true");
    }
}
// 不加 volatile，子线程可能永不停止（C2 编译后 flag 被优化到寄存器）
// 加上 volatile 后，子线程能立即感知 flag 的变化
```
