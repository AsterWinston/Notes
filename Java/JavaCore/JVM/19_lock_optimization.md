
# 1. Java 中锁的优化历程

```
JDK 1.0-1.2: synchronized 直接使用 OS 互斥量（重量级锁），非常慢
JDK 1.3-1.4: 初步优化
JDK 1.5: 引入 J.U.C (ReentrantLock, AtomicXXX, ConcurrentHashMap 等)
JDK 1.6: 锁优化重大升级 —— 偏向锁、轻量级锁、自旋锁、锁消除、锁粗化
JDK 1.7-1.8: 持续优化，ConcurrentHashMap 改用 CAS + synchronized
JDK 15: 默认关闭偏向锁
JDK 21: 虚拟线程 + 进一步优化
```

# 2. 对象头中的锁信息

回顾 Mark Word（64位）：

| 锁状态 | 标志位(2bit) | 偏向锁位(1bit) | 存储内容 |
|--------|-------------|---------------|----------|
| 无锁 | 01 | 0 | hashCode（25bit），分代年龄（4bit） |
| 偏向锁 | 01 | 1 | 线程 ID（54bit），epoch（2bit），分代年龄（4bit） |
| 轻量级锁 | 00 | - | 指向栈中锁记录（Lock Record）的指针 |
| 重量级锁 | 10 | - | 指向 ObjectMonitor 的指针 |
| GC 标记 | 11 | - | 空 |

# 3. 锁升级过程

```
                   ┌──────────┐
                   │   无锁    │
                   └────┬─────┘
             首次线程A 请求锁
                        │
                   ┌────▼─────┐
                   │  偏向锁   │  ← 记录线程A的ID，A再进无需CAS
                   └────┬─────┘
            线程B 竞争同一把锁
                        │
                   ┌────▼─────┐
     撤销偏向 ←────│  竞争     │
        │          └────┬─────┘
        撤销后            │
        升级              ▼
                   ┌──────────┐ ← 自旋获取（CAS）
                   │ 轻量级锁  │
                   └────┬─────┘
          自旋超过阈值 / 竞争激烈
                        │
                   ┌────▼─────┐
                   │ 重量级锁  │ ← ObjectMonitor, OS 互斥
                   └──────────┘
```

- **锁只能升级，不能降级**（偏向锁可撤销变为无锁）。

# 4. 偏向锁（Biased Locking）

- 当一个线程首次获取锁时，锁对象的 Mark Word 中记录该线程 ID。
- 该线程后续再次进入同步块，无需 CAS，无需加锁。
- **目的**：消除无竞争场景下的同步开销。

```java
// 偏向锁受益场景
public class BiasedLocking {
    // Vector 的 add 方法是 synchronized 的
    private final Vector<String> list = new Vector<>();
    
    public void add() {
        // 大多数情况只有单线程调用 → 偏向锁极高效
        list.add("item");
    }
}
```

**关闭偏向锁的原因（JDK 15+ 默认关闭）：**
- 现代应用大量使用 J.U.C 的并发集合，synchronized 使用场景减少。
- 偏向锁的撤销在竞争激烈时需要 STW，代价高。
- 高并发场景下偏向锁反而拖慢性能。

```bash
-XX:+UseBiasedLocking          # 开启偏向锁（JDK 15 之前默认）
-XX:-UseBiasedLocking          # 关闭
-XX:BiasedLockingStartupDelay=0  # 启动延迟（默认 4 秒）
```

# 5. 轻量级锁（Lightweight Locking）

- 线程在进入同步块前，在栈帧中创建 **Lock Record**。
- 使用 CAS 将对象的 Mark Word 更新为指向 Lock Record 的指针。
- 如果 CAS 成功 → 获得轻量级锁。
- 如果 CAS 失败 → 自旋重试（自适应自旋）。

```
线程栈                   堆中的对象
┌────────────┐         ┌──────────────────┐
│ 栈帧        │         │  对象头            │
│ ┌────────┐ │    CAS  │ ┌──────────────┐  │
│ │Lock    │ │←───────│ │ Mark Word    │  │
│ │Record  │ │         │ │ (指向LR)      │  │
│ └────────┘ │         │ ├──────────────┤  │
└────────────┘         │ │ Klass Pointer│  │
                       │ └──────────────┘  │
                       └──────────────────┘
```

**轻量级锁适用场景：** 两个线程交替执行同步块（很短暂的竞争）。

# 6. 重量级锁（Heavyweight Locking）

- 轻量级锁 CAS 失败，且自旋超过一定次数或有大量线程等待 → 升级为重量级锁。
- 依赖 OS 的互斥量（Mutex），线程阻塞/唤醒需要**系统调用**。
- 未获取锁的线程进入**阻塞**状态（WAITING / BLOCKED）。

**ObjectMonitor 结构：**
```
对象关联的 ObjectMonitor:
┌─────────────────────┐
│ _owner    → 持有锁的线程      │
│ _WaitSet  → wait() 等待的线程  │
│ _EntryList → 等待获取锁的线程  │
│ _count    → 重入计数          │
└─────────────────────┘
```

# 7. 自旋锁与自适应自旋

## 7.1 自旋锁
- 轻量级锁 CAS 失败后，线程不立即阻塞，而是执行空循环（自旋）。
- 如果锁很快释放，自旋避免了线程挂起/唤醒的开销。
- 如果自旋太久，浪费 CPU。

```bash
-XX:PreBlockSpin=10            # 自旋次数（JDK 6 及之前，已废弃）
```

## 7.2 自适应自旋（JDK 6+）
- 自旋时间不再固定，由前一次在**同一个锁**上的自旋时间和锁的拥有者状态决定。
- 如果之前自旋成功过 → 自旋更久。
- 如果之前很少成功 → 直接阻塞。

# 8. 锁粗化（Lock Coarsening）

- JIT 编译时优化，将频繁的加锁/解锁合并。

```java
// 优化前
for (int i = 0; i < 10000; i++) {
    synchronized (lock) {
        counter++;
    }
}

// JIT 粗化后
synchronized (lock) {
    for (int i = 0; i < 10000; i++) {
        counter++;
    }
}
```

# 9. 锁消除（Lock Elimination）

- 基于逃逸分析，消除无竞争的锁。

```java
// StringBuffer → 逃逸分析发现 sb 只在线程内使用 → 消除 synchronzied
public String build() {
    StringBuffer sb = new StringBuffer();
    sb.append("a");
    sb.append("b");
    return sb.toString();
}
// 优化后等效于 StringBuilder
```

# 10. synchronized 的底层实现

```java
public synchronized void doWork() {
    // ...
}

// 等效于
public void doWork() {
    synchronized (this) {
        // ...
    }
}
```

**字节码层面：**
```
// 同步方法：方法访问标志中设置 ACC_SYNCHRONIZED

// 同步代码块：
monitorenter        // 尝试获取锁
  ...代码...
monitorexit         // 正常释放锁
  ...
monitorexit         // 异常时释放锁（异常表）
```

**monitorenter 的实现（C++ 伪代码）：**
```cpp
void monitorenter(oop obj) {
    if (obj->isBiasedLocked()) {
        // 偏向锁路径：检查线程ID，匹配则直接获取
        if (obj->biasedThread() == currentThread) return;
        // 否则撤销偏向锁
        revokeBias(obj);
    }
    
    // 轻量级锁路径：CAS Mark Word → Lock Record
    if (CAS(obj->markWord, DisplacedMarkWord, pointerToLockRecord)) {
        return;  // 成功获取轻量级锁
    }
    
    // 重量级锁路径：膨胀为 ObjectMonitor
    ObjectMonitor* monitor = inflate(obj);
    monitor->enter();  // 可能阻塞
}
```

# 11. ReentrantLock vs synchronized

| 对比 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM 层面（C++） | JDK 层面（Java） |
| 锁释放 | 自动（出了作用域） | 必须手动 unlock（finally） |
| 可中断 | 不可中断 | 可中断（lockInterruptibly()） |
| 公平锁 | 非公平 | 可选公平/非公平 |
| 条件 | wait/notify 简单条件 | 多个 Condition |
| 锁绑定条件 | 一个对象一个条件 | 一个锁多个条件 |
| 性能 | JDK 6+ 优化后接近 | 之前更快，现在差距很小 |

```java
// synchronized
synchronized (lock) {
    // 业务代码
}

// ReentrantLock
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 业务代码
} finally {
    lock.unlock();  // 必须手动释放
}
```

# 12. CAS（Compare And Swap）

- 无锁算法的核心、AtomicInteger 等原子类的实现基础。
- CPU 提供的原子指令（`LOCK CMPXCHG`）。

```java
// AtomicInteger 的 incrementAndGet 底层实现
public final int incrementAndGet() {
    return U.getAndAddInt(this, VALUE, 1) + 1;
}

// Unsafe.getAndAddInt:
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
}
```

**CAS 的三大问题：**

| 问题 | 说明 | 解决 |
|------|------|------|
| ABA 问题 | 值从 A→B→A，CAS 认为没变 | AtomicStampedReference（加版本号） |
| 循环开销大 | 高并发下 CAS 失败 → 自旋 → CPU 浪费 | 分段、退避策略 |
| 只能保证一个变量的原子操作 | 多个变量需要原子更新 | AtomicReference / 加锁 |

```java
// ABA 问题示例
// 线程1: CAS(A→C)  先取到 A
// 线程2: CAS(A→B) → CAS(B→A)  (A→B→A)
// 线程1: CAS 成功！但它不知道经历过 A→B→A

// 解决方法：AtomicStampedReference
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 0);
int stamp = ref.getStamp();
ref.compareAndSet(100, 200, stamp, stamp + 1);  // 同时比较值和版本号
```

# 13. 实战：验证锁升级

```java
// -XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
// 使用 JOL 观察锁状态变化
import org.openjdk.jol.info.ClassLayout;

public class LockUpgrade {
    public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();
        
        System.out.println("===== 无锁状态 =====");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        
        // 偏向锁
        synchronized (obj) {
            System.out.println("===== 偏向锁 =====");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
        
        // 轻量级锁：两线程交替
        Thread t2 = new Thread(() -> {
            synchronized (obj) {
                System.out.println("===== 轻量级锁 =====");
                System.out.println(ClassLayout.parseInstance(obj).toPrintable());
            }
        });
        t2.start();
        t2.join();
        
        // 重量级锁：竞争
        synchronized (obj) {
            new Thread(() -> {
                synchronized (obj) {
                    // 竞争
                }
            }).start();
            Thread.sleep(100); // 确保发生竞争
            
            System.out.println("===== 重量级锁 =====");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }
    }
}
```

# 14. 锁相关 JVM 参数

```bash
# 偏向锁
-XX:+UseBiasedLocking            # 开启（JDK 15+ 默认关闭）
-XX:BiasedLockingStartupDelay=0  # 启动不延迟

# 自旋
-XX:+UseSpinning                 # 自旋锁（JDK 6+ 默认）
-XX:PreBlockSpin=10              # 自旋次数（已废弃，改用自适应）

# 轻量级锁
-XX:+UseHeavyMonitors            # 禁用轻量级锁，直接使用重量级锁

# 锁优化
-XX:+EliminateLocks              # 锁消除（默认）
-XX:+UseLockedTracing            # 锁信息追踪
```
