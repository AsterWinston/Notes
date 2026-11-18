# 1. GC 调优的前提

## 1.1 目标三角：延迟、吞吐、内存占用

```text
              延迟（停顿短）
                 ╱╲
                ╱  ╲
               ╱ GC ╲
              ╱______╲
   吞吐（少花时间                内存占用
   在 GC 上）                  （堆小而稳）
   —— 任意两者改善，几乎都在挤压第三者 ——
```

| 目标 | 说明 | 适用场景 | 代表手段（JDK 21 基线） |
|------|------|----------|------------------------|
| 吞吐量优先 | 用户代码运行 > 99%，GC 时间 < 1% | 批处理、后台任务 | Parallel + GCTimeRatio |
| 延迟优先 | 每次 GC 停顿 < N ms | Web 服务、API、游戏 | G1 限 CSet / ZGC 并发转移 |
| 内存占用最少 | 堆尽量小 | 嵌入式、容器限内存场景 | Serial / 缩小 Xmx + 快回收 |

三角互斥的机制根源（推导一遍就记住）：

```text
压低停顿  → 把工作拆小、搬去并发     → 并发线程抢 CPU（吞吐↓）、屏障常驻（吞吐↓）
压低 GC 时间 → 单次多回收点          → 单次停顿↑（延迟↓失败）、要更多暂存空间（内存↑）
压低内存  → 堆小 → 单位时间 GC 更频繁 → 吞吐与延迟双输
```

所以**先写下你的 SLO 是哪一角**，再谈参数。没有"三项全优"的调优，只有明确取舍。

## 1.2 先应用，后 JVM

GC 调优的性价比排序，越靠前收益越大：

| 优先级 | 手段 | 为什么 |
|--------|------|--------|
| ① | 数据结构与序列化：少造 byte[]/String、热点路径避免装箱、换紧凑序列化格式 | 直接砍**分配率**，GC 问题的总源头 |
| ② | 缓存治理：容量上限、过期策略、堆外缓存评估 | 决定老年代"稳态工作集"的大小 |
| ③ | 框架默认值：Netty 池化、连接池大小、日志异步化 | 框架代你分配了大部分对象 |
| ④ | 收集器选型 + 堆大小 | 换赛道，收益大但改动风险大 |
| ⑤ | 精细参数（SurvivorRatio、IHOP、阈值类） | 边际收益最小，最先被误用 |

> 原则：**GC 调优只能"延缓"和"整形"，降分配率才是"根治"。** 泄漏和滥用，任何收集器都救不了（先走 [22_troubleshooting](22_troubleshooting.md)）。

## 1.3 动手之前先建基线

- 不要提前优化，先上线跑起来；但**上线时必带 GC 日志与 JFR**（见 2.1）。
- 压测覆盖一个完整业务周期（含高峰），至少 24h 生产样本。
- 微基准用 JMH（防止 JIT 预热/死代码消除骗你，见 [17_jit_optimization](17_jit_optimization.md) 12.1）。
- 记录调优前的三组数：吞吐（1 - ΣSTW/总时长）、P99/P999 停顿、老年代稳态水位。**没有基线，调优 = 玄学。**

# 2. GC 日志

## 2.1 开启 GC 日志：新旧两代参数

### 2.1.1 JDK 8 及之前的 GC 日志参数串

```bash
-XX:+PrintGCDetails              # 打印详细 GC 日志
-XX:+PrintGCDateStamps           # 打印绝对时间
-XX:+PrintGCApplicationStoppedTime  # 打印应用停顿时间
-Xloggc:/path/to/gc.log           # 日志输出路径
-XX:+UseGCLogFileRotation         # 日志轮转
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=100M
```

> 这组参数在 JDK 9+ **全部失效**（启动仅报 warning）：统一日志系统（JEP 158/271）接管了一切，见 2.1.2。老参数写进新 JDK 脚本是常见事故源（另见第 7 节误区）。

### 2.1.2 JDK 9+ 统一日志：-Xlog 逐段解读

生产标准开局一条：

```bash
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```

```text
-Xlog : gc* : file=gc.log : time,uptime,level,tags : filecount=10,filesize=100M
   │     │        │                 │                        │
 what  选择器   输出目标          装饰器(前缀列)           输出选项(轮转)
```

| 段 | 写法 | 含义 |
|----|------|------|
| what | `gc*` | gc 及其全部子标签：gc、gc+heap、gc+phases、gc+age、gc+tlab… |
| 输出 | `file=gc.log` | 也可 `stdout`（容器场景）、`syslog` |
| 装饰器 | `time,uptime,level,tags` | 绝对时间 / JVM 启动后秒数 / 级别 / 标签，排障四件套 |
| 选项 | `filecount,filesize` | 10 个 × 100MB 环形轮转，防写爆磁盘 |

常用变体：

```bash
-Xlog:gc*=debug                   # 各阶段细分耗时（对应 gc+phases）
-Xlog:gc+age=trace                # 晋升年龄分布（替代老 -XX:+PrintTenuringDistribution）
-Xlog:gc*,safepoint=info          # 附带安全点 → 区分"GC 停顿"与"非 GC 停顿"（见 4.6）
-Xlog:gc:stdout                   # 最小开销，直接进容器日志
```

## 2.2 各收集器样例与逐字段标注

### 2.2.1 Parallel GC 日志（Young GC）

```text
2024-01-15T10:30:01.234+0800: 12.345: [GC (Allocation Failure) 
[PSYoungGen: 65536K->1024K(76288K)] 
65536K->2048K(249088K), 0.0023456 secs] 
[Times: user=0.01 sys=0.00, real=0.02 secs]
```

解读：

- `(Allocation Failure)`：触发原因（GC Cause），全表见 2.3
- `65536K->1024K(76288K)`：新生代，回收前 64M → 回收后 1M（容量 75M）
- `65536K->2048K(249088K)`：整堆，回收前 64M → 回收后 2M（容量 243M）；**两行相减即本次晋升量 ≈ 1M**
- `0.0023456 secs`：本次 STW ≈ 2.3ms
- `Times: user/sys/real`：real 明显大于 user/核数 → 并行度被抢或发生换页

### 2.2.2 G1 GC 日志（JDK 11+ 统一日志形态）

```text
[0.123s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
[0.135s][info][gc] GC(0) 64M->32M(256M) 12.345ms
```

| 字段 | 含义 |
|------|------|
| `GC(0)` | 第 0 个 GC 周期编号，全阶段共享，用于串起日志 |
| `Pause Young (Normal)` | 普通年轻代收集；`(Mixed)` = 附带老年代 Region 的混合回收 |
| `(G1 Evacuation Pause)` | GC Cause；较新 JDK 改名 `G1 Preventive Collection`，含义相同 |
| `64M->32M(256M)` | 整堆回收前/后/容量 |
| `12.345ms` | STW 总时长——**只有这一项算停顿** |

### 2.2.3 CMS GC 日志

```text
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4567K(81920K)] 
6789K(121920K), 0.0003456 secs]

[GC (CMS Final Remark) [YG occupancy: 1234K(40960K)]...
rescan: 0.2345 secs, weak refs: 0.0123 secs, total=0.3123 secs]
```

- 两条都是 STW 阶段（并发阶段标为 `concurrent mode` 不计停顿）；`Final Remark` 的 `rescan + weak refs` 是 CMS 经典停顿大头。
- 关键字警报：`concurrent mode failure`、`promotion failed` → 见 4.3/4.4。

### 2.2.4 ZGC 日志

阶段名即语义（`Pause Mark Start / Concurrent Mark / ...`），逐行解读见 [13_low_latency_gc](13_low_latency_gc.md) 第 6 节；唯一原则：**只看 Pause 行算停顿，Concurrent 行是 CPU 成本**。

## 2.3 GC Cause（触发原因）速查

日志括号里的 cause 决定排查方向：

| Cause 文本 | 含义 | 第一反应 |
|-----------|------|----------|
| `Allocation Failure` | Eden/老年代分配失败，最正常的触发 | 看频率与晋升是否正常 |
| `G1 Evacuation Pause`（旧）/ `G1 Preventive Collection`（新） | G1 按预测主动发起疏散 | 是否 Mixed 跟不上（IHOP 太晚） |
| `G1 Humongous Allocation` | 大对象分配找不到连续 Region | 应用拆大数组 / 调大 Region（4.5） |
| `Metadata GC Threshold` | 元空间达到阈值 | 查类加载泄漏；核对 MaxMetaspaceSize（4.2） |
| `Ergonomics` | Parallel 自适应策略决定 Full GC | 多为堆偏小或晋升过快 |
| `System.gc()` | 显式调用（含 JNI / 某些 NIO 清理路径） | 定位调用方，用 4.2 手段治理 |
| `Heap Dump` | jmap/jcmd dump 附带触发 Full GC | 别在高峰期 dump |
| `G1 Periodic Collection` | G1 周期性维护（超时未回收触发） | 一般无害 |
| `Allocation Rate` / `Warmup` / `Proactive` | ZGC 基于速率/预热/空闲的主动触发 | 正常；配 Stall 才要处理（13 章 2.8.2） |

## 2.4 日志分析工具

**GCViewer：**

```bash
# 使用 GCViewer 分析 gc.log
# 下载：https://github.com/chewiebug/GCViewer
java -jar gcviewer.jar gc.log
```

- 提供图形化展示：GC 类型分布、停顿时间、吞吐量趋势。

**GCEasy：**

- 在线分析：https://gceasy.io/
- 自动生成报告，包含：停顿分析、吞吐量、GC 次数、是否存在内存泄漏。
- 注意：内部日志上传第三方网站有合规风险，敏感系统用 GCViewer 本地分析。

# 3. 核心指标

## 3.1 吞吐量

```text
吞吐量 = 1 − Σ(STW 时长) / 进程总运行时长

例：1 小时日志里所有 Pause 行加起来 36s
    吞吐 = 1 − 36/3600 = 99%
Parallel 的 -XX:GCTimeRatio=n 即该目标的直接表达：吞吐目标 = n/(n+1)，默认 99 → 99%
```

| 服务类型 | 常见吞吐底线 |
|----------|-------------|
| 在线服务 | ≥ 99%（1 小时最多 36s STW） |
| 批处理 | ≥ 99% 起步，专用调优冲 99.9% |

## 3.2 停顿分布：为什么只看平均值会骗人

```text
方案 A: 100 次 × 100ms 停顿   ─ 平均 100ms，最大 100ms
方案 B:   1 次 × 10s  停顿    ─ 次数少得"好看"，但这一次：
         · 上游 1s 超时全部打回 · 集群同刻发布的兄弟实例一起停 → 重试风暴
```

| 指标 | 采集方式 |
|------|----------|
| P50/P99/P999 停顿 | 日志 Pause 行喂 GCViewer/GCEasy 出分位图 |
| 单次最大停顿 | 同上；对齐业务超时线检查 |
| 停顿组成（进安全点耗时 vs GC 本体） | `-Xlog:safepoint`，见 4.6 |

## 3.3 分配率与晋升率：调优的两大输入

### 3.3.1 两大速率公式

```text
Young GC 间隔 ≈ Eden 容量 ÷ 分配率          （Eden 2G、分配率 200MB/s → 约 10s 一次）
晋升速率  ≈ Young GC 频率 × 平均每次晋升量   （或直接从日志老年代增量差分）
```

### 3.3.2 观测手段（按侵入度从低到高）

```bash
# 1) jstat：免费常开，看各区水位与 YGC/FGC 增速
jstat -gcutil <pid> 1000         # 每秒输出一次

# 输出示例：
S0     S1     E      O      M     CCS     YGC   YGCT    FGC  FGCT    GCT
0.00  99.21  65.81  42.67  78.31  83.12   734   12.345   2    0.563   12.908

# 2) async-profiler 分配火焰图：直接定位"哪行代码在造对象"
./asprof -e alloc -d 60 -f alloc.html <pid>

# 3) JFR：生产常开低开销，jdk.ObjectAllocationSample(JDK 11 起采样事件)
#    及 jdk.ObjectAllocationInNewTLAB / OutsideTLAB 全量事件
jcmd <pid> JFR.start name=t settings=profile
jcmd <pid> JFR.dump name=t path=rec.jfr
```

### 3.3.3 jstat 列速查

| 列 | 含义 | 用法 |
|----|------|------|
| S0/S1/E/O | Survivor0/1、Eden、Old 使用率 % | O 列锯齿**不回落** → 泄漏或工作集增长 |
| M / CCS | 元空间 / 压缩类空间使用率 % | 逼近 100 → Metadata GC Threshold 前兆 |
| YGC / YGCT | Young GC 次数 / 累计耗时 | ΔYGCT/Δt 即停顿预算占用 |
| FGC / FGCT | Full GC 次数 / 累计耗时 | 在线服务里应 ≈ 0 |
| GCT | GC 总耗时 | 除运行时长 = 1 − 吞吐 |

### 3.3.4 健康指标与阈值

健康基线三件套（调参前先记录）：分配率、晋升速率、老年代稳态水位。指标含义与阈值：

| 指标 | 含义 | 健康标准 |
|------|------|----------|
| GC 频率 | GC 的次数/时间 | 由 Eden ÷ 分配率决定，稳定即可 |
| 单次 GC 停顿 | 单次 GC 的 STW | Young GC: <50ms, Full GC: <1s（在线服务更严） |
| GC 占比 | GC 时间 / 总运行时间 | <5%（批处理）/ <1%（在线服务） |
| 晋升量 | 每次 Minor GC 进入老年代的大小 | 稳定的低值 |
| Full GC 次数 | Full GC 的绝对数量 | 0 或极罕见（如每天 < 1） |

# 4. 典型问题排查手册

> 统一格式：**现象 → 定位 → 处置**。先看 2.3 的 cause 定性，再进对应小节。

## 4.1 Young GC 过于频繁

### 4.1.1 现象

```text
[GC ... 123ms][GC ... 98ms][GC ... 115ms][GC ... 102ms]...
每几秒一次 Young GC
```

### 4.1.2 定位

```text
Young GC 频率 = 分配率 ÷ Eden 容量        → 分子分母各查一半
  ① 分配率过高？ asprof -e alloc 看火焰图头部（多半是 String/byte[]/包装类）
  ② Eden 偏小？  jstat 看 EC/EU；G1 下看日志 Eden 实际大小（自适应可能压很小）
先算一笔账：晋升速率 = Young 频率 × 平均晋升量，若已接近老年代增速极限，
光调 Eden 没用，必须降分配率。
```

### 4.1.3 处置

```bash
# 方法1：增大新生代（Serial/Parallel/CMS 时代直接有效）
-Xmn2g

# 方法2：调 Eden/Survivor 比例
-XX:SurvivorRatio=12   # Eden:每个S = 12:1 → Eden 占比 80%→85.7%
                       # 注意：调大 SurvivorRatio 才是 Eden 变大、Survivor 变小；
                       # 设成 4（Eden 66.7%）方向相反——Survivor 变大、Eden 反而缩水，
                       # 只有"幸存者多到溢出"时才这么调。
# 方法3（G1）：别手设 Xmn，加大整堆让自适应模型有腾挪空间
-Xmx16g   # G1 自己决定年轻代占比（G1NewSizePercent/G1MaxNewSizePercent 是兜底而非日常旋钮）
```

## 4.2 Full GC 频繁

### 4.2.1 现象

```text
[Full GC ... 5.234s][Full GC ... 4.891s]...
Full GC 变得频繁且停顿久
```

### 4.2.2 排查清单（每条附定位手段）

| # | 嫌疑 | 定位 | 处置 |
|---|------|------|------|
| 1 | `System.gc()` / JNI 显式调用 | 日志 cause = `System.gc()`；常见来源：RMI、DirectByteBuffer 清理、监控 Agent | `-XX:+ExplicitGCInvokesConcurrent`（转并发）优于 `-XX:+DisableExplicitGC`（后者会饿死直接内存清理，见第 7 节） |
| 2 | 元空间不足 | cause = `Metadata GC Threshold`；`jstat -gcmetacapacity <pid>`；`jcmd <pid> GC.class_histogram` 看类量 | 调大 `-XX:MaxMetaspaceSize`；动态类泄漏（Groovy/CGLIB/热部署）单独查 |
| 3 | 直接内存不足 | `Direct buffer memory` OOM 前兆；`-XX:NativeMemoryTracking=summary` + `jcmd <pid> VM.native_memory` | 查 Netty/NIO 泄漏；`-XX:MaxDirectMemorySize` 显式设界 |
| 4 | 晋升过快 / 担保失败 | 日志 `promotion failed`、`(to-space exhausted)` | 见 4.3 |
| 5 | Heap Dump 触发 | cause = `Heap Dump` | dump 挪出高峰；探针别配置成每次抓取 |
| 6 | 大对象直入老年代 | cause = `G1 Humongous Allocation`；histo 看 `[B` 尺寸分布 | 见 4.5 |
| 7 | 并发回收启动太晚 | G1：Mixed 前老年代已逼近上限；CMS：见 4.4 | 降 IHOP / 提高触发灵敏度（默认自适应优先） |
| 8 | 内存泄漏 | Full GC 后老年代水位**不回落**；间隔 10min 两份 dump 对比 | MAT 支配树 → [21_visual_tools](21_visual_tools.md)，流程见 [22_troubleshooting](22_troubleshooting.md) |

## 4.3 晋升失败 / Evacuation Failure

### 4.3.1 原因与日志特征

**原因**：Minor GC 时 Survivor 区放不下，老年代也满了——本质都是**复制算法找不到 to-space**。

```text
ParNew: ... [promotion failed ...]                 ← JDK8 CMS 体系
G1:     ... Pause Young (Normal) (G1 Evacuation Pause) (to-space exhausted)
ZGC:    Allocation Stall                           ← 语义近亲：分配等回收（13 章 2.8.2）
```

### 4.3.2 处置

```bash
# CMS/ParNew 系：给幸存者扩容
-XX:SurvivorRatio=4          # Survivor 各 16.7%（Eden 缩小，Young 变频，组合评估）
-Xmn2g                       # 增大整个新生代
-XX:+PrintPromotionFailure   # 打印详细晋升失败信息

# G1：留足疏散兜底空间
-XX:G1ReservePercent=15      # 默认 10；晋升/复制预留不够时加大
                             # 并把并发标记提前：默认自适应 IHOP，必要时 -XX:InitiatingHeapOccupancyPercent=40
# 通用最后手段：加大堆
-Xmx16g
```

> G1 Full GC 自 JDK 10（JEP 307）起并行化，从"单线程分钟级"改善为"秒级"，但依然是兜底路径，出现即当事故分析。

## 4.4 并发模式失败（Concurrent Mode Failure — CMS）

### 4.4.1 现象与定位

```text
[GC ... Concurrent Mark ... 并发过程中老年代已满]
[Full GC ... 退化为 Serial Old]
```

**定位**：老年代在并发回收完成之前被晋升 + 直入对象填满；或碎片太多放不下晋升。

### 4.4.2 处置

```bash
# 降低触发阈值，留更多浮动垃圾空间
-XX:CMSInitiatingOccupancyFraction=60
-XX:+UseCMSInitiatingOccupancyOnly   # 关掉自适应推断，锁定上面的阈值
-Xmx8g -Xms8g                        # 预留足够的堆空间

-XX:+CMSScavengeBeforeRemark         # Final Remark 前先做一次 Young GC，缩短 remark 停顿
-XX:+UseParNewGC                     # 确认 Young 是并行的（CMS 标配组合）
```

> 碎片问题 CMS 无解（不搬移）——`-XX:+UseCMSCompactAtFullCollection` 只能在 Full 前整理，治标。**还在线上跑的 CMS 系统，中期方案是迁 G1**（JDK 14 起 CMS 已移除，见 [12_gc_collectors](12_gc_collectors.md)）。

## 4.5 G1 Humongous 分配问题

### 4.5.1 机理与危害

- 判定线：对象 ≥ Region 大小的一半；Humongous Region 天然属老年代、且需**连续**分配。
- 危害：大数组频繁进出 → 老年代被"巨物"占坑 + 碎片化 → `G1 Humongous Allocation` 触发 Full GC（旧版本要整堆回收才清理；较新 G1 已能在 Young/Mixed 顺带回收全空 Humongous Region，但别赌）。

### 4.5.2 定位与处置

```bash
# 定位：日志 cause 为 G1 Humongous Allocation；histo 看最大数组来自哪
jmap -histo <pid> | head -30          # 注意加 :live 会触发 STW，生产谨慎

# 处置一：抬高 Region，让"大对象不再大"
-XX:G1HeapRegionSize=16m              # 阈值随之升到 8M（默认 Region≈堆/2048，自动取 1~32M 的 2 幂）

# 处置二（更优）：应用层拆小——流式分段读文件代替 readAllBytes、
# 批量数据分片处理、避免一次性拼巨型 String
```

详见 [12_gc_collectors](12_gc_collectors.md) Humongous 一节与 [6_heap](6_heap.md) 3.4。

## 4.6 长时间 STW ≠ GC 的锅：安全点（Safepoint）

**现象**：监控看到应用停了 2s，GC 日志里却找不到 2s 的 Pause。

### 4.6.1 机理与日志判读

**机理**：`应用停顿 = 进入安全点时间 + GC 本体时间`。GC 只解释后者；线程迟迟到不了 safepoint（长循环无轮询点、native 调用不回返）才是常见元凶。

```bash
-Xlog:safepoint
# 输出示例：
# Total time for which application threads were stopped: 0.512s, Stopping threads took: 0.500s
#                                            ↑总停顿 0.512s              ↑光"集合"就花 0.5s → 不是 GC 慢！
```

### 4.6.2 定位与嫌疑清单

```bash
-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=2000   # 谁迟迟不进安全点，JVM 点名打印
# JFR: jdk.SafepointBegin 事件（settings=profile 已含）
```

**常见嫌疑**：超长无调用循环、线程数爆炸（根扫描本身变慢）、JVMTI/探针 Agent、dump/直方图类命令、老 JDK 的偏向锁撤销（JDK 15 默认关闭偏向锁，JEP 374，见 [19_lock_optimization](19_lock_optimization.md)）。

# 5. GC 调优标准流程（五步闭环）

## 5.1 五步闭环与产出物

```text
① 定目标 ──▶ ② 采数据 ──▶ ③ 分析 ──▶ ④ 调参 ──▶ ⑤ 回归 ──┐
   ▲                                                        │
   └────────────────── 不达标则回到 ③，达标则固化基线 ◀──────┘
```

| 步骤 | 做什么 | 产出物 |
|------|--------|--------|
| ① 定目标 | 写死数字：P99 停顿 ≤ X ms、吞吐 ≥ Y%、堆 ≤ Z G；**一次只追一个主目标** | SLO 卡片 |
| ② 采数据 | 挂 `-Xlog:gc*` + JFR（settings=profile），压测/生产跑满一个业务高峰 | gc.log、.jfr |
| ③ 分析 | 第 3 节指标 + 第 4 节问题归类；先问"分配率/晋升率正常吗"，再问参数 | 问题定性结论 |
| ④ 调参 | 先大后小（堆大小/收集器 → 阈值细节）；**一次只改一组**，可回滚 | 变更单 |
| ⑤ 回归 | 同流量同配置重采，JFR/日志与基线 diff：停顿分布、老年代水位、CPU | 新基线 |

## 5.2 细化决策树（保留自旧版）

```text
1. 明确目标
   ├── 吞吐量优先 → Parallel
   ├── 延迟优先 → G1 / ZGC
   └── 内存优先 → Serial

2. 收集 GC 日志
   └── 至少运行一天，产生高峰日志

3. 分析关键指标
   ├── GC 频率和停顿 → jstat / GCEasy
   ├── 堆使用趋势 → GCViewer
   └── Full GC 分析

4. 调整参数
   ├── 先从大的调整（堆大小、GC 选择器）
   └── 再调细节（SurvivorRatio、TenuringThreshold）

5. 验证效果
   └── A/B 对比，持续监控
```

# 6. 常见参数模板

> 模板是**起点不是终点**；每套都回答"为什么这几个参数"。

## 6.1 低延迟取向模板

### 6.1.1 4GB 传统应用（JDK 8 + ParNew/CMS，遗留系统维护）

```bash
-Xms4g -Xmx4g
-XX:+UseParNewGC -XX:+UseConcMarkSweepGC
-XX:+CMSClassUnloadingEnabled          # CMS 也卸载类，防元空间慢性上涨
-XX:CMSInitiatingOccupancyFraction=70
-XX:+UseCMSInitiatingOccupancyOnly     # 锁定阈值，排除自适应推断抖动
-XX:+ExplicitGCInvokesConcurrent       # System.gc() 转 CMS 并发而非 Serial Full
-XX:+CMSScavengeBeforeRemark           # 缩短 remark 停顿
-XX:ParallelGCThreads=8
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/logs/
-XX:+PrintGCDetails -XX:+PrintGCDateStamps
-Xloggc:/data/logs/gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M
```

为什么：JDK 8 小堆 CMS 是"低延迟默认答案"；固定 IHOP + 类卸载 + remark 前 Young 是 CMS 三件套保命符。**新系统不要选这套**——CMS 在 JDK 14 已删除。

### 6.1.2 4GB 以下 Web 应用（G1，现代默认）

```bash
-Xms2g -Xmx2g
-Xss256k
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/
-Xlog:gc*=info:file=/data/logs/gc.log:time,pid:filecount=10,filesize=100M
```

为什么：默认收集器 + 温和停顿目标；Xms=Xmx 防扩缩容抖动；日志和 dump 兜底常开。注意 `-Xmn` 缺席——G1 下让自适应模型工作（见第 7 节）。

### 6.1.3 4~32GB 服务端（G1 通用，JDK 11+）

```bash
-Xms8g -Xmx8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:G1HeapRegionSize=16m               # 显式 Region：把 Humongous 线抬到 8M，减少大数组直入老年代
-XX:InitiatingHeapOccupancyPercent=45  # 保持默认即可，G1 默认自适应 IHOP，勿轻易关
-XX:MetaspaceSize=512m
-XX:MaxMetaspaceSize=512m
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/
-Xlog:gc*:file=/data/logs/gc.log:time,uptime,tags:filecount=10,filesize=100M
```

为什么：8~32G 是 G1 舒适区；Region 尺寸是全模板里唯一"值得动手"的自适应破坏项——它是为了 Humongous 阈值服务的显式契约。停顿目标 100~200ms 是务实区间，设 10ms 会翻车（第 7 节）。

### 6.1.4 32GB+ 低延迟（分代 ZGC）

```bash
# JDK 21/22：显式开分代；JDK 23+：分代即默认，删掉 ZGenerational 一行
-Xms32g -Xmx32g
-XX:+UseZGC
-XX:+ZGenerational
-XX:SoftMaxHeapSize=28g                # 平时压回 28g，突发可越界——延迟与内存占用的折中
-XX:MetaspaceSize=512m
-XX:MaxMetaspaceSize=512m
-XX:+HeapDumpOnOutOfMemoryError
-Xlog:gc*:file=/data/logs/gc.log:time,uptime,tags:filecount=10,filesize=100M
```

为什么：>8G 且 P99 要求苛刻（<10ms）才是 ZGC 的主场；分代模式吞吐更好（JEP 439 → JEP 490，见 [13_low_latency_gc](13_low_latency_gc.md) 2.6）。前提检查：CPU 有并发 GC 线程的余量、JDK ≥ 21（macOS/Windows 要 ≥ 17）。

## 6.2 吞吐取向模板

### 6.2.1 批处理 / 吞吐优先（Parallel）

```bash
-Xms16g -Xmx16g
-XX:+UseParallelGC
-XX:GCTimeRatio=99                     # 吞吐目标 99%，GC 时间预算 1%
-XX:ParallelGCThreads=16               # ≈ 物理核数（容器里注意 23 章 CPU 配额）
-XX:+UseLargePages                     # 大堆 TLB 收益，需 OS 侧开 hugepage
-XX:+HeapDumpOnOutOfMemoryError
```

为什么：夜里跑批没人关心单次 500ms 停顿，只关心总时长——屏障与并发线程都是纯成本，Parallel 把它们全砍了。**不要用 G1 跑批**：它在为不存在的延迟目标交税。

## 6.3 换收集器之前：默认答案的力量

> **JDK 21 默认 G1，是 Oracle 用海量负载调出来的平衡点。** 堆 ≤ 8G、P99 ≤ 100ms 的绝大多数服务，"G1 + Xms=Xmx + 日志 + dump 兜底"就是最优解；每次"换收集器救火"的成功案例背后是十倍"换完更糟回来"的案例。换之前先确认第 4 节的分配率/晋升率问题已经排除。

# 7. 常见调优误区

## 7.1 常见误区对照表

### 7.1.1 分代与停顿参数类（误区 1~3）

| # | 误区 | 后果 | 正确姿势 |
|---|------|------|----------|
| 1 | 盲目调大 `-Xmn` | Young GC 单次停顿 ∝ Eden 大小变长；老年代被挤压，Full GC 更近 | 按 4.1 公式算需求，G1 下干脆不设 |
| 2 | `MaxGCPauseMillis=10` 设得过小 | G1 每轮只敢收几个 Region → GC 频率爆炸、浮动垃圾堆积、吞吐崩、目标照样 miss | 100~200ms 起步，用数据逐步下压 |
| 3 | 手动固定 Region 数 / 滥用 `-Xmn`、`G1NewSizePercent` | G1 的预测模型失去调节空间，自适应机制被废 | 只显式设 `G1HeapRegionSize`（为 Humongous 阈值），其余交给默认 |

### 7.1.2 日志、显式 GC 与 dump 类（误区 4~6）

| # | 误区 | 后果 | 正确姿势 |
|---|------|------|----------|
| 4 | JDK 9+ 继续配 `-XX:+PrintGCDetails` 全家桶 | 参数失效只剩 warning，**日志根本没开**，出事后干瞪眼 | 统一日志 `-Xlog:gc*`（2.1.2） |
| 5 | `-XX:+DisableExplicitGC` 一把梭 | DirectByteBuffer 等依赖显式 GC 清理的内存**无人回收** → 堆外 OOM | 优先 `-XX:+ExplicitGCInvokesConcurrent`，并管好直接内存上限 |
| 6 | `HeapDumpOnOutOfMemoryError` 不顾磁盘 | dump 文件 ≈ 堆大小，32G 堆写爆共享盘拖死邻居服务 | `HeapDumpPath` 指向独立大容量盘，容量 ≥ Xmx 并预留两份 |

### 7.1.3 容量规划与数据驱动类（误区 7~12）

| # | 误区 | 后果 | 正确姿势 |
|---|------|------|----------|
| 7 | 堆设太大"有备无患" | Full/并发周期变长、元数据开销涨、容器里还与 limit 冲突（另见 [23_container_jvm](23_container_jvm.md)） | 按稳态工作集 × 1.5~2 起步 |
| 8 | `-Xms` ≠ `-Xmx` | 扩缩容伴随额外 STW/页错误抖动 | 生产两者相等（ZGC 的软上限用 SoftMaxHeapSize 表达，见 6.1.4） |
| 9 | 频繁依赖 `System.gc()` | 触发 Full GC 打乱节奏，吞吐骤降 | 代码里删掉；框架自带的用 4.2 表第 1 行治理 |
| 10 | 只看平均停顿，不看最大停顿 | 99.9% 请求快没用，一次超时就是事故（3.2） | P99/P999 + 与超时线对齐 |
| 11 | 忽略元空间 | 堆没满，元空间不足照样 Full GC（Metadata GC Threshold） | `MaxMetaspaceSize` 显式设界 + jstat M 列监控 |
| 12 | 不看 GC 日志盲调参数 | 所有"经验值"都是别人的负载 | 数据驱动：第 5 节五步闭环 |
