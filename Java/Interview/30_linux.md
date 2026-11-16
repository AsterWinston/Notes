本篇覆盖 Linux 运维与排障的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：系统模型与基础、性能观测与排查、命令与文本处理、权限服务与自动化、工程实践。JVM 层的 GC/堆分析见 2_jvm.md，连接数与线程池见 5_tomcat.md，容器隔离与 cgroup 见 23_docker.md，K8s 资源限制见 32_kubernetes.md。

# 1. 系统模型与基础

## 1.1 进程、线程与资源

### 1.1.1 进程与线程在内核里是什么？进程状态、僵尸与孤儿怎么理解？

**内核视角**：进程 = `task_struct`（调度实体）+ 独立的地址空间（`mm_struct`）+ 文件描述符表 + 信号处理等；**线程 = 共享同一地址空间的 task**（Linux 用 `clone()` 的 `CLONE_VM/CLONE_FILES` 等标志区分）——所以 Linux 内核其实不区分"进程/线程"，统一叫 task，**线程是"共享资源的 task"**（这也是 Java 线程映射到内核线程、`jstack` 能看到每个线程的原因）。

**进程状态（`ps` 的 STAT 列）**：

| 状态 | 含义 | 排查意义 |
|------|------|---------|
| **R**（Running/Runnable） | 在 CPU 上跑或在运行队列等 CPU | R 多 = CPU 竞争/负载高 |
| **S**（Sleeping） | 可中断睡眠（等事件/锁/socket） | 正常；大量 S 可能是等锁/等网络 |
| **D**（Disk Sleep） | **不可中断睡眠**（等 IO，内核态） | **D 多 = IO 瓶颈**；`kill -9` 也杀不掉（要等 IO 返回/重启） |
| **T**（Stopped） | 被 SIGSTOP/调试暂停 | 调试/挂起 |
| **Z**（Zombie） | 已退出但父进程**没 wait() 回收** | 僵尸多了会耗尽 PID；根因在父进程 |
| **I**（Idle） | 空闲内核线程 | 正常 |

**僵尸 vs 孤儿**：**僵尸**=子进程已死、父进程没调 `wait()` 收尸——`ps` 里 `<defunct>`，**不占内存/CPU 只占 PID 表项**；处理：让父进程正确 wait（应用 bug），或杀掉父进程让 init 收养；**孤儿**=父进程先死、子进程被 init/systemd 收养（现代 systemd 会回收）——孤儿本身不是问题，**僵尸才是**。**`kill -9` 杀不掉僵尸**（它已经死了），只能处理父进程。

**易错**：把"线程数多"当进程多（`ps -eLf`/`top -H` 才看线程）；D 状态进程用 `kill -9` 无效就以为"系统坏了"（先看 IO）；容器里 PID 1 不回收僵尸（23_docker.md 2.1.1 的 `--init`）。

### 1.1.2 文件描述符与 ulimit 是什么？为什么 Java 服务要调大 fd？

**fd 的本质**：进程级的**文件描述符表索引**（非负整数）——指向内核的"打开文件对象"（文件、socket、管道、epoll 实例都算）；`stdin/stdout/stderr` 固定占 0/1/2。**`ulimit -n`** 限制**单进程**能打开的数量（soft/hard 两档，soft 可调、hard 是天花板）；系统级还有 `fs.file-max`/`fs.nr_open`。

**Java 服务为什么吃 fd**：每个 TCP 连接=1 个 fd（Tomcat/Netty 的连接）、每个日志文件/打开的文件、`/proc` 访问、jar 包（启动时批量打开）、NIO 的 selector/epoll；**高并发 + 长连接**下几千上万 fd 是常态——`ulimit -n 1024` 会直接报 **`Too many open files`**（表现为新连接失败/日志写失败/莫名其妙的 IO 异常）。

**排查与配置**：`ulimit -n`（当前 shell）、`cat /proc/<pid>/limits`（进程实际限制）、`ls /proc/<pid>/fd | wc -l`（当前打开数）、`lsof -p <pid> | wc -l`；持久化：`/etc/security/limits.conf`（`* soft nofile 65535`）+ systemd 的 `LimitNOFILE`（**systemd 服务不读 limits.conf 的某些项，要改 unit 或 `DefaultLimitNOFILE`**）；容器要同时在 docker `--ulimit nofile`/K8s `securityContext` 里放开（23_docker.md 5.1.1）。

**易错**：只改 shell 的 `ulimit`（服务由 systemd 启动，不继承）；只看进程数不看 fd 数；`lsof | wc -l` 统计的是"所有进程"（要按 PID 过滤）；fd 泄漏（连接/流不关闭）先修代码，调大限制只是争取时间（5_tomcat.md 的连接账）。

## 1.2 内存与文件系统

### 1.2.1 虚拟内存、page cache、buffer/cache、swap 怎么理解？free 该看哪一列？

**内存账本（`free -h` 各列）**：

```text
total     物理内存总量
used      已用（不含可回收缓存）
free      完全空闲
shared    tmpfs/共享内存
buff/cache  buffer（块设备元数据）+ cache（文件页缓存 page cache）
available **应用真正可用的估算值** = free + 大部分可回收的 cache（重点看它）
```

**page cache 是 Linux 的性能核心**：读写文件都经过页缓存——读命中直接返回（不落盘）、写先进缓存由内核异步刷盘；**"free 很小但 available 很大"是健康状态**（缓存被有效利用，内存不够时内核自动回收）；**不要看 free 就以为内存不足**（这是最经典的误读）。

**buffer vs cache**：`buffers` 是块设备的元数据/裸块缓存（较小），`cached` 是文件页缓存（主要部分）；现代内核合并显示 `buff/cache`，不必强行区分。

**swap 的取舍**：swap 把不活跃页换到磁盘——**能缓解内存压力但延迟是磁盘级**；Java 服务通常建议**关闭/极小 swap**（GC 与堆被换出=灾难级延迟抖动）；看 `vmstat` 的 `si/so`（换入/换出速率，持续非 0=正在 swap）；`vm.swappiness` 调低（如 1）只是"尽量少换"，不是"不换"。

**OOM Killer**：内存真正耗尽时内核按 `oom_score`（与内存占用、`oom_score_adj` 相关）**杀进程**——`dmesg -T | grep -i oom` 看"谁被杀、为什么"；**被杀的不一定是"罪魁"**（可能是最大的无辜进程），所以容器/JVM 场景要把限额与堆配好（23_docker.md、2_jvm.md 的容器感知）。

**易错**：把 `used` 高当泄漏（缓存可回收）；`free` 小就加内存（先看 available 与 si/so）；swap 开了就以为"内存够了"（性能塌方）；容器里 `free` 看到的是宿主内存（cgroup 限额要去 `/sys/fs/cgroup/.../memory.max` 看）。

### 1.2.2 文件系统、inode、软硬链接是怎么回事？"磁盘满了但 du 不大"怎么解释？

**inode（索引节点）**：文件系统给每个文件分配的**元数据块**（权限、属主、大小、时间、数据块指针），**不包含文件名**——文件名存在**目录项**里（目录也是文件，内容是"名字 → inode 号"的映射）。**一个 inode 可以被多个目录项指向**，这就是硬链接。

| | 硬链接 | 软链接（符号链接） |
|--|--------|-------------------|
| 本质 | **同一 inode 的另一个名字**（目录项指向同一 inode） | **独立文件**，内容是"目标路径"字符串 |
| 跨文件系统 | ❌ | ✅ |
| 指向目录 | ❌（普通用户） | ✅ |
| 删除源文件 | 数据仍在（link count 减 1，归零才释放） | 链接变"断链" |
| `ls -l` | 普通文件样子（link count >1） | `lrwxrwxrwx ... -> target` |

**"df 满了但 du 不大"的四个原因（经典事故）**：
1. **已删除但被进程占用的文件**：`rm` 只是删目录项，**进程还开着 fd 就不释放数据块**——`lsof +L1` 或 `lsof | grep deleted` 找到，重启进程或 `: > /proc/<pid>/fd/<n>` 清空；
2. **inode 耗尽**：小文件（日志碎片/临时文件/`node_modules`）把 inode 用完——`df -i` 看 inode 使用率（磁盘有空间也写不进去）；
3. **挂载点被覆盖**：往"被挂载覆盖的目录"写数据（写到了根分区），`df` 与 `du` 对不上；
4. **稀疏文件/预留块/文件系统元数据**：`du` 统计实际块、`df` 统计文件系统视角，二者天然有差（还有 root 预留 5%）。

**排查顺序**：`df -h`（哪个分区满）→ `df -i`（inode 满没）→ `du -x -h --max-depth=1 / | sort -rh | head`（逐层下钻，`-x` 不跨文件系统）→ `lsof +L1`（deleted 文件）→ 日志/核心转储/临时目录。

**易错**：`du -sh /*` 跨挂载/权限报错（用 `-x` 与 `2>/dev/null`）；删了日志文件进程没重启（空间不释放）；把硬链接当"快捷方式"（硬链接不是副本，改一个另一个也变）；`/tmp` 的 sticky bit 与清理策略（4.1.1）。

# 2. 性能观测与排查

## 2.1 CPU 与负载

### 2.1.1 load average 三个数字怎么读？"负载高但 CPU 不高"是什么情况？

**定义**：`uptime`/`top` 的 load average = **在最近 1/5/15 分钟内，处于"可运行（R）"+"不可中断睡眠（D）"状态的 task 平均数**——注意它**不等于 CPU 使用率**，D 状态（等磁盘 IO/NFS/内核锁）也算负载。

**读法三步**：
1. **归一化**：负载值 ÷ 逻辑 CPU 数 = "每核压力"（8 核 load 8 ≈ 满载；load 4 ≈ 50%）——**不要拿绝对值直接说"高"**；
2. **看趋势**：1 分钟 > 5/15 分钟 = 正在上升（突发）；1 分钟 < 5/15 = 正在消化（恢复中）；三者都高 = 持续过载；
3. **结合 CPU 使用率**：load 高 + CPU 高 = 真在算；**load 高 + CPU 空闲 = 在等 IO/D 状态堆积**（磁盘慢/NFS 挂起/内核锁），这是"负载高但 CPU 不高"的标准答案。

**容器环境的口径**：容器内 `uptime` 默认读**宿主**的 load（cgroup 没有 load 指标）——"容器里 load 很高但应用没流量"可能是邻居干扰；要看容器自己的压力，得用 `cpu.stat` 的 throttling、`memory.current` 等 cgroup 指标（5.2.2、23_docker.md）。

**易错**：单核 load 1.0 不是"100% 危险线"（等于满载、无余量）；load 高就杀进程（先分清 R 还是 D）；把 "1 分钟负载" 当长期趋势；VM 的 steal（`st`）导致"看起来 CPU 空闲但应用慢"（宿主机超卖）。

### 2.1.2 CPU 使用率各字段怎么解读？上下文切换怎么看？

**`top`/`vmstat` 的 CPU 时间分解（按 `us sy ni wa hi si st` 记）**：

| 字段 | 含义 | 偏高说明 |
|------|------|---------|
| **us** | 用户态 CPU | 应用在算（死循环/正则/序列化/业务高峰） |
| **sy** | 内核态 CPU | **系统调用频繁**（大量 IO/网络/线程唤醒）、上下文切换多、锁竞争 |
| **ni** | nice 调整过的进程 | 低优先级任务 |
| **wa** | 等 IO 的 CPU 空闲 | **磁盘/网络慢**（CPU 在等，不是 CPU 瓶颈） |
| **hi/si** | 硬/软中断 | **si 高 = 网络包处理重**（小包风暴/中断未分流） |
| **st** | 被宿主机偷走的时间 | **虚拟机超卖**（云主机"CPU 明明空闲却很慢"） |

**上下文切换（`vmstat` 的 `cs`、`pidstat -w`）**：切换分**自愿**（等锁/IO 主动让出）与**非自愿**（时间片用完被抢占，说明 CPU 竞争）；`cs` 每秒几万是常态，**突增且伴随 sy 高**要看：线程数暴涨、锁竞争（大量 futex）、频繁短任务、日志/网络唤醒；`pidstat -w -p <pid> 1` 看单进程的切换来源。

**中断与软中断**：`vmstat` 的 `in`、`/proc/interrupts`、`/proc/softirqs`——网卡多队列没开/单队列打满时，`si` 飙高、某颗 CPU 100%（`top` 按 `1` 看每核），解法是 RSS/多队列/`irqbalance`（云上通常已托管）。

**答题收口**："**us 高查代码、sy 高查系统调用/切换、wa 高查 IO、si 高查网络中断、st 高找云厂商**"——一句话定位方向，再进具体工具。

### 2.1.3 线上 CPU 100% 的完整排查链路是什么？

**标准链路（Java 场景，八步）**：
```bash
① top                 # 找到 CPU 最高的进程 PID，并看是 us 还是 sy 高
② top -Hp <pid>       # 进程内线程排行（或 ps -mp <pid> -o THREAD,tid,time）
③ printf '%x\n' <tid> # 线程 ID 转十六进制（jstack 里的 nid）
④ jstack <pid> | grep -A 30 nid=0x<hex>   # 看该线程栈
⑤ 判型：RUNNABLE + 业务栈 = 死循环/正则回溯/大计算
        GC 线程（"GC task thread"/G1） = 频繁 GC（转 jstat，2.2.1）
        VM Thread/编译线程 = JIT/类加载异常
⑥ Arthas：thread -n 3（最忙线程栈）、trace 方法耗时、profiler 火焰图
⑦ 系统层：pidstat -u 1（进程）、perf top（内核/用户热点）、strace -p（系统调用）
⑧ 修复：代码/参数/容量，并在压测环境验证
```

**不用 jstack 的替代**：`Arthas thread -n`（最快）、async-profiler 生成火焰图（定位热点方法最直观）、`jcmd <pid> Thread.print`（等价 jstack）。

**六个常见根因**：**死循环/自旋**（RUNNABLE、栈固定）；**正则回溯**（用户输入触发 `.*` 嵌套）；**频繁 GC**（GC 线程高、堆快满，2_jvm.md）；**锁竞争**（大量 BLOCKED/WAITING + sy 高）；**日志/序列化大对象**（us 高、栈在 IO/JSON）；**线程数爆炸**（切换多、栈里大量同型线程）。

**易错**：只 top 不 top -H（看不到线程）；`jstack` 的 nid 是十六进制（忘了转换）；把 GC 线程高当"应用在算"（要看 `jstat -gcutil`）；容器里 `top` 看到宿主进程（`docker exec` 进容器才是容器视图）；**没保留现场就重启**（事后无法定位，5.2.1 的纪律）。

## 2.2 内存与 IO

### 2.2.1 内存泄漏/OOM 怎么排查？系统层与 JVM 层分别看什么？

**先分型（三种"内存问题"）**：
1. **真泄漏**：对象被长期引用（静态集合、缓存无上限、ThreadLocal、监听器未注销、类加载器泄漏）——堆/元空间**持续上涨且 Full GC 后不降**；
2. **容量不足**：堆配太小/流量涨了——Full GC 后能降但很快又满（加堆/加机器/优化对象）；
3. **堆外泄漏**：DirectByteBuffer、Metaspace、线程栈、JNI/本地库——**堆看着正常，进程 RSS 一直涨**（`jmap` 看不出来，要 NMT/pmap）。

**系统层**：`free -h`（available/swap）、`vmstat 1`（si/so）、`dmesg -T | grep -i oom`（OOM Killer 杀了谁）、`/proc/<pid>/status`（VmRSS/VmSwap）、`pmap -x <pid>`（内存段分布）、`smem`（按进程统计）。

**JVM 层（Java 排查主线）**：
```bash
jstat -gcutil <pid> 1000 10     # 看 O（老年代）/M（元空间）是否持续接近 100、FGC 增长
jmap -histo:live <pid> | head   # 活对象直方图（会触发 Full GC，慎用）
jmap -dump:format=b,file=heap.hprof <pid>   # 或 -XX:+HeapDumpOnOutOfMemoryError
# MAT/VisualVM：Leak Suspects → Dominator Tree（Retained Heap）→ GC Root 引用链
```
**两次 dump 间隔对比**（隔 10~30 分钟）找"只增不减"的类，比单次 dump 更能定位泄漏；**NMT**（`-XX:NativeMemoryTracking=summary` + `jcmd <pid> VM.native_memory`）定位堆外；**线程数**（`jstack | grep -c 'java.lang.Thread.State'`）排除"线程栈吃内存"。

**容器场景（必答）**：容器 OOM 是 **cgroup 限额**触发（不是宿主），进程被 SIGKILL → 退出码 137；`docker inspect` 看 `OOMKilled=true`（23_docker.md 5.1.1）；**堆 + 堆外 + 线程栈 + 元空间 < 容器限额**是硬约束。

**易错**：只看堆不看堆外（Netty 直接内存/元空间）；`jmap -histo` 不加 `:live`（包含垃圾，误判）；dump 文件太大把磁盘写满；生产直接 `kill -3` 拿线程栈但没拿堆（信息不全）；MAT 只看 Histogram 不看引用链（不知道"谁持有"）。

### 2.2.2 磁盘 IO 瓶颈怎么定位？iostat 的哪些指标最有用？

**工具分工**：`iostat -x 1`（**设备级**：利用率与排队）、`pidstat -d 1`（**进程级**：谁在读写在等）、`iotop`（实时按进程）、`vmstat 1`（`bi/bo` 全局块 IO）、`fio`（压测基线）、`strace -e trace=file,read,write`（单进程系统调用）。

**`iostat -x` 的关键列（新版 sysstat 口径）**：

| 列 | 含义 | 判读 |
|----|------|------|
| `r/s` `w/s` | 每秒读/写次数（IOPS） | 结合设备能力看（SSD vs HDD） |
| `rkB/s` `wkB/s` | 吞吐 | 顺序大 IO 看它 |
| `await` | **平均 IO 响应时间（含排队）**，ms | 明显高于设备基线 = 排队严重 |
| `aqu-sz` | 平均队列长度 | 持续 >1 说明在排队（旧版 `avgqu-sz`） |
| `%util` | 设备忙碌时间占比 | 接近 100% = 饱和（但 **NVMe/多队列下 %util 会失真**，看 await/aqu-sz 更准） |

**判型**：**IOPS 打满**（小文件随机 IO）→ 应用层合并/批量、换 SSD、加缓存；**吞吐打满**（大文件顺序）→ 带宽/RAID/网络存储上限；**await 高但 %util 不高**→ 存储后端慢（云盘限流/NFS/RAID 卡缓存）或队列调度问题；**`wa` 高**（2.1.2）配合 iostat 定位设备。

**Java 常见源头**：日志同步刷盘（`fsync` 频繁）、大文件上传下载、数据库连接池慢查询、`logback` 同步 appender、临时文件写满 page cache 后集中回写（`dirty` 比例高，`/proc/meminfo` 的 `Dirty`）。

**易错**：`svctm` 已废弃/不可信（别再引用）；只看 `%util` 不看 `await`（云盘"100% util 但吞吐很低"是限流）；在容器里 `iostat` 看到的是宿主设备（要结合 cgroup `io.stat`）；把"IO 高"当"磁盘要坏"（先看业务写入模式）。

### 2.2.3 磁盘空间满怎么排查？"df 满了但 du 不大"和 inode 满怎么处理？

**四步排查（与 1.2.2 的原理对应）**：
```bash
df -h                                  # ① 哪个挂载点满
df -i                                  # ② inode 是否耗尽（小文件多）
du -x -h --max-depth=1 / | sort -rh | head    # ③ 逐层下钻（-x 不跨文件系统）
lsof +L1                               # ④ 已删除但被进程占用的文件
```
**逐层定位**：`du --max-depth=1 /var | sort -rh` → 下一层继续，直到具体目录；常见大户：应用日志（未轮转）、`journalctl`（systemd journal）、容器层与镜像（`docker system df`，23_docker.md 5.2.1）、core dump、`/tmp`、上传文件、数据库 binlog（16_mysql.md 4.1.2）。

**deleted 文件占空间**：`lsof | grep deleted` 找到持有 fd 的进程——**重启进程**（释放）或**清空 fd**（`: > /proc/<pid>/fd/<n>`，仅对可写文件有效）；根因是"日志清理脚本 rm 了但进程没重开文件"（logrotate 要用 `copytruncate` 或发信号让应用重开，4.2.2）。

**inode 满**：`df -i` 看 `IUse%`；`find /path -xdev -type f | wc -l` 按目录统计文件数，找"大量小文件"（会话文件/临时文件/邮件队列/`node_modules`）；清理或合并打包；**扩容 inode 只能重建文件系统**（格式化时定），所以"小文件海量"场景要在架构层避免。

**预防**：logrotate（按天/大小轮转 + 压缩 + 保留 N 天）、journald 的 `SystemMaxUse`、监控磁盘与 inode 双水位（80% 告警）、清理任务用 `find -mtime +N -delete` 而**不是** `rm -rf` 大目录（5.1.2）。

**易错**：`du` 与 `df` 对不上就以为"文件系统坏了"（先查 deleted/inode/挂载覆盖）；`rm` 大日志后空间没降（进程持有）；清理 `/var/lib/docker` 直接删目录（用 `docker system prune`，23_docker.md）；忘记 `/var/log/journal` 能占几十 GB。

## 2.3 网络

### 2.3.1 网络连通性怎么排查？ping、telnet、nc、curl、ss 各管哪一层？

**按层排查（这是本题的骨架，工具只是各层的探针）**：

| 层 | 问题 | 工具 |
|----|------|------|
| DNS | 域名解析不了/解析错 | `dig`/`nslookup`/`getent hosts`；`curl -v` 看解析结果 |
| 路由/连通 | 到不了对端 | `ping`（ICMP）、`traceroute`/`mtr`（路径与丢包） |
| 端口可达 | 端口没开/防火墙拦 | `nc -zv host port`、`telnet host port`、`/dev/tcp` |
| 服务协议 | HTTP 返回什么/超时 | `curl -v`（请求头/响应头/TLS）、`curl -w` 分段计时 |
| 本机监听 | 谁在监听/连接状态 | `ss -tlnp`、`lsof -i :port` |
| 抓包取证 | 到底发了什么/回了什么 | `tcpdump`（2.3.3） |

**四个易错点**：① **`ping` 通不代表端口通**（ICMP 放行 ≠ TCP 放行；很多云主机禁 ping）；② **`telnet` 连上不代表服务正常**（只证明三次握手成功，应用可能 500）；③ **`curl` 默认走代理/HTTPS 证书校验**（`--noproxy '*'`、`-k` 要谨慎）；④ **`ss -tlnp` 看不到进程**（非 root 看不到别人的 PID；`-p` 需要权限）。

**标准组合拳（服务"访问不了"）**：`dig` 确认 IP → `ping`/`mtr` 确认网络 → `nc -zv` 确认端口 → `curl -v` 确认应用响应 → 不通再看**本机 `ss -tlnp` 是否监听在 0.0.0.0/指定网卡**（监听 127.0.0.1 则外部不可达——23_docker.md 的端口发布同款）→ 防火墙/安全组（`iptables -L`、云安全组）。

### 2.3.2 TIME_WAIT 和 CLOSE_WAIT 过多分别说明什么？怎么处理？

**先看 TCP 状态机**：主动关闭方发 FIN → 进入 **FIN_WAIT** → 收到对方 FIN 后回 ACK → **TIME_WAIT**（等待 **2MSL**，Linux 约 60 秒）→ 关闭；被动关闭方收 FIN → **CLOSE_WAIT**（等应用调 `close()`）→ 发 FIN → LAST_ACK → 关闭。

**TIME_WAIT 多（正常现象，不是 bug）**：
- **谁主动关闭谁持有**：短连接高频（HTTP 短连接、DB 短连接）时，**主动关闭方**堆积 TIME_WAIT；
- **为什么需要**：保证最后一个 ACK 能重传（对端没收到的 FIN 可重发）、让旧连接的迷路包过期（防止污染新连接）；
- **处理**：**连接池/长连接**（治本：HTTP keep-alive、DB 连接池、Nginx upstream keepalive）；客户端可用 `net.ipv4.tcp_tw_reuse=1`（**只对主动发起的新连接有效**，需 timestamps）；`tcp_max_tw_buckets` 调大只是缓兵；**`tcp_tw_recycle` 已被移除**（NAT 环境下害人，别背旧参数）；端口耗尽时扩源端口范围（`ip_local_port_range`）。

**CLOSE_WAIT 多（几乎总是应用 bug）**：对端已发 FIN（连接已半关），**本端应用没有 `close()`**——常见于：流/连接未关闭（`InputStream`/`Socket`/`HttpClient` 响应未消费完）、线程阻塞导致 close 没执行、连接池泄漏。**处理**：`ss -tan state close-wait` 找到连接 → `lsof -p <pid>`/`ss -tanp` 找到进程 → 修代码（try-with-resources、超时、连接池归还）；**调内核参数没用**（CLOSE_WAIT 不归内核管）。

**其他状态速查**：`SYN_RECV` 大量（SYN flood 或 backlog 太小）、`ESTABLISHED` 过多（连接泄漏/并发高）、`FIN_WAIT2` 堆积（对端没关，`tcp_fin_timeout`）。

**答题收口**："**TIME_WAIT 是'礼貌的等待'（协议要求），CLOSE_WAIT 是'忘了关门'（代码问题）**——前者用连接复用治，后者用资源关闭治，别互相甩锅给内核参数。"

### 2.3.3 tcpdump 怎么抓包？select/poll/epoll 的区别是什么？

**tcpdump 常用姿势**：
```bash
tcpdump -i any -nn host 10.0.0.5 and port 8080 -w cap.pcap   # 抓指定主机+端口，存文件
tcpdump -i eth0 -nn -A 'tcp port 80 and (((ip[2:2]-((ip[0]&0xf)<<2))-((tcp[12]&0xf0)>>2)) != 0)'
tcpdump -i any -s0 -c 100 -X port 3306      # 不截断、抓100个、十六进制+ASCII
```
**过滤表达式（BPF）**：`host/src/dst`、`port/portrange`、`net`、`tcp/udp/icmp`、`and/or/not`、`tcp[tcpflags] & tcp-syn != 0`；**先想清楚"抓哪条流"**（否则全量包会拖垮机器/磁盘）。**注意**：需要 root/CAP_NET_RAW；**TLS/HTTP2 抓到的内容加密/二进制**（要密钥或看握手/时序）；生产抓包限时长/限大小（`-c`/`-G`/`-W` 轮转）。

**select/poll/epoll（IO 多路复用的演进，面试必考）**：

| | select | poll | epoll |
|--|--------|------|-------|
| 数据结构 | `fd_set` 位图 | `pollfd` 数组 | 内核事件表（红黑树 + 就绪链表） |
| fd 上限 | **1024**（`FD_SETSIZE`） | 无硬上限（数组大小） | 无（受系统限制） |
| 每次调用 | **把 fd 集合从用户态拷到内核 + O(n) 扫描** | 同（O(n)） | `epoll_ctl` 注册一次，`epoll_wait` 只返回就绪的（**O(就绪数)**） |
| 触发模式 | 水平触发 | 水平触发 | **水平/边缘触发（LT/ET）** |
| 适用 | 少量连接 | 少量~中量 | **高并发大量连接**（Nginx/Netty/Redis） |

**关键认知**：**epoll 快不是因为"用了什么黑科技"，而是"避免每次全量扫描 + 只在就绪时通知"**；ET 模式要求应用**一次把数据读干净**（否则不再通知）；**Java NIO 的 `Selector` 在 Linux 上就是 epoll**（`EPollSelectorImpl`），Netty 的 EventLoop 建立在它之上（1_java.md 的 NIO 章节、5_tomcat.md 的 NIO 连接器同源）。

**易错**：以为 epoll 一定比 select 快（连接少时差异不大）；ET 模式没读完数据导致"连接卡死"；抓包看到 RST 就怪网络（可能是应用关闭/端口未监听）；把 `tcpdump` 的"包丢失"（内核丢包计数）忽略（高流量下抓不全，要看 `-w` 与缓冲区）。

# 3. 命令与文本处理

## 3.1 日志分析三剑客

### 3.1.1 grep 的高阶用法有哪些？

**基础到高阶（按使用频率）**：
```bash
grep -rn "Exception" /app/logs --include="*.log"     # 递归+行号+文件过滤
grep -v "^#" nginx.conf | grep -v "^$"               # 去注释与空行（看有效配置）
grep -A 3 -B 2 "ERROR" app.log                       # 上下文（After/Before/Context）
grep -c "ERROR" app.log                              # 计数（行数，不是出现次数）
grep -o "orderId=[0-9]*" app.log | sort -u           # 只输出匹配部分（提字段）
grep -E "ERROR|WARN" app.log                         # 扩展正则（ERE）
grep -F "a.b.c" file                                 # 固定字符串（不当正则）
zgrep "ERROR" app.log.gz                             # 直接搜压缩日志
```

**三个"专业细节"**：① **退出码**（0=有匹配、1=无匹配、2=出错）——脚本里 `if grep -q ...` 就是靠它；② **`-P`（PCRE）** 支持 `\d`/环视，但并非所有环境编译了 PCRE（可移植性差，优先 `-E`）；③ **二进制文件**默认提示 "Binary file matches"（用 `-a` 强制按文本）。

**易错**：`grep -v "a" | grep -v "b"` 是"排除 a **且** 排除 b"（不是"排除 a 或 b"）；`grep -c` 数的是**行**（一行两次匹配算 1）；正则里的 `.`/`*`/`[]` 没转义（用 `-F` 或转义）；大文件 grep 慢用 `LC_ALL=C` 提速（字节匹配）。

### 3.1.2 awk 怎么按列处理与分组统计？

**核心模型**：`awk 'pattern { action }' file`——按行读入，自动按 `FS`（默认空白）切分成 `$1..$NF`，`$0` 是整行；`NR` 行号、`NF` 字段数、`FNR` 当前文件行号；`BEGIN/END` 做前置/收尾。

```bash
# 按列求和（第 3 列）
awk '{sum += $3} END {print "total:", sum}' data.txt

# 分组计数（第 1 列出现次数），再按次数排序
awk '{cnt[$1]++} END {for (k in cnt) print k, cnt[k]}' access.log | sort -k2 -nr | head

# 指定分隔符 + 条件 + 格式化
awk -F',' '$5 > 1000 {printf "%s\t%.2f\n", $1, $5}' sales.csv

# 统计接口平均耗时（假设第 4 列为耗时，第 7 列为 URL）
awk '{sum[$7]+=$4; n[$7]++} END {for (u in sum) printf "%s %.1fms\n", u, sum[u]/n[u]}' app.log
```

**四个要点**：① **关联数组无序**——要排序必须管道给 `sort`（`asort` 只有 gawk 有）；② **`-v` 传外部变量**（`awk -v limit=100 '$3>limit'`）；③ **`NF` 是字段数**（`$NF` 是最后一个字段，日志字段数不固定时取"末尾字段"很好用）；④ **性能**：awk 是流式单遍，比"多次 grep/sed"快，日志统计首选。

**易错**：`-F'\t'` 写法（制表符要引号/`$'\t'`）；默认 FS 把连续空白当一个分隔符（CSV 用 `-F,` 要小心字段内逗号）；`$0` 与 `$1` 混用；浮点格式化用 `printf`（默认输出精度会丢）。

### 3.1.3 sed 的替换、删除与行操作怎么用？

```bash
sed -i 's/old/new/g' file                 # 全局替换（-i 原地；macOS 要 -i ''）
sed -i.bak 's/old/new/' file              # 保留备份（file.bak）
sed -n '50,100p' file                     # 只看 50~100 行（-n 抑制默认输出）
sed '/^#/d; /^$/d' file                   # 删注释行与空行
sed '3a\inserted line' file               # 第 3 行后追加
sed '2i\inserted line' file               # 第 2 行前插入
sed 's/\([0-9]\+\)-\([0-9]\+\)/\2-\1/'    # 分组反向引用（ERE 用 -E）
```

**四个坑**：① **`-i` 的跨平台差异**（GNU 可省略后缀，BSD/macOS 必须给 `''`）；② **正则贪婪**（`s/<.*>/X/` 会吃掉一整行，用 `[^>]*` 或非贪婪扩展）；③ **特殊字符**（替换串里的 `/`、`&`、`\` 要转义；可用其他分隔符 `s#a/b#c/d#g`）；④ **原地修改前先备份/先 `-n p` 预览**（生产配置改错是事故）。

**选型**：简单替换/删行用 sed；**按字段**处理用 awk；**要"匹配后做复杂逻辑"**用 awk 或脚本；**大文件原地改**注意 IO 与原子性（先写临时文件再 mv）。

## 3.2 查找与统计

### 3.2.1 find 与 xargs 的正确姿势？有哪些坑？

```bash
find /data -type f -name "*.log" -mtime +7 -delete        # 7 天前的日志删除（先 -print 预览！）
find /data -type f -size +100M -exec ls -lh {} \;         # 找大文件并查看
find /data -type f -name "*.log" -print0 | xargs -0 -n 100 -P 4 gzip
find /data -newermt "2026-01-01" -type f -name "*.conf"   # 按时间（可读格式）
find /data -type d -perm -g+w                             # 组可写的目录（安全审计）
```

**`-exec ... \;` vs `-exec ... +`**：前者**每个文件起一个进程**（慢），后者**批量传参**（快，类似 xargs）；**文件名有空格/换行**时用 `-print0 | xargs -0`（否则会被拆错——"删错文件"的头号来源）。

**xargs 常用**：`-n` 每次传几个参数、`-P` 并行度、`-I{}` 占位符（`xargs -I{} cp {} /backup`）、`-r` 无输入不执行（GNU）。

**五条安全纪律**：① **删除/修改前先 `-print` 预览**（`find ... -print` 代替 `-delete` 跑一遍）；② **限定 `-maxdepth`/路径**（别从 `/` 开始）；③ **排除挂载点**（`-xdev`）；④ **防并发**（定时任务用 `flock`，4.2.2）；⑤ **`-delete` 的隐含条件**（find 的 `-delete` 会自动加 `-depth`，且**只删空目录除非配合 `-type f`**）。

**易错**：`find . -name *.log`（shell 先展开 `*.log`，find 收到的是文件名——必须加引号）；`xargs` 不处理空格（必须 `-0`）；`-mtime +7` 的语义是"**大于 7 天**"（不是"7 天内"，那是 `-mtime -7`）；`-exec rm` 与 `-delete` 混用导致重复。

### 3.2.2 sort、uniq、wc、head、tail 怎么组合出"TopN 统计"？

**经典四段式（访问日志 Top10 IP）**：

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
# ① 提取 IP  ② 排序（让相同值相邻）③ 计数  ④ 按次数降序  ⑤ 取前 10
```

**为什么必须 `sort` 在 `uniq` 前**：`uniq` 只合并**相邻**重复行（流式、省内存）——不排序直接 uniq 会得到"分段计数"，这是最经典的错误。

**各命令的关键参数**：`sort -t',' -k3 -n -r`（分隔符/第 3 列/数值/逆序）、`sort -u`（排序去重）、`sort -h`（人类可读大小）；`uniq -c`（计数）、`-d`（只看重复）、`-u`（只看唯一）；`wc -l/-w/-c`；`head -n`/`tail -n`；**`tail -f` vs `-F`**（`-f` 按 fd 追，日志轮转后失效；`-F` 按文件名重开，**生产用 `-F`**）。

**变体**：统计状态码分布 `awk '{print $9}' access.log | sort | uniq -c | sort -rn`；统计每分钟 QPS `awk '{print substr($4,2,17)}' access.log | uniq -c`（日志时间有序时可直接 uniq）；大文件用 `zcat *.gz | ...` 或 `rg`/`awk` 流式处理。

**易错**：`sort | uniq -c | sort -rn` 最后忘了 `head`（输出几百行）；`uniq -c` 后字段是"次数 + 值"，`sort -rn` 默认按整行（以次数开头，数值排序正确）；`awk` 取错列（不同日志格式列号不同，先 `head -1` 看格式）；`tail -f` 跟轮转日志丢行（用 `-F` 或采集器）。

# 4. 权限、服务与自动化

## 4.1 权限与安全

### 4.1.1 rwx 权限怎么算？SUID、SGID、Sticky 各解决什么问题？

**基础位（三组 × 三位）**：`rwx` = 4/2/1，三组分别是**属主 u / 属组 g / 其他 o**——`chmod 755 file` = `rwxr-xr-x`；**目录的权限语义不同**：`r` 可列目录、`w` 可在目录内创建/删除文件、`x` 可进入/穿过目录（**只给 r 不给 x，能看到名字但进不去**）。

**umask**：新建文件的默认权限 = `666 & ~umask`、目录 = `777 & ~umask`——`umask 022` → 文件 644、目录 755；**umask 是"减权限"的掩码**（不是直接设置权限）。

**三个特殊位（必须能讲出"为什么需要"）**：

| 位 | 作用对象 | 效果 | 典型 |
|----|---------|------|------|
| **SUID（4xxx）** | 可执行文件 | 执行时**以文件属主身份**运行（而非执行者） | `/usr/bin/passwd`（普通用户改密码要写 `/etc/shadow`，属主 root） |
| **SGID（2xxx）** | 文件/目录 | 文件：以属组身份运行；**目录：新建文件继承目录的属组** | 团队共享目录（成员创建的文件自动归项目组） |
| **Sticky（1xxx）** | 目录 | 目录内文件**只有属主/root 能删** | `/tmp`（人人可写但互不删除） |

**表示法**：`ls -l` 的 `s`（SUID/SGID，**位置上有 x 显示 s、没有 x 显示 S**）、`t/T`（Sticky）；`chmod u+s`、`chmod g+s`、`chmod +t`；`find / -perm -4000 -type f` 可审计全系统 SUID 文件（提权面）。

**现代替代：capabilities**——SUID 是"全有或全无"的 root 特权，`capabilities` 把特权拆细（如让 Java 绑定 1024 以下端口只需 `CAP_NET_BIND_SERVICE`）：`getcap -r /`、`setcap cap_net_bind_service+ep /path/java`；容器安全里的 `--cap-drop ALL` 就是同一机制（23_docker.md 5.2.2）。**ACL**（`setfacl -m u:alice:rw file`）用于"三位不够用"的精细授权。

**易错**：把目录 `x` 当"执行目录"（是"进入/遍历"）；SUID 对脚本无效（内核忽略脚本的 SUID）；`chmod 777` 解决一切（安全灾难，按需授权）；Sticky 只防"删除"不防"读取/修改内容"（文件自身权限仍生效）。

### 4.1.2 sudo 与 SSH 免密怎么做才安全？

**sudo 的正确姿势**：用 **`visudo`** 编辑（有语法检查，写错会锁死 sudo）；格式 `user host=(run_as_user:run_as_group) [NOPASSWD:] command`；**最小权限**——只授权需要的命令（`/usr/bin/systemctl restart myapp`），**别给 `NOPASSWD: ALL`**（等于免密 root）；`sudo -l` 查看当前用户能做什么；审计 `/var/log/auth.log`/`journalctl -u sudo`。

**SSH 免密的四步**：`ssh-keygen -t ed25519`（生成密钥对）→ `ssh-copy-id user@host`（公钥写入远端 `~/.ssh/authorized_keys`）→ 校验权限（**`~/.ssh` 700、`authorized_keys` 600**，权限过宽 sshd 会拒绝）→ `~/.ssh/config` 配别名/跳板（`Host`/`HostName`/`User`/`IdentityFile`/`ProxyJump`）。

**生产加固清单**：`PasswordAuthentication no`（只允许密钥）、`PermitRootLogin no`（禁止 root 直登）、改端口/限制来源 IP（安全组/`AllowUsers`）、`fail2ban` 防爆破、**跳板机/堡垒机**统一入口与审计、密钥加密（passphrase）+ `ssh-agent`、定期轮换离职人员密钥。**`scp` vs `rsync`**：scp 简单（`-r` 递归），rsync **增量同步**（`-avz --progress`），**`--delete` 会删目标端多余文件**（生产先 `--dry-run` 预览）。

**易错**：把私钥拷到服务器（私钥永不外传）；`authorized_keys` 权限 644 导致"密钥登录无效"（sshd 严格检查）；`sudo` 配了 `NOPASSWD` 但命令用相对路径（可被 PATH 劫持提权）；`rsync --delete` 方向写反（源/目标颠倒，数据被清）。

## 4.2 服务与调度

### 4.2.1 systemd 与 service 有什么区别？journalctl 怎么用？

**演进**：SysV init（`/etc/init.d` 脚本 + `service` 命令）是串行启动、脚本各自实现；**systemd** 用**声明式 unit** 管理（并行启动、依赖关系、自动重启、日志统一到 journal、cgroup 追踪），是现代发行版标准（`service` 命令通常只是转发到 systemctl）。

**一个 Java 服务的 unit 示例**：
```ini
[Unit]
Description=order-service
After=network.target

[Service]
User=app
ExecStart=/usr/bin/java -Xmx2g -jar /opt/app/order.jar
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment=SPRING_PROFILES_ACTIVE=prod

[Install]
WantedBy=multi-user.target
```
**命令**：`systemctl start/stop/restart/status`、`enable --now`（开机自启+立即启动）、改 unit 后 `daemon-reload`（**不 reload 不生效**）、`systemctl list-units --type=service`、`systemctl cat`（看生效的 unit）。

**journalctl（日志）**：`journalctl -u order-service -f`（跟踪）、`--since "10 min ago"`、`-p err`（级别）、`-o json`（结构化）、`--disk-usage`/`--vacuum-size=1G`（清理）；**持久化**（`/var/log/journal` 存在才跨重启保留，否则只在 `/run` 内存里）。**应用日志仍建议独立文件/采集**（journald 的二进制格式不便业务分析，30_linux 的日志治理与 24_elastic_search.md 的采集链路配合）。

**易错**：改了 unit 不 `daemon-reload`（改了白改）；`ExecStart` 用相对路径/依赖 shell 环境（systemd 不读 `.bashrc`，要绝对路径/`Environment`）；`Restart=always` 配无退避（崩溃循环刷日志）；把 `journalctl` 当业务日志检索（二进制 + 大小限制）。

### 4.2.2 crontab 的格式、坑与生产脚本怎么写？

**格式**：`分 时 日 月 周 命令`（`*` 任意、`*/5` 每 5、`1-5` 范围、`1,3,5` 列表）；`0 2 * * *` 每天 2 点；`@reboot`/`@daily` 是快捷别名。**用户 crontab**（`crontab -e`）与 **`/etc/cron.d/*`**（多一个"用户"字段）不同；`cron.d` 更适合配置管理。

**五个坑（生产事故高频）**：
1. **环境变量不继承**：cron 不加载 `.bashrc`/`/etc/profile`——脚本里**用绝对路径**、显式 `source` 环境、设置 `PATH`/`JAVA_HOME`；
2. **工作目录不确定**：脚本里 `cd` 到目标目录或全用绝对路径；
3. **输出丢失**：重定向 `>> /var/log/task.log 2>&1`（否则 cron 发邮件/丢弃）；`MAILTO=""` 关掉邮件；
4. **任务重叠**：上一次没跑完下一次又起——用 `flock -n /tmp/task.lock` 防并发；
5. **时区/夏令时**：容器/云主机时区是 UTC，`0 2 * * *` 可能不是你想要的 2 点（`timedatectl` 确认）。

**可靠脚本模板**：
```bash
#!/usr/bin/env bash
set -euo pipefail
exec 9>/tmp/clean.lock; flock -n 9 || exit 0        # 防重叠
LOG=/var/log/clean.log
echo "[$(date -Is)] start" >> "$LOG"
find /data/logs -type f -name "*.log.gz" -mtime +30 -delete
echo "[$(date -Is)] done" >> "$LOG"
```
**替代方案**：**systemd timer**（`OnCalendar=*-*-* 02:00:00`、`Persistent=true` 补跑错过的任务、依赖/日志统一管理）——新系统优先 timer，crond 适合简单任务/存量环境。

**易错**：`*/5` 与 `0/5` 混淆；周字段 0 和 7 都是周日；`%` 在 crontab 里有特殊含义（要转义 `\%`）；删了 crontab 没备份（用配置管理/版本库）；脚本里有交互命令（cron 环境无 TTY，会挂住）。

# 5. 工程实践

## 5.1 脚本与运维

### 5.1.1 服务健康检查与自愈脚本怎么写才可靠？

**先分清三种"不健康"**：**进程没了**（重启）、**进程在但假死**（健康检查失败→重启）、**下游依赖挂了**（重启应用没用，会引发重启风暴）——自愈脚本必须能区分。

**一个可靠的健康检查脚本**：
```bash
#!/usr/bin/env bash
set -euo pipefail
URL=http://127.0.0.1:8080/actuator/health
LOCK=/tmp/order-health.lock
exec 9>"$LOCK"; flock -n 9 || exit 0          # 防并发重启
if curl -fsS --max-time 3 -o /dev/null "$URL"; then
  exit 0
fi
# 连续 3 次失败才重启（防抖）
for i in 1 2 3; do sleep 5; curl -fsS --max-time 3 -o /dev/null "$URL" && exit 0; done
logger -t health "order-service unhealthy, restarting"
systemctl restart order-service                # 交给 systemd（有 Restart 策略更好）
```
**六条设计要点**：**超时**（不能让检查自己挂住）、**重试/防抖**（避免网络抖动误判）、**防并发**（flock）、**重启有上限/退避**（连续重启 N 次告警，而不是无限循环）、**日志与告警**（`logger`/监控系统）、**区分依赖故障**（健康检查里别把"DB 慢"算成"应用该死"——liveness 与 readiness 分开，32_kubernetes.md）。

**更好的姿势**：**systemd `Restart=on-failure` + `WatchdogSec`**（应用定期 `sd_notify`，超时 systemd 重启）比 crontab 轮询更及时；K8s 用 **liveness/readiness/startup 探针**（阈值别太激进）。**自愈脚本是兜底，不是主方案**。

**易错**：`curl` 不加 `-f`（4xx/5xx 也返回 0）；健康检查打业务接口（有副作用/慢）；重启用 `kill -9`（丢正在处理的请求，优雅停机见 10_springboot.md 4.1.2）；脚本无锁导致两个实例同时重启（端口冲突/重复发布）。

### 5.1.2 日志轮转与清理怎么设计？

**三层日志治理**：**应用层**（logback/log4j2 的 RollingPolicy：按天/大小、保留 N 天、压缩）、**系统层**（logrotate 管 Nginx/Tomcat/cron 等非应用日志）、**平台层**（journald 限额、容器日志驱动轮转，23_docker.md 5.1.2；ELK/Loki 采集）。

**logrotate 配置要点（`/etc/logrotate.d/nginx`）**：
```text
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        [ -f /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)
    endscript
}
```
**两种"重开文件"策略**：**`create` + 信号**（进程重新打开新文件，最干净，需应用支持，如 Nginx USR1）；**`copytruncate`**（拷贝后截断原文件，适用于不支持重开的进程——**有丢日志窗口 + 大文件拷贝开销**）。**验证**：`logrotate -d /etc/logrotate.conf`（dry-run 看规则）。

**清理纪律**：`find ... -mtime +N -delete`（**先 `-print` 预览**，3.2.1）；**别 `rm` 正在写的文件**（空间不释放，1.2.2）；按"压缩→归档→删除"三段式（热日志→压缩包→对象存储/冷存储，25_minio.md）；监控**磁盘 + inode 双水位**（2.2.3）。

**易错**：只配 logrotate 没配应用自身（应用自己写死的文件不受 logrotate 管控）；`copytruncate` 与"按大小切分"叠加导致丢行；保留策略与合规冲突（该留 180 天的被 14 天删了）；压缩任务与采集器抢 IO（低峰执行）。

## 5.2 综合

### 5.2.1 线上故障排查的通用方法论是什么？

**先立框架：USE + 黄金指标**：
- **USE**（资源视角）：对每类资源（CPU/内存/磁盘/网络）看 **U**tilization（使用率）、**S**aturation（饱和度/排队）、**E**rrors（错误）——比"凭感觉 top 一下"系统；
- **黄金指标**（服务视角）：**延迟、流量、错误、饱和度**（Google SRE 四信号）；接口层面用 RED（Rate/Errors/Duration）。

**六步排查法**：
1. **定义症状与影响面**（哪个接口/哪些用户/从几点开始）——**先止血还是先定位**要分清（可用性优先时先降级/回滚）；
2. **看时间线找变更**（发布/配置/流量/依赖变更——"最后一次改了什么"命中率最高，33_cicd.md 的发布记录）；
3. **分层定位**（入口网关→应用→中间件→OS/网络，每层用对应工具：12_nginx.md 的 502/504、2.1.3 的 CPU、2.2 的内存/IO、2.3 的网络）；
4. **形成假设并验证**（每次只改一个变量；用指标/日志/抓包证实或证伪，别"猜着改"）；
5. **保留现场**（jstack/heap dump/日志/监控快照——**重启前先取证**，2.1.3）；
6. **复盘与固化**（根因、改进项、监控/预案补齐；事故是系统缺陷的暴露，不是"某个人手滑"）。

**常见"看起来是 A 其实是 B"**：接口慢=GC/锁/下游/连接池（不一定是代码）；CPU 高=GC/正则/序列化（不一定是业务计算）；内存涨=堆外/线程/缓存（不一定是泄漏）；网络超时=对端慢/keepalive 竞态/DNS（不一定是网络）。

**收口**："**排障的胜负在'分层'与'变更'两件事**——分层让你知道去哪看，变更让你知道为什么现在发生；工具只是执行细节。"

### 5.2.2 容器时代的 Linux：cgroup/namespace 对排查有什么影响？

**两个机制回顾**（23_docker.md 1.1.2）：**namespace** 给"视图"（PID/网络/挂载/用户/UTS/IPC），**cgroup** 给"限额"（CPU/内存/IO/PID）——排查时的三个"错位"：

1. **看到的不是"自己的"**：容器里 `top`/`free`/`uptime` 默认读**宿主**数据（内核没按 namespace 隔离这些统计）——`free` 显示宿主内存、`load` 是宿主负载；**正确姿势**：看 cgroup 文件（`/sys/fs/cgroup/memory.max`、`cpu.stat` 的 `nr_throttled`/`throttled_usec`、`io.stat`）或 `docker stats`/K8s metrics；装了 lxcfs 的容器才会"伪装"成容器视角；
2. **OOM 是 cgroup 级的**：容器内存超 `memory.max` → **cgroup OOM Killer** 杀容器内进程（不是宿主 OOM），退出码 **137**，`docker inspect` 的 `OOMKilled=true`；**堆 + 堆外 + 线程栈 + 元空间**必须小于限额（2_jvm.md 的容器感知、23_docker.md 5.1.1）；
3. **CPU 是"配额+节流"**：`--cpus=2` 是 CFS 配额，超了会被 **throttle**（`cpu.stat` 的 `nr_throttled` 增长）——表现为**周期性的延迟毛刺**（GC/JIT 线程被限流），`top` 里不一定看到 100%（配额内跑满但被周期性掐断）。

**排查入口**：`docker exec` 进容器（看容器视图）vs 宿主机（看全貌）；`nsenter -t <pid> -n/-m/-p` 进容器命名空间（没 ssh 时）；`/proc/<pid>/cgroup` 确认进程属于哪个 cgroup；K8s 里 `kubectl exec`/`kubectl top`（需要 metrics-server）/`describe pod` 看 limits 与 OOMKilled（32_kubernetes.md）。

**易错**：容器里看到宿主 load 高就扩容（其实是邻居）；只调 JVM `-Xmx` 不看容器限额（超限被 OOMKill）；把 CPU throttle 当"应用慢"（看 `cpu.stat`）；在容器里 `df` 看到宿主磁盘（overlay 配额/`--storage-opt` 才是容器层限额，23_docker.md 5.2.1）。