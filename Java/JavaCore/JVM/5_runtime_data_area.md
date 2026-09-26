
# 1. 概述

运行时数据区（Runtime Data Area）是 JVM 在执行 Java 程序时分配的内存区域。Java 8 及之后的完整架构：

```
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
│              │              │ │  - 静态变量   │  │
│              │              │ │  - JIT代码缓存 │  │
│              │              │ └───────────────┘  │
└──────────────┴──────────────┴────────────────────┘
```

# 2. 线程私有 vs 线程共享

| 区域 | 线程私有/共享 | 生命周期 | 异常 |
|------|-------------|----------|------|
| 程序计数器 | 私有 | 与线程同生同灭 | 无 |
| 虚拟机栈 | 私有 | 与线程同生同灭 | StackOverflowError / OutOfMemoryError |
| 本地方法栈 | 私有 | 与线程同生同灭 | StackOverflowError / OutOfMemoryError |
| 堆 | 共享 | JVM 启动时创建 | OutOfMemoryError |
| 方法区/元空间 | 共享 | JVM 启动时创建 | OutOfMemoryError |
| 直接内存 | - | - | OutOfMemoryError |

# 3. 程序计数器（PC Register）

- 当前线程执行的字节码行号指示器。
- JVM 中唯一没有 OutOfMemoryError 情况的区域。
- 分支、循环、跳转、异常处理、线程恢复都依赖它。

```
线程1: PC = 0x0005 (正在执行字节码第5条指令)
线程2: PC = 0x0018 (正在执行字节码第24条指令)
```

- 执行 Java 方法时，PC 记录字节码指令地址。
- 执行 native 方法时，PC 值为空（Undefined）。
- 线程切换后能恢复到正确的执行位置，全靠 PC 寄存器。

# 4. 虚拟机栈（Java Virtual Machine Stack）

- 每个方法执行时，JVM 会同步创建一个**栈帧**（Stack Frame）。
- 栈帧存储：局部变量表、操作数栈、动态连接、方法返回值地址。
- 方法调用到执行完毕，对应栈帧的入栈和出栈。

```
方法调用链：
  main()               → 栈帧 main 压栈
    foo()              → 栈帧 foo 压栈
      bar()            → 栈帧 bar 压栈
      bar() 返回       → 栈帧 bar 出栈
    foo() 返回         → 栈帧 foo 出栈
  main() 返回          → 栈帧 main 出栈
```

异常情况：
- **StackOverflowError**：线程请求的栈深度超过 JVM 允许的最大深度（典型：无限递归）。
- **OutOfMemoryError**：栈容量可以动态扩展，但扩展时内存不足（如创建过多线程，每个线程分配栈空间）。

```bash
# 设置栈大小 (默认约 1M)
-Xss256k   # 256KB
-Xss1m     # 1MB
```

# 5. 本地方法栈（Native Method Stack）

- 与虚拟机栈作用类似，但服务于 native 方法（C/C++ 编写的方法）。
- HotSpot 中直接将本地方法栈和虚拟机栈合二为一。
- native 方法调用示例：
```java
// Thread 类
public static native void yield();
// System 类
public static native void arraycopy(Object src, int srcPos, 
        Object dest, int destPos, int length);
```

# 6. 堆（Heap）

- JVM 管理的**最大一块**内存区域，所有线程共享。
- 绝大多数对象实例分配于此（逃逸分析后可能栈上分配）。
- 物理上可以不连续，逻辑上连续。

```
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

**堆参数：**
```bash
-Xms512m    # 初始堆大小
-Xmx4g      # 最大堆大小
-Xmn1g      # 新生代大小 (同时设置 eden/s0/s1)
-XX:NewRatio=2         # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8    # Eden:Survivor = 8:1
```

# 7. 方法区 / 元空间（Method Area / Metaspace）

**Java 8 前后的演变：**

| 对比项 | Java 7 (永久代) | Java 8+ (元空间) |
|--------|----------------|------------------|
| 位置 | JVM 堆内 | 本地内存（Native Memory） |
| 存储内容 | 类元数据、常量池、静态变量 | 类元数据 |
| 大小限制 | -XX:PermSize / -XX:MaxPermSize | -XX:MetaspaceSize / -XX:MaxMetaspaceSize |
| 默认上限 | 固定 | 无上限（受物理内存限制） |
| GC | Full GC 回收 | 类卸载时回收 |
| 字符串常量池 | PermGen | Heap |

- 方法区存储内容：
  - 类的类型信息（全限定名、修饰符、父类、接口）
  - 字段信息、方法信息
  - 运行时常量池
  - 静态变量（Java 7 后移到堆中的 Class 对象内）
  - JIT 编译后的代码缓存

**元空间参数：**
```bash
-XX:MetaspaceSize=128m     # 初始元空间大小，触发 GC 的阈值
-XX:MaxMetaspaceSize=256m  # 最大元空间大小
-XX:MinMetaspaceFreeRatio=40  # GC 后最小空闲比例
-XX:MaxMetaspaceFreeRatio=70  # GC 后最大空闲比例
```

# 8. 直接内存（Direct Memory）

- 不是 JVM 运行时数据区的一部分，但被频繁使用。
- NIO 的 `ByteBuffer.allocateDirect()` 直接在堆外分配内存，避免了 Java 堆和 Native 堆之间的数据复制。
- 受 `-XX:MaxDirectMemorySize` 限制，默认等于 `-Xmx`。
- 直接内存溢出也会导致 OutOfMemoryError，但不会在 heap dump 中直接体现。

```java
// 分配 100MB 直接内存
ByteBuffer buffer = ByteBuffer.allocateDirect(100 * 1024 * 1024);
```

# 9. 运行时数据区参数速查

```bash
# 堆
-Xms<size>          # 初始堆大小
-Xmx<size>          # 最大堆大小
-Xmn<size>          # 新生代大小

# 栈
-Xss<size>          # 线程栈大小

# 元空间
-XX:MetaspaceSize=<size>
-XX:MaxMetaspaceSize=<size>

# 直接内存
-XX:MaxDirectMemorySize=<size>

# 代码缓存（JIT 编译后代码存放处，在方法区外）
-XX:ReservedCodeCacheSize=<size>
```

# 10. 各区域常见 OOM 场景速查

| 区域 | OOM 信息示例 | 常见原因 |
|------|-------------|----------|
| Heap | `java.lang.OutOfMemoryError: Java heap space` | 对象过多、内存泄漏 |
| Metaspace | `java.lang.OutOfMemoryError: Metaspace` | 动态生成类过多（CGLIB、Groovy） |
| Direct Memory | `java.lang.OutOfMemoryError: Direct buffer memory` | NIO 直接内存未释放 |
| Stack | `java.lang.OutOfMemoryError: unable to create new native thread` | 线程过多，无法分配栈空间 |
| GC Overhead | `java.lang.OutOfMemoryError: GC overhead limit exceeded` | GC 占用太多时间但回收很少内存 |