
# 1. 基础题

## 1.1 什么是 Docker？容器和虚拟机的区别？

Docker 是一种容器化平台，用于把应用及其运行依赖打包成标准化镜像，并以容器方式运行。它的核心价值在于标准化交付和环境一致性，而不是单纯“把程序关进一个沙箱”。

容器和虚拟机的本质区别是隔离边界不同：容器共享宿主机内核，通过 Namespace 和 Cgroups 做进程级隔离；虚拟机通过 Hypervisor 模拟硬件，每个 VM 拥有独立内核，隔离更强但开销也更高。

答题时可以从四个点展开：

1. 隔离级别不同。
2. 启动速度不同。
3. 资源开销不同。
4. 使用场景不同。

## 1.2 Docker 的核心组件有哪些？

- **Docker CLI**：用户命令入口，如 `docker run`、`docker build`。
- **dockerd**：Docker 守护进程，负责接收 API 请求和管理资源。
- **containerd**：容器运行时，负责容器和镜像生命周期。
- **runc**：OCI 低级运行时，负责真正创建 Linux 容器。
- **Docker Compose**：多容器编排工具，适合本地和小规模部署。

答这题时最好补一句：Docker 不是只有一个程序，而是由上层控制面和下层运行时协作完成容器创建。

## 1.3 什么是 Dockerfile？CMD 和 ENTRYPOINT 的区别？

Dockerfile 是镜像构建脚本，用于声明镜像如何生成。它不仅定义了应用怎么启动，还决定了镜像层数、缓存利用率、安全性和可维护性。

`CMD` 是默认命令或默认参数，可被 `docker run` 覆盖；`ENTRYPOINT` 定义容器主入口，一般不轻易覆盖，`CMD` 常作为它的默认参数。

答题时可以给出一个例子：

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
CMD ["--spring.profiles.active=prod"]
```

这样用户可以覆盖参数，但不会覆盖主入口。

# 2. 原理题

## 2.1 Docker 使用了 Linux 的哪些内核特性？

- **Namespace**：提供隔离视图，让容器“看起来”有独立系统环境。
- **Cgroups**：限制和统计资源使用，让容器“最多只能用这么多”。
- **UnionFS / OverlayFS**：提供镜像分层和写时复制，让镜像共享底层数据并支持可写层。

这是 Docker 原理的核心三件套。Namespace 负责隔离，Cgroups 负责限制，UnionFS 负责分层存储。

## 2.2 Docker 镜像的分层机制是什么？

每个 Dockerfile 指令通常会生成一个只读层。镜像由多个只读层叠加组成，容器启动时在最上面增加一个可写层。文件修改时采用 Copy-on-Write：先复制到可写层，再在可写层修改，不直接改底层只读层。

答题时可以补充分层的好处：

1. 多个镜像可复用基础层。
2. 构建可以命中缓存。
3. 镜像分发更高效。

## 2.3 Docker 网络有哪几种模式？

| 模式 | 说明 |
|------|------|
| bridge | 默认网络模式，基于 NAT 的网桥网络 |
| host | 容器直接使用宿主机网络 |
| none | 无网络 |
| overlay | 跨主机虚拟网络，常用于 Swarm/K8s 场景 |
| macvlan | 给容器分配物理网络上的独立 MAC 地址 |

答题时可以说明：bridge 适合大多数场景，host 适合极致性能或特殊网络需求，overlay 适合跨主机通信。

## 2.4 多阶段构建的好处是什么？

多阶段构建把构建环境和运行环境分离。构建阶段使用大镜像，包含 Maven、Gradle、JDK 和编译工具；运行阶段只保留最终产物和必要运行时依赖，因此镜像更小、攻击面更少、缓存更友好。

# 3. 实践题

## 3.1 如何优化 Docker 镜像大小？

1. 使用多阶段构建，构建工具不进入最终镜像。
2. 选择合适的基础镜像，优先考虑 JRE 或 slim 版。
3. 合并 `RUN` 指令并清理缓存。
4. 使用 `.dockerignore` 排除无关文件。
5. 避免在镜像中保留源码、测试文件和构建产物。

## 3.2 容器中 Java 应用的内存如何配置？

```bash
-XX:+UseContainerSupport          # 感知容器限制
-XX:MaxRAMPercentage=75.0         # 堆占容器内存的比例
-XX:+ExitOnOutOfMemoryError       # OOM 时快速退出，利于重启
```

答题时最好补充：Java 容器内存不是只看堆，还要考虑 Metaspace、线程栈、Direct Buffer、GC 结构和 Agent 占用。

## 3.3 如何调试容器启动失败？

1. `docker logs <container>` 看应用日志。
2. `docker ps -a` 看退出码。
3. `docker inspect` 看入口命令、环境变量、挂载和健康状态。
4. `docker run -it --entrypoint sh image` 进入容器手动排查。

# 4. 架构题

## 4.1 Docker 和 Kubernetes 的关系？

- Docker 是容器构建和运行平台。
- Kubernetes 是容器编排平台，负责调度、扩缩容、服务发现和自愈。
- Kubernetes 现在通常使用 containerd 作为运行时，不再直接依赖 Docker Engine。
- Docker 构建的 OCI 镜像仍然可以在 Kubernetes 中运行。

答题时要强调：Docker 和 Kubernetes 不是竞争关系，而是层次不同、职责不同。

## 4.2 Docker 的数据卷有哪几种？如何选择？

| 类型 | 适用场景 |
|------|---------|
| Volume | 生产推荐，Docker 统一管理，可迁移和备份 |
| Bind Mount | 开发调试，直接映射宿主机文件 |
| tmpfs | 临时敏感数据，只存内存，不落盘 |

答题时可补充：有状态应用优先使用 Volume，不要把关键数据直接写进容器可写层。

## 4.3 如何保证 Docker 容器的安全？

- 使用非 root 用户运行。
- 只读文件系统或最小化可写目录。
- 删除不必要能力，如 `--cap-drop=ALL`。
- 扫描镜像漏洞。
- 不在镜像中存储密钥。
- 限制资源，避免单容器拖垮宿主机。

# 5. 场景题

## 5.1 如何处理容器日志？

1. 应用日志输出到 stdout/stderr。
2. 使用 Docker logging driver 统一收集和轮转。
3. 生产环境接入 ELK、EFK、Loki 等日志系统。
4. 不要把 `docker logs` 当成长期日志归档方案。

## 5.2 Docker Compose 中如何控制服务启动顺序？

使用 `depends_on` 配合 `healthcheck`，让基础服务先启动并通过健康检查，再启动依赖服务：

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

要注意，`depends_on` 解决的是启动顺序，不等于业务就一定可用。真正可用性还要看服务本身是否完成初始化。

# 6. 高频追问

## 6.1 为什么容器比虚拟机轻量？

因为容器共享宿主机内核，不需要为每个实例启动完整操作系统。它本质上只是被 Namespace 和 Cgroups 限制的进程组。

## 6.2 为什么容器启动快？

因为容器不需要启动内核和完整 OS，只需准备文件系统、Namespace、Cgroups 和进程环境。

## 6.3 容器为什么不一定比 VM 更安全？

因为容器共享宿主机内核，隔离边界更薄。容器安全依赖镜像最小化、权限控制、资源限制和运行时加固。

## 6.4 Dockerfile 为什么要尽量小？

因为镜像越小，拉取越快，启动越快，攻击面越小，也更容易分发和回滚。

# 7. 面试速记总结

可以用这套关键词串起来回答 Docker 相关问题：

```text
Docker = 容器化平台
核心 = 镜像 + 容器 + 仓库 + Dockerfile
原理 = Namespace + Cgroups + UnionFS
优势 = 轻量、快速、标准化交付
局限 = 共享内核、隔离弱、编排需要额外平台
实践 = 多阶段构建、非 root、健康检查、资源限制、日志规范
```

# 8. 面试答题模板

遇到开放题时，可以按这个结构回答：

1. 先给定义。
2. 再讲原理。
3. 然后讲实际使用方式。
4. 最后讲优缺点和场景。

这样回答通常会更完整，也更符合面试官希望听到的“知识闭环”。
