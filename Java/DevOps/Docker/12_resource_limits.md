
# 1. 为什么要限制资源

容器共享宿主机内核，本质是“进程集合 + cgroup/namespace 约束”。资源限制的目的：

1. 防止单个容器把宿主机打满（CPU 抢占、内存吃光导致 OOM）。
2. 让应用在可预测的资源配额下运行，便于容量规划。
3. 为多服务共存提供隔离边界。

结论：

1. 不设置限制并不等于“更快”，往往意味着更不稳定。
2. 资源限制不是调优的终点，应用仍需做线程池/连接池/缓存边界等治理。

# 2. 资源限制的底层：cgroup 与可观测性

Docker 通过 cgroup 实现 CPU、内存、IO 等资源控制。

你需要知道两点：

1. cgroup v1/v2 的细节会影响某些指标与行为（比如内存统计口径）。
2. “限制生效”与“应用表现良好”不是一回事，要结合监控与压测验证。

快速观察容器资源：

```bash
docker stats
```

以及查看容器的 HostConfig（你到底配置了什么）：

```bash
docker inspect <container>
```

# 3. CPU：cpus / cpu-quota / cpu-shares / cpuset

## 3.1 限制 CPU 时间片：`--cpus`

最推荐的方式：

```bash
docker run --cpus=2 my-app:1.0
```

含义：容器最多可用 2 个核的 CPU 时间（不是绑定到 2 个固定核）。

## 3.2 CFS quota：`--cpu-quota` 与 `--cpu-period`

更底层的等价配置：

```bash
docker run --cpu-period=100000 --cpu-quota=200000 my-app:1.0
```

含义：每 100ms 周期允许使用 200ms CPU 时间，等价 2 CPU。

## 3.3 相对权重：`--cpu-shares`

```bash
docker run --cpu-shares=1024 my-app:1.0
```

这是“争抢时的相对权重”，不是硬限制。

## 3.4 绑定到指定核：`--cpuset-cpus`

```bash
docker run --cpuset-cpus="0-1" my-app:1.0
```

适用：对 CPU cache 亲和性敏感的场景、或想减少抖动。

风险：

1. 配置错误可能让容器只剩很少可用 CPU。
2. 宿主机 NUMA 拓扑复杂时更要谨慎。

# 4. 内存：memory / swap / oom

## 4.1 设置内存上限：`--memory`

```bash
docker run --memory=512m my-app:1.0
```

达到上限后可能触发 OOM kill（容器内进程被内核杀死）。

## 4.2 swap 行为：`--memory-swap`

典型用法：

```bash
# 禁用 swap（memory+swap 等于 memory）
docker run --memory=512m --memory-swap=512m my-app:1.0
```

说明：swap 的具体语义与宿主机配置、cgroup 版本相关，生产最好在目标环境验证。

## 4.3 OOM 行为与排查

常见现象：容器突然退出，日志里可能看不到 Java/Python 自己的异常。

排查方向：

1. `docker inspect` 看 `State.OOMKilled`。
2. 宿主机 `dmesg`/系统日志是否有 OOM 记录（需要宿主机权限）。
3. Java 应用要显式关注容器内存感知（不同 JDK 版本默认策略不同）。

# 5. PIDs：限制进程数防止 fork bomb

```bash
docker run --pids-limit=256 my-app:1.0
```

价值：防止异常创建线程/子进程把宿主机拖死。

# 6. IO：块设备 IO 权重/限速（按需使用）

块 IO 相关参数依赖宿主机存储驱动与内核能力，常见思路：

1. 权重：多容器竞争时的相对优先级
2. 限速：读写 BPS/IOPS 上限

示例（概念性，具体参数在不同环境差异较大）：

```bash
docker run --blkio-weight=500 my-app:1.0
```

工程建议：

1. 真要做 IO 隔离，优先从“存储架构与磁盘 QoS”整体方案考虑。
2. 仅靠容器参数可能无法解决根因（共享盘、宿主机缓存抖动等）。

# 7. Compose 中如何写资源限制（单机场景）

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
```

提示：不同 compose/实现对 `deploy` 的支持有差异。你可以用 `docker inspect` 来确认最终限制是否真正落到了容器上。

# 8. 经验法则（给一个可执行起点）

1. 先设置内存上限，再看压测与 GC/缓存命中率。
2. CPU 先用 `--cpus` 做“上限约束”，再从线程池与并发控制治理。
3. 对会大量创建线程/子进程的服务，加 `--pids-limit`。
4. 用 `docker stats` 只是起点，线上还是要有更系统的监控（见 `16_logging_monitoring.md`）。

# 9. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器资源限制靠什么实现 | cgroup（CPU/内存/IO/PIDs） |
| `--cpus` 是什么 | CPU 时间上限（CFS quota 的抽象） |
| `--cpu-shares` 是什么 | 相对权重，争抢时生效 |
| `--memory` 触发什么 | 超限可能 OOMKilled，需要从内核层面排查 |
| 为什么限制 PIDs | 防止进程/线程爆炸拖垮宿主机 |
