
# 1. 镜像解决了什么问题

镜像是容器的“交付物”。它把应用运行所需的文件系统内容、默认启动配置、元数据以标准格式打包，并且支持：

1. **分层复用**：多个镜像可共享相同的基础层，减少存储与分发成本。
2. **可追溯与可回滚**：镜像可用 tag/digest 标识，配合仓库实现版本化交付。
3. **环境一致性**：把运行时依赖固化到镜像，减少“本地能跑、线上不行”。

理解镜像的关键点：

1. **镜像 = 多个只读层 + 一份镜像配置（config）**。
2. **Tag 是可变引用，Digest 才是不可变身份**。
3. **构建与分发的性能，本质取决于层的设计与缓存命中率**。

# 2. 镜像分层与写时复制（Copy-on-Write）

镜像由多个只读层叠加组成，容器启动时会额外增加一个可写层。

```text
FROM openjdk:17-slim             -> Layer 1 (Base OS + JDK)   只读
RUN apt-get update && ...        -> Layer 2 (依赖安装)        只读
COPY target/app.jar /app/app.jar -> Layer 3 (应用)            只读
CMD ["java", "-jar", "/app/app.jar"] -> config 元数据（不占层文件大小）
-------------------------------------------------------------
Container Writable Layer         -> 容器可写层（运行时变更）
```

写时复制的直觉：

1. 读：直接从只读层读取。
2. 写：第一次修改只读层文件，会先把文件复制到可写层，再在可写层修改。
3. 删：通过 whiteout 标记“遮蔽”底层文件，而不是改动底层层内容。

工程结论：

1. 不要把业务数据和长期日志写到容器可写层（不可迁移、难备份、易撑爆磁盘）。
2. 镜像层设计决定构建缓存与分发效率（变化少的内容放前面）。

# 3. 镜像标识：Tag、Digest 与多架构

## 3.1 Tag vs Digest

Tag（如 `my-app:1.0`）本质是一个“名字到镜像引用”的指针，可能被覆盖；Digest（如 `my-app@sha256:...`）是内容寻址结果，不可变。

查看镜像 digest：

```bash
docker pull nginx:1.27
docker image inspect nginx:1.27 --format '{{json .RepoDigests}}'
```

生产建议：

1. 发布时保留可读 tag（语义化版本 / Git SHA / 日期）。
2. 关键环境部署尽量用 digest 锁定，避免同名 tag 被覆盖导致“部署内容漂移”。

## 3.2 多架构镜像（manifest list）

同一个 tag 可能对应多套架构（例如 `linux/amd64`、`linux/arm64`）。pull 时客户端会根据平台选择合适的镜像清单项。

常见问题：

1. Apple Silicon / ARM 机器拉到 `arm64`，但线上是 `amd64`，需要明确平台。
2. 某些基础镜像未提供目标架构，会出现 `no matching manifest for ...`。

# 4. Dockerfile 指令（与镜像相关的重点）

这一节只保留“会显著影响镜像体积、缓存、可维护性”的指令点；更系统的最佳实践见 `8_dockerfile_best.md`。

## 4.1 FROM：基础镜像与多阶段

```dockerfile
FROM openjdk:17-slim
FROM maven:3.9-eclipse-temurin-17 AS build
FROM scratch
```

要点：

1. 生产避免 `latest`，固定版本更可复现。
2. 多阶段用于隔离“构建环境”和“运行环境”，最终镜像只保留运行必需内容。

## 4.2 RUN：同层清理

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

要点：清理动作要放在同一个 `RUN`，否则镜像体积通常不会真正变小。

## 4.3 COPY vs ADD

结论：优先 `COPY`。`ADD` 的自动解压/URL 下载是“隐式行为”，容易让构建不可控。

## 4.4 CMD vs ENTRYPOINT

结论：两者都建议 exec 形式（JSON 数组），避免 shell 包装导致信号转发问题。

# 5. 构建：BuildKit 与 buildx（重点是可复现与缓存）

## 5.1 BuildKit 的价值

BuildKit 相比传统 builder，核心改进通常体现在：

1. 更好的并行与缓存。
2. 更好的输出控制（push/load）。
3. 支持 secret、SSH、cache mount（避免把敏感信息写入镜像层）。

## 5.2 buildx 构建多架构镜像

```bash
# 构建并推送多架构镜像（常用于 CI）
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t my-registry/my-app:1.0.0 \
  --push \
  .
```

注意：

1. `--push` 是把结果推到仓库。
2. 如果只想在本机使用，需要 `--load`（多架构场景通常不能一次性把多平台结果 load 到传统本地镜像存储）。

# 6. 镜像分发：push/pull 在做什么

push/pull 不是“传一个大文件”，而是按层上传/下载，层用 digest 标识。

工程结论：

1. 基础层复用可以显著降低带宽与时间。
2. 改动发生在越靠后的层，越能复用前面缓存。

# 7. 常用镜像命令（按目的分类）

## 7.1 查看与定位

```bash
docker image ls
docker image inspect my-app:1.0
docker history my-app:1.0
```

## 7.2 构建与标签

```bash
docker build -t my-app:1.0 .
docker build -f Dockerfile.prod -t my-app:1.0 .

docker tag my-app:1.0 my-registry/my-app:1.0
```

## 7.3 推送与拉取

```bash
docker push my-registry/my-app:1.0
docker pull my-registry/my-app:1.0
```

## 7.4 清理

```bash
docker image prune -a
docker rmi my-app:1.0
```

# 8. save/load 与 export/import 的区别（非常高频）

| 命令 | 对象 | 是否保留镜像层/历史 | 典型用途 |
|------|------|----------------------|----------|
| `docker save` / `docker load` | **镜像** | 保留（层、tag、元数据） | 离线分发镜像、跨环境搬运 |
| `docker export` / `docker import` | **容器文件系统快照** | 不保留（丢层信息与历史） | 临时打包容器文件系统（不推荐当正式发布流程） |

示例：

```bash
# 镜像离线包
docker save -o my-app_1.0.tar my-app:1.0
docker load -i my-app_1.0.tar

# 容器快照（会丢掉历史/层信息）
docker export my-app > my-app_fs.tar
docker import my-app_fs.tar my-app:snapshot
```

# 9. 镜像瘦身策略（与可维护性一起考虑）

| 策略 | 主要收益 | 代价/风险 |
|------|----------|-----------|
| 多阶段构建 | 运行镜像不含构建工具与源码 | Dockerfile 复杂度略升 |
| 合理指令顺序 | 缓存命中率更高，构建更快 | 需要理解依赖变化频率 |
| 同层清理 | 真正减少镜像体积 | 指令写法更“长” |
| 合适基础镜像 | 体积/补丁/兼容性平衡 | Alpine/musl 兼容性风险（Java 常见） |
| `.dockerignore` | 更快构建、更安全 | 需要维护忽略规则 |

# 10. 常见问题与排查

## 10.1 `no space left on device`

先确认是 Docker 数据目录空间不足还是构建上下文过大：

```bash
docker system df
```

## 10.2 `manifest unknown` / `no matching manifest`

1. tag 写错或不存在。
2. 目标架构无对应镜像（多发生在 ARM/特殊平台）。

## 10.3 镜像构建慢

优先检查：

1. 是否缺少 `.dockerignore` 导致上下文巨大。
2. 指令顺序是否导致依赖层频繁失效（先 COPY 全量源码再下依赖）。

# 11. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| Tag 和 Digest 区别 | Tag 可变引用，Digest 不可变内容身份 |
| 镜像为什么能复用 | 分层 + 内容寻址 + layer cache |
| save/load vs export/import | 镜像 vs 容器快照，是否保留层与历史 |
| 多架构镜像怎么做 | manifest list、buildx、--platform |
| 构建慢怎么优化 | `.dockerignore`、指令顺序、缓存命中、BuildKit |
