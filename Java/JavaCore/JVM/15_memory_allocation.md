
# 1. 对象优先在 Eden 分配

- 大多数新对象在 Eden 区分配。
- Eden 区满 → Minor GC。
- 存活对象进入 Survivor 区，熬过多次 GC 后进入老年代。

```java
// 验证对象在 Eden 分配
// -Xms20m -Xmx20m -Xmn10m -XX:SurvivorRatio=8
// -XX:+PrintGCDetails -XX:+UseSerialGC
public class EdenAllocation {
    private static final int _1MB = 1024 * 1024;
    
    public static void main(String[] args) {
        byte[] a1 = new byte[2 * _1MB];
        byte[] a2 = new byte[2 * _1MB];
        byte[] a3 = new byte[2 * _1MB];
        byte[] a4 = new byte[4 * _1MB];  // 触发 Minor GC
    }
}
```

# 2. 大对象直接进入老年代

- 大对象（特别是生命周期短的）对 JVM 是坏消息。
- 复制开销大，容易产生碎片。
- 超过阈值的大对象直接在老年代分配（仅对 Serial/ParNew 有效）。

```java
// -XX:PretenureSizeThreshold=3145728  (3MB)
// 仅对 Serial 和 ParNew 有效
byte[] large = new byte[4 * 1024 * 1024];  // 4MB，直接进老年代
```

```bash
-XX:PretenureSizeThreshold=3m   # 超过 3MB 直接进老年代
```

- 注意：这个参数只对 Serial GC 和 ParNew 有效，G1 由 Humongous Region 管理大对象。

# 3. 长期存活对象进入老年代

- 对象在 Survivor 区每熬过一次 Minor GC，年龄（Age）+1。
- 年龄达到 `-XX:MaxTenuringThreshold`（默认 15）→ 晋升到老年代。

```bash
-XX:MaxTenuringThreshold=15    # 默认 15（CMS 下为 6）
```

**对象年龄计数器**
- 存在对象的 Mark Word 中，占 4 位 → 最大值 = 15。

# 4. 动态年龄判定

- HotSpot 不严格按 `MaxTenuringThreshold` 晋升。
- **如果 Survivor 空间中相同年龄的对象大小之和 > Survivor 空间的一半，则年龄 >= 该年龄的对象都可直接晋升。**

```
示例：Survivor 区 100MB
年龄1: 30MB
年龄2: 30MB    ← 30+30=60 > 100/2
→ 年龄 >= 2 的所有对象晋升到老年代
```

```bash
# 查看年龄分布
-XX:+PrintTenuringDistribution
```

输出示例：
```
Desired survivor size 5242880 bytes, new threshold 2 (max 15)
- age 1: 3145728 bytes, 3145728 total
- age 2: 2097152 bytes, 5242880 total
- age 3: 1048576 bytes, 6291456 total
```

# 5. 空间分配担保（Handle Promotion Failure）

- Minor GC 之前，JVM 会检查老年代剩余空间是否够存放新生代所有对象。
- 如果不够，检查是否有分配担保：
  - 之前每次 Minor GC 进入老年代的**平均大小** < 老年代剩余空间 → 冒险执行 Minor GC。
  - 否则 → 先执行 Full GC。

```bash
# JDK 7+ 默认开启
-XX:+HandlePromotionFailure
```

# 6. 栈上分配（Stack Allocation）

- JIT 编译器经过**逃逸分析**发现对象不会逃逸出方法 → 可以在栈上分配。
- 对象随栈帧出栈而销毁，**减轻 GC 压力**。

```java
// 不会逃逸的对象，可能在栈上分配
public void test() {
    Point p = new Point(1, 2);  // p 只在这个方法内使用
    int area = p.x * p.y;
}

// 会逃逸的对象，必须在堆上分配
public Point createPoint() {
    return new Point(1, 2);     // 返回后外部仍可引用
}
```

# 7. 逃逸分析（Escape Analysis）

- JDK 6u23+ 默认开启。
- 分析对象的动态作用域。
- 如果对象不逃逸 → 可以进行优化。

```bash
-XX:+DoEscapeAnalysis        # 开启逃逸分析（默认）
-XX:-DoEscapeAnalysis        # 关闭
-XX:+PrintEscapeAnalysis     # 打印逃逸分析结果（需要 fastdebug 版 JDK）
```

**逃逸分析的三种优化：**

## 7.1 栈上分配
- 对象不逃逸 → 在栈上分配，GC 无需回收。

## 7.2 标量替换（Scalar Replacement）

- 如果对象不逃逸，可以不创建对象，直接把对象的成员变量当作局部变量。

```java
// 原始代码
class Point { int x; int y; }
public int area() {
    Point p = new Point();
    p.x = 1; p.y = 2;
    return p.x * p.y;
}

// 经过标量替换后等效于：
public int area() {
    int x = 1;
    int y = 2;
    return x * y;
}
// 连对象都不创建了，完全不产生 GC
```

```bash
-XX:+EliminateAllocations    # 开启标量替换（默认）
```

## 7.3 同步消除（Lock Elimination）

- 如果对象不逃逸，JIT 可以消除其上的同步锁。

```java
public String concat() {
    StringBuffer sb = new StringBuffer();  // 只在方法内使用
    sb.append("hello");                    // append 是 synchronized 的
    sb.append("world");
    return sb.toString();
}
// 经过同步消除：append 的 synchronized 被移除
```

```bash
-XX:+EliminateLocks           # 开启同步消除（默认）
```

# 8. TLAB 详解

- 每个线程在 Eden 中拥有独占的一小块内存（TLAB）。
- 线程在自己的 TLAB 内分配对象无需同步。
- TLAB 用完时，需分配新的 TLAB 或直接在 Eden 共享区分配（加锁）。

```bash
-XX:+UseTLAB                 # 启用 TLAB（默认开启）
-XX:TLABSize=64k             # TLAB 初始大小
-XX:-ResizeTLAB              # 禁止动态调整 TLAB 大小
-XX:+PrintTLAB               # 打印 TLAB 统计信息
```

**TLAB 分配流程伪代码：**
```java
boolean allocate(Object obj) {
    if (tlab.hasSpace(obj.size)) {
        tlab.allocate(obj);       // 快速路径，无锁
        return true;
    } else {
        // TLAB 满了，分配新 TLAB 或回退到 Eden 共享区域分配
        return allocateNewTLABOrFallback(obj);
    }
}
```

# 9. 内存分配实战优化

## 9.1 减少不必要的对象创建

```java
// 差：循环中频繁创建对象
for (int i = 0; i < 10000; i++) {
    String s = new String("prefix_" + i);   // 每次 new 新对象
}

// 好：复用或延迟创建
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.setLength(0);
    sb.append("prefix_").append(i);
}

// 好：基本类型优先
Integer boxed = 10;       // 自动装箱，创建新对象
int primitive = 10;       // 无对象创建
```

## 9.2 合理设置大对象阈值

```java
// 哪些场景适合直接进老年代？
// 1. 静态大缓存对象（生命周期长）
static final byte[] CACHE = new byte[10 * 1024 * 1024];

// 2. 一次性读取的大文件
byte[] fileData = Files.readAllBytes(Paths.get("large_file.bin"));
```

## 9.3 逃逸分析验证

```java
// -XX:+PrintEscapeAnalysis (fastdebug JDK)
public class EscapeTest {
    private int value;
    
    // 不会逃逸 → 栈上分配 + 标量替换
    public int safeAdd(int a, int b) {
        EscapeTest t = new EscapeTest();
        t.value = a + b;
        return t.value;
    }
    
    // 会逃逸 → 堆上分配
    public static EscapeTest instance;
    public void escape() {
        instance = new EscapeTest();  // 赋值给静态变量，逃逸
    }
}
```

# 10. 内存分配相关参数速查

```bash
# TLAB
-XX:+UseTLAB                   # 开启 TLAB
-XX:TLABSize=64k               # TLAB 大小
-XX:TLABWasteTargetPercent=1   # TLAB 浪费空间目标占比

# 大对象
-XX:PretenureSizeThreshold=3m  # 直接进老年代的大小（仅 Serial/ParNew）

# 晋升策略
-XX:MaxTenuringThreshold=15    # 最大晋升年龄
-XX:TargetSurvivorRatio=90     # Survivor 占用目标比例（动态年龄调整用）

# 逃逸分析
-XX:+DoEscapeAnalysis          # 逃逸分析
-XX:+EliminateAllocations      # 标量替换
-XX:+EliminateLocks            # 同步消除

# 分配担保
-XX:+HandlePromotionFailure    # 分配担保（JDK 7+ 默认）
```