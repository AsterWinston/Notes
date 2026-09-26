
# 1. JConsole

JDK 自带的图形化监控工具，适合快速查看 JVM 基本状态。

```bash
# 启动 JConsole
jconsole
# 或连接远程进程（需要启用 JMX）
jconsole <host>:<port>
```

**主要功能：**
- 堆内存使用曲线（实时）
- 线程数量和状态（支持死锁检测）
- 类加载数量
- MBean 管理（可远程调用操作）
- 手动 GC
- CPU 使用率概览

**启用 JMX 远程：**
```bash
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.ssl=false \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -jar app.jar
```

**局限**：功能简单，无历史数据，不适合深度分析。

# 2. VisualVM

- Oracle 开发，JDK 6-8 自带，后续需要单独下载。
- 比 JConsole 更强大，支持插件扩展。

```bash
# 启动
jvisualvm
```

**核心功能：**
| 功能 | 说明 |
|------|------|
| 本地/远程监控 | 自动发现本地 JVM 进程，支持 JMX 远程 |
| 内存实时曲线 | 堆、元空间、各个区域 |
| CPU 采样/Profiling | 找出 CPU 热点方法 |
| 内存采样/Profiling | 找出内存占用热点 |
| 线程分析 | 线程状态、死锁检测、时间线 |
| 堆转储分析 | 可直接 dump 并分析 |
| GC 可视化 | 将 GC 日志图形化显示 |
| 插件 | Btrace、MBeans、TDA 等 |

**VisualVM 插件推荐：**
- **Visual GC**：实时 GC 图形化（和 jstat 数据保持一致）
- **TDA**（Thread Dump Analyzer）：导入 thread dump 分析
- **BTrace**：动态字节码注入，不重启应用调试

## 2.1 Visual GC 插件解读

```
┌────────────────────────────────────────────────────────────┐
│  Visual GC 面板                                            │
│                                                            │
│  Eden ┌────────────────────────────┐ Survivor ┌──────┐     │
│       │################            │          │##    │     │
│       └────────────────────────────┘          └──────┘     │
│  Old  ┌──────────────────────────────────────────────┐    │
│       │##########################                    │    │
│       └──────────────────────────────────────────────┘    │
│  Perm/Metaspace ┌──────────────────────┐                   │
│                 │#######                │                   │
│                 └──────────────────────┘                   │
│  GC Time: 0.5%    Compile Time: 1.2%                      │
└────────────────────────────────────────────────────────────┘
```

# 3. JMC（Java Mission Control）+ JFR（Java Flight Recorder）

- **JFR**：JVM 内置的低开销数据采集框架（<2% 性能损耗）。
- **JMC**：分析 JFR 记录的可视化工具。

## 3.1 JDK 11 之前（JMC/JFR 是商业特性）

```bash
# 需要开启商业特性
-XX:+UnlockCommercialFeatures -XX:+FlightRecorder
```

## 3.2 JDK 11+（JFR 完全开源）

```bash
# 启动时开启 JFR
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

# 运行时动态开启 JFR（jcmd）
jcmd <pid> JFR.start duration=60s filename=recording.jfr
jcmd <pid> JFR.stop

# 持续采集
jcmd <pid> JFR.start name=myrecording settings=profile
```

## 3.3 JFR 能采集什么

| 类别 | 具体内容 |
|------|----------|
| GC | 每次 GC 类型、时长、回收量 |
| 内存 | 堆使用、分配速率、TLAB |
| 线程 | 线程创建/销毁、阻塞、锁竞争 |
| CPU | 方法采样、热点代码分析 |
| I/O | 文件 I/O、Socket I/O |
| 异常 | 异常种类、频次、堆栈 |
| 类加载 | 加载/卸载的类 |
| 系统 | CPU 负载、内存、环境变量 |

## 3.4 JMC 分析

启动 JMC：
```bash
jmc
# 打开 recording.jfr 文件进行分析
```

关键视图：
- **线程页面**：查看线程哪些在竞争锁、哪些在等待 I/O
- **内存页面**：GC 暂停分布、晋升大小
- **代码页面**：热点方法排序、内联情况
- **事件页面**：所有 JFR 事件的时间线

# 4. Arthas（阿里开源）

**生产环境诊断利器**，无需修改代码、无需重启应用。

## 4.1 安装和启动

```bash
# 下载
curl -O https://arthas.aliyun.com/arthas-boot.jar

# 启动并选择进程
java -jar arthas-boot.jar
```

## 4.2 核心命令

**1. dashboard — 仪表盘**
```bash
dashboard
# 实时显示：线程列表、CPU、内存、GC、运行环境
# 按 Ctrl+C 退出，类似 htop
```

**2. thread — 线程诊断**
```bash
thread                        # 所有线程信息
thread -n 3                   # 最忙的前 3 个线程
thread -b                     # 检出死锁线程
thread <id>                   # 查看指定线程的堆栈
thread --state BLOCKED        # 查看指定状态的线程
```

**3. jad — 反编译**
```bash
# 反编译查看源码（确认线上版本）
jad com.example.MyClass

# 反编译指定方法
jad com.example.MyClass doSomething
```

**4. monitor — 方法调用监控**
```bash
# 监控方法调用次数、耗时、失败率
monitor com.example.MyService doSomething -c 5
# -c 5：统计周期 5 秒
```

**5. watch — 方法执行数据观测**
```bash
# 查看方法入参、返回值、异常
watch com.example.MyService getUser '{params, returnObj, throwExp}' -x 4
# -x 4：展开深度 4 层
```

**6. trace — 方法内部调用路径**
```bash
# 查看方法的调用链路和耗时
trace com.example.MyController getOrder '#cost > 100'
# 过滤：只显示耗时 > 100ms 的调用
```

**7. stack — 方法调用堆栈**
```bash
# 查看方法被调用的路径
stack com.example.MyService doSomething
```

**8. tt — 时空隧道**
```bash
# 记录方法调用，可回放
tt -t com.example.UserDao findById
# 记录每次调用的参数、返回值、耗时
tt -l            # 列出记录
tt -i <index>    # 查看详情
tt --replay      # 回放某次调用
```

**9. profiler — 火焰图**
```bash
profiler start              # 开始采样
profiler stop               # 停止并生成火焰图
profiler start --event alloc # 内存分配火焰图
```

**10. vmtool — MBean 交互**
```bash
# 动态修改日志级别
vmtool --action getInstances \
       --className ch.qos.logback.classic.Logger \
       --express 'instances[0].setLevel(ch.qos.logback.classic.Level.toLevel("debug"))'
```

**11. heapdump**
```bash
heapdump /tmp/heapdump.hprof
# 导出堆转储，支持自动触发（OOM 等）
```

**12. logger**
```bash
logger                          # 查看所有 logger 信息
logger --name ROOT --level debug  # 动态修改日志级别
```

**13. redefine — 热替换**
```bash
# 修改 Java 文件后，编译为 .class，然后热替换（不重启）
redefine /path/to/MyClass.class
# 注意：不支持新增字段/方法签名变更
```

**14. ognl — 动态执行代码**
```bash
# 动态执行 OGNL 表达式
ognl '@java.lang.System@getProperty("java.version")'
ognl '@com.example.Config@value'
```

# 5. MAT（Memory Analyzer Tool）

Eclipse 出品，**堆转储分析**的最强工具。

```bash
# 下载并启动
# https://www.eclipse.org/mat/
```

## 5.1 核心分析功能

**Leak Suspects Report（泄漏嫌疑报告）：**
- 自动分析最大的内存消耗点，给出内存泄漏嫌疑。

**Histogram（直方图）：**
- 按类统计实例数和内存占用。
- 辅助找哪些类实例异常多。

**Dominator Tree（支配树）：**
- 找到占用最大内存的对象的依赖关系链。

**Path to GC Roots（GC Root 路径）：**
- 找出对象为什么不能被 GC 回收。
- 选择对象 → Path to GC Roots → exclude weak references

**OQL（对象查询语言）：**
```sql
SELECT * FROM com.example.User WHERE id = 123
SELECT * FROM java.lang.String WHERE toString() LIKE "hello%"
```

## 5.2 对比两个 dump
- "Compare Basket" 功能可以对比两个 heap dump，看哪些对象增长了。

## 5.3 MAT 常用技巧

```bash
# 启动时增大 MAT 自身内存
MemoryAnalyzer.exe -vmargs -Xmx8g
```

**排查思路**：
1. 打开 dump → Leak Suspects
2. 找到嫌疑对象 → Path to GC Roots
3. 排除弱引用 → 找到持有对象的引用链
4. 追踪到源码位置

# 6. GC 可视化分析工具

## 6.1 GCViewer
```bash
java -jar gcviewer.jar gc.log gc-summary.csv
```
- 生成 GC 停顿分布图、吞吐量趋势。

## 6.2 GCEasy
- 在线工具：https://gceasy.io/
- 上传 GC 日志，自动生成报告。

## 6.3 GCPlot
- 另一款在线 GC 日志分析工具。

# 7. 其他工具

| 工具 | 用途 |
|------|------|
| JProfiler | 商业性能分析工具 |
| YourKit | 商业性能分析工具 |
| async-profiler | 低开销采样分析器，生成火焰图 |
| PerfMa | 阿里的 JVM 参数调优平台 |
| Eclipse Memory Analyzer | Heap dump 分析 |
| BTrace | 安全的生产环境字节码注入诊断 |

# 8. 工具选择建议

| 场景 | 推荐工具 |
|------|----------|
| 本地快速查看 | JConsole / VisualVM |
| 生产环境在线诊断 | Arthas |
| 实时性能监控 | JFR + JMC |
| 堆转储分析 | MAT |
| GC 日志分析 | GCViewer / GCEasy |
| CPU/内存火焰图 | Arthas profiler / async-profiler |
| HotPatch 热修复 | Arthas redefine / BTrace |
| 长期性能追踪 | JFR 持续采集 + JMC 分析 |
