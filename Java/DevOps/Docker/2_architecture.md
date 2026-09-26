
# 1. 整体架构

Docker 不是一个单体程序，而是一组组件协作完成镜像构建、容器创建、网络管理、存储管理和生命周期控制。用户通常通过 `docker` CLI 操作 Docker，CLI 通过 Docker Engine API 与 `dockerd` 通信，`dockerd` 再调用 `containerd` 和 `runc` 完成真正的容器运行。

```text
┌──────────────────────────────────────────────────────────────┐
│                     Docker Engine                             │
│                                                              │
│  ┌──────────────┐                                            │
│  │  Docker CLI  │  REST API / Unix Socket                    │
│  │ docker ...   │ ───────────────────────────────┐           │
│  └──────────────┘                                │           │
│                                                  ▼           │
│  ┌────────────────────────────────────────────────────┐      │
│  │                  dockerd                            │      │
│  │  Image Manager / Network Manager / Volume Manager   │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │ gRPC                              │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────┐      │
│  │                 containerd                          │      │
│  │  Image Service / Container Service / Task Service   │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │ Shim API                           │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────┐      │
│  │              containerd-shim                        │      │
│  │  管理容器进程、STDIO、退出状态                       │      │
│  └───────────────────────┬────────────────────────────┘      │
│                          │ OCI Runtime                        │
│                          ▼                                   │
│  ┌────────────────────────────────────────────────────┐      │
│  │                    runc                             │      │
│  │  创建 Namespace、配置 Cgroups、启动容器进程          │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

各组件职责如下：

| 组件 | 职责 |
|------|------|
| Docker CLI | 用户命令入口，如 `docker run`、`docker build` |
| dockerd | Docker 守护进程，负责 API、镜像、网络、卷和容器管理 |
| containerd | 高级容器运行时，负责镜像、容器、任务生命周期 |
| containerd-shim | 容器进程的垫片进程，解耦 containerd 和容器进程 |
| runc | OCI 低级运行时，真正创建 Linux 容器 |

# 2. `docker run` 内部流程

执行一条 `docker run nginx` 时，Docker 内部大致会经历以下流程：

1. Docker CLI 将命令转换为 REST API 请求，发送到 `/var/run/docker.sock`。
2. `dockerd` 校验参数，检查本地是否存在 `nginx` 镜像。
3. 如果本地没有镜像，`dockerd` 从 Registry 拉取镜像层。
4. `dockerd` 准备容器配置，包括环境变量、端口、挂载、网络和资源限制。
5. `dockerd` 调用 `containerd` 创建容器元数据和运行任务。
6. `containerd` 启动 `containerd-shim`。
7. `containerd-shim` 调用 `runc`。
8. `runc` 创建 Namespace、配置 Cgroups、挂载根文件系统并启动容器进程。
9. 容器进程启动后，`runc` 退出，`containerd-shim` 继续管理容器进程。

这个流程说明了一个关键点：`runc` 不是常驻进程，它只负责创建容器；容器运行期间主要由容器进程本身和 `containerd-shim` 维持。

# 3. Docker API 与 Unix Socket

Docker CLI 默认通过 Unix Socket 访问 Docker Engine：

```bash
docker -H unix:///var/run/docker.sock ps
```

也可以通过 TCP 暴露远程 API，但生产环境必须启用 TLS，否则任何能访问该端口的人都可能获得宿主机级别权限：

```bash
docker -H tcp://192.168.1.100:2376 --tls ps
```

`/var/run/docker.sock` 本质上是 Docker 的控制入口。把它挂载到容器中，相当于让该容器可以控制宿主机 Docker，风险很高。CI/CD 工具如果需要构建镜像，应优先使用隔离构建器或最小权限方案，而不是随意挂载 Docker Socket。

# 4. Namespace：隔离系统视图

Linux Namespace 负责让容器进程“看起来”运行在独立环境中。不同 Namespace 隔离不同系统资源，组合起来形成容器的隔离视图。

| Namespace | 隔离内容 | 效果 |
|-----------|----------|------|
| PID | 进程 ID 空间 | 容器内进程看到自己的 PID 树，通常主进程是 PID 1 |
| NET | 网络设备、IP、端口、路由表 | 容器拥有独立网络栈 |
| MNT | 文件系统挂载点 | 容器拥有独立根文件系统 |
| UTS | hostname、domainname | 容器可设置独立主机名 |
| IPC | 信号量、消息队列、共享内存 | 隔离进程间通信资源 |
| USER | 用户 ID 和组 ID | 容器内 root 可映射为宿主机非 root |
| CGROUP | Cgroup 视图 | 隐藏或隔离部分 cgroup 层级 |

查看容器 Namespace：

```bash
docker inspect <container> | jq '.[0].State.Pid'
ls -la /proc/<pid>/ns/
```

Namespace 解决的是“看见什么”的问题。例如 PID Namespace 让容器内只能看到自己的进程，NET Namespace 让容器拥有独立网卡和路由表。但 Namespace 并不限制容器“能用多少资源”，资源限制由 Cgroups 负责。

# 5. Cgroups：限制资源使用

Cgroups 是 Control Groups 的缩写，用来限制、统计和隔离进程组的资源使用。Docker 通过 Cgroups 控制容器能使用多少 CPU、内存、IO 和进程数，避免单个容器耗尽宿主机资源。

```bash
# 限制容器最多使用 512MB 内存
docker run -m 512m my-app

# 限制容器最多使用 2 个 CPU
docker run --cpus=2 my-app

# 限制进程数量，避免 fork bomb
docker run --pids-limit=200 my-app
```

| Cgroup 子系统 | 控制资源 |
|---------------|----------|
| cpu | CPU 使用比例和调度权重 |
| cpuacct | CPU 使用统计 |
| cpuset | 绑定可使用的 CPU 核心 |
| memory | 内存、Swap、OOM 行为 |
| blkio | 块设备 IO 权重和限制 |
| pids | 进程数量上限 |

Cgroups 解决的是“最多能用多少”的问题。对于 Java 应用尤其重要，因为 JVM 堆、Metaspace、线程栈、Direct Buffer 等都会消耗容器内存。如果只设置 `-Xmx` 而不考虑堆外内存，很容易触发容器 OOM。

# 6. UnionFS：镜像分层和写时复制

Docker 镜像由多层只读层叠加而成，每个 Dockerfile 指令通常会生成一个镜像层。容器启动时，Docker 会在镜像只读层之上添加一个可写层，容器运行期间产生的文件修改默认写入这个可写层。

```text
容器文件系统：
┌──────────────────────────────┐
│  Container Layer 可写层       │  容器运行时修改写入这里
├──────────────────────────────┤
│  Image Layer 3 CMD           │  只读
├──────────────────────────────┤
│  Image Layer 2 COPY . /app   │  只读
├──────────────────────────────┤
│  Image Layer 1 RUN apt ...   │  只读
├──────────────────────────────┤
│  Base Layer FROM openjdk     │  只读
└──────────────────────────────┘
```

Copy-on-Write 的过程如下：

1. 容器读取文件时，直接从镜像只读层读取。
2. 容器第一次修改某个只读层文件时，Docker 会先把该文件复制到容器可写层。
3. 后续修改都发生在可写层，不影响原始镜像。
4. 删除文件时，通常通过 whiteout 标记隐藏底层文件，而不是修改底层镜像层。

镜像分层带来两个好处：一是复用基础层，多个镜像可以共享相同层；二是构建缓存，未变化的层可以直接复用。但层设计不合理也会导致缓存失效或镜像膨胀，例如先 `COPY . .` 再下载依赖，会让源代码每次变更都触发依赖重新下载。

# 7. 存储驱动

Docker 通过存储驱动实现镜像层和容器可写层管理。Linux 上主流选择是 `overlay2`。

| 驱动 | 特点 | 建议 |
|------|------|------|
| overlay2 | 主流、稳定、性能好，依赖 OverlayFS | 生产推荐 |
| aufs | 早期常用，现代环境逐渐淘汰 | 不推荐新项目 |
| devicemapper | 基于块设备映射，配置复杂 | 不推荐新项目 |
| btrfs | 依赖 btrfs 文件系统能力 | 特定场景使用 |
| zfs | 依赖 zfs，快照能力强 | 特定场景使用 |

查看当前存储驱动：

```bash
docker info | grep "Storage Driver"
```

# 8. containerd-shim 的作用

`containerd-shim` 的存在是为了解耦容器进程和 containerd。它通常承担以下职责：

1. 作为容器进程的父进程或管理进程，维持容器生命周期。
2. 保持 STDIO 流，支持 `docker logs`、`docker attach` 等能力。
3. 收集容器退出状态，上报给 containerd。
4. 让 containerd 重启或升级时，不必直接杀死所有容器进程。

```text
dockerd -> containerd -> containerd-shim -> runc -> container process
```

容器启动后，`runc` 通常会退出；`containerd-shim` 留下来管理已经启动的容器进程。因此排查进程树时，看到 shim 进程是正常现象。

# 9. 网络、存储、运行时的职责边界

Docker 架构中常见的三个能力边界如下：

| 能力 | 主要组件 | 说明 |
|------|----------|------|
| 运行时 | containerd、runc | 创建和管理容器进程 |
| 网络 | dockerd、libnetwork、iptables | 创建 bridge、端口映射、DNS、网络连接 |
| 存储 | dockerd、storage driver、volume driver | 管理镜像层、容器可写层、数据卷 |

排查问题时应先判断问题属于哪一层。例如容器进程启动失败通常看日志和入口命令；端口不通通常看网络、端口映射和防火墙；数据丢失通常看挂载方式和卷配置。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| Docker 架构包含哪些组件 | CLI、dockerd、containerd、containerd-shim、runc |
| `docker run` 发生了什么 | API 请求、拉镜像、创建配置、调用运行时、Namespace、Cgroups、启动进程 |
| Namespace 和 Cgroups 区别 | Namespace 管视图隔离，Cgroups 管资源限制 |
| UnionFS 的作用 | 镜像分层、层复用、写时复制、容器可写层 |
| containerd-shim 为什么存在 | 解耦运行时、维持 STDIO、收集退出状态、保持容器运行 |
