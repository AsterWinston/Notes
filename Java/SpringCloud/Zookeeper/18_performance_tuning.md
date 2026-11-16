
# 1. 性能调优概述

ZooKeeper 的性能调优涉及多个层面：JVM、操作系统、ZooKeeper 自身参数和集群架构。

```text
性能调优层次：
+----------------------------------------------------------------------+
|                                                                       |
|  层级    |  关注点                                  | 影响范围        |
|  -------|-----------------------------------------|---------------- |
|  JVM    |  GC 停顿、堆大小、垃圾回收器              | 所有操作        |
|  OS     |  磁盘 I/O、网络、文件句柄                | 所有操作        |
|  ZK参数 |  队列大小、超时、快照频率                | 特定操作        |
|  架构   |  集群规模、Observer 部署、读写分离        | 整体容量        |
|                                                                       |
+----------------------------------------------------------------------+
```

# 2. JVM 调优

## 2.1 GC 选择与调优

```bash
# ZooKeeper 3.4.x 推荐 CMS，3.5+ 推荐 G1GC

# G1GC 推荐参数
export JVMFLAGS="
  -Xms8g -Xmx8g                          # 堆固定大小，避免动态调整
  -XX:+UseG1GC                            # 使用 G1GC
  -XX:MaxGCPauseMillis=50                 # 目标最大停顿 50ms
  -XX:G1HeapRegionSize=16m                # Region 大小
  -XX:InitiatingHeapOccupancyPercent=35   # 触发并发标记的堆占比
  -XX:ConcGCThreads=4                     # 并发 GC 线程数
  -XX:ParallelGCThreads=8                 # 并行 GC 线程数
  -XX:+ParallelRefProcEnabled             # 并行处理引用
  -XX:+UnlockExperimentalVMOptions
  -XX:G1NewSizePercent=5                  # 新生代最小占比
  -XX:G1MaxNewSizePercent=60              # 新生代最大占比

  # GC 日志
  -Xlog:gc*:file=/var/log/zookeeper/gc.log:time,level,tags
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/zookeeper/
"
```

## 2.2 堆大小选择

```text
堆大小计算：
+----------------------------------------------------------------------+
|                                                                       |
|  堆大小 = 基础开销 + ZNode 数据总量 + Watch 数据 + Session 数据       |
|                                                                       |
|  基础开销：约 500MB ~ 1GB                                            |
|  ZNode 开销：每个 ZNode 约 150-300 字节（含 metadata）                |
|  Watch 开销：每个 Watch 约 100-200 字节                               |
|  Session 开销：每个 Session 约 1KB                                   |
|                                                                       |
|  示例计算：                                                           |
|    10 万 ZNode × 0.2KB ≈ 20MB                                        |
|    5 万 Watch × 0.15KB ≈ 7.5MB                                       |
|    1 万 Session × 1KB ≈ 10MB                                         |
|    基础开销 ≈ 800MB                                                   |
|    总堆 ≈ 800MB + 37.5MB ≈ 1GB                                       |
|    安全系数 2x → 2GB                                                 |
|                                                                       |
|  ⚠️ 重要：堆必须足够大，避免 GC 频繁和停顿超时                          |
|    syncLimit * tickTime 必须大于最大 GC 停顿时间                       |
|    (默认 10s > 某次 GC 停顿 5s 是安全的)                              |
|                                                                       |
+----------------------------------------------------------------------+
```

# 3. 操作系统调优

```bash
# ============= 磁盘 I/O 优化 =============

# 1. 使用 SSD（特别是数据日志目录 dataLogDir）
#    事务日志是顺序写，SSD 的 IOPS 远高于 HDD

# 2. 挂载选项（减少不必要的元数据更新）
# /etc/fstab
UUID=xxx /data ext4 defaults,noatime,nodiratime,data=writeback 0 2

# 3. I/O 调度器（SSD 使用 noop 或 none）
echo noop > /sys/block/sdb/queue/scheduler

# 4. 预读大小
blockdev --setra 256 /dev/sdb

# ============= 网络优化 =============

# 5. 增加 TCP 缓冲区
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# 6. 减少 TIME_WAIT
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=30

# 7. 增加连接队列
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# ============= 系统限制 =============

# 8. 文件句柄
ulimit -n 65536

# 9. 最大进程数
ulimit -u 65536

# 10. 禁用 Swap
swapoff -a
# 永久禁用：/etc/fstab 中注释 swap 行
```

# 4. ZooKeeper 参数调优

## 4.1 关键性能参数

```properties
# zoo.cfg 中的性能相关参数详解

# ============ 基础参数 ============

# tickTime：基本时钟周期
# 减小可加快超时检测，但增加 CPU 开销
# 建议：2000（默认），网络极好时可降为 1000
tickTime=2000

# ============ 并发参数 ============

# globalOutstandingLimit：Leader 最大未提交请求数
# 增大可提升写吞吐，但增加内存和崩溃恢复时间
# 默认 1000，高并发场景可调至 2000-5000
globalOutstandingLimit=1000

# maxClientCnxns：每个 IP 最大连接数
# 根据客户端数量调整，网关类应用可能需要更高
maxClientCnxns=200

# maxCnxns：全局最大连接数（3.6+）
# 0 表示不限制
maxCnxns=0

# ============ 快照参数 ============

# snapCount：触发快照的事务数量
# 减小：恢复速度快但 IOPS 提升
# 增大：减少快照频率但恢复慢
# 默认 100000，数据量大时可增至 500000
snapCount=100000

# preAllocSize：事务日志预分配大小（KB）
# 避免运行时动态扩展文件的开销
# 默认 65536（64MB）
preAllocSize=65536

# ============ 持久化参数 ============

# forceSync：是否每次事务都 fsync
# yes：最安全，每次写事务都刷盘
# no：高性能但可能丢失未刷盘的事务
# 生产必须设为 yes
forceSync=yes

# snapCount 和 forceSync 的权衡：
# +------------------------------------------------------------------+
# | 场景              | forceSync | snapCount | 写性能 | 持久性       |
# | 金融交易          | yes       | 50000     | 低     | 最高         |
# | 一般业务          | yes       | 100000    | 中     | 高           |
# | 日志/监控（可丢） | no        | 500000    | 极高   | 低           |
# +------------------------------------------------------------------+

# ============ 网络参数 ============

# nio.numSelectorThreads：NIO Selector 线程数（3.5+）
# 默认为核心数/2，可适当增大
nio.numSelectorThreads=4

# nio.numWorkerThreads：NIO Worker 线程数（3.5+）
# 0 表示使用默认（核心数 * 2）
nio.numWorkerThreads=0

# Netty 相关（3.6+，使用 Netty 取代 NIO 时）
serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory

# ============ 选举参数 ============

# electionAlg：选举算法
# 3.4.0+ 始终为 FastLeaderElection（无需配置）
# electionAlg=3

# cnxTimeout：选举连接超时
cnxTimeout=5000  # 5 秒
```

## 4.2 Session 参数

```properties
# Session 相关调优

# minSessionTimeout：最小超时（tickTime 倍数，默认 2）
# 防止客户端设置过短的超时导致频繁过期
minSessionTimeout=4000   # 4s

# maxSessionTimeout：最大超时（tickTime 倍数，默认 20）
# 防止客户端设置过长的超时导致故障检测太慢
maxSessionTimeout=60000  # 60s
```

# 5. 读性能优化

```text
读性能优化策略：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 增加 Observer 节点                                                |
|     · 所有数据在内存中，加 Observer 可线性扩展读吞吐                  |
|     · Observer 不参与投票，不影响写性能                               |
|                                                                       |
|  2. 客户端就近连接（就近读）                                          |
|     · 将 Observer 部署在应用所在机房或区域                            |
|     · 降低读延迟                                                      |
|                                                                       |
|  3. 使用 Curator Cache 减少请求                                       |
|     · NodeCache / PathChildrenCache / TreeCache                       |
|     · 本地缓存 + Watch 增量更新，减少直接请求次数                     |
|                                                                       |
|  4. 使用异步 API                                                      |
|     · ZooKeeper 的异步 API 可提高单客户端的吞吐                      |
|     · Curator 的 inBackground()                                      |
|                                                                       |
|  5. 合理设计 ZNode 结构                                               |
|     · 避免过深的树结构（多次 getChildren）                            |
|     · 批量数据放在一个 ZNode 中（一次 getData 获取所有）             |
|                                                                       |
+----------------------------------------------------------------------+
```

# 6. 写性能优化

```text
写性能优化策略：
+----------------------------------------------------------------------+
|                                                                       |
|  1. 独立事务日志磁盘（关键！）                                        |
|     · dataLogDir 使用独立 SSD                                       |
|     · 事务日志顺序写，SSD 的 IOPS 优势巨大                           |
|     · 这是性价比最高的优化                                           |
|                                                                       |
|  2. 调整 globalOutstandingLimit                                       |
|     · 增大可让 Leader 流水线处理更多请求                              |
|     · 但要注意内存占用                                                |
|                                                                       |
|  3. 减少 Follower 数量                                                |
|     · 写操作需要过半 Follower 的 ACK                                  |
|     · 3 节点优于 5 节点优于 7 节点（投票开销）                        |
|     · 需要更多读能力用 Observer，不增加 Follower                     |
|                                                                       |
|  4. 批量操作                                                          |
|     · 使用 multi 操作合并多个写为一个事务                             |
|     · 减少网络往返和 ZAB 协议开销                                    |
|                                                                       |
|  5. 低延迟网络                                                        |
|     · 节点间网络延迟直接影响写操作耗时                                |
|     · 使用万兆网络或 InfiniBand                                       |
|     · 将节点部署在同一机架/交换机下                                   |
|                                                                       |
|  6. 避免磁盘竞争                                                      |
|     · 事务日志和快照放在不同的磁盘上                                  |
|     · 避免与其他 I/O 密集型应用共享磁盘                               |
|                                                                       |
+----------------------------------------------------------------------+
```

# 7. 性能压测

```bash
# 官方自带性能测试工具：zk-smoketest / zk-latencies

# 使用 Java 自带基准测试
java -cp zookeeper.jar:lib/* org.apache.zookeeper.test.LatencyTest \
  -host localhost:2181 \
  -time 30000 \
  -operation create,getData,setData,delete

# 输出示例：
# create latency: avg=2.3ms, 99%=8.1ms
# getData latency: avg=0.5ms, 99%=1.2ms
```

```java
// 自定义性能压测代码
public class ZkBenchmark {
    private final CuratorFramework client;
    private final ExecutorService executor;
    private final AtomicLong successCount = new AtomicLong();
    private final AtomicLong failCount = new AtomicLong();

    public void benchmark(int threads, int operationsPerThread)
            throws Exception {
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch endLatch = new CountDownLatch(threads);

        long startTime = System.currentTimeMillis();

        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                try {
                    startLatch.await();
                    for (int j = 0; j < operationsPerThread; j++) {
                        try {
                            client.setData()
                                .forPath("/bench/test",
                                    ("value-" + j).getBytes());
                            successCount.incrementAndGet();
                        } catch (Exception e) {
                            failCount.incrementAndGet();
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    endLatch.countDown();
                }
            });
        }

        startLatch.countDown();  // 同时起跑
        endLatch.await();

        long duration = System.currentTimeMillis() - startTime;
        long total = successCount.get() + failCount.get();

        System.out.println("Duration: " + duration + "ms");
        System.out.println("Total ops: " + total);
        System.out.println("QPS: " + (total * 1000.0 / duration));
        System.out.println("Success: " + successCount.get());
        System.out.println("Failed: " + failCount.get());
        System.out.println("Avg latency: " +
            (duration * threads / (double) total) + "ms");
    }
}
```

# 8. 性能诊断工具

| 工具 | 用途 | 命令/方式 |
|------|------|----------|
| **mntr** | 关键性能指标总览 | `echo mntr \| nc localhost 2181` |
| **stat** | 服务端基本统计 | `echo stat \| nc localhost 2181` |
| **srst** | 重置统计计数器 | `echo srst \| nc localhost 2181` |
| **JMX** | JVM 和 ZK 运行指标 | jconsole / jmxterm |
| **async-profiler** | CPU/内存 Profiling | `profiler.sh -d 30 -f flamegraph.html PID` |
| **iostat** | 磁盘 I/O 监控 | `iostat -x 1` |
| **jstat** | JVM GC 统计 | `jstat -gcutil PID 1000` |

```text
mntr 输出关键指标解读：
+----------------------------------------------------------------------+
|                                                                       |
|  zk_version      3.9.2                                                |
|  zk_avg_latency  2          ← 平均延迟（ms），应 < 10ms              |
|  zk_max_latency  50         ← 最大延迟（ms）                         |
|  zk_min_latency  0                                                    |
|  zk_packets_received  1000000  ← 收到的请求包总数                    |
|  zk_packets_sent      1000000  ← 发送的响应包总数                    |
|  zk_num_alive_connections  50  ← 当前活跃连接数                      |
|  zk_outstanding_requests   5   ← 当前未完成请求数                    |
|  zk_server_state follower      ← 节点角色                            |
|  zk_znode_count     5000    ← ZNode 总数                             |
|  zk_watch_count     200     ← Watch 总数                             |
|  zk_ephemerals_count     100  ← 临时节点数                           |
|  zk_approximate_data_size 5000000  ← 数据总大小（字节）              |
|  zk_open_file_descriptor_count  300 ← 打开的文件描述符数             |
|  zk_max_file_descriptor_count  65536 ← 最大文件描述符数              |
|                                                                       |
+----------------------------------------------------------------------+
```
