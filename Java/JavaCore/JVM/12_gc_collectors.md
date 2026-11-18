# 1. 收集器概览

## 1.1 组合矩阵

### 1.1.1 配对关系图

```text
新生代收集器                             老年代收集器
Serial ────────────────────────────▶ Serial Old
ParNew ────────────────────────────▶ CMS
Parallel Scavenge ─────────────────▶ Parallel Old
                                       │
G1 ──▶ (混合回收，横跨新生代和老年代) ──┘
```

### 1.1.2 怎么读这张矩阵

- 连线表示可以组合使用。
- CMS（Concurrent Mark Sweep）在 JDK 9 中已标记为废弃，JDK 14 已移除（与它唯一搭档 ParNew 同批移除）。
- 新生代都使用标记-复制算法，老年代使用标记-清除或标记-整理（算法选择的原因见 [11_gc_algorithm](11_gc_algorithm.md) 第 1~3 节）。
- G1/ZGC/Shenandoah 是**整堆收集器**，不再需要"配对"。

## 1.2 各 JDK 默认收集器

| JDK 版本 | 全局默认 | Ergonomics 例外 |
|----------|----------|-----------------|
| ≤ 7 | Serial（客户端）/ Parallel（服务端，JDK 5 起） | 小内存单核 → Serial |
| **8** | **Parallel Scavenge + Parallel Old** | — |
| **9+** | **G1**（JEP 248） | 物理内存 <1792MB 或单核、或极小堆 → 退回 Serial |
| 11 / 17 / 21（LTS） | G1 | 同上 |

- 容器小规格场景（1~2 核、几百 MB）JDK 9+ 经常自动落回 Serial——评估是否显式 `-XX:+UseParallelGC` 或加大规格；容器内存识别见 [23_container_jvm](23_container_jvm.md)。

## 1.3 ZGC"默认化"路线的澄清

```text
G1 是 HotSpot 自 JDK 9 起的全局默认，至今未变（JDK 21/25 仍默认 G1）。
ZGC 的"默认"只发生在收集器【内部】：
   JDK 11 实验(JEP 333) → JDK 15 转正(JEP 377) → JDK 21 分代模式实验(JEP 439)
   → JDK 23 【ZGC 内部默认切为分代模式】(JEP 490)
   ——前提仍是你手动 -XX:+UseZGC；分代细节见 [13_low_latency_gc](13_low_latency_gc.md)
```

# 2. Serial / Serial Old 收集器

## 2.1 定位

### 2.1.1 特点：简单高效

- **单线程**收集器，GC 时必须暂停所有用户线程（STW）。
- 简单高效，无线程切换开销；是其他收集器在"单核机器"上的退化形态。

### 2.1.2 默认场景

- 适用**桌面应用、嵌入式、小堆**场景；也是 Client 模式 JVM 的新生代默认（JDK 8 时代），并作为小内存设备的 Ergonomics 兜底（1.2 节）。

## 2.2 执行模型

### 2.2.1 时间线

```text
线程0: ──应用────STW────────应用─────▶
线程1: ──应用────STW────────应用─────▶
GC线程:           [GC]
时间轴 ──────────────────────────────▶
```

### 2.2.2 "单线程"的双重含义

- 只用一条 GC 线程；且期间**只有**GC 线程在跑——所以它连"并行缩短停顿"都做不到，但换来最小的运行时开销和最好的紧凑性（嵌入式友好）。

## 2.3 参数

```bash
# 启用 Serial (新生代) + Serial Old (老年代)
-XX:+UseSerialGC
```

| 搭配 | 说明 |
|------|------|
| Serial + Serial Old | 标记-整理；也是 CMS 出 concurrent mode failure 时的退化对象（见 5.3.4） |
| `-XX:+UseSerialGC` | 两代一起指定，无独立老年代开关 |

# 3. ParNew 收集器

## 3.1 定位

### 3.1.1 与 CMS 的唯一搭档关系

- Serial 的**多线程版本**（仅新生代；老年代对应物就是 Serial Old，ParNew 自己不管老年代）。
- 除 Serial 外**唯一能与 CMS 配合**的新生代收集器——历史绑定：CMS 开发时 Parallel Scavenge 已存在却拒绝与它对接，JDK 14 移除 CMS 时 ParNew 一并移除。

### 3.1.2 单核/多核的性能取舍

- 单核性能不如 Serial（线程同步/切换开销），多核优于 Serial。

## 3.2 执行模型

```text
线程0: ──应用──────STW──────应用─────▶
线程1: ──应用──────STW──────应用─────▶
GC线程1:         [  新生代GC  ]
GC线程2:         [  并行回收  ]
```

## 3.3 参数

```bash
# ParNew + CMS (JDK 8 常用；开启 CMS 即自动启用 ParNew)
-XX:+UseParNewGC
-XX:ParallelGCThreads=4       # GC 线程数
```

### 3.3.1 ParallelGCThreads 的默认公式与容器坑

- 默认≈CPU 核数（>8 核时约 8 + (N-8)×5/8，防线程收益递减）。
- 容器里它认的是 **JVM 可见核数**，8u191 前需 `-XX:+UseContainerSupport`/`ActiveProcessorCount`，见 [23_container_jvm](23_container_jvm.md)。

# 4. Parallel Scavenge + Parallel Old 收集器

## 4.1 定位

- **吞吐量优先**的组合：吞吐量 = 用户代码时间 / (用户代码时间 + GC 时间)。
- Parallel Scavenge 新生代（复制、并行）+ Parallel Old 老年代（标记-整理、并行，**JDK 6u23 引入**，此前老年代只能配 Serial Old）。
- **JDK 8 的默认收集器组合**；适合批处理、编译、大数据离线任务等"不在乎单次停顿、在乎总耗时"的场景。
- 与 ParNew 同为"并行 STW"，但设计目标不同：ParNew 为 CMS 服务（降低延迟），Parallel 自带 ergonomics 追求吞吐（停顿可长达秒级）。

## 4.2 三参数逐条深讲

### 4.2.1 开关与目标参数

```bash
# 启用
-XX:+UseParallelGC            # Parallel Scavenge + Serial Old
-XX:+UseParallelOldGC         # Parallel Scavenge + Parallel Old (JDK 8 默认)

# 调优
-XX:MaxGCPauseMillis=200      # 最大 GC 停顿目标（毫秒）
-XX:GCTimeRatio=99            # 吞吐量 = 99/(99+1) = 99%
                               # 即 GC 时间不超过 1%
-XX:+UseAdaptiveSizePolicy    # 自适应调节（默认开启）
                               # JVM 自动调整 Eden/Survivor、晋升年龄等
```

### 4.2.2 语义与陷阱

| 参数 | 语义 | 陷阱 |
|------|------|------|
| `-XX:MaxGCPauseMillis=200` | 停顿**目标**（尽力而为，非承诺）。实现手段是**压缩新生代**让单次 GC 变快 | 无脑调小 → 新生代被压得极小 → GC 频率暴涨 → 吞吐反降、老年代水位涨快 |
| `-XX:GCTimeRatio=99` | 吞吐目标 99%，GC 时间占比 ≤ 1/(1+99) | 与上一参数**互斥**：两个都显式设置时，**MaxGCPauseMillis 生效、GCTimeRatio 失效**（《深入理解 Java 虚拟机》表述："后设置的 GCTimeRatio 不起作用"） |
| `-XX:+UseAdaptiveSizePolicy` | 开启 GC Ergonomics：自动调节新生代大小、Eden:Survivor 比例、晋升阈值 | 见 4.3 |

### 4.2.3 目标组合的直觉

```text
只设 MaxGCPauseMillis     → "快而频"：牺牲吞吐换小停顿（在线服务）
只设 GCTimeRatio          → "慢而稀"：牺牲停顿换吞吐（批处理）
都不设                    → Ergonomics 按机器规格给默认目标
```

## 4.3 自适应策略与手工参数的冲突

### 4.3.1 自适应策略（GC Ergonomics）调节的维度

- `UseAdaptiveSizePolicy`（默认开）会自动调整 `-Xmn`（新生代大小）、`SurvivorRatio`、`MaxTenuringThreshold` 等维度。

### 4.3.2 "设一让一"原则

- **手工显式设置了其中某维度，自适应就在该维度退让**：例如 `-Xmn2g -XX:SurvivorRatio=8` 后，Policy 不再动新生代与 Eden/S 比例，只剩晋升阈值可调——此时"自适应"名存实亡。
- 结论：要么信 Ergonomics（只给目标，不设尺寸），要么全手工（关 `-XX:-UseAdaptiveSizePolicy` 便于复现），**不要混着设**。

## 4.4 观测

```bash
-XX:+PrintGCDetails -XX:+PrintGCDateStamps        # JDK 8
-Xlog:gc*                                          # JDK 9+ 统一日志
```

- Full GC（Parallel Old 并行整理）在日志中标 `Full GC (Ergonomics)` / `(Metadata GC Threshold)` 等原因，逐条归因方法见 [14_gc_tuning](14_gc_tuning.md)。

# 5. CMS（Concurrent Mark Sweep）收集器

- 特点：**最短回收停顿**（以老年代回收停顿时间为目标的收集器）。
- 目标：GC 时应用线程不全部暂停。
- 算法：老年代使用**标记-清除**（因此会产生碎片，这是它全部缺点的总根源）。
- 启用即隐式绑定 ParNew 新生代 + 老年代 Full GC 退化为 Serial Old。

## 5.1 CMS 回收步骤（六阶段）

```text
初始标记(CM Initial Mark)
  ↓ (STW，极短—只标记 GC Roots 直接关联对象)
并发标记(Concurrent Mark)
  ↓ (与应用线程并发，遍历对象图)
并发预清理(Concurrent Preclean)
  ↓ (处理 JNI 根等"弱根"变化，为 Remark 减负；可多轮，与用户并发)
重新标记(Remark)
  ↓ (STW，修正并发标记阶段变动)
并发清除(Concurrent Sweep)
  ↓ (与应用线程并发，清除垃圾)
并发重置(Concurrent Reset)
  (重置 CMS 数据结构)
```

| 阶段 | 做什么 | STW？ | 时长感受 |
|------|--------|:---:|----------|
| 初始标记 | 只扫 GC Roots 直连对象 | **是** | 极短 |
| 并发标记 | DFS 遍历整个对象图（增量更新屏障工作） | 否 | 长，但用户无感 |
| 并发预清理 | 处理根相关变动 | 否 | 中 |
| 重新标记 | 重扫 dirty 对象，收敛三色（机制见 [11_gc_algorithm](11_gc_algorithm.md) 8.3） | **是** | **较短但随写流量抖动** |
| 并发清除 | 断开链接、修正 Free List | 否 | 长 |
| 并发重置 | 恢复状态位 | 否 | 短 |

```text
时间轴 ─────────────────────────────────▶

应用线程: ─────→ 运行 ──→ 运行 ──→ 运行 ──────→ 运行 ──→
                         │        │        │         │
CMS GC:         [Init]─→[并发标记]→[Remark]→[并发清除]→[Reset]
                 STW              STW
                 极短             较短
```

- 注意：STW 两次（初标 + Remark），**Remark 才是 CMS 的主要停顿来源**——并发标记期间改引用越多，dirty 集越大，Remark 越长。
- `-XX:+CMSScavengeBeforeRemark`：Remark 前先做一次 ParNew 清掉新生代引用，缩短 Remark，常开。

## 5.2 CMS 参数逐条

```bash
-XX:+UseConcMarkSweepGC       # 启用 CMS（自动带 ParNew）
-XX:CMSInitiatingOccupancyFraction=70  # 老年代占用达阈值时启动并发周期
                               # JDK 8 默认 92，且自适应策略会动态调整（历史上还有 68 的默认时代）
                               # 实践：固定 70~80 + UseCMSInitiatingOccupancyOnly，给并发期浮动垃圾留余量
-XX:+UseCMSInitiatingOccupancyOnly     # 只用阈值触发，不靠 JVM 动态判断
-XX:+CMSClassUnloadingEnabled          # 允许 CMS 卸载类（大量动态类加载必开）
-XX:ConcGCThreads=4                    # 并发 GC 线程数（默认 ≈ ParallelGCThreads/4）
-XX:+CMSScavengeBeforeRemark           # Remark 前先做一次 Minor GC
-XX:+ExplicitGCInvokesConcurrent       # System.gc() 走 CMS 并发周期而非 Full GC
```

## 5.3 CMS 四大缺点

### 5.3.1 缺点一：对 CPU 资源敏感

- 并发阶段 GC 线程与应用线程抢核：`ConcGCThreads` 默认仅核数的 1/4——**核越少，1/4 越不够用，或抢占越伤业务**。
- 两三代双核小机器上 CMS 的总停顿经常**输给 Parallel**（并发没优势、还多付屏障与抢占成本）；4~8 核以上、延迟敏感才有收益。
- 调小线程数 → 周期变长 → 更容易 concurrent mode failure；调大 → 业务被饿。

### 5.3.2 缺点二：无法处理浮动垃圾

- 并发标记/清除期间新产生的垃圾本周期收不掉（"快照已开始清除后出生的死亡对象"）→ 必须预留空间给并发期的消耗 → 阈值不能推太高（[11_gc_algorithm](11_gc_algorithm.md) 9.2）。
- 这就是 `CMSInitiatingOccupancyFraction` 默认值偏高（92）与生产建议偏低（70~80）矛盾的根源：默认信任自适应，生产要求确定性。

### 5.3.3 缺点三：内存碎片（标记-清除的固有病）

1. **内存碎片**：标记-清除产生碎片。
   - `-XX:+UseCMSCompactAtFullCollection`（默认开启）：Full GC 时整理。
   - `-XX:CMSFullGCsBeforeCompaction=0`：每次 Full GC 都整理（>0 则攒 N 次 Full 才整理一次）。
- 碎片表现为"老年代总空间还很多，但凑不出连续块" → 提前 Full GC + 整理 → 长停顿。G1 正是冲着这个死穴设计的（第 6 节）。

### 5.3.4 缺点四：Concurrent Mode Failure 退化为 Serial Old

```text
老年代水位涨到 100% 而并发清除还没完
        │  (晋升需求 > 空闲供给)
        ▼
Concurrent Mode Failure
        ▼
暂停并发周期 → 退化为 Serial Old 单线程标记-整理整堆
        ▼
STW = 单线程整堆扫描+搬移：堆越大越灾难（GB 级堆数秒起步）
```

- 另一条退化路径：Young GC **promotion failed**（Survivor+老年代放不下幸存者）→ 同样 Full GC。
- 日志出现 `concurrent mode failure` / `promotion failed` 即 CMS 调优失败的红旗；处理链条见 [11_gc_algorithm](11_gc_algorithm.md) 第 7 节与 [14_gc_tuning](14_gc_tuning.md)。

## 5.4 CMS 已被废弃（时间线）

| 事件 | 版本 |
|------|------|
| 标记废弃（JEP 291） | JDK 9 |
| 正式移除 | JDK 14 |
| 替代者 | **G1**（通用）与 **ZGC**（低延迟） |

- 存量 JDK 8 系统它仍是"比 Parallel 低延迟"的常规选项；新系统 JDK 9+ 直接 G1，不要在移除后版本上寻找 CMS。

# 6. G1（Garbage First）收集器

- **JDK 9 默认收集器**（JEP 248），JDK 7u4 起可用。
- 面向服务端、大内存（>4GB）应用。
- 跨代收集器——不严格区分新生代和老年代的物理空间。

## 6.1 核心思想：Region + Garbage First + 停顿预测

### 6.1.1 Region 化堆与 "Garbage First"

```text
G1 将堆划分成大小相等的 Region（默认约 2048 个）：

┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│E │S │E │E │O │H │E │O │O │S │O │E │  E = Eden
├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤  S = Survivor
│O │E │O │E │S │O │O │E │O │E │O │E │  O = Old
├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤  H = Humongous (大对象)
│  │  │  │  │  │  │  │  │  │  │  │  │
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘

每次回收时，优先选择 回收收益最高(垃圾最多) 的 Region 进行回收，
这就是 "Garbage First" 的由来。
```

### 6.1.2 Pause Prediction Model（停顿预测模型）

```text
G1 对每个 Region 持续统计：回收它需要的"时间成本"（≈ 与存活字节数成正比，
按历史衰减平均）与"收益"（可释放字节数）。

   候选 Region 按 收益/成本 降序排队
   → 在 -XX:MaxGCPauseMillis(默认 200ms) 预算内，贪心取前缀组成 CSet
   → "可预测停顿" = 用碎片化(Region 粒度)回收换取延迟目标
   没有"每次都能收完"的承诺——收不完全靠并发标记与 Mixed GC 追着分配跑。
```

## 6.2 Region 类型

| Region 类型 | 说明 |
|-------------|------|
| Eden | 新生代-Eden，新对象分配 |
| Survivor | 新生代-Survivor，幸存对象 |
| Old | 老年代，长久存活对象 |
| Humongous | 大对象（超过 Region 一半大小），单独存放 |

- Region 大小 = 堆 / 2048，取 2 的幂，范围 1~32MB；E/S/O/H 只是**逻辑标签**，GC 后原地改牌（对比 Serial/Parallel 的物理连续，见 [6_heap](6_heap.md) 9.2）。

## 6.3 Humongous 对象（大对象）

### 6.3.1 判定与分配

- 对象大小 **≥ Region 一半**时认定为 Humongous（`G1HeapRegionSize=4m` → 阈值 2MB）。
- 必须占用**连续**的若干个 Region（首个记 start，其余续接），直接算老年代体系。
- 分配：优先复用恰好合身的旧 Humongous 块，否则从自由 Region 里凑连续段——**凑不出连续空间 → 触发并发标记甚至 Full GC**（日志 `(G1 Humongous Allocation)`），堆碎片化时尤其危险。

```bash
# G1 的 Region 大小
-XX:G1HeapRegionSize=4m     # 4MB（范围 1~32MB，必须是 2 的幂次）
# Humongous 阈值 = Region 大小 / 2
```

### 6.3.2 回收时机与调参

- 回收：Humongous 对象不搬移；并发标记判定整段无引用后，在 Cleanup 阶段释放（较新 JDK 的部分路径可在 Young GC 清理时提前回收），**不必等 Full GC**。
- 长期大量碎小 humongous（如 2MB+ε 的对象配 4MB Region）会造成"回收了但凑不齐"的浪费 → 调大 Region 或业务分块。

## 6.4 G1 重要概念

### 6.4.1 RSet（Remembered Set 记忆集）

- 记录**其他 Region 到本 Region 的引用**（"谁指向我"的入边表）。
- G1 的 RSet 比 Card Table 更精确：底层按 512B 卡片记录，再**按 Region 分组哈希**——扫描某 Region 的根时 O(1) 拿到全部入边，无需全局扫卡表。
- 维护成本高（内存可占堆的百分之几）：引用赋值经**写后屏障**标脏卡 → **Refinement 线程**异步把脏卡并入目标 RSet；因此 G1 同时挂了 SATB 写前屏障——**两道屏障**是 G1 吞吐略输 Parallel 的直接原因。
- 与全局卡表的对比见 [11_gc_algorithm](11_gc_algorithm.md) 5.7。

### 6.4.2 SATB（Snapshot At The Beginning）

- 并发标记阶段开始时对堆做一个快照。
- 标记期间新增/删除的引用由 SATB 队列记录（记的是**删除前的旧值**）。
- 保证不产生漏标；代价是浮动垃圾。机制推导见 [11_gc_algorithm](11_gc_algorithm.md) 8.4。

### 6.4.3 CSet（Collection Set 回收集）

- 本次 GC 要回收的 Region 集合。
- Young GC：所有 Young Region。
- Mixed GC：所有 Young Region + 部分 Old Region。
- CSet 的大小由停顿预测模型按 MaxGCPauseMillis 裁剪（6.1.2）。

## 6.5 G1 回收模式

### 6.5.1 Young GC

- Eden 满了触发。
- STW，多线程将存活对象拷贝到 Survivor/Old Region。
- 回收整个 Eden。

### 6.5.2 Mixed GC 与触发链条

- 回收所有 Young Region + 部分 Old Region（垃圾较多的）。
- 有并发标记过程（Mixed 是并发标记完成后的复制阶段）。
- 分多次回收完所有待回收的 Old Region。
- 触发链条与关键参数：

```text
老年代垃圾占整堆比例 ≥ InitiatingHeapOccupancyPercent(默认 45%，JDK 9 起默认自适应)
   → 并发标记周期 → 标记完成的 Old Region 进入"候选集"
   → 随后的若干次 Young GC 升级为 Mixed GC
   → G1MixedGCCountTarget(默认 8)：候选老年代 Region 分摊到最多 8 次 Mixed 里回收
   → G1HeapWastePercent(默认 5%)：剩余可回收垃圾 < 全堆 5% 就直接放弃后续 Mixed（收益太小）
   → G1MixedGCLiveThresholdPercent(默认 85%)：存活率 >85% 的 Region 不进候选（搬了白搬）
```

### 6.5.3 Full GC（JDK 10 起并行化）

- 作为最后手段：JDK 9 及以前**单线程**标记-整理，STW，慢——曾是 G1 最大痛点。
- **JDK 10 起并行化（JEP 307）**：多线程标记-整理，耗时大幅下降，但仍是"设计失败"的信号。
- 触发场景：Evacuation Failure（to-space exhausted）、Humongous 凑不齐、Metaspace 不足、晋升担保失败。
- 避免 G1 退化为 Full GC 是调优关键（见 6.8 与 [14_gc_tuning](14_gc_tuning.md)）。

## 6.6 G1 回收步骤

```text
1. 初始标记 (Initial Mark)         [STW，附带在 Young GC 中]
2. 根区域扫描 (Root Region Scan)    [与应用并发]
3. 并发标记 (Concurrent Marking)    [与应用并发]
4. 重新标记 (Remark)               [STW]
5. 清除 (Cleanup)                  [STW]
   - 计算各 Region 存活对象并排序
6. 复制 (Copy/Cleanup)             [STW，多轮 Mixed GC 完成]
```

| 阶段 | 说明 |
|------|------|
| 1 初始标记 | 借用一次普通 Young GC 顺带扫老年代根（RSet），不单独加停顿 |
| 2 根区域扫描 | 并发扫 Survivor 区，为并发标记铺"跨代引用"入口 |
| 3 并发标记 | 对象图全遍历，SATB 屏障维持弱三色不变式 |
| 4 重新标记 | 排空 SATB 队列 + 处理遗留引用，Finalizable 对象挂回引用链 |
| 5 清除 | 统计各 Region 存活量、排序候选、重建 Remembered Set 数据（此阶段部分并发） |
| 6 复制 | 真正的 Mixed GC 复制：CSet 内存活对象搬进新 Region → 天然无碎片 |

## 6.7 G1 参数

```bash
-XX:+UseG1GC                    # 启用 G1
-XX:MaxGCPauseMillis=200        # 期望最大停顿（默认 200ms）
-XX:G1HeapRegionSize=4m         # Region 大小
-XX:InitiatingHeapOccupancyPercent=45 # 堆占用45%触发并发标记（默认45%）
-XX:G1ReservePercent=10         # 预留 10% 堆空间防止晋升失败
-XX:ConcGCThreads=4             # 并发 GC 线程数 ≈ 总线程数/4
-XX:ParallelGCThreads=8         # STW 阶段线程数
-XX:G1MixedGCLiveThresholdPercent=85  # Region 中存活对象 > 85% 不回收
-XX:G1MixedGCCountTarget=8      # 候选老年代分几次 Mixed 收完
-XX:G1HeapWastePercent=5        # 剩余垃圾 <5% 放弃 Mixed 周期
-XX:-G1UseAdaptiveIHOP          # 关掉 IHOP 自适应、固定用上面的 45%
```

## 6.8 调优要点

| 要点 | 原因 |
|------|------|
| **不要设 `-Xmn` / NewRatio** | G1 靠"动态调整 Young Region 个数"来满足停顿目标；固定新生代 = 夺走它唯一的调节旋钮（同 4.3 的自适应退让逻辑，G1 更敏感） |
| MaxGCPauseMillis 过小（如 20ms） | 每次 CSet 被裁太小 → 回收赶不上分配 → 并发标记连轴转 / Full GC；G1 是"尽力而为"不是硬承诺，低延迟需求应上 ZGC（[13_low_latency_gc](13_low_latency_gc.md)） |
| MaxGCPauseMillis 过大 | 等于放任单次收很多 Region，延迟目标形同虚设；一般 100~300ms 起步 |
| 频繁 Humongous | 调 Region 大小或改代码；对照 6.3 |
| 出现 Evacuation Failure | 加 `-XX:G1ReservePercent`（10→15/20）或增大堆、降低 IHOP |
| 堆别开太大 | G1 的并发标记要遍历全堆活跃集，超大堆（数十 GB）上标记周期变长 → 该换 ZGC/Shenandoah（[13_low_latency_gc](13_low_latency_gc.md)） |

- 演进方向（一句话）：G1 至今仍是 JDK 21/25 的默认收集器，每个 LTS 都在吞吐、IHOP 自适应、humongous 分配、Full GC 并行度上持续改善（如字符串去重转正、Region Pinning 等），无需"等下一代换收集器"。

# 7. 收集器对比

## 7.1 对比总表

| 收集器 | 目标 | 适用场景 | 区域 | STW | JDK 版本 | 默认/推荐 JDK |
|--------|------|----------|------|-----|----------|----------------|
| Serial | 简单 | 客户端、小堆 | 新生代 | 全部 | 所有版本 | 小内存 Ergonomics 兜底 |
| ParNew | 多线程新生代 | 配合 CMS | 新生代 | 全部 | ~JDK 14 | 8 时代 CMS 搭档 |
| Parallel Scavenge | 吞吐量 | 批处理、后台 | 新生代 | 全部 | 所有版本 | **JDK 8 默认** |
| Serial Old | 简单 | 客户端 | 老年代 | 全部 | 所有版本 | CMS 退化路径 |
| Parallel Old | 吞吐量 | 批处理 | 老年代 | 全部 | JDK 6u23+ | 8 默认配套 |
| CMS | 低延迟 | Web 服务 | 老年代 | 部分 | ~JDK 14 | JDK 8 遗留系统 |
| G1 | 平衡 | 大堆、服务端 | 全部 | 部分 | JDK 7u4+（9u 起生产可用） | **JDK 9+ 全局默认** |
| ZGC | 超低延迟 | 大堆、低延迟 | 全部 | <1ms 级 | JDK 15+（23+ 分代默认） | 大堆/严格 P999 |
| Shenandoah | 超低延迟 | 大堆、低延迟 | 全部 | 极短 | JDK 15+ | 同上（发行版相关） |

## 7.2 读表要点

### 7.2.1 "默认/推荐 JDK"列的来历

- "JDK 8 默认"= Parallel 组合（1.2 节表）；"JDK 9+ 全局默认"= G1（JEP 248）；ZGC 的"23+ 分代默认"只是收集器内部默认（1.3 节澄清）。
- "~JDK 14"的 CMS/ParNew 行：JDK 9 废弃（JEP 291）、JDK 14 移除，新 JDK 上不存在选项。

### 7.2.2 "STW"列与执行策略的对应

- "全部"= 串行/并行 STW 策略；"部分"= 并发收集（阶段拆分见第 4、5、6 章各时间线，策略定义见 [11_gc_algorithm](11_gc_algorithm.md) 4.2）。
- ZGC/Shenandoah 的 STW 描述为"固定小停顿"，细节见 [13_low_latency_gc](13_low_latency_gc.md)。

# 8. 选型决策树

## 8.1 决策树

```text
                         要选 GC 了
                             │
         堆 < ~200MB / 单核 / 客户端 / 容器极小规格？
                             ├─ 是 ──▶ Serial（-XX:+UseSerialGC）
                             │
         离线批处理/编译/跑分：吞吐唯一 KPI，秒级停顿无所谓？
                             ├─ 是 ──▶ Parallel（-XX:+UseParallelGC）
                             │
         在线服务（Web/RPC），JDK 版本？
              ├─ JDK 8：默认 Parallel
              │        ├─ 延迟敏感且 ≥4 核 ──▶ CMS(+ParNew)（8 上可用，勿再新增依赖）
              │        └─ 否则/想省心 ───────▶ G1（8u40+ 即可生产）
              ├─ JDK 11/17：默认 G1，直接调 MaxGCPauseMillis ✔
              └─ JDK 21+：G1 起步
         停顿硬指标（P999 < 10ms）或堆 ≥ 16G 且 CPU 有余量？
                             ├─ 是 ──▶ ZGC（JDK 15+；21 开分代实验 / 23+ 分代默认）
                             │          Shenandoah 为备选
                             └─ 否 ──▶ 留在 G1
 ─────────────────────────────────────────────────────────
 容器场景前置检查：8u191 之前 JVM 看不到 cgroup 内存/CPU 限制，
 默认堆按宿主机算 → OOM-Kill → 见 [23_container_jvm](23_container_jvm.md)
 任何选型之后：用 -Xlog:gc* 实测 P99 停顿与吞吐，再回到 [14_gc_tuning](14_gc_tuning.md)
```

## 8.2 决策解读

### 8.2.1 决策三轴

- **堆大小**（Serial<200M < G1 < 数G < ZGC 数十G+）、**延迟目标**（吞吐优先 Parallel → 停顿目标 G1 → 固定小停顿 ZGC）、**CPU 余量**（并发/屏障都要抢核，核少时低延迟收集器反而更差）。

### 8.2.2 配套验证闭环

- 与 [12_gc_collectors](12_gc_collectors.md) 各章参数表配套使用；JDK 21 基线的组合速查见 [14_gc_tuning](14_gc_tuning.md)。
- 决策树底部的"容器前置检查 + 实测 P99"不是可选项：任何静态选型都要回到日志验证。
