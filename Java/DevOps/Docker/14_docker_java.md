
# 1. Java 容器化的特点

Java 应用非常适合容器化，因为它通常以 jar 或 war 作为交付物，运行时依赖相对清晰。但 Java 容器化不能只把 jar 放进镜像然后 `java -jar`，还需要考虑 JVM 对容器资源的感知、堆内外内存、GC、启动信号、优雅关闭、镜像体积和线上排障。

Java 应用在容器中最常见的问题包括：

1. 容器内存限制为 1GB，但 JVM 按宿主机内存计算堆大小，最终被 OOM Kill。
2. `ENTRYPOINT` 使用 shell 形式，导致 SIGTERM 没有正确传给 Java 进程，应用不能优雅关闭。
3. 镜像中包含完整 JDK、Maven 仓库和源码，体积过大，漏洞面增加。
4. 使用 Alpine 镜像后遇到字体、时区、DNS、glibc 兼容性问题。
5. 日志写入容器文件系统，长期运行后撑满磁盘。

# 2. JDK 镜像选择

Java 镜像选择要同时考虑体积、兼容性、安全性和排障便利性。

| 镜像 | 特点 | 推荐场景 |
|------|------|----------|
| `eclipse-temurin:17-jre` | 兼容性好，基于常规 Linux 发行版 | 通用生产环境 |
| `eclipse-temurin:17-jre-alpine` | 体积较小，使用 musl libc | 对体积敏感且验证过兼容性的服务 |
| `eclipse-temurin:17-jdk` | 包含 JDK 工具 | 构建阶段、诊断阶段 |
| `maven:3.9-eclipse-temurin-17` | 包含 Maven 和 JDK | Docker 多阶段构建的 build stage |
| `amazoncorretto:17` | AWS 生态常用 | AWS 环境或公司标准镜像 |
| `gcr.io/distroless/java17` | 极简，无 shell | 高安全生产环境，排障依赖外部工具 |

Alpine 镜像体积小，但不是所有 Java 场景都适合。它使用 musl libc，某些依赖 glibc 的本地库、字体渲染、加密库、监控 Agent 或 JNI 组件可能出现兼容问题。生产环境选择 Alpine 前应经过完整压测和功能验证。

Distroless 镜像攻击面更小，但没有 shell、包管理器和常用调试工具。它适合成熟稳定的生产环境，不适合作为初期排障频繁的默认选择。

# 3. Spring Boot 多阶段 Dockerfile

Spring Boot fat jar 可以直接运行，但分层构建能更好地利用 Docker 缓存。依赖层变化频率低，业务代码变化频率高，分层复制可以减少镜像构建和推送成本。

```dockerfile
# Stage 1: 构建
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src/ ./src/
RUN mvn package -DskipTests

# Stage 2: 提取 Spring Boot 分层 jar
FROM eclipse-temurin:17-jre AS extractor
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 3: 运行
FROM eclipse-temurin:17-jre
RUN groupadd --system app && useradd --system --gid app app
WORKDIR /app

COPY --from=extractor --chown=app:app /app/dependencies/ ./
COPY --from=extractor --chown=app:app /app/spring-boot-loader/ ./
COPY --from=extractor --chown=app:app /app/snapshot-dependencies/ ./
COPY --from=extractor --chown=app:app /app/application/ ./

USER app
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

如果使用 Spring Boot 2.x，主类路径可能是 `org.springframework.boot.loader.JarLauncher`；Spring Boot 3.x 常见为 `org.springframework.boot.loader.launch.JarLauncher`。实际写法应以项目生成的分层结构和 Spring Boot 版本为准。

# 4. JVM 容器内存感知

早期 JVM 不一定能正确识别容器 Cgroup 限制，可能按宿主机总内存计算堆大小。现代 JDK 已经改进了容器感知能力，但仍建议显式配置内存比例，避免堆占满容器内存。

```bash
-XX:+UseContainerSupport
-XX:InitialRAMPercentage=50.0
-XX:MaxRAMPercentage=75.0
```

参数含义：

| 参数 | 说明 |
|------|------|
| `-XX:+UseContainerSupport` | 让 JVM 感知 Cgroup CPU 和内存限制 |
| `-XX:InitialRAMPercentage` | 初始堆占容器内存比例 |
| `-XX:MaxRAMPercentage` | 最大堆占容器内存比例 |
| `-XX:MinRAMPercentage` | 小内存场景下的堆比例控制 |

JDK 10+ 默认支持容器感知；JDK 8u191+ 支持但不同发行版默认行为可能不同。生产环境应通过启动日志或命令确认 JVM 识别到的内存限制。

```bash
java -XX:+PrintFlagsFinal -version | grep -E "UseContainerSupport|MaxRAMPercentage"
```

# 5. 容器内存如何计算

Java 进程使用的内存不只有堆。容器内存限制覆盖整个进程，因此需要给堆外内存预留空间。

```text
容器内存 = Java 堆 + Metaspace + 线程栈 + Direct Buffer + Code Cache + GC 结构 + JNI/Agent + 操作系统开销
```

示例：

```text
容器内存限制 = 2GB
MaxRAMPercentage = 75%
最大 Java 堆 ≈ 1.5GB
剩余 ≈ 512MB

剩余空间用于：
  - Metaspace
  - 线程栈
  - NIO Direct Buffer
  - JIT Code Cache
  - GC 内部结构
  - 监控 Agent 和本地库
```

如果服务线程很多、使用 Netty、Kafka Client、数据库连接池、APM Agent 或大量 Direct Buffer，`MaxRAMPercentage=75` 可能仍然偏高。内存紧张的服务应结合压测和 `jcmd`、NMT、容器监控指标进行调整。

# 6. 推荐 JVM 参数

通用生产参数示例：

```dockerfile
ENV JAVA_OPTS="-XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -XX:InitialRAMPercentage=50.0 \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:+ExitOnOutOfMemoryError \
    -Djava.security.egd=file:/dev/./urandom \
    -Duser.timezone=Asia/Shanghai"
```

如果需要通过环境变量动态传参，可以使用 shell 包装，但要注意信号转发问题。更推荐使用启动脚本并在脚本末尾使用 `exec`：

```sh
#!/bin/sh
exec java $JAVA_OPTS -jar /app/app.jar
```

Dockerfile：

```dockerfile
COPY docker-entrypoint.sh /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]
```

常用参数说明：

| 参数 | 说明 |
|------|------|
| `-XX:+UseContainerSupport` | 感知容器 Cgroup 限制 |
| `-XX:MaxRAMPercentage=75.0` | 设置最大堆占容器内存比例 |
| `-XX:+UseG1GC` | 服务端应用常用低暂停 GC |
| `-XX:+ExitOnOutOfMemoryError` | OOM 时退出进程，便于编排平台重启 |
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM 时生成 heap dump |
| `-Djava.security.egd=file:/dev/./urandom` | 避免旧环境随机数阻塞 |
| `-Duser.timezone=Asia/Shanghai` | 固定应用时区，避免环境差异 |

# 7. PID 1 与优雅关闭

容器中的主进程通常是 PID 1。Linux 中 PID 1 对信号和子进程回收有特殊行为。如果 Dockerfile 使用 shell 形式启动 Java，Java 进程可能不是 PID 1，SIGTERM 先到 shell，应用不一定能收到关闭信号。

推荐 exec 形式：

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

不推荐 shell 形式：

```dockerfile
CMD java -jar /app/app.jar
```

Spring Boot 开启优雅关闭：

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

优雅关闭的目标是在容器停止时完成以下动作：

1. 停止接收新请求。
2. 等待正在处理的请求完成。
3. 关闭线程池、连接池、消息消费者。
4. 在超时时间内退出进程。

运行容器时也要给足停止时间：

```bash
docker stop --time=30 my-app
```

# 8. CPU 感知与线程池

JVM 会根据可用 CPU 影响 GC 线程数、ForkJoinPool 并行度和部分框架默认线程数。容器中如果没有设置 CPU 限制，JVM 可能按宿主机 CPU 数计算，导致线程数过多。

```bash
docker run --cpus=2 -m 2g my-app
```

也可以显式设置 JVM 可用处理器数量：

```bash
-XX:ActiveProcessorCount=2
```

对于 Web 服务，除了 JVM 参数，还需要关注 Tomcat、Undertow、Netty、数据库连接池、消息消费者并发数。容器化后资源边界更明确，线程池配置也应随 CPU 和内存限制重新评估。

# 9. 日志与 GC 日志

容器应用日志建议输出到 stdout/stderr，由 Docker logging driver 或日志采集系统统一处理。不要把业务日志长期写在容器可写层中，否则容易造成磁盘膨胀，并且容器重建后日志丢失。

JDK 17 GC 日志示例：

```bash
docker run \
  -e JAVA_OPTS="-Xlog:gc*:stdout:time,uptime,level,tags" \
  my-app
```

如果必须写文件，应挂载 Volume，并配置滚动策略：

```bash
docker run \
  -v /var/log/my-app:/var/log/my-app \
  -e JAVA_OPTS="-Xlog:gc*:file=/var/log/my-app/gc.log:time,uptime:filecount=10,filesize=10M" \
  my-app
```

# 10. 健康检查

Spring Boot 应用通常可以使用 Actuator 作为健康检查入口：

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

生产环境需要区分“进程存活”和“服务可接流量”。例如应用进程启动后，依赖数据库、Redis 或配置中心还未准备好，此时不能直接接收流量。Kubernetes 中通常使用 readinessProbe 和 livenessProbe 分别表达这两种状态；单机 Docker 或 Compose 中可以用 `HEALTHCHECK` 辅助判断依赖启动顺序。

# 11. 常见问题排查

| 现象 | 可能原因 | 排查方向 |
|------|----------|----------|
| 容器退出码 137 | OOM Kill 或被 SIGKILL | `docker inspect`、`dmesg`、容器内存监控 |
| 停止容器很慢 | SIGTERM 未传递、应用未优雅关闭 | 检查 ENTRYPOINT、Spring Boot graceful shutdown |
| 镜像很大 | 构建工具和源码进入运行镜像 | 多阶段构建、`.dockerignore` |
| 启动慢 | 依赖初始化慢、随机数阻塞、类加载多 | 启动日志、JFR、Actuator startup |
| 字体或本地库异常 | Alpine musl 兼容问题、缺少字体包 | 更换 slim/glibc 镜像或补依赖 |
| 日志撑满磁盘 | 日志写入容器文件系统且无滚动 | stdout/stderr、logging driver、Volume |

OOM 排查示例：

```bash
docker inspect my-app | jq '.[0].State.OOMKilled'
docker stats --no-stream my-app
dmesg | grep -i oom
```

# 12. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| Java 容器化为什么要关注内存 | 容器限制覆盖整个进程，堆外内存也会触发 OOM |
| `MaxRAMPercentage` 的作用 | 按容器内存比例设置最大堆，避免固定 `-Xmx` 不灵活 |
| 为什么要用 exec 形式 ENTRYPOINT | Java 作为主进程接收 SIGTERM，支持优雅关闭 |
| 多阶段构建对 Java 的价值 | 构建环境和运行环境分离，减小镜像，降低攻击面 |
| Alpine 镜像有什么风险 | musl libc 兼容性、字体、本地库、Agent 问题 |
| OOM Kill 如何排查 | 退出码 137、`OOMKilled`、内核日志、JVM 堆和堆外分析 |
