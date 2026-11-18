# 1. 堆的内存划分

## 1.1 堆的定位

| 属性 | 说明 |
|------|------|
| 管理者 | JVM 自动管理（分配 + 回收），是唯一"几乎全部对象"驻留的区域 |
| 共享性 | **所有线程共享**，因此分配需要并发安全手段（TLAB / CAS） |
| 生命周期 | 虚拟机启动时创建，`-Xms` / `-Xmx` 控制 |
| 例外 | 经**逃逸分析**判定不逃逸的对象可栈上分配（见 [15_memory_allocation](15_memory_allocation.md)） |
| 调优地位 | GC 调优的主战场，80% 的 JVM 参数与堆有关 |

## 1.2 经典分代堆结构（Serial / Parallel）

```text
物理上：不连续的内存空间（逻辑上连续）

┌──────────────────────────────────────────────────┐
│                     堆 (Heap)                    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │            新生代 (Young Generation)       │    │
│  │                                          │    │
│  │  ┌──────────┬───────────┬───────────┐    │    │
│  │  │   Eden   │ Survivor0 │ Survivor1 │    │    │
│  │  │  (80%)   │   (10%)   │   (10%)   │    │    │
│  │  └──────────┴───────────┴───────────┘    │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │            老年代 (Old Generation)         │    │
│  │           Tenured / Old Gen               │    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘

比例关系：
  Young : Old     = 1 : 2    (-XX:NewRatio=2)
  Eden : S0 : S1  = 8 : 1 : 1 (-XX:SurvivorRatio=8)
```

> 注意：这个"物理连续"的画面只对 Serial / Parallel 成立。**G1 起堆被切成 Region，ZGC 起堆被切成 Page，分代只是逻辑概念**（见 9.2 / 9.3）。

## 1.3 分代收集的理论假说

### 1.3.1 弱分代假说（Weak Generational Hypothesis）

- 绝大多数对象都是**朝生夕死**的。
- 实测数据：典型 Web 应用中约 90% 的对象在 Young GC 时被回收。
- 推论：把"朝生夕死"的对象圈在小区域里高频回收 → 单次 GC 扫描成本极低。

### 1.3.2 强分代假说（Strong Generational Hypothesis）

- 熬过越多次 GC 的对象越难以消亡。
- 推论：幸存者可以"迁"进一个低频回收的大区域（老年代），不必反复扫描。

### 1.3.3 跨代引用假说（Intergenerational Hypothesis）

- 跨代引用相对于同代引用**极少**。
- 推论：为极少数跨代引用全量扫描老年代不划算 → 引入**记忆集 / 卡表**记录"老年代 → 新生代"的脏引用（详见 [11_gc_algorithm](11_gc_algorithm.md)）。

## 1.4 分代比例的计算

参数优先级：`-Xmn` > `-XX:NewRatio`；Eden/Survivor 由 `-XX:SurvivorRatio` 决定。

```text
示例：-Xmx8g -XX:NewRatio=2 -XX:SurvivorRatio=8

New = 8g / (1+2)          = 2.67g
Old = 8g - 2.67g          = 5.33g
Eden = 2.67g / (8+1+1)×8  = 2.13g
S0 = S1 = 2.67g × 1/10    = 0.27g

示例：-Xmx8g -Xmn3g -XX:SurvivorRatio=8

New = 3g（Xmn 直接指定，优先）
Eden = 2.4g，S0 = S1 = 0.3g
```

| 现象 | 原因与后果 |
|------|-----------|
| Eden 过小 | Young GC 过于频繁，吞吐被 STW 蚕食 |
| Survivor 过小 | 幸存者放不下 → 提前晋升 → Full GC 变多 |
| 新生代过大 | 单次 Young GC 变慢，且老年代被挤压 |

# 2. 新生代（Young Generation）

## 2.1 Eden 区

### 2.1.1 分配方式：指针碰撞（Bump the Pointer）

堆在使用-整理类算法（复制/整理）后内存连续，分配只需移动指针：

```text
空闲指针 ──▶ 新对象1 │ 新对象2 │ ... 剩余空间
           (top-of-heap pointer 原子加一个对象大小即完成分配)
```

- 与之相对的是**空闲列表（Free List）**：堆不连续时（标记-清除产物）需查表找空隙，Parallel Scavenge 的"堆连续"倾向即为此服务。
- Eden 是对象出生的默认地点，写满即触发 Young GC。

### 2.1.2 TLAB 位于 Eden 内

- 多线程并发分配会竞争"移动指针"这一原子操作 → JVM 在 Eden 内为每个线程划私有小块 **TLAB**（见第 4 节）。

## 2.2 Survivor 区

### 2.2.1 Minor GC 复制流程与 from/to 交换

```text
第 N 次 Young GC:                 第 N+1 次 Young GC:
┌─────┐ ┌───┐ ┌───┐              ┌───┐ ┌─────┐ ┌───┐
│Eden │ │S0 │ │S1 │   ──GC──▶    │Eden│ │S0 │ │S1 │
│ 垃圾│ │   │ │存活│  (From=S0)   │ 垃圾│ │存活│ │   │
└─────┘ └───┘ └─S0─┘  To=S1      └────┘ └─S1──┘ └───┘
```

- S0 / S1 没有固定身份，每次 GC 后**角色互换**（From ↔ To）。
- 复制时对象年龄 +1；年龄达阈值晋升老年代。

### 2.2.2 为什么需要两个 Survivor

- 若只有一个：每次复制目标在 Eden 之外，GC 结束又要"清空→重建"，对象无处暂存 → 要么频繁晋升、要么碎片化。
- 两个区 + 角色互换：始终有一块**完整空闲区**承接存活对象，复制算法零碎片，且年轻幸存者能在 Survivor 间多熬几轮，过滤掉"活但不老"的对象。

### 2.2.3 为什么是 8:1:1

- IBM 实验（1995）：新生代中约 88%+ 的对象"立刻死亡"。
- Eden : S = 8 : 1 时，若对象平均要熬 2~3 次 GC，Survivor 需求约 1/8 ~ 1/4，取 1/8 平衡空间利用率与复制损耗。
- 代价模型：

```text
复制损耗 ≈ 1/(R+1)   （R = Eden/S 中较小者比例）
R=8 时：真正可存活的"有效空间"只有 10/12 ≈ 83%（浪费的 1 个 Survivor 是复制算法的固有成本）
```

### 2.2.4 对象年龄如何记录

- 存在 Mark Word 的 **age 字段，4 bit，上限 15**（详见 [9_object_layout](9_object_layout.md)）。
- `-XX:MaxTenuringThreshold`：默认 15；**配合 CMS 时默认降为 6**（CMS 老年代回收更贵，希望早点分流）。
- 年龄 15 的对象下次 Young GC 直接进老年代，不再在 Survivor 间搬移。

## 2.3 Young GC 完整时序

```text
应用线程: ──分配──分配──分配(Eeden满)──[STW]───────────────恢复──▶
GC线程:                      │  1.扫根(栈+卡表命中)             │
                             │  2.Eden+From 存活者复制到 To     │
                             │  3.年龄+1，超阈值者晋升 Old       │
                             │  4.To 满 → 剩余直接晋升(担保)     │
                             └── 整个过程多线程并行 ──┘
```

# 3. 老年代（Old Generation）

## 3.1 特点与触发时机

- 存放长期存活对象；空间不足触发 Major GC / Full GC。
- Full GC 通常比 Young GC **慢 10 倍以上**：扫描整堆、老年代对象多、常用整理算法搬移量大。
- 调优第一目标：**压低 Full GC 频率**，而不是消灭 Young GC。

## 3.2 对象进入老年代的四条路径

| 路径 | 条件 | 相关参数 |
|------|------|----------|
| 年龄晋升 | 年龄 ≥ MaxTenuringThreshold | `-XX:MaxTenuringThreshold` |
| 动态年龄 | 相同(及更老)年龄对象总大小 > From 区 × TargetSurvivorRatio% | `-XX:TargetSurvivorRatio`（**默认 50**） |
| 大对象直入 | 对象超大，避免在 Survivor 间反复拷贝 | `-XX:PretenureSizeThreshold`（仅 Serial/ParNew 系有效） |
| 空间不足 | Survivor 放不下 / To 区溢出 → 直接进老年代 | 分配担保，见 5.2 |

## 3.3 动态年龄判定详解

规则：把 Survivor 中对象**按年龄从小到大累计字节数**，当累计值首次超过 `From 区大小 × TargetSurvivorRatio%` 时，**把 ≥ 该年龄的对象全部晋升**（实际阈值 = min(配置阈值, 动态阈值)）。

```text
示例：From 区 1MB，TargetSurvivorRatio=50（阈值 512KB）

年龄 1：200KB   累计 200KB  < 512KB
年龄 2：150KB   累计 350KB  < 512KB
年龄 3：200KB   累计 550KB  ≥ 512KB  ← 命中

结论：本次 GC 中年龄 ≥ 3 的对象全部晋升老年代，
      不必等 MaxTenuringThreshold=15。
```

- 意义：Survivor 有被幸存者撑爆的风险时，提前向老年代"泄洪"，避免 To-Eden 溢出。
- 观察：`-XX:+PrintTenuringDistribution` 输出 `desired survivor size` 与 `age   x   bytes` 表。

## 3.4 大对象直接分配的版本差异

| 收集器 | "大对象直入老年代"的实现 |
|--------|------------------------|
| Serial / ParNew(+CMS) | `-XX:PretenureSizeThreshold`，超过即老年代分配 |
| Parallel / G1 / ZGC | 该参数**无效**；G1 用 **Humongous**（对象 ≥ Region/2，见 [12_gc_collectors](12_gc_collectors.md)），ZGC 按大小路由 young/old 页 |

# 4. TLAB（Thread Local Allocation Buffer）

## 4.1 为什么需要

```text
无 TLAB：                          有 TLAB：
Eden ┌───────────────┐             Eden ┌──────┬──────┬──────┬────────┐
     │ 线程1 线程2    │             │TLAB1│TLAB2│TLAB3│ 共享区 │
     │ 抢 top 指针   │             │线程1│线程2│线程3│(CAS)  │
     │ (CAS 自旋)    │             └──────┴──────┴──────┴────────┘
     └───────────────┘             线程私有指针推进，无锁； refill 时小概率 CAS
```

## 4.2 分配与 refill 流程

1. 对象 ≤ TLAB 剩余空间 → 私有指针碰撞，直接分配。
2. 剩余空间不足：
   - 若剩余空间 < **新对象大小的 1/20**（`-XX:TLABWasteIncrement=20`，即浪费 ≤5% 才允许另开）→ 放弃旧 TLAB 剩余部分，**refill 新 TLAB**。
   - 否则直接从 Eden **共享区 CAS** 分配（慢路径），减少浪费。
3. TLAB 大小默认 = Eden /（并发线程数相关）动态调整，`-XX:+ResizeTLAB`（默认开）。

## 4.3 参数速查

| 参数 | 默认 | 说明 |
|------|------|------|
| `-XX:+UseTLAB` | 开启 | 关闭则全部走 Eden 共享 CAS |
| `-XX:TLABSize` | 0（自适应） | 固定 TLAB 大小，0 表示 JVM 计算 |
| `-XX:+ResizeTLAB` | 开启 | 允许动态调整 TLAB 大小 |
| `-XX:TLABWasteIncrement` | 20 | 剩余空间不足对象大小 1/20 才 refill |
| `-XX:+PrintTLAB` | 关闭 | 观察分配/浪费统计（JDK8；JDK9+ 用 `-Xlog:gc+tlab`） |

## 4.4 观察 PrintTLAB 输出（JDK 8 示例）

```text
TLAB stats, version:4, total time:2.34s
TLAB use: 98.3%  refill: 812  slow: 3  fast: 57210
Waste  due to refills: 0.01%  shared: 0.00%
# 解读：TLAB 使用率 >95% 健康；refill/slow 高 → TLAB 太小，考虑 -XX:TLABSize
```

# 5. 对象晋升与分配担保

## 5.1 对象创建-晋升总流程

```text
  new Object()
       │
       ▼
  ┌───────────┐
  │尝试栈上分配│──成功──▶ 栈上对象（GC无需回收）
  └────┬──────┘
       │失败
       ▼
  ┌────────────┐
  │尝试TLAB分配│──成功──▶ Eden 区分配完成
  └────┬───────┘
       │失败(TLAB满)
       ▼
  ┌──────────────┐
  │Eden 共享区域  │──失败──▶ 新TLAB或降级到CAS分配
  │  CAS 分配     │
  └────┬─────────┘
       │分配完成
       ▼
  ┌──────────┐
  │Eden 区中  │ ──Minor GC──▶ Survivor 区
  └──────────┘                  │
                          年龄 >= 15 或 动态年龄
                                 │
                                 ▼
                            老年代 (Old Gen)
```

## 5.2 分配担保机制（Handle Promotion）

### 5.2.1 JDK 6u24 之前

- 每次 Young GC 前检查：**老年代最大连续空闲 > 新生代全部对象总大小**？
- 不满足 → 先 Full GC 再 Young GC；Full 后仍不满足 → 本次 Young GC **失败**。

### 5.2.2 JDK 6u24 之后（现行规则）

- 检查简化为：**老年代连续剩余空间 > 新生代(Eden+From)已使用量**，满足即放心 Young GC，晋升失败时把存活对象直接进老年代（依赖"弱分代假说"——幸存者本来就少）。
- `-XX:+HandlePromotionFailure` 在 JDK 7+ 被**忽略**（担保策略固定为始终启用）。

### 5.2.3 G1 / ZGC 的做法

- G1：晋升目标不是"连续老年代"而是若干 Old Region；空间不足靠 `-XX:G1ReservePercent`（默认 10% 预留）兜底，仍不足 → **Evacuation Failure / to-space exhausted** → 退化 G1 Full GC。
- 分代 ZGC：同样有 young→old 晋升与 relocation 失败路径（见 [13_low_latency_gc](13_low_latency_gc.md)）。

## 5.3 晋升失败的日志特征

```text
(CMS)  promotion failed      → 调大 Survivor 或降低晋升阈值
(G1)    Evacuation Failure   → 调大 G1ReservePercent / 降低 IHOP
(ZGC)   Allocation Stall     → 分配等待 GC，堆偏小或分配率过高
```

排障流程见 [14_gc_tuning](14_gc_tuning.md) 与 [22_troubleshooting](22_troubleshooting.md)。

# 6. 字符串常量池（String Pool）

## 6.1 位置演进

| 时期 | 位置 | 原因 |
|------|------|------|
| JDK ≤ 6 | **永久代**中的 Hash 表 | 与类元数据混住 |
| JDK 7+ | **堆**中的 Hash 表（`StringTable`） | 永久代难调优、GC 效率低（回收常量池必须 Full GC）；移入堆后可被普通 GC 触达 |

```text
        JDK 6:  PermGen ┌ StringTable ┐        JDK 7+:  Heap ┌ StringTable ┐
        (Full GC 才能回收)                        (Young/Mixed GC 均可回收)
```

## 6.2 字面量与 new String

```java
String s1 = "abc";                    // 字面量 → 直接入池
String s2 = new String("abc");        // 堆中创建新对象（不在池中）
String s3 = s2.intern();              // intern()：若池中有则返回池中引用，否则入池
System.out.println(s1 == s2);         // false
System.out.println(s1 == s3);         // true
```

```text
"abc" 字面量编译期进 Class 常量池 → 运行时进 StringTable
            StringTable
              ┌─────┐
   s1 ──────▶ │"abc"│◀ ─ ─ ┌─────┐
              └─────┘      │"abc"│◀─ s2 （堆中独立对象，内容相同）
                           └─────┘
   s3 = s2.intern() → 返回 StringTable 中那个对象
```

## 6.3 intern() 的版本差异

### 6.3.1 JDK 6：拷贝入池

- 池中无该内容时，把字符数组**复制一份**进永久代池中 → `s1 == s3` 为 false（返回的是另一个对象）。

### 6.3.2 JDK 7+：引用入池

- 池中无该内容时，直接把**堆中对象的引用**记入池中 → `s1 == s3` 为 true。

```java
String s1 = new StringBuilder("hel").append("lo").toString();
System.out.println(s1.intern() == s1);  // JDK 7+ : true (s1 的引用入池)

String s2 = new StringBuilder("ja").append("va").toString();
System.out.println(s2.intern() == s2);  // false ("java" 早已被 JVM 加载入池)
```

## 6.4 字符串去重（G1 String Deduplication）

```bash
-XX:+UseStringDeduplication    # JDK 8u60 引入，JDK 11 转正，仅 G1
-XX:+PrintStringDeduplicationStatistics   # JDK8 观察
```

- 原理：Young GC / Mixed GC 时对**完全相同内容**的 `char[]`/`byte[]` 只保留一份，多余字符串改为共享数组（"垃圾堆积"式，非拷贝）。
- 适用：堆中大比例重复字符串（缓存、报文副本）；**只对老年代存活串生效**，收益滞后；CPU 有额外开销。
- 替代：业务层显式去重（如 `Map` 驻留），或干脆少造 String。

## 6.5 桶数与高并发争抢

```bash
-XX:StringTableSize=100000   # 默认 60013，JDK 9（JEP 251）后大小改为按堆自适应，该参数趋于废弃
```

- StringTable 是分段锁 Hash 表，桶数太少 → 高并发 `intern()` 锁争抢 → 全局变慢。
- 老系统若大量 intern 且 JDK ≤ 8，可调大桶数；JDK 9+ 无需关心。

## 6.6 字符串池的回收

- 移入堆后，**无人引用的驻留字符串可被普通 GC 回收**（与永久代"几乎不回收"本质不同）。
- 观察：`jmap -heap <pid>` 中 `String Table` 一节；`-Xlog strings`（JDK 11+ `jcmd <pid> VM.string_table` 相关能力有限，MAT 是主要工具）。

# 7. 堆参数详解

| 分组 | 参数 | 默认/建议 | 说明 |
|------|------|-----------|------|
| 基础 | `-Xms` / `-Xmx` | 建议相等 | 避免堆扩缩容抖动与额外 STW |
| 基础 | `-Xmn` | — | 新生代大小，**优先于 NewRatio** |
| 分代比例 | `-XX:NewRatio=2` | 2 | 老年:新生 = 2:1 |
| 分代比例 | `-XX:SurvivorRatio=8` | 8 | Eden : S = 8:1 |
| 晋升 | `-XX:MaxTenuringThreshold` | 15（CMS 6） | 4 bit 上限 15 |
| 晋升 | `-XX:TargetSurvivorRatio` | **50** | 动态年龄判定水位 |
| 大对象 | `-XX:PretenureSizeThreshold` | 0 | 仅 Serial/ParNew；0=不分大对象 |
| 预留 | `-XX:G1ReservePercent` | 10 | G1 晋升兜底空间 |
| 观察 | `-XX:+PrintTenuringDistribution` | — | JDK8；JDK9+ `-Xlog:gc+age` |
| 兜底 | `-XX:+HeapDumpOnOutOfMemoryError` | 关闭 | **生产强烈建议开启** |
| 兜底 | `-XX:HeapDumpPath` | 当前目录 | 目录需存在，`%p` 自动填 PID |

# 8. 堆溢出演示

## 8.1 复现代码

```java
// -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
public class HeapOOM {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true) {
            list.add(new byte[1024 * 1024]); // 每次分配1MB
        }
    }
}
// Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
// Dumping heap to java_pid<pid>.hprof ...
```

## 8.2 dump 之后的分析路径

```text
hprof 文件 → MAT 打开 → Leak Suspects Report → Dominator Tree → 追 GC Roots 引用链
（工具操作详见 [21_visual_tools](21_visual_tools.md)，流程详见 [22_troubleshooting](22_troubleshooting.md)）
```

## 8.3 堆相关 OOM 报错速查

| 报错 | 含义 | 第一步动作 |
|------|------|-----------|
| `Java heap space` | 堆真不够 / 泄漏 | dump + MAT |
| `GC overhead limit exceeded` | >98% 时间在 GC、回收 <2% | 同上，多为"堆将满未满但回收无效" |
| `Array size exceeds VM limit` | 数组长度超上限 | 检查异常数据量 |
| `unable to create new native thread` | **不是堆**，是线程/OS 资源 | 见 [22_troubleshooting](22_troubleshooting.md) 第 6 节 |

# 9. JDK 不同版本堆变化

## 9.1 版本演进表

| 版本 | 堆/回收相关变化 |
|------|----------------|
| JDK 7 及之前 | 永久代（PermGen），堆 + 永久代 |
| JDK 8 | 移除永久代，引入元空间（Metaspace），字符串常量池在堆中 |
| JDK 9 | 默认 GC 改为 G1（堆分代不再是连续物理空间）；intern 表自适应 |
| JDK 11 | 废弃 CMS 计划启动；ZGC 实验性（JEP 333，1TB 堆、<10ms） |
| JDK 14 | CMS 正式移除 |
| JDK 15 | ZGC 转正；**偏向锁默认关闭**（JEP 374） |
| JDK 17 | ZGC 支持 macOS/Windows；G1 持续优化 |
| JDK 21 | 虚拟线程转正（JEP 444）；**分代 ZGC 实验**（JEP 439）；紧凑对象头实验（JEP 450） |
| JDK 23 | **分代 ZGC 成为 ZGC 默认**（JEP 490） |
| JDK 25 | 紧凑对象头转正（JEP 519），默认开启 |

## 9.2 G1 下堆不再是物理连续

```text
G1：堆 = 等大 Region 数组，E/S/O/H 是贴标签的逻辑分代（见 [12_gc_collectors](12_gc_collectors.md)）
┌──┬──┬──┬──┐
│E │O │S │H │ … × 约 2048 个
└──┴──┴──┴──┘
```

## 9.3 分代 ZGC：兜兜转转又回到分代

- JDK 21 实验（JEP 439）→ JDK 23 默认（JEP 490）→ JDK 24 移除非分代模式。
- 动机：非分代 ZGC 每次处理**全堆**，大堆高分配率下 CPU 开销与回收滞后明显；重新引入 young/old 分代后，新生代可高频小范围回收。
- 堆视角：Eden/Survivor/Old 的逻辑分代**回来了**，但物理上仍是 Page 池 + 颜色指针（见 [13_low_latency_gc](13_low_latency_gc.md)）。

## 9.4 紧凑对象头（Compact Object Headers）

- JEP 450（JDK 21，实验）→ JEP 519（JDK 25，转正）。
- 对象头从 **96 bit 缩到 64 bit**，小对象整体可缩到 16B；典型实测堆内存节省 10%~20%。
- 对"堆里全是小对象"的服务是纯收益；细节见 [9_object_layout](9_object_layout.md)。
