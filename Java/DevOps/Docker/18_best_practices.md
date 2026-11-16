
# 1. 这篇的定位

`8_dockerfile_best.md` 讲"怎么写好一个 Dockerfile"，`12/13/16` 分别讲资源、安全与可观测，这一篇把它们收敛成**一张上线前的决策清单**：每一条都是"不照做就会在某个深夜付出代价"的工程纪律。按交付流水线顺序组织：Build → Run → Data → Network → Security → Observability → JVM 特坑 → Checklist。

# 2. 镜像构建（Build）

1. 多阶段构建：构建工具不进入运行镜像。
2. 基础镜像固定版本（必要时用 digest 锁定关键环境）。
3. 合理分层：先复制依赖描述文件再下载依赖，提升缓存命中。
4. `.dockerignore`：排除 `.git`、构建产物、密钥、无关文件。
5. 不在镜像里写密钥：避免 `ENV/ARG` 固化敏感信息。

层顺序的本质是**变更频率排序**：镜像层缓存从第一条失效指令开始全部重建，所以"每次构建都变的"（源码）永远放最后，"几个月才变一次"（JRE、依赖）放最前。`.dockerignore` 常被遗忘，它同时解决三件事：构建上下文上传变慢（monorepo 里 .git 几个 GB 全量传给 daemon）、敏感文件意外进镜像、缓存被无关文件（IDE 配置、日志）频繁击穿。

```text
# .dockerignore 最小集
.git/  target/  build/  *.log  .idea/  node_modules/
.env  *.pem  Dockerfile  docker-compose*.yml
```

# 3. 容器运行（Run）

1. 明确入口：ENTRYPOINT/CMD 清晰，不要在 entrypoint 里写复杂 shell 逻辑。
2. PID 1 行为：正确处理信号；必要时使用 `--init`。
3. 资源限制：至少设置内存上限与 CPU 上限；防止邻居噪音。
4. 重启策略：用 `restart: unless-stopped`（或根据场景选择），避免异常退出无人接管。
5. 健康检查：为关键服务提供轻量 healthcheck。

```bash
docker run -d --name order \
  --init \                          # tini 接管 PID1: 转发信号+回收僵尸
  --memory=1g --memory-swap=1g \    # 生产禁 swap
  --cpus=2 \
  --restart unless-stopped \
  --read-only --tmpfs /tmp \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --log-opt max-size=10m --log-opt max-file=3 \
  --health-cmd='curl -fsS http://localhost:8080/actuator/health || exit 1' \
  --health-start-period=40s \
  harbor.corp.com/pay/order-service:1.2.3-8f3c2a1
```

**PID 1 的特殊语义**是运行期最大的暗坑：内核默认**不向 PID 1 发送没有注册 handler 的信号**，shell 包装的入口命令（`CMD java -jar app.jar` 这种 shell 形式实际是 `/bin/sh -c`）会吞掉 SIGTERM——`docker stop` 等满 10 秒后收到 SIGKILL，数据写一半。解法三选一：exec 形式 ENTRYPOINT、`--init`（tini 转发信号）、或自带 shell 的镜像确保 `exec java`。

# 4. JVM in Container：五个高频坑

这是 Java 团队事故密度最高的区域，每个坑给"现象 → 根因 → 处方"：

1. **按宿主机内存设堆**。现象：512M 容器启动即 137。根因：JDK 8u191 前无容器感知，`-Xmx` 默认按宿主机 1/4 内存。处方：升级 JDK 17/21（容器感知默认开）+ `MaxRAMPercentage=60`。
2. **线程栈挤爆内存**。现象：压测一段时间后 OOMKilled，堆 dump 却显示堆很空。根因：线程数 × `-Xss`（默认 1M）+ DirectBuffer + Metaspace 在堆外。处方：`-XX:NativeMemoryTracking=summary` + `jcmd VM.native_memory` 核对总量，线程数压到与 CPU limit 匹配。
3. **availableProcessors 失真导致 GC/线程池超配**。现象：CPU limit 2 核但 GC 线程 32 个，throttle 严重、P99 毛刺。根因：老 JDK 读宿主机核数。处方：JDK 10+ 或显式 `-XX:ActiveProcessorCount=2`，线程池大小按 limit 而非机器核数配置。
4. **信号处理不当丢数据**。现象：滚动发布时最后一批请求 500、消息重复消费。根因：shell 吞 SIGTERM（见第 3 节）+ 无 graceful shutdown。处方：exec 形式入口 + `server.shutdown=graceful`（Spring Boot 2.3+）+ `docker stop -t 30` 超时大于应用排空时间 + K8s terminationGracePeriodSeconds 同步调大。
5. **时钟与时区、熵**。现象：日志时间与监控对不上、启动慢几秒。处方：`TZ=Asia/Shanghai` 环境变量或挂载 localtime，容器镜像内预装 `tzdata`； SecureRandom 用 `-Djava.security.egd=file:/dev/./urandom` 避免阻塞在低熵环境。

> **注意**：这五个坑的共同根因是"**JVM 以为自己独占一台机器**"。凡是 JVM 感知宿主资源的入口（堆、GC 线程、编译器线程、默认线程池 ForkJoinPool），都要与 cgroup 限制显式对齐，K8s 里同理。

# 5. 优雅停机：SIGTERM + 1 的完整链路

"优雅"不是应用层口号，而是一条时间轴上的协作：

```text
t=0    docker stop / K8s 删除 Pod
       ├─ kubelet: 并行执行 preStop hook + 向容器 PID1 发 SIGTERM
       ├─ 同时 endpoint 摘除传播(网关/Service 不再送新请求, 有秒级延迟)
t=0+   JVM ShutdownHook: 拒新请求 → 排空在途请求 → 关连接池/消费者
       Spring Boot: server.shutdown=graceful, spring.lifecycle.timeout-per-shutdown-phase=20s
t=+30s 超时阈值: 应用仍未退出 → SIGKILL(强杀, 数据完整性靠 WAL/幂等兜底)
```

**设计考量**：

- K8s 里 `terminationGracePeriodSeconds(30)` 必须 **≥** 应用排空时间 + preStop sleep；preStop `sleep 5-10s` 是为了让 endpoint 摘除先于 SIGTERM 生效，否则会出现"已经开始关停但还有新请求打进来"的窗口。
- Docker 单机对应 `docker stop -t 30`，默认 10s 对慢排空应用不够。
- 消费型服务（Kafka/RabbitMQ listener）关停顺序：先停消费、处理完在途消息、提交 offset 再退出，顺序反了就是重复消费或丢消息。

# 6. 数据与存储（Data）

1. 状态数据用命名 volume（数据库、队列等）。
2. 配置与证书用 bind mount 且只读。
3. 备份恢复有脚本与演练：不要等故障才第一次恢复。
4. 谨慎使用 `docker compose down -v`：它会删除 volume。

镜像可丢弃、容器可重建、**只有 volume 里的数据不可再生**——这条纪律的检验标准是：任意一台机器全损，你能否在目标 RTO 内用"新机器 + 镜像 + 备份"复原服务？

# 7. 网络与安全（Network / Security）

网络：

1. 最小暴露：只暴露入口服务端口，内部依赖不做 `ports` 映射。
2. 服务间互访用内部网络与服务名。
3. 明确边界：区分 internal 网络与对外网络（如需要）。

安全：

1. 禁止 `--privileged`（除非你能写出理由和补偿措施）。
2. 以非 root 运行，capabilities 最小化（`cap-drop=ALL` 再按需添加）。
3. 根文件系统只读（可行则启用），可写目录显式挂载。
4. 不挂载 docker.sock 到业务容器。
5. 镜像扫描与基础镜像升级纳入常规流程。

# 8. 日志与可观测（Ops）

1. 日志输出到 stdout/stderr。
2. 配置日志轮转（json-file 必配）。
3. 发生故障优先用：`docker logs`、`docker inspect`、`docker stats`、`docker events`。

日志里带 traceId/MDC、GC 日志开出来（`-Xlog:gc*:file=/dev/stdout` 或独立卷）、Actuator 暴露指标，这三件事决定了半夜排障的速度。

# 9. 镜像最小化：运行时镜像选型对照

| 基础镜像 | 大小(参考) | shell | 攻击面 | 排障能力 | 适用 |
|----------|-----------|-------|--------|----------|------|
| `eclipse-temurin:17-jre` (Ubuntu) | ~260MB | 有 | 大（完整发行版） | 强 | 需要工具链的排障环境 |
| `eclipse-temurin:17-jre-alpine` | ~80MB | busybox | 中（musl 兼容坑） | 中 | 通用, 验证 musl 兼容性后 |
| `gcr.io/distroless/java17` | ~300MB(含JRE) | **无** | 极小 | 弱(靠 debug tag/临时 sidecar) | 生产强基线 |
| `amazoncorretto:17-alpine` | ~100MB | busybox | 中 | 中 | AWS 环境 |
| `springio/graalvm...`(native 二进制+`scratch`) | **~50MB** | 无 | 最小 | 弱 | GraalVM Native Image 服务 |

> **行业实践**：Distroless 的"没有 shell"同时是安全优势与运维代价——`docker exec -it <c> sh` 直接不可用。标准补偿：临时调试镜像 `gcr.io/distroless/java17:debug`（带 shell），或 K8s ephemeral debug container 注入；排障能力没有消失，只是"按需注入"。用 cosign/SBOM/扫描保证"小即安全可证"，而不是靠发行版工具链兜底。

# 10. Compose 使用习惯（单机场景）

1. `depends_on` 只保证启动顺序；配合 `healthcheck` 才能更接近"依赖就绪"。
2. 多环境用 override 文件或 profiles，不要复制多份完全相同的 compose。
3. 用 `docker compose config` 检查最终渲染结果（变量替换/合并覆盖）。
4. 项目名固定（顶层 `name:` 字段），避免目录改名导致网络/卷分裂。

# 11. 上线前 Checklist（逐项打勾）

```text
镜像构建
  [ ] 多阶段构建, 运行层无构建工具与源码
  [ ] 基础镜像固定版本; trivy 扫描 HIGH/CRITICAL 通过
  [ ] .dockerignore 生效; 镜像内无密钥(gitleaks/docker history 复查)
  [ ] 非 root USER 且 UID 固定; 文件属主正确(--chown)
  [ ] SBOM 已生成并随制品归档

运行配置
  [ ] ENTRYPOINT exec 形式(PID1 能收到 SIGTERM)
  [ ] 内存 limit + Xmx/MaxRAMPercentage 匹配; CPU limit 与线程池配置匹配
  [ ] --init 或应用自行处理信号; stop 超时 > 应用排空时间
  [ ] restart 策略符合场景; healthcheck 含 start-period 且指向轻量端点
  [ ] 日志 driver 轮转参数已配(max-size/max-file)

数据与网络
  [ ] 有状态数据在命名 volume; 备份脚本演练过(做过一次真实恢复)
  [ ] 仅入口服务发布端口; 内部服务走自定义网络+服务名

安全基线
  [ ] 无 privileged / docker.sock / hostPath 根挂载
  [ ] cap-drop=ALL + 最小 cap-add; seccomp/apparmor 默认档未关闭
  [ ] rootfs 只读(或 K8s PSA restricted 达标)

发布
  [ ] tag 策略: sha-tag 不可变, 生产按 digest 部署
  [ ] 回滚路径演练过(切回上一 digest 的时间 < RTO)
  [ ] 监控面板与告警规则覆盖: OOM/重启/throttle/P99
```

# 12. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器化 Java 有哪些坑 | 堆按宿主机算、线程栈堆外、availableProcessors、信号丢失、时区熵 |
| 为什么 PID 1 特殊 | 默认忽略未注册 handler 的信号, shell 入口吞 SIGTERM → SIGKILL 强杀 |
| 优雅停机怎么做 | SIGTERM → 排空在途 → 超时才 SIGKILL; preStop sleep 躲摘除延迟 |
| distroless 好在哪 | 无 shell 无包管理器攻击面最小; 代价是排障需 debug 镜像/sidecar |
| 上线 checklist 关键项 | 扫描过、非 root、limit 匹配、信号正确、轮转配置、digest 可回滚 |

# 13. 小结

最佳实践的本质是把三件事做到位：**边界（limit/security/只读）、契约（exec 入口/信号/健康检查/日志 stdout）、可追溯（不可变 tag、digest、SBOM、备份演练）**。JVM 五坑与优雅停机是 Java 团队的主战场，根因都是"JVM 与编排器对资源/生命周期的假设不一致"——容器把机器参数交给 cgroup，应用就必须按 cgroup 规划线程、堆与关停时间。镜像选型上"越小越安全"成立的前提是排障手段同步就位（debug 镜像、ephemeral container、指标先行）。第 11 节的 checklist 建议固化进 CI（脚本化 lint：检查 USER、privileged、轮转参数），人肉清单会退化，流水线不会。替代品与生态之争（Podman、containerd、CRI）见 `20_docker_vs_others.md`，性能与容量话题见 `21_performance.md`。
