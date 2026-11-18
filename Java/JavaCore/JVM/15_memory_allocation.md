# 1. 对象优先在 Eden 分配

## 1.1 分配路径总览

绝大多数新对象在 Eden 出生（弱分代假说，见 [6_heap](6_heap.md) 1.3）。一次 `new` 只有两步快路径：

```text
new Klass(size)
   │
   ▼
① TLAB 快路径：私有指针碰撞（bump-the-pointer），几条指令、无锁
   │ 剩余空间不足
   ▼
② 共享慢路径：Eden 共享区 CAS 推进 top 指针，或 refill 新 TLAB
   │ 对象超大 / Eden 拿不到
   ▼
③ 老年代直入（大对象）或触发 Young GC 后再来（见 2、5 节）
```

TLAB 的存在意义与参数见 [6_heap](6_heap.md) 第 4 节与本章第 9 节。

## 1.2 为什么分配能这么快：大小编译期/类加载期已知

| 对象类型 | 大小何时确定 | 说明 |
|----------|-------------|------|
| 普通实例 | **类加载时**算好，记在 Klass 元数据里 | `new` 字节码执行时直接取 size → 指针 += size 即完成分配 |
| 数组 | **运行时**：头 + length × elem_size | 长度来自栈上参数，但仍是乘法，不需要"找空隙" |

> 对照：只有空闲列表（Free List）管理型分配（如 Parallel Old 的碎片区）才要查表找洞。复制/整理类收集器保证的"分配快"正来自这里（详见 [11_gc_algorithm](11_gc_algorithm.md)）。

## 1.3 演示：对象在 Eden 分配，Eden 满触发 Young GC

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

```text
布局：Young=10M → Eden=8M，S0=S1=1M；Old=10M

a1+a2+a3 = 6M < 8M        → 全部躺在 Eden，尚无 GC
a4 = 4M，Eden 只剩 2M      → Allocation Failure，Young GC
存活者 6M ≫ Survivor 1M   → 放不下，走分配担保直接进老年代（6M）
a4 在老年代分配            → Old: 6+4 = 10M，恰好占满

日志示意（数值随版本略有出入）：
[GC (Allocation Failure) [DefNew: 6656K->0K(9216K) ...
 6656K->10240K(20480K), 0.0xx secs]
```

结论：Survivor 是"年龄过滤器"而不是幸存者的家——幸存者一旦超过它的容量，路径立刻退化为"直入老年代"（第 5 节担保链条的入口）。

# 2. 大对象直接进入老年代

## 2.1 机制推导

### 2.1.1 为什么大对象要避开新生代（复制成本）

- 大对象（特别是生命周期短的）对 JVM 是坏消息：每次 Young GC 复制算法都要**整体搬一遍**，4MB 数组在 Eden/Survivor 间漂 15 次 = 搬 60MB。
- 复制成本 ∝ 存活字节数；大数组（`byte[]` 等）**没有内部引用**，搬运是最纯粹的 memcpy 浪费。
- 直接生老死老：Young GC 永远看不见它，一次 memcpy 都省掉；代价是它占老年代、拖慢老年代回收——所以这是"两害相权"。

### 2.1.2 各收集器的大对象策略

| 收集器 | 机制 | 判定阈值 | 备注 |
|--------|------|----------|------|
| Serial / ParNew(+CMS) | `-XX:PretenureSizeThreshold` | 显式设置才生效（默认 0 = 不区分） | 唯一吃这个参数的两员 |
| Parallel Scavenge | 参数无效 | 仅靠"TLAB 装不下→共享区/老年代"的粗规则 | — |
| G1 | Humongous 对象 | **size ≥ Region/2**，直接占老年代连续 Region | 见 2.1.3 与 [12_gc_collectors](12_gc_collectors.md) |
| ZGC | 按大小路由页 | 超大对象直接分配到老年代页，避免复制税（分代模式语义；细节随版本演进） | 见 [13_low_latency_gc](13_low_latency_gc.md) |

### 2.1.3 G1 Humongous 细则

```text
Region = 1~32MB（≈堆/2048，取 2 的幂，可 -XX:G1HeapRegionSize 显式定）
对象 ≥ Region/2  →  Humongous：从老年代划"连续" Region 串承接

7MB 对象 @ Region=8M：占 1 个 H-Region（用一半，浪费一半）
7MB 对象 @ Region=16M：普通对象（<8M 阈值），走 Eden —— 调大 Region 的动机
```

- 旧版本 H-Region 要等并发周期/Full GC 才回收，新版本 Young/Mixed 可顺带清理全空 H-Region——但碎片化的半空 H-Region 依然是 G1 Full GC 的经典导火索（排障见 [14_gc_tuning](14_gc_tuning.md) 4.5）。

## 2.2 演示

```java
// -XX:PretenureSizeThreshold=3145728  (3MB)
// 仅对 Serial 和 ParNew 有效
byte[] large = new byte[4 * 1024 * 1024];  // 4MB，直接进老年代
```

```bash
-XX:PretenureSizeThreshold=3m   # 超过 3MB 直接进老年代
```

- 注意：这个参数只对 Serial GC 和 ParNew 有效，G1 由 Humongous Region 管理大对象，Parallel/ZGC 系参数被忽略。

# 3. 长期存活对象进入老年代

## 3.1 年龄阈值晋升

### 3.1.1 阈值规则

- 对象在 Survivor 区每熬过一次 Minor GC，年龄（Age）+1。
- 年龄达到 `-XX:MaxTenuringThreshold` → 晋升到老年代。

```bash
-XX:MaxTenuringThreshold=15    # 默认 15（CMS 下为 6）
```

### 3.1.2 Mark Word 里的年龄计数器

- 存在对象的 Mark Word 中，占 4 位 → 最大值 = 15（头布局见 [9_object_layout](9_object_layout.md)；JDK 21+ 紧凑对象头实验（JEP 450）下宽度另算，上限语义不变）。
- 为什么 CMS 默认 6：CMS 老年代回收（并发重扫+清除）单价贵，希望对象**早点**定性分流；Serial/Parallel 老年代 Full 虽贵但很少来，让 Survivor 多滤几轮更划算。

## 3.2 动态年龄判定

### 3.2.1 判定规则与示例

- HotSpot 不严格按 `MaxTenuringThreshold` 晋升。
- **如果 Survivor 空间中相同年龄（及以上累计）的对象大小之和 > From 区 × TargetSurvivorRatio%（默认 50，即"一半"），则年龄 ≥ 该年龄的对象都可直接晋升。**

```text
示例：Survivor 区 100MB
年龄1: 30MB
年龄2: 30MB    ← 30+30=60 > 100/2
→ 年龄 >= 2 的所有对象晋升到老年代
```

### 3.2.2 观察：年龄分布输出

```bash
# 查看年龄分布
-XX:+PrintTenuringDistribution     # JDK 8；JDK 9+ 用 -Xlog:gc+age=trace
```

输出示例：

```text
Desired survivor size 5242880 bytes, new threshold 2 (max 15)
- age 1: 3145728 bytes, 3145728 total
- age 2: 2097152 bytes, 5242880 total
- age 3: 1048576 bytes, 6291456 total
```

- 读法：累计到 age 2 恰好摸到 desired survivor size（5242880）→ 本次动态阈值压到 2，age ≥ 2 的全部晋升。完整推导见 [6_heap](6_heap.md) 3.3。

## 3.3 哪些业务注定"长寿"

| 场景 | 晋升表现 | 本质 |
|------|----------|------|
| 本地缓存（Caffeine/Guava） | 启动后老年代快速爬升后趋稳 | 稳态工作集，正常 |
| 线程池/消息队列任务积压 | 突发流量后晋升速率陡增（任务对象在 S 区间反复搬运后被"熬"老） | 队列无界或过大是病根 |
| 会话/长连接对象 | 随连接数线性增长 | 关注回收是否与连接关闭联动 |
| 静态容器只增不减 | 老年代单调上涨不回落 | 泄漏，走 [22_troubleshooting](22_troubleshooting.md) |

# 4. 晋升观测与调优

## 4.1 两个必算指标

```text
平均晋升量 = Σ(每次 Young GC 老年代增量) / Young GC 次数     ← 日志差分或 jstat OU 差分
晋升速率   = 平均晋升量 × Young GC 频率                      ← MB/s
```

健康画像：老年代锯齿**低斜率、能回落**；晋升速率长期 ≈ 老年代并发回收速率 → 迟早 Full GC（G1 里表现为 Mixed 忙不过来，见 [14_gc_tuning](14_gc_tuning.md)）。

## 4.2 症状 → 杠杆对照表

| 症状 | 首选杠杆 | 副作用 |
|------|----------|--------|
| Survivor 溢出、大批对象以 age=1 晋升 | 增大 Survivor：`-XX:SurvivorRatio` 调小（8→4）或整体 `-Xmn` 加大 | Eden 变小，Young GC 变频 |
| Survivor 常年空闲，对象白熬 15 轮 | 调小 `MaxTenuringThreshold`，早分流 | 老年代压力前移 |
| 缓存类长寿对象占比高 | **应用层**限容/过期，JVM 参数救不了容量规划 | — |
| 动态阈值频繁 < 配置阈值 | 说明 Survivor 就是小了，回到第一行 | — |

## 4.3 老年代增速分析（区分"工作集"与"泄漏"）

```text
间隔 10~15min 抓两份 heap dump（jcmd <pid> GC.heap_dump / MAT）
→ MAT Compare Tables / 支配树 diff
→ 只增不减且引用链顶端是业务缓存/静态容器 = 泄漏；
  增到某水位后随 GC 回落 = 工作集，调堆即可。
工具操作详见 [21_visual_tools](21_visual_tools.md)。
```

# 5. 空间分配担保（Handle Promotion）

## 5.1 担保链条全景

```text
Eden 满 ──▶ 发起 Young GC
              │
              ├─ 幸存者 ≤ To 区容量 ──▶ 正常复制，年龄+1 ──▶ 完
              │
              ├─ 幸存者 > To 区容量 ──▶ 装不下的"直入"老年代
              │                          │
              │                          ├─ 老年代连续空间够 ──▶ 完（Young GC 成功）
              │                          └─ 老年代不够 ──▶ Promotion Failed /
              │                                            to-space exhausted
              │                                               │
              ▼                                               ▼
        （GC 前预判：老年代连续剩余 > 新生代已使用？否 → 先 Full GC）   Full GC 兜底
```

## 5.2 规则的版本演进

### 5.2.1 JDK 6u24 之前：基于历史平均的担保（旧版笔记描述的即此规则）

- Minor GC 之前，JVM 会检查老年代剩余空间是否够存放新生代所有对象。
- 如果不够，检查是否有分配担保：
  - 之前每次 Minor GC 进入老年代的**平均大小** < 老年代剩余空间 → 冒险执行 Minor GC。
  - 否则 → 先执行 Full GC。

### 5.2.2 JDK 6u24 之后：现行规则

检查放宽为"**老年代连续剩余 > 新生代已使用量**"即放心 GC——依据弱分代假说，幸存者本来就少，真溢出了直入老年代即可。完整推导见 [6_heap](6_heap.md) 5.2。

```bash
# 历史开关：JDK 6u24 引入；JDK 7u40 起被忽略（担保策略固定启用，无风险可拒）
-XX:+HandlePromotionFailure
```

## 5.3 G1 / ZGC 下的对应物

| 收集器 | 担保形式 | 失败日志特征 |
|--------|----------|--------------|
| CMS/ParNew | 连续老年代 + 平均晋升预估 | `promotion failed` |
| G1 | `G1ReservePercent`（默认 10%）预留晋升缓冲 | `(to-space exhausted)` → 退化 Full GC |
| 分代 ZGC | young→old 晋升失败路径存在但罕见（并发兜底） | `Allocation Stall` |

失败后的处置统一见 [14_gc_tuning](14_gc_tuning.md) 4.3。

# 6. 逃逸分析（Escape Analysis）

## 6.1 定义：对象会不会"逃出"它的方法/线程

逃逸分析研究对象的**动态作用域**：`new` 出来的引用，最终会被哪些代码看到？

### 6.1.1 动态作用域示意

```text
              ┌──── 方法作用域 ────┐
   new Point()│ p 在这里出生       │
              │ ① p 只被本方法用 ──┼──▶ 不逃逸（NoEscape）
              │ ② p 传进其它方法   ├──▶ 参数逃逸（ArgEscape，同线程）
              └────────┬───────────┘
                       │ ③ p 被返回/写静态字段/放进集合被别的线程读
                       ▼
                  全局逃逸（GlobalEscape）→ 只能老实待在堆里
```

### 6.1.2 三种逃逸级别

| 逃逸级别 | 典型写法 | 可做的优化 |
|----------|----------|-----------|
| NoEscape | 局部 new + 局部使用 | 标量替换/栈上分配 + 同步消除（第 7 节全家桶） |
| ArgEscape | `foo(new Point())`，foo 内部不再传出 | 内联后常降级为 NoEscape，仍有救 |
| GlobalEscape | return、赋静态字段/堆字段、跨线程传递 | 无 |

## 6.2 在哪做、何时开

| 要点 | 说明 |
|------|------|
| 实现位置 | **C2 编译器**（服务端编译器）内做，解释器与 C1 不做 → 预热期照旧在堆上分配（见 [16_execution_engine](16_execution_engine.md) 分层编译） |
| 默认状态 | JDK 6u23 引入并默认开启，**8u40+ 行为稳定**；现代 JDK 无需再显式开 |
| 依赖 | 强依赖内联与去优化生态——调用图变了，逃逸结论会变（6.3） |

```bash
-XX:+DoEscapeAnalysis        # 开启逃逸分析（默认）
-XX:-DoEscapeAnalysis        # 关闭
-XX:+PrintEscapeAnalysis     # 打印逃逸分析结果（需要 fastdebug 版 JDK）
```

## 6.3 最小判定示例

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

# 7. 逃逸分析带来的三大优化

## 7.1 优化一：栈上分配（效果演示）

### 7.1.1 演示代码

```java
// -Xmx16m -Xms16m -XX:+PrintGCDetails
public class StackAllocDemo {
    static class Point { int x; int y; }
    static int consume(Point p) { return p.x + p.y; }   // 同线程内传参，仍可被内联

    public static void main(String[] args) {
        long sum = 0;
        for (long i = 0; i < 1_000_000_000L; i++) {
            Point p = new Point();      // 不逃逸
            p.x = (int) i; p.y = (int) i;
            sum += consume(p);
        }
        System.out.println(sum);
    }
}
```

### 7.1.2 A/B 对照与内存布局

| 配置 | GC 行为 | 解释 |
|------|---------|------|
| `-XX:-DoEscapeAnalysis` | Young GC 刷屏 | 10^9 × ≥16B ≈ 15GB 从 Eden 流过 |
| 默认开启 | **几乎看不到 GC** | 对象根本不在堆上出生（被 7.2 消化） |

```text
堆路径（关闭逃逸分析）:                  逃逸分析开启后:
Eden ┌──┬──┬──┬──┐ 每 16B 占位           栈帧/寄存器 ┌ x: slot/reg ┐
     └──┴──┴──┴──┘ 满了→Young GC                    └ y: slot/reg ┘   无堆内存 touched
```

### 7.1.3 注脚：HotSpot 的"栈上分配"主要靠标量替换实现

> 诚实的注脚：HotSpot 的"栈上分配"**几乎总是经由标量替换实现**——把对象拆成字段放进寄存器/栈槽，"对象"作为一个整体从未存在；真正"按对象粒度在栈帧里开一块"的经典栈分配并不是主力形态。教科书把两者并称，机制上要分清。

## 7.2 标量替换（Scalar Replacement）

- 如果对象不逃逸，可以不创建对象，直接把对象的成员变量当作局部变量。

### 7.2.1 Java 等效改写

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

### 7.2.2 IR 前后对比与开关

```text
替换前的 IR（示意）:                     替换后:
  rax = alloc(Point)          8B头+8B字段    x ← 1   (寄存器/slot)
  [rax+8] = 1                               y ← 2   (寄存器/slot)
  [rax+12]= 2                               rax = x*y  → imul
  rbx = [rax+8] * [rax+12]                  （访存 2 次、堆分配 1 次 → 全没了）
```

- 与 7.1 的关系：标量替换是**实现手段**，"栈上分配"是对外表述的**效果**；被拆的字段落在寄存器还是栈槽由寄存器分配器决定。

```bash
-XX:+EliminateAllocations    # 开启标量替换（默认）
```

## 7.3 同步消除（Lock Elimination）

- 如果对象不逃逸，JIT 可以消除其上的同步锁——别的线程根本拿不到这个引用，加锁纯属自缚。

### 7.3.1 示例与开关

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

### 7.3.2 生产观察手段

- 观察：`-XX:+PrintEliminateLocks` 一类日志参数只在 **debug 版 JDK** 可用，生产别指望；替代观察法是 A/B 对比 `-XX:-DoEscapeAnalysis` 前后的 JMH 结果 + JFR 锁事件（`jdk.JavaMonitorEnter`），或 `-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly`（需 hsdis）看锁汇编是否消失。关联优化（锁粗化/轻量级锁）见 [19_lock_optimization](19_lock_optimization.md)。

# 8. 逃逸路径清单与分析的局限

## 8.1 对象逃逸路径速查

### 8.1.1 逃逸路径清单

| 逃逸路径 | 示例 | 判定提示 |
|----------|------|----------|
| 方法返回值 | `return new Point();` | 最常见 |
| 写入静态字段 | `instance = new EscapeTest();` | 见 8.1.2 演示代码 |
| 写入堆对象字段 | `holder.p = new Point();` | holder 本身逃逸则连带逃逸 |
| 放进容器再流出 | `list.add(obj)`，list 是字段/返回值 | 集合是逃逸分析的天敌 |
| 跨线程传递 | `executor.submit(() -> use(obj))`、synchronized(obj) 且 obj 可被他人获得 | 线程逃逸 = GlobalEscape |
| this 逸出 | 构造函数里 `Some.registry.add(this)`、启动内部线程 | 隐蔽；破坏的不止局部优化 |
| JNI / 反射句柄外泄 | 把引用交给 native / 序列化图 | 分析直接放弃 |

### 8.1.2 判定演示代码

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

## 8.2 局限：别把它当"确定性优化"

| 局限 | 机理 | 工程含义 |
|------|------|----------|
| 只在 C2、且要够热 | 解释执行/C1 阶段照常堆分配 | 短命进程/低频路径享受不到 |
| 大方法/深调用图放弃分析 | 指针分析代价 ∝ 调用图规模，超限直接不分析 | "上帝类"里的 new 基本不配被优化 |
| 依赖内联决策 | 先内联、再判逃逸；内联预算（方法大小/热度）一变，结论翻转 | 升级 JDK、加个注解都可能改变结果 |
| 激进优化 + 去优化 | 基于假设（类型、热路径），假设破裂整段回退重编译 | 逃逸优化不是"编译一次永久生效" |
| 虚拟调用/集合持有 | 无法静态确定接收者 → 保守判逃逸 | "接口 + 集合"现代码风格收益低 |

> 结论：**逃逸分析用来"解释性能为什么好"，不要用来"设计一个必须不逃逸才能活的架构"。** 要省下那次分配，靠第 10 节的硬手段。更多 JIT 视角见 [17_jit_optimization](17_jit_optimization.md) 第 2~4 节。

# 9. TLAB 细节

（TLAB 的定位与参数表见 [6_heap](6_heap.md) 第 4 节，这里只挖"浪费与回填"。）

分配入口的整体骨架（保留自旧版并扩注）：

```java
boolean allocate(Object obj) {
    if (tlab.hasSpace(obj.size)) {
        tlab.allocate(obj);       // 快速路径：私有指针碰撞，无锁
        return true;
    } else {
        // TLAB 满了：按 wastage 规则决定"refill 新 TLAB"还是"回退 Eden 共享区 CAS"
        return allocateNewTLABOrFallback(obj);   // 细化见 9.1
    }
}
```

## 9.1 wastage 与 refill 决策

### 9.1.1 决策流程

```text
对象要 1KB，TLAB 只剩 300B：
  ├─ 剩余 300B ≤ 对象大小/20（TLABWasteIncrement=20）?
  │     是 → 放弃残尾（记为 wastage），CAS 认领新 TLAB（refill）
  │     否 → 不新开，直接去 Eden 共享区 CAS 分配（省得浪费）
  └─ 新 TLAB 大小由 -XX:+ResizeTLAB 自适应；TLABWasteTargetPercent=1 控制
     单个 TLAB 最大占 Eden 的比例（默认 1%），间接决定 refill 频率
```

### 9.1.2 尺寸权衡

- **wastage 是设计内成本**：用 ≤5% 的尾部浪费换"绝大多数分配零同步"。
- TLAB 过小 → refill 频繁（每次 refill 仍要 CAS）；过大 → wastage 率与 Young GC 复制时的空转上升。`-XX:TLABSize` 固定大小属于诊断手段，日常让 ResizeTLAB 自适应。

## 9.2 快慢路径成本对比

```text
TLAB fast path:   load top / add size / cmp limit / store top
                  ≈ 1~2ns，纯本核 L1 热数据，无锁无原子指令
共享区 CAS path:  lock cmpxchg 同一 top 指针
                  ≈ 无争抢几十 ns；高争抢时缓存行在核间来回 invalidate，微秒级也可能
→ 高分配率服务里 TLAB 是分配性能的地基；-XX:-UseTLAB 只用于极端诊断
```

## 9.3 观测

```bash
# JDK 8
-XX:+PrintTLAB                          # 分配/refill/slow/wastage 统计（解读见 6_heap 4.4）
# JDK 9+：统一日志 + JFR
-Xlog:gc+tlab*=debug
jcmd <pid> JFR.dump name=t path=r.jfr   # 事件：jdk.ObjectAllocationInNewTLAB（快路径 refill）
                                        #      jdk.ObjectAllocationOutsideTLAB（直上慢路径/大对象）
```

OutsideTLAB 事件占比异常高 → 大对象直穿 TLAB，回到第 2 节检查尺寸分布。

# 10. 内存分配实战优化

## 10.1 减少不必要的对象创建

### 10.1.1 代码对比

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

### 10.1.2 延伸三件套

延伸三件套：

- **原始类型集合**：`Map<String,Integer>` 每个 entry + 装箱都是对象；用 fastutil / Eclipse Collections 的 `Int2ObjectOpenHashMap` 等，条目多时内存与分配双降。
- 热点路径的 `Iterator`、varargs、`String.split` 都是隐形分配大户（asprof -e alloc 一照便知，命令见 [14_gc_tuning](14_gc_tuning.md) 3.3）。
- 复用昂贵格式器：`DateTimeFormatter` 线程安全可 static；`SimpleDateFormat` 不行（换 API）。

## 10.2 合理设置大对象阈值

```java
// 哪些场景适合直接进老年代？
// 1. 静态大缓存对象（生命周期长）
static final byte[] CACHE = new byte[10 * 1024 * 1024];

// 2. 一次性读取的大文件
byte[] fileData = Files.readAllBytes(Paths.get("large_file.bin"));
```

- 第 1 类没问题：只分配一次，直入老年代反而干净。
- 第 2 类是事故现场：大文件 × 高并发 = 老年代被瞬时大数组打爆。改流式/分段读，或至少限制并发度——**参数救不了分配模式**（G1 侧调 Region 见 2.1.3）。

## 10.3 澄清："提前置空引用"意义不大

```java
byte[] buf = load();
process(buf);
buf = null;            // 老建议："帮 GC 早点回收"
useOtherStuff();       // 此后 buf 再也没被读过
```

- 现代 JIT 按**活跃性（liveness）**回收：`process(buf)` 之后 buf 已死，Young GC 即可回收它，**不等你置 null**。
- 仍建议置空的少数场景：极长方法中间"确认不再用"的超大对象（防后续维护加入引用）、以及被 deopt/调试信息延长引用的边角——属于保险习惯，不是性能手段。
- 真正影响回收时机的是**引用链是否从 GC Roots 可达**（详见 [10_gc_basics](10_gc_basics.md)），不是局部变量赋值。

## 10.4 对象池：默认反模式

```text
"池化复用" 的账：
  现代 TLAB 分配 ≈ ns 级（9.2）+ 年轻代回收 ∝ 存活数（死对象几乎免费）
  池化成本 = 取还簿记 + 池锁争抢 + 对象长期存活（反让 Young GC 变贵）+ 忘记归还=泄漏
→ 一般业务对象：别池化。
→ 值得池化的判据是"创建本身昂贵"：数据库连接、线程、DirectByteBuffer/Netty ByteBuf。
→ 虚拟线程（JDK 21，JEP 444）时代"海量短命对象"是鼓励的写法，进一步拉低池化的适用面。
```

## 10.5 缓存友好布局：GC 之外的分配红利

| 手法 | 说明 |
|------|------|
| 数组 > 链表 | 遍历时连续内存吃满 cache line + 硬件预取；节点化结构每跳一次大概率 miss |
| 对象内字段排布 | JVM 已按宽度/对齐重排字段（见 [9_object_layout](9_object_layout.md)）；自己再做"热字段聚拢" |
| `@Contended` 防伪共享 | 多线程高频写的相邻字段（计数器/序列号）落在同一缓存行 → 核间无效化风暴；`-XX:-RestrictContended` 后使用 |
| 大 workload 分批 | 一次处理 100 万行不如每批 1 万行：批次尺寸对齐 L2/L3，GC 也在批间自然发生 |

# 11. 参数与观测速查

## 11.1 参数速查

```bash
# TLAB
-XX:+UseTLAB                   # 开启 TLAB
-XX:TLABSize=64k               # TLAB 大小
-XX:TLABWasteTargetPercent=1   # TLAB 浪费空间目标占比

# 大对象
-XX:PretenureSizeThreshold=3m  # 直接进老年代的大小（仅 Serial/ParNew）

# 晋升策略
-XX:MaxTenuringThreshold=15    # 最大晋升年龄（CMS 默认 6）
-XX:TargetSurvivorRatio=50     # Survivor 目标占用水位（动态年龄判定用，默认 50）

# 逃逸分析
-XX:+DoEscapeAnalysis          # 逃逸分析（JDK 8u40+ 默认开，仅 C2）
-XX:+EliminateAllocations      # 标量替换
-XX:+EliminateLocks            # 同步消除

# 分配担保
-XX:+HandlePromotionFailure    # 历史参数：JDK 7u40+ 已忽略（固定启用，见 5.2）
```

## 11.2 观测命令速查

| 想看什么 | 命令 |
|----------|------|
| Eden/Survivor/Old 实时水位、YGC/FGC 计数 | `jstat -gc <pid> 1000`（列含义见 [14_gc_tuning](14_gc_tuning.md) 3.3） |
| 堆整体容量与使用概览 | `jcmd <pid> GC.heap_info` |
| 当前生效的 JVM 参数 | `jcmd <pid> VM.flags` |
| 谁在分配（分配火焰图） | `./asprof -e alloc -d 60 -f alloc.html <pid>` |
| TLAB 命中 / OutsideTLAB | JFR `jdk.ObjectAllocationInNewTLAB` / `OutsideTLAB` |
| 晋升年龄分布 | `-Xlog:gc+age=trace`（JDK 9+）/ `-XX:+PrintTenuringDistribution`（8） |
| 对象直方图 | `jmap -histo <pid>`（`:live` 会触发回收，生产慎用） |
| 逃逸分析效果 A/B | `-XX:-DoEscapeAnalysis` 重跑，对比 Young GC 次数与耗时 |
