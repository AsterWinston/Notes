# 1. 为什么需要低延迟收集器

## 1.1 停顿与堆大小的矛盾

先回忆 GC 成本的基本模型（详见 [11_gc_algorithm](11_gc_algorithm.md)）：

```text
标记成本       ∝ 存活对象规模（Live Set）
复制/整理成本  ∝ 存活对象规模（垃圾越多反而越省事）
Full GC 成本   ∝ 整堆
→ 堆越大、存活集越大，STW 越长 —— 但业务恰恰需要大堆
```

```text
堆增长 ──▶ 单次回收扫描/搬移量增长 ──▶ 停顿变长
   ▲                                      │
   └────── 应用要装下缓存/工作集，堆必须大 ◀┘
                    这就是矛盾
```

| 收集器 | 矛盾暴露方式 | 结局 |
|--------|-------------|------|
| CMS | 标记-清除**不搬移** → 碎片累积；并发回收追不上分配 → concurrent mode failure | 退化 Serial Old 单线程整理，大堆停顿**几十秒级**（JDK 9 废弃、JDK 14 移除，见 [12_gc_collectors](12_gc_collectors.md)） |
| G1 | 用 Region 粒度的 CSet "赌"停顿目标，赌输时（浮动垃圾 + 碎片 + 分配尖峰）退化 Full GC | Full GC 自 JDK 10（JEP 307）起**并行化**，但几十 GB 堆仍要秒级 |
| ZGC / Shenandoah | 把"扫描"和**搬移**都放到并发阶段，用屏障兜底引用正确性 | STW 只剩根扫描，停顿与堆大小**解耦** |

关键点：**传统收集器的停顿 ∝ 存活集大小；低延迟收集器的停顿只 ∝ GC Roots 规模**（线程栈、JNI 句柄——通常很小且稳定）。

## 1.2 什么业务在为一毫秒付费

| 场景 | 典型延迟约束 | 一次 GC 停顿的代价 |
|------|-------------|-------------------|
| 金融撮合 / 行情推送 | P99.9 < 5ms，报单超时约 50ms | 错过行情、订单堆积、风控误触发 |
| 广告竞价（RTB） | 全链路硬超时 100ms，Java 侧仅约 30ms | 直接丢标，真金白银损失 |
| 电信 / 5G 信令 | P99 < 10ms，可用性 99.999% | 超时引发上游重试风暴 |
| 游戏对局服务器 | tick 帧率 30~60fps（16~33ms/帧） | 全服玩家同时卡顿 |
| 电商 / 支付网关 | P99 < 100ms + SLA 赔付条款 | 超时雪崩、赔付 |

这些场景的共同点：**平均延迟毫无意义，尾部延迟（P99/P999）就是收入**。调优目标从"少 GC"变成"STW 恒定亚毫秒"。

## 1.3 停顿目标的分级

```text
P99 目标          可行方案（JDK 21 基线）
──────────────────────────────────────────────────────────
≤ 200 ms          G1 / Parallel 默认配置即可
≤ 10 ms           G1（调低 MaxGCPauseMillis；但停顿下限受最小 Region
                  粒度与根扫描成本限制，目标未必总能兑现）
≤ 1 ms (含 P999)  分代 ZGC / Shenandoah（并发转移）
```

经验法则：

- 堆 ≤ 8GB 且 P99 ≤ 20ms：G1 精调通常够用，别急着换收集器。
- 堆 > 8GB 且 P99 < 10ms，或要求 P999 也压住：低延迟收集器（本章主题）。
- "停顿与堆大小无关"不是免费午餐——代价是**常驻屏障指令 + 并发阶段抢 CPU**，见 2.3 与第 4 节。

# 2. ZGC（Z Garbage Collector）

## 2.1 设计目标与版本演进

| 特性 | 说明 |
|------|------|
| 核心目标 | 初版（JEP 333）：停顿 ≤ 10ms、吞吐损耗 ≤ 5%；转正后实际：**亚毫秒**（典型 0.05~0.5ms），分代模式吞吐进一步改善 |
| 支持堆大小 | 8MB ~ 4TB（JDK 11，42 位地址）→ ~16TB（JDK 14 扩展到 44 位地址） |
| STW 组成 | 仅 Pause Mark Start / Mark End / Relocate Start 三段，∝ 根集合大小 |
| 回收粒度 | Page（small/medium/large 三档，KB~MB 级），按分配压力自适应切分与合并 |
| JDK 11 | 实验性引入（JEP 333） |
| JDK 15 | 转正生产可用（JEP 377） |
| JDK 17 | 扩展到 macOS / Windows（此前仅 Linux） |
| JDK 21 | **分代 ZGC 实验性**（JEP 439） |
| JDK 23 | **分代成为 ZGC 默认**（JEP 490），非分代进入废弃流程 |
| JDK 24 | 移除非分代模式，只剩分代 |

```text
演进主线：
JDK 11 ──── JDK 15 ──────── JDK 17 ──────── JDK 21 ───── JDK 23 ──── JDK 24
实验版      生产可用         macOS/Windows   分代实验      分代默认     非分代移除
≤10ms目标   亚毫秒实测                       (JEP 439)   (JEP 490)
```

## 2.2 核心技术一：着色指针（Colored Pointers）

### 2.2.1 64 位指针的位域布局

x86-64 平台 64 位虚拟地址实际只用到低 48 位；ZGC 把高位借来存 GC 元数据。JDK 14 之后的 44 位地址布局：

```text
 63      52 51          50        49       48        47        46    44 43              0
┌─────────┬───────────┬─────────┬────────┬────────┬───────────┬───────┬─────────────────┐
│ 未使用  │ relocated │ remapped│ marked0│ marked1│finalizable│ 保留/ │   44 位地址      │
│         │ 已转移    │ 待修正  │ 标记0  │ 标记1  │ 仅终结器  │ 元数据│  (堆上限 16TB)   │
└─────────┴───────────┴─────────┴────────┴────────┴───────────┴───────┴─────────────────┘
          └──────────────── 颜色位（GC 状态） ────────────────┘
```

- **42 位地址时代（JDK 11~13）**：堆上限 2^42 = 4TB；JDK 14 放宽到 44 位（16TB）。位宽每 +1，堆上限翻倍——这是"着色"吃掉地址位的直接代价。
- marked0 / marked1 **互斥使用**（同一周期只认一种为"好颜色"），见 2.2.3。
- finalizable 位标记"仅被 finalizer 类引用"的对象；保留/元数据位在较新版本与紧凑对象头（JEP 450/519，见 [9_object_layout](9_object_layout.md)）配合使用，布局细节随版本变化。

### 2.2.2 元数据在指针里，不在对象头里

| 对比项 | 传统收集器（G1/CMS 标记位） | ZGC（着色指针） |
|--------|---------------------------|----------------|
| GC 状态存哪 | 对象头 Mark Word / 堆内位图 | **引用（指针）本身** + 堆外转置位图/转发表 |
| 改颜色成本 | 逐对象写头，O(存活对象数) | 换"当前好颜色"的定义，**O(1)**（整堆位不动） |
| 搬移对象 | 头里的 GC 信息要跟着搬/修正 | 对象体与 GC 状态解耦，**可任意搬** |
| 副作用 | 对象头吃缓存行 | 应用读取任何引用都要"洗颜色"（→ 读屏障，见 2.3） |

一句话：着色指针让"标记状态"从 **O(对象数)** 的操作变成 **O(1)** 的开关，这是并发回收能成立的前提。

### 2.2.3 颜色轮转：marked0 / marked1 为什么成对出现

```text
周期 N:   unmarked ──并发标记──▶ marked0 ──转移──▶ (remapped) ──下个周期洗白──▶ marked1(好)
周期 N+1: 定义"好颜色 = marked1"，新标记产出 marked1，无需把整堆颜色复位
```

- 若只有一个标记位，每个周期结束要把**全堆**颜色清零（O(堆大小) 的 STW 或漫长并发）。
- 两个位交替：翻转一个全局标志即完成"复位"。这就是 **tri-color 标记的并发实现技巧**（详见 [11_gc_algorithm](11_gc_algorithm.md) 三色抽象）。

## 2.3 核心技术二：读屏障（Load Barrier）

### 2.3.1 为什么指针着了色就必须有读屏障

着色 + 并发转移联合产生一个必然后果：**应用线程手里的引用可能是"过期颜色"**——对象已经被 GC 搬到新位置，旧引用还指着老入口。GC 又不能停下来修完全堆引用，于是把修复推迟到"引用被真正使用的那一刻"：

```text
应用线程读字段:  ref = obj.field          ← 可能拿到过期/脏色指针
                │
                ▼ 读屏障（JIT 内联在每条 load 之后）
   ┌────────────────────────────────────────────┐
   │ fast path（≈3~5 条指令，分支几乎必然预测命中）│
   │   if ((ref & COLOR_MASK) != GOOD_COLOR)     │
   │        goto slow_path                       │
   │   ref &= ADDRESS_MASK;  // 洗掉颜色，使用    │
   ├────────────────────────────────────────────┤
   │ slow_path（罕见，自愈式）：                  │
   │   1. 剥掉颜色位得到物理入口                  │
   │   2. 查转发表/标记表：对象搬了吗？            │
   │   3. 得到新地址，把"干净指针"**写回原字段**   │
   │   4. 返回 ref（同一字段下次直接走快路径）     │
   └────────────────────────────────────────────┘
```

伪代码（对应原文件的示意，扩充为带掩码判断的完整版本）：

```text
oop load_barrier(oop* field) {
    oop ref = *field;                          // 裸读（带颜色）
    if ((ref & MASK) != EXPECTED_COLOR) {      // "脏"了？
        ref = slow_path(ref);                  // 查转发表 → 新地址
        *field = ref | EXPECTED_COLOR;         // 修复写回 = 自愈（fixup）
    }
    return (ref & ADDRESS_MASK);               // 剥色后使用
}
```

### 2.3.2 屏障的成本画像

| 维度 | 结论 |
|------|------|
| 快路径成本 | 与 null check / load 指令融合后约 1~5 条指令；分支 99%+ 走"颜色正确"一侧，预测友好 |
| 慢路径频率 | 只在"引用指向已搬移对象"时触发，摊销后低 |
| 何时变贵 | **指针密集型遍历**（图、大 HashMap 扫描）：每条边都过屏障；C2 能对循环做屏障消除（barrier elision）缓解 |
| 吞吐损耗 | 初版目标 ≤5%；非分代实测略低于 G1 约 5%~15%；分代后收窄 |

### 2.3.3 与写屏障（SATB）对比——两者解决不同的问题

| | SATB 写屏障（CMS / G1 并发标记） | ZGC 读屏障 |
|--|----------------------------------|-----------|
| 触发时机 | 引用字段**写**入 | 引用字段**读**取 |
| 解决的问题 | **标记正确性**（并发标记期间引用被删，防漏标） | **转移正确性**（对象搬走，引用过期） |
| 生效窗口 | 仅并发标记期间开启 | 全程常驻（任何 load 都在读路径上） |
| 成本形态 | 记队列、批量消费，突发在 remark | 指令级常驻，摊销平稳 |
| 副作用 | 浮动垃圾（本轮删掉的引用也当活的） | 引用过期窗口 + 地址位宽损失 |

> G1 是"SATB 写屏障 + RSet"，Shenandoah 经典方案是 Brooks 指针（对象头内转发，见 3.2）；只有 ZGC 把宝全押在读屏障上。

## 2.4 回收阶段：三个微停顿 + 一串并发阶段

### 2.4.1 阶段表（非分代 ZGC，JDK 15~21 形态）

| # | 阶段 | 模式 | 耗时量级 | 成本 ∝ | 做什么 |
|---|------|------|----------|--------|--------|
| 1 | Pause Mark Start | **STW** | ~0.02ms | 根集合 | 扫描线程/JNI 根，启动并发标记（分代模式同时开始疏散新生代） |
| 2 | Concurrent Mark (+Preclean) | 并发 | 数十 ms~秒 | 存活集 | 遍历对象图着色，读屏障保证不漏 |
| 3 | Pause Mark End | **STW** | ~0.02ms | 根集合 | 收尾标记队列，宣告标记完成 |
| 4 | Concurrent Process Non-Strong References | 并发 | 数 ms | 弱/软/虚引用数 | 四阶段引用处理（详见 [10_gc_basics](10_gc_basics.md)） |
| 5 | Concurrent Reset Relocation Set | 并发 | 数 ms | 页表 | 剔除"已全空/引用已修正"的候选页 |
| 6 | Concurrent Select Relocation Set | 并发 | 数十 ms | 页表 | 按"垃圾密度"挑 CSet 页（JDK 11~14 曾是 STW，后移为并发） |
| 7 | Pause Relocate Start | **STW** | ~0.02ms | 根集合 | 修正指向 CSet 的根引用 |
| 8 | Concurrent Relocate | 并发 | 数十~数百 ms | CSet 存活对象 | 复制对象、建转发关系；旧引用由读屏障**懒修正** |

### 2.4.2 时间线全景

```text
应用线程: ━━━━━━━━━━┃PMS┃━━━━━━━━━━━━━━━━━━━━━━━━┃PME┃━━━━━━━━━━━━━━━┃PRS┃━━━━━━━━━━▶
                   ↕0.02ms                     ↕0.02ms             ↕0.02ms   全程仅 3 次微停顿
GC 并发:            └──── Concurrent Mark ────┘└─ 弱引用/Reset/Select ─┘└─ Concurrent Relocate ─┘
```

- 三个停顿阶段干的都是同一件事：**扫根**。根数量由线程数决定，与堆大小无关 → 停顿恒定。
- 其余阶段与 mutator 并行，抢的是 CPU 配额（ConcGCThreads），不是时间片停顿。
- 分代 ZGC 阶段名基本沿用，但新生代回收只扫 young 集合 + 记忆集，周期更快更密。

## 2.5 并发重映射 → 并发转移：从"懒修引用"到"查转发表"

### 2.5.1 初版方案：靠"并发重映射"洗白过期引用

非分代 ZGC 的搬移（Relocate）结束后，指向旧位置的引用**并不立即修正**，只是被染成 remapped 色；等下一次 GC 周期的 **Concurrent Remap** 阶段再整堆扫描逐个洗白：

```text
周期 N Relocate:  obj 从页A搬到页B，页A入口留"转发"标记
                 全堆仍有一堆引用 ──▶ 页A（remapped 色，过期）
周期 N+1 Remap:  整堆扫描，把这些引用逐条改指页B  ← 迟到一整个周期！
```

由此产生的三个结构性问题：

| 问题 | 机理 |
|------|------|
| 引用过期窗口长 | 最坏跨一个完整 GC 周期，页 A 无法及时整页腾空 → 可回收空间滞后 |
| 每周期整堆扫描 | 哪怕 95% 垃圾都在新生代规模的对象上，也要付全堆标记的 CPU 税 |
| 多地址视图（multi-mapping） | 同一物理页映射到多个虚拟"颜色视图"，回收滞后需要缓冲；**监控 RSS 虚高**、堆内存有放大 |

### 2.5.2 分代方案：转发表实现"真并发转移"

分代 ZGC（源自 Badlands 论文的设计思路，JEP 439 落地）改为：**搬移时把"旧→新"写进堆外转发表（forwarding table），读屏障慢路径当场查表修复**，不再等下个周期整堆重扫。

```text
对象搬移:  old ──copy──▶ new
           forwarding[old] = new          ← 外部表，不占对象头

应用读到过期引用 old:
  barrier slow_path → 查 forwarding → 得到 new → 写回字段   （同一周期内即修复）
```

| 维度 | 非分代（JDK 11~22 默认） | 分代（JDK 21 实验 / 23 默认） |
|------|--------------------------|-------------------------------|
| 引用修复时机 | 下一个周期的 Concurrent Remap | 读屏障慢路径**当场**查转发表 |
| 回收范围 | 每周期全堆 | 新生代高频小回收 + 老年代低频周期 |
| 地址视图 | 多视图 multi-mapping（RSS 统计失真） | 简化，不再依赖多视图 |
| CPU 开销 | 整堆标记税，高分配率下明显 | 大部分周期只扫 young + 记忆集 |
| 吞吐损耗 | 比 G1 低约 5%~15% | 明显收窄，堆占用也更省 |

## 2.6 分代 ZGC：兜兜转转又回到分代

### 2.6.1 为什么"停顿无关堆大小"的 ZGC 还需要分代

- **弱分代假说仍然成立**（见 [6_heap](6_heap.md) 1.3）：九成垃圾朝生夕死，全堆并发标记把它们反复扫过，纯浪费 CPU。
- 全堆周期太长 → 新生代垃圾要活等一个周期才被回收 → 堆必须留大缓冲；分代后 young 高频回收，内存占用和回收滞后同时改善。
- 结论：低延迟（并发、屏障）+ 分代（缩小每轮工作集）是**正交收益的叠加**，不是理念倒退。

### 2.6.2 机制要点

| 要点 | 说明 |
|------|------|
| 逻辑分代回来了 | young/old 重新成为回收单位；物理上仍是 Page 池 + 着色指针 |
| 跨代引用 | 老年代→新生代的引用由**屏障顺带写入记忆集**（读屏障实现 RemSet 是 Badlands 设计的核心创意），young GC 不再扫全堆 |
| 晋升 | 幸存者从 young 页复制进 old 页，与 6_heap 的晋升语义一致（见 [6_heap](6_heap.md) 9.3） |
| 停顿仍亚毫秒 | young 周期同样只有微秒~亚毫秒级 STW |

### 2.6.3 版本与开关速查

```bash
# JDK 21 / 22：分代是实验选项，必须显式打开
java -XX:+UseZGC -XX:+ZGenerational ...

# JDK 23（JEP 490）：分代即默认，ZGenerational 可省略
java -XX:+UseZGC ...

# JDK 24：非分代模式已移除，无开关可关
```

## 2.7 NUMA 感知

```text
Socket0 ──本地内存 N0        Socket1 ──本地内存 N1
   │ 分配/转移优先在 N0          │ 优先在 N1
   └────── UPI 跨节点访问慢 ~1.5~2x ──────┘
```

- ZGC 的堆页（Page）按 NUMA 节点分组管理：对象**优先分配在本线程所在节点**；并发转移时目标页也尽量选同节点 → 避免搬完变成远程访问。
- 只对**多插槽大机器**有意义；单 socket / 云主机通常单节点，开了也无收益。

```bash
-XX:+UseNUMA        # ZGC（部分能力 Serial 也支持）下生效
```

## 2.8 参数与日志

### 2.8.1 常用参数

```bash
-XX:+UseZGC                 # 启用 ZGC
-XX:+ZGenerational          # JDK 21/22 显式开分代；JDK 23+ 默认；JDK 24+ 已无意义
-Xms31g -Xmx31g             # ZGC 扩缩容是并发的，但仍建议相等以免边界抖动
-XX:SoftMaxHeapSize=24g     # 软上限：平时压在 24g 内，突发可临时越界（比直接调小 Xmx 稳）
-XX:ConcGCThreads=8         # 并发标记/转移线程数，默认按堆与 CPU 自动
-XX:ParallelGCThreads=16    # 三个 STW 阶段用的线程数（∝ 根扫描速度）
-XX:+UseNUMA                # NUMA 感知（见 2.7）
-XX:ZCollectionInterval=0   # 定时整堆回收（秒），0=关；非分代时代参数
-XX:ZAllocationSpikeTolerance=5  # 分配尖峰预测容差，越小越早启动 GC；非分代时代参数
-Xlog:gc*:file=gc.log:time,uptime,tags:filecount=5,filesize=50M
```

> 注意：分代 ZGC 精简/替换了部分非分代参数，个别旧参数在分代模式下被忽略。以 `java -XX:+UseZGC -XX:+PrintFlagsFinal -version | grep -i z` 实测为准，别抄旧文章。

### 2.8.2 ZGC 日志：保留原示例并逐行解读

```text
[0.123s] GC(1) ZGC Pause Mark Start 0.143ms
[15.234s] GC(1) ZGC Concurrent Mark 1456.789ms
[15.236s] GC(1) ZGC Pause Mark End 1.234ms
[17.891s] GC(1) ZGC Concurrent Process Non-Strong References ...
[17.893s] GC(1) ZGC Pause Relocate Start 0.089ms
[20.456s] GC(1) ZGC Concurrent Relocate 2567.123ms
```

| 行 | 解读 |
|----|------|
| Pause Mark Start 0.143ms | STW，仅扫根；>1ms 通常说明根太多（线程数爆炸）或机器被抢占 |
| Concurrent Mark 1456ms | 并发耗时 ∝ 存活集；**应用此刻仍在跑**，别拿它算停顿 |
| Pause Mark End 1.234ms | 同上，微停顿；示例值已偏大 |
| Process Non-Strong References | 弱/软/虚/终结引用处理，大量软缓存会拖长这一段 |
| Pause Relocate Start 0.089ms | 修正指向 CSet 的根 |
| Concurrent Relocate 2567ms | 并发搬移；耗时高 = CSet 里活对象多 = 碎片压力大 |

**Allocation Stall 出现说明什么：**

```text
[23.456s] GC(7) Allocation Stall (worker-7) 12.345ms
```

- 含义：应用线程分配不到内存，**被迫停下来等 GC**——这不再是 GC 线程的并发耗时，而是实打实的业务停顿，ZGC 的"亚毫秒承诺"已被击穿。
- 定位与处置：堆偏小 / 分配尖峰太陡 / ConcGCThreads 被应用抢光。三板斧：加大 `-Xmx`、调低触发水位、降应用分配率。完整排障流程见 [14_gc_tuning](14_gc_tuning.md) 与 [22_troubleshooting](22_troubleshooting.md)。

# 3. Shenandoah

## 3.1 概述

| 项 | 说明 |
|----|------|
| 主导方 | Red Hat，长期随其 OpenJDK 发行版打磨；**JDK 15 起进入 OpenJDK 主干**（-XX:+UseShenandoahGC 直接可用） |
| 核心目标 | 停顿与堆大小无关（亚毫秒~10ms 档），并发标记 + **并发压缩** |
| 经典机制 | Brooks 转发指针（对象头内，见 3.2） |
| 分代状态 | 长期非分代；分代化在其上游持续演进（落后于 ZGC 一步） |
| 平台 | x86-64 / AArch64 / PPC64 等，跟随各发行版打包 |
| 定位 | 与 ZGC 同赛道：低延迟 + 大堆；在 RHEL/Ubuntu 系 OpenJDK 上开箱即用是它的分发优势 |

## 3.2 关键技术：Brooks Pointer（转发指针）

### 3.2.1 对象布局：头部多一个字段

GC 状态不放在指针里（ZGC），而是**塞进对象自己身上**——每个对象开头多存一个"转发指针"，未搬移时指向自己：

```text
传统对象:                      Shenandoah 对象（经典 Brooks 模式）:
┌──────────────┐               ┌───────────────┐
│ Mark Word    │               │ forwarding ───┼──▶ 自身（未搬移）
│ Klass Pointer│               ├───────────────┤      或 ──▶ 新副本
│ 字段...      │               │ Mark Word     │
└──────────────┘               │ Klass Pointer │
                               │ 字段...       │
                               └───────────────┘
                               代价：每对象 +8B（一个指针宽）
```

### 3.2.2 并发转移流程（保留原示意并展开）

```text
转移线程:  1. 在空闲区分配新副本 → 2. 拷贝对象 → 3. CAS 把旧对象 forwarding 写成新地址
                                          （CAS 失败 = 已被别人抢着搬了，放弃本次拷贝）
应用线程:  读: p = obj.field;  if (p->forwarding != p) p = p->forwarding;   // 间接到新体
           写: obj.field = ref; 若 ref 已搬移则先转发再写入                  // 读写各设屏障
```

### 3.2.3 演进：从 Brooks 指针向"读引用屏障"收敛

经典 Brooks 方案有两个痛点：每对象 8B 常驻开销；**所有解引用都要多一跳内存间接**。近年 Shenandoah 重构了转移机制，改用与 ZGC 同族的 load-reference barrier（屏障 + 堆外信息）替代裸 Brooks 指针——两家思路合流，"对象头内 vs 指针外"这条传统分界线正在模糊。具体机制随 JDK 版本演进，以 OpenJDK 源码/发行说明为准。

## 3.3 回收阶段

| # | 阶段 | 模式 | 做什么 |
|---|------|------|--------|
| 1 | Init Mark | STW（微） | 扫根，启动并发标记 |
| 2 | Concurrent Mark | 并发 | SATB 遍历整堆 |
| 3 | Final Mark | STW（短） | 消费 SATB 队列、统计 Region 存活度、**选定 CSet** |
| 4 | Concurrent Cleanup | 并发 | 直接回收全空 Region |
| 5 | Init Update Refs | STW（微） | 准备根引用，进入转移窗口 |
| 6 | Concurrent Update Refs | 并发 | ★ 真正的转移：复制 CSet 存活对象 + 设 forwarding + 修正引用 |
| 7 | Final Update Refs | STW（微） | 根引用统一切到新对象 |
| 8 | Concurrent Cleanup | 并发 | 整 Region 释放旧空间 |

- 两次"并发清理"职责不同：第 4 步清"全空页"，第 8 步清"搬空的旧页"——Region 级整体释放，**无碎片**（这是与 CMS 的本质区别）。
- 三个 STW 都只碰根，停顿与堆大小无关。

## 3.4 参数

```bash
-XX:+UseShenandoahGC                    # 启用（需所用 JDK 构建包含 Shenandoah）
-XX:ShenandoahGCHeuristics=adaptive     # 触发策略: adaptive(默认)/static/compact/aggressive
-XX:ShenandoahMinFreeThreshold=10       # 空闲低于 10% 即开始回收
-XX:ShenandoahPacingMaxDelay=5          # 分配节流(pacing)时应用线程最多等 5ms，防雪崩
-Xlog:gc*:file=gc.log:time,tags
```

> 与 ZGC 的 Allocation Stall 对应：Shenandoah 用 **pacing（分配节流）** 提前减速线程，尽量不让人等 GC——理念差异见 3.5 对比表。

## 3.5 与 ZGC 的核心差异

| 维度 | ZGC | Shenandoah（经典） |
|------|-----|---------------------|
| 搬移期 GC 信息放哪 | **指针外**：着色位 + 堆外转置表/转发表 | **对象头内**：Brooks forwarding 指针 |
| 对硬件/地址空间的要求 | 需要闲置高位地址（44/48 位 VA 限制堆上限） | 无特殊要求，任何普通 64 位指针 |
| 屏障 | 读屏障（常驻快路径） | 经典为读写双查 forwarding；新版向 load-ref barrier 收敛 |
| 每对象内存开销 | 0（头里不加字段） | +8B/对象（Brooks 模式） |
| 分配压力应对 | Allocation Stall（硬等） | Pacing（软节流） |
| 分代 | JDK 23 起默认分代 | 长期非分代，分代化仍在演进 |
| 最大堆 | 4/16TB（着色位宽决定） | 无 16TB 这类设计上限，典型几十~几百 GB |
| 平台覆盖 | JDK 17+ Linux/Windows/macOS | OpenJDK 15+ 主干，发行版覆盖广 |
| NUMA | 原生感知（2.7） | 无对等能力 |

# 4. 低延迟收集器对比

## 4.1 三者对比总表

| 维度 | G1 | ZGC | Shenandoah |
|------|-----|-----|------------|
| 目标停顿 | ~100ms（MaxGCPauseMillis 可调，大堆下限难压） | <1ms | <10ms（典型可进亚毫秒区） |
| 最大堆 | 几十 GB 舒适区 | 16TB | 几十 GB ~ 数百 GB |
| 核心算法 | Region + RSet + CSet 预测模型 | 着色指针 + 读屏障 + 并发转移 | 并发标记-压缩（Brooks/屏障） |
| 屏障类型 | 写屏障（SATB 卡记） | 读屏障 | 读/写查转发 |
| 分代状态 | 一直分代 | JDK 23 起默认**分代** | 长期非分代 |
| 内存开销 | RSet（约 ~5%，JDK17 起显著下降） | 极少（着色位 + 堆外表） | forwarding 约 8B/对象（经典） |
| CPU 开销 | 中 | 较高（并发阶段 + 常驻屏障） | 较高 |
| 吞吐量 | 最高（三者中） | 略低于 G1；分代后差距收窄 | 略低于 G1 |
| 成熟度 | 非常成熟（JDK 9 起默认） | 成熟，演进极快 | 较成熟 |
| 适用场景 | 通用大堆、默认答案 | 极低延迟 + 大堆 | 低延迟 + 特定发行版/平台 |
| JDK 默认 | JDK 9+ 默认 | 否 | 否 |

# 5. GC 选择：决策矩阵与决策树

## 5.1 决策矩阵

| 约束组合 | 推荐 | 一句话理由 |
|----------|------|-----------|
| 客户端 / 堆 < 200MB | Serial | 内存与线程开销最小 |
| 批处理 / 离线任务，吞吐 > 一切 | Parallel | 无屏障无并发税，GCTimeRatio 直达 99%+ |
| 一般微服务，堆 ≤ 8GB，P99 ≤ 20ms | **G1（默认，别动）** | 综合均衡，运维经验最厚 |
| 堆 8~32GB，P99 < 10ms 且想守住 G1 习惯 | G1 深调（MaxGCPauseMillis=50~100） | 可尝试，但停顿下限可能压不下去 |
| 堆 > 8GB，P99 < 10ms 或 P999 严格 | ZGC | 并发转移，停顿恒定亚毫秒 |
| 同上，但用 Red Hat/Ubuntu 系 JDK 或有 PPC 等平台需求 | Shenandoah | 能力同档，分发覆盖好 |
| **JDK 23+ 大堆低延迟** | **分代 ZGC（默认形态）** | 直接开 `-XX:+UseZGC`，无需纠结参数 |

## 5.2 GC 选择总览（决策树）

```text
你是客户端/小堆（<100MB）？
  → -XX:+UseSerialGC

你是批处理/后台任务，追求吞吐量？
  → -XX:+UseParallelGC  （JDK 8 默认）

你是 Web 服务/API 网关，堆 4~32GB？
  → -XX:+UseG1GC        （JDK 9+ 默认）

你是堆 > 32GB？
  → -XX:+UseG1GC 或 -XX:+UseZGC（JDK23+ 直接分代 ZGC）

你需要亚毫秒级停顿？
  → -XX:+UseZGC 或 -XX:+UseShenandoahGC

拿不定？
  → 保持默认 G1，用 [14_gc_tuning](14_gc_tuning.md) 的数据说话再决定换
```

# 6. 实战：完整一轮 ZGC 日志逐行解读

## 6.1 采集命令

```bash
# ZGC（JDK 21 LTS 分代实验形态；JDK 23+ 去掉 ZGenerational 即为默认分代）
java -Xms31g -Xmx31g -XX:SoftMaxHeapSize=24g \
     -XX:+UseZGC -XX:+ZGenerational \
     -Xlog:gc,gc+heap=info:file=gc.log:time,uptime,tags -jar app.jar

# 对照：Shenandoah（需所用发行版包含）
java -Xms31g -Xmx31g -XX:+UseShenandoahGC \
     -XX:ShenandoahGCHeuristics=adaptive \
     -Xlog:gc*:file=gc.log:time,tags -jar app.jar
```

## 6.2 一轮完整 GC（示意，JDK 17 非分代形态）

```text
[12.345s][info][gc,start] GC(5) Pause Mark Start (Major)
[12.345s][info][gc      ] GC(5) Pause Mark Start (Major) 0.021ms
[12.999s][info][gc      ] GC(5) Concurrent Mark 653.902ms
[13.001s][info][gc      ] GC(5) Concurrent Preclean 1.221ms
[13.021s][info][gc      ] GC(5) Pause Mark End 0.019ms
[13.023s][info][gc      ] GC(5) Concurrent Process Non-Strong References 2.003ms
[13.024s][info][gc      ] GC(5) Concurrent Reset Relocation Set 0.512ms
[13.071s][info][gc      ] GC(5) Concurrent Select Relocation Set 47.331ms
[13.071s][info][gc      ] GC(5) Pause Relocate Start 0.015ms
[13.115s][info][gc      ] GC(5) Concurrent Relocate 43.776ms
[13.115s][info][gc,heap ] GC(5) Old Generation: 2556M(4096M)->1536M(4096M)
[13.115s][info][gc      ] GC(5) Garbage Collection (Proactive) 3690M(90%)->1728M(42%)
```

| 行 | 怎么读 |
|----|--------|
| Pause Mark Start (Major) 0.021ms | (Major) 表示含整堆标记；0.021ms 才是**业务真实停顿** |
| Concurrent Mark 653ms | 后台并发，不停业务；持续拉长 → 存活集膨胀信号 |
| Pause Mark End 0.019ms | 第二次微停顿；三段 STW 合计 ≈ 0.055ms |
| Process Non-Strong References | 若飙到几十 ms：软/弱引用过多（大缓存、Class 泄漏），查引用队列 |
| Select Relocation Set 47ms | 挑"垃圾密度高"的页；时长 ∝ 页表规模 |
| Concurrent Relocate 43ms | 搬存活对象；越久说明 CSet 里活的越多，碎片压力越大 |
| Old Generation: 2556M(4096M)->1536M(4096M) | 老年代占用 2.5G→1.5G，容量恒定（并发回收不缩容） |
| Garbage Collection (Proactive) 3690M(90%)->1728M(42%) | 本轮总账：堆占用 90%→42%；Proactive = ZGC 主动发起（非空间告急） |

**读 ZGC 日志的三条纪律**：

1. 只把 `Pause *` 行计入停顿，`Concurrent *` 是 CPU 成本不是延迟成本。
2. `Pause *` 稳定在 0.1ms 内但吞吐掉了 → 屏障/并发线程税，看 4.1 权衡。
3. 出现 `Allocation Stall` = 延迟 SLO 已被击穿，按 2.8.2 三板斧处理（另见 [22_troubleshooting](22_troubleshooting.md)）。

## 6.3 观察停顿分布（回归验证用）

```bash
# 提取所有 STW 行，交给脚本算 P50/P99/P999
grep -E 'GC\([0-9]+\) Pause ' gc.log | awk '{print $NF, $(NF-1)}' | sort -n | tail
# 配合 jstat：GC 行 + Stall 计数一目了然
jstat -gcutil <pid> 1000
```

停顿指标定义与调优闭环见 [14_gc_tuning](14_gc_tuning.md)。
