
# 1. 排障思路

Docker 排障最重要的不是记住很多命令，而是先判断问题层级。一个容器故障通常来自四个层面：

1. **应用层**：程序自身抛异常、配置错误、依赖服务不可用。
2. **容器层**：入口命令错误、镜像缺文件、权限不足、健康检查失败。
3. **宿主机层**：端口冲突、磁盘不足、内核 OOM、iptables 规则异常。
4. **网络与存储层**：DNS 解析失败、网络隔离错误、挂载卷异常。

建议采用“先现象、后日志、再配置、最后宿主机”的排查顺序，避免一开始就陷入复杂细节。

# 2. 容器启动失败

容器启动失败通常要先看日志和退出码。退出码是最快的线索之一。

```bash
# 查看最近日志
docker logs my-app --tail 50

# 查看容器状态和退出码
docker ps -a --filter name=my-app
```

常见退出码：

| 退出码 | 含义 | 常见原因 |
|--------|------|----------|
| 0 | 正常退出 | 程序正常结束 |
| 1 | 通用错误 | 应用启动异常、配置错误、入口命令错误 |
| 137 | 被 SIGKILL 终止 | OOM、被手工 kill、资源不足 |
| 143 | 被 SIGTERM 终止 | 正常停止容器、编排平台重启 |
| 126 | 命令不可执行 | 文件无执行权限或格式错误 |
| 127 | 命令不存在 | ENTRYPOINT/CMD 写错、镜像缺少命令 |

如果容器能启动但立即退出，常见原因包括：

1. `CMD`/`ENTRYPOINT` 指向了错误的可执行文件。
2. 应用缺少配置文件或环境变量。
3. 主进程不是前台进程，容器一启动就结束。
4. 入口脚本没有 `exec`，导致进程链异常。

调试时可以临时覆盖入口进入容器：

```bash
docker run -it --entrypoint sh my-app:1.0
```

然后检查镜像内容、配置文件、权限和启动命令。

# 3. 端口冲突

端口冲突分两类：宿主机端口被占用，或者容器内部服务监听的端口和映射不一致。

```bash
# 查看宿主机端口占用
netstat -tlnp | grep 8080
lsof -i :8080

# 查看容器端口映射
docker port my-app
```

常见错误原因：

1. 宿主机已有其他进程占用了 `-p` 映射端口。
2. 容器内服务监听的端口不是 8080。
3. 应用只监听 `127.0.0.1`，没有监听 `0.0.0.0`。
4. 防火墙或安全组拦截了端口。

解决方法：

```bash
docker run -p 8081:8080 my-app
```

对于容器化应用，还要注意应用内部配置。例如 Spring Boot 的 `server.address` 不能错误地绑定到环回地址，否则外部无法访问。

# 4. 磁盘空间满

磁盘满通常不是单一问题，而是镜像、容器日志、无用卷和构建缓存长期积累的结果。

```bash
docker system df
docker image prune -a --filter "until=24h"
docker system prune -a --volumes
```

| 资源 | 容易膨胀的原因 |
|------|----------------|
| Images | 旧版本镜像未清理 |
| Containers | 停止容器残留，日志持续增长 |
| Local Volumes | 数据未迁移、测试数据未清理 |
| Build Cache | CI/CD 构建缓存累积 |

## 4.1 日志膨胀

容器日志默认可能存储在 Docker 管理目录中，长期运行后会占用大量磁盘。

```bash
# 查看容器日志文件大小
du -sh /var/lib/docker/containers/*/

# 限制 json-file 日志大小
{
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

生产环境建议应用日志输出到 stdout/stderr，再交给日志采集系统统一处理。不要依赖 `docker logs` 作为长期日志存储方案。

# 5. OOM Kill

OOM 是容器化 Java 应用最常见的问题之一。容器内存限制是硬边界，超出后可能被内核直接杀死。

```bash
# 查看内核是否记录 OOM
dmesg | grep -i oom

# 查看容器是否被 OOM 杀死
docker inspect my-app | jq '.[0].State.OOMKilled'
```

排查时要区分两种情况：

1. **容器总内存超限**：被内核 OOM Killer 杀死，退出码通常是 137。
2. **应用自己抛 OutOfMemoryError**：JVM 还有机会打印堆栈、生成 heap dump。

解决方案通常包括：

1. 增加容器内存限制。
2. 调整 JVM 堆比例。
3. 检查堆外内存是否过大。
4. 排查内存泄漏、缓存无限增长、线程过多、Direct Buffer 过大。

# 6. 网络不通

网络问题通常要分层看：容器是否连得上宿主机、是否能解析 DNS、是否能访问外部网络、是否能访问其他容器服务。

```bash
# 查看容器网络信息
docker inspect my-app | jq '.[0].NetworkSettings.Networks'

# 进入容器测试
docker exec -it my-app sh
ping google.com
nslookup mysql

# 查看 DNS 配置
docker exec my-app cat /etc/resolv.conf

# 检查 Docker 网络规则
iptables -L DOCKER -n
```

常见原因：

1. DNS 配置异常，容器无法解析域名。
2. 容器连接了错误的网络。
3. 宿主机防火墙拦截了端口。
4. 服务名解析错误，Compose 或自定义网络配置有误。

# 7. 镜像拉取失败

镜像拉取失败通常涉及网络、认证、镜像名、Tag 或 Registry 配置。

```bash
docker pull my-app:1.0
```

常见提示和含义：

| 报错 | 常见原因 |
|------|----------|
| `pull access denied` | 没有权限、未登录、仓库私有 |
| `manifest unknown` | Tag 不存在、镜像名写错 |
| `x509: certificate signed by unknown authority` | Registry 证书不受信任 |
| `i/o timeout` | 网络不通、DNS 问题、代理问题 |

如果是私有仓库，先确认：

1. 是否执行了 `docker login`。
2. 仓库地址是否正确。
3. 证书是否配置正确。
4. 网络是否可达。

# 8. 容器内文件权限问题

很多容器启动失败不是程序问题，而是文件权限问题。例如非 root 用户无法读取配置文件或写入日志目录。

排查方式：

```bash
docker exec -it my-app sh
id
ls -l /app
touch /tmp/test
```

常见解决方法：

1. 在 Dockerfile 中使用 `--chown` 复制文件。
2. 提前创建并授权需要写入的目录。
3. 避免给整个文件系统放宽权限。
4. 尽量让应用只写日志目录和临时目录。

# 9. 快速自检清单

```bash
□ docker info                    # Docker 服务状态
□ docker ps -a                   # 容器状态和退出码
□ docker logs my-app --tail 50   # 应用日志
□ docker inspect my-app          # 容器配置、挂载、网络、健康状态
□ docker stats --no-stream       # 资源使用情况
□ docker system df               # 镜像/容器/卷空间
□ ss -tlnp | grep docker-proxy   # 宿主机端口映射
□ iptables -L DOCKER -n          # 网络规则
□ dmesg | grep -i oom            # 是否发生 OOM
□ cat /etc/docker/daemon.json    # Docker 守护进程配置
```

# 10. 排障方法论

一个高效的排障顺序通常是：

1. 看现象：容器起不来、端口不通、日志异常、性能下降。
2. 看日志：应用日志、Docker 日志、内核日志。
3. 看容器配置：入口命令、环境变量、挂载、网络、限制。
4. 看宿主机资源：CPU、内存、磁盘、端口、网络、内核。
5. 复现最小场景：缩小问题范围，逐层验证。

# 11. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器启动失败怎么排查 | 日志、退出码、入口命令、`docker inspect` |
| OOM 如何定位 | 137、`OOMKilled`、`dmesg`、JVM 堆和堆外内存 |
| 端口冲突如何处理 | 宿主机占用、端口映射、服务监听地址 |
| 网络不通怎么查 | DNS、网络模式、iptables、容器内连通性 |
| 镜像拉取失败怎么查 | 权限、Tag、网络、证书、Registry 配置 |
