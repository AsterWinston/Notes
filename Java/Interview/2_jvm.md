本篇覆盖 JVM 内存模型、垃圾回收、类加载、执行引擎与线上调优排查的面试题。结构为主题块 → 详细分块 → 题目三级标题；新增题目追加到对应分块即可。

# 1. 运行时内存区域

## 1.1 内存区域总览与线程私有区

### 1.1.1 JVM 内存区域有哪些？各自作用？

**结论**：线程私有 3 块 + 线程共享 2 块 + 堆外 1 块——程序计数器、虚拟机栈、本地方法栈；堆、方法区；直接内存。

| 区域 | 归属 | 存什么 | 异常 |
|------|------|--------|------|
| 程序计数器 | 私有 | 当前线程执行的字节码行号；分支/循环/异常恢复靠它接续 | **唯一无 OOM 的区域**（Native 方法时为 undefined） |
| 虚拟机栈 | 私有 | 每次方法调用创建一个**栈帧** | 深度超限 StackOverflowError；栈动态扩展失败 OOM |
| 本地方法栈 | 私有 | 服务 native 方法，HotSpot 与虚拟机栈合并实现 | 同上 |
| 堆 | 共享 | 对象实例与数组，GC 主战场 | OutOfMemoryError: Java heap space |
| 方法区（逻辑） | 共享 | 类元信息、运行时常量池、静态变量、JIT 代码缓存；JDK7 前永久代实现，JDK8 起元空间（本地内存） | Metaspace OOM |
| 直接内存 | 堆外 | NIO DirectByteBuffer，不受 -Xmx 限制 | 也可能 OOM（`Direct buffer memory`） |

**易错**：字符串常量池 JDK7 起在**堆**里、不在方法区/元空间；局部变量表在栈上，但 `int i` 的值在栈帧里、`new` 出来的对象在堆里——"引用在栈、对象在堆"。

### 1.1.2 栈帧里有什么？局部变量表和操作数栈是干嘛的？

方法执行一次 = 压入一个栈帧，包含四部分：

1. **局部变量表**：参数 + 方法体内局部变量，单位是变量槽（Slot）——基本类型各占 1 槽（long/double 占 2 槽），引用类型占 1 槽（压缩指针下 4 字节）。容量编译期确定（Code 属性 max_locals），**this 也占第 0 槽**（实例方法）。
2. **操作数栈**：字节码计算的"工作台"（LIFO）——`a+b` 就是把两个操作数压栈、iadd 弹出求和、结果再压栈；方法返回值也先放这里。
3. **动态链接**：指向运行时常量池中本方法的符号引用，支撑多态的**运行时绑定**。
4. **方法返回地址**：正常返回（遇到 return）或异常退出时回到调用方的位置。

> **面试追问**：栈帧随方法调用压栈，方法结束整个弹出——局部变量表里的引用消亡，对象本身还在堆上，等可达性分析判死刑。

### 1.1.3 StackOverflowError 和 OOM 有什么区别？栈上会出现 OOM 吗？

- **StackOverflowError**：栈深度超过 `-Xss` 限制——无限递归、超深递归、栈帧过大（方法参数/局部变量太多）。
- **OutOfMemoryError**：申请内存时区域无法扩展——堆、元空间、直接内存、GC overhead limit。

虚拟机栈**也能抛 OOM**：栈帧创建成功但动态扩展时申请不到内存（HotSpot 中栈会随递归增长触发 SOE 在先，OOM 罕见但规范允许）。常见异常文案定位区域：`Java heap space`（堆）、`Metaspace`（类元数据撑爆，如动态代理生成海量类）、`unable to create new native thread`（OS 线程数/内存限制，不是堆不够）、`Direct buffer memory`（堆外）。

### 1.1.4 为什么程序计数器是唯一没有 OOM 的区域？

PC 只保存"下一条字节码指令的地址"——定长、线程私有、随线程生死，容量恒定不增长，没有分配失败的可能。它是线程上下文的"书签"：线程被 CPU 切换走再回来，靠 PC 恢复到上次执行位置；执行 native 方法时值为 undefined。

## 1.2 堆、方法区与直接内存

### 1.2.1 HotSpot 的堆是怎么分代的？为什么这么分？

```text
堆（-Xms/-Xmx）
├─ 新生代 Young（默认 1/3）：Eden : S0 : S1 = 8 : 1 : 1
│    └─ Eden 90%，S0/S1 各 5%（From/To 角色轮换）
└─ 老年代 Old（默认 2/3）：长期存活对象、大对象
```

分代依据**对象存活率统计**："朝生夕灭"的对象占 98% 以上——新生代用复制算法只处理 10% 空间成本，老年代存活率高用标记-整理/清除。字符串常量池 JDK7 起在堆中。

**参数**：`-Xmn` 定新生代、`-XX:NewRatio=2` 老:新=2:1、`-XX:SurvivorRatio=8` Eden:Survivor=8:1。G1/ZGC 下物理分代被 Region 取代（见 2.3.3/2.3.5），但逻辑分代仍在。

### 1.2.2 JDK8 为什么移除永久代、改用元空间？

永久代（PermGen）的问题：

1. **大小难设定**：它挤占堆的管理逻辑，类元数据需求与应用规模、动态代理数量相关，`MaxPermSize` 设小频繁 `java.lang: PermGen space` OOM，设大浪费堆内存；
2. **GC 回收效率差**：永久代 Full GC 才回收类，Major GC 慢；
3. **JRockit 融合**：HotSpot 与 JRockit 合并，JRockit 本无永久代，统一实现。

元空间（Metaspace）改用**本地内存**：默认无上限（`-XX:MaxMetaspaceSize` 可封顶），类加载越多可自然增长；GC 时按类加载器粒度卸载。运行时常量池、字符串常量池随之挪进堆。**方法区是规范概念、永久代/元空间是 HotSpot 的实现**——"JDK8 方法区还存在吗？"答：规范上存在，实现换成元空间。

### 1.2.3 字符串常量池在哪个内存区域？各版本如何变化？

- JDK6：字符串常量池在**永久代**，`intern()` 把堆字符串**复制**一份进池。
- JDK7+：池移到**堆**，`intern()` 无则直接把堆对象引用入池（见 1_java.md 1.3.5）。
- 字面量编译期入池、`new String` 在堆；`-XX:StringTableSize` 控制池容量（默认 6 万+ 桶的哈希表，不是"一个池数组"），大表可调大防单桶过长。

**追问**：静态变量放哪？——随类元数据在元空间（JDK8+），它引用的对象在堆；`Class` 对象本身在堆。

### 1.2.4 什么是直接内存？为什么它快？

直接内存 = 堆外本地内存，`ByteBuffer.allocateDirect()` 经 Unsafe 分配，不受 `-Xmx` 约束、上限 `-XX:MaxDirectMemorySize`（默认约等于 Xmx）。

**快的原因**：堆内 Buffer 做 Socket/File IO 时，数据要先从 Java 堆拷到 native 临时缓冲（堆内存可能被 GC 移动，不能直接交给 OS）；直接内存地址固定，内核 DMA 直达，**省掉一次堆↔native 拷贝**。

**代价**：分配/释放贵（走系统调用 + Cleaner 回收，不受 GC 周期直接管理，堆内只留个"壳"）；泄漏表现为物理内存耗尽或 `Direct buffer memory` OOM。**Netty 大量使用池化直接内存**正是同时吃掉"零拷贝"与"分配贵"两端（见 1_java.md 5.2.4）。

## 1.3 对象的创建、布局与访问

### 1.3.1 对象创建过程是怎样的？（从 new 到可用）

1. **类加载检查**：new 指令先确认类已加载，否则触发加载；
2. **分配内存**：堆规整（指针碰撞，Serial/ParNew）或空闲列表（CMS 这类带碎片收集器）；并发安全靠 **TLAB + CAS**（见 1.3.4）；
3. **初始化零值**：字段先填默认零值，保证未赋值字段也可安全访问；
4. **设置对象头**：MarkWord（hash/年龄/锁状态）+ Klass 类型指针；
5. **执行 `<init>`**：按程序员写的构造器赋值——`new` 的最后一步。

**易错**：`int x = 1` 中 x 是栈上局部变量不是堆对象；对象头在 JDK6 后 hash 延迟计算；创建过程与 1_java.md 1.1.5 的"创建对象的方式"是两个考点。

### 1.3.2 对象的内存布局是怎样的？

三段 + 填充（64 位默认，压缩开）：

1. **对象头 MarkWord**（8 字节）：无锁态存 hashCode(25bit)、GC 分代年龄(4bit，最大 15)、偏向标志/锁标志(2bit)——**同一块内存按锁状态复用**（锁升级时含义变化，见 1_java.md 3.3.1）；
2. **类型指针**：指向元空间 Klass 元数据，开压缩类指针（`-XX:+UseCompressedClassPointers`）后 4 字节；
3. **实例数据**：字段本体，排列规则"宽字段在前、同宽聚拢、子类可填父类空洞"；
4. **对齐填充**：对象大小凑齐 8 字节倍数（CPU 缓存行友好 + 地址低位可作标记位）。

> **面试追问**：空 new Object() 多大？——16 字节：头 12（MarkWord 8 + Klass 4）+ 填充 4。数组多一个 length 字段再加 4。

### 1.3.3 如何定位访问对象？句柄访问和直接指针访问有什么区别？

- **句柄**：栈上 reference → 句柄池（对象实例数据指针 + 类型数据指针）→ 堆对象。GC 移动对象只改句柄，引用不变；代价是**多一次间接寻址**。
- **直接指针**：reference → 对象（头里带类型指针）。**一次寻址，访问快**，GC 移动时要更新所有引用。

**HotSpot 选直接指针**——配合分代 GC 移动成本可控，且用压缩类指针把类型引用降到 4 字节；代价换来性能。这也是"对象引用到底存的是什么"的答案：存的是（逻辑上的）对象地址。

### 1.3.4 TLAB 是什么？如何优化对象分配？

TLAB（Thread Local Allocation Buffer）：JVM 在 **Eden 中为每个线程预划的私有分配区**。

**动机**：所有线程共享 Eden 分配指针，裸用 CAS + 失败重试在高并发分配下竞争激烈。TLAB 让分配变成"私有区内指针碰撞（top += size），无锁"；TLAB 耗尽才 CAS 申请新块，把同步频率从"每次 new"降到"每填一块"。

流程：申请 TLAB（CAS）→ 区内无锁分配 → 剩余空间装不下当前对象时：太大走共享 Eden CAS 分配，否则把剩余作废、申请新 TLAB。

参数：`-XX:+UseTLAB`（默认开）、`-XX:TLABSize`、`-XX:+PrintTLAB`。**Eden 仍按"堆规整/不规整"决定指针碰撞/空闲列表**；TLAB 解决的是并发分配竞争，二者不是一回事。


# 2. 垃圾回收

## 2.1 对象存活判定与引用

### 2.1.1 如何判断对象是否存活？GC Roots 有哪些？

- **引用计数法**：引用 +1 / 失效 -1，归零回收。循环引用（A↔B）无法处理，还要每次赋值维护计数——Java 不采用。
- **可达性分析**：从 GC Roots 沿引用链遍历，不可达即可回收。

**GC Roots 清单**：

1. 虚拟机栈栈帧局部变量表引用的对象（在跑的方法里的参数/局部变量）；
2. 方法区静态变量引用的对象；
3. 方法区运行时常量引用的对象；
4. 本地方法栈 JNI 引用的对象；
5. JVM 内部常驻：Class 对象、类加载器、常驻异常对象（NPE/OOM）、synchronized 持有的锁对象、JMXBean 等。

**追问**：finalize() 能不能自救？——可回收判定后，实现了 finalize 且未执行过的对象先进 F-Queue，由低优先级 Finalizer 线程执行，若重新被引用则"复活"一次；只有一次机会，JDK9 废弃——答题时提一句"理论可救、实践别用"。

### 2.1.2 强引用、软引用、弱引用、虚引用分别是什么？

| 引用 | 回收时机 | 典型用途 |
|------|---------|---------|
| 强引用（默认） | 可达即不回收，宁可 OOM | 普通代码 |
| 软引用 SoftReference | **内存不足（OOM 前）才回收** | 内存敏感缓存（图片缓存） |
| 弱引用 WeakReference | 遇到下一次 GC 即回收 | WeakHashMap、ThreadLocalMap 的 key |
| 虚引用 PhantomReference | 随时，`get()` 恒 null | 唯一目的：**回收时收到通知**（Cleaner 管理堆外内存释放） |

配套 **ReferenceQueue**：被回收对象对应的引用对象入队，供程序做后置清理（NIO 的 DirectByteBuffer 正是靠 Cleaner+虚引用在 GC 时释放堆外内存）。软/弱引用还能配合缓存失效、监听器自动摘除；`WeakHashMap` key 弱引用——key 无其他强引用时条目自动清理（注意 value 反过来引用 key 会阻止回收）。

## 2.2 GC 算法与并发标记

### 2.2.1 GC 算法有哪些？什么是分代收集？

| 算法 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| 标记-清除 | 标记可达→清除其余 | 简单 | **碎片**、两遍慢 |
| 标记-复制 | 存活对象拷到另一块，原块整清 | 无碎片、快 | 浪费空间；吞吐随存活率升高恶化 |
| 标记-整理 | 标记后存活对象向一端移动 | 无碎片、不费额外空间 | 移动耗时（STW 长） |

**分代收集**不是新算法，而是**按区域特点配算法**：新生代 98% 朝生夕灭 → 复制只付 2% 拷贝成本，且 HotSpot 用 Eden+S0+S1 的 8:1:1 只浪费 10% 空间（而非教科书对半）；老年代存活率高 → 标记-清除（CMS）或标记-整理（Serial Old/Parallel Old）。

**追问**：新生代为什么留 2 个 Survivor？——复制算法要两块交替；纯对半浪费 50%，改为 Eden 干活、两个 5% 小Survivor 轮换，空间利用率 90%。为什么是 8:1:1？——经验值，绝大多数对象活不过第一轮 GC。

### 2.2.2 什么是三色标记法？CMS 和 G1 如何解决漏标？

并发标记把对象染色：**白**（未访问，结束仍白=可回收）、**灰**（自身已扫、引用未扫完）、**黑**（自身与引用全扫完）。

**漏标两条件**（同时满足才出错）：① 黑色对象新增了指向白色对象的引用；② 灰色对象到该白色对象的路径被删除。

- **CMS 增量更新（破①）**：写屏障记录"黑对象新引用了白对象"，重新标记阶段把黑的变回灰再扫。代价：重新标记要重扫黑对象引用，STW 偏长。
- **G1 SATB 原始快照（破②）**：标记开始拍"对象图快照"，写屏障把**被断开的旧引用**记下来，快照里的白对象本轮算活。代价：**浮动垃圾**（快照后才断根的死对象本轮不回收）。

ZGC 用类似 SATB 的快照语义 + 读屏障自愈（见 2.3.5）。

### 2.2.3 什么是写屏障？卡表、Remembered Set 是干什么的？

**写屏障**：引用赋值（Store）时插的一段钩子代码（不是读屏障！ZGC 的读屏障在 Load）。用途一：并发标记维护颜色（增量更新/SATB 的旧值、新值记录）；用途二：**跨代/跨区引用记账**。

- **卡表（Card Table）**：堆按 512B 切"卡页"，卡表字节数组标记 dirty——老年代对象引用了新生代对象时写屏障把对应卡置脏。Minor GC 因此不必扫整个老年代找 GC Roots，只扫脏卡。
- **RSet（G1）**：每个 Region 记"谁引用了我里面的对象"（points-into），Region 数量可控（约 2048 个）代价是 RSet 占堆 5%~20% 内存 + 写屏障更重。

追问：CMS 也有卡表（-XX:+CMSIncrementialMode 时代），为什么 G1 还要 RSet？——G1 回收任意 Region 子集，必须知道每个 Region 的外部入引用；传统分代只需"老→新"一个方向。

## 2.3 收集器详解与选型

### 2.3.1 Serial、ParNew、Parallel 收集器各有什么特点？

| 收集器 | 区域 | 算法 | 线程 | 特点 |
|--------|------|------|------|------|
| Serial | 新生代 | 复制 | 单 | STW 简单高效，无线程交互开销；Client 默认、小内存容器适用 |
| Serial Old | 老年代 | 标记-整理 | 单 | CMS 失败时的兜底（Concurrent Mode Failure 退化目标） |
| ParNew | 新生代 | 复制 | 多 | Serial 的多线程版，**只为搭配 CMS 而生** |
| Parallel Scavenge | 新生代 | 复制 | 多 | **吞吐量优先**，`-XX:MaxGCPauseMillis/-XX:GCTimeRatio` 可调，JDK8 默认 |
| Parallel Old | 老年代 | 标记-整理 | 多 | PS 的老年代搭档，JDK8 默认组合 |

选型记忆：吞吐（批处理/计算型）→ Parallel；低延迟（Web）→ CMS(旧)/G1(现在)；内存极小 → Serial。

### 2.3.2 CMS 收集器的工作原理和缺点？

老年代收集器，目标**最短停顿**。四阶段：

1. **初始标记**（STW，短）：只标 GC Roots 直接关联；
2. **并发标记**：与用户线程并发遍历对象图（最长）；
3. **重新标记**（STW）：修正并发期间变动（增量更新重扫）；
4. **并发清除**：与用户线程并发清死对象。

四大缺点：

1. **CPU 敏感**：并发阶段默认占 (CPU+3)/4 个核；
2. **浮动垃圾**：并发标记后新产生的垃圾本轮不管，需预留并发分配空间（`-XX:CMSInitiatingOccupancyFraction` 92%，太晚启动会失败）；
3. **Concurrent Mode Failure**：老年代放不下晋升对象 → 退化 Serial Old 单次长时间 STW（线上事故常见来源）；
4. **碎片**：标记-清除固有，触发 Full GC 前被迫整理（`-XX:+UseCMSCompactAtFullCollection`）。JDK9 弃用、14 移除（JEP 363）——面试答"历史方案，现存系统仍大量在跑"。

### 2.3.3 G1 收集器的核心特点和原理？

JDK9+ 默认，面向服务端、可预测停顿：

1. **Region 化**：堆划 1~32MB 等大 Region（约 2048 个），Eden/Survivor/Old/Humongous 只是**逻辑角色**，取消物理分代；Region 间回收独立。
2. **停顿可预测**：给每个 Region 算"回收价值"（垃圾比例×耗时建模），`-XX:MaxGCPauseMillis=200` 内挑价值最高的 Region 集合回收（Garbage **First** 之名）。
3. **RSet + 写屏障**：解决跨 Region 引用扫描（2.2.3）。
4. **SATB**：并发标记漏标方案。

**工作流**：Young GC（Eden 满，复制到 S + 可能晋升）→ 堆占用到 IHOP(45%) 触发**并发标记周期** → **Mixed GC**（Young + 一批价值高的 Old Region，最多连续 8 次回收 5% 以上才停）→ 实在不行 Full GC（Serial 单线程时代已过，现多线程但很慢，要避免）。大对象直接进 Humongous，可能提前触发并发标记。

### 2.3.4 G1 和 CMS 的区别？

| 维度 | CMS | G1 |
|------|-----|----|
| 目标 | 最短停顿 | 可预测停顿 + 吞吐兼顾 |
| 结构 | 物理分代（老年代整区） | Region 逻辑分代，任意子集回收 |
| 算法 | 标记-清除（**碎片**） | 整体标记-整理、局部复制（**无碎片**） |
| 漏标方案 | 增量更新（写屏障记新值） | SATB（写屏障记旧值，浮动垃圾换稳定） |
| 失败模式 | Concurrent Mode Failure → Serial Old 长停顿 | 堆满 → Mixed 提速或 Evacuation Failure |
| 适用 | <8GB 中小堆（历史） | 大堆、现代默认 |
| 开销 | 并发线程抢 CPU | RSet/写屏障更重，额外占 5~20% 内存 |

**一句话**：G1 = 把"整代回收"改造成"按价值挑 Region 回收 + 碎片整理"，用可调停顿模型换掉 CMS 的不可控。

### 2.3.5 ZGC 的工作原理和核心特点是什么？

JDK11 实验、15 生产可用、21 分代模式：停顿 **<1ms** 且**与堆大小无关**（最大 16TB）。

**两项核心技术**：

1. **着色指针**：64 位指针高几位塞 GC 状态（Finalizable/Remapped/Marked0/Marked1 等），**元数据在指针上而非对象头**——查状态不访对象、天然支持"对象已搬走但引用未更新"的过渡态；压缩指针因此不可用。
2. **读屏障**：应用线程 Load 引用时检查颜色位，指向旧地址则"顺手修正"为新地址（**自愈**）——对象转移的修正工作摊到每次访问，**并发整理无需 STW**。

流程：并发标记 → 并发转移（读屏障自愈）→ 引用处理；Shenandoah 同目标但用 **Brooks 转发指针**（对象头旁挂新地址）实现并发整理。代价：屏障开销吞吐降 5~10%、不支持压缩指针内存略高、JDK21 前无分代（弱分代假说浪费，21+ 分代 ZGC 缓解）。

### 2.3.6 各 JDK 版本垃圾收集器怎么选？

| JDK | 默认 | 推荐 |
|-----|------|------|
| 8 | Parallel | 批处理 Parallel；低延迟 Web 用 CMS（或考虑升 11+） |
| 11 | G1 | G1 通用；ZGC 实验可试 |
| 17 | G1 | G1 通用；ZGC 生产可用，低延迟大堆首选 |
| 21 | G1 | G1 通用；ZGC 分代（`-XX:+ZGenerational`）超大堆；虚拟线程配 G1/ZGC 均可 |

**决策矩阵**：吞吐型（离线/批处理）→ Parallel；通用在线服务 → G1；32GB+ 大堆或对 P999 延迟敏感 → ZGC；内存 <2GB 容器小应用 → Serial（省线程/内存开销）。

G1 关键调优参数：`MaxGCPauseMillis`（目标别设太小，否则 Region 变小吞吐掉）、`InitiatingHeapOccupancyPercent=45`、`G1HeapRegionSize`。

## 2.4 GC 触发与对象晋升

### 2.4.1 什么时候发生 Young GC、Major/Full GC？

- **Young GC (Minor GC)**：Eden 满。复制存活到 Survivor，年龄++。
- **Major GC**：老年代回收（日常口径常与 Full GC 混用）。
- **Full GC 触发条件**（以 Serial/PS/CMS 视角）：
  1. **老年代空间不足**（晋升前检查失败、分配担保失败）；
  2. **元空间不足**（触发前先 Young GC 清类加载死引用，无效才 Full）；
  3. `System.gc()` 显式调用（生产建议 `-XX:+DisableExplicitGC` 屏蔽，NIO DirectBuffer 场景例外，配 `-XX:+ExplicitGCInvokesConcurrent`）；
  4. CMS Concurrent Mode Failure / 空间分配担保失败；
  5. G1：并发标记周期兜不住、Evacuation Failure；
  6. 历史：永久代满、堆 dump 触发（jmap -dump 不加 --live 可避免）。

> **行业实践**：`-XX:+DisableExplicitGC` 别一刀切——Netty/DirectByteBuffer 靠 System.gc 触发 Cleaner 释放堆外，屏蔽不当会堆外内存持续涨到 OOM。

### 2.4.2 对象晋升老年代有哪些规则？

1. **年龄阈值**：每熬过一次 Young GC 年龄 +1，默认 `-XX:MaxTenuringThreshold=15`（4bit 上限）；
2. **动态年龄判定**：Survivor 中**同龄对象体积和 > Survivor 一半**（TargetSpacePercent 默认 50%）→ ≥该年龄直接晋升，不必等到 15；
3. **大对象直入老年代**：`-XX:PretenureSizeThreshold`（Serial/ParNew 生效）或数组/字符串超 Region 一半（G1 进 Humongous）——防 Eden 复制大对象折腾；
4. **空间分配担保**：Young GC 前检查老年代连续空间 ≥ 新生代全部对象（或历次晋升平均值），不够先 Full GC——CMS 下"担保失败"即退 Serial Old。

**追问**：年龄到 15 怎么存？——MarkWord 只有 4bit，GC 时按 -1 饱和处理；调大无意义。


# 3. 类加载与字节码

## 3.1 类加载过程与 Class 文件

### 3.1.1 类加载过程有哪几个阶段？各阶段做什么？

**加载 → 验证 → 准备 → 解析 → 初始化**（解析可后置，"先初始化后解析"的动态连接允许存在）：

1. **加载**：全限定名取字节流（来源不限于 class 文件：jar、网络、动态生成、加密）→ 方法区存运行时数据结构 → 堆生成 Class 对象作入口；
2. **验证**：文件格式（CAFEBABE、版本号）、元数据（继承合法性、final）、字节码（类型检查、跳转不越界）、符号引用；
3. **准备**：static 变量分配内存赋**零值**；`static final` 编译期常量直接赋定值；
4. **解析**：符号引用 → 直接引用；
5. **初始化**：执行 `<clinit>()`——静态赋值 + static 块按书写顺序合并；JVM 保证**线程安全**（一个线程执行，其余等待），且父类 `<clinit>` 先执行；接口的 `<clinit>` 不要求父接口先执行（只有用到父接口常量才触发）；
6. 使用、7. 卸载（类加载器可回收 + 堆内无实例 + Class 对象无引用三条同时满足，见 -XX:+ClassUnloadingWithConcurrentMark）。

主动/被动引用触发时机见 3.1.3 与 1_java.md 1.10.1。

### 3.1.2 Class 文件里有什么？为什么魔数是 CAFEBABE？

八字段结构：**魔数**（4B，CAFEBABE——"咖啡 baby"，识别文件类型且十六进制肉眼可辨；替代扩展名防止伪装）、**主次版本号**（JDK 与 class 版本对应：52=8、55=11、61=17、65=21，`UnsupportedClassVersionError` 即版本超前）、**常量池**（字面量 + 符号引用，索引 0 不用从 1 开始，"运行时常量池"的源头）、访问标志（public/final/abstract）、类索引/父类索引/接口索引集合、字段表集合、方法表集合（Code 属性里才是字节码 + max_stack/max_locals）、属性表集合（SourceFile、LineNumberTable…）。

**追问**：`javap -v` 看过常量池吗？——方法名/字段名以 UTF-8 常量存储，这是"混淆只改符号、不影响运行"的基础。

### 3.1.3 类初始化的触发时机？主动引用和被动引用有什么区别？

**主动引用（6 种，触发初始化）**：new/读写字段（非常量）、调静态方法、反射 `Class.forName`、初始化子类连带父类、启动含 main 的类、MethodHandle 的 REF_*Static 解析；接口有 default 方法时其实现类初始化需先初始化接口。

**被动引用（不触发）**：

```java
// 1. 子类引用父类静态字段 → 只初始化父类
Parent.value;           // Child 不初始化
// 2. 定义数组 → 不触发元素类初始化
Parent[] a = new Parent[10];
// 3. 编译期常量 → 值已内联进调用方常量池
String s = Const.CONST; // Const 不初始化
```

**易混**：`Class.forName(name)` 初始化，`ClassLoader.loadClass(name)` 只加载不初始化；`-XX:+TraceClassLoading` 观察。与 1_java.md 1.10.1 同源，那边侧重执行顺序，这里侧重 JVM 规范。

## 3.2 双亲委派模型

### 3.2.1 什么是双亲委派模型？有什么好处？

四层加载器 + 委托规则"先让父类加载器试，父加载不了才自己来"：

```text
Bootstrap（<JAVA_HOME>/lib，C++ 实现，加载 java.lang.*）
  ▲
Extension / JDK9+ Platform（lib/ext → 模块系统内建）
  ▲
Application（classpath）
  ▲
Custom（自定义）
```

`loadClass()` 流程：查缓存（已加载直接返回）→ 委托 parent → parent 为 null 到 Bootstrap → 都失败自己 findClass。

**两大好处**：

1. **唯一性**：同名类只被加载一次，且由最"信得过"的层级加载——类的**身份 = ClassLoader + 全限定名**，不同加载器加载的"同一个类"强转互认失败；
2. **防篡改**：自写 `java.lang.String` 永远轮不到 AppClassLoader 加载（Bootstrap 已返回核心库的），保护 API 基座。

### 3.2.2 什么场景要打破双亲委派？SPI 是怎么打破的？

双亲委派的局限：**父加载器加载的类，看不见子加载器范围里的类**（依赖方向倒了）。

1. **SPI/JDBC**：`DriverManager`（rt.jar，Bootstrap）要用 classpath 里的 MySQL 驱动实现——用**线程上下文类加载器**（TCCL，默认 AppClassLoader）加载实现类；`ServiceLoader.load` 内部 `Thread.currentThread().getContextClassLoader()` 即此。本质是"线程借给你，加载器也借给你"。
2. **Tomcat**：两个 WebApp 都有 `com.foo.User` 要互不可见——WebappClassLoader **先自己加载**（排除 JVM 核心后），违反委派换隔离。
3. **OSGi**：网状委派，按 bundle 依赖决定委托方向。
4. **热部署/字节码增强**：换一版 class = 新 ClassLoader 重新加载（旧加载器整体卸载才干净）；Arthas redefine 是另一条路。
5. **JDK9 模块化**：Platform 加载器与命名/未命名模块改造，TCCL 行为微调（`-XX:+TraceClassLoading` 可查）。

### 3.2.3 Tomcat 的类加载机制是怎样的？

Web 应用四件套：Bootstrap → System → **Common**（Tomcat 自身共享）→ 每个 WebApp 独立 **WebappClassLoader**（优先加载 `/WEB-INF/classes` → `/WEB-INF/lib` → JAR，排除 `javax.*`/JDK 核心仍委派）；另有 **Shared** 可配置跨应用共享层。

效果：应用之间类隔离（同名不同版本可共存，双排包冲突在 Tomcat 层被物理隔离）、应用可见容器公共类、容器看不见应用类。**追问**：为什么你的应用里能放 javax.servlet 而不冲突？——加载器边界 + 排除规则，重定位（shade）是另一派解法。

### 3.2.4 如何自定义 ClassLoader？

重写 **findClass()**（保留 loadClass 的委派/缓存骨架；只有要打破委派才覆写 loadClass）：

```java
public class MyLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] data = readAndMaybeDecrypt("/dir/" + name + ".enc"); // 自定义来源/解密
        return defineClass(name, data, 0, data.length);             // 字节流 → Class
    }
}
```

动机：热部署（新实例化加载器加载新 class）、类隔离（插件系统）、加密类加载、单元测试 mock 容器。**验证唯一性**：`MyLoader.loadClass("X") == AppLoader.loadClass("X")` 为 false，实例互相 instanceof 失败。

# 4. 执行引擎与运行时优化

## 4.1 解释器与 JIT

### 4.1.1 Java 一条语句是怎么执行的？（从源代码到运行）

`javac` 编译成 `.class`（字节码 + 常量池）→ 运行时数据区加载（类加载五阶段）→ **执行引擎混合执行**：热点未形成前**解释器**逐条翻译（启动快、执行慢）；方法/循环热度（计数器）达标后 **JIT（C1/C2）编译为本地机器码**缓存执行，配合内联、逃逸分析等激进优化；假设失效则**反优化（deoptimize）回退解释**重新收集画像。native 方法走 JNI/外函数。

这条主线能串起：常量池解析、动态链接、分层编译、SafePoint、JIT 对锁的优化。`-Xint`/`-Xcomp`/`-Xmixed` 三种模式。

### 4.1.2 JIT 的 C1 和 C2 有什么区别？什么是分层编译？

| 特性 | C1 | C2 |
|------|----|----|
| 目标 | 编译快、启动快 | 优化激进、峰值高 |
| 优化 | 简单内联、profiling 采集 | 完全内联、逃逸分析、锁消除/粗化、公共子表达式、去虚化 |
| 编译耗时 | 短 | 长（后台线程） |

**分层编译（JDK8 默认）**：Tier0 解释（无 profiling）→ Tier1/2 C1（轻/中 profiling）→ Tier3 C1（全 profiling）→ Tier4 C2（用画像做假设优化）。热度阈值：方法调用计数 + 回边计数器（循环）触发。**反优化**：C2 假设（如类型单一）不再成立，代码回退 Tier0 重采——"JIT 预热"现象的根源。

### 4.1.3 逃逸分析是什么？能带来哪些优化？

JIT 判定对象作用域是否**逃出方法/线程**：

1. **栈上分配**（未逃逸）：对象直接在栈帧分配，方法结束随栈帧销毁，无 GC 压力；
2. **标量替换**（默认开启，真正主力）：拆成局部变量 `x=1; y=2;`——对象压根不存在（`-XX:+EliminateAllocations`）；
3. **同步消除**：线程本地对象上的 synchronized 直接删除（-XX:+EliminateLocks）。

参数 `-XX:+DoEscapeAnalysis`（JDK8+ 默认开）。经典演示：循环内 new 短命对象，开/关逃逸吞吐差一个量级。局限：分析本身费编译时间；对象被存储到字段/返回/作参数传出即逃逸；数组暂不支持标量替换。**与 TLAB 的关系**：一个管"分配快"，一个管"少分配"，互补。

## 4.2 运行时协作机制

### 4.2.1 什么是用户态和内核态？与锁升级有什么关系？

CPU 特权级：用户态（进程受限，只能跑应用指令）/ 内核态（全权，处理系统调用、中断、线程调度）。切换靠**系统调用/中断/异常**，成本几百~几千周期（保存恢复上下文）。

与锁的关系：偏向锁、轻量级锁的 CAS 与自旋**全在用户态**；膨胀为重量级锁后挂起线程进 EntryList、unpark 唤醒依赖 OS 互斥量——**用户态↔内核态切换**，这是重量级锁慢的根本原因，也是"自旋换切换"的权衡点（见 1_java.md 3.3.1）。

### 4.2.2 什么是 SafePoint（安全点）？什么时候进入？

能"干净暂停线程"的指令位置集合——此处线程状态完整（寄存器/栈帧），可精确枚举 GC Roots。**STW 的本质是等所有线程跑进最近的安全点**，而非任意时刻掐停。

布置位置：方法返回后、循环回边（否则死循环无法进入 safepoint）、异常跳转处。触发进入的场景：GC 各 STW 阶段、偏向锁撤销、线程 dump、code cache 满、反优化、JVM 挂起。

**排查点**：GC 日志 `Time spent in kernels`/`sync` 偏长 → 某线程迟迟不到安全点（超大数组拷贝、counted loop 被 C2 优化掉检查），`-XX:+PrintSafepointStatistics` 定位。`-XX:SafePointPollingPage`/LL 轮询方式（自旋线程用 signal-based 更快）。


# 5. 调优参数与线上排查

## 5.1 参数配置

### 5.1.1 常用 JVM 参数有哪些？

| 参数 | 作用 | 备注 |
|------|------|------|
| `-Xms2g -Xmx2g` | 堆初/最大 | 生产设相同 |
| `-Xmn1g` | 新生代大小 | G1 下一般不手动固定 |
| `-Xss512k` | 线程栈大小 | 默认 1M；调小可跑更深递归/更多线程 |
| `-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m` | 元空间初/最大 | 后者不设=无上限 |
| `-XX:MaxDirectMemorySize=512m` | 直接内存上限 | 不设≈Xmx |
| `-XX:+UseG1GC / +UseZGC / +UseParallelGC` | 选收集器 | |
| `-XX:MaxGCPauseMillis=200` | G1 停顿目标 | |
| `-XX:InitiatingHeapOccupancyPercent=45` | G1 并发标记触发 | |
| `-Xlog:gc*:file=gc.log`（JDK9+） | GC 日志 | JDK8：`-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:` |
| `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/` | OOM 自动 dump | **生产必配** |
| `-XX:+ExitOnOutOfMemoryError` | OOM 直接退出 | 容器编排环境推荐（重启比半死不活好） |
| `-XX:+UseContainerSupport` | 感知容器限额 | 8u191+/10+ 默认开 |

### 5.1.2 为什么建议 -Xms 和 -Xmx 设置成一样？

堆扩容/缩容都有代价：

1. 扩容向 OS 要连续内存 + 触发 GC 决策波动，缩容后又要再扩——**内存抖动**；
2. 某些路径的扩容伴随 Full GC（如 Metaspace/老年代联动检查），带来不可预期停顿；
3. 固定大小让**GC 行为可复现**（晋升阈值、Region 分配稳定）。

配套：`-XX:+AlwaysPreTouch` 启动即物理触页（启动慢几秒，运行期零缺页中断——低延迟场景标配）。容器环境注意 Xmx ≤ limit 的 3/4~7/8（留堆外：线程栈、元空间、DirectBuffer、GC 结构、JIT code cache），否则被 OOMKilled。

### 5.1.3 为什么建议元空间初始大小和最大大小设置成一样？

与堆同理，另有元空间特性：

1. `MetaspaceSize` 实为"**触发 GC 的水位线**"而非初始分配大小——到线触发 Full GC 整理类并抬水位，反复抬升=反复 Full GC；把它直接设到终值（=MaxMetaspaceSize）消除这个爬坡过程；
2. 不设上限时，动态代理/Groovy/JSP/反射生成类会**无界增长吃光物理内存**，整机 OOM 比"快速失败重启"更糟；
3. 本地内存碎片：动态向 OS 申请小块元空间，长跑后碎片化。

配置示例：`-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m`。

### 5.1.4 JVM 调优的实践经验怎么落地？（方法论）

**结论**：调优不是"背一串参数"，而是**目标 → 基线 → 假设 → 单变量实验 → 灰度验证**的闭环；且 80% 的"该调优"其实是"该修代码"（泄漏、大对象、无上限缓存先排掉）。

**先定目标三角（取舍，不是全都要）**：

| 业务形态 | 第一指标 | 典型选择 |
|----------|----------|----------|
| 在线接口（交易/网关） | P99 停顿 | G1 + `MaxGCPauseMillis`，堆别太大；极致低延迟上 ZGC |
| 离线吞吐（批处理/Flink 类） | 吞吐量 | Parallel GC 或 G1 放宽停顿目标、堆给足 |
| 长驻稳定性（中间件） | 不重启、无 Full GC 风暴 | 容量留余量 + GC 日志常开 + OOM 自动 dump |

**五步落地**：

1. **压测定基线**：没有压测报告就没有"调优前"，GC 频率/停顿分布/分配速率（`jstat -gcutil`、JFR allocation 事件）先量化；
2. **按数据画像选收集器与容量**：堆大小≈老年代稳态存活集的 2~3 倍、年轻代按对象朝生夕死比例定；新生代太小→晋升风暴，太大→Young GC 变长；
3. **保守设参上线**（"三件套"先配齐再谈调优）：`-Xms=-Xmx`、`-Xlog:gc*:file=gc.log`、`-XX:+HeapDumpOnOutOfMemoryError`；容器环境 Xmx ≤ limit 的 3/4~7/8（见 5.1.2）；
4. **单变量迭代**：一次只动一个参数，对比 GC 日志同流量下的指标（GC 时间占比 <5%、Young GC 停顿 P99 达标为健康线）；工具链见 5.2（Arthas/JFR/MAT）；
5. **灰度验证再全量**：调优参数和代码一样走灰度，观察一个完整流量周期（含晚高峰/批量任务）再推广。

**常见手段清单（按性价比排序）**：堆容量与分代配比 → 收集器迁移（CMS→G1 的触发点：堆 >8G 或碎片导致 Full GC；G1→ZGC：停顿目标压不到 10ms 级）→ G1 的 `MaxGCPauseMillis`（压太低会牺牲吞吐、Region 拷贝不过载不了）与 IHOP 调整 → 治理大对象/缓存上限（软上限 + 淘汰策略，比任何参数都管用）→ 元空间/直接内存按观测值固化。

> **面试追问**："说一个你实际调过的案例？"——答题用 STAR：现象（Full GC 频繁 / P99 毛刺与 GC 停顿对齐）→ 定位（5.3 排查链路）→ 动作（改收集器/容量/代码，说明为什么）→ 结果（量化前后对比）。没案例的人背参数，有案例的人讲数据——后者才是这题的区分点。

## 5.2 诊断工具

### 5.2.1 JVM 故障诊断工具有哪些？各自用途？

**JDK 命令行**：`jps`（列 Java 进程）、`jstat -gcutil PID 1000 10`（各代占比/GC 次数耗时趋势）、`jinfo`（查看/动态改参数）、`jmap -histo[:live] / -dump:format=b,file=x.hprof`、`jstack PID`（线程快照，死锁/CPU 高）、`jcmd`（集大成：Thread.print/GC.heap_dump/VM.flags）、`jhat`（已过时，MAT 替代）。

**可视化/商业**：JVisualVM（+VisualGC 插件看分代动态）、JMC+JFR（**<1% 开销**的生产常开飞行记录，JDK11 开源）、MAT（堆 dump 支配树/泄漏嫌疑报告）、JProfiler/YourKit（开发期火焰图+调用树）。

**第三方**：**Arthas**（在线诊断首选）、BTrace、async-profiler（火焰图，safepoint 偏差小于 JFR 场景）。

**选择建议**：现场还在（线程/CPU/GC 状态）→ Arthas/jstack；需要离线分析对象 → dump+MAT；长期趋势 → GC 日志+JFR；接口级耗时 → trace 火焰图。

### 5.2.2 Arthas 常用命令有哪些？

| 命令 | 作用 |
|------|------|
| `dashboard` | 实时面板：线程/内存/GC |
| `thread -n 3` / `thread -b` | CPU Top3 线程 / 检测死锁 |
| `trace com.xx.Service method '#cost>100'` | 方法内部调用路径+逐段耗时，找慢点 |
| `watch class method '{params,returnObj,throwExp}' -n 5` | 观察出入参与异常 |
| `tt -t class method` | 时空隧道：录制每次调用现场，可**回放** |
| `stack class method` | 反查"谁调用了我" |
| `jad` / `mc` + `redefine` | 反编译确认线上代码版本 / 热替换（不能增删字段方法） |
| `ognl '@Class@STATIC_FIELD'` | 读写静态字段、调 Spring Bean 方法 |
| `profiler start/stop --file x.html` | async-profiler 集成，火焰图 |
| `heapdump --live` / `vmoption` | 导出堆 / 查看改 VM 参数 |

典型链路：dashboard 定位异常维度 → thread/trace/watch 取证 → 修复验证 → `reset` 清理增强（防字节码增强常驻拖慢性能）。

## 5.3 线上问题排查

### 5.3.1 Full GC 频繁怎么排查？

1. `jstat -gcutil PID 1000` 看趋势：老年代回收后又**快速涨满** → 分配速率问题或泄漏；老年代**水位阶梯抬升** → 泄漏/缓存膨胀；元空间贴顶 → 类加载泄漏（动态代理、Groovy、重复 ClassLoader）；
2. 分类处理：
   - Young GC 也频繁且对象大量晋升 → 大对象/缓存 list 直接进老年代 → 看代码批量 add 不清理、`-XX:PretenureSizeThreshold`、堆或新生代偏小；
   - `System.gc()` 显式触发 → GC 日志看调用栈，配 DisableExplicitGC（留意堆外副作用）；
   - 元空间类加载器数量持续涨（`jstat -class`）→ 热部署/代理滥用；
3. `jmap -histo:live`（**注意 :live 会触发一次 Full GC**，低峰用）找 Top 类；
4. dump + MAT：Leak Suspects、支配树、按类分组看引用链（经典：static Map、ThreadLocal 未 remove、无界队列积压）；
5. 复盘：调容量/换收集器/加缓存上限。

### 5.3.2 CPU 100% 怎么排查？

1. `top` 找高 CPU 进程 PID；
2. `top -Hp PID` 找高 CPU **线程** TID（或 `ps -mp PID -o tid,time`）；
3. `printf "%x\n" TID` 转十六进制 nid；
4. `jstack PID | grep -A 30 nid`——看该线程栈：
   - 在**业务代码死循环**（RUNNABLE + 自家栈帧）；
   - 在 **GC 线程**（VM Thread/GC task thread 高 CPU）→ 转 5.3.1 Full GC 流程；
   - 大量 BLOCKED/deadlock → `jstack` 末尾 deadlock 段或 Arthas `thread -b`；
   - 正则回溯（catastrophic backtracking）、频繁装箱/字符串拼接也常见。
5. 快路径：Arthas `thread -n 3` 直接给 Top CPU 线程与栈；`profiler` 出火焰图看聚合热点。

### 5.3.3 线上 OOM 怎么排查？

OOM 分型定位：

| 报错 | 方向 |
|------|------|
| `Java heap space` | dump→MAT 支配树：业务高峰数据 vs 泄漏 |
| `GC overhead limit exceeded` | GC 98% 时间回收 2% 内存——同样是堆不够/泄漏前兆 |
| `Metaspace` | 类加载泄漏：代理/脚本引擎/ClassLoader 反复创建 |
| `unable to create native thread` | 线程数失控（池滥用/每请求起线程）或 ulimit/内存不足，与堆无关 |
| `Direct buffer memory` | 堆外：Netty 泄漏检测、MaxDirectMemorySize 偏小 |
| 容器 `OOMKilled` 无 dump | RSS 超 limit：堆+堆外+栈总账没管住 |

流程：确认 `-XX:+HeapDumpOnOutOfMemoryError` 已生效拿 dump → MAT 三件套（Leak Suspects / 支配树 / Histogram 对比基线 dump）→ 定位持有者 → 修代码/配上限；无法复现的偶发泄漏用 `jcmd GC.heap_dump` 定期留档 + JFR Old GC 事件回看。

### 5.3.4 内存泄漏和内存溢出的区别？

- **内存溢出（OOM）**：申请时不够用，报错可见——**结果**；
- **内存泄漏（Leak）**：该回收的对象被无谓引用着回收不掉——**原因**，泄漏累积才溢出。

泄漏不必然快速溢出（慢漏型堆一点点涨）；溢出也不必源于泄漏（瞬时大对象、堆配小）。Java 常见泄漏点：static 集合只增不清、未 remove 的 ThreadLocal、未关闭的连接/流/监听器、内部类隐式持有外部类、缓存无上限无过期、自定义 equals/hashCode 错误导致 HashSet 去重失败、ClassLoader 泄漏（热部署）。
