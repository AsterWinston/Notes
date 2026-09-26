
# 1. 收集器概览

```
新生代收集器                             老年代收集器
Serial ────────────────────────────▶ Serial Old
ParNew ────────────────────────────▶ CMS
Parallel Scavenge ─────────────────▶ Parallel Old
                                       │
G1 ──▶ (混合回收，横跨新生代和老年代) ──┘
```

- 连线表示可以组合使用。
- CMS（Concurrent Mark Sweep）在 JDK 9 中已标记为废弃，JDK 14 已移除。
- 新生代都使用标记-复制算法，老年代使用标记-清除或标记-整理。

# 2. Serial 收集器

- **单线程**收集器，GC 时必须暂停所有用户线程（STW）。
- 简单高效，无线程切换开销。
- 适用于**桌面应用、嵌入式、小堆**场景。

```
线程0: ──应用────STW────────应用─────▶
线程1: ──应用────STW────────应用─────▶
GC线程:           [GC]
时间轴 ──────────────────────────────▶
```

```bash
# 启用 Serial (新生代) + Serial Old (老年代)
-XX:+UseSerialGC
```

# 3. ParNew 收集器

- Serial 的**多线程版本**。
- 除了 Serial，只有它能和 CMS 配合使用。
- 单核性能不如 Serial（线程切换开销），多核比 Serial 好。

```
线程0: ──应用──────STW──────应用─────▶
线程1: ──应用──────STW──────应用─────▶
GC线程1:         [  新生代GC  ]
GC线程2:         [  并行回收  ]
```

```bash
# ParNew + CMS (JDK 8 常用)
-XX:+UseParNewGC
-XX:ParallelGCThreads=4       # GC 线程数
```

# 4. Parallel Scavenge + Parallel Old

- **吞吐量优先**的组合。
- Parallel Scavenge 新生代 + Parallel Old 老年代。
- 关注**可控的吞吐量**：用户代码运行时间 / (用户代码运行时间 + GC 时间)。
- JDK 8 的默认收集器组合。

**核心参数：**
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

**自适应策略（GC Ergonomics）：**
- `-XX:+UseAdaptiveSizePolicy`（默认开启）会自动调整：
  - `-Xmn`（新生代大小）
  - `-XX:SurvivorRatio`（Eden:Survivor 比例）
  - `-XX:MaxTenuringThreshold`（晋升阈值）
- 如果只追求最大停顿时间，不需要手动调整这些参数。

# 5. CMS（Concurrent Mark Sweep）

- 特点：**最短回收停顿**。
- 目标：GC 时应用线程不全部暂停。
- 算法：老年代使用标记-清除（因此会产生碎片）。

## 5.1 CMS 回收步骤

```
初始标记(CM Initial Mark)
  ↓ (STW，极短—只标记 GC Roots 直接关联对象)
并发标记(Concurrent Mark)
  ↓ (与应用线程并发，遍历对象图)
重新标记(Remark)
  ↓ (STW，修正并发标记阶段变动)
并发清除(Concurrent Sweep)
  ↓ (与应用线程并发，清除垃圾)
并发重置(Concurrent Reset)
  (重置 CMS 数据结构)
```

```
时间轴 ─────────────────────────────────▶

应用线程: ─────→ 运行 ──→ 运行 ──→ 运行 ──────→ 运行 ──→
                        │        │        │         │
CMS GC:         [Init]─→[并发标记]→[Remark]→[并发清除]→[Reset]
                 STW              STW
                 极短             较短
```

## 5.2 CMS 参数

```bash
-XX:+UseConcMarkSweepGC       # 启用 CMS
-XX:CMSInitiatingOccupancyFraction=70  # 老年代占用 70% 时触发（默认 92% JDK 8）
-XX:+UseCMSInitiatingOccupancyOnly     # 只用阈值触发，不靠 JVM 动态判断
-XX:+CMSClassUnloadingEnabled          # 允许 CMS 卸载类
-XX:ConcGCThreads=4                    # 并发 GC 线程数
-XX:+CMSScavengeBeforeRemark           # Remark 前先做一次 Minor GC
```

## 5.3 CMS 的缺点

1. **对 CPU 资源敏感**：并发阶段占用应用线程。
2. **无法处理浮动垃圾**：并发标记阶段新产生的垃圾只能下次 GC 回收。
3. **内存碎片**：标记-清除产生碎片。
   - `-XX:+UseCMSCompactAtFullCollection`（默认开启）：Full GC 时整理。
   - `-XX:CMSFullGCsBeforeCompaction=0`：每次 Full GC 都整理。
4. **Concurrent Mode Failure**：并发清理过程中老年代写满 → 退化为 Serial Old。
   - 极端情况下停顿时间长。

## 5.4 CMS 已被废弃

- JDK 9：标记为废弃。
- JDK 14：正式移除。
- 替代者：**G1** 和 **ZGC**。

# 6. G1（Garbage First）

- **JDK 9 默认收集器**。
- 面向服务端、大内存（>4GB）应用。
- 跨代收集器——不严格区分新生代和老年代的物理空间。

## 6.1 G1 核心思想

```
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

## 6.2 G1 Region 类型

| Region 类型 | 说明 |
|-------------|------|
| Eden | 新生代-Eden，新对象分配 |
| Survivor | 新生代-Survivor，幸存对象 |
| Old | 老年代，长久存活对象 |
| Humongous | 大对象（超过 Region 一半大小），单独存放 |

## 6.3 Humongous 对象（大对象）

- 对象大小 ≥ Region 一半时，认定为 Humongous。
- 直接分配在老年代（Humongous Region）。
- 回收只在 GC 暂停的清理阶段或 Full GC 时。

```bash
# G1 的 Region 大小
-XX:G1HeapRegionSize=4m     # 4MB（范围 1~32MB，必须是 2 的幂次）
# Humongous 阈值 = Region 大小 / 2
```

## 6.4 G1 重要概念

**RSet（Remembered Set 记忆集）：**
- 记录其他 Region 到本 Region 的引用。
- G1 的 RSet 比 Card Table 更精确，粒度是 Card（512B）。
- 维护成本高，需要通过**写后屏障**更新。

**SATB（Snapshot At The Beginning）：**
- 并发标记阶段开始时对堆做一个快照。
- 标记期间新增/删除的引用由 SATB 队列记录。
- 保证不产生漏标。

**CSet（Collection Set 回收集）：**
- 本次 GC 要回收的 Region 集合。
- Young GC：所有 Young Region。
- Mixed GC：所有 Young Region + 部分 Old Region。

## 6.5 G1 回收模式

**Young GC**：
- Eden 满了触发。
- STW，多线程将存活对象拷贝到 Survivor/Old Region。
- 回收整个 Eden。

**Mixed GC**：
- 回收所有 Young Region + 部分 Old Region（垃圾较多的）。
- 有并发标记过程。
- 分多次回收完所有待回收的 Old Region。

**Full GC**：
- 作为最后手段，单线程，STW，慢。
- 避免 G1 退化为 Full GC 是调优关键。

## 6.6 G1 回收步骤

```
1. 初始标记 (Initial Mark)         [STW，附带在 Young GC 中]
2. 根区域扫描 (Root Region Scan)    [与应用并发]
3. 并发标记 (Concurrent Marking)    [与应用并发]
4. 重新标记 (Remark)               [STW]
5. 清除 (Cleanup)                  [STW]
   - 计算各 Region 存活对象并排序
6. 复制 (Copy/Cleanup)             [STW，多轮 Mixed GC 完成]
```

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
```

# 7. 收集器对比

| 收集器 | 目标 | 适用场景 | 区域 | STW | JDK 版本 |
|--------|------|----------|------|-----|----------|
| Serial | 简单 | 客户端、小堆 | 新生代 | 全部 | 所有版本 |
| ParNew | 多线程新生代 | 配合 CMS | 新生代 | 全部 | ~JDK 14 |
| Parallel Scavenge | 吞吐量 | 批处理、后台 | 新生代 | 全部 | 所有版本 |
| Serial Old | 简单 | 客户端 | 老年代 | 全部 | 所有版本 |
| Parallel Old | 吞吐量 | 批处理 | 老年代 | 全部 | JDK 6+ |
| CMS | 低延迟 | Web 服务 | 老年代 | 部分 | ~JDK 14 |
| G1 | 平衡 | 大堆、服务端 | 全部 | 部分 | JDK 7+ |
| ZGC | 超低延迟 | 大堆、低延迟 | 全部 | <10ms | JDK 15+ |
| Shenandoah | 超低延迟 | 大堆、低延迟 | 全部 | 极短 | JDK 15+ |