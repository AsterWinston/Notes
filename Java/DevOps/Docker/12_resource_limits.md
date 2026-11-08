
# 1. 为什么要限制资源

容器共享宿主机内核，本质是"进程集合 + cgroup/namespace 约束"。资源限制的目的：

1. 防止单个容器把宿主机打满（CPU 抢占、内存吃光导致 OOM）。
2. 让应用在可预测的资源配额下运行，便于容量规划。
3. 为多服务共存提供隔离边界。

结论：

1. 不设置限制并不等于"更快"，往往意味着更不稳定——"吵闹邻居"问题在没有配额时几乎必然发生。
2. 资源限制不是调优的终点，应用仍需做线程池/连接池/缓存边界等治理。cgroup 只保证"不超过"，不保证"用得好"：一个没有限制线程数的 Java 应用，在 64 核宿主机上会按 64 核创建 GC 线程与默认线程池，然后在被限到 2 核的容器里疯狂上下文切换。

# 2. 资源限制的底层：cgroup 与可观测性

Docker 通过 cgroup 实现 CPU、内存、IO 等资源控制。

你需要知道两点：

1. cgroup v1/v2 的细节会影响某些指标与行为（比如内存统计口径：v1 的 `memory.stat` 与 v2 的 `memory.current` 对 cache 的计入方式不同）。当前主流发行版（Fedora、Ubuntu 22.04+、Container-Optimized OS）默认 cgroup v2，Docker 24+ 已完整支持。
2. "限制生效"与"应用表现良好"不是一回事，要结合监控与压测验证。

快速观察容器资源：

```bash
docker stats                        # 实时 CPU/内存/IO/网络
docker inspect <container>          # 看 HostConfig 里你到底配置了什么

# 直接读 cgroup(以 cgroup v2 为例)
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/memory.max
cat /sys/fs/cgroup/system.slice/docker-<id>.scope/cpu.max   # "quota period" 或 "max 100000"
```

# 3. CPU：cpus / cpu-quota / cpu-shares / cpuset

## 3.1 限制 CPU 时间片：`--cpus`

最推荐的方式：

```bash
docker run --cpus=2 my-app:1.0
```

含义：容器最多可用 2 个核的 CPU 时间（不是绑定到 2 个固定核）。底层映射为 CFS quota：cgroup v2 里写入 `cpu.max = "200000 100000"`。

## 3.2 CFS quota：`--cpu-quota` 与 `--cpu-period`

更底层的等价配置：

```bash
docker run --cpu-period=100000 --cpu-quota=200000 my-app:1.0
```

含义：每 100ms 周期允许使用 200ms CPU 时间，等价 2 CPU。

**throttling（节流）是理解 CPU limit 的关键词**：容器用满 quota 后，CFS 会把整个 cgroup 组内所有线程挂起到下个周期，表现为"CPU 利用率明明不高，延迟却毛刺严重"。用 `docker stats --no-stream` 看不到这个，要看 cgroup 计数：

```bash
cat /sys/fs/cgroup/.../cpu.stat
# nr_throttled: 被节流周期数, throttled_usec: 节流总时长
# 压测时 nr_throttled 持续增长 = limit 已成为瓶颈
```

## 3.3 相对权重：`--cpu-shares` / `--cpus` 之外的软控制

```bash
docker run --cpu-shares=1024 my-app:1.0
```

这是"争抢时的相对权重"，不是硬限制。空闲时低 shares 容器一样能跑满所有核。

## 3.4 绑定到指定核：`--cpuset-cpus`

```bash
docker run --cpuset-cpus="0-1" my-app:1.0
docker run --cpuset-mems="0" my-app:1.0     # NUMA 内存绑定
```

适用：对 CPU cache 亲和性敏感的场景、或想减少抖动。

风险：

1. 配置错误可能让容器只剩很少可用 CPU。
2. 宿主机 NUMA 拓扑复杂时更要谨慎——绑定跨 NUMA 的核会引入远端内存访问延迟。

## 3.5 四种 CPU 参数对照

| 参数 | 类型 | 语义 | 典型场景 |
|------|------|------|----------|
| `--cpus=2` | 硬上限 | 最多 2 核时间（CFS quota） | 默认推荐 |
| `--cpu-quota/--cpu-period` | 硬上限 | quota 底层等价写法 | 精确控制周期 |
| `--cpu-shares` | 软权重 | 争抢时按比例分配 | 重要/次要服务分级 |
| `--cpuset-cpus` | 亲和绑定 | 只能用指定核 | 低延迟、license 限制、NUMA |

> **注意**：Java 应用对"quota 型限制"有一个历史坑——CFS throttle 会让 JVM 的 `Runtime.availableProcessors()` 依然返回宿主机核数（老 JDK），GC 线程按宿主机规模创建但只有 2 核额度可用，GC 暂停显著拉长。JDK 10+ 默认开启容器感知后才缓解，这是第 5 节主题。

# 4. 内存：memory / swap / oom

## 4.1 设置内存上限：`--memory`

```bash
docker run --memory=512m my-app:1.0
```

达到上限后可能触发 OOM kill（容器内进程被内核杀死）。内存是硬边界，**超了就是杀，没有 CPU 那种"变慢"的中间态**，所以内存 limit 是所有资源参数里最需要预留余量的。

## 4.2 swap 行为：`--memory-swap`

典型用法：

```bash
# 禁用 swap（memory+swap 等于 memory）
docker run --memory=512m --memory-swap=512m my-app:1.0

# 512M 内存 + 512M swap
docker run --memory=512m --memory-swap=1g my-app:1.0
```

说明：`--memory-swap` 是 memory+swap 的**总额**语义（cgroup v1），swap=memory 表示禁用 swap；cgroup v2 下对应 `memory.swap.max`。JVM 生产环境普遍建议**禁用 swap**——堆内存被换出会造成不可预期的超长停顿，宁可让容器被 OOM 杀了重启。

## 4.3 OOM 行为与 exit 137

常见现象：容器突然退出，日志里可能看不到 Java/Python 自己的异常——**OOM kill 是内核发出的 SIGKILL，进程没有机会打印任何错误**。

```bash
docker inspect <container> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
# true 137    ← 137 = 128 + 9(SIGKILL)
```

看到 exit 137 先分流：`OOMKilled=true` 是 cgroup 内存超限；`OOMKilled=false` 的 137 多半是人为 `docker kill`、编排平台删除、或宿主机会话被 OOM 回收。排查方向：

1. `docker inspect` 看 `State.OOMKilled`。
2. 宿主机 `dmesg -T | grep -i "killed process"`/系统日志，能看到 cgroup 路径、RSS、各内存分区，判断是堆、元空间还是堆外。
3. Java 应用要显式关注容器内存感知（不同 JDK 版本默认策略不同）。

> **行业实践**：给 JVM 容器加 `-XX:+ExitOnOutOfMemoryError -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps`，堆内 OOM 快速退场交给重启策略，堆外/cgroup OOM 留给 137 排查路径；两者信号分开，不要混为一谈。

# 5. Java 专属：limit 与堆计算（UseContainerParameters）

JVM 的堆默认值是 `Runtime.maxMemory()` 推导出来的，容器出现之前它直接读宿主机总内存——这是"512M 容器里 JVM 按 32G 宿主机设堆，启动即 OOMKilled"的经典事故。

**JDK 10+ 默认启用容器感知**（`-XX:+UseContainerSupport`），JVM 会读 cgroup 的 memory limit 而不是 /proc/meminfo。核心参数：

```bash
java -XX:+UseContainerSupport \
     -XX:MaxRAMPercentage=60.0 \        # 堆 = 容器 limit 的 60%
     -Xlog:gc*:file=gc.log \
     -jar app.jar

# 更直接: 显式 -Xmx, 但要自己保证整个进程内存 < limit
java -Xmx320m -XX:MaxDirectMemorySize=64m -jar app.jar
```

**设计考量**——为什么 `MaxRAMPercentage=60` 而不是 80/90？因为容器 memory limit 计的是**进程总内存（RSS）**，不只是堆：

```text
容器 memory limit 512M
┌────────────────────────────────────────────┐
│ Heap (Xmx / MaxRAMPercentage)   ≈ 307M(60%) │
│ Metaspace                        ≈ 60-100M  │
│ 线程栈: 200 线程 × 1M = 200M(可 -Xss 调小)   │
│ DirectByteBuffer/NIO 堆外       = MaxDirectMemorySize │
│ Code Cache/GC结构/JVM自身        ≈ 50-80M    │
│ glibc arena、mmap 文件            若干        │
└────────────────────────────────────────────┘
超 limit → 内核 OOM kill(exit 137), JVM 无任何日志
```

线程数是最大变量：Tomcat/虚拟线程/各类线程池叠加到几百线程时，仅栈内存就可能吃掉三分之一 limit。压测期用 `jcmd <pid> VM.native_memory summary`（需 `-XX:NativeMemoryTracking=summary`）看 JVM 自己承认的总占用，再对照 `docker stats` 差值估算堆外。

| 做法 | 效果 |
|------|------|
| 不设 limit + 不设 Xmx | JVM 按宿主机内存设堆, 多容器混部时整机 OOM |
| 设 limit + `-Xmx` 接近 limit | 堆外内存没空间, cgroup OOM 137 |
| 设 limit + `MaxRAMPercentage=50~70` | 推荐起点, 压测校准 |
| 用 ZGC/Shenandoah | 堆外与映射开销更大, 余量要再多留 |

# 6. PIDs 与 IO

## 6.1 限制进程数防止 fork bomb

```bash
docker run --pids-limit=256 my-app:1.0
```

价值：防止异常创建线程/子进程把宿主机拖死。注意 **cgroup pids 计的是线程数**（Linux 里线程即轻量进程），Java 大线程池服务设置过低会直接导致 `unable to create native thread`。

## 6.2 块设备 IO 权重/限速（按需使用）

```bash
# 权重: 竞争时相对优先级(cgroup v1 blkio)
docker run --blkio-weight=500 my-app:1.0

# 硬限速: 读写 BPS/IOPS 上限(cgroup v2 io.max 的实现)
docker run --device-read-bps /dev/sda=10mb \
           --device-write-bps /dev/sda=50mb \
           --device-read-iops /dev/sda=1000 \
           my-app:1.0
```

blkio 只对**块设备**生效：网络文件系统（NFS、云盘 EBS 的限流在云厂商侧）、tmpfs、overlay 上层与 volume 的实际设备路径都受影响或无效，行为依赖内核与存储驱动。

工程建议：

1. 真要做 IO 隔离，优先从"存储架构与磁盘 QoS"整体方案考虑。
2. 仅靠容器参数可能无法解决根因（共享盘、宿主机缓存抖动等）。

# 7. Compose 中写资源限制与 limit vs reservation

Compose 常见写法：

```yaml
services:
  app:
    image: my-app:1.0
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 512M
          pids: 256
        reservations:
          cpus: '0.5'
          memory: 256M
```

**limit 与 reservation 的语义差别**：

| 维度 | limits | reservations |
|------|--------|--------------|
| 性质 | 硬上限, 超过被 throttle/kill | 保障下限, 调度/资源分配参考 |
| docker run 对应 | `--cpus/--memory` | 单机下基本无强对应 |
| Swarm | 运行时生效 | **调度约束**: 节点剩余不足则该服务无法调度 |
| K8s 对应 | `limits` | `requests` |

`deploy.resources` 的调度语义（reservation 参与选节点）在 Swarm 里完整生效；`docker compose up`（单机）下 reservations 基本只是摆设，limits 会真实落到 cgroup。提示：不同 compose/实现对 `deploy` 的支持有差异，**永远用 `docker inspect` 确认最终限制是否真正落到了容器上**。

# 8. 与 Kubernetes requests/limits 对照及超卖策略

| Docker 参数 | K8s 字段 | cgroup 落点 | 超限行为 |
|-------------|----------|-------------|----------|
| `--cpus=2` | `resources.limits.cpu` | cpu.max | CFS throttle（变慢） |
| `--memory=512m` | `resources.limits.memory` | memory.max | OOMKilled（杀掉） |
| reservations.memory | `resources.requests.memory` | 调度依据; v1 下 memory.min/low | 保障/调度水位 |
| reservations.cpu | `resources.requests.cpu` | cpu.weight/shares | CPU 是可压缩资源, 不阻断调度 |

**超卖（overcommit）策略**是资源治理的深水区：

- **CPU 超卖是常态且相对安全**：requests 之和 > 节点容量也能调度，忙时按比例共享、throttle 掉多余部分，服务只是变慢不会死。
- **内存超卖是危险的**：内存不可压缩，所有 Pod 同时逼近 limit 时触发节点级系统 OOM，内核按 oom_score 挑受害者（可能是随机业务），比逐 Pod OOMKilled 更难排查。GKE 等托管集群允许 `overcommit`（Pod 密度换 CPU 成本），但要求内存 requests 总和 ≤ 可分配量减系统预留。
- **Guaranteed / Burstable / BestEffort** 三级 QoS 由 requests/limits 组合决定，节点资源紧张时驱逐顺序即按此排——Java 生产服务至少做到 requests==limits（Guaranteed 或接近），配合 `UseContainerSupport` 让 JVM 按 limit 精确规划堆。

# 9. 经验法则（给一个可执行起点）

1. 先设置内存上限，按 `MaxRAMPercentage≈60` 起步，再看压测与 GC/缓存命中率校准。
2. CPU 先用 `--cpus` 做"上限约束"，再从线程池与并发控制治理；盯 `cpu.stat` 的 throttled 指标。
3. 对会大量创建线程/子进程的服务，加 `--pids-limit`（按线程峰值 ×1.5 设置）。
4. 生产禁用容器 swap（`--memory-swap` 等于 `--memory`）。
5. 用 `docker stats` 只是起点，线上还是要有更系统的监控（见 `16_logging_monitoring.md`）。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器资源限制靠什么实现 | cgroup（CPU/内存/IO/PIDs），v1/v2 行为差异 |
| `--cpus` 是什么 | CPU 时间上限（CFS quota 的抽象），超限 throttle |
| `--cpu-shares` 是什么 | 相对权重，争抢时生效 |
| `--memory` 触发什么 | 超限 OOMKilled，exit 137，进程无日志 |
| JVM 容器堆怎么算 | UseContainerSupport(10+默认) + MaxRAMPercentage，limit 计 RSS 要留堆外余量 |
| 为什么限制 PIDs | 防止进程/线程爆炸拖垮宿主机；pids 计数含线程 |
| limit 与 reservation | 硬上限 vs 调度保障；K8s 对应 limits vs requests |

# 11. 小结

资源限制的知识结构是"一个机制、两条边界、一个 Java 特例"。机制是 cgroup：CPU 与内存的超限行为本质不同——CPU 可压缩、超限 throttle（变慢可恢复），内存不可压缩、超限 SIGKILL（exit 137 且无日志），因此内存参数永远优先设置并留足余量。Java 特例是 JVM 的堆规划：JDK 10+ 默认容器感知后，用 `MaxRAMPercentage` 在"limit = 进程总 RSS"的口径下给 Metaspace、线程栈、堆外留出 30-40% 空间，再用 NMT 与 `cpu.stat`、`memory.max` 实测校准。编排平台上限与保障分别对应 limits/requests，CPU 可以放心超卖、内存必须克制。这些参数在 Docker、Compose `deploy.resources`、K8s 之间一一对应，理解 cgroup 落点后换平台只是换语法。安全（`13_security.md`）是下一篇：资源边界之外的权限边界。
