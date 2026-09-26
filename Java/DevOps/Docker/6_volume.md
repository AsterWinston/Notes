
# 1. 为什么需要数据卷

容器的文件系统由“镜像只读层 + 容器可写层”组成。容器可写层有两个天然问题：

1. **不适合持久化**：容器删除后可写层一起消失。
2. **不适合迁移与备份**：它不在 Docker 卷体系里，管理、备份、迁移都不方便。

因此，业务数据、需要保留的日志文件、数据库文件目录等，应通过挂载的方式落到可管理的持久化存储上。

# 2. 三种挂载方式对比

| 方式 | 数据位置 | 持久化 | 共享 | 适用场景 |
|------|---------|--------|------|---------|
| **Volume** | Docker 管理目录（如 `/var/lib/docker/volumes/`） | 是 | 是 | **生产推荐**、数据库、状态数据 |
| **Bind Mount** | 宿主机任意路径 | 是 | 是 | 开发联调、挂载配置文件/证书 |
| **tmpfs** | 内存 | 否 | 否 | 临时敏感数据、临时缓存 |

直觉图：

```text
Volume:                        Bind Mount:                 tmpfs:
┌──────────┐                  ┌──────────┐                ┌──────────┐
│ Container │                  │ Container │                │ Container │
│  /data    │                  │  /app     │                │  /tmp     │
└────┬─────┘                  └────┬─────┘                └────┬─────┘
     │ Docker 管理                   │ 用户管理                   │ 内存
┌────▼────────────────┐       ┌────▼──────────┐          ┌──────▼──────┐
│ /var/lib/docker/...  │       │ /opt/app/...  │          │   tmpfs(RAM) │
└─────────────────────┘       └───────────────┘          └─────────────┘
```

# 3. `--mount` vs `-v`：推荐使用 `--mount`

两者都能完成挂载，但 `--mount` 语义更清晰、更不容易写错。

```bash
# volume
docker run --mount type=volume,source=my-data,target=/data my-app:1.0

# bind
docker run --mount type=bind,source=/opt/config,target=/app/config,readonly my-app:1.0

# tmpfs
docker run --mount type=tmpfs,destination=/tmp,tmpfs-size=64m my-app:1.0
```

对应的 `-v` 形式：

```bash
docker run -v my-data:/data my-app:1.0
docker run -v /opt/config:/app/config:ro my-app:1.0
```

# 4. Volume 的生命周期与常用命令

## 4.1 创建、查看、删除

```bash
docker volume create my-data
docker volume ls
docker volume inspect my-data

docker volume rm my-data
docker volume prune
```

## 4.2 Volume 与容器解耦

容器删除不等于 volume 删除：

1. `docker rm` 不会删除挂载的命名 volume。
2. `docker compose down -v` 才会删除 compose 创建的 volume。

工程建议：

1. 生产的数据卷应命名并纳入备份策略。
2. 谨慎使用 `volume prune`，避免误删。

# 5. Bind Mount：开发联调与配置挂载

典型场景：

1. 挂载配置文件/证书：容器无需重建镜像即可变更配置。
2. 开发联调：挂载源码/产物以实现快速迭代（注意性能与权限）。

```bash
docker run \
  --mount type=bind,source=/opt/my-app/config,target=/app/config,readonly \
  my-app:1.0
```

常见坑：

1. 容器内非 root 用户没有写权限（解决思路：只读挂载配置；需要写则调整宿主机目录属主/权限，或在镜像内创建对应 UID/GID）。
2. Windows/WSL2 bind mount 大量小文件可能明显变慢。

# 6. tmpfs：只存内存，不落盘

```bash
docker run --mount type=tmpfs,destination=/tmp,tmpfs-size=64m my-app:1.0
```

适合：临时敏感文件（短生命周期 token）、临时解压目录、对磁盘写入敏感的临时缓存。

# 7. 备份与恢复（Volume 级别）

备份的核心思路：用一个临时容器同时挂载 volume 与宿主机目录，把 volume 内容打成 tar。

```bash
# 备份到当前目录
docker run --rm \
  --mount type=volume,source=my-data,target=/data \
  --mount type=bind,source=$(pwd),target=/backup \
  alpine \
  tar czf /backup/my-data.tar.gz -C /data .

# 恢复
docker run --rm \
  --mount type=volume,source=my-data,target=/data \
  --mount type=bind,source=$(pwd),target=/backup \
  alpine \
  tar xzf /backup/my-data.tar.gz -C /data
```

注意：数据库等有状态服务备份通常需要一致性方案（停写、dump、或数据库自身备份机制）。上面的 tar 更适合通用文件类数据卷。

# 8. 常见问题与排查

## 8.1 数据“丢了”

优先确认写入位置：

1. 是否写到了容器可写层，而不是挂载路径。
2. 是否误用了匿名卷（`VOLUME` 指令可能隐式创建匿名卷）。

定位：

```bash
docker inspect my-app
docker volume ls
```

## 8.2 权限问题（Permission denied）

常见原因：

1. 容器内应用用户与宿主机目录属主不匹配。
2. 绑定挂载目录权限不足。

建议：

1. 生产数据用 volume，尽量避免依赖宿主机路径权限细节。
2. 非 root 镜像配合 `COPY --chown` 与明确的工作目录权限。

# 9. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 为什么不把数据写可写层 | 容器删即丢、难备份迁移、易撑爆磁盘 |
| Volume 与 Bind 区别 | Docker 管理 vs 用户管理、生产推荐 volume |
| tmpfs 用于什么 | 只存内存、临时敏感数据、不落盘 |
| 如何备份 volume | 临时容器挂载 volume + tar 备份 |
