
# 1. 为什么需要低延迟收集器

- CMS 停顿 ~100ms，已达瓶颈。
- G1 在堆 10~100GB 时停顿 ~100ms，也难以满足严格低延迟需求。
- ZGC 目标：目标停顿 < 1ms（JDK 11 实验性，JDK 15 正式版）。
- Shenandoah 目标：停顿与堆大小无关。

**低延迟的意义：**
- 金融交易系统（毫秒级别）
- 在线实时推荐
- 游戏服务器
- 高并发接口

# 2. ZGC（Z Garbage Collector）

## 2.1 ZGC 概述

| 特性 | 说明 |
|------|------|
| 核心目标 | 亚毫秒级停顿（<1ms） |
| 支持堆大小 | 8MB ~ 16TB |
| GC 线程 | 与用户线程几乎全程并发 |
| 停顿时间 | 不随堆大小增长 |
| 最大停顿 | < 1ms (通常 0.05~0.5ms) |
| JDK 版本 | JDK 11 实验性，JDK 15 正式版 |
| 分代 ZGC | JDK 21 实验性 |

## 2.2 核心技术：染色指针（Colored Pointers）

ZGC 的 64 位指针利用高位 bit 存储元数据：

```
64位指针布局：
┌────────────┬──────────┬───────────────────┐
│ 42位地址   │ 4位状态  │ 18位保留          │
│ （支持16TB）│          │ （未来扩展）       │
└────────────┴──────────┴───────────────────┘

4 位状态（染色位）：
   - Marked0/Marked1：标记位（交替使用）
   - Remapped：对象是否已被重新映射
   - Finalizable：对象是否仅被 finalizer 引用
```

- 通过指针染色，ZGC 不依赖对象头存储 GC 信息。
- 这使得 ZGC 可以在极小停顿内完成对象的并发重定位。

## 2.3 ZGC 回收阶段

```
┌─────────────┐   ┌────────────┐   ┌─────────────┐
│   标记       │   │   转移      │   │   重定位     │
│  (Mark)     │ → │ (Relocate) │ → │  (Remap)    │
│  并发       │   │  并发       │   │  并发        │
│  STW:初始   │   │  STW:选择  │   │  STW:无      │
│  /结束标记  │   │  CSet      │   │              │
└─────────────┘   └────────────┘   └─────────────┘
```

**详细步骤：**
1. **初始标记（Initial Mark）** [STW, <1ms]：标 GC Roots 直接引用。
2. **并发标记/重映射（Concurrent Mark/Remap）** [并发]：遍历对象图，标记存活+重映射。
3. **并发预备重分配（Concurrent Prepare for Relocate）** [并发]：找要整理的 Region。
4. **初始转移（Initial Transfer）** [STW, <1ms]：迁移 GC Roots 直接引用的对象。
5. **并发转移（Concurrent Transfer）** [并发]：将 CSet 中存活对象复制到新 Region。
6. **并发重映射（Concurrent Remap）** [并入下一次标记 Phase]。

## 2.4 读屏障（Load Barrier）

- ZGC 使用读屏障（而非写屏障），在读取对象引用时检查指针颜色。
- 如果读到"已转移"的指针，自修复——更新引用指向新位置。

```
读屏障伪代码：
oop load_barrier(oop* field, oop value) {
    if (is_bad_color(value)) {
        // 自修复：更新引用到新位置
        oop good = remap(value);
        *field = good;
        return good;
    }
    return value;
}
```

## 2.5 ZGC NUMA 感知

- NUMA（Non-Uniform Memory Access）：多 CPU 架构下访问不同内存区域速度不同。
- ZGC 支持 NUMA-aware，尽量将对象分配在访问最快的本地内存上。

```bash
-XX:+UseNUMA               # 开启 NUMA 支持
```

## 2.6 ZGC 参数

```bash
-XX:+UseZGC                          # 启用 ZGC
-Xmx16g                              # 堆最大 16GB（ZGC 支持到 16TB）
-XX:ZCollectionInterval=0            # GC 间隔（秒），0 = 不限制
-XX:ZAllocationSpikeTolerance=5      # 分配尖峰容忍度
-XX:+ZGenerational                  # JDK 21+ 分代 ZGC（实验性）
-XX:ConcGCThreads=4                  # 并发线程数
```

## 2.7 ZGC 适用场景

- 大堆（> 4GB）
- 极低延迟要求（<1ms）
- 堆动态变化剧烈（频繁分配和回收）

**局限：**
- CPU 密集型：需要额外的并发线程，CPU 占用较高。
- 吞吐量略低于 G1（约 5%~15%）。

# 3. Shenandoah

## 3.1 概述

- Red Hat 开发，贡献给 OpenJDK。
- JDK 12 实验性，JDK 15 正式版。
- 与 ZGC 目标相同：低停顿，停顿时间与堆大小无关。

## 3.2 与 ZGC 的核心差异

| 特性 | ZGC | Shenandoah |
|------|-----|------------|
| 染色指针 | 是（64位指针存元数据） | 否（用对象头 + 前向指针） |
| 并发转移 | 读屏障 | Brooks 前向指针 + 比较交换 |
| 编译屏障 | 需要读屏障 JIT 支持 | 需要不同屏障 JIT 支持 |
| 内存开销 | 约 0%（用指针位） | 多一个前向指针（每对象 8B） |
| 平台支持 | x86_64 / AArch64 | x86_64 / AArch64 / PPC |

## 3.3 关键技术：Brooks Pointer

- 每个对象头有一个"前向指针"（Forwarding Pointer）。
- 并发转移时，先在旧对象的前向指针处写入新地址。
- 其他线程通过这个指针自动找到新位置。

```
并发转移过程：
1. 拷贝对象到新位置
2. CAS 设置旧对象前向指针指向新位置
3. 如果 CAS 失败（其他线程已设置），放弃拷贝
4. 读取时检查前向指针，自动转发
```

## 3.4 Shenandoah 回收阶段

```
初始标记 [STW, 极短] → 扫描 GC Roots 直接引用
并发标记 [并发]       → 遍历整个堆
最终标记 [STW, 短]    → 处理标记队列（SATB）+ 计算回收区域
并发清理 [并发]       → 清理无存活对象的 Region
并发转移 [并发]       → 将存活对象从 CSet 复制到新 Region
初始更新引用 [STW]    → 准备更新引用
并发更新引用 [并发]   → 更新所有指向旧对象的引用
最终更新引用 [STW]    → 更新 GC Roots 中的引用
并发清理 [并发]       → 回收旧 Region
```

## 3.5 Shenandoah 参数

```bash
-XX:+UseShenandoahGC              # 启用 Shenandoah
-XX:ShenandoahGCHeuristics=adaptive  # GC 触发策略
    # adaptive, static (固定阈值), compact (低内存), aggressive (激进)
-XX:ShenandoahMinFreeThreshold=10  # 最小空闲空间阈值
-XX:ShenandoahPacingMaxDelay=5    # 控制分配速度的最大延迟
```

# 4. 三款低延迟收集器对比

| 维度 | G1 | ZGC | Shenandoah |
|------|-----|-----|------------|
| 目标停顿 | ~100ms | <1ms | <10ms |
| 最大堆 | > 几十 GB | 16TB | > 几十 GB |
| 核心算法 | Region + RSet | 染色指针 + 读屏障 | Brooks 指针 |
| 内存开销 | RSet(约 5%) | 极少 | 前向指针(约 8B/对象) |
| CPU 开销 | 中 | 较高 | 较高 |
| 吞吐量 | 较高 | 略低于 G1 | 略低于 G1 |
| 成熟度 | 非常成熟 | 较成熟 | 较成熟 |
| 推荐场景 | 通用大堆应用 | 极低延迟 | 需要跨平台的低延迟 |
| JDK 默认 | JDK 9+ | - | - |

# 5. GC 选择建议

```
你是客户端/小堆（<100MB）？
  → -XX:+UseSerialGC

你是批处理/后台任务，追求吞吐量？
  → -XX:+UseParallelGC  （JDK 8 默认）

你是 Web 服务/API 网关，堆 4~32GB？
  → -XX:+UseG1GC          （JDK 9+ 默认）

你是堆 > 32GB？
  → -XX:+UseG1GC 或 -XX:+UseZGC

你需要亚毫秒级停顿？
  → -XX:+UseZGC 或 -XX:+UseShenandoahGC
```

# 6. 实战：ZGC 日志示例

```bash
java -XX:+UseZGC -Xmx16g -Xlog:gc* -jar app.jar
```

日志示例：
```
[0.123s] GC(0) G1 Pause Young (Normal) (G1 Evacuation Pause) 
[0.123s] GC(1) ZGC Pause Mark Start 0.143ms
[15.234s] GC(1) ZGC Concurrent Mark 1456.789ms
[15.236s] GC(1) ZGC Pause Mark End 1.234ms
[17.891s] GC(1) ZGC Concurrent Process Non-Strong References ...
[17.893s] GC(1) ZGC Pause Relocate Start 0.089ms
[20.456s] GC(1) ZGC Concurrent Relocate 2567.123ms
```