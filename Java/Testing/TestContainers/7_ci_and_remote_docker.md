# 1. CI 中的三个前提

本地跑通不等于 CI 跑通。容器测试在流水线上失败，通常不是测试代码的问题，而是三个前提没满足：**Docker daemon 可达、资源足够、镜像拉得下来**。排错时按这三个方向逐个确认，比盯着测试日志有效。

```text
┌──────────────────────────────────────────────┐
│ CI Runner                                    │
│  ┌────────────┐   Docker API   ┌──────────┐  │
│  │ 测试 JVM    │ ─────────────▶ │ daemon   │  │
│  └────────────┘                └────┬─────┘  │
│                          ┌──────────▼─────┐  │
│                          │ 容器：DB/Kafka  │  │
│                          └────────────────┘  │
└──────────────────────────────────────────────┘
```

## 1.1 Docker daemon 可达

Runner 必须能连上 daemon。三种常见形态：

| 形态 | 连接方式 | 说明 |
|------|---------|------|
| 宿主 Docker | 默认 socket | GitHub Actions ubuntu runner、自建 agent |
| Docker-in-Docker | `DOCKER_HOST=tcp://docker:2375` | 容器内再跑一个 daemon |
| 远程 Docker | `DOCKER_HOST=tcp://host:2376` + TLS | 独立 Docker 主机 |

第一步永远是运行 `docker info`。这一步失败，后面所有排查都没有意义。

## 1.2 资源与镜像

每个数据库容器约占 100 MB 内存，Kafka 与 Elasticsearch 更高。CI 机器常见 2 核 7 GB，并行跑三四个容器就会紧张。资源不足的表现往往是"启动超时"或"进程被 OOM Killer 杀掉"，而不是明确错误。

CI 每次都是新环境，没有本地镜像缓存。Docker Hub 限流、企业网络代理、镜像架构不匹配，都会让拉取失败。**把镜像预热和镜像加速当成 CI 配置的一部分**，而不是偶发故障。

# 2. GitHub Actions 实践

## 2.1 直接用宿主 Docker

`ubuntu-latest` runner 预装 Docker，最简单的方式是什么都不配：

```yaml
name: test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
      - name: 预拉取测试镜像
        run: |
          docker pull postgres:16-alpine
          docker pull redis:7.4-alpine
      - name: 运行测试
        run: mvn -B verify
```

Testcontainers 会自动发现 runner 上的 Docker socket，无需设置 `DOCKER_HOST`。这是最推荐的方式，启动快、调试直观、没有嵌套网络问题。

## 2.2 必须用 DinD 的场景

有些 runner 本身跑在容器里，或者团队希望隔离 Docker 环境。这时用 service 容器提供 daemon：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      docker:
        image: docker:27-dind
        privileged: true
        env:
          DOCKER_TLS_CERTDIR: ""
        options: >-
          --health-cmd "docker info"
          --health-interval 5s
          --health-retries 20
    env:
      DOCKER_HOST: tcp://docker:2375
      TESTCONTAINERS_HOST_OVERRIDE: docker
      TESTCONTAINERS_RYUK_DISABLED: "true"
    steps:
      - uses: actions/checkout@v4
      - run: mvn -B verify
```

| 变量 | 作用 |
|------|------|
| `DOCKER_HOST` | 指向 DinD 的 daemon 地址 |
| `TESTCONTAINERS_HOST_OVERRIDE` | 告诉测试"映射端口在 docker 主机上"，而不是 `localhost` |
| `TESTCONTAINERS_RYUK_DISABLED` | 受限环境下关闭 Ryuk，由流水线清理 |

**注意**：`DOCKER_TLS_CERTDIR: ""` 关闭了 DinD 的 TLS。它只在**同一 job 的私有网络**内可接受，绝不能把 2375 暴露到公网。

# 3. GitLab CI 与 Jenkins

## 3.1 GitLab DinD

```yaml
variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""
  TESTCONTAINERS_HOST_OVERRIDE: docker
  TESTCONTAINERS_RYUK_DISABLED: "true"

test:
  image: maven:3.9-eclipse-temurin-17
  services:
    - name: docker:27-dind
      alias: docker
  script:
    - mvn -B verify
```

`alias: docker` 让 `docker` 这个主机名在 job 容器内可解析。`TESTCONTAINERS_HOST_OVERRIDE` 必须指向它，否则测试会尝试连 `localhost:<映射端口>`，而那个端口在 job 容器里并不存在。

## 3.2 Jenkins agent

Jenkins 的情况更分散：物理机或虚拟机 agent 装好 Docker、授予 Jenkins 用户 socket 权限即可；Kubernetes agent 需要挂载 `/var/run/docker.sock` 或启动 DinD sidecar；受限 Pod 可能没有特权，此时应考虑远程 Docker。

```groovy
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh 'docker info'
                sh 'mvn -B verify'
            }
        }
    }
}
```

在流水线最前面加一步 `docker info`，能让环境问题在测试开始前暴露，而不是等到某个容器启动超时。

# 4. 远程 Docker

## 4.1 连接配置

远程 Docker 把 daemon 放在独立主机上，测试 JVM 通过 TCP 连接，适合 runner 不便安装 Docker 或需要统一管理容器资源的场景。

```bash
export DOCKER_HOST=tcp://docker.example.com:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/etc/docker/certs
export TESTCONTAINERS_HOST_OVERRIDE=docker.example.com
```

| 变量 | 说明 |
|------|------|
| `DOCKER_HOST` | daemon 地址，TLS 端口通常是 2376 |
| `DOCKER_TLS_VERIFY` | 开启证书校验 |
| `DOCKER_CERT_PATH` | 存放 `ca.pem`、`cert.pem`、`key.pem` 的目录 |
| `TESTCONTAINERS_HOST_OVERRIDE` | 映射端口的访问主机，必须是测试 JVM 可达的地址 |

## 4.2 为什么需要 HOST_OVERRIDE

容器端口映射在 **daemon 所在主机**上，不是跑测试的机器上。Testcontainers 默认按"daemon 在本机"推断地址，远程场景下会得到 `localhost`，于是连接失败。`TESTCONTAINERS_HOST_OVERRIDE` 就是用来纠正这个推断的。

```text
测试机 192.168.1.10                 Docker 主机 192.168.1.20
┌──────────────┐   Docker API      ┌────────────────────┐
│ 测试 JVM      │ ────────────────▶ │ daemon              │
│              │ ◀── 连接映射端口 ──┼──│ 容器 :32768  │   │
└──────────────┘   192.168.1.20    └────────────────────┘
```

## 4.3 TLS 与网络边界

远程 daemon 等价于**远程 root 权限**。必须启用 TLS 双向认证、禁用 2375 明文端口；用防火墙限制来源 IP，只允许 CI 网段；使用独立 Docker 主机，不与生产业务共用；定期轮换证书与凭据。

> **警告**：把未认证的 Docker socket 暴露到公网，等于把整台机器的控制权交出去。远程 Docker 必须在私有网络内，并开启 TLS。

# 5. 缓存与加速

## 5.1 镜像预热与重试

最有效的优化是提前拉镜像，并配合有限重试：

```bash
for i in 1 2 3; do docker pull postgres:16-alpine && break || sleep 5; done
```

企业内网通常有 Harbor、Nexus 等镜像仓库，Testcontainers 支持统一重写 Docker Hub 前缀：

```bash
export TESTCONTAINERS_HUB_IMAGE_NAME_PREFIX=registry.example.com/dockerhub-mirror/
```

配置后 `postgres:16-alpine` 会被解析为 `registry.example.com/dockerhub-mirror/postgres:16-alpine`，适合规避公网限流。

## 5.2 缓存边界

- 依赖缓存：Maven `~/.m2`、Gradle `~/.gradle/caches`；
- 镜像缓存：自建 runner 可保留本地镜像，托管 runner 每次都是新的；
- 测试结果缓存：不要缓存容器运行状态，容器必须每次新建。

`testcontainers.reuse.enable=true` 只对**同一台机器上的多次 JVM 启动**有意义。托管 runner 每次全新，reuse 不生效；自建 runner 开启又可能读到上次的脏数据。**CI 中应显式关闭 reuse**。

# 6. 资源限制

## 6.1 容器内存

Testcontainers 默认不限制单个容器内存。为了防止某个容器吃光 runner 内存，可以显式限制：

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withCreateContainerCmdModifier(cmd ->
        cmd.getHostConfig().withMemory(512L * 1024 * 1024));
```

`withMemory` 单位是字节。限制过低会导致数据库启动失败，日志里常见 `out of memory` 或进程被 kill。

## 6.2 并行度与超时

测试 JVM 自身也占内存，CI 上常用 `MAVEN_OPTS="-Xmx1g"`。并行度要结合 runner 规格设置，2 核机器把 JUnit 并行度设成 8，容器会互相争抢 CPU，总时长反而更长。经验值是 `min(核数, 4)`，重容器再减半。

| 超时 | 位置 | 建议 |
|------|------|------|
| 容器启动 | `withStartupTimeout` | 本地 60s，CI 120–180s |
| 单次请求 | 客户端配置 | 适当放大，避免误判 |
| Job 总时长 | `timeout-minutes` | 留出镜像拉取与重试余量 |

**注意**：不要用无限超时掩盖问题。超时频繁触发，说明镜像太大、资源不足或等待策略不对，应优化根因。

# 7. 安全边界

能访问 Docker socket 的进程可以启动特权容器、挂载任意目录、读取任意文件，等价于 root。因此不要在不信任的代码（如外部 PR）上运行容器测试，不要把 socket 挂载进测试容器内部。

镜像方面固定版本、避免 `latest`，关键镜像可用 digest 固定，只从可信 registry 拉取并启用扫描。数据库密码、registry 凭据放 CI secret，不要写进代码或 compose 文件；测试容器的密码可以是固定弱口令，但**不能与任何真实环境共用**。

# 8. 常见错误

| 现象 | 根因 | 处理 |
|------|------|------|
| `Cannot connect to the Docker daemon` | `DOCKER_HOST` 未设置或 daemon 未启动 | 先跑 `docker info` |
| 容器启动后连接被拒绝 | DinD 下地址推断错误 | 设置 `TESTCONTAINERS_HOST_OVERRIDE` |
| 镜像拉取 429 | Docker Hub 限流 | 预热、mirror、私有仓库 |
| 启动超时但本地正常 | CI 资源少、镜像未缓存 | 提高超时、减少并行、预热 |
| 进程被 OOM kill | 同时启动太多容器 | 限制容器内存、降低并行度 |
| 容器在 job 结束后残留 | Ryuk 被禁用且无清理步骤 | 加清理步骤或恢复 Ryuk |
| 架构不匹配 | 本地 arm64、runner amd64 | 统一架构或显式 platform |
| 流水线偶发失败 | 无重试、超时太紧 | 加有限重试，合理放大超时 |

# 9. 练习与自检

**练习一**：在 GitHub Actions 上用宿主 Docker 跑一个 PostgreSQL 容器测试，记录从 checkout 到测试完成的总时长，再单独统计镜像拉取耗时。

**练习二**：用 DinD 方式重跑同一个测试，故意不设置 `TESTCONTAINERS_HOST_OVERRIDE`，观察报错信息，再补上变量确认通过。

**练习三**：配置 `TESTCONTAINERS_HUB_IMAGE_NAME_PREFIX` 指向一个内网镜像仓库（可用本地 registry 模拟），验证镜像名被正确重写。

**自检清单**：

- 能说出 CI 容器测试的三个前提；
- 能画出 DinD 与远程 Docker 的网络路径；
- 知道 `TESTCONTAINERS_HOST_OVERRIDE` 解决什么问题；
- 能列出至少三项 CI 安全注意事项。

# 10. 小结

| 主题 | 核心结论 |
|------|---------|
| 前提 | daemon 可达、资源足够、镜像可拉取 |
| GitHub Actions | 优先用宿主 Docker，DinD 才需要额外变量 |
| DinD | `DOCKER_HOST` + `HOST_OVERRIDE` + 视情况关闭 Ryuk |
| 远程 Docker | 映射端口在远端，必须设 host override 并启用 TLS |
| 加速 | 预热镜像、镜像前缀重写、依赖缓存 |
| 安全 | socket 等价 root，隔离网络、固定镜像、保护凭据 |

环境配好之后，剩下的就是"出问题时怎么查"。下一篇汇总常见故障：镜像拉取失败、端口冲突、启动超时、日志定位，并给出最佳实践与全模块总结。
