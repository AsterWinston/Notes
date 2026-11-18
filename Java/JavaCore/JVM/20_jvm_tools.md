# 1. 工具全景图

## 1.1 问题域 → 工具矩阵

| 问题域 | 首选工具 | 备用/进阶 | 输出去向 |
|--------|----------|-----------|----------|
| CPU 高 | `jstack`（多次采样）+ `top -Hp` | async-profiler 火焰图、Arthas `thread -n` | 文本 / HTML |
| 内存 / OOM | `jmap -histo`、`jcmd GC.heap_dump` | JFR 分配事件、MAT 分析 dump | hprof → MAT |
| GC 行为 | `jstat -gcutil`、GC 日志（`-Xlog:gc*`） | JFR（GC 页面）、GCEasy/GCPlot | 日志 → 可视化 |
| JVM 配置 | `jinfo`、`jcmd VM.flags` | `jcmd VM.log`（动态开日志） | 文本 |
| 线程 / 死锁 | `jstack -l` / `jcmd Thread.print -all` | Arthas `thread -b`、JFR 锁事件 | 文本 |
| 堆外内存 | `jcmd VM.native_memory`（NMT） | pmap、smaps | 文本，见第 10 节 |
| 运行时"不改代码改行为" | —— | Arthas watch/trace/retransform | 见 [21_visual_tools](21_visual_tools.md) |

## 1.2 家族谱系与版本节点

```text
JDK 5/6   jps  jstat  jmap  jstack  jinfo  (jhat)
JDK 7           + jcmd（逐步成为"万能入口"）
JDK 8     jhat 名存实亡；jmap -histo:live / -dump 仍是主力
JDK 9           jhat 移除 → jhsdb（clhsdb/jhsdb heap/stacks 等，低频使用）
                JFR/JMC 标记为商业特性（需 UnlockCommercialFeatures）
JDK 11          JFR 开源（JEP 328），生产常开成为可能；CMS 废弃
JDK 16+         部分诊断能力并入 jcmd/统一日志（-Xlog），jmap -heap 等
                老命令逐步被 "jcmd 对应子命令 + 日志装饰器" 取代
JDK 21 (LTS)    jcmd Thread.print -all 覆盖虚拟线程；NMT/JFR 成熟
外部生态        Arthas（阿里开源）、async-profiler：生产在线诊断主力
```

> 趋势判断：**单机命令行（j* 系列）→ jcmd 统一入口 → JFR 持续采集 + 外部 Agent（Arthas）**。新代码优先学 jcmd + JFR + Arthas 三件套，j* 命令作为快速兜底。

# 2. jps — 查看 Java 进程

## 2.1 用途与参数

`jps`（Java Process Status）列出当前用户可见的 JVM 进程。

```bash
jps          # PID + 主类名（简写）
jps -q       # 只输出 PID（适合脚本管道）
jps -m       # 附带动 main() 的参数：67890 MyApplication --spring.profiles.active=prod
jps -l       # 完整主类名 / jar 全路径：67890 com.example.MyApplication
jps -v       # 全部非默认 JVM 参数：67890 MyApplication -Xms4g -Xmx4g -XX:+UseG1GC
jps -lv      # 组合，最常用的"看一眼进程长什么样"
```

## 2.2 原理与坑

```text
JVM 启动时把进程信息写入共享内存文件：
  Linux/macOS: /tmp/hsperfdata_<用户名>/<pid>
  Windows:     %TMP%\hsperfdata_<用户名>\<pid>
jps 只是这个目录的"阅读器"，不是查 OS 进程表。
```

| 现象 | 原因 | 对策 |
|------|------|------|
| 进程明明活着，jps 看不到 | 其他用户启动的进程（hsperfdata 按用户隔离） | 用同用户执行，或 `ps -ef \| grep java` |
| 看不到某进程 | 该 JVM 加了 `-XX:+PerfDisableSharedMem`（为消除 perfdata 写文件导致的 safepoint 抖动） | 只能用 `ps`；jstat/jinfo 同样受影响 |
| 容器里看不到 PID 1 的进程 | 老版本 JDK 已知行为 | 直接 `jstack 1` / `jcmd 1` |

# 3. jstat — JVM 统计监控

## 3.1 语法骨架

```bash
jstat -<option> [-interval] <pid> [interval(ms)] [samples]
jstat -gcutil 12345 1000 10     # 每 1s 一次，共 10 次
```

> 旧的远程模式（`pid@rmid主机`）依赖已淘汰的 rmid，现代实践一律本地 attach 或进容器执行。

## 3.2 -gc：各代容量与用量（单位 KB）

### 3.2.1 输出样态

```bash
jstat -gc <pid> 1000
# S0C    S1C    S0U    S1U     EC      EU      OC      OU      MC     MU     CCSC   CCSU   YGC   YGCT   FGC   FGCT   GCT
```

### 3.2.2 字段速查表

| 列 | 含义 | 判读要点 |
|----|------|----------|
| S0C/S1C | Survivor 容量 | 两值相等（From/To 对称） |
| S0U/S1U | Survivor 用量 | 恒有一个接近 0（To 区） |
| EC/EU | Eden 容量/用量 | EU 锯齿触顶 → Young GC |
| OC/OU | 老年代容量/用量 | **每次 GC 后 OU 只增不减 → 泄漏** |
| MC/MU | Metaspace 容量/用量 | **持续攀升 → 类加载泄漏（动态代理/ClassLoader）** |
| CCSC/CCSU | Compressed Class Space 容量/用量 | JDK 8+ |
| YGC/YGCT | Young GC 次数/累计耗时 | YGCT/YGC = 平均单次停顿 |
| FGC/FGCT | Full GC 次数/累计耗时 | JDK 9+ G1 下 FGC 仅统计 STW Full GC |
| GCT | GC 总耗时 | GCT/运行时长 = GC 时间占比（吞吐） |

## 3.3 -gcutil：百分比版（最常用）

### 3.3.1 输出样态

```bash
jstat -gcutil <pid> 1000 10
#   S0     S1     E      O      M     CCS    YGC     YGCT     FGC    FGCT     GCT
#   0.00  12.34  45.67  67.89  78.90  80.12   100     5.23     2      1.23   6.46
# 列含义同 -gc，但全部换算成百分比，趋势判读更直观
```

### 3.3.2 实用组合与趋势判读

```bash
# 黄金 10 秒采样：先看趋势再下结论
jstat -gcutil <pid> 1000 10
```

| 观察 | 结论 | 下一步 |
|------|------|--------|
| O 区 GC 后仍阶梯上涨 | 堆内存泄漏 | dump + MAT，见 [22_troubleshooting](22_troubleshooting.md) |
| M 区持续上涨不回落 | 元空间/类加载问题 | `jcmd <pid> GC.class_histogram` 看类数量 |
| YGC 频率高但 OU 稳定 | 分配率高，新生代偏小 | 调 -Xmn / 查分配热点（JFR） |
| FGC 频繁 + FGCT 大 | Full GC 问题 | GC 日志 + GCEasy，见 [14_gc_tuning](14_gc_tuning.md) |

## 3.4 -gccapacity / -gcnew / -gcold：容量画像

```bash
jstat -gccapacity <pid>     # 各代"应有大小"：MNMC MAXNMC MNMC MNMUSE... + S0C..OC OU..
jstat -gcnew <pid>          # 新生代细节：S1C S1U S0C S0U MNC MDC MXX DESIRED TT YGC YGCT
                            #   DESIRED=目标 survivor 大小, TT=当前 tenuring threshold（动态年龄的体现）
jstat -gcnewcapacity <pid>  # 新生代容量历史：MC MNC MDC MXX(最大) MNMC MNC MAXNMC(历史峰值)
jstat -gcold <pid>          # 老年代：MNC MXX MSC MXSC MCMXSC MU CCSC CCSU FGC FGCT GCT
jstat -gcoldcapacity <pid>  # 老年代容量增长历史（看堆是否还在扩容）
```

> 用途：区分"JVM 还没扩到 -Xmx"与"堆真的满了"。`-gcoldcapacity` 的 MNC 仍在增长 → 堆尚未到位，Full GC 可能是扩容前的保守触发（MinHeapFreeRatio/MaxHeapFreeRatio，见 [6_heap](6_heap.md)）。

## 3.5 -class / -compiler / -printcompilation

```bash
jstat -class <pid>
#  Loaded  Bytes  Unloaded  Bytes     Time
#   12345  23456.7     12    100.2    3.45
# 解读：Loaded 持续增长 + Unloaded≈0 → 类加载泄漏嫌疑（对比 MU 增长）

jstat -compiler <pid>
# Compiled Failed Invalid   Time   FailedType  FailedMethod
#     4321      0       0   1.25   Standard      com.foo.Bar::baz
# 解读：Compiled 在"稳定期"仍暴涨 → JIT 编译风暴（启动期正常）

jstat -printcompilation <pid> 1000
# Compiled    Size  Type  Method                 # 最近一次编译：Type 1=C1, 2=C2, 4=C1+J9? 常见 1=C1 2=C2
#      4322     56     2  java/lang/String::hashCode
```

# 4. jmap — 内存映像工具

## 4.1 堆摘要

```bash
jmap -heap <pid>        # JDK 17 起移除，改用 jcmd <pid> GC.heap_info
# Heap Configuration:
#    MaxHeapSize = 4294967296 (4096.0MB)
#    ...
# Heap Usage:
#  PS Young Generation
#    Eden Space:  capacity/used/free, 49.7% used
```

## 4.2 对象直方图

### 4.2.1 三种用法

```bash
jmap -histo <pid>              # 全堆统计（含未回收垃圾），不触发 GC
jmap -histo:live <pid>         # 只统计存活对象 —— 会先触发一次 Full GC（STW）！
jmap -histo:file=/tmp/h.txt <pid>   # 输出到文件
```

### 4.2.2 输出样态与判读（含 `-histo:live` 代价）

```text
 num     #instances         #bytes  class name
   1:       1234567       98765432  [B                 ← byte[]，IO/序列化缓冲
   2:        654321       52425680  java.lang.String
   3:         12345       12345600  [C                 ← char[]（JDK8 字符串底层）
   4:          6789        5432100  java.util.HashMap$Node
```

判读：`[B`/`[C` 排第一是常态；**业务类、`ClassLoader`、`Class` 实例数异常多**才是泄漏信号；对比两次 `-histo`（不带 live，避免 STW）看增长量。

## 4.3 dump 的两种格式与两种通道

| 方式 | 产物 | 特点 |
|------|------|------|
| `jmap -dump:format=b,file=x.hprof <pid>` | hprof（对象图） | 可移植，MAT/JVisualVM 可开；STW |
| `jmap -dump:format=c,file=x.core <pid>` | core（进程内存镜像） | 仅同平台 jhsdb/MAT 特殊配置可读，几乎不用 |
| `jmap -dump:live,...` | hprof，仅存活对象 | **先 Full GC**，文件小但停顿更久 |
| `jcmd <pid> GC.heap_dump [-all] /tmp/x.hprof` | hprof | **生产首选**（走 safepoint 机制更稳，可控 -all 是否含垃圾） |

## 4.4 注意事项

- `jmap -F`（force）走 SA（Serviceability Agent）后端解析进程内存，慢且可能失败，仅进程僵死时用。
- 大堆（>8G）dump 的 STW 可达数十秒 + 磁盘 IO，**提前规划窗口**，或事先配好 `-XX:+HeapDumpOnOutOfMemoryError` 让 OOM 自动留现场。
- 较新 JDK 在 Windows 上 `jmap` 存在限制/不稳定，生产 Linux 无碍，Windows 优先 `jcmd`。
- `jmap -heap` JDK 17 起被移除：**统一改用 `jcmd GC.heap_info`**。

# 5. jstack — 线程堆栈快照

## 5.1 用途与参数

```bash
jstack <pid>            # 全部 Java 线程栈
jstack -l <pid>         # 附带锁信息（死锁检测必用）
jstack -m <pid>         # mixed mode：Java + 本地(C/C++) 帧混合栈
jstack -f <pid>         # 强制忽略本地方法帧
jstack -F <pid>         # force：线程无响应时用（走 sa 后端，可能失准/挂起）
jstack <pid> > t1.txt 10s 后再来一份 → 对比
```

## 5.2 输出逐字段解读

```text
"http-nio-8080-exec-1" #12 daemon prio=5 os_prio=0 cpu=1032.15ms elapsed=8421.03s tid=0x00007f1234001000 nid=0x1234 waiting on condition  [0x00007f1234567000]
│                       │        │           │              │                      │
│                       │        │           │              │                      └ 栈上当前帧地址
│                       │        │           │              └ 线程起始栈地址        └ nid=OS 线程ID(十六进制!)，与 top -Hp 对应
│                       │        │           └ tid=JVM 线程结构指针
│                       │        └ os_prio: OS 调度优先级；cpu: 累计 CPU 时间（找热点线程看这个!）
│                       └ 线程序号/是否守护
└ 线程名（线程池名在这里，统计池名=统计线程泄漏）

   java.lang.Thread.State: WAITING (parking)          ← 状态
    at sun.misc.Unsafe.park(Native Method)            ← 栈帧：at 类.方法(文件:行)
   - parking to wait for  <0x000000076ab12345>        ← 等待的对象（锁/条件）
      (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   - locked <0x000000076ab99999>                       ← 已持有的监视器锁（synchronized）
      (a java.lang.Object)
```

## 5.3 线程状态语义

| 状态 | 含义 | 排查含义 |
|------|------|----------|
| RUNNABLE | 运行中**或等 CPU**（含阻塞 IO? 含——IO 在 NIO 语义下也算 RUNNABLE，老 BIO 读也显示 RUNNABLE 卡在 socketRead0） | CPU 高找它；`socketRead0` 多 = 下游慢 |
| BLOCKED | 等 `synchronized` 监视器锁 | 聚集同一 `locked` 对象 → 单点锁 |
| WAITING | `wait()/join()/park()` 无限等 | 池空闲属正常；持续变多 = 依赖 hang |
| TIMED_WAITING | `sleep()/wait(timeout)` | 定时任务/限流器 |
| TERMINATED | 已退出 | 快照里看不到，只算线程数 |

## 5.4 死锁输出逐行解读

### 5.4.1 输出全文样例

```text
Found one Java-level deadlock:              ← jstack -l 的结论行
============================
"Thread-1":
  waiting to lock monitor 0x00007f...       ← 想要的锁地址（JVM 内部 monitor）
  (object 0x000000076ab12345, a com.example.LockB),
  which is held by "Thread-2"               ← 被 Thread-2 持有 —— 形成边 T1→B→T2
"Thread-2":
  waiting to lock monitor 0x00007f...
  (object 0x000000076ab54321, a com.example.LockA),
  which is held by "Thread-1"               ← 边 T2→A→T1，成环 → 判定死锁

Java stack information for the threads listed above:
"Thread-1":
    at com.example.ServiceA.methodA(ServiceA.java:10)
    - waiting to lock <0x000000076ab12345> (a com.example.LockB)   ← 卡在这里
    - locked <0x000000076ab54321> (a com.example.LockA)            ← 已拿着 A
"Thread-2":
    at com.example.ServiceB.methodB(ServiceB.java:15)
    - waiting to lock <0x000000076ab54321> (a com.example.LockA)
    - locked <0x000000076ab12345> (a com.example.LockB)
Found 1 deadlock.
```

### 5.4.2 解读要点

- `waiting to lock` + `locked` 两行对照即可手工复原"等待图"；成环即死锁。
- ReentrantLock 死锁 **jstack 不自动报**（不是 monitor），要人工看 `parking to wait for <ConditionObject>` 聚集 → 或 Arthas `thread -b`。

## 5.5 CPU 高线程定位法（经典四步，保留并加注释）

### 5.5.1 经典四步链路

```bash
# 1. 找到高 CPU 进程
top -c
# 2. 找到进程内高 CPU 线程（TID 为十进制）
top -Hp <pid>
# 3. 转十六进制（jstack 里的 nid 是十六进制）
printf "%x\n" <tid>          # 303e
# 4. 在栈里搜
jstack <pid> | grep -A 30 "nid=0x303e"
```

### 5.5.2 多次采样对比找热点

无火焰图工具时的土办法：

```bash
for i in $(seq 1 10); do jstack <pid> >> snaps.txt; sleep 1; done
# 统计：同一线程 10 次快照有 8 次停在同一方法 → 热点方法
grep -A 1 '"http-' snaps.txt | grep 'at ' | sort | uniq -c | sort -rn | head
```

# 6. jinfo — 查看与动态修改参数

```bash
jinfo <pid>                        # 输出全部 XX 参数 + 系统属性
jinfo -flag MaxHeapSize <pid>      # -XX:MaxHeapSize=4294967296
jinfo -flags <pid>                 # 非默认参数一览
jinfo -sysprops <pid>              # System.getProperties() 实况（含 -D 与容器环境变量注入差异）

jinfo -flag +PrintGC <pid>         # 开启 boolean 开关
jinfo -flag -PrintGC <pid>         # 关闭
jinfo -flag MaxHeapFreeRatio=70 <pid>   # 修改带值参数
```

限制与要点：

| 要点 | 说明 |
|------|------|
| 只能改 **manageable** 参数 | `jinfo -flag` 本质是写 flag 表；非 manageable（如 Xmx、GC 选型）会报 `Command failed in ...` |
| 查哪些能改 | `jcmd <pid> VM.set_flag <Name> <Value>` 官方支持集有限（CICompilerCount、ConcGCThreads、ParallelGCThreads、Max/MinHeapFreeRatio 等），**以现场 `jcmd help` 为准** |
| 依赖 perfdata | 与 jps/jstat 同源，`PerfDisableSharedMem` 开启后失效 |
| JDK 11+ 替代 | `jcmd VM.log` 可动态开/关任意日志域，比开 PrintGC 老式参数实用得多 |

# 7. jcmd — 综合诊断入口（JDK 7+，生产首选）

## 7.1 语法

```bash
jcmd                                   # 列出 JVM 进程（= jps -l）
jcmd <pid> help                        # 该进程支持的命令全集（不同 JDK 略有差异，以 help 为准）
jcmd 12345 GC.heap_dump /tmp/d.hprof   # 子命令 + 参数
```

## 7.2 常用命令全表（约 18 条，按组）

### 7.2.1 VM.* 组（配置与运行时信息）

| 命令 | 用途 |
|------|------|
| `VM.version` / `VM.uptime` | 版本 / 运行时长 |
| `VM.command_line` | 完整启动命令与参数（含 -D、-agentlib） |
| `VM.flags` / `VM.flags -all` | 非默认 flag / 全部 flag（含默认值，替代 jinfo -flags） |
| `VM.set_flag <N> <V>` / `VM.reset_flag <N>` | 动态改/恢复 manageable flag |
| `VM.system_properties` | 系统属性 |
| `VM.info` | 线程/GC/编译/内存综合摘要（一份命令看全貌） |
| `VM.log` | **动态管理统一日志**：`-list`、`-create what=gc*=debug`（不重启开 GC 日志） |
| `VM.native_memory summary` | NMT 报告，见第 10 节 |

### 7.2.2 Thread.* 与 GC.* 组（现场取证主力）

| 命令 | 用途 |
|------|------|
| `Thread.print` | 线程栈；`-all` 含虚拟线程（JDK 21+）；`-l` 锁信息 |
| `GC.heap_info` | 堆配置与各区用量（替代已移除的 `jmap -heap`） |
| `GC.class_histogram` | 类直方图；`-all` 含垃圾，默认只存活（**默认即触发 Full GC，注意 STW**） |
| `GC.heap_dump [-all] <file>` | 堆转储，**生产推荐**（相对 jmap 更安全、可控） |
| `GC.run` | 触发 Full GC（验证"GC 后 OU 是否回落"= 泄漏初筛） |
| `GC.finalizer_info` | Finalizer 队列状态（finalize 泄漏排查） |

### 7.2.3 JFR.* / Compiler.* / ManagementAgent.* 组

| 命令 | 用途 |
|------|------|
| `JFR.start/dump/checkpoint/stop` | 见 [21_visual_tools](21_visual_tools.md) JFR 节 |
| `Compiler.codecache` | CodeCache 使用与淘汰统计（CodeCache 满 → 停止 JIT，变慢） |
| `Compiler.per_method_thresholds` / `Compiler.command` | JIT 阈值查看 / dcmd 编译控制（进阶，少用） |
| `ManagementAgent.start_local` / `.status` | 不重启拉起本地 JMX（配合 JConsole） |

## 7.3 典型工作流

```bash
jcmd 12345 VM.flags -all > flags.txt
jcmd 12345 Thread.print -all > t_$(date +%H%M%S).txt   # 间隔 10s 采 3 份
jcmd 12345 GC.heap_info
jcmd 12345 GC.class_histogram | head -30
# 决定要现场时：
jcmd 12345 GC.heap_dump /data/dumps/d_$(date +%F_%H%M).hprof
```

# 8. Linux 配套工具

## 8.1 vmstat — 整机视角

```bash
vmstat 1
# r  b   swpd   free  buff  cache  si  so   bi   bo   in   cs  us sy id wa st
# cs 过高 → 线程争抢/锁太多；si/so 非 0 → 已用 swap，GC 停顿会被放大（容器应禁 swap，见 [23_container_jvm](23_container_jvm.md)）
# st 高 → 虚拟机被宿主抢占；wa 高 → 磁盘慢（dump/日志写入拖垮 GC）
```

## 8.2 pidstat — 进程/线程级 CPU

```bash
pidstat -p <pid> -t 1     # -t：按线程展开，直接给 %CPU，省掉手动转十六进制前的一步
pidstat -w -p <pid> 1     # 上下文切换：cswch/s（自愿=等锁） vs nvcswch/s（非自愿=CPU 不够）
```

## 8.3 perf — OS 级热点（含内核态）

```bash
perf top -p <pid>                 # 进程视角的实时函数热点（含 libc/内核符号）
perf record -g -p <pid> -- sleep 30 && perf report
# 场景：热点卡在 malloc/memcpy/futex 等 native 函数，Java 栈看不穿时用
# 需要 -XX:+PreserveFramePointer 或 fp 帧才能完整展开 Java 栈（要求较高，浅用即可）
```

## 8.4 async-profiler 一句话简介

低开销采样分析器，基于 JVMTI + perf_events，**JDK 8u112+ 的安全注入路线（不 suspend 进程）**：

```bash
./asprof -d 30 -f cpu.html  <pid>          # CPU 火焰图
./asprof -d 30 -e alloc -f alloc.html <pid>  # 分配火焰图（找垃圾源头）
./asprof -d 30 -e lock  -f lock.html  <pid>  # 锁争用火焰图
```

火焰图判读与 Arthas `profiler` 集成见 [21_visual_tools](21_visual_tools.md)；CPU 飙高实战见 [22_troubleshooting](22_troubleshooting.md)。

# 9. 工具选择速查表

| 排查目标 | 工具 | 常用命令 |
|----------|------|----------|
| 进程状态 | jps | `jps -lv` |
| GC 实时监控 | jstat | `jstat -gcutil <pid> 1000` |
| 内存 Top 类 | jmap/jcmd | `jcmd <pid> GC.class_histogram \| head -20`（注意默认触发 GC） |
| Heap Dump | **jcmd** | `jcmd <pid> GC.heap_dump dump.hprof` |
| 线程状态/死锁 | jstack | `jstack -l <pid>`；多次采样 |
| JVM 参数 | jinfo/jcmd | `jcmd <pid> VM.flags -all` |
| 动态开日志 | jcmd | `jcmd <pid> VM.log -create what=gc*=debug` |
| 堆外内存 | jcmd NMT | `jcmd <pid> VM.native_memory summary.diff`（需启动带参数） |
| CPU 高 | top + jstack | `top -Hp` → `nid=0x..` 搜索；或 async-profiler |
| 不重启插桩 | Arthas | trace/watch/tt（见 21） |

# 10. NMT — JVM 原生内存追踪

## 10.1 定位与开启

NMT（Native Memory Tracking）追踪 **JVM 自身 C++ 代码** 经 malloc/mmap 的分配（不含堆内对象，那由 GC 视角覆盖）：

```bash
# 只能启动时开启（manageable? 否，diagnostic 且不可动态开启）
-XX:NativeMemoryTracking=summary     # 按区域汇总，开销较小
-XX:NativeMemoryTracking=detail      # 细化到调用点，开销更大
-XX:+UnlockDiagnosticVMOptions -XX:NativeMemoryTracking=detail   # 部分版本需先解锁诊断参数
```

| 维度 | 数据 |
|------|------|
| 内存开销 | 约 5%~10%（额外维护每分配 16B 头） |
| CPU 开销 | 每次 malloc/free 加计数，可感知但通常 <5% |
| 结论 | 生产**按需开启**（怀疑堆外泄漏的实例灰度开），不开全量 |

## 10.2 输出分类表

### 10.2.1 区域一览

```bash
jcmd <pid> VM.native_memory summary
```

```text
Total: reserved=5234567KB, committed=4123456KB
-                 Heap: reserved=4194304KB (4096MB), committed=4123648KB   ← -Xms/-Xmx
-                Thread: reserved=204800KB, committed=204800KB             ← 线程栈 ×N（-Xss）
-                 Class: reserved=65536KB, committed=65120KB               ← 类元数据/Metaspace 块
-                  Code: reserved=249856KB, committed=98304KB              ← JIT CodeCache（含分段）
-                   GC: reserved=439808KB, committed=184320KB              ← 卡表/ Remembered set、标记位图
-             Compiler: reserved=22528KB, committed=22528KB                ← C1/C2 编译器自身
-               Internal: reserved=...                                     ← JVM 内部结构、Unsafe Arena 部分
-                 Symbol: reserved=...                                     ← 符号表/字符串表
-                  Other: reserved=...                                     ← 其他/分配器碎片
```

### 10.2.2 覆盖边界（NMT 不是什么）

> **覆盖边界**：NMT 只统计走 JVM 内存分配封装的路径。`Unsafe` 直接分配、第三方库（Netty PoolChunk 直配、压缩库、JNI 库）的 native 内存 **不被 NMT 完整追踪**——这类堆外要用 pmap/smaps 或库自带统计（Netty `PlatformDependent`、JFR）配合定位（见 [22_troubleshooting](22_troubleshooting.md) 堆外泄漏节）。DirectByteBuffer 的 `Unsafe.allocateMemory` 在部分 JDK 中计入 Internal，版本差异大，**不要把它当直接内存的可靠账单**。

## 10.3 baseline + diff 泄漏定位工作流

### 10.3.1 命令序列与增量输出

```bash
jcmd <pid> VM.native_memory baseline                    # 建立基线
# …… 运行一段时间 / 执行可疑操作 ……
jcmd <pid> VM.native_memory summary.diff                # 与基线的增量
```

```text
-                Thread: reserved=409600KB (diff= +204800KB, 200KB+  #200)   ← 线程从 0 涨到 200：线程泄漏实锤
-                 Class: reserved=... (diff= +30720KB)                        ← 类元数据增长：ClassLoader 泄漏
-               Internal: reserved=... (diff= +51200KB)                       ← 继续查：detail 模式看调用点
```

### 10.3.2 判读剧本与 detail 调用点

| diff 增长区域 | 嫌疑 | 动作 |
|---------------|------|------|
| Thread | 线程池不回收/每请求建线程 | jstack 数线程、池名统计（见 22 第 6 节） |
| Class | 动态类生成、ClassLoader 泄漏 | `GC.class_histogram` + 类加载器直方图 |
| Code | CodeCache 涨到 Reserved | `jcmd Compiler.codecache`，JIT 热点过多 |
| GC/Internal/Symbol | 堆结构随堆扩容、符号膨胀 | 对照 -Xmx 是否生效；重启前抓 detail |
| 全区域平稳但 RSS 上涨 | NMT 覆盖不到的 native 分配 | pmap -x / smaps + `detail` 的调用点栈 |

```bash
# detail 模式：定位到具体 malloc 调用点（配合 jhsdb/smaps 交叉验证）
jcmd <pid> VM.native_memory detail.diff | grep -A 8 "Total Malloc"
```

# 11. 生产环境安全建议

| 动作 | 风险 | 安全做法 |
|------|------|----------|
| `jmap -dump:live` / `GC.class_histogram`（默认） | 触发 Full GC → STW | 低峰执行；或 `jcmd <pid> GC.class_histogram -all` 不 GC 但含垃圾 |
| 大堆 dump | STW 数十秒 + 磁盘写满 | 预配 `HeapDumpOnOutOfMemoryError`；dump 盘独立 |
| `jstack -F` | 可能让 JVM 更僵 | 先常规 jstack 多次采样；-F 只作最后手段 |
| NMT detail | 5%~10% 额外内存 | 灰度实例开启 |
| 任何 attach 工具 | 与目标同用户，否则失败；attach 本身短暂 safepoint | 容器内用 `jcmd 1`；sidecar 共享 PID namespace |
| 兜底参数组（上线前配好，故障时就不手忙脚乱） | — | `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dumps -Xlog:gc*:file=gc.log:time -XX:NativeMemoryTracking=summary` |
