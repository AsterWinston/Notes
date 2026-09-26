
# 1. 故障排查全景流程

```
发现问题（监控告警 / 用户反馈）
    │
    ▼
快速判断故障类型
    │
    ├── CPU 飙高 ──────▶ 见 22.2
    ├── 内存泄漏/OOM ──▶ 见 22.3
    ├── GC 频繁/停顿 ──▶ 见 22.4
    ├── 线程死锁 ──────▶ 见 22.5
    ├── 线程过多 ──────▶ 见 22.6
    ├── 响应变慢 ──────▶ 见 22.7
    └── 直接内存溢出 ──▶ 见 22.8
```

# 2. CPU 飙高排查

## 2.1 完整排查链路

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

## 2.2 常见 CPU 飙高原因

| 原因 | 特征 | 定位 |
|------|------|------|
| 死循环 | 线程一直 RUNNABLE | jstack 看线程一直在哪个方法 |
| 大循环/复杂计算 | 线程 RUNNABLE | arthas trace 看耗时 |
| 频繁 GC | GC 线程占比高 | jstat -gc 查看 GC 频率 |
| 大量正则匹配 | 回溯导致 CPU 高 | jstack 定位正则相关方法 |
| JSON 大对象序列化 | 序列化/反序列化 CPU 高 | arthas trace |
| 上下文切换多 | vmstat cs 值高 | 线程数过多导致 |

## 2.3 Arthas 快速定位

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

# 3. 内存泄漏 / OOM 排查

## 3.1 堆内存溢出（Java heap space）

```java
// OOM 错误信息
// java.lang.OutOfMemoryError: Java heap space
```

**排查步骤：**

```bash
# 1. 确认是堆溢出还是非堆
# 堆溢出: Java heap space
# 元空间: Metaspace
# 直接内存: Direct buffer memory

# 2. 生成 heap dump（自动）
# 在启动参数中配置：
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/heapdump.hprof

# 3. 手动导出 heap dump
jcmd <pid> GC.heap_dump /data/logs/heapdump.hprof
# 注意：生产环境谨慎使用 live 选项（会 Full GC）

# 4. 使用 MAT 分析
# - 打开 hprof 文件
# - Leak Suspects Report
# - 查看 Dominator Tree
# - 追踪 GC Roots 路径

# 5. 使用 jmap 快速查看 Top 类
jmap -histo <pid> | head -20
```

## 3.2 内存泄漏的典型特征

```
jstat -gc <pid> 查看：
- FGC 次数持续增长
- OU (Old Used) 只增不减
- 每次 GC 后 OU 不下降（或下降很少）
```

**MAT 分析内存泄漏 —— 关键技巧：**

```
1. Histogram → 按 Retained Heap 排序
2. 找到可疑对象的 Dominator Tree
3. Path to GC Roots → exclude soft/weak/phantom references
4. 追踪到最终持有的代码
```

**常见内存泄漏场景：**

| 场景 | 原因 | 解决 |
|------|------|------|
| ThreadLocal 未清理 | 线程池中 ThreadLocal 值未 remove | finally 块中 remove() |
| 静态集合无限增长 | static List/Map 只增不减 | 使用 WeakHashMap 或定期清理 |
| 未关闭的资源 | InputStream/Connection 未关闭 | try-with-resources |
| 监听器未移除 | addListener 后未 removeListener | 成对使用 |
| 长生命周期对象持有短生命周期对象 | 缓存策略不对 | 使用弱引用/软引用 |
| CGLIB 动态代理无限生成类 | 类不断增加 | 限制 CGLIB 缓存或切换为 JDK Proxy |
| JSP 编译类过多 | 热加载过多 | 定期重启或限制编译 |

## 3.3 ThreadLocal 内存泄漏

```java
// 问题代码
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

**为什么 ThreadLocal 会泄漏：**

```
Thread
  └── ThreadLocalMap
        └── Entry (WeakReference) ← Key (ThreadLocal 对象) 是弱引用
             └── Value ← 强引用！Thread 不回收 → Value 不回收
```

## 3.4 Heap dump 生成方式对比

| 方式 | 说明 | 是否 Full GC |
|------|------|-------------|
| `jmap -dump:live,format=b,file=dump.hprof <pid>` | 先 GC 再 dump | 是 |
| `jmap -dump:format=b,file=dump.hprof <pid>` | 直接 dump（包含死对象） | 否 |
| `jcmd <pid> GC.heap_dump dump.hprof` | jcmd 方式 | 否（推荐） |
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时自动 dump | 否（OOM 发生时） |

# 4. GC 频繁 / 停顿过长

## 4.1 排查流程

```bash
# 1. 查看 GC 实时统计
jstat -gcutil <pid> 1000

# 关注：
# - YGC 增速（每秒几次 Young GC→频繁）
# - FGC 持续 > 0（Full GC 问题严重）
# - FGCT 单次时间（> 1 秒 → GC 停顿问题）

# 2. 查看 GC 日志
-Xlog:gc*=info:file=gc.log

# 3. 分析 GC 日志
# GCEasy 上传分析

# 4. 排查对象分配速率（Arthas）
vmtool --action getInstances --className java.lang.Object --limit 10
```

**解决方案：**
```bash
# Young GC 频繁 → 增大新生代
-Xmn2g

# Full GC 频繁 → 增大堆或老年代
-Xms8g -Xmx8g

# GC 停顿长 → 换成低延迟收集器
-XX:+UseG1GC -XX:MaxGCPauseMillis=100
# 或 ZGC
-XX:+UseZGC

# 大对象频繁 → 检查代码是否大数组/大集合
-XX:PretenureSizeThreshold=10m
```

# 5. 线程死锁

## 5.1 检测死锁

```bash
# jstack 检测
jstack <pid> | grep -A 50 "Found one Java-level deadlock"

# jcmd 检测
jcmd <pid> Thread.print | grep -A 50 "deadlock"

# VisualVM / JConsole → Thread 标签 → "Detect Deadlock"
```

**输出示例：**
```
Found one Java-level deadlock:
============================
"Thread-1":
  waiting to lock monitor 0x00007f... (object 0x0000000...)
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f... (object 0x0000000...)
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
    at com.example.ServiceA.methodA(ServiceA.java:10)
    - waiting to lock <0x0000000...> (a com.example.LockB)
    - locked <0x0000000...> (a com.example.LockA)
"Thread-2":
    at com.example.ServiceB.methodB(ServiceB.java:15)
    - waiting to lock <0x0000000...> (a com.example.LockA)
    - locked <0x0000000...> (a com.example.LockB)
```

## 5.2 死锁模式

```java
// 典型的死锁代码
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

**解决方案：**
1. **固定加锁顺序**：所有线程按相同顺序获取锁。
2. **使用 tryLock 超时**：`ReentrantLock.tryLock(timeout, unit)`。
3. **避免嵌套锁**：尽量不持有锁时去获取另一个锁。

# 6. 线程过多 / Unable to create native thread

## 6.1 现象

```
java.lang.OutOfMemoryError: unable to create new native thread
```

## 6.2 排查

```bash
# 查看当前线程数
jstack <pid> | grep "Thread" | wc -l

# 查看系统线程限制
ulimit -u

# 查看系统内存
free -h
```

## 6.3 原因和解决

| 原因 | 解决方案 |
|------|----------|
| 线程数太多（> 系统上限） | 使用线程池限制线程数 |
| 每个线程栈太大（Xss） | 减小 -Xss（如 -Xss256k） |
| 内存不足（线程栈占用物理内存） | 增加物理内存或减少 -Xmx |
| 线程泄漏（创建后不回收） | 排查线程池是否正确销毁 |

```bash
# 常见配置
-Xss256k    # 减小栈大小，允许创建更多线程
```

## 6.4 线程池泄漏排查

```bash
# 用 jstack 找到大量线程池命名的线程
jstack <pid> | grep "pool-" | wc -l

# 如果数量持续增长 → 线程池泄漏
# 检查代码：是否每次都 new ThreadPoolExecutor 且未 shutdown
```

# 7. 服务响应变慢

## 7.1 排查维度

| 可能原因 | 排查工具 | 关键指标 |
|----------|----------|----------|
| GC 停顿 | jstat / GC 日志 | FGC > 0, GC 停顿 > 1s |
| 线程 BLOCKED | jstack / arthas thread | 大量 BLOCKED 线程 |
| 线程池满 | jstack | 大量 WAITING (parking) 在队列取任务 |
| CPU 高 | top / arthas | CPU > 80% |
| 外部依赖慢 | arthas trace | 数据库/Redis/HTTP 调用耗时 |
| 连接池耗尽 | jstack | 大量线程等待获取连接 |

**Arthas 快速定位慢方法：**
```bash
# 监控方法耗时
monitor com.example.controller.* -c 10

# 追踪链路
trace com.example.controller.OrderController getOrder '#cost > 500'
```

# 8. 其他 OOM 场景

## 8.1 元空间溢出
```
java.lang.OutOfMemoryError: Metaspace
```
- 原因：动态生成类过多（CGLIB、动态代理、Groovy、JSP）。
- 解决：增大 `-XX:MaxMetaspaceSize` 或排查类生成。

## 8.2 直接内存溢出
```
java.lang.OutOfMemoryError: Direct buffer memory
```
- 原因：NIO 的 `ByteBuffer.allocateDirect()` 未释放。
- 排查：jcmd VM.native_memory summary。
- 解决：限制 `-XX:MaxDirectMemorySize`，排查代码是否正确释放。

## 8.3 GC Overhead 超限
```
java.lang.OutOfMemoryError: GC overhead limit exceeded
```
- 含义：超过 98% 的时间在 GC，但只回收了 <2% 的堆。
- 解决：分析堆转储、检查内存泄漏、增大堆。

```bash
# 关闭此限制（不推荐，但可紧急使用）
-XX:-UseGCOverheadLimit
```

## 8.4 无法创建线程
```
java.lang.OutOfMemoryError: unable to create new native thread
```
- 含义：系统内存不足以为新线程分配栈空间。
- 解决：减小 `-Xss`、限制线程数、排查线程泄漏。

# 9. 生产环境排查速查卡

```bash
# 快速检查清单
# 1. 查看进程基本信息
jps -lv

# 2. 查看 GC 状态（重点关注 FGC 和 GCT）
jstat -gcutil <pid> 1000 5

# 3. 查看 CPU 高的线程
top -Hp <pid>; jstack <pid> | grep -A 20 "nid=0x<hex>"

# 4. 查看内存分布 Top 10
jmap -histo <pid> | head -10

# 5. 查看 BLOCKED 线程
jstack <pid> | grep -A 20 BLOCKED

# 6. 查看死锁
jstack <pid> | grep -A 30 "deadlock"

# 7. 导出堆转储
jcmd <pid> GC.heap_dump /tmp/dump.hprof

# 8. 线程总数
jstack <pid> | grep "^\"" | wc -l

# 9. Arthas 在线诊断
curl -O arthas-boot.jar && java -jar arthas-boot.jar
dashboard
thread -n 5
```

# 10. 必备 JVM 参数（生产环境）

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
-XX:HeapDumpPath=/data/logs/         # dump 路径
-XX:ErrorFile=/data/logs/hs_err_pid%p.log  # 崩溃日志
-XX:+ExitOnOutOfMemoryError          # OOM 时退出（让 k8s 重启）
-XX:+PrintCommandLineFlags           # 打印 JVM 参数

# 元空间
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m

# 安全
-XX:+DisableExplicitGC               # 禁用 System.gc()
```
