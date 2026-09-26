
# 1. Dockerfile 的作用

Dockerfile 是镜像构建脚本，用声明式指令描述一个镜像如何生成。它不仅决定镜像能否运行，还会影响镜像大小、构建速度、缓存命中率、安全性和生产环境可维护性。

一个好的 Dockerfile 应该满足以下目标：

1. 构建结果可重复，同一份源码在相同输入下生成相同镜像。
2. 镜像尽量小，只包含运行时必需内容。
3. 构建速度快，能有效利用 Docker 构建缓存。
4. 默认安全，不以 root 运行，不包含密钥和无关工具。
5. 运行行为清晰，入口命令、端口、健康检查明确。

# 2. 指令顺序与构建缓存

Docker 构建镜像时会按 Dockerfile 指令顺序生成层，并对每一层做缓存。某一层缓存失效后，它后面的所有层都需要重新执行。因此 Dockerfile 的指令顺序非常重要。

基本原则是：变化频率低的内容放前面，变化频率高的内容放后面。

```dockerfile
# 推荐：先复制依赖描述文件，再下载依赖，最后复制源码
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /src

COPY pom.xml .
RUN mvn dependency:go-offline -B

COPY src/ ./src/
RUN mvn package -DskipTests
```

上面的写法中，只要 `pom.xml` 不变，依赖下载层就可以复用。日常开发中源码变化最频繁，但依赖变化相对少，这样可以显著减少构建时间。

反例：

```dockerfile
# 不推荐：源码任意文件变化都会导致依赖下载缓存失效
COPY . .
RUN mvn package
```

这种写法把所有内容一次性复制进镜像，任何源码、文档或临时文件变化都会让后续层缓存失效，构建速度会变慢。

# 3. 减少无意义层

Dockerfile 中的 `RUN`、`COPY`、`ADD` 等指令通常会创建新层。层本身不是越少越好，但无意义的层会增加镜像复杂度，并可能留下不必要的中间文件。

```dockerfile
# 不推荐：每个 RUN 创建一个层，缓存清理也在独立层中
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN rm -rf /var/lib/apt/lists/*
```

```dockerfile
# 推荐：安装和清理在同一层完成
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl vim && \
    rm -rf /var/lib/apt/lists/*
```

需要注意：如果在一个层中下载了缓存文件，又在后面的层中删除，镜像体积通常不会真正减少，因为文件已经存在于前面的只读层中。清理动作应该和产生缓存的动作放在同一个 `RUN` 中。

# 4. 多阶段构建

多阶段构建用于把“构建环境”和“运行环境”分离。构建阶段可以使用包含 Maven、Gradle、JDK、编译工具的大镜像；运行阶段只保留最终产物和运行时依赖。

```dockerfile
# Stage 1: 构建阶段，包含 Maven 和 JDK
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src/ ./src/
RUN mvn package -DskipTests

# Stage 2: 运行阶段，只包含 JRE 和 jar
FROM eclipse-temurin:17-jre-alpine AS runtime
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build --chown=appuser:appgroup /src/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

多阶段构建的收益：

1. 最终镜像不包含 Maven、Gradle、编译缓存和源码。
2. 镜像体积更小，拉取更快，攻击面更小。
3. 构建逻辑和运行逻辑在同一个 Dockerfile 中，易于维护。
4. CI/CD 中不需要在构建机预装完整语言环境。

# 5. `.dockerignore`

`.dockerignore` 用来控制哪些文件不发送到 Docker 构建上下文。它经常被忽视，但对构建性能和安全性非常重要。

```text
# .dockerignore
**/target/
**/build/
**/*.class
**/*.jar
**/.git
**/.gitignore
**/.idea/
**/*.iml
**/node_modules/
**/logs/
*.log
.env
Dockerfile
docker-compose.yml
```

如果没有 `.dockerignore`，Docker CLI 会把当前目录下的内容发送给 Docker daemon。大型项目中的 `target`、`.git`、`node_modules`、日志和临时文件会显著增加构建上下文大小，也可能把密钥、环境文件和内部文档带入镜像构建过程。

# 6. 基础镜像选择

基础镜像决定了镜像大小、安全补丁、系统库兼容性和排障便利性。不要只看体积，也要考虑应用依赖和运维能力。

| 场景 | 推荐镜像 | 特点 |
|------|----------|------|
| Java 通用生产 | `eclipse-temurin:17-jre` | 兼容性好，基于常规 Linux 发行版 |
| Java 小体积 | `eclipse-temurin:17-jre-alpine` | 体积小，但 musl libc 可能有兼容性问题 |
| Java 极简生产 | `gcr.io/distroless/java17` | 攻击面小，无 shell，排障不便 |
| 需要 glibc | `debian:bookworm-slim` | 兼容性较好，体积适中 |
| Go 静态二进制 | `scratch` | 极小，只适合无系统依赖二进制 |
| 通用轻量 | `alpine:3.19` | 小，但不是所有软件都完全兼容 musl |

生产环境建议固定基础镜像版本，避免使用 `latest`。`latest` 不是“最新稳定版”的语义，它只是一个普通标签，可能在不知情的情况下改变构建结果。

# 7. COPY 与 ADD

一般优先使用 `COPY`，因为语义明确：只复制文件。`ADD` 额外支持自动解压本地 tar 文件和从 URL 下载文件，但这些隐式行为容易让 Dockerfile 难以理解。

```dockerfile
# 推荐：普通文件复制使用 COPY
COPY target/app.jar /app/app.jar

# 只有确实需要自动解压本地 tar 时才考虑 ADD
ADD rootfs.tar.gz /
```

不建议用 `ADD` 下载远程 URL。远程下载更适合用 `RUN curl` 或 `RUN wget`，并配合校验 checksum，保证供应链安全和构建可重复性。

# 8. 非 root 用户运行

容器内默认 root 虽然不是宿主机 root 的完全等价物，但一旦存在容器逃逸、挂载敏感目录、错误授权等问题，root 权限会放大风险。生产镜像应尽量使用专用非 root 用户运行应用。

```dockerfile
# 不推荐：默认 root 运行
FROM openjdk:17
COPY app.jar /app.jar
CMD ["java", "-jar", "/app.jar"]
```

```dockerfile
# 推荐：创建专用用户运行
FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup app.jar /app/app.jar
USER appuser
CMD ["java", "-jar", "/app/app.jar"]
```

如果应用需要写文件，应只给必要目录授权，而不是给整个文件系统写权限。日志建议输出到 stdout/stderr，业务数据使用 Volume 或外部存储。

# 9. ENTRYPOINT 与 CMD

`ENTRYPOINT` 定义容器的主命令，`CMD` 定义默认参数。两者都建议使用 exec 形式，也就是 JSON 数组形式，避免 shell 包装导致信号转发问题。

```dockerfile
# 推荐：exec 形式，Java 进程可以直接收到 SIGTERM
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```dockerfile
# 不推荐：shell 形式，可能导致信号不能正确传递
CMD java -jar /app/app.jar
```

常见组合：

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
CMD ["--spring.profiles.active=prod"]
```

这样用户可以通过 `docker run image --spring.profiles.active=test` 覆盖默认参数，但主入口仍然固定为 Java 应用。

# 10. HEALTHCHECK

`HEALTHCHECK` 用于告诉 Docker 容器内应用是否真正可用。容器进程存在不代表服务健康，例如 Spring Boot 进程已经启动但数据库连接失败，或者 HTTP 线程池已经不可用。

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

健康检查要注意：

1. 检查命令应该轻量，避免对应用造成额外压力。
2. `timeout` 和 `retries` 要结合应用启动时间配置。
3. 健康检查失败不一定自动重启容器，具体行为取决于 Docker、Compose 或编排平台配置。
4. 生产环境可以区分 liveness 和 readiness，但 Dockerfile 原生 `HEALTHCHECK` 只有一种健康状态。

# 11. 标签策略

镜像标签应该支持版本追踪、快速回滚和问题定位。生产环境不建议只使用 `latest`。

```bash
docker build -t my-app:1.2.3 .
docker tag my-app:1.2.3 my-app:1.2
docker tag my-app:1.2.3 my-app:$(git rev-parse --short HEAD)
docker tag my-app:1.2.3 my-app:$(date +%Y%m%d)
```

常见策略：

| 标签 | 用途 |
|------|------|
| `1.2.3` | 语义化版本，适合发布和回滚 |
| `1.2` | 小版本浮动标签，适合测试或内部环境 |
| Git SHA | 精确定位源码提交 |
| 日期标签 | 方便按构建时间排查 |
| Digest | 最精确，不受标签覆盖影响 |

真正严格的生产发布可以使用镜像 Digest，例如 `my-app@sha256:...`，避免同名标签被覆盖导致部署内容变化。

# 12. 安全实践

Dockerfile 层面的安全实践包括：

1. 不在镜像中写入密码、Token、SSH 私钥和 `.env` 文件。
2. 使用非 root 用户运行应用。
3. 固定基础镜像版本，并定期更新安全补丁。
4. 构建后使用 Trivy、Grype 等工具扫描漏洞。
5. 只安装运行时必需依赖，减少攻击面。
6. 使用 `.dockerignore` 排除敏感文件。
7. 对下载的二进制或脚本做 checksum 校验。

错误示例：

```dockerfile
# 不推荐：密钥会进入镜像层，即使后续删除也可能被追溯
COPY id_rsa /root/.ssh/id_rsa
RUN git clone git@github.com:example/private-repo.git
RUN rm -f /root/.ssh/id_rsa
```

# 13. DO / DON'T

```text
DO:
  - 使用多阶段构建减小最终镜像
  - 把变化少的依赖层放在变化频繁的源码层之前
  - 使用 .dockerignore 减少上下文并排除敏感文件
  - 使用 exec 形式 ENTRYPOINT
  - 使用非 root 用户运行应用
  - 固定基础镜像版本
  - 给生产镜像配置合理健康检查

DON'T:
  - 在镜像中存储密钥、密码和环境文件
  - 在生产镜像中安装不必要的调试工具
  - 依赖 latest 标签做生产发布
  - 用 ADD 替代普通 COPY
  - 在后续层删除前面层产生的大文件并期望镜像变小
  - 在容器中运行 SSH 服务作为常规管理方式
```

# 14. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 如何优化镜像大小 | 多阶段构建、小基础镜像、清理缓存、`.dockerignore`、只保留运行时依赖 |
| Docker 构建缓存如何工作 | 指令顺序、层缓存、前层失效导致后层重建 |
| 为什么不用 `latest` | 标签可变、不可追溯、回滚困难 |
| COPY 和 ADD 区别 | COPY 只复制，ADD 有自动解压和 URL 能力，优先 COPY |
| 为什么使用非 root | 降低容器逃逸和误挂载时的权限风险 |
| ENTRYPOINT 和 CMD 区别 | ENTRYPOINT 是主命令，CMD 是默认参数 |
