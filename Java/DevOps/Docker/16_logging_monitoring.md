
# 1. 核心原则：应用输出到 stdout/stderr

容器时代最稳的日志模式：

1. 应用把日志写到 stdout/stderr。
2. Docker 负责收集与转发（logging driver）。
3. 需要集中化时，在宿主机侧或平台侧统一采集。

不要把日志写进容器文件系统当作主路径（容器可重建、文件可能丢失、权限复杂）。

# 2. Docker 日志基本命令

```bash
docker logs -f <container>
docker logs --since=10m <container>
docker logs --tail=200 <container>
```

提示：

1. `docker logs` 是否可用取决于 logging driver（例如部分 driver 不支持）。
2. 调试阶段很常用，但生产更需要集中采集与检索。

# 3. Logging driver：常见选择与取舍

查看当前 driver：

```bash
docker info
```

典型 driver：

1. `json-file`：默认常见，日志落在宿主机文件（JSON）。
2. `local`：Docker 自带的本地日志格式，通常更省空间，并支持轮转。
3. `syslog`/`journald`：对接系统日志。
4. `fluentd` 等：直接转发到日志系统。

工程建议（偏 Docker 原生）：

1. 单机/小规模优先 `local` 或 `json-file + 轮转`。
2. 更复杂的集中化再考虑专门的日志采集链路，但不要把它写成“必须上 ELK/Prometheus”。

# 4. 日志轮转（重点）

如果使用 `json-file`，必须配置轮转，否则宿主机磁盘可能被打爆。

daemon 级别配置（`/etc/docker/daemon.json`，Windows/WSL 路径不同）：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

也可以在容器级别覆盖：

```bash
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  my-app:1.0
```

提示：

1. 配完记得重启 Docker daemon 才生效（daemon 配置）。
2. `max-size/max-file` 是最常用的安全阀。

# 5. 监控：Docker 自带的几个观察入口

## 5.1 `docker stats`：实时资源

```bash
docker stats
```

它能快速告诉你：CPU、内存、网络、块 IO 是否异常。

## 5.2 `docker events`：事件流（排障非常有用）

```bash
docker events --since 30m
```

你能看到：容器重启、OOM、拉取镜像、网络变更等事件。

## 5.3 `docker inspect`：配置与状态真相

```bash
docker inspect <container>
```

重点字段：

1. `State`（退出码、是否 OOMKilled）
2. `HostConfig`（资源限制、重启策略、挂载、capabilities）

# 6. 健康检查：从“进程活着”到“服务可用”

Docker 的 healthcheck 是基础能力，可作为：

1. Compose 的依赖条件（`depends_on` + `service_healthy`）。
2. 运维排障的信号：容器 running 但 unhealthy。

示例：

```Dockerfile
HEALTHCHECK --interval=10s --timeout=3s --retries=5 \
  CMD curl -fsS http://localhost:8080/actuator/health || exit 1
```

注意：

1. healthcheck 应轻量、快速。
2. 不要把健康检查写成“全链路深度探测”，容易把依赖抖动放大。

# 7. 排障清单（日志 + 监控联动）

当你遇到“容器一直重启/服务不可用”：

1. `docker ps -a` 看重启频率与退出码。
2. `docker logs --tail=200 <container>` 看最后一段日志。
3. `docker inspect <container>` 看 `OOMKilled`、资源限制、启动命令。
4. `docker stats` 看是否 CPU/内存顶满。
5. `docker events --since 30m` 看是否有 OOM、kill、pull 等事件。

# 8. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器日志最佳实践 | stdout/stderr + logging driver |
| 为什么要配置轮转 | 防止 json-file 把磁盘打爆 |
| `docker stats/events` 用途 | stats 看资源，events 看生命周期/异常事件 |
| healthcheck 解决什么 | 区分 running 与可用，辅助编排依赖 |
