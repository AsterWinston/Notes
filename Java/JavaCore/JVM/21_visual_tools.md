# 1. JConsole

JDK 自带的图形化监控工具，适合快速查看 JVM 基本状态。

## 1.1 连接方式

### 1.1.1 三种连接途径

| 方式 | 操作 | 前提 |
|------|------|------|
| 本地 attach | `jconsole` → 选进程 | 与目标 JVM 同用户；读 hsperfdata |
| 本地 JMX 端口 | `jconsole host:port` | 启动带 jmxremote 参数 |
| 远程 JMX | 同上 + 密码文件 | 网络可达 + SSL/认证配置 |

### 1.1.2 远程 JMX 参数模板

```bash
# 启用 JMX 远程（生产要么不开，开必配认证）
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.ssl=false \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -jar app.jar
# 认证模式：-Dcom.sun.management.jmxremote.password.file / access.file 指定账号与权限文件（权限 600）
# JDK 默认 -Dcom.sun.management.jmxremote.local.only=true，本地 attach 不外放端口
```

## 1.2 六大标签页速查

| Tab | 看什么 | 关键细节 |
|-----|--------|----------|
| Summary | CPU 曲线、堆内存曲线、线程总数、类加载总数、VM 参数与系统属性 | 4 个小图叠在同一页，"体检第一眼" |
| Memory | 下拉框切换 **Eden / Survivor / Old / Perm(Metaspace) / 堆整体 / 非堆** | "used 锯齿 + GC 竖线"：Full GC 后 used 不回落 → 泄漏嫌疑 |
| Threads | 线程数曲线 + 线程列表 + **Detect Deadlock 按钮** | 点"Detect Deadlock"直接给出死锁栈（同 `jstack -l` 后端） |
| Classes | 已加载/卸载类计数 | Loaded 单调涨 + 无卸载 → ClassLoader 泄漏 |
| VM Statistics | 各内存池 used/committed 表格 | 与 `jcmd GC.heap_info` 同源 |
| MBeans | 平台 MBean（Memory/GarbageCollector/Threading/DiagnosticCommand） | 可远程调操作；`HotSpotDiagnostic` 里能 dumpHeap、dumpAllThreads |

## 1.3 适用场景与局限

- 适合：本地开发、功能演示、"这台机器 GC 大概什么样"的 5 分钟巡检。
- 局限：**采样轮询有延迟**（JMX MBean 读数，默认最快 1~2s）；无历史持久化；没有 dump 深度分析（点开 dumpHeap 也只是触发导出）；无火焰图、无方法级热点。→ 深度分析路线见 [22_troubleshooting](22_troubleshooting.md)。

# 2. VisualVM

- Oracle 开发，JDK 6-8 自带（`jvisualvm`），**JDK 9 起移除、独立下载**（github/oracle 版继续演进，插件兼容 JDK 21）。

## 2.1 核心功能

| 功能 | 说明 |
|------|------|
| 本地/远程监控 | 自动发现本地 JVM 进程，支持 JMX 远程 |
| 内存实时曲线 | 堆、元空间、各个区域；点 GC 按钮手动触发 |
| CPU 采样/Profiling | 找出 CPU 热点方法（采样模式开销可接受，instrument 模式很重） |
| 内存采样/Profiling | 找出内存占用热点（哪个方法 new 了什么） |
| 线程分析 | 线程状态、死锁检测、时间线（带颜色轨道） |
| 堆转储分析 | 可直接 dump 并做基础分析（深度分析交给 MAT） |
| GC 可视化 | 将 GC 日志图形化显示 |
| 插件 | Visual GC、TDA、Btrace、MBeans 等 |

## 2.2 插件推荐

- **Visual GC**：实时 GC 图形化（和 jstat 数据保持一致，本质读 HotGarbageCollector MBean）。
- **TDA**（Thread Dump Analyzer）：导入 thread dump 分析线程状态/锁等待图。
- **BTrace**：动态字节码注入，不重启应用调试（后被 Arthas 生态大量替代）。

## 2.3 Visual GC 面板逐区解读

### 2.3.1 面板结构（八区）

```text
┌──────────────────────────────────────────────────────────────┐
│  Memory（各区占用，自左而右）                                  │
│  Eden ┌────────────────────────┐ Survivor0 ┌──┐ Surv1 ┌──┐  │
│       │################# 95%   │           │##│       │  │  │ ← Eden 填满即 YGC
│       └────────────────────────┘           └──┘       └──┘  │
│  Old  ┌──────────────────────────────────────────────┐       │
│       │###################### 68%                    │       │ ← YGC 后台阶上跳=晋升
│       └──────────────────────────────────────────────┘       │
│  Metaspace ┌──────────────┐   CodeCache ┌────┐               │
│            │#######        │             │##   │              │
│            └──────────────┘              └────┘               │
├──────────────────────────────────────────────────────────────┤
│  GC（次数与耗时柱）                                            │
│  Young ┃┃┃┃┃┃   Full ┃   CMS ┃┃   Ref Proc ▁▂▁               │ ← 柱高=停顿时间!
├──────────────────────────────────────────────────────────────┤
│  Tenuring（每次 YGC 的晋升直方图 + Promotion 速率线）           │
│  age1▇ age2▅ age3▂ …   ──promotion rate──▶ 长期高位=老年代快涨 │
├──────────────────────────────────────────────────────────────┤
│  GC Ratio / GC Time（吞吐量百分比曲线）                        │
└──────────────────────────────────────────────────────────────┘
```

### 2.3.2 判读要点

| 视觉特征 | 含义 | 联动动作 |
|----------|------|----------|
| Eden 锯齿变密 | 分配率上升或新生代偏小 | 查分配热点（profiler alloc） |
| Survivor 常年近满 | 幸存者过多 → 提前晋升 | 调 SurvivorRatio / 查缓存膨胀 |
| Old 台阶持续上涨 | 晋升 > 回收 | Full GC 后仍涨 → dump |
| GC 柱普遍很高 | 单次停顿长 | 换收集器/缩堆，见 [14_gc_tuning](14_gc_tuning.md) |
| Promotion 线长期高位 | 对象长寿或大对象直入 | 检查批量任务、大集合 |

## 2.4 Sampler 与命令行工具的关系

- Sampler → CPU Sampling / Memory Allocation：方法级热点的"穷人版火焰图"。
- 数据通道与 JConsole/jstat 相同（JMX + attach），**粒度停在方法/区域级**；要行级/分配级下钻，用 async-profiler 或 JFR（见第 3 节与 [20_jvm_tools](20_jvm_tools.md)）。

# 3. JMC（Java Mission Control）+ JFR（Java Flight Recorder）

## 3.1 演进：从商业特性到生产标配

| 时间 | 状态 |
|------|------|
| JDK 8u40 左右 | JFR 引入但属 **商业特性**：`-XX:+UnlockCommercialFeatures -XX:+FlightRecorder` 才能开 |
| JDK 11 | **开源**（JEP 328），任意场景免费用，生产常开成为共识 |
| JDK 11+ | JMC 不再随 JDK 发布，改为**独立下载**（github.com/openjdk/jmc），插件化，支持容器/远程 |

## 3.2 录制配置

### 3.2.1 两种启动写法

```bash
# 一次性录制
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

# 生产常开：环形分块落盘，开销极低，出事时"往回看"
java -XX:StartFlightRecording=maxsize=512m,maxage=6h,dumponexit=true,name=main -jar app.jar
```

### 3.2.2 参数含义与开销

| 参数 | 含义 |
|------|------|
| `settings=default` | **开销 <1%**（官方口径），事件裁剪；生产基线 |
| `settings=profile` | 约 2%~3%：加上分配事件、CPU 采样间隔 10ms、更深栈 |
| `disk=true` + `maxsize` | 分块(chunk)写盘，写满按 LRU 淘汰旧 chunk → 环形缓冲 |
| `maxage=6h` | 保留最近 6 小时 |
| `dumponexit=true` | 退出时把当前录制固化成文件 |
| `path-to-gc-roots=true` | dump 时带 GC Roots（给 MAT 用） |

## 3.3 事件大类与代表性事件名

| 类别 | 事件（jdk.* 名称） | 用途 |
|------|--------------------|------|
| GC | `jdk.GarbageCollection`、各收集器细分、`jdk.GCPhasePause`、`jdk.AllocationRequiringGC` | 停顿分布、触发原因 |
| 对象分配 | `jdk.ObjectAllocationInNewTLAB`、`jdk.ObjectAllocationOutsideTLAB`（profile） | 谁在造垃圾、大对象 |
| 线程/锁 | `jdk.JavaMonitorEnter`（synchronized 争用）、`jdk.ThreadPark`、`jdk.VirtualThreadPinned`（JDK21） | 阻塞归因、虚拟线程 pinning |
| CPU/方法 | `jdk.ExecutionSample`、`jdk.NativeMethodSample` | 火焰图数据源 |
| 文件/网络 | `jdk.FileRead/FileWrite/SocketRead/SocketWrite`（阈值：如 >5ms 才记） | 慢 IO |
| 编译 | `jdk.CompilerStatistics`、`jdk.Compilation` | JIT 风暴、内联失败 |
| 类/元数据 | `jdk.ClassLoad`、`jdk.MetaspaceGCThreshold`、`jdk.MetaspaceSummary` | 类泄漏 |
| 虚拟机 | `jdk.CPULoad`、`jdk.ThreadStatistics`、`jdk.OSInformation`、`jdk.VirtualizationInformation` | 环境基线、容器识别 |

## 3.4 jcmd 运行中操作工作流

```bash
jcmd <pid> JFR.start name=rec settings=profile maxsize=256m   # 动态开始
jcmd <pid> JFR.check                                          # 看正在录的录制
jcmd <pid> JFR.dump name=rec filename=/data/rec_$(date +%H%M).jfr   # 截取当前内容成文件
jcmd <pid> JFR.stop name=rec
```

## 3.5 JMC 分析工作流

```text
recording.jfr ──JMC 打开──▶
  时间轴(所有事件叠在一条 Timeline) ──圈选故障窗口──▶
    Method Profiling(火焰图) / GC / Threads / Memory / System 各页面联动过滤 ──▶
      定位到方法/锁/分配点 ──▶ 改代码/调参
```

- 火焰图行号级精度受采样间隔与栈回溯影响；`profile` 模板 + 更深栈事件（`jdk.ExecutionSample` stack depth 可调）+ 诊断参数 `-XX:+DebugNonSafepoints` 可改善归因（收益视版本而定，别当刚需）。
- JMC 也支持**直接连接运行中的 JVM 实时录制/查看**（JMX 通道）。

## 3.6 生产常开建议

```bash
# 建议写进所有服务的启动参数（JDK 11+）
-XX:StartFlightRecording=maxsize=512m,maxage=6h,dumponexit=true,disk=true,settings=default
```

- 相当于给 JVM 装了**行车记录仪**：出问题时用 `JFR.dump` 取回故障前后窗口，无需"提前开工具"。
- 对比：jstat 只有"现在"，GC 日志只有 GC，JFR 是**全事件时间轴**。

# 4. Arthas（阿里开源）

**生产环境诊断利器**，无需修改代码、无需重启应用（外部开源工具，attach 后按需字节码增强）。

## 4.1 安装与连接

```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar                 # 列出 Java 进程，数字选择 attach
java -jar arthas-boot.jar --select myapp  # 按主类名自动挑进程
bash as.sh <pid>                          # 官方脚本入口（等价）
# telnet 端口默认 3658，http 端口默认 8563（内置 web console / 火焰图下载）
# 安全注意：生产务必 --username/--password 或防火墙限 127.0.0.1；用完 stop 卸载
```

## 4.2 先看一眼：dashboard 与 thread

```bash
dashboard             # 实时面板：线程 Top CPU、内存各池、GC 计数、运行时信息，类似 htop
thread                # 所有线程，按 CPU 排序
thread -n 3           # 最忙的前 3 个线程 + 完整栈
thread -b             # 找出"阻塞别人的线程"（锁持有者归因，含死锁）
thread --state BLOCKED
thread <id>           # 指定线程详情
```

## 4.3 方法四件套：watch / trace / stack / monitor

### 4.3.1 watch —— 现场取证（入参/返回值/异常）

```bash
watch com.example.MyService getUser '{params, returnObj, throwExp}' -x 4 -n 5
#   -x 4：对象展开 4 层；-n 5：命中 5 次后自动退出（防止刷屏+降开销）
#   条件表达（第 3 个参数位）：'params[0]==123'；只看异常：-e
```

### 4.3.2 trace —— 方法内部调用链 + 每段耗时（找慢点）

```bash
trace com.example.MyController getOrder '#cost > 100' -n 5
#   只显示总耗时 >100ms 的调用；输出树里每行 `+---[12.3ms] method()`
```

```text
trace 输出解读示例：
`---[156.3ms] com.example.OrderController:getOrder()
    +---[3.2ms]  com.example.AuthService:check()
    +---[140.1ms] com.example.OrderDao:selectById()   ← 大头在这（下游/DB）
    `---[9.5ms]   com.example.JsonUtils:serialize()
```

### 4.3.3 stack —— 反向问题：这个方法被谁调用？

```bash
stack com.example.MyService doSomething
```

### 4.3.4 monitor —— 聚合统计：次数/成功率/平均耗时

```bash
monitor com.example.MyService doSomething -c 5     # 统计周期 5 秒
```

## 4.4 tt — 时空隧道（录制回放）

```bash
tt -t com.example.UserDao findById      # 录制：每次调用的入参/返回值/异常/耗时/栈
tt -l                                   # 列出录像索引（INDEX 默认从 1000 起）
tt -i 1002                              # 查看索引 1002 的全部上下文
tt -i 1002 -w 'target.getDbUrl()'       # 对录像对象执行 OGNL（拿当时的 this/params）
tt -i 1002 -p                           # 用录像入参"重放"一次调用（重放参数各版本略有差异，以 `tt -h` 为准；幂等方法才敢！）
```

- 价值：**把"偶发一次的那次调用"钉住反复解剖**，包括用当时参数重放复现。
- 注意：重放=真实再执行一次业务方法，非幂等（写库、扣款）严禁 `-p`。

## 4.5 类问题三板斧：sc / sm / classloader

### 4.5.1 命令模板

```bash
sc *OrderService          # 搜索已加载类（支持通配）；-d 看细节：classloader(hash)、code-source(jar 路径)、模块
sc -d -f com.example.Config   # -f 连字段一起 dump
sm com.example.OrderService   # 方法签名清单
classloader -t            # 类加载器继承树（每行带 hash 与加载类/实例计数）
classloader -l            # 按加载器统计 loadedCount/unloadedCount/实例数
classloader -c 1b23c7d --load com.example.Foo   # 指定 loader 尝试加载：定位"该类到底谁能加载"
```

### 4.5.2 排 ClassNotFoundException / NoClassDefFoundError 思路

```text
NoClassDefFoundError 往往 = 类"找得到但静态初始化失败过"或 依赖的 B 类加载不到
  1. sc -d *Foo  → 确认在不在、来自哪个 jar（同名列多 jar → 排包）
  2. classloader -t → 看该 jar 是否被目标 ClassLoader 覆盖（父子委派没够着）
  3. classloader --load → 手工触发加载，第一次异常含根因
```

## 4.6 热更新流程：jad → mc → retransform

### 4.6.1 三步操作流程

```bash
# 1. 反编译确认线上代码现状（对齐版本，防止"以为修了"）
jad --source-only com.example.MyClass > /tmp/MyClass.java
# 2. 修改后，用 Arthas 的 classloader 上下文编译
sc -d com.example.MyClass | grep classLoaderHash    # 拿到 hash，如 1b23c7d
mc --classLoaderClass org.springframework...LaunchedURLClassLoader /tmp/MyClass.java -d /tmp
# 3. 热替换
retransform /tmp/com/example/MyClass.class
```

### 4.6.2 限制与审计风险

| 限制/风险 | 说明 |
|-----------|------|
| 不能加/删字段、方法，不能改签名/继承 | JVM redefine 语义决定 |
| **重启即失效** | 改动不落盘，必须补正式发版，否则"幽灵修复" |
| 与代码仓库不一致 | 事后必须登记审计；多人操作互相覆盖 |
| `redefine` 老命令 | 已被 `retransform` 取代（前者走旧 redefine API，易与增强冲突） |
| 生产慎用 | 应急止血通道，不是发布通道 |

## 4.7 火焰图：profiler 命令（集成 async-profiler）

```bash
profiler start                    # 默认 cpu 事件
profiler start --event alloc      # 内存分配火焰图
profiler start --event lock       # 锁争用火焰图
profiler stop --format html --file /tmp/f.html
# 浏览器打开：横向=时间占比(越宽越热)，纵向=调用深度；"宽平顶"即热点叶子方法
```

## 4.8 ognl 与 vmtool：隔空操作堆上的对象

```bash
ognl '@java.lang.System@getProperty("java.version")'       # 静态方法
ognl '@com.example.Config@DEBUG'                            # 读静态字段
ognl -c 1b23c7d '@com.example.MyClass@staticMethod()'       # 指定 classloader（应用类要用它的 loader）

vmtool --action getInstances --className java.util.concurrent.ThreadPoolExecutor \
       --express 'instances[0].getPoolSize()'                # 从堆里抓实例直接调方法

# 动态改日志级别（经典应急）
vmtool --action getInstances \
       --className ch.qos.logback.classic.Logger \
       --express 'instances[0].setLevel(ch.qos.logback.classic.Level.toLevel("debug"))'
logger --name ROOT --level debug                              # Arthas 自带 logger 命令（对 slf4j/logback 适配）

heapdump /tmp/heapdump.hprof                                  # = jcmd GC.heap_dump
```

## 4.9 开销控制与退出

```bash
options                              # 查看全部开关（结果条数/展开深度/大小限制等）
options unsafe true                  # 默认不增强 JDK 自身类；置 true 可跟踪 JDK 方法（开销大）
reset                                # 撤销当前所有字节码增强（watch/trace 退出后增强仍在！）
quit / Ctrl+C                        # 只退出客户端，Arthas 还驻留进程
stop / shutdown                      # 彻底卸载 Arthas 并还原增强 —— 诊断结束必须执行
```

> 纪律：`trace/watch` 带 `-n` 限量；增强过的类用 `reset` 复位；离开前 `stop`。详见官方文档场景案例与 [22_troubleshooting](22_troubleshooting.md)。

# 5. MAT（Memory Analyzer Tool）

Eclipse 出品，**堆转储分析**的最强工具。

```bash
# 下载并启动 https://www.eclipse.org/mat/
MemoryAnalyzer.exe -vmargs -Xmx8g        # dump 8G ≈ MAT 自身至少 8G+，内存按 dump 大小配
```

## 5.1 三视图速查

### 5.1.1 三个视图各自回答的问题

| 视图 | 回答的问题 | 关键点 |
|------|-----------|--------|
| Histogram | 每类多少实例/字节 | **Group by class vs group by package** 切换；排序选 *Retained Heap* 而不是 Shallow |
| Dominator Tree | 谁"实际上拥有"最多内存 | 默认视图首选，看**占用大头在哪个对象** |
| Leak Suspects | 系统替你猜了一遍 | 饼图 → 问题报告页 → 疑点详情 → 合并最短路径 |

### 5.1.2 浅堆 vs 深堆

| 概念 | 定义 | 用途 |
|------|------|------|
| Shallow Heap | 对象自身头+字段占用（不含引用目标） | String 对象本体仅几十字节 |
| Retained Heap | **释放该对象后可连带回收的总量** | 判断"谁真正吃内存"，看这个 |

## 5.2 Dominator Tree：支配树 ≠ 引用树

```text
引用图：                        支配树（MAT 实际展示）：
A ─▶ B ◀─┐                     A
A ─▶ C ─▶│D（多路径可达 D）      ├── B        ← D 被 B/C 共享
        E ─┘                   └── C           谁都不能单独释放它
                                    （D 挂在共同支配者 A 下，或由 B/C 分摊）
```

- 规则：删掉节点 X 导致 Y 必死 → X 支配 Y。支配树把"共享对象"上提到最近的公共支配者，避免重复计数。
- 右键链路：Dominator Tree 中可疑对象 → `Path to GC Roots → exclude weak/soft/phantom references` → 找到"为什么回收不掉"。
- `List objects → with incoming/outgoing references`：双向追引用链。

## 5.3 Leak Suspects 与 Top Components

- **Leak Suspects**：基于合并支配路径自动聚类出 "Problem Suspect 1/2/3 各占 xx%"，适合**拿到 dump 的第一站**。
- **Top Components**（报告页切到 "Leak Report → Top Components" 或单独跑）：按 package/classloader 聚合，适合**Tomcat/OSGi 多应用共存**定位"哪个应用/加载器泄漏"。
- 两者同源不同视角：Suspects 按对象、Components 按包/类加载器。

## 5.4 OQL 实用查询

```sql
SELECT * FROM com.example.User WHERE id = 123                -- 原始示例保留
SELECT * FROM java.lang.String WHERE toString() LIKE "hello%" -- 按内容搜串（LIKE 走正则匹配）
SELECT * FROM java.util.HashMap h WHERE h.size > 10000        -- 找超大 Map
SELECT s.value.length AS len, s AS object
  FROM java.lang.String s WHERE s.value.length > 10000        -- JDK17+ String 底层 byte[] 字段名随版本变（JDK8 是 char[] count/offset），列名报错就改字段
```

## 5.5 批量与索引

- 大 dump 首次打开会构建 `.index`（很慢）；**支持 batch 模式预解析**（`ParseHeapDump.sh`，可附带生成 suspects/top_components 报表）——适合"脚本先跑，人再看"的流水线。
- 对比两个 dump 的增长：Compare Basket（保留原能力），或对同一类做两次 `jmap -histo` 差值粗筛。

## 5.6 常见泄漏模式（拿到 MAT 先对号入座）

| 模式 | dump 中的表现 | 根治方向 |
|------|----------------|----------|
| static Map/List 无界缓存 | Dominator 顶部一个大 HashMap | WeakReference/LRU/外置缓存 |
| ClassLoader 泄漏 | Histogram 里 `ClassLoader`/`Class` 实例数递增，Top Components 多个同名应用包 | 见 [22_troubleshooting](22_troubleshooting.md) |
| ThreadLocal 未 remove | 大量 `Thread` → `ThreadLocalMap$Entry` value 存活 | finally remove |
| 监听器/回调注册未注销 | 对象 incoming refs 来自 listener 集合 | 成对注销 |
| Finalizer 队列堆积 | `java.lang.ref.Finalizer` 的 Retained 巨大 | 移除 finalize()，用 Cleaner/try-with-resources |

# 6. GC 可视化分析工具

| 工具 | 形态 | 特色 | 适用 |
|------|------|------|------|
| GCViewer | 本地 jar：`java -jar gcviewer.jar gc.log out.csv` | 吞吐/暂停分布图，老牌离线 | 单份 JDK8 格式日志 |
| GCEasy | 在线 https://gceasy.io/ | 自动报告 + **调优建议**（Tuning 页给出建议参数） | 快、不想装东西（注意日志可能含敏感信息，公司场景慎传） |
| GCPlot | 在线 https://gcplot.dev/（开源，本地可部署） | **多文件叠加对比**（调参前后 A/B）、**暂停时间热力图**、原生支持 JDK 9+ `-Xlog` 统一日志格式 | 版本对比、长时间序列观察 |

选型：一次性看结论 → GCEasy；要 A/B 对比参数改动效果 → GCPlot；日志敏感不能出门 → GCViewer/本地部署 GCPlot。

# 7. 其他工具

| 工具 | 用途 |
|------|------|
| JProfiler | 商业性能分析工具（IDE 集成强） |
| YourKit | 商业性能分析工具 |
| async-profiler | 低开销采样分析器，生成火焰图 |
| PerfMa | 阿里的 JVM 参数调优平台 |
| Eclipse Memory Analyzer | Heap dump 分析 |
| BTrace | 安全的生产环境字节码注入诊断 |

# 8. 场景工具选型

## 8.1 决策树

```text
                发现问题
                  │
        能停服务/本地可复现？──是──▶ VisualVM / JMC 本地 attach（随便采样）
                  │否
        只想"看一眼"总体状态？──是──▶ JConsole / Arthas dashboard / jstat -gcutil
                  │否，需要取证下钻
     ┌────────────┼─────────────────┬────────────────┐
   CPU 热点      内存/GC         响应慢归因        行为级观察
     │             │                │                │
async-profiler  dump 不可停？      JFR 时间轴     Arthas trace/watch/tt
火焰图 / JFR   是→ JFR 分配事件     对齐故障时刻     （按需增强，用完 reset）
             否→ jcmd GC.heap_dump + MAT
                  │
     长期趋势与告警 ──▶ 监控系统：JMX exporter + Prometheus 类方案（JFR 持续录制兜底现场）
     容器/K8s 环境 ──▶ 先读 [23_container_jvm](23_container_jvm.md)（attach 跨用户/namespace 坑更多）
```

## 8.2 速查表

| 场景 | 推荐工具 |
|------|----------|
| 本地快速查看 | JConsole / VisualVM |
| 生产环境在线诊断 | Arthas |
| 实时性能监控 | JFR + JMC |
| 堆转储分析 | MAT |
| GC 日志分析 | GCViewer / GCEasy / GCPlot |
| CPU/内存火焰图 | Arthas profiler / async-profiler |
| HotPatch 热修复 | Arthas retransform / BTrace |
| 长期性能追踪 | JFR 持续采集 + JMC 分析 |
