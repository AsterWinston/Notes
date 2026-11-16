# 1. 排错的总体方法

容器测试的失败信息往往具有误导性：测试报告说"连接被拒绝"，真正的原因可能是镜像没拉下来、容器启动超时、或者 Ryuk 被防火墙挡住。有效的排错不是从异常信息直接猜答案，而是**按层定位，先确认底层正常，再逐层向上**。

## 1.1 分层定位

```text
第 5 层  测试断言失败         → 业务逻辑、数据、时序
第 4 层  应用连接失败         → 地址、端口、账号、等待
第 3 层  容器未就绪           → 启动日志、健康检查、资源
第 2 层  容器创建失败         → 镜像、命令、挂载、权限
第 1 层  Docker 不可用        → daemon、socket、网络
```

从第 1 层往上查，任何一层不通过就不要继续。实践中最常见的浪费是：Docker 都没启动，却在反复调试 Spring 配置。

## 1.2 先看三类信息

- **`docker info` 的输出**：确认 daemon 可用、版本、存储驱动；
- **容器日志**：`docker logs <id>` 或 `container.getLogs()`，看服务是否真正启动；
- **完整堆栈**：不要只看 `Caused by` 最后一行，向上找第一个 Testcontainers 相关的帧。

## 1.3 最小复现

把问题从测试框架里剥离出来，先用命令行复现：

```bash
docker run --rm -p 5432:5432 -e POSTGRES_PASSWORD=test postgres:16-alpine
```

命令行也起不来，问题在镜像或 Docker；命令行正常而测试失败，问题在配置、等待或网络。**最小复现能把排查范围砍掉一半**。

# 2. 镜像问题

## 2.1 拉取失败

```text
Error response from daemon: manifest for postgres:16-alpine not found
Error response from daemon: Get "https://registry-1.docker.io/v2/": TLS handshake timeout
toomanyrequests: You have reached your pull rate limit
```

| 报错关键词 | 含义 | 处理 |
|-----------|------|------|
| `manifest unknown` | 标签不存在或拼写错误 | 换成真实存在的标签 |
| `TLS handshake timeout` | 网络到 registry 不通 | 配代理、换镜像源 |
| `toomanyrequests` | Docker Hub 限流 | 登录账号、用 mirror、预拉镜像 |
| `unauthorized` | 私有仓库未登录 | `docker login`，配置凭据 |
| `no matching manifest` | 架构不匹配 | 换多架构镜像或指定 platform |

**注意**：`postgres:16-alpine` 在部分企业镜像仓库里可能没有被同步。测试里写死公网标签，到了内网就会失败，最好配合 `TESTCONTAINERS_HUB_IMAGE_NAME_PREFIX` 统一重写。

## 2.2 认证与架构

CI 上频繁拉取公共镜像容易触发限流。缓解方式：CI 中 `docker login` 提高配额、使用企业 registry 或 mirror、在流水线开始阶段集中预拉并重试。用 `echo "$REGISTRY_PASSWORD" | docker login ... --password-stdin` 传凭据，用 `docker image inspect ... --format '{{.Architecture}}'` 检查架构。

Apple Silicon 本地构建、amd64 runner 运行，或者反过来，会出现"镜像能拉但容器起不来"或"启动极慢"。优先选择同时提供 `amd64` 与 `arm64` 的镜像；确实没有时，用 `platform` 指定并接受模拟层的性能损失。

## 2.3 标签策略

| 写法 | 风险 |
|------|------|
| `postgres:latest` | 大版本可能突变，行为不可预测 |
| `postgres:16` | 小版本漂移，通常可接受 |
| `postgres:16.4` | 明确，推荐 |
| `postgres@sha256:...` | 最严格，适合安全要求高的场景 |

**行业实践**：测试镜像固定到小版本，既能获得安全更新，又不会某天突然跨越大版本。生产与测试尽量使用同一版本线。

# 3. 启动与就绪

## 3.1 启动超时

```text
Container postgres:16-alpine started but did not become ready within 60000 ms
```

先分清"启动慢"还是"启动失败"，看日志最后几行：

```bash
docker logs <container-id> --tail 50
```

日志停在初始化脚本执行处，可能是脚本本身有问题；日志显示端口未监听，可能是配置错误；什么都没有，可能是资源不足或镜像架构不对。

## 3.2 等待策略

| 策略 | 适用 | 局限 |
|------|------|------|
| `Wait.forListeningPort()` | 绝大多数 TCP 服务 | 端口开不代表服务可用 |
| `Wait.forLogMessage(...)` | 日志有明确就绪标志 | 依赖日志文本，升级可能变化 |
| `Wait.forHttp("/health")` | 带 HTTP 健康端点的服务 | 需要额外端口与路径 |
| `Wait.forSuccessfulCommand(...)` | 可用命令行自检的服务 | 需要镜像内有对应命令 |

```java
new GenericContainer<>("postgres:16-alpine").withExposedPorts(5432)
    .waitingFor(Wait.forLogMessage(".*ready to accept connections.*\\n", 2))
    .withStartupTimeout(Duration.ofMinutes(2));
```

**注意**：PostgreSQL 初始化阶段会打印一次 "ready"，之后重启再打印一次，所以计数用 `2` 更稳。这类细节只能通过看日志确认。

## 3.3 容器反复退出

```bash
docker inspect <container-id> --format '{{.State.ExitCode}} {{.State.Error}}'
```

| 退出码 | 常见原因 |
|--------|---------|
| 0 | 主进程执行完就退出，镜像不适合常驻 |
| 1 | 配置错误、初始化脚本失败 |
| 137 | 被 SIGKILL，通常是 OOM |
| 139 | 段错误，架构或镜像损坏 |

退出码 137 要重点看 CI 机器内存，而不是业务配置。

# 4. 端口与连接

## 4.1 端口冲突

Testcontainers 使用随机宿主机端口，理论上不会冲突。出现冲突通常是因为 compose 写了固定映射、测试代码手工指定了端口，或上一次运行的容器残留。

Linux 与 macOS 用 `lsof -i :5432`，Windows 用 `Get-NetTCPConnection -LocalPort 5432`。**设计考量**：固定端口唯一的好处是"方便记忆"，代价是并行与残留问题。测试环境一律用随机端口，手工调试时从日志里拿实际端口。

## 4.2 地址写死

```java
// 错误示范：容器端口是随机的
String url = "jdbc:postgresql://localhost:5432/app";
// 正确做法：从容器读取
String url = postgres.getJdbcUrl();
```

容器内访问另一个容器时，`localhost` 指向自己，必须用服务名或网络别名；宿主机访问容器时，才用 `getHost()` + `getMappedPort()`。

## 4.3 容器网络

跨容器连接失败，按顺序检查：两个容器是否在同一 `Network`、目标容器是否设置了 `withNetworkAliases`、目标服务是否监听 `0.0.0.0` 而不是 `127.0.0.1`、防火墙是否拦截。

```java
Network network = Network.newNetwork();
redis.withNetwork(network).withNetworkAliases("cache");
app.withNetwork(network);
```

第三点尤其隐蔽：有些服务默认只监听回环地址，宿主机映射端口能连，容器间却连不上。

# 5. 日志排查

## 5.1 获取容器日志

```java
@Test
void dumpLogs() {
    System.out.println(postgres.getLogs());
    String stderr = postgres.getLogs(OutputFrame.OutputType.STDERR);
}
```

容器启动失败时，`getLogs()` 往往比异常堆栈更有价值，因为它记录了服务进程自己的失败原因。

## 5.2 实时日志消费者

调试阶段可以把容器日志接入测试日志框架：

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
    .withLogConsumer(new Slf4jLogConsumer(LoggerFactory.getLogger("postgres")));
```

`Slf4jLogConsumer` 支持 `withPrefix`，多容器时能区分来源。CI 中打开它会增加日志量，建议只在排查时启用，或用环境变量控制。

## 5.3 常见就绪日志

| 组件 | 就绪标志 |
|------|---------|
| PostgreSQL | `database system is ready to accept connections` |
| MySQL | `ready for connections` |
| Redis | `Ready to accept connections` |
| Kafka | `Kafka Server started` |
| RabbitMQ | `Server startup complete` |
| Elasticsearch | `started` |

**注意**：日志文本会随镜像版本变化，把等待策略押在一条脆弱正则上，升级镜像时可能失效。能用模块自带的等待策略就不要自己写正则。

## 5.4 容器内执行命令

```java
Container.ExecResult result = postgres.execInContainer(
    "psql", "-U", "test", "-c", "SELECT count(*) FROM orders");
System.out.println(result.getStdout());
System.out.println(result.getStderr());
```

这在验证初始化脚本是否生效、数据库是否真的可查时非常有用，比在测试里绕一大圈更直接。

# 6. 数据与状态

## 6.1 初始化脚本失败

脚本失败通常表现为启动超时或直接退出。先确认脚本被复制进容器：

```java
assertThat(postgres.execInContainer("ls", "/docker-entrypoint-initdb.d").getStdout())
    .contains("01_schema.sql");
```

再确认脚本在空数据目录上执行。如果镜像已经初始化过（例如挂载了持久卷），入口脚本不会再执行，这也是"第一次能过、第二次失败"的常见原因。

## 6.2 脏数据与残留卷

共享容器最容易出现脏数据，典型症状是"单独跑通过、全量跑失败"或"顺序一变就失败"。处理方式：每方法 `TRUNCATE ... RESTART IDENTITY CASCADE`；消息测试用唯一 topic 名或 group id；Redis 用 key 前缀并在方法后删除；必要时改用独立容器。

测试容器不要挂载持久卷。一旦挂了卷，容器删除后数据仍在，下次启动会读到旧状态，初始化脚本也不会重跑。用 `docker volume ls --filter "label=org.testcontainers=true"` 可以查看残留卷。

# 7. 最佳实践

## 7.1 镜像与配置

- 固定镜像小版本，禁止 `latest`；
- 用官方模块的容器类，少用 `GenericContainer` 手写配置；
- 初始化脚本与迁移工具统一，避免两份 schema；
- 显式设置字符集、时区、账号密码；
- 关键镜像配 healthcheck 或明确的等待策略。

## 7.2 生命周期

- 默认用 static 容器，配合数据清理；
- 多个测试类共享时抽基类或单例，保证 Spring 上下文复用；
- 本地可开 reuse，CI 必须关闭；
- 不要主动 stop 单例容器，交给 Ryuk；
- 并行度按机器规格设置，重容器单独限流。

## 7.3 测试设计与 CI

- 只验证真实依赖相关的行为，不把单元测试也搬进容器；
- 异步断言用 `Awaitility`，禁止固定 `Thread.sleep`；
- 测试之间不依赖执行顺序，断言业务结果而非容器内部实现；
- 流水线开头跑 `docker info`，预拉镜像并加重试；
- 配置镜像 mirror 或前缀重写；
- DinD 与远程 Docker 必须设 `TESTCONTAINERS_HOST_OVERRIDE`；
- 远程 daemon 启用 TLS，限制来源，并设置 job 超时与容器内存上限。

# 8. 常见错误速查表

| 现象 | 最可能的原因 | 第一步 |
|------|-------------|--------|
| `Could not find a valid Docker environment` | daemon 未启动或无权限 | `docker info` |
| 镜像拉取超时 | 网络或 registry 不通 | `docker pull` 手工验证 |
| `manifest unknown` | 标签写错 | 查 registry 页面 |
| 启动超时 | 资源不足、等待策略不对 | 看容器日志尾部 |
| 连接被拒绝 | 用了 `localhost` 或未启动 | 打印 `getJdbcUrl()` |
| 认证失败 | 账号密码与容器配置不一致 | 对齐容器与客户端配置 |
| 容器退出码 137 | 内存不足被 kill | 降并行、加内存限制 |
| 中文乱码 | 字符集不是 `utf8mb4` | 检查服务端变量 |
| 测试顺序影响结果 | 共享容器无清理 | 加 `TRUNCATE` 或独立容器 |
| CI 上端口连不上 | DinD 地址推断错误 | 设 `TESTCONTAINERS_HOST_OVERRIDE` |
| 容器残留 | Ryuk 被禁用 | 按标签手动清理 |
| 本地过、CI 挂 | 镜像版本或架构不同 | 对齐版本与 platform |

# 9. 练习与自检

**练习一**：故意把镜像标签改成不存在的版本，分别记录"拉取阶段"和"启动阶段"的报错，练习从异常中定位根因。

**练习二**：写一个等待策略错误的测试（只用 `Wait.forListeningPort()` 连需要初始化的 PostgreSQL），观察偶发失败；再改用日志等待，对比稳定性。

**练习三**：构造一个"单独跑通过、全量跑失败"的测试类，用 `TRUNCATE` 与独立容器两种方式修复，比较两者的耗时。

**自检清单**：

- 能按五层模型定位一个容器测试故障；
- 能区分启动超时、连接失败、断言失败三类问题；
- 会用 `getLogs()`、`execInContainer()`、`docker inspect` 收集信息；
- 能背出至少五条最佳实践并解释原因。

# 10. 小结

## 10.1 全模块回顾

| 篇目 | 核心结论 |
|------|---------|
| 概览与架构 | 用一次性真实容器替代不精确的测试替身，依赖 Docker，核心层加模块层分工 |
| 数据库容器 | 显式版本、初始化脚本、随机端口、static 容器加清理 |
| Kafka 与 Redis | 等待真正的就绪信号，异步断言用轮询，注意 listener 与容器网络 |
| Spring Boot 集成 | 属性注入有 `@DynamicPropertySource` 与 `@ServiceConnection` 两条路 |
| 生命周期与复用 | static 是默认，单例与 reuse 各有代价，清理必须可兜底 |
| Compose 与网络 | 复用 YAML 编排，服务名与映射端口是两套地址 |
| CI 与远程 Docker | daemon 可达、资源足够、镜像可拉取，DinD 与远程都要 host override |
| 排错与总结 | 分层定位、先看日志、最小复现、固定版本、控制资源 |

## 10.2 学习路径建议

如果刚接触 Testcontainers，建议按"概览 → 数据库 → Spring Boot 集成 → 生命周期"的顺序推进，先把一个数据库测试跑稳，再扩展到 Kafka、Redis 与 Compose。CI 与排错放在最后，因为它们的很多问题只有真实流水线才会暴露。

容器测试的最终目标不是"用上 Testcontainers"，而是**让集成测试可信、可重复、可维护**。当一套测试能在任意一台装了 Docker 的机器上稳定跑出相同结果时，它才真正具备了回归价值。

本模块到此结束。后续可以结合 `Java/Testing/SpringBootTest` 的测试切片与 `Java/Testing/Junit` 的扩展机制，把容器测试嵌入更完整的测试体系。
