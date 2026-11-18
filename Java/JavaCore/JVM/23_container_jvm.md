# 1. 容器中的 JVM 挑战总览

## 1.1 视角差异

```text
传统部署：                 容器化部署：
┌──────────────┐          ┌──────────────────┐
│   物理机/VM  │          │     Pod          │
│  ┌────────┐  │          │ ┌──────────────┐ │
│  │  JVM   │  │          │ │     JVM      │ │
│  │ 8个CPU │  │          │ │ 看到所有CPU  │ │  ← 旧版本 JVM 的自探测
│  │ 16G内存│  │          │ │ 看到所有内存 │ │
│  └────────┘  │          │ └──────────────┘ │
│              │          │  limits:         │
│              │          │    cpu: 2        │
│              │          │    memory: 4Gi   │
└──────────────┘          └──────────────────┘

问题：JVM 默认读取宿主机的 CPU 和内存，
     不知道容器的 limits！
```

## 1.2 cgroup 限制如何作用于 JVM

```text
CPU（cfs 调度器）：
  cfs_quota_us=200000 / cfs_period_us=100000   → 等效 2 核
  ┌────────── 100ms period ──────────┐
  │████ 200ms 配额（多线程分摊）│▒▒ 全部挂起（throttle）│
  └──────────────────────────────────┘
  → 配额耗尽时容器内所有线程一起停摆到下个周期

Memory：
  limit 是"硬顶"：RSS 超限 → cgroup OOM Killer 直接杀进程（exit 137，无 JVM 日志！）
  JVM 的 OOM 异常 vs 内核的 OOMKilled 是两套完全不同的死法（→ 22 第 3 节）

Swap：K8s 节点通常关 swap。若有 swap，停顿会被 paging 放大；limit 计费还包含 swap 用量
```

## 1.3 三大后果

| 后果 | 机制链 | 章节 |
|------|--------|------|
| OOMKilled | 内存预算只算了堆 → 总 RSS 超 limit | 3.1 |
| 停顿恶化 | GC 线程/并发阶段被 throttle；卡表/位图访问 paging | 3.2/3.3 |
| GC 线程错配 | 按宿主机核数开线程 → 配额瞬间烧光 | 3.3 |

# 2. Java 版本的容器感知演进

## 2.1 支持时间线

| 版本 | 容器支持情况 |
|------|-------------|
| ≤ JDK 8u131 | 不支持容器限制，读取宿主机资源（`/proc/cpuinfo`、`/proc/meminfo`） |
| JDK 8u131+ | 实验性支持：`-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap` + `MaxRAMFraction` |
| JDK 8u191+ | `-XX:+UseContainerSupport` **默认开启**，改用规范的 `MaxRAMPercentage` 系列；旧实验参数废弃 |
| JDK 10+ | `UseContainerSupport` 默认开启 |
| JDK 15+ | 支持 cgroup v2；后续 8u372+/11.0.16+ 等更新陆续回移（Ubuntu 22.04+/Containerd 新装环境多为 v2，老 JDK 需升级小版本） |
| JDK 21 LTS | 容器内行为成熟：CPU quota→核数、内存 limit→堆默认值、`-Xlog:os,container` 可直接打印识别结果 |

## 2.2 关键参数

```bash
-XX:+UseContainerSupport      # 启用容器支持（新版本默认开）
-XX:ActiveProcessorCount=2    # 手动指定可用 CPU 数（覆盖自动检测，排障/共享环境利器）
-XX:InitialRAMPercentage=75   # 初始堆占容器内存的百分比
-XX:MaxRAMPercentage=75       # 最大堆占容器内存的百分比（默认 25！）
-XX:MinRAMPercentage=50       # 小内存时堆占比
# 验证是否感知到位：
java -Xlog:os+container -version        # 打印检测到的 cgroup 配额
jcmd <pid> VM.flags | grep MaxHeap      # MaxHeapSize == limit×25%? 一眼识破"没感知"
```

**默认值陷阱**：不设 -Xmx 时堆 = **limit × MaxRAMPercentage(默认25%)**。有人嫌堆小把 limit 翻倍 → 非堆照旧涨 → 依然被杀；有人配了 `-Xmx` 却忘配非堆上限 → 见 3.1。

## 2.3 CPU 感知的一个盲区：requests ≠ limits

```text
K8s 只有 limits.cpu 会写进 cgroup quota；requests 只是调度权重。
  limits.cpu=2                → JVM 看到 2 核 ✅
  requests.cpu=2, 无 limits   → JVM 看到整个节点核数（超额可突发）
  → Burstable Pod 里 JVM 线程数/堆默认值都会"放大"，需要 ActiveProcessorCount 手工兜底
```

# 3. 四大陷阱

## 3.1 陷阱一：OOMKilled

### 3.1.1 进程内存构成公式

| 构成 | 受控参数 | 默认倾向 | NMT 归属（→ 20 第 10 节） |
|------|----------|----------|---------------------------|
| 堆 | -Xmx / MaxRAMPercentage | limit×25% | Heap |
| 元空间 | -XX:MaxMetaspaceSize | **无上限** | Class |
| 线程栈 | -Xss × 线程数 | 512k~1M/Linux 默认另算 | Thread |
| CodeCache | -XX:ReservedCodeCacheSize | 240M（reserved，committed 随 JIT 增长） | Code |
| GC 结构 | 随堆自动 | 卡表/标记位图/记忆集约**堆的百分之几，G1 RSet 重跨区引用可更高** | GC |
| 直接内存 | -XX:MaxDirectMemorySize | ≈Xmx！ | NMT 覆盖不完整，需 pmap 配合 |
| JNI/第三方库 | 不可控 | mmap 大户 | 不在 NMT |
| JVM 内部/碎片 | — | 少量 | Internal/Symbol/Other |

### 3.1.2 4Gi 的反例演算

```text
limit = 4Gi，配置 -Xmx3g（自以为留了 1G 余量）：
  堆 committed            2.8~3.0G（-Xmx3g 用满）
+ 线程栈 200×1M            0.2G
+ Metaspace（无上限）      实测 0.3~0.5G
+ CodeCache committed      ~0.2G
+ DirectBuffer             0.25G
+ GC/JVM 结构+碎片         ~0.5G
──────────────────────────────
合计                     ≈ 4.2G  →  OOMKilled（exit 137，日志里啥异常都没有）
```

```bash
# 错误配置（只有 4Gi 的容器）
-Xmx3g                       # 堆 3G
# 元空间默认无限制            # 可能到 1G
# 线程栈 200 线程 × 1MB      # 200MB
# 总内存 > 4Gi → OOMKilled

# 正确配置：显式给每个非堆组件"上镣铐"
-Xms2304m -Xmx2304m           # 堆 ~2.25G (< 75%)
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=128m
-XX:ReservedCodeCacheSize=128m
-Xss256k                      # 减小栈，允许更多线程
```

### 3.1.3 方案表（按优先级）

| 手段 | 参数 | 备注 |
|------|------|------|
| 压堆占比 | `-XX:MaxRAMPercentage=60~75` | 高线程/高缓冲服务取下限 |
| 限栈 | `-Xss256k~512k` | 深递归服务慎压（StackOverflowError） |
| 限直接内存 | `-XX:MaxDirectMemorySize` | Netty/Kafka 客户端必配 |
| 限类元数据 | `-XX:MaxMetaspaceSize` | 同时兜住 ClassLoader 泄漏 |
| 限线程 | 线程池上限、虚拟线程化（21） | 200 线程→2G? 栈开销随线程数线性 |
| 兜底可观测 | `-XX:NativeMemoryTracking=summary` + `ExitOnOutOfMemoryError` | 死后由 K8s 拉起，dump 留现场 |

### 3.1.4 判定与取证链

```bash
kubectl describe pod xxx | grep -A3 Last State    # Reason: OOMKilled, Exit Code: 137
dmesg | grep -i "killed process"                   # 节点视角
# 若 JVM 内一切"看起来正常" → 堆外记账问题：
jcmd <pid> VM.native_memory summary.diff           # 开 NMT 的实例
pmap -x <pid> | sort -k3 -n | tail -20             # top 匿名 mmap
```

## 3.2 陷阱二：CPU Throttling

### 3.2.1 机制：配额在周期内烧光

```text
limit=2C → quota=200ms / period=100ms
应用+GC+JIT+Netty IO 等 30 个线程共享这 200ms：
  0ms         60ms        100ms
  │───全员狂奔───│──配额用尽──▓▓▓▓ 全员冻结 ▓▓▓▓│──下个周期──▶
                     ↑ GC 并发标记线程也被冻 → 标记周期拉长 → 堆占用爬升 → 更多 GC
```

**对 JVM 的特殊伤害**：并发 GC 阶段（G1 标记/ZGC 重定位）本质是"后台带宽换低停顿"，被节流后并发追不上分配 → Allocation Stall / Mixed 堆积 / Full GC；这就是"CPU limit 压太紧 → GC 恶化"的完整因果链。

### 3.2.2 观测与对策

```bash
# cgroup 视角（v1 路径同理 v2 换目录）
cat /sys/fs/cgroup/cpu/cpu.stat          # nr_throttled / throttled_time（v2: nr_throttled_usec）
# 容器监控：cAdvisor 指标 container_cpu_cfs_throttled_periods_total /
#   _seconds_total —— 看 throttled periods 占比，>10%~20% 已伤及延迟
```

| 对策 | 做法 |
|------|------|
| Guaranteed QoS | `requests.cpu == limits.cpu`，尽量整数核（2 而不是 1.5） |
| 留余量 | limit 不是"够用"而是"峰值×1.3"；把 GC 线程也算进 CPU 预算 |
| 削线程 | `-XX:ParallelGCThreads`/`ConcGCThreads`/业务池并行度对齐 limit |
| 别误判 | 节点级 CPU 空闲但 throttle 高 ≠ 集群没资源 |

## 3.3 陷阱三：GC 线程错配

### 3.3.1 默认公式（感知正确后自动成立）

```text
可用核数 C = max(1, ceil(cfs_quota / cfs_period))     ← 容器 limit
ParallelGCThreads：C ≤ 8 时 = C；否则 8 + (C-8)×5/8（整除）
ConcGCThreads   ：ceil(ParallelGCThreads / 4)

原问题：旧 JVM 读宿主机 32 核 → ParallelGCThreads≈20+，2 核配额 → 风暴
容器 2 核时（感知正确）：
  ParallelGCThreads ≈ 2
  ConcGCThreads     ≈ 1
```

### 3.3.2 <2 核场景与显式覆盖

```bash
# ceil 后 500m 也算 1 核：GC 单线程化，并发 GC 名存实亡
# 对策：要么给到 ≥1 核 limit 并留余量，要么极小容器直接换收集器
-XX:+UseSerialGC                      # 0.5C~1C 边车型服务：省内存省线程，停顿可控
# 显式覆盖（共享/突发环境、JVM 没感知对时）
-XX:ParallelGCThreads=2
-XX:ConcGCThreads=1
-XX:ActiveProcessorCount=2            # 从源头纠偏：JIT/池默认大小/ForkJoin 都跟着变
```

### 3.3.3 与 Throttling 的相互作用

```text
G1：ConcGCThreads=1 时标记带宽极低 → IHOP 触发 Mixed 已晚 → Evacuation Failure 风险↑（见 22 4.4）
ZGC：重定位/预处理线程被节流 → Allocation Stall；低核容器上 ZGC 的"并发红利"会缩水
→ 小核数 + 低延迟诉求：先加 CPU limit，再谈换收集器；GC 理论见 [12_gc_collectors](12_gc_collectors.md)
```

### 3.3.4 观察手段

```bash
-Xlog:gc+phases=debug:file=gc.log      # 看 GC 各阶段耗时是否"周期性尖刺"（与 throttle 对齐）
jcmd <pid> VM.flags | grep -E "ParallelGCThreads|ConcGCThreads|MaxHeapSize"
kubectl exec po/xxx -- cat /sys/fs/cgroup/cpu.max    # 容器内亲眼看 quota
```

## 3.4 陷阱四：/proc 读取与旧版本坑

- JDK 8u131 之前，JVM 读 `/proc/cpuinfo` 和 `/proc/meminfo`，获得**宿主机**数据 → 堆默认值、线程默认数全放大。
- 新版本 JDK 改读 cgroup 文件（`/sys/fs/cgroup/...`），cgroup v2 的新路径/新单位（bytes 版 memory.max、`cpu.max` 一行两值）对老 JDK 是"读了个寂寞"→ 表现为"升级了 JDK 大版本但 8u 小版本太老"依旧翻车。
- 其他杂项坑：
  - `Runtime.availableProcessors()` 在感知失效时≠配额；第三方库（Netty EventLoop、ForkJoinPool）都以它为准 → 连锁放大。
  - 宿主机 `/proc/meminfo` 的 MemAvailable 被当作参考 → `MaxRAM` 计算错误。
  - DNS/hostname 解析在 K8s 里慢 → 启动期看似"JVM 卡住"，实际卡在类加载前的 JDK 网络初始化（老 bug，新小版本已修）。
- 自检口诀：**换任何镜像/JDK 小版本后，第一件事 `jcmd VM.flags | grep MaxHeapSize`，看堆默认值是不是 limit 的 25%。**

# 4. 容器化最佳实践

## 4.1 内存公式化配置（2C4G 完整预算表）

### 4.1.1 完整参数块

```bash
# 容器 2CPU / 4Gi 内存
-XX:MaxRAMPercentage=75              # 堆占 75%
-Xss256k                             # 栈 256K
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=128m
-XX:ReservedCodeCacheSize=128m
-XX:+UseContainerSupport             # 容器感知（默认开，写上表诚意）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError          # k8s 自动重启
```

### 4.1.2 内存预算表

| 预算项 | 配额 | 4Gi 占比 |
|--------|------|----------|
| 堆 | 3Gi（75%） | 75% |
| 线程栈 | ~100MB（按 200×512k 估） | 2.5% |
| Metaspace | 256MB | 6% |
| CodeCache | 128MB | 3% |
| Direct | 128MB | 3% |
| GC/JVM 结构 | ~300MB（堆的 10% 经验值） | 7% |
| **安全余量** | **~350MB** | **~8%** |

> 经验：堆占比 **60~75%**（线程多/Netty 重 → 60%；轻服务 → 75%），余量永远留给"你没算到的那部分"。dump 文件写 emptyDir/tmpfs 会**计入内存**！→ 落盘挂 PV。

## 4.2 Dockerfile 建议

```dockerfile
FROM eclipse-temurin:21-jre

# 通用 JVM 参数走环境变量，便于平台统一注入（JVM 会打印 "Picked up JAVA_TOOL_OPTIONS"）
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70 -XX:+ExitOnOutOfMemoryError -XX:NativeMemoryTracking=summary"

# exec 形式：java 是 PID 1，能直接收到 SIGTERM（shell 形式会被 /bin/sh 截胡，优雅停机失效）
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:+UseG1GC", \
    "-jar", "/app.jar"]
```

| 项 | 建议 | 理由 |
|----|------|------|
| PID 1 | exec form 或 tini | shell 吞 SIGTERM → K8s 只能等 30s 后 SIGKILL |
| `JAVA_TOOL_OPTIONS` | 平台级统一注入 | 镜像参数之后同名参数后者生效，注意覆盖顺序 |
| `NativeMemoryTracking=summary` | 可选开 | 堆外排障保险，开销 ~5%（→ 20 第 10 节） |
| `ExitOnOutOfMemoryError` | 开 | 云原生"快速失败"，配合 1.4/22 第 1 节两难取舍 |
| `AlwaysPreTouch` | 延迟敏感服务开 | 启动时把堆页全部实占，防运行期 paging；代价是启动变慢、limit 内即时占满 |

## 4.3 Kubernetes 配置

```yaml
resources:
  requests:
    memory: "4Gi"
    cpu: "2"
  limits:
    memory: "4Gi"      # request = limit → Guaranteed QoS，内存必须相等
    cpu: "2"           # 内存相等防超卖驱逐；CPU 相等防 throttle（见 2.3/3.2）

env:
  - name: JAVA_TOOL_OPTIONS
    value: >
      -XX:MaxRAMPercentage=70
      -XX:+UseG1GC
      -XX:ActiveProcessorCount=2
```

### 4.3.1 探针参数与 JVM 节奏

```yaml
startupProbe:                    # 覆盖启动期：类加载 + JIT warmup 可能 30~120s
  httpGet: {path: /healthz, port: 8080}
  failureThreshold: 30
  periodSeconds: 5               # 最长容忍 150s，冷启动 JIT 风暴期不被误杀
livenessProbe:
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 10
  failureThreshold: 3            # 容忍 30s 无响应：足够覆盖一次大 Full GC，再长就该重启
readinessProbe:                  # 与 liveness 解耦：GC 抖动时先摘流量而不是急重启
  ...
```

- 判读：**liveness 连挂 ≈ "停顿已超阈值"** —— 要么调大容忍、要么按 22 第 4 节治 GC，而不是两边都调大装看不见。

### 4.3.2 优雅停机链

```text
kubectl delete pod
  ├─ ① EndpointSlice 摘除（异步，有延迟！）
  ├─ ② preStop hook: sleep 10        ← 给 ① 铺路，否则摘除前新流量还在进
  ├─ ③ SIGTERM → java(PID1) → ShutdownHook：
  │      停接新请求 → 等 in-flight 完成/任务收尾 → 关池（shutdown+awaitTermination）
  └─ ④ terminationGracePeriodSeconds(默认30，可调 60) 到 → SIGKILL（dump 都来不及写）
```

### 4.3.3 弹性与观测的坑

- **HPA 基于 CPU 会误判**：GC 风暴同样烧 CPU → 越烂越扩容；用 QPS/P99/自定义指标（如 Prometheus adapter）驱动 HPA，CPU 只做辅助。
- JVM 指标经 Micrometer/JMX exporter 类方案上报；告警至少含：GC pause P99、throttled periods、容器内存 working set/limit、OOMKilled 事件。
- 容器里 attach 类工具（Arthas/jcmd）：需同用户 + 共享 PID namespace（sidecar/`kubectl debug --target`），JDK 工具链在 JRE 镜像里可能缺失 → 运维镜像带 `jlink` 定制或至少含 jcmd。

# 5. JVM 性能里程碑（各版本重要更新）

| 版本 | JVM 关键更新 |
|------|-------------|
| JDK 5 | 泛型、枚举、自动装箱、J.U.C |
| JDK 6 | 锁优化（偏向锁、轻量级锁、自旋锁、锁消除）、CMS 增强 |
| JDK 7 | G1 引入、`invokedynamic`、逃逸分析改进、永久代中字符串常量池移到堆 |
| JDK 8 | 永久代移除→元空间、Lambda、分层编译默认开启、G1 正式版；8u131/8u191 容器感知两级火箭 |
| JDK 9 | G1 为默认 GC、模块化（Jigsaw）、字符串压缩（Compact Strings）、统一日志 `-Xlog` |
| JDK 10 | 并行 Full GC (G1)、容器感知增强（默认开启） |
| JDK 11 | ZGC 实验性、CMS 废弃、JFR 开源 |
| JDK 12 | Shenandoah 实验性、G1 可中断 Mixed GC |
| JDK 13 | 动态 CDS 归档（AppCDS） |
| JDK 14 | CMS 移除、ZGC 支持 macOS/Windows、JFR 事件流 |
| JDK 15 | ZGC 正式版、Shenandoah 正式版、偏向锁默认关闭；cgroup v2 支持起步 |
| JDK 17 LTS | 密封类、G1 NUMA 感知、ZGC 持续优化 |
| JDK 19-20 | 虚拟线程（Virtual Threads）预览 |
| JDK 21 LTS | 虚拟线程正式版（JEP 444）、分代 ZGC（实验性，JEP 439）、字符串模板 |
| JDK 23 | 分代 ZGC 成为默认（JEP 490） |
| JDK 24 | JEP 491：synchronized 不再钉住虚拟线程；移除非分代 ZGC |
| JDK 25 LTS | 紧凑对象头转正（JEP 519），小对象堆占用降 10%~20%（容器小内存实例更受益） |

# 6. 高频参数速查（场景 → 参数组）

| 场景 | 参数组 | 一句话理由 |
|------|--------|-----------|
| 常规 Web（2C4G 起） | `-XX:MaxRAMPercentage=60~75 -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=128m -XX:+UseG1GC` | 全组件封顶，留 8%~10% 余量 |
| 微小边车（≤1C） | 上一条基础上 `-XX:+UseSerialGC -Xss512k` | 并发 GC 在 1 核下是负收益 |
| 延迟敏感（≥4C） | 上一条基础上 `-XX:+UseZGC -XX:+AlwaysPreTouch -XX:ActiveProcessorCount=<整数核>` | JDK 23+ ZGC 已默认分代；21/22 需 `-XX:+ZGenerational` 显式开启 |
| Burstable（limits>requests） | `-XX:ActiveProcessorCount=<limits 核数> -XX:ParallelGCThreads=<N>` | JVM 感知以 limits 为准，防突发期线程超配 |
| Netty/Kafka 重客户端 | 追加 `-XX:MaxDirectMemorySize=<明确值>` | 默认≈Xmx，双份堆内存必爆 limit |
| 排障/审计期 | `-XX:NativeMemoryTracking=summary -Xlog:os+container` | 先自证"感知到了容器限制" |
| 云原生兜底 | `-XX:+ExitOnOutOfMemoryError -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data(pv)` | 死得快 + 现场留得住（dump 别写 tmpfs） |

# 7. JVM 高频面试考点

## 7.1 必问（基础）
1. **JVM 内存模型**：堆、栈、方法区/元空间、程序计数器。
2. **类加载过程**：加载→验证→准备→解析→初始化，双亲委派模型。
3. **对象创建过程**：分配内存→零值→对象头→init，对象内存布局。
4. **GC 判断方法**：引用计数法 vs 可达性分析，GC Roots。
5. **GC 算法**：标记-清除、标记-复制、标记-整理，三色标记。
6. **GC 收集器**：Serial、Parallel、CMS、G1 各自特点和适用场景。

## 7.2 必问（进阶）
7. **volatile 原理**：可见性、禁止重排序、内存屏障、happens-before。
8. **synchronized 优化**：偏向锁→轻量级锁→重量级锁，锁消除/粗化。
9. **CAS 与 ABA**：CAS 原理、ABA 问题及解决（版本号）。
10. **DCL 为什么加 volatile**：防止指令重排序导致的半初始化对象。
11. **OOM 排查**：各区域 OOM 的原因和排查方法。
12. **内存泄漏排查**：MAT 分析、ThreadLocal 泄漏、常见泄漏模式。

## 7.3 加分项（高级）
13. **ZGC/Shenandoah**：染色指针、读屏障、Brooks 前向指针。
14. **JIT 编译**：逃逸分析、方法内联、标量替换、分层编译。
15. **Arthas 实战**：dashboard、thread、trace、watch、profiler。
16. **容器化 JVM**：容器感知、OOMKilled 内存预算公式、MaxRAMPercentage、CPU throttle 与 GC 的相互作用。
17. **线上故障排查**：CPU 飙高、OOM、死锁、GC 频繁的完整排查链路。
18. **happens-before 规则**：8 条规则分别背诵。
19. **Class 文件结构**：魔数、版本号、常量池、access flag。
20. **对象大小计算**：Mark Word + Klass Pointer + 实例数据 + padding，压缩指针；JDK 25 紧凑对象头的取舍。

## 7.4 面试模拟

**Q: CMS 和 G1 区别？**
- CMS 是老年代收集器，G1 是全堆收集器。
- CMS 基于标记-清除，有碎片；G1 基于标记-复制+整理，无碎片。
- CMS 停顿目标 ~200ms，G1 可配置 `-XX:MaxGCPauseMillis`。
- G1 有 RSet 和 CSet，可预测停顿。
- CMS 已在 JDK 9 废弃，JDK 14 移除。

**Q: 什么时候触发 Full GC？**
- `System.gc()`
- 老年代空间不足
- 元空间空间不足
- CMS/G1 并发模式失败
- Minor GC 时分配担保失败

**Q: 如何排查 CPU 飙高？**
1. `top -c` → 找到高 CPU 进程 PID
2. `top -Hp <pid>` → 找到高 CPU 线程 TID
3. 转为十六进制 → `jstack <pid>` 定位线程栈
4. 或用 Arthas：`thread -n 5` / `profiler start`

**Q: 类加载器有哪些？双亲委派为什么被打破？**
- Bootstrap / Platform / Application
- 打破场景：SPI（JDBC 的 ServiceLoader）、Tomcat（多应用隔离）、OSGi（网状依赖）
- 通过重写 `loadClass()` 或使用线程上下文类加载器

**Q: 容器里 Java 进程为什么被 OOMKilled？（新增高频）**
- 答题骨架：堆 ≠ 进程；给出内存构成公式（堆+栈×N+元+CodeCache+Direct+GC 结构）→ 4Gi 演算 → MaxRAMPercentage/封顶参数/余量三件套 → 取证 NMT+pmap（→ 本文件 3.1）。

# 8. JVM 学习路线图

```text
第一阶段：JVM 基础（对照 JDK 21 学）
├── JVM 架构和运行时数据区
├── 类加载机制（双亲委派）
├── 对象内存布局（含紧凑对象头趋势）
└── 对象创建过程

第二阶段：GC 深入
├── 可达性分析
├── GC 算法（标记-清除/复制/整理、三色标记）
├── 经典收集器（Serial/Parallel/CMS/G1）
├── 低延迟收集器（ZGC/Shenandoah，分代 ZGC 是 JDK 23+ 默认）
└── GC 调优实战

第三阶段：字节码与执行
├── Class 文件结构
├── 字节码指令
├── 执行引擎（解释器/JIT）
└── JIT 编译优化

第四阶段：并发与内存模型
├── JMM 与 happens-before
├── volatile/synchronized 底层
├── 锁升级与优化
└── CAS 原理

第五阶段：故障排查（企业实战）
├── 命令行工具（jps/jstat/jmap/jstack/jcmd）
├── 可视化工具（VisualVM/JMC/Arthas/MAT）
├── OOM/CPU/死锁/GC 频率排查
├── 虚拟线程排障（pinning、JFR）
└── 容器化 JVM 调优（本文件）
```

# 9. 推荐资源

| 资源 | 说明 |
|------|------|
| 《深入理解 Java 虚拟机》第 3 版（周志明） | JVM 圣经，必读；JDK 8 时代细节注意甄别 |
| Optimizing Java (2nd ed., O'Reilly) | 性能工程方法论，覆盖 JDK 11+ 工具链 |
| 《Java 虚拟机规范》 | 官方规范 |
| JDK 源码/JEP 索引 | https://openjdk.org/jeps/0 —— 按 JEP 号读最权威演进史 |
| JDK 21/25 Release Notes | HotSpot 行为变更与参数废弃的"判决书" |
| GC 日志分析 | https://gceasy.io/ 、 https://gcplot.dev/ |
| Arthas 文档 | https://arthas.aliyun.com/ 场景案例极全 |
| async-profiler | https://github.com/async-profiler/async-profiler |
| 美团技术博客 JVM 系列 | 企业实战案例丰富 |
| Oracle JVM 调优指南 | 官方文档 |
| perfma 参数调优 | 懒人调参平台 |
| JOL (Java Object Layout) | 对象内存分析工具 |
