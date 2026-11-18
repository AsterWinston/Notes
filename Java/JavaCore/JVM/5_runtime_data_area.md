# 1. 运行时数据区概述

## 1.1 定位

运行时数据区（Runtime Data Area）是 JVM 执行 Java 程序时的内存模型。注意两点：

- 它是《Java 虚拟机规范》定义的**逻辑划分**，HotSpot 实现与之有出入（本地方法栈与虚拟机栈合并实现、方法区从堆内搬到本地内存）。
- 图之外还有**直接内存**等区域：不在规范图里，却常常是"进程 RSS 远大于堆"的元凶。

Java 8 及之后的完整架构：

```text
┌──────────────────────────────────────────────────┐
│                   JVM 内存结构                    │
├──────────────┬──────────────┬────────────────────┤
│   线程私有    │   线程私有    │      线程共享       │
│              │              │                    │
│ ┌──────────┐ │ ┌──────────┐ │ ┌───────────────┐  │
│ │   程序   │ │ │ 虚拟机栈  │ │ │     堆        │  │
│ │  计数器   │ │ │          │ │ │   (Heap)      │  │
│ │  (PC)    │ │ │ ┌──────┐ │ │ │               │  │
│ │          │ │ │ │栈帧N │ │ │ │  - 新生代     │  │
│ └──────────┘ │ │ │栈帧2 │ │ │ │  - 老年代     │  │
│              │ │ │栈帧1 │ │ │ │  - 字符串常量池│  │
│ ┌──────────┐ │ │ └──────┘ │ │ └───────────────┘  │
│ │ 本地方法  │ │ │          │ │                    │
│ │   栈     │ │ └──────────┘ │ ┌───────────────┐  │
│ │ (Native) │ │              │ │ 方法区/元空间  │  │
│ └──────────┘ │              │ │ (Method Area) │  │
│              │              │ │               │  │
│              │              │ │  - 类元数据   │  │
│              │              │ │  - 运行时常量池│  │
│              │              │ │  - 静态变量→堆│  │
│              │              │ │  - JIT代码缓存 │  │
│              │              │ └───────────────┘  │
├──────────────┴──────────────┴────────────────────┤
│   直接内存 (Direct/Native Memory)：NIO Buffer 等  │
└──────────────────────────────────────────────────┘
```

## 1.2 一次代码执行，数据落在哪些区域

```java
class User {
    static int COUNT;            // 堆：存在 User 的 Class 对象内（JDK 7+）
    String name;                 // 指向的 String 对象在堆
    int add(int a, int b) {      // 栈：a、b 进局部变量表 slot
        int sum = a + b;         // 栈：sum 进 slot；运算过程用操作数栈
        return sum;              // 返回后结果压回调用者栈帧
    }
    void run() {
        ByteBuffer buf = ByteBuffer.allocateDirect(64);
        // 堆：DirectByteBuffer 壳对象；直接内存：真正的 64 字节数据
    }
}
```

```text
字节码执行到哪一行        → PC 寄存器
方法调用链 / 局部变量     → 虚拟机栈
对象实例 / 数组 / 字符串  → 堆
类结构 / 常量池 / JIT 码  → 元空间 + Code Cache（本地内存）
堆外大块缓冲            → 直接内存
```

# 2. 线程私有 vs 线程共享

## 2.1 总表（含版本演进列）

| 区域 | 私有/共享 | 存什么 | 生命周期 | 可能异常 | 随版本变化 |
|------|----------|--------|----------|----------|-----------|
| 程序计数器 | 私有 | 当前字节码指令地址 | 与线程同生同灭 | **无**（唯一） | 行为基本未变 |
| 虚拟机栈 | 私有 | 栈帧（局部变量表/操作数栈等） | 与线程同生同灭 | StackOverflowError / OOM | JDK 21 虚拟线程不再一一对应 OS 栈（JEP 444） |
| 本地方法栈 | 私有 | native 方法调用 | 与线程同生同灭 | StackOverflowError / OOM | HotSpot 中与虚拟机栈合并实现 |
| 堆 | 共享 | 对象实例、数组、字符串常量池 | JVM 启动时创建 | OOM（多种子类型） | JDK 9 起 G1 默认 → 堆不再物理连续；JDK 21 分代 ZGC 实验 |
| 方法区/元空间 | 共享 | 类元数据、运行时常量池 | JVM 启动时创建 | OOM: Metaspace | JDK 8 永久代→元空间（本地内存） |
| Code Cache | 共享 | JIT 编译的机器码 | JVM 启动时创建 | 满了 JIT 停摆（非 OOM） | JDK 9 起分三段管理 |
| 直接内存 | - | NIO Direct Buffer、mmap 等 | 不受 JVM 管辖 | OOM: Direct buffer memory | 不属于运行时数据区，但被频繁使用 |

## 2.2 私有与共享的工程含义

| 特性 | 私有区域 | 共享区域 |
|------|----------|----------|
| 并发 | 天然线程安全，分配即指针移动，无锁 | 需要 TLAB / CAS 保分配安全（见 [6_heap](6_heap.md)） |
| 回收 | 随栈帧出栈自动消失，**不需要 GC** | 全靠 GC，是调优主战场 |
| 大小问题 | 栈帧太多 → SOE；线程太多 → 栈总量爆 | 对象太多 → 堆 OOM |

# 3. 程序计数器（PC Register）

## 3.1 作用：字节码行号指示器

### 3.1.1 执行过程记录

PC 记录当前线程正在执行的字节码指令地址，分支、循环、跳转、异常处理、线程恢复都依赖它——线程被切换走再切回来，必须知道"执行到哪了"。

```text
字节码流:  0:iload_1  1:iload_2  2:iadd  3:istore_3  ...
线程1:     PC ─────────────────▲ (执行到第 3 条)
线程2:     PC ──────▲ (另一方法中第 6 条指令)
```

PC 是**每条线程独立一个**的（所以线程切换不互相污染），这也是它排在"线程私有"第一位的原因。

## 3.2 为什么它是唯一没有 OOM 的区域

### 3.2.1 大小固定、无分配动作

- 大小固定：一个机器字即可存下指令地址，**不随程序复杂度增长**。
- 不分配内存：规范层面它就是 CPU/解释器里的一个寄存器，没有"申请-扩容"动作，自然没有"申请失败"。

## 3.3 执行 native 方法时为 undefined

### 3.3.1 undefined 场景与 PrintAssembly 辨析

| 执行内容 | PC 值 |
|----------|-------|
| Java 方法（字节码） | 记录正在执行的字节码指令地址 |
| native 方法（C/C++） | **Undefined**——已进入本机代码，字节码地址失去意义，由 CPU 自己的 PC/寄存器接管 |

> 辨析：`-XX:+PrintAssembly` 打印的是 Code Cache 里 JIT 生成的**机器码**及其 CPU 寄存器，与本节的"字节码 PC"不是一回事，面试常被混为一谈。

# 4. 虚拟机栈（Java Virtual Machine Stack）

## 4.1 结构与栈帧

每个线程持有一个栈，每个方法执行时同步创建一个**栈帧**（Stack Frame），存局部变量表、操作数栈、动态链接、返回地址等。结构细节与字节码演示详见 [8_vm_stack](8_vm_stack.md)。

```text
方法调用链：
  main()               → 栈帧 main 压栈
    foo()              → 栈帧 foo 压栈
      bar()            → 栈帧 bar 压栈
      bar() 返回       → 栈帧 bar 出栈
    foo() 返回         → 栈帧 foo 出栈
  main() 返回          → 栈帧 main 出栈
```

## 4.2 StackOverflowError：单线程递归过深

```java
// -Xss256k
public class StackDepth {
    static int depth = 0;
    static void recurse() {
        depth++;
        recurse();
    }
    public static void main(String[] args) {
        try {
            recurse();
        } catch (StackOverflowError e) {
            System.out.println("max depth = " + depth); // 实测常见 3000~8000，随 JDK/平台变化
        }
    }
}
```

栈深上限 = `-Xss` / 单帧大小。方法局部变量越多、参数越多，帧越大，能容纳的深度越浅（估算见 4.4）。

## 4.3 OutOfMemoryError：栈的两种"不够"

### 4.3.1 异常触发方式对照

| 触发方式 | 说明 | 报错 |
|----------|------|------|
| 动态扩展失败 | 规范允许栈按需扩展，扩展时申请不到内存 | `OOM: unable to create new native thread`（实践中几乎都表现为这行） |
| 线程数过多 | 每线程一份栈：栈总量 ≈ 线程数 × Xss，挤占堆外内存或触到 OS 线程上限 | 同上 |

### 4.3.2 线程失控演示

```java
// OOM 演示：无限起线程，每个还带大栈
// -Xss512k -Xmx64m  （故意把堆调小，让内存压力落在栈上）
public class ThreadOOM {
    public static void main(String[] args) throws InterruptedException {
        for (;;) {
            new Thread(() -> {
                try { Thread.sleep(Long.MAX_VALUE); } catch (Exception ignored) {}
            }).start();
        }
    }
}
// Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
```

注意：这个报错**不是堆问题**——是 OS 层面创建线程失败（栈内存、`ulimit -u`、线程数上限）。处置见 [22_troubleshooting](22_troubleshooting.md)。

## 4.4 -Xss 与递归深度：数量级估算

### 4.4.1 大小参数

```bash
-Xss256k   # 256KB：适合"海量线程 + 浅调用"的服务，换更多线程数
-Xss1m     # 常见默认（Linux x64 JVM 默认 512K~1M 不等，平台相关）
```

### 4.4.2 深度估算公式与数量级表

```text
可容纳帧数 ≈ Xss / 单帧大小
单帧大小 ≈ (max_locals + max_stack) × 4B + 固定开销(数十 B)
```

| -Xss | 空递归方法典型深度（数量级） |
|------|------------------------------|
| 128k | 约 2 千 |
| 512k | 约 8 千 ~ 1 万 |
| 1m   | 约 2 万 ~ 3 万 |

> 数字随 JDK 版本、CPU 架构、方法复杂度明显波动，用来建立"栈大小 ↔ 深度 ↔ 线程数"的比例感，不要当基准值。

# 5. 本地方法栈（Native Method Stack）

## 5.1 作用

服务于 native 方法（C/C++ 实现），同样会抛 StackOverflowError 与 OutOfMemoryError。规范把它与虚拟机栈分开，是为了描述完整性。

## 5.2 HotSpot 的实现：合二为一

HotSpot 直接把本地方法栈和虚拟机栈**合并为同一个栈**，native 调用的帧与 Java 帧交替出现（`jstack` 里可见 `(JVM frame)` / `(Native frame)` 混排），因此 `-Xss` 同时约束两者。

## 5.3 native 方法示例与 JNI

```java
// Thread 类
public static native void yield();
// System 类
public static native void arraycopy(Object src, int srcPos,
        Object dest, int destPos, int length);
```

native 方法通过 JNI 与 JVM 交互，属于执行引擎侧话题，总览见 [1_jvm_overview](1_jvm_overview.md)。

# 6. 堆（Heap）

## 6.1 概览

- JVM 管理的**最大一块**内存，所有线程共享，几乎所有对象与数组的居所。
- GC 调优主战场；经逃逸分析判定不逃逸的对象可栈上分配（见 [15_memory_allocation](15_memory_allocation.md)）。
- 物理上可以不连续（G1 Region / ZGC Page 起，逻辑分代只是标签）。

```text
┌───────────────────────────────┐
│            堆 (Heap)          │
│  ┌─────────────────────────┐  │
│  │       新生代 (Young)     │  │
│  │  ┌─────┬─────┬─────┐    │  │  默认比例
│  │  │Eden │ S0  │ S1  │    │  │  Young : Old = 1 : 2
│  │  │ 8   :  1  :  1  │    │  │  Eden : S0 : S1 = 8 : 1 : 1
│  │  └─────┴─────┴─────┘    │  │
│  └─────────────────────────┘  │
│  ┌─────────────────────────┐  │
│  │       老年代 (Old)       │  │
│  └─────────────────────────┘  │
└───────────────────────────────┘
```

```bash
-Xms512m    # 初始堆大小
-Xmx4g      # 最大堆大小
-Xmn1g      # 新生代大小 (同时决定 eden/s0/s1)
-XX:NewRatio=2         # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8    # Eden:Survivor = 8:1
```

分代结构、晋升规则、字符串常量池等完整内容详见 [6_heap](6_heap.md)。

# 7. 方法区 / 元空间（Method Area / Metaspace）

## 7.1 概念演进一句话版

> 方法区是规范里的逻辑区域；HotSpot 先实现为堆里的**永久代（PermGen）**，JDK 8 起改为本地的**元空间（Metaspace）**——类元数据搬进本地内存，静态变量与字符串常量池早在 JDK 7 就进了堆。

| 对比项 | Java 7 (永久代) | Java 8+ (元空间) |
|--------|----------------|------------------|
| 位置 | JVM 堆内 | 本地内存（Native Memory） |
| 存储内容 | 类元数据、常量池、静态变量 | 类元数据 |
| 大小限制 | -XX:PermSize / -XX:MaxPermSize | -XX:MetaspaceSize / -XX:MaxMetaspaceSize |
| 默认上限 | 固定 | 无上限（受物理内存限制） |
| GC | Full GC 回收 | 类卸载时回收 |
| 字符串常量池 | PermGen | Heap |

方法区存储内容清单：类的类型信息（全限定名、修饰符、父类、接口）、字段与方法信息、运行时常量池、静态变量（JDK 7 后移到堆中的 Class 对象内）、JIT 代码（实际在独立的 Code Cache）。

```bash
-XX:MetaspaceSize=128m     # 首次 GC 的"高水位线"（不是初始大小也不是上限！）
-XX:MaxMetaspaceSize=256m  # 元空间真正的上限
-XX:MinMetaspaceFreeRatio=40  # GC 后最小空闲比例
-XX:MaxMetaspaceFreeRatio=70  # GC 后最大空闲比例
```

Chunk 分配机制、GC 触发链、Code Cache 分段等详见 [7_method_area](7_method_area.md)。

# 8. 直接内存（Direct Memory）

## 8.1 定位

不是 JVM 运行时数据区的一部分，但被 NIO、Netty、mmap 文件、压缩库等频繁使用；分配在**本地内存**，不受 `-Xmx` 管辖，也不出现在 heap dump 里。

## 8.2 NIO DirectByteBuffer 原理图

```text
        堆内（小）                                本地内存（大）
┌─────────────────────┐  address(long字段)  ┌──────────────────────┐
│  DirectByteBuffer    │ ──────────────────▶ │ unsafe.allocateMemory │
│  (对象头+几个字段,    │                     │ 分配的真实缓冲区        │
│   约几十字节)         │                     │ （可达 MB 级）          │
└─────────────────────┘                     └──────────┬───────────┘
                                                       │ 零拷贝读写
                                                       ▼
                                            ┌──────────────────────┐
                                            │  内核 PageCache/网卡   │
                                            └──────────────────────┘

普通 HeapBuffer:  Java数组 → 拷到native临时区 → write() 到内核   （一次额外拷贝）
DirectBuffer:    native缓冲区 直接 write()/read() 到内核         （少一次拷贝）
```

```java
// 分配 100MB 直接内存：堆内只多一个小壳对象
ByteBuffer buffer = ByteBuffer.allocateDirect(100 * 1024 * 1024);
```

## 8.3 大小限制：-XX:MaxDirectMemorySize

### 8.3.1 参数与物理内存关系

| 情形 | 行为 |
|------|------|
| 未设置 | 默认约等于 `-Xmx`（JDK 8/11 常见口径），上限最终还是物理内存/OS 配额 |
| 设置过小 | 高吞吐网络应用频繁触发 Cleaner 回收或 OOM: Direct buffer memory |
| 设置过大 | 堆 + 元空间 + 线程栈 + 直接内存 > 物理内存时，OS 层拒绝分配（`Direct buffer memory` 或 native malloc 失败） |

```bash
-XX:MaxDirectMemorySize=1g
```

### 8.3.2 Netty 池化的约束提示

> Netty `PooledByteBufAllocator` 自管内存池（jemalloc 思路，Arena + Chunk），减少分配/回收抖动；池内存**不走 java.nio.Bits 记账**，`-XX:MaxDirectMemorySize` 对其约束有限，需要配合 Netty 自身参数（如 `-Dio.netty.maxDirectMemory`）。

## 8.4 回收时机

### 8.4.1 Cleaner 回调与分配侧兜底链条

```text
DirectByteBuffer 壳对象不再被引用
        │ 随 Young/Full GC 判定为垃圾
        ▼
进入 ReferenceQueue，触发 Cleaner（虚引用的清理回调）
        ▼
unsafe.freeMemory() 释放本地内存

另一条路：分配新 Buffer 时记账不足
        ▼
Bits.reserveMemory() 内部先 System.gc() 兜底，等待 Cleaner 释放
        ▼
重试仍不足 → throw OOM: Direct buffer memory
```

### 8.4.2 DisableExplicitGC 坑

- 关键坑：`-XX:+DisableExplicitGC` 会屏蔽 System.gc()，直接内存"兜底回收"失效 → 本地内存持续膨胀。替代方案：`-XX:+ExplicitGCInvokesConcurrent`。
- 兜底机制与排查细节见 [15_memory_allocation](15_memory_allocation.md)。

## 8.5 典型现象：堆很小但进程 RSS 很大

### 8.5.1 RSS 构成拆解

```text
-Xmx512m 的应用，top 里 RES 显示 3G：
RSS = 堆(≤512M) + 元空间 + Code Cache + 每线程栈 + 直接内存 + JIT临时区 + JNI分配
```

### 8.5.2 NMT 排查入口

用 NMT 拆解本地内存去向（`-XX:NativeMemoryTracking=summary` 启动），命令与解读详见 [20_jvm_tools](20_jvm_tools.md)。

# 9. 各区域异常与参数速查总表

## 9.1 区域 → 异常 → 参数 → 命令

### 9.1.1 堆、栈与线程

| 区域 | 异常类型 | 相关参数 | 一行定位命令 |
|------|----------|----------|--------------|
| 堆 | `OutOfMemoryError: Java heap space` / `GC overhead limit exceeded` | `-Xms` `-Xmx` `-Xmn` | `jcmd <pid> GC.heap_info` |
| 虚拟机栈 | `StackOverflowError` | `-Xss` | `jstack <pid>` 看最深连续重复帧 |
| 栈总量/线程 | `OOM: unable to create new native thread` | `-Xss`、`ulimit -u`、`vm.max_map_count` | `jstack <pid> \| grep -c '"'` 数线程 |

### 9.1.2 元空间与 Code Cache

| 区域 | 异常类型 | 相关参数 | 一行定位命令 |
|------|----------|----------|--------------|
| 元空间 | `OOM: Metaspace` / `OOM: Compressed class space` | `-XX:MetaspaceSize` `-XX:MaxMetaspaceSize` | `jstat -gcutil <pid> 1000` 看 MU 列 |
| Code Cache | 无异常但 JIT 停摆（`CodeCache is full`） | `-XX:ReservedCodeCacheSize` | 启动日志 / `jcmd <pid> VM.flags \| grep -i codecache` |

### 9.1.3 直接内存与 PC

| 区域 | 异常类型 | 相关参数 | 一行定位命令 |
|------|----------|----------|--------------|
| 直接内存 | `OOM: Direct buffer memory` | `-XX:MaxDirectMemorySize` | `jcmd <pid> VM.native_memory summary`（需开 NMT） |
| PC 寄存器 | 无异常可能 | 无 | — |

# 10. 常见 OOM 场景速查

## 10.1 总表

| 区域 | OOM 信息示例 | 常见原因 |
|------|-------------|----------|
| Heap | `java.lang.OutOfMemoryError: Java heap space` | 对象过多、内存泄漏 |
| Heap | `java.lang.OutOfMemoryError: GC overhead limit exceeded` | GC 占用太多时间但回收很少内存 |
| Metaspace | `java.lang.OutOfMemoryError: Metaspace` | 动态生成类过多（CGLIB、Groovy） |
| Direct Memory | `java.lang.OutOfMemoryError: Direct buffer memory` | NIO 直接内存未释放 |
| Stack/Thread | `java.lang.OutOfMemoryError: unable to create new native thread` | 线程过多，无法分配栈空间 |
| 数组 | `java.lang.OutOfMemoryError: Array size exceeds VM limit` | 数组长度超限（`Integer.MAX_VALUE-8` 量级） |

## 10.2 每类一个最小复现骨架

```java
// ① Java heap space：无限增长的集合
List<byte[]> list = new ArrayList<>();
while (true) list.add(new byte[1024 * 1024]);           // -Xmx20m

// ② GC overhead limit：对象"将满未满"且几乎全是活的
//    （同上骨架 + -Xmx20m，JVM 会先报 overhead 再报 heap space，取决于版本开关）

// ③ Metaspace：每次 new ClassLoader 加载新类（动态代理/字节码框架同理）
GroovyShell shell = new GroovyShell();
while (true) shell.parse("class T" + System.nanoTime() + " {}"); // 详见 7 第 7 节

// ④ Direct buffer memory：分配后不留引用、也不触发清理
while (true) ByteBuffer.allocateDirect(10 * 1024 * 1024); // -XX:MaxDirectMemorySize=64m

// ⑤ unable to create native thread：见 4.3 ThreadOOM
// ⑥ StackOverflowError：见 4.2 StackDepth（注意它不是 OOM）
```

各场景的线上处置流程（dump、MAT、NMT 决策树）详见 [22_troubleshooting](22_troubleshooting.md)。
