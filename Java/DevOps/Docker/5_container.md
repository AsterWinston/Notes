
# 1. 容器是什么：镜像的运行时实例

容器是镜像运行后的实例。它不是“轻量 VM”，而是宿主机上的一组进程：

1. 通过 **Namespace** 获得隔离视图（PID/NET/MNT/UTS/IPC/USER 等）。
2. 通过 **Cgroups** 获得资源上限（CPU/内存/IO/PIDs 等）。
3. 通过 **镜像只读层 + 容器可写层** 得到独立根文件系统。

工程理解：

1. 容器的“存活”取决于主进程是否存活。
2. 容器可写层不是用来长期存储业务数据的（数据要用 volume/bind mount）。

# 2. 容器生命周期与状态

```text
            docker create
                 |
    +------------v------------+
    |         Created         |
    +------------+------------+
                 |
            docker start
                 |
    +------------v------------+     docker exec/attach
    |         Running         | <----------------------
    +----+----------+----+----+
         |          |    |
     docker stop  docker pause docker kill
         |          |    |
    +----v----+ +---v---+ v
    | Exited  | |Paused | (强制终止)
    +---------+ +-------+
```

常见退出码（速查）：

| 退出码 | 含义 | 常见原因 |
|--------|------|----------|
| 0 | 正常退出 | 程序正常结束 |
| 1 | 通用错误 | 应用启动异常、参数错误 |
| 137 | SIGKILL | OOM Kill 或被强制 kill |
| 143 | SIGTERM | 正常 stop 或编排平台重启 |
| 126 | 不可执行 | 权限/格式错误 |
| 127 | 命令不存在 | ENTRYPOINT/CMD 写错、镜像缺命令 |

# 3. `docker run` 参数体系（按类别记忆）

## 3.1 最常用模板

```bash
docker run -d --name my-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  --restart unless-stopped \
  my-app:1.0
```

## 3.2 端口与网络

```bash
docker run -p 8080:8080 my-app:1.0
docker run --network my-net my-app:1.0
```

提示：端口映射解决的是“外部访问容器”；容器间通信更推荐使用自定义 bridge 网络，通过容器名 DNS 互访。

## 3.3 挂载与数据

```bash
# volume（推荐用于生产数据）
docker run --mount type=volume,source=my-data,target=/data my-app:1.0

# bind mount（常用于开发/配置文件）
docker run --mount type=bind,source=/opt/config,target=/app/config,readonly my-app:1.0
```

## 3.4 资源限制（只列入口，细节见 12）

```bash
docker run --cpus=2 -m 2g --pids-limit=200 my-app:1.0
```

## 3.5 重启策略

```bash
docker run --restart=no my-app:1.0
docker run --restart=on-failure:3 my-app:1.0
docker run --restart=always my-app:1.0
docker run --restart=unless-stopped my-app:1.0
```

注意：重启策略能提高“进程级”健壮性，但不能替代健康检查与依赖治理。

# 4. PID 1、信号与优雅停止（非常关键）

## 4.1 Docker stop 做了什么

默认情况下：

1. `docker stop` 先向容器主进程发送 **SIGTERM**。
2. 等待一段时间（默认 10s）。
3. 超时后发送 **SIGKILL** 强制终止。

```bash
docker stop --time=30 my-app
```

## 4.2 为什么强调 exec 形式 ENTRYPOINT/CMD

如果你用 shell 形式启动（例如 `CMD java -jar ...`），实际主进程可能是 shell，Java 不是 PID 1，信号传递就可能出问题，导致：

1. stop 很慢（直到 SIGKILL）。
2. 应用无法优雅关闭。

推荐写法：

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

如果需要启动脚本，也要在脚本末尾 `exec`：

```sh
#!/bin/sh
exec java $JAVA_OPTS -jar /app/app.jar
```

## 4.3 `--init` 与僵尸进程

容器里如果会派生子进程（例如某些工具或多进程模型），可能出现僵尸进程无法回收。

可选方案：

```bash
docker run --init my-app:1.0
```

或显式使用 tini（见 Java 专章 `14_docker_java.md` 里也有类似内容）。

# 5. 查看、调试与进入容器

## 5.1 查看状态与配置

```bash
docker ps
docker ps -a
docker inspect my-app
```

## 5.2 日志

```bash
docker logs my-app
docker logs -f my-app
docker logs --tail 200 my-app
docker logs --since 30m my-app
```

## 5.3 exec vs attach

```bash
# exec：在容器内启动一个新进程（最常用）
docker exec -it my-app sh

# attach：附加到主进程（不常用，容易把终端信号/输入打到主进程）
docker attach my-app
```

## 5.4 覆盖入口进行排障

容器启动失败或一闪而过时，可以覆盖入口进入：

```bash
docker run -it --entrypoint sh my-app:1.0
```

# 6. 容器与文件：cp 与快照

## 6.1 `docker cp`

```bash
docker cp my-app:/app/logs/app.log ./app.log
docker cp ./config.yaml my-app:/app/config/config.yaml
```

## 6.2 export/import（仅用于理解，不推荐做发布流程）

```bash
docker export my-app > my-app_fs.tar
docker import my-app_fs.tar my-app:snapshot
```

`export/import` 处理的是“容器文件系统快照”，会丢失镜像层历史与元数据。正式离线分发应使用 `save/load`（见 `4_image.md`）。

# 7. 容器清理与资源回收

```bash
docker container prune
docker system prune -a --volumes
```

建议：生产不要直接 `system prune -a --volumes`，先评估是否会误删重要 volume。

# 8. 常见坑与排障提示

## 8.1 容器“一启动就退出”

常见原因：

1. 主进程不是前台进程。
2. ENTRYPOINT/CMD 写错或命令不存在（退出码 127）。
3. 缺少配置/环境变量导致应用启动失败。

推荐排查顺序：

```bash
docker ps -a
docker logs <container>
docker inspect <container>
docker run -it --entrypoint sh <image>
```

## 8.2 `docker stop` 很慢

常见原因：

1. shell 形式启动导致信号没到业务进程。
2. 应用未实现优雅关闭。

解决方向：exec 形式 ENTRYPOINT + 应用优雅关闭配置 + 调整 stop timeout。

# 9. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器的本质是什么 | 被 Namespace/Cgroups 限制的一组进程 |
| 容器为什么会退出 | 主进程退出即容器退出 |
| `docker stop` 做了什么 | SIGTERM 等待超时再 SIGKILL |
| 为什么强调 PID 1 | PID 1 的信号处理与子进程回收行为特殊 |
| exec vs attach 区别 | exec 启动新进程，attach 附着主进程 |
