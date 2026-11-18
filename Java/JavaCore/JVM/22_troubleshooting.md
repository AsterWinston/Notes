# 1. 故障排查方法论

> 版本基线：JDK 21 LTS。工具命令详见 [20_jvm_tools](20_jvm_tools.md)，可视化/Arthas 详见 [21_visual_tools](21_visual_tools.md)。

## 1.1 故障五类

| 类别 | 用户侧体感 | 核心指标 | 章节 |
|------|-----------|----------|------|
| CPU 飙高 | 机器满载、所有请求变慢 | load、us%、GC 线程占比 | 第 2 节 |
| 内存泄漏 / OOM | 间歇性 5xx、进程重启 | OU 趋势、RSS vs -Xmx | 第 3 节 |
| GC 频繁 / 停顿长 | P99 周期性尖刺 | YGC/FGC 频率、pause 时长 | 第 4 节 |
| 线程问题（死锁/阻塞） | 部分功能"卡死" | BLOCKED 聚集、等待图成环 | 第 5 节 |
| 线程/句柄资源耗尽 | 拒绝建线程、too many open files | 线程数、fd 数 | 第 6 节 |
| （交叉）延迟毛刺 | 偶发慢 | 上面全都要排除 | 第 7 节 |

## 1.2 排查五步

```text
发现问题（监控告警 / 用户反馈）
    │
    ▼
① 保全现场（重启/扩容之前！jstack + jstat + GC 日志，dump 视情况）
    ▼
② 快速判断故障类型
    │
    ├── CPU 飙高 ──────▶ 见第 2 节
    ├── 内存泄漏/OOM ──▶ 见第 3 节
    ├── GC 频繁/停顿 ──▶ 见第 4 节
    ├── 线程死锁 ──────▶ 见第 5 节
    ├── 线程过多 ──────▶ 见第 6 节
    ├── 响应变慢 ──────▶ 见第 7 节
    └── 直接内存溢出 ──▶ 见 3.6 / 第 8 节
    ▼
③ 工具链取证（jstat→jstack→histo→dump，成本递增）
    ▼
④ 假设-验证（改一个变量：参数/代码/流量，观察指标回摆）
    ▼
⑤ 根治 + 复盘（补监控项、把兜底参数配全）
```

## 1.3 现场保全优先级（按操作成本排序）

```text
便宜且安全 ──────────────────────────────▶ 昂贵/有停顿
jstat -gcutil   jstack(×3)   jinfo   histo   JFR.dump   heap_dump
（秒级）        （秒级）      （秒级） （可能GC）（需常开）  （STW+IO，分钟级，最后做）
```

- **重启/摘流量之前先抓现场**，否则证据随进程一起消失。
- heap dump 是"最后手段"：8G 堆可致分钟级停顿 + 大文件传输，先靠轻量命令收敛方向。
- 有 JFR 持续录制（见 21）则可直接 `JFR.dump` 取回故障窗口，优先级前移。

## 1.4 提前兜底参数组（上线前配好）

### 1.4.1 兜底参数组

```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dumps     # OOM 自动留现场
-Xlog:gc*:file=/data/logs/gc.log:time,pid,uptime:filecount=10,filesize=100M   # GC 日志常开
-XX:ErrorFile=/data/logs/hs_err_pid%p.log                        # JVM 崩溃现场
-XX:NativeMemoryTracking=summary                                 # 堆外可追溯（可选）
-XX:StartFlightRecording=maxsize=512m,maxage=6h,dumponexit=true  # 行车记录仪（JDK11+）
```

### 1.4.2 `-XX:OnOutOfMemoryError` 的两难

| 方案 | 命令 | 优点 | 代价 |
|------|------|------|------|
| 保留现场，慢失败 | `-XX:OnOutOfMemoryError="sleep 3600"` 之类挂住进程 | 可上机取证 | 半死服务继续接流量 |
| 快速失败 | `-XX:OnOutOfMemoryError="kill -9 %p"`（或 `-XX:+ExitOnOutOfMemoryError`） | 立刻摘除，K8s 拉起新实例 | 现场丢失（靠 HeapDumpOnOOM + JFR 兜底） |

现代结论：**云原生选快速失败**（`ExitOnOutOfMemoryError` + 自动 dump + JFR 常开），取证靠"事前埋的参数"而不是"事后手动抢救"。

# 2. CPU 飙高排查

## 2.1 完整排查链路（保留）

```bash
# 第一步：找到 CPU 最高的进程
top -c
# 假设找到 pid = 12345

# 第二步：找到进程中最耗 CPU 的线程
top -Hp 12345
# 假设找到线程 tid = 12350 (十进制)

# 第三步：线程 ID 转十六进制
printf "%x\n" 12350
# 输出：303e

# 第四步：导出线程堆栈，定位代码
jstack 12345 | grep -A 30 "nid=0x303e"
# 或者
jcmd 12345 Thread.print > thread.txt

# 第五步：查看线程状态和调用栈
# RUNNABLE → 真正在跑；死循环或大量计算
# BLOCKED  → 等待锁（结合 "waiting to lock" 查看）
```

## 2.2 先分流：应用线程烧 CPU 还是 GC/JIT 烧 CPU

### 2.2.1 线程名即分流器

```text
top -Hp 的线程名就是分流器：
  http-nio-exec-* / 业务线程名   → 应用代码问题（死循环/热点计算）
  GC Thread#0 / G1 Main Marker / G1 Conc#0 → GC 风暴 → 转第 4 节
  C1/C2 CompilerThread*          → JIT 编译风暴（启动期/流量爬升期正常，稳态则异常）
  VM Thread                      → safepoint 同步在等某个线程到达（见 4.3）
```

### 2.2.2 GC 风暴佐证

```bash
# 佐证：CPU 高 + jstat 里 YGC/FGC 增速快 + GCT 占比大 → GC 风暴无疑
jstat -gcutil <pid> 1000 10
```

## 2.3 常见原因清单（扩充版）

| 原因 | 特征 | 定位 |
|------|------|------|
| 死循环 | 线程一直 RUNNABLE，多次快照同一方法 | jstack 采 3 次对比栈顶 |
| 大循环/复杂计算 | 线程 RUNNABLE，栈稳定 | arthas trace 看耗时分布 |
| 频繁 GC | GC 线程占比高 | jstat -gc 查看 GC 频率 → 第 4 节 |
| 正则回溯（灾难性回溯） | RUNNABLE 卡 `java.util.regex`；嵌套量词 `(a+)+` 遇畸形长串指数爆炸 | jstack 定位 + 样本字符串单测复现 |
| JSON/序列化热点 | 大对象序列化、fastjson/jackson 方法宽热点 | arthas trace / 火焰图 |
| 随机数 | `SecureRandom` 高并发下争用 native 熵池 | 火焰图见 Native / SecureRandom 帧 |
| JIT 编译风暴 | 启动/刚扩容期 C2 线程满载 | 正常现象，观察是否衰减；`-XX:TieredStopAtLevel` 可牺牲峰值换启动 |
| HashMap 并发修改 | 见 2.6 案例 | jstack 见 `HashMap.getNode/resize` |
| 上下文切换多 | vmstat cs 值高 | 线程数过多/锁乒乓 → 第 6 节 |
| GC 后的"误伤"：safepoint 等待 | CPU 高但栈都正常 | -Xlog:safepoint（见 4.3） |

## 2.4 Arthas 快速定位（保留）

```bash
# 查看线程 CPU 排行
thread -n 5

# 使用火焰图
profiler start
# 等待 30 秒
profiler stop --format html

# 监控方法耗时
trace com.example.MyClass suspiciousMethod
```

## 2.5 火焰图判读三句话

```text
            ┌──────────────────────────┐
宽度 ──────▶│ run()      │ loop()      │   横向宽 = 时间占比（总宽=100%）
            │ ┌────────┐ │ ┌─────────┐ │   "平顶" = 热点叶子方法（时间真正花在这）
            └─┴ hash() ┴─┴─┴ regex() ┴─┘
高度 ─────── 纵向 = 调用深度，顶上是叶子
颜色 ─────── 默认仅区分相邻帧/包名（部分工具按配色区分内核/Java，不代表冷热）
```

- 找"最宽的平顶"；栈深但都窄 → 没问题；`Unsafe_park`/`futex` 宽 → 其实是**等锁/等队列**，转线程篇。
- 分配火焰图（alloc）宽 → 造垃圾源头，联动 GC 问题；lock 火焰图宽 → 争用点。

## 2.6 案例：HashMap 并发成环（标注版本）

```java
// 多线程同时对 HashMap put 触发扩容
// JDK 7：resize 头插法反转链表 → 并发下可成环 → get() 无限循环 → CPU 100%（经典死循环案例）
// JDK 8：尾插法 + 高低位拆分，环基本消除；但并发写仍会 数据覆盖丢失/size 错乱/读到中间态
```

- 结论：**JDK 7 表现是"CPU 烧穿的死循环"，JDK 8+ 表现是"数据错乱"**，两者都不是正确用法。
- 定位：jstack 里业务线程 RUNNABLE 停在 `java.util.HashMap.getNode`/`hash`。
- 解决：`ConcurrentHashMap` 或外部加锁；`Collections.synchronizedMap` 只是兜底（遍历仍需手动同步）。

# 3. 内存泄漏 / OOM 排查

## 3.1 第一步：堆内还是堆外

```text
RSS(容器/OS 看到的内存) vs -Xmx：
  RSS ≈ Xmx + 非堆正常项          → 堆内问题，dump 路线
  RSS ≫ Xmx（堆没满进程先没了）    → 堆外问题：
       ① jcmd <pid> VM.native_memory summary.diff → 找 NMT 增长区域（见 20 第 10 节）
       ② NMT 平稳但 RSS 仍涨 → pmap -x / smaps（NMT 覆盖不到的 native 分配：
          Netty/压缩库/JNI）→ 库自身统计 + heap dump 查 DirectByteBuffer 实例总量
被 cgroup 杀（OOMKilled）但日志无 OutOfMemoryError → 也是堆外路线 → [23_container_jvm](23_container_jvm.md)
```

## 3.2 堆内存溢出（Java heap space）（保留）

### 3.2.1 报错形态

```java
// OOM 错误信息
// java.lang.OutOfMemoryError: Java heap space
```

### 3.2.2 五步排查

```bash
# 1. 确认是堆溢出还是非堆
# 堆溢出: Java heap space
# 元空间: Metaspace
# 直接内存: Direct buffer memory

# 2. 生成 heap dump（自动）—— 启动参数提前配置：
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/heapdump.hprof

# 3. 手动导出 heap dump
jcmd <pid> GC.heap_dump /data/logs/heapdump.hprof
# 注意：生产环境谨慎使用 live 选项（会 Full GC）

# 4. 使用 MAT 分析（Leak Suspects → Dominator Tree → GC Roots 路径）

# 5. 使用 jmap 快速查看 Top 类（两次采样看增量）
jmap -histo <pid> | head -20
```

## 3.3 内存泄漏的典型特征（保留）

```text
jstat -gc <pid> 查看：
- FGC 次数持续增长
- OU (Old Used) 只增不减
- 每次 GC 后 OU 不下降（或下降很少）
- 极端：FGCT 增速 ≈ 墙钟时间 → "GC overhead limit exceeded" 前夜
```

**MAT 分析内存泄漏关键技巧（保留）：**

```text
1. Histogram → 按 Retained Heap 排序
2. 找到可疑对象的 Dominator Tree
3. Path to GC Roots → exclude soft/weak/phantom references
4. 追踪到最终持有的代码
```

## 3.4 报错 → 定位总表（扩展）

| 报错 | 本质 | 高频根因 | 第一动作 |
|------|------|----------|----------|
| `Java heap space` | 堆分配失败且 GC 无果 | 泄漏 / 容量真不足 | dump + MAT |
| `GC overhead limit exceeded` | 98% 时间在 GC、回收 <2% | 同上（慢性版） | 同上 |
| `Metaspace` | 类元数据区满 | CGLIB/动态代理未缓存、Groovy 每请求 parse、JSP 热编译、**ClassLoader 泄漏（热部署）** | `jcmd GC.class_histogram` 看类量；MAT 看 Loader |
| `Direct buffer memory` | 直接内存超 `MaxDirectMemorySize` | Netty 未 release、Buffer 复用失效 | 见 3.6 |
| `unable to create new native thread` | OS 资源，不是堆 | ulimit -u / 线程池泄漏 / -Xss 过大 | 见第 6 节 |
| `Requested array size exceeds VM limit` | 数组长度超上限 | 异常数据量/负数计算 | 读栈定位分配点 |
| `Compressed class space` | 类指针压缩空间满 | 同 Metaspace（类数量问题） | `-XX:CompressedClassSpaceSize`（启动期定死）或控制类量；根治同 Metaspace |
| 无报错被 kill（137） | cgroup OOMKilled | 堆外/总 RSS 超限 | 见 [23_container_jvm](23_container_jvm.md) |

## 3.5 ThreadLocal 泄漏专节（深化）

### 3.5.1 问题代码与结构成因

```java
// 问题代码（保留）
public class ThreadLocalLeak {
    private static ThreadLocal<UserInfo> local = new ThreadLocal<>();

    public void process(UserInfo user) {
        local.set(user);   // 线程池线程，ThreadLocalMap 的 Entry 永不释放
        doWork();
        // 忘记 local.remove() ！  ← 泄漏
        // 正确做法：finally { local.remove(); }
    }
}
```

**结构决定命运（保留图 + 扩展）：**

```text
Thread(池线程，永不退出)
  └── ThreadLocalMap
        └── Entry extends WeakReference<ThreadLocal<?>>
              ├── Key   = ThreadLocal 对象   ← 弱引用：外部不持有时被 GC 掉 → key=null
              └── Value = UserInfo           ← 强引用：Entry 活着它就活着！
                      ↑ 泄漏主体：key 先死、value 陪葬不掉（"幽灵 Entry"）
```

### 3.5.2 remove 时机与 Tomcat 重载泄漏链

- **remove 时机图**：

```text
请求进入 ──▶ set(ctx) ──▶ 业务 ──▶ 响应 ──▶ finally remove()   ← 唯一可靠的收尸点
                              │
                              └─ 中途抛异常且无 finally → Entry.value 挂在池线程上直到下次 set 同 key
```

- 弱引用 key 只是让"没被引用的 ThreadLocal 的 Entry"下次 probe 时**顺手清一批**（expungeStaleEntry），不保证及时——所以 `ThreadLocalMap` 满值仍可能 OOM。
- **Tomcat 热部署泄漏链**（经典）：

```text
webapp A 停止 → WebappClassLoader 本应可回收
   但池线程的 ThreadLocalMap 里：value 引用了 A 的类实例
   → 实例 → Class → ClassLoader → 全部存活
   → 每次 reload 泄漏一整套（MAT 特征：多个同名 WebappClassLoader 实例）
```

- 治理：池化线程必须"用完即清"（框架层：MDC、RequestContext 都靠这个）；Tomcat 泄漏探测器会打印 `created a ThreadLocal with key of type ...` 告警。
### 3.5.3 常见泄漏场景一览（原表并入）

| 场景 | 原因 | 解决 |
|------|------|------|
| ThreadLocal 未清理 | 线程池中值未 remove | finally 块中 remove() |
| 静态集合无限增长 | static List/Map 只增不减 | WeakHashMap/LRU/定期清理 |
| 未关闭的资源 | InputStream/Connection 未关闭 | try-with-resources |
| 监听器未移除 | addListener 后未 removeListener | 成对使用 |
| 长短生命周期耦合 | 缓存持短命对象 | 软/弱引用 |
| CGLIB/代理类无限生成 | Enhancer 未缓存 | 缓存代理类 / JDK Proxy |
| JSP/脚本热编译 | 编译类累积 | 限制热加载频率、重启 |

## 3.6 直接内存（Direct Buffer）专项

### 3.6.1 症状与上限参数

```bash
# 特征：-Xmx 远没满，进程 RSS 高；报错 Direct buffer memory
-XX:MaxDirectMemorySize=512m     # 不设置时默认约等于 -Xmx，容器里很危险（见 23）
```

### 3.6.2 回收机制推导

DirectByteBuffer 本体在堆上极小，真正内存是 native malloc；它靠 **PhantomReference + Cleaner** 在"堆对象被 GC 时"释放 native 内存 →

```text
分配 → ByteBuffer.allocateDirect → Bits.reserveMemory(配额不足时?)
                                      └─ 超限时会 System.gc() 等 Cleaner 跑
→ 症状：日志/jstack 出现 java.nio.Bits.reserveMemory + System.gc 的长停顿
→ 恶性循环：堆外满 ↔ Full GC 频发（这就是"Direct 内存问题常伪装成 GC 问题"）
```

### 3.6.3 取证组合与根因清单

```bash
jmap -histo:live <pid> | grep -E "DirectByteBuffer|Unintentional"   # 实例数≈未释放 buffer 数
jcmd <pid> VM.native_memory summary | grep Internal                  # NMT 只能部分反映，别当账单
# Netty 场景：
-Dio.netty.leakDetection.level=PARANOID   # 全采样定位到分配栈（验证环境用，线上 PARANOID 很重）
# SIMPLE(默认，抽样) / ADVANCED / DISABLED 按需
```

- 根因常见：`ByteBuf` 未 `release()`（Netty 引用计数）、`FileChannel.map` 的 MappedByteBuffer **不受 MaxDirectMemorySize 限制**（unmapper 靠 GC，堆小回收少 → mmap 堆积把 RSS 撑爆）。
- `jcmd <pid> GC.heap_dump` 后 MAT 里按 `java.nio.DirectByteBuffer` 聚合 retained/cleaner 链可数出真实持有者。

## 3.7 Heap dump 生成方式对比（保留）

| 方式 | 说明 | 是否 Full GC |
|------|------|-------------|
| `jmap -dump:live,format=b,file=dump.hprof <pid>` | 先 GC 再 dump | 是 |
| `jmap -dump:format=b,file=dump.hprof <pid>` | 直接 dump（包含死对象） | 否 |
| `jcmd <pid> GC.heap_dump dump.hprof` | jcmd 方式 | 否（推荐） |
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时自动 dump | 否（OOM 发生时） |

# 4. GC 频繁 / 停顿过长

## 4.1 排查流程（保留）

```bash
# 1. 查看 GC 实时统计
jstat -gcutil <pid> 1000
# 关注：
# - YGC 增速（每秒几次 Young GC→频繁）
# - FGC 持续 > 0（Full GC 问题严重）
# - FGCT 单次时间（> 1 秒 → GC 停顿问题）

# 2. 查看 GC 日志
-Xlog:gc*=info:file=gc.log

# 3. 分析 GC 日志（GCEasy/GCPlot 上传分析，见 21）

# 4. 排查对象分配速率（Arthas）
profiler start --event alloc   # 谁在造垃圾
vmtool --action getInstances --className java.lang.Object --limit 10
```

## 4.2 解决方案参数组（保留）

```bash
# Young GC 频繁 → 增大新生代（或降分配率）
-Xmn2g

# Full GC 频繁 → 增大堆或老年代
-Xms8g -Xmx8g

# GC 停顿长 → 换成低延迟收集器
-XX:+UseG1GC -XX:MaxGCPauseMillis=100
# 或 ZGC（JDK 21 建议直接分代模式）
-XX:+UseZGC

# 大对象频繁 → 检查代码是否大数组/大集合
-XX:PretenureSizeThreshold=10m   # 仅 Serial/ParNew 有效（见 6_heap 3.4）
```

## 4.3 关键区分：GC 停顿 vs Safepoint 停顿

### 4.3.1 症状与开启

**症状**：接口毛刺但 GC 日志显示 pause 很短 → 怀疑"不是 GC，是被 safepoint 卡住"。

```bash
-Xlog:safepoint:file=safepoint.log:time,uptime
```

### 4.3.2 输出与三种判读

```text
[5.123s][safepoint] Total time for which application threads were stopped: 0.456s,
                    Stopping threads took: 0.450s     ← 同步阶段占了几乎全部
解读：
  GC 日志 pause ≈ safepoint total   → GC 问题（正常路径）
  safepoint total ≫ GC 日志 pause   → 非 GC 原因的全线程停顿：
      ① Stopping 时间长 → 某线程迟迟不到 safepoint（长计数循环/JNI 不出来）
         缓解：-XX:+UseCountedLoopSafepoints（实验性）
      ② 触发者：Thread.print(你自己在 jstack!)、revoke bias(旧版本)、deopt、
         System.gc()(检查 DisableExplicitGC)、JFR 同步、lock 膨胀
      ③ 等待时间长(Stopping 小，Total 大) → GC 本身或大量线程状态收集
```

## 4.4 分配失败类事件定位

| 日志特征 | 收集器 | 含义 | 定位链 |
|----------|--------|------|--------|
| `Evacuation Failure` / `to-space exhausted` | G1 | 复制幸存者时目标区不够 → 退化为 Full GC | 增大堆/降 `-XX:InitiatingHeapOccupancyPercent`/调 `G1ReservePercent` → 见 [14_gc_tuning](14_gc_tuning.md) |
| `Allocation Stall` | ZGC | 分配等待 GC 腾地 | 堆偏小/分配率尖峰/并发线程被容器节流（→ [23_container_jvm](23_container_jvm.md)）→ 见 [13_low_latency_gc](13_low_latency_gc.md) |
| `promotion failed` | CMS(遗留) | 晋升失败 | 已移除，老系统调 Survivor |

# 5. 线程问题（死锁与阻塞）

## 5.1 检测死锁（保留）

### 5.1.1 检测命令

```bash
# jstack 检测（只认 synchronized monitor 死锁）
jstack <pid> | grep -A 50 "Found one Java-level deadlock"

# jcmd 检测
jcmd <pid> Thread.print | grep -A 50 "deadlock"

# VisualVM / JConsole → Thread 标签 → "Detect Deadlock"
# Arthas：thread -b   （能定位 ReentrantLock 阻塞源，不限 monitor）
```

### 5.1.2 输出样例

**输出示例（保留；逐行解读版见 [20_jvm_tools](20_jvm_tools.md) 5.4）：**

```text
Found one Java-level deadlock:
============================
"Thread-1":
  waiting to lock monitor 0x00007f... (object 0x0000000...)
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f... (object 0x0000000...)
  which is held by "Thread-1"
```

## 5.2 死锁四必要条件与打破点

```text
互斥      : 锁的本质，不可破          → 只能减小粒度/分段
持有并等待: 拿着 A 等 B               → tryLock 一次拿全或全不拿（或启动时预分配）
不可剥夺  : 拿不到只能死等            → lockInterruptibly / 超时放弃
循环等待  : A→B→A 成环               → 全局固定加锁顺序（最有效）
```

## 5.3 经典锁问题模式

### 5.3.1 锁问题模式表

| 模式 | 成因 | 特征 | 解法 |
|------|------|------|------|
| 锁顺序死锁 | 两处代码加锁顺序相反 | jstack 互指成环 | 按"对象散列/编号"统一顺序 |
| 动态锁序死锁 | 用可变值算锁（如 `key.hashCode()` 选锁），锁对象中途变化 | 低概率、难复现 | 锁选取用不可变标识（`lockOf(String key)` 里用 intern/ConcurrentHashMap 固定映射） |
| 事务嵌套死锁 | DB 行锁 + JVM 锁交织：T1 持 JVM 锁等行锁，T2 持行锁等 JVM 锁 | 应用 dump + DB `show engine innodb status` 死锁日志一起看 | 缩短持锁事务；锁外提交 |
| 线程池自死锁 | 任务 A 提交任务 B 并**同步等待** B，池已满 → 全员互等 | 全池 RUNNABLE/WAITING 在 Future.get | 禁止"池内任务同步等池内任务"；分层池 |

### 5.3.2 复现代码与解决方案

**复现代码（保留）**：

```java
public class DeadLock {
    private static Object lockA = new Object();
    private static Object lockB = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (lockA) {
                Thread.sleep(100);
                synchronized (lockB) {  // 等待 lockB（被线程2 持有）
                    System.out.println("Thread1 done");
                }
            }
        }).start();

        new Thread(() -> {
            synchronized (lockB) {
                Thread.sleep(100);
                synchronized (lockA) {  // 等待 lockA（被线程1 持有）
                    System.out.println("Thread2 done");
                }
            }
        }).start();
    }
}
```

**解决方案（保留 + 补充）**：

1. **固定加锁顺序**：所有线程按相同顺序获取锁。
2. **使用 tryLock 超时**：`ReentrantLock.tryLock(timeout, unit)`。
3. **避免嵌套锁**：尽量不持有锁时去获取另一个锁。
4. 上层化：单锁/无锁结构/消息串行化，从架构上消灭"多锁"。

## 5.4 线程堆积判读（不止死锁）

| 快照特征 | 判读 | 依据 |
|----------|------|------|
| 大量 `WAITING (parking)` 且 `parking to wait for ...ConditionObject` | **正常**：池空闲等任务 | 数量≈池 coreSize；别慌 |
| `parking` 线程数远超池配置，且池名多样 | 池泄漏/每请求建池 | 见 6.4 |
| 大量 BLOCKED `waiting to lock <同一地址>` | 单点锁热点 | 顺着 `<地址>` 找 `locked` 持有者的栈 |
| 大量 RUNNABLE `socketRead0` | 下游 hang（DB/HTTP） | 响应慢第 7 节 |
| TIMED_WAITING `poll` 于队列 | 队列空转消费，正常 | 结合队列指标 |

# 6. 线程过多 / Unable to create native thread

## 6.1 现象（保留）

```text
java.lang.OutOfMemoryError: unable to create new native thread
```

## 6.2 排查（保留 + 补充 OS 视角）

```bash
jstack <pid> | grep "Thread" | wc -l          # JVM 视角线程数
ls /proc/<pid>/task | wc -l                   # OS 视角（两者差异大 → 有非托管线程/JNI）
ulimit -u                                     # 用户进程/线程数限制（nproc）
cat /proc/sys/kernel/threads-max              # 系统全局上限
cat /proc/sys/vm/max_map_count                # 每线程栈至少 2 个 mmap，此值小会先撞墙
free -h                                       # 还有没有内存给新栈
```

## 6.3 原因和解决（保留）

| 原因 | 解决方案 |
|------|----------|
| 线程数太多（> 系统上限） | 使用线程池限制线程数 |
| 每个线程栈太大（Xss） | 减小 -Xss（如 -Xss256k） |
| 内存不足（线程栈占用物理内存） | 增加物理内存或减少 -Xmx |
| 线程泄漏（创建后不回收） | 排查线程池是否正确销毁 |

```bash
-Xss256k    # 减小栈大小，允许创建更多线程
```

## 6.4 线程池泄漏排查（保留 + 反模式清单）

### 6.4.1 池名统计命令

```bash
# 按池名统计（线程名是"出身证明"）
jstack <pid> | grep -oP '"\K[^"]+(?=")' | sed 's/[-_]\?\d*$//' | sort | uniq -c | sort -rn | head
jstack <pid> | grep "pool-" | wc -l     # pool-N-thread-M 计数持续增长 → 泄漏
```

### 6.4.2 线程池反模式清单

| 反模式 | 后果 |
|--------|------|
| 方法内 `new ThreadPoolExecutor(...)` 用完不 `shutdown()` | 每次调用漏一个核心线程，永不退出 |
| `Executors.newCachedThreadPool()` | 无界线程数，瞬时并发→几千线程 |
| 动态配置刷新时重建池不关旧池 | 双池并存，旧池"僵尸化" |
| 定时任务 `new Timer()`/`scheduleAtFixedRate` 不 cancel | 线程常驻 |
| 虚拟线程时代滥用 `newCachedThreadPool` | 同上；应换 `Executors.newVirtualThreadPerTaskExecutor()`（JDK 21） |

治理模板：池统一注册中心（命名 + shutdown hook 优雅关闭 + 指标暴露 activeSize/queueSize）。

# 7. 服务响应变慢

## 7.1 排查维度（保留）

| 可能原因 | 排查工具 | 关键指标 |
|----------|----------|----------|
| GC 停顿 | jstat / GC 日志 | FGC > 0, GC 停顿 > 1s |
| 线程 BLOCKED | jstack / arthas thread | 大量 BLOCKED 线程 |
| 线程池满 | jstack | 大量 WAITING (parking) 在队列取任务 |
| CPU 高 | top / arthas | CPU > 80% |
| 外部依赖慢 | arthas trace | 数据库/Redis/HTTP 调用耗时 |
| 连接池耗尽 | jstack | 大量线程等待获取连接 |

## 7.2 三段判定法（下游依赖 vs GC 停顿 vs 排队）

### 7.2.1 判定流程

```text
拿一次慢请求的时间戳 T，依次对表：
  ① GC/safepoint 日志在 T±1s 有停顿?         ──▶ GC 问题（第 4 节）
  ② arthas trace 显示耗时集中在外部调用帧    ──▶ 下游问题（DB/Redis/RPC，看超时与连接池）
  ③ 外部调用不慢但 total ≫ Σ内部           ──▶ 排队问题：
       线程池 active=max + 队列堆积（vmtool 取 pool.getQueue().size()）
       或 Tomcat acceptor 队列满 / 连接池等待
```

### 7.2.2 三段各一句话证据

| 段 | 一句话证据 |
|----|------------|
| 下游慢 | trace 里 `JedisCommands`/`HttpClient` 帧宽 |
| GC 停顿 | 毛刺周期性与 GC 对齐、safepoint 日志 |
| 排队 | 方法自身耗时正常但**端到端慢**；池指标饱和 |

## 7.3 工具链（保留 + JFR）

```bash
# Arthas：监控方法耗时 / 追踪链路
monitor com.example.controller.* -c 10
trace com.example.controller.OrderController getOrder '#cost > 500'

# JFR：不改代码回看历史毛刺
jcmd <pid> JFR.dump filename=slow.jfr   # JMC 打开看 Threads/Method Profiling 页面
```

# 8. 其他 OOM 场景

## 8.1 元空间溢出（保留）

```text
java.lang.OutOfMemoryError: Metaspace
```
- 原因：动态生成类过多（CGLIB、动态代理、Groovy、JSP）。
- 定位：`jcmd <pid> GC.class_histogram | head`；Arthas `classloader -l` 看各加载器实例计数；MAT 数 ClassLoader。
- 解决：增大/固化 `-XX:MaxMetaspaceSize`；缓存代理类；Groovy `parseClass` 结果缓存。

## 8.2 直接内存溢出（保留 → 详见 3.6）

```text
java.lang.OutOfMemoryError: Direct buffer memory
```
- 原因：NIO 的 `ByteBuffer.allocateDirect()` 未释放。
- 排查：jcmd VM.native_memory summary（部分可见）+ MAT 数 DirectByteBuffer。
- 解决：限制 `-XX:MaxDirectMemorySize`，排查代码是否正确释放。

## 8.3 GC Overhead 超限（保留）

```text
java.lang.OutOfMemoryError: GC overhead limit exceeded
```
- 含义：超过 98% 的时间在 GC，但只回收了 <2% 的堆。
- 解决：分析堆转储、检查内存泄漏、增大堆。

```bash
-XX:-UseGCOverheadLimit    # 关闭此限制（不推荐，但可紧急使用；结果往往只是晚点 OOM）
```

## 8.4 无法创建线程（保留 → 详见第 6 节）

```text
java.lang.OutOfMemoryError: unable to create new native thread
```
- 含义：系统资源不足以为新线程分配栈空间/超线程上限。
- 解决：减小 `-Xss`、限制线程数、排查线程泄漏。

# 9. 生产环境排查速查卡（升级版）

| 症状 | 三条命令（按序执行） | 判读指标 |
|------|----------------------|----------|
| CPU 100% | `top -Hp <pid>` → `printf %x` → `jstack <pid> \| grep -A30 nid=0x..` | 高线程是业务线程(GC 线程)？栈顶方法稳定吗？ |
| 响应 P99 毛刺 | `jstat -gcutil <pid> 1000 10` → `tail gc.log` → `-Xlog:safepoint` | FGC 增速、pause 时长、safepoint total≫pause？ |
| 内存告警/OOM | `jcmd <pid> GC.heap_info` → `jcmd <pid> GC.class_histogram \| head -30` → `jcmd <pid> GC.heap_dump /data/d.hprof` | OU 趋势、Top 类是否业务类、dump 交 MAT |
| 进程被 kill(137) | `kubectl describe pod` 看 lastState → `jcmd <pid> VM.native_memory summary.diff` → `pmap -x <pid>` | 是否 OOMKilled、NMT 哪区在涨 |
| 死锁/卡死 | `jstack -l <pid>` → grep deadlock 段 → Arthas `thread -b` | Found one Java-level deadlock？BLOCKED 聚集锁地址？ |
| 线程爆炸 | `jstack <pid> \| grep "^\"" \| wc -l` → 池名 `uniq -c` → `ulimit -u` | pool-N 计数、哪类线程增长 |
| GC 频繁 | `jstat -gcutil` → GC 日志上 GCPlot → JFR GC 页面 | YGC 频率、晋升速率、堆是否够 |
| 通用体检 | `jps -lv` → `jcmd <pid> VM.info` → `jcmd <pid> VM.flags` | 版本/uptime/实际生效参数（防"以为配了"） |

```bash
# 顺手保留的老版一键清单（原速查卡）
jps -lv; jstat -gcutil <pid> 1000 5; jmap -histo <pid> | head -10
jstack <pid> | grep -A 20 BLOCKED; jcmd <pid> GC.heap_dump /tmp/dump.hprof
```

# 10. 虚拟线程排障（JDK 21，JEP 444）

## 10.1 新故障形态：Pinning

### 10.1.1 机制

```text
虚拟线程 V1 ──挂载──▶ 载体线程 C1（ForkJoinPool，默认数量=CPU 核数）
   synchronized 块内发生阻塞（IO/park/sleep）
   → V1 无法卸载，C1 被"钉住"（pinned）
   → 载体池是全局稀缺资源，几个 pinning 就拖垮所有虚拟线程的调度
```

### 10.1.2 典型代码与对策

典型代码（21 仍会中招）：

```java
synchronized (cache) {
    var result = httpClient.send(...);  // 块内阻塞 IO → pinning！
}
// 对策一：换 ReentrantLock —— lock() 的 park 不 pin
private static final ReentrantLock lock = new ReentrantLock();
// 对策二：缩小 synchronized 范围，阻塞调用挪出块外
// 对策三：升级到 JDK 24（JEP 491）—— synchronized 不再 pin 虚拟线程
```

## 10.2 取证

### 10.2.1 两条命令通道

```bash
# ① JVM 参数打印 pinning 栈：short=仅标记一次；full=每次 pin 打全栈
-Djdk.tracePinnedThreads=full

# ② JFR 事件（推荐，可量化）：
jcmd <pid> JFR.start settings=profile ...
jfr print --events jdk.VirtualThreadPinned recording.jfr | head -50
# 事件含完整栈帧；默认阈值 20ms（持续 20ms 以上的 pin 才记录）
```

### 10.2.2 输出样态

```text
输出样态（示意，具体措辞随版本变化）：每条包含 "pinned" 标记 +
虚拟线程栈 + - locked <地址> 行 —— 用栈顶的 synchronized 方法定位改造点
```

## 10.3 其它虚拟线程观察项

| 观察 | 方法 | 说明 |
|------|------|------|
| 调度健康度 | JFR `jdk.VirtualThreadPinned` 计数 + `ForkJoinPool` 载体线程 CPU 利用率 | pinning 事件数≈0 为健康 |
| 载体池饥饿 | jstack 中 `ForkJoinPool-N-worker` 全 RUNNABLE 卡在业务阻塞帧 | 多为 pinning 或大量同步 native 调用 |
| 大栈成本 | 深栈/大局部引用的虚拟线程，unmount 时栈帧存于堆，内存与调度开销上升 | 避免池化大 Buffer 到局部变量；用对象池显式管理 |
| ThreadLocal 适配 | 虚拟线程数量大，ThreadLocal 使用需谨慎（内存放大） | JEP 429 提供 ScopedValue（21 预览）替代方案观察中 |
| 版本节点 | JDK 21 JEP 444 转正；**JDK 24 JEP 491 解决 synchronized pinning**；JNI 本地帧仍会 pin | 升级前先清 synchronized 习惯 |

# 11. 必备 JVM 参数（生产环境）（保留）

```bash
# 基础
-Xms4g -Xmx4g                       # 堆大小（建议相等）

# GC
-XX:+UseG1GC                         # 垃圾收集器
-XX:MaxGCPauseMillis=100             # 最大停顿目标

# 日志
-Xlog:gc*=info:file=/data/logs/gc.log:time,pid:filecount=10,filesize=100M

# 故障排查
-XX:+HeapDumpOnOutOfMemoryError       # OOM 时 dump
-XX:HeapDumpPath=/data/logs/          # dump 路径
-XX:ErrorFile=/data/logs/hs_err_pid%p.log  # 崩溃日志
-XX:+ExitOnOutOfMemoryError          # OOM 时退出（让 k8s 重启）
-XX:+PrintCommandLineFlags           # 打印 JVM 参数
-XX:StartFlightRecording=maxsize=512m,maxage=6h,dumponexit=true   # JFR 常开（11+）

# 元空间
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m

# 安全
-XX:+DisableExplicitGC               # 禁用 System.gc()
#   注意：若依赖 RMI/DirectByteBuffer 的系统级 System.gc 回收，需评估（见 3.6 回收机制）
```
