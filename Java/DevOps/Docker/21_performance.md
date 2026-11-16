
# 1. 先分清两类性能问题

Docker 性能通常分两类：

1. **构建性能**：`docker build` 慢，CI 慢。
2. **运行性能**：服务在容器里跑得慢，或抖动大。

两者优化手段完全不同，不要混在一起。再加一类常被归错类的：**冷启动/分发性能**（Pod 调度后拉镜像慢、扩容跟不上流量），它既不是 build 也不是运行时 CPU 问题，值得单独优化。所有结论背后的物理事实是：**Linux 容器没有虚拟化层，CPU 与内存路径≈原生**，容器场景的性能损耗集中在三处——存储驱动的文件叠加、iptables/NAT 的网络转发、以及 cgroup 调度与统计开销。找到这三处，就找到了所有容器性能问题的地图。

# 2. 构建性能优化

## 2.1 让缓存生效：Dockerfile 分层设计

原则：把"变化慢"的步骤放前面，"变化快"的步骤放后面。

常见例子（Java/Maven 思路）：

1. 先 COPY `pom.xml` 或 `build.gradle`。
2. 先下载依赖（可缓存）。
3. 再 COPY 源码并编译。

否则每次改一行代码都会导致依赖层缓存失效。

## 2.2 `.dockerignore` 减少上下文

构建上下文越大，上传与计算越慢——`docker build .` 第一步是把上下文目录**完整打包传给 daemon**，monorepo 里 .git 与 node_modules 会让这一步单独花费数十秒。

务必排除：

1. `.git/`
2. `target/`、`build/`
3. 本地缓存与临时文件
4. 密钥与配置

## 2.3 BuildKit 缓存复用

构建慢的主要原因往往不是 CPU，而是依赖下载。

策略：

1. CI 中启用 BuildKit。
2. 配置缓存导入/导出（跨 pipeline 复用，`type=registry,mode=max` 或本地 `type=local`）。
3. `RUN --mount=type=cache,target=/root/.m2` 让 Maven 仓库跨构建常驻（比层缓存更细粒度，且产物不落层）。

参考：`15_cicd.md`。

# 3. 存储驱动：overlay2 与选型

## 3.1 overlay2 的工作模型

Docker 在 Linux 上的唯一现实选择是 **overlay2**（基于内核 OverlayFS）：镜像只读层（lowerdir）叠加容器可写层（upperdir），合并视图（merged）由内核挂载。

```text
merged视图(容器看到的/)
┌────────────────────────────────────────────┐
│ upperdir(/var/lib/docker/.../diff)  可写    │ ← 修改与新建写这里
│ lowerdir1..N(各镜像层 diff, 反向序)  只读    │ ← 读穿透逐层查找
└────────────────────────────────────────────┘
  打开文件: 自顶向下查找第一个存在的层, 打开后即绕过 overlay 开销
  首次修改: 从只读层 copy-up 到 upperdir(整文件复制!), 之后改 upper
```

**性能结论全部从 copy-up 与"层数=查找深度"推出来**：

1. 读多写少的文件性能≈原生（打开后走底层 ext4 直接读）。
2. **大量小文件随机修改最伤**（每次修改先整文件复制上移），典型受害者：日志追加（单文件持续增长还好）、数据库类应用写容器层、频繁改配置的容器。
3. 层数过多拖慢查找与 `docker build`/镜像导入，建议运行层 ≤10 层左右，用多阶段构建收敛。
4. **删除大文件不会释放 lowerdir 空间**——在后续层 `RUN rm -rf` 前面层写的文件，镜像不会变小（whiteout 只是遮蔽），清理必须与产生放同一层（`8_dockerfile_best.md`）。

## 3.2 选型与相关文件系统

| 方案 | 现状 | 说明 |
|------|------|------|
| overlay2 + ext4 | **默认推荐** | 兼容性最好 |
| overlay2 + xfs(大 inode) | 大规模/海量小文件可用 | 需 `inode=512` 支持元数据大目录 |
| btrfs/zfs 驱动 | 特定场景 | 快照/克隆强, 但维护面小 |
| fuse-overlayfs | rootless 标配 | 用户态叠加, 性能低于内核 OverlayFS, rootless 的固有代价 |
| devicemapper/aufs | 淘汰 | 不要在 modern 环境使用 |

**设计考量**：

- **JVM 视角看 overlay**：应用启动的类加载是海量小文件**只读**路径，overlay 读穿透开销≈原生，不必焦虑；真正吃亏的是写路径——把日志写进容器可写层（copy-up + whiteout 放大）、把 H2/RocksDB/本地索引放容器层，都要迁到 volume。
- **性能问题先问"是不是 limit 设置的行为"**：throttle、回收、OOM 都属于"配额与负载不匹配"，不是 Docker 本身慢；换宿主机、删 limit 只会掩盖问题。

# 4. 运行性能：volume vs bind mount vs 容器层的 IO 差距

## 4.1 路径差异

| 写入路径 | 经过叠加层? | 相对性能 | 适用 |
|----------|------------|----------|------|
| 容器可写层(默认) | 是(copy-up) | 最差, 随机小写放大 | 临时文件, 别放 IO 密集数据 |
| named volume (local driver) | 否, 直接宿主目录 | ≈原生 | 数据库、队列 |
| bind mount | 否 | ≈原生 | 配置、共享数据 |
| tmpfs | 内存 | 读极快, 不持久 | /tmp、只读缓存 |

机制解释：volume/bind 挂载点**绕过 overlay 文件系统**，IO 直达宿主文件系统（ext4/xfs），没有 copy-up、没有 whiteout、没有查找穿透。MySQL/ES 这类随机 IO 大户跑在容器可写层上，性能可能差数倍且抖动——这是"**数据不进容器层**"纪律（`18_best_practices.md`）的物理学依据。

## 4.2 Windows/macOS 的额外税（Java 开发者高频遭遇）

Docker Desktop 里容器跑在 VM 中，**bind mount 跨 VM 边界**：macOS 的 virtiofs/gRPC-FUSE、Windows 的 9P/Docker Desktop WSL2 集成，随机 IO 比原生慢 5~50 倍；`node_modules`、Maven `target` 这类海量小文件目录是重灾区。实用处方：

1. 代码放 WSL2 文件系统内（Windows）或优化 `:delegated`/cached 挂载模式（macOS）。
2. 重 IO 路径改 named volume（数据不出 VM），只把源码 bind 进来。
3. Maven 构建产物目录（target）加 `.dockerignore` 避免宿主↔VM 双向同步。

> **注意**：容器内看到的"磁盘慢"先定位在哪一段：容器层→volume 换路径测一次；跨 OS bind→把同目录挂成 named volume 再测一次；宿主本身→`fio` 直接在宿主跑。三段各测一次，归因不会错。

# 5. CPU 与内存：记账、页缓存与 limit 抖动

## 5.1 内存记账的两个反直觉点

1. **页缓存计入 cgroup usage**。容器读 1GB 文件，`memory.current` 涨 1GB，但这部分可回收（工作集 working_set = usage - inactive_file），**内存 limit 偏小 + 重 IO 应用 = "内存明明没用满却 OOM"的常见根因**，内核回收慢于写入速度时 direct reclaim 还会引发延迟毛刺。
2. **tmpfs/内存盘同样计入容器内存**——`--tmpfs /tmp` 写满会 OOM，别把它当无限空间。

## 5.2 CPU throttle 与调度抖动

`--cpus` 的 CFS quota 是"每 100ms 周期用完额度就全体挂起"，Java 服务 GC 并发线程+应用线程同抢额度，会出现"quota 没用完却大量 throttling"（单线程打满一个核的瞬时也会触发 throttle 判定）。观测与处方在 `12_resource_limits.md`：`cpu.stat` 的 `nr_throttled/throttled_usec` 是必须上的监控项；延迟敏感服务考虑 cpuset 绑核 + 独占 CPU Manager（K8s `cpusetPolicy: static`）。

## 5.3 cgroup 本身开销有多大

直接回答：**系统调用路径上的 cgroup 检查是常数级微开销，对吞吐型服务可忽略**；`docker stats`/cAdvisor 的统计读取（周期性采样 cgroup 文件）才是可感成本（每容器约 0.5~2% CPU），高容器密度节点把采样间隔放宽即可。结论：性能优化不要从"去掉 cgroup"下手，要从 limit 值设置与 throttle 行为下手。

# 6. 启动与分发加速：拉镜像慢的解法阶梯

扩容速度 = 调度 + **拉镜像 + 解包** + 启动。Java 镜像动辄 300~500MB，冷节点拉取+解包常占 Pod Ready 时间的 80%。阶梯方案：

| 手段 | 原理 | 收益 |
|------|------|------|
| 镜像瘦身+分层复用(layertools) | 大依赖层跨版本共享, 命中本地缓存只拉 app 层 | 最划算的第一优先级 |
| 镜像预热(DaemonSet pull / imagePullPolicy 配合) | 新版本发布前把所有节点拉到镜像 | 扩容零等待, 部署流水线加一步 |
| Registry 就近复制/代理缓存(Harbor replication) | 跨地域拉取走本地副本 | 多机房场景 10x |
| **懒加载: stargazer/eStargz、Nydus、SOCI** | manifest 记录文件偏移, **边运行边按需取文件**, 不预下载全镜像 | 大镜像秒级启动 |
| 解包并行化/快照加速 | containerd 1.7+ 并行 unpack、本地 buildkit 快照 | 削解包时间 |

**eStargz 类懒加载机制**：

```text
传统: pull 完整镜像(500MB) → 全部解包 → 启动 JVM → 类加载(只碰到 20% 文件)
懒加载: 拉取 estargz 元数据(小) → 立即启动 → 类加载触发按需 HTTP Range 取文件 → 本地缓存
```

代价与边界（重要）：懒加载把"拉取一次"变成"运行期持续回源"，**网络抖动直接变成应用抖动**；生产上通常配"P2P 分发（kraken/Spegel）+ 启动后后台预热全层"混合模式，或仅对超大离线镜像启用。Nydus 用专用 fuse 后端、镜像转换需额外一步（`nydusify convert`），兼容性上要回归测试。

# 7. 网络运行性能

1. bridge 容器互访走网桥转发（L2 交换），**无 NAT**，性能≈宿主；经 `-p` 绕宿主机回环才吃 DNAT/conntrack，服务间调用永远走服务名直连（`7_network.md`）。
2. **conntrack 表是隐式炸弹**：高并发短连接（网关+重试风暴）会打满 `nf_conntrack_max`，症状是 `nf_conntrack: table full, dropping packet`（宿主机 dmesg 可见）而非应用报错——压测时延偶发数百 ms 先查它。
3. host 网络模式省掉 veth+bridge 路径，微基准可提升 10~30% p99，但放弃隔离，仅对极致延迟的中间件考虑；K8s 大规模场景的替代答案是 eBPF 数据平面（Cilium，见 `7_network.md`）。

# 8. 性能验证：压测方法论（fio 与分层基线）

优化前后至少要做两件事：

1. **对比基线**：同一负载下的 QPS/延迟/错误率。
2. **确认资源边界**：CPU/内存/IO 是否触顶。

容器场景压测要**分层归因**，每层先测宿主再测容器，差值即容器化成本：

```bash
# 存储 IO: 同一目录在宿主与容器(volume/可写层)分别跑 fio, 对比带宽与延迟分布
fio --name=randwrite --rw=randwrite --bs=4k --size=1G --numjobs=4 \
    --iodepth=32 --runtime=60 --time_based --group_reporting \
    --directory=/data/fiotest
# 关注: IOPS、p99 latency、带宽三项; 容器层跑同样的 fio 对比退化倍数

# CPU: 宿主与容器各跑 sysbench cpu --cpu-max-prime=20000 --threads=2
# 内存: sysbench memory; 网络: iperf3 两容器对打 / 容器↔宿主
```

> **行业实践**：压测环境必须**与生产同 cgroup 配置**（同样的 limit！不带 limit 压测等于测宿主机，数字虚高且毫无参考价值），并同步采集 `cpu.stat` 的 throttle、`memory.events` 的 oom 计数、fio 的延迟分布而非只记平均值——p99 毛刺才是限流阈值的真实来源。

# 9. 常见反模式

1. 不设资源限制：出现抖动时难定位根因。
2. 只看容器内指标，不看宿主机是否过载。
3. 数据写在容器可写层导致 IO 性能差。
4. 把构建慢归因于"Docker 慢"，但实际是上下文过大与缓存失效。
5. 跨 OS bind mount 跑海量小文件（Windows/macOS 开发机）。
6. 用平均值判断延迟健康（p99 被 throttle 打穿时平均值仍"正常"）。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 构建慢怎么优化 | Dockerfile 分层、.dockerignore、BuildKit 缓存/cache mount |
| 运行慢先看什么 | docker stats(注意 cache 计入内存)、cpu throttle、IO 路径 |
| 为什么 volume 比容器层快 | 绕过 overlay copy-up 与层查找, 直达宿主文件系统 |
| overlay2 的写放大 | 首次修改整文件 copy-up, 后续层删除不回收 lowerdir |
| 大镜像启动加速 | 瘦身+分层复用→预热→懒加载(eStargz/Nydus/SOCI)阶梯 |
| OOM 怎么排查 | inspect OOMKilled + working_set 口径 + 宿主机日志 + GC |

# 11. 小结

容器性能的心智模型一句话：**CPU/内存路径≈原生，损耗在存储叠加（overlay copy-up）、跨 VM/网络转发与 limit 引发的调度行为（CFS throttle）三处**。构建侧收益排序：`.dockerignore`→分层缓存→BuildKit registry cache/--mount=cache→layertools 分层复用；运行侧排查排序：先确认瓶颈层（宿主/容器层/volume/跨 OS bind），再看 throttle 与 working_set 两个口径，网络直连走服务名避开 NAT，conntrack 容量纳入压测观测项。冷启动用"瘦身→预热→懒加载"阶梯，懒加载要认清"启动快换运行期网络依赖"的取舍。最后把方法论钉死：**压测必须带生产同款 limit，指标看 p99 与 throttle 计数，不看平均值**。下一篇是面试问答汇总（`22_interview_qa.md`），本篇的表格适合直接当速查答案骨架。
