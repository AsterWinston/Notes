
# 1. 工具全景图

```
监控诊断工具家族：

jps   → 查看 Java 进程 PID
jstat → 监控 JVM 统计信息（GC、类加载、编译等）
jmap  → 查看内存、导出 heap dump
jstack → 查看线程堆栈
jinfo → 查看/修改 JVM 参数
jcmd  → 多功能综合命令（JDK 7+）
jhat  → 分析 heap dump（已废弃，推荐 MAT/JProfiler）
```

# 2. jps — 查看 Java 进程

```bash
# 普通模式：只显示进程 ID 和主类名
jps
# 12345 Jps
# 67890 MyApplication

# -l: 显示完整类名
jps -l
# 67890 com.example.MyApplication

# -v: 显示 JVM 启动参数
jps -v
# 67890 MyApplication -Xms4g -Xmx4g -XX:+UseG1GC

# -m: 显示传递给 main() 方法的参数
jps -m
# 67890 MyApplication --spring.profiles.active=prod

# 组合使用
jps -lv
```

# 3. jstat — JVM 统计监控

## 3.1 GC 相关统计

```bash
# 查看 GC 统计（最常用）
jstat -gc <pid> 1000 10    # 每秒输出一次，输出 10 次

# 输出列：S0C  S1C  S0U  S1U   EC    EU    OC     OU     MC    MU
#         ... YGC YGCT FGC FGCT GCT

# S0C/S1C  = Survivor 0/1 容量 (Capacity)
# S0U/S1U  = Survivor 0/1 使用量 (Used)
# EC/EU    = Eden 容量/使用量
# OC/OU    = Old 容量/使用量
# MC/MU    = Metaspace 容量/使用量
# YGC      = Young GC 次数
# YGCT     = Young GC 总时间
# FGC      = Full GC 次数
# FGCT     = Full GC 总时间
# GCT      = 总 GC 时间
```

```bash
# 查看 GC 使用率百分比
jstat -gcutil <pid> 1000

# 输出：S0  S1  E   O   M   CCS  YGC YGCT FGC FGCT GCT
#       0.0 12.3 45.6 67.8 78.9 80.1 100 5.23 2  1.23 6.46
```

## 3.2 其他 jstat 选项

```bash
# -class: 类加载统计
jstat -class <pid>
# Loaded  Bytes  Unloaded  Bytes  Time

# -compiler: JIT 编译统计
jstat -compiler <pid>
# Compiled  Failed  Invalid  Time  FailedType  FailedMethod

# -printcompilation: 最近编译的方法
jstat -printcompilation <pid>
# Compiled  Size  Type  Method
#    1234    56    1    java/lang/String::hashCode
```

# 4. jmap — 内存映像工具

## 4.1 查看堆摘要

```bash
# 堆配置和使用情况
jmap -heap <pid>

# 输出示例：
# Heap Configuration:
#    MinHeapFreeRatio = 0
#    MaxHeapFreeRatio = 100
#    MaxHeapSize = 4294967296 (4096.0MB)
#    NewSize = 1431699456 (1365.0MB)
#    ...
# Heap Usage:
# PS Young Generation
#    Eden Space:
#       capacity = 472907776 (451.0MB)
#       used = 234881024 (224.0MB)
#       free = 238026752 (227.0MB)
#       49.7% used
#    ...
```

## 4.2 查看对象直方图

```bash
# 查看所有对象统计（含死对象）
jmap -histo <pid>

# 只看存活对象（会触发 Full GC）
jmap -histo:live <pid>

# 输出示例：
# num   #instances   #bytes   class name
# 1:    123456       98765432  [B
# 2:    654321       52425680  java.lang.String
# 3:    12345        12345600  [C
# 4:    6789         5432100   java.util.HashMap$Node
```

解读：
- `[B` = byte[]（最常见，通常是 I/O 缓冲或序列化数据）
- `[C` = char[]
- 如果某个类实例数异常多 → 可能内存泄漏

## 4.3 导出堆转储

```bash
# 导出 heap dump（推荐方式）
jmap -dump:format=b,file=/path/to/heapdump.hprof <pid>

# 导出前先做 Full GC（去掉不可达对象）
jmap -dump:live,format=b,file=heapdump.hprof <pid>
```

**生产环境注意**：
- `jmap -dump:live` 会触发 Full GC，可能导致长时间停顿。
- 推荐使用 `-XX:+HeapDumpOnOutOfMemoryError` 自动 dump。
- 或者使用 `jcmd GC.heap_dump`（更安全）。

# 5. jstack — 线程堆栈

```bash
# 查看线程堆栈
jstack <pid>

# 检测死锁
jstack -l <pid>

# 输出关键信息：
# "main" #1 prio=5 os_prio=0 tid=0x00007f1234001000 nid=0x1234
#    Runnable [0x00007f1234567000]
#    java.lang.Thread.State: RUNNABLE
#         at com.example.MyClass.doSomething(MyClass.java:42)
#         at com.example.MyClass.main(MyClass.java:12)
#
# "Thread-1" #12 prio=5 tid=0x00007f1234123000 nid=0x5678
#    Waiting on condition [0x00007f1234556000]
#    java.lang.Thread.State: WAITING (parking)
#         at sun.misc.Unsafe.park(Native Method)
#         ...
```

**线程状态解读：**
| 状态 | 含义 |
|------|------|
| RUNNABLE | 正在运行或等待 CPU |
| BLOCKED | 等待获取锁（synchronized） |
| WAITING | 无限期等待（wait/join/park） |
| TIMED_WAITING | 有限等待（sleep/wait(timeout)/parkNanos） |
| TERMINATED | 线程已结束 |

**实际排查用法：**
```bash
# 快速统计线程状态
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c

# 查找 BLOCKED 状态的线程
jstack <pid> | grep -A 20 "BLOCKED"

# 查看占用最多 CPU 的线程
# 1. top -Hp <pid> 找到高 CPU 线程的 nid（十进制）
# 2. 转成十六进制
# 3. jstack <pid> | grep nid -A 10
```

# 6. jinfo — 查看和修改 JVM 参数

```bash
# 查看所有 JVM 参数
jinfo <pid>

# 查看某个参数值
jinfo -flag MaxHeapSize <pid>
# -XX:MaxHeapSize=4294967296

# 查看所有系统属性和 VM 参数
jinfo -sysprops <pid>
jinfo -flags <pid>

# 运行时修改参数（需要参数支持动态修改）
jinfo -flag +PrintGC <pid>      # 开启
jinfo -flag -PrintGC <pid>      # 关闭
jinfo -flag MaxHeapFreeRatio=70 <pid>  # 修改值
```

# 7. jcmd — 综合命令（JDK 7+）

```bash
# 列出所有 Java 进程和 jcmd 支持的命令
jcmd

# 列出某个进程支持的命令
jcmd <pid> help

# 常用命令：
jcmd <pid> GC.heap_dump /path/to/dump.hprof   # 导出堆转储（比 jmap 更安全）
jcmd <pid> GC.heap_info                       # 堆使用信息
jcmd <pid> GC.class_histogram                # 类直方图
jcmd <pid> Thread.print                      # 线程堆栈（= jstack）
jcmd <pid> VM.uptime                         # JVM 运行时间
jcmd <pid> VM.flags                          # JVM 参数
jcmd <pid> VM.system_properties             # 系统属性
jcmd <pid> VM.command_line                   # 启动命令
jcmd <pid> GC.run                            # 触发 GC
jcmd <pid> VM.version                        # JVM 版本
```

```bash
# 常用排查组合
jcmd <pid> Thread.print > thread.txt       # 导出线程堆栈
jcmd <pid> GC.heap_dump heapdump.hprof     # 导出堆转储
jcmd <pid> GC.class_histogram | head -30   # Top 30 类
jcmd <pid> VM.uptime                        # 查看运行时间
```

# 8. 其他常用命令

## 8.1 vmstat — 上下文切换

```bash
vmstat 1
# r  b   swpd   free   buff  cache   si   so    bi   bo   in   cs us sy id wa st
# cs (context switches) 过高 → 可能大量线程竞争
```

## 8.2 pidstat — 线程详细统计

```bash
pidstat -p <pid> -t 1
# 查看进程内各线程的 CPU 使用
```

## 8.3 获取线程 CPU 使用详情

```bash
# 1. 找到 CPU 最高的 Java 进程
top -c

# 2. 找到进程中最耗 CPU 的线程
top -Hp <pid>
# 输出的 PID 是十进制

# 3. 十进制转十六进制
printf "%x\n" <tid>

# 4. 在 thread dump 中搜索该 nid
jstack <pid> | grep -A 30 "nid=0x<hex>"
```

# 9. 工具选择速查表

| 排查目标 | 工具 | 常用命令 |
|----------|------|----------|
| 进程状态 | jps | `jps -lv` |
| GC 实时监控 | jstat | `jstat -gcutil <pid> 1000` |
| 内存分布 | jmap | `jmap -histo <pid> \| head -20` |
| Heap Dump | jmap/jcmd | `jcmd <pid> GC.heap_dump dump.hprof` |
| 线程状态 | jstack | `jstack <pid> \| grep -A 20 BLOCKED` |
| JVM 参数 | jinfo | `jinfo -flags <pid>` |
| CPU 高 | top + jstack | 结合线程 nid 定位 |
| 综合诊断 | jcmd | `jcmd <pid> help` |

# 10. 生产环境安全建议

- `jmap -dump:live` 会触发 Full GC，生产慎用。
- `jstack -F` 强制模式可能导致 JVM 短暂挂起。
- 大堆 dump 时 JVM 可能停顿，提前规划窗口。
- 优先使用 `jcmd` 替代 `jmap`（更安全）。
- 配置 `-XX:+HeapDumpOnOutOfMemoryError` 自动 dump。
