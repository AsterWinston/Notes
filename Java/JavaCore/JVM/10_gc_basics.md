# 1. 垃圾回收要解决的问题

| # | 问题 | 本质 | 由谁回答 |
|---|------|------|----------|
| 1 | 哪些对象是垃圾？ | 对象存活判定 | 第 2、3、4 节（可达性分析 + GC Roots + 引用类型） |
| 2 | 何时回收？ | GC 触发时机与分类 | 第 7 节（Young / Mixed / Full 的触发条件） |
| 3 | 如何回收？ | 收集算法与实现者 | 第 8、9 节（STW / 安全点）；算法见 [11_gc_algorithm](11_gc_algorithm.md)，收集器见 [12_gc_collectors](12_gc_collectors.md) |

```text
                          垃圾回收
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
    回收什么?             何时回收?            怎么回收?
    可达性分析            Young/Mixed/Full     复制/清除/整理
    引用类型              触发条件              三色标记
    (第 2~4 节)           (第 7 节)             (第 11/12 篇)
```

> 与 C/C++ 手动管理对照：Java 把 free 的责任移交给 JVM，代价是**回收时机不可控**与运行时开销，收益是消灭悬垂指针、double free 这类错误。"自动"并不等于"不用管"——时机与停顿正是 [14_gc_tuning](14_gc_tuning.md) 的全部内容。

# 2. 对象存活判断

## 2.1 引用计数法

### 2.1.1 原理与优缺点

- 每个对象维护一个引用计数器，被引用 +1，引用失效 -1，计数器为 0 即可回收。
- **优点**：判定是局部的，无需全局扫描；回收**及时**（计数归零立刻释放），停顿天然碎片化。
- **缺点**：
  - 每次赋值都要维护计数（两条附加指令 + 跨线程时的原子操作），分配/释放热点路径变慢；
  - 无法处理循环引用（见下）；
  - 计数器位宽有限，溢出处理麻烦。

### 2.1.2 循环引用死结

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

```text
   ┌────────┐  ref   ┌────────┐
   │   a    │──────▶│   b    │
   └────────┘        └────────┘
        ▲                 │
        └────── ref ──────┘
   a=null; b=null 之后两对象计数仍为 1 —— 永远无法归零
```

- **Python 同样以引用计数为主**，但额外配备分代**标记-清除**辅助阶段来断开循环引用——可见纯引用计数在通用语言运行时里独木难支。
- **HotSpot 不使用引用计数法**；历史上只有极少数商用实现（如 BEA JRockit Real Time）采用过"根追踪 + 引用计数"变体。

## 2.2 可达性分析（Reachability Analysis）

### 2.2.1 基本思想

- 以一系列 **GC Roots** 为起点，沿着引用链向下搜索，搜索路径称为**引用链**（Reference Chain）。
- 搜索不到的对象判定为**不可达**，可以被回收。

```text
GC Roots:
  ● (栈中引用)
  │
  │  可达对象
  ▼  (不会被回收)
  ○ ──→ ○ ──→ ○
  
  ○ (不可达) ← 将被回收
```

### 2.2.2 遍历算法本质：从根集出发的图搜索

- 对象-引用关系是一张**有向图**（可能有环），可达性分析 = 从根集出发的 DFS/BFS 遍历。
- 有环 → 必须记录"已访问"标记，否则无限循环；HotSpot 把标记存在对象头的 **Mark Word**（详见 [9_object_layout](9_object_layout.md)）。
- 每个对象最多访问一次 ⇒ 遍历成本 ∝ **可达对象数（活跃集）**，与垃圾数量无关——这是"存活对象越少 GC 越快"的根因。

### 2.2.3 HotSpot 不逐次全堆扫描：根扫描 + 记忆集

```text
天真做法：每次 Young GC 都从全堆任意对象出发遍历 → 成本 ∝ 整个堆 ❌
HotSpot： Young GC 的"根" = 传统 GC Roots
           ∪ 卡表命中的老年代对象（老→新引用的持有者，视为临时根）
         → 成本 ∝ 新生代大小，老年代只碰"脏卡"那几 KB   ✔
```

- 跨代引用极少（跨代引用假说，见 [6_heap](6_heap.md) 1.3.3），为少数引用扫描全堆不划算。
- 记忆集 / 卡表的完整机制详见 [11_gc_algorithm](11_gc_algorithm.md) 第 5 节。

### 2.2.4 枚举根必须 STW 的原因

- 运行中的线程不断进出方法：栈帧在 push/pop，局部槽位、寄存器里的引用随时被覆盖，存在"引用半更新"的中间态。
- 在中间态扫描栈 → 得到**错误根集** → 要么漏标（活对象被回收，程序崩溃），要么多标。
- 因此**根枚举阶段必须把程序冻结**在线程都停在"能精确定位所有引用"的位置——即安全点（第 9 节）。
- 注意范围：只有碰对象图的阶段必须 STW；CMS/G1 的**并发标记阶段**程序照常运行，引用变化靠写屏障补偿（见 11 篇第 8 节）。

# 3. GC Roots 有哪些

## 3.1 总表

| GC Root | 为什么能当根 | 示例 |
|---------|-------------|------|
| 虚拟机栈中引用的对象 | 线程活着 ⇒ 局部变量表槽位随时可被使用 ⇒ 必须存活 | 方法内 `new` 的对象 |
| 方法区中静态属性引用的对象 | 类已加载 ⇒ 从类出发是**静态可达**的 | `static Object obj = new Object()` |
| 方法区中常量引用的对象 | 被 StringTable 等运行时结构直接持有 | `static final String S = "abc"` (S 在池中) |
| 本地方法栈中 JNI 引用的对象 | Native 层拿着指针，Java 层看不到使用链，只能当根保命 | `jobject` 全局/局部引用 |
| JVM 内部引用 | 虚拟机自身运转依赖 | 基本类型 Class 对象、常驻异常（如预分配 OOM）、系统类加载器、main 线程组 |
| 被同步锁(synchronized)持有的对象 | 持有者随时可能唤醒并解锁/notify，对象不可消失 | 锁住的对象、`wait()` 队列中的对象 |
| 尚未完成 finalize 的"活跃"对象 | finalize() 可能把它复活 | F-Queue 中待处理对象（见第 5 节） |
| JVM 内部的 JMXBean、JVMTI 回调等 | 被 VM 级子系统强持有 | 注册的 MXBean |

> 根集**随 GC 类型变化**：Young GC 只关心"能触及新生代"的根（栈 + 卡表命中对象）；带类卸载的 GC 还要把 JNI Handle、系统类加载器触达的对象算进根（链接 [3_class_lifecycle](3_class_lifecycle.md)）。

## 3.2 JIT 内联之后，根从哪里来（OopMap）

```text
解释执行：按栈帧逐帧走，"局部变量表中引用类型槽位"位置固定 → 直接扫
JIT 编译：局部变量可能进了寄存器、被复用、被标量替换，局部变量表已不存在
           → C2 在编译时于每个安全点预生成 OopMap：
             "本安全点上，哪些寄存器/栈槽持有对象引用"
           → GC 枚举根 = 查 OopMap，而不是解析字节码
           方法内联后：被内联方法的"虚拟帧"消失，其引用并入调用者的 OopMap；
           需要反优化(deopt)时再恢复出虚拟帧结构
```

- 所以"安全点"不只是"能停的位置"，还是"VM 恰好知道引用在哪的位置"——这与 3.1 的根定义互为表里。
- 逃逸分析把对象整个消掉时，它连堆都不在，自然也不是根（见 [17_jit_optimization](17_jit_optimization.md)）。

## 3.3 卡表命中：老年代对象成为"临时根"

- Young GC 时，被脏卡覆盖的老年代对象会被当作根扫描，用于发现"老 → 新"引用。
- 这些对象**只是引用来源**，本身不在本次回收范围，不会被移动或回收。
- 机制与代价分析详见 [11_gc_algorithm](11_gc_algorithm.md) 第 5 节。

# 4. 引用类型

Java 中的引用分为 4 种，从强到弱。

## 4.1 四引用总表

| 引用 | 回收时机 | 能否配 ReferenceQueue | get() 结果 | 典型用途 |
|------|----------|----------------------|-----------|----------|
| 强 Strong | 引用不消失永不回收 | — | 恒非 null | 普通代码 |
| 软 Soft | **内存不足**时才回收 | 可选 | 回收后 null | 内存敏感缓存 |
| 弱 Weak | 下一次 GC 即回收 | 可选 | 回收后 null | WeakHashMap、ThreadLocal key |
| 虚 Phantom | 任意时刻，仅保证发生在内存回收**之前** | **必须** | **永远 null** | 堆外资源清理通知 |

## 4.2 强引用（Strong Reference）

- 最常见的引用：`Object obj = new Object();`
- 只要强引用还存在，GC 绝不会回收——**内存泄漏泄漏的几乎都是强引用**。

## 4.3 软引用（Soft Reference）

### 4.3.1 语义与基本用法

- 有用但非必需的对象；在 OOM 之前，GC 会回收软引用指向的对象，适合做**内存敏感缓存**。

```java
SoftReference<byte[]> softRef = new SoftReference<>(new byte[10 * 1024 * 1024]);
byte[] data = softRef.get();   // 可能返回 null（已被回收）
```

### 4.3.2 回收时机的公式：SoftRefLRUPolicyMSPerMB

"内存不足才回收"太模糊，HotSpot 实际按公式裁决（简化）：

```text
若 now - 上次访问时间 > SoftRefLRUPolicyMSPerMB × 当前空闲堆(MB)
    → 清除该软引用
默认 = 1000 ms/MB：空闲 1GB 堆时，软引用最后一次被访问后还能"活"约 1000 秒
→ 默认值下软引用极难被清，更像兜底而不是缓存淘汰策略
```

```bash
-XX:SoftRefLRUPolicyMSPerMB=0   # 激进：一旦有内存压力/Full GC 即清除
```

### 4.3.3 软引用缓存骨架

```java
class SoftCache<K, V> {
    private final Map<K, SoftReference<V>> map = new ConcurrentHashMap<>();
    void put(K k, V v)          { map.put(k, new SoftReference<>(v)); }
    V get(K k) {
        SoftReference<V> ref = map.get(k);
        V v = (ref == null) ? null : ref.get();   // 注意二次判空
        if (v == null && ref != null) map.remove(k);
        return v;                                    // null → 调用方回源
    }
}
// 生产环境更常用 Caffeine 的 Window-TinyLFU + 权重淘汰；SoftReference 缓存
// 的问题是淘汰时机由 VM 公式决定，不可预期，命中率难以规划
```

## 4.4 弱引用（Weak Reference）

### 4.4.1 语义与示例

- 非必需对象，比软引用更弱：**无论内存是否充足**，只要 GC 扫到就回收。

```java
WeakReference<Object> weakRef = new WeakReference<>(new Object());
System.gc();
Object obj = weakRef.get();  // 可能返回 null
```

### 4.4.2 WeakHashMap

```java
WeakHashMap<Object, String> map = new WeakHashMap<>();
Object key = new Object();
map.put(key, "value");

key = null;           // key 的唯一强引用消失
System.gc();          // 触发 GC
// map 中的该条目被自动清除
```

- 两个经典坑：
  1. 条目清理发生在**访问 Map 时**（get/containsKey 顺带回收），不访问则占着；
  2. value 若强引用了 key（如 `put(key, key.toString相关对象含key)`），key 永不可达引用消失不了 → 弱引用形同虚设。

### 4.4.3 ThreadLocalMap.Entry 继承 WeakReference

- `ThreadLocalMap.Entry extends WeakReference<ThreadLocal<?>>`：**key（ThreadLocal 对象）是弱引用，value 是强引用**。
- 后果：外部不再引用某 ThreadLocal 后，key 被 GC 置 null，但 value 仍被 Entry→线程→栈 强链拖住 → **不 `remove()` 就是泄漏**（线程池里线程长驻，泄漏累积）。
- 排查方法（MAT 最短路径）见 [22_troubleshooting](22_troubleshooting.md)。

## 4.5 虚引用（Phantom Reference）

### 4.5.1 语义：get() 永远 null，必须配 ReferenceQueue

- 完全不影响对象生命周期；唯一作用：**对象被回收时收到一个系统通知**。
- `new PhantomReference<>(x, null)` 无意义——没有队列就收不到通知，所以队列是**构造必填**；`get()` 恒返回 null，也无法借它复活对象。

```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(new Object(), queue);
// 对象被回收后，phantom 会被加入 queue
```

### 4.5.2 案例：DirectByteBuffer + Cleaner 的堆外回收

```text
DirectByteBuffer(Java 堆内)                          堆外内存(native)
┌─────────────────────┐        clean() → free()      ┌───────────────┐
│ address ────────────┼─────────────────────────────▶│ malloc 块      │
│ cleaner: Cleaner ───┼── 本质是 PhantomReference ── │               │
└─────────────────────┘     + ReferenceQueue          └───────────────┘
GC 判 DirectByteBuffer 不可达
   → Cleaner 入 ReferenceQueue
   → Reference Handler 线程取出并执行 clean() → free() 归还堆外内存
```

- JDK ≤ 8 用 `sun.misc.Cleaner`；JDK 9 起改为 `jdk.internal.ref.Cleaner`，并公开 `java.lang.ref.Cleaner` API——**官方立场：Cleaner 取代 finalize 做资源清理**（终结器线程机制保留给 finalize，但新代码不应再写 finalize）。
- 若堆外空间不足，`Bits.reserveMemory` 会主动调 `System.gc()` 逼这条链运转——这正是第 7.4 节警告 `-XX:+DisableExplicitGC` 的原因。堆外区域本身见 [5_runtime_data_area](5_runtime_data_area.md)。

# 5. finalize()

- `finalize()` 是 `Object` 的 protected 方法，GC 前可能被调用。
- **已被 JDK 9 标记为 `@Deprecated`，不推荐使用**（JDK 18 进一步通过 JEP 421 标记"计划移除"）。
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
4. JDK 9 deprecated，JDK 18（JEP 421）标记为待移除

## 5.1 对象复活流程

```text
对象即将被回收（第一次标记，判定可回收）
        │ 不入死亡名单，而是放入 F-Queue（待终结队列）
        ▼
   Finalizer 线程逐个执行 finalize()
        ├─ 方法里把自己赋给某个 GC Roots 可达的引用 ──▶ 复活：本轮不回收，
        │      但被打上"已终结"标记，第二次死亡时【不再执行 finalize】
        └─ 没有自救 ────────────────────────────────▶ 下一轮 GC 真正回收
```

- 确定规则（可背）：**finalize 至多执行一次；对象至多复活一次；第二次被判定回收时直接回收，不再调用 finalize**。
- 复活验证的经典套路：`static` 字段接住 `this`，配合 `System.gc()` + 观察输出（面试代码，生产禁用）。

## 5.2 代价清单

| 代价 | 机制 |
|------|------|
| 回收要"等两轮 GC" | 第一轮只标记 + 入队，对象多活一个 GC 周期，占用堆空间 |
| 终结可能永远追不上 | F-Queue 由**单线程**串行处理，finalize 本身可以任意慢/任意卡（阻塞 IO），队列积压 |
| 吞异常 | finalize 抛出的异常被 VM 忽略并打印，问题极难暴露（见 [22_troubleshooting](22_troubleshooting.md)） |
| 拖累优化 | 带 finalize 的对象无法参与部分逃逸分析等优化，且必须注册终结器，分配变贵 |

## 5.3 替代方案

```java
// 1. AutoCloseable + try-with-resources：清理时机确定、编译期强制
try (var res = new MyResource()) { ... }

// 2. Cleaner（JDK 9+）：虚引用 + 队列，注册的对象【不得引用】业务对象本身
private static final Cleaner CLEANER = Cleaner.create();
private static class State implements Runnable { /* 只放纯 native 句柄 */ }
final Cleaner.Cleanable cleanable = CLEANER.register(this, new State(handle));

// 3. 显式 close() + finally 兜底：老代码的最稳妥写法
```

- 共同点：把"资源释放"与"内存回收"解耦——**资源清理靠确定性代码，内存回收才交给 GC**，不要拿 finalize 当资源析构器。

# 6. 方法区的回收

方法区（元空间）也能被回收，但条件苛刻。

## 6.1 回收对象：废常量与无用类

- **废弃的常量**：字符串常量池 / 符号池中已无人引用的常量（注意：池在堆里的部分 Young GC 也能触达，但类元数据相关的回收通常发生在 Full GC / 并发标记周期）。
- **不再使用的类**（判定标准见 6.2）。
- 永久代时代（JDK ≤ 7）这块回收几乎"看缘分"——回收效率极差是移除永久代的直接动因之一（见 [6_heap](6_heap.md) 6.1、[7_method_area](7_method_area.md)）。

## 6.2 类卸载的三个条件（同时满足）

1. 该类所有实例已被回收；
2. 加载该类的**类加载器**已被回收；
3. 该类的 Class 对象没有任何地方引用。

- 条件 2 是分水岭：应用自带类加载器（AppClassLoader）永远活着 ⇒ 应用类**事实上永不卸载**；只有动态生成类、热部署（OSGi、Groovy、Tomcat 重部署）才会真正制造"可死类加载器"。类加载器与双亲委派见 [3_class_lifecycle](3_class_lifecycle.md)，元空间结构见 [7_method_area](7_method_area.md)。

## 6.3 为什么大流量应用建议显式设 MaxMetaspaceSize

```text
不设上限：默认上限≈可用物理内存 → 元空间几乎"永远不满"
   → 缺少"满了触发 GC"这一自然回收时机
   → 动态代理/Groovy 脚本类加载器泄漏时：
      只表现为 RSS 持续上涨，不抛 Java OOM，最后被 OS/容器 OOM-Kill，极难定位

设了上限：用量逼近阈值 → 触发 "Metaspace (expansion) GC threshold reached" Full GC
   → 类卸载获得执行机会；仍回收不动 → 干净地抛
      java.lang.OutOfMemoryError: Metaspace，dump 可见
```

- JDK 8 的 CMS 组合还需要 `-XX:+CMSClassUnloadingEnabled`（默认关）才会并发卸载类；G1 默认即可。参数细节见 [12_gc_collectors](12_gc_collectors.md) 与 [7_method_area](7_method_area.md)。

# 7. GC 类型

## 7.1 分类总表

| GC 类型 | 回收区域 | 触发条件 | 停顿量级（经验值） |
|---------|---------|----------|--------------------|
| Young GC / Minor GC | 新生代（Eden + Survivor） | Eden 空间不足 | 10 ~ 100 ms，极频繁 |
| Major GC | 老年代 | 老年代空间不足（通常伴随 Minor GC；CMS 时代概念，边界随实现漂移） | 数百 ms ~ 秒 |
| Full GC | 整堆 + 方法区/元空间 | 见 7.3 | 数百 ms ~ 数秒+，**最贵** |
| Mixed GC | 整个新生代 + 部分老年代（G1） | 并发标记结束后分批进行 | 与 Young GC 同量级 |
| Partial GC | 堆的一部分（非整堆） | JVM 规范新术语（见 7.2） | 视范围而定 |

- **Minor GC 非常频繁**，但速度快；**Full GC 尽量避免**；G1 的 Mixed GC 介于两者之间。
- 停顿大小与"存活集大小"正相关（2.2.2 的遍历成本模型）。

## 7.2 JDK 12 起 JVM 规范补充的 GC 术语

| 术语 | 含义 |
|------|------|
| Full GC | 回收堆的全部区域（+通常含元空间） |
| Partial GC | **部分堆**的回收，Minor/Major/Mixed 都是它的特例 |
| Regions Collect GC | 回收"任意一组内存区域"，Mixed GC 属于此类 |
| Incipient GC | 只收新生代且**不发生晋升**（如 Eden 全死时） |

> 背景：术语由 JEP 333/364 时代的 ZGC、Shenandoah 推动写进规范——非分代收集器出现后，旧分类（Minor/Major/Full）不够用了。

## 7.3 触发 Full GC 的常见场景

1. `System.gc()` 的显式调用
2. 老年代空间不足
3. 元空间/永久代空间不足（或达到 GC 阈值）
4. CMS GC 时的并发模式失败（Concurrent Mode Failure）
5. Minor GC 时的分配担保失败（Handle Promotion Failure）
6. 统计意义上的"晋升空间不足"：动态年龄/大对象导致老年代提前告急（见 [6_heap](6_heap.md) 第 3、5 节）

## 7.4 System.gc() 与两个互斥倾向的参数

| 配置 | 效果 | 风险 |
|------|------|------|
| `-XX:+DisableExplicitGC` | `System.gc()` 变 no-op，防"业务代码/RMI 触发的意外 Full GC" | **掐断 NIO 堆外内存的兜底回收**（4.5.2 的 Bits.reserveMemory → System.gc 链路）→ 可能提前抛 `OutOfMemoryError: Direct buffer memory` 或分配长时间自旋 |
| `-XX:+ExplicitGCInvokesConcurrent` | `System.gc()` 转为**并发周期**（G1 触发并发标记；CMS 触发并发回收），停顿温和；顺带完成堆外引用处理 | 类卸载不一定发生（配合 `...AndUnloadsClasses` 才保证清元空间） |

```bash
# 推荐姿势（有 DirectByteBuffer / RMI 的老应用）：
-XX:+ExplicitGCInvokesConcurrent        # 而不是 DisableExplicitGC
# 同时可加 -XX:+DisableExplicitGC 之前，先确认无堆外依赖：jmap -dump 看 DirectByteBuffer 数量
```

# 8. Stop The World（STW）

## 8.1 为什么必须有全局停顿

- GC 需要的是一张**一致性快照**：根集固定、对象图不被改（2.2.4）。
- 只要有任何线程还在 new/赋值，快照就在腐烂；"边跑边标"的并发阶段必须用写屏障把变化补偿回来（见 11 篇第 8 节）——**屏障做不到的部分才退化为 STW**。

## 8.2 一次 STW 的时间线

```text
应用线程0: ──运行─────┐ 停 顿 ──────────────┌──运行──▶
应用线程1: ──运行─────┤                     │
       ⋮             │(全部停到安全点)      │
应用线程N: ──运行─────┘                     └
GC 线程:              [枚举根|标记|复制/移动|引用处理]
                    t0                     t1            t2
停顿时长 = (t1-t0) 集合线程 + (t2-t1) GC 工作
→ "到达安全点的等待" 与 "GC 本身" 同样算进用户感知！(第 9 节)
```

## 8.3 观测实验

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

解读要点：

- PrintThread 正常时循环间隔 <1ms；`gap > 100` 的打印就是"整个进程被冻住"的证据——主线程 `System.gc()` 触发 Full GC 是最常见原因（也可能是 safepoint 同步等待、OS 抢占，注意甄别）。
- 该探针量到的是 **safepoint 停顿**而非 GC 停顿：两者相等只在"GC 触发 safepoint"时成立。
- 用 `-Xlog:gc*,safepoint`（JDK 9+）对照 GC 日志与 safepoint 日志即可归因。

## 8.4 STW ≠ GC 的全部时间

```text
GC 总耗时 = STW 阶段 + 并发阶段
              ↑ 影响延迟(P99)      ↑ 影响吞吐（抢 CPU）

Serial/Parallel: 几乎 100% STW（并行≠并发，见 11 篇第 4 节）
CMS/G1: 标记清除大部分并发，只有初标/Remark 等 STW
ZGC/Shenandoah: STW 只剩毫秒级固定开销，并发占比最高
```

- 所以"GC 用了 3 秒"未必有 3 秒停顿，反之"P99 延迟差"未必是 GC 直接造成——观测拆分见 [14_gc_tuning](14_gc_tuning.md)。

# 9. 安全点与安全区域

## 9.1 为什么"想停"要变成"能停"

- 2.2.4 说清了必须冻结程序；但 OS 层"立刻掐断线程"做不到精确定位引用（3.2），于是 HotSpot 的模型是：**VM 举旗，线程自己在合适位置停车**。

## 9.2 安全点设置在哪里

| 位置 | 为什么设在这 |
|------|--------------|
| 方法调用/返回点 | 栈帧完整，OopMap 已预生成，引用位置明确 |
| 循环回边（backward branch） | 没有它，长循环/死循环永远"到不了"安全点 |
| 异常抛出/处理点 | 帧状态本就可重建（deopt 通道现成） |
| JNI 返回点 | 从 native 回到可控的 Java 边界 |

- 反例：**计数循环（counted loop）为省开销被刻意"无安全点"**（数组拷贝、归约等）；缓解：近年 JDK（10+）为部分长计数循环插入了轮询中断点，方向是"长循环也能停车"。

## 9.3 轮询式主动中断

### 9.3.1 哨兵页原理与成本模型

```text
每个线程分配后，代码在安全点处插入一条检查（伪汇编）：
    test dword ptr [polling_page], eax   # 读一页哨兵内存，≈一条指令
正常运行：哨兵页可读 → 继续跑
VM 要停   ：把哨兵页置为不可读 (mprotect PROT_NONE) / 置标志
           → 线程跑到最近安全点时触发陷阱 → 转交 handler → 线程挂起等待
全部到齐 → VM 恢复哨兵页 → 线程继续
```

- 成本模型：中断成本 = 线程到**下一个**安全点的距离 → 安全点密度决定"集合时间"（8.2 的 t1-t0）。

### 9.3.2 SafepointSynchronize::begin 流程

- 入口函数 `SafepointSynchronize::begin()`（简化流程）：置同步标志 → 遍历线程表通知 → 等待所有 JavaThread 停在安全点或安全区域 → 状态切到 `at_safepoint` 执行 GC → `end()` 释放。等待期间的最长尾线程就是停顿的"隐形税"。

## 9.4 被动中断（信号）为什么不用

- POSIX 信号只在 OS 选定的边界投递，**不能保证落在引用一致的位置**；
- 更致命：可能打断正持有 VM 内部锁/正在分配内存的线程 → GC 线程去拿同一把锁 → **全局死锁**；
- 所以 HotSpot 只把信号用于崩溃处理，不用作 safepoint 机制。

## 9.5 安全区域（Safe Region）：解决"停不下来的线程"

- 线程处于 `sleep`/`blocked`/执行 JNI/native 代码时，**不在执行 Java 字节码**，永远碰不到安全点 → 全局 safepoint 会被它拖死。
- 安全区域定义：一段"引用关系不可能变化"的代码区间（native 调用、等待锁等）。线程进入时设本地标志，区内任意时刻 VM 视角都等价于"在安全点"；离开时若发现 safepoint 正在进行，**先自旋挂起等待放行再回到运行态**。
- 观测：`-Xlog:safepoint` 里"synchronizing 阶段耗时高"多半 = 某线程在安全区域滞留（长 JNI、锁等待）或计数循环太长。

```bash
java -Xlog:safepoint ...
# [0.412s][info][safepoint] Entering safepoint region: CollectForMetadataAllocation
# [0.415s][info][safepoint] Total time for which application threads were stopped:
#                          0.0031102 seconds, Stopping threads took: 0.0000198 seconds
#                                  └─ 整次停顿 ─┘              └─ 等待线程到齐 ─┘
```

- 定位"谁迟到"：`-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=2000`，超时会打印未到达的线程；GC 之外的 safepoint（偏向锁撤销——旧版本、撤销线程 dump、JIT 补丁等）同样计入停顿，排查流程见 [22_troubleshooting](22_troubleshooting.md)。

> 关联：偏向锁在 JDK 15 默认禁用、JDK 18 移除（JEP 374），其 safepoint 请求曾是"莫名长 safepoint"的常客；GC 阶段如何用安全点见 [11_gc_algorithm](11_gc_algorithm.md) 与 [13_low_latency_gc](13_low_latency_gc.md)。
