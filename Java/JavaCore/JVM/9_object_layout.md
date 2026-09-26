
# 1. HotSpot 对象内存布局

一个 Java 对象在堆内存中的布局分为三部分：

```
┌──────────────────────────┐
│     对象头 (Header)       │  ← Mark Word + Klass Pointer
├──────────────────────────┤
│     实例数据 (Instance    │  ← 各字段数据
│     Data)                │
├──────────────────────────┤
│     对齐填充 (Padding)    │  ← 保证对象大小是 8 字节的倍数
└──────────────────────────┘
```

# 2. 对象头（Object Header）

对象头包含两部分：

## 2.1 Mark Word（标记字段）

- 运行时数据，如哈希码、GC 分代年龄、锁状态标志、偏向锁信息。
- 32 位 JVM 占 4 字节，64 位 JVM 占 8 字节。

**64 位 JVM Mark Word 结构（简化）：**

| 锁状态 | 25bit | 31bit (见说明) | 1bit | 4bit | 1bit(偏向) | 2bit(锁标志) |
|--------|-------|----------------|------|------|------------|-------------|
| 无锁 | unused | hashCode(31) | unused | 分代年龄 | 0 | 01 |
| 偏向锁 | ThreadID(54) | Epoch(2) | unused | 分代年龄 | 1 | 01 |
| 轻量级锁 | 指向栈中锁记录的指针(62) |  |  |  |  | 00 |
| 重量级锁 | 指向重量级锁的指针(62) |  |  |  |  | 10 |
| GC标记 | 空 |  |  |  |  | 11 |

锁标志位（最后 2 bit）：
- `01`：无锁或偏向锁
- `00`：轻量级锁
- `10`：重量级锁
- `11`：GC 标记

## 2.2 Klass Pointer（类型指针）

- 对象指向它的类元数据的指针，JVM 通过它确定对象是哪个类的实例。
- 32 位 JVM 占 4 字节，64 位 JVM 占 8 字节（开启压缩指针后为 4 字节）。

## 2.3 数组长度

- 只有数组对象才有这个部分，4 字节。
- 因为 JVM 需要知道数组长度，而普通对象通过元数据确定大小。

# 3. 实例数据（Instance Data）

- 真正存放对象字段信息的地方。
- 不管是父类的还是自身的字段，都存储在此。
- 字段分配顺序受 JVM 分配策略影响：
  - 相同宽度的字段分配在一起
  - 父类字段在子类字段之前
  - `-XX:FieldsAllocationStyle=0/1` 控制策略

**HotSpot 默认字段排序：**
1. doubles / longs (8 字节)
2. ints / floats (4 字节)
3. shorts / chars (2 字节)
4. bytes / booleans (1 字节)
5. oops (Ordinary Object Pointers，引用类型)

这样排序可以减少对齐填充造成的空间浪费。

# 4. 对齐填充（Padding）

- HotSpot 要求对象起始地址必须是 8 字节的整数倍。
- 如果对象实例数据部分不够 8 字节对齐，就填充补齐。
- 目的：CPU 访问内存时最高效读 8 字节对齐数据。

```java
public class A {
    int a;       // 4 字节
    byte b;      // 1 字节
}
// 对象头(Mark Word + Klass Pointer) + 4 + 1 + 3(填充) = 对齐到 8 的倍数
```

# 5. 实战：JOL 分析对象布局

JOL（Java Object Layout）是 OpenJDK 的工具，可直接分析对象内存。

```java
import org.openjdk.jol.info.ClassLayout;

public class JOLDemo {
    public static void main(String[] args) {
        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
}
```

**输出示例：**
```
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION               VALUE
      0     4        (object header)           01 00 00 00 (00000001 ...)
      4     4        (object header)           00 00 00 00 (00000000 ...)
      8     4        (object header)           e5 01 00 f8 (11111001 ...)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

解读：
- OFFSET 0-3：Mark Word 第 1 部分（4B）
- OFFSET 4-7：Mark Word 第 2 部分（4B）
- OFFSET 8-11：Klass Pointer（4B，因为开启了压缩指针）
- OFFSET 12-15：对齐填充
- 一个空的 `Object` 占 16 字节（64 位，压缩指针开启）

对比关闭压缩指针：
```
# -XX:-UseCompressedOops
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION               VALUE
      0     4        (object header)           ...
      4     4        (object header)           ...
      8     4        (object header)           ...
     12     4        (object header)           ...
     16     8        (loss due to alignment)
Instance size: 24 bytes
```

关闭压缩指针后，空 Object 占 24 字节（Mark Word 8B + Klass Pointer 8B + 填充 8B）。

# 6. 压缩指针（Compressed Oops）

- "oop" = Ordinary Object Pointer（普通对象指针）。
- 64 位 JVM 中，对象引用本来占 8 字节，压缩后仅占 4 字节。
- 通过将堆地址右移 3 位实现压缩（因为对象 8 字节对齐，低 3 位始终为 0）。

```bash
# 默认开启（堆 < 32GB 时）
-XX:+UseCompressedOops        # 压缩对象指针

# 压缩类指针（JDK 8+ 默认开启，前提是 UseCompressedOops 已开启）
-XX:+UseCompressedClassPointers
```

- 压缩指针生效条件：堆内存 < 32GB（约 32G，因为 2^32 * 8 = 32GB）。
- 超过 32GB 的堆，压缩指针自动失效，每个引用变 8 字节，内存开销显著增加。

```
32GB 堆以下：
  引用 4B → 实际地址 = 引用 << 3
  
32GB 堆以上：
  引用 8B → 实际地址 = 引用（不压缩）
```

# 7. 对象的内存计算实例

```java
public class User {
    private int id;          // 4B
    private String name;     // 4B (压缩后)
    private int age;         // 4B
    private byte gender;     // 1B
    // 通过 JOL 分析：
    // Object Header: 12B (Mark Word 8B + Klass 4B)
    // Fields: id(4) + age(4) + name(4) + gender(1) = 13B
    // total = 12 + 13 = 25 → padding to 32B
    // Instance size: 32 bytes
}
```

# 8. 对象的访问定位

JVM 规范只规定栈中引用指向对象，但未规定实现方式。HotSpot 主要使用**直接指针访问**。

## 8.1 方式一：句柄访问（间接指针）

```
栈 (reference) → ┌──────────────┐
                 │   句柄池      │
                 │ ┌──────────┐ │    ┌──────────┐
                 │ │ 对象指针  │─┼───→│ 实例数据  │
                 │ │ 类型指针  │─┼─┐  └──────────┘
                 │ └──────────┘ │ │  ┌──────────┐
                 └──────────────┘ └─→│ 类型数据  │
                                    └──────────┘
```

- 优点：对象移动（GC 时）只需改句柄池，不改变 reference。
- 缺点：多一次间接访问，速度慢。

## 8.2 方式二：直接指针访问（HotSpot 使用）

```
栈 (reference) ──→ ┌──────────────────────────┐
                   │     对象实例               │
                   │ ┌──────────────────────┐  │
                   │ │  Klass Pointer       │──┼──→ 方法区/元空间中的类型数据
                   │ ├──────────────────────┤  │
                   │ │  实例数据             │  │
                   │ └──────────────────────┘  │
                   └──────────────────────────┘
```

- 优点：一次指针跳转，访问快。
- 缺点：对象被移动时（GC），需要修改 reference 本身。

# 9. 对象创建的详细步骤

```
1. 遇到 new 指令
   ↓
2. 检查类是否已加载、解析、初始化
   ↓ (若未加载则先加载类)
3. 分配内存
   - 指针碰撞 (Bump the Pointer)：Serial, ParNew 等带 compact 的收集器
   - 空闲列表 (Free List)：CMS 等基于 sweep 的收集器
   ↓
4. 初始化零值
   - 将分配到的空间全部置零（不包括对象头）
   ↓
5. 设置对象头
   - 设置 Mark Word、Klass Pointer、哈希码（延迟计算）
   ↓
6. 执行 <init>() 方法
   - 开发者写的构造函数
```

**内存分配并发安全：**
- **CAS + 失败重试**：乐观锁方式。
- **TLAB**：线程本地分配缓冲，更高效（默认开启）。
