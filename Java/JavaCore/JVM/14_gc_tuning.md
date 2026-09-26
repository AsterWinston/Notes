
# 1. GC 调优的前提

- 不要提前优化，先上线跑起来。
- 调优之前先建立性能指标（吞吐量、延迟、内存占用）。
- GC 调优的边界：GC 不是银弹，内存泄漏、代码问题需先解决。

**GC 调优的三个目标：**
| 目标 | 说明 | 适用场景 |
|------|------|----------|
| 吞吐量优先 | 用户代码运行 > 99%，GC 时间 < 1% | 批处理、后台任务 |
| 延迟优先 | 每次 GC 停顿 < N ms | Web 服务、API、游戏 |
| 内存占用最少 | 堆尽量小 | 嵌入式、移动端 |

# 2. GC 日志

## 2.1 Java 8 及之前的 GC 日志

```bash
-XX:+PrintGCDetails              # 打印详细 GC 日志
-XX:+PrintGCDateStamps           # 打印绝对时间
-XX:+PrintGCApplicationStoppedTime  # 打印应用停顿时间
-Xloggc:/path/to/gc.log           # 日志输出路径
-XX:+UseGCLogFileRotation         # 日志轮转
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=100M
```

## 2.2 Java 9+ 统一日志

```bash
# 打印 GC 基本信息
-Xlog:gc*=info:file=/path/to/gc.log:time,pid,tags:filecount=10,filesize=100M

# 打印详细 GC 日志（含堆信息）
-Xlog:gc*=debug:file=gc.log:time

# 打印安全点信息
-Xlog:gc*,safepoint=info:file=gc.log
```

## 2.3 日志解读

**Parallel GC 日志（Young GC）：**
```
2024-01-15T10:30:01.234+0800: 12.345: [GC (Allocation Failure) 
[PSYoungGen: 65536K->1024K(76288K)] 
65536K->2048K(249088K), 0.0023456 secs] 
[Times: user=0.01 sys=0.00, real=0.02 secs]
```

解读：
- `65536K->1024K(76288K)`：新生代，回收前 64M → 回收后 1M（总 75M）
- `65536K->2048K(249088K)`：整堆，回收前 64M → 回收后 2M（总 243M）
- `real=0.02`：实际停顿 20ms

**CMS GC 日志：**
```
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4567K(81920K)] 
6789K(121920K), 0.0003456 secs]

[GC (CMS Final Remark) [YG occupancy: 1234K(40960K)]...
rescan: 0.2345 secs, weak refs: 0.0123 secs, total=0.3123 secs]
```

**G1 GC 日志：**
```
[0.123s] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 
64M->32M(256M) 12.345ms
```

## 2.4 关键指标

| 指标 | 含义 | 健康标准 |
|------|------|----------|
| GC 频率 | GC 的次数/时间 | 根据应用 |
| 每次 GC 停顿 | 单次 GC 的 STW | Young GC: <50ms, Full GC: <1s |
| GC 占比 | GC 时间 / 总运行时间 | <5%（批处理）/ <1%（在线服务） |
| 晋升量 | 每次 Minor GC 进入老年代的大小 | 稳定的低值 |
| Full GC 次数 | Full GC 的绝对数量 | 0 或 极罕见（如每天 < 1） |

# 3. 常见 GC 问题及排查

## 3.1 Young GC 过于频繁

**现象**：
```
[GC ... 123ms][GC ... 98ms][GC ... 115ms][GC ... 102ms]...
每几秒一次 Young GC
```

**原因**：Eden 区太小。

**解决**：
```bash
# 方法1：增大新生代
-Xmn2g

# 方法2：减小 SurvivorRatio（此时 Eden 反而变小）
# 应增大 SurvivorRatio，让 Eden 更大
-XX:SurvivorRatio=4   # Eden:(S0+S1) = 4:1, 即 Eden=66%, Survivor各17%
```

## 3.2 Full GC 频繁

**现象**：
```
[Full GC ... 5.234s][Full GC ... 4.891s]...
Full GC 变得频繁且停顿久
```

**排查方向**：
1. **内存泄漏**—— jmap dump → MAT 分析
2. **老年代不够**—— 增大堆或老年代
3. **元空间不够**—— 增大 Metaspace
4. **System.gc() 调用**—— 排查代码或加 `-XX:+DisableExplicitGC`
5. **大对象频繁进老年代**—— 调整 PretenureSizeThreshold
6. **CMS/G1 并发失败**—— 提高触发阈值

## 3.3 晋升失败 / Promotion Failed

**原因**：Minor GC 时 Survivor 区放不下，老年代也满了。

```
ParNew: ... [promotion failed ...]
```

**解决**：
```bash
-XX:SurvivorRatio=4    # 增大 Survivor 区
-Xmn2g                 # 增大整个新生代
-XX:+PrintPromotionFailure  # 打印详细晋升失败信息
```

## 3.4 并发模式失败（Concurrent Mode Failure — CMS）

**现象**：
```
[GC ... Concurrent Mark ... 并发过程中老年代已满]
[Full GC ... 退化为 Serial Old]
```

**解决**：
```bash
# 降低触发阈值，留更多浮动垃圾空间
-XX:CMSInitiatingOccupancyFraction=60
# 预留足够的堆空间
-Xmx8g -Xms8g
```

# 4. GC 日志分析工具

## 4.1 GCViewer
```bash
# 使用 GCViewer 分析 gc.log
# 下载：https://github.com/chewiebug/GCViewer
java -jar gcviewer.jar gc.log
```
- 提供图形化展示：GC 类型分布、停顿时间、吞吐量趋势。

## 4.2 GCEasy
- 在线分析：https://gceasy.io/
- 自动生成报告，包含：停顿分析、吞吐量、GC 次数、是否存在内存泄漏。

## 4.3 在线 GC 监控
```bash
# jstat 实时监控 GC
jstat -gcutil <pid> 1000         # 每秒输出一次

# 输出示例：
S0     S1     E      O      M     CCS     YGC   YGCT    FGC  FGCT    GCT
0.00  99.21  65.81  42.67  78.31  83.12   734   12.345   2    0.563   12.908
```

各列含义：
- S0/S1: Survivor 使用率 %
- E: Eden 使用率 %
- O: Old 使用率 %
- M: Metaspace 使用率 %
- YGC/YGCT: Young GC 次数及总时间
- FGC/FGCT: Full GC 次数及总时间
- GCT: 总的 GC 时间

# 5. GC 调优的标准步骤

```
1. 明确目标
   ├── 吞吐量优先 → Parallel
   ├── 延迟优先 → G1 / ZGC
   └── 内存优先 → Serial

2. 收集 GC 日志
   └── 至少运行一天，产生高峰日志

3. 分析关键指标
   ├── GC 频率和停顿 → jstat/gceasy
   ├── 堆使用趋势 → GCViewer
   └── Full GC 分析

4. 调整参数
   ├── 先从大的调整 (堆大小, GC 选择器)
   └── 再调细节 (SurvivorRatio, TenuringThreshold)

5. 验证效果
   └── A/B 对比，持续监控
```

# 6. 常见参数模板

## 6.1 4GB 以下 Web 应用（传统）
```bash
-Xms2g -Xmx2g
-Xmn1g
-Xss256k
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/
-Xlog:gc*=info:file=/data/logs/gc.log:time,pid:filecount=10,filesize=100M
```

## 6.2 4~32GB 服务端（通用）
```bash
-Xms8g -Xmx8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:G1HeapRegionSize=16m
-XX:MetaspaceSize=512m
-XX:MaxMetaspaceSize=512m
-XX:+DisableExplicitGC
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/
```

## 6.3 32GB+ 低延迟（ZGC）
```bash
-Xms32g -Xmx32g
-XX:+UseZGC
-XX:+ZGenerational          # JDK 21+
-XX:MetaspaceSize=512m
-XX:MaxMetaspaceSize=512m
-XX:+HeapDumpOnOutOfMemoryError
```

# 7. 调优误区

1. **频繁调 `System.gc()`**—— 反而触发 Full GC，破坏 GC 节奏。
2. **堆设太大**—— Full GC 时停顿极长（几十秒），且可能与容器限制冲突。
3. **`-Xms` 和 `-Xmx` 不等**—— 堆动态伸缩也消耗性能。
4. **只看平均停顿，不看最大停顿**—— 99.9% 的请求快没用，0.1% 断了就是事故。
5. **忽略元空间**—— 即使堆没满，元空间不足也一样 Full GC。
6. **不看 GC 日志就盲调参数**—— 数据驱动，而非猜测。
