本篇覆盖 Docker 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：定位与底层原理、运行时与网络、Dockerfile 与构建、镜像分发与生态、生产实践。JVM 在容器里的内存参数见 2_jvm.md（5.1.1），Spring Boot 的部署形态见 10_springboot.md（2.2.2），容器编排的深水区见 32_kubernetes.md，Maven 构建缓存见 3_maven.md。

# 1. 定位与底层原理

## 1.1 容器与虚拟机的分界

### 1.1.1 Docker 和虚拟机的区别是什么？为什么说"容器不虚拟化内核"？

**一张表钉牢（面试第一问，答成"轻量级虚拟机"直接降档）**：

| 维度 | 虚拟机 | 容器 |
|------|--------|------|
| 抽象层 | 硬件虚拟化：Hypervisor 模拟 CPU/内存/设备 | **操作系统层虚拟化**：宿主机内核 + 隔离机制 |
| 内核 | 每个 VM 一套完整 Guest OS 内核 | **所有容器共享宿主内核** |
| 启动 | 引导内核与 init：秒级~分钟级 | 起进程：毫秒~秒级 |
| 体积 | GB 级（含完整 OS） | MB 级（只含应用与依赖） |
| 隔离强度 | 硬件级（强） | 进程级（弱——共享内核是安全模型的分界线） |
| 密度 | 单机数十台 | 单机数百~数千个容器 |
| 典型件 | VMware/KVM/Xen | Docker/containerd/Podman |

**"不虚拟化内核"的准确含义**：容器里的进程**直接就是宿主机内核上的普通进程**——只是被 namespace 切割了"视野"、被 cgroup 限定了"饭量"、被 capabilities/seccomp 削减了"权限"。没有指令翻译、没有内核态陷入的模拟开销，所以性能≈原生（网络/磁盘的损耗来自虚拟设备与桥接，不来自 CPU 虚拟化）；反过来，**内核版本、内核漏洞、内核参数（如 `vm.overcommit`、`net.core.somaxconn`）是所有容器共享的**——"Linux 容器不能跑在 Windows 内核上"、"容器逃逸漏洞一动全动"都源于这一条。

**由此推出的两条工程结论（这题的增量）**：
1. **隔离边界不是安全边界**：多租户/不可信负载（SaaS 跑客户代码、CI 跑外部 PR）不该裸用 runc——要么上**强隔离运行时**（Kata Containers 轻量 VM 包容器、gVisor 用户态内核拦截系统调用、Firecracker microVM），要么至少做 rootless + userns + seccomp 全套加固（5.2.2）；云厂商的"安全容器"产品全是这条谱系；
2. **容器 = 进程 + 打包 + 编排语义**：把容器当"部署单元"用时，它的价值大半在**镜像的不可变性与一致性**（构建一次到处运行），性能与"像 VM 一样隔离"都不是它的卖点——选型题到这一句收口即成熟。

**易错**：说"容器比虚拟机安全隔离差所以性能好"——因果是"共享内核所以省掉虚拟化开销"，隔离差是同一设计的代价而不是性能的原因；追问"容器里 `uname -a` 看到什么"——宿主内核版本（镜像里装的不是内核）。

### 1.1.2 容器的隔离与资源限制靠哪三件套实现？

**Linux 容器 = namespace（看见什么）+ cgroup（能用多少）+ 安全机制（能做什么）**，Docker 只是把它们包装成产品。

**① namespace：视图隔离**（每个容器一套"以为独占"的视图）：

| namespace | 隔离对象 | 容器里的效果 |
|-----------|---------|-------------|
| mnt | 挂载点 | 自己的根文件系统（镜像的 merged 视图） |
| pid | 进程号 | 容器内 PID 从 1 开始，看不到宿主进程 |
| net | 网络栈 | 自己的网卡/IP/端口/路由表（2.2.2 的网络模式本质是共享或新建这套） |
| ipc | 进程间通信 | 自己的信号量/共享内存 |
| uts | 主机名域名 | 自己的 hostname |
| user | 用户/UID 映射 | 容器内 root 可映射为宿主普通用户（rootless 的基础，5.2.2） |
| cgroup | cgroup 视图 | 看不到宿主其他 cgroup（较新内核） |
| time | 时钟偏移 | 较新特性，少用 |

**② cgroup v1/v2：资源限制**——CPU（`cpu.max` 配额/权重）、内存（`memory.max`，超限触发 **OOM Killer 杀进程**而不是"变慢"）、IO（`io.max` 带宽/IOPS）、PID 数（`pids.max`，防 fork 炸弹）。**v2 统一层级**是生产现状（新发行版默认；容器运行时与 cgroup 驱动要和 systemd 对齐，kubelet/containerd 配置错会"看不到限额"，32_kubernetes.md 的经典坑）。**这层是"限制"不是"隔离"**：超了直接杀或限流，不提供分区感——响应"容器内存超了会怎样"：**OOM Kill 容器内进程（通常杀最大 RSS 的那个，不一定是 Java 主进程的预期对象）**，退出码 137（128+9）是排查 OOM 的第一线索（2_jvm.md 的堆外与容器感知账在这对上）。

**③ 安全削减：从"全 root"到"最小权限"**——capabilities（把 root 的超级能力拆成细粒度位：`CAP_NET_ADMIN`、`CAP_SYS_ADMIN` 等，默认只给一小撮，`--cap-drop ALL` 起步再加）、seccomp（系统调用白名单，默认 profile 已拦危险 syscall）、AppArmor/SELinux（MAC 策略）、只读 rootfs（`--read-only` + tmpfs）。**"容器内 root ≠ 宿主 root"只有在 user namespace 映射后才成立**，裸 docker 默认容器 root 就是宿主 root（uid 0）——这是"容器不安全"论调的事实基础（5.2.2 的加固清单）。

**答题收口**：namespace 管"看不见"，cgroup 管"吃不多"，capabilities/seccomp 管"干不了"——三件套齐了才叫容器；Docker 的价值是把这三层的配置、镜像分发与网络编排做成了开箱即用。

## 1.2 组件、镜像与容器

### 1.2.1 Docker、containerd、runc 各管什么？OCI 标准解决什么问题？

**分层栈（自上而下，报得出每一层的"一句话职责"即过关）**：

```text
docker CLI ──REST──▶ dockerd（Docker Engine 守护进程：API、镜像构建/拉取、网络/卷编排）
                        │ gRPC
                        ▼
                     containerd（行业标准容器运行时：镜像/快照/任务/事件管理）
                        │ 调 OCI runtime
                        ▼
                     runc（OCI runtime 参考实现：建 namespace/cgroup、exec 容器进程，执行完即退出）
                        │
                        ▼
                     容器进程（直接跑在宿主内核上）
```

- **dockerd**：面向用户的编排层（`docker build/run/network/volume` 全在这层做产品化）；它**不亲自管容器生命周期**，重启 dockerd 不影响已运行容器（`live-restore` 更彻底，2.1.1）；
- **containerd**：CNCF 毕业项目，镜像分发、内容存储、快照、容器任务管理——**K8s 通过 CRI 直接对接它**（K8s 1.24 移除 dockershim 后，"集群不需要 Docker"指的就是这层：需要的是 containerd/CRI-O，32_kubernetes.md）；
- **runc**：最短命的角色——按 OCI 规范拉起容器（创建命名空间、挂载、capabilities、执行入口命令）后**把进程交给 shim/containerd 托管并退出**；出问题看它（"runc 版本过旧导致 CVE"的新闻主角就是它）。

**OCI 标准（开放容器倡议）解决"标准接口"**：**image-spec**（镜像格式：manifest/config/layers 的 tar+gzip 布局）+ **runtime-spec**（怎么根据配置拉起一个容器：bundle、config.json）+ **distribution-spec**（Registry API）。收益直白：**构建工具、运行时、镜像仓库可以互换**——BuildKit 构建的镜像能跑在 containerd+runc、Podman、K8s 任何 CRI 运行时上；Harbor 存的镜像 Docker/Podman 都能拉。**"标准化把容器从 Docker 公司产品变成行业基础设施"**——2015 年 OCI 成立、2017 年 containerd 捐 CNCF，是必须讲的历史线。

**易错**：① "Docker = containerd"——Docker 是**产品套件**（daemon/CLI/BuildKit/Compose/Desktop），containerd 是其中一个可拆卸零件；② "runc 是常驻进程"——它拉起容器后退场，常驻的是 containerd-shim（2.1.1）；③ "K8s 1.24 之后不能用 Docker 镜像"——镜像格式是 OCI 的，**任何 OCI 镜像都能用**，被移除的只是"用 dockerd 当节点容器运行时"这条适配层。

### 1.2.2 镜像、容器、仓库、数据卷是什么关系？容器生命周期如何流转？

**四件套一句话**：**镜像（只读模板）→ 容器（运行实例，叠加可写层）→ 仓库（分发镜像的存储与 API）→ 数据卷（脱离容器生命周期的存储）**。关系图：

```text
Registry（Harbor/Docker Hub/云镜像仓库）
   │ docker pull（按 digest 拉层，本地已存在则跳过）
   ▼
本地镜像（content-addressable 存储，层共享）
   │ docker run（在镜像栈顶加一个可写容器层）
   ▼
容器（进程 + namespace/cgroup + 可写层）
   ├── docker stop/start/restart：容器层保留（stop→start 数据还在）
   ├── docker rm：可写层销毁（数据丢失，除非挂卷）
   └── docker commit：把可写层固化成新镜像（不推荐，3.1.x/4.2.2 讲）
```

**生命周期状态机（`docker ps -a` 的 STATUS 全解）**：

```text
created ──start──▶ running ──exit/stop──▶ exited ──start──▶ running
   （docker create 只建不启）      │  主进程退出=容器退出（容器的"生死"是 PID1 的生死，
                                  │   2.1.2 的单进程模型）
                                  ├─ docker pause：SIGSTOP 冻结（running(Paused)）
                                  ├─ OOM：被杀，exited (137)
                                  └─ docker rm -f：强杀并删除
```

**三个概念性考点**：
1. **容器的状态由主进程决定**：容器里没有"系统"在跑，`CMD` 的进程（或其 exec 形式）退出，容器即退出——所以"后台守护进程"写法（`&`、daemonize）在容器里是反模式（主进程退出容器就死，或主进程不停则容器永不退出）；
2. **`docker stop` 与 `-t` 的优雅停机**：发 SIGTERM → 等 10 秒（可配）→ 不发 SIGKILL；**SIGTERM 打到 PID1 还是应用**取决于 shell/exec 形式（2.1.2/3.1.1 的经典事故）；
3. **仓库与 tag 的"可变性"**：`latest`、`v1` 这类 tag 可被重新 push 覆盖，**digest 不可变**——生产按 digest 部署（4.1.1）。

**易错**：`docker restart` 不重建容器（可写层数据保留、镜像更新不生效——"改了镜像重启容器没变化"就是这个）；`docker rm` 后**匿名卷（VOLUME 指令产生）可能残留**（`docker rm -v` 才连卷一起删），磁盘被无名卷吃满的经典现场（5.2.1）。

### 1.2.3 镜像分层与 overlay2 怎么工作？写时复制的代价是什么？

**分层模型（Docker 最有辨识度的设计）**：

```text
┌──────────────────────────────┐
│ 容器可写层（Container Layer） │ ← 运行时所有增删改，容器删则消失
├──────────────────────────────┤
│ 层4：COPY app.jar            │
│ 层3：RUN mvn package         │ ← 只读层，跨容器/跨镜像共享
│ 层2：RUN apt-get install ... │
│ 层1：FROM ubuntu:22.04       │
└──────────────────────────────┘
```

- **每层的内容寻址**：层的内容算 SHA256 → 相同内容 = 相同层 ID → 本地只存一份、拉取时 `Already exists` 跳过、推送时仓库不重复存（2.2/4.1）；
- **overlay2 的实现**：把镜像层作为 `lowerdir`（可多层）、容器层作为 `upperdir`、再加 `workdir`，内核 overlay 模块把它们联合挂到 `merged` 挂载点呈现给容器——**粒度是文件**：没被改的文件始终共享自 lower 层；
- **写时复制（CoW）**：改只读层里的文件 → **先把整个文件复制到 upper 层**再改（原文件在镜像里岿然不动）；新建文件直接落 upper；删除文件写 whiteout 标记（让下层同名文件"不可见"）。

**三层收益**给全（旧题"分层有什么好处"的升级版）：**存储共享**（基础镜像几十个服务共用一份）、**构建缓存**（前一层的输入不变就复用，3.2.1）、**分发增量**（pull/push 只传差异层）——加上**内容不可变与可验证**（digest 防篡改，4.1.1 的供应链信任基础）。

**CoW 的四个代价（这才是本题的区分度）**：
1. **大文件改动=全文件复制**：改镜像里一个 2GB 的 jar → upper 层瞬间多 2GB，且原层那份仍占着——**"容器磁盘越用越大"的第一根因是往容器层写大文件**（日志、临时文件、数据库文件必须挂卷，2.2.1）；
2. **首写延迟**：复制动作发生在写路径上，大文件首写毛刺（顺序大 IO 场景尤其明显）；
3. **删除不回收**：whiteout 只标记不删，**镜像层体积一分不少**——"删了日志文件磁盘不降"就是这个；容器层累积的删除标记还随镜像 commit 传承（所以不 commit 生产容器）；
4. **层数有上限**：overlay2 内核支持多 lowerdir 但 Docker 限制层数（约 125 层，且每条 RUN/COPY/ADD 一层）——层多不仅慢还占元数据，**合并 RUN/多阶段构建**的动机之一（3.2.1/3.2.2）。

**追问位**："`docker history` 里 `<missing>`"——层是本地拉下来的，缺少当时的构建中间容器信息，正常现象；"怎么验证分层效果"——`docker history`（每层指令与大小）+ `docker system df -v`（层磁盘占用）+ `dive`（第三方逐层文件浏览器）。**易错**：把 overlay2 说成"目录合并"——是**文件级 union**，同名文件上层胜出；把 CoW 说成"修改镜像"——镜像永远不变，变的是容器层副本。

# 2. 运行时与网络

## 2.1 运行时链路

### 2.1.1 docker run 一条命令背后经历了什么？容器 1 号进程为什么特殊？

**全链路（从敲回车到进程跑起来）**：

```text
① CLI → dockerd：REST /containers/create + /start（镜像不存在先 pull）
② dockerd：解析配置（网络/卷/资源限制）→ 交给 containerd
③ containerd：准备镜像快照（overlay2 的 upper/merged 挂载）、生成 OCI bundle
   （config.json：namespace 集合、rootfs、capabilities、seccomp、入口指令）
④ containerd 起 containerd-shim 进程（每个容器一个），shim 调 runc
⑤ runc：clone/exec 建 namespace + cgroup + pivot_root 切根 + exec 入口进程
   → 写 PID 文件 → 交还托管权后退出
⑥ dockerd 收到"started"，CLI 打印容器 ID，返回
   （-it 时再 attach 到容器 stdio；-d 时直接回提示符）
```

**关键角色三个"为什么"**：
- **shim 为什么存在**：让**容器生命周期不挂在 dockerd/containerd 上**——升级/重启这两个守护进程容器照跑（配置了 live-restore 更彻底），且 shim 是容器的**直接父进程**，负责回收僵尸、转发 stdout/stderr 日志、支撑 `exec` 进容器；
- **runc 为什么不常驻**：OCI runtime 的职责只有"按规范建容器"——建完退场，最小攻击面最小复杂度；**真正的隔离参数全在 bundle 的 config.json 里**（"容器到底怎么被拉起来的"追到这一层即满分）；
- **为什么是"进程树"不是"系统"**：容器没有 init system（除非自己带），**进程直接挂在内核上**——这就引出 PID1 的语义。

**PID1 的三个特殊性（Java 生产事故的高发区，与 3.1.1 呼应）**：
1. **信号语义不同**：内核**不把默认动作应用到 PID1**（这是"init 特殊对待"的规则）——如果应用是 PID1 而没注册 SIGTERM 处理，`docker stop` 的 SIGTERM 会被**忽略**，只能等超时被 SIGKILL；Java 进程有 shutdown hook 能接住，但**shell 形式启动时 SIGTERM 打在 `/bin/sh`（PID1）上、应用收不到**——"docker stop 变慢 10 秒硬杀、Spring 优雅停机不触发"的根因；
2. **僵尸进程无人回收**：PID1 的职责之一是 `wait()` 孤儿；应用进程当 PID1 又不 wait 子进程（启动外部脚本/子进程的场景）→ 僵尸累积（`ps` 里一堆 defunct）；
3. **PID1 死了容器就死**（1.2.2），且不能靠 `kill -9 1` 自杀式重启。

**解法四件套**：**exec 形式启动**（`ENTRYPOINT ["java","-jar","app.jar"]`，信号直达应用）；**`docker run --init`**（注入 tini 作 PID1：转发信号 + 回收僵尸）；或者应用自建进程管理器（不推荐）；以及**优雅停机预算对齐**——`stop_grace_period`/`-t` 要大于 Spring 优雅停机时长（10_springboot.md 4.1.2 的 `server.shutdown=graceful` 与 K8s `terminationGracePeriodSeconds` 是同一笔账，32_kubernetes.md）。

**易错**：`docker exec` 进去的 shell **不是 PID1**（是容器内新起的进程，`kill 1` 或乱杀进程的"容器重启"就是从这来的）；容器日志**不带时间戳归属**——多进程往 stdout 写的日志混在一起，诊断要靠单进程纪律与结构化日志（5.1.2）。

### 2.1.2 数据持久化三种方式怎么选？容器层写数据的坑有哪些？

**三种挂载的对照（选型主轴：谁管理、是否持久、性能）**：

| 方式 | 管理方 | 位置 | 典型场景 | 注意 |
|------|--------|------|---------|------|
| **volume** | Docker | `/var/lib/docker/volumes/` | **生产持久化**（DB 数据、上传文件）、多容器共享 | 删容器不删卷（`-v` 才删）；备份/迁移要认 Docker 的命名与路径 |
| **bind mount** | 用户 | 宿主任意路径 | 开发热更新（挂源码/配置）、宿主已有数据目录、`/etc/localtime` 这类系统文件 | **权限/属主即宿主原样**（容器内 UID 权限坑）、跨平台路径语义差异、可被写成危险挂载（挂 `/` 或 docker.sock） |
| **tmpfs** | 内核内存 | 宿主内存 | 敏感临时文件、高性能 scratch | 容器停即丢；计入内存账（cgroup 看到的是同一笔） |

**工程选择三条**：
1. **有状态服务一律 volume**（或云盘/CSI 卷）——容器层是"临时地板"，数据库写容器层=OOM 与 CoW 双杀（1.2.3 代价 1）；
2. **配置走挂载或环境变量，镜像内不留环境差异**（12-factor 思想；`--env-file`、config 挂载的坑是"改了容器内文件重启就没了"——因为没挂卷）；
3. **bind mount 的生产风险是权限模型**：容器内进程以非 root（5.2.2）跑时，宿主目录属主不匹配 → Permission denied；SELinux 还要 `:z/:Z` 标签；**最危险的是挂 `docker.sock`**——等于把宿主 root 交给容器（5.2.2 安全清单点名）。

**容器层写数据的坑清单**（旧题"写时复制"的实战化）：
- **日志写容器内文件**：默认 json-file 日志驱动写在宿主（`/var/lib/docker/containers/...`）——那部分是宿主的，但**应用自己写的文件**（`logs/app.log`）在容器层 → 删容器即丢 + CoW 膨胀；正解：stdout 输出（5.1.2）或挂卷；
- **临时文件**：`/tmp` 没挂 tmpfs 时落容器层——大文件处理场景瞬间吃盘；
- **清理无效**：容器内 `rm` 大文件，宿主的镜像层/upper 层体积**未必下降**（whiteout + 共享层，1.2.3 代价 3）——"容器内删了 10G 但宿主盘没变"的答案；
- **`docker cp` 不是持久化**：拷进容器层的文件删容器就没；拷出是救急手段不是数据通道。

**追问**："volume 备份怎么做"——停写后 tar 容器内挂载点（或 `--volumes-from` 辅助容器），更正规的答案是**数据不在容器里**：DB 用其自身的备份机制（16_mysql.md 4.x 的 binlog/xtrabackup 族），卷只是承载。

## 2.2 网络与端口

### 2.2.1 Docker 的网络模式有哪些？容器间、出网、入网分别怎么走？

**五种模式（`--network` 的取值）**：

| 模式 | 网络栈 | 隔离 | 场景 |
|------|--------|------|------|
| **bridge**（默认） | 每个容器独立 netns，接 `docker0`（或自定义 bridge） | 强 | 绝大多数单机服务；自定义 bridge 才有**容器名 DNS** |
| **host** | 共享宿主 netns，无独立 IP | 无 | 性能敏感/大量端口/需要宿主网络语义（如某些监控代理）；端口冲突=直接抢占宿主端口 |
| **none** | 只有 lo | 完全 | 纯计算任务、安全隔离的中间态 |
| **container:`<id>`** | 共享指定容器的 netns | 与目标一致 | Pod 模式雏形（sidecar 与主容器共享 localhost）、K8s pause 容器的思想来源 |
| **overlay**（Swarm 管理，`--network` 用自定义 overlay 网络） | 跨主机 VXLAN 隧道 | 逻辑网络 | 多机容器互联（Swarm）；K8s 里对应 CNI 插件（32_kubernetes.md） |

**三条数据路径（"怎么通"比"有哪些模式"重要）**：
1. **容器间**：同一 bridge 上通过**虚拟网桥**二层互通；**自定义网络**内置 DNS（127.0.0.11 的嵌入式解析器）——容器名/网络别名可直接当主机名；**默认 `docker0` 网络没有这个 DNS**，只能靠 IP 或 `--link`（已废弃）——这是"容器间通信要不要用自定义网络"的标准答案；**不同 bridge 网络之间默认不通**（要 `docker network connect` 把容器接进第二个网络）；
2. **容器出网（访问外网）**：容器 → veth → docker0 → 宿主**iptables MASQUERADE**（SNAT 成宿主 IP）→ 出公网；DNS 走宿主 `/etc/resolv.conf` 或 Docker 内嵌解析器（宿主自定义 DNS 的坑：容器拿不到 systemd-resolved 的 127.0.0.53，需在 daemon.json 配 dns）；
3. **容器入网（被外部访问）**：**端口发布** `-p 8080:80` → dockerd 在宿主 iptables 的 `nat/DOCKER` 链写 **DNAT** 规则（宿主 8080 → 容器 IP:80），同时配 FORWARD 放行——**真正的转发是内核 iptables，不是 docker-proxy 进程**（docker-proxy 只在特定场景兜底/兼容，2.2.3）。

**易错**：① "host 模式性能好所以生产都用"——失去端口隔离与统一发布，编排系统里基本禁；② "容器 IP 可以直接给外部用"——bridge 的容器 IP 是**宿主内部**的私网段，重启会变，跨宿主不可达（要 macvlan/overlay/云网络，2.2.3）；③ 把 `EXPOSE` 当发布端口——仅元数据声明（3.1.2），真正发布靠 `-p/-P`。

### 2.2.2 端口发布与防火墙有什么经典事故？跨主机网络怎么组？

**端口发布的三层账**：
1. **语法**：`-p 宿主IP:宿主端口:容器端口/协议`（省略宿主 IP = `0.0.0.0`，即**所有网卡**——公网暴露的默认姿势，生产必须显式绑内网 IP 或前置反代）；`-P` 才按 EXPOSE 随机映射（大 P 随机、小 p 指定，高频笔误）；端口冲突报 `bind: address already in use`——先 `ss -lntp` 定占用者（host 模式让容器与宿主直接抢端口，2.2.1）；
2. **iptables 规则族**：DNAT（入向）+ MASQUERADE（出向，2.2.1）+ **`DOCKER-USER` 链**（用户自定义过滤的官方插入点）；
3. **经典事故一：ufw/firewalld 白名单失效**——Docker 直接写 iptables 的 nat 表，报文在 PREROUTING 就被 DNAT 进容器，**根本不经过宿主 filter 链的默认拒绝规则**：`ufw deny 8080` 依然能从公网访问 8080。正解三条：把规则写进 `DOCKER-USER` 链（`iptables -I DOCKER-USER ...`，重启后由 docker 保留该链）、或 `daemon.json` 的 `iptables: false` 自己全权接管（代价：发布端口的手工配置全自理，不推荐除非有网络团队）、或**最简单**——发布绑到内网 IP + 入口只让 Nginx/SLB 访问（12_nginx.md 的分层入口思想才是终解）；
4. **经典事故二：`-p` 暴露数据库**——测试环境随手 `-p 3306:3306` 被打穿（红队最爱），纪律：数据服务只进自定义网络、不发布端口，需要外部连接走跳板/隧道。

**跨主机网络三条路（选型与代价）**：
- **overlay + Swarm**：VXLAN 封装（UDP 4789），swarm 的 ingress/自定义网络，控制面管成员与密钥；支持 `--opt encrypted`（IPsec 加密，性能代价）；**生态随 Swarm 萎缩**（4.2.2），K8s 场景等价物是 CNI（flannel/calico，32_kubernetes.md）；
- **macvlan**：容器直接挂到宿主物理网卡，**分到与宿主同网段的真实 IP（L2 直通）**——"像局域网里一台小机器"，无 NAT 开销；硬坑：**同一宿主上的 macvlan 容器与宿主自身不能通信**（内核设计）、交换机需放行多 MAC（云厂商 VPC 大多不支持，需要 ENI/辅助 IP）；适合需要二层可见/组播/旧系统直连的场景；
- **ipvlan**：共享宿主 MAC、按 L2/L3 模式分 IP——绕开 macvlan 的"交换机 MAC 限制"；局限类似（宿主通信）。
- **收口判断**：单机=自定义 bridge；跨主机=交给编排平台（K8s CNI/云网络）而不是手搓 overlay——"网络问题在平台层解决"是现代答案（1.1.1 的收敛、32_kubernetes.md）。

**易错追问**："docker0 网段冲突"（宿主机所在办公网恰好 172.17.0.0/16 → 容器出网黑洞，daemon.json 改 `bip`/`default-address-pools`）；"容器 ping 宿主"（默认 bridge 网关就是宿主 docker0 IP，可通；host 模式本机语义全在）。

# 3. Dockerfile 与构建

## 3.1 指令语义

### 3.1.1 RUN、CMD、ENTRYPOINT 怎么区分？shell 与 exec 形式为什么是生产事故的分水岭？

**三兄弟分工（一张表 + 一个时序）**：

| 指令 | 执行时机 | 是否产生新层 | 可否被 `docker run` 覆盖 |
|------|---------|-------------|------------------------|
| **RUN** | 构建时（build） | **是**（每条一层） | 不可（已固化成镜像） |
| **CMD** | 容器启动时（run） | 否（元数据） | **整体可被覆盖**（`docker run img <cmd>`） |
| **ENTRYPOINT** | 容器启动时（run） | 否（元数据） | 不被参数覆盖，只被 `--entrypoint` 显式替换 |

**组合语义**：ENTRYPOINT 是"固定程序"，CMD 是"默认参数"——`ENTRYPOINT ["java","-jar","app.jar"]` + `CMD ["--spring.profiles.active=prod"]`，`docker run img --server.port=8081` 的结果是 `java -jar app.jar --server.port=8081`（CMD 被替换、ENTRYPOINT 保留）。三个易错：**多条 CMD 只有最后一条生效**；**子镜像一旦声明 ENTRYPOINT，父镜像的 CMD 会被重置为空**（需要在新镜像里重新 CMD）；shell 形式下 CMD 只是字符串参数、不会进 shell 解析。

**shell 形式 vs exec 形式（本题真正的考点）**：

```dockerfile
# shell 形式：实际是 /bin/sh -c "java -jar app.jar"
ENTRYPOINT java -jar app.jar

# exec 形式：JSON 数组，直接 execve，java 就是容器 PID1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

两者差在**进程树与信号**：shell 形式下 PID1 是 `/bin/sh`，`docker stop` 的 **SIGTERM 打在 sh 上**——很多 sh 实现不转发信号给子进程，JVM 的 shutdown hook 不触发，10 秒后被 SIGKILL 硬杀：**"优雅停机失效、docker stop 每次都等满超时"的头号根因**（2.1.1 的 PID1 语义在此落地，10_springboot.md 4.1.2 的 `server.shutdown=graceful` 配了也白配）。

**但 Java 生产不能无脑 exec 形式**：`ENTRYPOINT ["java", "$JAVA_OPTS", "-jar", "app.jar"]` 里 **`$JAVA_OPTS` 不会被展开**（exec 形式没有 shell 解析）——JVM 报"找不到主类 $JAVA_OPTS"。标准解是**入口脚本 + exec 收尾**：

```bash
#!/bin/sh
# entrypoint.sh：脚本里做环境变量拼装，最后 exec 把自身替换成 JVM（信号直达 + PID1 正确）
exec java ${JAVA_OPTS} -jar /app/app.jar "$@"   # "$@" 接 docker run 追加的参数
```

`exec` 是点睛之笔：shell 被 JVM 进程替换，PID1 变成 JVM，信号与退出码全部直达（3.1.2 的脚本实践、K8s 的 preStop/优雅下线同源，32_kubernetes.md）。

**易错**：`RUN` 里装完软件就以为"运行时会跑"——RUN 只是构建期一次性执行；`CMD` 写多条期望"依序执行"——只留最后一条；`ENTRYPOINT` 用了 shell 形式还指望 `docker run img --port` 追加参数——shell 形式的参数是被追加到 `/bin/sh -c` 后面，语义完全走样。

### 3.1.2 COPY/ADD、ARG/ENV、HEALTHCHECK、VOLUME 各有什么坑？

**COPY vs ADD**：COPY 只做"从构建上下文复制"；ADD 多两个魔法——**URL 下载**与**本地 tar 自动解压**。生产口径：**默认 COPY**（行为可预期、可缓存、可审计）；ADD 的 URL 模式**不校验完整性、不自动解压、缓存语义还特殊**，等于把"构建期网络依赖"埋进 Dockerfile——要下载就用 `RUN curl -fsSL --output ... && sha256sum -c`（或新版本 `ADD --checksum`），解压显式 `RUN tar -xzf`。**COPY 的隐藏语法**：`--chown=uid:gid`（配合非 root 运行，3.2.2/5.2.2）、`--from=builder`（多阶段，3.2.2）、`--link`（BuildKit 独立层，缓存更细但依赖图要自己保证）；**尾斜杠与 glob 语义**（`COPY src/ /app/` 是内容复制，`COPY src /app/` 目录名语义不同）是高频笔误。

**ARG vs ENV（安全题的高发区）**：

| | ARG | ENV |
|--|-----|-----|
| 作用域 | **仅构建期**（`--build-arg` 传入） | 构建期 + **运行时持久**（进镜像配置） |
| 是否会留在镜像 | 不进最终环境变量，但**会出现在 `docker history` 的指令文本里** | 会 |
| 典型用途 | 版本号、构建开关、目标平台 | 应用配置、PATH、JAVA_HOME |

三个坑：① **把密钥放 ARG/ENV**——`docker history` 一览无余（构建机上任何能看镜像的人都能拿到），正解是 BuildKit 的 `--mount=type=secret`（密钥不落层、不进历史）；② `ARG` 在 `FROM` 之前的"全局段"只在 FROM 行可用，进阶段要**重新声明**才生效；③ 同名时 ENV 覆盖 ARG（构建期取值以 ENV 为准），跨阶段不继承 ARG——"多阶段里 ARG 丢了"常是这个。

**其余四条速记**：
- **EXPOSE**：纯文档声明（及 `-P` 随机映射的依据，2.2.2），不发布端口；
- **WORKDIR**：不存在则创建，相对路径**累加**（多次 WORKDIR 是相对上一次）——`RUN cd` 不持久，切目录用 WORKDIR；
- **USER**：只影响其后的 RUN/CMD/ENTRYPOINT；**COPY 进来的文件属主默认 root**，非 root 用户读不了——配 `--chown` 或 `RUN chown -R`；uid 数字与宿主映射问题在 5.2.2 展开；
- **VOLUME**：声明即"运行期匿名卷挂载点"——**构建期在该路径后续写入的数据会被丢弃**（经典陷阱：先 VOLUME 再 RUN 装依赖，依赖消失）；用它等于把"这里必须持久化"写进镜像契约，但很多团队主张在编排层显式声明卷（更可控），至少不要把 VOLUME 放在会写数据的目录之上。

**HEALTHCHECK**：`HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 CMD curl -f http://localhost:8080/actuator/health || exit 1`——退出码 0=healthy、1=unhealthy（容器不会被自动重启，只是状态标记，重启要靠编排层）；**K8s 不消费 Docker 的 HEALTHCHECK**（它有自己的 liveness/readiness/startup probe，32_kubernetes.md）——跨平台部署时两套健康模型要分别配；**代价**：探测进程本身占 CPU/进程数，间隔太密会干扰性能测试（5.1.2 的观测面）。

## 3.2 构建优化

### 3.2.1 构建缓存怎么工作？Dockerfile 怎么排布让命中率最高？

**缓存键的构成（BuildKit 口径）**：每条指令的缓存 = **父层链 + 指令文本 + 该指令的输入指纹**（COPY/ADD 是**被复制文件的内容哈希**，不是文件名；RUN 是指令字符串 + 环境/ARG 等参与值）。命中则直接复用层、跳过执行；**一层的输入变了，它及其后所有层全部重建**（"缓存失效链"）——所以 Dockerfile 的顺序就是缓存友好度的设计。

**黄金排序（依赖多、变更少 → 前；代码变更频繁 → 后）**：

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /build
COPY pom.xml .                                   # ① 依赖描述：很少变
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B dependency:go-offline                # ② 下依赖：pom 不变就不重跑（缓存目录不进镜像）
COPY src ./src                                   # ③ 源码：天天变
RUN --mount=type=cache,target=/root/.m2 mvn -B package -DskipTests
```

**五条优化手段按收益排**：
1. **依赖前置 + cache mount**：`--mount=type=cache`（BuildKit）把 `.m2`/`node_modules` 的下载缓存放到构建器本地——**比"依赖层"更快且不膨胀镜像**；注意 CI 是"一次性构建器"时 cache mount 无效，要配合 ② 的注册表缓存；
2. **注册表级缓存**：`--cache-to type=registry,ref=harbor/app:cache,mode=max` + `--cache-from`——跨 CI runner 复用（4.1.1 的仓库也当缓存用）；`mode=max` 连中间阶段都存（配多阶段构建才有意义）；
3. **`.dockerignore`**：`target/`、`.git/`、`node_modules/` 排除——**减小构建上下文上传**，且被忽略文件的变化不会触发 COPY 层失效（"改了 target 里的 class 也导致 COPY src 失效"的怪象就来自没 ignore）；
4. **一条 RUN 里完成"更新索引+安装+清理"**：`apt-get update && apt-get install -y --no-install-recommends ... && rm -rf /var/lib/apt/lists/*`——分开写会**缓存住过期的索引**（构建几个月后突然 404 的经典事故）；装包**固定版本**保可复现；
5. **调试手段别背锅**：`--no-cache`（全量重建）、`--pull`（基础镜像取新）、`--no-cache-filter=stage`（只废掉指定阶段，BuildKit 新能力）——"缓存命中错误"时逐层二分，而不是无脑 --no-cache。

**易错**：把 `COPY . .` 放最前面（任何文件变化都重装依赖——最高频的低级错）；`RUN` 里 `cd` 或 `source`（不生效/层内环境出不去，3.1.2）；缓存**不是安全边界**——缓存层里可能残留历史构建的文件（`--mount=type=secret` + 定期全量重建治理）。

### 3.2.2 多阶段构建怎么用？Java 镜像怎么瘦身？

**机制**：一个 Dockerfile 多个 `FROM`，用 `AS` 命名阶段，**最终镜像只保留最后一个阶段的文件**；前面阶段（编译工具链）不进生产镜像：

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /build
COPY . .
RUN --mount=type=cache,target=/root/.m2 mvn -B clean package -DskipTests

FROM eclipse-temurin:21-jre                      # 或 jre-alpine / distroless/java21
WORKDIR /app
COPY --from=builder /build/target/app.jar app.jar
USER 10001                                       # 非 root 运行（5.2.2）
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

收益三层：**体积**（JDK+Maven 仓库动辄 600MB+，最终只剩 JRE+jar）、**攻击面**（编译器/包管理器/测试依赖不进生产，CVE 扫描面骤减）、**构建缓存更有意义**（`--from` 复制可以精确到单个产物，配合 3.2.1 的依赖前置）。

**Java 专属的四级瘦身（按收益排）**：
1. **JRE 而非 JDK**（temurin 的 `-jre` 变体约省 200MB）+ **distroless**（无 shell/包管理，异常时 exec 不进去——排障成本换安全面，团队要能接受）；
2. **jlink/jdeps 定制运行时**：只保留应用用到的模块（21 模块化后收益可观），代价是反射/动态加载需显式 add-modules，**spring-boot 生态兼容性要实测**；
3. **Spring Boot 分层 jar（layertools）**：`java -Djarmode=tools -jar app.jar extract`（3.x 新姿势，旧版 `-Djarmode=layertools`）拆出 dependencies/spring-boot-loader/snapshot-dependencies/application 四层，Dockerfile 里**逐层 COPY**——业务代码变只重写最后一层，**依赖层永久命中缓存**（这正是 1.2.3 层共享设计的教科书用法：镜像推送/pull 只传变化层）；
4. **基础镜像家族**：alpine 省空间但 **musl libc ≠ glibc**——依赖 glibc 的原生库（部分 JNI、netty-tcnative、字体/时区数据）可能瘆人地失败，**没实测过别上 alpine**；`spring-boot:build-image`（Buildpacks）与 **Jib** 是"不用写 Dockerfile"的替代路线（Jib 无需 daemon、分层与可复现性好，适合 CI）。

**多平台**：`docker buildx build --platform linux/amd64,linux/arm64`（苹果 M/ARM 服务器混部必备，4.1.1 的 manifest list 机制）——注意**交叉构建下 JNI/原生依赖的平台适配**，不是"能构建就能跑"。

**易错**：`COPY --from=0` 用序号脆弱（阶段增删即错位，用命名）；多阶段没清理 builder 里的 `target/` 就 COPY 整个目录（把 class 与源码也带进镜像）；`docker history` 看不到 `--from` 层的来源（用 `dive`/`--provenance` 追）。

# 4. 镜像分发与生态

## 4.1 仓库与分发

### 4.1.1 tag 与 digest 的语义差在哪？镜像治理怎么做？

**语义**：**tag 是可变的"指针"**（`app:1.2.3` 今天指向 A 镜像，明天可被重新 push 成 B 内容）；**digest 是内容哈希**（`sha256:...`，内容变则 digest 必变）。`docker pull app@sha256:...` 按 digest 拉——**审计、回滚、供应链验证的锚点**。**多架构镜像**：一个 tag 对应 **manifest list/OCI index**（指向不同平台的 manifest），`--platform` 决定拉哪份，各平台 digest 不同——"同一 tag 在两个节点 digest 不一样"多半是架构差异，不是被篡改（4.1.1 的 buildx 产出在此消费）。

**治理四条（生产必备）**：
1. **部署按 digest 或不可变 tag**：`latest` 在生产是反模式（"重启拉不到新版本"与"悄悄拉到未验证版本"一体两面；K8s 的 `imagePullPolicy` 语义纠缠在此，32_kubernetes.md）；Harbor 的**不可变 tag 策略**从仓库侧强制（4.1.2）；
2. **私有仓库 + 镜像缓存**：Docker Hub 对匿名/低额度账号**按 IP 或账号限流**（具体额度以官方公告为准），生产节点直连 Hub 既慢又不稳——Harbor/云仓库做代理缓存（pull-through cache）或 CI 推送到内网仓库；
3. **仓库也是缓存**：`--cache-to type=registry` 把构建缓存推进仓库（3.2.1），与镜像共用存储生命周期——**配额与 GC 要一起规划**（Harbor 的 tag 保留策略 + registry GC）；
4. **分发链路可观测**：pull 耗时、层复用率、失败重试——大规模节点的"镜像拉取慢"往往不是网速而是**层没共享**（每个服务独立 base 镜像、每层都不同 → 无法复用，1.2.3 的收益归零）。

**追问**："Registry 的存储后端"——distribution 规范是"API 与存储分离"，本地 filesystem/S3/OSS 皆可；HA 靠共享存储+多副本（无状态 API 层）；**GC 的坑**：删除 manifest 后 blob 仍占空间，要显式 GC 且**避免与 push 并发**（悬空引用风险）。

### 4.1.2 Harbor 与镜像安全供应链各管什么？

**Harbor（CNCF 毕业的企业级 Registry）能力清单**（面试报"治理件"而非"能存镜像"）：
- **项目与 RBAC**：多租户隔离、角色（访客/开发者/维护者/项目管理员）、机器人账号（CI 专用，最小权限）；
- **复制（replication）**：多仓库/多集群/离线环境同步——"生产拉内网副本、灾备跨区复制"的标准件；
- **不可变 tag + 保留策略 + 配额**：从仓库侧封住"tag 被覆盖""镜像无限堆积"两类事故（4.1.1 治理 1/3 的落点）；
- **漏洞扫描**：Trivy 类引擎按 CVE 打分，可配"高危阻断拉取"（与 CI 门禁配合，见下）；
- **签名与准入**：Notary v1（DCT，历史件）与 **cosign（Sigstore）**签名验证——"只允许部署已签名镜像"靠准入控制器（K8s 侧 OPA/Kyverno，32_kubernetes.md）；
- **审计与 Webhook**：谁推了什么/谁删了镜像（合规刚需），事件推 CI/CD（33_cicd.md）。

**供应链安全五段式（现代答题框架，2026 高频）**：
1. **构建可信**：固定基础镜像 digest、最小化基础镜像、构建环境隔离（CI 专用 runner）；
2. **物料透明**：**SBOM**（软件物料清单，syft/`buildx --attest type=sbom`）+ **provenance**（谁在什么流水线用什么源构建——SLSA 框架的落点）；
3. **扫描门禁**：CI 阶段扫描 + 仓库侧持续扫描（新 CVE 会在镜像"静置"期间出现，只在构建时扫会漏）；
4. **签名验证**：cosign 签 digest，运行侧准入校验（4.1.2 上文的闭环）；
5. **运行时收敛**：非 root、只读根文件系统、drop capabilities、seccomp（5.2.2）——"镜像里干净"不等于"跑起来安全"。

**收口句**："Registry 是供应链的**仓库+关卡**——只管存，镜像就是黑盒；挂上不可变、扫描、签名、准入四件套，它才成为**证据链的一环**。"

## 4.2 编排与演进

### 4.2.1 Docker Compose 怎么用？和 Kubernetes 是什么关系？

**定位**：单机多容器编排的"声明式 docker run 集合"——`compose.yaml`（**Compose 规范**由 compose-spec 维护，独立于 Docker 公司）描述 services/networks/volumes，一条 `docker compose up -d` 拉起整套拓扑；v2 起 `docker compose` 是 CLI 插件（Python 版 `docker-compose` 为历史件）。

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment: [SPRING_PROFILES_ACTIVE=prod]
    depends_on:
      db:
        condition: service_healthy      # 等健康检查通过，而不是"启动顺序"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
  db:
    image: mysql:8.4
    volumes: [db-data:/var/lib/mysql]
    environment: [MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db_pwd]
    secrets: [db_pwd]
volumes:
  db-data:
secrets:
  db_pwd:
    file: ./db_pwd.txt
```

**核心概念与高频坑**：**services**（每个=一份运行配置，可 `scale` 但不能跨主机）、**networks**（默认建一个项目级 bridge，服务名即 DNS，2.2.1）、**volumes**（命名卷 + 生命周期独立）；`depends_on` 只保证**启动顺序**，就绪要 `condition: service_healthy`（"db 还没初始化完 app 就连接失败"的根因）；**profiles** 按场景启停（`--profile debug`）；**多文件覆盖**（`-f base.yml -f prod.yml`，后文件深合并——镜像 tag/端口覆盖的常用姿势）；`docker compose config` 验证合并结果、`down -v` 连卷删除（数据丢失的开关）。

**与 K8s 的关系（必考辨析）**：Compose 是**单机/开发/CI 编排**（无调度、无自愈、无多副本滚动、无跨节点网络）；K8s 是**集群编排**（副本控制、探针、滚动更新、服务发现、跨节点 CNI，32_kubernetes.md）。两者概念映射（用 Kompose 可粗转，但**不是 1:1**：compose 的 depends_on ↔ K8s 没有对应物、healthcheck ↔ probe 语义差、network ↔ Service/Ingress）。**实践口径**：本地开发用 compose（一份 yaml 秒起依赖中间件），生产用 K8s/云容器服务；**别把 compose 当生产编排**——单机故障域与无自愈两点就否决了。

### 4.2.2 Swarm 现在还能用吗？Podman 这类替代怎么选？

**Swarm 的现状（诚实版）**：内置于 Docker Engine（`docker swarm init` 即得集群），能力上"够用"（服务副本、滚动更新、overlay 网络、secrets、stack deploy）；但**迭代几乎停滞、生态与人才向 K8s 集中**，加上 Docker 公司 2019 年把企业版业务出售给 Mirantis 后战略重心转向 Desktop/Hub——**新建生产集群选 Swarm 是逆流**；小团队/边缘节点用它"够省事"仍成立，答题报"维护模式、非战略选择"即可（对比 32_kubernetes.md 的编排能力全景）。

**Podman（RHEL 系的默认答案）**：
- **无守护进程**：CLI 直接 fork/exec 容器（conmon 托管），没有 dockerd 这个 root 常驻单点；
- **rootless 一等公民**：普通用户跑容器（user namespace 映射，1.1.2 三件套的权限面），配合 `podman generate systemd`/quadlet 做开机自启，**"容器当系统服务"的姿势比 docker 更原生**；
- **pod 概念**：一组共享网络的容器（K8s 语义对齐，为迁移铺路）；
- **兼容**：`alias docker=podman` 基本可用；镜像格式是 OCI（1.2.1 的标准红利），Docker 构建的镜像直接跑；compose 支持靠 podman-compose/兼容层（成熟度略逊）；
- **生态位**：RHEL/CentOS 系已不提供 docker 包、默认 Podman（企业合规与 rootless 诉求），CI runner 上常见；**Docker Desktop 的商业许可**（大企业需订阅）也把一部分团队推向 Podman Desktop/colima 类替代。

**选型收口**：**开发机**跟随团队与公司许可（Docker Desktop 体验最顺，替代方案可用但文档/生态略欠）；**CI 构建**用 BuildKit/buildah 均可（OCI 产物统一，1.2.1）；**生产**的编排选择与运行时选择是两件事——K8s + containerd 是事实标准，**Podman 是"没有 K8s 时的单机/边缘方案"**（对标 docker engine 而非 K8s）。记住一句："**镜像标准统一后，'用哪个工具'退化为团队习惯与合规问题**——这点是 OCI 给行业最大的礼物。"

# 5. 生产实践

## 5.1 容器化运行

### 5.1.1 Java 应用容器化有哪些经典坑？内存、核数、时区、信号逐条拆

**① JVM 与 cgroup 内存对齐（第一大坑，OOMKilled 的头号来源）**
- JVM 从 **JDK 8u191/10 起默认开启容器感知**（`UseContainerSupport`）：按 cgroup 的内存限额推导 `MaxHeapSize`（≈限额 1/4）——**不再看宿主内存**；
- 但 1/4 是"保守默认"不是"最优"：现代姿势是 **`-XX:MaxRAMPercentage=75.0`**（显式、可审计），配 `-Xms` 与 Xmx 相同（2_jvm.md 5.1.2 的理由在容器里同样成立：避免运行时扩堆）；
- **剩余 25% 不是浪费**：元空间、code cache、线程栈（每线程 1MB 量级）、直接内存（Netty）、JNI、GC 结构都要吃堆外——**"Xmx 设成限额的 100%"必然被 OOMKilled**（退出码 137，2.1.1/1.1.2 的 cgroup 账在 JVM 场景的具象）；
- **cgroup v2 兼容性**：JDK 15+ 原生支持；老版本需较新的补丁版（8u372+/11.0.16+ 一族）或宿主回退 v1——"容器里 JVM 看到的是宿主内存、堆开到几十 G 然后被杀"的先查这条；
- **OOM 现场留证**：`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps` 且 `/dumps` **挂卷**（不然容器一杀，证据随容器层蒸发——2.2.1 的"数据不在容器里"）。

**② CPU 限额与"核数幻觉"**：JVM 按 cgroup CPU 配额推导 `availableProcessors`（GC/JIT/ForkJoinPool 的线程数全跟着它）——**`--cpus=2` 会让 JVM 以为是 2 核**；但若只配了 CPU 权重（shares）没配配额，JVM 看到的是宿主核数 → 线程开得多、实际被调度器**节流（throttling）**→ GC 停顿与 RT 毛刺（监控看 `container_cpu_cfs_throttled_periods`，5.1.2）。稳妥姿势：**配额与 `-XX:ActiveProcessorCount` 双显式**（尤其 K8s 的 requests/limits 分离场景，32_kubernetes.md）。

**③ 信号与停机（2.1.1 的落地）**：exec 形式/`exec` 脚本入口（3.1.1）；`docker stop -t 30` 与 Spring 优雅停机（10_springboot.md 4.1.2）的时长预算对齐；**K8s 场景还要留 preStop 与 terminationGracePeriodSeconds 的余量**——"发布时 1% 请求 502"的经典位置（12_nginx.md 的 upstream 摘除时序同款）。

**④ 时区、字符集、证书、字体（"环境差"四件套）**：
- 时区：镜像默认 UTC（无 tzdata）→ 日志差 8 小时、定时任务错时——`TZ=Asia/Shanghai` + `-Duser.timezone`，或挂 `/etc/localtime`（bind mount，2.1.2）；
- 证书：distroless/scratch 基础镜像**没有 CA 证书** → 出站 HTTPS 报 `PKIX path building failed`（Java 侧 `cacerts` 也要核对——`-Djavax.net.ssl.trustStore`）；
- 字体：PDF/报表导出缺 fontconfig → 中文变方块；
- 字符集：`LANG/LC_ALL` 缺省导致 `file.encoding` 非 UTF-8（Java 18+ 默认 UTF-8，老版本仍有坑）——`-Dfile.encoding=UTF-8` 兜底；
- **DNS**：容器内解析走 Docker 内嵌 DNS（127.0.0.11，2.2.1）——**JVM 的 DNS 缓存**在服务发现频繁变 IP 时会导致"连旧 IP"（`networkaddress.cache.ttl` 调小或依赖客户端重解析）；alpine 的 **musl 解析器与 glibc 行为差异**（搜索域、hosts 优先级）也是"本地能跑、容器里超时"的常见嫌疑。

**⑤ 文件描述符**：容器继承 daemon 的 ulimit（默认常 1024）——高并发 Netty/Tomcat 的 `Too many open files` 先查 `ulimit -n` 与 `--ulimit nofile`（5_tomcat.md 的连接数账在此换了个层）。

**收口**："Java 容器化的坑可以归纳成一句话——**JVM 看到的世界（内存、CPU、时钟、DNS、证书）由容器边界重新定义**，每一项都要显式对齐，不能吃默认值。"

### 5.1.2 日志、监控与重启策略怎么设计？

**日志：stdout 是契约，不是习惯**
- 原则：应用**只写 stdout/stderr**，日志的落盘、轮转、采集交给平台（12-factor）——写容器内文件=CoW 膨胀 + 删容器即丢 + 采集要进容器捞（2.2.1）；
- 默认 `json-file` 驱动**不轮转 → 无上限增长**（宿主盘被日志吃满的头号原因，5.2.1）：daemon.json 配 `log-opts: max-size=100m, max-file=3`（或换 `local` 驱动，自带轮转与压缩、性能更好）；
- **多行日志的坑**：Java 堆栈一行一条 → `docker logs`/采集侧被切成 N 条——结构化 JSON 日志（logback JSON encoder，字段含 traceId，15_spring_cloud.md 4.2.1）或采集侧多行合并规则；
- 集中化：Fluent Bit/Filebeat 以 DaemonSet/宿主级采集（32_kubernetes.md 的日志栈），或应用直接推 Loki/ES；`docker logs` 只当"单机急救"，不是观测体系（30_linux.md 的日志治理同源）。

**监控：容器层 + JVM 层两套视角**
- 容器层：`docker stats`（秒级粗看）→ cAdvisor/Prometheus 常驻；四组黄金指标——**CPU 使用与节流、内存 working set vs limit、OOM 事件、重启次数**（5.1.1 的 ①② 在此变成告警）；
- JVM 层：Actuator/Micrometer 暴露堆、GC、线程、连接池（10_springboot.md 5.1.1）；两层的关联点=**"容器 OOM 但 JVM 堆不满"**（堆外/线程栈/直接内存，5.1.1 ①）与**"CPU 节流导致 GC 停顿"**（5.1.1 ②）；
- **健康检查分两种语义**：`HEALTHCHECK`（Docker/Swarm 消费，3.1.2）与 K8s 的三探针（liveness 重启、readiness 摘流、startup 慢启动豁免）——**liveness 配太激进会把"启动慢"变成"无限重启"**，这是 K8s 高频事故（32_kubernetes.md）。

**重启策略：区分"崩了"与"人停了"**

| 策略 | 行为 | 适用 |
|------|------|------|
| `no`（默认） | 不重启 | 一次性任务/调试 |
| `on-failure[:N]` | 非 0 退出重启（上限 N 次） | 批处理/可容忍失败的服务 |
| `always` | 任何退出都重启（含守护进程重启后） | 常驻服务（但"手动 stop"后也会被拉起） |
| `unless-stopped` | 除"手动停止"外都重启 | **单机常驻服务的推荐值** |

**重启是退避的**（指数级拉长，封顶分钟级）——所以 `docker ps` 里"Restarting (1) 3 seconds ago"是 CrashLoop 的早期信号；**排查顺序**：`docker inspect` 的 `State.ExitCode/OOMKilled` → `docker logs --tail 200` → `docker events` 时间线（5.2.1 的故障表同源）。**注意**：容器"不健康"（HEALTHCHECK 失败）**不会**触发 docker 重启——自愈要靠编排层或 autoheal 类旁挂，这是"配了 healthcheck 仍不重启"的标准答案。

## 5.2 运维与加固

### 5.2.1 磁盘被 Docker 吃满了怎么查？清理纪律怎么定？

**排查五指山（按命中率排）**：
1. **先分层看账**：`docker system df -v`（镜像/容器/卷/构建缓存各占多少）——比 `du -sh /var/lib/docker` 快且能定位类型；
2. **日志**：`/var/lib/docker/containers/*/*-json.log` 体积——没配 max-size（5.1.2）的默认驱动是头号惯犯；
3. **悬空镜像与构建缓存**：`docker images -f dangling=true`、BuildKit 的 `docker builder prune`（3.2.1 的 cache mount/registry cache 都会攒）；
4. **匿名卷**：`docker volume ls -f dangling=true`——`VOLUME` 指令与 compose 默认卷留下的"无名卷"（1.2.2）；
5. **容器可写层**：`docker ps -s`（SIZE 列）+ `docker diff <容器>`——往容器层写数据/日志/临时文件（2.1.2 的坑清单）；
6. **inode 耗尽**（`df -i`）：大量小文件（node_modules、日志碎片）在 overlay 下按层放大——磁盘有空间但写不进去的暗坑。

**清理命令与"纪律"**（命令人人会背，纪律才是分水岭）：
- `docker system prune`（悬空镜像+停止容器+未用网络+构建缓存）、`-a` 连未使用的镜像一起、`--volumes` 连匿名卷（**生产慎用：可能删掉有数据的卷**）、`--filter until=168h` 按时间限缩；
- `docker image prune -a --filter "until=720h"` 保留两周内镜像（回滚窗口）；
- **纪律四条**：① **日志轮转在 daemon 层统一配**（不靠人清）；② prune 走**定时任务 + 维护窗口 + 排除生产卷**（自动化但保守）；③ **镜像保留策略在仓库侧做**（Harbor 的 tag 保留/配额，4.1.2——本地清理只是补救）；④ **容量水位告警**在吃到 80% 前触发（5.1.2 的监控面），别等 `No space left on device` 再救火；
- **根治手段**：`--storage-opt size`（xfs pquota 等）给容器层设限额、把"写数据"从容器层逐出（卷/tmpfs，2.1.2）、CI 里管镜像体积预算（3.2.2）。

**追问**："删了镜像/容器磁盘没降"——被其他镜像共享的层仍在用（1.2.3 的层共享）、whiteout 不回收、registry GC 未跑（4.1.1）；按这三点依次排除即闭环。

### 5.2.2 容器安全加固清单怎么列？

**按"镜像内 → 运行时 → 宿主 → 供应链"四层组织（比罗列参数更有说服力）**：

**① 镜像内（构建期）**：最小基础镜像（distroless/alpine 实测，3.2.2）、**非 root USER**（uid 数字，别用名字避免宿主映射歧义）、不在镜像里留密钥（`--mount=type=secret`，3.1.2）、固定 digest 拉基镜像、SBOM + 扫描门禁（4.1.2）。

**② 运行时（启动参数）**：
- `--read-only` + 需要写的目录挂 tmpfs/卷（把"改文件"这条攻击面关掉）；
- `--cap-drop ALL --cap-add` 按需（1.1.2 的 capabilities）；
- `--security-opt no-new-privileges`（禁 setuid 提权）、seccomp profile（默认已拦危险 syscall）、AppArmor/SELinux 标签；
- **资源限额**（`--memory/--pids-limit/--cpus`，1.1.2 的 cgroup）——既是稳定性也是防 DoS；
- **绝对禁止**：`--privileged`（≈宿主 root 全部能力）、挂 `docker.sock`（等于把引擎控制权交给容器——"用容器管理容器"场景要用只读代理或 CI 专用方案）、挂宿主 `/` 或 `/etc` 可写。

**③ 宿主与引擎**：**docker 组 = root 等价**（能起特权容器就能拿宿主）——生产把"能跑 docker"当 sudo 管理；引擎只监听 unix socket（远程 API 必须 TLS+鉴权）；内核与 runc 及时打补丁（共享内核的代价，1.1.1）；**rootless / userns-remap** 把容器 root 映射到宿主普通用户（兼容性与权限模型要评估）；防火墙接管（`DOCKER-USER` 链，2.2.2）。

**④ 供应链**：私有仓库 + 不可变 tag（4.1.1）、签名与准入（cosign + Kyverno/OPA，4.1.2）、镜像来源白名单、部署按 digest 审计。

**收口句**："容器安全不是一张 checklist，而是承认**共享内核 + 默认全 root** 这两个前提——所以路径永远是**三层收敛**：镜像更小（少东西可打）、进程更弱（非 root + 能力削减）、边界更硬（rootless/强隔离运行时）"；至于"强多租户要不要上 Kata/gVisor"（1.1.1 的谱系），答"按信任域决定，不以性能为名牺牲隔离"即完整。