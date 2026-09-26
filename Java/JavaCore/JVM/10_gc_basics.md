
# 1. 垃圾回收要解决的问题

1. **哪些对象是垃圾？**—— 判断对象是否存活。
2. **何时回收？**—— GC 触发时机。
3. **如何回收？**—— 垃圾收集算法。

# 2. 对象存活判断

## 2.1 引用计数法

- 每个对象维护一个引用计数器，被引用+1，引用失效-1，计数器为 0 即可回收。
- **优点**：简单高效。
- **缺点**：无法解决循环引用问题。

```java
public class RefCount {
    public Object ref;
    
    public static void main(String[] args) {
        RefCount a = new RefCount();
        RefCount b = new RefCount();
        a.ref = b;
        b.ref = a;
        a = null;
        b = null;
        // a 和 b 互相引用，引用计数都不为 0，但实际已经不可达
        // 引用计数法无法回收
    }
}
```

- **Python 也使用引用计数**，但通过标记-清除辅助解决循环引用。
- **HotSpot 不使用引用计数法**。

## 2.2 可达性分析（Reachability Analysis）

- 以一系列 **GC Roots** 为起点，沿着引用链向下搜索，搜索路径称为**引用链**（Reference Chain）。
- 搜索不到的对象判定为**不可达**，可以被回收。

```
GC Roots:
  ● (栈中引用)
  │
  │  可达对象
  ▼  (不会被回收)
  ○ ──→ ○ ──→ ○
  
  ○ (不可达) ← 将被回收
```

# 3. GC Roots 有哪些

| GC Root | 说明 | 示例 |
|---------|------|------|
| 虚拟机栈中引用的对象 | 各线程栈帧局部变量表中的引用 | 方法内 `new` 的对象 |
| 方法区中静态属性引用的对象 | static 字段引用的对象 | `static Object obj = new Object()` |
| 方法区中常量引用的对象 | final 常量引用的对象 | `static final String S = "abc"` (S 在池中) |
| 本地方法栈中 JNI 引用的对象 | Native 代码中引用的对象 | `jobject` |
| JVM 内部引用 | 基本类型的 Class 对象、常驻异常等 | `java.lang.Class` |
| 被同步锁(synchronized)持有的对象 | | 锁住的对象 |
| JVM 内部的 JMXBean、JVMTI 回调等 | | |

# 4. 引用类型

Java 中的引用分为 4 种，从强到弱：

## 4.1 强引用（Strong Reference）

- 最常见的引用：`Object obj = new Object();`
- 只要强引用还存在，GC 绝不会回收。

## 4.2 软引用（Soft Reference）

- 有用但非必需的对象。
- 在 OOM 之前，GC 会回收软引用指向的对象。
- **适合做缓存**（内存敏感缓存）。

```java
SoftReference<byte[]> softRef = new SoftReference<>(new byte[10 * 1024 * 1024]);
byte[] data = softRef.get();   // 可能返回 null（已被回收）
```

## 4.3 弱引用（Weak Reference）

- 非必需对象，比软引用更弱。
- 无论内存是否充足，只要 GC 就会回收。
- **适用于不会阻止被回收的附加数据，如 WeakHashMap、ThreadLocal 等。**

```java
WeakReference<Object> weakRef = new WeakReference<>(new Object());
System.gc();
Object obj = weakRef.get();  // 可能返回 null
```

**WeakHashMap 示例：**
```java
WeakHashMap<Object, String> map = new WeakHashMap<>();
Object key = new Object();
map.put(key, "value");

key = null;           // key 的唯一强引用消失
System.gc();          // 触发 GC
// map 中的该条目被自动清除
```

## 4.4 虚引用（Phantom Reference）

- 完全不影响对象生命周期。
- 唯一作用：对象被回收时收到一个系统通知。
- **用于跟踪对象回收活动，如 DirectByteBuffer 的堆外内存回收。**

```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(new Object(), queue);
// 对象被回收后，phantom 会被加入 queue
```

# 5. finalize()

- `finalize()` 是 `Object` 的 protected 方法，GC 前可能被调用。
- **已被 JDK 9 标记为 `@Deprecated`，不推荐使用。**
- 不确定性大（何时调用、是否调用都不确定），性能差。
- 替代方案：`Cleaner`（JDK 9+）或 `try-with-resources`。

```java
public class FinalizeDemo {
    @Override
    protected void finalize() throws Throwable {
        System.out.println("finalize called");
        super.finalize();
    }
    
    public static void main(String[] args) {
        new FinalizeDemo();
        System.gc();
        // 输出不确定——finalize 可能被调用，也可能不调用
    }
}
```

finalize 的问题：
1. 执行时机不确定
2. 可能导致对象"复活"（在 finalize 中被重新引用）
3. 减慢 GC（需要两次可达性分析）
4. JDK 9 deprecated, JDK 18+ 可能在后续版本中最终移除

# 6. 方法区的回收

方法区（元空间）也能被回收，但条件苛刻：
- 废弃的常量（字符串常量池中没有引用的常量）。
- 不再使用的类。

类卸载条件（三者同时满足）：
1. 该类所有实例已回收
2. 加载该类的类加载器已回收
3. 该类的 Class 对象没有被引用

# 7. GC 类型

| GC 类型 | 回收区域 | 触发条件 |
|---------|---------|----------|
| Young GC / Minor GC | 新生代（Eden + Survivor） | Eden 空间不足 |
| Major GC | 老年代 | 老年代空间不足（通常伴随 Minor GC） |
| Full GC | 整堆 + 元空间 | System.gc()、老年代/元空间不足、分配担保失败 |
| Mixed GC | 整个新生代 + 部分老年代（G1） | G1 的混合回收 |

- **Minor GC 非常频繁**，但速度快（几十毫秒）。
- **Full GC 尽量避免**，一般停顿时间长（几百毫秒到几秒）。
- G1 的 Mixed GC 介于两者之间。

**触发 Full GC 的常见场景：**
1. `System.gc()` 的显式调用
2. 老年代空间不足
3. 元空间/永久代空间不足
4. CMS GC 时的并发模式失败（Concurrent Mode Failure）
5. Minor GC 时的分配担保失败（Handle Promotion Failure）
6. `-XX:+DisableExplicitGC` 可以忽略 `System.gc()` 调用

# 8. Stop The World（STW）

- GC 过程中，所有应用程序线程都要暂停。
- 目的是保证 GC Roots 和分析过程中的一致性（一致性快照）。
- 不同 GC 收集器的 STW 时间差异巨大。

```java
// 体验 STW
public class StopTheWorld {
    public static void main(String[] args) {
        PrintThread thread = new PrintThread();
        thread.start();
        
        // 每 100ms 分配大量对象，触发 GC
        List<Object> list = new ArrayList<>();
        while (true) {
            for (int i = 0; i < 1000; i++) {
                list.add(new byte[1024 * 1024]);
            }
            list.clear();
            System.gc();
            Thread.sleep(100);
        }
    }
    
    static class PrintThread extends Thread {
        long last = System.currentTimeMillis();
        public void run() {
            while (true) {
                long now = System.currentTimeMillis();
                long gap = now - last;
                if (gap > 100) {
                    System.out.println("STW detected: " + gap + "ms");
                }
                last = now;
            }
        }
    }
}
```

# 9. 安全点与安全区域

**安全点（Safepoint）：**
- GC 时，不是任意位置都能停顿，需要等线程都到达**安全点**。
- 安全点通常设在：方法调用、循环跳回、异常跳转等位置。

**主动式中断**（HotSpot 采用）：
1. 设置一个标志
2. 各线程轮询这个标志，发现需要中断就自己挂起

**安全区域（Safe Region）：**
- 线程处于 Sleep/Blocked 状态时无法响应中断请求。
- 安全区域表示一段代码中，引用关系不会发生变化，在这段区域任何位置 GC 都是安全的。